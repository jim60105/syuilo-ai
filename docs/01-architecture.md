# 01 - 系統架構

本章節詳細說明藍的系統架構設計，包含應用程式啟動流程、資料流向、以及各元件之間的關係。

## 架構概覽

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Misskey Server                             │
│                                                                       │
│   ┌───────────────┐         ┌───────────────┐                        │
│   │   REST API    │         │   WebSocket   │                        │
│   │   (notes/*)   │         │   Streaming   │                        │
│   └───────┬───────┘         └───────┬───────┘                        │
└───────────┼─────────────────────────┼───────────────────────────────┘
            │                         │
            │ HTTP (got)              │ WS (reconnecting-websocket)
            │                         │
┌───────────┼─────────────────────────┼───────────────────────────────┐
│           │         藍 (Ai) Bot      │                               │
│           ▼                         ▼                               │
│   ┌───────────────┐         ┌───────────────┐                       │
│   │    ai.ts      │◄────────│   stream.ts   │                       │
│   │   (藍 Class)  │         │   (Stream)    │                       │
│   └───────┬───────┘         └───────────────┘                       │
│           │                                                          │
│           │ 註冊 & 呼叫                                              │
│           ▼                                                          │
│   ┌───────────────────────────────────────────────────────────────┐ │
│   │                      Module System                              │ │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │ │
│   │  │  core   │ │ aichat  │ │ reversi │ │ fortune │ ...          │ │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘              │ │
│   └───────────────────────────────────────────────────────────────┘ │
│           │                                                          │
│           ▼                                                          │
│   ┌───────────────┐                                                  │
│   │   LokiJS DB   │◄───────► memory.json                            │
│   └───────────────┘                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## 啟動流程

應用程式啟動時的完整流程如下：

### 1. 入口點 ([src/index.ts](../src/index.ts))

```typescript
// 1. 顯示啟動橫幅
console.log('   __    ____  _____  ___ ');
console.log('  /__\\  (_  _)(  _  )/ __)');
// ...

// 2. 透過 API 取得帳號資訊
promiseRetry(retry => {
    return got.post(`${config.apiUrl}/i`, {
        json: { i: config.i }
    }).json().catch(retry);
}, { retries: 3 })

// 3. 建立藍實例，傳入帳號和模組列表
new 藍(account, [
    new CoreModule(),
    new AiChatModule(),
    // ... 其他模組
]);
```

**重點**：模組陣列的順序決定處理優先度。陣列前面的模組會優先處理訊息。

### 2. 核心初始化 ([src/ai.ts](../src/ai.ts))

```typescript
constructor(account: User, modules: Module[]) {
    // 1. 載入 LokiJS 資料庫
    this.db = new loki(file, {
        autoload: true,
        autosave: true,
        autosaveInterval: 1000,
        autoloadCallback: err => {
            if (!err) this.run();
        }
    });
}

private run() {
    // 2. 初始化資料庫集合
    this.meta = this.getCollection('meta', {});
    this.contexts = this.getCollection('contexts', { indices: ['key'] });
    this.timers = this.getCollection('timers', { indices: ['module'] });
    this.friends = this.getCollection('friends', { indices: ['userId'] });
    this.moduleData = this.getCollection('moduleData', { indices: ['module'] });

    // 3. 建立 WebSocket 連線
    this.connection = new Stream();

    // 4. 訂閱主要串流事件
    const mainStream = this.connection.useSharedConnection('main');
    mainStream.on('mention', data => this.onReceiveMessage(...));
    mainStream.on('reply', data => this.onReceiveMessage(...));
    mainStream.on('renote', data => ...);
    mainStream.on('notification', data => this.onNotification(...));
    mainStream.on('newChatMessage', data => ...);

    // 5. 安裝所有模組
    this.modules.forEach(m => {
        m.init(this);
        const res = m.install();
        // 註冊 Hook
    });

    // 6. 啟動定時器監控
    setInterval(this.crawleTimer, 1000);
}
```

## 資料流程

### 訊息處理流程

當使用者提及或回覆藍時，訊息依序經過以下處理：

```
使用者訊息
    │
    ▼
┌─────────────────┐
│  WebSocket      │  mainStream.on('mention') / mainStream.on('reply')
│  串流接收       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  建立 Message   │  new Message(this, data, false)
│  物件           │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  檢查 Context   │  是否有進行中的對話上下文？
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
有 Context   無 Context
    │              │
    ▼              ▼
┌──────────┐  ┌──────────────┐
│ 呼叫該   │  │ 依序呼叫所有 │
│ 模組的   │  │ mentionHook  │
│ contextHook │  │ 直到有人處理 │
└────┬─────┘  └──────┬───────┘
     │               │
     └───────┬───────┘
             ▼
     ┌───────────────┐
     │  等待 1 秒    │  sleep(1000) - 模擬思考
     └───────┬───────┘
             ▼
     ┌───────────────┐
     │  送出反應     │  notes/reactions/create
     └───────────────┘
```

### Hook 系統

藍使用 Hook 模式來處理各種事件：

| Hook 類型 | 觸發時機 | 回傳值意義 |
|-----------|----------|------------|
| `mentionHook` | 收到新的提及/回覆時 | `true` 或 `HandlerResult` 表示已處理 |
| `contextHook` | 收到對已訂閱對話的回覆 | `false` 表示繼續嘗試其他 Hook |
| `timeoutCallback` | 定時器到期時 | 無回傳值 |

## 持久化機制

### LokiJS 資料庫

藍使用 LokiJS 作為嵌入式 NoSQL 資料庫，特點如下：

- **記憶體內運作**：所有資料存放在記憶體中，存取速度快
- **自動儲存**：每秒自動將資料寫入 `memory.json`
- **索引支援**：支援建立索引以加速查詢

### 資料集合 (Collections)

| 集合名稱 | 用途 | 索引 |
|----------|------|------|
| `meta` | 系統 metadata（如最後活動時間） | - |
| `contexts` | 對話上下文追蹤 | `key` |
| `timers` | 持久化定時器 | `module` |
| `friends` | 好友/親愛度資料 | `userId` |
| `moduleData` | 各模組的私有資料 | `module` |

## WebSocket 連線管理

### Stream 類別架構

```
Stream
├── sharedConnectionPools[]    # 共享連線池（如 main, homeTimeline）
│   └── Pool
│       ├── channel: string
│       ├── id: string
│       ├── users: number      # 使用者計數
│       └── connect()/disconnect()
│
├── sharedConnections[]        # 共享連線實例
│   └── SharedConnection
│       └── 共用同一 Pool 的 ID
│
└── nonSharedConnections[]     # 非共享連線（如 reversiGame）
    └── NonSharedConnection
        └── 獨立的連線 ID
```

### 連線類型比較

| 類型 | 使用場景 | 特點 |
|------|----------|------|
| SharedConnection | `main`, `homeTimeline`, `localTimeline` 等 | 多個監聽器共用同一連線，節省資源 |
| NonSharedConnection | `reversiGame`, `chatUser` 等 | 每次建立獨立連線，有獨立參數 |

### 自動重連機制

Stream 類別使用 `reconnecting-websocket` 套件實現自動重連：

```typescript
this.stream = new ReconnectingWebsocket(
    `${config.wsUrl}/streaming?i=${config.i}`,
    [],
    { WebSocket: WebSocket }
);
```

當連線斷開時：
1. 狀態變更為 `'reconnecting'`
2. 自動嘗試重新連線
3. 連線恢復後重新訂閱所有頻道

## 設定系統

### 設定結構 ([src/config.ts](../src/config.ts))

```typescript
type Config = {
    host: string;              // Misskey 實例 URL
    serverName?: string;       // 伺服器名稱（顯示用）
    i: string;                 // Bot API Token
    master?: string;           // 管理員使用者名稱
    wsUrl: string;             // WebSocket URL（自動產生）
    apiUrl: string;            // API URL（自動產生）
    
    // 功能開關
    keywordEnabled: boolean;
    reversiEnabled: boolean;
    notingEnabled: boolean;
    chartEnabled: boolean;
    serverMonitoring: boolean;
    checkEmojisEnabled?: boolean;
    checkEmojisAtOnce?: boolean;
    
    // AI 聊天設定
    geminiProApiKey?: string;
    pLaMoApiKey?: string;
    prompt?: string;
    aichatRandomTalkEnabled?: boolean;
    aichatRandomTalkProbability?: string;
    aichatRandomTalkIntervalMinutes?: string;
    aichatGroundingWithGoogleSearchAlwaysEnabled?: boolean;
    
    // MeCab 設定
    mecab?: string;
    mecabDic?: string;
    
    // 資料目錄
    memoryDir?: string;
};
```

### 設定檔載入

設定從 `config.json` 載入，並自動計算衍生值：

```typescript
import config from '../config.json' with { type: 'json' };

config.wsUrl = config.host.replace('http', 'ws');
config.apiUrl = config.host + '/api';
```

## 路徑別名

專案使用 TypeScript 路徑映射簡化 import：

```json
// tsconfig.json
{
    "paths": {
        "@/*": ["./src/*"]
    }
}
```

使用方式：
```typescript
import config from '@/config.js';  // 實際對應 ./src/config.js
import Module from '@/module.js';   // 實際對應 ./src/module.js
```

編譯時由 `typescript-transform-paths` 套件轉換為實際路徑。

## Docker 部署

### Dockerfile 結構

- 基於 Node.js 映像
- 可選擇性安裝 MeCab（透過 `enable_mecab` 建構參數）
- 將 `memory.json` 儲存在獨立 volume

### docker-compose.yml 使用

```yaml
version: '3'
services:
  ai:
    build:
      context: .
      args:
        enable_mecab: 'false'  # 設為 'true' 啟用 MeCab
    volumes:
      - ./data:/app/data       # 持久化 memory.json
      - ./config.json:/app/config.json:ro
```

## 下一章

接下來請閱讀 [02-core-classes.md](02-core-classes.md) 了解核心類別的詳細實作。
