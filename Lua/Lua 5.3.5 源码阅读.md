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

  - ###### 其他：

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

```c
// lobject.h

/*
** Closures
*/

#define ClosureHeader \
	CommonHeader; lu_byte nupvalues; GCObject *gclist

typedef struct CClosure {
  ClosureHeader;
  lua_CFunction f;
  TValue upvalue[1];  /* list of upvalues */
} CClosure;

typedef struct LClosure {
  ClosureHeader;
  struct Proto *p;
  UpVal *upvals[1];  /* list of upvalues */
} LClosure;

typedef union Closure {
  CClosure c;
  LClosure l;
} Closure;
```

```c
// lfunc.h

/*
** Upvalues for Lua closures
*/
struct UpVal {
  TValue *v;  /* points to stack or to its own value */
  lu_mem refcount;  /* reference counter */
  union {
    struct {  /* (when open) */
      UpVal *next;  /* linked list */
      int touched;  /* mark to avoid cycles with dead threads */
    } open;
    TValue value;  /* the value (when closed) */
  } u;
};
```

Lua用一种成为upvalue的结构来实现闭包。对任何外层局部变量的存取间接通过upvalue来进行。upvalue最初指向栈中变量活跃的地方。当离开变量作用域的时候（超过变量生存期），变量被复制到upvalue中。

Lua针对每个变量，至少创建一个upvalue并按情况进行重复利用。Lua为整个运行栈保存了一个链表将所有upvalue都存起来。当一个新的闭包创建的时候，会先去遍历这个链表查看是不是有相对应的upvalue，若存在则直接使用，若不存在则创建一个新的upvalue并加入链表。当这个upvalue关闭后则从链表中删除，当所有闭包都不使用这个upvalue时，那它的储存空间则马上被释放。

在Lua中我们function区分了三个变种类型Lua闭包、纯C函数、C闭包，纯C函数我们可以看到Value中直接用f使用，而Lua闭包和C闭包有独立的结构体。首先我们可以看到两种闭包结构都会有一个闭包头ClosureHeader，ClosureHeader中有CommonHeader、表示上值数的nupvalues、和gc相关的gclist。

CClosure这个结构表示了C闭包的结构，这个结构首先包含了一个闭包头，紧接着一个lua_CFunction类型的f指向真正的c函数，upvalue代表上值的链表。

LClosure这个结构表示了Lua的闭包，首先还是具有一个闭包头，紧接着是一个原型类型的指针p，upvals代表上值的链表。

最后是一个统一的闭包联合结构。

##### thread：

```c
// lstate.h

typedef struct stringtable {
  TString **hash;
  int nuse;  /* number of elements */
  int size;
} stringtable;

/*
** Information about a call.
** When a thread yields, 'func' is adjusted to pretend that the
** top function has only the yielded values in its stack; in that
** case, the actual 'func' value is saved in field 'extra'.
** When a function calls another with a continuation, 'extra' keeps
** the function index so that, in case of errors, the continuation
** function can be called with the correct top.
*/
typedef struct CallInfo {
  StkId func;  /* function index in the stack */
  StkId	top;  /* top for this function */
  struct CallInfo *previous, *next;  /* dynamic call link */
  union {
    struct {  /* only for Lua functions */
      StkId base;  /* base for this function */
      const Instruction *savedpc;
    } l;
    struct {  /* only for C functions */
      lua_KFunction k;  /* continuation in case of yields */
      ptrdiff_t old_errfunc;
      lua_KContext ctx;  /* context info. in case of yields */
    } c;
  } u;
  ptrdiff_t extra;
  short nresults;  /* expected number of results from this function */
  unsigned short callstatus;
} CallInfo;

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

/*
** 'per thread' state
*/
struct lua_State {
  CommonHeader;
  unsigned short nci;  /* number of items in 'ci' list */
  lu_byte status;
  StkId top;  /* first free slot in the stack */
  global_State *l_G;
  CallInfo *ci;  /* call info for current function */
  const Instruction *oldpc;  /* last pc traced */
  StkId stack_last;  /* last free slot in the stack */
  StkId stack;  /* stack base */
  UpVal *openupval;  /* list of open upvalues in this stack */
  GCObject *gclist;
  struct lua_State *twups;  /* list of threads with open upvalues */
  struct lua_longjmp *errorJmp;  /* current error recover point */
  CallInfo base_ci;  /* CallInfo for first level (C calling Lua) */
  volatile lua_Hook hook;
  ptrdiff_t errfunc;  /* current error handling function (stack index) */
  int stacksize;
  int basehookcount;
  int hookcount;
  unsigned short nny;  /* number of non-yieldable calls in stack */
  unsigned short nCcalls;  /* number of nested C calls */
  l_signalT hookmask;
  lu_byte allowhook;
};
```

在Lua中thread类型代表的是协程类型，在每个协程中（lua_State）Lua具有两个栈（一个数据栈、一个调用栈），这样我们就可以在多级函数嵌套调用内挂起一个协程。

首先我们先在看下lua_State的结构，lua_State的成员比较多我们在这里先看看一些与调用相关的。

首先我们看看调用栈，在lua_State中调用栈是用一个CallInfo组成的双向链表来构成，在lua_State中ci指向调用栈的顶部，base_ci是调用栈的底部。

而数据栈则是采用数组才实现，stack指向堆栈的底部，stack_last指向最后一个空闲的栈空间，top指向栈顶，同样在CallInfo中也保存了调用的数据栈情况，fun指向的是该函数堆栈的开始，top是当前栈的顶部，base代表的是栈基址[func, base]代表的是可变参数，[base, top]是固定参数和本地变量。

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

首先我们先看下第一步创建Lua状态机的luaL_newstate：

```c
// lauxlib.c

static void *l_alloc (void *ud, void *ptr, size_t osize, size_t nsize) {
  (void)ud; (void)osize;  /* not used */
  if (nsize == 0) {
    free(ptr);
    return NULL;
  }
  else
    return realloc(ptr, nsize);
}

static int panic (lua_State *L) {
  lua_writestringerror("PANIC: unprotected error in call to Lua API (%s)\n",
                        lua_tostring(L, -1));
  return 0;  /* return to Lua to abort */
}

LUALIB_API lua_State *luaL_newstate (void) {
  lua_State *L = lua_newstate(l_alloc, NULL);
  if (L) lua_atpanic(L, &panic);
  return L;
}
```

luaL_newstate对lua_newstate做了一个封装指定了一个默认的内存分配器，默认的内存分配是直接调用C的realloc和free来做内存分配和释放。lua状态创建过程是在lua_newstate中：

```c
// lua_state.c

LUA_API lua_State *lua_newstate (lua_Alloc f, void *ud) {
  int i;
  lua_State *L;
  global_State *g;
  LG *l = cast(LG *, (*f)(ud, NULL, LUA_TTHREAD, sizeof(LG)));
  if (l == NULL) return NULL;
  L = &l->l.l;
  g = &l->g;
  L->next = NULL;
  L->tt = LUA_TTHREAD;
  g->currentwhite = bitmask(WHITE0BIT);
  L->marked = luaC_white(g);
  preinit_thread(L, g);
  g->frealloc = f;
  g->ud = ud;
  g->mainthread = L;
  g->seed = makeseed(L);
  g->gcrunning = 0;  /* no GC while building state */
  g->GCestimate = 0;
  g->strt.size = g->strt.nuse = 0;
  g->strt.hash = NULL;
  setnilvalue(&g->l_registry);
  g->panic = NULL;
  g->version = NULL;
  g->gcstate = GCSpause;
  g->gckind = KGC_NORMAL;
  g->allgc = g->finobj = g->tobefnz = g->fixedgc = NULL;
  g->sweepgc = NULL;
  g->gray = g->grayagain = NULL;
  g->weak = g->ephemeron = g->allweak = NULL;
  g->twups = NULL;
  g->totalbytes = sizeof(LG);
  g->GCdebt = 0;
  g->gcfinnum = 0;
  g->gcpause = LUAI_GCPAUSE;
  g->gcstepmul = LUAI_GCMUL;
  for (i=0; i < LUA_NUMTAGS; i++) g->mt[i] = NULL;
  if (luaD_rawrunprotected(L, f_luaopen, NULL) != LUA_OK) {
    /* memory allocation error: free partial state */
    close_state(L);
    L = NULL;
  }
  return L;
}
```

调用lua_newstate的时候会创建一个global_State和一个lua_State作为global_State的mainthread，接下来的其他动作主要都是对global_State和lua_State的初始化，其中我们来看下对lua_State初始化的f_luaopen部分。

```c
// lstate.c

/*
** open parts of the state that may cause memory-allocation errors.
** ('g->version' != NULL flags that the state was completely build)
*/
static void f_luaopen (lua_State *L, void *ud) {
  global_State *g = G(L);
  UNUSED(ud);
  stack_init(L, L);  /* init stack */
  init_registry(L, g);
  luaS_init(L);
  luaT_init(L);
  luaX_init(L);
  g->gcrunning = 1;  /* allow gc */
  g->version = lua_version(NULL);
  luai_userstateopen(L);
}
```

其中stack_init(L, L)对lua_State中的数据栈和调用栈进行了初始化、init_registry(L, g)针对全局的注册表进行了初始化、luaS_init(L)对string相关内容进行了初始化、lusT_init(L)对元表相关内容进行初始化、luaX_init(L)对词法分析器进行初始化、最后全部初始化完成后再对g设置一个version，将状态机标记为初始化完成。

```c
// lstate.c

static void stack_init (lua_State *L1, lua_State *L) {
  int i; CallInfo *ci;
  /* initialize stack array */
  L1->stack = luaM_newvector(L, BASIC_STACK_SIZE, TValue);
  L1->stacksize = BASIC_STACK_SIZE;
  for (i = 0; i < BASIC_STACK_SIZE; i++)
    setnilvalue(L1->stack + i);  /* erase new stack */
  L1->top = L1->stack;
  L1->stack_last = L1->stack + L1->stacksize - EXTRA_STACK;
  /* initialize first ci */
  ci = &L1->base_ci;
  ci->next = ci->previous = NULL;
  ci->callstatus = 0;
  ci->func = L1->top;
  setnilvalue(L1->top++);  /* 'function' entry for this 'ci' */
  ci->top = L1->top + LUA_MINSTACK;
  L1->ci = ci;
}
```

stack_init中可以看到lua_State中stack数据栈是直接采用数组的方式进行组织，而调用栈ci则是采用双向链表的方式进行组织。

```c
// lstate.c

/*
** Create registry table and its predefined values
*/
static void init_registry (lua_State *L, global_State *g) {
  TValue temp;
  /* create registry */
  Table *registry = luaH_new(L);
  sethvalue(L, &g->l_registry, registry);
  luaH_resize(L, registry, LUA_RIDX_LAST, 0);
  /* registry[LUA_RIDX_MAINTHREAD] = L */
  setthvalue(L, &temp, L);  /* temp = L */
  luaH_setint(L, registry, LUA_RIDX_MAINTHREAD, &temp);
  /* registry[LUA_RIDX_GLOBALS] = table of globals */
  sethvalue(L, &temp, luaH_new(L));  /* temp = new table (global table) */
  luaH_setint(L, registry, LUA_RIDX_GLOBALS, &temp);
}
```

接下来是初始化注册表，在init_registry中我们可以看到，首先是申请了一个新的table对象，将g的l_registry设为改新的table，而后在新的l_registry中插入两个对象，在l_registry[1]中放入lua_State对象L，在l_registry[2]中放入一个新的全局变量table。

```c
// lstring.c

/*
** Initialize the string table and the string cache
*/
void luaS_init (lua_State *L) {
  global_State *g = G(L);
  int i, j;
  luaS_resize(L, MINSTRTABSIZE);  /* initial size of string table */
  /* pre-create memory-error message */
  g->memerrmsg = luaS_newliteral(L, MEMERRMSG);
  luaC_fix(L, obj2gco(g->memerrmsg));  /* it should never be collected */
  for (i = 0; i < STRCACHE_N; i++)  /* fill cache with valid strings */
    for (j = 0; j < STRCACHE_M; j++)
      g->strcache[i][j] = g->memerrmsg;
}
```

luaS_init主要是用来初始化在global_State中string相关的内容，首先luaS_resize的时候会先给global_State中的stringtable分配一个初始大小的数组，然后同时将strcache全部初始化为无效的字符串。

```c
// ltm.c

void luaT_init (lua_State *L) {
  static const char *const luaT_eventname[] = {  /* ORDER TM */
    "__index", "__newindex",
    "__gc", "__mode", "__len", "__eq",
    "__add", "__sub", "__mul", "__mod", "__pow",
    "__div", "__idiv",
    "__band", "__bor", "__bxor", "__shl", "__shr",
    "__unm", "__bnot", "__lt", "__le",
    "__concat", "__call"
  };
  int i;
  for (i=0; i<TM_N; i++) {
    G(L)->tmname[i] = luaS_new(L, luaT_eventname[i]);
    luaC_fix(L, obj2gco(G(L)->tmname[i]));  /* never collect these names */
  }
}
```

luaT_init主要是初始化global_State中tmname数组，初始化元表会用到的内置函数。

在所有内容初始化完成之后将mainthread返回出去。

创建完Lua状态机之后，2、3步分别是将pmain函数和使用到的参数相关的内容push到lua_State的栈中（见lapi.c），然后在第4步中调用lua_pcall来调用pmain函数。

lua_pcall的调用顺序：lua_pcall->lua_pcallk->luaD_pcall->f_call->luaD_callnoyield->luaD_call。

```c
// ldo.c

/*
** Call a function (C or Lua). The function to be called is at *func.
** The arguments are on the stack, right after the function.
** When returns, all the results are on the stack, starting at the original
** function position.
*/
void luaD_call (lua_State *L, StkId func, int nResults) {
  if (++L->nCcalls >= LUAI_MAXCCALLS)
    stackerror(L);
  if (!luaD_precall(L, func, nResults))  /* is a Lua function? */
    luaV_execute(L);  /* call it */
  L->nCcalls--;
}
```

在Lua中调用一个函数，最后都会走到luaD_call这个函数中，我们可以看到这个函数，首先会调用luaD_precall，在luaD_precall中回去判断该调用的函数是那种类型的Closure，如果是C闭包或者C函数那就会直接调用，如果是Lua闭包则返回，等待luaV_execute执行，在luaD_precall中，会创建一个新的callinfo放到lua_State的ci链表中。

接下来我们看下在pmain函数中的doREPL函数，doREPL主要是lua.c编译出来之后交互式解释器的主要的一个函数，我们借此来看下lua代码是怎么被执行的。

```c
// lua.c

/*
** Do the REPL: repeatedly read (load) a line, evaluate (call) it, and
** print any results.
*/
static void doREPL (lua_State *L) {
  int status;
  const char *oldprogname = progname;
  progname = NULL;  /* no 'progname' on errors in interactive mode */
  while ((status = loadline(L)) != -1) {
    if (status == LUA_OK)
      status = docall(L, 0, LUA_MULTRET);
    if (status == LUA_OK) l_print(L);
    else report(L, status);
  }
  lua_settop(L, 0);  /* clear stack */
  lua_writeline();
  progname = oldprogname;
}
```

从doREPL中可以看到整个交互式解释器就是通过一个死循环不断通过loadline读入代码，然后再调用docall执行代码。

loadline的调用顺序：loadline->addreturn(or multiline)->luaL_loadbuffer->luaL_loadbufferx->lua_load->luaD_protectedparser->f_parser。

```c
// ldo.c

static void f_parser (lua_State *L, void *ud) {
  LClosure *cl;
  struct SParser *p = cast(struct SParser *, ud);
  int c = zgetc(p->z);  /* read first character */
  if (c == LUA_SIGNATURE[0]) {
    checkmode(L, p->mode, "binary");
    cl = luaU_undump(L, p->z, p->name);
  }
  else {
    checkmode(L, p->mode, "text");
    cl = luaY_parser(L, p->z, &p->buff, &p->dyd, p->name, c);
  }
  lua_assert(cl->nupvalues == cl->p->sizeupvalues);
  luaF_initupvals(L, cl);
}
```

在f_parser中会去判断读入的数据是二进制数据还是文本数据，再分别通过luaU_undump和luaY_parser做解析。这里我们看下luaY_parser做了些什么。

```c
// lparser.c

LClosure *luaY_parser (lua_State *L, ZIO *z, Mbuffer *buff,
                       Dyndata *dyd, const char *name, int firstchar) {
  LexState lexstate;
  FuncState funcstate;
  LClosure *cl = luaF_newLclosure(L, 1);  /* create main closure */
  setclLvalue(L, L->top, cl);  /* anchor it (to avoid being collected) */
  luaD_inctop(L);
  lexstate.h = luaH_new(L);  /* create table for scanner */
  sethvalue(L, L->top, lexstate.h);  /* anchor it */
  luaD_inctop(L);
  funcstate.f = cl->p = luaF_newproto(L);
  funcstate.f->source = luaS_new(L, name);  /* create and anchor TString */
  lua_assert(iswhite(funcstate.f));  /* do not need barrier here */
  lexstate.buff = buff;
  lexstate.dyd = dyd;
  dyd->actvar.n = dyd->gt.n = dyd->label.n = 0;
  luaX_setinput(L, &lexstate, z, funcstate.f->source, firstchar);
  mainfunc(&lexstate, &funcstate);
  lua_assert(!funcstate.prev && funcstate.nups == 1 && !lexstate.fs);
  /* all scopes should be correctly finished */
  lua_assert(dyd->actvar.n == 0 && dyd->gt.n == 0 && dyd->label.n == 0);
  L->top--;  /* remove scanner's table */
  return cl;  /* closure is on the stack, too */
}
```

当我们读入一段代码的时候parser会先创建一个LClosure放入lua_State的数据栈中同时会创建一个Proto，然后针对输入的内容进行词法分析和语法分析，此处我们不深入去看词法分析器和语法分析器做了什么，我们通过语法分析和词法分析之后会向Proto中放入一系列分析后的指令。

然后我们再看来下docall，docall的流程和上面提到pmain的调用顺序是一样的，最后还是会调用到luaD_call，生成一个ci之后通过Lua的虚拟机执行刚刚解析得出的指令。

#### 四、虚拟机

现有主流的虚拟机主要是基于栈或者基于寄存器来实现，基于栈的虚拟机有JVM、CPython等，而基于寄存器的主要是Lua等。

基于栈的虚拟机主要是实现了一个操作栈，所有操作都在栈上执行，特点是实现方便，但是由于大量指令操作速度会比较慢。

基于寄存器的主要实现方式，则是一条指令中包含了操作和寄存器的地址，在一条指令中实现一个操作，基于寄存器的指令更为简洁，不过实现会更为复杂。

举一个例子：a = b + c

```
// 基于栈
I1:LOAD C
I2:LOAD B
I3:ADD
I4:STORE A

// 基于寄存器
I1:ADD A B C
```

可以看到基于栈的虚拟机指令会比基于寄存器的虚拟机更多。

##### 虚拟机指令：

Lua的虚拟机指令将32位分为2个、3个或4个域，如下：

```
|6	|8	|9	|9	|
|OP	|A	|B	|C	|
|OP	|A	|Bx		|
|OP	|A	|sBx	|
|OP	|Ax			|
```

OP域占6位，代表了指令的操作数。A域是一定存在的占8位，B、C域各占9位。B、C域可以组合成一个18位的Bx域（无符号）或sBx域（有符号），A、B、C可以组成一个26位的Ax域。

大部分指令使用三地址格式，其中A代表目的寄存器；B、C分别代表源操作数，可以是寄存器或常数。

```c
// lopcodes.h

/*
** R(x) - register
** Kst(x) - constant (in constant table)
** RK(x) == if ISK(x) then Kst(INDEXK(x)) else R(x)
*/

/*
** grep "ORDER OP" if you change these enums
*/

typedef enum {
/*----------------------------------------------------------------------
name		args	description
------------------------------------------------------------------------*/
OP_MOVE,/*	A B	R(A) := R(B)					*/
OP_LOADK,/*	A Bx	R(A) := Kst(Bx)					*/
OP_LOADKX,/*	A 	R(A) := Kst(extra arg)				*/
OP_LOADBOOL,/*	A B C	R(A) := (Bool)B; if (C) pc++			*/
OP_LOADNIL,/*	A B	R(A), R(A+1), ..., R(A+B) := nil		*/
OP_GETUPVAL,/*	A B	R(A) := UpValue[B]				*/

OP_GETTABUP,/*	A B C	R(A) := UpValue[B][RK(C)]			*/
OP_GETTABLE,/*	A B C	R(A) := R(B)[RK(C)]				*/

OP_SETTABUP,/*	A B C	UpValue[A][RK(B)] := RK(C)			*/
OP_SETUPVAL,/*	A B	UpValue[B] := R(A)				*/
OP_SETTABLE,/*	A B C	R(A)[RK(B)] := RK(C)				*/

OP_NEWTABLE,/*	A B C	R(A) := {} (size = B,C)				*/

OP_SELF,/*	A B C	R(A+1) := R(B); R(A) := R(B)[RK(C)]		*/

OP_ADD,/*	A B C	R(A) := RK(B) + RK(C)				*/
OP_SUB,/*	A B C	R(A) := RK(B) - RK(C)				*/
OP_MUL,/*	A B C	R(A) := RK(B) * RK(C)				*/
OP_MOD,/*	A B C	R(A) := RK(B) % RK(C)				*/
OP_POW,/*	A B C	R(A) := RK(B) ^ RK(C)				*/
OP_DIV,/*	A B C	R(A) := RK(B) / RK(C)				*/
OP_IDIV,/*	A B C	R(A) := RK(B) // RK(C)				*/
OP_BAND,/*	A B C	R(A) := RK(B) & RK(C)				*/
OP_BOR,/*	A B C	R(A) := RK(B) | RK(C)				*/
OP_BXOR,/*	A B C	R(A) := RK(B) ~ RK(C)				*/
OP_SHL,/*	A B C	R(A) := RK(B) << RK(C)				*/
OP_SHR,/*	A B C	R(A) := RK(B) >> RK(C)				*/
OP_UNM,/*	A B	R(A) := -R(B)					*/
OP_BNOT,/*	A B	R(A) := ~R(B)					*/
OP_NOT,/*	A B	R(A) := not R(B)				*/
OP_LEN,/*	A B	R(A) := length of R(B)				*/

OP_CONCAT,/*	A B C	R(A) := R(B).. ... ..R(C)			*/

OP_JMP,/*	A sBx	pc+=sBx; if (A) close all upvalues >= R(A - 1)	*/
OP_EQ,/*	A B C	if ((RK(B) == RK(C)) ~= A) then pc++		*/
OP_LT,/*	A B C	if ((RK(B) <  RK(C)) ~= A) then pc++		*/
OP_LE,/*	A B C	if ((RK(B) <= RK(C)) ~= A) then pc++		*/

OP_TEST,/*	A C	if not (R(A) <=> C) then pc++			*/
OP_TESTSET,/*	A B C	if (R(B) <=> C) then R(A) := R(B) else pc++	*/

OP_CALL,/*	A B C	R(A), ... ,R(A+C-2) := R(A)(R(A+1), ... ,R(A+B-1)) */
OP_TAILCALL,/*	A B C	return R(A)(R(A+1), ... ,R(A+B-1))		*/
OP_RETURN,/*	A B	return R(A), ... ,R(A+B-2)	(see note)	*/

OP_FORLOOP,/*	A sBx	R(A)+=R(A+2);
			if R(A) <?= R(A+1) then { pc+=sBx; R(A+3)=R(A) }*/
OP_FORPREP,/*	A sBx	R(A)-=R(A+2); pc+=sBx				*/

OP_TFORCALL,/*	A C	R(A+3), ... ,R(A+2+C) := R(A)(R(A+1), R(A+2));	*/
OP_TFORLOOP,/*	A sBx	if R(A+1) ~= nil then { R(A)=R(A+1); pc += sBx }*/

OP_SETLIST,/*	A B C	R(A)[(C-1)*FPF+i] := R(A+i), 1 <= i <= B	*/

OP_CLOSURE,/*	A Bx	R(A) := closure(KPROTO[Bx])			*/

OP_VARARG,/*	A B	R(A), R(A+1), ..., R(A+B-2) = vararg		*/

OP_EXTRAARG/*	Ax	extra (larger) argument for previous opcode	*/
} OpCode;
```

Lua有47条操作码，具体的操作码和参数我们可以从上面的代码中看到，其中R(X)代表寄存器，Kst(x)代表常数（在常量表中的位置），RK(x)代表如果是常数的话则返回常数否则返回寄存器。

#### 五、垃圾回收



