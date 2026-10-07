# smc-trading-radar
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SMC + 動態均線趨勢雷達 (日內專用升級版 v3.1)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body { background-color: #0a0a0a; color: #f3f4f6; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
        ::-webkit-scrollbar { width: 4px; height: 4px; }
        ::-webkit-scrollbar-thumb { background: #374151; border-radius: 4px; }
        .status-dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
        .online { background: #10b981; box-shadow: 0 0 8px #10b981; }
        .offline { background: #ef4444; }
        
        /* ⚡ 15m 突襲閃爍亮燈特效 */
        @keyframes pulse-glow {
            0%, 100% { border-color: rgba(16, 185, 129, 0.4); box-shadow: 0 0 5px rgba(16, 185, 129, 0.2); }
            50% { border-color: rgba(16, 185, 129, 1); box-shadow: 0 0 20px rgba(16, 185, 129, 0.6); }
        }
        .signal-flash-green { animation: pulse-glow 1.5s infinite; }
        @keyframes pulse-glow-red {
            0%, 100% { border-color: rgba(239, 68, 68, 0.4); box-shadow: 0 0 5px rgba(239, 68, 68, 0.2); }
            50% { border-color: rgba(239, 68, 68, 1); box-shadow: 0 0 20px rgba(239, 68, 68, 0.6); }
        }
        .signal-flash-red { animation: pulse-glow-red 1.5s infinite; }
    </style>
</head>
<body class="flex justify-center p-3">

<div class="w-full max-w-md space-y-4">
    <!-- 串流與對帳雷達狀態 -->
    <div class="bg-gray-900 border border-purple-500/40 rounded-2xl p-4 shadow-lg space-y-3">
        <div class="flex justify-between items-center">
            <div class="flex items-center gap-2">
                <span id="wsDot" class="status-dot offline"></span>
                <span id="wsText" class="text-[11px] font-bold text-gray-400">WebSocket 串流未連線</span>
            </div>
            <button type="button" onclick="clearTrackerData()" class="text-[10px] text-gray-400 hover:text-red-400 underline cursor-pointer">清除對帳紀錄</button>
        </div>
        
        <div class="grid grid-cols-3 gap-2 text-center pt-1">
            <div class="bg-gray-800/50 p-2 rounded-xl">
                <span class="text-[10px] text-gray-400 block">追蹤總數</span>
                <span id="stat-total" class="font-mono font-bold text-gray-100 text-sm">0</span>
            </div>
            <div class="bg-gray-800/50 p-2 rounded-xl">
                <span class="text-[10px] text-gray-400 block">勝 / 敗</span>
                <span id="stat-win-loss" class="font-mono font-bold text-emerald-400 text-xs">0 / 0</span>
            </div>
            <div class="bg-gray-800/50 p-2 rounded-xl">
                <span class="text-[10px] text-gray-400 block">自動勝率</span>
                <span id="stat-win-rate" class="font-mono font-bold text-purple-400 text-sm">0.0%</span>
            </div>
        </div>

        <div class="pt-2 border-t border-gray-800 space-y-1.5">
            <span class="text-[10px] text-gray-400 block font-semibold">📍 追蹤中與歷史幣種明細：</span>
            <div id="tracker-list-container" class="space-y-1.5 max-h-32 overflow-y-auto pr-1">
                <div class="text-[11px] text-gray-500 text-center py-2">尚無追蹤紀錄</div>
            </div>
        </div>
    </div>

    <!-- 🛡️ BTC 1H EMA 200 動態趨勢防線雷達 -->
    <div class="bg-gray-900 border border-blue-500/40 rounded-2xl p-4 shadow-lg space-y-3">
        <div class="flex justify-between items-center">
            <div class="flex items-center gap-2">
                <span class="status-dot online"></span>
                <span class="text-[11px] font-bold text-blue-400">BTC 1H EMA 200 動態趨勢防線</span>
            </div>
            <span class="text-[10px] bg-blue-500/20 text-blue-300 px-2 py-0.5 rounded-full">⚡ 日內彈性防線</span>
        </div>

        <div class="grid grid-cols-2 gap-2 text-xs">
            <div class="bg-gray-800/60 p-2 rounded-xl border border-gray-700/50">
                <div class="text-[10px] text-gray-400">1H EMA 200 動態生命線</div>
                <div id="instCostPrice" class="text-sm font-bold text-emerald-400 mt-0.5">計算中...</div>
            </div>
            <div class="bg-gray-800/60 p-2 rounded-xl border border-gray-700/50">
                <div class="text-[10px] text-gray-400">BTC 即時現價</div>
                <div id="btcLivePrice" class="text-sm font-bold text-purple-400 mt-0.5">--</div>
            </div>
        </div>

        <div class="p-2.5 bg-blue-950/30 border border-blue-500/30 rounded-xl text-[11px] text-blue-200 flex flex-col gap-1">
            <div class="flex justify-between items-center font-semibold">
                <span>🛡️ 大盤均線過濾狀態：</span>
                <span id="instStatus" class="text-emerald-400">分析中...</span>
            </div>
            <div id="instDesc" class="text-[10px] text-gray-300">
                正在載入 1H 動態 EMA 200，維持 15m 突擊靈活性與大盤風控提醒...
            </div>
        </div>
    </div>

    <!-- 搜尋與模式選擇 -->
    <div class="bg-gray-900 border border-gray-800 rounded-2xl p-4 shadow-lg space-y-3">
        <div class="flex justify-between items-center">
            <h1 class="font-bold text-gray-100 text-sm">🐋 SMC + 動態均線趨勢雷達 v3.1</h1>
            <div class="flex items-center gap-2">
                <button type="button" onclick="manualRetry()" class="text-[10px] text-purple-400 hover:underline cursor-pointer">🔄 重新整理</button>
                <span id="lock-status-badge" class="text-[10px] px-2 py-0.5 bg-emerald-500/20 text-emerald-400 rounded-md font-semibold">🔒 正常</span>
            </div>
        </div>
        
        <div class="flex gap-2">
            <input 
                id="search-input" 
                type="text" 
                placeholder="輸入標的 (例: BTC, ETH, SOL)" 
                class="w-full bg-gray-800 border border-gray-700 rounded-xl px-3 py-2 text-xs text-gray-100 focus:outline-none focus:border-purple-400"
                value="BTC"
            >
            <button type="button" onclick="changeSymbol()" class="bg-purple-600 hover:bg-purple-700 text-white font-bold px-4 py-2 rounded-xl text-xs transition cursor-pointer">切換並分析</button>
        </div>

        <div class="bg-gray-800/60 p-2.5 rounded-xl border border-gray-700 flex justify-between items-center text-xs">
            <span class="text-gray-300 font-bold">⚡ 執行策略模式：</span>
            <div class="flex gap-1 bg-gray-900 p-1 rounded-lg">
                <button type="button" id="mode-limit-btn" onclick="setEntryMode('limit')" class="px-3 py-1 rounded-md font-bold text-[11px] bg-purple-600 text-white transition cursor-pointer">回踩掛單 (Limit)</button>
                <button type="button" id="mode-breakout-btn" onclick="setEntryMode('breakout')" class="px-3 py-1 rounded-md font-bold text-[11px] text-gray-400 hover:text-white transition cursor-pointer">突破追價 (Stop)</button>
            </div>
        </div>
    </div>

    <!-- 🌟 CHoCH 與 Order Block 模組 -->
    <div class="bg-indigo-950/20 border border-indigo-500/40 rounded-2xl p-3 shadow-lg space-y-2">
        <div class="flex justify-between items-center">
            <h3 class="font-bold text-indigo-400 text-xs">⚡ 1H CHoCH (結構轉變) & Order Block (OB)</h3>
            <span id="choch-status-badge" class="text-[9px] bg-indigo-500/20 text-indigo-300 px-2 py-0.5 rounded-full">結構掃描中</span>
        </div>
        <div class="grid grid-cols-2 gap-2 text-xs">
            <div class="bg-gray-800/50 p-2 rounded-xl border border-gray-700/50">
                <span class="text-[10px] text-gray-400 block">CHoCH 狀態 (1H 趨勢)</span>
                <span id="choch-val" class="font-mono font-bold text-yellow-400 text-xs">分析中...</span>
            </div>
            <div class="bg-gray-800/50 p-2 rounded-xl border border-gray-700/50">
                <span class="text-[10px] text-gray-400 block">Order Block (機構訂單塊)</span>
                <span id="ob-val" class="font-mono font-bold text-orange-400 text-xs">計算中...</span>
            </div>
        </div>
        <div id="choch-desc" class="text-[10px] text-gray-300 px-1">
            正在比對 1H 最近高低點突破，判定市況是否發生 Change of Character (CHoCH) 翻轉...
        </div>
    </div>

    <!-- 🌟 FVG 公允價值缺口模組 -->
    <div class="bg-cyan-950/20 border border-cyan-500/40 rounded-2xl p-3 shadow-lg space-y-2">
        <div class="flex justify-between items-center">
            <h3 class="font-bold text-cyan-400 text-xs">📐 1H FVG (公允價值缺口) 機構失衡區間</h3>
            <span id="fvg-status-badge" class="text-[9px] bg-cyan-500/20 text-cyan-300 px-2 py-0.5 rounded-full">動態掃描中</span>
        </div>
        <div id="fvg-container" class="grid grid-cols-2 gap-2 text-xs">
            <div class="bg-gray-800/50 p-2 rounded-xl border border-gray-700/50">
                <span class="text-[10px] text-gray-400 block">多方失衡區 (Bullish FVG)</span>
                <span id="bullish-fvg-val" class="font-mono font-bold text-emerald-400 text-xs">計算中...</span>
            </div>
            <div class="bg-gray-800/50 p-2 rounded-xl border border-gray-700/50">
                <span class="text-[10px] text-gray-400 block">空方失衡區 (Bearish FVG)</span>
                <span id="bearish-fvg-val" class="font-mono font-bold text-red-400 text-xs">計算中...</span>
            </div>
        </div>
        <div id="fvg-desc" class="text-[10px] text-gray-300 px-1">
            正在透過 1H K 線過濾動能，尋找機構訂單塊未回踩的缺口區間...
        </div>
    </div>

    <!-- 主力複合式吸籌與合約過濾雷達 -->
    <div class="bg-purple-950/20 border border-purple-500/40 rounded-2xl p-3 shadow-lg space-y-2">
        <div class="flex justify-between items-center">
            <h3 class="font-bold text-purple-400 text-xs">🕵️‍♂️ 主力複合式吸籌雷達 (15分OI並行加速掃描)</h3>
            <span class="text-[9px] text-gray-400">高效平行運算</span>
        </div>
        <div id="stealth-accumulation-box" class="space-y-1.5 text-xs">
            <div class="text-gray-400 text-[11px] py-1 text-center">正在背景平行比對 15分 OI、SMC 訂單塊與資金費率...</div>
        </div>
    </div>

    <!-- 資金費率與 RSI -->
    <div class="grid grid-cols-2 gap-3">
        <div class="bg-gray-900 border border-gray-800 rounded-2xl p-3 shadow-lg space-y-1.5">
            <span class="text-[10px] text-gray-400 block">💰 8h 資金費率 (Funding Rate)</span>
            <div class="flex justify-between items-baseline">
                <span id="funding-val" class="font-mono font-bold text-base text-yellow-400">0.0100%</span>
                <span id="funding-status" class="text-[10px] text-gray-400">正常</span>
            </div>
        </div>
        <div class="bg-gray-900 border border-gray-800 rounded-2xl p-3 shadow-lg space-y-1.5">
            <span class="text-[10px] text-gray-400 block">📊 1H RSI 強弱</span>
            <div class="flex justify-between items-baseline">
                <span id="rsi-val" class="font-mono font-bold text-base text-purple-400">--</span>
                <span id="rsi-status" class="text-[10px] text-gray-300">計算中</span>
            </div>
        </div>
    </div>

    <!-- 漲跌幅排行榜 -->
    <div class="grid grid-cols-2 gap-3">
        <div class="bg-gray-900 border border-emerald-900/40 rounded-2xl p-3 shadow-lg space-y-2">
            <h3 class="font-bold text-emerald-400 text-xs">🟢 全市場多頭漲幅榜</h3>
            <div id="best-longs" class="space-y-1.5 text-xs"><div class="text-gray-500 text-[11px]">全市場掃描中...</div></div>
        </div>
        <div class="bg-gray-900 border border-red-900/40 rounded-2xl p-3 shadow-lg space-y-2">
            <h3 class="font-bold text-red-400 text-xs">🔴 全市場空頭跌幅榜</h3>
            <div id="best-shorts" class="space-y-1.5 text-xs"><div class="text-gray-500 text-[11px]">全市場掃描中...</div></div>
        </div>
    </div>

    <!-- 目標幣種分析卡片 -->
    <div id="target-card" class="bg-gray-900 border border-gray-800 rounded-2xl p-4 shadow-lg space-y-3 transition-all">
        <div class="flex justify-between items-center">
            <div>
                <div class="flex items-center gap-2">
                    <span id="coin-name" class="text-base font-bold text-purple-400">BTCUSDT</span>
                    <span class="text-[9px] bg-purple-500/20 text-purple-300 px-1.5 py-0.5 rounded font-mono">1H 主趨勢</span>
                    <span id="sub-tf-badge" class="text-[9px] bg-gray-800 text-gray-400 px-1.5 py-0.5 rounded font-mono">15m 待命</span>
                </div>
                <div id="coin-change" class="text-[11px] text-gray-400 mt-0.5">24h 漲跌: --</div>
            </div>
            <div class="text-right">
                <div id="coin-price" class="font-mono text-base font-bold text-emerald-400">--</div>
                <div class="text-[10px] text-gray-400">Bybit 串流現價</div>
            </div>
        </div>

        <div id="oi-box-container" class="bg-purple-950/30 p-3 rounded-xl border border-purple-500/40 space-y-1.5 text-xs transition-all">
            <div class="flex justify-between items-center">
                <span id="oi-title-label" class="text-purple-400 font-bold">⚡ 15分鐘內 OI 資金進場/撤離雷達</span>
                <span id="oi-change-val" class="font-mono text-[11px] text-gray-300">15m OI 載入中...</span>
            </div>
            <p id="oi-quadrant-desc" class="text-gray-300 text-[11px] leading-relaxed">正在解析最近 15 分鐘合約未平倉量增減與方向建議...</p>
        </div>

        <div class="p-3 rounded-xl border flex justify-between items-center" id="direction-box">
            <div>
                <span class="text-[10px] text-gray-400 block">1H SMC + 15m 資金雙軌建議</span>
                <span id="rec-direction" class="font-extrabold text-sm">--</span>
            </div>
            <div class="text-right">
                <span class="text-[10px] text-gray-400 block">預估動態盈虧比</span>
                <span id="rec-rr-ratio" class="font-bold text-xs text-emerald-400 font-mono">1 : --</span>
            </div>
        </div>

        <!-- 智能風控與動態開倉計算 (含快速增減按鈕) -->
        <div class="bg-purple-950/20 p-3 rounded-xl border border-purple-500/40 space-y-2 text-xs">
            <div class="flex justify-between items-center">
                <span class="text-purple-400 font-bold">🧮 智能風控與動態開倉計算 (ATR防插針)</span>
                <span class="text-[10px] text-gray-400">防爆倉保本機制</span>
            </div>
            <div class="flex items-center justify-between gap-2">
                <span class="text-gray-300 text-[11px]">容許最大虧損 (USDT)：</span>
                <div class="flex items-center gap-1">
                    <button type="button" onclick="adjustRisk(-1)" class="bg-gray-800 hover:bg-gray-700 text-gray-300 w-6 h-6 rounded-lg text-xs font-bold transition cursor-pointer flex items-center justify-center">-</button>
                    <input 
                        id="risk-loss-input" 
                        type="number" 
                        value="3" 
                        min="1" 
                        oninput="calculatePositionSize()" 
                        class="w-16 bg-gray-800 border border-gray-700 rounded-lg px-2 py-1 text-xs text-purple-400 font-mono font-bold text-center focus:outline-none focus:border-purple-400"
                    >
                    <button type="button" onclick="adjustRisk(1)" class="bg-gray-800 hover:bg-gray-700 text-gray-300 w-6 h-6 rounded-lg text-xs font-bold transition cursor-pointer flex items-center justify-center">+</button>
                </div>
            </div>
            <div class="grid grid-cols-3 gap-2 pt-1 border-t border-purple-900/30">
                <div class="bg-gray-800/50 p-2 rounded-lg">
                    <span class="text-[10px] text-gray-400 block">建議開倉量</span>
                    <span id="calc-qty" class="font-mono font-bold text-gray-100 text-xs">--</span>
                </div>
                <div class="bg-gray-800/50 p-2 rounded-lg">
                    <span class="text-[10px] text-gray-400 block">合約總價值</span>
                    <span id="calc-notional" class="font-mono font-bold text-purple-400 text-xs">--</span>
                </div>
                <div class="bg-gray-800/50 p-2 rounded-lg">
                    <span class="text-[10px] text-gray-400 block">建議槓桿 / 估算強平</span>
                    <span id="calc-leverage" class="font-mono font-bold text-yellow-400 text-xs">--</span>
                </div>
            </div>
            <div class="text-[10px] text-gray-400 pt-1 flex justify-between">
                <span>🛡️ 預估強平價格 (Liq. Price)：</span>
                <span id="calc-liq-price" class="font-mono font-bold text-red-400">--</span>
            </div>
        </div>

        <!-- 點位區塊 -->
        <div class="space-y-2 text-xs pt-1">
            <div class="p-3 bg-purple-950/30 rounded-xl border border-purple-500/40 space-y-2">
                <div class="flex justify-between items-center border-b border-purple-900/40 pb-1.5">
                    <span id="entry-mode-label" class="text-purple-400 font-bold">🎯 建議進場點</span>
                    <span id="rec-price" class="font-mono font-bold text-purple-400 text-sm">--</span>
                </div>
                <div class="flex justify-between items-center text-[11px]">
                    <span class="text-red-400">ATR 動態防插針止損 (SL)</span>
                    <span id="rec-sl" class="font-mono font-semibold text-red-400">--</span>
                </div>
            </div>

            <div class="grid grid-cols-2 gap-2 pt-1">
                <div class="p-2.5 bg-emerald-950/20 rounded-xl border border-emerald-900/40 space-y-1">
                    <div class="flex justify-between items-center">
                        <span class="text-[10px] text-emerald-400 font-bold">🛡️ TP1 (平50% + 保本)</span>
                        <span class="text-[9px] text-gray-400">1:1.0</span>
                    </div>
                    <span id="rec-tp1" class="font-mono font-bold text-emerald-400 text-xs block">--</span>
                </div>
                <div class="p-2.5 bg-emerald-950/40 rounded-xl border border-emerald-500/50 space-y-1">
                    <div class="flex justify-between items-center">
                        <span class="text-[10px] text-emerald-300 font-bold">🚀 TP2 (動態結構終點)</span>
                        <span id="tp2-rr-badge" class="text-[9px] text-purple-300 font-bold">1:--</span>
                    </div>
                    <span id="rec-tp2" class="font-mono font-bold text-emerald-300 text-xs block">--</span>
                </div>
            </div>

            <button type="button" onclick="addToWinTracker()" class="w-full py-2.5 bg-emerald-600 hover:bg-emerald-700 text-white font-bold rounded-xl text-xs transition flex items-center justify-center gap-1 shadow-lg shadow-emerald-600/20 cursor-pointer">
                🎯 納入勝率追蹤並推送 Discord
            </button>

            <div class="grid grid-cols-2 gap-2 pt-1">
                <button type="button" onclick="copyStrategy()" class="w-full py-2.5 bg-gray-800 hover:bg-gray-700 text-gray-200 font-bold rounded-xl text-xs transition border border-gray-700 cursor-pointer">
                    📋 複製操盤數據
                </button>
                <button type="button" onclick="quickAction()" class="w-full py-2.5 bg-purple-600 hover:bg-purple-700 text-white font-bold rounded-xl text-xs transition shadow-lg shadow-purple-600/20 cursor-pointer">
                    ⚡ 一鍵複製下單點位
                </button>
            </div>
        </div>
    </div>
</div>

<script>
const BUILTIN_WEBHOOK = "https://discord.com/api/webhooks/1543187656349974550/aBfFI8JELaPV-Z64u8Y7Q7rN6L15UZzjg4Z2fjxNd1EQItrEdok1a8WfbfzDgU70Zvkp";

let currentSymbol = "BTCUSDT";
let currentEntryMode = "limit";
let ws = null;
let tickersMap = {};
let currentStrategyData = {};
let bestKData = null;
let lastUiUpdate = 0;
let lastBtcStatus = null;
let isBtcBullishMarket = true; 

function init() {
    updateTrackerUI();
    initWebSocket();
    fetchBtcTrendDefense();
    fetchSymbolData(currentSymbol);
    fetchAllMarketTickers();
    setInterval(fetchAllMarketTickers, 30000);
    setInterval(fetchBtcTrendDefense, 60000);
    setInterval(runAutoAudit, 15000);
}

// 🛡️ 1H EMA 200 動態防線
async function fetchBtcTrendDefense() {
    try {
        let json = await fetchWithRetry('https://api.bybit.com/v5/market/kline?category=linear&symbol=BTCUSDT&interval=60&limit=210');
        if (json.retCode === 0 && json.result && json.result.list) {
            let klines = json.result.list.reverse();
            let closes = klines.map(k => parseFloat(k[4]));
            let currentBtcPrice = closes[closes.length - 1];

            let ema200 = calculateEMA(closes, 200);
            let isAbove = currentBtcPrice >= ema200;
            isBtcBullishMarket = isAbove;

            document.getElementById('instCostPrice').innerText = `$${ema200.toFixed(2)}`;
            document.getElementById('btcLivePrice').innerText = `$${currentBtcPrice.toLocaleString()}`;

            let statusEl = document.getElementById('instStatus');
            let descEl = document.getElementById('instDesc');
            let currentStatusText = isAbove ? "above" : "below";

            if (lastBtcStatus !== null && lastBtcStatus !== currentStatusText) {
                if (isAbove) {
                    sendToDiscord(`🛡️ **【BTC 動態防線戰報：收復 1H EMA 200】**\n即時價格：$${currentBtcPrice.toLocaleString()}\n動態生命線：$${ema200.toFixed(2)}\n👉 *山寨幣恢復多向開倉*`);
                } else {
                    sendToDiscord(`🚨 **【BTC 動態防線失守警報：跌破 1H EMA 200】**\n即時價格：$${currentBtcPrice.toLocaleString()}\n動態生命線：$${ema200.toFixed(2)}\n⚠️ *大盤偏空，留意短線逆勢風險*`);
                }
            }
            lastBtcStatus = currentStatusText;

            if (isAbove) {
                statusEl.className = "text-emerald-400";
                statusEl.innerText = "站穩 EMA 200 上方 (多頭優勢)";
                descEl.innerText = `現價高於 1H 動態生命線，多單順勢執行。`;
            } else {
                statusEl.className = "text-yellow-400";
                statusEl.innerText = "跌破 EMA 200 下方 (日內彈性風控)";
                descEl.innerText = `現價低於 1H 生命線，維持短線突擊，請嚴格執行 ATR 止損。`;
            }
        }
    } catch (e) {}
}

function calculateEMA(closes, period) {
    let k = 2 / (period + 1);
    let sum = 0;
    for (let i = 0; i < period; i++) sum += closes[i];
    let prevEma = sum / period;
    for (let i = period; i < closes.length; i++) {
        prevEma = (closes[i] - prevEma) * k + prevEma;
    }
    return prevEma;
}

function setEntryMode(mode) {
    currentEntryMode = mode;
    let limitBtn = document.getElementById('mode-limit-btn');
    let breakoutBtn = document.getElementById('mode-breakout-btn');
    
    if (mode === 'limit') {
        limitBtn.className = "px-3 py-1 rounded-md font-bold text-[11px] bg-purple-600 text-white transition cursor-pointer";
        breakoutBtn.className = "px-3 py-1 rounded-md font-bold text-[11px] text-gray-400 hover:text-white transition cursor-pointer";
    } else {
        breakoutBtn.className = "px-3 py-1 rounded-md font-bold text-[11px] bg-purple-600 text-white transition cursor-pointer";
        limitBtn.className = "px-3 py-1 rounded-md font-bold text-[11px] text-gray-400 hover:text-white transition cursor-pointer";
    }
    if (bestKData) renderStrategyUI(bestKData);
}

// 🚀 2. 快捷調整風險金額按鈕函數
function adjustRisk(amount) {
    let input = document.getElementById('risk-loss-input');
    let val = parseFloat(input.value) || 3;
    val = Math.max(1, val + amount);
    input.value = val;
    calculatePositionSize();
}

async function fetchWithRetry(url, retries = 3, timeout = 6000) {
    for (let i = 0; i < retries; i++) {
        try {
            let controller = new AbortController();
            let timer = setTimeout(() => controller.abort(), timeout);
            let response = await fetch(url, { signal: controller.signal });
            clearTimeout(timer);
            if (!response.ok) throw new Error('HTTP error! status: ' + response.status);
            return await response.json();
        } catch (e) {
            if (i === retries - 1) throw e;
            await new Promise(res => setTimeout(res, 1000 * (i + 1)));
        }
    }
}

function initWebSocket() {
    ws = new WebSocket('wss://stream.bybit.com/v5/public/linear');
    const dot = document.getElementById('wsDot');
    const text = document.getElementById('wsText');

    ws.onopen = () => {
        if(dot) dot.className = 'status-dot online';
        if(text) text.textContent = 'WS 毫秒串流已連線';
        ws.send(JSON.stringify({ op: 'subscribe', args: ['tickers.' + currentSymbol, 'tickers.BTCUSDT'] }));
    };

    ws.onmessage = (e) => {
        let msg = JSON.parse(e.data);
        if (msg.topic && msg.topic.startsWith('tickers.')) {
            let data = msg.data;
            let sym = data.symbol;
            
            if (!tickersMap[sym]) tickersMap[sym] = { symbol: sym, price: 0, change: 0, fundingRate: 0 };
            if (data.lastPrice) tickersMap[sym].price = parseFloat(data.lastPrice);
            if (data.price24hPcnt) tickersMap[sym].change = parseFloat(data.price24hPcnt) * 100;
            if (data.fundingRate) tickersMap[sym].fundingRate = parseFloat(data.fundingRate) * 100;

            let now = Date.now();
            if (now - lastUiUpdate > 300) {
                lastUiUpdate = now;
                if (sym === currentSymbol && tickersMap[currentSymbol].price > 0) {
                    let cur = tickersMap[currentSymbol];
                    updateLivePrice(cur.price, cur.change, cur.fundingRate);
                }
            }
        }
    };

    ws.onclose = () => {
        if(dot) dot.className = 'status-dot offline';
        if(text) text.textContent = 'WS 斷線重連中...';
        setTimeout(initWebSocket, 3000);
    };
}

async function fetchAllMarketTickers() {
    try {
        let json = await fetchWithRetry('https://api.bybit.com/v5/market/tickers?category=linear');
        if (json.retCode === 0 && json.result && json.result.list) {
            let list = json.result.list.filter(item => item.symbol.endsWith('USDT'));
            list.forEach(item => {
                let sym = item.symbol;
                if (!tickersMap[sym]) tickersMap[sym] = {};
                tickersMap[sym].symbol = sym;
                tickersMap[sym].price = parseFloat(item.lastPrice || 0);
                tickersMap[sym].change = parseFloat(item.price24hPcnt || 0) * 100;
                tickersMap[sym].fundingRate = parseFloat(item.fundingRate || 0.01) * 100;
            });
            renderAllMarketHotlist(list);
        }
    } catch (e) {}
}

function renderAllMarketHotlist(list) {
    let validList = list.filter(t => parseFloat(t.lastPrice) > 0);
    if (validList.length === 0) return;

    let longs = validList.filter(t => parseFloat(t.price24hPcnt) > 0).sort((a, b) => parseFloat(b.price24hPcnt) - parseFloat(a.price24hPcnt)).slice(0, 3);
    let shorts = validList.filter(t => parseFloat(t.price24hPcnt) < 0).sort((a, b) => parseFloat(a.price24hPcnt) - parseFloat(b.price24hPcnt)).slice(0, 3);

    let longEl = document.getElementById('best-longs');
    let shortEl = document.getElementById('best-shorts');

    if(longEl) {
        longEl.innerHTML = longs.map(item => `
            <div onclick="quickSelect('${item.symbol}')" class="flex justify-between items-center p-1.5 bg-gray-800/40 hover:bg-gray-800 rounded-lg cursor-pointer transition">
                <span class="font-bold text-gray-200 text-[11px]">${item.symbol.replace('USDT', '')}</span>
                <span class="font-mono font-bold text-emerald-400 text-[11px]">+${(parseFloat(item.price24hPcnt)*100).toFixed(2)}%</span>
            </div>
        `).join('');
    }

    if(shortEl) {
        shortEl.innerHTML = shorts.map(item => `
            <div onclick="quickSelect('${item.symbol}')" class="flex justify-between items-center p-1.5 bg-gray-800/40 hover:bg-gray-800 rounded-lg cursor-pointer transition">
                <span class="font-bold text-gray-200 text-[11px]">${item.symbol.replace('USDT', '')}</span>
                <span class="font-mono font-bold text-red-400 text-[11px]">${(parseFloat(item.price24hPcnt)*100).toFixed(2)}%</span>
            </div>
        `).join('');
    }

    run15mCompositeAccumulationScan(validList);
}

async function run15mCompositeAccumulationScan(allTickers) {
    let container = document.getElementById('stealth-accumulation-box');
    let topVolumeCoins = allTickers.sort((a, b) => parseFloat(b.turnover24h || 0) - parseFloat(a.turnover24h || 0)).slice(0, 15);
    
    let promises = topVolumeCoins.map(async (item) => {
        let sym = item.symbol;
        try {
            let json = await fetchWithRetry('https://api.bybit.com/v5/market/open-interest?category=linear&symbol=' + sym + '&intervalTime=15min&limit=2', 2, 3000);
            if (json.retCode === 0 && json.result && json.result.list && json.result.list.length >= 2) {
                let curOI = parseFloat(json.result.list[0].openInterest);
                let prevOI = parseFloat(json.result.list[1].openInterest);
                let oiDiffPct = ((curOI - prevOI) / prevOI) * 100;
                let tInfo = tickersMap[sym];
                let priceChange15m = tInfo ? tInfo.change : 0;

                if (sym === currentSymbol) {
                    let targetCard = document.getElementById('target-card');
                    let subTfBadge = document.getElementById('sub-tf-badge');
                    if (oiDiffPct >= 0.4) {
                        targetCard.className = "bg-gray-900 border rounded-2xl p-4 shadow-lg space-y-3 transition-all signal-flash-green";
                        subTfBadge.className = "text-[9px] bg-emerald-500/20 text-emerald-300 px-1.5 py-0.5 rounded font-mono font-bold";
                        subTfBadge.textContent = "⚡ 15m 資金突襲 (+" + oiDiffPct.toFixed(2) + "%)";
                    } else if (oiDiffPct <= -0.4) {
                        targetCard.className = "bg-gray-900 border rounded-2xl p-4 shadow-lg space-y-3 transition-all signal-flash-red";
                        subTfBadge.className = "text-[9px] bg-red-500/20 text-red-300 px-1.5 py-0.5 rounded font-mono font-bold";
                        subTfBadge.textContent = "🚨 15m 資金撤離 (" + oiDiffPct.toFixed(2) + "%)";
                    } else {
                        targetCard.className = "bg-gray-900 border border-gray-800 rounded-2xl p-4 shadow-lg space-y-3 transition-all";
                        subTfBadge.className = "text-[9px] bg-gray-800 text-gray-400 px-1.5 py-0.5 rounded font-mono";
                        subTfBadge.textContent = "15m 平穩";
                    }
                }

                if (oiDiffPct >= 0.4) {
                    let actionType = priceChange15m >= 0 ? "🟢 資金湧入建多 (做多)" : "🔴 資金湧入打壓 (做空)";
                    return { symbol: sym, oiDiff: oiDiffPct, type: actionType, isEntry: true };
                } else if (oiDiffPct <= -0.4) {
                    let actionType = priceChange15m >= 0 ? "⚠️ 空頭停損撤離 (短多)" : "🚨 多殺多資金逃跑 (做空/避險)";
                    return { symbol: sym, oiDiff: oiDiffPct, type: actionType, isEntry: false };
                }
            }
        } catch (e) {}
        return null;
    });

    let results = await Promise.all(promises);
    let detectedSignals = results.filter(r => r !== null);

    if (detectedSignals.length === 0) {
        container.innerHTML = '<div class="text-gray-400 text-[11px] py-1 text-center">最近15分鐘無劇烈資金進場或撤離，持續雷達掃描中...</div>';
    } else {
        container.innerHTML = detectedSignals.map(item => `
            <div onclick="quickSelect('${item.symbol}')" class="flex justify-between items-center p-1.5 bg-gray-800/60 hover:bg-gray-800 rounded-lg cursor-pointer transition">
                <span class="font-bold text-purple-300 text-[11px]">${item.isEntry ? '⚡' : '⚠️'} ${item.symbol.replace('USDT', '')} (${item.type})</span>
                <span class="font-mono ${item.isEntry ? 'text-emerald-400' : 'text-red-400'} text-[11px]">15m OI: ${item.oiDiff > 0 ? '+' : ''}${item.oiDiff.toFixed(2)}%</span>
            </div>
        `).join('');
    }
}

function getDecimalPlaces(price) {
    if (price >= 100) return 2;
    if (price >= 1) return 4;
    return 6;
}

async function fetchSymbolData(symbol) {
    document.getElementById('coin-name').textContent = symbol;
    try {
        let checkRes = await fetchWithRetry(`https://api.bybit.com/v5/market/tickers?category=linear&symbol=${symbol}`);
        if (checkRes.retCode !== 0 || !checkRes.result || checkRes.result.list.length === 0) {
            alert(`⚠️ 找不到標的 [${symbol}]，請確認代碼是否正確！已自動切回 BTCUSDT。`);
            currentSymbol = "BTCUSDT";
            document.getElementById('search-input').value = "BTC";
            symbol = "BTCUSDT";
            document.getElementById('coin-name').textContent = symbol;
        }
    } catch (e) {}

    await Promise.all([fetchKlineData(symbol), fetch15mOpenInterestData(symbol)]);
}

async function manualRetry() {
    fetchBtcTrendDefense();
    fetchSymbolData(currentSymbol);
    fetchAllMarketTickers();
    alert("🔄 1H 動態 EMA 防線與 15m 資金雷達已重新整理");
}

async function fetch15mOpenInterestData(symbol) {
    try {
        let json = await fetchWithRetry('https://api.bybit.com/v5/market/open-interest?category=linear&symbol=' + symbol + '&intervalTime=15min&limit=2');
        if (json.retCode === 0 && json.result && json.result.list && json.result.list.length >= 2) {
            let list = json.result.list;
            let curOI = parseFloat(list[0].openInterest);
            let prevOI = parseFloat(list[1].openInterest);
            let oiDiffPct = ((curOI - prevOI) / prevOI) * 100;
            
            document.getElementById('oi-change-val').textContent = '15m OI 增減: ' + (oiDiffPct >= 0 ? '+' : '') + oiDiffPct.toFixed(2) + '%';
            
            let desc = document.getElementById('oi-quadrant-desc');
            if (oiDiffPct >= 0.4) {
                document.getElementById('oi-title-label').textContent = "🟢 [資金湧入] 15分鐘內大量訂單進場";
                desc.textContent = `過去 15 分鐘內未平倉量大增 (+${oiDiffPct.toFixed(2)}%)！代表機構正在積極開倉。`;
            } else if (oiDiffPct <= -0.4) {
                document.getElementById('oi-title-label').textContent = "🔴 [資金撤離] 15分鐘內大量部位平倉";
                desc.textContent = `過去 15 分鐘內未平倉量大減 (${oiDiffPct.toFixed(2)}%)！市場正在停損或撤離。`;
            } else {
                document.getElementById('oi-title-label').textContent = "⚡ 15分鐘內 OI 資金進場/撤離雷達";
                desc.textContent = `最近 15 分鐘資金增減幅度平緩 (${oiDiffPct.toFixed(2)}%)。`;
            }
        }
    } catch (e) {
        document.getElementById('oi-change-val').textContent = '15m OI: --';
    }
}

async function fetchKlineData(symbol) {
    try {
        let json = await fetchWithRetry('https://api.bybit.com/v5/market/kline?category=linear&symbol=' + symbol + '&interval=60&limit=50');
        if (json.retCode === 0 && json.result && json.result.list) {
            let klines = json.result.list.reverse();
            let highs = klines.map(k => parseFloat(k[2]));
            let lows = klines.map(k => parseFloat(k[3]));
            let closes = klines.map(k => parseFloat(k[4]));

            let curPrice = closes[closes.length - 1];
            let decimals = getDecimalPlaces(curPrice);
            let priceChangePct = ((closes[closes.length-1] - closes[closes.length-2]) / closes[closes.length-2]) * 100;

            calculateAndRenderFVG(klines, decimals);
            calculateAndRenderCHoCHAndOB(klines, decimals, curPrice);

            let rsiVal = calculateRSI(closes, 14);
            let atrVal = calculateATR(highs, lows, closes, 14) || curPrice * 0.005;
            let resVal = Math.max(...highs.slice(-15));
            let supVal = Math.min(...lows.slice(-15));

            bestKData = { symbol, rsi: rsiVal, atr: atrVal, swingHigh: resVal, swingLow: supVal, priceChangePct, decimals, curPrice, highs, lows, closes };
            renderStrategyUI(bestKData);
        }
    } catch (e) {}
}

function calculateAndRenderFVG(klines, decimals) {
    let bullishFvgText = "未偵測有效區間";
    let bearishFvgText = "未偵測有效區間";
    let foundBullish = false;
    let foundBearish = false;

    for (let i = klines.length - 2; i >= 2; i--) {
        let k1 = klines[i - 1];
        let k3 = klines[i + 1];
        let h1 = parseFloat(k1[2]), l1 = parseFloat(k1[3]);
        let h3 = parseFloat(k3[2]), l3 = parseFloat(k3[3]);

        if (!foundBullish && h1 < l3) {
            bullishFvgText = `$${h1.toFixed(decimals)} ~ $${l3.toFixed(decimals)}`;
            foundBullish = true;
        }
        if (!foundBearish && l1 > h3) {
            bearishFvgText = `$${h3.toFixed(decimals)} ~ $${l1.toFixed(decimals)}`;
            foundBearish = true;
        }
        if (foundBullish && foundBearish) break;
    }

    document.getElementById('bullish-fvg-val').textContent = bullishFvgText;
    document.getElementById('bearish-fvg-val').textContent = bearishFvgText;
    document.getElementById('fvg-status-badge').textContent = (foundBullish || foundBearish) ? "1H 機構缺口已鎖定" : "盤整無顯著缺口";
}

function calculateAndRenderCHoCHAndOB(klines, decimals, curPrice) {
    let recentHighs = [];
    let recentLows = [];

    for (let i = 5; i < klines.length - 2; i++) {
        let h = parseFloat(klines[i][2]);
        let l = parseFloat(klines[i][3]);
        if (h >= parseFloat(klines[i-1][2]) && h >= parseFloat(klines[i+1][2])) recentHighs.push(h);
        if (l <= parseFloat(klines[i-1][3]) && l <= parseFloat(klines[i+1][3])) recentLows.push(l);
    }

    let lastHigh = recentHighs.length > 0 ? recentHighs[recentHighs.length - 1] : parseFloat(klines[klines.length - 10][2]);
    let lastLow = recentLows.length > 0 ? recentLows[recentLows.length - 1] : parseFloat(klines[klines.length - 10][3]);

    let chochBadge = document.getElementById('choch-status-badge');
    let chochValEl = document.getElementById('choch-val');
    let descEl = document.getElementById('choch-desc');
    let obValEl = document.getElementById('ob-val');

    if (curPrice > lastHigh) {
        chochValEl.className = "font-mono font-bold text-emerald-400 text-xs";
        chochValEl.textContent = "1H Bullish CHoCH 成立";
        chochBadge.className = "text-[9px] bg-emerald-500/20 text-emerald-300 px-2 py-0.5 rounded-full";
        descEl.textContent = `1H 價格向上突破關鍵結構高點 ($${lastHigh.toFixed(decimals)})。`;
    } else if (curPrice < lastLow) {
        chochValEl.className = "font-mono font-bold text-red-400 text-xs";
        chochValEl.textContent = "1H Bearish CHoCH 成立";
        chochBadge.className = "text-[9px] bg-red-500/20 text-red-300 px-2 py-0.5 rounded-full";
        descEl.textContent = `1H 價格向下跌破關鍵結構低點 ($${lastLow.toFixed(decimals)})，空頭結構確認！`;
    } else {
        chochValEl.className = "font-mono font-bold text-yellow-400 text-xs";
        chochValEl.textContent = "1H 箱體震盪";
        chochBadge.className = "text-[9px] bg-yellow-500/20 text-yellow-300 px-2 py-0.5 rounded-full";
        descEl.textContent = `1H 價格處於震盪區間。`;
    }

    let obText = "未偵測有效 OB";
    for (let i = klines.length - 3; i >= 5; i--) {
        let op = parseFloat(klines[i][1]);
        let cl = parseFloat(klines[i][4]);
        let nextCl = parseFloat(klines[i+1][4]);
        let nextOp = parseFloat(klines[i+1][1]);

        if (cl < op && nextCl > nextOp) {
            obText = `$${cl.toFixed(decimals)} ~ $${op.toFixed(decimals)} (1H 多頭OB)`;
            break;
        } else if (cl > op && nextCl < nextOp) {
            obText = `$${op.toFixed(decimals)} ~ $${cl.toFixed(decimals)} (1H 空頭OB)`;
            break;
        }
    }
    obValEl.textContent = obText;
}

function renderStrategyUI(data) {
    let { symbol, rsi, atr, swingHigh, swingLow, decimals, curPrice, closes } = data;
    
    let isShort = priceChangeTrend(data) < 0;

    let entryPrice, slPrice;
    let atrBuffer = atr * 1.5;

    if (currentEntryMode === 'limit') {
        entryPrice = isShort ? (swingHigh * 0.998) : (swingLow * 1.002);
        slPrice = isShort ? (entryPrice + atrBuffer) : (entryPrice - atrBuffer);
    } else {
        entryPrice = isShort ? (swingLow * 0.998) : (swingHigh * 1.002);
        slPrice = isShort ? (entryPrice + atrBuffer) : (entryPrice - atrBuffer);
    }

    let risk = Math.abs(entryPrice - slPrice) || curPrice * 0.002;
    let tp1Price = isShort ? (entryPrice - risk) : (entryPrice + risk);
    let tp2Price = isShort ? (swingLow * 0.99) : (swingHigh * 1.01);
    let reward = Math.abs(tp2Price - entryPrice);
    let rrRatio = risk > 0 ? (reward / risk).toFixed(1) : "2.0";

    document.getElementById('rsi-val').textContent = rsi.toFixed(1);
    document.getElementById('rec-direction').textContent = isShort ? "🔻 建議做空 (SHORT)" : "🟢 建議做多 (LONG)";

    // 🚀 3. R:R 風險報酬比動態色彩標記邏輯
    let rrEl = document.getElementById('rec-rr-ratio');
    let tp2Badge = document.getElementById('tp2-rr-badge');
    let numRr = parseFloat(rrRatio);

    rrEl.textContent = '1 : ' + rrRatio;
    tp2Badge.textContent = '1 : ' + rrRatio;

    if (numRr < 1.5) {
        rrEl.className = "font-bold text-xs text-yellow-400 font-mono";
        tp2Badge.className = "text-[9px] text-yellow-300 font-bold bg-yellow-500/20 px-1.5 py-0.5 rounded";
    } else if (numRr >= 2.5) {
        rrEl.className = "font-bold text-xs text-purple-300 font-mono";
        tp2Badge.className = "text-[9px] text-purple-300 font-bold bg-purple-500/20 px-1.5 py-0.5 rounded";
    } else {
        rrEl.className = "font-bold text-xs text-emerald-400 font-mono";
        tp2Badge.className = "text-[9px] text-purple-300 font-bold";
    }

    document.getElementById('rec-price').textContent = '$' + entryPrice.toFixed(decimals);
    document.getElementById('rec-sl').textContent = '$' + slPrice.toFixed(decimals) + ' (ATR動態防護)';
    document.getElementById('rec-tp1').textContent = '$' + tp1Price.toFixed(decimals);
    document.getElementById('rec-tp2').textContent = '$' + tp2Price.toFixed(decimals);

    currentStrategyData = { symbol, direction: isShort ? 'short' : 'long', entry: entryPrice, sl: slPrice, tp1: tp1Price, tp2: tp2Price, decimals, mode: currentEntryMode };
    calculatePositionSize();
}

function priceChangeTrend(data) {
    let closes = data.closes;
    return closes[closes.length - 1] - closes[closes.length - 5];
}

function updateLivePrice(price, change, funding) {
    let decimals = bestKData?.decimals || getDecimalPlaces(price);
    document.getElementById('coin-price').textContent = '$' + price.toFixed(decimals);
    document.getElementById('coin-change').textContent = '24h 漲跌: ' + (change >= 0 ? "+" : "") + change.toFixed(2) + '%';
    document.getElementById('funding-val').textContent = (funding || 0.01).toFixed(4) + '%';
}

function calculatePositionSize() {
    if (!currentStrategyData.entry) return;
    let maxLossUSDT = parseFloat(document.getElementById('risk-loss-input').value) || 3;
    let riskPerUnit = Math.abs(currentStrategyData.entry - currentStrategyData.sl);
    if (riskPerUnit === 0) return;
    let maxQty = maxLossUSDT / riskPerUnit;
    let notional = maxQty * currentStrategyData.entry;
    let leverage = Math.min(Math.max(Math.floor(notional / maxLossUSDT), 5), 30);

    let liqPrice = currentStrategyData.direction === 'long' 
        ? currentStrategyData.entry * (1 - (0.9 / leverage)) 
        : currentStrategyData.entry * (1 + (0.9 / leverage));

    document.getElementById('calc-qty').textContent = maxQty > 1000 ? maxQty.toFixed(2) : maxQty.toFixed(4);
    document.getElementById('calc-notional').textContent = '$' + notional.toFixed(2);
    document.getElementById('calc-leverage').textContent = leverage + 'x (安全防護)';
    document.getElementById('calc-liq-price').textContent = '$' + liqPrice.toFixed(currentStrategyData.decimals || 2);
}

function changeSymbol() {
    let query = document.getElementById('search-input').value.trim().toUpperCase() || "BTC";
    if (!query.endsWith('USDT')) query += 'USDT';
    currentSymbol = query;
    if (ws && ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ op: 'subscribe', args: ['tickers.' + currentSymbol] }));
    }
    fetchSymbolData(currentSymbol);
}

function quickSelect(sym) {
    document.getElementById('search-input').value = sym.replace('USDT', '');
    changeSymbol();
}

function calculateRSI(closes, period = 14) {
    let gains = 0, losses = 0;
    for (let i = 1; i <= period; i++) {
        let diff = closes[i] - closes[i - 1];
        if (diff >= 0) gains += diff; else losses -= diff;
    }
    let avgGain = gains / period, avgLoss = losses / period;
    for (let i = period + 1; i < closes.length; i++) {
        let diff = closes[i] - closes[i - 1];
        if (diff >= 0) { avgGain = (avgGain * 13 + diff) / 14; avgLoss = (avgLoss * 13) / 14; }
        else { avgGain = (avgGain * 13) / 14; avgLoss = (avgLoss * 13 - diff) / 14; }
    }
    return avgLoss === 0 ? 100 : 100 - (100 / (1 + (avgGain / avgLoss)));
}

function calculateATR(highs, lows, closes, period = 14) {
    let trs = [];
    for (let i = 1; i < highs.length; i++) {
        trs.push(Math.max(highs[i] - lows[i], Math.abs(highs[i] - closes[i-1]), Math.abs(lows[i] - closes[i-1])));
    }
    return trs.slice(-period).reduce((a,b)=>a+b,0) / period;
}

async function sendToDiscord(text) {
    try {
        await fetch(BUILTIN_WEBHOOK, {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({ content: text })
        });
    } catch (e) {}
}

function addToWinTracker() {
    if (!currentStrategyData.symbol) return;
    let trackers = JSON.parse(localStorage.getItem('ny_session_trackers') || '[]');
    trackers.push({ id: Date.now(), symbol: currentStrategyData.symbol, direction: currentStrategyData.direction, tp: currentStrategyData.tp2, sl: currentStrategyData.sl, status: 'pending' });
    localStorage.setItem('ny_session_trackers', JSON.stringify(trackers));
    
    let dec = currentStrategyData.decimals || 4;
    let msg = `🎯 **【新納入 ATR 防線策略勝率追蹤】**\n` +
              `----------------------------------\n` +
              `📌 標的：**${currentStrategyData.symbol}**\n` +
              `🧭 方向：**${currentStrategyData.direction === 'long' ? '🟢 做多 (LONG)' : '🔻 做空 (SHORT)'}**\n` +
              `🎯 進場點：\`$${currentStrategyData.entry.toFixed(dec)}\`\n` +
              `🛑 ATR 動態止損 (SL)：\`$${currentStrategyData.sl.toFixed(dec)}\`\n` +
              `🛡️ TP1 (50%保本)：\`$${currentStrategyData.tp1.toFixed(dec)}\`\n` +
              `🚀 TP2 (結構終點)：\`$${currentStrategyData.tp2.toFixed(dec)}\`\n` +
              `----------------------------------`;
              
    sendToDiscord(msg);
    alert("🎯 已納入勝率追蹤，並已成功推送到 Discord！");
    updateTrackerUI();
}

async function runAutoAudit() {
    let trackers = JSON.parse(localStorage.getItem('ny_session_trackers') || '[]');
    if (trackers.length === 0) return;
    let updated = false;
    for (let item of trackers) {
        if (item.status !== 'pending') continue;
        let curPrice = tickersMap[item.symbol]?.price;
        if (!curPrice) continue;
        if (item.direction === 'long') {
            if (curPrice >= item.tp) { item.status = 'win'; updated = true; sendToDiscord(`✅ **【 WIN】** ${item.symbol} 觸及 TP2！`); }
            else if (curPrice <= item.sl) { item.status = 'loss'; updated = true; sendToDiscord(`❌ **【 LOSS】** ${item.symbol} 觸及 SL。`); }
        } else {
            if (curPrice <= item.tp) { item.status = 'win'; updated = true; sendToDiscord(`✅ **【 WIN】** ${item.symbol} 觸及 TP2！`); }
            else if (curPrice >= item.sl) { item.status = 'loss'; updated = true; sendToDiscord(`❌ **【 LOSS】** ${item.symbol} 觸及 SL。`); }
        }
    }
    if (updated) { localStorage.setItem('ny_session_trackers', JSON.stringify(trackers)); updateTrackerUI(); }
}

function updateTrackerUI() {
    let trackers = JSON.parse(localStorage.getItem('ny_session_trackers') || '[]');
    let wins = trackers.filter(d => d.status === 'win').length;
    let losses = trackers.filter(d => d.status === 'loss').length;
    let finished = wins + losses;
    let rate = finished > 0 ? ((wins / finished) * 100).toFixed(1) : "0.0";

    document.getElementById('stat-total').textContent = trackers.length;
    document.getElementById('stat-win-loss').textContent = wins + ' / ' + losses;
    document.getElementById('stat-win-rate').textContent = rate + '%';

    let container = document.getElementById('tracker-list-container');
    if (trackers.length === 0) {
        container.innerHTML = '<div class="text-[11px] text-gray-500 text-center py-2">尚無追蹤紀錄</div>';
        return;
    }
    container.innerHTML = trackers.slice().reverse().map(i => `
        <div class="flex justify-between items-center p-2 bg-gray-800/40 rounded-xl border border-gray-800 text-[11px]">
            <span class="font-bold text-gray-200">${i.symbol} (${i.direction.toUpperCase()})</span>
            <div>${i.status === 'win' ? '✅ 獲勝' : i.status === 'loss' ? '❌ 停損' : '⏳ 追蹤中'}</div>
        </div>
    `).join('');
}

function clearTrackerData() {
    if (confirm("確定清除對帳紀錄？")) { localStorage.removeItem('ny_session_trackers'); updateTrackerUI(); }
}

function copyStrategy() {
    if (!currentStrategyData.symbol) return;
    let dec = currentStrategyData.decimals || 4;
    let text = `【動態 EMA 200 & ATR 防線 1H SMC 雷達 v3.1】\n` +
               `標的: ${currentStrategyData.symbol} (${currentStrategyData.direction.toUpperCase()})\n` +
               `進場價: ${currentStrategyData.entry.toFixed(dec)}\n` +
               `ATR 動態止損: ${currentStrategyData.sl.toFixed(dec)}\n` +
               `TP1: ${currentStrategyData.tp1.toFixed(dec)} | TP2: ${currentStrategyData.tp2.toFixed(dec)}`;
    navigator.clipboard.writeText(text);
    alert("📋 已成功複製操盤數據！");
}

function quickAction() { copyStrategy(); }

window.onload = init;
</script>

</body>
</html>
