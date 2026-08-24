# Artisan 主控台

- [簡介](#introduction)
    - [Tinker (REPL)](#tinker)
- [撰寫指令](#writing-commands)
    - [產生指令](#generating-commands)
    - [指令結構](#command-structure)
    - [閉包指令](#closure-commands)
    - [可隔離執行的指令](#isolatable-commands)
- [定義預期輸入](#defining-input-expectations)
    - [引數](#arguments)
    - [選項](#options)
    - [輸入陣列](#input-arrays)
    - [輸入說明](#input-descriptions)
    - [缺漏輸入時進行提示](#prompting-for-missing-input)
- [指令 I/O](#command-io)
    - [取得輸入](#retrieving-input)
    - [提示使用者輸入](#prompting-for-input)
    - [輸出內容](#writing-output)
- [註冊指令](#registering-commands)
- [以程式碼執行指令](#programmatically-executing-commands)
    - [從其他指令呼叫指令](#calling-commands-from-other-commands)
- [訊號處理](#signal-handling)
- [Dev 指令](#the-dev-command)
    - [自訂 Dev 行程](#customizing-dev-processes)
    - [過濾 Dev 行程](#filtering-dev-processes)
- [自訂 Stub 樣板](#stub-customization)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Artisan 是 Laravel 隨附的命令列介面。Artisan 作為 `artisan` 腳本存在於應用程式的根目錄中，並提供了許多實用的指令，可在您建置應用程式時提供協助。若要檢視所有可用的 Artisan 指令清單，您可以使用 `list` 指令：

```shell
php artisan list
```

每個指令還包含一個「說明」畫面，顯示並說明該指令可用的引數與選項。若要檢視說明畫面，請在指令名稱前加上 `help`：

```shell
php artisan help migrate
```

<a name="laravel-sail"></a>
#### Laravel Sail

如果您使用 [Laravel Sail](/docs/{{version}}/sail) 作為本機開發環境，請記得使用 `sail` 命令列來呼叫 Artisan 指令。Sail 會在您應用程式的 Docker 容器內執行您的 Artisan 指令：

```shell
./vendor/bin/sail artisan list
```

<a name="tinker"></a>
### Tinker (REPL)

[Laravel Tinker](https://github.com/laravel/tinker) 是適用於 Laravel 框架的強大 REPL，由 [PsySH](https://github.com/bobthecow/psysh) 套件所驅動。

<a name="installation"></a>
#### 安裝

所有 Laravel 應用程式預設都包含 Tinker。不過，如果您之前已將其從應用程式中移除，您可以使用 Composer 來安裝 Tinker：

```shell
composer require laravel/tinker
```

> [!NOTE]
> 在與您的 Laravel 應用程式互動時，是否正在尋找熱重載（Hot reloading）、多行程式碼編輯與自動完成功能？快來看看 [Tinkerwell](https://tinkerwell.app)！

<a name="usage"></a>
#### 使用方式

Tinker 允許您在命令列上與整個 Laravel 應用程式進行互動，包含您的 Eloquent 模型、任務、事件等等。若要進入 Tinker 環境，請執行 `tinker` Artisan 指令：

```shell
php artisan tinker
```

您可以使用 `vendor:publish` 指令發布 Tinker 的設定檔：

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> [!WARNING]
> `dispatch` 輔助函式以及 `Dispatchable` 類別上的 `dispatch` 方法依賴垃圾回收機制將任務放入佇列中。因此，使用 Tinker 時，您應該使用 `Bus::dispatch` 或 `Queue::push` 來發送任務。

<a name="command-allow-list"></a>
#### 指令允許清單

Tinker 利用「允許」清單來決定允許在其 Shell 中執行哪些 Artisan 指令。預設情況下，您可以執行 `clear-compiled`、`down`、`env`、`inspire`、`migrate`、`migrate:install`、`up` 和 `optimize` 指令。如果您想要允許更多指令，可以將它們新增至 `tinker.php` 設定檔中的 `commands` 陣列中：

```php
'commands' => [
    // App\Console\Commands\ExampleCommand::class,
],
```

<a name="classes-that-should-not-be-aliased"></a>
#### 不應自動建立別名的類別

通常，當您在 Tinker 中與類別進行互動時，Tinker 會自動為其建立別名。然而，您可能希望某些類別永遠不要建立別名。您可以透過將這些類別列出於 `tinker.php` 設定檔的 `dont_alias` 陣列中來達成此目的：

```php
'dont_alias' => [
    App\Models\User::class,
],
```

<a name="writing-commands"></a>
## 撰寫指令

除了 Artisan 提供的指令之外，您也可以建立自己的自訂指令。指令通常儲存在 `app/Console/Commands` 目錄中；不過，只要您指示 Laravel [掃描其他目錄以尋找 Artisan 指令](#registering-commands)，您也可以自由選擇儲存位置。


<a name="generating-commands"></a>
### 產生指令

若要建立新指令，您可以使用 `make:command` Artisan 指令。此指令將在 `app/Console/Commands` 目錄中建立一個新的指令類別。如果您的應用程式中不存在此目錄，不用擔心——它會在您首次執行 `make:command` Artisan 指令時自動建立：

```shell
php artisan make:command SendEmails
```


<a name="command-structure"></a>
### 指令結構

產生指令後，您應該使用 `Signature` 和 `Description` 屬性來定義指令的簽名與說明。`Signature` 屬性還允許您定義[指令預期的輸入內容](#defining-input-expectations)。當執行指令時，系統會呼叫 `handle` 方法。您可以將指令的邏輯放在這個方法中。

讓我們看一個指令範例。請注意，我們可以透過指令的 `handle` 方法請求所需的任何依賴。Laravel [服務容器](/docs/{{version}}/container)會自動注入該方法簽名中以型態提示 (Type-hinted) 的所有依賴：

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
> 為了提高程式碼的重用性，保持主控台指令簡潔並將任務委派給應用程式服務來完成是一個好習慣。在上方的範例中，請注意我們注入了一個服務類別來處理發送電子郵件這項「重度工作」。


<a name="exit-codes"></a>
#### 結束狀態碼

如果 `handle` 方法沒有回傳任何內容且指令順利執行完成，指令將以退出碼 `0` 結束，表示成功。不過，`handle` 方法也可以選擇回傳一個整數來手動指定指令的結束狀態碼：

```php
$this->error('Something went wrong.');

return 1;
```

如果您想在指令中的任何方法內讓指令「失敗」，可以使用 `fail` 方法。`fail` 方法會立即終止指令的執行並回傳結束狀態碼 `1`：

```php
$this->fail('Something went wrong.');
```


<a name="closure-commands"></a>
### 閉包指令

基於閉包的指令提供了將主控台指令定義為類別之外的另一種選擇。就像路由閉包是控制器的替代方案一樣，您可以將指令閉包視為指令類別的替代方案。

儘管 `routes/console.php` 檔案未定義 HTTP 路由，但它定義了進入您應用程式的主控台進入點（路由）。在此檔案中，您可以使用 `Artisan::command` 方法定義所有基於閉包的主控台指令。`command` 方法接收兩個引數：[指令簽名](#defining-input-expectations)以及一個接收指令引數與選項的閉包：

```php
Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
});
```

該閉包會綁定到底層的指令實例，因此您可以完整存取通常在完整指令類別中能夠存取的所有輔助方法。


<a name="type-hinting-dependencies"></a>
#### 型態提示依賴

除了接收指令的引數和選項之外，指令閉包還可以針對希望從[服務容器](/docs/{{version}}/container)解析出的額外依賴進行型態提示：

```php
use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Support\Facades\Artisan;

Artisan::command('mail:send {user}', function (DripEmailer $drip, string $user) {
    $drip->send(User::find($user));
});
```


<a name="closure-command-descriptions"></a>
#### 閉包指令說明

定義基於閉包的指令時，您可以使用 `purpose` 方法為該指令新增說明。此說明將在您執行 `php artisan list` 或 `php artisan help` 指令時顯示：

```php
Artisan::command('mail:send {user}', function (string $user) {
    // ...
})->purpose('Send a marketing email to a user');
```


<a name="isolatable-commands"></a>
### 可隔離執行的指令

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動器作為應用程式的預設快取驅動器。此外，所有伺服器都必須與同一個中央快取伺服器進行通訊。

有時您可能希望確保一次只能有一個指令實例在執行。為實現此目的，您可以在指令類別上實作 `Illuminate\Contracts\Console\Isolatable` 介面：

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

當您將指令標示為 `Isolatable` 時，Laravel 會自動為該指令提供 `--isolated` 選項，無需在指令的選項中明確定義它。當使用該選項呼叫指令時，Laravel 將確保該指令沒有其他實例正在執行。Laravel 是透過嘗試使用您應用程式的預設快取驅動器取得原子鎖來實現這一點的。如果有該指令的其他實例正在執行，則該指令將不會執行；不過，該指令仍將以成功的結束狀態碼退出：

```shell
php artisan mail:send 1 --isolated
```

如果您想指定當指令無法執行時應回傳的結束狀態碼，可以透過 `isolated` 選項提供所需的狀態碼：

```shell
php artisan mail:send 1 --isolated=12
```


<a name="lock-id"></a>
#### 鎖定 ID

預設情況下，Laravel 會使用指令名稱來產生字串金鑰，該金鑰用於在應用程式快取中取得原子鎖。不過，您可以透過在 Artisan 指令類別上定義 `isolatableId` 方法來自訂此金鑰，從而允許您將指令的引數或選項整合到金鑰中：

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

預設情況下，隔離鎖會在指令執行完畢後過期。或者，如果指令中斷且無法順利完成，鎖將在一小時後過期。不過，您可以透過在指令上定義 `isolationLockExpiresAt` 方法來調整鎖定過期時間：

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
## 定義預期輸入

在撰寫主控台指令時，通常需要透過引數或選項收集來自使用者的輸入。Laravel 讓你能夠利用指令上的 `signature` 屬性，非常便利地定義預期從使用者那裏取得的輸入。`signature` 屬性允許你使用簡潔、富有表達性且類似路由的語法，在單一屬性中定義指令的名稱、引數與選項。


<a name="arguments"></a>
### 引數

所有由使用者提供的引數與選項都會用大括號包覆。在下面的範例中，該指令定義了一個必填的引數：`user`：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user}';
```

你也可以將引數設定為選填，或是為引數定義預設值：

```php
// Optional argument...
'mail:send {user?}'

// Optional argument with default value...
'mail:send {user=foo}'
```


<a name="options"></a>
### 選項

選項與引數一樣，是另一種使用者輸入的形式。當透過命令列提供選項時，前綴會加上兩個連字號 (`--`)。選項有兩種類型：接收數值的選項與不接收數值的選項。不接收數值的選項充當布林「開關」。讓我們來看看這種選項類型的範例：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue}';
```

在這個範例中，呼叫 Artisan 指令時可以指定 `--queue` 開關。若傳入了 `--queue` 開關，該選項的值將為 `true`；否則其值為 `false`：

```shell
php artisan mail:send 1 --queue
```


<a name="options-with-values"></a>
#### 附帶數值的選項

接下來，讓我們看看預期接收數值的選項。若使用者必須為選項指定數值，你應該在選項名稱後面加上 `=` 符號：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue=}';
```

在這個範例中，使用者可以像這樣為選項傳入數值。如果在執行指令時未指定該選項，其值將為 `null`：

```shell
php artisan mail:send 1 --queue=default
```

你可以藉由在選項名稱後指定預設值來為選項指派預設值。若使用者未傳入選項數值，將會使用預設值：

```php
'mail:send {user} {--queue=default}'
```


<a name="option-shortcuts"></a>
#### 選項快捷鍵

若要在定義選項時指派快捷鍵，可以在選項名稱之前指定它，並使用 `|` 字元作為分隔符號來將快捷鍵與完整的選項名稱分開：

```php
'mail:send {user} {--Q|queue=}'
```

在終端機呼叫指令時，選項快捷鍵前應加上單個連字號，且為選項指定數值時不應包含 `=` 字元：

```shell
php artisan mail:send 1 -Qdefault
```


<a name="input-arrays"></a>
### 輸入陣列

如果你想定義預期多個輸入值的引數或選項，可以使用 `*` 字元。首先，讓我們看看指定此類引數的範例：

```php
'mail:send {user*}'
```

執行此指令時，可以在命令列中依序傳入 `user` 引數。例如，以下指令會將 `user` 的值設為以 `1` 和 `2` 為其數值的陣列：

```shell
php artisan mail:send 1 2
```

這個 `*` 字元可以與選填引數定義結合使用，以允許零個或多個引數實例：

```php
'mail:send {user?*}'
```


<a name="option-arrays"></a>
#### 選項陣列

當定義預期多個輸入值的選項時，傳給指令的每個選項值都應該加上選項名稱前綴：

```php
'mail:send {--id=*}'
```

可以透過傳入多個 `--id` 引數來呼叫此類指令：

```shell
php artisan mail:send --id=1 --id=2
```


<a name="input-descriptions"></a>
### 輸入說明

你可以透過使用冒號將引數名稱與說明隔開，來為輸入引數與選項指派說明。如果你需要更多空間來定義你的指令，歡迎將定義拆分到多行中：

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
### 缺漏輸入時進行提示

如果你的指令包含必填引數，使用者在未提供這些引數時會收到錯誤訊息。或者，你可以透過實作 `PromptsForMissingInput` 介面，將指令設定為當缺少必填引數時自動提示使用者：

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

如果 Laravel 需要向使用者收集必填引數，它會利用引數名稱或說明聰明地撰寫問題，自動向使用者詢問該引數。如果你希望自訂用於收集必填引數的問題，你可以實作 `promptForMissingArgumentsUsing` 方法，該方法會傳回一個以引數名稱為鍵 (key) 的問題陣列：

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

你也可以透過使用包含問題與占位符的元組來提供占位符文字：

```php
return [
    'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
];
```

如果你想要完全控制提示，可以提供一個應提示使用者並傳回其回答的閉包：

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
完整的 [Laravel Prompts](/docs/{{version}}/prompts) 文件包含了可用提示及其用法的更多資訊。

如果你希望提示使用者選擇或輸入[選項](#options)，可以在指令的 `handle` 方法中包含提示。但是，如果你只希望在自動提示缺少引數時才提示使用者，則可以實作 `afterPromptingForMissingArguments` 方法：

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

當指令在執行時，您很可能需要存取指令所接收的引數與選項值。若要做到這一點，您可以使用 `argument` 與 `option` 方法。若引數或選項不存在，將會傳回 `null`：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $userId = $this->argument('user');
}
```

若您需要將所有引數作為 `array` 取得，可呼叫 `arguments` 方法：

```php
$arguments = $this->arguments();
```

使用 `option` 方法取得選項就像取得引數一樣簡單。若要將所有選項作為陣列取得，可呼叫 `options` 方法：

```php
// Retrieve a specific option...
$queueName = $this->option('queue');

// Retrieve all options as an array...
$options = $this->options();
```

您可以使用 `input` 方法，將指令的引數與選項作為 `Illuminate\Console\CommandInput` 實例取得，它提供了與 HTTP 請求及其他資料容器上相同的型態化存取器（typed accessors）：

```php
use App\Enums\ReportType;

/**
 * Execute the console command.
 */
public function handle(): void
{
    $input = $this->input()->date('from');

    // ...
}
```

`input` 方法也可以用來從引數或選項中取得單一輸入值：

```php
$queue = $this->input('queue', 'default');
```


<a name="prompting-for-input"></a>
### 提示使用者輸入

> [!NOTE]
> [Laravel Prompts](/docs/{{version}}/prompts) 是一個 PHP 套件，用於為您的命令列應用程式新增美觀且使用者友善的表單，擁有類似瀏覽器的功能，包括預設提示文字（placeholder text）與驗證。

除了顯示輸出之外，您還可以在執行指令期間要求使用者提供輸入。`ask` 方法會以給定的問題提示使用者、接收其輸入，然後將使用者的輸入傳回給您的指令：

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

`ask` 方法也接受第二個可選的引數，用於指定當使用者未提供輸入時應傳回的預設值：

```php
$name = $this->ask('What is your name?', 'Taylor');
```

`secret` 方法與 `ask` 類似，但使用者在主控台中輸入時將無法看到其輸入內容。當詢問密碼等敏感資訊時，此方法非常有幫助：

```php
$password = $this->secret('What is the password?');
```


<a name="asking-for-confirmation"></a>
#### 詢問確認

若您需要詢問使用者簡單的「是或否（yes or no）」確認，可以使用 `confirm` 方法。預設情況下，此方法將傳回 `false`。然而，如果使用者針對提示回答 `y` 或 `yes`，該方法將傳回 `true`。

```php
if ($this->confirm('Do you wish to continue?')) {
    // ...
}
```

如有需要，您可以透過將 `true` 作為第二個引數傳遞給 `confirm` 方法，來指定確認提示預設應傳回 `true`：

```php
if ($this->confirm('Do you wish to continue?', true)) {
    // ...
}
```


<a name="auto-completion"></a>
#### 自動完成

`anticipate` 方法可用於為可能的選項提供自動完成功能。無論自動完成提示為何，使用者仍然可以提供任何答案：

```php
$name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以傳遞一個閉包作為 `anticipate` 方法的第二個引數。每當使用者輸入一個字元時，該閉包就會被呼叫。閉包應接收一個包含使用者目前為止所輸入內容的字串參數，並傳回用於自動完成的選項陣列：

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
#### 多選問題

若您需要在提問時給予使用者一組預定義的選項，可以使用 `choice` 方法。您可以透過將索引作為第三個引數傳遞給該方法，來設定未選擇任何選項時所傳回預設值的陣列索引：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex
);
```

此外，`choice` 方法還接受可選的第四與第五個引數，用於決定選擇有效回應的最大嘗試次數，以及是否允許多選：

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

若要向主控台傳送輸出，您可以使用 `line`、`newLine`、`info`、`comment`、`question`、`warn`、`alert` 和 `error` 方法。這些方法都會根據其用途使用適當的 ANSI 顏色。例如，讓我們向使用者顯示一些一般資訊。通常，`info` 方法在主控台中會顯示為綠色文字：

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

若要顯示錯誤訊息，請使用 `error` 方法。錯誤訊息文字通常會以紅色顯示：

```php
$this->error('Something went wrong!');
```

您可以使用 `line` 方法來顯示沒有顏色的純文字：

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

`table` 方法可以輕鬆地格式化多行 / 多欄的資料。您只需要提供欄位名稱和表格資料，Laravel 就會自動為您計算出合適的表格寬度與高度：

```php
use App\Models\User;

$this->table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```


<a name="progress-bars"></a>
#### 進度條

對於執行時間較長的任務，顯示一個告知使用者任務完成進度的進度條會很有幫助。使用 `withProgressBar` 方法，Laravel 會顯示進度條，並在對給定的可疊代（iterable）值進行每次疊代時推進進度：

```php
use App\Models\User;

$users = $this->withProgressBar(User::all(), function (User $user) {
    $this->performTask($user);
});
```

有時候，您可能需要對進度條如何推進進行更多的手動控制。首先，定義流程將疊代的總步驟數。然後，在處理每個項目後推進進度條：

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
> 若要瞭解更多進階選項，請參考 [Symfony Progress Bar 元件文件](https://symfony.com/doc/current/components/console/helpers/progressbar.html)。


<a name="registering-commands"></a>
## 註冊指令

預設情況下，Laravel 會自動註冊 `app/Console/Commands` 目錄下的所有指令。不過，您可以透過在應用程式的 `bootstrap/app.php` 檔案中使用 `withCommands` 方法，指示 Laravel 掃描其他目錄以尋找 Artisan 指令：

```php
->withCommands([
    __DIR__.'/../app/Domain/Orders/Commands',
])
```

如有需要，您也可以透過將指令的類別名稱提供給 `withCommands` 方法來手動註冊指令：

```php
use App\Domain\Orders\Commands\SendEmails;

->withCommands([
    SendEmails::class,
])
```

當 Artisan 啟動時，您應用程式中的所有指令都將由 [服務容器](/docs/{{version}}/container) 解析並註冊到 Artisan 中。

<a name="programmatically-executing-commands"></a>
## 以程式碼執行指令

有時候您可能希望在 CLI 之外執行 Artisan 指令。例如，您可能想從路由或控制器中執行 Artisan 指令。您可以透過 `Artisan` Facade 的 `call` 方法來達成。`call` 方法的第一個引數接受指令的簽名名稱或類別名稱，第二個引數則接受一個指令參數陣列。該方法會回傳結束碼 (Exit Code)：

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

或者，您也可以將整條 Artisan 指令作為字串傳遞給 `call` 方法：

```php
Artisan::call('mail:send 1 --queue=default');
```

<a name="passing-array-values"></a>
#### 傳遞陣列數值

如果您的指令定義了一個接受陣列的選項，您可以將一個包含多個值的陣列傳遞給該選項：

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

若您需要指定一個不接受字串值的選項（例如 `migrate:refresh` 指令上的 `--force` 旗標），您應該傳遞 `true` 或 `false` 作為該選項的值：

```php
$exitCode = Artisan::call('migrate:refresh', [
    '--force' => true,
]);
```

<a name="queueing-artisan-commands"></a>
#### 將 Artisan 指令推入佇列

使用 `Artisan` Facade 上的 `queue` 方法，您甚至可以將 Artisan 指令排入佇列，以便由 [佇列工作者](/docs/{{version}}/queues) 在背景處理。在使用此方法之前，請確保您已設定好佇列並正在執行佇列監聽器：

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

使用 `onConnection` 與 `onQueue` 方法，您可以指定 Artisan 指令應該派遣至哪一個連線或佇列：

```php
Artisan::queue('mail:send', [
    'user' => 1, '--queue' => 'default'
])->onConnection('redis')->onQueue('commands');
```

<a name="calling-commands-from-other-commands"></a>
### 從其他指令呼叫指令

有時候您可能想從現有的 Artisan 指令中呼叫其他指令。您可以使用 `call` 方法來做到這一點。這個 `call` 方法接受指令名稱以及一個包含指令引數 / 選項的陣列：

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

如果您想呼叫另一個主控台指令並隱藏其所有輸出，您可以使用 `callSilently` 方法。`callSilently` 方法與 `call` 方法有著相同的簽名：

```php
$this->callSilently('mail:send', [
    'user' => 1, '--queue' => 'default'
]);
```

<a name="signal-handling"></a>
## 訊號處理

如您所知，作業系統允許將訊號發送給正在執行的行程(Processes)。例如，作業系統會透過 `SIGTERM` 訊號要求程式平順地終止。如果您希望在 Artisan 主控台指令中監聽訊號，並在收到訊號時執行程式碼，您可以使用 `trap` 方法：

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

若要一次監聽多個訊號，您可以傳遞一個訊號陣列給 `trap` 方法：

```php
$this->trap([SIGTERM, SIGQUIT], function (int $signal) {
    $this->shouldKeepRunning = false;

    dump($signal); // SIGTERM / SIGQUIT
});
```

<a name="the-dev-command"></a>
## Dev 指令

`dev` Artisan 指令會在單一終端機視窗中啟動本機開發所需的所有行程(Processes)。預設情況下，它會同時執行 PHP 開發伺服器、佇列 Worker、透過 [Pail](/docs/{{version}}/logging#tailing-log-messages-using-pail) 追蹤日誌，以及 Vite 靜態資源編譯：

```shell
php artisan dev
```

在底層，`dev` 指令使用 `@laravel/multiplex` npm 套件來管理這些行程，為每個行程提供專屬的分頁，且輸出內容可進行搜尋與捲動。每個行程都有標籤與顏色標示，方便您輕鬆辨識。如果某個行程崩潰，系統將自動將其重啟，而當您結束程式時，所有輸出內容都會寫回您的終端機，因此不會遺失任何內容。

> [!NOTE]
> `dev` 指令需要 Node 22.13 或更高版本。在 Windows 上，它會退回使用 `concurrently` npm 套件，且無法使用分頁介面。

預設的行程為：

| 名稱 | 指令 |
| --- | --- |
| `server` | `php artisan serve --host=localhost` |
| `queue` | `php artisan queue:listen --tries=1 --timeout=0` |
| `logs` | `php artisan pail --timeout=0` |
| `vite` | `npm run dev` |

> [!NOTE]
> `vite` 行程會自動偵測您的 Node 套件管理器（npm、pnpm、Yarn 或 Bun）並使用適當的執行指令。


<a name="customizing-dev-processes"></a>
### 自訂 Dev 行程

您可以使用 `DevCommands` 類別來自訂 `dev` 指令所執行的行程，通常是在應用程式 `AppServiceProvider` 的 `boot` 方法中設定。`register` 方法接收一個指令字串和可選的名稱：

```php
use Illuminate\Foundation\DevCommands;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    DevCommands::register('some-command --flag', 'my-process');
}
```

在註冊 Artisan 指令時，您可以使用 `artisan` 方法，該方法會自動在指令前方加上 `php artisan` 前綴：

```php
DevCommands::artisan('horizon', 'horizon');
```

同樣地，`node` 方法會在指令前加上自動偵測到的套件管理器執行指令（例如 `npm run`），而 `nodeExec` 方法則會在指令前加上套件管理器的 exec 指令（例如 `npx`）：

```php
DevCommands::node('storybook', 'storybook');

DevCommands::nodeExec('tailwindcss -i resources/css/app.css -o public/css/app.css --watch', 'tailwind');
```

如果您註冊了與預設行程同名的行程，您的行程將會替換預設行程。例如，您可以自訂 server 行程以使用不同的連接埠：

```php
DevCommands::artisan('serve --host=localhost --port=9000', 'server');
```

您還可以自訂終端機中行程標籤的顏色。可用的顏色方法有 `blue`、`purple`、`pink`、`orange`、`green` 和 `yellow`。您也可以傳入自訂的十六進位顏色碼給 `color` 方法：

```php
DevCommands::register('my-command', 'my-process')->green();

DevCommands::register('my-command', 'my-process')->color('#ff6347');
```

若要查看所有已註冊的 dev 行程而不啟動它們，請使用 `dev:list` 指令：

```shell
php artisan dev:list
```


<a name="restarting-failed-processes"></a>
#### 重啟失敗的行程

如果某個行程崩潰，Laravel 將在短暫延遲後重新啟動它，最多重試五次，之後將其標記為失敗。如果在啟動後一秒內就停止運作的行程將不會被重啟，因為它很可能壓根就沒有成功啟動。透過按下 `r` 手動重啟行程會重置該計數器。

您可以使用 `--no-restart` 選項在單次執行時停用此行為：

```shell
php artisan dev --no-restart
```

或者，您可以使用 `disableAutoRestart` 方法為整個應用程式停用此行為：

```php
DevCommands::disableAutoRestart();
```


<a name="filtering-dev-processes"></a>
### 過濾 Dev 行程

您可以使用 `only` 方法指示 `dev` 指令在呼叫時僅執行特定的行程。同樣地，您可以使用 `except` 方法排除特定的行程：

```php
// Only run the server and vite processes...
DevCommands::only('server', 'vite');

// Run all processes except the queue worker...
DevCommands::except('queue');
```


<a name="stub-customization"></a>
## 自訂 Stub 樣板

Artisan 主控台的 `make` 指令用於建立各種類別，例如 Controller、Job、Migration 和 Test。這些類別是使用「stub」檔案產生的，並根據您的輸入填入相應的值。然而，您可能想對 Artisan 產生的檔案進行微調。為此，您可以使用 `stub:publish` 指令將最常見的 Stub 樣板發布到應用程式中，以便進行自訂：

```shell
php artisan stub:publish
```

發布的 Stub 樣板將位於應用程式根目錄下的 `stubs` 目錄中。您對這些 Stub 樣板進行的任何修改，都會在您使用 Artisan 的 `make` 指令產生對應類別時反映出來。


<a name="events"></a>
## 事件

Artisan 在執行指令時會觸發三個事件：`Illuminate\Console\Events\ArtisanStarting`、`Illuminate\Console\Events\CommandStarting` 和 `Illuminate\Console\Events\CommandFinished`。當 Artisan 開始執行時，會立即觸發 `ArtisanStarting` 事件。接著，在指令執行前會立即觸發 `CommandStarting` 事件。最後，當指令執行完畢時，會觸發 `CommandFinished` 事件。