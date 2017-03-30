---
title: AFNetworking内存泄露的问题
date: 2017-03-30 11:06:22
---
# 关于使用AFNetworking内存泄露的问题

这几天做完项目于是用instruments里面的Leaks检查了下内存情况,结果发现了一些AF里面的坑，AF Issues上面也有[哥们]("https://github.com/AFNetworking/AFNetworking/issues/3293")提出问题来，不过也并没有解释清楚，所以特意自己重新看了遍源码整理了下.

下面是我报内存泄露的部分代码

```
- (instancetype)init
{
    self = [super init];
    if (self) {
        self.manager=[AFHTTPSessionManager manager];
        self.manager.requestSerializer=[AFJSONRequestSerializer serializer];
        self.manager.requestSerializer.timeoutInterval=30;
        self.manager.responseSerializer=[AFJSONResponseSerializer serializer];
        self.manager.responseSerializer.acceptableContentTypes = [NSSet setWithObject:@"application/json"];
    }
    return self;
}

```

这里的manager方法并不是单例方法，也就是说一个manager对象会对应一个或多个NSURLSession对象，这样创建会造成内存增长，但是这样应该并不会造成内存泄露才对，况且我在dealloc方法里面已经把manager置空，于是我查找苹果官方文档，找到如下信息


Important

The session object keeps a strong reference to the delegate until your app exits or explicitly invalidates the session. If you do not invalidate the session, your app leaks memory until it exits.

大意如下：

会话对象保持对代理的强烈引用，直到您的应用程序退出或显式使会话无效。 如果您不使会话无效，您的应用程序会在内存溢出之前泄漏内存。

然后我在AF的源码里面的initWithSessionConfiguration方法找到如下代码

```

@property (readwrite, nonatomic, strong) NSURLSession *session;

self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

```

那么答案就出来了session被AFNetworkReachabilityManager对象强引用了，session的委托又强持有了AFNetworkReachabilityManager对象，所以导致AFNetworkReachabilityManager对象置空后session依然存在，从而导致内存泄漏了

所以，正确的用法应该在AF上面再封装多一层把manager写成单例，降低内存峰值，代码如下：

```
+ (AFHTTPSessionManager *)sharedManager {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager=[AFHTTPSessionManager manager];
        manager.requestSerializer=[AFJSONRequestSerializer serializer];
        manager.requestSerializer.timeoutInterval=30;
        manager.responseSerializer=[AFJSONResponseSerializer serializer];
        manager.responseSerializer.acceptableContentTypes = [NSSet setWithObject:@"application/json"];
    });
    return manager;
}

```

使用则直接[封装类名 sharedManager] POST如此使用即可.