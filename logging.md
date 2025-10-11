# 記錄

- [簡介](#introduction)
- [設定](#configuration)
    - [可用的通道驅動](#available-channel-drivers)
    - [通道先決條件](#channel-prerequisites)
    - [記錄棄用警告](#logging-deprecation-warnings)
- [建立記錄堆疊](#building-log-stacks)
- [寫入記錄訊息](#writing-log-messages)
    - [上下文資訊](#contextual-information)
    - [寫入特定通道](#writing-to-specific-channels)
- [Monolog 通道自訂](#monolog-channel-customization)
    - [為通道自訂 Monolog](#customizing-monolog-for-channels)
    - [建立 Monolog 處理器通道](#creating-monolog-handler-channels)
    - [透過工廠建立自訂通道](#creating-custom-channels-via-factories)
- [使用 Pail 追蹤記錄訊息](#tailing-log-messages-using-pail)
    - [安裝](#pail-installation)
    - [用法](#pail-usage)
    - [篩選記錄](#pail-filtering-logs)

<a name="introduction"></a>
## 簡介

為協助您深入了解應用程式中正在發生的事情，Laravel 提供強大的記錄服務，讓您可以將訊息記錄到檔案、系統錯誤記錄，甚至發送到 Slack 以通知您的整個團隊。

Laravel 的記錄功能基於「通道」。每個通道代表一種特定的記錄資訊方式。例如，`single` 通道將記錄檔案寫入單一記錄檔，而 `slack` 通道則將記錄訊息發送到 Slack。記錄訊息可以根據其嚴重性寫入多個通道。

底層，Laravel 利用 [Monolog](https://github.com/Seldaek/monolog) 函式庫，它提供對各種強大記錄處理器的支援。Laravel 讓配置這些處理器變得輕而易舉，讓您可以混搭它們來自訂應用程式的記錄處理。

<a name="configuration"></a>
## 設定

所有控制應用程式記錄行為的設定選項都位於 `config/logging.php` 設定檔中。此檔案允許您配置應用程式的記錄通道，因此請務必查看每個可用通道及其選項。我們將在下面回顧一些常見選項。

預設情況下，Laravel 在記錄訊息時將使用 `stack` 通道。`stack` 通道用於將多個記錄通道聚合到單一通道中。有關建立堆疊的更多資訊，請查閱 [下方文件](#building-log-stacks)。

<a name="available-channel-drivers"></a>
### 可用的通道驅動

每個記錄通道都由一個「驅動」提供動力。該驅動決定記錄訊息的實際方式和位置。以下記錄通道驅動在每個 Laravel 應用程式中都可用。大多數這些驅動的條目已經存在於應用程式的 `config/logging.php` 設定檔中，因此請務必查看此檔案以熟悉其內容：

<div class="overflow-auto">

| Name         | Description                                                          |
| ------------ | -------------------------------------------------------------------- |
| `custom`     | 一個呼叫指定工廠來建立通道的驅動。                                   |
| `daily`      | 一個基於 `RotatingFileHandler` 的 Monolog 驅動，每日輪替。           |
| `errorlog`   | 一個基於 `ErrorLogHandler` 的 Monolog 驅動。                         |
| `monolog`    | 一個 Monolog 工廠驅動，可以使用任何受支援的 Monolog 處理器。         |
| `papertrail` | 一個基於 `SyslogUdpHandler` 的 Monolog 驅動。                        |
| `single`     | 一個基於單一檔案或路徑的記錄器通道 (`StreamHandler`)。                 |
| `slack`      | 一個基於 `SlackWebhookHandler` 的 Monolog 驅動。                     |
| `stack`      | 一個用於建立「多通道」通道的包裝器。                                 |
| `syslog`     | 一個基於 `SyslogHandler` 的 Monolog 驅動。                           |

</div>

> [!NOTE]  
> 查閱 [進階通道自訂](#monolog-channel-customization) 文件，了解更多關於 `monolog` 和 `custom` 驅動的資訊。

<a name="configuring-the-channel-name"></a>
#### 配置通道名稱

預設情況下，Monolog 會以與目前環境（例如 `production` 或 `local`）匹配的「通道名稱」進行實例化。要更改此值，您可以將 `name` 選項添加到通道的設定中：

    'stack' => [
        'driver' => 'stack',
        'name' => 'channel-name',
        'channels' => ['single', 'slack'],
    ],

<a name="channel-prerequisites"></a>
### 通道先決條件

<a name="configuring-the-single-and-daily-channels"></a>
#### 配置 Single 與 Daily 通道

`single` 和 `daily` 通道有三個可選的設定選項：`bubble`、`permission` 和 `locking`。

<div class="overflow-auto">

| Name         | Description                                                                   | Default |
| ------------ | ----------------------------------------------------------------------------- | ------- |
| `bubble`     | 指示訊息在處理後是否應冒泡到其他通道。                                        | `true`  |
| `locking`    | 嘗試在寫入記錄檔案之前鎖定它。                                                | `false` |
| `permission` | 記錄檔案的權限。                                                              | `0644`  |

</div>

此外，`daily` 通道的保留策略可以透過 `LOG_DAILY_DAYS` 環境變數或設定 `days` 設定選項來配置。

<div class="overflow-auto">

| Name   | Description                                                 | Default |
| ------ | ----------------------------------------------------------- | ------- |
| `days` | 每日記錄檔案應保留的天數。                                    | `14`    |

</div>

<a name="configuring-the-papertrail-channel"></a>
#### 配置 Papertrail 通道

`papertrail` 通道需要 `host` 和 `port` 設定選項。這些可以透過 `PAPERTRAIL_URL` 和 `PAPERTRAIL_PORT` 環境變數來定義。您可以從 [Papertrail](https://help.papertrailapp.com/kb/configuration/configuring-centralized-logging-from-php-apps/#send-events-from-php-app) 獲取這些值。

<a name="configuring-the-slack-channel"></a>
#### 配置 Slack 通道

`slack` 通道需要 `url` 設定選項。此值可以透過 `LOG_SLACK_WEBHOOK_URL` 環境變數來定義。此 URL 應與您為 Slack 團隊配置的 [傳入 Webhook](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks) 的 URL 匹配。

預設情況下，Slack 將只接收 `critical` 級別及以上的記錄；但是，您可以透過 `LOG_LEVEL` 環境變數或修改 Slack 記錄通道設定陣列中的 `level` 設定選項來調整此行為。

<a name="logging-deprecation-warnings"></a>
### 記錄棄用警告

PHP、Laravel 和其他函式庫經常通知其用戶，某些功能已被棄用並將在未來版本中移除。如果您想記錄這些棄用警告，可以透過 `LOG_DEPRECATIONS_CHANNEL` 環境變數，或在應用程式的 `config/logging.php` 設定檔中指定您偏好的 `deprecations` 記錄通道：

    'deprecations' => [
        'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null'),
        'trace' => env('LOG_DEPRECATIONS_TRACE', false),
    ],

    'channels' => [
        // ...
    ]

或者，您可以定義一個名為 `deprecations` 的記錄通道。如果存在此名稱的記錄通道，它將始終用於記錄棄用警告：

    'channels' => [
        'deprecations' => [
            'driver' => 'single',
            'path' => storage_path('logs/php-deprecation-warnings.log'),
        ],
    ],

<a name="building-log-stacks"></a>
## 建立記錄堆疊

如前所述，`stack` 驅動允許您將多個通道組合成一個單一的記錄通道，以提供便利性。為了說明如何使用記錄堆疊，讓我們看看您在生產應用程式中可能會看到的範例設定：

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['syslog', 'slack'], // [tl! add]
        'ignore_exceptions' => false,
    ],

    'syslog' => [
        'driver' => 'syslog',
        'level' => env('LOG_LEVEL', 'debug'),
        'facility' => env('LOG_SYSLOG_FACILITY', LOG_USER),
        'replace_placeholders' => true,
    ],

    'slack' => [
        'driver' => 'slack',
        'url' => env('LOG_SLACK_WEBHOOK_URL'),
        'username' => env('LOG_SLACK_USERNAME', 'Laravel Log'),
        'emoji' => env('LOG_SLACK_EMOJI', ':boom:'),
        'level' => env('LOG_LEVEL', 'critical'),
        'replace_placeholders' => true,
    ],
],
```

讓我們剖析這個設定。首先，請注意我們的 `stack` 通道透過其 `channels` 選項聚合了另外兩個通道：`syslog` 和 `slack`。因此，在記錄訊息時，這兩個通道都有機會記錄該訊息。然而，如下文所述，這些通道是否實際記錄訊息可能會由訊息的嚴重程度 / 「等級」決定。

<a name="log-levels"></a>
#### 記錄等級

請注意在上述範例中 `syslog` 和 `slack` 通道設定中存在的 `level` 設定選項。此選項決定了訊息必須達到的最低「等級」才能被通道記錄。Monolog 支援 Laravel 的記錄服務，它提供了 [RFC 5424 規範](https://tools.ietf.org/html/rfc5424)中定義的所有記錄等級。依嚴重程度降序排列，這些記錄等級為：**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info** 和 **debug**。

因此，假設我們使用 `debug` 方法記錄一條訊息：

    Log::debug('An informational message.');

根據我們的設定，`syslog` 通道將把訊息寫入系統記錄；但是，由於錯誤訊息不是 `critical` 或更高級別，因此不會發送到 Slack。然而，如果我們記錄一條 `emergency` 訊息，它將被發送到系統記錄和 Slack，因為 `emergency` 等級高於我們兩個通道的最低等級閾值：

    Log::emergency('The system is down!');

<a name="writing-log-messages"></a>
## 寫入記錄訊息

您可以使用 `Log` [Facade](/docs/{{version}}/facades) 將資訊寫入記錄。如前所述，記錄器提供了 [RFC 5424 規範](https://tools.ietf.org/html/rfc5424)中定義的八個記錄等級：**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info** 和 **debug**：

    use Illuminate\Support\Facades\Log;

    Log::emergency($message);
    Log::alert($message);
    Log::critical($message);
    Log::error($message);
    Log::warning($message);
    Log::notice($message);
    Log::info($message);
    Log::debug($message);

您可以呼叫任何這些方法來記錄對應等級的訊息。預設情況下，訊息將寫入您的 `logging` 設定檔所設定的預設記錄通道：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Models\User;
    use Illuminate\Support\Facades\Log;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         */
        public function show(string $id): View
        {
            Log::info('Showing the user profile for user: {id}', ['id' => $id]);

            return view('user.profile', [
                'user' => User::findOrFail($id)
            ]);
        }
    }

<a name="contextual-information"></a>
### 上下文資訊

一個包含上下文資料的陣列可以傳遞給記錄方法。這些上下文資料將被格式化並與記錄訊息一起顯示：

    use Illuminate\Support\Facades\Log;

    Log::info('User {id} failed to login.', ['id' => $user->id]);

有時候，您可能希望指定一些應包含在特定通道所有後續記錄項目中的上下文資訊。例如，您可能希望記錄與應用程式的每個傳入請求相關聯的請求 ID。為此，您可以呼叫 `Log` Facade 的 `withContext` 方法：

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Log;
    use Illuminate\Support\Str;
    use Symfony\Component\HttpFoundation\Response;

    class AssignRequestId
    {
        /**
         * Handle an incoming request.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            $requestId = (string) Str::uuid();

            Log::withContext([
                'request-id' => $requestId
            ]);

            $response = $next($request);

            $response->headers->set('Request-Id', $requestId);

            return $response;
        }
    }

如果您想在 _所有_ 記錄通道之間共享上下文資訊，您可以呼叫 `Log::shareContext()` 方法。此方法將向所有已建立的通道以及任何隨後建立的通道提供上下文資訊：

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Log;
    use Illuminate\Support\Str;
    use Symfony\Component\HttpFoundation\Response;

    class AssignRequestId
    {
        /**
         * Handle an incoming request.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            $requestId = (string) Str::uuid();

            Log::shareContext([
                'request-id' => $requestId
            ]);

            // ...
        }
    }

> [!NOTE]  
> 如果您需要在處理排隊的任務時共享記錄上下文，您可以利用 [任務中介層](/docs/{{version}}/queues#job-middleware)。

<a name="writing-to-specific-channels"></a>
### 寫入特定通道

有時您可能希望將訊息記錄到應用程式預設通道以外的通道。您可以使用 `Log` Facade 上的 `channel` 方法來擷取並記錄到設定檔中定義的任何通道：

    use Illuminate\Support\Facades\Log;

    Log::channel('slack')->info('Something happened!');

如果您想建立一個由多個通道組成的按需記錄堆疊，您可以使用 `stack` 方法：

    Log::stack(['single', 'slack'])->info('Something happened!');

<a name="on-demand-channels"></a>
#### 按需通道

也可以透過在執行時提供設定來建立按需通道，而無需在應用程式的 `logging` 設定檔中存在該設定。為此，您可以將設定陣列傳遞給 `Log` Facade 的 `build` 方法：

    use Illuminate\Support\Facades\Log;

    Log::build([
      'driver' => 'single',
      'path' => storage_path('logs/custom.log'),
    ])->info('Something happened!');

您可能還希望在按需記錄堆疊中包含按需通道。這可以透過在傳遞給 `stack` 方法的陣列中包含您的按需通道實例來實現：

    use Illuminate\Support\Facades\Log;

    $channel = Log::build([
      'driver' => 'single',
      'path' => storage_path('logs/custom.log'),
    ]);

    Log::stack(['slack', $channel])->info('Something happened!');

<a name="monolog-channel-customization"></a>
## Monolog 通道自訂


<a name="customizing-monolog-for-channels"></a>
### 為通道自訂 Monolog

有時候，您可能需要完全控制現有通道的 Monolog 設定方式。例如，您可能想為 Laravel 內建的 `single` 通道設定自訂的 Monolog `FormatterInterface` 實作。

首先，在通道的設定中定義一個 `tap` 陣列。`tap` 陣列應包含一個類別列表，這些類別在 Monolog 實例建立後，有機會進行自訂（或「介入」）。這些類別沒有慣用的放置位置，因此您可以在應用程式中自由建立一個目錄來包含這些類別：

    'single' => [
        'driver' => 'single',
        'tap' => [App\Logging\CustomizeFormatter::class],
        'path' => storage_path('logs/laravel.log'),
        'level' => env('LOG_LEVEL', 'debug'),
        'replace_placeholders' => true,
    ],

設定好通道的 `tap` 選項後，您就可以定義將自訂 Monolog 實例的類別。這個類別只需要一個方法：`__invoke`，它會接收一個 `Illuminate\Log\Logger` 實例。`Illuminate\Log\Logger` 實例會將所有方法呼叫代理到底層的 Monolog 實例：

    <?php

    namespace App\Logging;

    use Illuminate\Log\Logger;
    use Monolog\Formatter\LineFormatter;

    class CustomizeFormatter
    {
        /**
         * Customize the given logger instance.
         */
        public function __invoke(Logger $logger): void
        {
            foreach ($logger->getHandlers() as $handler) {
                $handler->setFormatter(new LineFormatter(
                    '[%datetime%] %channel%.%level_name%: %message% %context% %extra%'
                ));
            }
        }
    }

> [!NOTE]  
> 所有「介入」類別都由 [服務容器](/docs/{{version}}/container) 解析，因此它們所需的任何建構式依賴都會自動注入。


<a name="creating-monolog-handler-channels"></a>
### 建立 Monolog 處理器通道

Monolog 擁有多種[可用處理器](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Handler)，而 Laravel 並沒有為每個處理器都包含一個內建通道。在某些情況下，您可能希望建立一個自訂通道，它僅是一個特定 Monolog 處理器的實例，而該處理器沒有對應的 Laravel 記錄驅動程式。這些通道可以使用 `monolog` 驅動程式輕鬆建立。

當使用 `monolog` 驅動程式時，`handler` 設定選項用於指定要實例化的處理器。選擇性地，處理器所需的任何建構式參數可以使用 `with` 設定選項來指定：

    'logentries' => [
        'driver'  => 'monolog',
        'handler' => Monolog\Handler\SyslogUdpHandler::class,
        'with' => [
            'host' => 'my.logentries.internal.datahubhost.company.com',
            'port' => '10000',
        ],
    ],


<a name="monolog-formatters"></a>
#### Monolog 格式化器

當使用 `monolog` 驅動程式時，Monolog `LineFormatter` 將被用作預設的格式化器。然而，您可以使用 `formatter` 和 `formatter_with` 設定選項來自訂傳遞給處理器的格式化器類型：

    'browser' => [
        'driver' => 'monolog',
        'handler' => Monolog\Handler\BrowserConsoleHandler::class,
        'formatter' => Monolog\Formatter\HtmlFormatter::class,
        'formatter_with' => [
            'dateFormat' => 'Y-m-d',
        ],
    ],

如果您使用的 Monolog 處理器能夠提供自己的格式化器，您可以將 `formatter` 設定選項的值設為 `default`：

    'newrelic' => [
        'driver' => 'monolog',
        'handler' => Monolog\Handler\NewRelicHandler::class,
        'formatter' => 'default',
    ],


<a name="monolog-processors"></a>
#### Monolog 處理器

Monolog 也能在記錄訊息之前處理它們。您可以建立自己的處理器，或使用 [Monolog 提供的現有處理器](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Processor)。

如果您想為 `monolog` 驅動程式自訂處理器，請在通道的設定中新增一個 `processors` 設定值：

     'memory' => [
         'driver' => 'monolog',
         'handler' => Monolog\Handler\StreamHandler::class,
         'with' => [
             'stream' => 'php://stderr',
         ],
         'processors' => [
             // Simple syntax...
             Monolog\Processor\MemoryUsageProcessor::class,

             // With options...
             [
                'processor' => Monolog\Processor\PsrLogMessageProcessor::class,
                'with' => ['removeUsedContextFields' => true],
            ],
         ],
     ],


<a name="creating-custom-channels-via-factories"></a>
### 透過工廠建立自訂通道

如果您想定義一個完全自訂的通道，並對 Monolog 的實例化和設定擁有完全控制權，您可以在 `config/logging.php` 設定檔中指定 `custom` 驅動程式類型。您的設定應包含一個 `via` 選項，其中包含將被呼叫以建立 Monolog 實例的工廠類別名稱：

    'channels' => [
        'example-custom-channel' => [
            'driver' => 'custom',
            'via' => App\Logging\CreateCustomLogger::class,
        ],
    ],

設定好 `custom` 驅動程式通道後，您就可以定義將建立 Monolog 實例的類別。這個類別只需要一個 `__invoke` 方法，該方法應回傳 Monolog logger 實例。該方法將接收通道設定陣列作為其唯一參數：

    <?php

    namespace App\Logging;

    use Monolog\Logger;

    class CreateCustomLogger
    {
        /**
         * Create a custom Monolog instance.
         */
        public function __invoke(array $config): Logger
        {
            return new Logger(/* ... */);
        }
    }

<a name="tailing-log-messages-using-pail"></a>
## 使用 Pail 追蹤記錄訊息

您可能經常需要即時追蹤應用程式的記錄。例如，在偵錯問題或監控應用程式記錄中的特定錯誤類型時。

Laravel Pail 是一個套件，可讓您直接從命令列輕鬆深入查看 Laravel 應用程式的記錄檔。與標準的 `tail` 指令不同，Pail 旨在與任何記錄驅動程式搭配使用，包括 Sentry 或 Flare。此外，Pail 提供了一組實用的篩選條件，可協助您快速找到所需資訊。

<img src="https://laravel.com/img/docs/pail-example.png">


<a name="pail-installation"></a>
### 安裝

> [!WARNING]  
> Laravel Pail 需要 [PHP 8.2+](https://php.net/releases/) 和 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) 擴充功能。

若要開始，請使用 Composer 套件管理工具將 Pail 安裝到您的專案中：

```bash
composer require laravel/pail
```


<a name="pail-usage"></a>
### 用法

若要開始追蹤記錄，請執行 `pail` 指令：

```bash
php artisan pail
```

若要增加輸出的詳細程度並避免截斷 (…)，請使用 `-v` 選項：

```bash
php artisan pail -v
```

若要達到最大詳細程度並顯示例外堆疊追蹤，請使用 `-vv` 選項：

```bash
php artisan pail -vv
```

若要停止追蹤記錄，隨時按下 `Ctrl+C`。


<a name="pail-filtering-logs"></a>
### 篩選記錄


<a name="pail-filtering-logs-filter-option"></a>
#### `--filter`

您可以使用 `--filter` 選項，依記錄類型、檔案、訊息和堆疊追蹤內容來篩選記錄：

```bash
php artisan pail --filter="QueryException"
```


<a name="pail-filtering-logs-message-option"></a>
#### `--message`

若要僅依記錄訊息來篩選記錄，您可以使用 `--message` 選項：

```bash
php artisan pail --message="User created"
```


<a name="pail-filtering-logs-level-option"></a>
#### `--level`

`--level` 選項可用於依 [記錄級別](#log-levels) 篩選記錄：

```bash
php artisan pail --level=error
```


<a name="pail-filtering-logs-user-option"></a>
#### `--user`

若要僅顯示在指定使用者已認證時所寫入的記錄，您可以將使用者 ID 提供給 `--user` 選項：

```bash
php artisan pail --user=1
```