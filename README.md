# Offers Tech Blog Boilerplate

- [Offers Tech Blog | Zenn](https://zenn.dev/p/overflow_offers)
- [📘 How to use Zenn CLI](https://zenn.dev/zenn/articles/zenn-cli-guide)

## ボイラープレートの使い方

共通設定が含まれるこのリポジトリを fork して個別に記事執筆に利用してください。
既に zenn 執筆用に個人リポジトリを持っている場合は、textlint 周りの設定を取り込んで publication 対象記事に適用をお願いします。

## 記事の作成

新しい記事（article）は `npx zenn new:article` で作成できます。
`--slug` オプションで公開 URL に使われる記事のユニークな文字列を指定できます。ファイル管理の観点と公開時の URL をきれいに保つためにも指定をお願いします。
`--publication-name overflow_offers` オプションで Tech Blog の Publication を指定できます。

```
npx zenn new:article --slug example-article --publication-name overflow_offers

=> articles/example-article.md が作成される
=> https://zenn.dev/offers/articles/example-article で表示される
```

- [Zenn のスラッグ（slug）とは](https://zenn.dev/zenn/articles/what-is-slug)

## 画像の追加

`/images` ディレクトリ内に記事の slug と同名のディレクトリを作成し、その中に記事内で使用したい画像を配置、Markdown ファイルから参照してください。記事ごとにディレクトリでまとめたほうが管理が容易になります。

```
# 例
mkdir images/april-fool-banzai
```

## 記事のプレビュー

```
npx zenn preview

=> http://localhost:8000 でプレビュー可能になる
```

# 執筆ルール

以下のルールに沿って本文を構成してください。

## タイトルと記事冒頭、末尾の規約

### タイトル

記事タイトルの末尾に `｜Offers Tech Blog` を suffix として挿入してください。zenn の仕様上、記事タイトルに 70 文字の制限がある点に注意してください。

```
title: 記事タイトル｜Offers Tech Blog
```

### 冒頭（書き出し）

サービス名と社名の紹介をお願いします。サービス名と社名が含まれていれば文言はカスタマイズいただいて構いません。

```
プロダクト開発人材の副業転職プラットフォーム[Offers](https://offers.jp/) を運営する株式会社 [overflow](https://overflow.co.jp/) で...
[Offers](https://offers.jp/) を運営している株式会社 [overflow](https://overflow.co.jp/) の...
```

### 末尾（関連記事・任意）

記事の末尾に任意で関連記事として、想定読者が好みそうな他の記事や人気記事を並べておくとよろしいです。

```
# 関連記事

https://zenn.dev/offers/articles/20220523-component-design-best-practice
https://zenn.dev/offers/articles/20220425-universal-attitude
https://zenn.dev/offers/articles/20220415-leader-and-manager-roles-in-overflow
```

⚠️ 求人訴求は zenn Publication の「本文下の固定メッセージ設定」にマージされました

## 文章表現などの規約

### 校正は textlint を使用してください

基本的な校正は [textlint](https://github.com/textlint/textlint) で検知される点を対象とします。
Offers Tech Blog では [textlint-rule-preset-overflow-techblog](https://github.com/overflow-tm/textlint-rule-preset-overflow-techblog) をプリセットルールとして使用しています。

```
# lint の実行
npm run lint

# auto fix できるエラーの解決
npm run lint:fix
```

警告されやすい厳密なルールの多くは無効にしています。やむをえず回避しづらいルールがある場合は原稿ファイル内で次のように特定ルールを無効にする宣言コメントを使用してください。

```
# 1つの文中に同じ助詞が連続して出てくるのを防ぐルールの無効化
<!-- textlint-disable ja-technical-writing/no-doubled-joshi -->
```

### 画像には代替テキストを設定してください

textlint で検知できないので各位が執筆時に留意してください。

```
×NG例 [](/images/so-delicious-ramen.jpg)
✓OK例 [めっちゃおいしそうなラーメンの写真](/images/so-delicious-ramen.jpg)
```

### 技術単語や固有名詞は正式名称を使用してください

頻出する単語は [textlint-rule-preset-overflow-techblog/dict/prh.yml](https://github.com/overflow-tm/textlint-rule-preset-overflow-techblog/tree/main/dict) への登録を検討してください。

```
×NG例 Rails、Wordpress、Overflow（社名）
✓OK例 Ruby on Rails、WordPress、overflow
```

その他の略称もなるべく初出で `SPA（Single Page Application）` のように注釈するよう配慮願います。
