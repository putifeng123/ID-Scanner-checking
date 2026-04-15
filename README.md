<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Mantra ID Screening</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0">
    
    <script src="https://cdn.jsdelivr.net/npm/@zxing/library@0.18.6/umd/index.min.js"></script>

    <style>
        body { font-family: sans-serif; text-align: center; background: #121212; color: white; margin: 0; padding: 15px; }
        .container { max-width: 500px; margin: 0 auto; }
        #inputBox { padding: 15px; width: 85%; font-size: 20px; border-radius: 8px; border: 2px solid #333; background: #1e1e1e; color: white; margin: 15px 0; text-align: center; }
        #video-container { display: none; margin: 15px 0; width: 100%; border-radius: 12px; overflow: hidden; background: #000; position: relative; border: 2px solid #444; }
        #video { width: 100%; height: auto; display: block; }
        .close-cam { position: absolute; top: 10px; right: 10px; background: rgba(255,0,0,0.8); color: white; border: none; padding: 8px 12px; border-radius: 5px; }
        #result { margin-top: 15px; padding: 25px 10px; border-radius: 15px; background: #1e1e1e; min-height: 120px; display: flex; flex-direction: column; justify-content: center; align-items: center; border: 4px solid #333; }
        #statusText { font-size: 42px; font-weight: bold; margin: 0; }
        #idDisplay { font-size: 18px; margin-top: 10px; color: #888; }
        .pass { color: #28a745; border-color: #28a745 !important; background: rgba(40,167,69,0.1) !important; }
        .fail { color: #dc3545; border-color: #dc3545 !important; background: rgba(220,53,69,0.1) !important; }
        .cam-btn { background: #007bff; color: white; border: none; padding: 15px; border-radius: 8px; font-size: 16px; width: 90%; cursor: pointer; font-weight: bold; }
    </style>
</head>
<body>

<div class="container">
    <h3 style="color:#888">Mantra Screening (India Network Fix)</h3>
    
    <input type="text" id="inputBox" placeholder="Scan or Type ID" autofocus autocomplete="off">
    
    <button id="toggleCam" class="cam-btn">📷 Open Camera</button>

    <div id="video-container">
        <video id="video" playsinline></video>
        <button class="close-cam" onclick="stopCamera()">Close</button>
    </div>
    
    <div id="result">
        <p id="statusText">READY</p>
        <div id="idDisplay">Waiting...</div>
    </div>
</div>

<script>
    const threshold = parseInt("172A5CF1", 16);
    const box = document.getElementById('inputBox');
    const resDiv = document.getElementById('result');
    const statusText = document.getElementById('statusText');
    const idDisplay = document.getElementById('idDisplay');
    const toggleCamBtn = document.getElementById('toggleCam');
    const videoContainer = document.getElementById('video-container');

    let codeReader = null;
    let isScanning = false;

    // 检查库是否加载成功
    function isLibLoaded() {
        return typeof ZXing !== 'undefined';
    }

    function processValue(valStr) {
        if (!valStr) return;
        const displayVal = valStr.toUpperCase();
        try {
            const val = parseInt(valStr.replace(/^0x/i, ''), 16);
            idDisplay.innerText = "ID: 0x" + displayVal.replace(/^0X/i, '');
            if (val < threshold) {
                statusText.innerText = "PASS";
                resDiv.className = "pass";
            } else {
                statusText.innerText = "FAIL";
                resDiv.className = "fail";
            }
        } catch(err) {
            statusText.innerText = "ERROR";
        }
        box.value = ""; 
    }

    box.addEventListener('keyup', (e) => {
        if (e.key === 'Enter') processValue(box.value.trim());
    });

    toggleCamBtn.addEventListener('click', () => {
        if (!isLibLoaded()) {
            alert("Network Error: Cannot load scan components. Please try a different WiFi or mobile data (Airtel/Jio).");
            return;
        }
        if (!isScanning) startCamera(); else stopCamera();
    });

    async function startCamera() {
        if (!codeReader) codeReader = new ZXing.BrowserMultiFormatReader();
        videoContainer.style.display = 'block';
        toggleCamBtn.innerText = "⌛ Starting...";
        isScanning = true;

        try {
            const videoDevices = await codeReader.listVideoInputDevices();
            let selectedDeviceId = videoDevices[0].deviceId;
            
            // 自动匹配后置摄像头
            for (const device of videoDevices) {
                if (device.label.toLowerCase().includes('back') || device.label.toLowerCase().includes('rear')) {
                    selectedDeviceId = device.deviceId;
                    break;
                }
            }

            codeReader.decodeFromVideoDevice(selectedDeviceId, 'video', (result, err) => {
                if (result) {
                    processValue(result.text);
                    if (navigator.vibrate) navigator.vibrate(100);
                }
            });
            toggleCamBtn.innerText = "📷 Scanning... (Tap to Close)";
        } catch (err) {
            alert("Camera access denied.");
            stopCamera();
        }
    }

    function stopCamera() {
        if (codeReader) codeReader.reset();
        videoContainer.style.display = 'none';
        toggleCamBtn.innerText = "📷 Open Camera";
        isScanning = false;
        box.focus();
    }

    setInterval(() => { if(!isScanning) box.focus(); }, 1500);
</script>
</body>
</html>
