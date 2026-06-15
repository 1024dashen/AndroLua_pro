# AndroLua+ APK 打包逻辑详解

## 概述

AndroLua+ 的 APK 打包采用**模板替换**策略：以自身 APK 为模板，替换用户 Lua 代码、修改 Manifest 配置、裁剪不必要的原生库和 Lua 模块，最终重新签名生成独立可运行的 APK。

## 涉及的关键文件

| 文件 | 职责 |
|------|------|
| `app/src/main/resources/lua/bin.lua` | 打包主逻辑，ZIP 构建、依赖分析、Manifest 修改、签名 |
| `app/src/main/assets/main.lua` | IDE 入口，触发打包、导出等操作 |
| `app/src/main/assets/init.lua` | 工程配置模板（appname、appver、packagename、权限等） |
| `app/src/main/resources/lua/import.lua` | `compile()` 函数，加载外部 dex 模块 |
| `app/src/main/java/com/androlua/LuaActivity.java` | `installApk()`、`newTask()` 等宿主方法 |
| `app/src/main/java/com/androlua/LuaAsyncTask.java` | 异步任务，打包在后台线程执行 |
| `app/src/main/java/com/androlua/LuaUtil.java` | 文件操作工具（copyFile、rmDir、getFileMD5） |
| `app/src/main/java/com/androlua/LuaDexLoader.java` | dex 模块加载器 |
| `app/src/main/assets/keys/testkey.pk8` | debug 签名私钥 |
| `app/src/main/assets/keys/testkey.x509.pem` | debug 签名证书 |
| 外部 `mao.dex` | 提供 `AXmlDecoder` 类，解析和修改二进制 AndroidManifest.xml |
| 外部 `sign.dex` | 提供 `Signer` 类和 `apksigner` 包，执行 APK 签名 |

> 注：`mao.dex` 和 `sign.dex` 为预编译的外部模块，不在源码仓库中，运行时通过 `compile` 动态加载。

## 完整流程

```
用户点击"打包"
       │
       ▼
读取 init.lua（appname / packagename / 权限等）
       │
       ▼
LuaAsyncTask 异步执行 binapk()
       │
       ├─ 1. 加载外部 dex 模块（compile "mao" / "sign"）
       ├─ 2. 依赖分析（checklib 扫描 require / import 语句）
       ├─ 3. 编译 Lua/ALY 脚本并写入新 ZIP 的 assets/
       ├─ 4. 从原始 APK 复制 classes.dex / resources.arsc 等非替换条目
       ├─ 5. AXmlDecoder 修改 AndroidManifest.xml
       ├─ 6. Signer.sign() 对 APK 签名
       └─ 7. activity.installApk() 唤起系统安装器
```

## 各阶段详细说明

### 阶段一：入口触发

**代码位置**：`main.lua` 第 1342-1348 行、`bin.lua` 第 384-395 行

用户在 IDE 中点击"打包"菜单项，触发 `func.build()`：

```lua
func.build = function()
    save()
    if not luaproject then
        Toast.makeText(activity, "仅支持工程打包.", Toast.LENGTH_SHORT).show()
        return
    end
    bin(luaproject .. "/")
end
```

`bin()` 函数加载工程的 `init.lua` 配置，然后创建异步任务：

```lua
local function bin(path)
    local p = {}
    local e, s = pcall(loadfile(path .. "init.lua", "bt", p))
    if e then
        create_error_dlg2()
        create_bin_dlg()
        bin_dlg.show()
        activity.newTask(binapk, update, callback).execute {
            path,                                           -- 工程路径
            activity.getLuaExtPath("bin", p.appname .. "_" .. p.appver .. ".apk")  -- 输出路径
        }
    else
        Toast.makeText(activity, "工程配置文件错误." .. s, Toast.LENGTH_SHORT).show()
    end
end
```

**工程配置（init.lua）示例**：

```lua
appname = "MyApp"
appver = "1.0"
appcode = "1"          -- 版本号（整数）
appsdk = "21"          -- 目标 SDK 版本
packagename = "com.example.myapp"
user_permission = {
    "INTERNET",
    "WRITE_EXTERNAL_STORAGE",
}
path_pattern = ".*\\.myext"  -- 可选，自定义文件关联
```

**异步回调**：
- `update(s)` — 更新进度对话框消息
- `callback(s)` — 打包完成回调，清理临时文件，显示结果

### 阶段二：加载外部 dex 模块

**代码位置**：`bin.lua` 第 56-57 行

```lua
compile "mao"    -- 加载 mao.dex → 提供 AXmlDecoder、mao.res 等类
compile "sign"   -- 加载 sign.dex → 提供 Signer、apksigner 等类
```

`compile` 函数定义在 `import.lua` 第 211-213 行：

```lua
function _M.compile(name)
    append(dexes, luacontext.loadDex(name))
end
```

内部调用 `LuaDexLoader.loadDex()`，使用 `DexClassLoader` 加载外部 dex 文件到类加载器中。

### 阶段三：依赖分析（checklib）

**代码位置**：`bin.lua` 第 108-174 行

打包时需要确定哪些原生库（.so）和 Lua 模块是用户代码实际依赖的，以减小输出 APK 体积。

**1. 初始化替换表**

```lua
local libs = File(activity.ApplicationInfo.nativeLibraryDir).list()
libs = luajava.astable(libs)
for k, v in ipairs(libs) do
    replace[v] = true    -- 标记所有 so 为"待排除"
end
```

同时遍历 `MdDir`（Lua 模块目录），标记所有内置 Lua 模块为"待排除"。

**2. 递归扫描依赖**

`checklib(path)` 函数读取 Lua 文件内容，用两个正则匹配：

```lua
-- 匹配 require "模块名"
for m, n in s:gmatch("require *%(? *\"([%w_]+)%.?([%w_]*)") do
    -- m = 模块名, n = 子模块名（可能为空）
end

-- 匹配 import "模块名"
for m, n in s:gmatch("import *%(? *\"([%w_]+)%.?([%w_]*)") do
end
```

对于每个匹配到的模块：
- 如果是原生模块（`lib{m}.so`），将其从排除列表中移除（`replace[cp] = false`，表示保留）
- 如果是 Lua 模块（`lua/{m}.lua`），同样保留，并递归扫描该模块自身的依赖

**3. 强制保留**

```lua
replace["libluajava.so"] = false    -- luajava 核心库始终保留
```

### 阶段四：编译脚本并写入 ZIP

**代码位置**：`bin.lua` 第 178-231 行

`addDir(out, dir, f)` 递归遍历工程目录，将所有文件写入新 ZIP：

| 文件类型 | 处理方式 |
|----------|----------|
| `.lua` | 调用 `console.build()` 编译为字节码，写入 `assets/{dir}{name}` |
| `.aly` | 调用 `console.build_aly()` 编译，扩展名改为 `.lua`，写入 `assets/{dir}{name}` |
| `.using` | 执行 `checklib()` 分析依赖 |
| `.apk` / `.luac` / `.*` | 跳过（不打包） |
| 子目录 | 递归调用 `addDir()` |
| 其他文件 | 直接复制到 `assets/{dir}{name}` |

**特殊文件**：

```lua
-- 工程目录下的 icon.png → 替换 APK 图标
if File(luapath .. "icon.png").exists() then
    -- 写入 res/drawable/icon.png
end

-- 工程目录下的 welcome.png → 替换启动图
if File(luapath .. "welcome.png").exists() then
    -- 写入 res/drawable/welcome.png
end
```

**编译后的 Lua 模块**（第 271-282 行）也会写入 ZIP：

```lua
for name, v in pairs(lualib) do
    local path, err = console.build(v)
    -- 写入 lua/{name}
end
```

**MD5 校验**：每个文件写入后计算 MD5，最终拼接到 ZIP 的 comment 字段：

```lua
table.insert(md5s, LuaUtil.getFileMD5(path))
-- ...
out.setComment(table.concat(md5s))
```

### 阶段五：从原始 APK 复制资源

**代码位置**：`bin.lua` 第 293-353 行

以当前 AndroLua+ APK（`info.publicSourceDir`）为模板，用 `ZipInputStream` 逐条读取：

```lua
local zipFile = File(info.publicSourceDir)
local zis = ZipInputStream(BufferedInputStream(FileInputStream(zipFile)))

local entry = zis.getNextEntry()
while entry do
    local name = entry.getName()
    -- 判断是否需要复制
    entry = zis.getNextEntry()
end
```

**过滤规则**：

| 条件 | 处理 |
|------|------|
| `replace[name] == true` | 跳过（已被新内容替换） |
| native so 且 `replace[lib] == true` | 跳过（未使用的 so） |
| `assets/` 开头 | 跳过（已被工程文件替换） |
| `lua/` 开头 | 跳过（已按需重新加入） |
| `META-INF/` 开头 | 跳过（旧签名，需要重新签名） |
| 其他 | 复制到新 ZIP |

被复制的条目包括 `classes.dex`、`resources.arsc`、非替换的 `res/` 资源、`lib/` 下的 so 库等。

### 阶段六：修改 AndroidManifest.xml

**代码位置**：`bin.lua` 第 306-347 行

这是打包最关键的步骤。AndroidManifest.xml 在 APK 中是二进制格式（AXML），需要用 `AXmlDecoder` 解析和修改：

```lua
local xml = AXmlDecoder.read(list, zis)
```

**替换内容**：

```lua
local req = {
    [activity.getPackageName()] = packagename,     -- 包名: com.androlua → 用户定义
    [info.nonLocalizedLabel]     = appname,         -- 应用名: AndroLua+ → 用户定义
    [ver]                        = appver,          -- 版本名
    [".*\\\\.alp"]               = path_pattern or "",  -- 文件关联
    [".*\\\\.lua"]               = "",              -- 清空 .lua 关联
    [".*\\\\.luac"]              = "",              -- 清空 .luac 关联
}
```

**权限过滤**：

```lua
for n = 0, list.size() - 1 do
    local v = list.get(n)
    if req[v] then
        list.set(n, req[v])          -- 替换已知字段
    elseif user_permission then
        local p = v:match("%.permission%.([%w_]+)$")
        if p and (not user_permission[p]) then
            list.set(n, "")          -- 未声明的权限清空
        end
    end
end
```

**修改版本号和 SDK 版本**（二进制操作）：

```lua
-- 将 versionCode 从当前值修改为用户定义的 appcode
s = string.gsub(s, touint32(code), touint32(tointeger(appcode) or 1), 1)
-- 将 targetSdkVersion 从 18 修改为用户定义的 appsdk
s = string.gsub(s, touint32(18), touint32(tointeger(appsdk) or 18), 1)
```

`touint32()` 函数将整数转为 4 字节小端序字节串，用于在二进制 XML 中定位和替换。

### 阶段七：APK 签名

**代码位置**：`bin.lua` 第 361-364 行

```lua
Signer.sign(tmp, apkpath)
```

使用 `Signer` 类（来自 sign.dex）对临时 APK 进行签名。签名使用的密钥为 `assets/keys/` 下的 debug 密钥：
- `testkey.pk8` — PKCS#8 格式私钥
- `testkey.x509.pem` — X.509 证书

签名完成后删除临时文件：

```lua
os.remove(tmp)
```

### 阶段八：安装

**代码位置**：`bin.lua` 第 365 行 → `LuaActivity.java` 第 841-848 行

```lua
activity.installApk(apkpath)
```

```java
public void installApk(String path) {
    Intent share = new Intent(Intent.ACTION_VIEW);
    File file = new File(path);
    share.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    share.setDataAndType(getUriForFile(file), getType(file));
    share.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    startActivity(Intent.createChooser(share, file.getName()));
}
```

通过 `ACTION_VIEW` Intent 唤起系统安装器，使用 FileProvider 提供 URI。

## 错误处理

- 编译失败时，`console.build()` / `console.build_aly()` 返回 `nil, err`，错误信息被收集到 `errbuffer`
- 所有文件处理完成后，如果 `errbuffer` 非空，则显示错误信息，不执行签名和安装
- 如果 `init.lua` 加载失败，直接 Toast 提示"工程配置文件错误"

## 打包输出

最终生成的 APK 结构：

```
MyApp_1.0.apk
├── AndroidManifest.xml          （已修改：包名、应用名、权限等）
├── classes.dex                  （从原始 APK 复制）
├── resources.arsc               （从原始 APK 复制）
├── res/
│   └── drawable/
│       ├── icon.png             （用户自定义图标，如有）
│       └── welcome.png          （用户自定义启动图，如有）
├── assets/
│   ├── main.lua                 （用户编译后的 Lua 字节码）
│   ├── init.lua                 （工程配置）
│   └── ...                      （其他用户文件）
├── lua/
│   ├── http.lua                 （按需包含的 Lua 模块）
│   └── ...
└── lib/
    └── armeabi/
        ├── libluajava.so         （始终包含）
        ├── libcrypt.so           （按需包含）
        └── ...
```

ZIP comment 中包含所有文件的 MD5 校验值拼接字符串。
