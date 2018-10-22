## NSURLSession

iOS9之后提供了新的网络开发类库NSURLSession

_NSURLSession不需要考虑线程问题，而之前的NSURLConnection需要保持线程存在，利用runloop监听端口来实现线程生命周期一直存在。_

NSURLSession的简单使用方式如下：

```objective-c
// 创建session，所有请求任务公用一个session
self.session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]
                                                           delegate:self
                                                      delegateQueue:[[NSOperationQueue alloc] init] ];

// 创建请求任务及回调处理
NSURLSessionTask *task = [self.session dataTaskWithURL:[NSURL URLWithString:@"https://www.baidu.com/"] completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        NSLog(@"");
    }];
// 开始执行
[task resume];
```

不同的请求任务有不同的task类，比如：

NSURLSessionUploadTask :上传任务

NSURLSessionDownloadTask:下载任务

session的使用场景主要分为三种，可以通过初始化时传入NSURLSessionConfiguration对象来创建session。

NSURLSessionConfiguration可以对session进行一些配置，当上传或下载时，首先就是要创建对应的configuration，可以设置超时信息、缓存策略、连接条件等。

configuration主要有三种类型，对应如下：

```objective-c
// 默认的configuration
@property (class, readonly, strong) NSURLSessionConfiguration *defaultSessionConfiguration;
// 该configuration不会将cookie，cache等保存到本地，只保存在内存中
@property (class, readonly, strong) NSURLSessionConfiguration *ephemeralSessionConfiguration;
// 可以创建能够运行在后台的请求，切换到后台会创建另外一个线程去单独跑这个请求，这样app就可以被挂起
+ (NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier NS_AVAILABLE(10_10, 8_0);

```



_当使用回调方式处理数据时，需要注意NSURLSession的delegate是强引用，而我们又强引用session，所以会造成循环引用，所以在最后需要调用session的invalidateAndCancel来解除强引用。_

### NSURLSessionDelegate的主要方法如下：

```objective-c
/* 在发生系统错误或手动取消session时才会掉用*/
- (void)URLSession:(NSURLSession *)session didBecomeInvalidWithError:(nullable NSError *)error;

/* 两种情况下才会掉用：
 * 1.远程服务器要求客户端提供合适证书
 * 2.session第一次连接到一个使用了SSL或TLS协议的远程服务器
 * 建立过连接后，再次请求就不会走该delegate了
 */
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler;

/* 当后台任务完成或需要认证时，application会在后台重启，并收到
 * -application:handleEventsForBackgroundURLSession:completionHandler:
 * 消息, session的delegate就会收到该delegate
 */
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session NS_AVAILABLE_IOS(7_0);
```

比如连接到https的服务器时，只有第一次连接才会收到第二个delegate：

```objective-c
// 最近百度使用了https，用session发起https请求如下
- (void)viewDidLoad {
  self.session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration] delegate:self delegateQueue:[[NSOperationQueue alloc] init]];
  // ...
}

- (void)buttonClicked:(id)sender {
  [[self.session dataTaskWithURL:[NSURL URLWithString:@"https://www.baidu.com"]] resume];
}

// 第一次点击按钮连接到百度时会触发该delegate，之后再点击按钮则不会再回调该方法了
- (void)URLSession:(NSURLSession *)session didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * __nullable credential))completionHandler {
  // completionHandler回调告诉session证书的处理方式
  if([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {//是服务器信任证书 
    NSURLCredential *credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];//是服务器信任证书 
    if(completionHandler) 
      completionHandler(NSURLSessionAuthChallengeUseCredential,credential); 
  }
}
```

### NSURLSessionTaskDelegate请求任务的主要方法如下：

```objective-c
/* 与NSURLSession的证书认证delegate一致，当不实现session的认证时才回调该方法，
 * 实现了session的认证delegate就不会回调该delegate
 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                            didReceiveChallenge:(NSURLAuthenticationChallenge *)challenge 
                              completionHandler:(void (^)(NSURLSessionAuthChallengeDisposition disposition, NSURLCredential * _Nullable credential))completionHandler;

/* 发送数据的进度 */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                                didSendBodyData:(int64_t)bytesSent
                                 totalBytesSent:(int64_t)totalBytesSent
                       totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;

/* 请求结束后不管成功与否都会回调该delegate，没错时error为nil */
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                           didCompleteWithError:(nullable NSError *)error;
```

### NSURLSessionDataDelegate数据操作：

```objective-c
/* 比如请求一个链接时，要下载数据之前会征求你的同意
 * 可以取消请求、同意请求，也可以将task转变成download task 或 stream task，disposition决定
 *
 * 后台的upload task不会调用该delegate
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                 didReceiveResponse:(NSURLResponse *)response
                                  completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler;

/* 接收到的数据，并不是完整数据，需要拼接
 * 可以在请求开始时创建data，这里拼接，结束时处理
 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                     didReceiveData:(NSData *)data;

/* 缓存到底存不存 */
- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
                                  willCacheResponse:(NSCachedURLResponse *)proposedResponse 
                                  completionHandler:(void (^)(NSCachedURLResponse * _Nullable cachedResponse))completionHandler;
```

