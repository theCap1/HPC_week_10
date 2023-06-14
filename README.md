# HPC_week_10
## Homework for the 10th week

### 11 Back to the Compiler

#### 11.1 Getting Started

I consider the following three hints the most useful:
- Use of ``__restrict__`` keyword: I didn't even know that the compiler would assume pointer aliasing reather than not. 
- Embedding SVE assembly code directly into C and C++ code: This is good to know if one wants to to have the best of both worlds. Ease of coding with a high level programming language as well as performance in a crucial part of the program. Although, it was well worth mentioning that in this way it is not possible at present for outputs to contain an SVE vector or predicate value.
- pragmas only affect loop statements immediately following it: I could have seen me debugging this for hours already if I didn't know...