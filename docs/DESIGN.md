# powerful-desktop 設計書

開発業務（GitHub Project / Notion / コード生成 AI エージェント）を 1 つのデスクトップアプリで完結させる。対象 OS は macOS と Windows。

## 1. 目的とスコープ

### 機能
| # | 機能 | MVP |
|---|---|---|
| F1 | GitHub Project（Projects v2）の閲覧・更新 | 連携フェーズ |
| F2 | Notion のページ・DB の閲覧・編集 | 連携フェーズ |
| F3 | Claude Code など AI エージェントへの指示出し | ◎ |
| F4 | 指定セッションの差分確認 | ◎ |
| F5 | 指定リポジトリ・ブランチの状態確認 | ◎（GitHub 認証を含む） |
| F6 | 異なる AI（Claude Code / Notion AI / Copilot）同士の会話による自律作業 | オーケストレーションフェーズ |

### スコープ外（初期）
- 複数セッションの並列実行（拡張可能な構造だけ用意する。§6）
- Claude のサブスク認証を自アプリに組み込むこと（§4.1）

## 2. 技術スタック

| 層 | 選定 |
|---|---|
| シェル | Electron + electron-vite + electron-builder |
| プロセス構成 | Main（窓管理・IPC 中継・秘密情報）+ Backend `utilityProcess` + Pty Host `utilityProcess`（§3） |
| 言語 | TypeScript |
| UI | React + TanStack Query + Zustand |
| IPC | 制御系: 型付き IPC（自前のチャネル定義。tRPC-electron はサブスクリプション対応と保守状況を見て判断）。PTY データ: `MessagePort` で Pty Host と Renderer を直結 |
| ターミナル | node-pty + xterm.js（WebGL レンダラ。Windows は ConPTY） |
| Git | system `git` + simple-git、差分表示は Monaco Diff Editor（遅延読み込み） |
| GitHub | Octokit GraphQL（Projects v2 は GraphQL のみ）、認証は Device Flow（§2.2） |
| Notion | `@notionhq/client`（REST）、Agents API、WebContentsView。認証は §2.2 |
| 永続化 | SQLite（Backend の worker_threads 内。`node:sqlite` を第一候補、不可なら better-sqlite3）+ Drizzle（`node:sqlite` ドライバ対応は要確認） |
| 秘密情報 | Electron `safeStorage`（Keychain / DPAPI） |
| 自動更新 | electron-updater（配信元は未定） |
| 署名 | macOS: Developer ID 署名 + notarization、Windows: コード署名（OV/EV 証明書または Azure Trusted Signing） |
| テスト | Vitest（単体）、Playwright の Electron サポート（E2E） |
| ログ・クラッシュ | electron-log + Electron `crashReporter` |
| パッケージ管理 | pnpm |

Electron を選ぶ理由:
- Copilot SDK（Node.js あり）や Notion Agents SDK が JS/TS で提供されており、Node プロセスで直接扱える。Tauri は sidecar として Node を同梱する必要があり、ランタイムのサイズ・メモリ面の利点が薄い。
- Notion webview 方式（§4.2）は CDP（`webContents.debugger`）に依存する。Tauri の macOS（WKWebView）は CDP を持たない。
- Monaco / xterm.js の描画を Chromium に統一でき、OS ごとの描画差を検証せずに済む。

### 2.1 性能方針

Electron 本体の基本コスト（アイドル時 150〜300MB 程度）は、`claude` プロセスや Notion webview の消費に比べて小さい。体感性能を決めるのは、どのプロセスのイベントループも塞がないことと、PTY 出力の流量制御である。

- Main プロセスでは同期 I/O・重い処理を行わない。業務ロジックは Backend、PTY は Pty Host に置く（§3）。
- SQLite は同期 API のため Backend の worker_threads 内でのみ使い、書き込みはバッチ化する。
- PTY → Renderer は `MessagePort` で直結し、一定間隔でバッチ送信する。流量制御は ACK 方式にする（Renderer が xterm.js の `write` コールバック後に処理済み文字数を返し、未 ACK 量が閾値を超えたら Pty Host が送信を止める）。ConPTY では node-pty の `pause()` が期待どおり効かない場合があるため、それに依存しない。
- xterm.js の WebGL コンテキスト数には上限がある。表示中のターミナルだけ WebGL を使い、非表示のものは DOM レンダラに切り替えるか破棄する。
- Monaco は差分ビューを開いたときに読み込む。
- Notion webview は使用時のみ生成し、非表示が続いたら破棄できるようにする。
- ネイティブモジュール（node-pty、better-sqlite3 を使う場合）:
  - mac arm64 / x64、Windows x64 の全組み合わせで CI ビルドする。
  - `asarUnpack` で asar の外に出す。
  - macOS では node-pty の `spawn-helper` に実行権限を付け、署名・notarization の対象に含める。

### 2.2 認証方式（要確認事項を含む）

- **GitHub**（MVP から必要。F5 で private リポジトリの状態を取得するため）
  - OAuth App の Device Flow を基本とする。OAuth App の設定で Device Flow を有効化する。
  - スコープ: `repo`（private リポジトリ、PR 作成）、`project`（Projects v2）、`read:org`。
  - 会社の org で OAuth App アクセス制限が有効な場合、org 管理者の承認が必要。導入前に手続きを確認する。
  - 代替として、ホストの `gh auth token` を流用するオプションを用意する。
  - GitHub App は個人アカウント所有の Projects v2 にアクセスできない制約がある（要確認）ため、対象を org の Project に限る場合のみ選択肢とする。GitHub App の秘密鍵はアプリに同梱しない。
- **Notion**
  - Public integration の OAuth はトークン交換に `client_secret` が必要で、デスクトップアプリに埋め込めない。次のいずれかを選ぶ。
    - トークン交換だけを行う小さなサーバを用意する。
    - ユーザーが作成した内部インテグレーションのトークンを入力してもらう（MVP はこちら）。
  - 内部インテグレーション方式の前提:
    - インテグレーションの作成にはワークスペースの権限が必要で、管理ポリシーで禁止されている場合がある。導入前に管理者へ確認する。
    - アクセスできるのは、ユーザーがインテグレーションを明示的に接続したページ・DB だけ。接続手順を UI で案内する。

## 3. アーキテクチャ

```
Renderer (React, sandbox)
   │ typed IPC（制御）                ┐
Main process                          │ MessagePort（PTY データ、ACK）
 ├─ BrowserWindow / WebContentsView 管理
 ├─ CdpProxy（webContents.debugger の allowlist 付き中継）
 ├─ SecretStore（safeStorage）        │
 └─ IPC 中継                          │
   │ MessagePort                      │
   ├──────────────┐                   │
Backend           Pty Host  ←─────────┘
utilityProcess    utilityProcess
 │                 └─ node-pty（claude / 汎用 CLI）
 ├─ HookReceiver   (127.0.0.1 の HTTP。Claude Code の http hook を受ける)
 ├─ Orchestrator
 │    ├─ Conversation (participants[], turn policy, limits)
 │    ├─ Workspace = worktree（Conversation が所有）
 │    └─ Session = AgentAdapter instance（Workspace を参照。任意）
 ├─ AgentAdapter
 │    ├─ ClaudeAdapter   (Pty Host の PTY + hooks + transcript)
 │    ├─ CopilotAdapter  (Copilot SDK)
 │    ├─ NotionAIAdapter (agents-api | webview。webview は CdpProxy 経由)
 │    └─ GenericPtyAdapter (任意の CLI)
 ├─ GitService      (worktree / status / diff)
 ├─ GitHubService   (Projects v2 GraphQL)
 ├─ NotionService   (REST)
 └─ Store (SQLite, worker_threads)
```

- **Main** は窓管理、秘密情報、IPC 中継、CDP 中継だけを担う。Main が止まると UI 全体が固まるため、業務ロジックを置かない。
- **Pty Host** は PTY だけを持つ。Backend がクラッシュ・再起動しても PTY と `claude` プロセスは生き残る。
- **Backend がクラッシュした場合**: Main が再起動し、Backend は DB から Session を復元して Pty Host に再接続する。
- **Pty Host がクラッシュした場合**: `claude --resume <session_id>` でセッションを再開する。
- **孤児プロセスの回収**: 起動した子プロセスの PID を DB に記録し、起動時に孤児を回収する。
- **Notion webview**: WebContentsView は Main が所有する。自動操作のロジックと DOM セレクタは Backend の NotionAIAdapter に置き、注入スクリプトとして CdpProxy に送る。CdpProxy は許可した CDP メソッド（`Runtime.evaluate` など）だけを中継する。
- **秘密情報の受け渡し**: Main が `safeStorage` で復号し、Backend 起動時に MessagePort 経由で渡す。Backend はメモリ上にだけ保持し、ディスク・ログ・子プロセスの環境変数には出さない。
- **並列化フェーズ（§10-5）**: Session ごとに Backend を分ける構成に拡張できる。

### AgentAdapter インターフェース

```ts
interface AgentAdapter {
  start(opts: { sessionId: string; cwd?: string }): Promise<void> // cwd は Workspace を使うエージェントのみ
  send(message: string): Promise<void>
  events(): AsyncIterable<AgentEvent>
  interrupt(): Promise<void>
  resume(): Promise<void>
  approve(requestId: string, decision: "allow" | "deny"): Promise<void>
  dispose(): Promise<void>
}
```

`AgentEvent` は正規化して保存する: `message`（assistant / user）、`tool_request`、`tool_result`、`file_edit`、`idle`（応答完了）、`error`。

## 4. エージェント別の方式

### 4.1 Claude Code（ホストの `claude` を起動）

公式ドキュメントでは、サードパーティ製品が claude.ai ログインを提供することは認められていない。このため Agent SDK を組み込んでサブスク認証を使う方式は採らない。

**起動**
- ユーザーのホストにインストール済みの `claude` を **PTY で対話モード起動**し、xterm.js で表示する。AI への入力は PTY への書き込みで行う。
- 実行ファイルは絶対パスに解決し、設定画面で上書きできるようにする。
  - macOS では Finder / Dock から起動したアプリはログインシェルの PATH を継承しない。起動時にログインシェルから環境変数を取得する（shell-env 相当）。
  - Windows ではインストール形態（ネイティブ版か npm の `.cmd` shim か）で起動方法が異なる。Git Bash が必要（`CLAUDE_CODE_GIT_BASH_PATH`）。

**hooks の注入**
- worktree の `.claude/settings.local.json`（gitignore 対象）か、アプリ専用 plugin で注入する。ユーザーのグローバル設定やリポジトリの共有設定は書き換えない。
- hook は `http` 型を使い、HookReceiver（127.0.0.1）に POST させる。`command` 型は Windows で実行シェルが変わるため使わない。
- ユーザー設定（`disableAllHooks`）や managed policy で hooks が無効な場合は検出し、承認 UI を使わず TUI だけで操作する縮退モードに落とす。

**承認 UI**
- `PermissionRequest` hook はアプリの判断が出るまで応答を保留し、`hookSpecificOutput.decision.behavior: "allow" | "deny"` を返す。これにより承認操作の経路をアプリ UI に一本化する。
- アプリが応答しないまま hook のタイムアウト（既定 600 秒）が近づいたら、判断をせず TUI 側のダイアログに委ねる。
- 保留中に TUI 側で操作された場合の扱い（どちらが優先されるか）はスパイクで確認する。
- 承認の決定は必ずアプリの UI から出す。hook やエージェント側からの承認入力は受け付けない。

**構造化情報の取得**
- hooks: 承認待ち、ツール実行、停止を検知する。応答完了（`idle`）は `Stop` hook（`last_assistant_message`）を一次情報とする。
- transcript: hook 入力の `transcript_path` / `session_id` を使って対象ファイルを特定し、メッセージとファイル編集を取得する。`~/.claude/projects/**` を glob で監視しない（`CLAUDE_CONFIG_DIR` で場所が変わるため）。transcript は非同期に書かれるため、遅延を前提にする。

**HookReceiver の脅威モデル**
- 127.0.0.1 に bind し、Host ヘッダを検証する。起動ごとのランダムトークンを hook 設定に埋め込んで照合する。
- このトークンで防げるのは、ブラウザ経由の CSRF / DNS rebinding と他ユーザーのプロセスまで。同じユーザーのプロセス（`claude` の Bash ツールが実行するコマンドを含む）は、トークンを読める前提で設計する。そのため受信口は通知と判断待ちだけを扱い、承認を確定させる入力は受け付けない。

**`claude -p --output-format stream-json`**
- 構造化された出力が得られるため、PTY 解析より制御面で優れる。承認 UI を成立させられるかを比較軸に、第 2 バックエンドとして評価する。
- サブスク利用での課金扱いは、一度変更が告知された後に撤回されたとの情報がある。公式サポート記事で最新の扱いを確認してから採否を決める（§9-3）。
- API キー運用に切り替える場合に備え、バックエンドを差し替えられる構造にする。

### 4.2 Notion AI

公式 Agents API は **Custom Agents のみ**が対象で、alpha（ウェイトリスト）。個人の Notion AI チャットは対象外。SDK のメソッド名と認証方式は alpha 参加後に確認する。

| バックエンド | 対象 | 方式 | リスク |
|---|---|---|---|
| `agents-api` | Custom Agents | 公式 SDK（チャット送信、スレッドのポーリング） | alpha。利用可否・認証方式に不確実性 |
| `webview` | 個人の Notion AI チャットを含む | WebContentsView に Notion を表示し、CDP（`webContents.debugger`）または preload で入力・応答取得 | DOM 変更に弱い。規約は要確認。SSO ログインが通らない可能性 |

webview 方式の方針:
- ログインはユーザー自身が行い、永続 partition に保存する。認証情報はアプリで保持しない。
- Google SSO や SAML SSO は、埋め込みブラウザからのサインインがブロックされる場合がある。スパイクで、実際に使うログイン方式で webview にログインできることを確認する。
- DOM セレクタは 1 ファイルに隔離する（Backend 側。§3）。
- 応答完了は MutationObserver でストリーム終了状態を検知する。
- ユーザーが画面を見て手動介入できる（半自動に落とせる）。

撤退基準: スパイクで webview のログインまたは安定した応答取得ができず、Agents API も利用できない場合は、F6 の参加者から Notion AI を外す。Notion は REST（F2）による仕様参照だけにする。

### 4.3 Copilot

Copilot SDK（Node.js）で Copilot CLI を JSON-RPC 経由で操作する。セッション、カスタムツール、MCP、hooks が使える。Copilot プランまたは BYOK が必要。

### 4.4 汎用 CLI

Codex、Gemini CLI などは PTY で起動する。構造化されないため制御は弱く、出力の完了検知はヒューリスティックになる。

## 5. AI 同士の会話（オーケストレーション）

- エージェントを直接つながず、**Orchestrator が仲介するメッセージバス**を置く。
- 1 つの `Conversation` が複数の `participant`（エージェント）と 0〜1 個の `Workspace`（worktree）を持つ。ターン制御はラウンドロビンまたは役割ベース。
- 役割例: Claude Code = 実装、Copilot = レビュー、Notion AI = 仕様参照。
- Workspace への書き込み権限を持つ participant は 1 つだけにする。他の participant は読み取り専用で参照する。
- 共有の「黒板」はまずローカル（DB または worktree 内のファイル）に置く。GitHub Issue / PR コメントや Notion ページへの同期は、区切りごとに人間の承認を経て行う。こうすることで、毎ターンの外部書き込みが承認ゲートで止まることを避ける。
- 暴走防止（必須）:
  - 上限の主軸は、最大ターン数、経過時間、ツール実行回数とする。トークン・金額の上限は、それを取得できるアダプタ（Copilot SDK、API キー運用時など）だけで適用する。PTY の `claude` や webview の Notion AI では直接取得できない。
  - 同一内容ループの検出
  - 破壊的操作（push、削除、外部書き込み）の人間承認ゲート

## 6. 並列拡張のための制約（MVP から守る）

- すべてのイベント・ログ・状態を `sessionId` キーで扱う。グローバルな「現在のエージェント」を持たない。
- Orchestrator は `Map<SessionId, Session>` を持つ。MVP は同時アクティブ数を 1 に制限するだけにする。
- 1 Session = 1 Adapter インスタンス + 1 PTY（または webview）。worktree は Session ではなく Workspace が持ち、Conversation 内の Session が共有する。
- DB: `conversations / workspaces / sessions / participants / messages / events`。AI 同士の会話は複数 participant の conversation で表現する。sessions には子プロセスの PID と Claude Code の `session_id` を記録する（§3 の復旧用）。

## 7. セキュリティ

- エージェントはユーザー権限でコードを実行する。worktree で隔離し、権限モードの既定は厳しめにする。
- 秘密情報は `safeStorage` に保存し、エージェントのコンテキストや子プロセスの環境変数に渡さない（受け渡し経路は §3）。
- `safeStorage`（特に Windows の DPAPI）は、同じユーザーで動く他プロセスからは秘密情報を守れない。他ユーザーやディスクの持ち出しへの対策と位置付ける。
- 外部書き込み（push、PR 作成、Notion/GitHub 更新）は既定で承認ゲートを通す。
- webview の Notion セッションはアプリ専用の partition に分離する。CDP は CdpProxy の allowlist にあるメソッドだけを許可する。
- Electron の既定値: すべての WebContents で `contextIsolation: true`、`sandbox: true`、`nodeIntegration: false`。preload は `contextBridge` で最小限の API だけ公開する。Notion webview の preload は応答取得に必要な処理だけにする。
- アプリ UI は CSP を設定し、外部 URL への遷移と `window.open` を既定で拒否する。
- hooks 受信口の脅威モデルと対策は §4.1 のとおり。
- Electron はサポート対象（最新 3 メジャー）から外れないよう、メジャー版リリースごとに追従する。

## 8. 実現可否

| 機能 | 実現度 | 備考 |
|---|---|---|
| GitHub Project の閲覧・更新 | ◎ | GraphQL |
| リポジトリ・ブランチ状態確認 | ◎ | git CLI + GitHub API |
| Claude Code への指示 | ◎ | PTY |
| Claude Code の承認 UI | ○ | `PermissionRequest` hook。TUI との競合の扱いはスパイクで検証 |
| セッション差分確認 | ◎ | worktree 単位の diff |
| Copilot への指示 | ○ | Copilot SDK |
| Notion の閲覧・編集 | ○ | REST。公式 MCP は機能が限定的 |
| AI 同士の会話 | ○ | 自前オーケストレータ |
| Notion AI を会話相手にする | △ | Agents API（alpha）またはブラウザ操作。スパイクで検証 |

## 9. リスクと未確認事項

1. Notion Agents API の利用権限（ウェイトリスト、プラン）と認証方式。
2. Notion の webview 自動操作の規約上の扱い。SSO ログインの可否（§4.2）。
3. `claude -p` 利用時の課金扱い。変更の告知と撤回があったとの情報があり、再変更もありうる。公式サポート記事で最新を確認する。
4. Windows:
   - Claude Code の前提（Git Bash / WSL）、インストール形態による起動方法の違い。
   - node-pty のビルド、パス差異。
   - CI で両 OS を最初から回す。
5. Copilot SDK の成熟度（Python 版が Alpha 表記との報告あり。Node 版を使う）。
6. Hooks と transcript の形式は Claude Code のバージョンで変わりうる。パーサーはバージョン検出と縮退動作を持たせる。
7. GitHub App から個人アカウント所有の Projects v2 にアクセスできるか（§2.2）。org の OAuth App アクセス制限の承認手続き。
8. Notion OAuth のトークン交換をサーバなしで行えるか。不可ならトークン交換サーバの運用、または内部インテグレーション方式（§2.2）。内部インテグレーション作成に対するワークスペース管理者の許可。
9. `node:sqlite` の安定度（採用する Electron 版の Node で実験的機能扱いかどうか）と Drizzle の対応。
10. 型付き IPC ライブラリ（tRPC-electron 系）の保守状況とサブスクリプション対応。
11. `PermissionRequest` hook で応答を保留している間に TUI で操作された場合の挙動。
12. 署名用証明書（Apple Developer Program、Windows のコード署名）の調達リードタイム。

## 10. フェーズ

1. **スパイク**: 各項目を最小構成で確認する。合格条件は次のとおり。
   - Claude（PTY + hooks）:
     - `PermissionRequest` hook でアプリから承認・拒否できる。
     - TUI との競合時の挙動が分かる。
     - Finder 起動相当の環境で `claude` を解決できる。
   - Notion AI:
     - webview: 実際に使うログイン方式でログインし、応答を取得できる。
     - Agents API: 利用権限を得られるか確認する。
     - どちらも不可なら §4.2 の撤退基準を適用する。
   - Copilot SDK の疎通。
   - 認証:
     - GitHub Device Flow と org 承認の手続き。
     - Notion 内部インテグレーションの作成可否。
   - Windows: node-pty + ConPTY で `claude` を起動できる。
2. **MVP**:
   - Electron の骨組み（Main + Backend + Pty Host、セキュリティ既定値、署名付き CI ビルド）。署名用証明書はスパイク中に調達を始める。
   - GitHub 認証。
   - リポジトリ/ブランチ一覧、worktree。
   - Claude セッション、差分ビュー、承認 UI。
3. **連携**: GitHub Projects、Notion DB/ページ、セッションと Issue のひも付け。
4. **会話オーケストレーション**: Claude / Notion AI / Copilot の会話、黒板、上限と承認ゲート。
5. **並列化**: 同時セッション数の解放、リソース制御。

## 11. 開発環境

- devcontainer（`.devcontainer/`）は型チェック、lint、単体テスト、ビルド用。Electron の GUI 実行、macOS / Windows 向けパッケージング、署名、Notion webview の確認はホストまたは CI で行う。
