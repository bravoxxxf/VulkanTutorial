## 选择物理设备

在通过VkInstance初始化Vulkan库之后，我们需要在系统中查找并选择支持我们所需功能的显卡。实际上，我们可以选择任意数量的显卡并同时使用它们，但在本教程中，我们将选择满足我们需求的第一张显卡。

我们将添加一个名为`pickPhysicalDevice`的函数，并在`initVulkan`函数中调用它。

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```

我们最终选择的显卡将会被存储在一个VkPhysicalDevice句柄中，并作为一个新的类成员添加进来。这个对象将在VkInstance被销毁时隐式地被销毁，因此我们不需要在`cleanup`函数中做任何新的处理。

```c++
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```

列出显卡的过程与列出扩展非常相似，首先需要查询显卡的数量。

```c++
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```

如果系统中没有支持Vulkan的显卡设备（数量为0），则继续进行下去没有任何意义。

```c++
if (deviceCount == 0) {
    throw std::runtime_error("没有找到支持Vulkan的显卡!");
}
```

否则，我们现在可以分配一个数组来保存所有的VkPhysicalDevice句柄。

```c++
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

现在我们需要评估每一个显卡，并检查它们是否适合我们要执行的操作，因为并不是所有的显卡都是相同的。为此，我们将引入一个新的函数。:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

我们将检查有任何物理设备，是否满足我们将在该函数中添加的要求。

```c++
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("未找到合适的显卡!");
}
```

下一节将在`isDeviceSuitable`函数中介绍我们将要检查的第一个要求。随着我们在后面的章节中开始使用更多的Vulkan功能，我们还将扩展此函数以包括更多的检查。

## 基本设备适用性检查

为了评估设备的适用性，我们可以首先查询一些详细信息。可以使用`vkGetPhysicalDeviceProperties`函数来查询基本的设备属性，比如设备名称、类型和支持的Vulkan版本。

```c++
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

可选功能的支持，比如纹理压缩、64位浮点数和多视口渲染（在虚拟现实中很有用），可以通过`vkGetPhysicalDeviceFeatures`函数进行查询:

```c++
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```

关于设备内存和队列族（见下一节）可以查询更多细节，我们将在稍后讨论。

举个例子，假设我们认为我们的应用程序只能在支持几何着色器的专用显卡上运行。那么`isDeviceSuitable`函数将如下所示：:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```

不仅仅检查设备是否适用并选择第一个，你还可以为每个设备评分并选择最高分的设备。这样，你可以通过给专用显卡一个更高的分数来优先选择它，但如果只有集成显卡可用，则可以回退至它。你可以按照以下方式实现这样的功能：:

```c++
#include <map>

...

void pickPhysicalDevice() {
    ...

    // Use an ordered map to automatically sort candidates by increasing score
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // Check if the best candidate is suitable at all
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // 独立显卡（Discrete GPUs）具有显著的性能优势。
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // 纹理的最大可能大小会影响图形质量。
    score += deviceProperties.limits.maxImageDimension2D;

    // 如果没有几何着色器，应用程序无法正常运行。
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```

在本教程中，您不需要为此实现所有功能，但这可以给您一个设计设备选择过程的思路。当然，您也可以只显示选择项的名称，并允许用户进行选择。

由于我们刚刚开始，我们只需要Vulkan支持，因此我们将只满足于任何GPU:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

在下一节中，我们将讨论第一个需要检查的真实必需特性。

## 队列族

之前已经简要提到，在Vulkan中几乎每个操作，从绘制到上传纹理，都需要将命令提交给队列。不同类型的队列源自不同的*队列族*，每个队列族只允许执行一部分命令。例如，可能存在一个只允许处理计算命令的队列族，或者一个只允许内存传输相关命令的队列族。

我们需要检查设备支持哪些队列族，以及其中哪个队列族支持我们要使用的命令。为此，我们将添加一个名为`findQueueFamilies`的新函数，用于查找我们所需的所有队列族。

目前，我们只会查找支持图形命令的队列，所以该函数可以如下所示：

```c++
uint32_t findQueueFamilies(VkPhysicalDevice device) {
    // 查询队列族的逻辑
}
```

然而，在接下来的章节中，我们将查找另一个队列，所以最好为此做好准备，并将索引捆绑到一个结构体中:

```c++
struct QueueFamilyIndices {
    uint32_t graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // 找到队列族索引并将其填充到结构体的逻辑。
    return indices;
}
```

但是，如果队列族不可用怎么办？我们可以在`findQueueFamilies`函数中抛出异常，但是该函数并不是决定设备是否适用的正确地方。例如，我们可能会*倾向于*具有专用传输队列族的设备，但并不是必需的。因此，我们需要一种方法来指示是否找到了特定的队列族。

使用一个特殊的值来表示队列族不存在并不是一个很好的方法，因为`uint32_t`类型的任何值理论上都可能是有效的队列族索引，包括`0`。幸运的是，C++17引入了一种数据结构，可以区分值存在与不存在的情况:

```c++
#include <optional>

...

std::optional<uint32_t> graphicsFamily;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // false

graphicsFamily = 0;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // true
```

`std::optional`是一个包装器，在您分配了某个值之前不包含任何值。在任何时候，您可以通过调用其`has_value()`成员函数查询它是否包含值。这意味着我们可以将逻辑更改为以下方式:

```c++
#include <optional>

...

struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // 将找到的队列族的索引赋值给它们。
    return indices;
}
```

现在我们可以开始实现`findQueueFamilies`函数了:

```c++
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```

检索队列族列表的过程与您预期的完全一样，使用了`vkGetPhysicalDeviceQueueFamilyProperties`函数:

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

`VkQueueFamilyProperties`结构包含有关队列族的一些详细信息，包括支持的操作类型以及可以基于该族创建的队列数量。我们需要找到至少一个支持`VK_QUEUE_GRAPHICS_BIT`的队列族。

```c++
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    i++;
}
```

现在我们有了这个高级的队列族查找函数，我们可以在`isDeviceSuitable`函数中使用它来确保设备可以处理我们想要使用的命令:

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.graphicsFamily.has_value();
}
```

为了让这一切更加方便，我们还将在结构体本身中添加一个通用的检查:

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};

...

bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```

现在我们也可以在`findQueueFamilies`函数中使用它进行早期退出:

```c++
for (const auto& queueFamily : queueFamilies) {
    ...

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```

太好了，现在我们已经具备找到合适的物理设备所需的一切！下一步是[创建逻辑设备](!en/Drawing_a_triangle/Setup/Logical_device_and_queues)来与它进行交互。

[C++ code](/code/03_physical_device_selection.cpp)
