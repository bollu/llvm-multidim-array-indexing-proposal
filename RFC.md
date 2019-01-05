# Introducing a new multidimensional array indexing intrinsic

Currently, the LLVM semantics of `getelementptr` talk purely of
"flattened" views of arrays. This inhibits the ability of the optimiser
to understand the multidimensional aliasing structure. This information
must ideally be expressible in LLVM, such that LLVM's optimisers and
alias analysis can use this information to model multidimensional-array
semantics.

As a quick example, according to C++ semantics, consider the program:

```cpp
#include <assert.h>
#include <stdio.h>
int A[3][3];

int ex1() {
    printf("A[0][3]: %d, ", &A[0][3] - &A[0][0]);
    printf("A[1][0]: %d\n", &A[1][0] - &A[0][0]);
    assert(&A[0][3] == &A[1][0]);
    return 0;
}
```

The `assert` will succeed, and the `printf` will print `A[0][3]: 3, A[1][0]: 3`.
This is *not* UB, since the C++ standard defined `A[i] := *(A + i)`, and so
the multidimensional array index is "correct" (with respect to C++ semantics).

However, this is only true for C/C++ since the semantics of the array indexing
is defined in terms of the "flattened" / 1-D representation.

For *most* array-based programming languages, such as Fortran, Julia, and Chapel,
this behaviour is pathological. We would like to infer: `i1 ≠ j1 & i1 ≠ j2  ⇒ &A[i1][i2] ≠ &A[j1][j2]`,
which is information that cannot be expressed in LLVM at this time.

Consider a second example, where guessing array dimensions leads to incorrect
analysis:


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
exceed the width of the array, since `(s1 + o1 = 6 + 6 = 12) > (n1 = 9)`. However,
now think of the array as a flattened 1D representation. In this case, the
total size of the array is `n1xn2 = 8x9 = 72`, while the largest element
we will access is at the largest value of `(i, j)`. That is,
`i = s0 - 1 = 2`, and `j = s1 - 1 = 5`.

The largest index will be `ix(i=2, j=5, n0=8, n1=9, o0=4, o1=6) = 8*2+8*4+5+6=59`.
Since `59 < 72`, we are clearly at _legal_ array indeces, by C semantics!

The definition of the semantics of the language **changed the illegal
multidimensional access** (which is illegal since it exceeds the `n1` dimension),
into a **legal flattened 1D access** (which is legal since the flattened array indeces
are inbounds).

LLVM has no way of expressing these two different semantics. Hence, we are
forced to:
1. Consider flattened 1D accesses, which makes analysis of index expressions
equivalent to analysis of polynomials over the integers, which is hard.
2. Guess multidimensional representations, and use them at the express of
soundness bugs as shown above.
3. Guess multidimensional representations, use them, and check their validity
at runtime, causing a runtime performance hit.


Currently, Polly opts for option (3), which is to emit runtime checks, which
will cause polly to bail out on the above program on the shown parameter configuration,
since it detects that the multidimensional view that was guessed is incorrect.



Ideally, we would like a mechanism to directly express the multidimensional 
semantics, which would eliminate this kind of guesswork from Polly/LLVM, 
which would both make code faster, and easier to analyze.


# The request for change: A new instruction, `multidim_array_index`.

We propose the addition of a new instruction, called `multidim_array_index`,
which allows one to represent such multidimensional array accesses without
flattening.


# Evaluation
- This has been implemented in an exprimental branch of Polly, and was used 
on the COSMO climate weather model. This greatly helped increase the accuracy
of Polly's analysis, since we eliminated the guessing game from the array analysis.

- This has also been implemented as part of a GSoC effort to unify
Chapel and Polly. 

- Molly had this as well (TODO: add references to this.)

# References
- [The chapel language specification](https://chapel-lang.org/docs/1.13/_downloads/chapelLanguageSpec.pdf)
- [Fortran 2003 standard](http://www.j3-fortran.org/doc/year/04/04-007.pdf}{Fortran 2003 standard)
- [C++ subscripting](http://eel.is/c++draft/expr.sub)
- [Molly's use of the intrinsic](https://github.com/Meinersbur/llvm/blob/molly/include/llvm/IR/IntrinsicsMolly.td#L3)
