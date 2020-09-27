# Project 2 Writeup

> ***建议直奔两处 Happy Reference***

欢迎转发给有需要的人，觉得写的不清晰的欢迎提issue。

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

## Spoiler

你们可能会喜欢的自问自答环节（别看了，没意思的）

- Q：Symbol Table有啥用？
  - A：用来告诉其他obj，来找全局标签的表（所以我个人觉得Undefined Global Label放在这里意义不大）
- Q：Relocation Table有啥用？
  - A：用来告诉自己obj，用来解决refer地址偏移的表
- Q：什么东西会出现在Symbol Table里？
  - A：所有Defined or Undefined Global Label
- Q：什么东西会出现在Relocation Table里？
  - A：所有在lw/sw/.fill右侧（被refer）的label
- Q：那beq有哪些特殊处理
  - A：不接受任何Undefined Local or Global Label，其余和p1相同。因为本身就计算转化成了在.text段里的相对位置，所以不需要考虑relocation了（不过如果给beq喂了一个来自于.data段的信息，程序还是会崩的吧）
- Q：TDU怎么区分，有啥用。
  - A：U用来指示**Undefined Label**。TD用来指示**Defined Label**所在的段，如果对应opcode是.fill那就是.data段，否则是.text段。这些信息在relocation重新计算偏移时会用到。
- Q：Relocation时如何计算偏移
  - A：要分成Local标签和Global标签来考虑
    - Local: 只需要考虑refer的目标在.data/.text段里的情况，建议画个图（多文件情况下.text段和.data段的信息是怎么堆叠的，要意识到local标签本身就是个offset）观察一下目标地址还进行了多少偏移。
    - Global: 直接用这个标签的绝对地址（可以算出来）来覆盖指令里的地址。
- Q：你为什么要写这玩意儿。
  - A：请大家有问题不要再打扰yxx了，yxx都没空闲时间卷370281376和research了（有空能不能多来找ktt玩）。
