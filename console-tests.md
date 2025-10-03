# 主控台測試

- [簡介](#introduction)
- [成功 / 失敗預期](#success-failure-expectations)
- [輸入 / 輸出預期](#input-output-expectations)
- [主控台事件](#console-events)

<a name="introduction"></a>
## 簡介

除了簡化 HTTP 測試之外，Laravel 還為測試應用程式的 [自訂主控台指令](/docs/{{version}}/artisan) 提供了一個簡潔的 API。


<a name="success-failure-expectations"></a>
## 成功 / 失敗預期

首先，讓我們探討如何針對 Artisan 指令的結束代碼進行斷言。為此，我們將使用 `artisan` 方法從測試中呼叫 Artisan 指令。接著，我們將使用 `assertExitCode` 方法斷言該指令已以指定的結束代碼完成：

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

您可以使用 `assertNotExitCode` 方法來斷言該指令並未以指定的結束代碼結束：

```php
$this->artisan('inspire')->assertNotExitCode(1);
```

當然，所有終端機指令在成功時通常會以狀態碼 `0` 結束，而在不成功時會以非零的結束代碼結束。因此，為了方便起見，您可以使用 `assertSuccessful` 和 `assertFailed` 斷言來斷言指定指令是否以成功的結束代碼結束：

```php
$this->artisan('inspire')->assertSuccessful();

$this->artisan('inspire')->assertFailed();
```


<a name="input-output-expectations"></a>
## 輸入 / 輸出預期

Laravel 允許您使用 `expectsQuestion` 方法輕鬆「模擬」主控台指令的使用者輸入。此外，您可以使用 `assertExitCode` 和 `expectsOutput` 方法來指定您預期主控台指令會輸出的結束代碼和文字。例如，考慮以下主控台指令：

```php
Artisan::command('question', function () {
    $name = $this->ask('What is your name?');

    $language = $this->choice('Which language do you prefer?', [
        'PHP',
        'Ruby',
        'Python',
    ]);

    $this->line('Your name is '.$name.' and you prefer '.$language.'.');
});
```

您可以使用以下測試來測試此指令：

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

如果您正在利用 [Laravel Prompts](/docs/{{version}}/prompts) 提供的 `search` 或 `multisearch` 函式，您可以使用 `expectsSearch` 斷言來模擬使用者的輸入、搜尋結果和選擇：

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

您也可以使用 `doesntExpectOutput` 方法來斷言主控台指令沒有產生任何輸出：

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

`expectsOutputToContain` 和 `doesntExpectOutputToContain` 方法可用於針對輸出的一部分進行斷言：

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

當您編寫一個預期以「是」或「否」回答來進行確認的指令時，您可以使用 `expectsConfirmation` 方法：

```php
$this->artisan('module:import')
    ->expectsConfirmation('Do you really wish to run this command?', 'no')
    ->assertExitCode(1);
```


<a name="table-expectations"></a>
#### 表格預期

如果您的指令使用 Artisan 的 `table` 方法顯示資訊表格，那麼為整個表格編寫輸出預期可能會很繁瑣。相反地，您可以使用 `expectsTable` 方法。此方法接受表格標頭作為其第一個參數，並接受表格資料作為其第二個參數：

```php
$this->artisan('users:all')
    ->expectsTable([
        'ID',
        'Email',
    ], [
        [1, 'taylor@example.com'],
        [2, 'abigail@example.com'],
    ]);
```


<a name="console-events"></a>
## 主控台事件

預設情況下，當執行應用程式的測試時，`Illuminate\Console\Events\CommandStarting` 和 `Illuminate\Console\Events\CommandFinished` 事件不會被分派。然而，您可以透過將 `Illuminate\Foundation\Testing\WithConsoleEvents` Trait 加入到指定的測試類別中來啟用這些事件：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\WithConsoleEvents;

pest()->use(WithConsoleEvents::class);

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