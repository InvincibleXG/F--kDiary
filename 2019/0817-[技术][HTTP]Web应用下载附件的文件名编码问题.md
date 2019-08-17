# HTTP文件上传下载时的名称编码问题
----

> 自上映以来，终于有时间去电影院围观一下当下大受好评的国漫《哪吒》，特效剧情都不错，为国产作品点赞！

之前在项目开发中，会遇到客户要求浏览器兼容 IE9 的需求，那么除了一些样式兼容性问题需要考虑之外，对于附件的导出下载功能，也会出现一定的问题。而文件名(多发为中文等非ASCII字符)编码出现乱码，是一个比较明显和影响用户使用体验的 `bug级 issue`。

为了解决这个问题，之前我使用的方法是对 `Response对象` 增加一个“filename*”属性。

```java
response.addHeader("Content-Disposition", "attachment; filename*=utf-8'zh-cn'" + 
URLEncoder.encode(fileName, "utf-8"));
```

但是就这样，问题被解决了，而我并不是深刻地知道为什么，在这里面究竟发生了什么？

正好今天看到一篇深度好文，解答了我的疑惑，我这里也不多矫情，直接提炼出关键点：

> 在1999年 RFC 2616 推出之时， Content-Dispostion 这个 Header 尚不是正式 HTTP 协议的一部分，因而几乎没有浏览器去支持 Content-Disposition 的多语言编码特性这样一个“扩展特性的扩展特性”。
> IE支持在 filename 中直接使用百分号编码：filename=”$encoded_text”（并非 MIME 编码！）。本来按照 RFC 2616 ，引号内的部分如果不是 MIME 编码，则应当直接被当作内容，就算它“看起来像是百分号编码后的字符串”；可是IE却会“自动”对这样的文件名进行解码——前提是该文件名必须有一个不会被编码的（即 ASCII）后缀名！
> 其他一些浏览器则支持一种更为粗暴的方式：允许在 filename=”TEXT” 中直接使用 UTF-8 编码的字符串！这也是直接违反了 RFC 2616 HTTP 头必须是 ASCII 编码的规定。

一个是标准文档的制定遗留下的问题，一个是各大浏览器对此的特性支持不同，两种原因共同导致了这个文件名编码会出现异常的问题。而 2010 年 RFC 5987 发布，正式规定了 HTTP Header 中多语言编码的处理方式采用 parameter*=charset'lang'value 的格式。

> 1.charset 和 lang 不区分大小写。
2.lang 是用来标注字段的语言，以供读屏软件朗诵或根据语言特性进行特殊渲染，可以留空。
3.value 根据 RFC 3986 Section 2.1 使用百分号编码，并且规定浏览器至少应该支持 ASCII 和 UTF-8 。
4.当 parameter 和 parameter* 同时出现在 HTTP 头中时，浏览器应当使用后者。

原文作者*robotshell*提出:
> 其好处是保持了向前兼容性：一来 HTTP 头仍然是 ASCII-only ，二来不支持该标准的旧版浏览器会按照当年 RFC 2616 的规定，把 parameter* 整体当作一个 field name ，从而当作一个未知的字段来忽略。随后，2011年 RFC 6266 发布，正式将 Content-Disposition 纳入 HTTP 标准，并再次强调了 RFC 5987 中多语言编码的方法，还给出了一个范例用于解决向后兼容的问题：

>> Content-Disposition: attachment;
filename="EURO rates";
filename*=utf-8''%e2%82%ac%20rates;

> 这个例子里，filename 的值是一个同义英语词组——这样符合 RFC 2616 ，普通的字段不应当被编码；至于使用 UTF-8 只是因为它是标准中强制要求必须支持的。然而，如果我们再仔细想想——目前市场上常见的旧版本浏览器多为 IE 。如此一来，我们可以适当变通一下，将 filename 字段也直接使用百分号编码后的字符串：

>> Content-Disposition: attachment;
filename="%e2%82%ac%20rates.txt";
filename*=utf-8''%e2%82%ac%20rates.txt

> 对于较新的 Firefox 、 Chrome 、 Opera 、 Safari 等浏览器，都支持并会使用新标准规定的 filename* ，即使它们不会自动解码 filename 也无所谓了；而对于旧版本的IE浏览器，它们无法识别 filename* ，会将其自动忽略并使用旧的 filename（唯一的小瑕疵是必须要有一个英文后缀名）。这样一来就完美解决了多浏览器的多语言兼容问题，既不需要 UA 判断，也较为符合标准。


这下对于附件下载的需求，后端只要在响应头中加入两个`filename(*)属性`并且按需对可能出现的访问客户端环境做调整，即可防止文件名乱码问题了。