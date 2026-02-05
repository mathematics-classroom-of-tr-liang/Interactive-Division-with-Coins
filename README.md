<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
    <title>除法平分練習 - 教學強化版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&display=swap');
        
        body {
            font-family: 'Noto Sans TC', sans-serif;
            user-select: none;
            background-color: #f8fafc;
            height: 100vh;
            overflow: hidden;
            display: flex;
            flex-direction: column;
        }

        .coin {
            cursor: grab;
            transition: transform 0.1s;
            touch-action: none;
        }
        .coin.dragging {
            opacity: 0.9;
            transform: scale(1.2);
            box-shadow: 0 10px 20px rgba(0,0,0,0.2);
            pointer-events: none; 
            position: fixed;
            z-index: 9999;
        }

        .drop-target-active {
            background-color: #fef3c7 !important;
            border-color: #f59e0b !important;
        }

        .bank-slot {
            @apply bg-white p-2 rounded-xl border border-slate-100 shadow-inner flex-1 overflow-y-auto;
        }
        
        .modal-animate {
            animation: fadeIn 0.15s ease-out forwards;
        }
        @keyframes fadeIn {
            from { opacity: 0; transform: scale(0.95); }
            to { opacity: 1; transform: scale(1); }
        }

        .numpad-btn {
            @apply h-14 bg-slate-100 rounded-xl text-xl font-black text-slate-700 hover:bg-indigo-100 active:scale-95 transition-all;
        }

        ::-webkit-scrollbar { width: 6px; }
        ::-webkit-scrollbar-track { background: #f1f5f9; }
        ::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 10px; }
    </style>
</head>
<body class="p-4 md:p-6">
    <div id="root" class="max-w-[1600px] mx-auto w-full h-full flex flex-col gap-4"></div>

    <script>
        let history = [];
        let state = {
            dividend: 42,
            divisor: 3,
            bank: { tens: 4, ones: 2 },
            piles: Array.from({ length: 3 }, () => ({ tens: 0, ones: 0 })),
            isSetting: false,
            quizStep: 'idle', 
            tempRemainder: "",
            message: "請開始平分錢幣。"
        };

        function saveToHistory() {
            history.push(JSON.parse(JSON.stringify(state)));
            if (history.length > 20) history.shift();
        }

        function setState(newState) {
            saveToHistory();
            state = { ...state, ...newState };
            updateMessage();
            render();
        }

        function updateMessage() {
            const totalInPiles = state.piles.reduce((acc, p) => acc + (p.tens * 10 + p.ones), 0);
            const bankTotal = state.bank.tens * 10 + state.bank.ones;
            if (totalInPiles === 0) state.message = "拖曳 10 元或 1 元到右側人員框。";
            else if (state.bank.tens >= state.divisor) state.message = "銀行還有 10 元，可繼續平分。";
            else if (state.bank.tens > 0 && state.bank.tens < state.divisor) state.message = "10 元不夠分了，請拖曳到換幣區。";
            else if (bankTotal >= state.divisor) state.message = "繼續平分剩下的 1 元。";
            else state.message = "分完了嗎？請點擊右下角檢查。";
        }

        let activeDrag = { el: null, type: null, offsetX: 20, offsetY: 20 };

        function handleDragStart(e, type) {
            const point = e.type.includes('touch') ? e.touches[0] : e;
            activeDrag.type = type;
            activeDrag.el = e.target.closest('.coin').cloneNode(true);
            activeDrag.el.classList.add('dragging');
            activeDrag.el.style.left = `${point.clientX - activeDrag.offsetX}px`;
            activeDrag.el.style.top = `${point.clientY - activeDrag.offsetY}px`;
            document.body.appendChild(activeDrag.el);

            const moveEvent = e.type.includes('touch') ? 'touchmove' : 'mousemove';
            const endEvent = e.type.includes('touch') ? 'touchend' : 'mouseup';
            window.addEventListener(moveEvent, handleDragMove, { passive: false });
            window.addEventListener(endEvent, handleDragEnd);
        }

        function handleDragMove(e) {
            if (!activeDrag.el) return;
            if (e.cancelable) e.preventDefault();
            const point = e.type.includes('touch') ? e.touches[0] : e;
            activeDrag.el.style.left = `${point.clientX - activeDrag.offsetX}px`;
            activeDrag.el.style.top = `${point.clientY - activeDrag.offsetY}px`;
            
            document.querySelectorAll('.drop-target').forEach(el => el.classList.remove('drop-target-active'));
            const target = document.elementFromPoint(point.clientX, point.clientY);
            const dropZone = target?.closest('.drop-target');
            if (dropZone) dropZone.classList.add('drop-target-active');
        }

        function handleDragEnd(e) {
            if (!activeDrag.el) return;
            const point = e.type.includes('touch') ? e.changedTouches[0] : e;
            const target = document.elementFromPoint(point.clientX, point.clientY);
            const dropZone = target?.closest('.drop-target');
            
            if (dropZone) {
                const pileIndex = dropZone.getAttribute('data-pile-index');
                if (pileIndex !== null) onDrop(parseInt(pileIndex));
                else if (dropZone.id === 'exchange-zone') onDropExchange();
            }
            
            activeDrag.el.remove();
            activeDrag.el = null;
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
            setState({ bank: { ...state.bank, [type]: state.bank[type] - 1 }, piles: newPiles });
        }

        function onDropExchange() {
            if (activeDrag.type === 'tens' && state.bank.tens > 0) {
                setState({ bank: { tens: state.bank.tens - 1, ones: state.bank.ones + 10 } });
            }
        }

        function handleManualBack(pileIndex, type) {
            const newPiles = JSON.parse(JSON.stringify(state.piles));
            if (newPiles[pileIndex][type] > 0) {
                newPiles[pileIndex][type]--;
                setState({ bank: { ...state.bank, [type]: state.bank[type] + 1 }, piles: newPiles });
            }
        }

        function checkFairness() {
            const amounts = state.piles.map(p => p.tens * 10 + p.ones);
            const allEqual = amounts.every(a => a === amounts[0]);
            if (amounts.every(a => a === 0)) customAlert("還沒開始分錢喔！", "info");
            else if (allEqual) customAlert("大家分到的一樣多，非常公平！", "success");
            else customAlert("大家分到的錢不一樣多，再檢查一下。", "warning");
        }

        function startQuiz() {
            const amounts = state.piles.map(p => p.tens * 10 + p.ones);
            if (!amounts.every(a => a === amounts[0])) {
                customAlert("大家分到的錢要一樣多喔！", "warning");
                return;
            }
            setState({ quizStep: 'judging' });
        }

        function handleJudge(choice) {
            const actualRemainder = state.bank.tens * 10 + state.bank.ones;
            if (choice === 'none') {
                if (actualRemainder === 0) { customAlert("太棒了！正確完成。", "success"); setState({ quizStep: 'idle' }); }
                else { customAlert("銀行還有錢喔，再看看。", "warning"); setState({ quizStep: 'idle' }); }
            } else setState({ quizStep: 'inputtingRemainder', tempRemainder: "" });
        }

        function submitRemainder() {
            const actualRemainder = state.bank.tens * 10 + state.bank.ones;
            if (parseInt(state.tempRemainder) === actualRemainder) {
                customAlert("答對了！分得非常正確。", "success"); setState({ quizStep: 'idle' });
            } else {
                customAlert("餘數不對喔，再數數看？", "warning"); setState({ quizStep: 'inputtingRemainder', tempRemainder: "" });
            }
        }

        function customAlert(msg, type = "info") {
            const alertBox = document.createElement('div');
            const bgClass = type === 'success' ? 'bg-green-600' : (type === 'warning' ? 'bg-orange-500' : 'bg-slate-800');
            alertBox.className = `fixed top-10 left-1/2 -translate-x-1/2 ${bgClass} text-white px-8 py-4 rounded-2xl shadow-2xl z-[9999] font-black modal-animate flex items-center gap-3 text-lg`;
            alertBox.innerHTML = `<i data-lucide="${type === 'success' ? 'check-circle' : 'alert-circle'}" size="28"></i> ${msg}`;
            document.body.appendChild(alertBox);
            lucide.createIcons();
            setTimeout(() => { alertBox.style.opacity = '0'; setTimeout(() => alertBox.remove(), 400); }, 2000);
        }

        function updateProblem() {
            const newDiv = parseInt(document.getElementById('input-dividend').value) || 1;
            const newDivisor = parseInt(document.getElementById('input-divisor').value) || 1;
            state = {
                ...state, dividend: newDiv, divisor: newDivisor,
                bank: { tens: Math.floor(newDiv / 10), ones: newDiv % 10 },
                piles: Array.from({ length: newDivisor }, () => ({ tens: 0, ones: 0 })),
                isSetting: false, quizStep: 'idle'
            };
            history = []; setState(state);
        }

        function render() {
            const root = document.getElementById('root');
            const bankTotal = state.bank.tens * 10 + state.bank.ones;

            root.innerHTML = `
                <!-- Top Header -->
                <div class="flex items-center justify-between shrink-0">
                    <div class="flex items-center gap-4">
                        <div class="bg-indigo-600 text-white w-10 h-10 rounded-xl flex items-center justify-center shadow-lg"><i data-lucide="help-circle" size="24"></i></div>
                        <h1 class="text-3xl font-black text-slate-800">
                            請將 <span class="text-indigo-600 px-2">${state.dividend}</span> 元平分給 <span class="text-orange-500 px-2">${state.divisor}</span> 個人
                        </h1>
                    </div>
                    <div class="flex gap-2">
                        <button onclick="history.length > 0 && setState(history.pop())" class="flex items-center gap-2 px-4 py-2 bg-white border border-slate-200 rounded-xl text-slate-500 hover:bg-slate-50 shadow-sm transition-all font-bold text-sm"><i data-lucide="undo-2" size="16"></i>上一步</button>
                        <button onclick="setState({isSetting: true})" class="p-2 bg-slate-800 text-white rounded-xl hover:bg-slate-700 shadow-md transition-all"><i data-lucide="settings" size="18"></i></button>
                    </div>
                </div>

                <div class="flex-1 flex gap-6 overflow-hidden min-h-0">
                    <!-- Left: Bank & Exchange (Teacher's Focus) -->
                    <div class="w-80 shrink-0 flex flex-col gap-4">
                        <div class="flex-[3] bg-white rounded-3xl border-4 border-indigo-600 shadow-xl overflow-hidden flex flex-col">
                            <div class="bg-indigo-600 p-4 text-white">
                                <span class="text-sm font-bold opacity-80 tracking-widest block mb-1">我的銀行</span>
                                <div class="flex items-end justify-between border-b border-indigo-400 pb-3 mb-3">
                                    <span class="text-4xl font-black">${bankTotal}</span>
                                    <span class="text-lg font-bold mb-1">元</span>
                                </div>
                                <div class="grid grid-cols-2 gap-2">
                                    <div class="bg-white/10 rounded-xl p-2 text-center">
                                        <div class="text-[10px] font-bold opacity-80">10 元</div>
                                        <div class="text-2xl font-black">${state.bank.tens} <span class="text-xs">個</span></div>
                                    </div>
                                    <div class="bg-white/10 rounded-xl p-2 text-center">
                                        <div class="text-[10px] font-bold opacity-80">1 元</div>
                                        <div class="text-2xl font-black">${state.bank.ones} <span class="text-xs">個</span></div>
                                    </div>
                                </div>
                            </div>
                            
                            <div class="p-3 flex flex-col gap-3 flex-1 overflow-hidden bg-slate-50">
                                <div class="bank-slot">
                                    <div class="flex flex-wrap gap-2 justify-start p-1">
                                        ${Array.from({ length: state.bank.tens }).map(() => `<div onmousedown="handleDragStart(event, 'tens')" ontouchstart="handleDragStart(event, 'tens')" class="coin w-12 h-12 bg-red-500 border-2 border-red-600 rounded-full flex items-center justify-center text-white font-black text-sm shadow-md ring-2 ring-white/20">10</div>`).join('')}
                                    </div>
                                </div>
                                <div class="bank-slot">
                                    <div class="flex flex-wrap gap-2 justify-start p-1">
                                        ${Array.from({ length: state.bank.ones }).map(() => `<div onmousedown="handleDragStart(event, 'ones')" ontouchstart="handleDragStart(event, 'ones')" class="coin w-9 h-9 bg-sky-400 border-2 border-sky-500 rounded-full flex items-center justify-center text-white font-black text-xs shadow-md ring-2 ring-white/20">1</div>`).join('')}
                                    </div>
                                </div>
                            </div>
                        </div>

                        <!-- Exchange Zone -->
                        <div id="exchange-zone" class="drop-target h-20 rounded-2xl bg-amber-50 border-2 border-dashed border-amber-400 flex flex-col items-center justify-center gap-1 group transition-all shrink-0">
                            <div class="flex items-center gap-2">
                                <i data-lucide="refresh-cw" class="text-amber-600" size="18"></i>
                                <span class="text-amber-800 font-black text-sm">換幣區 (把10元拖曳至此換為1元)</span>
                            </div>
                        </div>
                    </div>

                    <!-- Right: Allocation Area (Optimized Height) -->
                    <div class="flex-1 flex flex-col gap-4 overflow-hidden min-h-0">
                        <div class="flex-1 overflow-y-auto pr-2 bg-slate-200/30 rounded-3xl p-4">
                            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-2 gap-3 pb-2">
                                ${state.piles.map((pile, i) => `
                                    <div data-pile-index="${i}" class="drop-target bg-white rounded-2xl p-3 border-2 border-slate-100 shadow-sm flex items-center gap-4 h-24 transition-all">
                                        <div class="shrink-0 flex flex-col items-center justify-center bg-slate-50 w-16 h-full rounded-xl border border-slate-100">
                                            <span class="text-[10px] font-bold text-slate-400">對象</span>
                                            <span class="text-2xl font-black text-slate-700">${i+1}</span>
                                        </div>
                                        <div class="flex-1 flex flex-wrap gap-1.5 content-center h-full overflow-y-auto">
                                            ${Array.from({ length: pile.tens }).map(() => `<div onclick="handleManualBack(${i}, 'tens')" class="w-10 h-10 bg-red-500 border-2 border-red-600 rounded-full flex items-center justify-center text-white font-black text-xs shadow-sm cursor-pointer hover:scale-105 active:scale-90 transition-all shrink-0">10</div>`).join('')}
                                            ${Array.from({ length: pile.ones }).map(() => `<div onclick="handleManualBack(${i}, 'ones')" class="w-8 h-8 bg-sky-400 border-2 border-sky-500 rounded-full flex items-center justify-center text-white font-black text-[10px] shadow-sm cursor-pointer hover:scale-105 active:scale-90 transition-all shrink-0">1</div>`).join('')}
                                        </div>
                                        <div class="shrink-0 bg-indigo-600 px-3 py-1.5 rounded-xl text-white font-black text-base shadow-sm">
                                            ${pile.tens * 10 + pile.ones} <span class="text-[10px]">元</span>
                                        </div>
                                    </div>
                                `).join('')}
                            </div>
                        </div>

                        <!-- Right Bottom Controls -->
                        <div class="shrink-0 flex items-center justify-between bg-white p-4 rounded-2xl shadow-sm border border-slate-100">
                            <div class="flex items-center gap-3">
                                <div class="w-3 h-3 bg-amber-400 rounded-full animate-pulse"></div>
                                <span class="text-sm font-bold text-slate-500">${state.message}</span>
                            </div>
                            <div class="flex gap-3">
                                <button onclick="checkFairness()" class="px-6 py-3 bg-slate-100 text-slate-600 rounded-xl font-black hover:bg-slate-200 transition-all flex items-center gap-2">
                                    <i data-lucide="scale" size="18"></i>檢查平分
                                </button>
                                <button onclick="startQuiz()" class="px-8 py-3 bg-indigo-600 text-white rounded-xl font-black shadow-lg shadow-indigo-100 hover:bg-indigo-700 active:scale-95 transition-all flex items-center gap-2">
                                    <i data-lucide="check-circle-2" size="20"></i>完成分錢
                                </button>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Modals -->
                ${state.quizStep === 'judging' ? `
                    <div class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[5000] flex items-center justify-center p-6">
                        <div class="bg-white p-8 rounded-3xl w-full max-w-sm modal-animate shadow-2xl text-center">
                            <h2 class="text-2xl font-black mb-8 text-slate-800">請問分完了嗎？</h2>
                            <div class="grid grid-cols-2 gap-4">
                                <button onclick="handleJudge('none')" class="p-6 bg-green-50 border-2 border-green-500 rounded-2xl flex flex-col items-center gap-2 hover:bg-green-100 transition-all">
                                    <div class="bg-green-500 p-2 rounded-full text-white"><i data-lucide="check" size="24"></i></div>
                                    <span class="font-black text-green-700">剛好分完</span>
                                </button>
                                <button onclick="handleJudge('more')" class="p-6 bg-orange-50 border-2 border-orange-500 rounded-2xl flex flex-col items-center gap-2 hover:bg-orange-100 transition-all">
                                    <div class="bg-orange-500 p-2 rounded-full text-white"><i data-lucide="help-circle" size="24"></i></div>
                                    <span class="font-black text-orange-700">還有剩下</span>
                                </button>
                            </div>
                        </div>
                    </div>
                ` : ''}

                ${state.quizStep === 'inputtingRemainder' ? `
                    <div class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[5000] flex items-center justify-center p-6">
                        <div class="bg-white p-6 rounded-3xl w-full max-w-[320px] modal-animate shadow-2xl border-4 border-indigo-600">
                            <h2 class="text-xl font-black text-center mb-4 text-slate-700">銀行最後剩下多少元？</h2>
                            <div class="bg-slate-100 py-4 rounded-2xl mb-4 text-center">
                                <span class="text-4xl font-black text-indigo-600">${state.tempRemainder || '0'}</span>
                                <span class="text-xl text-indigo-400 ml-1">元</span>
                            </div>
                            <div class="grid grid-cols-3 gap-2">
                                ${[1,2,3,4,5,6,7,8,9].map(n => `<button onclick="setState({tempRemainder: state.tempRemainder.length < 2 ? state.tempRemainder + ${n} : state.tempRemainder})" class="numpad-btn">${n}</button>`).join('')}
                                <button onclick="setState({tempRemainder: ''})" class="numpad-btn bg-red-50 text-red-400"><i data-lucide="delete" class="mx-auto" size="18"></i></button>
                                <button onclick="setState({tempRemainder: state.tempRemainder.length < 2 ? state.tempRemainder + '0' : state.tempRemainder})" class="numpad-btn">0</button>
                                <button onclick="submitRemainder()" class="numpad-btn bg-indigo-600 text-white font-black">OK</button>
                            </div>
                        </div>
                    </div>
                ` : ''}

                ${state.isSetting ? `
                    <div class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[6000] flex items-center justify-center p-4">
                        <div class="bg-white p-8 rounded-3xl w-full max-w-[360px] border-4 border-slate-800 shadow-2xl">
                            <h2 class="text-2xl font-black text-center mb-6 text-slate-800">重設題目</h2>
                            <div class="space-y-6">
                                <div class="relative">
                                    <label class="absolute -top-2 left-4 bg-white px-2 text-[10px] font-black text-indigo-600 tracking-widest">總金額</label>
                                    <input id="input-dividend" type="number" value="${state.dividend}" class="w-full text-3xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl text-center outline-none focus:border-indigo-400 transition-all">
                                </div>
                                <div class="relative">
                                    <label class="absolute -top-2 left-4 bg-white px-2 text-[10px] font-black text-orange-500 tracking-widest">平分人數</label>
                                    <input id="input-divisor" type="number" value="${state.divisor}" min="1" max="9" class="w-full text-3xl font-black p-4 bg-slate-50 border-2 border-slate-100 rounded-2xl text-center text-orange-500 outline-none focus:border-orange-400 transition-all">
                                </div>
                                <div class="flex gap-2">
                                    <button onclick="setState({isSetting: false})" class="flex-1 py-3 font-bold text-slate-400 hover:text-slate-600">取消</button>
                                    <button onclick="updateProblem()" class="flex-1 py-3 font-black bg-slate-800 text-white rounded-xl shadow-md hover:bg-slate-700">更新設定</button>
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
