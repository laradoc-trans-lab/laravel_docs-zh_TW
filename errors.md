# 錯誤處理

- [簡介](#introduction)
- [配置](#configuration)
- [處理例外](#handling-exceptions)
    - [回報例外](#reporting-exceptions)
    - [例外日誌層級](#exception-log-levels)
    - [依類型忽略例外](#ignoring-exceptions-by-type)
    - [彩現例外](#rendering-exceptions)
    - [可回報與可彩現的例外](#renderable-exceptions)
- [限制回報的例外](#throttling-reported-exceptions)
- [HTTP 例外](#http-exceptions)
    - [自訂 HTTP 錯誤頁面](#custom-http-error-pages)

<a name="introduction"></a>
## 簡介

當您啟動新的 Laravel 專案時，錯誤與例外處理功能已經為您配置完成；然而，在任何時候，您都可以在應用程式的 `bootstrap/app.php` 檔案中使用 `withExceptions` 方法來管理例外如何被您的應用程式回報與彩現。

提供給 `withExceptions` 閉包的 `$exceptions` 物件是 `Illuminate\Foundation\Configuration\Exceptions` 的一個實例，它負責管理您應用程式中的例外處理。我們將在本文件中更深入地探討這個物件。


<a name="configuration"></a>
## 配置

您 `config/app.php` 配置檔中的 `debug` 選項，決定有多少錯誤資訊會實際顯示給使用者。預設情況下，此選項會遵循 `APP_DEBUG` 環境變數的值，該變數儲存在您的 `.env` 檔案中。

在本地開發期間，您應該將 `APP_DEBUG` 環境變數設為 `true`。**在您的正式環境中，這個值應該永遠是 `false`。如果在正式環境中將此值設為 `true`，您可能會將敏感的配置值暴露給應用程式的終端使用者。**

<a name="handling-exceptions"></a>
## 處理例外


<a name="reporting-exceptions"></a>
### 回報例外

在 Laravel 中，例外回報用於記錄例外或將其發送到外部服務，例如 [Sentry](https://github.com/getsentry/sentry-laravel) 或 [Flare](https://flareapp.io)。預設情況下，例外將根據您的 [日誌配置](/docs/{{version}}/logging) 進行記錄。但是，您可以自由地以您希望的任何方式記錄例外。

如果您需要以不同的方式回報不同類型的例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `report` 例外方法來註冊一個閉包，該閉包應在需要回報給定類型的例外時執行。Laravel 將透過檢查閉包的型別提示來判斷閉包回報的例外類型：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        });
    })

當您使用 `report` 方法註冊自訂例外回報回呼時，Laravel 仍將使用應用程式的預設日誌配置來記錄例外。如果您希望阻止例外傳播到預設日誌堆疊，您可以在定義回報回呼時使用 `stop` 方法，或從回呼中回傳 `false`：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        })->stop();

        $exceptions->report(function (InvalidOrderException $e) {
            return false;
        });
    })

> [!NOTE]
> 若要自訂給定例外的回報行為，您也可以利用 [可回報例外](/docs/{{version}}/errors#renderable-exceptions)。


<a name="global-log-context"></a>
#### 全域日誌上下文

如果可用，Laravel 會自動將目前使用者的 ID 添加到每個例外的日誌訊息中作為上下文資料。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `context` 例外方法來定義自己的全域上下文資料。此資訊將包含在應用程式寫入的每個例外日誌訊息中：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->context(fn () => [
            'foo' => 'bar',
        ]);
    })


<a name="exception-log-context"></a>
#### 例外日誌上下文

雖然將上下文添加到每個日誌訊息中很有用，但有時特定例外可能具有您想要包含在日誌中的獨特上下文。透過在應用程式的其中一個例外上定義 `context` 方法，您可以指定與該例外相關的任何資料，這些資料應添加到例外的日誌條目中：

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

有時您可能需要回報例外但繼續處理目前的請求。`report` 輔助函式允許您快速回報例外，而無需向使用者彩現錯誤頁面：

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
#### 排除重複的回報例外

如果您在整個應用程式中使用了 `report` 函式，您可能會偶爾多次回報同一個例外，從而在日誌中建立重複的條目。

如果您想確保一個例外的單一實例只被回報一次，您可以在應用程式的 `bootstrap/app.php` 檔案中呼叫 `dontReportDuplicates` 例外方法：

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReportDuplicates();
    })

現在，當 `report` 輔助函式被同一個例外實例呼叫時，只有第一次呼叫會被回報：

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

當訊息寫入應用程式的 [日誌](/docs/{{version}}/logging) 時，訊息會以指定的 [日誌層級](/docs/{{version}}/logging#log-levels) 寫入，這表示日誌訊息的嚴重性或重要性。

如上所述，即使您使用 `report` 方法註冊了自訂例外回報回呼，Laravel 仍將使用應用程式的預設日誌配置來記錄例外；但是，由於日誌層級有時會影響訊息記錄的管道，您可能希望配置某些例外記錄的日誌層級。

為了實現這一點，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `level` 例外方法。此方法接收例外類型作為其第一個參數，並接收日誌層級作為其第二個參數：

    use PDOException;
    use Psr\Log\LogLevel;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->level(PDOException::class, LogLevel::CRITICAL);
    })


<a name="ignoring-exceptions-by-type"></a>
### 依類型忽略例外

在建構應用程式時，某些類型的例外是您永遠不想回報的。為了忽略這些例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `dontReport` 例外方法。提供給此方法的任何類別將永遠不會被回報；但是，它們仍可能具有自訂彩現邏輯：

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

在內部，Laravel 已經為您忽略了某些類型的錯誤，例如由於 404 HTTP 錯誤或無效 CSRF 權杖產生 419 HTTP 回應而導致的例外。如果您希望指示 Laravel 停止忽略給定類型的例外，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `stopIgnoring` 例外方法：

    use Symfony\Component\HttpKernel\Exception\HttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->stopIgnoring(HttpException::class);
    })

<a name="rendering-exceptions"></a>
### 彩現例外

預設情況下，Laravel 例外處理器會將例外轉換為 HTTP 響應。然而，您可以自由地為給定類型的例外註冊一個自訂彩現閉包。您可以透過在應用程式的 `bootstrap/app.php` 檔案中使用 `render` 例外方法來完成此操作。

傳遞給 `render` 方法的閉包應回傳 `Illuminate\Http\Response` 的實例，此實例可透過 `response` 輔助函數生成。Laravel 將透過檢查閉包的型別提示來判斷閉包彩現何種類型的例外：

    use App\Exceptions\InvalidOrderException;
    use Illuminate\Http\Request;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (InvalidOrderException $e, Request $request) {
            return response()->view('errors.invalid-order', status: 500);
        });
    })

您也可以使用 `render` 方法來覆寫內建的 Laravel 或 Symfony 例外（例如 `NotFoundHttpException`）的彩現行為。如果傳遞給 `render` 方法的閉包沒有回傳值，將會使用 Laravel 預設的例外彩現：

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
#### 將例外彩現為 JSON

彩現例外時，Laravel 會根據請求的 `Accept` 標頭自動判斷例外應彩現為 HTML 或 JSON 響應。如果您想自訂 Laravel 如何判斷彩現 HTML 或 JSON 例外響應，可以使用 `shouldRenderJsonWhen` 方法：

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
#### 自訂例外響應

很少見地，您可能需要自訂由 Laravel 例外處理器彩現的整個 HTTP 響應。為此，您可以透過使用 `respond` 方法來註冊一個響應自訂閉包：

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
### 可回報與可彩現的例外

您可以不在應用程式的 `bootstrap/app.php` 檔案中定義自訂回報和彩現行為，而是直接在應用程式的例外上定義 `report` 和 `render` 方法。當這些方法存在時，框架會自動呼叫它們：

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

如果您的例外繼承了一個已是可彩現的例外，例如內建的 Laravel 或 Symfony 例外，您可以從例外的 `render` 方法中回傳 `false`，以彩現例外的預設 HTTP 響應：

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

如果您的例外包含僅在滿足特定條件時才需要的自訂回報邏輯，您可能需要指示 Laravel 有時使用預設的例外處理配置來回報例外。為此，您可以從例外的 `report` 方法中回傳 `false`：

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
> 您可以對 `report` 方法的任何所需依賴項進行型別提示，Laravel 的 [服務容器](/docs/{{version}}/container) 將自動將它們注入到該方法中。

<a name="throttling-reported-exceptions"></a>
### 限制回報的例外

如果您的應用程式回報非常大量的例外，您可能會希望限制實際記錄或傳送至您應用程式的外部錯誤追蹤服務的例外數量。

若要對例外進行隨機取樣，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `throttle` 例外方法。`throttle` 方法會接收一個閉包，該閉包應回傳一個 `Lottery` 實例：

    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return Lottery::odds(1, 1000);
        });
    })

也可以根據例外類型進行條件式取樣。如果您只想取樣特定例外類別的實例，您可以只為該類別回傳一個 `Lottery` 實例：

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

您也可以透過回傳 `Limit` 實例而非 `Lottery`，來限制記錄或傳送至外部錯誤追蹤服務的例外回報率。這對於防止例外突然大量湧入您的日誌非常有用，例如，當您的應用程式使用的第三方服務停機時：

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

預設情況下，限制會使用例外的類別作為速率限制鍵。您可以透過在 `Limit` 上使用 `by` 方法來指定您自己的鍵來自訂此行為：

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

當然，您可以為不同的例外回傳 `Lottery` 和 `Limit` 實例的組合：

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

有些例外描述了來自伺服器的 HTTP 錯誤碼。例如，這可能是「找不到頁面」錯誤 (404)、「未經授權錯誤」 (401)，甚至是開發者生成的 500 錯誤。為了在應用程式的任何地方產生此類回應，您可以使用 `abort` 輔助函式：

    abort(404);


<a name="custom-http-error-pages"></a>
### 自訂 HTTP 錯誤頁面

Laravel 讓為各種 HTTP 狀態碼顯示自訂錯誤頁面變得容易。例如，為了自訂 404 HTTP 狀態碼的錯誤頁面，請建立一個 `resources/views/errors/404.blade.php` 視圖範本。這個視圖將會被用於應用程式產生的所有 404 錯誤。這個目錄中的視圖應該以其對應的 HTTP 狀態碼命名。由 `abort` 函式引發的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例將會以 `$exception` 變數的形式傳遞到視圖：

    <h2>{{ $exception->getMessage() }}</h2>

您可以使用 `vendor:publish` Artisan 命令發佈 Laravel 預設的錯誤頁面範本。範本發佈後，您可以根據自己的喜好進行自訂：

```shell
php artisan vendor:publish --tag=laravel-errors
```


<a name="fallback-http-error-pages"></a>
#### 備用 HTTP 錯誤頁面

您也可以為指定系列的 HTTP 狀態碼定義一個「備用」錯誤頁面。如果沒有與特定發生的 HTTP 狀態碼相對應的頁面，此頁面將會被彩現。為了實現這一點，請在應用程式的 `resources/views/errors` 目錄中定義一個 `4xx.blade.php` 範本和一個 `5xx.blade.php` 範本。

當定義備用錯誤頁面時，備用頁面不會影響 `404`、`500` 和 `503` 錯誤回應，因為 Laravel 對這些狀態碼有內部專用的頁面。為了自訂這些狀態碼所彩現的頁面，您應該為每個狀態碼單獨定義一個自訂錯誤頁面。