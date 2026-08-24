# 任務排程

- [簡介](#introduction)
- [定義排程](#defining-schedules)
    - [排程 Artisan 指令](#scheduling-artisan-commands)
    - [排程佇列任務](#scheduling-queued-jobs)
    - [排程 Shell 指令](#scheduling-shell-commands)
    - [排程頻率選項](#schedule-frequency-options)
    - [時區](#timezones)
    - [防止任務重複執行](#preventing-task-overlaps)
    - [在單一伺服器上執行任務](#running-tasks-on-one-server)
    - [背景任務](#background-tasks)
    - [維護模式](#maintenance-mode)
    - [暫停排程任務](#pausing-scheduled-tasks)
    - [排程群組](#schedule-groups)
- [執行排程器](#running-the-scheduler)
    - [小於一分鐘的排程任務](#sub-minute-scheduled-tasks)
    - [在本機執行排程器](#running-the-scheduler-locally)
- [任務輸出](#task-output)
- [任務勾點](#task-hooks)
- [事件](#events)

<a name="introduction"></a>
## 簡介

過去，你可能必須在伺服器上為每個需要排程的任務撰寫一個 cron 設定項目。然而，這很快就會變得令人頭痛，因為你的任務排程不再納入版本控制中，而且你必須透過 SSH 登入伺服器才能檢視現有的 cron 項目或新增其他項目。

Laravel 的指令排程器為管理伺服器上的排程任務提供了一種全新的方法。排程器讓你能夠在 Laravel 應用程式本身流暢且具表達力地定義你的指令排程。使用排程器時，你的伺服器上只需要一個單一的 cron 項目。你的任務排程通常定義在應用程式的 `routes/console.php` 檔案中。

<a name="defining-schedules"></a>
## 定義排程

您可以在應用程式的 `routes/console.php` 檔案中定義所有的排程任務。首先，讓我們來看一個範例。在這個範例中，我們將排程一個閉包 (closure) 在每天午夜被呼叫。在該閉包中，我們將執行一個資料庫查詢來清空一張資料表：

```php
<?php

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->daily();
```

除了使用閉包進行排程外，您也可以排程 [可呼叫物件](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke)。可呼叫物件是包含 `__invoke` 方法的簡單 PHP 類別：

```php
Schedule::call(new DeleteRecentUsers)->daily();
```

如果您偏好將 `routes/console.php` 檔案僅保留用於定義指令，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `withSchedule` 方法來定義排程任務。此方法接受一個接收排程器實例的閉包：

```php
use Illuminate\Console\Scheduling\Schedule;

->withSchedule(function (Schedule $schedule) {
    $schedule->call(new DeleteRecentUsers)->daily();
})
```

如果您想檢視排程任務的總覽以及它們下一次預計執行的時間，可以使用 `schedule:list` 這一 Artisan 指令：

```shell
php artisan schedule:list
```

<a name="scheduling-artisan-commands"></a>
### 排程 Artisan 指令

除了排程閉包之外，您還可以排程 [Artisan 指令](/docs/{{version}}/artisan) 和系統指令。例如，您可以使用 `command` 方法，透過指令名稱或類別來排程 Artisan 指令。

當使用指令的類別名稱來排程 Artisan 指令時，您可以傳遞一個額外的命令列引數陣列，這些引數會在指令被調用時提供給它：

```php
use App\Console\Commands\SendEmailsCommand;
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send Taylor --force')->daily();

Schedule::command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();
```

<a name="scheduling-artisan-closure-commands"></a>
#### 排程 Artisan 閉包指令

如果您想要排程由閉包定義的 Artisan 指令，可以在定義指令後串接與排程相關的方法：

```php
Artisan::command('delete:recent-users', function () {
    DB::table('recent_users')->delete();
})->purpose('Delete recent users')->daily();
```

如果您需要向閉包指令傳遞引數，可以將它們提供給 `schedule` 方法：

```php
Artisan::command('emails:send {user} {--force}', function ($user) {
    // ...
})->purpose('Send emails to the specified user')->schedule(['Taylor', '--force'])->daily();
```

<a name="scheduling-queued-jobs"></a>
### 排程佇列任務

`job` 方法可用於排程 [佇列任務 (queued job)](/docs/{{version}}/queues)。此方法提供了一種便捷的方式來排程佇列任務，而不需要使用 `call` 方法來定義將任務推入佇列的閉包：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

Schedule::job(new Heartbeat)->everyFiveMinutes();
```

您可以為 `job` 方法提供選填的第二和第三個引數，用以指定推入任務時應使用的佇列名稱和佇列連線：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

// Dispatch the job to the "heartbeats" queue on the "sqs" connection...
Schedule::job(new Heartbeat, 'heartbeats', 'sqs')->everyFiveMinutes();
```

<a name="scheduling-shell-commands"></a>
### 排程 Shell 指令

`exec` 方法可用於向作業系統發送指令：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::exec('node /home/forge/script.js')->daily();
```

<a name="schedule-frequency-options"></a>
### 排程頻率選項

我們已經看過一些如何設定任務在特定時間間隔執行的範例。不過，你還可以將許多其他的排程頻率指派給任務：

<div class="overflow-auto">

| Method                             | Description                                              |
| ---------------------------------- | -------------------------------------------------------- |
| `->cron('* * * * *');`             | 使用自訂的 cron 排程執行任務。                            |
| `->everySecond();`                 | 每秒執行一次任務。                                       |
| `->everyTwoSeconds();`             | 每兩秒執行一次任務。                                     |
| `->everyFiveSeconds();`            | 每五秒執行一次任務。                                     |
| `->everyTenSeconds();`             | 每十秒執行一次任務。                                     |
| `->everyFifteenSeconds();`         | 每十五秒執行一次任務。                                   |
| `->everyTwentySeconds();`          | 每二十秒執行一次任務。                                   |
| `->everyThirtySeconds();`          | 每三十秒執行一次任務。                                   |
| `->everyMinute();`                 | 每分鐘執行一次任務。                                     |
| `->everyTwoMinutes();`             | 每兩分鐘執行一次任務。                                   |
| `->everyThreeMinutes();`           | 每三分鐘執行一次任務。                                   |
| `->everyFourMinutes();`            | 每四分鐘執行一次任務。                                   |
| `->everyFiveMinutes();`            | 每五分鐘執行一次任務。                                   |
| `->everyTenMinutes();`             | 每十分鐘執行一次任務。                                   |
| `->everyFifteenMinutes();`         | 每十五分鐘執行一次任務。                                 |
| `->everyThirtyMinutes();`          | 每三點鐘執行一次任務。                                   |
| `->hourly();`                      | 每小時執行一次任務。                                     |
| `->hourlyAt(17);`                  | 在每小時的第 17 分時執行任務。                            |
| `->everyOddHour($minutes = 0);`    | 在每個奇數小時執行任務。                                 |
| `->everyTwoHours($minutes = 0);`   | 每兩小時執行一次任務。                                   |
| `->everyThreeHours($minutes = 0);` | 每三小時執行一次任務。                                   |
| `->everyFourHours($minutes = 0);`  | 每四小時執行一次任務。                                   |
| `->everySixHours($minutes = 0);`   | 每六小時執行一次任務。                                   |
| `->daily();`                       | 每天午夜執行一次任務。                                   |
| `->dailyAt('13:00');`              | 每天 13:00 執行一次任務。                                 |
| `->twiceDaily(1, 13);`             | 每天 1:00 與 13:00 各執行一次任務。                      |
| `->twiceDailyAt(1, 13, 15);`       | 每天 1:15 與 13:15 各執行一次任務。                      |
| `->daysOfMonth([1, 10, 20]);`      | 在每個月的特定日期執行任務。                             |
| `->weekly();`                      | 每週日 00:00 執行一次任務。                              |
| `->weeklyOn(1, '8:00');`           | 每週一 8:00 執行一次任務。                               |
| `->monthly();`                     | 每個月的第一天 00:00 執行一次任務。                       |
| `->monthlyOn(4, '15:00');`         | 每個月的 4 號 15:00 執行一次任務。                        |
| `->twiceMonthly(1, 16, '13:00');`  | 每個月的 1 號與 16 號的 13:00 各執行一次任務。             |
| `->lastDayOfMonth('15:00');`       | 每個月的最後一天 15:00 執行一次任務。                    |
| `->quarterly();`                   | 每季的第一天 00:00 執行一次任務。                        |
| `->quarterlyOn(4, '14:00');`       | 每季的 4 號 14:00 執行一次任務。                          |
| `->yearly();`                      | 每年的第一天 00:00 執行一次任務。                        |
| `->yearlyOn(6, 1, '17:00');`       | 每年的 6 月 1 日 17:00 執行一次任務。                     |
| `->timezone('America/New_York');`  | 設定任務的時區。                                         |

</div>

這些方法可以與額外的限制條件結合，以建立更精細的排程，例如只在週中的某些天執行。例如，你可以排程一個指令，讓它在每週一執行：

```php
use Illuminate\Support\Facades\Schedule;

// Run once per week on Monday at 1 PM...
Schedule::call(function () {
    // ...
})->weekly()->mondays()->at('13:00');

// Run hourly from 8 AM to 5 PM on weekdays...
Schedule::command('foo')
    ->weekdays()
    ->hourly()
    ->timezone('America/Chicago')
    ->between('8:00', '17:00');
```

以下是其他排程限制條件的清單：

<div class="overflow-auto">

| Method                                   | Description                                            |
| ---------------------------------------- | ------------------------------------------------------ |
| `->weekdays();`                          | 限制任務只在工作日（週一至週五）執行。                 |
| `->weekends();`                          | 限制任務只在週末（週六與週日）執行。                   |
| `->sundays();`                           | 限制任務只在週日執行。                                 |
| `->mondays();`                           | 限制任務只在週一執行。                                 |
| `->tuesdays();`                          | 限制任務只在週二執行。                                 |
| `->wednesdays();`                        | 限制任務只在週三執行。                                 |
| `->thursdays();`                         | 限制任務只在週四執行。                                 |
| `->fridays();`                           | 限制任務只在週五執行。                                 |
| `->saturdays();`                         | 限制任務只在週六執行。                                 |
| `->days(array\|mixed);`                  | 限制任務只在特定日期（星期）執行。                     |
| `->between($startTime, $endTime);`       | 限制任務在開始與結束時間之間執行。                     |
| `->unlessBetween($startTime, $endTime);` | 限制任務不在開始與結束時間之間執行。                   |
| `->when(Closure);`                       | 根據真值測試結果限制任務執行。                         |
| `->environments($env);`                  | 限制任務只在特定的環境中執行。                         |

</div>


<a name="day-constraints"></a>
#### 星期限制

可以使用 `days` 方法來限制任務只在每週的特定幾天執行。例如，你可以排程一個指令，讓它在每週日和週三每小時執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->hourly()
    ->days([0, 3]);
```

或者，你也可以在定義任務執行的星期時，使用 `Illuminate\Console\Scheduling\Schedule` 類別中提供的常數：

```php
use Illuminate\Support\Facades;
use Illuminate\Console\Scheduling\Schedule;

Facades\Schedule::command('emails:send')
    ->hourly()
    ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);
```


<a name="between-time-constraints"></a>
#### 時間區間限制

可以使用 `between` 方法，根據一天的特定時間段來限制任務的執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->between('7:00', '22:00');
```

同樣地，可以使用 `unlessBetween` 方法在特定的時間段內排除任務的執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->unlessBetween('23:00', '4:00');
```


<a name="truth-test-constraints"></a>
#### 真值測試限制

可以使用 `when` 方法，根據指定的真值測試結果來限制任務的執行。換句話說，如果指定的閉包返回 `true`，只要沒有其他限制條件阻止，任務就會執行：

```php
Schedule::command('emails:send')->daily()->when(function () {
    return true;
});
```

而 `skip` 方法則可視為 `when` 的相反。如果 `skip` 方法返回 `true`，該排程任務將不會被執行：

```php
Schedule::command('emails:send')->daily()->skip(function () {
    return true;
});
```

當鏈結使用多個 `when` 方法時，只有在所有的 `when` 條件都返回 `true` 時，排程指令才會執行。


<a name="environment-constraints"></a>
#### 環境限制

可以使用 `environments` 方法限制任務只在指定的環境中執行（由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#environment-configuration)定義）：

```php
Schedule::command('emails:send')
    ->daily()
    ->environments(['staging', 'production']);
```

<a name="timezones"></a>
### 時區

使用 `timezone` 方法，您可以指定排程任務的時間應在給定的時區內進行解析：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->timezone('America/New_York')
    ->at('2:00')
```

如果您重複為所有排程任務分配相同的時區，可以透過在應用程式的 `app` 設定檔中定義 `schedule_timezone` 選項，來指定應分配給所有排程的時區：

```php
'timezone' => 'UTC',

'schedule_timezone' => 'America/Chicago',
```

> [!WARNING]
> 請記住，某些時區會使用日光節約時間。當日光節約時間發生變更時，您的排程任務可能會執行兩次，甚至完全不執行。因此，我們建議盡可能避免使用時區排程。

<a name="preventing-task-overlaps"></a>
### 防止任務重複執行

預設情況下，即使前一個任務實例仍在執行中，排程任務也會照常執行。為了防止這種情況，您可以使用 `withoutOverlapping` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->withoutOverlapping();
```

在此範例中，如果 `emails:send` [Artisan 指令](/docs/{{version}}/artisan) 尚未執行，則每分鐘都會執行一次。如果您的任務執行時間差異極大，導致您無法準確預測特定任務需要花費多少時間，那麼 `withoutOverlapping` 方法會特別有用。

如果需要，您可以指定在「無重複執行 (without overlapping)」鎖定過期之前必須經過多少分鐘。預設情況下，該鎖定會在 24 小時後過期：

```php
Schedule::command('emails:send')->withoutOverlapping(10);
```

在幕後，`withoutOverlapping` 方法會利用應用程式的 [快取](/docs/{{version}}/cache) 來取得鎖定。如有必要，您可以使用 `schedule:clear-cache` Artisan 指令來清除這些快取鎖定。這通常只在任務因未預期的伺服器問題而卡住時才需要。

<a name="running-tasks-on-one-server"></a>
### 在單一伺服器上執行任務

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `database`、`memcached`、`dynamodb` 或 `redis` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器進行通訊。

如果您的應用程式排程器執行在多台伺服器上，您可以限制排程任務僅在單一伺服器上執行。例如，假設您有一個排程任務，在每週五晚上產生一份新報告。如果任務排程器執行在三台背景工作 (worker) 伺服器上，該排程任務將在所有三台伺服器上執行，並產生三次報告。這可不好！

要指示任務僅在單一伺服器上執行，請在定義排程任務時使用 `onOneServer` 方法。第一台取得任務的伺服器將會對該任務確保一個原子鎖 (atomic lock)，以防止其他伺服器在同一時間執行相同的任務：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->fridays()
    ->at('17:00')
    ->onOneServer();
```

您可以使用 `useCache` 方法來自訂排程器所使用的快取存放區，以取得單一伺服器任務所需的原子鎖：

```php
Schedule::useCache('database');
```

<a name="naming-unique-jobs"></a>
#### 為單一伺服器任務命名

有時您可能需要排程將相同的任務以不同的參數分派，但仍希望指示 Laravel 在單一伺服器上執行該任務的每個排列組合。若要達到此目的，您可以使用 `name` 方法為每個排程定義分配一個不重複的名稱：

```php
Schedule::job(new CheckUptime('https://laravel.com'))
    ->name('check_uptime:laravel.com')
    ->everyFiveMinutes()
    ->onOneServer();

Schedule::job(new CheckUptime('https://vapor.laravel.com'))
    ->name('check_uptime:vapor.laravel.com')
    ->everyFiveMinutes()
    ->onOneServer();
```

同樣地，如果排程閉包 (closures) 打算在單一伺服器上執行，也必須為其分配一個名稱：

```php
Schedule::call(fn () => User::resetApiRequestCount())
    ->name('reset-api-request-count')
    ->daily()
    ->onOneServer();
```

<a name="background-tasks"></a>
### 背景任務

預設情況下，同時排程的多個任務將根據它們在 `schedule` 方法中定義的順序依序執行。如果您有執行時間很長的任務，這可能會導致後續任務的啟動時間比預期晚得多。如果您希望在背景執行任務，以便它們可以同時執行，您可以使用 `runInBackground` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('analytics:report')
    ->daily()
    ->runInBackground();
```

> [!WARNING]
> `runInBackground` 方法僅能在透過 `command` 與 `exec` 方法排程任務時使用。

<a name="maintenance-mode"></a>
### 維護模式

當應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，應用程式的排程任務將不會執行，因為我們不希望您的任務干擾您可能正在伺服器上進行的任何未完成維護。然而，如果您想強制任務即使在維護模式下也能執行，您可以在定義任務時呼叫 `evenInMaintenanceMode` 方法：

```php
Schedule::command('emails:send')->evenInMaintenanceMode();
```

<a name="pausing-scheduled-tasks"></a>
### 暫停排程任務

您可以使用 `schedule:pause` Artisan 指令暫時暫停處理排程任務，而無需變更已部署的程式碼：

```shell
php artisan schedule:pause
```

當排程器暫停時，將不會執行任何排程任務。您可以使用 `schedule:continue` 指令恢復排程任務處理：

```shell
php artisan schedule:continue
```

如果某個任務在排程器暫停時仍應執行，您可以使用 `evenWhenPaused` 方法對其進行標記：

```php
Schedule::command('emails:send')->evenWhenPaused();
```

<a name="schedule-groups"></a>
### 排程群組

在定義多個具有相似設定的排程任務時，您可以使用 Laravel 的任務群組功能，以避免為每個任務重複相同的設定。將任務進行群組可以簡化您的程式碼，並確保相關任務之間的一致性。

要建立一個排程任務群組，請呼叫所需的任務設定方法，接著使用 `group` 方法。`group` 方法接受一個閉包，該閉包負責定義共享該指定設定的任務：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::daily()
    ->onOneServer()
    ->timezone('America/New_York')
    ->group(function () {
        Schedule::command('emails:send --force');
        Schedule::command('emails:prune');
    });
```

<a name="running-the-scheduler"></a>
## 執行排程器

現在我們已經學會了如何定義排程任務，接著讓我們來探討如何在伺服器上實際執行它們。`schedule:run` 這個 Artisan 指令會評估你所有的排程任務，並根據伺服器當前的時間來判斷它們是否需要執行。

因此，當使用 Laravel 的排程器時，我們只需要在伺服器上新增單一一個 cron 設定項目，讓它每分鐘執行一次 `schedule:run` 指令即可。如果你不知道如何向伺服器新增 cron 項目，可以考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣的代管平台，它可以為你管理排程任務的執行：

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

<a name="sub-minute-scheduled-tasks"></a>
### 小於一分鐘的排程任務

在大多數作業系統上，cron 任務限制為最多每分鐘執行一次。然而，Laravel 的排程器允許你以更頻繁的間隔來排程任務，甚至可以頻繁到每秒執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->everySecond();
```

當你的應用程式中定義了小於一分鐘的任務時，`schedule:run` 指令會持續執行直到當前分鐘結束，而不是立即結束。這使得該指令能夠在這一分鐘內呼叫所有需要的小於一分鐘的任務。

由於執行時間超出預期的小於一分鐘任務可能會延遲後續小於一分鐘任務的執行，因此建議所有小於一分鐘的任務都分派佇列任務或背景指令來處理實際的任務流程：

```php
use App\Jobs\DeleteRecentUsers;

Schedule::job(new DeleteRecentUsers)->everyTenSeconds();

Schedule::command('users:delete')->everyTenSeconds()->runInBackground();
```

<a name="interrupting-sub-minute-tasks"></a>
#### 中斷小於一分鐘的任務

由於在定義了小於一分鐘的任務時，`schedule:run` 指令在被呼叫後會執行整整一分鐘，因此在部署應用程式時，你可能偶爾會需要中斷該指令。否則，一個已經在執行的 `schedule:run` 指令實例將會繼續使用你應用程式先前部署的程式碼，直到當前分鐘結束為止。

若要中斷執行中的 `schedule:run` 呼叫，你可以將 `schedule:interrupt` 指令新增到應用程式的部署指令碼中。這個指令應該在應用程式部署完成後被呼叫：

```shell
php artisan schedule:interrupt
```

<a name="running-the-scheduler-locally"></a>
### 在本機執行排程器

通常，你不需要在本地開發電腦上新增排程器的 cron 項目。相反地，你可以使用 `schedule:work` 這個 Artisan 指令。該指令會在前景執行，並每分鐘呼叫一次排程器，直到你結束該指令為止。當定義了小於一分鐘的任務時，排程器會繼續在每一分鐘內執行以處理這些任務：

```shell
php artisan schedule:work
```

<a name="task-output"></a>
## 任務輸出

Laravel 排程器提供了幾種方便的方法來處理排程任務產生的輸出。首先，使用 `sendOutputTo` 方法，你可以將輸出傳送到檔案中以便稍後檢查：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->daily()
    ->sendOutputTo($filePath);
```

如果你想要將輸出附加到指定的檔案中，可以使用 `appendOutputTo` 方法：

```php
Schedule::command('emails:send')
    ->daily()
    ->appendOutputTo($filePath);
```

使用 `emailOutputTo` 方法，你可以將輸出寄送到你指定的電子郵件地址。在寄送任務輸出之前，你應該先設定好 Laravel 的[電子郵件服務](/docs/{{version}}/mail)：

```php
Schedule::command('report:generate')
    ->daily()
    ->sendOutputTo($filePath)
    ->emailOutputTo('taylor@example.com');
```

如果你只想在排程的 Artisan 或系統指令以非零結束代碼結束時才寄送輸出，可以使用 `emailOutputOnFailure` 方法：

```php
Schedule::command('report:generate')
    ->daily()
    ->emailOutputOnFailure('taylor@example.com');
```

> [!WARNING]
> `emailOutputTo`、`emailOutputOnFailure`、`sendOutputTo` 與 `appendOutputTo` 方法僅適用於 `command` 與 `exec` 方法。

<a name="task-hooks"></a>
## 任務勾點

使用 `before` 與 `after` 方法，你可以指定在排程任務執行之前與之後要執行的程式碼：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->daily()
    ->before(function () {
        // The task is about to execute...
    })
    ->after(function () {
        // The task has executed...
    });
```

`onSuccess` 和 `onFailure` 方法允許你指定在排程任務成功或失敗時執行的程式碼。失敗表示排程的 Artisan 或系統指令以非零結束代碼結束：

```php
Schedule::command('emails:send')
    ->daily()
    ->onSuccess(function () {
        // The task succeeded...
    })
    ->onFailure(function () {
        // The task failed...
    });
```

如果你的指令有輸出，你可以在 `after`、`onSuccess` 或 `onFailure` 勾點中存取它，方法是在你的勾點閉包定義中，將 `Illuminate\Support\Stringable` 實例型別提示為 `$output` 引數：

```php
use Illuminate\Support\Stringable;

Schedule::command('emails:send')
    ->daily()
    ->onSuccess(function (Stringable $output) {
        // The task succeeded...
    })
    ->onFailure(function (Stringable $output) {
        // The task failed...
    });
```

<a name="pinging-urls"></a>
#### Ping URL

使用 `pingBefore` 與 `thenPing` 方法，排程器可以在任務執行之前或之後自動 Ping 指定的 URL。這個方法對於通知外部服務（例如 [Envoyer](https://envoyer.io)）你的排程任務正在開始或已完成執行非常有用：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingBefore($url)
    ->thenPing($url);
```

`pingOnSuccess` 與 `pingOnFailure` 方法可以用來在任務成功或失敗時才 Ping 指定的 URL。失敗表示排程的 Artisan 或系統指令以非零結束代碼結束：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingOnSuccess($successUrl)
    ->pingOnFailure($failureUrl);
```

`pingBeforeIf`、`thenPingIf`、`pingOnSuccessIf` 以及 `pingOnFailureIf` 方法可以用來在給定條件為 `true` 時才 Ping 指定的 URL：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingBeforeIf($condition, $url)
    ->thenPingIf($condition, $url);

Schedule::command('emails:send')
    ->daily()
    ->pingOnSuccessIf($condition, $successUrl)
    ->pingOnFailureIf($condition, $failureUrl);
```

<a name="events"></a>
## 事件

Laravel 在排程過程中會發送多種[事件](/docs/{{version}}/events)。你可以為以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| Event Name                                                  |
| ----------------------------------------------------------- |
| `Illuminate\Console\Events\ScheduledTaskStarting`           |
| `Illuminate\Console\Events\ScheduledTaskFinished`           |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` |
| `Illuminate\Console\Events\ScheduledTaskSkipped`            |
| `Illuminate\Console\Events\ScheduledTaskFailed`             |

</div>