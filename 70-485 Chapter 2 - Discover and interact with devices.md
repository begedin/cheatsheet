# Discover and interact with devices
## Capture media with the camera and microphone
Use the provided user interface (with the `CameraCaptureUI` API) whenever possible, or use code to get complete control of the operation if needed.

You need to **declare the Webcam capability** to use the Webcam API.The user will then be prompted to allow the use of the webcam.If the use is denied, there will simply be no photo taken, so the code that handles capturing should take that into account. 
```
var camera = new CameraCaptureUI();
var img = await camera.CaputreFileAsync(CameraCaptureUIMode.Photo);
if (img != null) {
  var stream = await img.OpenAsync(FileAccessMode.Read);
  var bitmap = new BitmapImage();
  bitmap.SetSource(stream);
  myImageControl.Source = bitmap;
} else {
  // do something else
}
```
The system remembers the user's choice for permissions, but it can be restored later in the application's settings panel on the charms bar.

The CameraCaptureUI also has some setting properties:
```
camera.PhotoSettings.AllowCropping = true;
camera.PhotoSettings.Format = CameraCaptureUIPhotoFormat.Jpeg; //Jpeg, JpexXR or Png
camera.PhotoSettings.MaxResolution = CameraCaptureUIMaxPhotoResolution.HighestAvailable;
```

For resolution, the options are `HighestAvailable`, `VerySmallQvga` (up to 320x240), `SmallVga` (also up to 320x240), `MediumXga` (up to 1024x768), `Large3M` (up to 1920x1080), `VeryLarge5M` (up to 5MP).

Once a photo has been taken, the `CroppedAspectRatio` and `CroppedSizeInPixels` properties of the `PhotoSettings` property of the `CameraCaptureUI` instance can be read to determine the actual size of the cropped area.

With `CameraCaptureUIMode.Vide`, we can also capture video. There, we can set properties on the `VideoSettings` property:
```
camera.VideoSettings.AllowTrimming = true;
camera.VideoSettings.Format = CameraCaptureUIVideoFormat.Mp4; // or Wmv
camera.VideoSettings.MaxDurationInSeconds = 30;
camera.VideoSettings.MaxResolution = CameraCaptureUIMaxVideoResolution.HighDefinition;
```
For resolution, the options are `HighestAvailable`, `LowDefinition`, `StandardDefinition` and `HighDefinition`. If audio should be recorded along side video, the Microphone capability needs to be declared in the manifest.

### Using `MediaCapture` to capture pictures, video or audio

For more direct control of the capturing process, the `MediaCapture` class can be used. The `CameraCaptureUI` API actually uses the `MediaCapture` API internally. To use this API, capabilities such as Webcam or Microphone need to be declared here as well. To store the captured file to say, the Video Library, the Video Library capability should also be declared.

#### How to use the `MediaCapture` API to display a video stream from the webcam
```
private MediaCapture _mediaCapture;

private async void StartDevice 
{
  try 
  {
    this._mediaCapture = new MediaCapture();
  
    this._mediaCapture.RecordLimitationExceeded += MediaCapture_RecordLimitationExceeded;
    this._mediaCapture.Failed += MediaCapture_Failed;
    
    await this._mediaCapture.InitializeAsync();
    
    // there is an overload here, so we can also do it this way:
    var settings = new MediaCaptureInitializationSettings();
    settings.StreamingCaptureMode = StreamingCaptureMode.AudioAndVideo;
    settings.PhotoCaptureSource = PhotoCaptureSource.VideoPreview;
    await this._mediaCapture.InitializeAsync(settings);
  } 
  catch (UnauthorizedAccessException ex) 
  {
    // do stuff
  }
  catch (Exception ex)
  {
    // do other stuff
  }
}

private async void MediaCapture_failed (MediaCapture sender, MediaCaptureFailedEventArgs errorEventArgs)
{
  await Dispatcher.RunAsync(Windows.UI.Core.CoreDispatcherPriority.Normal, () =>
  {
    // display error
  }
}

private async void MediaCapture_RecordLimitationExceeded (MediaCapture sender) 
{
  await this._mediaCapture.StopRecordAsync();
  await Dispatcher.RunAsync(Windows.UI.Core.CoreDispatcherPriority.Normal, () => 
  {
    // display error
  }
}
```

After initialization, here's how we start the preview:
```
await this._mediaCapture.AddEffectAsync(MediaStreamType.VideoPreview, VideoEffects.VideoStabilization, null);
PreviewElement.Source = this._MediaCapture;

await this._mediaCapture.StartPreviewAsync();
```
Note the `AddEffectAsync` method. There's a counterpart - `ClearEffectsAsync`. The third parameter is a dictionary implementing the `IPropertySet` interface and containing configuration parameters for the effect.

#### How to take a photo
```
ImageEncodingProperties encodingProperties = ImageEncodingProperties.CreateJpeg();
WriteableBitmap bitmap = new WriteableBitmap((int)ImageElement.Width, (int)ImageElement.Height);

using (var imageStream = new InMemoryRandomAccessStream()) 
{
  await this._mediaCapture.CapturePhotoStreamAsync(encodingProperties, imageStream);
  await imageStream.FlushAsync();
  imageStream.Seek(0);
  bitmap.SetSource(imageStream);
  
  await Dispatcher.RunAsync(Windows.Ui:Core.CoreDispatcherPriority.normal, () => 
  {
    ImageElement.Source = bitmap;
  }
}
```
There is also the `CapturePhotoToStorageFileAsync` for capturing and saving directly as a storage file.

The `MediaCaptureInitializationSettings` class also has `VideoDeviceId` and `AudioDeviceId` properties for selecting a specific video/audio device when there are multiple present.

```
var devices = await Windows.Devices.Enumeration.DeviceInformation.FindAllAsync(
                      Windows.Devices.Enumeration:DeviceClass.VideoCapture);
//...                      
settings.VideoDeviceId = devices[0].Id;
```
For videos, there are `StartRecordToStreamAsync` and `StartRecordToStorageFileAsync` classes, which accepts a `MediaEncodingProfile` object as the first parameter. There's a third class - `StartRecordToCustomSinkAsync`, which records to a custom media sink. 

Some basic camera settings can be adjusted through `CameraOptionsUI.Show()`:
```
  if (this._mediaCapture != null) 
  {
    CameraOptionsUI.Show(this._mediaCapture);
  }
```
To stop recording, we can call `StopRecordAsync`.

## Get data from sensors

### Example of using sensors with the Accelerometer

The various sensor APIs are built on top of the **Windows Sensor and Location platform**. The `Windows.Devices.Sensors` namespace provides classes, methods and types to access them.

In most cases, we obtain a reference to the class that wraps a specific sensor with the `GetDefault` static method of the sensor class and then either poll the device at regular intervals, or subscribe to the`ReadingChanged` event.
```
Accelerometer accelerometer = Accelerometer.GetDefault();
if (accelerometer == null) { /* no accelerometer found */ }
else 
{
  uint minReportInterval = accelerometer.MinimumReportInterval;
  var desiredReportInterval = minReportInterval > 16 ? minReportInterval : 16;
  accelerometer.ReportInterval = desiredReportInterval; // desired interval (in ms) between two readings. 0 means default.
  
  accelerometer.ReadingChanged += Accelerometer_ReadingChanged;
}

private async void Accelerometer_ReadingChanged(Accelerometer sender, AccelerometerReadingChangedEventArgs args) 
{
  if (args.Reading != null) 
  {
    // do stuff with args.Reading.AccelerationX,args.Reading.AccelerationY,args.Reading.AccelerationZ, args.Reading.Timestamp
  }
}
```

Polling the sensor at regular intervals usually involves using a `DispatcherTimer`
```
// in initialization method
DispatcherTimer timer = new DispatcherTimer();
timer.Tick += PolLAccelerometerSensorReadings; // the method then uses GetCurrentReading() ont he accelerometer instance
timer.Interval = new TimeSpan(0,0,0,0, (Int32)desiredReportInterval);
timer.Start();
```
Specifically, the `Accelerometer` class also exposes a `Shaken` event, so we can subscribe to that.

### The gryrometer sensor
The gyrometer measuers angular velocity along three axes. The usage is again involes the `GetDefault` static method.The properties for the reading are `AngularVelocityX`, `AngularVelocityY`, `AngularVelocityZ` and `Timestamp`. 

The report interval actually affects change sensitivity. For intervals of 1 to 16, the sensitivity is 0.1 degrees per second. For intervals from 17 to 32, it's 0.5 and for itnervals 33 or greater, it's 1 degree per second.

### The compass sensor
The compass indicates the heading in degrees relativ to the magnetic north, or depending on the implementation of the sensors, the geographic north. For that, it has two properties in the `CompassReading` class - `HeadingMagneticNorth` and `HeadingTrueNorth`.

The report interval again affects change sensitivity. 1 to 16 equals to a sensitivity of 0.01 degrees, 17 to 32 is 0.5 and 33 or more is 2 degrees.

### The orientation sensors

The orientations sensor combines the other 3 sensors to provide more complex data. It returns an orientation,w hich can be one of several things - `NotRotated`, `Rotated90DegreesCounterClockwise`, `Rotated180DegreesCounterClockwise`, `Rotated270DegreesCounterClockwise`, `Faceup` and `Facedown`.

Using the `SimpleOrientationSensor` class follows the same patters of the other sensors, for the most part. The event is now `OrientationChanged`. 

There is also the `OrientationSensor` class which returns more detailed data int he form o f a `SensorRotationMatrix` or a `Quaternion`, retrieved from `args.Reading.RotationMatrix` and `args.Reading.Quaternion` respectively.

### The inclinometer sensor

The inclinometer determines rotation around the three axes. X is also known as pitch or beta, Y is roll or gamma and Z is yaw or alpha. Because of that, there are `RolLDegrees`, `PitchDegrees` and `YawDegrees properties` on the reading. The sensitivity depends on the report interval in the exact same way as the gyrometer sensor does.

### The light sensor

The light sensor detects light levels. They are measured as lumminence in lux and stored in the `LuminenceInLux` property of the `LightSensorReading` class. The sensitivity for a report interval of 1 to 32 ms is 1%, and anything after 33 ms is 5%.

### Location

The location can be retrieved with the Windows Location Provider, which uses Wi-Fi triangulation and IP address data, or throught he GPS sensor. The Location API determines the most accurate sensor for a given scenario. Both require the location capability to be declared in the manifest.
```
Geolocator geolocator = new Geolocator();
geolocator.DesiredAccuracy = PositionAccuracy.High;
/* PositionStatus enum { Ready, Initializing, NoData, Disabled, notInitialized, NotAvailable } */
geolocator.StatusChanged += Geolocator_StatusChanged; 
var position = await geolocator.GetGeopositionAsync();
```

The `Geoposition.Coordinate` object will contain `Longitude`, `Lattitude`, `Heading`, `Speed`, `Altitude`, `Accuracy` and `Timestamp` properties. The `Geoposition` object can also contain the `CivicAddress` property object, but only if a Civic Addres Provider is available on the system. Otherwise, it will simply return the regional information accessible from the Control Panel.

`GetGeopositionAsync` also has an overload with two `TimeSpan` objects as parameters. The first is the maximum acceptable age of the date, and the second is the timeout.

There is also the `PositionChanged` event, raised every time there's a change in the geographical location detected. For that, there's a `MovementTresholdProperty` accepting values in meters.

## Enumerating devices

```
var list = new List<DeviceItem>();

var devices = await Windows.Devices.Enumeration.DeviceInformation.FindAllAsync();
if (devices != null && devices.Count > 0) 
{
  foreach (DeviceInformation device in devices) 
  {
    var glyph = await device.GetGlyphThumbnailAsync();  // returns BitmapImage
    var thumb = await device.GetThumbnailAsync();       // returns BitmapImage
    
    list.Add(new DeviceItem(device, thumb, glyph)); // DeviceItem is a custom class, not a default one
  }
}
```

The `DeviceInformation` class exposes various overloaded `FindAllAsync` methods, some of which accept a `DeviceClass` parameter, which is an enum with possible values of `All`, `AudioCapture`, `AudioRender`, `PortableStorageDevice` and `VideoCapture`. `FindAllAsync` can also accept an Advanced Query Synthax (AQS) selector.

The `DeviceWatcher` class raises watches devices dynamically and raises events - `Added`, `EnumerationCompleted`, `Removed`, `Stopped` and `Updated`. A reference to the class is botained by calling the static `CreateWatcher` method of the `DeviceInformation` class. Search can begin by calling the `Start()` metod on the obtained reference. 

`DeviceWatcher` also has various overloads accepting filters. It raises an `Added` event every time it finds and adds a device and ends it all with an `EnumerationCompleted` event. It stores its current status within the `Status` property as a `DeviceWatcherStatus` enum. The possible values are consistent with the events triggered.

### Enumerating Plug and Play (PnP) devices
Plug and Play device enumeration is enabled with the `Windows.Devices.Enumeration.PnP` namespace. It enables enumerating devices, device interfaces, device interface classes and device containers.

A device interface is a symbolic link to the plug and play device that an application can use to access it. A device interface class is the interface that represents functionalities exposed by a class of devices. A device container represents the pyhsical device as seen by the user. It allows you to obtain information related to the actual hardware product.

This is all provided through the `PnpObject` class, the constructor of which expects a `PnpObjectType` enun, which can be `Unknown`, `DeviceInterface`, `DeviceContainer` or `DeviceInterfaceClass`.
```
var deviceContainers = await PnpObject.FindAllAsync(pnpObjectType.DeviceContainer);
```

There is also a `PnpDeviceWatcher` object with `Added`, `Removed`, `Updated` and `EnumerationCompleted` events, used in the same way as `DeviceWatcher`.
