# 主控台測試

- [簡介](#introduction)
- [成功 / 失敗預期](#success-failure-expectations)
- [輸入 / 輸出預期](#input-output-expectations)
- [主控台事件](#console-events)

<a name="introduction"></a>
## 簡介

除了簡化 HTTP 測試之外，Laravel 也提供了一個簡單的 API，用於測試應用程式的[自訂主控台命令](/docs/{{version}}/artisan)。

<a name="success-failure-expectations"></a>
## 成功 / 失敗預期

首先，讓我們探討如何針對 Artisan 命令的結束代碼進行斷言。為此，我們將使用 `artisan` 方法從測試中呼叫 Artisan 命令。然後，我們將使用 `assertExitCode` 方法斷言該命令以給定的結束代碼完成：

```php tab=Pest
test('console command', function () {
    $this->artisan('inspire')->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('inspire')->assertExitCode(0);
}
```

您可以使用 `assertNotExitCode` 方法來斷言該命令沒有以給定的結束代碼結束：

    $this->artisan('inspire')->assertNotExitCode(1);

當然，所有終端機命令在成功時通常會以狀態碼 `0` 結束，而在不成功時則以非零的結束代碼結束。因此，為了方便起見，您可以使用 `assertSuccessful` 和 `assertFailed` 斷言來斷言給定的命令是否以成功的結束代碼結束：

    $this->artisan('inspire')->assertSuccessful();

    $this->artisan('inspire')->assertFailed();

<a name="input-output-expectations"></a>
## 輸入 / 輸出預期

Laravel 允許您使用 `expectsQuestion` 方法輕鬆地「模擬」主控台命令的使用者輸入。此外，您可以使用 `assertExitCode` 和 `expectsOutput` 方法指定您預期主控台命令輸出的結束代碼和文字。例如，考慮以下主控台命令：

    Artisan::command('question', function () {
        $name = $this->ask('What is your name?');

        $language = $this->choice('Which language do you prefer?', [
            'PHP',
            'Ruby',
            'Python',
        ]);

        $this->line('Your name is '.$name.' and you prefer '.$language.'.');
    });

您可以使用以下測試來測試此命令：

```php tab=Pest
test('console command', function () {
    $this->artisan('question')
        ->expectsQuestion('What is your name?', 'Taylor Otwell')
        ->expectsQuestion('Which language do you prefer?', 'PHP')
        ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
        ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
        ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('question')
        ->expectsQuestion('What is your name?', 'Taylor Otwell')
        ->expectsQuestion('Which language do you prefer?', 'PHP')
        ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
        ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
        ->assertExitCode(0);
}
```

如果您正在使用 [Laravel Prompts](/docs/{{version}}/prompts) 提供的 `search` 或 `multisearch` 函數，您可以使用 `expectsSearch` 斷言來模擬使用者的輸入、搜尋結果和選擇：

```php tab=Pest
test('console command', function () {
    $this->artisan('example')
        ->expectsSearch('What is your name?', search: 'Tay', answers: [
            'Taylor Otwell',
            'Taylor Swift',
            'Darian Taylor'
        ], answer: 'Taylor Otwell')
        ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('example')
        ->expectsSearch('What is your name?', search: 'Tay', answers: [
            'Taylor Otwell',
            'Taylor Swift',
            'Darian Taylor'
        ], answer: 'Taylor Otwell')
        ->assertExitCode(0);
}
```

您也可以使用 `doesntExpectOutput` 方法斷言主控台命令不會產生任何輸出：

```php tab=Pest
test('console command', function () {
    $this->artisan('example')
        ->doesntExpectOutput()
        ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('example')
            ->doesntExpectOutput()
            ->assertExitCode(0);
}
```

`expectsOutputToContain` 和 `doesntExpectOutputToContain` 方法可用於對輸出的一部分進行斷言：

```php tab=Pest
test('console command', function () {
    $this->artisan('example')
        ->expectsOutputToContain('Taylor')
        ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('example')
            ->expectsOutputToContain('Taylor')
            ->assertExitCode(0);
}
```

<a name="confirmation-expectations"></a>
#### 確認預期

當編寫一個預期「是」或「否」答案形式的確認命令時，您可以使用 `expectsConfirmation` 方法：

    $this->artisan('module:import')
        ->expectsConfirmation('Do you really wish to run this command?', 'no')
        ->assertExitCode(1);

<a name="table-expectations"></a>
#### 表格預期

如果您的命令使用 Artisan 的 `table` 方法顯示資訊表格，為整個表格編寫輸出預期可能會很麻煩。您可以改用 `expectsTable` 方法。此方法將表格的標頭作為其第一個引數，將表格的資料作為其第二個引數：

    $this->artisan('users:all')
        ->expectsTable([
            'ID',
            'Email',
        ], [
            [1, 'taylor @example.com'],
            [2, 'abigail @example.com'],
        ]);

<a name="console-events"></a>
## 主控台事件

預設情況下，`Illuminate\Console\Events\CommandStarting` 和 `Illuminate\Console\Events\CommandFinished` 事件在執行應用程式測試時不會被分派。但是，您可以透過將 `Illuminate\Foundation\Testing\WithConsoleEvents` trait 加入到類別中，為給定的測試類別啟用這些事件：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\WithConsoleEvents;

uses(WithConsoleEvents::class);

// ...
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\WithConsoleEvents;
use Tests\TestCase;

class ConsoleEventTest extends TestCase
{
    use WithConsoleEvents;

    // ...
}
```
