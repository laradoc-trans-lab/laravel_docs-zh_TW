# 通知

- [簡介](#introduction)
- [生成通知](#generating-notifications)
- [發送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定傳送通道](#specifying-delivery-channels)
    - [佇列通知](#queueing-notifications)
    - [即時通知](#on-demand-notifications)
- [郵件通知](#mail-notifications)
    - [格式化郵件訊息](#formatting-mail-messages)
    - [自訂寄件人](#customizing-the-sender)
    - [自訂收件人](#customizing-the-recipient)
    - [自訂主旨](#customizing-the-subject)
    - [自訂郵件發送器](#customizing-the-mailer)
    - [自訂範本](#customizing-the-templates)
    - [附件](#mail-attachments)
    - [新增標籤與中繼資料](#adding-tags-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
    - [使用 Mailables](#using-mailables)
    - [預覽郵件通知](#previewing-mail-notifications)
- [Markdown 郵件通知](#markdown-mail-notifications)
    - [生成訊息](#generating-the-message)
    - [撰寫訊息](#writing-the-message)
    - [自訂元件](#customizing-the-components)
- [資料庫通知](#database-notifications)
    - [先決條件](#database-prerequisites)
    - [格式化資料庫通知](#formatting-database-notifications)
    - [存取通知](#accessing-the-notifications)
    - [將通知標記為已讀](#marking-notifications-as-read)
- [廣播通知](#broadcast-notifications)
    - [先決條件](#broadcast-prerequisites)
    - [格式化廣播通知](#formatting-broadcast-notifications)
    - [監聽通知](#listening-for-notifications)
- [SMS 通知](#sms-notifications)
    - [先決條件](#sms-prerequisites)
    - [格式化 SMS 通知](#formatting-sms-notifications)
    - [自訂「寄件人」號碼](#customizing-the-from-number)
    - [新增客戶參考](#adding-a-client-reference)
    - [路由 SMS 通知](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [先決條件](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [路由 Slack 通知](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [本地化通知](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自訂通道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援[發送電子郵件](/docs/{{version}}/mail)外，Laravel 還支援透過各種傳送通道發送通知，包括電子郵件、SMS (透過 [Vonage](https://www.vonage.com/communications-apis/)，前身為 Nexmo) 和 [Slack](https://slack.com)。此外，還有各種[社群建構的通知通道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)可用於透過數十種不同的通道發送通知！通知也可以儲存在資料庫中，以便在您的網頁介面中顯示。

通常，通知應該是簡短的資訊性訊息，用於通知使用者應用程式中發生的事情。例如，如果您正在編寫一個帳務應用程式，您可能會透過電子郵件和 SMS 通道向使用者發送「發票已支付」通知。

<a name="generating-notifications"></a>
## 生成通知

在 Laravel 中，每個通知都由一個單獨的類別表示，通常儲存在 `app/Notifications` 目錄中。如果您的應用程式中沒有看到此目錄，請不用擔心，當您執行 `make:notification` Artisan 命令時，它將會為您建立：

```shell
php artisan make:notification InvoicePaid
```

此命令將在您的 `app/Notifications` 目錄中放置一個全新的通知類別。每個通知類別都包含一個 `via` 方法和數量不等的訊息建構方法，例如 `toMail` 或 `toDatabase`，這些方法將通知轉換為針對特定通道量身定制的訊息。

<a name="sending-notifications"></a>
## 發送通知

<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可以透過兩種方式發送：使用 `Notifiable` trait 的 `notify` 方法，或使用 `Notification` [Facade](/docs/{{version}}/facades)。`Notifiable` trait 預設包含在您的應用程式的 `App\Models\User` 模型中：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;
}
```

此 trait 提供的 `notify` 方法預期接收一個通知實例：

```php
use App\Notifications\InvoicePaid;

$user->notify(new InvoicePaid($invoice));
```

> [!NOTE]
> 請記住，您可以在任何模型上使用 `Notifiable` trait。您不限於僅將其包含在您的 `User` 模型中。

<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

或者，您可以透過 `Notification` [Facade](/docs/{{version}}/facades) 發送通知。當您需要向多個可通知實體 (例如使用者集合) 發送通知時，此方法非常有用。要使用 Facade 發送通知，請將所有可通知實體和通知實例傳遞給 `send` 方法：

```php
use Illuminate\Support\Facades\Notification;

Notification::send($users, new InvoicePaid($invoice));
```

您也可以使用 `sendNow` 方法立即發送通知。即使通知實作了 `ShouldQueue` 介面，此方法也會立即發送通知：

```php
Notification::sendNow($developers, new DeploymentCompleted($deployment));
```

<a name="specifying-delivery-channels"></a>
### 指定傳送通道

每個通知類別都有一個 `via` 方法，用於確定通知將透過哪些通道傳送。通知可以透過 `mail`、`database`、`broadcast`、`vonage` 和 `slack` 通道發送。

> [!NOTE]
> 如果您想使用其他傳送通道，例如 Telegram 或 Pusher，請查看社群驅動的 [Laravel Notification Channels 網站](http://laravel-notification-channels.com)。

`via` 方法接收一個 `$notifiable` 實例，該實例將是通知發送到的類別的實例。您可以使用 `$notifiable` 來確定通知應該透過哪些通道傳送：

```php
/**
 * 取得通知的傳送通道。
 *
 * @return array<int, string>
 */
public function via(object $notifiable): array
{
    return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
}
```

<a name="queueing-notifications"></a>
### 佇列通知

> [!WARNING]
> 在佇列通知之前，您應該配置您的佇列並[啟動一個工作者](/docs/{{version}}/queues#running-the-queue-worker)。

發送通知可能需要時間，特別是當通道需要進行外部 API 呼叫才能傳送通知時。為了加快應用程式的回應時間，您可以透過將 `ShouldQueue` 介面和 `Queueable` trait 添加到您的類別中，讓您的通知進入佇列。`make:notification` 命令生成的所有通知都已導入介面和 trait，因此您可以立即將它們添加到您的通知類別中：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    // ...
}
```

一旦 `ShouldQueue` 介面已添加到您的通知中，您就可以像往常一樣發送通知。Laravel 將檢測類別上的 `ShouldQueue` 介面，並自動將通知的傳送排入佇列：

```php
$user->notify(new InvoicePaid($invoice));
```

當佇列通知時，每個收件人和通道組合都會建立一個佇列工作。例如，如果您的通知有三個收件人和兩個通道，則會有六個工作被分派到佇列。

<a name="delaying-notifications"></a>
#### 延遲通知

如果您想延遲通知的傳送，您可以將 `delay` 方法鏈接到您的通知實例化：

```php
$delay = now()->addMinutes(10);

$user->notify((new InvoicePaid($invoice))->delay($delay));
```

您可以將一個陣列傳遞給 `delay` 方法，以指定特定通道的延遲時間：

```php
$user->notify((new InvoicePaid($invoice))->delay([
    'mail' => now()->addMinutes(5),
    'sms' => now()->addMinutes(10),
]));
```

或者，您可以在通知類別本身上定義一個 `withDelay` 方法。`withDelay` 方法應該返回一個通道名稱和延遲值的陣列：

```php
/**
 * 確定通知的傳送延遲。
 *
 * @return array<string, \Illuminate\Support\Carbon>
 */
public function withDelay(object $notifiable): array
{
    return [
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ];
}
```

<a name="customizing-the-notification-queue-connection"></a>
#### 自訂通知佇列連線

預設情況下，佇列通知將使用應用程式的預設佇列連線進行佇列。如果您想為特定通知指定不同的連線，您可以在通知的建構函式中呼叫 `onConnection` 方法：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    /**
     * 建立新的通知實例。
     */
    public function __construct()
    {
        $this->onConnection('redis');
    }
}
```

或者，如果您想為通知支援的每個通知通道指定一個特定的佇列連線，您可以在通知上定義一個 `viaConnections` 方法。此方法應該返回一個通道名稱/佇列連線名稱對的陣列：

```php
/**
 * 確定每個通知通道應使用的連線。
 *
 * @return array<string, string>
 */
public function viaConnections(): array
{
    return [
        'mail' => 'redis',
        'database' => 'sync',
    ];
}
```

<a name="customizing-notification-channel-queues"></a>
#### 自訂通知通道佇列

如果您想為通知支援的每個通知通道指定一個特定的佇列，您可以在通知上定義一個 `viaQueues` 方法。此方法應該返回一個通道名稱/佇列名稱對的陣列：

```php
/**
 * 確定每個通知通道應使用的佇列。
 *
 * @return array<string, string>
 */
public function viaQueues(): array
{
    return [
        'mail' => 'mail-queue',
        'slack' => 'slack-queue',
    ];
}
```

<a name="queued-notification-middleware"></a>
#### 佇列通知中介層

佇列通知可以定義中介層，[就像佇列工作一樣](/docs/{{version}}/queues#job-middleware)。首先，在您的通知類別上定義一個 `middleware` 方法。`middleware` 方法將接收 `$notifiable` 和 `$channel` 變數，這允許您根據通知的目的地自訂返回的中介層：

```php
use Illuminate\Queue\Middleware\RateLimited;

/**
 * 取得通知工作應通過的中介層。
 *
 * @return array<int, object>
 */
public function middleware(object $notifiable, string $channel)
{
    return match ($channel) {
        'mail' => [new RateLimited('postmark')],
        'slack' => [new RateLimited('slack')],
        default => [],
    };
}
```

<a name="queued-notifications-and-database-transactions"></a>
#### 佇列通知與資料庫交易

當佇列通知在資料庫交易中分派時，它們可能會在資料庫交易提交之前由佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的通知依賴於這些模型，則在處理發送佇列通知的工作時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 配置選項設定為 `false`，您仍然可以透過在發送通知時呼叫 `afterCommit` 方法來指示特定佇列通知應在所有開啟的資料庫交易提交後分派：

```php
use App\Notifications\InvoicePaid;

$user->notify((new InvoicePaid($invoice))->afterCommit());
```

或者，您可以在通知的建構函式中呼叫 `afterCommit` 方法：

```php
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    /**
     * 建立新的通知實例。
     */
    public function __construct()
    {
        $this->afterCommit();
    }
}
```

> [!NOTE]
> 要了解有關解決這些問題的更多資訊，請查閱有關[佇列工作和資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 判斷佇列通知是否應發送

佇列通知被分派到佇列進行背景處理後，通常會被佇列工作者接受並發送給預期的收件人。

但是，如果您想在佇列工作者處理佇列通知後，最終決定是否發送該通知，您可以在通知類別上定義一個 `shouldSend` 方法。如果此方法返回 `false`，則通知將不會發送：

```php
/**
 * 判斷通知是否應發送。
 */
public function shouldSend(object $notifiable, string $channel): bool
{
    return $this->invoice->isPaid();
}
```

<a name="on-demand-notifications"></a>
### 即時通知

有時您可能需要向未儲存為應用程式「使用者」的人發送通知。使用 `Notification` Facade 的 `route` 方法，您可以在發送通知之前指定臨時通知路由資訊：

```php
use Illuminate\Broadcasting\Channel;
use Illuminate\Support\Facades\Notification;

Notification::route('mail', 'taylor@example.com')
    ->route('vonage', '5555555555')
    ->route('slack', '#slack-channel')
    ->route('broadcast', [new Channel('channel-name')])
    ->notify(new InvoicePaid($invoice));
```

如果您想在向 `mail` 路由發送即時通知時提供收件人的姓名，您可以提供一個陣列，其中包含電子郵件地址作為鍵，姓名作為陣列中第一個元素的值：

```php
Notification::route('mail', [
    'barrett@example.com' => 'Barrett Blair',
])->notify(new InvoicePaid($invoice));
```

使用 `routes` 方法，您可以一次為多個通知通道提供臨時路由資訊：

```php
Notification::routes([
    'mail' => ['barrett@example.com' => 'Barrett Blair'],
    'vonage' => '5555555555',
])->notify(new InvoicePaid($invoice));
```

<a name="mail-notifications"></a>
## 郵件通知

<a name="formatting-mail-messages"></a>
### 格式化郵件訊息

如果通知支援以電子郵件形式發送，您應該在通知類別上定義一個 `toMail` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含一些簡單的方法，可幫助您建構交易電子郵件訊息。郵件訊息可以包含文字行以及「行動呼籲」。讓我們看看一個 `toMail` 方法的範例：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    $url = url('/invoice/'.$this->invoice->id);

    return (new MailMessage)
        ->greeting('Hello!')
        ->line('One of your invoices has been paid!')
        ->lineIf($this->amount > 0, "Amount paid: {$this->amount}")
        ->action('View Invoice', $url)
        ->line('Thank you for using our application!');
}
```

> [!NOTE]
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知生成訊息所需的任何資料傳遞到通知的建構函式中。

在此範例中，我們註冊了一個問候語、一行文字、一個行動呼籲，然後是另一行文字。`MailMessage` 物件提供的這些方法使格式化小型交易電子郵件變得簡單快捷。然後，郵件通道會將訊息組件轉換為美觀、響應式的 HTML 電子郵件範本，並附帶純文字對應項。以下是 `mail` 通道生成的電子郵件範例：

<img src="https://laravel.com/img/docs/notification-example-2.png">

> [!NOTE]
> 發送郵件通知時，請務必在 `config/app.php` 設定檔中設定 `name` 設定選項。此值將用於您的郵件通知訊息的標頭和頁腳。

<a name="error-messages"></a>
#### 錯誤訊息

有些通知會告知使用者錯誤，例如發票付款失敗。您可以在建構訊息時呼叫 `error` 方法，以指示郵件訊息與錯誤有關。當在郵件訊息上使用 `error` 方法時，行動呼籲按鈕將是紅色而不是黑色：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->error()
        ->subject('Invoice Payment Failed')
        ->line('...');
}
```

<a name="other-mail-notification-formatting-options"></a>
#### 其他郵件通知格式化選項

除了在通知類別中定義文字「行」之外，您還可以使用 `view` 方法指定應用於呈現通知電子郵件的自訂範本：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)->view(
        'mail.invoice.paid', ['invoice' => $this->invoice]
    );
}
```

您可以透過將視圖名稱作為傳遞給 `view` 方法的陣列的第二個元素來指定郵件訊息的純文字視圖：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)->view(
        ['mail.invoice.paid', 'mail.invoice.paid-text'],
        ['invoice' => $this->invoice]
    );
}
```

或者，如果您的訊息只有純文字視圖，您可以使用 `text` 方法：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)->text(
        'mail.invoice.paid-text', ['invoice' => $this->invoice]
    );
}
```

<a name="customizing-the-sender"></a>
### 自訂寄件人

預設情況下，電子郵件的寄件人/發件地址在 `config/mail.php` 設定檔中定義。但是，您可以使用 `from` 方法為特定通知指定發件地址：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->from('barrett@example.com', 'Barrett Blair')
        ->line('...');
}
```

<a name="customizing-the-recipient"></a>
### 自訂收件人

透過 `mail` 通道發送通知時，通知系統會自動在您的可通知實體上尋找 `email` 屬性。您可以透過在可通知實體上定義 `routeNotificationForMail` 方法來自訂用於傳送通知的電子郵件地址：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * 路由郵件通道的通知。
     *
     * @return  array<string, string>|string
     */
    public function routeNotificationForMail(Notification $notification): array|string
    {
        // 僅返回電子郵件地址...
        return $this->email_address;

        // 返回電子郵件地址和姓名...
        return [$this->email_address => $this->name];
    }
}
```

<a name="customizing-the-subject"></a>
### 自訂主旨

預設情況下，電子郵件的主旨是通知類別的名稱，格式化為「Title Case」。因此，如果您的通知類別名為 `InvoicePaid`，則電子郵件的主旨將是 `Invoice Paid`。如果您想為訊息指定不同的主旨，您可以在建構訊息時呼叫 `subject` 方法：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->subject('Notification Subject')
        ->line('...');
}
```

<a name="customizing-the-mailer"></a>
### 自訂郵件發送器

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件發送器發送。但是，您可以在建構訊息時呼叫 `mailer` 方法，在執行時指定不同的郵件發送器：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->mailer('postmark')
        ->line('...');
}
```

<a name="customizing-the-templates"></a>
### 自訂範本

您可以透過發布通知套件的資源來修改郵件通知使用的 HTML 和純文字範本。執行此命令後，郵件通知範本將位於 `resources/views/vendor/notifications` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 附件

要將附件添加到電子郵件通知中，請在建構訊息時使用 `attach` 方法。`attach` 方法將檔案的絕對路徑作為其第一個參數：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attach('/path/to/file');
}
```

> [!NOTE]
> 通知郵件訊息提供的 `attach` 方法也接受[可附加物件](/docs/{{version}}/mail#attachable-objects)。請查閱全面的[可附加物件文件](/docs/{{version}}/mail#attachable-objects)以了解更多資訊。

將檔案附加到訊息時，您還可以透過將 `array` 作為 `attach` 方法的第二個參數來指定顯示名稱和/或 MIME 類型：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attach('/path/to/file', [
            'as' => 'name.pdf',
            'mime' => 'application/pdf',
        ]);
}
```

與在可郵寄物件中附加檔案不同，您不能使用 `attachFromStorage` 直接從儲存磁碟附加檔案。您應該使用 `attach` 方法，並提供儲存磁碟上檔案的絕對路徑。或者，您可以從 `toMail` 方法返回一個[可郵寄物件](/docs/{{version}}/mail#generating-mailables)：

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;

/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): Mailable
{
    return (new InvoicePaidMailable($this->invoice))
        ->to($notifiable->email)
        ->attachFromStorage('/path/to/file');
}
```

必要時，可以使用 `attachMany` 方法將多個檔案附加到訊息中：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attachMany([
            '/path/to/forge.svg',
            '/path/to/vapor.svg' => [
                'as' => 'Logo.svg',
                'mime' => 'image/svg+xml',
            ],
        ]);
}
```

<a name="raw-data-attachments"></a>
#### 原始資料附件

`attachData` 方法可用於將原始位元組字串作為附件。呼叫 `attachData` 方法時，您應該提供應分配給附件的檔案名稱：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Hello!')
        ->attachData($this->pdf, 'name.pdf', [
            'mime' => 'application/pdf',
        ]);
}
```

<a name="adding-tags-metadata"></a>
### 新增標籤與中繼資料

一些第三方電子郵件提供商，例如 Mailgun 和 Postmark，支援訊息「標籤」和「中繼資料」，可用於對您的應用程式發送的電子郵件進行分組和追蹤。您可以透過 `tag` 和 `metadata` 方法將標籤和中繼資料添加到電子郵件訊息中：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->greeting('Comment Upvoted!')
        ->tag('upvote')
        ->metadata('comment_id', $this->comment->id);
}
```

如果您的應用程式正在使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件以獲取有關[標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags)和[中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages)的更多資訊。同樣，也可以查閱 Postmark 文件以獲取有關其對[標籤](https://postmarkapp.com/blog/tags-support-for-smtp)和[中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq)支援的更多資訊。

如果您的應用程式正在使用 Amazon SES 發送電子郵件，您應該使用 `metadata` 方法將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包，該閉包將在發送訊息之前使用 Symfony Message 實例調用。這讓您有機會在訊息傳送之前深度自訂訊息：

```php
use Symfony\Component\Mime\Email;

/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->withSymfonyMessage(function (Email $message) {
            $message->getHeaders()->addTextHeader(
                'Custom-Header', 'Header Value'
            );
        });
}
```

<a name="using-mailables"></a>
### 使用 Mailables

如有需要，您可以從通知的 `toMail` 方法返回一個完整的[可郵寄物件](/docs/{{version}}/mail)。當返回 `Mailable` 而不是 `MailMessage` 時，您需要使用可郵寄物件的 `to` 方法指定訊息收件人：

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;
use Illuminate\Mail\Mailable;

/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): Mailable
{
    return (new InvoicePaidMailable($this->invoice))
        ->to($notifiable->email);
}
```

<a name="mailables-and-on-demand-notifications"></a>
#### Mailables 與即時通知

如果您正在發送[即時通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將是 `Illuminate\Notifications\AnonymousNotifiable` 的實例，它提供了一個 `routeNotificationFor` 方法，可用於檢索即時通知應發送到的電子郵件地址：

```php
use App\Mail\InvoicePaid as InvoicePaidMailable;
use Illuminate\Notifications\AnonymousNotifiable;
use Illuminate\Mail\Mailable;

/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): Mailable
{
    $address = $notifiable instanceof AnonymousNotifiable
        ? $notifiable->routeNotificationFor('mail')
        : $notifiable->email;

    return (new InvoicePaidMailable($this->invoice))
        ->to($address);
}
```

<a name="previewing-mail-notifications"></a>
### 預覽郵件通知

在設計郵件通知範本時，方便地在瀏覽器中像典型的 Blade 範本一樣快速預覽呈現的郵件訊息。因此，Laravel 允許您直接從路由閉包或控制器返回郵件通知生成的任何郵件訊息。當返回 `MailMessage` 時，它將在瀏覽器中呈現並顯示，讓您無需將其發送到實際的電子郵件地址即可快速預覽其設計：

```php
use App\Models\Invoice;
use App\Notifications\InvoicePaid;

Route::get('/notification', function () {
    $invoice = Invoice::find(1);

    return (new InvoicePaid($invoice))
        ->toMail($invoice->user);
});
```

<a name="markdown-mail-notifications"></a>
## Markdown 郵件通知

Markdown 郵件通知讓您可以利用郵件通知的預建範本，同時讓您有更多自由撰寫更長、更自訂的訊息。由於訊息以 Markdown 撰寫，Laravel 能夠為訊息呈現美觀、響應式的 HTML 範本，同時自動生成純文字對應項。

<a name="generating-the-message"></a>
### 生成訊息

要生成帶有相應 Markdown 範本的通知，您可以使用 `make:notification` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

與所有其他郵件通知一樣，使用 Markdown 範本的通知應在其通知類別上定義一個 `toMail` 方法。但是，不是使用 `line` 和 `action` 方法來建構通知，而是使用 `markdown` 方法來指定應使用的 Markdown 範本的名稱。您可以將希望提供給範本的資料陣列作為方法的第二個參數傳遞：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    $url = url('/invoice/'.$this->invoice->id);

    return (new MailMessage)
        ->subject('Invoice Paid')
        ->markdown('mail.invoice.paid', ['url' => $url]);
}
```

<a name="writing-the-message"></a>
### 撰寫訊息

Markdown 郵件通知結合了 Blade 元件和 Markdown 語法，讓您可以輕鬆建構通知，同時利用 Laravel 預先製作的通知元件：

```blade
<x-mail::message>
# Invoice Paid

Your invoice has been paid!

<x-mail::button :url="$url">
View Invoice
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

> [!NOTE]
> 撰寫 Markdown 電子郵件時請勿過度縮排。根據 Markdown 標準，Markdown 解析器會將縮排內容呈現為程式碼區塊。

<a name="button-component"></a>
#### 按鈕元件

按鈕元件呈現一個置中的按鈕連結。該元件接受兩個參數：`url` 和一個可選的 `color`。支援的顏色有 `primary`、`green` 和 `red`。您可以根據需要向通知添加任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```

<a name="panel-component"></a>
#### 面板元件

面板元件將給定的文字區塊呈現在一個面板中，該面板的背景顏色與通知的其餘部分略有不同。這讓您可以將注意力吸引到給定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### 表格元件

表格元件允許您將 Markdown 表格轉換為 HTML 表格。該元件將 Markdown 表格作為其內容。表格欄位對齊支援使用預設的 Markdown 表格對齊語法：

```blade
<x-mail::table>
| Laravel       | Table         | Example       |
| ------------- | :-----------: | ------------: |
| Col 2 is      | Centered      | $10           |
| Col 3 is      | Right-Aligned | $20           |
</x-mail::table>
```

<a name="customizing-the-components"></a>
### 自訂元件

您可以將所有 Markdown 通知元件匯出到您自己的應用程式中進行自訂。要匯出元件，請使用 `vendor:publish` Artisan 命令發布 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件元件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含一個 `html` 和一個 `text` 目錄，每個目錄都包含其各自的每個可用元件的表示。您可以隨意自訂這些元件。

<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將自動內聯到 Markdown 通知中 HTML 表示中。

如果您想為 Laravel 的 Markdown 元件建構一個全新的主題，您可以將 CSS 檔案放置在 `html/themes` 目錄中。命名並儲存您的 CSS 檔案後，更新 `mail` 設定檔的 `theme` 選項以符合您的新主題名稱。

要為個別通知自訂主題，您可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受發送通知時應使用的主題名稱：

```php
/**
 * 取得通知的郵件表示。
 */
public function toMail(object $notifiable): MailMessage
{
    return (new MailMessage)
        ->theme('invoice')
        ->subject('Invoice Paid')
        ->markdown('mail.invoice.paid', ['url' => $url]);
}
```

<a name="database-notifications"></a>
## 資料庫通知

<a name="database-prerequisites"></a>
### 先決條件

`database` 通知通道將通知資訊儲存在資料庫表中。此表將包含通知類型以及描述通知的 JSON 資料結構等資訊。

您可以查詢該表以在應用程式的使用者介面中顯示通知。但是，在此之前，您需要建立一個資料庫表來儲存您的通知。您可以使用 `make:notifications-table` 命令生成一個具有正確表結構的[遷移](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的可通知模型正在使用 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，您應該在通知表遷移中將 `morphs` 方法替換為 [uuidMorphs](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [ulidMorphs](/docs/{{version}}/migrations#column-method-ulidMorphs)。

<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援儲存在資料庫表中，您應該在通知類別上定義一個 `toDatabase` 或 `toArray` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個純 PHP 陣列。返回的陣列將被編碼為 JSON 並儲存在 `notifications` 表的 `data` 欄位中。讓我們看看一個 `toArray` 方法的範例：

```php
/**
 * 取得通知的陣列表示。
 *
 * @return array<string, mixed>
 */
public function toArray(object $notifiable): array
{
    return [
        'invoice_id' => $this->invoice->id,
        'amount' => $this->invoice->amount,
    ];
}
```

當通知儲存在應用程式的資料庫中時，`type` 欄位預設將設定為通知的類別名稱，而 `read_at` 欄位將為 `null`。但是，您可以透過在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來自訂此行為：

```php
use Illuminate\Support\Carbon;

/**
 * 取得通知的資料庫類型。
 */
public function databaseType(object $notifiable): string
{
    return 'invoice-paid';
}

/**
 * 取得「read_at」欄位的初始值。
 */
public function initialDatabaseReadAtValue(): ?Carbon
{
    return null;
}
```

<a name="todatabase-vs-toarray"></a>
#### `toDatabase` 與 `toArray`

`toArray` 方法也用於 `broadcast` 通道，以確定要廣播到您的 JavaScript 前端哪些資料。如果您想為 `database` 和 `broadcast` 通道提供兩種不同的陣列表示，您應該定義一個 `toDatabase` 方法而不是 `toArray` 方法。

<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知儲存在資料庫中，您需要一種方便的方式從您的可通知實體存取它們。`Illuminate\Notifications\Notifiable` trait (包含在 Laravel 的預設 `App\Models\User` 模型中) 包含一個 `notifications` [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)，它返回實體的通知。要獲取通知，您可以像存取任何其他 Eloquent 關聯一樣存取此方法。預設情況下，通知將按 `created_at` 時間戳排序，最新通知位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->notifications as $notification) {
    echo $notification->type;
}
```

如果您只想檢索「未讀」通知，您可以使用 `unreadNotifications` 關聯。同樣，這些通知將按 `created_at` 時間戳排序，最新通知位於集合的開頭：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    echo $notification->type;
}
```

如果您只想檢索「已讀」通知，您可以使用 `readNotifications` 關聯：

```php
$user = App\Models\User::find(1);

foreach ($user->readNotifications as $notification) {
    echo $notification->type;
}
```

> [!NOTE]
> 要從您的 JavaScript 客戶端存取您的通知，您應該為您的應用程式定義一個通知控制器，該控制器返回可通知實體 (例如當前使用者) 的通知。然後，您可以從您的 JavaScript 客戶端向該控制器的 URL 發出 HTTP 請求。

<a name="marking-notifications-as-read"></a>
### 將通知標記為已讀

通常，當使用者查看通知時，您會希望將其標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，該方法會更新通知資料庫記錄上的 `read_at` 欄位：

```php
$user = App\Models\User::find(1);

foreach ($user->unreadNotifications as $notification) {
    $notification->markAsRead();
}
```

但是，您也可以直接在通知集合上使用 `markAsRead` 方法，而不是遍歷每個通知：

```php
$user->unreadNotifications->markAsRead();
```

您也可以使用大量更新查詢來將所有通知標記為已讀，而無需從資料庫中檢索它們：

```php
$user = App\Models\User::find(1);

$user->unreadNotifications()->update(['read_at' => now()]);
```

您可以 `delete` 通知以將它們從表中完全刪除：

```php
$user->notifications()->delete();
```

<a name="broadcast-notifications"></a>
## 廣播通知

<a name="broadcast-prerequisites"></a>
### 先決條件

在廣播通知之前，您應該配置並熟悉 Laravel 的[事件廣播](/docs/{{version}}/broadcasting)服務。事件廣播提供了一種從您的 JavaScript 前端回應伺服器端 Laravel 事件的方式。

<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 通道使用 Laravel 的[事件廣播](/docs/{{version}}/broadcasting)服務廣播通知，允許您的 JavaScript 前端即時捕獲通知。如果通知支援廣播，您可以在通知類別上定義一個 `toBroadcast` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `BroadcastMessage` 實例。如果 `toBroadcast` 方法不存在，則將使用 `toArray` 方法來收集應廣播的資料。返回的資料將被編碼為 JSON 並廣播到您的 JavaScript 前端。讓我們看看一個 `toBroadcast` 方法的範例：

```php
use Illuminate\Notifications\Messages\BroadcastMessage;

/**
 * 取得通知的可廣播表示。
 */
public function toBroadcast(object $notifiable): BroadcastMessage
{
    return new BroadcastMessage([
        'invoice_id' => $this->invoice->id,
        'amount' => $this->invoice->amount,
    ]);
}
```

<a name="broadcast-queue-configuration"></a>
#### 廣播佇列配置

所有廣播通知都已排入佇列等待廣播。如果您想配置用於佇列廣播操作的佇列連線或佇列名稱，您可以使用 `BroadcastMessage` 的 `onConnection` 和 `onQueue` 方法：

```php
return (new BroadcastMessage($data))
    ->onConnection('sqs')
    ->onQueue('broadcasts');
```

<a name="customizing-the-notification-type"></a>
#### 自訂通知類型

除了您指定的資料之外，所有廣播通知還具有一個 `type` 欄位，其中包含通知的完整類別名稱。如果您想自訂通知 `type`，您可以在通知類別上定義一個 `broadcastType` 方法：

```php
/**
 * 取得正在廣播的通知類型。
 */
public function broadcastType(): string
{
    return 'broadcast.message';
}
```

<a name="listening-for-notifications"></a>
### 監聽通知

通知將在以 `{notifiable}.{id}` 慣例格式化的私有通道上廣播。因此，如果您向 ID 為 `1` 的 `App\Models\User` 實例發送通知，則通知將在 `App.Models.User.1` 私有通道上廣播。使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以使用 `notification` 方法輕鬆監聽通道上的通知：

```js
Echo.private('App.Models.User.' + userId)
    .notification((notification) => {
        console.log(notification.type);
    });
```

<a name="using-react-or-vue"></a>
#### 使用 React 或 Vue

Laravel Echo 包含 React 和 Vue Hooks，讓監聽通知變得輕而易舉。首先，調用 `useEchoNotification` Hook，它用於監聽通知。當消費元件卸載時，`useEchoNotification` Hook 將自動離開通道：

```js tab=React
import { useEchoNotification } from "@laravel/echo-react";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
);
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoNotification } from "@laravel/echo-vue";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
);
</script>
```

預設情況下，Hook 會監聽所有通知。要指定您想監聽的通知類型，您可以向 `useEchoNotification` 提供一個字串或類型陣列：

```js tab=React
import { useEchoNotification } from "@laravel/echo-react";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
```

```vue tab=Vue
<script setup lang="ts">
import { useEchoNotification } from "@laravel/echo-vue";

useEchoNotification(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
</script>
```

您還可以指定通知酬載資料的形狀，提供更高的類型安全性和編輯便利性：

```ts
type InvoicePaidNotification = {
    invoice_id: number;
    created_at: string;
};

useEchoNotification<InvoicePaidNotification>(
    `App.Models.User.${userId}`,
    (notification) => {
        console.log(notification.invoice_id);
        console.log(notification.created_at);
        console.log(notification.type);
    },
    'App.Notifications.InvoicePaid',
);
```

<a name="customizing-the-notification-channel"></a>
#### 自訂通知通道

如果您想自訂實體的廣播通知將在哪個通道上廣播，您可以在可通知實體上定義一個 `receivesBroadcastNotificationsOn` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * 使用者接收通知廣播的通道。
     */
    public function receivesBroadcastNotificationsOn(): string
    {
        return 'users.'.$this->id;
    }
}
```

<a name="sms-notifications"></a>
## SMS 通知

<a name="sms-prerequisites"></a>
### 先決條件

Laravel 中的 SMS 通知由 [Vonage](https://www.vonage.com/) (前身為 Nexmo) 提供支援。在您透過 Vonage 發送通知之前，您需要安裝 `laravel/vonage-notification-channel` 和 `guzzlehttp/guzzle` 套件：

```shell
composer require laravel/vonage-notification-channel guzzlehttp/guzzle
```

該套件包含一個[設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。但是，您不需要將此設定檔匯出到您自己的應用程式中。您只需使用 `VONAGE_KEY` 和 `VONAGE_SECRET` 環境變數來定義您的 Vonage 公鑰和密鑰。

定義您的金鑰後，您應該設定一個 `VONAGE_SMS_FROM` 環境變數，該變數定義您的 SMS 訊息預設應從哪個電話號碼發送。您可以在 Vonage 控制面板中生成此電話號碼：

```ini
VONAGE_SMS_FROM=15556666666
```

<a name="formatting-sms-notifications"></a>
### 格式化 SMS 通知

如果通知支援以 SMS 形式發送，您應該在通知類別上定義一個 `toVonage` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * 取得通知的 Vonage / SMS 表示。
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content('Your SMS message content');
}
```

<a name="unicode-content"></a>
#### Unicode 內容

如果您的 SMS 訊息將包含 Unicode 字元，您應該在建構 `VonageMessage` 實例時呼叫 `unicode` 方法：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * 取得通知的 Vonage / SMS 表示。
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content('Your unicode message')
        ->unicode();
}
```

<a name="customizing-the-from-number"></a>
### 自訂「寄件人」號碼

如果您想從與 `VONAGE_SMS_FROM` 環境變數指定的電話號碼不同的電話號碼發送某些通知，您可以在 `VonageMessage` 實例上呼叫 `from` 方法：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * 取得通知的 Vonage / SMS 表示。
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->content('Your SMS message content')
        ->from('15554443333');
}
```

<a name="adding-a-client-reference"></a>
### 新增客戶參考

如果您想追蹤每個使用者、團隊或客戶的成本，您可以將「客戶參考」添加到通知中。Vonage 將允許您使用此客戶參考生成報告，以便您更好地了解特定客戶的 SMS 使用情況。客戶參考可以是任何長度不超過 40 個字元的字串：

```php
use Illuminate\Notifications\Messages\VonageMessage;

/**
 * 取得通知的 Vonage / SMS 表示。
 */
public function toVonage(object $notifiable): VonageMessage
{
    return (new VonageMessage)
        ->clientReference((string) $notifiable->id)
        ->content('Your SMS message content');
}
```

<a name="routing-sms-notifications"></a>
### 路由 SMS 通知

要將 Vonage 通知路由到正確的電話號碼，請在您的可通知實體上定義一個 `routeNotificationForVonage` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * 路由 Vonage 通道的通知。
     */
    public function routeNotificationForVonage(Notification $notification): string
    {
        return $this->phone_number;
    }
}
```

<a name="slack-notifications"></a>
## Slack 通知

<a name="slack-prerequisites"></a>
### 先決條件

在發送 Slack 通知之前，您應該透過 Composer 安裝 Slack 通知通道：

```shell
composer composer require laravel/slack-notification-channel
```

此外，您必須為您的 Slack 工作區建立一個 [Slack App](https://api.slack.com/apps?new_app=1)。

如果您只需要向建立 App 的相同 Slack 工作區發送通知，您應該確保您的 App 具有 `chat:write`、`chat:write.public` 和 `chat:write.customize` 範圍。這些範圍可以從 Slack 內部的「OAuth & Permissions」App 管理選項卡中添加。

接下來，複製 App 的「Bot User OAuth Token」並將其放置在您的應用程式的 `services.php` 設定檔中的 `slack` 設定陣列中。此權杖可以在 Slack 內部的「OAuth & Permissions」選項卡上找到：

```php
'slack' => [
    'notifications' => [
        'bot_user_oauth_token' => env('SLACK_BOT_USER_OAUTH_TOKEN'),
        'channel' => env('SLACK_BOT_USER_DEFAULT_CHANNEL'),
    ],
],
```

<a name="slack-app-distribution"></a>
#### App 分發

如果您的應用程式將向您的應用程式使用者擁有的外部 Slack 工作區發送通知，您將需要透過 Slack「分發」您的 App。App 分發可以從您的 App 在 Slack 內部的「Manage Distribution」選項卡中管理。一旦您的 App 已分發，您可以使用 [Socialite](/docs/{{version}}/socialite) 代表您的應用程式使用者[獲取 Slack Bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。

<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援以 Slack 訊息形式發送，您應該在通知類別上定義一個 `toSlack` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Slack\SlackMessage` 實例。您可以使用 [Slack 的 Block Kit API](https://api.slack.com/block-kit) 建構豐富的通知。以下範例可以在 [Slack 的 Block Kit Builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中預覽：

```php
use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
use Illuminate\Notifications\Slack\SlackMessage;

/**
 * 取得通知的 Slack 表示。
 */
public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text('One of your invoices has been paid!')
        ->headerBlock('Invoice Paid')
        ->contextBlock(function (ContextBlock $block) {
            $block->text('Customer #1234');
        })
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('An invoice has been paid.');
            $block->field("*Invoice No:*\n1000")->markdown();
            $block->field("*Invoice Recipient:*\ntaylor@laravel.com")->markdown();
        })
        ->dividerBlock()
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('Congratulations!');
        });
}
```

<a name="using-slacks-block-kit-builder-template"></a>
#### 使用 Slack 的 Block Kit Builder 範本

除了使用流暢的訊息建構器方法來建構您的 Block Kit 訊息之外，您還可以將 Slack 的 Block Kit Builder 生成的原始 JSON 酬載提供給 `usingBlockKitTemplate` 方法：

```php
use Illuminate\Notifications\Slack\SlackMessage;
use Illuminate\Support\Str;

/**
 * 取得通知的 Slack 表示。
 */
public function toSlack(object $notifiable): SlackMessage
{
    $template = <<<JSON
        {
          "blocks": [
            {
              "type": "header",
              "text": {
                "type": "plain_text",
                "text": "Team Announcement"
              }
            },
            {
              "type": "section",
              "text": {
                "type": "plain_text",
                "text": "We are hiring!"
              }
            }
          ]
        }
    JSON;

    return (new SlackMessage)
        ->usingBlockKitTemplate($template);
}
```

<a name="slack-interactivity"></a>
### Slack 互動性

Slack 的 Block Kit 通知系統提供了強大的功能來[處理使用者互動](https://api.slack.com/interactivity/handling)。要利用這些功能，您的 Slack App 應該啟用「Interactivity」並配置一個指向您的應用程式提供的 URL 的「Request URL」。這些設定可以從 Slack 內部的「Interactivity & Shortcuts」App 管理選項卡中管理。

在以下範例中，它利用了 `actionsBlock` 方法，Slack 將向您的「Request URL」發送一個 `POST` 請求，其中包含一個酬載，其中包含點擊按鈕的 Slack 使用者、點擊按鈕的 ID 等。您的應用程式可以根據酬載確定要採取的動作。您還應該[驗證請求](https://api.slack.com/authentication/verifying-requests-from-slack)是由 Slack 發出的：

```php
use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
use Illuminate\Notifications\Slack\SlackMessage;

/**
 * 取得通知的 Slack 表示。
 */
public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text('One of your invoices has been paid!')
        ->headerBlock('Invoice Paid')
        ->contextBlock(function (ContextBlock $block) {
            $block->text('Customer #1234');
        })
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('An invoice has been paid.');
        })
        ->actionsBlock(function (ActionsBlock $block) {
             // ID defaults to "button_acknowledge_invoice"...
            $block->button('Acknowledge Invoice')->primary();

            // Manually configure the ID...
            $block->button('Deny')->danger()->id('deny_invoice');
        });
}
```

<a name="slack-confirmation-modals"></a>
#### 確認對話框

如果您希望使用者在執行動作之前必須確認，您可以在定義按鈕時調用 `confirm` 方法。`confirm` 方法接受一個訊息和一個閉包，該閉包接收一個 `ConfirmObject` 實例：

```php
use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
use Illuminate\Notifications\Slack\BlockKit\Composites\ConfirmObject;
use Illuminate\Notifications\Slack\SlackMessage;

/**
 * 取得通知的 Slack 表示。
 */
public function toSlack(object $notifiable): SlackMessage
{
    return (new SlackMessage)
        ->text('One of your invoices has been paid!')
        ->headerBlock('Invoice Paid')
        ->contextBlock(function (ContextBlock $block) {
            $block->text('Customer #1234');
        })
        ->sectionBlock(function (SectionBlock $block) {
            $block->text('An invoice has been paid.');
        })
        ->actionsBlock(function (ActionsBlock $block) {
            $block->button('Acknowledge Invoice')
                ->primary()
                ->confirm(
                    'Acknowledge the payment and send a thank you email?',
                    function (ConfirmObject $dialog) {
                        $dialog->confirm('Yes');
                        $dialog->deny('No');
                    }
                );
        });
}
```

<a name="inspecting-slack-blocks"></a>
#### 檢查 Slack 區塊

如果您想快速檢查您正在建構的區塊，您可以在 `SlackMessage` 實例上調用 `dd` 方法。`dd` 方法將生成並轉儲一個 URL 到 Slack 的 [Block Kit Builder](https://app.slack.com/block-kit-builder/)，它會在您的瀏覽器中顯示酬載和通知的預覽。您可以將 `true` 傳遞給 `dd` 方法以轉儲原始酬載：

```php
return (new SlackMessage)
    ->text('One of your invoices has been paid!')
    ->headerBlock('Invoice Paid')
    ->dd();
```

<a name="routing-slack-notifications"></a>
### 路由 Slack 通知

要將 Slack 通知導向適當的 Slack 團隊和通道，請在您的可通知模型上定義一個 `routeNotificationForSlack` 方法。此方法可以返回以下三個值之一：

- `null` - 將路由延遲到通知本身中配置的通道。您可以在建構 `SlackMessage` 時使用 `to` 方法在通知中配置通道。
- 指定要發送通知的 Slack 通道字串，例如 `#support-channel`。
- 一個 `SlackRoute` 實例，它允許您指定 OAuth 權杖和通道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應用於向外部工作區發送通知。

例如，從 `routeNotificationForSlack` 方法返回 `#support-channel` 將把通知發送到與您的應用程式的 `services.php` 設定檔中 Bot User OAuth 權杖相關聯的工作區中的 `#support-channel` 通道：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * 路由 Slack 通道的通知。
     */
    public function routeNotificationForSlack(Notification $notification): mixed
    {
        return '#support-channel';
    }
}
```

<a name="notifying-external-slack-workspaces"></a>
### 通知外部 Slack 工作區

> [!NOTE]
> 在向外部 Slack 工作區發送通知之前，您的 Slack App 必須已[分發](#slack-app-distribution)。

當然，您通常會希望向您的應用程式使用者擁有的 Slack 工作區發送通知。為此，您首先需要為使用者獲取一個 Slack OAuth 權杖。幸運的是，[Laravel Socialite](/docs/{{version}}/socialite) 包含一個 Slack 驅動程式，它將允許您輕鬆地將您的應用程式使用者與 Slack 進行身份驗證並[獲取一個 Bot 權杖](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您獲得了 Bot 權杖並將其儲存在您的應用程式的資料庫中，您就可以使用 `SlackRoute::make` 方法將通知路由到使用者的工作區。此外，您的應用程式可能需要提供一個機會讓使用者指定通知應發送到哪個通道：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Slack\SlackRoute;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * 路由 Slack 通道的通知。
     */
    public function routeNotificationForSlack(Notification $notification): mixed
    {
        return SlackRoute::make($this->slack_channel, $this->slack_token);
    }
}
```

<a name="localizing-notifications"></a>
## 本地化通知

Laravel 允許您以 HTTP 請求當前語系以外的語系發送通知，即使通知已排入佇列，它也會記住此語系。

為此，`Illuminate\Notifications\Notification` 類別提供了 `locale` 方法來設定所需的語言。應用程式將在評估通知時切換到此語系，然後在評估完成後恢復到先前的語系：

```php
$user->notify((new InvoicePaid($invoice))->locale('es'));
```

多個可通知實體的本地化也可以透過 `Notification` Facade 實現：

```php
Notification::locale('es')->send(
    $users, new InvoicePaid($invoice)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時，應用程式會儲存每個使用者偏好的語系。透過在您的可通知模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在發送通知時使用此儲存的語系：

```php
use Illuminate\Contracts\Translation\HasLocalePreference;

class User extends Model implements HasLocalePreference
{
    /**
     * 取得使用者偏好的語系。
     */
    public function preferredLocale(): string
    {
        return $this->locale;
    }
}
```

一旦您實作了介面，Laravel 將在向模型發送通知和可郵寄物件時自動使用偏好的語系。因此，在使用此介面時無需呼叫 `locale` 方法：

```php
$user->notify(new InvoicePaid($invoice));
```

<a name="testing"></a>
## 測試

您可以使用 `Notification` Facade 的 `fake` 方法來阻止通知發送。通常，發送通知與您實際測試的程式碼無關。最有可能的是，只需斷言 Laravel 已被指示發送給定的通知就足夠了。

呼叫 `Notification` Facade 的 `fake` 方法後，您可以斷言通知已被指示發送給使用者，甚至檢查通知收到的資料：

```php tab=Pest
<?php

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;

test('orders can be shipped', function () {
    Notification::fake();

    // 執行訂單發貨...

    // 斷言沒有發送任何通知...
    Notification::assertNothingSent();

    // 斷言已向給定使用者發送通知...
    Notification::assertSentTo(
        [$user], OrderShipped::class
    );

    // 斷言沒有發送通知...
    Notification::assertNotSentTo(
        [$user], AnotherNotification::class
    );

    // 斷言通知已發送兩次...
    Notification::assertSentTimes(WeeklyReminder::class, 2);

    // 斷言已發送指定數量的通知...
    Notification::assertCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_orders_can_be_shipped(): void
    {
        Notification::fake();

        // 執行訂單發貨...

        // 斷言沒有發送任何通知...
        Notification::assertNothingSent();

        // 斷言已向給定使用者發送通知...
        Notification::assertSentTo(
            [$user], OrderShipped::class
        );

        // 斷言沒有發送通知...
        Notification::assertNotSentTo(
            [$user], AnotherNotification::class
        );

        // 斷言通知已發送兩次...
        Notification::assertSentTimes(WeeklyReminder::class, 2);

        // 斷言已發送指定數量的通知...
        Notification::assertCount(3);
    }
}
```

您可以將閉包傳遞給 `assertSentTo` 或 `assertNotSentTo` 方法，以斷言已發送通過給定「真值測試」的通知。如果至少有一個通知通過給定的真值測試，則斷言將成功：

```php
Notification::assertSentTo(
    $user,
    function (OrderShipped $notification, array $channels) use ($order) {
        return $notification->order->id === $order->id;
    }
);
```

<a name="on-demand-notifications"></a>
#### 即時通知

如果您正在測試的程式碼發送[即時通知](#on-demand-notifications)，您可以透過 `assertSentOnDemand` 方法測試即時通知是否已發送：

```php
Notification::assertSentOnDemand(OrderShipped::class);
```

透過將閉包作為 `assertSentOnDemand` 方法的第二個參數，您可以確定即時通知是否已發送到正確的「路由」地址：

```php
Notification::assertSentOnDemand(
    OrderShipped::class,
    function (OrderShipped $notification, array $channels, object $notifiable) use ($user) {
        return $notifiable->routes['mail'] === $user->email;
    }
);
```

<a name="notification-events"></a>
## 通知事件

<a name="notification-sending-event"></a>
#### 通知發送事件

當通知正在發送時，通知系統會分派 `Illuminate\Notifications\Events\NotificationSending` 事件。此事件包含「可通知」實體和通知實例本身。您可以在應用程式中為此事件建立[事件監聽器](/docs/{{version}}/events)：

```php
use Illuminate\Notifications\Events\NotificationSending;

class CheckNotificationStatus
{
    /**
     * 處理事件。
     */
    public function handle(NotificationSending $event): void
    {
        // ...
    }
}
```

如果 `NotificationSending` 事件的事件監聽器從其 `handle` 方法返回 `false`，則通知將不會發送：

```php
/**
 * 處理事件。
 */
public function handle(NotificationSending $event): bool
{
    return false;
}
```

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification` 和 `channel` 屬性，以了解有關通知收件人或通知本身的更多資訊：

```php
/**
 * 處理事件。
 */
public function handle(NotificationSending $event): void
{
    // $event->channel
    // $event->notifiable
    // $event->notification
}
```

<a name="notification-sent-event"></a>
#### 通知已發送事件

當通知發送後，通知系統會分派 `Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events)。此事件包含「可通知」實體和通知實例本身。您可以在應用程式中為此事件建立[事件監聽器](/docs/{{version}}/events)：

```php
use Illuminate\Notifications\Events\NotificationSent;

class LogNotification
{
    /**
     * 處理事件。
     */
    public function handle(NotificationSent $event): void
    {
        // ...
    }
}
```

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification`、`channel` 和 `response` 屬性，以了解有關通知收件人或通知本身的更多資訊：

```php
/**
 * 處理事件。
 */
public function handle(NotificationSent $event): void
{
    // $event->channel
    // $event->notifiable
    // $event->notification
    // $event->response
}
```

<a name="custom-channels"></a>
## 自訂通道

Laravel 附帶了一些通知通道，但您可能希望編寫自己的驅動程式以透過其他通道傳送通知。Laravel 使其變得簡單。首先，定義一個包含 `send` 方法的類別。該方法應接收兩個參數：一個 `$notifiable` 和一個 `$notification`。

在 `send` 方法中，您可以呼叫通知上的方法以檢索通道理解的訊息物件，然後以您希望的任何方式將通知發送給 `$notifiable` 實例：

```php
<?php

namespace App\Notifications;

use Illuminate\Notifications\Notification;

class VoiceChannel
{
    /**
     * 發送給定的通知。
     */
    public function send(object $notifiable, Notification $notification): void
    {
        $message = $notification->toVoice($notifiable);

        // 將通知發送給 $notifiable 實例...
    }
}
```

一旦您的通知通道類別已定義，您可以從任何通知的 `via` 方法返回類別名稱。在此範例中，通知的 `toVoice` 方法可以返回您選擇的任何物件來表示語音訊息。例如，您可以定義自己的 `VoiceMessage` 類別來表示這些訊息：

```php
<?php

namespace App\Notifications;

use App\Notifications\Messages\VoiceMessage;
use App\Notifications\VoiceChannel;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification
{
    use Queueable;

    /**
     * 取得通知通道。
     */
    public function via(object $notifiable): string
    {
        return VoiceChannel::class;
    }

    /**
     * 取得通知的語音表示。
     */
    public function toVoice(object $notifiable): VoiceMessage
    {
        // ...
    }
}
```
