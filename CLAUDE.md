# 注意
- 応答は日本語で
- 私はtypescript初心者です

# Claude Code Web UI

A web-based interface for the `claude` command line tool that provides streaming responses in a chat interface.

## サブスクリプション版への変更計画

### 変更点の概要
- **API利用** → **サブスクリプション利用** へ変更
- Claude Code SDKの設定を無効化し、直接的なコマンド実行を採用
- セッション管理をサブスクリプション版の仕様に合わせて調整

### アーキテクチャ
```
[ブラウザ] ↔ [WebSocket] ↔ [Node.jsサーバー] ↔ [子プロセス] ↔ [Claude Code Subscription]
```

## Code Quality

This project uses automated quality checks to ensure consistent code standards:

- **Lefthook**: Git hooks manager that runs `make check` before every commit
- **Quality Commands**: Use `make check` to run all quality checks manually  
- **CI/CD**: GitHub Actions runs the same quality checks on every push

The pre-commit hook prevents commits with formatting, linting, or test failures.

### Setup for New Contributors

1. **Install Lefthook**: 
   ```bash
   # macOS
   brew install lefthook
   
   # Or download from https://github.com/evilmartians/lefthook/releases
   ```

2. **Install hooks**:
   ```bash
   lefthook install
   ```

3. **Verify setup**:
   ```bash
   lefthook run pre-commit
   ```

The `.lefthook.yml` configuration is tracked in the repository, ensuring consistent quality checks across all contributors.

## サブスクリプション版の主要修正点

### バックエンドの変更

#### Claude Code実行設定の変更
**修正対象**: `backend/handlers/chat.ts`

```typescript
// 修正前: API用の設定
const executionConfig = getClaudeExecutionConfig(claudePath, runtime);

for await (
  const sdkMessage of query({
    prompt: processedMessage,
    options: {
      abortController,
      ...executionConfig, // API用設定
      ...(sessionId ? { resume: sessionId } : {}),
      ...(allowedTools ? { allowedTools } : {}),
      ...(workingDirectory ? { cwd: workingDirectory } : {}),
    },
  })
)

// 修正後: サブスクリプション用の設定
// 直接コマンド実行に変更
const claudeProcess = runtime.spawn('claude', [
  processedMessage,
  ...(sessionId ? ['--resume', sessionId] : []),
  ...(workingDirectory ? ['--cwd', workingDirectory] : []),
], {
  cwd: workingDirectory,
  env: {
    ...process.env,
  }
});
```

#### バージョンチェックの簡素化
**修正対象**: `backend/cli/deno.ts`

```typescript
// 修正前: API向けのバージョンチェック
async function validateClaudeCli(runtime: DenoRuntime) {
  try {
    const result = await runtime.runCommand("claude", ["--version"]);
    if (result.success) {
      console.log(`✅ Claude CLI found: ${result.stdout.trim()}`);
    } else {
      console.warn("⚠️  Claude CLI check failed - some features may not work");
    }
  } catch (_error) {
    console.warn("⚠️  Claude CLI not found - please install claude-code");
  }
}

// 修正後: サブスクリプション向けの簡易チェック
async function validateClaudeCli(runtime: DenoRuntime) {
  try {
    const result = await runtime.runCommand("claude", ["--version"]);
    if (result.success) {
      console.log(`✅ Claude CLI found: ${result.stdout.trim()}`);
    } else {
      console.warn("⚠️  Claude CLI check failed - some features may not work");
    }
  } catch (_error) {
    console.warn("⚠️  Claude CLI not found - please install claude-code");
    console.warn("   Visit: https://claude.ai/code for installation instructions");
  }
}
```

#### メッセージ処理の調整
**修正対象**: `backend/handlers/chat.ts`

```typescript
// サブスクリプション版では、より直接的なストリーミング処理
async function* executeClaudeCommand(
  message: string,
  requestId: string,
  requestAbortControllers: Map<string, AbortController>,
  runtime: Runtime,
  sessionId?: string,
  allowedTools?: string[],
  workingDirectory?: string,
  debugMode?: boolean,
): AsyncGenerator<StreamResponse> {
  let abortController: AbortController;

  try {
    abortController = new AbortController();
    requestAbortControllers.set(requestId, abortController);

    // サブスクリプション版用のコマンド構築
    const claudeArgs = [
      message,
      ...(sessionId ? ['--resume', sessionId] : []),
      ...(workingDirectory ? ['--cwd', workingDirectory] : []),
      '--stream', // ストリーミング出力を有効化
    ];

    // 子プロセスとして実行
    const claudeProcess = runtime.spawn('claude', claudeArgs, {
      cwd: workingDirectory,
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    // ストリーミング出力を処理
    for await (const chunk of claudeProcess.stdout) {
      if (abortController.signal.aborted) break;
      
      const output = new TextDecoder().decode(chunk);
      // サブスクリプション版の出力形式に合わせてパース
      const lines = output.split('\n').filter(line => line.trim());
      
      for (const line of lines) {
        try {
          // JSONラインかプレーンテキストかを判定
          if (line.startsWith('{')) {
            const data = JSON.parse(line);
            yield {
              type: "claude_json",
              data,
            };
          } else {
            // プレーンテキストの場合はメッセージとして処理
            yield {
              type: "claude_json",
              data: {
                type: "assistant",
                message: {
                  role: "assistant",
                  content: [{ type: "text", text: line }],
                },
                session_id: sessionId,
              },
            };
          }
        } catch (parseError) {
          if (debugMode) {
            console.debug("Failed to parse line:", line, parseError);
          }
        }
      }
    }

    yield { type: "done" };
  } catch (error) {
    // エラーハンドリング
    yield {
      type: "error",
      error: error instanceof Error ? error.message : String(error),
    };
  } finally {
    if (requestAbortControllers.has(requestId)) {
      requestAbortControllers.delete(requestId);
    }
  }
}
```

### 設定管理の変更

#### 環境設定
```typescript
// backend/types.ts に追加
export interface SubscriptionConfig {
  // サブスクリプション特有の設定
  enableDirectExecution: boolean;
  maxConcurrentSessions: number;
}

// デフォルト設定
const DEFAULT_SUBSCRIPTION_CONFIG: SubscriptionConfig = {
  enableDirectExecution: true,
  maxConcurrentSessions: 5,
};
```

#### 起動時設定
```bash
# 起動時の簡易チェック
echo "🚀 Claude Code WebUI (サブスクリプション版) を起動中..."
node server.js
```

### ルーティングとハンドラーの簡素化

既存の `/api/chat` エンドポイントを直接コマンド実行用に修正するのみで、新しいAPIエンドポイントは追加しません。

## Architecture

This project consists of three main components:

### Backend (Deno)

- **Location**: `backend/`
- **Port**: 8080 (configurable via CLI argument or PORT environment variable)
- **Technology**: Deno with TypeScript + Hono framework
- **Purpose**: Executes `claude` commands and streams JSON responses to frontend

**Key Features**:

- **Runtime Abstraction**: Clean separation between business logic and platform-specific code
- **Modular Architecture**: CLI, application core, and runtime layers clearly separated
- Command line interface with `--port`, `--help`, `--version` options  
- Startup validation to check Claude CLI availability
- Executes `claude --output-format stream-json --verbose -p <message>`
- Streams raw Claude JSON responses without modification
- Sets working directory to project root for claude command execution
- Provides CORS headers for frontend communication
- Single binary distribution support
- Session continuity support using Claude Code SDK's resume functionality
- **Comprehensive Testing**: Mock runtime enables full unit testing without external dependencies

**API Endpoints**:

- `GET /api/projects` - Retrieves list of available project directories
  - Response: `{ projects: ProjectInfo[] }` - Array of project info objects with path and encodedName
- `POST /api/chat` - Accepts chat messages and returns streaming responses
  - Request body: `{ message: string, sessionId?: string, requestId: string, allowedTools?: string[], workingDirectory?: string }`
  - `requestId` is required for request tracking and abort functionality
  - Optional `sessionId` enables conversation continuity within the same chat session
  - Optional `allowedTools` array restricts which tools Claude can use
  - Optional `workingDirectory` specifies the project directory for Claude execution
- `POST /api/abort/:requestId` - Aborts an ongoing request by request ID
- `GET /api/projects/:encodedProjectName/histories` - Retrieves list of conversation histories for a project
  - Response: `{ conversations: ConversationSummary[] }` - Array of conversation summaries with session metadata
- `GET /api/projects/:encodedProjectName/histories/:sessionId` - Retrieves detailed conversation history for a specific session
  - Response: `ConversationHistory` - Complete conversation with messages and metadata
- `/*` - Serves static frontend files (in single binary mode)

### Frontend (React)

- **Location**: `frontend/`
- **Port**: 3000 (configurable via `--port` CLI argument to `npm run dev`)
- **Technology**: Vite + React + SWC + TypeScript + TailwindCSS + React Router
- **Purpose**: Provides project selection and chat interface with streaming responses

**Key Features**:

- **Project Directory Selection**: Choose working directory before starting chat sessions
- **Routing System**: Separate pages for project selection, chat interface, and demo mode
- **Conversation History**: Browse and restore previous chat sessions with full message history
- **Demo Mode**: Interactive demonstration system with automated scenarios and mock responses
- Real-time streaming response display with modular message processing
- Parses different Claude JSON message types (system, assistant, result, tool messages)
- TailwindCSS utility-first styling for responsive design
- Light/dark theme toggle with system preference detection and localStorage persistence
- Bottom-to-top message flow layout (messages start at bottom like modern chat apps)
- Auto-scroll to bottom with smart scroll detection (only auto-scrolls when user is near bottom)
- Accessibility features with ARIA attributes for screen readers
- Responsive chat interface with component-based architecture
- Comprehensive component testing with Vitest and Testing Library
- Automatic session tracking for conversation continuity within the same chat instance
- Request abort functionality with real-time cancellation
- Permission dialog handling for Claude tool permissions
- Enhanced error handling and user feedback
- Modular hook architecture for state management and business logic separation
- Reusable UI components with consistent design patterns
- **History Management**: View conversation summaries, timestamps, and message previews
- **Demo Automation**: Automated demo recording and playback for presentations

### Shared Types

- **Location**: `shared/`
- **Purpose**: TypeScript type definitions shared between backend and frontend

**Key Types**:

- `StreamResponse` - Backend streaming response format with support for claude_json, error, done, and aborted types
- `ChatRequest` - Chat request structure for API communication
  - `message: string` - User's message content
  - `sessionId?: string` - Optional session ID for conversation continuity
  - `requestId: string` - Required unique identifier for request tracking and abort functionality
  - `allowedTools?: string[]` - Optional array to restrict which tools Claude can use
  - `workingDirectory?: string` - Optional project directory path for Claude execution
- `AbortRequest` - Request structure for aborting ongoing operations
  - `requestId: string` - ID of the request to abort
- `ProjectInfo` - Project information structure
  - `path: string` - Full file system path to the project directory
  - `encodedName: string` - URL-safe encoded project name
- `ProjectsResponse` - Response structure for project directory list
  - `projects: ProjectInfo[]` - Array of project information objects
- `ConversationSummary` - Summary information for conversation history
  - `sessionId: string` - Unique session identifier
  - `startTime: string` - ISO timestamp of first message
  - `lastTime: string` - ISO timestamp of last message
  - `messageCount: number` - Total number of messages in conversation
  - `lastMessagePreview: string` - Preview text of the last message
- `HistoryListResponse` - Response structure for conversation history list
  - `conversations: ConversationSummary[]` - Array of conversation summaries
- `ConversationHistory` - Complete conversation history structure
  - `sessionId: string` - Session identifier
  - `messages: unknown[]` - Array of timestamped SDK messages (typed as unknown[] to avoid frontend dependency)
  - `metadata: object` - Conversation metadata with startTime, endTime, and messageCount

**Note**: Enhanced message types (`ChatMessage`, `SystemMessage`, `ToolMessage`, `ToolResultMessage`, etc.) are defined in `frontend/src/types.ts` for comprehensive frontend message handling.

## Claude Command Integration

The backend uses the Claude Code SDK to execute claude commands. The SDK internally handles the claude command execution with appropriate parameters including:

- `--output-format stream-json` - Returns streaming JSON responses
- `--verbose` - Includes detailed execution information
- `-p <message>` - Prompt mode with user message

The SDK returns three types of JSON messages:

1. **System messages** (`type: "system"`) - Initialization and setup information
2. **Assistant messages** (`type: "assistant"`) - Actual response content
3. **Result messages** (`type: "result"`) - Execution summary with costs and usage

## サブスクリプション版の動作概要

### 起動フロー
```
1. Deno ランタイム初期化 (backend/cli/deno.ts)
2. CLI引数解析 (--port, --host, --debug)
3. Claude CLI の存在確認 (claude --version)
4. Honoアプリケーション作成 (backend/app.ts)
5. 静的ファイル配信設定
6. APIルート設定
7. HTTPサーバー起動 (デフォルト localhost:8080)
```

### 主要APIエンドポイント
- `GET /api/projects` - プロジェクト一覧取得
- `GET /api/projects/:encodedProjectName/histories` - 会話履歴一覧
- `GET /api/projects/:encodedProjectName/histories/:sessionId` - 特定会話の詳細
- `POST /api/chat` - チャット実行（メイン機能）
- `POST /api/abort/:requestId` - リクエスト中断
- `GET /*` - SPA用フォールバック（index.html配信）

### チャット実行の詳細フロー (`POST /api/chat`)

#### 現在の実装（API版）
```typescript
// 1. リクエスト受信
ChatRequest {
  message: string,
  requestId: string,
  sessionId?: string,
  allowedTools?: string[],
  workingDirectory?: string
}

// 2. Claude Code SDK経由で実行
query({
  prompt: message,
  options: {
    abortController,
    executable: "node",
    pathToClaudeCodeExecutable: claudePath,
    resume: sessionId,
    allowedTools,
    cwd: workingDirectory
  }
})

// 3. SDK出力をストリーミング処理
for await (const sdkMessage of query(...)) {
  // JSON形式でフロントエンドに送信
  yield {
    type: "claude_json",
    data: sdkMessage
  }
}
```

#### 修正後（サブスクリプション版）
```typescript
// 1. 同じリクエスト形式を受信

// 2. 直接claudeコマンド実行
const claudeArgs = [
  message,
  ...(sessionId ? ['--resume', sessionId] : []),
  ...(workingDirectory ? ['--cwd', workingDirectory] : []),
  '--stream'
];

const process = spawn('claude', claudeArgs, {
  cwd: workingDirectory,
  stdio: ['pipe', 'pipe', 'pipe']
});

// 3. プロセス出力をストリーミング処理
for await (const chunk of process.stdout) {
  const output = new TextDecoder().decode(chunk);
  // 出力をパースしてフロントエンドに送信
}
```

### ストリーミング通信
```
[フロントエンド] 
    ↓ WebSocket/fetch (POST /api/chat)
[バックエンド]
    ↓ Claude Code SDK/Direct Command
[Claude]
    ↑ NDJSON Stream
[バックエンド]
    ↑ NDJSON Stream 
[フロントエンド]
```

### セッション・履歴管理
```
~/.claude/projects/
├── {encoded-project-name}/
│   ├── session-123.jsonl
│   ├── session-456.jsonl
│   └── ...
```

- 各セッションはJSONLファイルとして保存
- プロジェクトパスは`~/.claude.json`から取得
- エンコードされたディレクトリ名で履歴管理

### 並行処理とリソース管理
```typescript
// リクエスト毎のAbortController管理
const requestAbortControllers = new Map<string, AbortController>();

// リクエスト開始時
abortController = new AbortController();
requestAbortControllers.set(requestId, abortController);

// 中断時
POST /api/abort/:requestId → abortController.abort()

// 完了時
requestAbortControllers.delete(requestId);
```

### エラーハンドリング
```typescript
// 段階的エラー処理
1. Claude CLI存在確認エラー → 警告ログ
2. コマンド実行エラー → StreamResponse error
3. パース エラー → デバッグログ + 継続
4. ネットワークエラー → HTTP 500
```

### 設定とランタイム抽象化
```typescript
// Runtime interface による抽象化
interface Runtime {
  readTextFile(path: string): Promise<string>;
  runCommand(command: string, args: string[]): Promise<CommandResult>;
  spawn(command: string, args: string[], options: SpawnOptions): Process;
  // ...
}

// Deno実装
class DenoRuntime implements Runtime {
  // Deno固有の実装
}
```

## 修正による変更点

| 項目 | 現在（API版） | 修正後（サブスクリプション版） |
|------|---------------|--------------------------------|
| **実行方法** | Claude Code SDK | 直接`claude`コマンド |
| **認証** | API Key設定 | サブスクリプション前提 |
| **プロセス管理** | SDK内部処理 | 子プロセス直接管理 |
| **出力形式** | SDK標準化済み | Raw出力をパース |
| **エラー処理** | SDK統一形式 | コマンドレベルエラー |

## Session Continuity

The application supports conversation continuity within the same chat session using Claude Code SDK's built-in session management.

### How It Works

1. **Initial Message**: First message in a chat session starts a new Claude session
2. **Session Tracking**: Frontend automatically extracts `session_id` from incoming SDK messages
3. **Continuation**: Subsequent messages include the `session_id` to maintain conversation context
4. **Backend Integration**: Backend passes `session_id` to Claude Code SDK via `options.resume` parameter

### Technical Implementation

- **Frontend**: Tracks `currentSessionId` state and includes it in API requests
- **Backend**: Accepts optional `sessionId` in `ChatRequest` and uses it with SDK's `resume` option
- **Streaming**: Session IDs are extracted from all SDK message types (`system`, `assistant`, `result`)
- **Automatic**: No user intervention required - session continuity is handled transparently

### Benefits

- **Context Preservation**: Maintains conversation context across multiple messages
- **Improved UX**: Users can reference previous messages and build on earlier discussions
- **Efficient**: Leverages Claude Code SDK's native session management
- **Seamless**: Works automatically without user configuration

## サブスクリプション版のドキュメント更新

### README.md の修正点

```markdown
# Claude Code Web UI - サブスクリプション版

## 前提条件

- Claude Code CLI がインストールされていること
- Claude サブスクリプションが有効であること

## セットアップ

1. Claude Code CLI のインストール
   ```bash
   # Claude Code CLI のインストール
   npm install -g @anthropic-ai/claude-code
   ```

2. WebUI の起動
   ```bash
   # 依存関係のインストール
   npm install
   
   # 開発サーバーの起動
   npm run dev
   ```

## 注意事項

- Claude サブスクリプションが必要です
- API キーの設定は不要です
```

### package.json の scripts 更新

```json
{
  "scripts": {
    "dev": "npm run dev:backend & npm run dev:frontend",
    "dev:backend": "cd backend && deno run --env-file --allow-net --allow-run --allow-read --allow-env --watch cli/deno.ts --debug",
    "dev:frontend": "cd frontend && npm run dev"
  }
}
```

### エラーハンドリングの調整

```typescript
// backend/handlers/chat.ts での基本的なエラーハンドリング
function handleExecutionError(error: Error): StreamResponse {
  return {
    type: "error",
    error: `Claude execution failed: ${error.message}`
  };
}
```

### テストの更新

基本機能テストの維持
既存のテスト構造を維持し、API関連のテストのみを直接実行用に調整します。

```typescript
// backend/handlers/chat.test.ts（修正例）
Deno.test("Chat handler - direct execution", async () => {
  const mockRuntime = {
    spawn: () => ({
      stdout: createMockStream("Hello from Claude"),
      stderr: createMockStream(""),
      exitCode: Promise.resolve(0)
    })
  };

  // テスト実装...
});
```

### 本番環境での考慮事項

#### プロセス管理
- 子プロセスの適切な管理とクリーンアップ
- プロセス数の制限とリソース管理

#### エラーログ
- Claude CLI の実行エラーの適切なログ記録
- ユーザーへの分かりやすいエラーメッセージ

#### パフォーマンス
- 並行実行の最適化
- メモリ使用量の監視

## Development

### Prerequisites

- Deno (for backend)
- Node.js (for frontend)
- Claude CLI tool installed and configured

### Port Configuration

The application supports flexible port configuration for development:

#### Unified Backend Port Management

Create a `.env` file in the project root to set the backend port:

```bash
# .env
PORT=9000
```

Both backend startup and frontend proxy configuration will automatically use this port:

```bash
cd backend && deno task dev     # Starts backend on port 9000
cd frontend && npm run dev      # Configures proxy to localhost:9000
```

#### Alternative Configuration Methods

- **Environment Variable**: `PORT=9000 deno task dev`
- **CLI Argument**: `deno run --env-file --allow-net --allow-run --allow-read --allow-env cli/deno.ts --port 9000`
- **Frontend Port**: `npm run dev -- --port 4000` (for frontend UI port)

### Running the Application

1. **Start Backend**:

   ```bash
   cd backend
   deno task dev
   ```

2. **Start Frontend**:

   ```bash
   cd frontend
   npm run dev
   ```

3. **Access Application**:
   - Frontend: http://localhost:3000 (or custom port via `npm run dev -- --port XXXX`)
   - Backend API: http://localhost:8080 (or PORT from .env file)

### Project Structure

```
├── backend/           # Deno backend server with runtime abstraction
│   ├── deno.json     # Deno configuration with permissions
│   ├── app.ts        # Runtime-agnostic core application
│   ├── types.ts      # Backend-specific type definitions
│   ├── VERSION       # Version file for releases
│   ├── cli/          # CLI-specific entry points
│   │   ├── deno.ts           # Deno entry point and server startup
│   │   └── args.ts           # CLI argument parsing with runtime abstraction
│   ├── runtime/      # Runtime abstraction layer
│   │   ├── types.ts          # Runtime interface definitions
│   │   └── deno.ts           # Deno runtime implementation
│   ├── handlers/     # API handlers using runtime abstraction
│   │   ├── abort.ts         # Request abortion handler
│   │   ├── chat.ts          # Chat streaming handler
│   │   ├── conversations.ts # Conversation details handler
│   │   ├── histories.ts     # History listing handler
│   │   └── projects.ts      # Project listing handler
│   ├── history/      # History processing utilities
│   │   ├── conversationLoader.ts  # Load specific conversations
│   │   ├── grouping.ts             # Group conversation files
│   │   ├── parser.ts               # Parse history files
│   │   ├── pathUtils.ts            # Path validation utilities
│   │   └── timestampRestore.ts     # Restore message timestamps
│   ├── middleware/   # Middleware modules
│   │   └── config.ts        # Configuration middleware with runtime injection
│   ├── pathUtils.test.ts    # Path utility tests with mock runtime
│   └── dist/         # Frontend build output (copied during build)
├── frontend/         # React frontend application
│   ├── src/
│   │   ├── App.tsx   # Main application component with routing
│   │   ├── main.tsx  # Application entry point
│   │   ├── types.ts  # Frontend-specific type definitions
│   │   ├── config/
│   │   │   └── api.ts                 # API configuration and URLs
│   │   ├── utils/
│   │   │   ├── constants.ts           # UI and application constants
│   │   │   ├── messageTypes.ts        # Type guard functions for messages
│   │   │   ├── toolUtils.ts           # Tool-related utility functions
│   │   │   ├── time.ts                # Time utilities
│   │   │   ├── id.ts                  # ID generation utilities
│   │   │   ├── messageConversion.ts   # Message conversion utilities
│   │   │   └── mockResponseGenerator.ts # Demo response generator
│   │   ├── hooks/
│   │   │   ├── useClaudeStreaming.ts  # Simplified streaming interface
│   │   │   ├── useTheme.ts            # Theme management hook
│   │   │   ├── useHistoryLoader.ts    # History loading hook
│   │   │   ├── useMessageConverter.ts # Message conversion hook
│   │   │   ├── useDemoAutomation.ts   # Demo automation hook
│   │   │   ├── chat/
│   │   │   │   ├── useChatState.ts    # Chat state management
│   │   │   │   ├── usePermissions.ts  # Permission handling logic
│   │   │   │   └── useAbortController.ts # Request abortion logic
│   │   │   └── streaming/
│   │   │       ├── useMessageProcessor.ts # Message creation and processing
│   │   │       ├── useToolHandling.ts     # Tool-specific message handling
│   │   │       └── useStreamParser.ts     # Stream parsing and routing
│   │   ├── components/
│   │   │   ├── ChatPage.tsx           # Main chat interface page
│   │   │   ├── ProjectSelector.tsx    # Project directory selection page
│   │   │   ├── MessageComponents.tsx  # Message display components
│   │   │   ├── PermissionDialog.tsx   # Permission handling dialog
│   │   │   ├── TimestampComponent.tsx # Timestamp display
│   │   │   ├── HistoryView.tsx        # Conversation history view
│   │   │   ├── DemoPage.tsx           # Demo mode page
│   │   │   ├── DemoPermissionDialogWrapper.tsx # Demo permission wrapper
│   │   │   ├── chat/
│   │   │   │   ├── ThemeToggle.tsx    # Theme toggle button
│   │   │   │   ├── ChatInput.tsx      # Chat input component
│   │   │   │   ├── ChatMessages.tsx   # Chat messages container
│   │   │   │   └── HistoryButton.tsx  # History access button
│   │   │   └── messages/
│   │   │       ├── MessageContainer.tsx   # Reusable message wrapper
│   │   │       └── CollapsibleDetails.tsx # Collapsible content component
│   │   ├── types/
│   │   │   └── window.d.ts     # Window type extensions
│   │   ├── scripts/            # Demo recording scripts
│   │   │   ├── record-demo.ts         # Demo recorder
│   │   │   ├── demo-constants.ts      # Demo configuration
│   │   │   └── compare-demo-videos.ts # Demo comparison
│   │   ├── tests/              # End-to-end tests
│   │   │   └── demo-validation.spec.ts # Demo validation tests
│   │   ├── package.json
│   │   └── vite.config.ts     # Vite config with @tailwindcss/vite plugin
├── shared/           # Shared TypeScript types
│   └── types.ts
├── CLAUDE.md        # Technical documentation
└── README.md        # User documentation
```

## Key Design Decisions

1. **Runtime Abstraction Architecture**: Complete separation between business logic and platform-specific code using a minimal Runtime interface. All handlers, utilities, and CLI components use runtime abstraction instead of direct Deno APIs, enabling comprehensive testing with mock runtime and future platform flexibility.

2. **Modular Entry Points**: CLI-specific code separated into `cli/` directory with `deno.ts` as the main entry point, while `app.ts` contains the runtime-agnostic core application. This enables clean separation of concerns and potential future Node.js implementation.

3. **Raw JSON Streaming**: Backend passes Claude JSON responses without modification to allow frontend flexibility in handling different message types.

4. **Configurable Ports**: Backend port configurable via PORT environment variable or CLI argument, frontend port via CLI argument to allow independent development and deployment.

5. **TypeScript Throughout**: Consistent TypeScript usage across all components with shared type definitions.

6. **TailwindCSS Styling**: Uses @tailwindcss/vite plugin for utility-first CSS without separate CSS files.

7. **Theme System**: Light/dark theme toggle with automatic system preference detection and localStorage persistence.

8. **Project Directory Selection**: Users choose working directory before starting chat sessions, with support for both configured projects and custom directory selection.

9. **Routing Architecture**: React Router separates project selection and chat interfaces for better user experience.

10. **Dynamic Working Directory**: Claude commands execute in user-selected project directories for contextual file access.

11. **Request Management**: Unique request IDs enable request tracking and abort functionality for better user control.

12. **Tool Permission Handling**: Frontend permission dialog allows users to grant/deny tool access with proper state management.

13. **Comprehensive Error Handling**: Enhanced error states and user feedback for better debugging and user experience.

14. **Modular Architecture**: Frontend code is organized into specialized hooks and components for better maintainability and testability.

15. **Separation of Concerns**: Business logic, UI components, and utilities are clearly separated into different modules.

16. **Configuration Management**: Centralized configuration for API endpoints and application constants.

17. **Reusable Components**: Common UI patterns are extracted into reusable components to reduce duplication.

18. **Hook Composition**: Complex functionality is built by composing smaller, focused hooks that each handle a specific concern.

## Claude Code SDK Types Reference

**SDK Types**: `frontend/node_modules/@anthropic-ai/claude-code/sdk.d.ts`

### Common Patterns
```typescript
// Type extraction
const systemMsg = sdkMessage as Extract<SDKMessage, { type: "system" }>;
const assistantMsg = sdkMessage as Extract<SDKMessage, { type: "assistant" }>;
const resultMsg = sdkMessage as Extract<SDKMessage, { type: "result" }>;

// Assistant content access (nested structure!)
for (const item of assistantMsg.message.content) {
  if (item.type === "text") {
    const text = (item as { text: string }).text;
  } else if (item.type === "tool_use") {
    const toolUse = item as { name: string; input: Record<string, unknown> };
  }
}

// System message (no .message property)
console.log(systemMsg.cwd); // Direct access, no nesting
```

### Key Points
- **System**: Fields directly on object (`systemMsg.cwd`, `systemMsg.tools`)
- **Assistant**: Content nested under `message.content` 
- **Result**: Has `subtype` field (`success` | `error_max_turns` | `error_during_execution`)
- **Type Safety**: Always use `Extract<SDKMessage, { type: "..." }>` for narrowing

## Frontend Architecture Benefits

The modular frontend architecture provides several key benefits:

### Code Organization
- **Reduced File Size**: Main App.tsx reduced from 467 to 262 lines (44% reduction)
- **Focused Responsibilities**: Each file has a single, clear purpose
- **Logical Grouping**: Related functionality is organized into coherent modules

### Maintainability
- **Easier Debugging**: Issues can be isolated to specific modules
- **Simplified Testing**: Individual components and hooks can be tested in isolation
- **Clear Dependencies**: Import structure clearly shows component relationships

### Reusability
- **Shared Components**: `MessageContainer` and `CollapsibleDetails` reduce UI duplication
- **Utility Functions**: Common operations are centralized and reusable
- **Configuration**: API endpoints and constants are easily configurable

### Developer Experience
- **Type Safety**: Enhanced TypeScript coverage with stricter type definitions
- **IntelliSense**: Better IDE support with smaller, focused modules
- **Hot Reload**: Faster development cycles with smaller change surfaces

### Performance
- **Bundle Optimization**: Tree-shaking is more effective with modular code
- **Code Splitting**: Easier to implement lazy loading for large features
- **Memory Efficiency**: Reduced memory footprint with focused hooks

## Testing

The project includes comprehensive test suites for both frontend and backend components:

### Frontend Testing

- **Framework**: Vitest with Testing Library
- **Coverage**: Component testing, hook testing, and integration tests
- **Location**: Tests are co-located with source files (`*.test.ts`, `*.test.tsx`)
- **Run**: `make test-frontend` or `cd frontend && npm run test:run`

### Backend Testing  

- **Framework**: Deno's built-in test runner with std/assert
- **Coverage**: Path encoding utilities, API handlers, and integration tests
- **Location**: `backend/pathUtils.test.ts` and other `*.test.ts` files
- **Run**: `make test-backend` or `cd backend && deno task test`

### Unified Testing

- **All Tests**: `make test` - Runs both frontend and backend tests
- **Quality Checks**: `make check` - Includes tests in pre-commit quality validation
- **CI Integration**: GitHub Actions automatically runs all tests on push/PR

## Single Binary Distribution

The project supports creating self-contained executables for all major platforms:

### Local Building

```bash
# Build for current platform
cd backend && deno task build

# Cross-platform builds are handled by GitHub Actions
```

### Automated Releases

- **Trigger**: Push git tags (e.g., `git tag v1.0.0 && git push origin v1.0.0`)
- **Platforms**: Linux (x64/ARM64), macOS (x64/ARM64)
- **Output**: GitHub Releases with downloadable binaries
- **Features**: Frontend is automatically bundled into each binary

## Claude Code Dependency Management

### Current Version Policy

Both frontend and backend use **fixed versions** (without caret `^`) to ensure consistency:

- **Frontend**: `frontend/package.json` - `"@anthropic-ai/claude-code": "1.0.33"`
- **Backend**: `backend/deno.json` imports - `"@anthropic-ai/claude-code": "npm:@anthropic-ai/claude-code@1.0.33"`

### Version Update Procedure

When updating to a new Claude Code version (e.g., 1.0.40):

1. **Check current versions**:
   ```bash
   # Frontend
   grep "@anthropic-ai/claude-code" frontend/package.json
   
   # Backend  
   grep "@anthropic-ai/claude-code" backend/deno.json
   ```

2. **Update Frontend**:
   ```bash
   # Edit frontend/package.json - change version number
   # "@anthropic-ai/claude-code": "1.0.XX"
   cd frontend && npm install
   ```

3. **Update Backend**:
   ```bash
   # Edit backend/deno.json imports - change version number
   # "@anthropic-ai/claude-code": "npm:@anthropic-ai/claude-code@1.0.XX"
   cd backend && rm deno.lock && deno cache main.ts
   ```

4. **Verify and test**:
   ```bash
   make check
   ```

### Version Consistency Check

Ensure both environments use the same version:
```bash
# Should show the same version number
grep "@anthropic-ai/claude-code" frontend/package.json backend/deno.json
```

## Commands for Claude

### Unified Commands (from project root)

- **Format**: `make format` - Format both frontend and backend
- **Lint**: `make lint` - Lint both frontend and backend
- **Type Check**: `make typecheck` - Type check both frontend and backend
- **Test**: `make test` - Run both frontend and backend tests
- **Quality Check**: `make check` - Run all quality checks before commit
- **Format Specific Files**: `make format-files FILES="file1 file2"` - Format specific files with prettier

### Individual Commands

- **Development**: `make dev-backend` / `make dev-frontend`
- **Testing**: `make test-frontend` / `make test-backend`
- **Build Binary**: `make build-backend`
- **Build Frontend**: `make build-frontend`

**Note**: Lefthook automatically runs `make check` before every commit. GitHub Actions will also run all quality checks on push and pull requests.

## Development Workflow

### Pull Request Process

1. Create a feature branch from `main`: `git checkout -b feature/your-feature-name`
2. Make your changes and commit them (Lefthook runs `make check` automatically)
3. Push your branch and create a pull request
4. **Add appropriate labels** to categorize the changes (see Labels section below)
5. **Include essential PR information** as outlined in the Labels section
6. Request review and address feedback
7. Merge after approval and CI passes

#### Creating Pull Requests

Create pull requests with appropriate labels and essential information:

```bash
gh pr create --title "Your PR Title" \
  --label "appropriate,labels" \
  --body "Brief description"
```

**Note**: CHANGELOG.md is now automatically managed by tagpr - no manual updates needed!

### Labels

The project uses the following labels for categorizing pull requests and issues:

- 🐛 **`bug`** - Bug fixes (non-breaking changes that fix issues)
- ✨ **`feature`** - New features (non-breaking changes that add functionality)
- 💥 **`breaking`** - Breaking changes (changes that would cause existing functionality to not work as expected)
- 📚 **`documentation`** - Documentation improvements or additions
- ⚡ **`performance`** - Performance improvements
- 🔨 **`refactor`** - Code refactoring (no functional changes)
- 🧪 **`test`** - Adding or updating tests
- 🔧 **`chore`** - Maintenance, dependencies, tooling updates
- 🖥️ **`backend`** - Backend-related changes
- 🎨 **`frontend`** - Frontend-related changes

**For Claude**: When creating PRs, always include:

1. **Type of Change checkboxes**: Include the checkbox list from the template to categorize changes:
   ```
   - [ ] 🐛 `bug` - Bug fix (non-breaking change which fixes an issue)
   - [ ] ✨ `feature` - New feature (non-breaking change which adds functionality)
   - [ ] 💥 `breaking` - Breaking change
   - [ ] 📚 `documentation` - Documentation update
   - [ ] ⚡ `performance` - Performance improvement
   - [ ] 🔨 `refactor` - Code refactoring
   - [ ] 🧪 `test` - Adding or updating tests
   - [ ] 🔧 `chore` - Maintenance, dependencies, tooling
   - [ ] 🖥️ `backend` - Backend-related changes
   - [ ] 🎨 `frontend` - Frontend-related changes
   ```
2. **Description**: Brief summary of what changed and why
3. **GitHub labels**: Add corresponding labels using `--label` flag: `gh pr create --label "feature,documentation"`
4. **Test plan**: Include testing information if relevant

Multiple labels can be applied if the PR covers multiple areas.

### Release Process (Automated with tagpr)

1. **Feature PRs merged to main** → tagpr automatically creates/updates release PR
2. **Add version labels** to PRs if needed:
   - No label = patch version (v1.0.0 → v1.0.1)
   - `minor` label = minor version (v1.0.0 → v1.1.0)
   - `major` label = major version (v1.0.0 → v2.0.0)
3. **Review and merge release PR** → tagpr creates git tag automatically
4. **GitHub Actions builds binaries** and creates GitHub Release automatically
5. Update documentation if needed

**Manual override**: Edit `backend/VERSION` file directly if specific version needed

### GitHub Sub-Issues API

**For Claude**: When creating sub-issues to break down larger features:

```bash
# 1. Create the sub-issue normally
gh issue create --title "Sub-issue title" --body "..." --label "feature,enhancement"

# 2. Get the sub-issue ID
SUB_ISSUE_ID=$(gh api repos/owner/repo/issues/ISSUE_NUMBER --jq '.id')

# 3. Add it as sub-issue to parent issue
gh api repos/owner/repo/issues/PARENT_ISSUE_NUMBER/sub_issues \
  --method POST \
  --field sub_issue_id=$SUB_ISSUE_ID

# 4. Verify the relationship
gh api repos/owner/repo/issues/PARENT_ISSUE_NUMBER/sub_issues
```

**Key points**:
- Use issue **ID** (not number) for `sub_issue_id` parameter
- Endpoint is `/sub_issues` (plural) for POST operations
- Parent issue will show `sub_issues_summary` with total/completed counts
- Sub-issues automatically link to parent in GitHub UI

### Viewing Copilot Review Comments

**For Claude**: Copilot inline review comments are not shown in regular `gh pr view` output. To see them:

```bash
# View all inline review comments from Copilot
gh api repos/owner/repo/pulls/PR_NUMBER/comments

# Example for this repository
gh api repos/sugyan/claude-code-webui/pulls/39/comments
```

**Why this matters**:
- Copilot provides valuable code improvement suggestions
- These comments include security, performance, and code quality feedback
- They appear as inline comments on specific lines of code
- Missing these can lead to suboptimal code being merged
- Always check for Copilot feedback when reviewing PRs

**Important for Claude**: Always run commands from the project root directory. When using `cd` commands for backend/frontend, use full paths like `cd /path/to/project/backend` to avoid getting lost in subdirectories.
