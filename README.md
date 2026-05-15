<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="referrer" content="strict-origin-when-cross-origin">
    <title>YouTube Player Ultra - 播放輔助</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Sortable/1.15.0/Sortable.min.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        body {
            background-color: #0f172a;
            color: #f1f5f9;
        }
        .video-container {
            position: relative;
            padding-bottom: 56.25%; /* 16:9 Aspect Ratio */
            height: 0;
            overflow: hidden;
            border-radius: 0.75rem;
            background: #000;
        }
        .video-container iframe, .video-placeholder {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
        }
        .custom-scrollbar::-webkit-scrollbar {
            width: 6px;
        }
        .custom-scrollbar::-webkit-scrollbar-track {
            background: #1e293b;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb {
            background: #475569;
            border-radius: 10px;
        }
        .ghost-card {
            opacity: 0.5;
            background: #334155 !important;
        }
        .btn-active { color: #38bdf8; }
        .btn-inactive { color: #64748b; }
        .infinite-badge { font-size: 1.2rem; line-height: 1; }
    </style>
</head>
<body class="min-h-screen p-4 md:p-8">

    <div class="max-w-7xl mx-auto">
        <!-- A. Header & Input Section -->
        <header class="mb-8 border-b border-slate-700 pb-6">
            <h1 class="text-3xl font-bold mb-6 flex items-center gap-3 text-white">
                <i class="fab fa-youtube text-red-500 text-4xl"></i>
                YouTube Player Ultra
            </h1>
            
            <div class="flex flex-col md:flex-row gap-4 bg-slate-800/50 p-4 rounded-xl border border-slate-700 shadow-xl">
                <div class="relative flex-grow">
                    <span class="absolute inset-y-0 left-0 pl-3 flex items-center text-slate-400">
                        <i class="fab fa-youtube"></i>
                    </span>
                    <input type="text" id="videoInput" 
                        class="block w-full pl-10 pr-12 py-3 bg-slate-900 border border-slate-700 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition-all text-white placeholder-slate-500" 
                        placeholder="請貼上影片網址...">
                    <button id="pasteBtn" title="從剪貼簿貼上"
                        class="absolute inset-y-0 right-0 pr-3 flex items-center text-slate-400 hover:text-white transition-colors">
                        <i class="far fa-paste text-xl"></i>
                    </button>
                </div>
                <button id="loadBtn" 
                    class="bg-red-600 hover:bg-red-700 text-white font-bold py-3 px-8 rounded-lg transition-all flex items-center justify-center gap-2 shadow-lg active:scale-95">
                    <i class="fas fa-plus"></i> 載入影片
                </button>
            </div>
        </header>

        <div class="grid grid-cols-1 lg:grid-cols-12 gap-8">
            <!-- B. Left: Player Section -->
            <div class="lg:col-span-7 xl:col-span-8">
                <div class="sticky top-8">
                    <div class="video-container shadow-2xl border border-slate-700">
                        <div id="player"></div>
                        <div id="placeholder" class="video-placeholder flex flex-col items-center justify-center text-slate-500 gap-4">
                            <i class="fab fa-youtube text-6xl opacity-20"></i>
                            <p class="text-lg font-medium opacity-50">等待播放清單載入...</p>
                        </div>
                    </div>
                    
                    <div id="errorDisplay" class="mt-4 p-4 bg-red-900/30 border border-red-500/50 rounded-lg text-red-200 hidden">
                        <i class="fas fa-exclamation-circle mr-2"></i>
                        <span id="errorMsg"></span>
                    </div>

                    <div id="currentInfo" class="mt-6 p-6 bg-slate-800/30 rounded-xl border border-slate-700 hidden">
                        <span class="text-blue-400 text-sm font-bold uppercase tracking-wider">正在播放</span>
                        <h2 id="currentTitle" class="text-xl font-semibold mt-1 text-white"></h2>
                    </div>
                </div>
            </div>

            <!-- C. Right: Playlist Section -->
            <div class="lg:col-span-5 xl:col-span-4">
                <div class="bg-slate-800 rounded-2xl border border-slate-700 shadow-xl overflow-hidden flex flex-col max-h-[80vh]">
                    <div class="p-4 border-b border-slate-700 bg-slate-800/80 backdrop-blur flex justify-between items-center">
                        <h3 class="font-bold flex items-center gap-2">
                            <i class="fas fa-list-ul text-blue-400"></i> 待播清單
                        </h3>
                        <span id="playlistCount" class="bg-slate-700 text-xs px-2 py-1 rounded-full text-slate-300">0</span>
                    </div>
                    
                    <div id="playlist" class="flex-grow overflow-y-auto p-4 space-y-3 custom-scrollbar min-h-[300px]">
                        <!-- Video items will appear here -->
                    </div>

                    <div class="p-4 bg-slate-900/50 border-t border-slate-700 text-center">
                        <p class="text-xs text-slate-500">
                            <i class="fas fa-info-circle mr-1"></i> 拖曳影片左側可調整播放順序
                        </p>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <!-- UI Templates (Invisible) -->
    <template id="video-item-template">
        <div class="video-item group flex items-center gap-3 p-3 bg-slate-900 border border-slate-700 rounded-xl transition-all hover:border-slate-500 hover:bg-slate-800/80">
            <div class="handle cursor-grab active:cursor-grabbing text-slate-600 hover:text-slate-400 p-1">
                <i class="fas fa-grip-vertical"></i>
            </div>
            
            <div class="flex-grow min-w-0">
                <h4 class="title text-sm font-medium truncate text-slate-200">影片載入中...</h4>
            </div>

            <div class="flex items-center gap-2">
                <!-- Button 1: Repeat Counter -->
                <button class="repeat-btn w-8 h-8 flex items-center justify-center bg-slate-700 hover:bg-slate-600 rounded text-xs transition-colors" title="重複次數">
                    <span class="count-num">1</span>
                </button>

                <!-- Button 4: One-time (Ghost) -->
                <button class="once-btn w-8 h-8 flex items-center justify-center bg-slate-700 hover:bg-slate-600 rounded text-xs transition-colors" title="一次性播放 (結束後移除)">
                    <i class="fas fa-ghost"></i>
                </button>

                <!-- Button 3: Random -->
                <button class="random-btn w-8 h-8 flex items-center justify-center bg-slate-700 hover:bg-slate-600 rounded text-xs transition-colors" title="播放後隨機跳轉">
                    <i class="fas fa-random"></i>
                </button>

                <!-- Button 2: Remove -->
                <button class="delete-btn w-8 h-8 flex items-center justify-center text-slate-500 hover:text-red-400 hover:bg-red-400/10 rounded transition-colors" title="移除影片">
                    <i class="fas fa-trash-alt"></i>
                </button>
            </div>
        </div>
    </template>

    <script>
        // --- State Management ---
        let playlist = [];
        let player = null;
        let currentIndex = -1;
        let currentRepeatCount = 0;
        let isPlayerReady = false;

        // --- YouTube API Setup ---
        const tag = document.createElement('script');
        tag.src = "https://www.youtube.com/iframe_api";
        const firstScriptTag = document.getElementsByTagName('script')[0];
        firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);

        function onYouTubeIframeAPIReady() {
            player = new YT.Player('player', {
                height: '100%',
                width: '100%',
                playerVars: {
                    'autoplay': 0,
                    'controls': 1,
                    'modestbranding': 1,
                    'rel': 0,
                    'origin': window.location.origin
                },
                events: {
                    'onReady': onPlayerReady,
                    'onStateChange': onPlayerStateChange,
                    'onError': onPlayerError
                }
            });
        }

        function onPlayerReady(event) {
            isPlayerReady = true;
            document.getElementById('placeholder').style.display = 'flex';
        }

        function onPlayerStateChange(event) {
            // YT.PlayerState.ENDED = 0
            if (event.data === YT.PlayerState.ENDED) {
                handleVideoEnd();
            }
        }

        function onPlayerError(event) {
            let msg = "影片加載出錯。";
            if (event.data === 101 || event.data === 150) {
                msg = "版權方禁止在此環境嵌入。自動跳至下一首。";
            }
            showError(msg);
            setTimeout(playNext, 3000);
        }

        // --- Core Logic ---
        function handleVideoEnd() {
            if (currentIndex === -1) return;
            const video = playlist[currentIndex];
            currentRepeatCount++;

            // Handle Repeats
            const targetRepeats = video.repeats === '∞' ? Infinity : parseInt(video.repeats);
            
            if (currentRepeatCount < targetRepeats) {
                player.playVideo();
                return;
            }

            // End of its turn
            currentRepeatCount = 0;

            // Handle One-time (Remove after play)
            if (video.once) {
                const idToRemove = video.id;
                playNext(true); // Special next that considers the current one is being deleted
                removeFromPlaylist(idToRemove);
            } else {
                playNext();
            }
        }

        function playNext(wasDeleted = false) {
            if (playlist.length === 0) {
                currentIndex = -1;
                updateUI();
                return;
            }

            const currentVideo = playlist[currentIndex];
            
            if (currentVideo && currentVideo.random) {
                // Pick random index different from current if possible
                if (playlist.length > 1) {
                    let nextIdx = currentIndex;
                    while(nextIdx === currentIndex) {
                        nextIdx = Math.floor(Math.random() * playlist.length);
                    }
                    currentIndex = nextIdx;
                }
            } else {
                // If the current video was just deleted, the "next" is actually at the same index
                if (!wasDeleted) {
                    currentIndex = (currentIndex + 1) % playlist.length;
                } else {
                    if (currentIndex >= playlist.length) currentIndex = 0;
                }
            }

            loadVideo(currentIndex);
        }

        function loadVideo(index) {
            if (index < 0 || index >= playlist.length) return;
            currentIndex = index;
            currentRepeatCount = 0;
            const video = playlist[currentIndex];

            player.loadVideoById(video.youtubeId);
            document.getElementById('placeholder').style.display = 'none';
            document.getElementById('currentInfo').classList.remove('hidden');
            document.getElementById('currentTitle').innerText = video.title;
            document.getElementById('errorDisplay').classList.add('hidden');
            
            updateUI();
        }

        // --- Helper Functions ---
        function extractVideoID(url) {
            const regExp = /^.*(youtu\.be\/|v\/|u\/\w\/|embed\/|watch\?v=|\&v=)([^#\&\?]*).*/;
            const match = url.match(regExp);
            return (match && match[2].length === 11) ? match[2] : null;
        }

        async function fetchVideoTitle(videoId) {
            // Simplified title fetch (could use YouTube Data API with key for real titles)
            // Here we just return a placeholder or use the ID if we don't have a backend
            return `影片 ID: ${videoId}`;
        }

        function addToPlaylist(ytId) {
            const newItem = {
                id: Date.now().toString(),
                youtubeId: ytId,
                title: `載入中: ${ytId}`,
                repeats: 1, // 1, 2, 3, ∞
                once: true,
                random: false
            };

            playlist.push(newItem);
            renderPlaylist();
            updatePlaylistCount();

            // Try to auto-play if first video
            if (currentIndex === -1) {
                loadVideo(0);
            }

            // Attempt to get OEmbed title for better UI
            fetch(`https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=${ytId}&format=json`)
                .then(res => res.json())
                .then(data => {
                    newItem.title = data.title;
                    renderPlaylist();
                })
                .catch(() => {
                    newItem.title = `YouTube 影片 (${ytId})`;
                    renderPlaylist();
                });
        }

        function removeFromPlaylist(id) {
            const idx = playlist.findIndex(v => v.id === id);
            if (idx === -1) return;

            const isCurrent = (idx === currentIndex);
            playlist.splice(idx, 1);

            if (isCurrent) {
                if (playlist.length > 0) {
                    playNext(true);
                } else {
                    player.stopVideo();
                    currentIndex = -1;
                    document.getElementById('currentInfo').classList.add('hidden');
                    document.getElementById('placeholder').style.display = 'flex';
                }
            } else if (idx < currentIndex) {
                currentIndex--;
            }

            renderPlaylist();
            updatePlaylistCount();
        }

        function renderPlaylist() {
            const container = document.getElementById('playlist');
            container.innerHTML = '';
            const template = document.getElementById('video-item-template');

            playlist.forEach((video, index) => {
                const clone = template.content.cloneNode(true);
                const root = clone.querySelector('.video-item');
                
                // Active state
                if (index === currentIndex) {
                    root.classList.add('ring-2', 'ring-blue-500', 'bg-blue-900/20');
                }

                root.querySelector('.title').innerText = video.title;
                root.querySelector('.title').onclick = () => loadVideo(index);
                root.querySelector('.title').classList.add('cursor-pointer');

                // Buttons logic
                const repBtn = root.querySelector('.repeat-btn');
                const repNum = repBtn.querySelector('.count-num');
                repNum.innerText = video.repeats;
                if (video.repeats !== 1) repBtn.classList.add('text-blue-400');
                repBtn.onclick = () => {
                    if (video.repeats === 1) video.repeats = 2;
                    else if (video.repeats === 2) video.repeats = 3;
                    else if (video.repeats === 3) video.repeats = '∞';
                    else video.repeats = 1;
                    renderPlaylist();
                };

                const onceBtn = root.querySelector('.once-btn');
                if (video.once) onceBtn.classList.add('text-blue-400');
                else onceBtn.classList.add('text-slate-500');
                onceBtn.onclick = () => {
                    video.once = !video.once;
                    renderPlaylist();
                };

                const randBtn = root.querySelector('.random-btn');
                if (video.random) randBtn.classList.add('text-green-400');
                else randBtn.classList.add('text-slate-500');
                randBtn.onclick = () => {
                    video.random = !video.random;
                    renderPlaylist();
                };

                root.querySelector('.delete-btn').onclick = () => removeFromPlaylist(video.id);

                container.appendChild(clone);
            });
        }

        function updatePlaylistCount() {
            document.getElementById('playlistCount').innerText = playlist.length;
        }

        function updateUI() {
            renderPlaylist();
        }

        function showError(msg) {
            const err = document.getElementById('errorDisplay');
            const errMsg = document.getElementById('errorMsg');
            err.classList.remove('hidden');
            errMsg.innerText = msg;
            setTimeout(() => err.classList.add('hidden'), 5000);
        }

        // --- Event Listeners ---
        document.getElementById('loadBtn').onclick = () => {
            const input = document.getElementById('videoInput');
            const id = extractVideoID(input.value.trim());
            if (id) {
                addToPlaylist(id);
                input.value = '';
            } else {
                showError("網址格式錯誤，請重新貼上。");
            }
        };

        document.getElementById('videoInput').onkeydown = (e) => {
            if (e.key === 'Enter') document.getElementById('loadBtn').click();
        };

        document.getElementById('pasteBtn').onclick = async () => {
            try {
                const text = await navigator.clipboard.readText();
                document.getElementById('videoInput').value = text;
            } catch (err) {
                showError("無法讀取剪貼簿，請手動貼上。");
            }
        };

        // Initialize Draggable Playlist
        new Sortable(document.getElementById('playlist'), {
            animation: 150,
            handle: '.handle',
            ghostClass: 'ghost-card',
            onEnd: function (evt) {
                const item = playlist.splice(evt.oldIndex, 1)[0];
                playlist.splice(evt.newIndex, 0, item);
                
                // Keep currentIndex updated
                if (currentIndex === evt.oldIndex) {
                    currentIndex = evt.newIndex;
                } else if (evt.oldIndex < currentIndex && evt.newIndex >= currentIndex) {
                    currentIndex--;
                } else if (evt.oldIndex > currentIndex && evt.newIndex <= currentIndex) {
                    currentIndex++;
                }
                renderPlaylist();
            }
        });
    </script>
</body>
</html>
