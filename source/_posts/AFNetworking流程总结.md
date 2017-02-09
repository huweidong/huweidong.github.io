# AFNetworking流程总结

### 一.调用POST或GET方法,并把成功、失败方法块传入.

### 二.调用dataTaskWithHTTPMethod方法返回NSURLSessionDataTask对象dataTask(下面是生成NSURLSessionDataTask对象的时候里面做的事情)

#### (1).根据POST或GET方法、URL、参数生成NSMutableURLRequest对象request

##### 1.判断参数有效与否

##### 2.根据URL和HTTPMethod方法类型初始化NSMutableURLRequest对象

##### 3.self.mutableObservedChangedKeyPaths里面的值会在初始化requestSerializer中相关的属性的时候通过kvo赋值或移除

##### 4.根据初始化选择的可序列化的类型(json、html、xml)调用不同类的方法赋值给mutableRequest

#### (2).根据(1)中生成的request对象并传入上传进度、下载进度方法块生成NSURLSessionDataTask对象并传入回调方法块(回调方法块里面根据返回的参数选择调用成功或失败方法块)

##### 1.判断线程安全
		
##### 2.根据NSURLSessionDataTask对象dataTask绑定方法块(成功回调方法块、下载上传进度方法块)和委托,并添加相关通知如暂停任务和继续任务通知、添加相关任务属性kov的检测如countOfBytesReceived，countOfBytesExpectedToReceive属性等
### 三.调用dataTask对象的方法resume发送网络请求
### 四.然后从网络请求委托回调返回数据转发给AFURLSessionManagerTaskDelegate委托中的对应的方法
### 五.解析数据，然后从AFURLSessionManagerTaskDelegate中取出completionHandler方法块回调(解析数据是在另外开一个进程里面解析的)