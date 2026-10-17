# AudioShare Android 16+ / 16 KB 对齐 APK 构建说明

基于 https://github.com/p88oo/AudioShare 现有资源，构建面向 **Android 16（API 36）**、通过 **16 KB 页面大小对齐检查**的 APK。

## 1. 结论

已产出并通过验证的 APK（release 签名，sign.jks，O=picapico）：

- 产物：`AudioShare_Android16_16KB-release.apk`（约 7.9 MB），已发布为 **Release v0.2**（tag `v0.2`，指向 master）
- 通过两项官方 16 KB 检查（参照 https://developer.android.com/16kb-page-size）：
  1. **ELF 段对齐**：`llvm-objdump -p lib/arm64-v8a/libflacJNI.so | grep LOAD` → 所有 LOAD 段 `align 2**14`（16 KB），且 RELRO 段 `(vaddr+memsz) % 0x4000 == 0`
  2. **ZIP 对齐**：`zipalign -v -c -P 16 4 app-release.apk` → `Verification successful`

## 2. 为什么需要改动

原仓库 APK 里唯一的原生库 `libflacJNI.so`（来自 `com.github.HeHang0:exoplayer-extension-flac:1.3.1`，libFLAC 的 JNI 封装）是用旧 NDK 预编译的，其 ELF LOAD 段对齐为 **2\*\*12（4 KB）**，**不满足 16 KB**。官方要求：预编译库须以 `-Wl,-z,max-page-size=16384 -Wl,-z,common-page-size=16384` 重新编译（或使用 NDK r28+，默认 16 KB 对齐）。

## 3. 改动清单（已合并进 master，commit `5ac7eb8`）

| 位置 | 改动 |
|---|---|
| `android/build.gradle` | AGP **8.5.0 → 8.5.1**（自动 16 KB ZIP 对齐的最低版本） |
| `android/app/build.gradle` | 依赖 `com.github.HeHang0:exoplayer-extension-flac:1.3.1` → 本地模块 `project(':flac')` |
| `android/settings.gradle` | 新增 `include ':flac'` |
| `android/flac/`（新增） | HeHang0 FLAC 扩展模块（Java 封装 + 原生库），其中 `libflacJNI.so` 替换为用 NDK r28 重编译的 **16 KB 对齐**版本；`build.gradle` 移除 maven-publish、`compileSdk` 改 36 |

保持不变：`compileSdk/targetSdk 36`、`minSdk 21`、`useLegacyPackaging false`（原生库不压缩，16 KB ZIP 对齐的前提）、release 签名 sign.jks。

## 4. libflacJNI.so 重编译方法（NDK r28，默认 16 KB ELF 对齐）

- 源码：androidx/media `libraries/decoder_flac/src/main/jni`（flac_jni.cc、flac_parser.cc、flac_sources.mk、Android.mk、Application.mk、include/）+ **libFLAC 1.3.2**
- 构建：`ndk-build NDK_PROJECT_PATH=<dir> APP_ABI="arm64-v8a armeabi-v7a" APP_PLATFORM=android-21`
- `Android.mk` 增加：`LOCAL_LDFLAGS += -Wl,-z,max-page-size=16384 -Wl,-z,common-page-size=16384`
- 产出两个 ABI 的 16 KB 对齐 `libflacJNI.so`，替换模块内同名文件。

## 5. 复现

- **本地**：SDK build-tools 36 + platform android-36 + NDK r28 + JDK 17 + Gradle 8.7；在 `android/` 下 `gradle :app:assembleRelease`。
- **GitHub Actions**：`p88oo/AudioShare-16k-build` 仓库的 workflow（`.github/workflows/build-16kb.yml`）在 Ubuntu runner 上自动完成“拉取源码 → 重编译 16 KB libflacJNI.so → 组装模块 → 构建 APK → 官方验证 → 上传 artifact”，无需任何密钥。

## 6. 验证记录（实测）

```
$ readelf -lW lib/arm64-v8a/libflacJNI.so | grep LOAD
  LOAD 0x000000 vaddr 0x0 align 0x4000   ← 16 KB
  LOAD 0x06e050 vaddr 0x72050 align 0x4000
  LOAD 0x072b98 vaddr 0x7ab98 align 0x4000
$ zipalign -v -c -P 16 4 app-release.apk
  ... Verification successful
```

注：4 KB 页设备上也可正常安装运行（16 KB 对齐向下兼容 4 KB）。
