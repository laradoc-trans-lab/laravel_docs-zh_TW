# Laravel Horizon

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [儀表板授權](#dashboard-authorization)
    - [最大任務嘗試次數](#max-job-attempts)
    - [任務超時](#job-timeout)
    - [任務重試延遲](#job-backoff)
    - [靜音任務](#silenced-jobs)
- [平衡策略](#balancing-strategies)
    - [自動平衡](#auto-balancing)
    - [簡單平衡](#simple-balancing)
    - [不進行平衡](#no-balancing)
- [升級 Horizon](#upgrading-horizon)
- [執行 Horizon](#running-horizon)
    - [部署 Horizon](#deploying-horizon)
- [標籤](#tags)
- [通知](#notifications)
- [指標](#metrics)
- [刪除失敗的任務](#deleting-failed-jobs)
- [清除佇列中的任務](#clearing-jobs-from-queues)

<a name="introduction"></a>
## 簡介

> [!NOTE]
> 在深入研究 Laravel Horizon 之前，您應該先熟悉 Laravel 的基礎 [佇列服務](/docs/{{version}}/queues)。Horizon 為 Laravel 的佇列增添了額外功能，如果您還不熟悉 Laravel 提供的基本佇列功能，這些功能可能會讓您感到困惑。

[Laravel Horizon](https://github.com/laravel/horizon) 為您由 Laravel 驅動的 [Redis 佇列](/docs/{{version}}/queues) 提供了一個美觀的儀表板與程式碼驅動的設定方式。Horizon 讓您能輕鬆監控佇列系統的關鍵指標，例如任務吞吐量、執行時間與任務失敗情況。

使用 Horizon 時，您所有的佇列工作者設定都儲存在一個單一且簡單的設定檔中。藉由在受版本控制的檔案中定義應用程式的工作者設定，您可以在部署應用程式時輕鬆地縮放或修改應用程式的佇列工作者。

<img src="https://laravel.com/img/docs/horizon-example.png">

<a name="installation"></a>
## 安裝

> [!WARNING]
> Laravel Horizon 要求您使用 [Redis](https://redis.io) 來驅動您的佇列。因此，您應該確保應用程式的 `config/queue.php` 設定檔中，佇列連線被設定為 `redis`。Horizon 目前不支援 Redis Cluster。

您可以使用 Composer 套件管理員將 Horizon 安裝到您的專案中：

```shell
composer require laravel/horizon
```

安裝 Horizon 後，使用 `horizon:install` Artisan 指令發布其素材：

```shell
php artisan horizon:install
```


<a name="configuration"></a>
### 設定

發布 Horizon 的素材後，其主要設定檔將位於 `config/horizon.php`。此設定檔允許您為應用程式設定佇列工作者選項。每個設定選項都包含其用途說明，因此請務必詳閱此檔案。

> [!WARNING]
> Horizon 內部使用名為 `horizon` 的 Redis 連線。此 Redis 連線名稱是保留的，不應在 `database.php` 設定檔中分配給另一個 Redis 連線，也不應作為 `horizon.php` 設定檔中 `use` 選項的值。


<a name="environments"></a>
#### 環境

安裝後，您應該熟悉的主要 Horizon 設定選項是 `environments`。此設定選項是您的應用程式執行的環境陣列，並定義了每個環境的工作程序選項。預設情況下，此項目包含 `production` 和 `local` 環境。但是，您可以根據需要自由新增更多環境：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
        ],
    ],

    'local' => [
        'supervisor-1' => [
            'maxProcesses' => 3,
        ],
    ],
],
```

您也可以定義萬用字元環境 (`*`)，當找不到其他相符的環境時將使用該環境：

```php
'environments' => [
    // ...

    '*' => [
        'supervisor-1' => [
            'maxProcesses' => 3,
        ],
    ],
],
```

當您啟動 Horizon 時，它將根據您應用程式正在執行的環境使用工作程序設定選項。通常，環境是由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#determining-the-current-environment) 的值決定的。例如，預設的 `local` Horizon 環境被設定為啟動三個工作程序，並自動平衡分配給每個佇列的工作程序數量。預設的 `production` 環境被設定為最多啟動 10 個工作程序，並自動平衡分配給每個佇列的工作程序數量。

> [!WARNING]
> 您應該確保 `horizon` 設定檔中的 `environments` 部分包含您計畫執行 Horizon 的每個[環境](/docs/{{version}}/configuration#environment-configuration)項目。


<a name="supervisors"></a>
#### Supervisors

正如您在 Horizon 的預設設定檔中所見，每個環境可以包含一個或多個「Supervisor」。預設情況下，設定檔將此 Supervisor 定義為 `supervisor-1`；但是，您可以自由命名您的 Supervisor。每個 Supervisor 本質上負責「監督」一組工作程序，並負責平衡跨佇列的工作程序。

如果您想定義一組應在該環境中執行的新工作程序，您可以向給定環境新增額外的 Supervisor。如果您想為應用程式使用的特定佇列定義不同的平衡策略或工作程序數量，您可以選擇這樣做。


<a name="maintenance-mode"></a>
#### 維護模式

當您的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，除非在 Horizon 設定檔中將 Supervisor 的 `force` 選項定義為 `true`，否則 Horizon 將不會處理佇列任務：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'force' => true,
        ],
    ],
],
```


<a name="default-values"></a>
#### 預設值

在 Horizon 的預設設定檔中，您會注意到一個 `defaults` 設定選項。此設定選項指定了應用程式 [Supervisors](#supervisors) 的預設值。Supervisor 的預設設定值將合併到每個環境的 Supervisor 設定中，讓您在定義 Supervisor 時避免不必要的重複。


<a name="dashboard-authorization"></a>
### 儀表板授權

可以透過 `/horizon` 路由存取 Horizon 儀表板。預設情況下，您只能在 `local` 環境中存取此儀表板。但是，在您的 `app/Providers/HorizonServiceProvider.php` 檔案中，有一個[授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制了在**非本地**環境中對 Horizon 的存取。您可以根據需要自由修改此 Gate，以限制對 Horizon 安裝的存取：

```php
/**
 * Register the Horizon gate.
 *
 * This gate determines who can access Horizon in non-local environments.
 */
protected function gate(): void
{
    Gate::define('viewHorizon', function (User $user) {
        return in_array($user->email, [
            'taylor@laravel.com',
        ]);
    });
}
```


<a name="alternative-authentication-strategies"></a>
#### 替代的身份驗證策略

請記住，Laravel 會自動將已驗證的使用者注入到 Gate 閉包中。如果您的應用程式透過其他方法（例如 IP 限制）提供 Horizon 安全性，則您的 Horizon 使用者可能不需要「登入」。因此，您需要將上面的 `function (User $user)` 閉包簽章更改為 `function (User $user = null)`，以強制 Laravel 不要求驗證。


<a name="max-job-attempts"></a>
### 最大任務嘗試次數

> [!NOTE]
> 在調整這些選項之前，請確保您熟悉 Laravel 預設的[佇列服務](/docs/{{version}}/queues#max-job-attempts-and-timeout)以及「嘗試 (attempts)」的概念。

您可以在 Supervisor 的設定中定義任務可以消耗的最大嘗試次數：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'tries' => 10,
        ],
    ],
],
```

> [!NOTE]
> 此選項類似於使用 Artisan 指令處理佇列時的 `--tries` 選項。

當使用諸如 `WithoutOverlapping` 或 `RateLimited` 之類的中介層時，調整 `tries` 選項至關重要，因為它們會消耗嘗試次數。要處理此問題，請在 Supervisor 層級調整 `tries` 設定值，或在任務類別中定義 `$tries` 屬性。

如果您沒有設定 `tries` 選項，Horizon 預設為單次嘗試，除非任務類別定義了 `$tries`，其優先級高於 Horizon 設定。

將 `tries` 或 `$tries` 設定為 0 允許無限次嘗試，這在嘗試次數不確定時非常理想。為了防止無止盡的失敗，您可以透過在任務類別上設定 `$maxExceptions` 屬性來限制允許的例外數量。

<a name="job-timeout"></a>
### 任務超時

同樣地，您可以在 supervisor 層級設定 `timeout` 值，這指定了工作者程序在被強制終止之前可以執行任務的秒數。一旦被終止，該任務將根據您的佇列設定進行重試或標記為失敗：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...¨
            'timeout' => 60,
        ],
    ],
],
```

> [!WARNING]
> 當使用 `auto` 平衡策略時，Horizon 會將處理中的工作者視為「掛起 (hanging)」，並在縮減規模時，於 Horizon 超時後強制刪除它們。請務必確保 Horizon 的超時時間大於任何任務層級的超時時間，否則任務可能會在執行到一半時被終止。此外，`timeout` 值應始終比 `config/queue.php` 設定檔中定義的 `retry_after` 值短至少幾秒鐘。否則，您的任務可能會被處理兩次。


<a name="job-backoff"></a>
### 任務重試延遲

您可以在 supervisor 層級定義 `backoff` 值，以指定 Horizon 在遇到未處理的異常並重試任務之前應等待多長時間：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'backoff' => 10,
        ],
    ],
],
```

您也可以透過將 `backoff` 值設定為陣列來配置「指數型」重試延遲。在此範例中，第一次重試的延遲時間為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有剩餘的嘗試次數，則後續每次重試皆為 10 秒：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'backoff' => [1, 5, 10],
        ],
    ],
],
```


<a name="silenced-jobs"></a>
### 靜音任務

有時，您可能對查看應用程式或第三方套件發送的某些任務不感興趣。您可以將這些任務靜音，而不是讓它們佔用「已完成任務 (Completed Jobs)」列表的空間。要開始使用，請將任務的類別名稱添加到應用程式 `horizon` 設定檔中的 `silenced` 設定選項中：

```php
'silenced' => [
    App\Jobs\ProcessPodcast::class,
],
```

除了將單個任務類別靜音之外，Horizon 還支援基於[標籤](#tags)靜音任務。如果您想隱藏共享相同標籤的多個任務，這將非常有用：

```php
'silenced_tags' => [
    'notifications'
],
```

或者，您希望靜音的任務可以實作 `Laravel\Horizon\Contracts\Silenced` 介面。如果任務實作了此介面，它將自動被靜音，即使它不在 `silenced` 設定陣列中：

```php
use Laravel\Horizon\Contracts\Silenced;

class ProcessPodcast implements ShouldQueue, Silenced
{
    use Queueable;

    // ...
}
```

<a name="balancing-strategies"></a>
## 平衡策略

每個 supervisor 可以處理一個或多個佇列，但與 Laravel 預設的佇列系統不同，Horizon 允許您從三種 worker 平衡策略中進行選擇：`auto`、`simple` 和 `false`。


<a name="auto-balancing"></a>
### 自動平衡

`auto` 策略是預設的策略，它會根據佇列目前的工作負載調整每個佇列的 worker 進程數量。例如，如果您的 `notifications` 佇列有 1,000 個待處理任務，而您的 `default` 佇列是空的，Horizon 將會分配更多 worker 給 `notifications` 佇列，直到該佇列清空為止。

使用 `auto` 策略時，您還可以設定 `minProcesses` 和 `maxProcesses` 選項：

<div class="content-list" markdown="1">

- `minProcesses` 定義每個佇列的最少 worker 進程數。此值必須大於或等於 1。
- `maxProcesses` 定義 Horizon 在所有佇列中可擴充的最大 worker 進程總數。此值通常應大於佇列數量乘以 `minProcesses` 的值。若要防止 supervisor 產生任何進程，您可以將此值設為 0。

</div>

例如，您可以設定 Horizon 為每個佇列維持至少一個進程，並擴充到總共 10 個 worker 進程：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['default', 'notifications'],
            'balance' => 'auto',
            'autoScalingStrategy' => 'time',
            'minProcesses' => 1,
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
        ],
    ],
],
```

`autoScalingStrategy` 設定選項決定了 Horizon 如何將更多 worker 進程分配給佇列。您可以從兩種策略中選擇：

<div class="content-list" markdown="1">

- `time` 策略會根據清除佇列所需的預估總時間來分配 worker。
- `size` 策略會根據佇列中的任務總數來分配 worker。

</div>

`balanceMaxShift` 和 `balanceCooldown` 的設定值決定了 Horizon 擴充以滿足 worker 需求的快速程度。在上面的範例中，每三秒最多會建立或銷毀一個新進程。您可以根據應用程式的需求隨時調整這些值。


<a name="auto-queue-priorities"></a>
#### 佇列優先級與自動平衡

當使用 `auto` 平衡策略時，Horizon 不會在佇列之間強制執行嚴格的優先級。supervisor 設定中佇列的順序不會影響 worker 進程的分配方式。相反地，Horizon 依靠選定的 `autoScalingStrategy` 根據佇列負載動態分配 worker 進程。

例如，在以下設定中，儘管 high 佇列出現在列表的第一個，但它並不會比 default 佇列具有更高的優先級：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'queue' => ['high', 'default'],
            'minProcesses' => 1,
            'maxProcesses' => 10,
        ],
    ],
],
```

如果您需要強制執行佇列之間的相對優先級，可以定義多個 supervisor 並顯式分配處理資源：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'queue' => ['default'],
            'minProcesses' => 1,
            'maxProcesses' => 10,
        ],
        'supervisor-2' => [
            // ...
            'queue' => ['images'],
            'minProcesses' => 1,
            'maxProcesses' => 1,
        ],
    ],
],
```

在此範例中，default `queue` 可以擴充到 10 個進程，而 `images` 佇列則限制為一個進程。此設定可確保您的佇列可以獨立擴充。

> [!NOTE]
> 在發送資源密集型任務時，有時最好將它們分配給具有限制 `maxProcesses` 值的專用佇列。否則，這些任務可能會消耗過多的 CPU 資源並導致系統過載。


<a name="simple-balancing"></a>
### 簡單平衡

`simple` 策略會在指定的佇列中平均分配 worker 進程。使用此策略，Horizon 不會自動調整 worker 進程的數量。相反地，它使用固定數量的進程：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'queue' => ['default', 'notifications'],
            'balance' => 'simple',
            'processes' => 10,
        ],
    ],
],
```

在上面的範例中，Horizon 將為每個佇列分配 5 個進程，平均分配總共 10 個進程。

如果您想單獨控制分配給每個佇列的 worker 進程數量，可以定義多個 supervisor：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'queue' => ['default'],
            'balance' => 'simple',
            'processes' => 10,
        ],
        'supervisor-notifications' => [
            // ...
            'queue' => ['notifications'],
            'balance' => 'simple',
            'processes' => 2,
        ],
    ],
],
```

透過此設定，Horizon 將為 `default` 佇列分配 10 個進程，並為 `notifications` 佇列分配 2 個進程。


<a name="no-balancing"></a>
### 不進行平衡

當 `balance` 選項設為 `false` 時，Horizon 會嚴格按照佇列列出的順序處理它們，這與 Laravel 預設的佇列系統相似。然而，如果任務開始累積，它仍然會擴充 worker 進程的數量：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'queue' => ['default', 'notifications'],
            'balance' => false,
            'minProcesses' => 1,
            'maxProcesses' => 10,
        ],
    ],
],
```

在上面的範例中，`default` 佇列中的任務總是比 `notifications` 佇列中的任務具有更高的優先級。例如，如果 `default` 中有 1,000 個任務，而 `notifications` 中只有 10 個任務，Horizon 將會先完整處理所有 `default` 任務，然後才處理來自 `notifications` 的任何任務。

您可以使用 `minProcesses` 和 `maxProcesses` 選項控制 Horizon 擴充 worker 進程的能力：

<div class="content-list" markdown="1">

- `minProcesses` 定義總共的最少 worker 進程數。此值必須大於或等於 1。
- `maxProcesses` 定義 Horizon 可擴充的最大 worker 進程總數。

</div>


<a name="upgrading-horizon"></a>
## 升級 Horizon

當升級到 Horizon 的新重大版本時，請務必仔細閱讀 [升級指南](https://github.com/laravel/horizon/blob/master/UPGRADE.md)。

<a name="running-horizon"></a>
## 執行 Horizon

當您在應用程式的 `config/horizon.php` 設定檔中設定好您的 supervisors 與 workers 後，您可以使用 `horizon` Artisan 指令來啟動 Horizon。此單一指令將啟動目前環境中所有已設定的 worker 處理程序：

```shell
php artisan horizon
```

您可以使用 `horizon:pause` 和 `horizon:continue` Artisan 指令來暫停 Horizon 處理程序，或是指示其繼續處理任務：

```shell
php artisan horizon:pause

php artisan horizon:continue
```

您也可以使用 `horizon:pause-supervisor` 和 `horizon:continue-supervisor` Artisan 指令來暫停或繼續特定的 Horizon [supervisors](#supervisors)：

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

您可以使用 `horizon:status` Artisan 指令來檢查 Horizon 處理程序的目前狀態：

```shell
php artisan horizon:status
```

您可以使用 `horizon:supervisor-status` Artisan 指令來檢查特定 Horizon [supervisor](#supervisors) 的目前狀態：

```shell
php artisan horizon:supervisor-status supervisor-1
```

您可以使用 `horizon:terminate` Artisan 指令來優雅地終止 Horizon 處理程序。任何目前正在處理中的任務都會被完成，然後 Horizon 就會停止執行：

```shell
php artisan horizon:terminate
```


<a name="automatically-restarting-horizon"></a>
#### 自動重新啟動 Horizon

在本地開發期間，您可以執行 `horizon:listen` 指令。使用 `horizon:listen` 指令時，當您想要重新載入更新後的程式碼時，不必手動重新啟動 Horizon。在使用此功能之前，您應確保本地開發環境中已安裝 [Node](https://nodejs.org)。此外，您應該在專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監控函式庫：

```shell
npm install --save-dev chokidar
```

安裝 Chokidar 後，您可以使用 `horizon:listen` 指令啟動 Horizon：

```shell
php artisan horizon:listen
```

在 Docker 或 Vagrant 中執行時，您應使用 `--poll` 選項：

```shell
php artisan horizon:listen --poll
```

您可以在應用程式的 `config/horizon.php` 設定檔中使用 `watch` 設定選項，來設定應該被監控的目錄和檔案：

```php
'watch' => [
    'app',
    'bootstrap',
    'config',
    'database',
    'public/**/*.php',
    'resources/**/*.php',
    'routes',
    'composer.lock',
    '.env',
],
```


<a name="deploying-horizon"></a>
### 部署 Horizon

當您準備好將 Horizon 部署到應用程式的正式伺服器時，您應該設定一個處理程序監控工具來監控 `php artisan horizon` 指令，並在它意外結束時重新啟動它。別擔心，我們將在下面討論如何安裝處理程序監控工具。

在應用程式的部署過程中，您應該指示 Horizon 處理程序終止，以便它能被處理程序監控工具重新啟動並接收您的程式碼變更：

```shell
php artisan horizon:terminate
```


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的處理程序監控工具，當您的 `horizon` 處理程序停止執行時，它會自動重新啟動它。要在 Ubuntu 上安裝 Supervisor，您可以使用以下指令。如果您不是使用 Ubuntu，您通常可以使用作業系統的套件管理器來安裝 Supervisor：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果您覺得手動設定 Supervisor 太過繁瑣，可以考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它可以為您的 Laravel 應用程式管理背景處理程序。


<a name="supervisor-configuration"></a>
#### Supervisor 設定

Supervisor 設定檔通常儲存在伺服器的 `/etc/supervisor/conf.d` 目錄中。在此目錄內，您可以建立任意數量的設定檔，以指示 Supervisor 如何監控您的處理程序。例如，讓我們建立一個 `horizon.conf` 檔案來啟動並監控 `horizon` 處理程序：

```ini
[program:horizon]
process_name=%(program_name)s
command=php /home/forge/example.com/artisan horizon
autostart=true
autorestart=true
user=forge
redirect_stderr=true
stdout_logfile=/home/forge/example.com/horizon.log
stopwaitsecs=3600
```

在定義 Supervisor 設定時，您應確保 `stopwaitsecs` 的值大於執行時間最長的任務所消耗的秒數。否則，Supervisor 可能會在任務完成處理之前就將其刪除。

> [!WARNING]
> 雖然上述範例適用於基於 Ubuntu 的伺服器，但 Supervisor 設定檔預期的位置和副檔名可能會因其他伺服器作業系統而異。請參閱您伺服器的說明文件以取得更多資訊。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

建立設定檔後，您可以使用以下指令更新 Supervisor 設定並啟動受監控的處理程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> [!NOTE]
> 有關執行 Supervisor 的更多資訊，請參閱 [Supervisor 說明文件](http://supervisord.org/index.html)。


<a name="tags"></a>
## 標籤

Horizon 允許您為任務分配「標籤 (tags)」，包括 mailables、廣播事件、通知和佇列事件監聽器。事實上，Horizon 會根據附加在任務上的 Eloquent 模型，智慧且自動地為大多數任務標記標籤。例如，看看以下任務：

```php
<?php

namespace App\Jobs;

use App\Models\Video;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class RenderVideo implements ShouldQueue
{
    use Queueable;

    /**
     * Create a new job instance.
     */
    public function __construct(
        public Video $video,
    ) {}

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        // ...
    }
}
```

如果此任務在加入佇列時帶有一個 `App\Models\Video` 實例，且該實例的 `id` 屬性為 `1`，它將自動獲得 `App\Models\Video:1` 標籤。這是因為 Horizon 會在任務的屬性中搜尋任何 Eloquent 模型。如果找到了 Eloquent 模型，Horizon 將使用該模型的類別名稱和主鍵智慧地為任務標記標籤：

```php
use App\Jobs\RenderVideo;
use App\Models\Video;

$video = Video::find(1);

RenderVideo::dispatch($video);
```


<a name="manually-tagging-jobs"></a>
#### 手動標記任務

如果您想要手動為其中一個可加入佇列的物件定義標籤，您可以在類別中定義一個 `tags` 方法：

```php
class RenderVideo implements ShouldQueue
{
    /**
     * Get the tags that should be assigned to the job.
     *
     * @return array<int, string>
     */
    public function tags(): array
    {
        return ['render', 'video:'.$this->video->id];
    }
}
```


<a name="manually-tagging-event-listeners"></a>
#### 手動標記事件監聽器

在取得佇列事件監聽器的標籤時，Horizon 會自動將事件實例傳遞給 `tags` 方法，讓您可以將事件資料加入標籤中：

```php
class SendRenderNotifications implements ShouldQueue
{
    /**
     * Get the tags that should be assigned to the listener.
     *
     * @return array<int, string>
     */
    public function tags(VideoRendered $event): array
    {
        return ['video:'.$event->video->id];
    }
}
```

<a name="notifications"></a>
## 通知

> [!WARNING]
> 當設定 Horizon 傳送 Slack 或 SMS 通知時，您應該查看 [相關通知管道的先決條件](/docs/{{version}}/notifications)。

如果您希望在佇列等待時間過長時收到通知，可以使用 `Horizon::routeMailNotificationsTo`、`Horizon::routeSlackNotificationsTo` 以及 `Horizon::routeSmsNotificationsTo` 方法。您可以在應用程式的 `App\Providers\HorizonServiceProvider` 的 `boot` 方法中呼叫這些方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    parent::boot();

    Horizon::routeSmsNotificationsTo('15556667777');
    Horizon::routeMailNotificationsTo('example@example.com');
    Horizon::routeSlackNotificationsTo('slack-webhook-url', '#channel');
}
```


<a name="configuring-notification-wait-time-thresholds"></a>
#### 設定通知等待時間臨界值

您可以在應用程式的 `config/horizon.php` 設定檔中設定多少秒數被視為「等待時間過長」。該檔案中的 `waits` 設定選項允許您控制每個連線 / 佇列組合的過長等待臨界值。任何未定義的連線 / 佇列組合將預設為 60 秒的臨界值：

```php
'waits' => [
    'redis:critical' => 30,
    'redis:default' => 60,
    'redis:batch' => 120,
],
```

將佇列的臨界值設為 `0` 將停用該佇列的等待時間過長通知。


<a name="metrics"></a>
## 指標

Horizon 包含一個指標儀表板，提供有關任務與佇列等待時間以及吞吐量的資訊。為了產生此儀表板的資料，您應該在應用程式的 `routes/console.php` 檔案中設定 Horizon 的 `snapshot` Artisan 指令每五分鐘執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('horizon:snapshot')->everyFiveMinutes();
```

如果您想刪除所有指標資料，可以執行 `horizon:clear-metrics` Artisan 指令：

```shell
php artisan horizon:clear-metrics
```


<a name="deleting-failed-jobs"></a>
## 刪除失敗的任務

如果您想刪除一個失敗的任務，可以使用 `horizon:forget` 指令。`horizon:forget` 指令接受失敗任務的 ID 或 UUID 作為唯一參數：

```shell
php artisan horizon:forget 5
```

如果您想刪除所有失敗的任務，可以為 `horizon:forget` 指令提供 `--all` 選項：

```shell
php artisan horizon:forget --all
```


<a name="clearing-jobs-from-queues"></a>
## 清除佇列中的任務

如果您想從應用程式的預設佇列中刪除所有任務，可以使用 `horizon:clear` Artisan 指令：

```shell
php artisan horizon:clear
```

您可以提供 `queue` 選項來刪除特定佇列中的任務：

```shell
php artisan horizon:clear --queue=emails
```