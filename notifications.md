# 通知

- [簡介](#introduction)
- [產生通知](#generating-notifications)
- [傳送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定傳送管道](#specifying-delivery-channels)
    - [將通知加入佇列](#queueing-notifications)
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
    - [使用 Mailable](#using-mailables)
    - [預覽郵件通知](#previewing-mail-notifications)
- [Markdown 郵件通知](#markdown-mail-notifications)
    - [產生訊息](#generating-the-message)
    - [撰寫訊息](#writing-the-message)
    - [自訂元件](#customizing-the-components)
- [資料庫通知](#database-notifications)
    - [前置作業](#database-prerequisites)
    - [格式化資料庫通知](#formatting-database-notifications)
    - [存取通知](#accessing-the-notifications)
    - [將通知標記為已讀](#marking-notifications-as-read)
- [廣播通知](#broadcast-notifications)
    - [前置作業](#broadcast-prerequisites)
    - [格式化廣播通知](#formatting-broadcast-notifications)
    - [監聽通知](#listening-for-notifications)
- [SMS 通知](#sms-notifications)
    - [前置作業](#sms-prerequisites)
    - [格式化 SMS 通知](#formatting-sms-notifications)
    - [Unicode 內容](#unicode-content)
    - [自訂「寄件人」號碼](#customizing-the-from-number)
    - [新增客戶參考](#adding-a-client-reference)
    - [路由 SMS 通知](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [前置作業](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [路由 Slack 通知](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [通知本地化](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自訂管道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援[傳送電子郵件](/docs/{{version}}/mail)之外，Laravel 還支援透過各種傳送管道傳送通知，包括電子郵件、SMS (透過 [Vonage](https://www.vonage.com/communications-apis/)，前身為 Nexmo) 和 [Slack](https://slack.com)。此外，還有各種[社群建立的通知管道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)可用於透過數十種不同的管道傳送通知！通知也可以儲存在資料庫中，以便在您的網頁介面中顯示。

通常，通知應該是簡短的資訊性訊息，用於告知使用者應用程式中發生的事情。例如，如果您正在撰寫一個帳務應用程式，您可能會透過電子郵件和 SMS 管道向使用者傳送「發票已支付」通知。

<a name="generating-notifications"></a>
## 產生通知

在 Laravel 中，每個通知都由一個單一的類別表示，該類別通常儲存在 `app/Notifications` 目錄中。如果您的應用程式中沒有看到此目錄，請不用擔心，當您執行 `make:notification` Artisan 命令時，它將會為您建立：

```shell
php artisan make:notification InvoicePaid
```

此命令將在您的 `app/Notifications` 目錄中放置一個全新的通知類別。每個通知類別都包含一個 `via` 方法和數量不定的訊息建構方法，例如 `toMail` 或 `toDatabase`，這些方法會將通知轉換為針對該特定管道量身打造的訊息。

<a name="sending-notifications"></a>
## 傳送通知

<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可以透過兩種方式傳送：使用 `Notifiable` trait 的 `notify` 方法，或使用 `Notification` [facade](/docs/{{version}}/facades)。`Notifiable` trait 預設包含在您應用程式的 `App\Models\User` 模型中：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable
    {
        use Notifiable;
    }

此 trait 提供的 `notify` 方法預期接收一個通知實例：

    use App\Notifications\InvoicePaid;

    $user->notify(new InvoicePaid($invoice));

> [!NOTE]
> 請記住，您可以在任何模型上使用 `Notifiable` trait。您不限於僅將其包含在 `User` 模型中。

<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

或者，您可以透過 `Notification` [facade](/docs/{{version}}/facades) 傳送通知。當您需要向多個可通知實體 (例如使用者集合) 傳送通知時，此方法非常有用。要使用 facade 傳送通知，請將所有可通知實體和通知實例傳遞給 `send` 方法：

    use Illuminate\Support\Facades\Notification;

    Notification::send($users, new InvoicePaid($invoice));

您也可以使用 `sendNow` 方法立即傳送通知。即使通知實作了 `ShouldQueue` 介面，此方法也會立即傳送通知：

    Notification::sendNow($developers, new DeploymentCompleted($deployment));

<a name="specifying-delivery-channels"></a>
### 指定傳送管道

每個通知類別都有一個 `via` 方法，用於決定通知將透過哪些管道傳送。通知可以透過 `mail`、`database`、`broadcast`、`vonage` 和 `slack` 管道傳送。

> [!NOTE]
> 如果您想使用其他傳送管道，例如 Telegram 或 Pusher，請查看社群驅動的 [Laravel Notification Channels 網站](http://laravel-notification-channels.com)。

`via` 方法接收一個 `$notifiable` 實例，該實例將是通知傳送到的類別實例。您可以使用 `$notifiable` 來決定通知應該透過哪些管道傳送：

    /**
     * 取得通知的傳送管道。
     *
     * @return array<int, string>
     */
    public function via(object $notifiable): array
    {
        return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
    }

<a name="queueing-notifications"></a>
### 將通知加入佇列

> [!WARNING]
> 在將通知加入佇列之前，您應該設定您的佇列並[啟動一個 Worker](/docs/{{version}}/queues#running-the-queue-worker)。

傳送通知可能需要時間，特別是當管道需要進行外部 API 呼叫才能傳送通知時。為了加快應用程式的回應時間，您可以透過在類別中新增 `ShouldQueue` 介面和 `Queueable` trait 來讓您的通知排入佇列。使用 `make:notification` 命令產生的所有通知都已匯入此介面和 trait，因此您可以立即將它們新增到您的通知類別中：

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

一旦 `ShouldQueue` 介面已新增到您的通知中，您就可以像往常一樣傳送通知。Laravel 將偵測類別上的 `ShouldQueue` 介面，並自動將通知的傳送排入佇列：

    $user->notify(new InvoicePaid($invoice));

當通知排入佇列時，每個收件人和管道組合都會建立一個佇列工作。例如，如果您的通知有三個收件人和兩個管道，則會將六個工作分派到佇列中。

<a name="delaying-notifications"></a>
#### 延遲通知

如果您想延遲通知的傳送，您可以在通知實例化時鏈接 `delay` 方法：

    $delay = now()->addMinutes(10);

    $user->notify((new InvoicePaid($invoice))->delay($delay));

您可以向 `delay` 方法傳遞一個陣列，以指定特定管道的延遲時間：

    $user->notify((new InvoicePaid($invoice))->delay([
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ]));

或者，您可以在通知類別本身上定義一個 `withDelay` 方法。`withDelay` 方法應該返回一個包含管道名稱和延遲值的陣列：

    /**
     * 決定通知的傳送延遲。
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

<a name="customizing-the-notification-queue-connection"></a>
#### 自訂通知佇列連線

預設情況下，排入佇列的通知將使用您應用程式的預設佇列連線進行排入佇列。如果您想為特定通知指定不同的連線，您可以在通知的建構函式中呼叫 `onConnection` 方法：

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

或者，如果您想為通知支援的每個通知管道指定一個特定的佇列連線，您可以在通知上定義一個 `viaConnections` 方法。此方法應該返回一個管道名稱/佇列連線名稱對的陣列：

    /**
     * 決定每個通知管道應該使用哪些連線。
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

<a name="customizing-notification-channel-queues"></a>
#### 自訂通知管道佇列

如果您想為通知支援的每個通知管道指定一個特定的佇列，您可以在通知上定義一個 `viaQueues` 方法。此方法應該返回一個管道名稱/佇列名稱對的陣列：

    /**
     * 決定每個通知管道應該使用哪些佇列。
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

<a name="queued-notification-middleware"></a>
#### 排入佇列的通知 Middleware

排入佇列的通知可以像[排入佇列的工作](/docs/{{version}}/queues#job-middleware)一樣定義 Middleware。首先，在您的通知類別上定義一個 `middleware` 方法。`middleware` 方法將接收 `$notifiable` 和 `$channel` 變數，這允許您根據通知的目的地自訂返回的 Middleware：

    use Illuminate\Queue\Middleware\RateLimited;

    /**
     * 取得通知工作應該通過的 Middleware。
     *
     * @return array<int, object>
     */
    public function middleware(object $notifiable, string $channel)
    {
        return match ($channel) {
            'email' => [new RateLimited('postmark')],
            'slack' => [new RateLimited('slack')],
            default => [],
        };
    }

<a name="queued-notifications-and-database-transactions"></a>
#### 排入佇列的通知與資料庫交易

當排入佇列的通知在資料庫交易中分派時，它們可能會在資料庫交易提交之前由佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的通知依賴於這些模型，則在處理傳送排入佇列通知的工作時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設定為 `false`，您仍然可以在傳送通知時呼叫 `afterCommit` 方法，以指示特定排入佇列的通知應該在所有開啟的資料庫交易提交後分派：

    use App\Notifications\InvoicePaid;

    $user->notify((new InvoicePaid($invoice))->afterCommit());

或者，您可以在通知的建構函式中呼叫 `afterCommit` 方法：

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

> [!NOTE]
> 要了解有關解決這些問題的更多資訊，請查閱有關[排入佇列的工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 判斷排入佇列的通知是否應該傳送

排入佇列的通知已分派到佇列進行背景處理後，通常會由佇列 Worker 接受並傳送給預期的收件人。

但是，如果您想在佇列 Worker 處理排入佇列的通知後，最終決定是否應該傳送該通知，您可以在通知類別上定義一個 `shouldSend` 方法。如果此方法返回 `false`，則通知將不會傳送：

    /**
     * 判斷通知是否應該傳送。
     */
    public function shouldSend(object $notifiable, string $channel): bool
    {
        return $this->invoice->isPaid();
    }

<a name="on-demand-notifications"></a>
### 即時通知

有時您可能需要向未儲存為應用程式「使用者」的人傳送通知。使用 `Notification` facade 的 `route` 方法，您可以在傳送通知之前指定臨時的通知路由資訊：

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Support\Facades\Notification;

    Notification::route('mail', 'taylor @example.com')
        ->route('vonage', '5555555555')
        ->route('slack', '#slack-channel')
        ->route('broadcast', [new Channel('channel-name')])
        ->notify(new InvoicePaid($invoice));

如果您想在向 `mail` 路由傳送即時通知時提供收件人的姓名，您可以提供一個陣列，其中包含電子郵件地址作為鍵，姓名作為陣列中第一個元素的值：

    Notification::route('mail', [
        'barrett @example.com' => 'Barrett Blair',
    ])->notify(new InvoicePaid($invoice));

使用 `routes` 方法，您可以一次為多個通知管道提供臨時路由資訊：

    Notification::routes([
        'mail' => ['barrett @example.com' => 'Barrett Blair'],
        'vonage' => '5555555555',
    ])->notify(new InvoicePaid($invoice));

<a name="mail-notifications"></a>
## 郵件通知

<a name="formatting-mail-messages"></a>
### 格式化郵件訊息

如果通知支援以電子郵件形式傳送，您應該在通知類別上定義一個 `toMail` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含一些簡單的方法，可幫助您建構交易電子郵件訊息。郵件訊息可以包含文字行以及「呼籲行動」。讓我們看看一個 `toMail` 方法的範例：

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

> [!NOTE]
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知生成訊息所需的任何資料傳遞到通知的建構函式中。

在此範例中，我們註冊了一個問候語、一行文字、一個呼籲行動，然後是另一行文字。`MailMessage` 物件提供的這些方法使格式化小型交易電子郵件變得簡單快捷。郵件管道隨後會將訊息元件轉換為美觀、響應式的 HTML 電子郵件範本，並附帶純文字對應項。以下是 `mail` 管道生成的電子郵件範例：

<img src="https://laravel.com/img/docs/notification-example-2.png">

> [!NOTE]
> 傳送郵件通知時，請務必在 `config/app.php` 設定檔中設定 `name` 設定選項。此值將用於郵件通知訊息的標頭和頁尾。

<a name="error-messages"></a>
#### 錯誤訊息

有些通知會告知使用者錯誤，例如發票付款失敗。您可以在建構訊息時呼叫 `error` 方法，以指示郵件訊息與錯誤有關。當在郵件訊息上使用 `error` 方法時，呼籲行動按鈕將顯示為紅色而不是黑色：

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

<a name="other-mail-notification-formatting-options"></a>
#### 其他郵件通知格式選項

除了在通知類別中定義文字「行」之外，您還可以使用 `view` 方法指定應用於呈現通知電子郵件的自訂範本：

    /**
     * 取得通知的郵件表示。
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            'mail.invoice.paid', ['invoice' => $this->invoice]
        );
    }

您可以透過將視圖名稱作為陣列的第二個元素傳遞給 `view` 方法，為郵件訊息指定一個純文字視圖：

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

或者，如果您的訊息只有純文字視圖，您可以使用 `text` 方法：

    /**
     * 取得通知的郵件表示。
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->text(
            'mail.invoice.paid-text', ['invoice' => $this->invoice]
        );
    }

<a name="customizing-the-sender"></a>
### 自訂寄件人

預設情況下，電子郵件的寄件人/發件地址在 `config/mail.php` 設定檔中定義。但是，您可以使用 `from` 方法為特定通知指定發件地址：

    /**
     * 取得通知的郵件表示。
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->from('barrett @example.com', 'Barrett Blair')
            ->line('...');
    }

<a name="customizing-the-recipient"></a>
### 自訂收件人

透過 `mail` 管道傳送通知時，通知系統會自動在您的可通知實體上尋找 `email` 屬性。您可以透過在可通知實體上定義 `routeNotificationForMail` 方法來自訂用於傳送通知的電子郵件地址：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * 路由郵件管道的通知。
         *
         * @return  array<string, string>|string
         */
        public function routeNotificationForMail(Notification $notification): array|string
        {
            // 只返回電子郵件地址...
            return $this->email_address;

            // 返回電子郵件地址和姓名...
            return [$this->email_address => $this->name];
        }
    }

<a name="customizing-the-subject"></a>
### 自訂主旨

預設情況下，電子郵件的主旨是通知類別名稱，格式化為「Title Case」。因此，如果您的通知類別名稱為 `InvoicePaid`，則電子郵件的主旨將為 `Invoice Paid`。如果您想為訊息指定不同的主旨，您可以在建構訊息時呼叫 `subject` 方法：

    /**
     * 取得通知的郵件表示。
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Notification Subject')
            ->line('...');
    }

<a name="customizing-the-mailer"></a>
### 自訂郵件發送器

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件發送器傳送。但是，您可以在建構訊息時呼叫 `mailer` 方法，在執行時指定不同的郵件發送器：

    /**
     * 取得通知的郵件表示。
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->mailer('postmark')
            ->line('...');
    }

<a name="customizing-the-templates"></a>
### 自訂範本

您可以透過發布通知套件的資源來修改郵件通知使用的 HTML 和純文字範本。執行此命令後，郵件通知範本將位於 `resources/views/vendor/notifications` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 附件

要將附件新增到電子郵件通知中，請在建構訊息時使用 `attach` 方法。`attach` 方法將檔案的絕對路徑作為其第一個引數：

    /**
     * 取得通知的郵件表示。
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->greeting('Hello!')
            ->attach('/path/to/file');
    }

> [!NOTE]
> 通知郵件訊息提供的 `attach` 方法也接受[可附加物件](/docs/{{version}}/mail#attachable-objects)。請查閱全面的[可附加物件文件](/docs/{{version}}/mail#attachable-objects)以了解更多資訊。

將檔案附加到訊息時，您還可以透過將 `array` 作為第二個引數傳遞給 `attach` 方法來指定顯示名稱和/或 MIME 類型：

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

與在 Mailable 物件中附加檔案不同，您不能使用 `attachFromStorage` 直接從儲存磁碟附加檔案。您應該使用 `attach` 方法，並提供儲存磁碟上檔案的絕對路徑。或者，您可以從 `toMail` 方法返回一個 [Mailable](/docs/{{version}}/mail#generating-mailables)：

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

必要時，可以使用 `attachMany` 方法將多個檔案附加到訊息中：

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

<a name="raw-data-attachments"></a>
#### 原始資料附件

`attachData` 方法可用於將原始位元組字串作為附件。呼叫 `attachData` 方法時，您應該提供應分配給附件的檔案名稱：

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

<a name="adding-tags-metadata"></a>
### 新增標籤與中繼資料

一些第三方電子郵件供應商，例如 Mailgun 和 Postmark，支援訊息「標籤」和「中繼資料」，可用於對您的應用程式傳送的電子郵件進行分組和追蹤。您可以透過 `tag` 和 `metadata` 方法將標籤和中繼資料新增到電子郵件訊息中：

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

如果您的應用程式使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件以獲取有關[標籤](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1)和[中繼資料](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages)的更多資訊。同樣，也可以查閱 Postmark 文件以獲取有關其對[標籤](https://postmarkapp.com/blog/tags-support-for-smtp)和[中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq)的支援的更多資訊。

如果您的應用程式使用 Amazon SES 傳送電子郵件，您應該使用 `metadata` 方法將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包，該閉包將在傳送訊息之前使用 Symfony Message 實例調用。這讓您有機會在訊息傳送之前深度自訂訊息：

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

<a name="using-mailables"></a>
### 使用 Mailables

如有需要，您可以從通知的 `toMail` 方法返回一個完整的 [Mailable 物件](/docs/{{version}}/mail)。當返回 `Mailable` 而不是 `MailMessage` 時，您需要使用 Mailable 物件的 `to` 方法指定訊息收件人：

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

<a name="mailables-and-on-demand-notifications"></a>
#### Mailables 與即時通知

如果您正在傳送[即時通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將是 `Illuminate\Notifications\AnonymousNotifiable` 的實例，它提供了一個 `routeNotificationFor` 方法，可用於檢索即時通知應該傳送到的電子郵件地址：

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

<a name="previewing-mail-notifications"></a>
### 預覽郵件通知

設計郵件通知範本時，方便快速地在瀏覽器中預覽渲染的郵件訊息，就像典型的 Blade 範本一樣。因此，Laravel 允許您直接從路由閉包或控制器返回由郵件通知生成的任何郵件訊息。當返回 `MailMessage` 時，它將在瀏覽器中渲染並顯示，讓您可以快速預覽其設計，而無需將其傳送給實際的電子郵件地址：

    use App\Models\Invoice;
    use App\Notifications\InvoicePaid;

    Route::get('/notification', function () {
        $invoice = Invoice::find(1);

        return (new InvoicePaid($invoice))
            ->toMail($invoice->user);
    });

<a name="markdown-mail-notifications"></a>
## Markdown 郵件通知

Markdown 郵件通知讓您可以利用郵件通知的預建範本，同時讓您有更多自由撰寫更長、更自訂的訊息。由於訊息是以 Markdown 撰寫的，Laravel 能夠為訊息渲染出美觀、響應式的 HTML 範本，同時自動生成純文字對應項。

<a name="generating-the-message"></a>
### 產生訊息

要產生帶有相應 Markdown 範本的通知，您可以使用 `make:notification` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

與所有其他郵件通知一樣，使用 Markdown 範本的通知應在其通知類別上定義一個 `toMail` 方法。但是，不要使用 `line` 和 `action` 方法來建構通知，而是使用 `markdown` 方法來指定應使用的 Markdown 範本名稱。您可以將希望提供給範本的資料陣列作為方法的第二個引數傳遞：

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

<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染一個置中的按鈕連結。該元件接受兩個引數：`url` 和一個可選的 `color`。支援的顏色有 `primary`、`green` 和 `red`。您可以根據需要向通知新增任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```

<a name="panel-component"></a>
#### 面板元件

面板元件會將給定的文字區塊渲染在一個面板中，該面板的背景顏色與通知的其餘部分略有不同。這讓您可以將注意力吸引到給定的文字區塊：

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

此命令會將 Markdown 郵件元件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含一個 `html` 和一個 `text` 目錄，每個目錄都包含每個可用元件的相應表示。您可以隨意自訂這些元件。

<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將自動內聯到 Markdown 通知中的 HTML 表示中。

如果您想為 Laravel 的 Markdown 元件建構一個全新的主題，您可以將 CSS 檔案放置在 `html/themes` 目錄中。命名並儲存您的 CSS 檔案後，更新 `mail` 設定檔的 `theme` 選項以符合您的新主題名稱。

要為個別通知自訂主題，您可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受傳送通知時應使用的主題名稱：

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

<a name="database-notifications"></a>
## 資料庫通知

<a name="database-prerequisites"></a>
### 前置作業

`database` 通知管道將通知資訊儲存在資料庫表格中。此表格將包含通知類型以及描述通知的 JSON 資料結構等資訊。

您可以查詢表格以在應用程式的使用者介面中顯示通知。但是，在此之前，您需要建立一個資料庫表格來儲存您的通知。您可以使用 `make:notifications-table` 命令來產生一個具有正確表格結構的 [migration](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的可通知模型使用 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，您應該在通知表格 migration 中將 `morphs` 方法替換為 [`uuidMorphs`](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [`ulidMorphs`](/docs/{{version}}/migrations#column-method-ulidMorphs)。

<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援儲存在資料庫表格中，您應該在通知類別上定義一個 `toDatabase` 或 `toArray` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個純 PHP 陣列。返回的陣列將被編碼為 JSON 並儲存在 `notifications` 表格的 `data` 欄位中。讓我們看看一個 `toArray` 方法的範例：

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

當通知儲存在應用程式的資料庫中時，`type` 欄位預設將設定為通知的類別名稱，`read_at` 欄位將為 `null`。但是，您可以透過在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來自訂此行為：

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

<a name="todatabase-vs-toarray"></a>
#### `toDatabase` 與 `toArray`

`toArray` 方法也由 `broadcast` 管道使用，以決定要廣播到您的 JavaScript 驅動前端的資料。如果您想為 `database` 和 `broadcast` 管道提供兩種不同的陣列表示，您應該定義一個 `toDatabase` 方法而不是 `toArray` 方法。

<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知儲存在資料庫中，您需要一種方便的方式從您的可通知實體存取它們。`Illuminate\Notifications\Notifiable` trait 包含在 Laravel 的預設 `App\Models\User` 模型中，它包含一個 `notifications` [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)，該關聯返回實體的通知。要獲取通知，您可以像任何其他 Eloquent 關聯一樣存取此方法。預設情況下，通知將按 `created_at` 時間戳記排序，最新的通知位於集合的開頭：

    $user = App\Models\User::find(1);

    foreach ($user->notifications as $notification) {
        echo $notification->type;
    }

如果您只想檢索「未讀」通知，您可以使用 `unreadNotifications` 關聯。同樣，這些通知將按 `created_at` 時間戳記排序，最新的通知位於集合的開頭：

    $user = App\Models\User::find(1);

    foreach ($user->unreadNotifications as $notification) {
        echo $notification->type;
    }

> [!NOTE]
> 要從您的 JavaScript 客戶端存取您的通知，您應該為您的應用程式定義一個通知控制器，該控制器返回可通知實體 (例如當前使用者) 的通知。然後，您可以從您的 JavaScript 客戶端向該控制器的 URL 發出 HTTP 請求。

<a name="marking-notifications-as-read"></a>
### 將通知標記為已讀

通常，當使用者查看通知時，您會希望將其標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，該方法會更新通知資料庫記錄上的 `read_at` 欄位：

    $user = App\Models\User::find(1);

    foreach ($user->unreadNotifications as $notification) {
        $notification->markAsRead();
    }

但是，您可以使用 `markAsRead` 方法直接在通知集合上，而不是遍歷每個通知：

    $user->unreadNotifications->markAsRead();

您還可以使用大量更新查詢將所有通知標記為已讀，而無需從資料庫中檢索它們：

    $user = App\Models\User::find(1);

    $user->unreadNotifications()->update(['read_at' => now()]);

您可以 `delete` 通知以將它們從表格中完全刪除：

    $user->notifications()->delete();

<a name="broadcast-notifications"></a>
## 廣播通知

<a name="broadcast-prerequisites"></a>
### 前置作業

在廣播通知之前，您應該設定並熟悉 Laravel 的[事件廣播](/docs/{{version}}/broadcasting)服務。事件廣播提供了一種從您的 JavaScript 驅動前端對伺服器端 Laravel 事件做出反應的方式。

<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 管道使用 Laravel 的[事件廣播](/docs/{{version}}/broadcasting)服務廣播通知，讓您的 JavaScript 驅動前端能夠即時捕獲通知。如果通知支援廣播，您可以在通知類別上定義一個 `toBroadcast` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `BroadcastMessage` 實例。如果 `toBroadcast` 方法不存在，則將使用 `toArray` 方法來收集應廣播的資料。返回的資料將被編碼為 JSON 並廣播到您的 JavaScript 驅動前端。讓我們看看一個 `toBroadcast` 方法的範例：

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

<a name="broadcast-queue-configuration"></a>
#### 廣播佇列設定

所有廣播通知都排入佇列進行廣播。如果您想設定用於廣播操作的佇列連線或佇列名稱，您可以使用 `BroadcastMessage` 的 `onConnection` 和 `onQueue` 方法：

    return (new BroadcastMessage($data))
        ->onConnection('sqs')
        ->onQueue('broadcasts');

<a name="customizing-the-notification-type"></a>
#### 自訂通知類型

除了您指定的資料之外，所有廣播通知還具有一個 `type` 欄位，其中包含通知的完整類別名稱。如果您想自訂通知 `type`，您可以在通知類別上定義一個 `broadcastType` 方法：

    /**
     * 取得正在廣播的通知類型。
     */
    public function broadcastType(): string
    {
        return 'broadcast.message';
    }

<a name="listening-for-notifications"></a>
### 監聽通知

通知將在一個使用 `{notifiable}.{id}` 慣例格式化的私有管道上廣播。因此，如果您向 ID 為 `1` 的 `App\Models\User` 實例傳送通知，則通知將在 `App.Models.User.1` 私有管道上廣播。使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以使用 `notification` 方法輕鬆監聽管道上的通知：

    Echo.private('App.Models.User.' + userId)
        .notification((notification) => {
            console.log(notification.type);
        });

<a name="customizing-the-notification-channel"></a>
#### 自訂通知管道

如果您想自訂實體的廣播通知在哪個管道上廣播，您可以在可通知實體上定義一個 `receivesBroadcastNotificationsOn` 方法：

    <?php

    namespace App\Models;

    use Illuminate\Broadcasting\PrivateChannel;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * 使用者接收通知廣播的管道。
         */
        public function receivesBroadcastNotificationsOn(): string
        {
            return 'users.'.$this->id;
        }
    }

<a name="sms-notifications"></a>
## SMS 通知

<a name="sms-prerequisites"></a>
### 前置作業

Laravel 中的 SMS 通知傳送由 [Vonage](https://www.vonage.com/) (前身為 Nexmo) 提供支援。在您透過 Vonage 傳送通知之前，您需要安裝 `laravel/vonage-notification-channel` 和 `guzzlehttp/guzzle` 套件：

    composer require laravel/vonage-notification-channel guzzlehttp/guzzle

該套件包含一個[設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。但是，您不需要將此設定檔匯出到您自己的應用程式中。您可以簡單地使用 `VONAGE_KEY` 和 `VONAGE_SECRET` 環境變數來定義您的 Vonage 公鑰和密鑰。

定義您的金鑰後，您應該設定一個 `VONAGE_SMS_FROM` 環境變數，該變數定義您的 SMS 訊息預設應從哪個電話號碼傳送。您可以在 Vonage 控制面板中產生此電話號碼：

    VONAGE_SMS_FROM=15556666666

<a name="formatting-sms-notifications"></a>
### 格式化 SMS 通知

如果通知支援以 SMS 形式傳送，您應該在通知類別上定義一個 `toVonage` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 取得 Vonage / SMS 通知表示。
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->content('Your SMS message content');
    }

<a name="unicode-content"></a>
#### Unicode 內容

如果您的 SMS 訊息將包含 Unicode 字元，您應該在建構 `VonageMessage` 實例時呼叫 `unicode` 方法：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 取得 Vonage / SMS 通知表示。
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->content('Your unicode message')
            ->unicode();
    }

<a name="customizing-the-from-number"></a>
### 自訂「寄件人」號碼

如果您想從與 `VONAGE_SMS_FROM` 環境變數指定的電話號碼不同的電話號碼傳送某些通知，您可以在 `VonageMessage` 實例上呼叫 `from` 方法：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 取得 Vonage / SMS 通知表示。
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->content('Your SMS message content')
            ->from('15554443333');
    }

<a name="adding-a-client-reference"></a>
### 新增客戶參考

如果您想追蹤每個使用者、團隊或客戶的成本，您可以將「客戶參考」新增到通知中。Vonage 將允許您使用此客戶參考產生報告，以便您更好地了解特定客戶的 SMS 使用情況。客戶參考可以是任何長度不超過 40 個字元的字串：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * 取得 Vonage / SMS 通知表示。
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->clientReference((string) $notifiable->id)
            ->content('Your SMS message content');
    }

<a name="routing-sms-notifications"></a>
### 路由 SMS 通知

要將 Vonage 通知路由到正確的電話號碼，請在您的可通知實體上定義一個 `routeNotificationForVonage` 方法：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * 路由 Vonage 管道的通知。
         */
        public function routeNotificationForVonage(Notification $notification): string
        {
            return $this->phone_number;
        }
    }

<a name="slack-notifications"></a>
## Slack 通知

<a name="slack-prerequisites"></a>
### 前置作業

在傳送 Slack 通知之前，您應該透過 Composer 安裝 Slack 通知管道：

```shell
composer require laravel/slack-notification-channel
```

此外，您必須為您的 Slack 工作區建立一個 [Slack App](https://api.slack.com/apps?new_app=1)。

如果您只需要將通知傳送給建立 App 的同一個 Slack 工作區，您應該確保您的 App 具有 `chat:write`、`chat:write.public` 和 `chat:write.customize` 範圍。如果您想以您的 Slack App 傳送訊息，您應該確保您的 App 也具有 `chat:write:bot` 範圍。這些範圍可以從 Slack 中「OAuth & Permissions」App 管理標籤新增。

接下來，複製 App 的「Bot User OAuth Token」並將其放置在您應用程式 `services.php` 設定檔中的 `slack` 設定陣列中。此 Token 可以在 Slack 中「OAuth & Permissions」標籤上找到：

    'slack' => [
        'notifications' => [
            'bot_user_oauth_token' => env('SLACK_BOT_USER_OAUTH_TOKEN'),
            'channel' => env('SLACK_BOT_USER_DEFAULT_CHANNEL'),
        ],
    ],

<a name="slack-app-distribution"></a>
#### App 分發

如果您的應用程式將向您的應用程式使用者擁有的外部 Slack 工作區傳送通知，您將需要透過 Slack「分發」您的 App。App 分發可以從 Slack 中 App 的「Manage Distribution」標籤進行管理。一旦您的 App 已分發，您可以使用 [Socialite](/docs/{{version}}/socialite) [代表您的應用程式使用者取得 Slack Bot Token](/docs/{{version}}/socialite#slack-bot-scopes)。

<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援以 Slack 訊息形式傳送，您應該在通知類別上定義一個 `toSlack` 方法。此方法將接收一個 `$notifiable` 實體，並應返回一個 `Illuminate\Notifications\Slack\SlackMessage` 實例。您可以使用 [Slack 的 Block Kit API](https://api.slack.com/block-kit) 建構豐富的通知。以下範例可以在 [Slack 的 Block Kit Builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor @laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中預覽：

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
                $block->field("*Invoice No:*\n1000")->markdown();
                $block->field("*Invoice Recipient:*\ntaylor @laravel.com")->markdown();
            })
            ->dividerBlock()
            ->sectionBlock(function (SectionBlock $block) {
                $block->text('Congratulations!');
            });
    }

<a name="using-slacks-block-kit-builder-template"></a>
#### 使用 Slack 的 Block Kit Builder 範本

除了使用流暢的訊息建構器方法來建構您的 Block Kit 訊息之外，您還可以將 Slack 的 Block Kit Builder 生成的原始 JSON 負載提供給 `usingBlockKitTemplate` 方法：

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

<a name="slack-interactivity"></a>
### Slack 互動性

Slack 的 Block Kit 通知系統提供了強大的功能來[處理使用者互動](https://api.slack.com/interactivity/handling)。要利用這些功能，您的 Slack App 應該啟用「Interactivity」並設定一個指向您的應用程式提供的 URL 的「Request URL」。這些設定可以在 Slack 中「Interactivity & Shortcuts」App 管理標籤進行管理。

在以下範例中，該範例使用了 `actionsBlock` 方法，Slack 將向您的「Request URL」傳送一個 `POST` 請求，其中包含一個負載，其中包含點擊按鈕的 Slack 使用者、點擊按鈕的 ID 等。您的應用程式隨後可以根據負載決定要採取的動作。您還應該[驗證請求](https://api.slack.com/authentication/verifying-requests-from-slack)是由 Slack 發出的：

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
                 // ID 預設為 "button_acknowledge_invoice"...
                $block->button('Acknowledge Invoice')->primary();

                // 手動設定 ID...
                $block->button('Deny')->danger()->id('deny_invoice');
            });
    }

<a name="slack-confirmation-modals"></a>
#### 確認對話框

如果您希望使用者在執行動作之前必須確認，您可以在定義按鈕時調用 `confirm` 方法。`confirm` 方法接受一個訊息和一個接收 `ConfirmObject` 實例的閉包：

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

<a name="inspecting-slack-blocks"></a>
#### 檢查 Slack 區塊

如果您想快速檢查您正在建構的區塊，您可以在 `SlackMessage` 實例上調用 `dd` 方法。`dd` 方法將生成並轉儲一個 URL 到 Slack 的 [Block Kit Builder](https://app.slack.com/block-kit-builder/)，該工具會在您的瀏覽器中顯示負載和通知的預覽。您可以將 `true` 傳遞給 `dd` 方法以轉儲原始負載：

    return (new SlackMessage)
            ->text('One of your invoices has been paid!')
            ->headerBlock('Invoice Paid')
            ->dd();

<a name="routing-slack-notifications"></a>
### 路由 Slack 通知

要將 Slack 通知導向適當的 Slack 團隊和管道，請在您的可通知模型上定義一個 `routeNotificationForSlack` 方法。此方法可以返回三個值之一：

- `null` - 這會將路由延遲到通知本身中設定的管道。您可以在建構 `SlackMessage` 時使用 `to` 方法在通知中設定管道。
- 指定要傳送通知的 Slack 管道的字串，例如 `#support-channel`。
- `SlackRoute` 實例，它允許您指定 OAuth Token 和管道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應用於向外部工作區傳送通知。

例如，從 `routeNotificationForSlack` 方法返回 `#support-channel` 將會將通知傳送給與您應用程式 `services.php` 設定檔中 Bot User OAuth Token 相關聯的工作區中的 `#support-channel` 管道：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * 路由 Slack 管道的通知。
         */
        public function routeNotificationForSlack(Notification $notification): mixed
        {
            return '#support-channel';
        }
    }

<a name="notifying-external-slack-workspaces"></a>
### 通知外部 Slack 工作區

> [!NOTE]
> 在向外部 Slack 工作區傳送通知之前，您的 Slack App 必須[分發](#slack-app-distribution)。

當然，您通常會希望將通知傳送給您的應用程式使用者擁有的 Slack 工作區。為此，您首先需要為使用者取得一個 Slack OAuth Token。幸運的是，[Laravel Socialite](/docs/{{version}}/socialite) 包含一個 Slack 驅動程式，它將允許您輕鬆地使用 Slack 驗證您的應用程式使用者並[取得 Bot Token](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您取得了 Bot Token 並將其儲存在您的應用程式資料庫中，您就可以使用 `SlackRoute::make` 方法將通知路由到使用者的工作區。此外，您的應用程式可能需要提供一個機會讓使用者指定通知應該傳送到的管道：

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
         * 路由 Slack 管道的通知。
         */
        public function routeNotificationForSlack(Notification $notification): mixed
        {
            return SlackRoute::make($this->slack_channel, $this->slack_token);
        }
    }

<a name="localizing-notifications"></a>
## 通知本地化

Laravel 允許您以 HTTP 請求當前語系以外的語系傳送通知，即使通知已排入佇列，它也會記住此語系。

為此，`Illuminate\Notifications\Notification` 類別提供了一個 `locale` 方法來設定所需的語言。當通知正在評估時，應用程式將切換到此語系，然後在評估完成後恢復到先前的語系：

    $user->notify((new InvoicePaid($invoice))->locale('es'));

多個可通知實體的本地化也可以透過 `Notification` facade 實現：

    Notification::locale('es')->send(
        $users, new InvoicePaid($invoice)
    );

<a name="user-preferred-locales"></a>
### 使用者偏好語系

有時，應用程式會儲存每個使用者的偏好語系。透過在您的可通知模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在傳送通知時使用此儲存的語系：

    use Illuminate\Contracts\Translation\HasLocalePreference;

    class User extends Model implements HasLocalePreference
    {
        /**
         * 取得使用者的偏好語系。
         */
        public function preferredLocale(): string
        {
            return $this->locale;
        }
    }

一旦您實作了介面，Laravel 將在向模型傳送通知和 Mailable 時自動使用偏好語系。因此，在使用此介面時無需呼叫 `locale` 方法：

    $user->notify(new InvoicePaid($invoice));

<a name="testing"></a>
## 測試

您可以使用 `Notification` facade 的 `fake` 方法來防止通知被傳送。通常，傳送通知與您實際測試的程式碼無關。最有可能的是，只需斷言 Laravel 已被指示傳送給定的通知就足夠了。

呼叫 `Notification` facade 的 `fake` 方法後，您可以斷言通知已被指示傳送給使用者，甚至檢查通知收到的資料：

```php tab=Pest
<?php

use App\Notifications\OrderShipped;
use Illuminate\Support\Facades\Notification;

test('orders can be shipped', function () {
    Notification::fake();

    // Perform order shipping...

    // Assert that no notifications were sent...
    Notification::assertNothingSent();

    // Assert a notification was sent to the given users...
    Notification::assertSentTo(
        [$user], OrderShipped::class
    );

    // Assert a notification was not sent...
    Notification::assertNotSentTo(
        [$user], AnotherNotification::class
    );

    // Assert that a given number of notifications were sent...
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

        // Perform order shipping...

        // Assert that no notifications were sent...
        Notification::assertNothingSent();

        // Assert a notification was sent to the given users...
        Notification::assertSentTo(
            [$user], OrderShipped::class
        );

        // Assert a notification was not sent...
        Notification::assertNotSentTo(
            [$user], AnotherNotification::class
        );

        // Assert that a given number of notifications were sent...
        Notification::assertCount(3);
    }
}
```

您可以將閉包傳遞給 `assertSentTo` 或 `assertNotSentTo` 方法，以斷言已傳送通過給定「真實性測試」的通知。如果至少有一個通知通過了給定的真實性測試，則斷言將成功：

    Notification::assertSentTo(
        $user,
        function (OrderShipped $notification, array $channels) use ($order) {
            return $notification->order->id === $order->id;
        }
    );

<a name="on-demand-notifications"></a>
#### 即時通知

如果您正在測試的程式碼傳送[即時通知](#on-demand-notifications)，您可以透過 `assertSentOnDemand` 方法測試即時通知是否已傳送：

    Notification::assertSentOnDemand(OrderShipped::class);

透過將閉包作為第二個引數傳遞給 `assertSentOnDemand` 方法，您可以判斷即時通知是否已傳送給正確的「路由」地址：

    Notification::assertSentOnDemand(
        OrderShipped::class,
        function (OrderShipped $notification, array $channels, object $notifiable) use ($user) {
            return $notifiable->routes['mail'] === $user->email;
        }
    );

<a name="notification-events"></a>
## 通知事件

<a name="notification-sending-event"></a>
#### 通知傳送事件

當通知正在傳送時，通知系統會分派 `Illuminate\Notifications\Events\NotificationSending` 事件。此事件包含「可通知」實體和通知實例本身。您可以在應用程式中為此事件建立[事件監聽器](/docs/{{version}}/events)：

    use Illuminate\Notifications\Events\NotificationSending;

    class CheckNotificationStatus
    {
        /**
         * 處理給定的事件。
         */
        public function handle(NotificationSending $event): void
        {
            // ...
        }
    }

如果 `NotificationSending` 事件的事件監聽器從其 `handle` 方法返回 `false`，則通知將不會傳送：

    /**
     * 處理給定的事件。
     */
    public function handle(NotificationSending $event): bool
    {
        return false;
    }

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification` 和 `channel` 屬性，以了解有關通知收件人或通知本身的更多資訊：

    /**
     * 處理給定的事件。
     */
    public function handle(NotificationSending $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
    }

<a name="notification-sent-event"></a>
#### 通知已傳送事件

當通知傳送後，通知系統會分派 `Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events)。此事件包含「可通知」實體和通知實例本身。您可以在應用程式中為此事件建立[事件監聽器](/docs/{{version}}/events)：

    use Illuminate\Notifications\Events\NotificationSent;

    class LogNotification
    {
        /**
         * 處理給定的事件。
         */
        public function handle(NotificationSent $event): void
        {
            // ...
        }
    }

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification`、`channel` 和 `response` 屬性，以了解有關通知收件人或通知本身的更多資訊：

    /**
     * 處理給定的事件。
     */
    public function handle(NotificationSent $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
        // $event->response
    }

<a name="custom-channels"></a>
## 自訂管道

Laravel 附帶了一些通知管道，但您可能希望編寫自己的驅動程式以透過其他管道傳送通知。Laravel 使其變得簡單。首先，定義一個包含 `send` 方法的類別。該方法應接收兩個引數：`$notifiable` 和 `$notification`。

在 `send` 方法中，您可以呼叫通知上的方法以檢索您的管道理解的訊息物件，然後以您希望的任何方式將通知傳送給 `$notifiable` 實例：

    <?php

    namespace App\Notifications;

    use Illuminate\Notifications\Notification;

    class VoiceChannel
    {
        /**
         * 傳送給定的通知。
         */
        public function send(object $notifiable, Notification $notification): void
        {
            $message = $notification->toVoice($notifiable);

            // 將通知傳送給 $notifiable 實例...
        }
    }

一旦您的通知管道類別已定義，您可以從任何通知的 `via` 方法返回類別名稱。在此範例中，您的通知的 `toVoice` 方法可以返回您選擇的任何物件來表示語音訊息。例如，您可以定義自己的 `VoiceMessage` 類別來表示這些訊息：

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
         * 取得通知管道。
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
