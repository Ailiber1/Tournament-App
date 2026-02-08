# トーナメント管理アプリ 仕様書

> **バージョン**: v2.1
> **構成**: 単一HTMLファイル (`index.html`) + 音声ファイル3点
> **外部ライブラリ**: なし（Vanilla JS + Firebase のみ）

---

## 1. 概要

リアルタイム対戦トーナメントの作成・運営・参加を1つのWebページで完結するアプリ。
Firebase Firestoreによるリアルタイム同期で、管理者と参加者が同時に操作できる。

### デザインテーマ
- **マトリックス風HUD** — 黒背景に緑のネオン発光
- フォント: `Orbitron`（見出し）+ `Noto Sans JP`（本文）
- 落下するカタカナ文字のアニメーション背景

---

## 2. 技術スタック

| 項目 | 詳細 |
|------|------|
| フロントエンド | HTML / CSS / Vanilla JavaScript（単一ファイル） |
| データベース | Firebase Firestore (v9.22.0 compat SDK) |
| フォント | Google Fonts — Orbitron (400,700,900), Noto Sans JP (400,700) |
| 音声 | HTML5 Audio + Web Audio API（合成フォールバック） |
| ホスティング | GitHub Pages |
| ビルドツール | 不要（ブラウザで直接実行） |

### Firebase SDK
```html
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore-compat.js"></script>
```

---

## 3. CSSデザインシステム

### カラーパレット
```css
:root {
  --hud-green: #00ff41;                        /* メイン緑 */
  --hud-green-dim: #00aa2a;                    /* 暗い緑 */
  --hud-green-glow: rgba(0, 255, 65, 0.6);    /* 緑グロー */
  --hud-gold: #ffcc00;                         /* ゴールド */
  --hud-red: #ff3366;                          /* 赤（警告・削除） */
  --bg-dark: #000a04;                          /* 背景色 */
  --bg-panel: rgba(0, 20, 10, 0.85);          /* パネル背景 */
  --text-primary: #c0ffc0;                     /* テキスト */
  --text-dim: #4a8a4a;                         /* 薄いテキスト */
}
```

### レスポンシブ
| ブレイクポイント | レイアウト |
|-----------------|-----------|
| < 900px (スマホ) | 1カラム、縦スタック |
| ≥ 900px (PC) | Grid 2カラム（ルーレット左 + 参加者リスト右） |

### スクロール制御（スマホ対策）
```css
html { overflow: hidden; height: 100%; }
body { overflow: hidden; height: 100%; overscroll-behavior: none; }
.main-container { height: 100%; overflow-y: auto; overflow-x: hidden;
  -webkit-overflow-scrolling: touch; overscroll-behavior: contain; }
```

### hover制限（タッチデバイス対策）
すべての `transform: scale()` ホバーは `@media (hover: hover)` で囲み、
タッチ端末での意図しないレイアウトジャンプを防止。

### 主要アニメーション
| 名前 | 用途 | 内容 |
|------|------|------|
| `matrix-fall` | 背景 | カタカナ落下 (translateY 0→220vh) |
| `vs-pulse` | VS文字 | scale 1→1.1 (1.5s) |
| `lottery-pulse` | 入場ボタン | box-shadow明滅 (2s) |
| `champion-glow` | 優勝者 | 金色box-shadow明滅 (2s) |
| `gameSelectSlide` | ゲーム選択 | opacity+translateY (0.2s) |

---

## 4. 画面構成とナビゲーション

### 画面一覧
```
titleScreen（タイトル）
├── entryScreen（エントリー登録）
├── tournamentScreen（大会画面）
│   ├── lotteryEntrance（入場ボタン → 締切後に解放）
│   ├── lotteryArena（抽選会場）
│   │   ├── lottery-main（ルーレット + ゲーム選択）
│   │   ├── bracketContainer（トーナメント表）
│   │   └── lottery-sidebar（参加者リスト）
│   └── preArenaList（締切前の参加者一覧）
├── historyScreen（大会履歴）
├── adminScreen（管理者登録）
└── adminSetupScreen（大会設定 + ゲーム登録）
```

### モーダル一覧
| ID | 目的 |
|----|------|
| `userGuideModal` | ユーザーガイド表示 |
| `adminGuideModal` | 管理者ガイド表示 |
| `confirmCancelEntryModal` | エントリー解除確認 |
| `cancelCompleteModal` | 解除完了通知 |
| `settingsCompleteModal` | 設定保存完了 |
| `entryCompleteModal` | エントリー完了＋ID表示 |
| `seedLotteryModal` | シード抽選確認 |
| `confirmWinnerModal` | 勝者確定確認 |
| `endTournamentModal` | 大会終了確認 |
| `resetTournamentModal` | 大会リセット確認 |
| `calendarModal` | 日時選択カレンダー |

---

## 5. Firestoreデータ構造

### `tournaments/current`（現在の大会）
```javascript
{
  admin_name: "管理者名",
  admin_device: "デバイスID",
  game_name: "ゲーム名",
  deadline: Timestamp,
  updated_at: Timestamp,
  roulette_state: {
    spinning: boolean,
    rotation: number,
    player1_id: number,
    player2_id: number,
    seed_lottery: boolean,
    seed_player_id: number,
    timestamp: Timestamp
  },
  groups_assigned: boolean,
  group_phase: "group" | "finals" | "merged",
  active_group: "A" | "B" | ... | "finals",
  group_winners: { "A": { username, entry_id }, ... }
}
```

### `tournaments/games_registry`（ゲーム登録）
```javascript
{
  [gameId]: {
    name: "ゲーム名",
    url: "https://...",
    admin_device: "デバイスID",
    created_at: "ISO文字列"
  }
}
```

### `entries/{device_id}`（参加者）
```javascript
{
  entry_id: number,       // 表示用ID（1, 2, 3...）
  username: "ユーザー名",
  device_id: "デバイスID",
  icon_url: "base64..." | null,
  status: "ACTIVE" | "LOST",
  has_seed: boolean,
  group: "A" | null,
  created_at: Timestamp
}
```

### `matches/{auto_id}`（対戦記録）
```javascript
{
  player1_id: number,
  player1_name: "名前",
  player2_id: number,
  player2_name: "名前",
  winner_id: number | null,
  round: number,
  group: "A" | null,
  created_at: Timestamp
}
```

### `history/{auto_id}`（大会履歴）
```javascript
{
  date: Timestamp,
  game_name: "ゲーム名",
  champion_name: "優勝者名",
  admin_name: "管理者名",
  total_participants: number,
  participants: [{ entry_id, username }]
}
```

### `cooldowns/{device_id}`（レート制限）
```javascript
{
  device_id: "デバイスID",
  type: "user" | "admin",
  reason: "操作名",
  expiry: Timestamp,
  created_at: Timestamp
}
```

### `action_logs/{auto_id}`（操作ログ）
```javascript
{
  device_id: "デバイスID",
  type: "user" | "admin",
  action: "操作名",
  timestamp: Timestamp
}
```

### `logs/{auto_id}`（HUDログ）
```javascript
{
  time: "HH:MM:SS",
  message: "ログメッセージ",
  timestamp: Timestamp
}
```

---

## 6. JavaScript状態管理

### グローバル appState
```javascript
appState = {
  deviceId: string,          // localStorage永続化
  isAdmin: boolean,
  adminName: string | null,  // localStorage永続化
  tournament: Object | null, // Firestore同期
  entries: Array,            // Firestore同期
  matches: Array,            // Firestore同期
  games: Array,              // Firestore同期
  selectedPlayer1: Object | null,
  selectedPlayer2: Object | null,
  isSpinning: boolean,
  pendingWinner: number | null,
  matchingIds: Array,
  activeGroup: string,
  groupPhase: string
}
```

### 音声関連
```javascript
bgmEnabled = true      // BGMオン/オフ
seEnabled = true        // SEオン/オフ
bgmVolume = 0.3         // BGM音量 (0-1)
seVolume = 0.5           // SE音量 (0-1)
currentBgm = 'normal'   // 'normal' | 'battle'
```

---

## 7. 主要機能の実装詳細

### 7.1 ルーレット抽選
1. `spinRoulette()` — アクティブ参加者からランダムに2名選出
2. 回転: `1800 + random * 720` 度（5〜7回転）
3. アニメーション: 4秒 (`cubic-bezier(0.17, 0.67, 0.12, 0.99)`)
4. 結果: VS表示でカード表示 → 勝者クリックで確定
5. Firestoreの `roulette_state` で全端末に同期

### 7.2 トーナメント表
- **スロット方式**: 参加者数から `2^n` のブラケットサイズを自動計算
- **ラウンド数**: `Math.ceil(Math.log2(participants))`
- **BYE処理**: 2^nに満たない場合は空きスロット
- **SVG描画**: 勝者ラインを緑発光で表示
- **レスポンシブ**: 画面サイズに応じてボックスサイズ自動調整

### 7.3 グループフェーズ（22名以上）
```
参加者22名以上 → 自動グループ分け（最大20名/グループ）
  ↓
各グループで個別トーナメント → グループ勝者決定
  ↓
全グループ完了 → 決勝戦タブ出現
  ↓
グループ勝者同士で最終トーナメント → 優勝者決定
```
- グループタブ: `G-A (5/20)` 形式表示、完了マーク ✓
- 一括マッチメイク機能（管理者用）

### 7.4 ゲーム選択システム
- `lottery-main-top` 内に配置（PC: ルーレット右横、スマホ: ルーレット下）
- クリックで開閉するドロップダウン（`position: absolute` オーバーレイ）
- 大会のゲーム名と一致するゲームのみ選択可（PLAYバッジ付き）
- 非一致ゲームは「対象外」表示で選択不可
- 隠し `<a target="_blank">` 経由で新タブオープン
- `max-height: 200px` + `overflow-y: auto` でスクロール対応

### 7.5 レート制限（スパム対策）
```javascript
COOLDOWN_TIME = 5 * 60 * 1000      // クールダウン: 5分
COOLDOWN_THRESHOLD = 3             // 閾値: 3回
COOLDOWN_WINDOW = 5 * 60 * 1000    // 監視ウィンドウ: 5分
```
**適用対象:**
- エントリー登録 / 解除（userタイプ）
- ゲーム登録 / 削除（adminタイプ）

**フロー:** 操作実行 → `checkCooldown()` → `recordAction()` → 5分3回でロック

### 7.6 CSV出力
```csv
大会日,ゲーム名,管理者,優勝者,参加者数
"2024/2/8","スマブラ","管理者名","優勝者名",32

ID,参加者名
ID-01,"ユーザー名1"
ID-02,"ユーザー名2"
```
- BOM付きUTF-8
- ファイル名: `YYYY_ゲーム名_参加者名簿.csv`

---

## 8. 音声システム

### ファイル
| ファイル名 | 用途 | タイミング |
|-----------|------|-----------|
| `tournament.mp3` | 通常BGM | タイトル画面 |
| `Battle Match.mp3` | バトルBGM | 抽選会場入場時 |
| `VS用.mp3` | VS効果音 | 対戦カード表示時 |

### SE合成（Web Audio API）
| 関数名 | 音 | 用途 |
|--------|-----|------|
| `playClick()` | 800Hz正弦波 0.1s | ボタンクリック |
| `playHover()` | 600Hz正弦波 0.04s | ホバー |
| `playSpinSound()` | 200→800Hz 3.5s | ルーレット回転 |
| `playVsSound()` | 80Hz + 800Hz | VS表示 |

---

## 9. マトリックス背景

```javascript
function createMatrixBg() {
  // 35カラム生成
  // 文字: アイウエオ...ワヲン + 0-9 + A-F
  // 各カラム: 45文字
  // left: i * 2.9 + random (%)
  // duration: 8〜20秒（ランダム）
  // delay: 0〜-15秒（ランダム、開始位置をずらす）
  // opacity: 0.12（非常に薄い）
  // 落下距離: 220vh
}
```

---

## 10. 大会のライフサイクル

```
1. 管理者登録
   └─ 管理者名入力 → localStorage + Firestore保存

2. 大会設定
   └─ ゲーム名 + 締切日時設定 → tournaments/current に保存

3. エントリー受付（締切前）
   └─ ユーザー名 + アイコン(任意) → entries/{deviceId} に保存

4. 締切到達
   └─ 「抽選会場へ」ボタン解放 → BGM切り替え

5. 抽選・対戦（締切後）
   ├─ ルーレット回転 → 2名マッチング
   ├─ 勝者選択 → matches に記録、敗者status='LOST'
   ├─ 22名以上 → グループフェーズ → 決勝戦
   └─ 最終1名 → 優勝者決定

6. 大会終了
   ├─ history に記録保存
   ├─ CSV自動ダウンロード
   ├─ 「タイトルに戻る」ボタン表示（黄色発光）
   └─ tournaments/current 削除 → 全端末リセット
```

---

## 11. HTMLレイアウト構造（簡略）

```html
<body>
  <div class="matrix-bg">...</div>         <!-- 背景アニメーション -->
  <div class="main-container">              <!-- スクロール対象 -->
    <header class="header">...</header>     <!-- ヘッダー（戻る/タイトル/ガイド） -->
    <div class="screen" id="titleScreen">   <!-- タイトル画面 -->
    <div class="screen" id="adminScreen">   <!-- 管理者登録 -->
    <div class="screen" id="adminSetupScreen"> <!-- 大会設定 -->
    <div class="screen" id="entryScreen">   <!-- エントリー -->
    <div class="screen" id="tournamentScreen"> <!-- 大会メイン -->
      ├─ info-display                        <!-- 大会情報4カラム -->
      ├─ lotteryEntrance                     <!-- 入場ボタン -->
      └─ lotteryArena                        <!-- 抽選会場 -->
          ├─ lottery-main                    <!-- 左: ルーレット -->
          │   ├─ lottery-main-top            <!-- 横並び(PC) -->
          │   │   ├─ roulette-container      <!-- ルーレット本体 -->
          │   │   └─ gameDropdownArea        <!-- ゲーム選択 -->
          │   └─ roulette-controls           <!-- 操作ボタン -->
          ├─ bracketContainer                <!-- トーナメント表 -->
          └─ lottery-sidebar                 <!-- 右: 参加者リスト -->
    <div class="screen" id="historyScreen"> <!-- 履歴 -->
  </div>
  <!-- モーダル群 -->
  <!-- 音声要素 -->
  <script>...</script>                       <!-- 全JavaScript -->
</body>
```

---

## 12. PC/スマホ レイアウト対応表

| 要素 | PC (≥900px) | スマホ (<900px) |
|------|------------|---------------|
| タイトル画面 | 3カラム (メニュー/中央/情報) | 縦スタック |
| 抽選会場 | Grid: `1fr 350px` | Block: 縦スタック |
| ルーレット | 520px | 280px |
| ルーレット+ゲーム選択 | `flex-row`（横並び） | `flex-column`（縦） |
| 操作ボタン | `width:520px` ルーレット幅中央 | 画面幅中央 |
| トーナメント表 | `min-height:60vh` | 自然な高さ |
| VSアイコン | 90px | 60px |

---

## 13. ゲーム登録上限・制約

| 制約 | 値 |
|------|-----|
| ゲーム登録上限 | 20件 |
| 参加者上限 | 100名 |
| ユーザー名最大長 | 20文字 |
| レート制限(操作回数) | 5分以内に3回 |
| クールダウン時間 | 5分 |

---

## 14. ブラウザAPI使用一覧

| API | 用途 |
|-----|------|
| `localStorage` | デバイスID、管理者名の永続化 |
| `Web Audio API` | SE合成、BGMフォールバック |
| `File API` | アイコン画像アップロード |
| `Blob / URL.createObjectURL` | CSVダウンロード |
| `AudioContext` | 音声再生・合成 |

---

## 15. 再現に必要なファイル

```
Tournament-App/
├── index.html            # アプリ本体（HTML + CSS + JS 全部入り）
├── tournament.mp3        # 通常BGM
├── Battle Match.mp3      # バトルBGM
└── VS用.mp3              # VS効果音
```

### Firebase設定（別途必要）
1. Firebase Consoleでプロジェクト作成
2. Firestoreデータベース作成（テストモードでOK）
3. ウェブアプリ登録 → `firebaseConfig` をindex.html内に設定
4. GitHub Pagesにデプロイ
