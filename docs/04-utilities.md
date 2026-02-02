# 04 - 工具函式庫

本章節詳細說明 `src/utils/` 目錄下的所有工具函式。

## 目錄結構

```
src/utils/
├── acct.ts              # 產生使用者帳號字串
├── get-date.ts          # 取得日期字串
├── includes.ts          # 文字包含檢查（寬鬆）
├── japanese.ts          # 日文處理工具
├── log.ts               # 日誌輸出
├── or.ts                # 文字匹配檢查（嚴格）
├── safe-for-interpolate.ts  # 安全字串檢查
├── sleep.ts             # 延遲函式
├── url2base64.ts        # URL 轉 Base64
└── url2json.ts          # URL 預覽取得
```

---

## 日誌工具

### log.ts

```typescript
import chalk from 'chalk';

export default function(msg: string) {
    const now = new Date();
    const date = `${zeroPad(now.getHours())}:${zeroPad(now.getMinutes())}:${zeroPad(now.getSeconds())}`;
    console.log(`${chalk.gray(date)} ${msg}`);
}

function zeroPad(num: number, length: number = 2): string {
    return ('0000000000' + num).slice(-length);
}
```

**功能**：輸出帶時間戳記的日誌。

**輸出範例**：
```
12:34:56 [AiOS]: Starting modules...
12:34:56 [core]: Module installed
12:34:57 [aichat]: AI Chat ready
```

---

## 日文處理工具

### japanese.ts

提供日文文字轉換功能。

#### katakanaToHiragana(str: string): string

將片假名轉換為平假名。

```typescript
katakanaToHiragana('アイ')  // → 'あい'
katakanaToHiragana('カタカナ')  // → 'かたかな'
```

**原理**：片假名的 Unicode 碼點比對應的平假名高 `0x60`。

```typescript
return str.replace(/[\u30a1-\u30f6]/g, match => {
    const char = match.charCodeAt(0) - 0x60;
    return String.fromCharCode(char);
});
```

---

#### hiraganaToKatagana(str: string): string

將平假名轉換為片假名。

```typescript
hiraganaToKatagana('あい')  // → 'アイ'
```

---

#### zenkakuToHankaku(str: string): string

將全形片假名轉換為半形片假名。

```typescript
zenkakuToHankaku('アイウエオ')  // → 'ｱｲｳｴｵ'
```

---

#### hankakuToZenkaku(str: string): string

將半形片假名轉換為全形片假名。

```typescript
hankakuToZenkaku('ｱｲｳｴｵ')  // → 'アイウエオ'
```

---

## 文字匹配工具

### includes.ts

寬鬆的文字包含檢查，適用於判斷使用者是否在談論某個主題。

```typescript
import { katakanaToHiragana, hankakuToZenkaku } from './japanese.js';

export default function(text: string, words: string[]): boolean {
    if (text == null) return false;
    
    // 標準化文字：半形→全形、片假名→平假名、小寫
    text = katakanaToHiragana(hankakuToZenkaku(text)).toLowerCase();
    words = words.map(word => katakanaToHiragana(word).toLowerCase());
    
    // 只要包含任一詞彙即回傳 true
    return words.some(word => text.includes(word));
}
```

**使用範例**：
```typescript
// 以下都會回傳 true
includes('今日の運勢を教えて', ['占', '運勢'])
includes('おみくじ引いて', ['占', 'おみくじ'])
includes('ウラナイして', ['うらない'])  // 片假名→平假名
```

---

### or.ts

嚴格的文字匹配檢查，適用於判斷使用者的確切意圖。

```typescript
export default function(text: string, words: (string | RegExp)[]): boolean {
    if (text == null) return false;
    
    text = katakanaToHiragana(hankakuToZenkaku(text));
    words = words.map(word => typeof word == 'string' ? katakanaToHiragana(word) : word);
    
    return words.some(word => {
        // denoise 函式移除裝飾性文字
        function denoise(text: string): string {
            text = text.trim();
            
            // 移除開頭的 @提及
            if (text.startsWith('@')) {
                text = text.replace(/^@[a-zA-Z0-9\-_]+/, '');
                text = text.trim();
            }
            
            function fn() {
                text = text.replace(/[！!]+$/, '');      // 移除驚嘆號
                text = text.replace(/っ+$/, '');         // 移除促音
                text = text.replace(/ー+$/, '') + ...;   // 處理長音
                text = text.replace(/。$/, '');          // 移除句號
                text = text.replace(/です$/, '');        // 移除「です」
                text = text.replace(/(\.|…)+$/, '');    // 移除省略號
                text = text.replace(/[♪♥]+$/, '');      // 移除裝飾符號
                text = text.replace(/^藍/, '');          // 移除開頭的「藍」
                text = text.replace(/^ちゃん/, '');      // 移除「ちゃん」
                text = text.replace(/、+$/, '');         // 移除逗號
            }
            
            // 重複執行直到文字不再變化
            let textBefore = text;
            let textAfter: string | null = null;
            while (textBefore != textAfter) {
                textBefore = text;
                fn();
                textAfter = text;
            }
            return text;
        }
        
        if (typeof word == 'string') {
            return (text == word) || (denoise(text) == word);
        } else {
            return (word.test(text)) || (word.test(denoise(text)));
        }
    });
}
```

**使用範例**：
```typescript
// 嚴格匹配「好き」
or('好き', ['好き', 'すき'])  // → true
or('好きです！', ['好き'])    // → true（移除「です！」後匹配）
or('好きだよ', ['好き'])      // → false（「だよ」未被移除）

// 支援正規表達式
or('はぐして', [/^はぐ(し(て|よ|よう)?)?$/])  // → true
or('はぐしよう', [/^はぐ(し(て|よ|よう)?)?$/])  // → true
```

---

## 日期/時間工具

### get-date.ts

取得當日日期字串，用於每日限制功能。

```typescript
export default function(): string {
    const now = new Date();
    const y = now.getFullYear();
    const m = now.getMonth();
    const d = now.getDate();
    const today = `${y}/${m + 1}/${d}`;
    return today;
}
```

**輸出範例**：`"2024/1/15"`

**使用場景**：
```typescript
const today = getDate();

// 檢查今天是否已經問候過
if (data.lastGreetedAt === today) return;

data.lastGreetedAt = today;
```

---

### sleep.ts

非同步延遲函式。

```typescript
export function sleep(msec: number) {
    return new Promise<void>(res => {
        setTimeout(() => {
            res();
        }, msec);
    });
}
```

**使用範例**：
```typescript
// 等待 2 秒
await sleep(2000);

// 模擬思考延遲
if (!opts?.immediate) {
    await sleep(2000);
}
```

---

## 使用者相關工具

### acct.ts

產生使用者帳號字串（用於提及）。

```typescript
export function acct(user: { username: string; host?: string | null; }): string {
    return user.host
        ? `@${user.username}@${user.host}`
        : `@${user.username}`;
}
```

**輸出範例**：
- 本地使用者：`@username`
- 遠端使用者：`@username@remote.example.com`

---

### safe-for-interpolate.ts

檢查字串是否適合用於訊息插值（避免特殊字元造成問題）。

```typescript
const invalidChars = [
    '@',   // 可能觸發提及
    '#',   // 可能觸發 hashtag
    '*',   // Markdown 格式
    ':',   // 可能觸發表情符號
    '(',   // 可能觸發表情符號
    ')',
    '[',   // Markdown 連結
    ']',
    ' ',   // 空格
    '　',  // 全形空格
];

export function safeForInterpolate(text: string): boolean {
    return !invalidChars.some(c => text.includes(c));
}
```

**使用場景**：
```typescript
// 設定暱稱時檢查
if (!safeForInterpolate(name)) {
    msg.reply(serifs.core.invalidName);
    return true;
}
```

---

## URL 處理工具

### url2base64.ts

將 URL 內容下載並轉換為 Base64 字串。

```typescript
import got from 'got';

export default async function(url: string): Promise<string> {
    try {
        const buffer = await got(url).buffer();
        const base64File = buffer.toString('base64');
        return base64File;
    } catch (err: unknown) {
        log('Error in url2base64');
        if (err instanceof Error) {
            log(`${err.name}\n${err.message}\n${err.stack}`);
        }
        throw err;
    }
}
```

**使用場景**：AI 聊天模組中將圖片轉為 Base64 供 Gemini API 使用。

---

### url2json.ts

取得 URL 的預覽資訊（使用 Misskey 的 URL 預覽 API）。

```typescript
export default async function(url: string): Promise<string> {
    try {
        const urlPreviewUrl = config.host + '/url';
        return await got(
            urlPreviewUrl, {
                searchParams: {
                    url: url,
                    lang: 'ja-JP'
                },
                timeout: {
                    lookup: 500,
                    send: 500,
                    response: 10000
                },
            }).json();
    } catch (err: unknown) {
        log('Error in url2json');
        throw err;
    }
}
```

**回傳型別**：
```typescript
type UrlPreview = {
    title: string;
    icon: string;
    description: string;
    thumbnail: string;
    player: {
        url: string;
        width: number;
        height: number;
        allow: [];
    };
    sitename: string;
    sensitive: boolean;
    activityPub: string;
    url: string;
};
```

**使用場景**：AI 聊天模組中提取使用者分享的 URL 資訊，作為對話上下文。

---

## 工具函式使用最佳實踐

### 1. 使用 includes vs or

```typescript
// 使用 includes：寬鬆匹配，適合判斷主題
if (msg.includes(['占', '運勢', 'おみくじ'])) {
    // 使用者可能在談論運勢相關話題
}

// 使用 or：嚴格匹配，適合判斷指令
if (msg.or(['好き', 'すき'])) {
    // 使用者明確表達「喜歡」
}
```

### 2. 日期限制功能

```typescript
import getDate from '@/utils/get-date.js';

private someAction(msg: Message): boolean {
    const today = getDate();
    const data = msg.friend.getPerModulesData(this);
    
    // 每日只能執行一次
    if (data.lastActionAt === today) {
        return false;
    }
    
    // 執行動作...
    
    // 記錄執行日期
    data.lastActionAt = today;
    msg.friend.setPerModulesData(this, data);
    
    return true;
}
```

### 3. 處理 URL

```typescript
import urlToJson from '@/utils/url2json.js';
import urlToBase64 from '@/utils/url2base64.js';

// 取得 URL 預覽資訊
const preview = await urlToJson('https://example.com');
console.log(preview.title, preview.description);

// 將圖片轉為 Base64
const base64 = await urlToBase64('https://example.com/image.png');
```

### 4. 暱稱安全檢查

```typescript
import { safeForInterpolate } from '@/utils/safe-for-interpolate.js';

private setName(msg: Message): boolean {
    const name = msg.extractedText.match(/\s*(.+?)って呼んで/)![1];
    
    // 長度檢查
    if (name.length > 10) {
        msg.reply(serifs.core.tooLong);
        return true;
    }
    
    // 安全性檢查
    if (!safeForInterpolate(name)) {
        msg.reply(serifs.core.invalidName);
        return true;
    }
    
    msg.friend.updateName(name);
    return true;
}
```

---

## 下一章

接下來請閱讀 [05-adding-features.md](05-adding-features.md) 了解如何開發新功能模組。
