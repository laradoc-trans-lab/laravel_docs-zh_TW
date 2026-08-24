# HTTP 用戶端

- [簡介](#introduction)
- [發送請求](#making-requests)
    - [請求資料](#request-data)
    - [標頭](#headers)
    - [認證](#authentication)
    - [逾時](#timeout)
    - [重試](#retries)
    - [錯誤處理](#error-handling)
    - [Guzzle 中介層](#guzzle-middleware)
    - [Guzzle 選項](#guzzle-options)
- [同時發送請求](#concurrent-requests)
    - [請求池](#request-pooling)
    - [請求批次處理](#request-batching)
- [巨集](#macros)
- [測試](#testing)
    - [偽造回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止未模擬的請求](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 圍繞著 [Guzzle HTTP 用戶端](http://docs.guzzlephp.org/en/stable/) 提供了一套具表達力且極簡的 API，讓你能快速發送對外的 HTTP 請求，以與其他 Web 應用程式進行溝通。Laravel 對 Guzzle 的這層封裝專注於最常見的使用場景，並提供極佳的開發者體驗。

<a name="making-requests"></a>
## 發送請求

要發送請求，你可以使用 `Http` Facade 提供的 `head`、`get`、`post`、`put`、`patch` 與 `delete` 方法。首先，讓我們看看如何向另一個 URL 發送基本的 `GET` 請求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::get('http://example.com');
```

`get` 方法會回傳一個 `Illuminate\Http\Client\Response` 的實例，該實例提供了多種可用於檢查回應的方法：

```php
$response->body() : string;
$response->json($key = null, $default = null, $flags = null) : mixed;
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

`Illuminate\Http\Client\Response` 物件也實作了 PHP 的 `ArrayAccess` 介面，讓你能夠直接在回應上存取 JSON 回應資料：

```php
return Http::get('http://example.com/users/1')['name'];
```

除了上述列出的回應方法外，還可以使用以下方法來判定回應是否具有特定的狀態碼：

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

HTTP 用戶端還允許你使用 [URI 模板規範](https://www.rfc-editor.org/rfc/rfc6570) 來建構請求 URL。若要定義可由 URI 模板展開的 URL 參數，你可以使用 `withUrlParameters` 方法：

```php
Http::withUrlParameters([
    'endpoint' => 'https://laravel.com',
    'page' => 'docs',
    'version' => '13.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```


<a name="dumping-requests"></a>
#### 印出請求

如果你想在發送對外請求實例之前將其印出並終止腳本的執行，可以在請求定義的開頭加上 `dd` 方法：

```php
return Http::dd()->get('http://example.com');
```


<a name="request-data"></a>
### 請求資料

當然，在發送 `POST`、`PUT` 和 `PATCH` 請求時，隨請求發送額外資料是很常見的，因此這些方法接受一個資料陣列作為其第二個引數。預設情況下，資料將使用 `application/json` Content-Type 發送：

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```


<a name="get-request-query-parameters"></a>
#### GET 請求查詢參數

發送 `GET` 請求時，你可以直接在 URL 後面附加查詢字串，也可以將鍵 / 值對的陣列作為第二個引數傳遞給 `get` 方法：

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
#### 發送 Form URL Encoded 請求

如果你想使用 `application/x-www-form-urlencoded` Content-Type 發送資料，應該在發送請求之前呼叫 `asForm` 方法：

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```


<a name="sending-a-raw-request-body"></a>
#### 發送原始請求主體

如果你想在發送請求時提供原始的請求主體 (Raw request body)，可以使用 `withBody` 方法。Content-Type 可以透過該方法的第二個引數提供：

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```


<a name="multi-part-requests"></a>
#### Multi-Part 請求

如果你想將檔案作為 Multi-Part 請求發送，應該在發送請求之前呼叫 `attach` 方法。此方法接受檔案名稱及其內容。如果需要，你可以提供第三個引數作為該檔案的檔名，而第四個引數則可用於提供與該檔案關聯的標頭：

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
)->post('http://example.com/attachments');
```

除了傳遞檔案的原始內容外，你也可以傳遞串流資源 (Stream resource)：

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```


<a name="headers"></a>
### 標頭

可以使用 `withHeaders` 方法將標頭新增至請求中。此 `withHeaders` 方法接受一個鍵 / 值對陣列：

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

你可以使用 `accept` 方法來指定你的應用程式在回應請求時所期望的 Content-Type：

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

為方便起見，你可以使用 `acceptJson` 方法快速指定你的應用程式期望在回應請求時收到 `application/json` Content-Type：

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

你可以分別使用 `withBasicAuth` 與 `withDigestAuth` 方法來指定 Basic 與 Digest 認證憑證：

```php
// Basic authentication...
$response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

// Digest authentication...
$response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);
```


<a name="bearer-tokens"></a>
#### Bearer Token

如果你想快速將 Bearer Token 新增至請求的 `Authorization` 標頭中，可以使用 `withToken` 方法：

```php
$response = Http::withToken('token')->post(/* ... */);
```


<a name="timeout"></a>
### 逾時

`timeout` 方法可用於指定等待回應的最長秒數。預設情況下，HTTP 用戶端會在 30 秒後逾時：

```php
$response = Http::timeout(3)->get(/* ... */);
```

如果超過給定的逾時時間，將會拋出 `Illuminate\Http\Client\ConnectionException` 實例。

你可以使用 `connectTimeout` 方法指定嘗試連接伺服器時的最長等待秒數。預設為 10 秒：

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>
### 重試

若您希望 HTTP 用戶端在發生用戶端或伺服器錯誤時自動重試請求，您可以使用 `retry` 方法。`retry` 方法接收請求應該嘗試的最大次數，以及 Laravel 在兩次嘗試之間應該等待的毫秒數：

```php
$response = Http::retry(3, 100)->post(/* ... */);
```

若您想手動計算每次嘗試之間要暫停的毫秒數，您可以傳遞一個閉包作為 `retry` 方法的第二個引數：

```php
use Exception;

$response = Http::retry(3, function (int $attempt, Exception $exception) {
    return $attempt * 100;
})->post(/* ... */);
```

為方便起見，您也可以提供一個陣列作為 `retry` 方法的第一個引數。此陣列將用於決定後續每次嘗試之間要暫停多少毫秒：

```php
$response = Http::retry([100, 200])->post(/* ... */);
```

如果有需要，您可以傳遞第三個引數給 `retry` 方法。第三個引數應該是一個可呼叫函式，用來決定是否真的要嘗試重試。例如，您可能只想在初始請求遇到 `ConnectionException` 時才進行重試：

```php
use Illuminate\Http\Client\PendingRequest;
use Throwable;

$response = Http::retry(3, 100, function (Throwable $exception, PendingRequest $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

若請求嘗試失敗，您可能希望在進行新的嘗試之前修改請求。您可以透過修改傳給提供給 `retry` 方法的可呼叫函式的請求引數來達成此目的。例如，若第一次嘗試回傳認證錯誤，您可能希望使用新的授權令牌重試請求：

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

若所有請求皆失敗，系統將拋出一個 `Illuminate\Http\Client\RequestException` 實例。若您想停用此行為，可以提供值為 `false` 的 `throw` 引數。停用時，在嘗試完所有重試後，將回傳用戶端收到的最後一個回應：

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> [!WARNING]
> 若所有請求皆因連線問題而失敗，即使將 `throw` 引數設為 `false`，仍會拋出 `Illuminate\Http\Client\ConnectionException`。

<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP 用戶端包裝器不會在發生用戶端或伺服器錯誤（來自伺服器的 `400` 和 `500` 層級回應）時拋出例外。您可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否回傳了這些錯誤：

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

若您有一個回應實例，且希望在回應狀態碼指示為用戶端或伺服器錯誤時拋出 `Illuminate\Http\Client\RequestException` 實例，您可以使用 `throw` 或 `throwIf` 方法：

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

`Illuminate\Http\Client\RequestException` 實例包含一個公開的 `$response` 屬性，允許您檢查回傳的回應。

若未發生錯誤，`throw` 方法會回傳回應實例，允許您在 `throw` 方法之後鏈結其他操作：

```php
return Http::post(/* ... */)->throw()->json();
```

若您希望在拋出例外之前執行一些額外的邏輯，您可以傳遞一個閉包給 `throw` 方法。在閉包被呼叫後，例外將自動被拋出，因此您不需要在閉包內部重新拋出該例外：

```php
use Illuminate\Http\Client\Response;
use Illuminate\Http\Client\RequestException;

return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
    // ...
})->json();
```

預設情況下，當記錄或回報 `RequestException` 訊息時，該訊息會截斷為 120 個字元。若要自訂或停用此行為，您可以在 `bootstrap/app.php` 檔案中設定應用程式的註冊行為時，使用 `truncateAt` 與 `dontTruncate` 方法：

```php
use Illuminate\Http\Client\RequestException;

->registered(function (): void {
    // Truncate request exception messages to 240 characters...
    RequestException::truncateAt(240);

    // Disable request exception message truncation...
    RequestException::dontTruncate();
})
```

或者，您也可以使用 `truncateExceptionsAt` 方法針對單一請求自訂例外截斷行為：

```php
return Http::truncateExceptionsAt(240)->post(/* ... */);
```

<a name="guzzle-middleware"></a>
### Guzzle 中介層

由於 Laravel 的 HTTP 用戶端是由 Guzzle 所驅動，因此您可以利用 [Guzzle 中介層](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來修改發出的請求或檢查收到的回應。若要修改發出的請求，可以透過 `withRequestMiddleware` 方法來註冊 Guzzle 中介層：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withRequestMiddleware(
    function (RequestInterface $request) {
        return $request->withHeader('X-Example', 'Value');
    }
)->get('http://example.com');
```

同樣地，您可以透過 `withResponseMiddleware` 方法註冊中介層來檢查收到的 HTTP 回應：

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

有時候，您可能希望註冊一個套用到每個發出請求與收到回應的中介層。為此，您可以使用 `globalRequestMiddleware` 與 `globalResponseMiddleware` 方法。通常，這些方法應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法內呼叫：

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

您可以使用 `withOptions` 方法為發出的請求指定額外的 [Guzzle 請求選項](http://docs.guzzlephp.org/en/stable/request-options.html)。`withOptions` 方法接受一個鍵值對陣列：

```php
$response = Http::withOptions([
    'debug' => true,
])->get('http://example.com/users');
```

<a name="global-options"></a>
#### 全域選項

若要為每個發出的請求設定預設選項，您可以使用 `globalOptions` 方法。通常，這個方法應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法內呼叫：

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
## 同時發送請求

有時候，您可能希望同時發送多個 HTTP 請求。換句話說，您希望同時發送數個請求，而不是按順序依次發送。這在與回應較慢的 HTTP API 進行互動時，可以帶來顯著的效能提升。


<a name="request-pooling"></a>
### 請求池

幸運的是，您可以使用 `pool` 方法來達成這個目的。`pool` 方法接受一個接收 `Illuminate\Http\Client\Pool` 實例的閉包，讓您可以輕鬆地將請求新增至請求池中以進行發送：

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

如您所見，每個回應實例都可以依據加入請求池的順序來存取。若有需要，您也可以使用 `as` 方法為請求命名，這樣就能透過名稱存取對應的回應：

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

您可以透過向 `pool` 方法提供 `concurrency` 引數來控制請求池的最大同時處理數量。此數值決定了處理請求池時，最多可以同時進行中的 HTTP 請求數量：

```php
$responses = Http::pool(fn (Pool $pool) => [
    // ...
], concurrency: 5);
```


<a name="customizing-concurrent-requests"></a>
#### 自訂同時發送的請求

`pool` 方法無法與其他 HTTP 用戶端方法（例如 `withHeaders` 或 `middleware` 方法）進行鏈結呼叫。如果您想對請求池內的請求套用自訂標頭或中介層，應該在請求池中的每個請求上獨立設定這些選項：

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

在 Laravel 中處理同時請求的另一種方式是使用 `batch` 方法。就像 `pool` 方法一樣，它接受一個接收 `Illuminate\Http\Client\Batch` 實例的閉包，讓您可以輕鬆新增請求至請求池進行發送，但它還允許您定義完成後的回呼函式：

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

如同 `pool` 方法，您可以使用 `as` 方法為您的請求命名：

```php
$responses = Http::batch(fn (Batch $batch) => [
    $batch->as('first')->get('http://localhost/first'),
    $batch->as('second')->get('http://localhost/second'),
    $batch->as('third')->get('http://localhost/third'),
])->send();
```

在呼叫 `send` 方法啟動 `batch` 之後，您就無法再新增新的請求。如果嘗試這麼做，將會拋出 `Illuminate\Http\Client\BatchInProgressException` 例外。

請求批次的最大同時處理數量可以透過 `concurrency` 方法來控制。此數值決定了處理請求批次時，最多可以同時進行中的 HTTP 請求數量：

```php
$responses = Http::batch(fn (Batch $batch) => [
    // ...
])->concurrency(5)->send();
```


<a name="inspecting-batches"></a>
#### 檢查批次

傳遞給批次完成回呼函式的 `Illuminate\Http\Client\Batch` 實例擁有多種屬性與方法，可協助您與指定的請求批次進行互動及檢查：

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
#### 延遲執行批次

當呼叫 `defer` 方法時，批次請求不會立即執行。相反地，Laravel 會在將當前應用程式請求的 HTTP 回應發送給使用者之後才執行該批次，從而保持應用程式的高速運作與即時回應感：

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
## 巨集

Laravel HTTP 用戶端允許您定義「巨集」，這可以作為一種順暢且具表現力的機制，用來在整個應用程式中與各種服務互動時設定常見的請求路徑和標頭。首先，您可以在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法內定義巨集：

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

設定好巨集後，您可以在應用程式中的任何地方呼叫它，以建立帶有指定設定的待處理請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了能幫助你輕鬆且具表達力地撰寫測試的功能，Laravel 的 HTTP 用戶端也不例外。`Http` Facade 的 `fake` 方法允許你指示 HTTP 用戶端在發送請求時傳回預擬 (Stubbed) / 虛擬回應。

<a name="faking-responses"></a>
### 偽造回應

舉例來說，若要指示 HTTP 用戶端對每個請求都傳回空白且狀態碼為 `200` 的回應，你可以在呼叫 `fake` 方法時不傳入任何引數：

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```

<a name="faking-specific-urls"></a>
#### 偽造特定 URL

此外，你也可以向 `fake` 方法傳入一個陣列。陣列的鍵 (Key) 應代表你希望偽造的 URL 樣式 (Pattern) 以及與其關聯的回應。`*` 字元可以用作萬用字元。你可以使用 `Http` Facade 的 `response` 方法來為這些端點 (Endpoint) 建構預擬 (Stub) / 虛擬回應：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // Stub a string response for Google endpoints...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

任何對未被偽造的 URL 發出的請求都將實際執行。如果你想指定一個備用 (Fallback) 的 URL 樣式來預擬所有未匹配的 URL，可以使用單個 `*` 字元：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // Stub a string response for all other endpoints...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

為了方便起見，你可以直接提供字串、陣列或整數作為回應，藉此產生簡單的字串、JSON 與空白回應：

```php
Http::fake([
    'google.com/*' => 'Hello World',
    'github.com/*' => ['foo' => 'bar'],
    'chatgpt.com/*' => 200,
]);
```

<a name="faking-connection-exceptions"></a>
#### 偽造例外

有時，你可能需要測試應用程式在 HTTP 用戶端嘗試發送請求時遇到 `Illuminate\Http\Client\ConnectionException` 的行為。你可以使用 `failedConnection` 方法指示 HTTP 用戶端拋出連線例外：

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
#### 偽造回應序列

有時你可能需要指定單一 URL 依特定順序傳回一連串的偽造回應。你可以使用 `Http::sequence` 方法來建立回應：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

當回應序列中的所有回應都被用盡時，任何後續的請求都將導致該回應序列拋出例外。如果你想指定序列為空時應傳回的預設回應，可以使用 `whenEmpty` 方法：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

如果你想偽造一連串的回應，但不需要指定特定要偽造的 URL 樣式，可以使用 `Http::fakeSequence` 方法：

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```

<a name="fake-callback"></a>
#### 偽造 Callback

若你需要更複雜的邏輯來決定特定端點要傳回什麼回應，可以向 `fake` 方法傳入一個 Closure。該 Closure 將接收一個 `Illuminate\Http\Client\Request` 實例，並應傳回一個回應實例。在你的 Closure 中，你可以執行任何必要的邏輯來判斷要傳回哪種類型的回應：

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```

<a name="inspecting-requests"></a>
### 檢查請求

在偽造回應時，你偶爾會希望檢查用戶端收到的請求，以確保你的應用程式發送了正確的資料或標頭。你可以藉由在呼叫 `Http::fake` 之後呼叫 `Http::assertSent` 方法來達成此目的。

`assertSent` 方法接受一個 Closure，該 Closure 會接收一個 `Illuminate\Http\Client\Request` 實例，並應傳回一個布林值，用以說明該請求是否符合你的預期的需求。為了讓測試通過，必須至少發送了一個符合給定預期的請求：

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

如有需要，你可以使用 `assertNotSent` 方法來斷言特定請求未被發送：

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

你可以使用 `assertSentCount` 方法來斷言測試期間「發送」了多少個請求：

```php
Http::fake();

Http::assertSentCount(5);
```

或者，你可以使用 `assertNothingSent` 方法來斷言測試期間沒有發送任何請求：

```php
Http::fake();

Http::assertNothingSent();
```

<a name="recording-requests-and-responses"></a>
#### 記錄請求 / 回應

你可以使用 `recorded` 方法來收集所有請求及其對應的回應。`recorded` 方法會傳回一個由陣列組成的集合 (Collection)，其中包含 `Illuminate\Http\Client\Request` 與 `Illuminate\Http\Client\Response` 的實例：

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

此外，`recorded` 方法也接受一個 Closure，該 Closure 將接收 `Illuminate\Http\Client\Request` 與 `Illuminate\Http\Client\Response` 的實例，可用於根據你的預期過濾請求 / 回應組合：

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
### 防止未模擬的請求

若您想確保在單一測試或整個測試套件中，透過 HTTP 用戶端發送的所有請求都已被偽造，您可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應偽造回應的請求都將拋出例外，而非發送實際的 HTTP 請求：

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

有時，您可能希望防止大多數未模擬的請求，但仍允許特定的請求執行。若要達成此目的，您可以傳遞包含 URL 樣式的陣列給 `allowStrayRequests` 方法。任何符合給定樣式之一的請求都將被允許執行，而所有其他請求則會繼續拋出例外：

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

Laravel 在發送 HTTP 請求的過程中會觸發三個事件。`RequestSending` 事件會在發送請求之前觸發，而 `ResponseReceived` 事件會在收到特定請求的回應後觸發。如果在特定請求中沒有收到任何回應，則會觸發 `ConnectionFailed` 事件。

`RequestSending` 與 `ConnectionFailed` 事件都包含一個公開的 `$request` 屬性，您可以用它來檢查 `Illuminate\Http\Client\Request` 實例。同樣地，`ResponseReceived` 事件包含一個 `$request` 屬性以及一個 `$response` 屬性，可用於檢查 `Illuminate\Http\Client\Response` 實例。您可以在應用程式中為這些事件建立[事件監聽器](/docs/{{version}}/events)：

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