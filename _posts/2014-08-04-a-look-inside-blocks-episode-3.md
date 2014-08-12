---
layout: post
title: 深度围观block：第三集(转)
---


- 目录
- 介绍
- 已知内容
- Block_copy()
- Block_release()
- 
##正文
###介绍

本文话费了很长时间才出炉。实际上，几个月之前就已经打好草稿了，只不过一直忙于写我的这本书:Effective Objective-C 2.0，所以没有时间完成本文。
接着之前的两篇文章：**深度围观block：第一集**和**深度围观block：第二集**，本文将更进一步了解当block被拷贝时发生了什么。可能你已经听过这样的说辞“block开始于栈”，以及“如果你希望将block保存下来，以便后续使用，那么必须对block进行拷贝”。那么，这是为什么呢？而在拷贝过程中实际又会发生什么情况？我一直在思考拷贝block时是利用了什么机制。就如之前介绍的block在进行值拷贝时发生了什么。本文我将揭晓这些疑问。
###已知内容
通过第一集和第二集两篇文章，我们可以知道block的内存布局如下图所示：
![](http://beyondvincent.com/wp-content/uploads/2013/07/block_layout.png)
block_layout

在第二集中，我们也知道了当block初始化的时候，会在栈中创建像上图这样的一个结构。由于这个结构是在栈上，而在栈空间是会被重复使用的。那么如果我们想要在以后继续使用该block，就必须要对block进行拷贝操作。拷贝操作需要调用Block_copy()函数，或者可以理解为给block发送一个copy消息(因为block可以看成一个Objective-C对象)，这也会调用**Block_copy()**函数。
下面我们就来看看Block_copy()函数都做了什么。
Block_copy()

我们首先来看看Block.h文件，在这里面可以看到如下定义：

	#define Block_copy(...) ((__typeof(__VA_ARGS__))_Block_copy((const void *)(__VA_ARGS__)))

	void *_Block_copy(const void *arg);

可以看出，**Block_copy()**实际上就是一个宏定义(**#define**)，该宏定义将传入的参数(const void *)做强制类型转换，然后再传给_Block_copy()。我们也可以在实现文件runtime.c中找到**_Block_copy()**的原型：

	void *_Block_copy(const void *arg) {
    	return _Block_copy_internal(arg, WANTS_ONE);
	}

上面的方法调用了**_Block_copy_internal()**函数，并传入block本身(arg)以及WANTS_ONE。要弄白具体意思，需要查看_Block_copy_internal方法的实现，该方法也是在**runtime.c**文件中。如下代码所示(已经去除掉了一些无关的内容：主要是垃圾回收相关)：

	static void *_Block_copy_internal(const void *arg, const int flags) {
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;

    // 1
    if (!arg) return NULL;

    // 2
    aBlock = (struct Block_layout *)arg;

    // 3
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }

    // 4
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }

    // 5
    struct Block_layout *result = malloc(aBlock>descriptor->size);
    if (!result) return (void *)0;

    // 6
    memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first

    // 7
    result->flags &= ~(BLOCK_REFCOUNT_MASK);    // XXX not needed
    result->flags |= BLOCK_NEEDS_FREE | 1;

    // 8
    result->isa = _NSConcreteMallocBlock;

    // 9
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) {
        (*aBlock->descriptor->copy)(result, aBlock); // do fixup
    }

    return result;
	}

下面来看看该方法都做了些什么事情：


1. 如果传入的参数是NULL则直接返回NULL。这样可以保证传入一个NULL block时函数的安全性。
2. 将参数强制转换为一个指针，该指针指向一个Block_layout结构对象。实际上在第一集中就介绍了Block_layout结构：这是一个内部使用的数据结构，该结构组成一个block，其中包含一个block的实现函数，以及另外几个元数据。
3. 如果block的flags包含BLOCK_NEEDS_FREE，说明这是一个堆block(a heap block)。这种情况下，需要做的事情就是增加引用计数(reference count)，然后将同一个的block返回。
4. 如果block是一个全局block(参考第一集)，那么不用做任何事情，直接返回同一个block即可——因为全局block是一个单例(singleton)。
5. 如果到这一步了，可以肯定该block肯定被分配在栈上。这种情况，需要将block拷贝到堆上。这也是最有趣的一部分。首先是利用malloc()函数在堆上创建block对应size大小的内存空间。如果失败了，就返回NULL，否则继续往下执行。
6. 利用memmove()函数将分配在栈中的block按位拷贝至刚刚在堆上分配的空间中。按位拷贝可以确保block中的所有元数据都能准确的进行拷贝，例如block的descriptor。
7. 接着需要更新一下block的flags。第一行代码是确保引用计数被设置为0。后面紧跟的注释表示这不是必须的——估计此时引用计数已经是0了。我猜测这行代码的作用是为了防止潜在的bug，会引起引用计数不为0的情况。第二行代码是设置BLOCK_NEEDS_FREE标志，这标示该block是一个堆block，当引用计数变为0时，需要free掉。后面紧跟的| 1是将block的引用计数设置为1。
8. 将block的isa指针设置为 _NSConcreteMallocBlock，这就意味着该block是一个堆block。
9. 最后，如果block有一个拷贝辅助函数(a copy helper function)，那么就调用它。如果有必要的话，表一起会生成一个拷贝辅助函数。例如block需要拷贝对象的时候，拷贝辅助函数会retain住已经拷贝的对象。

思路很清晰吧！现在你应该知道当block被拷贝时会发什么了！下面还需要了解一下当release时又回发生什么？

###Block_release
与Block_copy对应的是Block_release()。同样，Block_release()也是一个宏定义，如下所示：

	#define Block_release(...) _Block_release((const void *)(__VA_ARGS__))

实际上，跟**Block_copy()**类似，**Block_release()**会为我们把参数进行强制类型转换。这样开发者就不用亲自来处理转换的事情了。
下面我们来看看**_Block_release()**函数(为了看起来清晰点，我对代码重排了一下，并移除了垃圾回收相关的代码)：

	void _Block_release(void *arg) {
    // 1
    struct Block_layout *aBlock = (struct Block_layout *)arg;
    if (!aBlock) return;

    // 2
    int32_t newCount;
    newCount = latching_decr_int(&aBlock->flags) & BLOCK_REFCOUNT_MASK;

    // 3
    if (newCount > 0) return;

    // 4
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        if (aBlock->flags & BLOCK_HAS_COPY_DISPOSE)(*aBlock->descriptor->dispose)(aBlock);
        _Block_deallocator(aBlock);
    }

    // 5
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        ;
    }

    // 6
    else {
        printf("Block_release called upon a stack Block: %p, ignored\n", (void *)aBlock);
    }
	}
来看看他们都做了些什么：

1. 首先将参数强制转换为Block_layout结构。如果传入的是NULL，那么为了函数的安全起见，将直接返回。
2. 将block的引用计数标志位减1(还记得Block_copy()中将这个引用计数标志位设置为1吗？)。
3. 如果newCount大于0，说明还有别的对象引用了这个block，所以并不需要立即释放block，只需简单的返回即可。
4. 否则，如果flags中包含BLOCK_NEEDS_FREE，那么说明这个block是分配到堆上的，并且如果引用计数为0，那么需要释放这个block。首先是调用了block的dispose辅助函数，该函数跟copy辅助函数相反，负责做相反的操作，例如释放掉所有在block中拷贝的变量等。最后使用_Block_deallocator函数释放掉block，如果你去runtime.c文件中看看，会发现该函数的尾部是一个指向free的函数指针，也就是释放掉malloc分配的内存。
5. 如果block是全局的，那么什么事情也不用做。
6. 如果代码执行到这里了，会发生一些奇怪的事情：因为正在尝试将栈上的block释放掉，所以这行代码是为了提醒开发者的。在程序实际运行过程中，永远不会看到这里的提示。

Coool！就是这些了，没有更多，也没有再复杂的东西了！

转载：[破船之家](http://beyondvincent.com/blog/2013/07/09/99/)
