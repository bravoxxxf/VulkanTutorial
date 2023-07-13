在这一章中，我们将为您设置开发Vulkan应用程序的环境并安装一些有用的库。除了编译器外，我们将使用的所有工具都与Windows、Linux和MacOS兼容，但安装它们的步骤有所不同，因此在这里分别进行描述。

## Windows

如果您正在为Windows开发，则我将假设您正在使用Visual Studio来编译代码。为了完全支持C++17，您需要使用Visual Studio 2017或2019。下面概述的步骤适用于VS 2017。

### Vulkan SDK

开发Vulkan应用程序所需的最重要组件是SDK。它包括头文件、标准验证层、调试工具和Vulkan函数的加载器。加载器在运行时在驱动程序中查找函数，类似于OpenGL的GLEW（如果您对此熟悉的话）。


SDK可以从这里下载 [the LunarG website](https://vulkan.lunarg.com/). 你无需登录账号，但登陆后可以获得更多的信息。

![](/images/vulkan_sdk_download_buttons.png)

按照安装过程继续进行，并注意SDK的安装位置。我们将首先验证您的显卡和驱动程序是否正确支持Vulkan。转到您安装SDK的目录，打开`Bin`目录并运行`vkcube.exe`演示。您应该会看到以下内容：

![](/images/cube_demo.png)

如果您收到错误消息，请确保您的驱动程序已经更新到最新版本，包括安装了Vulkan运行时，并且您的显卡得到支持。请参考[introduction chapter](!en/Introduction)获取主要供应商提供的驱动程序链接。

在这个目录中还有另一个在开发中非常有用的程序。`glslangValidator.exe`和`glslc.exe`程序用于将可读的[GLSL](https://en.wikipedia.org/wiki/OpenGL_Shading_Language)着色器编译成字节码。我们将在[shader modules](!en/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules)章节中详细介绍这个内容。`Bin`目录中还包含了Vulkan加载器和验证层的二进制文件，而`Lib`目录则包含了库文件。

最后，还有一个`Include`目录，其中包含了Vulkan的头文件。您可以随意浏览其他文件，但在本教程中我们不需要它们。

### GLFW

如前所述，Vulkan本身是一个与平台无关的API，并不包含创建用于显示渲染结果的窗口的工具。为了从Vulkan的跨平台优势中受益，并避免Win32的困扰，我们将使用[GLFW库](http://www.glfw.org/)来创建窗口，它支持Windows、Linux和MacOS。还有其他可用于此目的的库，如[SDL](https://www.libsdl.org/)，但GLFW的优点在于它除了窗口创建之外，还将Vulkan中的其他一些特定于平台的内容进行了抽象。

您可以在[官方网站](http://www.glfw.org/download.html)上找到GLFW的最新版本。在本教程中，我们将使用64位的二进制文件，但您当然也可以选择以32位模式构建。在这种情况下，请确保链接到`Lib32`目录中的Vulkan SDK二进制文件，而不是`Lib`目录。下载后，将存档文件提取到一个方便的位置。我选择在文档目录下的Visual Studio目录中创建一个`Libraries`目录。

![](/images/glfw_directory.png)

### GLM

与DirectX 12不同，Vulkan并不包含用于线性代数操作的库，因此我们需要下载一个库。[GLM](http://glm.g-truc.net/)是一个很好的库，专为与图形API一起使用，并且通常与OpenGL一起使用。

GLM是一个仅包含头文件的库，所以只需下载[最新版本](https://github.com/g-truc/glm/releases)并将其存储在一个方便的位置。现在，您应该有类似下面的目录结构：

![](/images/library_directory.png)

### 设置 Visual Studio

现在您已经安装了所有的依赖项，我们可以为Vulkan设置一个基本的Visual Studio项目，并编写一些代码以确保一切正常。

启动Visual Studio，并通过输入一个名称并点击“OK”来创建一个新的“Windows Desktop Wizard”项目。

![](/images/vs_new_cpp_project.png)

确保选择`Console Application (.exe)`作为应用程序类型，这样我们就有一个可以打印调试消息的地方，并勾选`Empty Project`以防止Visual Studio添加样板代码。

![](/images/vs_application_settings.png)

按“确定”创建项目并添加 C++ 源文件。 你应该已经知道如何执行此操作，但为了完整起见，此处包含了这些步骤。

![](/images/vs_new_item.png)

![](/images/vs_new_source_file.png)

现在将以下代码添加到文件中。暂时不用担心理解它；我们只是确保您能够编译和运行Vulkan应用程序。在下一章中，我们将从头开始。

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

现在让我们配置项目以消除错误。打开项目属性对话框，并确保选择了“所有配置”，因为大多数设置适用于“Debug”和“Release”模式。

![](/images/vs_open_project_properties.png)

![](/images/vs_all_configs.png)

Go to `C++ -> General -> Additional Include Directories` and press `<Edit...>`
in the dropdown box.

![](/images/vs_cpp_general.png)

Add the header directories for Vulkan, GLFW and GLM:

![](/images/vs_include_dirs.png)

Next, open the editor for library directories under `Linker -> General`:

![](/images/vs_link_settings.png)

And add the locations of the object files for Vulkan and GLFW:

![](/images/vs_link_dirs.png)

Go to `Linker -> Input` and press `<Edit...>` in the `Additional Dependencies`
dropdown box.

![](/images/vs_link_input.png)

Enter the names of the Vulkan and GLFW object files:

![](/images/vs_dependencies.png)

And finally change the compiler to support C++17 features:

![](/images/vs_cpp17.png)

您现在可以关闭项目属性对话框。如果您做得一切正确，那么您不再会看到代码中出现任何错误的标记。

最后，确保你选择的是64位模式编译：

![](/images/vs_build_mode.png)

Press `F5` to compile and run the project and you should see a command prompt
and a window pop up like this:

![](/images/vs_test_window.png)

扩展数量应该是非零的。 恭喜，你已经准备好了 [playing with Vulkan](!en/Drawing_a_triangle/Setup/Base_code)!