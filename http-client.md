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
- [並行請求](#concurrent-requests)
- [巨集](#macros)
- [測試](#testing)
    - [模擬回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止雜散請求](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 為 [Guzzle HTTP 客戶端](http://docs.guzzlephp.org/en/stable/) 提供了一個富有表達力、精簡的 API，讓您能夠快速發出外部 HTTP 請求，以便與其他 Web 應用程式通訊。Laravel 對 Guzzle 的封裝專注於其最常見的使用案例以及提供絕佳的開發者體驗。

<a name="making-requests"></a>
## 發出請求

要發出請求，你可以使用 `Http` facade 提供的 `head`、`get`、`post`、`put`、`patch` 和 `delete` 方法。首先，我們來看看如何向另一個 URL 發出一個基本的 `GET` 請求：

    use Illuminate\Support\Facades\Http;

    $response = Http::get('http://example.com');

`get` 方法會回傳一個 `Illuminate\Http\Client\Response` 實例，該實例提供了各種方法，可用於檢查回應：

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

`Illuminate\Http\Client\Response` 物件也實作了 PHP 的 `ArrayAccess` 介面，讓你可以直接從回應中存取 JSON 回應資料：

    return Http::get('http://example.com/users/1')['name'];

除了以上列出的回應方法外，還可以使用以下方法來判斷回應是否具有指定的狀態碼：

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

<a name="uri-templates"></a>
#### URI 範本

HTTP 客戶端也允許你使用 [URI 範本規範](https://www.rfc-editor.org/rfc/rfc6570) 來建構請求 URL。要定義可以由 URI 範本展開的 URL 參數，你可以使用 `withUrlParameters` 方法：

```php
Http::withUrlParameters([
    'endpoint' => 'https://laravel.com',
    'page' => 'docs',
    'version' => '11.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```

<a name="dumping-requests"></a>
#### 傾印請求

如果你想在發出請求前傾印發出的請求實例並終止 Script 執行，你可以將 `dd` 方法加到你的請求定義的開頭：

    return Http::dd()->get('http://example.com');

<a name="request-data"></a>
### 請求資料

當然，發出 `POST`、`PUT` 和 `PATCH` 請求時，通常會隨請求發送額外資料，因此這些方法會接受一個資料陣列作為第二個參數。預設情況下，資料將使用 `application/json` 的 Content Type 發送：

    use Illuminate\Support\Facades\Http;

    $response = Http::post('http://example.com/users', [
        'name' => 'Steve',
        'role' => 'Network Administrator',
    ]);

<a name="get-request-query-parameters"></a>
#### GET 請求查詢參數

發出 `GET` 請求時，你可以直接將查詢字串附加到 URL，或者傳遞一個鍵/值對 (key / value pairs) 的陣列作為 `get` 方法的第二個參數：

    $response = Http::get('http://example.com/users', [
        'name' => 'Taylor',
        'page' => 1,
    ]);

或者，可以使用 `withQueryParameters` 方法：

    Http::retry(3, 100)->withQueryParameters([
        'name' => 'Taylor',
        'page' => 1,
    ])->get('http://example.com/users')

<a name="sending-form-url-encoded-requests"></a>
#### 發送表單 URL 編碼請求

如果你想使用 `application/x-www-form-urlencoded` 的 Content Type 發送資料，你應該在發出請求之前呼叫 `asForm` 方法：

    $response = Http::asForm()->post('http://example.com/users', [
        'name' => 'Sara',
        'role' => 'Privacy Consultant',
    ]);

<a name="sending-a-raw-request-body"></a>
#### 發送原始請求主體

如果你想在發出請求時提供原始請求主體，你可以使用 `withBody` 方法。Content Type 可以透過方法的第二個參數提供：

    $response = Http::withBody(
        base64_encode($photo), 'image/jpeg'
    )->post('http://example.com/photo');

<a name="multi-part-requests"></a>
#### 多部分請求

如果你想以多部分請求的形式發送檔案，你應該在發出請求之前呼叫 `attach` 方法。此方法接受檔案名稱及其內容。如果需要，你可以提供第三個參數，該參數將被視為檔案的檔案名稱，而第四個參數可用於提供與檔案相關的標頭：

    $response = Http::attach(
        'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
    )->post('http://example.com/attachments');

除了傳遞檔案的原始內容外，你也可以傳遞一個 Stream 資源：

    $photo = fopen('photo.jpg', 'r');

    $response = Http::attach(
        'attachment', $photo, 'photo.jpg'
    )->post('http://example.com/attachments');

<a name="headers"></a>
### 標頭

標頭可以透過 `withHeaders` 方法添加到請求中。此 `withHeaders` 方法接受一個鍵/值對 (key / value pairs) 的陣列：

    $response = Http::withHeaders([
        'X-First' => 'foo',
        'X-Second' => 'bar'
    ])->post('http://example.com/users', [
        'name' => 'Taylor',
    ]);

你可以使用 `accept` 方法來指定你的應用程式在回應中預期的 Content Type：

    $response = Http::accept('application/json')->get('http://example.com/users');

為了方便，你可以使用 `acceptJson` 方法快速指定你的應用程式在回應中預期 `application/json` 的 Content Type：

    $response = Http::acceptJson()->get('http://example.com/users');

`withHeaders` 方法會將新的標頭合併到請求的現有標頭中。如果需要，你可以使用 `replaceHeaders` 方法完全替換所有標頭：

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

你可以分別使用 `withBasicAuth` 和 `withDigestAuth` 方法來指定基本和摘要認證憑證：

    // Basic authentication...
    $response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

    // Digest authentication...
    $response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);

<a name="bearer-tokens"></a>
#### Bearer 權杖

如果你想快速將 Bearer 權杖加到請求的 `Authorization` 標頭中，你可以使用 `withToken` 方法：

    $response = Http::withToken('token')->post(/* ... */);

<a name="timeout"></a>
### 逾時

`timeout` 方法可用於指定等待回應的最大秒數。預設情況下，HTTP 客戶端將在 30 秒後逾時：

    $response = Http::timeout(3)->get(/* ... */);

如果超過指定的逾時時間，將會丟出一個 `Illuminate\Http\Client\ConnectionException` 實例。

你可以使用 `connectTimeout` 方法指定嘗試連線到伺服器時等待的最大秒數：

    $response = Http::connectTimeout(3)->get(/* ... */);

<a name="retries"></a>
### 重試

如果您希望 HTTP 客戶端在發生客戶端或伺服器錯誤時自動重試請求，您可以使用 `retry` 方法。`retry` 方法接受請求應嘗試的最大次數，以及 Laravel 在兩次嘗試之間應等待的毫秒數：

    $response = Http::retry(3, 100)->post(/* ... */);

如果您想手動計算每次嘗試之間休眠的毫秒數，您可以將一個閉包作為第二個參數傳遞給 `retry` 方法：

    use Exception;

    $response = Http::retry(3, function (int $attempt, Exception $exception) {
        return $attempt * 100;
    })->post(/* ... */);

為方便起見，您也可以提供一個陣列作為 `retry` 方法的第一個參數。此陣列將用於決定後續嘗試之間休眠的毫秒數：

    $response = Http::retry([100, 200])->post(/* ... */);

如有需要，您可以將第三個參數傳遞給 `retry` 方法。第三個參數應為一個可呼叫的函式 (callable)，用於判斷是否實際應嘗試重試。例如，您可能只希望在初始請求遇到 `ConnectionException` 時重試：

    use Exception;
    use Illuminate\Http\Client\PendingRequest;

    $response = Http::retry(3, 100, function (Exception $exception, PendingRequest $request) {
        return $exception instanceof ConnectionException;
    })->post(/* ... */);

如果請求嘗試失敗，您可能希望在新嘗試之前對請求進行更改。您可以透過修改傳遞給 `retry` 方法中可呼叫函式 (callable) 的請求參數來實現此目的。例如，如果第一次嘗試返回了認證錯誤，您可能希望使用新的授權憑證 (authorization token) 重試請求：

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

如果所有請求都失敗，將會拋出 `Illuminate\Http\Client\RequestException` 實例。如果您想禁用此行為，您可以提供一個 `throw` 參數，其值為 `false`。禁用後，在所有重試嘗試完成後，將會返回客戶端收到的最後一個回應：

    $response = Http::retry(3, 100, throw: false)->post(/* ... */);

> [!WARNING]  
> 如果所有請求因連線問題而失敗，即使 `throw` 參數設定為 `false`，仍會拋出 `Illuminate\Http\Client\ConnectionException`。

<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP 客戶端包裝器不會在客戶端或伺服器錯誤（來自伺服器的 `400` 和 `500` 級別回應）時拋出例外。您可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否返回了這些錯誤之一：

    // 判斷狀態碼是否 >= 200 且 < 300...
    $response->successful();

    // 判斷狀態碼是否 >= 400...
    $response->failed();

    // 判斷回應的狀態碼是否為 400 級別...
    $response->clientError();

    // 判斷回應的狀態碼是否為 500 級別...
    $response->serverError();

    // 如果發生客戶端或伺服器錯誤，立即執行給定的回呼函數...
    $response->onError(callable $callback);

<a name="throwing-exceptions"></a>
#### 拋出例外

如果您有一個回應實例，並且希望在回應狀態碼指示客戶端或伺服器錯誤時拋出 `Illuminate\Http\Client\RequestException` 實例，您可以使用 `throw` 或 `throwIf` 方法：

    use Illuminate\Http\Client\Response;

    $response = Http::post(/* ... */);

    // 如果發生客戶端或伺服器錯誤，則拋出例外...
    $response->throw();

    // 如果發生錯誤且給定條件為真，則拋出例外...
    $response->throwIf($condition);

    // 如果發生錯誤且給定的閉包解析為真，則拋出例外...
    $response->throwIf(fn (Response $response) => true);

    // 如果發生錯誤且給定條件為假，則拋出例外...
    $response->throwUnless($condition);

    // 如果發生錯誤且給定的閉包解析為假，則拋出例外...
    $response->throwUnless(fn (Response $response) => false);

    // 如果回應具有特定狀態碼，則拋出例外...
    $response->throwIfStatus(403);

    // 除非回應具有特定狀態碼，否則拋出例外...
    $response->throwUnlessStatus(200);

    return $response['user']['id'];

`Illuminate\Http\Client\RequestException` 實例具有一個公共的 `$response` 屬性，它將允許您檢查返回的回應。

如果沒有發生錯誤，`throw` 方法會返回回應實例，讓您可以將其他操作鏈接到 `throw` 方法：

    return Http::post(/* ... */)->throw()->json();

如果您想在拋出例外之前執行一些額外邏輯，您可以將閉包傳遞給 `throw` 方法。例外將在閉包被呼叫後自動拋出，因此您不需要在閉包內部重新拋出例外：

    use Illuminate\Http\Client\Response;
    use Illuminate\Http\Client\RequestException;

    return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
        // ...
    })->json();

預設情況下，`RequestException` 訊息在記錄或報告時會被截斷為 120 個字元。要自訂或禁用此行為，您可以在 `bootstrap/app.php` 檔案中配置應用程式的例外處理行為時，利用 `truncateRequestExceptionsAt` 和 `dontTruncateRequestExceptions` 方法：

    ->withExceptions(function (Exceptions $exceptions) {
        // 將請求例外訊息截斷至 240 個字元...
        $exceptions->truncateRequestExceptionsAt(240);

        // 停用請求例外訊息截斷...
        $exceptions->dontTruncateRequestExceptions();
    })

<a name="guzzle-middleware"></a>
### Guzzle 中介層

由於 Laravel 的 HTTP 客戶端是由 Guzzle 提供支援，您可以利用 [Guzzle Middleware](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來操作傳出的請求或檢查傳入的回應。若要操作傳出的請求，可以透過 `withRequestMiddleware` 方法註冊一個 Guzzle 中介層：

    use Illuminate\Support\Facades\Http;
    use Psr\Http\Message\RequestInterface;

    $response = Http::withRequestMiddleware(
        function (RequestInterface $request) {
            return $request->withHeader('X-Example', 'Value');
        }
    )->get('http://example.com');

同樣地，您可以透過 `withResponseMiddleware` 方法註冊一個中介層來檢查傳入的 HTTP 回應：

    use Illuminate\Support\Facades\Http;
    use Psr\Http\Message\ResponseInterface;

    $response = Http::withResponseMiddleware(
        function (ResponseInterface $response) {
            $header = $response->getHeader('X-Example');

            // ...

            return $response;
        }
    )->get('http://example.com');


<a name="global-middleware"></a>
#### 全域中介層

有時，您可能會想註冊一個中介層，它能應用於每個傳出的請求和傳入的回應。要實現這點，您可以使用 `globalRequestMiddleware` 和 `globalResponseMiddleware` 方法。通常，這些方法應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中被呼叫：

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

您可以使用 `withOptions` 方法為傳出的請求指定額外的 [Guzzle 請求選項](http://docs.guzzlephp.org/en/stable/request-options.html)。`withOptions` 方法接受一個鍵值對 (key / value pairs) 陣列：

    $response = Http::withOptions([
        'debug' => true,
    ])->get('http://example.com/users');


<a name="global-options"></a>
#### 全域選項

若要為每個傳出的請求設定預設選項，您可以使用 `globalOptions` 方法。通常，此方法應在應用程式的 `AppServiceProvider` 的 `boot` 方法中被呼叫：

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

有時候，您可能希望並行發出多個 HTTP 請求。換句話說，您希望同時分派多個請求，而不是依序發出請求。這在與緩慢的 HTTP API 互動時，能顯著提升效能。

幸運的是，您可以使用 `pool` 方法來達成此目的。`pool` 方法接受一個閉包，該閉包會接收一個 `Illuminate\Http\Client\Pool` 實例，讓您能夠輕鬆地將請求加入請求池中以進行分派：

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

如您所見，每個回應實例都可以根據其加入池中的順序來存取。如果您願意，可以使用 `as` 方法為請求命名，這將讓您能夠透過名稱存取對應的回應：

    use Illuminate\Http\Client\Pool;
    use Illuminate\Support\Facades\Http;

    $responses = Http::pool(fn (Pool $pool) => [
        $pool->as('first')->get('http://localhost/first'),
        $pool->as('second')->get('http://localhost/second'),
        $pool->as('third')->get('http://localhost/third'),
    ]);

    return $responses['first']->ok();


<a name="customizing-concurrent-requests"></a>
#### 自訂並行請求

`pool` 方法不能與其他 HTTP 客戶端方法（例如 `withHeaders` 或 `middleware` 方法）鏈接使用。如果您希望對池中的請求應用自訂標頭或中介層，則應在池中的每個請求上配置這些選項：

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

Laravel HTTP 客戶端允許您定義「巨集」，它可以作為一種流暢、富有表現力的方式，在與應用程式中的服務互動時配置常見的請求路徑和標頭。若要開始使用，您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義巨集：

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

一旦您的巨集配置完成，您就可以在應用程式的任何地方調用它，以創建具有指定配置的待處理請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

許多 Laravel 服務提供了功能，能幫助你輕鬆且清晰地編寫測試，Laravel 的 HTTP 客戶端也不例外。`Http` facade 的 `fake` 方法允許你指示 HTTP 客戶端在發出請求時返回模擬 / 虛擬的回應。

<a name="faking-responses"></a>
### 模擬回應

舉例來說，要指示 HTTP 客戶端對每個請求返回空的 `200` 狀態碼回應，你可以呼叫不帶任何參數的 `fake` 方法：

    use Illuminate\Support\Facades\Http;

    Http::fake();

    $response = Http::post(/* ... */);

<a name="faking-specific-urls"></a>
#### 模擬特定 URL

或者，你可以傳遞一個陣列給 `fake` 方法。該陣列的鍵應代表你希望模擬的 URL 模式及其相關回應。`*` 字元可以用作萬用字元。任何發送給未模擬的 URL 的請求將會實際執行。你可以使用 `Http` facade 的 `response` 方法為這些端點建構模擬 / 虛假回應：

    Http::fake([
        // Stub a JSON response for GitHub endpoints...
        'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

        // Stub a string response for Google endpoints...
        'google.com/*' => Http::response('Hello World', 200, $headers),
    ]);

如果你想指定一個後備 URL 模式來模擬所有不匹配的 URL，你可以使用單個 `*` 字元：

    Http::fake([
        // Stub a JSON response for GitHub endpoints...
        'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

        // Stub a string response for all other endpoints...
        '*' => Http::response('Hello World', 200, ['Headers']),
    ]);

為方便起見，簡單的字串、JSON 和空回應可以透過提供字串、陣列或整數作為回應來生成：

    Http::fake([
        'google.com/*' => 'Hello World',
        'github.com/*' => ['foo' => 'bar'],
        'chatgpt.com/*' => 200,
    ]);

<a name="faking-connection-exceptions"></a>
#### 模擬連線例外

有時你可能需要測試你的應用程式在 HTTP 客戶端嘗試發出請求時遇到 `Illuminate\Http\Client\ConnectionException` 的行為。你可以使用 `failedConnection` 方法指示 HTTP 客戶端拋出連線例外：

    Http::fake([
        'github.com/*' => Http::failedConnection(),
    ]);

<a name="faking-response-sequences"></a>
#### 模擬回應序列

有時你可能需要指定一個單一的 URL 應該按特定順序返回一系列的模擬回應。你可以使用 `Http::sequence` 方法來建構這些回應：

    Http::fake([
        // Stub a series of responses for GitHub endpoints...
        'github.com/*' => Http::sequence()
            ->push('Hello World', 200)
            ->push(['foo' => 'bar'], 200)
            ->pushStatus(404),
    ]);

當回應序列中的所有回應都被使用完畢後，任何進一步的請求都會導致回應序列拋出例外。如果你想指定一個預設回應，在序列為空時返回，你可以使用 `whenEmpty` 方法：

    Http::fake([
        // Stub a series of responses for GitHub endpoints...
        'github.com/*' => Http::sequence()
            ->push('Hello World', 200)
            ->push(['foo' => 'bar'], 200)
            ->whenEmpty(Http::response()),
    ]);

如果你想模擬一系列回應，但不需要指定應模擬的特定 URL 模式，你可以使用 `Http::fakeSequence` 方法：

    Http::fakeSequence()
        ->push('Hello World', 200)
        ->whenEmpty(Http::response());

<a name="fake-callback"></a>
#### 模擬回呼

如果你需要更複雜的邏輯來決定為特定端點返回什麼回應，你可以將一個閉包傳遞給 `fake` 方法。這個閉包將接收一個 `Illuminate\Http\Client\Request` 實例，並應返回一個回應實例。在你的閉包中，你可以執行任何必要的邏輯來決定要返回哪種回應：

    use Illuminate\Http\Client\Request;

    Http::fake(function (Request $request) {
        return Http::response('Hello World', 200);
    });

<a name="preventing-stray-requests"></a>
### 防止雜散請求

如果你想確保透過 HTTP 客戶端發送的所有請求在你的單獨測試或整個測試套件中都被模擬，你可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應模擬回應的請求都將拋出例外，而不是發出實際的 HTTP 請求：

    use Illuminate\Support\Facades\Http;

    Http::preventStrayRequests();

    Http::fake([
        'github.com/*' => Http::response('ok'),
    ]);

    // An "ok" response is returned...
    Http::get('https://github.com/laravel/framework');

    // An exception is thrown...
    Http::get('https://laravel.com');

<a name="inspecting-requests"></a>
### 檢查請求

當模擬回應時，您可能偶爾會希望檢查客戶端收到的請求，以確保您的應用程式正在傳送正確的資料或標頭。您可以在呼叫 `Http::fake` 方法之後，透過呼叫 `Http::assertSent` 方法來實現此目的。

`assertSent` 方法接受一個閉包，該閉包將會收到一個 `Illuminate\Http\Client\Request` 實例，並且應該回傳一個布林值，指示該請求是否符合您的預期。為了讓測試通過，至少必須發出一個符合指定預期的請求：

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

如有需要，您可以使用 `assertNotSent` 方法來斷言某個特定請求未被發送：

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

您可以使用 `assertSentCount` 方法來斷言測試期間發送了多少個請求：

    Http::fake();

    Http::assertSentCount(5);

或者，您可以使用 `assertNothingSent` 方法來斷言測試期間沒有發送任何請求：

    Http::fake();

    Http::assertNothingSent();

<a name="recording-requests-and-responses"></a>
#### 記錄請求與回應

您可以使用 `recorded` 方法來收集所有請求及其對應的回應。`recorded` 方法會回傳一個陣列的 Collection，其中包含 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例：

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

此外，`recorded` 方法接受一個閉包，該閉包將會收到一個 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例，並可用於根據您的預期來過濾請求/回應對：

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

<a name="events"></a>
## 事件

Laravel 在發送 HTTP 請求的過程中會觸發三個事件。`RequestSending` 事件會在請求發送前觸發，而 `ResponseReceived` 事件則會在收到指定請求的回應後觸發。如果指定請求沒有收到任何回應，則會觸發 `ConnectionFailed` 事件。

`RequestSending` 和 `ConnectionFailed` 事件都包含一個公共的 `$request` 屬性，您可以使用它來檢查 `Illuminate\Http\Client\Request` 實例。同樣地，`ResponseReceived` 事件包含一個 `$request` 屬性以及一個 `$response` 屬性，可以用來檢查 `Illuminate\Http\Client\Response` 實例。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

    ```php
    use Illuminate\Http\Client\Events\RequestSending;

    class LogRequest
    {
        /**
         * Handle the given event.
         */
        public function handle(RequestSending $event): void
        {
            // $event->request ...
        }
    }
    ```