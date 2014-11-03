---
layout: post
title: "WKWebView和UIWebView之间的cookie共享"
date: 2014-11-03 23:52:27 +0900
comments: true
categories: iOS WKWebView UIWebView Cookie
---


## 前言
本文主要讨论一下，如何在iOS的UIWebView以及WKWebView之间进行cookie的同步。


## 为什么要做Cookie同步

最近在给公司游戏平台开发一个SDK，我主要负责了iOS版本的一部分。  

需要给开发者提供UIWebView和WKWebView，在一个Webview打开登陆页面登录了之后，
cookie可以同步到其它Webview，保证登陆状态能够持续。

## NSHTTPCookieStorage和NSHttpCookie
NSHTTPCookieStorage是objective c中cookie存储类。使用时候通过一个单例来进行调用
```objc
// call
NSHTTPCookieStorage *storage = [NSHTTPCookieStorage sharedHTTPCookieStorage];

// set cookies. cookies is NSArray of NSHttpCookie
[storage setCookies:cookies forURL:url mainDocumentURL:nil]

// delete cookie. cookie is NSHttpCookie
[storage deleteCookie:cookie]

```

## UIWebView内cookie同步
同一个应用，不同UIWebView之间的Cookie是自动同步的。它们都是保存在NSHTTPCookieStorage中。
当UIWebView加载一个URL的时候，在加载完成时候，这个URL response中包含的cookie会自动以NSHttpCookie的形式保存到
NSHTTPCookieStorage中。同时，如果在http response中，对cookie进行更新或者删除的话，其结果也会直接反应到NSHTTPCookieStorage
存储的cookie数据中。


所以，如果需要取出cookie数值，只要调用`[NSHTTPCookieStorage sharedHTTPCookieStorage]`即可。

## WKWebView内cookie同步

WKWebView的cookie并不会再加载url后自动保存到NSHTTPCookieStorage中。

>原因是因为现在WKWebView会忽视默认的网络存储， NSURLCache, NSHTTPCookieStorage, NSCredentialStorage。
>目前是这样的，WKWebView有自己的进程，同样也有自己的存储空间用来存储cookie和cache， 其他的网络类如NSURLConnection是无法访问到的。
>同时WKWebView发起的资源请求也是不经过NSURLProtocol的，导致无法自定义请求。  

同一个应用，不同WKWebView之间的cookie同步，可以通过使用同一个WKProcessPool实现cookie的同步。
```
WKWebViewConfiguration *internalConfig = [[WKWebViewConfiguration alloc] init];
internalConfig.processPool = [WKCookieSyncManager sharedInstance].processPool;  
```
实现方法，可以建立一个WKCookieSyncManager来管理共通的WKProcessPool
## UIWebView和WKWebView之间cookie同步
可以通过javascript实现同步。  
 * 在上面说到的WKCookieSyncManager中，创建一个WKWebView实例，实现WKNavigationDelegate。
 * 每次通过调用WKCookieSyncManager的setCookie方法的时候，先加载一个不存在的url，如`http://{SERVER_HOST}/_fake`.
 * 在`webView:didFinishNavigation:`中，如WKWebView实例将cookie转换成javascript。通过WKWebView的`evaluateJavaScript`
 方法，设置cookie.  

这里，SERVER_HOST最好使用服务器端想要设置的cookie的url host。而且要注意区别http和https，当NSHTTPCookieStorage使用
 `setCookies:forURL:mainDocumentURL:`所使用的host和这里SERVER_HOST使用的host的scheme不一样的时候，有可能会出现cookie不能同步的现象。


cookie格式:  
```js
document.cookie = "xxxxx";
```
设置cookie代码：  
```objc
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
    // make javascript cookie
    [webView evaluateJavaScript:cookieJS completionHandler:^(id result, NSError *error) {
        //...
    }];
}
```

最后，要注意清除cookie的时候，不仅需要从NSHTTPCookieStorage中将cookie清除，
也需要将javascript同步的cookie清除。我采用的方法是重新设置一遍cookie，给它们
设置一个过期时间，然后会被webview自动无视。

## References
 * NSHTTPCookieStorage: https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/Foundation/Classes/NSHTTPCookieStorage_Class/index.html
 * NSHttpCookie: https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/Foundation/Classes/NSHTTPCookie_Class/index.html
 * http://www.zhihu.com/question/24923190  
 * http://stackoverflow.com/questions/24464397/how-can-i-retrieve-a-file-using-WKWebView
