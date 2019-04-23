---
title: 如何优雅地在Swift4中实现Method Swizzling
author: Yao Li
author_site: http://blog.yaoli.site
---

> We can change the world and make it a better place. 
It is in your hands to make a difference.
<br>
我们可以改变世界，让它成为一个更好的地方。
有所作为的关键就在你手里。
<br>
**——Nelson Rolihlahla Mandela 纳尔逊·罗利赫拉赫拉·曼德拉**

**了解Objective-C runtime的人应该都或多或少的知道method swizzling这个黑魔法。这个技术可以让我们在运行时改变selector的实现，从而可以达到一些“不可告人”的目的。这可能是runtime机制中最有争议的一项技术。这里就不过多对这个技术作什么评价，本文只是探讨如何在Swift中更好地使用这个技术。**

首先来看一段[**Mattt Thompson**](http://nshipster.com)大神（AFNetworking作者之一）的实现

```objective-c
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(viewWillAppear:);
        SEL swizzledSelector = @selector(xxx_viewWillAppear:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

#pragma mark - Method Swizzling

- (void)xxx_viewWillAppear:(BOOL)animated {
    [self xxx_viewWillAppear:animated];
    NSLog(@"viewWillAppear: %@", self);
}

@end
```
Mattt Thompson大神在他的一篇文章[**Method Swizzling**](http://nshipster.com/method-swizzling/)中提到method swizzling使用的几个要点，其中有2个要点的实现是我们在转换到Swift中可能遇到困难的地方：
> 
- swizzling 应该只在 dispatch_once 中完成
- swizzling应该只在+load中完成。

## dispatch_once
在Swift3.x的Dispatch API中我们发现`dispatch_once`已经没有了，但是我们知道`static let`这样声明的变量其实已经用到`dispatch_once`了，比如
```swift
class SingletonClass  {
    static let sharedInstance = SingletonClass()
}
```
这是Swift中单例的实现，在这里我们并有看到`dispatch_once`的出现，但是let定义的属性本身就是thread safe的。那么如果不想定义变量，只想保证某一段代码只执行一次该怎么做呢。上代码：
```swift
class MyClass  {
    static let doOneTime: Void = {
        print("Just do time")
    }()
}
```
直接定义一个`Void`的属性，看上去后面跟着一个在初始化中会立刻执行的闭包，但是其实这是懒加载的，只有当我们调用`MyClass.justAOneTimeThing`时闭包中的代码才会执行。当然有些可能会觉得通过调用一个参数来执行一些代码有些怪，我们可以在外面再包一层函数，就像这样：
```swift
class MyClass  {
    private static let doOneTime: Void = {
        print("Just do time")
    }()
    
    static func doOneTimeFunction（）{
        doOneTime
    }
}
```
这样我们就可以这样调用了`MyClass.doOneTimeFunction()`，至此我们找到了解决`dispatch_once`的方案了。有关`dispatch_once`在Swift中的替代方案可以[参阅这里](https://stackoverflow.com/questions/37801407/whither-dispatch-once-in-swift-3)

## +load
在OC中我们都知道`+load()`方法会在类被装载时调用，而Swift中已经没有了该方法。好在我们在Swift中还有方法`+initialize()`方法可以用，这个方法会在第一次调用这个类的类方法或者实例方法时调用。
但是到了Swift3.1，有的朋友可能发现`+initialize()`这个方法已经不被苹果所推荐使用并且在未来会被弃用。更新到Xcode9的朋友可能发现了`+initialize()`这个方法果然被禁用了。。。
> Method 'initialize()' defines Objective-C class method 'initialize', which is not permitted by Swift

有一种通常的解法是放到`AppDelegate`的`applicationDidFinishLaunching`中，我们只需添加一个我们希望初始化的静态函数就可以了。但是这的方法不够灵活，而且存在一种情况，如果我们想build的一个`framework`，但是我们又不希望让使用者还要在`applicationDidFinishLaunching`中调用。
<br>
于是我又开始在网上寻找解决方案，最终[**JORDAN SMITH**的文章**Handling the Deprecation of initialize()**](http://jordansmith.io/handling-the-deprecation-of-initialize/)给出了一个解决方案.

```swift
protocol SelfAware: class {
    static func awake()
}

class NothingToSeeHere {
    static func harmlessFunction() {
        let typeCount = Int(objc_getClassList(nil, 0))
        let types = UnsafeMutablePointer<AnyClass>.allocate(capacity: typeCount)
        let autoreleasingTypes = AutoreleasingUnsafeMutablePointer<AnyClass>(types)
        objc_getClassList(autoreleasingTypes, Int32(typeCount))
        for index in 0 ..< typeCount {
            (types[index] as? SelfAware.Type)?.awake()
        }
        types.deallocate(capacity: typeCount)
    }
}

extension UIApplication {
    private static let runOnce: Void = {
        NothingToSeeHere.harmlessFunction()
    }()
    
    override open var next: UIResponder? {
        // Called before applicationDidFinishLaunching
        UIApplication.runOnce
        return super.next
    }
}
```
[**JORDAN SMITH**](http://jordansmith.io)想法其实很简单，是通过runtime获取到所有类的列表，然后向所有遵循`SelfAware`协议的类发送消息，并且他把这些操作放到了UIApplication的next属性的调用中，同时发现了`next`属性会在`applicationDidFinishLaunching`之前被调用。
<br>
至此我们也还算优雅地解决了`+load`的问题。

***

## 最后
看一下如何对`UIViewController`的`viewWillAppear`方法做method swizzling：
```swift
import UIKit

class ViewController: UIViewController {
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        print("viewWillAppear")
    }

    override func viewDidLoad() {
        super.viewDidLoad()
    }

}

extension UIViewController: SelfAware {
    static func awake() {
        UIViewController.classInit()
    }
    
    static func classInit() {
        swizzleMethod
    }
    
    @objc func swizzled_viewWillAppear(_ animated: Bool) {
        swizzled_viewWillAppear(animated)
        print("swizzled_viewWillAppear")
    }
    
    private static let swizzleMethod: Void = {
        let originalSelector = #selector(viewWillAppear(_:))
        let swizzledSelector = #selector(swizzled_viewWillAppear(_:))
        swizzlingForClass(UIViewController.self, originalSelector: originalSelector, swizzledSelector: swizzledSelector)
    }()
    
    private static func swizzlingForClass(_ forClass: AnyClass, originalSelector: Selector, swizzledSelector: Selector) {
        let originalMethod = class_getInstanceMethod(forClass, originalSelector)
        let swizzledMethod = class_getInstanceMethod(forClass, swizzledSelector)
        
        guard (originalMethod != nil && swizzledMethod != nil) else {
            return
        }
        
        if class_addMethod(forClass, originalSelector, method_getImplementation(swizzledMethod!), method_getTypeEncoding(swizzledMethod!)) {
            class_replaceMethod(forClass, swizzledSelector, method_getImplementation(originalMethod!), method_getTypeEncoding(originalMethod!))
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod)
        }
    }
    
}
```
<br>
在`ViewController`加载显示出来后的log是这样的
>swizzled_viewWillAppear
viewWillAppear

<br>
这里需要说一下的是`awake()`这个方法在上面这个例子里会被调用很多次，因为其实系统有很多继承自`UIViewController`的子类，但由于我们的`swizzleMethod`是一个线程安全的静态变量，所以该swizzling方法仅会被调用一次。
另外还发现一个问题，尽管有了`extension UIViewController: SelfAware`，在`harmlessFunction()`调用的时候，发现`UIViewController`并没有符合`SelfAware`，但是`UIViewController`的所有子类却都符合`SelfAware`。
虽然这样没啥大问题，还是欢迎大家提出改进的方案。


### 参考文章
>[Method Swizzling](http://nshipster.com/method-swizzling/) by Mattt Thompson

>[Handling the Deprecation of initialize()](http://jordansmith.io/handling-the-deprecation-of-initialize/) by JORDAN SMITH













