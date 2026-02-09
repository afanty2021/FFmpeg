# libswresample - FFmpeg 音频重采样库

[根目录](../CLAUDE.md) > **libswresample**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libswresample 提供音频重采样、格式转换和混音功能。

### 核心功能
- **重采样**：改变音频采样率（44.1kHz → 48kHz）
- **格式转换**：PCM 格式转换（S16 → FLTP）
- **声道转换**：立体声 → 单声道、5.1 → 7.1
- **混音**：多声道混合、音量调整
- **延迟优化**：最小转换延迟

## 入口与启动

### 主要头文件
```c
#include <libswresample/swresample.h>
```

### 典型使用流程
```c
// 1. 创建重采样上下文
SwrContext *swr_ctx = swr_alloc();

// 2. 设置参数
av_opt_set_int(swr_ctx, "in_channel_layout", AV_CH_LAYOUT_STEREO, 0);
av_opt_set_int(swr_ctx, "out_channel_layout", AV_CH_LAYOUT_5POINT1, 0);
av_opt_set_int(swr_ctx, "in_sample_rate", 44100, 0);
av_opt_set_int(swr_ctx, "out_sample_rate", 48000, 0);
av_opt_set_sample_fmt(swr_ctx, "in_sample_fmt", AV_SAMPLE_FMT_S16, 0);
av_opt_set_sample_fmt(swr_ctx, "out_sample_fmt", AV_SAMPLE_FMT_FLTP, 0);

// 3. 初始化
swr_init(swr_ctx);

// 4. 转换
uint8_t **out_data;
int out_linesize;
int out_samples = swr_get_out_samples(swr_ctx, in_samples);
av_samples_alloc_array_and_samples(&out_data, &out_linesize,
    out_channels, out_samples, out_sample_fmt, 0);
swr_convert(swr_ctx, out_data, out_samples,
    (const uint8_t **)in_data, in_samples);

// 5. 清理
av_freep(&out_data[0]);
av_freep(&out_data);
swr_free(&swr_ctx);
```

## 对外接口

### 核心 API

#### 上下文管理
```c
// 创建和释放
SwrContext *swr_alloc(void);
SwrContext *swr_alloc_set_opts(struct SwrContext *s,
    int64_t out_ch_layout, enum AVSampleFormat out_sample_fmt, int out_sample_rate,
    int64_t in_ch_layout,  enum AVSampleFormat in_sample_fmt,  int in_sample_rate,
    int log_offset, void *log_ctx);
void swr_free(SwrContext **s);

// 初始化和配置
int swr_init(SwrContext *s);
int swr_config_frame(SwrContext *s, const AVFrame *out, const AVFrame *in);
```

#### 转换函数
```c
// 基本转换
int swr_convert(SwrContext *s, uint8_t **out, int out_count,
    const uint8_t **in, int in_count);

// 帧转换（推荐）
int swr_convert_frame(SwrContext *s, AVFrame *out, const AVFrame *in);
```

#### 查询函数
```c
// 获取输出样本数
int swr_get_out_samples(SwrContext *s, int in_samples);

// 获取延迟
int64_t swr_get_delay(SwrContext *s, int64_t base);

// 设置补偿
int swr_set_compensation(SwrContext *s, int sample_delta, int compensation_distance);
```

### 常见转换场景

#### 重采样（改变采样率）
```c
// 44.1kHz → 48kHz
SwrContext *swr = swr_alloc_set_opts(NULL,
    AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 44100,
    0, NULL);
swr_init(swr);
```

#### 格式转换（改变位深）
```c
// S16 → FLTP（平面浮点）
SwrContext *swr = swr_alloc_set_opts(NULL,
    AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 44100,
    AV_CH_LAYOUT_STEREO, AV_SAMPLE_FMT_S16,  44100,
    0, NULL);
swr_init(swr);
```

#### 声道转换
```c
// 立体声 → 5.1
SwrContext *swr = swr_alloc_set_opts(NULL,
    AV_CH_LAYOUT_5POINT1, AV_SAMPLE_FMT_FLTP, 48000,
    AV_CH_LAYOUT_STEREO,  AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr);

// 自动混合矩阵（默认）
// 或自定义混音矩阵
```

#### 综合转换
```c
// 立体声 44.1kHz S16 → 5.1 48kHz FLTP
SwrContext *swr = swr_alloc_set_opts(NULL,
    AV_CH_LAYOUT_5POINT1_BACK, AV_SAMPLE_FMT_FLTP, 48000,
    AV_CH_LAYOUT_STEREO,       AV_SAMPLE_FMT_S16,  44100,
    0, NULL);
swr_init(swr);
```

## 关键依赖与配置

### 编译配置
```bash
# 启用 libswresample
--enable-swr

# 启用优化
--enable-swscale-x86  # x86 SIMD

# 禁用（使用 libsoxr 替代）
--disable-swr --enable-libsoxr
```

### 依赖关系
- **依赖**：libavutil
- **可选依赖**：
  - libsoxr：高质量重采样替代
  - libavutil 的 DSP 优化

## 数据模型

### 声道布局
```c
// 常见布局
AV_CH_LAYOUT_MONO              // 单声道
AV_CH_LAYOUT_STEREO            // 立体声
AV_CH_LAYOUT_2POINT1           // 2.1
AV_CH_LAYOUT_SURROUND          // 4.0
AV_CH_LAYOUT_5POINT1           // 5.1
AV_CH_LAYOUT_7POINT1           // 7.1

// 获取声道数
int channels = av_get_channel_layout_nb_channels(AV_CH_LAYOUT_5POINT1);
```

### 样本格式
```c
// 打包格式（交错）
AV_SAMPLE_FMT_U8   // uint8
AV_SAMPLE_FMT_S16  // int16
AV_SAMPLE_FMT_S32  // int32
AV_SAMPLE_FMT_FLT  // float

// 平面格式
AV_SAMPLE_FMT_U8P
AV_SAMPLE_FMT_S16P
AV_SAMPLE_FMT_S32P
AV_SAMPLE_FMT_FLTP

// 检查
int is_planar = av_sample_fmt_is_planar(AV_SAMPLE_FMT_FLTP);
```

### 混音矩阵
```c
// 自定义混音矩阵
double matrix[6][2];  // [out_ch][in_ch]
// 设置矩阵值...
swr_set_matrix(swr, (const double *)matrix, 6);
```

## 性能优化

### 质量设置
```c
// 设置重采样质量（0-15）
av_opt_set_int(swr_ctx, "resample_quality", 5, 0);
// 0 = 最快
// 5 = 默认
// 10 = 高质量
// 15 = 最佳质量

// 设置滤波器大小
av_opt_set_int(swr_ctx, "filter_size", 32, 0);  // 默认 16
```

### 最小化延迟
```c
// 使用紧凑模式
av_opt_set_int(swr_ctx, "compact", 1, 0);

// 最小化缓冲
av_opt_set_int(swr_ctx, "min_comp", 1, 0);
```

### SIMD 优化
```c
// 自动启用（如果可用）
// x86: SSE2/AVX2
// ARM: NEON
// PPC: AltiVec
// RISC-V: RVV
```

## 常见问题（FAQ）

### Q: 如何处理累积延迟？
A:
```c
// 获取当前延迟
int64_t delay = swr_get_delay(swr, in_sample_rate);

// 冲洗缓冲区
swr_convert(swr, out_data, out_samples, NULL, 0);
```

### Q: 如何实现实时重采样？
A:
```c
// 使用固定输出大小
int out_samples = swr_get_out_samples(swr, in_samples);

// 部分转换
while (in_count > 0) {
    int converted = swr_convert(swr, out, out_count, in, in_count);
    in += converted;
    in_count -= converted;
}
```

### Q: 如何创建自定义混音矩阵？
A:
```c
// 立体声 → 5.1 示例
double matrix[6][2] = {
    {1, 0},  // L: L
    {0, 1},  // R: R
    {0.5, 0},  // C: L/2
    {0, 0.5},  // LFE: R/2
    {1, 0},   // RL: L
    {0, 1},   // RR: R
};
swr_set_matrix(swr, (const double *)matrix, 6);
```

### Q: 为什么需要平面格式？
A: SIMD 优化更容易处理独立声道：
```c
// 平面格式（每个声道独立）
float *ch0 = data[0];
float *ch1 = data[1];

// SIMD 处理单个声道
for (int i = 0; i < samples; i += 4) {
    // 处理 ch0[i...i+3]
}
```

## 相关文件清单

### 头文件（公共 API）
```
libswresample/swresample.h   # 主入口
```

### 源文件（内部实现）
```
libswresample/swresample.c   # 核心实现
libswresample/options.c      # AVOption 处理
libswresample/rematrix.c     # 声道混合
libswresample/resample.c     # 重采样
libswresample/resample_dsp.c # DSP 优化
libswresample/audioconvert.c # 格式转换
libswresample/dither.c       # 抖动（Dithering）
```

### 模板和优化
```
libswresample/rematrix_template.c  # 混合模板
libswresample/resample_template.c  # 重采样模板
libswresample/dither_template.c    # 抖动模板
libswresample/soxr_resample.c      # libsoxr 封装
```

### 优化目录
```
libswresample/x86/           # x86 SIMD 优化
libswresample/aarch64/       # ARM64 NEON
libswresample/arm/           # ARM NEON
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libswresample 模块文档
- 🔊 整理音频转换场景
- ⚡ 记录性能优化技巧
- ✨ 添加常见问题解答

---

*此模块是音频处理的核心，所有音频转码都需要它。*
