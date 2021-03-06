## 序章
### :books:概要
- 集約のイベントストリーム
- イベントソーシングを使って、集約の状態を管理したり、永続化させたりする手法のことをA+ES(Aggregates and Event Sourcing)と、呼ぶことにする

#### メリット
- 追記（Insert）のみとなるので、パフォーマンスが非常に良好になるかつ、データのレプリケーションも様々な選択肢がある
- イベントのログを残すこと自体に、大きなメリットがあること
- イベント中心のアプローチの場合、オブジェクトリレーショナルマッピングとのインピーダンスミスマッチを回避できるので、システムはより堅牢になり、変更に強いものになる
    - なんで回避できるんだろう？

#### デメリット
- イベントストリームのクエリを実装する時点で、CQRSがなんらかの形で併用することが求められてくる為、より学習が難しい

### :question:疑問点


### :memo:書記欄




## A.1 アプリケーションサービスの内部構造

### :books:概要

- アプリケーションサービスからイベントストアを操作する場合の流れを以下の図で示されている

1. クライアントが、アプリケーションサービス上のメソッドを実行する
2. ビジネス操作を実行するために必要なドメインサービスがあれば、それを取得する
3. クライアントが提供する集約インスタンスの識別子を使って、その集約用のイベントストリームを取得する
4. ストリームから取得したすべてのイベントを順に適用して、集約のインスタンスを復元する
5. 集約が提供するビジネス操作を実行し、インターフェースの契約で求められているすべてのパラメータを渡す
6. 集約はダブルディスパッチを使って、渡されたドメインサービスやその他の集約に処理を渡すかもしれない。其後、操作の結果を表す新しいイベントを作る
7. ビジネス操作が失敗しなかったと仮定して、新しく作られたすべてのイベントをストリームに追加する。その際に、ストリームのバージョンを確認して、平行性の衝突を回避する
8. 新しく追加されたイベントをイベントストアから取り出して、何らかのメッセージングインフラストラクチャを用いてサブスクライバに発行する

![](https://i.imgur.com/bT3vf2x.png)

これを、C#のサンプルコードで表すと、以下のようになる。 `LockForAccountOverdraft` メソッド内。
- [GitHub内のコード](https://github.com/abdullin/iddd-sample/blob/e617636c29a9fdec82c56156a952f50299346430/Sample/Domain/CustomerAggregate/CustomerApplicationService.cs#L69)

![](https://i.imgur.com/LqNLsIP.png)

- イベントストアからイベントを読み込むときには、集約の一意な識別子を使う

- ソースコードを見る限り、`Customer` のコンストラクタは、eventから受け取って構築することを前提にしているようだが、初期生成をすることはない？
    - Customer.Createの中で、CustomerCreatedイベントを発行しながら、初期化していた


- 集約にイベントを適用する動きは、Mutate()メソッドを使用する

![](https://i.imgur.com/llO026C.png)


- 新しく追加されたイベントを、プロセッサが検出して、イベントを購読している他システムにPublishする

![](https://i.imgur.com/tUrWdI3.png)

- [GitHubコード](https://github.com/abdullin/iddd-sample/blob/8072d39155a94982526f26d15856a15dbe40304d/Sample/Domain/CustomerAggregate/CustomerState.cs#L63)の動きがよくわかっていない。
    - P.526の説明だと、.NETの仕組みを用いて、指定したイベントの型に対応するWhen()メソッドを探すとのことらしい。
    
### :question:疑問点


- P.528の「状態の変更が完了したら、Changesコレクションをイベントストアにコミットしなければいけない。新しい変更をすべて追加したので、他の書き込みスレッドとの並行処理の衝突は発生しない」
    - これは本当に？　今まで再生してきたイベントストリームにアプリケーションサービス内で追加して、イベントストアにコミットしているように見える。そうだとしたら、同時に他の書き込みスレッドがイベントストリームを読み込んだ後に、コミットしたら、競合が発生しないか？


- [GitHubコード](https://github.com/abdullin/iddd-sample/blob/e617636c29a9fdec82c56156a952f50299346430/Sample/Domain/CustomerAggregate/CustomerApplicationService.cs#L32)での、イベントが発生したときの処理（Whenメソッド）を、アプリケーションサービス上で、定義している理由がよくわかっていない。
    - CustomerApplicationServiceが、Domain->CustomerAggregateのパッケージ内にいるのが、なんか変？


### :memo:書記欄

- P.528の「状態の変更が完了したら、Changesコレクションをイベントストアにコミットしなければいけない。新しい変更をすべて追加したので、他の書き込みスレッドとの並行処理の衝突は発生しない」
    - これは本当に？　今まで再生してきたイベントストリームにアプリケーションサービス内で追加して、イベントストアにコミットしているように見える。そうだとしたら、同時に他の書き込みスレッドがイベントストリームを読み込んだ後に、コミットしたら、競合が発生しないか？
        - 最初にリードして (step.2.1)、他から影響を受けなくなっている範囲でオブジェクトを再生 (Step 2.2) して、処理結果を書き込んでいる(step 4) 。ので、リードした時点から、文理性が確保されている。ということかしらん。その最初に読んだ状態を前提とすることで、書き込む処理をするところまでの処理の破綻は発生しない。ただし、他のスレッドで並行して同じオブジェクトに処理をしていると競合が発生するので、楽観的排他制御を実施するのかしら?
        - よくあるトランザクションの例で、二つのトランザクションで、後から読んだ方 T2 が先に書いたら、先に読んでしまった方 T1 の書く処理はアボートされる必要がある。
            - T1: Read(x, y, z, ...),  none, none, Write(x, y, z, ...) → アボート
            - T2: none, Read(x, y, z, ...), Write(x, y, z, ...), none → コミット



---

## A.2 コマンドハンドラ

### :books:概要

- A+ESの実装の改良の選択肢として、コマンドハンドラとラムダがその一つである
- アプリケーションサービスのメソッドを、コマンドオブジェクトとしてクラスにする

![](https://i.imgur.com/RViCQ6C.png)

↓

![](https://i.imgur.com/Ilv7M7p.png)
![](https://i.imgur.com/sABCD88.png)

#### メリット


- 負荷分散に有利
    - クライアントをアプリケーションサービスから切り離せる
    - コマンドオブジェクトはシリアライズ化できる
    - シリアライズ化されたコマンドオブジェクトは、メッセージで送信ができる
    - メッセージキューに溜めておくことができるので、アプリケーションサービスがオフラインでも、柔軟に対応が可能

![](https://i.imgur.com/SvIwLem.png)
![](https://i.imgur.com/okrGg9g.png)


### :question:疑問点
- P.534の「Capped Exponential Back-off」形式とは
- この章で書かれていることは、コマンドパターンのことで合っている？



### :memo:書記欄
- この章で書かれていることは、コマンドパターンのことで合っている？
    - 私もCQRSの文脈のコマンドという認識です。GoFのコマンドパターンはもっと別物でめんどうだった気がします

- P.534の「Capped Exponential Back-off」形式とは
    - エクスポネンシャル・バックオフ・アンド・ジッター

　この問題を解決する手法として、「エクスポネンシャル・バックオフ・アンド・ジッター（Exponential backoff and jitter）」というリトライのアルゴリズムがあります。これは、エクスポネンシャル・バックオフとジッターという2つのアルゴリズムを合わせたものです。

　エクスポネンシャル・バックオフは、リクエスト処理が失敗した後のリトライの際、現実的に成功しそうな程度のリトライを、許容可能な範囲で徐々に減らしつつ継続するアルゴリズムです。具体的には、再試行する度に、1秒後、2秒後、4秒後と指数関数的に待ち時間を加えていきます。これにより、全体のリトライ回数を抑え効率的なリトライを実現します。

　ジッターとは、random関数を用いたジッター（ゆらぎ）を導入することを指しています。これによりリクエスト間のタイミングの衝突を回避する効果があります。

　AWS Solution Architectブログの記事「Exponential Backoff And Jitter」にて、詳細な検証結果がまとめられています。本稿では、さらに補足を加えつつ平易に説明を試みたいと思います。
https://codezine.jp/article/detail/10739

- TCPのパケット再送アルゴリズムもそんな感じ。

---

## A.3 ラムダ構文

### :books:概要

- ラムダ式をサポートする言語であれば、イベントを読み込んで、イベントストアに新しく追加するまでの処理を、楽に書くことができる

![](https://i.imgur.com/cvwDHsR.png)

![](https://i.imgur.com/qlWF6zQ.png)

[ラムダ関数を受け取るUpdateメソッド](https://github.com/abdullin/iddd-sample/blob/e617636c29a9fdec82c56156a952f50299346430/Sample/Domain/CustomerAggregate/CustomerApplicationService.cs#L88-L98)
[Updateメソッドにラムダ関数をパラメータにして渡すWhenメソッドたち](https://github.com/abdullin/iddd-sample/blob/e617636c29a9fdec82c56156a952f50299346430/Sample/Domain/CustomerAggregate/CustomerApplicationService.cs#L32-L60)

### :question:疑問点



### :memo:書記欄

---


## A.4 並行性制御

### :books:概要

- 複数のスレッドから、同時にイベントストリームにアクセスがあることを考慮すると、並行処理の衝突が発生する可能性がある

![](https://i.imgur.com/oL3fDz1.png)

- この第4ステップで、衝突が起きた場合は例外を発生させ、再試行を行う方法が以下のコードである

【やっていること】

1. スレッド2が例外をキャッチして、処理に失敗する。whileの先頭に戻り、スレッド1で更新されたイベント1〜5を新たに読み込む
2. スレッド2は、再試行する。このとき、スレッド1が追加したイベント5の後ろに、イベント6、7を追加する

![](https://i.imgur.com/XIzKbv8.png)

- 再試行ではなく、以下の図のような衝突の回避をするパターンもある

![](https://i.imgur.com/EBOc1RM.png)

![](https://i.imgur.com/91l261m.png)

[GitHub上のコード](https://github.com/abdullin/iddd-sample/blob/e617636c29a9fdec82c56156a952f50299346430/Sample/Domain/CustomerAggregate/CustomerApplicationService.cs#L101-L132)

ここの章に関して、かとじゅんさんからの、問題点の指摘。

- A.4 並行性制御のところ、この構成ではEventStoreにイベントを追記する際に強力な一貫性が必要になってますね。そうしないと競合を排除できないからです。複数サーバからこのロックの奪い合いが起こるとスケーラビリティが損なわれる…。悲観ロックでも楽観ロックでも。なので、この方式はスケールしにくい。サーバを増やしてもアムダールの法則でリターンの減少が起きますね。



### :question:疑問点
ｰ 衝突のチェックってものすごく重そう。ライトヘビーになってくると衝突祭りでエラーしか出なくなって、使い物にならなくなりそうな気がするのだけど、どうやって解決するといいんだろう。

### :memo:書記欄

---


## A.5 構造上の束縛から解法されたA+ES

### :books:概要

- イベントストリームは、お好みのシリアライザを使ってバイトブロックに変換したメッセージを、追記限定のリストにまとめたもの
- その性質上、イベントストリームの永続化には、どんな仕組みでも使える
    - 本当か？

### :question:疑問点
- P.539の永続化の仕組みは、どんなものでも使える、というのは本当にそうなんだろうか？　と思っているが、イベントソーシングを使っている人は、どんな仕組みを選択をしている？

### :memo:書記欄

---


## A.6 パフォーマンス

### :books:概要
- 巨大なストリームを読み込もうとするとパフォーマンスの問題が出てくる（ひとつのストリームに数十万件〜数百万件のイベントが入っている場合など）
- 問題に解決するためのいくつかのパターン
    1. イベントストリームをサーバのメモリ内にキャッシュする
    2. イベントのスナップショットを取得する方法

#### スナップショットを使うパターン

スナップショットを作るパターンだと、以下のようなサンプルコードと図

![](https://i.imgur.com/KgJ0oOt.png)
 
![](https://i.imgur.com/x2QiNr6.png)

`ReplayEvent()` を使用して、最新のスナップショットの取得以降のイベントで集約のインスタンスを最新の状態にする必要がある。


### :question:疑問点
- P.539の「イベントストリームをメモリ内にキャッシュする」方法については、最初（？）の読み込みが大変そうだと思った
- スナップショットを保存する先って、やっぱりESと同じ場所(RDBなら同じテーブル) がいいのだろうか？　

### :memo:書記欄

- スナップショットを保存する先って、やっぱりESと同じ場所(RDBなら同じテーブル) がいいのだろうか？　
    - 他の場所というか、他の永続化機構を選ぶのが順当かと。
- P.539の「イベントストリームをメモリ内にキャッシュする」方法については、最初（？）の読み込みが大変そうだと思った
    - 再生→コマンド処理→イベント永続化→しばらくコマンド来ない→集約を破棄
    - また読むときに、再生しなおす。もしくは、チェックポイントで特定の時点のスナップショットを保存しておく。
    - akkaを使うと良い


---


## A.7 イベントストアの実装

### :books:概要


### :question:疑問点


### :memo:書記欄

---


## A.8 リレーショナル・データベースへの永続化

### :books:概要

- テーブル定義
![](https://i.imgur.com/YDNooep.png)
- トランザクション内でイベントを特定のストリームに追加する手順
    - トランザクションを開始する
    - イベントストアのバージョンが、期待するものから変わっていないことを確かめる。変わっていた場合は、例外を投げる
    - 並行処理の衝突がなければ、イベントを追加する
    - トランザクションをコミットする

### :question:疑問点

- イベントストアのバージョンって何に使うんだっけ

### :memo:書記欄
- イベントストリームごとに、バージョンを管理する
    - ロックのためということである。

---


## A.9 BLOBの永続化

### :books:概要

- データベースを使わずに、イベントストアを構築する際（たとえば、BLOBストレージやファイルシステムストレージなど）の設計の指針
    - 追記限定のBLOBファイルあるいはそれと同等のなにかの組み合わせで構成すること
        - ストレージへの書き込みを行うコンポーネントは、追記の際には排他ロックを行うが、並行した読み込みアクセスは許可する
    - ストレージの分け方の選択肢
        - ひとつの境界づけられたコンテキスト内のすべての型の集約のインスタンスを管理する
        - 集約の型ごとに個別のBLOBストアを用意する（それぞれのBLOBストアには、特定の型のインスタンスだけを格納する）
        - 個々の集約型用のBLOBストレージをインスタンスごとに分けて、ひとつのインスタンスのイベントストリームをそのインスタンス用のBLOBに格納できる

![](https://i.imgur.com/vhX5seV.png)


### :question:疑問点
- 設計の指針の②にある「あるいは、集約の型ごとに個別のBLOBストアを用意するという方法もある。それぞれれのBLOBには、特定の型のインスタンスだけを格納する。あるいは、個々の集約型用のBLOBストレージをインスタンスごとに分けて、ひとつのインスタンスのイベントストリームをそのインスタンス用のBLOBに格納できる」
    - この2つの違いがよくわからない。どちらも、集約の型ごとにストアを用意するように見える

### :memo:書記欄
- 集約の型ごと = 集約クラスごと
- インスタンスごと = 集約インスタンスごとでは
- 例えば、イベントストリームからオブジェクト状態の再生が簡単になる (小さくなる)。
 - 発展的な応用として、シャーディング的なことができそう。


---


## A.10 集約に注目する

### :books:概要
- ドメインモデリングを行う際のはじめの一歩として、入ってくるコマンドと出ていくイベント、そして実行する振る舞いから考えていくというのはいい方法だ
    - Event Stormingの話っぽい

### :question:疑問点
- 『経験上、A+ESを使って設計した集約は小さくなる傾向にある。』　とあるが、実際のところどのような力が働いてそうなるのだろうか？
- 本当に集約の再設計が簡単になるのだろうか。イメージがあまりわかない。(感想)

### :memo:書記欄

- そもそもなんで、この付録はA+ESと言っているのかな？

---


## A.11 リードモデルプロジェクション

### :books:概要

- イベントソーシングの場合、プロパティを用いた集約へのクエリをどのように行うかを気にしないといけない
    - 「先月の受注額の合計は？」　というクエリだと、1ヶ月分のイベントを読み込み、そこから合計を計算しないといけない
- それの助けになるのが、リードモデルプロジェクションである
- サブスクライバを使って、イベント通知を受け取り、永続リードモデルの生成と更新を行う
- リードモデルはドキュメントデータベースに永続化するのが一般的だが、それ以外の選択肢も利用が可能
    - [ドキュメントデータベース](https://aws.amazon.com/jp/nosql/document/#:~:text=%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E3%81%AF%E3%80%81%E9%9D%9E%E3%83%AA%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%8A%E3%83%AB,%E3%81%AB%E8%A8%AD%E8%A8%88%E3%81%95%E3%82%8C%E3%81%A6%E3%81%84%E3%81%BE%E3%81%99%E3%80%82&text=%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88%E3%81%8A%E3%82%88%E3%81%B3%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88%E3%83%87%E3%83%BC%E3%82%BF%E3%83%99%E3%83%BC%E3%82%B9%E3%81%AF,%E3%81%95%E3%81%9B%E3%82%8B%E3%81%93%E3%81%A8%E3%81%8C%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99%E3%80%82)

### :question:疑問点
- サブスクライバを使って、イベントをリードモデルに置いておくというのは、CQRSの話に通じるような気がしているっぽい（感想）
- 割と修正等から取り残されてしまいそうだけど、大丈夫なのかな？


### :memo:書記欄

- 「最新の状態」の表現形態はDBレコードかもしれないし、pdfかもしれないし、テキストファイルかもしれない
- 「イベントをプロジェクションする」ことが肝であって、「どんな形でプロジェクションするか」は肝要ではない（アプリケーションの任意）、という

---


## A.12 集約の設計への利用

### :books:概要


### :question:疑問点


### :memo:書記欄

---

## A.13 イベントの拡張

### :books:概要

- イベントを使って集約の永続化を行う一方で、ドメインレベルのさまざまな出来事を伝える際にもイベントを利用している
- 欲しい情報が変わる、通知する相手を変えるときなどで、リードモデルプロジェクションをするときに、様々なイベントの情報を拾わないといけなくなることがある（下図参照）
![](https://i.imgur.com/tkzO0dc.png)
- その場合、イベントの内容を拡張することによって、単純化ができる
![](https://i.imgur.com/j58sOaV.png)
↓　↓　↓
![](https://i.imgur.com/8rDDQgj.png)
![](https://i.imgur.com/YIjCh2P.png)


### :question:疑問点
- 日本語的な質問ですが、「ドメインイベントを作る際の経験則として、「80%のサブスクライバの要求を満たすだけの情報を含めること」がある。多くのサブスクライバにとっては、無駄な情報になってしまうかもしれない。それでも、そうしておくほうがいい。」
    - 残りの20%が、多くのサブスクライバなのだろうか？　また翻訳が微妙に間違っている？
- イベントのフィールドを拡張したときに、既存の発生したイベントに対しても変更を掛けるのか、掛けないのかという話は何も載っていないが、どうするべきだろう。
- だれがどのフィールドを使っているか分からなくなりそう。最終的には、消していいかわからないし似てるけどもいっこ作っちゃえ！ってなりそう(感想)

### :memo:書記欄

- UserNameChangedV2 を作るイメージ？
    - そうですね、V2つくるイメージ

---


## A.14 活用できるツールやパターン

### :books:概要

- イベントシリアライザ
    - [Protocol Buffers](https://developers.google.com/protocol-buffers)
        - [Qiitaでの紹介記事](https://qiita.com/yugui/items/160737021d25d761b353)
    - サンプルコード内では、[DataContractSerializer](https://docs.microsoft.com/ja-jp/dotnet/api/system.runtime.serialization.datacontractserializer?view=netcore-3.1)やJsonSerializerを使用している
- イベントの不変性
    - ドメインイベントの不変性を守るために、フィールドはreadonlyプロパティにして、コンストラクタのみで値を設定できるようにする
- 値オブジェクト
    - 型チェックやIDEのサポートを受けることができる
    - 値オブジェクトをコマンドやイベントで使う場合には、両者を一緒にデプロイするか、共有カーネルを作る必要がある
        - とはいえ、デシリアライズの型安全性のために値オブジェクトを共有カーネルに置くと、値オブジェクト内の複雑なビジネスロジックが共有されてしまうので、脆い設計となってしまう
            - 対策として、2通り
                - コマンドのデシリアライズに使うシンプルな共有クラスと、コアドメインで使用する値オブジェクトを明確に区別する（2種類の値オブジェクトを用意する）
                - 13章で説明されていた、イベント通知を受け取るときに、動的型付けの手法を選ぶ。そうすれば、サブスクライバ側に値オブジェクトの型情報をデプロイする必要がなくなる


### :question:疑問点


### :memo:書記欄

---


## A.15 契約の自動生成

### :books:概要


### :question:疑問点


### :memo:書記欄

---


## A.16 ユニットテストとスペシフィケーション

### :books:概要

- イベントソーシングでのテストの場合、Given-When-Expect形式のテストを定義するのが簡単になる
    - Given: 過去に発生したイベントに対して
    - When: 集約のメソッドが呼び出されたとき
    - Expect: 皇族のイベントあるいは例外のいずれかが発生する


### :question:疑問点


### :memo:書記欄

---

## A.17 関数型言語におけるイベントソーシング

### :books:概要

- 集約の実装をオブジェクト指向→関数型に変更する場合の注意点
    - シンプルかつ、不変な状態レコードと、状態を変更する関数群に切り替える必要がある。
    - 不変な値オブジェクトを作るときの考え方に似ている
        - 値オブジェクトの副作用のない関数は、自身の状態と関数への引数にもとづいて、新しい値をつくる
    - 集約のメソッドは、ステートレスな関数群にも変換できる

### :question:疑問点

- イベントストアは、関数データベースとして扱える
    - 関数データベースとは？
### :memo:書記欄

---


## 輪読会感想ふりかえり用

## 読書会進め方ふりかえり用

## その他、メモとか用の領域


