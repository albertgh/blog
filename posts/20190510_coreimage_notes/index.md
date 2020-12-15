# CoreImage with Metal 笔记

`date`: 2019-05-10
---

`iOS 12` 开始 `CIKernel Language` 已被苹果标记废弃, 并推荐转而使用 `Core Image Metal Library`, 关于 shader 语法的转变以及如何生成 `.metallib`, 官方文档都有详细说明 [MetalCIKLReference](https://developer.apple.com/metal/MetalCIKLReference6.pdf), 按照文档在 `Build Settings` 中将 `Other Metal Compiler Flags` 设为 `-fcikernel`, 在 `User-Defined` 中增加 `MTLLINKER_FLAGS` 并设为 `-cikernel`, OK, demo 运行成功.

但这里有一个问题, 默认是只生成一个 `default.metallib` 单个文件的, 且需要把所有 shader 写在一个 `.metal` 文件里, 网上很多 Sample 也都是如此操作的... 

这显然不符合实际开发中的需求, 那么我们应该可以自行手动编好各司其职的 `.metallib` 文件. 通过查阅相关文档 [Building a Library with Metal's Command-Line Tools](https://developer.apple.com/documentation/metal/libraries/building_a_library_with_metal_s_command-line_tools), 以及前一个文档, 都有提到如何生成 `.metallib`

>
```
To compile a Metal shader with CIKernel objects, specify the -fcikernel option. 
xcrun metal -fcikernel MyKernels.metal -o MyKernels.air 
To link a Metal shader with CIKernel code, specify the -cikernel option. 
xcrun metallib -cikernel MyKernels.air -o MyKernels.metallib 
```

但实际操作后, Xcode 会各种提示找不到 shader 里的方法, 在网络上搜索了半天才发现是文档没写全, 生成 `.air` 文件时还需要加上 `-c` 命令 [metallib: Error](https://stackoverflow.com/questions/52992783/metallib-error-reading-module-invalid-bitcode-signature), 所以例如给 `iOS` 使用的 `.metallib` 文件可由如下命令生成.

```
xcrun -sdk iphoneos metal -fcikernel YourFilter.metal -c -o YourFilter.air
xcrun -sdk iphoneos metallib -cikernel YourFilter.air -o YourFilter.metallib
```

这样就可以让各个自定义 filter 加载不同的 `.metallib` 文件了.

filter 比较多的话还可以写个脚本自动跑一下, 如下.

```sh
#!/bin/sh

xcrun -sdk iphoneos metal -fcikernel ACImageMaskCIFilter.metal -c -o ACImageMaskCIFilter.air
xcrun -sdk iphoneos metallib -cikernel ACImageMaskCIFilter.air -o ACImageMaskCIFilter.metallib

xcrun -sdk iphoneos metal -fcikernel ACImageOpacityCIFilter.metal -c -o ACImageOpacityCIFilter.air
xcrun -sdk iphoneos metallib -cikernel ACImageOpacityCIFilter.air -o ACImageOpacityCIFilter.metallib

xcrun -sdk iphoneos metal -fcikernel ACImageClipCIFilter.metal -c -o ACImageClipCIFilter.air
xcrun -sdk iphoneos metallib -cikernel ACImageClipCIFilter.air -o ACImageClipCIFilter.metallib

xcrun -sdk iphoneos metal -fcikernel ACImageLuminanceCIDetector.metal -c -o ACImageLuminanceCIDetector.air
xcrun -sdk iphoneos metallib -cikernel ACImageLuminanceCIDetector.air -o ACImageLuminanceCIDetector.metallib

rm *.air

echo "done"
```

也可以通过给 `Xcode` 加上 `Build Rules` 实现 [Compile each .metal file into a separate .metallib](https://stackoverflow.com/questions/54914564/compile-each-metal-file-into-a-separate-metallib/54921611#54921611).

---
#### 还有一个值得说一下的就是 `CIFilter` 自带的一些滤镜的参数列表以及参数值域在哪找的问题

在调试 `CIExposureAdjust` 滤镜的时候想确认 `kCIInputEVKey` 的值域是否和以前一样时发现并不能直接看到, 查阅官方文档 [Core Image Filter Reference](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Reference/CoreImageFilterReference/#//apple_ref/doc/filter/ci/CIExposureAdjust), 好像也没能看到... 

最后才找到了查参的方法 [Value ranges for iOS Core Image Filters](https://stackoverflow.com/questions/22704372/value-ranges-for-ios-core-image-filters-exposure-hue-sepia-temparature-tint), [Querying the System for Filters](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_concepts/ci_concepts.html#//apple_ref/doc/uid/TP30001185-CH2-TPXREF101).

简单说就是打印 `CIFilter` 对象的 `attributes`. 例如 `CIExposureAdjust` 的 `kCIInputEVKey` 值域可从如下信息中找到, 为 `-10`~`10`.

```js
{
    "CIAttributeFilterAvailable_Mac" = "10.4";
    "CIAttributeFilterAvailable_iOS" = 5;
    CIAttributeFilterCategories =     (
        CICategoryColorAdjustment,
        CICategoryVideo,
        CICategoryStillImage,
        CICategoryInterlaced,
        CICategoryNonSquarePixels,
        CICategoryBuiltIn,
        CICategoryXMPSerializable
    );
    CIAttributeFilterDisplayName = "Exposure Adjust";
    CIAttributeFilterName = CIExposureAdjust;
    CIAttributeReferenceDocumentation = "http://developer.apple.com/library/ios/documentation/GraphicsImaging/Reference/CoreImageFilterReference/index.html#//apple_ref/doc/filter/ci/CIExposureAdjust";
    inputEV =     {
        CIAttributeClass = NSNumber;
        CIAttributeDefault = 0;
        CIAttributeDescription = "The amount to adjust the exposure of the image by. The larger the value, the brighter the exposure.";
        CIAttributeDisplayName = EV;
        CIAttributeIdentity = 0;
        CIAttributeSliderMax = 10;
        CIAttributeSliderMin = "-10";
        CIAttributeType = CIAttributeTypeScalar;
    };
    inputImage =     {
        CIAttributeClass = CIImage;
        CIAttributeDescription = "The image to use as an input image. For filters that also use a background image, this is the foreground image.";
        CIAttributeDisplayName = Image;
        CIAttributeType = CIAttributeTypeImage;
    };
}
```

---
#### 完整 Demo 见 [ACCoreImageFilters](https://github.com/albertgh/ACCoreImageFilters).


