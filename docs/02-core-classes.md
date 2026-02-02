# 02 - 核心類別詳解

本章節深入剖析藍的核心類別，包含其公開方法、屬性以及設計理念。

## 藍 Class ([src/ai.ts](../src/ai.ts))

`藍` 是整個應用程式的核心類別，負責管理所有子系統並協調模組間的通訊。

### 類別定義

```typescript
export default class 藍 {
    public readonly version = pkg._v;     // 版本號
    public account: User;                  // Bot 帳號資訊
    public connection: Stream;             // WebSocket 連線
    public modules: Module[] = [];         // 已安裝模組
    public db: loki;                       // LokiJS 資料庫實例
    public lastSleepedAt: number;          // 上次休眠時間
    
    // 資料庫集合
    public friends: loki.Collection<FriendDoc>;
    public moduleData: loki.Collection<any>;
    
    // 私有成員
    private mentionHooks: MentionHook[] = [];
    private contextHooks: { [moduleName: string]: ContextHook } = {};
    private timeoutCallbacks: { [moduleName: string]: TimeoutCallback } = {};
    private meta: loki.Collection<Meta>;
    private contexts: loki.Collection<...>;
    private timers: loki.Collection<...>;
}
```

### 公開方法

#### `constructor(account: User, modules: Module[])`

建立藍實例。

**參數**：
- `account`: Bot 在 Misskey 上的帳號資訊
- `modules`: 要安裝的模組陣列（前面的模組優先處理訊息）

**行為**：
1. 儲存帳號資訊和模組列表
2. 初始化 LokiJS 資料庫
3. 資料庫載入完成後呼叫 `run()` 啟動 Bot

---

#### `log(msg: string): void`

輸出帶有時間戳記和 `[AiOS]` 前綴的日誌。

```typescript
this.log('Starting modules...');
// 輸出: 12:34:56 [AiOS]: Starting modules...
```

---

#### `getCollection(name: string, opts?: any): loki.Collection`

取得或建立資料庫集合。

**參數**：
- `name`: 集合名稱
- `opts`: LokiJS 集合選項（如 `{ indices: ['key'] }`）

**回傳**：LokiJS Collection 實例

**使用範例**：
```typescript
this.contexts = this.getCollection('contexts', {
    indices: ['key']
});
```

---

#### `lookupFriend(userId: string): Friend | null`

根據使用者 ID 查找好友。

**參數**：
- `userId`: Misskey 使用者 ID

**回傳**：找到則回傳 `Friend` 實例，否則回傳 `null`

---

#### `upload(file: Buffer | fs.ReadStream, meta: { filename: string, contentType: string }): Promise<any>`

上傳檔案到 Misskey Drive。

**參數**：
- `file`: 檔案內容（Buffer 或 ReadStream）
- `meta`: 檔案元資料
  - `filename`: 檔案名稱
  - `contentType`: MIME 類型

**回傳**：上傳後的檔案資訊

**使用範例**：
```typescript
const file = await this.ai.upload(imageBuffer, {
    filename: 'chart.png',
    contentType: 'image/png'
});
```

---

#### `post(param: any): Promise<any>`

發布貼文（Note）。

**參數**：
- `param`: 貼文參數物件，可包含：
  - `text`: 貼文內容
  - `cw`: Content Warning
  - `fileIds`: 附件檔案 ID 陣列
  - `replyId`: 回覆的貼文 ID
  - `renoteId`: 引用的貼文 ID
  - `visibility`: 可見性 (`public`, `home`, `followers`, `specified`)
  - `poll`: 投票選項

**回傳**：建立的貼文物件

**使用範例**：
```typescript
await this.ai.post({
    text: '早安！',
    visibility: 'home'
});
```

---

#### `sendMessage(userId: string, param: any): Promise<any>`

發送私訊（Chat Message）給指定使用者。

**參數**：
- `userId`: 目標使用者 ID
- `param`: 訊息參數
  - `text`: 訊息內容
  - `fileId`: 附件檔案 ID

**使用範例**：
```typescript
await this.ai.sendMessage(friend.userId, {
    text: '生日快樂！🎉'
});
```

---

#### `api(endpoint: string, param?: any): Promise<any>`

呼叫 Misskey API。

**參數**：
- `endpoint`: API 端點（如 `'notes/create'`）
- `param`: API 參數

**回傳**：API 回應

**使用範例**：
```typescript
const note = await this.ai.api('notes/show', {
    noteId: 'xxxxxxxxxx'
});
```

---

#### `subscribeReply(module: Module, key: string | null, isChat: boolean, id: string, data?: any): void`

訂閱回覆，建立對話上下文。

**參數**：
- `module`: 發起訂閱的模組
- `key`: 上下文識別鍵（可為 `null`）
- `isChat`: 是否為私訊對話
- `id`: 私訊對話為使用者 ID，一般對話為貼文 ID
- `data`: 附加的上下文資料

**運作原理**：
訂閱後，當使用者回覆該貼文（或在私訊中繼續對話）時，會呼叫該模組的 `contextHook` 而非 `mentionHook`。

**使用範例**：
```typescript
// 在訊息處理中訂閱回覆
msg.reply('你想要哪一個？').then(reply => {
    this.subscribeReply(reply.id, msg.userId, { options: ['A', 'B', 'C'] });
});
```

---

#### `unsubscribeReply(module: Module, key: string | null): void`

取消訂閱回覆，移除對話上下文。

**參數**：
- `module`: 模組實例
- `key`: 上下文識別鍵

---

#### `setTimeoutWithPersistence(module: Module, delay: number, data?: any): void`

設定持久化定時器。

**參數**：
- `module`: 模組實例
- `delay`: 延遲毫秒數
- `data`: 定時器到期時傳遞給回呼的資料

**特點**：
- 定時器資訊儲存在資料庫中
- Bot 重啟後定時器仍然有效
- 適用於提醒、遊戲超時等場景

---

#### `getMeta(): Meta`

取得系統 metadata。

**回傳**：
```typescript
{
    lastWakingAt: number  // 上次活動時間戳記
}
```

---

#### `setMeta(meta: Partial<Meta>): void`

更新系統 metadata。

---

## Message Class ([src/message.ts](../src/message.ts))

封裝接收到的訊息，提供統一的存取介面。

### 類別定義

```typescript
export default class Message {
    private ai: 藍;
    private chatMessage: { ... } | null;  // 私訊資料
    private note: { ... } | null;          // 貼文資料
    public isChat: boolean;                // 是否為私訊
    public friend: Friend;                 // 訊息發送者的 Friend 實例
}
```

### 屬性 (Getters)

| 屬性 | 型別 | 說明 |
|------|------|------|
| `id` | `string` | 訊息 ID |
| `user` | `User` | 發送者資訊 |
| `userId` | `string` | 發送者 ID |
| `text` | `string` | 原始訊息文字 |
| `quoteId` | `string \| null` | 引用貼文 ID |
| `replyId` | `string \| null` | 回覆貼文 ID |
| `visibility` | `string \| null` | 可見性設定 |
| `extractedText` | `string` | 移除提及後的文字 |

### `extractedText` 說明

`extractedText` 會移除訊息開頭的 Bot 提及，便於解析使用者的實際輸入：

```typescript
// 原始訊息: "@ai@example.com 占卜"
// extractedText: "占卜"
```

### 公開方法

#### `reply(text: string | null, opts?: { file?: any, cw?: string, renote?: string, immediate?: boolean }): Promise<any>`

回覆此訊息。

**參數**：
- `text`: 回覆內容（`null` 時不回覆）
- `opts`: 選項
  - `file`: 附件檔案物件
  - `cw`: Content Warning
  - `renote`: 引用貼文 ID
  - `immediate`: 是否立即回覆（不等待 2 秒）

**回傳**：建立的回覆貼文

**使用範例**：
```typescript
await msg.reply('好的！', { immediate: true });

// 附帶檔案
const file = await this.ai.upload(imgBuffer, { ... });
await msg.reply('這是圖片', { file });
```

---

#### `includes(words: string[]): boolean`

檢查訊息是否包含指定詞彙（任一即可）。

**特點**：
- 自動進行日文片假名/平假名轉換
- 自動進行半形/全形轉換
- 不分大小寫

**使用範例**：
```typescript
if (msg.includes(['占', 'うらな', '運勢', 'おみくじ'])) {
    // 處理占卜請求
}
```

---

#### `or(words: (string | RegExp)[]): boolean`

檢查訊息是否嚴格匹配指定詞彙（支援正規表達式）。

**特點**：
- 會移除訊息中的裝飾性文字（如 `!`、`♪` 等）
- 適用於需要精確匹配的場景

**使用範例**：
```typescript
if (msg.or(['好き', 'すき'])) {
    // 處理「喜歡」的回應
}

if (msg.or([/^はぐ(し(て|よ|よう)?)?$/])) {
    // 匹配「ハグ」、「はぐして」等
}
```

---

## Friend Class ([src/friend.ts](../src/friend.ts))

管理使用者的好友關係和親愛度系統。

### 類別定義

```typescript
export type FriendDoc = {
    userId: string;
    user: User;
    name?: string | null;           // 暱稱
    love?: number;                  // 親愛度 (-30 ~ 100)
    lastLoveIncrementedAt?: string; // 上次增加親愛度的日期
    todayLoveIncrements?: number;   // 今日增加次數
    perModulesData?: any;           // 各模組的私有資料
    married?: boolean;              // 結婚狀態
    transferCode?: string;          // 轉移碼
    reversiStrength?: number | null; // 黑白棋強度設定
};

export default class Friend {
    private ai: 藍;
    public doc: FriendDoc;
    
    public get userId(): string;
    public get name(): string | null;
    public get love(): number;
    public get married(): boolean;
}
```

### 親愛度系統

親愛度範圍：`-30` ~ `100`

| 親愛度範圍 | 狀態說明 |
|------------|----------|
| 100 | 最大值（不會再下降） |
| 15+ | 深度信賴 |
| 7+ | 友好關係 |
| 5+ | 好感 |
| 0 | 中立 |
| -1 ~ -5 | 略有不滿 |
| -5 ~ -10 | 不滿 |
| -10 ~ -15 | 厭惡 |
| -15 ~ -30 | 極度厭惡 |

### 公開方法

#### `constructor(ai: 藍, opts: { user?: User, doc?: FriendDoc })`

建立 Friend 實例。

**參數**：
- 透過 `user` 建立：會自動在資料庫中查找或建立記錄
- 透過 `doc` 建立：直接使用現有的資料庫文件

---

#### `updateUser(user: Partial<User>): void`

更新使用者資訊。

---

#### `getPerModulesData(module: Module): any`

取得該模組對此好友的私有資料。

**使用範例**：
```typescript
const data = friend.getPerModulesData(this);
if (data.lastGreetedAt === today) return;
```

---

#### `setPerModulesData(module: Module, data: any): void`

設定該模組對此好友的私有資料。

**使用範例**：
```typescript
data.lastGreetedAt = today;
friend.setPerModulesData(this, data);
```

---

#### `incLove(amount: number = 1): void`

增加親愛度。

**限制**：
- 每日最多增加 3 點
- 最大值為 100

---

#### `decLove(amount: number = 1): void`

減少親愛度。

**特點**：
- 親愛度為 100 時不會減少
- 最小值為 -30
- 親愛度負值時會忘記暱稱

---

#### `updateName(name: string): void`

更新暱稱。

---

#### `updateReversiStrength(strength: number | null): void`

更新黑白棋強度設定（0~5）。

---

#### `generateTransferCode(): string`

產生記憶轉移碼，用於在不同帳號間轉移好友資料。

---

#### `transferMemory(code: string): boolean`

使用轉移碼接收另一好友的記憶。

---

## Stream Class ([src/stream.ts](../src/stream.ts))

管理與 Misskey 的 WebSocket 連線。

### 公開方法

#### `useSharedConnection(channel: string): SharedConnection`

取得或建立共享連線。

**參數**：
- `channel`: 頻道名稱（如 `'main'`, `'homeTimeline'`）

**使用範例**：
```typescript
const mainStream = this.connection.useSharedConnection('main');
mainStream.on('mention', data => { ... });
```

---

#### `connectToChannel(channel: string, params?: any): NonSharedConnection`

建立非共享連線（帶參數）。

**參數**：
- `channel`: 頻道名稱
- `params`: 連線參數

**使用範例**：
```typescript
const gameStream = this.connection.connectToChannel('reversiGame', {
    gameId: game.id
});
```

---

## 裝飾器 ([src/decorators.ts](../src/decorators.ts))

### `@bindThis`

自動綁定方法的 `this` 參考。

**用途**：確保方法在作為回呼函式傳遞時，`this` 仍指向正確的實例。

**使用範例**：
```typescript
class MyModule extends Module {
    @bindThis
    private async mentionHook(msg: Message) {
        // this 永遠指向 MyModule 實例
        this.log('Message received');
    }
}
```

**原理**：
- 第一次存取屬性時，建立綁定版本的函式
- 將綁定版本存儲在實例上
- 後續存取直接回傳綁定版本

## 型別定義

### HandlerResult ([src/ai.ts](../src/ai.ts))

```typescript
export type HandlerResult = {
    reaction?: string | null;  // 要添加的反應，null 表示不反應
    immediate?: boolean;       // 是否立即回覆
};
```

### InstallerResult ([src/ai.ts](../src/ai.ts))

```typescript
export type InstallerResult = {
    mentionHook?: MentionHook;       // 提及處理函式
    contextHook?: ContextHook;        // 上下文處理函式
    timeoutCallback?: TimeoutCallback; // 定時器回呼
};
```

### User ([src/misskey/user.ts](../src/misskey/user.ts))

```typescript
export type User = {
    id: string;
    name: string;
    username: string;
    host?: string | null;
    isFollowing?: boolean;
    isBot: boolean;
};
```

### Note ([src/misskey/note.ts](../src/misskey/note.ts))

```typescript
export type Note = {
    id: string;
    text: string | null;
    reply: any | null;
    poll?: {
        choices: { votes: number; text: string; }[];
        expiredAfter: number;
        multiple: boolean;
    } | null;
};
```

## 下一章

接下來請閱讀 [03-module-system.md](03-module-system.md) 了解模組系統的設計與開發方法。
