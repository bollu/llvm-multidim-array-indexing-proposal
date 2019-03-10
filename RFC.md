# Introducing a new multidimensional array indexing intrinsic
	
## Motivating the need for a multi-dimensional array indexing intrinsic
There are primarily one of two views of array indexing involving
multiple dimensions most languages take, which we discuss to motivate 
the need for a multi-dimensional array index. This consideration impacts
the kinds of analysis we can perform on the program. In Polly, we care about
dependence analysis, so the examples here will focus on that particular problem.

Let us consider an array indexing operation of the form:
```cpp
int ex1(int n, int m, B[n][m], int x1, int x2, int y1, int y2) {
	assert(x1 != x2);
	assert(y1 != y2);
	B[x1][y1] = 1;
	printf("%d", B[x2][y2]);
	exit(0);
}
```

One would like to infer that since the array indeces _interpreted as tuples_
`(x1, y1)` and `(x2, y2)` do not alias, due to the guarding asserts
that `x1 != x2` and `y1 != y2`. the write `B[x1][y1] = 1` can 
in no way interfere with the value of `B[x2][y2]`. Consquently,
we can optimise the program into:


```cpp
int ex1_opt(int n, int m, B[n][m], int x1, int x2, int y1, int y2) {
	printf("%d", B[x2][y2]);
	exit(0);
}
```

However, alas, this is illegal, for the C language does not provide these
semantics. It is conceivable that `x1 != x2, y1 != y2`, but the indeces
do actually alias, since according to C semantics, the two indeces alias
if the _flattened representation of the indeces alias_. So, in this case,
pick the parameter values:

```
n = m = 3
x1 = 1, y1 = 0; B[x1][y1] = nx1+y1 = 3*1+0=3
x2 = 0, y2 = 3; B[x2][y2] = nx2+y2 = 3*0+3=3
```

Hence, the arrays _do alias_, and the transformation proposed in `ex1_opt`
is unsound in general.

However, there are languages that provide this "tuple-indexed" semantics,
where one can infer that the indexing:
`[x1][y1] != [x2][y2]` iff `x1 != x2 || y1 != y2`.

Julia, Fortran, and Chapel are examples of such languages which target
LLVM.

Currently, the LLVM semantics of `getelementptr` talk purely of
the C style flattened views of arrays. This inhibits the ability of the optimiser
to understand the multidimensional examples as given above, and we
are forced to make conservative assumptions, inhibiting optimisation. 
This information must ideally be expressible in LLVM, such that LLVM's optimisers and
alias analysis can use this information to model multidimensional-array
semantics.



## The request for change: A new instruction, `multidim_array_index`.

We propose the addition of a new instruction, called `multidim_array_index`,
which allows one to represent such multidimensional array accesses without
flattening.


## Evaluation of the impact of the intrinsic on accuracy of dependence analysis
- This has been implemented in an exprimental branch of Polly, and was used 
on the COSMO climate weather model. This greatly helped increase the accuracy
of Polly's analysis, since we eliminated the guessing game from the array analysis.

- This has also been implemented as part of a GSoC effort to unify
Chapel and Polly. 

- Molly had this as well (TODO: add references to this.)

# Representations

## Instruction / Intrinsic

### Transitioning: Allow `multidim_array_index` to refer to a GEP instruction:

```llvm
i1 foo(... arr):
    %gep = getelementptr ...
    %multidim = multidim_array_array(%size0, %size1, %ix0, %ix1, %arr, %gep)
    %arr.load = load %multidim
```

We will ensure that calls such as `isGEP` and `dyn_cast<GEP>` will proxy
through `%gep` to return the correct values. This will ensure that we don't
lose the current optimiser when trying to teach the optimiser about
`multidim_array_index`.

## Metadata

###  Transitioning:


## Appendix: A second, more involved example of dependence analysis going wrong

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
(i + o0) * n0 + (j + o1) = n0i + n0o0 + j + o1
ix(i, j, n0, n1, o0, o1) = n0i + n0o0 + j + o1
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
Since `59 < 72`, we are clearly at _legal_ array indeces, by C semantics!

The definition of the semantics of the language **changed the illegal
multidimensional access** (which is illegal since it exceeds the `n1`
dimension), into a **legal flattened 1D access** (which is legal since the
flattened array indeces are inbounds).

LLVM has no way of expressing these two different semantics. Hence, we are
forced to:
1. Consider flattened 1D accesses, which makes analysis of index expressions
equivalent to analysis of polynomials over the integers, which is hard.
2. Guess multidimensional representations, and use them at the express of
soundness bugs as shown above.
3. Guess multidimensional representations, use them, and check their validity
at runtime, causing a runtime performance hit.


Currently, Polly opts for option (3), which is to emit runtime checks, which
will cause polly to bail out on the above program on the shown parameter
configuration, since it detects that the multidimensional view that was
guessed is incorrect.

Ideally, we would like a mechanism to directly express the multidimensional
semantics, which would eliminate this kind of guesswork from Polly/LLVM,
which would both make code faster, and easier to analyze.

## References
- [The chapel language specification](https://chapel-lang.org/docs/1.13/_downloads/chapelLanguageSpec.pdf)
- [Fortran 2003 standard](http://www.j3-fortran.org/doc/year/04/04-007.pdf}{Fortran 2003 standard)
- [C++ subscripting](http://eel.is/c++draft/expr.sub)
- [Molly's use of the intrinsic](https://github.com/Meinersbur/llvm/blob/molly/include/llvm/IR/IntrinsicsMolly.td#L3)
