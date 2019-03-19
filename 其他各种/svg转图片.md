---
id: '2019-03-19-15-42'
date: '2019-03-19-15-42'
title: '纯html实现svg文件转png/jpg/bmp图片'
tags: ['html', 'svg转换', 'png', 'jpg', 'bmp']
categories:
  - '其他'
---

**本文原创发布于:**[www.tapme.top/blog/detail/2019-03-19-15-42](www.tapme.top/blog/detail/2019-03-19-15-42)

**源码存放于：**[github](https://github.com/FleyX/demo-project/blob/master/%E6%9D%82%E9%A1%B9/1.svg%E8%BD%ACpng%E3%80%81jpg.html)

&emsp;&emsp;svg 文件实际上是一个 xml 文件，用于描述一个矢量图。可直接用浏览器打开，浏览器会自动解析为图片。但是有时候我们可能会需要将 svg 转成 png/jpeg 等图片格式，有什么简单易用的办法呢？

&emsp;&emsp;最简单的就是通过 html 来进行转换操作。主要流程如下：

1. 使用`<input type='file'>`获取到文件信息

```javascript
let files = document.getElementById('file').files;
```

<!-- more -->

2. 使用 html5 中的`FileReader`将文件内容读取出来

```javascript
for (let i = 0; i < files.length; i++) {
  let reader = new FileReader();
  reader.readAsText(files[i]);
  reader.onload = function(e) {
    arr.push({
      name: files[i].name,
      content: this.result
    });
  };
}
```

3. 使用`DOMParser`解析 svg 内容

```javascript
let xml = new DOMParser().parseFromString(item.content, 'text/xml');
let svgContent = xml.getElementsByTagName('svg')[0].outerHTML;
```

4. 将 svg 内容放到一个`Image`中,然后再将该 img 写入到 canvas 中,最后将 canvas 画布转为 data 字符串，使用`a`标签下载下来。

```javascript
let img = new Image();
img.src = 'data:image/svg+xml;base64,' + window.btoa(unescape(encodeURIComponent(svgContent)));
//这里要等img加载完毕再写到canvas中，否则canvas可能为空
img.onload = function() {
  let canvas = document.getElementById('canvas');
  canvas.width = img.width;
  canvas.height = img.height;
  let ctx = canvas.getContext('2d');
  ctx.drawImage(img, 0, 0, img.width, img.height, 0, 0, img.width, img.height);
  //构建下载
  let a = document.createElement('a');
  a.href = canvas.toDataURL('image/' + postfix);
  a.download = item.name.substr(0, item.name.lastIndexOf('.')) + '.' + postfix;
  //调用点击开始下载
  a.click();
};
```

# 效果如下

![svg转png](https://raw.githubusercontent.com/FleyX/files/master/blogImg/20190319161059.gif)
