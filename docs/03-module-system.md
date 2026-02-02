# 03 - 模組系統

本章節詳細說明藍的模組系統設計，並深入分析每個現有模組的功能與實作方式。

## 模組基礎類別 ([src/module.ts](../src/module.ts))

所有功能模組都必須繼承自抽象類別 `Module`。

### 類別定義

```typescript
export default abstract class Module {
    public abstract readonly name: string;  // 模組唯一名稱
    protected ai: 藍;                        // Bot 核心實例
    private doc: any;                        // 模組私有資料文件
    
    public abstract install(): InstallerResult;
}
```

### 生命週期

```
1. 建構 (constructor)
       │
       ▼
2. 初始化 (init)
   - 接收 ai 實例
   - 載入/建立模組資料
       │
       ▼
3. 安裝 (install)
   - 回傳 mentionHook、contextHook、timeoutCallback
   - 可進行額外初始化（如訂閱串流、設定定時器）
       │
       ▼
4. 運行中
   - 透過 Hook 處理訊息
   - 透過 timeoutCallback 處理定時器事件
```

### 受保護方法

#### `log(msg: string): void`

輸出帶有模組名稱前綴的日誌。

```typescript
this.log('Processing fortune...');
// 輸出: 12:34:56 [fortune]: Processing fortune...
```

---

#### `subscribeReply(key: string | null, isChat: boolean, id: string, data?: any): void`

訂閱使用者回覆（包裝自 `ai.subscribeReply`）。

---

#### `unsubscribeReply(key: string | null): void`

取消訂閱回覆。

---

#### `setTimeoutWithPersistence(delay: number, data?: any): void`

設定持久化定時器。

---

#### `getData(): any`

取得模組的全域私有資料。

**與 `friend.getPerModulesData` 的區別**：
- `getData()`: 模組全域資料
- `friend.getPerModulesData()`: 針對特定使用者的資料

---

#### `setData(data: any): void`

設定模組的全域私有資料。

**使用範例**：
```typescript
// 記錄上次發文日期
const data = this.getData();
if (data.lastPosted === today) return;
data.lastPosted = today;
this.setData(data);
```

---

## 模組分類與清單

### 核心互動模組

| 模組 | 功能 | 預設啟用 |
|------|------|----------|
| `core` | 基礎互動（設定暱稱、版本查詢、記憶轉移） | ✅ |
| `talk` | 日常對話（問候、誇獎、摸頭等） | ✅ |
| `emoji` | 絵文字建議 | ✅ |
| `emoji-react` | 自動絵文字反應 | ✅ |
| `ping` | Ping/Pong 測試 | ✅ |
| `follow` | 處理追蹤請求 | ✅ |

### 遊戲模組

| 模組 | 功能 | 設定項 |
|------|------|--------|
| `reversi` | 黑白棋對戰 | `reversiEnabled` |
| `guessing-game` | 數字猜謎 | - |
| `kazutori` | 數取遊戲（多人） | - |
| `maze` | 迷宮生成 | - |
| `dice` | 骰子 | - |

### AI 聊天模組

| 模組 | 功能 | 設定項 |
|------|------|--------|
| `aichat` | AI 聊天（Gemini/PLaMo） | `geminiProApiKey`, `pLaMoApiKey` |

### 系統功能模組

| 模組 | 功能 | 設定項 |
|------|------|--------|
| `server` | 伺服器監控 | `serverMonitoring` |
| `chart` | 統計圖表 | `chartEnabled` |
| `check-custom-emojis` | 自訂表情符號檢查 | `checkEmojisEnabled` |
| `keyword` | 關鍵字學習（MeCab） | `keywordEnabled` |
| `noting` | 隨機發文 | `notingEnabled` |
| `poll` | 隨機投票 | - |

### 提醒/通知模組

| 模組 | 功能 |
|------|------|
| `reminder` | 待辦提醒 |
| `timer` | 計時器 |
| `birthday` | 生日祝福 |
| `valentine` | 情人節巧克力 |
| `sleep-report` | Bot 休眠報告 |
| `welcome` | 歡迎新使用者 |

### 占卜模組

| 模組 | 功能 |
|------|------|
| `fortune` | 運勢占卜 |

---

## 重點模組詳解

### core 模組 ([src/modules/core/index.ts](../src/modules/core/index.ts))

提供基礎互動功能。

#### 功能列表

| 功能 | 觸發詞 | 說明 |
|------|--------|------|
| 設定暱稱 | `〇〇って呼んで` | 設定藍對使用者的稱呼 |
| 記憶轉移（開始） | `引継ぎ`, `引っ越し` | 產生轉移碼 |
| 記憶轉移（完成） | `「〇〇」` | 使用轉移碼 |
| 查詢版本 | `version`, `バージョン` | 顯示 Bot 版本 |
| 列出模組 | `modules` | 顯示已安裝模組 |

#### 實作重點 - 使用 Context 進行多輪對話

```typescript
// 設定暱稱時，使用 context 確認是否加敬稱
private setName(msg: Message): boolean {
    if (!msg.text.includes('って呼んで')) return false;
    
    const name = msg.extractedText.match(/\s*(.+?)って呼んで/)![1];
    
    const withSan = titles.some(t => name.endsWith(t));
    
    if (withSan) {
        // 已有敬稱，直接儲存
        msg.friend.updateName(name);
        msg.reply(serifs.core.setNameOk(name));
    } else {
        // 詢問是否加「さん」
        msg.reply(serifs.core.san).then(reply => {
            // 訂閱使用者的回覆
            this.subscribeReply(msg.userId, msg.isChat, 
                msg.isChat ? msg.userId : reply.id,
                { name: name }
            );
        });
    }
    return true;
}

// contextHook 處理使用者的回答
private async contextHook(key: any, msg: Message, data: any) {
    if (msg.text.includes('はい')) {
        msg.friend.updateName(data.name + 'さん');
        this.unsubscribeReply(key);
        msg.reply(serifs.core.setNameOk(msg.friend.name));
    } else if (msg.text.includes('いいえ')) {
        msg.friend.updateName(data.name);
        this.unsubscribeReply(key);
        msg.reply(serifs.core.setNameOk(msg.friend.name));
    } else {
        // 無法理解回答，重新詢問
        msg.reply(serifs.core.yesOrNo).then(reply => {
            this.subscribeReply(msg.userId, msg.isChat, reply.id, data);
        });
    }
}
```

---

### aichat 模組 ([src/modules/aichat/index.ts](../src/modules/aichat/index.ts))

整合 Google Gemini 和 PLaMo API 的 AI 聊天功能。

#### 功能特色

- 支援多種 AI 模型切換（`&gemini`, `&plamo`, `&gemini-pro`, `&gemini-flash`）
- 支援檔案附件（圖片、PDF、音訊、影片）
- URL 預覽整合
- 對話歷史記錄（最多 10 輪）
- Google Search Grounding 支援（`ggg` 關鍵字）
- 隨機對話功能（主動與使用者互動）

#### 實作重點

```typescript
// 處理 AI 聊天
private async handleAiChat(exist: AiChatHist, msg: Message) {
    // 1. 建立 prompt
    let prompt = config.prompt || '';
    const now = new Date().toLocaleString('ja-JP', { ... });
    let systemInstructionText = prompt + '現在日時は' + now + '...';
    
    // 2. 處理 URL 預覽
    const urlarray = [...aiChat.question.matchAll(urlexp)];
    for (const url of urlarray) {
        const urlpreview = await urlToJson(url[0]);
        systemInstructionText += '補足URL情報...' + urlpreview.title;
    }
    
    // 3. 處理檔案附件
    const base64Files = await this.note2base64File(msg.id);
    
    // 4. 建立對話歷史
    let contents: GeminiContents[] = [];
    if (aiChat.history) {
        aiChat.history.forEach(entry => {
            contents.push({ role: entry.role, parts: [{text: entry.content}] });
        });
    }
    
    // 5. 呼叫 API
    const res_data = await got.post(options).json();
    
    // 6. 回覆並訂閱後續對話
    msg.reply(serifs.aichat.post(text, exist.type)).then(reply => {
        // 更新歷史記錄
        exist.history.push({ role: 'user', content: question });
        exist.history.push({ role: 'model', content: text });
        
        // 儲存到資料庫
        this.aichatHist.insertOne({ postId: reply.id, ... });
        
        // 訂閱回覆以繼續對話
        this.subscribeReply(reply.id, reply.id);
        
        // 設定超時（30 分鐘後結束對話）
        this.setTimeoutWithPersistence(TIMEOUT_TIME, { id: reply.id });
    });
}
```

---

### reversi 模組 ([src/modules/reversi/index.ts](../src/modules/reversi/index.ts))

黑白棋遊戲，使用獨立的子程序處理 AI 思考。

#### 架構設計

```
主程序 (index.ts)              子程序 (back.ts)
     │                              │
     │ fork()                       │
     │──────────────────────────────▶
     │                              │
     │   send({type: '_init_'})     │
     │──────────────────────────────▶
     │                              │
     │                              ▼
     │                         ┌─────────────┐
     │                         │ AI 思考引擎  │
     │                         │ (engine.ts) │
     │                         └─────────────┘
     │                              │
     │   send({type: 'putStone'})   │
     │◀──────────────────────────────
     │                              │
```

#### 分離程序的原因

由於 AI 思考可能耗時較長，如果在主程序中進行會阻塞 WebSocket 連線。因此使用 `child_process.fork()` 建立獨立子程序。

#### 遊戲引擎 ([src/modules/reversi/engine.ts](../src/modules/reversi/engine.ts))

- **αβ 剪枝演算法**：用於決定最佳走法
- **靜態評估**：考慮角位、邊緣位置等因素
- **可調整強度**：0（接待模式）~ 5（最強）
- **Undo 機制**：支援回溯以進行深度搜索

---

### reminder 模組 ([src/modules/reminder/index.ts](../src/modules/reminder/index.ts))

待辦提醒功能，展示持久化定時器的典型用法。

#### 使用方式

```
使用者: "@ai remind 買牛奶"
藍: 🆗
(12 小時後)
藍: "使用者，這做了嗎？"
使用者: "done"
藍: "做得很好！"
```

#### 實作重點

```typescript
private async mentionHook(msg: Message) {
    // 建立提醒記錄
    const remind = this.reminds.insertOne({
        id: msg.id,
        userId: msg.userId,
        thing: thing,
        quoteId: msg.quoteId,
        times: 0,
        createdAt: Date.now(),
    });
    
    // 訂閱回覆
    this.subscribeReply(remind.id, msg.isChat, msg.id, { id: remind.id });
    
    // 設定 12 小時後提醒
    this.setTimeoutWithPersistence(1000 * 60 * 60 * 12, { id: remind.id });
    
    return { reaction: '🆗', immediate: true };
}

private async timeoutCallback(data) {
    const remind = this.reminds.findOne({ id: data.id });
    
    // 發送提醒
    const reply = await this.ai.post({
        renoteId: remind.id,
        text: acct(friend.doc.user) + ' ' + serifs.reminder.notify(friend.name)
    });
    
    // 重新訂閱並設定下次提醒
    this.subscribeReply(remind.id, remind.isChat, reply.id, { id: remind.id });
    this.setTimeoutWithPersistence(NOTIFY_INTERVAL, { id: remind.id });
}
```

---

### talk 模組 ([src/modules/talk/index.ts](../src/modules/talk/index.ts))

日常對話處理，根據親愛度給予不同回應。

#### 親愛度相關回應範例

```typescript
private nadenade(msg: Message): boolean {
    if (!msg.includes(['なでなで'])) return false;
    
    // 根據親愛度選擇不同回應
    msg.reply(getSerif(
        msg.friend.love >= 10 ? serifs.core.nadenade.love3 :  // 好感度高
        msg.friend.love >= 5 ? serifs.core.nadenade.love2 :   // 有好感
        msg.friend.love <= -15 ? serifs.core.nadenade.hate4 : // 非常討厭
        msg.friend.love <= -10 ? serifs.core.nadenade.hate3 : // 討厭
        msg.friend.love <= -5 ? serifs.core.nadenade.hate2 :  // 有點討厭
        msg.friend.love <= -1 ? serifs.core.nadenade.hate1 :  // 略微不滿
        serifs.core.nadenade.normal                           // 普通
    ));
    
    return true;
}
```

#### 親愛度變化

- 每日問候、摸頭都會增加親愛度
- 說「ぽんこつ」或「rm -rf」會減少親愛度

---

### fortune 模組 ([src/modules/fortune/index.ts](../src/modules/fortune/index.ts))

運勢占卜，使用 seedrandom 確保同一天同一使用者結果一致。

```typescript
private async mentionHook(msg: Message) {
    if (msg.includes(['占', 'うらな', '運勢', 'おみくじ'])) {
        const date = new Date();
        // 使用日期 + 使用者 ID 作為種子
        const seed = `${date.getFullYear()}/${date.getMonth()}/${date.getDate()}@${msg.userId}`;
        const rng = seedrandom(seed);
        
        // 產生固定結果
        const omikuji = blessing[Math.floor(rng() * blessing.length)];
        const item = genItem(rng);
        
        msg.reply(`**${omikuji}🎉**\nラッキーアイテム: ${item}`, {
            cw: serifs.fortune.cw(msg.friend.name)
        });
        return true;
    }
    return false;
}
```

---

### maze 模組 ([src/modules/maze/index.ts](../src/modules/maze/index.ts))

迷宮生成與渲染。

#### 架構

```
index.ts
├── gen-maze.ts   # 迷宮生成演算法
├── render-maze.ts # 迷宮渲染（使用 canvas）
├── maze.ts       # 迷宮格子型別定義
└── themes.ts     # 顏色主題
```

#### 難度等級

| 難度 | 觸發詞 | 迷宮大小 |
|------|--------|----------|
| 接待 | `接待` | 3~5 |
| 簡單 | `簡単`, `かんたん` | 8~15 |
| 普通 | - | 11~31 |
| 困難 | `難しい`, `大きい` | 22~34 |
| 極難 | `死`, `鬼`, `地獄` | 40~59 |
| 藍本氣 | `藍` + `本気` | 100 |

---

### server 模組 ([src/modules/server/index.ts](../src/modules/server/index.ts))

伺服器監控，當 CPU 使用率過高時發出警告。

```typescript
private check() {
    // 計算最近 60 秒的平均 CPU 使用率
    const cpuPercentages = this.statsLogs.map(s => s && (s.cpu_usage || s.cpu) * 100 || 0);
    const cpuPercentage = average(cpuPercentages);
    
    if (cpuPercentage >= 70) {
        this.warn();  // 發出警告
    } else if (cpuPercentage <= 30) {
        this.warned = false;  // 重設警告狀態
    }
}
```

---

### emoji-react 模組 ([src/modules/emoji-react/index.ts](../src/modules/emoji-react/index.ts))

自動對時間軸上的貼文添加反應。

#### 反應規則

1. **自訂表情符號**：如果貼文只有一種自訂表情符號，用相同表情符號反應
2. **Unicode 表情符號**：如果只有一種，用相同表情符號反應
3. **猜拳**：✊→🖐、✌→✊、🖐→✌
4. **特定詞彙**：
   - `ぴざ` → 🍕
   - `ぷりん` → 🍮
   - `寿司` → 🍣
   - `藍` → 🙌

---

## 台詞系統 ([src/serifs.ts](../src/serifs.ts))

所有機器人的對話內容都集中在 `serifs.ts` 中管理。

### 結構

```typescript
export default {
    core: {
        setNameOk: name => `わかりました。これからは${name}とお呼びしますね！`,
        hello: name => name ? `こんにちは、${name}♪` : `こんにちは♪`,
        nadenade: {
            normal: 'ひゃっ…！ びっくりしました',
            love2: ['わわっ… 恥ずかしいです', 'あうぅ… 恥ずかしい…', 'ふやぁ…？'],
            // 陣列表示隨機選擇
        },
        // ...
    },
    fortune: { ... },
    reversi: { ... },
    // 各模組的台詞
};
```

### getSerif 函式

```typescript
export function getSerif(variant: string | string[]): string {
    if (Array.isArray(variant)) {
        return variant[Math.floor(Math.random() * variant.length)];
    } else {
        return variant;
    }
}
```

---

## 詞彙系統 ([src/vocabulary.ts](../src/vocabulary.ts))

隨機物品生成器，用於占卜、投票等功能。

### genItem 函式

```typescript
export function genItem(seedOrRng?: (() => number) | string | number) {
    const rng = seedOrRng
        ? typeof seedOrRng === 'function'
            ? seedOrRng
            : seedrandom(seedOrRng.toString())
        : Math.random;
    
    let item = '';
    // 80% 機率加前綴
    if (Math.floor(rng() * 5) !== 0) 
        item += itemPrefixes[Math.floor(rng() * itemPrefixes.length)];
    item += items[Math.floor(rng() * items.length)];
    // 10% 機率加連接詞和第二個物品
    if (Math.floor(rng() * 10) === 0) {
        item += and[Math.floor(rng() * and.length)];
        // ...
    }
    return item;
}
```

### 範例輸出

- `プラチナ製ナス`
- `古代の量子コンピューター`
- `反重力メガネに擬態した放射性廃棄物`

---

## 下一章

接下來請閱讀 [04-utilities.md](04-utilities.md) 了解工具函式庫。
