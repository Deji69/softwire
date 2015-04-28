# Overview #

SoftWire is a run-time x86 assembler. This makes it useful for a compiler's code generator, a JIT-compiler for scripting languages, or for eliminating branches in tight inner loops. In this tutorial we will focus on SoftWire's use for a JIT-compiler's back-end.

Writing a back-end for a compiler that targets x86 processors typically requires good knowledge of machine code. With the features offered by the SoftWire library this is not required. All that needs to be done is to translate the intermediate code to x86 assembly instructions. SoftWire does all the rest, like register allocation, for you. Writing a peephole optimizer can also be done at the same time.

One thing we won't use in this tutorial is SoftWire's built in assembly parser. It allows you to take an Intel-like syntax source file as input. Here we won't take that detour but will instead generate the code directly. As we'll see this has great advantages. Nevertheless, SoftWire can generate a listing file of the assembly code you're generating, which can again be assembled at a later point.

This tutorial is targeted at Windows applications and assumes the Visual C++ compiler. However, SoftWire should be operating-system and compiler independent. The only restriction is the x86 architecture. Good knowledge of x86 assembly is assumed.

# The CodeGenerator Class #

The main class we'll use is `CodeGenerator`. It is defined in the _CodeGenerator.hpp_ file which we have to include. All of SoftWire is in the `SoftWire` namespace so our heading will look like this:
```
#include "CodeGenerator.hpp"
using namespace SoftWire;
```
The `CodeGenerator` class can be constructed without arguments.
```
CodeGenerator x86;
```
Using the class happens in two phases. First the assembly code sequence is produced, and then it is translated to binary format and loaded into memory so it is ready to be called. Don't worry if that's not clear right now, just read on. Let's first focus on how to produce the code.

# Run-Time Intrinsics #

Producing the code is done through the use of run-time intrinsics. These are functions with the same name as x86 instructions. Whenever such a function is called, SoftWire will store this in a buffer which is later used to translate to binary format and load it. Here's a simple example of the use of run-time intrinsics:
```
x86.add(eax, ebx);
```
As you can see this resembles the Intel assembly syntax a lot. All registers are usable just like that. It is important to note that this does not execute the `add` instruction yet. It is not in any way related to inline assembly or compile-time intrinsics. Also, the registers you use here are not the real ones you see in the debug window. We'll get back to this later.

Note the `x86.` at the start of the line. This is of course the `CodeGenerator` instance we constructed above. For one instruction it's not a problem to write this explicitly, but usually we'd like to translate dozens of intermediate code instructions so it becomes annoying. If however we derive our compiler class from `CodeGenerator`, we can omit the `x86`. I will assume this for the rest of the tutorial.

The syntax to use memory operands also resembles Intel syntax a lot. An example:
```
mov(eax, dword_ptr [esp+4*edx]);
```
This syntax is possible thank to the use of operator overloading. Note that `dword_ptr` requires an underscore in the middle. The above example references the stack. Using static memory is just as easy:
```
static char data;
mov(byte_ptr [&data], cl);
```
Note the use of the address operator. This is necessary because else the value of `data` would be used, which is not our intention. Remember this because it is a common error. The address is not taken implicitly because more often you will use pointers.

It is important to know how run-time intrinsics are implemented, in case you would want to modify or extend them, or want to track a bug. They are defined in `CodeGenerator`'s base class, `Assembler`. Because there are so many run-time intrinsics, they are separated from the _Assembler.hpp_ header in _Intrinsics.hpp_, which then gets included in `Assembler`'s class body.

The _Intrinsics.hpp_ file was generated automatically from the x86 instruction set. For every possible combination of arguments the functions are overloaded. They pass the instruction's ID number and the arguments to a private `Assembler` member function which stores the information in a buffer. This method ensures all syntax checking is done by the C++ compiler. The only exception is the scale in a memory reference.

# Executing Your Code #

Now that you know how to create some basic code, let's see how we can load it into memory and call it. The only method we need is `callable`. It requires no arguments, and returns a pointer to the loaded code. The type of this pointer is a function that takes no arguments and returns void. Often the code you produced is the same kind of function, so it can be called directly like this:
```
callable()();
```
Note the double parenthesis. The first is for calling the `callable` method, and the second if for calling the function pointer returned by callable. In case your produced code accepts arguments or returns a value, you have to cast the function pointer to the correct type. For example if the code takes two integers and returns one character:
```
char (*script)(int, int) = (char (*)(int, int))callable();
```
Here I've named generated the function `script`, which can be called at any time as long as the `CodeGenerator` instance is not destroyed. In some cases you might want to keep the function even if the instance is destroyed. This can be accomplished with the `acquire` method. It hands the task of deallocating the function over to you by returning a pointer to it. Beware that this is often not the pointer returned by `callable`.

Another method for controlling memory usage is `finalize`. As its name implies, it deallocates any temporary memory and prevents you from producing extra code. It is advised to call this method after all code has been produced. Only call the method when absolutely needed. It minimizes the footprint of the `CodeGenerator` class, but for the next use it will have to be re-initialized, which requires some time.

Note that the standard calling convention is used (usually `__cdecl`), so the produced assembly code should also use that convention. Other calling conventions can be used by specifying the `__fastcall` or `__stdcall` keyword.

# Jumps and Calls #

Now that we've seen the basics of what run-time intrinsics are, how to produce code with them and call it, let's take a look at their more advanced uses.

The simplest branching instruction is `jmp`. It takes an integer as argument, which is a relative offset indicating how many bytes to jump ahead. This is of course not convenient to work with. Therefore we also have named labels. They can be created with the `label` run-time intrinsic and use a string as argument. The `jmp` can then use this string to reference the label:
```
label("target");
jmp("target");
```
You can place a label anywhere between run-time intrinsics. Since we're still writing C++ you can choose whatever method you prefer to store the label names. They can easily be placed in a symbol table like structure.

Function calls can be made exactly the same way. Place a label before the function and use the label name in the `call` run-time intrinsic. A great feature of using SoftWire's run-time intrinsics is that you can share all data declared in C++, including functions! For example calling the `printf` function can be done this way:
```
#include "stdio.h"
...
call((int)printf);
```
The cast to `int` is required because else `printf`, which is a pointer to the function, would be interpreted as an address where the pointer is stored. This is caused by the limitations of run-time intrinsics and C++ implicit casting. So it's just something you have to remember.

# Complete Example #

With the above introduction you should be able to understand following compilable example:
```
#include "CodeGenerator.hpp"
using namespace SoftWire;
 
#include <stdio.h>
 
class Script : public CodeGenerator
{
public:
    void compile()
    {
        static char *string = "Hello world!";
 
        push((int)string);
        call((int)printf);
        add(esp, 4);
        ret();
    }
};
 
void main()
{
    Script script;
 
    script.compile();
    script.callable()();
}
```
The cast to `int` for the `push` intrinsic is required because else `"Hello world!"` is interpreted as a label name! Again this is a situation where a compromise was made. Easy of use for labels is prioritized so try to avoid this caveat with strings. The easiest way to remember this is that assembly is typeless, so pointers are treated like any other integer.

You can study the execution of this example by placing a breakpoint at the `callable`. Step into it, immediately step out of it, and then go to the disassembly window by pressing Alt+8. Step further into the generated code. The Visual C++ debugger even recognizes the `printf` pointer!

# Conditional Compilation #

The above example doesn't have much practical use. It's just a very laborious way of printing "Hello world!". But it is the basics of a compiler back-end since it is generated at run-time.

As noted before, run-time intrinsics are standard C++. They are just functions that register the corresponding instruction mnemonic and operands. This gives us a lot of freedom in how we manipulate and use them. In this section we will discuss conditional compilation, and in the next we will discuss register allocation.

Conditional compilation is not a real compiler technique, but it is a nice application of run-time intrinsics that shows their real strength. It has a lot in common with self-modifying code, but it is much more convenient and powerful. The idea is simple: based on one or more parameters a run-time intrinsic is executed or not:
```
if(condition)
{
    imul(ebx, edx);
}
```
This is especially useful for optimizing code. A mispredicted jump instruction costs dozens of clock cycles. Even highly predictable compare and jump instructions can take a considerable amount of total execution time. They also put extra stress on instruction caches. Especially in inner loops this can be unacceptable. If however the result of the compare instructions is known some time beforehand, these instructions could be eliminated...

This is nearly impossible with pre-compiled code, but very easy with run-time compiled or assembled code by using conditional compilation. An extra advantage of run-time intrinsics is that it is fast. Parsing and syntax checking is already done by the C++ compiler. So all that needs to be done at run-time is generating the machine code, and SoftWire is quite efficient at this.

An example of this is supporting multiple processors. You might have optimized code for Intel's SSE or AVX extensions. The common method to deal with this is to check the processor type at run-time, and use a conditional statement to decide what code to execute. This is not optimal since the processor type does not change, but having two or more executables isn't economical either. Conditional compilation solves this at the heart of the problem, by selecting exactly those instructions that need to be executed.

# Register Allocation #

The concept of conditional compilation is already one step closer to the creation of a compiler back-end, but we're not done yet. A back-end takes intermediate code as input, which is often in the form of three-address statements. The x86 processor however does not have instructions that match these statements, but most of the time rather works with registers and stack variables. Obviously we would like to use the registers as much as possible since this is much faster than working with the memory all the time.

The hard way to solve this is to keep information about whether a variable is stored in global memory, on the stack or in a register in the symbol table. This method is hard to work with, and would require a lot of complex conditional compilation constructions. What we really need is an abstraction of register allocation.

The flexibility of run-time intrinsics again makes this possible. Imagine we had a function `r32` which took a memory reference as argument and returns a register corresponding with that variable. This would solve most of our problems. A trivial implementation of `r32` would be to use the `mov` run-time intrinsic to load the variable from memory into a certain register. Obviously this doesn't win us anything but it's already the first step towards automatic register allocation because now we only have to work with the memory references, whether they are global or on the stack.

A first optimization of `r32` is not to re-load it if it already stores the variable pointed to by the memory reference. Next is to use all available registers, except esp and ebp because they represent the stack. When we're out of registers, we have to write one back to memory and overwrite it with the new data. This is called register spilling, and can happen fully automatically. A priority system can decide which register is the best candidate for spilling.

This is exactly how SoftWire's register allocator works. No more worrying about what variable is stored in which register, it's all handled automatically and as optimal as possible. Let's look at an example to see how it works in practice:
```
add(r32(esp+0), r32(esp+8));
adc(r32(esp+4), r32(esp+12));
mov(dword_ptr [esp+16], r32(esp+0));
mov(dword_ptr [esp+20], r32(esp+4));
```
This is a typical 64-bit addition with all operands on the stack. Note that nowhere we explicitly used a register. But since the `r32` function itself is implemented using run-time intrinsics the code that is produced might look like this:
```
mov eax, dword ptr [esp+0]
mov ebx, dword ptr [esp+8]
add eax, ebx
mov ecx, dword ptr [esp+4]
mov edx, dword ptr [esp+12]
adc ecx, edx
mov dword ptr [esp+16], eax
mov dword ptr [esp+20], edx
```
If there were not enough unused registers available there would also be some spilling code. You can notice a slight inefficiency in the above code. Since, in this example, the data in ebx and edx is not reused, we could have added directly from memory. This would save us two instructions and kept more register available. For this purpose SoftWire also has a m32 function. If the data is already in a register, it returns the register, else it returns the memory reference. This corresponds closely to the r/m32 symbol in the Intel instruction set reference.

There is also another situation where the use of `r32` is sub-optimal. Some instructions, like mov, do not operate on the destination operand, but completely overwrite its previous value. Using `r32` for the destination operand introduces a useless load operation. For this situation the `x32` function is more optimal. It assigns a register to a memory reference but does not copy its data into this register. So an assignment operation will look like this:
```
mov(x32(var1), m32(var2));
```
Often when translating intermediate code, you will need temporary registers. Using `x32` can be awkward because it requires a memory reference where the register value could be stored should it be spilled. For these temporaries you would also prefer that they never get spilled. For this purpose there is the `t32` function. It works like `x32` but takes an index as argument. This index can only be 0 to 5, since `t32` directly represents a physical register that never gets spilled. How to free it again will be explained in the next section.

Use this function with care. If you use up too many physical registers, and then try to use the other register allocation functions, the register allocator will fail and throw an error. So try to use `t32` as little as possible. An alternative is to have static locations that you can use together with `x32` to use for the temporary variables. This makes the registers spillable and avoid running out of registers. The `t32` function is only for convenience when just a few temporary registers are required which should not be spilled. Free them as soon as possible as explained in the next section.

SoftWire does not only do automatic register allocation for 32-bit general purpose registers, but also for 64-bit MMX and 128-bit SSE registers. For MMX registers you can use the `r64`, `x64`, `m64` and `t64` functions. For SSE registers you can use the `r128`, `x128`, `m128` and `t128` functions. Unlike for general purpose registers where esp and ebp are never used by the register allocator, for MMX and SSE all eight registers are used. So for the `t64` and `t128` functions the index can go from 0 to 7.

# Manual Spilling and Freeing #

Some instructions require specific registers as operands. Generally these kind of instructions should be avoided, but sometimes there is no alternative. When using automatic register allocation, this register is most probably used for another variable. The solution is to force that particular register to be spilled. Also when attempting to use 8-bit or 16-bit registers a similar approach must be followed. For example the `mul` instruction implicitly used eax as first operand, so it must be written back to memory:
```
spill(eax);
```
Even though the priority mechanism produces code with very little spills, it isn't optimal. The problem is that it cannot look ahead. For example, some registers might become available in the following instructions because their associated variable isn't used any more. So these are the best candidates for the next spill. But if this register was used frequently then the priority mechanism attempts to preserve it as long as possible. To give the register allocator a help you can free registers explicitly:
```
free(eax);
free(esp+0);
```
The second line frees the register associated with the variable at esp+0, if any. Note that the +0 makes is a memory reference instead of a register. As soon as you know that a certain variable is not used any more, you can use its memory reference to free its register. The difference with spilling is that a spill writes back the content of the register to memory so the variable can be used further. A free only makes the register available again for allocation.

Also for control transfer, explicit spilling and freeing is required. Let's take for example a conditional block. Inside the block certain registers might get spilled, which might cause variables to switch register. However, this happens conditionally at run-time, so the variable could falsely be expected in another register. To prevent this, explicit spilling (or freeing) of all registers is required. There are `spillAll` and `freeAll` methods provided.

Note that this is not ideal. In code with lots of small basic blocks, it might generate a lot of load operations at the begin and a lot of store operations at the end. Peephole optimization techniques could optimize this but there are other alternatives. This is a situation where `t32` can be very useful since its register can't be spilled. So for short control statements a few variables could be stored in fixed registers. An example is a loop counter. Again keep in mind that they have to be freed manually afterwards.

# Instruction Selection #

You should now be able to write the instruction selection phase yourself using conditional compilation and automatic register allocation. But let's look at some example implementations to get your started and point out some pitfalls. We've already partially seen the assignment intermediate instruction:
```
void emitAssign(const OperandREF &lhs, const OperandREF &rhs)
{
    mov(x32(lhs), m32(rhs));
}
```
The `OperandREF` type is a general reference, so it normally also corresponds with the information you have stored in the symbol table. This is a two argument intermediate code, but most operations are of the form a := b op c, with op being an arithmetic or logical operation. For example a divide operation could be done like this:
```
void emitSignedDivide(const OperandREF &lhs,
                      const OperandREF &op1, const OperandREF &op2)
{
    spill(eax);
    mov(eax, r32(op1));
    spill(edx);
    cdq();
    idiv(m32(op2));
    mov(m32(lhs), eax);
}
```
Note how tricky this code is. The `m32` in the `idiv` instruction can't be replaced by a `r32`. That is because it could allocate `op2` to eax or edx. Remember that `m32` never does an allocation. Just as an exercise, how could we put `op2` in a register? One option would be to call `r32(op2)` before the spills. This increases the chance that `op2` is in a register but does not guarantee it. To do guarantee it there is no other option than to spill a third register...

Cases like these, where specific registers are required, are rare. But be aware of the pitfalls when you're in such a situation. As a rule of thumb, use `m32` whenever possible. This also minimizes the number of allocations and spills. In a situation that demands total control over the registers, just `spillAll()` and use the registers and memory references directly.

Lastly let's look at how to create static data. Although all storage can be allocated in C++, it is mostly more convenient to just store static variables between functions. This is easy thanks to the `db`, `dw` and `dd` run-time intrinsics. To be able to reference the data, a label must be placed:
```
OperandREF emitStaticInt(const char *name)
{
    label(name);
    dd();
    return OperandREF(name);
}
```

# Peephole Optimization #

To a limited extend, SoftWire also allows peephole optimization thanks to conditional compilation. These require a deeper understanding of SoftWire so don't start optimizing prematurely. As a first example, we have a `mov` to the same register. Although rare, this situation will definitely occur. The divide operation from the previous section has a `mov` instruction where the source operand could already be in eax because of the register allocator. Optimizing this case can easily be done by overloading the mov run-time intrinsic:
```
int mov(OperandREG32 reg, OperandR_M32 r_m)
{
    if(r_m.type !=  Operand::REG32 || reg.reg != r_m.reg)
    {
        return Assembler::mov(r1, r2);
    }
}
```
Similar optimizations are arithmetic and logical operations with neutral constants, like a shift by zero bits. Note that when overloading a function, you have to overload all variants. Just take a look at _Intrinsics.hpp_ to know which they are. Also instruction length can be optimized, most notably when using constants:
```
int add(OperandREG32 reg, int imm)
{
    if(imm <= 127 && imm >= -128)
    {
        return Assembler::add(reg, (char)imm);
    }
 
    if(reg.type == Operand::EAX)
    {
        return Assembler::add(eax, imm);
    }
 
    return Assembler::add(reg, imm);
}
```
The first variant saves three bytes, the second saves one. There are thousands of these optimizations possible, and they have to be written manually so they are not integrated in SoftWire. To save yourself from drudgery, just analyze which instructions are used most frequently and focus on those.

Working with the FPU isn't advised since its stack architecture doesn't allow simple register management. So you are forced to use the register stack directly and generate rather suboptimal code. Don't even think of trying to mix it with MMX code. But when 3DNow! or SSE are available you can make floating-point operations very efficient and also use MMX without trouble. Just place an `emms` at the end of your application. So a floating-point multiplication could be done like this:
```
void emitFloatMultiply(const OperandREF &lhs,
                       const OperandREF &op1, const OperandREF &op2)
{
    if(sseSupport)
    {
        movss(x128(lhs), (OperandXMM32&)m128(op1));
        mulss(r128(lhs), (OperandXMM32&)m128(op2));
    }
    else
    {
        spill(lhs);
        spill(op1);
        spill(op2);
 
        fld(dword_ptr [op1]);
        fmul(dword_ptr [op2]);
        fstp(dword_ptr [lhs]);
    }
}
```
When requiring double-precision floating-point operations, SSE 2 can again be a big help, but fall-back paths have to be coded to keep compatibility with older processors.

# Debugging #

Run-time generated code can be hard to debug. Therefore several methods can be used to simplify this task.

First of all, as mentioned before, the 'registers' SoftWire uses in its run-time intrinsics are symbols of their own. This gives some trouble when also using inline assembly. Most debuggers like Visual C++ will not show the value of the registers, but the 'registers' defined by SoftWire. Luckily Visual C++ also has a separate register debugging window, which can be invoked by pressing alt+5. Together with alt+8 you'll be able to press these keys blindly after a while. But you should feel lucky that you can analyze your code with this mighty debugger. Code that is not run-time generated or interpreted is generally much harder to debug.

Using the debugger is not the only way to get a copy of the generated assembly code. SoftWire can also 'echo' the run-time intrinsics, by writing them to a file. The `setEchoFile` method can be used to specify the file to which they are written. The file can be changed between run-time intrinsics so you can write to different echo files. It uses the standard Intel syntax, and it's compatible with SoftWire's parser, so it can also be used for restoring the code.

Adding your own comments to the echo file can be done with the annotate method. It automatically adds a semicolon and a newline so it will never be read by the parser. It is particularly interesting to write intermediate instruction names, so the code is much easier to read. To debug the automatic register allocation, for example to detect when you should have used `r32` instead of `x32`, comments can be placed. For example you could overload `x32` to see if an allocation happened or not:
```
const OperandREG32 &x32(const OperandREF &ref)
{
    if((Operand&)CodeGenerator::m32(ref) !=
       (Operand&)CodeGenerator::x32(ref))
    {
        annotate("%s allocated to %s",
                 ref.string(), CodeGenerator::x32(ref).string());
    }
 
    return CodeGenerator::x32(ref);
}
```
Can you figure out why `m32` is used? Note that it has to be used before `x32`. For `r32` you can use the same code because no re-allocations will be made. The string method returns the Intel syntax string for the operand. Annotate accepts a formatted string and a variable number of arguments to make it easier to write any kind of comment.

# Conclusion #

Although assembly and code generation is never an easy task, I hope I have convinced you that SoftWire can make it much easier. First and foremost, run-time intrinsics are very convenient to use the complete x86 instruction set and forget about the machine code generation. Conditional compilation and automatic register allocation allow you to directly translate intermediate instructions to x86 instructions. This and the other tools SoftWire provides makes it just as easy to write a JIT-compiler than to write an interpreter.

2004 - Nicolas Capens