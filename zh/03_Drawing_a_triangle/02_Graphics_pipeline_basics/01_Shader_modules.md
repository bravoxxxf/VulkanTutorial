与早期的API不同，Vulkan中的着色器代码必须使用字节码格式来指定，而不是像[GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)和[HLSL](https://en.wikipedia.org/wiki/High-Level_Shading_Language)这样的人类可读的语法。这个字节码格式称为[SPIR-V](https://www.khronos.org/spir)，并且被设计用于Vulkan和OpenCL（两个Khronos API）。它是一种可以用于编写图形和计算着色器的格式，但在本教程中，我们将专注于Vulkan图形管线中使用的着色器。

使用字节码格式的优势在于，由GPU供应商编写的将着色器代码转换为本地代码的编译器要简单得多。过去的经验表明，使用GLSL等人类可读的语法，一些GPU供应商对标准的解释相当灵活。如果您不小心在来自这些供应商之一的GPU上编写非平凡的着色器，则可能会因为语法错误而导致其他供应商的驱动程序拒绝您的代码，或者更糟糕的是，由于编译器错误而使您的着色器运行不同。使用像SPIR-V这样直接的字节码格式，这些问题有望得到避免。

然而，这并不意味着我们需要手动编写这个字节码。Khronos发布了自己的供应商无关的编译器，将GLSL编译成SPIR-V。该编译器旨在验证您的着色器代码是否完全符合标准，并生成一个SPIR-V二进制文件，您可以随程序一起提供。您还可以将此编译器作为库包含在内，以在运行时生成SPIR-V，但在本教程中我们不会这样做。虽然我们可以通过`glslangValidator.exe`直接使用此编译器，但我们将使用Google提供的`glslc.exe`。 `glslc` 的优势在于它使用与GCC和Clang等著名编译器相同的参数格式，并包含一些额外的功能，例如*includes*。这两个编译器都已包含在Vulkan SDK中，因此您无需下载任何额外的内容。

GLSL是一种具有C风格语法的着色语言。用它编写的程序有一个`main`函数，对每个对象都会调用一次。GLSL不像传统语言那样使用参数作为输入和返回值作为输出，而是使用全局变量来处理输入和输出。该语言包括许多用于辅助图形编程的功能，如内置的向量和矩阵基元。它包含用于交叉乘积、矩阵-向量乘积和绕向量反射等操作的函数。向量类型称为`vec`，后面跟着元素的数量。例如，3D位置将存储在`vec3`中。可以通过成员`.x`访问单个分量，但也可以同时从多个分量创建新的向量。例如，表达式`vec3(1.0, 2.0, 3.0).xy`将产生`vec2`。向量的构造函数还可以使用向量对象和标量值的组合。例如，可以使用`vec3(vec2(1.0, 2.0), 3.0)`构造`vec3`。

正如上一章所述，要在屏幕上显示一个三角形，我们需要编写一个顶点着色器和一个片段着色器。接下来的两个小节将涵盖这两者的GLSL代码，并在此之后，我将向您展示如何生成两个SPIR-V二进制文件并将它们加载到程序中。

## 顶点着色器

顶点着色器处理每个传入的顶点。它以属性作为输入，如世界位置、颜色、法线和纹理坐标。输出是剪裁坐标中的最终位置以及需要传递给片段着色器的属性，如颜色和纹理坐标。这些值将在光栅化器中被插值到片段上，以产生平滑的渐变效果。

*剪裁坐标*是从顶点着色器中传出的四维向量，随后通过将整个向量除以其最后一个分量而转换为*规范化设备坐标*。这些规范化设备坐标是[齐次坐标](https://en.wikipedia.org/wiki/Homogeneous_coordinates)，将帧缓冲映射到一个[-1, 1]乘[-1, 1]的坐标系，如下图所示：

![](/images/normalized_device_coordinates.svg)

如果您之前涉足过计算机图形学，您应该对这些坐标系很熟悉。如果您之前使用过OpenGL，那么您会注意到Y坐标的符号现在被反转了。Z坐标现在使用与Direct3D中相同的范围，从0到1。

对于我们的第一个三角形，我们不会应用任何变换，而是直接将三个顶点的位置指定为规范化设备坐标，创建如下形状：

![](/images/triangle_coordinates.svg)

我们可以直接输出规范化设备坐标，将它们作为剪裁坐标从顶点着色器输出，并将最后一个分量设置为`1`。这样，剪裁坐标到规范化设备坐标的除法将不会改变任何内容。

通常，这些坐标将存储在顶点缓冲区中，但在Vulkan中创建顶点缓冲区并填充数据并不是一件简单的事情。因此，我决定在我们看到三角形在屏幕上弹出之后再来处理这一部分。在此期间，我们要做一些不太正统的事情：直接在顶点着色器中包含坐标。代码如下:

```glsl
#version 450

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

`main` 函数对每个顶点都会调用。内置的 `gl_VertexIndex` 变量包含当前顶点的索引。通常，这是一个对顶点缓冲区的索引，但在我们的例子中，它将是对硬编码顶点数据数组的索引。每个顶点的位置从着色器中的常量数组中获取，并与虚拟的 `z` 和 `w` 分量组合以生成剪裁坐标中的位置。内置变量 `gl_Position` 用作输出。

## 片段着色器

由顶点着色器生成的位置形成了屏幕上的一个区域，并被片段填充。片段着色器在这些片段上调用，以为帧缓冲区（或帧缓冲区）生成颜色和深度。一个简单的片段着色器，为整个三角形输出红色，如下所示：

```glsl
#version 450

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

`main` 函数对每个片段都会被调用，就像顶点着色器的 `main` 函数对每个顶点都会被调用一样。在 GLSL 中，颜色是具有 R、G、B 和 alpha 通道的 4 维向量，其值在 [0, 1] 范围内。与顶点着色器中的 `gl_Position` 不同，片段着色器中没有用于输出当前片段颜色的内置变量。您必须为每个帧缓冲区指定自己的输出变量，其中 `layout(location = 0)` 修饰符指定了帧缓冲区的索引。将红色写入 `outColor` 变量，该变量与索引为 `0` 的第一个（也是唯一一个）帧缓冲区相关联。

## 每顶点颜色

让整个三角形都是红色并不是很有趣，看起来像下面这样的效果会更好：

![](/images/triangle_coordinates_colors.png)

为了实现这个效果，我们需要对两个着色器进行一些更改。首先，我们需要为三个顶点中的每一个指定一个不同的颜色。顶点着色器现在应该包含一个用于颜色的数组，就像它对位置所做的那样:

```glsl
vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);
```

现在我们只需要将这些每个顶点的颜色传递给片段着色器，以便它可以将它们的插值值输出到帧缓冲区。在顶点着色器中添加一个用于颜色的输出，并在 `main` 函数中写入它：

```glsl
layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

Next, we need to add a matching input in the fragment shader:

```glsl
layout(location = 0) in vec3 fragColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

输入变量不一定要使用相同的名称，它们将通过`location`指令指定的索引进行链接。`main`函数已经修改，以输出颜色和 alpha 值。如上图所示，`fragColor`的值将自动在三个顶点之间的片段中进行插值，从而产生平滑的渐变效果。

## Compiling the shaders

在您的项目的根目录下创建一个名为`shaders`的目录，并将顶点着色器保存在名为`shader.vert`的文件中，将片段着色器保存在名为`shader.frag`的文件中。GLSL着色器没有官方的扩展名，但这两个通常用于区分它们。

`shader.vert`文件的内容应为：

```glsl
#version 450

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

`shader.frag` 的内容应该是:

```glsl
#version 450

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

现在，我们将使用`glslc`程序将这些着色器编译成SPIR-V字节码。

**Windows**

创建一个文件 `compile.bat`:

```bash
C:/VulkanSDK/x.x.x.x/Bin/glslc.exe shader.vert -o vert.spv
C:/VulkanSDK/x.x.x.x/Bin/glslc.exe shader.frag -o frag.spv
pause
```

将文件放在VulkanSDK的安装目录的 `glslc.exe` 同文件路径下。双击执行.

**Linux**

创建一个文件 `compile.sh`:

```bash
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.vert -o vert.spv
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslc shader.frag -o frag.spv
```

将文件放在VulkanSDK的安装目录的 `glslc` 同文件路径下. 使用以下命令使脚本可执行 `chmod +x compile.sh` ，然后执行它.

**平台特定指令结束**

这两个命令告诉编译器读取GLSL源文件，并使用`-o`（输出）标志输出SPIR-V字节码文件。

如果您的着色器包含语法错误，编译器将告诉您行号和问题，就像您预期的那样。例如，尝试省略一个分号，然后再次运行编译脚本。还可以尝试运行编译器而不带任何参数，以查看它支持的各种标志。例如，它还可以将字节码输出到可读格式，这样您可以查看着色器的具体操作以及在此阶段应用的任何优化。

在命令行上编译着色器是最直接的选项之一，也是我们在本教程中使用的选项，但也可以直接从您的代码中编译着色器。 Vulkan SDK包含[libshaderc](https://github.com/google/shaderc)库，该库可以从程序内部将GLSL代码编译为SPIR-V。

## Loading a shader

Now that we have a way of producing SPIR-V shaders, it's time to load them into
our program to plug them into the graphics pipeline at some point. We'll first
write a simple helper function to load the binary data from the files.

```c++
#include <fstream>

...

static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
}
```

The `readFile` function will read all of the bytes from the specified file and
return them in a byte array managed by `std::vector`. We start by opening the
file with two flags:

* `ate`: Start reading at the end of the file
* `binary`: Read the file as binary file (avoid text transformations)

The advantage of starting to read at the end of the file is that we can use the
read position to determine the size of the file and allocate a buffer:

```c++
size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
```

After that, we can seek back to the beginning of the file and read all of the
bytes at once:

```c++
file.seekg(0);
file.read(buffer.data(), fileSize);
```

And finally close the file and return the bytes:

```c++
file.close();

return buffer;
```

We'll now call this function from `createGraphicsPipeline` to load the bytecode
of the two shaders:

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");
}
```

Make sure that the shaders are loaded correctly by printing the size of the
buffers and checking if they match the actual file size in bytes. Note that the code doesn't need to be null terminated since it's binary code and we will later be explicit about its size.

## Creating shader modules

Before we can pass the code to the pipeline, we have to wrap it in a
`VkShaderModule` object. Let's create a helper function `createShaderModule` to
do that.

```c++
VkShaderModule createShaderModule(const std::vector<char>& code) {

}
```

The function will take a buffer with the bytecode as parameter and create a
`VkShaderModule` from it.

Creating a shader module is simple, we only need to specify a pointer to the
buffer with the bytecode and the length of it. This information is specified in
a `VkShaderModuleCreateInfo` structure. The one catch is that the size of the
bytecode is specified in bytes, but the bytecode pointer is a `uint32_t` pointer
rather than a `char` pointer. Therefore we will need to cast the pointer with
`reinterpret_cast` as shown below. When you perform a cast like this, you also
need to ensure that the data satisfies the alignment requirements of `uint32_t`.
Lucky for us, the data is stored in an `std::vector` where the default allocator
already ensures that the data satisfies the worst case alignment requirements.

```c++
VkShaderModuleCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
createInfo.codeSize = code.size();
createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());
```

The `VkShaderModule` can then be created with a call to `vkCreateShaderModule`:

```c++
VkShaderModule shaderModule;
if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS) {
    throw std::runtime_error("failed to create shader module!");
}
```

The parameters are the same as those in previous object creation functions: the
logical device, pointer to create info structure, optional pointer to custom
allocators and handle output variable. The buffer with the code can be freed
immediately after creating the shader module. Don't forget to return the created
shader module:

```c++
return shaderModule;
```

Shader modules are just a thin wrapper around the shader bytecode that we've previously loaded from a file and the functions defined in it. The compilation and linking of the SPIR-V bytecode to machine code for execution by the GPU doesn't happen until the graphics pipeline is created. That means that we're allowed to destroy the shader modules again as soon as pipeline creation is finished, which is why we'll make them local variables in the `createGraphicsPipeline` function instead of class members:

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");

    VkShaderModule vertShaderModule = createShaderModule(vertShaderCode);
    VkShaderModule fragShaderModule = createShaderModule(fragShaderCode);
```

The cleanup should then happen at the end of the function by adding two calls to `vkDestroyShaderModule`. All of the remaining code in this chapter will be inserted before these lines.

```c++
    ...
    vkDestroyShaderModule(device, fragShaderModule, nullptr);
    vkDestroyShaderModule(device, vertShaderModule, nullptr);
}
```

## Shader stage creation

To actually use the shaders we'll need to assign them to a specific pipeline stage through `VkPipelineShaderStageCreateInfo` structures as part of the actual pipeline creation process.

We'll start by filling in the structure for the vertex shader, again in the
`createGraphicsPipeline` function.

```c++
VkPipelineShaderStageCreateInfo vertShaderStageInfo{};
vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
```

The first step, besides the obligatory `sType` member, is telling Vulkan in
which pipeline stage the shader is going to be used. There is an enum value for
each of the programmable stages described in the previous chapter.

```c++
vertShaderStageInfo.module = vertShaderModule;
vertShaderStageInfo.pName = "main";
```

The next two members specify the shader module containing the code, and the
function to invoke, known as the *entrypoint*. That means that it's possible to combine multiple fragment
shaders into a single shader module and use different entry points to
differentiate between their behaviors. In this case we'll stick to the standard
`main`, however.

There is one more (optional) member, `pSpecializationInfo`, which we won't be
using here, but is worth discussing. It allows you to specify values for shader
constants. You can use a single shader module where its behavior can be
configured at pipeline creation by specifying different values for the constants
used in it. This is more efficient than configuring the shader using variables
at render time, because the compiler can do optimizations like eliminating `if`
statements that depend on these values. If you don't have any constants like
that, then you can set the member to `nullptr`, which our struct initialization
does automatically.

Modifying the structure to suit the fragment shader is easy:

```c++
VkPipelineShaderStageCreateInfo fragShaderStageInfo{};
fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
fragShaderStageInfo.module = fragShaderModule;
fragShaderStageInfo.pName = "main";
```

Finish by defining an array that contains these two structs, which we'll later
use to reference them in the actual pipeline creation step.

```c++
VkPipelineShaderStageCreateInfo shaderStages[] = {vertShaderStageInfo, fragShaderStageInfo};
```

That's all there is to describing the programmable stages of the pipeline. In
the next chapter we'll look at the fixed-function stages.

[C++ code](/code/09_shader_modules.cpp) /
[Vertex shader](/code/09_shader_base.vert) /
[Fragment shader](/code/09_shader_base.frag)
