# Lua 5.3.5 源码阅读

#### 一、目录结构

- ##### includes

| 文件      | 作用                    |
| --------- | ----------------------- |
| lua.h     | lua的通用定义           |
| lauxlib.h | 绑定lua内嵌库的辅助函数 |
| lualib.h  | lua标准库               |
| luaconf.h | lua配置文件             |

- ##### 核心(core)

  - ###### 数据类型：

  | 文件                | 作用                                 |
  | ------------------- | ------------------------------------ |
  | lobject.h/lobject.c | lua对象定义，和操作lua对象的通用函数 |
  | lstring.h/lstring.c | 字符串表(保存所有字符串)             |
  | lfunc.h/lfunc.c     | 原型(prototypes)和闭包的操作函数     |
  | ltable.h/ltable.c   | Lua表                                |
  | lstate.h/lstate.c   | 全局状态                             |

  - ###### 虚拟机：

  | 文件                  | 作用                     |
  | --------------------- | ------------------------ |
  | ldo.h/ldo.c           | Lua栈和调用的结构        |
  | lopcodes.h/lopcodes.c | 虚拟机操作码             |
  | ltm.h/ltm.c           | 原方法，操作元方法的函数 |
  | lvm.h/lvm.c           | Lua虚拟机                |

  - ###### 语法解析和代码生成：

  | 文件                | 作用                   |
  | ------------------- | ---------------------- |
  | lctype.h/lctype.c   | Lua对ctype.h的优化函数 |
  | ldump.c             | 保存预编译的Lua代码块  |
  | lundump.h/lundump.c | 加载预编译的Lua代码块  |
  | lcode.h/lcode.c     | 代码生成器             |
  | llex.h/llex.c       | 词法生成器             |
  | lparser.h/lparser.c | Lua语法解析器          |

  - ###### C API：

  | 文件          | 作用                                      |
  | ------------- | ----------------------------------------- |
  | lapi.h/lapi.c | Lua API，实现了大部分Lua C API(lua_*函数) |

  - ###### 通用函数：

  | 文件              | 作用                               |
  | ----------------- | ---------------------------------- |
  | lprefix.h         | 必须放在其他头文件前的Lua定义      |
  | llimits.h         | 限制，基础类型和一些安装依赖的定义 |
  | lmem.h/lmem.c     | 内存管理                           |
  | lgc.h/lgc.c       | 垃圾回收                           |
  | lzio.h/lzio.c     | 通用的输入流接口                   |
  | ldebug.h/ldebug.c | Debug接口                          |

- ##### 内嵌库(libraries)

| 文件       | 作用                    |
| ---------- | ----------------------- |
| lauxlib.c  | 绑定lua内嵌库的辅助函数 |
| lbaselib.c | 基础库                  |
| lbitlib.c  | 位运算库                |
| lcorolib.c | 协程库                  |
| ldblib.c   | debug库                 |
| liolib.c   | IO库                    |
| lmathlib.c | 数学库                  |
| loadlib.c  | 动态库加载器，包、模块  |
| loslib.c   | OS库                    |
| lstrlib.c  | 字符串库                |
| ltablib.c  | 表操作库                |
| lutf8lib.c | UTF8库                  |
| linit.c    | 加载所有库              |

- ##### 解释器(interpreter)

| 文件  | 作用      |
| ----- | --------- |
| lua.c | lua解释器 |

- ##### 编译器(compiler)

| 文件   | 作用      |
| ------ | --------- |
| luac.c | lua编译器 |

#### 二、基本类型

Lua有八种基本的数据类型：nil、boolean、number、string、userdata、function、thread和table。在Lua中所有的基本数据类型都是第一类值（即可以存入全局变量、局部变量或table中，或作为实际参数传递给函数，或从函数中返回值）。

Lua的类型定义可以在lua.h中找到：

```c
// lua.h

#define LUA_TNIL		0
#define LUA_TBOOLEAN		1
#define LUA_TLIGHTUSERDATA	2
#define LUA_TNUMBER		3
#define LUA_TSTRING		4
#define LUA_TTABLE		5
#define LUA_TFUNCTION		6
#define LUA_TUSERDATA		7
#define LUA_TTHREAD		8
```

其中lightuserdata和userdata都代表userdata类型，其中区别在于userdata的分配是由Lua内部来完成的，而lightuserdata的分配是由使用者来完成的，即后者需要使用者来关注它的生命周期，而前者不用。

关于Lua基本类型的定义和操作我们可以在lobject.h/lobject.c中找到：

```c
// lobject.h

/*
** Common type for all collectable objects
*/
typedef struct GCObject GCObject;

/*
** Common Header for all collectable objects (in macro form, to be
** included in other objects)
*/
#define CommonHeader	GCObject *next; lu_byte tt; lu_byte marked

/*
** Common type has only the common header
*/
struct GCObject {
  CommonHeader;
};

/*
** Tagged Values. This is the basic representation of values in Lua,
** an actual value plus a tag with its type.
*/

/*
** Union of all Lua values
*/
typedef union Value {
  GCObject *gc;    /* collectable objects */
  void *p;         /* light userdata */
  int b;           /* booleans */
  lua_CFunction f; /* light C functions */
  lua_Integer i;   /* integer numbers */
  lua_Number n;    /* float numbers */
} Value;

#define TValuefields	Value value_; int tt_

typedef struct lua_TValue {
  TValuefields;
} TValue;
```

从代码中可以看到，Lua采用一个TValue的结构体来代表Lua的基本数据类型，在TValue中主要包含一个表明类型的tt_和一个实际数据的联合体Value。

联合体Value中gc代表了可回收的对象（包括string、userdata、function、thread和table），p代表了lightuserdata，b代表了boolean，f代表了light C functions（functions的一个变种类型），i代表整型，n代表浮点数类型。其中我们可以看到GCObject类型包括了一个CommonHeader，后面的所有GCObject类型都会包含一个CommonHeader（方便做转换），其中有一个GCObject的指针next（所有GC对象构成的链表，用于GC中），一个代表类型的tt，一个用于GC的marked。

```c
// lobject.h

/*
** tags for Tagged Values have the following use of bits:
** bits 0-3: actual tag (a LUA_T* value)
** bits 4-5: variant bits
** bit 6: whether value is collectable
*/

/*
** LUA_TFUNCTION variants:
** 0 - Lua function
** 1 - light C function
** 2 - regular C function (closure)
*/

/* Variant tags for functions */
#define LUA_TLCL	(LUA_TFUNCTION | (0 << 4))  /* Lua闭包 - Lua closure */
#define LUA_TLCF	(LUA_TFUNCTION | (1 << 4))  /* 纯C函数 - light C function */
#define LUA_TCCL	(LUA_TFUNCTION | (2 << 4))  /* C闭包 - C closure */

/* Variant tags for strings */
#define LUA_TSHRSTR	(LUA_TSTRING | (0 << 4))  /* 短字符串 - short strings */
#define LUA_TLNGSTR	(LUA_TSTRING | (1 << 4))  /* 长字符串 - long strings */

/* Variant tags for numbers */
#define LUA_TNUMFLT	(LUA_TNUMBER | (0 << 4))  /* 浮点数 - float numbers */
#define LUA_TNUMINT	(LUA_TNUMBER | (1 << 4))  /* 整型 - integer numbers */

/* Bit mark for collectable types */
#define BIT_ISCOLLECTABLE	(1 << 6)

/* mark a tag as collectable */
#define ctb(t)			((t) | BIT_ISCOLLECTABLE)
```

其中tt代表的类型，0-3位代表了真实的类型（LUA_TNIL等），4-5位代表了变种类型（其中function、string、number具有变种类型），第6位则表示是否能够被回收。

在Value中除了作为gcobject时需要转换之后再使用，其他类型都直接使用即可。

下面我们分别讲下各个类型的实现。

##### string：

```c
// lobject.h

/*
** Header for string value; string bytes follow the end of this structure
** (aligned according to 'UTString'; see next).
*/
typedef struct TString {
  CommonHeader;
  lu_byte extra;  /* reserved words for short strings; "has hash" for longs */
  lu_byte shrlen;  /* length for short strings */
  unsigned int hash;
  union {
    size_t lnglen;  /* length for long strings */
    struct TString *hnext;  /* linked list for hash table */
  } u;
} TString;

/*
** Ensures that address after this type is always fully aligned.
*/
typedef union UTString {
  L_Umaxalign dummy;  /* ensures maximum alignment for strings */
  TString tsv;
} UTString;
```

string类型用TString来表示，而UTString是用来进行内存对齐。

TString类型首先是一个CommonHeader方便GCObject做转换，其中extra代表长字符串（长度大于LUAI_MAXSHORTLEN 40）是否具有哈希值（若存在则不重复计算，在短字符串中无效），shrlen代表短字符串长度，hash代表哈希值，其中u联合结构如果为长字符串时则代表长字符串的长度，短字符串时则代表一个TString的指针（用于短字符串的哈希表）。

此处TString只是代表了string类型的结构数据，string真正的数据会在跟在TString结构后面。

在Lua中所有的字符串会被缓存起来，global_State中有两个值用于缓存字符串，stringtable类型的strt和 TString *\[\]\[\]类型的strcache，stringtable是将所有出现过的短字符串（长度小于等于40）用哈希数组的方式全部村起来，如果哈希冲突的情况则用链表串起来。而strcache是将出现的字符串进行有限的缓存，免去重复创建或者搜索的开销。

##### userdata：

```c
// lobject.h

/*
** Header for userdata; memory area follows the end of this structure
** (aligned according to 'UUdata'; see next).
*/
typedef struct Udata {
  CommonHeader;
  lu_byte ttuv_;  /* user value's tag */
  struct Table *metatable;
  size_t len;  /* number of bytes */
  union Value user_;  /* user value */
} Udata;

/*
** Ensures that address after this type is always fully aligned.
*/
typedef union UUdata {
  L_Umaxalign dummy;  /* ensures maximum alignment for 'local' udata */
  Udata uv;
} UUdata;
```

userdata用Udata来表示，而UUdata是用来进行内存对齐。

Udata类型首先是一个CommonHeader方便GCObject做转换，接着是一个ttuv_用以表示这一块userdata代表什么类型，接着是一个metatable代表userdata的元表，len代表此块内存的长度，user\_代表真正的userdata。

##### table：

```c
// lobject.h

/*
** Tables
*/

typedef union TKey {
  struct {
    TValuefields;
    int next;  /* for chaining (offset for next node) */
  } nk;
  TValue tvk;
} TKey;

typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;

typedef struct Table {
  CommonHeader;
  lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
  lu_byte lsizenode;  /* log2 of size of 'node' array */
  unsigned int sizearray;  /* size of 'array' array */
  TValue *array;  /* array part */
  Node *node;
  Node *lastfree;  /* any free position is before this position */
  struct Table *metatable;
  GCObject *gclist;
} Table;
```

table的实现主要分为两个部分，一个部分是数组部分，另外一个部分是哈希表部分，数组部分不储存key，数组部分将试图保存那些键值介于1到某个上限n之间的值，非整数部分和超过数组范围n的整数键对应的值将被存入到散列表中。当表需要增长时，Lua会重新计算散列表部分和数组部分的大小。最初表的两个部分可能都是空的。新的数组部分的大小时满足以下条件的最大的n值：1到n之间至少一半的空间会被利用；且n/2+1到n之间的空间至少有一个空间被利用。当新的大小计算出来后，Lua为数组重新申请空间，并将原来的数据存入新的空间。哈希部分的大小是能容纳所有数据的最小2次方。

这种混合结构有两个有优点。第一，存取整数的值很快，因为无需计算散列值。第二，相比将数据存入散列部分，数组大概只需要相对散列一半的空间，因为在数组部分，键是隐含的。

在看Table的结构之前，我们先看下哈希用到的Node结构部分，Node具有一个TValue类型的i_val来表示Node中的值，一个TKey类型的i_key来表示Node中的键值，由于在table中任意的第一类值都可以作为键值，所以TKey采用了一个联合体包含了一个TValue和指向下个键值的next。

此处指向下个键值的next和Lua中实现table的方式有关，在Lua中table的实现是采用哈希表的方式，哈希的部分实际上是一个Node组成的数组，将键值进行哈希之后放入对应的数组位置，当发生冲突的时候，先计算出冲突点是否在该哈希的主位置上，如果在主位置上，则将新插入的点放到一个空的节点中，且让next指向新的节点，若冲突点不在该哈希的主位置上，则将冲突点移到一个空节点上，并将新的键值放到主位置上。

这种做法的好处是：第一，不用开链的方式不用创建新的节点，节省了创建的开销。第二，相比进行二次哈希，直接采用next的偏移值，来获取该哈希值上的键值效率更高。

回到Table的结构，首先还是一个CommonHeader，flags用以表示这个表提供了哪些meta method，lsizenode表示哈希数组的log2值，sizearray表示数组的大小，array数组部分，node哈希数组部分，lastfree表示空节点的最后一个地址，metatable元表，gclist用于gc的相关链表。

##### function：

todo

##### thread：



```c
// lstate.h

/*
** 'global state', shared by all threads of this state
*/
typedef struct global_State {
  lua_Alloc frealloc;  /* Lua的全局内存分配器，用户可以替换成自己的 - function to reallocate memory */
  void *ud;         /* frealloc的userdata - auxiliary data to 'frealloc' */
  l_mem totalbytes;  /* 已分配的空间字节数减去GCdebt - number of bytes currently allocated - GCdebt */
  l_mem GCdebt;  /* 需要回收的内存数量 - bytes allocated not yet compensated by the collector */
  lu_mem GCmemtrav;  /* memory traversed by the GC */
  lu_mem GCestimate;  /* 内存实际使用的估计值 - an estimate of the non-garbage memory in use */
  stringtable strt;  /* 字符串的哈希表，存储所有短字符串 - hash table for strings */
  TValue l_registry;
  unsigned int seed;  /* randomized seed for hashes */
  lu_byte currentwhite;
  lu_byte gcstate;  /* gc的状态 - state of garbage collector */
  lu_byte gckind;  /* gc的类型 - kind of GC running */
  lu_byte gcrunning;  /* true if GC is running */
  GCObject *allgc;  /* 所有会被gc的对象链表 - list of all collectable objects */
  GCObject **sweepgc;  /* current position of sweep in list */
  GCObject *finobj;  /* list of collectable objects with finalizers */
  GCObject *gray;  /* list of gray objects */
  GCObject *grayagain;  /* list of objects to be traversed atomically */
  GCObject *weak;  /* list of tables with weak values */
  GCObject *ephemeron;  /* list of ephemeron tables (weak keys) */
  GCObject *allweak;  /* list of all-weak tables */
  GCObject *tobefnz;  /* list of userdata to be GC */
  GCObject *fixedgc;  /* 不会被gc的对象链表 - list of objects not to be collected */
  struct lua_State *twups;  /* 具有开放上值的其他线程 - list of threads with open upvalues */
  unsigned int gcfinnum;  /* number of finalizers to call in each GC step */
  int gcpause;  /* size of pause between successive GCs */
  int gcstepmul;  /* GC 'granularity' */
  lua_CFunction panic;  /* to be called in unprotected errors */
  struct lua_State *mainthread; /* 主线程 */
  const lua_Number *version;  /* Lua版本号，用以标记state是否创建完成 - pointer to version number */
  TString *memerrmsg;  /* 内存报错字符串 - memory-error message */
  TString *tmname[TM_N];  /* 所有tag-method的名字 - array with tag-method names */
  struct Table *mt[LUA_NUMTAGS];  /* 所有基本类型的metatable - metatables for basic types */
  TString *strcache[STRCACHE_N][STRCACHE_M];  /* 字符串缓存，存储长字符串 cache for strings in API */
} global_State;
```

#### 三、Lua主流程

```c
// lua.c

int main (int argc, char **argv) {
  int status, result;
  lua_State *L = luaL_newstate();  /* create state */
  if (L == NULL) {
    l_message(argv[0], "cannot create state: not enough memory");
    return EXIT_FAILURE;
  }
  lua_pushcfunction(L, &pmain);  /* to call 'pmain' in protected mode */
  lua_pushinteger(L, argc);  /* 1st argument */
  lua_pushlightuserdata(L, argv); /* 2nd argument */
  status = lua_pcall(L, 2, 1, 0);  /* do the call */
  result = lua_toboolean(L, -1);  /* get result */
  report(L, status);
  lua_close(L);
  return (result && status == LUA_OK) ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

从lua.c(lua解释器)的main来看lua的执行流程。

1.lua通过luaL_newstate()创建了一个新的Lua状态机。

2.创建成功之后调用lua_pushcfunction(L, &pmain)，向栈中压入一个c函数对象。

3.通过lua_pushinteger(L, argc)和lua_pushlightuserdata(L, argv)压入参数个数和参数。

4.通过lua_pcall(L, 2, 1, 0)来执行刚刚压入的函数。

5.通过lua_toboolean(L, -1)获取刚刚执行的结果。

6.通过lua_close(L)关闭lua状态机。

接下来我们看下每一步中，代码都做了些什么。

#### 

