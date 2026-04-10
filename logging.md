# 日誌

- [簡介](#introduction)
- [設定](#configuration)
    - [可用頻道驅動程式](#available-channel-drivers)
    - [頻道先決條件](#channel-prerequisites)
    - [記錄廢棄警告](#logging-deprecation-warnings)
- [建立日誌堆疊](#building-log-stacks)
- [撰寫日誌訊息](#writing-log-messages)
    - [上下文資訊](#contextual-information)
    - [寫入特定頻道](#writing-to-specific-channels)
- [Monolog 頻道自訂](#monolog-channel-customization)
    - [自訂頻道的 Monolog](#customizing-monolog-for-channels)
    - [建立 Monolog 處理器頻道](#creating-monolog-handler-channels)
    - [透過工廠建立自訂頻道](#creating-custom-channels-via-factories)
- [使用 Pail 追蹤日誌訊息](#tailing-log-messages-using-pail)
    - [安裝](#pail-installation)
    - [使用方式](#pail-usage)
    - [過濾日誌](#pail-filtering-logs)

<a name="introduction"></a>
## 簡介

為幫助您更了解應用程式內部發生的情況，Laravel 提供了強大的日誌服務，讓您能將訊息記錄到檔案、系統錯誤日誌，甚至傳送到 Slack 以通知您的整個團隊。

Laravel 的日誌功能基於「頻道」。每個頻道都代表一種特定的日誌資訊寫入方式。例如，`single` 頻道將日誌檔案寫入單一的日誌檔，而 `slack` 頻道則會將日誌訊息傳送到 Slack。日誌訊息可根據其嚴重程度寫入多個頻道。

底層，Laravel 利用了 [Monolog](https://github.com/Seldaek/monolog) 函式庫，該函式庫支援多種強大的日誌處理器。Laravel 讓配置這些處理器變得輕而易舉，允許您混合搭配它們以自訂應用程式的日誌處理。

<a name="configuration"></a>
## 設定

所有控制應用程式日誌行為的設定選項都儲存在 `config/logging.php` 設定檔中。此檔案允許您配置應用程式的日誌頻道，因此請務必檢視每個可用頻道及其選項。我們將在下方檢視一些常見選項。

預設情況下，Laravel 在記錄訊息時將使用 `stack` 頻道。`stack` 頻道用於將多個日誌頻道聚合到一個單一頻道中。有關建立堆疊的更多資訊，請查看 [下方文件](#building-log-stacks)。

<a name="available-channel-drivers"></a>
### 可用頻道驅動程式

每個日誌頻道都由一個「驅動程式」驅動。驅動程式決定了日誌訊息實際記錄的方式和位置。每個 Laravel 應用程式都提供以下日誌頻道驅動程式。其中大多數驅動程式的條目已存在於您應用程式的 `config/logging.php` 設定檔中，因此請務必檢視此檔案以熟悉其內容：

<div class="overflow-auto">

| 名稱         | 描述                                                          |
| ------------ | -------------------------------------------------------------------- |
| `custom`     | 一個呼叫指定工廠來建立頻道的驅動程式。         |
| `daily`      | 一個基於 `RotatingFileHandler` 的 Monolog 驅動程式，每日輪替。    |
| `errorlog`   | 一個基於 `ErrorLogHandler` 的 Monolog 驅動程式。                           |
| `monolog`    | 一個 Monolog 工廠驅動程式，可以使用任何支援的 Monolog 處理器。 |
| `papertrail` | 一個基於 `SyslogUdpHandler` 的 Monolog 驅動程式。                           |
| `single`     | 一個單一檔案或路徑基礎的日誌記錄器頻道 (`StreamHandler`)。        |
| `slack`      | 一個基於 `SlackWebhookHandler` 的 Monolog 驅動程式。                        |
| `stack`      | 一個用於協助建立「多頻道」頻道的包裝器。           |
| `syslog`     | 一個基於 `SyslogHandler` 的 Monolog 驅動程式。                              |

</div>

> [!NOTE]
> 查看 [進階頻道自訂](#monolog-channel-customization) 文件，了解更多關於 `monolog` 和 `custom` 驅動程式的資訊。

<a name="configuring-the-channel-name"></a>
#### 設定頻道名稱

預設情況下，Monolog 會以與目前環境（例如 `production` 或 `local`）匹配的「頻道名稱」實例化。要更改此值，您可以為您的頻道設定添加一個 `name` 選項：

```php
'stack' => [
    'driver' => 'stack',
    'name' => 'channel-name',
    'channels' => ['single', 'slack'],
],
```

<a name="channel-prerequisites"></a>
### 頻道先決條件

<a name="configuring-the-single-and-daily-channels"></a>
#### 設定 Single 與 Daily 頻道

`single` 和 `daily` 頻道有三個選用的設定選項：`bubble`、`permission` 和 `locking`。

<div class="overflow-auto">

| 名稱         | 描述                                                                   | 預設值 |
| ------------ | ----------------------------------------------------------------------------- | ------- |
| `bubble`     | 指示訊息在處理後是否應向上傳遞至其他頻道。 | `true`  |
| `locking`    | 在寫入日誌檔前嘗試鎖定它。                            | `false` |
| `permission` | 日誌檔的權限。                                                   | `0644`  |

</div>

此外，`daily` 頻道的保留策略可以透過 `LOG_DAILY_DAYS` 環境變數或設定 `days` 設定選項來配置。

<div class="overflow-auto">

| 名稱   | 描述                                                 | 預設值 |
| ------ | ----------------------------------------------------------- | ------- |
| `days` | 每日日誌檔應保留的天數。 | `14`    |

</div>

<a name="configuring-the-papertrail-channel"></a>
#### 設定 Papertrail 頻道

`papertrail` 頻道需要 `host` 和 `port` 設定選項。這些可以透過 `PAPERTRAIL_URL` 和 `PAPERTRAIL_PORT` 環境變數來定義。您可以從 [Papertrail](https://help.papertrailapp.com/kb/configuration/configuring-centralized-logging-from-php-apps/#send-events-from-php-app) 取得這些值。

<a name="configuring-the-slack-channel"></a>
#### 設定 Slack 頻道

`slack` 頻道需要一個 `url` 設定選項。此值可以透過 `LOG_SLACK_WEBHOOK_URL` 環境變數來定義。此 URL 應與您為 Slack 團隊配置的 [傳入的 Webhook](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks) 的 URL 相匹配。

預設情況下，Slack 將僅接收 `critical` 級別及以上的日誌；但是，您可以透過使用 `LOG_LEVEL` 環境變數或修改 Slack 日誌頻道的設定陣列中的 `level` 設定選項來調整此行為。

<a name="logging-deprecation-warnings"></a>
### 記錄廢棄警告

PHP、Laravel 和其他函式庫經常通知其使用者，某些功能已被廢棄並將在未來版本中移除。如果您想記錄這些廢棄警告，您可以透過 `LOG_DEPRECATIONS_CHANNEL` 環境變數，或在應用程式的 `config/logging.php` 設定檔中指定您偏好的 `deprecations` 日誌頻道：

```php
'deprecations' => [
    'channel' => env('LOG_DEPRECATIONS_CHANNEL', 'null'),
    'trace' => env('LOG_DEPRECATIONS_TRACE', false),
],

'channels' => [
    // ...
]
```

或者，您可以定義一個名為 `deprecations` 的日誌頻道。如果存在此名稱的日誌頻道，它將始終用於記錄廢棄項：

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

如前所述，`stack` 驅動程式可讓您將多個頻道組合成一個單一的日誌頻道，以方便使用。為了說明如何使用日誌堆疊，讓我們看看一個您在生產環境應用程式中可能會看到的範例設定：

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

讓我們分析這個設定。首先，請注意我們的 `stack` 頻道透過其 `channels` 選項，彙整了另外兩個頻道：`syslog` 和 `slack`。因此，當記錄訊息時，這兩個頻道都有機會記錄該訊息。然而，如下所述，這些頻道是否實際記錄訊息，可能由訊息的嚴重程度 / 「層級」決定。

<a name="log-levels"></a>
#### 日誌層級

請注意上例中 `syslog` 和 `slack` 頻道設定上的 `level` 設定選項。此選項決定了訊息必須達到哪個最低「層級」才能被該頻道記錄。Monolog 支援 Laravel 的日誌服務，提供了 [RFC 5424 規範](https://tools.ietf.org/html/rfc5424)中定義的所有日誌層級。依嚴重程度遞減排列，這些日誌層級為：**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info** 和 **debug**。

因此，假設我們使用 `debug` 方法記錄一則訊息：

```php
Log::debug('An informational message.');
```

根據我們的設定，`syslog` 頻道會將訊息寫入系統日誌；但是，由於該錯誤訊息未達 `critical` 或更高層級，因此不會傳送至 Slack。然而，如果我們記錄一則 `emergency` 訊息，它將同時傳送至系統日誌和 Slack，因為 `emergency` 層級高於我們為這兩個頻道設定的最低層級閾值：

```php
Log::emergency('The system is down!');
```

<a name="writing-log-messages"></a>
## 撰寫日誌訊息

您可以使用 `Log` [Facade](/docs/{{version}}/facades) 將資訊寫入日誌。如前所述，日誌器提供了 [RFC 5424 規範](https://tools.ietf.org/html/rfc5424)中定義的八個日誌層級：**emergency**、**alert**、**critical**、**error**、**warning**、**notice**、**info** 和 **debug**：

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

您可以呼叫其中任何一個方法來記錄對應層級的訊息。預設情況下，訊息將寫入您的 `logging` 設定檔所設定的預設日誌頻道：

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
### 上下文資訊

您可以將上下文資料陣列傳遞給日誌方法。此上下文資料將與日誌訊息一起格式化並顯示：

```php
use Illuminate\Support\Facades\Log;

Log::info('User {id} failed to login.', ['id' => $user->id]);
```

有時，您可能希望指定一些應包含在特定頻道中所有後續日誌項目中的上下文資訊。例如，您可能希望記錄與您應用程式中每個傳入請求相關聯的請求 ID。為此，您可以呼叫 `Log` Facade 的 `withContext` 方法：

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

如果您想在 _所有_ 日誌頻道中分享上下文資訊，您可以調用 `Log::shareContext()` 方法。此方法會將上下文資訊提供給所有已建立的頻道以及任何隨後建立的頻道：

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
> 如果您需要在處理佇列工作時分享日誌上下文，您可以利用 [job middleware](/docs/{{version}}/queues#job-middleware)。

<a name="writing-to-specific-channels"></a>
### 寫入特定頻道

有時您可能希望將訊息記錄到應用程式預設頻道以外的頻道。您可以利用 `Log` Facade 上的 `channel` 方法來取得並記錄到設定檔中定義的任何頻道：

```php
use Illuminate\Support\Facades\Log;

Log::channel('slack')->info('Something happened!');
```

如果您想建立一個由多個頻道組成的隨選日誌堆疊，可以使用 `stack` 方法：

```php
Log::stack(['single', 'slack'])->info('Something happened!');
```

<a name="on-demand-channels"></a>
#### 隨選頻道

也可以在執行時提供設定來建立隨選頻道，而無需將該設定納入您應用程式的 `logging` 設定檔中。為此，您可以將設定陣列傳遞給 `Log` Facade 的 `build` 方法：

```php
use Illuminate\Support\Facades\Log;

Log::build([
  'driver' => 'single',
  'path' => storage_path('logs/custom.log'),
])->info('Something happened!');
```

您可能還希望將隨選頻道納入隨選日誌堆疊中。這可以透過將您的隨選頻道實例包含在傳遞給 `stack` 方法的陣列中來實現：

```php
use Illuminate\Support\Facades\Log;

$channel = Log::build([
  'driver' => 'single',
  'path' => storage_path('logs/custom.log'),
]);

Log::stack(['slack', $channel])->info('Something happened!');
```

<a name="monolog-channel-customization"></a>
## Monolog 頻道自訂


<a name="customizing-monolog-for-channels"></a>
### 自訂頻道的 Monolog

有時候您可能需要完全控制現有頻道的 Monolog 設定方式。例如，您可能希望為 Laravel 內建的 `single` 頻道設定一個自訂的 Monolog `FormatterInterface` 實作。

首先，在頻道的設定中定義一個 `tap` 陣列。`tap` 陣列應包含一個類別列表，這些類別在 Monolog 實例建立後，有機會自訂（或「輕觸 (tap)」）該實例。這些類別沒有慣例上的放置位置，因此您可以自由地在應用程式中建立一個目錄來包含這些類別：

```php
'single' => [
    'driver' => 'single',
    'tap' => [App\Logging\CustomizeFormatter::class],
    'path' => storage_path('logs/laravel.log'),
    'level' => env('LOG_LEVEL', 'debug'),
    'replace_placeholders' => true,
],
```

一旦您在頻道上設定了 `tap` 選項，您就可以定義將自訂 Monolog 實例的類別。此類別只需要一個方法：`__invoke`，它會接收一個 `Illuminate\Log\Logger` 實例。`Illuminate\Log\Logger` 實例會將所有方法呼叫代理到底層的 Monolog 實例：

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
> 您的所有「輕觸 (tap)」類別都將由 [服務容器](/docs/{{version}}/container) 解析，因此它們所需的任何建構子依賴項都會自動注入。


<a name="creating-monolog-handler-channels"></a>
### 建立 Monolog 處理器頻道

Monolog 擁有多種[可用的處理器](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Handler)，而 Laravel 並未為每個處理器都包含內建頻道。在某些情況下，您可能希望建立一個自訂頻道，它僅僅是特定 Monolog 處理器的實例，而該處理器沒有對應的 Laravel 日誌驅動程式。這些頻道可以使用 `monolog` 驅動程式輕鬆建立。

使用 `monolog` 驅動程式時，`handler` 設定選項用於指定將要實例化的處理器。選擇性地，處理器所需的任何建構子參數可以使用 `handler_with` 設定選項來指定：

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
#### Monolog 格式器

使用 `monolog` 驅動程式時，Monolog `LineFormatter` 將用作預設格式器。然而，您可以使用 `formatter` 和 `formatter_with` 設定選項來自訂傳遞給處理器的格式器類型：

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

如果您使用的 Monolog 處理器能夠提供自己的格式器，您可以將 `formatter` 設定選項的值設為 `default`：

```php
'newrelic' => [
    'driver' => 'monolog',
    'handler' => Monolog\Handler\NewRelicHandler::class,
    'formatter' => 'default',
],
```


<a name="monolog-processors"></a>
#### Monolog 處理器

Monolog 也可以在記錄訊息之前處理這些訊息。您可以建立自己的處理器，或使用 [Monolog 提供的現有處理器](https://github.com/Seldaek/monolog/tree/main/src/Monolog/Processor)。

如果您想為 `monolog` 驅動程式自訂處理器，請在您的頻道設定中加入一個 `processors` 設定值：

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

如果您想定義一個完全自訂的頻道，並能完全控制 Monolog 的實例化和設定，您可以在 `config/logging.php` 設定檔中指定一個 `custom` 驅動程式類型。您的設定應包含一個 `via` 選項，其中包含將被呼叫以建立 Monolog 實例的工廠類別名稱：

```php
'channels' => [
    'example-custom-channel' => [
        'driver' => 'custom',
        'via' => App\Logging\CreateCustomLogger::class,
    ],
],
```

一旦您設定了 `custom` 驅動程式頻道，您就可以定義將建立您的 Monolog 實例的類別。此類別只需要一個 `__invoke` 方法，該方法應返回 Monolog logger 實例。該方法將接收頻道設定陣列作為其唯一參數：

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
## 使用 Pail 追蹤日誌訊息

通常，您可能需要即時追蹤應用程式的日誌。例如，在偵錯問題或監控應用程式日誌以查找特定類型的錯誤時。

Laravel Pail 是一個套件，可讓您直接從命令列輕鬆深入 Laravel 應用程式的日誌檔案。與標準的 `tail` 命令不同，Pail 旨在與任何日誌驅動程式配合使用，包括 Sentry 或 Flare。此外，Pail 提供了一組有用的過濾器，可幫助您快速找到所需的內容。

<img src="https://laravel.com/img/docs/pail-example.png" alt="Pail 範例">


<a name="pail-installation"></a>
### 安裝

> [!WARNING]
> Laravel Pail 需要 [PCNTL](https://www.php.net/manual/en/book.pcntl.php) PHP 擴充功能。

首先，使用 Composer 套件管理器將 Pail 安裝到您的專案中：

```shell
composer require --dev laravel/pail
```


<a name="pail-usage"></a>
### 使用方式

要開始追蹤日誌，請執行 `pail` 命令：

```shell
php artisan pail
```

若要增加輸出的詳細程度並避免截斷（…），請使用 `-v` 選項：

```shell
php artisan pail -v
```

若要達到最大詳細程度並顯示例外堆疊追蹤，請使用 `-vv` 選項：

```shell
php artisan pail -vv
```

若要停止追蹤日誌，請隨時按下 `Ctrl+C`。


<a name="pail-filtering-logs"></a>
### 過濾日誌


<a name="pail-filtering-logs-filter-option"></a>
#### `--filter`

您可以使用 `--filter` 選項按日誌類型、檔案、訊息和堆疊追蹤內容來過濾日誌：

```shell
php artisan pail --filter="QueryException"
```


<a name="pail-filtering-logs-message-option"></a>
#### `--message`

若要僅按訊息過濾日誌，您可以使用 `--message` 選項：

```shell
php artisan pail --message="User created"
```


<a name="pail-filtering-logs-level-option"></a>
#### `--level`

`--level` 選項可用於按日誌的[日誌層級](#log-levels)過濾日誌：

```shell
php artisan pail --level=error
```


<a name="pail-filtering-logs-user-option"></a>
#### `--user`

若要僅顯示在特定使用者已驗證時寫入的日誌，您可以將該使用者的 ID 提供給 `--user` 選項：

```shell
php artisan pail --user=1
```