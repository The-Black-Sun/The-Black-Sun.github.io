---
title: Lua学习总结
date: 2023-09-06 20:54:23
tags: [Lua]
ategories: Lua
description: 一篇关于Lua知识的学习总结
copyright_author: 墨墨辰
copyright_author_href: https://the-black-sun.github.io/
copyright_url: https://the-black-sun.github.io/2023/09/06/Lua学习总结/
cover: https://the-black-sun-imgs.pages.dev/Img/Lua.jpg
---
## 前言

工作一直在写Lua相关的游戏逻辑脚本，但一直没做个总结，前端时间看了一下Lua语言的底层逻辑，做个记录总结让以后可以拿出来翻翻看。文章有点长，可以通过索引查找对应内容。

---

## Lua的底层与编译

### 语言类型与区别

Lua语言是以C语言为底层的`脚本语言`（动态语言，运行时编译），与C语言，C++等`静态语言`（静态语言，运行前编译）不同。其中Lua的执行主要运行流程如下：

- 程序员编辑Lua脚本代码，保存.lua文件
- 语法词法分析，并生成指令集（*lua.byte文件），属于编译过程
- Lua虚拟机执行指令集，属于执行过程
- 输出结果

这里放一张由大佬（**烟雨迷离半世殇**）做的对比表格：

| 语言 | 编辑 | 预编译 | 运行时编译 | 执行 |
| --- | --- | --- | --- | --- |
| Lua | 编写lua文件 | 无 | 虚拟机读取字节码并转换成虚拟机指令，汇编器编译成机器码 | CPU执行机器码 |
| C# | 编写cs文件 | 被C#编译器编译成dll，包含IL代码 | CLR使用JIT编译把IL转换成机器码 | CPU执行机器码 |

### Lua语言编译原理

Lua语言的的编译，首先需要明白代码块（chunk），闭包（closure），和原型（proto）的关系。

1. chunk：代码块，一段符合Lua语法的代码。
2. closure：Lua运行期间的一个实例对象，在运行期间调用的大多是一个closure。
3. proto：（原型）Lua语言中closure的原型（类似与C++中对象声明与实例的关系），定义了有关代码段的大部分信息，包括：
    - 指令列表：包含了函数编译后生成的`虚拟机指令`。
    - 常量表：这个函数运行期需要的`所有常量`，在指令中，常量使用常量表id进行索引。
    - 子proto表：所有内嵌于这个函数的`proto列表`，在OP_CLOSURE指令中的proto id就是索引的这个表。
    - 局部变量描述：这个函数使用到的`所有局部变量名称，以及生命期`。由于`所有的局部变量运行期都被转化成了寄存器id`，所以这些信息只是debug使用。
    - Upvalue描述：upvalue是内嵌函数中能够访问到的外包函数中的局部变量；称为外部局部变量感觉更为贴切。在创建closure时（创建函数实例）初始化Upvalue。

每一个proto在运行期间可以产生多个closure对象来表示函数实例。

![Untitled](../Lua学习总结/Untitled.png)

**`closure是运行期的对象`**，与运行期关系更大；而`与编译期相关的其实是proto对象`，他才是编译过程真正需要生成的目标对象。

### Lua语言编译流程

- 首先调用`lua_load` api，将一块符合lua语法的代码块`Chunk`进行编译。
- 编译为当前的Chunk生成一个`mainfunc proto`对象，并生成一个父对象`mainfunc closure`对象放到当前的栈顶中，等待接下来的执行。
- Chunk内部的每个`function statement（函数语句）`也都会生成一个对应的`子proto`，保存在外层函数的子函数列表中（proto中有子proto列表）。
- 所有最外层的**`function statement`**的proto会被保存到`mainfunc proto`的子函数列表中。
- 按照层次依次编译，形成一个以`mainfunc`为根节点的proto树。

![Untitled](../Lua学习总结/Untitled1.png)

---

## Lua的底层数据结构

只要讲到有关于Lua与其他语言交互的过程的相关原理，虚拟机一定是一个不可能绕过的概念。需要注意的是不同于像**`C#，java等语言使用的是基于堆栈的虚拟机`**，Lua5.0之后，`**Lua语言开始改用基于寄存器的虚拟机`。**首先了解一下，Lua底层数据结构的实现方式。

### C语言的实现面向对象（注意可能存在Lua版本差异）

由于Lua的底层是通过C语言进行实现的，所以在设置Lua的数据结构的时候，也是通过C语言来进行实现。而主要的实现思路是：

- 定义一个公共的数据结构作为基础类型，用来存储表达数据的基础信息，其他类型由此派生（**需要注意的是这个基础类型需要包含所有数据的存储方式，通过Union进行实现**）。
- 使用联合（ union ）来将所有数据包进来，
    - **Value**提供基础数据的存储方式（GC对象，指针，number与布朗值）以及**GCObject**。
    - **GCObject**提供所有非基础类型的存储方式（table，string，closure等），需要进行垃圾回收

```c
struct lua_TValue {
  Value value_; 
  int tt_;         //数据类型标识，新版11种（下截图），同时使用TValuefields进行表示
} TValue;

typedef union {
	GCObject *gc;    //上述所有需要进行数据回收的联合体，指针，指向联合体GCObject定义（table，thread，closure等）
	void *p;         //轻量级light，userdata，指针
	lua_CFunction f; //旧版没有这个。C语言函数
	lua_Integer i;   //整形类型，Lua5.1版本中只使用了lua_Number来进行整数和浮点数表示，但范围比int64_t类型小，之后Lua5.3扩充了整形类型来表示整数
	lua_Number n;    //默认为double类型，Lua重编译可更换
	int b;           //boolean值
}Value;

//内部包括table，string，usedata等定义
union GCObject{
	GCheader gch;
	union UTString ts;
	union Udata u;
	union Closure cl;
	struct Table h;
	struct Proto p;
	struct UpVal uv;
	struct lua_State th; /*thread*/
}

//以下不涉及数据结构表示，只是垃圾回收GC相关定义
//GCheader结构体
typedef struct GCheader{
	CommonHeader;
} GCheader;

//CommonHeader定义
#define CommonHeader GCObject *next;lu_byte tt;lu_byte marked
//next:下一个GC链表的成员
//tt：表示数据的类型，即上文图表2.1中的相关类型宏定义
//marked：GC相关标记位
```

![Untitled.png](../Lua学习总结/Untitled2.png)

综上所述，Lua中的数据结构是基于**TValue**来进行表示的。需要注意的是**Value**中存在的指针*p是用来存储数据结构lightusedate（自定义类型），而**GCObject**中的的对象是用来存储usedate（自定义类型）对象，分别对应上图（数据结构）中的类型2与类型7。这也表示了**`usedate会由Lua自动进行回收，但是lightusedate需要程序员自己进行管理`**。有上述代码可以得出结论：

- number、boolean、nil、light userdata 四种类型的值是直接存在栈上元素里的，和垃圾回收无关。
- string、table、closure、userdata、thread等存在栈上元素里的只是指针，数据在堆中他们都会在生命周期结束后被垃圾回收。
- function类型的存储，通过上述编译原理部分内容以及**GCObject**的Proto 与Closure进行实现

### 数据结构：表Table

Lua中的数据结构中，表Table是最重要的一种。在逻辑上是一个关联数组（哈希表，实际上是有一个哈希表与数组组成），`可以通过任何值（除了 nil）来索引表项，表项可以存储任何类型的值`。原因可以看一下存储在**GCObject**中的struct表table的代码结构：

```c
typedef struct Table { 
	CommonHeader; 
	lu_byte flags; //元方法存在标记，标记为1，表示元方法不存在
	lu_byte lsizenode; //散列桶数组的大小的 log2(size)
	struct Table netatable; //元表
	TValue *array; //指针，指向数组表，数组表中的数据，起始键为1
	Node *node; //散列桶起始指针
	Node *lastfree; //散列桶末尾指针
	GCObject *gclist; //GC相关的链表
	int sizearray; //数组表的长度
} Table;

//table中node所包含的数据与结构（如上图表所示）
typedef union Tkey{
	struct {
		TValuefield;
		struct Node *next; 
	}nk;
	Tvalue tvk;
}Tkey;

typedef struct Node{
	TValue i_val;
	Tkey i_key;
}Node;
```

![Untitled](../Lua学习总结/Untitled3.png)

有上图以及逻辑代码可以看出：

- Table的数据存储分为两个部分：所有键值在1与n（上限）之间的数据存储在数组表中，非整数键值或超过键值表示范围的通过散列表进行存储。（这就是Lua中迭代器`pair与ipair遍历的区别`原理）
- 数据存储通过TValue类型进行存储的。这表示表中可以存放所有的Lua数据结构。
- 表**`netatable`**与标记`flags`实现了元表元方法的相关功能。

除了以上可以由结构看出的内容之外，有关于表的存储内容相关的部分也需要注意：

- Table的存储的动态的，也就是数组表与散列表的大小是可以进行动态变化的（动态扩容）。
- 最初表的两个部分，都是空的，表的扩容需要Lua重新计算散列表与数组表的大小，找到满足一下条件的最大n值作为长度：**`1到 n 之间至少一半的空间会被利用`**（避免像稀疏数组一样浪费空间）；**`并且 n/2+1到 n 之间的空间至少有一个空间被利用`**（避免 n/2 个空间就能容纳所有数据时申请 n 个空间而造成浪费）
- Lua并非在空间上直接增加表大小（结构非链表，不能直接增加节点），而是申请新的空间，并将元数据存放到新空间中。
- 表的两个存储部分：数组表与散列表是分开扩容的。这种混合型结构让表在作为数组使用时，有数组的优点（存储紧凑，性能高，键值隐含，不用在意哈希表的空间与计算开销）。作为散列表使用时，数组部分又常常不存在，从而节省内存空间。

PS：除了以上表述内容之外，Table数据结构的散列表部分还使用了双重散列技术，又叫双重哈希法，有兴趣可以查找一下相关资料以及Brent论文提到的HashTable增查新方法 （地址在最后参考文献中）。

### 数据结构：字符串String

字符串String类型与本身是结构体的table类似。它本身是union联合提结构，底层代码如下：

```c
//长短字符串定义
#define LUA_TSHRSTR (LUA_TSTRING | (0 << 4))  /* short strings */
#define LUA_TLNGSTR (LUA_TSTRING | (1 << 4))  /* long strings */

//5.2版本，区分长短字符串
typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
                  /* 对于短字符串：这个标示是否是保留字，长字符串：是否已经哈希① */
  lu_byte shrlen;  /* 短字符串的长度 */
  unsigned int hash;//hash值
  union {
    size_t lnglen;  /* 长字符串的长度 */
    struct TString *hnext;  //哈希表的链表
  } u;
} TString;

typedef union UTString {
  L_Umaxalign dummy;  /* 内存对齐 */
  TString tsv;
} UTString;

typedef struct stringtable { 
	GCObject **hash; 
	lu_int32 nuse; /* number of elements */ 
	int size; 
} stringtable;
```

![Untitled](../Lua学习总结/Untitled4.png)

在Lua5.2之后，字符串存储就被分成了两种：长字符串与短字符串。

- 短字符串存储在`全局stringtable`当中，相同字符串只会有一份实际数据拷贝，每份相同的TString对象只是存放一个hash值，用来索引stringtable。
- 长字符串直接存储在union的`hnext`当中，相同字符串在内存都是单独一份数据拷贝。
- 为了避免链表中数据过多导致的可能的一次线性查找过程，除了长字符串单独存储外，还有luaS_resize(lua_State *L,int newsize)专门进行重新的散列排列，通常在以下两种情况调用：
    - lgc.c的checkSizes函数：散列桶数量太大（是实际存放的字符串4倍），减少为原本一倍
    - lstring.c的newlstr函数：字符串数量太大（字符串数量大于桶数量，并且桶数组的数量小于MAX_INT/2）,对桶进行翻倍扩容

### 数据结构：thread（叫线程，是协程）

Lua 5.0 版开始， `Lua 实现不对称协程`（也称为半不对称协程或不完全协程） 。

Lua将所有关于协同程序的函数放置在一个名为“coroutine”的table中。

1. coroutine.create创建一个thread类型的值表示新的协同程序，返回一个协同程序。
2. coroutine.status检查协同程序的状态（挂起suspended、运行running、死亡dead、正常normal）。
3. coroutine.resume启动或再次启动一个协同程序，并将其状态由挂起改为运行。
4. coroutine.yield让一个协同程序挂起。
5. coroutine.wrap同样创建一个新的协同程序，返回一个函数。

Lua 中协程是有栈的，这样我们就可以在多级函数嵌套调用内挂起（暂停执行）一个协程。解释器只是简单地将整个栈放在一边而在另一个栈上继续执行。 一个程序可以任意重启任何挂起的协程。当与栈相关的协程不可用时，垃圾回收器就回收栈空间。

---

## Lua虚拟机的跨语言调用

Lua提供了一个虚拟栈，这个虚拟栈可以完成Lua语言与其他语言之间的数据交换。Lua API本身提供了一系列接口可以让我们操作这个虚拟栈。

以下是C语言对虚拟栈的操作API：

```c
/*
** push functions (C -> stack)
*/
//数据从C语言到虚拟栈中
LUA_API void  (lua_pushnil) (lua_State *L);
LUA_API void  (lua_pushnumber) (lua_State *L, lua_Number n);
LUA_API void  (lua_pushinteger) (lua_State *L, lua_Integer n);
LUA_API void  (lua_pushlstring) (lua_State *L, const char *s, size_t l);
LUA_API void  (lua_pushstring) (lua_State *L, const char *s);
LUA_API const char *(lua_pushvfstring) (lua_State *L, const char *fmt,
                                                      va_list argp);
LUA_API const char *(lua_pushfstring) (lua_State *L, const char *fmt, ...);
LUA_API void  (lua_pushcclosure) (lua_State *L, lua_CFunction fn, int n);
LUA_API void  (lua_pushboolean) (lua_State *L, int b);
LUA_API void  (lua_pushlightuserdata) (lua_State *L, void *p);
LUA_API int   (lua_pushthread) (lua_State *L);

//数据由虚拟栈到C语言中
/*
** access functions (stack -> C)
*/
LUA_API int             (lua_isnumber) (lua_State *L, int idx);
LUA_API int             (lua_isstring) (lua_State *L, int idx);
LUA_API int             (lua_iscfunction) (lua_State *L, int idx);
LUA_API int             (lua_isuserdata) (lua_State *L, int idx);
LUA_API int             (lua_type) (lua_State *L, int idx);
LUA_API const char     *(lua_typename) (lua_State *L, int tp);

LUA_API int            (lua_equal) (lua_State *L, int idx1, int idx2);
LUA_API int            (lua_rawequal) (lua_State *L, int idx1, int idx2);
LUA_API int            (lua_lessthan) (lua_State *L, int idx1, int idx2);

// 经实测，to* 函数不会出栈
LUA_API lua_Number      (lua_tonumber) (lua_State *L, int idx);
LUA_API lua_Integer     (lua_tointeger) (lua_State *L, int idx);
LUA_API int             (lua_toboolean) (lua_State *L, int idx);
LUA_API const char     *(lua_tolstring) (lua_State *L, int idx, size_t *len);
LUA_API size_t          (lua_objlen) (lua_State *L, int idx);
LUA_API lua_CFunction   (lua_tocfunction) (lua_State *L, int idx);
LUA_API void           *(lua_touserdata) (lua_State *L, int idx);
LUA_API lua_State      *(lua_tothread) (lua_State *L, int idx);
LUA_API const void     *(lua_topointer) (lua_State *L, int idx);
```

以下是Lua对虚拟栈的操作API：

```c
/*
** get functions (Lua -> stack)
*/
//Lua数据入栈
LUA_API void  (lua_gettable) (lua_State *L, int idx);
LUA_API void  (lua_getfield) (lua_State *L, int idx, const char *k);
LUA_API void  (lua_rawget) (lua_State *L, int idx);
LUA_API void  (lua_rawgeti) (lua_State *L, int idx, int n);
LUA_API void  (lua_createtable) (lua_State *L, int narr, int nrec);
LUA_API void *(lua_newuserdata) (lua_State *L, size_t sz);
LUA_API int   (lua_getmetatable) (lua_State *L, int objindex);
LUA_API void  (lua_getfenv) (lua_State *L, int idx);

/*
** set functions (stack -> Lua)
*/
//Lua获取栈中数据或修改栈中数据
LUA_API void  (lua_settable) (lua_State *L, int idx);
LUA_API void  (lua_setfield) (lua_State *L, int idx, const char *k);
LUA_API void  (lua_rawset) (lua_State *L, int idx);
LUA_API void  (lua_rawseti) (lua_State *L, int idx, int n);
LUA_API int   (lua_setmetatable) (lua_State *L, int obj
```

除此之外，还有一系列用来控制堆栈的相关函数（栈操作）：

```c
/*
** basic stack manipulation
*/
LUA_API int   (lua_gettop) (lua_State *L);
LUA_API void  (lua_settop) (lua_State *L, int idx);
LUA_API void  (lua_pushvalue) (lua_State *L, int idx);
LUA_API void  (lua_remove) (lua_State *L, int idx);
LUA_API void  (lua_insert) (lua_State *L, int idx);
LUA_API void  (lua_replace) (lua_State *L, int idx);
LUA_API int   (lua_checkstack) (lua_State *L, int sz);
```

### **C/C++调用Lua**

**C/C++ 获取 Lua 值**

1. 使用 lua_getglobal 来获取值并将其压栈。
2. 使用 lua_toXXX 将栈中元素取出（此时元素并不会出栈）转成相应的 C/C++ 类型的值。

**C/C++ 调用 Lua 函数**

1. 使用 lua_getglobal 来获取函数并将其压栈。
2. 如果这个函数有参数的话，就需要依次将函数的参数也压入栈。
3. 调用 lua_pcall 开始调用函数，调用完成以后，会将返回值压入栈中。
4. 取返回值，调用完毕。

### **Lua 调用 C/C++**

Lua 可以调用 C/C++ 的函数，步骤为：

1. `将 C 的函数包装成 Lua 环境认可的函数`：将被调用的 C/C++ 函数从普通的 C/C++ 函数包装成 Lua_CFunction 格式，并需要在函数中将返回值压入栈中，并返回返回值个数;
2. `将包装好的函数注册到 Lua 环境中`：使用宏`lua_register` 调用`lua_pushfunction(L,f)` 和`lua_setglobal(L,n)`，将函数存放在一个全局 table 中。
3. 像使用普通 Lua 函数那样使用注册函数。

PS：注意XLua或ToLua都是对Lua虚拟机做了上层封装，方便进行相关接口调用。

---

## Lua的闭包（主要通过upValue实现）

### Lua的闭包定义与使用

闭包：***能够读取其他函数内部变量的函数***。

常见形式：通过调用含有一个内部函数加上该外部函数持有的外部局部变量（upvalue）的外部函数产生的一个函数实例。（外部函数+外部局部变量+内部函数（闭包函数））。

```lua
function test()
        local i=0
        return function()//尾调用
            i+=1
            return i
        end
    end
    c1=test()
    c2=test()--c1,c2是建立在同一个函数，同一个局部变量的不同实例上面的两个不同的闭包
             --闭包中的upvalue各自独立，调用一次test（）就会产生一个新的闭包

    print(c1()) -->1
    print(c1()) -->2//重复调用时每一个调用都会记住上一次调用后的值，就是说i=1了已经
    print(c2())    -->1//闭包不同所以upvalue不同
    print(c2()) -->2
```

由上文输出结果可知：闭包中的外部局部变量在每个函数实例中各自独立，两个实例的调用结果不会相互干扰。同时实例中的外部局部变量会被保存。

PS：由于迭代器需要保存上一次调用的状态与下一次成功调用的状态，可以正好用闭包的机制实现。（迭代器只是一个生成器，本身不带有循环）以下是迭代器的实现。

```lua
　 function list_iter(t)
            local i=0
            local n=table.getn(t)
            return function()
                i=i+1
                if i<=n then return t[i] end
            end
    end
--使用
t={10,20,90}            
iter=list_iter(t)--调用迭代器产生一个闭包
	while true do
		local element=iter()
    if element==nil then break end
       print(element)
    end
end

--泛型for使用：
t={10,0,29}            
for element in list_iter(t) do                
		print(element)
end
--这里的list_iter()工厂函数只会被调用一次产生一个闭包函数，后面的每一次迭代都是用该闭包函数，而不是工厂函数
```

### Lua闭包的底层实现

在Lua语言运行期间，任何时候只要 Lua执行一个 function...end 表达式， 它就会创建一个新的闭包。同时每个闭包会包含以下内容：

- 一个对`函数原型proto`的引用
- 一个对`环境`的引用（环境其实是一个表，函数可在该表中索引全局变量）
- 一个数组，数组中每个元素都是一个对 `upvalue` 的引用，可通过该数组来存取外层的局部变量 （upvalue是proto中的一个信息，见上文）

闭包最主要的特点是能够获取其他函数内部变量。Lua是通过`upvalue 的结构`来实现该功能的，对任何外层局部变量的存取都能间接地通过 upvalue 来进行。upvalue 最初指向栈中变量活跃的地方（图 4 左边） 。当离开变量作用域时（超过变量生存期时） ，变量被复制到 upvalue 中（图 4 右边） 。由于对变量的存取是通过 upvalue 里的指针间接进行的，因此复制动作对任何存取此变量的代码来说都是没有影响的。

![Untitled](../Lua学习总结/Untitled5.png)

与内层函数不同的是， 声明该局部变量的函数直接在堆栈中存取它的局部变量。`通过为每个变量至少创建一个 upvalue 并按所需情况进行重复利用`，保证了未决状态（是否超过生存期）的局部变量（pending vars）能够在闭包间正确地共享。`为了保证这种唯一性， Lua 为整个运行栈保存了一个链接着所有正打开着的 upvalue（那些当前正指向栈内局部变量的 upvalue）的链表`（图 4 中未决状态的局部变量的链表） 。当 Lua 创建一个新的闭包时，它开始遍历所有的外层局部变量，对于其中的每一个，若在上述 upvalue 链表中找到它，就重用此 upvalue，否则， Lua 将创建一个新的 upvalue 并加入链表中。注意，`一般情况下这种遍历过程在探查了少数几个节点后就结束了`， 因为对于每个被内层函数用到的外层局部变量来说，该链表至少包含一个与其对应的入口（upvalue） 。一旦某个关闭的upvalue 不再被任何闭包所引用，那么它的存储空间就立刻被回收。

一个函数有可能存取其更外层函数而非直接外层函数的局部变量。 这种情况下，有可能当闭包创建时，此局部变量尚不存在。 `Lua 使用 flat 闭包`来处理这种情况。`有了 flat 闭包，无论何时只要函数存取更外层的局部变量，该变量也会进入其直接外层函数的闭包中`。这样，当一个函数被实例化时，所有进入其闭包的变量就在直接外层函数的栈或闭包中了。

---

### Lua的元表与元方法（面向对象）

在Lua table中我们可以访问对应的key来得到value值，但是却无法对两个table进行操作。因此`Lua 提供了元表(Metatable)，允许我们改变table的行为，每个行为关联了对应的元方法`。通俗来说，元表就像是一个“操作指南”，里面包含了一系列操作的解决方案，例如__index方法就是定义了这个表在索引失败的情况下该怎么办，__add方法就是告诉table在相加的时候应该怎么做。这里面的__index，__add就是元方法。（上文table表的结构中可以看见元表与元方法标记）

### 面向对象的实现

```lua
--基类Account
Account = {balance = 0}

function Account:new (o)
	o = o or {}
	setmetatable(o, self)//设置元表为自身
	self.__index = self//设置__index元方法为自身
	return o
end

function Account:deposit (v)
	self.balance = self.balance + v
end

function Account:withdraw (v)
	if v > self.balance then error"insufficient funds" end
	self.balance = self.balance - v
end

--子类SpecialAccount
--此时代表SpecialAccount是Account的一个实例，即继承了Account所有内容
SpecialAccount = Account:new()

--SpecialAccount从Account继承了new方法
--new执行时，self指向SpecialAccount
--s的metable，__index是SpecialAccount
s = SpecialAccount:new{limit = 1000.00}//s继承了SpecialAccount

--在s中找不到deposit域，会到SpecialAccount找，然后到Account中找
s:deposit(100.00)
--------------------------------------------------------------------
--根据上面的描述，我们就可以在SpecialAccount中重写Account方法
function SpecialAccount:withdraw (v)
	if v - self.balance >= self:getLimit() then
		error"insufficient funds"
	end
	self.balance = self.balance - v
end

function SpecialAccount:getLimit ()
	return self.limit or 0
end

--调用方法 s:withdraw(200.00)，Lua 不会到 Account 中查找
--因为它第一次就在 SpecialAccount 中发现了新的 withdraw 方法
--由于 s.limit 等于 1000.00（记住我们创建 s 的时候初始化了这个值）
--程序执行了取款操作，s 的 balance 变成了负值
s:withdraw(200.00)
```

---

## Lua语言的GC算法

### 两种常见的垃圾回收算法

- **自动引用计数(Automatic Reference Counting)算法**（****ARC算法****）
    - 对每一个对象保存一个整形的引用计数属性，用来记录对象被引用的情况，当记录对象被引用数维0时，将会被GC进行回收。
    - 实现简单，判定效率高，回收没有延迟
    - 单独的存储计数器字段，增加了存储空间的开销；每次赋值需要更新存储计数器增加了时间开销；**无法处理循环引用问题**（致命）
    - 解决方法（Java）：可达性分析（不可达对象不等于无引用对象，最少要分析标记两次）
    
    ![Untitled](../Lua学习总结/Untitled6.png)
    
- ****标记-清除（Mark - Sweep）算法****
    - 当堆中的有效内存空间被耗尽时，停止整个程序，进行标记工作与清除工作
        - 标记：****Collector****从根节点开始遍历，标记所有被引用对象。（一般在对象的Header中记录为可达对象）
        - 清除：****Collector****对堆内存进行从头到尾的线性遍历，发现没有标记为可达对象的对象将其回收
    - 效率低，GC时需要停止整个用户进程，用户体验差，清理出的内存不连续没需要维护一个空闲链表

### Lua的GC原理与算法设计

- Lua语言的GC算法采用`标记-清除（Mark - Sweep）算法`
- 在Lua5.1之前，Lua的GC过程是一次性不可打断的过程，采用的Mark算法是双色标记算法，黑色不被回收，白色被回收。但是如果在GC过程的回收阶段中，加入新的对象，不论标记成什么颜色都不对，所以在后来被改进
- Lua5.1之后采用了分布回收（增量GC的实现）以及`三色增量标记清除算法`
    
    更新之后的节点主要分为三种：
    
    - **黑色节点**：已经完全扫描过的节点。
    - **灰色节点**：在扫描黑色节点时候初步扫描到，但是还未完全扫描的obj，这类obj会被放到一个待处理列表中进行逐个完全扫描。
    - **白色节点**：还未被任何黑色节点所引用的节点(因为一旦被黑色节点引用将被置为黑色或灰色)。这里白色又被进一步细分为**cur white**和**old white**，lua会记录当前的**cur white**颜色，每个节点新创建的时候都是**cur white**，lua会在mark阶段结束的时候翻转这个**cur white**的位，从而使得这之前创建的白色节点都是**old**的，在sweep阶段能够得到正确释放。

GC的主要流程如下：

```c
为每一个新创建的节点设置为cur white（当前白色）
//初始化阶段
遍历root根节点的引用对象，将white（所有白色）设置成灰色，放入灰色节点链表中
//标记阶段（存在引用屏障，保证黑色节点不指向白色节点，灰色是屏障）
循环扫描灰色链表中的元素，将其置为黑色，然后扫描与该元素关联的其他元素，白色置灰加入灰色节点链表
结束回收阶段，将所有cur white的节点设置成old white的节点（保证这些节点都是要被回收的，回收阶段新加入的白色节点不回收）
//回收阶段
（可能存在新的白色节点加入，设置为cur white）
遍历所有对象，回收所有old white节点
```

---

## Lua语言的基础写法

指路：[Lua 教程 | 菜鸟教程 (runoob.com)](https://www.runoob.com/lua/lua-tutorial.html)

---

## 参考文献与学习资料

- [探索Lua5.2内部实现:编译系统(1) 概述_yuanlin2008的博客-CSDN博客](https://blog.csdn.net/yuanlin2008/article/details/8486463)
- [Lua5.0原理探究 | 登峰造极者，殊途亦同归。 (lfzxb.top)](https://www.lfzxb.top/the-theory-of-lua-5-0/)
- [The Implementation of Lua5.0.pdf (codingnow.com)](https://www.codingnow.com/2000/download/The%20Implementation%20of%20Lua5.0.pdf)
- [Lua 跨语言调用 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/454695276?utm_id=0)
- Brent论文提到的HashTable增查新方法 地址：[https://maths-people.anu.edu.au/~brent/pd/rpb013.pdf](https://maths-people.anu.edu.au/~brent/pd/rpb013.pdf)
- [Lua相关 - 随笔分类 - 天山鸟 - 博客园 (cnblogs.com)](https://www.cnblogs.com/Jaysonhome/category/1557006.html)
- [https://link.zhihu.com/?target=https%3A//blog.csdn.net/v_xchen_v/article/details/77249332](https://link.zhihu.com/?target=https%3A//blog.csdn.net/v_xchen_v/article/details/77249332)
- 《Lua的设计与实现》
- 《Lua5.3 王桂林》