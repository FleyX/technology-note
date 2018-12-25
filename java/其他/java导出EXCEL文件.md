---
id="2018-09-22-15-57-00"
title="java导出EXCEL文件"
headWord="最近在java上做了一个EXCEL的导出功能，写了一个通用类，在这里分享分享，该类支持多sheet，且无需手动进行复杂的类型转换."
tags=["reflex", "java","excel","hssfworksheet"]
category="java"
serie="java工具集"
---

## 一、背景

&emsp;&emsp;最近在 java 上做了一个 EXCEL 的导出功能，写了一个通用类，在这里分享分享，该类支持多 sheet，且无需手动进行复杂的类型转换，只需提供三个参数即可：

- `fileName`

  excel 文件名

- `HasMap<String,List<?>> data`

  具体的数据，每个 List 代表一张表的数据，？表示可为任意的自定义对象

- `LinkedHashMap<String,String[][]> headers`

  `Stirng`代表 sheet 名。每个`String[][]`代表一个 sheet 的定义，举个例子如下：

  ```java
  String[][] header = {
      {"field1","参数1"}
      ，{"field2","参数2"}
      ，{"field3","参数3"}
  }
  ```

  其中的 field1，field2，field3 为对象中的属性名，参数 1，参数 2，参数 3 为列名，实际上这个指定了列的名称和这个列用到数据对象的哪个属性。

## 二、怎么用

&emsp;&emsp;以一个例子来说明怎么用，假设有两个类 A 和 B 定义如下：

```java
public class A{
    private String name;
    private String address;
}
public class B{
    private int id;
    private double sum;
    private String cat;
}
```

现在我们通过查询数据库获得了 A 和 B 的两个列表：

```java
List<A> dataA = .....;
List<B> dataB = .....;
```

我们将这两个导出到 excel 中，首先需要定义 sheet：

```java
String[][] sheetA = {
    {"name","姓名"}
    ,{"address","住址"}
}
String[][] sheetB = {
    {"id","ID"}
    ,{"sum","余额"}
    ,{"cat","猫的名字"}
}
```

然后将数据汇总构造一个 ExcelUtil：

```java
String fileName = "测试Excel";
HashMap<String,List<?>> data = new HashMap<>();
//ASheet为表名，后面headers里的key要跟这里一致
data.put("ASheet",dataA);
data.put("BSheet",dataB);
LinkedHashMap<String,String[][]> headers = new LinkedHashMap<>();
headers.put("ASheet",sheetA);
headers.put("BSheet",sheetB);
ExcelUtil excelUtil = new ExcelUtil(fileName,data,headers);
//获取表格对象
HSSFWorkbook workbook = excelUtil.createExcel();
//这里内置了一个写到response的方法（判断浏览器类型设置合适的参数），如果想写到文件也是类似的
workbook.writeToResponse(workbook,request,response);
```

当然通常数据是通过数据库查询的，这里为了演示方便没有从数据库查找。

## 三、实现原理

&emsp;&emsp;这里简单说明下实现过程，从调用`createExcel()`这里开始

#### 1、遍历 headers 创建 sheet

```java
    public HSSFWorkbook createExcel() throws Exception {
        try {
            HSSFWorkbook workbook = new HSSFWorkbook();
            //遍历headers创建表格
            for (String key : headers.keySet()) {
                this.createSheet(workbook, key, headers.get(key), this.data.get(key));
            }
            return workbook;
        } catch (Exception e) {
            log.error("创建表格失败:{}", e.getMessage());
            throw e;
        }
    }
```

将 workbook，sheet 名，表头数据，行数据传入 crateSheet 方法中创建 sheet。

#### 2、创建表头

&emsp;&emsp;表头也就是一个表格的第一行，通常用来对列进行说明

```java
        HSSFSheet sheet = workbook.createSheet(sheetName);
        // 列数
        int cellNum = header.length;
        // 单元行，单元格
        HSSFRow row;
        HSSFCell cell;
        // 表头单元格样式
        HSSFCellStyle columnTopStyle = this.getColumnTopStyle(workbook);
        // 设置表头
        row = sheet.createRow(0);
        for (int i = 0; i < cellNum; i++) {
            cell = row.createCell(i);
            cell.setCellStyle(columnTopStyle);
            String str = header[i][1];
            cell.setCellValue(str);
            // 设置列宽为表头的文字宽度+6个半角符号宽度
            sheet.setColumnWidth(i, (str.getBytes("utf-8").length + 6) * 256);
        }
```

#### 3、插入行数据

&emsp;&emsp;这里是最重要的部分，首先通过数据的类对象获取它的反射属性 Field 类，然后将属性名和 Field 做一个 hash 映射，避免循环查找，提高插入速度，接着通过一个 switch 语句，根据属性类别设值，主要代码如下：

```java
/**
 * 设置单元格,根据fieldName获取对应的Field类，使用反射得到值
 *
 * @param cell 单元格实例
 * @param obj 存有属性的对象实例
 * @param fieldMap  属性名与Field的映射
 * @param fieldName 属性名
 */
private void setCell(HSSFCell cell, Object obj, Map<String, Field> fieldMap, String fieldName) throws Exception {
    //获取该属性的Field对象
    Field field = fieldMap.get(fieldName);
    //通过反射获取属性的值，由于不能确定该值的类型，用下面的判断语句进行合适的转型
    Object value = field.get(obj);
    if (value == null) {
        cell.setCellValue("");
    } else {
        switch (field.getGenericType().getTypeName()) {
        case "java.lang.String":
            cell.setCellValue((String) value);
            break;
        case "java.lang.Integer":
        case "int":
            cell.setCellValue((int) value);
            break;
        case "java.lang.Double":
        case "double":
            cell.setCellValue((double) value);
            break;
        case "java.util.Date":
            cell.setCellValue(this.dateFormat.format((Date) value));
            break;
        default:
            cell.setCellValue(obj.toString());
        }
    }
}
```

完整代码可以到 github 上查看下载，这里就不列出来了。

github 地址：[点击跳转]()
