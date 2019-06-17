# jacob之word转pdf注意点 <导出> Word转PDF 实战思考(补)
---
之前写过` 《<导出> Word转PDF 实战思考》` 系列三篇文章，但是突然发现说了半天也没讲 Word文档 究竟是用啥方式 专为 PDF 类型的文件的！好吧，是我太粗心了！

也正好，今天给客户进行生产环境裸机部署，也遇到了一个 jacob项目(word)转PDF 的小问题，于是正好补上这篇系列的最后一块拼图。

**这里先说转化方法**

`jacob`是一个专门调用 `Windows COM组件` 的桥接器，其实也就是个中间件，转PDF 这个功能其实是 `MS Office` 本身的 API，因此我们要做的就是以合适的代码驱动桥接器调用`COM组件`——此组件在安装Office后会自动注册到系统的`组件服务`（mmc comexp.msc -32）中，这也是_为啥WPS不行_的缘故。

核心驱动代码如下(很通用，网上到处都是)：
```java
@Slf4j
public class PDFUtil
{
    private static final int wdFormatPDF = 17;
    private static final int xlTypePDF = 0;
    private static final int ppSaveAsPDF = 32;
    public static final int EXCEL_HTML = 44;

    private static Integer pdf_running=0;
    /**
     *  将办公文档类的文件转化为pdf文件
     * @param file 源文件绝对全路径
     * @param pdf 输出PDF全路径
     * @return 0 - 输出文件参数不为pdf文件
     *             -1 - 复制失败或转化异常
     *             -2 - 文件不存在
     *             -3 - 源文件就是pdf文件
     *             -4 - 源文件类型不支持
     */
    public static Integer convert2PDF(String file, String pdf)
    {
        String kind = getFileSuffix(file);
        if (!pdf.endsWith(".pdf")) return -2;
        String pdfFile = pdf;
        File f = new File(file);
        if (!f.exists()) {
            return -2;//文件不存在
        }
        if (kind.equals("pdf")) {
            if (copyFile(file, pdfFile)){
                return -1; //复制失败
            }
            return -3;//原文件就是PDF文件
        }
        if (kind.equals("doc") || kind.equals("docx") || kind.equals("txt")) {
            return word2PDF(file, pdfFile);
        } else if (kind.equals("ppt") || kind.equals("pptx")) {
            return ppt2PDF(file, pdfFile);
        } else if (kind.equals("xls") || kind.equals("xlsx")) {
            return excel2PDF(file, pdfFile);
        } else {
            return -4;
        }
    }

    private static int word2PDF(String inputFile, String pdfFile)
    {
        startPDFJob();
        try {
            ComThread.InitSTA();
            // 打开Word应用程序
            ActiveXComponent app = new ActiveXComponent("Word.Application");
            log.info("开始转化{}为{}...", inputFile, pdfFile);
            long date = new Date().getTime();
            // 设置Word不可见
            app.setProperty("Visible", new Variant(false));
            // 禁用宏
            app.setProperty("AutomationSecurity", new Variant(3));
            // 获得Word中所有打开的文档，返回documents对象
            Dispatch docs = app.getProperty("Documents").toDispatch();
            // 调用Documents对象中Open方法打开文档，并返回打开的文档对象Document
            Dispatch doc = Dispatch.call(docs, "Open", inputFile, false, true).toDispatch();
            Dispatch.call(doc, "ExportAsFixedFormat", pdfFile, wdFormatPDF);// word保存为pdf格式宏，值为17
            System.out.println(doc);
            // 关闭文档
            long date2 = new Date().getTime();
            int time = (int) ((date2 - date) / 1000);

            Dispatch.call(doc, "Close", false);
            // 关闭Word应用程序
            app.invoke("Quit", 0);
            ComThread.Release();

            finishPDFJob();

            return time;
        } catch (Exception e) {
            finishPDFJob();
            log.error(e.getMessage(), e);
            return -1;
        }

    }

    private static int excel2PDF(String inputFile, String pdfFile)
    {
        try {
            ComThread.InitSTA(true);
            ActiveXComponent ax = new ActiveXComponent("Excel.Application");
            log.info("开始转化Excel为PDF...");
            long date = new Date().getTime();
            ax.setProperty("Visible", false);
            ax.setProperty("AutomationSecurity", new Variant(3)); // 禁用宏
            Dispatch excels = ax.getProperty("Workbooks").toDispatch();

            Dispatch excel = Dispatch.invoke(excels, "Open", Dispatch.Method, new Object[]{
                    inputFile, new Variant(false), new Variant(false)}, new int[9]).toDispatch();
            Dispatch.invoke(excel, "ExportAsFixedFormat", Dispatch.Method,
                    new Object[]{ new Variant(0), // PDF格式=0
                            pdfFile,
                            new Variant(xlTypePDF) // 0=标准(生成的PDF图片不会变模糊) 1=最小文件(生成的PDF图片糊的一塌糊涂)
                    },
                    new int[1]);
            long date2 = new Date().getTime();
            int time = (int) ((date2 - date) / 1000);
            Dispatch.call(excel, "Close", new Variant(false));
            if (ax != null) {
                ax.invoke("Quit", new Variant[]{});
            }
            ComThread.Release();
            return time;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return -1;
        }
    }

    private static int ppt2PDF(String inputFile, String pdfFile)
    {
        try {
            ComThread.InitSTA(true);
            ActiveXComponent app = new ActiveXComponent("PowerPoint.Application");
//            app.setProperty("Visible", false);
            log.info("开始转化PPT为PDF...");
            long date = new Date().getTime();
            Dispatch ppts = app.getProperty("Presentations").toDispatch();
            Dispatch ppt = Dispatch.call(ppts, "Open", inputFile, true, // ReadOnly
                    //    false, // Untitled指定文件是否有标题
                    false// WithWindow指定文件是否可见
            ).toDispatch();
            Dispatch.invoke(ppt, "SaveAs", Dispatch.Method,
                    new Object[]{ pdfFile, new Variant(ppSaveAsPDF) },
                    new int[1]);
            System.out.println("PPT");
            Dispatch.call(ppt, "Close");
            long date2 = new Date().getTime();
            int time = (int) ((date2 - date) / 1000);
            app.invoke("Quit");
            return time;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            return -1;
        }
    }

    public static int excelToHTML(String xlsFile, String htmlFile)
    {
        ActiveXComponent app = new ActiveXComponent("Excel.Application"); // 启动Excel
        try {
            long date = new Date().getTime();
            app.setProperty("Visible", new Variant(false));
            Dispatch excels = app.getProperty("Workbooks").toDispatch();
            Dispatch excel = Dispatch.invoke(excels, "Open", Dispatch.Method, new Object[]{xlsFile,
                    new Variant(false), new Variant(true)}, new int[1]).toDispatch();
//				Dispatch sheet = Dispatch.invoke(excel, "sheet(0)", arg2, arg3, arg4)
            Dispatch.invoke(excel, "SaveAs", Dispatch.Method,
                    new Object[]{ htmlFile, new Variant(EXCEL_HTML) },
                    new int[1]);
            Variant f = new Variant(false);
            Dispatch.call(excel, "Close", f);
            long date2 = new Date().getTime();
            int time = (int) ((date2 - date) / 1000);
            app.invoke("Quit");
            return time;
        } catch (Exception e) {
            log.error(e.getMessage());
            return -1;
        } finally {
            app.invoke("Quit", new Variant[]{});
        }
    }

    public static String getFileSuffix(String fileName)
    {
        int splitIndex = fileName.lastIndexOf(".");
        return fileName.substring(splitIndex + 1).toLowerCase();
    }

    public static String getPureFileName(String filePath)
    {
        return filePath.substring(0, filePath.lastIndexOf('.'));
    }

    public static void deleteFile(String file)
    {
        File f=new File(file);
        if(f.exists()&&f.isFile()){
            ScheduleTask.requestForDeleteFile(file); // 删除当然是要定时任务来做
        }
    }

    public static boolean copyFile(String inPath, String outPath)
    {
        try{
            log.info("正在进行pdf复制操作------------...");
            Files.copy(new File(inPath).toPath(), new File(outPath).toPath());
            log.info(" pdf复制操作已完成------------...");
            return true;
        }catch (Exception e){
            log.error(" pdf复制操作失败-------------...", e);
            return false;
        }
    }

    public synchronized static void startPDFJob()
    {
        pdf_running++;
    }

    public synchronized static void finishPDFJob()
    {
        if (pdf_running<1) {
            log.error("PDF转化任务数量异常");
        }else {
            pdf_running--;
        }
    }

    public synchronized static Integer getPDFJobCount()
    {
        return pdf_running;
    }
}
```

这边代码都大差不差，那么经常出现的一些问题，其实往往都是因环境配置不对造成的。

必备条件:
> 引入 jacob 对应版本的依赖 jar (屁话)

> 在所使用 JDK/JRE 的 bin 目录下放置 jacob 对应版本的动态链接库 dll，这一点很重要，有的人是放错了位置，有的人是32位64位没搞对，还有的是jacob的jar和dll版本不匹配等等。当然了，这个dll的路径也应该是有API可以去指定一个位置的(只是按软件设计思路去猜测没有验证因为没必要)

> 所用 Windows 系统的组策略配置正确！（这点容易被忽略）因为有的系统会默认 “仅用户” 表示只能通过UI接口进行`COM调用`，而应当设置为 “非交互用户” 意思是用户和service都可以调用！这个时候如果配置不对，异常信息很可能是`Variant ChangeType failed`，而在网上也有人说这个错可能由于C:\Windows\System32\config\systemprofile\Desktop目录无法自动生成而引起

> 被转的文档文件状态是正常的，不能说被加锁或者不存在


好了以上就是使用 `jacob` 这个 `COM桥接器` 驱动 `MS office API` 进行文档转PDF的一些注意点啦。希望可以帮到你~ 