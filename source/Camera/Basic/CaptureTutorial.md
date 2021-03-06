## Introduction

This tutorial describes how to use Zivid SDK to capture point clouds and 2D images.

1. [Initialize](#initialize)
2. [Connect](#connect)
   1. [Specific Camera](#connect---specific-camera)
   2. [Virtual Camera](#connect---virtual-camera)
3. [Configure](#configure)
   1. [Capture Assistant](#capture-assistant)
   2. [Manual Configuration](#manual-configuration)
      1. [Single](#single-frame)
      2. [HDR](#hdr-frame)
      3. [2D](#2d-settings)
   3. [From File](#from-file)
4. [Capture](#capture)
    1. [HDR](#capture-hdr)
    2. [2D](#capture-2d)
5. [Save](#save)
    1. [2D](#save-2d)

### Prerequisites

You should have installed Zivid SDK and C# samples. For more details see [Instructions][installation-instructions-url].

## Initialize

Before calling any of the APIs in the Zivid SDK, we have to start up the Zivid Application. This is done through a simple instantiation of the application ([go to source][start_app-url]).
```csharp
var zivid = new Zivid.NET.Application();
```

## Connect

Now we can connect to the camera ([go to source][connect-url]).
```csharp
var camera = zivid.ConnectCamera();
```

### Connect - Specific Camera

Sometime multiple cameras are connected to the same computer. It might then be necessary to work with a specific camera in the code. This can be done by providing the serial number of the wanted camera.
```csharp
var camera = zivid.ConnectCamera(new Zivid.NET.SerialNumber("2020C0DE"));
```

---
**Note** 

The serial number of your camera is shown in the Zivid Studio.

---

You may also list all cameras connected to the computer, and view their serial numbers through
```csharp
foreach (var cam in zivid.Cameras)
{
    Console.WriteLine("Available camera: " + cam.SerialNumber);
}
```

### Connect - Virtual Camera

You may want to experiment with the SDK, without access to a physical camera. Minor changes are required to keep the sample working ([go to source][filecamera-url]).
```csharp
var zdfFile = Zivid.NET.Environment.DataPath + "/MiscObjects.zdf";
var camera = zivid.CreateFileCamera(zdfFile);
```

---
**Note**

The quality of the point cloud you get from *MiscObjects.zdf* is not representative of the Zivid One+.

---

## Configure

As with all cameras there are settings that can be configured. These may be set manually, or you use our Capture Assistant.

### Capture Assistant

It can be difficult to know what settings to configure. Luckily we have the Capture Assistant. This is available in the Zivid SDK to help configure camera settings ([go to source][captureassistant-url]).
```csharp
var suggestSettingsParameters = new Zivid.NET.CaptureAssistant.SuggestSettingsParameters(Duration.FromMilliseconds(1200), Zivid.NET.CaptureAssistant.AmbientLightFrequency.none);
Console.WriteLine("Running Capture Assistant with parameters: {0}", suggestSettingsParameters);
var settingsList = Zivid.NET.CaptureAssistant.SuggestSettings(camera, suggestSettingsParameters);
```

These settings can be used in an [HDR capture](#capture-hdr), which we will discuss later.

As opposed to manual configuration of settings, there are only two parameters to consider with Capture Assistant.

1. **Maximum Capture Time** in number of milliseconds.
    1. Minimum capture time is 200ms. This allows only one frame to be captured.
    2. The algorithm will combine multiple frames if the budget allows.
    3. The algorithm will attempt to cover as much of the dynamic range in the scene as possible.
    4. A maximum capture time of more than 1 second will get good coverage in most scenarios.
2. **Ambient light compensation**
    1. May restrict capture assistant to exposure periods that are multiples of the ambient light period.
    2. 60Hz is found in (amongst others) Japan, Americas, Taiwan, South Korea and Philippines.
    3. 50Hz is found in most rest of the world.

### Manual configuration

We may choose to configure settings manually. For more information about what each settings does, please see [Zivid One+ Camera Settings][kb-camera_settings-url].

#### Single Frame

We can configure settings for an individual frame directly to the camera ([go to source][settings-url]).
```csharp
camera.UpdateSettings(settings =>
{
    settings.Iris = 20;
    settings.ExposureTime = Duration.FromMicroseconds(8333);
    settings.Brightness = 1;
    settings.Gain = 1;
    settings.Bidirectional = false;
    settings.Filters.Contrast.Enabled = true;
    settings.Filters.Contrast.Threshold = 5;
    settings.Filters.Gaussian.Enabled = true;
    settings.Filters.Gaussian.Sigma = 1.5;
    settings.Filters.Outlier.Enabled = true;
    settings.Filters.Outlier.Threshold = 5;
    settings.Filters.Reflection.Enabled = true;
    settings.Filters.Saturated.Enabled = true;
    settings.BlueBalance = 1.081;
    settings.RedBalance = 1.709;
});
```

#### HDR Frame

We may also set a list of settings to be used in an [HDR capture](#capture-hdr).
```csharp
var irisList = new List<ulong>() { 14, 21, 35 };
var settingsList = new List<Zivid.NET.Settings>();
foreach (var iris in irisList)
{
    Console.WriteLine("Add settings for frame with iris = " + iris);
    var settings = new Zivid.NET.Settings();
    settings.Iris = iris;
    settingsList.Add(settings);
}
```

#### 2D Settings

It is possible to only capture a 2D image. This is faster than a 3D capture, and can be used . 2D settings are configured as follows ([go to source][settings2d-url]).
```csharp
var settings2D = new Zivid.NET.Settings2D()
{
    Iris = 35,
    ExposureTime = Duration.FromMicroseconds(10000),
    Gain = 1.0,
    Brightness = 1
};
```

### From File

Zivid Studio can store the current settings to .yml files. These can be read and applied in the API. You may find it easier to modify the settings in these (human-readable) yaml-files in your preferred editor.
```csharp
camera.SetSettings(new Zivid.NET.Settings("frame_01.yml"));
```

## Capture

Now we can capture a frame. The default capture is a single 3D point cloud ([go to source][capture-url]).
```csharp
var frame = camera.Capture();
```

### Capture HDR

As was revealed in the [Capture Assistant](#capture-assistant) section, a capture may consist of multiple frames. In order to capture multiple frames, and combine them, we can do as follows ([go to source][captureHDR-url])
```csharp
var hdrFrame{ Zivid.NET.HDR.Capture(camera, settingsList) };
```
It is possible to [manually create](#hdr-frame) the `settingsList`, if not set via [Capture Assistant](#capture-assistant).

### Capture 2D

If we only want to capture a 2D image, which is faster than 3D, we can do so via the 2D API ([go to source][capture2d-url]).
```csharp
var frame2D = camera.Capture2D(settings2D);
var image2D = frame2D.Image<Zivid.NET.RGBA8>();
```

## Save

We can now save our results ([go to source][save-url]).
```csharp
frame.Save("result.zdf");
```
The API detects which format to use. See [Point Cloud][kb-point_cloud-url] for a list of supported formats.

### Save 2D

If we captured a 2D image, we can save it ([go to source][save2d-url]).
```csharp
image2D.Save("result.png");
```

## Conclusion

This tutorial shows how to use the Zivid SDK to connect to, configure and capture from the Zivid camera.

[//]: ### "Recommended further reading"

[installation-instructions-url]: ../../../README.md#instructions
[start_app-url]: Capture/Capture.cs#L10
[connect-url]: Capture/Capture.cs#L15
[captureassistant-url]: CaptureAssistant/CaptureAssistant.cs#L15-L17
[settings-url]: Capture/Capture.cs#L18-L24
[kb-camera_settings-url]: https://zivid.atlassian.net/wiki/spaces/ZividKB/pages/99713044/Zivid+One+Camera+Settings
[capture-url]: Capture/Capture.cs#L27
[capture2d-url]: Capture2D/Capture2D.cs#L26
[settings2d-url]: Capture2D/Capture2D.cs#L18-L23
[captureHDR-url]: CaptureAssistant/CaptureAssistant.cs#L26
[save-url]: Capture/Capture.cs#L30
[save2d-url]: Capture2D/Capture2D.cs#L45-L47
[kb-point_cloud-url]: https://zivid.atlassian.net/wiki/spaces/ZividKB/pages/427396/Point+Cloud
[filecamera-url]: CaptureFromFile/CaptureFromFile.cs#L14-L18
