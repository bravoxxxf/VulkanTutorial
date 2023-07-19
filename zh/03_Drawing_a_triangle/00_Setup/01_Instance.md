## Creating an instance

你需要做的第一件事情就是通过创建一个 instance Vulkan实例来初始化。实例是应用程序和 Vulkan 库之间的连接，创建它涉及向驱动程序指定有关应用程序的一些详细信息。

```c++
void initVulkan() {
    createInstance();
}
```

增加一个instance成员变量:

```c++
private:
VkInstance instance;
```

现在，我们需要通过一个数据结构描述我们的应用程序并创建我们的实例。这是一个可选项, 但是它可以提供一些有用的信息给驱动程序来优化我们的应用程序 （例如，使用具有某些特殊行为的知名图形引擎）. 这个结构是 `VkApplicationInfo`:

```c++
void createInstance() {
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
}
```

如前所述，Vulkan中的许多结构体要求您在`sType`成员中明确指定类型。这也是许多结构体之一，具有一个`pNext`成员，可以在将来指向扩展信息。我们在这里使用值初始化将其保留为`nullptr`。

在Vulkan中，大量的信息通过结构体而不是函数参数传递，我们需要填写一个额外的结构体，以提供创建实例所需的足够信息。这个下一个结构体是必需的，并告诉Vulkan驱动程序我们想要使用哪些全局扩展和验证层。这里的全局意味着它们适用于整个程序，而不是特定的设备，这将在接下来的几章中变得清楚。

```c++
VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;
```

前两个参数很简单。接下来的两个层指定了所需的全局扩展。正如在概述章节中提到的那样，Vulkan是一个与平台无关的API，这意味着您需要一个扩展来与窗口系统进行交互。GLFW有一个方便的内置函数，可以返回它所需的扩展，我们可以将其传递给结构体：

```c++
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions;

glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;
```

结构体的最后两个成员确定要启用的全局验证层。我们将在下一章中更详细地讨论这些内容，所以现在只需将它们留空即可。

```c++
createInfo.enabledLayerCount = 0;
```

现在，我们已经指定了Vulkan创建实例所需的所有内容，我们最终可以发出`vkCreateInstance`调用：

```c++
VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```

正如您将看到的，Vulkan中对象创建函数参数的一般模式如下：

* 包含创建信息的结构体指针
* 自定义分配器回调函数的指针，在本教程中始终为`nullptr`
* 存储新对象句柄的变量的指针

如果一切顺利，那么实例的句柄将存储在`VkInstance`类成员中。几乎所有的Vulkan函数都返回一个`VkResult`类型的值，它要么是`VK_SUCCESS`，要么是一个错误代码。为了检查实例是否成功创建，我们不需要存储结果，只需对成功值进行检查即可：

```c++
if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

现在运行程序，确保实例成功创建。

## 遇到 VK_ERROR_INCOMPATIBLE_DRIVER 错误：
如果在最新的MoltenVK SDK上使用MacOS，可能会从`vkCreateInstance`返回`VK_ERROR_INCOMPATIBLE_DRIVER`。根据[Getting Start Notes](https://vulkan.lunarg.com/doc/sdk/1.3.216.0/mac/getting_started.html)的说明，从1.3.216版本的Vulkan SDK开始，`VK_KHR_PORTABILITY_subset`扩展是强制性的。

为了解决此错误，首先将`VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR`位添加到`VkInstanceCreateInfo`结构体的标志中，然后将`VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME`添加到实例启用的扩展列表中。

通常代码可能如下所示：
```c++
...

std::vector<const char*> requiredExtensions;

for(uint32_t i = 0; i < glfwExtensionCount; i++) {
    requiredExtensions.emplace_back(glfwExtensions[i]);
}

requiredExtensions.emplace_back(VK_KHR_PORTABILITY_ENUMERATION_EXTENSION_NAME);

createInfo.flags |= VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR;

createInfo.enabledExtensionCount = (uint32_t) requiredExtensions.size();
createInfo.ppEnabledExtensionNames = requiredExtensions.data();

if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

## 检查扩展支持

如果查看`vkCreateInstance`的文档，您将看到其中一个可能的错误代码是`VK_ERROR_EXTENSION_NOT_PRESENT`。对于像窗口系统接口这样的基本扩展，我们可以简单地指定所需的扩展，并在返回该错误代码时终止。但是，如果我们想检查可选功能呢？

在创建实例之前，可以使用`vkEnumerateInstanceExtensionProperties`函数获取支持的扩展列表。它接受一个指向存储扩展数量的变量的指针，以及一个用于存储扩展详细信息的`VkExtensionProperties`数组。它还接受一个可选的第一个参数，允许我们按特定验证层过滤扩展，但我们现在忽略它。

为了分配一个数组来存储扩展详细信息，我们首先需要知道有多少个扩展。您可以通过将后面的参数留空来仅请求扩展的数量：

```c++
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```

现在，分配一个数组来存储扩展详细信息 (`include <vector>`):

```c++
std::vector<VkExtensionProperties> extensions(extensionCount);
```

最后我们可以检索扩展详细信息:

```c++
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

每一个 `VkExtensionProperties` 结构 都包含了扩展的版本和名称。 我们可以通过一个简单的循环来查看 (`\t` is a tab for indentation):

```c++
std::cout << "available extensions:\n";

for (const auto& extension : extensions) {
    std::cout << '\t' << extension.extensionName << '\n';
}
```

如果您希望提供有关Vulkan支持的一些详细信息，可以将此代码添加到`createInstance`函数中。作为一个挑战，尝试创建一个函数，检查`glfwGetRequiredInstanceExtensions`返回的所有扩展是否包含在支持的扩展列表中。

## Cleaning up

 `VkInstance` 应该在程序退出时被正确销毁. 可以在 `cleanup` 中执行 `vkDestroyInstance` 函数来销毁:

```c++
void cleanup() {
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

`vkDestroyInstance`函数的参数很简单。如前一章所述，Vulkan中的分配和释放函数具有一个可选的分配器回调，我们将通过将`nullptr`传递给它来忽略它。在销毁实例之前，我们应该清理掉之前在后续章节中创建的所有其他Vulkan资源。

在继续实例创建后的更复杂步骤之前，是时候通过查看[验证层](https://vulkan.lunarg.com/doc/sdk/latest/windows/validation_layers.html)来评估我们的调试选项了。

[C++ code](/code/01_instance_creation.cpp)
