# Blob、ArrayBuffer和Buffer对象

> 本站作者：[微雨星晗](https://github.com/wyxh2004)
>
> 本站地址：[https://wyssixsixsix.top](https://wyssixsixsix.top)

`Blob` 、 `ArrayBuffer` 和 `Buffer` 都是在处理**二进制数据**时常用的对象类型，它们分别在不同的编程环境（主要是Web浏览器和Node.js）中被广泛使用。

### 概述
- Blob: 前端的一个专门用于支持文件操作的二进制对象
- ArrayBuffer：前端的一个通用的二进制缓冲区，类似数组，但在API和特性上却有诸多不同
- Buffer：Node.js提供的一个二进制缓冲区，常用来处理I/O操作

> 借鉴知乎大神的一张图
![借鉴知乎大神的一张图](https://pic2.zhimg.com/v2-ed143b043805e01fbbea5712c7e27789_r.jpg)

## Blob (Binary Large Object)
### 适用环境：Web浏览器

Blob（二进制大对象）是Web API提供的一个接口，用于表示不可变的、原始数据的类文件对象。它可以包含任意类型的二进制数据，如**文本、图像、音频、视频**等。Blob对象通常用于以下场景：

**文件上传**：通过`<input type="file">`元素获取用户选择的文件，其 `files属性` 返回一个包含Blob对象的数组。

**创建URL供浏览器预览或下载**：使用 `URL.createObjectURL(blob)` 方法将 `Blob` 转换为一个**临时的URL**，可以嵌入到 `<img>、<video>、<audio>` 标签或通过 `window.open(url)` 打开新窗口预览。完成后记得使用 `URL.revokeObjectURL(url)` 释放资源。
使用 `fetch` 或 `XMLHttpRequest` 发送二进制数据：作为请求体发送给服务器。

Blob有两种子类型：`Blob`和`File`。`File`继承自`Blob`，增加了与文件系统相关的元数据（如文件名、类型、大小等）。创建Blob的常见方法有：

```Javascript
// 字符串转Blob
const textBlob = new Blob(["Hello, World!"], { type: "text/plain" });

// 数组Buffer转Blob（在浏览器中使用ArrayBuffer）
const arrayBuffer = new ArrayBuffer(8);
const blobFromArrayBuffer = new Blob([arrayBuffer]);

// 创建File对象（通常用于用户选择的文件）
const file = new File(["content"], "filename.txt", { type: "text/plain" });
```

## ArrayBuffer
### 适用环境：Web浏览器

ArrayBuffer是Web API提供的*另一种*表示二进制数据的类型，它代表一块**固定长度、原始二进制数据缓冲区**。ArrayBuffer本身并不直接提供访问其内部数据的方法，而是需要通过**视图**（如 `Uint8Array、Int16Array` 等）来操作。ArrayBuffer常用于以下场景：

处理来自 `fetch`、`XMLHttpRequest`、 `FileReader` 等API返回的二进制数据。
与 `WebAssembly` 配合：WebAssembly模块的编译结果通常以ArrayBuffer形式提供。
创建和操作ArrayBuffer的例子：

```Javascript
// 创建一个8字节的ArrayBuffer
const buffer = new ArrayBuffer(8);

// 创建一个视图来访问ArrayBuffer
const uint8View = new Uint8Array(buffer);
uint8View[0] = 123; // 写入数据

// 从其他源获取ArrayBuffer
fetch("binary-resource.bin").then(response => response.arrayBuffer());
```
### 通过ArrayBuffer的格式读取本地数据
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    <input type="file" id="fileInput" />
    <script>
      function handleFileChange(evt) {
        const file = evt.target.files[0];
        if (!file) return;

        const reader = new FileReader(); // 创建一个FileReader对象用于读取文件
        reader.onload = function (e) {
          console.log(e.target.result); // 输出文件内容
        };
        reader.readAsArrayBuffer(file); //读取文件内容并以ArrayBuffer对象的形式返回
      }

      document
        .getElementById("fileInput")
        .addEventListener("change", handleFileChange);
    </script>
  </body>
</html>

```
运行结果：
![](https://p.ananas.chaoxing.com/star3/origin/bf75789a319323e910a4bb0c83470967.png)

> FileReader接口提供了以下方法用于读取不同的文件内容格式：
```js
readAsArrayBuffer(file) // 读取文件内容
readAsBinaryString(file) // 包含原始二进制数据的字符串
readAsDataURL(file) // 将文件内容读取为一个DataURL字符串(一种内联包含文件数据的URL格式，通常用于小型文件)
readAsText(file, [encoding]) // 将文件内容读取为文本字符串
abort() // 中断当前正在进行的读取操作
```

## Buffer
### 适用环境：Node.js

Buffer是Node.js环境中特有的一个全局对象，用于处理二进制数据。它类似于ArrayBuffer，但提供了更丰富的API和更高效的内存管理。Buffer主要用于：

- 处理文件I/O：读写文件、流操作等。
- 网络通信：在 `Node.js` 中，`HTTP`、`TCP`、`UDP`等网络协议的收发数据都表现为 `Buffer` 对象。
- 加密解密、哈希计算：Node.js的加密模块（如 `crypto` ）通常接受和返回Buffer对象。
- 创建和操作Buffer的例子：

```Javascript
// 创建一个包含"Hello, World!"的Buffer
const buffer = Buffer.from("Hello, World!");

// 获取Buffer长度
console.log(buffer.length); // 输出：13

// 读取Buffer内容
console.log(buffer.toString()); // 输出："Hello, World!"

// 从其他源获取Buffer
fs.readFile("file.txt", (err, data) => {
  if (err) throw err;
  console.log(data); // 输出：<Buffer ...>
});
```