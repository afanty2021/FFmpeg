# libswscale - FFmpeg 图像缩放库

[根目录](../CLAUDE.md) > **libswscale**

> 最后更新：2026-01-17 08:46:49

## 模块职责

libswscale 提供高性能的图像缩放、颜色空间转换和像素格式转换功能。

### 核心功能
- **图像缩放**：任意尺寸缩放、质量可控
- **颜色转换**：YUV ↔ RGB、各种像素格式
- **色彩空间转换**：BT.601、BT.709、BT.2020
- **Gamma 校正**：线性/感知转换
- **平台优化**：SIMD 加速（SSE2、AVX2、NEON 等）

## 入口与启动

### 主要头文件
```c
#include <libswscale/swscale.h>
```

### 典型使用流程
```c
// 1. 创建缩放上下文
struct SwsContext *sws_ctx = sws_getContext(
    src_width, src_height, src_pix_fmt,   // 输入
    dst_width, dst_height, dst_pix_fmt,   // 输出
    SWS_BILINEAR,                         // 算法
    NULL, NULL, NULL                      // 滤镜参数
);

// 2. 执行转换
sws_scale(sws_ctx,
    (const uint8_t *const *)src_data, src_linesize, 0, src_height,
    dst_data, dst_linesize);

// 3. 释放上下文
sws_freeContext(sws_ctx);
```

## 对外接口

### 核心 API

#### 上下文管理
```c
// 创建和释放
struct SwsContext *sws_getContext(
    int srcW, int srcH, enum AVPixelFormat srcFormat,
    int dstW, int dstH, enum AVPixelFormat dstFormat,
    int flags, SwsFilter *srcFilter, SwsFilter *dstFilter,
    const double *param
);
void sws_freeContext(struct SwsContext *swsContext);

// 帧转换（推荐）
int sws_scale_frame(struct SwsContext *c, AVFrame *dst, const AVFrame *src);
```

#### 缩放算法
```c
// 速度与质量权衡
SWS_FAST_BILINEAR     // 最快，质量最低
SWS_BILINEAR          // 双线性（默认）
SWS_BICUBIC           // 双三次
SWS_X                 // Lanczos
SWS_POINT             // 最近邻
SWS_AREA              // 面积平均
SWS_BICUBLIN          // 双三次 + 双线性
SWS_GAUSS             // 高斯
SWS_SINC              // Sinc
SWS_LANCZOS           // Lanczos（质量最好）
SWS_SPLINE            // Spline

// 推荐组合
SWS_BILINEAR | SWS_ACCURATE_RND
```

#### 特殊标志
```c
// 准确舍入
SWS_ACCURATE_RND

// 全色范围输入
SWS_FULL_CHR_IN

// 全色范围输出
SWS_FULL_CHR_OUT

// 打印信息
SWS_PRINT_INFO
```

### 常见转换场景

#### 基本缩放
```c
// 缩小 1920x1080 → 1280x720
struct SwsContext *sws = sws_getContext(
    1920, 1080, AV_PIX_FMT_YUV420P,
    1280, 720, AV_PIX_FMT_YUV420P,
    SWS_BILINEAR, NULL, NULL, NULL);
```

#### 格式转换
```c
// YUV420P → RGB24
struct SwsContext *sws = sws_getContext(
    1920, 1080, AV_PIX_FMT_YUV420P,
    1920, 1080, AV_PIX_FMT_RGB24,
    SWS_FAST_BILINEAR, NULL, NULL, NULL);
```

#### 综合转换
```c
// YUV420P 1920x1080 → RGB24 1280x720
struct SwsContext *sws = sws_getContext(
    1920, 1080, AV_PIX_FMT_YUV420P,
    1280, 720, AV_PIX_FMT_RGB24,
    SWS_BICUBIC, NULL, NULL, NULL);
```

#### 奇数尺寸处理
```c
// 确保 YUV420P 尺寸为偶数
int dst_width = src_width & ~1;  // 向下取偶
int dst_height = src_height & ~1;
```

## 高级功能

### 颜色空间转换
```c
// 设置颜色空间特性
sws_setColorspaceDetails(sws,
    inv_table, srcRange,     // 输入
    table, dstRange,         // 输出
    brightness,              // 亮度 (0-1)
    contrast,                // 对比度 (0-1)
    saturation               // 饱和度 (0-1)
);

// 常见颜色空间
SWS_CS_ITU601       // BT.601 (SD)
SWS_CS_ITU709       // BT.709 (HD)
SWS_CS_ITU2020      // BT.2020 (UHD)
```

### 范围转换
```c
// 限制范围 → 全范围
sws_setColorspaceDetails(sws,
    sws_getCoefficients(SWS_CS_ITU709), 0,  // 输入：限制
    sws_getCoefficients(SWS_CS_ITU709), 1,  // 输出：全范围
    0, 1, 1);
```

### Alpha 混合
```c
// 支持透明度
struct SwsContext *sws = sws_getContext(
    src_w, src_h, AV_PIX_FMT_RGBA,
    dst_w, dst_h, AV_PIX_FMT_RGBA,
    SWS_BICUBIC, NULL, NULL, NULL);

// 使用 alpha 通道混合
sws_scale(sws, ...);
```

### SIMD 优化
```c
// 自动检测并使用 SIMD
// x86: MMX、SSE2、SSE4、AVX2、AVX-512
// ARM: NEON
// PPC: AltiVec、VSX
// RISC-V: RVV

// 强制使用特定优化
// 通过环境变量或编译时选项
```

## 性能优化

### 缓存行对齐
```c
// 确保缓冲区对齐（64 字节）
av_frame_get_buffer(frame, 64);
```

### 多线程处理
```c
// 切片处理
int slice_height = src_height / 4;
for (int i = 0; i < 4; i++) {
    sws_scale(sws,
        src_data, src_linesize,
        i * slice_height, slice_height,
        dst_data, dst_linesize);
}
```

### 上下文复用
```c
// 复用上下文（避免重复初始化）
static struct SwsContext *sws_ctx = NULL;
if (!sws_ctx || need_reinit) {
    sws_freeContext(sws_ctx);
    sws_ctx = sws_getContext(...);
}
```

### 质量优化
```c
// 使用高质量算法
struct SwsContext *sws = sws_getContext(..., SWS_LANCZOS, ...);

// 自定义滤镜参数
double param[2] = {2.0, 0.0};  // Lanczos 参数
sws_getContext(..., SWS_LANCZOS, NULL, NULL, param);
```

## 关键依赖与配置

### 编译配置
```bash
# 启用 libswscale
--enable-swscale

# 启用特定优化
--enable-swscale-x86     # x86 SIMD
--enable-swscale-alpha    # Alpha 支持
```

### 依赖关系
- **依赖**：libavutil
- **无外部依赖**：完全独立实现

## 数据模型

### 像素格式
```c
// YUV 格式（最常用）
AV_PIX_FMT_YUV420P       // YUV 4:2:0 平面
AV_PIX_FMT_YUV422P       // YUV 4:2:2 平面
AV_PIX_FMT_YUV444P       // YUV 4:4:4 平面
AV_PIX_FMT_NV12          // YUV 4:2:0 半平面

// RGB 格式
AV_PIX_FMT_RGB24         // RGB 打包
AV_PIX_FMT_RGBA          # RGBA 打包
AV_PIX_FMT_ARGB          // ARGB 打包

// 硬件格式
AV_PIX_FMT_CUDA          // CUDA
AV_PIX_FMT_VAAPI         # VAAPI
AV_PIX_FMT_D3D11         # Direct3D 11
```

### 布局和步长
```c
// YUV420P 布局
// data[0]: Y 平面 (linesize[0] * height)
// data[1]: U 平面 (linesize[1] * height/2)
// data[2]: V 平面 (linesize[2] * height/2)

// RGB24 布局
// data[0]: RGB 打包 (linesize[0] * height * 3)
```

## 常见问题（FAQ）

### Q: 如何处理奇数尺寸？
A:
```c
// 确保 YUV420P 的宽高为偶数
dst_width = (src_width + 1) & ~1;
dst_height = (src_height + 1) & ~1;
```

### Q: 为什么颜色看起来不对？
A: 检查颜色空间和范围：
```c
// 使用正确的颜色空间
int coefs = SWS_CS_ITU709;  // HD 视频
sws_setColorspaceDetails(sws,
    sws_getCoefficients(coefs), srcRange,
    sws_getCoefficients(coefs), dstRange,
    0, 1, 1);
```

### Q: 如何提高转换速度？
A:
```c
// 使用快速算法
sws_getContext(..., SWS_FAST_BILINEAR, ...);

// 确保数据对齐
av_frame_get_buffer(frame, 64);

// 复用上下文
static struct SwsContext *sws = NULL;
```

### Q: 如何实现高质量缩放？
A:
```c
// 使用 Lanczos
double param[2] = {3.0, 0.0};  // Lanczos 窗口
sws_getContext(..., SWS_LANCZOS, NULL, NULL, param);

// 或使用 Bicubic
sws_getContext(..., SWS_BICUBIC | SWS_ACCURATE_RND, ...);
```

### Q: 如何处理 Alpha 通道？
A:
```c
// 使用支持 Alpha 的格式
struct SwsContext *sws = sws_getContext(
    src_w, src_h, AV_PIX_FMT_RGBA,
    dst_w, dst_h, AV_PIX_FMT_RGBA,
    SWS_BICUBIC, NULL, NULL, NULL);

// 或单独处理 Alpha
sws_scale(sws_alpha, ...);  // 只转换 RGB
sws_scale(sws_alpha, ...);  // 转换 Alpha
```

## 相关文件清单

### 头文件（公共 API）
```
libswscale/swscale.h        # 主入口
```

### 源文件（内部实现）
```
libswscale/swscale.c        # 核心实现
libswscale/utils.c          # 工具函数
libswscale/options.c        # AVOption 处理
libswscale/yuv2rgb.c        # YUV → RGB
libswscale/rgb2yuv.c        # RGB → YUV
```

### 缩放算法
```
libswscale/scale.c          # 缩放核心
libswscale/hscale.c         # 水平缩放
libswscale/vscale.c         # 垂直缩放
libswscale/hscale_fast_bilinear.c  # 快速双线性
```

### 颜色转换
```
libswscale/input.c          # 输入格式处理
libswscale/output.c         # 输出格式处理
libswscale/csputils.c       # 颜色空间工具
libswscale/gamma.c          # Gamma 校正
```

### 优化目录
```
libswscale/x86/             # x86 SIMD 优化
libswscale/aarch64/         # ARM64 NEON
libswscale/arm/             # ARM NEON
libswscale/ppc/             # PowerPC AltiVec
libswscale/riscv/           # RISC-V RVV
```

### 模板和辅助
```
libswscale/rgb2rgb.c        # RGB ↔ RGB
libswscale/rgb2rgb_template.c
libswscale/slice.c          # 切片处理
```

## 变更记录 (Changelog)

### 2026-01-17 08:46:49 - 初始化文档
- 📝 创建 libswscale 模块文档
- 🎨 整理像素格式转换场景
- ⚡ 记录性能优化技巧
- ✨ 添加常见问题解答

---

*此模块是视频处理的核心，所有视频缩放和颜色转换都需要它。*
