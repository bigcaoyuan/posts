# JavaScript中的文件对象与 Blob

文件对象是 Web 应用中很重要的一部分，我们经常实现的上传功能就会用到文件对象，而对于 Blob 就没有那么熟悉了。

文件对象和 Blob 都是浏览器 File API 的一部分。我们先来回顾一下 File API 的构成。

##  File API 构成
File API 一共包含五个部分：
- File
- FileList
- FileReader
- Object URL
- Blob

简单来说，File、FileList 提供文件对象，FileReader 提供读取文件的方法，URL 可以生成指向本地文件的链接。

### File 与 FileList
可能我们最熟悉的 File API 就是 File 和 FileList 了。
FileList 表示一个文件列表，是一个类数组对象，包含的每个元素都是 File 对象。File 对象不能直接通过代码创建，只能从 FileList 中取得。

通常我们通过 input 元素上传文件后，这个元素的 `files` 属性就是一个 FileList 对象。
```html
<input id="f" type="file">
```
```js
const f = document.getElementById('f');
console.log(f.files); // FileList {0: File(414), length: 1}
```

如果我们使用 HTML5 的拖拽接口，可以通过事件对象的 `DataTransfer` 属性取到 FileList。
```html
<div id="f"></div>
```
```js
const f = document.getElementById('f');
f.addEventListener('drop', event => {
  console.log(event.dataTransfer.files); // FileList {0: File(414), length: 1}
}, false);
```

### FileReader
FileReader 用来读取文件的内容，它的工作形式和 XHR（XMLHTTPRequest）非常像，不同的是 XHR 读取的是远程服务器上的资源，而 FileReader 读取的是本地文件。
FileReader 主要提供这三个方法，可以以不同形式读取文件的内容：
- `readAsText()`：读取成字符串
- `readAsDataURL()`：读取成Data URLs
- `readAsArrayBuffer()`：读取成 ArrayBuffer 对象

Data URLs 即 `data:` 协议的URL，比如我们常常看到的这种直接把图片的内容用 base64 编码，然后嵌到 HTML 中的形式：
```html
<img  src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAMAAAAoLQ9TAAAABGdBTUEAALGPC/xhBQAAAAFzUkdCAK7OHOkAAAAwUExURUdwTL+/v7+/v8DAwL+/v7+/v7+/v7+/v7q6usLCwr+/v7+/v8HBwcPDw8DAwL+/v8zqaoIAAAAQdFJOUwD0Z8uCPVRQFh7ttBEViXTv1nCIAAAAaUlEQVQY022PUQ7AIAhDwapDdNv9bzumksVFPgx9lopEu5IIZkRxnRkhpQDNrsNoAncirl9y2hkxVDGOaA2mod5meS85EV1lGBNPkHFoc9BHKh89FR7auh6hot+zKrvFjOi6OtH5+9xSD/eZAePNKLIVAAAAAElFTkSuQmCC">
```

另外还有中止读取操作的方法 `abort()`，以及已经废弃的方法 `readAsBinaryString()`（被 `readAsArrayBuffer()` 替代）。

以 `readAsText()` 为例读取文件内容，可以看到确实和 XHR 非常像：
```js
const reader = new FileReader();
reader.onload = event => {
  console.log(event.target.result);
};
reader.readAsText(file);
```

### URL
URL 是 File API 定义的一个全局对象，也叫 Blob URL 或 Object URL。
比如我们上传一张图片，可以通过`URL.createObjectURL()`为图片创建一个 URL。
```js
const url = URL.createObjectURL(file);
console.log(url); // "blob:https://www.example.com/0373d367-e899-4cd6-a2ea-8d8a0fa559f5"
```
可以看到，URL 是`blob:https://www.example.com/0373d367-e899-4cd6-a2ea-8d8a0fa559f5`这种形式的链接，可以直接作为 `<img>` 标签的 `src`。
```html
<img src="blob:https://www.example.com/0373d367-e899-4cd6-a2ea-8d8a0fa559f5">
```

可以使用`URL.revokeObjectURL()`销毁创建的 URL 实例，释放内存：
```
URL.revokeObjectURL(url);
```
有一点类似上面提到的 Data URLs，但 Data URLs 是直接表示图片内容，Blob URL 是指向本地图片的一个链接。

## Blob
终于说到 Blob 了。

Blob 是 Binary Large Object 的缩写，翻译成中文是二进制大型对象。MDN 对它的描述是「a file-like object of immutable, raw data」，规范中的描述是「represents immutable raw binary data」，可以把 Blob 理解成存储二进制原始数据的类文件对象。大部分情况下，可以像操作文件对象一样操作 Blob，因为文件对象的接口继承自 Blob，也可以说文件对象是基于 Blob 实现的。

先来看一下 Blob 都有哪些接口吧。

### 接口
其实 Blob 的接口比较简单：
- 构造函数（new Blob( array[, options])），通过一个数组参数和一个可选配置对象创建一个 Blob：
    - array：可以是 ArrayBuffer、ArrayBufferView、Blob、或 DOMString（其实就是 String）
    - options：
        - type：MIME 类型
        - endings：指定包含行结束符\n的字符串如何被写入
- 两个实例属性：
    - size：获取 Blob 对象的长度，以字节为单位
    - type：获取 Blob 对象的 MIME 类型
- 一个实例方法（slice([start[, end[, contentType]]])），根据一个 range 切片，生成一个新的 Blob：
    - start/end：新 Blob 的起始/结束位置
    - contentType：新 Blob 的 MIME 类型

我们来用一下这些接口：
```js
const blob = new Blob(['Hello Blob'], {type : 'text/plain'});
console.log(blob); // Blob(10) {size: 10, type: "text/plain"}

const newBlob = blob.slice(0,5,'text/plain');
console.log(newBlob); // Blob(5) {size: 5, type: "text/plain"}
```
可以看到 `size` 和 `type` 两个属性的内容，还可以通过上面提到的 FileReader 读取新 Blob 对象中的内容：
```js
const reader = new FileReader();
reader.onload = event => {
  console.log(event.target.result); // Hello
};
reader.readAsText(newBlob);
```

## Blob 小应用
最后以一个简单的小栗子作为结束吧。
```js
// 创建一个脚本类型的Blob对象
const blob = new Blob(['alert("a blob script")'], { type: 'text/javascript' });
const script = document.createElement('script');
// 创建链接
script.src = window.URL.createObjectURL(blob);
document.body.appendChild(script);
```
这段代码使用 Blob 创建了一个脚本文件并生成 URL 链接，然后创建了一个 `<script>` 标签引入脚本、插到文档里。

在浏览器控制台输入这段代码，会弹出一个"a blob script"框，是不是有点好玩呢。

PS：少数网站，比如github，不会弹框，会在控制台报错
```
Refused to load the script 'blob:https://github.com/b3a5b7a2-102d-4466-9e1e-5ed456b21c82' because it violates the following Content Security Policy directive: "script-src assets-cdn.github.com".
```
这是出于安全考虑，github.com 只信任自己的 cdn 站点——`assets-cdn.github.com`上的脚本。

## 参考
- File API 规范：https://w3c.github.io/FileAPI/
- Nicholas C. Zakas 的 File API 系列文章：https://humanwhocodes.com/blog/tag/file-api/
- An Introduction To JavaScript Blobs and File Interface：http://qnimate.com/an-introduction-to-javascript-blobs-and-file-interface/
- MDN：[File](https://developer.mozilla.org/en-US/docs/Web/API/File)、[FileList](https://developer.mozilla.org/en-US/docs/Web/API/FileList)、[FileReader](https://developer.mozilla.org/en-US/docs/Web/API/FileReader)、[URL](https://developer.mozilla.org/en-US/docs/Web/API/URL)、[Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
