# 任務排程

- [簡介](#introduction)
- [定義排程](#defining-schedules)
    - [排程 Artisan 命令](#scheduling-artisan-commands)
    - [排程佇列任務](#scheduling-queued-jobs)
    - [排程 Shell 命令](#scheduling-shell-commands)
    - [排程頻率選項](#schedule-frequency-options)
    - [時區](#timezones)
    - [避免任務重疊](#preventing-task-overlaps)
    - [在單一伺服器上執行任務](#running-tasks-on-one-server)
    - [背景任務](#background-tasks)
    - [維護模式](#maintenance-mode)
    - [排程群組](#schedule-groups)
- [執行排程器](#running-the-scheduler)
    - [每分鐘多次執行的排程任務](#sub-minute-scheduled-tasks)
    - [在本機執行排程器](#running-the-scheduler-locally)
- [任務輸出](#task-output)
- [任務掛鉤](#task-hooks)
- [事件](#events)

<a name="introduction"></a>
## 簡介

過去，您可能需要為伺服器上每個排程任務撰寫一個 cron 設定項目。然而，這很快就會變得麻煩，因為您的任務排程不再受原始碼控制，而且您必須透過 SSH 連線到伺服器才能查看現有的 cron 項目或新增其他項目。

Laravel 的命令排程器為管理伺服器上的排程任務提供了一種全新的方法。排程器讓您可以在 Laravel 應用程式中流暢且清晰地定義命令排程。使用排程器時，您的伺服器上只需要一個 cron 項目。您的任務排程通常定義在應用程式的 `routes/console.php` 檔案中。

<a name="defining-schedules"></a>
## 定義排程

您可以在應用程式的 `routes/console.php` 檔案中定義所有排程任務。首先，讓我們看一個範例。在此範例中，我們將排程一個閉包，使其每天午夜執行。在閉包中，我們將執行一個資料庫查詢來清除一個資料表：

    <?php

    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Facades\Schedule;

    Schedule::call(function () {
        DB::table('recent_users')->delete();
    })->daily();

除了使用閉包進行排程外，您還可以排程[可呼叫物件](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke)。可呼叫物件是包含 `__invoke` 方法的簡單 PHP 類別：

    Schedule::call(new DeleteRecentUsers)->daily();

如果您希望將 `routes/console.php` 檔案僅保留用於命令定義，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `withSchedule` 方法來定義排程任務。此方法接受一個閉包，該閉包會接收一個排程器實例：

    use Illuminate\Console\Scheduling\Schedule;

    ->withSchedule(function (Schedule $schedule) {
        $schedule->call(new DeleteRecentUsers)->daily();
    })

如果您想查看排程任務的概覽以及它們下次排程執行的時間，您可以使用 `schedule:list` Artisan 命令：

```bash
php artisan schedule:list
```

<a name="scheduling-artisan-commands"></a>
### 排程 Artisan 命令

除了排程閉包之外，您還可以排程 [Artisan 命令](/docs/{{version}}/artisan)和系統命令。例如，您可以使用 `command` 方法透過命令的名稱或類別來排程 Artisan 命令。

當使用命令的類別名稱排程 Artisan 命令時，您可以傳遞一個額外的命令列引數陣列，這些引數應在命令被呼叫時提供給命令：

    use App\Console\Commands\SendEmailsCommand;
    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send Taylor --force')->daily();

    Schedule::command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();

<a name="scheduling-artisan-closure-commands"></a>
#### 排程 Artisan 閉包命令

如果您想排程由閉包定義的 Artisan 命令，您可以在命令定義之後鏈接排程相關的方法：

    Artisan::command('delete:recent-users', function () {
        DB::table('recent_users')->delete();
    })->purpose('Delete recent users')->daily();

如果您需要將引數傳遞給閉包命令，您可以將它們提供給 `schedule` 方法：

    Artisan::command('emails:send {user} {--force}', function ($user) {
        // ...
    })->purpose('Send emails to the specified user')->schedule(['Taylor', '--force'])->daily();

<a name="scheduling-queued-jobs"></a>
### 排程佇列任務

`job` 方法可用於排程[佇列任務](/docs/{{version}}/queues)。此方法提供了一種方便的方式來排程佇列任務，而無需使用 `call` 方法來定義閉包以將任務排入佇列：

    use App\Jobs\Heartbeat;
    use Illuminate\Support\Facades\Schedule;

    Schedule::job(new Heartbeat)->everyFiveMinutes();

`job` 方法可以提供可選的第二個和第三個引數，用於指定應將任務排入佇列的佇列名稱和佇列連線：

    use App\Jobs\Heartbeat;
    use Illuminate\Support\Facades\Schedule;

    // 將任務分派到 "sqs" 連線上的 "heartbeats" 佇列...
    Schedule::job(new Heartbeat, 'heartbeats', 'sqs')->everyFiveMinutes();

<a name="scheduling-shell-commands"></a>
### 排程 Shell 命令

`exec` 方法可用於向作業系統發出命令：

    use Illuminate\Support\Facades\Schedule;

    Schedule::exec('node /home/forge/script.js')->daily();

<a name="schedule-frequency-options"></a>
### 排程頻率選項

我們已經看過一些如何將任務設定為在指定間隔執行的範例。然而，還有更多任務排程頻率可以分配給任務：

<div class="overflow-auto">

| 方法                             | 說明                                                     |
| ---------------------------------- | -------------------------------------------------------- |
| `->cron('* * * * *');`             | 根據自訂 cron 排程執行任務。                             |
| `->everySecond();`                 | 每秒執行任務。                                           |
| `->everyTwoSeconds();`             | 每兩秒執行任務。                                         |
| `->everyFiveSeconds();`            | 每五秒執行任務。                                         |
| `->everyTenSeconds();`             | 每十秒執行任務。                                         |
| `->everyFifteenSeconds();`         | 每十五秒執行任務。                                       |
| `->everyTwentySeconds();`          | 每二十秒執行任務。                                       |
| `->everyThirtySeconds();`          | 每三十秒執行任務。                                       |
| `->everyMinute();`                 | 每分鐘執行任務。                                         |
| `->everyTwoMinutes();`             | 每兩分鐘執行任務。                                       |
| `->everyThreeMinutes();`           | 每三分鐘執行任務。                                       |
| `->everyFourMinutes();`            | 每四分鐘執行任務。                                       |
| `->everyFiveMinutes();`            | 每五分鐘執行任務。                                       |
| `->everyTenMinutes();`             | 每十分鐘執行任務。                                       |
| `->everyFifteenMinutes();`         | 每十五分鐘執行任務。                                     |
| `->everyThirtyMinutes();`          | 每三十分鐘執行任務。                                     |
| `->hourly();`                      | 每小時執行任務。                                         |
| `->hourlyAt(17);`                  | 每小時的第 17 分鐘執行任務。                             |
| `->everyOddHour($minutes = 0);`    | 每奇數小時執行任務。                                     |
| `->everyTwoHours($minutes = 0);`   | 每兩小時執行任務。                                       |
| `->everyThreeHours($minutes = 0);` | 每三小時執行任務。                                       |
| `->everyFourHours($minutes = 0);`  | 每四小時執行任務。                                       |
| `->everySixHours($minutes = 0);`   | 每六小時執行任務。                                       |
| `->daily();`                       | 每天午夜執行任務。                                       |
| `->dailyAt('13:00');`              | 每天 13:00 執行任務。                                    |
| `->twiceDaily(1, 13);`             | 每天 1:00 和 13:00 執行任務。                            |
| `->twiceDailyAt(1, 13, 15);`       | 每天 1:15 和 13:15 執行任務。                            |
| `->weekly();`                      | 每週日 00:00 執行任務。                                  |
| `->weeklyOn(1, '8:00');`           | 每週一 8:00 執行任務。                                   |
| `->monthly();`                     | 每月第一天 00:00 執行任務。                              |
| `->monthlyOn(4, '15:00');`         | 每月 4 日 15:00 執行任務。                               |
| `->twiceMonthly(1, 16, '13:00');`  | 每月 1 日和 16 日 13:00 執行任務。                       |
| `->lastDayOfMonth('15:00');`       | 每月最後一天 15:00 執行任務。                            |
| `->quarterly();`                   | 每季第一天 00:00 執行任務。                              |
| `->quarterlyOn(4, '14:00');`       | 每季 4 日 14:00 執行任務。                               |
| `->yearly();`                      | 每年第一天 00:00 執行任務。                              |
| `->yearlyOn(6, 1, '17:00');`       | 每年 6 月 1 日 17:00 執行任務。                          |
| `->timezone('America/New_York');`  | 設定任務的時區。                                         |

</div>

這些方法可以與額外的限制條件結合，以建立更精確的排程，使其僅在特定星期幾執行。例如，您可以排程一個命令在每週一執行：

    use Illuminate\Support\Facades\Schedule;

    // 每週一 1 PM 執行一次...
    Schedule::call(function () {
        // ...
    })->weekly()->mondays()->at('13:00');

    // 工作日從上午 8 點到下午 5 點每小時執行一次...
    Schedule::command('foo')
        ->weekdays()
        ->hourly()
        ->timezone('America/Chicago')
        ->between('8:00', '17:00');

以下列出了其他排程限制條件：

<div class="overflow-auto">

| 方法                                   | 說明                                                     |
| ---------------------------------------- | ------------------------------------------------------ |
| `->weekdays();`                          | 將任務限制在工作日執行。                               |
| `->weekends();`                          | 將任務限制在週末執行。                                 |
| `->sundays();`                           | 將任務限制在星期日執行。                               |
| `->mondays();`                           | 將任務限制在星期一執行。                               |
| `->tuesdays();`                          | 將任務限制在星期二執行。                               |
| `->wednesdays();`                        | 將任務限制在星期三執行。                               |
| `->thursdays();`                         | 將任務限制在星期四執行。                               |
| `->fridays();`                           | 將任務限制在星期五執行。                               |
| `->saturdays();`                         | 將任務限制在星期六執行。                               |
| `->days(array\|mixed);`                  | 將任務限制在特定日期執行。                             |
| `->between($startTime, $endTime);`       | 將任務限制在開始時間和結束時間之間執行。               |
| `->unlessBetween($startTime, $endTime);` | 將任務限制在開始時間和結束時間之間不執行。             |
| `->when(Closure);`                       | 根據真值測試限制任務。                                 |
| `->environments($env);`                  | 將任務限制在特定環境中執行。                           |

</div>

<a name="day-constraints"></a>
#### 日期限制

`days` 方法可用於將任務的執行限制在特定的星期幾。例如，您可以排程一個命令在星期日和星期三每小時執行一次：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')
        ->hourly()
        ->days([0, 3]);

或者，您可以在定義任務應執行的日期時，使用 `Illuminate\Console\Scheduling\Schedule` 類別上可用的常數：

    use Illuminate\Support\Facades;
    use Illuminate\Console\Scheduling\Schedule;

    Facades\Schedule::command('emails:send')
        ->hourly()
        ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);

<a name="between-time-constraints"></a>
#### 時間區間限制

`between` 方法可用於根據一天中的時間限制任務的執行：

    Schedule::command('emails:send')
        ->hourly()
        ->between('7:00', '22:00');

同樣地，`unlessBetween` 方法可用於在一段時間內排除任務的執行：

    Schedule::command('emails:send')
        ->hourly()
        ->unlessBetween('23:00', '4:00');

<a name="truth-test-constraints"></a>
#### 真值測試限制

`when` 方法可用於根據給定真值測試的結果來限制任務的執行。換句話說，如果給定的閉包返回 `true`，只要沒有其他限制條件阻止任務執行，任務就會執行：

    Schedule::command('emails:send')->daily()->when(function () {
        return true;
    });

`skip` 方法可以看作是 `when` 的反向操作。如果 `skip` 方法返回 `true`，則排程任務將不會執行：

    Schedule::command('emails:send')->daily()->skip(function () {
        return true;
    });

當使用鏈接的 `when` 方法時，排程命令只有在所有 `when` 條件都返回 `true` 時才會執行。

<a name="environment-constraints"></a>
#### 環境限制

`environments` 方法可用於僅在給定環境中執行任務（由 `APP_ENV` [環境變數](/docs/{{version}}/configuration#environment-configuration)定義）：

    Schedule::command('emails:send')
        ->daily()
        ->environments(['staging', 'production']);

<a name="timezones"></a>
### 時區

使用 `timezone` 方法，您可以指定排程任務的時間應在給定時區內解釋：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('report:generate')
        ->timezone('America/New_York')
        ->at('2:00')

如果您重複為所有排程任務分配相同的時區，您可以透過在應用程式的 `app` 設定檔中定義 `schedule_timezone` 選項來指定應分配給所有排程的時區：

    'timezone' => 'UTC',

    'schedule_timezone' => 'America/Chicago',

> [!WARNING]  
> 請記住，某些時區會使用日光節約時間。當日光節約時間發生變化時，您的排程任務可能會執行兩次，甚至根本不執行。因此，我們建議盡可能避免時區排程。

<a name="preventing-task-overlaps"></a>
### 避免任務重疊

預設情況下，排程任務即使在前一個實例仍在執行時也會執行。為了防止這種情況，您可以使用 `withoutOverlapping` 方法：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')->withoutOverlapping();

在此範例中，如果 `emails:send` [Artisan 命令](/docs/{{version}}/artisan)尚未執行，它將每分鐘執行一次。`withoutOverlapping` 方法特別有用，如果您的任務執行時間差異很大，可以防止您精確預測給定任務將花費多長時間。

如果需要，您可以指定在「不重疊」鎖定過期之前必須經過多少分鐘。預設情況下，鎖定將在 24 小時後過期：

    Schedule::command('emails:send')->withoutOverlapping(10);

在幕後，`withoutOverlapping` 方法利用您應用程式的 [快取](/docs/{{version}}/cache)來取得鎖定。如有必要，您可以使用 `schedule:clear-cache` Artisan 命令清除這些快取鎖定。這通常僅在任務因意外的伺服器問題而卡住時才需要。

<a name="running-tasks-on-one-server"></a>
### 在單一伺服器上執行任務

> [!WARNING]  
> 要使用此功能，您的應用程式必須使用 `database`、`memcached`、`dynamodb` 或 `redis` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與相同的中央快取伺服器通訊。

如果您的應用程式排程器在多個伺服器上執行，您可以將排程任務限制為僅在單一伺服器上執行。例如，假設您有一個排程任務，每週五晚上產生一份新報告。如果任務排程器在三台工作伺服器上執行，則排程任務將在所有三台伺服器上執行並產生報告三次。這不好！

要指示任務應僅在單一伺服器上執行，請在定義排程任務時使用 `onOneServer` 方法。第一個取得任務的伺服器將取得任務的原子鎖定，以防止其他伺服器同時執行相同的任務：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('report:generate')
        ->fridays()
        ->at('17:00')
        ->onOneServer();

<a name="naming-unique-jobs"></a>
#### 命名單一伺服器任務

有時您可能需要排程相同的任務以不同的參數分派，同時仍指示 Laravel 在單一伺服器上執行每個任務排列。為此，您可以透過 `name` 方法為每個排程定義分配一個唯一的名稱：

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

同樣地，如果排程的閉包旨在在單一伺服器上執行，則必須為其分配一個名稱：

```php
Schedule::call(fn () => User::resetApiRequestCount())
    ->name('reset-api-request-count')
    ->daily()
    ->onOneServer();
```

<a name="background-tasks"></a>
### 背景任務

預設情況下，同時排程的多個任務將根據它們在 `schedule` 方法中定義的順序依序執行。如果您有長時間執行的任務，這可能會導致後續任務比預期晚得多才開始。如果您希望在背景執行任務，以便它們可以同時執行，您可以使用 `runInBackground` 方法：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('analytics:report')
        ->daily()
        ->runInBackground();

> [!WARNING]  
> `runInBackground` 方法只能在使用 `command` 和 `exec` 方法排程任務時使用。

<a name="maintenance-mode"></a>
### 維護模式

當應用程式處於[維護模式](/docs/{{version}}/configuration#maintenance-mode)時，您的應用程式排程任務將不會執行，因為我們不希望您的任務干擾您可能正在伺服器上執行的任何未完成的維護工作。但是，如果您想強制任務即使在維護模式下也執行，您可以在定義任務時呼叫 `evenInMaintenanceMode` 方法：

    Schedule::command('emails:send')->evenInMaintenanceMode();

<a name="schedule-groups"></a>
### 排程群組

當定義多個具有相似設定的排程任務時，您可以使用 Laravel 的任務分組功能來避免為每個任務重複相同的設定。分組任務可以簡化您的程式碼並確保相關任務之間的一致性。

要建立一組排程任務，請呼叫所需的任務設定方法，然後呼叫 `group` 方法。`group` 方法接受一個閉包，該閉包負責定義共用指定設定的任務：

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

現在我們已經學習了如何定義排程任務，接下來讓我們討論如何在伺服器上實際執行它們。`schedule:run` Artisan 命令將評估您的所有排程任務，並根據伺服器的當前時間判斷它們是否需要執行。

因此，當使用 Laravel 的排程器時，我們只需要在伺服器上新增一個 cron 設定項目，該項目每分鐘執行 `schedule:run` 命令。如果您不知道如何在伺服器上新增 cron 項目，請考慮使用 [Laravel Forge](https://forge.laravel.com) 等服務，它可以為您管理 cron 項目：

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

<a name="sub-minute-scheduled-tasks"></a>
### 每分鐘多次執行的排程任務

在大多數作業系統上，cron 任務的執行頻率最多限制為每分鐘一次。然而，Laravel 的排程器允許您排程任務以更頻繁的間隔執行，甚至每秒一次：

    use Illuminate\Support\Facades\Schedule;

    Schedule::call(function () {
        DB::table('recent_users')->delete();
    })->everySecond();

當應用程式中定義了每分鐘多次執行的任務時，`schedule:run` 命令將繼續執行直到當前分鐘結束，而不是立即退出。這允許命令在整個分鐘內呼叫所有必需的每分鐘多次執行的任務。

由於執行時間超出預期的每分鐘多次執行的任務可能會延遲後續每分鐘多次執行的任務的執行，因此建議所有每分鐘多次執行的任務都分派佇列任務或背景命令來處理實際的任務處理：

    use App\Jobs\DeleteRecentUsers;

    Schedule::job(new DeleteRecentUsers)->everyTenSeconds();

    Schedule::command('users:delete')->everyTenSeconds()->runInBackground();

<a name="interrupting-sub-minute-tasks"></a>
#### 中斷每分鐘多次執行的任務

由於當定義了每分鐘多次執行的任務時，`schedule:run` 命令會在呼叫的整個分鐘內執行，因此您有時可能需要在部署應用程式時中斷該命令。否則，一個已經在執行的 `schedule:run` 命令實例將繼續使用您應用程式先前部署的程式碼，直到當前分鐘結束。

要中斷正在進行的 `schedule:run` 呼叫，您可以將 `schedule:interrupt` 命令新增到應用程式的部署腳本中。此命令應在應用程式部署完成後呼叫：

```shell
php artisan schedule:interrupt
```

<a name="running-the-scheduler-locally"></a>
### 在本機執行排程器

通常，您不會將排程器 cron 項目新增到您的本機開發機器。相反，您可以使用 `schedule:work` Artisan 命令。此命令將在前台執行，並每分鐘呼叫排程器，直到您終止該命令。當定義了每分鐘多次執行的任務時，排程器將在每分鐘內繼續執行以處理這些任務：

```shell
php artisan schedule:work
```

<a name="task-output"></a>
## 任務輸出

Laravel 排程器提供了幾種方便的方法來處理排程任務產生的輸出。首先，使用 `sendOutputTo` 方法，您可以將輸出傳送到檔案以供稍後檢查：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')
        ->daily()
        ->sendOutputTo($filePath);

如果您想將輸出附加到給定檔案，您可以使用 `appendOutputTo` 方法：

    Schedule::command('emails:send')
        ->daily()
        ->appendOutputTo($filePath);

使用 `emailOutputTo` 方法，您可以將輸出透過電子郵件傳送到您選擇的電子郵件地址。在透過電子郵件傳送任務輸出之前，您應該設定 Laravel 的[電子郵件服務](/docs/{{version}}/mail)：

    Schedule::command('report:generate')
        ->daily()
        ->sendOutputTo($filePath)
        ->emailOutputTo('taylor @example.com');

如果您只想在排程的 Artisan 或系統命令以非零結束代碼終止時才透過電子郵件傳送輸出，請使用 `emailOutputOnFailure` 方法：

    Schedule::command('report:generate')
        ->daily()
        ->emailOutputOnFailure('taylor @example.com');

> [!WARNING]  
> `emailOutputTo`、`emailOutputOnFailure`、`sendOutputTo` 和 `appendOutputTo` 方法僅適用於 `command` 和 `exec` 方法。

<a name="task-hooks"></a>
## 任務掛鉤

使用 `before` 和 `after` 方法，您可以指定在排程任務執行之前和之後執行的程式碼：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')
        ->daily()
        ->before(function () {
            // 任務即將執行...
        })
        ->after(function () {
            // 任務已執行...
        });

`onSuccess` 和 `onFailure` 方法允許您指定在排程任務成功或失敗時執行的程式碼。失敗表示排程的 Artisan 或系統命令以非零結束代碼終止：

    Schedule::command('emails:send')
        ->daily()
        ->onSuccess(function () {
            // 任務成功...
        })
        ->onFailure(function () {
            // 任務失敗...
        });

如果命令有輸出，您可以透過在掛鉤閉包定義中將 `Illuminate\Support\Stringable` 實例類型提示為 `$output` 引數來存取它：

    use Illuminate\Support\Stringable;

    Schedule::command('emails:send')
        ->daily()
        ->onSuccess(function (Stringable $output) {
            // 任務成功...
        })
        ->onFailure(function (Stringable $output) {
            // 任務失敗...
        });

<a name="pinging-urls"></a>
#### Ping URL

使用 `pingBefore` 和 `thenPing` 方法，排程器可以在任務執行之前或之後自動 ping 給定的 URL。此方法對於通知外部服務（例如 [Envoyer](https://envoyer.io)）您的排程任務正在開始或已完成執行非常有用：

    Schedule::command('emails:send')
        ->daily()
        ->pingBefore($url)
        ->thenPing($url);

`pingOnSuccess` 和 `pingOnFailure` 方法可用於僅在任務成功或失敗時 ping 給定的 URL。失敗表示排程的 Artisan 或系統命令以非零結束代碼終止：

    Schedule::command('emails:send')
        ->daily()
        ->pingOnSuccess($successUrl)
        ->pingOnFailure($failureUrl);

`pingBeforeIf`、`thenPingIf`、`pingOnSuccessIf` 和 `pingOnFailureIf` 方法可用於僅在給定條件為 `true` 時 ping 給定的 URL：

    Schedule::command('emails:send')
        ->daily()
        ->pingBeforeIf($condition, $url)
        ->thenPingIf($condition, $url);             

    Schedule::command('emails:send')
        ->daily()
        ->pingOnSuccessIf($condition, $successUrl)
        ->pingOnFailureIf($condition, $failureUrl);

<a name="events"></a>
## 事件

Laravel 在排程過程中會分派各種[事件](/docs/{{version}}/events)。您可以為以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱 |
| --- |
| `Illuminate\Console\Events\ScheduledTaskStarting` |
| `Illuminate\Console\Events\ScheduledTaskFinished` |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` |
| `Illuminate\Console\Events\ScheduledTaskSkipped` |
| `Illuminate\Console\Events\ScheduledTaskFailed` |

</div>
