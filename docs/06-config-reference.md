# 06 - 設定參考手冊

本章節詳細說明 `config.json` 中所有可用的設定項目。

## 設定檔位置

設定檔位於專案根目錄的 `config.json`。可參考 `example.json` 作為範本。

---

## 必要設定

### `host`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | ✓ | - |

Misskey 實例的 URL（不含結尾斜線）。

```json
{
    "host": "https://misskey.io"
}
```

---

### `i`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | ✓ | - |

Bot 帳號的存取權杖。可在 Misskey 的「設定 → API」頁面取得。

```json
{
    "i": "your-access-token-here"
}
```

**權限需求**：

- 查看帳號資訊
- 發表貼文
- 傳送私訊
- 存取雲端硬碟
- 添加表情反應
- 查看通知

---

## 基本設定

### `master`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | - |

管理員的使用者名稱（不含 @）。主人會有特殊互動，例如撒嬌、報告等功能。

```json
{
    "master": "admin"
}
```

---

### `name`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | `'藍'` |

Bot 的名字。用於自我介紹和被提及時的判斷。

```json
{
    "name": "藍"
}
```

---

### `serverName`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | `null` |

伺服器的顯示名稱。用於統計圖表等功能。

```json
{
    "serverName": "Misskey.io"
}
```

---

### `memoryDir`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | `'.'`（專案根目錄） |

`memory.json` 檔案的存放目錄。Docker 環境建議設為 `data`。

```json
{
    "memoryDir": "data"
}
```

---

## 功能開關

### `notingEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `true` |

是否啟用自動發文功能（noting 模組）。

```json
{
    "notingEnabled": true
}
```

---

### `keywordEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `false` |

是否啟用關鍵字學習功能（keyword 模組）。需要安裝 MeCab。

```json
{
    "keywordEnabled": false
}
```

---

### `reversiEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `true` |

是否啟用黑白棋遊戲功能（reversi 模組）。

```json
{
    "reversiEnabled": true
}
```

---

### `chartEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `true` |

是否啟用統計圖表功能（chart 模組）。需要 `font.ttf` 字型檔。

```json
{
    "chartEnabled": true
}
```

---

### `serverMonitoring`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `false` |

是否啟用伺服器監控功能（server 模組）。會監控 CPU 和記憶體使用率。

```json
{
    "serverMonitoring": true
}
```

---

### `checkEmojisEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `false` |

是否啟用自訂表情符號監控功能。會在新增表情符號時通知管理員。

```json
{
    "checkEmojisEnabled": true
}
```

---

## AI 聊天設定

### `geminiProApiKey`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | - |

Google Gemini API 金鑰。用於 AI 聊天功能。

```json
{
    "geminiProApiKey": "your-gemini-api-key"
}
```

**取得方式**：
1. 前往 [Google AI Studio](https://aistudio.google.com/)
2. 建立專案並啟用 Gemini API
3. 建立 API 金鑰

---

### `pLaMoApiKey`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | - |

Preferred Networks PLaMo API 金鑰。提供替代的 AI 聊天服務。

```json
{
    "pLaMoApiKey": "your-plamo-api-key"
}
```

---

### `prompt`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | 內建人設 |

AI 聊天的人格設定提示詞（日文）。定義藍的說話方式和個性。

```json
{
    "prompt": "あなたは「藍」という名前のAIアシスタントです。可愛らしく、親切に応答してください。"
}
```

---

### `aichatRandomTalkEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `false` |

是否啟用隨機對話功能。啟用後藍會對時間軸上的貼文隨機回應。

```json
{
    "aichatRandomTalkEnabled": false
}
```

---

### `aichatRandomTalkProbability`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | `'0.01'` |

隨機回應的機率。`'0.01'` 表示 1% 的機率。

```json
{
    "aichatRandomTalkProbability": "0.01"
}
```

---

### `aichatRandomTalkIntervalMinutes`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | `'720'` |

隨機回應的最小間隔時間（分鐘）。避免過於頻繁發言。

```json
{
    "aichatRandomTalkIntervalMinutes": "720"
}
```

---

### `aichatGroundingWithGoogleSearchAlwaysEnabled`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `boolean` | - | `false` |

是否總是啟用 Google 搜尋增強。啟用後 AI 回應會參考網路搜尋結果。

```json
{
    "aichatGroundingWithGoogleSearchAlwaysEnabled": false
}
```

---

## MeCab 設定

MeCab 是日文形態素解析器，用於關鍵字學習功能。

### `mecab`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | - |

MeCab 執行檔路徑。

```json
{
    "mecab": "/usr/bin/mecab"
}
```

---

### `mecabDic`

| 類型 | 必要 | 預設值 |
|------|------|--------|
| `string` | - | - |

MeCab 字典目錄路徑。建議使用 mecab-ipadic-neologd 以獲得更好的效果。

```json
{
    "mecabDic": "/usr/lib/x86_64-linux-gnu/mecab/dic/mecab-ipadic-neologd/"
}
```

---

## 完整範例

```json
{
    "host": "https://misskey.example.com",
    "i": "your-access-token",
    "master": "admin",
    "name": "藍",
    "serverName": "Misskey Example",
    "memoryDir": "data",
    
    "notingEnabled": true,
    "keywordEnabled": false,
    "reversiEnabled": true,
    "chartEnabled": true,
    "serverMonitoring": false,
    "checkEmojisEnabled": false,
    
    "geminiProApiKey": "your-gemini-key",
    "pLaMoApiKey": null,
    "prompt": "あなたは「藍」というAIです。",
    "aichatRandomTalkEnabled": false,
    "aichatRandomTalkProbability": "0.01",
    "aichatRandomTalkIntervalMinutes": "720",
    "aichatGroundingWithGoogleSearchAlwaysEnabled": false
}
```

---

## 環境變數

除了 `config.json`，部分設定也可以透過環境變數覆寫：

| 環境變數 | 對應設定 |
|----------|----------|
| `MISSKEY_HOST` | host |
| `MISSKEY_TOKEN` | i |
| `GEMINI_API_KEY` | geminiProApiKey |

---

## Docker 設定注意事項

使用 Docker 時，請注意：

1. **memoryDir**：必須設為掛載的目錄（如 `data`）以確保資料持久化
2. **MeCab**：需在 Dockerfile 中安裝，可透過 `enable_mecab` 參數控制
3. **font.ttf**：需掛載到容器內或在建構時複製

```yaml
# docker-compose.yml 範例
services:
  ai:
    build:
      context: .
      args:
        enable_mecab: "false"
    volumes:
      - ./data:/app/data
      - ./config.json:/app/config.json:ro
      - ./font.ttf:/app/font.ttf:ro
```
