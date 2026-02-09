# FFmpeg 硬件加速框架

[根目录](./CLAUDE.md) > **硬件加速框架**

> 最后更新：2026-01-17 15:30:00

## 概述

FFmpeg 硬件加速框架提供统一的硬件抽象层，支持跨平台的 GPU 视频编解码、处理和转码操作。该框架通过模块化设计，支持多种硬件加速后端，包括 CUDA、VAAPI、QSV、VideoToolbox、D3D11VA 等。

### 核心特性
- **统一抽象**：跨平台、跨厂商的统一 API
- **零拷贝**：尽可能减少 GPU ↔ CPU 数据传输
- **灵活派生**：支持从现有设备派生新设备类型
- **可扩展**：易于添加新的硬件后端

## 支持的硬件平台

### 平台支持矩阵

| 平台 | 设备类型 | 解码 | 编码 | 滤镜 | 操作系统 | 厂商 |
|------|---------|------|------|------|---------|------|
| **CUDA** | `AV_HWDEVICE_TYPE_CUDA` | ✅ | ✅ (NVENC) | ✅ | Linux/Windows | NVIDIA |
| **VAAPI** | `AV_HWDEVICE_TYPE_VAAPI` | ✅ | ✅ | ✅ | Linux | Intel/AMD |
| **QSV** | `AV_HWDEVICE_TYPE_QSV` | ✅ | ✅ | ❌ | Linux/Windows | Intel |
| **VideoToolbox** | `AV_HWDEVICE_TYPE_VIDEOTOOLBOX` | ✅ | ✅ | ❌ | macOS | Apple |
| **D3D11VA** | `AV_HWDEVICE_TYPE_D3D11VA` | ✅ | ✅ | ✅ | Windows | Microsoft/AMD/NVIDIA |
| **D3D12VA** | `AV_HWDEVICE_TYPE_D3D12VA` | ✅ | ✅ | ❌ | Windows | Microsoft |
| **Vulkan** | `AV_HWDEVICE_TYPE_VULKAN` | ✅ | ❌ | ✅ | Linux/Windows/Android | Khronos |
| **OpenCL** | `AV_HWDEVICE_TYPE_OPENCL` | ❌ | ❌ | ✅ | Linux/Windows/macOS | Khronos |
| **DRM** | `AV_HWDEVICE_TYPE_DRM` | ✅ | ❌ | ✅ | Linux | Kernel |
| **VDPAU** | `AV_HWDEVICE_TYPE_VDPAU` | ✅ | ❌ | ❌ | Linux | NVIDIA |
| **DXVA2** | `AV_HWDEVICE_TYPE_DXVA2` | ✅ | ❌ | ❌ | Windows | Microsoft |
| **MediaCodec** | `AV_HWDEVICE_TYPE_MEDIACODEC` | ✅ | ✅ | ❌ | Android | Google |
| **AMF** | `AV_HWDEVICE_TYPE_AMF` | ✅ | ✅ | ❌ | Windows | AMD |

### 支持的编解码器

#### 解码器硬件加速
```c
// H.264/AVC
h264_cuvid       // CUDA (NVDEC)
h264_vaapi       // VAAPI
h264_qsv         // Intel QSV
h264_videotoolbox // Apple VideoToolbox
h264_d3d11va     // DirectX 11
h264_d3d12va     // DirectX 12

// H.265/HEVC
hevc_cuvid
hevc_vaapi
hevc_qsv
hevc_videotoolbox
hevc_d3d11va
hevc_d3d12va

// VP8/VP9
vp8_vaapi, vp8_cuvid
vp9_vaapi, vp9_cuvid
vp9_qsv, vp9_videotoolbox

// AV1
av1_cuvid
av1_vaapi
av1_qsv
av1_videotoolbox
av1_d3d11va

// 其他
mpeg2_vaapi, mpeg2_cuvid
mpeg4_vaapi, mpeg4_cuvid
vc1_vaapi, vc1_cuvid
wmv3_vaapi, wmv3_cuvid
```

#### 编码器硬件加速
```c
// NVIDIA NVENC
h264_nvenc       // H.264
hevc_nvenc       // H.265
av1_nvenc        // AV1 (较新显卡)

// Intel QSV
h264_qsv         // H.264
hevc_qsv         // H.265
av1_qsv          // AV1 (11代+)
mpeg2_qsv        // MPEG-2
vp9_qsv          // VP9
jpeg_qsv         // JPEG

// AMD AMF
h264_amf         // H.264
hevc_amf         // H.265

// VAAPI
h264_vaapi       // H.264
hevc_vaapi       // H.265
mpeg2_vaapi      // MPEG-2
vp8_vaapi        // VP8
vp9_vaapi        // VP9

// Apple VideoToolbox
h264_videotoolbox
hevc_videotoolbox
```

## 架构设计

### 三层架构

```
┌─────────────────────────────────────────────────────┐
│          应用层 (libavformat/libavfilter)           │
├─────────────────────────────────────────────────────┤
│              硬件抽象层 (libavutil/hwcontext)        │
│  - AVHWDeviceContext (设备抽象)                      │
│  - AVHWFramesContext (帧池抽象)                     │
│  - 数据传输和映射 (transfer/map)                    │
├─────────────────────────────────────────────────────┤
│          硬件后端层 (hwcontext_*.c)                  │
│  - CUDA, VAAPI, QSV, VideoToolbox, D3D11VA...      │
│  - 平台特定的设备管理和内存分配                      │
└─────────────────────────────────────────────────────┘
```

### 核心数据结构

#### AVHWDeviceContext（设备上下文）
```c
typedef struct AVHWDeviceContext {
    const AVClass *av_class;        // 日志类
    enum AVHWDeviceType type;       // 设备类型
    void *hwctx;                    // 设备特定上下文
    void (*free)(struct AVHWDeviceContext *ctx);  // 释放回调
    void *user_opaque;              // 用户数据
} AVHWDeviceContext;
```

**职责**：
- 管理硬件设备（GPU、编码器等）的生命周期
- 提供设备级别的配置和初始化
- 被多个 AVHWFramesContext 共享

#### AVHWFramesContext（帧池上下文）
```c
typedef struct AVHWFramesContext {
    const AVClass *av_class;
    AVBufferRef *device_ref;        // 设备引用
    AVHWDeviceContext *device_ctx;  // 设备指针
    void *hwctx;                    // 帧池特定上下文

    AVBufferPool *pool;             // 缓冲池
    int initial_pool_size;          // 初始池大小

    enum AVPixelFormat format;      // 硬件像素格式
    enum AVPixelFormat sw_format;   // 软件像素格式
    int width, height;              // 帧尺寸
} AVHWFramesContext;
```

**职责**：
- 管理硬件帧的内存池
- 定义帧的格式和尺寸
- 为 AVFrame 分配硬件内存

#### HWContextType（后端类型定义）
```c
typedef struct HWContextType {
    enum AVHWDeviceType type;       // 设备类型
    const char *name;               // 类型名称
    const enum AVPixelFormat *pix_fmts;  // 支持的格式

    size_t device_hwctx_size;       // 设备上下文大小
    size_t frames_hwctx_size;       // 帧上下文大小

    // 设备操作
    int (*device_create)(AVHWDeviceContext *ctx, const char *device,
                         AVDictionary *opts, int flags);
    int (*device_derive)(AVHWDeviceContext *dst_ctx,
                         AVHWDeviceContext *src_ctx,
                         AVDictionary *opts, int flags);
    int (*device_init)(AVHWDeviceContext *ctx);
    void (*device_uninit)(AVHWDeviceContext *ctx);

    // 帧操作
    int (*frames_init)(AVHWFramesContext *ctx);
    void (*frames_uninit)(AVHWFramesContext *ctx);
    int (*frames_get_buffer)(AVHWFramesContext *ctx, AVFrame *frame);
    int (*frames_get_constraints)(AVHWDeviceContext *ctx,
                                   const void *hwconfig,
                                   AVHWFramesConstraints *constraints);

    // 数据传输
    int (*transfer_data_to)(AVHWFramesContext *ctx, AVFrame *dst,
                            const AVFrame *src);
    int (*transfer_data_from)(AVHWFramesContext *ctx, AVFrame *dst,
                              const AVFrame *src);
    int (*transfer_get_formats)(AVHWFramesContext *ctx,
                                enum AVHWFrameTransferDirection dir,
                                enum AVPixelFormat **formats);

    // 映射
    int (*map_to)(AVHWFramesContext *ctx, AVFrame *dst,
                  const AVFrame *src, int flags);
    int (*map_from)(AVHWFramesContext *ctx, AVFrame *dst,
                    const AVFrame *src, int flags);
} HWContextType;
```

## 使用模式

### 1. 基础设备创建

#### 简单创建（推荐用于简单场景）
```c
AVBufferRef *device_ref = NULL;
int ret = av_hwdevice_ctx_create(&device_ref,
                                  AV_HWDEVICE_TYPE_CUDA,
                                  "0",  // 设备名称
                                  NULL,  // 选项
                                  0);    // 标志
if (ret < 0) {
    // 错误处理
}
```

#### 手动创建（更多控制）
```c
// 1. 分配设备上下文
AVBufferRef *device_ref = av_hwdevice_ctx_alloc(AV_HWDEVICE_TYPE_CUDA);
AVHWDeviceContext *device_ctx = (AVHWDeviceContext *)device_ref->data;

// 2. 配置设备特定参数（如果需要）
AVCUDADeviceContext *cuda_ctx = (AVCUDADeviceContext *)device_ctx->hwctx;
// cuda_ctx->cuda_ctx = ...;  // 设置 CUDA 上下文

// 3. 初始化设备
int ret = av_hwdevice_ctx_init(device_ref);
if (ret < 0) {
    av_buffer_unref(&device_ref);
}
```

### 2. 帧池创建与管理

```c
// 1. 创建帧池上下文
AVBufferRef *frames_ref = av_hwframe_ctx_alloc(device_ref);
AVHWFramesContext *frames_ctx = (AVHWFramesContext *)frames_ref->data;

// 2. 配置帧池
frames_ctx->format = AV_PIX_FMT_CUDA;      // 硬件格式
frames_ctx->sw_format = AV_PIX_FMT_NV12;   // 软件格式
frames_ctx->width = 1920;
frames_ctx->height = 1080;
frames_ctx->initial_pool_size = 20;        // 池大小

// 3. 初始化帧池
int ret = av_hwframe_ctx_init(frames_ref);
if (ret < 0) {
    av_buffer_unref(&frames_ref);
}
```

### 3. 硬件解码

```c
// 1. 查找硬件加速解码器
const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);

// 2. 设置硬件设备
codec_ctx->hw_device_ctx = av_buffer_ref(device_ref);

// 3. 打开解码器
avcodec_open2(codec_ctx, codec, NULL);

// 4. 解码循环
AVPacket *pkt = av_packet_alloc();
AVFrame *frame = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    ret = avcodec_send_packet(codec_ctx, pkt);
    if (ret < 0) {
        av_packet_unref(pkt);
        continue;
    }

    while (ret >= 0) {
        ret = avcodec_receive_frame(codec_ctx, frame);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            break;

        // frame->format 可能是 AV_PIX_FMT_CUDA
        if (frame->format == AV_PIX_FMT_CUDA) {
            // 帧在 GPU 内存中
            process_hardware_frame(frame);
        } else {
            // 帧在 CPU 内存中（fallback）
            process_software_frame(frame);
        }

        av_frame_unref(frame);
    }
    av_packet_unref(pkt);
}
```

### 4. 硬件编码

```c
// 1. 查找硬件编码器
const AVCodec *codec = avcodec_find_encoder_by_name("h264_nvenc");
AVCodecContext *codec_ctx = avcodec_alloc_context3(codec);

// 2. 设置参数
codec_ctx->width = 1920;
codec_ctx->height = 1080;
codec_ctx->pix_fmt = AV_PIX_FMT_CUDA;
codec_ctx->time_base = (AVRational){1, 30};
codec_ctx->framerate = (AVRational){30, 1};
codec_ctx->bit_rate = 5000000;

// 3. 设置硬件帧池
AVBufferRef *frames_ref = av_hwframe_ctx_alloc(device_ref);
AVHWFramesContext *frames_ctx = (AVHWFramesContext *)frames_ref->data;
frames_ctx->format = AV_PIX_FMT_CUDA;
frames_ctx->sw_format = AV_PIX_FMT_NV12;
frames_ctx->width = codec_ctx->width;
frames_ctx->height = codec_ctx->height;
av_hwframe_ctx_init(frames_ref);

codec_ctx->hw_frames_ctx = frames_ref;

// 4. 打开编码器
avcodec_open2(codec_ctx, codec, NULL);

// 5. 编码循环
AVFrame *frame = av_frame_alloc();
AVPacket *pkt = av_packet_alloc();

while (get_input_frame(frame) >= 0) {
    // 确保 frame 在硬件内存中
    if (frame->format != AV_PIX_FMT_CUDA) {
        AVFrame *hw_frame = av_frame_alloc();
        av_hwframe_get_buffer(frames_ref, hw_frame, 0);
        av_hwframe_transfer_data(hw_frame, frame, 0);
        av_frame_copy_props(hw_frame, frame);
        av_frame_unref(frame);
        frame = hw_frame;
    }

    ret = avcodec_send_frame(codec_ctx, frame);
    if (ret < 0) {
        av_frame_unref(frame);
        continue;
    }

    while (ret >= 0) {
        ret = avcodec_receive_packet(codec_ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            break;

        // 写入编码数据
        av_interleaved_write_frame(fmt_ctx, pkt);
        av_packet_unref(pkt);
    }

    av_frame_unref(frame);
}
```

### 5. GPU ↔ CPU 数据传输

```c
AVFrame *hw_frame = ...;  // 硬件帧
AVFrame *sw_frame = av_frame_alloc();

// 从 GPU 传输到 CPU
int ret = av_hwframe_transfer_data(sw_frame, hw_frame, 0);
if (ret < 0) {
    // 错误处理
}

// 现在可以访问 sw_frame->data[] 中的 CPU 数据
process_cpu_frame(sw_frame);

av_frame_free(&sw_frame);
```

### 6. 设备派生（Device Derivation）

```c
// 从现有设备派生新设备（零拷贝互操作）
AVBufferRef *cuda_device = ...;
AVBufferRef *vulkan_device = NULL;

// 从 CUDA 设备派生 Vulkan 设备
int ret = av_hwdevice_ctx_create_derived(&vulkan_device,
                                          AV_HWDEVICE_TYPE_VULKAN,
                                          cuda_device,
                                          0);
```

### 7. 硬件滤镜链

```c
// 创建硬件滤镜图
AVFilterGraph *graph = avfilter_graph_alloc();

// 源滤镜（从硬件帧）
AVFilterContext *src;
avfilter_graph_create_filter(&src,
                              avfilter_get_by_name("buffer"),
                              "in",
                              args, NULL, graph);

// 设置硬件帧池
src->hw_device_ctx = av_buffer_ref(device_ref);
av_opt_set_bin(src, "hw_frames_ctx",
                (uint8_t*)&frames_ref, sizeof(frames_ref),
                AV_OPT_SEARCH_CHILDREN);

// 处理滤镜（例如 scale_cuda）
AVFilterContext *scale;
avfilter_graph_create_filter(&scale,
                              avfilter_get_by_name("scale_cuda"),
                              "scale",
                              "w=1280:h=720",
                              NULL, graph);

// 连接滤镜
avfilter_link(src, 0, scale, 0);

// 配置图
avfilter_graph_config(graph, NULL);
```

## 硬件编码器基础架构

### FFHWBaseEncodeContext

FFmpeg 提供了通用的硬件编码器基础架构，被多个硬件编码器共享：

**文件**：
- `libavcodec/hw_base_encode.h` - 头文件
- `libavcodec/hw_base_encode.c` - 基础实现
- `libavcodec/hw_base_encode_h264.c` - H.264 特定
- `libavcodec/hw_base_encode_h265.c` - H.265 特定

**关键功能**：
- GOP 结构管理（I/P/B 帧决策）
- 参考帧管理
- 异步编码支持
- 时间戳处理
- ROI（感兴趣区域）支持

**使用此架构的编码器**：
- VAAPI 编码器（`vaapi_encode_*.c`）
- QSV 编码器（`qsvenc_*.c`）
- 其他平台特定的编码器

## 常见使用场景

### 场景 1：GPU 转码（零拷贝）

```c
// 解码 -> 滤镜 -> 编码，全程 GPU
AVBufferRef *decode_device = ...;  // 解码设备
AVBufferRef *encode_device = ...;  // 编码设备（可能相同）

// 解码
AVCodecContext *decoder_ctx = ...;
decoder_ctx->hw_device_ctx = av_buffer_ref(decode_device);

// 编码
AVCodecContext *encoder_ctx = ...;
encoder_ctx->hw_device_ctx = av_buffer_ref(encode_device);

// 如果设备不同，派生设备
if (decode_device != encode_device) {
    // 可能需要中间格式转换
}
```

### 场景 2：多 GPU 处理

```c
// 创建多个 CUDA 设备
AVBufferRef *gpu0 = NULL;
AVBufferRef *gpu1 = NULL;
av_hwdevice_ctx_create(&gpu0, AV_HWDEVICE_TYPE_CUDA, "0", NULL, 0);
av_hwdevice_ctx_create(&gpu1, AV_HWDEVICE_TYPE_CUDA, "1", NULL, 0);

// 在不同 GPU 上处理不同流
// 流 1 -> GPU 0
// 流 2 -> GPU 1
```

### 场景 3：混合 CPU/GPU 处理

```c
// 部分处理在 GPU，部分在 CPU
AVFrame *hw_frame = ...;

// 只在需要时传输到 CPU
if (needs_cpu_processing) {
    AVFrame *sw_frame = av_frame_alloc();
    av_hwframe_transfer_data(sw_frame, hw_frame, 0);
    cpu_process(sw_frame);

    // 传输回 GPU
    AVFrame *result_hw = av_frame_alloc();
    av_hwframe_get_buffer(frames_ref, result_hw, 0);
    av_hwframe_transfer_data(result_hw, sw_frame, 0);
    av_frame_free(&sw_frame);
    hw_frame = result_hw;
}

// 继续在 GPU 上处理
gpu_process(hw_frame);
```

## 最佳实践

### 1. 错误处理
```c
// 检查硬件加速是否可用
enum AVHWDeviceType type = av_hwdevice_find_type_by_name("cuda");
if (type == AV_HWDEVICE_TYPE_NONE) {
    av_log(NULL, AV_LOG_ERROR, "CUDA not available\n");
    // 降级到软件处理
}

// 检查解码器是否支持硬件加速
const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
if (!codec) {
    // 错误
}

for (int i = 0;; i++) {
    const AVCodecHWConfig *config = avcodec_get_hw_config(codec, i);
    if (!config) {
        av_log(NULL, AV_LOG_ERROR, "Decoder %s does not support device type %s.\n",
               codec->name, av_hwdevice_get_type_name(type));
        break;
    }
    if (config->methods & AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_TYPE &&
        config->device_type == type) {
        // 找到匹配的硬件加速配置
        break;
    }
}
```

### 2. 内存管理
```c
// 总是使用引用计数
AVBufferRef *device_ref = av_hwdevice_ctx_alloc(...);
// 使用...
av_buffer_unref(&device_ref);  // 自动清理

// 帧池会自动管理帧内存
AVFrame *frame = av_frame_alloc();
av_hwframe_get_buffer(frames_ref, frame, 0);
// 使用...
av_frame_free(&frame);  // 释放到池
```

### 3. 格式查询
```c
// 查询支持的格式
AVHWFramesConstraints *constraints =
    av_hwdevice_get_hwframe_constraints(device_ref, NULL);

if (constraints) {
    printf("Supported formats:\n");
    for (int i = 0; constraints->valid_sw_formats[i] != AV_PIX_FMT_NONE; i++) {
        printf("  %s\n",
               av_get_pix_fmt_name(constraints->valid_sw_formats[i]));
    }

    printf("Size range: %dx%d to %dx%d\n",
           constraints->min_width, constraints->min_height,
           constraints->max_width, constraints->max_height);

    av_hwframe_constraints_free(&constraints);
}
```

### 4. 性能优化
```c
// 使用适当的池大小
frames_ctx->initial_pool_size = decoder_delay + async_depth + extra;

// 启用异步编码（如果支持）
av_opt_set_int(codec_ctx->priv_data, "async_depth", 4, 0);

// 避免不必要的数据传输
if (frame->format == hw_pix_fmt) {
    // 直接在 GPU 上处理
} else {
    // 仅在需要时传输
}
```

### 5. 线程安全
```c
// 检查硬件加速器是否支持多线程
// 某些硬件加速器标记为 HWACCEL_CAP_THREAD_SAFE

// 线程间共享设备上下文（只读）
// 不在线程间共享帧池上下文（可变状态）

// 使用独立的设备上下文用于不同线程
AVBufferRef *device_ref_thread[N];
for (int i = 0; i < N; i++) {
    av_hwdevice_ctx_create(&device_ref_thread[i], type, NULL, NULL, 0);
}
```

## 命令行工具使用

### ffmpeg 命令

#### 硬件解码
```bash
# 使用 CUDA 解码
ffmpeg -hwaccel cuda -hwaccel_output_format cuda \
       -i input.mp4 output.mkv

# 使用 VAAPI 解码
ffmpeg -hwaccel vaapi -hwaccel_output_format vaapi \
       -i input.mp4 output.mkv

# 使用 VideoToolbox 解码
ffmpeg -hwaccel videotoolbox -hwaccel_output_format videotoolbox \
       -i input.mp4 output.mkv
```

#### 硬件编码
```bash
# 使用 NVENC 编码
ffmpeg -i input.mp4 -c:v h264_nvenc -preset fast -b:v 5M output.mp4

# 使用 QSV 编码
ffmpeg -i input.mp4 -c:v h264_qsv -preset fast -b:v 5M output.mp4

# 使用 VAAPI 编码
ffmpeg -i input.mp4 -c:v h264_vaapi -b:v 5M output.mp4

# 使用 VideoToolbox 编码
ffmpeg -i input.mp4 -c:v h264_videotoolbox -b:v 5M output.mp4
```

#### 硬件滤镜
```bash
# 使用 CUDA 滤镜
ffmpeg -hwaccel cuda -i input.mp4 \
       -vf scale_cuda=1280:720,hwdownload,format=nv12 \
       output.mp4

# 使用 VAAPI 滤镜
ffmpeg -hwaccel vaapi -hwaccel_output_format vaapi -i input.mp4 \
       -vf 'scale_vaapi:w=1280:h=720' \
       output.mp4
```

#### GPU 转码（零拷贝）
```bash
# 解码和编码都在 GPU 上
ffmpeg -hwaccel cuda -hwaccel_output_format cuda \
       -i input.mp4 -c:v h264_nvenc output.mp4
```

### ffprobe 命令

```bash
# 查询支持的硬件加速方法
ffprobe -hwaccels

# 查询解码器支持
ffmpeg -h decoder=h264_cuvid
```

## 调试和故障排除

### 日志级别
```c
// 启用详细日志
av_log_set_level(AV_LOG_DEBUG);

// 在代码中
av_log(ctx, AV_LOG_VERBOSE, "Hardware device created: %s\n",
       av_hwdevice_get_type_name(device_ctx->type));
```

### 常见问题

#### 问题 1：硬件设备创建失败
```bash
# 检查驱动
nvidia-smi  # NVIDIA
vainfo      # VAAPI

# 检查 ffmpeg 配置
ffmpeg -version | grep hwaccel
```

#### 问题 2：格式不兼容
```bash
# 查询支持的格式
ffmpeg -h decoder=h264_cuvid | grep pix_fmts

# 使用格式转换
ffmpeg -hwaccel cuda -i input.mp4 \
       -vf hwdownload,format=nv12,hwupload_cuda \
       -c:v h264_nvenc output.mp4
```

#### 问题 3：性能不佳
```bash
# 检查是否真的在使用硬件加速
ffmpeg -loglevel debug -hwaccel cuda ...

# 增加异步深度
ffmpeg -c:v h264_nvenc -async_depth 4 ...

# 调整预设
ffmpeg -c:v h264_nvenc -preset fast ...
```

## API 参考

### 核心函数

#### 设备管理
```c
// 查找设备类型
enum AVHWDeviceType av_hwdevice_find_type_by_name(const char *name);
const char *av_hwdevice_get_type_name(enum AVHWDeviceType type);
enum AVHWDeviceType av_hwdevice_iterate_types(enum AVHWDeviceType prev);

// 创建设备
AVBufferRef *av_hwdevice_ctx_alloc(enum AVHWDeviceType type);
int av_hwdevice_ctx_init(AVBufferRef *ref);
int av_hwdevice_ctx_create(AVBufferRef **device_ctx, enum AVHWDeviceType type,
                           const char *device, AVDictionary *opts, int flags);

// 设备派生
int av_hwdevice_ctx_create_derived(AVBufferRef **dst_ctx,
                                   enum AVHWDeviceType type,
                                   AVBufferRef *src_ctx, int flags);
```

#### 帧池管理
```c
// 创建帧池
AVBufferRef *av_hwframe_ctx_alloc(AVBufferRef *device_ctx);
int av_hwframe_ctx_init(AVBufferRef *ref);

// 分配帧
int av_hwframe_get_buffer(AVBufferRef *hwframe_ctx, AVFrame *frame, int flags);

// 帧池派生
int av_hwframe_ctx_create_derived(AVBufferRef **derived_frame_ctx,
                                  enum AVPixelFormat format,
                                  AVBufferRef *derived_device_ctx,
                                  AVBufferRef *source_frame_ctx,
                                  int flags);
```

#### 数据传输
```c
// GPU ↔ CPU 传输
int av_hwframe_transfer_data(AVFrame *dst, const AVFrame *src, int flags);

// 查询支持的传输格式
int av_hwframe_transfer_get_formats(AVBufferRef *hwframe_ctx,
                                    enum AVHWFrameTransferDirection dir,
                                    enum AVPixelFormat **formats, int flags);

// 映射硬件帧
int av_hwframe_map(AVFrame *dst, const AVFrame *src, int flags);
```

#### 约束查询
```c
// 获取设备约束
void *av_hwdevice_hwconfig_alloc(AVBufferRef *device_ctx);
AVHWFramesConstraints *av_hwdevice_get_hwframe_constraints(
    AVBufferRef *ref, const void *hwconfig);
void av_hwframe_constraints_free(AVHWFramesConstraints **constraints);
```

## 平台特定信息

### NVIDIA CUDA

**设备创建**：
```c
// 使用特定 GPU
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_CUDA, "0", NULL, 0);

// 使用所有 GPU（多 GPU）
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_CUDA, NULL, NULL, 0);
```

**支持的格式**：
- NV12、P010、YUV420P、YUV444P
- RGB32、BGRA
- 8/10/16-bit 深度

**NVENC 选项**：
```c
// 预设
av_opt_set(ctx->priv_data, "preset", "p1", 0);  // fastest to p7 (slowest)

// 配置文件
av_opt_set(ctx->priv_data, "profile", "high", 0);

// 码率控制
av_opt_set(ctx->priv_data, "rc", "cbr", 0);  // cbr, vbr, cbr_hq, vbr_hq

// 其他
av_opt_set(ctx->priv_data, "delay", "0", 0);  // 初始延迟
av_opt_set(ctx->priv_data, "tune", "ll", 0);  // 质量/延迟平衡
```

### Intel VAAPI

**设备创建**：
```c
// 使用 DRM
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_VAAPI,
                       "/dev/dri/renderD128", NULL, 0);

// 使用 X11
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_VAAPI,
                       ":0", NULL, 0);
```

**支持的格式**：
- NV12、YUV420P、YUV422P、YUV444P
- P010、P012（10-bit）
- RGB 格式

**VAAPI 编码器选项**：
```c
// 码率控制
av_opt_set(ctx->priv_data, "rc_mode", "cbr", 0);

// 质量
av_opt_set(ctx->priv_data, "quality", "medium", 0);

// GOP
av_opt_set(ctx->priv_data, "g", "30", 0);
```

### Intel QSV

**设备创建**：
```c
// QSV 通常从其他设备派生
AVBufferRef *vaapi_device = ...;
av_hwdevice_ctx_create_derived(&qsv_device, AV_HWDEVICE_TYPE_QSV,
                               vaapi_device, 0);
```

**特点**：
- 同时支持解码和编码
- 可以与 VAAPI 互操作
- 支持转码（解码+编码在一个管道）

### Apple VideoToolbox

**设备创建**：
```c
// VideoToolbox 不需要显式设备创建
// 使用默认设备
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_VIDEOTOOLBOX,
                       NULL, NULL, 0);
```

**支持的格式**：
- NV12、YUV420P、UYVY422
- P010、P210（10-bit）
- BGRA、RGBA

**选项**：
```c
// 实时编码
av_opt_set(ctx->priv_data, "realtime", "true", 0);

// 质量
av_opt_set(ctx->priv_data, "quality", "normal", 0);

// 比特率
av_opt_set(ctx->priv_data, "b", "5000000", 0);
```

### DirectX 11/12

**设备创建**：
```c
// 使用默认适配器
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_D3D11VA,
                       NULL, NULL, 0);

// 使用特定适配器
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_D3D11VA,
                       "0", NULL, 0);  // 适配器索引
```

**特点**：
- Windows 平台主要选择
- 支持 AMD、NVIDIA、Intel GPU
- D3D12 支持更新的功能

### Vulkan

**设备创建**：
```c
// 使用默认设备
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_VULKAN,
                       NULL, NULL, 0);

// 指定设备
av_hwdevice_ctx_create(&device_ref, AV_HWDEVICE_TYPE_VULKAN,
                       "0", NULL, 0);
```

**特点**：
- 跨平台（Linux、Windows、Android）
- 适合计算任务
- 支持与其他 API 互操作

## 性能对比

### 编码器性能（相对）

| 编码器 | 质量 | 速度 | 功耗 | 适用场景 |
|--------|------|------|------|----------|
| **NVENC (P7)** | ★★★★★ | ★★★ | ★★ | 高质量录制 |
| **NVENC (P1)** | ★★★ | ★★★★★ | ★★★ | 直播流 |
| **QSV** | ★★★★ | ★★★★ | ★★★★ | 通用 |
| **VAAPI** | ★★★ | ★★★★ | ★★★★★ | Linux 服务器 |
| **VideoToolbox** | ★★★★ | ★★★★ | ★★★ | macOS 通用 |
| **x264 (slow)** | ★★★★★ | ★ | ★★★★★ | 离线高质量 |
| **软件 AV1** | ★★★★ | ★ | ★ | 未来标准 |

### 解码器性能

大多数硬件解码器性能相似，主要取决于：
- GPU 型号
- 驱动版本
- 视频分辨率和复杂度

**通常**：
- 硬件解码 > 软件解码（高分辨率）
- 软件解码 > 硬件解码（低分辨率，避免传输开销）

## 相关文件清单

### 公共头文件
```
libavutil/hwcontext.h                    # 核心硬件抽象接口
libavutil/hwcontext_vaapi.h               # VAAPI 特定接口
libavutil/hwcontext_qsv.h                 # QSV 特定接口
libavutil/hwcontext_cuda.h                # CUDA 特定接口
libavutil/hwcontext_videotoolbox.h        # VideoToolbox 特定接口
libavutil/hwcontext_d3d11va.h             # D3D11VA 特定接口
libavutil/hwcontext_vulkan.h              # Vulkan 特定接口
libavutil/hwcontext_opencl.h              # OpenCL 特定接口
```

### 内部头文件
```
libavutil/hwcontext_internal.h            # 内部抽象接口
libavcodec/hwaccels.h                     # 硬件加速器声明
libavcodec/hwaccel_internal.h             # 硬件加速器内部结构
libavcodec/hwconfig.h                     # 硬件配置宏
```

### 实现文件
```
# 核心框架
libavutil/hwcontext.c                     # 核心实现

# 硬件后端
libavutil/hwcontext_cuda.c                # CUDA 实现
libavutil/hwcontext_vaapi.c               # VAAPI 实现
libavutil/hwcontext_qsv.c                 # QSV 实现
libavutil/hwcontext_videotoolbox.c        # VideoToolbox 实现
libavutil/hwcontext_d3d11va.c             # D3D11VA 实现
libavutil/hwcontext_d3d12va.c             # D3D12VA 实现
libavutil/hwcontext_vulkan.c              # Vulkan 实现
libavutil/hwcontext_opencl.c              # OpenCL 实现
libavutil/hwcontext_drm.c                 # DRM 实现
libavutil/hwcontext_vdpau.c               # VDPAU 实现
libavutil/hwcontext_dxva2.c               # DXVA2 实现
libavutil/hwcontext_mediacodec.c          # MediaCodec 实现
libavutil/hwcontext_amf.c                 # AMF 实现

# 硬件编码器基础
libavcodec/hw_base_encode.c               # 通用编码器基础
libavcodec/hw_base_encode_h264.c          # H.264 编码基础
libavcodec/hw_base_encode_h265.c          # H.265 编码基础

# 硬件加速器
libavcodec/cuviddec.c                     # CUDA 解码
libavcodec/nvenc.c                        # NVIDIA 编码
libavcodec/vaapi_encode.c                 # VAAPI 编码
libavcodec/vaapi_encode_h264.c
libavcodec/vaapi_encode_h265.c
libavcodec/vaapi_encode_av1.c
libavcodec/qsvenc.c                       # Intel QSV 编码
libavcodec/qsvdec.c                       # Intel QSV 解码
libavcodec/videotoolboxenc.c              # VideoToolbox 编码
```

### 命令行工具
```
ffmpeg.c                                  # 主程序
fftools/ffmpeg_hw.c                       # 硬件加速支持
ffprobe.c                                 # 探测工具
```

## 变更记录 (Changelog)

### 2026-01-17 15:30:00 - 初始版本
- 📝 创建硬件加速框架文档
- 🎯 记录所有支持的硬件平台
- 💡 添加使用模式和最佳实践
- 📊 提供性能对比和故障排除指南

---

*硬件加速是 FFmpeg 的核心优势，正确使用可大幅提升性能。*
