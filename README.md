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

#### 11.3 Optimization levels and flags

In this section compiled code shall be discussed when different optimization flags are set. Since different optimization levels alias a large number of optimization flags ([see documentation](https://www.genome.gov/)) the focus is restricted to optimizations on vectorizations.

Most optimizations are completely disabled at `-O0` or if an `-O` level is not set on the command line, even if individual optimization flags are specified. The result is the most simple disassembled SIMD code:

<details>
  <summary> -O0 optimization </summary>

```s
   0:   b4000140        cbz     x0, 28 <_Z12triad_simplemPKfS0_Pf+0x28>
   4:   d2800004        mov     x4, #0x0                        // #0
   8:   bc647840        ldr     s0, [x2, x4, lsl #2]
   c:   1e202800        fadd    s0, s0, s0
  10:   bc647821        ldr     s1, [x1, x4, lsl #2]
  14:   1e212800        fadd    s0, s0, s1
  18:   bc247860        str     s0, [x3, x4, lsl #2]
  1c:   91000484        add     x4, x4, #0x1
  20:   eb04001f        cmp     x0, x4
  24:   54ffff21        b.ne    8 <_Z12triad_simplemPKfS0_Pf+0x8>  // b.any
  28:   d65f03c0        ret
```

</details>

It is clearly visible that only general purpose registers and SIMD operations are compiled. The priority of the compiler in optimization level 0 lies on compile time so no efforts are made to analyze the program on possible vectorization oportunities.



<details>
  <summary> -O1 optimization </summary>

```s
   0:   b4000140        cbz     x0, 28 <_Z12triad_simplemPKfS0_Pf+0x28>
   4:   d2800004        mov     x4, #0x0                        // #0
   8:   bc647840        ldr     s0, [x2, x4, lsl #2]
   c:   1e202800        fadd    s0, s0, s0
  10:   bc647821        ldr     s1, [x1, x4, lsl #2]
  14:   1e212800        fadd    s0, s0, s1
  18:   bc247860        str     s0, [x3, x4, lsl #2]
  1c:   91000484        add     x4, x4, #0x1
  20:   eb04001f        cmp     x0, x4
  24:   54ffff21        b.ne    8 <_Z12triad_simplemPKfS0_Pf+0x8>  // b.any
  28:   d65f03c0        ret
```

</details>

The disassembled code does not doffer from the code generated from optimization level `-O0`. This level does not include any optimization flags that have the purpose of vectorization.

<details>
  <summary> -O2 optimization </summary>

```s
   0:   b4000160        cbz     x0, 2c <_Z12triad_simplemPKfS0_Pf+0x2c>
   4:   d2800004        mov     x4, #0x0                        // #0
   8:   1e201002        fmov    s2, #2.000000000000000000e+00
   c:   d503201f        nop
  10:   bc647841        ldr     s1, [x2, x4, lsl #2]
  14:   bc647820        ldr     s0, [x1, x4, lsl #2]
  18:   1f020020        fmadd   s0, s1, s2, s0
  1c:   bc247860        str     s0, [x3, x4, lsl #2]
  20:   91000484        add     x4, x4, #0x1
  24:   eb04001f        cmp     x0, x4
  28:   54ffff41        b.ne    10 <_Z12triad_simplemPKfS0_Pf+0x10>  // b.any
  2c:   d65f03c0        ret
```

</details>

In optimization level `-O2` the compiler still does not do any vectorization. Compared to the `-O1` or `-O2` optimization level we get a performance boost from the `FMADD` operation where the final addition sub operation is doesn't use up an entire CPU cycle.

<details>
  <summary> -O3 optimization </summary>

```s
   0:   b40002e0        cbz     x0, 5c <_Z12triad_simplemPKfS0_Pf+0x5c>
   4:   91001044        add     x4, x2, #0x4
   8:   91001025        add     x5, x1, #0x4
   c:   cb050065        sub     x5, x3, x5
  10:   cb040064        sub     x4, x3, x4
  14:   eb05009f        cmp     x4, x5
  18:   0420e3e6        cntb    x6
  1c:   9a859084        csel    x4, x4, x5, ls  // ls = plast
  20:   d10020c5        sub     x5, x6, #0x8
  24:   eb05009f        cmp     x4, x5
  28:   d2800004        mov     x4, #0x0                        // #0
  2c:   540001a9        b.ls    60 <_Z12triad_simplemPKfS0_Pf+0x60>  // b.plast
  30:   04a0e3e5        cntw    x5
  34:   25a01fe0        whilelo p0.s, xzr, x0
  38:   25b9c002        fmov    z2.s, #2.000000000000000000e+00
  3c:   2518e3e1        ptrue   p1.b
  40:   a5444021        ld1w    {z1.s}, p0/z, [x1, x4, lsl #2]
  44:   a5444040        ld1w    {z0.s}, p0/z, [x2, x4, lsl #2]
  48:   65a18440        fmad    z0.s, p1/m, z2.s, z1.s
  4c:   e5444060        st1w    {z0.s}, p0, [x3, x4, lsl #2]
  50:   8b050084        add     x4, x4, x5
  54:   25a01c80        whilelo p0.s, x4, x0
  58:   54ffff41        b.ne    40 <_Z12triad_simplemPKfS0_Pf+0x40>  // b.any
  5c:   d65f03c0        ret
  60:   1e201002        fmov    s2, #2.000000000000000000e+00
  64:   d503201f        nop
  68:   bc647841        ldr     s1, [x2, x4, lsl #2]
  6c:   bc647820        ldr     s0, [x1, x4, lsl #2]
  70:   1f020020        fmadd   s0, s1, s2, s0
  74:   bc247860        str     s0, [x3, x4, lsl #2]
  78:   91000484        add     x4, x4, #0x1
  7c:   eb04001f        cmp     x0, x4
  80:   54ffff41        b.ne    68 <_Z12triad_simplemPKfS0_Pf+0x68>  // b.any
  84:   d65f03c0        ret
```

</details>

<details>
  <summary> -O3 optimization with SVE vectorization flags </summary>

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