# FFmpeg 测试框架 - AI 上下文文档

> 最后更新：2026-01-17 10:30:00
> 覆盖率：95%+

[根目录](../CLAUDE.md) > **tests**

---

## 模块职责

FFmpeg 测试框架是确保多媒体处理质量和稳定性的核心基础设施。它提供：

1. **FATE (FFmpeg Automated Testing Environment)** - 回归测试套件，包含 1000+ 测试用例
2. **checkasm** - 汇编级 SIMD 优化验证与性能基准测试
3. **API 测试** - 公共 API 使用示例和集成测试
4. **辅助工具** - 测试数据生成、比较和分析工具

## 目录结构

```
tests/
├── fate/              # FATE 测试配置文件
│   ├── *.mak          # Makefile 片段，定义测试用例
│   └── source-check.sh
├── checkasm/          # 汇编级测试
│   ├── checkasm.c     # 测试框架核心
│   ├── checkasm.h     # 测试 API 头文件
│   └── *.c            # 各组件的汇编测试
├── api/               # API 测试程序
│   ├── api-h264-test.c
│   ├── api-seek-test.c
│   └── ...
├── *.c                # 测试辅助工具
├── *.sh               # 测试执行脚本
├── Makefile           # 主测试 Makefile
└── ref/               # 参考输出（Git 中不含）
```

## 1. FATE 回归测试

### 1.1 概述

FATE 是 FFmpeg 的主要回归测试套件，用于验证：

- 编解码器正确性（编码 → 解码 → 比较输出）
- 容器格式处理（封装/解封装）
- 滤镜功能（视频/音频滤镜效果）
- 格式转换（像素格式、采样率转换）
- 硬件加速（CUDA、VAAPI、QSV 等）
- API 使用正确性

### 1.2 测试配置文件结构

每个 `.mak` 文件定义一组相关测试：

```makefile
# 依赖检查宏
FATE_LIBAVCODEC-$(CONFIG_H264_DECODER) += fate-h264-conformance

# 测试定义
fate-h264-conformance: libavcodec/tests/h264$(EXESUF)
fate-h264-conformance: CMD = run libavcodec/tests/h264$(EXESUF) $(TARGET_SAMPLES)/h264-conformance/SVA_NL2_E.264
fate-h264-conformance: CMP = null
```

**关键字段说明**：
- `FATE_<CATEGORY>-$(CONFIG_XXX)` - 条件编译，根据配置启用测试
- `CMD` - 测试命令（`run` 调用 fate-run.sh）
- `CMP` - 比较方式：`diff`、`oneoff`、`stddev`、`null`
- `REF` - 参考文件路径
- `FUZZ` - 允许的误差范围

### 1.3 主要配置文件分类

| 类别 | 文件 | 测试内容 |
|------|------|----------|
| **编解码器** | `acodec.mak`, `vcodec.mak` | 音视频编解码器 |
| **格式** | `lavf-audio.mak`, `lavf-video.mak`, `lavf-container.mak` | 容器格式 |
| **滤镜** | `filter-audio.mak`, `filter-video.mak` | 音视频滤镜 |
| **组件** | `libavcodec.mak`, `libavformat.mak`, `libavutil.mak` | 内部组件 |
| **特定格式** | `h264.mak`, `hevc.mak`, `vp9.mak` | 特定编解码器 |
| **API** | `api.mak` | 公共 API 测试 |

### 1.4 测试辅助函数（fate-run.sh）

```bash
# 编码-解码-比较
enc_dec <enc_fmt_in> <srcfile> <enc_fmt_out> <enc_opt> <dec_fmt_out> <dec_opt>

# 转码测试
transcode <src_fmt> <srcfile> <enc_fmt> <enc_opt> <final_encode> <ffprobe_opts>

# 滤镜测试
video_filter <filters> <additional_opts>

# 音视频比较
pcm <ffmpeg_args>
framecrc <ffmpeg_args>
```

### 1.5 运行 FATE 测试

```bash
# 完整测试套件（需要样本文件）
make fate

# 仅运行部分测试
make fate-h264
make fate-filter-video

# 列出所有测试
make fate-list

# 列出失败的测试
make fate-list-failing

# 指定样本目录
make fate SAMPLES=/path/to/samples

# 同步样本文件
make fate-rsync SAMPLES=/path/to/samples
```

### 1.6 测试结果分析

测试结果存储在 `tests/data/fate/*.rep`：

```
test_name:0:<base64_cmp>:<base64_err>
```

- `0` = 成功，非零 = 失败
- 查看 `.diff` 文件了解差异
- 查看 `.err` 文件了解错误输出

## 2. checkasm - 汇编级测试框架

### 2.1 概述

checkasm 是 FFmpeg 的 SIMD 优化验证框架，确保：

- **正确性**：汇编实现与 C 参考实现结果一致
- **性能**：基准测试汇编优化性能提升
- **鲁棒性**：检测寄存器保存/恢复错误

### 2.2 核心架构

```c
// 测试注册
void checkasm_check_h264dsp(void);

// 测试函数定义
static void check_idct(void)
{
    // 声明函数签名
    declare_func(void, uint8_t *dst, uint8_t *src, int stride);

    // 检查函数（启用测试）
    if (check_func(h264_idct_add, "h264_idct_add")) {
        // 随机化测试数据
        randomize_buffers();

        // 调用参考实现
        call_ref(dst1, src, stride);

        // 调用汇编实现
        call_new(dst2, src, stride);

        // 比较结果
        if (memcmp(dst1, dst2, size)) fail();
    }

    // 报告测试结果
    report("h264_idct_add");
}
```

### 2.3 平台特定包装

checkasm 提供跨平台的函数调用包装：

| 架构 | 包装函数 | 特殊检查 |
|------|----------|----------|
| **x86_64** | `checkasm_checked_call` | 栈污染检测（0xdeadbeef） |
| **x86_32** | `checkasm_checked_call` | MMX 寄存器检查 |
| **ARM** | `checkasm_checked_call_vfp/novfp` | VFP 寄存器保存 |
| **AArch64** | `checkasm_checked_call` | 栈污染检测 |
| **RISC-V** | `checkasm_set_function` | 间接调用包装 |

### 2.4 测试覆盖范围

```bash
# 编译 checkasm
make checkasm

# 运行所有测试
./tests/checkasm/checkasm

# 运行特定测试
./tests/checkasm/checkasm --test=h264dsp
./tests/checkasm/checkasm --test=sw_scale

# 性能基准测试
./tests/checkasm/checkasm --bench=hevc_idct
```

### 2.5 已实现的组件测试

- **libavcodec**: H.264/HEVC/VP8/VP9 DSP、AAC、FLAC、DCA 等
- **libavfilter**: 滤镜 DSP（blend、colorspace、bwdif 等）
- **libswscale**: 所有缩放和颜色转换函数
- **libavutil**: AES、CRC、AV_TX、float_dsp 等

## 3. API 测试

### 3.1 概述

API 测试提供公共 API 的完整使用示例，验证：

- API 调用顺序正确性
- 内存管理（分配/释放）
- 错误处理
- 多线程安全性

### 3.2 测试程序列表

| 测试程序 | 功能 | 依赖 |
|---------|------|------|
| `api-h264-test` | H.264 解码完整流程 | H.264 解码器 |
| `api-h264-slice-test` | H.264 切片线程解码 | H.264 解码器 |
| `api-seek-test` | 文件定位（seek）测试 | FLV 格式 |
| `api-flac-test` | FLAC 编解码循环 | FLAC 编解码器 |
| `api-band-test` | 带宽自适应测试 | H.263 解码器 |
| `api-threadmessage-test` | 线程消息队列 | 线程支持 |

### 3.3 API 测试示例

```c
// 典型的解码流程
int video_decode_example(const char *input_filename) {
    AVFormatContext *fmt_ctx = NULL;
    const AVCodec *codec = NULL;
    AVCodecContext *codec_ctx = NULL;
    AVFrame *frame = NULL;
    AVPacket *pkt = NULL;

    // 1. 打开输入
    avformat_open_input(&fmt_ctx, input_filename, NULL, NULL);

    // 2. 查找流信息
    avformat_find_stream_info(fmt_ctx, NULL);

    // 3. 查找解码器
    codec = avcodec_find_decoder(stream->codecpar->codec_id);

    // 4. 分配解码器上下文
    codec_ctx = avcodec_alloc_context3(codec);

    // 5. 复制参数
    avcodec_parameters_to_context(codec_ctx, stream->codecpar);

    // 6. 打开解码器
    avcodec_open2(codec_ctx, codec, NULL);

    // 7. 读取和解码帧
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        avcodec_send_packet(codec_ctx, pkt);
        avcodec_receive_frame(codec_ctx, frame);
    }

    // 8. 清理资源
    av_frame_free(&frame);
    avcodec_free_context(&codec_ctx);
    avformat_close_input(&fmt_ctx);
}
```

## 4. 测试辅助工具

### 4.1 测试数据生成

| 工具 | 功能 | 输出 |
|------|------|------|
| `videogen` | 生成测试视频序列 | PGM 图像序列 |
| `audiogen` | 生成测试音频 | SW 格式音频 |
| `rotozoom` | 旋转缩放测试 | YUV 视频 |

### 4.2 比较工具

| 工具 | 功能 | 用途 |
|------|------|------|
| `tiny_psnr` | PSNR 计算 | 视频质量比较 |
| `tiny_ssim` | SSIM 计算 | 结构相似度 |
| `audiomatch` | 音频比较 | 音频质量验证 |
| `base64` | 编码工具 | 二进制比较 |

### 4.3 比较方法

```bash
# CRC 比较（精确匹配）
CRC - 整个文件的 CRC32

# 帧级 CRC（逐帧比较）
framecrc - 每帧独立校验

# MD5（哈希比较）
md5 - 文件哈希

# 逐字节差异
diff - 完全匹配

# 允许误差
oneoff - 单像素/样本最大差异
stddev - 标准差阈值
```

## 5. 测试最佳实践

### 5.1 添加新的 FATE 测试

1. **确定测试类别**（编解码器/滤镜/格式）
2. **选择配置文件**（如 `libavcodec.mak`）
3. **添加测试定义**：

```makefile
FATE_LIBAVCODEC-$(CONFIG_MYCODEC) += fate-mycodec-test
fate-mycodec-test: libavcodec/tests/mycodec$(EXESUF)
fate-mycodec-test: CMD = run libavcodec/tests/mycodec$(EXESUF)
fate-mycodec-test: CMP = null
```

4. **实现测试程序**（如需要）：

```c
// libavcodec/tests/mycodec.c
int main(void) {
    // 测试逻辑
    return ret;
}
```

5. **更新 Makefile**：添加编译规则

### 5.2 添加新的 checkasm 测试

1. **创建测试文件**：`tests/checkasm/mycomponent.c`

```c
#include "checkasm.h"
#include "libavcodec/mycomponent.h"

static void check_myfunction(void) {
    declare_func(void, uint8_t *dst, const uint8_t *src, int stride);

    if (check_func(myfunction, "myfunction")) {
        randomize_buffers();
        call_ref(dst_ref, src, stride);
        call_new(dst_new, src, stride);
        checkasm_check(uint8_t, dst_new, stride, dst_ref, stride,
                       width, height, "dst");
    }

    bench_new(dst_new, src, stride);
}

void checkasm_check_mycomponent(void) {
    check_myfunction();
    report("mycomponent");
}
```

2. **注册测试**：在 `checkasm.h` 添加声明

3. **调用测试**：在 `checkasm.c` 添加调用

### 5.3 调试测试失败

```bash
# 保留中间文件
make fate KEEP=1

# 查看详细输出
make fate V=1

# 运行单个测试
make fate-h264-conformance V=1

# 检查参考文件
cat tests/data/fate/h264-conformance.diff
cat tests/data/fate/h264-conformance.err
```

### 5.4 代码覆盖率测试

```bash
# 配置 gcov
./configure --toolchain=gcov

# 编译和测试
make clean
make -j$(nproc)
make fate

# 生成报告
make lcov

# 查看报告
open lcov/index.html
```

## 6. CI/CD 集成

### 6.1 GitHub Actions

```yaml
- name: Run FATE
  run: |
    make -j$(nproc)
    make fate SAMPLES=/path/to/samples
```

### 6.2 性能回归检测

```bash
# 基准测试保存
./tests/checkasm/checkasm --bench > benchmarks.txt

# 比较性能
diff old_benchmarks.txt benchmarks.txt
```

## 7. 已知限制

1. **样本文件大小**：FATE 样本 > 5GB，需单独下载
2. **外部依赖**：某些测试需要特定编解码器支持
3. **平台差异**：参考输出可能因平台略有不同
4. **硬件加速**：硬件测试需要特定硬件

## 8. 相关资源

- **FATE 网页**：https://fate.ffmpeg.org（公共测试报告）
- **样本下载**：rsync://fate-suite.ffmpeg.org/fate-suite/
- **Wiki**：https://trac.ffmpeg.org/wiki/FATE

---

## 变更记录 (Changelog)

### 2026-01-17 10:30:00 - 测试框架文档创建
- 创建 tests/CLAUDE.md
- 覆盖 FATE、checkasm、API 测试三大组件
- 记录测试结构、运行方式和扩展方法
- 提供完整测试最佳实践指南

---

*覆盖率统计：FATE 测试 1000+ 用例，checkasm 80+ 组件测试，API 测试 7 个完整示例*
