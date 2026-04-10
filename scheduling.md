# 任務排程

- [簡介](#introduction)
- [定義排程](#defining-schedules)
    - [排程 Artisan 命令](#scheduling-artisan-commands)
    - [排程佇列工作](#scheduling-queued-jobs)
    - [排程 Shell 命令](#scheduling-shell-commands)
    - [排程頻率選項](#schedule-frequency-options)
    - [時區](#timezones)
    - [防止任務重疊](#preventing-task-overlaps)
    - [在單一伺服器上執行任務](#running-tasks-on-one-server)
    - [背景任務](#background-tasks)
    - [維護模式](#maintenance-mode)
    - [暫停排程任務](#pausing-scheduled-tasks)
    - [排程群組](#schedule-groups)
- [執行排程器](#running-the-scheduler)
    - [次分鐘排程任務](#sub-minute-scheduled-tasks)
    - [在本地執行排程器](#running-the-scheduler-locally)
- [任務輸出](#task-output)
- [任務鉤子](#task-hooks)
- [事件](#events)

<a name="introduction"></a>
## 簡介

過去，您可能需要為伺服器上需要排程的每個任務編寫一條 cron 配置項目。然而，這很快會變得令人困擾，因為您的任務排程不再處於版本控制中，而且您必須透過 SSH 進入伺服器來查看現有的 cron 項目或新增項目。

Laravel 的命令排程器提供了一種全新的方式來管理伺服器上的排程任務。排程器允許您在 Laravel 應用程式本身中，以流暢且具表達力的方式定義您的命令排程。使用排程器時，您的伺服器只需要單一條 cron 項目。您的任務排程通常定義在應用程式的 `routes/console.php` 檔案中。

<a name="defining-schedules"></a>
## 定義排程

您可以在應用程式的 `routes/console.php` 檔案中定義所有的排程任務。首先，讓我們來看一個範例。在這個範例中，我們將排程一個 closure，使其每天在午夜被呼叫。在這個 closure 之中，我們將執行一個資料庫查詢來清除某個資料表：

```php
<?php

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->daily();
```

除了使用 closure 進行排程之外，您也可以排程 [可呼叫物件 (invokable objects)](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke)。可呼叫物件是指包含 `__invoke` 方法的簡單 PHP 類別：

```php
Schedule::call(new DeleteRecentUsers)->daily();
```

如果您希望將 `routes/console.php` 檔案僅保留用於命令定義，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `withSchedule` 方法來定義您的排程任務。此方法接受一個 closure，該 closure 會接收一個排程器實例：

```php
use Illuminate\Console\Scheduling\Schedule;

->withSchedule(function (Schedule $schedule) {
    $schedule->call(new DeleteRecentUsers)->daily();
})
```

如果您想查看排程任務的概覽以及它們下次預定執行的時間，可以使用 `schedule:list` Artisan 命令：

```shell
php artisan schedule:list
```


<a name="scheduling-artisan-commands"></a>
### 排程 Artisan 命令

除了排程 closure 之外，您也可以排程 [Artisan 命令](/docs/{{version}}/artisan) 和系統命令。例如，您可以使用 `command` 方法，透過命令名稱或類別來排程 Artisan 命令。

當使用命令類別名稱來排程 Artisan 命令時，您可以傳遞一個包含額外命令列引數的陣列，這些引數將在命令被呼叫時提供：

```php
use App\Console\Commands\SendEmailsCommand;
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send Taylor --force')->daily();

Schedule::command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();
```


<a name="scheduling-artisan-closure-commands"></a>
#### 排程 Artisan Closure 命令

如果您想要排程一個由 closure 定義的 Artisan 命令，您可以在命令定義之後鏈接排程相關的方法：

```php
Artisan::command('delete:recent-users', function () {
    DB::table('recent_users')->delete();
})->purpose('Delete recent users')->daily();
```

如果您需要將引數傳遞給 closure 命令，可以將它們提供給 `schedule` 方法：

```php
Artisan::command('emails:send {user} {--force}', function ($user) {
    // ...
})->purpose('Send emails to the specified user')->schedule(['Taylor', '--force'])->daily();
```


<a name="scheduling-queued-jobs"></a>
### 排程佇列工作

`job` 方法可用於排程 [佇列工作](/docs/{{version}}/queues)。此方法提供了一種便捷的方式來排程佇列工作，而不需要使用 `call` 方法來定義 closure 以將工作放入佇列：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

Schedule::job(new Heartbeat)->everyFiveMinutes();
```

`job` 方法可以提供選填的第二和第三個引數，用來指定將工作放入佇列時應使用的佇列名稱和佇列連線：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

// Dispatch the job to the "heartbeats" queue on the "sqs" connection...
Schedule::job(new Heartbeat, 'heartbeats', 'sqs')->everyFiveMinutes();
```


<a name="scheduling-shell-commands"></a>
### 排程 Shell 命令

`exec` 方法可用於向作業系統發出命令：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::exec('node /home/forge/script.js')->daily();
```

<a name="schedule-frequency-options"></a>
### 排程頻率選項

我們已經看過幾個關於如何設定任務在指定間隔執行的範例。然而，您還可以為任務指定更多不同的排程頻率：

<div class="overflow-auto">

| 方法                             | 描述                                                                             |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| `->cron('* * * * *');`             | 根據自定義的 cron 排程執行任務。                                                  |
| `->everySecond();`                 | 每秒執行一次任務。                                                                  |
| `->everyTwoSeconds();`             | 每兩秒執行一次任務。                                                                |
| `->everyFiveSeconds();`            | 每五秒執行一次任務。                                                                |
| `->everyTenSeconds();`             | 每十秒執行一次任務。                                                                |
| `->everyFifteenSeconds();`         | 每十五秒執行一次任務。                                                              |
| `->everyTwentySeconds();`          | 每二十秒執行一次任務。                                                              |
| `->everyThirtySeconds();`          | 每三十秒執行一次任務。                                                              |
| `->everyMinute();`                 | 每分鐘執行一次任務。                                                                |
| `->everyTwoMinutes();`             | 每兩分鐘執行一次任務。                                                              |
| `->everyThreeMinutes();`           | 每三分鐘執行一次任務。                                                              |
| `->everyFourMinutes();`            | 每四分鐘執行一次任務。                                                              |
| `->everyFiveMinutes();`            | 每五分鐘執行一次任務。                                                              |
| `->everyTenMinutes();`             | 每十分鐘執行一次任務。                                                              |
| `->everyFifteenMinutes();`         | 每十五分鐘執行一次任務。                                                              |
| `->everyThirtyMinutes();`          | 每三十分鐘執行一次任務。                                                              |
| `->hourly();`                      | 每小時執行一次任務。                                                                |
| `->hourlyAt(17);`                  | 每小時在第 17 分鐘執行一次任務。                                                    |
| `->everyOddHour($minutes = 0);`    | 在每個奇數小時執行一次任務。                                                        |
| `->everyTwoHours($minutes = 0);`   | 每兩小時執行一次任務。                                                              |
| `->everyThreeHours($minutes = 0);` | 每三小時執行一次任務。                                                              |
| `->everyFourHours($minutes = 0);`  | 每四小時執行一次任務。                                                              |
| `->everySixHours($minutes = 0);`   | 每六小時執行一次任務。                                                              |
| `->daily();`                       | 每天在午夜執行一次任務。                                                            |
| `->dailyAt('13:00');`              | 每天在 13:00 執行一次任務。                                                        |
| `->twiceDaily(1, 13);`             | 每天在 1:00 和 13:00 執行任務。                                                    |
| `->twiceDailyAt(1, 13, 15);`       | 每天在 1:15 和 13:15 執行任務。                                                    |
| `->daysOfMonth([1, 10, 20]);`      | 在每月的特定日期執行任務。                                                          |
| `->weekly();`                      | 每週日 00:00 執行一次任務。                                                        |
| `->weeklyOn(1, '8:00');`           | 每週一 8:00 執行一次任務。                                                          |
| `->monthly();`                     | 每月第一天 00:00 執行一次任務。                                                     |
| `->monthlyOn(4, '15:00');`         | 每月 4 日 15:00 執行一次任務。                                                      |
| `->twiceMonthly(1, 16, '13:00');`  | 每月 1 日和 16 日的 13:00 執行任務。                                                 |
| `->lastDayOfMonth('15:00');`       | 每月最後一天 15:00 執行一次任務。                                                   |
| `->quarterly();`                   | 每季第一天 00:00 執行一次任務。                                                     |
| `->quarterlyOn(4, '14:00');`       | 每季 4 日 14:00 執行一次任務。                                                      |
| `->yearly();`                      | 每年第一天 00:00 執行一次任務。                                                     |
| `->yearlyOn(6, 1, '17:00');`       | 每年 6 月 1 日 17:00 執行一次任務。                                                  |
| `->timezone('America/New_York');`  | 為任務設定時區。                                                                  |

</div>

這些方法可以與額外的限制條件結合，以建立更精細的排程，使其僅在每週的特定日期執行。例如，您可以排程一個命令每週一執行：

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

額外的排程限制清單如下：

<div class="overflow-auto">

| 方法                                   | 描述                                                                           |
| ---------------------------------------- | -------------------------------------------------------------------------------------- |
| `->weekdays();`                          | 將任務限制在工作日。                                                                |
| `->weekends();`                          | 將任務限制在週末。                                                                  |
| `->sundays();`                           | 將任務限制在週日。                                                                  |
| `->mondays();`                           | 將任務限制在週一。                                                                  |
| `->tuesdays();`                          | 將任務限制在週二。                                                                  |
| `->wednesdays();`                        | 將任務限制在週三。                                                                  |
| `->thursdays();`                         | 將任務限制在週四。                                                                  |
| `->fridays();`                           | 將任務限制在週五。                                                                  |
| `->saturdays();`                         | 將任務限制在週六。                                                                  |
| `->days(array\|mixed);`                  | 將任務限制在特定日期。                                                            |
| `->between($startTime, $endTime);`       | 將任務限制在開始時間與結束時間之間執行。                                         |
| `->unlessBetween($startTime, $endTime);` | 將任務限制在不於開始時間與結束時間之間執行。                                     |
| `->when(Closure);`                       | 根據真值測試結果來限制任務。                                                      |
| `->environments($env);`                  | 將任務限制在特定環境中執行。                                                      |

</div>


<a name="day-constraints"></a>
#### 日期限制

可以使用 `days` 方法將任務的執行限制在每週的特定日期。例如，您可以排程一個命令在每週日和週三每小時執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->hourly()
    ->days([0, 3]);
```

或者，在定義任務應執行的日期時，您可以使用 `Illuminate\Console\Scheduling\Schedule` 類別中提供的常數：

```php
use Illuminate\Support\Facades;
use Illuminate\Console\Scheduling\Schedule;

Facades\Schedule::command('emails:send')
    ->hourly()
    ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);
```


<a name="between-time-constraints"></a>
#### 時間區間限制

可以使用 `between` 方法根據一天中的時間來限制任務的執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->between('7:00', '22:00');
```

同樣地，可以使用 `unlessBetween` 方法來排除一段時間內執行任務：

```php
Schedule::command('emails:send')
    ->hourly()
    ->unlessBetween('23:00', '4:00');
```


<a name="truth-test-constraints"></a>
#### 真值測試限制

可以使用 `when` 方法根據給定真值測試的結果來限制任務的執行。換句話說，只要給定的閉包返回 `true`，且沒有其他限制條件阻止任務執行，該任務就會執行：

```php
Schedule::command('emails:send')->daily()->when(function () {
    return true;
});
```

`skip` 方法可以被視為 `when` 的反向操作。如果 `skip` 方法返回 `true`，則排程任務將不會執行：

```php
Schedule::command('emails:send')->daily()->skip(function () {
    return true;
});
```

當使用鏈接的 `when` 方法時，只有在所有 `when` 條件都返回 `true` 時，排程命令才會執行。


<a name="environment-constraints"></a>
#### 環境限制

可以使用 `environments` 方法僅在給定的環境中執行任務（由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#environment-configuration) 定義）：

```php
Schedule::command('emails:send')
    ->daily()
    ->environments(['staging', 'production']);
```

<a name="timezones"></a>
### 時區

使用 `timezone` 方法，您可以指定排程任務的時間應在特定的時區內解釋：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->timezone('America/New_York')
    ->at('2:00')
```

如果您重複將相同的時區分配給所有排程任務，您可以透過在應用程式的 `app` 設定檔中定義 `schedule_timezone` 選項，來指定所有排程應分配的時區：

```php
'timezone' => 'UTC',

'schedule_timezone' => 'America/Chicago',
```

> [!WARNING]
> 請記住，某些時區使用日光節約時間。當日光節約時間變更時，您的排程任務可能會執行兩次，甚至完全不執行。因此，我們建議在可能的情況下，儘量避免使用時區排程。


<a name="preventing-task-overlaps"></a>
### 防止任務重疊

預設情況下，即使該任務的前一個實例仍在執行，排程任務仍會被執行。為了防止這種情況，您可以使用 `withoutOverlapping` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->withoutOverlapping();
```

在這個範例中，如果 `emails:send` [Artisan 命令](/docs/{{version}}/artisan) 尚未在執行中，它將每分鐘執行一次。如果您有執行時間差異巨大的任務，導致您無法準確預測特定任務需要花多少時間，那麼 `withoutOverlapping` 方法會特別有用。

如果需要，您可以指定在「防止重疊」鎖定過期之前必須經過多少分鐘。預設情況下，鎖定將在 24 小時後過期：

```php
Schedule::command('emails:send')->withoutOverlapping(10);
```

在幕後，`withoutOverlapping` 方法利用您應用程式的 [快取](/docs/{{version}}/cache) 來獲取鎖定。如有必要，您可以使用 `schedule:clear-cache` Artisan 命令來清除這些快取鎖定。這通常僅在任務因意外的伺服器問題而卡住時才需要。


<a name="running-tasks-on-one-server"></a>
### 在單一伺服器上執行任務

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `database`、`memcached`、`dynamodb` 或 `redis` 快取驅動程式作為預設快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器通訊。

如果您的應用程式排程器在多台伺服器上執行，您可以將排程工作限制在僅在單一伺服器上執行。例如，假設您有一個排程任務在每週五晚上產生一份新報告。如果任務排程器在三台工作伺服器上執行，該排程任務將在所有三台伺服器上執行並產生三份報告。這是不對的！

若要指示任務僅在單一伺服器上執行，請在定義排程任務時使用 `onOneServer` 方法。第一個獲取該任務的伺服器將獲取該工作的原子鎖，以防止其他伺服器在同一時間執行相同的任務：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->fridays()
    ->at('17:00')
    ->onOneServer();
```

您可以使用 `useCache` 方法來自訂排程器用於獲取單伺服器任務所需原子鎖的快取儲存：

```php
Schedule::useCache('database');
```


<a name="naming-unique-jobs"></a>
#### 為單一伺服器工作命名

有時候您可能需要排程同一個工作，但使用不同的參數發送，同時仍要求 Laravel 在單一伺服器上執行該工作的每個排列組合。為了實現這一點，您可以使用 `name` 方法為每個排程定義分配一個唯一的名稱：

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

同樣地，如果排程的閉包打算在單一伺服器上執行，則必須分配名稱：

```php
Schedule::call(fn () => User::resetApiRequestCount())
    ->name('reset-api-request-count')
    ->daily()
    ->onOneServer();
```


<a name="background-tasks"></a>
### 背景任務

預設情況下，在同一時間排程的多個任務將根據它們在 `schedule` 方法中定義的順序依序執行。如果您有執行時間較長的任務，這可能會導致後續任務的啟動時間比預期晚得多。如果您希望在背景執行任務，以便它們可以同時運行，可以使用 `runInBackground` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('analytics:report')
    ->daily()
    ->runInBackground();
```

> [!WARNING]
> `runInBackground` 方法僅適用於透過 `command` 和 `exec` 方法排程的任務。


<a name="maintenance-mode"></a>
### 維護模式

當應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，您的應用程式排程任務將不會執行，因為我們不希望您的任務干擾您在伺服器上可能正在進行的未完成維護。但是，如果您希望強制任務即使在維護模式下也能執行，可以在定義任務時呼叫 `evenInMaintenanceMode` 方法：

```php
Schedule::command('emails:send')->evenInMaintenanceMode();
```


<a name="pausing-scheduled-tasks"></a>
### 暫停排程任務

您可以使用 `schedule:pause` Artisan 命令，在不更改已部署程式碼的情況下暫時暫停排程任務的處理：

```shell
php artisan schedule:pause
```

當排程器被暫停時，沒有排程任務會執行。您可以使用 `schedule:continue` 命令恢復排程任務的處理：

```shell
php artisan schedule:continue
```

如果某個任務在排程器暫停時仍應執行，您可以使用 `evenWhenPaused` 方法將其標記：

```php
Schedule::command('emails:send')->evenWhenPaused();
```


<a name="schedule-groups"></a>
### 排程群組

在定義多個具有相似配置的排程任務時，您可以使用 Laravel 的任務分組功能，以避免為每個任務重複設定相同的設定。分組任務可以簡化您的程式碼，並確保相關任務之間的一致性。

要建立一組排程任務，請先呼叫所需的任務配置方法，然後呼叫 `group` 方法。`group` 方法接受一個閉包，該閉包負責定義共享指定配置的任務：

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

現在我們已經學習了如何定義排程任務，讓我們來討論如何在伺服器上實際執行它們。`schedule:run` Artisan 命令會評估所有排程任務，並根據伺服器目前的時間決定是否需要執行。

因此，在使用 Laravel 的排程器時，我們只需要在伺服器上新增一條 cron 設定，每分鐘執行一次 `schedule:run` 命令。如果您不知道如何在伺服器上新增 cron 設定，可以考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣的託管平台，它可以為您管理排程任務的執行：

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```


<a name="sub-minute-scheduled-tasks"></a>
### 次分鐘排程任務

在大多數作業系統中，cron 工作最多每分鐘只能執行一次。然而，Laravel 的排程器允許您將任務排程在更頻繁的間隔執行，甚至可以每秒執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->everySecond();
```

當應用程式中定義了次分鐘任務時，`schedule:run` 命令將持續執行直到目前這一分鐘結束，而不會立即退出。這使得該命令能夠在這一分鐘內調用所有必要的次分鐘任務。

由於執行時間超出預期的次分鐘任務可能會延遲後續次分鐘任務的執行，因此建議所有次分鐘任務都應發送佇列工作或背景命令來處理實際的任務處理：

```php
use App\Jobs\DeleteRecentUsers;

Schedule::job(new DeleteRecentUsers)->everyTenSeconds();

Schedule::command('users:delete')->everyTenSeconds()->runInBackground();
```


<a name="interrupting-sub-minute-tasks"></a>
#### 中斷次分鐘任務

由於定義次分鐘任務時 `schedule:run` 命令會在調用的整分鐘內執行，因此在部署應用程式時，您有時可能需要中斷該命令。否則，已經在執行中的 `schedule:run` 命令實例將繼續使用先前部署的應用程式程式碼，直到目前這一分鐘結束。

要中斷正在進行中的 `schedule:run` 調用，您可以將 `schedule:interrupt` 命令添加到應用程式的部署指令碼中。此命令應在應用程式完成部署後調用：

```shell
php artisan schedule:interrupt
```


<a name="running-the-scheduler-locally"></a>
### 在本地執行排程器

通常，您不會在本地開發機上新增排程器的 cron 設定。相反地，您可以使用 `schedule:work` Artisan 命令。此命令將在前台執行，並每分鐘調用一次排程器，直到您終止該命令。當定義了次分鐘任務時，排程器將在每分鐘內持續執行以處理這些任務：

```shell
php artisan schedule:work
```


<a name="task-output"></a>
## 任務輸出

Laravel 排程器提供了幾種方便的方法來處理排程任務產生的輸出。首先，使用 `sendOutputTo` 方法，您可以將輸出發送到檔案以便日後檢查：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->daily()
    ->sendOutputTo($filePath);
```

如果您想將輸出附加到指定檔案，可以使用 `appendOutputTo` 方法：

```php
Schedule::command('emails:send')
    ->daily()
    ->appendOutputTo($filePath);
```

使用 `emailOutputTo` 方法，您可以將輸出發送到您選擇的電子郵件地址。在發送任務輸出郵件之前，您應該配置 Laravel 的 [電子郵件服務](/docs/{{version}}/mail)：

```php
Schedule::command('report:generate')
    ->daily()
    ->sendOutputTo($filePath)
    ->emailOutputTo('taylor@example.com');
```

如果您僅希望在排程的 Artisan 或系統命令以非零退出碼終止時才發送輸出郵件，請使用 `emailOutputOnFailure` 方法：

```php
Schedule::command('report:generate')
    ->daily()
    ->emailOutputOnFailure('taylor@example.com');
```

> [!WARNING]
> `emailOutputTo`、`emailOutputOnFailure`、`sendOutputTo` 和 `appendOutputTo` 方法僅限於 `command` 和 `exec` 方法使用。


<a name="task-hooks"></a>
## 任務鉤子

使用 `before` 和 `after` 方法，您可以指定在排程任務執行之前和之後要執行的程式碼：

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

`onSuccess` 和 `onFailure` 方法允許您指定在排程任務成功或失敗時要執行的程式碼。失敗表示排程的 Artisan 或系統命令以非零退出碼終止：

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

如果您的命令有輸出，您可以在 `after`、`onSuccess` 或 `onFailure` 鉤子中，透過將 `Illuminate\Support\Stringable` 實例指定為鉤子閉包定義的 `$output` 引數來存取它：

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

使用 `pingBefore` 和 `thenPing` 方法，排程器可以在任務執行之前或之後自動 ping 一個指定的 URL。此方法可用於通知外部服務（例如 [Envoyer](https://envoyer.io)），您的排程任務正準備開始或已完成執行：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingBefore($url)
    ->thenPing($url);
```

`pingOnSuccess` 和 `pingOnFailure` 方法可用於僅在任務成功或失敗時才 ping 指定的 URL。失敗表示排程的 Artisan 或系統命令以非零退出碼終止：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingOnSuccess($successUrl)
    ->pingOnFailure($failureUrl);
```

`pingBeforeIf`、`thenPingIf`、`pingOnSuccessIf` 和 `pingOnFailureIf` 方法可用於僅在給定條件為 `true` 時才 ping 指定的 URL：

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

Laravel 在排程過程中會發送多種 [事件](/docs/{{version}}/events)。您可以為以下任何事件 [定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| Event Name                                                  |
| ----------------------------------------------------------- |
| `Illuminate\Console\Events\ScheduledTaskStarting`           |
| `Illuminate\Console\Events\ScheduledTaskFinished`           |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` |
| `Illuminate\Console\Events\ScheduledTaskSkipped`            |
| `Illuminate\Console\Events\ScheduledTaskFailed`             |

</div>