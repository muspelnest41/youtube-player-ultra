<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- 核心修正：允許傳遞 Referrer，這對版權影片驗證至關重要 -->
    <meta name="referrer" content="always">
    <title>YouTube 播放小幫手 Ultra</title>
    
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <script src="https://cdn.jsdelivr.net/npm/sortablejs@latest/Sortable.min.js"></script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body { background-color: #020617; color: #f1f5f9; font-family: 'Inter', sans-serif; height: 100vh; overflow: hidden; }
        .glass { background: rgba(30, 41, 59, 0.7); backdrop-filter: blur(12px); border: 1px solid rgba(255,255,255,0.1); }
        .playlist-item { transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1); }
        .playlist-item.active { background: linear-gradient(90deg, rgba(239, 68, 68, 0.2) 0%, rgba(30, 41, 59, 0.5) 100%); border-left: 4px solid #ef4444; }
        .sortable-ghost { opacity: 0.2; background: #ef4444; }
        ::-webkit-scrollbar { width: 5px; }
        ::-webkit-scrollbar-thumb { background: #334155; border-radius: 10px; }
        /* 播放器容器 */
        .video-wrapper { position: relative; width: 100%; height: 0; padding-bottom: 56.25%; background: #000; border-radius: 1rem; overflow: hidden; box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5); }
        #player { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
    </style>
</head>
<body class="flex flex-col">

    <!-- A. 頂部輸入區 -->
    <header class="p-4 border-b border-slate-800 bg-slate-900/80 flex-none z-10">
        <div class="max-w-6xl mx-auto flex flex-col md:flex-row gap-3">
            <div class="flex-grow flex items-center bg-black/40 rounded-xl px-4 border border-slate-700 focus-within:border-red-500 transition-all shadow-inner">
                <i class="fab fa-youtube text-red-500 mr-3"></i>
                <input type="text" id="urlInput" placeholder="請輸入 YouTube 網址 (支援帶參數的長短網址)..." 
                       class="w-full bg-transparent py-3 text-sm focus:outline-none">
                <button id="pasteBtn" class="p-2 text-slate-400 hover:text-white transition-colors" title="貼上剪貼簿">
                    <i class="fas fa-paste"></i>
                </button>
            </div>
            <button id="loadBtn" class="bg-red-600 hover:bg-red-500 text-white font-bold px-8 py-3 rounded-xl transition-all active:scale-95 flex items-center justify-center gap-2">
                <i class="fas fa-plus"></i> 載入影片
            </button>
        </div>
    </header>

    <!-- 主內容區 -->
    <main class="flex-grow flex flex-col md:flex-row overflow-hidden p-4 gap-4">
        
        <!-- B. 左側：播放區域 -->
        <section class="flex-[3] flex flex-col justify-center">
            <div class="video-wrapper">
                <div id="player"></div>
                <div id="emptyOverlay" class="absolute inset-0 flex flex-col items-center justify-center text-slate-500 bg-slate-950 pointer-events-none">
                    <i class="fas fa-play-circle text-6xl mb-4 opacity-20"></i>
                    <p class="text-lg">請在右側加入影片</p>
                </div>
            </div>
            <div id="errorAlert" class="mt-4 p-4 bg-red-900/30 border border-red-500/50 rounded-xl text-red-200 text-sm hidden flex items-center gap-3">
                <i class="fas fa-exclamation-triangle text-lg"></i>
                <div>
                    <p class="font-bold">播放受限</p>
                    <p id="errorMsg">此影片可能禁止嵌入。若此問題頻繁出現，請嘗試使用線上伺服器開啟此頁面。</p>
                </div>
            </div>
        </section>

        <!-- C. 右側：清單區域 -->
        <section class="flex-grow md:w-96 glass rounded-2xl flex flex-col overflow-hidden">
            <div class="p-4 border-b border-slate-800 flex justify-between items-center bg-slate-800/30">
                <div class="flex items-center gap-2">
                    <i class="fas fa-stream text-red-500"></i>
                    <span class="font-bold">待播清單</span>
                </div>
                <span id="videoCount" class="text-[10px] px-2 py-1 bg-slate-700 rounded-full text-slate-300 font-mono">0</span>
            </div>
            
            <div id="playlist" class="flex-grow overflow-y-auto p-2 space-y-2">
                <!-- 影片項目 -->
            </div>

            <div class="p-4 border-t border-slate-800 text-[10px] text-slate-500 text-center">
                <i class="fas fa-info-circle mr-1"></i> 版權 MV 若出現「請到 YouTube 觀看」屬官方硬性限制
            </div>
        </section>
    </main>

    <div id="toast" class="fixed bottom-6 right-6 p-4 rounded-xl glass border-slate-700 text-sm opacity-0 translate-y-10 transition-all duration-300 shadow-2xl z-50"></div>

    <script>
        let player;
        let playlistData = [];
        let currentIndex = -1;
        let currentRepeatCount = 0;
        let isPlayerReady = false;

        const tag = document.createElement('script');
        tag.src = "https://www.youtube.com/iframe_api";
        const firstScriptTag = document.getElementsByTagName('script')[0];
        firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);

        function onYouTubeIframeAPIReady() {
            // 使用正確的 Origin 以最大化版權影片相容性
            let origin = window.location.origin;
            if (origin === "null" || !origin) origin = "http://localhost"; 

            player = new YT.Player('player', {
                height: '100%',
                width: '100%',
                // 修正：使用標準 youtube.com 而非 nocookie
                host: 'https://www.youtube.com',
                playerVars: {
                    'autoplay': 1,
                    'controls': 1,
                    'rel': 0,
                    'enablejsapi': 1,
                    'origin': origin,
                    'widget_referrer': origin,
                    'modestbranding': 1
                },
                events: {
                    'onReady': () => { isPlayerReady = true; },
                    'onStateChange': handleStateChange,
                    'onError': handleError
                }
            });
        }

        function handleStateChange(event) {
            if (event.data === YT.PlayerState.ENDED) {
                const video = playlistData[currentIndex];
                currentRepeatCount++;

                const needsRepeat = (video.repeat === 'infinity') || (currentRepeatCount < video.repeat);

                if (needsRepeat) {
                    player.playVideo();
                } else {
                    nextVideo();
                }
            }
        }

        function nextVideo() {
            if (playlistData.length === 0) return;

            const currentVideo = playlistData[currentIndex];
            const wasOneTime = currentVideo.oneTime;

            // 移除一次性影片
            if (wasOneTime) {
                playlistData.splice(currentIndex, 1);
                // 移除後索引不需要增加，因為後面的補上來了
            } else {
                currentIndex++;
            }

            // 隨機邏輯
            if (currentVideo.random && playlistData.length > 1) {
                currentIndex = Math.floor(Math.random() * playlistData.length);
            }

            // 循環檢查
            if (currentIndex >= playlistData.length) {
                currentIndex = 0;
            }

            if (playlistData.length > 0) {
                playVideo(currentIndex);
            } else {
                stopEverything();
            }
        }

        function handleError(e) {
            const errorArea = document.getElementById('errorAlert');
            const errorMsg = document.getElementById('errorMsg');
            errorArea.classList.remove('hidden');

            let txt = "發生未知錯誤 (代碼 " + e.data + ")";
            if (e.data === 101 || e.data === 150) txt = "版權擁有者禁止在外部嵌入播放。這類官方影片無法透過本工具播放，請跳過。";
            if (e.data === 153) txt = "YouTube 安全驗證失敗。建議您將此 HTML 檔案上傳到伺服器 (如 GitHub Pages) 或使用 Live Server 開啟。";
            
            errorMsg.innerText = txt;
            
            // 自動跳下一首
            setTimeout(() => {
                errorArea.classList.add('hidden');
                nextVideo();
            }, 4000);
        }

        function extractId(url) {
            const regExp = /^.*(youtu.be\/|v\/|u\/\w\/|embed\/|watch\?v=|\&v=)([^#\&\?]*).*/;
            const match = url.match(regExp);
            return (match && match[2].length === 11) ? match[2] : null;
        }

        async function addVideo() {
            const input = document.getElementById('urlInput');
            const id = extractId(input.value.trim());

            if (!id) {
                showToast("⚠️ 無效的網址", "bg-red-500");
                return;
            }

            showToast("🔍 抓取影片資訊...", "bg-blue-600");
            
            let title = "未命名影片";
            try {
                const res = await fetch(`https://noembed.com/embed?url=https://www.youtube.com/watch?v=${id}`);
                const data = await res.json();
                title = data.title || "未知標題";
            } catch(e) {}

            playlistData.push({
                id: id,
                title: title,
                repeat: 1,
                oneTime: true,
                random: false
            });

            input.value = "";
            renderPlaylist();
            
            if (currentIndex === -1) playVideo(0);
        }

        function playVideo(index) {
            if (!isPlayerReady || index >= playlistData.length) return;
            
            currentIndex = index;
            currentRepeatCount = 0;
            document.getElementById('emptyOverlay').style.display = 'none';
            document.getElementById('errorAlert').classList.add('hidden');
            
            player.loadVideoById(playlistData[currentIndex].id);
            renderPlaylist();
        }

        function stopEverything() {
            currentIndex = -1;
            player.stopVideo();
            document.getElementById('emptyOverlay').style.display = 'flex';
            renderPlaylist();
        }

        function renderPlaylist() {
            const container = document.getElementById('playlist');
            container.innerHTML = "";
            document.getElementById('videoCount').innerText = playlistData.length;

            playlistData.forEach((item, i) => {
                const div = document.createElement('div');
                div.className = `playlist-item p-3 rounded-xl flex items-center gap-3 border border-slate-800 shadow-sm ${i === currentIndex ? 'active' : 'bg-slate-900/40'}`;
                
                div.innerHTML = `
                    <div class="drag-handle cursor-grab text-slate-600 hover:text-slate-400">
                        <i class="fas fa-ellipsis-v"></i>
                    </div>
                    <div class="flex-grow min-w-0 cursor-pointer" onclick="playVideo(${i})">
                        <p class="text-[13px] font-semibold truncate ${i === currentIndex ? 'text-white' : 'text-slate-400'}">${item.title}</p>
                    </div>
                    <div class="flex items-center gap-1">
                        <button onclick="toggleAttr(${i}, 'repeat')" class="w-7 h-7 rounded-lg text-[10px] font-bold border border-slate-700 hover:bg-slate-700 transition-colors ${item.repeat !== 1 ? 'text-yellow-400 border-yellow-500/50' : 'text-slate-500'}">
                            ${item.repeat === 'infinity' ? '∞' : item.repeat}
                        </button>
                        <button onclick="toggleAttr(${i}, 'oneTime')" class="w-7 h-7 rounded-lg text-xs border border-slate-700 hover:bg-slate-700 transition-colors ${item.oneTime ? 'text-blue-400 border-blue-500/50' : 'text-slate-500'}" title="一次性播放">
                            <i class="fas fa-ghost"></i>
                        </button>
                        <button onclick="toggleAttr(${i}, 'random')" class="w-7 h-7 rounded-lg text-xs border border-slate-700 hover:bg-slate-700 transition-colors ${item.random ? 'text-green-400 border-green-500/50' : 'text-slate-500'}" title="隨機下首">
                            <i class="fas fa-random"></i>
                        </button>
                        <button onclick="removeVideo(${i})" class="w-7 h-7 rounded-lg text-xs text-slate-600 hover:text-red-500 hover:bg-red-500/10">
                            <i class="fas fa-trash"></i>
                        </button>
                    </div>
                `;
                container.appendChild(div);
            });
            initSortable();
        }

        function toggleAttr(index, attr) {
            if (attr === 'repeat') {
                const steps = [1, 2, 3, 'infinity'];
                const curIdx = steps.indexOf(playlistData[index].repeat);
                playlistData[index].repeat = steps[(curIdx + 1) % steps.length];
            } else {
                playlistData[index][attr] = !playlistData[index][attr];
            }
            renderPlaylist();
        }

        function removeVideo(index) {
            playlistData.splice(index, 1);
            if (index === currentIndex) {
                if (playlistData.length > 0) nextVideo();
                else stopEverything();
            } else if (index < currentIndex) {
                currentIndex--;
            }
            renderPlaylist();
        }

        function initSortable() {
            new Sortable(document.getElementById('playlist'), {
                handle: '.drag-handle',
                animation: 250,
                ghostClass: 'sortable-ghost',
                onEnd: (evt) => {
                    const item = playlistData.splice(evt.oldIndex, 1)[0];
                    playlistData.splice(evt.newIndex, 0, item);
                    if (currentIndex === evt.oldIndex) currentIndex = evt.newIndex;
                    else if (currentIndex > evt.oldIndex && currentIndex <= evt.newIndex) currentIndex--;
                    else if (currentIndex < evt.oldIndex && currentIndex >= evt.newIndex) currentIndex++;
                    renderPlaylist();
                }
            });
        }

        function showToast(msg, bgColor) {
            const t = document.getElementById('toast');
            t.innerText = msg;
            t.className = `fixed bottom-6 right-6 p-4 rounded-xl text-white text-sm opacity-100 translate-y-0 transition-all shadow-2xl z-50 ${bgColor}`;
            setTimeout(() => {
                t.classList.add('opacity-0', 'translate-y-10');
            }, 3000);
        }

        document.getElementById('loadBtn').onclick = addVideo;
        document.getElementById('urlInput').onkeydown = (e) => { if(e.key==='Enter') addVideo(); };
        document.getElementById('pasteBtn').onclick = async () => {
            try {
                const text = await navigator.clipboard.readText();
                document.getElementById('urlInput').value = text;
                showToast("📋 已從剪貼簿貼上", "bg-slate-700");
            } catch (err) {
                showToast("❌ 權限不足，請手動貼上", "bg-red-600");
            }
        };
    </script>
</body>
</html>
