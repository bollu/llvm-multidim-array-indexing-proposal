# Introducing a new multidimensional array indexing intrinsic

## The request for change: A new intrinsic, `llvm.multidim.array.index.*`.

We propose the addition of a new family of intrinsics, `llvm.multidim.array.index.*`. 
This will allow us to represent array indexing into an array `A[d1][d2][..][dk]`
as `llvm.multidim.array.index.k A d1, d2, d3, ... dk` instead of flattening
the information into `gep A, d1 * n1 + d2 * n2 + ... + dk * nk`. The former
unflattened representation is advantageous for analysis and optimisation. It also
allows us to represent array indexing semantics of languages such as Fortran
and Chapel, which differs from that of C based array indexing.
	
## Motivating the need for a multi-dimensional array indexing intrinsic
There are primarily one of two views of array indexing involving
multiple dimensions most languages take, which we discuss to motivate 
the need for a multi-dimensional array index. This consideration impacts
the kinds of analysis we can perform on the program. In Polly, we care about
dependence analysis, so the examples here will focus on that particular problem.

Let us consider an array indexing operation of the form:
```cpp
int ex1(int n, int m, B[n][m], int x1, int x2, int y1, int y2) {
	__builtin_assume(x1 != x2);
	__builtin_assume(y1 != y2);
	B[x1][y1] = 1;
	printf("%d", B[x2][y2]);
	exit(0);
}
```

One would like to infer that the array indices _interpreted as tuples_
`(x1, y1)` and `(x2, y2)` do not have the same value, due to the guarding asserts
that `x1 != x2` and `y1 != y2`. As a result, the write `B[x1][y1] = 1` can 
in no way interfere with the value of `B[x2][y2]`. Consquently,
we can optimise the program into:


```cpp
int ex1_opt(int n, int m, B[n][m], int x1, int x2, int y1, int y2) {
    // B[x1][y1] = 1 is omitted because the result 
    // cannot be used: 
    //  It is not used in the print and then the program exits
	printf("%d", B[x2][y2]);
	exit(0);
}
```

However, alas, this is illegal, for the C language does not provide
semantics that allow the final inference above. It is conceivable that
`x1 != x2, y1 != y2`, but the indices do actually alias, since
according to C semantics, the two indices alias if the _flattened
representation of the indices alias_. Consider the parameter
values:

```
n = m = 3
x1 = 1, y1 = 0; B[x1][y1] = nx1+y1 = 3*1+0=3
x2 = 0, y2 = 3; B[x2][y2] = nx2+y2 = 3*0+3=3
```

Hence, the array elements `B[x1][y1]` and `B[x2][y2]` _can alias_, and
so the transformation proposed in `ex1_opt` is unsound in general.


In contrast, many langagues other than C require that index
expressions for multidimensional arrays have each component within the
array dimension for that component. As a result, in the example above,
the index pair `(0,3)` would be out-of-bounds. In languages with these
semantics, one can infer that the indexing:

`[x1][y1] != [x2][y2]` iff `x1 != x2 || y1 != y2`.

While this particular example is not very interesting, it shows the
spirit of the lack of expressiveness in LLVM we are trying to
improve. 

Julia, Fortran, and Chapel are examples of such languages which target
LLVM.

Currently, the LLVM semantics of `getelementptr` talk purely of
the C style flattened views of arrays. This inhibits the ability of the optimiser
to understand the multidimensional examples as given above, and we
are forced to make conservative assumptions, inhibiting optimisation. 
This information must ideally be expressible in LLVM, such that LLVM's optimisers and
alias analysis can use this information to model multidimensional-array
semantics.

There is a more realistic (and involved) example in [Appendix A](#Appendix-A)
in the same spirit as the above simple example, but one a compiler might
realistically wish to perform.

## Evaluation of the impact of the intrinsic on accuracy of dependence analysis

- This has been implemented in an exprimental branch of Polly, and was used 
on the COSMO climate weather model. This greatly helped increase the accuracy
of Polly's analysis, since we eliminated the guessing game from the array analysis.

- This has also been implemented as part of a GSoC effort to unify
Chapel and Polly. 

- Molly, the distributed version of Polly written by Michael Kruse for his PhD
  also implemented a similar scheme. In his use-case, optimistic run-time checks
  with delinearization was not possible, so this kind of intrinsic was _necessary_
  for the application, not just _good to have_. More details are available
  in his PhD thesis: [Lattice QCD Optimization and Polytopic Representations of Distributed Memory](https://tel.archives-ouvertes.fr/tel-01078440).
  In particular, Chapter 9 contains a detailed discussion.

# Representations

## Intrinsic

### Syntax
```
<result> = llvm.multidim.array.index.* <ty> <ty>* <ptrval> {<stride>, <idx>}*
```

### Overview:

The `llvm.multidim.array.index.*` intrinsic is used to get the address of 
an element from an array. It performs address calcuation only and 
does not access memory. It is similar to `getelementptr`. However, it imposes
additional semantics which allows the optimiser to provide better optimisations
than `getlementptr`.



### Arguments:

The first argument is always a type used as the basis for the calculations. The
second argument is always a pointer, and is the base address to start the
calculation from. The remaining arguments are a list of pairs. Each pair
contains a dimension stride, and an offset with respect to that stride.


### Semantics:

`llvm.multidim.array.index.*` represents a multi-dimensional array index, In particular, this will
mean that we will assume that all indices `<idx_i>` are non-negative.

Additionally, we assume that, for each `<str_i>`, `<idx_i>` pair, that 
`0 <= idx_i < str_i`.

Optimizations can assume that, given two llvm.multidim.array.index.* instructions with matching types:

```
llvm.multidim.array.index.* <ty> <ty>* <ptrvalA> <strA_1>, <idxA_1>, ..., <strA_N>, <idxA_N>
llvm.multidim.array.index.* <ty> <ty>* <ptrvalB> <strB_1>, <idxB_1>, ..., <strB_N>, <idxb_N>
```

If `ptrvalA == ptrvalB` and the strides are equal `(strA_1 == strB_1 && ... && strA_N == strB_N)` then:

- If all the indices are equal (that is, `idxA_1 == idxB_1 && ... && idxA_N == idxB_N`), 
  then the resulting pointer values are equal.

- If any index value is not equal (that is, there exists an `i` such that `idxA_i != idxB_i`),
  then the resulting pointer values are not equal.


##### Address computation:
Consider an invocation of `llvm.multidim.array.index.*`:

```
<result> = call @llvm.multidim.array.index.* <ty> <ty>* <ptrval> <str_0>, <idx_0>, <str_1> <idx_1>, ..., <str_n> <idx_n>
```

If the pairs are denoted by `(str_i, idx_i)`, where `str_i` denotes the stride
and `idx_i` denotes the index of the ith pair, then the final address (in bytes)
is computed as:

```
ptrval + len(ty) * [(str_0 * idx_0) + (str_1 * idx_1) + ... (str_n * idx_n)]
```

## Transitioning to `llvm.multidim.array.index.*`: Allow `multidim_array_index` to refer to a GEP instruction:

This is a sketch of how we might gradually introduce the `llvm.multidim.array.index.*`
intrinsic into LLVM without immediately losing the analyses
that are performed on `getelememtptr` instructions. This section
lists out some possible choices that we have, since the authors
do not have a "best" solution.

##### Choice 1: Write a `llvm.multidim.array.index.*` to `GEP` pass, with the `GEP` annotated with metadata

This pass will flatten all `llvm.multidim.array.index.*` expressions into a `GEP` annotated with metadata. This metadata will indicate that the index expression computed by the lowered GEP is guaranteed to be in a canonical form which allows the analysis
to infer stride and index sizes.

A multidim index of the form:
```
%arrayidx = llvm.multidim.array.index.* i64 i64* %A, %str_1, %idx_1, %str_2, %idx_2
```

is lowered to:

```
%mul1 = mul nsw i64 %str_1, %idx_1
%mul2 = mul1 nsw i64 %str_2, %idx_2
%total = add nsw i64 %mul2, %mul1
%arrayidx = getelementptr inbounds i64, i64* %A, i64 %total, !multidim !1
```
with guarantees that the first term in each multiplication is the stride
and the second term in each multiplication is the index. (What happens
if intermediate transformation passes decide to change the order? This seems
complicated).

**TODO:** Lookup how to attach metadata such that the metadata can communicate
which of the values are strides and which are indeces



# Caveats

Currently, we assume that the array shape is immutable. However, we will need to deal with 
being able to express `reshape` like primitives where the array shape can be mutated. However,
this appears to make expressing this information quite difficult: We now need to attach the shape
information to an array per "shape-live-range". 


## Appendix-A

##### A realistic, more involved example of dependence analysis going wrong

```cpp
// In an array A of size (n0 x n1), 
// fill a subarray of size (s0 x s1)
// which starts at an offset (o0 x o1) in the larger array A
void set_subarray(unsigned n0, unsigned n1,
	unsigned o0, unsigned o1,
	unsigned s0, unsigned s1,
	float A[n0][n1]) {
	for (unsigned i = 0; i < s0; i++)
		for (unsigned j = 0; j < s1; j++)
				S: A[i + o0][j + o1] = 1;
}
```
We first reduce this index expression to a sum of products:

```
(i + o0) * n1 + (j + o1) = n1i + n1o0 + j + o1
ix(i, j, n0, n1, o0, o1) = n1i + n1o0 + j + o1
```

`ix` is the index expression which `LLVM` will see, since it is fully
flattened, in comparison with the multi-dimensional index expression
`index:[i + o0][j + o1] | sizes:[][n1]`.

We will now show _why_ this multi-dimensional index is not always correct,
and why guessing for one language will not work for another:

Consider a call `set_subarray(n0=8, n1=9, o0=4, o1=6, s0=3, s1=6)`. At face
value, this is incorrect if we view the array as 2D grid, since the size
of the array is `(n0, n1) = (8, 9)`, but we are writing an array of size
`(s0, s1) = (3, 6)` starting from `(o0, o1) = (4, 6)`. Clearly, we will
exceed the width of the array, since `(s1 + o1 = 6 + 6 = 12) > (n1 = 9)`.
However, now think of the array as a flattened 1D representation. In this
case, the total size of the array is `n1xn2 = 8x9 = 72`, while the largest
element we will access is at the largest value of `(i, j)`. That is, 
`i = s0 - 1 = 2`, and `j = s1 - 1 = 5`.

The largest index will be `ix(i=2, j=5, n0=8, n1=9, o0=4, o1=6) = 8*2+8*4+5+6=59`.
Since `59 < 72`, we are clearly at _legal_ array indices, by C semantics!

The definition of the semantics of the language **changed the illegal
multidimensional access** (which is illegal since it exceeds the `n1`
dimension), into a **legal flattened 1D access** (which is legal since the
flattened array indices are inbounds).

LLVM has no way of expressing these two different semantics. Hence, we are
forced to:
1. Consider flattened 1D accesses, which makes analysis of index expressions
equivalent to analysis of polynomials over the integers, which is hard.
2. Guess multidimensional representations, and use them at the expense of
soundness bugs as shown above.
3. Guess multidimensional representations, use them, and check their validity
at runtime, causing a runtime performance hit. This implementation follows
the description from the paper [Optimistic Delinearization of Parametrically Sized Arrays](optim-delin-parametrically-sized-arrays).


Currently, Polly opts for option (3), which is to emit runtime checks. If
the run-time checks fail, then Polly will not run its optimised code. Instead,
It keeps a copy of the unoptimised code around, which is run in this case.
Note that this effectively doubles the amount of performance-sensitive code
which is finally emitted after running Polly.

Ideally, we would like a mechanism to directly express the multidimensional
semantics, which would eliminate this kind of guesswork from Polly/LLVM,
which would both make code faster, and easier to analyze.

## References
- [The chapel language specification](https://chapel-lang.org/docs/1.13/_downloads/chapelLanguageSpec.pdf)
- [Fortran 2003 standard](http://www.j3-fortran.org/doc/year/04/04-007.pdf}{Fortran 2003 standard)
- [C++ subscripting](http://eel.is/c++draft/expr.sub)
- [Michael Kruse's PhD thesis: Lattice QCD Optimization and Polytopic Representations of Distributed Memory](https://tel.archives-ouvertes.fr/tel-01078440).
- [Molly source code link for the intrinsic](https://github.com/Meinersbur/llvm/blob/molly/include/llvm/IR/IntrinsicsMolly.td#L3)
-[Optimistic Delinearization of Parametrically Sized Arrays](optim-delin-parametrically-sized-arrays)

[optim-delin-parametrically-sized-arrays]: http://delivery.acm.org/10.1145/2760000/2751248/p351-grosser.pdf?ip=93.3.109.183&id=2751248&acc=CHORUS&key=4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E6D218144511F3437&__acm__=1557948443_2f56675c6d04796f27b84593535c9f70

