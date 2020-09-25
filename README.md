# Project 2 Writeup

建议直奔两处 Happy Reference 

## Assembler

### Assmbely File Format

前M条指令，后N条 `.fill` 段

区分local global标签

最多65536条指令

局部变量小写开始，必须有定义；全局大写开始，可以没有。

Branch不准用undefined global。

### Object File Format

| Section Name | Number of lines | Description                                                                                                      |
| ------------ | --------------- | ---------------------------------------------------------------------------------------------------------------- |
| `Header`     | Fixed: 1        | `t d s r`                                                                                                        |
| `Text`       | `t`             | 指令                                                                                                             |
| `Data`       | `d`             | `.fill`                                                                                                          |
| `Symbol`     | `s`             | `<Label> <T,D,U> [offset]` A line offset from the start of the T/D section (0 if the letter was ‘U’). 唯一，无序 |
| `Relocation` | `r`             | `[offset] [opcode] <label>` 无序                                                                                 |

#### Happy Reference

| Defined   | Locality | Segment     | beq    | SymTable | status | offset |
| --------- | -------- | ----------- | ------ | -------- | ------ | ------ |
| Defined   | Global   | text(sw/lw) | offset | yes      | T      | addr   |
| Defined   | Global   | data(.fill) | offset | yes      | D      | addr-t |
| Defined   | Local    | text(sw/lw) | offset | no       |        |        |
| Defined   | Local    | data(.fill) | offset | no       |        |        |
| Undefined | Global   | text(sw/lw) | GG     | yes      | U      | 0      |
| Undefined | Global   | data(.fill) | GG     | yes      | U      | 0      |

| Referred          | RelTable | offset          | opcode |
| ----------------- | -------- | --------------- | ------ |
| Referred by sw/lw | yes      | referred_addr   | sw/lw  |
| Referred by .fill | yes      | reffered_addr-t | .fill  |
| Referred by beq   | no       |                 |        |



#### 其他重要信息

1.  Undefined global symbolic addresses which are temporarily assembled as 0.
2.  ST和RT记的是相对位置

### Error Check

-   [x] Duplicate defined labels (same local or global label within one assembly file)
-   [x] Undefined local symbolic address
-   [x] `beq` using an undefined global symbolic address
-   [x] `offsetFields` that don’t fit in 16 bits
-   [x] Unrecognized opcodes

### Specification

`./assemble program.as program.obj`

## Linker

### Description

按照传入linker的顺序来link。

### Stack Label

由linker加入，紧跟在所有数据段以后

### Error Checking

-   [x] duplicate defined global labels
-   [x] undefined global labels
-   [x] `Stack` label defined by an object file

### Happy Referencing

数据段更新规则(Local Reference指文件内部引用，与标签是否开头大写无关)

判断refer from的起始地址，可以由指令推断；判断refer to的目标地址，应当解码原指令的地址段与t比较。

| Type             | Refer from   | Refer to | update                                                                      |
| ---------------- | ------------ | -------- | --------------------------------------------------------------------------- |
| Local Reference  | .text(lw/sw) | .data    | 加上目标指令地址的绝对变化量（所有.text的长度+之前的.data段长度-所在段的t） |
| Local Reference  | .data(.fill) | .data    | 加上目标指令地址的绝对变化量（所有.text的长度+之前的.data段长度-所在段的t） |
| Local Reference  | .text(lw/sw) | .text    | 加上目标指令地址的绝对变化量（之前的.text段长度）                           |
| Local Reference  | .data(.fill) | .text    | 加上目标指令地址的绝对变化量（之前的.text段长度）                           |
| Global Reference |              |          | 直接用绝对地址覆盖指令的地址段（因为之前记做0）                             |

### Specification

`./linker file_0.obj file_1.obj machine_code.mc`


## Example

```
–––––––– main.as ––––––––

        lw          0       1       five
        lw          0       4       SubAdr
start   jalr        4       7               
        beq         0       1       done
        beq         0       0       start
done    halt
five    .fill       5

–––––––– main.obj ––––––––

6 1 1 2                 (Header)
8454150                 (Text)
8650752
23527424
16842753
16842749
25165824
5                       (Data)
SubAdr U 0              (Symbol Table)
0 lw five               (Relocation Table)
1 lw SubAdr

```

```
–––––––– subone.as ––––––––

subOne  lw          0       2       neg1
        add         1       2       1         
        jalr        7       6
neg1    .fill       -1
SubAdr  .fill       subOne


–––––––– subone.obj ––––––––

3 2 1 2                 (Header)
8519683                 (Text)
655361
25034752
-1                      (Data)
0
SubAdr D 1              (Symbol Table)
0 lw neg1               (Relocation Table)
1 .fill subOne
```

```
–––––––– main.obj ––––––––

6 1 1 2                 (Header)
8454150                 (Text)
8650752
23527424
16842753
16842749
25165824
5                       (Data)
SubAdr U 0              (Symbol Table)
0 lw five               (Relocation Table)
1 lw SubAdr

–––––––– subone.obj ––––––––

3 2 1 2                 (Header)
8519683                 (Text)
655361
25034752
-1                      (Data)
0
SubAdr D 1              (Symbol Table)
0 lw neg1               (Relocation Table)
1 .fill subOne

–––––––– count5.mc ––––––––

8454153             (main.as TEXT)
8650763
23527424
16842753
16842749
25165824
8519690             (subone.as TEXT)
655361
25034752
5                   (main.as DATA)
-1                  (subone.as DATA)
6

```

## Combination

        r0  value 0
        r1  n input to function - ENFORCED
        r2  r input to function - ENFORCED
        r3  return value of function - ENFORCED
        r4  local variable for function
        r5  stack pointer
        r6  temporary value (can hold different values at different times, e.g., +1, -1, function address)
        r7  return address - ENFORCED

