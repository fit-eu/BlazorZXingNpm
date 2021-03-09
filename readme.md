# BlazorZXingNpm 
## inspired by and partially based on: https://github.com/sabitertan/BlazorBarcodeScanner 
## focused on how to use npm @zxing/library

* Create a new Blazor WASM Project BlazorZXingNpm

* create a folder "NpmJs" and inside another one "src"
* open this folder in Terminal

* npm init  ==> keep default values

* npm install @zxing/library
* npm install webpack webpack-cli --save-dev

* inside NpmJs/src create zxing.js with:
```
    import {  BrowserMultiFormatReader, NotFoundException} from '@zxing/library';
    export {  BrowserMultiFormatReader, NotFoundException};
```


* inside NpmJs create zxing.config.js with:
```
    const path = require("path");

    module.exports = {
        module: {
            rules: [
                {
                    test: /\.(js|jsx)$/,
                    exclude: /node_modules/,
                }
            ]
        },
        output: {
            path: path.resolve(__dirname, '../wwwroot/js'),
            filename: "myzxing.js",
            library: "ZXing"
        }
    };
```


* in scripts section of NpmJs/package.json insert :
```
    "build_zxing": "webpack ./src/zxing.js --mode production --config zxing.config.js"
```

```
    {
      "name": "npmjs",
      "version": "1.0.0",
      "description": "",
      "main": "index.js",
        "scripts": {
            "test": "echo \"Error: no test specified\" && exit 1",
            "build_zxing": "webpack ./src/zxing.js --mode production --config zxing.config.js"
        },
      "author": "",
      "license": "ISC",
      "dependencies": {
        "@zxing/library": "^0.18.4"
      }
    }
```

* in Terminal execute:
    npm run build_zxing

*  this creates wwwroot/js/myzxing.js

* at the end of the body section of  wwwroot\index.html insert
```
    <script src="js/myzxing.js"></script>
```

* create a new file /JsInteropClass.cs wíth (you may need to update namespace if different)


```
    using Microsoft.JSInterop;  
    using System;  
    using System.Collections.Generic;  
    using System.Threading.Tasks;  
      
    namespace BlazorZXingNpm
    {
        public class JsInteropClass  
        {  
            public static ValueTask<List<VideoInputDevice>> GetVideoInputDevices(IJSRuntime jsRuntime, string message)  
            {  
                // Implemented in BlazorBarcodeScannerJsInterop.js  
                  
                return jsRuntime.InvokeAsync<List<VideoInputDevice>>(
                    "BlazorBarcodeScanner.listVideoInputDevices",
                    message);
            }
            public static void StartDecoding(IJSRuntime jSRuntime, string videoElementId)
            {
                jSRuntime.InvokeVoidAsync("BlazorBarcodeScanner.startDecoding", videoElementId);
            }
            public static void StopDecoding(IJSRuntime jSRuntime)
            {
                jSRuntime.InvokeVoidAsync("BlazorBarcodeScanner.stopDecoding");
            }
            public static void SetVideoInputDevice(IJSRuntime jSRuntime, string deviceId)
            {
                jSRuntime.InvokeVoidAsync("BlazorBarcodeScanner.setSelectedDeviceId", deviceId);
            }
            [JSInvokable]
            public static void ReceiveBarcode (string barcodeText)
            {
                if (!String.IsNullOrEmpty(barcodeText))
                {
                    Console.WriteLine(barcodeText);
                    BarcodeReceivedEventArgs args = new BarcodeReceivedEventArgs();
                    args.BarcodeText = barcodeText;
                    args.TimeReceived = DateTime.Now;
                    OnBarcodeReceived(args);
                }

            }
            protected static void OnBarcodeReceived(BarcodeReceivedEventArgs args)
            {
                BarcodeReceivedEventHandler handler = BarcodeReceived;
                BarcodeReceived?.Invoke(args); //same as below
                // if(handler != null ){
                //    handler(this, e);
                // }
            }
            public static event BarcodeReceivedEventHandler BarcodeReceived;
        }
        public class BarcodeReceivedEventArgs : EventArgs
        {
            public string BarcodeText { get; set; }
            public DateTime TimeReceived { get; set; } = new DateTime();
        }
        public delegate void BarcodeReceivedEventHandler(BarcodeReceivedEventArgs args);
        public class VideoInputDevice
        {
            public string DeviceId { get; set; }
            public string GroupId { get; set; }
            public string Kind { get; set; }
            public string Label { get; set; }

        }
    }
```

* create a new file BlazorBarcodeScanner.js with

```

console.log("Init BlazorBarcodeScanner");
window.BlazorBarcodeScanner = {
    codeReader: new ZXing.BrowserMultiFormatReader(),
    listVideoInputDevices: async function () { return await this.codeReader.listVideoInputDevices(); },
    selecteDeviceId: undefined,
    setSelectedDeviceId: function (deviceId) {
        this.selecteDeviceId = deviceId;
    },
    startDecoding: function (videoElementId) {
        this.codeReader.decodeFromVideoDevice(this.selectedDeviceId, videoElementId, (result, err) => {
            if (result) {
                console.log(result);
                DotNet.invokeMethodAsync('BlazorZXingNpm','ReceiveBarcode', result.text)
                    .then(message => {
                        console.log(message);
                    });
            }
            if (err && !(err instanceof ZXing.NotFoundException)) {
                console.error(err);
                DotNet.invokeMethodAsync('BlazorZXingNpm', 'ReceiveBarcode', err)
                    .then(message => {
                        console.log(message);
                    });
            }
        })
        console.log(`Started continous decode from camera with id ${selectedDeviceId}`);
    },
    stopDecoding: function () {
        this.codeReader.reset();
        DotNet.invokeMethodAsync('BlazorZXingNpm', 'ReceiveBarcode', '')
            .then(message => {
                console.log(message);
            });
        console.log('Reset.');
    }
};

```


* at the end of the body section of  wwwroot\index.html insert
```
    <script src="js/BlazorBarcodeScanner.js"></script>
```

* create new folder Components
* inside Components create a new component BarcodeReader.razor with

```
    @using Microsoft.JSInterop
    @inject IJSRuntime JSRuntime

    <section class="zxing-container" id="zxingContent">
        <h1 class="zxing-title">@Title</h1>
        <div>
            @if (ShowStart)
            {
                <a class="btn btn-primary" id="zxingStartButton" @onclick="StartDecoding">Start</a>
            }

            @if (ShowReset)
            {
                <a class="btn btn-primary" id="zxingResetButton" @onclick="StopDecoding">Reset</a>
            }
        </div>

        <br />
        <div id="zxingVideoContainer">
            <video id="zxingVideo" width="@VideoWidth" height="@VideoHeigth"></video>
        </div>
        @if (ShowVideoDeviceList)
        {


            @if (videoInputDevices == null)
            {
                <p>looking for devices</p> }
            else
            {
                <div id="sourceSelectPanel">
                    <label for="zxingSourceSelect">changevideosource:</label>
                    <select id="zxingSourceSelect" class="zxing-video-input" @onchange="ChangeVideoInputSource">
                        @foreach (var videoInputDevice in videoInputDevices)
                        {
                            <option value="@videoInputDevice.DeviceId">@videoInputDevice.Label</option>
                        }
                    </select>
                </div>}}

    </section>
    
    @code{
        //private string xtest { get; set; } = "---";

        [Parameter]
        public string Title { get; set; } = "Scan Barcode from Camera";

        [Parameter]
        public bool StartCameraAutomatically { get; set; } = false;

        [Parameter]
        public bool ShowStart { get; set; } = true;

        [Parameter]
        public bool ShowReset { get; set; } = true;

        [Parameter]
        public bool ShowVideoDeviceList { get; set; } = true;

        [Parameter]
        public int VideoWidth { get; set; } = 300;

        [Parameter]
        public int VideoHeigth { get; set; } = 200;

        public string BarcodeText { get; set; }

        List<VideoInputDevice> videoInputDevices;



        protected override async Task OnInitializedAsync()
        {
            videoInputDevices = await JsInteropClass.GetVideoInputDevices(JSRuntime, "get");
            JsInteropClass.BarcodeReceived += ReceivedBarcodeText;
            if (StartCameraAutomatically)
            {
                StartDecoding();
            }
        }

        private async Task GetVideoInputDevicesAsync()
        {
            videoInputDevices = await JsInteropClass.GetVideoInputDevices(JSRuntime, "get");
        }

        private void StartDecoding()
        {
            JsInteropClass.StartDecoding(JSRuntime, "zxingVideo");
        }

        private void StopDecoding()
        {
            JsInteropClass.StopDecoding(JSRuntime);
        }

        private void ReceivedBarcodeText(BarcodeReceivedEventArgs args)
        {
            JsInteropClass.StopDecoding(JSRuntime);

            this.BarcodeText = args.BarcodeText;
            StateHasChanged();
        }

        private void ChangeVideoInputSource(ChangeEventArgs args)
        {
            JsInteropClass.SetVideoInputDevice(JSRuntime, args.Value.ToString());
        }
    }

```

* edit Pages/Index.razor (create a new component in pages) with

```
    @page "/"
    @inject IJSRuntime JSRuntime
    @using System.Text.Json



    <BlazorZXingNpm.Components.BarcodeReader StartCameraAutomatically="true"
                                                                Title="Scanner A"
                                                                VideoWidth="400"
                                                                VideoHeigth="300" />

    <p>result : @LocalBarcodeText</p>




    <div class="row mt-4">
        <div class="col-12">
            <table class="table table-bordered table-hover">
                <thead>
                    <tr>
                        <th>Barcodes/QRCodes</th>
                    </tr>
                </thead>
                <tbody>
                    @if (ResultList.Any())
                    {
                        foreach (var result in ResultList)
                        {
                            <tr>
                                @result
                            </tr>
                        }
                    }
                    else
                    {
                        <tr>
                            <td colspan="5">No records found</td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    </div>


    @code{

        private string LocalBarcodeText;
        private List<string> ResultList = new List<string>();

        protected override async Task OnInitializedAsync()
        {
            await base.OnInitializedAsync();
            BlazorZXingNpm.JsInteropClass.BarcodeReceived += LocalReceivedBarcodeText; // attach to Barcodereceived event
        }

        private async void LocalReceivedBarcodeText(BarcodeReceivedEventArgs args)
        {
            this.LocalBarcodeText = args.BarcodeText;

            ResultList.Insert(0, args.BarcodeText);

            StateHasChanged();
        }

    }
```

