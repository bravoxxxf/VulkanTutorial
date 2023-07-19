## 介绍

在选择了要使用的物理设备之后，我们需要设置一个*逻辑设备*来与其进行交互。逻辑设备的创建过程类似于实例的创建过程，它描述了我们想要使用的功能。由于我们已经查询了可用的队列族，现在我们还需要指定要创建哪些队列。如果您有不同的需求，甚至可以从同一个物理设备创建多个逻辑设备。

首先，在类中添加一个新的成员来存储逻辑设备句柄。

```c++
VkDevice device;
```

下一步, 在 `initVulkan` 中增加 `createLogicalDevice` 函数.

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

## 指定要创建的队列

逻辑设备的创建涉及再次在结构体中指定一堆细节，其中第一个结构体将是`VkDeviceQueueCreateInfo`。该结构体描述了我们希望为单个队列族创建的队列数量。目前我们只对具备图形功能的队列感兴趣。

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

目前可用的驱动程序只允许您为每个队列族创建少量的队列，而实际上您不需要多个队列。这是因为您可以在多个线程上创建所有的命令缓冲区，然后在主线程上用一个低开销的调用将它们一次性提交。

Vulkan允许您为队列分配优先级，使用浮点数介于`0.0`和`1.0`之间，以影响命令缓冲区执行的调度。即使只有一个队列，这也是必需的:

```c++
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

## 指定使用的设备功能

接下来需要指定的信息是我们将要使用的设备功能。这些功能是我们在上一节中使用`vkGetPhysicalDeviceFeatures`查询到的支持特性，比如几何着色器。目前我们不需要任何特殊功能，所以我们可以简单地定义它并将所有内容都设置为`VK_FALSE`。等到我们准备开始在Vulkan中进行更有趣的操作时，我们会回到这个结构体。

```c++
VkPhysicalDeviceFeatures deviceFeatures{};
```

## 创建逻辑设备

有了前面两个结构体，我们可以开始填写主要的`VkDeviceCreateInfo`结构体了。

```c++
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

首先，将队列创建信息和设备功能结构体的指针添加进去:

```c++
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

剩余的信息与`VkInstanceCreateInfo`结构体类似，需要您指定设备特定的扩展和验证层。不同之处在于这些是这次是针对设备的。

一个设备特定的扩展的例子是`VK_KHR_swapchain`，它允许您将从该设备渲染的图像呈现到窗口。可能存在不支持该功能的Vulkan设备，例如因为它们只支持计算操作。我们将在交换链章节再次涉及到这个扩展。

之前的Vulkan实现区分了实例和设备特定的验证层，但现在[不再是这样](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap40.html#extendingvulkan-layers-devicelayerdeprecation)。这意味着`VkDeviceCreateInfo`的`enabledLayerCount`和`ppEnabledLayerNames`字段在最新的实现中被忽略。然而，为了与旧实现兼容，设置它们仍然是一个好主意:

```c++
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

目前我们不需要任何设备特定的扩展。

现在我们准备好通过调用名为`vkCreateDevice`的函数来实例化逻辑设备了。

```c++
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("创建逻辑设备失败!");
}
```

这些参数包括要与之交互的物理设备、我们刚刚指定的队列和使用信息、可选的分配回调指针，以及一个用于存储逻辑设备句柄的变量指针。与实例创建函数类似，这个调用可能会根据启用不存在的扩展或指定不支持的功能使用导致错误返回。

设备实例需要在 `cleanup` 中调用 `vkDestroyDevice` 函数删除:

```c++
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

逻辑设备不直接与实例进行交互，这就是为什么它不作为参数包含在内的原因。

## 获取队列句柄

队列会随着逻辑设备的自动创建而创建，但我们还没有一个用于与它们进行交互的句柄。首先添加一个类成员来存储指向图形队列的句柄：

```c++
VkQueue graphicsQueue;
```

设备队列在设备销毁时会自动清理，所以我们不需要在`cleanup`函数中做任何处理。

我们可以使用`vkGetDeviceQueue`函数来检索每个队列族的队列句柄。参数包括逻辑设备、队列族、队列索引以及一个用于存储队列句柄的变量的指针。由于我们只从该族中创建了一个队列，所以我们只需使用索引`0`。

```c++
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

现在有了逻辑设备和队列句柄，我们实际上可以开始使用显卡来进行操作了！在接下来的几章中，我们将设置资源来向窗口系统呈现结果。

[C++ code](/code/04_logical_device.cpp)
