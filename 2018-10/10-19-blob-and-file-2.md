# JavaScript中的文件对象与 Blob 续：svg 转 png

写完上篇文章后发现 Blob 的部分说的太少了。
这次再补上一个简单的 Blob 应用：把一段 svg 代码转成 png 图片。

## demo
地址：[codepen demo]

我从 [iconfont.cn] 上找了一段 svg 代码作为原材料，目标就是把它转成 png 图片下载下来。（虽然 iconfont 支持直接下载成 png 文件0.0）

代码如下（删掉了一些不必要的字段）：
```html
<svg viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg" width="128" height="128">
    <path d="M160.832 521.536l-17.216 11.392-36.032 33.664M153.984 519.744l-25.792 21.76c-10.88 9.344-27.456 23.296-27.456 23.296" fill="#D94437" p-id="3360"></path>
    <path d="M469.184 814.464c0.64-26.432-10.752-88.832 32.96-132.928 44.992-45.504 137.024-58.624 143.04-63.68 16.256-14.784 22.848-33.728 27.584-53.12-45.504 15.04-110.528-34.176-148.48-114.24-23.936-50.56-10.304-123.264-2.56-160.576-26.88-6.848-53.248-4.608-80.064 17.984-21.696 18.368-40.64 78.272-86.208 124.224-64 64.64-153.92 65.664-190.336 86.464-13.696 6.272-31.04 18.88-47.808 35.776a229.76 229.76 0 0 0-24 28.672 171.2 171.2 0 0 0 15.488 225.856l69.312 68.608a171.52 171.52 0 0 0 220.096 17.728c9.984-6.016 21.888-14.976 33.856-26.176 24.32-22.72 40.064-46.208 37.12-54.592z" fill="#D94437" p-id="3361"></path>
    <path d="M454.464 694.336c3.648-6.912 13.888-23.296 38.016-47.616 21.056-21.44 79.168-36.352 81.984-38.72 7.744-6.976 5.632-17.728 7.808-26.944-21.504 7.104-75.648-43.264-93.568-81.024-11.264-23.808-16.704-37.952-12.992-55.552-12.608-3.2-12.416 1.024-25.024 11.712-10.24 8.64-19.136 36.864-40.64 58.56-31.04 31.36-79.616 37.76-89.152 43.968l-26.24 16.448c-32.512 32.32-32.512 85.824-0.832 117.12l32.64 32.384a80.832 80.832 0 0 0 114.176-0.64l13.824-29.696z" fill="#FBE9EB" p-id="3362"></path>
    <path d="M455.616 635.2l-48.384-47.808 432.384-426.112 37.632 37.248z" fill="#5B4037" p-id="3363"></path>
    <path d="M846.592 256.512l-78.784-43.968 124.8-115.648 73.344 38.784z" fill="#5A4035" p-id="3364"></path>
    <path d="M278.912 647.488l101.312 100.288-28.416 28.8-101.312-100.352zM332.992 592.832l101.312 100.224-30.848 31.168L302.08 624z" fill="#4B5359" p-id="3365"></path>
</svg>
```

把这段 svg 代码粘到 demo的框里，然后点一下「Download PNG」按钮，就会得到一张 png 图片。

## 实现
实现有两个比较关键的点，一个是我们如何把 svg 转成图片，另一个是如何把图片下载下来。针对第二个问题，想要下载图片，就要拿到图片的地址，上一篇文章提到可以使用 URL 为 blob 对象生成地址链接，有了链接就好办了。针对第一个问题，我们要稍微做下转换，先用 svg 创建一个 blob 对象，通过这个 blob 对象创建一个 img 元素，把这个 img 绘制到 canvas 上，然后通过 canvas 再创建一个 blob 对象（png 类型），这个 blob 对象就是最终要被下载的 png 图片。

### 主要步骤
梳理了下，整个过程是：

```
svg->blob->img->canvas->blob->下载

```

分解一下每个步骤的关键点：
1. svg -> blob
上一篇文章我们介绍到，blob 的构造函数支持 DOMString，所以我们是可以直接基于 svg 代码串构建 blob 对象的。

2. blob -> img
上一篇文章我们介绍到，可以使用`URL.createObjectURL()`生成链接，作为`img`元素的`src`。我们可以用这个`src` new 一个 Image 对象。

3. img -> canvas
这个大家都知道的，可以使用 canvas 的`drawImage()`方法把`img`元素绘制到 canvas 上。

4. canvas -> blob
canvas 还有一个方法叫`toBlob()`，可以为画布上的图像创建一个 blob 对象，默认类型为`image/png`。

5. blob->png
再次使用`URL.createObjectURL()`方法为刚刚画布创建的 blob 生成链接，然后下载。

再提一下下载，我们知道`<a>`标签可以链接到一个图片，
```html
<a href="https://example.com/demo.png">
```
但浏览器并不会下载这张图片，而是会在页面内打开图片预览。想要浏览器以附件形式下载，有两种方案：
1. 图片服务器设置响应头`Content-Disposition: attachment;`；
2. 为`<a>`标签加上 [download属性]。
第一种涉及到服务端操作，有些麻烦，我们使用第二种方案就好。


下面来看一下具体的代码。

## 代码

### 核心代码
在看转换过程的核心代码之前，先介绍一下转换过程中用到的两个工具函数，第一个是`drawCanvas`，作用是把 img 绘制到 canvas 上，接收一个 img 元素，返回绘制好的 canvas 对象；第二个是`downloadBlob`，作用是下载 blob 对象，接收一个 blob 对象，把它下载下来，没有返回值。

后面会详细介绍这两个函数。先看一下转换过程：

```js
  // svg -> blob
  const blob = new Blob([text.value], { type: 'image/svg+xml;charset=utf-8' })

  // blob -> image
  const link = URL.createObjectURL(blob)

  const img = new Image()
  img.onload = () => {
    URL.revokeObjectURL(link)
    // img -> canvas
    const canvas = drawCanvas(img)
    // canvas -> blob -> png
    canvas.toBlob(downloadBlob)
  }
  img.src = link
```

在关键步骤都写了注释，感觉整体的逻辑还是比较清晰的。
最后的`canvas.toBlob(downloadBlob)`可以了解一下，`canvas.toBlob()`方法的第一个参数是个回调函数，回调函数的参数是创建好的 blob 对象，这行代码相当于：

```js
canvas.toBlob(blob => {
    downloadBlob(blob)
})
```

toBlob 的第二个参数是 blob 的 MIME 类型，默认就是`image/png`，所以不需要传；第三个参数是图片质量，我们写demo也不需要考虑什么质量/压缩，也不传了。

### 工具函数
最后看一下两个工具函数。

#### drawCanvas
```js
function drawCanvas(img){
  const canvas = document.createElement('canvas')
  canvas.width = img.width
  canvas.height = img.height
  const ctx = canvas.getContext('2d')
  ctx.drawImage(img, 0, 0)
  return canvas;
}
```

主要就是创建 canvas 然后用`ctx.drawImage(img, 0, 0);`把 img 元素绘制上去。

#### downloadBlob
```js
function downloadBlob(blob){
  const a = document.createElement('a')
  a.download = `image-${(new Date).toLocaleString()}`
  a.href = URL.createObjectURL(blob)
  a.click()
}
```

主要是使用`URL.createObjectURL(blob)`生成链接，用 download 属性让浏览器以附件形式下载图片、定义下载文件名，创建`a`标签并模拟点击。

## 总结
其实我感觉我这边拆开说有点不连贯，还不如直接在去[codepen demo]看完整的代码，实现还是比较简单清晰的，再查一查不熟的 api，就都好理解了。

## 参考
MDN：[`<a>`标签]、canvas 的 [drawImage()方法]、[toBlob()方法]、[响应头 Content-Disposition]

[`<a>`标签]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a
[drawImage()方法]: https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawImage
[toBlob()方法]: https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLCanvasElement/toBlob
[响应头 Content-Disposition]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition

[iconfont.cn]: http://iconfont.cn/
[download属性]: https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/a
[codepen demo]: https://codepen.io/JeremyFan/pen/YJYKNe?editors=1010
