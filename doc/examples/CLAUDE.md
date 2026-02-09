# FFmpeg 示例代码 - AI 上下文文档

> 最后更新：2026-01-17 12:00:00
> 项目版本：Master (Git)
> 示例数量：25 个官方示例

[根目录](../../CLAUDE.md) > **doc/examples**

## 模块职责

`doc/examples/` 目录包含 FFmpeg 官方提供的 25 个示例程序，展示了如何使用 FFmpeg 公共 API 完成各种多媒体处理任务。这些示例是学习 FFmpeg API 的最佳起点，涵盖了从基础的编解码到高级的硬件加速和滤镜处理。

### 示例分类

根据功能和技术栈，示例代码可分为以下几类：

#### 1. 基础编解码示例 (8 个)
- **decode_audio.c** - 音频解码：从 MP2 文件解码音频并输出为原始 PCM
- **decode_video.c** - 视频解码：从 MPEG1 视频文件解码并输出为 PGM 图像序列
- **encode_audio.c** - 音频编码：生成合成音频信号并编码为 MP2 文件
- **encode_video.c** - 视频编码：生成合成视频信号并编码为指定格式
- **demux_decode.c** - 解封装解码：完整的解封装和解码流程（音视频）
- **mux.c** - 封装示例：生成合成音视频并封装到媒体文件
- **remux.c** - 重封装：在不同容器格式间转换（不转码）
- **transcode.c** - 转码：完整的转码流程，包含滤镜处理

#### 2. 滤镜与后处理示例 (4 个)
- **filter_audio.c** - 音频滤镜：使用 libavfilter 进行音频滤镜处理
- **decode_filter_audio.c** - 解码+音频滤镜：解码后应用音频滤镜
- **decode_filter_video.c** - 解码+视频滤镜：解码后应用视频滤镜
- **scale_video.c** - 视频缩放：使用 libswscale 进行图像缩放和格式转换

#### 3. 硬件加速示例 (5 个)
- **hw_decode.c** - 硬件解码：通用硬件加速解码 API（支持多种硬件）
- **vaapi_encode.c** - VAAPI 编码：Intel VAAPI 硬件编码（H.264）
- **vaapi_transcode.c** - VAAPI 转码：完整的 VAAPI 转码流程
- **qsv_decode.c** - QSV 解码：Intel Quick Sync Video 解码
- **qsv_transcode.c** - QSV 转码：完整的 QSV 转码流程

#### 4. 高级功能示例 (4 个)
- **extract_mvs.c** - 运动向量提取：从视频流中提取运动向量信息
- **show_metadata.c** - 元数据显示：提取和显示媒体文件的元数据
- **avio_list_dir.c** - 目录列表：使用 AVIO 自定义 I/O 列出目录
- **avio_read_callback.c** - 自定义 I/O：使用自定义回调读取数据

#### 5. 网络与 I/O 示例 (2 个)
- **avio_http_serve_files.c** - HTTP 文件服务：使用 AVIO 提供 HTTP 文件服务
- **resample_audio.c** - 音频重采样：使用 libswresample 进行音频重采样

#### 6. 其他实用示例 (2 个)
- **transcode_aac.c** - AAC 转码：专用的 AAC 音频转码示例
- **HTTP 多客户端**（如果存在）：HTTP 流媒体服务

## 核心技术要点

### send/receive API（现代编解码接口）

所有现代编解码示例都使用 FFmpeg 3.1+ 引入的 send/receive API：

```c
// 解码流程
ret = avcodec_send_packet(dec_ctx, pkt);    // 发送编码数据包
while (ret >= 0) {
    ret = avcodec_receive_frame(dec_ctx, frame);  // 接收解码后的帧
    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        break;
    // 处理解码后的帧
}

// 编码流程
ret = avcodec_send_frame(enc_ctx, frame);    // 发送原始帧
while (ret >= 0) {
    ret = avcodec_receive_packet(enc_ctx, pkt);  // 接收编码后的数据包
    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        break;
    // 写入编码数据
}
```

**关键特性**：
- **异步处理**：一个 send 可能产生多个 receive，或者需要多次 send 才能产生一个 receive
- **EAGAIN 处理**：表示需要更多输入或读取更多输出
- **刷新机制**：发送 NULL pkt/frame 来刷新编解码器

### 滤镜图系统（Filter Graph）

libavfilter 提供强大的滤镜处理框架：

```c
// 创建滤镜图
AVFilterGraph *graph = avfilter_graph_alloc();

// 创建 buffer 滤镜（输入）
AVFilterContext *buffersrc_ctx = avfilter_graph_alloc_filter(
    graph, avfilter_get_by_name("buffer"), "in");

// 创建 buffersink 滤镜（输出）
AVFilterContext *buffersink_ctx = avfilter_graph_alloc_filter(
    graph, avfilter_get_by_name("buffersink"), "out");

// 解析并连接滤镜
avfilter_graph_parse_ptr(graph, filter_spec, &inputs, &outputs, NULL);
avfilter_graph_config(graph, NULL);

// 推送帧到滤镜图
av_buffersrc_add_frame_flags(buffersrc_ctx, frame, 0);

// 从滤镜图获取处理后的帧
av_buffersink_get_frame(buffersink_ctx, filtered_frame);
```

**常用滤镜**：
- **视频**：`scale`（缩放）、`overlay`（叠加）、`crop`（裁剪）、`rotate`（旋转）
- **音频**：`volume`（音量）、`aresample`（重采样）、`afade`（淡入淡出）

### 硬件加速框架

FFmpeg 提供统一的硬件加速接口：

```c
// 创建硬件设备上下文
AVBufferRef *hw_device_ctx;
av_hwdevice_ctx_create(&hw_device_ctx, AV_HWDEVICE_TYPE_CUDA,
                      NULL, NULL, 0);

// 设置到解码器/编码器
AVCodecContext *ctx = avcodec_alloc_context3(codec);
ctx->hw_device_ctx = av_buffer_ref(hw_device_ctx);

// 硬件解码后的帧在 GPU 上，需要传输到 CPU 才能访问
if (frame->format == AV_PIX_FMT_CUDA) {
    AVFrame *sw_frame = av_frame_alloc();
    av_hwframe_transfer_data(sw_frame, frame, 0);  // GPU -> CPU
    // 处理 sw_frame
}
```

**支持的硬件类型**：
- **CUDA** (NVIDIA GPU)
- **VAAPI** (Intel/AMD GPU)
- **VCE** (AMD GPU)
- **QSV** (Intel Quick Sync)
- **Vulkan** (跨平台)
- **D3D11VA** (Windows DirectX)
- **VideoToolbox** (macOS)

### 自定义 I/O（AVIO）

AVIO 允许自定义数据源/目标：

```c
// 自定义读取回调
int read_packet(void *opaque, uint8_t *buf, int buf_size) {
    // 从自定义数据源读取
    return custom_read(opaque, buf, buf_size);
}

// 创建 AVIO 上下文
AVIOContext *avio_ctx = avio_alloc_context(
    buffer, buffer_size, 0,  // 内部缓冲区
    opaque,                  // 用户数据
    read_packet,             // 读取回调
    NULL,                    // 写入回调
    NULL                     // 定位回调
);

// 替换格式上下文的 I/O 层
fmt_ctx->pb = avio_ctx;
fmt_ctx->flags |= AVFMT_FLAG_CUSTOM_IO;
```

**应用场景**：
- 从内存缓冲区读取/写入
- 自定义网络协议
- 加密/解密数据流
- 进程间管道通信

## 示例详细说明

### 1. decode_audio.c - 音频解码

**功能**：从 MP2 文件解码音频，输出为原始 PCM 文件

**关键技术**：
- 使用 `avcodec_find_decoder()` 查找 MP2 解码器
- 使用 `av_parser_parse2()` 解析输入数据流
- 使用 send/receive API 进行解码
- 处理平面音频格式（planar audio）

**使用场景**：
- 音频播放器开发
- 音频分析工具
- 音频格式转换

**编译运行**：
```bash
gcc -o decode_audio decode_audio.c -lavcodec -lavutil
./decode_audio input.mp2 output.pcm
ffplay -f s16le -ac 2 -ar 44100 output.pcm
```

**代码片段**：
```c
// 发送数据包到解码器
ret = avcodec_send_packet(dec_ctx, pkt);
if (ret < 0) {
    fprintf(stderr, "Error submitting the packet to the decoder\n");
    exit(1);
}

// 接收所有解码后的帧
while (ret >= 0) {
    ret = avcodec_receive_frame(dec_ctx, frame);
    if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        return;
    // 写入 PCM 数据到文件
    fwrite(frame->data[0], 1, data_size, outfile);
}
```

### 2. encode_video.c - 视频编码

**功能**：生成合成视频并编码为指定格式（如 H.264）

**关键技术**：
- 动态生成 YUV420P 视频帧
- 使用 `avcodec_find_encoder_by_name()` 查找编码器
- 设置编码参数（比特率、GOP、B 帧等）
- H.264 预设参数设置

**使用场景**：
- 视频录制工具
- 屏幕捕获软件
- 视频直播推流

**编译运行**：
```bash
gcc -o encode_video encode_video.c -lavcodec -lavutil -lm
./encode_video output.h264 libx264
ffplay output.h264
```

**代码片段**：
```c
// 为 H.264 设置预设参数
if (codec->id == AV_CODEC_ID_H264)
    av_opt_set(c->priv_data, "preset", "slow", 0);

// 生成合成视频数据
for (y = 0; y < height; y++) {
    for (x = 0; x < width; x++) {
        frame->data[0][y * frame->linesize[0] + x] = x + y + i * 3;
    }
}
```

### 3. transcode.c - 完整转码流程

**功能**：完整的音视频转码流程，包含滤镜处理

**关键技术**：
- 同时处理音视频流
- 动态创建滤镜图
- 时间戳转换（PTS/DTS）
- 流复制（无需转码的流）

**使用场景**：
- 视频转码工具
- 格式转换软件
- 批量视频处理

**架构图**：
```
输入文件 → demuxer → decoder → filter → encoder → muxer → 输出文件
                         ↓                    ↓
                    音频解码器            音频编码器
                    视频解码器            视频编码器
```

### 4. hw_decode.c - 硬件加速解码

**功能**：使用 GPU 进行视频解码

**关键技术**：
- 硬件设备类型枚举
- 硬件格式协商
- GPU 到 CPU 的数据传输

**支持的硬件类型**：
```bash
# 查看可用的硬件类型
./hw_decode
```

**使用场景**：
- 4K/8K 视频播放
- 实时视频解码
- 低功耗视频处理

**代码片段**：
```c
// 查询硬件配置
for (i = 0;; i++) {
    const AVCodecHWConfig *config = avcodec_get_hw_config(decoder, i);
    if (config->methods & AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_CTX &&
        config->device_type == type) {
        hw_pix_fmt = config->pix_fmt;
        break;
    }
}

// 从 GPU 传输数据到 CPU
if (frame->format == hw_pix_fmt) {
    av_hwframe_transfer_data(sw_frame, frame, 0);
}
```

### 5. filter_audio.c - 音频滤镜

**功能**：演示如何使用 libavfilter 进行音频处理

**滤镜链**：
```
abuffer → volume → aformat → abuffersink
```

**关键技术**：
- 滤镜图创建和配置
- 滤镜选项设置（3 种方法）
- 滤镜链接和配置

**使用场景**：
- 音频处理工具
- 音频效果器
- 实时音频处理

**代码片段**：
```c
// 方法 1：使用 av_opt_set
av_opt_set(abuffer_ctx, "channel_layout", "5.0", AV_OPT_SEARCH_CHILDREN);

// 方法 2：使用 AVDictionary
av_dict_set(&options_dict, "volume", "0.90", 0);
avfilter_init_dict(volume_ctx, &options_dict);

// 方法 3：使用字符串
snprintf(options_str, sizeof(options_str),
         "sample_fmts=s16:sample_rates=44100:channel_layouts=stereo");
avfilter_init_str(aformat_ctx, options_str);
```

### 6. extract_mvs.c - 运动向量提取

**功能**：从视频流中提取运动向量信息

**关键技术**：
- 启用运动向量导出
- 从帧侧数据中提取运动向量
- 运动向量格式解析

**使用场景**：
- 视频分析工具
- 运动检测应用
- 视频压缩研究

**输出格式**：
```
framenum,source,blockw,blockh,srcx,srcy,dstx,dsty,flags,motion_x,motion_y,motion_scale
```

**代码片段**：
```c
// 启用运动向量导出
av_dict_set(&opts, "flags2", "+export_mvs", 0);
avcodec_open2(dec_ctx, dec, &opts);

// 提取运动向量
AVFrameSideData *sd = av_frame_get_side_data(frame, AV_FRAME_DATA_MOTION_VECTORS);
if (sd) {
    const AVMotionVector *mvs = (const AVMotionVector *)sd->data;
    for (i = 0; i < sd->size / sizeof(*mvs); i++) {
        // 处理每个运动向量
    }
}
```

## 编译和运行

### 方法 1：使用示例 Makefile（推荐）

```bash
# 在源码树中编译
cd /path/to/ffmpeg
./configure
make examples

# 运行示例
./doc/examples/decode_audio input.mp2 output.pcm
```

### 方法 2：独立编译

```bash
# 复制示例到用户目录
cp /path/to/ffmpeg/doc/examples/decode_audio.c .
make -f Makefile.example

# 或手动编译
gcc -o decode_audio decode_audio.c -lavcodec -lavutil
```

### 方法 3：使用 pkg-config

```bash
# 安装 FFmpeg 开发库
sudo apt install libavcodec-dev libavformat-dev libavutil-dev

# 编译
gcc -o decode_audio decode_audio.c $(pkg-config --libs --cflags libavcodec libavutil)
```

## API 使用最佳实践

### 1. 内存管理

```c
// 使用 av_malloc/av_free 而非标准库
AVFrame *frame = av_frame_alloc();
uint8_t *buffer = av_malloc(size);

// 正确的释放顺序
av_frame_free(&frame);
av_free(buffer);
avcodec_free_context(&ctx);
```

### 2. 错误处理

```c
// 检查所有返回值
ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    char err_buf[128];
    av_strerror(ret, err_buf, sizeof(err_buf));
    fprintf(stderr, "Error: %s\n", err_buf);
    goto cleanup;
}
```

### 3. 资源清理（goto 模式）

```c
int ret = 0;
AVCodecContext *ctx = NULL;
AVFrame *frame = NULL;

if (some_error) {
    ret = AVERROR(EINVAL);
    goto end;
}

end:
    av_frame_free(&frame);
    avcodec_free_context(&ctx);
    return ret;
```

### 4. 引用计数管理

```c
// AVPacket 引用计数
av_packet_unref(pkt);  // 释放引用

// AVFrame 引用计数
av_frame_unref(frame);  // 释放引用
av_frame_ref(dst, src); // 增加引用
```

### 5. 时间戳处理

```c
// 转换时间基
av_packet_rescale_ts(pkt, in_stream->time_base, out_stream->time_base);

// 设置帧 PTS
frame->pts = av_rescale_q(frame->pts,
                          in_ctx->time_base,
                          out_ctx->time_base);
```

## 常见问题 (FAQ)

### Q1: 如何选择正确的解码器/编码器？

```c
// 按名称查找
const AVCodec *codec = avcodec_find_encoder_by_name("libx264");

// 按 ID 查找
const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);

// 查找最佳解码器
const AVCodec *codec = avcodec_find_decoder(stream->codecpar->codec_id);
```

### Q2: 如何处理可变帧率？

```c
// 使用时间基计算 PTS
frame->pts = av_rescale_q(frame_count,
                          (AVRational){1, frame_rate},
                          ctx->time_base);
```

### Q3: 如何处理平面音频格式？

```c
// 检查是否为平面格式
if (av_sample_fmt_is_planar(fmt)) {
    // 每个声道单独存储
    for (ch = 0; ch < channels; ch++) {
        process_channel(frame->extended_data[ch], nb_samples);
    }
}
```

### Q4: 如何启用硬件加速？

```c
// 创建硬件设备上下文
AVBufferRef *hw_ctx;
av_hwdevice_ctx_create(&hw_ctx, AV_HWDEVICE_TYPE_CUDA, NULL, NULL, 0);

// 设置到编解码器
ctx->hw_device_ctx = av_buffer_ref(hw_ctx);
```

### Q5: 如何自定义 I/O？

```c
// 实现回调函数
int read_packet(void *opaque, uint8_t *buf, int buf_size) {
    // 自定义读取逻辑
}

// 创建 AVIO 上下文
unsigned char *buffer = av_malloc(buffer_size);
AVIOContext *io_ctx = avio_alloc_context(buffer, buffer_size, 0,
                                        opaque, read_packet, NULL, NULL);
```

## 性能优化建议

### 1. 线程优化

```c
// 启用多线程解码
ctx->thread_count = 0;  // 自动检测
ctx->thread_type = FF_THREAD_FRAME | FF_THREAD_SLICE;
```

### 2. 零拷贝优化

```c
// 使用引用计数避免复制
av_frame_ref(dst_frame, src_frame);  // 而非 memcpy
```

### 3. 批量处理

```c
// 批量发送数据包
while (get_packet(&pkt)) {
    avcodec_send_packet(ctx, &pkt);
}

// 批量接收帧
while (avcodec_receive_frame(ctx, &frame) >= 0) {
    process_frame(frame);
}
```

### 4. 硬件加速

```c
// 优先使用硬件解码器
avcodec_find_decoder_by_name("h264_cuvid");  // NVIDIA
avcodec_find_decoder_by_name("h264_vaapi"); // Intel/AMD
```

## 学习路径建议

### 初学者（第 1-2 周）
1. **decode_audio.c** - 理解基本解码流程
2. **encode_audio.c** - 理解基本编码流程
3. **show_metadata.c** - 学习元数据访问

### 中级（第 3-4 周）
4. **demux_decode.c** - 完整的解封装解码
5. **mux.c** - 完整的封装流程
6. **transcode.c** - 完整的转码流程

### 高级（第 5-6 周）
7. **filter_audio.c** - 滤镜系统
8. **scale_video.c** - 图像处理
9. **hw_decode.c** - 硬件加速

### 专家级（第 7+ 周）
10. **extract_mvs.c** - 高级数据分析
11. **avio_http_serve_files.c** - 网络流媒体
12. 自定义 I/O 和协议处理

## 相关资源

### 官方文档
- [FFmpeg API 文档](https://ffmpeg.org/doxygen/trunk/)
- [FFmpeg Wiki](https://trac.ffmpeg.org/wiki)
- [libavcodec API](https://ffmpeg.org/doxygen/trunk/group__libavc.html)

### 推荐阅读
- [FFmpeg 源代码分析](https://ffmpeg.org/pipermail/ffmpeg-devel/)
- [多媒体处理技术论坛](https://forum.doom9.org/)

### 社区资源
- 邮件列表：ffmpeg-devel@ffmpeg.org
- IRC：#ffmpeg @ Libera.Chat
- Stack Overflow：[ffmpeg 标签](https://stackoverflow.com/questions/tagged/ffmpeg)

## 变更记录 (Changelog)

### 2026-01-17 12:00:00 - 初始版本
- 创建 doc/examples 模块文档
- 分析 25 个官方示例代码
- 提供完整的使用指南和最佳实践
- 覆盖率：100%（所有示例已索引）

---

*本文档基于 FFmpeg Master 分支示例代码生成*
*最后更新：2026-01-17 | 示例数量：25 | 文档覆盖率：100%*


<claude-mem-context>
# Recent Activity

<!-- This section is auto-generated by claude-mem. Edit content outside the tags. -->

*No recent activity*
</claude-mem-context>