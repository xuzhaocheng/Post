---
title: NSURLSession - Background Session
date: 2019-11-19 15:09:11
categories: Tech
tags:
    - iOS
---
`Background Session`是`NSURLSession`中一类特殊的`Session`。在App处于`inactive`（包括`suspend`和`terminated`）的状态时，使用`background Session`能使我们有能力执行下载任务。事实上，iOS系统是将我们提交的任务放到了另外的进程里执行，待执行完成后，再通知给App，去一个指定的路径下获取结果。
<!-- more -->

`Background Session`的使用也不算很复杂。首先我们需要创建一个`Background Session`：
```objc
- (NSURLSession *)urlSession {
    static dispatch_once_t onceToken;
    static NSURLSession *_instance = nil;
    dispatch_once(&onceToken, ^{
        NSURLSessionConfiguration *config = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:@"MySession"];
        _instance = [NSURLSession sessionWithConfiguration:config delegate:self delegateQueue:nil];
    });
    return _instance;
}
```
其中`identifier`是App内的一个唯一标识符，iOS并不要求每个App使用的`identifier`唯一。

接着需要一个方法启动`background task`：
```objc
- (void)startBackgroundTask:(NSString *)urlString {
    NSURL *url = [NSURL URLWithString:urlString];
    NSURLSessionDownloadTask *task = [[self urlSession] downloadTaskWithURL:url];
    [task resume];
}
```

为了处理App的`suspend`状态，我们需要实现一个`application: handleEventsForBackgroundURLSession:completionHandler:`代理方法：
```objc
- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier completionHandler:(void (^)(void))completionHandler {
    [self urlSession];
    self.completionHandler = completionHandler;
}
```
当`background task`执行完成时，如果App处于`inactive`状态，iOS系统就会唤起App，并调用这个方法。在这个方法里我们缓存了`completionHandler`，我们会在稍后用到它。
`[self urlSession]`是为了重新将`Background task`和你的App创建的`Session`关联起来（因为你的App可能已经被`terminate`了），这很重要，否则系统就不知道这些`task`是属于哪个`Session`的。

然后`NSURLSessionDelegate`的方法`URLSessionDidFinishEventsForBackgroundURLSession:`会被回调，我们在这里调用刚刚缓存的`completionHandler`。
```objc
- (void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    dispatch_async(dispatch_get_main_queue(), ^{
        self.completionHandler();
    });
}
```
因为回调并不保证在主线程，而`completionHandler`必须在主线程中调用，所以我们需要把它dispatch到主线程中。

最后，如果下载完成，我们可以在`NSURLSessionDownloadDelegate`的回调`URLSession:downloadTask:didFinishDownloadingToURL:`中获取下载的文件。当然你也可以实现`URLSession:task:didCompleteWithError:`来检查任务是否成功，实现`URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite:`来查看下载进度。
你可以这样使用它们：
```objc
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didFinishDownloadingToURL:(NSURL *)location {
    NSLog(@"Did download to %@", location.absoluteString);

    NSString *filename = [location lastPathComponent];

    NSURL *target = [[self targetURL] URLByAppendingPathComponent:filename];
    if ([[NSFileManager defaultManager] fileExistsAtPath:target.path]) {
        [[NSFileManager defaultManager] removeItemAtURL:target error:nil];
    }

    NSError *error;
    [[NSFileManager defaultManager] moveItemAtURL:location toURL:target error:&error];
    if (error) {
        NSLog(@"error occurs while moving file: %@", error);
    }
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    NSLog(@"complete with error: %@", error);
}

- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didWriteData:(int64_t)bytesWritten totalBytesWritten:(int64_t)totalBytesWritten totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {
    NSLog(@"%@: %f", downloadTask, 1.0 * totalBytesWritten / totalBytesExpectedToWrite);
}
```

**Note**
`Background Session`可以在App被`terminate`之后唤起它，但是这仅限于被iOS系统`ternimate`，如果是用户手动kill，是没办法唤起我们的App的。[backgroundSessionConfigurationWithIdentifier:](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration/1407496-backgroundsessionconfigurationwi?language=objc)的文档里提到了这一点。

> If an iOS app is terminated by the system and relaunched, the app can use the same identifier to create a new configuration object and session and to retrieve the status of transfers that were in progress at the time of termination. This behavior applies only for normal termination of the app by the system. If the user terminates the app from the multitasking screen, the system cancels all of the session’s background transfers. In addition, the system does not automatically relaunch apps that were force quit by the user. The user must explicitly relaunch the app before transfers can begin again.

## Reference
[Downloading Files in the Background](https://developer.apple.com/documentation/foundation/url_loading_system/downloading_files_in_the_background?language=objc#2928518)
[URLSession downloadTask behavior when running in the background?](https://stackoverflow.com/questions/46939142/urlsession-downloadtask-behavior-when-running-in-the-background)
[Downloading files in background with URLSessionDownloadTask](https://www.ralfebert.de/ios-examples/networking/urlsession-background-downloads/)