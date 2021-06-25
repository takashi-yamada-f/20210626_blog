# SNS・ブログで学ぶシステム設計・概要

どうやったら、SNSを開発できるのかというご質問があったので、今日はそのあたりのお話をしたいと思います。

## 【１】既存システムの解析



## 【2】ブログのデータ構造を考えてみよう。

### (1)どんなテーブルが必要？
主なテーブルはこんな所？各テーブルがIDを持って、結びつきます。
* ユーザー情報　users
* ブログ blogs
* 記事 entries
* コメント comments
* 画像 images
* ブックマーク bookmarks
* ハッシュタグ hashtags
* カテゴリ categories
<br>など

### (2)必要なサーバは？

* スタートアップはこんな感じ?<br>
```
[WEBサーバ] - [DBサーバー]
```

* サービスが軌道に乗れば、こんな感じ?<br>
```
[ロードバランサー冗長化構成] - [WEBアプリケーションサーバ × X台] - [CACHEサーバ × X台] - [DBサーバー × X台]
　　　　　　　　　　　　　　　　　　　　　　　　　          　　- [画像・静的コンテンツサーバ × X台]
```
フロントにロードバランサーがあって、複数のWEBサーバーがあり、<br>
レスポンスを高速化する為の、RADISやMemCacheのようなキャッシュサーバがあって、バックエンドにMySQL等のDBサーバがあります。<br>
これとは別に静的なファイルを返す画像サーバ、もしくはCDN（コンテンツ・デリバリー・ネットワーク）があるがあるといった構成です。<br>

### (3)ソーシャル機能の実現
ブログでもソーシャル機能が実装されています。
* update ping（XMLRPC）による更新情報共有
* 検索エンジンの活用
* バッチ処理による、友人更新情報の共有
* API等による、システムを横断した情報取得
<br>（非同期通信のAPIなどで構築すれば、よりSNS的な造りになります。）

### (4)大規模化の問題
サーバーを増やしてみても、ざっと、以下のような問題があります。
* 時間と共に増大する記事データ。DBテーブルが肥大化し続けると、性能が劣化し、数年も経てばサービスが破綻します。
* 画像が増えれば、iノード数の上限に達し、参照も遅くなります。
* コストの問題。これは最後に。

## 【3】DB分割でDBテーブルが肥大化回避

### (1)DB分割とは
オンラインゲームをよくやる方は、「サーバー」を選択させられる場合がありますよね？<br>
あれは、「サーバー（１システム）」あたりの利用者（アクセス、データ）を制限して、パフォーマンスを下げない工夫です。<br>
<br>
ブログでも、DBを「DBテーブル群」に分割することで、DBテーブルが肥大化を回避することで、パフォーマンスを維持することができます。<br>
各DBテーブル群内で、ブログ固有のデータが完結するイメージです。<br>
```
DBテーブル群(A)
 ブログ blogs_00
 記事 entries_00
 コメント comments_00
 画像 images_00
 ブックマーク bookmarks_00

DBテーブル群(B)
 ブログ blogs_01
 記事 entries_01
...
```

### (2)DB分割しようと思ったときのオートインクリメントIDの問題
テーブルのIDは普通、DBの自動採番(オートインクリメント)になっているので、単純にテーブルを分割するとそれぞれのテーブルで、重複したIDが発生してしまいます。<br>
オートインクリメントに依存しないID採番の方法が必要になります。<br>
<br>
あくまで一例ですが、ブログIDをDBに依存したオートインクリメント値ではなく、ランダム１６進数にます。32桁以上であれば、数億回発行しても被ることはありません。<br>

[phpの例]
```
$ブログID = md5(uniqid(rand(), true));// ランダムに16進数32桁のブログIDを生成
例
719deecdbfec1e13b4ed4e755ed39a47
bde6ea4cb4bfcb8a995a81c0dc0ced78
cf2a0969f094946174dcf2fb8a8a0821
247ae1d5444ce5cee6ada532310fa0e3
```
末尾二桁をとると00〜ff、10進数だと0〜256になります。<br>
```
例
$DBテーブル群番号 = hexdec(ブログIDの末尾二桁)%DB分割数;// ブログIDの末尾二桁を10進数に戻し、分割数で割った余り

下記のようなテーブル名になる。
blogs_DBテーブル群番号
entries_DBテーブル群番号
comments_DBテーブル群番号
images_DBテーブル群番号
bookmarks_DBテーブル群番号
```
この方法を使えば最大256個までテーブル群を分割することができます。<br>
データが増えるに従って、適宜、2分割、4分割（２分割したモノを更に2分割、以下同様）、8分割、、、、２５６分割を行っていきます。　<br>
<br>
「md5(uniqid(rand(), true))」　のIDは便利！<br> 
 
## 【4】iノードの上限値

### (1)静的なファイルの作成数には上限があります。
iノードには上限値があります。平たく言うと、上記の表題のとおり。

inode (アイノード) を枯渇させてみる | CUBE SUGAR CONTAINER<br>
https://blog.amedama.jp/entry/2018/04/21/133823

### (2)iノードの上限を回避

回避するには、画像IDを元にランダムなディレクトリを計算して、画像を配置するとよいです。<br>
```
例
user_image/7d/f2/8edf5384da0cfffb58310c603a0923ac.jpg
```

## 【5】コストの問題

* ブログ　サービス終了 | google
https://www.google.com/search?q=%E3%83%96%E3%83%AD%E3%82%B0+%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E7%B5%82%E4%BA%86&rlz=1C5CHFA_enJP719JP719&oq=%E3%83%96%E3%83%AD%E3%82%B0%E3%80%80%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9&aqs=chrome.2.69i57j0l5j69i61.15395j0j4&sourceid=chrome&ie=UTF-8

* Twitter黒字転換　1～3月、利用者数は予想未達<br>
https://www.nikkei.com/article/DGXZQOGN3004T0Q1A430C2000000/

なぜ、ブログ、SNSは収益化が難しいのか？考えてみましょう！
