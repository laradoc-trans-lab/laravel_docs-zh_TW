# 命令列測試

- [介紹](#introduction)
- [成功 / 失敗預期](#success-failure-expectations)
- [輸入 / 輸出預期](#input-output-expectations)
- [命令列事件](#console-events)

<a name="introduction"></a>
## 介紹

除了簡化 HTTP 測試外，Laravel 還提供了一個簡單的 API 用於測試應用程式的[自訂命令列指令](/docs/{{version}}/artisan)。

<a name="success-failure-expectations"></a>
## 成功 / 失敗預期

首先，讓我們探討如何對 Artisan 指令的結束碼進行斷言。為此，我們將在測試中使用 `artisan` 方法來呼叫 Artisan 指令。然後，我們將使用 `assertExitCode` 方法來斷言指令以給定的結束碼完成：

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

您可以使用 `assertNotExitCode` 方法來斷言指令未以給定的結束碼結束：

    $this->artisan('inspire')->assertNotExitCode(1);

當然，所有終端機指令在成功時通常以狀態碼 `0` 結束，而在不成功時則以非零結束碼結束。因此，為方便起見，您可以利用 `assertSuccessful` 和 `assertFailed` 斷言來斷言給定指令是否以成功結束碼結束：

    $this->artisan('inspire')->assertSuccessful();

    $this->artisan('inspire')->assertFailed();

<a name="input-output-expectations"></a>
## 輸入 / 輸出預期

Laravel 允許您使用 `expectsQuestion` 方法輕鬆「模擬」命令列指令的使用者輸入。此外，您可以使用 `assertExitCode` 和 `expectsOutput` 方法來指定您預期命令列指令輸出的結束碼和文字。例如，考慮以下命令列指令：

    Artisan::command('question', function () {
        $name = $this->ask('What is your name?');

        $language = $this->choice('Which language do you prefer?', [
            'PHP',
            'Ruby',
            'Python',
        ]);

        $this->line('Your name is '.$name.' and you prefer '.$language.'.');
    });

您可以透過以下測試來測試此指令：

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

如果您正在利用 [Laravel Prompts](/docs/{{version}}/prompts) 提供的 `search` 或 `multisearch` 函數，您可以使用 `expectsSearch` 斷言來模擬使用者輸入、搜尋結果和選擇：

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

您也可以使用 `doesntExpectOutput` 方法來斷言命令列指令未產生任何輸出：

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

可以使用 `expectsOutputToContain` 和 `doesntExpectOutputToContain` 方法來對部分輸出進行斷言：

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

在編寫需要「是」或「否」形式確認的指令時，您可以使用 `expectsConfirmation` 方法：

    $this->artisan('module:import')
        ->expectsConfirmation('Do you really wish to run this command?', 'no')
        ->assertExitCode(1);

<a name="table-expectations"></a>
#### 表格預期

如果您的指令使用 Artisan 的 `table` 方法顯示資訊表格，那麼為整個表格編寫輸出預期可能會很繁瑣。您可以改用 `expectsTable` 方法。此方法接受表格標頭作為其第一個參數，並接受表格資料作為其第二個參數：

    $this->artisan('users:all')
        ->expectsTable([
            'ID',
            'Email',
        ], [
            [1, 'taylor@example.com'],
            [2, 'abigail@example.com'],
        ]);

<a name="console-events"></a>
## 命令列事件

依預設，在執行應用程式的測試時，`Illuminate\Console\Events\CommandStarting` 和 `Illuminate\Console\Events\CommandFinished` 事件不會被分派。但是，您可以透過將 `Illuminate\Foundation\Testing\WithConsoleEvents` Trait 添加到類別中，為給定的測試類別啟用這些事件：

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