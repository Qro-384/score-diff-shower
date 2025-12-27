<script setup>
  import { ref, computed, onMounted, onUnmounted } from 'vue'
  
  // --- 状態管理 ---
  const ocrScores = ref({ p1: 0, p2: 0 }) // Pythonから来るスコア
  const config = ref({
    p1Color: '#ff4b4b',
    p2Color: '#4b4bff',
    drawColor: 'black',
    swapSides: false,
    // コントローラーと同じ初期値
    manualMode: false,
    manualScores: { p1: 0, p2: 0 }
  })
  
    // --- 定数 ---
    const MAX_SCORE_DIFF = 20000 // ゲージが振り切れる点差

    const scores = computed(() => {
    if (config.value.manualMode) {
      return config.value.manualScores
    } else {
      return ocrScores.value  
    }
  })
  
  // --- 計算プロパティ (Computed) ---
  // スコア差分
  const diff = computed(() => scores.value.p1 - scores.value.p2)
  const absDiff = computed(() => Math.abs(diff.value))
  
  // 差分の色判定
  const diffColor = computed(() => {
    if (diff.value === 0) return config.value.drawColor
    // P1が勝っている(プラス)ならP1色、負けているならP2色
    // ※左右反転(swapSides)していても「勝っている側の色」を出すためロジックは不変でOK
    return diff.value > 0 ? config.value.p1Color : config.value.p2Color
  })
  
  // ゲージの長さ計算 (50%を基準に増減)
  const p1BarPercent = computed(() => {
    let diffRatio = diff.value / MAX_SCORE_DIFF
    // 振り切れ防止 (-0.95 ～ 0.95)
    if (diffRatio > 0.95) diffRatio = 0.95
    if (diffRatio < -0.95) diffRatio = -0.95
    
    return 50 + (diffRatio * 50)
  })
  
  const p2BarPercent = computed(() => 100 - p1BarPercent.value)
  
  // --- WebSocket 接続 ---
  let ws = null
  
  const connect = () => {
    // 環境に合わせてURLを変更してください (Fly.ioなら wss://...)
    const WS_URL = "wss://score-diff-server.fly.dev/ws"
    
    ws = new WebSocket(WS_URL)
  
    ws.onmessage = (event) => {
      try {
        const msg = JSON.parse(event.data)
  
        // データタイプで分岐 (将来の拡張用)
        // もし今のサーバーが {p1: 100, p2: 200} しか送ってこないなら
        // そのまま scores.value に入れれば動くようにしています
        if (msg.type === 'config') {
          // コントローラーからの設定変更
          config.value = { ...config.value, ...msg.data }
        } else if (msg.type === 'score') {
          // スコア更新 (形式: { type: 'score', data: { p1:..., p2:... } })
          scores.value = { 
            p1: parseInt(msg.data.p1), 
            p2: parseInt(msg.data.p2) 
          }
        } else if (msg.p1 !== undefined) {
          // 旧形式 (形式: { p1: 100, p2: 200 }) への対応
          scores.value = { 
            p1: parseInt(msg.p1), 
            p2: parseInt(msg.p2) 
          }
        }
      } catch (e) {
        console.error("Data parse error", e)
      }
    }
  
    ws.onclose = () => {
      // 切断されたら再接続を試みる (3秒後)
      setTimeout(connect, 3000)
    }
  }
  
  onMounted(() => {
    connect()
  })
  
  onUnmounted(() => {
    if (ws) ws.close()
  })
  </script>
  
  <template>
    <div class="overlay-container">
      
      <div class="bar-container">
        <div class="bar-inner">
          <div 
          class="bar bar-p1" 
          :style="{ 
            width: (config.swapSides ? p1BarPercent : p2BarPercent) + '%',
            background:  (config.swapSides ? config.p1Color : config.p2Color)
          }"
          ></div>

          <div 
          class="bar bar-p2" 
          :style="{ 
            width: (config.swapSides ? p2BarPercent : p1BarPercent) + '%',
            background:  (config.swapSides ? config.p2Color : config.p1Color)
          }"
          ></div>
        </div>
      </div>
  
      <div class="separator"></div>
  
      <div class="score-text-area" :class="{ 'is-swapped': config.swapSides }">
        <div class="score-item" id="p1-score">{{ scores.p1 }}</div>
        <div class="score-item" id="p2-score">{{ scores.p2 }}</div>
      </div>
  
      <div 
        class="difference-score-area" 
        :style="{ color: diffColor }"
      >
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
    -webkit-text-stroke: 1px black;
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
    width: 4px;
    background-color: white;
    position: absolute;
    left: 50%;
    transform: translateX(-50%);
    top: 4px;
    height: 32px;
    z-index: 10;
  }
  
  </style>