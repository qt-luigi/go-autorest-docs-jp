# 本リポジトリーについて

[Azure/go-autorest](https://github.com/Azure/go-autorest) のドキュメントを日本語に翻訳してみました。
Google翻訳を使用して翻訳した後に自分でチェックして修正しています。
翻訳時のバージョンは [v14.1.1](https://github.com/Azure/go-autorest/tree/v14.1.1) です。

# go-autorest

[![GoDoc](https://godoc.org/github.com/Azure/go-autorest/autorest?status.png)](https://godoc.org/github.com/Azure/go-autorest/autorest)
[![Build Status](https://dev.azure.com/azure-sdk/public/_apis/build/status/go/Azure.go-autorest?branchName=master)](https://dev.azure.com/azure-sdk/public/_build/latest?definitionId=625&branchName=master)
[![Go Report Card](https://goreportcard.com/badge/Azure/go-autorest)](https://goreportcard.com/report/Azure/go-autorest)

go-autorestパッケージは、 [Autorest](https://github.com/Azure/autorest.go) が生成したAPIクライアントパッケージで使用するHTTPリクエストクライアントを提供します。

Azure Active Directory (AAD) でテストされた認証クライアントも、 `github.com/Azure/go-autorest/autorest/adal` パッケージとしてこのリポジトリーで提供されています。
その名前にもかかわらず、このパッケージはAzure Go SDKの一部としてのみ維持され、[github.com/AzureAD](https://github.com/AzureAD) の他の"ADAL"ライブラリとは関係ありません。

## 概要

go-autorestパッケージは、複数のゴルーチンでの使用に適したHTTPリクエストパイプラインを実装し、 [Autorest](https://github.com/Azure/autorest.go) によって生成されたパッケージで使用される共有ルーチンを提供します。

このパッケージは、HTTPリクエストの送信と応答を、準備、送信、および応答の3つのフェーズに分割します。
典型的なパターンは次のとおりです:

```go
  req, err := Prepare(&http.Request{},
    token.WithAuthorization())

  resp, err := Send(req,
    WithLogging(logger),
    DoErrorIfStatusCode(http.StatusInternalServerError),
    DoCloseIfError(),
    DoRetryForAttempts(5, time.Second))

  err = Respond(resp,
		ByDiscardingBody(),
    ByClosing())
```

各フェーズは、デコレーターを使用して処理を変更または管理します。
デコレーターは、最初にデータを変更してから渡し、最初にデータを渡してから結果を変更するか、データの受け渡しをラップします（ロガーのように）。
デコレーターは指定された順序で実行されます。 たとえば、次のとおりです:

```go
  req, err := Prepare(&http.Request{},
    WithBaseURL("https://microsoft.com/"),
    WithPath("a"),
    WithPath("b"),
    WithPath("c"))
```

URLを次のように設定します。

```
  https://microsoft.com/a/b/c
```

PreparerとResponderは共有して再利用できます (基礎となるデコレーターが共有と再利用をサポートしている場合) 。
複数のゴルーチン間で共有される1つ以上のPreparerとResponder、および複数の送信ゴルーチン間で共有される単一のSenderを作成することにより、これらはすべて、入出力チャネルによって結合され、パフォーマンスの高い使用が得られます。

デコレーターは、渡された状態をクロージャー内に保持します (上記の例のパスコンポーネントなど) 。
そのような保留状態が適用される状況でのみ、PreparerとResponderを共有するように注意してください。
たとえば、固定値のセットからクエリー文字列を適用するPreparerを共有しても意味がない場合があります。
同様に、レスポンスボディを渡された構造体 (`ByUnmarshallingJson` など) に読み込むResponderを共有することは、おそらく正しくありません。

autorestオブジェクトおよびメソッドによって発生したエラーは、 `autorest.Error` インターフェースに準拠します。

詳細については、含まれている例を参照してください。
生成されたクライアントによるこのパッケージの推奨される使用法の詳細については、以下で説明するクライアントを参照してください。

## ヘルパー

### Swagger日付の処理

AutoRest（https://github.com/Azure/autorest/) を駆動するSwagger仕様 (https://swagger.io) は、日付と日付時刻の2つの日付形式を正確に定義しています。
github.com/Azure/go-autorest/autorest/date パッケージは、正しい解析とフォーマットを確実にするための派生time.Timeを提供します。

### 空の値の処理

JSONでは、欠損値は空の値とは意味が異なります。
これは、HTTPのPATCHメソッドを使用するサービスに特に当てはまります。
PATCHリクエストで送信されたJSONには、通常、変更する値のみが含まれています。
欠損値は変更されません。
したがって、開発者は、空の値を指定し、送信されたJSONから値を除外する手段の両方を必要とします。

GoのJSONパッケージ (`encoding/json`) は `omitempty` タグをサポートしています。
指定すると、レンダリングされたJSONから空の値が省略されます。
Goはすべての基本型 (文字列の""やintの0など) のデフォルト値を定義し、値を実際に空としてマークする手段を提供しないため、JSONパッケージはデフォルト値を空の意味として扱い、レンダリングされたJSONからそれらを省略します。
つまり、デフォルトのJSONパッケージでエンコードされたGoの基本型を使用して、サーバーで値をクリアするJSONを作成することはできません。

Goコミュニティ内の回避策は、JSONにマップする構造内の基本型の代わりに、基本型へのポインターを使用することです。
たとえば、 `string` 型の値の代わりに、回避策は `*string` を使用します。
これにより、空の値と変更されない値を区別できますが、基本型へのポインター (特に、定数のインライン値) を作成するには、追加の変数が必要です。
これは、例えば、

```go
  s := struct {
    S *string
  }{ S: &"foo" }
```
失敗しますが、これは

```go
  v := "foo"
  s := struct {
    S *string
  }{ S: &v }
```
で、成功します。

ポインターの使用を容易にするために、サブパッケージ `to` には、Swaggerアナログを持つGoの基本型のポインターとの間で変換を行うヘルパーが含まれています。
また、 `map[string]string` と `map[string]*string` の間で変換するヘルパーを提供し、JSONがキーに関連付けられた値をクリアする必要があることを指定できるようにします。
ヘルパーを使用すると、前の例は次のようになります

```go
  s := struct {
    S *string
  }{ S: to.StringPtr("foo") }
```

## インストール

```bash
go get github.com/Azure/go-autorest/autorest
go get github.com/Azure/go-autorest/autorest/azure
go get github.com/Azure/go-autorest/autorest/date
go get github.com/Azure/go-autorest/autorest/to
```

### Goモジュールでの使用

[v12.0.1](https://github.com/Azure/go-autorest/pull/386) では、このリポジトリーは次のモジュールを導入しました。

- autorest/adal
- autorest/azure/auth
- autorest/azure/cli
- autorest/date
- autorest/mocks
- autorest/to
- autorest/validation
- autorest
- logger
- tracing

累積的なSDKリリース全体 ( `v12.3.0` など) へのタグ付けは、まだモジュールに移行されていないこのリポジトリーの利用者をサポートするために引き続き有効です。

## ライセンス

LICENSEファイルを参照してください。

-----

このプロジェクトは、 [Microsoftオープンソース行動規範](https://opensource.microsoft.com/codeofconduct/) を採用しています。
詳細については、 [行動規範に関するFAQ](https://opensource.microsoft.com/codeofconduct/faq/) を参照するか、 [opencode@microsoft.com](mailto:opencode@microsoft.com) に質問やコメントを追加してください。
