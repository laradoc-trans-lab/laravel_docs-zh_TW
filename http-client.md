# HTTP Client

- [簡介](#introduction)
- [發出請求](#making-requests)
    - [請求資料](#request-data)
    - [標頭](#headers)
    - [認證](#authentication)
    - [逾時](#timeout)
    - [重試](#retries)
    - [錯誤處理](#error-handling)
    - [Guzzle Middleware](#guzzle-middleware)
    - [Guzzle 選項](#guzzle-options)
- [並行請求](#concurrent-requests)
- [巨集 (Macros)](#macros)
- [測試](#testing)
    - [偽造回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止意外請求](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 針對 [Guzzle HTTP client](http://docs.guzzlephp.org/en/stable/) 提供了一個表達性強且簡潔的 API，讓您可以快速發出對外的 HTTP 請求，以便與其他 Web 應用程式進行通訊。Laravel 對 Guzzle 的封裝專注於其最常見的用途，並提供絕佳的開發者體驗。

<a name="making-requests"></a>
## 發出請求

要發出請求，您可以使用 `Http` Facade 提供的 `head`、`get`、`post`、`put`、`patch` 和 `delete` 方法。首先，讓我們看看如何向另一個 URL 發出基本的 `GET` 請求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('http://example.com');
```

`get` 方法會回傳一個 `Illuminate\Http\Client\Response` 實例，該實例提供了多種方法可用於檢查回應：

```php
$response->body() : string;
$response->json($key = null, $default = null) : mixed;
$response->object() : object;
$response->collect($key = null) : Illuminate\Support\Collection;
$response->resource() : resource;
$response->status() : int;
$response->successful() : bool;
$response->redirect(): bool;
$response->failed() : bool;
$response->clientError() : bool;
$response->header($header) : string;
$response->headers() : array;
```

`Illuminate\Http\Client\Response` 物件也實作了 PHP 的 `ArrayAccess` 介面，讓您可以直接透過回應物件存取 JSON 回應資料：

```php
return Http::get('http://example.com/users/1')['name'];
```

除了上面列出的回應方法外，還可以使用以下方法來判斷回應是否具有特定的狀態碼：

```php
$response->ok() : bool;                  // 200 OK
$response->created() : bool;             // 201 Created
$response->accepted() : bool;            // 202 Accepted
$response->noContent() : bool;           // 204 No Content
$response->movedPermanently() : bool;    // 301 Moved Permanently
$response->found() : bool;               // 302 Found
$response->badRequest() : bool;          // 400 Bad Request
$response->unauthorized() : bool;        // 401 Unauthorized
$response->paymentRequired() : bool;     // 402 Payment Required
$response->forbidden() : bool;           // 403 Forbidden
$response->notFound() : bool;            // 404 Not Found
$response->requestTimeout() : bool;      // 408 Request Timeout
$response->conflict() : bool;            // 409 Conflict
$response->unprocessableEntity() : bool; // 422 Unprocessable Entity
$response->tooManyRequests() : bool;     // 429 Too Many Requests
$response->serverError() : bool;         // 500 Internal Server Error
```

<a name="uri-templates"></a>
#### URI 範本

HTTP client 也允許您使用 [URI 範本規範](https://www.rfc-editor.org/rfc/rfc6570)來建構請求 URL。要定義可由 URI 範本展開的 URL 參數，您可以使用 `withUrlParameters` 方法：

```php
Http::withUrlParameters([
    'endpoint' => 'https://laravel.com',
    'page' => 'docs',
    'version' => '12.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```

<a name="dumping-requests"></a>
#### 傾印請求

如果您想在發送請求實例之前傾印它並終止腳本執行，您可以將 `dd` 方法添加到請求定義的開頭：

```php
return Http::dd()->get('http://example.com');
```

<a name="request-data"></a>
### 請求資料

當然，在發出 `POST`、`PUT` 和 `PATCH` 請求時，通常會隨請求發送額外的資料，因此這些方法接受一個資料陣列作為它們的第二個參數。預設情況下，資料將使用 `application/json` 內容類型發送：

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```

<a name="get-request-query-parameters"></a>
#### GET 請求的查詢參數

發出 `GET` 請求時，您可以直接將查詢字串附加到 URL，或者將鍵/值對陣列作為第二個參數傳遞給 `get` 方法：

```php
$response = Http::get('http://example.com/users', [
    'name' => 'Taylor',
    'page' => 1,
]);
```

或者，可以使用 `withQueryParameters` 方法：

```php
Http::retry(3, 100)->withQueryParameters([
    'name' => 'Taylor',
    'page' => 1,
])->get('http://example.com/users');
```

<a name="sending-form-url-encoded-requests"></a>
#### 發送表單 URL 編碼請求

如果您想使用 `application/x-www-form-urlencoded` 內容類型發送資料，您應該在發出請求之前呼叫 `asForm` 方法：

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```

<a name="sending-a-raw-request-body"></a>
#### 發送原始請求主體

如果您想在發出請求時提供原始請求主體，可以使用 `withBody` 方法。內容類型可以透過方法的第二個參數提供：

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```

<a name="multi-part-requests"></a>
#### 多部分請求

如果您想以多部分請求的形式發送檔案，您應該在發出請求之前呼叫 `attach` 方法。此方法接受檔案名稱及其內容。如果需要，您可以提供第三個參數作為檔案的檔名，而第四個參數可用於提供與檔案相關的標頭：

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
)->post('http://example.com/attachments');
```

除了傳遞檔案的原始內容，您也可以傳遞一個串流資源：

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```

<a name="headers"></a>
### 標頭

可以使用 `withHeaders` 方法將標頭添加到請求中。此 `withHeaders` 方法接受一個鍵/值對陣列：

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

您可以使用 `accept` 方法來指定您的應用程式預期在回應中接收的內容類型：

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

為了方便起見，您可以使用 `acceptJson` 方法快速指定您的應用程式預期在回應中接收 `application/json` 內容類型：

```php
$response = Http::acceptJson()->get('http://example.com/users');
```

`withHeaders` 方法會將新的標頭合併到請求的現有標頭中。如果需要，您可以使用 `replaceHeaders` 方法完全替換所有標頭：

```php
$response = Http::withHeaders([
    'X-Original' => 'foo',
])->replaceHeaders([
    'X-Replacement' => 'bar',
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

<a name="authentication"></a>
### 認證

您可以分別使用 `withBasicAuth` 和 `withDigestAuth` 方法指定基本和摘要認證憑證：

```php
// Basic authentication...
$response = Http::withBasicAuth('taylor @laravel.com', 'secret')->post(/* ... */);

// Digest authentication...
$response = Http::withDigestAuth('taylor @laravel.com', 'secret')->post(/* ... */);
```

<a name="bearer-tokens"></a>
#### Bearer Token

如果您想快速將 Bearer Token 添加到請求的 `Authorization` 標頭中，您可以使用 `withToken` 方法：

```php
$response = Http::withToken('token')->post(/* ... */);
```

<a name="timeout"></a>
### 逾時

`timeout` 方法可用於指定等待回應的最長時間（秒）。預設情況下，HTTP client 將在 30 秒後逾時：

```php
$response = Http::timeout(3)->get(/* ... */);
```

如果超過給定的逾時時間，將會拋出 `Illuminate\Http\Client\ConnectionException` 實例。

您可以使用 `connectTimeout` 方法指定嘗試連接伺服器時等待的最長時間（秒）。預設為 10 秒：

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>
### 重試

如果您希望 HTTP client 在發生客戶端或伺服器錯誤時自動重試請求，您可以使用 `retry` 方法。`retry` 方法接受請求應嘗試的最大次數，以及 Laravel 在每次嘗試之間應等待的毫秒數：

```php
$response = Http::retry(3, 100)->post(/* ... */);
```

如果您想手動計算每次嘗試之間休眠的毫秒數，您可以將一個閉包作為第二個參數傳遞給 `retry` 方法：

```php
use Exception;

$response = Http::retry(3, function (int $attempt, Exception $exception) {
    return $attempt * 100;
})->post(/* ... */);
```

為了方便起見，您也可以將一個陣列作為第一個參數傳遞給 `retry` 方法。這個陣列將用於確定後續嘗試之間休眠的毫秒數：

```php
$response = Http::retry([100, 200])->post(/* ... */);
```

如果需要，您可以將第三個參數傳遞給 `retry` 方法。第三個參數應該是一個可呼叫 (callable) 的函式，它決定是否應該實際嘗試重試。例如，您可能只希望在初始請求遇到 `ConnectionException` 時才重試請求：

```php
use Exception;
use Illuminate\Http\Client\PendingRequest;

$response = Http::retry(3, 100, function (Exception $exception, PendingRequest $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

如果請求嘗試失敗，您可能希望在進行新的嘗試之前對請求進行更改。您可以透過修改提供給 `retry` 方法的可呼叫函式中的請求參數來實現這一點。例如，如果第一次嘗試返回認證錯誤，您可能希望使用新的授權 Token 重試請求：

```php
use Exception;
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Http\Client\RequestException;

$response = Http::withToken($this->getToken())->retry(2, 0, function (Exception $exception, PendingRequest $request) {
    if (! $exception instanceof RequestException || $exception->response->status() !== 401) {
        return false;
    }

    $request->withToken($this->getNewToken());

    return true;
})->post(/* ... */);
```

如果所有請求都失敗，將會拋出 `Illuminate\Http\Client\RequestException` 實例。如果您想禁用此行為，您可以提供一個 `throw` 參數，其值為 `false`。禁用後，在所有重試嘗試後，將返回客戶端收到的最後一個回應：

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> [!WARNING]
> 如果所有請求都因連接問題而失敗，即使 `throw` 參數設定為 `false`，仍會拋出 `Illuminate\Http\Client\ConnectionException`。

<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP client 封裝不會在客戶端或伺服器錯誤（來自伺服器的 `400` 和 `500` 級別回應）時拋出例外。您可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否返回了這些錯誤之一：

```php
// 判斷狀態碼是否 >= 200 且 < 300...
$response->successful();

// 判斷狀態碼是否 >= 400...
$response->failed();

// 判斷回應是否具有 400 級別的狀態碼...
$response->clientError();

// 判斷回應是否具有 500 級別的狀態碼...
$response->serverError();

// 如果發生客戶端或伺服器錯誤，立即執行給定的回呼函式...
$response->onError(callable $callback);
```

<a name="throwing-exceptions"></a>
#### 拋出例外

如果您有一個回應實例，並且希望在回應狀態碼表示客戶端或伺服器錯誤時拋出 `Illuminate\Http\Client\RequestException` 實例，您可以使用 `throw` 或 `throwIf` 方法：

```php
use Illuminate\Http\Client\Response;

$response = Http::post(/* ... */);

// 如果發生客戶端或伺服器錯誤，則拋出例外...
$response->throw();

// 如果發生錯誤且給定條件為 true，則拋出例外...
$response->throwIf($condition);

// 如果發生錯誤且給定閉包解析為 true，則拋出例外...
$response->throwIf(fn (Response $response) => true);

// 如果發生錯誤且給定條件為 false，則拋出例外...
$response->throwUnless($condition);

// 如果發生錯誤且給定閉包解析為 false，則拋出例外...
$response->throwUnless(fn (Response $response) => false);

// 如果回應具有特定的狀態碼，則拋出例外...
$response->throwIfStatus(403);

// 除非回應具有特定的狀態碼，否則拋出例外...
$response->throwUnlessStatus(200);

return $response['user']['id'];
```

`Illuminate\Http\Client\RequestException` 實例有一個公共的 `$response` 屬性，它將允許您檢查返回的回應。

如果沒有發生錯誤，`throw` 方法會返回回應實例，讓您可以將其他操作鏈接到 `throw` 方法：

```php
return Http::post(/* ... */)->throw()->json();
```

如果您想在拋出例外之前執行一些額外的邏輯，您可以將一個閉包傳遞給 `throw` 方法。例外將在閉包被調用後自動拋出，因此您無需在閉包中重新拋出例外：

```php
use Illuminate\Http\Client\Response;
use Illuminate\Http\Client\RequestException;

return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
    // ...
})->json();
```

預設情況下，`RequestException` 訊息在記錄或報告時會被截斷為 120 個字元。要自訂或禁用此行為，您可以在 `bootstrap/app.php` 檔案中設定應用程式的例外處理行為時，使用 `truncateRequestExceptionsAt` 和 `dontTruncateRequestExceptions` 方法：

```php
use Illuminate\Foundation\Configuration\Exceptions;

->withExceptions(function (Exceptions $exceptions): void {
    // 將請求例外訊息截斷為 240 個字元...
    $exceptions->truncateRequestExceptionsAt(240);

    // 禁用請求例外訊息截斷...
    $exceptions->dontTruncateRequestExceptions();
})
```

或者，您可以使用 `truncateExceptionsAt` 方法按請求自訂例外截斷行為：

```php
return Http::truncateExceptionsAt(240)->post(/* ... */);
```

<a name="guzzle-middleware"></a>
### Guzzle Middleware

由於 Laravel 的 HTTP client 是由 Guzzle 提供支援的，因此您可以利用 [Guzzle Middleware](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來操作發出的請求或檢查收到的回應。要操作發出的請求，請透過 `withRequestMiddleware` 方法註冊一個 Guzzle Middleware：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withRequestMiddleware(
    function (RequestInterface $request) {
        return $request->withHeader('X-Example', 'Value');
    }
)->get('http://example.com');
```

同樣地，您可以透過 `withResponseMiddleware` 方法註冊一個 Middleware 來檢查收到的 HTTP 回應：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\ResponseInterface;

$response = Http::withResponseMiddleware(
    function (ResponseInterface $response) {
        $header = $response->getHeader('X-Example');

        // ...

        return $response;
    }
)->get('http://example.com');
```

<a name="global-middleware"></a>
#### 全域 Middleware

有時，您可能希望註冊一個適用於每個發出請求和收到回應的 Middleware。為此，您可以使用 `globalRequestMiddleware` 和 `globalResponseMiddleware` 方法。通常，這些方法應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中調用：

```php
use Illuminate\Support\Facades\Http;

Http::globalRequestMiddleware(fn ($request) => $request->withHeader(
    'User-Agent', 'Example Application/1.0'
));

Http::globalResponseMiddleware(fn ($response) => $response->withHeader(
    'X-Finished-At', now()->toDateTimeString()
));
```

<a name="guzzle-options"></a>
### Guzzle 選項

您可以使用 `withOptions` 方法為發出的請求指定額外的 [Guzzle 請求選項](http://docs.guzzlephp.org/en/stable/request-options.html)。`withOptions` 方法接受一個鍵/值對陣列：

```php
$response = Http::withOptions([
    'debug' => true,
])->get('http://example.com/users');
```

<a name="global-options"></a>
#### 全域選項

要為每個發出的請求設定預設選項，您可以使用 `globalOptions` 方法。通常，此方法應在應用程式的 `AppServiceProvider` 的 `boot` 方法中調用：

```php
use Illuminate\Support\Facades\Http;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Http::globalOptions([
        'allow_redirects' => false,
    ]);
}
```

<a name="concurrent-requests"></a>
## 並行請求

有時，您可能希望並行發出多個 HTTP 請求。換句話說，您希望同時發送多個請求，而不是依序發出請求。這在與緩慢的 HTTP API 互動時可以顯著提高效能。

幸運的是，您可以使用 `pool` 方法來實現這一點。`pool` 方法接受一個閉包，該閉包會收到一個 `Illuminate\Http\Client\Pool` 實例，讓您可以輕鬆地將請求添加到請求池中以進行分派：

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$responses = Http::pool(fn (Pool $pool) => [
    $pool->get('http://localhost/first'),
    $pool->get('http://localhost/second'),
    $pool->get('http://localhost/third'),
]);

return $responses[0]->ok() &&
       $responses[1]->ok() &&
       $responses[2]->ok();
```

如您所見，每個回應實例都可以根據其添加到池中的順序進行存取。如果您願意，您可以使用 `as` 方法命名請求，這允許您透過名稱存取相應的回應：

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$responses = Http::pool(fn (Pool $pool) => [
    $pool->as('first')->get('http://localhost/first'),
    $pool->as('second')->get('http://localhost/second'),
    $pool->as('third')->get('http://localhost/third'),
]);

return $responses['first']->ok();
```

<a name="customizing-concurrent-requests"></a>
#### 自訂並行請求

`pool` 方法不能與其他 HTTP client 方法（例如 `withHeaders` 或 `middleware` 方法）鏈接使用。如果您想將自訂標頭或 Middleware 應用於池化請求，您應該在池中的每個請求上設定這些選項：

```php
use Illuminate\Http\Client\Pool;
use Illuminate\Support\Facades\Http;

$headers = [
    'X-Example' => 'example',
];

$responses = Http::pool(fn (Pool $pool) => [
    $pool->withHeaders($headers)->get('http://laravel.test/test'),
    $pool->withHeaders($headers)->get('http://laravel.test/test'),
    $pool->withHeaders($headers)->get('http://laravel.test/test'),
]);
```

<a name="macros"></a>
## 巨集 (Macros)

Laravel HTTP client 允許您定義「巨集」，這可以作為一種流暢、表達性強的機制，用於在應用程式中與服務互動時設定常見的請求路徑和標頭。首先，您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義巨集：

```php
use Illuminate\Support\Facades\Http;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Http::macro('github', function () {
        return Http::withHeaders([
            'X-Example' => 'example',
        ])->baseUrl('https://github.com');
    });
}
```

一旦您的巨集設定完成，您就可以在應用程式中的任何地方調用它，以使用指定的設定建立一個待處理的請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了功能，可幫助您輕鬆且表達性強地編寫測試，Laravel 的 HTTP client 也不例外。`Http` Facade 的 `fake` 方法允許您指示 HTTP client 在發出請求時返回存根/虛擬回應。

<a name="faking-responses"></a>
### 偽造回應

例如，要指示 HTTP client 為每個請求返回空的 `200` 狀態碼回應，您可以不帶任何參數呼叫 `fake` 方法：

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```

<a name="faking-specific-urls"></a>
#### 偽造特定 URL

或者，您可以將一個陣列傳遞給 `fake` 方法。陣列的鍵應該代表您希望偽造的 URL 模式及其相關回應。`*` 字元可以用作萬用字元。您可以使用 `Http` Facade 的 `response` 方法為這些端點建構存根/偽造回應：

```php
Http::fake([
    // 為 GitHub 端點偽造 JSON 回應...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // 為 Google 端點偽造字串回應...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

對未偽造的 URL 發出的任何請求都將實際執行。如果您想指定一個後備 URL 模式，該模式將偽造所有不匹配的 URL，您可以使用單個 `*` 字元：

```php
Http::fake([
    // 為 GitHub 端點偽造 JSON 回應...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // 為所有其他端點偽造字串回應...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

為了方便起見，可以透過提供字串、陣列或整數作為回應來生成簡單的字串、JSON 和空回應：

```php
Http::fake([
    'google.com/*' => 'Hello World',
    'github.com/*' => ['foo' => 'bar'],
    'chatgpt.com/*' => 200,
]);
```

<a name="faking-connection-exceptions"></a>
#### 偽造例外

有時您可能需要測試應用程式的行為，如果 HTTP client 在嘗試發出請求時遇到 `Illuminate\Http\Client\ConnectionException`。您可以使用 `failedConnection` 方法指示 HTTP client 拋出連接例外：

```php
Http::fake([
    'github.com/*' => Http::failedConnection(),
]);
```

要測試應用程式在拋出 `Illuminate\Http\Client\RequestException` 時的行為，您可以使用 `failedRequest` 方法：

```php
Http::fake([
    'github.com/*' => Http::failedRequest(['code' => 'not_found'], 404),
]);
```

<a name="faking-response-sequences"></a>
#### 偽造回應序列

有時您可能需要指定單個 URL 應按特定順序返回一系列偽造回應。您可以使用 `Http::sequence` 方法來建構回應：

```php
Http::fake([
    // 為 GitHub 端點偽造一系列回應...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

當回應序列中的所有回應都被消耗完畢後，任何進一步的請求都將導致回應序列拋出例外。如果您想指定當序列為空時應返回的預設回應，您可以使用 `whenEmpty` 方法：

```php
Http::fake([
    // 為 GitHub 端點偽造一系列回應...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

如果您想偽造一系列回應，但不需要指定應偽造的特定 URL 模式，您可以使用 `Http::fakeSequence` 方法：

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```

<a name="fake-callback"></a>
#### 偽造回呼函式

如果您需要更複雜的邏輯來確定某些端點要返回什麼回應，您可以將一個閉包傳遞給 `fake` 方法。這個閉包將收到一個 `Illuminate\Http\Client\Request` 實例，並且應該返回一個回應實例。在您的閉包中，您可以執行任何必要的邏輯來確定要返回的回應類型：

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```

<a name="inspecting-requests"></a>
### 檢查請求

在偽造回應時，您可能偶爾會希望檢查客戶端收到的請求，以確保您的應用程式正在發送正確的資料或標頭。您可以在呼叫 `Http::fake` 後呼叫 `Http::assertSent` 方法來實現這一點。

`assertSent` 方法接受一個閉包，該閉包將收到一個 `Illuminate\Http\Client\Request` 實例，並且應該返回一個布林值，指示請求是否符合您的預期。為了讓測試通過，至少必須發出一個符合給定預期的請求：

```php
use Illuminate\Http\Client\Request;
use Illuminate\Support\Facades\Http;

Http::fake();

Http::withHeaders([
    'X-First' => 'foo',
])->post('http://example.com/users', [
    'name' => 'Taylor',
    'role' => 'Developer',
]);

Http::assertSent(function (Request $request) {
    return $request->hasHeader('X-First', 'foo') &&
           $request->url() == 'http://example.com/users' &&
           $request['name'] == 'Taylor' &&
           $request['role'] == 'Developer';
});
```

如果需要，您可以使用 `assertNotSent` 方法斷言未發送特定請求：

```php
use Illuminate\Http\Client\Request;
use Illuminate\Support\Facades\Http;

Http::fake();

Http::post('http://example.com/users', [
    'name' => 'Taylor',
    'role' => 'Developer',
]);

Http::assertNotSent(function (Request $request) {
    return $request->url() === 'http://example.com/posts';
});
```

您可以使用 `assertSentCount` 方法斷言在測試期間「發送」了多少個請求：

```php
Http::fake();

Http::assertSentCount(5);
```

或者，您可以使用 `assertNothingSent` 方法斷言在測試期間沒有發送任何請求：

```php
Http::fake();

Http::assertNothingSent();
```

<a name="recording-requests-and-responses"></a>
#### 記錄請求/回應

您可以使用 `recorded` 方法收集所有請求及其對應的回應。`recorded` 方法返回一個陣列集合，其中包含 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例：

```php
Http::fake([
    'https://laravel.com' => Http::response(status: 500),
    'https://nova.laravel.com/' => Http::response(),
]);

Http::get('https://laravel.com');
Http::get('https://nova.laravel.com/');

$recorded = Http::recorded();

[$request, $response] = $recorded[0];
```

此外，`recorded` 方法接受一個閉包，該閉包將收到 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例，並且可以用於根據您的預期過濾請求/回應對：

```php
use Illuminate\Http\Client\Request;
use Illuminate\Http\Client\Response;

Http::fake([
    'https://laravel.com' => Http::response(status: 500),
    'https://nova.laravel.com/' => Http::response(),
]);

Http::get('https://laravel.com');
Http::get('https://nova.laravel.com/');

$recorded = Http::recorded(function (Request $request, Response $response) {
    return $request->url() !== 'https://laravel.com' &&
           $response->successful();
});
```

<a name="preventing-stray-requests"></a>
### 防止意外請求

如果您想確保透過 HTTP client 發送的所有請求在您的個別測試或整個測試套件中都已被偽造，您可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應偽造回應的請求都將拋出例外，而不是發出實際的 HTTP 請求：

```php
use Illuminate\Support\Facades\Http;

Http::preventStrayRequests();

Http::fake([
    'github.com/*' => Http::response('ok'),
]);

// 返回 "ok" 回應...
Http::get('https://github.com/laravel/framework');

// 拋出例外...
Http::get('https://laravel.com');
```

有時，您可能希望防止大多數意外請求，同時仍允許特定請求執行。為此，您可以將一個 URL 模式陣列傳遞給 `allowStrayRequests` 方法。任何與給定模式之一匹配的請求都將被允許，而所有其他請求將繼續拋出例外：

```php
use Illuminate\Support\Facades\Http;

Http::preventStrayRequests();

Http::allowStrayRequests([
    'http://127.0.0.1:5000/*',
]);

// 此請求被執行...
Http::get('http://127.0.0.1:5000/generate');

// 拋出例外...
Http::get('https://laravel.com');
```

<a name="events"></a>
## 事件

Laravel 在發送 HTTP 請求的過程中會觸發三個事件。`RequestSending` 事件在請求發送之前觸發，而 `ResponseReceived` 事件在收到給定請求的回應之後觸發。如果給定請求沒有收到回應，則觸發 `ConnectionFailed` 事件。

`RequestSending` 和 `ConnectionFailed` 事件都包含一個公共的 `$request` 屬性，您可以使用它來檢查 `Illuminate\Http\Client\Request` 實例。同樣地，`ResponseReceived` 事件包含一個 `$request` 屬性以及一個 `$response` 屬性，可用於檢查 `Illuminate\Http\Client\Response` 實例。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

```php
use Illuminate\Http\Client\Events\RequestSending;

class LogRequest
{
    /**
     * Handle the event.
     */
    public function handle(RequestSending $event): void
    {
        // $event->request ...
    }
}
```
