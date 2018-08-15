---
author: walrusmcd
title: Integrate a model into your app
description: Learn how to use Windows ML to integrate trained machine learning models into your Windows applications.
ms.author: paulm
ms.date: 08/08/2018
ms.topic: article
ms.prod: windows
ms.technology: uwp
keywords: windows 10, windows ai, windows ml, winml, windows machine learning
ms.localizationpriority: medium
---

# Integrate a model into your app with Windows ML

> [!NOTE]
> Windows ML is a **pre-released** product which may be substantially modified before it’s commercially released. Microsoft makes no warranties, express or implied, with respect to the information provided here.
>
> To try out the pre-released Windows ML, you'll need the [Windows 10 Insider Preview](https://www.microsoft.com/en-us/software-download/windowsinsiderpreviewiso) (Build 17728 or higher) and the [Windows 10 SDK](https://www.microsoft.com/en-us/software-download/windowsinsiderpreviewSDK) (Build 17723 or higher).

In this guide, we'll cover how to use the Windows ML APIs to integrate a model into your Windows app. Alternatively, if you'd like to use Windows ML's automatic code generator, check out [mlgen](mlgen.md).

> **Important APIs**: [Windows.AI.MachineLearning](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning)

We'll go over the basic building blocks of Windows ML, which are:

* Models
* Sessions
* Devices
* Bindings

You'll use these to load, bind, and evaluate your models with Windows ML.

We also recommend taking a look at our [sample apps on GitHub](https://github.com/Microsoft/Windows-Machine-Learning/tree/master) to see end-to-end Windows ML code examples.

## Load models

> [!IMPORTANT]
> Windows ML requires ONNX models, version 1.2 or higher.

Once you [get a trained ONNX model](get-onnx-model.md), you'll distribute the .onnx model file(s) with your app. You can include the .onnx file(s) in your APPX package, or, for desktop apps, they can be anywhere your app can access on the hard drive.

There are several ways to load the models using static methods on [**LearningModel**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodel):

* LearningModel::LoadFromStreamAsync
* LearningModel::LoadFromStream
* LearningModel::LoadFromStorageFileAsync
* LearningModel::LoadFromFilePath

The stream versions of load() allow applications to have more control over where the model comes from. For example, an app could choose to have the model encrypted on disk and decrypt it only in memory prior to calling load(). Other options include loading the model stream from a network share or other media.

> [!TIP]
> Loading a model can take some time, so take care not to call load() from your UI thread.

## Create a session

Once you load a [**LearningModel**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodel), you create a [**LearningModelSession**](https://docs.microsoft.com/en-us/uwp/api/windows.ai.machinelearning.learningmodelsession), which binds the model to a device that runs and evaluates the model.

## Choose a device

You can select a device when you create a session. If the device becomes unavailable, or if you'd like to use a different device, you must close the session and create a new session.

You choose a device of type [**LearningModelDeviceKind**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodeldevicekind
):

* **Default**
	* Let the system decide which device to use. Currently, the default device is CPU.
* **CPU**
	* Use the CPU, even if other devices are available.
* **DirectX**
	* Use a DirectX hardware acceleration device, specifically the first adapter enumerated by IDXGIFactory1::EnumAdapters().
* **DirectXHighPerformance**
	* Same as DirectX, but will use DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE when enumerating adapters.
* **DirectXMinPower**
	* Same as DirectX, but will use DXGI_GPU_PREFERENCE_MINIMUM_POWER when enumerating adapters.

If you don't specify a device, the system uses **Default**. We recommend using **Default** to get the flexibility of allowing the system choose for you in the future.

### Device removal (advanced)

In some cases, graphics devices might need to be unloaded and reloaded, as explained in the [DirectX documentation](https://docs.microsoft.com/windows/uwp/gaming/handling-device-lost-scenarios).

When using Windows ML, you'll need to detect this case and close the session. To recover from a device removal or re-initialization, you'll create a new session, which triggers the device selection logic to run again.

The most common case where you will see this error is during [LearningModelSession::Evaluate()](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelsession.evaluate). In the case of device removal or reset, [LearningModelEvaluationResult::ErrorStatus](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelevaluationresult.errorstatus#Windows_AI_MachineLearning_LearningModelEvaluationResult_ErrorStatus) will be DXGI_ERROR_DEVICE_REMOVED or DXGI_ERROR_DEVICE_RESET.

## Reflect on model features

A machine learning model has input and output features, which pass information into and out of the model.

After you load your model as a LearningModel, you can use [LearningModel::InputFeatures()](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodel.inputfeatures) and [LearningModel::OutputFeatures()](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodel.outputfeatures) to get ILearningModelFeatureDescriptors. These list the model's expected input and output feature types.

Windows ML supports all ONNX feature types, which are enumerated in [**LearningModelFeatureKind**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelfeaturekind):

* Tensor
* Sequence
* Map
* Image

You can reference an ILearningModelFeatureDescriptor ([TensorFeatureDescriptor](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.tensorfeaturedescriptor), [SequenceFeatureDescriptor](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.sequencefeaturedescriptor), [MapFeatureDescriptor](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.mapfeaturedescriptor), [ImageFeatureDescriptor](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.imagefeaturedescriptor)) by its Name property.

## Bind inputs and outputs

You use [**LearningModelBinding**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelbinding) to bind values to a feature, referencing the ILearningModelFeatureDescriptor by its Name property.

### Tensors

Tensors are multi-dimensional arrays, and the most common tensor is a tensor of 32bit floats. The dimensions of tensors are row-major, with tightly packed contiguous data representing each dimension. The tensor's total size is the multiplication product of the size of each dimension.

### Images

When working with images, you'll need to be aware of image formats and tensorization.

#### Image formats

Models are trained with image training data, and the weights are saved and tailored for that training set. When you pass an image input into the model, its format must match the training images' format.

In many cases, the model describes the expected image format; ONNX models can use [metadata](https://github.com/onnx/onnx/blob/master/docs/MetadataProps.md) describe expected image formats.  

Most models use the following formats, but this is not universal to all models.

* Image.BitmapPixelFormat	Bgr8
* Image.ColorSpaceGamma	SRGB
* Image.NominalPixelRange	NominalRange_0_255

#### Tensorization

Images are represented in Windows ML in a tensor format. Tensorization is the process of converting an image into a tensor and happens during evaluation.

Windows ML converts images into 4 dimensional tensors of 32bit floats in the "NCHW tensor format":
* N is batch size (or number of images). Windows ML supports a batch size N of 1.
* C is channel count (1 for Gray8, 3 for Bgr8).
* H is height.
* W is width.

Each pixel of the image is an 8bit color number that is stored in the range of 0-255 and packed into a 32bit float.

#### How to pass images into the model

There are 2 ways you can pass images into models:

1. [**ImageFeatureValue**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.imagefeaturevalue)

    We recommend using **ImageFeatureValue** to bind images as inputs and outputs, as it takes care of both conversion and tensorization, so the images match the model's required image format. The currently supported model format types are Gray8, Rgb8, and Bgr8, and the currently supported pixel range is 0-255.

    You can create an ImageFeatureValue using the static method **ImageFeatureValue::CreateFromVideoFrame**.

    To find out what format the model needs, we use the following logic and precedence order:

	1. **BindWithProperties** will override all image settings.
	2. **Model metadata** will then be checked and used if available.
	3. **Best match**: If no model metadata is provided, and no caller supplied properties, the runtime will attempt to make a best match.   If the tensor looks like NCHW (4 dim float32, N==1), the runtime will assume either Gray8 or Bgr8 depending on the channel count.

	There are several optional properties that you can pass into [Bind() with properties](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelbinding.bind):

	* BitmapBounds - if specified, these are the cropping boundaries to apply prior to sending the image to the model.
	* BitmapPixelFormat - if specified, this is the pixel format that will be used as the MODEL pixel format during image conversion.

	For image shapes, the model can specify either a specific shape that it takes (e.g. SqueezeNet takes 224,224), or the model can specify free dimensions for any shape image (many StyleTransfer type models can take variable sized images). The caller can use BitmapBounds to choose which section of the image they would like to use. If not specified, the runtime will scale the image to the model size (respecting aspect ratio) and then center crop.  

2. [**TensorFloat**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.tensorfloat)

    If Windows ML does not support your model's color format or pixel range, then you can implement conversions and tensorization. You'll create a NCHW four dimensional tensor for 32bit floats for your input value. <We have a sample of how to do this here: link to manual image tensorization sample>

    When this code path is used, any image metadata on the model is ignored.

### Maps

Maps are key/value pairs of information. Classification models commonly return a string/float map that describes the float probability for each labeled classification name.

### Sequences

Sequences are vectors of values. A common use of sequence types is a vector of float probabilities. Some classification models return a sequence of floats, which represent the resulting probabilities.  

### Scalars

Most maps and sequences will have values that are scalars. These show up where TensorFeatureDescriptor.Shape.Size is zero (0). In this case, the map or sequence will be of the scalar type. The most common is `float`.  For example, a string to float map would be:

```cs
MapFeatureDescriptor.KeyKind == TensorKind.String
MapFeatureDescriptor.ValueDescriptor.Kind == LearningModelFeatureKind.Tensor
MapFeatureDescriptor.ValueDescriptor.as<TensorFeatureDescriptor>().Shape.Size == 0
```

The actual map feature value will be a `IMap<string, float>`.

## Call evaluate

Finally, to run the model, you call any of the Evaluate() methods on your [**LearningModelSession**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelsession). You can use the [**LearningModelEvaluationResult**](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning.learningmodelevaluationresult) to look at the output features.

## Related topics

* [mlgen](mlgen.md)
* [Performance and memory](performance-memory.md)
* [API reference](https://docs.microsoft.com/uwp/api/windows.ai.machinelearning)
* [Code examples](https://github.com/Microsoft/Windows-Machine-Learning/tree/master)