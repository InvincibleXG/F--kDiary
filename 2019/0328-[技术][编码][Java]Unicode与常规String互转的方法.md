# Unicode与常规String互转的方法
----
今天收到了国际化需求中向客户讨要的翻译文件，是 Excel 的 中文 - 英文 形式。
  	
	| 中文短语 	| English Phrase |
    | 中文短语	| -------------- |
    | 中文短语 	| English Phrase |
    | 中文短语 	| English Phrase |

大概这样，就两列。而我的国际化文件只差英文了，所以**理所当然要写代码来帮我完成国际化资源文件的英文翻译**(其实就是把已经存在的中文国际化值根据Excel换为匹配的英文国际化值，涉及到读取+写入)。但是，中文资源中，汉字读出来是**Unicode**形式。

刚开始，我想的是，把从 Excel 中读取的‘中文短语’也转成 Unicode ，这样就可以全局替换为对应的英文短语啦~

#### 代码如下

```java
public static String stringToUnicode(String chinese)
{
    StringBuffer sb = new StringBuffer();
    for (int a = 0; a < chinese.length(); a++) {
        char c = chinese.charAt(a);
        sb.append("\\u" + Integer.toHexString(c));
    }
    return sb.toString();
}
```

结果遇到了一点小问题，就是**'\u'在regex中不受待见, 转义后一个'\\'为'\\\\\\\\'**，但是最后效果不尽如人意。因为中文短语里有英文字母，而英文字母就是本身不是 Unicode 。。导致上面那个方法还得修改，作为一个通用方法，如此修改绝非好事。

因此第二个办法，就是将读入的 Unicode 转为 正常的汉字String，然后再去 Excel 中匹配！

#### 代码如下
```java
public static String unicodeToString(String str)
{
    Pattern pattern = Pattern.compile("(\\\\u(\\p{XDigit}{4}))");
    Matcher matcher = pattern.matcher(str);
    char ch;
    while (matcher.find()) {
        ch = (char) Integer.parseInt(matcher.group(2), 16);
        str = str.replace(matcher.group(1), ch+"" );
    }
    return str;
}
```

由于是按照`\u+十六进制字符`匹配，自然无惧字母咯。

----

# 搞定