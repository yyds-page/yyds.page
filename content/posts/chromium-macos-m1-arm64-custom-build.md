---
title: "Chromium Checkout & Custom Build (on Apple M1 Chip)"
date: 2022-01-09T21:44:59+08:00
---

# chromium 国内下载&自定义编译（在Apple M1芯片）

## 说明

作者想要编译Chromium, 但手头是Apple Silicon (M1 Pro)开发设备，相关资料都是X86 CPU架构的，想要编译却受限于相关中文资料匮乏。

经过一番摸（踩）索（坑），编译出了M1可用的Arm 64版本，分享于此，方便后来人。

    注： Chromium采用的是Stable分支的`96.0.4664.125`

## 参考信息

* https://www.chromium.org/developers/how-tos/get-the-code
* https://www.chromium.org/developers/how-tos/old-get-the-code


## 一、Checkout, Build, & Run Chromium


### 1. 设置Http/Https代理 
```shell
# 设置环境变量
export http_proxy=http://127.0.0.1:7070;export https_proxy=http://127.0.0.1:7070;

# gclient使用代理配置
cat > ~/.boto <<EOF
[Boto]
proxy = 127.0.0.1
# 端口号和上面相同
proxy_port = 1087
proxy_type = http
#ca_certificates_file = /path/to/certificate.crt
EOF

# 设置gclient代理环境变量
export NO_AUTH_BOTO_CONFIG=~/.boto

# 配置git参数
git config --global pack.packSizeLimit 1g
git config --global pack.deltaCacheSize 1g
git config --global pack.windowMemory 1g
git config --global core.packedGitLimit 1g
git config --global core.packedGitWindowSize 1g
git config --global core.precomposeUnicode true

```


### 2. 下载chromium 96.0.4664.125

这里做了改进，不需要下载官方说明里的大量文件，属于最快的方式

* 解压后文件，Linux(Ubuntu 18.04)上大约26G, MacOS 20G（包括三方包，未排除.git目录）

```shell
cd ~ 

git clone  -v --progress --depth 1 https://chromium.googlesource.com/chromium/tools/depot_tools.git

export PATH="$PATH:$HOME/depot_tools"

mkdir chromium && cd chromium

# 配置gclient
gclient config --spec 'solutions = [
  {
    "name": "src",
    "url": "https://chromium.googlesource.com/chromium/src.git",
    "managed": False,
    "custom_deps": {},
    "custom_vars": {},
  },
]
'

# 查看配置
cat .gclient

# 下载chromium的96稳定版本
# 克隆指定标签，最小化下载代码，无分支和提交日志数据
time git clone -v --progress --depth 1 --branch 96.0.4664.125 https://chromium.googlesource.com/chromium/src.git

cd src

# 根据DEPS文件和操作系统类型下载依赖到相应目录，最终下载的依赖数量可在~/chromiun/.gclient_entries文件中(约120个条目)
time gclient sync -v --jobs=16 --nohooks --no-history

```


### 3. 安装依赖
```shell
# 若是Linux系统，安装依赖，其他系统不执行
./build/install-build-deps.sh

# 若为MacOS, 解决两个LASTCHANGE文件不存在的问题
# 针对96.0.4664.125版本
cat > build/util/LASTCHANGE <<EOF
LASTCHANGE=f9fcf99e5f982d3e9e3fe439c0e7c69ddc965761-refs/heads/96.0.4664.125
LASTCHANGE_YEAR=2021
EOF

cat > build/util/LASTCHANGE.committime <<EOF
1640311313
EOF

cat > build/util/LASTCHANGE.dummy <<EOF
LASTCHANGE=0000000000000000000000000000000000000000-0000000000000000000000000000000000000000
EOF
#或者（不推荐）
./build/util/lastchange.py  build/util/LASTCHANGE
./build/util/lastchange.py -s third_party/blink/ -o build/util/LASTCHANGE.blink

# 执行Hooks，会下载一些二进制文件和三方包build-tools
time gclient runhooks -v --jobs=8

# 特别注意: M1 ARM: vpython软件包在M1 ARM系统上的问题解决：使用高版本chromiun的src/.vpython3替换旧的src/.vpython3
# 主要为numpy, see https://chromium.googlesource.com/chromium/src.git/+/refs/heads/main/.vpython3#60
# # wheel: <
#   name: "infra/python/wheels/numpy/${vpython_platform}"
#   version: "version:1.20.3"
#   # A newer version of numpy is required on ARM64, but it breaks older OS versions.
#   not_match_tag <
#     platform: "macosx_11_0_arm64"
#   >
# >
# wheel: <
#   name: "infra/python/wheels/numpy/mac-arm64_cp38_cp38"
#   version: "version:1.21.1"
#   match_tag <
#     platform: "macosx_11_0_arm64"
#   >
# >
# 然后继续执行 time gclient runhooks -v --jobs=8

```


### 4. 编译

* 编译选项
  * Tips: 查看当前所有编译选项：`gn args --list <out_dir>`

- Release编译选项：
```shell
is_debug=false
dcheck_always_on=true
symbol_level=0
# use_jumbo_build
is_component_build=false
is_official_build=true
enable_nacl=false
blink_symbol_level=0
chrome_pgo_phase=0
cc_wrapper="ccache"
enable_js_type_check=false
enable_remoting=false
enable_extensions=true
```

* MacOS
```shell
brew install ccache

sudo xcode-select -r

# 生成配置
gn gen out/Profile --args='is_debug=false dcheck_always_on=true symbol_level=0 is_component_build=false is_official_build=true enable_nacl=false blink_symbol_level=0 chrome_pgo_phase=0 cc_wrapper="ccache" enable_js_type_check=false enable_remoting=false enable_extensions=true '

# 编译（时间根据cpu能力不同，首次编译耗时1-6个小时不等）
time autoninja -C out/Profile chrome

```

* Windows
```powershell
# TODO: 
```

* Linux
```shell
apt install -y ccache

gn gen out/Profile --args='is_debug=false dcheck_always_on=true symbol_level=0 is_component_build=false is_official_build=true enable_nacl=false blink_symbol_level=0 chrome_pgo_phase=0 cc_wrapper="ccache" enable_js_type_check=false enable_remoting=false enable_extensions=true '

time autoninja -C out/Profile chrome

```


## 二、其他

### 附1：删除不必要的文件(可选)

```shell
# 编译.git的大小
find . -name .git | xargs du -hsm | cut -f 1 | awk '{s+=$1} END {printf "total .git: %d MB\n", s}'
# output: total .git: 9571 MB

# 可选：删除.git目录，因占用空间，在使用期间不要删除
find . -name .git | xargs rm -rf
```

### 附2：MacOS M1编译错误解决：

error: functions that differ only in their return type cannot be overloaded

summary: Get target "chrome" to build with the macOS 12.0 SDK

see patch: https://chromium.googlesource.com/chromium/src.git/+/11a0c895953506d063c3dc8076a6bfd1f037eaf5%5E%21/

补丁涉及文件：`accessibility_tree_formatter_utils_mac.mm`、 `browser_accessibility_cocoa.mm`

生成补丁：

```shell
LC_ALL=C TZ=UTC0 diff -Naur content/browser/accessibility content/browser/accessibility-modify > accessibility_mac_12_0_sdk.patch
```

补丁文件`accessibility_mac_12_0_sdk.patch`

```objc
diff -Naur content/browser/accessibility/accessibility_tree_formatter_utils_mac.mm content/browser/accessibility-modify/accessibility_tree_formatter_utils_mac.mm
--- content/browser/accessibility/accessibility_tree_formatter_utils_mac.mm 2021-12-04 09:59:45.000000000 +0000
+++ content/browser/accessibility-modify/accessibility_tree_formatter_utils_mac.mm  2021-12-05 03:39:46.000000000 +0000
@@ -10,12 +10,19 @@
 
 using ui::AXPropertyNode;
 
-extern "C" {
+#if !defined(MAC_OS_VERSION_12_0) || \
+    MAC_OS_X_VERSION_MAX_ALLOWED < MAC_OS_VERSION_12_0
+using AXTextMarkerRangeRef = CFTypeRef;
+using AXTextMarkerRef = CFTypeRef;
+
 
-CFTypeRef AXTextMarkerRangeCopyStartMarker(CFTypeRef);
+extern "C" {
 
-CFTypeRef AXTextMarkerRangeCopyEndMarker(CFTypeRef);
+AXTextMarkerRef AXTextMarkerRangeCopyStartMarker(AXTextMarkerRangeRef);
+AXTextMarkerRef AXTextMarkerRangeCopyEndMarker(AXTextMarkerRangeRef);
 }  // extern "C"
+#endif
+
 
 namespace content {
 namespace a11y {
@@ -261,12 +268,12 @@
     const id target,
     const AXPropertyNode& property_node) const {
   if (property_node.name_or_value == "anchor")
-    return OptionalNSObject(static_cast<id>(
-        AXTextMarkerRangeCopyStartMarker(static_cast<CFTypeRef>(target))));
+    return OptionalNSObject(static_cast<id>(AXTextMarkerRangeCopyStartMarker(
+        static_cast<AXTextMarkerRangeRef>(target))));
 
   if (property_node.name_or_value == "focus")
-    return OptionalNSObject(static_cast<id>(
-        AXTextMarkerRangeCopyEndMarker(static_cast<CFTypeRef>(target))));
+    return OptionalNSObject(static_cast<id>(AXTextMarkerRangeCopyEndMarker(
+        static_cast<AXTextMarkerRangeRef>(target))));
 
   // Unmatched attribute. No error for a tree dump calls because the tree dump
   // sets generic property filters not depending on a node, so we can be called
diff -Naur content/browser/accessibility/browser_accessibility_cocoa.mm content/browser/accessibility-modify/browser_accessibility_cocoa.mm
--- content/browser/accessibility/browser_accessibility_cocoa.mm  2021-12-04 09:59:45.000000000 +0000
+++ content/browser/accessibility-modify/browser_accessibility_cocoa.mm 2021-12-05 03:43:13.000000000 +0000
@@ -40,8 +40,6 @@
 
 #import "ui/accessibility/platform/ax_platform_node_mac.h"
 
-using AXTextMarkerRangeRef = CFTypeRef;
-using AXTextMarkerRef = CFTypeRef;
 using StringAttribute = ax::mojom::StringAttribute;
 using content::AccessibilityMatchPredicate;
 using content::BrowserAccessibility;
@@ -241,33 +239,34 @@
 // VoiceOver uses -1 to mean "no limit" for AXResultsLimit.
 const int kAXResultsLimitNoLimit = -1;
 
-extern "C" {
 
 // The following are private accessibility APIs required for cursor navigation
 // and text selection. VoiceOver started relying on them in Mac OS X 10.11.
+// They are public as of the 12.0 SDK.
+#if !defined(MAC_OS_VERSION_12_0) || \
+    MAC_OS_X_VERSION_MAX_ALLOWED < MAC_OS_VERSION_12_0
+using AXTextMarkerRangeRef = CFTypeRef;
+using AXTextMarkerRef = CFTypeRef;
+extern "C" {
 CFTypeID AXTextMarkerGetTypeID();
 
-AXTextMarkerRef AXTextMarkerCreate(CFAllocatorRef allocator,
+AXTextMarkerRef AXTextMarkerCreate(CFAllocatorRef,
                                    const UInt8* bytes,
                                    CFIndex length);
 
-const UInt8* AXTextMarkerGetBytePtr(AXTextMarkerRef text_marker);
-
-size_t AXTextMarkerGetLength(AXTextMarkerRef text_marker);
+size_t AXTextMarkerGetLength(AXTextMarkerRef);
+const UInt8* AXTextMarkerGetBytePtr(AXTextMarkerRef);
 
 CFTypeID AXTextMarkerRangeGetTypeID();
 
-AXTextMarkerRangeRef AXTextMarkerRangeCreate(CFAllocatorRef allocator,
-                                             AXTextMarkerRef start_marker,
-                                             AXTextMarkerRef end_marker);
-
-AXTextMarkerRef AXTextMarkerRangeCopyStartMarker(
-    AXTextMarkerRangeRef text_marker_range);
-
-AXTextMarkerRef AXTextMarkerRangeCopyEndMarker(
-    AXTextMarkerRangeRef text_marker_range);
+AXTextMarkerRangeRef AXTextMarkerRangeCreate(CFAllocatorRef,
+                                             AXTextMarkerRef start,
+                                             AXTextMarkerRef end);
+AXTextMarkerRef AXTextMarkerRangeCopyStartMarker(AXTextMarkerRangeRef);
+AXTextMarkerRef AXTextMarkerRangeCopyEndMarker(AXTextMarkerRangeRef);
 
 }  // extern "C"
+#endif
 
 // AXTextMarkerCreate is a system function that makes a copy of the data buffer
 // given to it.
@@ -795,7 +794,9 @@
 
 id content::AXTextMarkerRangeFrom(id anchor_textmarker, id focus_textmarker) {
   AXTextMarkerRangeRef cf_marker_range = AXTextMarkerRangeCreate(
-      kCFAllocatorDefault, anchor_textmarker, focus_textmarker);
+      kCFAllocatorDefault, static_cast<AXTextMarkerRef>(anchor_textmarker),
+      static_cast<AXTextMarkerRef>(focus_textmarker));
+
   return [static_cast<id>(cf_marker_range) autorelease];
 }
 
```

打补丁：

```shell
cd ${HOME}/chromium/src
patch -p0 < accessibility_mac_12_0_sdk.patch
```
