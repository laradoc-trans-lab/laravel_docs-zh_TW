# 錯誤處理

- [簡介](#introduction)
- [設定](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外日誌層級](#exception-log-levels)
    - [依型別忽略例外](#ignoring-exceptions-by-type)
    - [渲染例外](#rendering-exceptions)
    - [可回報與可渲染的例外](#renderable-exceptions)
- [限制例外回報頻率](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自訂 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當您建立新的 Laravel 專案時，錯誤與例外處理已經預先為您配置完成；然而，在任何時候，您都可以使用應用程式 `bootstrap/app.php` 中的 `withExceptions` 方法，來管理應用程式如何回報與渲染例外。

傳遞給 `withExceptions` 閉包的 `$exceptions` 物件是 `Illuminate\Foundation\Configuration\Exceptions` 的實例，負責管理應用程式中的例外處理。我們將在整份文件中更深入地探討此物件。


<a name="configuration"></a>
## 設定

您的 `config/app.php` 設定檔中的 `debug` 選項決定了實際上向使用者顯示多少關於錯誤的資訊。預設情況下，此選項設定為遵循存放在 `.env` 檔案中的 `APP_DEBUG` 環境變數值。

在本地端開發期間，您應該將 `APP_DEBUG` 環境變數設定為 `true`。

> [!WARNING]
> 在正式環境中，`APP_DEBUG` 的值應始終為 `false`。若在正式環境中將該值設為 `true`，您可能會面臨將機密設定值暴露給應用程式終端使用者的風險。

<a name="handling-exceptions"></a>
## 處理例外

<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於記錄例外日誌或將其發送到外部服務，例如 [Laravel Nightwatch](https://nightwatch.laravel.com)、[Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，系統會根據您的[記錄](/docs/{{version}}/logging)設定來記錄例外日誌。不過，您可以完全自由地以任何想要的方式記錄例外。

如果您需要以不同方式回報不同型別的例外，可以使用應用程式 `bootstrap/app.php` 中的 `report` 例外方法來註冊一個閉包，該閉包會在需要回報特定型別的例外時執行。Laravel 會透過檢查閉包的型別提示 (Type-hint) 來判斷該閉包負責回報哪種型別的例外：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```

當您使用 `report` 方法註冊自訂的例外回報回呼 (Callback) 時，Laravel 仍會使用應用程式的預設日誌設定來記錄該例外。如果您希望停止將該例外傳播到預設日誌堆疊，可以在定義回報回呼時使用 `stop` 方法，或從回呼中回傳 `false`：

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
> 若要為特定例外自訂例外回報，您也可以使用[可回報的例外](/docs/{{version}}/errors#renderable-exceptions)。

<a name="global-log-context"></a>
#### 全域日誌上下文

在可用的情況下，Laravel 會自動將當前使用者的 ID 作為上下文資料加入到每個例外的日誌訊息中。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `context` 例外方法來定義自己的全域上下文資料。這些資訊將會包含在應用程式寫入的每條例外日誌訊息中：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->context(fn () => [
        'foo' => 'bar',
    ]);
})
```

<a name="exception-log-context"></a>
#### 例外日誌上下文

雖然在每個日誌訊息中加入上下文很有用，但有時特定例外可能包含您希望加入日誌中的獨特上下文。透過在應用程式的某個例外中定義 `context` 方法，您可以指定該例外相關的任何資料，這些資料將會加入到該例外的日誌記錄項目中：

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

有時您可能需要回報例外，但仍繼續處理當前的請求。`report` 輔助函式可讓您快速回報例外，而無需向使用者渲染錯誤頁面：

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

如果您在整個應用程式中使用 `report` 函式，可能會偶爾多次回報同一個例外，從而在日誌中產生重複的記錄。

如果您希望確保單一例外實例僅被回報一次，可以在應用程式的 `bootstrap/app.php` 檔案中呼叫 `dontReportDuplicates` 例外方法：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReportDuplicates();
})
```

現在，當使用相同的例外實例呼叫 `report` 輔助函式時，只會回報第一次呼叫：

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

當訊息被寫入應用程式的[日誌](/docs/{{version}}/logging)時，訊息會以指定的[日誌層級](/docs/{{version}}/logging#log-levels)寫入，該層級表示所記錄訊息的嚴重性或重要性。

如上所述，即使您使用 `report` 方法註冊了自訂的例外回報回呼，Laravel 仍會使用應用程式的預設日誌設定來記錄例外；然而，由於日誌層級有時會影響訊息寫入的頻道，您可能希望設定特定例外記錄時所使用的日誌層級。

為了達成這一點，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `level` 例外方法。該方法接收例外型別作為其第一個引數，並接收日誌層級作為其第二個引數：

```php
use PDOException;
use Psr\Log\LogLevel;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);
})
```

<a name="ignoring-exceptions-by-type"></a>
### 依型別忽略例外

在建構應用程式時，某些型別的例外您可能永遠不想回報。若要忽略這些例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `dontReport` 例外方法。傳遞給此方法的任何類別都永遠不會被回報；然而，它們仍然可以擁有自訂的渲染邏輯：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);
})
```

或者，您可以簡單地使用 `Illuminate\Contracts\Debug\ShouldntReport` 介面來「標記」一個例外類別。當例外被標記此介面時，Laravel 的例外處理常式將永遠不會回報它：

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

如果您需要對何時忽略特定型別的例外擁有更多控制權，可以向 `dontReportWhen` 方法提供一個閉包：

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

在內部，Laravel 已經為您忽略了某些型別的錯誤，例如由 404 HTTP 錯誤產生的例外、由來源不符產生的 403 HTTP 回應，或是由無效 CSRF tokens 產生的 419 HTTP 回應。如果您想指示 Laravel 停止忽略特定型別的例外，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `stopIgnoring` 例外方法：

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->stopIgnoring(HttpException::class);
})
```

<a name="rendering-exceptions"></a>
### 渲染例外

預設情況下，Laravel 的例外處理常式會自動將例外轉換為 HTTP 回應。不過，你可以自由地為特定型別的例外註冊自訂的渲染閉包。你可以在應用程式的 `bootstrap/app.php` 檔案中使用 `render` 例外方法來達成此目的。

傳遞給 `render` 方法的閉包應回傳 `Illuminate\Http\Response` 的實例，該實例可透過 `response` 輔助函式產生。Laravel 將透過檢查閉包的型別提示來判斷該閉包要渲染哪種型別的例外：

```php
use App\Exceptions\InvalidOrderException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });
})
```

你也可以使用 `render` 方法來覆寫內建 Laravel 或 Symfony 例外（例如 `NotFoundHttpException`）的渲染行為。如果傳入 `render` 方法的閉包沒有回傳值，將會使用 Laravel 預設的例外渲染：

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

在渲染例外時，Laravel 會根據請求的 `Accept` 標頭自動判斷該例外應渲染為 HTML 還是 JSON 回應。如果你想要自訂 Laravel 判斷是否渲染 HTML 或 JSON 例外回應的邏輯，可以使用 `shouldRenderJsonWhen` 方法：

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

在少數情況下，你可能需要自訂由 Laravel 例外處理常式所渲染的整個 HTTP 回應。為此，你可以使用 `respond` 方法註冊一個回應自訂閉包：

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

除了在應用程式的 `bootstrap/app.php` 檔案中定義自訂的回報和渲染行為之外，你也可以直接在應用程式的例外類別上定義 `report` 和 `render` 方法。當這些方法存在時，框架會自動呼叫它們：

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

如果你的例外繼承了一個已經可渲染的例外（例如內建的 Laravel 或 Symfony 例外），你可以從例外的 `render` 方法中回傳 `false`，以渲染該例外的預設 HTTP 回應：

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

如果你的例外包含只有在符合特定條件時才需要的自訂回報邏輯，你可能需要指示 Laravel 有時使用預設的例外處理設定來回報該例外。為此，你可以從例外的 `report` 方法中回傳 `false`：

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
> 你可以在 `report` 方法中對任何所需的依賴項目進行型別提示，它們將由 Laravel 的[服務容器](/docs/{{version}}/container)自動注入到該方法中。


<a name="throttling-reported-exceptions"></a>
### 限制例外回報頻率

如果你的應用程式回報了大量的例外，你可能希望限制實際記錄或傳送到應用程式外部錯誤追蹤服務的例外數量。

若要對例外進行隨機抽樣，你可以在應用程式的 `bootstrap/app.php` 檔案中使用 `throttle` 例外方法。`throttle` 方法接收一個應回傳 `Lottery` 實例的閉包：

```php
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        return Lottery::odds(1, 1000);
    });
})
```

也可以根據例外型別進行條件式抽樣。如果你只想抽樣特定例外類別的實例，可以僅針對該類別回傳 `Lottery` 實例：

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

你也可以透過回傳 `Limit` 實例而非 `Lottery` 來對記錄或傳送到外部錯誤追蹤服務的例外進行速率限制。這在你想防止突發的大量例外灌爆日誌時非常有用，例如當應用程式使用的第三方服務發生故障時：

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

預設情況下，限制將使用例外的類別名稱作為速率限制鍵值。你可以透過在 `Limit` 上使用 `by` 方法指定自訂的鍵值來進行自訂：

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

當然，你也可以針對不同的例外混合回傳 `Lottery` 和 `Limit` 實例：

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

某些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「找不到頁面」錯誤 (404)、「未授權錯誤」 (401)，甚至是開發人員產生的 500 錯誤。為了從應用程式中的任何位置產生此類回應，您可以使用 `abort` 輔助函式：

```php
abort(404);
```


<a name="custom-http-error-pages"></a>
### 自訂 HTTP 錯誤頁面

Laravel 讓您可以輕鬆為各種 HTTP 狀態碼顯示自訂錯誤頁面。例如，若要自訂 404 HTTP 狀態碼的錯誤頁面，請建立 `resources/views/errors/404.blade.php` 視圖模板。應用程式產生的所有 404 錯誤都將渲染此視圖。此目錄中的視圖名稱應與其對應的 HTTP 狀態碼相符。由 `abort` 函式引發的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將作為 `$exception` 變數傳遞給視圖：

```blade
<h2>{{ $exception->getMessage() }}</h2>
```

您可以使用 `vendor:publish` Artisan 指令發布 Laravel 的預設錯誤頁面模板。發布模板後，您可以根據自己的喜好進行自訂：

```shell
php artisan vendor:publish --tag=laravel-errors
```


<a name="fallback-http-error-pages"></a>
#### 備用 HTTP 錯誤頁面

您也可以為特定系列的 HTTP 狀態碼定義一個「備用 (fallback)」錯誤頁面。如果發生的特定 HTTP 狀態碼沒有對應的頁面，將會渲染此頁面。為此，請在應用程式的 `resources/views/errors` 目錄中定義一個 `4xx.blade.php` 模板和一個 `5xx.blade.php` 模板。

定義備用錯誤頁面時，備用頁面不會影響 `404`、`500` 和 `503` 錯誤回應，因為 Laravel 為這些狀態碼提供了內部的專用頁面。若要自訂為這些狀態碼渲染的頁面，您應該為它們各自單獨定義一個自訂錯誤頁面。