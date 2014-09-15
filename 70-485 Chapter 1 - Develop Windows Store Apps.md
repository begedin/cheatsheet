# Develop Windows Store Apps

## Create background tasks

* A background task implements the `IBackgroundTask` interface, which defines a `Run` method with the signature 
```
public void Run(IBackgroundTaskInstance taskInstance)
```
* You can later use this task with the `BackgroundTaskBuilder` class:
```
var builder = new BackgroundTaskBuilder();
builder.Name = taskName;
builder.TaskEntryPoint = "BikeGPS.BikePositionUpdateBackgroundTask";
```
* Also, you need to set a trigger for when the background task will run:
```
builder.SetTrigger(new SystemTrigger(SystemTriggerType.TimeZoneChange, false));
```
* Possible triggers are `Invalid, SMSReceived, UserAway, NetworkStateChange, ControlChannelReset, InternetAvailable, SessionConnected, ServicingComplete, LockScreenApplicationRemoved, TimeZoneChange, OnlineIdConnectedStateChange`
* An application that registers a background task needs to declare the feature as an extension in the application manifest. The extension section needs to be inserted as a child of the Application tag.
```
<Extensions>
 <Extension Category="windows.backgroundTasks" EntryPoint="Namespace.MyBackgroundTask">
  <BackgroundTasks>
   <Task Type="systemEvent" />
  </BackgroundTasks>
 </Extension>
</Extensions>
```
* The Visual Studio App Manifest Designer can also be used to add a Background Tasks declaration, the details of which can then be edited in a graphical interface.
* To check if a task is active, one can look for it by name through the `BackgroundTaskRegistration.AllTasks` enumerable. An individual item's `task.Value.Name` property can be compared to the locally stored task name.
* For tasks that perform asynchronous work, the Run method needs to return a deferral
```
public async void Run(IBackgroundTaskInstance taskInstance)
{
 BackgroundTaskDeferral _deferral = taskInstance.GetDeferral();

 await MyWorkMethodAsync();
 _deferral.Complete();
}
```
## Consume background tasks

### Understanding task triggers and conditions

* Tasks respond to different kinds of triggers. These can be
 * `MaintenanceTrigger` - raised when it's time to execute system maintenance tasks
 * `SystemEventTrigger` - raised when a specific system event occurs
* Both of these types of triggers implement the `IBackgroundTask` interface
* A `MaintenanceTrigger` is created via
```
var myTaskTrigger = new MaintenanceTrigger(60, true)
```
 * The first parameter is the freshness time, expressed in minutes
 * The second parameter indicates if the trigger should fire only once, or on every freshness time occurence

* Whenever a system event occurs, we can check if specific conditions are met to see if our background task should be performed. These conditions are listed in the `SystemConditionType` enum
 * `InternetAvailable`
 * `InternetNotAvailable`
 * `SessionConnected`
 * `SessionNotConnected`
 * `UserNotPresent`
 * `UserPresent`

* The maintenance trigger can schedule a background task once every 15 minutes, if the device is plugged into a power source, but it will not fire when the device runs on battery.

* An application that makes use of the lock-screen features can also make use of several other triggers
 * `PushNotificationTrigger`- raised when a notificationa arrives on the Windows Push Notification Service channel
 * `TimeTrigger` - raised on scheduled intervals, as frequentyl as once every 15 minutes
 * `ControlChannelTrigger` - raised when there are incomming messages on the control channel for applications that keep connections alive
* The user must place the application on the lock screen before it can access triggers. This can be done with the `RequestAccessAsync` method. The following conditions are available to lock screen enabled applications:
 * ControlChannelReset
 * SessionConnected
 * UserAway
 * UserPresent
* There are two related , self-explanatory events here - `LockScreenApplicationAdded` and `LockScreenApplicationRemoved`
* Time triggered tasks are defined similarly to maintenace triggered tasks
```
TimeTrigger myTaskTrigger = new TimeTrigger(60, true);
```
* Note that if the runOnce flag is set to false, the first time the trigger fires will be at the next, fixed 15 minute interval, meaning sometime between now and in 15 minutes. If it's set to fire only once, then it will fire at exactly 15 minutes after registration time.

### Progressing through and completing background tasks
To get the result of a task execution, the application can provide a callback for the `OnCompleted` event. The callback will receive an instance of a `BackgroundTaskRegistration` initially used to register the task and the event arguments.
```
var builder = new BackgroundTaskBuilder();
builder.name = taskName;
builder.TaskEntryPoint = "MyNamespace.MyBackgroundTask";
builder.SetTrigger(new SystemTrigger(SystemTrigger.Type.TimeZoneChange, false));
builder.AddCondition(new SystemCondition(SystemConditionType.InternetAvailable));
BackgroundTaskRegistration taskRegistration = builder.Register();
taskRegistration.Completed += myCallbackHandler;

void myCallbackHandler(BackgroundTaskRegistration sender, BackgroundTaskCompletedEventArgs args) {
 if (sender.Name == "MyNamespace.MyBackgroundTask") {
  // do stuff
 }
}
```

A background task can be executed when the application is suspended or even terminated. The `OnCompleted` callback will fire then the application resumes or is re-launched. To see the result of a task execution, we can check `args.CheckResult()` It will throw an error if something went wrong.

There is also the `OnProgress` event. A task can set the progress in the `Run()` method through the `Progress` property of the `IBackgroundTask` instance.

### Understanding task constraints
**A background task needs to be ligthweight**. The CPU time for applications not on the lock screen is limited to 1 second. The task can run every 2 hours at a minimum. For applications on the lock screen, the CPU time is limited to 2 seconds and it can run at a maximum interval of 15 minutes.

As for network time, there's a usage limit based on the amount of energy used by the network card when the app is using it.This can vary depending on the device.

To prevent these quotas from affecting real-time communication apps, tasks using `ControlChannelTrigger` and `PushNotificationTrigger` receive a guaranteeed resource quota. If the task exceeds these quotas, it's suspended by the runtime. The `SuspendedCount` property can be checked on the `IBackgroundTask` instance in the `Run` method.

### Cancelling a task
An executing task cannot be stopped unless it recognizes a cancellation request. This has to be checked in the `Run` method.The best way to do this is to attach an `OnCanceled` handler to the `Cancel` event of the task and then set a `volatile bool` private property to `true` in the handler. The `Run` method can then check the private property and `return` if it's set to true.

If the task wants to communicate some data to the application, it can use the local persistent storage to do that, which the application can then check.

### Updating a background task
Tasks can "live through" application updates. If the aplication needs to update a task to modify its behavior, it can register a background task with the 'ServicingComplete' trigger. This task can then unregister no-longer valid tasks:
```
var task = FindTask("MyNameSpace.MyTask")
if (task != null) task.Unregister(true);
```
If the parameter is set to `true`, it will force task cancellation, provided it's implemented.

### Debugging tasks
To debug a background task, a windows store application needs to reference the background task procject and the background task project needs to be configured as a WinMD file. The background task also needs to be declared in the application manifest.

A breakpoint can then be placed in the run metod (or the Debug class can be used to output values). The project needs to be started at least once to register the task in the system, and then the **Debug Location** toolbar can be used to activate the background task.

### Understanding task usage
* Watch out for CPU and network quotas
* The application should get a list of registered background tasks, track their progress oand completion. The tasks should report these things and support cancellation.
* If the Run method uses async code, use deferrals to avoid premature termination by the Windows Runtime.
* All background tasks and associated triggers should be declared in the application manifest.
* Use the `ServicingComplete` trigger to handle updating gracefuly.
* Use the lock screen capabilites, but keep in mind that the user can disable them.
* Use tiles and badges to provide visual information in the run method, as well as notification mechanisms, but do not use any other UI elements.
* Use persistent storage as `ApplicationData` to share data between the task and the application. Do not rely on user interaction in the task.
* Write short-lived tasks.

### Transferring data in the background
The `BackgroundTransfer` namespaces provides options to avoid problems with transferring data in the background. It supports HTTP and HTTPS for download and upload, as well as FTP for download only. It's intended for large file uploads and downloads.

The process started by this class runs separately from the app, so it will not be stopped when the application is suspended.

There are the `BackgroundDownloader` and `BackgroundUploader` classes within the namespace. Both operations support credentials, cookies, use of HTTP headers, can be stopped, paused, resumed or cancelled. They can each be called multiple times because the Windows Runtime handles each operation individually.

To enable these operations, the network needs to be enabled in the manifest, either by enabling the **Internet (Client)**, **Internet (Client & Server)** or **Private Networks** capabilities.
```
var filename = "myFile.mp3"
Uri source = new Uri("http://www.myuri.com/" + filename);
var targetFile = await KnownFolders.MusicLibrary.CreateFileAsync(fileName, CreationCollisionUption.GenerateUniqueName);
var downloader = new BackgroundDownloader();
var downloadOperation = downloader.CreateDownload(source, targetFile);
await downloadOperation.StartAsync();
```
Alternatively, we can do:

```
Progress<DownloadOperation> progressCallback = new Progress<DownloadOperation>(DownloadProgress);
await downloadOperation.StartAsync().AsTask(progressCallback)

private void DownloadProgress(DownloadOperation downloadOperation) 
{
 // do stuff with downloadOperation.Guid, downloadOperation.RequestedUri, downloadOperation.Progress, etc.
}

```

The `BackgroundDownloader` instance has a few useful properties such as `Guid` (unique id), `RequestedUri` (URI from which te file is downloaded, read-only) and `ResultFile` (the `IStorageFile` object provided by the caler when creating the operation. There are also the `Pause` and `Resume` methods, as well as the ˙CostPolicy` property.

Since the system can terminate the application, we can and should reattach the progress and completion event handlers during next launch:
```
IReadOnlyList<DownloadOperation> downloads = await BackgroundDownloader.GetCurrentDownloadsAsync();

if (downloads.count > 0) {
 List<Task> tasks = new List<Task>();
 foreach (DownloadOperation download in downloads) {
  Progress<DownloadOperation> progressCallback = new Progress<DownloadOperation>(DownloadProgress);
  await download.AttachAsync().AsTask(progressCallback);
 }
}
```

The same logic applies for uploads, but there are several overloads for the method.
* `CreateUploadAsync(Uri, IIterable(BackgroundTransferContentPart))` - returns `UploadOperation` with `Uri` and part
* `CreateUploadAsync(Uri, IIterable(BackgroundTransferContentPart, String)` - also returns multipart subtype
* `CreateUploadAsync(Uri, IIterable(backgroundTransferContentPart, String, String)` - also returns multipart delimiter boundary

There's also `CreateUploadfromStreamAsync` which returns an `Uri` and an `IInputStream`

### Keeping communication channels open
To keep a persistent transport connection in the background, an app needs to be lock screen enabled in Windows 8.
Windows Push Notification Service (WNS) is another way to keep communication. It's a cloud service hosted by Microsoft and store apps cna use it to send notifications that can run code, update tiels, or raise on-screen notifications.
If an application si not suitable for WNS but needs a persistend connection (e-mail with some POP3 servers, for instance), it can use the `ControlChannelTrigger`.

## Create and consume WinMD components

### Understanding the Windows Runtime and WinMD
The Windows Runtime is a set of APIs built for the Windows 8 os. It sits on top of the WinRT core engine, which is a set of C++ libraries.

### Consuming a native WinMD library
In a CLR Windows 8 app:
```
var camera = new Windows.Media.Capture.CameraCaptureUI();
var photo = await camera.CaptureFileAsync(Windows.Media.Capture.CameraCaptureUIMode.Photo);
```
we cannot step into the CameraCaptureUI constructor code, because it's external code. On the other hand, in a C++ Windows 8 app:
```
auto camera = ref new Windows::Media::Caputre::CameraCaptureUI();
camera->CaptureFileAsync(Windows::Media::Capture::CameraCaptureUIMode::Photo);
```
we can easily do this.

WinMD is a multilanguage API that adapts its synthax and style to the host language and maintains a common set of behavior capabilities under the cover.

The windows metadata files (WinMD) for this feature are stored in `<OS root path>\System32\WinMetadata`. The default ones are:
* Windows.ApplicationModel.winmd
* Windows.data.winmd
* Windows.Devices.winmd
* Windows.Foundation.winmd
* Windows.Globalization.winmd
* Windows.Graphics.winmd
* Windows.Management.winmd
* Windows.Media.winmd
* Windows.Networking.winmd
* Windows.Security.winmd
* Windows.Storage.winmd
* Windows.System.winmd
* Windows.UI.winmd
* Windows.UI.Xaml.winmd
* Windows.Web.winmd

Any WinMD file can be inspected using the **Intermediate Language Disassembler (IL-DASM)** shipped with Visual Studio 2012 or downloadable with the Microsoft .NET Framework SDK.

### Creating a WinMD library
**Requirements:**
* Fields, parameters and return values of all public types must be WinRT types
* Public structures cannot have any members other than public fields and those fields must be value types or strings
* Public classes must be sealed (`NotInheritable` in Visual Basic). For polymorphism, you can create a public interface and implement it on classes that must be polymorphic. XAML controls are the exception.
* All public types must have a root namespace that matches the assembly name and it must not begin with "Windows".

A new WinMD library is created with the Windows Runtime Component template in Visual Studio. The project outputs a DLL and a WinMD file. 
