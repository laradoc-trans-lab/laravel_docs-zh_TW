# 行程(Processes)

- [簡介](#introduction)
- [呼叫行程](#invoking-processes)
    - [行程選項](#process-options)
    - [行程輸出](#process-output)
    - [管道](#process-pipelines)
- [非同步行程](#asynchronous-processes)
    - [行程 ID 與訊號](#process-ids-and-signals)
    - [非同步行程輸出](#asynchronous-process-output)
    - [非同步行程逾時](#asynchronous-process-timeouts)
- [同時執行多個行程](#concurrent-processes)
    - [命名行程池中的行程](#naming-pool-processes)
    - [行程池 ID 與訊號](#pool-process-ids-and-signals)
- [測試](#testing)
    - [偽造行程](#faking-processes)
    - [偽造特定行程](#faking-specific-processes)
    - [偽造行程序列](#faking-process-sequences)
    - [偽造非同步行程生命週期](#faking-asynchronous-process-lifecycles)
    - [可用的斷言](#available-assertions)
    - [防止遺漏未偽造的行程](#preventing-stray-processes)

<a name="introduction"></a>
## 簡介

Laravel 針對 [Symfony Process 元件](https://symfony.com/doc/current/components/process.html) 提供了一個表達力極佳且精簡的 API，讓您可以方便地從 Laravel 應用程式中呼叫外部行程。Laravel 的行程功能專注於最常見的用途以及極佳的開發者體驗。


<a name="invoking-processes"></a>
## 呼叫行程

要呼叫行程，您可以使用 `Process` Facade 提供的 `run` 與 `start` 方法。`run` 方法會呼叫行程並等待行程執行完畢，而 `start` 方法則用於非同步執行行程。我們將在這份文件中檢視這兩種方法。首先，讓我們看看如何呼叫一個基礎的同步行程並檢查其結果：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

return $result->output();
```

當然，`run` 方法返回的 `Illuminate\Contracts\Process\ProcessResult` 實例提供了許多好用的方法，可用於檢查行程結果：

```php
$result = Process::run('ls -la');

$result->command();
$result->successful();
$result->failed();
$result->output();
$result->errorOutput();
$result->exitCode();
```


<a name="throwing-exceptions"></a>
#### 拋出例外

如果您取得行程結果後，希望在結束代碼 (Exit code) 大於零（代表失敗）時拋出 `Illuminate\Process\Exceptions\ProcessFailedException` 實例，您可以使用 `throw` 及 `throwIf` 方法。若行程沒有失敗，則會返回 `ProcessResult` 實例：

```php
$result = Process::run('ls -la')->throw();

$result = Process::run('ls -la')->throwIf($condition);
```


<a name="process-options"></a>
### 行程選項

當然，您可能需要在呼叫行程之前自訂其行為。幸運的是，Laravel 允許您微調各種行程功能，例如工作目錄、逾時時間以及環境變數。


<a name="working-directory-path"></a>
#### 工作目錄路徑

您可以使用 `path` 方法來指定行程的工作目錄。如果不呼叫此方法，行程將繼承目前正在執行的 PHP 腳本的工作目錄：

```php
$result = Process::path(__DIR__)->run('ls -la');
```


<a name="input"></a>
#### 輸入

您可以使用 `input` 方法經由行程的「標準輸入 (Standard input)」提供輸入：

```php
$result = Process::input('Hello World')->run('cat');
```


<a name="timeouts"></a>
#### 逾時

預設情況下，當行程執行超過 60 秒時，會拋出 `Illuminate\Process\Exceptions\ProcessTimedOutException` 實例。不過，您可以透過 `timeout` 方法自訂此行為：

```php
$result = Process::timeout(120)->run('bash import.sh');
```

`timeout` 與 `idleTimeout` 方法也接受 `CarbonInterval` 實例：

```php
use function Illuminate\Support\minutes;

$result = Process::timeout(minutes(2))->run('bash import.sh');
```

或者，如果您想要完全停用行程逾時，可以呼叫 `forever` 方法：

```php
$result = Process::forever()->run('bash import.sh');
```

`idleTimeout` 方法可用於指定行程在未返回任何輸出的情況下可執行的最大秒數：

```php
$result = Process::timeout(60)->idleTimeout(30)->run('bash import.sh');
```


<a name="environment-variables"></a>
#### 環境變數

環境變數可以透過 `env` 方法提供給行程。被呼叫的行程也會繼承系統中所定義的所有環境變數：

```php
$result = Process::forever()
    ->env(['IMPORT_PATH' => __DIR__])
    ->run('bash import.sh');
```

如果您希望從被呼叫的行程中移除某個繼承的環境變數，可以將該環境變數的值設為 `false`：

```php
$result = Process::forever()
    ->env(['LOAD_PATH' => false])
    ->run('bash import.sh');
```


<a name="tty-mode"></a>
#### TTY 模式

`tty` 方法可用於為您的行程啟用 TTY 模式。TTY 模式會將行程的輸入與輸出連接到您程式的輸入與輸出，允許您的行程開啟像 Vim 或 Nano 這類的編輯器：

```php
Process::forever()->tty()->run('vim');
```

> [!WARNING]
> Windows 不支援 TTY 模式。


<a name="process-output"></a>
### 行程輸出

如前所述，可以使用行程結果上的 `output` (stdout) 與 `errorOutput` (stderr) 方法存取行程輸出：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

然而，也可以透過傳遞 Closure 作為 `run` 方法的第二個引數來即時蒐集輸出。該 Closure 將接收兩個引數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$result = Process::run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Laravel 還提供了 `seeInOutput` 與 `seeInErrorOutput` 方法，這提供了一種便捷的方式來判斷行程輸出中是否包含指定的字串：

```php
if (Process::run('ls -la')->seeInOutput('laravel')) {
    // ...
}
```


<a name="disabling-process-output"></a>
#### 停用行程輸出

如果您的行程會寫入您不感興趣的大量輸出，您可以透過完全停用輸出存取來節省記憶體。要達成此目的，請在建置行程時呼叫 `quietly` 方法：

```php
use Illuminate\Support\Facades\Process;

$result = Process::quietly()->run('bash import.sh');
```


<a name="process-pipelines"></a>
### 管道

有時候您可能希望將一個行程的輸出作為另一個行程的輸入。這通常被稱為將行程的輸出「透過管道傳送 (Piping)」給另一個行程。`Process` Facade 提供的 `pipe` 方法讓您可以輕鬆完成此任務。`pipe` 方法將同步執行管道中的行程，並返回管道中最後一個行程的行程結果：

```php
use Illuminate\Process\Pipe;
use Illuminate\Support\Facades\Process;

$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
});

if ($result->successful()) {
    // ...
}
```

如果您不需要自訂組成管道的個別行程，您可以直接傳送一個命令字串陣列給 `pipe` 方法：

```php
$result = Process::pipe([
    'cat example.txt',
    'grep -i "laravel"',
]);
```

也可以透過傳遞 Closure 作為 `pipe` 方法的第二個引數來即時蒐集行程輸出。該 Closure 將接收兩個引數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
}, function (string $type, string $output) {
    echo $output;
});
```

Laravel 還允許您透過 `as` 方法在管道內為每個行程指派字串鍵 (Keys)。此鍵也將傳遞給提供給 `pipe` 方法的輸出 Closure，讓您可以判斷輸出屬於哪個行程：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->as('first')->command('cat example.txt');
    $pipe->as('second')->command('grep -i "laravel"');
}, function (string $type, string $output, string $key) {
    // ...
});
```

<a name="asynchronous-processes"></a>
## 非同步行程

雖然 `run` 方法是同步呼叫行程，但可以使用 `start` 方法來非同步地呼叫行程。這允許您的應用程式在背景執行行程時，繼續執行其他任務。一旦呼叫了行程，您可以使用 `running` 方法來確定該行程是否仍在執行中：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    // ...
}

$result = $process->wait();
```

您可能已經注意到，您可以呼叫 `wait` 方法來等待行程執行完成，並取得 `ProcessResult` 實例：

```php
$process = Process::timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```


<a name="process-ids-and-signals"></a>
### 行程 ID 與訊號

`id` 方法可以用於取得作業系統分配給正在執行的行程之行程 ID：

```php
$process = Process::start('bash import.sh');

return $process->id();
```

您可以使用 `signal` 方法發送「訊號」給正在執行的行程。預定義的訊號常數清單可以在 [PHP 官方文件](https://www.php.net/manual/en/pcntl.constants.php)中找到：

```php
$process->signal(SIGUSR2);
```


<a name="asynchronous-process-output"></a>
### 非同步行程輸出

當非同步行程正在執行時，您可以使用 `output` 與 `errorOutput` 方法來存取其目前完整的輸出；然而，您也可以利用 `latestOutput` 與 `latestErrorOutput` 來存取自上次擷取輸出以來所產生的行程輸出：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

就像 `run` 方法一樣，也可以透過將閉包做為第二個引數傳遞給 `start` 方法，即時收集來自非同步行程的輸出。該閉包將接收兩個引數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$process = Process::start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

您也可以使用 `waitUntil` 方法根據行程的輸出來停止等待，而不是一直等到行程結束。當傳給 `waitUntil` 方法的閉包傳回 `true` 時，Laravel 就會停止等待行程結束：

```php
$process = Process::start('bash import.sh');

$process->waitUntil(function (string $type, string $output) {
    return $output === 'Ready...';
});
```


<a name="asynchronous-process-timeouts"></a>
### 非同步行程逾時

當非同步行程正在執行時，您可以使用 `ensureNotTimedOut` 方法來驗證行程是否尚未逾時。如果行程已經逾時，此方法將拋出[逾時例外](#timeouts)：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    $process->ensureNotTimedOut();

    // ...

    sleep(1);
}
```


<a name="concurrent-processes"></a>
## 同時執行多個行程

Laravel 還能讓您輕鬆管理並行非同步行程的行程池，讓您可以輕鬆同時執行許多任務。首先，呼叫 `pool` 方法，該方法接受一個接收 `Illuminate\Process\Pool` 實例的閉包。

在此閉包中，您可以定義屬於該行程池的行程。一旦透過 `start` 方法啟動行程池後，您就可以透過 `running` 方法存取正在執行的行程[集合](/docs/{{version}}/collections)：

```php
use Illuminate\Process\Pool;
use Illuminate\Support\Facades\Process;

$pool = Process::pool(function (Pool $pool) {
    $pool->path(__DIR__)->command('bash import-1.sh');
    $pool->path(__DIR__)->command('bash import-2.sh');
    $pool->path(__DIR__)->command('bash import-3.sh');
})->start(function (string $type, string $output, int $key) {
    // ...
});

while ($pool->running()->isNotEmpty()) {
    // ...
}

$results = $pool->wait();
```

如您所見，您可以透過 `wait` 方法等待行程池中的所有行程執行完成並解析其結果。`wait` 方法傳回一個支援陣列存取的物件，允許您透過鍵名存取行程池中每個行程的 `ProcessResult` 實例：

```php
$results = $pool->wait();

echo $results[0]->output();
```

或者為了方便起見，可以使用 `concurrently` 方法啟動非同步行程池並立即等待其結果。結合 PHP 的陣列解構功能時，這能提供特別具表達力的語法：

```php
[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```


<a name="naming-pool-processes"></a>
### 命名行程池中的行程

透過數字鍵名存取行程池結果並不夠直觀具體；因此，Laravel 允許您透過 `as` 方法為行程池中的每個行程指定字串鍵名。這個鍵名也會傳遞給提供給 `start` 方法的閉包，讓您能夠判斷該輸出屬於哪一個行程：

```php
$pool = Process::pool(function (Pool $pool) {
    $pool->as('first')->command('bash import-1.sh');
    $pool->as('second')->command('bash import-2.sh');
    $pool->as('third')->command('bash import-3.sh');
})->start(function (string $type, string $output, string $key) {
    // ...
});

$results = $pool->wait();

return $results['first']->output();
```


<a name="pool-process-ids-and-signals"></a>
### 行程池 ID 與訊號

由於行程池的 `running` 方法提供了行程池內所有被呼叫行程的集合，您可以輕鬆地存取底層行程池中的行程 ID：

```php
$processIds = $pool->running()->each->id();
```

而且，為了方便起見，您可以對行程池呼叫 `signal` 方法，向行程池內的每一個行程發送訊號：

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了能幫助您輕鬆且具表達力地撰寫測試的功能，Laravel 的行程服務也不例外。`Process` Facade 的 `fake` 方法允許您指示 Laravel 在呼叫行程時傳回虛擬 / 模擬（stubbed / dummy）的結果。


<a name="faking-processes"></a>
### 偽造行程

為了探索 Laravel 偽造行程的能力，讓我們想像有一個呼叫行程的路由：

```php
use Illuminate\Support\Facades\Process;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    Process::run('bash import.sh');

    return 'Import complete!';
});
```

測試此路由時，我們可以透過呼叫不帶引數的 `Process` Facade 的 `fake` 方法，來指示 Laravel 為每個被呼叫的行程傳回偽造且成功的行程結果。此外，我們甚至可以[斷言](#available-assertions)給定的行程已經「執行」：

```php tab=Pest
<?php

use Illuminate\Contracts\Process\ProcessResult;
use Illuminate\Process\PendingProcess;
use Illuminate\Support\Facades\Process;

test('process is invoked', function () {
    Process::fake();

    $response = $this->get('/import');

    // Simple process assertion...
    Process::assertRan('bash import.sh');

    // Or, inspecting the process configuration...
    Process::assertRan(function (PendingProcess $process, ProcessResult $result) {
        return $process->command === 'bash import.sh' &&
               $process->timeout === 60;
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Contracts\Process\ProcessResult;
use Illuminate\Process\PendingProcess;
use Illuminate\Support\Facades\Process;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_process_is_invoked(): void
    {
        Process::fake();

        $response = $this->get('/import');

        // Simple process assertion...
        Process::assertRan('bash import.sh');

        // Or, inspecting the process configuration...
        Process::assertRan(function (PendingProcess $process, ProcessResult $result) {
            return $process->command === 'bash import.sh' &&
                   $process->timeout === 60;
        });
    }
}
```

如前所述，呼叫 `Process` Facade 上的 `fake` 方法將指示 Laravel 始終傳回沒有輸出的成功行程結果。不過，您可以使用 `Process` Facade 的 `result` 方法輕鬆指定偽造行程的輸出與結束代碼：

```php
Process::fake([
    '*' => Process::result(
        output: 'Test output',
        errorOutput: 'Test error output',
        exitCode: 1,
    ),
]);
```


<a name="faking-specific-processes"></a>
### 偽造特定行程

正如您在之前的範例中所注意到的，`Process` Facade 允許您透過向 `fake` 方法傳遞陣列，來為每個行程指定不同的偽造結果。

陣列的鍵應代表您希望偽造的指令模式及其相關結果。可以使用 `*` 字元作為萬用字元。任何未被偽造的行程指令實際上都會被呼叫執行。您可以使用 `Process` Facade 的 `result` 方法為這些指令建構虛擬 / 偽造結果：

```php
Process::fake([
    'cat *' => Process::result(
        output: 'Test "cat" output',
    ),
    'ls *' => Process::result(
        output: 'Test "ls" output',
    ),
]);
```

若您不需要自訂偽造行程的結束代碼或錯誤輸出，可能會發現將偽造行程結果指定為簡單的字串會更方便：

```php
Process::fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```


<a name="faking-process-sequences"></a>
### 偽造行程序列

若您正在測試的程式碼使用相同的指令呼叫了多個行程，您可能希望為每次行程呼叫分配不同的偽造行程結果。您可以透過 `Process` Facade 的 `sequence` 方法來實現此目的：

```php
Process::fake([
    'ls *' => Process::sequence()
        ->push(Process::result('First invocation'))
        ->push(Process::result('Second invocation')),
]);
```


<a name="faking-asynchronous-process-lifecycles"></a>
### 偽造非同步行程生命週期

到目前為止，我們主要討論了偽造使用 `run` 方法同步呼叫的行程。但是，如果您嘗試測試與透過 `start` 呼叫的非同步行程互動的程式碼，您可能需要更複雜的方法來描述您的偽造行程。

例如，讓我們想像以下與非同步行程互動的路由：

```php
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    $process = Process::start('bash import.sh');

    while ($process->running()) {
        Log::info($process->latestOutput());
        Log::info($process->latestErrorOutput());
    }

    return 'Done';
});
```

為了正確地偽造此行程，我們需要能夠描述 `running` 方法應該傳回 `true` 的次數。此外，我們可能希望指定應按順序傳回的多行輸出。為了實現這一點，我們可以使用 `Process` Facade 的 `describe` 方法：

```php
Process::fake([
    'bash import.sh' => Process::describe()
        ->output('First line of standard output')
        ->errorOutput('First line of error output')
        ->output('Second line of standard output')
        ->exitCode(0)
        ->iterations(3),
]);
```

讓我們深入探討上面的範例。使用 `output` 和 `errorOutput` 方法，我們可以指定將依序傳回的多行輸出。`exitCode` 方法可用於指定偽造行程的最終結束代碼。最後，`iterations` 方法可用於指定 `running` 方法應該傳回 `true` 的次數。

<a name="available-assertions"></a>
### 可用的斷言

如[先前所述](#faking-processes)，Laravel 為您的功能測試提供了多種行程斷言。我們將在下方討論其中的每個斷言。

<a name="assert-process-ran"></a>
#### assertRan

斷言指定的行程已被呼叫：

```php
use Illuminate\Support\Facades\Process;

Process::assertRan('ls -la');
```

當行程是使用引數陣列呼叫時，您可以將相同的陣列傳遞給斷言：

```php
Process::assertRan(['php', 'artisan', 'migrate']);
```

`assertRanTimes` 與 `assertDidntRun` 方法也支援陣列形式的命令。

`assertRan` 方法也接受一個閉包，該閉包將接收行程實例與行程結果，讓您可以檢查行程設定的選項。如果此閉包回傳 `true`，則斷言將「通過」：

```php
Process::assertRan(fn ($process, $result) =>
    $process->command === 'ls -la' &&
    $process->path === __DIR__ &&
    $process->timeout === 60
);
```

傳遞給 `assertRan` 閉包的 `$process` 為 `Illuminate\Process\PendingProcess` 的實例，而 `$result` 則是 `Illuminate\Contracts\Process\ProcessResult` 的實例。

<a name="assert-process-didnt-run"></a>
#### assertDidntRun

斷言指定的行程未被呼叫：

```php
use Illuminate\Support\Facades\Process;

Process::assertDidntRun('ls -la');
```

如同 `assertRan` 方法，`assertDidntRun` 方法也接受一個閉包，該閉包將接收行程實例與行程結果，讓您可以檢查行程設定的選項。如果此閉包回傳 `true`，則斷言將「失敗」：

```php
Process::assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```

<a name="assert-process-ran-times"></a>
#### assertRanTimes

斷言指定的行程已被呼叫了指定的次數：

```php
use Illuminate\Support\Facades\Process;

Process::assertRanTimes('ls -la', times: 3);
```

`assertRanTimes` 方法也接受一個閉包，該閉包將接收 `PendingProcess` 與 `ProcessResult` 的實例，讓您可以檢查行程設定的選項。如果此閉包回傳 `true` 且行程被呼叫了指定的次數，則斷言將「通過」：

```php
Process::assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="assert-processes-ran-in-order"></a>
#### assertRanInOrder

斷言行程是按照指定的順序被呼叫：

```php
Process::assertRanInOrder([
    'git fetch',
    'composer install',
]);
```

`assertRanInOrder` 方法與其他行程斷言一樣，接受命令字串、命令引數陣列或閉包。

<a name="preventing-stray-processes"></a>
### 防止遺漏未偽造的行程

如果您想確保在單一測試或整個測試套件中所有被呼叫的行程都已經過偽造，您可以呼叫 `preventStrayProcesses` 方法。呼叫此方法後，任何沒有對應偽造結果的行程都將拋出例外，而不是啟動實際的行程：

```php
use Illuminate\Support\Facades\Process;

Process::preventStrayProcesses();

Process::fake([
    'ls *' => 'Test output...',
]);

// Fake response is returned...
Process::run('ls -la');

// An exception is thrown...
Process::run('bash import.sh');
```