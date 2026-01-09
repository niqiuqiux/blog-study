# Android 文件系统详解

## 📚 目录

1. [Android 文件系统概述](#1-android-文件系统概述)
2. [系统目录结构](#2-系统目录结构)
3. [APK 文件结构](#3-apk-文件结构)
4. [DEX 文件格式](#4-dex-文件格式)
5. [OAT/ART 运行时](#5-oatart-运行时)
6. [文件系统操作](#6-文件系统操作)
7. [权限与安全](#7-权限与安全)
8. [逆向分析中的应用](#8-逆向分析中的应用)

---

## 1. Android 文件系统概述

### 1.1 文件系统类型

#### 主要文件系统
- **ext4**: 主要文件系统（Android 4.0+）
- **f2fs**: Flash-Friendly File System（部分设备）
- **vfat**: FAT32（外部存储）

#### 分区结构
```
Android 设备存储
├── /system          # 系统分区（只读）
├── /data            # 数据分区（可读写）
├── /cache           # 缓存分区
├── /vendor          # 厂商分区（Android 8.0+）
└── /sdcard          # 外部存储（模拟或物理）
```

### 1.2 挂载点

#### 查看挂载信息
```bash
# 查看所有挂载点
adb shell mount

# 输出示例：
# /dev/block/platform/soc/1da4000.ufshc/by-name/system on /system type ext4 (ro,seclabel,relatime)
# /dev/block/platform/soc/1da4000.ufshc/by-name/userdata on /data type ext4 (rw,seclabel,nosuid,nodev,noatime)
```

#### 挂载选项说明
- **ro**: 只读（read-only）
- **rw**: 可读写（read-write）
- **noatime**: 不更新访问时间
- **seclabel**: SELinux 安全标签
- **nosuid**: 不允许 setuid
- **nodev**: 不允许设备文件

---

## 2. 系统目录结构

### 2.1 根目录 (/)

#### 主要目录
```bash
/                    # 根目录
├── system/          # 系统文件
├── data/            # 用户数据
├── cache/           # 缓存
├── vendor/          # 厂商文件
├── proc/            # 进程信息
├── sys/             # 系统信息
├── dev/             # 设备文件
├── sdcard/          # 外部存储
├── storage/         # 存储挂载点
├── mnt/             # 临时挂载点
└── acct/            # 进程统计
```

### 2.2 /system 目录

#### 目录结构
```
/system/
├── app/             # 系统应用 APK
├── priv-app/        # 特权系统应用
├── framework/       # 框架 JAR 文件
│   ├── framework.jar
│   ├── framework-res.apk
│   └── ...
├── lib/             # 系统库（32位）
├── lib64/           # 系统库（64位）
├── bin/             # 系统二进制文件
├── xbin/            # 扩展二进制文件
├── etc/             # 配置文件
├── fonts/           # 字体文件
├── media/           # 媒体文件
├── usr/             # 用户数据
└── build.prop       # 系统属性
```

#### 重要文件
```bash
# 查看系统属性
cat /system/build.prop

# 查看系统应用
ls /system/app/
ls /system/priv-app/

# 查看系统库
ls /system/lib/
ls /system/lib64/
```

### 2.3 /data 目录

#### 目录结构
```
/data/
├── data/            # 应用数据目录
│   └── <package_name>/
│       ├── files/           # 应用文件
│       ├── databases/       # 数据库
│       ├── shared_prefs/    # SharedPreferences
│       ├── cache/           # 缓存
│       └── lib/             # 应用库
├── app/             # 安装的 APK
│   └── <package_name>/
│       └── base.apk
├── dalvik-cache/    # Dalvik 缓存（旧版本）
├── app-private/     # 私有应用数据
├── system/          # 系统数据
│   ├── packages.xml         # 已安装包信息
│   ├── users/               # 用户信息
│   └── ...
└── local/           # 本地数据
```

#### 应用数据目录
```bash
# 查看应用数据（需要 root）
ls -la /data/data/com.example.app/

# 查看应用文件
ls /data/data/com.example.app/files/

# 查看数据库
ls /data/data/com.example.app/databases/

# 查看 SharedPreferences
ls /data/data/com.example.app/shared_prefs/
```

### 2.4 /proc 目录

#### 虚拟文件系统
`/proc` 是一个虚拟文件系统，提供内核和进程信息。

#### 重要文件
```bash
# 进程信息
/proc/<pid>/
├── cmdline          # 命令行参数
├── environ          # 环境变量
├── maps             # 内存映射
├── status           # 进程状态
├── stat             # 统计信息
└── fd/              # 文件描述符

# 系统信息
/proc/
├── version          # 内核版本
├── cpuinfo          # CPU 信息
├── meminfo          # 内存信息
├── mounts           # 挂载信息
└── ...
```

#### 查看进程内存映射
```bash
# 查看进程内存映射
cat /proc/<pid>/maps

# 输出示例：
# 12c00000-12c01000 r-xp 00000000 103:09 1441794 /system/lib/libc.so
# 地址范围    权限 偏移  设备   inode  文件路径
```

### 2.5 /sys 目录

#### 系统信息
```bash
/sys/
├── class/           # 设备类
├── devices/         # 设备信息
├── kernel/          # 内核信息
└── ...
```

### 2.6 外部存储

#### 目录结构
```
/sdcard/             # 主外部存储（可能是符号链接）
/storage/
├── emulated/        # 模拟存储
│   └── 0/          # 用户 0 的存储
├── sdcard0/         # 物理 SD 卡（如果有）
└── usbdisk/         # USB 存储（如果有）
```

#### 外部存储路径
```java
// 获取外部存储目录
File externalStorage = Environment.getExternalStorageDirectory();
// /storage/emulated/0 或 /sdcard

// 获取应用外部存储目录
File appExternalDir = getExternalFilesDir(null);
// /storage/emulated/0/Android/data/<package_name>/files
```

---

## 3. APK 文件结构

### 3.1 APK 文件格式

#### APK 本质
APK（Android Package）文件实际上是一个 ZIP 压缩文件，包含应用的所有资源。

#### 基本结构
```
app.apk (ZIP 格式)
├── META-INF/              # 签名和清单信息
│   ├── MANIFEST.MF        # 清单文件
│   ├── CERT.SF            # 签名文件
│   └── CERT.RSA           # 证书文件
├── AndroidManifest.xml    # 应用清单（二进制 XML）
├── classes.dex            # DEX 字节码文件
├── resources.arsc         # 编译后的资源索引
├── res/                   # 资源文件目录
│   ├── layout/            # 布局文件
│   ├── values/            # 值资源
│   ├── drawable/          # 图片资源
│   ├── mipmap/            # 应用图标
│   └── ...
└── assets/                # 原始资源文件（可选）
```

### 3.2 META-INF 目录

#### 签名文件
```bash
# MANIFEST.MF - 清单文件
# 包含所有文件的 SHA-1 摘要

# CERT.SF - 签名文件
# 包含 MANIFEST.MF 的签名

# CERT.RSA - 证书文件
# 包含公钥证书
```

#### 查看签名信息
```bash
# 使用 jarsigner 查看
jarsigner -verify -verbose -certs app.apk

# 使用 apksigner 查看（Android SDK）
apksigner verify --print-certs app.apk
```

### 3.3 AndroidManifest.xml

#### 二进制 XML 格式
AndroidManifest.xml 在 APK 中是二进制格式（AXML），需要工具解析。

#### 解析工具
```bash
# 使用 aapt
aapt dump xmltree app.apk AndroidManifest.xml

# 使用 apktool
apktool d app.apk
# 输出到 app/AndroidManifest.xml（文本格式）

# 使用 axml2xml（Python 工具）
axml2xml AndroidManifest.xml
```

#### 重要信息
- 包名（package）
- 版本信息（versionCode, versionName）
- 权限声明（uses-permission）
- 组件声明（activity, service, receiver, provider）
- 应用配置（application）

### 3.4 classes.dex

#### DEX 文件
- **格式**: Dalvik Executable
- **内容**: 编译后的 Java/Kotlin 代码
- **位置**: classes.dex（主文件），classes2.dex, ...（多 DEX）

#### 查看 DEX 信息
```bash
# 使用 dexdump
dexdump classes.dex

# 使用 baksmali（反编译为 Smali）
baksmali d classes.dex -o output/

# 使用 jadx（反编译为 Java）
jadx -d output/ app.apk
```

### 3.5 resources.arsc

#### 资源索引
- **格式**: 二进制资源表
- **内容**: 资源 ID 到资源值的映射
- **用途**: 快速查找资源

#### 解析资源
```bash
# 使用 aapt
aapt dump resources app.apk

# 使用 apktool
apktool d app.apk
# 输出到 app/res/ 目录
```

### 3.6 res/ 目录

#### 资源类型
```
res/
├── layout/          # 布局文件（XML）
├── values/          # 值资源（XML）
│   ├── strings.xml
│   ├── colors.xml
│   ├── dimens.xml
│   └── ...
├── drawable/        # 图片资源
│   ├── ic_launcher.png
│   └── ...
├── mipmap/          # 应用图标（多分辨率）
│   ├── mipmap-mdpi/
│   ├── mipmap-hdpi/
│   └── ...
├── raw/             # 原始资源文件
└── ...
```

#### 资源 ID
```java
// 资源 ID 格式：0x7f0a0001
// 7f: 应用资源标识
// 0a: 资源类型（如 layout）
// 0001: 资源索引

// 在代码中使用
R.layout.activity_main    // 0x7f0a0001
R.id.button              // 0x7f0a0002
R.string.app_name        // 0x7f0b0001
```

### 3.7 assets/ 目录

#### 原始资源
- **格式**: 原始文件（不编译）
- **访问**: 使用 AssetManager
- **用途**: 存储需要直接读取的文件

#### 访问 assets
```java
// 读取 assets 文件
AssetManager am = getAssets();
InputStream is = am.open("file.txt");
BufferedReader reader = new BufferedReader(new InputStreamReader(is));
String line = reader.readLine();
```

---

## 4. DEX 文件格式

### 4.1 DEX 文件结构

#### 基本结构
```
DEX 文件
├── Header            # 文件头
├── String Table      # 字符串表
├── Type Table        # 类型表
├── Proto Table       # 方法原型表
├── Field Table       # 字段表
├── Method Table      # 方法表
├── Class Table       # 类表
├── Data Section      # 数据区
└── Link Section      # 链接区（可选）
```

### 4.2 DEX 文件头

#### Header 结构
```c
struct DexHeader {
    u1  magic[8];           // "dex\n035\0" 或 "dex\n036\0"
    u4  checksum;           // 校验和
    u1  signature[20];      // SHA-1 签名
    u4  fileSize;           // 文件大小
    u4  headerSize;         // 头大小
    u4  endianTag;          // 字节序标记
    u4  linkSize;           // 链接区大小
    u4  linkOff;             // 链接区偏移
    u4  mapOff;              // 映射表偏移
    u4  stringIdsSize;       // 字符串 ID 数量
    u4  stringIdsOff;        // 字符串 ID 表偏移
    u4  typeIdsSize;         // 类型 ID 数量
    u4  typeIdsOff;          // 类型 ID 表偏移
    // ... 更多字段
};
```

#### 查看 DEX 头
```bash
# 使用 hexdump 查看
hexdump -C classes.dex | head -20

# 使用 dexdump
dexdump -h classes.dex
```

### 4.3 字符串表

#### 字符串存储
- **格式**: UTF-8 编码，以 null 结尾
- **索引**: 字符串 ID 表指向字符串数据区
- **用途**: 类名、方法名、字段名等

#### 解析字符串
```python
# Python 示例
def read_string(dex_file, string_id):
    # 读取字符串 ID 表
    string_id_offset = read_u32(dex_file, string_id * 4 + string_ids_off)
    # 读取字符串数据
    string_data = read_string_data(dex_file, string_id_offset)
    return string_data
```

### 4.4 类定义

#### 类结构
```c
struct DexClassDef {
    u4  classIdx;           // 类型索引
    u4  accessFlags;        // 访问标志
    u4  superclassIdx;      // 父类索引
    u4  interfacesOff;      // 接口偏移
    u4  sourceFileIdx;      // 源文件索引
    u4  annotationsOff;     // 注解偏移
    u4  classDataOff;       // 类数据偏移
    u4  staticValuesOff;    // 静态值偏移
};
```

#### 访问标志
- **ACC_PUBLIC**: 0x1
- **ACC_PRIVATE**: 0x2
- **ACC_PROTECTED**: 0x4
- **ACC_STATIC**: 0x8
- **ACC_FINAL**: 0x10
- **ACC_SYNCHRONIZED**: 0x20
- **ACC_NATIVE**: 0x100

### 4.5 方法定义

#### 方法结构
```c
struct DexMethod {
    u2  methodIdx;          // 方法 ID
    u2  accessFlags;        // 访问标志
    u4  codeOff;            // 代码偏移
};
```

#### 方法代码
```c
struct DexCode {
    u2  registersSize;      // 寄存器数量
    u2  insSize;            // 参数数量
    u2  outsSize;           // 输出参数数量
    u2  triesSize;          // try 块数量
    u4  debugInfoOff;       // 调试信息偏移
    u4  insnsSize;          // 指令大小
    u2  insns[];           // 指令数组
};
```

### 4.6 Dalvik 字节码

#### 指令格式
- **格式**: 16 位或 32 位指令
- **操作码**: 8 位操作码
- **操作数**: 寄存器、常量、方法引用等

#### 常见指令
```
opcode   指令        说明
0x00     nop         空操作
0x01     move        移动数据
0x12     const       加载常量
0x1a     const-string 加载字符串
0x6e     invoke-virtual 调用虚方法
0x0e     return-void 返回 void
0x0f     return      返回值
```

#### 查看字节码
```bash
# 使用 baksmali
baksmali d classes.dex -o output/

# 输出 Smali 代码
.method public onCreate(Landroid/os/Bundle;)V
    .registers 2
    .param p1, "savedInstanceState"    # Landroid/os/Bundle;
    
    .line 10
    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V
    
    return-void
.end method
```

---

## 5. OAT/ART 运行时

### 5.1 ART vs Dalvik

#### Dalvik（旧版本）
- **Android 4.4 及之前**
- **JIT 编译**: 运行时编译
- **DEX 文件**: 直接执行 DEX 字节码

#### ART（Android 5.0+）
- **AOT 编译**: 安装时编译为本地代码
- **OAT 文件**: 编译后的本地代码
- **性能提升**: 启动速度更快

### 5.2 OAT 文件格式

#### OAT 文件结构
```
OAT 文件
├── OAT Header        # OAT 文件头
├── DEX 文件          # 原始 DEX 文件
├── OAT Class         # OAT 类信息
├── OAT Method        # OAT 方法信息
└── Native Code       # 编译后的本地代码
```

#### OAT 文件位置
```bash
# 系统应用 OAT
/data/dalvik-cache/arm64/system@app@<package>@classes.dex@classes.oat

# 用户应用 OAT
/data/dalvik-cache/arm64/data@app@<package>@base.apk@classes.dex@classes.oat
```

### 5.3 OAT 文件分析

#### 查看 OAT 信息
```bash
# 使用 oatdump（需要 root）
oatdump --oat-file=classes.oat

# 使用 readelf（查看 ELF 结构）
readelf -h classes.oat
```

#### OAT 文件特点
- **ELF 格式**: OAT 文件是 ELF 格式
- **包含 DEX**: 原始 DEX 文件嵌入在 OAT 中
- **本地代码**: 包含编译后的 ARM/x86 代码

### 5.4 ART 运行时文件

#### 相关文件
```bash
/data/dalvik-cache/
├── arm64/            # 64位 ARM 架构
│   └── system@app@...@classes.dex@classes.oat
├── arm/              # 32位 ARM 架构
└── ...

/data/app/
└── <package>/
    └── oat/
        └── <arch>/
            └── base.odex    # 优化的 DEX
```

#### 提取 DEX
```bash
# 从 OAT 文件中提取 DEX
oatdump --oat-file=classes.oat --export-dex-to=output/

# 或使用工具
python oat2dex.py classes.oat
```

---

## 6. 文件系统操作

### 6.1 使用 adb 操作

#### 基本命令
```bash
# 列出文件
adb shell ls /data/data/

# 查看文件内容
adb shell cat /data/data/com.example.app/files/config.txt

# 复制文件到 PC
adb pull /data/data/com.example.app/files/config.txt ./

# 复制文件到设备
adb push config.txt /sdcard/

# 删除文件
adb shell rm /sdcard/file.txt

# 修改权限
adb shell chmod 644 /sdcard/file.txt
```

#### 需要 root 的操作
```bash
# 切换到 root
adb root

# 重新挂载为可写
adb remount

# 现在可以修改系统文件
adb shell rm /system/app/SomeApp.apk
```

### 6.2 使用 Python 脚本

#### 文件操作
```python
import subprocess

# 执行 adb 命令
def adb_shell(command):
    result = subprocess.run(['adb', 'shell', command],
                           capture_output=True, text=True)
    return result.stdout

# 读取文件
def read_file(path):
    content = adb_shell(f'cat {path}')
    return content

# 列出目录
def list_dir(path):
    files = adb_shell(f'ls -la {path}')
    return files.split('\n')
```

### 6.3 使用 Frida

#### 文件操作
```javascript
// 读取文件
var File = Java.use("java.io.File");
var file = File.$new("/sdcard/file.txt");
var fis = Java.use("java.io.FileInputStream").$new(file);
var bis = Java.use("java.io.BufferedInputStream").$new(fis);
var bytes = [];
var b;
while ((b = bis.read()) !== -1) {
    bytes.push(b);
}
var content = String.fromCharCode.apply(null, bytes);
console.log(content);

// 写入文件
var fos = Java.use("java.io.FileOutputStream").$new(file);
var bos = Java.use("java.io.BufferedOutputStream").$new(fos);
var contentBytes = [72, 101, 108, 108, 111]; // "Hello"
for (var i = 0; i < contentBytes.length; i++) {
    bos.write(contentBytes[i]);
}
bos.flush();
bos.close();
```

---

## 7. 权限与安全

### 7.1 文件权限

#### Linux 权限
```bash
# 权限格式：rwxrwxrwx
# 所有者  组用户  其他用户
# r=读, w=写, x=执行

# 查看权限
ls -l /data/data/com.example.app/

# 修改权限
chmod 644 file.txt    # rw-r--r--
chmod 755 script.sh   # rwxr-xr-x
chmod 600 secret.txt  # rw-------
```

#### SELinux 上下文
```bash
# 查看 SELinux 上下文
ls -Z /data/data/com.example.app/

# 输出示例：
# u:object_r:app_data_file:s0:c512,c768
```

### 7.2 应用沙箱

#### 数据隔离
- 每个应用有独立的数据目录
- 应用无法直接访问其他应用的数据
- 需要权限才能访问系统资源

#### 数据目录权限
```bash
# 应用数据目录（需要 root 或应用 UID）
/data/data/<package_name>/
# 权限：drwx------ (700)
# 所有者：应用 UID
```

### 7.3 存储权限

#### Android 10+ 作用域存储
- **应用私有目录**: 无需权限
- **媒体文件**: 需要 READ_MEDIA_* 权限
- **其他文件**: 需要 MANAGE_EXTERNAL_STORAGE 权限

#### 权限声明
```xml
<!-- Android 10 之前 -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

<!-- Android 10+ -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
```

---

## 8. 逆向分析中的应用

### 8.1 APK 分析流程

#### 基本流程
1. **获取 APK**: 从设备提取或下载
2. **解压 APK**: 查看文件结构
3. **分析清单**: 解析 AndroidManifest.xml
4. **反编译 DEX**: 使用 jadx 或 apktool
5. **分析资源**: 查看 res/ 目录
6. **分析 Native**: 分析 SO 文件

#### 工具链
```bash
# 1. 解压 APK
unzip app.apk -d app_extracted/

# 2. 解析 AndroidManifest.xml
aapt dump xmltree app.apk AndroidManifest.xml

# 3. 反编译 DEX
jadx -d output/ app.apk

# 4. 分析资源
aapt dump resources app.apk

# 5. 分析 SO 文件
file app_extracted/lib/arm64-v8a/libnative.so
readelf -h app_extracted/lib/arm64-v8a/libnative.so
```

### 8.2 提取应用数据

#### 提取 APK
```bash
# 从设备提取 APK
adb shell pm path com.example.app
# package:/data/app/com.example.app/base.apk

adb pull /data/app/com.example.app/base.apk ./

# 或使用包管理器
adb shell pm list packages
adb shell pm path com.example.app
```

#### 提取数据文件
```bash
# 需要 root
adb root
adb pull /data/data/com.example.app/ ./

# 提取数据库
adb pull /data/data/com.example.app/databases/ ./

# 提取 SharedPreferences
adb pull /data/data/com.example.app/shared_prefs/ ./
```

### 8.3 分析 DEX 文件

#### 反编译工具
```bash
# 使用 jadx（推荐）
jadx -d output/ app.apk

# 使用 apktool
apktool d app.apk -o output/

# 使用 baksmali
baksmali d classes.dex -o output/
```

#### 分析字节码
```bash
# 使用 dexdump
dexdump -d classes.dex > classes.dump

# 使用 jadx GUI
jadx-gui app.apk
```

### 8.4 分析 OAT 文件

#### 提取 DEX
```bash
# 从 OAT 提取 DEX
oatdump --oat-file=classes.oat --export-dex-to=output/

# 或使用工具
python oat2dex.py classes.oat
```

#### 分析本地代码
```bash
# 使用 IDA Pro 或 Ghidra
# OAT 文件是 ELF 格式，可以直接加载
```

### 8.5 文件系统取证

#### 提取关键信息
```bash
# 提取已安装应用列表
adb shell pm list packages > packages.txt

# 提取应用权限
adb shell dumpsys package com.example.app > package_info.txt

# 提取系统属性
adb shell getprop > build_prop.txt

# 提取进程信息
adb shell ps > processes.txt
```

---

## 9. 实用工具

### 9.1 命令行工具

#### aapt (Android Asset Packaging Tool)
```bash
# 查看 APK 信息
aapt dump badging app.apk

# 查看清单
aapt dump xmltree app.apk AndroidManifest.xml

# 查看资源
aapt dump resources app.apk

# 列出文件
aapt list app.apk
```

#### apktool
```bash
# 反编译
apktool d app.apk -o output/

# 重新打包
apktool b output/ -o new_app.apk
```

#### dexdump
```bash
# 转储 DEX 信息
dexdump classes.dex

# 反汇编
dexdump -d classes.dex
```

### 9.2 GUI 工具

#### jadx
```bash
# 命令行
jadx -d output/ app.apk

# GUI
jadx-gui app.apk
```

#### Android Studio
- 可以打开 APK 文件
- 查看 DEX 文件
- 分析资源

### 9.3 Python 工具

#### 解析 DEX
```python
# 使用 androguard
from androguard.core.bytecodes import apk

a = apk.APK('app.apk')
print(a.get_package())
print(a.get_activities())
print(a.get_permissions())
```

#### 解析 AndroidManifest.xml
```python
# 使用 axml2xml
from axmlparserpy import axmlprinter

ap = axmlprinter.AXMLPrinter(open('AndroidManifest.xml', 'rb').read())
print(ap.get_buff())
```

---

## 10. 总结

### 10.1 核心知识点
1. **系统目录**: /system, /data, /proc 等
2. **APK 结构**: ZIP 格式，包含 DEX、资源等
3. **DEX 格式**: Dalvik 字节码文件格式
4. **OAT/ART**: Android 5.0+ 的运行时格式
5. **文件操作**: adb, Python, Frida 等工具
6. **权限安全**: 文件权限、SELinux、应用沙箱

### 10.2 逆向分析要点
1. **APK 分析**: 解压、反编译、分析清单
2. **DEX 分析**: 反编译为 Java/Smali
3. **资源分析**: 提取和分析资源文件
4. **数据提取**: 从设备提取应用数据
5. **OAT 分析**: 从 OAT 提取 DEX

### 10.3 学习建议
1. **动手实践**: 分析真实的应用
2. **使用工具**: 熟悉各种分析工具
3. **理解格式**: 深入理解文件格式
4. **关注安全**: 了解权限和安全机制

---

