# 程序的机器级表示

## 寄存器与数据传送

### 寄存器一般功能(部分)	

| **%rax**  Return value - Caller saved | **%rsi**  Argument #2 - Caller saved |
| :-----------------------------------: | :----------------------------------: |
|        **%rbx**  Callee saved         | **%rdi**  Argument #1 - Caller saved |
| **%rcx**  Argument #4 - Caller saved  |        **%rbp**  Callee saved        |
| **%rdx**  Argument #3 - Caller saved  |       **%rsp**  Stack pointer        |
|   **%r8** Argument #5 -Caller saved   |  **%r9** Argument #6 -Caller saved   |
|      **%r10-%r12** Callee saved       |      **%r13-%r15** Callee saved      |

- Caller saved：调用者保存，调用者在调用之前保存临时值在寄存器中，调用其他函数后，寄存器中不会被恢复，会受调用过程的影响
- Callee saved：被调用者保存，被调用者在使用前先保存值，在返回调用者之前重置

### 指令

- 操作码 （ + 操作数 ）

> 操作数:
>
> - 立即数
> - 寄存器
> - 内存引用 通常用括号括起来

#### MOV 指令

- MOV Source operand（源操作数）  Destination operand（目的操作数）
  - 源操作数 可以是立即数、寄存器、或者内存引用
  - 目的操作数 只能是地址（所以可以是寄存器、内存引用），因为 MOV 是将某个操作数的值移动到某个地方
  - X86-64 限制：源操作数和目的操作数不能都是内存地址，所以mem -> mem => mem -> reg -> mem

#### 栈 （pushq popq）

在计算机系统中 栈 的使用

栈底（高地址区） -> 栈顶（低地址区）

- 压栈操作：将栈顶指针减去4个字节，将数据保存进内存
- 出栈操作：将栈顶指针数据读出，将栈顶指针加上4个字节

可以看到出栈时，原先内存中栈顶处的值并没有被清空，只有在重新压栈时，该位置内存的值才会被修改和重新使用

#### 算数逻辑运算指令

**leaq** ：load effective address 加载有效地址

> leaq S D
>
> leaq 7(%rdx, %rdx, 4), %rax : 前面的内存引用并不是将该内存地址的值写入寄存器 %rax，而是将7 + %rdx + 4 * %rdx 这个有效地址写入 %rax 寄存器 （假设 x 保存在 %rdx 中， 则上述指令将 5 * x + 7 写入 %rax，有效地址可以理解成这个地址就是一个值）

关于 lea 与 mov 的区别，其实都是值的转移只不过一个是值传递一个是引用传递，对比以下两个命令：

> lea 4(%rdx) %rax
>
> mov 4(%rdx) %rax

这样的结果 lea 将 %rdx 的值加上了4 然后将这个数的地址写进 %rax，此时 %rax 里是一个引用，也就是一个地址

而 mov 则是将 %rdx 里的值写进了 %rax，此时 %rax 里保存的是值

#### 条件码寄存器

CF: Carry Flag 进位标志

ZF: Zero FLag 零标志

SF: Sign Flag 符号标志 （ <0 置 1）

OF: Overflow Flag 溢出标志

#### 跳转指令与循环

根据某些条件寄存器的值的运算组合跳转到指定位置执行



### 过程（Procedures）

#### 函数调用

>  callq + 地址：
>
> 会将栈顶指针下移，将函数调用后的下一个地址压栈，同时将 程序计数器（%rip）的值变成要调用的函数的首地址

> retq：
>
> 会将栈顶指针的值写入程序计数器（%rip），同时指针上移出栈。
>

>  函数返回值放在 %rax



函数前六个参数（整型，指针类型）所存储的寄存器（超过六个会有栈来保存参数）：

| %rdi | %rsi | %rdx | %rcx | %r8  | %r9  |
| ---- | ---- | ---- | ---- | ---- | ---- |
|      |      |      |      |      |      |


栈帧：一个函数调用从开始到返回，就是栈中，从入栈到出栈分配出的一块内存就是这个函数的一个栈帧，可以由 %rbp 基指针指示栈帧顶部，但一般不常见（对于很多操作系统，对于栈的内存提供最大限额，保证不会无限递归，也就是栈溢出报错）