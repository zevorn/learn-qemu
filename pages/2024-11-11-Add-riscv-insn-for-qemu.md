---
layout: center
---

# QEMU RISC-V 添加自定义指令

---

# QEMU Decodetree

Decodetree 则是由 Bastian Koppelmann 于 2017 年在移植 RISC-V QEMU 的时候所提出来的机制。提出该机制主要是因为过往的 instruction decoders (如：ARM) 都是采用一堆 switch-case 来做判断。不仅难阅读，也难以维护。

``` {*}{maxHeight:'250px'}
[Qemu-devel] RFC: QEMU RISC-V modular ISA decoding
From:	Bastian Koppelmann
Subject:	[Qemu-devel] RFC: QEMU RISC-V modular ISA decoding
Date:	Tue, 25 Jul 2017 15:04:18 +0200
User-agent:	Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Thunderbird/52.2.1
Hi QEMU devs, hi risc-v-sw devs,

I'm posting this cross mailing list since I'd like to get feedback from
the both sides.

Right now the RISC-V port for QEMU uses the classic decoding scheme of
one function decoding the first opcode (and prefixes) and then branches
to different functions for decoding the rest (as in target/arm or most
of the other targets). This is all done using switch, case statements.

This is a little bit tricky to extend, especially for third parties. I
don't think it's too bad, but it can definitely be improved and I really
like the way target/s390x does it, but I'm not sure about it's drawbacks.

I see three options to proceed from here:

    1) Completely move to a decoding scheme similar to target/s390x in
       QEMU. On the plus side it make it super easy to add new
       instructions and/or new instruction formats, and reduces decoding
       errors. I don't know the major drawbacks to this approach, maybe
       performance. Does anyone know? Other than that it needs a major
       rewrite of the decoder, which will take some time and thus delay
       the development of RISC-V QEMU upstream. (I think RV32/64I can
       be left as is, since everybody has to implement it anyways)

    2) The compromise: Leave the core as is, i.e. RV32GC, and provide a
       nice interface for any other extension similar to target/s390.
       The big plus here is that no code needs to be changed and only
       the interface needs to be added. We don't add any performance
       overhead (if s390x-style decoding adds any), but this might
       result in nobody using it, since they don't know about the
       interface and they just hack their stuff in. Then it was a waste
       of our time to implement the interface.

    3) The status quo. Just leave it as is.

Any comments?

Cheers,
Bastian

[Qemu-devel] [RFC] RISC-V Decoder generator
From:	Bastian Koppelmann
Subject:	[Qemu-devel] [RFC] RISC-V Decoder generator
Date:	Fri, 20 Oct 2017 15:46:54 +0200
User-agent:	Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Thunderbird/52.4.0
Hi QEMU and RISC-V folks,

I asked you for feedback some while ago regarding a modular RISC-V QEMU
target (see discussion [1]). I tried getting it to work with the good
old C preprocessor and quickly realized that it is too limiting. Instead
I created a data-driven decoder generator written in python (see the
code on github [2]) using YAML as a description language.

I'd like to get some feedback whether this is acceptable to be
upstreamed to QEMU or if you have any suggestions for improvements.
Right now only RV32I instruction are implemented in this scheme.

For this I will roughly sketch how the data format for RISC-V
instructions works and then how the result after processing looks like.
You can find the full tree on github [2] with all the relevant files in
'target-riscv/decode-gen/'.

The RISC-V data in YAML consists of:

1) prefixes
2) instruction formats
3) instruction specification using the data of 1) and 2)

1) prefixes have:
   - name
   - bitmask that identifies instructions belonging to it
   - length that specifies the length in bits for all instructions with
     this prefix

Example prefix description for 16 bit instructions:

16Bit: {
    bitmask: ~0b11,
    len: 16
    }

2) instruction formats have:
    - name
    - list of immediate fields
    - list of 'normal' fields (e.g. register identifier)

    The immediate fields are specified separately in the same file with:
        - name
        - data: in order list of the individual fields + possible sign i

                extension
        - fill: how is the immediate filled from the LSB (for now only
                '0' fills are allowed)

    Example for B-type immediate:
        B: {data: [31s, 7,  30..25, 11..8], fill: '0'}
        Note here that bit 31 is used for sign extension as indicated by
        the 's'.

    The 'normal' fields are similar to immediate fields but only have:
        - name
        - single field description

    Example for 'rs1' field:
        - rs1: 19..15

Example for a full instruction format 'SB':
    - SB: {
      fields: [rs1,rs2],
      immediates: [B]
      }

3) instruction specification have:
    - name
    - instruction format from 2)
    - prefix from 1)
    - keymap of opcode fields and the value they take for this
      instruction
    - func for binary translation

    The opcode fields are specified separately in the same file in the
    same manner as 'normal' opcode fields.
    Example for FUNC3:
        - FUNC3_32: 14..12

Example for the 'BEQ' instruction:
    BEQ: {format: SB, prefix: 32Bit, opcodes: {OP1_32: 99, FUNC3_32: 0},
          func: gen_beq}

    'OP1_32: 99' means that field OP1_32 defined by bits 6 until 0 have
    to have the value 99


The python script then converts these instructions into a tree of
switch-case statements depending on the specified opcodes and generates
at each case for one instruction two function calls:

1) decode_format_* depending on the instruction format used by this
   instruction.
   The decode_format_* functions are also generated by the script and
   take of decoding the 'normal' fields and the immediate fields. This
   data is stored in a data structure 'DisasFormats' which needs to be
   passed to any translation function used by instructions.

2) A call to the translation function defined by 'func' in 3)
   These need to be defined by the developer in translate.c. For example
   for LUI this looks like:

static void gen_lui(CPURISCVState *env, DisasContext *ctx)
{
    if (ctx->fmt.field_rd == 0) {
            return; /* NOP */
    }
    tcg_gen_movi_tl(cpu_gpr[ctx->fmt.field_rd], ctx->fmt.immediate_U);
}

Thanks for any comments,
Bastian

[1] https://lists.gnu.org/archive/html/qemu-devel/2017-07/msg07735.html
[2] https://github.com/bkoppelmann/riscv-qemu/tree/decoder
```

因此 Bastian Koppelmann 就提出了 Decodetree 的机制，开发者只需要通过 Decodetree 的语法定义各个指令的格式，便可交由 Decodetree 来动态生成对应包含 switch case 的 instruction decoder.c。

---

Decodetree 本质是一个 python 脚本，输入定义了体系结构指令格式的文件，输出指令解码器源码文件。

```
+-----------+           +-----------+            +-------------------+
| arch-insn |   input   |  scripts/ |   output   | decode-@BASENAME@ |
|  .decode  +---------->| decode.py +----------->|       .c.in       |
+-----------+           +-----------+            +-------------------+
```

- input: 体系结构定义的指令编码格式文件
- output: 指令解码器的源代码（参与 QEMU 编译）

<br>

#### 语法介绍

Decodetree 的语法共分为：Fields、Argument Sets、Formats、Patterns 四部分。

- Fields，描述指令编码中的寄存器、立即数等字段；
- Argument Sets，描述用来保存从指令中所截取出来各字段的值；
- Formats，描述指令的格式，并生成相应的 decode function；
- Pattern，描述一个指令的 decode 方式。

---

## Decodetree Field

Field 定义如何取出一指令中，各字段 (eg: rd, rs1, rs2, imm) 的值。

```
field_def     := '%' identifier ( unnamed_field )* ( !function=identifier )?
unnamed_field := number ':' ( 's' ) number  eg: %rd 7:5 => insn[11:7]
```

- identifier 可由开发者自定，如：rd、imm… 等
- unamed_field 定义了该字段的所在比特/位域， s 字符来标明在取出该字段后，是否需要做符号扩展
- !function 定义在截取出该字段的值后，所会再调用的 function

<br>

### Field 示例

```c {*}{maxHeight:'160px'}
以 RISC-V 的 U-type 指令为例：
31                              12  11                 7  6    0
+----------------------------------+--------------------+------+
|            imm[31:12]            |         rd         |opcode| U-type
+----------------------------------+--------------------+------+

可以声明为：
%rd       7:5
%imm_u    12:s20                 !function=ex_shift_12

最后会生成如下的代码：
static void decode_insn32_extract_u(DisasContext *ctx, arg_u *a, uint32_t insn)
{
    a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20)); // 是由 insn[31:12] 所取得并做符号扩展，且会再调用 ex_shift_12() 来左移 12 个 bits
    a->rd = extract32(insn, 7, 5); // 由 insn[11:7] 所取得
}
```

---

## Decodetree Argument Sets

Argument Set 定义用来保存从指令中所截取出来各字段的值。

```
args_def    := '&' identifier ( args_elt )+ ( !extern )?
args_elt    := identifier
```

- identifier 可由开发者自定义，如：regs、loadstore… 等
- !extern 则表示是否在其他地方已经由其他的 decoder 定义过。如果有该字段，就不会再次生成对应的 argument set struct

<br>

### Argument Sets 示例

```c
# U-type 指令格式示例
&u    imm rd

# 生成如下代码
typedef struct {
    int imm;
    int rd;
} arg_u;
```

---

## Decodetree Format

Format 定义了指令的格式 (如 RISC-V 中的 R、I、S、B、U、J-type)，并会生成对应的 decode function。

```
fmt_def      := '@' identifier ( fmt_elt )+
fmt_elt      := fixedbit_elt | field_elt | field_ref | args_ref
fixedbit_elt := [01.-]+
field_elt    := identifier ':' 's'? number
field_ref    := '%' identifier | identifier '=' '%' identifier
args_ref     := '&' identifier
```

- identifier 可由开发者自定义，如：opr、opi… 等。

- fmt_elt 则可以采用以下不同的语法：
    - fixedbit_elt 包含一个或多个 `0`、`1`、`.`、`-`，每一个代表指令中的 1 个 bit。
        `.` 代表该 bit 可以用 0 或是 1 来表示。
        `-` 代表该 bit 完全被忽略。
- field_elt 可以用 Field 的语法来声明,Eg：ra:5、rb:5、lit:8

---

- field_ref 有下列两种格式 (以下范例参考上文所定义之 Field)：
    - `'%' identifier`：直接参考一个被定义过的 Field。

        - 如：`%rd`，会生成：
        ```
        a->rd = extract32(insn, 7, 5);
        ```
    
    - `identifier '=' '%' identifier`：直接参考一个被定义过的 Field，但通过第一个 identifier 来重命名其所对应的 argument 名称。此方式可以用来指定不同的 argument 名称来参考至同一个 Field
    
        - 如：`my_rd=%rd`，会生成：
        ```
        a->my_rd = extract32(insn, 7, 5)
        ```
    - args_ref 指定所传入 decode function 的 Argument Set。若没有指定 args_ref 的话，Decodetree 会根据 field_elt 或 field_ref 自动生成一个 Argument Set。此外，一个 Format 最多只能包含一个 args_ref

> PS: 当 fixedbit_elt 或 field_ref 被定义时，该 Format 的所有的 bits 都必须被定义（可通过 `fixedbit_elt` 或 `.` 来定义各个 bits，空格会被忽略）。

---

## Format 示例

```
@opi    ...... ra:5 lit:8    1 ....... rc:5
```

- insn[31:26] 可为 0 或 1
- insn[25:21] 为 ra， insn[20:13] 为 lit
- insn[12] 固定为 1
- insn[11:5] 可为 0 或 1
- insn[4:0] 为 rc

此 Format 会生成以下的 decode function：

```c {*}{maxHeight:'170px'}
// 由于我们没有指定 args_ref，因此 Decodetree 根据了 field_elt 的定义，自动生成了 arg_decode_insn320 这个 Argument Set
typedef struct {
    int lit;
    int ra;
    int rc;
} arg_decode_insn320;
static void decode_insn32_extract_opi(DisasContext *ctx, arg_decode_insn320 *a, uint32_t insn)
{
    a->ra = extract32(insn, 21, 5);
    a->lit = extract32(insn, 13, 8);
    a->rc = extract32(insn, 0, 5);
}
```

---

以 RISC-V I-type 指令为例：

``` {*}{maxHeight:'160px'}
31           20 19    15 14     12  11                 7  6    0
+--------------+--------+----------+--------------------+------+
|   imm[11:0]  |  rs1   |  funct3  |         rd         |opcode| I-type
+--------------+--------+----------+--------------------+------+

# Fields:
%rs1       15:5
%rd        7:5

# immediates:
%imm_i    20:s12

# Argment sets:
&i    imm rs1 rd

@i       ........ ........ ........ ........ &i      imm=%imm_i     %rs1 %rd
```

此范例会生成以下的 decode function：

```
typedef struct {
    int imm;
    int rd;
    int rs1;
} arg_i;

static void decode_insn32extract_i(DisasContext *ctx, arg_i *a, uint32_t insn)
{
    a->imm = sextract32(insn, 20, 12); 
    a->rs1 = extract32(insn, 15, 5);
    a->rd = extract32(insn, 7, 5);
}
```

---

回到先前的 RISC-V U-type 指令，我们可以如同 I-type 指令定义其格式：

```
# Fields:
%rd        7:5

# immediates:
%imm_u    12:s20                 !function=ex_shift_12

# Argument sets:
&u    imm rd

@u       ....................      ..... ....... &u      imm=%imm_u          %rd
```

会生成以下的 decode function：

```
typedef struct {
    int imm;
    int rd;
} arg_u;

static void decode_insn32_extract_u(DisasContext *ctx, arg_u *a, uint32_t insn)
{
    a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
    a->rd = extract32(insn, 7, 5);
}
```

---

## Decodetree Pattern

Pattern 实际定义了一个指令的 decode 方式。Decodetree 会根据 Patterns 的定义，来动态产生出对应的 switch-case decode 判断分支。

```
pat_def      := identifier ( pat_elt )+
pat_elt      := fixedbit_elt | field_elt | field_ref | args_ref | fmt_ref | const_elt
fmt_ref      := '@' identifier
const_elt    := identifier '=' number
```

- identifier 可由开发者自定义，如：addl_r、addli … 等。

- pat_elt 则可以采用以下不同的语法：

    - fixedbit_elt 与在 Format 中 fixedbit_elt 的定义相同。
    - field_elt 与在 Format 中 field_elt 的定义相同。
    - field_ref 与在 Format 中 field_ref 的定义相同。
    - args_ref 与在 Format 中 args_ref 的定义相同。
    - fmt_ref 直接参考一个被定义过的Format。
    - const_elt 可以直接指定某一个 argument 的值。

---

### Pattern 示例

```
addl_i   010000 ..... ..... .... 0000000 ..... @opi
```

定义了 addl_i 这个指令的 Pattern，其中：

- insn[31:26] 为 010000。
- insn[11:5] 为 0000000。
- 参考了 Format 示例中 定义的 @opi Format。
- 由于 Pattern 的所有 bits 都必须**明确的被定义**，因此 @opi 必须包含其余 insn[25:12] 及 insn[4:0] 的格式定义，否则 Decodetree 便会报错。

最后 addl_i 的 decoder 还会调用 trans_addl_i() 这个 translator function

我们结合源代码和手册，再继续分析一下。

---

# 设计 RISC-V 指令

设计一条算数指令 cube，指令编码格式遵循 R-type，语义为：`rd = [rs1] * [rs1] * [rs1]`。

```c
31      25 24  20 19    15 14     12  11                7 6     0
+---------+------+--------+----------+-------------------+-------+
|  func7  | rs2  |  rs1   |  funct3  |         rd        | opcode| R-type
+---------+------+--------+----------+-------------------+-------+
     6                         6                            0x7b
+---------+------+--------+----------+-------------------+-------+
| 000110  |00000 |  rs1   |    110   |         rd        |1111011| cube
+---------+------+--------+----------+-------------------+-------+

```

示例 C 代码：

```c
static int custom_cube(uintptr_t addr)
{
    int cube;
    asm volatile (
       ".insn r 0x7b, 6, 6, %0, %1, x0"
        :"=r"(cube)  // 将结果存储在变量 cube 中
        :"r"(addr)); // 将变量 addr 的值作为输入
    return cube; 
}
```

---

在 QEMU 中添加 cube 的指令译码：

```c
// target/riscv/insn32.decode
@r_cube  ....... ..... .....    ... ..... ....... %rs1 %rd
cube     0000110 00000 .....    110 ..... 1111011 @r_cube
```

添加 cube 的指令模拟逻辑（采用 helper 实现）：

```c
// target/riscv/helper.h
DEF_HELPER_3(cube, void, env, tl, tl)

// target/riscv/op_helper.c
void helper_cube(CPURISCVState *env, target_ulong rd, target_ulong rs1)
{
    MemOpIdx oi = make_memop_idx(MO_TEUQ, 0);
    target_ulong val = cpu_ldq_mmu(env, env->gpr[rs1], oi, GETPC());
    env->gpr[rd] = val * val * val;
}

// target/riscv/insn_trans/trans_rvi.c.inc
static bool trans_cube(DisasContext *ctx, arg_cube *a)
{
    gen_helper_cube(tcg_env, tcg_constant_tl(a->rd), tcg_constant_tl(a->rs1));
    return true;
}

```

---

### 编译测试

编写一个简单的示例程序：

```c
int main(void) {
    int a = 3;
    int ret = 0;
    ret = custom_cube((uintptr_t)&a);
    if (ret == a * a * a) {
        printf("ok!\n");
    } else {
        printf("err! ret=%d\n", ret);
    }
    return 0;
}
```

编译运行测试：

```bash
$ riscv64-linux-musl-gcc main.c -o cube_demo --static
$ qemu-rsicv64 cube_demo
$ ok!
```
