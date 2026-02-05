<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
    <title>除法平分與換幣練習 - 平板觸控優化版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&display=swap');
        
        :root {
            --external-header-offset: 80px;
        }

        body {
            font-family: 'Noto Sans TC', sans-serif;
            user-select: none;
            background-color: #f8fafc;
            overflow: hidden; /* 防止整體滾動干擾拖曳 */
            height: 100vh;
            display: flex;
            flex-direction: column;
            touch-action: none; /* 禁用瀏覽器手勢，由程式接管 */
        }

        #root {
            flex: 1;
            display: flex;
            flex-direction: column;
            width: 100%;
            overflow-y: auto; /* 內容過長時在此滾動 */
            padding-bottom: 100px;
        }

        /* 拖曳中的錢幣樣式 */
        .coin {
            cursor: grab;
            transition: transform 0.1s;
            touch-action: none;
            position: relative;
            z-index: 10;
        }
        .coin.dragging {
            opacity: 0.9;
            transform: scale(1.3);
            box-shadow: 0 15px 30px rgba(0,0,0,0.3);
            pointer-events: none; /* 極重要：讓觸控穿透到下方元素 */
            position: fixed;
            z-index: 9999;
        }

        .drop-zone-active {
            background-color: #fef3c7 !important;
            border-color: #f59e0b !important;
            transform: scale(1.03);
            box-shadow: 0 0 15px rgba(245, 158, 11, 0.2);
        }

        .exchange-zone {
            background: linear-gradient(145deg, #fffbeb, #fef3c7);
            border: 2px dashed #f59e0b;
        }

        .bank-label {
            @apply bg-white px-3 py-1 rounded-xl border shadow-sm font-black text-center;
        }

        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 10px; }
        
        @keyframes fadeInOut {
            0% { opacity: 0; transform: translate(-50%, -60%); }
            15% { opacity: 1; transform: translate(-50%, -50%); }
            85% { opacity: 1; transform: translate(-50%, -50%); }
            100% { opacity: 0; transform: translate(-50%, -40%); }
        }
        .toast-message {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 2000;
            animation: fadeInOut 2s ease-in-out forwards;
        }

        .footer-fixed {
            position: fixed;
            bottom: 1rem;
            left: 1rem;
            right: 1rem;
            z-index: 100;
        }
    </style>
</head>
<body class="p-3 md:p-4">
    <div id="root" class="gap-3"></div>
    <div id="toast-container"></div>

    <script>
        let history = [];
        let state = {
            dividend: 42,
            divisor: 3,
            bank: { tens: 4, ones: 2 },
            piles: Array.from({ length: 3 }, () => ({ tens: 0, ones: 0 })),
            isSetting: false 
        };

        let activeDrag = {
            el: null,
            type: null,
            offsetX: 25, // 拖曳物件與觸控點的偏移量
            offsetY: 60  // 往上偏移一點，讓手指不會擋住視野
        };

        function saveToHistory() {
            history.push(JSON.parse(JSON.stringify(state)));
            if (history.length > 20) history.shift();
        }

        function setState(newState) {
            saveToHistory();
            state = { ...state, ...newState };
            render();
        }

        // --- 核心平板觸控邏輯 ---
        function handleDragStart(e, type) {
            // 避免觸發瀏覽器選取文字
            e.preventDefault();
            
            const point = e.type.includes('touch') ? e.touches[0] : e;
            const clientX = point.clientX;
            const clientY = point.clientY;
            
            activeDrag.type = type;
            activeDrag.el = e.target.closest('.coin').cloneNode(true);
            activeDrag.el.classList.add('dragging');
            
            // 初始位置稍微偏移手指上方，避免擋住 target 偵測
            activeDrag.el.style.left = `${clientX - activeDrag.offsetX}px`;
            activeDrag.el.style.top = `${clientY - activeDrag.offsetY}px`;
            
            document.body.appendChild(activeDrag.el);
            
            const moveEvent = e.type.includes('touch') ? 'touchmove' : 'mousemove';
            const endEvent = e.type.includes('touch') ? 'touchend' : 'mouseup';
            
            window.addEventListener(moveEvent, handleDragMove, { passive: false });
            window.addEventListener(endEvent, handleDragEnd);
        }

        function handleDragMove(e) {
            if (!activeDrag.el) return;
            e.preventDefault(); 
            
            const point = e.type.includes('touch') ? e.touches[0] : e;
            const clientX = point.clientX;
            const clientY = point.clientY;
            
            // 更新複本位置
            activeDrag.el.style.left = `${clientX - activeDrag.offsetX}px`;
            activeDrag.el.style.top = `${clientY - activeDrag.offsetY}px`;

            // 移除舊的啟動狀態
            document.querySelectorAll('.drop-target').forEach(el => el.classList.remove('drop-zone-active'));
            
            // 偵測手指正下方的元素 (因複本設定了 pointer-events: none，所以可以偵測到後方)
            const target = document.elementFromPoint(clientX, clientY);
            const dropZone = target?.closest('.drop-target');
            if (dropZone) dropZone.classList.add('drop-zone-active');
        }

        function handleDragEnd(e) {
            if (!activeDrag.el) return;
            
            const point = e.type.includes('touch') ? e.changedTouches[0] : e;
            const clientX = point.clientX;
            const clientY = point.clientY;
            
            const target = document.elementFromPoint(clientX, clientY);
            const dropZone = target?.closest('.drop-target');
            
            if (dropZone) {
                const pileIndex = dropZone.getAttribute('data-pile-index');
                if (pileIndex !== null) {
                    onDrop(parseInt(pileIndex));
                } else if (dropZone.id === 'exchange-zone') {
                    onDropExchange();
                }
            }

            activeDrag.el.remove();
            activeDrag.el = null;
            activeDrag.type = null;
            document.querySelectorAll('.drop-target').forEach(el => el.classList.remove('drop-zone-active'));
            
            window.removeEventListener('mousemove', handleDragMove);
            window.removeEventListener('mouseup', handleDragEnd);
            window.removeEventListener('touchmove', handleDragMove);
            window.removeEventListener('touchend', handleDragEnd);
        }

        function onDrop(pileIndex) {
            const type = activeDrag.type;
            if (!type || state.bank[type] <= 0) return;
            const newPiles = JSON.parse(JSON.stringify(state.piles));
            newPiles[pileIndex][type]++;
            setState({
                bank: { ...state.bank, [type]: state.bank[type] - 1 },
                piles: newPiles
            });
        }

        function onDropExchange() {
            if (activeDrag.type === 'tens' && state.bank.tens > 0) {
                setState({
                    bank: { tens: state.bank.tens - 1, ones: state.bank.ones + 10 }
                });
                showToast("10 元 換成 10 個 1 元", "info");
            }
        }

        function updateProblem() {
            const newDiv = parseInt(document.getElementById('input-dividend').value) || 0;
            const newDivisor = parseInt(document.getElementById('input-divisor').value) || 1;
            
            if (newDiv > 99) { showToast("金額建議不超過 99 元喔！", "info"); }
            if (newDivisor > 9) {
                showToast("人數上限為 9 人喔！", "warning");
                return;
            }

            state = {
                ...state,
                dividend: newDiv,
                divisor: newDivisor,
                bank: { tens: Math.floor(newDiv / 10), ones: newDiv % 10 },
                piles: Array.from({ length: newDivisor }, () => ({ tens: 0, ones: 0 })),
                isSetting: false
            };
            history = [];
            render();
            showToast("佈題成功！", "success");
        }

        function recallCoin(pileIndex, type) {
            const newPiles = JSON.parse(JSON.stringify(state.piles));
            if (newPiles[pileIndex][type] > 0) {
                newPiles[pileIndex][type]--;
                setState({
                    bank: { ...state.bank, [type]: state.bank[type] + 1 },
                    piles: newPiles
                });
            }
        }

        function checkDivision() {
            const bankTotal = state.bank.tens * 10 + state.bank.ones;
            const values = state.piles.map(p => p.tens * 10 + p.ones);
            const allEqual = values.every(v => v === values[0]);
            
            if (bankTotal > 0) {
                showToast("銀行還有錢沒分完喔！", "warning");
            } else if (!allEqual) {
                showToast("每個人分到的錢不一樣多喔！", "warning");
            } else {
                showToast("太棒了！每個人分得一樣多！", "success");
            }
        }

        function completeExercise() {
            const bankTotal = state.bank.tens * 10 + state.bank.ones;
            if (bankTotal > 0) {
                showToast("先分完再點完成吧！", "info");
                return;
            }
            showToast("挑戰成功！準備下一題。", "success");
            setTimeout(() => setState({ isSetting: true }), 1500);
        }

        function showToast(message, type) {
            const container = document.getElementById('toast-container');
            container.innerHTML = ""; 
            const bgColor = type === 'success' ? 'bg-green-600' : (type === 'warning' ? 'bg-amber-500' : 'bg-blue-600');
            const toast = document.createElement('div');
            toast.className = `toast-message px-6 py-3 rounded-2xl shadow-xl text-white text-xl font-black flex items-center gap-3 ${bgColor}`;
            toast.innerHTML = `<i data-lucide="${type === 'success' ? 'check' : (type === 'warning' ? 'alert-circle' : 'info')}"></i><span>${message}</span>`;
            container.appendChild(toast);
            lucide.createIcons();
            setTimeout(() => toast.remove(), 2000);
        }

        function render() {
            const root = document.getElementById('root');
            const bankTotal = state.bank.tens * 10 + state.bank.ones;

            let gridCols = "grid-cols-1";
            if (state.divisor > 3) gridCols = "grid-cols-2";
            if (state.divisor >= 7) gridCols = "grid-cols-3";

            root.innerHTML = `
                <!-- Header -->
                <div class="bg-white px-6 py-3 rounded-2xl shadow-sm border border-blue-100 flex items-center justify-between shrink-0 m-1">
                    <div class="flex items-center gap-4">
                        <div class="bg-blue-600 text-white p-2 rounded-xl shadow-md"><i data-lucide="book-open" size="24"></i></div>
                        <h1 class="text-xl md:text-2xl font-black text-slate-800">
                            把 <span class="text-blue-600 text-2xl md:text-3xl mx-1 underline underline-offset-4 decoration-2">${state.dividend}</span> 元，
                            平分給 <span class="text-orange-500 text-2xl md:text-3xl mx-1 underline underline-offset-4 decoration-2">${state.divisor}</span> 人
                        </h1>
                    </div>
                    <div class="flex gap-2">
                        <button onclick="history.length > 0 && setState(history.pop())" class="p-2 bg-white border border-slate-200 text-slate-400 rounded-xl hover:border-blue-500 hover:text-blue-600 transition-all active:scale-90">
                            <i data-lucide="undo-2" size="20"></i>
                        </button>
                        <button onclick="setState({isSetting: true})" class="p-2 bg-slate-100 text-slate-500 hover:bg-blue-600 hover:text-white rounded-xl transition-all active:scale-90">
                            <i data-lucide="settings" size="20"></i>
                        </button>
                    </div>
                </div>

                <!-- Main Content Area -->
                <div class="flex flex-col md:flex-row gap-4 p-1">
                    
                    <!-- Left Area: Bank -->
                    <div class="w-full md:w-1/3 max-w-[320px] flex flex-col gap-3 shrink-0">
                        <div class="bg-white rounded-[32px] border-2 border-blue-500 shadow-lg flex flex-col p-4">
                            <div class="text-center mb-3 font-black text-slate-700">
                                <div class="inline-block bg-blue-600 text-white px-3 py-0.5 rounded-full text-[10px] mb-1 uppercase tracking-tighter">Bank 銀行</div>
                                <div>剩餘：<span class="text-blue-600 text-3xl">${bankTotal}</span> 元</div>
                            </div>

                            <div class="flex flex-col gap-3">
                                <div class="bg-red-50 rounded-2xl p-3 border border-red-100 flex flex-col items-center">
                                    <div class="bank-label border-red-200 text-red-600 mb-2 w-full">
                                        <span class="text-2xl font-black">${state.bank.tens}</span> <span class="text-sm">個 10 元</span>
                                    </div>
                                    <div class="flex flex-wrap justify-center gap-2 min-h-[60px] max-h-[150px] overflow-y-auto custom-scrollbar w-full p-1">
                                        ${Array.from({ length: state.bank.tens }).map(() => `
                                            <div onmousedown="handleDragStart(event, 'tens')" ontouchstart="handleDragStart(event, 'tens')"
                                                 class="coin w-10 h-10 bg-red-500 border-2 border-red-600 rounded-full flex items-center justify-center text-white font-black text-sm shadow-md">10</div>
                                        `).join('')}
                                    </div>
                                </div>

                                <div class="bg-blue-50 rounded-2xl p-3 border border-blue-100 flex flex-col items-center">
                                    <div class="bank-label border-blue-200 text-blue-600 mb-2 w-full">
                                        <span class="text-2xl font-black">${state.bank.ones}</span> <span class="text-sm">個 1 元</span>
                                    </div>
                                    <div class="flex flex-wrap justify-center gap-1.5 min-h-[60px] max-h-[180px] overflow-y-auto custom-scrollbar w-full p-1">
                                        ${Array.from({ length: state.bank.ones }).map(() => `
                                            <div onmousedown="handleDragStart(event, 'ones')" ontouchstart="handleDragStart(event, 'ones')"
                                                 class="coin w-7 h-7 bg-blue-400 border border-blue-500 rounded-full flex items-center justify-center text-white font-black text-[10px] shadow-sm">1</div>
                                        `).join('')}
                                    </div>
                                </div>
                            </div>
                        </div>

                        <div id="exchange-zone" class="drop-target exchange-zone h-16 rounded-2xl flex flex-col items-center justify-center transition-all shadow-sm">
                            <i data-lucide="refresh-cw" class="text-amber-500" size="18"></i>
                            <p class="text-[10px] font-black text-amber-700">將 10 元拖至此處換幣</p>
                        </div>
                    </div>

                    <!-- Right Area: Piles -->
                    <div class="flex-1">
                        <div class="grid ${gridCols} gap-3 content-start">
                            ${state.piles.map((pile, i) => `
                                <div data-pile-index="${i}"
                                     class="drop-target bg-white rounded-3xl p-4 border border-slate-200 shadow-sm flex flex-col gap-2 transition-all min-h-[140px]">
                                    
                                    <div class="flex justify-between items-center">
                                        <div class="flex items-center gap-2">
                                            <div class="w-7 h-7 bg-orange-100 rounded-lg flex items-center justify-center text-orange-500 border border-orange-200">
                                                <i data-lucide="user" size="14"></i>
                                            </div>
                                            <span class="text-xs font-black text-slate-500">第 ${i+1} 人</span>
                                        </div>
                                        <div class="bg-blue-50 px-2 py-0.5 rounded-lg border border-blue-100">
                                            <span class="text-blue-600 font-black text-lg">${pile.tens * 10 + pile.ones}</span>
                                            <span class="text-[10px] font-bold text-slate-400">元</span>
                                        </div>
                                    </div>

                                    <div class="flex-1 flex flex-wrap gap-1.5 content-start p-3 rounded-2xl bg-slate-50 shadow-inner min-h-[60px]">
                                        ${Array.from({ length: pile.tens }).map(() => `
                                            <div onclick="recallCoin(${i}, 'tens')" class="coin w-8 h-8 bg-red-500 border border-red-600 rounded-full flex items-center justify-center text-white font-black text-[10px] shadow-sm active:scale-110 transition-transform">10</div>
                                        `).join('')}
                                        ${Array.from({ length: pile.ones }).map(() => `
                                            <div onclick="recallCoin(${i}, 'ones')" class="coin w-6 h-6 bg-blue-400 border border-blue-500 rounded-full flex items-center justify-center text-white font-black text-[8px] shadow-xs active:scale-110 transition-transform">1</div>
                                        `).join('')}
                                    </div>
                                </div>
                            `).join('')}
                        </div>
                    </div>
                </div>

                <!-- Footer Bar (Fixed) -->
                <div class="footer-fixed bg-slate-800 rounded-2xl p-3 text-white shadow-xl flex flex-col lg:flex-row items-center justify-between px-6 gap-4">
                    <div class="flex flex-wrap items-center justify-center gap-4">
                        <div class="flex items-center gap-2 lg:border-r border-slate-600 lg:pr-6">
                            <i data-lucide="info" size="18" class="text-blue-400"></i>
                            <span class="text-xs font-black tracking-widest">操作說明</span>
                        </div>
                        <div class="flex flex-wrap justify-center gap-4 text-[10px] font-bold">
                            <div class="flex items-center gap-2"><div class="w-1.5 h-1.5 bg-blue-400 rounded-full"></div>拖曳分錢</div>
                            <div class="flex items-center gap-2 text-amber-100"><div class="w-1.5 h-1.5 bg-amber-400 rounded-full"></div>點擊盤中錢幣收回</div>
                        </div>
                    </div>

                    <div class="flex items-center gap-3">
                        <button onclick="checkDivision()" class="px-4 py-1.5 bg-slate-700 border border-slate-600 rounded-xl text-sm font-black flex items-center gap-2 transition-all hover:bg-slate-600 active:scale-95">
                            <i data-lucide="shield-check" size="16" class="text-blue-400"></i>檢查
                        </button>
                        <button onclick="completeExercise()" class="px-5 py-1.5 bg-blue-600 rounded-xl text-sm font-black flex items-center gap-2 shadow-lg hover:bg-blue-500 active:scale-95">
                            <i data-lucide="party-popper" size="16"></i>完成
                        </button>
                    </div>
                </div>

                <!-- Settings Modal -->
                ${state.isSetting ? `
                    <div class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[2500] flex items-center justify-center p-4">
                        <div class="bg-white p-8 rounded-[40px] shadow-2xl w-full max-w-[400px] border-4 border-blue-600">
                            <h2 class="text-2xl font-black mb-6 text-slate-800 flex items-center gap-3">
                                <i data-lucide="settings-2" class="text-blue-600"></i> 設定題目
                            </h2>
                            <div class="space-y-6">
                                <div class="space-y-2">
                                    <label class="text-sm font-black text-slate-400">總金額 (1-99)</label>
                                    <input id="input-dividend" type="number" value="${state.dividend}" class="w-full text-3xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-blue-500 outline-none">
                                </div>
                                <div class="space-y-2">
                                    <label class="text-sm font-black text-slate-400">平分人數 (1-9)</label>
                                    <input id="input-divisor" type="number" value="${state.divisor}" min="1" max="9" class="w-full text-3xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-blue-500 outline-none">
                                </div>
                                <div class="flex gap-4">
                                    <button onclick="setState({isSetting: false})" class="flex-1 py-4 font-black text-slate-400 active:scale-95">取消</button>
                                    <button onclick="updateProblem()" class="flex-1 py-4 font-black bg-blue-600 text-white rounded-2xl shadow-lg active:scale-95">確認</button>
                                </div>
                            </div>
                        </div>
                    </div>
                ` : ''}
            `;
            lucide.createIcons();
        }

        render();
    </script>
</body>
</html>
