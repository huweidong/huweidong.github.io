---
title: AFNetworking图片加载理解
date: 2017-02-19 14:06:22
---

### [先是UIImageView+AFNetworking分类](https://github.com/MyExam-hu/SummaryPro/blob/master/Pods/AFNetworking/UIKit%2BAFNetworking/UIImageView%2BAFNetworking.m)

**主要方法setImageWithURLRequest**

```
- (void)setImageWithURLRequest:(NSURLRequest *)urlRequest
              placeholderImage:(UIImage *)placeholderImage
                       success:(void (^)(NSURLRequest *request, NSHTTPURLResponse * _Nullable response, UIImage *image))success
                       failure:(void (^)(NSURLRequest *request, NSHTTPURLResponse * _Nullable response, NSError *error))failure
```
该方法主要做了如下几件事:

* 如果请求的url为空则直接设置占位符图片为最终图片。

* 当前的回调的request和需要请求的request是不是为同一个，是的话则判断为重复调用，直接返回。

* 开始请求前，先取消之前的task,即解绑回调。

* 从缓存中根据这个请求获取缓存,如果有缓存则直接回调成功方法块。（这里和后面设置从网络请求数据中设置的缓存策略相对应）

* 无缓存有占位符图片则设置占位符图片。然后去下载图片，并得到一个receipt(AFImageDownloadReceipt对象)，可以用来取消回调，分别传入成功回调方法块 **(判断receiptID和downloadID是否相同 成功回调，设置图片，清空receipt)** 和失败回调方法块 **(清空receipt，回调上层失败方法块)**

### [然后是AFImageDownloader类](https://github.com/MyExam-hu/SummaryPro/blob/master/Pods/AFNetworking/UIKit%2BAFNetworking/AFImageDownloader.m)

这个类里面的方法主要负责从网络加载图片，当然如果本地有缓存也会优先获取缓存

**主要方法downloadImageForURLRequest:**

```
- (nullable AFImageDownloadReceipt *)downloadImageForURLRequest:(NSURLRequest *)request
                                                  withReceiptID:(nonnull NSUUID *)receiptID
                                                        success:(nullable void (^)(NSURLRequest *request, NSHTTPURLResponse  * _Nullable response, UIImage *responseObject))success
                                                        failure:(nullable void (^)(NSURLRequest *request, NSHTTPURLResponse * _Nullable response, NSError *error))failure
```

该方法主要做了如下几件事(方法里面皆为同步串行处理):

* 判断请求的url是否为空，如果为空则直接返回错误信息。

* 判断这个任务是否已经存在，存在则添加成功失败Block,然后直接返回，即一个url用一个request,可以响应好几个block(防止重复请求)

* 根据request的缓存策略，加载缓存，如果有缓存则直接加载成功方法块(缓存类AFAutoPurgingImageCache下面总结)

- 用sessionManager(AFHTTPSessionManager)去请求，注意，只是创建task,还是挂起状态。回调处理方法块如下(方法里面皆为异步并行处理)：
	* 请求完毕，安全移除AF task，并返回当前被移除的AF task
	* 如果请求失败则遍历AF task里面的失败方法，并在主线程里面回调
	* 如果请求成功则遍历AF task里面的成功方法，并在主线程里面回调
	* 回调完成，减少活跃的任务数，如果可以，则开启下一个任务

* 根据createdTask任务创建mergedTask并往当前任务字典里添加任务

* 判断当前并行限制数，如果小于，则开始任务下载resume，否则等待

* 然后则回到AF里面的各种委托回调了

### [接下来是AFAutoPurgingImageCache类](https://github.com/MyExam-hu/SummaryPro/blob/master/Pods/AFNetworking/UIKit%2BAFNetworking/AFAutoPurgingImageCache.m)

主要方法是addImage，作用是用来添加image到cache里里面，同时这个方法会判断当前的缓存是否有溢出，如果有溢出则会根据时间的排序取出最早时候的缓存不断清除，知道到达没有溢出最大设置缓存为止。方法比较简单，直接点链接看吧!