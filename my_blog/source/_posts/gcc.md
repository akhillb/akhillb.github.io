---
title: GCC compile time optimizations
date: 2018-03-12 22:07:36
tags:
-	GCC
- 	Compiler
-  Optimizations
-  C
-  Assembler
-  Assembly
---

Recently, over a discussion, my friend claimed that NodeJS is approximately 1.5 times faster than C, even for a task that would just include a simple for loop over 100000000 numbers. Now that is a huge claim and of course I did not believe it.

<!-- more -->
The following were his code samples in NodeJS and C.

*Note:* All the provided numbers were obtained by running the code samples on a MacBook Pro with a 2.7 GHz processor and 8 GB RAM. I used GCC version 7.2.0 and Node version 7.10.1. 

{% codeblock  loop.js lang:javascript %}
var sum = 0;
for(var i = 0; i < 100000000; i++) {
  sum += i;
}
{% endcodeblock %}

{% codeblock  loop.c lang:c %}
int main() {
  long sum = 0;
  for (long i = 0; i < 100000000; i++) {
  	sum += i;
  }
  return 0;
}
{% endcodeblock %}

Then he timed both of the above. The following were the recordings

{% codeblock lang:bash %}
$ time node loop.js

real	0m0.241s
user	0m0.153s
sys	0m0.021s

$ time ./a.out

real	0m0.264s
user	0m0.257s
sys	0m0.003s
{% endcodeblock %}

Of course after looking the way he compiled his binary, I figured out how to his C code run faster without making any changes. I took his terminal and tried the following

{% codeblock lang:bash %}
$ gcc -O2 loop.c

real	0m0.004s
user	0m0.001s
sys	0m0.001s
{% endcodeblock %}

Now the C code was way faster than it was before. The `-O2` flag just tells the compiler to apply all a bunch of known optimisations while compiling the binary. You can read more about it over [here](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html). I was expecting the binary to run faster than the NodeJS script, but I did not expect it to improve it by such a huge factor. So, I was curious.

I took a look at all the options that are applied for optimizations over [here](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html). For O2 level there were too many options that were being applied for it and I didn't want to spend time reading each and every option and deciding if that option would have impacted it or not. I took a different approach. I wanted to see how different were the compiled machine code. Since it was a small program, hopefully it shouldn't take too much time to understand the generated assembly code. So I got the assembly code for both cases, one without -O2 option specified and the other with -O2 option.

This was the assembly output that was generated when optimizations were not enabled.
{% codeblock  unoptimized.s lang:assembly %}
	.text
	.globl _main
_main:
LFB1:
	pushq	%rbp
LCFI0:
	movq	%rsp, %rbp
LCFI1:
	movq	$0, -8(%rbp)
	movq	$0, -16(%rbp)
	jmp	L2
L3:
	movq	-16(%rbp), %rax
	addq	%rax, -8(%rbp)
	addq	$1, -16(%rbp)
L2:
	cmpq	$99999999, -16(%rbp)
	jle	L3
	movl	$0, %eax
	popq	%rbp
LCFI2:
	ret
LFE1:
	.section __TEXT,__eh_frame,coalesced,no_toc+strip_static_syms+live_support
EH_frame1:
	.set L$set$0,LECIE1-LSCIE1
	.long L$set$0
LSCIE1:
	.long	0
	.byte	0x1
	.ascii "zR\0"
	.byte	0x1
	.byte	0x78
	.byte	0x10
	.byte	0x1
	.byte	0x10
	.byte	0xc
	.byte	0x7
	.byte	0x8
	.byte	0x90
	.byte	0x1
	.align 3
LECIE1:
LSFDE1:
	.set L$set$1,LEFDE1-LASFDE1
	.long L$set$1
LASFDE1:
	.long	LASFDE1-EH_frame1
	.quad	LFB1-.
	.set L$set$2,LFE1-LFB1
	.quad L$set$2
	.byte	0
	.byte	0x4
	.set L$set$3,LCFI0-LFB1
	.long L$set$3
	.byte	0xe
	.byte	0x10
	.byte	0x86
	.byte	0x2
	.byte	0x4
	.set L$set$4,LCFI1-LCFI0
	.long L$set$4
	.byte	0xd
	.byte	0x6
	.byte	0x4
	.set L$set$5,LCFI2-LCFI1
	.long L$set$5
	.byte	0xc
	.byte	0x7
	.byte	0x8
	.align 3
LEFDE1:
	.subsections_via_symbols
{% endcodeblock %}


The following is the generated assembly when optimizations were enabled
{% codeblock  optimized.s lang:assembly %}
	.section __TEXT,__text_startup,regular,pure_instructions
	.align 4
	.globl _main
_main:
LFB1:
	xorl	%eax, %eax
	ret
LFE1:
	.section __TEXT,__eh_frame,coalesced,no_toc+strip_static_syms+live_support
EH_frame1:
	.set L$set$0,LECIE1-LSCIE1
	.long L$set$0
LSCIE1:
	.long	0
	.byte	0x1
	.ascii "zR\0"
	.byte	0x1
	.byte	0x78
	.byte	0x10
	.byte	0x1
	.byte	0x10
	.byte	0xc
	.byte	0x7
	.byte	0x8
	.byte	0x90
	.byte	0x1
	.align 3
LECIE1:
LSFDE1:
	.set L$set$1,LEFDE1-LASFDE1
	.long L$set$1
LASFDE1:
	.long	LASFDE1-EH_frame1
	.quad	LFB1-.
	.set L$set$2,LFE1-LFB1
	.quad L$set$2
	.byte	0
	.align 3
LEFDE1:
	.subsections_via_symbols
{% endcodeblock %}

If you take a look at the generated output, you can see the main difference apart from few simple instructions removed or added is that there are fewer labels in it. This must imply that the compiler has applied few optimizations to our for loop. Let's go over each difference one by one. I shall be taking the optimized.s and going through each line and explaining how it is different in the unoptimized code.

To begin with, the initial `.text` directive is replaced with `.section` directive. According to [Apple's developer documentation for their Assembler](https://developer.apple.com/library/content/documentation/DeveloperTools/Reference/Assembler/040-Assembler_Directives/asm_directives.html#//apple_ref/doc/uid/TP30000823-CJBDFFDF), both of them are actually equivalent when dynamic linking is enabled. This might hint that when optimizations are not enabled, the binary is statically linked ( I could not find anything that would confirm this ). The next instruction uses `align` to 4, which should be the length of `word` for this particular machine. Aligning your instructions / data is a known to help optimize your system's fetching respective instruction / data. So, it makes sense to have this, especially when there is a loop involved. Now, the only other differences you can see are the instructions between `LFB1` and `LFE1` and instructions appearing after `_EH_frame1`.

Going through the [dwarf2out.c](https://github.com/gcc-mirror/gcc/blob/gcc-4_8_2-release/gcc/dwarf2out.c) file, we can see that `LFB` indicates a beginning of the function and `LFE` indicates the end of the function. Adding an `L` before the label is GCC's notation of indicating it's a local label. `LCFI` would indicate call frame So everything between is the assembly output for our logic. As you can see there is little in between these lines in our optimized code, whereas the unoptimized code clearly has `jmp`, `cmpq` instructions. The presence of `jmp`, `cmpq` prooves that it is trying to increment by 1, check if the value is equal to certain thing and otherwise jump back to previous instruction. Our optmized simply has a `xorl %eax, %eax`, which is just clearing the value in `%eax%` register and setting it to 0 and then just a `ret` statement. So it is not even going in the loop. This would explain the reason for such a huge improvement. Logically it makes sense if we look at our for loop in C. It simply loops over and the resultant variable is never used. It can be considered as dead code. I tried to add a simple `printf` to prove if I were right. The resulting assembly code was different.

{% codeblock  loop.c with printf lang:C %}
#include <stdio.h>

int main() {
  long sum = 0;
  for (long i = 0; i < 100000000; i++) {
  	sum += i;
  }
  printf("%ld", sum);
  return 0;
}
{% endcodeblock %}

This was compiled with optimizations enabled.
{% codeblock print.s lang:assembly %}
	.cstring
lC0:
	.ascii "%ld\0"
	.section __TEXT,__text_startup,regular,pure_instructions
	.align 4
	.globl _main
_main:
LFB1:
	movabsq	$4999999950000000, %rsi
	subq	$8, %rsp
LCFI0:
	xorl	%eax, %eax
	leaq	lC0(%rip), %rdi
	call	_printf
	xorl	%eax, %eax
	addq	$8, %rsp
LCFI1:
	ret
LFE1:
	.section __TEXT,__eh_frame,coalesced,no_toc+strip_static_syms+live_support
EH_frame1:
	.set L$set$0,LECIE1-LSCIE1
	.long L$set$0
LSCIE1:
	.long	0
	.byte	0x1
	.ascii "zR\0"
	.byte	0x1
	.byte	0x78
	.byte	0x10
	.byte	0x1
	.byte	0x10
	.byte	0xc
	.byte	0x7
	.byte	0x8
	.byte	0x90
	.byte	0x1
	.align 3
LECIE1:
LSFDE1:
	.set L$set$1,LEFDE1-LASFDE1
	.long L$set$1
LASFDE1:
	.long	LASFDE1-EH_frame1
	.quad	LFB1-.
	.set L$set$2,LFE1-LFB1
	.quad L$set$2
	.byte	0
	.byte	0x4
	.set L$set$3,LCFI0-LFB1
	.long L$set$3
	.byte	0xe
	.byte	0x10
	.byte	0x4
	.set L$set$4,LCFI1-LCFI0
	.long L$set$4
	.byte	0xe
	.byte	0x8
	.align 3
LEFDE1:
	.subsections_via_symbols
{% endcodeblock %}

As you can see, this time it directly assigns the value 4999999950000000 to `rsi` register, which is the sum that we get in the loop. The compiler calculated it during compile time and replaced it in the assembly code without needing to run the loop during runtime.

To explain the differences in `.EH_frame1` section, we need to know what section implies. `.EH_frame1` section is made used for exception handling and printing stack traces. Since, we had more blocks in our unoptimized code, it would explain the shorter tables in this section.

There were other smaller optimizations that were use. Example being use `xorl %eax, %eax` to instead of `mov %eax, 0` since it uses only 2 bytes and on many CPUs it doesn't use an execution unit, thus saving power and resources. You can read more about it in this [fantastic article](https://randomascii.wordpress.com/2012/12/29/the-surprising-subtleties-of-zeroing-a-register/). These optimisations are just for a small program that does a simple job of looping over huge numbers, imagine the optimisations that the compilers do over huge programs. Do try the optimized option, I think most C and C++ compilers provide it. There is one catch, enabling optimizations might increase your compile time, so only try if your project is small in size or compile time is not an issue.