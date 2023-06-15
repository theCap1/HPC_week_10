# HPC_week_10
## Homework for the 10th week

### 11 Back to the Compiler

#### 11.1 Getting Started

I consider the following three hints the most useful:
- Use of ``__restrict__`` keyword: I didn't even know that the compiler would assume pointer aliasing reather than not. 
- Embedding SVE assembly code directly into C and C++ code: This is good to know if one wants to to have the best of both worlds. Ease of coding with a high level programming language as well as performance in a crucial part of the program. Although, it was well worth mentioning that in this way it is not possible at present for outputs to contain an SVE vector or predicate value.
- pragmas only affect loop statements immediately following it: I could have seen me debugging this for hours already if I didn't know...

The Pragma that is now explained is the `assume_safety` directive:
This directive is used for loop when the developer can ensure, that there are no data dependencies between the individual iterations. In many cases the compiler would even be able to determine that itself but in some cases this can not be anticipated without any high level information from the developer. In the *coding consideration* info guide the example shows this very well. Here the info of what data is supposed to be processed is provided at runtime. This can lead to unpredictable behavior, if the same data needed to be processt multiple times and/ or in a specific order. With the knowledge that this will never be the case the pragma `assume_safety` can be provided as directive for the compiler to enable vectorization anyways.

#### 11.2 SVE enabling

In order to convince the comoiler to generate SVE code it is necessary to add the compiler arguments
- -ftree-vectorize: Enable auto-vectorization
- -msve-vector-bits=256: specify the vector length. In this case 256 bit

Also the `-march` flag shall be set to specify the target architecture that supports SVE. I used `-march=armv8-a+sve` for *ARMv8-A*