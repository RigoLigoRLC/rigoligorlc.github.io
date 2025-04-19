
# Clang？

## 2025/04/19

首先得定义一个clang的target triplet。`<arch><sub>-<vendor>-<sys>-<abi>`，我设想是loongarch64-unknown-windows-gnu。

参见：
- clang/lib/Basic/Targets/OSTargets.cpp 其中有关于对不同Target如何添加编译器定义宏（好像不太需要关注这个）
- clang/lib/Basic/Targets.cpp 其中有关于对不同llvm::Triple如何获取TargetInfo结构的内容。
- MinGWARM64TargetInfo：可以参考ARM64下MinGW的Windows ABI定义。

  - 文件：clang\lib\Basic\Targets\AArch64.h，clang\lib\Basic\Targets\AArch64.cpp
  - `MinGWARM64TargetInfo`和`MicrosoftARM64TargetInfo`只差了一个`TheCXXABI.set`的参数，MinGW用的是`TargetCXXABI::GenericAArch64`，Microsoft的用的则是`TargetCXXABI::Microsoft`。这里有什么区别呢？（事后发现傻了，这不就是指定使用哪种C++ ABI嘛）
  - clang\include\clang\Basic\TargetCXXABI.def 这里面有：`ITANIUM_CXXABI(GenericAArch64, "aarch64")`的注释，让你去看 https://github.com/ARM-software/abi-aa/blob/main/cppabi64/cppabi64.rst ，然后说其实和Itanium ABI差不多。
  - LoongArch64的Linux Target没有这个TheCXXABI，可能Linux默认就是Itanium。也合理。除非龙平台会出MSVC，还是先不要管MS ABI了。不过真出了的话，可能是个坑。

其实编译到各种平台都有点大同小异的感觉。不过，Linux和Windows还是不太一样的，比如Linux有GOT PLT，而Windows不使用GOT PLT机制（但是用的类似的办法）；Linux的TLS机制和Win不太一样（不过，线程环境指针应该都是在GS寄存器里，如果是x86的话）；其他的，应该是链接器的活。还有SEH。栈结构、栈回溯信息啥的。

主要想看的就是为同一种CPU编译时，Linux target和Windows target的编译器它们走的代码路径有何不同。不然都不知道怎么下手改。

产生汇编码应该是在SSA形式之后，把SSA降低到机器操作的环节。这部分代码应该在哪里呢？

问问LLM，ChatGPT说这其实是LLVM干的。也对，Clang就是个语言前端，产生的都是LLVM IR。LLVM后端把这些SSA变成机器码。看看ChatGPT的原话：

> 1. Clang Frontend:
>
>    - Clang parses C/C++/ObjC source code and lowers it into LLVM IR (in SSA form).
>    - This part ends with an LLVM Module in memory, filled with functions in SSA form.
>
> 2. LLVM Middle-End / Optimizer:
>
>    - LLVM passes perform various IR-level optimizations, all still in SSA form.
>
>3. LLVM Back End (the part you're asking about):
>
>    - The IR in SSA is lowered to machine code (non-SSA).
>    - This involves instruction selection, register allocation, and code emission.

然后LLVM和GCC其实都有Machine description这样的东西，LLVM的在llvm/lib/Target/<目标机器>里面，后缀叫.td（TableGen Description）。这玩意在编译LLVM的时候会用Tblgen生成.inc文件，里面是C++代码，应该是会编译进LLVM里。

x86这里面可以看到x86的Calling convention，这个也很重要，因为x86和ARM是Win calling convention的唯二重大用户，是重要参考。

`llvm\lib\Target\X86\X86CallingConv.td`里面先看看对Win做了什么特判记个笔记：

- `CCIfIsVarArgOnWin` 检测是否是在Win 32位上的vararg函数调用。vararg调用约定确实是个坑。
- `RC_X86_32_RegCallv4_Win` Windows 32位的RegCall调用约定（ICC特有的好像是）。继承自`RC_X86_RegCall`，其实后者就是一个只填了FP register的模板，不同的调用约定继承自它，然后填充上各自用到哪些寄存器。
- `RC_X86_32_RegCallv4_Win` 同上，RegCallv4的版本。
- `RC_X86_64_RegCall_Win` 同上，RegCall x86_64版本。
- `RC_X86_64_RegCallv4_Win` 不想写介绍了
- 这个不是对Win特判，但是这个`multiclass X86_RegCall_base<RC_X86_RegCall RC>`应该说才是实现了RegCall的地方。前面充其量是描述了能用哪些寄存器，这个地方写了寄存器应该怎么分配。
- `CCIfNotSubtarget<"isTargetWin64()", CCIfType<[f80], CCAssignToReg<[FP0, FP1]>>>` 这个是RetCC_X86Common，也就是描述x86统一的返回约定的部分给Win的特判，仅在非Win64条件下，才把long double使用x87寄存器返回。说起来x87又是一座史山啊……
- `RetCC_X86_Win64_C` x86 Win64下GCC把浮点数从RAX里返回（什么鬼）的特判。注意到这里没有继承，而是用`CCDelegateTo<调用约定def名称>`来在没匹配到此处的规则时使用公用的规则。
- `RetCC_X86_64_Vectorcall` VectorCall，应该是给含SIMD的函数调用准备的。这我都没听过

后面的东西有点大同小异了，我觉得没必要列出来，看得到怎么做的特判就差不多了。然后再列出一些我觉得重要的地方。

- `def CC_X86_Win32_CFGuard_Check : CallingConv` Windows Control Flow Guard的CFGuard_Check专用的一个调用约定，目标函数地址要占用一个RCX。这里给它分配进了RCX。感觉是一个可能需要注意的部分。
- `def CC_X86_64 : CallingConv` 这里是x86_64调用约定的起始点。也有特判。
  
  - `CCIfCC<"CallingConv::Win64", CCDelegateTo<CC_X86_Win64_C>>,` 里面用的这种的东西去检测使用的什么调用约定，然后把什么调用约定匹配去
  - 有意思的是MinGW64专门有一个判定：
    
    ```
    // Mingw64 and native Win64 use Win64 CC
    CCIfSubtarget<"isTargetWin64()", CCDelegateTo<CC_X86_Win64_C>>,
    ```

    不知道有什么具体影响。

再来看看ARM64的Windows调用约定吧。很相似地被放在了类似路径的文件里：`llvm\lib\Target\AArch64\AArch64CallingConvention.td`

- `def CC_AArch64_AAPCS : CallingConv` 里面写Darwin和Windows使用X18作为platform register（其实龙芯保留的r21应该就是用于这个用途的），不能用于“nest”参数。搜了一下没搞懂这是什么。ChatGPT说这是nested function隐含的参数，我暂且蒙古，不过如果在龙上不管这个nest的分配应该无所谓。
- `def CC_AArch64_Win64PCS : CallingConv<AArch64_Common>;` Win64 Procedure Call Standard，和AArch64是完全一样的。新平台就是好，没有那么多弯弯绕绕的东西。
- `CC_AArch64_Win64_VarArg` Win64下vararg函数，浮点会塞进整数寄存器传参。有点恶心……
- `RetCC_AArch64_Arm64EC_Thunk` Arm64EC ABI，为了兼容一些x86行为产生的，感觉属于没屎硬吃行为。比如_m64从RAX返回这种操作需要这一层来兼容……我对Arm64EC没什么兴趣，跳过。这个也和龙没有什么关系。
- `CC_AArch64_Win64_CFGuard_Check` ARM64 Control Flow Guard，需要使用X15寄存器传递目标函数地址。那么看来我们也需要给龙平台定义一个Control Flow Guard调用ABI了。
  
  关于Control Flow Guard到底是什么可以看一下微软的文档：https://learn.microsoft.com/en-us/windows/win32/secbp/control-flow-guard#how-does-cfg-really-work 我的TLDR版本是：当程序里出现需要调用某函数指针的时候，编译器的CFG支持功能会检测这个回调所有可能的目标地址，并且存起来留到Runtime；运行时，所有对这个函数指针的调用都会调用CFGuard_Check检测我要跳转到的目的地址是不是这些可能地址中的一个，如果不对劲就立马终止程序。不过微软说内核也有支持，但我不知道内核需要做什么。
  
  搜索了一下找到了系统如何对内核态程序（比如驱动程序）应用CFG：https://community.osr.com/t/control-flow-guard/54703 TLDR：这个机制需要HVCI（Hypervisor代码完整性保护），也就是说如果我只是搞ReactOS的话，这个都算后话了（？）
  
  TODO: 文档里说的CFG的Kernel runtime support究竟是指需要Kernel support才能支持用户态程序使用CFG，还是仅指Kernel driver之类的东西需要使用的runtime support？

最后，Callee saved register （CSR，不是Control and Status Register）有另一堆东西定义（x86也有但我忘了写），对于Win ARM64来说还有一个细节：

  ```
  // Win64 has unwinding codes for an (FP,LR) pair, save_fplr and save_fplr_x.
  // We put FP before LR, so that frame lowering logic generates (FP,LR) pairs,
  // and not (LR,FP) pairs.
  def CSR_Win_AArch64_AAPCS : CalleeSavedRegs<(add X19, X20, X21, X22, X23, X24,
                                              X25, X26, X27, X28, FP, LR,
                                              D8, D9, D10, D11,
                                              D12, D13, D14, D15)>;
  ```

看起来栈回溯方案是如何实现的也需要考虑到。

然后就是Control Flow Guard在被调用时保存哪些寄存器，

```
// The Control Flow Guard check call uses a custom calling convention that also
// preserves X0-X8 and Q0-Q7.
def CSR_Win_AArch64_CFGuard_Check : CalleeSavedRegs<(add CSR_Win_AArch64_AAPCS,
                                               (sequence "X%u", 0, 8),
                                               (sequence "Q%u", 0, 7))>;
```

看起来就是X0-X8（整数）Q0-Q7（SIMD），显然它们都是参数寄存器。设计龙的CFG时应该也是这样用。

再看看龙的Calling convention在哪里定义的吧。理论上，需要添加的Calling convention应该和AArch64_Win64PCS一样直接复用就好，关于Vararg也不用太担心，毕竟根本没有标准，自己定义就行。文件一样的在`llvm\lib\Target\LoongArch\LoongArchCallingConv.td`：

一看就乐了，非常简洁只有一点点东西。它只定义了软浮点、单精度浮点、双精度浮点三种情况下的Callee-saved register，其实就是所有在ELF psABI里定义的静态寄存器。至于具体的Calling convention，后面再看看。

需要了解一下LoongArch上Linux上的栈回溯方式进行参考，不然SEH什么的没法定义。ARM64的Windows unwinding数据输出参考：`llvm\lib\Target\AArch64\MCTargetDesc\AArch64WinCOFFStreamer.cpp`，`llvm\lib\MC\MCWin64EH.cpp`

# Misc chat log

> 📍 Where SSA Is Lowered in LLVM
> 1. Instruction Selection (IR → MachineInstrs)
> 
> This is where SSA form begins to be "lowered":
> 
>     File: llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp
> 
>     File: llvm/lib/CodeGen/GlobalISel/IRTranslator.cpp (if using GlobalISel instead of SelectionDAG)
> 
> These translate LLVM IR (SSA) into MachineInstr (not in SSA, but in a virtual register format).
> 2. Register Allocation (Virtual → Physical registers)
> 
> This finalizes the break from SSA, as registers are now assigned:
> 
>     File: llvm/lib/CodeGen/RegAlloc/
> 
>     Look at files like:
> 
>         RegAllocGreedy.cpp
> 
>         RegAllocBase.cpp
> 
>     Also involves:
> 
>         llvm/lib/CodeGen/MachineRegisterInfo.cpp
> 
> 3. PHI Node Elimination / SSA Destruction
> 
> LLVM destroys SSA form by inserting copies where needed:
> 
>     File: llvm/lib/CodeGen/PhiElimination.cpp
> 
> 💡 Tip: Tracing from a Function
> 
> To trace from your own IR:
> 
>     Use llc (LLVM static compiler) with -debug and -print-after-all flags.
> 
>     Or run opt and llc with -view-dag-combine1-dags to see SelectionDAGs.
> 
>     Look at MachineFunction, MachineBasicBlock, and MachineInstr dumps.
> 
> Summary
> |Stage | LLVM Subsystem | Relevant File(s)|
> |-|-|-|
> |IR (SSA) → MachineInstr | SelectionDAG / GlobalISel | SelectionDAGISel.cpp, IRTranslator.cpp|
> |PHI Lowering / SSA Destruction | PhiElimination | PhiElimination.cpp|
> |Register Allocation | RegAlloc | RegAllocGreedy.cpp, RegAllocBase.cpp|