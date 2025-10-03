# 任務排程

-   [簡介](#introduction)
-   [定義排程](#defining-schedules)
    -   [排程 Artisan 指令](#scheduling-artisan-commands)
    -   [排程佇列 Jobs](#scheduling-queued-jobs)
    -   [排程 Shell 指令](#scheduling-shell-commands)
    -   [排程頻率選項](#schedule-frequency-options)
    -   [時區](#timezones)
    -   [避免任務重疊](#preventing-task-overlaps)
    -   [在單一伺服器上執行任務](#running-tasks-on-one-server)
    -   [背景任務](#background-tasks)
    -   [維護模式](#maintenance-mode)
    -   [排程群組](#schedule-groups)
-   [執行排程器](#running-the-scheduler)
    -   [秒級排程任務](#sub-minute-scheduled-tasks)
    -   [在本機執行排程器](#running-the-scheduler-locally)
-   [任務輸出](#task-output)
-   [任務鉤子](#task-hooks)
-   [事件](#events)

<a name="introduction"></a>
## 簡介

過去，你可能為伺服器上需要排程的每個任務寫過一個 cron 設定條目。然而，這很快就會變得麻煩，因為你的任務排程不再受版本控制，且你必須 SSH 進入你的伺服器才能查看現有的 cron 條目或新增額外的條目。

Laravel 的指令排程器為管理伺服器上的排程任務提供了一種全新的方法。該排程器讓你可以流暢且具表達力地在 Laravel 應用程式中定義指令排程。當使用該排程器時，你的伺服器上只需要一個單一的 cron 條目。你的任務排程通常會被定義在應用程式的 `routes/console.php` 檔案中。

<a name="defining-schedules"></a>
## 定義排程

您可以在應用程式的 `routes/console.php` 檔案中定義所有排程任務。為了開始，讓我們先看一個範例。在此範例中，我們將排程一個閉包，使其每天午夜執行。在此閉包中，我們將執行一個資料庫查詢來清空一個資料表：

```php
<?php

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->daily();
```

除了使用閉包進行排程，您也可以排程 [可呼叫物件](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke)。可呼叫物件是包含 `__invoke` 方法的簡單 PHP 類別：

```php
Schedule::call(new DeleteRecentUsers)->daily();
```

如果您偏好將 `routes/console.php` 檔案僅保留給指令定義使用，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `withSchedule` 方法來定義排程任務。這個方法接受一個接收排程器實例的閉包：

```php
use Illuminate\Console\Scheduling\Schedule;

->withSchedule(function (Schedule $schedule) {
    $schedule->call(new DeleteRecentUsers)->daily();
})
```

如果您想查看排程任務的概覽以及下次執行時間，您可以使用 `schedule:list` Artisan 指令：

```shell
php artisan schedule:list
```

<a name="scheduling-artisan-commands"></a>
### 排程 Artisan 指令

除了排程閉包，您也可以排程 [Artisan 指令](/docs/{{version}}/artisan) 和系統指令。例如，您可以使用 `command` 方法來排程 Artisan 指令，透過指令的名稱或類別。

當使用指令的類別名稱排程 Artisan 指令時，您可以傳遞一個包含額外命令列參數的陣列，這些參數應在指令被呼叫時提供給指令：

```php
use App\Console\Commands\SendEmailsCommand;
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send Taylor --force')->daily();

Schedule::command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();
```

<a name="scheduling-artisan-closure-commands"></a>
#### 排程 Artisan 閉包指令

如果您想排程一個由閉包定義的 Artisan 指令，您可以在指令定義之後鏈接排程相關的方法：

```php
Artisan::command('delete:recent-users', function () {
    DB::table('recent_users')->delete();
})->purpose('Delete recent users')->daily();
```

如果您需要將參數傳遞給閉包指令，您可以將它們提供給 `schedule` 方法：

```php
Artisan::command('emails:send {user} {--force}', function ($user) {
    // ...
})->purpose('Send emails to the specified user')->schedule(['Taylor', '--force'])->daily();
```

<a name="scheduling-queued-jobs"></a>
### 排程佇列 Jobs

`job` 方法可用於排程 [佇列 Job](/docs/{{version}}/queues)。這個方法提供了一種方便的方式來排程佇列 Jobs，而無需使用 `call` 方法來定義閉包以將 Job 加入佇列：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

Schedule::job(new Heartbeat)->everyFiveMinutes();
```

可以向 `job` 方法提供可選的第二和第三個參數，用於指定應將 Job 加入佇列的佇列名稱和佇列連線：

```php
use App\Jobs\Heartbeat;
use Illuminate\Support\Facades\Schedule;

// Dispatch the job to the "heartbeats" queue on the "sqs" connection...
Schedule::job(new Heartbeat, 'heartbeats', 'sqs')->everyFiveMinutes();
```

<a name="scheduling-shell-commands"></a>
### 排程 Shell 指令

`exec` 方法可用於向作業系統發出指令：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::exec('node /home/forge/script.js')->daily();
```

<a name="schedule-frequency-options"></a>
### 排程頻率選項

我們已經看過一些範例，說明如何設定任務以特定的間隔執行。然而，您還可以為任務指定更多任務排程頻率選項：

<div class="overflow-auto">

| 方法                             | 說明                                              |
| ---------------------------------- | -------------------------------------------------------- |
| `->cron('* * * * *');`             | 依自訂 cron 排程執行任務。                  |
| `->everySecond();`                 | 每秒執行任務。                               |
| `->everyTwoSeconds();`             | 每兩秒執行任務。                          |
| `->everyFiveSeconds();`            | 每五秒執行任務。                         |
| `->everyTenSeconds();`             | 每十秒執行任務。                          |
| `->everyFifteenSeconds();`         | 每十五秒執行任務。                      |
| `->everyTwentySeconds();`          | 每二十秒執行任務。                       |
| `->everyThirtySeconds();`          | 每三十秒執行任務。                       |
| `->everyMinute();`                 | 每分鐘執行任務。                               |
| `->everyTwoMinutes();`             | 每兩分鐘執行任務。                          |
| `->everyThreeMinutes();`           | 每三分鐘執行任務。                          |
| `->everyFourMinutes();`            | 每四分鐘執行任務。                          |
| `->everyFiveMinutes();`            | 每五分鐘執行任務。                         |
| `->everyTenMinutes();`             | 每十分鐘執行任務。                          |
| `->everyFifteenMinutes();`         | 每十五分鐘執行任務。                      |
| `->everyThirtyMinutes();`          | 每三十秒執行任務。                       |
| `->hourly();`                      | 每小時執行任務。                                 |
| `->hourlyAt(17);`                  | 每小時的第 17 分鐘執行任務。     |
| `->everyOddHour($minutes = 0);`    | 每隔一小時執行任務 (例如 1 點、3 點、5 點)。                             |
| `->everyTwoHours($minutes = 0);`   | 每兩小時執行任務。                            |
| `->everyThreeHours($minutes = 0);` | 每三小時執行任務。                          |
| `->everyFourHours($minutes = 0);`  | 每四小時執行任務。                           |
| `->everySixHours($minutes = 0);`   | 每六小時執行任務。                            |
| `->daily();`                       | 每天午夜執行任務。                      |
| `->dailyAt('13:00');`              | 每天 13:00 執行任務。                         |
| `->twiceDaily(1, 13);`             | 每天 1:00 和 13:00 執行任務。                      |
| `->twiceDailyAt(1, 13, 15);`       | 每天 1:15 和 13:15 執行任務。                      |
| `->weekly();`                      | 每週日 00:00 執行任務。                      |
| `->weeklyOn(1, '8:00');`           | 每週一 8:00 執行任務。               |
| `->monthly();`                     | 每月第一天 00:00 執行任務。   |
| `->monthlyOn(4, '15:00');`         | 每月 4 日 15:00 執行任務。            |
| `->twiceMonthly(1, 16, '13:00');`  | 每月 1 日和 16 日 13:00 執行任務。       |
| `->lastDayOfMonth('15:00');`       | 每月最後一天 15:00 執行任務。      |
| `->quarterly();`                   | 每季度第一天 00:00 執行任務。 |
| `->quarterlyOn(4, '14:00');`       | 每季度 4 日 14:00 執行任務。          |
| `->yearly();`                      | 每年第一天 00:00 執行任務。    |
| `->yearlyOn(6, 1, '17:00');`       | 每年 6 月 1 日 17:00 執行任務。            |
| `->timezone('America/New_York');`  | 為任務設定時區。                           |

</div>

這些方法可以與額外的約束條件結合，以建立更精確的排程，使其僅在每週的特定日子執行。例如，您可以排程一個指令，使其在每週一執行：

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

更多排程約束條件的列表如下：

<div class="overflow-auto">

| 方法                                   | 說明                                            |
| ---------------------------------------- | ------------------------------------------------------ |
| `->weekdays();`                          | 將任務限制在工作日執行。                            |
| `->weekends();`                          | 將任務限制在週末執行。                            |
| `->sundays();`                           | 將任務限制在週日執行。                              |
| `->mondays();`                           | 將任務限制在週一執行。                              |
| `->tuesdays();`                          | 將任務限制在週二執行。                             |
| `->wednesdays();`                        | 將任務限制在週三執行。                             |
| `->thursdays();`                         | 將任務限制在週四執行。                             |
| `->fridays();`                           | 將任務限制在週五執行。                              |
| `->saturdays();`                         | 將任務限制在週六執行。                             |
| `->days(array\|mixed);`                  | 將任務限制在特定日期執行。                       |
| `->between($startTime, $endTime);`       | 將任務限制在起始時間與結束時間之間執行。     |
| `->unlessBetween($startTime, $endTime);` | 將任務限制在起始時間與結束時間之間不執行。 |
| `->when(Closure);`                       | 依據真值測試結果限制任務。                  |
| `->environments($env);`                  | 將任務限制在特定環境中執行。               |

</div>

<a name="day-constraints"></a>
#### 日期約束條件

`days` 方法可用於將任務的執行限制在每週的特定日期。例如，您可以排程一個指令，使其在週日和週三每小時執行：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')
    ->hourly()
    ->days([0, 3]);
```

或者，您可以在定義任務應執行日期時，使用 `Illuminate\Console\Scheduling\Schedule` 類別中可用的常數：

```php
use Illuminate\Support\Facades;
use Illuminate\Console\Scheduling\Schedule;

Facades\Schedule::command('emails:send')
    ->hourly()
    ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);
```

<a name="between-time-constraints"></a>
#### 時間範圍約束條件

`between` 方法可用於依據每日時間限制任務的執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->between('7:00', '22:00');
```

同樣地，`unlessBetween` 方法可用於在特定時間段內排除任務的執行：

```php
Schedule::command('emails:send')
    ->hourly()
    ->unlessBetween('23:00', '4:00');
```

<a name="truth-test-constraints"></a>
#### 真值測試約束條件

`when` 方法可用於根據給定真值測試的結果來限制任務的執行。換句話說，如果給定的 closure 返回 `true`，任務將會執行，前提是沒有其他約束條件阻止任務執行：

```php
Schedule::command('emails:send')->daily()->when(function () {
    return true;
});
```

`skip` 方法可視為 `when` 的反向操作。如果 `skip` 方法返回 `true`，則排程任務將不會執行：

```php
Schedule::command('emails:send')->daily()->skip(function () {
    return true;
});
```

當使用鏈接的 `when` 方法時，排程的指令只有在所有 `when` 條件都返回 `true` 時才會執行。

<a name="environment-constraints"></a>
#### 環境約束條件

`environments` 方法可用於僅在給定的環境（由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#environment-configuration) 定義）中執行任務：

```php
Schedule::command('emails:send')
    ->daily()
    ->environments(['staging', 'production']);
```

<a name="timezones"></a>
### 時區

使用 `timezone` 方法，你可以指定一個排程任務的時間應在特定時區內解釋：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->timezone('America/New_York')
    ->at('2:00')
```

如果你重複為所有排程任務指定相同的時區，你可以在應用程式的 `app` 設定檔中定義 `schedule_timezone` 選項，以此來指定所有排程任務的時區：

```php
'timezone' => 'UTC',

'schedule_timezone' => 'America/Chicago',
```

> [!WARNING]
> 請記住，某些時區會使用日光節約時間。當日光節約時間發生變化時，你的排程任務可能會執行兩次，甚至完全不執行。因此，我們建議盡可能避免使用時區排程。

<a name="preventing-task-overlaps"></a>
### 避免任務重疊

預設情況下，即使任務的上一個實例仍在執行，排程任務也會繼續執行。為了防止這種情況，你可以使用 `withoutOverlapping` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->withoutOverlapping();
```

在此範例中，若 `emails:send` [Artisan 指令](/docs/{{version}}/artisan)尚未執行，它將每分鐘執行一次。如果你有執行時間差異很大的任務，`withoutOverlapping` 方法特別有用，因為它可以防止你精確預測特定任務將花費多長時間。

如果需要，你可以指定「無重疊 (without overlapping)」鎖定在多少分鐘後過期。預設情況下，該鎖定會在 24 小時後過期：

```php
Schedule::command('emails:send')->withoutOverlapping(10);
```

在幕後，`withoutOverlapping` 方法利用應用程式的 [快取](/docs/{{version}}/cache) 來取得鎖定。如有需要，你可以使用 `schedule:clear-cache` Artisan 指令來清除這些快取鎖定。這通常只在任務因伺服器意外問題而卡住時才需要。

<a name="running-tasks-on-one-server"></a>
### 在單一伺服器上執行任務

> [!WARNING]
> 要使用此功能，你的應用程式必須使用 `database`、`memcached`、`dynamodb` 或 `redis` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器通訊。

如果你的應用程式排程器正在多個伺服器上執行，你可以限制排程作業只在單一伺服器上執行。例如，假設你有一個排程任務，在每個週五晚上產生一份新報告。如果任務排程器在三台 worker 伺服器上執行，那麼排程任務將在所有三台伺服器上執行並產生三次報告。這可不好！

若要指示任務只在單一伺服器上執行，請在定義排程任務時使用 `onOneServer` 方法。第一個取得任務的伺服器將會對該作業取得原子鎖定，以防止其他伺服器同時執行相同的任務：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('report:generate')
    ->fridays()
    ->at('17:00')
    ->onOneServer();
```

你可以使用 `useCache` 方法來自訂排程器用於取得單一伺服器任務所需原子鎖定的快取儲存驅動器：

```php
Schedule::useCache('database');
```

<a name="naming-unique-jobs"></a>
#### 命名單一伺服器 Job

有時你可能需要排程相同的 Job 以不同的參數分派，同時仍指示 Laravel 在單一伺服器上執行該 Job 的每個變體。為了實現此目的，你可以透過 `name` 方法為每個排程定義指定一個獨特的名稱：

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

同樣地，排程閉包 (closures) 如果旨在單一伺服器上執行，則必須給予名稱：

```php
Schedule::call(fn () => User::resetApiRequestCount())
    ->name('reset-api-request-count')
    ->daily()
    ->onOneServer();
```

<a name="background-tasks"></a>
### 背景任務

預設情況下，同時排程的多個任務將根據它們在你 `schedule` 方法中的定義順序依序執行。如果你有長時間執行的任務，這可能會導致後續任務比預期晚得多才啟動。如果你想在背景中執行任務，讓它們可以同時執行，你可以使用 `runInBackground` 方法：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('analytics:report')
    ->daily()
    ->runInBackground();
```

> [!WARNING]
> `runInBackground` 方法僅在使用 `command` 和 `exec` 方法排程任務時才能使用。

<a name="maintenance-mode"></a>
### 維護模式

當應用程式處於 [維護模式](/docs/{{version}}/configuration#maintenance-mode) 時，你的應用程式排程任務將不會執行，因為我們不希望你的任務干擾你可能正在伺服器上執行的任何未完成的維護工作。但是，如果你想強制任務即使在維護模式下也能執行，你可以在定義任務時呼叫 `evenInMaintenanceMode` 方法：

```php
Schedule::command('emails:send')->evenInMaintenanceMode();
```

<a name="schedule-groups"></a>
### 排程群組

當定義多個具有相似設定的排程任務時，你可以使用 Laravel 的任務分組功能，以避免為每個任務重複相同的設定。任務分組可以簡化你的程式碼，並確保相關任務之間的一致性。

若要建立一組排程任務，請呼叫所需的任務設定方法，接著呼叫 `group` 方法。`group` 方法接受一個閉包 (closure)，該閉包負責定義共享指定設定的任務：

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

現在我們已經學會如何定義排程任務，接下來討論如何在伺服器上實際執行這些任務。`schedule:run` Artisan 指令會評估您所有的排程任務，並根據伺服器的當前時間判斷它們是否需要執行。

因此，當使用 Laravel 的排程器時，我們只需要在伺服器上新增一個單一的 cron 設定項目，讓它每分鐘執行一次 `schedule:run` 指令。如果您不知道如何在伺服器上新增 cron 項目，可以考慮使用像 [Laravel Cloud](https://cloud.laravel.com) 這樣的託管平台，它可以為您管理排程任務的執行：

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

<a name="sub-minute-scheduled-tasks"></a>
### 秒級排程任務

在大多數作業系統中，cron 作業最多只能每分鐘執行一次。然而，Laravel 的排程器允許您以更頻繁的間隔排程任務，甚至可以頻繁到每秒一次：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    DB::table('recent_users')->delete();
})->everySecond();
```

當您的應用程式中定義了秒級任務時，`schedule:run` 指令會持續執行直到當前分鐘結束，而不會立即退出。這使得該指令能夠在該分鐘內呼叫所有必要的秒級任務。

由於執行時間可能超出預期的秒級任務會延遲後續秒級任務的執行，因此建議所有秒級任務都分派佇列 Jobs 或背景指令來處理實際的任務處理：

```php
use App\Jobs\DeleteRecentUsers;

Schedule::job(new DeleteRecentUsers)->everyTenSeconds();

Schedule::command('users:delete')->everyTenSeconds()->runInBackground();
```

<a name="interrupting-sub-minute-tasks"></a>
#### 中斷秒級任務

由於定義秒級任務時，`schedule:run` 指令會在整個呼叫分鐘內執行，因此在部署您的應用程式時，您可能需要中斷此指令。否則，一個已經在執行的 `schedule:run` 指令實例將會繼續使用您應用程式先前部署的程式碼，直到當前分鐘結束。

要中斷正在進行的 `schedule:run` 呼叫，您可以將 `schedule:interrupt` 指令加入到您的應用程式部署腳本中。此指令應在您的應用程式部署完成後呼叫：

```shell
php artisan schedule:interrupt
```

<a name="running-the-scheduler-locally"></a>
### 在本機執行排程器

通常，您不會在您的本機開發機器上新增排程器 cron 項目。相反地，您可以使用 `schedule:work` Artisan 指令。此指令會在前景執行，並每分鐘呼叫排程器，直到您終止此指令。當定義了秒級任務時，排程器將在每一分鐘內持續執行以處理這些任務：

```shell
php artisan schedule:work
```

<a name="task-output"></a>
## 任務輸出

Laravel 排程器提供了幾種便利的方法來處理排程任務產生的輸出。首先，使用 `sendOutputTo` 方法，您可以將輸出發送到檔案中以便日後檢查：

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

使用 `emailOutputTo` 方法，您可以將輸出發送到您選擇的電子郵件地址。在透過電子郵件發送任務輸出之前，您應該設定 Laravel 的 [電子郵件服務](/docs/{{version}}/mail)：

```php
Schedule::command('report:generate')
    ->daily()
    ->sendOutputTo($filePath)
    ->emailOutputTo('taylor@example.com');
```

如果您只想在排程的 Artisan 或系統指令以非零的結束代碼終止時才透過電子郵件發送輸出，請使用 `emailOutputOnFailure` 方法：

```php
Schedule::command('report:generate')
    ->daily()
    ->emailOutputOnFailure('taylor@example.com');
```

> [!WARNING]
> `emailOutputTo`、`emailOutputOnFailure`、`sendOutputTo` 和 `appendOutputTo` 方法僅適用於 `command` 和 `exec` 方法。

<a name="task-hooks"></a>
## 任務鉤子

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

`onSuccess` 和 `onFailure` 方法允許您指定在排程任務成功或失敗時執行的程式碼。失敗表示排程的 Artisan 或系統指令以非零的結束代碼終止：

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

如果您的指令有輸出，您可以透過將 `Illuminate\Support\Stringable` 實例型別提示為鉤子閉包定義的 `$output` 參數來在 `after`、`onSuccess` 或 `onFailure` 鉤子中存取它：

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

使用 `pingBefore` 和 `thenPing` 方法，排程器可以在任務執行之前或之後自動 Ping 指定的 URL。此方法對於通知外部服務（例如 [Envoyer](https://envoyer.io)）您的排程任務已開始或已完成執行非常有用：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingBefore($url)
    ->thenPing($url);
```

`pingOnSuccess` 和 `pingOnFailure` 方法可以用來僅在任務成功或失敗時 Ping 指定的 URL。失敗表示排程的 Artisan 或系統指令以非零的結束代碼終止：

```php
Schedule::command('emails:send')
    ->daily()
    ->pingOnSuccess($successUrl)
    ->pingOnFailure($failureUrl);
```

`pingBeforeIf`、`thenPingIf`、`pingOnSuccessIf` 和 `pingOnFailureIf` 方法可以用來僅在給定條件為 `true` 時 Ping 指定的 URL：

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

Laravel 在排程過程中會分派各種 [事件](/docs/{{version}}/events)。您可以為以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱                                                    |
| :---------------------------------------------------------- |
| `Illuminate\Console\Events\ScheduledTaskStarting`           |
| `Illuminate\Console\Events\ScheduledTaskFinished`           |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` |
| `Illuminate\Console\Events\ScheduledTaskSkipped`            |
| `Illuminate\Console\Events\ScheduledTaskFailed`             |

</div>