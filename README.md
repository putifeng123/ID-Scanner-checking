<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Mantra Rapid Continuous Scanner</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0">
    <script src="https://cdn.jsdelivr.net/npm/@zxing/library@0.18.6/umd/index.min.js"></script>
    <style>
        body { font-family: sans-serif; text-align: center; background: #121212; color: white; margin: 0; padding: 10px; overflow-x: hidden; }
        .container { max-width: 500px; margin: 0 auto; }
        
        /* 预览窗口优化 */
        #video-container { display: none; margin: 10px 0; width: 100%; border-radius: 15px; overflow: hidden; background: #000; position: relative; border: 2px solid #333; line-height: 0; }
        #video { width: 100%; height: auto; }
        
        /* 扫描线动画 */
        .laser { position: absolute; top: 0; left: 0; width: 100%; height: 2px; background: #00ff00; box-shadow: 0 0 10px #00ff00; animation: scan 2s infinite; z-index: 5; }
        @keyframes scan { 0% { top: 0; } 50% { top: 100%; } 100% { top: 0; } }

        /* 结果展示大框 */
        #result { margin-top: 10px; padding: 30px 10px; border-radius: 15px; background: #1e1e1e; min-height: 150px; display: flex; flex-direction: column; justify-content: center; align-items: center; border: 4px solid #333; transition: transform 0.1s; }
        #statusText { font-size: 50px; font-weight: bold; margin: 0; }
        #idDisplay { font-size: 20px; margin-top: 10px; color: #aaa; font-family: monospace; }

        .pass { color: #28a745; border-color: #28a745 !important; background: rgba(40,167,69,0.15) !important; }
        .fail { color: #dc3545; border-color: #dc3545 !important; background: rgba(220,53,69,0.15) !important; }
        
        /* 成功扫描时的震动/闪烁效果 */
        .flash { transform: scale(1.05); }

        .cam-btn { background: #007bff; color: white; border: none; padding: 15px; border-radius: 8px; font-size: 16px; width: 95%; cursor: pointer; font-weight: bold; margin: 10px 0; }
        .close-btn { background: #444; color: #ccc; border: none; padding: 8px 15px; border-radius: 5px; margin-bottom: 10px; }
    </style>
</head>
<body>

<div class="container">
    <h3 style="color:#666; margin: 10px 0;">Mantra Continuous Scanner</h3>
    
    <button id="toggleCam" class="cam-btn">📷 启动连续扫描模式</button>

    <div id="video-container">
        <div class="laser"></div>
        <video id="video" playsinline></video>
        <button class="close-btn" onclick="stopCamera()" style="position:absolute; bottom:10px; right:10px; z-index:10; opacity:0.7;">停止相机</button>
    </div>
    
    <div id="result">
        <p id="statusText">READY</p>
        <div id="idDisplay">等待数据...</div>
    </div>

    <input type="text" id="inputBox" style="opacity:0; position:absolute; z-index:-1;">
</div>

<script>
    const threshold = parseInt("172A5CF1", 16);
    const resDiv = document.getElementById('result');
    const statusText = document.getElementById('statusText');
    const idDisplay = document.getElementById('idDisplay');
    const toggleCamBtn = document.getElementById('toggleCam');
    const videoContainer = document.getElementById('video-container');

    let codeReader = new ZXing.BrowserMultiFormatReader();
    let isScanning = false;
    let lastResult = "";
    let isPaused = false; // 用于连续扫描的冷却时间

    function processValue(valStr) {
        if (!valStr || isPaused) return;
        
        // 如果和上次结果一样，且在冷却期内，则跳过
        if (valStr === lastResult) return;
        lastResult = valStr;

        // 触发视觉闪烁反馈
        resDiv.classList.add('flash');
        setTimeout(() => resDiv.classList.remove('flash'), 100);

        const cleanHex = valStr.replace(/^0x/i, '').trim();
        const currentVal = parseInt(cleanHex, 16);
        
        idDisplay.innerText = "ID: 0x" + cleanHex.toUpperCase();

        if (currentVal < threshold) {
            statusText.innerText = "PASS";
            resDiv.className = "pass";
        } else {
            statusText.innerText = "NG";
            resDiv.className = "fail";
        }

        // 开启 1.5 秒冷却，防止同一条码重复触发
        isPaused = true;
        setTimeout(() => { 
            isPaused = false; 
            lastResult = ""; // 清空记录，允许扫描下一个
        }, 1500); 

        // 手机震动
        if (navigator.vibrate) navigator.vibrate(100);
    }

    toggleCamBtn.addEventListener('click', () => {
        if (!isScanning) startCamera(); else stopCamera();
    });

    async function startCamera() {
        videoContainer.style.display = 'block';
        toggleCamBtn.innerText = "正在扫描中...";
        toggleCamBtn.style.background = "#444";
        isScanning = true;

        try {
            const videoDevices = await codeReader.listVideoInputDevices();
            let selectedDeviceId = videoDevices[0].deviceId;
            
            // 自动寻找后置摄像头
            for (const device of videoDevices) {
                if (device.label.toLowerCase().includes('back') || device.label.toLowerCase().includes('rear')) {
                    selectedDeviceId = device.deviceId;
                    break;
                }
            }

            // 使用连续解码模式
            codeReader.decodeFromVideoDevice(selectedDeviceId, 'video', (result, err) => {
                if (result) {
                    processValue(result.text);
                }
            });
        } catch (err) {
            alert("无法访问摄像头: " + err);
            stopCamera();
        }
    }

    function stopCamera() {
        codeReader.reset();
        videoContainer.style.display = 'none';
        toggleCamBtn.innerText = "📷 启动连续扫描模式";
        toggleCamBtn.style.background = "#007bff";
        isScanning = false;
        lastResult = "";
    }

    // 同时也支持扫码枪输入
    const box = document.getElementById('inputBox');
    box.addEventListener('keyup', (e) => {
        if (e.key === 'Enter') {
            processValue(box.value.trim());
            box.value = "";
        }
    });
    setInterval(() => { if(!isScanning) box.focus(); }, 1000);
</script>
</body>
</html>
