<script setup>
    import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue'
    
    // --- 設定: 事前に決めた5色 ---
    const PRESET_COLORS = [
      '#4eaec6', // ABOVE
      '#ffef49', // Limit Break3r
      '#5a4498', // Cobalt Echo
      '#aa0000', // 汝の縦連を愛せよ
      '#00ffff'  // Lightning Speed
    ]
    
    // --- 状態管理 ---
    const config = ref({
      p1Color: PRESET_COLORS[0],
      p2Color: PRESET_COLORS[1],
      swapSides: false,
      // ★追加: 手動モードフラグと、その時の固定スコア
      manualMode: false,
      manualScores: { p1: 0, p2: 0 }
    })
    
    // 表示用スコア（手動モードがOFFならサーバーからの値を表示、ONなら手動値を表示）
    const liveScores = ref({ p1: 0, p2: 0 })
    
    const isConnected = ref(false)
    const isRemoteUpdate = ref(false)
    let ws = null
    
    // --- WebSocket 通信 ---
    const connect = () => {
      // 環境に合わせてURLを変更 (Fly.ioなら wss://...)
      const WS_URL = "wss://score-diff-server.fly.dev/ws"
      ws = new WebSocket(WS_URL)
    
      ws.onopen = () => { isConnected.value = true }
    
      ws.onmessage = (event) => {
        try {
          const msg = JSON.parse(event.data)
          
          if (msg.type === 'config') {
            // サーバーから最新設定を受け取る
            // 手動モード中は入力欄が勝手に書き換わると困るので、モード同期は慎重に行う
            // ここでは単純に全て同期します
            isRemoteUpdate.value = true
            config.value = { ...config.value, ...msg.data }
            nextTick(() => { isRemoteUpdate.value = false })
          } else if (msg.type === 'score') {
            // OCRからの生スコア（監視用）
            liveScores.value = msg.data
            
            // もし手動モードがOFFなら、手動入力欄にも今のスコアを反映させておく
            // (ONにした瞬間に今のスコアから編集できるようにするため)
            if (!config.value.manualMode) {
              config.value.manualScores = { ...msg.data }
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
      
      const payload = {
        type: 'config',
        data: config.value
      }
      ws.send(JSON.stringify(payload))
    }
    
    // 色変更
    const setColor = (player, color) => {
      if (player === 'p1') config.value.p1Color = color
      else config.value.p2Color = color
      sendConfig()
    }
    
    // 手動モード切替
    const toggleManualMode = () => {
      config.value.manualMode = !config.value.manualMode
      sendConfig()
    }
    
    // 手動スコア変更（入力があるたびに送信）
    // watchを使って、manualScoresの中身が変わったら自動送信する
    watch(() => config.value.manualScores, () => {
      if (isRemoteUpdate.value) return
      if (config.value.manualMode) {
        sendConfig()
      }
    }, { deep: true })
    
    onMounted(() => connect())
    onUnmounted(() => { if (ws) ws.close() })
    </script>
    
    <template>
      <div class="admin-container">
        <header :class="{ active: isConnected }">
          <h1>🎛 コントローラー</h1>
          <span class="status">{{ isConnected ? '● ONLINE' : '● OFFLINE' }}</span>
        </header>
    
        <main>
          <section class="card">
            <h2>🎨 カラープリセット</h2>
            <div class="control-row">
              <div class="color-group">
                <label>1P Color</label>
                <div class="color-buttons">
                  <button 
                    v-for="c in PRESET_COLORS" :key="c"
                    class="color-btn"
                    :style="{ backgroundColor: c }"
                    :class="{ selected: config.p1Color === c }"
                    @click="setColor('p1', c)"
                  ></button>
                </div>
              </div>
              <div class="color-group">
                <label>2P Color</label>
                <div class="color-buttons">
                  <button 
                    v-for="c in PRESET_COLORS" :key="c"
                    class="color-btn"
                    :style="{ backgroundColor: c }"
                    :class="{ selected: config.p2Color === c }"
                    @click="setColor('p2', c)"
                  ></button>
                </div>
              </div>
            </div>
          </section>
    
          <section class="card">
            <h2>⇄ レイアウト</h2>
            <button 
              class="toggle-btn" 
              :class="{ active: config.swapSides }"
              @click="() => { config.swapSides = !config.swapSides; sendConfig(); }"
            >
              左右入れ替え: {{ config.swapSides ? 'ON (反転)' : 'OFF (通常)' }}
            </button>
          </section>
    
          <section class="card danger-zone" :class="{ 'mode-active': config.manualMode }">
            <div class="manual-header">
              <h2>✏️ スコア手動修正</h2>
              <button 
                class="mode-toggle" 
                :class="{ active: config.manualMode }"
                @click="toggleManualMode"
              >
                {{ config.manualMode ? '固定モードON (編集中)' : '自動モード (OCR)' }}
              </button>
            </div>
    
            <p class="desc">
              スイッチをONにすると、入力した値でスコアが固定されます。
            </p>
            
            <div class="score-inputs">
              <div class="input-group">
                <label>1P Score</label>
                <input 
                  type="number" 
                  v-model="config.manualScores.p1" 
                  :disabled="!config.manualMode"
                >
              </div>
              <div class="input-group">
                <label>2P Score</label>
                <input 
                  type="number" 
                  v-model="config.manualScores.p2" 
                  :disabled="!config.manualMode"
                >
              </div>
            </div>
          </section>
        </main>
      </div>
    </template>
    
    <style scoped>
    .admin-container { max-width: 600px; margin: 0 auto; padding: 20px; font-family: sans-serif; color: #333; }
    header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #ddd; padding-bottom: 10px; }
    header.active .status { color: #00cc00; }
    .card { background: #f9f9f9; border: 1px solid #ddd; border-radius: 8px; padding: 16px; margin-bottom: 20px; }
    .card h2 { font-size: 1rem; margin-top: 0; border-bottom: 1px solid #eee; padding-bottom: 8px; margin-bottom: 12px; }
    
    /* カラーボタン */
    .control-row { display: flex; gap: 20px; justify-content: space-around; flex-wrap: wrap; }
    .color-group { display: flex; flex-direction: column; align-items: center; gap: 8px; }
    .color-buttons { display: flex; gap: 8px; }
    .color-btn { width: 32px; height: 32px; border-radius: 50%; border: 2px solid transparent; cursor: pointer; transition: transform 0.1s; }
    .color-btn:hover { transform: scale(1.1); }
    .color-btn.selected { border-color: #333; box-shadow: 0 0 0 2px #fff inset; transform: scale(1.2); }
    
    /* トグルボタン */
    .toggle-btn { width: 100%; padding: 12px; background: #eee; border: 1px solid #ccc; border-radius: 4px; font-weight: bold; cursor: pointer; }
    .toggle-btn.active { background: #ffe0b2; border-color: #ffb74d; color: #e65100; }
    
    /* 手動モード */
    .danger-zone.mode-active { border: 2px solid #ff4b4b; background: #fff0f0; }
    .manual-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
    .mode-toggle { padding: 6px 16px; border-radius: 20px; border: none; background: #ccc; cursor: pointer; font-weight: bold; color: #fff; }
    .mode-toggle.active { background: #ff4b4b; }
    .score-inputs { display: flex; gap: 10px; }
    .input-group { flex: 1; }
    .input-group input { width: 100%; padding: 12px; font-size: 1.5rem; text-align: right; border: 1px solid #ccc; border-radius: 4px; }
    .input-group input:disabled { background: #eee; color: #999; }
    </style>