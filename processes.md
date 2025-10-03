# 進程

- [簡介](#introduction)
- [調用進程](#invoking-processes)
    - [進程選項](#process-options)
    - [進程輸出](#process-output)
    - [管道](#process-pipelines)
- [非同步進程](#asynchronous-processes)
    - [進程 ID 與信號](#process-ids-and-signals)
    - [非同步進程輸出](#asynchronous-process-output)
    - [非同步進程超時](#asynchronous-process-timeouts)
- [並發進程](#concurrent-processes)
    - [命名池進程](#naming-pool-processes)
    - [池進程 ID 與信號](#pool-process-ids-and-signals)
- [測試](#testing)
    - [模擬進程](#faking-processes)
    - [模擬特定進程](#faking-specific-processes)
    - [模擬進程序列](#faking-process-sequences)
    - [模擬非同步進程生命週期](#faking-asynchronous-process-lifecycles)
    - [可用斷言](#available-assertions)
    - [防止迷失進程](#preventing-stray-processes)

<a name="introduction"></a>
## 簡介

Laravel 圍繞 [Symfony Process 元件](https://symfony.com/doc/current/components/process.html) 提供了一個表達力強大且極簡的 API，讓您可以方便地從 Laravel 應用程式中調用外部進程。Laravel 的進程功能專注於最常見的用例和絕佳的開發者體驗。


<a name="invoking-processes"></a>
## 調用進程

要調用進程，您可以使用 `Process` facade 提供的 `run` 和 `start` 方法。`run` 方法將調用進程並等待進程執行完成，而 `start` 方法則用於非同步進程執行。我們將在本文件中探討這兩種方法。首先，讓我們看看如何調用一個基本的同步進程並檢查其結果：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

return $result->output();
```

當然，`run` 方法返回的 `Illuminate\Contracts\Process\ProcessResult` 實例提供了多種有用的方法，可用於檢查進程結果：

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

如果您有一個進程結果，並且希望在結束代碼大於零（表示失敗）時拋出 `Illuminate\Process\Exceptions\ProcessFailedException` 的實例，您可以使用 `throw` 和 `throwIf` 方法。如果進程沒有失敗，將返回 `ProcessResult` 實例：

```php
$result = Process::run('ls -la')->throw();

$result = Process::run('ls -la')->throwIf($condition);
```


<a name="process-options"></a>
### 進程選項

當然，您可能需要在調用進程之前自訂其行為。幸運的是，Laravel 允許您調整各種進程功能，例如工作目錄、超時和環境變數。


<a name="working-directory-path"></a>
#### 工作目錄路徑

您可以使用 `path` 方法來指定進程的工作目錄。如果未調用此方法，進程將繼承當前執行中的 PHP 腳本的工作目錄：

```php
$result = Process::path(__DIR__)->run('ls -la');
```


<a name="input"></a>
#### 輸入

您可以使用 `input` 方法透過進程的「標準輸入」提供輸入：

```php
$result = Process::input('Hello World')->run('cat');
```


<a name="timeouts"></a>
#### 超時

預設情況下，進程在執行超過 60 秒後會拋出 `Illuminate\Process\Exceptions\ProcessTimedOutException` 的實例。但是，您可以透過 `timeout` 方法自訂此行為：

```php
$result = Process::timeout(120)->run('bash import.sh');
```

或者，如果您想完全禁用進程超時，您可以調用 `forever` 方法：

```php
$result = Process::forever()->run('bash import.sh');
```

`idleTimeout` 方法可用於指定進程在不返回任何輸出情況下可以運行的最大秒數：

```php
$result = Process::timeout(60)->idleTimeout(30)->run('bash import.sh');
```


<a name="environment-variables"></a>
#### 環境變數

環境變數可以透過 `env` 方法提供給進程。被調用的進程還將繼承系統定義的所有環境變數：

```php
$result = Process::forever()
    ->env(['IMPORT_PATH' => __DIR__])
    ->run('bash import.sh');
```

如果您希望從被調用的進程中移除繼承的環境變數，您可以將該環境變數的值設為 `false`：

```php
$result = Process::forever()
    ->env(['LOAD_PATH' => false])
    ->run('bash import.sh');
```


<a name="tty-mode"></a>
#### TTY 模式

`tty` 方法可用於為進程啟用 TTY 模式。TTY 模式將進程的輸入和輸出連接到您程式的輸入和輸出，允許您的進程作為進程打開 Vim 或 Nano 等編輯器：

```php
Process::forever()->tty()->run('vim');
```

> [!WARNING]
> TTY 模式在 Windows 上不受支援。


<a name="process-output"></a>
### 進程輸出

如前所述，進程輸出可以使用進程結果上的 `output` (stdout) 和 `errorOutput` (stderr) 方法來訪問：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

然而，也可以透過將閉包作為第二個參數傳遞給 `run` 方法來即時收集輸出。該閉包將接收兩個參數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$result = Process::run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Laravel 還提供了 `seeInOutput` 和 `seeInErrorOutput` 方法，它們提供了一種便捷的方式來判斷給定字串是否包含在進程的輸出中：

```php
if (Process::run('ls -la')->seeInOutput('laravel')) {
    // ...
}
```


<a name="disabling-process-output"></a>
#### 禁用進程輸出

如果您的進程正在寫入大量您不感興趣的輸出，您可以透過完全禁用輸出檢索來節省記憶體。為此，在建構進程時調用 `quietly` 方法：

```php
use Illuminate\Support\Facades\Process;

$result = Process::quietly()->run('bash import.sh');
```


<a name="process-pipelines"></a>
### 管道

有時您可能希望將一個進程的輸出作為另一個進程的輸入。這通常被稱為將一個進程的輸出「管道」到另一個進程。`Process` facade 提供的 `pipe` 方法使這變得容易實現。`pipe` 方法將同步執行管道中的進程，並返回管道中最後一個進程的結果：

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

如果您不需要自訂構成管道的個別進程，您可以簡單地將一個命令字串陣列傳遞給 `pipe` 方法：

```php
$result = Process::pipe([
    'cat example.txt',
    'grep -i "laravel"',
]);
```

可以透過將閉包作為第二個參數傳遞給 `pipe` 方法來即時收集進程輸出。該閉包將接收兩個參數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
}, function (string $type, string $output) {
    echo $output;
});
```

Laravel 還允許您透過 `as` 方法為管道中的每個進程分配字串鍵。此鍵也將傳遞給提供給 `pipe` 方法的輸出閉包，允許您確定輸出屬於哪個進程：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->as('first')->command('cat example.txt');
    $pipe->as('second')->command('grep -i "laravel"');
}, function (string $type, string $output, string $key) {
    // ...
});
```

<a name="asynchronous-processes"></a>
## 非同步進程

儘管 `run` 方法以同步方式調用進程，但 `start` 方法可用於以非同步方式調用進程。這讓您的應用程式可以在進程於背景執行時，繼續執行其他任務。進程一旦被調用，您就可以使用 `running` 方法來判斷進程是否仍在執行：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    // ...
}

$result = $process->wait();
```

如您所見，您可以調用 `wait` 方法，以等待進程執行完畢並取得 `ProcessResult` 實例：

```php
$process = Process::timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```

<a name="process-ids-and-signals"></a>
### 進程 ID 與信號

`id` 方法可用於取得執行中進程由作業系統分配的進程 ID：

```php
$process = Process::start('bash import.sh');

return $process->id();
```

您可以使用 `signal` 方法向執行中的進程發送「信號」。預定義的信號常數列表可在 [PHP 文件](https://www.php.net/manual/en/pcntl.constants.php)中找到：

```php
$process->signal(SIGUSR2);
```

<a name="asynchronous-process-output"></a>
### 非同步進程輸出

當非同步進程執行時，您可以使用 `output` 和 `errorOutput` 方法取得其所有當前輸出；然而，您可以使用 `latestOutput` 和 `latestErrorOutput` 方法來取得自上次取得輸出後，進程產生的輸出：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

與 `run` 方法一樣，也可以通過將閉包作為 `start` 方法的第二個參數傳遞，從非同步進程即時收集輸出。該閉包將接收兩個參數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$process = Process::start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

除了等待進程完成，您可以使用 `waitUntil` 方法根據進程輸出停止等待。當給予 `waitUntil` 方法的閉包返回 `true` 時，Laravel 將停止等待進程完成：

```php
$process = Process::start('bash import.sh');

$process->waitUntil(function (string $type, string $output) {
    return $output === 'Ready...';
});
```

<a name="asynchronous-process-timeouts"></a>
### 非同步進程超時

當非同步進程執行時，您可以使用 `ensureNotTimedOut` 方法驗證該進程是否未超時。如果進程超時，此方法將拋出一個 [超時異常](#timeouts)：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    $process->ensureNotTimedOut();

    // ...

    sleep(1);
}
```

<a name="concurrent-processes"></a>
## 並發進程

Laravel 也讓管理並發的非同步進程池變得輕而易舉，讓您可以輕鬆地同時執行多個任務。首先，調用 `pool` 方法，它接受一個閉包，該閉包會收到一個 `Illuminate\Process\Pool` 實例。

在此閉包中，您可以定義屬於該進程池的進程。一旦進程池透過 `start` 方法啟動，您就可以透過 `running` 方法訪問執行中進程的 [集合](/docs/{{version}}/collections)：

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

如您所見，您可以等待所有池進程執行完畢，並透過 `wait` 方法取得它們的結果。`wait` 方法會返回一個可透過陣列訪問的物件，讓您可以透過其鍵來訪問池中每個進程的 `ProcessResult` 實例：

```php
$results = $pool->wait();

echo $results[0]->output();
```

或者，為了方便起見，`concurrently` 方法可用於啟動一個非同步進程池並立即等待其結果。當與 PHP 的陣列解構功能結合使用時，這可以提供特別富有表達性的語法：

```php
[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```

<a name="naming-pool-processes"></a>
### 命名池進程

透過數字鍵訪問進程池結果並非富有表達性；因此，Laravel 允許您透過 `as` 方法為池中的每個進程分配字串鍵。此鍵也會傳遞給提供給 `start` 方法的閉包，讓您能夠判斷輸出屬於哪個進程：

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
### 池進程 ID 與信號

由於進程池的 `running` 方法提供了池中所有調用進程的集合，因此您可以輕鬆地訪問底層池進程 ID：

```php
$processIds = $pool->running()->each->id();
```

為了方便起見，您可以在進程池上調用 `signal` 方法，向池中的每個進程發送信號：

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## 測試

許多 Laravel 服務提供了功能，能幫助您輕鬆且明確地撰寫測試，而 Laravel 的進程服務也不例外。`Process` facade 的 `fake` 方法允許您指示 Laravel 在調用進程時返回模擬 (stubbed) / 虛擬 (dummy) 結果。

<a name="faking-processes"></a>
### 模擬進程

為了探討 Laravel 模擬進程的能力，讓我們想像一個調用進程的路由：

```php
use Illuminate\Support\Facades\Process;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    Process::run('bash import.sh');

    return 'Import complete!';
});
```

測試此路由時，我們可以透過在 `Process` facade 上調用不帶任何參數的 `fake` 方法，指示 Laravel 為每個調用的進程返回一個模擬的成功進程結果。此外，我們甚至可以 [斷言](#available-assertions) 某個進程確實被「執行 (run)」了：

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

如前所述，在 `Process` facade 上調用 `fake` 方法會指示 Laravel 總是返回一個沒有輸出的成功進程結果。但是，您可以使用 `Process` facade 的 `result` 方法輕鬆指定模擬進程的輸出和結束代碼：

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
### 模擬特定進程

正如您可能在前面的範例中注意到的，`Process` facade 允許您透過將一個陣列傳遞給 `fake` 方法，為每個進程指定不同的模擬結果。

該陣列的鍵應代表您希望模擬的命令模式及其相關結果。`*` 字元可用作萬用字元 (wildcard character)。任何未被模擬的進程命令都將實際調用。您可以使用 `Process` facade 的 `result` 方法為這些命令建構模擬 / 虛擬結果：

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

如果您不需要自訂模擬進程的結束代碼或錯誤輸出，您可能會覺得將模擬進程結果指定為簡單的字串會更方便：

```php
Process::fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```

<a name="faking-process-sequences"></a>
### 模擬進程序列

如果您正在測試的程式碼使用相同的命令調用多個進程，您可能希望為每次進程調用分配不同的模擬進程結果。您可以透過 `Process` facade 的 `sequence` 方法來實現這一點：

```php
Process::fake([
    'ls *' => Process::sequence()
        ->push(Process::result('First invocation'))
        ->push(Process::result('Second invocation')),
]);
```

<a name="faking-asynchronous-process-lifecycles"></a>
### 模擬非同步進程生命週期

到目前為止，我們主要討論了使用 `run` 方法同步調用的模擬進程。但是，如果您嘗試測試與透過 `start` 調用的非同步進程交互的程式碼，您可能需要一種更複雜的方法來描述您的模擬進程。

例如，讓我們想像以下與非同步進程交互的路由：

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

為了正確地模擬這個進程，我們需要能夠描述 `running` 方法應該返回 `true` 的次數。此外，我們可能還希望指定應該按順序返回的多行輸出。為了實現這一點，我們可以使用 `Process` facade 的 `describe` 方法：

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

讓我們深入探討上面的範例。使用 `output` 和 `errorOutput` 方法，我們可以指定將按順序返回的多行輸出。`exitCode` 方法可用於指定模擬進程的最終結束代碼。最後，`iterations` 方法可用於指定 `running` 方法應該返回 `true` 的次數。

<a name="available-assertions"></a>
### 可用斷言

正如 [前面討論](#faking-processes) 的，Laravel 為您的功能測試提供了多個進程斷言。我們將在下面討論每個斷言。

<a name="assert-process-ran"></a>
#### assertRan

斷言給定的進程已被調用：

```php
use Illuminate\Support\Facades\Process;

Process::assertRan('ls -la');
```

`assertRan` 方法也接受一個閉包，該閉包將接收一個進程實例和一個進程結果，允許您檢查進程的配置選項。如果此閉包返回 `true`，則斷言將「通過 (pass)」：

```php
Process::assertRan(fn ($process, $result) =>
    $process->command === 'ls -la' &&
    $process->path === __DIR__ &&
    $process->timeout === 60
);
```

傳遞給 `assertRan` 閉包的 `$process` 是一個 `Illuminate\Process\PendingProcess` 實例，而 `$result` 是一個 `Illuminate\Contracts\Process\ProcessResult` 實例。

<a name="assert-process-didnt-run"></a>
#### assertDidntRun

斷言給定的進程未被調用：

```php
use Illuminate\Support\Facades\Process;

Process::assertDidntRun('ls -la');
```

與 `assertRan` 方法類似，`assertDidntRun` 方法也接受一個閉包，該閉包將接收一個進程實例和一個進程結果，允許您檢查進程的配置選項。如果此閉包返回 `true`，則斷言將「失敗 (fail)」：

```php
Process::assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```

<a name="assert-process-ran-times"></a>
#### assertRanTimes

斷言給定的進程被調用指定次數：

```php
use Illuminate\Support\Facades\Process;

Process::assertRanTimes('ls -la', times: 3);
```

`assertRanTimes` 方法也接受一個閉包，該閉包將接收一個 `PendingProcess` 實例和一個 `ProcessResult` 實例，允許您檢查進程的配置選項。如果此閉包返回 `true` 並且進程被調用指定次數，則斷言將「通過 (pass)」：

```php
Process::assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="preventing-stray-processes"></a>
### 防止迷失進程

如果您想確保所有調用的進程在您的個別測試或完整測試套件中都被模擬，您可以調用 `preventStrayProcesses` 方法。調用此方法後，任何沒有相應模擬結果的進程都將拋出異常，而不是啟動實際的進程：

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

<a name="preventing-stray-processes"></a>
### 防止迷失進程

如果您希望確保在您的個別測試或整個測試套件中，所有調用的進程都已被模擬，您可以調用 `preventStrayProcesses` 方法。調用此方法後，任何沒有對應模擬結果的進程將會拋出例外，而不是啟動實際進程：

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