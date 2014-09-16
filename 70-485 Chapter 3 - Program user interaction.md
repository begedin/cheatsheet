# Program user interaction

## Implement printing

### Registering a Windows Store app for printing

* Obtain a reference to the `PrintManager` instance for each view that requires printing
```
protected override void OnNavigatedTo(NavigationEventArgs e)
{
  PrintManager printManager = PrintManager.GetForCurrentView();
  printManager.PrintTaskRequested += PrintManager_PrintTaskRequested; // when the user requests printing through the charm bar
}
```
* Remeber to unsubscribe when leaving the page
```
protected override void OnNavigatedFrom(NavigationEventArgs e)
{
  PrintManager printManager = PrintManager.GetForCurrentView();
  printManager.PrintTaskRequested -= PrintManager_PrintTaskRequested;
}
```
* Implement a `PrintTask` instance representing the actual printing operation
```
private void PrintManager_PrintTaskRequested(PrintManager sender, PrintTaskRequestedEventArgs args)
{
  PrintTask printTask = null;
  
  var deferral = args.Request.GetDeferral(); // long-running operation, so a deferral is used
  
  printTask = args.Request.CreatePrintTask(“Windows 8 Print Sample”, (sourceRequested) =>
  {
    sourceRequested.SetSource(this._documentSource);
  });
  
  deferral.Complete();
}
```
* Create a `PrintDocument` instance to hold the content that you want to print and handle print-related events. In the above coude, `this._documentSource` is the required instance.


