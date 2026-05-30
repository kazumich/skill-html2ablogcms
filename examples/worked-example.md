# 通し例：Tailwind 飲食店サンプルのお知らせ動的化

> ## ⛔ この中の HTML をそのまま別案件へコピペしない
> 以下のスニペットの `class="…"`・タグ構成（`<ul>`/`<article>`/Tailwind ユーティリティ等）は、**この特定の入力HTML（飲食店サンプル）にしか属さない**。別サイトに貼ると確実に崩れる。
>
> **転用できるのは“置換の型”だけ**：①繰り返しの最小単位を1つ残して `entry:loop` で囲む ②**literal 値だけ**を変数（`{title}` `{url}` `{date#…}` 等）に差し替える ③**囲っているマークアップ・クラスは“変換対象HTML自身のもの”をそのまま残す** ④区分→`categoryField`、タグ→`tag:loop`、本文→ユニット。
>
> 正典は SKILL.md 手順3（変換アルゴリズム）。**本ファイルは「型を当てるとどう見えるか」の1サンプル**にすぎない。「コピペできない」ことは**末尾の「別マークアップに同じ型を当てた対照例」**で実証している（必ず目を通すこと）。

対象は a-blog cms 標準チュートリアル相当の構成（トップ抜粋 / 一覧 / 詳細の3ビューが揃ったケース）。**`class` はすべて入力HTML由来。あなたの案件のクラスに読み替えること。**

---

## ① トップの最新お知らせ抜粋 → `Entry_Summary` (`news_top`)

**静的（複数 `<li>` が並ぶ）→ 1件を loop 化。`border-t` 等のクラスは保持。**

```html
<!-- BEGIN_MODULE Entry_Summary id="news_top" -->
<ul class="border-t border-gray-200">
  <!-- BEGIN notFound -->
  <li class="p-4 text-gray-500">お知らせはまだありません。</li>
  <!-- END notFound -->
  <!-- BEGIN entry:loop -->
  <li>
    <a href="{url}" class="flex items-center gap-x-4 p-4 border-b border-gray-200 hover:bg-gray-50">
      <time datetime="{date#Y}-{date#m}-{date#d}" class="w-28 flex-shrink-0 text-sm text-gray-500">{date#Y}年{date#m}月{date#d}日</time>
      <!-- BEGIN categoryField:veil --><!-- BEGIN categoryField -->
      <span class="flex-shrink-0 rounded-full bg-sky-100 px-2.5 py-0.5 text-xs font-semibold text-sky-800">{fieldCategoryName}</span>
      <!-- END categoryField --><!-- END categoryField:veil -->
      <span class="font-medium text-gray-800 truncate">{title}</span>
    </a>
  </li>
  <!-- END entry:loop -->
</ul>
<!-- END_MODULE Entry_Summary -->
```

※ バッジ色（`bg-sky-100` など、元HTMLでは区分ごとに違った）は CSS の話。CMS では区分を**カテゴリー**に正規化し、色分けが要るなら category のコード名を使った CSS で対応する（テンプレ側で出し分けるなら `categoryField` 内で条件分岐）。「お知らせ一覧へ」リンクは静的のまま `/news/` を指す。

---

## ② お知らせ一覧ページ → `Entry_Summary` (`news_index`) ＋ ページャー

**サムネ・概要・タグ付きの記事ブロックを loop 化。「もっと見る」をページャーに置換。**

```html
<!-- BEGIN_MODULE Entry_Summary id="news_index" -->
<ul class="space-y-8">
  <!-- BEGIN notFound -->
  <li class="text-gray-500">お知らせはまだありません。</li>
  <!-- END notFound -->
  <!-- BEGIN entry:loop -->
  <li>
    <a href="{url}" class="group block p-6 sm:p-8 rounded-lg hover:bg-gray-50">
      <article class="grid md:grid-cols-3 gap-8 items-start">
        <!-- BEGIN_IF [{path}/nem] -->
        <div class="md:col-span-1 overflow-hidden rounded-lg">
          <img src="%{ROOT_DIR}{path}[resizeImg(600)]" alt="{alt}" class="w-full h-56 object-cover shadow-md">
        </div>
        <!-- END_IF -->
        <div class="md:col-span-2">
          <p class="text-sm text-gray-500"><time datetime="{date#Y}-{date#m}-{date#d}">{date#Y}年{date#n}月{date#j}日</time></p>
          <h2 class="mt-2 text-xl font-bold text-gray-900 group-hover:text-sky-600">{title}</h2>
          <!-- BEGIN summary:veil -->
          <p class="mt-3 text-gray-600">{summary}[trim(120,'…')]</p>
          <!-- END summary:veil -->
          <div class="mt-4 flex flex-wrap items-center gap-x-4 gap-y-2">
            <!-- BEGIN categoryField:veil --><!-- BEGIN categoryField -->
            <span class="bg-sky-100 text-sky-800 text-xs font-semibold px-2.5 py-0.5 rounded-full">{fieldCategoryName}</span>
            <!-- END categoryField --><!-- END categoryField:veil -->
            <!-- BEGIN tag:loop -->
            <span class="text-sm text-gray-500">#{name}</span>
            <!-- END tag:loop -->
          </div>
        </div>
      </article>
    </a>
  </li>
  <!-- END entry:loop -->
</ul>

<!-- BEGIN pager:veil -->
<div class="mt-16 text-center">
  <!-- BEGIN forwardLink -->
  <a href="{url}" class="inline-flex items-center rounded-md bg-sky-600 px-6 py-3 text-base font-medium text-white hover:bg-sky-700">もっと見る</a>
  <!-- END forwardLink -->
</div>
<!-- END pager:veil -->
<!-- END_MODULE Entry_Summary -->
```

番号ページャー（1・2・3…）にする場合は SKILL.md 手順4の `page:loop` / `link#front` / `link#rear` 構成を使う。

---

## ③ お知らせ詳細ページ → `Entry_Body` (`news_body`)

**`<article>` を `entry:loop` で包み、本文は静的マークアップを捨ててユニット表示に置換。** 静的HTMLにあった `<p>…</p>` や `<h2>` の本文は「サンプル記事の中身」であり、テンプレには残さない（人がユニットとして入力する）。

```html
<!-- BEGIN_MODULE Entry_Body id="news_body" -->
<!-- BEGIN notFound -->
<p>お探しのお知らせは見つかりませんでした。</p>
<!-- END notFound -->
<!-- BEGIN entry:loop -->
<!-- BEGIN_MODULE Touch_Login -->@include("/admin/entry/action.html")<!-- END_MODULE Touch_Login -->
<article>
  <p class="text-base font-semibold text-sky-600 text-center">
    <time datetime="{date#Y}-{date#m}-{date#d}">{date#Y}年{date#m}月{date#d}日</time>
  </p>
  <h1 class="mt-2 text-3xl font-bold tracking-tight text-gray-900 sm:text-4xl text-center">{title}</h1>
  <div class="mt-6 flex flex-wrap items-center justify-center gap-x-4 gap-y-2">
    <!-- BEGIN categoryField:veil --><!-- BEGIN categoryField -->
    <a href="{fieldCategoryUrl}" class="bg-sky-100 text-sky-800 text-sm font-semibold px-3 py-1 rounded-full">{fieldCategoryName}</a>
    <!-- END categoryField --><!-- END categoryField:veil -->
    <!-- BEGIN tag:loop -->
    <a href="%{ROOT_CATEGORY_URL}tag/{name}" class="text-sm text-gray-500 hover:text-gray-700">#{name}</a>
    <!-- END tag:loop -->
  </div>
  <!-- 本文：静的の prose 群は捨て、ユニット描画に置換。prose ラッパーだけ残す -->
  <!-- BEGIN unit:veil -->
  <div class="mt-12 prose prose-lg max-w-none prose-sky">
    @include("/include/unit.html")
  </div>
  <!-- END unit:veil -->
</article>
<!-- END entry:loop -->
<!-- END_MODULE Entry_Body -->
```

- `include/unit.html` は `@extends("/_layouts/unit.html")` でよい（beginner から流用）。
- パンくずを動的にするなら `Topicpath` モジュール（変数は `themes/system/acms-code/vars/Topicpath.html` 参照）。最小構成なら静的のままでも可。
- カスタムフィールドを足すなら `admin/entry/field.html`、SEO欄は `admin/entry/field-seo.html` を beginner からコピー。

---

## このサンプルでのデータモデリング結果（手順2の適用例）

| 静的HTMLの要素 | 判定 | a-blog cms 概念 |
|---|---|---|
| `<time datetime="2025-08-07">` | 投稿日 | 投稿日 `{date#…}` |
| バッジ「新商品/お知らせ/メディア/イベント」（固定・排他） | 種類分け・絞り込み | **カテゴリー** `{fieldCategoryName}` |
| `#季節限定 #夏メニュー`（自由・複数） | 付箋的キーワード | **タグ** `{name}` |
| サムネイル画像（記事1枚） | 代表画像 | **メインイメージ** `{path}` |
| 詳細の `<p>/<h2>/<ul>/<blockquote>` 本文 | 本文 | **ユニット**（人が入力） |
| リンク先 `20250807.html` 等 | URL | `{url}`（自動採番。静的パスは捨てる） |

この表が「手順2の振り分けルールを実際のHTMLに当てた結果」。別のHTMLなら、同じルールで違う結果になる（例：区分が無いサイトならカテゴリー無しでよい、日付がタイトル内テキストだけなら投稿日として入力してもらう、等）。

---

## 別マークアップに同じ型を当てた対照例（＝コピペできない証明）

上の Tailwind 例を**そのまま貼ったら別サイトでは崩れる**。下は **`<dl>` ベース・独自クラス・フレームワーク無し**のお知らせ一覧（よくある別パターン）に、**同じ置換の型**を当てたもの。**変数（`{url}`/`{date#…}`/`categoryField`/`{title}`）は同一なのに、囲うタグ・クラスは入力HTML由来でまったく別物**になっている点に注目。

**入力（静的・3件並ぶ）**
```html
<dl class="info-list">
  <dt class="info-list__date">2025.08.07</dt>
  <dd class="info-list__body"><span class="cat cat--new">新商品</span>
    <a href="/news/20250807.html">新しい季節限定メニューが登場しました</a></dd>
  <dt class="info-list__date">2025.08.01</dt>
  <dd class="info-list__body">…</dd>
</dl>
```

**変換後（最小単位＝1組の dt+dd を loop 化。`info-list` 等のクラスは“入力のもの”を保持）**
```html
<!-- BEGIN_MODULE Entry_Summary id="news_top" -->
<dl class="info-list">
  <!-- BEGIN notFound --><dt class="info-list__date">—</dt><dd class="info-list__body">お知らせはまだありません。</dd><!-- END notFound -->
  <!-- BEGIN entry:loop -->
  <dt class="info-list__date">{date#Y}.{date#m}.{date#d}</dt>
  <dd class="info-list__body">
    <!-- BEGIN categoryField:veil --><!-- BEGIN categoryField --><span class="cat">{fieldCategoryName}</span><!-- END categoryField --><!-- END categoryField:veil -->
    <a href="{url}">{title}</a>
  </dd>
  <!-- END entry:loop -->
</dl>
<!-- END_MODULE Entry_Summary -->
```

→ 変数とブロック（`entry:loop` / `categoryField` / `notFound`）は Tailwind 例と**完全に同じ**。違うのは `<ul><li>` か `<dl><dt><dd>` か、`flex items-center gap-x-4 …` か `info-list__date` か、という**入力HTML側の見た目だけ**。
**だから「型（変数とブロック）を覚え、マークアップは“目の前のHTML”から取る」。Tailwind 例の `class` を別案件へ持ち込まない。**
