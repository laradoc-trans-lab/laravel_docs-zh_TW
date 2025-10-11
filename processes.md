# 處理程序

- [簡介](#introduction)
- [呼叫處理程序](#invoking-processes)
    - [處理程序選項](#process-options)
    - [處理程序輸出](#process-output)
    - [管線](#process-pipelines)
- [非同步處理程序](#asynchronous-processes)
    - [處理程序 ID 與訊號](#process-ids-and-signals)
    - [非同步處理程序輸出](#asynchronous-process-output)
- [並行處理程序](#concurrent-processes)
    - [命名集區處理程序](#naming-pool-processes)
    - [集區處理程序 ID 與訊號](#pool-process-ids-and-signals)
- [測試](#testing)
    - [偽造處理程序](#faking-processes)
    - [偽造特定處理程序](#faking-specific-processes)
    - [偽造處理程序序列](#faking-process-sequences)
    - [偽造非同步處理程序生命週期](#faking-asynchronous-process-lifecycles)
    - [可用斷言](#available-assertions)
    - [防止逸散處理程序](#preventing-stray-processes)

<a name="introduction"></a>
## 簡介

Laravel 圍繞 [Symfony Process 元件](https://symfony.com/doc/7.0/components/process.html) 提供了一個富有表達力、簡潔的 API，讓您能夠方便地從 Laravel 應用程式中呼叫外部處理程序。Laravel 的處理程序功能專注於最常見的用例和絕佳的開發者體驗。

<a name="invoking-processes"></a>
## 呼叫處理程序

要呼叫處理程序，您可以使用 `Process` facade 提供的 `run` 和 `start` 方法。`run` 方法將呼叫處理程序並等待處理程序執行完成，而 `start` 方法用於非同步處理程序執行。我們將在本文件中探討這兩種方法。首先，讓我們看看如何呼叫一個基本的同步處理程序並檢查其結果：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

return $result->output();
```

當然，`run` 方法返回的 `Illuminate\Contracts\Process\ProcessResult` 實例提供了多種有用的方法，可用於檢查處理程序結果：

```php
$result = Process::run('ls -la');

$result->successful();
$result->failed();
$result->exitCode();
$result->output();
$result->errorOutput();
```

<a name="throwing-exceptions"></a>
#### 拋出例外

如果您有一個處理程序結果，並且想在結束碼大於零 (表示失敗) 時拋出 `Illuminate\Process\Exceptions\ProcessFailedException` 實例，您可以使用 `throw` 和 `throwIf` 方法。如果處理程序沒有失敗，將會返回處理程序結果實例：

```php
$result = Process::run('ls -la')->throw();

$result = Process::run('ls -la')->throwIf($condition);
```

<a name="process-options"></a>
### 處理程序選項

當然，您可能需要在呼叫處理程序之前自訂處理程序的行為。幸運的是，Laravel 允許您調整各種處理程序功能，例如工作目錄、逾時和環境變數。

<a name="working-directory-path"></a>
#### 工作目錄路徑

您可以使用 `path` 方法來指定處理程序的工作目錄。如果未呼叫此方法，處理程序將繼承目前執行 PHP 腳本的工作目錄：

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

預設情況下，處理程序在執行超過 60 秒後，將會拋出 `Illuminate\Process\Exceptions\ProcessTimedOutException` 實例。但是，您可以透過 `timeout` 方法自訂此行為：

```php
$result = Process::timeout(120)->run('bash import.sh');
```

或者，如果您想完全停用處理程序逾時，您可以呼叫 `forever` 方法：

```php
$result = Process::forever()->run('bash import.sh');
```

`idleTimeout` 方法可用於指定處理程序在沒有任何輸出情況下可執行的最大秒數：

```php
$result = Process::timeout(60)->idleTimeout(30)->run('bash import.sh');
```

<a name="environment-variables"></a>
#### 環境變數

環境變數可以透過 `env` 方法提供給處理程序。被呼叫的處理程序也將繼承系統定義的所有環境變數：

```php
$result = Process::forever()
    ->env(['IMPORT_PATH' => __DIR__])
    ->run('bash import.sh');
```

如果您希望從被呼叫的處理程序中移除一個繼承的環境變數，您可以為該環境變數提供 `false` 值：

```php
$result = Process::forever()
    ->env(['LOAD_PATH' => false])
    ->run('bash import.sh');
```

<a name="tty-mode"></a>
#### TTY 模式

`tty` 方法可用於啟用處理程序的 TTY 模式。TTY 模式將處理程序的輸入與輸出連接到您程式的輸入與輸出，讓您的處理程序能夠以處理程序的形式開啟像是 Vim 或 Nano 的編輯器：

```php
Process::forever()->tty()->run('vim');
```

<a name="process-output"></a>
### 處理程序輸出

如前所述，處理程序輸出可以使用處理程序結果上的 `output` (標準輸出) 和 `errorOutput` (標準錯誤輸出) 方法來存取：

```php
use Illuminate\Support\Facades\Process;

$result = Process::run('ls -la');

echo $result->output();
echo $result->errorOutput();
```

然而，輸出也可以透過將一個閉包作為第二個引數傳遞給 `run` 方法來即時收集。該閉包將會接收兩個引數：輸出類型 (`stdout` 或 `stderr`) 和輸出字串本身：

```php
$result = Process::run('ls -la', function (string $type, string $output) {
    echo $output;
});
```

Laravel 還提供了 `seeInOutput` 和 `seeInErrorOutput` 方法，它們提供了一種方便的方式，用於判斷給定的字串是否包含在處理程序的輸出中：

```php
if (Process::run('ls -la')->seeInOutput('laravel')) {
    // ...
}
```

<a name="disabling-process-output"></a>
#### 停用處理程序輸出

如果您的處理程序正在寫入大量您不感興趣的輸出，您可以透過完全停用輸出擷取來節省記憶體。要實現這一點，請在建構處理程序時呼叫 `quietly` 方法：

```php
use Illuminate\Support\Facades\Process;

$result = Process::quietly()->run('bash import.sh');
```

<a name="process-pipelines"></a>
### 管線

有時您可能希望將一個處理程序的輸出成為另一個處理程序的輸入。這通常被稱為將處理程序的輸出「管線化」到另一個處理程序。`Process` facades 提供的 `pipe` 方法使得這很容易實現。`pipe` 方法將同步執行管線化的處理程序，並返回管線中最後一個處理程序的結果：

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

如果您不需要自訂構成管線的個別處理程序，您可以直接將命令字串陣列傳遞給 `pipe` 方法：

```php
$result = Process::pipe([
    'cat example.txt',
    'grep -i "laravel"',
]);
```

處理程序輸出可以透過將一個閉包作為第二個引數傳遞給 `pipe` 方法來即時收集。該閉包將會接收兩個引數：輸出類型 (`stdout` 或 `stderr`) 和輸出字串本身：

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
})->start(function (string $type, string $output, string $key) {
    // ...
});
```

<a name="asynchronous-processes"></a>
## 非同步處理程序

儘管 `run` 方法同步地呼叫處理程序，但 `start` 方法可用於非同步地呼叫處理程序。這讓您的應用程式可以在處理程序於背景執行時，繼續執行其他任務。一旦處理程序被呼叫，您就可以利用 `running` 方法來判斷處理程序是否仍在執行：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    // ...
}

$result = $process->wait();
```

如您所見，您可以呼叫 `wait` 方法來等待處理程序執行完畢，並取得處理程序結果實例：

```php
$process = Process::timeout(120)->start('bash import.sh');

// ...

$result = $process->wait();
```

<a name="process-ids-and-signals"></a>
### 處理程序 ID 與訊號

`id` 方法可用於取得執行中處理程序由作業系統指派的處理程序 ID：

```php
$process = Process::start('bash import.sh');

return $process->id();
```

您可以使用 `signal` 方法向執行中的處理程序傳送一個「訊號」。預定義的訊號常數列表可在 [PHP 文件](https://www.php.net/manual/en/pcntl.constants.php)中找到：

```php
$process->signal(SIGUSR2);
```

<a name="asynchronous-process-output"></a>
### 非同步處理程序輸出

當非同步處理程序正在執行時，您可以使用 `output` 和 `errorOutput` 方法存取其全部目前輸出；然而，您可以使用 `latestOutput` 和 `latestErrorOutput` 來存取自上次取得輸出以來發生的處理程序輸出：

```php
$process = Process::timeout(120)->start('bash import.sh');

while ($process->running()) {
    echo $process->latestOutput();
    echo $process->latestErrorOutput();

    sleep(1);
}
```

與 `run` 方法一樣，也可以透過將閉包作為第二個引數傳遞給 `start` 方法，從非同步處理程序即時收集輸出。該閉包將接收兩個引數：輸出的「類型」（`stdout` 或 `stderr`）以及輸出字串本身：

```php
$process = Process::start('bash import.sh', function (string $type, string $output) {
    echo $output;
});

$result = $process->wait();
```

您可以使用 `waitUntil` 方法根據處理程序的輸出停止等待，而不是等到處理程序完成。當傳給 `waitUntil` 方法的閉包回傳 `true` 時，Laravel 將停止等待處理程序完成：

```php
$process = Process::start('bash import.sh');

$process->waitUntil(function (string $type, string $output) {
    return $output === 'Ready...';
});
```

<a name="concurrent-processes"></a>
## 並行處理程序

Laravel 還讓管理並行非同步處理程序的集區變得輕而易舉，讓您可以輕鬆同時執行許多任務。首先，呼叫 `pool` 方法，它接受一個閉包，該閉包會接收 `Illuminate\Process\Pool` 的實例。

在此閉包中，您可以定義屬於該集區的處理程序。一旦透過 `start` 方法啟動處理程序集區，您就可以透過 `running` 方法存取執行中處理程序的 [Collection](/docs/{{version}}/collections)：

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

如您所見，您可以等待所有集區處理程序執行完畢，並透過 `wait` 方法解析其結果。`wait` 方法會回傳一個可透過陣列存取的物件，讓您可以透過其鍵值存取集區中每個處理程序的結果實例：

```php
$results = $pool->wait();

echo $results[0]->output();
```

或者，為了方便起見，`concurrently` 方法可用於啟動一個非同步處理程序集區並立即等待其結果。當與 PHP 的陣列解構功能結合使用時，這可以提供特別具表達性的語法：

```php
[$first, $second, $third] = Process::concurrently(function (Pool $pool) {
    $pool->path(__DIR__)->command('ls -la');
    $pool->path(app_path())->command('ls -la');
    $pool->path(storage_path())->command('ls -la');
});

echo $first->output();
```

<a name="naming-pool-processes"></a>
### 命名集區處理程序

透過數字鍵存取處理程序集區結果不具表達性；因此，Laravel 允許您透過 `as` 方法為集區中的每個處理程序指派字串鍵。此鍵值也將傳遞給傳給 `start` 方法的閉包，讓您可以判斷輸出屬於哪個處理程序：

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
### 集區處理程序 ID 與訊號

由於處理程序集區的 `running` 方法提供集區中所有被呼叫處理程序的 Collection，您可以輕鬆存取底層集區處理程序 ID：

```php
$processIds = $pool->running()->each->id();
```

而且，為了方便起見，您可以對處理程序集區呼叫 `signal` 方法，向集區中每個處理程序傳送訊號：

```php
$pool->signal(SIGUSR2);
```

<a name="testing"></a>
## 測試

許多 Laravel 服務提供了功能，幫助你輕鬆且具表達性地撰寫測試，Laravel 的處理程序服務也不例外。`Process` Facade 的 `fake` 方法允許你指示 Laravel 在處理程序被呼叫時回傳模擬 / 虛擬結果。

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

當測試此路由時，我們可以透過呼叫 `Process` Facade 上不帶任何參數的 `fake` 方法，指示 Laravel 為每個被呼叫的處理程序回傳一個偽造的、成功的處理程序結果。此外，我們甚至可以 [斷言](#available-assertions) 一個指定的處理程序已被「執行」：

```php tab=Pest
<?php

use Illuminate\Process\PendingProcess;
use Illuminate\Contracts\Process\ProcessResult;
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

use Illuminate\Process\PendingProcess;
use Illuminate\Contracts\Process\ProcessResult;
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

如前所述，呼叫 `Process` Facade 上的 `fake` 方法將指示 Laravel 總是回傳一個沒有任何輸出的成功處理程序結果。然而，你可以輕鬆指定偽造處理程序的輸出和結束代碼，透過使用 `Process` Facade 的 `result` 方法：

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

如你可能已在之前的範例中注意到的，`Process` Facade 允許你透過將陣列傳遞給 `fake` 方法，為每個處理程序指定不同的偽造結果。

陣列的鍵應代表你希望偽造的指令模式及其相關聯的結果。`*` 字元可以用作萬用字元。任何未被偽造的處理程序指令將會實際被呼叫。你可以使用 `Process` Facade 的 `result` 方法來建構預設 / 偽造結果，用於這些指令：

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

如果你不需要自訂偽造處理程序的結束代碼或錯誤輸出，你可能會覺得將偽造的處理程序結果指定為簡單的字串更方便：

```php
Process::fake([
    'cat *' => 'Test "cat" output',
    'ls *' => 'Test "ls" output',
]);
```

<a name="faking-process-sequences"></a>
### 偽造處理程序序列

如果你正在測試的程式碼呼叫多個具有相同指令的處理程序，你可能會希望為每次處理程序呼叫指定不同的偽造處理程序結果。你可以透過 `Process` Facade 的 `sequence` 方法達成此目的：

```php
Process::fake([
    'ls *' => Process::sequence()
        ->push(Process::result('First invocation'))
        ->push(Process::result('Second invocation')),
]);
```

<a name="faking-asynchronous-process-lifecycles"></a>
### 偽造非同步處理程序生命週期

迄今為止，我們主要討論了偽造那些使用 `run` 方法同步呼叫的處理程序。然而，如果你正在嘗試測試與透過 `start` 呼叫的非同步處理程序互動的程式碼，你可能需要一個更複雜的方法來描述你的偽造處理程序。

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

為了正確偽造此處理程序，我們需要能夠描述 `running` 方法應該回傳 `true` 多少次。此外，我們可能希望指定多行輸出，應按順序回傳。為了達成此目的，我們可以使用 `Process` Facade 的 `describe` 方法：

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

讓我們深入探討上面的範例。使用 `output` 和 `errorOutput` 方法，我們可以指定多行輸出，將會按順序回傳。`exitCode` 方法可以用來指定偽造處理程序的最終結束代碼。最後，`iterations` 方法可以用來指定 `running` 方法應該回傳 `true` 多少次。

<a name="available-assertions"></a>
### 可用斷言

如 [前所述](#faking-processes)，Laravel 提供了幾個處理程序斷言，用於你的功能測試。我們將在下面討論這些斷言。

<a name="assert-process-ran"></a>
#### assertRan

斷言一個指定的處理程序已被呼叫：

```php
use Illuminate\Support\Facades\Process;

Process::assertRan('ls -la');
```

`assertRan` 方法也接受一個閉包，它將會接收一個處理程序實例與一個處理程序結果，讓你能夠檢查處理程序的配置選項。如果這個閉包回傳 `true`，該斷言將會「通過」：

```php
Process::assertRan(fn ($process, $result) =>
    $process->command === 'ls -la' &&
    $process->path === __DIR__ &&
    $process->timeout === 60
);
```

傳遞給 `assertRan` 閉包的 `$process` 是 `Illuminate\Process\PendingProcess` 的一個實例，而 `$result` 則是 `Illuminate\Contracts\Process\ProcessResult` 的一個實例。

<a name="assert-process-didnt-run"></a>
#### assertDidntRun

斷言一個指定的處理程序未被呼叫：

```php
use Illuminate\Support\Facades\Process;

Process::assertDidntRun('ls -la');
```

就像 `assertRan` 方法一樣，`assertDidntRun` 方法也接受一個閉包，它將會接收一個處理程序實例與一個處理程序結果，讓你能夠檢查處理程序的配置選項。如果這個閉包回傳 `true`，該斷言將會「失敗」：

```php
Process::assertDidntRun(fn (PendingProcess $process, ProcessResult $result) =>
    $process->command === 'ls -la'
);
```

<a name="assert-process-ran-times"></a>
#### assertRanTimes

斷言一個指定的處理程序已被呼叫指定次數：

```php
use Illuminate\Support\Facades\Process;

Process::assertRanTimes('ls -la', times: 3);
```

`assertRanTimes` 方法也接受一個閉包，它將會接收一個處理程序實例與一個處理程序結果，讓你能夠檢查處理程序的配置選項。如果這個閉包回傳 `true` 並且該處理程序已被呼叫指定次數，該斷言將會「通過」：

```php
Process::assertRanTimes(function (PendingProcess $process, ProcessResult $result) {
    return $process->command === 'ls -la';
}, times: 3);
```

<a name="preventing-stray-processes"></a>
### 防止逸散處理程序

如果您想確保所有呼叫的處理程序在個別測試或整個測試套件中都已被偽造，您可以呼叫 `preventStrayProcesses` 方法。呼叫此方法後，任何沒有對應偽造結果的處理程序將會拋出例外，而不是啟動實際的處理程序：

    use Illuminate\Support\Facades\Process;

    Process::preventStrayProcesses();

    Process::fake([
        'ls *' => 'Test output...',
    ]);

    // Fake response is returned...
    Process::run('ls -la');

    // An exception is thrown...
    Process::run('bash import.sh');