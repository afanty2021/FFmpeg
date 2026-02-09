# libavdevice - FFmpeg 设备库

[根目录](../CLAUDE.md) > **libavdevice**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libavdevice 提供对输入输出设备的抽象，支持捕获和播放音视频设备。

### 核心功能
- **输入设备**：摄像头、麦克风、屏幕捕获、音频卡
- **输出设备**：音频输出、视频显示
- **特殊设备**：滤波器作为设备（lavfi）

## 入口与启动

### 主要头文件
```c
#include <libavdevice/avdevice.h>
```

### 典型使用流程
```c
// 1. 注册所有设备（FFmpeg 4.0+ 自动注册）
avdevice_register_all();

// 2. 查找输入格式
AVInputFormat *iformat = av_find_input_format("x11grab");

// 3. 打开设备
AVFormatContext *fmt_ctx = NULL;
AVDictionary *opts = NULL;
av_dict_set(&opts, "framerate", "30", 0);
av_dict_set(&opts, "video_size", "1920x1080", 0);
avformat_open_input(&fmt_ctx, ":0.0", iformat, &opts);

// 4. 读取帧（与文件相同）
while (av_read_frame(fmt_ctx, &pkt) >= 0) {
    // 处理帧
}

// 5. 关闭
avformat_close_input(&fmt_ctx);
```

## 对外接口

### 支持的输入设备

#### Linux
```bash
# X11 屏幕捕获
x11grab              # X11 屏幕
fbdev                # framebuffer 设备
v4l2                 # Video4Linux2 摄像头
alsa                 # ALSA 音频
oss                  # OSS 音频
jack                 # JACK 音频
pulse                # PulseAudio
sndio                # OpenBSD sndio
```

#### macOS
```bash
# AVFoundation
avfoundation         # 摄像头/屏幕/音频
```

#### Windows
```bash
# DirectShow
dshow                # DirectShow 设备
vfwcap               # Video for Windows
gdigrab              # GDI 屏幕捕获
```

#### 跨平台
```bash
# 滤镜作为输入
lavfi                # 使用 libavfilter 滤镜生成数据
```

### 支持的输出设备

#### Linux
```bash
alsa                 # ALSA 音频输出
oss                  # OSS 音频输出
pulse                # PulseAudio 输出
sndio                # sndio 输出
fbdev                # framebuffer 输出
 xv                   # X11 视频输出
```

#### macOS
```bash
# CoreAudio
coreaudio            # 音频输出
```

#### Windows
```bash
# DirectSound
dsound               # DirectSound 音频
```

## 常见使用示例

### 屏幕捕获（Linux）
```bash
ffmpeg -f x11grab -framerate 30 -video_size 1920x1080 \
  -i :0.0 output.mp4
```

### 摄像头捕获（Linux）
```bash
ffmpeg -f v4l2 -i /dev/video0 output.mp4
```

### 摄像头捕获（macOS）
```bash
# 列出设备
ffmpeg -f avfoundation -list_devices true -i ""

# 捕获
ffmpeg -f avfoundation -i "0:0" output.mp4
```

### 屏幕捕获（Windows）
```bash
ffmpeg -f gdigrab -framerate 30 -i desktop output.mp4
```

### 音频捕获
```bash
# ALSA (Linux)
ffmpeg -f alsa -i hw:0 output.wav

# PulseAudio
ffmpeg -f pulse -i default output.wav

# CoreAudio (macOS)
ffmpeg -f coreaudio -i ":0" output.wav

# DirectSound (Windows)
ffmpeg -f dsound -i audio=0 output.wav
```

### 滤镜作为输入
```bash
# 生成测试视频
ffmpeg -f lavfi -i testsrc=duration=10:size=1920x1080:rate=30 output.mp4

# 生成音频
ffmpeg -f lavfi -i sine=frequency=1000:duration=5 output.wav

# 合成
ffmpeg -f lavfi -i "testsrc=s=320x256:d=10" -f lavfi -i "sine=f=1000:d=10" output.mp4
```

## 关键依赖与配置

### 编译配置
```bash
# 启用所有设备
--enable-indevs
--enable-outdevs

# 启用特定设备
--enable-indev=x11grab
--enable-indev=v4l2
--enable-outdev=alsa

# 禁用特定设备
--disable-indev=jack
```

### 依赖关系
- **依赖**：libavutil、libavformat
- **可选依赖**：
  - Linux：libasound (ALSA)、libpulse (PulseAudio)
  - macOS：CoreAudio 框架
  - Windows：DirectShow、DirectSound

## 设备特定选项

### x11grab（Linux X11）
```bash
# 显示器编号
ffmpeg -f x11grab -i :0.0 ...

# 偏移和尺寸
ffmpeg -f x11grab -video_size 1920x1080 -i :0.0+10,20 ...

# 跟踪鼠标
ffmpeg -f x11grab -follow_mouse centered -i :0.0 ...
```

### v4l2（Linux 摄像头）
```bash
# 列出格式
ffmpeg -f v4l2 -list_formats all -i /dev/video0

# 设置输入格式
ffmpeg -f v4l2 -input_format mjpeg -i /dev/video0 ...

# 帧率和尺寸
ffmpeg -f v4l2 -framerate 30 -video_size 1920x1080 -i /dev/video0 ...
```

### avfoundation（macOS）
```bash
# 列出设备
ffmpeg -f avfoundation -list_devices true -i ""

# 捕获特定设备
ffmpeg -f avfoundation -i "0:0" ...  # 视频:音频
ffmpeg -f avfoundation -video_device_index 0 -audio_device_index 0 ...
```

### gdigrab（Windows）
```bash
# 捕获整个桌面
ffmpeg -f gdigrab -i desktop ...

# 捕获窗口
ffmpeg -f gdigrab -i title="Window Title" ...

# 显示区域
ffmpeg -f gdigrab -offset_x 10 -offset_y 20 -video_size 1920x1080 -i desktop ...
```

## 常见问题（FAQ）

### Q: 如何列出可用设备？
A:
```bash
# Linux (v4l2)
ffmpeg -f v4l2 -list_formats all -i /dev/video0

# macOS
ffmpeg -f avfoundation -list_devices true -i ""

# Windows (dshow)
ffmpeg -f dshow -list_devices true -i dummy
```

### Q: 如何同时捕获屏幕和音频？
A:
```bash
ffmpeg -f x11grab -i :0.0 -f alsa -i hw:0 \
  -c:v libx264 -c:a aac output.mp4
```

### Q: 如何处理设备权限？
A:
```bash
# 将用户添加到相关组
sudo usermod -a -G video $USER  # v4l2
sudo usermod -a -G audio $USER  # ALSA

# 设置设备权限
sudo chmod 666 /dev/video0
```

### Q: 如何实现低延迟捕获？
A:
```bash
# 设置缓冲区大小
ffmpeg -f alsa -buffer_size 128 -i hw:0 ...

# 使用预设
ffmpeg -preset ultrarealtime ...
```

## 相关文件清单

### 源文件（设备实现）
```
libavdevice/alldevices.c     # 设备注册
libavdevice/avdevice.c       # 核心函数

# Linux
libavdevice/xcbgrab.c        # X11 屏幕捕获
libavdevice/v4l2.c           # Video4Linux2
libavdevice/alsa.c           # ALSA 音频
libavdevice/oss.c            # OSS 音频
libavdevice/jack.c           # JACK 音频
libavdevice/pulse_audio_dec.c # PulseAudio 输入
libavdevice/pulse_audio_enc.c # PulseAudio 输出

# macOS
libavdevice/avfoundation.m   # AVFoundation
libavdevice/openal-dec.c     # OpenAL 输入

# Windows
libavdevice/dshow.c          # DirectShow
libavdevice/vfwcap.c         # Video for Windows
libavdevice/gdigrab.c        # GDI 屏幕捕获

# 跨平台
libavdevice/lavfi.c          # 滤镜作为设备
```

### 辅助文件
```
libavdevice/utils.c          # 工具函数
libavdevice/file_open.c      # 文件操作
libavdevice/timefilter.c     # 时间滤波器
libavdevice/version.c        # 版本信息
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libavdevice 模块文档
- 🎥 整理捕获设备列表
- 💻 添加平台特定示例
- 🔧 记录常见问题

---

*此模块让 FFmpeg 能够直接访问硬件设备，用于实时捕获和播放。*
