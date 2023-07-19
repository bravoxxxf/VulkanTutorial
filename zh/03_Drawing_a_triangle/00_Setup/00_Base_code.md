## General structure

直接上代码：

```c++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

我们首先包含了Vulkan的头文件, 它提供了函数、数据结构和枚举对象.  `stdexcept` 和 `iostream` 头文件包含了错误的输出和传播. `cstdlib`
头文件提供了 `EXIT_SUCCESS` 和 `EXIT_FAILURE` 宏.

这个程序本身被封装在一个类中，我们将把Vulkan对象作为私有类成员进行存储，并添加函数来初始化每个对象，这些函数将从函数中调用。一旦所有准备工作完成，我们进入主循环以开始渲染帧。稍后我们将填写`mainLoop`函数，其中包含一个循环，直到窗口关闭为止。一旦窗口关闭并且`mainLoop`返回，我们将确保在`cleanup`函数中释放我们使用的资源。

如果在执行过程中发生任何致命错误，我们将抛出一个带有描述性消息的`std::runtime_error`异常，该异常将传播回`main`函数并打印到命令提示符上。为了处理各种标准异常类型，我们还捕获更通用的`std::exception`异常。我们很快将处理的一个错误示例是发现某个所需的扩展不受支持。

在此之后的每个章节大致都会添加一个新函数，该函数将从`initVulkan`中调用，并且会向私有类成员添加一个或多个需要在`cleanup`结束时释放的新的Vulkan对象。

## 资源管理

就像使用`malloc`分配的每个内存块都需要调用`free`一样，我们创建的每个Vulkan对象在不再需要时都需要显式地销毁。在C++中，可以使用[RAII（资源获取即初始化）](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)或`<memory>`头文件中提供的智能指针来执行自动资源管理。然而，在本教程中，我选择在Vulkan对象的分配和释放方面明确一些。毕竟，Vulkan的特点是明确每个操作，以避免错误，因此明确对象的生命周期有助于学习API的工作原理。

在完成本教程后，您可以通过编写C++类来实现自动资源管理，这些类在其构造函数中获取Vulkan对象，并在其析构函数中释放它们，或者根据您的所有权要求为`std::unique_ptr`或`std::shared_ptr`提供自定义删除器。对于较大的Vulkan程序，推荐使用RAII模型，但是出于学习目的，了解幕后发生的情况总是很有好处。

Vulkan对象可以直接使用诸如`vkCreateXXX`的函数创建，也可以通过其他对象使用诸如`vkAllocateXXX`的函数进行分配。确保对象在任何地方都不再使用后，您需要使用相应的`vkDestroyXXX`和`vkFreeXXX`函数将其销毁。这些函数的参数通常因不同类型的对象而异，但它们都共享一个参数：`pAllocator`。这是一个可选参数，允许您指定自定义内存分配器的回调函数。在本教程中，我们将忽略此参数，并始终将`nullptr`作为参数传递。

## 集成 GLFW

如果您想将Vulkan用于屏幕外渲染，而不创建窗口，它仍然可以正常工作，但实际上展示一些内容会更加令人兴奋！首先将`#include <vulkan/vulkan.h>`行替换为：

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

That way GLFW will include its own definitions and automatically load the Vulkan
header with it. Add a `initWindow` function and add a call to it from the `run`
function before the other calls. We'll use that function to initialize GLFW and
create a window.

```c++
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```

`initWindow`函数中的第一个调用应该是`glfwInit()`，它用于初始化GLFW库。由于GLFW最初设计用于创建OpenGL上下文，我们需要通过后续的调用告诉它不要创建OpenGL上下文：

```c++
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

因为处理调整大小的窗口需要特殊的注意，我们稍后会详细介绍，暂时禁用它，使用另一个窗口提示调用：

```c++
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

现在只剩下创建实际窗口了。添加一个`GLFWwindow* window;`私有类成员来存储对它的引用，并使用以下代码初始化窗口：

```c++
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

前三个参数指定窗口的宽度、高度和标题。第四个参数允许您可选择地指定要在其上打开窗口的监视器，而最后一个参数只与OpenGL有关。

最好使用常量而不是硬编码的宽度和高度值，因为在将来我们将多次引用这些值。我已经在`HelloTriangleApplication`类定义之前添加了以下行：

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;
```

并用以下代码替换了窗口创建调用：

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

现在，您的`initWindow`函数应该如下所示：

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

为了在发生错误或窗口关闭之前保持应用程序运行，我们需要在`mainLoop`函数中添加一个事件循环，代码如下所示：

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

这段代码是非常清晰明了的。它一直循环检查事件，直到用户手动关闭窗口。 这也是我们稍后用来渲染单个帧的循环。

一旦窗口关闭。我们需要通过销毁它来释放资源，终止GLFW程序本身。这将是我们第一个清理代码:

```c++
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

[C++ code](/code/00_base_code.cpp)
