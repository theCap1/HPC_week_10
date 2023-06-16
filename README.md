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
GCC:
- -ftree-vectorize: Enable auto-vectorization
- -msve-vector-bits=256: specify the vector length. In this case 256 bit

CLANG
- -Rpass=loop-vectorize: Enable auto-vectorization
- -mllvm -aarch64-sve-vector-bits-max=256: specify the vector length. In this case 256 bit

Also the `-march` flag shall be set to specify the target architecture that supports SVE. I used `-march=armv8-a+sve` for *ARMv8-A*.

The following code that includes SVE instructions was disassembled:

<details>
  <summary> g++ </summary>

```s
      0:   b4000260        cbz     x0, 4c <_Z12triad_simplemPKfS0_Pf+0x4c>
   4:   91001025        add     x5, x1, #0x4
   8:   91001044        add     x4, x2, #0x4
   c:   cb050065        sub     x5, x3, x5
  10:   cb040064        sub     x4, x3, x4
  14:   f10060bf        cmp     x5, #0x18
  18:   fa588880        ccmp    x4, #0x18, #0x0, hi  // hi = pmore
  1c:   d2800004        mov     x4, #0x0                        // #0
  20:   54000189        b.ls    50 <_Z12triad_simplemPKfS0_Pf+0x50>  // b.plast
  24:   25a01fe0        whilelo p0.s, xzr, x0
  28:   25b9c002        fmov    z2.s, #2.000000000000000000e+00
  2c:   2518e141        ptrue   p1.b, vl32
  30:   a5444021        ld1w    {z1.s}, p0/z, [x1, x4, lsl #2]
  34:   a5444040        ld1w    {z0.s}, p0/z, [x2, x4, lsl #2]
  38:   65a18440        fmad    z0.s, p1/m, z2.s, z1.s
  3c:   e5444060        st1w    {z0.s}, p0, [x3, x4, lsl #2]
  40:   91002084        add     x4, x4, #0x8
  44:   25a01c80        whilelo p0.s, x4, x0
  48:   54ffff41        b.ne    30 <_Z12triad_simplemPKfS0_Pf+0x30>  // b.any
  4c:   d65f03c0        ret
  50:   1e201002        fmov    s2, #2.000000000000000000e+00
  54:   d503201f        nop
  58:   bc647841        ldr     s1, [x2, x4, lsl #2]
  5c:   bc647820        ldr     s0, [x1, x4, lsl #2]
  60:   1f020020        fmadd   s0, s1, s2, s0
  64:   bc247860        str     s0, [x3, x4, lsl #2]
  68:   91000484        add     x4, x4, #0x1
  6c:   eb04001f        cmp     x0, x4
  70:   54ffff41        b.ne    58 <_Z12triad_simplemPKfS0_Pf+0x58>  // b.any
  74:   d65f03c0        ret
```

</details>

<details>
  <summary> clang </summary>

```s
   0:   b4000260        cbz     x0, 4c <_Z12triad_simplemPKfS0_Pf+0x4c>
   4:   91001025        add     x5, x1, #0x4
   8:   91001044        add     x4, x2, #0x4
   c:   cb050065        sub     x5, x3, x5
  10:   cb040064        sub     x4, x3, x4
  14:   f10060bf        cmp     x5, #0x18
  18:   fa588880        ccmp    x4, #0x18, #0x0, hi  // hi = pmore
  1c:   d2800004        mov     x4, #0x0                        // #0
  20:   54000189        b.ls    50 <_Z12triad_simplemPKfS0_Pf+0x50>  // b.plast
  24:   25a01fe0        whilelo p0.s, xzr, x0
  28:   25b9c002        fmov    z2.s, #2.000000000000000000e+00
  2c:   2518e141        ptrue   p1.b, vl32
  30:   a5444021        ld1w    {z1.s}, p0/z, [x1, x4, lsl #2]
  34:   a5444040        ld1w    {z0.s}, p0/z, [x2, x4, lsl #2]
  38:   65a18440        fmad    z0.s, p1/m, z2.s, z1.s
  3c:   e5444060        st1w    {z0.s}, p0, [x3, x4, lsl #2]
  40:   91002084        add     x4, x4, #0x8
  44:   25a01c80        whilelo p0.s, x4, x0
  48:   54ffff41        b.ne    30 <_Z12triad_simplemPKfS0_Pf+0x30>  // b.any
  4c:   d65f03c0        ret
  50:   1e201002        fmov    s2, #2.000000000000000000e+00
  54:   d503201f        nop
  58:   bc647841        ldr     s1, [x2, x4, lsl #2]
  5c:   bc647820        ldr     s0, [x1, x4, lsl #2]
  60:   1f020020        fmadd   s0, s1, s2, s0
  64:   bc247860        str     s0, [x3, x4, lsl #2]
  68:   91000484        add     x4, x4, #0x1
  6c:   eb04001f        cmp     x0, x4
  70:   54ffff41        b.ne    58 <_Z12triad_simplemPKfS0_Pf+0x58>  // b.any
  74:   d65f03c0        ret
```

</details>

</br>

#### 11.3 Optimization levels and flags

In this section compiled code shall be discussed when different optimization flags are set. Since different optimization levels alias a large number of optimization flags ([see documentation](https://www.genome.gov/)) the focus is restricted to optimizations on vectorizations.

Most optimizations are completely disabled at `-O0` or if an `-O` level is not set on the command line, even if individual optimization flags are specified. The result is the most simple disassembled SIMD code:

<details>
  <summary> -O0 optimization </summary>

```s
   0:   d100c3ff        sub     sp, sp, #0x30
   4:   f9000fe0        str     x0, [sp, #24]
   8:   f9000be1        str     x1, [sp, #16]
   c:   f90007e2        str     x2, [sp, #8]
  10:   f90003e3        str     x3, [sp]
  14:   f90017ff        str     xzr, [sp, #40]
  18:   14000015        b       6c <_Z12triad_simplemPKfS0_Pf+0x6c>
  1c:   f94017e0        ldr     x0, [sp, #40]
  20:   d37ef400        lsl     x0, x0, #2
  24:   f9400be1        ldr     x1, [sp, #16]
  28:   8b000020        add     x0, x1, x0
  2c:   bd400001        ldr     s1, [x0]
  30:   f94017e0        ldr     x0, [sp, #40]
  34:   d37ef400        lsl     x0, x0, #2
  38:   f94007e1        ldr     x1, [sp, #8]
  3c:   8b000020        add     x0, x1, x0
  40:   bd400000        ldr     s0, [x0]
  44:   1e202800        fadd    s0, s0, s0
  48:   f94017e0        ldr     x0, [sp, #40]
  4c:   d37ef400        lsl     x0, x0, #2
  50:   f94003e1        ldr     x1, [sp]
  54:   8b000020        add     x0, x1, x0
  58:   1e202820        fadd    s0, s1, s0
  5c:   bd000000        str     s0, [x0]
  60:   f94017e0        ldr     x0, [sp, #40]
  64:   91000400        add     x0, x0, #0x1
  68:   f90017e0        str     x0, [sp, #40]
  6c:   f94017e1        ldr     x1, [sp, #40]
  70:   f9400fe0        ldr     x0, [sp, #24]
  74:   eb00003f        cmp     x1, x0
  78:   54fffd23        b.cc    1c <_Z12triad_simplemPKfS0_Pf+0x1c>  // b.lo, b.ul, b.last
  7c:   d503201f        nop
  80:   d503201f        nop
  84:   9100c3ff        add     sp, sp, #0x30
  88:   d65f03c0        ret
```

</details>