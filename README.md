<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Mantra ID Pro Scanner</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0">
    <script src="https://cdn.jsdelivr.net/npm/@zxing/library@0.18.6/umd/index.min.js"></script>
    <style>
        body { font-family: sans-serif; text-align: center; background: #121212; color: white; margin: 0; padding: 10px; }
        #video-container { display: none; margin: 10px auto; width: 95%; max-width: 400px; border-radius: 15px; overflow: hidden; background: #000; position: relative; border: 2px solid #444; }
        #video { width: 100%; height: auto; }
        
        /* 扫描反馈效果 */
        #result { margin-top: 10px; padding: 30px 10px; border-radius: 15px; background: #1e1e1e; min-height: 140px; display: flex; flex-direction: column; justify-content: center; align-items: center; border: 4px solid #333; }
        #statusText { font-size: 50px; font-weight: bold; margin: 0; }
        #idDisplay { font-size: 20px; margin-top: 10px; color: #888; font-family: monospace; }

        .pass { color: #28a745; border-color: #28a745 !important; background: rgba(40,167,69,0.2) !important; }
        .fail { color: #dc3545; border-color: #dc3545 !important; background: rgba(220,53,69,0.2) !important; }
        
        .cam-btn { background: #007bff; color: white; border: none; padding: 15px; border-radius: 8px; font-size: 16px; width: 95%; cursor: pointer; font-weight: bold; }
        .close-btn { background: #ff4444; color: white; border: none; padding: 10px 20px; border-radius: 5px; margin-top: 10px; }
    </style>
</head>
<body>

    <h3 style="color:#666">Mantra Fast Scan v3.0</h3>
    
    <button id="startBtn" class="cam-btn">📷 开启扫码模式</button>

    <div id="video-container">
        <video id="video" playsinline></video>
        <button class="close-btn" onclick="stopCamera()">关闭相机</button>
    </div>
    
    <div id="result">
        <p id="statusText">READY</p>
        <div id="idDisplay">等待数据...</div>
    </div>

<script>
    const threshold = parseInt("172A5CF1", 16);
    const startBtn = document.getElementById('startBtn');
    const videoContainer = document.getElementById('video-container');
    const statusText = document.getElementById('statusText');
    const idDisplay = document.getElementById('idDisplay');
    const resDiv = document.getElementById('result');

    let codeReader = new ZXing.BrowserMultiFormatReader();
    let isScanning = false;

    // 启动相机并开始循环扫描
    startBtn.onclick = async () => {
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

            videoContainer.style.display = 'block';
            startBtn.style.display = 'none';
            isScanning = true;
            
            // 开始循环扫描逻辑
            scanCycle(selectedDeviceId);
            
        } catch (err) {
            alert("无法打开相机: " + err);
        }
    };

    // 循环扫描核心函数
    async function scanCycle(deviceId) {
        if (!isScanning) return;

        try {
            // 使用 decodeOnce 模式，这样每次识别成功都会立即返回结果
            const result = await codeReader.decodeOnceFromVideoDevice(deviceId, 'video');
            
            if (result) {
                const rawText = result.text.trim();
                updateUI(rawText);
                
                // 扫码成功震动
                if (navigator.vibrate) navigator.vibrate(80);

                // 停顿 1 秒，给操作员反应时间，然后自动开始下一次扫描
                setTimeout(() => {
                    if (isScanning) scanCycle(deviceId);
                }, 1000);
            }
        } catch (err) {
            // 如果没扫到（每秒会触发多次异常），继续尝试
            if (isScanning) {
                setTimeout(() => scanCycle(deviceId), 100);
            }
        }
    }

    // 实时更新判定结果
    function updateUI(valStr) {
        const cleanHex = valStr.replace(/^0x/i, '');
        const currentVal = parseInt(cleanHex, 16);
        
        idDisplay.innerText = "ID: 0x" + cleanHex.toUpperCase();

        if (currentVal < threshold) {
            statusText.innerText = "PASS";
            resDiv.className = "pass";
        } else {
            statusText.innerText = "NG";
            resDiv.className = "fail";
        }
    }

    function stopCamera() {
        isScanning = false;
        codeReader.reset();
        videoContainer.style.display = 'none';
        startBtn.style.display = 'block';
        statusText.innerText = "READY";
        idDisplay.innerText = "等待数据...";
        resDiv.className = "";
    }
</script>
</body>
</html>
