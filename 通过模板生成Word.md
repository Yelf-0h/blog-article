# Poi-tl的使用

## 1、说明

[poi-tl官方文档地址](https://deepoove.com/poi-tl/#_license)

[官方源码地址](https://github.com/Sayi/poi-tl)

poi-tl（poi template language）是Word模板引擎，使用Word模板和数据创建很棒的Word文档。

在文档的任何地方做任何事情（Do Anything Anywhere）是poi-tl的星辰大海。

可以方便的生成word文档

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d33a6130fe2ae8b74481caf756f292e8.png)

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/b66488ac5ce3ec00f0ae58d42be38e3e.png)

这边需要注意，poi-tl的版本与Apache POI、JDK的版本，要在要求之内

## 2、使用

模板是Docx格式的Word文档，你可以使用Microsoft office、WPS Office、Pages等任何你喜欢的软件制作模板，也可以使用Apache POI代码来生成模板。

所有的标签都是以{{开头，以}}结尾，标签可以出现在任何位置，包括页眉，页脚，表格内部，文本框等，表格布局可以设计出很多优秀专业的文档，推荐使用表格布局。

poi-tl模板遵循“所见即所得”的设计，模板和标签的样式会被完全保留。

这里只介绍一些基本标签，具体的标签以及一些高级语法可以去看官方文档

### 2.1、文本

通过 {{var}} 接收

```java
put("name", "Sayi");
put("author", new TextRenderData("000000", "Sayi")); //黑色
put("link", new HyperlinkTextRenderData("website", "http://deepoove.com"));
put("anchor", new HyperlinkTextRenderData("anchortxt", "anchor:appendix1"));

//或者链式设置
put("author", Texts.of("Sayi").color("000000").create());
put("link", Texts.of("website").link("http://deepoove.com").create());
put("anchor", Texts.of("anchortxt").anchor("appendix1").create());

//文本换行使用 \n 字符。
```

数据模型：

String ：文本

TextRenderData ：有样式的文本

HyperlinkTextRenderData ：超链接和锚点文本

Object ：调用 toString() 方法转化为文本

### 2.2、图片

图片标签以@开始：{{@var}}

```java
// 指定图片路径
put("image", "logo.png");
// svg图片
put("svg", "https://img.shields.io/badge/jdk-1.6%2B-orange.svg");

// 设置图片宽高
put("image1", Pictures.ofLocal("logo.png").size(120, 120).create());

// 图片流
put("streamImg", Pictures.ofStream(new FileInputStream("logo.jpeg"), PictureType.JPEG)
  .size(100, 120).create());

// 网络图片(注意网络耗时对系统可能的性能影响)
put("urlImg", Pictures.ofUrl("http://deepoove.com/images/icecream.png")
  .size(100, 100).create());

// java图片
put("buffered", Pictures.ofBufferedImage(bufferImage, PictureType.PNG)
  .size(100, 100).create());

// 不同版本可能方法不一样 注意版本的区别 网络图片
put("urlPicture", new PictureRenderData(100, 100, ".png", BytePictureUtils.getUrlBufferedImage("https://avatars3.githubusercontent.com/u/1394854")));

```

数据模型：

String ：图片url或者本地路径，默认使用图片自身尺寸

PictureRenderData

ByteArrayPictureRenderData

FilePictureRenderData

UrlPictureRenderData

### 2.3、表格

表格标签以#开始：{{#var}}

```java

// 例子1 一个2行2列的表格 直接创建
put("table0", Tables.of(new String[][] {
                new String[] { "00", "01" },
                new String[] { "10", "11" }
            }).border(BorderStyle.DEFAULT).create());

// 例子2 第0行居中且背景为蓝色的表格
RowRenderData row0 = Rows.of("姓名", "学历").textColor("FFFFFF")
      .bgColor("4472C4").center().create();
RowRenderData row1 = Rows.create("李四", "博士");
put("table1", Tables.create(row0, row1));

// 例子3 合并第1行所有单元格的表格
RowRenderData row0 = Rows.of("列0", "列1", "列2").center().bgColor("4472C4").create();
RowRenderData row1 = Rows.create("没有数据", null, null);
MergeCellRule rule = MergeCellRule.builder().map(Grid.of(1, 0), Grid.of(1, 2)).build();
put("table3", Tables.of(row0, row1).mergeRule(rule).create());

```

数据模型：

TableRenderData

TableRenderData表格模型在单元格内可以展示文本和图片，同时也可以指定表格样式、行样式和单元格样式，而且在N行N列渲染完成后可以应用单元格合并规则 MergeCellRule ，从而实现更复杂的表格。

### 2.4、列表

列表标签以*开始：{{*var}}

```java
put("list", Numberings.create("Plug-in grammar",
                    "Supports word text, pictures, table...",
                    "Not just templates"));

/*
编号样式支持罗马字符、有序无序等，可以通过 Numberings.of(NumberingFormat) 来指定。
DECIMAL //1. 2. 3.
DECIMAL_PARENTHESES //1) 2) 3)
BULLET //● ● ●
LOWER_LETTER //a. b. c.
LOWER_ROMAN //i ⅱ ⅲ
UPPER_LETTER //A. B. C.
*/

```

数据模型：

List<String>

NumberingRenderData

推荐使用工厂 Numberings 构建列表模型。

### 更多的使用方法可以去看官方文档

## 3、使用例子

### 3.1、上方基础标签的使用示例

创建一个word模板
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d92813a272e725e8d736ec0911f1967e.png)

代码示例

```java
    public void word2() throws Exception {

        HashMap<String, Object> result = new HashMap<>();
        // 名称
        result.put("name", new TextRenderData("000000", "小叶")); //黑色
        // 性别切换颜色
        result.put("sex", new TextRenderData("269846", "男")); //绿色
        // 头像 100x100
        result.put("avatar", new PictureRenderData(100, 100, ".png", BytePictureUtils.getUrlBufferedImage("http://kodo.yelingfa.top/lyblog/static/xx.png")));
        // 技能
        RowRenderData header = RowRenderData.build(new TextRenderData("269846", "技能"), new TextRenderData("269846", "水平"));
        RowRenderData row0 = RowRenderData.build("针灸学", "低");
        RowRenderData row1 = RowRenderData.build("神农本草经", "无");
        RowRenderData row2 = RowRenderData.build("皇帝内经", "无");
        RowRenderData row3 = RowRenderData.build("小六壬", "低");
        result.put("skill", new MiniTableRenderData(header, Arrays.asList(row0, row1, row2,row3)));
        // 优点
        result.put("advantage", new NumbericRenderData(new ArrayList<TextRenderData>() {
            {
                add(new TextRenderData("优点11111"));
                add(new TextRenderData("优点22222"));
                add(new TextRenderData("优点33333"));
            }
        }));
        XWPFTemplate template = XWPFTemplate.compile("C:\\Users\\Administrator\\Desktop\\temp2.docx").render(result);
        FileOutputStream out = new FileOutputStream("C:\\Users\\Administrator\\Desktop\\out_简单示例2.docx");
        template.write(out);
        out.flush();
        out.close();
        template.close();
    }

```

最后生成的效果
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4df6ccd8fcb2000ec4b392acf7de5dfb.png)

### 3.2、扩展用法

自定义插件

模板word
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/86ca6647acb1a789685aaba940ce7d39.png)

代码示例

创建自定义插件

```java
public class DetailTablePolicy extends DynamicTableRenderPolicy {

    private static Logger LOG = LoggerFactory.getLogger(MiniTableRenderPolicy.Helper.class);
    /**
     * 设备填充数据所在行数
     */
    int goodsStartRow = 3;
    /**
     * 物料填充数据所在行数
     */
    int laborsStartRow = 3;

    /**
     * 结束行
     */
    int endRow = 0;

    @Override
    public void render(XWPFTable table, Object data) {

        if (null == data) return;
        DetailData detailData = (DetailData) data;
        // 人工费循环渲染
        List<RowRenderData> goods = detailData.getGoods();
        if (CollectionUtils.isNotEmpty(goods)) {
            laborsStartRow = laborsStartRow + goods.size();
            table.removeRow(goodsStartRow);
            // 循环插入行
            for (RowRenderData good : goods) {
                XWPFTableRow insertNewTableRow = table.insertNewTableRow(goodsStartRow);
                for (int j = 0; j < 7; j++) {
                    XWPFTableCell cell = insertNewTableRow.createCell();
                    cell.setVerticalAlignment(XWPFTableCell.XWPFVertAlign.CENTER);
                    CTTcBorders ctTcBorders = cell.getCTTc().addNewTcPr().addNewTcBorders();
                    ctTcBorders.addNewTop().setVal(STBorder.SINGLE);
                    ctTcBorders.addNewBottom().setVal(STBorder.SINGLE);
                    ctTcBorders.addNewLeft().setVal(STBorder.SINGLE);
                    ctTcBorders.addNewRight().setVal(STBorder.SINGLE);
                    table.getRow(goodsStartRow - 1).getCell(j).getCTTc().getTcPr().addNewTcBorders().addNewBottom().setVal(STBorder.SINGLE);
                }
                // 渲染单行人工费数据
                renderRow(table, goodsStartRow, good);
            }
        }

        // 货品明细
        List<RowRenderData> labors = detailData.getLabors();
        if (CollectionUtils.isNotEmpty(labors)) {
            table.removeRow(laborsStartRow);
            endRow = endRow + laborsStartRow + labors.size();
            for (RowRenderData labor : labors) {
                XWPFTableRow insertNewTableRow = table.insertNewTableRow(laborsStartRow);
                for (int j = 0; j < 7; j++) {
                    XWPFTableCell cell = insertNewTableRow.createCell();
                    cell.setVerticalAlignment(XWPFTableCell.XWPFVertAlign.CENTER);
                    CTTcBorders ctTcBorders = cell.getCTTc().addNewTcPr().addNewTcBorders();
                    ctTcBorders.addNewTop().setVal(STBorder.SINGLE);
                    ctTcBorders.addNewBottom().setVal(STBorder.SINGLE);
                    ctTcBorders.addNewLeft().setVal(STBorder.SINGLE);
                    ctTcBorders.addNewRight().setVal(STBorder.SINGLE);
                }
                // 渲染单行货品明细数据
                renderRow(table, laborsStartRow, labor);
            }
        }
        if (CollectionUtils.isNotEmpty(goods)) {
            TableTools.mergeCellsVertically(table, 1, goodsStartRow, goods.size() + goodsStartRow - 1);
        }
        if (CollectionUtils.isNotEmpty(labors)) {
            TableTools.mergeCellsVertically(table, 1, laborsStartRow, labors.size() + laborsStartRow - 1);
        }
        table.getRow(endRow).getCell(0).getCTTc().getTcPr().addNewTcBorders().addNewTop().setVal(STBorder.SINGLE);
    }



    // **** 以下是poi-tl源码部分，源码不符合需求，拉出来小做修改 ****
    public static void renderRow(XWPFTable table, int row, RowRenderData rowData) {
        if (null != rowData && rowData.size() > 0) {
            XWPFTableRow tableRow = table.getRow(row);
            ObjectUtils.requireNonNull(tableRow, "Row " + row + " do not exist in the table");
            TableStyle rowStyle = rowData.getRowStyle();
            List<CellRenderData> cellList = rowData.getCellDatas();
            XWPFTableCell cell = null;

            for(int i = 0; i < cellList.size(); ++i) {
                cell = tableRow.getCell(i);
                if (null == cell) {
                    LOG.warn("Extra cell data at row {}, but no extra cell: col {}", row, cell);
                    break;
                }

                renderCell(cell, (CellRenderData)cellList.get(i), rowStyle);
            }

        }
    }


    public static void renderCell(XWPFTableCell cell, CellRenderData cellData, TableStyle rowStyle) {
        TableStyle cellStyle = null == cellData.getCellStyle() ? rowStyle : cellData.getCellStyle();
        if (null != cellStyle && null != cellStyle.getBackgroundColor()) {
            cell.setColor(cellStyle.getBackgroundColor());
        }

        TextRenderData renderData = cellData.getRenderData();
        String cellText = renderData.getText();
        if (!StringUtils.isBlank(cellText)) {
            CTTc ctTc = cell.getCTTc();
            CTP ctP = ctTc.sizeOfPArray() == 0 ? ctTc.addNewP() : ctTc.getPArray(0);
            XWPFParagraph par = new XWPFParagraph(ctP, cell);
            StyleUtils.styleTableParagraph(par, cellStyle);
            String text = renderData.getText();
            String[] fragment = text.split("\\n", -1);
            if (fragment.length <= 1) {
                XWPFRun run = par.createRun();
                run.setFontSize(12);
                run.setFontFamily("宋体");
                com.deepoove.poi.policy.TextRenderPolicy.Helper.renderTextRun(run, renderData);
            } else {
                for(int j = 0; j < fragment.length; ++j) {
                    if (0 != j) {
                        par = cell.addParagraph();
                        StyleUtils.styleTableParagraph(par, cellStyle);
                    }

                    XWPFRun run = par.createRun();
                    run.setFontSize(12);
                    run.setFontFamily("宋体");
                    StyleUtils.styleRun(run, renderData.getStyle());
                    run.setText(fragment[j]);
                }
            }

        }
    }
}

```

test方法

```java
public void word() throws Exception {
        // 行样式
        TableStyle rowStyle = new TableStyle();
        rowStyle.setAlign(STJc.CENTER);

        DynamicFormForConstructionVo vo = equipmentFormService.getConstructionById(2912);

        HashMap<Object, Object> result = new HashMap<>();
        result.put("city",vo.getCityName());
        result.put("merchant",vo.getMerchantName());
        String[] split = vo.getMarketerName().split("-");
        result.put("user",split.length > 1 ? split[1] : vo.getMarketerName());
        DetailData detailData = new DetailData();
        List<RowRenderData> equipments = new ArrayList<>();
        List<RowRenderData> materials = new ArrayList<>();
        Integer num = 1;
        for (DynamicFormForConstructionVo.Equipment equipment : vo.getEquipments()) {
            List<DynamicFormForConstructionVo.Equipment.Device> devices = equipment.getDevices();
            for (DynamicFormForConstructionVo.Equipment.Device device : devices) {
                StringBuilder sb = new StringBuilder();
                List<DynamicFormForConstructionVo.Equipment.Column> columns = device.getColumns();
                for (DynamicFormForConstructionVo.Equipment.Column column : columns) {
                    sb.append(column.getColumnName()).append("：").append(Objects.isNull(column.getValue()) ? "-" : column.getValue()).append("\n");
                }
                if (sb.length() > 1) {
                    sb.delete(sb.length() - 1, sb.length());
                }
                if (device.getQuantity() > 0) {
                    RowRenderData good = RowRenderData.build(String.valueOf(num++), "设备", equipment.getEquipmentName(), new String(sb), String.valueOf(device.getQuantity()), "  ", "  ");
                    good.setRowStyle(rowStyle);
                    equipments.add(good);
                }
            }
        }
        for (DynamicFormForConstructionVo.Equipment equipment : vo.getMaterials()) {
            List<DynamicFormForConstructionVo.Equipment.Device> devices = equipment.getDevices();
            for (DynamicFormForConstructionVo.Equipment.Device device : devices) {
                StringBuilder sb = new StringBuilder();
                List<DynamicFormForConstructionVo.Equipment.Column> columns = device.getColumns();
                for (DynamicFormForConstructionVo.Equipment.Column column : columns) {
                    sb.append(column.getColumnName()).append("：").append(Objects.isNull(column.getValue()) ? "-" : column.getValue()).append("\n");
                }
                if (sb.length() > 1) {
                    sb.delete(sb.length() - 1, sb.length());
                }
                if (device.getQuantity() > 0) {
                    RowRenderData material = RowRenderData.build(String.valueOf(num++), "物料", equipment.getEquipmentName(), new String(sb), String.valueOf(device.getQuantity()), "  ", "  ");
                    material.setRowStyle(rowStyle);
                    materials.add(material);
                }
            }
        }
        Collections.reverse(equipments);
        Collections.reverse(materials);
        detailData.setGoods(equipments);
        detailData.setLabors(materials);
        result.put("table_detail",detailData);

        Configure config = Configure.newBuilder().customPolicy("table_detail", new DetailTablePolicy()).build();
        XWPFTemplate template = XWPFTemplate.compile("C:\\Users\\Administrator\\Desktop\\temp1.docx", config).render(result);
        FileOutputStream out = new FileOutputStream("C:\\Users\\Administrator\\Desktop\\out_配置清单2.docx");
        template.write(out);
        out.flush();
        out.close();
        template.close();
    }

```

最终效果

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d0726e024865289301b234b1afd6a3cc.png)
