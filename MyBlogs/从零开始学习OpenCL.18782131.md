# 前言
&emsp;&emsp;之前在读《**Real-Time Rendering**》的时候，书中稍微提了一嘴**GPGPU**的概念，**GPGPU**即使用图形处理单元的一般用途计算（**General Purpose Computation Using Graphics Processor**）。之前对这个挺感兴趣的，于是想学一点关于这个的内容。
&emsp;&emsp;能利用设备进行通用大规模并行运算的底层**API**有三个，分别为**OpenCL**（**Open Computing Language**）、**CUDA**（**Compute Unified Device Architecture**）、**HIP**（**Heterogeneous-computing Interface for Portability**）。**OpenCL**由**Khronos**作为异构计算的开放标准发布，而**CUDA**是**NVIDIA**所专有的而且只能用于**NVIDIA**研发的**GPU**，**HIP**由**AMD**开发与**CUDA**在**API**和其它方面有很大的相似处。在当前这个时间节点，进行并行运算的话应该是**GPU**比较擅长一些，我目前使用的是**NVIDIA**研发的**GPU**，**OpenCL**虽然支持多平台多设备，但是应该在某些方面做了取舍，如果你正在使用**NVIDIA**研发的**GPU**，有能力的话最好使用**CUDA**，因为不管是在生态还是对**GPU**的利用率上**CUDA**都优于**OpenCL**。不过我稍微看了下**OpenCL**以及**CUDA**的一些概念性的知识还有计算程序的编写，两者在很多地方都有相似处，例如非常重要的**内核**（**Kernel**）这一概念。我觉得多学点应该没什么问题，于是想学学**OpenCL**。
&emsp;&emsp;某网站上的一个叫**Developer Central**的上传者发了一系列介绍**OpenCL**的视频，虽然视频总长度只有1小时左右但是实际上介绍了非常多的东西，例如它的执行模型、平台模型、内存模型等等。下面我将以视频的时间线来介绍**OpenCL**，最后会写点代码做个小练习，计算一张分形图像并保存下来。

# 什么是OpenCL？
&emsp;&emsp;**OpenCL**即开放式计算语言（**Open Computing Language**），使用它可以利用**CPU**、**GPU**、**DSP**等设备来加速**并行运算**（**Parallel Computation**），可以在需要非常大的运算量的问题上获得巨大的速度提升，而且还可以写跨平台的代码让不同架构的设备都能运行。为什么**OpenCL**能利用设备进行并行运算呢？我们先来看看它的执行模型。

# 执行模型(Execution Model)
&emsp;&emsp;**OpenCL**利用设备进行的并行运算实际上是在**内核**（**Kernel**）上执行的，**内核**是可执行代码的基本单元，可以把它比做C函数，它的执行方式可以是**数据并行**或者**任务并行**。**计算程序**（**Program**）是**内核**的集合，可以把它比做动态库。接下来我们了解下**数据并行**和**任务并行**这两种执行方式。

## 任务并行的表达
&emsp;&emsp;调用**API**让设备进行并行运算时，得通过**clEnqueueNDRangeKernel**来让内核入列计算，入列计算可以是有序或无序的，我们可以让不同内核按顺序依次执行或者不按顺序同时执行。当内核之间没有依赖关系如下图所示进行无序入列计算时，这样我们就达到了**任务并行**这一方式，接下来我们看看**数据并行**是如何表达的。
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330211632437-713441245.png)

## 数据并行的表达
&emsp;&emsp;首先我们来看看使用单线程顺序计算的代码和并行运算使用到的内核的代码
```
void scalar_mul(int n,const float* a,const float* b,float* result)
{
    for(int i=0;i<n;i++)
    {
        result[i]=a[i]*b[i];
    }
}
```

```
kernel dp_mul(global const float* a,global const float* b,global float* result)
{
    int id=get_global_id(0);

    result[id]=a[id]*b[id];
}
```
&emsp;&emsp;顺序运算的result数组的每个元素都是在单个线程上依次计算的，而并行运算的result数组的每个元素实际上是由每个在**全局域**（**Global Domain**）上工作的**工作项**（**Work-item**）计算的，如下图所示。这些在**全局域**上的每个**工作项**都会执行这个叫**dp_mul**的**内核**，我们可以观察下**内核**代码，首先dp_mul内核提供了输入输出数据**a**、**b**、**result**，此外dp_mul还描述了数据处理方法即```result[id]=a[id]*b[id];```，然后每个**工作项**还附带了独特的**ID**即```int id=get_global_id(0);```，有了上面的三个情报，每个**工作项**就知道要负责哪些数据元素的输出以及该怎么进行具体的运算，这就是**数据并行**的表达。
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250331123826909-1157014482.png)
&emsp;&emsp;通常要处理的数据元素数量是有限的，计算一个长度1000的数组的输出最多需要1000个**工作项**，对一张1920x1080的图像进行后处理最多需要1920x1080个**工作项**，处理一个1024x1024x16的3D图像最多需要1024x1024x16个**工作项**，所以**全局维度**（**Global Dimension**）就被用来描述计算的范围也就是并行运算的**工作项**的数量。在这一步之上一些一定区域内的**工作项**被进一步规划到了**工作组**（**Work Group**）里，如下图所示。**本地维度**（**Local Dimension**）定义了每个**工作组**的大小，假设要处理一张1024x1024的图像，**工作组**总共有8x8个，这个时候**本地维度**就是128x128。这么做有两个好处，首先每个**工作组**内的**工作项**可以共享一定大小的**本地内存**而且组内的**工作项**可以进行同步。同步可以通过**屏障**（**Barrier**）来同步进度，还可以用**内存屏障**（**Memory Fence**）来同步内存读写。
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330212222321-1255621990.png)

# 平台模型（Platform Model）
&emsp;&emsp;上述就是**OpenCL**的处理模型，接下来我们了解下它的平台模型。通常地来说，一台**主机**（**Host**）会连接一个或多个**OpenCL设备**（**OpenCL Device**）,这些设备可以是任何能为**OpenCL**提供处理能力的设备例如**CPU**、**GPU**、**DSP**。一个**OpenCL Device**一般有一个或多个**计算单元**（**Compute Unit**）,每个**计算单元**又是由一个或多个**处理元素**（**Processing Element**）组成，这些**处理元素**可以以**SIMD**（**Single Instruction Multiple Data**）或**SPMD**（**Single Program Multiple Data**）的方式来执行代码。
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330212450987-944731936.png)

# 内存模型（Memory Model）
&emsp;&emsp;了解完**OpenCL**的平台模型后接下来我们了解下**OpenCL**的内存模型。**OpenCL**的内存模型中总共有5种类型的内存，分别是**主机内存**（**Host Memory**）、**全局内存**（**Global Memory**）、**常量内存**(**Constant Memory**)、**本地内存**（**Local Memory**）、**私有内存**（**Private Memory**）。**主机内存**是运行当前操作系统的处理器能随意支配的内存，后面四种类型的内存都是在**运算设备**（**Compute Device**）当中的。其中**全局内存**和**常量内存**对于全域中的**工作项**都是可见的，此外每一个**工作组**都有可以随意支配的**本地内存**，因此在一个**工作组**内的**工作项**共享所属于这个**工作组**的**本地内存**，最后每个**工作项**都有可以随意支配的**私有内存**。
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330212531152-368965946.png)
&emsp;&emsp;关于内存模型有两点要注意的地方，第一点编程者要进行显式内存管理，主动调用**API**让内存在**主机内存**和**全局/常量内存**之间迁移。第二点**本地内存**、**全局内存**、**常量内存**是没有同步的，对于**本地内存**来说编程者可以用之前讲到的**屏障**和**内存屏障**进行同步，而**全局/常量内存**可以使用用于同步的**API**以及**事件**（**Event**）来进行。有效的管理内存迁移以及内存同步可以让设备计算效率最大化从而获得最大的性能。

# OpenCL中的对象以及任务提交
&emsp;&emsp;以上就是**OpenCL**非常重要的三个模型的概念，接下来我们先粗略的了解下**OpenCL**对象以及向**设备**提交任务的流程。

## OpenCL对象
- **初始化对象**
    - **设备**（**Device**）：**GPU**、**CPU**、**DSP**......
    - **上下文**（**Context**）：**设备**的集合
    - **队列**（**Queue**）：向**设备**提交工作

- **内存对象**
    - **缓冲**（**Buffer**）：一块内存
    - **图像**（**Image**）：2D或3D有格式图像

- **执行对象**
    - **程序**（**Program**）：**内核**的集合
    - **内核**（**Kernel**）：具体会执行的代码以及与之相关联的输入参数

- **同步与性能评估**
    - **事件**（**Event**）

## 任务提交流程
1. 从文件读取源代码或者预编译的二进制代码
2. 创建并编译**计算程序**
3. 从**计算程序**中取出**内核对象**
4. 创建**内存对象**并为**内存对象**初始化数据
5. 设置**内核对象**的参数
6. **内核**入列计算（有序或无序）

&emsp;&emsp;这里有要注意的一点，当**内核**无序入列计算时，这个时候运行时会根据目前的资源决定哪些**内核**该先计算。如果你想让**内核**按顺序执行计算的话得用**事件**来解决**内核**之间的依赖关系。

# 初始化对象的初始化
&emsp;&emsp;了解完**OpenCL对象**以及**任务提交**后接下来就可以调用相关**API**利用设备进行并行运算了。首先我们获取**平台**（**Platform**），接着获取所属**平台**的**设备**，然后根据获取的设备创建**上下文**，创建完上下文后最后我们可以为**上下文**中的**设备**创建**队列**。

初始化过程有几个要注意的地方
1. 多核**CPU**会被看作一个**设备**，**OpenCL**会在所有核心上以**数据并行**的方式执行**内核**
2. 之前说过**上下文**是**设备**的集合，事实上**上下文**可以让当中的**设备**进行内存共享
3. 所有工作都是通过**队列**来提交的，因此要让**设备**进行并行运算，每个**设备**都得至少有一个**队列**
4. **设备**挑选可以通过**clGetDeviceInfo**来询问与**设备**相关的信息，例如专用内存大小以及时钟频率等等

下面是相关的代码

```
cl_platform_id platforms[16];

cl_uint numPlatforms = 0;

CK(clGetPlatformIDs(16, platforms, &numPlatforms));

std::cout << "find " << numPlatforms << " available platforms" << std::endl;

for (cl_uint i = 0; i < numPlatforms; i++)
{
	char platformName[128];

	CK(clGetPlatformInfo(platforms[i], CL_PLATFORM_NAME, 128, platformName, nullptr));

	std::cout << "platform " << i << " " << platformName << std::endl;
}

cl_device_id device;

cl_uint numDevices = 0;

CK(clGetDeviceIDs(platforms[0], CL_DEVICE_TYPE_ALL, 1, &device, &numDevices));

{
	char deviceName[256];

	CK(clGetDeviceInfo(device, CL_DEVICE_NAME, 256, deviceName, nullptr));

	std::cout << "device name " << deviceName << std::endl;
}

{
	cl_uint computeUnit;

	CK(clGetDeviceInfo(device, CL_DEVICE_MAX_COMPUTE_UNITS, sizeof(cl_uint), &computeUnit, nullptr));

	std::cout << "device max compute unit " << computeUnit << std::endl;
}

{
	cl_uint clockFrequency;

	CK(clGetDeviceInfo(device, CL_DEVICE_MAX_CLOCK_FREQUENCY, sizeof(cl_uint), &clockFrequency, nullptr));

	std::cout << "device max clock frequency " << clockFrequency << std::endl;
}

{
	cl_ulong memorySize;

	CK(clGetDeviceInfo(device, CL_DEVICE_GLOBAL_MEM_SIZE, sizeof(cl_ulong), &memorySize, nullptr));

	std::cout << "device memory size " << memorySize / 1024 / 1024 << " megabytes" << std::endl;
}

cl_int err;

cl_context context = clCreateContext(nullptr, 1, &device, nullptr, nullptr, &err);

if (err != CL_SUCCESS)
{
	std::cout << "create context failed with " << err << std::endl;

	__debugbreak();
}

cl_command_queue queue = clCreateCommandQueueWithProperties(context, device, nullptr, &err);

if (err != CL_SUCCESS)
{
	std::cout << "create command queue failed with " << err << std::endl;

	__debugbreak();
}
```

# 内存对象的初始化
&emsp;&emsp;**内存对象**的初始化包括**缓冲**和**图像**的初始化。**缓冲**只是一定大小的内存，**内核**可以读写**缓冲**，并且可以以任何方式看待它，比如把它当作一个结构体数组。**图像**是2D或3D的有格式数据结构，**内核**可以通过**read_image**和**write_image**来读写图像，这里要注意的是读写不能同时进行。**缓冲**的初始化还是挺简单的，这里以**图像**的初始化为例，相关的代码如下。
```
const uint32_t imageWidth = 7680;

const uint32_t imageHeight = 4800;

cl_image_format fmt = {};
fmt.image_channel_order = CL_RGBA;
fmt.image_channel_data_type = CL_UNORM_INT8;

cl_image_desc desc = {};
desc.image_type = CL_MEM_OBJECT_IMAGE2D;
desc.image_width = imageWidth;
desc.image_height = imageHeight;

cl_mem outputImage = clCreateImage(context, CL_MEM_WRITE_ONLY, &fmt, &desc, nullptr, &err);
```
它的初始化需要指定**context**、**cl_mem_flags**、**host_ptr**以及填充**cl_image_format**和**cl_image_desc**这两个结构体。

- **cl_mem_flags**：这个是用于初始化的内存标志，有如下可选
    - **CL_MEM_READ_WRITE**：**设备**可读写
    - **CL_MEM_WRITE_ONLY**：**设备**只能写入
    - **CL_MEM_READ_ONLY**：**设备**只能读取
    - **CL_MEM_USE_HOST_PTR**：被启用时**OpenCL**会分配设备内存而且会利用**host_ptr**指向的内存区域，在**内核**执行计算之前**host_ptr**的数据会被拷贝到被分配的设备内存。如果想要让**主机**读取被分配的设备内存，可以使用**clEnqueueMapBuffer**这类**API**让设备内存传输到**host_ptr**指向的内存区域，从而让**主机**读写**内存对象**，它的原理是通过**PCIe**实现的**DMA**数据传输，速度还是挺快的
    ![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330233155091-533446202.png)
    - **CL_MEM_ALLOC_HOST_PTR**：被启用时**OpenCL**会分配**主机**可访问的设备内存，当**主机**和**设备**使用的是同一块内存区域时，这个时候**主机**和**设备**共享相同的设备内存，此时**主机**使用**clEnqueueMapBuffer**这类**API**读写被分配的设备内存时，几乎无花费，这个标志主要用于可以共享内存的**设备**例如**ARM Mali**
    ![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330233213039-1884043799.png)
    - **CL_MEM_COPY_HOST_PTR**：被启用时**OpenCL**会分配设备内存，并用**host_ptr**指向的内存初始化被分配的设备内存，此时**host_ptr**不能为**nullptr**
    - **CL_MEM_HOST_WRITE_ONLY**：**主机**只能写入
    - **CL_MEM_HOST_READ_ONLY**：**主机**只能读取

- **cl_image_format**
    - **image_channel_order**：通道顺序，包括通道数量以及通道布局比如**CL_RGBA**
    - **image_channel_data_type**：通道数据类型比如**CL_UNORM_INT8**

- **cl_image_desc**
    - **image_type**：图像类型，可以是以下几种
        - **CL_MEM_OBJECT_IMAGE1D**
        - **CL_MEM_OBJECT_IMAGE1D_BUFFER**
        - **CL_MEM_OBJECT_IMAGE1D_ARRAY**
        - **CL_MEM_OBJECT_IMAGE2D**
        - **CL_MEM_OBJECT_IMAGE2D_ARRAY**
        - **CL_MEM_OBJECT_IMAGE3D**

    - **image_width**：图像宽度
    - **image_height**：图像高度
    - **image_depth**：图像深度
    - **image_array_size**：图像数组的大小
    - **image_row_pitch**：图像行间距大小
    - **image_slice_pitch**：图像切片大小
    - **num_mip_levels**：层级数量一般为0，需要查询**cl_khr_mipmap_image**扩展是否被支持
    - **num_samples**：必须为0
    - **mem_object**：可以用来引用有效的**缓冲**和**图像**内存对象

## 内存迁移
&emsp;&emsp;上面就是**内存对象**初始化比较简略的介绍，官网上有更加详细的说明，这里介绍实在是太麻烦了。初始化之后其实要注意的一点就是内存迁移，之前说过编程者要手动调用**API**进行内存迁移，迁移方向可以是在**内存对象**与**主机内存**之间或者是**内存对象**与**内存对象**之间，可以通过**clEnqueueRead\*\***、**clEnqueueWrite\*\***、**clEnqueueCopy\*\***、**clEnqueueMap\*\***等**API**完成。

# 执行对象的初始化
&emsp;&emsp;接下来是**执行对象**的初始化，包括**程序对象**的初始化以及**内核对象**的初始化。

**程序对象**包括：
- 程序源或者预编译的二进制代码
- 一系列**设备**以及与**设备**对应的可执行程序
- 一系列**内核对象**

**内核对象**包括：
- 一个通过**内核修饰词**（**Kernel Qualifier**）指定的**内核**
- 与**内核**对应的参数

这里要注意的**内核对象**只能在可执行程序被编译后创建，**执行对象**初始化的代码如下


创建**程序**要求提供一个**上下文**以及程序源或预编译的二进制代码，提供的**上下文**有可能在编译**程序**时会用到
```
const std::string source = readAllText("Fractal.cl");

const char* str[1] = { source.c_str() };

cl_program program = clCreateProgramWithSource(context, 1, str, nullptr, &err);

if (err != CL_SUCCESS)
{
	std::cout << "create program failed with " << err << std::endl;

	__debugbreak();
}
```

为了编译**程序**，我们得提交一系列会用到**程序**中**内核**的**设备**，这样的话就能为不同**设备**编译可执行程序。当没有**设备**提交时，这个时候会为创建**程序**时提供的**上下文**中的所有**设备**编译可执行程序
```
err = clBuildProgram(program, 1, &device, nullptr, nullptr, nullptr);

if (err != CL_SUCCESS)
{
	std::cout << "build program failed with " << err << std::endl;

	__debugbreak();
}

char log[10240] = "";

err = clGetProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, sizeof(log), log, nullptr);

if (err != CL_SUCCESS)
{
	std::cout << "get program build info failed with " << err << std::endl;

	__debugbreak();
}

std::cout << log << std::endl;
```

编译完**程序**后我们可以取出**内核对象**
```
cl_kernel kernel = clCreateKernel(program, "FractalCompute", &err);

if (err != CL_SUCCESS)
{
	std::cout << "create kernel failed with " << err << std::endl;

	__debugbreak();
}
```

# 任务提交
&emsp;&emsp;完成所有的准备工作后，接着我们可以进行任务提交，相关的代码如下

首先设置**内核对象**的参数
```
CK(clSetKernelArg(kernel, 0, sizeof(outputImage), &outputImage));
```

接着定义**全局维度**
```
size_t globalDimension[3] = { imageWidth,imageHeight,1 };
```

然后入列计算
```
cl_event completionEvent;

CK(clEnqueueNDRangeKernel(queue, kernel, 2, nullptr, globalDimension, nullptr, 0, nullptr, &completionEvent));
```

接下来我们调用**clEnqueueReadImage**拷贝图像数据到**pixels**下的**主机内存**，**clEnqueueNDRangeKernel**会返回一个事件，我们可以让**clEnqueueReadImage**等待这个事件完成。当**clEnqueueReadImage**的**blocking_read**参数为**CL_TRUE**时**clEnqueueReadImage**会阻塞线程直到拷贝完成
```
uint8_t* pixels = new uint8_t[imageWidth * imageHeight * 4];

size_t origin[3] = { 0,0,0 };

size_t region[3] = { imageWidth,imageHeight,1 };

CK(clEnqueueReadImage(queue, outputImage, CL_TRUE, origin, region, 0, 0, pixels, 1, &completionEvent, nullptr));
```

最后把**pixels**中的数据写入到**Fractal.png**中，还有别忘了释放手动分配的内存以及各种**OpenCL对象**
```
stbi_write_png("Fractal.png", imageWidth, imageHeight, 4, pixels, imageWidth * 4);

delete[] pixels;

CK(clReleaseEvent(completionEvent));

CK(clReleaseMemObject(outputImage));

CK(clReleaseKernel(kernel));

CK(clReleaseProgram(program));

CK(clReleaseCommandQueue(queue));

CK(clReleaseContext(context));

CK(clReleaseDevice(device));
```

# OpenCL C 编程语言
&emsp;&emsp;上述就是比较简略的**OpenCL对象**初始化以及如何调用**设备**计算的流程，除了这些之外另外一个要了解的就是**OpenCL**使用的**OpenCL C**编程语言，这个非常重要因为涉及到**内核**的编写。它源自**C99**标准，但是没有**C99**头文件、函数指针、回调、可变长度的数组还有位域等特性，不过它为并行增加了**工作项**、**工作组**、**矢量类型**、**同步**，而且还有**地址空间修饰词**（**Address Space Qualifier**）、优化的图像读取以及许多内建函数，下面挑点重要的地方讲讲。

&emsp;&emsp;每个**工作项**都有独特的三个**ID**分别为**全局ID**、**组内ID**、**组ID**可以分别通过**get_global_id**、**get_local_id**、**get_group_id**来获取，获取这些**ID**能让**工作项**知道该负责哪些数据元素的输出。假设要把一个数组平方化，下图是个比较形象的说明
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330230237343-728684310.png)
此外每个**工作项**都能通过**get_global_size**、**get_local_size**、**get_num_groups**来分别获取**全局维度**、**本地维度**、**工作组数目**。

**OpenCL C**支持的数据类型包括：

- 标量类型：
    - **char**、**uchar**、**short**、**ushort**、**int**、**uint**、**long**、**ulong**、**bool**、**intptr_t**、**ptrdiff_t**、**size_t**、**uintptr_t**、**void**、**half**......

- 图像类型：
    - **image2d_t**、**image3d_t**、**sampler_t**

- 矢量类型，支持长度**2**，**3**，**4**，**8**，**16**包括：
    - **char2**、**ushort4**、**int8**、**float16**......

矢量可以用如下方式初始化
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330225559515-397650378.png)

对矢量的操作也很简单例如
```
float2 ComplexSqr(const float2 c)
{
    return (float2)(c.x * c.x - c.y * c.y, 2.0f * c.x * c.y);
}
```

&emsp;&emsp;关于类型转换有要注意的几点，首先**OpenCL C**支持多方式的类型转换比如四舍五入到最近的偶数或者四舍五入到最近的数，其次隐式类型转换是不被允许的，你得调用```convert_destType<_sat><_roundingMode>```这种函数来手动进行类型转换
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330221155208-1018335646.png)
除了类型转换你还可以进行如下的重新解释转换，就好比C++中的**reinterpret_cast**
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330231756711-1481778048.png)

&emsp;&emsp;之前已经提到了，在**设备**内部的内存有**全局内存**、**常量内存**、**本地内存**、**私有内存**，这些类型的内存分别得用**global**、**constant**、**local**、**private**关键字来指定，下面有要注意的几点：
1. **内核**指针参数的**内存空间**必须从**global**、**local**、**constant**这三个修饰词中选择
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330214527614-759680826.png)
2. 默认的**内存空间**是**私有内存**
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330214753630-438521088.png)
3. **image2d_t**和**image3d_t**数据类型永远在**全局内存**空间中
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330214606168-1761586773.png)
4. **程序**的全局变量必须在**常量内存**空间中
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330214631170-963421086.png)
5. 转换到不同的**内存空间**是未定义行为
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330214729974-1316504587.png)

&emsp;&emsp;**工作组**内的同步依赖**屏障**以及**内存屏障**，得通过**OpenCL C**中的内建函数**barrier**、**mem_fence**、**read_mem_fence**、**write_mem_fence**进行。如果**内核**中存在**屏障**或者**内存屏障**的调用，那么所有**工作组**内的**工作项**都得调用用于同步的函数，而且调用的参数都得是一致的，下图这种不是所有**工作项**都会调用**屏障**的情况就有问题
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330215555516-1018754994.png)

&emsp;&emsp;优化的图像读取实际上是依赖**采样器**进行的，关于**采样器**有三个参数**过滤模式**、**寻址模式**、**是否标准化**，在编写内核代码时可以直接这样写
```
sampler_t sampler=CLK_FILTER_LINEAR|CLK_ADDRESS_CLAMP|CLK_NORMALIZED_COORDS_FALSE;
```
可以使用如下这种方式利用采样器采样图像
```
read_imagef(inputTex,sampler,pixel);
```

**OpenCL**还提供了超多的内建函数如下图所示
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330220137648-1291595006.png)

&emsp;&emsp;最后你还能通过**clGetDeviceInfo**来获取对扩展的支持例如是否支持64位整数，关于**OpenCL C**的特性实在是太多了，这里根本讲不完推荐去官网看看。下面是我写的示例**OpenCL C内核**代码
```
__constant float2 location = (float2)(0.f,0.f);

__constant float scale=0.444f;

__constant float values[6]={0.0f, 0.16f, 0.42f, 0.6425f, 0.8575f, 1.0f};

__constant float3 colors[6]=
{
    (float3)( 0.f, 7.f, 100.f ),
    (float3)( 32.f, 107.f, 203.f ),
    (float3)( 237.f, 255.f, 255.f ),
    (float3)( 255.f, 170.f, 0.f),
    (float3)( 0.f, 2.f, 0.f ),
    (float3)( 0.f, 7.f, 100.f ),
};

__constant float3 tangent[6] =
{
    (float3)( 200.f, 625.f, 643.75f ),
    (float3)( 494.231f, 597.115f, 421.875f ),
    (float3)( 434.68f, 93.6041f, -473.034f ),
    (float3)( -552.574f, -581.709f, 0.f ),
    (float3)( 0.f, -373.154f, 0.f ),
    (float3)( 0.f, 35.0877f, 701.754f )
};

float3 interpolateColor(float t)
{
    for (uint i = 0; i < 6; i++)
    {
        if (values[i + 1] >= t)
        {
            t = (t - values[i]) / (values[i + 1] - values[i]);
            
            return (2.f * pow(t, 3.f) - 3.f * pow(t, 2.f) + 1.f) * colors[i] +
					(pow(t, 3.f) - 2.f * pow(t, 2.f) + t) * (values[i + 1] - values[i]) * tangent[i] +
					(-2.f * pow(t, 3.f) + 3.f * pow(t, 2.f)) * colors[i + 1] +
					(pow(t, 3.f) - pow(t, 2.f)) * (values[i + 1] - values[i]) * tangent[i + 1];
        }
    }

    return (float3)(0.0f, 0.0f, 0.0f);
}

#define MAXITERATION 500

#define ESCAPEBOUNDARY 16.0f

float2 ComplexSqr(const float2 c)
{
    return (float2)(c.x * c.x - c.y * c.y, 2.0f * c.x * c.y);
}

float2 ComplexMul(const float2 a, const float2 b)
{
    return (float2)(a.x * b.x - a.y * b.y, a.y * b.x + a.x * b.y);
}

float2 getLocationFromPixelCoordinate(const float2 pixelCoor)
{
    const uint2 textureSize=(uint2)(get_global_size(0),get_global_size(1));

    const float aspectRatio=convert_float(get_global_size(0))/convert_float(get_global_size(1));

    return ((pixelCoor+(float2)0.5f)/convert_float2(textureSize)-(float2)0.5f)*(float2)(2.2f*aspectRatio,2.2f)*scale+location;
}

float3 getColorAt(const float2 originPosition)
{
    float2 z=originPosition;

    float2 dz=(float2)(1.0f,0.0f);

    const float2 c=(float2)(-0.5251993f, -0.5251993f);

    uint i=0;

    uint reason=0;

    const float pixelSize=1e-8;

    float power=1.0f;

    for(;i<MAXITERATION;i++)
    {
        z=ComplexSqr(z)+c;

        dz=2.0f*ComplexMul(z,dz);

        power=power*2.0f;

        if(dot(z,z)<dot(dz*pixelSize,dz*pixelSize))
        {
            reason=1;

            break;
        }

        if(dot(z,z)>ESCAPEBOUNDARY)
        {
            reason=2;

            break;
        }
    }

    const float smoothed_i = convert_float(i) - log2(max(1.0f, log2(length(z))));

    const float iter_ratio = clamp(smoothed_i / convert_float(MAXITERATION),0.f,1.f);

    float3 color = (float3)(0.0f, 0.0f, 0.0f);
    
    if (reason == 1)
    {
        color = interpolateColor(iter_ratio*0.4f)/255.0f;
    }
    else if (reason == 2)
    {
        const float dist = 2.0f * log(length(z)) * length(z) / length(dz);
    
        const float t = clamp(tanh(dist * 1080.0f),0.f,1.f);

        color = interpolateColor(iter_ratio*0.7f)/255.0f*t;
    }
    
    float fadeFactor = (1.0f - log(dot(z, z)) / power);

    if(fadeFactor<0.f)
    {
        fadeFactor=0.f;
    }
    
    color *= fadeFactor;

    return color;
}

__constant float exposure=1.9f;

__constant float gamma=1.2f;

float3 ACESToneMapping(float3 color)
{
    const float A = 2.51;
    
    const float B = 0.03;
    
    const float C = 2.43;
    
    const float D = 0.59;
    
    const float E = 0.14;

    color *= exposure;
    
    return (color * (A * color + B)) / (color * (C * color + D) + E);
}

#define NUMSAMPLELOCATION 64

kernel void FractalCompute(write_only image2d_t img)
{
    const float2 pixelLocation=(float2)(convert_float(get_global_id(0)),convert_float(get_global_id(1)));

    float3 sumColor=(float3)(0.f,0.f,0.f);

    const float interpolateMin=-0.5f+1.f/(2.f*convert_float(NUMSAMPLELOCATION));

    const float interpolateMax=0.5f-1.f/(2.f*convert_float(NUMSAMPLELOCATION));

    for(uint i=0;i<NUMSAMPLELOCATION;i++)
    {
        for(uint j=0;j<NUMSAMPLELOCATION;j++)
        {
            const float dx=mix(interpolateMin,interpolateMax,convert_float(i)/convert_float(NUMSAMPLELOCATION-1));

            const float dy=mix(interpolateMin,interpolateMax,convert_float(j)/convert_float(NUMSAMPLELOCATION-1));

            sumColor+=getColorAt(getLocationFromPixelCoordinate(pixelLocation+(float2)(dx,dy)));
        }
    }

    sumColor/=convert_float(NUMSAMPLELOCATION)*convert_float(NUMSAMPLELOCATION);

    sumColor=ACESToneMapping(sumColor);

    sumColor=pow(sumColor,1.f/gamma);

    write_imagef(img,convert_int2(pixelLocation),(float4)(sumColor,1.0f));
}
```

运行结果如下:
![img](https://img2023.cnblogs.com/blog/2774734/202503/2774734-20250330215238897-2047570637.png)