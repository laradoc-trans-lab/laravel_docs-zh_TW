# Laravel Horizon

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [儀表板授權](#dashboard-authorization)
    - [任務最大嘗試次數](#max-job-attempts)
    - [任務逾時](#job-timeout)
    - [任務退避延遲](#job-backoff)
    - [其他 Worker 選項](#other-worker-options)
    - [靜音任務](#silenced-jobs)
- [負載平衡策略](#balancing-strategies)
    - [自動平衡](#auto-balancing)
    - [簡易平衡](#simple-balancing)
    - [無平衡](#no-balancing)
- [升級 Horizon](#upgrading-horizon)
- [執行 Horizon](#running-horizon)
    - [部署 Horizon](#deploying-horizon)
- [標籤](#tags)
- [通知](#notifications)
- [度量指標](#metrics)
- [刪除失敗任務](#deleting-failed-jobs)
- [從佇列清除任務](#clearing-jobs-from-queues)

<a name="introduction"></a>
## 簡介

> [!NOTE]
> 在深入了解 Laravel Horizon 之前，您應該先熟悉 Laravel 的基礎[佇列服務](/docs/{{version}}/queues)。Horizon 為 Laravel 的佇列增強了額外功能，如果您尚未熟悉 Laravel 提供的基本佇列功能，可能會感到困惑。

[Laravel Horizon](https://github.com/laravel/horizon) 為由 Laravel 驅動的 [Redis 佇列](/docs/{{version}}/queues)提供了精美的儀表板與程式碼驅動的設定。Horizon 讓您可以輕鬆監控佇列系統的關鍵指標，例如任務吞吐量、執行時間以及任務失敗狀況。

使用 Horizon 時，所有佇列 worker 的設定都儲存在單一且簡單的設定檔中。透過將應用程式的 worker 設定定義在受版本控制的檔案中，您可以在部署應用程式時輕鬆擴展或修改應用程式的佇列 worker。

<img src="https://laravel.com/img/docs/horizon-example.png">

<a name="installation"></a>
## 安裝

> [!WARNING]
> Laravel Horizon 需要使用 [Redis](https://redis.io) 來驅動您的佇列。因此，您應該確保應用程式的 `config/queue.php` 設定檔中，佇列連線已設定為 `redis`。目前 Horizon 與 Redis Cluster 不相容。

您可以使用 Composer 套件管理器將 Horizon 安裝到專案中：

```shell
composer require laravel/horizon
```

安裝 Horizon 之後，請使用 `horizon:install` Artisan 命令發布其靜態資源：

```shell
php artisan horizon:install
```


<a name="configuration"></a>
### 設定

發布 Horizon 的靜態資源後，其主要設定檔將位於 `config/horizon.php`。此設定檔允許您為應用程式設定佇列 Worker 選項。每個設定選項都包含其用途的說明，因此請務必詳細閱讀此檔案。

> [!WARNING]
> Horizon 在內部使用名為 `horizon` 的 Redis 連線。此 Redis 連線名稱為保留名稱，不應在 `database.php` 設定檔中指派給其他 Redis 連線，也不應作為 `horizon.php` 設定檔中 `use` 選項的值。


<a name="content-security-policy-csp-nonce"></a>
#### 內容安全政策 (CSP) Nonce

如果您想在 Horizon 視圖中使用的 script 與 style 標籤上使用 [nonce 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/nonce) 作為[內容安全政策 (Content Security Policy)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，您可以使用 `Horizon::cspNonce` 方法來指定要使用的 nonce。此方法通常應該在中介層內調用，以便為每個請求指派一個新的 nonce：

```php
use Closure;
use Illuminate\Http\Request;
use Laravel\Horizon\Horizon;
use Symfony\Component\HttpFoundation\Response;

public function handle(Request $request, Closure $next): Response
{
    Horizon::cspNonce('csp-nonce');

    return $next($request);
}
```

您可以將此中介層新增到應用程式的 `config/horizon.php` 設定檔中的 `middleware` 選項：

```php
'middleware' => [
    'web',
    App\Http\Middleware\AddHorizonCspNonce::class,
],
```


<a name="environments"></a>
#### 環境

安裝完成後，您應該熟悉的主要 Horizon 設定選項是 `environments` 設定選項。此設定選項是應用程式所執行環境的陣列，並定義了每個環境的 Worker 行程選項。預設情況下，此項目包含 `production` 與 `local` 環境。不過，您可以根據需要隨意新增更多環境：

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

您也可以定義萬用字元環境 (`*`)，當找不到其他相符的環境時將會使用該環境：

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

當您啟動 Horizon 時，它將會使用應用程式目前執行環境的 Worker 行程設定選項。通常，環境是由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#determining-the-current-environment) 的值決定的。例如，預設的 `local` Horizon 環境設定為啟動三個 Worker 行程，並自動平衡分配給每個佇列的 Worker 行程數量。預設的 `production` 環境設定為最多啟動 10 個 Worker 行程，並自動平衡分配給每個佇列的 Worker 行程數量。

> [!WARNING]
> 您應該確保 `horizon` 設定檔中的 `environments` 部分包含您打算在其上執行 Horizon 的每個[環境](/docs/{{version}}/configuration#environment-configuration)項目。


<a name="supervisors"></a>
#### Supervisors

正如您在 Horizon 預設設定檔中所看到的，每個環境可以包含一個或多個「Supervisor」。預設情況下，設定檔將此 Supervisor 定義為 `supervisor-1`；然而，您可以隨意為 Supervisor 命名。每個 Supervisor 本質上負責「監督」一組 Worker 行程，並負責跨佇列平衡 Worker 行程。

如果您想定義在該環境中執行的新 Worker 行程群組，可以為給定環境新增額外的 Supervisor。如果您想為應用程式使用的特定佇列定義不同的平衡策略或 Worker 行程數量，可以選擇這樣做。


<a name="maintenance-mode"></a>
#### 維護模式

當您的應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，除非在 Horizon 設定檔中將 Supervisor 的 `force` 選項定義為 `true`，否則 Horizon 將不會處理佇列中的任務：

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

Horizon 儀表板可透過 `/horizon` 路由存取。預設情況下，您只能在 `local` 環境中存取此儀表板。然而，在您的 `app/Providers/HorizonServiceProvider.php` 檔案中，有一個[授權 Gate](/docs/{{version}}/authorization#gates) 定義。此授權 Gate 控制在**非本地 (non-local)** 環境中對 Horizon 的存取。您可以根據需要隨意修改此 Gate，以限制對 Horizon 安裝的存取：

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
#### 其他認證策略

請記住，Laravel 會自動將已認證的使用者注入到 Gate 閉包中。如果您的應用程式透過其他方式（例如 IP 限制）提供 Horizon 安全防護，則您的 Horizon 使用者可能不需要「登入」。因此，您需要將上述的 `function (User $user)` 閉包簽名更改為 `function (User $user = null)`，以強制 Laravel 不需要認證。

<a name="max-job-attempts"></a>
### 任務最大嘗試次數

> [!NOTE]
> 在調整這些選項之前，請確保您已熟悉 Laravel 預設的[佇列服務](/docs/{{version}}/queues#max-job-attempts-and-timeout)以及「嘗試次數 (attempts)」的概念。

您可以在 supervisor 的設定中定義任務可消耗的最大嘗試次數：

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

當使用如 `WithoutOverlapping` 或 `RateLimited` 等中介層時，調整 `tries` 選項至關重要，因為它們會消耗嘗試次數。若要處理此情況，您可以在 supervisor 層級調整 `tries` 設定值，或者在任務類別上定義 `$tries` 屬性。

若未設定 `tries` 選項，Horizon 預設為單次嘗試，除非任務類別定義了 `$tries`，其優先級高於 Horizon 設定。

將 `tries` 或 `$tries` 設定為 0 允許無限次嘗試，這在嘗試次數不確定時非常理想。為防止無休止的失敗，您可以透過在任務類別上設定 `$maxExceptions` 屬性來限制允許的例外狀況數量。


<a name="job-timeout"></a>
### 任務逾時

同樣地，您可以在 supervisor 層級設定 `timeout` 值，該值指定 Worker 行程在被強制終止之前可以執行任務的秒數。一旦終止，任務將根據您的佇列設定重試或標記為失敗：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'timeout' => 60,
        ],
    ],
],
```

> [!WARNING]
> 當使用 `auto` 平衡策略時，Horizon 會將正在進行中的 Worker 視為「停滯 (hanging)」，並在縮減規模期間於 Horizon 逾時後強制終止它們。請務必確保 Horizon 的逾時時間大於任何任務層級的逾時時間，否則任務可能會在執行途中被終止。此外，`timeout` 值應始終比 `config/queue.php` 設定檔中定義的 `retry_after` 值至少少幾秒鐘。否則，您的任務可能會被重複處理。


<a name="job-backoff"></a>
### 任務退避延遲

您可以在 supervisor 層級定義 `backoff` 值，以指定 Horizon 在重試遇到未處理例外的任務之前應等待的時間：

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

您也可以透過將陣列指定給 `backoff` 值來設定「指數」退避。在此範例中，第一次重試的重試延遲為 1 秒，第二次重試為 5 秒，第三次重試為 10 秒，若還有剩餘的嘗試次數，則後續每次重試皆為 10 秒：

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


<a name="other-worker-options"></a>
### 其他 Worker 選項

除了 `tries`、`timeout` 和 `backoff` 之外，每個 supervisor 還接受其他幾個選項，用於控制其 Worker 行程的行為以及自動重啟的時機。定期重啟 Worker 是長時間運行行程的良好實踐，因為這有助於防止記憶體流失：

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            // ...
            'memory' => 128,
            'maxJobs' => 1000,
            'maxTime' => 3600,
            'sleep' => 3,
            'rest' => 0,
            'nice' => 0,
        ],
    ],
],
```

<div class="content-list" markdown="1">

- `memory` 定義單個 Worker 行程在重啟之前可消耗的最大記憶體量（以 MB 為單位）。預設情況下，此值為 `128`。
- `maxJobs` 定義 Worker 在重啟前應處理的任務數量。值為 `0` 表示不應根據處理的任務數量重啟 Worker。預設情況下，此值為 `0`。
- `maxTime` 定義 Worker 在重啟前應運行的秒數。值為 `0` 表示不應根據時間重啟 Worker。預設情況下，此值為 `0`。
- `sleep` 定義當沒有可用任務時，Worker 在重新輪詢佇列以獲取新任務之前應等待的秒數。預設情況下，此值為 `3`。
- `rest` 定義在處理每個任務之間暫停的秒數。預設情況下，此值為 `0`。
- `nice` 定義 Worker 行程的「niceness」（排程優先權）。較高的值會賦予行程較低的優先權。預設情況下，此值為 `0`。

</div>


<a name="silenced-jobs"></a>
### 靜音任務

有時，您可能不想查看由應用程式或第三方套件分派的某些任務。為了避免這些任務佔用「Completed Jobs」列表中的空間，您可以將它們靜音。首先，將任務的類別名稱新增至應用程式 `horizon` 設定檔中的 `silenced` 設定選項：

```php
'silenced' => [
    App\Jobs\ProcessPodcast::class,
],
```

除了靜音單個任務類別外，Horizon 還支援根據[標籤](#tags)靜音任務。若您想隱藏共享同一標籤的多個任務，這會非常有用：

```php
'silenced_tags' => [
    'notifications'
],
```

或者，您希望靜音的任務可以實作 `Laravel\Horizon\Contracts\Silenced` 介面。若任務實作了此介面，即使它不存在於 `silenced` 設定陣列中，它也會自動被靜音：

```php
use Laravel\Horizon\Contracts\Silenced;

class ProcessPodcast implements ShouldQueue, Silenced
{
    use Queueable;

    // ...
}
```

<a name="balancing-strategies"></a>
## 負載平衡策略

每個 supervisor 可以處理一個或多個佇列，但與 Laravel 預設的佇列系統不同的是，Horizon 允許你從三種 worker 負載平衡策略中進行選擇：`auto`、`simple` 和 `false`。

<a name="auto-balancing"></a>
### 自動平衡

`auto` 策略是預設的策略，它會根據佇列當前的工作負載來調整每個佇列的 worker 行程數量。例如，如果你的 `notifications` 佇列有 1,000 個待處理任務，而 `default` 佇列是空的，Horizon 將會為你的 `notifications` 佇列分配更多的 worker，直到該佇列清空為止。

使用 `auto` 策略時，你還可以設定 `minProcesses` 與 `maxProcesses` 設定選項：

<div class="content-list" markdown="1">

- `minProcesses` 定義每個佇列的最小 worker 行程數量。此值必須大於或等於 1。
- `maxProcesses` 定義 Horizon 在所有佇列中最多可擴展的 worker 行程總數。此值通常應大於佇列數量乘以 `minProcesses` 的值。若要防止 supervisor 產生任何行程，你可以將此值設為 0。

</div>

例如，你可以設定 Horizon 讓每個佇列至少維持一個行程，並最多擴展到總共 10 個 worker 行程：

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

`autoScalingStrategy` 設定選項決定了 Horizon 如何為佇列分配更多的 worker 行程。你可以從兩種策略中進行選擇：

<div class="content-list" markdown="1">

- `time` 策略將根據清空佇列預估所需的總時間來分配 worker。
- `size` 策略將根據佇列中的任務總數來分配 worker。

</div>

`balanceMaxShift` 和 `balanceCooldown` 設定值決定了 Horizon 擴展以滿足 worker 需求的速度。在上述範例中，每三秒最多會建立或銷毀一個新行程。你可以根據應用程式的需求隨意調整這些數值。

<a name="auto-queue-priorities"></a>
#### 佇列優先級與自動平衡

使用 `auto` 負載平衡策略時，Horizon 不會強制執行佇列之間的嚴格優先級。Supervisor 設定中佇列的順序不會影響 worker 行程的分配方式。相反地，Horizon 依賴所選的 `autoScalingStrategy` 根據佇列負載動態分配 worker 行程。

例如，在以下設定中，儘管 high 佇列排在清單的第一位，但它並不會優先於 default 佇列：

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

如果你需要強制設定佇列之間的相對優先級，可以定義多個 supervisor 並明確分配處理資源：

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

在此範例中，default `queue` 最多可以擴展至 10 個行程，而 `images` 佇列則限制為一個行程。此設定可確保你的佇列能夠獨立擴展。

> [!NOTE]
> 分派高度消耗資源的任務時，有時最好將它們分配給具有限制 `maxProcesses` 值的專用佇列。否則，這些任務可能會消耗過多的 CPU 資源並使你的系統不堪重負。

<a name="simple-balancing"></a>
### 簡易平衡

`simple` 策略將 worker 行程平均分配到指定的佇列中。在此策略下，Horizon 不會自動擴展 worker 行程的數量，而是使用固定數量的行程：

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

在上面的範例中，Horizon 將為每個佇列分配 5 個行程，將總數 10 平均分配。

如果你想單獨控制分配給每個佇列的 worker 行程數量，可以定義多個 supervisor：

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

透過此設定，Horizon 將為 `default` 佇列分配 10 個行程，為 `notifications` 佇列分配 2 個行程。

<a name="no-balancing"></a>
### 無平衡

當 `balance` 選項設為 `false` 時，Horizon 會嚴格按照佇列列出的順序處理，類似於 Laravel 預設的佇列系統。不過，如果任務開始累積，它仍然會擴展 worker 行程的數量：

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

在上面的範例中，`default` 佇列中的任務始終優先於 `notifications` 佇列中的任務。例如，如果 `default` 中有 1,000 個任務，而 `notifications` 中只有 10 個任務，Horizon 將會完整處理完所有 `default` 任務，然後才處理來自 `notifications` 的任何任務。

你可以使用 `minProcesses` 和 `maxProcesses` 選項來控制 Horizon 擴展 worker 行程的能力：

<div class="content-list" markdown="1">

- `minProcesses` 定義總共的最小 worker 行程數量。此值必須大於或等於 1。
- `maxProcesses` 定義 Horizon 最多可擴展的 worker 行程總數。

</div>

<a name="upgrading-horizon"></a>
## 升級 Horizon

升級至 Horizon 的新主要版本時，請務必仔細閱讀[升級指南](https://github.com/laravel/horizon/blob/master/UPGRADE.md)。

<a name="running-horizon"></a>
## 執行 Horizon

一旦您在應用程式的 `config/horizon.php` 設定檔中設定好了 Supervisor 與 Worker，就可以使用 `horizon` Artisan 指令來啟動 Horizon。這個單一指令將會為目前的環境啟動所有已設定的 Worker 行程：

```shell
php artisan horizon
```

您可以使用 `horizon:pause` 與 `horizon:continue` Artisan 指令來暫停 Horizon 行程，或指示其繼續處理任務：

```shell
php artisan horizon:pause

php artisan horizon:continue
```

您也可以使用 `horizon:pause-supervisor` 與 `horizon:continue-supervisor` Artisan 指令來暫停與繼續特定的 Horizon [Supervisor](#supervisors)：

```shell
php artisan horizon:pause-supervisor supervisor-1

php artisan horizon:continue-supervisor supervisor-1
```

您可以使用 `horizon:status` Artisan 指令檢查 Horizon 行程的當前狀態：

```shell
php artisan horizon:status
```

您可以使用 `horizon:supervisor-status` Artisan 指令檢查特定 Horizon [Supervisor](#supervisors) 的當前狀態：

```shell
php artisan horizon:supervisor-status supervisor-1
```

您可以使用 `horizon:terminate` Artisan 指令優雅地終止 Horizon 行程。任何目前正在處理中的任務都將會執行完成，隨後 Horizon 才會停止執行：

```shell
php artisan horizon:terminate
```

<a name="automatically-restarting-horizon"></a>
#### 自動重新啟動 Horizon

在本地端開發期間，您可以執行 `horizon:listen` 指令。使用 `horizon:listen` 指令時，當您想要重新載入更新後的程式碼，就不需要手動重新啟動 Horizon。在使用此功能之前，您應確保本地開發環境中已安裝 [Node](https://nodejs.org)。此外，您應該在專案中安裝 [Chokidar](https://github.com/paulmillr/chokidar) 檔案監聽程式庫：

```shell
npm install --save-dev chokidar
```

安裝 Chokidar 後，您可以使用 `horizon:listen` 指令啟動 Horizon：

```shell
php artisan horizon:listen
```

在 Docker 或 Vagrant 中執行時，您應該使用 `--poll` 選項：

```shell
php artisan horizon:listen --poll
```

您可以在應用程式的 `config/horizon.php` 設定檔中使用 `watch` 設定選項來設定應監聽的目錄和檔案：

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

當您準備將 Horizon 部署到應用程式的實際伺服器時，應該設定一個行程監控器來監控 `php artisan horizon` 指令，並在其意外結束時重新啟動它。別擔心，我們將在下方討論如何安裝行程監控器。

在應用程式的部署過程中，您應該指示 Horizon 行程終止，以便讓您的行程監控器將其重新啟動並套用您的程式碼變更：

```shell
php artisan horizon:terminate
```

<a name="installing-supervisor"></a>
#### 安裝 Supervisor

Supervisor 是 Linux 作業系統的行程監控器，如果 `horizon` 行程停止執行，它將自動為您重新啟動。要在 Ubuntu 上安裝 Supervisor，您可以使用以下指令。如果您不是使用 Ubuntu，通常可以使用您作業系統的套件管理器來安裝 Supervisor：

```shell
sudo apt-get install supervisor
```

> [!NOTE]
> 如果自行設定 Supervisor 聽起來過於繁瑣，可以考慮使用 [Laravel Cloud](https://cloud.laravel.com)，它能為您的 Laravel 應用程式管理背景行程。

<a name="supervisor-configuration"></a>
#### Supervisor 設定

Supervisor 設定檔通常儲存在伺服器的 `/etc/supervisor/conf.d` 目錄中。在此目錄下，您可以建立任意數量的設定檔，指示 Supervisor 應如何監控您的行程。例如，讓我們建立一個啟動並監控 `horizon` 行程的 `horizon.conf` 檔案：

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

在定義 Supervisor 設定時，您應該確保 `stopwaitsecs` 的值大於您執行時間最長的任務所消耗的秒數。否則，Supervisor 可能會在任務完成處理之前將其中止。

> [!WARNING]
> 雖然上述範例適用於基於 Ubuntu 的伺服器，但在其他伺服器作業系統上，Supervisor 設定檔所需的位置和副檔名可能會有所不同。請參閱您伺服器的相關說明文件以獲取更多資訊。

<a name="starting-supervisor"></a>
#### 啟動 Supervisor

設定檔建立完成後，您可以使用以下指令更新 Supervisor 設定並啟動受監控的行程：

```shell
sudo supervisorctl reread

sudo supervisorctl update

sudo supervisorctl start horizon
```

> [!NOTE]
> 關於執行 Supervisor 的更多資訊，請參閱 [Supervisor 官方文件](http://supervisord.org/index.html)。

<a name="tags"></a>
## 標籤

Horizon 允許您為任務分配「標籤 (Tags)」，包括郵件物件 (Mailables)、廣播事件、通知以及佇列中的事件監聽器。事實上，Horizon 會根據附加在任務上的 Eloquent 模型，智慧且自動地為大多數任務標記標籤。例如，請看以下任務：

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

如果此任務在佇列中帶有一個 `id` 屬性為 `1` 的 `App\Models\Video` 實例，它將自動獲得 `App\Models\Video:1` 標籤。這是因為 Horizon 會搜尋該任務的屬性中是否存在任何 Eloquent 模型。若找到 Eloquent 模型，Horizon 將會使用該模型的類別名稱與主鍵智慧地標記該任務：

```php
use App\Jobs\RenderVideo;
use App\Models\Video;

$video = Video::find(1);

RenderVideo::dispatch($video);
```

<a name="manually-tagging-jobs"></a>
#### 手動標記任務

如果您想要手動為可排入佇列的物件定義標籤，可以在該類別上定義一個 `tags` 方法：

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

當取得已排入佇列的事件監聽器標籤時，Horizon 會自動將事件實例傳遞給 `tags` 方法，讓您可以將事件資料加入標籤中：

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
> 當設定 Horizon 發送 Slack 或 SMS 通知時，請務必先參閱[相關通知頻道的先決條件](/docs/{{version}}/notifications)。

如果您希望在某個佇列出現長時間等待時收到通知，可以使用 `Horizon::routeMailNotificationsTo`、`Horizon::routeSlackNotificationsTo` 以及 `Horizon::routeSmsNotificationsTo` 方法。您可以在應用程式的 `App\Providers\HorizonServiceProvider` 中的 `boot` 方法呼叫這些方法：

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

您可以在應用程式的 `config/horizon.php` 設定檔中設定多少秒數被視為「長時間等待」。該檔案中的 `waits` 設定選項可讓您控制每個連線／佇列組合的長時間等待閾值。任何未定義的連線／佇列組合將預設採用 60 秒的長時間等待閾值：

```php
'waits' => [
    'redis:critical' => 30,
    'redis:default' => 60,
    'redis:batch' => 120,
],
```

將佇列的閾值設定為 `0` 將會停用該佇列的長時間等待通知。

<a name="metrics"></a>
## 度量指標

Horizon 包含一個度量指標儀表板，提供有關您的任務與佇列等待時間及吞吐量的資訊。為了填入此儀表板的資料，您應該在應用程式的 `routes/console.php` 檔案中設定 Horizon 的 `snapshot` Artisan 指令每五分鐘執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('horizon:snapshot')->everyFiveMinutes();
```

您可以使用應用程式 `config/horizon.php` 設定檔中的 `metrics.trim_snapshots` 選項，來設定 Horizon 的度量指標圖表要保留多少個快照。由於此選項限制的是快照的數量而非保存時間，因此保留期間取決於 `horizon:snapshot` 指令執行的頻率：

```php
'metrics' => [
    'trim_snapshots' => [
        'job' => 24,
        'queue' => 24,
    ],
],
```

如果您想要刪除所有度量指標資料，可以執行 `horizon:clear-metrics` Artisan 指令：

```shell
php artisan horizon:clear-metrics
```

<a name="deleting-failed-jobs"></a>
## 刪除失敗任務

如果您想要刪除失敗的任務，可以使用 `horizon:forget` 指令。`horizon:forget` 指令僅接受失敗任務的 ID 或 UUID 作為其唯一的引數：

```shell
php artisan horizon:forget 5
```

如果您想要刪除所有失敗的任務，可以為 `horizon:forget` 指令提供 `--all` 選項：

```shell
php artisan horizon:forget --all
```

<a name="clearing-jobs-from-queues"></a>
## 從佇列清除任務

如果您想要從應用程式的預設佇列中刪除所有任務，可以使用 `horizon:clear` Artisan 指令：

```shell
php artisan horizon:clear
```

您可以提供 `queue` 選項以刪除指定佇列中的任務：

```shell
php artisan horizon:clear --queue=emails
```