# 行程(Processes)

- [簡介](#introduction)
- [執行行程](#invoking-processes)
    - [行程選項](#process-options)
    - [行程輸出](#process-output)
    - [管線](#process-pipelines)
- [非同步行程](#asynchronous-processes)
    - [行程 ID 與信號](#process-ids-and-signals)
    - [非同步行程輸出](#asynchronous-process-output)
    - [非同步行程逾時](#asynchronous-process-timeouts)
- [並行行程](#concurrent-processes)
    - [為行程池命名](#naming-pool-processes)
    - [行程池 ID 與信號](#pool-process-ids-and-signals)
- [測試](#testing)
    - [模擬行程](#faking-processes)
    - [模擬特定行程](#faking-specific-processes)
    - [模擬行程序列](#faking-process-sequences)
    - [模擬非同步行程生命週期](#faking-asynchronous-process-lifecycles)
    - [可用的斷言](#available-assertions)
    - [防止遺留行程](#preventing-stray-processes)

<a name="introduction"></a>
## 簡介

Laravel 針對 [Symfony Process 元件](https://symfony.com/doc/current/components/process.html) 提供了一個表達力強且簡潔的 API，讓您可以方便地從 Laravel 應用程式中呼叫外部行程。Laravel 的行程功能專注於最常見的使用情境以及絕佳的開發者體驗。


<a name="invoking-processes"></a>
## 執行行程

要執行行程，您可以使用 `Process` Facade 提供的 `run` 與 `start` 方法。`run` 方法將執行行程並等待該行程執行完畢，而 `start` 方法則用於非同步行程執行。我們將在本文件中探討這兩種方法。首先，讓我們來看看如何執行一個基本的同步行程並檢查其結果：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

return $result->output();
```

當然，`run` 方法回傳的 `Illuminate\Contracts\Process\ProcessResult` 實例提供了一系列實用的方法，可用於檢查行程結果：

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

如果您有一個行程結果，且希望在結束碼 (exit code) 大於零（表示執行失敗）時拋出 `Illuminate\Process\Exceptions\ProcessFailedException` 實例，您可以使用 `throw` 與 `throwIf` 方法。如果行程沒有失敗，將回傳 `ProcessResult` 實例：

```php
$result = Process::run('ls -la')->throw();

$result = Process::run('ls -la')->throwIf($condition);
```


<a name="process-options"></a>
### 行程選項

當然，您可能需要在執行行程前自定義其行為。幸好，Laravel 允許您調整多項行程功能，例如工作目錄、逾時以及環境變數。


<a name="working-directory-path"></a>
#### 工作目錄路徑

您可以使用 `path` 方法來指定行程的工作目錄。如果沒有呼叫此方法，行程將繼承目前執行中的 PHP 腳本的工作目錄：

```php
$result = Process::path(__DIR__)->run('ls -la');
```


<a name="input"></a>
#### 輸入

您可以使用 `input` 方法透過行程的「標準輸入 (standard input)」提供輸入：

```php
$result = Process::input('Hello World')->run('cat');
```


<a name="timeouts"></a>
#### 逾時

預設情況下，行程在執行超過 60 秒後會拋出 `Illuminate\Process\Exceptions\ProcessTimedOutException` 實例。不過，您可以透過 `timeout` 方法自定義此行為：

```php
$result = Process::timeout(120)->run('bash import.sh');
```

`timeout` 與 `idleTimeout` 方法也接受 `CarbonInterval` 實例：

```php
use function Illuminate\Support\minutes;

$result = Process::timeout(minutes(2))->run('bash import.sh');
```

或者，如果您想完全禁用行程逾時，可以呼叫 `forever` 方法：

```php
$result = Process::forever()->run('bash import.sh');
```

`idleTimeout` 方法可用於指定行程在不回傳任何輸出情況下可執行的最大秒數：

```php
$result = Process::timeout(60)->idleTimeout(30)->run('bash import.sh');
```


<a name="environment-variables"></a>
#### 環境變數

可以透過 `env` 方法為行程提供環境變數。被執行的行程也會繼承您系統中定義的所有環境變數：

```php
$result = Process::forever()
    ->env(['IMPORT_PATH' => __DIR__])
    ->run('bash import.sh');
```

如果您希望從被執行的行程中移除繼承的環境變數，可以將該環境變數的值設定為 `false`：

```php
$result = Process::forever()
    ->env(['LOAD_PATH' => false])
    ->run('bash import.sh');
```


<a name="tty-mode"></a>
#### TTY 模式

`tty` 方法可用於為您的行程啟用 TTY 模式。TTY 模式將行程的輸入與輸出連接到您程式的輸入與輸出，讓您的行程能夠以行程形式開啟像 Vim 或 Nano 這樣的編輯器：

```php
Process::forever()->tty()->run('vim');
```

> [!WARNING]
> Windows 不支援 TTY 模式。


<a name="process-output"></a>
### 行程輸出

如前所述，可以使用行程結果上的 `output` (stdout) 與 `errorOutput` (stderr) 方法來存取行程輸出：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

不過，也可以透過將閉包 (closure) 作為 `run` 方法的第二個引數來即時收集輸出。該閉包將接收兩個引數：輸出的「類型」(`stdout` 或 `stderr`) 以及輸出字串本身：

```php
$result = Process::run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Laravel 還提供了 `seeInOutput` 與 `seeInErrorOutput` 方法，提供一種方便的方式來判斷行程輸出中是否包含給定的字串：

```php
if (Process::run('ls -la')->seeInOutput('laravel')) {
    // ...
}
```


<a name="disabling-process-output"></a>
#### 禁用行程輸出

如果您的行程正在寫入大量您不感興趣的輸出，您可以透過完全禁用輸出擷取來節省記憶體。若要實現此目的，請在建立行程時呼叫 `quietly` 方法：

```php
use Illuminate\Support\Facades\Process;

$result = Process::quietly()->run('bash import.sh');
```


<a name="process-pipelines"></a>
### 管線

有時您可能希望將一個行程的輸出作為另一個行程的輸入。這通常被稱為將行程的輸出「管線化 (piping)」到另一個行程。`Process` Facade 提供的 `pipe` 方法讓這變得很容易實現。`pipe` 方法將同步執行管線中的行程，並回傳管線中最後一個行程的結果：

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

如果您不需要自定義組成管線的單個行程，只需將指令字串陣列傳遞給 `pipe` 方法即可：

```php
$result = Process::pipe([
    'cat example.txt',
    'grep -i "laravel"',
]);
```

可以透過將閉包作為 `pipe` 方法的第二個引數來即時收集行程輸出。該閉包將接收兩個引數：輸出的「類型」(`stdout` 或 `stderr`) 以及輸出字串本身：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
}, function (string $type, string $output) {
    echo $output;
});
```

Laravel 還允許您透過 `as` 方法為管線中的每個行程分配字串鍵。此鍵也將被傳遞給提供給 `pipe` 方法的輸出閉包，讓您能夠判斷輸出屬於哪個行程：

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

`run` 方法是以同步方式執行行程，而 `start` 方法可用於非同步地執行行程。這讓您的應用程式在行程於背景執行時，能繼續執行其他任務。行程啟動後，您可以使用 `running` 方法來判斷該行程是否仍在執行：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    // ...
}

$result = $process->wait();
```

如您所見，您可以呼叫 `wait` 方法直到行程執行完畢，並取得 `ProcessResult` 實例：

```php
$process = Process::timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```


<a name="process-ids-and-signals"></a>
### 行程 ID 與信號

`id` 方法可用於取得作業系統分配給執行中行程的行程 ID：

```php
$process = Process::start('bash import.sh');

return $process->id();
```

您可以使用 `signal` 方法向執行中的行程發送「信號」。預定義信號常數的列表可以在 [PHP 說明文件](https://www.php.net/manual/en/pcntl.constants.php) 中找到：

```php
$process->signal(SIGUSR2);
```


<a name="asynchronous-process-output"></a>
### 非同步行程輸出

當非同步行程執行時，您可以使用 `output` 與 `errorOutput` 方法存取其目前的所有輸出；然而，您可以使用 `latestOutput` 與 `latestErrorOutput` 來存取自上次取得輸出以來，行程所產生的新輸出：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

與 `run` 方法類似，您也可以透過將閉包 (closure) 作為 `start` 方法的第二個引數，來即時收集非同步行程的輸出。該閉包將接收兩個引數：輸出的「類型」(`stdout` 或 `stderr`) 以及輸出字串本身：

```php
$process = Process::start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

您可以使用 `waitUntil` 方法根據行程的輸出來停止等待，而不需要等到行程完全結束。當傳遞給 `waitUntil` 方法的閉包回傳 `true` 時，Laravel 將停止等待該行程完畢：

```php
$process = Process::start('bash import.sh');

$process->waitUntil(function (string $type, string $output) {
    return $output === 'Ready...';
});
```


<a name="asynchronous-process-timeouts"></a>
### 非同步行程逾時

當非同步行程執行時，您可以使用 `ensureNotTimedOut` 方法來驗證行程是否尚未逾時。如果行程已逾時，此方法將拋出 [逾時例外](#timeouts)：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    $process->ensureNotTimedOut();

    // ...

    sleep(1);
}
```


<a name="concurrent-processes"></a>
## 並行行程

Laravel 也讓管理並行且非同步的行程池 (process pool) 變得輕而易舉，讓您能輕鬆地同時執行多個任務。首先，呼叫 `pool` 方法，該方法接收一個閉包，而閉包會收到一個 `Illuminate\Process\Pool` 實例。

在此閉包中，您可以定義屬於該池的行程。一旦行程池透過 `start` 方法啟動，您就可以透過 `running` 方法存取執行中行程的 [集合](/docs/{{version}}/collections)：

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

如您所見，您可以透過 `wait` 方法等待行程池中所有行程執行完畢並取得其結果。`wait` 方法回傳一個可像陣列一樣存取的物件，讓您能透過鍵值 (key) 存取池中每個行程的 `ProcessResult` 實例：

```php
$results = $pool->wait();

echo $results[0]->output();
```

或者，為了方便起見，可以使用 `concurrently` 方法來啟動非同步行程池並立即等待其結果。當與 PHP 的陣列解構 (array destructuring) 功能結合時，這能提供非常簡潔的語法：

```php
[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```


<a name="naming-pool-processes"></a>
### 為行程池命名

透過數字鍵值存取行程池結果並不直觀；因此，Laravel 允許您透過 `as` 方法為池中的每個行程指定字串鍵值。此鍵值也會被傳遞給 `start` 方法提供的閉包，讓您能判斷該輸出屬於哪個行程：

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
### 行程池 ID 與信號

由於行程池的 `running` 方法提供了池中所有已啟動行程的集合，您可以輕鬆存取底層的行程 ID：

```php
$processIds = $pool->running()->each->id();
```

此外，為了方便起見，您可以在行程池上呼叫 `signal` 方法，向池中的每個行程發送信號：

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了功能來幫助您輕鬆且具表達力地撰寫測試，Laravel 的行程服務也不例外。`Process` Facade 的 `fake` 方法允許您指示 Laravel 在執行行程時返回預設 (stubbed) 或虛擬 (dummy) 的結果。


<a name="faking-processes"></a>
### 模擬行程

為了探索 Laravel 模擬行程的能力，讓我們想像一個執行行程的路由：

```php
use Illuminate\Support\Facades\Process;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    Process::run('bash import.sh');

    return 'Import complete!';
});
```

在測試此路由時，我們可以透過呼叫 `Process` Facade 且不傳遞任何參數的 `fake` 方法，來指示 Laravel 為每個被執行的行程返回一個模擬的成功結果。此外，我們甚至可以[斷言](#available-assertions)某個給定的行程已被「執行」：

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

如前所述，在 `Process` Facade 上呼叫 `fake` 方法會指示 Laravel 始終返回一個沒有輸出的成功行程結果。然而，您可以使用 `Process` Facade 的 `result` 方法輕鬆為模擬的行程指定輸出內容與結束碼：

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
### 模擬特定行程

您可能在之前的範例中注意到，`Process` Facade 允許您透過向 `fake` 方法傳遞陣列，來為每個行程指定不同的模擬結果。

陣列的鍵 (key) 應代表您希望模擬的指令模式及其相關結果。`*` 字元可用作萬用字元。任何未被模擬的行程指令將會被實際執行。您可以使用 `Process` Facade 的 `result` 方法來為這些指令建構預設/模擬結果：

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

如果您不需要自定義模擬行程的結束碼或錯誤輸出，您可能會發現將模擬行程結果指定為簡單的字串更為方便：

```php
Process::fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```


<a name="faking-process-sequences"></a>
### 模擬行程序列

如果您測試的程式碼使用相同的指令執行了多個行程，您可能希望為每次行程呼叫分配不同的模擬結果。您可以透過 `Process` Facade 的 `sequence` 方法來實現：

```php
Process::fake([
    'ls *' => Process::sequence()
        ->push(Process::result('First invocation'))
        ->push(Process::result('Second invocation')),
]);
```


<a name="faking-asynchronous-process-lifecycles"></a>
### 模擬非同步行程生命週期

到目前為止，我們主要討論了使用 `run` 方法同步執行行程的模擬。然而，如果您嘗試測試與透過 `start` 執行之非同步行程互動的程式碼，您可能需要一種更複雜的方法來描述您的模擬行程。

例如，讓我們想像以下一個與非同步行程互動的路由：

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

為了正確模擬此行程，我們需要能夠描述 `running` 方法應該返回 `true` 多少次。此外，我們可能想要指定應按順序返回的多行輸出。為了實現這一點，我們可以使用 `Process` Facade 的 `describe` 方法：

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

讓我們深入分析上面的範例。使用 `output` 和 `errorOutput` 方法，我們可以指定將按順序返回的多行輸出。`exitCode` 方法可用於指定模擬行程的最終結束碼。最後，`iterations` 方法可用於指定 `running` 方法應該返回 `true` 多少次。


<a name="available-assertions"></a>
### 可用的斷言

如[前文所述](#faking-processes)，Laravel 為您的功能測試提供了數個行程斷言。我們將在下方討論這些斷言。


<a name="assert-process-ran"></a>
#### assertRan

斷言給定的行程已被執行：

```php
use Illuminate\Support\Facades\Process;

Process::assertRan('ls -la');
```

`assertRan` 方法也接受一個閉包，該閉包將接收一個行程實例和一個行程結果，允許您檢查行程的配置選項。如果此閉包返回 `true`，則斷言「通過」：

```php
Process::assertRan(fn ($process, $result) =>
    $process->command === 'ls -la' &&
    $process->path === __DIR__ &&
    $process->timeout === 60
);
```

傳遞給 `assertRan` 閉包的 `$process` 是 `Illuminate\Process\PendingProcess` 的實例，而 `$result` 是 `Illuminate\Contracts\Process\ProcessResult` 的實例。


<a name="assert-process-didnt-run"></a>
#### assertDidntRun

斷言給定的行程未被執行：

```php
use Illuminate\Support\Facades\Process;

Process::assertDidntRun('ls -la');
```

與 `assertRan` 方法類似，`assertDidntRun` 方法也接受一個閉包，該閉包將接收一個行程實例和一個行程結果，允許您檢查行程的配置選項。如果此閉包返回 `true`，則斷言「失敗」：

```php
Process::assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```


<a name="assert-process-ran-times"></a>
#### assertRanTimes

斷言給定的行程被執行了指定次數：

```php
use Illuminate\Support\Facades\Process;

Process::assertRanTimes('ls -la', times: 3);
```

`assertRanTimes` 方法也接受一個閉包，該閉包將接收 `PendingProcess` 和 `ProcessResult` 實例，允許您檢查行程的配置選項。如果此閉包返回 `true` 且行程被執行了指定的次數，則斷言「通過」：

```php
Process::assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="preventing-stray-processes"></a>
### 防止遺留行程

如果您想要確保在個別測試或整個測試套件中，所有執行的行程都已被模擬，您可以呼叫 `preventStrayProcesses` 方法。在呼叫此方法後，任何沒有對應模擬結果的行程都將拋出例外，而非啟動一個實際的行程：

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