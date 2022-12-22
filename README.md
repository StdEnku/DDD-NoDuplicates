# 重複名なしを実現するドメインモデルの設計例

重複を許さないというビジネスルールを適用するためのいくつかの設計アプローチ。

## 問題点

あなたのドメインモデルにはあるエンティティがあります。<br/>
このエンティティは `name` プロパティを持っています。<br/>
この名前はアプリケーション内で一意であることを保証する必要があります。<br/>
この問題をどのように解決するのでしょうか？<br/>
このリポジトリではビジネスロジックやルールをドメインモデル内に<br/>
保持することを目的としたDDDアプリケーションで<br/>
これを行うための11の異なる方法を示しています。<br/>

## アプローチ

以下はこの問題を解決するためのさまざまなアプローチです。<br/>
それぞれのケースで必要なクラスはフォルダにまとめられています。<br/>
実際のアプリケーションではこれらはいくつかのプロジェクトに分割され<br/>
ドメインモデル、リポジトリ実装、テスト（または実際のUIクライアント）が<br/>
それぞれ別の場所に配置されると想像してください。<br/>

## 方法1.データベース

この最も単純なアプローチは<br/>
ドメインモデルのルールを無視し[^1]<br/>
あなたのためにそれを処理する何らかのインフラに依存することです。<br/>
一般的にこれはSQLデータベースの一意制約の形をとり<br/>
エンティティのテーブルに重複する値を挿入または更新しようとすると例外を発生させます。<br/>
これは機能しますが永続性の変更[^2]やドメインモデル自体のビジネスルールの保持はできません。<br/>
しかしこれも簡単で性能も良いので検討する価値はあります。<br/>

本リポジトリでは実際のデータベースを使用していないので<br/>
`ProductRepository`で動作を偽っています。<br/>

[Approach 1 - Database](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/01_Database)<br/>

[^1]: これは基本的にDDDの観点からは不正行為です
[^2]: 一意制約をサポートしない、またはコストやパフォーマンスが許容できないシステムへ

## 方法2.ドメインサービス

1つの選択肢はドメインサービス内で名前が重複しているかどうかを検出し<br/>
名前の更新を強制的にそのサービスを通して流すことである。<br/>
これはドメインモデル内にロジックを保持するが貧弱なエンティティをもたらす。<br/>
このアプローチではエンティティにはロジックがなく<br/>
エンティティに対するあらゆる作業は<br/>
ドメインサービスを介して行われる必要があることに注意すること。<br/>
この設計の問題点は時間が経つにつれてすべてのロジックをサービスに置き<br/>
サービスが直接エンティティを操作するようになり<br/>
エンティティ内のロジックのカプセル化をすべて排除してしまうことである。<br/>
なぜこのようなことになるのでしょうか？<br/>
ドメインモデルのクライアントは一貫したAPIで作業したいのであって<br/>
ある時はエンティティ上のメソッドである時はサービス経由のメソッドで<br/>
どちらかを使わなければならない理由や<br/>
根拠がないまま作業することは望まないからです。<br/>
そしてエンティティ上で始まったメソッドは依存関係がある場合<br/>
サービスに移動する必要があるかもしれません。<br/>

[Approach 2 - Domain Service](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/02_DomainService)<br/>

## 方法3.エンティティメソッドに必要なデータを渡す

ひとつの方法としてエンティティメソッドに<br/>
チェックを行うために必要なデータをすべて渡すことができます。<br/>
この場合異なる名前であると想定されるすべての名前のリストを渡す必要があります。<br/>もちろんエンティティの現在の名前は含めないでください。<br/>なぜなら当然ながら現在の名前がそのまま使用されるからです。<br/>数百万以上のエンティティがある場合この方法ではうまくスケールしないでしょう。<br/>

[Approach 3 - Pass Necessary Data into Entity Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/03_PassDataToMethod)<br/>

## 方法4.エンティティメソッドに一意性チェックのためのサービスを渡す

このオプションではメソッドインジェクションによって依存性注入を行い<br/>必要な依存性をエンティティに渡します。<br/>残念ながらこれは呼び出し元がメソッドを呼び出すために<br/>この依存関係を取得する方法を見つけ出す必要があることを意味します。<br/>また呼び出し元は間違ったサービスを渡すか<br/>あるいはまったくサービスを渡さない可能性もあります。<br/>

[Approach 4 - Pass Service to Method<br/>](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/04_MethodInjectionService)

## 方法5.エンティティメソッドに一意性チェックのための関数を渡す

これは前のオプションと同じですが型を渡す代わりに関数を渡すだけです。<br/>残念なことにこの関数は作業を行うために<br/>必要な依存関係やデータをすべて持っている必要があり<br/>そうでなければエンティティメソッドにカプセル化されているはずです。<br/>また呼び出し側のコードに適切な関数や有用な関数を<br/>渡すことを要求するような設計にはなっていません（no-op関数は簡単に渡すことができました）。<br/>カプセル化がされていないため<br/>ビジネスルールの検証はドメインモデルでは全く実施されず<br/>クライアントアプリケーション開発者の注意力と規律によってのみ実施されます<br/>（たとえあなたであっても、簡単に見逃してしまいます）。<br/>

[Approach 5 - Pass Function to Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/05_MethodInjectionFunction)<br/>

## 方法6.エンティティメソッドにフィルタリングされたデータを渡す

これはアプローチ3のバリエーションで<br/>呼び出し側のコードが提案された名前にマッチする既存の名前だけを渡して<br/>新しい名前がすでに存在しているかどうかをメソッドが判断できるようにしたものです。<br/>これは一意性のチェックを実際に行わずに<br/>ビジネスルールに必要な有用な作業をほぼすべて行っているように見えます。<br/>これは呼び出し側のコードにあまりにも多くの作業と知識を要求しています。<br/>

[Approach 6 - Pass Filtered Data to Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/06_PassFilteredDataToMethod)<br/>

## 方法7.貧血症の子には集約を使用する

この問題では当該エンティティが単体なのか<br/>集合体の一部なのかについて言及されていない。<br/>集約を導入する場合すべての名前の変更または追加に責任を持たせることで<br/>ビジネス上の不変性（一意名）に責任を持たせることができます。<br/>集約にその不変量に責任を持たせることは<br/>特に子エンティティ間に関連する場合<br/>多くの場合意味があります。<br/>しかし集約ルートが神クラスとなり<br/>すべての子エンティティが貧弱なDTOにならないように<br/>注意する必要があります（このアプローチは基本的にそうなります）。<br/>

[Approach 7 - Aggregate with Anemic Children<br/>](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/07_AggregateWithAnemicChildren)

## 方法8.ダブルディスパッチで集約を使用する

ダブルディスパッチはクラスのインスタンスをメソッドに渡して<br/>メソッドがそのインスタンスをコールバックできるようにするパターンです。<br/>多くの場合渡されるのは「現在のインスタンス」または `this` です。<br/>これは集約が子プロセスの振る舞いに責任を持ちつつ<br/>集約にコールバックして不変性を強化する方法を提供します。<br/>私はドメインモデルにおいて一方通行の関係を維持することを好みます。<br/>したがって集約からその子へのナビゲーションプロパティがある一方で<br/>他の方法へのナビゲーションはありません。<br/>したがって集約への参照を取得するには<br/> `UpdateName` メソッドに渡す必要があります。<br/>そしてもちろん期待されるものが実際にここで渡されることを強制するものは何もありません。<br/>呼び出し側のコードでは、NULLや集合体の新しいインスタンスなどを渡すことができます。<br/>

[Approach 8 - Aggregate with Double Dispatch<br/>](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/08_AggregateWithDblDispatch)

## 方法9.C#のイベントで集約を利用する

これはイベントの使い方を紹介する最初のオプションです。<br/>イベントはあるアクションに反応する必要があるロジックがある場合に意味を持ちます。<br/>例えば「誰かが製品の名前を変更しようとしたとき<br/>その名前がすでに使用されている場合は例外を投げる」ようにします。<br/>この方法ではC#言語のイベントを使用しますが<br/>残念ながら適切に実装するためには多くのコードが必要になります。<br/>

[Approach 9 - Aggregate with C# Events](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/09_AggregateWithEvents)<br/>

## 方法10.MediatRのイベントで集約を使用する

このアプローチでは<br/>MediatRで管理されたイベントと組み合わせたアグリゲートを使用します。<br/>エンティティやそのメソッドにMediatRを渡す必要がないように<br/>静的なヘルパークラスを使っています。<br/>これは一般的なアプローチで、<br/>10年以上にわたってうまく使われてきたものです。<br/>このアプローチはAPIの観点から最もクリーンなクライアント体験を提供します。<br/>テストを見てください。<br/>各テストが行っているのはリポジトリからアグリゲートを取得し<br/>メソッドを呼び出すことだけであることに注意してください。<br/>この場合実際のクライアントコードを模倣したテストコードは<br/> 配管コードをいじったり関数や特別なサービスを渡したりする必要がありません。<br/>メソッドはクリーンでAPIもクリーンそして<br/>ビジネスロジックは特定のクラスで責任をもって実行されます。<br/>

[Approach 10 - Aggregates with MediatR Events<br/>](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/10_AggregateWithMediatR)


## 方法11.ドメインイベントを使用する(集約なし)

時には集合体を巻き込んだり<br/>モデル化したりすることが意味をなさないことがあります。<br/>この場合にもインフラストラクチャの依存関係を活用しながら<br/>エンティティにカプセル化されたロジックを維持する方法を提供するために<br/>ドメインイベントを活用することができます。<br/>このアプローチにおけるクライアントエクスペリエンスは<br/>[新製品を追加するときにもう少しクライアントロジックが必要](https://github.com/ardalis/DDD-NoDuplicates/blob/master/NoDuplicatesDesigns/11_DomainEventsMediatR/AddProductTests.cs#L44-L45)という点を除いて<br/>前のアプローチと非常によく似ています。<br/>それ以外はクライアント体験は非常にクリーンで一貫性があり<br/>ドメイン層から漏れる実装の詳細で混乱することはありません。<br/>

[Approach 11 - MediatR Domain Events](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/11_DomainEventsMediatR)<br/>

ASP.NET Coreアプリでの設定方法を解説した記事、[MediatRでドメインイベントの即時救済](https://ardalis.com/immediate-domain-event-salvation-with-mediatr/)もあります。<br/>

## 備考

複数のエンティティやそのピアに関わるビジネスルールがある場合<br/>集約を導入することが理にかなっていることが多いようです。集約ルートを使用する場合<br/>すべてのロジックをルートに置いて<br/>貧弱な子エンティティを持つことにならないように注意する必要があります。<br/>私は子エンティティからアグリゲートルートへの通信に<br/>イベントを使用して成功しました（このサンプルに加え、[AggregateEvents](https://github.com/ardalis/AggregateEvents)を参照してください）。<br/>

集約がない場合または集約が1つしかなく<br/>その中に何百万もの他のエンティティがある場合（これは実現不可能かもしれません）、<br/>最後の例で示したようにドメインイベントを使用して<br/>コレクションに制約を強制することができます。<br/>最後の2つのアプローチで示したようにドメインイベントを使用すると<br/>[Explicit Dependencies Principle](https://deviq.com/explicit-dependencies-principle/) に違反しますが<br/>この原則は主にエンティティではなくサービスに適用されます。<br/>この場合ドメインイベントを活用して自分のドメインにきれいなインターフェースを提供し<br/>エンティティにカプセル化した論理を維持するというトレードオフは価値があると感じています。<br/>

## 参考文献

This project was inspired by [this exchange on twitter between Kamil and me](https://twitter.com/kamgrzybek/status/1280868055627763713). Kamil's ideas for approaching this problem included:

1. Pass all the current names to the update method. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/03_PassDataToMethod)
2. Pass all the names matching the proposed name to the update method. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/06_PassFilteredDataToMethod)
3. Pass an `IUniquenessChecker` into the update method which returns a count of entities with that name. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/04_MethodInjectionService)
4. Pass a function that performs the same logic as #3. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/05_MethodInjectionFunction)
5. Check the invariant in an Aggregate. [Done - Anemic Children](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/07_AggregateWithAnemicChildren)
6. Create a unique constraint in the database. [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/01_Database)

My own two approaches include:

7. Use a domain service (which will lead to an anemic entity) [Done](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/02_DomainService)
8. Use domain events and a handler to perform the logic

### その他

[Learn Domain-Driven Design Fundamentals](https://app.pluralsight.com/library/courses/fundamentals-domain-driven-design)
