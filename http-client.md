# HTTP Client

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
    - [請求分批](#request-batching)
- [巨集](#macros)
- [測試](#testing)
    - [偽造回應](#faking-responses)
    - [檢查請求](#inspecting-requests)
    - [防止未預期的請求](#preventing-stray-requests)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Laravel 圍繞著 [Guzzle HTTP 用戶端](http://docs.guzzlephp.org/en/stable/) 提供了一套簡潔且富有表達力的 API，讓你能夠快速發送對外 HTTP 請求，與其他 Web 應用程式進行溝通。Laravel 對 Guzzle 所做的包裝專注於最常見的使用情境，並提供了極佳的開發者體驗。

<a name="making-requests"></a>
## 發送請求

要發送請求，你可以使用 `Http` Facade 提供的 `head`、`get`、`post`、`put`、`patch` 和 `delete` 方法。首先，讓我們看看如何向另一個 URL 發送基本的 `GET` 請求：

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

`Illuminate\Http\Client\Response` 物件也實作了 PHP 的 `ArrayAccess` 介面，讓你能夠直接在回應物件上存取 JSON 回應資料：

```php
return Http::get('http://example.com/users/1')['name'];
```

除了上述列出的回應方法之外，還可以使用以下方法來判斷回應是否具有特定的狀態碼：

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

HTTP 用戶端還允許你使用 [URI 範本規格](https://www.rfc-editor.org/rfc/rfc6570) 來構建請求的 URL。若要定義可由 URI 範本展開的 URL 參數，你可以使用 `withUrlParameters` 方法：

```php
Http::withUrlParameters([
    'endpoint' => 'https://laravel.com',
    'page' => 'docs',
    'version' => '13.x',
    'topic' => 'validation',
])->get('{+endpoint}/{page}/{version}/{topic}');
```

<a name="dumping-requests"></a>
#### 印出請求內容

如果你想在發送請求之前印出即將發送的請求實例並終止指令碼的執行，可以在定義請求時於最前面加上 `dd` 方法：

```php
return Http::dd()->get('http://example.com');
```

<a name="request-data"></a>
### 請求資料

當然，在發送 `POST`、`PUT` 與 `PATCH` 請求時，通常會隨請求發送額外資料，因此這些方法接受一個資料陣列作為其第二個引數。預設情況下，資料將使用 `application/json` 內容型態 (Content Type) 發送：

```php
use Illuminate\Support\Facades\Http;

$response = Http::post('http://example.com/users', [
    'name' => 'Steve',
    'role' => 'Network Administrator',
]);
```

<a name="get-request-query-parameters"></a>
#### GET 請求查詢參數

在發送 `GET` 請求時，你可以直接在 URL 後面附加查詢字串，或是將鍵值對 (key / value pairs) 陣列作為第二個引數傳遞給 `get` 方法：

```php
$response = Http::get('http://example.com/users', [
    'name' => 'Taylor',
    'page' => 1,
]);
```

另外，也可以使用 `withQueryParameters` 方法：

```php
Http::retry(3, 100)->withQueryParameters([
    'name' => 'Taylor',
    'page' => 1,
])->get('http://example.com/users');
```

<a name="sending-form-url-encoded-requests"></a>
#### 發送 Form URL Encoded 請求

如果你想使用 `application/x-www-form-urlencoded` 內容型態來發送資料，應該在發送請求前呼叫 `asForm` 方法：

```php
$response = Http::asForm()->post('http://example.com/users', [
    'name' => 'Sara',
    'role' => 'Privacy Consultant',
]);
```

<a name="sending-a-raw-request-body"></a>
#### 發送原始請求主體

如果你想在發送請求時提供原始的請求內容 (Raw Body)，可以使用 `withBody` 方法。內容型態可透過該方法的第二個引數來指定：

```php
$response = Http::withBody(
    base64_encode($photo), 'image/jpeg'
)->post('http://example.com/photo');
```

<a name="multi-part-requests"></a>
#### Multi-Part 請求

如果你想以 multi-part 請求形式發送檔案，應該在發送請求前呼叫 `attach` 方法。此方法接受檔案欄位名稱與其內容。如有需要，你可以提供第三個引數作為檔案的檔名，而第四個引數則可用於提供與該檔案關聯的標頭：

```php
$response = Http::attach(
    'attachment', file_get_contents('photo.jpg'), 'photo.jpg', ['Content-Type' => 'image/jpeg']
)->post('http://example.com/attachments');
```

除了傳遞檔案的原始內容外，你也可以傳入串流資源 (Stream Resource)：

```php
$photo = fopen('photo.jpg', 'r');

$response = Http::attach(
    'attachment', $photo, 'photo.jpg'
)->post('http://example.com/attachments');
```

<a name="headers"></a>
### 標頭

可以使用 `withHeaders` 方法將標頭新增至請求中。`withHeaders` 方法接受鍵值對陣列：

```php
$response = Http::withHeaders([
    'X-First' => 'foo',
    'X-Second' => 'bar'
])->post('http://example.com/users', [
    'name' => 'Taylor',
]);
```

你可以使用 `accept` 方法來指定應用程式期望回應此請求的內容型態：

```php
$response = Http::accept('application/json')->get('http://example.com/users');
```

為方便起見，你可以使用 `acceptJson` 方法快速指定應用程式期望回應的內容型態為 `application/json`：

```php
$response = Http::acceptJson()->get('http://example.com/users');
```

`withHeaders` 方法會將新標頭合併至請求現有的標頭中。如有需要，你可以使用 `replaceHeaders` 方法完全替換所有的標頭：

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

你可以分別使用 `withBasicAuth` 和 `withDigestAuth` 方法來指定 Basic 與 Digest 認證憑證：

```php
// Basic authentication...
$response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(/* ... */);

// Digest authentication...
$response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(/* ... */);
```

<a name="bearer-tokens"></a>
#### Bearer Tokens

如果你想快速在請求的 `Authorization` 標頭中加入 Bearer Token，可以使用 `withToken` 方法：

```php
$response = Http::withToken('token')->post(/* ... */);
```

<a name="timeout"></a>
### 逾時

`timeout` 方法可以用來指定等待回應的最大秒數。預設情況下，HTTP 用戶端會在 30 秒後逾時：

```php
$response = Http::timeout(3)->get(/* ... */);
```

如果超過給定的逾時時間，系統將拋出 `Illuminate\Http\Client\ConnectionException` 實例。

你可以使用 `connectTimeout` 方法指定嘗試連線至伺服器時的最長等待秒數。預設值為 10 秒：

```php
$response = Http::connectTimeout(3)->get(/* ... */);
```

<a name="retries"></a>
### 重試

如果您希望 HTTP 客戶端在發生客戶端或伺服器錯誤時自動重試請求，您可以使用 `retry` 方法。`retry` 方法接收請求嘗試的最大次數，以及 Laravel 在每次嘗試之間應該等待的毫秒數：

```php
$response = Http::retry(3, 100)->post(/* ... */);
```

如果您想手動計算每次嘗試之間要暫停的毫秒數，可以傳遞一個閉包作為 `retry` 方法的第二個引數：

```php
use Exception;

$response = Http::retry(3, function (int $attempt, Exception $exception) {
    return $attempt * 100;
})->post(/* ... */);
```

為了方便起見，您也可以提供一個陣列作為 `retry` 方法的第一個引數。該陣列將用於決定後續嘗試之間要暫停的毫秒數：

```php
$response = Http::retry([100, 200])->post(/* ... */);
```

如果需要，您可以向 `retry` 方法傳遞第三個引數。第三個引數應該是一個 Callable（可呼叫物件），用來判斷是否真的應該進行重試。例如，您可能希望僅在初始請求遇到 `ConnectionException` 時才重試該請求：

```php
use Illuminate\Http\Client\PendingRequest;
use Throwable;

$response = Http::retry(3, 100, function (Throwable $exception, PendingRequest $request) {
    return $exception instanceof ConnectionException;
})->post(/* ... */);
```

如果某次請求嘗試失敗，您可能希望在進行新的嘗試前對請求進行修改。您可以透過修改傳遞給 `retry` 方法之 Callable 的請求引數來達成此目的。例如，如果第一次嘗試回傳了認證錯誤，您可能希望使用新的授權令牌重試請求：

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

如果所有請求皆失敗，將會拋出 `Illuminate\Http\Client\RequestException` 實例。如果您想停用此行為，可以提供值為 `false` 的 `throw` 引數。停用時，在嘗試所有重試之後，將會回傳客戶端接收到的最後一個回應：

```php
$response = Http::retry(3, 100, throw: false)->post(/* ... */);
```

> [!WARNING]
> 如果所有請求都因為連線問題而失敗，即使將 `throw` 引數設定為 `false`，仍然會拋出 `Illuminate\Http\Client\ConnectionException`。

<a name="error-handling"></a>
### 錯誤處理

與 Guzzle 的預設行為不同，Laravel 的 HTTP 客戶端包裝器不會在發生客戶端或伺服器錯誤（來自伺服器的 `400` 和 `500` 層級回應）時拋出例外。您可以使用 `successful`、`clientError` 或 `serverError` 方法來判斷是否回傳了這些錯誤之一：

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

如果您有一個回應實例，且當回應狀態碼表示客戶端或伺服器錯誤時希望拋出 `Illuminate\Http\Client\RequestException` 實例，您可以使用 `throw` 或 `throwIf` 方法：

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

`Illuminate\Http\Client\RequestException` 實例有一個公開的 `$response` 屬性，讓您可以檢查回傳的回應。

如果沒有發生錯誤，`throw` 方法會回傳回應實例，允許您在 `throw` 方法後鏈結其他操作：

```php
return Http::post(/* ... */)->throw()->json();
```

如果您希望在拋出例外之前執行一些額外邏輯，可以傳遞一個閉包給 `throw` 方法。在閉包被呼叫後，例外將會自動被拋出，因此您不需要在閉包內重新拋出例外：

```php
use Illuminate\Http\Client\Response;
use Illuminate\Http\Client\RequestException;

return Http::post(/* ... */)->throw(function (Response $response, RequestException $e) {
    // ...
})->json();
```

預設情況下，當記錄（log）或通報 `RequestException` 時，其訊息會被截斷為 120 個字元。若要自訂或停用此行為，可以在 `bootstrap/app.php` 檔案中設定應用程式的註冊行為時使用 `truncateAt` 與 `dontTruncate` 方法：

```php
use Illuminate\Http\Client\RequestException;

->registered(function (): void {
    // Truncate request exception messages to 240 characters...
    RequestException::truncateAt(240);

    // Disable request exception message truncation...
    RequestException::dontTruncate();
})
```

或者，您可以使用 `truncateExceptionsAt` 方法針對個別請求自訂例外截斷行為：

```php
return Http::truncateExceptionsAt(240)->post(/* ... */);
```

<a name="guzzle-middleware"></a>
### Guzzle 中介層

由於 Laravel 的 HTTP 用戶端是由 Guzzle 所驅動，您可以利用 [Guzzle 中介層](https://docs.guzzlephp.org/en/stable/handlers-and-middleware.html) 來修改發送的請求或檢查接收到的回應。若要修改發送的請求，可透過 `withRequestMiddleware` 方法註冊 Guzzle 中介層：

```php
use Illuminate\Support\Facades\Http;
use Psr\Http\Message\RequestInterface;

$response = Http::withRequestMiddleware(
    function (RequestInterface $request) {
        return $request->withHeader('X-Example', 'Value');
    }
)->get('http://example.com');
```

同樣地，您可以透過 `withResponseMiddleware` 方法註冊中介層來檢查接收到的 HTTP 回應：

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

有時候，您可能希望註冊一個適用於所有發送請求和接收回應的中介層。為此，您可以使用 `globalRequestMiddleware` 和 `globalResponseMiddleware` 方法。通常，這些方法應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

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

您可以使用 `withOptions` 方法為發送的請求指定額外的 [Guzzle 請求選項](http://docs.guzzlephp.org/en/stable/request-options.html)。`withOptions` 方法接受鍵 / 值對的陣列：

```php
$response = Http::withOptions([
    'debug' => true,
])->get('http://example.com/users');
```

<a name="global-options"></a>
#### 全域選項

若要為每個發送的請求設定預設選項，您可以使用 `globalOptions` 方法。通常，此方法應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法裡呼叫：

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

有時候，您可能希望同時發送多個 HTTP 請求。換句話說，您希望同時發送多個請求，而不是依序發送請求。當與回應較慢的 HTTP API 進行互動時，這可以帶來顯著的效能提升。


<a name="request-pooling"></a>
### 請求池

幸運的是，您可以透過 `pool` 方法來實現此功能。`pool` 方法接受一個閉包，該閉包接收一個 `Illuminate\Http\Client\Pool` 實例，讓您可以輕鬆地將請求新增至請求池中進行分發：

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

如您所見，您可以依據各回應實例被新增至請求池的順序來存取它們。如果您需要，也可以使用 `as` 方法為請求命名，這讓您可以透過名稱存取對應的回應：

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

請求池的最大同時連線數可透過向 `pool` 方法傳入 `concurrency` 引數來控制。這個數值決定了處理請求池時，可同時執行的最大 HTTP 請求數量：

```php
$responses = Http::pool(fn (Pool $pool) => [
    // ...
], concurrency: 5);
```

如果請求池中的請求在連線層級失敗（例如逾時或 DNS 解析失敗），則 `$responses` 陣列中對應的項目將會是 `Illuminate\Http\Client\ConnectionException` 實例，而非 `Response` 實例：

```php
foreach ($responses as $response) {
    if ($response instanceof Throwable) {
        // The request failed to connect...
    } elseif ($response->failed()) {
        // The request connected but received an error response...
    }
}
```


<a name="customizing-concurrent-requests"></a>
#### 客製化同時發送請求

`pool` 方法無法與其他 HTTP 用戶端方法（如 `withHeaders` 或 `middleware` 方法）進行鏈結呼叫。如果您想對池化請求套用自訂標頭或中介層，應該在請求池中的每個請求上獨立設定這些選項：

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
### 請求分批

在 Laravel 中處理同時發送請求的另一種方式是使用 `batch` 方法。與 `pool` 方法類似，它接受一個接收 `Illuminate\Http\Client\Batch` 實例的閉包，讓您可以輕鬆地將請求新增至請求池中進行分發，但它還允許您定義完成後的回調函式：

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

與 `pool` 方法相同，您可以使用 `as` 方法為請求命名：

```php
$responses = Http::batch(fn (Batch $batch) => [
    $batch->as('first')->get('http://localhost/first'),
    $batch->as('second')->get('http://localhost/second'),
    $batch->as('third')->get('http://localhost/third'),
])->send();
```

呼叫 `send` 方法啟動 `batch` 之後，您就無法再新增請求進去。如果嘗試這麼做，將會拋出 `Illuminate\Http\Client\BatchInProgressException` 異常。

請求批次的最大同時連線數可透過 `concurrency` 方法控制。這個數值決定了處理請求批次時，可同時執行的最大 HTTP 請求數量：

```php
$responses = Http::batch(fn (Batch $batch) => [
    // ...
])->concurrency(5)->send();
```


<a name="inspecting-batches"></a>
#### 檢查批次

提供給批次完成回調函式的 `Illuminate\Http\Client\Batch` 實例具備多種屬性與方法，能協助您與指定的請求批次進行互動及檢查：

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
#### 延遲批次

當呼叫 `defer` 方法時，請求批次不會立即執行。相反地，Laravel 會在目前應用程式請求的 HTTP 回應發送給使用者之後才執行該批次，從而保持您的應用程式運作迅速且流暢：

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

Laravel HTTP 用戶端允許您定義「巨集 (Macros)」，在整個應用程式與外部服務互動時，巨集可作為一種流暢且直覺的機制，用來設定常見的請求路徑與標頭。首先，您可以在應用程式的 `App\Providers\AppServiceProvider` 類別之 `boot` 方法中定義巨集：

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

當您的巨集設定完成後，可以在應用程式中的任何地方呼叫它，以建立帶有指定設定的待發送請求：

```php
$response = Http::github()->get('/');
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了能協助您輕鬆且直觀撰寫測試的功能，Laravel 的 HTTP 客戶端也不例外。`Http` Facade 的 `fake` 方法允許您指示 HTTP 客戶端在發送請求時傳回 Stub / 虛擬的回應。


<a name="faking-responses"></a>
### 偽造回應

例如，若要指示 HTTP 客戶端為每個請求都傳回空的 `200` 狀態碼回應，您可以呼叫不帶引數的 `fake` 方法：

```php
use Illuminate\Support\Facades\Http;

Http::fake();

$response = Http::post(/* ... */);
```


<a name="faking-specific-urls"></a>
#### 偽造特定 URL

或者，您可以向 `fake` 方法傳遞一個陣列。該陣列的 Key 應代表您想要偽造的 URL 樣式及其對應的回應。`*` 字元可用作萬用字元。您可以使用 `Http` Facade 的 `response` 方法為這些端點建構 Stub / 偽造回應：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, $headers),

    // Stub a string response for Google endpoints...
    'google.com/*' => Http::response('Hello World', 200, $headers),
]);
```

發送到未被偽造之 URL 的任何請求都將被實際執行。若您想要指定一個保底的 URL 樣式來為所有未符合的 URL 提供 Stub，您可以使用單個 `*` 字元：

```php
Http::fake([
    // Stub a JSON response for GitHub endpoints...
    'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

    // Stub a string response for all other endpoints...
    '*' => Http::response('Hello World', 200, ['Headers']),
]);
```

為方便起見，只要提供字串、陣列或整數作為回應，就能生成簡單的字串、JSON 及空回應：

```php
Http::fake([
    'google.com/*' => 'Hello World',
    'github.com/*' => ['foo' => 'bar'],
    'chatgpt.com/*' => 200,
]);
```


<a name="faking-connection-exceptions"></a>
#### 偽造例外

有時您可能需要測試當 HTTP 客戶端嘗試發送請求時遇到 `Illuminate\Http\Client\ConnectionException` 時應用程式的行為。您可以使用 `failedConnection` 方法指示 HTTP 客戶端拋出連線例外：

```php
Http::fake([
    'github.com/*' => Http::failedConnection(),
]);
```

若要測試當拋出 `Illuminate\Http\Client\RequestException` 時應用程式的行為，您可以使用 `failedRequest` 方法：

```php
$this->mock(GithubService::class);
    ->shouldReceive('getUser')
    ->andThrow(
        Http::failedRequest(['code' => 'not_found'], 404)
    );
```


<a name="faking-response-sequences"></a>
#### 偽造回應序列

有時您可能需要指定單一 URL 應該按特定順序傳回一連串的偽造回應。您可以使用 `Http::sequence` 方法建立回應來達成此目的：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->pushStatus(404),
]);
```

當回應序列中的所有回應都被耗盡時，任何後續的請求都會導致回應序列拋出例外。若您想指定當序列為空時應傳回的預設回應，您可以使用 `whenEmpty` 方法：

```php
Http::fake([
    // Stub a series of responses for GitHub endpoints...
    'github.com/*' => Http::sequence()
        ->push('Hello World', 200)
        ->push(['foo' => 'bar'], 200)
        ->whenEmpty(Http::response()),
]);
```

若您想要偽造一連串的回應，但不需要指定應被偽造的特定 URL 樣式，您可以使用 `Http::fakeSequence` 方法：

```php
Http::fakeSequence()
    ->push('Hello World', 200)
    ->whenEmpty(Http::response());
```


<a name="fake-callback"></a>
#### 偽造回呼

若您需要更複雜的邏輯來決定特定端點要傳回什麼回應，您可以將閉包傳遞給 `fake` 方法。此閉包將接收一個 `Illuminate\Http\Client\Request` 實例，並應傳回一個回應實例。在您的閉包中，您可以執行決定要傳回何種回應類型所需的任何邏輯：

```php
use Illuminate\Http\Client\Request;

Http::fake(function (Request $request) {
    return Http::response('Hello World', 200);
});
```


<a name="inspecting-requests"></a>
### 檢查請求

在偽造回應時，您偶爾可能會想檢查客戶端收到的請求，以確保您的應用程式正在發送正確的資料或標頭。您可以在呼叫 `Http::fake` 之後呼叫 `Http::assertSent` 方法來達成此目的。

`assertSent` 方法接受一個閉包，該閉包將接收一個 `Illuminate\Http\Client\Request` 實例，並應傳回一個布林值，指示該請求是否符合您的預期。為了使測試通過，必須至少發出一個符合給定預期的請求：

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

如有需要，您可以使用 `assertNotSent` 方法斷言未發送特定的請求：

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

您可以使用 `assertSentCount` 方法來斷言測試期間「發送」了多少個請求：

```php
Http::fake();

Http::assertSentCount(5);
```

或者，您可以使用 `assertNothingSent` 方法斷言測試期間沒有發送任何請求：

```php
Http::fake();

Http::assertNothingSent();
```


<a name="recording-requests-and-responses"></a>
#### 記錄請求與回應

您可以使用 `recorded` 方法來收集所有的請求及其對應的回應。`recorded` 方法會傳回一個包含 `Illuminate\Http\Client\Request` 與 `Illuminate\Http\Client\Response` 實例的陣列集合：

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

此外，`recorded` 方法接受一個閉包，該閉包將接收 `Illuminate\Http\Client\Request` 與 `Illuminate\Http\Client\Response` 的實例，可用於根據您的預期篩選請求與回應對：

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
### 防止未預期的請求

如果您希望確保在個別測試或整個測試套件中，透過 HTTP 用戶端發送的所有請求都已被偽造，可以呼叫 `preventStrayRequests` 方法。呼叫此方法後，任何沒有對應偽造回應的請求都將拋出例外，而不是發送實際的 HTTP 請求：

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

有時，您可能希望阻止大多數未預期的請求，但仍允許特定請求執行。為此，您可以將 URL 樣式的陣列傳遞給 `allowStrayRequests` 方法。任何符合其中一個樣式的請求都將被允許，而所有其他請求則會繼續拋出例外：

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

Laravel 在發送 HTTP 請求的過程中會觸發三個事件。`RequestSending` 事件會在請求發送之前觸發，而 `ResponseReceived` 事件則會在收到指定請求的回應後觸發。如果指定請求未收到任何回應，則會觸發 `ConnectionFailed` 事件。

`RequestSending` 與 `ConnectionFailed` 事件都包含一個公開的 `$request` 屬性，您可以用來檢查 `Illuminate\Http\Client\Request` 實例。同樣地，`ResponseReceived` 事件包含 `$request` 屬性以及可以用來檢查 `Illuminate\Http\Client\Response` 實例的 `$response` 屬性。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

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