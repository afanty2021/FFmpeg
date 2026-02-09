# FFmpeg 编解码器与格式实现分析

> 最后更新：2026-01-17
> 文档类型：技术深度分析
> 覆盖范围：主流格式和编解码器实现模式

## 概述

本文档深入分析 FFmpeg 中主流格式和编解码器的实现细节，包括关键算法、数据结构、优化策略和性能技巧。

---

## 第一部分：主流封装格式实现

### 1. MP4/MOV 格式（libavformat/mov.c）

#### 文件结构
```c
// 核心数据结构
typedef struct MOVContext {
    AVFormatContext *fc;              // 格式上下文
    int time_scale;                   // 时间刻度
    int64_t duration;                 // 时长
    MOVStreamContext **streams;       // 流数组
    int nb_streams;                   // 流数量
    HEIFItem **heif_item;             // HEIF 项目
    int nb_heif_item;                 // HEIF 项目数
    int cur_item_id;                  // 当前项目 ID
    // ... 更多字段
} MOVContext;

typedef struct MOVStreamContext {
    int id;                           // 流 ID
    int time_scale;                   // 时间刻度
    int64_t duration;                 // 时长
    int sample_size;                  // 样本大小
    int keyframe_count;               // 关键帧计数
    int64_t *keyframe_times;          // 关键帧时间
    int64_t *keyframe_filepositions;  // 关键帧位置
    // ... 更多字段
} MOVStreamContext;

// Atom 解析表
typedef struct MOVParseTableEntry {
    uint32_t type;                    // Atom 类型
    int (*parse)(MOVContext *ctx, AVIOContext *pb, MOVAtom atom);
} MOVParseTableEntry;
```

#### 关键实现模式

**1. Atom 树解析**
```c
// MP4 文件是基于 Atom 的容器结构
// 典型 Atom 层级：
// ftyp (文件类型)
// moov (元数据容器)
//   - mvhd (电影头)
//   - trak (轨道)
//     - tkhd (轨道头)
//     - mdia (媒体)
//       - mdhd (媒体头)
//       - hdlr (处理器)
//       - minf (媒体信息)
//         - stbl (样本表)
//           - stsd (样本描述)
//           - stts (时间到样本)
//           - stsc (样本到块)
//           - stsz (块大小)
//           - stco (块偏移)
// mdat (媒体数据)

// 解析入口
static int mov_read_default(MOVContext *c, AVIOContext *pb, MOVAtom atom)
{
    // 递归解析子 Atom
    while (atom.size > 8) {
        uint32_t atom_type = avio_rb32(pb);
        int64_t atom_size = avio_rb32(pb);
        // 查找并调用对应的解析函数
    }
}
```

**2. 时间戳处理**
```c
// MP4 使用复杂的时间戳系统
// 时间单位：time_scale (每秒的 tick 数)
// 转换为 AVStream 的 time_base

// 样本时间计算
int64_t sample_time = sample_dts * av_rescale(
    1, st->time_base.den, st->time_base.num * time_scale
);

// 编辑列表（ELST）处理
// 支持空编辑（跳过）和时间偏移
```

**3. 索引与 Seek**
```c
// 使用 STSS (Sync Sample) 原子定位关键帧
// 使用 STCO (Chunk Offset) 原子定位数据块

// Seek 实现
static int mov_read_seek(AVFormatContext *s, int stream_index,
                        int64_t target_ts, int flags)
{
    // 1. 二分查找关键帧索引
    // 2. 定位到对应的 chunk
    // 3. 读取并解析样本
}
```

#### 元数据处理
```c
// 元数据 Atom
// - udta (用户数据)
//   - meta (元数据)
//     - ilst (信息列表)
//       //_covr (封面)
//       // ©nam (标题)
//       // ©ART (艺术家)

// 特殊元数据
static int mov_metadata_loci(MOVContext *c, AVIOContext *pb, unsigned len)
{
    // 3GPP TS 26.244 地理位置信息
    // 包含：经度、纬度、高度、地点名称
}

static int mov_read_covr(MOVContext *c, AVIOContext *pb, int type, int len)
{
    // 提取嵌入的封面图片
    // 支持：JPEG、PNG、BMP
}
```

#### HEIF/HEIC 支持
```c
// HEIF (High Efficiency Image File Format)
// 基于 MP4 容器的图像格式

typedef struct HEIFItem {
    unsigned int item_id;             // 项目 ID
    AVStream *st;                     // 关联的流
    // ... 更多字段
} HEIFItem;

// HEIF 项目管理
static HEIFItem *get_heif_item(MOVContext *c, unsigned id)
{
    // 按项目 ID 查找
    for (int i = 0; i < c->nb_heif_item; i++) {
        if (c->heif_item[i]->item_id == id)
            return c->heif_item[i];
    }
    return NULL;
}
```

---

### 2. MPEG-TS 格式（libavformat/mpegts.c）

#### 数据结构
```c
// MPEG-TS 包结构（188 字节）
// - Sync Byte (0x47) [1 byte]
// - Transport Error Indicator [1 bit]
// - Payload Unit Start Indicator [1 bit]
// - Transport Priority [1 bit]
// - PID [13 bits]
// - Scrambling Control [2 bits]
// - Adaptation Field Control [2 bits]
// - Continuity Counter [4 bits]
// - Adaptation Field (可选)
// - Payload (数据)

struct MpegTSContext {
    const AVClass *class;
    AVFormatContext *stream;          // 格式上下文
    int raw_packet_size;              // 包大小（通常 188）
    int64_t cur_pcr;                  // 当前 PCR
    int64_t pcr_incr;                 // PCR 增量

    // 程序映射
    unsigned int nb_prg;              // 程序数量
    struct Program *prg;              // 程序数组

    // PID 过滤器
    MpegTSFilter *pids[NB_PID_MAX];  // 每个 PID 一个过滤器

    // 缓冲池
    AVBufferPool* pools[32];
};

// PID 过滤器
struct MpegTSFilter {
    int pid;                          // PID 值
    enum MpegTSFilterType type;       // 过滤器类型
    union {
        MpegTSPESFilter pes_filter;   // PES 过滤器
        MpegTSSectionFilter section_filter;  // 节过滤器
    } u;
};

// 程序结构
struct Program {
    unsigned int id;                  // 程序 ID
    unsigned int nb_pids;             // PID 数量
    unsigned int pids[MAX_PIDS_PER_PROGRAM];  // PID 数组
    unsigned int nb_streams;          // 流数量
    struct Stream streams[MAX_STREAMS_PER_PROGRAM];
    int pmt_found;                    // 是否找到 PMT
};
```

#### 关键实现模式

**1. 包解析流程**
```c
// 1. 同步查找 (Sync Byte 0x47)
if (data[i] != 0x47)
    continue;

// 2. 读取包头
pid = AV_RB16(data + 1) & 0x1FFF;
cc  = data[3] & 0x0F;

// 3. 连续性检查
if (last_cc[pic] != (cc - 1) & 0x0F)
    // 丢包或错误

// 4. 分发到对应 PID 的过滤器
if (pids[pid])
    pids[pid]->cb(pids[pid], data, len);
```

**2. PSI (Program Specific Information) 处理**
```c
// PAT (Program Association Table)
// 关联 program_number 和 PMT PID

// PMT (Program Map Table)
// 关联 PID 和流类型（视频/音频/字幕）

// PCR (Program Clock Reference)
// 用于时钟同步

// 处理流程：
// 1. 解析 PAT，找到 PMT PID
// 2. 解析 PMT，找到各流的 PID
// 3. 为每个 PID 创建过滤器
// 4. 开始解析实际流数据
```

**3. PES (Packetized Elementary Stream) 处理**
```c
typedef struct MpegTSPESFilter {
    PESCallback *pes_cb;              // 回调函数
    void *opaque;
} MpegTSPESFilter;

// PES 包结构
// - Packet Start Code (0x000001)
// - Stream ID
// - PES Packet Length
// - Optional PES Header
// - Payload

// 组装完整帧
// 跨包缓冲直到收到完整帧
```

#### 时间戳处理
```c
// PCR (Program Clock Reference)
// 用于同步解码器时钟
int64_t pcr = (pcr_base * 300) + pcr_ext;

// PTS/DTS (Presentation/Decode Timestamp)
// 从 PES 头提取
int64_t pts = get_pts(pb);
int64_t dts = get_dts(pb);

// 时间基转换
// MPEG-TS 使用 90 kHz 时钟
// 转换为 AVStream 的 time_base
```

---

### 3. Matroska/WebM 格式（libavformat/matroska.c）

#### 数据结构
```c
// Matroska 使用 EBML (Extensible Binary Meta Language)
// 类似 XML 的二进制格式

// EBML 元素结构
// - Element ID (变长)
// - Element Size (变长)
// - Element Data

// Codec 映射表
const CodecTags ff_mkv_codec_tags[] = {
    {"V_MPEG4/ISO/AVC",  AV_CODEC_ID_H264},
    {"V_MPEGH/ISO/HEVC", AV_CODEC_ID_HEVC},
    {"V_VP8",            AV_CODEC_ID_VP8},
    {"V_VP9",            AV_CODEC_ID_VP9},
    {"V_AV1",            AV_CODEC_ID_AV1},
    {"A_AAC",            AV_CODEC_ID_AAC},
    {"A_VORBIS",         AV_CODEC_ID_VORBIS},
    {"A_OPUS",           AV_CODEC_ID_OPUS},
    // ... 更多映射
};

// WebM 限制的子集
const CodecTags ff_webm_codec_tags[] = {
    {"V_VP8",  AV_CODEC_ID_VP8},
    {"V_VP9",  AV_CODEC_ID_VP9},
    {"V_AV1",  AV_CODEC_ID_AV1},
    {"A_VORBIS", AV_CODEC_ID_VORBIS},
    {"A_OPUS",   AV_CODEC_ID_OPUS},
    // ... WebM 只支持 VP8/VP9/AV1 + Vorbis/Opus
};
```

#### 关键实现模式

**1. EBML 解析**
```c
// 读取变长整数
static uint64_t ebml_read_length(AVIOContext *pb)
{
    int num_bytes = 0;
    uint64_t len = 0;
    uint8_t b;

    do {
        b = avio_r8(pb);
        len = (len << 7) | (b & 0x7F);
        num_bytes++;
    } while (b & 0x80);

    return len;
}

// 读取 EBML 元素
static int ebml_read_elem(AVIOContext *pb, int *id, uint64_t *size)
{
    *id = ebml_read_id(pb);
    *size = ebml_read_length(pb);
    return 0;
}
```

**2. 关键 Matroska 元素**
```c
// Segment (容器)
//   - SeekHead (索引)
//   - Info (信息)
//     - TimecodeScale
//     - Duration
//   - Tracks (轨道)
//     - TrackEntry
//       - CodecID
//       - CodecPrivate (编解码器私有数据)
//   - Clusters (数据簇)
//     - Cluster
//       - Timecode
//       - SimpleBlock (或 BlockGroup)
//   - Cues (提示信息)
//   - Tags (元数据)

// SimpleBlock 结构
// - Track Number (变长)
// - Timecode (int16_t, 相对 Cluster)
// - Flags (关键帧等)
// - Data (实际帧数据)
```

**3. Seek 优化**
```c
// 使用 Cues 元素加速 Seek
// Cues 包含关键帧的位置和时间戳

// 使用 SeekHead 快速定位
// SeekHead 是其他顶层元素的索引

// 二分查找 Clusters
// 每个 Cluster 有时间戳
static int matroska_read_seek(AVFormatContext *s, int stream_index,
                              int64_t timestamp, int flags)
{
    // 1. 使用 Cues 查找最近的关键帧
    // 2. 定位到对应的 Cluster
    // 3. 读取 Block 直到目标时间戳
}
```

#### 元数据处理
```c
// 标签转换
const AVMetadataConv ff_mkv_metadata_conv[] = {
    {"LEAD_PERFORMER", "performer"},
    {"PART_NUMBER",    "track"},
    {NULL}
};

// WebM 特定标签
// - Title
// - Artist
// - Album
// - Date
// - Comment
```

---

### 4. FLV 格式（libavformat/flvdec.c）

#### 数据结构
```c
// FLV 包结构
// - Header [13 bytes]
//   - Signature ('FLV')
//   - Version
//   - Flags (音频/视频存在)
//   - Header Length
// - Body
//   - Tag [11 bytes header + data]
//     - Tag Type (音频/视频/脚本)
//     - Data Size
//     - Timestamp (3 bytes + 1 byte extension)
//     - Stream ID
//     - Tag Data
//   - Previous Tag Size [4 bytes]

typedef struct FLVContext {
    const AVClass *class;
    int trust_metadata;               // 信任元数据
    int trust_datasize;               // 信任数据大小
    int dump_full_metadata;           // 转储完整元数据
    int wrong_dts;                    // 错误的 DTS

    // 额外数据
    uint8_t *new_extradata[FLV_STREAM_TYPE_NB];
    int new_extradata_size[FLV_STREAM_TYPE_NB];

    // 索引验证
    struct {
        int64_t dts;
        int64_t pos;
    } validate_index[2];
    int validate_next;
    int validate_count;

    // 关键帧索引
    int last_keyframe_stream_index;
    int keyframe_count;
    int64_t *keyframe_times;
    int64_t *keyframe_filepositions;

    // 比特率计算
    int64_t video_bit_rate;
    int64_t audio_bit_rate;

    // 颜色信息
    FLVMetaVideoColor meta_color_info;
    enum FLVMetaColorInfoFlag meta_color_info_flag;
} FLVContext;

// AMF (Action Message Format) 数据
typedef struct amf_date {
    double milliseconds;               // 毫秒数
    int16_t timezone;                  // 时区
} amf_date;
```

#### 关键实现模式

**1. Tag 解析**
```c
// 读取 Tag 头
static int flv_read_packet(AVFormatContext *s, AVPacket *pkt)
{
    // 1. 读取 Tag 类型
    tag_type = avio_r8(pb) & 0x1F;

    // 2. 读取数据大小
    data_size = avio_rb24(pb);

    // 3. 读取时间戳
    timestamp = avio_rb24(pb);
    timestamp |= avio_r8(pb) << 24;  // 扩展

    // 4. 读取流 ID（通常为 0）
    stream_id = avio_rb16(pb);

    // 5. 根据类型处理
    switch (tag_type) {
    case FLV_TAG_TYPE_AUDIO:
        return flv_read_audio_packet(s, pb, pkt, data_size, timestamp);
    case FLV_TAG_TYPE_VIDEO:
        return flv_read_video_packet(s, pb, pkt, data_size, timestamp);
    case FLV_TAG_TYPE_META:
        return flv_read_meta(s, pb, data_size);
    }
}
```

**2. 元数据解析（AMF）**
```c
// AMF 类型
enum AMFType {
    AMF_DATA_TYPE_NUMBER      = 0x00,
    AMF_DATA_TYPE_BOOL        = 0x01,
    AMF_DATA_TYPE_STRING      = 0x02,
    AMF_DATA_TYPE_OBJECT      = 0x03,
    AMF_DATA_TYPE_ECMA_ARRAY  = 0x08,
    AMF_DATA_TYPE_ARRAY       = 0x0A,
};

// 读取 AMF 值
static int flv_read_amf_value(AVIOContext *pb, int *key_len,
                               char **key, double *val)
{
    int type = avio_r8(pb);

    switch (type) {
    case AMF_DATA_TYPE_NUMBER:
        *val = av_int2double(avio_rb64(pb));
        break;
    case AMF_DATA_TYPE_BOOL:
        *val = avio_r8(pb);
        break;
    case AMF_DATA_TYPE_STRING:
        len = avio_rb16(pb);
        avio_read(pb, str, len);
        break;
    // ... 更多类型
    }
}

// 解析 onMetaData 脚本对象
// 包含：duration、width、height、videodatarate、audiodatarate、framerate
```

**3. 音频/视频标签格式**
```c
// 音频标签
// - SoundFormat [4 bits]
// - SoundRate [2 bits]
// - SoundSize [1 bit]
// - SoundType [1 bit]
// - Audio Data

// 视频标签
// - FrameType [4 bits]
// - CodecID [4 bits]
// - Video Data

// 特殊处理：VP6 需要额外数据
// VP6F 包含 1 字节 extradata
// 高 4 位：编码宽度与显示宽度差
// 低 4 位：编码高度与显示高度差
```

**4. 索引构建**
```c
// 添加关键帧索引
static void add_keyframes_index(AVFormatContext *s)
{
    FLVContext *flv = s->priv_data;
    AVStream *stream = s->streams[flv->last_keyframe_stream_index];

    for (i = 0; i < flv->keyframe_count; i++) {
        av_add_index_entry(stream,
            flv->keyframe_filepositions[i],  // 文件位置
            flv->keyframe_times[i],          // 时间戳
            0, 0,
            AVINDEX_KEYFRAME);
    }
}
```

#### 颜色信息支持
```c
// FLV 元数据颜色信息
typedef struct FLVMetaVideoColor {
    enum AVColorSpace matrix_coefficients;     // 矩阵系数
    enum AVColorTransferCharacteristic trc;    // 传输特性
    enum AVColorPrimaries primaries;           // 原色
    uint16_t max_cll;                          // 最大内容光级
    uint16_t max_fall;                         // 最大帧平均光级
    FLVMasteringMeta mastering_meta;           // 硕士显示元数据
} FLVMetaVideoColor;

// 从 onMetaData 读取颜色信息
```

#### 直播流支持
```c
// 直播 FLV 检测
static int live_flv_probe(const AVProbeData *p)
{
    // 检测 "NGINX RTMP" 标记
    if (!memcmp(d + offset + 40, "NGINX RTMP", 10))
        return AVPROBE_SCORE_MAX;
}

// 直播流特点：
// - 没有 seek 支持
// - 持续读取直到连接关闭
// - 可能丢失数据
```

---

## 第二部分：主流编解码器实现

### 1. H.264/AVC 解码器（libavcodec/h264dec.c）

#### 架构概述
```c
// H.264 解码器采用模块化架构
// 核心组件：
// 1. SPS/PSP 解析器
// 2. 切片解析器
// 3. 宏块解码器
// 4. 运动补偿
// 5. 去块滤波器

typedef struct H264Context {
    AVCodecContext *avctx;              // 编解码上下文

    // 参数集
    H264ParamSets ps;                   // SPS/PPS 集合
    int max_contexts;                   // 最大上下文数

    // 切片上下文
    H264SliceContext *slice_ctx;        // 切片上下文数组
    int nb_slice_ctx;                   // 切片上下文数量

    // 参考帧管理
    H264Picture *DPB[16];               // 解码图像缓冲区
    H264Picture cur_pic;                // 当前图像
    int cur_pic_ptr;                    // 当前图像指针

    // 错误恢复
    ERContext er;                       // 错误恢复上下文

    // 线程支持
    int execute;                        // 线程执行模式
    int nb_slice_ctx_queued;            // 队列中的切片数
} H264Context;

typedef struct H264SliceContext {
    H264Context *h;                     // 父上下文

    // 切片信息
    int slice_type;                     // I/P/B 切片
    int slice_num;                      // 切片号
    int mb_x, mb_y;                     // 宏块坐标
    int mb_xy;                          // 宏块索引

    // 参考列表
    H264Ref ref_list[2][32];            // 参考列表 0/1
    int ref_count[2];                   // 参考数量

    // 运动矢量
    int16_t mv[2][4][2];                // 运动矢量缓存
    int8_t ref_cache[2][4][8];          // 参考索引缓存

    // 残差数据
    int16_t *mb;                        // 宏块残差
    int mb_width, mb_height;            // 宏块尺寸
} H264SliceContext;

typedef struct H264Picture {
    AVFrame *f;                         // AVFrame
    int reference;                      // 是否为参考帧
    int long_ref;                       // 长期参考
    int poc;                            // 图像序号
    int frame_num;                      // 帧号
    int field_picture;                  // 场图标志
} H264Picture;
```

#### 解码流程

**1. 参数集解析**
```c
// SPS (Sequence Parameter Set)
// 包含：
// - Profile/Level
// - 分辨率
// - 参考帧数
// - 色度格式
// - 时域信息

// PPS (Picture Parameter Set)
// 包含：
// - 熵编码方式
// - 参考帧索引方式
// - 去块滤波参数
// - 变换大小

// 解析流程
static int decode_nal_units(H264Context *h, const uint8_t *buf, int size)
{
    // 1. 分离 NAL 单元
    // 2. 根据 NAL 类型分发
    switch (nal_type) {
    case NAL_SPS:
        ff_h264_decode_sps(h);
        break;
    case NAL_PPS:
        ff_h264_decode_pps(h);
        break;
    case NAL_SLICE:
        ff_h264_decode_slice(h);
        break;
    }
}
```

**2. 切片解析**
```c
// 切片头包含：
// - 第一宏块地址
// - 切片类型
// - 帧号
// - 参考帧索引
// - 量化参数
// - IDR 图片标志

static int h264_slice_header_parse(H264Context *h, H264SliceContext *sl)
{
    // 1. 读取 first_mb_in_slice
    // 2. 读取 slice_type
    // 3. 读取 frame_num
    // 4. 读取参考帧列表
    // 5. 读取量化参数 delta
}
```

**3. 宏块解码**
```c
// 宏块类型
enum H264MbType {
    MB_TYPE_INTRA4x4   = 0x0001,
    MB_TYPE_INTRA16x16 = 0x0002,
    MB_TYPE_INTRA8x8   = 0x0004,
    MB_TYPE_16x16      = 0x0008,
    MB_TYPE_16x8       = 0x0010,
    MB_TYPE_8x16       = 0x0020,
    MB_TYPE_8x8        = 0x0040,
    MB_TYPE_SKIP       = 0x0080,
    // ... 更多类型
};

// 宏块解码流程
static int h264_decode_mb(H264Context *h, H264SliceContext *sl)
{
    // 1. 预测模式解析
    // 2. 运动矢量解析（P/B 切片）
    // 3. 残差系数解析
    // 4. 逆量化
    // 5. 逆变换
    // 6. 运动补偿
    // 7. 预测 + 残差
}
```

**4. 运动补偿**
```c
// 运动补偿类型
// - 16x16, 16x8, 8x16, 8x8
// - 子像素插值：1/4, 1/2, 整像素

// 亮度插值（6 抽头滤波器）
// 半像素：[-1, 4, -11, 40, 20, -6] / 32
// 1/4 像素：更多复杂滤波器

// 色度插值（双线性）
// 半像素：[1, 1] / 2

static void mc_luma(H264Context *h, H264SliceContext *sl,
                    int list, int ref, int x, int y,
                    int width, int height)
{
    // 1. 获取参考帧
    H264Ref *ref_pic = &sl->ref_list[list][ref];

    // 2. 计算整数像素位置
    int mx = sl->mv[list][...][0];
    int my = sl->mv[list][...][1];

    // 3. 子像素插值
    // 根据分数部分选择滤波器

    // 4. 边界处理
    // 使用扩展边界处理边缘宏块
}
```

**5. 去块滤波器**
```c
// 去块滤波强度计算
// Strength = 0-4
// 0: 不滤波
// 1-3: 不同滤波强度
// 4: 强滤波

// 边界类型
// - 宏块边界（内部/外部）
// - 4x4 块边界

// 滤波流程
static void filter_slice(H264Context *h, H264SliceContext *sl,
                        int start_x, int end_x)
{
    // 1. 垂直边界滤波
    for (mb_y = start_y; mb_y < end_y; mb_y++) {
        for (mb_x = start_x; mb_x < end_x; mb_x++) {
            // 滤亮度边界
            filter_mb_edgeh(h, mb_x, mb_y);
        }
    }

    // 2. 水平边界滤波
    for (mb_y = start_y; mb_y < end_y; mb_y++) {
        for (mb_x = start_x; mb_x < end_x; mb_x++) {
            // 滤亮度边界
            filter_mb_edgev(h, mb_x, mb_y);
        }
    }

    // 3. 色度边界滤波（类似）
}
```

**6. 错误恢复**
```c
// 隐藏错误宏块
static int h264_er_decode_mb(void *opaque, int ref, int mv_dir,
                              int mv_type, int (*mv)[2][4][2],
                              int mb_x, int mb_y, int mb_intra,
                              int mb_skipped)
{
    H264Context *h = opaque;
    H264SliceContext *sl = &h->slice_ctx[0];

    // 1. 设置运动矢量
    sl->mb_x = mb_x;
    sl->mb_y = mb_y;

    // 2. 隐藏残差
    // 使用运动补偿替代残差

    // 3. 解码宏块
    ff_h264_hl_decode_mb(h, sl);
}
```

#### 线程支持
```c
// 帧级并行
// - 切片级并行
// - 宏块级并行

// 线程安全
// - DPB 访问同步
// - 参考帧计数
// - 上下文切换

static int h264_decode_frame(AVCodecContext *avctx, AVFrame *frame,
                             int *got_frame, AVPacket *avpkt)
{
    H264Context *h = avctx->priv_data;

    // 1. 初始化切片
    // 2. 分发到线程
    // 3. 等待完成
    // 4. 输出帧
}
```

#### 优化技巧
```c
// SIMD 优化
// - IDCT: SSE2/AVX2
// - 运动补偿: SSE2/AVX2
// - 去块滤波: SSE2/AVX2

// 内存优化
// - 帧池化
// - 参考帧复用
// - 缓存行对齐

// 数据结构优化
// - 查找表
// - 位操作
// - 分支预测
```

---

### 2. AV1 解码器（libavcodec/av1dec.c）

#### 架构概述
```c
// AV1 是 AOMedia Video 1，免版税的下一代视频编解码器
// 特点：
// - 高压缩效率
// - 分层结构
// - 先进的帧内/帧外预测
// - 高效的熵编码

typedef struct AV1DecContext {
    AVCodecContext *avctx;
    AV1Frame ref[8];                   // 参考帧（最多 7 个 + 当前帧）
    AV1Frame cur_frame;                // 当前帧

    // 序列头
    AV1RawSequenceHeader *raw_seq;     // 原始序列头
    int operating_point_idc;           // 操作点

    // 帧头
    AV1RawFrameHeader *raw_frame_header;  // 原始帧头
    int current_frame_id;              // 当前帧 ID

    // tile 解码
    int tiling_col_num;                // tile 列数
    int tiling_row_num;                // tile 行数
    AV1TileGroup *tile_group;          // tile 组

    // 环路滤波器
    AV1RawFilmGrainParams film_grain;   // 胶片颗粒
    AV1RawLoopFilterParams loop_filter; // 环路滤波器参数

    // CDEF（受约束方向增强滤波器）
    int cdef_preskip;                  // CDEF 在环路滤波前

    // 超分辨率
    int use_superres;                  // 启用超分辨率
    int upscaled_width;                // 上缩放宽度

    // 全局运动
    int global_motion_params[7][8];    // 全局运动参数（最多 7 个参考帧）
} AV1DecContext;

typedef struct AV1Frame {
    AVFrame *f;                        // AVFrame
    int order_hint;                    // 顺序提示
    int refresh_frame_flags;           // 刷新帧标志
    int gm_params[7][6];               // 全局运动参数
} AV1Frame;
```

#### 解码流程

**1. 带内 OBU（Open Bitstream Unit）解析**
```c
// AV1 使用 OBU 作为基本容器
// OBU 结构：
// - OBU header [1-2 bytes]
//   - obu_forbidden_bit
//   - obu_type
//   - obu_extension_flag
//   - obu_size_flag
// - OBU size (变长)
// - OBU payload

// OBU 类型
enum AV1OBUType {
    OBU_SEQUENCE_HEADER = 1,
    OBU_TEMPORAL_DELIMITER = 2,
    OBU_FRAME_HEADER = 3,
    OBU_TILE_GROUP = 4,
    OBU_METADATA = 5,
    OBU_FRAME = 6,
    OBU_REDUNDANT_FRAME_HEADER = 7,
    OBU_TILE_LIST = 8,
    // ... 更多类型
};

static int parse_obu_header(AV1DecContext *s, const uint8_t *buf,
                            int size, int *obu_size)
{
    // 1. 读取 OBU 头
    // 2. 解析 OBU 大小（Leb128 编码）
    // 3. 返回 payload 大小
}
```

**2. 序列头解析**
```c
// 序列头包含全局参数
// - Profile (0-3)
// - Level
// - 色度格式
// - 分辨率
// - 参考帧数量
// - 时域层次数
// - 编解码器配置

static av_cold int av1_decode_sequence_header(AV1DecContext *s,
                                               const uint8_t *data,
                                               int size)
{
    GetBitContext gb;
    init_get_bits(&gb, data, size * 8);

    // 1. 读取 profile
    s->seq_hdr->profile = get_bits(&gb, 3);

    // 2. 读取 level
    s->seq_hdr->level = get_bits(&gb, 5);

    // 3. 读取色度格式
    // ...

    // 4. 初始化解码器上下文
    // ...
}
```

**3. 帧头解析**
```c
// 帧头包含：
// - 帧类型（KEY_FRAME, INTER_FRAME, INTRA_ONLY_FRAME, S_FRAME）
// - show_existing_frame
// - 帧尺寸（超分辨率支持）
// - 渲染尺寸
// - 参考帧索引
// - 环路滤波器参数
// - 量化参数
// - 分割参数
// - 全局运动参数

static int av1_decode_frame_header(AV1DecContext *s, const uint8_t *data,
                                    int size, int ref_refresh_mask)
{
    // 1. 读取帧类型
    s->raw_frame_header->frame_type = get_bits(&gb, 2);

    // 2. 读取 show_existing_frame
    if (get_bits(&gb, 1))
        return decode_existing_frame(s);

    // 3. 读取帧尺寸
    if (get_bits(&gb, 1)) {
        // 帧尺寸 override
    }

    // 4. 读取参考帧索引
    for (i = 0; i < REFS_PER_FRAME; i++) {
        s->raw_frame_header->ref_frame_idx[i] = get_bits(&gb, 3);
    }

    // 5. 读取全局运动参数
    read_global_motion_params(s);

    // ... 更多参数
}
```

**4. Tile 解码**
```c
// AV1 将帧划分为 tile（矩形区域）
// Tile 可以独立解码，支持并行

typedef struct AV1TileGroup {
    int tg_start;                      // 起始 tile
    int tg_end;                        // 结束 tile
    uint8_t *data;                     // 数据
    int data_size;                     // 数据大小
} AV1TileGroup;

// Tile 结构
// - Frame Header
// - Tile Group OBU
//   - Tile data (可能多个 tile)

static int av1_decode_tile_group(AV1DecContext *s, const uint8_t *data,
                                  int size)
{
    // 1. 初始化 tile
    // 2. 解析 tile 数据
    // 3. 重建 tile
    // 4. 应用环路滤波器
}
```

**5. 帧内预测**
```c
// AV1 支持多种帧内预测模式
// - DC 预测
// - 平预测
// - 角度预测（35+ 模式）
// - 调色板模式
// - 帧内块复制

// 预测模式
enum AV1IntraPredMode {
    DC_PRED,           // DC 预测
    V_PRED,            // 垂直预测
    H_PRED,            // 水平预测
    D45_PRED,          // 对角线预测
    D135_PRED,
    // ... 更多角度模式
    PAETH_PRED,        // Paeth 预测
    SMOOTH_PRED,       // 平滑预测
    SMOOTH_H_PRED,
    SMOOTH_V_PRED,
};

// 亮度预测（基于 4x4 块）
static void intra_prediction(AV1DecContext *s, int plane,
                              int x, int y, int tx_size,
                              enum AV1IntraPredMode mode)
{
    // 1. 获取参考像素
    // 2. 根据模式预测
    switch (mode) {
    case DC_PRED:
        predict_dc(...);
        break;
    case V_PRED:
        predict_vertical(...);
        break;
    case H_PRED:
        predict_horizontal(...);
        break;
    default:
        predict_angular(...);
    }
}
```

**6. 帧间预测**
```c
// 运动矢量精度
// - 1/4 像素（8 比特）
// - 1/8 像素（10/12 比特）

// 迂回运动补偿（Warped Motion）
// - 平移
// - 仿射
// - 透视

typedef struct AV1WarpedMotionParams {
    int32_t gm_params[8];              // 全局运动参数
    enum AV1TransformationType type;   // 变换类型
    int invalid;                       // 无效标志
} AV1WarpedMotionParams;

// 运动补偿
static void inter_prediction(AV1DecContext *s, int block_x, int block_y,
                              int width, int height, int ref_idx)
{
    AV1Frame *ref = &s->ref[ref_idx];

    // 1. 获取运动矢量
    int mv_x = s->mv[...][0];
    int mv_y = s->mv[...][1];

    // 2. 检查全局运动
    if (s->cur_frame.gm_params[ref_idx][2] != 0) {
        // 使用迂回运动补偿
        warped_motion_compensation(s, ref, mv_x, mv_y);
    } else {
        // 常规运动补偿
        motion_compensation(s, ref, mv_x, mv_y);
    }

    // 3. 子像素插值
    // AV1 使用 8 抽头滤波器
    subpixel_interpolation(...);
}
```

**7. 环路滤波器**
```c
// AV1 的环路滤波器包括：
// 1. 去块滤波器（Deblocking）
// 2. CDEF（受约束方向增强滤波器）
// 3. 超分辨率上采样
// 4. 环路恢复滤波器（LRF）

// 去块滤波器
static void loop_deblocking_filter(AV1DecContext *s)
{
    // 1. 计算滤波强度
    // 2. 滤水平边界
    // 3. 滤垂直边界
}

// CDEF
static void cdef_filter(AV1DecContext *s, int plane,
                         int cdef_y, int cdef_x,
                         int dir, int var)
{
    // 1. 方向估计
    // 2. 应用约束
    // 3. 增强滤波
}

// 环路恢复滤波器
static void loop_restoration_filter(AV1DecContext *s, int plane)
{
    // 1. Wiener 滤波器
    // 2. 自引导滤波器
}
```

**8. 胶片颗粒**
```c
// AV1 支持胶片颗粒合成
// 特点：
// - 在解码端生成颗粒
// - 不增加比特率
// - 保持原始质量

typedef struct AV1FilmGrainParams {
    int apply_grain;                   // 应用颗粒
    int seed;                          // 随机种子
    int num_y_points;                  // 亮度点数
    uint8_t point_y[14][2];            // 亮度点
    int chroma_scaling_from_luma;      // 色度从亮度缩放
    int num_cb_points, num_cr_points;  // 色度点数
    // ... 更多参数
} AV1FilmGrainParams;

static void apply_film_grain(AV1DecContext *s, AVFrame *frame)
{
    // 1. 生成噪声
    // 2. 应用到亮度
    // 3. 应用到色度
}
```

#### 优化策略
```c
// SIMD 优化
// - 帧内预测：SSE4.1/AVX2
// - 运动补偿：SSE4.1/AVX2
// - 环路滤波器：SSE4.1/AVX2
// - 帧内/帧间重建：AVX-512

// 并行解码
// - Tile 级并行
// - 超级帧并行
// - 参考帧缓存优化

// 内存优化
// - 对齐分配
// - 缓存行优化
// - 预取策略
```

---

### 3. AAC 编解码器（libavcodec/aac*.c）

#### 架构概述
```c
// AAC (Advanced Audio Coding) 是高效音频编解码器
// 特点：
// - 支持多声道（最多 48 声道）
// - 支持多种采样率（8-96 kHz）
// - 支持多种比特率
// - 高质量音频

// AAC 解码器核心组件
typedef struct AACDecContext {
    AVClass *class;
    AVCodecContext *avctx;

    // 谱系数据
    FFTContext mdct1024;                // 1024 点 MDCT
    FFTContext mdct128;                 // 128 点 MDCT
    float *mdct_buf;                    // MDCT 缓冲

    // 工具
    AVFloatDSPContext *fdsp;            // 浮点 DSP
    AACEncContext *aacenc;              // AAC 编码器（如果存在）

    // 状态
    int is_saved;                       // 保存状态
    float saved[1024];                  // 保存样本

    // 特殊功能
    AACPSContext ps;                    // 参数立体声
    AACSBRContext sbr;                  // 频谱带复制
} AACDecContext;

// AAC 配置
typedef struct AACConfig {
    int object_type;                    // 对象类型（LC, Main, SSR, LTP）
    int sample_rate;                    // 采样率
    int channels;                       // 声道数
    int channel_config;                 // 声道配置
    int sampling_frequency_index;       // 采样率索引
    int frame_length;                   // 帧长度（通常 1024）
} AACConfig;

// AAC 编码器（简化）
typedef struct AACEncContext {
    AVCodecContext *avctx;
    AACDecContext *aacdsp;              // AAC DSP

    // 心理声学模型
    FFPsyContext psy;
    AACPsyModel psy_model;

    // 比特分配
    AACCoefficients coefs;
    int lam;                            // λ 参数

    // 缓冲
    float *samples;                     // 输入样本
    float *spectrum;                    // 频谱

    // 特殊功能
    AACPSContext ps;                    // 参数立体声
    AACSBRContext sbr;                  // 频谱带复制
} AACEncContext;
```

#### 解码流程

**1. ADTS 帧解析**
```c
// ADTS (Audio Data Transport Stream) 格式
// 用于传输 AAC 数据
// 帧结构：
// - ADTS Fixed Header [7 bytes]
//   - Syncword (0xFFF) [12 bits]
//   - ID [1 bit]
//   - Layer [2 bits]
//   - Protection Absent [1 bit]
//   - Profile [2 bits]
//   - Sampling Frequency Index [4 bits]
//   - Private Bit [1 bit]
//   - Channel Configuration [3 bits]
//   - Originality/Copy/Home [3 bits]
// - ADTS Variable Header [3 bytes]
//   - Copyright [1 bit]
//   - Copyright ID Start [1 bit]
//   - Frame Length [13 bits]
//   - Buffer Fullness [11 bits]
//   - Number of Raw Data Blocks [2 bits]
// - Raw Data Blocks

static int aac_decode_frame(AVCodecContext *avctx, AVFrame *frame,
                            int *got_frame, AVPacket *avpkt)
{
    AACDecContext *ac = avctx->priv_data;
    GetBitContext gb;
    AACRawDataBlock *che;
    int elem_id, ch, ret;

    // 1. 解析 ADTS 头
    if (ac->oc[1].m4ac.frame_length_short)
        parse_adts_frame_header(ac, gb);

    // 2. 解析原始数据块
    for (elem_id = 0; elem_id < elements; elem_id++) {
        che = get_che(ac, elem_type, elem_id);

        // 3. 解析元素
        switch (elem_type) {
        case AAC_SCE:
        case AAC_CCE:
        case AAC_LFE:
            ret = decode_ics(ac, che, gb, 0);
            break;
        case AAC_CPE:
            ret = decode_cpe(ac, che, gb);
            break;
        case AAC_DSE:
            ret = decode_dse(ac, gb);
            break;
        case AAC_PCE:
            ret = decode_pce(ac, gb);
            break;
        case AAC_FIL:
            ret = decode_fil(ac, gb, elem_id);
            break;
        }
    }

    // 4. 输出音频
    return frame_size;
}
```

**2. 素引通道元素（ICS）解码**
```c
// ICS (Individual Channel Stream) 是 AAC 的核心
// 包含：
// - Global gain
// - ICS 信息
// - 谱系数据
// - 脉冲数据
// - TNS (时域噪声整形)
// - 增益控制
// - 频谱系数

static int decode_ics(AACDecContext *ac, SingleChannelElement *sce,
                       GetBitContext *gb, int common_window)
{
    IndividualChannelStream *ics = &sce->ics;
    int i, k, g;

    // 1. 读取 ICS 信息
    // - window_sequence (长/短/开始/停止)
    // - window_shape (正弦/KBD)
    // - max_sfb (最大刻度因子带)
    // - scale_factor_grouping

    // 2. 读取刻度因子
    for (g = 0; g < ics->num_windows; g++) {
        for (k = 0; k < ics->max_sfb; k++) {
            sce->sf_idx[g][k] = get_bits(gb, ...);
        }
    }

    // 3. 读取脉冲数据
    if (get_bits(gb, 1))
        decode_pulse_data(ac, sce, gb);

    // 4. 读取 TNS 数据
    if (get_bits(gb, 1))
        decode_tns(ac, sce, gb);

    // 5. 读取谱系系数
    decode_spectrum(ac, sce, gb);

    // 6. 频谱反量化
    spectral_to_sample(ac, sce);
}
```

**3. 频谱反量化**
```c
// 将量化的频谱系数转换回浮点
// 步骤：
// 1. 反量化
// 2. 应用刻度因子
// 3. 重新排序（短窗口）

static void spectral_to_sample(AACDecContext *ac,
                                SingleChannelElement *sce)
{
    IndividualChannelStream *ics = &sce->ics;
    int i, k, g;

    for (g = 0; g < ics->num_windows; g++) {
        for (k = 0; k < ics->max_sfb; k++) {
            // 反量化
            float coef = sce->coeffs[g][k];

            // 应用刻度因子
            float sf = ff_aac_pow2sf_tab[ sce->sf_idx[g][k] ];

            // 查找表加速
            sce->coeffs[g][k] = coef * sf;
        }
    }

    // MDCT 变换
    for (g = 0; g < ics->num_windows; g++) {
        if (ics->window_sequence == ONLY_LONG_SEQUENCE)
            ac->mdct1024.mdct_calc(&ac->mdct1024,
                                   sce->ret, sce->coeffs);
        else
            ac->mdct128.mdct_calc(&ac->mdct128,
                                   sce->ret, sce->coeffs);
    }
}
```

**4. 特殊功能**

**参数立体声（PS）**
```c
// 参数立体声用于高效编码立体声
// 只编码一个声道，另一个声道用参数表示

typedef struct AACPSContext {
    int start;                          // 起始频带
    int envelope[AAC_MAX_CHANNELS][AAC_MAX_ENVELOPES];  // 包络
    int iid_par;                        // 强度差
    int icc_par;                        // 相关性
    // ... 更多参数
} AACPSContext;

static int ps_decode(AACDecContext *ac, AACPSContext *ps,
                     GetBitContext *gb)
{
    // 1. 读取 PS 参数
    // 2. 合成立体声
    // 3. 应用到频谱
}
```

**频谱带复制（SBR）**
```c
// SBR 用于高频增强
// 复制低频到高频，用参数调整

typedef struct AACSBRContext {
    int sample_rate;                    // 采样率
    int amp_res;                        // 幅度分辨率
    int kx;                             // 起始频带
    int M;                              // 频带数
    uint8_t table_map_k_to_x[64];       // 频带映射
    // ... 更多参数
} AACSBRContext;

static int sbr_decode(AACDecContext *ac, AACSBRContext *sbr,
                      GetBitContext *gb)
{
    // 1. 读取 SBR 参数
    // 2. 生成高频
    // 3. 调整包络
}
```

**时域噪声整形（TNS）**
```c
// TNS 用于控制预回声
// 在频域应用时域滤波器

static int decode_tns(AACDecContext *ac, SingleChannelElement *sce,
                      GetBitContext *gb)
{
    IndividualChannelStream *ics = &sce->ics;
    int w, filt, top, tns_max_order;

    // 1. 读取 TNS 滤波器
    for (w = 0; w < ics->num_windows; w++) {
        for (filt = 0; filt < ics->tns_num_filters[w]; filt++) {
            // 读取滤波器系数
            // 应用 LPC 分析/合成
        }
    }

    return 0;
}
```

#### 编码流程

**1. 心理声学模型**
```c
// 计算掩蔽阈值
// 用于决定量化参数

static void psy_analyze(AACEncContext *s, int chan,
                        const float *data, int len)
{
    // 1. FFT 分析
    // 2. 计算能量
    // 3. 计算掩蔽阈值
    // 4. 计算比特分配
}
```

**2. 比特分配**
```c
// 根据心理声学模型分配比特
// 目标：最大化感知质量

static void aac_encode_bit_allocation(AACEncContext *s,
                                       AACCoefficients *coefs)
{
    int i, w;

    // 1. 计算全局增益
    // 2. 计算刻度因子
    // 3. 分配比特
    // 4. 量化系数
}
```

**3. 量化与编码**
```c
// 量化频谱系数
static int aac_quantize_coeffs(AACEncContext *s,
                                AACCoefficients *coefs)
{
    int i, w;

    for (w = 0; w < coefs->num_windows; w++) {
        for (i = 0; i < coefs->num_bins; i++) {
            // 量化
            float val = coefs->coeffs[w][i];
            int quantized = (int)(pow(fabs(val), 0.75) * sign(val));

            // 缩放
            quantized *= sf[w][i];

            // 存储量化系数
            coefs->qcoefs[w][i] = quantized;
        }
    }

    return 0;
}

// 熵编码（哈夫曼编码）
static int aac_encode_spectrum(AACEncContext *s,
                                 AACCoefficients *coefs,
                                 PutBitContext *pb)
{
    int i, w;

    for (w = 0; w < coefs->num_windows; w++) {
        for (i = 0; i < coefs->num_bins; i++) {
            // 使用哈夫曼编码
            int code = coefs->qcoefs[w][i];
            int bits = ff_aac_spectral_bits[w][code];
            put_bits(pb, bits, code);
        }
    }

    return 0;
}
```

#### 查找表优化
```c
// AAC 解码器使用大量查找表优化

// 刻度因子表
extern float ff_aac_pow2sf_tab[428];
extern float ff_aac_pow34sf_tab[428];

// 窗口系数
DECLARE_ALIGNED(32, extern float, ff_aac_kbd_long_1024)[1024];
DECLARE_ALIGNED(32, extern float, ff_aac_kbd_short_128)[128];

// 窗口频带
extern const uint8_t ff_aac_num_swb_1024[];
extern const uint8_t ff_aac_num_swb_512[];
extern const uint8_t ff_aac_num_swb_128[];

// 哈夫曼码表
extern const uint16_t * const ff_aac_spectral_codes[11];
extern const uint8_t  * const ff_aac_spectral_bits[11];
```

---

## 第三部分：滤镜实现

### 1. 视频缩放滤镜（libavfilter/vf_scale.c）

#### 架构概述
```c
// scale 滤镜使用 libswscale 进行图像缩放和格式转换
// 特点：
// - 支持任意分辨率缩放
// - 支持所有像素格式
// - 支持宽高比保持
// - 支持高级缩放算法

typedef struct ScaleContext {
    const AVClass *class;
    SwsContext *sws;                   // SwsContext
    FFFrameSync fs;                    // 帧同步

    // 输出尺寸
    int w, h;                          // 宽/高
    char *size_str;                    // 尺寸表达式
    double param[2];                   // sws 参数

    // 色度子采样
    int hsub, vsub;

    // 表达式
    char *w_expr;                      // 宽度表达式
    char *h_expr;                      // 高度表达式
    AVExpr *w_pexpr;                   // 宽度解析表达式
    AVExpr *h_pexpr;                   // 高度解析表达式

    // 颜色参数
    int in_color_matrix;               // 输入颜色矩阵
    int out_color_matrix;              // 输出颜色矩阵
    int in_primaries;                  // 输入原色
    int out_primaries;                 // 输出原色
    int in_transfer;                   // 输入传输特性
    int out_transfer;                  // 输出传输特性
    int in_range;                      // 输入范围
    int out_range;                     // 输出范围

    // 特殊选项
    int force_original_aspect_ratio;   // 强制原始宽高比
    int force_divisible_by;            // 强制可被某数整除
    int eval_mode;                     // 评估模式

    int uses_ref;                      // 使用参考
} ScaleContext;
```

#### 表达式求值
```c
// 尺寸可以使用表达式求值
// 可用变量：
// - in_w, iw: 输入宽度
// - in_h, ih: 输入高度
// - out_w, ow: 输出宽度
// - out_h, oh: 输出高度
// - a: 输入宽高比 (iw / ih)
// - dar: 输入显示宽高比
// - sar: 输入像素宽高比
// - n: 帧序号
// - t: 时间戳（秒）
// - ref_w, rw: 参考宽度
// - ref_h, rh: 参考高度

static int eval_expr(AVFilterContext *ctx)
{
    ScaleContext *scale = ctx->priv;

    // 评估宽度表达式
    scale->var_values[VAR_IN_W]  = inlink->w;
    scale->var_values[VAR_IN_H]  = inlink->h;
    scale->var_values[VAR_A]     = (double)inlink->w / inlink->h;
    scale->var_values[VAR_SAR]   = av_q2d(inlink->sample_aspect_ratio);
    scale->var_values[VAR_DAR]   = scale->var_values[VAR_A] *
                                   scale->var_values[VAR_SAR];
    scale->var_values[VAR_N]     = inlink->frame_count_out;
    scale->var_values[VAR_T]     = ts == AV_NOPTS_VALUE ? NAN :
                                   ts * av_q2d(inlink->time_base);

    // 计算宽度
    scale->w = av_expr_eval(scale->w_pexpr,
                             scale->var_values, NULL);

    // 计算高度
    scale->h = av_expr_eval(scale->h_pexpr,
                             scale->var_values, NULL);

    // 强制宽高比
    if (scale->force_original_aspect_ratio) {
        // 保持宽高比
        double aspect = scale->var_values[VAR_A];
        if (scale->w != scale->h * aspect) {
            scale->h = (int)(scale->w / aspect);
        }
    }

    // 强制可整除
    if (scale->force_divisible_by) {
        scale->w = FFALIGN(scale->w, scale->force_divisible_by);
        scale->h = FFALIGN(scale->h, scale->force_divisible_by);
    }
}
```

#### 缩放算法
```c
// libswscale 支持多种缩放算法
// - SWS_FAST_BILINEAR: 快速双线性
// - SWS_BILINEAR: 双线性
// - SWS_BICUBIC: 双三次
// - SWS_X: 8x8 采样
// - SWS_POINT: 邻近
// - SWS_AREA: 像素区域
// - SWS_BICUBLIN: 双三次+双线性
// - SWS_GAUSS: 高斯
// - SWS_SINC: Lanczos
// - SWS_LANCZOS: Lanczos
// - SWS_SPLINE: 样条

// 设置缩放算法
sws_ctx = sws_getCachedContext(sws_ctx,
    src_w, src_h, src_fmt,
    dst_w, dst_h, dst_fmt,
    SWS_BICUBIC,                    // 缩放算法
    NULL, NULL, NULL);
```

#### 帧处理
```c
static int filter_frame(AVFilterLink *link, AVFrame *in)
{
    AVFilterContext *ctx = link->dst;
    ScaleContext *scale = ctx->priv;
    AVFilterLink *outlink = ctx->outputs[0];
    AVFrame *out;

    // 1. 分配输出帧
    out = ff_get_video_buffer(outlink, outlink->w, outlink->h);

    // 2. 缩放
    sws_scale(scale->sws,
              (const uint8_t * const *)in->data, in->linesize,
              0, in->height,
              out->data, out->linesize);

    // 3. 复制属性
    av_frame_copy_props(out, in);

    // 4. 输出
    return ff_filter_frame(outlink, out);
}
```

---

### 2. 视频叠加滤镜（libavfilter/vf_overlay.c）

#### 架构概述
```c
// overlay 滤镜将一个视频叠加到另一个视频上
// 特点：
// - 支持动态位置（表达式）
// - 支持 Alpha 混合
// - 支持多种格式

typedef struct OverlayContext {
    const AVClass *class;
    FFFrameSync fs;                    // 帧同步

    // 位置
    int x, y;                          // 叠加位置
    char *x_expr, *y_expr;             // 位置表达式
    AVExpr *x_pexpr, *y_pexpr;         // 解析后的表达式

    // 格式
    int format;                        // 输出格式
    int main_pix_fmt;                  // 主视频格式
    int overlay_pix_fmt;               // 叠加视频格式

    // 色度子采样
    int hsub, vsub;

    // 评估模式
    int eval_mode;                     // 表达式评估时机

    // 变量
    double var_values[VAR_VARS_NB];
} OverlayContext;

// 可用变量：
// - main_w, W: 主视频宽度
// - main_h, H: 主视频高度
// - overlay_w, w: 叠加视频宽度
// - overlay_h, h: 叠加视频高度
// - x, y: 叠加位置
// - n: 帧序号
// - t: 时间戳（秒）
// - hsub: 水平色度子采样
// - vsub: 垂直色度子采样
```

#### 帧同步
```c
// overlay 滤镜使用帧同步来处理不同帧率的输入

static int config_output(AVFilterLink *outlink)
{
    AVFilterContext *ctx = outlink->src;
    OverlayContext *s = ctx->priv;
    AVFilterLink *mainlink = ctx->inputs[0];
    AVFilterLink *overlaylink = ctx->inputs[1];

    // 初始化帧同步
    return ff_framesync_init(&s->fs, ctx, 2);
}

// 帧同步回调
static int process_frame(FFFrameSync *fs)
{
    AVFilterContext *ctx = fs->parent;
    OverlayContext *s = ctx->priv;
    AVFrame *main, *overlay, *out;

    // 1. 获取帧
    main = ff_framesync_get_frame(fs, 0);
    overlay = ff_framesync_get_frame(fs, 1);

    // 2. 分配输出
    out = ff_get_video_buffer(ctx->outputs[0],
                              ctx->outputs[0]->w,
                              ctx->outputs[0]->h);

    // 3. 混合
    blend_frame(ctx, main, overlay, out);

    // 4. 输出
    return ff_filter_frame(ctx->outputs[0], out);
}
```

#### Alpha 混合
```c
// YUVA 混合
static void blend_yuv(AVFilterContext *ctx, AVFrame *dst,
                      const AVFrame *src, const AVFrame *overlay,
                      int x, int y)
{
    OverlayContext *s = ctx->priv;
    int i, j;

    // 对每个像素
    for (i = 0; i < overlay->height; i++) {
        for (j = 0; j < overlay->width; j++) {
            // 获取像素值
            int y = overlay->data[0][i * overlay->linesize[0] + j];
            int u = overlay->data[1][...];
            int v = overlay->data[2][...];
            int a = overlay->data[3][...];  // Alpha 通道

            // Alpha 混合
            // out = main * (1 - a) + overlay * a
            int out_y = main->data[0][...] * (255 - a) +
                         y * a;
            int out_u = main->data[1][...] * (255 - a) +
                         u * a;
            int out_v = main->data[2][...] * (255 - a) +
                         v * a;

            // 写入输出
            dst->data[0][...] = out_y >> 8;
            dst->data[1][...] = out_u >> 8;
            dst->data[2][...] = out_v >> 8;
        }
    }
}

// RGBA 混合（类似）
static void blend_rgb(AVFilterContext *ctx, AVFrame *dst,
                      const AVFrame *src, const AVFrame *overlay,
                      int x, int y)
{
    // ... 类似 YUVA，但使用 RGB
}
```

---

### 3. 音量滤镜（libavfilter/af_volume.c）

#### 架构概述
```c
// volume 滤镜调整音频音量
// 特点：
// - 支持表达式
// - 支持 ReplayGain
// - 支持多种精度

typedef struct VolumeContext {
    const AVClass *class;
    AVFloatDSPContext *fdsp;           // 浮点 DSP

    // 音量
    char *volume_expr;                 // 音量表达式
    AVExpr *volume_pexpr;              // 解析后的表达式
    double volume;                     // 当前音量值

    // 精度
    enum Precision {
        PRECISION_FIXED,                // 固定点（8/16/32 位）
        PRECISION_FLOAT,                // 浮点（32 位）
        PRECISION_DOUBLE,               // 双精度（64 位）
    } precision;

    // 评估模式
    int eval_mode;                     // 表达式评估时机

    // ReplayGain
    enum ReplayGainMode {
        REPLAYGAIN_DROP,                // 丢弃
        REPLAYGAIN_IGNORE,              // 忽略
        REPLAYGAIN_TRACK,               // 轨道增益
        REPLAYGAIN_ALBUM,               // 专辑增益
    } replaygain;
    double replaygain_preamp;           // ReplayGain 前放
    int replaygain_noclip;              // 防削波

    // 变量
    double var_values[VAR_VARS_NB];
} VolumeContext;

// 可用变量：
// - n: 帧序号
// - nb_channels: 声道数
// - nb_consumed_samples: 消耗的样本数
// - nb_samples: 当前帧的样本数
// - pts: 帧时间戳
// - sample_rate: 采样率
// - startpts: 流开始时间戳
// - startt: 流开始时间
// - t: 帧时间（秒）
// - tb: 时间基
// - volume: 最后设置的音量值
```

#### 音量调整
```c
// 固定点（U8）
static inline void scale_samples_u8(uint8_t *dst, const uint8_t *src,
                                    int nb_samples, int volume)
{
    int i;
    for (i = 0; i < nb_samples; i++)
        dst[i] = av_clip_uint8(((((int64_t)src[i] - 128) * volume + 128) >> 8) + 128);
}

// 固定点（S16）
static inline void scale_samples_s16(uint8_t *dst, const uint8_t *src,
                                     int nb_samples, int volume)
{
    int i;
    int16_t *smp_dst = (int16_t *)dst;
    const int16_t *smp_src = (const int16_t *)src;
    for (i = 0; i < nb_samples; i++)
        smp_dst[i] = av_clip_int16(((int64_t)smp_src[i] * volume + 128) >> 8);
}

// 浮点（FLT）
static void scale_samples_float(AVFloatDSPContext *fdsp,
                                 float *dst, const float *src,
                                 int nb_samples, float volume)
{
    fdsp->vector_fmul_scalar(dst, src, volume, nb_samples);
}

// 双精度（DBL）
static void scale_samples_double(double *dst, const double *src,
                                  int nb_samples, double volume)
{
    int i;
    for (i = 0; i < nb_samples; i++)
        dst[i] = src[i] * volume;
}
```

#### ReplayGain 支持
```c
// ReplayGain 是一种音量标准化方法
// 存储在帧侧数据中

static int replaygain_detect(AVFrame *frame, double *gain)
{
    AVFrameSideData *sd;

    // 查找 ReplayGain 侧数据
    sd = av_frame_get_side_data(frame, AV_FRAME_DATA_REPLAYGAIN);

    if (!sd)
        return 0;

    // 提取增益
    *gain = ((AVReplayGain *)sd->data)->track_gain;
    return 1;
}

// 应用 ReplayGain
static void apply_replaygain(VolumeContext *vol, AVFrame *frame,
                              double volume)
{
    double gain;

    // 检测 ReplayGain
    if (replaygain_detect(frame, &gain)) {
        // 计算最终音量
        double db = gain + vol->replaygain_preamp;
        volume = pow(10, db / 20);

        // 防削波
        if (vol->replaygain_noclip && volume > 1.0)
            volume = 1.0;
    }

    // 应用音量
    vol->volume = volume;
}
```

---

## 第四部分：SIMD 优化策略

### 1. x86 SIMD 优化

#### CPU 特性检测
```c
// libavutil/x86/cpu.c

// 获取 CPU 标志
int av_get_cpu_flags(void)
{
    int flags = 0;

    // 使用 CPUID 指令检测
    if (INLINE_cpuid(1, &eax, &ebx, &ecx, &edx)) {
        // 检测特性
        if (edx & 0x04000000) flags |= AV_CPU_FLAG_SSE;
        if (edx & 0x08000000) flags |= AV_CPU_FLAG_SSE2;
        if (ecx & 0x00000001) flags |= AV_CPU_FLAG_SSE3;
        if (ecx & 0x00000200) flags |= AV_CPU_FLAG_SSSE3;
        if (ecx & 0x00080000) flags |= AV_CPU_FLAG_SSE4_1;
        if (ecx & 0x00100000) flags |= AV_CPU_FLAG_SSE4_2;
        if (ecx & 0x00001000) flags |= AV_CPU_FLAG_AVX;
        if (ebx & 0x00000020) flags |= AV_CPU_FLAG_AVX2;
        // ... 更多特性
    }

    return flags;
}

// 使用宏测试特性
#define EXTERNAL_SSE(flags)      HAVE_SSE     _EXTERNAL(flags)
#define EXTERNAL_SSE2(flags)     HAVE_SSE2    _EXTERNAL(flags)
#define EXTERNAL_AVX(flags)      HAVE_AVX     _EXTERNAL(flags)
#define EXTERNAL_AVX2(flags)     HAVE_AVX2    _EXTERNAL(flags)
#define EXTERNAL_FMA3(flags)     HAVE_FMA3    _EXTERNAL(flags)
```

#### 函数指针初始化
```c
// libavutil/x86/float_dsp_init.c

av_cold void ff_float_dsp_init_x86(AVFloatDSPContext *fdsp)
{
    int cpu_flags = av_get_cpu_flags();

    // SSE
    if (EXTERNAL_SSE(cpu_flags)) {
        fdsp->vector_fmul = ff_vector_fmul_sse;
        fdsp->vector_fmac_scalar = ff_vector_fmac_scalar_sse;
        fdsp->vector_fmul_scalar = ff_vector_fmul_scalar_sse;
        fdsp->vector_fmul_window = ff_vector_fmul_window_sse;
        fdsp->vector_fmul_add = ff_vector_fmul_add_sse;
        fdsp->scalarproduct_float = ff_scalarproduct_float_sse;
    }

    // SSE2
    if (EXTERNAL_SSE2(cpu_flags)) {
        fdsp->vector_dmul = ff_vector_dmul_sse2;
        fdsp->vector_dmac_scalar = ff_vector_dmac_scalar_sse2;
        fdsp->vector_dmul_scalar = ff_vector_dmul_scalar_sse2;
        fdsp->scalarproduct_double = ff_scalarproduct_double_sse2;
    }

    // AVX
    if (EXTERNAL_AVX_FAST(cpu_flags)) {
        fdsp->vector_fmul = ff_vector_fmul_avx;
        fdsp->vector_dmul = ff_vector_dmul_avx;
        fdsp->vector_fmac_scalar = ff_vector_fmac_scalar_avx;
        fdsp->vector_dmac_scalar = ff_vector_dmac_scalar_avx;
        fdsp->vector_fmul_add = ff_vector_fmul_add_avx;
        fdsp->scalarproduct_double = ff_scalarproduct_double_avx;
    }

    // FMA3
    if (EXTERNAL_FMA3_FAST(cpu_flags)) {
        fdsp->vector_fmac_scalar = ff_vector_fmac_scalar_fma3;
        fdsp->vector_fmul_add = ff_vector_fmul_add_fma3;
        fdsp->vector_dmac_scalar = ff_vector_dmac_scalar_fma3;
        fdsp->scalarproduct_float = ff_scalarproduct_float_fma3;
    }
}
```

#### SIMD 实现示例
```c
// SSE 向量乘法
void ff_vector_fmul_sse(float *dst, const float *src0,
                        const float *src1, int len)
{
    int i;
    int n = (len & ~3);  // 对齐到 4 的倍数

    // 处理 4 个浮点数为一组
    for (i = 0; i < n; i += 4) {
        // 加载 4 个浮点数
        __m128 a = _mm_loadu_ps(&src0[i]);
        __m128 b = _mm_loadu_ps(&src1[i]);

        // 相乘
        __m128 result = _mm_mul_ps(a, b);

        // 存储
        _mm_storeu_ps(&dst[i], result);
    }

    // 处理剩余元素
    for (; i < len; i++)
        dst[i] = src0[i] * src1[i];
}

// AVX 向量乘法（8 个浮点数）
void ff_vector_fmul_avx(float *dst, const float *src0,
                        const float *src1, int len)
{
    int i;
    int n = (len & ~7);  // 对齐到 8 的倍数

    for (i = 0; i < n; i += 8) {
        __m256 a = _mm256_loadu_ps(&src0[i]);
        __m256 b = _mm256_loadu_ps(&src1[i]);
        __m256 result = _mm256_mul_ps(a, b);
        _mm256_storeu_ps(&dst[i], result);
    }

    // 处理剩余元素
    for (; i < len; i++)
        dst[i] = src0[i] * src1[i];
}

// FMA3 融合乘加
void ff_vector_fmac_scalar_fma3(float *dst, const float *src,
                                 float mul, int len)
{
    int i;
    int n = (len & ~7);

    __m256 m = _mm256_set1_ps(mul);

    for (i = 0; i < n; i += 8) {
        __m256 d = _mm256_loadu_ps(&dst[i]);
        __m256 s = _mm256_loadu_ps(&src[i]);

        // d += s * mul (单指令)
        d = _mm256_fmadd_ps(s, m, d);

        _mm256_storeu_ps(&dst[i], d);
    }

    for (; i < len; i++)
        dst[i] += src[i] * mul;
}
```

---

### 2. ARM64 NEON 优化

#### CPU 特性检测
```c
// libavutil/aarch64/cpu.c

int av_get_cpu_flags(void)
{
    int flags = 0;

    // 使用系统调用获取 CPU 特性
    // 或者通过 /proc/cpuinfo

    if (have_neon)   flags |= AV_CPU_FLAG_NEON;
    if (have_vfpv4)  flags |= AV_CPU_FLAG_VFPV4;

    return flags;
}
```

#### NEON 实现示例
```c
// NEON 向量乘法
void ff_vector_fmul_neon(float *dst, const float *src0,
                         const float *src1, int len)
{
    int i;
    int n = (len & ~3);  // 对齐到 4 的倍数

    for (i = 0; i < n; i += 4) {
        // 加载 4 个浮点数
        float32x4_t a = vld1q_f32(&src0[i]);
        float32x4_t b = vld1q_f32(&src1[i]);

        // 相乘
        float32x4_t result = vmulq_f32(a, b);

        // 存储
        vst1q_f32(&dst[i], result);
    }

    // 处理剩余元素
    for (; i < len; i++)
        dst[i] = src0[i] * src1[i];
}
```

---

### 3. RISC-V 优化

#### CPU 特性检测
```c
// libavutil/riscv/cpu.c

int av_get_cpu_flags(void)
{
    int flags = 0;

    // 使用 RISC-V 的 misa 寄存器检测
    // 或者通过 /proc/cpuinfo

    if (have_rvv)    flags |= AV_CPU_FLAG_RVV;
    if (have_rvf)    flags |= AV_CPU_FLAG_RVF;
    if (have_rvzve)  flags |= AV_CPU_FLAG_RVZVE;

    return flags;
}
```

#### RISC-V RVV 实现
```c
// RVV 向量乘法（假设 RVV 1.0）
void ff_vector_fmul_rvv(float *dst, const float *src0,
                         const float *src1, int len)
{
    size_t vl;

    // 设置向量长度
    vsetvli(vl, e32, m1, ta, ma);

    for (int i = 0; i < len; i += vl) {
        // 加载向量
        vfloat32m1_t a = vle32_v_f32m1(&src0[i], vl);
        vfloat32m1_t b = vle32_v_f32m1(&src1[i], vl);

        // 相乘
        vfloat32m1_t result = vfmul_vv_f32m1(a, b, vl);

        // 存储
        vse32_v_f32m1(&dst[i], result, vl);
    }
}
```

---

## 总结

### 实现模式总结

**1. 格式实现**
- 基于解析表的处理模式（MOV）
- 基于状态机的解析（MPEG-TS）
- 基于 EBML 的解析（Matroska）
- 基于 Tag 的解析（FLV）

**2. 编解码器实现**
- 分层架构（参数集 → 帧 → 宏块）
- 查找表优化
- SIMD 加速
- 多线程支持

**3. 滤镜实现**
- 表达式求值
- 帧同步
- 格式协商
- Alpha 混合

**4. 优化策略**
- SIMD 向量化
- 缓存行优化
- 内存池化
- 查找表

---

*本文档持续更新中...*

*最后更新：2026-01-17*
