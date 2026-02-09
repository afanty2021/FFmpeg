# libavutil - FFmpeg 基础工具库

[根目录](../CLAUDE.md) > **libavutil**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libavutil 是 FFmpeg 的基础工具库，为其他所有库提供核心功能支持。它不依赖任何其他 FFmpeg 库，可独立使用。

### 核心功能
- **内存管理**：安全的内存分配/释放、引用计数
- **数学运算**：整数/浮点运算、CRC、哈希
- **数据结构**：字典、缓冲区、队列、树
- **多媒体基类**：像素格式、样本格式、帧结构
- **工具函数**：日志、错误处理、时间戳、字节序
- **加密与哈希**：AES、DES、MD5、SHA
- **硬件抽象**：硬件设备上下文

## 入口与启动

### 主要头文件
```c
#include <libavutil/avutil.h>      // 核心头文件（包含大部分功能）
#include <libavutil/opt.h>         // 选项系统
#include <libavutil/log.h>         // 日志系统
#include <libavutil/frame.h>       // 帧结构
#include <libavutil/pixfmt.h>      // 像素格式
#include <libavutil/samplefmt.h>   // 音频样本格式
#include <libavutil/timestamp.h>   // 时间戳工具
```

### 初始化
```c
// 设置日志级别
av_log_set_level(AV_LOG_DEBUG);

// 注册硬件设备类型（用于硬件加速）
av_hwdevice_register_all();
```

## 对外接口

### 核心数据结构

#### AVFrame（多媒体帧容器）
```c
AVFrame *frame = av_frame_alloc();
// 使用后释放
av_frame_free(&frame);
```

#### AVDictionary（键值对容器）
```c
AVDictionary *metadata = NULL;
av_dict_set(&metadata, "title", "My Video", 0);
av_dict_set(&metadata, "author", "FFmpeg", 0);
// 使用后释放
av_dict_free(&metadata);
```

#### AVBuffer（引用计数缓冲区）
```c
// 创建缓冲区
AVBufferRef *buf = av_buffer_alloc(1024);
// 创建引用
AVBufferRef *ref = av_buffer_ref(buf);
// 释放
av_buffer_unref(&buf);
av_buffer_unref(&ref);
```

### 主要 API 分类

| 分类 | 头文件 | 功能描述 |
|------|--------|----------|
| **错误处理** | error.h | AVERROR 宏、错误码转换 |
| **日志系统** | log.h | av_log()、日志级别控制 |
| **内存管理** | mem.h | av_malloc()、av_free()、av_strdup() |
| **数学运算** | mathematics.h | 浮点比较、整数运算、舍入 |
| **哈希与校验** | md5.h, sha.h, crc.h | 各种哈希算法实现 |
| **像素格式** | pixfmt.h | AVPixelFormat 定义、转换 |
| **音频格式** | samplefmt.h | 样本格式、布局、平面/交错 |
| **时间工具** | time.h | 时间戳、时间基准转换 |
| **选项系统** | opt.h | AVOption、av_opt_*() 函数 |
| **Base64** | base64.h | av_base64_encode()、decode() |
| **评估表达式** | eval.h | av_expr_eval() 表达式求值 |
| **随机数** | random_seed.h | av_get_random_seed() |

## 关键依赖与配置

### 编译配置（configure）
```bash
# 启用特定组件
--enable-libavutil-xyz
```

### 依赖关系
- **被依赖**：所有其他 FFmpeg 库
- **外部依赖**：无（完全独立）

## 数据模型

### AVFrame 结构（简化）
```c
typedef struct AVFrame {
    uint8_t *data[8];      // 图像/音频数据平面
    int linesize[8];       // 每平面字节数
    int width, height;     // 视频
    int nb_samples;        // 音频
    int format;            // 像素/样本格式
    int64_t pts;           // 显示时间戳
    AVBufferRef *buf[8];   // 引用计数
    AVDictionary *metadata; // 元数据
} AVFrame;
```

### AVPixelFormat（像素格式）
- **RGB 格式**：AV_PIX_FMT_RGB24、AV_PIX_FMT_RGBA
- **YUV 格式**：AV_PIX_FMT_YUV420P、AV_PIX_FMT_YUV422P
- **平面/打包**：P 后缀表示平面格式
- **硬件格式**：AV_PIX_FMT_CUDA、AV_PIX_FMT_VAAPI

### AVSampleFormat（音频格式）
- **打包格式**：AV_SAMPLE_FMT_S16、AV_SAMPLE_FMT_FLT
- **平面格式**：AV_SAMPLE_FMT_S16P、AV_SAMPLE_FMT_FLTP（P 后缀）
- **平面/转换**：av_sample_fmt_is_planar()、av_get_packed_sample_fmt()

## 测试与质量

### 测试文件
- **位置**：`tests/fate/libavutil.mak`
- **示例测试**：CRC、哈希、数学函数、像素格式转换

### 常见问题（FAQ）

#### Q: 如何安全地分配内存？
A: 始终使用 `av_malloc()`/`av_free()` 而非标准库：
```c
void *ptr = av_malloc(1024);
if (!ptr) {
    // 处理 OOM
    return AVERROR(ENOMEM);
}
// 使用...
av_free(ptr);
```

#### Q: AVFrame 的引用计数如何工作？
A: 使用 `av_frame_ref()` 增加引用，`av_frame_unref()` 减少：
```c
AVFrame *src = av_frame_alloc();
AVFrame *dst = av_frame_alloc();
av_frame_ref(dst, src);  // dst 引用 src 的数据
av_frame_unref(dst);     // 释放 dst 的引用
av_frame_free(&src);
av_frame_free(&dst);
```

#### Q: 如何处理时间戳？
A: 使用时间基（time_base）进行转换：
```c
// 从毫秒转换为 AV_TIME_BASE_Q
int64_t ts = av_rescale_q(timestamp_msec, AV_TIME_BASE_Q, stream->time_base);
```

#### Q: 日志级别如何控制？
A:
```c
// 设置全局日志级别
av_log_set_level(AV_LOG_WARNING);

// 为特定上下文设置日志
av_log(ctx, AV_LOG_DEBUG, "Debug message: %s\n", details);
```

## 相关文件清单

### 头文件（公共 API）
```
libavutil/avutil.h          # 主入口
libavutil/common.h          # 内部宏（编译时）
libavutil/error.h           # 错误处理
libavutil/log.h             # 日志系统
libavutil/opt.h             # 选项系统
libavutil/mem.h             # 内存管理
libavutil/frame.h           # AVFrame
libavutil/rational.h        # 有理数
libavutil/pixfmt.h          # 像素格式
libavutil/samplefmt.h       # 样本格式
libavutil/buffer.h          # 引用计数缓冲区
libavutil/dict.h            # 字典
libavutil/channel_layout.h  # 音频声道布局
libavutil/hwcontext.h       # 硬件上下文
```

### 源文件（内部实现）
```
libavutil/utils.c           # 工具函数
libavutil/mem.c             # 内存管理
libavutil/frame.c           # 帧操作
libavutil/opt.c             # 选项系统
libavutil/log.c             # 日志实现
libavutil/pixdesc.c         # 像素格式描述
libavutil/samplefmt.c       # 样本格式转换
libavutil/hash.c            # 哈希算法
libavutil/md5.c             # MD5
libavutil/sha.c             # SHA
libavutil/aes.c             # AES
libavutil/bprint.c          # 动态字符串
libavutil/eval.c            # 表达式求值
```

### 平台优化
```
libavutil/x86/              # x86 SIMD 优化
libavutil/arm/              # ARM NEON 优化
libavutil/aarch64/          # ARM64 优化
libavutil/ppc/              # PowerPC AltiVec
libavutil/riscv/            # RISC-V 优化
libavutil/mips/             # MIPS 优化
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libavutil 模块文档
- 📊 整理核心数据结构和 API
- 🔧 记录常见使用模式
- ✨ 添加 FAQ 和示例代码

---

*此模块是 FFmpeg 的基础，所有其他库都依赖它。*
