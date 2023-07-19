要在渲染管线中使用任何`VkImage`，包括交换链中的图像，我们必须创建`VkImageView`对象。图像视图实际上是对图像的视图。它描述了如何访问图像以及要访问的图像部分，例如是否应将其视为2D纹理深度纹理而不包含任何mip映射级别。

在本章中，我们将编写一个`createImageViews`函数，为交换链中的每个图像创建一个基本的图像视图，以便以后可以将它们用作颜色目标。

首先添加一个类成员来存储图像视图：

```c++
std::vector<VkImageView> swapChainImageViews;
```

创建函数 `createImageViews` ，并在创建交换链的函数后调用：

```c++
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

首先，我们需要调整列表的大小以适应我们将要创建的所有图像视图：

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```

接下来，设置一个循环，遍历所有交换链图像：

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

图像视图的创建参数在`VkImageViewCreateInfo`结构体中指定。前几个参数很简单：

```c++
VkImageViewCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

`viewType`和`format`字段指定了如何解释图像数据。`viewType`参数允许你将图像视为1D纹理、2D纹理、3D纹理和立方体贴图。

```c++
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

`components`字段允许你重新排列颜色通道。例如，你可以将所有通道映射到红色通道来创建一个单色纹理。你还可以将常量值`0`和`1`映射到通道。在我们的情况下，我们将使用默认映射。

```c++
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

`subresourceRange`字段描述了图像的用途以及应该访问图像的哪个部分。我们的图像将用作没有任何mipmapping级别或多个图层的颜色目标。

```c++
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

如果您正在开发一个立体3D应用程序，那么可以创建一个具有多个图层的交换链。然后，您可以为每个图像创建多个图像视图，通过访问不同的图层来表示左右眼的视图。

现在创建图像视图只需调用`vkCreateImageView`即可:

```c++
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```

与图像不同，图像视图是由我们显式创建的，因此我们需要在程序结束时添加一个类似的循环来销毁它们:

```c++
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```

图像视图足以将图像用作纹理，但还不能作为渲染目标使用。这需要另一步操作，称为帧缓冲区。但首先，我们需要设置图形管线。

[C++ code](/code/07_image_views.cpp)
