---
layout: post
title: 深度围观block：第一集(转)
---

##深度围观block：第一集

![Alt text](http://beyondvincent.com/wp-content/uploads/2013/07/blocks_2x.png)

- 目录：
- 简介
- 基础知识
- 深入一个简单示例
- 源码在这里
- 何去何从

##正文

###简介
今天我们从编译器的角度观察一下block内部是如何工作的。这里说的block是指苹果为C语言增加的具有闭包性(closure)的一个功能，block已经是clang/LLVM编译器所支持的一部分了。我一直在想block是什么，以及它是如何奇迹般的出现在Objective-C对象中(开发者可以像处理实例对象一样，对block进行copy、retain、release)。本文我首先深入的介绍一点关于block的那些事。

**基础知识**

用过block的开发者都知道，下面的代码就是一个block：

    void(^block)(void) = ^{  
    NSLog(@"I'm a block!");
    };

上面的代码中创建了一个名为block的变量，并把一个简单的block代码赋值给这个变量。代码很简单，不是吗？不！！！在这里我想要搞清楚编译器对这点代码都做了些什么。
更进一步，下面的代码我给block传递了一个变量：

     void(^block)(int a) = ^{
     NSLog(@"I'm a block! a = %i", a);
    };

而下面的代码是从block中返回一个值：

    	int(^block)(void) = ^{
     	   NSLog(@"I'm a block!");
      	  return 1;
    	};

作为一个封闭的包，block将所处的上下文封装到了block中：

    int a = 1;
    void(^block)(void) = ^{
    NSLog(@"I'm a block! a = %i", a);
    };


编译器对上面这些代码具体是如何处理的——这才是我所感兴趣的。

**深入一个简单示例**

首先我的思路是看看编译器是如何编译一个非常简单的block。来看看如下代码：

    #import <dispatch/dispatch.h>
    
    typedef void(^BlockA)(void);
    
    __attribute__((noinline))
   	 void runBlockA(BlockA block) {
  	  block();
    }
    
    void doBlockA() {
    	BlockA block = ^{
   		 // Empty block
   		 };

   		 runBlockA(block);
    }

之所以要用上面这样的代码，是因为我想看看block是如何创建的，以及如何调用一个block。如果block的创建和调用都在一个函数里面，那么优化器(optimiser)可能会对代码做优化处理，导致我们看不到任何感兴趣的东西，所以我给runBlockA函数添加了noinline，这样优化器就不会在doBlockA函数中对runBlockA的调用做内联优化处理。
上面代码通过编译器编译之后(armv7，03)，会得到如下汇编指令：

    .globl  _runBlockA
   		 .align  2
   		 .code   16  @ @runBlockA
   		 .thumb_func _runBlockA
    _runBlockA:
 		   @ BB#0:
  	 	   ldr r1, [r0, #12]
  		   bx  r1
上面的汇编代码是对应runBlockA函数——这相当的简单。注意观察之前的源码，可以知道这个函数只是简单的调用了block。在ARM EABI中，将r0(寄存器r0)设置为第一个参数。第一条指令(r1)是将存储在地址为r0 + 12的值装载到寄存器r1中。这可以理解为指针的解引用——读12个字节到寄存器中。然后跳转到这个地址执行后面的指令。注意，这里使用了r1，而r0没有被修改，仍然是原来的block。所以这里很有可能是利用第一个参数来调用block。
据此，可以确定block在结构中的一些排序规则：block被当做执行的函数时存储在某个结构中，并占据了12个字节。当传递一个block时，指向这些结构的一个指针被传递进来了。
下面来看看doBlockA函数：

	.globl  _doBlockA
	    .align  2
  		  .code   16                      @ @doBlockA
 	   .thumb_func     _doBlockA
	_doBlockA:
	    movw    r0, :lower16:(___block_literal_global-(LPC1_0+4))
 		movt    r0, :upper16:(___block_literal_global-(LPC1_0+4))
	LPC1_0:
  	   	 add     r0, pc
    	 b.w     _runBlockA

OK，上面的代码也不复杂——这是关于pc(program counter)的相关加载。你可以将其看做是把变量___block_literal_global的地址加载到r0中。然后调用runBlockA函数。因为从之前的源码中，可以知道我们把block传递给了runBlockA，所以这里的___block_literal_global一定就是那个被传递的block对象了。
到目前为止，我们对上面的源码的运作有一些眉目了！不过这里的___block_literal_global是什么呢？继续看汇编代码，可以找到如下这样的内容：

	.align  2                       @ @__block_literal_global
	___block_literal_global:
 	   .long   __NSConcreteGlobalBlock
 	   .long   1342177280              @ 0x50000000
 	   .long   0                       @ 0x0
       .long   ___doBlockA_block_invoke_0
  	   .long   ___block_descriptor_tmp
Cool！上面的汇编代码看起来像是一个结构体。在结构体中又5个值，每个值有4个字节(long)。这肯定就是RunBlockA调用中涉及到的那个block对象。再细看一下，12个字节所在处就像一个函数指针：___doBlockA_block_invoke_0。这也是runBlockA函数中跳转执行的那个分支(bx r1)。
那么上面的汇编代码中__NSConcreteGlobalBlock又是何物？OK，现在先不介绍这个，后面会做介绍哦！下面我们来看看另外两个感兴趣的东西：___doBlockA_block_invoke_0和___block_descriptor_tmp，这两个东东同样出现在了汇编代码中：

	.align  2
	    .code   16                      @ @__doBlockA_block_invoke_0
  	  .thumb_func     ___doBlockA_block_invoke_0
	___doBlockA_block_invoke_0:
	    bx      lr

  	   .section        __DATA,__const
  	   .align  2                       @ @__block_descriptor_tmp
	___block_descriptor_tmp:
  	  .long   0                       @ 0x0
  	  .long   20                      @ 0x14
  	  .long   L_.str
  	  .long   L_OBJC_CLASS_NAME_

  	  .section        __TEXT,__cstring,cstring_literals
	L_.str:                                 @ @.str
   	 .asciz   "v4@?0"

   	 .section        __TEXT,__objc_classname,cstring_literals
	L_OBJC_CLASS_NAME_:                     @ @"\01L_OBJC_CLASS_NAME_"
    .asciz   "\001"

上面的代码中
**___doBlockA_block_invoke_0**
看起来有点像block的实现部分，只不过这里的block是空的，所以会立即返回(刚开始我们就期望编译一个空的block哦)。
接着看看
**___block_descriptor_tmp** 。这里可以看到另外一个数据结构——有4个值。其中第2个是20，这表示___block_literal_global的大小。接着是一个名为.str的C字符串，它的值为v4@?0，看起来有点像某个类型的编码形式。这可能是block 类型的编码(例如返回void和不携带任何参数)。上面代码中别的一些值我暂时还不清楚。
源码在这里
没错，这里有源代码！这是LLVM中compiler-rt项目的一部分。查看代码，我发现在Block_private.h文件中，有如下相关代码：

	struct Block_descriptor {
  	  unsigned long int reserved;
   	  unsigned long int size;
      void (*copy)(void *dst, void *src);
 	  void (*dispose)(void *);
	};

	struct Block_layout {
 	   void *isa;
  	   int flags;
   	   int reserved;
   	   void (*invoke)(void *, ...);
       struct Block_descriptor *descriptor;
    	 /* Imported variables. */
	};

这看起来很熟悉吧！其中Block_layout结构体就是
**___block_literal_global** ，而**Block_descriptor**结构体则是**__block_descriptor_tmp** 。细看 **Block_descriptor**中的第2个变量size正如我之前描述的一样(表示___block_literal_global的大小)。在Block_descriptor中的第3和第4个值有点奇怪。这看起来有点想函数指针，但是在上面的汇编代码中看起来更像是两个字符串。现在我忽略掉这个细节。
Block_layout 中的isa肯定就是
** __NSConcreteGlobalBlock**，这也将确定block如何能够模拟Objective-C对象。如果__NSConcreteGlobalBlock是一个Class，那么Objective-C消息派送系统会将block对象当做一个普通的对象来处理。这跟如何处理toll-free bridging工作类似。更多相关toll-free bridging信息，可以阅读Mike Ash写的[一篇优秀文章](https://mikeash.com/pyblog/friday-qa-2010-01-22-toll-free-bridging-internals.html)。
将所有的代码片段拼凑起来，编译器做的工作内容看起来如下所示：

	#import <dispatch/dispatch.h>

	__attribute__((noinline))
	void runBlockA(struct Block_layout *block) {
	    block->invoke();
	}

	void block_invoke(struct Block_layout *block) {
 	   // Empty block function
	}

	void doBlockA() {
  	  struct Block_descriptor descriptor;
  	  descriptor->reserved = 0;
 	   descriptor->size = 20;
 	   descriptor->copy = NULL;
  	  descriptor->dispose = NULL;

    	struct Block_layout block;
       	block->isa = _NSConcreteGlobalBlock;
   		 block->flags = 1342177280;
   		 block->reserved = 0;
   		 block->invoke = block_invoke;
   		 block->descriptor = descriptor;

  	     runBlockA(&block);
	}
非常不错！通过上面的介绍，我们可以了解很多关于block内部的东西。

转载：[破船之家](http://beyondvincent.com/blog/2013/07/09/99/)
