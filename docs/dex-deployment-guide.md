# AndroLua 闭源 dex 模块部署指南

## 📌 问题说明

AndroLua 的 APK 打包功能依赖两个闭源预编译模块：
- `mao.dex` - APK 打包核心组件
- `sign.dex` - APK 签名组件

这两个文件**不在开源代码中**，需要手动获取并部署。

## 🔍 获取方法

### 方法 1: 从已发布的 AndroLua+ APK 提取（推荐）

如果你有 AndroLua+ 的 APK 文件：

#### Windows PowerShell 脚本：
```powershell
# 设置 APK 路径
$apkPath = "C:\path\to\AndroLua.apk"

# 加载压缩程序集
Add-Type -AssemblyName System.IO.Compression.FileSystem

# 打开 APK
$zip = [System.IO.Compression.ZipFile]::OpenRead($apkPath)

# 查找目标 dex 文件
$dexFiles = $zip.Entries | Where-Object { $_.Name -match "^(mao|sign)\.dex$" }

# 提取到 assets 目录
$outputDir = "d:\ShenProject\AndroLua_pro\app\src\main\assets"
foreach ($entry in $dexFiles) {
    $destPath = Join-Path $outputDir $entry.Name
    [System.IO.Compression.ZipFileExtensions]::ExtractToFile($entry, $destPath, $true)
    Write-Host "✅ 已提取: $destPath" -ForegroundColor Green
}

$zip.Dispose()
```

#### 手动提取：
1. 将 `.apk` 文件改名为 `.zip`
2. 用解压软件打开
3. 在 `assets/` 目录找到 `mao.dex` 和 `sign.dex`
4. 复制到 `app/src/main/assets/` 目录

### 方法 2: 使用 ADB 从已安装的应用提取

如果手机上有正常工作的 AndroLua：

```bash
# 1. 提取 APK
adb pull /data/app/com.androlua-*/base.apk ./androlua.apk

# 2. 按方法 1 解压提取 dex 文件

# 或者直接提取已部署的 dex 文件（需要 root）
adb shell "cat /data/data/com.androlua/files/mao.dex" > mao.dex
adb shell "cat /data/data/com.androlua/files/sign.dex" > sign.dex
```

### 方法 3: 联系原作者或社区

- GitHub 仓库: https://github.com/worker500/AndroLua_pro
- 在 Issues 中询问 dex 文件获取方式
- 查找 AndroLua 相关社区或论坛

## 📦 部署步骤

### 步骤 1: 放置到 assets 目录

将获取到的 `mao.dex` 和 `sign.dex` 放到：
```
d:\ShenProject\AndroLua_pro\app\src\main\assets\mao.dex
d:\ShenProject\AndroLua_pro\app\src\main\assets\sign.dex
```

### 步骤 2: 重新编译 APK

```bash
# 在项目根目录执行
.\gradlew clean
.\gradlew assembleDebug
```

生成的 APK 位置：
```
app\build\outputs\apk\debug\app-debug.apk
```

### 步骤 3: 安装到手机

```bash
# 卸载旧版本（可选）
adb uninstall com.androlua

# 安装新版本
adb install app\build\outputs\apk\debug\app-debug.apk
```

### 步骤 4: 验证部署

应用首次启动时会自动将 dex 文件复制到：
```
/data/user/0/com.androlua/files/mao.dex
/data/user/0/com.androlua/files/sign.dex
```

使用文件管理器或 ADB 验证：
```bash
adb shell "ls -l /data/data/com.androlua/files/*.dex"
```

## 🔧 代码自动部署机制

已在 `LuaApplication.java` 中添加自动部署逻辑：

```java
// 应用启动时自动从 assets 复制 dex 文件到私有目录
copyDexFromAssets("mao.dex");
copyDexFromAssets("sign.dex");
```

**工作原理：**
1. 应用启动时检查 `/data/user/0/com.androlua/files/` 目录
2. 如果 dex 文件不存在，从 assets 复制
3. 如果已存在，跳过复制（避免覆盖用户自定义版本）

## ⚠️ 注意事项

1. **文件权限**: dex 文件必须有正确的读取权限
2. **文件大小**: 正常的 dex 文件通常在几百 KB 到几 MB
3. **版本匹配**: 确保 dex 文件与 AndroLua 版本兼容
4. **安全检查**: 只使用可信来源的 dex 文件

## 🐛 故障排查

### 问题 1: 仍然报错 "mao not found"

**检查清单：**
- [ ] dex 文件是否在 `app/src/main/assets/` 目录
- [ ] 是否重新编译并安装了 APK
- [ ] 应用是否有读写存储权限
- [ ] 检查日志: `adb logcat | grep -i "androlua"`

### 问题 2: 文件复制失败

**查看日志：**
```bash
adb logcat -s LuaApplication:I
```

**手动复制（临时方案）：**
```bash
adb push mao.dex /data/data/com.androlua/files/
adb push sign.dex /data/data/com.androlua/files/
adb shell chmod 644 /data/data/com.androlua/files/*.dex
```

### 问题 3: 签名错误

确保 `sign.dex` 文件完整且未损坏：
```bash
# 检查 MD5
adb shell "md5sum /data/data/com.androlua/files/sign.dex"
```

## 📝 目录结构

```
AndroLua_pro/
├── app/src/main/
│   ├── assets/
│   │   ├── mao.dex          ← 放置在这里
│   │   ├── sign.dex         ← 放置在这里
│   │   └── ...
│   └── java/com/androlua/
│       └── LuaApplication.java  ← 自动部署代码
└── ...

手机运行时：
/data/user/0/com.androlua/
└── files/
    ├── mao.dex              ← 自动复制到这里
    ├── sign.dex             ← 自动复制到这里
    └── ...
```

## 🎯 完成标志

成功部署后，你应该能够：
- ✅ 打开 AndroLua 应用无报错
- ✅ 使用 APK 打包功能
- ✅ 在日志中看到 dex 加载成功

---

**更新日期**: 2026-06-16
**适用版本**: AndroLua_pro (GitHub: worker500/AndroLua_pro)
