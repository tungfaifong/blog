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

  - ###### 基本数据类型：

  | 文件                | 作用                                 |
  | ------------------- | ------------------------------------ |
  | lobject.h/lobject.c | lua对象定义，和操作lua对象的通用函数 |
  | lfunc.h/lfunc.c     | 原型(prototypes)和闭包的操作函数     |
  | lstate.h/lstate.c   | 全局状态                             |
  | lstring.h/lstring.c | 字符串表(保存所有字符串)             |
  | ltable.h/ltable.c   | Lua表                                |

  - ###### C API：

  | 文件          | 作用                                      |
  | ------------- | ----------------------------------------- |
  | lapi.h/lapi.c | Lua API，实现了大部分Lua C API(lua_*函数) |

  - ###### 语法解析和代码生成：

  | 文件                | 作用                   |
  | ------------------- | ---------------------- |
  | lcode.h/lcode.c     | 代码生成器             |
  | lctype.h/lctype.c   | Lua对ctype.h的优化函数 |
  | ldump.c             | 保存预编译的Lua代码块  |
  | llex.h/llex.c       | 词法生成器             |
  | lparser.h/lparser.c | Lua语法解析器          |
  | lundump.h/lundump.c | 加载预编译的Lua代码块  |

  - ###### 通用函数：

  | 文件              | 作用                               |
  | ----------------- | ---------------------------------- |
  | ldebug.h/ldebug.c | Debug接口                          |
  | lgc.h/lgc.c       | 垃圾回收                           |
  | llimits.h         | 限制，基础类型和一些安装依赖的定义 |
  | lmem.h/lmem.c     | 内存管理                           |
  | lprefix.h         | 必须放在其他头文件前的Lua定义      |
  | lzio.h/lzio.c     | 通用的输入流接口                   |

  - ###### 虚拟机：

  | 文件                  | 作用                     |
  | --------------------- | ------------------------ |
  | ldo.h/ldo.c           | Lua栈和调用的结构        |
  | lopcodes.h/lopcodes.c | 虚拟机操作码             |
  | ltm.h/ltm.c           | 原方法，操作元方法的函数 |
  | lvm.h/lvm.c           | Lua虚拟机                |

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

