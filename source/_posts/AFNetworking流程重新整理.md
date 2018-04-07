---
title: AFNetworking流程重新整理
date: 2018-04-07 15:22:22
---

### AFNetworking生成 NSURLSessionDataTask 对象

#### 生成请求request

1、根据URL生成请求对象。

2、如有设置请求属性则根据self.mutableObservedChangedKeyPaths里面的值会在初始化requestSerializer中相关的属性的时候通过kvo赋值或移除。

3、根据设置的不同的解析管理器调用不同的方法(解析器有json解析器、http解析器等解析器，这里以http解析器为例)，在解析器中首先设置请求头参数，然后再如果有定义相关参数定义特殊处理代码块则进行相关的处理，再然后如果为get或者是head请求则拼接相关的字符串并赋值给mutableRequest.URL中，如果是post请求则把参数经过特殊处理变成字符串赋值到请求的头里面。

4、生成失败则直接回调失败代码块。

#### 根据请求和相关参数生成 NSURLSessionDataTask 对象

1、在保证线程安全的前提下调用系统根据请求生成NSURLSession对象的方法。

2、用字典以AFURLSessionManagerTaskDelegate对象的方式保存相关的代码块。字典的key为taskIdentifier，并且添加开始请求和完成请求的相关通知。

### AFNetworking数据解析

1、存储解析器的类型(解析器有json，http，xml等类型)。

2、建立局部数据存储，释放全局数据存储data。

3、如返回的网络连接发生错误则直接在主线程回调完成代码块并发送相关通知。

4、反则进行数据解析，先判断数据的有效性(主要判断是否存在可接受的数据类型、MIME类型是否存在、data的长度是否存在可接受的状态码，请求的URL是否存在)，然后进行json解码把data变成相关的可解析字符串对象(如设置的是json数据格式)。

5、如是下载文件则把responseObject对象设置成下载文件的URL

6、然后再主线程回调完成代码块和发送完成请求的通知。