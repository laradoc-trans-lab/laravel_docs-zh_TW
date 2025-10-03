# 錯誤處理

- [簡介](#introduction)
- [設定](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外記錄等級](#exception-log-levels)
    - [依類型忽略例外](#ignoring-exceptions-by-type)
    - [渲染例外](#rendering-exceptions)
    - [可回報與可渲染的例外](#renderable-exceptions)
- [限制回報例外的頻率](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自訂 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當您開始一個新的 Laravel 專案時，錯誤與例外處理功能已經為您設定好；不過，您隨時都可以在應用程式的 `bootstrap/app.php` 檔案中使用 `withExceptions` 方法來管理例外如何被應用程式回報與渲染。

提供給 `withExceptions` 閉包的 `$exceptions` 物件是 `Illuminate\Foundation\Configuration\Exceptions` 的一個實例，負責管理您應用程式中的例外處理。我們將在本文件中更深入探討此物件。

<a name="configuration"></a>
## 設定

應用程式的 `config/app.php` 設定檔中的 `debug` 選項，決定了實際向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存於您的 `.env` 檔案中。

在本地開發期間，您應該將 `APP_DEBUG` 環境變數設定為 `true`。**在您的正式環境中，這個值應該永遠是 `false`。如果在正式環境中將此值設定為 `true`，您將冒著將敏感設定值洩露給應用程式終端使用者的風險。**

<a name="handling-exceptions"></a>
## 處理例外

<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於記錄例外或將其傳送至外部服務，例如 [Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，例外將根據您的 [日誌](/docs/{{version}}/logging) 設定進行記錄。不過，您可以依自己喜好的方式記錄例外。

如果您需要以不同方式回報不同類型的例外，您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `report` 例外方法來註冊一個閉包。當需要回報特定類型的例外時，將會執行此閉包。Laravel 會透過檢查閉包的類型提示來判斷閉包回報的例外類型：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->report(function (InvalidOrderException $e) {
        // ...
    });
})
```

當您使用 `report` 方法註冊自訂例外回報回呼時，Laravel 仍會使用應用程式的預設日誌設定記錄例外。如果您希望停止例外傳播到預設日誌堆疊，您可以在定義回報回呼時使用 `stop` 方法，或從回呼中傳回 `false`：

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
> 要自訂特定例外的回報功能，您也可以使用[可回報的例外](/docs/{{version}}/errors#renderable-exceptions)。

<a name="global-log-context"></a>
#### 全域日誌情境

如果可用，Laravel 會自動將當前使用者的 ID 作為情境資料，新增到每個例外的日誌訊息中。您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `context` 例外方法來定義自己的全域情境資料。此資訊將包含在應用程式寫入的每個例外日誌訊息中：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->context(fn () => [
        'foo' => 'bar',
    ]);
})
```

<a name="exception-log-context"></a>
#### 例外日誌情境

雖然將情境新增到每個日誌訊息中很有用，但有時特定例外可能具有您希望包含在日誌中的獨特情境。透過在應用程式的其中一個例外上定義 `context` 方法，您可以指定任何與該例外相關的資料，這些資料應新增到例外的日誌條目中：

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
#### `report` 輔助函數

有時您可能需要回報例外，但仍繼續處理當前請求。`report` 輔助函數可讓您快速回報例外，而無需向使用者渲染錯誤頁面：

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

如果您在整個應用程式中使用 `report` 函數，您可能會偶爾多次回報相同的例外，從而在日誌中建立重複條目。

如果您希望確保一個例外的單一實例只被回報一次，您可以在應用程式的 `bootstrap/app.php` 檔案中調用 `dontReportDuplicates` 例外方法：

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReportDuplicates();
})
```

現在，當 `report` 輔助函數以相同的例外實例被呼叫時，只有第一次呼叫會被回報：

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
### 例外記錄等級

當訊息寫入您的應用程式的 [日誌](/docs/{{version}}/logging) 時，訊息會以指定的 [日誌等級](/docs/{{version}}/logging#log-levels) 寫入，這表示所記錄訊息的嚴重性或重要性。

如上所述，即使您使用 `report` 方法註冊了自訂例外回報回呼，Laravel 仍會使用應用程式的預設日誌設定記錄例外；然而，由於日誌等級有時會影響訊息記錄的通道，您可能希望設定某些例外記錄的日誌等級。

為此，您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `level` 例外方法。此方法將例外類型作為其第一個參數，並將日誌等級作為其第二個參數：

```php
use PDOException;
use Psr\Log\LogLevel;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->level(PDOException::class, LogLevel::CRITICAL);
})
```

<a name="ignoring-exceptions-by-type"></a>
### 依類型忽略例外

在建構應用程式時，有些例外類型是您永遠不想回報的。要忽略這些例外，您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `dontReport` 例外方法。提供給此方法的任何類別將永遠不會被回報；但是，它們仍然可能具有自訂的渲染邏輯：

```php
use App\Exceptions\InvalidOrderException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->dontReport([
        InvalidOrderException::class,
    ]);
})
```

或者，您也可以簡單地使用 `Illuminate\Contracts\Debug\ShouldntReport` 介面「標記」例外類別。當例外被此介面標記時，它將永遠不會被 Laravel 的例外處理器回報：

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

在內部，Laravel 已經為您忽略了一些錯誤類型，例如因 404 HTTP 錯誤或無效 CSRF token 產生 419 HTTP 回應的例外。如果您想指示 Laravel 停止忽略給定類型的例外，您可以使用應用程式的 `bootstrap/app.php` 檔案中的 `stopIgnoring` 例外方法：

```php
use Symfony\Component\HttpKernel\Exception\HttpException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->stopIgnoring(HttpException::class);
})
```

<a name="rendering-exceptions"></a>
### 渲染例外

預設情況下，Laravel 例外處理器會將例外轉換為 HTTP 回應。不過，您可以自由地為特定類型的例外註冊一個自訂的渲染閉包。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `render` 例外方法來完成此操作。

傳遞給 `render` 方法的閉包應回傳 `Illuminate\Http\Response` 的實例，該實例可透過 `response` 輔助函數產生。Laravel 會透過檢查閉包的類型提示來判斷該閉包會渲染何種類型的例外。

```php
use App\Exceptions\InvalidOrderException;
use Illuminate\Http\Request;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidOrderException $e, Request $request) {
        return response()->view('errors.invalid-order', status: 500);
    });
})
```

您也可以使用 `render` 方法來覆寫內建 Laravel 或 Symfony 例外（例如 `NotFoundHttpException`）的渲染行為。如果傳遞給 `render` 方法的閉包沒有回傳值，則會使用 Laravel 的預設例外渲染。

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
#### 以 JSON 格式渲染例外

渲染例外時，Laravel 會根據請求的 `Accept` 標頭自動判斷例外應渲染為 HTML 或 JSON 回應。如果您想自訂 Laravel 如何判斷是渲染 HTML 還是 JSON 例外回應，您可以使用 `shouldRenderJsonWhen` 方法：

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

很少會需要自訂 Laravel 例外處理器所渲染的整個 HTTP 回應。為此，您可以使用 `respond` 方法註冊一個回應自訂閉包：

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

您可以在應用程式的例外中直接定義 `report` 和 `render` 方法，而非在應用程式的 `bootstrap/app.php` 檔案中定義自訂的回報與渲染行為。當這些方法存在時，框架會自動呼叫它們：

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

如果您的例外繼承自已可渲染的例外 (例如內建的 Laravel 或 Symfony 例外)，您可以從例外的 `render` 方法回傳 `false` 以渲染該例外的預設 HTTP 回應：

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

如果您的例外包含僅在滿足特定條件時才需要的自訂回報邏輯，您可能需要指示 Laravel 有時使用預設例外處理設定來回報該例外。為此，您可以從例外的 `report` 方法回傳 `false`：

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
> 您可以對 `report` 方法所需的任何依賴項進行類型提示，它們將由 Laravel 的 [服務容器](/docs/{{version}}/container) 自動注入到該方法中。

<a name="throttling-reported-exceptions"></a>
### 限制回報例外的頻率

如果您的應用程式回報了大量的例外，您可能希望限制實際被記錄或發送到應用程式外部錯誤追蹤服務的例外數量。

若要對例外採取隨機抽樣率，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `throttle` 例外方法。`throttle` 方法接收一個閉包，該閉包應回傳一個 `Lottery` 實例：

```php
use Illuminate\Support\Lottery;
use Throwable;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->throttle(function (Throwable $e) {
        return Lottery::odds(1, 1000);
    });
})
```

也可以根據例外類型進行有條件的抽樣。如果您只想抽樣特定例外類別的實例，您可以僅為該類別回傳一個 `Lottery` 實例：

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

您也可以透過回傳 `Limit` 實例而不是 `Lottery` 來限制被記錄或發送到外部錯誤追蹤服務的例外速率。這在您想防止例外突然大量湧入日誌時很有用，例如當您的應用程式使用的第三方服務停機時：

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

預設情況下，限制會使用例外的類別作為速率限制鍵。您可以透過在 `Limit` 上使用 `by` 方法指定自己的鍵來進行自訂：

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

當然，您可以針對不同的例外回傳 `Lottery` 和 `Limit` 實例的組合：

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

有些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「頁面不存在」錯誤 (404)、一個「未授權錯誤」 (401)，甚至是開發者產生的 500 錯誤。為了在應用程式的任何地方產生這樣的響應，您可以使用 `abort` 輔助函式：

```php
abort(404);
```


<a name="custom-http-error-pages"></a>
### 自訂 HTTP 錯誤頁面

Laravel 讓您可以輕鬆地為各種 HTTP 狀態碼顯示自訂錯誤頁面。例如，要自訂 404 HTTP 狀態碼的錯誤頁面，請建立一個 `resources/views/errors/404.blade.php` 視圖模板。此視圖將會被應用程式產生的所有 404 錯誤渲染。此目錄中的視圖應根據其對應的 HTTP 狀態碼來命名。由 `abort` 函式引發的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將作為 `$exception` 變數傳遞給視圖：

```blade
<h2>{{ $exception->getMessage() }}</h2>
```

您可以使用 `vendor:publish` Artisan 命令來發布 Laravel 的預設錯誤頁面模板。一旦模板發布後，您便可以根據您的喜好進行自訂：

```shell
php artisan vendor:publish --tag=laravel-errors
```


<a name="fallback-http-error-pages"></a>
#### 備用 HTTP 錯誤頁面

您也可以為給定系列的 HTTP 狀態碼定義一個「備用」錯誤頁面。如果沒有對應特定 HTTP 狀態碼的頁面，此頁面將會被渲染。為此，請在您的應用程式 `resources/views/errors` 目錄中定義一個 `4xx.blade.php` 模板和一個 `5xx.blade.php` 模板。

在定義備用錯誤頁面時，備用頁面不會影響 `404`、`500` 和 `503` 錯誤響應，因為 Laravel 為這些狀態碼提供了內部專用頁面。若要自訂這些狀態碼渲染的頁面，您應該為每個狀態碼單獨定義一個自訂錯誤頁面。