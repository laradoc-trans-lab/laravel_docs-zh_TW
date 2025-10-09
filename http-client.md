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
    - [請求池](#request-pooling)
    - [請求批次處理](#request-batching)
- [巨集](#macros)
- [測試](#testing)
    - [模擬回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止離題請求](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 為 [Guzzle HTTP 客戶端](http://docs.guzzlephp.org/en/stable/) 提供了一個簡潔、具表達性的 API，讓您能快速發出對外的 HTTP 請求以與其他網路應用程式溝通。Laravel 對 Guzzle 的封裝專注於其最常見的使用案例以及極佳的開發者體驗。

<a name="making-requests"></a>
## 發出請求

要發出請求，你可以使用 `Http` 外觀所提供的 `head`、`get`、`post`、`put`、`patch` 和 `delete` 方法。首先，讓我們看看如何向另一個 URL 發出基本的 `GET` 請求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('http://example.com');
```

`get` 方法會回傳一個 `Illuminate\Http\Client\Response` 實例，它提供了多種方法可用於檢查回應：

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

`Illuminate\Http\Client\Response` 物件也實現了 PHP 的 `ArrayAccess` 介面，允許你直接在回應上存取 JSON 回應資料：

```php
return Http::get('http://example.com/users/1')['name'];
```

除了上面列出的回應方法之外，還可以使用以下方法來判斷回應是否具有特定的狀態碼：

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

HTTP 客戶端也允許你使用 [URI 範本規範](https://www.rfc-editor.org/rfc/rfc6570)來建構請求 URL。要定義可由你的 URI 範本擴展的 URL 參數，你可以使用 `withUrlParameters` 方法：

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

如果你想在送出請求實例之前將其傾印並終止腳本執行，你可以在請求定義的開頭加上 `dd` 方法：

```php
return Http::dd()->get('http://example.com');
```

<a name="request-data"></a>
### 請求資料

當然，在發出 `POST`、`PUT` 和 `PATCH` 請求時，通常會隨請求傳送額外資料，因此這些方法接受一個資料陣列作為第二個參數。預設情況下，資料將使用 `application/json` 內容類型傳送：

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```

<a name="get-request-query-parameters"></a>
#### GET 請求查詢參數

發出 `GET` 請求時，你可以直接將查詢字串附加到 URL，或將鍵/值對陣列作為 `get` 方法的第二個參數傳遞：

```php
$response = Http::get('http://example.com/users', [
    'name' => 'Taylor',
    'page' => 1,
]);
```

或者，也可以使用 `withQueryParameters` 方法：

```php
Http::retry(3, 100)->withQueryParameters([
    'name' => 'Taylor',
    'page' => 1,
])->get('http://example.com/users');
```

<a name="sending-form-url-encoded-requests"></a>
#### 傳送表單 URL 編碼請求

如果你想使用 `application/x-www-form-urlencoded` 內容類型傳送資料，你應該在發出請求之前呼叫 `asForm` 方法：

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```

<a name="sending-a-raw-request-body"></a>
#### 傳送原始請求主體

如果你想在發出請求時提供原始請求主體，可以使用 `withBody` 方法。內容類型可以透過方法的第二個參數提供：

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```

<a name="multi-part-requests"></a>
#### 多部分請求

如果你想以多部分請求的形式傳送檔案，你應該在發出請求之前呼叫 `attach` 方法。此方法接受檔案名稱及其內容。如果需要，你可以提供第三個參數作為檔案的檔案名稱，而第四個參數可用於提供與檔案相關聯的標頭：

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
)->post('http://example.com/attachments');
```

你可以傳遞資料流資源，而不是傳遞檔案的原始內容：

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```

<a name="headers"></a>
### 標頭

可以使用 `withHeaders` 方法將標頭添加到請求中。此 `withHeaders` 方法接受一個鍵/值對的陣列：

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

你可以使用 `accept` 方法來指定你的應用程式預期在回應中收到的內容類型：

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

為方便起見，你可以使用 `acceptJson` 方法快速指定你的應用程式預期在回應中收到 `application/json` 內容類型：

```php
$response = Http::acceptJson()->get('http://example.com/users');
```

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

你可以分別使用 `withBasicAuth` 和 `withDigestAuth` 方法指定基本和摘要認證憑證：

```php
// Basic authentication...
$response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

// Digest authentication...
$response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);
```

<a name="bearer-tokens"></a>
#### 不記名權杖

如果你想快速將不記名權杖添加到請求的 `Authorization` 標頭中，可以使用 `withToken` 方法：

```php
$response = Http::withToken('token')->post(/* ... */);
```

<a name="timeout"></a>
### 逾時

`timeout` 方法可用於指定等待回應的最大秒數。預設情況下，HTTP 客戶端將在 30 秒後逾時：

```php
$response = Http::timeout(3)->get(/* ... */);
```

如果超過給定的逾時時間，將拋出 `Illuminate\Http\Client\ConnectionException` 實例。

你可以使用 `connectTimeout` 方法指定嘗試連接伺服器時等待的最大秒數。預設為 10 秒：

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>
### 重試

如果您希望 HTTP 客戶端在發生客戶端或伺服器錯誤時自動重試請求，您可以使用 `retry` 方法。`retry` 方法接受最大重試次數，以及 Laravel 在每次嘗試之間應等待的毫秒數：

```php
$response = Http::retry(3, 100)->post(/* ... */);
```

如果您想手動計算兩次嘗試之間應暫停的毫秒數，您可以將一個閉包作為第二個引數傳遞給 `retry` 方法：

```php
use Exception;

$response = Http::retry(3, function (int $attempt, Exception $exception) {
    return $attempt * 100;
})->post(/* ... */);
```

為方便起見，您也可以將陣列作為第一個引數傳遞給 `retry` 方法。此陣列將用於判斷後續嘗試之間應暫停多少毫秒：

```php
$response = Http::retry([100, 200])->post(/* ... */);
```

如有需要，您可以將第三個引數傳遞給 `retry` 方法。第三個引數應該是一個 callable，用於判斷是否應實際嘗試重試。例如，您可能只希望在初始請求遇到 `ConnectionException` 時才重試請求：

```php
use Exception;
use Illuminate\Http\Client\PendingRequest;

$response = Http::retry(3, 100, function (Exception $exception, PendingRequest $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

如果請求嘗試失敗，您可能希望在進行新的嘗試之前對請求進行變更。您可以透過修改提供給 `retry` 方法 callable 的請求引數來實現此目的。例如，如果第一次嘗試回傳認證錯誤，您可能希望使用新的授權 token 重試請求：

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

如果所有請求都失敗，將會拋出 `Illuminate\Http\Client\RequestException` 實例。如果您想禁用此行為，您可以提供 `throw` 引數的值為 `false`。禁用後，在所有重試嘗試完成後，客戶端收到的最後一個回應將會被回傳：

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> [!WARNING]
> 如果所有請求因連線問題而失敗，即使 `throw` 引數設定為 `false`，仍會拋出 `Illuminate\Http\Client\ConnectionException`。

<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP 客戶端封裝層不會在客戶端或伺服器錯誤（來自伺服器的 `400` 和 `500` 級別回應）時拋出例外。您可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否回傳了這些錯誤之一：

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

如果您有一個回應實例，並且希望在回應狀態碼表示客戶端或伺服器錯誤時拋出 `Illuminate\Http\Client\RequestException` 的實例，您可以使用 `throw` 或 `throwIf` 方法：

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

`Illuminate\Http\Client\RequestException` 實例有一個公開的 `$response` 屬性，可讓您檢查回傳的回應。

`throw` 方法在沒有發生錯誤時回傳回應實例，讓您可以將其他操作串接到 `throw` 方法上：

```php
return Http::post(/* ... */)->throw()->json();
```

如果您想在拋出例外之前執行一些額外的邏輯，您可以傳遞一個閉包給 `throw` 方法。在閉包被調用後，例外將會自動拋出，因此您不需要在閉包中重新拋出例外：

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

或者，您可以使用 `truncateExceptionsAt` 方法為每個請求自訂例外截斷行為：

```php
return Http::truncateExceptionsAt(240)->post(/* ... */);
```

<a name="guzzle-middleware"></a>
### Guzzle 中介層

由於 Laravel 的 HTTP 客戶端由 Guzzle 提供支援，您可以利用 [Guzzle 中介層](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來操作發出的請求或檢查收到的回應。若要操作發出的請求，可以透過 `withRequestMiddleware` 方法註冊一個 Guzzle 中介層：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withRequestMiddleware(
    function (RequestInterface $request) {
        return $request->withHeader('X-Example', 'Value');
    }
)->get('http://example.com');
```

同樣地，您可以透過 `withResponseMiddleware` 方法註冊一個中介層來檢查收到的 HTTP 回應：

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

有時候，您可能會希望註冊一個適用於每個發出請求和收到回應的中介層。若要實現此目的，您可以使用 `globalRequestMiddleware` 和 `globalResponseMiddleware` 方法。通常，這些方法應該在應用程式的 `AppServiceProvider` 裡的 `boot` 方法中呼叫：

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

若要為每個發出的請求配置預設選項，您可以使用 `globalOptions` 方法。通常，此方法應該在應用程式的 `AppServiceProvider` 裡的 `boot` 方法中呼叫：

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

有時，您可能希望併發地發出多個 HTTP 請求。換句話說，您希望同時派發多個請求，而不是依序發出請求。這在與緩慢的 HTTP API 互動時，可以大幅提升效能。

<a name="request-pooling"></a>
### 請求池

幸運的是，您可以使用 `pool` 方法來實現此目的。`pool` 方法接受一個閉包，該閉包會收到一個 `Illuminate\Http\Client\Pool` 實例，讓您可以輕鬆地將請求加入請求池中以供派發：

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

如您所見，每個回應實例都可以根據其被加入池中的順序來存取。如果您願意，可以使用 `as` 方法命名請求，這讓您可以根據名稱存取相對應的回應：

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
#### 客製化併發請求

`pool` 方法無法與其他 HTTP 客戶端方法鏈接，例如 `withHeaders` 或 `middleware` 方法。如果您想對池化請求應用客製化標頭或中介層，您應該在池中的每個請求上設定這些選項：

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

<a name="request-batching"></a>
### 請求批次處理

在 Laravel 中處理併發請求的另一種方式是使用 `batch` 方法。與 `pool` 方法類似，它接受一個閉包，該閉包會收到一個 `Illuminate\Http\Client\Batch` 實例，讓您可以輕鬆地將請求加入請求池中以供派發，但它也允許您定義完成回呼：

```php
use Illuminate\Http\Client\Batch;
use Illuminate\Http\Client\RequestException;
use Illuminate\Http\Client\Response;
use Illuminate\Support\Facades\Http;

$responses = Http::batch(fn (Batch $batch) => [
    $batch->get('http://localhost/first'),
    $batch->get('http://localhost/second'),
    $batch->get('http://localhost/third'),
])->before(function (Batch $batch) {
    // The batch has been created but no requests have been initialized...
})->progress(function (Batch $batch, int|string $key, Response $response) {
    // An individual request has completed successfully...
})->then(function (Batch $batch, array $results) {
    // All requests completed successfully...
})->catch(function (Batch $batch, int|string $key, Response|RequestException $response) {
    // First batch request failure detected...
})->finally(function (Batch $batch, array $results) {
    // The batch has finished executing...
})->send();
```

與 `pool` 方法一樣，您可以使用 `as` 方法命名您的請求：

```php
$responses = Http::batch(fn (Batch $batch) => [
    $batch->as('first')->get('http://localhost/first'),
    $batch->as('second')->get('http://localhost/second'),
    $batch->as('third')->get('http://localhost/third'),
])->send();
```

在呼叫 `send` 方法啟動 `batch` 後，您就不能再向其中新增請求。試圖這樣做將會導致拋出 `Illuminate\Http\Client\BatchInProgressException` 例外。

<a name="inspecting-batches"></a>
#### 檢查批次

提供給批次完成回呼的 `Illuminate\Http\Client\Batch` 實例擁有多種屬性與方法，以協助您與給定的請求批次互動並進行檢查：

```php
// The number of requests assigned to the batch...
$batch->totalRequests;
 
// The number of requests that have not been processed yet...
$batch->pendingRequests;
 
// The number of requests that have failed...
$batch->failedRequests;

// The number of requests that have been processed thus far...
$batch->processedRequests();

// Indicates if the batch has finished executing...
$batch->finished();

// Indicates if the batch has request failures...
$batch->hasFailures();
```

<a name="macros"></a>
## 巨集

Laravel HTTP 客戶端允許您定義「巨集」(macros)，這可以作為一種流暢、表達力強的機制，用於在應用程式中與服務互動時設定常見的請求路徑和標頭。首先，您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義 macro：

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

一旦您的 macro 設定完成，您就可以從應用程式中的任何地方呼叫它，以建立一個帶有指定設定的待處理請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了功能，可幫助您輕鬆且富有表達力地撰寫測試，Laravel 的 HTTP 客戶端也不例外。`Http` Facade 的 `fake` 方法允許您指示 HTTP 客戶端在發出請求時回傳模擬 / 虛擬回應。


<a name="faking-responses"></a>
### 模擬回應

例如，若要指示 HTTP 客戶端對每個請求回傳空的 `200` 狀態碼回應，您可以呼叫不帶任何引數的 `fake` 方法：

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```


<a name="faking-specific-urls"></a>
#### 模擬特定 URL

或者，您可以將一個陣列傳遞給 `fake` 方法。該陣列的鍵應代表您希望模擬的 URL 模式，以及它們相關的回應。`*` 字元可用作萬用字元。您可以使用 `Http` Facade 的 `response` 方法為這些端點建構模擬 / 虛擬回應：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // Stub a string response for Google endpoints...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

任何對尚未模擬的 URL 發出的請求，都將實際執行。如果您想指定一個回退（fallback）URL 模式來模擬所有不匹配的 URL，您可以使用單一的 `*` 字元：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // Stub a string response for all other endpoints...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

為了方便起見，您可以透過提供字串、陣列或整數作為回應，來產生簡單的字串、JSON 和空回應：

```php
Http::fake([
    'google.com/*' => 'Hello World',
    'github.com/*' => ['foo' => 'bar'],
    'chatgpt.com/*' => 200,
]);
```


<a name="faking-connection-exceptions"></a>
#### 模擬例外

有時，您可能需要測試應用程式的行為，如果 HTTP 客戶端在嘗試發出請求時遇到 `Illuminate\Http\Client\ConnectionException`。您可以指示 HTTP 客戶端使用 `failedConnection` 方法拋出連線例外：

```php
Http::fake([
    'github.com/*' => Http::failedConnection(),
]);
```

若要測試應用程式在拋出 `Illuminate\Http\Client\RequestException` 時的行為，您可以使用 `failedRequest` 方法：

```php
Http::fake([
    'github.com/*' => Http::failedRequest(['code' => 'not_found'], 404),
]);
```


<a name="faking-response-sequences"></a>
#### 模擬回應序列

有時您可能需要指定單一 URL 應按特定順序回傳一系列模擬回應。您可以使用 `Http::sequence` 方法來建構這些回應：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

當回應序列中的所有回應都已被消耗時，任何進一步的請求將導致回應序列拋出例外。如果您想指定一個預設回應，當序列為空時應回傳，您可以使用 `whenEmpty` 方法：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

如果您想模擬一系列回應，但不需要指定應該模擬的特定 URL 模式，您可以使用 `Http::fakeSequence` 方法：

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```


<a name="fake-callback"></a>
#### 模擬回呼

如果您需要更複雜的邏輯來決定對於某些端點要回傳什麼回應，您可以將一個閉包傳遞給 `fake` 方法。這個閉包將會收到一個 `Illuminate\Http\Client\Request` 實例，並且應該回傳一個回應實例。在您的閉包中，您可以執行任何必要的邏輯來決定要回傳哪種類型的回應：

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```


<a name="inspecting-requests"></a>
### 檢查請求

當模擬回應時，您偶爾可能希望檢查客戶端收到的請求，以確保您的應用程式正在發送正確的資料或標頭。您可以在呼叫 `Http::fake` 之後，透過呼叫 `Http::assertSent` 方法來實現此目的。

`assertSent` 方法接受一個閉包，該閉包將會收到一個 `Illuminate\Http\Client\Request` 實例，並應回傳一個布林值，指示請求是否符合您的預期。為了讓測試通過，至少一個請求必須已經發出且符合給定的預期：

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

如果需要，您可以使用 `assertNotSent` 方法斷言特定請求未被發送：

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

您可以使用 `assertSentCount` 方法來斷言在測試期間有多少請求被「發送」：

```php
Http::fake();

Http::assertSentCount(5);
```

或者，您可以使用 `assertNothingSent` 方法斷言在測試期間沒有請求被發送：

```php
Http::fake();

Http::assertNothingSent();
```


<a name="recording-requests-and-responses"></a>
#### 記錄請求 / 回應

您可以使用 `recorded` 方法來收集所有請求及其對應的回應。`recorded` 方法回傳一個包含 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 實例的陣列集合：

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
### 防止離題請求

如果您想確保透過 HTTP 客戶端發出的所有請求在您的個別測試或整個測試套件中都已模擬，您可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應模擬回應的請求將會拋出例外，而不是發出實際的 HTTP 請求：

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

有時，您可能希望阻止大多數離題請求，同時仍允許特定的請求執行。為此，您可以將 URL 模式的陣列傳遞給 `allowStrayRequests` 方法。任何符合指定模式的請求將被允許，而所有其他請求將繼續拋出例外：

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

Laravel 在傳送 HTTP 請求的過程中觸發三個事件。`RequestSending` 事件在請求被傳送前觸發，而 `ResponseReceived` 事件則在給定請求的回應被接收後觸發。`ConnectionFailed` 事件則在針對給定請求沒有收到回應時觸發。

`RequestSending` 和 `ConnectionFailed` 事件都包含一個公開的 `$request` 屬性，您可以用來檢查 `Illuminate\Http\Client\Request` 實例。同樣地，`ResponseReceived` 事件包含一個 `$request` 屬性以及一個 `$response` 屬性，可用來檢查 `Illuminate\Http\Client\Response` 實例。您可以在應用程式中為這些事件建立[事件監聽器](/docs/{{version}}/events)：

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