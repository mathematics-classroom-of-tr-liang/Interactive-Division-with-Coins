<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=yes, viewport-fit=cover">
    <title>除法平分與換幣練習 - 平板相容版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&display=swap');
        
        body {
            font-family: 'Noto Sans TC', sans-serif;
            user-select: none;
            background-color: #f8fafc;
            /* 允許捲動，防止內容超出 */
            overflow-y: auto; 
            min-height: 100vh;
            padding-bottom: 140px; /* 為底部按鈕預留空間 */
        }

        /* 拖曳中的錢幣樣式 */
        .coin {
            cursor: grab;
            transition: transform 0.1s;
            touch-action: none; /* 僅錢幣本身禁止原生手勢，確保拖曳順暢 */
        }
        .coin.dragging {
            opacity: 0.9;
            transform: scale(1.3);
            box-shadow: 0 15px 30px rgba(0,0,0,0.3);
            pointer-events: none; 
            position: fixed;
            z-index: 9999;
        }

        .drop-zone-active {
            background-color: #fef3c7 !important;
            border-color: #f59e0b !important;
            transform: scale(1.02);
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
            bottom: 0;
            left: 0;
            right: 0;
            z-index: 100;
            margin: 1rem;
        }

        .coin-in-pile {
            transition: transform 0.1s;
            cursor: pointer;
        }
    </style>
</head>
<body class="p-3 md:p-6">
    <div id="root" class="flex flex-col gap-4 max-w-7xl mx-auto"></div>
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

        // 用於手動處理平板雙擊
        let lastClickTime = 0;
        let lastClickedItem = null;

        let activeDrag = {
            el: null,
            type: null,
            offsetX: 25, 
            offsetY: 40  
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

        // 處理拖曳開始
        function handleDragStart(e, type) {
            const point = e.type.includes('touch') ? e.touches[0] : e;
            const clientX = point.clientX;
            const clientY = point.clientY;
            
            activeDrag.type = type;
            activeDrag.el = e.target.closest('.coin').cloneNode(true);
            activeDrag.el.classList.add('dragging');
            
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
            // 只有在拖曳錢幣時才阻止默認捲動，方便操作
            e.preventDefault(); 
            
            const point = e.type.includes('touch') ? e.touches[0] : e;
            const clientX = point.clientX;
            const clientY = point.clientY;
            
            activeDrag.el.style.left = `${clientX - activeDrag.offsetX}px`;
            activeDrag.el.style.top = `${clientY - activeDrag.offsetY}px`;

            document.querySelectorAll('.drop-target').forEach(el => el.classList.remove('drop-zone-active'));
            
            const target = document.elementFromPoint(clientX, clientY);
            const dropZone = target?.closest('.drop-target');
            if (dropZone) dropZone.classList.add('drop-zone-active');
        }

        function handleDragEnd(e) {
            if (!activeDrag.el) return;
            
            const point = e.type.includes('touch') ? e.changedTouches[0] : e;
            const clientX = point.clientX || 0;
            const clientY = point.clientY || 0;
            
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
                showToast("10 元換成 10 個 1 元成功！", "info");
            }
        }

        // 手動雙擊處理器 (相容平板 Chrome)
        function handleManualDblClick(pileIndex, type, event) {
            const currentTime = new Date().getTime();
            const tapGap = currentTime - lastClickTime;
            const currentItemKey = `${pileIndex}-${type}`;

            if (tapGap < 300 && tapGap > 0 && lastClickedItem === currentItemKey) {
                // 觸發收回
                recallCoin(pileIndex, type);
                lastClickTime = 0; // 重置
            } else {
                lastClickTime = currentTime;
                lastClickedItem = currentItemKey;
            }
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
            const values = state.piles.map(p => p.tens * 10 + p.ones);
            const firstValue = values[0];
            const allEqual = values.every(v => v === firstValue);
            
            if (!allEqual) {
                showToast("每個人盤內的金額不相等喔！", "warning");
            } else {
                showToast(`檢查完成！每個人盤內都是 ${firstValue} 元，一樣多。`, "success");
            }
        }

        function completeExercise() {
            const bankTotal = state.bank.tens * 10 + state.bank.ones;
            const values = state.piles.map(p => p.tens * 10 + p.ones);
            const allEqual = values.every(v => v === values[0]);

            if (bankTotal === 0 && allEqual) {
                showToast("挑戰成功！正在準備下一題...", "success");
                setTimeout(() => setState({ isSetting: true }), 1500);
            } else if (bankTotal > 0) {
                showToast(`銀行還有 ${bankTotal} 元沒分完喔！`, "warning");
            } else {
                checkDivision();
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

        function showToast(message, type) {
            const container = document.getElementById('toast-container');
            container.innerHTML = ""; 
            const bgColor = type === 'success' ? 'bg-green-600' : (type === 'warning' ? 'bg-amber-500' : 'bg-blue-600');
            const toast = document.createElement('div');
            toast.className = `toast-message px-6 py-3 rounded-2xl shadow-xl text-white text-xl font-black flex items-center gap-3 ${bgColor}`;
            toast.innerHTML = `<i data-lucide="${type === 'success' ? 'check' : (type === 'warning' ? 'alert-circle' : 'info')}"></i><span>${message}</span>`;
            container.appendChild(toast);
            lucide.createIcons();
            setTimeout(() => toast.remove(), 2500);
        }

        function render() {
            const root = document.getElementById('root');
            const bankTotal = state.bank.tens * 10 + state.bank.ones;

            let gridCols = "grid-cols-1";
            if (state.divisor > 3) gridCols = "md:grid-cols-2";
            if (state.divisor >= 7) gridCols = "md:grid-cols-3";

            root.innerHTML = `
                <!-- Header -->
                <div class="bg-white px-6 py-4 rounded-3xl shadow-sm border border-blue-100 flex items-center justify-between shrink-0">
                    <div class="flex items-center gap-4">
                        <div class="bg-blue-600 text-white p-2 rounded-xl"><i data-lucide="book-open" size="24"></i></div>
                        <h1 class="text-xl md:text-2xl font-black text-slate-800">
                            把 <span class="text-blue-600">${state.dividend}</span> 元，
                            平分給 <span class="text-orange-500">${state.divisor}</span> 人
                        </h1>
                    </div>
                    <div class="flex gap-2">
                        <button onclick="history.length > 0 && setState(history.pop())" class="p-2 border border-slate-200 rounded-xl hover:text-blue-600">
                            <i data-lucide="undo-2" size="20"></i>
                        </button>
                        <button onclick="setState({isSetting: true})" class="p-2 bg-slate-100 text-slate-500 rounded-xl">
                            <i data-lucide="settings" size="20"></i>
                        </button>
                    </div>
                </div>

                <!-- Main Content Area -->
                <div class="flex flex-col lg:flex-row gap-6">
                    
                    <!-- Bank Area -->
                    <div class="w-full lg:w-72 shrink-0 flex flex-col gap-4">
                        <div class="bg-white rounded-[40px] border-2 border-blue-500 shadow-xl p-5">
                            <div class="text-center mb-4">
                                <span class="bg-blue-600 text-white px-3 py-1 rounded-full text-xs font-bold uppercase">Bank 銀行</span>
                                <div class="text-3xl font-black text-blue-600 mt-1">${bankTotal} <span class="text-sm">元</span></div>
                            </div>

                            <div class="space-y-4">
                                <div class="bg-red-50 rounded-2xl p-4 border border-red-100">
                                    <div class="bank-label border-red-200 text-red-600 text-lg mb-3">10元：${state.bank.tens}</div>
                                    <div class="flex flex-wrap justify-center gap-2 min-h-[60px]">
                                        ${Array.from({ length: state.bank.tens }).map(() => `
                                            <div onmousedown="handleDragStart(event, 'tens')" ontouchstart="handleDragStart(event, 'tens')"
                                                 class="coin w-12 h-12 bg-red-500 border-2 border-red-600 rounded-full flex items-center justify-center text-white font-black shadow-md active:scale-90">10</div>
                                        `).join('')}
                                    </div>
                                </div>

                                <div class="bg-blue-50 rounded-2xl p-4 border border-blue-100">
                                    <div class="bank-label border-blue-200 text-blue-600 text-lg mb-3">1元：${state.bank.ones}</div>
                                    <div class="flex flex-wrap justify-center gap-2 min-h-[60px]">
                                        ${Array.from({ length: state.bank.ones }).map(() => `
                                            <div onmousedown="handleDragStart(event, 'ones')" ontouchstart="handleDragStart(event, 'ones')"
                                                 class="coin w-9 h-9 bg-blue-400 border border-blue-500 rounded-full flex items-center justify-center text-white font-black text-xs shadow-md active:scale-90">1</div>
                                        `).join('')}
                                    </div>
                                </div>
                            </div>
                        </div>

                        <div id="exchange-zone" class="drop-target exchange-zone h-20 rounded-[30px] flex flex-col items-center justify-center transition-all">
                            <i data-lucide="refresh-cw" class="text-amber-500 mb-1" size="24"></i>
                            <p class="text-xs font-black text-amber-700">10 元換 10 個 1 元</p>
                        </div>
                    </div>

                    <!-- Piles Area -->
                    <div class="flex-1 grid ${gridCols} gap-4">
                        ${state.piles.map((pile, i) => `
                            <div data-pile-index="${i}"
                                 class="drop-target bg-white rounded-[32px] p-5 border border-slate-200 shadow-sm flex flex-col gap-3 min-h-[160px]">
                                
                                <div class="flex justify-between items-center">
                                    <div class="flex items-center gap-3">
                                        <div class="w-10 h-10 bg-orange-100 rounded-xl flex items-center justify-center text-orange-600">
                                            <i data-lucide="user" size="20"></i>
                                        </div>
                                        <span class="font-black text-slate-700 text-lg">第 ${i+1} 人</span>
                                    </div>
                                    <div class="bg-blue-600 px-4 py-1 rounded-full text-white font-black text-xl shadow-inner">
                                        ${pile.tens * 10 + pile.ones} <span class="text-xs">元</span>
                                    </div>
                                </div>

                                <div class="flex-1 flex flex-wrap gap-2 content-start p-4 rounded-2xl bg-slate-50 shadow-inner min-h-[80px]">
                                    ${Array.from({ length: pile.tens }).map(() => `
                                        <div onclick="handleManualDblClick(${i}, 'tens', event)"
                                             class="coin-in-pile w-11 h-11 bg-red-500 border border-red-600 rounded-full flex items-center justify-center text-white font-black text-sm shadow-sm active:scale-125">10</div>
                                    `).join('')}
                                    ${Array.from({ length: pile.ones }).map(() => `
                                        <div onclick="handleManualDblClick(${i}, 'ones', event)"
                                             class="coin-in-pile w-8 h-8 bg-blue-400 border border-blue-500 rounded-full flex items-center justify-center text-white font-black text-[10px] shadow-xs active:scale-125">1</div>
                                    `).join('')}
                                </div>
                            </div>
                        `).join('')}
                    </div>
                </div>

                <!-- Footer -->
                <div class="footer-fixed bg-slate-800 rounded-3xl p-4 text-white shadow-2xl flex flex-col md:flex-row items-center justify-between px-8 gap-4">
                    <div class="flex items-center gap-6">
                        <div class="flex items-center gap-2 text-blue-400">
                            <i data-lucide="info" size="20"></i>
                            <span class="text-sm font-black">操作：</span>
                        </div>
                        <div class="flex gap-6 text-sm font-bold text-slate-300">
                            <span class="flex items-center gap-2"><i data-lucide="mouse-pointer-2" size="14"></i> 拖曳分錢</span>
                            <span class="flex items-center gap-2 text-amber-300"><i data-lucide="hand" size="14"></i> 雙擊盤中錢幣可收回</span>
                        </div>
                    </div>

                    <div class="flex gap-4">
                        <button onclick="checkDivision()" class="px-6 py-2 bg-slate-700 rounded-2xl font-black flex items-center gap-2 hover:bg-slate-600 active:scale-95 transition-all">
                            <i data-lucide="scale" size="18" class="text-blue-400"></i>檢查平分
                        </button>
                        <button onclick="completeExercise()" class="px-8 py-2 bg-blue-600 rounded-2xl font-black flex items-center gap-2 shadow-lg hover:bg-blue-500 active:scale-95 transition-all">
                            <i data-lucide="check-circle" size="18"></i>完成分錢
                        </button>
                    </div>
                </div>

                <!-- Settings Modal -->
                ${state.isSetting ? `
                    <div class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[2500] flex items-center justify-center p-4">
                        <div class="bg-white p-8 rounded-[40px] shadow-2xl w-full max-w-[420px] border-4 border-blue-600">
                            <h2 class="text-2xl font-black mb-6 text-slate-800 flex items-center gap-3">
                                <i data-lucide="settings-2" class="text-blue-600"></i> 設定題目
                            </h2>
                            <div class="space-y-6">
                                <div class="space-y-2">
                                    <label class="text-sm font-black text-slate-400 uppercase tracking-widest">總金額 (1-99 元)</label>
                                    <input id="input-dividend" type="number" value="${state.dividend}" class="w-full text-4xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-blue-500 outline-none">
                                </div>
                                <div class="space-y-2">
                                    <label class="text-sm font-black text-slate-400 uppercase tracking-widest">平分人數 (1-9 人)</label>
                                    <input id="input-divisor" type="number" value="${state.divisor}" min="1" max="9" class="w-full text-4xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-blue-500 outline-none">
                                </div>
                                <div class="flex gap-4 pt-4">
                                    <button onclick="setState({isSetting: false})" class="flex-1 py-4 font-black text-slate-400 active:scale-95">取消</button>
                                    <button onclick="updateProblem()" class="flex-1 py-4 font-black bg-blue-600 text-white rounded-2xl shadow-lg active:scale-95">更新題目</button>
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
