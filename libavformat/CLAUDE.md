# libavformat - FFmpeg 格式处理库

[根目录](../CLAUDE.md) > **libavformat**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libavformat 处理多媒体容器格式（封装/解封装）和 I/O 协议，是 FFmpeg 的流管理层。

### 核心功能
- **Demuxers（解封装）**：将容器文件拆分为音视频流
- **Muxers（封装）**：将音视频流封装到容器文件
- **协议**：文件、HTTP、RTMP、RTSP、TCP 等数据访问
- **元数据处理**：读取/写入文件元数据
- **Seek 支持**：时间定位和随机访问

## 入口与启动

### 主要头文件
```c
#include <libavformat/avformat.h>    // 核心格式 API
#include <libavformat/avio.h>        // I/O API
```

### 典型使用流程（解封装）
```c
// 1. 打开输入文件
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.mp4", NULL, NULL);

// 2. 读取流信息
avformat_find_stream_info(fmt_ctx, NULL);

// 3. 查找视频/音频流
int video_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);

// 4. 读取包
AVPacket *pkt = av_packet_alloc();
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == video_idx) {
        // 处理视频包
    }
    av_packet_unref(pkt);
}

// 5. 关闭
avformat_close_input(&fmt_ctx);
av_packet_free(&pkt);
```

### 典型使用流程（封装）
```c
// 1. 分配输出上下文
AVFormatContext *out_fmt = NULL;
avformat_alloc_output_context2(&out_fmt, NULL, NULL, "output.mp4");

// 2. 添加流（需要来自编码器）
AVStream *stream = avformat_new_stream(out_fmt, NULL);
avcodec_parameters_from_context(stream->codecpar, codec_ctx);

// 3. 打开输出
avio_open(&out_fmt->pb, "output.mp4", AVIO_FLAG_WRITE);
avformat_write_header(out_fmt, NULL);

// 4. 写入包
av_interleaved_write_frame(out_fmt, pkt);

// 5. 完成写入
av_write_trailer(out_fmt);
avio_closep(&out_fmt->pb);
avformat_free_context(out_fmt);
```

## 对外接口

### 核心 API

#### 格式查找
```c
// 查找输入格式
const AVInputFormat *fmt = av_find_input_format("mp4");

// 查找输出格式
const AVOutputFormat *fmt = av_guess_format(NULL, "output.mp4", NULL);

// 遍历所有格式
void *iter = NULL;
while ((fmt = av_demuxer_iterate(&iter))) {
    printf("Format: %s\n", fmt->name);
}
```

#### 流信息
```c
// 查找最佳流
int idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);

// 获取流
AVStream *stream = fmt_ctx->streams[idx];

// 时长（秒）
double duration = fmt_ctx->duration / (double)AV_TIME_BASE;
```

#### Seek 操作
```c
// Seek 到指定时间
int64_t timestamp = 10 * AV_TIME_BASE;  // 10 秒
av_seek_frame(fmt_ctx, -1, timestamp, AVSEEK_FLAG_BACKWARD);

// Seek 到特定流
av_seek_frame(fmt_ctx, video_idx, timestamp, 0);
```

### 主要数据结构

#### AVFormatContext（格式上下文）
```c
typedef struct AVFormatContext {
    const AVInputFormat *iformat;   // 输入格式
    const AVOutputFormat *oformat;  // 输出格式
    AVIOContext *pb;                 // I/O 上下文
    unsigned int nb_streams;         // 流数量
    AVStream **streams;              // 流数组
    int64_t duration;                // 时长（微秒）
    int64_t bit_rate;                // 比特率
    char filename[1024];             // 文件名
    AVDictionary *metadata;          // 元数据
} AVFormatContext;
```

#### AVStream（流描述）
```c
typedef struct AVStream {
    int index;                       // 流索引
    AVCodecParameters *codecpar;     // 编解码参数
    AVRational time_base;            // 时间基
    int64_t duration;                // 流时长
    int64_t nb_frames;               // 帧数
    AVDictionary *metadata;          // 流元数据
    AVDiscard discard;               // 丢弃策略
} AVStream;
```

#### AVInputFormat / AVOutputFormat
```c
typedef struct AVInputFormat {
    const char *name;                // 格式名称
    const char *long_name;           // 长名称
    int flags;                       // 标志
    const char *extensions;          // 扩展名（逗号分隔）
    // ... 内部函数指针
} AVInputFormat;
```

### 协议层（avio）
```c
// 自定义 I/O
AVIOContext *io_ctx = avio_alloc_context(
    buffer, buffer_size,
    write_flag, opaque,
    read_packet, write_packet, seek
);

// 打开网络（需要）
avformat_network_init();

// 使用自定义 I/O
avformat_open_input(&fmt_ctx, NULL, NULL, &opts);
av_dict_set(&opts, "protocol_whitelist", "file,http,tcp", 0);
```

## 关键依赖与配置

### 编译配置
```bash
# 启用所有 demuxers/muxers
--enable-demuxers
--enable-muxers

# 启用协议
--enable-protocols

# 启用网络
--enable-network

# 禁用特定协议（安全考虑）
--disable-protocol=http,https,rtmp

# 协议白名单/黑名单
--protocol-whitelist=file,http,tcp
--protocol-blacklist=rtmp,rtsp
```

### 依赖关系
- **依赖**：libavutil、libavcodec
- **可选依赖**：
  - 网络：openssl、gnutls
  - 特定格式：libmodplug、libbluray

## 数据模型

### 时间戳处理
```c
// 不同时间基之间的转换
int64_t src_pts = ...;
int64_t dst_pts = av_rescale_q(
    src_pts,
    src_stream->time_base,
    dst_stream->time_base
);

// 从秒转换为时间基
AVRational time_base = {1, 90000};  // 90 kHz
int64_t pts = (int64_t)(seconds / av_q2d(time_base));
```

### 流索引
```c
// FFmpeg 中的流索引
// streams[0] = 第一个流
// streams[1] = 第二个流
// ...

// AVPacket 中的 stream_index 指向该流
pkt->stream_index == video_idx
```

### 元数据
```c
// 读取元数据
AVDictionaryEntry *tag = NULL;
while ((tag = av_dict_iterate(fmt_ctx->metadata, tag))) {
    printf("%s: %s\n", tag->key, tag->value);
}

// 设置元数据
av_dict_set(&fmt_ctx->metadata, "title", "My Title", 0);
av_dict_set(&fmt_ctx->metadata, "artist", "Artist Name", 0);
```

## 测试与质量

### 测试文件
- **位置**：`tests/fate/lavf-*.mak`
- **测试**：容器格式读写、协议测试

### FATE 测试覆盖
- **音频容器**：`lavf-audio.mak`
- **视频容器**：`lavf-video.mak`
- **图片容器**：`lavf-image.mak`
- **Seek 测试**：`seek.mak`

### 常见问题（FAQ）

#### Q: 如何处理网络流？
A: 启用网络支持并设置超时：
```c
avformat_network_init();

AVDictionary *opts = NULL;
av_dict_set(&opts, "timeout", "5000000", 0);  // 5 秒
av_dict_set(&opts, "reconnect", "1", 0);      // 自动重连
avformat_open_input(&fmt_ctx, "http://...", NULL, &opts);
```

#### Q: 如何实现流式传输？
A:
```c
// 使用 pipe 协议（stdin/stdout）
avformat_open_input(&fmt_ctx, "pipe:0", NULL, NULL);  // 从 stdin 读取

// 写入到 stdout
avio_open(&out_fmt->pb, "pipe:1", AVIO_FLAG_WRITE);
```

#### Q: 如何处理变帧率（VFR）？
A:
```c
// 从包时间戳计算实际帧率
AVRational framerate = av_guess_frame_rate(fmt_ctx, stream, NULL);
double fps = av_q2d(framerate);
```

#### Q: 如何提取嵌入的封面？
A:
```c
// 查找附加流（通常是封面）
int cover_idx = -1;
for (int i = 0; i < fmt_ctx->nb_streams; i++) {
    if (fmt_ctx->streams[i]->disposition & AV_DISPOSITION_ATTACHED_PIC) {
        cover_idx = i;
        break;
    }
}
```

#### Q: 如何处理损坏的文件？
A:
```c
// 启用错误恢复
AVDictionary *opts = NULL;
av_dict_set(&opts, "err_detect", "ignore_err", 0);
av_dict_set(&opts, "skip_frame", "nonkey", 0);  // 跳过非关键帧
avformat_open_input(&fmt_ctx, filename, NULL, &opts);
```

## 相关文件清单

### 头文件（公共 API）
```
libavformat/avformat.h       # 主入口
libavformat/avio.h           # I/O API
```

### 源文件（格式实现）
```
libavformat/allformats.c     # 格式注册
libavformat/utils.c          # 通用工具
libavformat/demux.c          # 解封装核心
libavformat/mux.c            # 封装核心

# Demuxers
libavformat/mov.c            # MP4/MOV
libavformat/mpegts.c         # MPEG-TS
libavformat/matroskadec.c    # MKV
libavformat/flvdec.c         # FLV
libavformat/avi.c            # AVI
libavformat/wav.c            # WAV

# Muxers
libavformat/movenc.c         # MP4/MOV 封装
libavformat/mpegtsenc.c      # MPEG-TS 封装
libavformat/matroskaenc.c    # MKV 封装
```

### 协议实现
```
libavformat/protocols.c      # 协议注册
libavformat/file.c           # file://
libavformat/http.c           # http://
libavformat/rtmpproto.c      # rtmp://
libavformat/rtsp.c           # rtsp://
libavformat/tcp.c            # tcp://
libavformat/udp.c            # udp://
```

### 网络相关
```
libavformat/network.h        # 网络抽象
libavformat/httpauth.c       # HTTP 认证
libavformat/urldecode.c      # URL 解码
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libavformat 模块文档
- 🌐 整理协议和格式处理
- 🔧 添加常见使用模式
- ✨ 记录时间戳和元数据处理

---

*此模块处理几乎所有已知的多媒体容器格式。*
