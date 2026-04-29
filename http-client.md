# HTTP Client

- [簡介](#introduction)
- [發送請求](#making-requests)
    - [請求資料](#request-data)
    - [標頭 (Headers)](#headers)
    - [認證](#authentication)
    - [逾時](#timeout)
    - [重試](#retries)
    - [錯誤處理](#error-handling)
    - [Guzzle 中介層](#guzzle-middleware)
    - [Guzzle 選項](#guzzle-options)
- [同時發送多個請求](#concurrent-requests)
    - [請求集區 (Request Pooling)](#request-pooling)
    - [請求批次處理 (Request Batching)](#request-batching)
- [Macros](#macros)
- [測試](#testing)
    - [模擬回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止非預期的請求 (Preventing Stray Requests)](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 提供了一個簡潔且富有表現力的 API，封裝了 [Guzzle HTTP client](http://docs.guzzlephp.org/en/stable/)，讓你能夠快速發送 HTTP 請求來與其他網頁應用程式進行通訊。Laravel 對 Guzzle 的封裝專注於最常見的使用情境，並提供絕佳的開發者體驗。

<a name="making-requests"></a>
## 發送請求

若要發送請求，你可以使用 `Http` Facade 提供的 `head`、`get`、`post`、`put`、`patch` 與 `delete` 方法。首先，讓我們看看如何對另一個 URL 發送一個基本的 `GET` 請求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('http://example.com');
```

`get` 方法會回傳一個 `Illuminate\Http\Client\Response` 的執行個體，該執行個體提供了多種可用於檢查回應的方法：

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

`Illuminate\Http\Client\Response` 物件也實作了 PHP 的 `ArrayAccess` 介面，讓你可以在回應上直接存取 JSON 回應資料：

```php
return Http::get('http://example.com/users/1')['name'];
```

除了上述列出的回應方法外，還可以使用以下方法來判斷回應是否具有特定的狀態碼：

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
#### URI 模板

HTTP 用戶端也允許你使用 [URI 模板規範 (URI template specification)](https://www.rfc-editor.org/rfc/rfc6570) 來建構請求 URL。若要定義可被 URI 模板展開的 URL 參數，你可以使用 `withUrlParameters` 方法：

```php
Http::withUrlParameters([
    'endpoint' => 'https://laravel.com',
    'page' => 'docs',
    'version' => '13.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```

<a name="dumping-requests"></a>
#### 傾印請求

如果你想在發送傳出請求執行個體之前將其傾印 (Dump) 並終止指令碼執行，可以在請求定義的開頭加上 `dd` 方法：

```php
return Http::dd()->get('http://example.com');
```

<a name="request-data"></a>
### 請求資料

當然，在發送 `POST`、`PUT` 與 `PATCH` 請求時，通常會隨請求發送額外的資料，因此這些方法接受一個資料陣列作為其第二個引數。預設情況下，資料將使用 `application/json` 內容類型 (Content Type) 發送：

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```

<a name="get-request-query-parameters"></a>
#### GET 請求查詢參數

發送 `GET` 請求時，你可以直接在 URL 後方附加查詢字串，或是將鍵值對 (Key / Value) 陣列作為 `get` 方法的第二個引數傳遞：

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
#### 發送表單 URL 編碼請求

如果你想使用 `application/x-www-form-urlencoded` 內容類型發送資料，應該在發送請求前呼叫 `asForm` 方法：

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```

<a name="sending-a-raw-request-body"></a>
#### 發送原始請求主體

如果你想在發送請求時提供原始 (Raw) 請求主體，可以使用 `withBody` 方法。內容類型可以透過該方法的第二個引數提供：

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```

<a name="multi-part-requests"></a>
#### 多部分請求 (Multi-Part Requests)

如果你想以多部分請求 (Multi-part Requests) 發送檔案，應該在發送請求前呼叫 `attach` 方法。此方法接受檔案名稱及其內容。如果需要，你可以提供第三個引數作為檔案的名稱 (Filename)，而第四個引數則可用於提供與檔案相關的標頭：

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
)->post('http://example.com/attachments');
```

除了傳遞檔案的原始內容外，你也可以傳遞串流 (Stream) 資源：

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```

<a name="headers"></a>
### 標頭 (Headers)

可以使用 `withHeaders` 方法將標頭加入到請求中。此 `withHeaders` 方法接受一個鍵值對陣列：

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

你可以使用 `accept` 方法來指定你的應用程式所期望的回應內容類型：

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

為了方便起見，你可以使用 `acceptJson` 方法快速指定應用程式期望回應為 `application/json` 內容類型：

```php
$response = Http::acceptJson()->get('http://example.com/users');
```

`withHeaders` 方法會將新標頭合併到請求現有的標頭中。如果需要，你可以使用 `replaceHeaders` 方法完全替換所有標頭：

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

你可以分別使用 `withBasicAuth` 和 `withDigestAuth` 方法來指定基本 (Basic) 和摘要 (Digest) 認證憑證：

```php
// Basic authentication...
$response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

// Digest authentication...
$response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);
```

<a name="bearer-tokens"></a>
#### Bearer 令牌

如果你想快速地將 Bearer 令牌加入到請求的 `Authorization` 標頭中，可以使用 `withToken` 方法：

```php
$response = Http::withToken('token')->post(/* ... */);
```

<a name="timeout"></a>
### 逾時

`timeout` 方法可用於指定等待回應的最大秒數。預設情況下，HTTP 用戶端將在 30 秒後逾時：

```php
$response = Http::timeout(3)->get(/* ... */);
```

如果超過指定的逾時時間，將會拋出 `Illuminate\Http\Client\ConnectionException` 的執行個體。

你可以使用 `connectTimeout` 方法指定嘗試連線到伺服器時等待的最大秒數。預設為 10 秒：

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>
### 重試

如果你希望 HTTP 客戶端在發生用戶端或伺服器錯誤時自動重試請求，你可以使用 `retry` 方法。`retry` 方法接受請求應嘗試的最大次數，以及 Laravel 在各次嘗試之間應等待的毫秒數：

```php
$response = Http::retry(3, 100)->post(/* ... */);
```

如果你想手動計算每次嘗試之間的等待毫秒數，可以將一個閉包 (closure) 作為第二個引數傳遞給 `retry` 方法：

```php
use Exception;

$response = Http::retry(3, function (int $attempt, Exception $exception) {
    return $attempt * 100;
})->post(/* ... */);
```

為了方便起見，你也可以提供一個陣列作為 `retry` 方法的第一個引數。此陣列將用於決定後續每次嘗試之間要等待的毫秒數：

```php
$response = Http::retry([100, 200])->post(/* ... */);
```

如果需要，你可以向 `retry` 方法傳遞第三個引數。第三個引數應該是一個可呼叫對象 (callable)，用來決定是否真的要進行重試。例如，你可能只想在初始請求遇到 `ConnectionException` 時才重試請求：

```php
use Illuminate\Http\Client\PendingRequest;
use Throwable;

$response = Http::retry(3, 100, function (Throwable $exception, PendingRequest $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

如果請求嘗試失敗，你可能希望在進行新嘗試之前對請求進行更改。你可以透過修改傳遞給 `retry` 方法之可呼叫對象的請求引數來達成此目的。例如，如果第一次嘗試回傳了認證錯誤，你可能想使用新的授權令牌 (authorization token) 來重試請求：

```php
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Http\Client\RequestException;
use Throwable;

$response = Http::withToken($this->getToken())->retry(2, 0, function (Throwable $exception, PendingRequest $request) {
    if (! $exception instanceof RequestException || $exception->response->status() !== 401) {
        return false;
    }

    $request->withToken($this->getNewToken());

    return true;
})->post(/* ... */);
```

如果所有請求都失敗，將會拋出 `Illuminate\Http\Client\RequestException` 實例。如果你想停用此行為，可以提供一個值為 `false` 的 `throw` 引數。停用後，在完成所有重試嘗試後，將回傳客戶端收到的最後一個回應：

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> [!WARNING]
> 如果所有請求都因為連線問題而失敗，即使 `throw` 引數被設定為 `false`，仍然會拋出 `Illuminate\Http\Client\ConnectionException`。


<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP 客戶端封裝層在發生用戶端或伺服器錯誤（來自伺服器的 `400` 和 `500` 層級回應）時不會拋出例外。你可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否回傳了其中一個錯誤：

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

如果你有一個回應實例，並希望在回應狀態碼指示用戶端或伺服器錯誤時拋出 `Illuminate\Http\Client\RequestException` 實例，你可以使用 `throw` 或 `throwIf` 方法：

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

// Throw an exception if a server error occurred (status >500)...
$response->throwIfServerError();

// Throw an exception if a client error occurred (status >400 and <500)...
$response->throwIfClientError();

return $response['user']['id'];
```

`Illuminate\Http\Client\RequestException` 實例有一個公開的 `$response` 屬性，讓你可以檢查回傳的回應。

如果沒有發生錯誤，`throw` 方法會回傳回應實例，讓你可以在 `throw` 方法之後串接其他操作：

```php
return Http::post(/* ... */)->throw()->json();
```

如果你想在拋出例外之前執行一些額外邏輯，你可以傳遞一個閉包給 `throw` 方法。例外會在該閉包被呼叫後自動拋出，因此你不需要在閉包內手動重新拋出例外：

```php
use Illuminate\Http\Client\Response;
use Illuminate\Http\Client\RequestException;

return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
    // ...
})->json();
```

預設情況下，`RequestException` 訊息在記錄或回報時會被縮減至 120 個字元。若要自訂或停用此行為，你可以在 `bootstrap/app.php` 檔案中設定應用程式的註冊行為時，利用 `truncateAt` 和 `dontTruncate` 方法：

```php
use Illuminate\Http\Client\RequestException;

->registered(function (): void {
    // Truncate request exception messages to 240 characters...
    RequestException::truncateAt(240);

    // Disable request exception message truncation...
    RequestException::dontTruncate();
})
```

或者，你也可以使用 `truncateExceptionsAt` 方法為每個請求自訂例外縮減行為：

```php
return Http::truncateExceptionsAt(240)->post(/* ... */);
```

<a name="guzzle-middleware"></a>
### Guzzle 中介層

由於 Laravel 的 HTTP 客戶端是由 Guzzle 驅動的，因此你可以利用 [Guzzle 中介層](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來修改發出的請求或檢查收到的回應。若要修改發出的請求，請透過 `withRequestMiddleware` 方法註冊一個 Guzzle 中介層：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withRequestMiddleware(
    function (RequestInterface $request) {
        return $request->withHeader('X-Example', 'Value');
    }
)->get('http://example.com');
```

同樣地，你可以透過 `withResponseMiddleware` 方法註冊一個中介層來檢查傳入的 HTTP 回應：

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

有時，你可能想要註冊一個適用於每個發出的請求和傳入的回應的中介層。要達成此目的，你可以使用 `globalRequestMiddleware` 與 `globalResponseMiddleware` 方法。通常，這些方法應該在你應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

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

你可以使用 `withOptions` 方法為發出的請求指定額外的 [Guzzle 請求選項](http://docs.guzzlephp.org/en/stable/request-options.html)。`withOptions` 方法接受一個鍵值對 (key / value pairs) 陣列：

```php
$response = Http::withOptions([
    'debug' => true,
])->get('http://example.com/users');
```


<a name="global-options"></a>
#### 全域選項

若要為每個發出的請求設定預設選項，你可以利用 `globalOptions` 方法。通常，此方法應該在你應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

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
## 同時發送多個請求

有時，你可能希望同時發送多個 HTTP 請求。換句話說，你希望多個請求同時發出，而不是依序發送。在與回應較慢的 HTTP API 互動時，這可以顯著提升效能。

<a name="request-pooling"></a>
### 請求集區 (Request Pooling)

幸運的是，你可以使用 `pool` 方法來達成此目的。`pool` 方法接受一個閉包 (Closure)，該閉包接收一個 `Illuminate\Http\Client\Pool` 實例，讓你能夠輕鬆地將請求加入請求集區中以進行發送：

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

如你所見，每個回應實例可以根據其加入集區的順序來存取。如果你願意，可以使用 `as` 方法為請求命名，這讓你可以透過名稱存取對應的回應：

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

請求集區的最大同時發送數量可以透過提供 `concurrency` 引數給 `pool` 方法來控制。此值決定了在處理請求集區時，可以同時進行中的 HTTP 請求最大數量：

```php
$responses = Http::pool(fn (Pool $pool) => [
    // ...
], concurrency: 5);
```

<a name="customizing-concurrent-requests"></a>
#### 自定義同時發送的請求

不可將 `pool` 方法與其他 HTTP Client 方法（例如 `withHeaders` 或 `middleware` 方法）鏈接使用。如果你想對集區內的請求套用自定義標頭或中介層，你應該在集區中的每個請求上分別配置這些選項：

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
### 請求批次處理 (Request Batching)

在 Laravel 中處理同時發送請求的另一種方式是使用 `batch` 方法。與 `pool` 方法類似，它接受一個接收 `Illuminate\Http\Client\Batch` 實例的閉包，讓你能夠輕鬆地將請求加入請求集區中發送，但它還允許你定義完成後的回呼函數 (Completion Callbacks)：

```php
use Illuminate\Http\Client\Batch;
use Illuminate\Http\Client\ConnectionException;
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
})->catch(function (Batch $batch, int|string $key, Response|RequestException|ConnectionException $response) {
    // Batch request failure detected...
})->finally(function (Batch $batch, array $results) {
    // The batch has finished executing...
})->send();
```

與 `pool` 方法一樣，你可以使用 `as` 方法為請求命名：

```php
$responses = Http::batch(fn (Batch $batch) => [
    $batch->as('first')->get('http://localhost/first'),
    $batch->as('second')->get('http://localhost/second'),
    $batch->as('third')->get('http://localhost/third'),
])->send();
```

在呼叫 `send` 方法啟動 `batch` 之後，你就不能再向其中添加新的請求。嘗試這樣做將導致拋出 `Illuminate\Http\Client\BatchInProgressException` 例外。

請求批次處理的最大同時發送數量可以透過 `concurrency` 方法控制。此值決定了在處理請求批次時，可以同時進行中的 HTTP 請求最大數量：

```php
$responses = Http::batch(fn (Batch $batch) => [
    // ...
])->concurrency(5)->send();
```

<a name="inspecting-batches"></a>
#### 檢查批次處理

提供給批次完成回呼的 `Illuminate\Http\Client\Batch` 實例擁有多種屬性和方法，可協助你與特定的請求批次進行互動及檢查：

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

<a name="deferring-batches"></a>
#### 延遲執行批次處理

當呼叫 `defer` 方法時，批次請求不會立即執行。相反地，Laravel 會在目前應用程式請求的 HTTP 回應發送給使用者後才執行該批次，讓你的應用程式保持快速的回應感：

```php
use Illuminate\Http\Client\Batch;
use Illuminate\Support\Facades\Http;

$responses = Http::batch(fn (Batch $batch) => [
    $batch->get('http://localhost/first'),
    $batch->get('http://localhost/second'),
    $batch->get('http://localhost/third'),
])->then(function (Batch $batch, array $results) {
    // All requests completed successfully...
})->defer();
```

<a name="macros"></a>
## Macros

Laravel HTTP Client 允許你定義「Macros」，這可以作為一種流暢且具表達性的機制，用來配置與整個應用程式中的服務互動時常見的請求路徑與標頭。要開始使用，你可以在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法內定義 Macro：

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

一旦配置了 Macro，你就可以在應用程式的任何地方呼叫它，以使用指定的配置建立一個待處理的請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了能協助你輕鬆且具表達力地撰寫測試的功能，Laravel 的 HTTP client 也不例外。`Http` facade 的 `fake` 方法讓你可以在發送請求時，指示 HTTP client 傳回模擬 (stubbed) 或虛構 (dummy) 的回應。

<a name="faking-responses"></a>
### 模擬回應

例如，若要指示 HTTP client 對於每個請求都傳回空的且狀態碼為 `200` 的回應，你可以呼叫不帶引數的 `fake` 方法：

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```

<a name="faking-specific-urls"></a>
#### 模擬特定 URL

或者，你可以向 `fake` 方法傳遞一個陣列。陣列的鍵 (key) 應代表你想要模擬的 URL 模式及其關聯的回應。`*` 字元可以用作通配符。你可以使用 `Http` facade 的 `response` 方法來為這些端點建構模擬的回應：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // Stub a string response for Google endpoints...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

任何發送到未被模擬的 URL 的請求都將被實際執行。如果你想指定一個備用的 URL 模式來模擬所有未匹配的 URL，可以使用單個 `*` 字元：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // Stub a string response for all other endpoints...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

為了方便起見，你可以藉由提供字串、陣列或整數作為回應，來產生簡單的字串、JSON 和空回應：

```php
Http::fake([
    'google.com/*' => 'Hello World',
    'github.com/*' => ['foo' => 'bar'],
    'chatgpt.com/*' => 200,
]);
```

<a name="faking-connection-exceptions"></a>
#### 模擬異常

有時你可能需要測試當 HTTP client 嘗試發送請求但遇到 `Illuminate\Http\Client\ConnectionException` 時，應用程式的行為。你可以使用 `failedConnection` 方法指示 HTTP client 拋出連線異常：

```php
Http::fake([
    'github.com/*' => Http::failedConnection(),
]);
```

若要測試當拋出 `Illuminate\Http\Client\RequestException` 時應用程式的行為，你可以使用 `failedRequest` 方法：

```php
$this->mock(GithubService::class);
    ->shouldReceive('getUser')
    ->andThrow(
        Http::failedRequest(['code' => 'not_found'], 404)
    );
```

<a name="faking-response-sequences"></a>
#### 模擬回應序列

有時你可能需要指定單個 URL 應按特定順序傳回一系列模擬回應。你可以使用 `Http::sequence` 方法來建構這些回應：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

當回應序列中的所有回應都已被消耗完畢時，任何後續請求都將導致該回應序列拋出異常。如果你想指定當序列為空時應傳回的預設回應，可以使用 `whenEmpty` 方法：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

如果你想模擬一系列回應，但不需要指定特定的 URL 模式，可以使用 `Http::fakeSequence` 方法：

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```

<a name="fake-callback"></a>
#### 模擬回呼

如果你需要更複雜的邏輯來決定某些端點應傳回什麼回應，可以向 `fake` 方法傳遞一個閉包。該閉包將接收一個 `Illuminate\Http\Client\Request` 實例並應傳回一個回應實例。在閉包內，你可以執行任何必要的邏輯來判斷要傳回哪種型別的回應：

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```

<a name="inspecting-requests"></a>
### 檢查請求

在模擬回應時，你偶爾可能希望檢查 client 接收到的請求，以確保你的應用程式正在發送正確的資料或標頭。你可以在呼叫 `Http::fake` 之後呼叫 `Http::assertSent` 方法來達成此目的。

`assertSent` 方法接受一個閉包，該閉包將接收一個 `Illuminate\Http\Client\Request` 實例，並應傳回一個布林值，指示該請求是否符合你的預期。為了讓測試通過，必須至少發出一個符合給定預期的請求：

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

如果需要，你可以使用 `assertNotSent` 方法斷言特定的請求未被發送：

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

你可以使用 `assertSentCount` 方法斷言測試期間「發送」了多少個請求：

```php
Http::fake();

Http::assertSentCount(5);
```

或者，你可以使用 `assertNothingSent` 方法斷言測試期間沒有發送任何請求：

```php
Http::fake();

Http::assertNothingSent();
```

<a name="recording-requests-and-responses"></a>
#### 記錄請求與回應

你可以使用 `recorded` 方法收集所有請求及其對應的回應。`recorded` 方法會傳回一個陣列集合，其中包含 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例：

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

此外，`recorded` 方法接受一個閉包，該閉包將接收 `Illuminate\Http\Client\Request` 和 `Illuminate\Http\Client\Response` 的實例，並可用於根據你的預期過濾請求/回應對：

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
### 防止非預期的請求 (Preventing Stray Requests)

如果您想確保在個別測試或整個測試套件中，所有透過 HTTP 用戶端發送的請求都已被模擬，您可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應模擬回應的請求都將拋出例外，而不是發送實際的 HTTP 請求：

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

有時候，您可能希望防止大多數非預期的請求，但仍允許特定請求執行。若要達成此目的，您可以將一個包含 URL 模式的陣列傳遞給 `allowStrayRequests` 方法。任何符合給定模式之一的請求都將被允許，而所有其他請求則會繼續拋出例外：

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

Laravel 在發送 HTTP 請求的過程中會觸發三個事件。`RequestSending` 事件在請求發送之前觸發，而 `ResponseReceived` 事件則在收到指定請求的回應後觸發。若指定請求未收到任何回應，則會觸發 `ConnectionFailed` 事件。

`RequestSending` 與 `ConnectionFailed` 事件都包含一個公開的 `$request` 屬性，您可以用它來檢查 `Illuminate\Http\Client\Request` 執行個體。同樣地，`ResponseReceived` 事件也包含 `$request` 屬性以及 `$response` 屬性，可用於檢查 `Illuminate\Http\Client\Response` 執行個體。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

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