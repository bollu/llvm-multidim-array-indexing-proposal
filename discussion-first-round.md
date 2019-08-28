# Collection of first round of discussion points from the mailing list about the RFC

### Arguments for extending GEP:


The most obvious why for me was changing GEP to allow variable-sized
multi-dimensional arrays in the first argument, such as

    %1 = getelementptr double, double* %ptr, inrange i64 %i, inrange i64 %j

(normally GEP would only allow a single index argument for a
pointer-typed base pointer).
Since %A has variable size, there is not enough information to compute
the result, we need to pass at least the stride of the innermost
dimension, such as:

    %1 = getelementptr double, double* %A, inrange i64 %i, inrange i64
%j, i64 %n

It should be clear that this breaks a lot of assumptions then went
into writing code that handle GEPs, not only the number of arguments
the GEP should have. As a result, existing code may trigger
assertions, crash, or miscompile when encountering such a modified
GEP. I think it is unfeasible to change all code to properly handle
the new form at once.


2.
Johannes' interpretation is to add some kind of metadata to GEPs, in
addition to the naive computation, such as:

    %offset1= mul i64. %i, %n
    %offset2 = add i64, %j, %offset1
    %1 = getelementptr double, double* %A, inrange i64 %offset2 [ "multi-dim"(i64 %n) ]

The code above uses operand bundle syntax.  During our discussing for this RFC
we briefly discussed metadata, which unfortunately do not allow referencing
local SSA values.


3.  For completeness, here is Johannes other suggestion without modifying
GEP/Load/Store:

    %offset1= mul i64. %i, %n
    %offset2= add i64, %j, %offset1
    %1 = getelementptr double, double* %A, inrange i64 %offset2
    %2 = llvm.multidim.access.pdouble.pdouble.i64.i64.i64.i64(double* %A, i64 %n, i64 %m, i64 %i, i64 %j)
    %cmp = icmp eq %1, %2
    call void @llvm.assume(i1 %cmp)


### Type-system issues with keeping it as an intrinsic

##### Kaylor, Andrew's concern about the type system validity of the intrinsic

```
<result> = llvm.multidim.array.index.* <ty> <ty>* <ptrval> {<stride>, <idx>}*
```

It isn't clear to me what that means. The later example is also in a somewhat generalized form:

```
%arrayidx = llvm.multidim.array.index.* i64 i64* %A, %str_1, %idx_1, %str_2, %idx_2
```

Trying to expand this into something concrete it looks to me like the extra
value-less type argument ('i64' immediately following the intrinsic name) won't
work, and if I'm reading it correctly that's a necessary element. The GEP
instruction accepts a raw type as a pseudo-argument. It does this to prepare
for the coming transition to opaque pointers, and it can do it because it's a
first class instruction. However, an intrinsic can't do this. And yet without
it we won't know what to use for the GEP's source element type "argument" after
pointer's lose their pointee type.

That is, given opaque pointers, the above will convert to:

```
%arrayidx = call i64 @llvm.multidim.array.index.i64.p0.i64.i64.i64.i64 void* %A, i64 %str_1, i64 %idx_1, i64 %str_2, i64 %idx_2
```

where the `void*` informs us nothing about the element size...

### Extend the `inrange` keyword for multidim (Philip Reames, Johannes Doerfert)
- Extend the `inrange` mechiasm
-  a type-system extension to make multi-dimensional "inrange" GEPs for
   symbolic sized arrays a natural GEP extension


### Should this even be an intrinsic? 

> Apart from all that, I'm pretty disappointed to see this as an
> intrinsic though. GEP is such a fundamental part of addressing in LLVM
> that bifurcating it into an intrinsic for either a language or an
> analysis seems like we'd be papering over a language deficiency. --- (Tim Northover)


> Adding ‘experiemental’ intrinsics is cheap and easy, so if you think this
> direction has promise, I’d recommend starting with that, building out the
> optimizers that you expect to work and measure them in practice --- (Chris Lattner)



### Beg, Borrow, Steal from MLIR

- IMHO the issue is representation since LLVM-IR does not have the
primitives that MLIR has, specifically there is no Memref type that --
in contrast to LLVM-IR pointer types are multi-dimensional, and -- in
contrast to LLVM-IR array types can have dependent and dynamic shape.

Adding a MemRef type this would go quite deep into LLVM-IR
fundamentals. Do you think it would be worth it? -- (Michael Kruse)



### Summary by Johannes

Access-centric:
 - new "operand/metadata" to encode the multi-dim coordinate:
     %coord_3D = llvm.coord.3d(%N, %M, %K, %i, %j, %p)
     %val = load %p, [coord] %coord_3D
   Benefits: pointer computation is not disturbed, even if it is
             transformed, e.g., part of the pointer computation is
             hoisted and coalesced. Annotated operations are known to
             be UB if the bounds are violated.
   Caveats: New load operand need to be hidden, e.g., as operand bundle
   operands for calls. Alternatively, coord can be passed as metadata.
 - new "operand/metadata" to encode a shape:
     %shape_3D = llvm.shape.3d(%N, %M, %K)
     %val = load %p, [shape] %shape_3D
   Benefits: see above. When a shape and `inrange` is provided it
             applies to all dimensions of the shape.
   Caveats: see above. Reconstruction of coordinate needs to happen
            still. Though, statically complex parts, e.g., guessing the
            shape and computing when offsets would be "out-of-range",
            is much simpler.

Type-based:
 - Have multi-dimensional variable sized arrays. This was discussed in
   various contexts and I cannot summarize all the pros and cons.
 
GEP-centric:
 - an intrinsic-based solutions applied to GEPs
 - add the multi-dim coordinate inputs directly to GEPs (no intrinsic)
 - build the coordinate as above and assume it equal to the GEP:
     call @llvm.assume(icmp eq %gep, %coord_3D)
