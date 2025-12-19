# 任務排程

- [介紹](#introduction)
- [定義排程](#defining-schedules)
    - [排程 Artisan 命令](#scheduling-artisan-commands)
    - [排程佇列 Job](#scheduling-queued-jobs)
    - [排程 Shell 命令](#scheduling-shell-commands)
    - [排程頻率選項](#schedule-frequency-options)
    - [時區](#timezones)
    - [防止任務重疊](#preventing-task-overlaps)
    - [在單一伺服器上執行任務](#running-tasks-on-one-server)
    - [背景任務](#background-tasks)
    - [維護模式](#maintenance-mode)
    - [排程群組](#schedule-groups)
- [執行排程器](#running-the-scheduler)
    - [每分鐘多次執行的任務](#sub-minute-scheduled-tasks)
    - [在本機執行排程器](#running-the-scheduler-locally)
- [任務輸出](#task-output)
- [任務掛勾](#task-hooks)
- [事件](#events)

<a name="introduction"></a>
## 介紹

過去，您可能曾經為伺服器上需要排程的每個任務編寫一個 cron 設定條目。然而，這很快就會變得麻煩，因為您的任務排程不再受版本控制，而且您必須透過 SSH 登入您的伺服器才能查看現有的 cron 條目或新增額外的條目。

Laravel 的命令排程器提供了一種全新的方法來管理您伺服器上的排程任務。該排程器讓您能夠在 Laravel 應用程式中，流暢且清晰地定義您的命令排程。當使用排程器時，您的伺服器上只需要一個 cron 條目。您的任務排程通常定義在應用程式的 `routes/console.php` 檔案中。

<a name="defining-schedules"></a>
## 定義排程

您可以在應用程式的 `routes/console.php` 檔案中定義所有排程任務。首先，讓我們來看一個範例。在此範例中，我們將排程一個閉包，使其每天午夜執行。在該閉包內，我們將執行一個資料庫查詢來清空一個資料表：

```php
<?php

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->daily();
```

除了使用閉包進行排程外，您也可以排程 [可呼叫物件 (invokable objects)](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke)。可呼叫物件是包含 `__invoke` 方法的簡單 PHP 類別：

```php
Schedule::call(new DeleteRecentUsers)->daily();
```

如果您偏好將 `routes/console.php` 檔案僅保留給命令定義使用，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `withSchedule` 方法來定義您的排程任務。這個方法接受一個閉包，該閉包會接收排程器的一個實例：

```php
use Illuminate\Console\Scheduling\Schedule;

->withSchedule(function (Schedule $schedule) {
    $schedule->call(new DeleteRecentUsers)->daily();
})
```

如果您想查看排程任務的概覽以及它們下次的執行時間，您可以使用 `schedule:list` Artisan 命令：

```shell
php artisan schedule:list
```

<a name="scheduling-artisan-commands"></a>
### 排程 Artisan 命令

除了排程閉包之外，您還可以排程 [Artisan 命令](/docs/{{version}}/artisan)和系統命令。例如，您可以使用 `command` 方法，透過命令的名稱或類別來排程 Artisan 命令。

當使用命令的類別名稱排程 Artisan 命令時，您可以傳入一個額外的命令列參數陣列，這些參數會在命令被呼叫時提供給它：

```php
use App\Console\Commands\SendEmailsCommand;
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send Taylor --force')->daily();

Schedule::command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();
```

<a name="scheduling-artisan-closure-commands"></a>
#### 排程 Artisan 閉包命令

如果您想排程由閉包定義的 Artisan 命令，您可以在命令定義後串聯排程相關的方法：

```php
Artisan::command('delete:recent-users', function () {
    DB::table('recent_users')->delete();
})->purpose('Delete recent users')->daily();
```

如果您需要將參數傳遞給閉包命令，您可以將它們提供給 `schedule` 方法：

```php
Artisan::command('emails:send {user} {--force}', function ($user) {
    // ...
})->purpose('Send emails to the specified user')->schedule(['Taylor', '--force'])->daily();
```

<a name="scheduling-queued-jobs"></a>
### 排程佇列 Job

`job` 方法可用於排程 [佇列 Job](/docs/{{version}}/queues)。此方法提供了一種便捷的方式來排程佇列 Job，而無需使用 `call` 方法來定義排入佇列的閉包：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

Schedule::job(new Heartbeat)->everyFiveMinutes();
```

`job` 方法可以提供選用的第二和第三個參數，這些參數用於指定應該用來將 Job 排入佇列的佇列名稱和佇列連線：

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

我們已經看過一些範例，說明如何設定任務以在特定時間間隔執行。然而，您還可以為任務分配更多排程頻率：

<div class="overflow-auto">

| Method                             | Description                                              |
| :--------------------------------- | :------------------------------------------------------- |
| `->cron('* * * * *');`             | 依自訂的 cron 排程執行任務。                                            |
| `->everySecond();`                 | 每秒執行任務。                                                          |
| `->everyTwoSeconds();`             | 每兩秒執行任務。                                                        |
| `->everyFiveSeconds();`            | 每五秒執行任務。                                                        |
| `->everyTenSeconds();`             | 每十秒執行任務。                                                        |
| `->everyFifteenSeconds();`         | 每十五秒執行任務。                                                      |
| `->everyTwentySeconds();`          | 每二十秒執行任務。                                                      |
| `->everyThirtySeconds();`          | 每三十秒執行任務。                                                      |
| `->everyMinute();`                 | 每分鐘執行任務。                                                        |
| `->everyTwoMinutes();`             | 每兩分鐘執行任務。                                                      |
| `->everyThreeMinutes();`           | 每三分鐘執行任務。                                                      |
| `->everyFourMinutes();`            | 每四分鐘執行任務。                                                      |
| `->everyFiveMinutes();`            | 每五分鐘執行任務。                                                      |
| `->everyTenMinutes();`             | 每十分鐘執行任務。                                                      |
| `->everyFifteenMinutes();`         | 每十五分鐘執行任務。                                                    |
| `->everyThirtyMinutes();`          | 每三十分鐘執行任務。                                                    |
| `->hourly();`                      | 每小時執行任務。                                                        |
| `->hourlyAt(17);`                  | 每小時的第 17 分鐘執行任務。                                            |
| `->everyOddHour($minutes = 0);`    | 每奇數小時執行任務。                                                    |
| `->everyTwoHours($minutes = 0);`   | 每兩小時執行任務。                                                      |
| `->everyThreeHours($minutes = 0);` | 每三小時執行任務。                                                      |
| `->everyFourHours($minutes = 0);`  | 每四小時執行任務。                                                      |
| `->everySixHours($minutes = 0);`   | 每六小時執行任務。                                                      |
| `->daily();`                       | 每天午夜執行任務。                                                      |
| `->dailyAt('13:00');`              | 每天 13:00 執行任務。                                                   |
| `->twiceDaily(1, 13);`             | 每天 1:00 和 13:00 執行任務。                                           |
| `->twiceDailyAt(1, 13, 15);`       | 每天 1:15 和 13:15 執行任務。                                           |
| `->daysOfMonth([1, 10, 20]);`      | 在每月特定日期執行任務。                                                |
| `->weekly();`                      | 每週日的 00:00 執行任務。                                               |
| `->weeklyOn(1, '8:00');`           | 每週一的 8:00 執行任務。                                                |
| `->monthly();`                     | 每月的第一天 00:00 執行任務。                                           |
| `->monthlyOn(4, '15:00');`         | 每月 4 號的 15:00 執行任務。                                            |
| `->twiceMonthly(1, 16, '13:00');`  | 每月 1 號和 16 號的 13:00 執行任務。                                    |
| `->lastDayOfMonth('15:00');`       | 在每月最後一天 15:00 執行任務。                                         |
| `->quarterly();`                   | 每季度的第一天 00:00 執行任務。                                         |
| `->quarterlyOn(4, '14:00');`       | 每季度 4 號的 14:00 執行任務。                                          |
| `->yearly();`                      | 每年的第一天 00:00 執行任務。                                           |
| `->yearlyOn(6, 1, '17:00');`       | 每年 6 月 1 號的 17:00 執行任務。                                       |
| `->timezone('America/New_York');`  | 設定任務的時區。                                                        |

</div>

這些方法可以與額外的約束條件結合使用，以建立更精確的排程，使其僅在每週的特定日期執行。例如，您可以排程一個命令在每週一執行：

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

下面列出了額外的排程約束條件：

<div class="overflow-auto">

| Method                                   | Description                                            |
| :--------------------------------------- | :----------------------------------------------------- |
| `->weekdays();`                          | 將任務限制在工作日執行。                                          |
| `->weekends();`                          | 將任務限制在週末執行。                                            |
| `->sundays();`                           | 將任務限制在週日執行。                                            |
| `->mondays();`                           | 將任務限制在週一執行。                                            |
| `->tuesdays();`                          | 將任務限制在週二執行。                                            |
| `->wednesdays();`                        | 將任務限制在週三執行。                                            |
| `->thursdays();`                         | 將任務限制在週四執行。                                            |
| `->fridays();`                           | 將任務限制在週五執行。                                            |
| `->saturdays();`                         | 將任務限制在週六執行。                                            |
| `->days(array\|mixed);`                  | 將任務限制在特定日期執行。                                        |
| `->between($startTime, $endTime);`       | 將任務限制在開始時間與結束時間之間執行。                          |
| `->unlessBetween($startTime, $endTime);` | 將任務限制在開始時間與結束時間之間不執行。                        |
| `->when(Closure);`                       | 根據真值測試的結果限制任務。                                      |
| `->environments($env);`                  | 將任務限制在特定環境中執行。                                      |

</div>

<a name="day-constraints"></a>
#### 日期約束

`days` 方法可用於將任務的執行限制在每週的特定日期。例如，您可以排程一個命令在週日和週三每小時執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->hourly()
    ->days([0, 3]);
```

或者，您可以在定義任務應執行的日期時，使用 `Illuminate\Console\Scheduling\Schedule` 類別上可用的常數：

```php
use Illuminate\Support\Facades;
use Illuminate\Console\Scheduling\Schedule;

Facades\Schedule::command('emails:send')
    ->hourly()
    ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);
```

<a name="between-time-constraints"></a>
#### 時間區間約束

`between` 方法可用於根據一天中的時間限制任務的執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->between('7:00', '22:00');
```

同樣地，`unlessBetween` 方法可用於排除任務在某段時間內執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->unlessBetween('23:00', '4:00');
```

<a name="truth-test-constraints"></a>
#### 真值測試約束

`when` 方法可用於根據給定真值測試的結果來限制任務的執行。換句話說，如果給定的閉包返回 `true`，只要沒有其他約束條件阻止任務執行，任務就會執行：

```php
Schedule::command('emails:send')->daily()->when(function () {
    return true;
});
```

`skip` 方法可以看作是 `when` 方法的反向操作。如果 `skip` 方法返回 `true`，則排程任務將不會執行：

```php
Schedule::command('emails:send')->daily()->skip(function () {
    return true;
});
```

當使用鏈接的 `when` 方法時，排程命令只有在所有 `when` 條件都返回 `true` 時才會執行。

<a name="environment-constraints"></a>
#### 環境約束

`environments` 方法可用於僅在給定環境中執行任務（由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#environment-configuration) 定義）：

```php
Schedule::command('emails:send')
    ->daily()
    ->environments(['staging', 'production']);
```

<a name="timezones"></a>
### 時區

使用 `timezone` 方法，您可以指定排程任務的時間應在給定的時區內進行解讀：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->timezone('America/New_York')
    ->at('2:00')
```

如果您重複地將相同的時區分配給所有排程任務，您可以透過在應用程式的 `app` 設定檔中定義 `schedule_timezone` 選項，來指定所有排程都應分配的時區：

```php
'timezone' => 'UTC',

'schedule_timezone' => 'America/Chicago',
```

> [!WARNING]
> 請記住，有些時區會使用日光節約時間。當日光節約時間發生變化時，您的排程任務可能會執行兩次甚至完全不執行。因此，我們建議盡可能避免使用時區排程。


<a name="preventing-task-overlaps"></a>
### 防止任務重疊

預設情況下，即使先前的任務實例仍在執行，排程任務也會繼續執行。為了防止這種情況，您可以使用 `withoutOverlapping` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->withoutOverlapping();
```

在此範例中，如果 `emails:send` [Artisan 命令](/docs/{{version}}/artisan)尚未執行，它將每分鐘執行一次。 `withoutOverlapping` 方法特別有用，當您的任務執行時間差異很大，導致您無法精確預測特定任務將花費多長時間時，此方法能防止任務重疊。

如果需要，您可以指定「不重疊」鎖定過期前必須經過多少分鐘。預設情況下，鎖定將在 24 小時後過期：

```php
Schedule::command('emails:send')->withoutOverlapping(10);
```

在幕後，`withoutOverlapping` 方法利用您應用程式的 [快取](/docs/{{version}}/cache) 來取得鎖定。如有必要，您可以使用 `schedule:clear-cache` Artisan 命令來清除這些快取鎖定。這通常只在任務因意外的伺服器問題而卡住時才需要。


<a name="running-tasks-on-one-server"></a>
### 在單一伺服器上執行任務

> [!WARNING]
> 為了使用此功能，您的應用程式必須使用 `database`、`memcached`、`dynamodb` 或 `redis` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器進行通訊。

如果您的應用程式排程器在多個伺服器上執行，您可以限制排程 Job 只在單一伺服器上執行。例如，假設您有一個排程任務，它在每個星期五晚上產生一份新報告。如果任務排程器在三個 worker 伺服器上執行，則該排程任務將在所有三個伺服器上執行，並產生三次報告。這可不好！

若要指示任務只在單一伺服器上執行，請在定義排程任務時使用 `onOneServer` 方法。第一個取得任務的伺服器將會對該 Job 取得一個原子鎖定，以防止其他伺服器同時執行相同的任務：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->fridays()
    ->at('17:00')
    ->onOneServer();
```

您可以使用 `useCache` 方法來自訂排程器用於取得單一伺服器任務所需原子鎖定的快取儲存：

```php
Schedule::useCache('database');
```


<a name="naming-unique-jobs"></a>
#### 命名單一伺服器 Job

有時候，您可能需要排程相同的 Job 以不同的參數來分派，同時仍指示 Laravel 在單一伺服器上執行該 Job 的每個排列。為了實現這一點，您可以透過 `name` 方法為每個排程定義分配一個唯一的名稱：

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

同樣地，如果排程的閉包旨在單一伺服器上執行，則必須為其指定一個名稱：

```php
Schedule::call(fn () => User::resetApiRequestCount())
    ->name('reset-api-request-count')
    ->daily()
    ->onOneServer();
```


<a name="background-tasks"></a>
### 背景任務

預設情況下，同時排程的多個任務將根據其在 `schedule` 方法中定義的順序依序執行。如果您有長時間執行的任務，這可能會導致後續任務比預期晚得多才開始。如果您想在背景執行任務，以便它們可以同時執行，您可以使用 `runInBackground` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('analytics:report')
    ->daily()
    ->runInBackground();
```

> [!WARNING]
> `runInBackground` 方法只能在透過 `command` 和 `exec` 方法排程任務時使用。


<a name="maintenance-mode"></a>
### 維護模式

當應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，應用程式的排程任務將不會執行，因為我們不希望您的任務干擾您可能正在伺服器上執行的任何未完成的維護工作。然而，如果您想強制任務即使在維護模式下也要執行，您可以在定義任務時呼叫 `evenInMaintenanceMode` 方法：

```php
Schedule::command('emails:send')->evenInMaintenanceMode();
```


<a name="schedule-groups"></a>
### 排程群組

當定義具有相似設定的多個排程任務時，您可以使用 Laravel 的任務分組功能，以避免為每個任務重複相同的設定。任務分組簡化了您的程式碼，並確保相關任務之間的一致性。

若要建立一組排程任務，請呼叫所需的任務設定方法，接著是 `group` 方法。`group` 方法接受一個閉包，該閉包負責定義共享指定設定的任務：

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

現在我們已經學會如何定義排程任務，接下來我們來討論如何在伺服器上實際執行它們。`schedule:run` Artisan 命令將會評估所有排程任務，並根據伺服器的目前時間判斷是否需要執行。

因此，在使用 Laravel 的排程器時，我們只需要在伺服器上新增一個 cron 設定項目，讓它每分鐘執行一次 `schedule:run` 命令。如果您不知道如何在伺服器上新增 cron 項目，請考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣的託管平台，它可以為您管理排程任務的執行：

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

<a name="sub-minute-scheduled-tasks"></a>
### 每分鐘多次執行的任務

在大多數作業系統中，cron job 被限制為每分鐘最多執行一次。然而，Laravel 的排程器允許您以更頻繁的間隔排程任務，甚至可以每秒執行一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->everySecond();
```

當在應用程式中定義了每分鐘多次執行的任務時，`schedule:run` 命令將會持續執行直到當前分鐘結束，而不是立即退出。這使得該命令可以在一分鐘內呼叫所有必需的每分鐘多次執行的任務。

由於執行時間比預期更長的每分鐘多次執行的任務可能會延遲後續的每分鐘多次執行的任務的執行，建議所有每分鐘多次執行的任務都分派佇列 job 或背景命令來處理實際的任務處理：

```php
use App\Jobs\DeleteRecentUsers;

Schedule::job(new DeleteRecentUsers)->everyTenSeconds();

Schedule::command('users:delete')->everyTenSeconds()->runInBackground();
```

<a name="interrupting-sub-minute-tasks"></a>
#### 中斷每分鐘多次執行的任務

當定義了每分鐘多次執行的任務時，`schedule:run` 命令會執行整個呼叫分鐘，因此在部署應用程式時，您可能需要中斷該命令。否則，正在執行的 `schedule:run` 命令實例將會繼續使用應用程式先前部署的程式碼，直到當前分鐘結束。

若要中斷正在進行的 `schedule:run` 呼叫，您可以將 `schedule:interrupt` 命令新增到應用程式的部署腳本中。此命令應在應用程式部署完成後呼叫：

```shell
php artisan schedule:interrupt
```

<a name="running-the-scheduler-locally"></a>
### 在本機執行排程器

通常，您不會在本地開發機器上新增排程器 cron 項目。相反地，您可以使用 `schedule:work` Artisan 命令。此命令將在前景執行，並每分鐘呼叫排程器，直到您終止該命令。當定義了每分鐘多次執行的任務時，排程器將在每分鐘內持續執行以處理這些任務：

```shell
php artisan schedule:work
```

<a name="task-output"></a>
## 任務輸出

Laravel 排程器提供了幾種方便的方法來處理排程任務產生的輸出。首先，使用 `sendOutputTo` 方法，您可以將輸出發送到一個檔案，以便稍後檢查：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->daily()
    ->sendOutputTo($filePath);
```

如果您想將輸出附加到給定的檔案中，可以使用 `appendOutputTo` 方法：

```php
Schedule::command('emails:send')
    ->daily()
    ->appendOutputTo($filePath);
```

使用 `emailOutputTo` 方法，您可以將輸出傳送到您選擇的電子郵件地址。在透過電子郵件傳送任務輸出之前，您應該配置 Laravel 的[電子郵件服務](/docs/{{version}}/mail)：

```php
Schedule::command('report:generate')
    ->daily()
    ->sendOutputTo($filePath)
    ->emailOutputTo('taylor@example.com');
```

如果您只想在排程的 Artisan 或系統命令以非零結束碼終止時才透過電子郵件傳送輸出，請使用 `emailOutputOnFailure` 方法：

```php
Schedule::command('report:generate')
    ->daily()
    ->emailOutputOnFailure('taylor@example.com');
```

> [!WARNING]
> `emailOutputTo`、`emailOutputOnFailure`、`sendOutputTo` 和 `appendOutputTo` 方法僅適用於 `command` 和 `exec` 方法。

<a name="task-hooks"></a>
## 任務掛勾

使用 `before` 和 `after` 方法，您可以指定在排程任務執行之前和之後執行的程式碼：

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

`onSuccess` 和 `onFailure` 方法允許您指定在排程任務成功或失敗時執行的程式碼。失敗表示排程的 Artisan 或系統命令以非零結束碼終止：

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

如果命令有輸出，您可以透過在掛勾的 closure 定義中將 `Illuminate\Support\Stringable` 實例型別提示為 `$output` 參數，來在 `after`、`onSuccess` 或 `onFailure` 掛勾中存取它：

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
#### Ping 網址

使用 `pingBefore` 和 `thenPing` 方法，排程器可以在任務執行之前或之後自動 ping 給定的網址。此方法對於通知外部服務（例如 [Envoyer](https://envoyer.io)）您的排程任務正在開始或已完成執行非常有用：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingBefore($url)
    ->thenPing($url);
```

`pingOnSuccess` 和 `pingOnFailure` 方法可用於僅在任務成功或失敗時才 ping 給定的網址。失敗表示排程的 Artisan 或系統命令以非零結束碼終止：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingOnSuccess($successUrl)
    ->pingOnFailure($failureUrl);
```

`pingBeforeIf`、`thenPingIf`、`pingOnSuccessIf` 和 `pingOnFailureIf` 方法可用於僅在給定條件為 `true` 時才 ping 給定的網址：

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

Laravel 在排程過程中會分派各種[事件](/docs/{{version}}/events)。您可以為以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| Event Name                                                  |
| ----------------------------------------------------------- |
| `Illuminate\Console\Events\ScheduledTaskStarting`           |
| `Illuminate\Console\Events\ScheduledTaskFinished`           |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` |
| `Illuminate\Console\Events\ScheduledTaskSkipped`            |
| `Illuminate\Console\Events\ScheduledTaskFailed`             |

</div>