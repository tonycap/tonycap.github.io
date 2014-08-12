---
layout: post
title: Block剧终：Objective-C中的闭包性和匿名函数(转)
---

- 目录
- 1、关于
	- 1.1匿名函数
	-	 1.2闭包性
- 2、Objective-C中的实现
	-     2.1将block当做参数来传递
	-     2.2闭包性
	-     2.3内存管理
	-     2.4示例

##正文

##1、关于

许多脚本语言都支持lambda表达式和匿名函数。这两个概念经常与闭包性(closure)相关。例如在JavaScript、ActionScript或PHP(5.3之后)中都有相关的概念。

其实在Objective-C语言中也提供了这两个概念的实现：叫做block。
自从Mac OS X 10.6之后，就可以使用block了，其实这样归功于Clang。

###1.1 匿名函数
就如名字暗示的一样，匿名函数实际上就是一个没有名字或者标示(identifier)的函数。匿名函数只有内容(也可以叫做body)，我们可以将其存储在一个变量中，以便之后使用，或者将其当做一个参数传递给另外一个函数使用。
在脚本语言的回调中经常使用到这个概念。
例如，在下面的JavaScript中，有一个名为foo的标准函数，接收一个callback当做参数，在函数中，调用了这个callback：

	function foo( callback )
	{
    	callback();
	}

这里有可能是定义了另外一个标准函数， 然后将这个标准函数当做参数传递给上面的函数：

	function bar()
	{
	    alert( 'hello, world' );
	}

	foo( bar );

不过这样一来，bar函数就会被声明在全局范围内，这就会带来一个风险：被另外一个相同名称的函数覆盖(override)了。

但是别担心，JavaScript语言允许callback函数在调用的时候才进行声明：

	foo
	{
	    function()
	    {
	        alert( 'hello, world' );
	    }
	);

在上面，可以看到这个callback实际上并没有标示(identifier)。它也不会存在于全局范围，因此也不会与别的已有函数产生冲突。

我们也可以把callback存储到一个变量中，同样也不回存在于全局范围，不过我们可以通过这个变量对这个callback进行重复利用：

	myCallback = function()
	{
	    alert( 'hello, world' );
	};
	
	foo( myCallback );

### 1.2闭包性
闭包性这个概念是允许一个函数访问其所声明上下文中的变量，甚至是在不同的运行上下文中。

下面我们再来看看JavaScript的相关代码：
	
	function foo( callback )
	{
	    alert( callback() );
	}
	
	function bar()
	{
	    var str = 'hello, world';
	
	    foo
	    (
	        function()
	        {
	            return str;
	        }
	    );
	}
	
	bar();

上面的代码中，callback被从bar函数的运行上下文中传递给了foo函数，该callback函数返回变量str的值。

不过在这里请注意，变量str是声明在bar函数中的，也就是说这个变量仅存于bar函数中。

而callback是在另外一个不同的函数中被执行的(跟变量str不在一起)，我们这是可能会猜测foo函数中什么也不会显示出来。

但是，在这里引入了闭包性这个概念。

也就是说在不同的函数中(运行上下文中)，一个函数可以访问到变量所声明上下文中的内容。

因此上面的代码中，callback可以访问到str变量——即使这个callback所在的foo函数不能直接访问这个str变量。
 
##2、Objective-C中的实现
实际上闭包性和匿名函数在Objective-C中是可以使用的，只不过Objective-C是构建于C语言之上，属于强类型编译语言，所以跟上面介绍的解释性脚本语言有许多不同之处。

注意：block其实在纯C或C++(以及Objective-C++)中都是可用的。

在标准C函数中，定义block(匿名函数)之前需要先声明原型。

block的语法有一点点棘手，不过要是熟悉函数指针的话，就非常容易理解了。
下面是block的原型：
	
NSString * ( ^ myBlock )( int );

上面的代码是声明了一个block(^)原型，名字就叫做myBlock，携带一个int参数，返回只为NSString类型的指针。

下面来看看block的定义：

	myBlock = ^( int number )
	{
	    return [ NSString stringWithFormat: @"Passed number: %i", number ];
	};

如上所示，将一个函数体赋值给了myBlock变量，其接收一个名为number的参数。该函数返回一个NSString对象。

注意：不要忘记block后面的分号。

在脚本语言中是可以忽略掉分号的，但是在编译性语言(如Objective-C)是必须有的。

如果没有写这个分号，编译器会产生一个错误，当然也不会生成可执行文件。
定义好block之后，就可以像使用标准函数一样使用它了：
	myBlock(5);

下面是完整的Objective-C程序源代码：

	#import <Cocoa/Cocoa.h>

	int main( void )
	{
	    NSAutoreleasePool * pool;
	    NSString * ( ^ myBlock )( int );
	
	    pool    = [ [ NSAutoreleasePool alloc ] init ];
	    myBlock = ^( int number )
	    {
	        return [ NSString stringWithFormat: @"Passed number: %i", number ];
	    };
	
	    NSLog( @"%@", myBlock(5) );
	
	    [ pool release ];
	
	    return EXIT_SUCCESS;
	}

我们可以用下面的命令来编译(在Terminal中)：
	
	gcc -Wall -framework Cocoa -o test test.m

上面的命令会根据test.m源代码文件生成一个名为name的可执行文件。可以用下面的命令来运行这个可执行文件：
	./test
如果不把block赋值给变量的话，可以忽略掉block原型的声明，例如直接将block当做参数进行传递。如下所示：

	someFunction( ^ NSString * ( void ) { return @"hello, world" } );

注意，上面这种情况必须声明返回值的类型——这里是返回NSString对象。

###2.1将block当做参数来传递

之前说过了，block可以当做参数传递给某个C函数。

如下所示：

	void logBlock( NSString * ( ^ theBlock )( int ) )
	{
	    NSLog( @"Block returned: %@", theBlock() );
	}

由于Objective-C是强制类型语言，所以作为函数参数的block也必须要指定返回值的类型，以及相关参数类型(如果需要的话)。

其实在Objective-C方法中也是一样的：

	- ( void )logBlock: ( NSString * ( ^ )( int ) )theBlock;
	
###2.2闭包性
之前有说过，闭包性在Objective-C中是可用的，只不过其行为跟解释性语言有所不同罢了。

我们来看看下面的程序：

	#import <Cocoa/Cocoa.h>
	
	void logBlock( int ( ^ theBlock )( void ) )
	{
	    NSLog( @"Closure var X: %i", theBlock() );
	}
	
	int main( void )
	{
	    NSAutoreleasePool * pool;
	    int ( ^ myBlock )( void );
	    int x;
	
	    pool = [ [ NSAutoreleasePool alloc ] init ];
	    x    = 42;
	
	    myBlock = ^( void )
	    {
	        return x;
	    };
	
	    logBlock( myBlock );

	    [ pool release ];
	
	    return EXIT_SUCCESS;
	}

上面的代码在main函数中声明了一个整型，并赋值42，另外还声明了一个block，该block会将42返回。

然后将block传递给logBlock函数，该函数会显示出返回的值42。

即使是在函数logBlock中执行block，而block又声明在main函数中，但是block仍然可以访问到x变量，并将这个值返回。

注意：block同样可以访问全局变量，即使是static。

下面来看看第一点不同之处：通过block进行闭包的变量是const的。也就是说不能在block中直接修改这些变量。

来看看当block试着增加x的值时，会发生什么：

	myBlock = ^( void )
	{
	    x++
	
	    return x;
	};

编译器会生成一个错误：大概意思是在block中x变量是只读的。

不过也别担心，为了允许在block中修改变量，也是可以做到的：用__block关键字来声明变量即可。

基于之前的代码，给x变量添加__block关键字，如下：
	
	__block int x;

###2.3内存管理
从C语言的角度来看，实际上block是一个结构体，可以被拷贝和销毁的。有两个函数可以使用：Block_copy和Block_destroy()。

而在Objective-C中，block可以接收retain、release和copie消息，这就跟普通对象一样。如果一个block需要被存储下来供以后使用，这些消息是非常重要的(例如，将block存储到一个类的实例变量中)。例如，为了避免错误的使用block，对block进行retain是非常有必要的。

###2.4示例
Block可以用在许多不同的环境中，这样可以让代码更加简单，以及减少函数声明的数量。

下面有一个实例：

我们将给NSArrary类添加一个static方法(类方法)，该方法通过一个callback，根据另外一个数组中的内容产生一个新的数组。

在PHP程序员眼里，该方法就如一个array_filter()。

首先，需要为NSArray类声明一个category。（通过category可以给已有的类添加新方法）。

	@interface NSArray( BlockExample )
	
	+ ( NSArray * )arrayByFilteringArray: ( NSArray * )source withCallback: ( BOOL ( ^ )( id ) )callback;
	
	@end

上面，声明了一个方法，该方法返回一个NSArray对象，另外接收两个参数：NSArray对象，以及一个callback (为block)。

在callback中会判断根据传入数组参数中的每一个元素。并将返回一个boolean值，以确定当前array中的元素是否需要存储到返回的数组中。

block只有一个参数，代表数组中的某个元素。

我们来看看该方法的具体实现：
	
	@implementation NSArray( BlockExample )
	
	+ ( NSArray * )arrayByFilteringArray: ( NSArray * )source withCallback: ( BOOL ( ^ )( id ) )callback
	{
    	NSMutableArray * result;
    	id               element;
	
    	result = [ NSMutableArray arrayWithCapacity: [ source count ] ];
	
    	for( element in source ) {
	
    	    if( callback( element ) == YES ) {
	
    	        [ result addObject: element ];
    	    }
    	}
	
    	return result;
	}
	
	@end

上面的代码中，首先是创建了一个可以动态改变尺寸的数组：NSMutableArray，然后根据source array的数目来初始化该数组。

然后对source array中的每个元素进行迭代， 如果callback返回值为YES的话，就将该元素添加到result数组中。

下面是使用该方法的一个完整示例：利用callback构建一个数组：该数组中只包含source array中为NSString类型的元素：

	#import <Cocoa/Cocoa.h>

	@interface NSArray( BlockExample )
	
	+ ( NSArray * )arrayByFilteringArray: ( NSArray * )source withCallback: ( BOOL ( ^ )( id ) )callback;
	
	@end
	
	@implementation NSArray( BlockExample )
	
	+ ( NSArray * )arrayByFilteringArray: ( NSArray * )source withCallback: ( BOOL ( ^ )( id ) )callback
	{
	    NSMutableArray * result;
	    id               element;
	
	    result = [ NSMutableArray arrayWithCapacity: [ source count ] ];
	
	    for( element in source ) {
	
	        if( callback( element ) == YES ) {
	
	            [ result addObject: element ];
	        }
	    }
	
	    return result;
	}
	
	@end
	
	int main( void )
	{
	    NSAutoreleasePool * pool;
	    NSArray           * array1;
	    NSArray           * array2;
	
	    pool   = [ [ NSAutoreleasePool alloc ] init ];
	    array1 = [ NSArray arrayWithObjects: @"hello, world	[ NSDate date ], @"hello, universe", nil ];
   	 	array2 = [ NSArray arrayByFilteringArray: array1
                    withCallback:          ^ BOOL ( id element )
                    {
                        return [ element isKindOfClass: [ NSString class ] ];
                    }
             ];

   		 NSLog( @"%@", array2 );

	    [ pool release ];
	
	    return EXIT_SUCCESS;
	}



转载：[破船之家](http://beyondvincent.com/blog/2013/07/09/99/)
