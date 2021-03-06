## 如何监听实例化对象什么时候释放

先说下这个知识点，因为在接下来的好几个地方都会用到，会有一些异常的情况，所以需要一种知道当前创建者啥时候释放，首先会想到dealloc,这样会Hook的NSObject,在一定程度会影响性能，后面发现一种比较优雅的方法,原理来自于Runtime源码:
```
/***********************************************************************
* objc_destructInstance
* Destroys an instance without freeing memory. 
* Calls C++ destructors.
* Calls ARR ivar cleanup.
* Removes associative references.
* Returns `obj`. Does nothing if `obj` is nil.
* Be warned that GC DOES NOT CALL THIS. If you edit this, also edit finalize.
* CoreFoundation and other clients do call this under GC.
**********************************************************************/
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = !UseGC && obj->hasAssociatedObjects();
        bool dealloc = !UseGC;

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        if (dealloc) obj->clearDeallocating();
    }

    return obj;
}
```

`_object_remove_assocations`会释放所有的用AssociatedObject数据。


`objc_setAssociatedObject`给当前对象添加一个中间对象，当前对象释放时，会清理AssociatedObject数据，AssociatedObject的中间对象将被清理释放，中间对象的dealloc方法将被执行。

最终清理被遗漏的监听者,会用在KVO和NSNotification清理没用的监听者,不过这种方式有以下问题需要注意:
* 清理的时候线程安全问题
* 清理时机偏晚，是否适合你当前的情况


## Unrecognized Selector Sent to Instance
由于Objective-c是Message机制，而且对象在转换的时候，会有拿到的对象和预期不一致，所以会有方法找不到的情况，在找不到方法时，查找方法将会进入方法Forward流程,系统给了三次补救的机会，所以我们要解决这个问题，在这三次均可以解决这个问题

![forward](https://upload-images.jianshu.io/upload_images/1654054-5e5737afb54d4654.png)

* resolveInstanceMethod:(SEL)sel
这是实例化方法没有找到方法，最先执行的函数，首先会流转到这里来，返回值是BOOL,没有找到就是NO,找到就返回YES,如果要解决就需要再当前的实例中加入不存在的Selector,并绑定IMP，示例如下:
```objc
static void xxxInstanceName(id self, SEL cmd, id value) {
    NSLog(@"resolveInstanceMethod %@", value);
}

+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    NSLog(@"resolveInstanceMethod");
    
    NSMethodSignature* sign = [self methodSignatureForSelector:selector];
    if (!sign) {
        class_addMethod([self class], sel, (IMP)xxxInstanceName, "v@:@");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

* forwardingTargetForSelector:(SEL)aSelector

如果resolveInstanceMethod没有处理，将进行到forwardingTargetForSelector这步来，这时候你可以返回nil，你也可以用一个Stub对象来接住，把消息流程流转到了你的Stub那边了，然后在你的Stub里添加不存在的Selector，这样就不会crash了，示例如下:
```objc
- (id)forwardingTargetForSelectorSwizzled:(SEL)selector{
    NSMethodSignature* sign = [self methodSignatureForSelector:selector];
    if (!sign) {
        id stub = [[UnrecognizedSelectorHandle new] autorelease];
        class_addMethod([stub class], selector, (IMP)unrecognizedSelector, "v@:");
        return stub;
    }
    return [self forwardingTargetForSelectorSwizzled:selector];
}
```

* methodSignatureForSelector:(SEL)aSelector

* forwardInvocation:(NSInvocation \*)anInvocation


这两个方法一起说，因为他们之间有关联，
1. 当methodSignatureForSelector返回nil时，会Crash
2. 如果methodSignatureForSelector返回一个定义好的NSMethodSignature，但是没有实现forwardInvocation，也会闪退，如果实现了forwardInvocation，__会先返回到resolveInstanceMethod然后再才会到forwardInvocation__
3. 当流转到`forwardInvocation`,通过以下方法:
```
[anInvocation invokeWithTarget:xxxtarget1];
[anInvocation invokeWithTarget:xxxtarget2];
```
还可以流转到多个对象,[anInvocation invokeWithTarget:xxxtarget2]是为了让不存在的方法有着陆点

* doesNotRecognizeSelector:(SEL)aSelector
执行到这里的时候，两种情况:
1. 当methodSignatureForSelector返回一种任意的方法签名的时候，也会进入doesNotRecognizeSelector，但是不会闪退
2. 当methodSignatureForSelector返回nil时，进入doesNotRecognizeSelector就会闪退

根据以上流程，最终还是选择流程2,原因如下:
1. resolveInstanceMethod虽然可以解决问题，给不存在的方法增加到示例中去，会污染当前示例
2. forwardInvocation在三步中式最后一步，会导致流转的周期变长，而且会产生NSInvocation,性能不是最好的选择

__2018-10-7__

根据热心的网友提供的[Bug](https://github.com/jezzmemo/JJException/issues/9),在使用协议时，但是并没有具体实现，导致没有找到方法闪退。

原因就是`[self methodSignatureForSelector:selector]`返回了方法签名，不是nil,导致了最终没有找到方法，最终找到原因是如下：

1. 在执行协议的方法时，先去找到协议的方法，找到并返回，并未检查是否实现
2. 执行普通的方法，检查了是否实现

下面来看看为什么会这样，`methodSignatureForSelector`最终执行的方法是`CoreFoundation __methodDescriptionForSelector:`,大概的流程是这样的：

```
    0x10fa4f77e <+62>:  callq  0x10fb7428a   ; symbol stub for: class_copyProtocolList
    0x10fa4f7ab <+107>: callq  0x10fb742b4   ; symbol stub for: class_isMetaClass
    0x10fa4f7c0 <+128>: callq  0x10fb74788   ; symbol stub for: protocol_getMethodDescription
    0x10fa4f7d7 <+151>: callq  0x10fb742b4   ; symbol stub for: class_isMetaClass
    0x10fa4f7e9 <+169>: callq  0x10fb74788   ; symbol stub for: protocol_getMethodDescription
    0x10fa4f837 <+247>: callq  0x10fb7444c   ; symbol stub for: free
    0x10fa4f845 <+261>: callq  0x10fb742ae   ; symbol stub for: class_getSuperclass
    0x10fa4f85e <+286>: callq  0x10fb74296   ; symbol stub for: class_getInstanceMethod
    0x10fa4f86b <+299>: callq  0x10fb745d8   ; symbol stub for: method_getDescription
```

这里面我去掉了逻辑跳转，留下了关键的一些symbol，可以看出里面的一些关键动作，用图表示更直观:

![\_\_methodDescriptionForSelector](http://zorrochen.qiniudn.com/blog_RemakeMethodSignatureForSelector_2.png)

所以最终调整了找不到方法的处理方式，要Hook以下两个函数:

* methodSignatureForSelector:(SEL)aSelector
* forwardInvocation:(NSInvocation \*)anInvocation

会出现三种情况:
1. 正常有方法签名的，按照正常方法调用流程走
2. 没有方法签名的，没有实现，给出一个我们自定义的签名`v@:@`，并走到forwardInvocation方法记录错误
3. 有方法签名，但是没有实现，用自己的方法签名，并走到forwardInvocation方法记录错误


## NSArray,NSMutableArray,NSDictonary,NSMutableDictionary

* 类族(Class Cluster)

NSDictonary，NSArray,NSString等，都使用了类族，这种模式最大的好处就是，可以隐藏抽象基类背后的复杂细节，使用者只需调用基类简单的方法就可以返回不同的子类实例

* Swizzle Hook

这里就不赘述Swizzle概念了，Google到处都是讲解的，这里给一个典型的例子:
```
swizzleInstanceMethod(NSClassFromString(@"__NSArrayI"), @selector(objectAtIndex:), @selector(hookObjectAtIndex:));

- (id) hookObjectAtIndex:(NSUInteger)index {
    if (index < self.count) {
        return [self hookObjectAtIndex:index];
    }
    handleCrashException(@"HookObjectAtIndex invalid index");
    return nil;
}
```

## Zombie Pointer

让野指针不闪退是模仿了XCode debug的Zombie Object，也参考了网易和美团的做法,主要是以下步骤:

1. Hook住dealloc方法
2. 如果当前示例在黑名单里，就把当年前示例加入集合，并把当前对象`objc_destructInstance`清理引用关系，并未真正释放内存，并将`object_setClass`设置成自己的中间对象
3. Hook中间对象的方法，收到的消息都由中间对象来处理
4. 维护的野指针集合，要么根据个数来维护，要么根据总大小来维护，当满了，就需要真正释放对象内存`free(obj)`

存在的问题:

1. 需要单独的内存那些问题对象
2. 最后释放内存后，再访问时会闪退，这个方法只是一定程度延迟了闪退时间
3. 需要后台维护黑名单机制，来指定那些问题对象

## KVO

KVO在以下情况会导致闪退:
* 添加监听后没有清除会导致闪退
* 清除不存在的key也会闪退
* 添加重复的key导致闪退

需要Hook以下方法:
* addObserver:forKeyPath:options:context:
* removeObserver:forKeyPath:

主要解决以下问题:
* 在注册监听后,中间对象需要维护注册的数据集合，当宿主释放时，清除还在集合中的监听者
* 保护key不存在的情况
* 保护重复添加的情况

## NSTimer

NSTimer存在以下问题:

* Target是强引用，内存泄漏
* 在宿主不存在的时候，清理NSTimer

Hook以下方法:
* scheduledTimerWithTimeInterval:target:selector:userInfo:repeats

解决方法:
1.当repeats为NO时，走原始方法
2.当repeats为YES时，新建一个对象，声明一个target属性为weak类型，指向参数的target,当中间对象的target为空时，清理NSTimer

## NSNotification

NSNotification的主要问题是:
* 添加通知后，没有移除导致Crash的问题(不过在iOS9以后没有这个问题,我在真机8.3测试也没有这个问题，不知道iOS8是否有这个问题)

Hook以下方法:
* addObserver:selector:name:object:

原因和解决办法:
问题就在在于和assign和weak问题，野指针问题，要么置空指针或者删除空指针的集合

## MRC

这里单独说下，为什么工程选择了MRC，因为在Hook集合类型的时候，启动的时候就闪退了，Crash的地方在系统类里，Stack里显示在CF这层，这里只能猜测系统底层对ARC的支持不好导致的，后续改成MRC就没有问题，所以这个需要继续研究和追踪，如果有知道的同学记得告知我下

## 性能

本来是没有打算注意性能这个问题的，因为从Hook原理的角度来说，只是交换IMP的指向，时间复杂度来说，只是在系统级别上增加了几条逻辑判断指令，所以这个影响是极小的，基本可以忽略，我经过测试，循环1000000次，没有HOOK和HOOK相差0.0x秒的，所以减少Crash，来增加这么点时间复杂度来说，是值得的。

不过最后说一点，就是dealloc确实需要注意，因为这里存在集合的操作，所以要注意时间复杂度，dealloc执行的很频繁的，而且主线程和子线程都会涉及到，尤其是主线程一定注意，否则会影响到UI的体验。

## CallStackSymbols偏移问题

在记录出错信息上，调用栈信息会辅助帮我们快速定位问题，但是`[NSThread callStackSymbols]`的地址都是偏移过的，app在每次启动的时候都会ASLR（Address space layout randomization）,获取这个偏移代码如下:
```
uintptr_t get_slide_address(void) {
    uintptr_t vmaddr_slide = 0;
    for (uint32_t i = 0; i < _dyld_image_count(); i++) {
        const struct mach_header *header = _dyld_get_image_header(i);
        if (header->filetype == MH_EXECUTE) {
            vmaddr_slide = _dyld_get_image_vmaddr_slide(i);
            break;
        }
    }
    
    return (uintptr_t)vmaddr_slide;
}
```

给一段出错调用栈示例,xxxxx代表应用名称:
```
1 xxxxx 0x00000001007d20a4 xxxxx + 8134820
2 xxxxx 0x00000001007d190c xxxxx + 8132876
3 xxxxx 0x0000000100857e80 xxxxx + 8683136
4 xxxxx 0x000000010089b958 xxxxx + 8960344
```

真正的查询地址:__真正地址 =  0x00000001007d20a4 - get_slide_address__

接下来使用ATOS来查询Symbol:
```
atos -arch arm64 -o xxxxx.dSYM/Contents/Resources/DWARF/xxxxx 真正地址
```

还有一种方法是获取Base address:
```
uintptr_t get_load_address(void) {
    const struct mach_header *exe_header = NULL;
    for (uint32_t i = 0; i < _dyld_image_count(); i++) {
        const struct mach_header *header = _dyld_get_image_header(i);
        if (header->filetype == MH_EXECUTE) {
            exe_header = header;
            break;
        }
    }
    return (uintptr_t)exe_header;
}
```

__注意是有-l参数的__

```
atos -arch arm64 -o xxxxx.dSYM/Contents/Resources/DWARF/xxxxx -l get_load_address 0x00000001007d20a4
```


## 参考资料

[https://github.com/opensource-apple/objc4/blob/master/runtime/objc-runtime-new.mm](https://github.com/opensource-apple/objc4/blob/master/runtime/objc-runtime-new.mm)
[大白健康系统](https://neyoufan.github.io/2017/01/13/ios/BayMax_HTSafetyGuard/)
[反编译分析并模拟实现methodSignatureForSelector方法](http://tutuge.me/2017/04/08/diy-methodSignatureForSelector/)
