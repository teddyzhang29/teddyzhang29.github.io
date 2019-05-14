---
layout: post
title: "ExcelLENT - Manual"
subtitle: "Unity导表插件"
description: "ExcelLENT 是为游戏设计的导表插件."
tags: [Unity,AssetStore,Plugin,Excel]
category: "ExcelLent"
hide: true
---

<!-- TOC -->

- [ExcelLENT - Manual](#excellent---manual)
    - [1. Header definition](#1-header-definition)
    - [2. Field definition](#2-field-definition)
        - [2.1 Field Type](#21-field-type)
        - [2.2 Simple type](#22-simple-type)
        - [2.3 List](#23-list)
        - [2.4 Custom Type](#24-custom-type)
        - [2.5 Nesting](#25-nesting)
    - [3. Serialization](#3-serialization)
    - [3.1 Simple type](#31-simple-type)
    - [3.2 List](#32-list)
    - [3.3 Custom Type](#33-custom-type)
    - [4. Export](#4-export)

<!-- /TOC -->

# ExcelLENT - Manual

## 1. Header definition

ExcelLENT will iterate through all the Excel files in the specified directory and export the sheet if it meets the conditions.
Firstly, you need to write ```[ExcelLENT]``` in the first column of any row to mark that the sheet needs to be exported. Write the name of the table in the second column of the same row.
Secondly, the next line declares the primary key, and each cell declares a primary key. Primary keys can be separated by commas to represent a union primary key.
The next line is a reserved line for future additions.
The above completes the definition of the header.

![head_definition]

## 2. Field definition

Next, we use three lines for field definition.
The first line is the field type definition, the second line is the field name, and the third line is the field description.

![field_definition]

### 2.1 Field Type

The field types include three types: ``` simple type ```, ``` list ```, ``` custom type ```.

The production is:
Field -> objField:id | listField:id | simpleField:id
objField -> obj{field objFieldRemain}
objFieldRemain -> ;field objFieldRemain | ε
listField -> list{objField} | list{listField} | list{simpleField}
simpleField -> int | float | double | bool | string

### 2.2 Simple type

Simple types include: ```int```, ```float```, ```double```, ```bool```, ```string```.

### 2.3 List

A list can contain a subtype, for example: ```list{int}``` represents a list of integers. ```list{string}``` represents a list of strings.
Lists can also be nested: ```list{list{int}}``` represents a list of integer list.

### 2.4 Custom Type

A custom type corresponds to a Class, which can contain any number of subtypes. Each subtype has the format: ``` type: name ```, and subtypes are separated by semicolons.
For example: ```obj{float:x; int:y; string:z; list{int}:listField}``` generate structure code:

```c#
Public class Obj // Obj is a name for example
{
    Public float x;
    Public int y;
    Public string z;
    Public List<int> listField;
}
```

### 2.5 Nesting

Lists and custom types can be nested arbitrarily, for example: ```list{obj{int:x; list{string}:y; obj{bool:a}:z}}```.

## 3. Serialization

When the configuration table is serialized, the content is parsed according to the definition of each field.

## 3.1 Simple type

A simple type directly converts the content to a corresponding type, and throws an exception if the conversion fails.

## 3.2 List

The parsing rules for list types are: ```{value 1; value 2; value 3...}```. That is, surrounded by braces, separated by semicolons, each value is parsed according to the subtype of the list.
For example, a list defined as: ```list{bool}``` can fill in the content like this: ```{true; true; false; true}```.
For example, the subtype is not a simple type: ```list{obj{int:id; bool:result}}``` can be filled with: {% raw %}```{{1;true};{2;false};{ 3; true}}```{% endraw %}.

**Note:** To simplify the configuration, the outermost braces of the list type field do not need to be filled in by yourself and will be added by the parser.  
So the contents of the above two examples need to be changed to: ```true; true; false; true``` and ```{1;true}; {2;false};{3;true}```.

## 3.3 Custom Type

The parsing rules for custom types are: ```{value 1; value 2; value 3...}```. That is, surrounded by braces, separated by semicolons, each value is parsed according to its corresponding type.
For example, the definition is: ```obj{int:id; string:name}``` can be filled in like this: ```{1;alice}```.
For example, the nested type: ```obj{int:id;obj{float:x;float:y}:position;string:name}``` can be filled in like this: ```{1;{100;100}; alice}```.

**Note:** To simplify the configuration, the outermost braces of the custom type field do not need to be filled in by yourself and will be added by the parser.  
Therefore, the contents of the above two examples need to be changed to: ```1; alice``` and ```1; {100; 100}; alice```.

## 4. Export

Use ```Tools->BBGo->Export Excel``` to export excels. You can open the ```ExcelLENTUtility``` to check code.

```c#
Public static void ExportExcel()
{
    Parser parser = new Parser();
    parser.Parse(new ParseParam()
    {
        ExcelDir = "Assets/BBGo/ExcelLENT/Demo/Excel",
        Logger = new UnityLogger(),
        Serializations = new SerializationParam[]
        {
            New SerializationParam()
            {
                OutDir = "Assets/BBGo/ExcelLENT/Demo/Output/Serialization",
                Serializer = new TupledJsonSerializer(),
            },
        },
        Generations = new GenerationParam[]
        {
            New GenerationParam()
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

The default Excel directory here is ```Assets/BBGo/ExcelLENT/Demo/Excel```. If you need to export other directories, you can modify ExcelDir.