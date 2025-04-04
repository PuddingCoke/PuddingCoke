# 前言
&emsp;&emsp;之前在写图形引擎的时候就有个想法，想让我的图形引擎以一个固定的时间步进（**DeltaTime**）来渲染材质，并且把连续渲染的材质以视频的方式保存下来。其实我很久之前就把这个东西实现了，最近也是修改了下代码，准备写一篇关于这个的随笔。

# 介绍
&emsp;&emsp;看了些网上的视频以及相关的文章，把连续渲染的材质保存为视频的过程，其实是先将材质编码为压缩视频帧，然后用容器来封装。到今天为止，压缩视频帧有三种主流编码格式分别为：**H264**、**HEVC**、**AV1**。去**NVIDIA**的官网上面看了下，发现我目前使用的**GPU**支持**H264**和**HEVC**这两种编码格式，而且相关的API已经写好了，直接从动态链接库载入相关的函数就行了。接下来我们让图形引擎以60帧的速率渲染出BGRA材质，然后用**NVENC API**把材质编码为H264压缩视频帧，最后借助**FFMPEG**把H264压缩视频帧封装到MP4容器。

# 准备过程

## 编码器的初始化
&emsp;&emsp;首先我们需要载入相关的API，用**NvEncodeAPICreateInstance**这个函数就行，而这个函数就在**nvEncodeAPI64.dll**里。

```
moduleNvEncAPI = LoadLibraryA("nvEncodeAPI64.dll");

if (moduleNvEncAPI == 0)
{
	throw "unable to load nvEncodeAPI64.dll";
}

NVENCSTATUS(__stdcall * NVENCAPICreateInstance)(NV_ENCODE_API_FUNCTION_LIST*) = (NVENCSTATUS(*)(NV_ENCODE_API_FUNCTION_LIST*))GetProcAddress(moduleNvEncAPI, "NvEncodeAPICreateInstance");

std::cout << "[class NvidiaEncoder] api instance create status " << NVENCAPICreateInstance(&nvencAPI) << "\n";
```

&emsp;&emsp;**NvEncodeAPICreateInstance**要求传入一个名为**NV_ENCODE_API_FUNCTION_LIST**结构体的地址，这个结构体实际上是一个和编码有关的函数表，接下来我们可以用**nvencAPI**来为我们实现编码的具体操作。载入API后我们的下一步操作就是打开编码会议。

```
NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS sessionParams = { NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS_VER };
sessionParams.device = GraphicsDevice::get();
sessionParams.deviceType = NV_ENC_DEVICE_TYPE_DIRECTX;
sessionParams.apiVersion = NVENCAPI_VERSION;

NVENCCALL(nvencAPI.nvEncOpenEncodeSessionEx(&sessionParams, &encoder));
```

&emsp;&emsp;其中**device**是**ID3D12Device**指针，打开编码会议后**NVENC API**会给我们一个编码器实例也就是上面代码的**encoder**，接下来我们需要初始化**encoder**。初始化**encoder**需要提供一个叫**NV_ENC_INITIALIZE_PARAMS**的结构体，由它的名字就可以知道这个结构体储存的是编码器的初始化参数。这个结构体有很多成员，官方的文档说明了该如何设置这个结构体，而我们要关注的成员有以下这些：**bufferFormat**、**encodeConfig**、**encodeGUID**、**presetGUID**、**tuningInfo**、**encodeWidth**、**encodeHeight**、**darWidth**、**darHeight**、**maxEncodeWidth**、**maxEncodeHeight**、**frameRateNum**、**frameRateDen**。接下来我们介绍下这些参数是什么。

**bufferFormat**：输入的材质格式，如果引擎渲染出的是**DXGI_B8G8R8A8_UNORM**的材质，这个参数应该设置为**NV_ENC_BUFFER_FORMAT_ARGB**

**encodeConfig**：和编码有关的配置，它涉及到很多和视频编码相关的知识，例如比特率控制模式、平均比特率、最大比特率等等。我们可以通过**nvEncGetEncodePresetConfigEx**得到一个粗糙的配置，然后可以在这基础上进行修改。
```
NV_ENC_PRESET_CONFIG presetConfig = { NV_ENC_PRESET_CONFIG_VER,{NV_ENC_CONFIG_VER} };

NVENCCALL(nvencAPI.nvEncGetEncodePresetConfigEx(encoder, codec, preset, tuningInfo, &presetConfig));

NV_ENC_CONFIG config;
memcpy(&config, &presetConfig.presetCfg, sizeof(NV_ENC_CONFIG));
config.profileGUID = profile;

//high quality encode
config.gopLength = 120;
config.frameIntervalP = 1;
config.rcParams.enableLookahead = 0;
config.rcParams.rateControlMode = NV_ENC_PARAMS_RC_VBR;
config.rcParams.averageBitRate = 20000000U;
config.rcParams.maxBitRate = 40000000U;
config.rcParams.vbvBufferSize = config.rcParams.maxBitRate * 4;
config.rcParams.enableAQ = 1;
```

这里的**profileGUID**指的是与编码格式对应的规格，指代的是一系列用于编码的特性，应该和具体的应用场景有关
```
// =========================================================================================
// *   Encode Profile GUIDS supported by the NvEncodeAPI interface.
// =========================================================================================

// {BFD6F8E7-233C-4341-8B3E-4818523803F4}
static const GUID NV_ENC_CODEC_PROFILE_AUTOSELECT_GUID =
{ 0xbfd6f8e7, 0x233c, 0x4341, { 0x8b, 0x3e, 0x48, 0x18, 0x52, 0x38, 0x3, 0xf4 } };

// {0727BCAA-78C4-4c83-8C2F-EF3DFF267C6A}
static const GUID  NV_ENC_H264_PROFILE_BASELINE_GUID =
{ 0x727bcaa, 0x78c4, 0x4c83, { 0x8c, 0x2f, 0xef, 0x3d, 0xff, 0x26, 0x7c, 0x6a } };

// {60B5C1D4-67FE-4790-94D5-C4726D7B6E6D}
static const GUID  NV_ENC_H264_PROFILE_MAIN_GUID =
{ 0x60b5c1d4, 0x67fe, 0x4790, { 0x94, 0xd5, 0xc4, 0x72, 0x6d, 0x7b, 0x6e, 0x6d } };

// {E7CBC309-4F7A-4b89-AF2A-D537C92BE310}
static const GUID NV_ENC_H264_PROFILE_HIGH_GUID =
{ 0xe7cbc309, 0x4f7a, 0x4b89, { 0xaf, 0x2a, 0xd5, 0x37, 0xc9, 0x2b, 0xe3, 0x10 } };

// {7AC663CB-A598-4960-B844-339B261A7D52}
static const GUID  NV_ENC_H264_PROFILE_HIGH_444_GUID =
{ 0x7ac663cb, 0xa598, 0x4960, { 0xb8, 0x44, 0x33, 0x9b, 0x26, 0x1a, 0x7d, 0x52 } };

// {40847BF5-33F7-4601-9084-E8FE3C1DB8B7}
static const GUID NV_ENC_H264_PROFILE_STEREO_GUID =
{ 0x40847bf5, 0x33f7, 0x4601, { 0x90, 0x84, 0xe8, 0xfe, 0x3c, 0x1d, 0xb8, 0xb7 } };

// {B405AFAC-F32B-417B-89C4-9ABEED3E5978}
static const GUID NV_ENC_H264_PROFILE_PROGRESSIVE_HIGH_GUID =
{ 0xb405afac, 0xf32b, 0x417b, { 0x89, 0xc4, 0x9a, 0xbe, 0xed, 0x3e, 0x59, 0x78 } };

// {AEC1BD87-E85B-48f2-84C3-98BCA6285072}
static const GUID NV_ENC_H264_PROFILE_CONSTRAINED_HIGH_GUID =
{ 0xaec1bd87, 0xe85b, 0x48f2, { 0x84, 0xc3, 0x98, 0xbc, 0xa6, 0x28, 0x50, 0x72 } };

// {B514C39A-B55B-40fa-878F-F1253B4DFDEC}
static const GUID NV_ENC_HEVC_PROFILE_MAIN_GUID =
{ 0xb514c39a, 0xb55b, 0x40fa, { 0x87, 0x8f, 0xf1, 0x25, 0x3b, 0x4d, 0xfd, 0xec } };

// {fa4d2b6c-3a5b-411a-8018-0a3f5e3c9be5}
static const GUID NV_ENC_HEVC_PROFILE_MAIN10_GUID =
{ 0xfa4d2b6c, 0x3a5b, 0x411a, { 0x80, 0x18, 0x0a, 0x3f, 0x5e, 0x3c, 0x9b, 0xe5 } };

// For HEVC Main 444 8 bit and HEVC Main 444 10 bit profiles only
// {51ec32b5-1b4c-453c-9cbd-b616bd621341}
static const GUID NV_ENC_HEVC_PROFILE_FREXT_GUID =
{ 0x51ec32b5, 0x1b4c, 0x453c, { 0x9c, 0xbd, 0xb6, 0x16, 0xbd, 0x62, 0x13, 0x41 } };

// {5f2a39f5-f14e-4f95-9a9e-b76d568fcf97}
static const GUID NV_ENC_AV1_PROFILE_MAIN_GUID =
{ 0x5f2a39f5, 0xf14e, 0x4f95, { 0x9a, 0x9e, 0xb7, 0x6d, 0x56, 0x8f, 0xcf, 0x97 } };
```

**encodeGUID**：编码格式，可选的有三个
```
// =========================================================================================
// Encode Codec GUIDS supported by the NvEncodeAPI interface.
// =========================================================================================

// {6BC82762-4E63-4ca4-AA85-1E50F321F6BF}
static const GUID NV_ENC_CODEC_H264_GUID =
{ 0x6bc82762, 0x4e63, 0x4ca4, { 0xaa, 0x85, 0x1e, 0x50, 0xf3, 0x21, 0xf6, 0xbf } };

// {790CDC88-4522-4d7b-9425-BDA9975F7603}
static const GUID NV_ENC_CODEC_HEVC_GUID =
{ 0x790cdc88, 0x4522, 0x4d7b, { 0x94, 0x25, 0xbd, 0xa9, 0x97, 0x5f, 0x76, 0x3 } };

// {0A352289-0AA7-4759-862D-5D15CD16D254}
static const GUID NV_ENC_CODEC_AV1_GUID =
{ 0x0a352289, 0x0aa7, 0x4759, { 0x86, 0x2d, 0x5d, 0x15, 0xcd, 0x16, 0xd2, 0x54 } };
```

**presetGUID**：预设等级，官方的介绍如下。数字越高质量越高但是性能越低，对于H264编码格式有P3到P7可选
```
// =========================================================================================
// *   Preset GUIDS supported by the NvEncodeAPI interface.
// =========================================================================================
// Performance degrades and quality improves as we move from P1 to P7. Presets P3 to P7 for H264 and Presets P2 to P7 for HEVC have B frames enabled by default
// for HIGH_QUALITY and LOSSLESS tuning info, and will not work with Weighted Prediction enabled. In case Weighted Prediction is required, disable B frames by
// setting frameIntervalP = 1
// {FC0A8D3E-45F8-4CF8-80C7-298871590EBF}
static const GUID NV_ENC_PRESET_P1_GUID   =
{ 0xfc0a8d3e, 0x45f8, 0x4cf8, { 0x80, 0xc7, 0x29, 0x88, 0x71, 0x59, 0xe, 0xbf } };

// {F581CFB8-88D6-4381-93F0-DF13F9C27DAB}
static const GUID NV_ENC_PRESET_P2_GUID   =
{ 0xf581cfb8, 0x88d6, 0x4381, { 0x93, 0xf0, 0xdf, 0x13, 0xf9, 0xc2, 0x7d, 0xab } };

// {36850110-3A07-441F-94D5-3670631F91F6}
static const GUID NV_ENC_PRESET_P3_GUID   =
{ 0x36850110, 0x3a07, 0x441f, { 0x94, 0xd5, 0x36, 0x70, 0x63, 0x1f, 0x91, 0xf6 } };

// {90A7B826-DF06-4862-B9D2-CD6D73A08681}
static const GUID NV_ENC_PRESET_P4_GUID   =
{ 0x90a7b826, 0xdf06, 0x4862, { 0xb9, 0xd2, 0xcd, 0x6d, 0x73, 0xa0, 0x86, 0x81 } };

// {21C6E6B4-297A-4CBA-998F-B6CBDE72ADE3}
static const GUID NV_ENC_PRESET_P5_GUID   =
{ 0x21c6e6b4, 0x297a, 0x4cba, { 0x99, 0x8f, 0xb6, 0xcb, 0xde, 0x72, 0xad, 0xe3 } };

// {8E75C279-6299-4AB6-8302-0B215A335CF5}
static const GUID NV_ENC_PRESET_P6_GUID   =
{ 0x8e75c279, 0x6299, 0x4ab6, { 0x83, 0x2, 0xb, 0x21, 0x5a, 0x33, 0x5c, 0xf5 } };

// {84848C12-6F71-4C13-931B-53E283F57974}
static const GUID NV_ENC_PRESET_P7_GUID   =
{ 0x84848c12, 0x6f71, 0x4c13, { 0x93, 0x1b, 0x53, 0xe2, 0x83, 0xf5, 0x79, 0x74 } };
```

**tuningInfo**：调教信息，用来应对不同的视频编码用途，例如串流、云游戏、视频会议等等，有以下可选
```
typedef enum NV_ENC_TUNING_INFO
{
    NV_ENC_TUNING_INFO_UNDEFINED         = 0,                                     /**< Undefined tuningInfo. Invalid value for encoding. */
    NV_ENC_TUNING_INFO_HIGH_QUALITY      = 1,                                     /**< Tune presets for latency tolerant encoding.*/
    NV_ENC_TUNING_INFO_LOW_LATENCY       = 2,                                     /**< Tune presets for low latency streaming.*/
    NV_ENC_TUNING_INFO_ULTRA_LOW_LATENCY = 3,                                     /**< Tune presets for ultra low latency streaming.*/
    NV_ENC_TUNING_INFO_LOSSLESS          = 4,                                     /**< Tune presets for lossless encoding.*/
    NV_ENC_TUNING_INFO_COUNT                                                      /**< Count number of tuningInfos. Invalid value. */
}NV_ENC_TUNING_INFO;
```

**encodeWidth**：编码材质的宽度

**encodeHeight**：编码材质的高度

**darWidth**：分辨率比例分子

**darHeight**：分辨率比例分母

**maxEncodeWidth**：编码材质的最大宽度

**maxEncodeHeight**：编码材质的最大高度

**frameRateNum**：帧率分子

**frameRateDen**：帧率分母

NVENC的编程指南提供了如下推荐设置
![img](https://img2023.cnblogs.com/blog/2774734/202501/2774734-20250114191426725-290602521.png)

我使用的**NV_ENC_INITIALIZE_PARAMS**设置如下：
```
static constexpr NV_ENC_BUFFER_FORMAT bufferFormat = NV_ENC_BUFFER_FORMAT_ARGB;

static constexpr NV_ENC_TUNING_INFO tuningInfo = NV_ENC_TUNING_INFO_HIGH_QUALITY;

const GUID codec = NV_ENC_CODEC_H264_GUID;

const GUID preset = NV_ENC_PRESET_P7_GUID;

const GUID profile = NV_ENC_H264_PROFILE_HIGH_GUID;
```

```
NV_ENC_PRESET_CONFIG presetConfig = { NV_ENC_PRESET_CONFIG_VER,{NV_ENC_CONFIG_VER} };

NVENCCALL(nvencAPI.nvEncGetEncodePresetConfigEx(encoder, codec, preset, tuningInfo, &presetConfig));

NV_ENC_CONFIG config;
memcpy(&config, &presetConfig.presetCfg, sizeof(NV_ENC_CONFIG));
config.version = NV_ENC_CONFIG_VER;
config.profileGUID = profile;

//high quality encode
config.gopLength = 120;
config.frameIntervalP = 1;
config.rcParams.enableLookahead = 0;
config.rcParams.rateControlMode = NV_ENC_PARAMS_RC_VBR;
config.rcParams.averageBitRate = 20000000U;
config.rcParams.maxBitRate = 40000000U;
config.rcParams.vbvBufferSize = config.rcParams.maxBitRate * 4;
config.rcParams.enableAQ = 1;

NV_ENC_INITIALIZE_PARAMS encoderParams = { NV_ENC_INITIALIZE_PARAMS_VER };
encoderParams.bufferFormat = bufferFormat;
encoderParams.encodeConfig = &config;
encoderParams.encodeGUID = codec;
encoderParams.presetGUID = preset;
encoderParams.tuningInfo = tuningInfo;
encoderParams.encodeWidth = Graphics::getWidth();
encoderParams.encodeHeight = Graphics::getHeight();
encoderParams.darWidth = Graphics::getWidth();
encoderParams.darHeight = Graphics::getHeight();
encoderParams.maxEncodeWidth = Graphics::getWidth();
encoderParams.maxEncodeHeight = Graphics::getHeight();
encoderParams.frameRateNum = frameRate;
encoderParams.frameRateDen = 1;
encoderParams.enablePTD = 1;
encoderParams.enableOutputInVidmem = 0;
encoderParams.enableEncodeAsync = 0;
```

设置好后我们可以用它来初始化编码器
```
NVENCCALL(nvencAPI.nvEncInitializeEncoder(encoder, &encoderParams));
```

## FFMPEG的初始化
&emsp;&emsp;初始化编码器后，我们就可以开始用相关的API把材质编码为压缩视频帧了。但是我们还有另外一个问题，也就是压缩视频帧封装的处理。我准备借助**FFMPEG**来封装压缩视频帧。我们首先初始化一个输出容器类型为**mp4**，输出文件名称为**output.mp4**的输出上下文。
```
avformat_alloc_output_context2(&outCtx, nullptr, "mp4", "output.mp4");
```

接下来为输出上下文初始化一个视频流，并且设置其编码格式、编码数据类型、帧的宽、帧的高
```
outStream = avformat_new_stream(outCtx, nullptr);

outStream->id = 0;

AVCodecParameters* vpar = outStream->codecpar;
vpar->codec_id = AV_CODEC_ID_H264;
vpar->codec_type = AVMEDIA_TYPE_VIDEO;
vpar->width = Graphics::getWidth();
vpar->height = Graphics::getHeight();
```

设置完后我们打开输出文件，并写入头部信息
```
avio_open(&outCtx->pb, "output.mp4", AVIO_FLAG_WRITE);

avformat_write_header(outCtx, nullptr);
```

最后我们初始化AVPacket来写入我们的压缩视频帧
```
pkt = av_packet_alloc();
```

## 编码资源的初始化
&emsp;&emsp;输出的压缩视频帧实际上是用一定字节大小的比特流来表示的，官方的编程指南指出对于编码D3D12材质来说，我们还要分配一个ReadbackHeap用于取出比特流，它的推荐大小为材质大小的两倍。
```
readbackHeap(new ReadbackHeap(2 * 4 * Graphics::getWidth() * Graphics::getHeight()))
```

# 编码和封装过程

## 每帧的编码和封装
&emsp;&emsp;如果要将**外部创建的资源**用于输入材质和输出比特流这两个用途，官方的编程指南指出应该先**注册资源**然后**映射资源**。接下来我们分别讲解输入材质和输出比特流的注册以及输入材质和输出比特流的映射。

我们可以用**nvEncRegisterResource**来注册资源
```
NV_ENC_REGISTER_RESOURCE registerInputResource = { NV_ENC_REGISTER_RESOURCE_VER };
registerInputResource.bufferFormat = bufferFormat;
registerInputResource.bufferUsage = NV_ENC_INPUT_IMAGE;
registerInputResource.resourceType = NV_ENC_INPUT_RESOURCE_TYPE_DIRECTX;
registerInputResource.resourceToRegister = inputTexture->getResource();
registerInputResource.subResourceIndex = 0;
registerInputResource.width = Graphics::getWidth();
registerInputResource.height = Graphics::getHeight();
registerInputResource.pitch = 0;
registerInputResource.pInputFencePoint = nullptr;

NVENCCALL(nvencAPI.nvEncRegisterResource(encoder, &registerInputResource));
```

```
NV_ENC_REGISTER_RESOURCE registerOutputResource = { NV_ENC_REGISTER_RESOURCE_VER };
registerOutputResource.bufferFormat = NV_ENC_BUFFER_FORMAT_U8;
registerOutputResource.bufferUsage = NV_ENC_OUTPUT_BITSTREAM;
registerOutputResource.resourceType = NV_ENC_INPUT_RESOURCE_TYPE_DIRECTX;
registerOutputResource.resourceToRegister = readbackHeap->getResource();
registerOutputResource.subResourceIndex = 0;
registerOutputResource.width = 2 * 4 * Graphics::getWidth() * Graphics::getHeight();
registerOutputResource.height = 1;
registerOutputResource.pitch = 0;
registerOutputResource.pInputFencePoint = nullptr;

NVENCCALL(nvencAPI.nvEncRegisterResource(encoder, &registerOutputResource));
```

这里的**resourceToRegister**指的是ID3D12Resource指针，注册资源后**NVENC API**会返回一个注册句柄例如```registerInputResource.registeredResource```，我们接下来可以用这个句柄和**nvEncMapInputResource**来映射资源
```
NV_ENC_MAP_INPUT_RESOURCE mapInputResource = { NV_ENC_MAP_INPUT_RESOURCE_VER };
mapInputResource.registeredResource = registerInputResource.registeredResource;

NVENCCALL(nvencAPI.nvEncMapInputResource(encoder, &mapInputResource));
```

```
NV_ENC_MAP_INPUT_RESOURCE mapOutputResource = { NV_ENC_MAP_INPUT_RESOURCE_VER };
mapOutputResource.registeredResource = registerOutputResource.registeredResource;

NVENCCALL(nvencAPI.nvEncMapInputResource(encoder, &mapOutputResource));
```

映射资源后同样的也会给我们一个映射句柄，比如```mapInputResource.mappedResource```，有了映射句柄后我们接下来可以用**nvEncEncodePicture**提交我们的输入材质和输出比特流。对于D3D12来说，这个函数要求提供一个**NV_ENC_INPUT_RESOURCE_D3D12**结构体和一个**NV_ENC_OUTPUT_RESOURCE_D3D12**结构体，代码如下
```
NV_ENC_INPUT_RESOURCE_D3D12 inputResource = { NV_ENC_INPUT_RESOURCE_D3D12_VER };
inputResource.pInputBuffer = mapInputResource.mappedResource;
inputResource.inputFencePoint = NV_ENC_FENCE_POINT_D3D12{ NV_ENC_FENCE_POINT_D3D12_VER };

NV_ENC_OUTPUT_RESOURCE_D3D12 outputResource = { NV_ENC_INPUT_RESOURCE_D3D12_VER };
outputResource.pOutputBuffer = mapOutputResource.mappedResource;
outputResource.outputFencePoint = NV_ENC_FENCE_POINT_D3D12{ NV_ENC_FENCE_POINT_D3D12_VER };
outputResource.outputFencePoint.pFence = outputFence.Get();
outputResource.outputFencePoint.signalValue = ++outputFenceValue;
outputResource.outputFencePoint.bSignal = true;
```

相信你也注意到了，对于输入材质和输出比特流分别需要一个**inputFencePoint**和一个**outputFencePoint**。**inputFencePoint**用来知道什么时候材质被渲染完了可以用作编码用途，而**outputFencePoint**用来知道什么时候材质被编码完了可以取出对应的比特流。由于我的图形引擎有等待当前帧完成的方法，于是省略了**inputFencePoint**。接下来我们使用**nvEncEncodePicture**来指定提交的材质以及输出的比特流，代码如下
```
NV_ENC_PIC_PARAMS picParams = { NV_ENC_PIC_PARAMS_VER };

picParams.pictureStruct = NV_ENC_PIC_STRUCT_FRAME;

picParams.inputBuffer = &inputResource;

picParams.outputBitstream = &outputResource;

picParams.bufferFmt = bufferFormat;

picParams.inputWidth = Graphics::getWidth();

picParams.inputHeight = Graphics::getHeight();

picParams.completionEvent = nullptr;

NVENCCALL(nvencAPI.nvEncEncodePicture(encoder, &picParams));
```

编码完成后我们可以使用**nvEncLockBitstream**取出比特流，代码如下
```
NV_ENC_LOCK_BITSTREAM lockBitstream = { NV_ENC_LOCK_BITSTREAM_VER };

lockBitstream.outputBitstream = &outputResource;

lockBitstream.doNotWait = 0;

NVENCCALL(nvencAPI.nvEncLockBitstream(encoder, &lockBitstream));

uint8_t* const bitstreamPtr = (uint8_t*)lockBitstream.bitstreamBufferPtr;

const int bitstreamSize = lockBitstream.bitstreamSizeInBytes;
```

有了比特流指针以及比特流字节大小，我们接下来可以进行封装。查阅相关资料后，写入压缩视频帧得通过**av_write_frame**这个函数来执行，它要求输入一个输出上下文以及一个**AVPacket**，我们的比特流就是用**AVPacket**来承载的，官方文档对于这个函数的解释如下
![img](https://img2023.cnblogs.com/blog/2774734/202501/2774734-20250114170419716-1133469553.png)

查看参数解释后，我们应该把**AVPacket**的**stream_index**、**pts**、**dts**、**duration**设置到正确的值。**stream_index**指代的应该就是我们之前创建的视频流的索引```outStream->index```，而**pts**、**dts**、**duration**这三个成员我是一点也不懂，查阅了下官方的文档

![img](https://img2023.cnblogs.com/blog/2774734/202501/2774734-20250114165959747-279162322.png)
发现这三个成员都是以```AVStream->time_base```为单位的值，其中**duration**是帧的持续时间，**dts**和**pts**分别是解压时间戳和呈现时间戳，前者决定了压缩比特流什么时候被解压，后者决定了被解压后的画面什么时候被呈现到用户面前。**FFMPEG**里有对应的函数```av_rescale_q```用来计算这几个成员的值，我们可以通过已编码的帧数和帧率还有```AVStream->time_base```来计算，下面是代码
```
frameEncoded++;

pkt->pts = av_rescale_q(frameEncoded, AVRational{ 1,(int)frameRate }, outStream->time_base);

pkt->dts = pkt->pts;

pkt->duration = av_rescale_q(1, AVRational{ 1,(int)frameRate }, outStream->time_base);

pkt->stream_index = outStream->index;

pkt->data = bitstreamPtr;

pkt->size = bitstreamSize;

av_write_frame(outCtx, pkt);

av_write_frame(outCtx, nullptr);
```

编码完这一帧并封装好比特流后，我们接下来解除对比特流的锁定、取消映射、取消注册
```
NVENCCALL(nvencAPI.nvEncUnlockBitstream(encoder, lockBitstream.outputBitstream));

NVENCCALL(nvencAPI.nvEncUnmapInputResource(encoder, mapInputResource.mappedResource));

NVENCCALL(nvencAPI.nvEncUnregisterResource(encoder, registerInputResource.registeredResource));

NVENCCALL(nvencAPI.nvEncUnmapInputResource(encoder, mapOutputResource.mappedResource));

NVENCCALL(nvencAPI.nvEncUnregisterResource(encoder, registerOutputResource.registeredResource));
```

## 结束编码和封装
将所有的帧编码封装完毕后，我们得用**NVENC API**来发送**EOS**信息
```
NV_ENC_PIC_PARAMS picParams = { NV_ENC_PIC_PARAMS_VER };

picParams.encodePicFlags = NV_ENC_PIC_FLAG_EOS;

NVENCCALL(nvencAPI.nvEncEncodePicture(encoder, &picParams));
```

发送完**EOS**信息后我们做点收尾工作，例如结束对文件的输出还有释放资源什么的
```
NVENCCALL(nvencAPI.nvEncDestroyEncoder(encoder));
delete readbackHeap;
FreeLibrary(moduleNvEncAPI);

av_packet_free(&pkt);
av_write_trailer(outCtx);
avio_close(outCtx->pb);
avformat_free_context(outCtx);
```

# 开启Lookahead
&emsp;&emsp;上面就是最基本的步骤，也许你也注意到了在配置编码参数时我用```config.rcParams.enableLookahead = 0;```关闭了**Lookahead**，首先我们了解下什么是**Lookahead**，NVENC API编程指南对**Lookahead**的解释如下
![img](https://img2023.cnblogs.com/blog/2774734/202501/2774734-20250114173319593-159911647.png)

由此可以见在编码当前帧时，可以通过参考后面的视频帧，来更精确地分配比特率，从而获得更好的画质。如果要开启**Lookahead**我们首先应该把```config.rcParams.enableLookahead```设置为1然后设置```config.rcParams.lookaheadDepth```。假设```config.rcParams.lookaheadDepth = 4;```下面是启用**Lookahead**的编码流程
```
nvEncEncodePicture frame0

nvEncEncodePicture frame1

nvEncEncodePicture frame2

nvEncEncodePicture frame3

nvEncEncodePicture frame4

nvEncLockBitstream frame0

nvEncEncodePicture frame5

nvEncLockBitstream frame1

......
```

我们首先得分配```config.rcParams.lookaheadDepth + 1```个材质供图形引擎循环渲染，下面是修改后的和编码有关的代码，分为**初始化**、**结束输出并释放资源**、**每帧的编码和封装**这三个部分
```
NvidiaEncoder::NvidiaEncoder(const UINT frameToEncode) :
	Encoder(frameToEncode), encoder(nullptr),
	readbackHeap(new ReadbackHeap(2 * 4 * Graphics::getWidth() * Graphics::getHeight())),
	outCtx(nullptr), outStream(nullptr), pkt(nullptr),
	nvencAPI{ NV_ENCODE_API_FUNCTION_LIST_VER },
	outputFenceValue(0)
{
	moduleNvEncAPI = LoadLibraryA("nvEncodeAPI64.dll");

	if (moduleNvEncAPI == 0)
	{
		throw "unable to load nvEncodeAPI64.dll";
	}

	NVENCSTATUS(__stdcall * NVENCAPICreateInstance)(NV_ENCODE_API_FUNCTION_LIST*) = (NVENCSTATUS(*)(NV_ENCODE_API_FUNCTION_LIST*))GetProcAddress(moduleNvEncAPI, "NvEncodeAPICreateInstance");

	std::cout << "[class NvidiaEncoder] api instance create status " << NVENCAPICreateInstance(&nvencAPI) << "\n";

	NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS sessionParams = { NV_ENC_OPEN_ENCODE_SESSION_EX_PARAMS_VER };
	sessionParams.device = GraphicsDevice::get();
	sessionParams.deviceType = NV_ENC_DEVICE_TYPE_DIRECTX;
	sessionParams.apiVersion = NVENCAPI_VERSION;

	NVENCCALL(nvencAPI.nvEncOpenEncodeSessionEx(&sessionParams, &encoder));

	NV_ENC_PRESET_CONFIG presetConfig = { NV_ENC_PRESET_CONFIG_VER,{NV_ENC_CONFIG_VER} };

	NVENCCALL(nvencAPI.nvEncGetEncodePresetConfigEx(encoder, codec, preset, tuningInfo, &presetConfig));

	NV_ENC_CONFIG config;
	memcpy(&config, &presetConfig.presetCfg, sizeof(NV_ENC_CONFIG));
	config.version = NV_ENC_CONFIG_VER;
	config.profileGUID = profile;

	//high quality encode
	config.gopLength = 120;
	config.frameIntervalP = 1;
	config.rcParams.enableLookahead = 1;
	config.rcParams.lookaheadDepth = lookaheadDepth;
	config.rcParams.rateControlMode = NV_ENC_PARAMS_RC_VBR;
	config.rcParams.averageBitRate = 20000000U;
	config.rcParams.maxBitRate = 40000000U;
	config.rcParams.vbvBufferSize = config.rcParams.maxBitRate * 4;
	config.rcParams.enableAQ = 1;

	NV_ENC_INITIALIZE_PARAMS encoderParams = { NV_ENC_INITIALIZE_PARAMS_VER };
	encoderParams.bufferFormat = bufferFormat;
	encoderParams.encodeConfig = &config;
	encoderParams.encodeGUID = codec;
	encoderParams.presetGUID = preset;
	encoderParams.tuningInfo = tuningInfo;
	encoderParams.encodeWidth = Graphics::getWidth();
	encoderParams.encodeHeight = Graphics::getHeight();
	encoderParams.darWidth = Graphics::getWidth();
	encoderParams.darHeight = Graphics::getHeight();
	encoderParams.maxEncodeWidth = Graphics::getWidth();
	encoderParams.maxEncodeHeight = Graphics::getHeight();
	encoderParams.frameRateNum = frameRate;
	encoderParams.frameRateDen = 1;
	encoderParams.enablePTD = 1;
	encoderParams.enableOutputInVidmem = 0;
	encoderParams.enableEncodeAsync = 0;

	NVENCCALL(nvencAPI.nvEncInitializeEncoder(encoder, &encoderParams));

	GraphicsDevice::get()->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&outputFence));

	std::cout << "[class NvidiaEncoder] render at " << Graphics::getWidth() << " x " << Graphics::getHeight() << "\n";

	std::cout << "[class NvidiaEncoder] frameRate " << frameRate << "\n";

	std::cout << "[class NvidiaEncoder] frameToEncode " << frameToEncode << "\n";

	std::cout << "[class NvidiaEncoder] start encoding\n";

	avformat_alloc_output_context2(&outCtx, nullptr, "mp4", "output.mp4");

	outStream = avformat_new_stream(outCtx, nullptr);

	outStream->id = 0;

	AVCodecParameters* vpar = outStream->codecpar;
	vpar->codec_id = AV_CODEC_ID_H264;
	vpar->codec_type = AVMEDIA_TYPE_VIDEO;
	vpar->width = Graphics::getWidth();
	vpar->height = Graphics::getHeight();

	avio_open(&outCtx->pb, "output.mp4", AVIO_FLAG_WRITE);

	avformat_write_header(outCtx, nullptr);

	pkt = av_packet_alloc();

	NV_ENC_REGISTER_RESOURCE registerOutputResource = { NV_ENC_REGISTER_RESOURCE_VER };
	registerOutputResource.bufferFormat = NV_ENC_BUFFER_FORMAT_U8;
	registerOutputResource.bufferUsage = NV_ENC_OUTPUT_BITSTREAM;
	registerOutputResource.resourceType = NV_ENC_INPUT_RESOURCE_TYPE_DIRECTX;
	registerOutputResource.resourceToRegister = readbackHeap->getResource();
	registerOutputResource.subResourceIndex = 0;
	registerOutputResource.width = 2 * 4 * Graphics::getWidth() * Graphics::getHeight();
	registerOutputResource.height = 1;
	registerOutputResource.pitch = 0;
	registerOutputResource.pInputFencePoint = nullptr;

	NVENCCALL(nvencAPI.nvEncRegisterResource(encoder, &registerOutputResource));

	registeredOutputResourcePtr = registerOutputResource.registeredResource;

	NV_ENC_MAP_INPUT_RESOURCE mapOutputResource = { NV_ENC_MAP_INPUT_RESOURCE_VER };
	mapOutputResource.registeredResource = registerOutputResource.registeredResource;

	NVENCCALL(nvencAPI.nvEncMapInputResource(encoder, &mapOutputResource));

	mappedOutputResourcePtr = mapOutputResource.mappedResource;
}
```

```
NvidiaEncoder::~NvidiaEncoder()
{
	if (moduleNvEncAPI)
	{
		nvencAPI.nvEncUnmapInputResource(encoder, mappedOutputResourcePtr);

		nvencAPI.nvEncUnregisterResource(encoder, registeredOutputResourcePtr);

		while (mappedInputResourcePtrs.size())
		{
			nvencAPI.nvEncUnmapInputResource(encoder, mappedInputResourcePtrs.front());

			mappedInputResourcePtrs.pop();
		}

		while (registeredInputResourcePtrs.size())
		{
			nvencAPI.nvEncUnregisterResource(encoder, registeredInputResourcePtrs.front());

			registeredInputResourcePtrs.pop();
		}

		NVENCCALL(nvencAPI.nvEncDestroyEncoder(encoder));

		delete readbackHeap;

		FreeLibrary(moduleNvEncAPI);

		av_packet_free(&pkt);
		av_write_trailer(outCtx);
		avio_close(outCtx->pb);
		avformat_free_context(outCtx);
	}
}
```

```
bool NvidiaEncoder::encode(Texture* const inputTexture)
{
	timeStart = std::chrono::steady_clock::now();

	if (frameEncoded == frameToEncode)
	{
		NV_ENC_PIC_PARAMS picParams = { NV_ENC_PIC_PARAMS_VER };

		picParams.encodePicFlags = NV_ENC_PIC_FLAG_EOS;

		NVENCCALL(nvencAPI.nvEncEncodePicture(encoder, &picParams));

		encoding = false;

		std::cout << "\n[class NvidiaEncoder] encode complete!\n";

		std::cout << "[class NvidiaEncoder] frame encode avg speed " << frameToEncode / encodeTime << "\n";
	}
	else
	{
		NV_ENC_REGISTER_RESOURCE registerInputResource = { NV_ENC_REGISTER_RESOURCE_VER };
		registerInputResource.bufferFormat = bufferFormat;
		registerInputResource.bufferUsage = NV_ENC_INPUT_IMAGE;
		registerInputResource.resourceType = NV_ENC_INPUT_RESOURCE_TYPE_DIRECTX;
		registerInputResource.resourceToRegister = inputTexture->getResource();
		registerInputResource.subResourceIndex = 0;
		registerInputResource.width = Graphics::getWidth();
		registerInputResource.height = Graphics::getHeight();
		registerInputResource.pitch = 0;
		registerInputResource.pInputFencePoint = nullptr;

		NVENCCALL(nvencAPI.nvEncRegisterResource(encoder, &registerInputResource));

		registeredInputResourcePtrs.push(registerInputResource.registeredResource);

		NV_ENC_MAP_INPUT_RESOURCE mapInputResource = { NV_ENC_MAP_INPUT_RESOURCE_VER };
		mapInputResource.registeredResource = registerInputResource.registeredResource;

		NVENCCALL(nvencAPI.nvEncMapInputResource(encoder, &mapInputResource));

		mappedInputResourcePtrs.push(mapInputResource.mappedResource);

		NV_ENC_INPUT_RESOURCE_D3D12 inputResource = { NV_ENC_INPUT_RESOURCE_D3D12_VER };
		inputResource.pInputBuffer = mapInputResource.mappedResource;
		inputResource.inputFencePoint = NV_ENC_FENCE_POINT_D3D12{ NV_ENC_FENCE_POINT_D3D12_VER };

		NV_ENC_OUTPUT_RESOURCE_D3D12 outputResource = { NV_ENC_INPUT_RESOURCE_D3D12_VER };
		outputResource.pOutputBuffer = mappedOutputResourcePtr;
		outputResource.outputFencePoint = NV_ENC_FENCE_POINT_D3D12{ NV_ENC_FENCE_POINT_D3D12_VER };
		outputResource.outputFencePoint.pFence = outputFence.Get();
		outputResource.outputFencePoint.signalValue = ++outputFenceValue;
		outputResource.outputFencePoint.bSignal = true;

		outputResources.push(outputResource);

		NV_ENC_PIC_PARAMS picParams = { NV_ENC_PIC_PARAMS_VER };

		picParams.pictureStruct = NV_ENC_PIC_STRUCT_FRAME;

		picParams.inputBuffer = &inputResource;

		picParams.outputBitstream = &outputResource;

		picParams.bufferFmt = bufferFormat;

		picParams.inputWidth = Graphics::getWidth();

		picParams.inputHeight = Graphics::getHeight();

		picParams.completionEvent = nullptr;

		const NVENCSTATUS status = nvencAPI.nvEncEncodePicture(encoder, &picParams);

		if ((outputResources.size() == lookaheadDepth + 1) && (status == NV_ENC_SUCCESS || status == NV_ENC_ERR_NEED_MORE_INPUT))
		{
			frameEncoded++;

			NV_ENC_LOCK_BITSTREAM lockBitstream = { NV_ENC_LOCK_BITSTREAM_VER };

			lockBitstream.outputBitstream = &outputResources.front();

			lockBitstream.doNotWait = 0;

			NVENCCALL(nvencAPI.nvEncLockBitstream(encoder, &lockBitstream));

			uint8_t* const bitstreamPtr = (uint8_t*)lockBitstream.bitstreamBufferPtr;

			const int bitstreamSize = lockBitstream.bitstreamSizeInBytes;

			pkt->pts = av_rescale_q(frameEncoded, AVRational{ 1,(int)frameRate }, outStream->time_base);

			pkt->dts = pkt->pts;

			pkt->duration = av_rescale_q(1, AVRational{ 1,(int)frameRate }, outStream->time_base);

			pkt->stream_index = outStream->index;

			pkt->data = bitstreamPtr;

			pkt->size = bitstreamSize;

			av_write_frame(outCtx, pkt);

			av_write_frame(outCtx, nullptr);

			NVENCCALL(nvencAPI.nvEncUnlockBitstream(encoder, lockBitstream.outputBitstream));

			outputResources.pop();

			nvencAPI.nvEncUnmapInputResource(encoder, mappedInputResourcePtrs.front());

			mappedInputResourcePtrs.pop();

			nvencAPI.nvEncUnregisterResource(encoder, registeredInputResourcePtrs.front());

			registeredInputResourcePtrs.pop();
		}
		else if (status != NV_ENC_SUCCESS && status != NV_ENC_ERR_NEED_MORE_INPUT)
		{
			const char* error = nvencAPI.nvEncGetLastErrorString(encoder);

			std::cout << "status " << status << "\n";

			std::cout << error << "\n";

			__debugbreak();
		}
	}

	displayProgress();

	timeEnd = std::chrono::steady_clock::now();

	const float frameTime = std::chrono::duration<float>(timeEnd - timeStart).count();

	encodeTime += frameTime;

	return encoding;
}
```

# 总结
&emsp;&emsp;上述就是所有编码和封装的流程，完成这件事还是学到了挺多知识的。目前只遇到了一个问题，使用H264编码格式时在启用**Lookahead**和**B帧**时编码出现了问题导致无法开启**B帧**，NVENC API头文件对**B帧**编码的解释如下
![img](https://img2023.cnblogs.com/blog/2774734/202501/2774734-20250114175824471-1540397172.png)
也就是说**B帧**的编码需要后面的帧提交来参考并进行编码，但是我已经开启了**Lookahead**，理论上来说应该是不需要考虑这个问题的，但是**D3D12 Debug Layer**和**NVENC API**会报如下错误
```
D3D12 ERROR: ID3D12CommandQueue::ExecuteCommandLists: Non-simultaneous-access Texture Resource (0x000001FC1F9B38F0:'Unnamed Object') is still referenced by write|transition_barrier GPU operations in-flight on another Command Queue (0x000001FC1EF2B1E0:'Unnamed ID3D12CommandQueue Object'). It is not safe to start transition_barrier GPU operations now on this Command Queue (0x000001FC1F201230:'Unnamed ID3D12CommandQueue Object'). This can result in race conditions and application instability. [ EXECUTION ERROR #1047: OBJECT_ACCESSED_WHILE_STILL_IN_USE]
D3D12 ERROR: ID3D12CommandQueue::ExecuteCommandLists: Non-simultaneous-access Texture Resource (0x000001FC1F9B6930:'Unnamed Object') is still referenced by write|transition_barrier GPU operations in-flight on another Command Queue (0x000001FC1EF2B1E0:'Unnamed ID3D12CommandQueue Object'). It is not safe to start transition_barrier GPU operations now on this Command Queue (0x000001FC1F201230:'Unnamed ID3D12CommandQueue Object'). This can result in race conditions and application instability. [ EXECUTION ERROR #1047: OBJECT_ACCESSED_WHILE_STILL_IN_USE]
D3D12 ERROR: ID3D12CommandQueue::ExecuteCommandLists: Non-simultaneous-access Texture Resource (0x000001FC1F9B7540:'Unnamed Object') is still referenced by write|transition_barrier GPU operations in-flight on another Command Queue (0x000001FC1EF2B1E0:'Unnamed ID3D12CommandQueue Object'). It is not safe to start transition_barrier GPU operations now on this Command Queue (0x000001FC1F201230:'Unnamed ID3D12CommandQueue Object'). This can result in race conditions and application instability. [ EXECUTION ERROR #1047: OBJECT_ACCESSED_WHILE_STILL_IN_USE]
```

```
error occured at function nvencAPI.nvEncLockBitstream(encoder, &lockBitstream)
status 8 INVALID PARAM
```

真的是个超级奇怪的问题，虽然**B帧**有最大的压缩率，但是好像不擅长动作非常复杂的情况。有时候可能会让引擎渲染一些非常复杂的画面，关掉**B帧**对我来说其实还挺合适的，索性就不管这个问题了。

# 参考资料
[NVENC Video Encoder API Programming Guide](https://docs.nvidia.com/video-technologies/video-codec-sdk/12.2/nvenc-video-encoder-api-prog-guide/index.html)

[Official Samples](https://developer.nvidia.com/nvidia-video-codec-sdk/download)