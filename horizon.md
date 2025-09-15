# Laravel Horizon

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [平衡策略](#balancing-strategies)
    - [儀表板授權](#dashboard-authorization)
    - [靜默任務](#silenced-jobs)
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
> 在深入了解 Laravel Horizon 之前，您應該先熟悉 Laravel 的基礎 [佇列服務](/docs/{{version}}/queues)。Horizon 增強了 Laravel 的佇列功能，如果您不熟悉 Laravel 提供的基本佇列功能，可能會感到困惑。

[Laravel Horizon](https://github.com/laravel/horizon) 為您的 Laravel [Redis 佇列](/docs/{{version}}/queues) 提供了美觀的儀表板和程式碼驅動的設定。Horizon 讓您可以輕鬆監控佇列系統的關鍵指標，例如任務吞吐量、執行時間和任務失敗情況。

使用 Horizon 時，所有佇列 Worker 的設定都儲存在一個簡單的設定檔中。透過在版本控制的檔案中定義應用程式的 Worker 設定，您可以在部署應用程式時輕鬆擴展或修改應用程式的佇列 Worker。

<img src="https://laravel.com/img/docs/horizon-example.png">

<a name="installation"></a>
## 安裝

> [!WARNING]
> Laravel Horizon 要求您使用 [Redis](https://redis.io) 來驅動您的佇列。因此，您應該確保在應用程式的 `config/queue.php` 設定檔中，將佇列連線設定為 `redis`。

您可以使用 Composer 套件管理器將 Horizon 安裝到您的專案中：

```shell
composer require laravel/horizon
```

安裝 Horizon 後，使用 `horizon:install` Artisan 命令發布其資產：

```shell
php artisan horizon:install
```

<a name="configuration"></a>
### 設定

發布 Horizon 的資產後，其主要設定檔將位於 `config/horizon.php`。此設定檔允許您設定應用程式的佇列 Worker 選項。每個設定選項都包含其用途的描述，因此請務必徹底探索此檔案。

> [!WARNING]
> Horizon 內部使用名為 `horizon` 的 Redis 連線。此 Redis 連線名稱是保留的，不應在 `database.php` 設定檔中或作為 `horizon.php` 設定檔中 `use` 選項的值分配給另一個 Redis 連線。

<a name="environments"></a>
#### 環境

安裝後，您應該熟悉的主要 Horizon 設定選項是 `environments` 設定選項。此設定選項是應用程式執行的環境陣列，並定義每個環境的 Worker 處理程序選項。預設情況下，此項目包含 `production` 和 `local` 環境。但是，您可以根據需要自由添加更多環境：

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

您也可以定義一個萬用字元環境 (`*`)，當找不到其他匹配的環境時將使用它：

    'environments' => [
        // ...

        '*' => [
            'supervisor-1' => [
                'maxProcesses' => 3,
            ],
        ],
    ],

當您啟動 Horizon 時，它將使用應用程式執行環境的 Worker 處理程序設定選項。通常，環境由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#determining-the-current-environment) 的值決定。例如，預設的 `local` Horizon 環境設定為啟動三個 Worker 處理程序，並自動平衡分配給每個佇列的 Worker 處理程序數量。預設的 `production` 環境設定為啟動最多 10 個 Worker 處理程序，並自動平衡分配給每個佇列的 Worker 處理程序數量。

> [!WARNING]
> 您應該確保 `horizon` 設定檔的 `environments` 部分包含您計劃執行 Horizon 的每個 [環境](/docs/{{version}}/configuration#environment-configuration) 的項目。

<a name="supervisors"></a>
#### Supervisor

如您在 Horizon 的預設設定檔中所見，每個環境可以包含一個或多個「Supervisor」。預設情況下，設定檔將此 Supervisor 定義為 `supervisor-1`；但是，您可以自由地為您的 Supervisor 命名。每個 Supervisor 本質上負責「監督」一組 Worker 處理程序，並負責平衡佇列之間的 Worker 處理程序。

如果您想定義一組新的 Worker 處理程序，可以在給定環境中添加額外的 Supervisor。如果您想為應用程式使用的給定佇列定義不同的平衡策略或 Worker 處理程序計數，您可以選擇這樣做。

<a name="maintenance-mode"></a>
#### 維護模式

當您的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，除非在 Horizon 設定檔中將 Supervisor 的 `force` 選項定義為 `true`，否則佇列中的任務將不會由 Horizon 處理：

    'environments' => [
        'production' => [
            'supervisor-1' => [
                // ...
                'force' => true,
            ],
        ],
    ],

<a name="default-values"></a>
#### 預設值

在 Horizon 的預設設定檔中，您會注意到一個 `defaults` 設定選項。此設定選項指定了應用程式 [Supervisor](#supervisors) 的預設值。Supervisor 的預設設定值將合併到每個環境的 Supervisor 設定中，讓您在定義 Supervisor 時避免不必要的重複。

<a name="balancing-strategies"></a>
### 平衡策略

與 Laravel 的預設佇列系統不同，Horizon 允許您從三種 Worker 平衡策略中選擇：`simple`、`auto` 和 `false`。`simple` 策略將傳入的任務平均分配給 Worker 處理程序：

    'balance' => 'simple',

`auto` 策略是設定檔的預設值，它根據佇列的當前工作負載調整每個佇列的 Worker 處理程序數量。例如，如果您的 `notifications` 佇列有 1,000 個待處理任務，而您的 `render` 佇列為空，Horizon 將為您的 `notifications` 佇列分配更多 Worker，直到佇列為空。

使用 `auto` 策略時，您可以定義 `minProcesses` 和 `maxProcesses` 設定選項，以控制每個佇列的最小處理程序數量以及 Horizon 應該擴展和縮減的 Worker 處理程序總數：

    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default'],
                'balance' => 'auto',
                'autoScalingStrategy' => 'time',
                'minProcesses' => 1,
                'maxProcesses' => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'tries' => 3,
            ],
        ],
    ],

`autoScalingStrategy` 設定值決定了 Horizon 是否會根據清除佇列所需的總時間（`time` 策略）或佇列中的任務總數（`size` 策略）來為佇列分配更多 Worker 處理程序。

`balanceMaxShift` 和 `balanceCooldown` 設定值決定了 Horizon 擴展以滿足 Worker 需求的快慢。在上面的範例中，每三秒最多會建立或銷毀一個新處理程序。您可以根據應用程式的需求自由調整這些值。

當 `balance` 選項設定為 `false` 時，將使用預設的 Laravel 行為，其中佇列按照它們在設定中列出的順序進行處理。

<a name="dashboard-authorization"></a>
### 儀表板授權

Horizon 儀表板可以透過 `/horizon` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板。但是，在您的 `app/Providers/HorizonServiceProvider.php` 檔案中，有一個 [授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在 **非 local** 環境中對 Horizon 的存取。您可以根據需要自由修改此 Gate，以限制對 Horizon 安裝的存取：

    /**
     * Register the Horizon gate.
     *
     * This gate determines who can access Horizon in non-local environments.
     */
    protected function gate(): void
    {
        Gate::define('viewHorizon', function (User $user) {
            return in_array($user->email, [
                'taylor @laravel.com',
            ]);
        });
    }

<a name="alternative-authentication-strategies"></a>
#### 替代驗證策略

請記住，Laravel 會自動將已驗證的使用者注入到 Gate 閉包中。如果您的應用程式透過其他方法（例如 IP 限制）提供 Horizon 安全性，那麼您的 Horizon 使用者可能不需要「登入」。因此，您需要將上述 `function (User $user)` 閉包簽章更改為 `function (User $user = null)`，以強制 Laravel 不要求驗證。

<a name="silenced-jobs"></a>
### 靜默任務

有時，您可能不希望查看應用程式或第三方套件分派的某些任務。您可以將這些任務靜默，而不是讓它們佔用「已完成任務」列表中的空間。首先，將任務的類別名稱添加到應用程式 `horizon` 設定檔中的 `silenced` 設定選項：

    'silenced' => [
        App\Jobs\ProcessPodcast::class,
    ],

或者，您希望靜默的任務可以實作 `Laravel\Horizon\Contracts\Silenced` 介面。如果任務實作此介面，即使它不存在於 `silenced` 設定陣列中，它也會自動靜默：

    use Laravel\Horizon\Contracts\Silenced;

    class ProcessPodcast implements ShouldQueue, Silenced
    {
        use Queueable;

        // ...
    }

<a name="upgrading-horizon"></a>
## 升級 Horizon

升級到 Horizon 的新主要版本時，務必仔細查閱 [升級指南](https://github.com/laravel/horizon/blob/master/UPGRADE.md)。

<a name="running-horizon"></a>
## 執行 Horizon

一旦您在應用程式的 `config/horizon.php` 設定檔中設定了 Supervisor 和 Worker，您就可以使用 `horizon` Artisan 命令啟動 Horizon。此單一命令將啟動當前環境的所有已設定 Worker 處理程序：

```shell
php artisan horizon
```

您可以使用 `horizon:pause` 和 `horizon:continue` Artisan 命令暫停 Horizon 處理程序並指示它繼續處理任務：

```shell
php artisan horizon:pause

php artisan horizon:continue
```

您還可以使用 `horizon:pause-supervisor` 和 `horizon:continue-supervisor` Artisan 命令暫停和繼續特定的 Horizon [Supervisor](#supervisors)：

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

您可以使用 `horizon:status` Artisan 命令檢查 Horizon 處理程序的當前狀態：

```shell
php artisan horizon:status
```

您可以使用 `horizon:supervisor-status` Artisan 命令檢查特定 Horizon [Supervisor](#supervisors) 的當前狀態：

```shell
php artisan horizon:supervisor-status supervisor-1
```

您可以使用 `horizon:terminate` Artisan 命令優雅地終止 Horizon 處理程序。任何當前正在處理的任務都將完成，然後 Horizon 將停止執行：

```shell
php artisan horizon:terminate
```

<a name="deploying-horizon"></a>
### 部署 Horizon

當您準備好將 Horizon 部署到應用程式的實際伺服器時，您應該設定一個處理程序監控器來監控 `php artisan horizon` 命令，並在它意外退出時重新啟動它。別擔心，我們將在下面討論如何安裝處理程序監控器。

在應用程式的部署過程中，您應該指示 Horizon 處理程序終止，以便它將由您的處理程序監控器重新啟動並接收您的程式碼更改：

```shell
php artisan horizon:terminate
```

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是一個適用於 Linux 作業系統的處理程序監控器，如果您的 `horizon` 處理程序停止執行，它將自動重新啟動。要在 Ubuntu 上安裝 Supervisor，您可以使用以下命令。如果您沒有使用 Ubuntu，您可能可以使用作業系統的套件管理器安裝 Supervisor：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定 Supervisor 聽起來很麻煩，請考慮使用 [Laravel Forge](https://forge.laravel.com)，它將自動為您的 Laravel 專案安裝和設定 Supervisor。

<a name="supervisor-configuration"></a>
#### Supervisor 設定

Supervisor 設定檔通常儲存在伺服器的 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的設定檔，這些設定檔指示 Supervisor 如何監控您的處理程序。例如，讓我們建立一個 `horizon.conf` 檔案，它啟動並監控一個 `horizon` 處理程序：

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

定義 Supervisor 設定時，您應該確保 `stopwaitsecs` 的值大於您執行時間最長的任務所消耗的秒數。否則，Supervisor 可能會在任務完成處理之前終止任務。

> [!WARNING]
> 雖然上面的範例適用於基於 Ubuntu 的伺服器，但 Supervisor 設定檔的位置和預期的副檔名可能因其他伺服器作業系統而異。請查閱您的伺服器文件以獲取更多資訊。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

建立設定檔後，您可以使用以下命令更新 Supervisor 設定並啟動受監控的處理程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> [!NOTE]
> 有關執行 Supervisor 的更多資訊，請查閱 [Supervisor 文件](http://supervisord.org/index.html)。

<a name="tags"></a>
## 標籤

Horizon 允許您為任務分配「標籤」，包括可郵寄物件 (mailables)、廣播事件 (broadcast events)、通知 (notifications) 和佇列事件監聽器 (queued event listeners)。事實上，Horizon 會根據附加到任務的 Eloquent 模型智慧地自動標記大多數任務。例如，請看以下任務：

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

如果此任務與 `id` 屬性為 `1` 的 `App\Models\Video` 實例一起排入佇列，它將自動收到標籤 `App\Models\Video:1`。這是因為 Horizon 會在任務的屬性中搜尋任何 Eloquent 模型。如果找到 Eloquent 模型，Horizon 將使用模型的類別名稱和主鍵智慧地標記任務：

    use App\Jobs\RenderVideo;
    use App\Models\Video;

    $video = Video::find(1);

    RenderVideo::dispatch($video);

<a name="manually-tagging-jobs"></a>
#### 手動標記任務

如果您想手動定義其中一個可佇列物件的標籤，您可以在類別上定義一個 `tags` 方法：

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

<a name="manually-tagging-event-listeners"></a>
#### 手動標記事件監聽器

當擷取佇列事件監聽器的標籤時，Horizon 會自動將事件實例傳遞給 `tags` 方法，讓您可以將事件資料添加到標籤中：

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

<a name="notifications"></a>
## 通知

> [!WARNING]
> 設定 Horizon 發送 Slack 或 SMS 通知時，您應該查閱 [相關通知通道的先決條件](/docs/{{version}}/notifications)。

如果您希望在其中一個佇列等待時間過長時收到通知，您可以使用 `Horizon::routeMailNotificationsTo`、`Horizon::routeSlackNotificationsTo` 和 `Horizon::routeSmsNotificationsTo` 方法。您可以在應用程式 `App\Providers\HorizonServiceProvider` 的 `boot` 方法中呼叫這些方法：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        parent::boot();

        Horizon::routeSmsNotificationsTo('15556667777');
        Horizon::routeMailNotificationsTo('example @example.com');
        Horizon::routeSlackNotificationsTo('slack-webhook-url', '#channel');
    }

<a name="configuring-notification-wait-time-thresholds"></a>
#### 設定通知等待時間閾值

您可以在應用程式的 `config/horizon.php` 設定檔中設定多少秒被視為「長時間等待」。此檔案中的 `waits` 設定選項允許您控制每個連線/佇列組合的長時間等待閾值。任何未定義的連線/佇列組合將預設為 60 秒的長時間等待閾值：

    'waits' => [
        'redis:critical' => 30,
        'redis:default' => 60,
        'redis:batch' => 120,
    ],

<a name="metrics"></a>
## 指標

Horizon 包含一個指標儀表板，提供有關您的任務和佇列等待時間以及吞吐量的資訊。為了填充此儀表板，您應該設定 Horizon 的 `snapshot` Artisan 命令在應用程式的 `routes/console.php` 檔案中每五分鐘執行一次：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('horizon:snapshot')->everyFiveMinutes();

<a name="deleting-failed-jobs"></a>
## 刪除失敗的任務

如果您想刪除失敗的任務，您可以使用 `horizon:forget` 命令。`horizon:forget` 命令接受失敗任務的 ID 或 UUID 作為其唯一參數：

```shell
php artisan horizon:forget 5
```

如果您想刪除所有失敗的任務，您可以向 `horizon:forget` 命令提供 `--all` 選項：

```shell
php artisan horizon:forget --all
```

<a name="clearing-jobs-from-queues"></a>
## 清除佇列中的任務

如果您想從應用程式的預設佇列中刪除所有任務，您可以使用 `horizon:clear` Artisan 命令：

```shell
php artisan horizon:clear
```

您可以提供 `queue` 選項以從特定佇列中刪除任務：

```shell
php artisan horizon:clear --queue=emails
```
