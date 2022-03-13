---
title: 修改LitJson以支持int类型的字典Key
date: 2022-01-17 21:44:17
tags:
  - Json
categories: 
  - 程序
  - 踩坑记录
description: 一次对LitJson源码修改的记录
---

# 前言

前几天因为项目需求，需要使用Json来做某个功能的数据的序列化，但不能引入新的第三方库，所以只能使用项目里已有的LitJson库

然而后面要用到int类型作为字典Key，但是LitJson又不支持这种做法，所以只能自己想办法对其源码进行修改



# 源码修改

要做到支持字典key为int类型，只需要对JsonMapper.cs文件进行修改即可

总共需要修改3个方法



## AddObjectMetadata

在AddObjectMetadata方法中，修改

```csharp
foreach (PropertyInfo p_info in type.GetProperties())
{
    if (p_info.Name == "Item")
    {
        //省略...

        //字典key除了是字符串还可以是int
        if (parameters[0].ParameterType == typeof(string) || parameters[0].ParameterType == typeof(int))
            data.ElementType = p_info.PropertyType;

        continue;
    }

    //省略...
}
```

增加对Object的Metadata的int类型key的识别判断



## ReadValue

在ReadValue方法中，修改

```csharp
else if (reader.Token == JsonToken.ObjectStart)
{
    AddObjectMetadata(value_type);
    ObjectMetadata t_data = object_metadata[value_type];

    instance = Activator.CreateInstance(value_type);

    //字典key是否为int?
    bool isIntKey = t_data.IsDictionary && value_type.GetGenericArguments()[0] == typeof(int);

    while (true)
    {
        //省略...

        if (t_data.Properties.ContainsKey(property))
        {

            //省略...

        }
        else
        {
            //省略...

            //处理字典key为int的情况
            if (isIntKey)
            {
                ((IDictionary)instance).Add(Convert.ToInt32(property), ReadValue(t_data.ElementType, reader));
            }
            else
            {
                ((IDictionary)instance).Add(property, ReadValue(t_data.ElementType, reader));
            }


        }

    }

}
```

增加在读取json文本时对字典int类型key的检查



## WriteValue

在WriteValue方法中，修改

```csharp
if (obj is IDictionary)
{
    writer.WriteObjectStart();
    foreach (DictionaryEntry entry in (IDictionary)obj)
    {

        if (entry.Key is string key)
        {
            writer.WritePropertyName(key);
            WriteValue(entry.Value, writer, writer_is_private,
                depth + 1);
        }
        else
        {
            //处理字典key为int的情况
            writer.WritePropertyName(entry.Key.ToString());
            WriteValue(entry.Value, writer, writer_is_private,
                depth + 1);
        }
    }
    writer.WriteObjectEnd();

    return;
}

```

增加在写入时对非string类型的key的支持

