# モジュールID の YAML 生成＆インポート（実証済みレシピ）

a-blog cms のモジュールIDは「コンフィグ > モジュールID」の **エクスポート／インポート**で YAML をやり取りできる。
**スキルが YAML を生成 → 人がインポートするだけ**、という運用が実用的に成立する（実機検証済み）。本書はその生成方法・仕組み・落とし穴を、お知らせ以外（商品・サービス・実績など）や別モジュールにも転用できる形でまとめる。

> 大原則は不変：**インポート操作（管理画面）は必ず人が行う**。Claude は YAML を生成するだけ。DB は直接さわらない。

---

## 1. ファイル構造（4セクション）

エクスポートYAMLは DB の生レコードを並べた形。最小単位は次の4キー：

```yaml
config:        # 設定キー = 1行。モジュールの表示設定の実体（config テーブル行）
    - { config_key: entry_summary_limit, config_value: '5', config_sort: 1, config_rule_id: null, config_module_id: 1, config_set_id: null, config_blog_id: 1 }
    - ...
meta:          # 数値ID → コード の辞書（環境間の橋渡し）
    cid: { 1: news }
module:        # モジュール本体 = 1行（module テーブル行）。複数並べてよい
    - { module_id: 1, module_identifier: news_top, module_name: Entry_Summary, ... module_cid: '1', module_cid_axis: descendant-or-self, module_eid_scope: local, module_page_scope: local, module_blog_id: 1 }
field: {  }    # モジュールのカスタムフィールド（無ければ空）
```

### ★ 複数モジュールは「1ファイルにまとめて」生成する（原則・モジュールごとに分割しない）

`config:` と `module:` は**配列**なので、関連する複数モジュール（例 `news_top`/`news_index`/`news_body`）を**1ファイルに全部入れる**（ブログ全体エクスポートと同じ形）。

- **理由**：人のインポートが**1回で済む**（3ファイルなら3回アップロード＝手間＆入れ忘れの元）。`module_id` は仮番号で重複しないよう連番（1,2,3…）にし、各 `config_*` 行の `config_module_id` をそれぞれの `module_id` に合わせるだけ。`meta` は全モジュール分を1つにマージ。
- ファイル名は用途が分かる1つに（例：`module-configs/<theme>-modules.yaml`／`news-modules.yaml`）。
- **やってはいけない**：`news_top.yaml` `news_index.yaml` `news_body.yaml` のように**1モジュール1ファイルで分割**して渡す（インポート回数が増え、順序ミス・入れ忘れを招く）。

> 例：3モジュールを1枚にした最小形は `config`（各モジュール分を連結）＋ `meta`（cid 等をマージ）＋ `module`（3行）＋ `field: {}`。

---

## 2. インポート時の自動再マップ（＝環境非依存の核心）

インポータ（`php/Services/Config/Import.php`）は値を賢く変換する。だから**他環境で作った YAML がそのまま通る**：

| YAML の値 | インポート時の扱い |
|---|---|
| `*_blog_id`（config/module/field） | **取り込み先の現在ブログIDへ自動置換**。`1` のままでよい |
| `module_id`（と `config_module_id`/`field_mid`） | **新しい連番を採番し直し、参照も自動追従**。数値は“紐づけ用の仮番号”でよい |
| `module_cid` / `module_eid` 等の対象ID | `meta` 経由で **コードから取り込み先の実IDを引き直す**（下記） |
| 同じ `module_identifier` | **ドロップしてから再挿入＝上書き／冪等**（他のモジュールは消さない） |

### meta によるコード解決（重要）
`module_cid: '1'` のような数値は、`meta.cid: { 1: news }` の対応表で **コード `news`** に変換され、取り込み先で `category_code='news'` のカテゴリーを引いて**実 cid** に置き換わる。

- ⇒ **`module_cid` は仮番号でよい。ただし `meta.cid:{仮番号: 実カテゴリーコード}` を必ず付ける。**
- 検証実績：YAML が `cid:'999'`＋`meta{999:news}`、取り込み先の news 実cidが `13`（元は3から変更済み）でも、**コード経由で正しく 13 に解決**された。数値は完全に無視される。
- コードが取り込み先に存在しないと、その値は null になり当該絞り込みが外れる（致命エラーにはならない）。→ **カテゴリーは事前に人が作っておく**（SETUP の②）。

同様に `meta` には `uid/eid/aid/bid` も入りうる。詳細系（Entry_Body）は `module_cid` 空＋`module_eid_scope: global` で、URL の eid から記事解決するので cid の meta は不要。

---

## 3. config セクションの作り方（変数駆動 ＋ 性能最適）

`config` 行はモジュール種別ごとに **config_key の集合が決まっている**。全集合は対象環境の
`<docroot>/themes/system/admin/config/{ns}/{sub}_body.html`（例 `entry/summary_body.html`, `entry/body_body.html`）の `name="..."` で確認できる（実機エクスポートを1枚もらえれば最速）。

### 生成ルール（これが中核）
テンプレートに**変数/ブロックを置くたびに「それは config スイッチで制御される機能か？」を判定**し、次の3分岐で YAML に落とす：

1. **使う機能 → ON を明記**（既定OFFのものは必須）
2. **使わない & 既定ON → OFF を明記**（＝無駄な DB アクセスを止める。性能最適）
3. **使わない & 既定OFF → 省略**（OFF が継承される。ファイルも短く＝ミスも減る）

> a-blog cms は**多くの機能スイッチが既定 ON**。放置すると未使用機能のクエリが走るので、2 を必ず実施する。

### 既定 ON の主な機能スイッチ（未使用なら明示 OFF 対象）
- **Entry_Summary**：`entry_summary_entry_field` / `_fulltext` / `_tag` / `_image_on` / `_category_on`
- **Entry_Body**：`entry_body_tag_on` / `_summary_on` / `_comment_on` / `_pager_on` / `_serial_navi_on` / `_category_info_on` / `_entry_field_on` / `_image_on`

（`user_on`/`blog_on`/`user_info_on`/`blog_info_on` などは既定 OFF → 通常は省略でよい）

### 変数 → config スイッチ 対応表

**Entry_Summary（一覧系）**
| テンプレに置く変数/ブロック | config_key | 備考 |
|---|---|---|
| `{date#…}` | `entry_summary_base_date` | 日付を使うなら ON |
| `categoryField`→`{fieldCategoryName}` | **`entry_summary_category_on`** | 既定ON。`category_field_on` ではない |
| `{summary}` | `entry_summary_fulltext` | |
| `{path}` 画像 | `entry_summary_image_on`（＋`_main_image_target`） | |
| `tag:loop`→`{name}` | `entry_summary_tag` | |
| `pager:veil` | `entry_summary_pager_on` | 一覧ページングは module 行の `module_page_scope: global` も必要 |
| `entryField`/`{エントリーフィールド}` | `entry_summary_entry_field` | 使わないなら OFF（既定ON） |
| `{title}` `{url}` | —（常時） | スイッチ無し |

**Entry_Body（詳細系）**
| 変数/ブロック | config_key | 備考 |
|---|---|---|
| `{date#…}` | `entry_body_date_on` | |
| `categoryField`→`{fieldCategoryName}` | **`entry_body_category_info_on`** | 既定ON。`category_field_on` ではない |
| `tag:loop` | `entry_body_tag_on` | |
| `unit:veil`（本文ユニット） | —（常時） | スイッチ無し |
| `{summary}` | `entry_body_summary_on` | |
| 前後ナビ（serial） | `entry_body_serial_navi_on` | 使わないなら OFF（既定ON） |
| コメント | `entry_body_comment_on` | 同上 |
| エントリーページャー/マイクロページ | `entry_body_pager_on` | 同上 |
| `{エントリーフィールド}` | `entry_body_entry_field_on` | 同上 |
| メイン画像 `{path}` | `entry_body_image_on` | 本文画像をユニットで出すなら主画像は不要→OFF |

> 別モジュール（Category_EntrySummary, Entry_Headline 等）も考え方は同じ。config_key の接頭辞が変わるだけ＝その `*_body.html` を見て同じ判定をする。

---

## 4. module 行の“ノブ”（案件ごとに変える所）

| カラム | 役割 | 値の例 |
|---|---|---|
| `module_identifier` | **テンプレの `id="…"` と完全一致**（最重要） | `news_top` / `product_index` … |
| `module_name` | モジュール種別 | `Entry_Summary` / `Entry_Body` … |
| `module_label` | 管理用ラベル | 任意 |
| `module_cid` ＋ `meta.cid` | 絞り込みカテゴリー（仮番号＋コード） | `'1'` ＋ `{1: news}`（**cid_scope=local のときだけ必要**） |
| `module_cid_scope` | カテゴリーの決め方 | **`global`＝URLコンテキストで決定（cid不要）** / `local`＝`module_cid` で固定 |
| `module_cid_axis` | 下層カテゴリーを含むか | `descendant-or-self`（含む）/ `self`（自分のみ） |
| `module_eid_scope` | 詳細でURLの記事を解決 | 詳細＝`global`、一覧＝`local` |
| `module_page_scope` | 一覧のページURLコンテキスト | 一覧＝`global`、その他＝`local` |
| `module_status` | 公開 | `open` |
| `module_blog_id` | 自動置換されるので | `1` のままでよい |

### ★ cid は「固定」か「URLコンテキスト任せ」か（重要な設計判断）

ビューがURLのどこに置かれるかで、cid の決め方が変わる：

| ビュー | 置かれる文脈 | 推奨 | 理由 |
|---|---|---|---|
| **一覧（index）** | カテゴリーURL（`/service/` 等） | **`module_cid_scope: global`（cid 明示しない）** | URLのカテゴリー文脈から自動で絞り込む。**カテゴリー未確定でも設定でき、複数カテゴリーで同じ一覧テンプレを使い回せる**。`meta.cid` 不要 |
| **詳細（body）** | エントリーURL（`/service/entry-12.html`） | **`module_eid_scope: global` / cid 空** | URLの eid で記事決定。同上でカテゴリー非依存 |
| **トップ抜粋（top）** | サイトトップ `/`（**カテゴリー文脈なし**） | **`module_cid_scope: local` ＋ `module_cid` 明示（＋`meta.cid`）** | URLから決められないので**固定が必須**。→ そのカテゴリーは**存在必須** |

> ⇒ 「カテゴリーが無くて cid 空」で詰むのは **トップ抜粋など“URL文脈の無い”モジュールだけ**。一覧/詳細は **global スコープでURL任せ**にすれば cid を書かずに済む（実機で service_body が詳細URLで正常表示することを確認済み）。
> ⇒ 一方、**トップ抜粋の cid は固定必須**なので、その種別の格納カテゴリーは先に作っておく（着手前の前提確認 #2）。

---

## 5. 落とし穴チェックリスト（生成時）

- [ ] `on`/`off`/`yes`/`no` は **必ずクォート**（`'on'`）。素のままだと YAML が真偽値に変換し、文字列カラムに入って壊れる
- [ ] `module_identifier` がテンプレの `id="…"` と1文字違わず一致
- [ ] `module_cid` を使うなら `meta.cid:{同じ番号: コード}` を付けたか（付け忘れると絞り込みが外れる）
- [ ] 1ファイル複数モジュールのとき、`module_id` と各行の `config_module_id` の対応が崩れていないか（仮番号でよいが**ファイル内で一貫**）
- [ ] 使う機能=ON、未使用の既定ON機能=OFF、を変数対応表どおりに設定したか
- [ ] 詳細系は `module_cid` 空・`module_eid_scope: global`／一覧系のページングは `module_page_scope: global`＋`pager_on`
- [ ] 保存先はテーマ外（公開ディレクトリの外）。例：プロジェクト直下 `module-configs/`
- [ ] **インポートは人に依頼**（手順を SETUP に明記）。生成後 `ruby -ryaml` 等でパース確認しておくと安全

---

## 6. アンカーの取り方（確実にやるなら）

バージョンでキー集合や既定値が変わりうる。確実を期すなら：

1. 対象環境で**該当モジュールを1つ管理画面で作り、エクスポート**してもらう（＝その版の正解）。
2. それをテンプレにして、本書の「変数駆動 ON/OFF」と「module 行ノブ」を編集 → 複数モジュールへ展開。

エクスポートが得られない場合は `themes/system/admin/config/.../*_body.html` の `name=` 一覧で config_key を、`config.system.default.yaml` で既定値を確認して組む。
