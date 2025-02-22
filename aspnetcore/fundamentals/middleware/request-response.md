---
title: ASP.NET Core での要求と応答の操作
author: jkotalik
description: ASP.NET Core で要求本文の読み取りと応答本文の書き込みを行う方法について説明します。
monikerRange: '>= aspnetcore-3.0'
ms.author: jukotali
ms.custom: mvc
ms.date: 02/26/2019
uid: fundamentals/middleware/request-response
ms.openlocfilehash: c9f6509738ef6290666a58268fbb0584913db9d6
ms.sourcegitcommit: 357a7120632b20465801c093e4e5bd4a315496a8
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 07/08/2019
ms.locfileid: "67649225"
---
# <a name="request-and-response-operations-in-aspnet-core"></a>ASP.NET Core での要求と応答の操作

作成者: [Justin Kotalik](https://github.com/jkotalik)

この記事では、要求本文からの読み取りと、応答本文への書き込みを行う方法について説明します。 ミドルウェアを記述している場合、これらの操作に関するコードを記述する必要がある場合があります。 それ以外の場合は、MVC と Razor Pages によって操作が処理されるため、通常はこのコードを記述する必要はありません。

ASP.NET Core 3.0 には、要求と応答の本文に関する 2 つの抽象化があります: <xref:System.IO.Stream> と <xref:System.IO.Pipelines.Pipe> です。 要求の読み取りでは、[HttpRequest.Body](xref:Microsoft.AspNetCore.Http.HttpRequest.Body) は <xref:System.IO.Stream> に、`HttpRequest.BodyPipe` は <xref:System.IO.Pipelines.PipeReader> になります。 応答の書き込みでは、[HttpResponse.Body](xref:Microsoft.AspNetCore.Http.HttpResponse.Body) は `HttpResponse.BodyPipe` は <xref:System.IO.Pipelines.PipeWriter> になります。

ストリームよりもパイプラインをお勧めします。 一部の単純な操作ではストリームの方が使いやすい場合がありますが、パイプラインの方がパフォーマンスに優れていて、ほとんどのシナリオでより簡単に使えます。 ASP.NET Core では、3.0 から、内部的にストリームではなくパイプラインが使われ始めています。 その例は次のとおりです。

- `FormReader`
- `TextReader`
- `TextWriter`
- `HttpResponse.WriteAsync`

ストリームはなくなりません。 それらは .NET 全体で使われ続けます。また、ストリームの種類の多くには対応するパイプラインがありません (`FileStreams` や `ResponseCompression` など)。

## <a name="stream-examples"></a>ストリームの例

要求本文全体を、改行で分割した文字列のリストとして読み取るミドルウェアを作成したいとします。 単純なストリームの実装は、次の例のようになります。

[!code-csharp[](request-response/samples/3.x/RequestResponseSample/Startup.cs?name=GetListOfStringsFromStream)]

このコードで動作しますが、いくつか問題があります。

- 例では、`StringBuilder` に追加する前に、すぐに破棄される別の文字列 (`encodedString`) を作成しています。 このプロセスはストリーム内のすべてのバイトに対して実行されるため、要求本文全体のサイズよりも余分にメモリの割り当てが発生します。
- 例では、改行で分割する前に文字列全体を読み取っています。 バイト配列内で改行をチェックした方がより効率的です。

これらの問題をいくつか修正した例を次に示します。

[!code-csharp[](request-response/samples/3.x/RequestResponseSample/Startup.cs?name=GetListOfStringsFromStreamMoreEfficient)]

この例の場合:

- 要求本文全体を `StringBuilder` にバッファーすることがありません (改行文字がない場合を除く)。
- 文字列の `Split` を呼び出しません。

ただし、まだいくつかの問題が残っています。

- 改行文字がスパースだった場合、要求本文の多くが文字列にバッファーされます。
- 依然として文字列 (`remainingString`) を作成し、それらを文字列バッファーに追加しているため、余分な割り当てが発生します。

これらの問題は修正可能ですが、わずかな改善でコードがますます複雑になっていきます。 パイプラインには、これらの問題を、コードの複雑さを最小限に抑えて解決する方法が用意されています。

## <a name="pipelines"></a>パイプライン

同じシナリオを、`PipeReader` を使って処理する方法を次の例に示します。

[!code-csharp[](request-response/samples/3.x/RequestResponseSample/Startup.cs?name=GetListOfStringFromPipe)]

この例では、ストリームの実装で発生していた多くの問題が修正されています。

- 使われていないバイトを `PipeReader` で処理するため、文字列バッファーが不要です。
- エンコードされた文字列は、返される文字列のリストに直接追加されます。
- 文字列で使うメモリを除いて、文字列の作成に関する割り当てが発生しません (`ToArray()` の呼び出しを除く)。

## <a name="adapters"></a>アダプター

これで、`Body` と `BodyPipe` プロパティの両方を `HttpRequest` と `HttpResponse` に対して使用できるようになりました。`Body` を別のストリームに設定した場合はどうなるでしょうか。 3\.0 では、新しいアダプターのセットにより、各種類を別のものに自動的に適応させます。 たとえば、`HttpRequest.Body` を新しいストリームに設定した場合、`HttpRequest.BodyPipe` は自動的に、`HttpRequest.Body` をラップする新しい `PipeReader` に設定されます。 `BodyPipe` プロパティを設定する場合にも同じ動作が適用されます。 `HttpResponse.BodyPipe` を新しい `PipeWriter` に設定した場合、`HttpResponse.Body` は自動的に、`HttpResponse.BodyPipe` をラップする新しいストリームに設定されます。

## <a name="startasync"></a>StartAsync

`HttpResponse.StartAsync` は 3.0 の新機能です。 これは、ヘッダーが変更不可能であり、また `OnStarting` コールバックを実行することを示すために使います。 3\.0-preview3 では、`HttpRequest.BodyPipe` を使う前に `StartAsync` を呼び出す必要がありました。今後のリリースでは、これは推奨事項になる予定です。 サーバーとして Kestrel を使う場合、`PipeReader` を使う前に StartAsync を呼び出すことで、`GetMemory` によって返されるメモリが、外部バッファーではなく Kestrel の内部 <xref:System.IO.Pipelines.Pipe> に属するよう保証できます。

## <a name="additional-resources"></a>その他の技術情報

- [System.IO.Pipelines の概要](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/)
- <xref:fundamentals/middleware/write>
