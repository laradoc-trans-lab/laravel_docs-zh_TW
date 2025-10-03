# HTTP 客戶端

- [簡介](#introduction)
- [發出請求](#making-requests)
    - [請求資料](#request-data)
    - [標頭](#headers)
    - [認證](#authentication)
    - [逾時](#timeout)
    - [重試](#retries)
    - [錯誤處理](#error-handling)
    - [Guzzle 中介層](#guzzle-middleware)
    - [Guzzle 選項](#guzzle-options)
- [併發請求](#concurrent-requests)
- [巨集](#macros)
- [測試](#testing)
    - [偽造回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止遺漏的請求](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 提供了一個表達力強、簡潔的 API，圍繞著 [Guzzle HTTP 客戶端](http://docs.guzzlephp.org/en/stable/)，讓您可以快速發出對外 HTTP 請求，以便與其他網路應用程式溝通。Laravel 對 Guzzle 的封裝著重於其最常見的用例，以及絕佳的開發者體驗。

<a name="making-requests"></a>
## 發出請求

若要發出請求，您可以使用 `Http` facade 提供的 `head`、`get`、`post`、`put`、`patch` 和 `delete` 方法。讓我們先來看看如何向另一個 URL 發出基本的 `GET` 請求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('http://example.com');
```

`get` 方法會回傳 `Illuminate\Http\Client\Response` 的實例，該實例提供了多種方法，可用於檢查回應：

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

`Illuminate\Http\Client\Response` 物件也實作了 PHP 的 `ArrayAccess` 介面，讓您可以直接從回應中存取 JSON 回應資料：

```php
return Http::get('http://example.com/users/1')['name'];
```

除了上述回應方法之外，您還可以使用以下方法來判斷回應是否具有特定的狀態碼：

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

HTTP 客戶端還允許您使用 [URI 範本規範](https://www.rfc-editor.org/rfc/rfc6570) 來建構請求 URL。若要定義可以由您的 URI 範本展開的 URL 參數，您可以使用 `withUrlParameters` 方法：

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

如果您希望在發送傳出請求實例之前傾印它並終止腳本的執行，您可以將 `dd` 方法添加到請求定義的開頭：

```php
return Http::dd()->get('http://example.com');
```

<a name="request-data"></a>
### 請求資料

當然，在發出 `POST`、`PUT` 和 `PATCH` 請求時，通常需要傳送額外的資料。因此，這些方法接受一個資料陣列作為它們的第二個引數。預設情況下，資料將使用 `application/json` 內容類型傳送：

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```

<a name="get-request-query-parameters"></a>
#### GET 請求查詢參數

發出 `GET` 請求時，您可以直接將查詢字串附加到 URL，或者將一個鍵/值對陣列作為第二個引數傳遞給 `get` 方法：

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
#### 傳送表單 URL 編碼請求

如果您想使用 `application/x-www-form-urlencoded` 內容類型傳送資料，您應該在發出請求之前呼叫 `asForm` 方法：

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```

<a name="sending-a-raw-request-body"></a>
#### 傳送原始請求主體

如果您想在發出請求時提供原始請求主體，可以使用 `withBody` 方法。內容類型可以透過方法的第二個引數提供：

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```

<a name="multi-part-requests"></a>
#### 多部分請求

如果您想以多部分請求的形式傳送檔案，您應該在發出請求之前呼叫 `attach` 方法。此方法接受檔案的名稱及其內容。如果需要，您可以提供第三個引數，該引數將被視為檔案的檔名，而第四個引數可用於提供與檔案相關聯的標頭：

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
)->post('http://example.com/attachments');
```

除了傳遞檔案的原始內容之外，您還可以傳遞一個串流資源：

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```

<a name="headers"></a>
### 標頭

標頭可以透過 `withHeaders` 方法添加到請求中。此 `withHeaders` 方法接受一個鍵/值對陣列：

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

您可以使用 `accept` 方法來指定您的應用程式預期在回應中收到的內容類型：

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

為了方便起見，您可以使用 `acceptJson` 方法快速指定您的應用程式預期在回應中收到 `application/json` 內容類型：

```php
$response = Http::acceptJson()->get('http://example.com/users');
```

`withHeaders` 方法會將新的標頭合併到請求現有的標頭中。如果需要，您可以使用 `replaceHeaders` 方法完全替換所有標頭：

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

您可以分別使用 `withBasicAuth` 和 `withDigestAuth` 方法來指定基本認證和摘要認證憑證：

```php
// Basic authentication...
$response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

// Digest authentication...
$response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);
```

<a name="bearer-tokens"></a>
#### Bearer Tokens

如果您想快速將 Bearer Token 添加到請求的 `Authorization` 標頭中，您可以使用 `withToken` 方法：

```php
$response = Http::withToken('token')->post(/* ... */);
```

<a name="timeout"></a>
### 逾時

`timeout` 方法可用於指定等待回應的最大秒數。預設情況下，HTTP 客戶端將在 30 秒後逾時：

```php
$response = Http::timeout(3)->get(/* ... */);
```

如果超過了給定的逾時時間，將會拋出 `Illuminate\Http\Client\ConnectionException` 的實例。

您可以使用 `connectTimeout` 方法指定嘗試連線到伺服器時等待的最大秒數。預設為 10 秒：

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>
### 重試

如果客戶端或伺服器發生錯誤時，您希望 HTTP 客戶端自動重試請求，您可以使用 `retry` 方法。`retry` 方法接受請求應嘗試的最大次數，以及 Laravel 在每次嘗試之間應等待的毫秒數：

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

為了方便起見，您也可以提供一個陣列作為 `retry` 方法的第一個參數。此陣列將用於決定後續嘗試之間休眠的毫秒數：

```php
$response = Http::retry([100, 200])->post(/* ... */);
```

如果需要，您可以向 `retry` 方法傳遞第三個參數。第三個參數應為一個可呼叫 (callable) 物件，用於判斷是否實際嘗試重試。例如，您可能希望僅在初始請求遇到 `ConnectionException` 時才重試請求：

```php
use Exception;
use Illuminate\Http\Client\PendingRequest;

$response = Http::retry(3, 100, function (Exception $exception, PendingRequest $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

如果請求嘗試失敗，您可能希望在新嘗試之前對請求進行更改。您可以透過修改您提供給 `retry` 方法的可呼叫 (callable) 物件中的請求參數來實現這一點。例如，如果第一次嘗試返回驗證錯誤，您可能希望使用新的授權 Token 重試請求：

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

如果所有請求都失敗，將拋出一個 `Illuminate\Http\Client\RequestException` 實例。如果您想禁用此行為，您可以提供一個值為 `false` 的 `throw` 參數。禁用後，在所有重試嘗試後，將返回客戶端收到的最後一個回應：

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> [!WARNING]
> 如果所有請求因連線問題而失敗，即使 `throw` 參數設為 `false`，仍將拋出 `Illuminate\Http\Client\ConnectionException`。

<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP 客戶端包裝器不會在客戶端或伺服器錯誤 (來自伺服器的 `400` 和 `500` 級別回應) 時拋出例外。您可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否返回了這些錯誤之一：

```php
// Determine if the status code is >= 200 and < 300...
$response->successful();

// Determine if the status code is >= 400...
$response->failed();

// Determine if the response has a 400 level status code...
$response->clientError();

// Determine if the response has a 500 level status code...
$response->serverError();

// Immediately execute the given callback if there was a client or server error...
$response->onError(callable $callback);
```

<a name="throwing-exceptions"></a>
#### 拋出例外

如果您有一個回應實例，並且希望在回應狀態碼指示客戶端或伺服器錯誤時拋出 `Illuminate\Http\Client\RequestException` 實例，您可以使用 `throw` 或 `throwIf` 方法：

```php
use Illuminate\Http\Client\Response;

$response = Http::post(/* ... */);

// Throw an exception if a client or server error occurred...
$response->throw();

// Throw an exception if an error occurred and the given condition is true...
$response->throwIf($condition);

// Throw an exception if an error occurred and the given closure resolves to true...
$response->throwIf(fn (Response $response) => true);

// Throw an exception if an error occurred and the given condition is false...
$response->throwUnless($condition);

// Throw an exception if an error occurred and the given closure resolves to false...
$response->throwUnless(fn (Response $response) => false);

// Throw an exception if the response has a specific status code...
$response->throwIfStatus(403);

// Throw an exception unless the response has a specific status code...
$response->throwUnlessStatus(200);

return $response['user']['id'];
```

`Illuminate\Http\Client\RequestException` 實例有一個公開的 `$response` 屬性，可讓您檢查返回的回應。

如果沒有發生錯誤，`throw` 方法將返回回應實例，允許您將其他操作鏈接到 `throw` 方法上：

```php
return Http::post(/* ... */)->throw()->json();
```

如果您想在拋出例外之前執行一些額外的邏輯，您可以將一個閉包傳遞給 `throw` 方法。例外將在閉包被調用後自動拋出，因此您無需在閉包內部重新拋出例外：

```php
use Illuminate\Http\Client\Response;
use Illuminate\Http\Client\RequestException;

return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
    // ...
})->json();
```

預設情況下，`RequestException` 訊息在記錄或報告時會被截斷為 120 個字元。若要自訂或禁用此行為，您可以在 `bootstrap/app.php` 檔案中設定應用程式的例外處理行為時，使用 `truncateRequestExceptionsAt` 和 `dontTruncateRequestExceptions` 方法：

```php
use Illuminate\Foundation\Configuration\Exceptions;

->withExceptions(function (Exceptions $exceptions): void {
    // Truncate request exception messages to 240 characters...
    $exceptions->truncateRequestExceptionsAt(240);

    // Disable request exception message truncation...
    $exceptions->dontTruncateRequestExceptions();
})
```

此外，您可以使用 `truncateExceptionsAt` 方法自訂每個請求的例外截斷行為：

```php
return Http::truncateExceptionsAt(240)->post(/* ... */);
```

<a name="guzzle-middleware"></a>
### Guzzle 中介層

由於 Laravel 的 HTTP client 由 Guzzle 提供支援，您可以利用 [Guzzle 中介層](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來操作發出的請求或檢查接收到的回應。若要操作發出的請求，可以透過 `withRequestMiddleware` 方法註冊一個 Guzzle 中介層：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withRequestMiddleware(
    function (RequestInterface $request) {
        return $request->withHeader('X-Example', 'Value');
    }
)->get('http://example.com');
```

同樣地，您可以透過 `withResponseMiddleware` 方法註冊一個中介層來檢查接收到的 HTTP 回應：

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
#### 全域中介層

有時候，您可能想要註冊一個適用於所有發出請求和接收回應的中介層。為此，您可以使用 `globalRequestMiddleware` 和 `globalResponseMiddleware` 方法。通常，這些方法應該在應用程式的 `AppServiceProvider` 中 `boot` 方法內呼叫：

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

您可以使用 `withOptions` 方法為發出的請求指定額外的 [Guzzle 請求選項](http://docs.guzzlephp.org/en/stable/request-options.html)。`withOptions` 方法接受一個鍵/值陣列：

```php
$response = Http::withOptions([
    'debug' => true,
])->get('http://example.com/users');
```

<a name="global-options"></a>
#### 全域選項

若要為每個發出請求設定預設選項，您可以使用 `globalOptions` 方法。通常，此方法應在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

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
## 併發請求

有時候，您可能會希望併發地發出多個 HTTP 請求。換句話說，您希望多個請求能夠同時發送，而不是依序發出。這在與緩慢的 HTTP API 互動時，可以顯著提升效能。

幸運的是，您可以使用 `pool` 方法來達成此目的。`pool` 方法接受一個閉包，該閉包會接收一個 `Illuminate\Http\Client\Pool` 實例，讓您可以輕鬆地將請求添加到請求池中以進行發送：

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

如您所見，每個回應實例都可以根據其添加到池中的順序進行存取。如果您願意，可以使用 `as` 方法為請求命名，這將允許您透過名稱存取對應的回應：

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
#### 自訂併發請求

`pool` 方法不能與其他 HTTP 客戶端方法，例如 `withHeaders` 或 `middleware` 方法鏈接使用。如果您希望將自訂標頭或中介層應用於池化請求，您應該在池中的每個請求上配置這些選項：

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
## 巨集

Laravel 的 HTTP 客戶端允許您定義「巨集 (macros)」，它可以作為一種流暢、富有表現力的方式，用於在應用程式中與服務互動時配置常見的請求路徑和標頭。要開始使用，您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義該巨集：

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

一旦您的巨集配置完成，您就可以在應用程式中的任何地方呼叫它，以建立一個具有指定配置的待處理請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

Laravel 提供了許多服務，可協助您輕鬆且具表達力地撰寫測試，Laravel 的 HTTP 客戶端也不例外。`Http` Facade 的 `fake` 方法允許您指示 HTTP 客戶端在發出請求時回傳假 / 模擬回應。

<a name="faking-responses"></a>
### 偽造回應

例如，要指示 HTTP 客戶端針對每個請求都回傳空的 `200` 狀態碼回應，您可以呼叫不帶任何參數的 `fake` 方法：

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```

<a name="faking-specific-urls"></a>
#### 偽造特定 URL

或者，您可以傳遞一個陣列給 `fake` 方法。該陣列的鍵應該代表您希望偽造的 URL 模式及其相關聯的回應。`*` 字元可用作萬用字元。您可以使用 `Http` Facade 的 `response` 方法來為這些端點建構假 / 模擬回應：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // Stub a string response for Google endpoints...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

任何針對未偽造 URL 發出的請求將會實際執行。如果您想要指定一個後備 URL 模式，用於模擬所有不匹配的 URL，您可以使用單個 `*` 字元：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // Stub a string response for all other endpoints...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

為了方便，您可以透過提供字串、陣列或整數作為回應來產生簡單的字串、JSON 和空回應：

```php
Http::fake([
    'google.com/*' => 'Hello World',
    'github.com/*' => ['foo' => 'bar'],
    'chatgpt.com/*' => 200,
]);
```

<a name="faking-connection-exceptions"></a>
#### 偽造異常

有時您可能需要測試應用程式的行為，如果 HTTP 客戶端在嘗試發出請求時遇到 `Illuminate\Http\Client\ConnectionException`。您可以使用 `failedConnection` 方法指示 HTTP 客戶端拋出連線異常：

```php
Http::fake([
    'github.com/*' => Http::failedConnection(),
]);
```

要測試您的應用程式行為，如果拋出 `Illuminate\Http\Client\RequestException`，您可以使用 `failedRequest` 方法：

```php
Http::fake([
    'github.com/*' => Http::failedRequest(['code' => 'not_found'], 404),
]);
```

<a name="faking-response-sequences"></a>
#### 偽造回應序列

有時您可能需要指定單一 URL 應該以特定順序回傳一系列的假回應。您可以使用 `Http::sequence` 方法來建構這些回應：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

當回應序列中的所有回應都被使用後，任何後續的請求都會導致回應序列拋出異常。如果您想指定一個預設回應，應該在序列為空時回傳，您可以使用 `whenEmpty` 方法：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

如果您想偽造一系列回應，但不需要指定應該被偽造的特定 URL 模式，您可以使用 `Http::fakeSequence` 方法：

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```

<a name="fake-callback"></a>
#### 偽造回呼函式

如果您需要更複雜的邏輯來決定針對特定端點回傳什麼回應，您可以傳遞一個閉包 (closure) 給 `fake` 方法。這個閉包將會收到一個 `Illuminate\Http\Client\Request` 實例，並且應該回傳一個回應實例。在您的閉包中，您可以執行任何必要的邏輯來決定要回傳哪種回應：

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```

<a name="inspecting-requests"></a>
### 檢查請求

在偽造回應時，您可能偶爾會希望檢查客戶端收到的請求，以確保您的應用程式傳送了正確的資料或標頭。您可以在呼叫 `Http::fake` 之後，透過呼叫 `Http::assertSent` 方法來實現此目的。

`assertSent` 方法接受一個閉包，該閉包將會收到一個 `Illuminate\Http\Client\Request` 實例，並且應該回傳一個布林值，指示請求是否符合您的預期。為了讓測試通過，至少必須發出一個符合給定預期的請求：

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

如有需要，您可以使用 `assertNotSent` 方法斷言特定請求未被發送：

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

您可以使用 `assertSentCount` 方法斷言在測試期間發送了多少個請求：

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
#### 記錄請求 / 回應

您可以使用 `recorded` 方法來收集所有請求及其對應的回應。`recorded` 方法回傳一個陣列的集合，該集合包含 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例：

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

此外，`recorded` 方法接受一個閉包，該閉包將會收到一個 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例，並且可以用來根據您的預期過濾請求 / 回應對：

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
### 防止遺漏的請求

如果您想確保所有透過 HTTP 客戶端發送的請求在您的個別測試或完整測試套件中都被偽造 (faked)，您可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應偽造回應的請求將會拋出例外，而不是發送實際的 HTTP 請求：

```php
use Illuminate\Support\Facades\Http;

Http::preventStrayRequests();

Http::fake([
    'github.com/*' => Http::response('ok'),
]);

// An "ok" response is returned...
Http::get('https://github.com/laravel/framework');

// An exception is thrown...
Http::get('https://laravel.com');
```

有時，您可能希望防止大部分的遺漏請求，同時仍然允許特定的請求執行。為了實現這一點，您可以向 `allowStrayRequests` 方法傳入一個 URL 模式陣列。任何符合所提供模式的請求將被允許，而所有其他請求則會繼續拋出例外：

```php
use Illuminate\Support\Facades\Http;

Http::preventStrayRequests();

Http::allowStrayRequests([
    'http://127.0.0.1:5000/*',
]);

// This request is executed...
Http::get('http://127.0.0.1:5000/generate');

// An exception is thrown...
Http::get('https://laravel.com');
```

<a name="events"></a>
## 事件

Laravel 在發送 HTTP 請求的過程中會觸發三個事件。`RequestSending` 事件會在請求發送之前觸發，而 `ResponseReceived` 事件則會在收到指定請求的回應之後觸發。如果指定請求沒有收到任何回應，則會觸發 `ConnectionFailed` 事件。

`RequestSending` 和 `ConnectionFailed` 這兩個事件都包含一個公開的 `$request` 屬性，您可以使用它來檢查 `Illuminate\Http\Client\Request` 實例。同樣地，`ResponseReceived` 事件也包含一個 `$request` 屬性以及一個 `$response` 屬性，您可以使用這些屬性來檢查 `Illuminate\Http\Client\Response` 實例。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

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