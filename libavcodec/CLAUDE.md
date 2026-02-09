# libavcodec - FFmpeg 编解码库

[根目录](../CLAUDE.md) > **libavcodec**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libavcodec 是 FFmpeg 的核心编解码库，提供最广泛的音视频编解码器实现（1000+ 编解码器）。

### 核心功能
- **解码器**：H.264、H.265、VP9、AV1、AAC、MP3 等
- **编码器**：libx264、libx265、libvpx、AAC、MP3 等
- **解析器**：将字节流分割为帧
- **比特流过滤器**：修改编码数据而不重新解码
- **硬件加速**：CUDA、VAAPI、QSV、D3D11VA 等
- **DSP 优化**：SIMD 加速的编解码核心

## 入口与启动

### 主要头文件
```c
#include <libavcodec/avcodec.h>    // 核心编解码 API
#include <libavcodec/codec.h>      // AVCodec 定义
#include <libavcodec/codec_id.h>   // 编解码器 ID 枚举
#include <libavcodec/codec_par.h>  // 编解码器参数
#include <libavcodec/packet.h>     // AVPacket
```

### 典型使用流程（解码）
```c
// 1. 查找解码器
const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);

// 2. 分配上下文
AVCodecContext *ctx = avcodec_alloc_context3(codec);

// 3. 设置参数（通常从 demuxer 复制）
avcodec_parameters_to_context(ctx, stream->codecpar);

// 4. 打开解码器
avcodec_open2(ctx, codec, NULL);

// 5. 发送包、接收帧
avcodec_send_packet(ctx, pkt);
avcodec_receive_frame(ctx, frame);

// 6. 清理
avcodec_free_context(&ctx);
```

## 对外接口

### 核心 API

#### 编解码器查找
```c
// 通过 ID 查找
const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
const AVCodec *encoder = avcodec_find_encoder(AV_CODEC_ID_AAC);

// 通过名称查找
const AVCodec *codec = avcodec_find_decoder_by_name("h264");
```

#### 发送/接收 API（推荐方式）
```c
// 解码
int ret = avcodec_send_packet(codec_ctx, pkt);  // 送入压缩数据
while (ret >= 0) {
    ret = avcodec_receive_frame(codec_ctx, frame);  // 取出解码帧
    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        break;
    // 处理 frame
}

// 编码
int ret = avcodec_send_frame(codec_ctx, frame);  // 送入原始数据
while (ret >= 0) {
    ret = avcodec_receive_packet(codec_ctx, pkt);  // 取出压缩包
    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        break;
    // 处理 pkt
}
```

#### 硬件加速
```c
// 创建设备
AVBufferRef *hw_device_ctx = NULL;
av_hwdevice_ctx_create(&hw_device_ctx, AV_HWDEVICE_TYPE_CUDA, NULL, NULL, 0);

// 设置到解码器
codec_ctx->hw_device_ctx = av_buffer_ref(hw_device_ctx);
avcodec_open2(codec_ctx, codec, NULL);

// 帧可能位于 GPU 内存
if (frame->format == AV_PIX_FMT_CUDA) {
    // 将 GPU 帧传输到 CPU
    av_hwframe_transfer_data(sw_frame, frame, 0);
}
```

### 主要数据结构

#### AVCodecContext（编解码上下文）
```c
typedef struct AVCodecContext {
    const AVCodec *codec;      // 编解码器
    int width, height;         // 视频
    int sample_rate, channels; // 音频
    enum AVPixelFormat pix_fmt;      // 视频像素格式
    enum AVSampleFormat sample_fmt;  // 音频样本格式
    uint64_t channel_layout;   // 声道布局
    AVRational time_base;      // 时间基
    int bit_rate;              // 比特率
    int flags;                 // 标志（低延迟、快速等）
    AVBufferRef *hw_device_ctx;  // 硬件设备
    AVBufferRef *hw_frames_ctx;  // 硬件帧池
    void *priv_data;           // 私有数据
} AVCodecContext;
```

#### AVPacket（压缩数据包）
```c
AVPacket *pkt = av_packet_alloc();
pkt->data = ...;  // 压缩数据
pkt->size = ...;  // 数据大小
pkt->pts = ...;   // 显示时间戳
pkt->dts = ...;   // 解码时间戳
av_packet_unref(pkt);  // 释放引用
av_packet_free(&pkt);
```

#### AVCodecParameters（流参数）
```c
// 来自 demuxer，用于初始化解码器
avcodec_parameters_to_context(codec_ctx, stream->codecpar);

// 更新参数到 muxer
avcodec_parameters_from_context(stream->codecpar, codec_ctx);
```

### 比特流过滤器
```c
// 初始化 BSF
const AVBitStreamFilter *bsf = av_bsf_getByName("h264_mp4toannexb");
AVBSFContext *bsf_ctx;
av_bsf_alloc(bsf, &bsf_ctx);
avcodec_parameters_copy(bsf_ctx->par_in, stream->codecpar);
av_bsf_init(bsf_ctx);

// 过滤包
av_bsf_send_packet(bsf_ctx, pkt);
av_bsf_receive_packet(bsf_ctx, filt_pkt);
```

## 关键依赖与配置

### 编译配置
```bash
# 启用所有解码器
--enable-decoders

# 启用特定编码器
--enable-libx264 --enable-libx265 --enable-libvpx

# 启用硬件加速
--enable-cuda --enable-cuvid --enable-nvenc
--enable-vaapi --enable-vdpau
--enable-libmfx --enable-qsv

# 启用/禁用解析器
--enable-parsers
```

### 依赖关系
- **依赖**：libavutil
- **可选依赖**：
  - 视频编码：libx264、libx265、libvpx、libaom
  - 音频编码：libmp3lame、libopus、libvorbis
  - 硬件加速：CUDA SDK、Intel Media SDK、Vulkan

## 数据模型

### 编解码器类型
```c
// 视频编解码器
AV_CODEC_ID_H264、AV_CODEC_ID_HEVC、AV_CODEC_ID_VP9
AV_CODEC_ID_AV1、AV_CODEC_ID_MPEG2VIDEO、AV_CODEC_ID_MJPEG

// 音频编解码器
AV_CODEC_ID_AAC、AV_CODEC_ID_MP3、AV_CODEC_ID_OPUS
AV_CODEC_ID_VORBIS、AV_CODEC_ID_FLAC、AV_CODEC_ID_AC3

// 字幕编解码器
AV_CODEC_ID_SUBRIP、AV_CODEC_ID_ASS、AV_CODEC_ID_WEBVTT
```

### 解码器标志
```c
// 低延迟（减少缓冲）
codec_ctx->flags |= AV_CODEC_FLAG_LOW_DELAY;

// 快速（质量换速度）
codec_ctx->flags2 |= AV_CODEC_FLAG2_FAST;

// 输出截断（允许不完整的帧）
codec_ctx->flags |= AV_CODEC_FLAG_TRUNCATED;
```

### 编码器选项
```c
// 通过 AVOption 设置
av_opt_set(codec_ctx->priv_data, "preset", "slow", 0);
av_opt_set(codec_ctx->priv_data, "crf", "23", 0);

// 常见选项：
// - preset: ultrafast, superfast, veryfast, faster, fast, medium, slow, slower, veryslow
// - crf: 恒定质量因子（0-51，越低越好）
// - bitrate: 目标比特率
// - g: GOP 大小
// - threads: 线程数
```

## 测试与质量

### 测试文件
- **位置**：`tests/fate/*.mak`（按编解码器分类）
- **示例**：`h264.mak`、`hevc.mak`、`aac.mak`

### FATE 测试覆盖
- **编码测试**：生成编码数据并验证
- **解码测试**：验证解码输出
- **回归测试**：参考文件哈希比对
- **Fuzz 测试**：`tools/target_dec_fuzzer.c`

### checkasm 测试
- **位置**：`tests/checkasm/`
- **覆盖**：DSP 函数、IDCT、运动补偿等

### 常见问题（FAQ）

#### Q: 如何处理多线程解码？
A: 解码器自动使用多线程（如果支持）：
```c
// 设置线程数（0 = 自动）
codec_ctx->thread_count = 0;
codec_ctx->thread_type = FF_THREAD_FRAME | FF_THREAD_SLICE;
```

#### Q: EAGAIN 错误是什么意思？
A: 需要更多输入数据：
```c
while ((ret = avcodec_send_packet(ctx, pkt)) == AVERROR(EAGAIN)) {
    // 调用 avcodec_receive_frame() 消耗数据
    avcodec_receive_frame(ctx, frame);
}
```

#### Q: 如何实现硬件加速解码？
A:
```c
// 1. 创建设备
AVBufferRef *hw_ctx = NULL;
av_hwdevice_ctx_create(&hw_ctx, AV_HWDEVICE_TYPE_CUDA, "0", NULL, 0);

// 2. 设置到解码器
codec_ctx->hw_device_ctx = av_buffer_ref(hw_ctx);

// 3. 可能需要软件格式回退
if (frame->format == hw_pix_fmt) {
    av_hwframe_transfer_data(sw_frame, frame, 0);
}
```

#### Q: 如何提取运动矢量？
A: 使用导出侧信息：
```c
// 启用运动矢量导出
codec_ctx->export_side_data |= AV_CODEC_EXPORT_DATA_MVS;

// 从帧中读取
AVFrameSideData *sd = av_frame_get_side_data(frame, AV_FRAME_DATA_MOTION_VECTORS);
if (sd) {
    AVMotionVector *mvs = (AVMotionVector *)sd->data;
    int nb_mvs = sd->size / sizeof(AVMotionVector);
}
```

## 相关文件清单

### 头文件（公共 API）
```
libavcodec/avcodec.h         # 主入口
libavcodec/codec.h           # AVCodec、AVCodecContext
libavcodec/codec_id.h        # AVCodecID 枚举
libavcodec/codec_par.h       # AVCodecParameters
libavcodec/packet.h          # AVPacket
libavcodec/bsf.h             # 比特流过滤器
libavcodec/codec_desc.h      # 编解码器描述
```

### 源文件（编解码器实现）
```
libavcodec/allcodecs.c       # 编解码器注册
libavcodec/utils.c           # 通用工具
libavcodec/options.c         # AVOption 处理

# 解码器
libavcodec/h264dec.c         # H.264 解码器
libavcodec/hevcdec.c         # H.265 解码器
libavcodec/vp9dec.c          # VP9 解码器
libavcodec/av1dec.c          # AV1 解码器
libavcodec/aacdec.c          # AAC 解码器

# 编码器
libavcodec/libx264.c         # x264 封装
libavcodec/libx265.c         # x265 封装
libavcodec/libvpxenc.c       # libvpx 封装
libavcodec/aacenc.c          # AAC 编码器

# 硬件加速
libavcodec/cuviddec.c        # CUDA 解码
libavcodec/nvenc.c           # NVIDIA 编码
libavcodec/vaapi_encode.c    # VAAPI 编码
libavcodec/qsv.c             # Intel QSV
```

### 优化目录
```
libavcodec/x86/              # x86 SIMD (SSE2/AVX2)
libavcodec/aarch64/          # ARM64 NEON
libavcodec/arm/              # ARM NEON
libavcodec/ppc/              # PowerPC AltiVec
libavcodec/mips/             # MIPS DSP
libavcodec/riscv/            # RISC-V RVV
```

### 解析器
```
libavcodec/parsers.c         # 解析器注册
libavcodec/h264_parser.c     # H.264 解析器
libavcodec/aac_parser.c      # AAC 解析器
```

### 比特流过滤器
```
libavcodec/bitstream_filters.c
libavcodec/h264_mp4toannexb_bsf.c
libavcodec/trace_headers_bsf.c
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libavcodec 模块文档
- 🎬 整理解码/编码工作流程
- ⚡ 记录硬件加速接口
- 🔧 添加常见问题解答

---

*此模块包含 FFmpeg 最大的代码量，1000+ 编解码器实现。*
