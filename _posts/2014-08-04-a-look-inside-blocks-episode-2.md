---
layout: post
title: 深度围观block：第二集(转)
---

- 目录
- 介绍
- block类型
- block的拷贝范围
- block拷贝对象的类型

##正文
##介绍

本文接着上一篇文章(深度围观block：第一集)，继续从编译器的角度深度围观block。在本文中，将介绍block并不是一成不变的，以及block在栈上的构成。

###block类型
在第一篇文章中，我们已经看到block有一个**_NSConcreteGlobalBlock**这样的类。由于所有变量都是已知的，所以在编译期间，block的结构(structure)和描述(descriptor)都将全部被初始化。关于block这里有几种不同的类型，每种类型都有对应的类。为了简单起见，这里只考虑其中三种：

1. _NSConcreteGlobalBlock是定义一个全局的block，在编译器就已经完成相关初始化任务。这种类型的block不会涉及到任何拷贝，例如一个空的block。
2. _NSConcreteStackBlock是一个分配在栈上的block。这里是所有最终被拷贝到堆(heap)上的block的开始。
3._NSConcreteMallocBlock是分配到堆(heap)上的block。拷贝完一个block之后，这就会结束。当block的引用计数变为0，该block就会被释放。

### block拷贝范围
这次我们来看看另外一些代码，如下所示：

	#import <dispatch/dispatch.h>

	typedef void(^BlockA)(void);
	void foo(int);
    
    __attribute__((noinline))
    void runBlockA(BlockA block) {
    	block();
    }
    
    void doBlockA() {
   		 int a = 128;
   		 BlockA block = ^{
    	foo(a);
    	};
   		 runBlockA(block);
    }

为了让block拷贝一些内容，上面的代码中调用了foo函数，并给这个函数传递了一个变量。再说一下，本文涉及到的汇编代码是与armv7相关指令。下面是其中一部分汇编指令：

	.globl  _runBlockA
  	  .align  2
  	  .code   16                      @ @runBlockA
   	 .thumb_func     _runBlockA
	_runBlockA:
	    ldr     r1, [r0, #12]
 	   bx      r1
上面的汇编代码与**runBlockA**函数相关，这跟第一篇文章中的相同——都是调用了block中的invoke函数。接着是**doBlockA**汇编代码，如下所示：

	.globl  _doBlockA
  	 	 .align  2
   		 .code   16                      @ @doBlockA
   		 .thumb_func     _doBlockA
	_doBlockA:
 	   push    {r7, lr}
 	   mov     r7, sp
 	   sub     sp, #24
  	   movw    r2, :lower16:(L__NSConcreteStackBlock$non_lazy_ptr-(LPC1_0+4))
  	   movt    r2, :upper16:(L__NSConcreteStackBlock$non_lazy_ptr-(LPC1_0+4))
   	   movw    r1, :lower16:(___doBlockA_block_invoke_0-(LPC1_1+4))
	LPC1_0:
  	   add     r2, pc
       movt    r1, :upper16:(___doBlockA_block_invoke_0-(LPC1_1+4))
       movw    r0, :lower16:(___block_descriptor_tmp-(LPC1_2+4))
	LPC1_1:
   	   add     r1, pc
  	   ldr     r2, [r2]
   	   movt    r0, :upper16:(___block_descriptor_tmp-(LPC1_2+4))
   	   str     r2, [sp]
   	   mov.w   r2, #1073741824
  	   str     r2, [sp, #4]
  	   movs    r2, #0
	LPC1_2:
   		 add     r0, pc
   		 str     r2, [sp, #8]
  	  	 str     r1, [sp, #12]
    	 str     r0, [sp, #16]
    	 movs    r0, #128
    	 str     r0, [sp, #20]
    	 mov     r0, sp
    	 bl      _runBlockA
     	 add     sp, #24
    	 pop     {r7, pc}
看看，这跟之前的代码有所不同了。看起来这不仅仅是从一个全局的符号中加载block，而且还做了额外的一些事情。乍一看这么多代码让人有点无从下手，不过认真看，还是很容易理解的。从上面的代码可以看出，编译器已经忽略了对代码排序的优化，为了方便阅读代码，我对上面的汇编代码重新进行排序(当然，请相信我，这不会影响任何功能)。下面是我重排好的代码效果：
_doBlockA:
        // 1
        push    {r7, lr}
        mov     r7, sp

        // 2
        sub     sp, #24

        // 3
        movw    r2, :lower16:(L__NSConcreteStackBlock$non_lazy_ptr-(LPC1_0+4))
        movt    r2, :upper16:(L__NSConcreteStackBlock$non_lazy_ptr-(LPC1_0+4))
LPC1_0:
        add     r2, pc
        ldr     r2, [r2]
        str     r2, [sp]

        // 4
        mov.w   r2, #1073741824
        str     r2, [sp, #4]

        // 5
        movs    r2, #0
        str     r2, [sp, #8]

        // 6
        movw    r1, :lower16:(___doBlockA_block_invoke_0-(LPC1_1+4))
        movt    r1, :upper16:(___doBlockA_block_invoke_0-(LPC1_1+4))
	LPC1_1:
        add     r1, pc
        str     r1, [sp, #12]

        // 7
        movw    r0, :lower16:(___block_descriptor_tmp-(LPC1_2+4))
        movt    r0, :upper16:(___block_descriptor_tmp-(LPC1_2+4))
	LPC1_2:
        add     r0, pc
        str     r0, [sp, #16]

        // 8
        movs    r0, #128
        str     r0, [sp, #20]

        // 9
        mov     r0, sp
        bl      _runBlockA

        // 10
        add     sp, #24
        pop     {r7, pc}
下面我们来看看这些代码都做了什么：


1. 开场白。首先将 r7 push到栈上面——因为r7会被覆盖，而r7寄存器中的内容在跨函数调用时是需要用到的。lr是链接寄存器(link register)，该寄存器中存储着当这个函数返回时需要执行下一条指令的地址。接着mov这条指令的作用是把栈指针保存到r7寄存器中。

2. 从栈指针所处位置开始减去24，也就是在栈空间上开辟24字节来存储数据。
3. 这里涉及到的代码是为了对符号L__NSConcreteStackBlock$non_lazy_ptr进行寻址，由于跟pc(program counter)相关联，所以无论代码处于二进制文件中任何位置，当最终链接时，都能对该符号做到正确的寻址。
4. 将值1073741824存储到栈指针 + 4 的位置。
5. 将值存储到栈指针 + 8 的位置。现在，将要发生什么可能已经变得逐渐清晰了——在栈上创建了一个Block_layout结构的对象！到现在为止，已经设置了该结构的3个值：isa指针，flags和reserved值。
6. 将___doBlockA_block_invoke_0存储至栈指针 + 12的位置。这是block结构中的invoke。
7. 将___block_descriptor_tmp存储至栈指针 + 16的位置。这是block结构中的descriptor。
8. 将值128存储到栈指针 + 20的位置。如果回头看看Block_layout结构，可以看到里面只应该有5个值。那么在这个block结构体后面存储的128是什么呢？——注意到这个128实际上就是在block中拷贝的变量的值。所以这肯定就是存储block使用到的值的地方——在Block_layout结构尾部。
9. 现在栈指针指向了已经完成初始化之后的block结构，在这里的汇编指令是将栈指针装载到r0中，然后调用runBlockA函数。(记住：在ARM EABI中，r0中存储的内容被当做函数的第一个参数)。
10. 最后将栈指针加上24，这样就能够把最开始减去的24(在栈上开辟的24位空间)收回来。接着将栈中的两个值pop到r7和pc寄存器中。这里pop到r7中的，跟最开始从r7中push至栈中的内容是一致的，而pc的值则是最开始push lr到栈中的值，这样当函数返回时，可以让CPU能够正确的继续执行后续指令。


Cooool！如果你一直认真看到这里，那么相信你的收获已经非常多了！

下面我们再看看block中的invoke函数和descriptor。希望跟第一集中的不要有太大差别。如下汇编代码：

	.align  2
  	  	.code   16                      @ @__doBlockA_block_invoke_0
  	 	 .thumb_func     ___doBlockA_block_invoke_0
	___doBlockA_block_invoke_0:
 	 	  ldr     r0, [r0, #20]
  		  b.w     _foo

  	 	 .section        __TEXT,__cstring,cstring_literals
	L_.str:                                 @ @.str
   		 .asciz   "v4@?0"

  	 	 .section        __TEXT,__objc_classname,cstring_literals
	L_OBJC_CLASS_NAME_:                     @ 	@"\01L_OBJC_CLASS_NAME_"
  	 	 .asciz   "\001P"

   		 .section        __DATA,__const
   		 .align  2                       @ @__block_descriptor_tmp
	___block_descriptor_tmp:
  	 	 .long   0                       @ 0x0
    	 .long   24                      @ 0x18
   		 .long   L_.str
   		 .long   L_OBJC_CLASS_NAME_

看着没错，跟第一集中的没多大区别。唯一不同的就是block descriptor中的size——现在是**24**(之前是**20**)。这是因为block拷贝了一个整型值，所以block的结构需要24个字节，而不再是标准的20个字节了。在之前的代码中，我们已经分析了在创建block时，多出的4个字节被添加到block结构的尾部。
在实际的block函数中，例如___doBlockA_block_invoke_0，可以看到从block结构尾部读取出相关值，如r0 + 20，就是在block中拷贝的变量。
###block拷贝对象的类型
下面我们来看看如果block拷贝的是别的对象类型(例如 NSString)，而不是integer，会发生什么呢？如下代码：

	#import <dispatch/dispatch.h>

	typedef void(^BlockA)(void);
	void foo(NSString*);

	__attribute__((noinline))
	void runBlockA(BlockA block) {
  	  block();
	}

	void doBlockA() {
  	  NSString *a = @"A";
   	  BlockA block = ^{
    	    foo(a);
   	 };
   	 runBlockA(block);
	}
由于**doBlockA**变化不大，所以在此不深入介绍。这里感兴趣的是根据上面代码创建的block descriptor结构：

	.section        __DATA,__const
  	  .align  4                       @ @__block_descriptor_tmp
	___block_descriptor_tmp:
	    .long   0                       @ 0x0
	    .long   24                      @ 0x18
	    .long   ___copy_helper_block_
    	.long   ___destroy_helper_block_
    	.long   L_.str1
    	.long   L_OBJC_CLASS_NAME_
注意看上面的汇编代码中有指向两个函数(__copy_helper_block和__destroy_helper_block)的指针。下面是这两个函数的定义：

	.align  2
	    .code   16                      @ @__copy_helper_block_
	    .thumb_func     ___copy_helper_block_
	___copy_helper_block_:
	    ldr     r1, [r1, #20]
	    adds    r0, #20
	    movs    r2, #3
	    b.w     __Block_object_assign
	
	    .align  2
	    .code   16                      @ @__destroy_helper_block_
	    .thumb_func     ___destroy_helper_block_
	___destroy_helper_block_:
	    ldr     r0, [r0, #20]
	    movs    r1, #3
	    b.w     __Block_object_dispose

这里我先假设当block被拷贝和销毁时，都会调用这里的函数。那么被block拷贝的对象肯定会发生reatain和release。上面的代码中，可以看出如果r0和r1包含有效数据时，拷贝函数接收两个参数(r0和r1)。而销毁函数接收一个参数。可以看出所有的拷贝和销毁任务都应该是由__Block_object_assign和__Block_object_dispose两个函数完成的。这两个函数位于block的运行时代码中(是LLVM里面compiler-rt工程的一部分)。
如果你希望了解一下block运行时相关代码，可以来这里下载源码：http://compiler-rt.llvm.org。特别关注一下里面的runtime.c文件。

转载：[破船之家](http://beyondvincent.com/blog/2013/07/09/99/)
