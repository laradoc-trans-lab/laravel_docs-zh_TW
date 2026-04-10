# Artisan 主控台

- [簡介](#introduction)
    - [Tinker (REPL)](#tinker)
- [撰寫指令](#writing-commands)
    - [產生指令](#generating-commands)
    - [指令結構](#command-structure)
    - [Closure 指令](#closure-commands)
    - [可隔離指令](#isolatable-commands)
- [定義輸入預期](#defining-input-expectations)
    - [引數](#arguments)
    - [選項](#options)
    - [輸入陣列](#input-arrays)
    - [輸入描述](#input-descriptions)
    - [提示缺失的輸入](#prompting-for-missing-input)
- [指令 I/O](#command-io)
    - [擷取輸入](#retrieving-input)
    - [提示輸入](#prompting-for-input)
    - [輸出內容](#writing-output)
- [註冊指令](#registering-commands)
- [以程式方式執行指令](#programmatically-executing-commands)
    - [從其他指令呼叫指令](#calling-commands-from-other-commands)
- [訊號處理](#signal-handling)
- [Stub 自訂](#stub-customization)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Artisan 是 Laravel 內建的命令列介面。Artisan 以 `artisan` 腳本的形式存在於應用程式的根目錄中，並提供許多實用的指令，能在您建構應用程式時提供協助。若要查看所有可用的 Artisan 指令列表，您可以使用 `list` 指令：

```shell
php artisan list
```

每個指令也都包含一個「說明」畫面，用於顯示並描述該指令可用的引數與選項。若要查看說明畫面，請在指令名稱前加上 `help`：

```shell
php artisan help migrate
```


<a name="laravel-sail"></a>
#### Laravel Sail

如果您使用 [Laravel Sail](/docs/{{version}}/sail) 作為本地開發環境，請記得使用 `sail` 命令列來呼叫 Artisan 指令。Sail 會在應用程式的 Docker 容器中執行您的 Artisan 指令：

```shell
./vendor/bin/sail artisan list
```


<a name="tinker"></a>
### Tinker (REPL)

[Laravel Tinker](https://github.com/laravel/tinker) 是為 Laravel 框架提供的強大 REPL，由 [PsySH](https://github.com/bobthecow/psysh) 套件驅動。


<a name="installation"></a>
#### 安裝

所有 Laravel 應用程式預設都包含 Tinker。然而，如果您之前將其從應用程式中移除，可以使用 Composer 安裝 Tinker：

```shell
composer require laravel/tinker
```

> [!NOTE]
> 正在尋找與 Laravel 應用程式互動時的熱重新載入、多行程式碼編輯和自動完成功能嗎？請看看 [Tinkerwell](https://tinkerwell.app)！


<a name="usage"></a>
#### 使用

Tinker 允許您在命令列上與整個 Laravel 應用程式進行互動，包括您的 Eloquent 模型、作業 (jobs)、事件 (events) 等。若要進入 Tinker 環境，請執行 `tinker` Artisan 指令：

```shell
php artisan tinker
```

您可以使用 `vendor:publish` 指令發布 Tinker 的設定檔：

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> [!WARNING]
> `dispatch` 輔助函數和 `Dispatchable` 類別上的 `dispatch` 方法依賴於垃圾回收來將作業放入佇列。因此，在使用 Tinker 時，您應該使用 `Bus::dispatch` 或 `Queue::push` 來發送作業。


<a name="command-allow-list"></a>
#### 指令允許清單

Tinker 使用「允許」清單來決定哪些 Artisan 指令允許在其 shell 中執行。預設情況下，您可以執行 `clear-compiled`、`down`、`env`、`inspire`、`migrate`、`migrate:install`、`up` 和 `optimize` 指令。如果您想允許更多指令，可以將其添加到 `tinker.php` 設定檔中的 `commands` 陣列：

```php
'commands' => [
    // App\Console\Commands\ExampleCommand::class,
],
```


<a name="classes-that-should-not-be-aliased"></a>
#### 不應設定別名的類別

通常，當您在 Tinker 中與類別互動時，Tinker 會自動為其設定別名。然而，您可能希望某些類別永遠不要被設定別名。您可以將這些類別列在 `tinker.php` 設定檔的 `dont_alias` 陣列中來實現此目的：

```php
'dont_alias' => [
    App\Models\User::class,
],
```

<a name="writing-commands"></a>
## 撰寫指令

除了 Artisan 提供的指令之外，您也可以建立自訂指令。指令通常儲存在 `app/Console/Commands` 目錄中；然而，只要您指示 Laravel [掃描其他目錄以尋找 Artisan 指令](#registering-commands)，您可以自由選擇儲存位置。


<a name="generating-commands"></a>
### 產生指令

若要建立新指令，您可以使用 `make:command` Artisan 指令。此指令將在 `app/Console/Commands` 目錄中建立一個新的指令類別。如果您的應用程式中不存在此目錄，請不必擔心 — 在您第一次執行 `make:command` Artisan 指令時，它將會被建立：

```shell
php artisan make:command SendEmails
```


<a name="command-structure"></a>
### 指令結構

產生指令後，您應該使用 `Signature` 與 `Description` 屬性來定義指令的簽署 (signature) 與描述。`Signature` 屬性還允許您定義 [指令的輸入預期](#defining-input-expectations)。當您的指令執行時，會呼叫 `handle` 方法。您可以將指令邏輯放置在此方法中。

讓我們來看看一個指令範例。請注意，我們能夠透過指令的 `handle` 方法請求任何需要的相依項目。Laravel [服務容器](/docs/{{version}}/container) 會自動注入此方法簽署中所有型別提示 (type-hinted) 的相依項目：

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Console\Attributes\Description;
use Illuminate\Console\Attributes\Signature;
use Illuminate\Console\Command;

#[Signature('mail:send {user}')]
#[Description('Send a marketing email to a user')]
class SendEmails extends Command
{
    /**
     * Execute the console command.
     */
    public function handle(DripEmailer $drip): void
    {
        $drip->send(User::find($this->argument('user')));
    }
}
```

> [!NOTE]
> 為了提高程式碼複用性，建議保持主控台指令簡潔，並將其任務委派給應用程式服務來完成。在上述範例中，請注意我們注入了一個服務類別來執行發送電子郵件的「繁重工作」。


<a name="exit-codes"></a>
#### 結束碼 (Exit Codes)

如果 `handle` 方法沒有回傳任何內容且指令成功執行，指令將以 `0` 結束碼 (exit code) 結束，表示成功。然而，`handle` 方法可以選擇性地回傳一個整數，以手動指定指令的結束碼：

```php
$this->error('Something went wrong.');

return 1;
```

如果您想從指令中的任何方法使指令「失敗」，可以使用 `fail` 方法。`fail` 方法將立即終止指令的執行並回傳結束碼 `1`：

```php
$this->fail('Something went wrong.');
```


<a name="closure-commands"></a>
### Closure 指令

基於 Closure 的指令提供了除了將主控台指令定義為類別之外的另一種選擇。正如路由 Closure 是控制器的替代方案，您可以將指令 Closure 視為指令類別的替代方案。

雖然 `routes/console.php` 檔案不定義 HTTP 路由，但它定義了進入您應用程式的主控台進入點 (路由)。在此檔案中，您可以使用 `Artisan::command` 方法定義所有基於 Closure 的主控台指令。`command` 方法接受兩個引數：[指令簽署](#defining-input-expectations) 以及一個接收指令引數與選項的 Closure：

```php
Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
});
```

該 Closure 會綁定到底層的指令實例，因此您可以完整存取通常在完整指令類別中可以存取的所有輔助方法。


<a name="type-hinting-dependencies"></a>
#### 相依項目的型別提示

除了接收指令的引數與選項外，指令 Closure 也可以對您希望從 [服務容器](/docs/{{version}}/container) 解析的額外相依項目進行型別提示：

```php
use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Support\Facades\Artisan;

Artisan::command('mail:send {user}', function (DripEmailer $drip, string $user) {
    $drip->send(User::find($user));
});
```


<a name="closure-command-descriptions"></a>
#### Closure 指令描述

定義基於 Closure 的指令時，您可以使用 `purpose` 方法為指令添加描述。當您執行 `php artisan list` 或 `php artisan help` 指令時，會顯示此描述：

```php
Artisan::command('mail:send {user}', function (string $user) {
    // ...
})->purpose('Send a marketing email to a user');
```


<a name="isolatable-commands"></a>
### 可隔離指令

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為預設快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器通訊。

有時您可能希望確保一次只能執行一個指令實例。若要達成此目的，您可以在指令類別中實作 `Illuminate\Contracts\Console\Isolatable` 介面：

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Contracts\Console\Isolatable;

class SendEmails extends Command implements Isolatable
{
    // ...
}
```

當您將指令標記為 `Isolatable` 時，Laravel 會自動為該指令提供 `--isolated` 選項，而無需在指令選項中明確定義。當指令使用該選項被呼叫時，Laravel 將確保沒有其他該指令的實例正在運行。Laravel 透過嘗試使用您應用程式的預設快取驅動程式來獲取原子鎖 (atomic lock) 來實現這一點。如果其他指令實例正在運行，該指令將不會執行；然而，指令仍將以成功的結束狀態碼結束：

```shell
php artisan mail:send 1 --isolated
```

如果您想指定指令在無法執行時應回傳的結束狀態碼，可以透過 `isolated` 選項提供所需的狀態碼：

```shell
php artisan mail:send 1 --isolated=12
```


<a name="lock-id"></a>
#### 鎖 ID (Lock ID)

預設情況下，Laravel 將使用指令名稱來產生用於在應用程式快取中獲取原子鎖的字串鍵 (key)。然而，您可以在 Artisan 指令類別中定義 `isolatableId` 方法來自訂此鍵，讓您可以將指令的引數或選項整合到鍵中：

```php
/**
 * Get the isolatable ID for the command.
 */
public function isolatableId(): string
{
    return $this->argument('user');
}
```


<a name="lock-expiration-time"></a>
#### 鎖過期時間

預設情況下，隔離鎖在指令結束後會過期。或者，如果指令被中斷且無法完成，鎖將在一小時後過期。然而，您可以在指令中定義 `isolationLockExpiresAt` 方法來調整鎖的過期時間：

```php
use DateTimeInterface;
use DateInterval;

/**
 * Determine when an isolation lock expires for the command.
 */
public function isolationLockExpiresAt(): DateTimeInterface|DateInterval
{
    return now()->plus(minutes: 5);
}
```

<a name="defining-input-expectations"></a>
## 定義輸入預期

在撰寫主控台指令時，通常會透過引數或選項來收集使用者的輸入。Laravel 讓您可以使用指令中的 `signature` 屬性，非常方便地定義您預期使用者提供的輸入。`signature` 屬性允許您使用單一且具表達力的類路由語法，來定義指令的名稱、引數以及選項。


<a name="arguments"></a>
### 引數

所有使用者提供的引數和選項都包含在花括號中。在以下範例中，該指令定義了一個必填引數：`user`：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user}';
```

您也可以將引數設為選填，或為引數定義預設值：

```php
// Optional argument...
'mail:send {user?}'

// Optional argument with default value...
'mail:send {user=foo}'
```


<a name="options"></a>
### 選項

選項與引數一樣，是另一種使用者輸入的形式。當透過命令列提供時，選項會以兩個連字號 (`--`) 作為前綴。選項分為兩種類型：接收值的選項和不接收值的選項。不接收值的選項充當布林值「開關」。讓我們來看看這種類型選項的範例：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue}';
```

在此範例中，在呼叫 Artisan 指令時可以指定 `--queue` 開關。如果傳遞了 `--queue` 開關，則該選項的值將為 `true`；否則，值將為 `false`：

```shell
php artisan mail:send 1 --queue
```


<a name="options-with-values"></a>
#### 帶有值的選項

接下來，讓我們看看一個需要值的選項。如果使用者必須為選項指定值，您應該在選項名稱後加上 `=` 符號：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue=}';
```

在此範例中，使用者可以像這樣傳遞選項值。如果在呼叫指令時未指定該選項，其值將為 `null`：

```shell
php artisan mail:send 1 --queue=default
```

您可以在選項名稱後指定預設值，以此為選項分配預設值。如果使用者沒有傳遞選項值，將使用預設值：

```php
'mail:send {user} {--queue=default}'
```


<a name="option-shortcuts"></a>
#### 選項快捷鍵

在定義選項時，若要分配快捷鍵，可以在選項名稱之前指定它，並使用 `|` 字元作為分隔符來區分快捷鍵與完整選項名稱：

```php
'mail:send {user} {--Q|queue=}'
```

在終端機呼叫指令時，選項快捷鍵應以前綴單個連字號開頭，且在指定選項值時不應包含 `=` 字元：

```shell
php artisan mail:send 1 -Qdefault
```


<a name="input-arrays"></a>
### 輸入陣列

如果您想要定義預期接收多個輸入值的引數或選項，可以使用 `*` 字元。首先，讓我們看看一個指定此類引數的範例：

```php
'mail:send {user*}'
```

執行此指令時，`user` 引數可以按順序傳遞到命令列。例如，以下指令將使 `user` 的值成為一個包含 `1` 和 `2` 的陣列：

```shell
php artisan mail:send 1 2
```

此 `*` 字元可以與選填引數定義結合，以允許零個或多個引數實例：

```php
'mail:send {user?*}'
```


<a name="option-arrays"></a>
#### 選項陣列

定義預期接收多個輸入值的選項時，傳遞給指令的每個選項值都應以前綴選項名稱開頭：

```php
'mail:send {--id=*}'
```

可以使用多個 `--id` 引數來呼叫此類指令：

```shell
php artisan mail:send --id=1 --id=2
```


<a name="input-descriptions"></a>
### 輸入描述

您可以使用冒號將引數名稱與描述分開，以此為輸入引數和選項分配描述。如果您需要更多空間來定義指令，可以隨意將定義分佈在多行中：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send
                        {user : The ID of the user}
                        {--queue : Whether the job should be queued}';
```


<a name="prompting-for-missing-input"></a>
### 提示缺失的輸入

如果您的指令包含必填引數，而使用者未提供，則會收到錯誤訊息。或者，您可以透過實作 `PromptsForMissingInput` 介面，將指令配置為在缺失必填引數時自動提示使用者：

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Contracts\Console\PromptsForMissingInput;

class SendEmails extends Command implements PromptsForMissingInput
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

    // ...
}
```

如果 Laravel 需要從使用者那裡收集必填引數，它會使用引數名稱或描述，以智能的方式措辭問題，自動詢問使用者該引數。如果您想自訂用於收集必填引數的問題，可以實作 `promptForMissingArgumentsUsing` 方法，並回傳一個以引數名稱為鍵的問題陣列：

```php
/**
 * Prompt for missing input arguments using the returned questions.
 *
 * @return array<string, string>
 */
protected function promptForMissingArgumentsUsing(): array
{
    return [
        'user' => 'Which user ID should receive the mail?',
    ];
}
```

您也可以使用包含問題和預留位置 (placeholder) 的元組 (tuple) 來提供預留文字：

```php
return [
    'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
];
```

如果您想要完全控制提示過程，可以提供一個 Closure，用於提示使用者並回傳其答案：

```php
use App\Models\User;
use function Laravel\Prompts\search;

// ...

return [
    'user' => fn () => search(
        label: 'Search for a user:',
        placeholder: 'E.g. Taylor Otwell',
        options: fn ($value) => strlen($value) > 0
            ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
            : []
    ),
];
```

> [!NOTE]
詳細的 [Laravel Prompts](/docs/{{version}}/prompts) 文件包含了關於可用提示及其用法的額外資訊。

如果您希望提示使用者選擇或輸入 [選項](#options)，可以在指令的 `handle` 方法中加入提示。但是，如果您只希望在使用者已被自動提示缺失引數時才提示他們，則可以實作 `afterPromptingForMissingArguments` 方法：

```php
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use function Laravel\Prompts\confirm;

// ...

/**
 * Perform actions after the user was prompted for missing arguments.
 */
protected function afterPromptingForMissingArguments(InputInterface $input, OutputInterface $output): void
{
    $input->setOption('queue', confirm(
        label: 'Would you like to queue the mail?',
        default: $this->option('queue')
    ));
}
```

<a name="command-io"></a>
## 指令 I/O


<a name="retrieving-input"></a>
### 擷取輸入

當您的指令正在執行時，您可能需要存取該指令所接受的引數 (arguments) 與選項 (options) 之值。若要達成此目的，您可以使用 `argument` 與 `option` 方法。如果該引數或選項不存在，將會回傳 `null`：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $userId = $this->argument('user');
}
```

如果您需要將所有引數以 `array` 形式擷取，請呼叫 `arguments` 方法：

```php
$arguments = $this->arguments();
```

使用 `option` 方法擷取選項與擷取引數一樣簡單。若要將所有選項以陣列形式擷取，請呼叫 `options` 方法：

```php
// Retrieve a specific option...
$queueName = $this->option('queue');

// Retrieve all options as an array...
$options = $this->options();
```


<a name="prompting-for-input"></a>
### 提示輸入

> [!NOTE]
> [Laravel Prompts](/docs/{{version}}/prompts) 是一個 PHP 套件，可用於為您的命令列應用程式添加美觀且使用者友好的表單，並提供類似瀏覽器的功能，包括預留文字 (placeholder text) 與驗證。

除了顯示輸出內容外，您也可以在指令執行過程中要求使用者提供輸入。`ask` 方法會使用給定的問題提示使用者，接受他們的輸入，然後將該輸入回傳給您的指令：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $name = $this->ask('What is your name?');

    // ...
}
```

`ask` 方法還接受一個可選的第二個引數，用於指定在未提供使用者輸入時應回傳的預設值：

```php
$name = $this->ask('What is your name?', 'Taylor');
```

`secret` 方法與 `ask` 類似，但使用者在主控台輸入時，其輸入內容將不會對他們可見。此方法在詢問密碼等敏感資訊時非常有用：

```php
$password = $this->secret('What is the password?');
```


<a name="asking-for-confirmation"></a>
#### 要求確認

如果您需要要求使用者進行簡單的「是或否」確認，可以使用 `confirm` 方法。預設情況下，此方法將回傳 `false`。然而，如果使用者在回應提示時輸入 `y` 或 `yes`，該方法將回傳 `true`。

```php
if ($this->confirm('Do you wish to continue?')) {
    // ...
}
```

如有必要，您可以透過將 `true` 作為 `confirm` 方法的第二個引數，來指定確認提示預設應回傳 `true`：

```php
if ($this->confirm('Do you wish to continue?', true)) {
    // ...
}
```


<a name="auto-completion"></a>
#### 自動完成

`anticipate` 方法可用於為可能的選項提供自動完成。無論自動完成的提示為何，使用者仍然可以提供任何答案：

```php
$name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);
```

或者，您也可以將 closure 作為 `anticipate` 方法的第二個引數。每當使用者輸入一個字元時，該 closure 都會被呼叫。該 closure 應接受一個包含目前使用者輸入內容的字串參數，並回傳一個用於自動完成的選項陣列：

```php
use App\Models\Address;

$name = $this->anticipate('What is your address?', function (string $input) {
    return Address::whereLike('name', "{$input}%")
        ->limit(5)
        ->pluck('name')
        ->all();
});
```


<a name="multiple-choice-questions"></a>
#### 多項選擇問題

如果您在詢問問題時需要給使用者一組預定義的選擇，可以使用 `choice` 方法。您可以透過將索引作為該方法的第三個引數，來設定在未選擇任何選項時應回傳的預設值陣列索引：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex
);
```

此外，`choice` 方法還接受可選的第四個和第五個引數，用於決定選擇有效回應的最大嘗試次數，以及是否允許複選：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex,
    $maxAttempts = null,
    $allowMultipleSelections = false
);
```


<a name="writing-output"></a>
### 輸出內容

若要將輸出發送到主控台，您可以使用 `line`、`newLine`、`info`、`comment`、`question`、`warn`、`alert` 與 `error` 方法。這些方法中的每一個都會根據其用途使用適當的 ANSI 顏色。例如，讓我們向使用者顯示一些一般資訊。通常，`info` 方法在主控台中會以綠色文字顯示：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    // ...

    $this->info('The command was successful!');
}
```

若要顯示錯誤訊息，請使用 `error` 方法。錯誤訊息文字通常以紅色顯示：

```php
$this->error('Something went wrong!');
```

您可以使用 `line` 方法來顯示純粹且無顏色的文字：

```php
$this->line('Display this on the screen');
```

您可以使用 `newLine` 方法來顯示一個空白行：

```php
// Write a single blank line...
$this->newLine();

// Write three blank lines...
$this->newLine(3);
```


<a name="tables"></a>
#### 表格

`table` 方法可以輕鬆地正確格式化多行/多欄的資料。您只需要提供欄位名稱和表格資料，Laravel 就會自動為您計算表格的適當寬度與高度：

```php
use App\Models\User;

$this->table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```


<a name="progress-bars"></a>
#### 進度條

對於執行時間較長的任務，顯示一個告知使用者任務完成進度的進度條會非常有幫助。使用 `withProgressBar` 方法，Laravel 會顯示一個進度條，並在每次疊代給定的可疊代值時推進其進度：

```php
use App\Models\User;

$users = $this->withProgressBar(User::all(), function (User $user) {
    $this->performTask($user);
});
```

有時，您可能需要對進度條如何推進有更多手動控制。首先，定義過程將要疊代的總步數。然後，在處理每個項目後推進進度條：

```php
$users = App\Models\User::all();

$bar = $this->output->createProgressBar(count($users));

$bar->start();

foreach ($users as $user) {
    $this->performTask($user);

    $bar->advance();
}

$bar->finish();
```

> [!NOTE]
> 如需更多進階選項，請參閱 [Symfony Progress Bar 元件文件](https://symfony.com/doc/current/components/console/helpers/progressbar.html)。


<a name="registering-commands"></a>
## 註冊指令

預設情況下，Laravel 會自動註冊 `app/Console/Commands` 目錄中的所有指令。不過，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `withCommands` 方法，指示 Laravel 掃描其他目錄以尋找 Artisan 指令：

```php
->withCommands([
    __DIR__.'/../app/Domain/Orders/Commands',
])
```

如有必要，您也可以透過將指令的類別名稱提供給 `withCommands` 方法來手動註冊指令：

```php
use App\Domain\Orders\Commands\SendEmails;

->withCommands([
    SendEmails::class,
])
```

當 Artisan 啟動時，應用程式中的所有指令都將由 [服務容器(service container)](/docs/{{version}}/container) 解析並註冊到 Artisan 中。

<a name="programmatically-executing-commands"></a>
## 以程式方式執行指令

有時候您可能希望在 CLI 之外執行 Artisan 指令。例如，您可能希望從路由或控制器中執行 Artisan 指令。您可以使用 `Artisan` facade 的 `call` 方法來達成此目的。`call` 方法的第一個引數可接收指令的簽名名稱 (signature name) 或類別名稱，第二個引數則為包含指令參數的陣列。該方法將回傳結束代碼 (exit code)：

```php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Route;

Route::post('/user/{user}/mail', function (string $user) {
    $exitCode = Artisan::call('mail:send', [
        'user' => $user, '--queue' => 'default'
    ]);

    // ...
});
```

或者，您也可以將完整的 Artisan 指令作為字串傳遞給 `call` 方法：

```php
Artisan::call('mail:send 1 --queue=default');
```


<a name="passing-array-values"></a>
#### 傳遞陣列值

如果您的指令定義了一個接收陣列的選項，您可以將值陣列傳遞給該選項：

```php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Route;

Route::post('/mail', function () {
    $exitCode = Artisan::call('mail:send', [
        '--id' => [5, 13]
    ]);
});
```


<a name="passing-boolean-values"></a>
#### 傳遞布林值

如果您需要指定一個不接收字串值的選項之值，例如 `migrate:refresh` 指令中的 `--force` 旗標，您應該傳遞 `true` 或 `false` 作為該選項的值：

```php
$exitCode = Artisan::call('migrate:refresh', [
    '--force' => true,
]);
```


<a name="queueing-artisan-commands"></a>
#### 將 Artisan 指令加入佇列

使用 `Artisan` facade 的 `queue` 方法，您甚至可以将 Artisan 指令加入佇列，以便由您的 [佇列工作者(queue workers)](/docs/{{version}}/queues) 在背景處理。在使用此方法之前，請確保您已配置好佇列並正在執行佇列監聽程式：

```php
use Illuminate\Support\Facades\Artisan;
use Illuminate\Support\Facades\Route;

Route::post('/user/{user}/mail', function (string $user) {
    Artisan::queue('mail:send', [
        'user' => $user, '--queue' => 'default'
    ]);

    // ...
});
```

使用 `onConnection` 和 `onQueue` 方法，您可以指定 Artisan 指令應該被派遣到哪個連線或佇列：

```php
Artisan::queue('mail:send', [
    'user' => 1, '--queue' => 'default'
])->onConnection('redis')->onQueue('commands');
```


<a name="calling-commands-from-other-commands"></a>
### 從其他指令呼叫指令

有時候您可能希望從現有的 Artisan 指令中呼叫其他指令。您可以使用 `call` 方法來達成。此 `call` 方法接收指令名稱以及包含指令引數 / 選項的陣列：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $this->call('mail:send', [
        'user' => 1, '--queue' => 'default'
    ]);

    // ...
}
```

如果您想呼叫另一個主控台指令並抑制其所有輸出，可以使用 `callSilently` 方法。`callSilently` 方法的簽名與 `call` 方法相同：

```php
$this->callSilently('mail:send', [
    'user' => 1, '--queue' => 'default'
]);
```


<a name="signal-handling"></a>
## 訊號處理

如您所知，作業系統允許將訊號發送到正在執行的行程(Processes)。例如，`SIGTERM` 訊號是作業系統要求程式終止的方式。如果您希望在 Artisan 主控台指令中監聽訊號，並在訊號發生時執行程式碼，可以使用 `trap` 方法：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $this->trap(SIGTERM, fn () => $this->shouldKeepRunning = false);

    while ($this->shouldKeepRunning) {
        // ...
    }
}
```

若要一次監聽多個訊號，您可以將訊號陣列提供給 `trap` 方法：

```php
$this->trap([SIGTERM, SIGQUIT], function (int $signal) {
    $this->shouldKeepRunning = false;

    dump($signal); // SIGTERM / SIGQUIT
});
```


<a name="stub-customization"></a>
## Stub 自訂

Artisan 主控台的 `make` 指令用於建立各種類別，例如控制器、作業 (jobs)、遷移 (migrations) 和測試。這些類別是使用 "stub" 檔案產生的，其內容會根據您的輸入來填充。然而，您可能想要對 Artisan 產生的檔案進行微調。為了達成此目的，您可以使用 `stub:publish` 指令將最常用的 stub 發佈到您的應用程式中，以便進行自訂：

```shell
php artisan stub:publish
```

發佈後的 stub 將位於應用程式根目錄的 `stubs` 目錄中。您對這些 stub 所做的任何更改，在之後使用 Artisan 的 `make` 指令產生對應類別時都會生效。


<a name="events"></a>
## 事件

Artisan 在執行指令時會派遣三個事件：`Illuminate\Console\Events\ArtisanStarting`、`Illuminate\Console\Events\CommandStarting` 以及 `Illuminate\Console\Events\CommandFinished`。`ArtisanStarting` 事件在 Artisan 開始執行時立即派遣。接著，`CommandStarting` 事件會在指令執行前立即派遣。最後，`CommandFinished` 事件會在指令執行完畢後派遣。