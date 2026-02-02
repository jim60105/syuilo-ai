# 00 - 專案概覽

## 藍 (Ai) - Misskey Bot 專案技術指南

歡迎來到**藍 (Ai)** 專案的技術維護指南。本文件將全面介紹這個 Misskey 社群網路平台 AI 機器人的架構設計、程式碼結構以及維護方法。

## 專案基本資訊

| 項目 | 內容 |
|------|------|
| 專案名稱 | 藍 (Ai) |
| 版本 | 2.0.1 |
| 授權 | MIT |
| 程式語言 | TypeScript |
| 執行環境 | Node.js (ES2022, ES Modules) |
| 主要用途 | Misskey 社群平台的互動式 AI 機器人 |

## 專案定位

藍是一個設計為 Misskey 平台的吉祥物角色「三須木藍 (Misuki Ai)」，她可以：

- 與使用者進行互動對話
- 玩遊戲（黑白棋、數字遊戲等）
- 提供運勢占卜
- AI 聊天功能（整合 Gemini、PLaMo API）
- 自動執行定時任務
- 提供伺服器監控

## 技術棧

### 核心依賴

| 套件 | 用途 |
|------|------|
| LokiJS | 記憶體內資料庫，JSON 持久化儲存 |
| got | HTTP 請求套件 |
| ws / reconnecting-websocket | WebSocket 連線管理 |
| chalk | 終端機彩色輸出 |
| seedrandom | 可重現的隨機數產生器 |
| node-canvas | 圖片生成 |
| uuid | 唯一識別碼產生 |
| twemoji-parser | Emoji 解析 |

### AI 整合

| 服務 | 用途 |
|------|------|
| Google Gemini API | AI 聊天功能 |
| PLaMo API | 替代 AI 聊天服務 |

### 選用依賴

| 套件 | 用途 |
|------|------|
| MeCab | 日語自然語言處理（關鍵字學習功能） |
| kuromoji | 日語分詞器 |

## 專案結構

```
syuilo-ai/
├── src/                    # 原始碼目錄
│   ├── index.ts           # 應用程式啟動點
│   ├── ai.ts              # 核心 AI 類別
│   ├── module.ts          # 抽象模組基礎類別
│   ├── config.ts          # 設定管理
│   ├── message.ts         # 訊息封裝類別
│   ├── stream.ts          # WebSocket 串流連線
│   ├── friend.ts          # 好友/親愛度系統
│   ├── serifs.ts          # 機器人台詞/對話內容
│   ├── vocabulary.ts      # 詞彙/物品產生器
│   ├── decorators.ts      # TypeScript 裝飾器
│   ├── misskey/           # Misskey 型別定義
│   │   ├── note.ts        # 貼文型別
│   │   └── user.ts        # 使用者型別
│   ├── modules/           # 功能模組目錄（20+ 個模組）
│   │   ├── aichat/        # AI 聊天
│   │   ├── reversi/       # 黑白棋遊戲
│   │   ├── fortune/       # 運勢占卜
│   │   ├── maze/          # 迷宮生成
│   │   └── ...            # 更多模組
│   └── utils/             # 工具函式
├── built/                  # 編譯後的 JavaScript 輸出
├── data/                   # 執行時資料目錄
├── config.json            # 設定檔（需自行建立）
├── memory.json            # 持久化記憶資料庫
├── font.ttf               # 自訂字型（選用）
├── package.json           # 專案配置
├── tsconfig.json          # TypeScript 配置
├── Dockerfile             # Docker 建構檔
└── docker-compose.yml     # Docker Compose 配置
```

## 文件導覽

本技術指南包含以下章節：

| 章節 | 內容概述 |
|------|----------|
| [01-architecture.md](01-architecture.md) | 系統架構與資料流程 |
| [02-core-classes.md](02-core-classes.md) | 核心類別詳解 |
| [03-module-system.md](03-module-system.md) | 模組系統與開發指南 |
| [04-utilities.md](04-utilities.md) | 工具函式庫 |
| [05-adding-features.md](05-adding-features.md) | 新功能開發教學 |

## 快速開始

### 安裝依賴

```bash
npm install
```

### 建構專案

```bash
npm run build
```

### 設定

複製 `example.json` 為 `config.json` 並填入必要設定：

```json
{
  "host": "https://your-misskey-instance.example",
  "i": "your-bot-access-token",
  "master": "admin-username",
  "geminiProApiKey": "your-gemini-api-key"
}
```

### 啟動

```bash
npm start
```

## 重要注意事項

1. **建構必要性**：每次修改程式碼後必須執行 `npm run build`，應用程式是從 `./built/` 目錄執行

2. **記憶體持久化**：`memory.json` 儲存所有執行時資料，包含好友資訊、遊戲狀態等

3. **字型檔案**：迷宮和圖表功能需要在專案根目錄放置 `font.ttf`

4. **API 權限**：某些功能（如自訂表情符號檢查）需要管理員權限的 API Token

## 下一步

建議按順序閱讀文件：

1. 先閱讀 [01-architecture.md](01-architecture.md) 了解整體架構
2. 接著閱讀 [02-core-classes.md](02-core-classes.md) 理解核心類別
3. 然後閱讀 [03-module-system.md](03-module-system.md) 學習模組開發
4. 最後閱讀 [05-adding-features.md](05-adding-features.md) 開始開發新功能
