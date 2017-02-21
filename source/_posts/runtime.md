---
title: runtime笔记
date: 2017-02-11 14:06:22
---

### 获取列表

<!-- more -->

```
-(void)runtimeLearn{
    unsigned int count;
    //获取属性列表
    objc_property_t *propertyList = class_copyPropertyList([clsDogName class], &count);
    for (unsigned int i=0; i<count; i++) {
        const char *propertyName = property_getName(propertyList[i]);
        NSLog(@"property---->%@", [NSString stringWithUTF8String:propertyName]);
    }
    
    //获取方法列表
    Method *methodList = class_copyMethodList([clsDogName class], &count);
    for (unsigned int i; i<count; i++) {
        Method method = methodList[i];
        NSLog(@"method---->%@", NSStringFromSelector(method_getName(method)));
    }
    
    //获取成员变量列表
    Ivar *ivarList = class_copyIvarList([clsDogName class], &count);
    for (unsigned int i; i<count; i++) {
        Ivar myIvar = ivarList[i];
        const char *ivarName = ivar_getName(myIvar);
        NSLog(@"Ivar---->%@", [NSString stringWithUTF8String:ivarName]);
    }
    
    //获取协议列表
    __unsafe_unretained Protocol **protocolList = class_copyProtocolList([clsDogName class], &count);
    for (unsigned int i; i<count; i++) {
        Protocol *myProtocal = protocolList[i];
        const char *protocolName = protocol_getName(myProtocal);
        NSLog(@"protocol---->%@", [NSString stringWithUTF8String:protocolName]);
    }
}
```
解析各种奇怪数据的时候可以使用这些方法获取特定的方法名字或者属性名字来完成各种需求,包括MJExtension解析的主要实现方法都有使用到。

### 方法交换
```
Method originalMethod=class_getInstanceMethod([NSString class], @selector(lowercaseString));
Method swappedMethod=class_getInstanceMethod([NSString class], @selector(eoc_myLowercaseString));
method_exchangeImplementations(originalMethod, swappedMethod);
```
替换两个方法的实现达到输出测试结果的目的(作用范围,整个运行期),或者如AFNetworking中的替换掉NSUrlSession中的resume和suspend方法为自身逻辑服务.

### 关联对象方法的使用
```
-(void)loadAlertView{
    UIAlertView *alert=[[UIAlertView alloc] initWithTitle:@"Question" message:@"2333" delegate:self cancelButtonTitle:@"Cancel" otherButtonTitles:@"Submit", nil];
    void (^block)(NSInteger) = ^(NSInteger btnIndex){
        if (btnIndex==0) {
            NSLog(@"0000");
        }else{
            NSLog(@"other");
        }
    };
    objc_setAssociatedObject(alert, EOCMYAlertViewKey, block, OBJC_ASSOCIATION_COPY);
    [alert show];
}

#pragma mark - UIAlertViewDelegate
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex NS_DEPRECATED_IOS(2_0, 9_0){
    void (^block)(NSInteger)=objc_getAssociatedObject(alertView, EOCMYAlertViewKey);
    block(buttonIndex);
}
```
使用关联对象方法可以把逻辑集中写在定义控件的地方使得逻辑不会过于分散.

### 拦截调用

程序执行中，如果没有找到方法就会转向拦截调用。拦截调用就是，在找不到调用的方法程序崩溃之前，你有机会通过重写NSObject的四个方法来处理。

```
+ (BOOL)resolveClassMethod:(SEL)sel;
+ (BOOL)resolveInstanceMethod:(SEL)sel;
//后两个方法需要转发到其他的类处理
- (id)forwardingTargetForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```
* 第一个方法是当你调用一个不存在的类方法的时候，会调用这个方法，默认返回NO，你可以加上自己的处理然后返回YES。
* 第二个方法和第一个方法相似，只不过处理的是实例方法。
* 第三个方法是将你调用的不存在的方法重定向到一个其他声明了这个方法的类，只需要你返回一个有这个方法的target。
* 第四个方法是将你调用的不存在的方法打包成NSInvocation传给你。做完你自己的处理后，调用invokeWithTarget:方法让某个target触发这个方法。