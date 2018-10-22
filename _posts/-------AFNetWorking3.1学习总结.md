## AFNetWorking3.1源码学习报告

AFNetWorking3.1中网络请求是通过AFURLSessionManager来实现的；

而AFHTTPSessionManager是对AFURLSessionManager的进一步封装，提供了HTTP请求的接口。



AFURLSessionManager为每个创建的task分配一个delegate用来处理task执行过程中的进度监控及网络回调，该delegate为AFURLSessionManagerTaskDelegate类。

#### AFURLSessionManagerTaskDelegate简介：

为每个task的回调及状态监听进行统一处理。

```objective-c
@interface AFURLSessionManagerTaskDelegate : NSObject 
@property (nonatomic, strong) NSMutableData *mutableData; // task接收的数据
@property (nonatomic, strong) NSProgress *uploadProgress; // 进度
@property (nonatomic, strong) NSProgress *downloadProgress;
// ...
@property (nonatomic, copy) AFURLSessionTaskProgressBlock uploadProgressBlock; // 上传进度的回调
@property (nonatomic, copy) AFURLSessionTaskProgressBlock downloadProgressBlock; // 下载进度的回调
@property (nonatomic, copy) AFURLSessionTaskCompletionHandler completionHandler; // 请求完成的回调
@end

@implementation AFURLSessionManagerTaskDelegate

- (instancetype)init {
    // 初始化data及回调
}

#pragma mark - NSProgress Tracking

- (void)setupProgressForTask:(NSURLSessionTask *)task {
  	// 根据task的数据信息初始化progress的数据信息
  	// 通过kvo监听task的上传下载的总数据和已处理的数据的变化
    // 并监听progress的变化
}

- (void)cleanUpProgressForTask:(NSURLSessionTask *)task {
    // 移除各个kvo
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
    if ([object isKindOfClass:[NSURLSessionTask class]] || [object isKindOfClass:[NSURLSessionDownloadTask class]]) {
        // 根据task的任务的变化修改progress的数据
    }
    else if ([object isEqual:self.downloadProgress]) {
        if (self.downloadProgressBlock) {
            self.downloadProgressBlock(object); // 回调进度
        }
    }
    else if ([object isEqual:self.uploadProgress]) {
        if (self.uploadProgressBlock) {
            self.uploadProgressBlock(object); // 回调进度
        }
    }
}

#pragma mark - NSURLSessionTaskDelegate
// 当task完成或出错时调用，完成时error为nil
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
	// 将url、返回结果、error等保存在字典中，并回调completeHandler和发送完成通知
}

#pragma mark - NSURLSessionDataTaskDelegate
// 每个task的任务的数据可能分多次接收到
- (void)URLSession:(__unused NSURLSession *)session
          dataTask:(__unused NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{
    [self.mutableData appendData:data]; // 拼接数据
}

#pragma mark - NSURLSessionDownloadTaskDelegate

- (void)URLSession:(NSURLSession *)session
      downloadTask:(NSURLSessionDownloadTask *)downloadTask
didFinishDownloadingToURL:(NSURL *)location
{
    // 保存file
}

@end
```

该delegate是为了方便处理task的回调而创建的，AFURLSessionManager中为每一个新创建的task都创建了一个delegate类对象来管理task的数据的回调。

当创建一个dataTask时，其调用如下 ：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    // 该方法保证task的id唯一
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });
	// 对task进行管理，并添加对应的delegate
    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

#### url_session_manager_create_task_safely:

```objective-c
static void url_session_manager_create_task_safely(dispatch_block_t block) {
    if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
        // 在iOS8直接调用block可能造成task的id不唯一，所以在一个串行队列里执行创建工作
        // 可能当时苹果没对id做线程保护工作
        dispatch_sync(url_session_manager_creation_queue(), block);
    } else {
        block();
    }
}
```

该方法是为了保证task的id在iOS8也能保证唯一性。

#### AFURLSessionManager对task的管理:

如上，当创建好task后，会调用[self addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:]

```objective-c
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    // 这里主要对delegate进行配置工作
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask]; // 调用下面的方法

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}
// 对delegate和task的管理加保护
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate; // 字典管理，task的id做key
    [delegate setupProgressForTask:task]; // delegate监听task的各种状态，详情见上面delegate实现
    [self addNotificationObserverForTask:task]; // 监听task的状态
    [self.lock unlock];
}
```

获取对应的delegate：

```objective-c
- (AFURLSessionManagerTaskDelegate *)delegateForTask:(NSURLSessionTask *)task {
    AFURLSessionManagerTaskDelegate *delegate = nil;
    [self.lock lock]; // 加锁
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)];
    [self.lock unlock]; 

    return delegate;
}
```

移除delegate:

```objective-c
- (void)removeDelegateForTask:(NSURLSessionTask *)task {
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
    [self.lock lock];
    [delegate cleanUpProgressForTask:task]; // delegate移除监听
    [self removeNotificationObserverForTask:task]; // self移除对task状态的监听
    [self.mutableTaskDelegatesKeyedByTaskIdentifier removeObjectForKey:@(task.taskIdentifier)];
    [self.lock unlock];
}
```

#### task回调的处理：

所有的网络回调都是在AFURLSessionManager中进行的，要回调到对应的delegate中处理如下：

```objective-c
- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:task];
	// 获取delegate并调用对应方法
    if (delegate) {
        [delegate URLSession:session task:task didCompleteWithError:error];

        [self removeDelegateForTask:task];
    }

    if (self.taskDidComplete) { // block可自定义
        self.taskDidComplete(session, task, error);
    }
}
```

#### AFURLSessionManager对task的状态的监听：

task的执行与暂停方法如下：resume和suspend

AFN用method swizzling来实现在调用这两个方法时发送通知。

与resume方法交换的方法如下：

```objective-c
- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_resume];
    
    if (state != NSURLSessionTaskStateRunning) { // 发送通知
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}
```



### AFHTTPSessionManager：

该类对HTTP请求进行了进一步的封装，继承AFURLSessionManager。

主要提供了序列化工具，AFHTTPRequestSerializer和AFHTTPResponseSerializer。

已AFHTTPRequestSerializer为例可以直接设置参数，并生成想要的request类实例。

以post请求为例：

```objective-c
- (NSURLSessionDataTask *)POST:(NSString *)URLString
                    parameters:(id)parameters
     constructingBodyWithBlock:(void (^)(id <AFMultipartFormData> formData))block
                      progress:(nullable void (^)(NSProgress * _Nonnull))uploadProgress
                       success:(void (^)(NSURLSessionDataTask *task, id responseObject))success
                       failure:(void (^)(NSURLSessionDataTask *task, NSError *error))failure
{
    NSError *serializationError = nil;
    // 直接生成需要的request
    NSMutableURLRequest *request = [self.requestSerializer multipartFormRequestWithMethod:@"POST" URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters constructingBodyWithBlock:block error:&serializationError];
	// 还是调用AFURLSessionManager的接口
    __block NSURLSessionDataTask *task = [self uploadTaskWithStreamedRequest:request progress:uploadProgress completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(task, error);
            }
        } else {
            if (success) {
                success(task, responseObject);
            }
        }
    }];

    [task resume];  // 直接执行

    return task;
}
```

#### AFURLRequestSerialization:

提供序列化request功能：

```objective-c
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    // 设置request的header的所有自定义的值
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    // 将parameters转化为想要的格式
    NSString *query = nil;
    if (parameters) {
        if (self.queryStringSerialization) { // 自定义的转换方式
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            switch (self.queryStringSerializationStyle) { // 默认转换方式
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }
	// HTTPMethodsEncodingParametersInURI为NSSet,初始化时值为GET,DELETE,HEAD
    // 如果是GET,DELETE,HEAD则采用拼接在URL尾部的模式，否则放在body中
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```

以上是创建并序列化request 的过程，可以自定义所要添加的request的header的值，通过属性HTTPRequestHeaders

也可以自定义参数的转换格式，通过queryStringSerializationStyle 

最后，AFURLRequestSerialization会帮你放在request中。



### 总结：

AFNetWorking实现网络请求的核心在于AFURLSessionManager。

AFURLSessionManager通过持有NSURLSession，并作为其delegate，接收所有网络回调；

AFURLSessionManager提供task的创建接口，并为每个task分配对应的delegate，并管理，当收到网络回调时会分配到对应的task的delegate去处理；



AFHTTPSessionManager继承AFURLSessionManager，并提供了一组HTTP请求的接口；

AFHTTPSessionManager内部的request使用AFURLRequestSerialization序列化

