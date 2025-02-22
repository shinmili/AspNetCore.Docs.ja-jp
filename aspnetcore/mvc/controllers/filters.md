---
title: ASP.NET Core フィルター
author: ardalis
description: フィルターのしくみと ASP.NET Core でそれを使用する方法について説明します。
ms.author: riande
ms.custom: mvc
ms.date: 05/08/2019
uid: mvc/controllers/filters
ms.openlocfilehash: 50b199744f32ad19335080da406db69665ec1ae9
ms.sourcegitcommit: 7a40c56bf6a6aaa63a7ee83a2cac9b3a1d77555e
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 07/12/2019
ms.locfileid: "67856153"
---
# <a name="filters-in-aspnet-core"></a>ASP.NET Core フィルター

作成者: [Kirk Larkin](https://github.com/serpent5)、[Rick Anderson](https://twitter.com/RickAndMSFT)、[Tom Dykstra](https://github.com/tdykstra/)、[Steve Smith](https://ardalis.com/)

ASP.NET Core で*フィルター*を使用すると、要求処理パイプラインの特定のステージの前または後にコードを実行できます。

組み込みのフィルターでは次のようなタスクが処理されます。

* 許可 (ユーザーに許可が与えられていないリソースの場合、アクセスを禁止する)。
* 応答キャッシュ (要求パイプラインを迂回し、キャッシュされている応答を返す)。

横断的な問題を処理するカスタム フィルターを作成できます。 横断的な問題の例には、エラー処理、キャッシュ、構成、認証、ログなどがあります。  フィルターにより、コードの重複が回避されます。 たとえば、エラー処理例外フィルターではエラー処理を統合できます。

このドキュメントは、Razor Pages、API コントローラー、ビューのあるコントローラーに当てはまります。

[サンプルを表示またはダウンロード](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/controllers/filters/sample)します ([ダウンロード方法](xref:index#how-to-download-a-sample))。

## <a name="how-filters-work"></a>フィルターのしくみ

*ASP.NET Core のアクション呼び出しパイプライン*内で実行されるフィルターは、*フィルター パイプライン*と呼ばれることがあります。  フィルター パイプラインは、ASP.NET Core が実行するアクションを選択した後に実行されます。

![要求は、他のミドルウェア、ルーティング ミドルウェア、アクション選択、ASP.NET Core のアクション呼び出しパイプラインを通じて処理されます。 要求の処理は、クライアントに送信される応答になる前に、アクション選択、ルーティング ミドルウェア、およびさまざまな他のミドルウェアを遡って続行されます。](filters/_static/filter-pipeline-1.png)

### <a name="filter-types"></a>フィルターの種類

フィルターの種類はそれぞれ、フィルター パイプラインの異なるステージで実行されます。

* [承認フィルター](#authorization-filters)は、最初に実行され、ユーザーが要求に対して承認されているかどうかを判断するために使用されます。 承認フィルターでは、要求が承認されていない場合、パイプラインがショートサーキットされます。

* [リソース フィルター](#resource-filters):

  * 承認後に実行されます。  
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter.OnResourceExecuting*> では、残りのフィルター パイプラインの前にコードを実行できます。 たとえば、`OnResourceExecuting` では、モデル バインディングの前にコードを実行できます。
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter.OnResourceExecuted*> では、残りのパイプラインの完了後にコードを実行できます。

* [アクション フィルター](#action-filters)は、個々のアクション メソッドが呼び出される直前と直後にコードを実行できます。 アクションに渡される引数とアクションから返される結果を操作するために使用できます。 アクション フィルターは Razor Pages ではサポート**されていません**。

* [例外フィルター](#exception-filters)は、応答本文に何かが書き込まれる前に発生する未処理の例外にグローバル ポリシーを適用するために使用されます。

* [結果フィルター](#result-filters)は、個々のアクション結果の実行の直前と直後にコードを実行できます。 アクション メソッドが正常に実行された場合にのみ実行されます。 ビューまたはフォーマッタ実行を取り囲む必要があるロジックに便利です。

フィルターの種類がフィルター パイプラインでどのように連携しているかを、次の図に示します。

![要求は、承認フィルター、リソース フィルター、モデル バインド、アクション フィルター、アクションの実行とアクション結果の変換、例外フィルター、結果フィルター、結果の実行を介して処理されます。 終了間際に、要求は結果フィルターとリソース フィルターのみで処理されてから、クライアントに送信される応答になります。](filters/_static/filter-pipeline-2.png)

## <a name="implementation"></a>実装

フィルターは、異なるインターフェイス定義を介して、同期と非同期の実装をサポートします。

同期フィルターでは、パイプライン ステージの前 (`On-Stage-Executing`) と後 (`On-Stage-Executed`) にコードを実行できます。 たとえば、`OnActionExecuting` はアクション メソッドの呼び出し前に呼び出されます。 `OnActionExecuted` は、アクション メソッドが戻った後に呼び出されます。

[!code-csharp[](./filters/sample/FiltersSample/Filters/MySampleActionFilter.cs?name=snippet_ActionFilter)]

非同期フィルターにより `On-Stage-ExecutionAsync` メソッドが定義されます。

[!code-csharp[](./filters/sample/FiltersSample/Filters/SampleAsyncActionFilter.cs?name=snippet)]

上記のコードでは、`SampleAsyncActionFilter` にアクション メソッドを実行する <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutionDelegate> (`next`) があります。  各 `On-Stage-ExecutionAsync` メソッドは、フィルターのパイプライン ステージを実行する `FilterType-ExecutionDelegate` を受け取ります。

### <a name="multiple-filter-stages"></a>複数のフィルター ステージ

複数のフィルター ステージのためのインターフェイスを 1 つのクラスで実装できます。 たとえば、<xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> クラスは、`IActionFilter`、`IResultFilter`、およびそれらの非同期バージョンを実装します。

フィルター インターフェイスの同期と非同期バージョンの**両方ではなく**、**いずれか**を実装します。 ランタイムは、最初にフィルターが非同期インターフェイスを実装しているかどうかをチェックして、している場合はそれを呼び出します。 していない場合は、同期インターフェイスのメソッドを呼び出します。 非同期インターフェイスと同期インターフェイスの両方が 1 つのクラスで実装される場合、非同期メソッドのみが呼び出されます。 <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> などの抽象クラスを使用する場合は、同期メソッドのみをオーバーライドするか、フィルターの種類ごとに非同期メソッドをオーバーライドします。

### <a name="built-in-filter-attributes"></a>組み込みのフィルター属性

ASP.NET Core には、サブクラスを作成したり、カスタマイズしたりできる組み込みの属性ベースのフィルターが含まれます。 たとえば、次の結果フィルターは、応答にヘッダーを追加します。

<a name="add-header-attribute"></a>

[!code-csharp[](./filters/sample/FiltersSample/Filters/AddHeaderAttribute.cs?name=snippet)]

上記の例のように、属性によってフィルターは引数を受け取ることができます。 `AddHeaderAttribute` をコントローラーまたはアクション メソッドに適用し、HTTP ヘッダーの名前と値を指定します。

[!code-csharp[](./filters/sample/FiltersSample/Controllers/SampleController.cs?name=snippet_AddHeader&highlight=1)]

<!-- `https://localhost:5001/Sample` -->

フィルター インターフェイスのいくつかには対応する属性があり、カスタムの実装に基底クラスとして使用できます。

フィルター属性:

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.ExceptionFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.Filters.ResultFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.FormatFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute>

## <a name="filter-scopes-and-order-of-execution"></a>フィルターのスコープと実行の順序

フィルターは、3 つの*スコープ*のいずれかでパイプラインに追加することができます。

* アクションで属性を使用します。
* コントローラーで属性を使用します。
* 次のコードのように、すべてのコントローラーとアクションに対してグローバルに:

[!code-csharp[](./filters/sample/FiltersSample/StartupGF.cs?name=snippet_ConfigureServices)]

上記のコードでは、[MvcOptions.Filters](xref:Microsoft.AspNetCore.Mvc.MvcOptions.Filters) コレクションを利用し、3 つのフィルターがグローバルに追加されます。

### <a name="default-order-of-execution"></a>実行の既定の順序

パイプラインの特定のステージに対して複数のフィルターがある場合に、スコープがフィルターの実行の既定の順序を決定します。  グローバル フィルターがクラス フィルターを囲み、クラス フィルターがメソッド フィルターを囲みます。

フィルターの入れ子の結果として、フィルターの *after* コードが *before* コードと逆の順序で実行されます。 フィルター シーケンス:

* グローバル フィルターの *before* コード。
  * コントローラー フィルターの *before* コード。
    * アクション メソッド フィルターの *before* コード。
    * アクション メソッド フィルターの *after* コード。
  * コントローラー フィルターの *after* コード。
* グローバル フィルターの *after* コード。
  
次の例は、同期アクション フィルターに対してフィルター メソッドが呼び出される順序を示しています。

| シーケンス | フィルターのスコープ | フィルター メソッド |
|:--------:|:------------:|:-------------:|
| 1 | Global | `OnActionExecuting` |
| 2 | コントローラー | `OnActionExecuting` |
| 3 | メソッド | `OnActionExecuting` |
| 4 | メソッド | `OnActionExecuted` |
| 5 | コントローラー | `OnActionExecuted` |
| 6 | Global | `OnActionExecuted` |

このシーケンスが示すもの:

* メソッド フィルターは、コントローラー フィルター内で入れ子になります。
* コントローラー フィルターは、グローバル フィルター内で入れ子になります。

### <a name="controller-and-razor-page-level-filters"></a>コントローラーと Razor Page のレベル フィルター

<xref:Microsoft.AspNetCore.Mvc.Controller> 基底クラスから継承するすべてのコントローラーに、[Controller.OnActionExecuting](xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecuting*)、[Controller.OnActionExecutionAsync](xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecutionAsync*)、[Controller.OnActionExecuted](xref:Microsoft.AspNetCore.Mvc.Controller.OnActionExecuted*)
`OnActionExecuted` メソッドが含まれています。 これらのメソッド:

* 特定のアクションに対して実行されるフィルターをラップします。
* `OnActionExecuting` は、あらゆるアクション フィルターの前に呼び出されます。
* `OnActionExecuted` は、あらゆるアクション フィルターの後に呼び出されます。
* `OnActionExecutionAsync` は、あらゆるアクション フィルターの前に呼び出されます。 `next` の後のフィルターのコードは、アクション メソッドの後に実行されます。

たとえば、ダウンロード サンプルで、`MySampleActionFilter` は起動中、グローバルに適用されます。

`TestController`:

* `SampleActionFilterAttribute` (`[SampleActionFilter]`) が `FilterTest2` アクションに適用されます。
* `OnActionExecuting` と `OnActionExecuted` がオーバーライドされます。

[!code-csharp[](./filters/sample/FiltersSample/Controllers/TestController.cs?name=snippet)]

`https://localhost:5001/Test/FilterTest2` に移動すると、次のコードが実行されます。

* `TestController.OnActionExecuting`
  * `MySampleActionFilter.OnActionExecuting`
    * `SampleActionFilterAttribute.OnActionExecuting`
      * `TestController.FilterTest2`
    * `SampleActionFilterAttribute.OnActionExecuted`
  * `MySampleActionFilter.OnActionExecuted`
* `TestController.OnActionExecuted`

Razor Pages の場合、「[フィルター メソッドをオーバーライドして Razor ページにフィルターを実装する](xref:razor-pages/filter#implement-razor-page-filters-by-overriding-filter-methods)」を参照してください。

### <a name="overriding-the-default-order"></a>既定の順序のオーバーライド

<xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter> を実装することで、実行の既定の順序をオーバーライドできます。 `IOrderedFilter` により、実行の順序を決定するために、スコープよりも優先される <xref:Microsoft.AspNetCore.Mvc.Filters.IOrderedFilter.Order> プロパティが公開されます。 より低い `Order` 値を持つフィルター:

* より高い `Order` の値を持つフィルターのそれよりも前に *before* コードが実行されます。
* より高い `Order` の値を持つフィルターのそれよりも後に *after* コードが実行されます。

`Order` プロパティはコンストラクター パラメーターで設定できます。

```csharp
[MyFilter(Name = "Controller Level Attribute", Order=1)]
```

上記の例にある同じ 3 つのアクション フィルターを検討してください。 コントローラーとグローバル フィルターの `Order` プロパティが 1 と 2 にそれぞれ設定される場合、実行順序が逆になります。

| シーケンス | フィルターのスコープ | `Order` プロパティ | フィルター メソッド |
|:--------:|:------------:|:-----------------:|:-------------:|
| 1 | メソッド | 0 | `OnActionExecuting` |
| 2 | コントローラー | 1  | `OnActionExecuting` |
| 3 | Global | 2  | `OnActionExecuting` |
| 4 | Global | 2  | `OnActionExecuted` |
| 5 | コントローラー | 1  | `OnActionExecuted` |
| 6 | メソッド | 0  | `OnActionExecuted` |

フィルターの実行順序を決定するときに、`Order` プロパティによりスコープがオーバーライドされます。 最初に順序でフィルターが並べ替えられ、次に同じ順位の優先度を決めるためにスコープが使用されます。 組み込みのフィルターはすべて `IOrderedFilter` を実装し、既定の `Order` 値を 0 に設定します。 組み込みのフィルターの場合、`Order` をゼロ以外の値に設定しない限り、スコープによって順序が決定されます。

## <a name="cancellation-and-short-circuiting"></a>キャンセルとショートサーキット

フィルター メソッドに提供される <xref:Microsoft.AspNetCore.Mvc.Filters.ResourceExecutingContext> パラメーターで <xref:Microsoft.AspNetCore.Mvc.Filters.ResourceExecutingContext.Result> プロパティを設定することで、フィルター パイプラインをショートサーキットできます。 たとえば、次のリソース フィルターは、パイプラインの残りの部分が実行されるのを防止します。

<a name="short-circuiting-resource-filter"></a>

[!code-csharp[](./filters/sample/FiltersSample/Filters/ShortCircuitingResourceFilterAttribute.cs?name=snippet)]

次のコードでは、`ShortCircuitingResourceFilter` と `AddHeader` の両方のフィルターが `SomeResource` アクション メソッドをターゲットにしています。 `ShortCircuitingResourceFilter`:

* これはリソース フィルターであり、`AddHeader` はアクション フィルターであるため、最初に実行されます。
* パイプラインの残りの部分は迂回されます。

そのため、`SomeResource` アクションの場合、`AddHeader` フィルターが実行されることはありません。 `ShortCircuitingResourceFilter` が最初に実行された場合は、両方のフィルターがアクション メソッド レベルで適用されると、この動作が同じになります。 そのフィルターの種類が原因で、あるいは `Order` プロパティの明示的な使用により、`ShortCircuitingResourceFilter` が最初に実行されます。

[!code-csharp[](./filters/sample/FiltersSample/Controllers/SampleController.cs?name=snippet_AddHeader&highlight=1,9)]

## <a name="dependency-injection"></a>依存関係の挿入

フィルターは種類ごとまたはインスタンスごとに追加できます。 インスタンスが追加される場合、そのインスタンスはすべての要求に対して使用されます。 種類が追加される場合、それは種類でアクティブ化されます。 種類でアクティブ化されるフィルターの意味:

* インスタンスは要求ごとに作成されます。
* コンストラクターの依存関係は、[依存関係の挿入](xref:fundamentals/dependency-injection) (DI) によって入力されます。

属性として実装され、コントローラー クラスまたはアクション メソッドに直接追加されるフィルターは、[依存関係の挿入](xref:fundamentals/dependency-injection) (DI) によって提供されるコンストラクターの依存関係を持つことはできません。 コンストラクターの依存関係は、次の理由から DI によって与えられません。

* 属性には、適用される場所で提供される独自のコンストラクター パラメーターが必要です。 
* これは、属性のしくみの制限です。

次のフィルターでは、DI から提供されるコンストラクターの依存関係がサポートされます。

* <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute>
* <xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute>
* 属性に実装された <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory>。

上記のフィルターは、コントローラーまたはアクション メソッドに適用できます。

ロガーは DI から利用できます。 ただし、ログ目的でのみフィルターを作成し、使用することは避けてください。 [組み込みフレームワークのログ機能](xref:fundamentals/logging/index)で通常、ログ記録に必要なものが与えられます。 フィルターに追加されるログ記録:

* ビジネス ドメインの懸念事項やフィルターに固有の動作に焦点を合わせます。
* アクションやその他のフレームワーク イベントはログに記録**しない**でください。 組み込みフィルターでは、アクションとフレームワーク イベントが記録されます。

### <a name="servicefilterattribute"></a>ServiceFilterAttribute

サービス フィルターの実装の種類は `ConfigureServices` に登録されています。 <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> は DI からフィルターのインスタンスを取得します。

このコードでは、`AddHeaderResultServiceFilter` が示されています。

[!code-csharp[](./filters/sample/FiltersSample/Filters/LoggingAddHeaderFilter.cs?name=snippet_ResultFilter)]

次のコードでは、`AddHeaderResultServiceFilter` が DI コンテナーに追加されます。

[!code-csharp[](./filters/sample/FiltersSample/Startup.cs?name=snippet_ConfigureServices&highlight=4)]

次のコードでは、`ServiceFilter` 属性により、DI から `AddHeaderResultServiceFilter` フィルターのインスタンスが取得されます。

[!code-csharp[](./filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet_ServiceFilter&highlight=1)]

`ServiceFilterAttribute` を使用するときに、[ServiceFilterAttribute.IsReusable](xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute.IsReusable) を設定します。

* フィルター インスタンスが、それが作成された要求範囲の外で再利用される*可能性がある*ことを示唆します。 ASP.NET Core ランタイムで保証されないこと:

  * フィルターのインスタンスが 1 つ作成されます。
  * フィルターが後の時点で、DI コンテナーから再要求されることはありません。

* シングルトン以外で有効期間があるサービスに依存するフィルターと共に使用しないでください。

 <xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> は、<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> を実装します。 `IFilterFactory` は、<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> インスタンスを作成するために <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance*> メソッドを公開します。 `CreateInstance` により、DI から指定の型が読み込まれます。

### <a name="typefilterattribute"></a>TypeFilterAttribute

<xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute> は<xref:Microsoft.AspNetCore.Mvc.ServiceFilterAttribute> と似ていますが、その型は DI コンテナーから直接解決されません。 <xref:Microsoft.Extensions.DependencyInjection.ObjectFactory?displayProperty=fullName> を使って型をインスタンス化します。

`TypeFilterAttribute` 型は DI コンテナーから直接解決されないためです。

* `TypeFilterAttribute` を利用して参照される型は、DI コンテナーに登録する必要がありません。  DI コンテナーによって依存関係が満たされています。
* `TypeFilterAttribute` は必要に応じて、型のコンストラクター引数を受け取ることができます。

`TypeFilterAttribute` を使用するときに、[TypeFilterAttribute.IsReusable](xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute.IsReusable) を設定します。
* フィルター インスタンスが、それが作成された要求範囲の外で再利用される*可能性がある*ことを示唆します。 ASP.NET Core ランタイムでは、フィルターの 1 インスタンスが作成されるという保証はありません。

* シングルトン以外で有効期間があるサービスに依存するフィルターと共に使用しないでください。

次の例は、`TypeFilterAttribute` を使用して、型に引数を渡す方法を示しています。

[!code-csharp[](../../mvc/controllers/filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet_TypeFilter&highlight=1,2)]
[!code-csharp[](../../mvc/controllers/filters/sample/FiltersSample/Filters/LogConstantFilter.cs?name=snippet_TypeFilter_Implementation&highlight=6)]

<!-- 
https://localhost:5001/home/hi?name=joe
VS debug window shows 
FiltersSample.Filters.LogConstantFilter:Information: Method 'Hi' called
-->

## <a name="authorization-filters"></a>承認フィルター

承認フィルター:

* フィルター パイプライン内で実行される最初のフィルターです。
* アクション メソッドへのアクセスを制御します。
* before メソッドが与えられ、after メソッドは与えられません。

カスタム承認フィルターには、カスタム承認フレームワークが必要です。 カスタム フィルターを記述するよりも、独自の承認ポリシーを構成するか、カスタム承認ポリシーを記述することを選びます。 組み込み承認フィルター:

* 承認システムを呼び出します。
* 要求は承認されません。

承認フィルター内で例外をスロー**しません**。

* 例外は処理されません。
* 例外フィルターで例外が処理されません。

承認フィルターで例外が発生した場合、チャレンジ発行を検討してください。

承認の詳細については、[こちら](xref:security/authorization/introduction)を参照してください。

## <a name="resource-filters"></a>リソース フィルター

リソース フィルター:

* <xref:Microsoft.AspNetCore.Mvc.Filters.IResourceFilter> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResourceFilter> のインターフェイスを実装します。
* 実行により、ほとんどのフィルター パイプラインがラップされます。
* [承認フィルター](#authorization-filters)のみ、リソース フィルターの前に実行されます。

リソース フィルターは、ほとんどのパイプラインをショートサーキットする目的で役に立ちます。 たとえば、キャッシュ フィルターにより、キャッシュ ヒットがあるとき、残りのパイプラインを回避できます。

リソース フィルターの例:

* 前に示した[ショートサーキットするリソース フィルター](#short-circuiting-resource-filter)。
* [DisableFormValueModelBindingAttribute](https://github.com/aspnet/Entropy/blob/rel/2.0.0-preview2/samples/Mvc.FileUpload/Filters/DisableFormValueModelBindingAttribute.cs):

  * モデル バインドがフォーム データにアクセスすることを禁止します。
  * メモリにフォーム データが読み込まれないようにする目的で大きなファイルのアップロードに使用されます。

## <a name="action-filters"></a>アクション フィルター

> [!IMPORTANT]
> アクション フィルターは Razor Pages に適用**されません**。 Razor Pages では <xref:Microsoft.AspNetCore.Mvc.Filters.IPageFilter> と <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncPageFilter> がサポートされています。 詳細については、[Razor ページのフィルター メソッド](xref:razor-pages/filter)に関するページを参照してください。

アクション フィルター:

* <xref:Microsoft.AspNetCore.Mvc.Filters.IActionFilter> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncActionFilter> のインターフェイスを実装します。
* この実行はアクション メソッドの実行を取り囲みます。

次のコードは、サンプル アクション フィルターを示しています。

[!code-csharp[](./filters/sample/FiltersSample/Filters/MySampleActionFilter.cs?name=snippet_ActionFilter)]

<xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext> では次のプロパティが提供されます。

* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.ActionArguments> - アクション メソッドへの入力を読み取ることができます。
* <xref:Microsoft.AspNetCore.Mvc.Controller> - コントローラー インスタンスを操作できます。
* <xref:System.Web.Mvc.ActionExecutingContext.Result> - `Result` を設定すると、アクション メソッドと後続のアクション フィルターの実行がショートサーキットされます。

アクション メソッドで例外をスローする:

* 後続のフィルターの実行を回避します。
* `Result` の設定とは異なり、結果は成功ではなく、失敗として処理されます。

<xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext> は、`Controller` と `Result` に加え、次のプロパティを提供します。

* <xref:System.Web.Mvc.ActionExecutedContext.Canceled> - 別のフィルターによってアクションの実行がショートサーキットされた場合は、true になります。
* <xref:System.Web.Mvc.ActionExecutedContext.Exception> - アクションまたは前に実行されたアクション フィルターが例外をスローした場合は、null 以外になります。 このプロパティを null に設定する:

  * 例外が効果的に処理されます。
  * `Result` は、アクション メソッドから返されたかのように実行されます。

`IAsyncActionFilter` の場合、<xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutionDelegate> の呼び出しによって:

* 後続のすべてのアクション フィルターとアクション メソッドが実行されます。
* `ActionExecutedContext` を返します。

ショートサーキットするには、<xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutingContext.Result?displayProperty=fullName> を結果インスタンスに割り当てます。`next` (`ActionExecutionDelegate`) は呼び出さないでください。

このフレームワークからは、サブクラス化できる抽象 <xref:Microsoft.AspNetCore.Mvc.Filters.ActionFilterAttribute> が与えられます。

`OnActionExecuting` アクション フィルターを次の目的で使用できます。

* モデルの状態を検証します。
* 状態が有効でない場合は、エラーを返します。

[!code-csharp[](./filters/sample/FiltersSample/Filters/ValidateModelAttribute.cs?name=snippet)]

`OnActionExecuted` メソッドは、アクション メソッドの後に実行されます。

* また、<xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Result> プロパティを介してアクションの結果を表示したり、操作したりできます。
* <xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Canceled> は、アクションの実行が別のフィルターによってショートサーキットされた場合、true に設定されます。
* アクションまたは後続のアクション フィルターが例外をスローした場合、<xref:Microsoft.AspNetCore.Mvc.Filters.ActionExecutedContext.Exception> は null 以外の値に設定されます。 `Exception` を null に設定すると:

  * 例外が効果的に処理されます。
  * `ActionExecutedContext.Result` は、アクション メソッドから通常どおり返されたかのように実行されます。

[!code-csharp[](./filters/sample/FiltersSample/Filters/ValidateModelAttribute.cs?name=snippet2&higlight=12-99)]

## <a name="exception-filters"></a>例外フィルター

例外フィルター:

* <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncExceptionFilter> を実装します。 
* 共通のエラー処理ポリシーを実装する目的で使用できます。

次の例外フィルターのサンプルでは、カスタムのエラー ビューを使用して、アプリの開発中に発生する例外に関する詳細を表示します。

[!code-csharp[](./filters/sample/FiltersSample/Filters/CustomExceptionFilterAttribute.cs?name=snippet_ExceptionFilter&highlight=16-19)]

例外フィルター:

* before イベントと after イベントが与えられません。
* <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter.OnException*> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncExceptionFilter.OnExceptionAsync*> を実装します。
* Razor Page またはコントローラーの作成、[モデル バインド](xref:mvc/models/model-binding)、アクション フィルター、またはアクション メソッドで発生する未処理の例外を処理します。
* リソース フィルター、結果フィルター、または MVC 結果の実行で発生した例外はキャッチ**しません**。

例外を処理するには、<xref:System.Web.Mvc.ExceptionContext.ExceptionHandled> プロパティを `true` に設定するか、応答を記述します。 これにより、例外の伝達を停止します。 例外フィルターで例外を "成功" に変えることはできません。 それができるのは、アクション フィルターだけです。

例外フィルター:

* アクション内で発生する例外のトラップに適しています。
* エラー処理ミドルウェアほど柔軟ではありません。

例外処理にはミドルウェアを選択してください。 呼び出されたアクション メソッドに基づいてエラー処理が*異なる*状況でのみ例外フィルターを使用します。 たとえば、アプリには、API エンドポイントとビュー/HTML の両方に対するアクション メソッドがある場合があります。 API エンドポイントは、JSON としてのエラー情報を返す可能性がある一方で、ビュー ベースのアクションがエラー ページを HTML として返す可能性があります。

## <a name="result-filters"></a>結果フィルター

結果フィルター:

* インターフェイスを実装します:
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter>
  * <xref:Microsoft.AspNetCore.Mvc.Filters.IAlwaysRunResultFilter> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncAlwaysRunResultFilter>
* この実行はアクション結果の実行を取り囲みます。

### <a name="iresultfilter-and-iasyncresultfilter"></a>IResultFilter および IAsyncResultFilter

次は、HTTP ヘッダーを追加する結果フィルターのコードです。

[!code-csharp[](./filters/sample/FiltersSample/Filters/LoggingAddHeaderFilter.cs?name=snippet_ResultFilter)]

実行されている結果の種類は、アクションに依存します。 ビューを返すアクションには、実行されている <xref:Microsoft.AspNetCore.Mvc.ViewResult> の一部として、すべての razor 処理が含まれます。 API メソッドは、結果の実行の一部としていくつかのシリアル化を実行できます。 アクション結果に関する詳細は、[こちら](xref:mvc/controllers/actions)を参照してください。

結果フィルターは、アクションまたはアクション フィルターがアクションの結果を生成するときに、成功した結果に対してのみ実行されます。 例外フィルターが例外を処理するときには、結果フィルターは実行されません。

<xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter.OnResultExecuting*?displayProperty=fullName> メソッドは、<xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutingContext.Cancel?displayProperty=fullName> を `true` に設定することで、アクションの結果と後続の結果フィルターの実行をショートサーキットできます。 ショートサーキットする場合は、空の応答が生成されないように応答オブジェクトに記述します。 `IResultFilter.OnResultExecuting` で例外をスローすることで:

* アクション結果と後続フィルターの実行が回避されます。
* 結果は成功ではなく、失敗として処理されます。

<xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter.OnResultExecuted*?displayProperty=fullName> メソッドが実行されると:

* 応答はおそらくクライアントに送信されており、変更できません。
* 例外がスローされた場合、応答本文は送信されていません。

<!-- Review preceding "If an exception was thrown: Original 
When the OnResultExecuted method runs, the response has likely been sent to the client and cannot be changed further (unless an exception was thrown).

SHould that be , 
If an exception was thrown **IN THE RESULT FILTER**, the response body is not sent.

 -->

別のフィルターによってアクションの結果の実行がショートサーキットされた場合、`ResultExecutedContext.Canceled` は `true` に設定されます。

アクションの結果または後続の結果フィルターが例外をスローした場合、`ResultExecutedContext.Exception` は null 以外の値に設定されます。 `Exception` を null に設定すると、例外を効果的に処理でき、パイプラインの後方で ASP.NET Core によって例外が再スローされることを防げます。 結果フィルターの例外を処理するとき、応答にデータを書き込む目的で信頼できる方法はありません。 アクションの結果により例外がスローされるとき、ヘッダーがクライアントにフラッシュされている場合、エラー コードを送信する目的で信頼できるメカニズムはありません。

<xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter> の場合、<xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutionDelegate> で `await next` を呼び出すと、後続のすべての結果フィルターとアクションの結果が実行されます。 ショートサーキットするには、[ResultExecutingContext.Cancel](xref:Microsoft.AspNetCore.Mvc.Filters.ResultExecutingContext.Cancel) を `true` に設定し、`ResultExecutionDelegate` を呼び出しません。

[!code-csharp[](./filters/sample/FiltersSample/Filters/MyAsyncResponseFilter.cs?name=snippet)]

このフレームワークからは、サブクラス化できる抽象 `ResultFilterAttribute` が与えられます。 前に示した [AddHeaderAttribute](#add-header-attribute) クラスは、結果フィルター属性の一例です。

### <a name="ialwaysrunresultfilter-and-iasyncalwaysrunresultfilter"></a>IAlwaysRunResultFilter および IAsyncAlwaysRunResultFilter

<xref:Microsoft.AspNetCore.Mvc.Filters.IAlwaysRunResultFilter> および <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncAlwaysRunResultFilter> インターフェイスでは、すべてのアクションの結果に対して実行される <xref:Microsoft.AspNetCore.Mvc.Filters.IResultFilter> の実装が宣言されます。 次の場合を除き、フィルターはすべてのアクションの結果に適用されます。

* <xref:Microsoft.AspNetCore.Mvc.Filters.IExceptionFilter> または <xref:Microsoft.AspNetCore.Mvc.Filters.IAuthorizationFilter> が適用され、応答がショートサーキットされます。
* アクションの結果を生成することで、例外フィルターによって例外が処理されます。

`IExceptionFilter` と `IAuthorizationFilter` 以外のフィルターによって `IAlwaysRunResultFilter` と `IAsyncAlwaysRunResultFilter` がショートサーキットされることはありません。

たとえば、次のフィルターは常に実行され、コンテンツ ネゴシエーションが失敗した場合に "*422 処理不可エンティティ*" 状態コードを使ってアクションの結果 (<xref:Microsoft.AspNetCore.Mvc.ObjectResult>) を設定します。

[!code-csharp[](./filters/sample/FiltersSample/Filters/UnprocessableResultFilter.cs?name=snippet)]

### <a name="ifilterfactory"></a>IFilterFactory

<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> は、<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> を実装します。 そのため、`IFilterFactory` インスタンスはフィルター パイプライン内の任意の場所で `IFilterMetadata` インスタンスとして使用できます。 ランタイムでは、フィルターを呼び出す準備をする際、`IFilterFactory` へのキャストが試行されます。 そのキャストが成功した場合、呼び出される `IFilterMetadata` インスタンスを作成するために <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance*> メソッドが呼び出されます。 これにより、アプリの起動時に正確なフィルター パイプラインを明示的に設定する必要がないため、柔軟なデザインが可能になります。

フィルターを作成するための別の方法として、カスタムの属性の実装で `IFilterFactory` を実装できます。

[!code-csharp[](./filters/sample/FiltersSample/Filters/AddHeaderWithFactoryAttribute.cs?name=snippet_IFilterFactory&highlight=1,4,5,6,7)]

[ダウンロード サンプル](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/controllers/filters/sample)を実行することで、前のコードをテストできます。

* F12 開発者ツールを呼び出します。
* `https://localhost:5001/Sample/HeaderWithFactory` に移動します。

F12 開発者ツールでは、サンプル コードによって追加された次の応答ヘッダーが表示されます。

* **author:** `Joe Smith`
* **globaladdheader:** `Result filter added to MvcOptions.Filters`
* **internal:** `My header`

上記のコードでは、**internal:** `My header` という応答ヘッダーが作成されます。

### <a name="ifilterfactory-implemented-on-an-attribute"></a>属性に実装された IFilterFactory

<!-- Review 
This section needs to be rewritten.
What's a non-named attribute?
-->

`IFilterFactory` を実装するフィルターは次のようなフィルターに便利です。

* パラメーターの引き渡しを必要としません。
* DI で満たす必要があるコンストラクター依存関係があります。

<xref:Microsoft.AspNetCore.Mvc.TypeFilterAttribute> は、<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory> を実装します。 `IFilterFactory` は、<xref:Microsoft.AspNetCore.Mvc.Filters.IFilterMetadata> インスタンスを作成するために <xref:Microsoft.AspNetCore.Mvc.Filters.IFilterFactory.CreateInstance*> メソッドを公開します。 `CreateInstance` により、サービス コンテナー (DI) から指定の型が読み込まれます。

[!code-csharp[](./filters/sample/FiltersSample/Filters/SampleActionFilterAttribute.cs?name=snippet_TypeFilterAttribute&highlight=1,3,7)]

次のコードでは、`[SampleActionFilter]` を適用する 3 つの手法を確認できます。

[!code-csharp[](./filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet&highlight=1)]

上記のコードでは、`SampleActionFilter` を適用する方法としては、`[SampleActionFilter]` でメソッドを装飾する方法が推奨されます。

## <a name="using-middleware-in-the-filter-pipeline"></a>フィルター パイプラインでのミドルウェアの使用

リソース フィルターは、パイプラインの後方で登場するすべての実行を囲む点で、[ミドルウェア](xref:fundamentals/middleware/index)のように機能します。 ただし、フィルターは ASP.NET Core ランタイムの一部である点がミドルウェアとは異なります。つまり、フィルターには ASP.NET Core コンテキストとコンストラクトへのアクセスがあります。

ミドルウェアをフィルターとして使用するには、フィルター パイプラインに挿入するミドルウェアを指定する `Configure` メソッドを使用して型を作成します。 ローカリゼーション ミドルウェアを使用して要求の現在のカルチャを確立する例を次に示します。

[!code-csharp[](./filters/sample/FiltersSample/Filters/LocalizationPipeline.cs?name=snippet_MiddlewareFilter&highlight=3,21)]

<xref:Microsoft.AspNetCore.Mvc.MiddlewareFilterAttribute> を使用し、ミドルウェアを実行します。

[!code-csharp[](./filters/sample/FiltersSample/Controllers/HomeController.cs?name=snippet_MiddlewareFilter&highlight=2)]

ミドルウェア フィルターは、フィルター パイプラインのリソース フィルターと同じステージ (モデル バインドの前、残りのパイプラインの後) で実行されます。

## <a name="next-actions"></a>次の操作

* [Razor Pages のフィルター メソッド](xref:razor-pages/filter)に関するページをご覧ください
* フィルターを試すには、[GitHub のサンプルをダウンロードして、テストおよび変更を行います](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/mvc/controllers/filters/sample)。
