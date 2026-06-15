# 从零开发一个 AndroLua+ 类似项目 — 入门教程

本教程将带你从零理解 AndroLua+ 的架构设计，并逐步实现一个简化版的"Android 脚本引擎"项目。你将掌握嵌入脚本引擎、Lua-Java 桥接、声明式 UI、APK 打包等核心技术。

---

## 前置知识

在开始之前，你需要了解：

| 领域 | 要求 |
|------|------|
| Java | 熟练掌握面向对象、反射、泛型 |
| Android | 了解 Activity/Service 生命周期、View 体系、Intent 机制 |
| Lua | 基本语法（表、元表、闭包、require 机制） |
| JNI/NDK | 了解 Java 调用 C 的基本流程 |
| Gradle | 能看懂 build.gradle 配置 |

---

## 第一部分：理解项目全貌

### 1.1 AndroLua+ 是什么

一句话概括：**一个在 Android 上运行 Lua 脚本、并能将脚本打包成独立 APK 的集成开发环境。**

它由三层组成：

```
┌─────────────────────────────────────────┐
│           Lua 脚本层 (.lua)             │  ← 用户写的代码
├─────────────────────────────────────────┤
│         Lua-Java 桥接层 (luajava)        │  ← 让 Lua 能调 Java
├─────────────────────────────────────────┤
│    Android 宿主层 (Java + JNI + C)      │  ← 引擎、UI、打包
└─────────────────────────────────────────┘
```

### 1.2 核心模块一览

| 模块 | 位置 | 作用 |
|------|------|------|
| Lua 引擎 | `jni/lua/` | Lua 5.3.3 C 源码，编译为 liblua.a 静态库 |
| luajava 桥接 | `jni/luajava/luajava.c` + `java/com/luajava/` | JNI 层让 Lua 调 Java，Java 调 Lua |
| LuaContext 接口 | `java/com/androlua/LuaContext.java` | 定义脚本运行环境的统一接口 |
| LuaActivity | `java/com/androlua/LuaActivity.java` | 脚本运行的 Activity 宿主 |
| LuaApplication | `java/com/androlua/LuaApplication.java` | 全局初始化、目录管理 |
| LuaDexLoader | `java/com/androlua/LuaDexLoader.java` | 动态加载外部 dex/jar |
| LuaThread/Timer/Task | `java/com/androlua/LuaThread.java` 等 | 多线程脚本执行 |
| import 模块 | `resources/lua/import.lua` | 类/包导入、compile 加载 dex |
| loadlayout | `resources/lua/loadlayout.lua` | 声明式布局表系统 |
| bin.lua | `resources/lua/bin.lua` | APK 打包逻辑 |
| 代码编辑器 | `java/com/myopicmobile/textwarrior/` | 语法高亮编辑器控件 |

### 1.3 目录结构精简版

```
app/src/main/
├── AndroidManifest.xml
├── assets/                    # 随 APK 分发的资源
│   ├── init.lua               # 默认工程配置
│   ├── main.lua               # IDE 主界面脚本
│   ├── javaapi/               # Lua 封装的 Java API
│   ├── keys/                  # 签名密钥
│   └── layouthelper/          # 可视化布局助手
├── java/
│   ├── com/androlua/          # 宿主核心代码（50+ Java 文件）
│   ├── com/luajava/           # Lua-Java 桥接 Java 层（16 个文件）
│   ├── android/support/       # 兼容库
│   └── com/myopicmobile/      # 代码编辑器控件
├── jni/                       # C/JNI 原生代码
│   ├── lua/                   # Lua 5.3.3 源码
│   ├── luajava/               # luajava JNI 桥接
│   ├── cjson/                 # JSON 模块
│   ├── crypt/                 # 加密模块
│   └── ...                    # 其他 C 扩展模块
└── resources/lua/             # Lua 运行时模块（不进 assets，单独加载）
    ├── import.lua
    ├── loadlayout.lua
    ├── http.lua
    └── ...
```

---

## 第二部分：嵌入 Lua 引擎

这是整个项目的基础。你需要让 Android 应用能运行 Lua 脚本。

### 2.1 编译 Lua 为 Android 原生库

AndroLua+ 使用 NDK (ndk-build) 将 Lua C 源码编译为静态库，再链接到 luajava.so 中。

**步骤：**

1. 下载 [Lua 5.3.x 源码](https://www.lua.org/source/5.3/)
2. 在 `jni/lua/` 下放置源码，编写 `Android.mk`：

```makefile
# jni/lua/Android.mk
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE := liblua
LOCAL_CFLAGS := -DLUA_USE_POSIX
# 列出所有 .c 文件（排除 lua.c 和 luac.c，它们是独立可执行文件）
LOCAL_SRC_FILES := lapi.c lcode.c lctype.c ldebug.c ldo.c ldump.c \
                   lfunc.c lgc.c llex.c lmem.c lobject.c lopcodes.c \
                   lparser.c lstate.c lstring.c ltable.c ltm.c \
                   lundump.c lvm.c lzio.c lauxlib.c linit.c \
                   lutf8lib.c ltablib.c lcorolib.c liolib.c \
                   lmathlib.c loslib.c lstrlib.c ldblib.c
include $(BUILD_STATIC_LIBRARY)
```

3. 项目级 `jni/Android.mk` 引用子目录：

```makefile
APP_ABI = armeabi-v7a
include $(call all-subdir-makefiles)
```

### 2.2 构建 luajava JNI 桥接

luajava 是连接 Lua 和 Java 的桥梁。它由 C 层 (`luajava.c`) 和 Java 层 (`LuaState.java` 等) 组成。

**C 层核心职责：**
- 管理 `lua_State` 的生命周期
- 将 Java 对象推入 Lua 栈（作为 userdata）
- 从 Lua 栈取值转为 Java 类型
- 实现 `luajava.bindClass`、`luajava.newInstance`、`luajava.createProxy` 等原生函数

**`jni/luajava/Android.mk` 示例：**

```makefile
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_C_INCLUDES += $(LOCAL_PATH)/../lua
LOCAL_MODULE     := luajava
LOCAL_SRC_FILES  := luajava.c
LOCAL_LDLIBS     += -L$(SYSROOT)/usr/lib -llog -ldl
LOCAL_STATIC_LIBRARIES := liblua
include $(BUILD_SHARED_LIBRARY)
```

**Java 层关键类：**

| 类 | 作用 |
|----|------|
| `LuaState` | 封装 `lua_State` 指针，提供 Lua C API 的 Java 映射 |
| `LuaStateFactory` | 创建/管理 LuaState 实例（单例模式） |
| `LuaObject` | Lua 栈上任意值的 Java 包装，支持 `call()`、`getField()` 等 |
| `JavaFunction` | 抽象类，将 Java 方法注册为 Lua 函数 |
| `LuaException` | 统一异常 |

### 2.3 初始化 Lua 环境

在 `LuaActivity.initLua()` 中可以看到完整的初始化流程：

```java
private void initLua() throws Exception {
    // 1. 创建 Lua 状态机
    L = LuaStateFactory.newLuaState();
    
    // 2. 打开标准库
    L.openLibs();
    
    // 3. 将 Activity 对象暴露给 Lua，命名为 "activity"
    L.pushJavaObject(this);
    L.setGlobal("activity");
    
    // 4. 同时起个别名 "this"
    L.getGlobal("activity");
    L.setGlobal("this");
    
    // 5. 设置 luajava 的上下文（用于类加载）
    L.pushContext(this);
    
    // 6. 将路径信息存入 luajava 表
    L.getGlobal("luajava");
    L.pushString(luaExtDir);   L.setField(-2, "luaextdir");
    L.pushString(luaDir);      L.setField(-2, "luadir");
    L.pushString(luaPath);     L.setField(-2, "luapath");
    L.pop(1);
    
    // 7. 注册自定义 print 函数（重定向到界面显示）
    JavaFunction print = new LuaPrint(this, L);
    print.register("print");
    
    // 8. 设置 Lua 模块搜索路径
    L.getGlobal("package");
    L.pushString(luaLpath);    L.setField(-2, "path");
    L.pushString(luaCpath);    L.setField(-2, "cpath");
    L.pop(1);
    
    // 9. 注册线程间通信函数 set/call
    // ...
}
```

### 2.4 执行 Lua 脚本

```java
// 执行文件
public Object doFile(String path, Object... arg) {
    int ok = L.LloadFile(path);   // 编译
    if (ok == 0) {
        L.pcall(0, 0, 0);         // 执行
    }
    // ... 错误处理
}
```

### 2.5 练习：最小 Lua 宿主

尝试创建一个最小 Android 项目，只需：

1. 按上述步骤编译 `libluajava.so`
2. 创建一个 Activity，在 `onCreate` 中初始化 LuaState
3. 执行 `L.LdoString("print('Hello from Lua!')")`
4. 让 `print` 重定向到 `Log.d()` 或 `Toast`

成功后，你就拥有了脚本引擎的核心能力。

---

## 第三部分：Lua-Java 互操作

这是让 Lua 真正能操控 Android 的关键。

### 3.1 绑定 Java 类 (`luajava.bindClass`)

```lua
-- 在 Lua 中导入 Java 类
local Button = luajava.bindClass("android.widget.Button")
```

C 层实现原理：
1. 通过 JNI 调用 `Class.forName(name)` 获取 Class 对象
2. 将 Class 对象作为 userdata 推入 Lua 栈
3. 设置元表，使 `Button(...)` 调用触发 `newInstance`，`Button.XXX` 访问静态成员

### 3.2 创建 Java 对象 (`luajava.newInstance`)

```lua
local btn = luajava.newInstance("android.widget.Button", activity)
-- 或使用语法糖（类被导入后直接调用）
btn = Button(activity)
```

C 层通过反射找到匹配参数类型的构造方法，`CallNonvirtualObjectMethod` 创建实例。

### 3.3 调用 Java 方法

```lua
btn.setText("点击我")           -- 常规调用
btn.Text = "点击我"             -- setter 简写
local text = btn.Text           -- getter 简写
```

实现原理：
- 当 Lua 访问 `btn.setText` 时，元表 `__index` 通过反射查找 `setText` 方法
- 返回一个 Lua function，调用时通过 JNI 执行该方法
- getter/setter 简写：`btn.Text = "x"` → 元表 `__newindex` 检测到 `set` 前缀方法

### 3.4 创建 Java 接口代理 (`luajava.createProxy`)

```lua
-- 用 Lua 函数实现 Java 接口
local listener = luajava.createProxy(
    "android.view.View$OnClickListener",
    { onClick = function(v) print("clicked!") end }
)
btn.setOnClickListener(listener)
```

C 层使用 `java.lang.reflect.Proxy` 创建动态代理，将接口方法调用转发到 Lua 函数。

### 3.5 import 模块设计

AndroLua+ 的 `import.lua` 是最精巧的 Lua 代码之一，它实现了：

1. **`import "android.widget.*"`** — 导入整个包，返回一个懒加载表
2. **`import "android.widget.Button"`** — 导入单个类，放入全局环境
3. **`import "http"`** — 导入 Lua 模块（先 `require`，再尝试 Java 类）
4. **`compile "mao"`** — 动态加载 dex 模块

**懒加载包的实现**（关键代码）：

```lua
local pkgMT = {
    __index = function(T, classname)
        -- 当访问 pkg.Button 时，才真正加载类
        local ret, class = pcall(luajava.bindClass, rawget(T, "__name") .. classname)
        if ret then
            rawset(T, classname, class)  -- 缓存
            return class
        end
    end
}

local function import_pacckage(packagename)
    local pkg = { __name = packagename }
    setmetatable(pkg, pkgMT)
    return pkg
end
```

**全局自动补全**（当你写 `Button(...)` 时，Lua 查找 `_G.Button`）：

```lua
local globalMT = {
    __index = function(T, classname)
        -- 依次从已导入的包中查找
        for i, p in ipairs(loaders) do
            local class = loaded[classname] or p(classname)
            if class then
                T[classname] = class  -- 缓存到全局
                return class
            end
        end
        return nil
    end
}
setmetatable(_G, globalMT)
```

### 3.6 练习：实现 import 机制

1. 在你的最小项目中，实现 `import` 函数
2. 支持 `import "类全名"` 和 `import "包.*"` 两种形式
3. 设置 `_G` 的元表，使未定义的全局变量自动尝试从 Java 类加载

---

## 第四部分：声明式布局系统

AndroLua+ 最大的创新之一是用 Lua 表代替 XML 来描述 UI。

### 4.1 布局表语法

```lua
local layout = {
    LinearLayout,                    -- 控件类（表的第一个元素）
    orientation = "vertical",        -- 属性
    layout_width = "fill",
    layout_height = "fill",
    {                                 -- 子控件（数字索引的表）
        EditText,
        id = "edit",                 -- id 会成为全局变量名
        hint = "请输入",
        layout_width = "fill",
    },
    {
        Button,
        text = "确定",
        layout_width = "fill",
        onClick = function(v)        -- 事件直接绑定函数
            Toast.makeText(activity, edit.Text, 0).show()
        end,
    },
}
activity.setContentView(loadlayout(layout))
```

### 4.2 loadlayout 实现原理

核心函数在 `resources/lua/loadlayout.lua` 中，约 675 行。工作流程：

```
loadlayout(t, root)
    │
    ├─ 1. t[1] 是控件类 → 调用 t[1](context) 创建 View
    ├─ 2. 创建 LayoutParams，解析 layout_width/height
    ├─ 3. 设置 style（如有）
    ├─ 4. 遍历表的其他键值：
    │   ├─ 数字键 + table 值 → 递归 loadlayout 创建子 View
    │   ├─ "id" 键 → rawset(root, v, view) 注册全局变量
    │   └─ 其他键 → setattribute() 设置属性
    └─ 5. view.setLayoutParams(params) 返回
```

### 4.3 属性解析 (`setattribute`)

`setattribute` 函数处理几十种属性到 Java 方法的映射：

| 属性 | 处理方式 |
|------|---------|
| `layout_margin*` | `params.setMargins(...)` |
| `textSize` | `view.setTextSize(type, size)`，支持 `"14sp"` 格式 |
| `background` | 颜色 `"#FF0000"` / 图片路径 / LuaDrawable 函数 |
| `onClick` | `OnClickListener{onClick = func}` 创建代理 |
| `src` | 本地图片 / 网络图片（异步 task 加载） |
| `scaleType` | 字符串 `"fitCenter"` → 枚举值映射 |
| 通用属性 | `view["set" + 首字母大写(key)](value)` 反射调用 |

**值转换 (`checkValue`)** 是最巧妙的部分：

```lua
local function checkValue(var)
    return tonumber(var) or checkNumber(var) or var
end
```

`checkNumber` 能识别：
- `"fill"` / `"wrap"` → `-1` / `-2`（Android LayoutParams 常量）
- `"center"` → `17`（Gravity.CENTER）
- `"#FF0000"` → 颜色整数
- `"14sp"` / `"10dp"` → `TypedValue.applyDimension()` 换算
- `"50%w"` / `"30%h"` → 屏幕宽高百分比

### 4.4 练习：实现简化版 loadlayout

1. 实现一个 `loadlayout(t)` 函数，支持 `LinearLayout`、`TextView`、`Button`、`EditText`
2. 支持 `text`、`layout_width/height`、`id`、`onClick` 属性
3. 实现嵌套子视图（数字键 + 表值 = 子控件）

---

## 第五部分：多线程与异步

Android 不允许在主线程做耗时操作，Lua 脚本也需要异步能力。

### 5.1 三种异步方式

| 方式 | Java 类 | Lua 函数 | 特点 |
|------|---------|---------|------|
| Task | `LuaAsyncTask` | `task(code, args, callback)` | 后台执行，完成后主线程回调 |
| Thread | `LuaThread` | `thread(code, args)` | 独立 Lua 状态机，可双向通信 |
| Timer | `LuaTimer` | `timer(func, delay, period)` | 定时重复执行 |

### 5.2 LuaThread 通信机制

每个 `LuaThread` 拥有独立的 `LuaState`，通过 Handler 与主线程通信：

```
主线程 (LuaState L)          子线程 (LuaState L')
    │                              │
    │  t.call("func", args)        │
    ├─────────────────────────────→│  Handler 消息
    │                              │  执行 L'.func(args)
    │                              │
    │  call("func", args)          │
    │←─────────────────────────────┤  Handler 消息
    │  执行 L.func(args)            │  主线程 Handler
```

Java 层 `LuaThread` 同时实现 `LuaMetaTable` 接口，使 Lua 中可以写：

```lua
t = thread(code, 1, 2)   -- 创建线程
t.add()                    -- 调用线程中的 add 函数（通过 __index 元方法）
t.x = 100                  -- 设置线程变量（通过 __newindex 元方法）
```

### 5.3 LuaAsyncTask 实现

`LuaAsyncTask` 继承 `AsyncTaskX`，将 Lua 函数包装为异步任务：

```java
// 简化逻辑
public class LuaAsyncTask extends AsyncTaskX {
    private LuaObject mFunc;       // 后台执行的 Lua 函数
    private LuaObject mCallback;   // 主线程回调
    
    @Override
    protected Object doInBackground(Object[] args) {
        return mFunc.call(args);    // 在后台线程执行 Lua 函数
    }
    
    @Override
    protected void onPostExecute(Object result) {
        mCallback.call(result);     // 在主线程回调
    }
}
```

### 5.4 练习：实现 task 函数

1. 创建 `LuaAsyncTask` 子类
2. 在 `doInBackground` 中创建新 LuaState，执行传入的 Lua 代码
3. 在 `onPostExecute` 中调用回调函数

---

## 第六部分：APK 打包

打包是将用户脚本变成独立应用的关键功能。核心思路：**以自身 APK 为模板，替换和修改特定部分。**

> 完整逻辑详见 [docs/apk-build-logic.md](apk-build-logic.md)

### 6.1 打包流程概览

```
原始 APK (AndroLua+.apk)
         │
         ▼  ZipInputStream 逐条读取
    ┌────────────────────┐
    │ 判断每个 ZIP 条目：  │
    ├────────────────────┤
    │ assets/ → 替换为用户代码 │
    │ classes.dex → 原样保留  │
    │ lib/*.so → 按需裁剪     │
    │ AndroidManifest.xml → 修改 │
    │ META-INF/ → 跳过（重新签名）│
    └────────────────────┘
         │
         ▼  ZipOutputStream 写入新文件
    临时 APK
         │
         ▼  Signer.sign() 签名
    最终 APK
```

### 6.2 关键技术点

**1. 依赖分析** — 扫描所有 Lua 文件的 `require`/`import` 语句，确定需要哪些 .so 和 .lua 模块

```lua
for m, n in s:gmatch("require *%(? *\"([%w_]+)%.?([%w_]*)") do
    -- m=模块名, n=子模块名
    -- 标记 lib{m}.so 为"需要保留"
    -- 标记 lua/{m}.lua 为"需要保留"
end
```

**2. 编译 Lua 脚本** — `console.build(path)` 将 .lua 编译为 .luac 字节码，防止源码泄露

**3. 修改 AndroidManifest.xml** — APK 中的 Manifest 是二进制格式（AXML），需要专用解码器

```lua
local xml = AXmlDecoder.read(list, zis)
-- 替换包名、应用名、版本号
-- 过滤权限（未声明的清空）
-- 修改 versionCode 和 targetSdkVersion（二进制字节替换）
```

**4. APK 签名** — 使用 debug 密钥（testkey.pk8 + testkey.x509.pem）对齐并签名

```lua
Signer.sign(tmp, apkpath)
```

### 6.3 练习：实现简化版打包

1. 学习 Java `java.util.zip` 包的 `ZipInputStream` / `ZipOutputStream` 用法
2. 实现"读取一个 APK，修改其中一个 assets 文件，重新打包"的流程
3. 了解 Android 二进制 XML 格式，尝试用 AXML 解码器修改包名

---

## 第七部分：动态加载与插件系统

### 7.1 DexClassLoader 动态加载

AndroLua+ 支持运行时加载外部 dex/jar，扩展 Java 层能力：

```java
// LuaDexLoader.loadDex()
DexClassLoader dex = new LuaDexClassLoader(
    path,                    // dex/jar 文件路径
    odexDir,                 // 优化后的 dex 输出目录
    nativeLibraryDir,        // native 库目录
    parentClassLoader        // 父类加载器
);
dexCache.put(id, dex);
dexList.add(dex);
```

在 Lua 中通过 `compile "模块名"` 或 `import "模块名:类名"` 调用。

### 7.2 Native 库动态加载

```java
// LuaDexLoader.loadLib()
// 将项目目录下的 .so 文件复制到私有目录，再通过 Lua 的 package.cpath 加载
String libPath = libDir + "/lib" + fn + ".so";
LuaUtil.copyFile(luaDir + "/libs/lib" + fn + ".so", libPath);
```

### 7.3 练习：实现 dex 插件

1. 创建一个独立的 Android Library 模块，编译为 .jar
2. 用 `dx --output=plugin.jar library.jar` 转换为 dex 格式
3. 在主项目中用 `DexClassLoader` 加载并调用插件类的方法

---

## 第八部分：代码编辑器

AndroLua+ 集成了一个代码编辑器（基于 textwarrior 开源项目），具备：
- 语法高亮（Lua 关键字、字符串、注释、数字）
- 自动缩进
- 自动补全（Lua 全局变量 + Java 类名）
- 行号显示
- 搜索与替换

这部分相对独立，可以后期集成。开源编辑器选择：
- [CodeMirror](https://codemirror.net/)（WebView 方案）
- [Roslyn](https://github.com/dotnet/roslyn)（仅参考思路）
- [textwarrior](https://github.com/ricky9667/textwarrior)（AndroLua+ 使用的方案）

---

## 第九部分：组装完整项目

### 9.1 推荐开发顺序

```
Week 1-2:  Lua 引擎嵌入
           ├── 编译 liblua.a
           ├── 编译 libluajava.so
           └── Activity 中执行 Lua 脚本

Week 3-4:  Lua-Java 桥接增强
           ├── import 模块
           ├── getter/setter 简写
           └── 接口代理自动转换

Week 5-6:  声明式布局
           ├── loadlayout 核心逻辑
           ├── 属性解析与值转换
           └── onClick 事件绑定

Week 7-8:  多线程与异步
           ├── task 异步任务
           ├── thread 线程间通信
           └── timer 定时器

Week 9-10: APK 打包
           ├── ZIP 操作基础
           ├── Manifest 修改
           └── 签名

Week 11-12: IDE 与集成
           ├── 代码编辑器
           ├── 工程管理
           └── 调试功能
```

### 9.2 项目骨架

```
MyScriptEngine/
├── app/
│   ├── build.gradle
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/myscript/
│       │   ├── ScriptActivity.java      # 脚本宿主 Activity
│       │   ├── ScriptApplication.java   # 全局初始化
│       │   ├── ScriptContext.java        # 接口定义
│       │   ├── ScriptThread.java         # 多线程
│       │   ├── ScriptDexLoader.java      # 动态加载
│       │   └── ScriptUtil.java           # 工具方法
│       ├── jni/
│       │   ├── lua/                      # Lua 源码
│       │   ├── luajava/                  # 桥接层
│       │   └── Android.mk
│       ├── assets/
│       │   └── main.lua                  # 入口脚本
│       └── resources/lua/
│           ├── import.lua
│           └── loadlayout.lua
├── build.gradle
└── settings.gradle
```

### 9.3 build.gradle 关键配置

```groovy
android {
    compileSdkVersion 28
    defaultConfig {
        applicationId "com.myscript"
        minSdkVersion 21
        targetSdkVersion 28
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a'
        }
    }
    externalNativeBuild {
        ndkBuild {
            path 'src/main/jni/Android.mk'
        }
    }
}
```

---

## 第十部分：进阶主题

### 10.1 安全性

- **代码加密**：将 Lua 脚本编译为字节码（.luac），并在 C 层实现自定义加载器
- **反调试**：AndroLua+ 中有 Xposed 检测逻辑（`check2()`/`check3()`）
- **签名校验**：通过 ZIP comment 中的 MD5 链校验文件完整性

### 10.2 性能优化

- **方法缓存**：luajava 在 C 层缓存 Java 方法 ID，避免每次反射查找
- **setter 效率**：3.3 版本优化后 setter 效率提升 800%，核心是减少 JNI 调用次数
- **布局优化**：loadlayout 使用 `pcall` 包裹属性设置，单属性出错不影响整体

### 10.3 扩展方向

| 方向 | 思路 |
|------|------|
| 支持 JavaScript | 集成 V8 或 QuickJS，参考 luajava 设计 JS-Java 桥接 |
| 支持 Python | 集成 CPython，用 JNI 桥接 |
| 可视化编辑器 | 在 loadlayout 基础上，增加拖拽生成布局表的能力 |
| 云端编译 | 将 APK 打包放到服务器，支持 v2/v3 签名和渠道打包 |
| 插件市场 | 设计标准插件 API（.dex + .lua），实现下载和动态加载 |

---

## 附录

### A. 关键源码阅读清单

按阅读顺序排列：

1. `LuaContext.java` — 理解接口设计（63 行，最短）
2. `LuaApplication.java` — 理解全局初始化和目录管理
3. `LuaActivity.java` 的 `initLua()` + `onCreate()` — 理解 Lua 环境搭建
4. `LuaState.java` — 理解 Lua C API 的 Java 映射
5. `import.lua` — 理解类加载和模块系统（504 行，精华）
6. `loadlayout.lua` — 理解声明式布局（675 行，精华）
7. `bin.lua` — 理解 APK 打包（398 行）
8. `LuaThread.java` — 理解多线程通信
9. `LuaDexLoader.java` — 理解动态加载
10. `jni/luajava/luajava.c` — 理解 JNI 底层实现

### B. 推荐学习资源

| 主题 | 资源 |
|------|------|
| Lua C API | 《Programming in Lua》第 26-28 章 |
| JNI | Oracle JNI Specification |
| luajava 原理 | [kepler-project/luajava](https://github.com/kizzycode/luajava) |
| Android APK 结构 | Android官方文档 "Build System" |
| AXML 格式 | [android4me/AXMLParser](https://github.com/android4me/AXMLParser) |
| APK 签名 | [apksigner 文档](https://developer.android.com/studio/command-line/apksigner) |
| DexClassLoader | [动态加载技术](https://developer.android.com/reference/dalvik/system/DexClassLoader) |

### C. 常见坑点

1. **LuaState 不是线程安全的** — 每个线程必须创建独立的 LuaState
2. **Lua userdata 的 GC** — Java 对象被 Lua 引用时不释放，需要手动管理或使用 `LuaGcable`
3. **AXML 修改** — 二进制 XML 中字符串按索引引用，修改时不能改变已有条目的索引
4. **签名兼容** — Android 7.0+ 要求 APK v2 签名，v1 签名可能被拒绝
5. **Method 缓存失效** — 不同 ClassLoader 加载的同名类不是同一个类，缓存需按 ClassLoader 隔离
