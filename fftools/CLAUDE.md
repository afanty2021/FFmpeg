# fftools - FFmpeg 命令行工具

[根目录](../CLAUDE.md) > **fftools**

> 最后更新：2026-01-17 08:46:49

## 模块职责

fftools 包含 FFmpeg 的三个主要命令行工具，是用户与 FFmpeg 库交互的主要方式。

### 包含工具
- **ffmpeg**：音视频转码工具（最常用）
- **ffplay**：基于 SDL 的简易播放器
- **ffprobe**：媒体文件分析工具

## 工具详解

### 1. ffmpeg（转码工具）

#### 功能
- 格式转换（MKV → MP4、AVI → MP4 等）
- 编解码器转换（H.264 → H.265、AAC → MP3 等）
- 视频处理（缩放、裁剪、旋转、水印）
- 音频处理（混音、音量调整、提取）
- 流媒体推拉流（RTMP、HLS、DASH）

#### 典型用法
```bash
# 基本转码
ffmpeg -i input.mp4 output.mkv

# 指定编解码器
ffmpeg -i input.avi -c:v libx264 -c:a aac output.mp4

# 视频缩放
ffmpeg -i input.mp4 -vf scale=1280:720 output.mp4

# 提取音频
ffmpeg -i input.mp4 -vn -c:a copy output.aac

# 提取视频
ffmpeg -i input.mp4 -an -c:v copy output.mp4

# 码率控制
ffmpeg -i input.mp4 -b:v 2M -maxrate 2M -bufsize 4M output.mp4

# 截取片段
ffmpeg -i input.mp4 -ss 00:01:00 -t 00:00:30 -c copy output.mp4

# 推流到 RTMP
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://server/live/stream
```

#### 核心选项
```bash
# 输入选项
-i <file>          # 输入文件
-ss <time>         # 起始时间
-t <duration>      # 持续时间
-to <time>         # 结束时间

# 视频选项
-c:v <codec>       # 视频编解码器
-b:v <bitrate>     # 视频比特率
-r <fps>           # 帧率
-s <size>          # 尺寸（WxH）
-aspect <ratio>    # 宽高比
-vf <filter>       # 视频滤镜
-pix_fmt <fmt>     # 像素格式

# 音频选项
-c:a <codec>       # 音频编解码器
-b:a <bitrate>     # 音频比特率
-ar <rate>         # 采样率
-ac <channels>     # 声道数
-af <filter>       # 音频滤镜

# 输出选项
-f <format>        # 强制格式
-threads <count>   # 线程数
-preset <preset>   # 编码预设（ultrafast~veryslow）
-crf <quality>     # 恒定质量（0-51）

# 其他
-y                 # 覆盖输出
-n                 # 不覆盖
-v <level>         # 日志级别
-stats             # 显示统计
```

### 2. ffplay（播放器）

#### 功能
- 播放几乎所有音视频格式
- 实时滤镜应用
- 屏幕截图
- 音频可视化

#### 典型用法
```bash
# 基本播放
ffplay input.mp4

# 无音频播放
ffplay -an input.mp4

# 循环播放
ffplay -loop 0 input.mp4

# 全屏播放
ffplay -fs input.mp4

# 应用滤镜
ffplay -vf "eq=brightness=0.1" input.mp4
ffplay -af "volume=1.5" input.mp4

# 截图（按 s）
ffplay -window_title "My Video" input.mp4
```

#### 快捷键
```
q, ESC          # 退出
f               # 全屏切换
p, SPACE        # 暂停/播放
→ / ←           # 前进/后退 10 秒
↑ / ↓           # 前进/后退 1 分钟
+ / -           # 音量 +/- 0.1
w               # 切换音频流
s               # 切换字幕流
c               # 切换节目
t               # 切换轨道
```

### 3. ffprobe（分析工具）

#### 功能
- 查看文件信息（格式、流、元数据）
- 分析包、帧、章节
- JSON/XML/CSV 输出
- 平面格式输出（脚本友好）

#### 典型用法
```bash
# 基本信息查看
ffprobe input.mp4

# 显示格式
ffprobe -show_format input.mp4

# 显示流信息
ffprobe -show_streams input.mp4

# 显示包信息
ffprobe -show_packets input.mp4

# JSON 输出
ffprobe -print_format json input.mp4

# 只显示视频流
ffprobe -select_streams v -show_streams input.mp4

# 查看特定信息
ffprobe -show_entries stream=codec_name,width,height input.mp4

# 计算帧数
ffprobe -v error -select_streams v:0 -count_frames \
  -show_entries stream=nb_read_frames -of default=nokey=1:noprint_wrappers=1 input.mp4

# 查找持续时间
ffprobe -v error -show_entries format=duration \
  -of default=nokey=1:noprint_wrappers=1 input.mp4
```

## 代码结构

### 文件组织
```
fftools/
├── ffmpeg.c              # ffmpeg 主程序
├── ffmpeg.h              # 公共头文件
├── ffmpeg_dec.c          # 解码逻辑
├── ffmpeg_enc.c          # 编码逻辑
├── ffmpeg_demux.c        # 解封装逻辑
├── ffmpeg_mux.c          # 封装逻辑
├── ffmpeg_filter.c       # 滤镜逻辑
├── ffmpeg_hw.c           # 硬件加速
├── ffmpeg_opt.c          # 选项处理
├── ffmpeg_sched.c        # 调度器
├── ffplay.c              # ffplay 主程序
├── ffprobe.c             # ffprobe 主程序
├── cmdutils.c            # 命令行工具
├── cmdutils.h
├── opt_common.c          # 通用选项
├── opt_common.h
├── thread_queue.c        # 线程队列
├── thread_queue.h
├── sync_queue.c          # 同步队列
├── sync_queue.h
└── resources/            # 图形资源（Windows）
```

### 核心流程（ffmpeg）
```c
// 主程序简化流程
main()
  └─→ parse_options()          // 解析命令行
  └─→ open_input_file()        // 打开输入
      └─→ avformat_open_input()
      └─→ avformat_find_stream_info()
  └─→ init_input_filter()      // 初始化滤镜图
  └─→ open_output_file()       // 打开输出
      └─→ avformat_alloc_output_context()
      └→ avformat_new_stream()
  └─→ transcode()              // 转码循环
      └─→ transcode_step()
          ├─→ receive_frame()  // 接收/解码
          ├─→ filter_frame()   // 滤镜处理
          └─→ encode_frame()   // 编码/发送
```

## 配置与依赖

### 编译配置
```bash
# 启用所有工具
--enable-ffmpeg
--enable-ffplay
--enable-ffprobe

# ffplay 需要 SDL
--enable-sdl
--disable-sdl2  # 使用 SDL 1.2

# 静态链接
--enable-static --disable-shared
```

### 依赖关系
- **ffmpeg**：libavutil、libavcodec、libavformat、libavfilter、libswscale、libswresample
- **ffplay**：上述 + SDL（音频/视频输出）
- **ffprobe**：libavutil、libavcodec、libavformat

## 常见使用场景

### 场景 1：视频压缩
```bash
ffmpeg -i input.mp4 -c:v libx264 -crf 23 -preset slow -c:a aac -b:a 128k output.mp4
```

### 场景 2：提取字幕
```bash
ffmpeg -i input.mkv -map 0:s:0 subs.srt
```

### 场景 3：合并视频
```bash
ffmpeg -f concat -safe 0 -i filelist.txt -c copy output.mp4
# filelist.txt:
# file '/path/to/video1.mp4'
# file '/path/to/video2.mp4'
```

### 场景 4：添加水印
```bash
ffmpeg -i input.mp4 -i watermark.png -filter_complex \
  "overlay=10:10" output.mp4
```

### 场景 5：录制屏幕
```bash
# macOS
ffmpeg -f avfoundation -i "1:0" output.mp4

# Linux (X11)
ffmpeg -f x11grab -i :0.0 output.mp4

# Windows (gdigrab)
ffmpeg -f gdigrab -i desktop output.mp4
```

### 场景 6：HLS 流化
```bash
ffmpeg -i input.mp4 -c:v libx264 -c:a aac -f hls -hls_time 10 \
  -hls_list_size 0 output.m3u8
```

## 测试与调试

### 调试选项
```bash
# 显示日志
ffmpeg -v debug -i input.mp4 output.mp4

# 显示统计
ffmpeg -stats -i input.mp4 output.mp4

# 保留中间文件（调试滤镜）
ffmpeg -y -i input.mp4 -vf scale=1280:720 -c:v rawvideo output.yuv
```

### 性能分析
```bash
# 测量速度
time ffmpeg -i input.mp4 output.mp4

# 显示性能统计
ffmpeg -v info -i input.mp4 output.mp4 2>&1 | grep "frame=" | wc -l

# 硬件加速
ffmpeg -hwaccel cuda -i input.mp4 output.mp4
```

## 常见问题（FAQ）

### Q: 如何指定多个输入？
A:
```bash
ffmpeg -i input1.mp4 -i input2.mp4 -filter_complex \
  "[0:v][1:v]hstack[out]" -map "[out]" output.mp4
```

### Q: 如何保持质量？
A:
```bash
# 使用 crf（恒定质量）
ffmpeg -i input.mp4 -c:v libx264 -crf 18 -preset slow output.mp4

# 禁用二次编码
ffmpeg -i input.mp4 -qscale:v 2 output.mp4
```

### Q: 如何实时转码？
A:
```bash
# 使用 -re（实时输入）
ffmpeg -re -i input.mp4 -c copy -f flv rtmp://server/live

# 使用 UDP（低延迟）
ffmpeg -i udp://@:1234 -c copy output.mp4
```

### Q: 如何处理音频同步？
A:
```bash
# 使用 aresample
ffmpeg -i input.mp4 -af "aresample=async=1" output.mp4

# 使用 video sync
ffmpeg -i input.mp4 -vsync vfr -r 30 output.mp4
```

## 相关文件清单

### 主要源文件
```
fftools/ffmpeg.c            # ffmpeg 主程序（~4000 行）
fftools/ffmpeg_dec.c        # 解码模块
fftools/ffmpeg_enc.c        # 编码模块
fftools/ffmpeg_filter.c     # 滤镜处理
fftools/ffmpeg_hw.c         # 硬件加速
fftools/ffmpeg_opt.c        # 选项解析
fftools/ffplay.c            # ffplay 主程序（~3000 行）
fftools/ffprobe.c           # ffprobe 主程序（~1000 行）
fftools/cmdutils.c          # 通用工具
```

### 辅助文件
```
fftools/sync_queue.c        # 音视频同步队列
fftools/thread_queue.c      # 线程安全队列
fftools/opt_common.c        # 通用选项
```

### 资源文件
```
fftools/resources/          # 图标、版本信息（Windows）
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 fftools 模块文档
- 🎬 整理三个工具的用法
- 💡 添加常见使用场景
- 🔧 记录调试技巧

---

*这些工具是 FFmpeg 生态的用户接口，被广泛用于媒体处理工作流。*
