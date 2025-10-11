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
- [從佇列清除任務](#clearing-jobs-from-queues)

<a name="introduction"></a>
## 簡介

> [!NOTE]  
> 在深入了解 Laravel Horizon 之前，您應該先熟悉 Laravel 的基礎[佇列服務](/docs/{{version}}/queues)。如果您還不熟悉 Laravel 提供的基本佇列功能，Horizon 對 Laravel 佇列的增強功能可能會讓您感到困惑。

[Laravel Horizon](https://github.com/laravel/horizon) 為您的 Laravel 驅動 [Redis 佇列](/docs/{{version}}/queues)提供美觀的儀表板和程式碼驅動的設定。Horizon 讓您能夠輕鬆監控佇列系統的關鍵指標，例如任務吞吐量、執行時間和任務失敗。

使用 Horizon 時，您所有的佇列工作者設定都儲存於單一、簡單的設定檔中。透過將您應用程式的工作者設定定義於版本控制的檔案中，您可以在部署應用程式時輕鬆擴展或修改您應用程式的佇列工作者。

<img src="https://laravel.com/img/docs/horizon-example.png">

<a name="installation"></a>
## 安裝

> [!WARNING]  
> Laravel Horizon 需要使用 [Redis](https://redis.io) 來驅動你的佇列。因此，你應該確保在應用程式的 `config/queue.php` 設定檔中，佇列連線已設為 `redis`。

你可以使用 Composer 套件管理工具將 Horizon 安裝到你的專案中：

```shell
composer require laravel/horizon
```

安裝 Horizon 後，可以使用 `horizon:install` Artisan 指令來發佈其資產：

```shell
php artisan horizon:install
```


<a name="configuration"></a>
### 設定

發佈 Horizon 的資產後，其主要設定檔將位於 `config/horizon.php`。此設定檔允許你為應用程式設定佇列 Worker 選項。每個設定選項都包含其用途的說明，因此務必詳細探索此檔案。

> [!WARNING]  
> Horizon 在內部使用一個名為 `horizon` 的 Redis 連線。此 Redis 連線名稱是保留的，不應在 `database.php` 設定檔中將其指派給另一個 Redis 連線，或作為 `horizon.php` 設定檔中 `use` 選項的值。


<a name="environments"></a>
#### 環境

安裝後，你應該熟悉的主要 Horizon 設定選項是 `environments` 設定選項。此設定選項是一個應用程式運行環境的陣列，並定義了每個環境的 worker 處理程序選項。預設情況下，此項目包含 `production` 和 `local` 環境。但是，你可以根據需要自由添加更多環境：

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

你也可以定義一個萬用字元環境 (`*`)，當找不到其他匹配的環境時將會使用它：

    'environments' => [
        // ...

        '*' => [
            'supervisor-1' => [
                'maxProcesses' => 3,
            ],
        ],
    ],

當你啟動 Horizon 時，它將使用應用程式當前運行環境的 worker 處理程序設定選項。通常，環境是由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#determining-the-current-environment) 的值來決定的。例如，預設的 `local` Horizon 環境配置為啟動三個 worker 處理程序，並自動平衡分配給每個佇列的 worker 處理程序數量。預設的 `production` 環境配置為啟動最多 10 個 worker 處理程序，並自動平衡分配給每個佇列的 worker 處理程序數量。

> [!WARNING]  
> 你應該確保 `horizon` 設定檔中的 `environments` 部分包含你打算運行 Horizon 的每個 [環境](/docs/{{version}}/configuration#environment-configuration) 的條目。


<a name="supervisors"></a>
#### 監督器

如同你在 Horizon 預設設定檔中看到的，每個環境都可以包含一個或多個「監督器」。預設情況下，設定檔將此監督器定義為 `supervisor-1`；然而，你可以隨意命名你的監督器。每個監督器本質上負責「監督」一組 worker 處理程序，並負責平衡佇列中的 worker 處理程序。

如果你希望定義一組新的 worker 處理程序在特定環境中運行，你可以為該環境添加額外的監督器。如果你希望為應用程式使用的特定佇列定義不同的平衡策略或 worker 處理程序數量，你可以選擇這樣做。


<a name="maintenance-mode"></a>
#### 維護模式

當你的應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，佇列中的任務將不會被 Horizon 處理，除非在 Horizon 設定檔中將監督器的 `force` 選項定義為 `true`：

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

在 Horizon 的預設設定檔中，你會注意到一個 `defaults` 設定選項。此設定選項指定了應用程式 [監督器](#supervisors) 的預設值。監督器的預設設定值將合併到每個環境的監督器設定中，讓你可以在定義監督器時避免不必要的重複。


<a name="balancing-strategies"></a>
### 平衡策略

與 Laravel 的預設佇列系統不同，Horizon 允許你選擇三種 worker 平衡策略：`simple`、`auto` 和 `false`。`simple` 策略將傳入的任務平均分配給 worker 處理程序：

    'balance' => 'simple',

`auto` 策略是設定檔的預設值，它根據佇列的當前工作負載調整每個佇列的 worker 處理程序數量。例如，如果你的 `notifications` 佇列有 1,000 個待處理任務，而你的 `render` 佇列為空，Horizon 將為你的 `notifications` 佇列分配更多 worker，直到該佇列為空。

當使用 `auto` 策略時，你可以定義 `minProcesses` 和 `maxProcesses` 設定選項，以控制每個佇列的最小處理程序數量，以及 Horizon 總共應該擴展和縮減的最大 worker 處理程序數量：

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

`autoScalingStrategy` 設定值決定了 Horizon 是否會根據清除佇列所需的總時間（`time` 策略）或佇列中任務的總數（`size` 策略）來為佇列分配更多的 worker 處理程序。

`balanceMaxShift` 和 `balanceCooldown` 設定值決定了 Horizon 擴展以滿足 worker 需求的快慢。在上面的範例中，每三秒最多會創建或銷毀一個新的處理程序。你可以根據應用程式的需求，視情況調整這些值。

當 `balance` 選項設定為 `false` 時，將會使用預設的 Laravel 行為，佇列將按照你在設定中列出的順序進行處理。

<a name="dashboard-authorization"></a>
### 儀表板授權

Horizon 儀表板可透過 `/horizon` 路由存取。依預設，您只能在 `local` 環境中存取此儀表板。然而，在您的 `app/Providers/HorizonServiceProvider.php` 檔案中，定義了一個 [授權 Gate](/docs/{{version}}/authorization#gates)。此授權 Gate 控制在 **非 local** 環境中對 Horizon 的存取。您可以根據需要修改此 Gate 以限制對您的 Horizon 安裝的存取：

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


<a name="alternative-authentication-strategies"></a>
#### 替代驗證策略

請記住，Laravel 會自動將已驗證的使用者注入到 Gate 閉包中。如果您的應用程式透過其他方法（例如 IP 限制）提供 Horizon 安全性，那麼您的 Horizon 使用者可能不需要「登入」。因此，您需要將上述 `function (User $user)` 閉包簽名更改為 `function (User $user = null)`，以強制 Laravel 不要求驗證。


<a name="silenced-jobs"></a>
### 靜默任務

有時，您可能不希望查看由應用程式或第三方套件分派的某些 jobs。為了不讓這些 jobs 佔用您的「已完成任務」列表空間，您可以將其靜默。要開始使用，請將 job 的類別名稱新增到應用程式 `horizon` 設定檔中的 `silenced` 設定選項：

    'silenced' => [
        App\Jobs\ProcessPodcast::class,
    ],

或者，您希望靜默的 job 可以實作 `Laravel\Horizon\Contracts\Silenced` 介面。如果一個 job 實作了此介面，即使它不存在於 `silenced` 設定陣列中，它也會自動被靜默：

    use Laravel\Horizon\Contracts\Silenced;

    class ProcessPodcast implements ShouldQueue, Silenced
    {
        use Queueable;

        // ...
    }

<a name="upgrading-horizon"></a>
## 升級 Horizon

當升級到 Horizon 的新主要版本時，請務必仔細查閱 [升級指南](https://github.com/laravel/horizon/blob/master/UPGRADE.md)。


<a name="running-horizon"></a>
## 執行 Horizon

在您的應用程式的 `config/horizon.php` 設定檔中配置好 supervisors 和 workers 後，您可以使用 `horizon` Artisan 指令啟動 Horizon。此單一指令將啟動目前環境中所有已配置的 worker 處理程序：

```shell
php artisan horizon
```

您可以使用 `horizon:pause` 和 `horizon:continue` Artisan 指令來暫停 Horizon 處理程序並指示其繼續處理任務：

```shell
php artisan horizon:pause

php artisan horizon:continue
```

您也可以使用 `horizon:pause-supervisor` 和 `horizon:continue-supervisor` Artisan 指令來暫停和繼續特定的 Horizon [supervisors](#supervisors)：

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

您可以使用 `horizon:status` Artisan 指令檢查 Horizon 處理程序的目前狀態：

```shell
php artisan horizon:status
```

您可以使用 `horizon:supervisor-status` Artisan 指令檢查特定 Horizon [supervisor](#supervisors) 的目前狀態：

```shell
php artisan horizon:supervisor-status supervisor-1
```

您可以使用 `horizon:terminate` Artisan 指令優雅地終止 Horizon 處理程序。任何正在處理的任務都將完成，然後 Horizon 將停止執行：

```shell
php artisan horizon:terminate
```


<a name="deploying-horizon"></a>
### 部署 Horizon

當您準備將 Horizon 部署到應用程式的實際伺服器時，您應該配置一個處理程序監控器來監控 `php artisan horizon` 指令，並在它意外終止時重新啟動。別擔心，我們將在下面討論如何安裝處理程序監控器。

在應用程式的部署過程中，您應該指示 Horizon 處理程序終止，以便它能被處理程序監控器重新啟動並接收您的程式碼變更：

```shell
php artisan horizon:terminate
```


<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是一個用於 Linux 作業系統的處理程序監控器，如果您的 `horizon` 處理程序停止執行，它將自動重新啟動。要在 Ubuntu 上安裝 Supervisor，您可以使用以下指令。如果您未使用 Ubuntu，您很可能可以使用作業系統的套件管理器安裝 Supervisor：

```shell
sudo apt-get install supervisor
```

> [!NOTE]  
> 如果自行配置 Supervisor 聽起來很複雜，請考慮使用 [Laravel Forge](https://forge.laravel.com)，它將自動為您的 Laravel 專案安裝和配置 Supervisor。


<a name="supervisor-configuration"></a>
#### Supervisor 配置

Supervisor 配置檔通常儲存在您伺服器的 `/etc/supervisor/conf.d` 目錄中。在此目錄中，您可以建立任意數量的配置檔，指示 supervisor 如何監控您的處理程序。例如，讓我們建立一個 `horizon.conf` 檔，用於啟動和監控 `horizon` 處理程序：

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

定義 Supervisor 配置時，您應確保 `stopwaitsecs` 的值大於您執行最長的任務所花費的秒數。否則，Supervisor 可能會在任務完成處理之前終止該任務。

> [!WARNING]  
> 雖然上述範例適用於基於 Ubuntu 的伺服器，但 Supervisor 配置檔的預期位置和副檔名可能因其他伺服器作業系統而異。請查閱您伺服器的文件以獲取更多資訊。


<a name="starting-supervisor"></a>
#### 啟動 Supervisor

配置檔建立後，您可以使用以下指令更新 Supervisor 配置並啟動受監控的處理程序：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> [!NOTE]  
> 有關執行 Supervisor 的更多資訊，請查閱 [Supervisor 文件](http://supervisord.org/index.html)。


<a name="tags"></a>
## 標籤

Horizon 允許您為任務分配「標籤」，包括 mailables、廣播事件、通知和佇列事件監聽器。實際上，Horizon 將根據附加到任務的 Eloquent model 智慧地自動為大多數任務打上標籤。例如，請看以下任務：

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

如果此任務與一個 `id` 屬性為 `1` 的 `App\Models\Video` 實例一起入佇列，它將自動收到標籤 `App\Models\Video:1`。這是因為 Horizon 會搜尋任務的屬性以查找任何 Eloquent model。如果找到 Eloquent model，Horizon 將使用 model 的類別名稱和主鍵智慧地為任務打上標籤：

    use App\Jobs\RenderVideo;
    use App\Models\Video;

    $video = Video::find(1);

    RenderVideo::dispatch($video);


<a name="manually-tagging-jobs"></a>
#### 手動為任務打上標籤

如果您想手動定義您的其中一個可佇列物件的標籤，您可以在該類別上定義一個 `tags` 方法：

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
#### 手動為事件監聽器打上標籤

當為佇列事件監聽器檢索標籤時，Horizon 將自動把事件實例傳遞給 `tags` 方法，讓您能將事件資料添加到標籤中：

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
> 當設定 Horizon 傳送 Slack 或 SMS 通知時，您應該檢閱[相關通知頻道的前提條件](/docs/{{version}}/notifications)。

如果您希望在其中一個佇列有長時間等待時收到通知，您可以使用 `Horizon::routeMailNotificationsTo`、`Horizon::routeSlackNotificationsTo` 和 `Horizon::routeSmsNotificationsTo` 方法。您可以在應用程式的 `App\Providers\HorizonServiceProvider` 檔案的 `boot` 方法中呼叫這些方法：

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


<a name="configuring-notification-wait-time-thresholds"></a>
#### 設定通知等待時間閾值

您可以在應用程式的 `config/horizon.php` 設定檔中配置多少秒被視為「長時間等待」。此檔案中的 `waits` 設定選項允許您控制每個連線 / 佇列組合的長時間等待閾值。任何未定義的連線 / 佇列組合將預設為 60 秒的長時間等待閾值：

    'waits' => [
        'redis:critical' => 30,
        'redis:default' => 60,
        'redis:batch' => 120,
    ],


<a name="metrics"></a>
## 指標

Horizon 包含一個指標儀表板，提供有關任務和佇列等待時間及吞吐量的資訊。為了填充此儀表板，您應該設定 Horizon 的 `snapshot` Artisan 指令，使其在應用程式的 `routes/console.php` 檔案中每五分鐘執行一次：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('horizon:snapshot')->everyFiveMinutes();


<a name="deleting-failed-jobs"></a>
## 刪除失敗的任務

如果您想刪除失敗的任務，可以使用 `horizon:forget` 指令。`horizon:forget` 指令接受失敗任務的 ID 或 UUID 作為其唯一引數：

```shell
php artisan horizon:forget 5
```

如果您想刪除所有失敗的任務，可以向 `horizon:forget` 指令提供 `--all` 選項：

```shell
php artisan horizon:forget --all
```


<a name="clearing-jobs-from-queues"></a>
## 從佇列清除任務

如果您想從應用程式的預設佇列中刪除所有任務，可以使用 `horizon:clear` Artisan 指令：

```shell
php artisan horizon:clear
```

您可以提供 `queue` 選項來刪除特定佇列中的任務：

```shell
php artisan horizon:clear --queue=emails
```