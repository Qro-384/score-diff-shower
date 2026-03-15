<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const ocrScores = ref({ p1: 0, p2: 0 }) 
const config = ref({
  p1Color: '#ff4b4b',
  p2Color: '#4b4bff',
  drawColor: 'white',
  swapSides: false,
  swapOcr: false,
  manualMode: false,
  manualScores: { p1: 0, p2: 0 }
})

// --- パラメータ ---
const MAX_SCORE_DIFF = 20000 
// ※遅延用の DELAY_MS や timedHistory は不要になったため削除しました

// データ受信 (これが呼ばれた瞬間に即座にVueの変数に代入)
const updateScores = (p1, p2, sourceTs) => {
  ocrScores.value.p1 = p1;
  ocrScores.value.p2 = p2;
}

// --- Computed & Config ---
const scores = computed(() => {
  // 1. 手動モードがONなら手動スコアを表示
  if (config.value.manualMode) {
    return config.value.manualScores;
  }
  // 2. OCRデータ交差(swapOcr)がONなら、データを入れ替えて表示
  if (config.value.swapOcr) {
    return {
      p1: ocrScores.value.p2,
      p2: ocrScores.value.p1
    };
  }
  // 3. どちらもOFFなら生のデータをそのまま表示
  return ocrScores.value;
})

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
      if (msg.type === 'config') {
        // コントローラからの設定変更を受信
        config.value = { ...config.value, ...msg.data }
      } else if (msg.type === 'score') {
        // OCRからのスコアを受信
        updateScores(parseInt(msg.data.p1), parseInt(msg.data.p2), msg.data.ts*1000)
      }
    } catch (e) { console.error(e) }
  }
  ws.onclose = () => setTimeout(connect, 3000)
}

onMounted(() => {
  connect()
  // ※runPlayback() の呼び出しは不要になったため削除しました
})
onUnmounted(() => {
  if (ws) ws.close()
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