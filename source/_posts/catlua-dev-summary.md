---
title: CatLua开发总结
date: 2021-11-09 21:24:46
tags: 
  - Lua
  - 编译原理
categories: 程序
description: 基于C#实现的简易Lua解释器
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_1.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_1.png
---

# 前言

前段时间出于对Lua底层机制与编译原理的学习，笔者尝试写了个简易的Lua解释器[CatLua](https://github.com/CatImmortal/CatLua)，主要参考《自己动手实现Lua》（原书是Go语言实现，笔者选择了自己熟悉的C#语言，并且对原书很多地方的代码进行了符合自身审美的重构），其余则参考了《Lua设计与实现》、《Lua源码欣赏》

最终代码结构如下：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_1.png)

- Chunk：包含保存读取到的Lua字节码信息的数据结构
- Compiler：包含词法分析器（Lexer）、语法分析器（Parser）与函数原型编译器（Compiler）
- Instruction：包含所有Lua虚拟机可执行的指令的定义与实现
- LuaStack：包含用于Lua虚拟机进行数据操作、指令逻辑执行的Lua虚拟栈，以及Lua数据类型在C#层的对应实现
- LuaState：包含核心Lua虚拟机，封装了大量对LuaStack的操作
- Operator：包含运算符的定义与实现，包括数学运算、位运算、逻辑运算
- StdLib：包含标准库的实现
- Util：包含各类辅助代码



此文主要用于对CatLua的总结以及各部分实现思路的梳理，因此不会对每一处细节都进行讲解，若欲详细了解可自行到Github下载源码



# Lua字节码读取

在借助Lua官方的编译器的情况下，可以先跳过比较麻烦的编译器实现，而直接从字节码读取开始



## Chunk

一般而言，一个Lua文件就被视为一个Chunk（严格意义上来说，任何一段可被Lua解释器执行的代码就是一个Chunk）

Chunk定义如下：

```csharp
	/// <summary>
    /// lua的二进制字节码chunk
    /// </summary>
    public class Chunk
    {
        /// <summary>
        /// 头信息
        /// </summary>
        public ChunkHeader Header;

        /// <summary>
        /// 主函数的upvalue数量
        /// </summary>
        public byte UpvaluesSize;

        /// <summary>
        /// 主函数
        /// </summary>
        public FuncPrototype MainFunc;

        /// <summary>
        /// 从字节流解码trunk
        /// </summary>
        public static Chunk Undump(byte[] data)
        {
            Chunk chunk = new Chunk();
            ChunkReader reader = new ChunkReader(data);
            chunk.Header = reader.CheckHeader();
            chunk.UpvaluesSize = reader.ReadByte();
            chunk.MainFunc = reader.ReadProto(string.Empty);
            return chunk;
        }
    }
```



## 函数

Chunk被视为自动拥有一个主函数，而在其中定义的任何变量与函数都会被处理为这个主函数内部定义的变量与子函数，因此执行一段Chunk总是从该Chunk的主函数开始执行的，

**这也就是为什么在Lua文件里不用定义“Main”函数，就可以直接执行里面的代码**

函数定义如下：

```csharp
 	/// <summary>
    /// 函数原型
    /// </summary>
    public class FuncPrototype
    {
        /// <summary>
        /// 源文件名
        /// </summary>
        public string Source;

        /// <summary>
        /// 起始行号
        /// </summary>
        public uint LineDefined;

        /// <summary>
        /// 结束行号
        /// </summary>
        public uint LastLineDefined;

        /// <summary>
        /// 固定参数个数
        /// </summary>
        public byte NumParams;

        /// <summary>
        /// 是否有变长参数
        /// </summary>
        public byte IsVararg;

        /// <summary>
        /// 寄存器数量
        /// </summary>
        public byte MaxStackSize;

        /// <summary>
        /// 指令表
        /// </summary>
        public uint[] Code;

        /// <summary>
        /// 常量表
        /// </summary>
        public LuaConstantUnion[] Constants;

        /// <summary>
        /// upvalue表
        /// </summary>
        public UpvalueInfo[] UpvalueInfos;

        /// <summary>
        /// 子函数原型表
        /// </summary>
        public FuncPrototype[] Protos;

        /// <summary>
        /// 行号表（每条指令对应的行号）
        /// </summary>
        public uint[] LineInfo;

        /// <summary>
        /// 局部变量表
        /// </summary>
        public LocalVarInfo[] Locvars;

        /// <summary>
        /// upvalue名列表
        /// </summary>
        public string[] UpvalueNames;

    }
```



可以看出函数的本质就是一系列的“**指令**”，此外FuncPrototype还记录了与此函数相关的信息，如参数个数，局部变量信息等



# 指令

Lua是一种**面向虚拟机的编程语言**（与此相对立的则是**面向硬件的编程语言**，如C/C++），依靠同一套指令集规范在不同实现的虚拟机上运行，达到**跨平台与嵌入宿主语言作为脚本存在**的效果

Lua在5.0前是基于栈的虚拟机，而在5.0开始改为了基于寄存器的虚拟机，二者的主要区别在于：

- 基于栈的虚拟机只能使用push、pop来操作栈顶，因此需要的指令集较大，相同逻辑需要执行的指令较多，但是指令长度较短
- 基于寄存器的虚拟机可以直接对栈进行寻址（比如直接从栈底获取值，把栈顶倒数第n个值复制到栈顶等），所以需要的指令集较小，相同逻辑需要执行的指令较少，但由于需要把地址编码进指令中所以指令长度较长



一般而言，虚拟机在执行指令时，采用的都是将一个巨大的对指令的Switch Case包裹在循环中来不断执行指令，比如Lua官方C版虚拟机：

```C
void luaV_execute (lua_State *L) {
  //.....省略
    
  /* main loop of interpreter */
  for (;;) {
    Instruction i;
    StkId ra;
    vmfetch();
    vmdispatch (GET_OPCODE(i)) {
      vmcase(OP_MOVE) {
        setobjs2s(L, ra, RB(i));
        vmbreak;
      }
      vmcase(OP_LOADK) {
        TValue *rb = k + GETARG_Bx(i);
        setobj2s(L, ra, rb);
        vmbreak;
      }
      vmcase(OP_LOADKX) {
        TValue *rb;
        lua_assert(GET_OPCODE(*ci->u.l.savedpc) == OP_EXTRAARG);
        rb = k + GETARG_Ax(*ci->u.l.savedpc++);
        setobj2s(L, ra, rb);
        vmbreak;
      }
     
        //省略...
      
    }
  }
}
```



CatLua则是采用**表驱动**的写法，将Lua指令与其相关参数及其对应实现逻辑封装为一个InstructionConfig类，并放入数组里，根据指令码ID进行索引（运算符实现也与此类似，后续便不重复了）

Instruction定义如下：

```csharp
 /// <summary>
    /// 指令
    /// </summary>
    public class Instructoin
    {
        public Instructoin(uint code)
        {
            this.code = code;
        }

        //省略...

        /// <summary>
        /// 执行指令
        /// </summary>
        public void Execute(LuaState vm)
        {
            Action<Instructoin, LuaState> function = InstructionConfig.Configs[OpCode].Func;
            if (function != null)
            {
                function(this, vm);
            }
            else
            {
                throw new Exception("指令没有对应的函数实现：" + OpType.ToString());
            }
        }
    }
```



InstructionConfig定义如下：

```csharp
 	/// <summary>
    /// 指令配置
    /// </summary>
    public class InstructionConfig
    {
        public InstructionConfig(byte testFlag, byte setAFlag, OpArgType argBType, OpArgType argCType, OpMode opMode, OpCodeType type, Action<Instructoin, LuaState> func = null)
        {
            TestFlag = testFlag;
            SetAFlag = setAFlag;
            ArgBType = argBType;
            ArgCType = argCType;
            OpMode = opMode;
            Type = type;
            Func = func;
        }

        //省略...

        /// <summary>
        /// 操作码类型
        /// </summary>
        public OpCodeType Type;

        /// <summary>
        /// 指令的函数实现
        /// </summary>
        public Action<Instructoin, LuaState> Func;

        /// <summary>
        /// 所有指令的指令配置
        /// </summary>
        public static InstructionConfig[] Configs =
        {
            new InstructionConfig(0,1,OpArgType.R,OpArgType.N,OpMode.IABC,OpCodeType.Move,InstructionFuncs.MoveFunc),
            new InstructionConfig(0,1,OpArgType.K,OpArgType.N,OpMode.IABx,OpCodeType.LoadK,InstructionFuncs.LoadKFunc),
            new InstructionConfig(0,1,OpArgType.N,OpArgType.N,OpMode.IABx,OpCodeType.LoadKX,InstructionFuncs.LoadKXFunc),
            new InstructionConfig(0,1,OpArgType.U,OpArgType.U,OpMode.IABC,OpCodeType.LoadBool,InstructionFuncs.LoadBoolFunc),
			//省略...

      
    }
```



这样对指令的执行就变成了在循环中查表

```csharp
while (true)
{
	//不断取出指令执行
	Instructoin i = new Instructoin(Fetch());
	i.Execute(this);
}
```



# Lua数据类型

在整个虚拟机运行过程中，都是基于对Lua虚拟栈的操作来实现的，而栈中所保存的则是Lua内置的数据类型

Lua中有多种数据类型，如nil、布尔、数字、Table、函数等，这些数据类型在CatLua中都被统一封装为LuaDataUnion，其定义如下：

```csharp
    /// <summary>
    /// Lua数据的模拟Union
    /// </summary>
    public struct LuaDataUnion : IEqualityComparer<LuaDataUnion>
    {
        //省略...
        
        public LuaDataType Type
        {
            get;
            private set;
        }


        public bool Boolean
        {
            get;
            private set;
        }


        public long Integer
        {
            get;
            private set;
        }


        public double Number
        {
            get;
            private set;
        }


        public string Str
        {
            get;
            private set;
        }


        public LuaTable Table
        {
            get;
            private set;
        }


        public Closure Closure
        {
            get;
            private set;
        }

        //省略...


    }
```



其中比较重要的有Table和Closure（闭包），接下来将着重讲解这两者



## Table

Table作为Lua中唯一的数据容器，可以说是十分万能的存在，其既能表现出数组的特性，又能表现出字典的特性，因此在CatLua中便直接使用了C#自带的数组和字典来实现Table，同时嵌套了一个Table来表示元表



LuaTable定义如下：

```csharp
 	/// <summary>
    /// Lua中的Table数据结构
    /// </summary>
    public class LuaTable
    {
    	public LuaTable(int arrSize = 0,int dictSize = 0)
        {

            arr = new List<LuaDataUnion>(arrSize);
            for (int i = 0; i < arrSize; i++)
            {
                arr.Add(default);
            }

            dict = new Dictionary<LuaDataUnion, LuaDataUnion>(dictSize);
        }

        /// <summary>
        /// 数组部分
        /// </summary>
        private List<LuaDataUnion> arr;

        /// <summary>
        /// 字典部分
        /// </summary>
        private Dictionary<LuaDataUnion, LuaDataUnion> dict;

        /// <summary>
        /// 元表
        /// </summary>
        public LuaTable MetaTable;
        
        //省略...
    }
```



元表，即**MetaTable**，在Lua中起到的作用相当于其他语言中运算符重载功能，通过设置与之对应的元方法，开发者可以自定义对Table的一些操作，比如赋值（Index）、取值(newindex)、调用(call)等



## Closure（闭包）与Upvalue

闭包指的是使用了其外层函数局部变量的内部函数，而Upvalue则是这个被引用的局部变量

```lua

local func1 = function()

    local a = 1

    local func2 = function()

        a = 2

    end
    
end
```

上面的func2就是一个闭包，而变量a就是一个Upvalue

### 闭包

闭包的定义如下：

```csharp
    /// <summary>
    /// 闭包
    /// </summary>
    public class Closure
    {
        public Closure(FuncPrototype proto)
        {
            Proto = proto;
            CSFunc = null;

            if (Proto.UpvalueInfos.Length > 0)
            {
                Upvalues = new Upvalue[Proto.UpvalueInfos.Length];
            }
        }

        public Closure(Func<LuaState, int, int> csFunc,int upvalueNum = 0)
        {
            Proto = null;
            CSFunc = csFunc;

            if (upvalueNum > 0)
            {
                Upvalues = new Upvalue[upvalueNum];
            }
        }

        /// <summary>
        /// Lua函数闭包
        /// </summary>
        public FuncPrototype Proto;

        /// <summary>
        /// C#函数闭包
        /// </summary>
        public Func<LuaState, int,int> CSFunc;

        /// <summary>
        /// 捕获到的Upvalue列表
        /// </summary>
        public Upvalue[] Upvalues;
    }
```



可以看到闭包内部还被分为了两种：Lua函数闭包与C#函数闭包

因为在虚拟机实现上**所有函数都是以闭包的形式存在的**（即便是Chunk主函数也捕获到了名为"_Env"的Upvalue），所以如果要在Lua代码中调用C#层定义的函数，就需要将其封装为**可供虚拟机调用的函数形式**，然后以闭包的方式传入

如print函数：

```csharp
 private static int Print(LuaState vm, int argsNum)
        {
            string s = string.Empty;
            s += "Lua Print:";
            LuaDataUnion[] datas = vm.PopN(argsNum);
            for (int i = 0; i < argsNum; i++)
            {
                s += datas[i].ToString();
                s += '\t';
            }

            Debug.Log(s);

            return 0;
        }
```

C#闭包其固定参数为LuaState和参数个数，返回值则为该函数的返回值数量，如print函数无返回值则其C#闭包返回0



### Upvalue

之所以会将被使用的局部变量称为Upvalue，是因为闭包会延长这个局部变量的生命周期，使其即便离开作用域仍然有可能存活



Upvalue的定义如下：

```csharp
public class Upvalue
    {
        public Upvalue(LuaDataUnion value,bool isOpen = false,int globalStackIndex = 0)
        {
            Value = value;
            IsOpen = isOpen;
            GlobalStackIndex = globalStackIndex;
        }

        /// <summary>
        /// 是否为开放状态（引用到的Lua值是否在栈中）
        /// </summary>

        public bool IsOpen;

        public LuaDataUnion Value
        {
            get;
            private set;
        }

        /// <summary>
        /// 引用到的Lua值在栈中的全局索引
        /// </summary>
        public int GlobalStackIndex
        {
            get;
            private set;
        }


        /// <summary>
        /// 修改引用到的Lua值
        /// </summary>
        public void SetValue(LuaDataUnion value,LuaState vm)
        {

            Value = value;

            if (IsOpen)
            {
                //upvalue处于开放状态 意味着引用的Lua值在栈中，需要一并更新栈
                vm.Push(Value);
                vm.PopAndCopy(GlobalStackIndex);
            }
        }
    }
```



在Lua中Upvalue是被区分为**开放状态**与**闭合状态**的

- 开放状态意味着此时此Upvalue所代表的局部变量仍然存活在栈中，修改Upvalue也需要修改栈中的局部变量
- 闭合状态意味着那个局部变量已经离开作用域了

在Lua官方C版虚拟机的实现中，若Upvalue处于开放状态则使用指针来引用那个局部变量，这样可以做到Upvalue与栈中局部变量的一并修改，而一旦闭合了，就将局部变量复制一份进来并将指针置空，这样局部变量就成为了此Upvalue独占的存在

由于C#通常情况下没有指针，所以只能记录开放状态下的局部变量的栈中索引，然后在被修改时一并对栈中数据进行修改



# Lua虚拟栈

有了对Lua数据类型的定义就可以着手于Lua虚拟栈了

其定义如下：

```csharp
    /// <summary>
    /// Lua虚拟栈
    /// </summary>
    public class LuaStack
    {
        public LuaStack (int size)
        {
            stack = new LuaDataUnion[size];
            Top = -1;
        }

        /// <summary>
        /// 存放Lua数据的可索引的栈
        /// </summary>
        private LuaDataUnion[] stack;

        /// <summary>
        /// 栈顶索引
        /// </summary>
        public int Top;

		//省略...

        /// <summary>
        /// 往栈顶压入值
        /// </summary>
        public void Push(LuaDataUnion data)
        {
            //省略...
        }



        /// <summary>
        /// 从栈顶弹出值
        /// </summary>
        public LuaDataUnion Pop()
        {
            //省略...
        }
        

        /// <summary>
        /// 根据索引从栈中获取值
        /// </summary>
        public LuaDataUnion Get(int index)
        {
            //省略...
        }

        /// <summary>
        /// 根据索引在栈中设置值
        /// </summary>
        public void Set(int index, LuaDataUnion value)
        {
            //省略...
        }


		//省略...
     
    }

}
```

Lua虚拟栈的本质就是使用LuaDataUnion数组来模拟栈，并记录栈顶位置，然后封装了对栈的基本操作



# LuaState

LuaState可理解为Lua虚拟机对象，其主要持有一个LuaStack作为全局虚拟栈，以及一个LuaTable作为全局注册表，定义如下：

```csharp
    /// <summary>
    /// Lua解释器核心
    /// </summary>
    public partial class LuaState
    {
        public LuaState(int size)
        {
            globalStack = new LuaStack(size);

            //将全局环境表放入注册表
            registry[Constants.GlobalEnvKey] = Factory.NewTable(new LuaTable());

            //省略...
        }

        /// <summary>
        /// 全局Lua虚拟栈
        /// </summary>
        private LuaStack globalStack;


        /// <summary>
        /// 全局Lua注册表
        /// </summary>
        private LuaTable registry = new LuaTable();


		//省略...

        /// <summary>
        /// 加载Lua字节码
        /// </summary>
        public int LoadChunk(byte[] bytes, string chunkName)
        {
            Chunk chunk = Chunk.Undump(bytes);
            LoadMainFunc(chunk.MainFunc);
            return 0;
        }

        /// <summary>
        /// 加载主函数原型,将其实例化为闭包，压入栈顶
        /// </summary>
        private void LoadMainFunc(FuncPrototype mainFunc)
        {
            Closure c = new Closure(mainFunc);
            Push(c);

            if (c.Proto.UpvalueInfos.Length > 0)
            {
                //设置全局环境表到入口函数的upvalue中
                LuaDataUnion g = registry[Constants.GlobalEnvKey];
                c.Upvalues[0] = new Upvalue(g);
            }

        }

        //省略...


    }
```



此外还封装了大量的对LuaStack的操作，如调用函数，设置元表，开启标准库等

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_2.png)



# 函数调用

函数调用部分可谓是整个CatLua中相当麻烦的一部分，笔者在反复测试重构后，采取了基于栈帧的函数调用流程



所谓栈帧，指的是**对栈的一部分切片的抽象**，每次函数调用都会产生一个栈帧来处理此函数中对栈数据的操作

```
local func1 = function()
    func2()
end

local func2 = function()
    func3()
end

local func3 = function()
end

func1()  
```

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_3.png)

同时，在调用函数时还会区分主调栈帧和被调栈帧，比如在func1中调用func2时，func1的栈帧就是主调栈帧，func2的栈帧就是被调栈帧



栈帧的定义如下：

```csharp
/// <summary>
    /// 函数调用栈帧
    /// </summary>
    public class FuncCallFrame
    {

        public FuncCallFrame(Closure closure = null, int bottom = 0)
        {
            Closure = closure;
            Bottom = bottom;

            int size = 0;
            if (Closure != null && Closure.Proto != null)
            {
                size = Closure.Proto.MaxStackSize;
            }
            ReserveRegisterMaxIndex = (Bottom - 1) + size; 
        }


        /// <summary>
        /// 变长参数
        /// </summary>
        public LuaDataUnion[] VarArgs;

        /// <summary>
        /// 指令索引
        /// </summary>
        public int PC;

        /// <summary>
        /// 前一个函数的调用栈帧
        /// </summary>
        public FuncCallFrame Prev;

        /// <summary>
        /// 闭包
        /// </summary>
        public Closure Closure
        {
            get;
            private set;
        }

        /// <summary>
        /// 栈帧的预留寄存器区域最大索引
        /// </summary>
        public int ReserveRegisterMaxIndex
        {
            get;
            private set;
        }

        /// <summary>
        /// 栈帧的栈底索引
        /// </summary>
        public int Bottom
        {
            get;
            private set;
        }

    }
```



因为一个LuaState只有一个全局的LuaStack，因为每个栈帧就需要记录下当前栈帧的栈底，方便弹出栈帧时正确恢复栈顶，同时还要记录下前一个函数调用栈帧，方便进行恢复操作



此外还需要注意的是ReserveRegisterMaxIndex这个值，每个栈帧都有N个为寄存器预留的位置，因此**在压入新栈帧时就不能占用这些预留的位置**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_4.png)



同时可能该栈帧因为操作的数据过多导致栈顶范围溢出了ReserveRegisterMaxIndex，所以**在压入新栈帧时也不能占用这些溢出的位置**

在考虑上面两个限制后，就需要通过`Max(ReserveRegisterMaxIndex + 1,Top +1)`来计算出新栈帧的栈底位置，然后调整栈顶到新栈帧的栈底（保证后续压入的函数参数从新栈帧的Bottom开始）



以Lua函数调用为例（C#函数调用流程则是Lua函数调用流程的简化版本），在CatLua中将整个调用流程分为了3个阶段：

1. 调用Lua函数前
2. 执行Lua函数调用
3. Lua函数调用后

```csharp
PreLuaFuncCall(argsNum);
ExcuteLuaFuncCall();
PostLuaFuncCall(resultNum);
```

其中argsNum为调用函数的参数数量，resultNum为返回值数量



接下来会以下述代码中func1调用func2的流程来进行讲解

```lua
local function func1(){
    func2(1)
}

local function func2(num){
    return 2
}

func1()
```





## PreLuaFuncCall

在PreLuaFuncCall前，会先将要调用的函数和参数从指定位置复制并压入到主调栈帧的栈顶，并根据Call指令的指令参数和栈顶位置计算出函数参数的数量（如果有变长参数，那么实际参数数量也是在此时被计算出）

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_5.png)



进入PreLuaFuncCal后，要从主调栈帧的栈顶弹出函数与参数，并为被调函数创建新栈帧，压入新栈帧和参数

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_6.png)



接下来再次修改Top，将其指向ReserveRegisterMaxIndex后的位置，**保证后续压入新值不会占用预留的寄存器**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_7.png)

接下来如果还有变长参数，就会将变长参数收集到栈帧中

PreLuaFuncCall完整代码如下：

```csharp
        /// <summary>
        /// 为Lua函数调用作准备，将被调函数和参数压入新栈帧内
        /// </summary>
        private void PreLuaFuncCall(int argsNum)
        {
            //从主调栈帧的栈顶弹出函数与参数
            LuaDataUnion[] FuncAndParams = globalStack.PopN(argsNum + 1);

            Closure c = FuncAndParams[0].Closure;
           
            int paramsNum = c.Proto.NumParams;
            bool isVarArg = c.Proto.IsVararg != 0;

            //为被调函数创建栈帧

            //设置被调栈帧的栈底 需要保证在之前的栈帧之上 不会和之前的栈帧数据重叠
            //如果之前的栈帧里保存的数据没超过预留寄存器数量，就设为curFrame.ReserveRegisterMaxIndex + 1，否则设为Top + 1
            int bottom = Math.Max(curFrame.ReserveRegisterMaxIndex + 1, Top + 1);

            FuncCallFrame newFrame = new FuncCallFrame(c,bottom);

            //压入被调栈帧 并修改栈顶
            PushFuncCallFrameAndSetTop(newFrame);

            //将固定参数压入被调栈帧
            globalStack.PushN(FuncAndParams, 1, paramsNum);

            //再次修改栈顶 指向最后一个预留寄存器
            //这样后续push任意新值都不会占用到栈帧自己预留的寄存器位置
            SetTop(newFrame.ReserveRegisterMaxIndex);

            if (argsNum > paramsNum && isVarArg)
            {
                //实际参数多于固定参数 并且这个函数有变长参数
                //就把多出来的参数收集到变长参数里处理
                LuaDataUnion[] varArgs = new LuaDataUnion[argsNum - paramsNum];
                Array.Copy(FuncAndParams, paramsNum + 1, varArgs, 0, varArgs.Length);
                newFrame.VarArgs = varArgs;
            }

        }
```



## ExcuteLuaFuncCall

执行函数调用部分处理，就是通过一个while true循环不断取出指令执行，然后在碰到Return指令的时候结束循环就可以了

```csharp
		/// <summary>
        /// 执行Lua函数调用
        /// </summary>
        private void ExcuteLuaFuncCall()
        {
            while (true)
            {
                //不断取出指令执行 直到遇到return指令
                Instructoin i = new Instructoin(Fetch());
                i.Execute(this);
                if (i.OpType == OpCodeType.Return)
                {
                   //此时已经将返回值准备在栈顶上了
                    break;
                }
            }
        }
```



需要注意的是，在处理Return指令的时候，会将返回值都压入栈顶，并记录返回值数量（如果有变长返回值，那么实际返回值数量也是在此时被计算出并记录）

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_8.png)



## PostLuaFuncCall

函数调用结束后，所有返回值都被Return指令放到了栈顶，那么接下来就需要取出返回值，恢复栈顶到Buttom-1，弹出被调栈帧，闭合Upvalue

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_9.png)



最后将resultNum个返回值压入到主调栈帧上，不足的部分用nil补齐即可

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatLuaDevSummary/CatLuaDevSummary_10.png)

如此便完成了整个函数调用流程

PostLuaFuncCall完整代码如下：

```csharp
        /// <summary>
        /// Lua函数调用完成后，将被调函数栈帧的栈顶返回值压入主调函数栈帧的栈顶
        /// </summary>
        private void PostLuaFuncCall(int resultNum)
        {
            if (resultNum == 0)
            {
                //没返回值 直接弹出被调栈帧 并闭合upvalue以及恢复栈顶
                PopFuncCallFrameAndSetTop();
                return;
            }


            //取出所有返回值
            LuaDataUnion[] results = globalStack.PopN(CallFrameReturnResultNum);

            //弹出被调栈帧 并闭合upvalue以及恢复栈顶
            PopFuncCallFrameAndSetTop();

            //压入resultNum个返回值到主调栈帧上，不足的部分用nil补
            //resultNum为-1时就全部压入
            globalStack.PushN(results, 0, resultNum);
        }
```

