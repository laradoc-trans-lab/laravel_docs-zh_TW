# 錯誤處理

- [簡介](#introduction)
- [配置](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外紀錄層級](#exception-log-levels)
    - [按類型忽略例外](#ignoring-exceptions-by-type)
    - [渲染例外](#rendering-exceptions)
    - [可回報與可渲染的例外](#renderable-exceptions)
- [限制回報例外的頻率](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自定義 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當您啟動一個新的 Laravel 專案時，錯誤與例外處理已經為您配置好了；然而，您可以在任何時間點，使用應用程式中 `bootstrap/app.php` 的 `withExceptions` 方法來管理例外如何被您的應用程式回報與渲染。

提供給 `withExceptions` 閉包的 `$exceptions` 物件是一個 `Illuminate\Foundation\Configuration\Exceptions` 的實例，負責管理您應用程式中的例外處理。我們將在本文件中深入探討這個物件。


<a name="configuration"></a>
## 配置

`config/app.php` 設定檔中的 `debug` 選項決定了實際上會向使用者顯示多少關於錯誤的資訊。預設情況下，此選項設定為遵循儲存在 `.env` 檔案中的 `APP_DEBUG` 環境變數值。

在本地開發期間，您應該將 `APP_DEBUG` 環境變數設定為 `true`。

> [!WARNING]
> 在您的正式環境中，`APP_DEBUG` 的值應該始終為 `false`。如果在正式環境中將其設定為 `true`，您將面臨將敏感的設定值洩漏給應用程式最終使用者的風險。**

<a name="handling-exceptions"></a>
## 處理例外


<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於將例外紀錄到日誌或發送到外部服務，例如 [Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，例外將根據您的 [紀錄(logging)](/docs/{{version}}/logging) 配置進行紀錄。然而，您可以隨意決定如何紀錄例外。

如果您需要以不同方式回報不同類型的例外，您可以使用應用程式 `bootstrap/app.php` 中的 `report` 例外方法來註冊一個在需要回報特定類型的例外時應執行的閉包。Laravel 會透過檢查閉包的型別提示 (type-hint) 來決定該閉包回報的是哪種類型的例外：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```

當您使用 `report` 方法註冊自定義的例外回報回呼函式 (callback) 時，Laravel 仍然會使用應用程式的預設紀錄配置來紀錄該例外。如果您希望停止將例外傳播到預設的紀錄堆疊，您可以在定義回報回呼函式時使用 `stop` 方法，或從回呼函式中回傳 `false`：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    })->stop();

    $exceptions->report(function (InvalidOrderException $e) {
        return false;
    });
})
```

> [!NOTE]
> 若要自定義特定例外的回報方式，您也可以利用 [可回報例外(reportable exceptions)](/docs/{{version}}/errors#renderable-exceptions)。


<a name="global-log-context"></a>
#### 全域紀錄上下文

如果可用，Laravel 會自動將目前使用者的 ID 作為上下文數據添加到每個例外的紀錄訊息中。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `context` 例外方法來定義自己的全域上下文數據。此資訊將包含在應用程式寫入的每個例外紀錄訊息中：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->context(fn () => [
        'foo' => 'bar',
    ]);
})
```


<a name="exception-log-context"></a>
#### 例外紀錄上下文

雖然為每條紀錄訊息添加上下文很有用，但有時特定的例外可能有您想要包含在紀錄中的唯一上下文。透過在應用程式的其中一個例外類別中定義 `context` 方法，您可以指定與該例外相關的任何數據，並將其添加到例外的紀錄條目中：

```php
<?php

namespace App\Exceptions;

use Exception;

class InvalidOrderException extends Exception
{
    // ...

    /**
     * Get the exception's context information.
     *
     * @return array<string, mixed>
     */
    public function context(): array
    {
        return ['order_id' => $this->orderId];
    }
}
```


<a name="the-report-helper"></a>
#### `report` 輔助函式

有時您可能需要回報一個例外，但仍要繼續處理目前的請求。`report` 輔助函式允許您快速回報例外，而不會向使用者渲染錯誤頁面：

```php
public function isValid(string $value): bool
{
    try {
        // Validate the value...
    } catch (Throwable $e) {
        report($e);

        return false;
    }
}
```


<a name="deduplicating-reported-exceptions"></a>
#### 去除重複回報的例外

如果您在整個應用程式中使用 `report` 函式，您可能會偶爾多次回報同一個例外，從而在紀錄中產生重複的條目。

如果您想確保同一個例外的實例僅被回報一次，您可以在應用程式的 `bootstrap/app.php` 檔案中調用 `dontReportDuplicates` 例外方法：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReportDuplicates();
})
```

現在，當 `report` 輔助函式被傳入同一個例外的實例時，只有第一次調用會被回報：

```php
$original = new RuntimeException('Whoops!');

report($original); // reported

try {
    throw $original;
} catch (Throwable $caught) {
    report($caught); // ignored
}

report($original); // ignored
report($caught); // ignored
```


<a name="exception-log-levels"></a>
### 例外紀錄層級

當訊息寫入應用程式的 [紀錄(logs)](/docs/{{version}}/logging) 時，訊息會以指定的 [紀錄層級(log level)](/docs/{{version}}/logging#log-levels) 寫入，這表示被紀錄訊息的嚴重程度或重要性。

如上所述，即使您使用 `report` 方法註冊自定義的例外回報回呼函式，Laravel 仍然會使用應用程式的預設紀錄配置來紀錄該例外；然而，由於紀錄層級有時會影響訊息被紀錄的頻道，您可能希望配置某些例外被紀錄時的紀錄層級。

要實現此功能，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `level` 例外方法。此方法接收例外類型作為第一個引數，紀錄層級作為第二個引數：

```php
use PDOException;
use Psr\Log\LogLevel;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);
})
```


<a name="ignoring-exceptions-by-type"></a>
### 按類型忽略例外

在構建應用程式時，會有一些您永遠不想回報的例外類型。要忽略這些例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `dontReport` 例外方法。提供給此方法的任何類別都將永遠不會被回報；但它們仍然可以具有自定義的渲染邏輯：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);
})
```

或者，您可以簡單地使用 `Illuminate\Contracts\Debug\ShouldntReport` 介面來「標記」一個例外類別。當一個例外被此介面標記時，它將永遠不會被 Laravel 的例外處理器回報：

```php
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Contracts\Debug\ShouldntReport;

class PodcastProcessingException extends Exception implements ShouldntReport
{
    //
}
```

如果您需要對特定類型的例外在何時被忽略有更精細的控制，您可以為 `dontReportWhen` 方法提供一個閉包：

```php
use App\Exceptions\InvalidOrderException;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReportWhen(function (Throwable $e) {
        return $e instanceof PodcastProcessingException &&
               $e->reason() === 'Subscription expired';
    });
})
```

在內部，Laravel 已經為您忽略了某些類型的錯誤，例如 404 HTTP 錯誤導致的例外、由來源不匹配產生的 403 HTTP 回應，或由無效的 CSRF 令牌產生的 419 HTTP 回應。如果您想指示 Laravel 停止忽略特定類型的例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `stopIgnoring` 例外方法：

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->stopIgnoring(HttpException::class);
})
```

<a name="rendering-exceptions"></a>
### 渲染例外

預設情況下，Laravel 的例外處理器會為您將例外轉換為 HTTP 回應。不過，您可以針對特定類型的例外註冊自定義的渲染閉包。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `render` 例外方法來達成此目的。

傳遞給 `render` 方法的閉包應該回傳一個 `Illuminate\Http\Response` 實例，這可以透過 `response` 輔助函式來產生。Laravel 會透過檢查閉包的型別提示 (type-hint) 來決定該閉包負責渲染哪種類型的例外：

```php
use App\Exceptions\InvalidOrderException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });
})
```

您也可以使用 `render` 方法來覆寫 Laravel 或 Symfony 內建例外的渲染行為，例如 `NotFoundHttpException`。如果傳遞給 `render` 方法的閉包沒有回傳值，則會使用 Laravel 預設的例外渲染機制：

```php
use Illuminate\Http\Request;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (NotFoundHttpException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => 'Record not found.'
            ], 404);
        }
    });
})
```


<a name="rendering-exceptions-as-json"></a>
#### 將例外渲染為 JSON

在渲染例外時，Laravel 會根據請求的 `Accept` 標頭自動決定該例外應渲染為 HTML 還是 JSON 回應。如果您想要自定義 Laravel 如何決定要渲染 HTML 還是 JSON 例外回應，可以使用 `shouldRenderJsonWhen` 方法：

```php
use Illuminate\Http\Request;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
        if ($request->is('admin/*')) {
            return true;
        }

        return $request->expectsJson();
    });
})
```


<a name="customizing-the-exception-response"></a>
#### 自定義例外回應

在極少數情況下，您可能需要自定義由 Laravel 例外處理器渲染的整個 HTTP 回應。為了達成這個目的，您可以使用 `respond` 方法註冊一個回應自定義閉包：

```php
use Symfony\Component\HttpFoundation\Response;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->respond(function (Response $response) {
        if ($response->getStatusCode() === 419) {
            return back()->with([
                'message' => 'The page expired, please try again.',
            ]);
        }

        return $response;
    });
})
```


<a name="renderable-exceptions"></a>
### 可回報與可渲染的例外

您可以直接在應用程式的例外類別中定義 `report` 與 `render` 方法，而不需要在 `bootstrap/app.php` 檔案中定義自定義的回報與渲染行為。當這些方法存在時，框架會自動呼叫它們：

```php
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class InvalidOrderException extends Exception
{
    /**
     * Report the exception.
     */
    public function report(): void
    {
        // ...
    }

    /**
     * Render the exception as an HTTP response.
     */
    public function render(Request $request): Response
    {
        return response(/* ... */);
    }
}
```

如果您的例外繼承自一個已經可渲染的例外（例如 Laravel 或 Symfony 的內建例外），您可以從例外的 `render` 方法回傳 `false`，以渲染該例外的預設 HTTP 回應：

```php
/**
 * Render the exception as an HTTP response.
 */
public function render(Request $request): Response|bool
{
    if (/** Determine if the exception needs custom rendering */) {

        return response(/* ... */);
    }

    return false;
}
```

如果您的例外包含僅在滿足特定條件時才需要的自定義回報邏輯，您可能需要指示 Laravel 在某些情況下使用預設的例外處理配置來回報該例外。為了達成此目的，您可以從例外的 `report` 方法回傳 `false`：

```php
/**
 * Report the exception.
 */
public function report(): bool
{
    if (/** Determine if the exception needs custom reporting */) {

        // ...

        return true;
    }

    return false;
}
```

> [!NOTE]
> 您可以在 `report` 方法中對任何需要的依賴項使用型別提示 (type-hint)，Laravel 的 [服務容器 (service container)](/docs/{{version}}/container) 會自動將它們注入到該方法中。


<a name="throttling-reported-exceptions"></a>
### 限制回報例外的頻率

如果您的應用程式回報了大量例外，您可能希望限制實際紀錄或發送到外部錯誤追蹤服務的例外數量。

若要對例外進行隨機抽樣，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `throttle` 例外方法。`throttle` 方法接收一個應該回傳 `Lottery` 實例的閉包：

```php
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        return Lottery::odds(1, 1000);
    });
})
```

您也可以根據例外類型進行條件抽樣。如果您只想對特定例外類別的實例進行抽樣，可以僅針對該類別回傳 `Lottery` 實例：

```php
use App\Exceptions\ApiMonitoringException;
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        if ($e instanceof ApiMonitoringException) {
            return Lottery::odds(1, 1000);
        }
    });
})
```

您也可以透過回傳 `Limit` 實例而非 `Lottery` 實例，來限制紀錄或發送到外部錯誤追蹤服務的例外速率。當您想要防止突發的大量例外淹沒您的日誌時（例如，當應用程式使用的第三方服務宕機時），這會非常有用：

```php
use Illuminate\Broadcasting\BroadcastException;
use Illuminate\Cache\RateLimiting\Limit;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        if ($e instanceof BroadcastException) {
            return Limit::perMinute(300);
        }
    });
})
```

預設情況下，限制將使用例外的類別作為速率限制金鑰 (rate limit key)。您可以使用 `Limit` 上的 `by` 方法來指定自己的金鑰以進行自定義：

```php
use Illuminate\Broadcasting\BroadcastException;
use Illuminate\Cache\RateLimiting\Limit;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        if ($e instanceof BroadcastException) {
            return Limit::perMinute(300)->by($e->getMessage());
        }
    });
})
```

當然，您也可以針對不同的例外回傳 `Lottery` 與 `Limit` 實例的混合組合：

```php
use App\Exceptions\ApiMonitoringException;
use Illuminate\Broadcasting\BroadcastException;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        return match (true) {
            $e instanceof BroadcastException => Limit::perMinute(300),
            $e instanceof ApiMonitoringException => Lottery::odds(1, 1000),
            default => Limit::none(),
        };
    });
})
```

<a name="http-exceptions"></a>
## HTTP 例外

某些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「找不到頁面」錯誤 (404)、「未經授權錯誤」(401)，甚至是開發者產生的 500 錯誤。為了在應用程式的任何地方產生此類回應，您可以使用 `abort` 輔助函式：

```php
abort(404);
```


<a name="custom-http-error-pages"></a>
### 自定義 HTTP 錯誤頁面

Laravel 讓您能夠輕鬆地為各種 HTTP 狀態碼顯示自定義錯誤頁面。例如，要自定義 404 HTTP 狀態碼的錯誤頁面，請建立 `resources/views/errors/404.blade.php` 視圖模板。此視圖將用於渲染應用程式產生的所有 404 錯誤。此目錄中的視圖名稱應與其對應的 HTTP 狀態碼一致。由 `abort` 函式拋出的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將作為 `$exception` 變數傳遞至視圖中：

```blade
<h2>{{ $exception->getMessage() }}</h2>
```

您可以使用 `vendor:publish` Artisan 命令來發布 Laravel 預設的錯誤頁面模板。模板發布後，您可以根據喜好對其進行自定義：

```shell
php artisan vendor:publish --tag=laravel-errors
```


<a name="fallback-http-error-pages"></a>
#### 備用 HTTP 錯誤頁面

您還可以為一系列特定的 HTTP 狀態碼定義「備用 (fallback)」錯誤頁面。如果發生特定 HTTP 狀態碼且沒有對應的頁面時，將渲染此頁面。若要實現此功能，請在應用程式的 `resources/views/errors` 目錄中定義 `4xx.blade.php` 模板和 `5xx.blade.php` 模板。

在定義備用錯誤頁面時，備用頁面不會影響 `404`、`500` 和 `503` 錯誤回應，因為 Laravel 為這些狀態碼提供了內部專用頁面。若要自定義這些狀態碼所渲染的頁面，您應該為其中的每一個分別定義自定義錯誤頁面。