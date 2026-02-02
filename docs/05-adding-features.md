# 05 - 新功能開發教學

本章節將透過實際範例，教你如何為藍開發新的功能模組。

## 模組開發流程

```
1. 建立模組檔案
       │
       ▼
2. 實作模組類別
       │
       ▼
3. 註冊模組
       │
       ▼
4. 新增台詞（如需要）
       │
       ▼
5. 新增設定項（如需要）
       │
       ▼
6. 編譯與測試
```

---

## 範例一：簡單問答模組

最基本的模組，只處理單一訊息並回覆。

### 步驟 1：建立模組檔案

建立 `src/modules/hello/index.ts`：

```typescript
import { bindThis } from '@/decorators.js';
import Module from '@/module.js';
import Message from '@/message.js';

export default class extends Module {
    // 模組唯一名稱
    public readonly name = 'hello';
    
    @bindThis
    public install() {
        return {
            mentionHook: this.mentionHook
        };
    }
    
    @bindThis
    private async mentionHook(msg: Message) {
        // 檢查觸發詞
        if (!msg.includes(['hello', 'こんにちは', '你好'])) {
            return false;  // 未處理，讓其他模組嘗試
        }
        
        // 回覆訊息
        msg.reply('Hello! 你好！こんにちは！');
        
        return true;  // 已處理
    }
}
```

### 步驟 2：註冊模組

編輯 `src/index.ts`，加入新模組：

```typescript
// 在 import 區段加入
import HelloModule from './modules/hello/index.js';

// 在模組陣列中加入（注意順序會影響處理優先度）
new 藍(account, [
    new CoreModule(),
    new HelloModule(),  // 新增這行
    new AiChatModule(),
    // ...
]);
```

### 步驟 3：編譯與執行

```bash
npm run build
npm start
```

---

## 範例二：帶有反應的模組

透過 `HandlerResult` 回傳自訂反應。

```typescript
import { bindThis } from '@/decorators.js';
import Module from '@/module.js';
import Message from '@/message.js';

export default class extends Module {
    public readonly name = 'goodjob';
    
    @bindThis
    public install() {
        return {
            mentionHook: this.mentionHook
        };
    }
    
    @bindThis
    private async mentionHook(msg: Message) {
        if (!msg.includes(['やった', '完成', '達成'])) {
            return false;
        }
        
        msg.reply('すごい！おめでとう！');
        
        // 回傳自訂反應
        return {
            reaction: 'congrats',  // 使用 'congrats' 表情符號
            immediate: true        // 立即發送（不延遲）
        };
    }
}
```

### 反應選項說明

| 回傳值 | 行為 |
|--------|------|
| `false` | 未處理，讓其他模組嘗試 |
| `true` | 已處理，使用預設反應 'love' |
| `{ reaction: 'xxx' }` | 使用指定反應 |
| `{ reaction: null }` | 不添加反應 |
| `{ immediate: true }` | 立即處理（不等待 1 秒） |

---

## 範例三：多輪對話模組

使用 Context 機制實現需要多次互動的功能。

### 情境：猜數字遊戲

```typescript
import { bindThis } from '@/decorators.js';
import loki from 'lokijs';
import Module from '@/module.js';
import Message from '@/message.js';

type GameSession = {
    oddsOrEvens: string;
    answer: number;
    startedAt: number;
};

export default class extends Module {
    public readonly name = 'oddeven';
    
    private sessions: loki.Collection<GameSession>;
    
    @bindThis
    public install() {
        // 建立資料庫集合
        this.sessions = this.ai.getCollection('oddeven_sessions', {
            indices: ['oddsOrEvens']
        });
        
        return {
            mentionHook: this.mentionHook,
            contextHook: this.contextHook
        };
    }
    
    @bindThis
    private async mentionHook(msg: Message) {
        if (!msg.includes(['奇偶', '単偶'])) {
            return false;
        }
        
        // 產生答案
        const answer = Math.floor(Math.random() * 10) + 1;
        
        // 建立遊戲記錄
        const session: GameSession = {
            oddsOrEvens: msg.userId,
            answer: answer,
            startedAt: Date.now()
        };
        this.sessions.insertOne(session);
        
        // 回覆並訂閱使用者的回覆
        msg.reply('好！我想了一個 1~10 的數字，請猜是奇數還是偶數？').then(reply => {
            this.subscribeReply(
                msg.userId,           // key: 使用者 ID
                msg.isChat,           // 是否為私訊
                msg.isChat ? msg.userId : reply.id,  // 要訂閱的 ID
                { oddsOrEvens: msg.userId }          // 附加資料
            );
        });
        
        return true;
    }
    
    @bindThis
    private async contextHook(key: any, msg: Message, data: any) {
        const session = this.sessions.findOne({
            oddsOrEvens: data.oddsOrEvens
        });
        
        if (!session) {
            this.unsubscribeReply(key);
            return;
        }
        
        const text = msg.extractedText;
        let playerGuess: 'odd' | 'even' | null = null;
        
        if (text.includes('奇') || text.includes('odd')) {
            playerGuess = 'odd';
        } else if (text.includes('偶') || text.includes('even')) {
            playerGuess = 'even';
        }
        
        if (!playerGuess) {
            msg.reply('「奇數」か「偶数」で答えてね！').then(reply => {
                this.subscribeReply(msg.userId, msg.isChat, reply.id, data);
            });
            return;
        }
        
        const isOdd = session.answer % 2 === 1;
        const correctGuess = (isOdd && playerGuess === 'odd') || (!isOdd && playerGuess === 'even');
        
        // 清理
        this.sessions.remove(session);
        this.unsubscribeReply(key);
        
        if (correctGuess) {
            msg.reply(`正解！答えは ${session.answer} でした！🎉`);
            return { reaction: 'congrats' };
        } else {
            msg.reply(`残念！答えは ${session.answer} でした...`);
            return { reaction: 'hmm' };
        }
    }
}
```

---

## 範例四：定時任務模組

使用持久化定時器實現提醒功能。

```typescript
import { bindThis } from '@/decorators.js';
import loki from 'lokijs';
import Module from '@/module.js';
import Message from '@/message.js';

type Alarm = {
    id: string;
    userId: string;
    time: number;
    message: string;
};

export default class extends Module {
    public readonly name = 'alarm';
    
    private alarms: loki.Collection<Alarm>;
    
    @bindThis
    public install() {
        this.alarms = this.ai.getCollection('alarms', {
            indices: ['id', 'userId']
        });
        
        return {
            mentionHook: this.mentionHook,
            timeoutCallback: this.timeoutCallback
        };
    }
    
    @bindThis
    private async mentionHook(msg: Message) {
        // 解析「30分後に〇〇を思い出させて」
        const match = msg.extractedText.match(/(\d+)(分|時間)後に(.+)を思い出させて/);
        
        if (!match) return false;
        
        const amount = parseInt(match[1]);
        const unit = match[2];
        const task = match[3];
        
        // 計算延遲時間
        const delay = unit === '時間'
            ? amount * 60 * 60 * 1000
            : amount * 60 * 1000;
        
        // 限制最長時間
        if (delay > 24 * 60 * 60 * 1000) {
            msg.reply('24時間以上は設定できません...');
            return true;
        }
        
        // 建立鬧鐘記錄
        const alarm: Alarm = {
            id: msg.id,
            userId: msg.userId,
            time: Date.now() + delay,
            message: task
        };
        this.alarms.insertOne(alarm);
        
        // 設定持久化定時器
        this.setTimeoutWithPersistence(delay, {
            alarmId: msg.id
        });
        
        msg.reply(`わかりました！${amount}${unit}後にお知らせしますね！`);
        
        return { reaction: '⏰', immediate: true };
    }
    
    @bindThis
    private async timeoutCallback(data: { alarmId: string }) {
        const alarm = this.alarms.findOne({ id: data.alarmId });
        
        if (!alarm) return;
        
        // 取得好友資訊
        const friend = this.ai.lookupFriend(alarm.userId);
        const name = friend?.name;
        
        // 發送提醒
        await this.ai.post({
            text: `@${friend?.doc.user.username} ${name ? name + '、' : ''}「${alarm.message}」の時間ですよ！`
        });
        
        // 清理
        this.alarms.remove(alarm);
    }
}
```

---

## 範例五：串流監聽模組

監聽時間軸並自動處理。

```typescript
import { bindThis } from '@/decorators.js';
import Module from '@/module.js';
import Stream from '@/stream.js';
import type { Note } from '@/misskey/note.js';

export default class extends Module {
    public readonly name = 'autoreact';
    
    private htl: ReturnType<Stream['useSharedConnection']>;
    
    @bindThis
    public install() {
        // 訂閱首頁時間軸
        this.htl = this.ai.connection.useSharedConnection('homeTimeline');
        this.htl.on('note', this.onNote);
        
        return {};  // 不使用 mentionHook
    }
    
    @bindThis
    private async onNote(note: Note) {
        // 過濾條件
        if (note.reply != null) return;  // 忽略回覆
        if (note.text == null) return;   // 忽略無文字貼文
        if (note.text.includes('@')) return;  // 忽略提及
        
        // 自動反應邏輯
        if (note.text.includes('藍')) {
            await this.ai.api('notes/reactions/create', {
                noteId: note.id,
                reaction: '🙌'
            });
        }
    }
}
```

---

## 範例六：定時發文模組

定期自動發文。

```typescript
import { bindThis } from '@/decorators.js';
import Module from '@/module.js';
import config from '@/config.js';

export default class extends Module {
    public readonly name = 'dailygreeting';
    
    @bindThis
    public install() {
        // 設定定時任務
        setInterval(this.checkAndPost, 1000 * 60 * 3);  // 每 3 分鐘檢查
        
        return {};
    }
    
    @bindThis
    private async checkAndPost() {
        const now = new Date();
        
        // 只在早上 7 點發文
        if (now.getHours() !== 7) return;
        
        // 檢查今天是否已發文
        const date = `${now.getFullYear()}-${now.getMonth()}-${now.getDate()}`;
        const data = this.getData();
        if (data.lastPosted === date) return;
        
        // 更新記錄
        data.lastPosted = date;
        this.setData(data);
        
        // 發文
        await this.ai.post({
            text: 'おはようございます！今日も良い一日を！☀️'
        });
        
        this.log('Daily greeting posted');
    }
}
```

---

## 新增台詞

當模組需要多種回覆時，應將台詞集中到 `serifs.ts`。

### 步驟 1：編輯 serifs.ts

```typescript
export default {
    // ... 現有台詞 ...
    
    // 新增模組的台詞
    mymodule: {
        greeting: name => name ? `こんにちは、${name}！` : 'こんにちは！',
        
        result: {
            win: ['やったね！', 'すごい！', 'おめでとう！'],  // 陣列 = 隨機選擇
            lose: 'ざんねん...',
        },
        
        error: 'エラーが発生しました...',
    },
};
```

### 步驟 2：在模組中使用

```typescript
import serifs, { getSerif } from '@/serifs.js';

// 單一台詞
msg.reply(serifs.mymodule.greeting(msg.friend.name));

// 從陣列隨機選擇
msg.reply(getSerif(serifs.mymodule.result.win));
```

---

## 新增設定項

當模組需要可配置的選項時，應加入設定系統。

### 步驟 1：編輯 config.ts

```typescript
type Config = {
    // ... 現有設定 ...
    
    // 新增設定項
    myModuleEnabled?: boolean;
    myModuleApiKey?: string;
};
```

### 步驟 2：在模組中使用

```typescript
import config from '@/config.js';

@bindThis
public install() {
    // 根據設定決定是否啟用
    if (config.myModuleEnabled === false) {
        return {};  // 不註冊任何 Hook
    }
    
    // 檢查必要設定
    if (!config.myModuleApiKey) {
        this.log('API key not configured, module disabled');
        return {};
    }
    
    return {
        mentionHook: this.mentionHook
    };
}
```

### 步驟 3：更新 example.json

```json
{
    "myModuleEnabled": true,
    "myModuleApiKey": "your-api-key-here"
}
```

---

## 常見模式

### 1. 每日限制

```typescript
import getDate from '@/utils/get-date.js';

private someAction(msg: Message): boolean {
    const today = getDate();
    const data = msg.friend.getPerModulesData(this);
    
    if (data.lastActionAt === today) {
        msg.reply('今日はもうやったよ！');
        return true;
    }
    
    // 執行動作
    data.lastActionAt = today;
    msg.friend.setPerModulesData(this, data);
    return true;
}
```

### 2. 親愛度相關回應

```typescript
private respond(msg: Message) {
    const love = msg.friend.love;
    
    if (love >= 10) {
        msg.reply('大好き！♡');
    } else if (love >= 5) {
        msg.reply('好きだよ！');
    } else if (love <= -5) {
        msg.reply('...');
    } else {
        msg.reply('何？');
    }
}
```

### 3. 管理員限定功能

```typescript
import config from '@/config.js';

private adminOnly(msg: Message): boolean {
    if (msg.user.username !== config.master) {
        return false;  // 不處理，讓其他模組嘗試
    }
    
    // 管理員功能
    return true;
}
```

### 4. 附帶檔案回覆

```typescript
private async replyWithImage(msg: Message) {
    // 產生圖片
    const imageBuffer = await this.generateImage();
    
    // 上傳到 Drive
    const file = await this.ai.upload(imageBuffer, {
        filename: 'result.png',
        contentType: 'image/png'
    });
    
    // 回覆並附帶檔案
    msg.reply('できました！', { file });
}
```

---

## 除錯技巧

### 1. 使用 log 輸出

```typescript
this.log(`Processing message: ${msg.id}`);
this.log(`User: ${msg.userId}, Love: ${msg.friend.love}`);
```

### 2. 檢查資料庫內容

```typescript
// 在程式碼中暫時加入
const allData = this.ai.friends.find({});
console.log(JSON.stringify(allData, null, 2));
```

### 3. 測試定時器

```typescript
// 開發時使用較短的延遲
const delay = process.env.NODE_ENV === 'development'
    ? 1000 * 10  // 10 秒
    : 1000 * 60 * 60;  // 1 小時
```

---

## 提交前檢查清單

- [ ] 執行 `npm run build` 確認編譯成功
- [ ] 測試所有觸發詞都能正常匹配
- [ ] 測試多輪對話能正確追蹤狀態
- [ ] 測試定時器在 Bot 重啟後仍能正常運作
- [ ] 確認台詞沒有拼寫錯誤
- [ ] 確認設定項有在 example.json 中說明
- [ ] 確認程式碼有適當的錯誤處理

---

## 結語

恭喜你完成了藍的技術指南！你現在應該能夠：

1. 理解藍的整體架構和資料流程
2. 理解核心類別的功能和 API
3. 開發新的功能模組
4. 使用各種工具函式
5. 處理多輪對話和定時任務

如果有任何問題，可以參考現有模組的實作，或查看 [AGENTS.md](../AGENTS.md) 獲得更多資訊。

Happy coding! 🎉
