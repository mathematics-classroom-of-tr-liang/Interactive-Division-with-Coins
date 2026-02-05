# Interactive-Division-with-Coins
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>除法平分與換幣練習</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&display=swap');
        body {
            font-family: 'Noto Sans TC', sans-serif;
            user-select: none;
            overflow: hidden;
            height: 100vh;
            background-color: #f8fafc;
        }
        .coin {
            cursor: grab;
            transition: transform 0.15s, box-shadow 0.15s;
            touch-action: none;
        }
        .coin:active {
            cursor: grabbing;
            transform: scale(1.15);
        }
        .drop-zone-active {
            background-color: #fef3c7 !important;
            border-color: #f59e0b !important;
            transform: scale(1.01);
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
            z-index: 1000;
            animation: fadeInOut 2s ease-in-out forwards;
        }
    </style>
</head>
<body class="p-3 md:p-4">
    <div id="root" class="h-full flex flex-col gap-3"></div>
    <div id="toast-container"></div>

    <script>
        let history = [];
        let state = {
            dividend: 42,
            divisor: 3,
            bank: { tens: 4, ones: 2 },
            piles: Array.from({ length: 3 }, () => ({ tens: 0, ones: 0 })),
            dragging: null,
            isSetting: false 
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

        function onDragStart(type) { state.dragging = type; }

        function onDrop(pileIndex) {
            if (!state.dragging || state.bank[state.dragging] <= 0) return;
            const newPiles = JSON.parse(JSON.stringify(state.piles));
            newPiles[pileIndex][state.dragging]++;
            setState({
                bank: { ...state.bank, [state.dragging]: state.bank[state.dragging] - 1 },
                piles: newPiles,
                dragging: null
            });
        }

        function onDropExchange() {
            if (state.dragging === 'tens' && state.bank.tens > 0) {
                setState({
                    bank: { tens: state.bank.tens - 1, ones: state.bank.ones + 10 },
                    dragging: null
                });
                showToast("10 元 換成 10 個 1 元", "info");
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
                <div class="bg-white px-6 py-3 rounded-2xl shadow-sm border border-blue-100 flex items-center justify-between shrink-0">
                    <div class="flex items-center gap-4">
                        <div class="bg-blue-600 text-white p-2 rounded-xl shadow-md"><i data-lucide="book-open" size="24"></i></div>
                        <h1 class="text-2xl font-black text-slate-800">
                            把 <span class="text-blue-600 text-3xl mx-1 underline underline-offset-4 decoration-2">${state.dividend}</span> 元，
                            平分給 <span class="text-orange-500 text-3xl mx-1 underline underline-offset-4 decoration-2">${state.divisor}</span> 個人
                        </h1>
                    </div>
                    <div class="flex gap-2">
                        <button onclick="history.length > 0 && setState(history.pop())" class="p-2 bg-white border border-slate-200 text-slate-400 rounded-xl hover:border-blue-500 hover:text-blue-600 transition-all">
                            <i data-lucide="undo-2" size="20"></i>
                        </button>
                        <button onclick="setState({isSetting: true})" class="p-2 bg-slate-100 text-slate-500 hover:bg-blue-600 hover:text-white rounded-xl transition-all">
                            <i data-lucide="settings" size="20"></i>
                        </button>
                    </div>
                </div>

                <!-- Main -->
                <div class="flex-1 flex gap-4 overflow-hidden min-h-0">
                    
                    <!-- Left: Bank Area -->
                    <div class="w-1/3 max-w-[320px] flex flex-col gap-3 shrink-0 h-full">
                        <div class="bg-white rounded-[32px] border-2 border-blue-500 shadow-lg flex-1 flex flex-col p-4 overflow-hidden">
                            <div class="text-center mb-3">
                                <div class="inline-block bg-blue-600 text-white px-3 py-0.5 rounded-full text-[10px] font-black tracking-widest mb-1">BANK 銀行</div>
                                <div class="text-xl font-black text-slate-700">剩餘：<span class="text-blue-600 text-3xl">${bankTotal}</span> 元</div>
                            </div>

                            <div class="flex-1 flex flex-col gap-3 min-h-0 overflow-hidden">
                                <!-- 10元區 -->
                                <div class="bg-red-50 rounded-2xl p-3 border border-red-100 flex flex-col items-center">
                                    <div class="bank-label border-red-200 text-red-600 mb-2 w-full">
                                        <span class="text-2xl font-black">${state.bank.tens}</span> <span class="text-sm font-bold">個 10 元</span>
                                    </div>
                                    <div class="flex flex-wrap justify-center gap-1.5 overflow-y-auto max-h-[110px] custom-scrollbar">
                                        ${Array.from({ length: state.bank.tens }).map(() => `
                                            <div draggable="true" ondragstart="onDragStart('tens')" 
                                                 class="coin w-10 h-10 bg-red-500 border-2 border-red-600 rounded-full flex items-center justify-center text-white font-black text-sm shadow-md ring-2 ring-red-50">10</div>
                                        `).join('')}
                                    </div>
                                </div>

                                <!-- 1元區 -->
                                <div class="bg-blue-50 rounded-2xl p-3 border border-blue-100 flex flex-col items-center flex-1 overflow-hidden">
                                    <div class="bank-label border-blue-200 text-blue-600 mb-2 w-full">
                                        <span class="text-2xl font-black">${state.bank.ones}</span> <span class="text-sm font-bold">個 1 元</span>
                                    </div>
                                    <div class="flex-1 flex flex-wrap justify-center gap-1 overflow-y-auto custom-scrollbar w-full">
                                        ${Array.from({ length: state.bank.ones }).map(() => `
                                            <div draggable="true" ondragstart="onDragStart('ones')" 
                                                 class="coin w-7 h-7 bg-blue-400 border border-blue-500 rounded-full flex items-center justify-center text-white font-black text-[10px] shadow-sm ring-1 ring-blue-50">1</div>
                                        `).join('')}
                                    </div>
                                </div>
                            </div>
                        </div>

                        <!-- Exchange Area -->
                        <div ondragover="event.preventDefault(); this.classList.add('drop-zone-active')" 
                             ondrop="this.classList.remove('drop-zone-active'); onDropExchange()"
                             class="exchange-zone h-16 rounded-2xl flex flex-col items-center justify-center transition-all shadow-sm shrink-0">
                            <i data-lucide="refresh-cw" class="text-amber-500" size="18"></i>
                            <p class="text-xs font-black text-amber-700">10 元拖到此處換幣</p>
                        </div>
                    </div>

                    <!-- Right: Piles -->
                    <div class="flex-1 flex flex-col gap-3 overflow-hidden h-full">
                        <div class="flex-1 grid ${gridCols} gap-3 overflow-y-auto pr-2 custom-scrollbar pb-6 content-start">
                            ${state.piles.map((pile, i) => `
                                <div ondragover="event.preventDefault(); this.classList.add('drop-zone-active')" 
                                     ondrop="this.classList.remove('drop-zone-active'); onDrop(${i})"
                                     class="bg-white rounded-3xl p-4 border border-slate-200 shadow-sm flex flex-col gap-2 transition-all min-h-[130px]">
                                    
                                    <div class="flex justify-between items-center">
                                        <div class="flex items-center gap-2">
                                            <div class="w-8 h-8 bg-orange-100 rounded-lg flex items-center justify-center text-orange-500 border border-orange-200">
                                                <i data-lucide="user" size="16"></i>
                                            </div>
                                            <span class="text-sm font-black text-slate-500">第 ${i+1} 人</span>
                                        </div>
                                        <div class="bg-blue-50 px-3 py-1 rounded-xl border border-blue-100">
                                            <span class="text-blue-600 font-black text-xl">${pile.tens * 10 + pile.ones}</span>
                                            <span class="text-xs font-bold text-slate-400">元</span>
                                        </div>
                                    </div>

                                    <div class="flex-1 flex flex-wrap gap-1.5 content-start p-3 rounded-2xl bg-slate-50 shadow-inner min-h-[50px]">
                                        ${Array.from({ length: pile.tens }).map(() => `
                                            <div ondblclick="recallCoin(${i}, 'tens')" class="coin w-8 h-8 bg-red-500 border border-red-600 rounded-full flex items-center justify-center text-white font-black text-xs shadow-sm">10</div>
                                        `).join('')}
                                        ${Array.from({ length: pile.ones }).map(() => `
                                            <div ondblclick="recallCoin(${i}, 'ones')" class="coin w-6 h-6 bg-blue-400 border border-blue-500 rounded-full flex items-center justify-center text-white font-black text-[8px] shadow-xs">1</div>
                                        `).join('')}
                                    </div>
                                </div>
                            `).join('')}
                        </div>
                    </div>
                </div>

                <!-- Footer Instruction & Actions Bar -->
                <div class="bg-slate-800 rounded-2xl p-3 text-white shadow-md shrink-0 flex items-center justify-between px-6">
                    <div class="flex items-center gap-8">
                        <div class="flex items-center gap-2 border-r border-slate-600 pr-6">
                            <i data-lucide="info" size="18" class="text-blue-400"></i>
                            <span class="text-xs font-black tracking-widest uppercase">操作說明</span>
                        </div>
                        <div class="flex items-center gap-6">
                            <div class="flex items-center gap-2">
                                <div class="w-1.5 h-1.5 bg-blue-400 rounded-full"></div>
                                <p class="text-[11px] font-bold">拖曳錢幣平分</p>
                            </div>
                            <div class="flex items-center gap-2">
                                <div class="w-1.5 h-1.5 bg-amber-400 rounded-full"></div>
                                <p class="text-[11px] font-bold text-amber-100">雙擊錢幣收回</p>
                            </div>
                            <div class="flex items-center gap-2">
                                <div class="w-1.5 h-1.5 bg-green-400 rounded-full"></div>
                                <p class="text-[11px] font-bold text-green-100">換幣區可拆10元</p>
                            </div>
                        </div>
                    </div>

                    <!-- Action Buttons -->
                    <div class="flex items-center gap-3">
                        <button onclick="checkDivision()" class="px-4 py-1.5 bg-slate-700 hover:bg-slate-600 text-white rounded-xl text-sm font-black flex items-center gap-2 transition-all border border-slate-600">
                            <i data-lucide="shield-check" size="16" class="text-blue-400"></i>
                            檢查平分
                        </button>
                        <button onclick="completeExercise()" class="px-5 py-1.5 bg-blue-600 hover:bg-blue-500 text-white rounded-xl text-sm font-black flex items-center gap-2 transition-all shadow-lg ring-2 ring-blue-500/20">
                            <i data-lucide="party-popper" size="16"></i>
                            完成
                        </button>
                    </div>
                </div>

                <!-- Settings Modal -->
                ${state.isSetting ? `
                    <div class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[500] flex items-center justify-center p-4">
                        <div class="bg-white p-8 rounded-[40px] shadow-2xl w-full max-w-[400px] border-4 border-blue-600 animate-in zoom-in-95">
                            <h2 class="text-2xl font-black mb-6 text-slate-800 flex items-center gap-3">
                                <i data-lucide="settings-2" class="text-blue-600"></i> 設定題目
                            </h2>
                            <div class="space-y-6">
                                <div class="space-y-2">
                                    <label class="text-sm font-black text-slate-400">總金額 (1-99)</label>
                                    <input id="input-dividend" type="number" value="${state.dividend}" class="w-full text-3xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-blue-500 outline-none transition-all">
                                </div>
                                <div class="space-y-2">
                                    <label class="text-sm font-black text-slate-400">平分人數 (1-9)</label>
                                    <input id="input-divisor" type="number" value="${state.divisor}" min="1" max="9" class="w-full text-3xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-blue-500 outline-none transition-all">
                                </div>
                                <div class="flex gap-4">
                                    <button onclick="setState({isSetting: false})" class="flex-1 py-4 font-black text-slate-400 hover:bg-slate-100 rounded-2xl">取消</button>
                                    <button onclick="updateProblem()" class="flex-1 py-4 font-black bg-blue-600 text-white rounded-2xl shadow-lg hover:bg-blue-700">確認</button>
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
