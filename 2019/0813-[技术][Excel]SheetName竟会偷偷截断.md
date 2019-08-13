# 好大的胆子，截断字符串并不会告知调用方！
----

有如下一段代码，旨在获取 DN号 作为 Excel Sheet Name，并且获取这个 sheet 的`下标`。但是由于合作开发，DN号 需要调用某个接口来获取，因此暂时我就想以记录的唯一标识 GUID 来作为 Sheet Name。然而令我无语的是，创建后我拿到的下标一直是 -1，以致于表现为报表导出为空。

```C#
int sheetIndex = excel.CreateSheet(string.IsNullOrEmpty(item.DNNo) ? item.guid : item.DNNo);
public int CreateSheet(string sheetName)
{
    ISheet sheet = workbook.CreateSheet(sheetName);
    int n = workbook.NumberOfSheets; // n有正确的、创建sheet后的值
    if (sheet != null)
    {
        return workbook.GetSheetIndex(sheetName); // 在此上下文中始终返回-1表示找不到sheet
    }
    return -1;
}
```

于是我开始调试。。。。。但是却发现： Sheet 的 Name 被截断了 5 个字符！

![](https://github.com/InvincibleXG/imgs/blob/master/20190813-sheetName截断.jpg)


```C#
public int CreateSheet(string sheetName)
{
    if (string.IsNullOrEmpty(sheetName) || sheetName.Length > 31)
    {
        throw new Exception("sheetName 无效或将被截断");
    }
    ISheet sheet = workbook.CreateSheet(sheetName);
    if (sheet != null)
    {
        return workbook.GetSheetIndex(sheetName);
    }
    return -1;
}
```

从小到大用了 6~7 年的 Excel，第一次知道这个 Name 会一声不响直接给你截断了。。。

我再去 Excel 办公软件中操作了一下，果然也是这样，行吧。