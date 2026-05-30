# html2ablogcms

静的HTMLサイトの **「お知らせ / News / 新着情報」など、繰り返し構造を持つコンテンツ**だけを [a-blog cms](https://www.a-blogcms.jp/) の動的コンテンツに変換するための、Claude Code 用スキルです。サイト全体のテーマ化ではなく「**まず1本、更新情報をきちんと動的化する**」ことに集中します。

News に限らず、Service / 商品 / 実績など**カテゴリーで束ねる繰り返しコンテンツ全般**に同じ手順で適用できます。

## このスキルがやること

- 静的HTMLから「お知らせ相当」の繰り返しコンテンツを特定し、**存在するビューだけ**を動的化
  - トップの抜粋（最新数件）／一覧（ページング）／詳細 → `Entry_Summary` / `Entry_Body`
- 変換は「繰り返しの literal 値を変数に置換、囲うマークアップは元のまま保持」
- 管理画面で人が行う作業（テーマ適用・カテゴリー作成・モジュールID登録・記事投入）を、**順序付きの手順書 `SETUP.md`** として生成
- 必要なら**モジュールID設定の YAML を生成**（1ファイルに集約・環境非依存）。インポートは人が行う

## 重要な原則（守ること）

- **DB（`acms_*`）を SQL で直接書き換えない／記事を SQL でシードしない。** a-blog cms は管理画面での設定を前提に作られている。
- テーマ適用・カテゴリー作成・モジュールID設定・エントリー投入は**人が管理画面で行う**。スキルは手順書を渡すだけ。
- 着手前に前提（特に**格納カテゴリー**）を確認し、未確定のまま実装に入らない。

## 前提環境

- **a-blog cms が beginner テーマでインストール済み**のローカル開発環境。
- **Claude Code が、その a-blog cms と同じローカル環境で動く**こと（テーマファイルを読み書きでき、できれば稼働中のローカルサイトがある）。スキルは以下を利用します：
  - `<docroot>/themes/system/...` のリファレンス（変数・config_key の裏取り）を**ファイルとして読む**
  - 変換したテーマファイルを直接生成・編集する
  - ローカルサイトを開いて表示を確認する
- **フォールバック**：ファイルにアクセスできない場合は変数名・config_key を[公式ドキュメント](https://developer.a-blogcms.jp/document/)で代替し、稼働サイトが無い場合はライブ検証を省略してタグ整合の静的チェック＋`SETUP.md` の受け渡しに留めます（動作確認は人に依頼）。

## インストール

`~/.claude/skills/html2ablogcms/` に置きます。

```bash
git clone https://github.com/kazumich/skill-html2ablogcms.git ~/.claude/skills/html2ablogcms
```

または、リポジトリを別の場所に置いて symlink を張る（`.git` をスキルディレクトリに入れたくない場合）：

```bash
git clone https://github.com/kazumich/skill-html2ablogcms.git ~/github/skill-html2ablogcms
ln -s ~/github/skill-html2ablogcms ~/.claude/skills/html2ablogcms
```

## 使い方

Claude Code で、対象のローカル a-blog cms 環境を開いて依頼するだけ：

> themes/sample の「お知らせ」部分を a-blog cms で動的化して

スキルが着手前に前提（格納カテゴリー・作るビュー・テーマ名等）を確認し、テンプレート一式と `SETUP.md` を生成します。最後に `SETUP.md` の手順を**人が管理画面で**実施して完了です。

## 構成

```
html2ablogcms/
├── SKILL.md                       # 判断手順 + a-blog cms の作法 + SETUP生成ルール
├── examples/
│   └── worked-example.md          # 変換の通し例（※写経用ではなく「型」を示す一例）
└── references/
    ├── developer-docs.md          # a-blog cms 公式ドキュメントのリンク集
    └── module-id-yaml.md          # モジュールID設定YAMLの生成＆インポート
```

## クレジット

`examples/worked-example.md` のサンプルHTMLは、a-blog cms 公式チュートリアル「静的サイトからの移行」相当の構成（飲食店サンプル）を題材にしています。

## ライセンス

（LICENSE ファイル参照）
