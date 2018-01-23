---
title: Swift网络层架构
date: 2017-12-26 17:16:21
tags:
---

## 简介

本篇文章记录一下自己常用的网络层架构，个人觉得十分高效易用。基于Swift 4.0编写。暂时命名为`Spider`。

`Spider`具有以下优点:

- 网络请求管理: 能够批量取消网络请求，也能使未返回的请求跟随其`ViewController`的销毁而取消。
- 缓存: 网络层自动对`Response`进行缓存，并支持不同粒度的请求-缓存映射。
- 精简的报错: 提供精简的错误类型和附带错误提示
- 数据格式校验: 对服务器下发的json数据进行自定义的校验，校验失败视为错误，由网络层抛出
- 路由配置

`Spider`会不是类似于`Alamofire`, `AFNetworking`这样的通用的网络组件库。而是会和我们采用的和服务器交互的协议耦合，本文以`Restful`协议来实现`Spider`。


## 结构


## 组件

### ApiError & ApiResult 封装返回数据

`Spider`通过枚举的方式抛出网络错误。通过关联值得方式附加更精准的错误信息。

```
public typealias HTTPStatusCode = Int

public enum ApiError: Error {
    case library(Error)                 // 调用网络库报错
    case http(HTTPStatusCode, String?)  // http返回不为200-300之间，附带解析response里服务器返回的错误字段
    case invalidJSON                    // 返回数据格式校验失败
    
    var hintText: String {
        switch self {
        case .library(let error):
            return "依赖库错误 \(error.localizedDescription)"
        case .http(let statusCode, let hint):
            return "\(statusCode) \(hint ?? "请求失败")"
        case .invalidJSON:
            return "返回数据格式不合法"
        }
    }
}
``` 

`case http(HTTPStatusCode, String?)`的string为服务器返回的错误提示。也可以客户端自己构造，这在下文ApiRequest介绍时会说到。我更倾向于服务器返回接口调用失败的提示，这样的好处是，可以降低不同平台客户端成本，对国际化等支持也更友好。

`ApiResult`也是通过枚举关联值的方式来封装返回数据。

```
public enum ApiResult<Value> {
    case success(Value)
    case failure(ApiError)
}
```

> 可以参考[Result GitHub](https://github.com/antitypical/Result)，`ReactiveSwift`也在使用`Result`来封装返回数据。

### ApiRequest 对请求进行封装

