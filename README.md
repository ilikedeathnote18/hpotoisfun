<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 專業攝影評分</title>
    <style>
        /* ------ CSS 美化設定 (與上次相同，新增中文分數樣式) ------ */
        :root {
            --primary: #4f46e5;     /* 主色調：靛藍 */
            --bg: #f8fafc;          /* 背景色 */
            --surface: #ffffff;     /* 卡片背景 */
            --text: #1e293b;        /* 文字色 */
            --grade-bg: #eef2ff;    /* 中文分數背景 */
            --grade-text: #3730a3;  /* 中文分數文字 */
        }

        body {
            font-family: 'Segoe UI', Roboto, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
            margin: 0;
            padding: 20px;
            box-sizing: border-box;
        }

        .app-container {
            background: var(--surface);
            width: 100%;
            max-width: 420px;
            padding: 2rem;
            border-radius: 24px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.08);
            text-align: center;
            transition: all 0.3s ease;
        }

        h1 { margin-top: 0; font-size: 1.5rem; color: #0f172a; }

        /* API Key 輸入框 (測試用) */
        .api-box {
            margin-bottom: 1.5rem;
            background: #eef2ff;
            padding: 10px;
            border-radius: 8px;
        }
        .api-input {
            width: 100%; padding: 8px; border: 1px solid #c7d2fe;
            border-radius: 6px; box-sizing: border-box; font-size: 0.9rem;
        }

        /* 1. 上傳區塊 */
        .upload-zone {
            border: 2px dashed #cbd5e1; border-radius: 16px;
            padding: 3rem 1rem; cursor: pointer; transition: 0.2s;
            background: #fff;
        }
        .upload-icon { font-size: 3rem; margin-bottom: 1rem; display: block; }
        #fileInput { display: none; }

        /* 2. 載入動畫 (與上次相同) */
        .loading-zone { display: none; padding: 2rem 0; }
        .spinner {
            width: 40px; height: 40px; border: 4px solid #e2e8f0;
            border-top-color: var(--primary); border-radius: 50%;
            animation: spin 1s linear infinite; margin: 0 auto 1rem;
        }
        @keyframes spin { to { transform: rotate(360deg); } }
        
        /* 3. 結果展示區 (照片+分數) */
        .result-zone {
            display: none;
            animation: fadeIn 0.5s ease;
        }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

        /* 照片容器 */
        .image-wrapper {
            width: 100%;
            margin-bottom: -40px;
            position: relative;
            z-index: 1;
        }
        .preview-img {
            width: 100%;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
            display: block;
        }

        /* 分數圓圈 + 中文分數容器 */
        .score-wrapper {
            position: relative;
            z-index: 2;
            display: flex;
            flex-direction: column; /* 垂直排列：圓圈在上，中文分數在下 */
            align-items: center;
            margin-bottom: 1rem;
        }
        .score-circle {
            width: 100px; height: 100px; border-radius: 50%;
            background: conic-gradient(var(--primary) 0%, #e2e8f0 0%);
            display: flex; align-items: center; justify-content: center;
            box-shadow: 0 4px 10px rgba(0,0,0,0.2); background-color: white;
        }
        .score-val { font-size: 2.2rem; font-weight: 800; color: var(--primary); line-height: 1; }

        /* 中文分數樣式 */
        .grade-display {
            margin-top: 15px;
            padding: 8px 18px;
            background: var(--grade-bg);
            color: var(--grade-text);
            border-radius: 20px;
            font-size: 1.2rem;
            font-weight: 700;
            letter-spacing: 1px;
            box-shadow: 0 3px 8px rgba(0,0,0,0.05);
        }

        /* 評語 */
        .comment-box {
            background: #f1f5f9; padding: 1rem; border-radius: 12px;
            margin-bottom: 1.5rem; text-align: left; font-size: 0.95rem;
            line-height: 1.6; color: #334155;
        }

        /* 按鈕群 (與上次相同) */
        .btn-group { display: flex; gap: 10px; }
        .btn { flex: 1; padding: 12px; border: none; border-radius: 10px; font-size: 1rem; font-weight: 600; cursor: pointer; transition: 0.2s; }
        .btn-share { background: var(--primary); color: white; }
        .btn-retry { background: #e2e8f0; color: #475569; }
        .btn:active { transform: scale(0.96); }

    </style>
</head>
<body>

    <div class="app-container">
        <h1>📸 AI 攝影評分</h1>

        <div class="api-box" id="apiKeyBox">
            <input type="password" id="apiKeyInput" class="api-input" placeholder="貼上 OpenAI API Key (sk-...)">
        </div>

        <div class="upload-zone" id="uploadZone" onclick="checkKeyAndUpload()">
            <span class="upload-icon">☁️</span>
            <p><strong>點擊上傳照片</strong></p>
            <p style="font-size:0.8rem; color:#94a3b8">支援 JPG, PNG</p>
            <input type="file" id="fileInput" accept="image/*" onchange="processImage(event)">
        </div>

        <div class="loading-zone" id="loadingZone">
            <div class="spinner"></div>
            <p>AI 正在分析構圖與光影...</p>
        </div>

        <div class="result-zone" id="resultZone">
            <div class="image-wrapper">
                <img id="displayImage" class="preview-img" src="" alt="User Photo">
            </div>

            <div class="score-wrapper">
                <div class="score-circle" id="scoreCircleBg">
                    <div class="score-inner">
                        <span class="score-val" id="scoreText">0</span>
                        <span class="score-label">分數</span>
                    </div>
                </div>
                
                <div class="grade-display" id="chineseGradeDisplay">--</div>
            </div>

            <div class="comment-box">
                <strong style="color:var(--primary)">🤖 AI 講評：</strong><br>
                <span id="commentText">評語載入中...</span>
            </div>

            <div class="btn-group">
                <button class="btn btn-share" onclick="shareResult()">📤 分享分數</button>
                <button class="btn btn-retry" onclick="resetApp()">🔄 上訴 (重測)</button>
            </div>
        </div>
    </div>

    <script>
        // ------ JavaScript 邏輯 ------
        
        // ------------------------------------
        // 新增：中文分數轉換邏輯
        // ------------------------------------
        function getChineseGrade(score) {
            // 95以上(甲上)、90-95(甲)、85-90(甲下)、80-85(乙上)75-80(乙)、70-75(乙下)、70以下(丙)
            if (score >= 95) return '甲上';
            if (score >= 90) return '甲';
            if (score >= 85) return '甲下';
            if (score >= 80) return '乙上';
            if (score >= 75) return '乙';
            if (score >= 70) return '乙下';
            return '丙';
        }


        // 1. 檢查 Key 並觸發上傳 (與上次相同)
        function checkKeyAndUpload() {
            const key = document.getElementById('apiKeyInput').value.trim();
            if (!key.startsWith('sk-')) {
                alert('請先在上方輸入有效的 OpenAI API Key');
                return;
            }
            document.getElementById('fileInput').click();
        }

        // 2. 處理圖片與呼叫 API (與上次相同，只調整呼叫 showResult)
        async function processImage(event) {
            const file = event.target.files[0];
            if (!file) return;

            // UI 切換：顯示 Loading
            document.getElementById('uploadZone').style.display = 'none';
            document.getElementById('apiKeyBox').style.display = 'none';
            document.getElementById('loadingZone').style.display = 'block';

            try {
                const base64Image = await toBase64(file);
                document.getElementById('displayImage').src = `data:${file.type};base64,${base64Image}`;

                const apiKey = document.getElementById('apiKeyInput').value.trim();
                const aiResult = await getAIScore(apiKey, base64Image);

                showResult(aiResult); // 呼叫顯示結果

            } catch (error) {
                console.error(error);
                alert('發生錯誤：' + error.message);
                resetApp();
            }
        }

        // 3. 呼叫 OpenAI API (GPT-4o) (與上次相同)
        async function getAIScore(apiKey, base64Image) {
            const response = await fetch("https://api.openai.com/v1/chat/completions", {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    "Authorization": `Bearer ${apiKey}`
                },
                body: JSON.stringify({
                    model: "gpt-4o",
                    messages: [
                        {
                            role: "system",
                            content: `你是一位嚴格但幽默的專業攝影評審。請針對照片的構圖、光線、色彩給出 0-100 的整數評分。
                            並給出一段繁體中文短評(50字內)。
                            必須回傳純 JSON 格式：{"score": number, "comment": "string"}`
                        },
                        {
                            role: "user",
                            content: [
                                { type: "text", text: "請評分這張照片" },
                                { type: "image_url", image_url: { url: `data:image/jpeg;base64,${base64Image}` } }
                            ]
                        }
                    ],
                    response_format: { type: "json_object" },
                    max_tokens: 200
                })
            });

            if (!response.ok) {
                const err = await response.json();
                throw new Error(err.error?.message || 'API 連線失敗');
            }

            const data = await response.json();
            return JSON.parse(data.choices[0].message.content);
        }

        // 4. 顯示結果動畫 (重點更新：加入中文分數)
        function showResult(data) {
            document.getElementById('loadingZone').style.display = 'none';
            document.getElementById('resultZone').style.display = 'block';

            const score = data.score;
            const chineseGrade = getChineseGrade(score); // 計算中文分數

            // 數值分數更新
            document.getElementById('scoreText').innerText = score;
            
            // 中文分數更新
            document.getElementById('chineseGradeDisplay').innerText = chineseGrade;

            // 評語更新
            document.getElementById('commentText').innerText = data.comment;

            // 圓環顏色動畫
            let color = '#ef4444'; // 丙
            if (score >= 70) color = '#f59e0b'; // 乙系列
            if (score >= 85) color = '#4f46e5'; // 甲系列
            
            // 更新樣式
            document.getElementById('scoreText').style.color = color;
            document.getElementById('chineseGradeDisplay').style.background = color + '20'; // 淡色背景
            document.getElementById('chineseGradeDisplay').style.color = color; // 強烈的主色
            
            // 動態畫圓餅圖
            const deg = score * 3.6;
            document.getElementById('scoreCircleBg').style.background = 
                `conic-gradient(${color} ${deg}deg, #e2e8f0 0deg)`;
        }

        // 5. 工具函數：轉 Base64 (與上次相同)
        function toBase64(file) {
            return new Promise((resolve, reject) => {
                const reader = new FileReader();
                reader.readAsDataURL(file);
                reader.onload = () => resolve(reader.result.split(',')[1]);
                reader.onerror = error => reject(error);
            });
        }

        // 6. 分享功能 (與上次相同，加入中文分數)
        async function shareResult() {
            const score = document.getElementById('scoreText').innerText;
            const grade = document.getElementById('chineseGradeDisplay').innerText;
            const comment = document.getElementById('commentText').innerText;
            
            const shareData = {
                title: 'AI 攝影評分',
                text: `我的照片獲得 ${score} 分 (${grade})！AI 說：「${comment}」`,
                url: window.location.href
            };

            if (navigator.share) {
                try { await navigator.share(shareData); } catch (e) {}
            } else {
                navigator.clipboard.writeText(`${shareData.text} ${shareData.url}`);
                alert('結果已複製！快去貼給朋友看！');
            }
        }

        // 7. 重置 (上訴) (與上次相同)
        function resetApp() {
            document.getElementById('resultZone').style.display = 'none';
            document.getElementById('loadingZone').style.display = 'none';
            document.getElementById('uploadZone').style.display = 'block';
            document.getElementById('apiKeyBox').style.display = 'block';
            document.getElementById('fileInput').value = '';
        }
    </script>
</body>
</html>
