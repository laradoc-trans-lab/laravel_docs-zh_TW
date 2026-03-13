# 錯誤處理

- [簡介](#introduction)
- [設定](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外記錄層級](#exception-log-levels)
    - [按類型忽略例外](#ignoring-exceptions-by-type)
    - [渲染例外](#rendering-exceptions)
    - [可回報與可渲染的例外](#renderable-exceptions)
- [限制回報例外的頻率](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自訂 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當你開始一個新的 Laravel 專案時，錯誤與例外處理已經為你設定好了；不過，在任何時候，你都可以使用應用程式 `bootstrap/app.php` 中的 `withExceptions` 方法，來管理應用程式如何回報與渲染例外。

傳遞給 `withExceptions` 閉包的 `$exceptions` 物件是 `Illuminate\Foundation\Configuration\Exceptions` 的實例，負責管理應用程式中的例外處理。我們將在整份文件中深入探討此物件。


<a name="configuration"></a>
## 設定

`config/app.php` 設定檔中的 `debug` 選項決定了實際上要向使用者顯示多少關於錯誤的資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在你的 `.env` 檔案中。

在本地開發期間，你應該將 `APP_DEBUG` 環境變數設定為 `true`。

> [!WARNING]
> 在你的正式環境中，`APP_DEBUG` 的值應該始終為 `false`。如果在正式環境中將該值設定為 `true`，你將面臨向應用程式終端使用者洩漏敏感設定值的風險。**

<a name="handling-exceptions"></a>
## 處理例外


<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於記錄例外或將其發送到外部服務，如 [Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，例外會根據您的 [記錄](/docs/{{version}}/logging) 設定進行記錄。不過，您可以隨意記錄例外。

如果您需要以不同方式回報不同類型的例外，可以在應用程式的 `bootstrap/app.php` 中使用 `report` 例外方法來註冊一個當需要回報特定類型的例外時應執行的閉包。Laravel 將透過檢查閉包的型別提示來確定該閉包回報的例外類型：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```

當您使用 `report` 方法註冊自訂例外回報回呼時，Laravel 仍會使用應用程式的預設記錄設定來記錄該例外。如果您希望停止將例外傳遞到預設記錄堆疊，可以在定義回報回呼時使用 `stop` 方法，或從回呼中回傳 `false`：

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
> 若要自訂特定例外的例外回報，您也可以利用 [可回報的例外](/docs/{{version}}/errors#renderable-exceptions)。


<a name="global-log-context"></a>
#### 全域記錄上下文

如果可用，Laravel 會自動將目前使用者的 ID 作為上下文資料加入到每個例外的記錄訊息中。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `context` 例外方法來定義自己的全域上下文資料。此資訊將包含在應用程式寫入的每個例外記錄訊息中：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->context(fn () => [
        'foo' => 'bar',
    ]);
})
```


<a name="exception-log-context"></a>
#### 例外記錄上下文

雖然在每條記錄訊息中加入內容很有用，但有時特定的例外可能具有您想要包含在記錄中的獨特上下文。透過在應用程式的其中一個例外中定義 `context` 方法，您可以指定應加入到該例外記錄項目中的任何與該例外相關的資料：

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

有時您可能需要回報例外，但繼續處理目前的請求。`report` 輔助函式讓您可以快速回報例外，而無需向使用者渲染錯誤頁面：

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
#### 去除重複的回報例外

如果您在整個應用程式中使用 `report` 函式，有時可能會多次回報同一個例外，從而在記錄中產生重複的項目。

如果您想確保一個例外的單個執行個體只被回報一次，可以在應用程式的 `bootstrap/app.php` 檔案中呼叫 `dontReportDuplicates` 例外方法：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReportDuplicates();
})
```

現在，當使用相同的例外執行個體呼叫 `report` 輔助函式時，只有第一次呼叫會被回報：

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
### 例外記錄層級

當訊息被寫入應用程式的 [記錄](/docs/{{version}}/logging) 時，訊息會以指定的 [記錄層級](/docs/{{version}}/logging#log-levels) 寫入，這表示正在記錄的訊息的嚴重程度或重要性。

如上所述，即使您使用 `report` 方法註冊了自訂例外回報回呼，Laravel 仍會使用應用程式的預設記錄設定來記錄該例外；然而，由於記錄層級有時會影響訊息記錄到的頻道，您可能希望設定某些例外記錄時的層級。

若要實現此目的，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `level` 例外方法。此方法接收例外類型作為其第一個參數，並接收記錄層級作為其第二個參數：

```php
use PDOException;
use Psr\Log\LogLevel;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);
})
```


<a name="ignoring-exceptions-by-type"></a>
### 按類型忽略例外

在建構應用程式時，有些類型的例外您永遠不想回報。若要忽略這些例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `dontReport` 例外方法。提供給此方法的任何類別都永遠不會被回報；但是，它們可能仍然具有自訂的渲染邏輯：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);
})
```

或者，您可以簡單地用 `Illuminate\Contracts\Debug\ShouldntReport` 介面「標記」一個例外類別。當一個例外被標記為此介面時，它將永遠不會被 Laravel 的例外處理程序回報：

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

如果您需要更進一步控制何時忽略特定類型的例外，可以為 `dontReportWhen` 方法提供一個閉包：

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

在內部，Laravel 已經為您忽略了一些類型的錯誤，例如由 404 HTTP 錯誤產生的例外，或由無效 CSRF 權杖產生的 419 HTTP 回應。如果您想指示 Laravel 停止忽略特定類型的例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `stopIgnoring` 例外方法：

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->stopIgnoring(HttpException::class);
})
```

<a name="rendering-exceptions"></a>
### 渲染例外

預設情況下，Laravel 例外處理器會為您將例外轉換為 HTTP 回應。然而，您可以自由地為特定類型的例外註冊自訂渲染閉包。您可以透過在應用程式的 `bootstrap/app.php` 檔案中使用 `render` 例外方法來完成此操作。

傳遞給 `render` 方法的閉包應該回傳一個 `Illuminate\Http\Response` 執行個體，這可以透過 `response` 輔助函式產生。Laravel 將透過檢查閉包的型別提示 (Type-hint) 來決定閉包要渲染哪種類型的例外：

```php
use App\Exceptions\InvalidOrderException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });
})
```

您也可以使用 `render` 方法來覆寫 Laravel 或 Symfony 內建例外的渲染行為，例如 `NotFoundHttpException`。如果傳遞給 `render` 方法的閉包沒有回傳值，則會使用 Laravel 預設的例外渲染：

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
#### 以 JSON 渲染例外

在渲染例外時，Laravel 會根據請求的 `Accept` 標頭自動決定例外應渲染為 HTML 還是 JSON 回應。如果您想自訂 Laravel 如何決定渲染 HTML 還是 JSON 例外回應，可以使用 `shouldRenderJsonWhen` 方法：

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
#### 自訂例外回應

在極少數情況下，您可能需要自訂由 Laravel 例外處理器渲染的整個 HTTP 回應。若要實現此目的，您可以使用 `respond` 方法註冊一個回應自訂閉包：

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

除了在應用程式的 `bootstrap/app.php` 檔案中定義自訂回報與渲染行為，您也可以直接在應用程式的例外類別中定義 `report` 和 `render` 方法。當這些方法存在時，框架會自動呼叫它們：

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

如果您的例外繼承了一個已經可以渲染的例外（例如 Laravel 或 Symfony 內建的例外），您可以從該例外的 `render` 方法回傳 `false`，以渲染該例外的預設 HTTP 回應：

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

如果您的例外包含僅在滿足特定條件時才需要的自訂回報邏輯，您可能需要指示 Laravel 有時使用預設例外處理設定來回報例外。若要實現此目的，您可以從該例外的 `report` 方法回傳 `false`：

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
> 您可以對 `report` 方法所需的任何依賴項進行型別提示，它們將由 Laravel 的 [服務容器](/docs/{{version}}/container) 自動注入到方法中。


<a name="throttling-reported-exceptions"></a>
### 限制回報例外的頻率

如果您的應用程式回報了大量的例外，您可能希望限制實際記錄或發送到應用程式外部錯誤追蹤服務的例外數量。

若要對例外進行隨機取樣，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `throttle` 例外方法。`throttle` 方法接收一個應回傳 `Lottery` 執行個體的閉包：

```php
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        return Lottery::odds(1, 1000);
    });
})
```

也可以根據例外類型進行有條件的取樣。如果您只想對特定例外類別的執行個體進行取樣，可以僅針對該類別回傳 `Lottery` 執行個體：

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

您也可以透過回傳 `Limit` 執行個體而非 `Lottery` 來對記錄或發送到外部錯誤追蹤服務的例外進行速率限制 (Rate limit)。如果您想防止突然爆發的例外淹沒您的日誌，這很有用，例如當您的應用程式使用的第三方服務發生故障時：

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

預設情況下，限制會使用例外的類別作為速率限制金鑰。您可以透過在 `Limit` 上使用 `by` 方法指定您自己的金鑰來進行自訂：

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

當然，您可以針對不同的例外回傳 `Lottery` 和 `Limit` 執行個體的組合：

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

某些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「頁面未找到」錯誤 (404)、「未授權錯誤」(401)，甚至是開發者產生的 500 錯誤。若要在應用程式中的任何位置產生此類回應，您可以使用 `abort` 輔助函式：

```php
abort(404);
```


<a name="custom-http-error-pages"></a>
### 自訂 HTTP 錯誤頁面

Laravel 讓為各種 HTTP 狀態碼顯示自訂錯誤頁面變得非常容易。例如，要為 404 HTTP 狀態碼自訂錯誤頁面，請建立一個 `resources/views/errors/404.blade.php` 視圖模板。此視圖將為您的應用程式產生的所有 404 錯誤進行渲染。此目錄中的視圖命名應與其對應的 HTTP 狀態碼相符。由 `abort` 函式引發的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將作為 `$exception` 變數被傳遞至視圖中：

```blade
<h2>{{ $exception->getMessage() }}</h2>
```

您可以使用 `vendor:publish` 這個 Artisan 指令來發布 Laravel 預設的錯誤頁面模板。一旦模板發布後，您就可以根據自己的喜好進行自訂：

```shell
php artisan vendor:publish --tag=laravel-errors
```


<a name="fallback-http-error-pages"></a>
#### 回退 HTTP 錯誤頁面

您也可以為特定系列的 HTTP 狀態碼定義一個「回退 (fallback)」錯誤頁面。如果發生的特定 HTTP 狀態碼沒有對應的頁面，則會渲染此頁面。若要實現此功能，請在應用程式的 `resources/views/errors` 目錄中定義一個 `4xx.blade.php` 模板和一個 `5xx.blade.php` 模板。

在定義回退錯誤頁面時，回退頁面不會影響 `404`、`500` 和 `503` 錯誤回應，因為 Laravel 對於這些狀態碼已有內部的專用頁面。若要自訂這些狀態碼所渲染的頁面，您應該為其中的每一個分別定義一個自訂錯誤頁面。