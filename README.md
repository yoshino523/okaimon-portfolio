# おかいもん（okaimon）

家族向けの買い物リスト共有PWAアプリ。Google認証でログインし、グループ内でリアルタイムにリストを共有・編集できる。

## 背景・目的

- もともと家族で買い物リスト共有アプリを使っていたが、UIが使いづらく課金催促も増えた。それなら自分で作ろう、と開発を開始
- 家族間で「あれ買っておいて」の伝達が口頭やLINEだと漏れる → 共有リストで解決
- 既存アプリでは欲しい機能がなく、使い勝手も悪い → 買い物に特化したUIを自前で構築
- PWAにすることでスマホ・PC両対応、インストール不要、オフライン対応、プッシュ通知も実現
- 運用コスト0円（Firebase Blazeプランの無料枠内）
- 実際に家族に日常的に使ってもらいながら改善を続けており、ユーザーフィードバックを取り入れた開発サイクルを回している

## 技術スタック

| レイヤー | 技術 | バージョン | 選定理由 |
|---------|------|-----------|---------|
| UI | React | 19.x | コンポーネント指向、hooks |
| 型 | TypeScript | 5.9 | 型安全、リファクタ容易 |
| UIライブラリ | MUI (Material UI) | 7.x | マテリアルデザイン準拠のコンポーネント群 |
| CSS-in-JS | Emotion | 11.x | MUIのスタイルエンジン |
| ルーティング | React Router | 7.x | SPA内画面遷移 |
| ビルド | Vite | 8.x | 高速HMR、PWAプラグイン |
| PWA | vite-plugin-pwa | 1.2 | Service Worker自動生成 |
| D&D | @dnd-kit | - | リスト並び替え |
| 認証 | Firebase Authentication | - | Google OAuth |
| DB | Cloud Firestore | - | リアルタイム同期NoSQL |
| ホスティング | Firebase Hosting | - | CDN + 自動デプロイ |
| サーバーレス | Cloud Functions (Node.js 22) | - | 通知・リマインダー処理 |
| タスクキュー | Cloud Tasks | - | リマインダー実行スケジューリング |
| 通知 | Web Push API | - | プッシュ通知 |
| CI/CD | GitHub Actions | - | 自動ビルド・デプロイ |

## システム構成

```
┌─────────────────────────────────────────────────┐
│              クライアント（PWA）                    │
│                                                 │
│  React + TypeScript + MUI                       │
│  Service Worker（オフライン対応）                   │
│  Web Push API（プッシュ通知受信）                   │
└─────────┬───────────────────────────┬───────────┘
          │ Firestore SDK             │ FCM
          │ (リアルタイム同期)          │
          ▼                           ▼
┌─────────────────────┐    ┌─────────────────────┐
│   Cloud Firestore    │    │  Cloud Functions     │
│                     │    │  (Node.js 22)        │
│ ・ユーザー情報       │    │                     │
│ ・グループ          │    │ ・onItemCreate       │
│ ・リスト            │    │   → まとめ通知        │
│ ・アイテム          │    │ ・onReminderSet      │
│ ・ユーザー設定       │    │   → Cloud Tasks登録  │
│ ・デバイストークン    │    │ ・onReminderFire     │
│                     │    │   → Push通知送信     │
│ Firestoreルール      │    │ ・onBudgetAlert      │
│ (Custom Claims認証)  │    │   → コスト監視       │
└─────────────────────┘    └──────────┬──────────┘
                                      │
                                      ▼
                           ┌─────────────────────┐
                           │    Cloud Tasks       │
                           │  リマインダースケジュール │
                           └─────────────────────┘

┌─────────────────────────────────────────────────┐
│              CI/CD（GitHub Actions）               │
│                                                 │
│  develop push → STG自動デプロイ（okaimon-stg）     │
│  main push    → PROD自動デプロイ（okaimon-99daf）  │
│  PR (develop→main) → version-check（自動検証）     │
└─────────────────────────────────────────────────┘
```

## データモデル（Firestore）

```
allowedUsers/{email}                    # 招待制ホワイトリスト
users/{uid}                             # ユーザー基本情報
  ├── settings/preferences              # 設定（テーマ・通知等）
  ├── settings/wallpaperData            # カスタム壁紙（Base64）
  ├── devices/{deviceId}                # プッシュ通知トークン
  ├── itemStats/{statId}                # アイテム利用統計（サジェスト用）
  └── lists/{listId}                    # 個人リスト
      └── items/{itemId}

groups/{groupId}                        # グループ情報
  └── lists/{listId}                    # 共有リスト
      └── items/{itemId}
```

### 認証・セキュリティ設計

- **Custom Claims方式**: Firebase AuthのJWTに `allowed` フラグと `groupId` を埋め込み
- **Firestoreルール**: Custom Claimsで認証 + `allowedUsers` コレクションでフォールバック
- **招待制**: `allowedUsers/{email}` に登録されたメールアドレスのみログイン可能
- **グループ制御**: `groups/{groupId}.members` でリスト・アイテムの読み書きを制限

## 主要機能と実装のポイント

### リアルタイム同期

- Firestoreの `onSnapshot()` でリスナー設定 → 家族の誰かがアイテムを追加すると全端末に即反映
- `forceOwnership: true` でIndexedDBロック残り対策
- Firestore Persistenceでオフライン時もローカルで操作可能、復帰時に自動同期

### スワイプ操作（2段階削除）

- 1回目スワイプ: アイテムが灰色化（チェック済み状態）
- 2回目スワイプ: 削除確定
- 誤削除防止のUX設計。`useSwipeToReveal` カスタムhookで実装

### ドラッグ&ドロップ並び替え

- @dnd-kit で実装
- **浮動小数点ソート方式**: 並び替え時に前後のアイテムの中間値を `sortOrder` に設定 → 1アイテムの書き込みだけで済む（N件の連番振り直しが不要）

### プッシュ通知

- **まとめ通知**: アイテム追加から5分間分をまとめて1通知（Cloud Functionsで制御）
- **リマインダー**: Cloud Tasksでスケジューリング → 指定時刻にCloud Functionsが発火 → Web Push送信
- **不可時間帯**: ユーザーごとに通知しない時間帯を設定可能

### カスタマイズ

- テーマカラー選択
- 壁紙プリセット + カスタム画像アップロード（トリミング対応）
- カテゴリ表示順序のカスタマイズ
- テキスト視認性確保（text-shadow + text-stroke）

### アイテムサジェスト

- `itemStats` サブコレクションにアイテムの使用回数を記録
- 入力時に過去の使用頻度順でサジェスト表示

### PINロック（生体認証）

- Web Credentials APIを使った生体認証ロック
- `signInWithPopup` を維持（`signInWithRedirect` はPINとの競合で使用不可）

## ディレクトリ構成

```
okaimon/
├── src/
│   ├── App.tsx                     # ルーティング定義
│   ├── main.tsx                    # エントリーポイント
│   ├── theme.ts                    # MUIテーマ定義
│   ├── sw.ts                       # Service Worker
│   │
│   ├── lib/firebase.ts             # Firebase SDK初期化
│   ├── constants/                  # 定数（カテゴリ、壁紙等）
│   ├── types/                      # 型定義（User, Group, List, Item等）
│   │
│   ├── hooks/                      # カスタムhooks
│   │   ├── useSwipeToReveal.ts     # スワイプ削除
│   │   ├── useTabSwipe.ts          # タブ横スワイプ
│   │   └── useSwipeActive.tsx      # スワイプ状態Context
│   │
│   ├── components/                 # 共通コンポーネント
│   │   ├── AppLayout.tsx           # レイアウト（BottomNav + Drawer）
│   │   ├── SwipeableListCard.tsx   # スワイプ可能カード
│   │   └── SwipeableListItem.tsx   # スワイプ可能アイテム
│   │
│   └── features/                   # 機能別モジュール
│       ├── auth/                   # 認証（AuthProvider, Login, LockScreen）
│       ├── group/                  # グループ（Provider, Setup, Info）
│       ├── home/                   # ホーム画面（リスト一覧）
│       ├── list/                   # リスト詳細（アイテム操作）
│       ├── settings/               # 設定（テーマ, 壁紙, 通知, セキュリティ）
│       └── profile/                # プロフィール
│
├── functions/src/                  # Cloud Functions
│   ├── onItemCreate.ts             # アイテム追加 → まとめ通知
│   ├── onReminderSet.ts            # リマインダー → Cloud Tasks登録
│   ├── onReminderFire.ts           # リマインダー実行 → Push送信
│   ├── onBudgetAlert.ts            # コスト監視アラート
│   └── lib/webpush.ts              # Web Push APIラッパー
│
├── docs/
│   ├── architecture/               # 設計書（11ドキュメント）
│   ├── specs/                      # 機能仕様書（27件）
│   ├── RELEASE_NOTES.md
│   └── HELP.md                     # ユーザー向けガイド
│
├── .github/workflows/              # CI/CD
│   ├── deploy.yml                  # 自動デプロイ（STG/PROD）
│   └── version-check.yml           # PRマージ前バージョンチェック
│
├── firestore.rules                 # Firestoreセキュリティルール
├── firebase.json                   # Firebase設定
├── package.json
├── tsconfig.json
└── vite.config.ts
```

## 開発プロセス

### 仕様書駆動開発

全ての機能をチケット番号付き仕様書（Markdown）で管理。累計27件。

- **フロー**: 方針承認 → 仕様書作成・確認 → 実装 → ステータス更新
- Phase 1（基盤構築）〜 Phase 14（コード精査・リファクタリング）まで段階的に進行
- 設計書も11ドキュメント整備（システム構成図、画面遷移図、認証フロー、データモデル等）

### CI/CD

- `develop` へのpush → STG環境に自動デプロイ
- `main` へのpush → PROD環境に自動デプロイ（Cloud Functions、Firestoreルール含む）
- PR作成時に `version-check.yml` で package.json のバージョンとリリースノートを自動検証

### Git運用

- `main` ← `develop` ← `feat/チケット番号` の3層ブランチ構成
- PRベースのマージ
- pre-pushフックで `tsc -b`（型チェック）を自動実行

### バージョン管理

- 現在 v1.4.1（2026-03-22時点）
- RELEASE_NOTES.md で全バージョンの変更履歴を管理

## 技術的チャレンジと解決策

### 1. 通知のまとめ送信

買い物リストは短時間に複数アイテムを連続追加する使い方が多い → アイテムごとに通知すると大量通知になる。

- **解決**: Cloud Functionsのトリガーで5分間の「まとめ待ち」を実装。最初のアイテム追加から5分後に、その間の追加分をまとめて1通知で送信

### 2. リマインダーの実行基盤

「明日の朝8時にリマインダー」を実現するには、指定時刻にサーバー側で処理を起動する必要がある。

- **解決**: Cloud Tasksでスケジュール登録 → 指定時刻にCloud Functionsを呼び出し → Web Push APIで通知送信。Firebase Blazeプランの無料枠内で実現

### 3. ドラッグ並び替えの書き込みコスト

Firestoreは書き込み回数で課金 → 並び替えでN件全部書き直すとコスト増。

- **解決**: 浮動小数点ソート方式を採用。並び替え時は移動したアイテムの `sortOrder` を前後の中間値に設定するだけ → 1 write で完了

### 4. オフライン対応とリアルタイム同期の両立

PWAとしてオフラインでも動作しつつ、オンライン時はリアルタイム同期したい。

- **解決**: Firestore Persistenceを有効化。オフライン時はローカルキャッシュで操作、復帰時にFirestoreが自動で同期

### 5. 運用コスト0円の維持

家族向けアプリなのでコストをかけたくない。

- **解決**: Firebase Blazeプランの無料枠内に収まる設計。Cloud Tasksの無料枠活用、まとめ通知でFunction呼び出し回数削減、`onBudgetAlert` でコスト監視

### 6. 壁紙設定時のテキスト視認性

ユーザーが自由に壁紙を設定できる → 明るい壁紙でも暗い壁紙でもテキストが読めるようにする必要がある。

- **解決**: text-shadow（ぼかし影）+ text-stroke（縁取り）の組み合わせで、どんな背景色でも視認性を確保

## 規模感

- **フロントエンド**: React + TypeScript 約50ファイル
- **Cloud Functions**: 約10ファイル
- **仕様書**: 累計27件
- **設計書**: 11ドキュメント
- **開発期間**: 2025年〜継続中（個人開発）
- **ユーザー**: 家族（少人数）
- **バージョン**: v1.4.1（14フェーズの機能追加を経て）
- **リポジトリ**: Private
