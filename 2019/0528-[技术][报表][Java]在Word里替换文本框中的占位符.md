# 如何在Word里替换文本框中的占位符？ <导出> Word转PDF 实战思考(二)
----
前置知识： 微软的 Office 体系，底层是什么？ 

你可以试试将任意 office 系文档，后缀名改为 .zip，打开以后发现里面是媒体资源 + xml。 所以这些办公文档文件其实骨子里是 XML 在支持，通过一些 图形化技术 和 其他计算技术结合起来，启动后一系列渲染操作，形成用户可见的办公文档。

**因此无论任何 API，只要是操作文档的非媒体资源部分，都是变相地在操作 XML 文档。**

以上知识纯属个人意淫，没有完整的实验体系，没有论文支撑，请勿当作真理和信仰，以免给我带来不必要的麻烦谢谢。

### 正文部分

其实，在 Java 的 POI 里，只要细心看过源码，多少能发现一点端倪。之前大概也说过咱们从 XWPFDocument 对象可以获取 页眉XWPFHeader、页脚XWPFFooter、正文XWPFParagraph和表格XWPFTable，然而页眉页脚表格包括文本框中，都还是嵌套的XWPFParagraph文段对象。如果从 XML 老母猪带套标签一套又一套 去理解，就非常易懂了。其实对 Document 的操作，就是要知道目标属于哪个标签下，然后是标签下的什么内容，大概如此。

文本框，在 POI 中，并没有一个专门的类去操作它，因此只能通过 XML 底层修改来操作。

Word 里的 XML 对象封装接口名为 CTP （无法得知这三个字母什么意思，反正他继承了XMLObject接口），拿到 CTP 对象后，可以调用多种选择器方法来获取指定命名空间的 XML 标签节点，至于命名空间是什么、节点的标签是什么、有什么属性特征……这些完全需要开发者自己打印出 xml 字符串后自行判断，在网络上我并没有看到相关的教学文档。比如我这里需要替换文本框中的第一个出现的占位符（模板是我自己设计的，每个文本框中只有一个占位符），那么打印出来看：
	文本框的代码可以通过 w:txtboxContent 大标签的路径选择器获取到；
	在文本框的标签中，有样式节点和段落节点p；
	段落节点p中有子节点r，姑且认为是‘行’的意思；
	在r节点中，有文本t节点，估计是text的意思，t节点的内容就是我的占位符啦！
因此代码如下：

```java
private static void dealWithTextBox(XWPFParagraph paragraph, Map<String, Object> map)
    {
        // 获取段落中的文本框
        CTP ctp=paragraph.getCTP();
        XmlObject[] textBoxObjects =  ctp.selectPath("declare namespace w=\"http://schemas.openxmlformats.org/wordprocessingml/2006/main\" .//*/w:txbxContent");
        for (int i =0; i < textBoxObjects.length; i++) {
            try {
                // 命名空间 w=http://schemas.openxmlformats.org/wordprocessingml/2006/main localPart节点标签=p -> r -> t
                XmlObject[] paraObjects = textBoxObjects[i].selectChildren(new QName("http://schemas.openxmlformats.org/wordprocessingml/2006/main", "p"));
                for (int j=0; j<paraObjects.length; j++) {
                    XmlObject[] rowObjects = paraObjects[j].selectChildren(new QName("http://schemas.openxmlformats.org/wordprocessingml/2006/main", "r"));
                    for (int k=0; k<rowObjects.length; k++) {
                        XmlObject[] textObjects = rowObjects[k].selectChildren(new QName("http://schemas.openxmlformats.org/wordprocessingml/2006/main", "t"));
                        for (int x=0; x<textObjects.length; x++) {
                            String text = textObjects[x].toString();
                            if (text != null && text.contains("${")) {
                                XmlCursor cursor = textObjects[j].newCursor();
                                String content=cursor.getTextValue();
                                for (Map.Entry<String, Object> entry : map.entrySet()) {
                                    String key = entry.getKey();
                                    if (content.contains(key)) {
                                        Object value = entry.getValue();
                                        if (value==null) value="";
                                        content = content.replace(key, value.toString());
                                        break;
                                    }
                                }
                                cursor.setTextValue(content);
                            }
                        }
                    }
                }
            } catch (Exception e) {
                log.error("Word模板文本框处理时发生异常", e);
            }
        }
    }
```

好了，只要传入一个文段对象和带占位符与对应值的Map就可以动态把数据渲染到文本框中了。而原理图的插入方法看上一篇即可，比较简单，只是需要手动调节像素等清晰度参数。




---- 
### 文末叨逼叨

之前写过一篇斩仓割肉的实记，里面总结了一下今年2月底以来在股市里学到的经验：

> 1. 选股第一步就是基本面，然后根据当前的局势大环境确定赚钱逻辑，按行业分析投资期限
> 2. 对待消息，千万不要冲动，冷静下来分析第一点，确定是赚钱的硬逻辑还是已经利好出尽的韭菜陷阱
> 3. 永远不要在历史的高点买入股票，即使是白马蓝筹，前30%也不要去买入
> 4. 买跌不买涨，分段建仓，持有为主，抄底为辅
> 5. 少看盘，不要想着赚热点的钱去无脑追涨；买入下跌中的股票前先去了解基本面和动态PE

然后我也在个人的圈子里给大家分享了我之前研究过财报的一些股票，分析做在前面是认清候选股的基本面，当买入时机来临时，才可以放心大胆义无反顾地抄底！我也多次提示底部在即，当然市场仍然还在底部并且仍有继续下穿的可能。

我一直坚信巴菲特的名言 “ 收益产生在买入时 ”，只要买对了时机，就不会踏空行情，至少在时机上是正确的；再结合精心的选股，从基本面到技术面到消息面再到宏观经济，确认过眼神怎么会亏损? 炒股是一件简单的事情，关键在于你如何对待它。

**这里我作为福利，分享一下个人资产配置的方法论雏形：** （按股市行情划分）

*  【赚钱效应极差的市场情况下（普遍恐慌并且没有突出赚钱逻辑的个股）】： 进行**指数基金**定投，部分可靠的主动基金和可转债基金也可考虑
*  【震荡的绞肉机行情下（高频震荡，无明显的中长期普涨普跌现象）】： 依靠结构性行情，选择4~5只不同行业的**个股**，高抛低吸，赚钱赚经验
*  【赚钱效应极强的市场情况下（明显的季周期MACD金叉和牛市启动信号）】： 持仓 60% 指数基金，减仓主动基金，个股选龙头，必要时可以追板
*  【大牛市（明显的高成交额，资金不断流入，就连垃圾股也涨）】： **逐步减仓**，如果有波动可以高抛低吸玩一玩，主要赚经验
*  【超级大牛市（大妈都在炒股）】： 赶紧**清仓**跑路吧，玩尼玛呢！ 