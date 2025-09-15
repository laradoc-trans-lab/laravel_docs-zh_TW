# 錯誤處理

- [簡介](#introduction)
- [設定](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外日誌層級](#exception-log-levels)
    - [依類型忽略例外](#ignoring-exceptions-by-type)
    - [渲染例外](#rendering-exceptions)
    - [可回報與可渲染的例外](#renderable-exceptions)
- [節流回報的例外](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自訂 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當您啟動一個新的 Laravel 專案時，錯誤與例外處理功能已經為您配置完成；然而，在任何時候，您都可以使用應用程式 `bootstrap/app.php` 檔案中的 `withExceptions` 方法來管理應用程式如何回報與渲染例外。

提供給 `withExceptions` 閉包的 `$exceptions` 物件是 `Illuminate\Foundation\Configuration\Exceptions` 的實例，負責管理應用程式中的例外處理。我們將在本文件中深入探討這個物件。

<a name="configuration"></a>
## 設定

`config/app.php` 設定檔中的 `debug` 選項決定了錯誤的資訊會實際顯示多少給使用者。預設情況下，此選項會設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在您的 `.env` 檔案中。

在本地開發期間，您應該將 `APP_DEBUG` 環境變數設定為 `true`。**在您的正式環境中，此值應始終為 `false`。如果該值在正式環境中設定為 `true`，您將面臨向應用程式終端使用者暴露敏感設定值的風險。**

<a name="handling-exceptions"></a>
## 處理例外

<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於記錄例外或將其發送到外部服務，例如 [Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，例外將根據您的[日誌](/docs/{{version}}/logging)設定進行記錄。然而，您可以自由地以任何您希望的方式記錄例外。

如果您需要以不同方式回報不同類型的例外，您可以使用應用程式 `bootstrap/app.php` 中的 `report` 例外方法來註冊一個閉包，該閉包應在需要回報給定類型例外時執行。Laravel 將透過檢查閉包的型別提示來確定閉包回報的例外類型：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        });
    })

當您使用 `report` 方法註冊自訂例外回報回呼時，Laravel 仍將使用應用程式的預設日誌設定來記錄例外。如果您希望停止例外傳播到預設日誌堆疊，您可以在定義回報回呼時使用 `stop` 方法，或從回呼中返回 `false`：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        })->stop();

        $exceptions->report(function (InvalidOrderException $e) {
            return false;
        });
    })

> [!NOTE]
> 若要自訂特定例外的回報行為，您也可以利用[可回報例外](/docs/{{version}}/errors#renderable-exceptions)。

<a name="global-log-context"></a>
#### 全域日誌 Context

如果可用，Laravel 會自動將當前使用者的 ID 作為 Context 資料添加到每個例外的日誌訊息中。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `context` 例外方法來定義自己的全域 Context 資料。此資訊將包含在應用程式寫入的每個例外的日誌訊息中：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->context(fn () => [
            'foo' => 'bar',
        ]);
    })

<a name="exception-log-context"></a>
#### 例外日誌 Context

雖然將 Context 添加到每個日誌訊息中可能很有用，但有時特定例外可能具有您希望包含在日誌中的獨特 Context。透過在應用程式的其中一個例外上定義 `context` 方法，您可以指定與該例外相關的任何資料，這些資料應添加到例外的日誌項目中：

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

<a name="the-report-helper"></a>
#### `report` 輔助函式

有時您可能需要回報例外但繼續處理當前請求。`report` 輔助函式允許您快速回報例外，而無需向使用者渲染錯誤頁面：

    public function isValid(string $value): bool
    {
        try {
            // Validate the value...
        } catch (Throwable $e) {
            report($e);

            return false;
        }
    }

<a name="deduplicating-reported-exceptions"></a>
#### 排除重複回報的例外

如果您在整個應用程式中使用 `report` 函式，您偶爾可能會多次回報相同的例外，從而在日誌中建立重複的項目。

如果您想確保單一例外實例只會被回報一次，您可以在應用程式 `bootstrap/app.php` 檔案中呼叫 `dontReportDuplicates` 例外方法：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReportDuplicates();
    })

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

當訊息寫入應用程式的[日誌](/docs/{{version}}/logging)時，訊息會以指定的[日誌層級](/docs/{{version}}/logging#log-levels)寫入，該層級表示所記錄訊息的嚴重性或重要性。

如上所述，即使您使用 `report` 方法註冊自訂例外回報回呼，Laravel 仍將使用應用程式的預設日誌設定來記錄例外；然而，由於日誌層級有時會影響訊息記錄的通道，您可能希望設定某些例外記錄的日誌層級。

為此，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `level` 例外方法。此方法將例外類型作為其第一個引數，日誌層級作為其第二個引數：

    use PDOException;
    use Psr\Log\LogLevel;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->level(PDOException::class, LogLevel::CRITICAL);
    })

<a name="ignoring-exceptions-by-type"></a>
### 依類型忽略例外

在建構應用程式時，有些類型的例外是您永遠不想回報的。要忽略這些例外，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `dontReport` 例外方法。提供給此方法的任何類別將永遠不會被回報；然而，它們可能仍具有自訂渲染邏輯：

    use App\Exceptions\InvalidOrderException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReport([
            InvalidOrderException::class,
        ]);
    })

或者，您可以簡單地使用 `Illuminate\Contracts\Debug\ShouldntReport` 介面「標記」例外類別。當例外被此介面標記時，它將永遠不會被 Laravel 的例外處理器回報：

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

在內部，Laravel 已經為您忽略了一些類型的錯誤，例如由 404 HTTP 錯誤或無效 CSRF 權杖產生的 419 HTTP 回應所導致的例外。如果您想指示 Laravel 停止忽略給定類型的例外，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `stopIgnoring` 例外方法：

    use Symfony\Component\HttpKernel\Exception\HttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->stopIgnoring(HttpException::class);
    })

<a name="rendering-exceptions"></a>
### 渲染例外

預設情況下，Laravel 例外處理器會為您將例外轉換為 HTTP 回應。然而，您可以自由地為給定類型的例外註冊自訂渲染閉包。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `render` 例外方法來實現此目的。

傳遞給 `render` 方法的閉包應返回 `Illuminate\Http\Response` 的實例，該實例可以透過 `response` 輔助函式產生。Laravel 將透過檢查閉包的型別提示來確定閉包渲染的例外類型：

    use App\Exceptions\InvalidOrderException;
    use Illuminate\Http\Request;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (InvalidOrderException $e, Request $request) {
            return response()->view('errors.invalid-order', status: 500);
        });
    })

您也可以使用 `render` 方法來覆寫內建 Laravel 或 Symfony 例外（例如 `NotFoundHttpException`）的渲染行為。如果傳遞給 `render` 方法的閉包沒有返回值，將使用 Laravel 的預設例外渲染：

    use Illuminate\Http\Request;
    use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (NotFoundHttpException $e, Request $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Record not found.'
                ], 404);
            }
        });
    })

<a name="rendering-exceptions-as-json"></a>
#### 將例外渲染為 JSON

渲染例外時，Laravel 會根據請求的 `Accept` 標頭自動判斷例外應渲染為 HTML 或 JSON 回應。如果您想自訂 Laravel 如何判斷渲染 HTML 或 JSON 例外回應，您可以使用 `shouldRenderJsonWhen` 方法：

    use Illuminate\Http\Request;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
            if ($request->is('admin/*')) {
                return true;
            }

            return $request->expectsJson();
        });
    })

<a name="customizing-the-exception-response"></a>
#### 自訂例外回應

在極少數情況下，您可能需要自訂 Laravel 例外處理器渲染的整個 HTTP 回應。為此，您可以使用 `respond` 方法註冊一個回應自訂閉包：

    use Symfony\Component\HttpFoundation\Response;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->respond(function (Response $response) {
            if ($response->getStatusCode() === 419) {
                return back()->with([
                    'message' => 'The page expired, please try again.',
                ]);
            }

            return $response;
        });
    })

<a name="renderable-exceptions"></a>
### 可回報與可渲染的例外

除了在應用程式 `bootstrap/app.php` 檔案中定義自訂回報和渲染行為之外，您還可以直接在應用程式的例外上定義 `report` 和 `render` 方法。當這些方法存在時，框架會自動呼叫它們：

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
         * Render the exception into an HTTP response.
         */
        public function render(Request $request): Response
        {
            return response(/* ... */);
        }
    }

如果您的例外繼承自已經可渲染的例外，例如內建的 Laravel 或 Symfony 例外，您可以從例外的 `render` 方法返回 `false` 以渲染例外的預設 HTTP 回應：

    /**
     * Render the exception into an HTTP response.
     */
    public function render(Request $request): Response|bool
    {
        if (/** Determine if the exception needs custom rendering */) {

            return response(/* ... */);
        }

        return false;
    }

如果您的例外包含僅在滿足特定條件時才需要的自訂回報邏輯，您可能需要指示 Laravel 有時使用預設例外處理設定來回報例外。為此，您可以從例外的 `report` 方法返回 `false`：

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

> [!NOTE]
> 您可以對 `report` 方法的任何所需依賴項進行型別提示，它們將由 Laravel 的 [Service Container](/docs/{{version}}/container) 自動注入到該方法中。

<a name="throttling-reported-exceptions"></a>
### 節流回報的例外

如果您的應用程式回報了大量的例外，您可能希望限制實際記錄或發送到應用程式外部錯誤追蹤服務的例外數量。

要對例外進行隨機抽樣，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `throttle` 例外方法。`throttle` 方法接收一個閉包，該閉包應返回一個 `Lottery` 實例：

    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return Lottery::odds(1, 1000);
        });
    })

也可以根據例外類型進行條件抽樣。如果您只想抽樣特定例外類別的實例，您可以只為該類別返回一個 `Lottery` 實例：

    use App\Exceptions\ApiMonitoringException;
    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof ApiMonitoringException) {
                return Lottery::odds(1, 1000);
            }
        });
    })

您還可以透過返回 `Limit` 實例而不是 `Lottery` 來限制記錄或發送到外部錯誤追蹤服務的例外。這在您想要防止例外突然爆發淹沒日誌時很有用，例如，當您的應用程式使用的第三方服務關閉時：

    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof BroadcastException) {
                return Limit::perMinute(300);
            }
        });
    })

預設情況下，限制將使用例外的類別作為速率限制鍵。您可以透過在 `Limit` 上使用 `by` 方法指定自己的鍵來自訂此行為：

    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof BroadcastException) {
                return Limit::perMinute(300)->by($e->getMessage());
            }
        });
    })

當然，您可以為不同的例外返回 `Lottery` 和 `Limit` 實例的混合：

    use App\Exceptions\ApiMonitoringException;
    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return match (true) {
                $e instanceof BroadcastException => Limit::perMinute(300),
                $e instanceof ApiMonitoringException => Lottery::odds(1, 1000),
                default => Limit::none(),
            };
        });
    })

<a name="http-exceptions"></a>
## HTTP 例外

有些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「找不到頁面」錯誤 (404)、「未經授權錯誤」 (401)，甚至是開發者產生的 500 錯誤。為了在應用程式的任何地方產生此類回應，您可以使用 `abort` 輔助函式：

    abort(404);

<a name="custom-http-error-pages"></a>
### 自訂 HTTP 錯誤頁面

Laravel 讓為各種 HTTP 狀態碼顯示自訂錯誤頁面變得容易。例如，要自訂 404 HTTP 狀態碼的錯誤頁面，請建立 `resources/views/errors/404.blade.php` 視圖模板。此視圖將為應用程式產生的所有 404 錯誤渲染。此目錄中的視圖應命名為與其對應的 HTTP 狀態碼相符。由 `abort` 函式引發的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將作為 `$exception` 變數傳遞給視圖：

    <h2>{{ $exception->getMessage() }}</h2>

您可以使用 `vendor:publish` Artisan 命令發布 Laravel 的預設錯誤頁面模板。一旦模板發布，您可以根據自己的喜好進行自訂：

```shell
php artisan vendor:publish --tag=laravel-errors
```

<a name="fallback-http-error-pages"></a>
#### 備用 HTTP 錯誤頁面

您還可以為給定系列的 HTTP 狀態碼定義一個「備用」錯誤頁面。如果沒有對應的頁面用於發生的特定 HTTP 狀態碼，則會渲染此頁面。為此，請在應用程式的 `resources/views/errors` 目錄中定義 `4xx.blade.php` 模板和 `5xx.blade.php` 模板。

定義備用錯誤頁面時，備用頁面不會影響 `404`、`500` 和 `503` 錯誤回應，因為 Laravel 為這些狀態碼提供了內部專用頁面。要自訂為這些狀態碼渲染的頁面，您應該為每個狀態碼單獨定義一個自訂錯誤頁面。
