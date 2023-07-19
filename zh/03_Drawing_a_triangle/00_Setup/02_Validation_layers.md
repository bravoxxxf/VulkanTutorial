## What are validation layers?

Vulkan API的设计理念是减少驱动程序开销，其中一个体现这一目标的是API中默认的错误检查时非常有限的。即使是简单的错误，例如将枚举设置为错误的值或将空指针传递给必需的参数，通常也不会被显式处理，而只会导致崩溃或未定义行为。因为Vulkan要求您对所有操作都非常明确，所以很容易犯许多小错误，比如使用新的GPU功能但忘记在逻辑设备创建时请求它。

然而，这并不意味着这些检查无法添加到API中。Vulkan引入了一种称为*验证层*的优雅系统。验证层是可选的组件，它们钩入Vulkan函数调用以应用额外的操作。验证层中常见的操作包括：

* 根据规范检查参数的值，以检测误用情况
* 跟踪对象的创建和销毁，以查找资源泄漏
* 通过跟踪调用来源的线程来检查线程安全性
* 将每个调用及其参数记录到标准输出
* 对Vulkan调用进行追踪以进行性能分析和回放

下面是一个诊断验证层中函数实现的示例：

```c++
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("所需参数必须是空指针!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

这些验证层可以自由地堆叠，以包括您感兴趣的所有调试功能。您可以在调试构建中启用验证层，并在发布构建中完全禁用它们，这样可以兼顾两者的优点！

Vulkan本身不带任何内置的验证层，但LunarG Vulkan SDK提供了一套良好的验证层，用于检查常见错误。它们也是[开源的](https://github.com/KhronosGroup/Vulkan-ValidationLayers)，因此您可以查看它们检查的错误类型并作出贡献。使用验证层是避免应用程序因意外依赖于未定义行为而在不同驱动程序上崩溃的最佳方式。

只有在系统上安装了验证层后，才能使用验证层。例如，LunarG验证层仅适用于安装了Vulkan SDK的PC。

在Vulkan中，以前存在两种不同类型的验证层：实例验证层和设备特定验证层。其设计理念是，实例验证层只检查与全局Vulkan对象（如实例）相关的调用，而设备特定验证层只检查与特定GPU相关的调用。现在，设备特定验证层已经被弃用，这意味着实例验证层适用于所有Vulkan调用。规范文档仍建议您在设备级别上启用验证层以实现兼容性，某些实现要求如此。在逻辑设备级别上，我们将简单地指定与实例相同的验证层，我们将在[后面](https://renderdoc.org/vulkan-in-30-minutes.html#logical-device-and-queues)看到。

## 使用验证层

在本节中，我们将看到如何启用Vulkan SDK提供的标准验证层。与扩展类似，验证层需要通过指定它们的名称来启用。所有有用的标准验证都捆绑在一个名为`VK_LAYER_KHRONOS_validation`的SDK中。

让我们首先向程序添加两个配置变量，以指定要启用的层以及是否启用它们。我选择基于程序是否在调试模式下编译来确定该值。`NDEBUG`宏是C++标准的一部分，表示“非调试”。

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::vector<const char*> validationLayers = {
    "VK_LAYER_KHRONOS_validation"
};

#ifdef NDEBUG
    const bool enableValidationLayers = false;
#else
    const bool enableValidationLayers = true;
#endif
```

我们将添加一个名为`checkValidationLayerSupport`的新函数，用于检查是否所有请求的层都可用。首先，使用`vkEnumerateInstanceLayerProperties`函数列出所有可用的层。它的用法与在实例创建章节中讨论的`vkEnumerateInstanceExtensionProperties`相同。

```c++
bool checkValidationLayerSupport() {
    uint32_t layerCount; 
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);// 返回验证层的数量

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());// 获取所有验证层

    return false;
}
```

接下来，检查`validationLayers`中的所有层是否存在于`availableLayers`列表中。您可能需要包含`<cstring>`头文件以使用`strcmp`函数。


```c++
for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
        return false;
    }
}

return true;
```

我们现在可以在 `createInstance` 中调用了:

```c++
void createInstance() {
    if (enableValidationLayers && !checkValidationLayerSupport()) {
        throw std::runtime_error("已请求验证层，但不可用!");
    }

    ...
}
```

现在以调试模式运行程序，确保错误不会发生。如果出现错误，查阅常见问题解答（FAQ）。

最后，在实例化`VkInstanceCreateInfo`结构时，如果启用了验证层（validation layer），请修改代码以包含验证层的名称：

```c++
if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

如果检查成功，那么`vkCreateInstance`不应该返回`VK_ERROR_LAYER_NOT_PRESENT`错误，但是你应该运行程序来确认。

## 消息回调

验证层默认会将调试消息打印到标准输出，但我们也可以通过在程序中提供显式回调来自行处理这些消息。这样做还可以让你决定希望看到哪种类型的消息，因为并不是所有的消息都是（致命的）错误。如果你暂时不想这样做，你可以直接跳到本章的最后一节。

要在程序中设置回调来处理消息及其相关细节，我们需要使用`VK_EXT_debug_utils`扩展来设置一个带有回调的调试信使（debug messenger）。

我们首先会创建一个`getRequiredExtensions`函数，根据验证层是否启用，它将返回所需的扩展列表：

```c++
std::vector<const char*> getRequiredExtensions() {
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

    std::vector<const char*> extensions(glfwExtensions, glfwExtensions + glfwExtensionCount);

    if (enableValidationLayers) {
        extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
    }

    return extensions;
}
```

GLFW指定的扩展始终是必需的，但调试信使扩展是有条件添加的。请注意，我在这里使用了`VK_EXT_DEBUG_UTILS_EXTENSION_NAME`宏，它等同于字符串字面量"VK_EXT_debug_utils"。使用这个宏可以避免拼写错误。

现在我们可以在`createInstance`函数中使用这个函数了:

```c++
auto extensions = getRequiredExtensions();
createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
createInfo.ppEnabledExtensionNames = extensions.data();
```

运行程序，确保不会收到`VK_ERROR_EXTENSION_NOT_PRESENT`错误。我们实际上不需要检查该扩展是否存在，因为验证层的可用性应该已经暗示了它的存在。

现在让我们来看看调试回调函数是什么样的。添加一个新的静态成员函数，名为`debugCallback`，其原型为`PFN_vkDebugUtilsMessengerCallbackEXT`。`VKAPI_ATTR`和`VKAPI_CALL`确保该函数的签名对于Vulkan来说是正确的，以便Vulkan能够调用它。

```c++
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData) {

    std::cerr << "validation layer: " << pCallbackData->pMessage << std::endl;

    return VK_FALSE;
}
```

第一个参数指定了消息的严重程度，它是以下标志之一：

* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT`：诊断消息
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`：信息消息，例如资源的创建
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT`：关于行为的消息，这个行为可能不一定是错误，但很可能是你的应用程序中的一个bug
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT`：关于无效行为的消息，可能会导致崩溃

这个枚举的值被设置成可以使用比较运算来检查消息是否与某个严重程度水平相等或更严重，例如:

```c++
if (messageSeverity >= VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) {
    // 该消息足够重要以便显示
}
```

`messageType`参数可以具有以下值：

* `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT`：某个与规范或性能无关的事件发生了
* `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT`：发生了违反规范或指示可能出现错误的事件
* `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT`：可能存在对Vulkan的非最佳使用

`pCallbackData`参数指向一个`VkDebugUtilsMessengerCallbackDataEXT`结构体，其中包含消息本身的详细信息，其中最重要的成员有：

* `pMessage`：以空字符结尾的调试消息字符串
* `pObjects`：与消息相关的Vulkan对象句柄数组
* `objectCount`：数组中对象的数量

最后，`pUserData`参数包含一个在回调设置期间指定的指针，允许您将自己的数据传递给回调函数。

回调函数返回一个布尔值，指示触发验证层消息的Vulkan调用是否应该中止。如果回调函数返回`true`，则该调用将中止，并显示`VK_ERROR_VALIDATION_FAILED_EXT`错误。这通常仅用于测试验证层本身，所以您应该始终返回`VK_FALSE`。

现在剩下的就是让Vulkan知道这个回调函数。也许有点令人惊讶，即使在Vulkan中，调试回调也是使用需要显式创建和销毁的句柄来管理的。这样的回调属于*调试信使（debug messenger）*的一部分，您可以拥有任意数量的调试信使。在`instance`下面添加一个类成员来保存这个句柄:

```c++
VkDebugUtilsMessengerEXT debugMessenger;
```

现在在`initVulkan`中的`createInstance`之后添加一个名为`setupDebugMessenger`的函数：

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
}

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

}
```

我们需要填写一个结构，其中包含有关调试信使及其回调的详细信息:

```c++
VkDebugUtilsMessengerCreateInfoEXT createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
createInfo.pfnUserCallback = debugCallback;
createInfo.pUserData = nullptr; // Optional
```

`messageSeverity`字段允许您指定您希望回调被调用的所有严重性类型。在这里，我指定了除了`VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`之外的所有类型，以便接收有关可能问题的通知，同时排除冗长的一般调试信息。

类似地，`messageType`字段允许您过滤回调函数将收到的消息类型。在这里，我简单地启用了所有类型。如果某些类型对您没有用处，您总是可以禁用它们。

最后，`pfnUserCallback`字段指定了回调函数的指针。您可以选择将指针传递给`pUserData`字段，它将通过`pUserData`参数传递给回调函数。您可以使用这个功能来传递指向`HelloTriangleApplication`类的指针，例如。

请注意，有许多其他的配置验证层消息和调试回调的方法，但这是本教程的一个很好的设置。有关更多可能性的信息，请参阅[扩展规范](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap50.html#VK_EXT_debug_utils)。

这个结构体应该传递给`vkCreateDebugUtilsMessengerEXT`函数来创建`VkDebugUtilsMessengerEXT`对象。不幸的是，因为这个函数是一个扩展函数，它不会自动加载。我们必须使用`vkGetInstanceProcAddr`来自己查找它的地址。我们将在后台创建一个处理这个问题的代理函数。我已经在`HelloTriangleApplication`类定义的上方添加了它。

```c++
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger) {
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    if (func != nullptr) {
        return func(instance, pCreateInfo, pAllocator, pDebugMessenger);
    } else {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
```

如果`vkGetInstanceProcAddr`函数无法加载该函数，它将返回`nullptr`。如果该函数可用，我们现在可以调用`vkGetInstanceProcAddr`函数来创建扩展对象:

```c++
if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
    throw std::runtime_error("设置调试信使失败!");
}
```

倒数第二个参数再次是可选的分配器回调，我们将其设置为`nullptr`，除此之外其他参数都相当简单。由于调试信使是特定于我们的Vulkan实例及其层级的，它需要作为第一个参数显式指定。您将在后面看到其他*子*对象也会使用这种模式。

`VkDebugUtilsMessengerEXT`对象也需要使用`vkDestroyDebugUtilsMessengerEXT`调用进行清理。与`vkCreateDebugUtilsMessengerEXT`类似，需要显式加载这个函数。

在`CreateDebugUtilsMessengerEXT`的下面创建另一个代理函数:

```c++
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator) {
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if (func != nullptr) {
        func(instance, debugMessenger, pAllocator);
    }
}
```

请确保这个函数要么是一个静态类函数，要么是一个类外部的函数。然后我们可以在`cleanup`函数中调用它:

```c++
void cleanup() {
    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

## 调试实例的创建和销毁

尽管我们现在已经添加了验证层的调试功能到程序中，但还没有覆盖到所有情况。`vkCreateDebugUtilsMessengerEXT`调用需要一个有效的实例被创建，而`vkDestroyDebugUtilsMessengerEXT`必须在实例被销毁之前调用。目前这样会导致我们无法调试`vkCreateInstance`和`vkDestroyInstance`调用中的任何问题。

然而，如果您仔细阅读了[扩展文档](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/appendices/VK_EXT_debug_utils.adoc#examples)，您将会发现有一种方法可以为这两个函数调用单独创建一个调试信使。它要求您只需在`VkInstanceCreateInfo`的`pNext`扩展字段中传递一个指向`VkDebugUtilsMessengerCreateInfoEXT`结构的指针。首先将调试信使创建信息的填充提取为一个单独的函数:

```c++
void populateDebugMessengerCreateInfo(VkDebugUtilsMessengerCreateInfoEXT& createInfo) {
    createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
    createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
    createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
    createInfo.pfnUserCallback = debugCallback;
}

...

void setupDebugMessenger() {
    if (!enableValidationLayers) return;

    VkDebugUtilsMessengerCreateInfoEXT createInfo;
    populateDebugMessengerCreateInfo(createInfo);

    if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
        throw std::runtime_error("设置调试信使失败!");
    }
}
```

我们现在可以重复使用 `createInstance` 函数了:

```c++
void createInstance() {
    ...

    VkInstanceCreateInfo createInfo{};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;

    ...

    VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo{};
    if (enableValidationLayers) {
        createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
        createInfo.ppEnabledLayerNames = validationLayers.data();

        populateDebugMessengerCreateInfo(debugCreateInfo);
        createInfo.pNext = (VkDebugUtilsMessengerCreateInfoEXT*) &debugCreateInfo;
    } else {
        createInfo.enabledLayerCount = 0;

        createInfo.pNext = nullptr;
    }

    if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
        throw std::runtime_error("创建实例失败!");
    }
}
```

`debugCreateInfo`变量被放置在if语句之外，以确保在`vkCreateInstance`调用之前不会被销毁。通过这种方式创建了一个额外的调试信使，它将在`vkCreateInstance`和`vkDestroyInstance`期间自动使用，并在之后进行清理。

## 测试

现在让我们有意制造一个错误来查看验证层的工作情况。暂时在`cleanup`函数中删除对`DestroyDebugUtilsMessengerEXT`的调用，然后运行您的程序。一旦程序退出，您应该会看到类似下面这样的消息：

![](/images/validation_layer_test.png)

>如果您没有看到任何消息，请[检查您的安装](https://vulkan.lunarg.com/doc/view/1.2.131.1/windows/getting_started.html#user-content-verify-the-installation)。

如果您想查看是哪个调用触发了消息，可以在消息回调函数中添加断点并查看堆栈跟踪。

## 配置

验证层的行为设置远不止在`VkDebugUtilsMessengerCreateInfoEXT`结构中指定的标志。请浏览Vulkan SDK并进入`Config`目录。在那里，您将找到一个名为`vk_layer_settings.txt`的文件，其中解释了如何配置这些层。

要为自己的应用程序配置层设置，请将该文件复制到项目的`Debug`和`Release`目录中，并按照说明设置所需的行为。然而，在本教程的其余部分中，我将假设您使用默认设置。

在本教程的过程中，我将故意犯一些错误，以向您展示验证层在捕捉这些错误方面有多么有用，并教导您对于Vulkan来说，了解自己在做什么是多么重要。现在是时候看一下[Vulkan系统中的设备](!zh/Drawing_a_triangle/Setup/Physical_devices_and_queue_families)了。

[C++代码](/code/02_validation_layers.cpp)
