# 日誌 (Logging)

- [簡介](#introduction)
- [設定](#configuration)
    - [可用的頻道驅動程式](#available-channel-drivers)
    - [頻道前置需求](#channel-prerequisites)
    - [紀錄棄用警告](#logging-deprecation-warnings)
- [建立日誌堆疊](#building-log-stacks)
- [撰寫日誌訊息](#writing-log-messages)
    - [情境資訊](#contextual-information)
    - [寫入特定頻道](#writing-to-specific-channels)
- [Monolog 頻道客製化](#monolog-channel-customization)
    - [為頻道客製化 Monolog](#customizing-monolog-for-channels)
    - [建立 Monolog Handler 頻道](#creating-monolog-handler-channels)
    - [透過工廠建立自訂頻道](#creating-custom-channels-via-factories)
- [使用 Pail 即時追蹤日誌訊息](#tailing-log-messages-using-pail)
    - [安裝](#pail-installation)
    - [使用方式](#pail-usage)
    - [過濾日誌](#pail-filtering-logs)

<a name="introduction"></a>
## 簡介

為了幫助您更深入了解應用程式內部發生的事情，Laravel 提供了強大的日誌服務，允許您將訊息記錄到檔案、系統錯誤日誌、甚至是 Slack，以通知您的整個團隊。

Laravel 日誌是基於「頻道 (Channels)」運作的。每個頻道代表一種特定的日誌資訊寫入方式。例如，`single` 頻道會將日誌訊息寫入單一檔案，而 `slack` 頻道則會將日誌訊息傳送到 Slack。日誌訊息也可以根據其嚴重程度寫入到多個頻道中。

在底層，Laravel 利用了 [Monolog](https://github.com/Seldaek/monolog) 函式庫，它提供了對各種強大日誌處理常式 (Handlers) 的支援。Laravel 讓設定這些處理常式變得非常簡單，允許您自由混合搭配它們，以自訂應用程式的日誌處理方式。


<a name="configuration"></a>
## 設定

所有控制應用程式日誌行為的設定選項都基於 `config/logging.php` 設定檔中。此檔案允許您設定應用程式的日誌頻道，因此請務必審閱每個可用的頻道及其選項。我們將在下方審閱一些常見的選項。

預設情況下，Laravel 在記錄訊息時會使用 `stack` 頻道。`stack` 頻道用於將多個日誌頻道整合為單一頻道。關於建立堆疊的更多資訊，請參考[下方文件](#building-log-stacks)。


<a name="available-channel-drivers"></a>
### 可用的頻道驅動程式

每個日誌頻道都由一個「驅動程式」提供動力。驅動程式決定了日誌訊息實際上如何以及在何處被記錄。以下日誌頻道驅動程式在每個 Laravel 應用程式中皆可使用。您的應用程式 `config/logging.php` 設定檔中已經存在大多數這些驅動程式的條目，因此請務必審閱此檔案以熟悉其內容：

<div class="overflow-auto">

| 名稱         | 說明                                                                 |
| ------------ | -------------------------------------------------------------------- |
| `custom`     | 呼叫指定工廠來建立頻道的驅動程式。                                   |
| `daily`      | 基於 `RotatingFileHandler` 的 Monolog 驅動程式，按天輪替。           |
| `monthly`    | 基於 `RotatingFileHandler` 的 Monolog 驅動程式，按月輪替。           |
| `errorlog`   | 基於 `ErrorLogHandler` 的 Monolog 驅動程式。                         |
| `monolog`    | 可以使用任何受支援的 Monolog handler 的 Monolog 工廠驅動程式。       |
| `papertrail` | 基於 `SyslogUdpHandler` 的 Monolog 驅動程式。                        |
| `single`     | 基於單一檔案或路徑的記錄器頻道 (`StreamHandler`)。                   |
| `slack`      | 基於 `SlackWebhookHandler` 的 Monolog 驅動程式。                     |
| `stack`      | 用於方便建立「多頻道」頻道的封裝器。                                 |
| `syslog`     | 基於 `SyslogHandler` 的 Monolog 驅動程式。                           |

</div>

> [!NOTE]
> 請參考[進階頻道客製化](#monolog-channel-customization)的文件，以深入了解 `monolog` 與 `custom` 驅動程式。


<a name="configuring-the-channel-name"></a>
#### 設定頻道名稱

預設情況下，Monolog 實例化時的「頻道名稱」會與當前環境相符，例如 `production` 或 `local`。若要更改此數值，您可以在頻道的設定中新增 `name` 選項：

```php
'stack' => [
    'driver' => 'stack',
    'name' => 'channel-name',
    'channels' => ['single', 'slack'],
],
```


<a name="channel-prerequisites"></a>
### 頻道前置需求


<a name="configuring-the-single-daily-and-monthly-channels"></a>
#### 設定 Single、Daily 與 Monthly 頻道

`single`、`daily` 與 `monthly` 頻道有三個可選的設定選項：`bubble`、`permission` 與 `locking`。

<div class="overflow-auto">

| 名稱         | 說明                                                                         | 預設值 |
| ------------ | ---------------------------------------------------------------------------- | ------ |
| `bubble`     | 表示訊息在被處理後是否應冒泡 (Bubble up) 傳遞至其他頻道。                    | `true` |
| `locking`    | 在寫入日誌檔案之前嘗試鎖定該檔案。                                           | `false`|
| `permission` | 日誌檔案的權限。                                                             | `0644` |

</div>

此外，`daily` 與 `monthly` 頻道的保留政策可以透過 `max_files` 設定選項來設定。環境變數 `LOG_DAILY_DAYS` 也可以用來設定 `daily` 頻道的保留天數。


<a name="configuring-the-papertrail-channel"></a>
#### 設定 Papertrail 頻道

`papertrail` 頻道需要 `host` 與 `port` 設定選項。這些可以透過 `PAPERTRAIL_URL` 與 `PAPERTRAIL_PORT` 環境變數來定義。您可以從 [Papertrail](https://help.papertrailapp.com/kb/configuration/configuring-centralized-logging-from-php-apps/#send-events-from-php-app) 取得這些值。


<a name="configuring-the-slack-channel"></a>
#### 設定 Slack 頻道

`slack` 頻道需要 `url` 設定選項。此值可以透過 `LOG_SLACK_WEBHOOK_URL` 環境變數來定義。此 URL 應符合您為 Slack 團隊設定的 [Incoming Webhook](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks) URL。

預設情況下，Slack 只會接收 `critical` 層級及以上的日誌；不過，您可以透過調整 `LOG_LEVEL` 環境變數，或修改 Slack 日誌頻道設定陣列中的 `level` 設定選項來調整此限制。


<a name="logging-deprecation-warnings"></a>
### 紀錄棄用警告

PHP、Laravel 與其他函式庫經常會通知使用者某些功能已被棄用，並將在未來的版本中刪除。如果您想要記錄這些棄用警告，您可以使用 `LOG_DEPRECATIONS_CHANNEL` 環境變數或在應用程式的 `config/logging.php` 設定檔中指定偏好的 `deprecations` 日誌頻道：

```php
'deprecations' => [
    'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null'),
    'trace' => env('LOG_DEPRECATIONS_TRACE', false),
],

'channels' => [
    // ...
]
```

或者，您可以定義一個名為 `deprecations` 的日誌頻道。如果存在此名稱的日誌頻道，它將始終用於記錄棄用警告：

```php
'channels' => [
    'deprecations' => [
        'driver' => 'single',
        'path' => storage_path('logs/php-deprecation-warnings.log'),
    ],
],
```

<a name="building-log-stacks"></a>
## 建立日誌堆疊

如前所述，`stack` 驅動程式允許你為了方便而將多個頻道組合成一個單一的日誌頻道。為了說明如何使用日誌堆疊，讓我們來看看一個在正式環境應用程式中可能會看到的設定範例：

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

讓我們剖析這個設定。首先，注意到我們的 `stack` 頻道透過其 `channels` 選項聚合了另外兩個頻道：`syslog` 與 `slack`。因此，在紀錄訊息時，這兩個頻道都有機會紀錄該訊息。然而，如下所述，這些頻道是否真的會紀錄該訊息，可能取決於訊息的嚴重程度 /「等級」。

<a name="log-levels"></a>
#### 日誌等級

請注意上述範例中 `syslog` 與 `slack` 頻道設定裡出現的 `level` 設定選項。此選項決定了訊息必須達到的最低「等級」，才會被該頻道紀錄。為 Laravel 日誌服務提供支援的 Monolog 提供了 [RFC 5424 規範](https://tools.ietf.org/html/rfc5424)中定義的所有日誌等級。按嚴重程度降序排列，這些日誌等級依次為：**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info** 及 **debug**。

因此，想像一下我們使用 `debug` 方法紀錄一條訊息：

```php
Log::debug('An informational message.');
```

根據我們的設定，`syslog` 頻道會將訊息寫入系統日誌；但是，由於該錯誤訊息並未達到 `critical` 或以上的等級，因此不會發送到 Slack。不過，如果我們紀錄一條 `emergency` 訊息，它將會同時發送到系統日誌與 Slack，因為 `emergency` 等級高於這兩個頻道的最低等級門檻：

```php
Log::emergency('The system is down!');
```

<a name="writing-log-messages"></a>
## 撰寫日誌訊息

你可以使用 `Log` [Facade](/docs/{{version}}/facades) 將資訊寫入日誌。如前所述，日誌記錄器提供了 [RFC 5424 規範](https://tools.ietf.org/html/rfc5424)中定義的八個日誌等級：**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info** 與 **debug**：

```php
use Illuminate\Support\Facades\Log;

Log::emergency($message);
Log::alert($message);
Log::critical($message);
Log::error($message);
Log::warning($message);
Log::notice($message);
Log::info($message);
Log::debug($message);
```

你可以呼叫這些方法中的任何一個來紀錄對應等級的訊息。預設情況下，訊息會寫入你的 `logging` 設定檔中所設定的預設日誌頻道：

```php
<?php

namespace App\Http\Controllers;

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
```

<a name="contextual-information"></a>
### 情境資訊

可以將情境資料陣列傳遞給日誌方法。這些情境資料將被格式化並與日誌訊息一起顯示：

```php
use Illuminate\Support\Facades\Log;

Log::info('User {id} failed to login.', ['id' => $user->id]);
```

有時候，你可能希望指定一些情境資訊，並將其包含在特定頻道中所有後續的日誌項目中。例如，你可能希望紀錄與傳入應用程式的每個請求相關聯的請求 ID。為實現此目的，你可以呼叫 `Log` Facade 的 `withContext` 方法：

```php
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
```

如果你想跨 _所有_ 日誌頻道共享情境資訊，你可以呼叫 `Log::shareContext()` 方法。此方法將提供情境資訊給所有已建立的頻道以及隨後建立的任何頻道：

```php
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
```

> [!NOTE]
> 如果你需要於處理佇列任務 (Queued jobs) 時共享日誌情境，可以使用 [Job 中介層](/docs/{{version}}/queues#job-middleware)。

<a name="writing-to-specific-channels"></a>
### 寫入特定頻道

有時你可能希望將訊息紀錄到應用程式預設頻道之外的其他頻道。你可以使用 `Log` Facade 上的 `channel` 方法來取得設定檔中定義的任何頻道並對其寫入日誌：

```php
use Illuminate\Support\Facades\Log;

Log::channel('slack')->info('Something happened!');
```

如果你想建立一個由多個頻道組成的隨需 (On-demand) 日誌堆疊，可以使用 `stack` 方法：

```php
Log::stack(['single', 'slack'])->info('Something happened!');
```

<a name="on-demand-channels"></a>
#### 隨需頻道

你也可以透過在執行期提供設定來建立隨需頻道，而不需要將該設定寫在應用程式的 `logging` 設定檔中。為實現此目的，你可以將設定陣列傳遞給 `Log` Facade 的 `build` 方法：

```php
use Illuminate\Support\Facades\Log;

Log::build([
  'driver' => 'single',
  'path' => storage_path('logs/custom.log'),
])->info('Something happened!');
```

你可能還希望將隨需頻道包含在隨需日誌堆疊中。這可以透過將隨需頻道實例包含在傳遞給 `stack` 方法的陣列中來實現：

```php
use Illuminate\Support\Facades\Log;

$channel = Log::build([
  'driver' => 'single',
  'path' => storage_path('logs/custom.log'),
]);

Log::stack(['slack', $channel])->info('Something happened!');
```

<a name="monolog-channel-customization"></a>
## Monolog 頻道客製化


<a name="customizing-monolog-for-channels"></a>
### 為頻道客製化 Monolog

有時候，你可能需要完全控制現有頻道的 Monolog 設定方式。例如，你可能想為 Laravel 內建的 `single` 頻道設定自訂的 Monolog `FormatterInterface` 實作。

首先，在頻道的設定中定義一個 `tap` 陣列。`tap` 陣列應該包含一個類別清單，這些類別可以在 Monolog 實例建立後，獲得客製化（或介入「tap」）該實例的機會。Laravel 沒有規定這些類別應該放在哪裡，因此你可以自由地在應用程式中建立目錄來放置這些類別：

```php
'single' => [
    'driver' => 'single',
    'tap' => [App\Logging\CustomizeFormatter::class],
    'path' => storage_path('logs/laravel.log'),
    'level' => env('LOG_LEVEL', 'debug'),
    'replace_placeholders' => true,
],
```

當你設定好頻道的 `tap` 選項後，就可以準備定義用來客製化 Monolog 實例的類別。這個類別只需要一個 `__invoke` 方法，該方法會接收一個 `Illuminate\Log\Logger` 實例。`Illuminate\Log\Logger` 實例會將所有方法呼叫代理（proxy）至底層的 Monolog 實例：

```php
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
```

> [!NOTE]
> 所有你的「tap」類別都是由[服務容器](/docs/{{version}}/container)所解析，因此它們需要的任何建構子依賴都會自動被注入。


<a name="creating-monolog-handler-channels"></a>
### 建立 Monolog Handler 頻道

Monolog 擁有各種[可用的 Handler](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Handler)，而 Laravel 並未為每一個 Handler 都提供內建頻道。在某些情況下，你可能希望建立一個自訂頻道，該頻道只是某個特定 Monolog Handler 的實例，但沒有相對應的 Laravel 日誌驅動程式。這些頻道可以使用 `monolog` 驅動程式輕鬆建立。

使用 `monolog` 驅動程式時，`handler` 設定選項用於指定將被實例化的 Handler。此外，若 Handler 需要任何建構子參數，可以使用 `handler_with` 設定選項來指定：

```php
'logentries' => [
    'driver'  => 'monolog',
    'handler' => Monolog\Handler\SyslogUdpHandler::class,
    'handler_with' => [
        'host' => 'my.logentries.internal.datahubhost.company.com',
        'port' => '10000',
    ],
],
```


<a name="monolog-formatters"></a>
#### Monolog 格式化器

使用 `monolog` 驅動程式時，預設會使用 Monolog 的 `LineFormatter` 作為格式化器。不過，你可以使用 `formatter` 與 `formatter_with` 設定選項來客製化傳遞給 Handler 的格式化器類型：

```php
'browser' => [
    'driver' => 'monolog',
    'handler' => Monolog\Handler\BrowserConsoleHandler::class,
    'formatter' => Monolog\Formatter\HtmlFormatter::class,
    'formatter_with' => [
        'dateFormat' => 'Y-m-d',
    ],
],
```

如果你使用的 Monolog Handler 本身就能夠提供自己的格式化器，你可以將 `formatter` 設定選項的值設為 `default`：

```php
'newrelic' => [
    'driver' => 'monolog',
    'handler' => Monolog\Handler\NewRelicHandler::class,
    'formatter' => 'default',
],
```


<a name="monolog-processors"></a>
#### Monolog 處理器

Monolog 也可以在記錄訊息之前對其進行處理。你可以建立自己的處理器（Processor），或是使用 [Monolog 提供的現有處理器](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Processor)。

如果你想為 `monolog` 驅動程式客製化處理器，請在頻道的設定中加入 `processors` 設定值：

```php
'memory' => [
    'driver' => 'monolog',
    'handler' => Monolog\Handler\StreamHandler::class,
    'handler_with' => [
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
```


<a name="creating-custom-channels-via-factories"></a>
### 透過工廠建立自訂頻道

如果你想定義一個全新的自訂頻道，並完全掌控 Monolog 的實例化與設定，可以在 `config/logging.php` 設定檔中指定 `custom` 驅動程式類型。你的設定應該包含一個 `via` 選項，其中包含用於建立 Monolog 實例的工廠類別名稱：

```php
'channels' => [
    'example-custom-channel' => [
        'driver' => 'custom',
        'via' => App\Logging\CreateCustomLogger::class,
    ],
],
```

當你設定好 `custom` 驅動程式頻道後，就可以準備定義用來建立 Monolog 實例的類別。這個類別只需要一個 `__invoke` 方法，該方法應該回傳 Monolog 日誌器實例。此方法會接收頻道的設定陣列作為其唯一的引數：

```php
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
```

<a name="tailing-log-messages-using-pail"></a>
## 使用 Pail 即時追蹤日誌訊息

通常，您可能需要即時追蹤應用程式的日誌。例如，在排查問題或監控應用程式日誌中的特定類型錯誤時。

Laravel Pail 是一個能讓您直接從命令列輕鬆深入檢視 Laravel 應用程式日誌檔案的套件。與標準的 `tail` 指令不同，Pail 旨在搭配任何日誌驅動程式運作，包括 [Laravel Nightwatch](https://nightwatch.laravel.com)、Sentry 或 Flare。此外，Pail 還提供了一組實用的過濾器，幫助您快速找到正在尋找的內容。

<img src="https://laravel.com/img/docs/pail-example.png">


<a name="pail-installation"></a>
### 安裝

> [!WARNING]
> Laravel Pail 需要 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充套件。

若要開始使用，請使用 Composer 套件管理員將 Pail 安裝至您的專案中：

```shell
composer require --dev laravel/pail
```


<a name="pail-usage"></a>
### 使用方式

若要開始即時追蹤日誌，請執行 `pail` 指令：

```shell
php artisan pail
```

若要增加輸出的詳細程度並避免內容被截斷 (…)，可以使用 `-v` 選項：

```shell
php artisan pail -v
```

若要取得最高詳細度並顯示例外狀況的堆疊追蹤，可以使用 `-vv` 選項：

```shell
php artisan pail -vv
```

若要停止追蹤日誌，可以隨時按下 `Ctrl+C`。


<a name="pail-filtering-logs"></a>
### 過濾日誌


<a name="pail-filtering-logs-filter-option"></a>
#### `--filter`

您可以使用 `--filter` 選項根據日誌的類型、檔案、訊息和堆疊追蹤內容來過濾日誌：

```shell
php artisan pail --filter="QueryException"
```


<a name="pail-filtering-logs-message-option"></a>
#### `--message`

若要僅根據日誌訊息過濾日誌，可以使用 `--message` 選項：

```shell
php artisan pail --message="User created"
```


<a name="pail-filtering-logs-level-option"></a>
#### `--level`

`--level` 選項可用於根據[日誌等級](#log-levels)來過濾日誌：

```shell
php artisan pail --level=error
```


<a name="pail-filtering-logs-user-option"></a>
#### `--user`

若要僅顯示在特定使用者通過認證期間寫入的日誌，可以將該使用者的 ID 提供給 `--user` 選項：

```shell
php artisan pail --user=1
```