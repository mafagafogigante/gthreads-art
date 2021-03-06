## The x86-64 architecture and ABI

If you know how the x86-64 architecture works in big lines and you are familiar with its ABI, you should move directly to the next section.

Before jumping into the implementation I need to make sure you know the basics of the x86-64 architecture. The reason for this is that a small part of every user threading library is architecture dependent. I decided to choose this architecture for the implementation because it is used by most common computers and it is also the architecture of the computer I am using to write the examples!

### Basic architecture features

This architecture features a set of 16 registers (rax, rbx, rcx, rdx, rsi, rdi, rbp, rsp, rip, r8, r9, r10, r12, r13, r14, r15) which can store 64-bits numbers. They can be seen as global programming language variables stored in the CPU; contrary to C variables they come only in a fixed number! Moreover some of them are reserved for special purposes.

  * The **rip** register, also called *instruction pointer* or *program counter*. This register cannot be manipulated directly by machine instructions, it is dedicated to store the memory address of the next instruction to be executed.
  * The **rsp** register, also called *stack pointer*. All modern computers feature a stack that is used when calling and returning from functions, it is also used by C compilers to store automatic storage class variables.

Computers running x86-64 feature a main random access memory (RAM) which can be seen from our point of view as an array of bytes. Bytes can be accessed either by using their direct address (a 64-bits unsigned integer) or by using an address stored in any register of the CPU.

You can find more details and explanations on the [wikipedia page](http://en.wikipedia.org/wiki/X86-64).

### Stack management

A stack is a data structure on which one can *push* data on top, and *pop* to retrieve it. The machine stack is stored as a single chunk of memory. For this data structure to work properly, one simply needs a pointer that keeps the top address of the last pushed data &mdash; this is the role of rsp.

On x86-64, the stack grows downward, meaning that when a 64-bits value is pushed on the stack, we first subtract 8 to rsp and then store the value at this new address. Dually, when we need to retrieve a 64-bits value, we first get the value at rsp and add 8 to it to free the space taken by the value and uncover values below.

To understand why a stack is convenient, one needs to wonder how function calls are implemented on the bare hardware. What is provided by all hardware architectures is a way to jump from one point to another (a low level goto). However, this is not sufficient to implement functions. Consider the example below written in a pseudo-assembly language.

<div class="row">
	<div class="span5">
<pre><code>f:
		call g
		move rax -&gt; a_memory_location
		[...]</code></pre>
    </div>
	<div class="span5">
<pre><code>g:
		[...]
		move 0 -&gt; rax
		return</code></pre>
    </div>
</div>

Here `f` is calling the function `g` which will perform some operations then store 0 in the rax register and return. One way or another, when `f` calls `g`, the machine has to "remember" that when `g` terminates it will have to return into `f` to execute the `move` operation right after the function call. Using a register, say ebx, to remember it would be sufficient in this case, however if `g` calls another function `h`, then ebx would get overwritten and the returning address we need once we are back from `g` will be lost.

This is where a stack comes handy! The call instruction can be seen as a sequence of two operations, first it will push rip register (which points to the instruction right after the call), then it will jump to the address of the called function. Dually, the return instruction will pop the return address from the stack and jump to it. Note that on x86-64 it is impossible to implement the call operation as described above because no instruction (except call) allows to push rip.

As mentioned earlier, the stack can be used to store temporary variables. In this case what we do is simply decrement the stack pointer to allocate space on the stack, use the allocated space and when we no longer need it, increment the stack pointer by the same amount.

### The ABI (aka calling conventions)

To ensure compatibility between kernel and user programs, and between machine code generated by different tools, (at least) one Application Binary Interface (ABI) is defined for each couple of platform architecture. In these articles we will focus on the x86-64 System V ABI. The [reference document](http://www.x86-64.org/documentation/abi.pdf) describes it in great details. Fortunately, we will not need much of it!

We will simply focus on the way arguments are passed to functions and the way registers and stack are handled. Indeed, when a function is called, what assumptions the caller can make about the state of registers (which are global) when the called function returns? And you might have remarked that the call instruction I used above is not able to handle function arguments. So how do we know where the code in `g` expects its arguments? These two questions (and many more) are answered in the ABI.

Parameter passing is actually a complicated subject in the System V ABI, fortunately we will only need to pass values of size less than 64-bits that fit in registers (they are said to be of class INTEGER using the standard's terminology). The relevant quote from the standard follows.

> After the argument values have been computed, they are placed in registers, or pushed on the stack. [...] If the class is INTEGER, the next available register of the sequence %rdi, %rsi, %rdx, %rcx, %r8 and %r9 is used.

So when we need to call a function with first argument 0 and second argument 1, we will move 0 into rdi, 1 into rsi and use the call instruction provided by the machine. Because the ABI is respected by all the code running on the machine, we can be sure that the called function will "know" that its first argument is in rdi and its second argument is in rsi.

Concerning the register usage conventions, the table 3.4 in the ABI specification imposes that registers rbx, rbp, r12, r13, r14 and r15 must be conserved across function calls. This means in particular that if we have a value stored in rcx before calling a function we cannot assume that it will remain unchanged by the called function. The proper way to deal with this problem is usually to push the contents of rcx on top of the stack before calling the function and pop it after the called function returned.

### An example of compiled C

As a simple example to demonstrate both stack manipulations and register management, we will use the compiled version of the following C program.

	!c
	void f(int *x, int y) {
		*x = y;
	}

	int main(void) {
		int x;

		x = 0;
		f(&x, 2);
		return x;
	}

Note that you should not use the assembly generated by a C compiler to understand C semantics. Indeed, some choices made by the compiler may be implementation defined and they could change from one compiler to another; or even from one version to another of the same C compiler!

When we will have to write and read assembly in these tutorials, I will use the AT&T syntax because that is the one used by the GNU assembler (gas). This syntax is a lot less readable than the alternative (Intel) but you will eventually get used to it! Here are some basic hints to help you navigate through assembly

  * the prefix of an operand indicates its type
    + register names are prefixed by % as in `%rbp`,
    + constants are prefixed by $ as in `$16`
    + and symbols are not prefixed as in `f`;
  * operations suffixed by 'q' operate on 64-bits operands, operations suffixed by 'l' operate on 32-bits operands;
  * the 'e' version of a register (eax) is used to designate the low 32-bits of the corresponding 64-bits 'r' register (rax);
  * the destination operand is the second, meaning that `add %rax, %rbx` will add rbx and rax and store the result in rbx;
  * operands of the form n(%reg) designate the memory address stored in reg shifted by an offset of n.

Now you should be ready to see what gcc produces when fed the original input file. We will go through this code by logical blocks.

	!asm
	f:
	        pushq   %rbp
	        movq    %rsp, %rbp
	        movq    %rdi, -8(%rbp)
	        movl    %esi, -12(%rbp)
	        movq    -8(%rbp), %rax
	        movq    -12(%rbp), %edx
	        movl    %edx, (%rax)
	        popq    %rbp
	        ret

	main:
	        pushq   %rbp
	        movq    %rsp, %rbp
	        subq    $16, %rsp
	        movl    $0, -4(%rbp)
	        leaq    -4(%rbp), %rax
	        movl    $2, %esi
	        movq    %rax, %rdi
	        call    f
	        movl    -4(%rbp), %eax
	        leave
	        ret

I shrunk all the noise from the generated assembly and simply kept the essential assembly instructions. By now, you should be familiar with all registers named above. Let's see how calling conventions and stack manipulations apply here.

The function `f` takes two arguments, a pointer and an integer. This implementation of C uses 32-bits wide integers but the pointers, which must store memory addresses, have to be stored on 64-bits. Both are in the INTEGER class defined by the ABI meaning that they can be passed into registers. As we saw before, rdi is used to store the first argument (the `&x` pointer) and rsi is used to store the second argument (the `y` integer). Since `y` is 32-bits wide, we mov it into the esi register; the x86-64 semantics will ensure that the high 32-bits of rsi are zeroed by this instruction.

You might wonder what the `leaq -4(%rbp), %rax` instruction does; lea means Load Effective Address, this will simply compute the effective address of the source operand, here rbp minus 4, and store it in the destination operand, here rax. Be careful not to confuse it with `movq -4(%rbp), %rax` which will read a 64-bits value at rbp minus 4 in memory and store the result in rax.

We can also see that the local variable `x` is stored by the compiler in main's stack. Some room is made on the stack by `subq $16, %rsp` then `x` is referred to as `-4(%rbp)`. You might wonder why the compiler allocated 16 bytes when only 4 was necessary. This is answered by another aspect of the ABI: the stack alignment properties. The compiler must ensure, before calling a function, that the stack pointer is a multiple of 16.

The leave instruction, which is not critical in the next part of the article is simply an alias for the very common `movq %rbp, %rsp; popq %rbp`, it ensures that rbp (which is callee save) is preserved across function calls and frees the stack space allocated by the function.

After this short but intense introduction to the x86-64 machine architecture you should realize that there is nothing very special in a C program state. Implementing green threads basically adds fancy control features to C while ensuring that the invariants the compiler expects to be true (the ABI) are preserved at all times.

Now, let's move to the code.