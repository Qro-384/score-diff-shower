<script setup>
  import { ref, computed, onMounted, onUnmounted } from 'vue'
  
  const ocrScores = ref({ p1: 0, p2: 0 }) 
  const config = ref({
    p1Color: '#ff4b4b',
    p2Color: '#4b4bff',
    drawColor: 'white',
    swapSides: false,
    manualMode: false,
    manualScores: { p1: 0, p2: 0 }
  })
  
  // --- パラメータ ---
  const MAX_SCORE_DIFF = 20000 
  const FILTER_WINDOW_SIZE = 3 
  const DELAY_MS = 1000        // 全体のバッファ時間（大きめに確保）
  const MISS_TIMEOUT_MS = 1500 // 「ミス」と確定するまでの待機時間（ラグの最大許容時間）
  
  // --- 内部バッファ ---
  const timedHistory = [] 
  const scoreHistory = { p1: [], p2: [] }
  
  // メジアンフィルタ
  const getMedian = (arr) => {
    if (arr.length === 0) return 0
    const sorted = [...arr].sort((a, b) => a - b)
    return sorted[Math.floor(sorted.length / 2)]
  }

  let serverOffset = null;
  
  // データ受信
  const updateScores = (p1, p2, sourceTs) => {
    const currentOffset = Date.now() - sourceTs
      
    if (serverOffset === null) {
      serverOffset = currentOffset
    } else {
      // ★移動平均：今のオフセットに 5% だけ混ぜる（ゆっくり追従）
      serverOffset = serverOffset * 0.95 + currentOffset * 0.05
    }
    scoreHistory.p1.push(p1)
    scoreHistory.p2.push(p2)
    if (scoreHistory.p1.length > FILTER_WINDOW_SIZE) scoreHistory.p1.shift()
    if (scoreHistory.p2.length > FILTER_WINDOW_SIZE) scoreHistory.p2.shift()
    
    const filtered = { 
      p1: getMedian(scoreHistory.p1), 
      p2: getMedian(scoreHistory.p2),
      ts: sourceTs
    }
    timedHistory.push(filtered)
  
    if (timedHistory.length > 500) {
      const cutoff = sourceTs - 10000
      while (timedHistory.length > 0 && timedHistory[0].ts < cutoff) timedHistory.shift()
    }
  }

let playbackLoop = null;

// ===============================================
// 定数設定 (環境に合わせて調整してください)
// ===============================================
const SMALL_DIFF_TIMEOUT = 200;  // 精度差/小ミスを反映するまでの待機時間

// Gate: 1フレームの最大増加量 (スパイクノイズ対策)
// 同時押し等を考慮し、通常のプレイでは絶対に出ない値に設定
const MAX_VALID_JUMP = 100000; 

// Threshold: これ未満のズレは「ラグ」とみなさず即座に出す
const LAG_GUARD_THRESHOLD = 300;

// ===============================================
// 状態変数 (Vueのref外で管理)
// ===============================================
let pendingDelta = { p1: 0, p2: 0 };     // バケツ (負の値も許容)
let lastProcessedRaw = { p1: 0, p2: 0 }; // 前回処理したRawデータ
let mismatchStartTime = null;            // タイマー開始時刻
let lastProcessedIndex = -1;

// ===============================================
// メインループ関数
// ===============================================
const runPlayback = () => {
  const now = Date.now();
  const targetTs = now - DELAY_MS;
  
  // 対象データの検索
  const baseIdx = timedHistory.findLastIndex(d => d.ts <= targetTs);
  
  if (baseIdx !== -1) {
    const raw = timedHistory[baseIdx];

    // -------------------------------------------------
    // 【パート1：入力】 データ更新時のみ実行
    // -------------------------------------------------
    if (baseIdx !== lastProcessedIndex) {
        
        // 初回初期化
        if (lastProcessedRaw.p1 === 0 && lastProcessedRaw.p2 === 0) {
             lastProcessedRaw.p1 = raw.p1;
             lastProcessedRaw.p2 = raw.p2;
             // 初回は画面も同期させる
             ocrScores.value.p1 = raw.p1;
             ocrScores.value.p2 = raw.p2;
        } 
        else {
            // --- P1の計算 ---
            let d1 = 0;
            const diff1 = raw.p1 - lastProcessedRaw.p1;
            
            // Gate判定
            if (diff1 > MAX_VALID_JUMP) {
                d1 = 0; // スパイク無視
            } else if (diff1 < -MAX_VALID_JUMP) {
                // リセット検知（曲の最初に戻った等）
                lastProcessedRaw.p1 = raw.p1;
                pendingDelta.p1 = 0; // バケツリセット
                ocrScores.value.p1 = raw.p1; // 画面リセット
                d1 = 0;
            } else {
                // ★ここがポイント: マイナスの値もそのまま通す
                d1 = diff1;
                lastProcessedRaw.p1 = raw.p1;
            }

            // --- P2の計算 ---
            let d2 = 0;
            const diff2 = raw.p2 - lastProcessedRaw.p2;

            if (diff2 > MAX_VALID_JUMP) {
                d2 = 0;
            } else if (diff2 < -MAX_VALID_JUMP) {
                lastProcessedRaw.p2 = raw.p2;
                pendingDelta.p2 = 0;
                ocrScores.value.p2 = raw.p2;
                d2 = 0;
            } else {
                d2 = diff2;
                lastProcessedRaw.p2 = raw.p2;
            }

            // バケツに加算（借金があれば相殺される）
            pendingDelta.p1 += d1;
            pendingDelta.p2 += d2;
        }
        
        // インデックス更新
        lastProcessedIndex = baseIdx;
    }

    // -------------------------------------------------
    // 【パート2：出力】 毎フレーム実行
    // -------------------------------------------------

    // --- A. 共通分の放出 (同期) ---
    // どちらかがマイナス(借金中)の場合、commonはマイナスになる
    const common = Math.min(pendingDelta.p1, pendingDelta.p2);
    
    // プラスの時だけ放出する
    // ※借金がある間は共通項が0以下になるので、バーは動かない（フリーズして待つ）
        ocrScores.value.p1 += common;
        ocrScores.value.p2 += common;
        pendingDelta.p1 -= common;
        pendingDelta.p2 -= common;

    // --- B. 残留分の監視 (タイムアウト処理) ---
    const currentDiff = Math.abs(pendingDelta.p1 - pendingDelta.p2);

    if (currentDiff === 0) {
        mismatchStartTime = null;
    } 
    else {
        if (mismatchStartTime === null) mismatchStartTime = now;

        const timeoutDuration = (currentDiff < LAG_GUARD_THRESHOLD) 
                                ? SMALL_DIFF_TIMEOUT 
                                : MISS_TIMEOUT_MS;

        if (now - mismatchStartTime > timeoutDuration) {
            // 強制放出ロジック
            // ※「バケツにある分」しか出せないので、マイナスの場合は無視される
            
            if (pendingDelta.p1 > pendingDelta.p2) {
                // P1が多い（P1が進んでいる or P2が借金中）
                const diff = pendingDelta.p1 - pendingDelta.p2;
                
                // P1の手持ちがプラスなら、差分を解消するために吐き出す
                if (pendingDelta.p1 > 0) {
                    // 「必要な差分」と「手持ち」の少ない方を採用
                    const flush = Math.min(pendingDelta.p1, diff);
                    ocrScores.value.p1 += flush;
                    pendingDelta.p1 -= flush;
                }
            } 
            else { // P2が多い
                const diff = pendingDelta.p2 - pendingDelta.p1;
                if (pendingDelta.p2 > 0) {
                    const flush = Math.min(pendingDelta.p2, diff);
                    ocrScores.value.p2 += flush;
                    pendingDelta.p2 -= flush;
                }
            }
            mismatchStartTime = null;
        }
    }
  }
  
  playbackLoop = requestAnimationFrame(runPlayback);
}
    
  // --- Computed & Config (変更なし) ---
  const scores = computed(() => config.value.manualMode ? config.value.manualScores : ocrScores.value)
  const diff = computed(() => scores.value.p1 - scores.value.p2)
  const absDiff = computed(() => Math.round(Math.abs(diff.value)))
  const diffColor = computed(() => {
    if (Math.abs(diff.value) < 10) return config.value.drawColor
    return diff.value > 0 ? config.value.p1Color : config.value.p2Color
  })
  const p1BarPercent = computed(() => {
    const _diff = diff.value
    let diffRatio = Math.sign(_diff) * Math.pow(Math.abs(_diff) / MAX_SCORE_DIFF, 0.5) * 0.95
    diffRatio = Math.max(-0.95, Math.min(0.95, diffRatio))
    return 50 + (diffRatio * 50)
  })
  const p2BarPercent = computed(() => 100 - p1BarPercent.value)
  
  let ws = null
  const connect = () => {
    const WS_URL = "wss://score-diff-server.fly.dev/ws"
    ws = new WebSocket(WS_URL)
    ws.onmessage = (event) => {
      try {
        const msg = JSON.parse(event.data)
        if (msg.type === 'config') config.value = { ...config.value, ...msg.data }
        else if (msg.type === 'score') updateScores(parseInt(msg.data.p1), parseInt(msg.data.p2), msg.data.ts*1000)
      } catch (e) { console.error(e) }
    }
    ws.onclose = () => setTimeout(connect, 3000)
  }
  
  onMounted(() => {
    connect()
    runPlayback()
  })
  onUnmounted(() => {
    if (ws) ws.close()
    if (playbackLoop) cancelAnimationFrame(playbackLoop)
  })
  </script>
  
  <template>
    <div class="overlay-container">
      <div class="bar-container">
        <div class="bar-inner">
          <div class="bar bar-p1" :style="{ 
            width: (config.swapSides ? p2BarPercent : p1BarPercent) + '%',
            background: (config.swapSides ? config.p2Color : config.p1Color)
          }"></div>
          <div class="bar bar-p2" :style="{ 
            width: (config.swapSides ? p1BarPercent : p2BarPercent) + '%',
            background: (config.swapSides ? config.p1Color : config.p2Color)
          }"></div>
        </div>
      </div>
  
      <div class="separator"></div>
      <div class="team-separator" :style="{ left: (config.swapSides ? p2BarPercent : p1BarPercent) + '%' }"></div>
  
      <div class="score-text-area" :class="{ 'is-swapped': config.swapSides }">
        <div class="score-item">{{ scores.p1 }}</div>
        <div class="score-item">{{ scores.p2 }}</div>
      </div>
  
      <div class="difference-score-area" :style="{ color: diffColor }">
        {{ absDiff }}
      </div>
    </div>
  </template>
  
  <style scoped>
  /* フォント定義: publicフォルダにファイルを置いてください */
  @font-face {
    font-family: 'GenEiLateMinv2';
    /* Nuxtでは / から始めると publicフォルダを参照します */
    src: url('/GenEiLateMin_v2.woff2') format('woff2'),
         url('/GenEiLateMin_v2.woff') format('woff');
    font-weight: 500;
    font-style: normal;
    font-display: swap;
  }
  
  /* Nuxtのレイアウトに影響されないよう全体設定 */
  .overlay-container {
    font-family: 'Arial Black', sans-serif;
    position: absolute;
    bottom: 50px;
    left: 50%;
    transform: translateX(-50%);
    width: 90%;
    max-width: 1600px;
    /* 文字選択などを無効化 */
    user-select: none; 
  }
  
  /* --- スコアテキスト --- */
  .score-text-area {
    font-family: "GenEiLateMinv2", sans-serif;
    display: flex;
    justify-content: space-between;
    font-size: 60px;
    margin-bottom: 10px;
    font-weight: 700;
    color: black;
    /* 左右入れ替えのアニメーション */
    transition: all 0.5s ease;
  }
  
  /* 左右反転時のスタイル (Flexboxの順序逆転) */
  .score-text-area.is-swapped {
    flex-direction: row-reverse;
  }
  
  /* --- 差分スコア --- */
  .difference-score-area {
    font-family: "GenEiLateMinv2", sans-serif;
    position: absolute;
    left: 50%;
    top: 60%;
    transform: translate(-50%, -50%);
    z-index: 20;
    white-space: nowrap;
    pointer-events: none;
    font-size: 50px;
    filter: drop-shadow(0px 0px 2px black) drop-shadow(0px 0px 2px black)
  }
  
  /* --- ゲージバー --- */
  .bar-container {
    display: flex;
    width: 100%;
    height: 40px;
    background-color: white;
    clip-path: polygon(5% 0%, 0% 50%, 5% 100%, 95% 100%, 100% 50%, 95% 0%);
    padding: 4px;
    box-sizing: border-box;
  }
  
  .bar-inner {
    display: flex;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    clip-path: polygon(5% 0%, 0% 50%, 5% 100%, 95% 100%, 100% 50%, 95% 0%);
    /* 左右反転のアニメーション */
    transition: all 0.5s ease;
  }
  
  /* バーの左右反転 */
  .bar-inner.is-swapped {
    flex-direction: row-reverse;
  }
  
  .bar {
    height: 100%;
    /* 幅の変化を滑らかにする */
    transition: width 0.5s cubic-bezier(0.22, 1, 0.36, 1);
  }
  
  .bar-p1 {
    background: linear-gradient(90deg, #ff4b4b, #ff0000);
  }
  
  .bar-p2 {
    background: linear-gradient(90deg, #0000ff, #4b4bff);
  }
  
  .separator {
    width: 0;
    height: 0;
    border-style: solid;
    border-right: 7px solid transparent;
    border-left: 7px solid transparent;
    border-top: 20px solid #000000;
    border-bottom: 0;
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
    top: -10px;
    z-index: 20;
  }

  .team-separator {
    width: 4px;
    background-color: white;
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
    top: 4px;
    height: 32px;
    z-index: 10;
    transition: left 0.5s cubic-bezier(0.22, 1, 0.36, 1);
  }
  
  </style>