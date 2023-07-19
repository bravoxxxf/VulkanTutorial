由于Vulkan是一个与平台无关的API，它不能直接与窗口系统交互。为了在屏幕上呈现结果并建立Vulkan与窗口系统之间的连接，我们需要使用WSI（Window System Integration）扩展。在本章中，我们将讨论第一个WSI扩展，即`VK_KHR_surface`。它暴露了一个`VkSurfaceKHR`对象，表示用于呈现渲染图像的抽象类型的表面。在我们的程序中，这个表面将由我们已经用GLFW打开的窗口支持。

`VK_KHR_surface`扩展是一个实例级扩展，实际上我们已经启用它，因为它包含在`glfwGetRequiredInstanceExtensions`返回的列表中。该列表还包括一些其他我们将在接下来的几章中使用的WSI扩展。

窗口表面需要在实例创建后立即创建，因为它实际上可以影响物理设备的选择。我们推迟了这一步是因为窗口表面是渲染目标和呈现的一部分，解释这些内容会使基本设置变得混乱。还应该注意的是，窗口表面在Vulkan中是完全可选的组件，如果您只需要离屏渲染，Vulkan允许您创建一个不可见的窗口（OpenGL是必须的）。

## 窗口表面的创建

首先，在调试回调函数的下面添加一个名为`surface`的类成员。

```c++
VkSurfaceKHR surface;
```

尽管`VkSurfaceKHR`对象及其使用是与平台无关的，但它的创建不是，因为它取决于窗口系统的详细信息。例如，在Windows上，它需要`HWND`和`HMODULE`句柄。因此，扩展中有一个与平台相关的附加部分，在Windows上称为`VK_KHR_win32_surface`，它也会自动包含在`glfwGetRequiredInstanceExtensions`的列表中。

我将演示如何使用这个特定于平台的扩展来在Windows上创建一个表面，但在本教程中我们实际上不会使用它。使用GLFW这样的库，然后继续使用特定于平台的代码没有任何意义。GLFW实际上有`glfwCreateWindowSurface`函数，它为我们处理了平台差异。不过，在我们开始依赖它之前，了解它在幕后的工作原理还是很有好处的。

要访问本地平台函数，您需要在文件顶部更新包含的头文件:

```c++
#define VK_USE_PLATFORM_WIN32_KHR
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
#define GLFW_EXPOSE_NATIVE_WIN32
#include <GLFW/glfw3native.h>
```

因为窗口表面是一个Vulkan对象，所以它带有一个需要填充的`VkWin32SurfaceCreateInfoKHR`结构体。它有两个重要的参数：`hwnd`和`hinstance`。它们是窗口和进程的句柄。

```c++
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

使用`glfwGetWin32Window`函数可以从GLFW窗口对象中获取原始的`HWND`句柄。`GetModuleHandle`函数返回当前进程的`HINSTANCE`句柄。

之后，可以使用`vkCreateWin32SurfaceKHR`函数创建表面，其中包括实例、表面创建细节、自定义分配器以及用于存储表面句柄的变量。从技术上讲，这是一个WSI扩展函数，但它是如此常用，以至于标准的Vulkan加载器将其包含在内，因此与其他扩展不同，您不需要显式加载它。

```c++
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

对于其他平台（如Linux），过程类似，`vkCreateXcbSurfaceKHR`函数接受XCB连接和窗口作为创建细节参数，适用于X11窗口系统。

`glfwCreateWindowSurface`函数对于每个平台执行的操作完全相同，只是具体实现有所不同。我们现在将把它整合到我们的程序中。在`initVulkan`函数中的实例创建和`setupDebugMessenger`之后添加一个名为`createSurface`的函数，并从该函数调用它。

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

GLFW的调用接受简单的参数而不是一个结构体，这使得函数的实现非常直观简单:

```c++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

参数包括`VkInstance`、GLFW窗口指针、自定义分配器以及指向`VkSurfaceKHR`变量的指针。它只是简单地传递相关平台调用的`VkResult`结果。GLFW并没有提供一个专门销毁表面的函数，但可以通过原始API很容易地完成这一操作:

```c++
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
    }
```


在销毁实例之前，确保先销毁表面。

## 查询是否支持呈现

尽管Vulkan的实现可能支持窗口系统集成，但并不意味着系统中的每个设备都支持。因此，我们需要扩展`isDeviceSuitable`函数，以确保设备能够向我们创建的表面呈现图像。由于呈现是队列特定的功能，实际上问题是找到一个支持向我们创建的表面呈现的队列族。

实际上，支持绘制命令的队列族和支持呈现的队列族可能不重叠。因此，我们必须考虑到可能存在一个独立的呈现队列，修改`QueueFamilyIndices`结构以反映这一点:

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

接下来，我们将修改`findQueueFamilies`函数，以查找具有向我们的窗口表面呈现功能的队列族。用于检查的函数是`vkGetPhysicalDeviceSurfaceSupportKHR`，它接受物理设备、队列族索引和表面作为参数。在与`VK_QUEUE_GRAPHICS_BIT`相同的循环中添加对它的调用：

```c++
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

然后简单地检查布尔值的值并存储呈现队列族的索引：

```c++
if (presentSupport) {
    indices.presentFamily = i;
}
```

请注意，最终这两个队列族很可能是相同的，但在整个程序中，我们将把它们视为独立的队列，以实现统一的处理。然而，您可以添加逻辑来明确优先选择同时支持绘制和呈现的物理设备，以提高性能。

## 创建呈现队列

现在只剩下修改逻辑设备创建过程来创建呈现队列并检索`VkQueue`句柄。添加一个成员变量来存储该句柄：

```c++
VkQueue presentQueue;
```

接下来，我们需要使用多个`VkDeviceQueueCreateInfo`结构来从这两个队列族创建队列。一个优雅的方法是创建一个包含所有必要队列的唯一队列族的集合:

```c++
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

然后修改`VkDeviceCreateInfo`，将其指向这个向量：

```c++
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

如果队列族是相同的，我们只需要将其索引传递一次。最后，添加一个调用来检索队列句柄:

```c++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

如果队列族是相同的，那么这两个句柄现在很可能具有相同的值。在下一章中，我们将研究交换链以及它们如何让我们有能力向表面呈现图像。

[C++ code](/code/05_window_surface.cpp)
