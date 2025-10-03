# Laravel Horizon

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [Dashboard 授權](#dashboard-authorization)
    - [任務最大嘗試次數](#max-job-attempts)
    - [任務逾時](#job-timeout)
    - [任務退避](#job-backoff)
    - [靜默任務](#silenced-jobs)
- [平衡策略](#balancing-strategies)
    - [自動平衡](#auto-balancing)
    - [簡單平衡](#simple-balancing)
    - [不平衡](#no-balancing)
- [升級 Horizon](#upgrading-horizon)
- [執行 Horizon](#running-horizon)
    - [部署 Horizon](#deploying-horizon)
- [標籤](#tags)
- [通知](#notifications)
- [指標](#metrics)
- [刪除失敗的任務](#deleting-failed-jobs)
- [從佇列中清除任務](#clearing-jobs-from-queues)

<a name="introduction"></a>
## 簡介

> [!NOTE]
> 在深入研究 Laravel Horizon 之前，建議先熟悉 Laravel 的基礎[佇列服務](/docs/{{version}}/queues)。Horizon 為 Laravel 佇列擴充了額外的功能，若您還不熟悉 Laravel 提供的基本佇列功能，可能會對這些額外功能感到困惑。

[Laravel Horizon](https://github.com/laravel/horizon) 為您由 Laravel 驅動的 [Redis 佇列](/docs/{{version}}/queues) 提供了一個美觀的儀表板與程式碼驅動的設定。Horizon 讓您能輕鬆監控佇列系統的關鍵指標，例如任務吞吐量、執行時間與任務失敗狀況。

使用 Horizon 時，您所有的佇列 Worker 設定都儲存在一個簡單的設定檔中。透過在版本控制的檔案中定義您應用程式的 Worker 設定，您便可以在部署應用程式時輕鬆地擴展或修改應用程式的佇列 Worker。

<img src="https://laravel.com/img/docs/horizon-example.png">

<a name="installation"></a>
## 安裝

> [!WARNING]
> Laravel Horizon 需要使用 [Redis](https://redis.io) 來驅動你的佇列。因此，你應確保應用程式 `config/queue.php` 設定檔中的佇列連線已設為 `redis`。

你可以使用 Composer 套件管理器將 Horizon 安裝到你的專案中：

```shell
composer require laravel/horizon
```

安裝 Horizon 後，請使用 `horizon:install` Artisan 指令來發布其資產：

```shell
php artisan horizon:install
```


<a name="configuration"></a>
### 設定

發布 Horizon 的資產後，其主要設定檔會位於 `config/horizon.php`。此設定檔可讓你為應用程式設定佇列 Worker 的選項。每個設定選項都包含了其用途的說明，所以請務必徹底地詳閱這個檔案。

> [!WARNING]
> Horizon 內部使用一個名為 `horizon` 的 Redis 連線。這個 Redis 連線名稱是保留的，不應在 `database.php` 設定檔中指派給另一個 Redis 連線，也不應作為 `horizon.php` 設定檔中 `use` 選項的值。


<a name="environments"></a>
#### 環境

安裝後，你應該熟悉的主要 Horizon 設定選項是 `environments` 設定選項。此設定選項是一個應用程式執行的環境陣列，並為每個環境定義了 Worker 行程的選項。預設情況下，這個項目包含一個 `production` 和一個 `local` 環境。但是，你可以根據需要自由地新增更多環境：

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

你也可以定義一個萬用字元環境 (`*`)，當找不到其他匹配的環境時，就會使用這個環境：

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

當你啟動 Horizon 時，它會使用應用程式目前執行環境的 Worker 行程設定選項。通常，環境是由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#determining-the-current-environment) 的值決定的。例如，預設的 `local` Horizon 環境被設定為啟動三個 Worker 行程，並自動平衡分配給每個佇列的 Worker 行程數量。預設的 `production` 環境被設定為最多啟動 10 個 Worker 行程，並自動平衡分配給每個佇列的 Worker 行程數量。

> [!WARNING]
> 你應確保 `horizon` 設定檔的 `environments` 部分包含了你計劃執行 Horizon 的每個[環境](/docs/{{version}}/configuration#environment-configuration)的項目。


<a name="supervisors"></a>
#### Supervisors

正如你在 Horizon 的預設設定檔中看到的，每個環境可以包含一個或多個「supervisors」。預設情況下，設定檔將這個 supervisor 定義為 `supervisor-1`；但是，你可以隨意為你的 supervisors 命名。每個 supervisor 基本上負責「監督」一群 Worker 行程，並處理跨佇列的 Worker 行程平衡。

如果你想定義一個應在該環境中執行的新 Worker 行程群組，你可以為指定的環境新增額外的 supervisors。如果你想為應用程式使用的特定佇列定義不同的平衡策略或 Worker 行程數量，你可以選擇這樣做。


<a name="maintenance-mode"></a>
#### 維護模式

當你的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，佇列中的任務將不會被 Horizon 處理，除非在 Horizon 設定檔中將 supervisor 的 `force` 選項定義為 `true`：

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

在 Horizon 的預設設定檔中，你會注意到一個 `defaults` 設定選項。此設定選項指定了你的應用程式 [supervisors](#supervisors) 的預設值。Supervisor 的預設設定值將被合併到每個環境的 supervisor 設定中，讓你能夠在定義 supervisors 時避免不必要的重複。


<a name="dashboard-authorization"></a>
### Dashboard 授權

可以透過 `/horizon` 路由存取 Horizon Dashboard。預設情況下，你只能在 `local` 環境中存取此 Dashboard。然而，在你的 `app/Providers/HorizonServiceProvider.php` 檔案中，有一個[授權 Gate](/docs/{{version}}/authorization#gates) 的定義。這個授權 Gate 控制著在**非本地**環境中對 Horizon 的存取。你可以根據需要自由修改此 Gate，以限制對 Horizon 安裝的存取：

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
#### 替代的驗證策略

請記住，Laravel 會自動將已驗證的使用者注入到 Gate 的閉包中。如果你的應用程式是透過其他方法（例如 IP 限制）來提供 Horizon 的安全性，那麼你的 Horizon 使用者可能不需要「登入」。因此，你需要將上面的 `function (User $user)` 閉包簽名更改為 `function (User $user = null)`，以強制 Laravel 不要求驗證。


<a name="max-job-attempts"></a>
### 任務最大嘗試次數

> [!NOTE]
> 在調整這些選項之前，請確保你熟悉 Laravel 預設的[佇列服務](/docs/{{version}}/queues#max-job-attempts-and-timeout)以及「attempts」的概念。

你可以在 supervisor 的設定中定義一個任務可以消耗的最大嘗試次數：

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
> 此選項與使用 Artisan 指令處理佇列時的 `--tries` 選項相似。

當使用像 `WithoutOverlapping` 或 `RateLimited` 這樣的中介層時，調整 `tries` 選項是至關重要的，因為它們會消耗嘗試次數。要處理這個問題，可以在 supervisor 層級調整 `tries` 設定值，或在任務類別上定義 `$tries` 屬性。

如果你沒有設定 `tries` 選項，Horizon 預設為單次嘗試，除非任務類別定義了 `$tries`，它的優先級高於 Horizon 的設定。

將 `tries` 或 `$tries` 設定為 0 允許無限次嘗試，這在嘗試次數不確定的情況下是理想的。為了防止無休止的失敗，你可以透過在任務類別上設定 `$maxExceptions` 屬性來限制允許的例外數量。

<a name="job-timeout"></a>
### 任務逾時

同樣地，你可以在 supervisor 層級設定 `timeout` 值，用來指定一個 Worker 行程在被強制終止前，可以執行一個任務多久（秒）。一旦被終止，該任務將會被重試或標記為失敗，這取決於你的佇列設定：

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
> 當使用 `auto` 平衡策略時，Horizon 在縮減規模（scale down）期間會將處理中的 Worker 視為「懸置（hanging）」，並在 Horizon 逾時後強制終止它們。務必確保 Horizon 的逾時時間大於任何任務層級的逾時時間，否則任務可能會在執行中被終止。此外，`timeout` 的值應永遠比 `config/queue.php` 設定檔中定義的 `retry_after` 值短至少幾秒鐘。否則，你的任務可能會被處理兩次。

<a name="job-backoff"></a>
### 任務退避

你可以在 supervisor 層級定義 `backoff` 值，來指定 Horizon 在遇到未處理的例外時，應等待多久才重試任務：

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

你也可以透過使用陣列作為 `backoff` 值來設定「指數型」退避。在此範例中，第一次重試的延遲時間為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，如果還有更多嘗試次數，之後的每次重試都將是 10 秒：

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
### 靜默任務

有時候，你可能不想看到應用程式或第三方套件所分派的某些任務。與其讓這些任務佔用你「已完成任務」列表的空間，你可以將它們設為靜默。要開始使用，請將任務的類別名稱加到應用程式 `horizon` 設定檔的 `silenced` 設定選項中：

```php
'silenced' => [
    App\Jobs\ProcessPodcast::class,
],
```

除了靜默個別的任務類別外，Horizon 也支援根據[標籤](#tags)來靜默任務。如果你想隱藏共用相同標籤的多個任務，這會很有用：

```php
'silenced_tags' => [
    'notifications'
],
```

或者，你希望靜默的任務可以實作 `Laravel\Horizon\Contracts\Silenced` 介面。如果任務實作了這個介面，它將會自動被靜默，即使它不存在於 `silenced` 設定陣列中：

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

每個 Supervisor 都可以處理一個或多個佇列，但與 Laravel 的預設佇列系統不同，Horizon 允許你從三種 Worker 平衡策略中選擇：`auto`、`simple` 和 `false`。


<a name="auto-balancing"></a>
### 自動平衡

`auto` 策略是預設策略，它會根據佇列目前的負載來調整每個佇列的 Worker Process 數量。例如，如果你的 `notifications` 佇列有 1,000 個待處理的任務，而你的 `default` 佇列是空的，Horizon 會分配更多的 Worker 到你的 `notifications` 佇列，直到該佇列為空。

使用 `auto` 策略時，你也可以設定 `minProcesses` 和 `maxProcesses` 設定選項：

<div class="content-list" markdown="1">

- `minProcesses` 定義了每個佇列的最小 Worker Process 數量。此值必須大於或等於 1。
- `maxProcesses` 定義了 Horizon 在所有佇列中最多可以擴展到的 Worker Process 總數。此值通常應大於佇列數量乘以 `minProcesses` 的值。若要防止 Supervisor 產生任何 Process，你可以將此值設為 0。

</div>

例如，你可以設定 Horizon 為每個佇列至少維持一個 Process，並最多擴展到總共 10 個 Worker Process：

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

`autoScalingStrategy` 設定選項決定了 Horizon 如何將更多的 Worker Process 分配給佇列。你可以選擇兩種策略：

<div class="content-list" markdown="1">

- `time` 策略會根據清除佇列所需的總預估時間來分配 Worker。
- `size` 策略會根據佇列上的任務總數來分配 Worker。

</div>

`balanceMaxShift` 和 `balanceCooldown` 設定值決定了 Horizon 擴展以滿足 Worker 需求的速度。在上面的範例中，每三秒最多會建立或銷毀一個新的 Process。你可以根據應用程式的需求自由調整這些值。


<a name="auto-queue-priorities"></a>
#### 佇列優先權與自動平衡

當使用 `auto` 平衡策略時，Horizon 不會在佇列之間強制執行嚴格的優先順序。Supervisor 設定中佇列的順序不會影響 Worker Process 的分配方式。相反地，Horizon 依賴所選的 `autoScalingStrategy` 來根據佇列負載動態分配 Worker Process。

例如，在以下設定中，儘管 high 佇列在列表中排在前面，但它並不會優先於 default 佇列：

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

如果你需要在佇列之間強制執行相對優先權，你可以定義多個 Supervisor 並明確分配處理資源：

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

在這個範例中，預設的 `queue` 可以擴展到 10 個 Process，而 `images` 佇列則限制為一個 Process。此設定確保你的佇列可以獨立擴展。

> [!NOTE]
> 當派送資源密集型任務時，有時最好將它們分配到一個具有有限 `maxProcesses` 值的專用佇列中。否則，這些任務可能會消耗過多的 CPU 資源並使你的系統超載。


<a name="simple-balancing"></a>
### 簡單平衡

`simple` 策略會將 Worker Process 均勻地分配到指定的佇列中。使用此策略時，Horizon 不會自動擴展 Worker Process 的數量，而是使用固定的 Process 數量：

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

在上面的範例中，Horizon 將為每個佇列分配 5 個 Process，將總共 10 個 Process 均分。

如果你想單獨控制分配給每個佇列的 Worker Process 數量，你可以定義多個 Supervisor：

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

透過此設定，Horizon 將為 `default` 佇列分配 10 個 Process，並為 `notifications` 佇列分配 2 個 Process。


<a name="no-balancing"></a>
### 不平衡

當 `balance` 選項設為 `false` 時，Horizon 會嚴格按照佇列的列出順序來處理它們，類似於 Laravel 的預設佇列系統。然而，如果任務開始累積，它仍然會擴展 Worker Process 的數量：

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

在上面的範例中，`default` 佇列中的任務總是優先於 `notifications` 佇列中的任務。例如，如果 `default` 中有 1,000 個任務，而 `notifications` 中只有 10 個，Horizon 會在處理任何來自 `notifications` 的任務之前，完全處理所有 `default` 任務。

你可以使用 `minProcesses` 和 `maxProcesses` 選項來控制 Horizon 擴展 Worker Process 的能力：

<div class="content-list" markdown="1">

- `minProcesses` 定義了總共的最小 Worker Process 數量。此值必須大於或等於 1。
- `maxProcesses` 定義了 Horizon 最多可以擴展到的 Worker Process 總數。

</div>


<a name="upgrading-horizon"></a>
## 升級 Horizon

當升級到 Horizon 的新主要版本時，請務必仔細查閱[升級指南](https://github.com/laravel/horizon/blob/master/UPGRADE.md)。

<a name="running-horizon"></a>
## 執行 Horizon

在應用程式的 `config/horizon.php` 設定檔中設定好 Supervisor 與 Worker 後，就可以使用 `horizon` Artisan 指令來啟動 Horizon。這個指令會為當前環境啟動所有已設定的 Worker 行程：

```shell
php artisan horizon
```

可以使用 `horizon:pause` 與 `horizon:continue` Artisan 指令來暫停 Horizon 行程並指示它繼續處理任務：

```shell
php artisan horizon:pause

php artisan horizon:continue
```

也可以使用 `horizon:pause-supervisor` 與 `horizon:continue-supervisor` Artisan 指令來暫停或繼續特定的 Horizon [Supervisor](#supervisors)：

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

可以使用 `horizon:status` Artisan 指令來檢查 Horizon 行程目前的狀態：

```shell
php artisan horizon:status
```

可以使用 `horizon:supervisor-status` Artisan 指令來檢查特定 Horizon [Supervisor](#supervisors) 的目前狀態：

```shell
php artisan horizon:supervisor-status supervisor-1
```

可以使用 `horizon:terminate` Artisan 指令來平滑地終止 Horizon 行程。任何正在處理的任務都會被完成，然後 Horizon 就會停止執行：

```shell
php artisan horizon:terminate
```


<a name="deploying-horizon"></a>
### 部署 Horizon

當準備好要將 Horizon 部署到應用程式的正式伺服器上時，應設定一個行程監控器來監控 `php artisan horizon` 指令，並在該指令意外結束時重啟它。別擔心，我們下面會討論如何安裝行程監控器。

在應用程式部署的過程中，應指示 Horizon 行程終止，這樣它就會被行程監控器重啟並接收到程式碼的變更：

```shell
php artisan horizon:terminate
```


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統上的一個行程監控器，若 `horizon` 行程停止執行，它會自動重啟。若要在 Ubuntu 上安裝 Supervisor，可使用下列指令。若不是使用 Ubuntu，則大概也可以用作業系統的套件管理員來安裝 Supervisor：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 若覺得自行設定 Supervisor 太過困難，可以考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它能為你的 Laravel 應用程式管理背景行程。


<a name="supervisor-configuration"></a>
#### Supervisor 設定

Supervisor 的設定檔通常儲存在伺服器的 `/etc/supervisor/conf.d` 目錄下。在這個目錄中，可以建立任意數量的設定檔來指示 Supervisor 應如何監控你的行程。舉例來說，讓我們來建立一個 `horizon.conf` 檔案來啟動並監控 `horizon` 行程：

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

定義 Supervisor 設定時，應確保 `stopwaitsecs` 的值大於執行時間最長的任務所花費的秒數。否則，Supervisor 可能會在任務處理完成前就將其終止。

> [!WARNING]
> 雖然上述範例適用於基於 Ubuntu 的伺服器，但 Supervisor 設定檔預期的位置與副檔名可能會因伺服器作業系統而異。請參考伺服器的說明文件以了解更多資訊。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

建立好設定檔後，就可以使用下列指令來更新 Supervisor 設定並啟動受監控的行程：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> [!NOTE]
> 有關執行 Supervisor 的更多資訊，請參考 [Supervisor 說明文件](http://supervisord.org/index.html)。


<a name="tags"></a>
## 標籤

Horizon 允許我們為任務指派「標籤」，包含 Mailable、廣播事件、通知、以及佇列化的事件監聽器。實際上，Horizon 會根據附加到任務上的 Eloquent Model，為大多數的任務智慧地自動加上標籤。舉例來說，請參考下列任務：

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

若這個任務在佇列中時帶有一個 `id` 屬性為 `1` 的 `App\Models\Video` 實體，則該任務會自動收到 `App\Models\Video:1` 這個標籤。這是因為 Horizon 會在任務的屬性中搜尋所有的 Eloquent Model。若有找到 Eloquent Model，Horizon 會使用 Model 的類別名稱與主鍵來智慧地為任務加上標籤：

```php
use App\Jobs\RenderVideo;
use App\Models\Video;

$video = Video::find(1);

RenderVideo::dispatch($video);
```


<a name="manually-tagging-jobs"></a>
#### 手動為任務加上標籤

若想為其中一個可佇列物件手動定義標籤，可以在該類別上定義一個 `tags` 方法：

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
#### 手動為事件監聽器加上標籤

在擷取佇列化事件監聽器的標籤時，Horizon 會自動將事件實體傳給 `tags` 方法，讓你可以將事件資料加到標籤上：

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
> 在設定 Horizon 來傳送 Slack 或 SMS 通知時，應先詳閱[相關通知頻道的先決條件](/docs/{{version}}/notifications)。

若想在其中一個佇列等待時間過長時收到通知，可使用 `Horizon::routeMailNotificationsTo`、`Horizon::routeSlackNotificationsTo`、以及 `Horizon::routeSmsNotificationsTo` 等方法。可以在應用程式 `App\Providers\HorizonServiceProvider` 的 `boot` 方法中呼叫這些方法：

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
#### 設定通知等待時間閾值

我們可以在應用程式的 `config/horizon.php` 設定檔中設定幾秒算是「長時間等待」。該檔案中的 `waits` 設定選項讓我們能為每個連線 / 佇列組合控制長時間等待的閾值。任何未定義的連線 / 佇列組合都會預設為 60 秒的長時間等待閾值：

```php
'waits' => [
    'redis:critical' => 30,
    'redis:default' => 60,
    'redis:batch' => 120,
],
```

將佇列的閾值設為 `0` 即可停用該佇列的長時間等待通知。

<a name="metrics"></a>
## 指標

Horizon 包含了一個指標 Dashboard，其中提供了有關任務與佇列等待時間及吞吐量的資訊。為了填入這個 Dashboard 的資料，我們應在應用程式的 `routes/console.php` 檔中設定 Horizon 的 `snapshot` Artisan 指令，讓該指令每五分鐘執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('horizon:snapshot')->everyFiveMinutes();
```

若想刪除所有指標資料，可以叫用 `horizon:clear-metrics` 這個 Artisan 指令：

```shell
php artisan horizon:clear-metrics
```


<a name="deleting-failed-jobs"></a>
## 刪除失敗的任務

若想刪除失敗的任務，可使用 `horizon:forget` 指令。`horizon:forget` 指令可接受失敗任務的 ID 或 UUID 作為其唯一的引數：

```shell
php artisan horizon:forget 5
```

若想刪除所有失敗的任務，可在 `horizon:forget` 指令後加上 `--all` 選項：

```shell
php artisan horizon:forget --all
```


<a name="clearing-jobs-from-queues"></a>
## 從佇列中清除任務

若想從應用程式的預設佇列中刪除所有任務，可使用 `horizon:clear` Artisan 指令：

```shell
php artisan horizon:clear
```

我們也可以提供 `queue` 選項來從指定的佇列中刪除任務：

```shell
php artisan horizon:clear --queue=emails
```