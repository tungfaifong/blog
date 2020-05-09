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

#### 二、Lua主流程

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

#### 三、全局状态机

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
  stringtable strt;  /* 字符串的哈希表 - hash table for strings */
  TValue l_registry;
  unsigned int seed;  /* randomized seed for hashes */
  lu_byte currentwhite;
  lu_byte gcstate;  /* gc的状态 - state of garbage collector */
  lu_byte gckind;  /* gc的类型 - kind of GC running */
  lu_byte gcrunning;  /* true if GC is running */
  GCObject *allgc;  /* list of all collectable objects */
  GCObject **sweepgc;  /* current position of sweep in list */
  GCObject *finobj;  /* list of collectable objects with finalizers */
  GCObject *gray;  /* list of gray objects */
  GCObject *grayagain;  /* list of objects to be traversed atomically */
  GCObject *weak;  /* list of tables with weak values */
  GCObject *ephemeron;  /* list of ephemeron tables (weak keys) */
  GCObject *allweak;  /* list of all-weak tables */
  GCObject *tobefnz;  /* list of userdata to be GC */
  GCObject *fixedgc;  /* list of objects not to be collected */
  struct lua_State *twups;  /* 具有开放上值的其他线程 - list of threads with open upvalues */
  unsigned int gcfinnum;  /* number of finalizers to call in each GC step */
  int gcpause;  /* size of pause between successive GCs */
  int gcstepmul;  /* GC 'granularity' */
  lua_CFunction panic;  /* to be called in unprotected errors */
  struct lua_State *mainthread; /* 主线程 */
  const lua_Number *version;  /* pointer to version number */
  TString *memerrmsg;  /* memory-error message */
  TString *tmname[TM_N];  /* 所有tag-method的名字 - array with tag-method names */
  struct Table *mt[LUA_NUMTAGS];  /* 所有基本类型的metatable - metatables for basic types */
  TString *strcache[STRCACHE_N][STRCACHE_M];  /* cache for strings in API */
} global_State;
```



