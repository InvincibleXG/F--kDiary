# SVG转PNG <导出> Word转PDF 实战思考(一)

----

之所以写这个系列的博客，一是为了记录下来项目研发中所遇到的问题，二是总结经验，三是通过梳理知识技能应用来开阔思路。

**项目背景**

客户需要导出非常复杂的PNG报表，其中内容包括但不限于：文本、表格、复杂折(曲)线图、带动态数值的原理图(矢量图)。其中文本和表格，无需多言，如果采用 Word转PDF 来实现，那就是 XWPFPragraph、XWPFTable 这种老生常谈的操作。 折曲线图采用 JFreecharts 应当可以实现，暂时还没有开发到与此相关的内容。

**原理图导出实现**

今天主要讨论的是原理图，原理图在 H5 中用 SVG 可以轻松画出来，并且利用 String.replace 就可以轻松替换动态数值。只是把 SVG 输出进 PDF 里再导出这种操作我就毫无经验了，如有小伙伴熟悉可以指教一下。我这里选择的第一种思路是： SVG 用 Batik 转换为 PNG 格式的流，将流写入 Word 的 Image 对象，实现插入 Word，再用 Jacob 和 office 链接库进行 Word 2 PDF转换。

这里提一下 Batik SVG转图 所需要的依赖! 网上很多人都甩一个 batik-all 给你，然而根本不存在这个依赖或者说这个依赖是无效的，最小有效的依赖如下，其中几何操作的两个包是可以不要的，我只是尝试能否在内存中对 SVG 的画布大小进行调整。


```xml

	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-transcoder</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-bridge</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-svg-dom</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-dom</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-util</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-ext</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-xml</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-css</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-script</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-gvt</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-awt-util</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-parser</artifactId>
	    <version>1.6</version>
	</dependency>
	<!-- 用于几何操作SVG -->
	<dependency>
	    <groupId>batik</groupId>
	    <artifactId>batik-swing</artifactId>
	    <version>1.6</version>
	</dependency>
	<dependency>
	<groupId>batik</groupId>
		<artifactId>batik-svggen</artifactId>
		<version>1.6</version>
	</dependency>
	<!-- 转图若出现XML解析异常可引入此包 -->
	<dependency>
	    <groupId>xerces</groupId>
	    <artifactId>xercesImpl</artifactId>
	    <version>2.12.0</version>
	</dependency>

```

```java

    private static void insertSVGByMark(XWPFDocument document, XWPFRun run, String mark, String svg) throws Exception
    {
        String text=run.getText(0);
        if (text==null || !text.equals(mark)) return;
        ByteArrayOutputStream pngOut=convertToPNG(svg);
        if (pngOut==null){
            log.error("SVG2PNG转化失败");
        }else {
            String blipId=document.addPictureData(pngOut.toByteArray(), XWPFDocument.PICTURE_TYPE_PNG);
            addPictureToRun(run, blipId, XWPFDocument.PICTURE_TYPE_PNG, 1000, 1000);
        }
    }
    public static ByteArrayOutputStream convertToPNG(String svgCode) throws IOException
    {
        ByteArrayOutputStream outputStream=new ByteArrayOutputStream();
        try {
            byte[] bytes = svgCode.getBytes("UTF-8");
            PNGTranscoder t = new PNGTranscoder();
//            Transcoder t= new JPEGTranscoder();
//            t.addTranscodingHint(JPEGTranscoder.KEY_QUALITY, 0.99f);
            TranscoderInput input = new TranscoderInput(new ByteArrayInputStream(bytes));
            TranscoderOutput output = new TranscoderOutput(outputStream);
            t.transcode(input, output);
            return outputStream;
        } catch (Exception e){
            log.error(e.getMessage(), e);
            outputStream.close();
        }
        return null;
    }
    /**
     * 往Run中插入图片(解决在word中不显示的问题)
     * @param run
     * @param blipId 图片的id
     * @param format 图片的类型
     * @param width 图片的宽
     * @param height 图片的高
     */
    public static void addPictureToRun(XWPFRun run, String blipId, int format, int width, int height)
    {
        final int EMU = 9525;
        width *= EMU;
        height *= EMU;
        CTInline inline = run.getCTR().addNewDrawing().addNewInline();
        String picXml = "<a:graphic xmlns:a=\"http://schemas.openxmlformats.org/drawingml/2006/main\">" + "   <a:graphicData uri=\"http://schemas.openxmlformats.org/drawingml/2006/picture\">" + "      <pic:pic xmlns:pic=\"http://schemas.openxmlformats.org/drawingml/2006/picture\">" + "         <pic:nvPicPr>" + "            <pic:cNvPr id=\"" + format + "\" name=\"Generated\"/>" + "            <pic:cNvPicPr/>" + "         </pic:nvPicPr>" + "         <pic:blipFill>" + "            <a:blip r:embed=\"" + blipId + "\" xmlns:r=\"http://schemas.openxmlformats.org/officeDocument/2006/relationships\"/>" + "            <a:stretch>" + "               <a:fillRect/>" + "            </a:stretch>" + "         </pic:blipFill>" + "         <pic:spPr>" + "            <a:xfrm>" + "               <a:off x=\"0\" y=\"0\"/>" + "               <a:ext cx=\"" + width + "\" cy=\"" + height + "\"/>" + "            </a:xfrm>" + "            <a:prstGeom prst=\"rect\">" + "               <a:avLst/>" + "            </a:prstGeom>" + "         </pic:spPr>" + "      </pic:pic>" + "   </a:graphicData>" + "</a:graphic>";
        //CTGraphicalObjectData graphicData = inline.addNewGraphic().addNewGraphicData();
        XmlToken xmlToken = null;
        try {
            xmlToken = XmlToken.Factory.parse(picXml);
        } catch (XmlException xe) {
            xe.printStackTrace();
        }
        inline.set(xmlToken);
        //graphicData.set(xmlToken);

        inline.setDistT(0);
        inline.setDistB(0);
        inline.setDistL(0);
        inline.setDistR(0);

        CTPositiveSize2D extent = inline.addNewExtent();
        extent.setCx(width);
        extent.setCy(height);

        CTNonVisualDrawingProps docPr = inline.addNewDocPr();
        docPr.setId(format);
        docPr.setName("Picture " + format);
        docPr.setDescr("Generated");
    }

```

但是实际上， H5 页面会自动缩放 SVG 使其清晰，而 Batik 转出来的 PNG 有时很模糊，大概像素只有 400*400，也没有相关的 API 可以很好地解决这个问题。

因此，最终经过整整半天的无用功，我决定放弃 SVG 转 PNG 这个思路，改用 Word 模板中拖入 SVG 自动转的清晰 PNG 无数值原理图后，手动调整模板——用文本框加上原理图所需的数字符号等信息。但是这样做就需要动态替换 Word 中的文本框中的占位符，读取文本框跟读取正常的文本段和表格都是不一样的，非常麻烦。

由于时间原因，得等到项目完全结束后才有时间详述。敬请期待系列下篇~