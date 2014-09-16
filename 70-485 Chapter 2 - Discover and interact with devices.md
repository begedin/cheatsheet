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
