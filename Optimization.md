# Introduction #

For script compilers and Just-In-Time-compilers (JIT), compilation time is critical, but we also want code that is not entirely unoptimized. When code optimization is performed, most algorithms however take a significant amount of processing time and extra memory. In this article I will present algorithms that require no new passes and take virtually no extra memory. The focus is on the implementation of the back-end of the compiler. It is intended to be of practical nature, targeted at developers who wish to apply the presented ideas quickly in their own projects. Basic knowledge of assembly language and compiler technology is assumed.

The compiler back-end I'll use is SoftWire. It was developed as a run-time x86 code generator to construct processing pipelines for 3D applications without specialized hardware. It is also used to compile shaders, short stream processing programs for programmable lighting effects. Although most of the presented algorithms were originally created for the purpose of efficient 3D rendering, here we'll focus on SoftWire's use in a generic x86 JIT-compiler. It should be noted that they are even more widely applicable than x86 code, and can be an inspiration for anyone interested in optimization techniques.

SoftWire's low-level interface consists of functions with the same name as assembly mnemonics, which put the corresponding code in a buffer. I call these functions run-time intrinsics. For example, `add(eax, ebx);` will generate the binary code for add eax, ebx and place it in an internal temporary buffer. This way we can construct functions one instruction at a time. When finished, we can ask SoftWire for a callable function. For this it places the instructions in a final buffer and resolves label addresses. It returns a pointer to this newly created function, which can directly be called. All this is exposed via the `Assembler` class. It is highly recommended to read the SoftWire introductory [Tutorial](Tutorial.md) first to understand this article better.

This way of creating functions at run-time is very powerful, but for a compiler back-end we need an abstraction of the underlying architecture. We have to be able to compile intermediate code in a convenient, flexible way, while still keeping high performance.

# Linear Register Allocation #

The first step to achieve flexibility is register allocation. It assigns variables to registers, so we can keep the most actively used data right in the processor core instead of in memory. This allows us to abstract the register set of the x86 processor, so we can work with generic variables everywhere. In this article I will only discuss 32-bit integer variables as to not make things more complicated. Below is an illustration of what register allocation tries to achieve. Intermediate instructions use only variables, while the assembly code has to use registers:
```
b = a;
c += b;
```
Will translate into something like:
```
mov eax, [a]   // Allocate a to eax
mov ebx, [b]   // Allocate b to ebx
mov ebx, eax
mov ecx, [c]   // Allocate c to ecx
add ecx ebx
```
Here we can clearly see the primary task of the register allocator. It has to assign registers to variables, load the variables from memory into the registers, and keep using the registers in further instructions. Another task is to write registers back to memory. This is required when we attempt to allocate a new variable, but there are no more free registers. Just like pouring more water into a full glass, this is called spilling. An allocated variable gets written back to memory and a new variable takes its place in the same register. So the problem to solve is when to allocate, what register to use, and when to spill.

We have one big restriction to perform good register allocation. SoftWire has no look-ahead. Since binary code is created when run-time intrinsics are called, there is no way to look at what instructions or variables will be used next. It does not have access to the list of intermediate code instructions (SoftWire does not perform instruction selection). Hence it is not possible to directly determine when a variable will not be used any more. So we can't know in advance when a register is free, to avoid spilling. The limitation of having no look-ahead is the main issue this article addresses. Eventually it is a big advantage for reduced compile time and memory requirement.

SoftWire's register allocator is inspired by the [linear-scan register allocation algorithm](http://www.cs.ucla.edu/~palsberg/course/cs132/linearscan.pdf). It works quite straight-forward and just as illustrated above: it keeps assigning any newly used variable to a new register, until we're out of registers. When this happens, we spill the register, which then becomes available for the new variable. One crucial step is to determine which register to spill, as we can choose practically any of the registers. SoftWire uses the least recently used. It's most likely that the variable that is allocated to this register is actually not used again. While this doesn't work entirely optimally, it does yield very good results, and I'll show you in the rest of the article how the allocation and spilling process can be improved.

The actual implementation is simple. SoftWire keeps a small list of allocation states, one per register. Each of these contains a reference to a variable. A reference can be anything to uniquely identify a variable's memory location. In practice, this is often a simple pointer. But it can also be a stack location, which I'll explain in the next section. Allocating a register is done by filling in the variable's reference in the corresponding position in the allocation list. Then a `mov` run-time intrinsic is called, to generate the code that loads the variable from memory into the register. Spilling is just as simple, we call the run-time intrinsic that writes the register to its allocated variable's reference. Freeing can be done by nullifying the reference. This is also how free registers are identified internally.

SoftWire conveniently wraps all this functionality into one function, `r32()`. It is defined in the `RegisterAllocator` class, derived from Assembler. Similar functions are available for other sizes and types than 32-bit integers. As argument, it takes the reference to a variable, the output is a register. So we can use it to write code this way:
```
static int a;
static int b;
 
mov(r32(&b), r32(&a));
add(r32(&c), r32(&b));
```
The right `r32()` gets called first, then the left one, and then the run-time intrinsic. When executed, it will produce the assembly code given above. So this provides the first abstraction we required to generate assembly code from intermediate instructions. Because we also would like to make it somewhat processor independent, and because there's only a limited set of intermediate code instructions, we can abstract it further like this:
```
void emitAssign(const int &a, const int &b)
{
    mov(r32(&a), r32(&b));
}
 
void emitUnaryAddition(const int &a, const int &b)
{
    add(r32(&a), r32(&b));
}
```
There's still one thing that highly influences register allocation: jumps. In C++, every if, else, for, etc. creates jumps. Now imagine that inside an if-block we access a variable for the first time. So we would call `r32()`, and this allocates a new register. Using SoftWire, the code is generated linearly. It is not aware of the if-block. We just keep calling run-time intrinsics and it adds the corresponding binary code to the buffer. The jump instruction is treated no different. However, when we execute the code, we might actually jump over the if-block. If a new variable is allocated there, then we skipped that and there's a big chance that the register doesn't contain the value we expected after the jump. At function calls we also expect (some) register content to be preserved.

The brute-force solution is to write back all registers to memory before the jump, and before the target address of the jump. This way, all registers are free and they are re-allocated both inside the block and after it. We've seen this operation before. It's exactly the same as register spilling, except that we don't allocate a new variable in their place. We can call SoftWire's  spill() function for every register to do this. There's also a `spillAll()` function to make this more convenient.

So that's it! With these tools we can create a compiler back-end in a very convenient way. Make sure you've read the older SoftWire [tutorial](Tutorial.md) to see a few more examples on how to get started. There's still a lot to be done though. The generated code is far from optimal and I promised to present some new algorithms to optimize the code and keep the linear nature of SoftWire.

# Local Variables #

Note how I used static data in the previous examples. However, local variables are usually stored on the stack, so that memory space can be reused. To enable this, SoftWire uses a more generic reference; `OperandREF`. This is an internally used structure that can hold the pointer to an integer, but also the x86 specific base register, index register and offset. This needs a bit of illustration. Memory references can be written in this general format:
```
dword ptr [baseRegister + {1, 2, 4, 8} * indexRegister + offset]
```
Although this is x86 specific, most other architectures have memory addressing modes with a base register and offset, to access the stack. The stack pointer is stored in the esp register, and ebp is commonly used to construct a local frame. So, in SoftWire, esp and ebp are excluded from the register allocation process, and can be used freely by higher layers, such as the `CodeGenerator` class. It is inherited from the `RegisterAllocator` class, so it also exposes the run-time intrinsics. The added functionality is that it provides the mechanism for creating and destroying local variables on the stack. It has child-classes that automatically behave like local variables. This is possible thanks to C++ constructors, destructors, and cast operators. One of these classes is Int, with capital. When constructed, it looks for a location on the stack to be stored. This is a memory reference in the form of `[ebp + offset]`. When used in a run-time intrinsic, it will automatically cast itself to a register. To do this it simply calls the `r32()` function and uses its stack location as argument. This allows for example to implement binary intermediate instructions very conveniently:
```
Int emitBinaryAddition(const Int &a, const Int &b)
{
    Int t;      // Temporary, so we don't modify a
    mov(t, a);
    add(t, b);
    return t;
}
```
Two more C++ operators had to be implemented to make this work. First of a copy constructor was needed. The implementation is very simple, using a `mov` run-time intrinsic just like `emitAssign()`. The other operator is the destructor. Here, the local variable can be removed from the stack, and the register freed. Removing it from the stack is done by comparing all the stack pointers that have been used as reference, and using the offset of the first unused stack location as the new stack top. Actually freeing the register can be done by nullifying its reference. This is implemented in the free() function.

Setting up the stack is done by the `CodeGenerator::prologue()` function. As argument it takes the number of arguments that are used by the generated function. These arugments can then be accessed with the argument() function, which takes an integer index and returns the memory location of the argument. SoftWire will automatically keep track of how many variables require stack space. Ending the function can be done conveniently with the epilogue() function. Summing this up, the general framework for generating a function looks like this:
```
void(*)(int,int) emitFunctionX()   // Returns a function pointer
{
    prologue(2);   // Generated function takes two arguments

    Int a;
    Int b;

    mov(a, argument(0));   // They are memory operands
    mov(b, argument(1));

    /* Call other funcions to generate the desired function */

    epilogue();

    return (void (*)(int , int))callable();
}
```
You might have noticed that using these classes for local variables can be made even more powerful, by implementing more C++ operators. We can have an `operator+`, which gets the implementation of `emitBinaryAddition()`. This way we can generate code for all arithmetic intermediate code instructions, by simply performing the operation on the variables. Also, generating code for whole C++ expressions is no problem:
```
struct Vector
{
    Int x;
    Int y;
    Int z;
};
 
Int emitDotProduct(const Vector &v1, const Vector &v2)
{
    return v1.x * v2.x + v1.y * v2.y + v1.z * v2.z;
}
```
Don't confuse this to be a function which computes a dot product, all it does is generate the code to compute a dot product, and keep that internally in SoftWire. Only after you retrieve a pointer to the code it can be used as a function to compute a dot product.

# Intermediate Code Free #

As mentioned before, SoftWire's biggest restriction is that it has no look-ahead, so it can't detect that variables will no longer be used in further code. Freeing registers only happens when they are spilled and reallocated, and when the `free()` function is explicitly called for local variables. However, this usually happens later than the actual point at which the variable is no longer used. This can be too late to avoid unnecessary spilling.

The compiler front-end doesn't have that restriction. It can do complex analysis on the code, and determine the last use of variables. Hence, it can insert free() commands into the intermediate code. For JIT-compilers, compiling the source code to intermediate byte code is not as time-critical as code generation and optimization. Note that byte code doesn't use generic variables, but virtual registers. Either way, the same register allocation algorithms apply since the x86 architecture has only six general purpose registers, while there can be many more virtual registers in the byte code.

So this simple technique gives SoftWire the look-ahead capabilities from a higher level. It shows that its restriction isn't really a restriction, and the end result will be a faster code generator that uses no extra memory. The next three sections will focus on optimizations that can also be done close to optimal without any need for look-ahead, but also without help of the compiler front-end.

# Load Elimination #

Let's go back to generating binary code for intermediate instructions. Take the most trivial example, an assignment of a variable `a` to a variable `b`:
```
b = a;
```
Recall that with run-time intrinsics and the allocation functions, this can be implemented as:
```
Int &CodeGenerator::Int::operator=(const Int &x)
{
    mov(*this, x);
    return *this;
}
```
The assembly code that gets generated is:
```
mov eax, [a]   // Allocate a to eax
mov ebx, [b]   // Allocate b to ebx
mov ebx, eax   // Assign
```
Obviously, the second instruction is redundant. The value of ebx gets overwritten, so we didn't have to load it with b. Because again, SoftWire has no look-ahead, this could not be prevented. There is no convenient way to know in advance that ebx will get overwritten. But the story doesn't end here. While SoftWire doesn't have look-ahead, looking 'back' is less of a problem. So the idea is that when a register gets overwritten, we erase the load instruction. This is easy to implement, just keep a pointer to the load instruction for every register. This gets stored together with the variable reference in the register's allocation state. So when the load can be eliminated, SoftWire uses this pointer to notify that this instruction can be erased from the internal buffer.

But wait, we can't just eliminate all load instructions. Assume that the next intermediate operation assigns b to a. Then the load instruction of a would be erased, while it's actually required. To prevent this, every time a register is read, we set the pointer to the load instruction of that register to null. Note that for arithmetic x86 instructions, the first and second operand is read, while for `mov` only the source operand is read. So we make an exception for `mov` only, by overloading this run-time intrinsic function in the `RegisterAllocator` class. The end result is that every time a register gets allocated and overwritten before it is used in another instruction, the load instruction gets eliminated. The same elimination is possible when the register is freed before used.

# Copy Propagation #

The approach of looking back is really powerful. I will show you that it does not only allow to identify and eliminate `mov` instructions that load a value from memory into a register, but also `mov` instructions that copy from one register to another, and `mov` instructions that stores a register back in memory. Eliminating copy instructions is called copy propagation, while eliminating store instructions is called spill elimination. The latter will be explained in the next section, and I'll also make it clear why it isn't called store elimination.

Copy propagation is a classical optimization in compiler technology. When a register is copied to another register, and the first register is never used again, then this copy was redundant. In this case we could have done all the calculations with the first register. This isn't trivial to implement in SoftWire, because we can't/won't keep a pointer to every instruction that uses a certain register, nor is it even feasible to adjust all the registers in the binary format. The only look-back we have is to eliminate instructions.

The solution is quite simple in nature. Similar to load elimination, we keep a pointer to the copy instruction per register. Then, we 'blindly' assume that the first variable won't be used again, so we keep working with the first register. This can be achieved by swapping the references of both registers. So when the second variable is used, `r32()` will return the first register. When the first variable is freed before being used again, we know for certain that the copy instruction can be eliminated. We then set the pointer back to null. Here's an illustration of how it works:
```
b = a;
c += b;
free(a);
```
Would get translated into:
```
mov(eax, a);     // Allocate a to eax
mov(ebx, b);     // Allocate b to ebx, load eliminated!
mov(ebx, eax);   // Copy propagation, assign b to eax, a to ebx
mov(ecx, c);     // Allocate c to ecx
add(ecx, eax);
free(ebx);       // Eliminate ebx's copy instruction
```
Note that if a was still used, then ebx would be returned, and its copy instruction pointer nullified, to prevent elimination. What would have effectively happened is that eax and ebx switched place. The copy instruction makes them perfectly equivalent, so at that point it doesn't really matter which register is associated to which variable. This nice property enabled copy propagation with simple look-back. Other compiler back-ends use complicated graphs to perform this optimization!

# Spill Elimination #

Copy propagation works like a charm, but there's still a significant problem. In the above example, ebx was never used, but a was associated to it anyway. Imagine that we didn't have any free registers before generating code for these intermediate instructions. Then we would have spilled eax, ebx and ecx. We can't avoid that eax and ecx have to be spilled, but spilling ebx was redundant, since it never got used. The cost of this isn't only the spill instruction, but also the load instruction that follows, to get useful data into ebx again. Ideally, we want to avoid the spill, and the load.

This can be achieved by keeping the previous reference of a variable for a register that has been spilled, and a pointer to the spill instruction. The idea is that when we detect a redundant spill, we eliminate the instruction, and restore the previous reference, as if the register was never associated to another variable.

The implementation is trickier than that. Recall that for branches we have to spill the registers ourselves, to store their value in memory. It is not uncommon, that after this some registers are freed. After all, a spill makes the register available for allocation again, so it shouldn't make any difference when it is also freed afterwards. Unfortunately, with the current approach for spill elimination it does. At the free, the register is considered to no longer be used. So the spill is eliminated. But when this register was copy propagated, this means that also the previously allocated variable, which is the actual contained value, would not be written back to memory. For the branch, this is absolutely required.

The actual problem is that there are two types of spilling. One, which really deserves the name 'spill', is to write registers back to memory, to allocate another variable in its place. The other type of spill is to store the value in memory, but leave the register free. It can take several instructions before this register is then back allocated. It is this latter type of spill that we use at branches, so it should be available as a public function. Spill elimination only targets the former type of spill. And it should happen only when we try to allocate a variable and find no more free registers. This variable can then be copy propagated, and when the associated register is unused, the spill can be eliminated and the previous variable re-allocated at the same variable.

To preserve the previous interface and semantic of the `spill()` function, the internal behavior has to be altered. Keeping the pointer to the spill instruction should only be done at the `r32()` function. At the spill() function, we always write the register back to memory, no exceptions.

We only restore the previous allocation when the register is free and the reference of the previous allocation, which I call the spill allocation, matches the reference passed to `r32()`. When a register is used in another instruction, we erase the spill allocation information because the current allocation proves to be useful. Copy instructions skip this rule because we apply copy propagation and the register might not be used effectively after all, as described in the begin of this section.

# Unmodified Registers #

The previous section dealt with the type of spilling that happens when we're out of registers and need to allocate a new variable. The other type of spill can be optimized as well. When a register is only read, never written to, then we never have to write it back to memory. This is one of the simplest optimizations, but very effective. For every register, we just have to keep a flag to indicate whether it has been modified. For x86 instructions, all destination operands can be considered modified, while source operands are only read. Note that copy propagation makes the implementation a little more complicated.

# Minimal Register Restore #

Spilling all registers at branches isn't efficient. Most code consists of jumps every dozen instructions or so, which would result in code where hardly any variables are kept in registers. A more intelligent approach is to spill and reallocate only registers to restore the previous allocation state. For the example of a simple if-statement, it's even likely that all registers can keep their current allocation state. Situations are very similar for forward and backward jumps. In the former case, we wish to capture the register allocation state before the jump, and restore it to that state before the target label. For the latter case we'll capture the state before the label, and restore it before the jump. SoftWire implements `capture()` and `restore()` functions with exactly this functionality. `CodeGenerator::capture()` returns a structure of the `CodeGenerator::State` type, and `CodeGenerator::restore()` takes such state as argument. They are used in practice like this:
```
if(x == true)
{
   a++;
}
 
b = a;
```
Could be translated using:
```
cmp(x, 1);
State x = capture();
jne("ifBranch");
inc(a);
restore(x);
label("ifBranch");
mov(b, a);
```
Assuming that ``a was already allocated, the allocation state before and after the branch are the same, so no spills or reallocations would be made at all. Compared to spilling the whole register set this is a massive improvement!

The implementation is again quite compact but tricky. `capture()` takes a copy of the whole allocation state at the moment of the call. But it also disables all load elimination, copy propagation and spill elimination. This is required because all these optimizations assume we're working in a 'basic block' where code is executed strictly linearly. The `restore()` function first also disables all optimizations, then it compares the current allocation state for every register with the one stored in the passed state. All registers which have different allocation states are spilled. After all the spills, the allocation states are compared again and the old variables are allocated to the registers. Note that spilling and reallocating registers one at a time wouldn't work.

There are two more optimizations possible here. Very often, variables are just allocated to other registers inside a conditional block. So at the `restore(`) we don't have to spill two registers and load them again. We can just copy the value from the register that has the allocation state we're trying to attain. The second optimization is to avoid that variables get reallocated to different registers at all. At every spill operation, we can remember which variable was last allocated at that register. When the variable is reallocated, we can prefer the same register, if it's available. So not every free register is equivalent. Nothing is left at chance.

This optimization makes branches a lot more attractive. In any case it attempts to keep a maximum number of variables allocated. Other optimizations are not possible across basic blocks, but in many cases formulas are limited to one basic block so the impact of this is minimal.

# Memory Operands #

Until now we've always required that a variable gets allocated to a register before it is used. While it is safe to assume that registers are faster than memory accesses, this strategy can actually work adversely. On an x86 processor we only have six general-purpose registers, so we have to use them sparingly. In functions with more than six variables, which is quickly the case, we will be spilling very frequently to keep them in registers, but this way we actually miss our goal.

A more efficient approach is to require only destination operands to be in registers. An arithmetic x86 instruction with a destination operand in memory will cause the processor core to read the value of the destination operand, apply the operation to it, and write it back. So the destination operand is both read and written while the source operand is only read. Hence it's best to make the destination operand a register, while the source operand can remain in memory. This is also the way it's actually implemented in SoftWire's local variable classes' operators. If the source operand is not yet allocated in a register, we read the value from memory, else we use the register. For this we use SoftWire's `m32()` function.

This time having no look-ahead is a hard limitation. If we knew in advance which variables will be used intensively, even if only as source operand, we could force it to be allocated to a register. But look-back approaches can't help us here. The compiler front-end, as always, can give hints though. But in my experience this is hardly worth it. Modern processors deal with frequent accesses to the same memory quite efficiently. Also, frequently used variables will often be used as destination operand as well, so they will quickly be allocated to a register.

Either way we can also use some hybrid approach. When sufficient registers are available, we can assume that it's worth it to allocate variables to registers even when they are first used as source operand. Real code benchmarks would be required to determine the optimal decision parameters.

# Practical Results #

Several key optimizations have been presented. Implementing them all successfully was sometimes very complicated. I do not know of any previous work like this so it was all very experimental for me. So even though the above explanations might sound very logical, making it all work in every situation is very tricky. The allocation state of one register consists of more than a dozen components, so at every related function they have to be updated correctly.

The first approach to deal with this complexity was of course divide-and-conquer. Just as the optimizations are described one by one, they were implemented one by one. This resulted in adding several functions in the `RegisterAllocator` class to switch the specific optimizations on and off. This allowed also to test them separately, or in combinations, until finally they all worked together.

The actual testing was done by automatically generating very long functions, with a random combination of arithmetic operations, copy operations, introducing new variables, and spilling. Using various distributions it was possible to simulate extreme situations such as having multiple copy propagations of the same registers. Errors were detected by executing the same operations, but as a regular C++ function. Differences in output values were analyzed by reducing the situation to a minimum number of operations, and looking at what the last couple of instructions did wrong.

This technique of stress-testing was very efficient, but not flawless. The ultimate testing was done by swShader, the project for which SoftWire was created in the first place. It creates several functions at run-time, which all depend on each other's output. When an optimization caused a minor error, this was immediately visible, literally, in the produced 3D images. Sometimes it also caused crashes because of bad memory accesses, which made it easier to track the bug.

So, although any code can never be fully flawless, I am confident that SoftWire's new optimizations can be used in many applications without trouble. In case you do find errors, and believe it's caused by SoftWire internally, please contact me to have a look at the issue!

There are currently no hard theoretical proofs of these algorithms. Although feasible, they are more complicated than one would expect. For example, load elimination is very simple in concept, but becomes harder when combined with copy propagation, spill elimination, etc. To prove it, we would have to identify the dependencies between the techniques, and show that for every operation related to register allocation, the total allocation state remains valid. Formal proofs can be expected in future 'academic' versions of this article. For now, their application in complex dynamically generated functions in swShader should be convincing that these proofs do exist.

# Conclusion #

In this article I've presented new algorithms that allow a compiler back-end to generate vastly optimized code without the need for multiple passes or extra memory requirements. This makes the approach very interesting for fast JIT-compilers and native script compilers. But it doesn't end here, as there are many more optimizations that could be tried out. Since this is a work in progress, any kind of feedback is highly appreciated! Don't hesitate to contact me and discuss this article.

2005 - Nicolas Capens