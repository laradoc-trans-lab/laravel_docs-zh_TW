# Artisan 命令列

- [簡介](#introduction)
    - [Tinker (REPL)](#tinker)
- [撰寫命令](#writing-commands)
    - [產生命令](#generating-commands)
    - [命令結構](#command-structure)
    - [閉包命令](#closure-commands)
    - [可隔離命令](#isolatable-commands)
- [定義輸入預期](#defining-input-expectations)
    - [引數](#arguments)
    - [選項](#options)
    - [輸入陣列](#input-arrays)
    - [輸入說明](#input-descriptions)
    - [提示遺失的輸入](#prompting-for-missing-input)
- [命令 I/O](#command-io)
    - [擷取輸入](#retrieving-input)
    - [提示輸入](#prompting-for-input)
    - [撰寫輸出](#writing-output)
- [註冊命令](#registering-commands)
- [以程式化方式執行命令](#programmatically-executing-commands)
    - [從其他命令呼叫命令](#calling-commands-from-other-commands)
- [訊號處理](#signal-handling)
- [Stub 自訂](#stub-customization)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Artisan 是 Laravel 內建的命令列介面。Artisan 以 `artisan` 腳本的形式存在於應用程式的根目錄中，並提供了許多有用的命令，可以在您建構應用程式時提供協助。若要查看所有可用的 Artisan 命令列表，您可以使用 `list` 命令：

```shell
php artisan list
```

每個命令也包含一個「說明」畫面，其中顯示並描述了該命令可用的引數和選項。若要查看說明畫面，請在命令名稱前加上 `help`：

```shell
php artisan help migrate
```

<a name="laravel-sail"></a>
#### Laravel Sail

如果您使用 [Laravel Sail](/docs/{{version}}/sail) 作為您的本機開發環境，請記得使用 `sail` 命令列來呼叫 Artisan 命令。Sail 將在您應用程式的 Docker 容器中執行您的 Artisan 命令：

```shell
./vendor/bin/sail artisan list
```

<a name="tinker"></a>
### Tinker (REPL)

[Laravel Tinker](https://github.com/laravel/tinker) 是一個強大的 Laravel 框架 REPL (Read-Eval-Print Loop)，由 [PsySH](https://github.com/bobthecow/psysh) 套件提供支援。

<a name="installation"></a>
#### 安裝

所有 Laravel 應用程式預設都包含 Tinker。但是，如果您之前已將 Tinker 從應用程式中移除，則可以使用 Composer 安裝它：

```shell
composer require laravel/tinker
```

> [!NOTE]
> 正在尋找與 Laravel 應用程式互動時的熱重載、多行程式碼編輯和自動完成功能嗎？請查看 [Tinkerwell](https://tinkerwell.app)！

<a name="usage"></a>
#### 用法

Tinker 允許您在命令列上與整個 Laravel 應用程式互動，包括您的 Eloquent 模型、任務 (jobs)、事件 (events) 等。若要進入 Tinker 環境，請執行 `tinker` Artisan 命令：

```shell
php artisan tinker
```

您可以使用 `vendor:publish` 命令發布 Tinker 的設定檔：

```shell
php artisan vendor:publish --provider="Laravel\Tinker\TinkerServiceProvider"
```

> [!WARNING]
> `dispatch` 輔助函式和 `Dispatchable` 類別上的 `dispatch` 方法依賴於垃圾回收來將任務放入佇列。因此，當使用 Tinker 時，您應該使用 `Bus::dispatch` 或 `Queue::push` 來分派任務。

<a name="command-allow-list"></a>
#### 命令允許列表

Tinker 利用「允許」列表來決定哪些 Artisan 命令可以在其 Shell 中執行。預設情況下，您可以執行 `clear-compiled`、`down`、`env`、`inspire`、`migrate`、`migrate:install`、`up` 和 `optimize` 命令。如果您想允許更多命令，可以將它們新增到 `tinker.php` 設定檔中的 `commands` 陣列：

```php
'commands' => [
    // App\Console\Commands\ExampleCommand::class,
],
```

<a name="classes-that-should-not-be-aliased"></a>
#### 不應別名的類別

通常，Tinker 會在您與類別互動時自動為其建立別名。但是，您可能希望永遠不要為某些類別建立別名。您可以透過在 `tinker.php` 設定檔的 `dont_alias` 陣列中列出這些類別來實現此目的：

```php
'dont_alias' => [
    App\Models\User::class,
],
```

<a name="writing-commands"></a>
## 撰寫命令

除了 Artisan 提供的命令之外，您還可以建構自己的自訂命令。命令通常儲存在 `app/Console/Commands` 目錄中；但是，您可以自由選擇自己的儲存位置，只要您指示 Laravel [掃描其他目錄以尋找 Artisan 命令](#registering-commands)即可。

<a name="generating-commands"></a>
### 產生命令

若要建立新命令，您可以使用 `make:command` Artisan 命令。此命令將在 `app/Console/Commands` 目錄中建立一個新的命令類別。如果此目錄在您的應用程式中不存在，請不用擔心 — 它將在您第一次執行 `make:command` Artisan 命令時建立：

```shell
php artisan make:command SendEmails
```

<a name="command-structure"></a>
### 命令結構

產生命令後，您應該為類別的 `signature` 和 `description` 屬性定義適當的值。這些屬性將在 `list` 畫面顯示您的命令時使用。`signature` 屬性還允許您定義 [命令的輸入預期](#defining-input-expectations)。`handle` 方法將在執行您的命令時被呼叫。您可以將命令邏輯放在此方法中。

讓我們看一個命令範例。請注意，我們可以透過命令的 `handle` 方法請求任何我們需要的依賴。Laravel [服務容器](/docs/{{version}}/container) 將自動注入此方法簽章中型別提示的所有依賴：

```php
<?php

namespace App\Console\Commands;

use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Console\Command;

class SendEmails extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'mail:send {user}';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Send a marketing email to a user';

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
> 為了提高程式碼重用性，建議您的主控台命令保持輕量，並讓它們委託應用程式服務來完成其任務。在上面的範例中，請注意我們注入了一個服務類別來執行發送電子郵件的「繁重工作」。

<a name="exit-codes"></a>
#### 結束代碼

如果 `handle` 方法沒有回傳任何值且命令成功執行，則命令將以 `0` 的結束代碼退出，表示成功。但是，`handle` 方法可以選擇回傳一個整數來手動指定命令的結束代碼：

```php
$this->error('Something went wrong.');

return 1;
```

如果您想從命令中的任何方法「失敗」命令，您可以使用 `fail` 方法。`fail` 方法將立即終止命令的執行並回傳結束代碼 `1`：

```php
$this->fail('Something went wrong.');
```

<a name="closure-commands"></a>
### 閉包命令

基於閉包的命令提供了一種替代方案，可以將主控台命令定義為類別。就像路由閉包是控制器的一種替代方案一樣，您可以將命令閉包視為命令類別的替代方案。

儘管 `routes/console.php` 檔案沒有定義 HTTP 路由，但它定義了基於主控台的應用程式進入點 (路由)。在此檔案中，您可以使用 `Artisan::command` 方法定義所有基於閉包的主控台命令。`command` 方法接受兩個引數：[命令簽章](#defining-input-expectations) 和一個接收命令引數和選項的閉包：

```php
Artisan::command('mail:send {user}', function (string $user) {
    $this->info("Sending email to: {$user}!");
});
```

閉包綁定到底層命令實例，因此您可以完全存取通常可以在完整命令類別上存取的所有輔助方法。

<a name="type-hinting-dependencies"></a>
#### 型別提示依賴

除了接收命令的引數和選項之外，命令閉包還可以型別提示您希望從 [服務容器](/docs/{{version}}/container) 中解析的其他依賴：

```php
use App\Models\User;
use App\Support\DripEmailer;
use Illuminate\Support\Facades\Artisan;

Artisan::command('mail:send {user}', function (DripEmailer $drip, string $user) {
    $drip->send(User::find($user));
});
```

<a name="closure-command-descriptions"></a>
#### 閉包命令說明

定義基於閉包的命令時，您可以使用 `purpose` 方法為命令新增說明。當您執行 `php artisan list` 或 `php artisan help` 命令時，將顯示此說明：

```php
Artisan::command('mail:send {user}', function (string $user) {
    // ...
})->purpose('Send a marketing email to a user');
```

<a name="isolatable-commands"></a>
### 可隔離命令

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與相同的中央快取伺服器通訊。

有時您可能希望確保一次只能執行一個命令實例。為此，您可以在命令類別上實作 `Illuminate\Contracts\Console\Isolatable` 介面：

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

當您將命令標記為 `Isolatable` 時，Laravel 會自動為該命令提供 `--isolated` 選項，而無需在命令選項中明確定義它。當使用該選項呼叫命令時，Laravel 將確保沒有其他該命令的實例正在執行。Laravel 透過嘗試使用應用程式的預設快取驅動程式取得原子鎖來實現此目的。如果該命令的其他實例正在執行，則該命令將不會執行；但是，該命令仍將以成功的結束狀態碼退出：

```shell
php artisan mail:send 1 --isolated
```

如果您想指定命令在無法執行時應回傳的結束狀態碼，您可以透過 `isolated` 選項提供所需的狀態碼：

```shell
php artisan mail:send 1 --isolated=12
```

<a name="lock-id"></a>
#### 鎖定 ID

預設情況下，Laravel 將使用命令的名稱來產生用於在應用程式快取中取得原子鎖的字串鍵。但是，您可以透過在 Artisan 命令類別上定義 `isolatableId` 方法來自訂此鍵，從而允許您將命令的引數或選項整合到鍵中：

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

預設情況下，隔離鎖在命令完成後過期。或者，如果命令被中斷且無法完成，則鎖將在一小時後過期。但是，您可以透過在命令上定義 `isolationLockExpiresAt` 方法來調整鎖定過期時間：

```php
use DateTimeInterface;
use DateInterval;

/**
 * Determine when an isolation lock expires for the command.
 */
public function isolationLockExpiresAt(): DateTimeInterface|DateInterval
{
    return now()->addMinutes(5);
}
```

<a name="defining-input-expectations"></a>
## 定義輸入預期

撰寫主控台命令時，通常會透過引數或選項從使用者那裡收集輸入。Laravel 讓您可以使用命令上的 `signature` 屬性非常方便地定義您期望從使用者那裡獲得的輸入。`signature` 屬性允許您以單一、富有表現力、類似路由的語法定義命令的名稱、引數和選項。

<a name="arguments"></a>
### 引數

所有使用者提供的引數和選項都用大括號括起來。在以下範例中，命令定義了一個必需的引數：`user`：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user}';
```

您也可以將引數設為可選或為引數定義預設值：

```php
// 可選引數...
'mail:send {user?}'

// 帶有預設值的可選引數...
'mail:send {user=foo}'
```

<a name="options"></a>
### 選項

選項，就像引數一樣，是使用者輸入的另一種形式。選項在透過命令列提供時，會以兩個連字號 (`--`) 作為前綴。選項有兩種型別：一種是接收值的，另一種是不接收值的。不接收值的選項充當布林「開關」。讓我們看一個這種型別選項的範例：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue}';
```

在此範例中，可以在呼叫 Artisan 命令時指定 `--queue` 開關。如果傳遞了 `--queue` 開關，則選項的值將為 `true`。否則，值將為 `false`：

```shell
php artisan mail:send 1 --queue
```

<a name="options-with-values"></a>
#### 帶有值的選項

接下來，讓我們看一個期望值的選項。如果使用者必須為選項指定一個值，您應該在選項名稱後加上 `=` 符號：

```php
/**
 * The name and signature of the console command.
 *
 * @var string
 */
protected $signature = 'mail:send {user} {--queue=}';
```

在此範例中，使用者可以像這樣為選項傳遞一個值。如果在呼叫命令時未指定該選項，則其值將為 `null`：

```shell
php artisan mail:send 1 --queue=default
```

您可以透過在選項名稱後指定預設值來為選項分配預設值。如果使用者沒有傳遞任何選項值，則將使用預設值：

```php
'mail:send {user} {--queue=default}'
```

<a name="option-shortcuts"></a>
#### 選項捷徑

若要在定義選項時指定捷徑，您可以在選項名稱之前指定它，並使用 `|` 字元作為分隔符號將捷徑與完整選項名稱分開：

```php
'mail:send {user} {--Q|queue}'
```

在終端機上呼叫命令時，選項捷徑應以單個連字號作為前綴，並且在為選項指定值時不應包含 `=` 字元：

```shell
php artisan mail:send 1 -Qdefault
```

<a name="input-arrays"></a>
### 輸入陣列

如果您想定義引數或選項以期望多個輸入值，您可以使用 `*` 字元。首先，讓我們看一個指定此類引數的範例：

```php
'mail:send {user*}'
```

執行此命令時，`user` 引數可以依序傳遞給命令列。例如，以下命令將 `user` 的值設定為一個包含 `1` 和 `2` 作為其值的陣列：

```shell
php artisan mail:send 1 2
```

此 `*` 字元可以與可選引數定義結合使用，以允許零個或多個引數實例：

```php
'mail:send {user?*}'
```

<a name="option-arrays"></a>
#### 選項陣列

定義期望多個輸入值的選項時，傳遞給命令的每個選項值都應以選項名稱作為前綴：

```php
'mail:send {--id=*}'
```

此類命令可以透過傳遞多個 `--id` 引數來呼叫：

```shell
php artisan mail:send --id=1 --id=2
```

<a name="input-descriptions"></a>
### 輸入說明

您可以透過使用冒號將引數名稱與說明分開來為輸入引數和選項分配說明。如果您需要額外的空間來定義命令，請隨意將定義分散到多行：

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
### 提示遺失的輸入

如果您的命令包含必需的引數，則在未提供這些引數時，使用者將收到錯誤訊息。或者，您可以透過實作 `PromptsForMissingInput` 介面來設定您的命令，以便在缺少必需引數時自動提示使用者：

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

如果 Laravel 需要從使用者那裡收集必需的引數，它將透過智慧地使用引數名稱或說明來措辭問題，自動向使用者詢問該引數。如果您希望自訂用於收集必需引數的問題，您可以實作 `promptForMissingArgumentsUsing` 方法，回傳一個以引數名稱為鍵的問題陣列：

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

您也可以透過使用包含問題和佔位符的元組來提供佔位符文字：

```php
return [
    'user' => ['Which user ID should receive the mail?', 'E.g. 123'],
];
```

如果您想完全控制提示，您可以提供一個閉包，該閉包應該提示使用者並回傳他們的答案：

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
> 全面的 [Laravel Prompts](/docs/{{version}}/prompts) 說明文件包含有關可用提示及其用法的更多資訊。

如果您希望提示使用者選擇或輸入 [選項](#options)，您可以在命令的 `handle` 方法中包含提示。但是，如果您只希望在使用者也被自動提示遺失引數時才提示使用者，那麼您可以實作 `afterPromptingForMissingArguments` 方法：

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
## 命令 I/O

<a name="retrieving-input"></a>
### 擷取輸入

在命令執行期間，您可能需要存取命令接受的引數和選項的值。為此，您可以使用 `argument` 和 `option` 方法。如果引數或選項不存在，將回傳 `null`：

```php
/**
 * Execute the console command.
 */
public function handle(): void
{
    $userId = $this->argument('user');
}
```

如果您需要將所有引數擷取為 `array`，請呼叫 `arguments` 方法：

```php
$arguments = $this->arguments();
```

選項可以像引數一樣輕鬆地使用 `option` 方法擷取。若要將所有選項擷取為陣列，請呼叫 `options` 方法：

```php
// 擷取特定選項...
$queueName = $this->option('queue');

// 將所有選項擷取為陣列...
$options = $this->options();
```

<a name="prompting-for-input"></a>
### 提示輸入

> [!NOTE]
> [Laravel Prompts](/docs/{{version}}/prompts) 是一個 PHP 套件，用於為您的命令列應用程式新增美觀且使用者友善的表單，具有類似瀏覽器的功能，包括佔位符文字和驗證。

除了顯示輸出之外，您還可以在命令執行期間要求使用者提供輸入。`ask` 方法將向使用者提示給定的問題，接受他們的輸入，然後將使用者的輸入回傳給您的命令：

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

`ask` 方法還接受一個可選的第二個引數，該引數指定如果未提供使用者輸入，則應回傳的預設值：

```php
$name = $this->ask('What is your name?', 'Taylor');
```

`secret` 方法類似於 `ask`，但使用者在主控台中輸入時，他們的輸入將不可見。此方法在詢問敏感資訊（例如密碼）時很有用：

```php
$password = $this->secret('What is the password?');
```

<a name="asking-for-confirmation"></a>
#### 詢問確認

如果您需要向使用者詢問簡單的「是或否」確認，您可以使用 `confirm` 方法。預設情況下，此方法將回傳 `false`。但是，如果使用者在提示中輸入 `y` 或 `yes`，則該方法將回傳 `true`。

```php
if ($this->confirm('Do you wish to continue?')) {
    // ...
}
```

如有必要，您可以透過將 `true` 作為第二個引數傳遞給 `confirm` 方法來指定確認提示應預設回傳 `true`：

```php
if ($this->confirm('Do you wish to continue?', true)) {
    // ...
}
```

<a name="auto-completion"></a>
#### 自動完成

`anticipate` 方法可用於為可能的選擇提供自動完成功能。使用者仍然可以提供任何答案，無論自動完成提示如何：

```php
$name = $this->anticipate('What is your name?', ['Taylor', 'Dayle']);
```

或者，您可以將閉包作為第二個引數傳遞給 `anticipate` 方法。每次使用者輸入一個字元時，都會呼叫該閉包。該閉包應接受一個包含使用者目前輸入的字串參數，並回傳一個用於自動完成的選項陣列：

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

如果您需要在提問時為使用者提供一組預定義的選擇，您可以使用 `choice` 方法。您可以透過將索引作為第三個引數傳遞給該方法來設定如果未選擇任何選項則應回傳的預設值的陣列索引：

```php
$name = $this->choice(
    'What is your name?',
    ['Taylor', 'Dayle'],
    $defaultIndex
);
```

此外，`choice` 方法接受可選的第四個和第五個引數，用於確定選擇有效回應的最大嘗試次數以及是否允許多重選擇：

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
### 撰寫輸出

若要將輸出傳送到主控台，您可以使用 `line`、`newLine`、`info`、`comment`、`question`、`warn`、`alert` 和 `error` 方法。這些方法中的每一個都將使用適當的 ANSI 顏色來達到其目的。例如，讓我們向使用者顯示一些一般資訊。通常，`info` 方法將在主控台中顯示為綠色文字：

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

您可以使用 `line` 方法顯示純色、無色的文字：

```php
$this->line('Display this on the screen');
```

您可以使用 `newLine` 方法顯示空白行：

```php
// 寫入一個空白行...
$this->newLine();

// 寫入三個空白行...
$this->newLine(3);
```

<a name="tables"></a>
#### 表格

`table` 方法可以輕鬆地正確格式化多行/多列資料。您只需提供欄位名稱和表格資料，Laravel 將自動為您計算表格的適當寬度和高度：

```php
use App\Models\User;

$this->table(
    ['Name', 'Email'],
    User::all(['name', 'email'])->toArray()
);
```

<a name="progress-bars"></a>
#### 進度條

對於長時間執行的任務，顯示進度條以告知使用者任務完成度會很有幫助。使用 `withProgressBar` 方法，Laravel 將顯示一個進度條，並為給定可迭代值的每次迭代推進其進度：

```php
use App\Models\User;

$users = $this->withProgressBar(User::all(), function (User $user) {
    $this->performTask($user);
});
```

有時，您可能需要更手動地控制進度條的推進方式。首先，定義過程將迭代的總步驟數。然後，在處理每個項目後推進進度條：

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
> 有關更多進階選項，請查看 [Symfony 進度條元件說明文件](https://symfony.com/doc/current/components/console/helpers/progressbar.html)。

<a name="registering-commands"></a>
## 註冊命令

預設情況下，Laravel 會自動註冊 `app/Console/Commands` 目錄中的所有命令。但是，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `withCommands` 方法指示 Laravel 掃描其他目錄以尋找 Artisan 命令：

```php
->withCommands([
    __DIR__.'/../app/Domain/Orders/Commands',
])
```

如有必要，您也可以透過向 `withCommands` 方法提供命令的類別名稱來手動註冊命令：

```php
use App\Domain\Orders\Commands\SendEmails;

->withCommands([
    SendEmails::class,
])
```

當 Artisan 啟動時，應用程式中的所有命令都將由 [服務容器](/docs/{{version}}/container) 解析並註冊到 Artisan。

<a name="programmatically-executing-commands"></a>
## 以程式化方式執行命令

有時您可能希望在 CLI 之外執行 Artisan 命令。例如，您可能希望從路由或控制器執行 Artisan 命令。您可以使用 `Artisan` Facade 上的 `call` 方法來實現此目的。`call` 方法接受命令的簽章名稱或類別名稱作為其第一個引數，以及命令參數陣列作為第二個引數。將回傳結束代碼：

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

或者，您可以將整個 Artisan 命令作為字串傳遞給 `call` 方法：

```php
Artisan::call('mail:send 1 --queue=default');
```

<a name="passing-array-values"></a>
#### 傳遞陣列值

如果您的命令定義了一個接受陣列的選項，您可以將一個值陣列傳遞給該選項：

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

如果您需要指定不接受字串值的選項的值，例如 `migrate:refresh` 命令上的 `--force` 旗標，您應該將 `true` 或 `false` 作為選項的值傳遞：

```php
$exitCode = Artisan::call('migrate:refresh', [
    '--force' => true,
]);
```

<a name="queueing-artisan-commands"></a>
#### 將 Artisan 命令排入佇列

使用 `Artisan` Facade 上的 `queue` 方法，您甚至可以將 Artisan 命令排入佇列，以便它們由您的 [佇列工作者](/docs/{{version}}/queues) 在背景處理。在使用此方法之前，請確保您已設定佇列並正在執行佇列監聽器：

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

使用 `onConnection` 和 `onQueue` 方法，您可以指定 Artisan 命令應分派到的連線或佇列：

```php
Artisan::queue('mail:send', [
    'user' => 1, '--queue' => 'default'
])->onConnection('redis')->onQueue('commands');
```

<a name="calling-commands-from-other-commands"></a>
### 從其他命令呼叫命令

有時您可能希望從現有的 Artisan 命令呼叫其他命令。您可以使用 `call` 方法來實現此目的。此 `call` 方法接受命令名稱和命令引數/選項陣列：

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

如果您想呼叫另一個主控台命令並抑制其所有輸出，您可以使用 `callSilently` 方法。`callSilently` 方法與 `call` 方法具有相同的簽章：

```php
$this->callSilently('mail:send', [
    'user' => 1, '--queue' => 'default'
]);
```

<a name="signal-handling"></a>
## 訊號處理

如您所知，作業系統允許向正在執行的程序發送訊號。例如，`SIGTERM` 訊號是作業系統要求程序終止的方式。如果您希望在 Artisan 主控台命令中監聽訊號並在它們發生時執行程式碼，您可以使用 `trap` 方法：

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

若要同時監聽多個訊號，您可以向 `trap` 方法提供一個訊號陣列：

```php
$this->trap([SIGTERM, SIGQUIT], function (int $signal) {
    $this->shouldKeepRunning = false;

    dump($signal); // SIGTERM / SIGQUIT
});
```

<a name="stub-customization"></a>
## Stub 自訂

Artisan 主控台的 `make` 命令用於建立各種類別，例如控制器、任務、遷移和測試。這些類別是使用「stub」檔案產生的，這些檔案根據您的輸入填充值。但是，您可能希望對 Artisan 產生的檔案進行小幅更改。為此，您可以使用 `stub:publish` 命令將最常見的 stub 發布到您的應用程式，以便您可以自訂它們：

```shell
php artisan stub:publish
```

發布的 stub 將位於應用程式根目錄中的 `stubs` 目錄中。您對這些 stub 所做的任何更改都將在您使用 Artisan 的 `make` 命令產生其對應類別時反映出來。

<a name="events"></a>
## 事件

Artisan 在執行命令時會分派三個事件：`Illuminate\Console\Events\ArtisanStarting`、`Illuminate\Console\Events\CommandStarting` 和 `Illuminate\Console\Events\CommandFinished`。`ArtisanStarting` 事件在 Artisan 開始執行時立即分派。接下來，`CommandStarting` 事件在命令執行之前立即分派。最後，`CommandFinished` 事件在命令執行完成後分派。
