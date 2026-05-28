# 錯誤處理

- [簡介](#introduction)
- [設定](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外日誌層級](#exception-log-levels)
    - [依類型忽略例外](#ignoring-exceptions-by-type)
    - [渲染例外](#rendering-exceptions)
    - [可回報與可渲染的例外](#renderable-exceptions)
- [限制例外回報頻率](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自訂 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當你開始一個新的 Laravel 專案時，錯誤與例外處理已經為你設定好了；不過，在任何時候，你都可以在應用程式的 `bootstrap/app.php` 中使用 `withExceptions` 方法來管理應用程式如何回報與渲染例外。

提供給 `withExceptions` 閉包的 `$exceptions` 物件是 `Illuminate\Foundation\Configuration\Exceptions` 的一個實例，負責管理應用程式中的例外處理。我們將在整份文件中深入探討這個物件。


<a name="configuration"></a>
## 設定

在 `config/app.php` 設定檔中的 `debug` 選項決定了實際上要向使用者顯示多少關於錯誤的資訊。預設情況下，此選項設定為遵循存儲在 `.env` 檔案中的 `APP_DEBUG` 環境變數值。

在本地開發期間，你應該將 `APP_DEBUG` 環境變數設定為 `true`。

> [!WARNING]
> 在你的正式環境中，`APP_DEBUG` 的值應始終為 `false`。如果在正式環境中將該值設為 `true`，你將冒著向應用程式終端使用者洩露敏感設定值的風險。

<a name="handling-exceptions"></a>
## 處理例外

<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於記錄例外日誌，或將其傳送到外部服務，例如 [Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，例外將根據您的 [日誌](/docs/{{version}}/logging) 設定進行記錄。然而，您可以自由地以任何您想要的方式記錄例外。

如果您需要以不同方式回報不同類型的例外，您可以使用應用程式 `bootstrap/app.php` 中的 `report` 例外方法來註冊一個閉包，該閉包應在需要回報指定類型的例外時執行。Laravel 將透過檢查閉包的型別提示來判斷該閉包回報哪種類型的例外：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```

當您使用 `report` 方法註冊自訂例外回報回呼時，Laravel 仍會使用應用程式的預設日誌設定來記錄該例外。如果您希望停止將例外傳播到預設日誌堆疊，您可以在定義回報回呼時使用 `stop` 方法，或從回呼中回傳 `false`：

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
> 若要為特定例外自訂例外回報，您也可以利用 [可回報的例外](/docs/{{version}}/errors#renderable-exceptions)。

<a name="global-log-context"></a>
#### 全域日誌上下文

如果可用，Laravel 會自動將目前使用者的 ID 作為上下文資料添加到每個例外的日誌訊息中。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `context` 例外方法來定義您自己的全域上下文資料。這些資訊將包含在您的應用程式編寫的每個例外日誌訊息中：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->context(fn () => [
        'foo' => 'bar',
    ]);
})
```

<a name="exception-log-context"></a>
#### 例外日誌上下文

雖然在每條日誌訊息中添加上下文很有用，但有時特定的例外可能具有您想包含在日誌中的獨特上下文。透過在您的應用程式例外之一中定義 `context` 方法，您可以指定與該例外相關的任何資料，並將其添加到該例外的日誌項目中：

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

有時您可能需要回報例外，但繼續處理目前的請求。`report` 輔助函式允許您快速回報例外，而不會向使用者呈現錯誤頁面：

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
#### 排除重複回報的例外

如果您在整個應用程式中都使用 `report` 函式，有時可能會多次回報相同的例外，從而在日誌中建立重複項目。

如果您想確保單個例外實例僅被回報一次，您可以在應用程式的 `bootstrap/app.php` 檔案中調用 `dontReportDuplicates` 例外方法：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReportDuplicates();
})
```

現在，當使用相同的例外實例呼叫 `report` 輔助函式時，只有第一次呼叫會被回報：

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
### 例外日誌層級

當訊息被寫入您的應用程式 [日誌](/docs/{{version}}/logging) 時，訊息會以指定的 [日誌層級](/docs/{{version}}/logging#log-levels) 寫入，這表示所記錄訊息的嚴重性或重要性。

如上所述，即使您使用 `report` 方法註冊了自訂例外回報回呼，Laravel 仍會使用應用程式的預設日誌設定來記錄該例外；然而，由於日誌層級有時會影響記錄訊息的頻道，您可能希望針對特定例外設定其記錄的日誌層級。

為此，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `level` 例外方法。此方法接收例外類型作為其第一個引數，日誌層級作為其第二個引數：

```php
use PDOException;
use Psr\Log\LogLevel;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);
})
```

<a name="ignoring-exceptions-by-type"></a>
### 依類型忽略例外

在開發應用程式時，某些類型的例外您可能永遠不想回報。若要忽略這些例外，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `dontReport` 例外方法。提供給此方法的任何類別都不會被回報；但是，它們仍可能有自訂的渲染邏輯：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);
})
```

或者，您可以簡單地用 `Illuminate\Contracts\Debug\ShouldntReport` 介面「標記」一個例外類別。當例外被標記為此介面時，它將永遠不會被 Laravel 的例外處理程序回報：

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

如果您需要更精確地控制何時忽略特定類型的例外，您可以向 `dontReportWhen` 方法提供一個閉包：

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

在內部，Laravel 已經為您忽略了某些類型的錯誤，例如 404 HTTP 錯誤產生的例外、來源不匹配產生的 403 HTTP 回應，或由無效 CSRF Token 產生的 419 HTTP 回應。如果您想指示 Laravel 停止忽略指定的例外類型，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `stopIgnoring` 例外方法：

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->stopIgnoring(HttpException::class);
})
```

<a name="rendering-exceptions"></a>
### 渲染例外

預設情況下，Laravel 的例外處理器會為你將例外轉換為 HTTP 回應。然而，你可以自由地為特定類型的例外註冊自訂的渲染閉包。你可以在應用程式的 `bootstrap/app.php` 檔案中使用 `render` 例外方法來達成此目的。

傳遞給 `render` 方法的閉包應該回傳一個 `Illuminate\Http\Response` 實例，這可以透過 `response` 輔助函式產生。Laravel 會透過檢查閉包的型別提示來判斷該閉包要渲染哪種類型的例外：

```php
use App\Exceptions\InvalidOrderException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });
})
```

你也可以使用 `render` 方法來覆寫 Laravel 內建或 Symfony 例外（例如 `NotFoundHttpException`）的渲染行為。如果傳遞給 `render` 方法的閉包沒有回傳值，則會使用 Laravel 預設的例外渲染機制：

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

在渲染例外時，Laravel 會根據請求的 `Accept` 標頭自動判斷例外應該被渲染為 HTML 還是 JSON 回應。如果你想自訂 Laravel 如何判斷是否要渲染 JSON 例外回應，可以使用 `shouldRenderJsonWhen` 方法：

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

極少數情況下，你可能需要自訂由 Laravel 例外處理器渲染的整個 HTTP 回應。為此，你可以使用 `respond` 方法註冊一個回應自訂閉包：

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

除了在應用程式的 `bootstrap/app.php` 檔案中定義自訂的回報與渲染行為外，你也可以直接在應用程式的例外類別中定義 `report` 與 `render` 方法。當這些方法存在時，框架會自動呼叫它們：

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

如果你的例外繼承了一個已經具備可渲染能力的例外，例如 Laravel 內建或 Symfony 的例外，你可以在該例外的 `render` 方法中回傳 `false`，以便渲染該例外預設的 HTTP 回應：

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

如果你的例外包含僅在滿足某些條件時才需要的自訂回報邏輯，你可能需要指示 Laravel 有時使用預設的例外處理設定來回報該例外。為此，你可以在該例外的 `report` 方法中回傳 `false`：

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
> 你可以在 `report` 方法中對任何需要的依賴項進行型別提示，它們將由 Laravel 的 [服務容器(service container)](/docs/{{version}}/container) 自動注入。


<a name="throttling-reported-exceptions"></a>
### 限制例外回報頻率

如果你的應用程式回報了大量的例外，你可能會想要限制實際記錄或發送到應用程式外部錯誤追蹤服務的例外數量。

若要對例外進行隨機比例的取樣，你可以在應用程式的 `bootstrap/app.php` 檔案中使用 `throttle` 例外方法。`throttle` 方法接收一個應回傳 `Lottery` 實例的閉包：

```php
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        return Lottery::odds(1, 1000);
    });
})
```

也可以根據例外類型進行條件取樣。如果你只想對特定例外類別的實例進行取樣，可以僅針對該類別回傳 `Lottery` 實例：

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

你也可以透過回傳 `Limit` 實例而非 `Lottery` 來對記錄或發送到外部錯誤追蹤服務的例外進行速率限制。如果你想防止突然爆發的例外淹沒日誌，例如當應用程式使用的第三方服務暫時失效時，這會非常有用：

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

預設情況下，限制將使用例外的類別名稱作為速率限制的金鑰。你可以透過在 `Limit` 上使用 `by` 方法來指定自己的金鑰：

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

當然，你也可以針對不同的例外回傳 `Lottery` 與 `Limit` 實例的混合：

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

某些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「找不到頁面」錯誤 (404)、「未經授權錯誤」 (401)，甚至是開發者產生的 500 錯誤。為了從應用程式中的任何位置產生此類回應，你可以使用 `abort` 輔助函式：

```php
abort(404);
```

<a name="custom-http-error-pages"></a>
### 自訂 HTTP 錯誤頁面

Laravel 讓你可以輕鬆地為各種 HTTP 狀態碼顯示自訂錯誤頁面。例如，要為 404 HTTP 狀態碼自訂錯誤頁面，請建立一個 `resources/views/errors/404.blade.php` 視圖模板。此視圖將為你的應用程式產生的所有 404 錯誤進行渲染。該目錄中的視圖命名應與其對應的 HTTP 狀態碼一致。由 `abort` 函式引發的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將作為 `$exception` 變數傳遞給視圖：

```blade
<h2>{{ $exception->getMessage() }}</h2>
```

你可以使用 `vendor:publish` Artisan 指令發布 Laravel 預設的錯誤頁面模板。模板發布後，你就可以根據自己的喜好進行自訂：

```shell
php artisan vendor:publish --tag=laravel-errors
```

<a name="fallback-http-error-pages"></a>
#### 備用 HTTP 錯誤頁面

你也可以為一系列的 HTTP 狀態碼定義「備用 (Fallback)」錯誤頁面。如果沒有與發生的特定 HTTP 狀態碼相對應的頁面，則會渲染此頁面。若要實現此功能，請在應用程式的 `resources/views/errors` 目錄中定義 `4xx.blade.php` 模板和 `5xx.blade.php` 模板。

在定義備用錯誤頁面時，備用頁面不會影響 `404`、`500` 和 `503` 錯誤回應，因為 Laravel 對於這些狀態碼有內建的專屬頁面。要自訂這些狀態碼所渲染的頁面，你應該分別為它們定義個別的自訂錯誤頁面。