---
layout: post
title: block 相关
---

{{ page.title }}
================
<p class="meta">4 August 2014 - tony cap</p>

* 目录：
```
1.Block简介
2.Block的创建
3.不带参数的Block
4.Block的闭包性(closure)
5.修改非局部变量
6.Block作为函数的参数
7.定义Block类型
8.总结
延伸阅读
```

* 正文

## Block简介
Apple 在C, Objective-C, C++加上Block這個延申用法。目前只有Mac 10.6 和iOS 4有支援。Block是由一堆可執行的程式組成，也可以稱做沒有名字的Function (Anonymous function)。

我们可以把Block当做Objective-C的匿名函数。Block允许开发者在两个对象之间将任意的语句当做数据进行传递，往往这要比引用定义在别处的函数直观。另外，block的实现具有封闭性(closure)，而又能够很容易获取上下文的相关状态信息。

## Block的创建
实际上，block使用了与函数相同的机制：可以像声明函数一样，来声明一个bock变量；可以利用定义一个函数的方法来定义一个block；也可以将block当做一个函数来调用。

```
// main.m 
#import <Foundation/Foundation.h> 
  
int main(int argc, const char * argv[]) { 
    @autoreleasepool { 
        // Declare the block variable 
        double (^distanceFromRateAndTime)(double rate, double time); 
  
        // Create and assign the block 
        distanceFromRateAndTime = ^double(double rate, double time) { 
            return rate * time; 
        }; 
        // Call the block 
        double dx = distanceFromRateAndTime(35, 1.5); 
  
        NSLog(@"A car driving 35 mph will travel " 
              @"%.2f miles in 1.5 hours.", dx); 
    } 
    return 0; 
} 
```
在上面的代码中，利用插入符(^)将distanceFromRateAndTime变量标记为一个block。就像声明函数一样，需要包含返回值的类型，以及参数的类型，这样编译器才能安全的进行强制类型转换。插入符(^)跟指针(例如 int *aPointer)前面的星号(*)类似——只是在声明的时候需要使用，之后用法跟普通的变量一样。
 
block的定义本质上跟函数一样——只不过不需要函数名。block以签名字符串开始：^double(double rate, double time)标示返回一个double，以及接收两个同样为double的参数(如果不需要返回值，可以忽略掉)。在签名后面是一个大括弧({})，在这个括弧里面可以编写任意的语句代码，这跟普通的函数一样。
 
当把block赋值给distanceFromRateAndTime后，我们就可以像调用函数一样调用这个变量了。

###不带参数的Block
如果block不需要任何的参数，那么可以忽略掉参数列表。另外，在定义block的时候，返回值的类型也是可选的，所以这样情况下，block可以简写为^ { … }：
```
double (^randomPercent)(void) = ^ { 
    return (double)arc4random() / 4294967295; 
}; 
NSLog(@"Gas tank is %.1f%% full", 
      randomPercent() * 100); 
 ```
 
在上面的代码中，利用内置的arc4random()方法返回一个32位的整型随机数——为了获得0-1之间的一个值，通过除以arc4random()方法能够获取到的最大值(4294967295)。
 
到现在为止，block看起来可能有点像利用一种复杂的方式来定义一个方法。事实上，block是被设计为闭包的(closure)——这就提供了一种新的、令人兴奋的编程方式。
 
###Block的闭包性(closure)
在block内部，可以像普通函数一样访问数据：局部变量、传递给block的参数，全局变量/函数。并且由于block具有闭包性，所以还能访问非局部变量(non-local variable)。非局部变量定义在block之外，但是在block内部有它的作用域。例如，getFullCarName可以使用定义在block前面的make变量：
```
NSString *make = @"Honda"; 
NSString *(^getFullCarName)(NSString *) = ^(NSString *model) { 
    return [make stringByAppendingFormat:@" %@", model]; 
}; 
NSLog(@"%@", getFullCarName(@"Accord"));    // Honda Accord 
 ```
 
非局部变量会以const变量被拷贝并存储到block中，也就是说block对其是只读的。如果尝试在block内部给make变量赋值，会抛出编译器错误。

以const拷贝的方式访问非局部变量，意味着block实际上并不是真正的访问了非局部变量——只不过在block中创建了非局部变量的一个快照。当定义block时，无论非局部变量的值是什么，都将被冻结，并且block会一直使用这个值，即使在之后的代码中修改了非局部变量的值。下面通过代码来看看，在创建好block之后，修改make变量的值，会发生什么：
```
NSString *make = @"Honda"; 
NSString *(^getFullCarName)(NSString *) = ^(NSString *model) { 
    return [make stringByAppendingFormat:@" %@", model]; 
}; 
NSLog(@"%@", getFullCarName(@"Accord"));    // Honda Accord 
  
// Try changing the non-local variable (it won't change the block) 
make = @"Porsche"; 
NSLog(@"%@", getFullCarName(@"911 Turbo")); // Honda 911 Turbo 
 ```
 
block的闭包性为block与上下文交互的时候带来极大的便利性，当block需要额外的数据时，可以避免使用参数——只需要简单的使用非局部变量即可。
 
### 修改非局部变量
冻结中的非局部变量是一个常量值，这也是一种默认的安全行为——因为这可以防止在block中的代码对非局部变量做了意外的修改。那么如果我们希望在block中对非局部变量值进行修改要如何做呢——用__block存储修饰符(storage modifier)来声明非局部变量：
__block NSString *make = @"Honda"; 
这将告诉block对非局部变量做引用处理，在block外部make变量和内部的make变量创建一个直接的链接(direct link)。现在就可以在block外部修改make，然后反应到block内部，反过来，也是一样。
 

通过引用的方式访问非局部变量
这跟普通函数中的静态局部变量(static local variable)类似，用__block修饰符声明的变量可以记录着block多次调用的结果。例如下面的代码创建了一个block，在block中对i进行累加。
```
__block int i = 0; 
int (^count)(void) = ^ { 
    i += 1; 
    return i; 
}; 
NSLog(@"%d", count());    // 1 
NSLog(@"%d", count());    // 2 
NSLog(@"%d", count());    // 3 
 ```
Block作为函数的参数
把block存储在变量中有时候非常有用，比如将其用作函数的参数。这可以解决类似函数指针能解决的问题，不过我们也可以定义内联的block，这样代码更加易读。
 
例如下面Car interface中声明了一个方法，该方法用来计算汽车的里程数。这里并没有强制要求调用者给该方法传递一个常量速度，相反可以改方法接收一个block——该block根据具体的时间来定义汽车的速度。
```
// Car.h 
#import <Foundation/Foundation.h> 
  
@interface Car : NSObject 
  
@property double odometer; 
  
- (void)driveForDuration:(double)duration 
       withVariableSpeed:(double (^)(double time))speedFunction 
                   steps:(int)numSteps; 
  
@end 
 ```

上面代码中block的数据类型是double (^)(double time)，也就是说block的调用者需要传递一个double类型的参数，并且该block的返回值为double类型。注意：上面代码中的语法基本与本文开头介绍的block变量声明相同，只不过没有变量名字。
 
在函数的实现里面可以通过speedFunction来调用block。下面的示例通过算法计算出汽车行驶的大约距离。其中steps参数是由调用者确定的一个准确值。
```
// Car.m 
#import "Car.h" 
  
@implementation Car 
  
@synthesize odometer = _odometer; 
  
- (void)driveForDuration:(double)duration 
       withVariableSpeed:(double (^)(double time))speedFunction 
                   steps:(int)numSteps { 
    double dt = duration / numSteps; 
    for (int i=1; i<=numSteps; i++) { 
        _odometer += speedFunction(i*dt) * dt; 
    } 
} 
  
@end 
```

在下面的代码中，有一个main函数，在main函数中block定义在另一个函数的调用过程中。虽然理解其中的语法需要话几秒钟时间，不过这比起另外声明一个函数，再定义withVariableSpeed参数要更加直观。
```
// main.m 
#import <Foundation/Foundation.h> 
#import "Car.h" 
  
int main(int argc, const char * argv[]) { 
    @autoreleasepool { 
        Car *theCar = [[Car alloc] init]; 
  
        // Drive for awhile with constant speed of 5.0 m/s 
        [theCar driveForDuration:10.0 
               withVariableSpeed:^(double time) { 
                           return 5.0; 
                       } steps:100]; 
        NSLog(@"The car has now driven %.2f meters", theCar.odometer); 
  
        // Start accelerating at a rate of 1.0 m/s^2 
        [theCar driveForDuration:10.0 
               withVariableSpeed:^(double time) { 
                           return time + 5.0; 
                       } steps:100]; 
        NSLog(@"The car has now driven %.2f meters", theCar.odometer); 
    } 
    return 0; 
} 
 ```
上面利用一个简单的示例演示了block的通用性。在iOS的SDK中有许多API都利用了block的其它一些功能。NSArray的sortedArrayUsingComparator:方法可以使用一个block对元素进行排序，而UIView的animateWithDuration:animations:方法使用了一个block来定义动画的最终状态。此外，block在并发编程中具有强大的作用。
 
###定义Block类型
由于block数据类型的语法会很快把函数的声明搞得难以阅读，所以经常使用typedef对block的签名(signature)做处理。例如，下面的代码创建了一个叫做SpeedFunction的新类型，这样我们就可以对withVariableSpeed参数使用一个更加有语义的数据类型。
```
// Car.h 
#import <Foundation/Foundation.h> 
  
// Define a new type for the block 
typedef double (^SpeedFunction)(double); 
  
@interface Car : NSObject 
  
@property double odometer; 
  
- (void)driveForDuration:(double)duration 
       withVariableSpeed:(SpeedFunction)speedFunction 
                   steps:(int)numSteps; 
  
@end 
 ```
许多标准的Objective-C框架也使用了这样的技巧，例如NSComparator。
 
###总结
Block不仅提供了C函数同样的功能，而且block看起来更加直观。block可以定义为内联(inline)，这样在函数内部调用的时候就非常方便，由于block具有闭包性(closure)，所以block可以很容易获得上下文信息，而又不会对这些数据产生负面影响。
 
延伸阅读
```
[A look inside blocks: Episode 1](http://www.galloway.me.uk/2012/10/a-look-inside-blocks-episode-1/)
[A look inside blocks: Episode 2](http://www.galloway.me.uk/2012/10/a-look-inside-blocks-episode-2/)
[A look inside blocks: Episode 3 (Block_copy)](http://www.galloway.me.uk/2013/05/a-look-inside-blocks-episode-3-block-copy/)
[Closure and anonymous functions in Objective-C](http://www.xs-labs.com/en/archives/articles/objc-blocks/)
```
来源：破船的博客










