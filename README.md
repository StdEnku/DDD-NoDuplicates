# 重複名なしを実現するドメインモデルの設計例

重複を許さないというビジネスルールを適用するためのいくつかの設計アプローチ。

## 問題点

あなたのドメインモデルにはあるエンティティがあります。
このエンティティは `name` プロパティを持っています。
この名前はアプリケーション内で一意であることを保証する必要があります。
この問題をどのように解決するのでしょうか？
このリポジトリではビジネスロジックやルールをドメインモデル内に
保持することを目的としたDDDアプリケーションで
これを行うための11の異なる方法を示しています。

## アプローチ

以下はこの問題を解決するためのさまざまなアプローチです。
それぞれのケースで必要なクラスはフォルダにまとめられています。
実際のアプリケーションではこれらはいくつかのプロジェクトに分割され
ドメインモデル、リポジトリ実装、テスト（または実際のUIクライアント）が
それぞれ別の場所に配置されると想像してください。

## 方法1.データベース

この最も単純なアプローチは
ドメインモデルのルールを無視し[^1]
あなたのためにそれを処理する何らかのインフラに依存することです。
一般的にこれはSQLデータベースの一意制約の形をとり
エンティティのテーブルに重複する値を挿入または更新しようとすると例外を発生させます。
これは機能しますが永続性の変更[^2]やドメインモデル自体のビジネスルールの保持はできません。
しかしこれも簡単で性能も良いので検討する価値はあります。

本リポジトリでは実際のデータベースを使用していないので`ProductRepository`で動作を偽っています。

[Approach 1 - Database](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/01_Database)

[^1]: これは基本的にDDDの観点からは不正行為です
[^2]: 一意制約をサポートしない、またはコストやパフォーマンスが許容できないシステムへ

## 方法2.ドメインサービス

1つの選択肢はドメインサービス内で名前が重複しているかどうかを検出し
名前の更新を強制的にそのサービスを通して流すことである。
これはドメインモデル内にロジックを保持するが貧弱なエンティティをもたらす。
このアプローチではエンティティにはロジックがなく
エンティティに対するあらゆる作業はドメインサービスを介して行われる必要があることに注意すること。
この設計の問題点は時間が経つにつれてすべてのロジックをサービスに置き
サービスが直接エンティティを操作するようになり
エンティティ内のロジックのカプセル化をすべて排除してしまうことである。
なぜこのようなことになるのでしょうか？
ドメインモデルのクライアントは一貫したAPIで作業したいのであって
ある時はエンティティ上のメソッドである時はサービス経由のメソッドで
どちらかを使わなければならない理由や根拠がないまま作業することは望まないからです。
そしてエンティティ上で始まったメソッドは依存関係がある場合
サービスに移動する必要があるかもしれません。

[Approach 2 - Domain Service](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/02_DomainService)

## 方法3.エンティティメソッドに必要なデータを渡す

ひとつの方法として、エンティティメソッドにチェックを行うために必要なデータをすべて渡すことができます。この場合、異なる名前であると想定されるすべての名前のリストを渡す必要があります。もちろん、エンティティの現在の名前は含めないでください。なぜなら、当然ながら現在の名前がそのまま使用されるからです。数百万以上のエンティティがある場合、この方法ではうまくスケールしないでしょう。

[Approach 3 - Pass Necessary Data into Entity Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/03_PassDataToMethod)

## 方法4.エンティティメソッドに一意性チェックのためのサービスを渡す

このオプションでは、メソッドインジェクションによって依存性注入を行い、必要な依存性をエンティティに渡します。残念ながら、これは呼び出し元がメソッドを呼び出すために、この依存関係を取得する方法を見つけ出す必要があることを意味します。また、呼び出し元は間違ったサービスを渡すか、あるいはまったくサービスを渡さない可能性もあります。

[Approach 4 - Pass Service to Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/04_MethodInjectionService)

## 方法5.エンティティメソッドに一意性チェックのための関数を渡す

これは前のオプションと同じですが、型を渡す代わりに関数を渡すだけです。残念なことに、この関数は作業を行うために必要な依存関係やデータをすべて持っている必要があり、そうでなければエンティティメソッドにカプセル化されているはずです。また、呼び出し側のコードに適切な関数や有用な関数を渡すことを要求するような設計にはなっていません（no-op関数は簡単に渡すことができました）。カプセル化がされていないため、ビジネスルールの検証はドメインモデルでは全く実施されず、クライアントアプリケーション開発者の注意力と規律によってのみ実施されます（たとえあなたであっても、簡単に見逃してしまいます）。

[Approach 5 - Pass Function to Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/05_MethodInjectionFunction)

## 方法6.エンティティメソッドにフィルタリングされたデータを渡す

これはアプローチ3のバリエーションで、呼び出し側のコードが提案された名前にマッチする既存の名前だけを渡して、新しい名前がすでに存在しているかどうかをメソッドが判断できるようにしたものです。これは、一意性のチェックを実際に行わずに、ビジネスルールに必要な有用な作業をほぼすべて行っているように見えます。これは、呼び出し側のコードにあまりにも多くの作業と知識を要求しています。

[Approach 6 - Pass Filtered Data to Method](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/06_PassFilteredDataToMethod)

## 方法7.貧血症の子には集約を使用する

この問題では、当該エンティティが単体なのか、集合体の一部なのかについて言及されていない。集約を導入する場合、すべての名前の変更または追加に責任を持たせることで、ビジネス上の不変性（一意名）に責任を持たせることができます。集約にその不変量に責任を持たせることは、特に子エンティティ間に関連する場合、多くの場合、意味があります。しかし、集約ルートが神クラスとなり、すべての子エンティティが貧弱なDTOにならないように注意する必要があります（このアプローチは基本的にそうなります）。

[Approach 7 - Aggregate with Anemic Children](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/07_AggregateWithAnemicChildren)

## 方法8.ダブルディスパッチで集約を使用する

ダブルディスパッチは、クラスのインスタンスをメソッドに渡して、メソッドがそのインスタンスをコールバックできるようにするパターンです。多くの場合、渡されるのは「現在のインスタンス」または `this` です。これは、集約が子プロセスの振る舞いに責任を持ちつつ、集約にコールバックして不変性を強化する方法を提供します。私はドメインモデルにおいて一方通行の関係を維持することを好みます。したがって、集約からその子へのナビゲーションプロパティがある一方で、他の方法へのナビゲーションはありません。したがって、集約への参照を取得するには、 `UpdateName` メソッドに渡す必要があります。そしてもちろん、期待されるものが実際にここで渡されることを強制するものは何もありません。呼び出し側のコードでは、NULLや集合体の新しいインスタンスなどを渡すことができます。

[Approach 8 - Aggregate with Double Dispatch](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/08_AggregateWithDblDispatch)

## 方法9.C#のイベントで集約を利用する

これは、イベントの使い方を紹介する最初のオプションです。イベントは、あるアクションに反応する必要があるロジックがある場合に意味を持ちます。例えば、「誰かが製品の名前を変更しようとしたとき、その名前がすでに使用されている場合は例外を投げる」ようにします。この方法では、C#言語のイベントを使用しますが、残念ながら適切に実装するためには多くのコードが必要になります。

[Approach 9 - Aggregate with C# Events](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/09_AggregateWithEvents)

## 方法10.MediatRのイベントで集約を使用する

このアプローチでは、MediatRで管理されたイベントと組み合わせたアグリゲートを使用します。エンティティやそのメソッドにMediatRを渡す必要がないように、静的なヘルパークラスを使っています。これは一般的なアプローチで、10年以上にわたってうまく使われてきたものです([Domain Event Salvation from 2009](http://udidahan.com/2009/06/14/domain-events-salvation/)を参照)。このアプローチは、API の観点から最もクリーンなクライアント体験を提供します。テストを見てください。各テストが行っているのは、リポジトリからアグリゲートを取得し、メソッドを呼び出すことだけであることに注意してください。この場合、実際のクライアントコードを模倣したテストコードは、 配管コードをいじったり関数や特別なサービスを渡したりする必要がありません。メソッドはクリーンで、APIもクリーン、そしてビジネスロジックは特定のクラスで責任をもって実行されます。

[Approach 10 - Aggregates with MediatR Events](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/10_AggregateWithMediatR)


## 方法11.ドメインイベントを使用する(集約なし)

時には、集合体を巻き込んだり、モデル化したりすることが意味をなさないことがあります。この場合にも、インフラストラクチャの依存関係を活用しながら、エンティティにカプセル化されたロジックを維持する方法を提供するために、ドメインイベントを活用することができます。このアプローチにおけるクライアントエクスペリエンスは、[新製品を追加するときにもう少しクライアントロジックが必要](https://github.com/ardalis/DDD-NoDuplicates/blob/master/NoDuplicatesDesigns/11_DomainEventsMediatR/AddProductTests.cs#L44-L45)という点を除いて、前のアプローチと非常によく似ています。それ以外は、クライアント体験は非常にクリーンで一貫性があり、ドメイン層から漏れる実装の詳細で混乱することはありません。

[Approach 11 - MediatR Domain Events](https://github.com/ardalis/DDD-NoDuplicates/tree/master/NoDuplicatesDesigns/11_DomainEventsMediatR)

ASP.NET Coreアプリでの設定方法を解説した記事、[MediatRでドメインイベントの即時救済](https://ardalis.com/immediate-domain-event-salvation-with-mediatr/)もあります。

## 備考

複数のエンティティやそのピアに関わるビジネスルールがある場合、集約を導入することが理にかなっていることが多いようです。集約ルートを使用する場合、すべてのロジックをルートに置いて、貧弱な子エンティティを持つことにならないように注意する必要があります。私は、子エンティティからアグリゲートルートへの通信にイベントを使用して成功しました（このサンプルに加え、[AggregateEvents](https://github.com/ardalis/AggregateEvents)を参照してください）。

集約がない場合、または集約が1つしかなく、その中に何百万もの他のエンティティがある場合（これは実現不可能かもしれません）、最後の例で示したように、ドメインイベントを使用して、コレクションに制約を強制することができます。最後の2つのアプローチで示したようにドメインイベントを使用すると、[Explicit Dependencies Principle](https://deviq.com/explicit-dependencies-principle/) に違反しますが、この原則は主にエンティティではなくサービスに適用されます。この場合、ドメインイベントを活用して自分のドメインにきれいなインターフェースを提供し、エンティティにカプセル化した論理を維持するというトレードオフは価値があると感じています。

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
