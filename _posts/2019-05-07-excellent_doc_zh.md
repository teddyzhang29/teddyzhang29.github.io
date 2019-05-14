---
layout: post
title: "ExcelLENT - 文档"
subtitle: "Unity导表插件"
description: "ExcelLENT 是为游戏设计的导表插件."
tags: [Unity,AssetStore,Plugin,Excel]
category: "ExcelLent"
hide: true
---

<!-- TOC -->

- [ExcelLENT - 文档](#excellent---文档)
    - [1. 表头声明](#1-表头声明)
    - [2. 字段声明](#2-字段声明)
        - [2.1 字段类型](#21-字段类型)
        - [2.2 简单类型](#22-简单类型)
        - [2.3 列表](#23-列表)
        - [2.4 自定义类型](#24-自定义类型)
        - [2.5 嵌套](#25-嵌套)
    - [3. 序列化](#3-序列化)
    - [3.1 简单类型](#31-简单类型)
    - [3.2 列表](#32-列表)
    - [3.3 自定义类型](#33-自定义类型)
    - [4. 导表](#4-导表)

<!-- /TOC -->

# ExcelLENT - 文档

## 1. 表头声明

ExcelLENT会遍历指定目录中的所有Excel文件，若符合条件则会导出该配置表。 
首先需要在任意一行的第一列写上```[ExcelLENT]```来标记该配置表需要被导出。 在同一行的第二列写该配置表的名称，这个名称会被应用于生成的结构类型和序列化文件。  
接着，下一行声明主键，每一个单元格声明一个主键。主键可以用逗号分隔来表示联合主键。  
再下一行是保留行，用于将来新增功能。  
以上就完成了表头的声明。

![head_definition]

## 2. 字段声明

接下来用三行进行字段声明。  
第一行是字段类型声明，第二行是字段名称，第三行是字段描述。

![field_definition]

### 2.1 字段类型

字段类型主要包括三种:```简单类型```,```列表```，```自定义类型```。  

产生式为：  
field -> objField:id | listField:id | simpleField:id  
objField -> obj{field objFieldRemain}  
objFieldRemain -> ;field objFieldRemain | ε  
listField -> list{objField} | list{listField} | list{simpleField}  
simpleField -> int | float | double | bool | string  

### 2.2 简单类型

简单类型包括：```int```, ```float```, ```double```, ```bool```, ```string```。  

### 2.3 列表

列表可以包含一个子类型， 例如：```list{int}```表示整数列表。 ```list{string}```表示字符串列表。   
列表也可以嵌套:```list{list{int}}``` 表示整形列表的列表。

### 2.4 自定义类型

自定义类型对应于一个Class，其中可以包含任意个数的子类型，每个子类型的格式为:```类型:名称```,子类型使用分号分隔。  
例如：```obj{float:x; int:y; string:z; list{int}:listField}```生成的对应结构为:  

```c#
public class Obj // Obj is a name for example
{
    public float x;
    public int y;
    public string z;
    public List<int> listField;
}
```

### 2.5 嵌套

列表和自定义类型可以任意嵌套，例如:```list{obj{int:x; list{string}:y; obj{bool:a}:z}}```。

## 3. 序列化

序列化配置表时会根据每个字段的定义去解析内容。

## 3.1 简单类型

简单类型直接将内容转换为对应类型，若转换失败则会抛出异常。

## 3.2 列表

列表类型的解析规则是:```{值1;值2;值3...}```。即使用大括号包围，以分号分隔的值，每个值按照列表的子类型去解析。  
例如定义为:```list{bool}```的列表你可以这么填写内容:```{true; true; false; true}```。  
再例如子类型不为简单类型:```list{obj{int:id; bool:result}}```的内容可以填:{% raw %}```{{1;true};{2;false};{3;true}}```{% endraw %}。

**注意：** 为了简化配置，列表类型的字段最外层的大括号无需自己填写，将由解析器统一添加。  
因此以上2个例子的内容需要改成：```true; true; false; true``` 和 ```{1;true};{2;false};{3;true}```。

## 3.3 自定义类型

自定义类型的解析规则是:```{值1;值2;值3...}```。即使用大括号包围，以分号分隔的值，每个值按照其对应类型去解析。  
例如定义为:```obj{int:id; string:name}```的内容可以这么填写:```{1;alice}```。  
例如嵌套类型:```obj{int:id;obj{float:x;float:y}:position;string:name}```可以这么填写:```{1;{100;100};alice}```。

**注意：** 为了简化配置，自定义类型的字段最外层的大括号无需自己填写，将由解析器统一添加。  
因此以上2个例子的内容需要改成：```1;alice``` 和 ```1;{100;100};alice```。

## 4. 导表

使用```Tools->BBGo->Export Excel```导出配置表。 可以打开```ExcelLENTUtility```查阅代码。

```c#
public static void ExportExcel()
{
    Parser parser = new Parser();
    parser.Parse(new ParseParam()
    {
        ExcelDir = "Assets/BBGo/ExcelLENT/Demo/Excel",
        Logger = new UnityLogger(),
        Serializations = new SerializationParam[]
        {
            new SerializationParam()
            {
                OutDir = "Assets/BBGo/ExcelLENT/Demo/Output/Serialization",
                Serializer = new TupledJsonSerializer(),
            },
        },
        Generations = new GenerationParam[]
        {
            new GenerationParam()
            {
                OutDir = "Assets/BBGo/ExcelLENT/Demo/Output/Generation",
                Generator = new UnityCSharpGenerator(),
            },
        },
    });
    AssetDatabase.Refresh();
    Debug.Log("Export Excel Finished!");
}
```

这里默认的Excel目录是```Assets/BBGo/ExcelLENT/Demo/Excel```，若需要导出其他目录可以修改ExcelDir。