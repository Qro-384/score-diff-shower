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
  
  // ★バケツ（保留中のスコア増分）
  const pendingDelta = { p1: 0, p2: 0 }
  // ★タイムアウト管理（最後に増分が来てからの経過時間）
  const lastMoveTime = { p1: 0, p2: 0 }
  // ★前回の生データ（増分計算用）
  const lastProcessedRaw = { p1: 0, p2: 0 } 
  
  // メジアンフィルタ
  const getMedian = (arr) => {
    if (arr.length === 0) return 0
    const sorted = [...arr].sort((a, b) => a - b)
    return sorted[Math.floor(sorted.length / 2)]
  }
  
  // データ受信
  const updateScores = (p1, p2) => {
    scoreHistory.p1.push(p1)
    scoreHistory.p2.push(p2)
    if (scoreHistory.p1.length > FILTER_WINDOW_SIZE) scoreHistory.p1.shift()
    if (scoreHistory.p2.length > FILTER_WINDOW_SIZE) scoreHistory.p2.shift()
    
    const filtered = { 
      p1: getMedian(scoreHistory.p1), 
      p2: getMedian(scoreHistory.p2),
      ts: Date.now() 
    }
    timedHistory.push(filtered)
  
    if (timedHistory.length > 500) {
      const cutoff = Date.now() - 10000
      while (timedHistory.length > 0 && timedHistory[0].ts < cutoff) timedHistory.shift()
    }
  }
  
  // --- 状態変数に追加 ---
  let mismatchStartTime = null; // バケツの中身が食い違い始めた時刻
  let playbackLoop = null;

  const runPlayback = () => {
    const now = Date.now()
    const targetTs = now - DELAY_MS 
    
    const baseIdx = timedHistory.findLastIndex(d => d.ts <= targetTs)
    
    if (baseIdx !== -1) {
      const raw = timedHistory[baseIdx]

      // (初期化処理は省略...)
      if (lastProcessedRaw.p1 === 0 && lastProcessedRaw.p2 === 0 && (raw.p1 > 0 || raw.p2 > 0)) {
          lastProcessedRaw.p1 = raw.p1;
          lastProcessedRaw.p2 = raw.p2;
          ocrScores.value = { p1: raw.p1, p2: raw.p2 };
      }

      // 1. バケツへの追加
      const d1 = Math.max(0, raw.p1 - lastProcessedRaw.p1)
      const d2 = Math.max(0, raw.p2 - lastProcessedRaw.p2)

      if (d1 > 0) pendingDelta.p1 += d1
      if (d2 > 0) pendingDelta.p2 += d2
      
      lastProcessedRaw.p1 = raw.p1
      lastProcessedRaw.p2 = raw.p2

      // 2. 同期放出（共通分の処理）
      // 両方のバケツにある「共通量」は、ラグ関係なく即座に反映してOK
      const common = Math.min(pendingDelta.p1, pendingDelta.p2)
      
      if (common > 0) {
        ocrScores.value.p1 += common
        ocrScores.value.p2 += common
        pendingDelta.p1 -= common
        pendingDelta.p2 -= common
      }

      // 3. 【修正版】タイムアウト判定（残存分の処理）
      // バケツの中身に差があるか？（＝片方だけが多く溜まっているか？）
      if (pendingDelta.p1 !== pendingDelta.p2) {
        
        // 差が生まれた瞬間なら、タイマー開始
        if (mismatchStartTime === null) {
          mismatchStartTime = now;
        } 
        // すでに差がある状態が続いているなら、時間をチェック
        else if (now - mismatchStartTime > MISS_TIMEOUT_MS) {
          
          // --- タイムアウト発生！強制放出 ---
          
          // P1が多く残っている場合（P2がミス、またはP1だけ連打など）
          if (pendingDelta.p1 > pendingDelta.p2) {
            const diff = pendingDelta.p1 - pendingDelta.p2;
            ocrScores.value.p1 += diff; // 差分を画面に反映（バーが動く）
            pendingDelta.p1 -= diff;    // バケツから消す
          } 
          // P2が多く残っている場合
          else {
            const diff = pendingDelta.p2 - pendingDelta.p1;
            ocrScores.value.p2 += diff;
            pendingDelta.p2 -= diff;
          }

          // 強制放出したので、バケツは平らになった（不整合解消）
          mismatchStartTime = null;
        }

      } else {
        // バケツの中身が同じ（両方0、または両方同じだけ保留中）ならタイマーリセット
        mismatchStartTime = null;
      }
    }

    playbackLoop = requestAnimationFrame(runPlayback)
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
        else if (msg.type === 'score') updateScores(parseInt(msg.data.p1), parseInt(msg.data.p2))
        else if (msg.p1 !== undefined) updateScores(parseInt(msg.p1), parseInt(msg.p2))
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