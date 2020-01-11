# # Android 部分

## HandlerThread

### 为什么要HandlerThread

一个线程中有唯一的Looper对象，负责消息循环；一个Looper中有唯一的MessageQueue，管理消息队列；Handler在发送消息前，必须与一个Looper进行绑定，也就是与MessageQueue进行绑定。一切准备就绪后，可以通过Handler把消息Message发送出去，Message会在MessageQueue中入队，然后等待Looper把消息取出，分发给Handler，最后Handler进行处理。

### HandlerThreads是什么？

1. 本质是一个继承自Thread的线程类
2. 内部有自己的Looper对象，可以进行Looper循环
3. 通过



## WebView

### WebView是什么

WebView是一种特殊的View，基于webkit引擎，展示web的控件，4.4之后直接使用Chrome内核。

### webView基本用法

```java
//方式1：加载url
webView.loadUrl("http://139.196.35.30:8080/OkHttpTest/apppackage/test.html");

//方式2：加载asset文件夹下html
webView.loadUrl("file:///android_asset/test.html");

//方式3：加载手机sdcard上的html页面
webView.loadUrl("content://com.ansen.webview/sdcard/test.html");

//方式4 使用webview显示html代码
webView.loadDataWithBaseURL(null,"<html><head><title> 欢迎您 </title></head>" +
    "<body><h2>使用webview显示 html代码</h2></body></html>", "text/html" , "utf-8", null);
```

### WebSettings

对WebView进行各种配置和管理。

- setJavaScriptEnabled(boolean flag):是否支持 Js 使用。

- setCacheMode(int mode):设置 WebView 的缓存模式。

- setAppCacheEnabled(boolean flag):是否启用缓存模式。

- setSupportZoom(boolean support):是否支持缩放。

- setLoadsImagesAutomatically(boolean flag):支持自动加载图片

### WebViewClient

