Vulkan没有“默认帧缓冲”的概念，因此在我们将图像可视化到屏幕之前，它需要一个管理我们将渲染的缓冲区的基础设施。这个基础设施称为*交换链*，在Vulkan中必须显式地创建它。交换链实质上是一个等待呈现到屏幕的图像队列。我们的应用程序会获取这样的图像来绘制，然后将它返回到队列中。队列的工作原理以及从队列中呈现图像的条件取决于交换链的设置方式，但交换链的一般目的是将图像的呈现与屏幕的刷新率同步。

## 检查交换链支持

并非所有的显卡都能直接向屏幕呈现图像，原因有很多，例如它们可能专为服务器而设计，没有任何显示输出。其次，由于图像呈现与窗口系统以及与窗口相关联的表面紧密相关，它实际上并不是Vulkan核心的一部分。在查询支持后，您必须启用`VK_KHR_swapchain`设备扩展。

为此，我们首先扩展`isDeviceSuitable`函数，以检查是否支持此扩展。我们之前已经看到如何列出`VkPhysicalDevice`支持的扩展，所以这样做应该非常简单。请注意，Vulkan头文件提供了一个很好的宏`VK_KHR_SWAPCHAIN_EXTENSION_NAME`，它被定义为`VK_KHR_swapchain`。使用这个宏的优点是编译器会捕捉拼写错误。

首先声明一个所需设备扩展的列表，类似于启用的验证层的列表。

```c++
const std::vector<const char*> deviceExtensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME
};
```

接下来，创建一个名为`checkDeviceExtensionSupport`的新函数，将其作为`isDeviceSuitable`的附加检查来调用:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    bool extensionsSupported = checkDeviceExtensionSupport(device);

    return indices.isComplete() && extensionsSupported;
}

bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    return true;
}
```

修改函数体来枚举扩展并检查是否包含所有所需的扩展。

```c++
bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());

    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

我选择使用字符串的集合来表示未确认的所需扩展。这样，在枚举可用扩展的序列时，我们可以轻松地对其进行标记。当然，您也可以像在`checkValidationLayerSupport`中那样使用嵌套循环。性能的差异是无关紧要的。现在运行代码并验证您的显卡确实支持创建交换链。值得注意的是，如我们在前一章中检查的呈现队列的可用性，意味着必须支持交换链扩展。然而，仍然明确说明这些事情是很好的，而且扩展必须显式地启用。

## 启用设备扩展

使用交换链需要先启用`VK_KHR_swapchain`扩展。启用这个扩展只需要对逻辑设备创建结构做一个小的修改:

```c++
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

确保在进行修改时替换现有的代码行`createInfo.enabledExtensionCount = 0;`。

## 查询交换链支持的详细信息

仅检查是否有交换链是不够的，因为它可能与我们的窗口表面不兼容。创建交换链还涉及比实例和设备创建更多的设置，因此我们需要在继续之前查询一些更多的详细信息。

基本的表面功能有三种类型需要检查：

* 基本表面功能（交换链中图像的最小/最大数量，图像的最小/最大宽度和高度）
* 表面格式（像素格式，颜色空间）
* 可用的呈现模式

类似于`findQueueFamilies`，我们将使用一个结构体来传递这些详细信息，一旦查询完成。这三种前述的属性类型以以下结构体和结构体列表的形式出现:

```c++
struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

我们现在将创建一个名为`querySwapChainSupport`的新函数，该函数将填充这个结构体。

```c++
SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device) {
    SwapChainSupportDetails details;

    return details;
}
```

本节将介绍如何查询包含此信息的结构体。这些结构体的含义以及它们包含的具体数据将在下一节中讨论。

让我们从基本表面功能开始。这些属性很容易查询，并被返回到一个`VkSurfaceCapabilitiesKHR`结构体中。

```c++
vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
```

这个函数在确定支持的功能时，考虑了指定的`VkPhysicalDevice`和`VkSurfaceKHR`窗口表面。所有的支持查询函数都以这两个参数作为第一个参数，因为它们是交换链的核心组件。

接下来的步骤是查询支持的表面格式。因为这是一个结构体列表，所以需要两个函数调用来完成:

```c++
uint32_t formatCount;
vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);

if (formatCount != 0) {
    details.formats.resize(formatCount);
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
}
```

确保向量被调整大小以容纳所有可用的格式。最后，查询支持的呈现模式的方式与`vkGetPhysicalDeviceSurfacePresentModesKHR`完全相同:

```c++
uint32_t presentModeCount;
vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, nullptr);

if (presentModeCount != 0) {
    details.presentModes.resize(presentModeCount);
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());
}
```

现在所有的细节都在结构体中了，因此让我们再次扩展`isDeviceSuitable`来利用这个函数，以验证交换链的支持是否足够。如果我们拥有至少一个支持的图像格式和一个支持的呈现模式，那么交换链的支持对于本教程就足够了。

```c++
bool swapChainAdequate = false;
if (extensionsSupported) {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
    swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
}
```

在验证扩展是否可用后，我们只需尝试查询交换链的支持。函数的最后一行更改为:

```c++
return indices.isComplete() && extensionsSupported && swapChainAdequate;
```

## 为交换链选择正确的设置

如果满足了`swapChainAdequate`的条件，则支持肯定是足够的，但仍然可能有许多不同的优化模式。现在我们将编写几个函数来找到最佳交换链的正确设置。有三种类型的设置要确定：

* 表面格式（颜色深度）
* 呈现模式（将图像“交换”到屏幕的条件）
* 交换范围（交换链中图像的分辨率）

对于这些设置中的每一个，我们都有一个理想值，如果它可用，我们将使用该值，否则我们将创建一些逻辑来找到下一个最佳选择。

### 表面格式

这个设置的函数起始如下。我们稍后将把`SwapChainSupportDetails`结构的`formats`成员作为参数传递。

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {

}
```

每个`VkSurfaceFormatKHR`条目都包含一个`format`和一个`colorSpace`成员。`format`成员指定了颜色通道和类型。例如，`VK_FORMAT_B8G8R8A8_SRGB`表示我们以B、G、R和alpha通道的顺序存储，每个通道使用8位无符号整数，总共每个像素32位。`colorSpace`成员使用`VK_COLOR_SPACE_SRGB_NONLINEAR_KHR`标志指示是否支持SRGB颜色空间。请注意，该标志在旧版本的规范中曾被称为`VK_COLORSPACE_SRGB_NONLINEAR_KHR`。

对于颜色空间，如果可用，我们将使用SRGB，因为它[会产生更准确的感知颜色](http://stackoverflow.com/questions/12524623/)。对于图像，如我们之后将使用的纹理，它也几乎是标准的颜色空间。
因此，我们应该使用一个SRGB颜色格式，其中最常见的是`VK_FORMAT_B8G8R8A8_SRGB`。

让我们遍历列表，看看是否有可用的首选组合:

```c++
for (const auto& availableFormat : availableFormats) {
    if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
        return availableFormat;
    }
}
```

如果这也失败了，我们可以根据可用的格式对其进行排序，根据它们的“好”程度来选择，但在大多数情况下，只选择指定的第一个格式也是可以的。

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {
    for (const auto& availableFormat : availableFormats) {
        if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
            return availableFormat;
        }
    }

    return availableFormats[0];
}
```

### 呈现模式

演示模式可能是交换链中最重要的设置，因为它代表了向屏幕显示图像的实际条件。在Vulkan中有四种可能的模式：

* `VK_PRESENT_MODE_IMMEDIATE_KHR`：应用程序提交的图像会立即传输到屏幕，这可能会导致撕裂现象。
* `VK_PRESENT_MODE_FIFO_KHR`：交换链是一个队列，当显示刷新时，显示器从队列前面获取一个图像，并且程序将渲染图像插入到队列的末尾。如果队列已满，则程序必须等待。这与现代游戏中的垂直同步类似。显示器刷新时的时刻称为“垂直空白期”。
* `VK_PRESENT_MODE_FIFO_RELAXED_KHR`：如果应用程序延迟且队列在上一个垂直空白期为空，则此模式与上一个模式不同。不等待下一个垂直空白期，图像一到达就立即传输。这可能导致可见的撕裂现象。
* `VK_PRESENT_MODE_MAILBOX_KHR`：这是第二个模式的另一种变体。如果队列已满，则不会阻塞应用程序，已排队的图像会被较新的图像替换。这种模式可以在避免撕裂的同时以尽可能快的速度渲染帧，相对于标准垂直同步，可以减少延迟问题。这通常称为“三重缓冲”，尽管仅存在三个缓冲区并不一定意味着帧速率是无锁的。

只有`VK_PRESENT_MODE_FIFO_KHR`模式保证可用，因此我们需要编写一个函数来查找可用的最佳模式。:

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    return VK_PRESENT_MODE_FIFO_KHR;
}
```

我个人认为，如果不关心能源消耗，`VK_PRESENT_MODE_MAILBOX_KHR`是一个非常好的折衷方案。它允许我们避免撕裂现象，同时通过在垂直空白期之前渲染尽可能新的图像，保持相当低的延迟。在移动设备上，能源使用更为重要，您可能更希望使用`VK_PRESENT_MODE_FIFO_KHR`。现在，让我们浏览一下列表，看看`VK_PRESENT_MODE_MAILBOX_KHR`是否可用:

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        }
    }

    return VK_PRESENT_MODE_FIFO_KHR;
}
```

### 交换分辨率

这样我们就只剩下一个重要的属性，为此我们将添加最后一个函数：

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {

}
```

交换链的交换范围是交换链图像的分辨率，通常几乎完全等于我们绘制窗口的分辨率，以像素为单位（稍后会详细讨论）。可能的分辨率范围在`VkSurfaceCapabilitiesKHR`结构中定义。 Vulkan要求我们通过设置`currentExtent`成员的宽度和高度来匹配窗口的分辨率。然而，一些窗口管理器允许我们在此处不同，这是通过将`currentExtent`中的宽度和高度设置为一个特殊值：`uint32_t`的最大值来指示的。在这种情况下，我们将在`minImageExtent`和`maxImageExtent`边界内选择最适合窗口的分辨率。但我们必须以正确的单位指定分辨率。

GLFW在测量尺寸时使用两种单位：像素和[屏幕坐标](https://www.glfw.org/docs/latest/intro_guide.html#coordinate_systems)。例如，我们在创建窗口时指定的分辨率`{WIDTH，HEIGHT}`是以屏幕坐标为单位的。但是，Vulkan使用像素，因此交换链范围也必须以像素为单位指定。不幸的是，如果您正在使用高DPI显示（例如苹果的Retina显示器），则屏幕坐标不对应于像素。由于像素密度较高，窗口的像素分辨率将大于屏幕坐标中的分辨率。因此，如果Vulkan没有为我们修复交换范围，我们不能简单地使用原始的`{WIDTH，HEIGHT}`。相反，我们必须使用`glfwGetFramebufferSize`在匹配最小和最大图像范围之前查询窗口的像素分辨率。

```c++
#include <cstdint> // Necessary for uint32_t
#include <limits> // Necessary for std::numeric_limits
#include <algorithm> // Necessary for std::clamp

...

VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max()) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        actualExtent.width = std::clamp(actualExtent.width, capabilities.minImageExtent.width, capabilities.maxImageExtent.width);
        actualExtent.height = std::clamp(actualExtent.height, capabilities.minImageExtent.height, capabilities.maxImageExtent.height);

        return actualExtent;
    }
}
```

`clamp`函数在这里用于将`width`和`height`的值限制在实现支持的允许的最小和最大范围之间。

## 创建交换链

现在我们有了所有这些辅助函数来帮助我们在运行时做出选择，我们终于拥有了创建一个工作交换链所需的所有信息。

创建一个`createSwapChain`函数，它首先使用这些调用的结果，并确保在逻辑设备创建后从`initVulkan`中调用它。

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
}

void createSwapChain() {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);
}
```

除了这些属性之外，我们还必须决定在交换链中需要多少图像。实现规定了其需要的最小图像数来正常运行:

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount;
```

然而，仅满足最小要求意味着有时我们可能必须等待驱动程序完成内部操作，然后才能获取另一个图像进行渲染。因此，建议请求比最小值多至少一个图像:

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
```

同时，我们还要确保不要超过最大图像数量，其中 `0` 是一个特殊值，表示没有最大限制:

```c++
if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount) {
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

按照 Vulkan 对象的传统，创建交换链对象需要填充一个庞大的结构体。它的开始部分非常熟悉:

```c++
VkSwapchainCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;
```

在指定与哪个表面绑定交换链之后，需要指定交换链图像的详细信息:

```c++
createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

`imageArrayLayers`指定每个图像包含的层的数量。这通常是`1`，除非您正在开发一个立体3D应用程序。`imageUsage`位字段指定我们将在交换链图像中使用的操作类型。在本教程中，我们将直接将图像渲染到它们，这意味着它们将用作颜色附件。也有可能您会首先将图像渲染到单独的图像中，以执行后处理等操作。在这种情况下，您可以使用`VK_IMAGE_USAGE_TRANSFER_DST_BIT`等值，并使用内存操作将渲染的图像传输到交换链图像。

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if (indices.graphicsFamily != indices.presentFamily) {
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0; // Optional
    createInfo.pQueueFamilyIndices = nullptr; // Optional
}
```

接下来，我们需要指定如何处理在多个队列族中使用的交换链图像。如果图形队列族与显示队列族不同，这在我们的应用程序中将是一种情况。我们将在图形队列中绘制交换链图像，然后在显示队列上提交它们。在从多个队列访问图像的情况下，有两种处理方式：

* `VK_SHARING_MODE_EXCLUSIVE`：一个图像一次只能由一个队列族拥有，并且在在另一个队列族中使用之前必须显式地转让所有权。此选项提供最佳性能。
* `VK_SHARING_MODE_CONCURRENT`：图像可以在多个队列族之间使用而无需显式地转让所有权。

在本教程中，如果队列族不同，我们将使用并发模式，以避免在所有权章节中进行一些更好在后面的时间解释的概念。并发模式要求您预先指定共享所有权的队列族，使用`queueFamilyIndexCount`和`pQueueFamilyIndices`参数。如果图形队列族和显示队列族相同（这在大多数硬件上都是这样），则应坚持使用独占模式，因为并发模式要求您至少指定两个不同的队列族。

```c++
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
```

如果支持的话（在`capabilities`中的`supportedTransforms`），我们可以指定应用于交换链中的图像的特定变换，比如90度顺时针旋转或水平翻转。如果您不想进行任何变换，只需指定当前变换即可。

```c++
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
```

`compositeAlpha` 字段指定是否应该使用 alpha 通道与窗口系统中的其他窗口进行混合。您几乎总是希望忽略 alpha 通道，因此使用 `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`。

```c++
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
```

`presentMode` 成员很明显。如果 `clipped` 成员设置为 `VK_TRUE`，则表示我们不关心被遮挡的像素的颜色，例如因为另一个窗口在它们的前面。除非您真的需要能够读取这些像素并获得可预测的结果，否则通过启用裁剪可以获得最佳性能。

```c++
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

那么剩下最后一个字段，`oldSwapChain`。使用Vulkan时，当您的应用程序运行时，您的交换链可能会变得无效或未优化，例如因为窗口大小被调整。在这种情况下，实际上需要从头开始重新创建交换链，并且必须在这个字段中指定旧交换链的引用。这是一个复杂的主题，我们将在[未来的章节](/en/Drawing_a_triangle/Swap_chain_recreation)中学习更多相关知识。目前，我们假设我们只会创建一个交换链。

现在添加一个类成员来存储 `VkSwapchainKHR` 对象：

```c++
VkSwapchainKHR swapChain;
```

现在创建交换链就像调用 `vkCreateSwapchainKHR` 一样简单:

```c++
if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS) {
    throw std::runtime_error("创建交换链失败!");
}
```

参数是逻辑设备、交换链创建信息、可选的自定义分配器以及一个指向存储句柄的变量的指针。没有什么意外。在设备销毁之前应该使用 `vkDestroySwapchainKHR` 进行清理:

```c++
void cleanup() {
    vkDestroySwapchainKHR(device, swapChain, nullptr);
    ...
}
```

现在运行应用程序，确保交换链已成功创建！如果在这一步中 `vkCreateSwapchainKHR` 函数出现访问违例错误，或者看到类似 `Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll` 的消息，则请参阅关于 Steam 覆盖层的[常见问题解答](https://vulkan-tutorial.com/FAQ)。

在启用验证层的情况下，尝试删除 `createInfo.imageExtent = extent;` 这一行。你会发现其中一个验证层立即捕获了错误，并打印了一条有用的消息：

![swap_chain_validation_layer.png](https://vulkan-tutorial.com/images/swap_chain_validation_layer.png)

## 检索交换链图像

现在已经创建了交换链，接下来就是检索其中的 `VkImage` 句柄。在后面的章节中，我们将在渲染操作中引用这些句柄。添加一个类成员来存储这些句柄:

```c++
std::vector<VkImage> swapChainImages;
```

这些图像是由实现为交换链创建的，并且它们在交换链被销毁时会自动清理，因此我们不需要添加任何清理代码。

我将检索这些句柄的代码添加到 `createSwapChain` 函数的末尾，就在 `vkCreateSwapchainKHR` 调用之后。检索它们与我们之前从 Vulkan 检索对象数组的方式非常相似。请记住，我们只在交换链中指定了最小数量的图像，因此实现允许创建更多图像的交换链。这就是为什么我们首先用 `vkGetSwapchainImagesKHR` 查询最终的图像数量，然后调整容器的大小，最后再次调用它以检索句柄。

```c++
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

最后一件事，在成员变量中存储我们为交换链图像选择的格式和范围。我们将在后续章节中需要它们。

```c++
VkSwapchainKHR swapChain;
std::vector<VkImage> swapChainImages;
VkFormat swapChainImageFormat;
VkExtent2D swapChainExtent;

...

swapChainImageFormat = surfaceFormat.format;
swapChainExtent = extent;
```

现在我们有了一组可用于绘制的图像，并可以将它们呈现到窗口上。下一章将开始介绍如何将这些图像设置为渲染目标，并深入研究实际的图形管线和绘制命令！

[C++ code](/code/06_swap_chain_creation.cpp)
