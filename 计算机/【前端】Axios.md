# Axios by wyxh2004

> 本站作者：[微雨星晗](https://github.com/WeiYuXingHan)
>
> 本站地址：[https://wyssixsixsix.top](https://wyssixsixsix.top)

## Axios 是什么?

Axios 是一个基于` promise 网络请求库`，作用于node.js 和浏览器中。 它是 `isomorphic` 的(即同一套代码可以运行在浏览器和node.js中)。在服务端它使用原生 node.js http 模块, 而在客户端 (浏览端) 则使用 XMLHttpRequests。

## 特性
- 从浏览器创建 `XMLHttpRequests` 
- 从 node.js 创建 `http` 请求
- 支持 `Promise API`
- 拦截请求和响应
- 转换请求和响应数据
- 取消请求
- 超时处理
- 查询参数序列化支持嵌套项处理
- 自动将请求体序列化为：
  - JSON (`application/json`)
  - Multipart / FormData (`multipart/form-data`)
  - URL encoded form (`application/x-www-form-urlencoded`)
- 将 HTML Form 转换成 JSON 进行请求
- 自动转换JSON数据
- 获取浏览器和 node.js 的请求进度，并提供额外的信息（速度、剩余时间）
- 为 node.js 设置带宽限制
- 兼容符合规范的 FormData 和 Blob（包括 node.js）
- 客户端支持防御XSRF
  
## 安装

在此仅展示使用 `npm` 安装:

```bash
$ npm install axios
```
