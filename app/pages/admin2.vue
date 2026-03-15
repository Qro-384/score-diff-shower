<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick, computed } from 'vue'

// --- 設定: 事前に決めた5チーム(色・画像・名前) ---
// ※画像のパスはご自身の環境に合わせて変更してください
const PRESET_TEAMS = [
  { color: '#4eaec6', name: 'ABOVE', image: '/img/above.webp' },
  { color: '#ffef49', name: 'Limit Break3r', image: '/img/limit.webp' },
  { color: '#5a4498', name: 'Cobalt Echo', image: '/img/cobalt.jpeg' },
  { color: '#aa0000', name: '汝の縦連を愛せよ', image: '/img/tateren.webp' },
  { color: '#00ffff', name: 'Lightning Speed', image: '/img/lightning.webp' }
]

// --- 状態管理 ---
const config = ref({
  p1Name: 'Player 1',
  p2Name: 'Player 2',
  p1Color: PRESET_TEAMS[0].color,
  p2Color: PRESET_TEAMS[1].color,
  swapSides: false,   // 見た目の左右反転 (座席)
  swapOcr: false,     // データの交差 (OCR紐づけ)
  manualMode: false,
  manualScores: { p1: 0, p2: 0 }
})

// OCRから送られてくる純粋な生データ（枠A, 枠B）
const liveScores = ref({ p1: 0, p2: 0 })

// --- 算出プロパティ ---
// ルーティング（紐づけ）後のスコア計算
const currentOutputScores = computed(() => {
  if (config.value.manualMode) return config.value.manualScores;
  
  if (config.value.swapOcr) {
    return { p1: liveScores.value.p2, p2: liveScores.value.p1 };
  }
  return liveScores.value;
})

// 選択されている色から、プレビュー用の画像URLを割り出す
const p1Image = computed(() => {
  const team = PRESET_TEAMS.find(t => t.color === config.value.p1Color)
  return team ? team.image : null
})

const p2Image = computed(() => {
  const team = PRESET_TEAMS.find(t => t.color === config.value.p2Color)
  return team ? team.image : null
})

const isConnected = ref(false)
const isRemoteUpdate = ref(false)
let ws = null

// --- WebSocket 通信 ---
const connect = () => {
  // 環境に合わせてURLを変更
  const WS_URL = "wss://score-diff-server.fly.dev/ws"
  ws = new WebSocket(WS_URL)

  ws.onopen = () => { isConnected.value = true }

  ws.onmessage = (event) => {
    try {
      const msg = JSON.parse(event.data)
      
      if (msg.type === 'config') {
        isRemoteUpdate.value = true
        config.value = { ...config.value, ...msg.data }
        nextTick(() => { isRemoteUpdate.value = false })
      } else if (msg.type === 'score') {
        liveScores.value = msg.data
        if (!config.value.manualMode) {
          config.value.manualScores = config.value.swapOcr 
            ? { p1: msg.data.p2, p2: msg.data.p1 } 
            : { p1: msg.data.p1, p2: msg.data.p2 };
        }
      }
    } catch (e) { console.error(e) }
  }

  ws.onclose = () => {
    isConnected.value = false
    setTimeout(connect, 3000)
  }
}

// --- 送信ロジック ---
const sendConfig = () => {
  if (!ws || ws.readyState !== WebSocket.OPEN) return
  ws.send(JSON.stringify({ type: 'config', data: config.value }))
}

// カラー変更
const setColor = (player, color) => {
  if (player === 'p1') config.value.p1Color = color
  else config.value.p2Color = color
  sendConfig()
}


const setTeam = (player, team) => {
  if (player === 'p1') {
    config.value.p1Color = team.color
    config.value.p1Name = team.name   // ★名前も自動代入
  } else {
    config.value.p2Color = team.color
    config.value.p2Name = team.name   // ★名前も自動代入
  }
  sendConfig()
}

// 手動モード切替
const toggleManualMode = () => {
  config.value.manualMode = !config.value.manualMode
  sendConfig()
}

watch(() => config.value.manualScores, () => {
  if (isRemoteUpdate.value) return
  if (config.value.manualMode) sendConfig()
}, { deep: true })

onMounted(() => connect())
onUnmounted(() => { if (ws) ws.close() })
</script>

<template>
  <div class="admin-container">
    <header :class="{ active: isConnected }">
      <h1>🎛 大会コントローラー</h1>
      <span class="status">{{ isConnected ? '● ONLINE' : '● OFFLINE' }}</span>
    </header>

    <main>
      <section class="card preview-zone">
        <h2>📺 配信画面プレビュー (配置・データ確認)</h2>
        <p class="desc">現在のプレイヤー配置と、適用されているOCR枠を確認できます。</p>
        
        <div class="preview-layout" :class="{ 'is-swapped': config.swapSides }">
          
          <div class="preview-names">
            <div class="player-info">
              <img v-if="p1Image" :src="p1Image" class="team-logo" alt="1P Logo">
              <div class="p-name" :style="{ color: config.p1Color }">{{ config.p1Name || '1P' }}</div>
              <div class="ocr-badge" :class="{ 'is-swapped-data': config.swapOcr }">
                {{ config.swapOcr ? '📥 OCR枠B (下)' : '📥 OCR枠A (上)' }}
              </div>
            </div>
            
            <div class="player-info">
              <img v-if="p2Image" :src="p2Image" class="team-logo" alt="2P Logo">
              <div class="p-name" :style="{ color: config.p2Color }">{{ config.p2Name || '2P' }}</div>
              <div class="ocr-badge" :class="{ 'is-swapped-data': config.swapOcr }">
                {{ config.swapOcr ? '📥 OCR枠A (上)' : '📥 OCR枠B (下)' }}
              </div>
            </div>
          </div>
          
          <div class="static-bar-container">
            <div class="static-bar" :style="{ background: config.p1Color }"></div>
            <div class="static-bar" :style="{ background: config.p2Color }"></div>
          </div>
        </div>
      </section>

      <section class="card routing-zone">
        <h2>🔄 1. スコアの紐づけ (データルーティング)</h2>
        <p class="desc">管理画面の並び順に合わせて、OCRデータを選手に割り当てます。</p>
        
        <button 
          class="toggle-btn routing-btn" 
          :class="{ active: config.swapOcr }"
          @click="() => { config.swapOcr = !config.swapOcr; sendConfig(); }"
        >
          <div v-if="!config.swapOcr">
            <span class="badge normal">通常</span><br>
            OCR枠A ➔ <b>{{ config.p1Name || '1P' }}</b><br>
            OCR枠B ➔ <b>{{ config.p2Name || '2P' }}</b>
          </div>
          <div v-else>
            <span class="badge swapped">交差</span><br>
            OCR枠A ➔ <b>{{ config.p2Name || '2P' }}</b><br>
            OCR枠B ➔ <b>{{ config.p1Name || '1P' }}</b>
          </div>
        </button>

        <div class="monitor-box">
          <div class="monitor-item">
            <span class="label">📥 入力 (OCR枠A / 枠B):</span>
            <span class="value">{{ liveScores.p1 }} / {{ liveScores.p2 }}</span>
          </div>
          <div class="monitor-item highlight">
            <span class="label">📺 出力 (1P / 2P):</span>
            <span class="value">
              <span class="p-score">{{ currentOutputScores.p1 }}</span>
              <span class="divider">/</span>
              <span class="p-score">{{ currentOutputScores.p2 }}</span>
            </span>
          </div>
        </div>
      </section>

      <section class="card">
        <h2>🎨 2. 配信画面の見た目 (カラー・座席)</h2>
        
        <div class="control-row">
            <div class="color-group">
            <label class="dynamic-label">{{ config.p1Name || '1P' }}</label>
            <div class="color-buttons">
                <button 
                v-for="team in PRESET_TEAMS" :key="team.color" 
                class="color-btn" 
                :style="{ backgroundColor: team.color }" 
                :class="{ selected: config.p1Color === team.color }" 
                :title="team.name"
                @click="setTeam('p1', team)">
                </button>
            </div>
            </div>
            <div class="color-group">
            <label class="dynamic-label">{{ config.p2Name || '2P' }}</label>
            <div class="color-buttons">
                <button 
                v-for="team in PRESET_TEAMS" :key="team.color" 
                class="color-btn" 
                :style="{ backgroundColor: team.color }" 
                :class="{ selected: config.p2Color === team.color }" 
                :title="team.name"
                @click="setTeam('p2', team)">
                </button>
            </div>
            </div>
        </div>

        <div class="separator-line"></div>

        <p class="desc">※選手が座る座席位置（カメラの配置）に合わせてバーの左右を丸ごと反転します。</p>
        <button 
          class="toggle-btn" 
          :class="{ active: config.swapSides }" 
          @click="() => { config.swapSides = !config.swapSides; sendConfig(); }"
        >
          バーの配置: {{ config.swapSides ? `[左: ${config.p2Name}] / [右: ${config.p1Name}]` : `[左: ${config.p1Name}] / [右: ${config.p2Name}]` }}
        </button>
      </section>

      <section class="card danger-zone" :class="{ 'mode-active': config.manualMode }">
        <div class="manual-header">
          <h2>✏️ 4. 緊急手動修正</h2>
          <button class="mode-toggle" :class="{ active: config.manualMode }" @click="toggleManualMode">
            {{ config.manualMode ? '手動モードON (編集中)' : '自動モード (OCR)' }}
          </button>
        </div>
        <div class="score-inputs">
          <div class="input-group">
            <label>{{ config.p1Name || '1P' }}</label>
            <input type="number" v-model="config.manualScores.p1" :disabled="!config.manualMode">
          </div>
          <div class="input-group">
            <label>{{ config.p2Name || '2P' }}</label>
            <input type="number" v-model="config.manualScores.p2" :disabled="!config.manualMode">
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

<style scoped>
.admin-container { max-width: 600px; margin: 0 auto; padding: 20px; font-family: sans-serif; color: #333; }
header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #ddd; padding-bottom: 10px; }
header.active .status { color: #00cc00; font-weight: bold; }
.card { background: #f9f9f9; border: 1px solid #ddd; border-radius: 8px; padding: 16px; margin-bottom: 20px; }
.card h2 { font-size: 1.1rem; margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 8px; margin-bottom: 12px; }
.desc { font-size: 0.85rem; color: #666; margin-top: 0; margin-bottom: 12px; line-height: 1.4; }

/* プレビューエリア */
.preview-zone { background: #1a1a1a; color: #fff; border: 2px solid #444; }
.preview-zone h2 { color: #eee; border-bottom-color: #333; }
.preview-names, .static-bar-container { display: flex; justify-content: space-between; transition: all 0.3s ease; }
.preview-layout.is-swapped .preview-names, .preview-layout.is-swapped .static-bar-container { flex-direction: row-reverse; }
.preview-names { margin-bottom: 8px; padding: 0 5px; }
.player-info { display: flex; flex-direction: column; align-items: center; width: 45%; }
/* チームロゴ追加分 */
.team-logo { height: 40px; width: auto; object-fit: contain; margin-bottom: 5px; filter: drop-shadow(0px 2px 2px rgba(0,0,0,0.5)); }
.p-name { font-size: 1.1rem; font-weight: bold; text-shadow: 1px 1px 2px rgba(0,0,0,0.8); margin-bottom: 4px; text-align: center; }
.ocr-badge { font-size: 0.75rem; background: #333; color: #aaa; padding: 2px 8px; border-radius: 4px; border: 1px solid #555; font-family: monospace; text-align: center; }
.ocr-badge.is-swapped-data { background: #4a4515; color: #ffef49; border-color: #ffef49; }
.static-bar-container { width: 100%; height: 20px; background-color: #fff; padding: 2px; box-sizing: border-box; clip-path: polygon(2% 0%, 0% 50%, 2% 100%, 98% 100%, 100% 50%, 98% 0%); }
.static-bar { flex: 1; height: 100%; transition: background 0.3s; }

/* 入力フォーム */
.score-inputs { display: flex; gap: 15px; }
.input-group { flex: 1; display: flex; flex-direction: column; gap: 6px; }
.input-group label { font-size: 0.9rem; font-weight: bold; color: #555; }
.input-group input[type="text"] { padding: 10px; font-size: 1rem; border: 1px solid #ccc; border-radius: 4px; }
.input-group input[type="number"] { width: 100%; padding: 12px; font-size: 1.5rem; text-align: right; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box; }
.input-group input:disabled { background: #eee; color: #999; }

/* ルーティングとモニター */
.routing-zone { border: 2px solid #5a4498; background: #f5f3fa; }
.routing-btn { height: 80px; font-size: 1.1rem; display: flex; align-items: center; justify-content: center; line-height: 1.5; background: #fff; border: 2px solid #5a4498; color: #333; }
.routing-btn.active { background: #5a4498; color: #fff; }
.badge { display: inline-block; padding: 2px 8px; border-radius: 12px; font-size: 0.8rem; font-weight: bold; margin-bottom: 4px; }
.badge.normal { background: #eee; color: #333; }
.badge.swapped { background: #ffef49; color: #333; }
.monitor-box { margin-top: 15px; background: #1e1e1e; border-radius: 6px; padding: 12px; font-family: 'Courier New', monospace; color: #aaa; display: flex; flex-direction: column; gap: 8px; }
.monitor-item { display: flex; justify-content: space-between; align-items: center; }
.monitor-item.highlight { color: #fff; padding-top: 8px; border-top: 1px dashed #444; font-size: 1.1rem; font-weight: bold; }
.monitor-item .value { display: flex; gap: 10px; }
.monitor-item .p-score { width: 80px; text-align: right; }
.monitor-item .divider { color: #555; }

/* カラーボタン等 */
.control-row { display: flex; gap: 20px; justify-content: space-around; flex-wrap: wrap; margin-bottom: 10px; }
.color-group { display: flex; flex-direction: column; align-items: center; gap: 8px; }
.dynamic-label { font-weight: bold; font-size: 0.95rem; }
.color-buttons { display: flex; gap: 8px; }
.color-btn { width: 32px; height: 32px; border-radius: 50%; border: 2px solid transparent; cursor: pointer; transition: transform 0.1s; }
.color-btn:hover { transform: scale(1.1); }
.color-btn.selected { border-color: #333; box-shadow: 0 0 0 2px #fff inset; transform: scale(1.2); }
.separator-line { height: 1px; background: #ddd; margin: 15px 0; }
.toggle-btn { width: 100%; padding: 12px; border-radius: 4px; font-weight: bold; cursor: pointer; transition: 0.2s; border: 1px solid #ccc; background: #eee; }

/* 手動モード */
.danger-zone.mode-active { border: 2px solid #ff4b4b; background: #fff0f0; }
.manual-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
.mode-toggle { padding: 6px 16px; border-radius: 20px; border: none; background: #ccc; cursor: pointer; font-weight: bold; color: #fff; }
.mode-toggle.active { background: #ff4b4b; }
</style>