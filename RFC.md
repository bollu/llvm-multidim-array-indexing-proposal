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

int main() {
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

Hence, we propose the addition of a new intrinsic / instruction, 
henceforth referred to as `array_index(...)` (name subject to bikeshedding),
which enables us to represent such "multidimensional" views of arrays.

# Specification


# Example
## allocating something of that shape
## indexing into something of that shape
## loop analyses that uses this

# Caveats

# Math
Please look at [the LaTeX document](formalism/main.pdf) which formalises
the indexing expressions used by the languages in consideration
(C++, Fortran, Chapel) and the corresponding equivalences.

# References
- [The chapel language specification](https://chapel-lang.org/docs/1.13/_downloads/chapelLanguageSpec.pdf)
- [Fortran 2003 standard](http://www.j3-fortran.org/doc/year/04/04-007.pdf}{Fortran 2003 standard)
- [C++ subscripting](http://eel.is/c++draft/expr.sub)
- [Molly's use of the intrinsic](https://github.com/Meinersbur/llvm/blob/molly/include/llvm/IR/IntrinsicsMolly.td#L3)
