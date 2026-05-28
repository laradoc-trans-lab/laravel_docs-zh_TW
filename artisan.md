# Artisan 主控台

- [介紹](#introduction)
    - [Tinker (REPL)](#tinker)
- [撰寫指令](#writing-commands)
    - [產生指令](#generating-commands)
    - [指令結構](#command-structure)
    - [Closure 指令](#closure-commands)
    - [可隔離的指令](#isolatable-commands)
- [定義預期的輸入](#defining-input-expectations)
    - [引數](#arguments)
    - [選項](#options)
    - [輸入陣列](#input-arrays)
    - [輸入描述](#input-descriptions)
    - [提示缺少輸入](#prompting-for-missing-input)
- [指令 I/O](#command-io)
    - [取得輸入](#retrieving-input)
    - [提示輸入](#prompting-for-input)
    - [輸出內容](#writing-output)
- [註冊指令](#registering-commands)
- [以程式化方式執行指令](#programmatically-executing-commands)
    - [在指令中呼叫其他指令](#calling-commands-from-other-commands)
- [訊號處理](#signal-handling)
- [Stub 自訂](#stub-customization)
- [事件](#events)

<a name="introduction"></a>
## 介紹

Artisan 是 Laravel 內建的命令列介面。Artisan 以 `artisan` 指令碼的形式存在於應用程式的根目錄中，並提供了許多實用的指令，可在你建立應用程式時提供協助。若要查看所有可用的 Artisan 指令清單，可以使用 `list` 指令：

```shell
php artisan list
```

每個指令也都包含一個「說明 (help)」畫面，用來顯示並描述該指令可用的引數與選項。若要查看說明畫面，請在指令名稱前加上 `help`：

```shell
php artisan help migrate
```


<a name="laravel-sail"></a>
#### Laravel Sail

如果你使用 [Laravel Sail](/docs/{{version}}/sail) 作為本地開發環境，請記得使用 `sail` 命令列來呼叫 Artisan 指令。Sail 會在你應用程式的 Docker 容器中執行你的 Artisan 指令：

```shell
./vendor/bin/sail artisan list
```


<a name="tinker"></a>
### Tinker (REPL)

[Laravel Tinker](https://github.com/laravel/tinker) 是一款適用於 Laravel 框架的強大 REPL，由 [PsySH](https://github.com/bobthecow/psysh) 套件驅動。


<a name="installation"></a>
#### 安裝

所有 Laravel 應用程式預設都已包含 Tinker。不過，如果你之前曾從應用程式中移除過 Tinker，可以使用 Composer 重新安裝：

```shell
composer require laravel/tinker
```

> [!NOTE]
> 想要在與 Laravel 應用程式互動時，擁有熱重載 (Hot reloading)、多行程式碼編輯以及自動完成功能嗎？快來試試 [Tinkerwell](https://tinkerwell.app)！


<a name="usage"></a>
#### 使用方式

Tinker 允許你在命令列上與整個 Laravel 應用程式進行互動，包含你的 Eloquent 模型、任務 (Job)、事件等。若要進入 Tinker環境，請執行 `tinker` Artisan 指令：

```shell
php artisan tinker
```

你可以使用 `vendor:publish` 指令來發布 Tinker 的設定檔：

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> [!WARNING]
> `dispatch` 輔助函式和 `Dispatchable` 類別上的 `dispatch` 方法，需依賴垃圾回收機制來將任務放入佇列中。因此，在使用 Tinker 時，你應該使用 `Bus::dispatch` 或 `Queue::push` 來發送任務。


<a name="command-allow-list"></a>
#### 指令允許清單

Tinker 使用一個「允許」清單來決定哪些 Artisan 指令可以在其 shell 中執行。預設情況下，你可以執行 `clear-compiled`、`down`、`env`、`inspire`、`migrate`、`migrate:install`、`up` 和 `optimize` 指令。如果你想允許更多指令，可以將它們加入到 `tinker.php` 設定檔中的 `commands` 陣列中：

```php
'commands' => [
    // App\Console\Commands\ExampleCommand::class,
],
```


<a name="classes-that-should-not-be-aliased"></a>
#### 不應為其設定別名的類別

通常，當你在 Tinker 中與類別進行互動時，Tinker 會自動為其設定別名。然而，你可能不希望某些類別被設定別名。你可以透過在 `tinker.php` 設定檔的 `dont_alias` 陣列中列出這些類別來達到這個目的：

```php
'dont_alias' => [
    App\Models\User::class,
],
```

<a name="writing-commands"></a>
## 撰寫指令

除了 Artisan 提供的指令之外，您也可以建立自訂的指令。指令通常會儲存在 `app/Console/Commands` 目錄中；不過，只要您指示 Laravel [掃描其他目錄以尋找 Artisan 指令](#registering-commands)，您就可以自由選擇要儲存在哪裡。


<a name="generating-commands"></a>
### 產生指令

要建立一個新的指令，您可以使用 `make:command` 這個 Artisan 指令。此指令會在 `app/Console/Commands` 目錄中建立一個新的指令類別。如果您的應用程式中沒有這個目錄也不必擔心——在您第一次執行 `make:command` Artisan 指令時，系統就會自動建立該目錄：

```shell
php artisan make:command SendEmails
```


<a name="command-structure"></a>
### 指令結構

產生指令之後，您應該使用 `Signature` 和 `Description` 屬性（Attribute）來定義指令的簽章與描述。`Signature` 屬性還能讓您定義 [您的指令所預期的輸入](#defining-input-expectations)。當執行指令時，系統會呼叫 `handle` 方法。您可以將指令的邏輯寫在這個方法中。

讓我們來看一個範例指令。請注意，我們可以透過指令的 `handle` 方法來請求任何我們需要的依賴項目。Laravel 的 [服務容器](/docs/{{version}}/container) 會自動注入在此方法簽章中型別提示（Type-hinted）的所有依賴項目：

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
> 為了提高程式碼的重複使用率，良好的實作習慣是保持主控台指令的精簡，並將實際任務交由應用程式服務來完成。在上述範例中，請注意我們注入了一個服務類別來執行發送電子郵件的「繁重工作」。


<a name="exit-codes"></a>
#### 結束碼

如果 `handle` 方法沒有回傳任何內容且指令成功執行，則該指令將會以結束碼 `0` 結束，表示成功。不過，`handle` 方法也可以選擇性地回傳一個整數，以手動指定指令的結束碼：

```php
$this->error('Something went wrong.');

return 1;
```

如果您想在指令內的任何方法中使該指令「失敗」，可以使用 `fail` 方法。`fail` 方法會立即終止指令的執行，並回傳結束碼 `1`：

```php
$this->fail('Something went wrong.');
```


<a name="closure-commands"></a>
### Closure 指令

基於 Closure 的指令提供了另一種將主控台指令定義為類別的替代方案。就像路由 Closure 是控制器的替代方案一樣，您可以將指令 Closure 視為指令類別的替代方案。

雖然 `routes/console.php` 檔案並非用來定義 HTTP 路由，但它定義了應用程式中基於主控台的進入點（路由）。在此檔案中，您可以使用 `Artisan::command` 方法來定義所有基於 Closure 的主控台指令。`command` 方法接受兩個引數：[指令簽章](#defining-input-expectations) 以及一個接收該指令引數與選項的 Closure：

```php
Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
});
```

此 Closure 會綁定到背後的指令實例，因此您可以完整使用通常在完整指令類別中可以存取的所有輔助方法。


<a name="type-hinting-dependencies"></a>
#### 型別提示依賴項目

除了接收您的指令引數和選項之外，指令 Closure 還可以對您希望從 [服務容器](/docs/{{version}}/container) 解析出來的其他依賴項目進行型別提示：

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

定義基於 Closure 的指令時，您可以使用 `purpose` 方法為該指令新增描述。當您執行 `php artisan list` 或 `php artisan help` 指令時，將會顯示此描述：

```php
Artisan::command('mail:send {user}', function (string $user) {
    // ...
})->purpose('Send a marketing email to a user');
```


<a name="isolatable-commands"></a>
### 可隔離的指令

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器進行通訊。

有時您可能希望確保同一時間只有一個指令實例在執行。若要達到此目的，您可以在您的指令類別上實作 `Illuminate\Contracts\Console\Isolatable` 介面：

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

當您將指令標記為 `Isolatable` 時，Laravel 會自動為該指令提供 `--isolated` 選項，而不需要在指令的選項中明確定義它。當使用該選項呼叫該指令時，Laravel 會確保沒有其他該指令的實例正在執行。Laravel 是藉由嘗試使用應用程式的預設快取驅動程式取得原子鎖（Atomic Lock）來實現這一點。如果該指令的其他實例正在執行，則該指令將不會執行；不過，該指令仍然會以代表成功的結束狀態碼結束：

```shell
php artisan mail:send 1 --isolated
```

如果您想指定指令在無法執行時應回傳的結束狀態碼，可以透過 `isolated` 選項提供所需的狀態碼：

```shell
php artisan mail:send 1 --isolated=12
```


<a name="lock-id"></a>
#### 鎖定 ID

預設情況下，Laravel 會使用指令的名稱來產生用於在應用程式快取中取得原子鎖的字串鍵值（Key）。不過，您可以透過在 Artisan 指令類別中定義 `isolatableId` 方法來客製化此鍵值，這能讓您將指令的引數或選項整合到該鍵值中：

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
#### 鎖定過期時間

預設情況下，隔離鎖定會在指令完成後過期。或者，如果指令被中斷且無法完成，鎖定將在一個小時後過期。不過，您可以透過在指令中定義 `isolationLockExpiresAt` 方法來調整鎖定過期時間：

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
## 定義預期的輸入

當撰寫主控台指令時，通常會透過引數或選項來收集使用者的輸入。Laravel 讓你能夠非常方便地透過指令上的 `signature` 屬性來定義預期從使用者那裡接收的輸入。`signature` 屬性允許你使用單一、具表達力且類似路由的語法來定義指令的名稱、引數與選項。


<a name="arguments"></a>
### 引數

所有使用者提供的引數與選項都包在花括號（大括號）中。在以下範例中，該指令定義了一個必填的引數：`user`：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user}';
```

你也可以將引數設為選填，或為引數定義預設值：

```php
// Optional argument...
'mail:send {user?}'

// Optional argument with default value...
'mail:send {user=foo}'
```


<a name="options"></a>
### 選項

選項與引數一樣，是另一種形式的使用者輸入。當在命令列提供選項時，其前綴為兩個連字號 (`--`)。選項有兩種：接收值的選項與不接收值的選項。不接收值的選項充當布林值 "開關"。讓我們來看看這種類型選項的範例：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue}';
```

在此範例中，可以在呼叫 Artisan 指令時指定 `--queue` 開關。如果傳遞了 `--queue` 開關，該選項的值將為 `true`。否則，其值將為 `false`：

```shell
php artisan mail:send 1 --queue
```


<a name="options-with-values"></a>
#### 具有值的選項

接下來，我們來看看一個預期有值的選項。如果使用者必須為選項指定值，你應該在選項名稱後面加上 `=` 符號：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue=}';
```

在此範例中，使用者可以像這樣為該選項傳遞一個值。如果在呼叫指令時未指定該選項，其值將為 `null`：

```shell
php artisan mail:send 1 --queue=default
```

你可以透過在選項名稱後面指定預設值來為選項分配預設值。如果使用者沒有傳遞選項值，則將使用預設值：

```php
'mail:send {user} {--queue=default}'
```


<a name="option-shortcuts"></a>
#### 選項捷徑

要在定義選項時指定捷徑，你可以在選項名稱之前指定它，並使用 `|` 字元作為分隔符號，將捷徑與完整的選項名稱隔開：

```php
'mail:send {user} {--Q|queue=}'
```

在終端機呼叫指令時，選項捷徑應加上單個連字號前綴，且在指定選項值時不應包含 `=` 字元：

```shell
php artisan mail:send 1 -Qdefault
```


<a name="input-arrays"></a>
### 輸入陣列

如果你想要定義預期有多個輸入值的引數或選項，可以使用 `*` 字元。首先，讓我們看一個指定此類引數的範例：

```php
'mail:send {user*}'
```

執行此指令時，可以將 `user` 引數按順序傳遞給命令列。例如，以下指令將把 `user` 的值設為一個包含 `1` 與 `2` 的陣列：

```shell
php artisan mail:send 1 2
```

此 `*` 字元可以與選填引數定義結合，以允許零個或多個引數實例：

```php
'mail:send {user?*}'
```


<a name="option-arrays"></a>
#### 選項陣列

當定義一個預期有多個輸入值的選項時，傳遞給指令的每個選項值都應該加上選項名稱前綴：

```php
'mail:send {--id=*}'
```

可以透過傳遞多個 `--id` 引數來呼叫此類的指令：

```shell
php artisan mail:send --id=1 --id=2
```


<a name="input-descriptions"></a>
### 輸入描述

你可以透過使用冒號將引數名稱與描述隔開，來為輸入引數和選項指派描述。如果你需要多一點空間來定義你的指令，可以隨意將定義拆分到多行：

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
### 提示缺少輸入

如果你的指令包含必填引數，當使用者未提供時，將會收到錯誤訊息。或者，你可以透過實作 `PromptsForMissingInput` 介面，將指令設定為在缺少必填引數時自動提示使用者：

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

如果 Laravel 需要向使用者收集必填引數，它會自動智慧地使用引數名稱或描述來撰寫問題並詢問使用者。如果你想自訂用於收集必填引數的問題，可以實作 `promptForMissingArgumentsUsing` 方法，並回傳一個以引數名稱為鍵（Key）的問題陣列：

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

你也可以透過使用包含問題與佔位符的元組（Tuple）來提供佔位文字：

```php
return [
    'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
];
```

如果你想要完全控制提示，可以提供一個閉包，該閉包將提示使用者並回傳他們的答案：

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
完整的 [Laravel Prompts](/docs/{{version}}/prompts) 文件包含了關於可用提示及其用法的其他資訊。

如果你希望提示使用者選擇或輸入[選項](#options)，可以在指令的 `handle` 方法中加入提示。然而，如果你只希望在自動提示使用者缺少引數時才提示選項，則可以實作 `afterPromptingForMissingArguments` 方法：

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
### 取得輸入

當您的指令正在執行時，您很可能需要存取該指令所接收的引數與選項的值。為此，您可以使用 `argument` 與 `option` 方法。若該引數或選項不存在，將會返回 `null`：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $userId = $this->argument('user');
}
```

如果您需要將所有的引數以 `array` 形式取得，可以呼叫 `arguments` 方法：

```php
$arguments = $this->arguments();
```

使用 `option` 方法，取得選項就像取得引數一樣簡單。若要將所有選項以陣列形式取得，請呼叫 `options` 方法：

```php
// Retrieve a specific option...
$queueName = $this->option('queue');

// Retrieve all options as an array...
$options = $this->options();
```


<a name="prompting-for-input"></a>
### 提示輸入

> [!NOTE]
> [Laravel Prompts](/docs/{{version}}/prompts) 是一個 PHP 套件，用於為您的命令列應用程式新增精美且易於使用的表單，並具有類似瀏覽器的功能，包括佔位字元文字與驗證。

除了顯示輸出之外，您還可以在執行指令期間要求使用者提供輸入。`ask` 方法會使用給定的問題提示使用者、接收他們的輸入，然後將使用者的輸入返回給您的指令：

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

`ask` 方法也接受選填的第二個引數，用來指定在使用者未提供輸入時應返回的預設值：

```php
$name = $this->ask('What is your name?', 'Taylor');
```

`secret` 方法類似於 `ask`，但使用者在主控台輸入時將無法看到其輸入的內容。此方法在要求輸入密碼等敏感資訊時非常有用：

```php
$password = $this->secret('What is the password?');
```


<a name="asking-for-confirmation"></a>
#### 要求確認

如果您需要向使用者詢問簡單的「是或否」確認，可以使用 `confirm` 方法。預設情況下，此方法將返回 `false`。但是，如果使用者輸入 `y` 或 `yes` 來回應提示，該方法將返回 `true`。

```php
if ($this->confirm('Do you wish to continue?')) {
    // ...
}
```

如有需要，您可以透過將 `true` 作為第二個引數傳遞給 `confirm` 方法，來指定確認提示預設應返回 `true`：

```php
if ($this->confirm('Do you wish to continue?', true)) {
    // ...
}
```


<a name="auto-completion"></a>
#### 自動完成

`anticipate` 方法可用於為可能的選項提供自動完成功能。無論自動完成提示如何，使用者仍可以提供任何回答：

```php
$name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以將 Closure 作為第二個引數傳遞給 `anticipate` 方法。每當使用者輸入字元時，該 Closure 都會被呼叫。該 Closure 應接收一個包含使用者目前輸入內容的字串參數，並返回用於自動完成的選項陣列：

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
#### 多選一問題

如果您需要在提問時提供給使用者一組預定義的選項，可以使用 `choice` 方法。您可以透過將索引值作為第三個引數傳遞給該方法，來設定當未選擇任何選項時要返回的預設值陣列索引：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex
);
```

此外，`choice` 方法還接受選填的第四個與第五個引數，用於決定選擇有效回應的最大嘗試次數，以及是否允許複選：

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

若要將輸出發送到主控台，您可以使用 `line`、`newLine`、`info`、`comment`、`question`、`warn`、`alert` 與 `error` 方法。這些方法中的每一種都會根據其用途使用適當的 ANSI 顏色。例如，讓我們向使用者顯示一些一般資訊。通常，`info` 方法在主控台中會顯示為綠色文字：

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

要顯示錯誤訊息，請使用 `error` 方法。錯誤訊息文字通常會以紅色顯示：

```php
$this->error('Something went wrong!');
```

您可以使用 `line` 方法來顯示一般的無顏色文字：

```php
$this->line('Display this on the screen');
```

您可以使用 `newLine` 方法來顯示空行：

```php
// Write a single blank line...
$this->newLine();

// Write three blank lines...
$this->newLine(3);
```


<a name="tables"></a>
#### 表格

`table` 方法可以輕鬆地正確格式化多行 / 多列的資料。您只需要提供資料表欄位名稱與資料，Laravel 就會自動為您計算表格合適的寬度與高度：

```php
use App\Models\User;

$this->table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```


<a name="progress-bars"></a>
#### 進度條

對於執行時間較長的任務，顯示一個告知使用者任務完成進度的進度條會很有幫助。使用 `withProgressBar` 方法，Laravel 將顯示進度條，並在對給定的可反覆運算值進行每次反覆運算時推進其進度：

```php
use App\Models\User;

$users = $this->withProgressBar(User::all(), function (User $user) {
    $this->performTask($user);
});
```

有時候，您可能需要對進度條的推進方式進行更多手動控制。首先，定義此處理程序將反覆運算的總步驟數。然後，在處理完每個項目後推進進度條：

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
> 關於更多進階選項，請參閱 [Symfony Progress Bar 元件說明文件](https://symfony.com/doc/current/components/console/helpers/progressbar.html)。


<a name="registering-commands"></a>
## 註冊指令

預設情況下，Laravel 會自動註冊 `app/Console/Commands` 目錄中的所有指令。但是，您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `withCommands` 方法，指示 Laravel 掃描其他目錄以尋找 Artisan 指令：

```php
->withCommands([
    __DIR__.'/../app/Domain/Orders/Commands',
])
```

如有必要，您也可以透過將指令的類別名稱提供給 `withCommands` 方法，來手動註冊指令：

```php
use App\Domain\Orders\Commands\SendEmails;

->withCommands([
    SendEmails::class,
])
```

當 Artisan 啟動時，應用程式中的所有指令都將由 [服務容器(service container)](/docs/{{version}}/container) 解析並向 Artisan 註冊。

<a name="programmatically-executing-commands"></a>
## 以程式化方式執行指令

有時你可能會希望在 CLI 之外執行 Artisan 指令。例如，你可能想從路由或控制器中執行 Artisan 指令。你可以使用 `Artisan` Facade 上的 `call` 方法來達成此目的。`call` 方法接受指令的簽名名稱或類別名稱作其第一個引數，並接受一個指令參數陣列作為第二個引數。執行後將會返回結束碼（Exit Code）：

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

或者，你也可以將整個 Artisan 指令作為字串傳遞給 `call` 方法：

```php
Artisan::call('mail:send 1 --queue=default');
```


<a name="passing-array-values"></a>
#### 傳遞陣列值

如果你的指令定義了一個接受陣列的選項，你可以向該選項傳遞一個值陣列：

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

如果你需要指定一個不接受字串值的選項值，例如 `migrate:refresh` 指令上的 `--force` 旗標，你應該傳遞 `true` 或 `false` 作為該選項的值：

```php
$exitCode = Artisan::call('migrate:refresh', [
    '--force' => true,
]);
```


<a name="queueing-artisan-commands"></a>
#### 將 Artisan 指令排入佇列

透過使用 `Artisan` Facade 上的 `queue` 方法，你甚至可以將 Artisan 指令排入佇列，以便由你的 [佇列處理程序(queue workers)](/docs/{{version}}/queues) 在背景處理。在使用此方法之前，請確保你已設定好佇列並正在執行佇列監聽器：

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

使用 `onConnection` 和 `onQueue` 方法，你可以指定 Artisan 指令應該被派遣（Dispatch）到的連線或佇列：

```php
Artisan::queue('mail:send', [
    'user' => 1, '--queue' => 'default'
])->onConnection('redis')->onQueue('commands');
```


<a name="calling-commands-from-other-commands"></a>
### 在指令中呼叫其他指令

有時你可能會希望從現有的 Artisan 指令中呼叫其他指令。你可以使用 `call` 方法來做到這一點。這個 `call` 方法接受指令名稱和一個指令引數與選項的陣列：

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

如果你想呼叫另一個主控台指令並隱藏其所有輸出，可以使用 `callSilently` 方法。`callSilently` 方法與 `call` 方法具有相同的簽名（Signature）：

```php
$this->callSilently('mail:send', [
    'user' => 1, '--queue' => 'default'
]);
```


<a name="signal-handling"></a>
## 訊號處理

如你所知，作業系統允許將訊號傳送給正在執行的行程(Processes)。例如，`SIGTERM` 訊號就是作業系統要求程式優雅地終止的方式。如果你希望在 Artisan 主控台指令中監聽訊號，並在訊號發生時執行程式碼，可以使用 `trap` 方法：

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

若要同時監聽多個訊號，你可以向 `trap` 方法提供一個訊號陣列：

```php
$this->trap([SIGTERM, SIGQUIT], function (int $signal) {
    $this->shouldKeepRunning = false;

    dump($signal); // SIGTERM / SIGQUIT
});
```


<a name="stub-customization"></a>
## Stub 自訂

Artisan 主控台的 `make` 指令用於建立各種專用類別，例如控制器（Controller）、任務（Job）、資料庫遷移（Migration）和測試。這些類別是使用「stub（範本）」檔案生成的，並會根據你的輸入填充對應的值。然而，你可能想對 Artisan 生成的檔案進行微調。若要做到這一點，你可以使用 `stub:publish` 指令將最常見的 stub 檔案發布到你的應用程式中，以便你可以自訂它們：

```shell
php artisan stub:publish
```

發布後的 stub 檔案將位於應用程式根目錄下的 `stubs` 目錄中。你對這些 stub 檔案所做的任何更改，都會在之後使用 Artisan 的 `make` 指令生成其對應類別時反映出來。


<a name="events"></a>
## 事件

Artisan 在執行指令時會發布（Dispatch）三個事件：`Illuminate\Console\Events\ArtisanStarting`、`Illuminate\Console\Events\CommandStarting` 以及 `Illuminate\Console\Events\CommandFinished`。當 Artisan 開始執行時，會立即發布 `ArtisanStarting` 事件。接著，在指令執行之前，會立即發布 `CommandStarting` 事件。最後，在指令執行完畢後，會發布 `CommandFinished` 事件。