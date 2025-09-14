# 處理程序 (Processes)

- [簡介](#introduction)
- [呼叫處理程序](#invoking-processes)
    - [處理程序選項](#process-options)
    - [處理程序輸出](#process-output)
    - [管線](#process-pipelines)
- [非同步處理程序](#asynchronous-processes)
    - [處理程序 ID 與訊號](#process-ids-and-signals)
    - [非同步處理程序輸出](#asynchronous-process-output)
    - [非同步處理程序逾時](#asynchronous-process-timeouts)
- [並行處理程序](#concurrent-processes)
    - [命名 Pool 處理程序](#naming-pool-processes)
    - [Pool 處理程序 ID 與訊號](#pool-process-ids-and-signals)
- [測試](#testing)
    - [偽造處理程序](#faking-processes)
    - [偽造特定處理程序](#faking-specific-processes)
    - [偽造處理程序序列](#faking-process-sequences)
    - [偽造非同步處理程序生命週期](#faking-asynchronous-process-lifecycles)
    - [可用的斷言](#available-assertions)
    - [防止意外處理程序](#preventing-stray-processes)

<a name="introduction"></a>
## 簡介

Laravel 圍繞著 [Symfony Process component](https://symfony.com/doc/current/components/process.html) 提供了一個表達性強且簡潔的 API，讓您能夠方便地從 Laravel 應用程式中呼叫外部處理程序。Laravel 的處理程序功能專注於最常見的使用情境與絕佳的開發者體驗。

<a name="invoking-processes"></a>
## 呼叫處理程序

要呼叫處理程序，您可以使用 `Process` Facade 提供的 `run` 和 `start` 方法。`run` 方法將呼叫處理程序並等待其執行完成，而 `start` 方法則用於非同步處理程序執行。我們將在本文件中探討這兩種方法。首先，讓我們看看如何呼叫一個基本的同步處理程序並檢查其結果：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

return $result->output();
```

當然，`run` 方法返回的 `Illuminate\Contracts\Process\ProcessResult` 實例提供了各種有用的方法，可用於檢查處理程序結果：

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

如果您有一個處理程序結果，並且希望在結束代碼大於零（表示失敗）時拋出 `Illuminate\Process\Exceptions\ProcessFailedException` 實例，您可以使用 `throw` 和 `throwIf` 方法。如果處理程序沒有失敗，則會返回 `ProcessResult` 實例：

```php
$result = Process::run('ls -la')->throw();

$result = Process::run('ls -la')->throwIf($condition);
```

<a name="process-options"></a>
### 處理程序選項

當然，您可能需要在呼叫處理程序之前自訂其行為。幸運的是，Laravel 允許您調整各種處理程序功能，例如工作目錄、逾時和環境變數。

<a name="working-directory-path"></a>
#### 工作目錄路徑

您可以使用 `path` 方法指定處理程序的工作目錄。如果未呼叫此方法，處理程序將繼承目前執行中 PHP 腳本的工作目錄：

```php
$result = Process::path(__DIR__)->run('ls -la');
```

<a name="input"></a>
#### 輸入

您可以使用 `input` 方法透過處理程序的「標準輸入」提供輸入：

```php
$result = Process::input('Hello World')->run('cat');
```

<a name="timeouts"></a>
#### 逾時

預設情況下，處理程序在執行超過 60 秒後將拋出 `Illuminate\Process\Exceptions\ProcessTimedOutException` 實例。但是，您可以透過 `timeout` 方法自訂此行為：

```php
$result = Process::timeout(120)->run('bash import.sh');
```

或者，如果您想完全禁用處理程序逾時，可以呼叫 `forever` 方法：

```php
$result = Process::forever()->run('bash import.sh');
```

`idleTimeout` 方法可用於指定處理程序在不返回任何輸出的情況下可以執行的最大秒數：

```php
$result = Process::timeout(60)->idleTimeout(30)->run('bash import.sh');
```

<a name="environment-variables"></a>
#### 環境變數

環境變數可以透過 `env` 方法提供給處理程序。被呼叫的處理程序也將繼承您系統定義的所有環境變數：

```php
$result = Process::forever()
    ->env(['IMPORT_PATH' => __DIR__])
    ->run('bash import.sh');
```

如果您希望從被呼叫的處理程序中移除繼承的環境變數，您可以將該環境變數的值設為 `false`：

```php
$result = Process::forever()
    ->env(['LOAD_PATH' => false])
    ->run('bash import.sh');
```

<a name="tty-mode"></a>
#### TTY 模式

`tty` 方法可用於為您的處理程序啟用 TTY 模式。TTY 模式將處理程序的輸入和輸出連接到您程式的輸入和輸出，允許您的處理程序作為一個處理程序打開像 Vim 或 Nano 這樣的編輯器：

```php
Process::forever()->tty()->run('vim');
```

> [!WARNING]
> TTY 模式在 Windows 上不受支援。

<a name="process-output"></a>
### 處理程序輸出

如前所述，處理程序輸出可以使用處理程序結果上的 `output` (stdout) 和 `errorOutput` (stderr) 方法來存取：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

然而，也可以透過將一個閉包作為 `run` 方法的第二個參數傳遞來即時收集輸出。該閉包將接收兩個參數：輸出的「類型」（`stdout` 或 `stderr`）和輸出字串本身：

```php
$result = Process::run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Laravel 還提供了 `seeInOutput` 和 `seeInErrorOutput` 方法，它們提供了一種方便的方式來判斷給定字串是否包含在處理程序的輸出中：

```php
if (Process::run('ls -la')->seeInOutput('laravel')) {
    // ...
}
```

<a name="disabling-process-output"></a>
#### 禁用處理程序輸出

如果您的處理程序正在寫入大量您不感興趣的輸出，您可以透過完全禁用輸出檢索來節省記憶體。要實現這一點，請在建構處理程序時呼叫 `quietly` 方法：

```php
use Illuminate\Support\Facades\Process;

$result = Process::quietly()->run('bash import.sh');
```

<a name="process-pipelines"></a>
### 管線

有時您可能希望將一個處理程序的輸出作為另一個處理程序的輸入。這通常被稱為將一個處理程序的輸出「導向」另一個處理程序。`Process` Facade 提供的 `pipe` 方法使這變得容易實現。`pipe` 方法將同步執行導向的處理程序，並返回管線中最後一個處理程序的結果：

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

如果您不需要自訂構成管線的個別處理程序，您可以簡單地將命令字串陣列傳遞給 `pipe` 方法：

```php
$result = Process::pipe([
    'cat example.txt',
    'grep -i "laravel"',
]);
```

處理程序輸出可以透過將一個閉包作為 `pipe` 方法的第二個參數傳遞來即時收集。該閉包將接收兩個參數：輸出的「類型」（`stdout` 或 `stderr`）和輸出字串本身：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->command('cat example.txt');
    $pipe->command('grep -i "laravel"');
}, function (string $type, string $output) {
    echo $output;
});
```

Laravel 還允許您透過 `as` 方法為管線中的每個處理程序分配字串鍵。此鍵也將傳遞給提供給 `pipe` 方法的輸出閉包，讓您能夠判斷輸出屬於哪個處理程序：

```php
$result = Process::pipe(function (Pipe $pipe) {
    $pipe->as('first')->command('cat example.txt');
    $pipe->as('second')->command('grep -i "laravel"');
}, function (string $type, string $output, string $key) {
    // ...
});
```

<a name="asynchronous-processes"></a>
## 非同步處理程序

雖然 `run` 方法同步呼叫處理程序，但 `start` 方法可用於非同步呼叫處理程序。這允許您的應用程式在處理程序於背景執行時繼續執行其他任務。一旦處理程序被呼叫，您可以使用 `running` 方法來判斷處理程序是否仍在執行：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    // ...
}

$result = $process->wait();
```

您可能已經注意到，您可以呼叫 `wait` 方法來等待處理程序執行完成並檢索 `ProcessResult` 實例：

```php
$process = Process::timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```

<a name="process-ids-and-signals"></a>
### 處理程序 ID 與訊號

`id` 方法可用於檢索執行中處理程序由作業系統分配的處理程序 ID：

```php
$process = Process::start('bash import.sh');

return $process->id();
```

您可以使用 `signal` 方法向執行中的處理程序發送「訊號」。預定義訊號常數的列表可以在 [PHP 說明文件](https://www.php.net/manual/en/pcntl.constants.php)中找到：

```php
$process->signal(SIGUSR2);
```

<a name="asynchronous-process-output"></a>
### 非同步處理程序輸出

當非同步處理程序正在執行時，您可以使用 `output` 和 `errorOutput` 方法存取其所有目前輸出；但是，您可以使用 `latestOutput` 和 `latestErrorOutput` 來存取自上次檢索輸出以來發生的處理程序輸出：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

與 `run` 方法一樣，也可以透過將一個閉包作為 `start` 方法的第二個參數傳遞來即時從非同步處理程序收集輸出。該閉包將接收兩個參數：輸出的「類型」（`stdout` 或 `stderr`）和輸出字串本身：

```php
$process = Process::start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

您可以使用 `waitUntil` 方法根據處理程序的輸出停止等待，而不是等到處理程序完成。當給定給 `waitUntil` 方法的閉包返回 `true` 時，Laravel 將停止等待處理程序完成：

```php
$process = Process::start('bash import.sh');

$process->waitUntil(function (string $type, string $output) {
    return $output === 'Ready...';
});
```

<a name="asynchronous-process-timeouts"></a>
### 非同步處理程序逾時

當非同步處理程序正在執行時，您可以使用 `ensureNotTimedOut` 方法驗證處理程序是否沒有逾時。如果處理程序已逾時，此方法將拋出一個[逾時例外](#timeouts)：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    $process->ensureNotTimedOut();

    // ...

    sleep(1);
}
```

<a name="concurrent-processes"></a>
## 並行處理程序

Laravel 還讓管理並行、非同步處理程序 Pool 變得輕而易舉，讓您能夠輕鬆地同時執行許多任務。要開始使用，請呼叫 `pool` 方法，該方法接受一個閉包，該閉包接收 `Illuminate\Process\Pool` 的實例。

在此閉包中，您可以定義屬於 Pool 的處理程序。一旦透過 `start` 方法啟動處理程序 Pool，您就可以透過 `running` 方法存取執行中處理程序的 [collection](/docs/{{version}}/collections)：

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

如您所見，您可以等待所有 Pool 處理程序執行完成並透過 `wait` 方法解析其結果。`wait` 方法返回一個可陣列存取的物件，允許您透過其鍵存取 Pool 中每個處理程序的 `ProcessResult` 實例：

```php
$results = $pool->wait();

echo $results[0]->output();
```

或者，為了方便起見，`concurrently` 方法可用於啟動非同步處理程序 Pool 並立即等待其結果。這與 PHP 的陣列解構功能結合使用時，可以提供特別具表達性的語法：

```php
[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```

<a name="naming-pool-processes"></a>
### 命名 Pool 處理程序

透過數字鍵存取處理程序 Pool 結果的表達性不強；因此，Laravel 允許您透過 `as` 方法為 Pool 中的每個處理程序分配字串鍵。此鍵也將傳遞給提供給 `start` 方法的閉包，讓您能夠判斷輸出屬於哪個處理程序：

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
### Pool 處理程序 ID 與訊號

由於處理程序 Pool 的 `running` 方法提供了 Pool 中所有被呼叫處理程序的 collection，您可以輕鬆存取底層的 Pool 處理程序 ID：

```php
$processIds = $pool->running()->each->id();
```

而且，為了方便起見，您可以呼叫處理程序 Pool 上的 `signal` 方法，向 Pool 中的每個處理程序發送訊號：

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## 測試

許多 Laravel 服務都提供了功能，可幫助您輕鬆且具表達性地編寫測試，Laravel 的處理程序服務也不例外。`Process` Facade 的 `fake` 方法允許您指示 Laravel 在呼叫處理程序時返回 Stub / 虛擬結果。

<a name="faking-processes"></a>
### 偽造處理程序

為了探索 Laravel 偽造處理程序的能力，讓我們想像一個呼叫處理程序的路由：

```php
use Illuminate\Support\Facades\Process;
use Illuminate\Support\Facades\Route;

Route::get('/import', function () {
    Process::run('bash import.sh');

    return 'Import complete!';
});
```

在測試此路由時，我們可以透過呼叫 `Process` Facade 上不帶參數的 `fake` 方法，指示 Laravel 為每個被呼叫的處理程序返回一個偽造的、成功的處理程序結果。此外，我們甚至可以[斷言](#available-assertions)給定的處理程序是否被「執行」：

```php tab=Pest
<?php

use Illuminate\Contracts\Process\ProcessResult;
use Illuminate\Process\PendingProcess;
use Illuminate\Support\Facades\Process;

test('process is invoked', function () {
    Process::fake();

    $response = $this->get('/import');

    // 簡單的處理程序斷言...
    Process::assertRan('bash import.sh');

    // 或者，檢查處理程序配置...
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

        // 簡單的處理程序斷言...
        Process::assertRan('bash import.sh');

        // 或者，檢查處理程序配置...
        Process::assertRan(function (PendingProcess $process, ProcessResult $result) {
            return $process->command === 'bash import.sh' &&
                   $process->timeout === 60;
        });
    }
}
```

如前所述，在 `Process` Facade 上呼叫 `fake` 方法將指示 Laravel 始終返回一個沒有輸出的成功處理程序結果。但是，您可以使用 `Process` Facade 的 `result` 方法輕鬆指定偽造處理程序的輸出和結束代碼：

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
### 偽造特定處理程序

您可能已經在前面的範例中注意到，`Process` Facade 允許您透過將陣列傳遞給 `fake` 方法，為每個處理程序指定不同的偽造結果。

陣列的鍵應表示您希望偽造的命令模式及其相關結果。`*` 字元可用作萬用字元。任何未被偽造的處理程序命令都將實際被呼叫。您可以使用 `Process` Facade 的 `result` 方法為這些命令建構 Stub / 偽造結果：

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

如果您不需要自訂偽造處理程序的結束代碼或錯誤輸出，您可能會發現將偽造處理程序結果指定為簡單字串更方便：

```php
Process::fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```

<a name="faking-process-sequences"></a>
### 偽造處理程序序列

如果您正在測試的程式碼使用相同的命令呼叫多個處理程序，您可能希望為每個處理程序呼叫分配不同的偽造處理程序結果。您可以透過 `Process` Facade 的 `sequence` 方法實現此目的：

```php
Process::fake([
    'ls *' => Process::sequence()
        ->push(Process::result('First invocation'))
        ->push(Process::result('Second invocation')),
]);
```

<a name="faking-asynchronous-process-lifecycles"></a>
### 偽造非同步處理程序生命週期

到目前為止，我們主要討論了偽造使用 `run` 方法同步呼叫的處理程序。但是，如果您嘗試測試與透過 `start` 呼叫的非同步處理程序互動的程式碼，您可能需要更複雜的方法來描述您的偽造處理程序。

例如，讓我們想像以下與非同步處理程序互動的路由：

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

為了正確偽造此處理程序，我們需要能夠描述 `running` 方法應該返回 `true` 的次數。此外，我們可能希望指定應按順序返回的多行輸出。為此，我們可以使用 `Process` Facade 的 `describe` 方法：

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

讓我們深入研究上面的範例。使用 `output` 和 `errorOutput` 方法，我們可以指定將按順序返回的多行輸出。`exitCode` 方法可用於指定偽造處理程序的最終結束代碼。最後，`iterations` 方法可用於指定 `running` 方法應該返回 `true` 的次數。

<a name="available-assertions"></a>
### 可用的斷言

如[前所述](#faking-processes)，Laravel 為您的功能測試提供了幾個處理程序斷言。我們將在下面討論這些斷言。

<a name="assert-process-ran"></a>
#### assertRan

斷言給定的處理程序已被呼叫：

```php
use Illuminate\Support\Facades\Process;

Process::assertRan('ls -la');
```

`assertRan` 方法也接受一個閉包，該閉包將接收處理程序實例和處理程序結果，允許您檢查處理程序的配置選項。如果此閉包返回 `true`，則斷言將「通過」：

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

斷言給定的處理程序未被呼叫：

```php
use Illuminate\Support\Facades\Process;

Process::assertDidntRun('ls -la');
```

與 `assertRan` 方法一樣，`assertDidntRun` 方法也接受一個閉包，該閉包將接收處理程序實例和處理程序結果，允許您檢查處理程序的配置選項。如果此閉包返回 `true`，則斷言將「失敗」：

```php
Process::assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```

<a name="assert-process-ran-times"></a>
#### assertRanTimes

斷言給定的處理程序已被呼叫指定次數：

```php
use Illuminate\Support\Facades\Process;

Process::assertRanTimes('ls -la', times: 3);
```

`assertRanTimes` 方法也接受一個閉包，該閉包將接收 `PendingProcess` 和 `ProcessResult` 的實例，允許您檢查處理程序的配置選項。如果此閉包返回 `true` 並且處理程序已被呼叫指定次數，則斷言將「通過」：

```php
Process::assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="preventing-stray-processes"></a>
### 防止意外處理程序

如果您想確保所有被呼叫的處理程序在您的個別測試或完整測試套件中都已被偽造，您可以呼叫 `preventStrayProcesses` 方法。呼叫此方法後，任何沒有相應偽造結果的處理程序都將拋出例外，而不是啟動實際的處理程序：

```php
use Illuminate\Support\Facades\Process;

Process::preventStrayProcesses();

Process::fake([
    'ls *' => 'Test output...',
]);

// 返回偽造的回應...
Process::run('ls -la');

// 拋出例外...
Process::run('bash import.sh');
```
