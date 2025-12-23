# ブログ記事候補調査 (2025-12-21)

この調査は、直近のGitHubでのコミット・PR活動から、ブログ記事にできそうなトピックをピックアップしたものです。

---

## 調査対象期間
- 2025年12月中旬〜現在

---

## ブログ記事候補 (優先度順)

### A. 高優先度 (記事にしやすい・需要がありそう)

#### 1. Zig でインベーダーゲームを作る
- **リポジトリ**: [til](https://github.com/kokoichi206/til)
- **関連コミット**:
  - `feat: defeat the invaders`
  - `feat: draw invaders`
  - `feat: add bullet`
  - `feat: setup player`
  - `add shields`
  - `add game_over flag`
  - `chore: restart game config`
- **記事の方向性**:
  - Zig言語でのゲーム開発入門
  - Zigの特徴的な機能 (メモリ管理, コンパイル時計算) の実践
  - ゲームループの実装パターン
- **推定難易度**: 中〜高
- **記事想定読者**: Zig に興味がある人、ゲーム開発に興味がある人

---

#### 2. NextAuth への移行 (Supabase Auth → NextAuth)
- **リポジトリ**: [Wareware-PJ/ads-report-pro](https://github.com/Wareware-PJ/ads-report-pro)
- **関連PR**:
  - `Replace Supabase auth with NextAuth and refresh docs/env (Vibe Kanban)`
- **記事の方向性**:
  - Supabase Auth から NextAuth への移行手順
  - NextAuth の設定とセッション管理
  - Google OAuth 連携の実装
- **推定難易度**: 中
- **記事想定読者**: Next.js で認証を実装したい人

---

#### 3. Drizzle ORM 導入と既存スキーマからの移行
- **リポジトリ**: [Wareware-PJ/ads-report-pro](https://github.com/Wareware-PJ/ads-report-pro)
- **関連PR**:
  - `Drizzle ORM 導入と Flask モックからのスキーマ移行`
- **記事の方向性**:
  - Drizzle ORM の基本的な使い方
  - 既存のスキーマ定義からの移行手順
  - Prisma との比較
- **推定難易度**: 中
- **記事想定読者**: TypeScript でのDB操作に興味がある人

---

#### 4. Knip を使った未使用コード検出とCI連携
- **リポジトリ**: [Wareware-PJ/ads-report-pro](https://github.com/Wareware-PJ/ads-report-pro)
- **関連PR**:
  - `Knip 導入と CI チェック追加`
- **記事の方向性**:
  - Knip の導入方法と設定
  - CI (GitHub Actions) での自動チェック設定
  - 実際に検出された未使用コードの例
- **推定難易度**: 低〜中
- **記事想定読者**: TypeScript プロジェクトの品質を上げたい人

---

#### 5. OpenRouter Structured Outputs への移行
- **リポジトリ**: [tsumugi-official/tsumugi](https://github.com/tsumugi-official/tsumugi)
- **関連コミット**:
  - `OpenRouter Structured Outputs への移行`
- **記事の方向性**:
  - OpenRouter の Structured Outputs 機能の紹介
  - 既存の LLM 呼び出しからの移行手順
  - JSON Schema を使った出力制御
- **推定難易度**: 中
- **記事想定読者**: LLM を使ったアプリケーション開発者

---

### B. 中優先度

#### 6. pino ロガーの導入と設定
- **リポジトリ**: [Wareware-PJ/ads-report-pro](https://github.com/Wareware-PJ/ads-report-pro)
- **関連PR**:
  - `Add pino logger to shared utilities`
- **記事の方向性**:
  - Node.js での pino ロガー導入
  - 環境ごとのログレベル設定
  - Next.js との連携

---

#### 7. Chrome 拡張機能: Google 検索結果の Vim ライクナビゲーション
- **リポジトリ**: [chrome-extension-google-search-navigator](https://github.com/kokoichi206/chrome-extension-google-search-navigator)
- **関連コミット**:
  - `feat: support vim like key handling`
  - `feat: filter links to only include <a> tags containing <h3> elements`
  - `fix: prevent interference with modifier key shortcuts in key bindings`
- **記事の方向性**:
  - Chrome 拡張機能の作り方
  - キーボードイベントのハンドリング
  - DOM操作の実践例

---

#### 8. Supabase セッションキャッシュによるパフォーマンス改善
- **リポジトリ**: [tsumugi-official/tsumugi](https://github.com/tsumugi-official/tsumugi)
- **関連コミット**:
  - `Supabaseセッションをキャッシュし getUser の通信を削減`
- **記事の方向性**:
  - Supabase Auth のパフォーマンス問題
  - セッションキャッシュの実装方法

---

#### 9. Renovate で GitHub Actions 更新を1つのPRにグループ化
- **リポジトリ**: 複数 (Shodan-Pro/shodan-pro, Shodan-Pro/scraping)
- **関連PR**:
  - `chore(renovate): GitHub Actions の更新を1つの PR にグループ化`
- **記事の方向性**:
  - Renovate の設定カスタマイズ
  - PRの肥大化防止テクニック

---

### C. 既に記事化済み (参考)

以下のトピックは既にZenn記事として投稿済みです:

- `add: 2025-12-19-terraform-cancel` - Terraform キャンセル関連
- `add: 2025-12-19-gh-actions-concurrency` - GitHub Actions の concurrency 設定
- `add: 2025-12-18-asdf-resham` - asdf 関連
- `add: 2025-12-17-vibe-kanban` - Vibe Kanban 関連

---

## その他の活動 (記事候補としては弱いが参考)

### 技術的な改善
- `workflow での terraform コマンドに -input=false フラグを追加` - Terraform の自動化改善
- `記事一覧ページのフィルター機能を URL クエリパラメータと同期させる` - UX 改善
- `自動投稿前に AI チェッカーを完了させるよう修正` - ワークフロー改善
- `JSONC で trailingComma ありをデフォルトにする` - コードスタイル設定

### インフラ/DevOps
- `Ignore TypeScript and ESLint build errors for demo branches` - CI/CD の柔軟な設定
- `compose・ci の追加` - Docker Compose + CI 構築

---

## おすすめ記事化順序

1. **Zig でインベーダーゲームを作る** - 珍しいテーマで差別化しやすい
2. **Knip を使った未使用コード検出** - 実用的で需要が高そう
3. **NextAuth への移行** - 多くの開発者が直面する課題
4. **Drizzle ORM 導入** - 新しめのORMで情報が少ない
5. **OpenRouter Structured Outputs** - LLM活用の実践例

---

## 調査メモ

- tsumugi-official/tsumugi は本格的なプロダクト開発をしている印象
- Wareware-PJ/ads-report-pro は新規プロジェクトの立ち上げフェーズ (Next.js + Drizzle + NextAuth)
- til リポジトリは学習記録として様々な技術を試している
- 自動化 (Renovate, GitHub Actions) の設定改善を複数のリポジトリで行っている

---

*調査日: 2025-12-21*
*調査者: Claude Code*
