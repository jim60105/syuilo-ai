# 藍 (Ai) - Misskey Bot Project

## Project Overview

This is **藍 (Ai)**, an AI-powered bot for Misskey social network platform. Ai is designed as a helpful mascot character (三須木藍/Misuki Ai) who interacts with users through various features including games, fortune-telling, AI chat, and automated tasks.

**Project Type**: TypeScript Node.js application  
**Version**: 2.0.1  
**License**: MIT  
**Primary Language**: Japanese (code comments and user interactions are in Japanese)

## Architecture

### High-Level Structure

The project follows a modular architecture where each feature is implemented as a separate module:

```
├── src/
│   ├── index.ts              # Application bootstrap
│   ├── ai.ts                 # Core AI class
│   ├── module.ts             # Abstract base module class
│   ├── config.ts             # Configuration management
│   ├── modules/              # Feature modules directory
│   │   ├── aichat/          # AI chat with Gemini/PLaMo API
│   │   ├── reversi/         # Reversi game
│   │   ├── fortune/         # Fortune-telling
│   │   ├── maze/            # Maze generation
│   │   ├── core/            # Core functionality
│   │   ├── reminder/        # Reminder system
│   │   └── ... (20+ modules)
│   ├── serifs.ts            # Bot dialogue/responses
│   ├── vocabulary.ts        # Word/phrase collections
│   └── utils/               # Utility functions
├── built/                    # Compiled JavaScript output
├── data/                     # Runtime data directory
├── config.json              # Configuration file (not in repo)
├── memory.json              # Persistent in-memory database
├── font.ttf                 # Custom font file (optional)
└── package.json
```

### Key Technologies

- **Runtime**: Node.js (ES2022, ES Modules)
- **Language**: TypeScript 5.8.2 with experimental decorators
- **Database**: LokiJS (in-memory with JSON persistence)
- **AI APIs**: Google Gemini API, Preferred Networks PLaMo API
- **Misskey Integration**: WebSocket streaming, REST API
- **Canvas**: node-canvas for image generation
- **Japanese NLP**: MeCab (optional, for keyword learning)

### Module System

All feature modules extend the abstract `Module` class and implement:

```typescript
export default class extends Module {
  public readonly name = 'module-name';
  
  @bindThis
  public install(): InstallerResult {
    // Return hooks and callbacks
    return {
      mentionHook?: (msg: Message) => Promise<boolean | HandlerResult>,
      contextHook?: (key, msg, data) => Promise<void | boolean | HandlerResult>,
      timeoutCallback?: (data) => void
    };
  }
}
```

The `@bindThis` decorator ensures proper `this` binding for class methods.

## Build and Development

### Setup Commands

```bash
# Install dependencies
npm install

# Build TypeScript to JavaScript
npm run build

# Start the bot
npm start

# Development mode with auto-restart
npm start-daemon
```

**Important**: Always run `npm run build` after making code changes. The application runs from the `./built/` directory.

### Docker Support

```bash
# Build and run with Docker Compose
docker-compose build
docker-compose up
```

The `docker-compose.yml` allows toggling MeCab installation via `enable_mecab` argument.

### Configuration

Create `config.json` in project root (see `example.json`):

```json
{
  "host": "https://your-misskey-instance.example",
  "i": "your-bot-access-token",
  "master": "admin-username",
  "notingEnabled": true,
  "keywordEnabled": false,
  "reversiEnabled": true,
  "chartEnabled": true,
  "serverMonitoring": true,
  "checkEmojisEnabled": false,
  "geminiProApiKey": "your-gemini-api-key",
  "pLaMoApiKey": "your-plamo-api-key",
  "prompt": "AI personality prompt in Japanese",
  "aichatRandomTalkEnabled": false,
  "aichatRandomTalkProbability": "0.01",
  "aichatRandomTalkIntervalMinutes": "720",
  "aichatGroundingWithGoogleSearchAlwaysEnabled": false,
  "mecab": "/usr/bin/mecab",
  "mecabDic": "/usr/lib/x86_64-linux-gnu/mecab/dic/mecab-ipadic-neologd/",
  "memoryDir": "data"
}
```

**Critical Settings**:
- `host`: Misskey instance URL (no trailing slash)
- `i`: Bot account access token from Misskey settings
- `geminiProApiKey`: Required for AI chat functionality
- `memoryDir`: Directory for `memory.json` persistence (default: `data/`)

### Path Aliases

The project uses TypeScript path mapping:
- `@/*` maps to `./src/*`
- Transformed at compile-time by `typescript-transform-paths`

## Coding Conventions

### Language Usage

- **Code comments**: Japanese (日本語)
- **Variable/function names**: English or Japanese depending on context
- **User-facing text**: Japanese (stored in `serifs.ts`)
- **Commit messages**: English (conventional commits format)

### TypeScript Style

- Use `@bindThis` decorator for class methods that reference `this`
- Prefer `async/await` over Promise chains
- Use type annotations for function parameters and return types
- Enable strict mode checks (`strict: true` in tsconfig)

### Module Development Pattern

1. Create new module in `src/modules/<name>/index.ts`
2. Extend abstract `Module` class
3. Implement `install()` method returning hooks
4. Use `this.ai` to access bot API (post notes, react, etc.)
5. Use `this.log()` for logging (automatically prefixed with module name)
6. Register module in `src/index.ts` module array

### Common Patterns

```typescript
// Message hook pattern
@bindThis
private async mentionHook(msg: Message) {
  if (!msg.includes(['keyword'])) return false;
  msg.reply('response text');
  return true; // or return { reaction: 'emoji' }
}

// Context pattern (stateful conversations)
this.subscribeReply(msg.userId, true, msg.userId, { data: 'value' });

// LokiJS collection usage
this.collection = this.ai.getCollection('collection-name');
this.collection.insertOne({ data });
```

## Feature Modules

### Core Modules (Always Active)

- **core**: Basic interactions (greetings, name learning, version info)
- **emoji/emoji-react**: Emoji-based interactions
- **ping**: Simple ping/pong response
- **welcome**: Welcome new users

### Opt-in Modules (Configurable)

- **aichat**: AI-powered conversations using Gemini/PLaMo APIs
  - Supports file attachments (images, PDF, audio, video, text)
  - URL preview integration
  - Conversation history (up to 10 exchanges)
  - Google Search grounding support
  - Type switching: `&gemini`, `&plamo`, `&chatgpt` prefixes
  
- **reversi**: Play Reversi/Othello game with users
- **fortune**: Daily fortune-telling with custom algorithm
- **maze**: Generate and visualize mazes with various difficulty levels
- **guessing-game**: Number guessing game (0-100)
- **kazutori**: Number-taking game (multiplayer)
- **reminder**: User reminder system with persistence
- **timer**: Countdown timer functionality
- **poll**: Random poll generation
- **noting**: Periodic random notes
- **chart**: Server statistics charts
- **server**: Server monitoring and alerts
- **keyword**: MeCab-based keyword learning (requires MeCab)
- **check-custom-emojis**: Monitor custom emoji additions

## Testing and Validation

### Pre-deployment Checks

1. **Build verification**: `npm run build` must complete without errors
2. **Type checking**: TypeScript strict mode enforced
3. **Configuration**: Verify `config.json` has required fields
4. **Font file**: Place `font.ttf` in root if using maze/chart modules
5. **API keys**: Test Gemini/PLaMo API connectivity if using aichat

### Runtime Monitoring

- Check `memory.json` is being written to configured `memoryDir`
- Monitor WebSocket connection stability
- Watch for rate limiting from Misskey API
- Validate AI API responses (they may timeout for long requests)

## Common Pitfalls

1. **MeCab dependency**: Set `keywordEnabled: false` if MeCab not installed
2. **Memory file location**: Docker users must use `"memoryDir": "data"` and mount volume
3. **API rate limits**: Gemini API has request limits; handle gracefully
4. **Font requirement**: Maze/chart features need `font.ttf` in project root
5. **File attachments**: Sensitive files in aichat may cause unexpected responses
6. **Build before run**: Always `npm run build` after code changes
7. **Access token permissions**: Some features require admin permissions

## Maintenance Notes

- The bot logs activity with colored output using `chalk`
- Failed operations are retried with `promise-retry` library
- WebSocket auto-reconnects on disconnection
- Timer/reminder modules check every 1 second for pending tasks
- Memory is persisted on every database write operation

## Dependencies Requiring Attention

- **canvas**: Native binding, may need system libraries (Cairo, Pango)
- **kuromoji** (optional): Large dictionary files for Japanese tokenization
- **typescript-transform-paths**: Build-time path transformation
- **@types packages**: Keep synchronized with runtime package versions

## Special Conventions

- Use `serifs.ts` for all user-facing text (centralized localization)
- Bot personality defined in `config.json` `prompt` field
- `vocabulary.ts` contains random word generators for games
- Friend system tracks user interactions in memory database
- Module `name` property must be unique across all modules

---

**For detailed feature documentation**, see [torisetu.md](torisetu.md) (Japanese user manual).  
**For setup instructions**, see [README.md](README.md).
