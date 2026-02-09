# libavfilter - FFmpeg 滤镜库

[根目录](../CLAUDE.md) > **libavfilter**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libavfilter 提供基于有向图（DAG）的音视频滤镜处理框架，支持复杂的滤镜链组合。

### 核心功能
- **视频滤镜**：缩放、裁剪、旋转、叠加、颜色调整
- **音频滤镜**：混音、音量、重采样、降噪
- **源滤镜**：测试视频、音频生成
- **Sink 滤镜**：输出到文件、设备
- **滤镜图**：复杂的多分支滤镜组合

## 入口与启动

### 主要头文件
```c
#include <libavfilter/avfilter.h>    // 核心滤镜 API
#include <libavfilter/buffersink.h>  // Sink 滤镜
#include <libavfilter/buffersrc.h>   // Source 滤镜
```

### 典型使用流程（简单滤镜）
```c
// 1. 创建滤镜图
AVFilterGraph *graph = avfilter_graph_alloc();

// 2. 创建 source（输入）
AVFilterContext *src_ctx;
avfilter_graph_create_filter(&src_ctx,
    avfilter_get_by_name("buffer"), "in",
    "video_size=1920x1080:pix_fmt=0:time_base=1/30:pixel_aspect=1/1",
    NULL, graph);

// 3. 创建 sink（输出）
AVFilterContext *sink_ctx;
avfilter_graph_create_filter(&sink_ctx,
    avfilter_get_by_name("buffersink"), "out", NULL, NULL, graph);

// 4. 创建滤镜
AVFilterContext *filter_ctx;
avfilter_graph_create_filter(&filter_ctx,
    avfilter_get_by_name("scale"), "scale",
    "1280:720", NULL, graph);

// 5. 连接滤镜链
avfilter_link(src_ctx, 0, filter_ctx, 0);
avfilter_link(filter_ctx, 0, sink_ctx, 0);

// 6. 配置图
avfilter_graph_config(graph, NULL);

// 7. 发送帧、接收帧
av_buffersrc_add_frame(src_ctx, frame);
av_buffersink_get_frame(sink_ctx, filt_frame);
```

### 字符串滤镜图（推荐）
```c
// 使用字符串描述滤镜图
const char *filters = "scale=1280:720,format=yuv420p";
AVFilterGraph *graph = avfilter_graph_alloc();
avfilter_graph_parse_ptr(graph, filters, &inputs, &outputs, NULL);
avfilter_graph_config(graph, NULL);
```

## 对外接口

### 核心 API

#### 滤镜查找
```c
// 通过名称查找
const AVFilter *filter = avfilter_get_by_name("scale");

// 遍历所有滤镜
void *iter = NULL;
while ((filter = avfilter_next(iter))) {
    printf("%s: %s\n", filter->name, filter->description);
}
```

#### 帧处理
```c
// 发送帧到滤镜图
av_buffersrc_add_frame_flags(src_ctx, frame, AV_BUFFERSRC_FLAG_KEEP_REF);

// 从滤镜图接收帧
int ret = av_buffersink_get_frame(sink_ctx, out_frame);
if (ret == AVERROR(EAGAIN)) {
    // 需要更多输入
} else if (ret == AVERROR_EOF) {
    // 滤镜图结束
}
```

### 主要数据结构

#### AVFilterContext（滤镜实例）
```c
typedef struct AVFilterContext {
    const AVFilter *filter;      // 滤镜定义
    AVFilterPad *input_pads;     // 输入 pad
    AVFilterPad *output_pads;    // 输出 pad
    AVFilterLink **inputs;       // 输入链接
    AVFilterLink **outputs;      // 输出链接
    char *name;                  // 实例名
    void *priv;                  // 私有数据
} AVFilterContext;
```

#### AVFilterGraph（滤镜图）
```c
typedef struct AVFilterGraph {
    AVFilterContext **filters;   // 滤镜数组
    unsigned nb_filters;         // 滤镜数量
    void *class;                 // AVClass
    AVFilterLink **sink_links;   // Sink 链接
} AVFilterGraph;
```

#### AVFilterLink（滤镜链接）
```c
typedef struct AVFilterLink {
    AVFilterContext *src;        // 源滤镜
    AVFilterContext *dst;        // 目标滤镜
    AVFilterPad *srcpad;         // 源 pad
    AVFilterPad *dstpad;         // 目标 pad
    enum AVMediaType type;       // 媒体类型
    int w, h;                    // 视频：宽高
    AVRational sample_aspect_ratio;  // SAR
    enum AVPixelFormat format;   // 视频格式
    int sample_rate;             // 音频：采样率
    AVChannelLayout ch_layout;   // 音频：声道布局
} AVFilterLink;
```

### 常用滤镜

#### 视频滤镜
```bash
# 缩放
scale=1280:720
scale=1280:-1  # 保持宽高比
scale=iw/2:ih/2  # 相对缩放

# 裁剪
crop=w:h:x:y
crop=1920:1080:0:0

# 旋转
rotate=angle  # 弧度
rotate=PI/4

# 叠叠
overlay=x:y
overlay=10:10

# 色彩调整
format=pix_fmt
format=yuv420p

# 文字
drawtext=text='Hello':x=10:y=10:fontsize=24

# 速度
setpts=PTS/2  # 2 倍速
setpts=PTS*2  # 0.5 倍速
```

#### 音频滤镜
```bash
# 音量
volume=0.5  # 50% 音量
volume=2.0  # 200% 音量

# 淡入淡出
afade=t=in:st=0:d=5  # 淡入
afade=t=out:st=95:d=5  # 淡出

# 混音
amix=inputs=2

# 速度
atempo=1.5  # 1.5 倍速
atempo=0.5  # 0.5 倍速
```

#### 复杂滤镜图
```bash
# 分支
split=2[main][tmp]
[tmp]crop=iw:ih/2:0:0,vflip[flip]
[main][flip]overlay=0:H/2

# 合并
[in0]split=2[a][b]
[a]scale=iw/2:ih/2[a]
[b]scale=iw*2:ih*2[b]
[a][b]hstack[out]
```

## 关键依赖与配置

### 编译配置
```bash
# 启用所有滤镜
--enable-filters

# 启用特定滤镜
--enable-filter=scale,crop,overlay

# 禁用特定滤镜
--disable-filter=drawtext  # 需要 libfreetype
```

### 依赖关系
- **依赖**：libavutil、libavcodec、libavformat
- **可选依赖**：
  - 文字：libfreetype、libfontconfig
  - 视频：libopencv
  - 音频：libsoxr、librubberband

## 数据模型

### 滤镜 Pad
```c
// 输入 pad（接收数据）
static const AVFilterPad inputs[] = {
    {
        .name = "default",
        .type = AVMEDIA_TYPE_VIDEO,
        .filter_frame = filter_frame,  // 回调函数
    },
};

// 输出 pad（发送数据）
static const AVFilterPad outputs[] = {
    {
        .name = "default",
        .type = AVMEDIA_TYPE_VIDEO,
    },
};
```

### 滤镜协商
```c
// 格式协商（自动）
AVFilterFormats *formats;
ff_formats_ref(ff_make_format_list(pix_fmts), &link->in_formats);
ff_formats_ref(ff_make_format_list(pix_fmts), &link->out_formats);

// 尺寸协商
AVFilterLink *link = src->outputs[0];
avfilter_get_video_buffer(link, w, h);
```

### 帧属性
```c
// 带时间戳的帧
frame->pts = pts;  // 显示时间戳
frame->duration = duration;  // 时长
frame->sample_aspect_ratio = sar;  // SAR
```

## 测试与质量

### 测试文件
- **位置**：`tests/fate/filter-video.mak`、`tests/fate/filter-audio.mak`
- **示例**：每个滤镜都有独立测试

### checkasm 测试
- **位置**：`tests/checkasm/vf_*.c`
- **覆盖**：滤镜核心函数的 SIMD 实现

### 常见问题（FAQ）

#### Q: 如何动态调整滤镜参数？
A: 使用 AVOption：
```c
av_opt_set(filter_ctx->priv, "width", "1920", 0);
av_opt_set(filter_ctx->priv, "height", "1080", 0);
```

#### Q: 如何处理多输入/多输出？
A: 使用复杂滤镜图：
```c
const char *filter_desc =
    "[in0]split=2[a][b]"
    "[in1]split=2[c][d]"
    "[a][c]hstack[top]"
    "[b][d]hstack[bottom]"
    "[top][bottom]vstack[out]";
```

#### Q: 如何实现实时滤镜？
A:
```c
// 使用 FIFO 缓冲
avfilter_graph_create_filter(&fifo_ctx,
    avfilter_get_by_name("fifo"), "fifo", NULL, NULL, graph);
```

#### Q: 滤镜图的性能如何优化？
A:
```c
// 启用线程
graph->nb_threads = 4;

// 使用零拷贝
av_buffersrc_add_frame_flags(src_ctx, frame,
    AV_BUFFERSRC_FLAG_PUSH | AV_BUFFERSRC_FLAG_KEEP_REF);
```

## 相关文件清单

### 头文件（公共 API）
```
libavfilter/avfilter.h       # 主入口
libavfilter/buffersrc.h      # Source 滤镜
libavfilter/buffersink.h     # Sink 滤镜
```

### 源文件（滤镜实现）
```
libavfilter/allfilters.c     # 滤镜注册
libavfilter/avfilter.c       # 核心框架
libavfilter/graphdump.c      # 调试输出

# 视频滤镜
libavfilter/vf_scale.c       # 缩放
libavfilter/vf_crop.c        # 裁剪
libavfilter/vf_overlay.c     # 叠叠
libavfilter/vf_rotate.c      # 旋转
libavfilter/vf_fade.c        # 淡入淡出
libavfilter/vf_drawtext.c    # 文字
libavfilter/vf_format.c      # 格式转换

# 音频滤镜
libavfilter/af_volume.c      # 音量
libavfilter/af_afade.c       # 淡入淡出
libavfilter/af_amix.c        # 混音
libavfilter/af_aresample.c   # 重采样

# 源和 Sink
libavfilter/src_movie.c      # 文件源
libavfilter/src_buffer.c     # Buffer 源
libavfilter/sink_buffer.c    # Buffer Sink
```

### 内部工具
```
libavfilter/formats.c        # 格式协商
libavfilter/video.c          # 视频工具
libavfilter/audio.c          # 音频工具
libavfilter/transform.c      # 几何变换
```

### 优化目录
```
libavfilter/x86/             # x86 SIMD 优化
libavfilter/aarch64/         # ARM64 优化
libavfilter/arm/             # ARM 优化
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libavfilter 模块文档
- 🎨 整理滤镜分类和使用
- 🔧 添加滤镜图示例
- ✨ 记录常见问题解答

---

*此模块提供了强大的音视频处理能力，支持几乎所有常见操作。*
