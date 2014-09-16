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

