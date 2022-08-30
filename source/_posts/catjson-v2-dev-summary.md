---
title: CatJsonV2开发总结
date: 2022-08-17 14:20:17
tags:
  - Json
  - 序列化
  - 编译原理
categories: 
  - 程序
  - 开发总结
description: 适用于Unity开发者的Json库
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatJsonV2DevSummary/CatJsonV2DevSummary_1.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatJsonV2DevSummary/CatJsonV2DevSummary_1.png
---

# 前言

**建议在阅读本文之前先阅读[CatJson开发总结](http://cathole.top/2021/12/05/catjson-dev-summary/)**



在前几天正式发布了[CatJson](https://github.com/CatImmortal/CatJson)的V2版本，此版本对旧代码进行了大幅度的重构，使其拥有了更清晰简洁的结构与更方便的扩展性

代码结构如下：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatJsonV2DevSummary/CatJsonV2DevSummary_1.png)

- Attributes：包含内置的特性标签，主要用于忽略字段/属性和处理默认值
- Formatter：包含各种类型对应的Json格式化器的实现
- JsonObject：包含通用Json对象与Json值
- Lexer：包含用于读取Json词法单元的Json词法分析器
- MetaData：包含用于反射处理的类型元数据
- Misc：包含各类Util与Helper
- JsonParser.cs：CatJson的序列化/反序列化调用入口



# 格式化器

V2版本相对于V1最大的重构即是将对所有类型的Json序列化/反序列化行为都统一抽象为了**Formatter**进行处理



`JsonParser`中的相关代码如下：

```csharp
    /// <summary>
    /// Json解析器
    /// </summary>
    public static class JsonParser
    {
        //省略...
        
        private static NullFormatter nullFormatter = new NullFormatter();
        private static EnumFormatter enumFormatter = new EnumFormatter();
        private static ArrayFormatter arrayFormatter = new ArrayFormatter();
        private static ReflectionFormatter reflectionFormatter = new ReflectionFormatter();
        private static PolymorphicFormatter polymorphicFormatter = new PolymorphicFormatter();
        
        /// <summary>
        /// Json格式化器字典
        /// </summary>
        private static readonly Dictionary<Type, IJsonFormatter> formatterDict = new Dictionary<Type, IJsonFormatter>()
        {
            //基元类型
            {typeof(bool), new BooleanFormatter()},
            
            {typeof(byte), new ByteFormatter()},
            {typeof(sbyte), new SByteFormatter()},
            
            {typeof(short), new Int16Formatter()},
            {typeof(ushort), new UInt16Formatter()},
            
            {typeof(int), new Int32Formatter()},
            {typeof(uint), new UInt32Formatter()},
            
            {typeof(long), new Int64Formatter()},
            {typeof(ulong), new UInt64Formatter()},
            
            {typeof(float), new SingleFormatter()},
            {typeof(double), new DoubleFormatter()},
            {typeof(decimal), new DecimalFormatter()},
            
            {typeof(char), new CharFormatter()},
            {typeof(string), new StringFormatter()},
            
            //容器类型
            {typeof(List<>), new ListFormatter()},
            {typeof(Dictionary<,>), new DictionaryFormatter()},
            
            //Json通用对象类型
            {typeof(JsonObject), new JsonObjectFormatter()},
            {typeof(JsonValue), new JsonValueFormatter()},
            
            //Unity特有类型
            {typeof(Hash128), new Hash128Formatter()},
            {typeof(Vector2),new Vector2Formatter()},
            {typeof(Vector3),new Vector3Formatter()},
            {typeof(Vector4),new Vector4Formatter()},
            {typeof(Quaternion),new QuaternionFormatter()},
            {typeof(Color),new ColorFormatter()},
            {typeof(Bounds),new BoundsFormatter()},
            {typeof(Rect),new RectFormatter()},
            {typeof(Keyframe),new KeyFrameFormatter()},
            
            //其他
            {Type.GetType("System.RuntimeType,mscorlib"),new RuntimeTypeFormatter()},  //Type类型的变量其对象一般为RuntimeType类型，但是不能直接typeof(RuntimeType)，只能这样了
            {typeof(DateTime),new DateTimeFormatter()},
        };

        /// <summary>
        /// 添加自定义的Json格式化器
        /// </summary>
        public static void AddCustomJsonFormatter(Type type, IJsonFormatter formatter)
        {
            formatterDict[type] = formatter;
        }
        
        //省略...
        
        
        /// <summary>
        /// 将指定类型的对象序列化为Json文本
        /// </summary>
        internal static void InternalToJson(object obj, Type type, Type realType = null, int depth = 1,bool checkPolymorphic = true)
        {
             //省略...

            if (!realType.IsGenericType)
            {
                if (formatterDict.TryGetValue(realType, out IJsonFormatter formatter))
                {
                    //使用通常的formatter处理
                    formatter.ToJson(obj,type,realType, depth);
                    return;
                }
            }
            else
            {
                if (formatterDict.TryGetValue(realType.GetGenericTypeDefinition(), out IJsonFormatter formatter))
                {
                    //使用泛型类型formatter处理
                    formatter.ToJson(obj,type,realType,depth);
                    return;
                }
            }

            
#if FUCK_LUA
            if (type is ILRuntime.Reflection.ILRuntimeType ilrtType && ilrtType.ILType.IsEnum)
            {
                //热更层枚举 使用int formatter处理
                formatterDict[typeof(int)].ToJson(obj, type, realType,depth);
                return;
            }
#endif
            
            if (obj is Enum e)
            {
                //使用枚举formatter处理
                enumFormatter.ToJson(e, type, realType, depth);
                return;
            }
            
            if (obj is Array array)
            {
                //使用数组formatter处理
                arrayFormatter.ToJson(array,type,realType, depth);
                return;
            }
            
            
            //使用反射formatter处理
            reflectionFormatter.ToJson(obj,type,realType,depth);
        }
        
        //省略...
        
        /// <summary>
        /// 将Json文本反序列化为指定类型的对象
        /// </summary>
        internal static object InternalParseJson(Type type,Type realType = null,bool checkPolymorphic = true)
        {
             //省略...
            
            object result;
            
             //省略...
            else if (formatterDict.TryGetValue(realType, out IJsonFormatter formatter))
            {
                //使用通常的formatter处理
                result = formatter.ParseJson(type, realType);
            }
            else if (realType.IsGenericType && formatterDict.TryGetValue(realType.GetGenericTypeDefinition(), out formatter))
            {
                //使用泛型类型formatter处理
                result = formatter.ParseJson(type,realType);
            }
#if FUCK_LUA
            else if (type is ILRuntime.Reflection.ILRuntimeType ilrtType && ilrtType.ILType.IsEnum)
            {
                //热更层枚举 使用int formatter处理
                result = formatterDict[typeof(int)].ParseJson(type, realType);
            }
#endif
            else if (realType.IsEnum)
            {
                //使用枚举formatter处理
                result = enumFormatter.ParseJson(type, realType);
            }
            else if (realType.IsArray)
            {
                //使用数组formatter处理
                result = arrayFormatter.ParseJson(type,realType);
            }
            else
            {
                //使用反射formatter处理
                result = reflectionFormatter.ParseJson(type,realType);
            }

            if (result is IJsonParserCallbackReceiver receiver)
            {
                //触发序列化结束回调
                receiver.OnParseJsonEnd();
            }

            return result;
        }

       
    }
```



Formatter的接口与基类定义如下：

```csharp
	/// <summary>
    /// Json格式化器接口
    /// </summary>
    public interface IJsonFormatter
    {
        /// <summary>
        /// 将对象序列化为Json文本
        /// </summary>
        void ToJson(object value, Type type, Type realType, int depth);
        
        /// <summary>
        /// 将Json文本反序列化为对象
        /// </summary>
        object ParseJson(Type type, Type realType);
    }
```

```csharp
	/// <summary>
    /// Json格式化器基类
    /// </summary>
    public abstract class BaseJsonFormatter<TValue> : IJsonFormatter
    {
    
        /// <inheritdoc />
        void IJsonFormatter.ToJson(object value, Type type, Type realType, int depth)
        {
            ToJson((TValue)value,type,realType, depth);
        }
    
        /// <inheritdoc />
        object IJsonFormatter.ParseJson(Type type, Type realType)
        {
            return ParseJson(type,realType);
        }
        
        /// <summary>
        /// 将对象序列化为Json文本
        /// </summary>
        public abstract void ToJson(TValue value, Type type, Type realType, int depth);
        
        /// <summary>
        /// 将Json文本反序列化为对象
        /// </summary>
        public abstract TValue ParseJson(Type type, Type realType);
    }
```



想要定义一个类型的Json序列化/反序列化行为，只需要继承基类，实现`ToJson`与`ParseJson`方法，并将其注册到`JsonParser`中即可

以`Vector3`类型为例：

```csharp
	/// <summary>
    /// Vector3类型的Json格式化器
    /// </summary>
    public class Vector3Formatter : BaseJsonFormatter<Vector3>
    {
        /// <inheritdoc />
        public override void ToJson(Vector3 value, Type type, Type realType, int depth)
        {
            TextUtil.Append('{');
            TextUtil.Append(value.x.ToString());
            TextUtil.Append(", ");
            TextUtil.Append(value.y.ToString());
            TextUtil.Append(", ");
            TextUtil.Append(value.z.ToString());
            TextUtil.Append('}');
        }

        /// <inheritdoc />
        public override Vector3 ParseJson(Type type, Type realType)
        {
            JsonParser.Lexer.GetNextTokenByType(TokenType.LeftBrace);
            float x = JsonParser.Lexer.GetNextTokenByType(TokenType.Number).AsFloat();
            JsonParser.Lexer.GetNextTokenByType(TokenType.Comma);
            float y = JsonParser.Lexer.GetNextTokenByType(TokenType.Number).AsFloat();
            JsonParser.Lexer.GetNextTokenByType(TokenType.Comma);
            float z = JsonParser.Lexer.GetNextTokenByType(TokenType.Number).AsFloat();
            JsonParser.Lexer.GetNextTokenByType(TokenType.RightBrace);
            return new Vector3(x,y,z);
        }
    }
```



可以看出`Vector3Formatter`在`ToJson`中通过写入x，y，z三个值完成序列化，在`ParseJson`中，通过Lexer提取x，y，z三个数字值完成反序列化



# 词法分析

Lexer部分所做的修改较少，主要是将提取数字token的方法返回值从string改为了RangeString

```csharp
		/// <summary>
        /// 扫描数字
        /// </summary>
        private RangeString ScanNumber()
        {
            int startIndex = CurIndex;
            
            //第一个字符是0-9或者-
            Next();

            while (
                !(CurIndex >= json.Length)&&
                (
                char.IsDigit(json[CurIndex]) || json[CurIndex] == '.' || json[CurIndex] == '+'|| json[CurIndex] == '-'|| json[CurIndex] == 'e'|| json[CurIndex] == 'E')
                )
            {

                Next();
            }

            int endIndex = CurIndex - 1;

            RangeString rs = new RangeString(json, startIndex, endIndex);

            return rs;
            
        }
```



之所以做出这样的修改，是因为在V1中，`ScanNumber`会将扫描出的数字部分字符串给截取出来直接调用`ToString`，这样不可避免的会产生额外字符串内存分配

而借助高版本C#的`ReadOnlySpan`，可以实现一个0内存分配的数字解析



为了实现这个目标，首先在`RangeString`中添加对应AsSpan方法

```csharp
 	public ReadOnlySpan<char> AsSpan()
        {
            int length = endIndex - startIndex + 1;
            ReadOnlySpan<char> span = source.AsSpan(startIndex, length);
            return span;
        }
```



然后在数字类型的Formatter的`ParseJson`实现中调用此方法进行解析即可，以`Int64Formatter`为例：

```csharp
	/// <summary>
    /// long类型的Json格式化器
    /// </summary>
    public class Int64Formatter : BaseJsonFormatter<long>
    {
        /// <inheritdoc />
        public override void ToJson(long value, Type type, Type realType, int depth)
        {
            TextUtil.Append(value.ToString());
        }

        /// <inheritdoc />
        public override long ParseJson(Type type, Type realType)
        {
            RangeString rs = JsonParser.Lexer.GetNextTokenByType(TokenType.Number);
            return long.Parse(rs.AsSpan());
        }
    }
```

得益于高版本C#中对于大量基础类型的`Parse`方法对`ReadOnlySpan`参数的支持，可以简单方便的实现出中间过程中不产生字符串分配的，0GC数字解析



最后值得一提的是，不仅局限于类型，包括反射处理、多态处理以及null值处理都是通过实现对应的Formatter完成的



# JsonValue

V2版本对表示Json值的`JsonValue`添加了以int为参数与以string为参数的索引器，以及大量以对应类型为参数的构造器与隐式转换运算符重载，加强了其易用性

```csharp
    /// <summary>
    /// Json值
    /// </summary>
    [StructLayout(LayoutKind.Explicit)]
    public class JsonValue
    {
        [FieldOffset(0)]
        public ValueType Type;
        
        [FieldOffset(1)]
        private bool boolean;
        
        [FieldOffset(1)]
        private double number;
        
        [FieldOffset(8)]
        private string str;
        
        [FieldOffset(8)]
        private List<JsonValue> array;
        
        [FieldOffset(8)]
        private JsonObject obj;

        #region 构造方法

        public JsonValue()
        {
            Type = ValueType.Null;
        }
        
        public JsonValue(bool b)
        {
            Type = ValueType.Boolean;
            boolean = b;
        }
        public JsonValue(double d)
        {
            Type = ValueType.Number;
            number = d;
        }
        //省略...

        #endregion
        
        
        public JsonValue this[int index]
        {
            get
            {
                if (Type != ValueType.Array)
                {
                    return default;
                }

                return array[index];
            }
            set
            {
                if (Type != ValueType.Array)
                {
                    return;
                }

                array[index] = value;
            }
        }

        public JsonValue this[string key]
        {
            get
            {
                if (Type != ValueType.Object)
                {
                    return default;
                }

                return obj[key];
            }
            set
            {
                if (Type != ValueType.Object)
                {
                    return;
                }

                obj[key] = value;
            }
        }

        #region 隐式类型转换

        public static implicit operator JsonValue(bool b)
        {
            JsonValue value = new JsonValue(b);
            return value;
        }
        
        public static implicit operator bool(JsonValue value)
        {
            if (value.Type != ValueType.Boolean)
            {
                throw new Exception("JsonValue转换bool失败");
            }
            return value.boolean;
        }

        public static implicit operator JsonValue(double d)
        {
            JsonValue value = new JsonValue(d);
            return value;
        }
        
        public static implicit operator double(JsonValue value)
        {
            if (value.Type != ValueType.Number)
            {
                throw new Exception("JsonValue转换double失败");
            }
            
            return value.number;
        }
        
        //省略...

        #endregion
        
        
        //省略...
    }
```

# 类型元数据

V2中增加了`TypeMetaDataManager`和`TypeMetaData`统一管理类型元数据

`TypeMetaData`主要用于记录每个类型的字段/属性反射信息，以及相应特性标签信息

```csharp
	/// <summary>
    /// 类型的反射元数据
    /// </summary>
    public class TypeMetaData
    {

        /// <summary>
        /// 类型信息
        /// </summary>
        private Type type;

        /// <summary>
        /// 是否序列化此类型下的默认值字段/属性
        /// </summary>
        public bool IsCareDefaultValue { get; }
        
        /// <summary>
        /// 字段信息
        /// </summary>
        public Dictionary<RangeString, FieldInfo> FieldInfos { get; } = new Dictionary<RangeString, FieldInfo>();

        /// <summary>
        /// 属性信息
        /// </summary>
        public Dictionary<RangeString, PropertyInfo> PropertyInfos { get; } = new Dictionary<RangeString, PropertyInfo>();

        /// <summary>
        /// 需要忽略处理的字段/属性名
        /// </summary>
        private HashSet<string> ignoreMembers = new HashSet<string>();
        
        public TypeMetaData(Type type)
        {
            this.type = type;
            IsCareDefaultValue = Attribute.IsDefined(type, typeof(JsonCareDefaultValueAttribute));
            
            //收集字段信息
            FieldInfo[] fis = type.GetFields(TypeMetaDataManager.Flags);
            foreach (FieldInfo fi in fis)
            {
                if (IsIgnoreMember(fi,fi.Name))
                {
                    continue;
                }
                
                FieldInfos.Add(new RangeString(fi.Name), fi);
            }
            
            //收集属性信息
            PropertyInfo[] pis = type.GetProperties(TypeMetaDataManager.Flags);
            foreach (PropertyInfo pi in pis)
            {
                if (IsIgnoreMember(pi,pi.Name))
                {
                    continue;
                }
                
                if (pi.SetMethod != null && pi.GetMethod != null && pi.Name != "Item")
                {
                    //属性必须同时具有get set 并且不能是索引器item
                    PropertyInfos.Add(new RangeString(pi.Name),pi);
                }
            }
            
          
        }

        /// <summary>
        /// 是否需要忽略此字段/属性
        /// </summary>
        private bool IsIgnoreMember(MemberInfo mi,string name)
        {
            if (Attribute.IsDefined(mi, typeof(JsonIgnoreAttribute)))
            {
                return true;
            }

            if (ignoreMembers.Contains(name))
            {
                return true;
            }

            return false;
        }

        /// <summary>
        /// 添加需要忽略的成员
        /// </summary>
        public void AddIgnoreMember(string memberName)
        {
            ignoreMembers.Add(memberName);
        }
        
        public override string ToString()
        {
            return type.ToString();
        }
    }
```

# 多态序列化/反序列化

如果调用`ToJson`时传入的类型与对象实际类型不一致，就会触发多态序列化将真实类型信息写入，而如果反序列化时能够读取到写入的真实类型信息，就会触发多态反序列化

在`JsonParser`中的相关代码如下：

```csharp
    /// <summary>
    /// Json解析器
    /// </summary>
    public static class JsonParser
    {
        //省略...
        
        /// <summary>
        /// 将指定类型的对象序列化为Json文本
        /// </summary>
        internal static void InternalToJson(object obj, Type type, Type realType = null, int depth = 1,bool checkPolymorphic = true)
        { 
            //省略...

            if (realType == null)
            {
                realType = TypeUtil.GetType(obj,type);
            }
            
            if (checkPolymorphic && !TypeUtil.TypeEquals(type,realType))
            {
                //开启了多态序列化检测
                //并且定义类型和真实类型不一致
                //就要进行多态序列化
                polymorphicFormatter.ToJson(obj,type,realType,depth);
                return;;
            }

           //省略...
        }
        
        //省略...
        
        /// <summary>
        /// 将Json文本反序列化为指定类型的对象
        /// </summary>
        internal static object InternalParseJson(Type type,Type realType = null,bool checkPolymorphic = true)
        {
            //省略...

            if (realType == null && !ParserHelper.TryParseRealType(type,out realType))
            {
                //未传入realType并且读取不到realType，就把type作为realType使用
                //这里不能直接赋值type，因为type有可能是一个包装了主工程类型的ILRuntimeWrapperType
                //直接赋值type会导致无法从formatterDict拿到正确的formatter从而进入到reflectionFormatter的处理中
                //realType = type;  
                realType = TypeUtil.CheckType(type);
            }
            
            object result;
            
            if (checkPolymorphic && !TypeUtil.TypeEquals(type,realType))
            {
                //开启了多态检查并且type和realType不一致
                //进行多态处理
                result = polymorphicFormatter.ParseJson(type, realType);
            }
            
           //省略...

            return result;
        }

       
    }
```



无论是多态序列化还是反序列化都是通过`PolymorphicFormatter`来统一处理的，其代码如下：

```csharp
	 /// <summary>
    /// 处理多态序列化/反序列化的Json格式化器
    /// </summary>
    public class PolymorphicFormatter : IJsonFormatter
    {
        /// <summary>
        /// 真实类型key
        /// </summary>
        public const string RealTypeKey = "<>RealType";

        /// <summary>
        /// 对象Json文本key
        /// </summary>
        private const string objectKey = "<>Object";
        
        /// <inheritdoc />
        public void ToJson(object value, Type type, Type realType, int depth)
        {
            TextUtil.AppendLine("{");
                
            //写入真实类型
            TextUtil.Append("\"", depth);
            TextUtil.Append(RealTypeKey);
            TextUtil.Append("\"");
            TextUtil.Append(":");
            TextUtil.Append(TypeUtil.GetTypeString(realType));
                
            TextUtil.AppendLine(",");
                
            //写入对象的json文本
            TextUtil.Append("\"", depth);
            TextUtil.Append(objectKey);
            TextUtil.Append("\"");
            TextUtil.Append(":");
            JsonParser.InternalToJson(value,type,realType,depth + 1,false);
                
            TextUtil.AppendLine(string.Empty);
            TextUtil.Append("}", depth);
        }

        /// <inheritdoc />
        public object ParseJson(Type type, Type realType)
        {
           
            //{
            //"<>RealType":"xxxx"
            //在进入此方法前，已经将这之前的部分提取掉了
            
            //接下来只需要提取下面这部分就行
            //","
            //"<>Object":xxxx
            //}
            
            //跳过,
            JsonParser.Lexer.GetNextTokenByType(TokenType.Comma);
            
            //跳过"<>Object"
            JsonParser.Lexer.GetNextTokenByType(TokenType.String);
            
            //跳过 :
            JsonParser.Lexer.GetNextTokenByType(TokenType.Colon);
            
            //读取被多态序列化的对象的Json文本并反序列化
            object obj = JsonParser.InternalParseJson(type,realType,false);
            
            //跳过}
            JsonParser.Lexer.GetNextTokenByType(TokenType.RightBrace);

            
            
            return obj;
        }
```



对于多态对象的序列化，会通过一组{}将真实类型信息与多态对象的序列化结果包起来，反序列化时则提取出对应信息即可
