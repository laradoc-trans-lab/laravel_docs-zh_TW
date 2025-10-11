# 通知

- [簡介](#introduction)
- [生成通知](#generating-notifications)
- [傳送通知](#sending-notifications)
    - [使用 Notifiable Trait](#using-the-notifiable-trait)
    - [使用 Notification Facade](#using-the-notification-facade)
    - [指定傳送通道](#specifying-delivery-channels)
    - [佇列通知](#queueing-notifications)
    - [即時通知](#on-demand-notifications)
- [郵件通知](#mail-notifications)
    - [格式化郵件訊息](#formatting-mail-messages)
    - [自訂寄件者](#customizing-the-sender)
    - [自訂收件者](#customizing-the-recipient)
    - [自訂主旨](#customizing-the-subject)
    - [自訂郵件服務](#customizing-the-mailer)
    - [自訂樣板](#customizing-the-templates)
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
    - [必要條件](#database-prerequisites)
    - [格式化資料庫通知](#formatting-database-notifications)
    - [存取通知](#accessing-the-notifications)
    - [標記通知為已讀](#marking-notifications-as-read)
- [廣播通知](#broadcast-notifications)
    - [必要條件](#broadcast-prerequisites)
    - [格式化廣播通知](#formatting-broadcast-notifications)
    - [監聽通知](#listening-for-notifications)
- [簡訊通知](#sms-notifications)
    - [必要條件](#sms-prerequisites)
    - [格式化簡訊通知](#formatting-sms-notifications)
    - [Unicode 內容](#unicode-content)
    - [自訂「寄件者」號碼](#customizing-the-from-number)
    - [新增客戶參考](#adding-a-client-reference)
    - [簡訊通知路由](#routing-sms-notifications)
- [Slack 通知](#slack-notifications)
    - [必要條件](#slack-prerequisites)
    - [格式化 Slack 通知](#formatting-slack-notifications)
    - [Slack 互動性](#slack-interactivity)
    - [Slack 通知路由](#routing-slack-notifications)
    - [通知外部 Slack 工作區](#notifying-external-slack-workspaces)
- [通知本地化](#localizing-notifications)
- [測試](#testing)
- [通知事件](#notification-events)
- [自訂通道](#custom-channels)

<a name="introduction"></a>
## 簡介

除了支援[傳送電子郵件](/docs/{{version}}/mail)外，Laravel 還支援透過各種傳送通道傳送通知，包括電子郵件、簡訊 (透過 [Vonage](https://www.vonage.com/communications-apis/)，前身為 Nexmo)，以及 [Slack](https://slack.com)。此外，還有各種[社群建立的通知通道](https://laravel-notification-channels.com/about/#suggesting-a-new-channel)被建立，用於透過數十種不同的通道傳送通知！通知也可以儲存於資料庫中，以便在您的網頁介面中顯示。

通常，通知應該是簡短且提供資訊的訊息，用於告知使用者在應用程式中發生了什麼。例如，如果您正在撰寫一個帳務應用程式，您可能會透過郵件和簡訊通道向使用者傳送一則「帳單已支付」的通知。

<a name="generating-notifications"></a>
## 生成通知

在 Laravel 中，每個通知都由一個單一類別表示，通常儲存於 `app/Notifications` 目錄中。如果您在應用程式中沒有看到此目錄，請不用擔心 – 當您執行 `make:notification` Artisan 指令時，它將會為您建立：

```shell
php artisan make:notification InvoicePaid
```

此指令會在您的 `app/Notifications` 目錄中放置一個全新的通知類別。每個通知類別都包含一個 `via` 方法以及數量不等的訊息建構方法，例如 `toMail` 或 `toDatabase`，這些方法會將通知轉換為針對特定通道量身訂做的訊息。

<a name="sending-notifications"></a>
## 傳送通知

<a name="using-the-notifiable-trait"></a>
### 使用 Notifiable Trait

通知可以透過兩種方式傳送：使用 `Notifiable` Trait 的 `notify` 方法，或是使用 `Notification` [Facade](/docs/{{version}}/facades)。`Notifiable` Trait 預設包含在您的應用程式的 `App\Models\User` model 中：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable
    {
        use Notifiable;
    }

此 Trait 提供的 `notify` 方法預期會接收一個通知實例：

    use App\Notifications\InvoicePaid;

    $user->notify(new InvoicePaid($invoice));

> [!NOTE]
> 請記住，您可以在任何 model 上使用 `Notifiable` Trait。您不僅限於只能將其包含在 `User` model 中。

<a name="using-the-notification-facade"></a>
### 使用 Notification Facade

或者，您可以透過 `Notification` [Facade](/docs/{{version}}/facades) 傳送通知。當您需要向多個可通知實體（例如一組使用者）傳送通知時，此方法很有用。若要使用 Facade 傳送通知，請將所有可通知實體和通知實例傳遞給 `send` 方法：

    use Illuminate\Support\Facades\Notification;

    Notification::send($users, new InvoicePaid($invoice));

您也可以使用 `sendNow` 方法立即傳送通知。即使通知實作了 `ShouldQueue` 介面，此方法也會立即傳送通知：

    Notification::sendNow($developers, new DeploymentCompleted($deployment));

<a name="specifying-delivery-channels"></a>
### 指定傳送通道

每個通知類別都有一個 `via` 方法，用於判斷通知將透過哪些通道傳送。通知可以透過 `mail`、`database`、`broadcast`、`vonage` 和 `slack` 通道傳送。

> [!NOTE]
> 如果您想使用其他傳送通道，例如 Telegram 或 Pusher，請查看社群驅動的 [Laravel 通知通道網站](http://laravel-notification-channels.com)。

該 `via` 方法會接收一個 `$notifiable` 實例，該實例將是通知傳送目標類別的一個實例。您可以使用 `$notifiable` 來判斷通知應該透過哪些通道傳送：

    /**
     * Get the notification's delivery channels.
     *
     * @return array<int, string>
     */
    public function via(object $notifiable): array
    {
        return $notifiable->prefers_sms ? ['vonage'] : ['mail', 'database'];
    }

<a name="queueing-notifications"></a>
### 佇列通知

> [!WARNING]
> 在佇列通知之前，您應該設定佇列並[啟動一個 worker](/docs/{{version}}/queues#running-the-queue-worker)。

傳送通知可能需要一些時間，特別是當通道需要進行外部 API 呼叫來傳送通知時。為了加速應用程式的回應時間，你可以將你的通知進行佇列，方法是將 `ShouldQueue` 介面和 `Queueable` trait 加入你的類別。使用 `make:notification` 命令生成的所有通知類別，都已自動匯入此介面和 trait，因此你可以立即將它們加入你的通知類別：

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

一旦 `ShouldQueue` 介面已加入你的通知中，你就可以正常傳送通知。Laravel 會自動偵測類別上的 `ShouldQueue` 介面，並自動將通知傳送進行佇列：

    $user->notify(new InvoicePaid($invoice));

當通知進行佇列時，每個收件者和通道的組合都會建立一個佇列工作。例如，如果你的通知有三個收件者和兩個通道，將會向佇列分派六個工作。

<a name="delaying-notifications"></a>
#### 延遲通知

如果你想要延遲通知的傳送，你可以將 `delay` 方法串接到你的通知實例上：

    $delay = now()->addMinutes(10);

    $user->notify((new InvoicePaid($invoice))->delay($delay));

你可以傳入陣列給 `delay` 方法，以指定特定通道的延遲時間：

    $user->notify((new InvoicePaid($invoice))->delay([
        'mail' => now()->addMinutes(5),
        'sms' => now()->addMinutes(10),
    ]));

此外，你可以在通知類別本身定義一個 `withDelay` 方法。`withDelay` 方法應回傳一個包含通道名稱和延遲時間的陣列：

    /**
     * Determine the notification's delivery delay.
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

預設情況下，佇列通知會使用應用程式的預設佇列連線。如果你想要為特定的通知指定不同的連線，你可以在通知的建構子中呼叫 `onConnection` 方法：

    <?php

    namespace App\Notifications;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        /**
         * Create a new notification instance.
         */
        public function __construct()
        {
            $this->onConnection('redis');
        }
    }

或者，如果你想要為通知支援的每個通知通道指定特定的佇列連線，你可以在通知上定義一個 `viaConnections` 方法。這個方法應回傳一個包含通道名稱與佇列連線名稱配對的陣列：

    /**
     * Determine which connections should be used for each notification channel.
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
#### 自訂通知通道佇列

如果你想要為通知支援的每個通知通道指定特定的佇列，你可以在通知上定義一個 `viaQueues` 方法。這個方法應回傳一個包含通道名稱與佇列名稱配對的陣列：

    /**
     * Determine which queues should be used for each notification channel.
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
#### 佇列通知中介層

佇列通知可以定義中介層 (middleware)，[就像佇列工作](/docs/{{version}}/queues#job-middleware)一樣。首先，在你的通知類別上定義一個 `middleware` 方法。`middleware` 方法會接收 `$notifiable` 和 `$channel` 變數，這允許你根據通知的目的地自訂回傳的中介層：

    use Illuminate\Queue\Middleware\RateLimited;

    /**
     * Get the middleware the notification job should pass through.
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
#### 佇列通知與資料庫交易

當佇列通知在資料庫交易中被分派時，它們可能會在資料庫交易提交之前被佇列處理。當這種情況發生時，你在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果你的通知依賴於這些模型，當處理傳送佇列通知的工作時，可能會發生意外錯誤。

如果你的佇列連線的 `after_commit` 設定選項為 `false`，你仍然可以透過呼叫 `afterCommit` 方法來指示在所有未決資料庫交易提交之後才分派特定的佇列通知：

    use App\Notifications\InvoicePaid;

    $user->notify((new InvoicePaid($invoice))->afterCommit());

此外，你可以在通知的建構子中呼叫 `afterCommit` 方法：

    <?php

    namespace App\Notifications;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Notifications\Notification;

    class InvoicePaid extends Notification implements ShouldQueue
    {
        use Queueable;

        /**
         * Create a new notification instance.
         */
        public function __construct()
        {
            $this->afterCommit();
        }
    }

> [!NOTE]
> 要了解更多關於解決這些問題的資訊，請查閱關於[佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="determining-if-the-queued-notification-should-be-sent"></a>
#### 判斷是否應傳送佇列通知

當佇列通知已被分派到佇列進行背景處理後，它通常會被佇列 worker 接受並傳送給預期的收件者。

然而，如果你想要在佇列 worker 處理佇列通知之後，才最終決定是否傳送該通知，你可以在通知類別上定義一個 `shouldSend` 方法。如果這個方法回傳 `false`，通知將不會被傳送：

    /**
     * Determine if the notification should be sent.
     */
    public function shouldSend(object $notifiable, string $channel): bool
    {
        return $this->invoice->isPaid();
    }

<a name="on-demand-notifications"></a>
### 即時通知

有時，您可能需要向未儲存為應用程式「使用者」的人傳送通知。使用 `Notification` Facade 的 `route` 方法，您可以在傳送通知前指定臨時的通知路由資訊：

    use Illuminate\Broadcasting\Channel;
    use Illuminate\Support\Facades\Notification;

    Notification::route('mail', 'taylor@example.com')
        ->route('vonage', '5555555555')
        ->route('slack', '#slack-channel')
        ->route('broadcast', [new Channel('channel-name')])
        ->notify(new InvoicePaid($invoice));

如果您想在向 `mail` 路由傳送即時通知時提供收件者姓名，您可以提供一個陣列，其中包含電子郵件地址作為鍵，姓名作為陣列中第一個元素的值：

    Notification::route('mail', [
        'barrett@example.com' => 'Barrett Blair',
    ])->notify(new InvoicePaid($invoice));

使用 `routes` 方法，您可以一次為多個通知通道提供臨時的路由資訊：

    Notification::routes([
        'mail' => ['barrett@example.com' => 'Barrett Blair'],
        'vonage' => '5555555555',
    ])->notify(new InvoicePaid($invoice));

<a name="mail-notifications"></a>
## 郵件通知

<a name="formatting-mail-messages"></a>
### 格式化郵件訊息

如果通知支援以電子郵件傳送，您應該在通知類別上定義一個 `toMail` 方法。這個方法會接收一個 `$notifiable` 實體，並應回傳一個 `Illuminate\Notifications\Messages\MailMessage` 實例。

`MailMessage` 類別包含幾個簡單的方法，可幫助您建構事務性電子郵件訊息。郵件訊息可以包含文字行以及「行動呼籲」。讓我們看看一個 `toMail` 方法的範例：

    /**
     * Get the mail representation of the notification.
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
> 請注意，我們在 `toMail` 方法中使用了 `$this->invoice->id`。您可以將通知生成訊息所需的任何資料傳遞給通知的建構子。

在這個範例中，我們設定了一個問候語、一行文字、一個行動呼籲，然後是另一行文字。`MailMessage` 物件提供的這些方法使得格式化小型事務性電子郵件變得簡單快速。郵件通道隨後會將訊息元件轉換為美觀、響應式的 HTML 電子郵件樣板，並附帶一個純文字對應版本。以下是 `mail` 通道產生的一封電子郵件範例：

<img src="https://laravel.com/img/docs/notification-example-2.png" alt="一張範例圖片">

> [!NOTE]
> 傳送郵件通知時，請務必在 `config/app.php` 設定檔中設定 `name` 設定選項。此值將用於您的郵件通知訊息的頁首和頁尾。

<a name="error-messages"></a>
#### 錯誤訊息

有些通知會告知使用者錯誤，例如發票付款失敗。您可以在建構訊息時呼叫 `error` 方法，以指示郵件訊息與錯誤有關。在郵件訊息上使用 `error` 方法時，行動呼籲按鈕將會是紅色而不是黑色：

    /**
     * Get the mail representation of the notification.
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

您可以使用 `view` 方法來指定自訂樣板，以呈現通知電子郵件，而非在通知類別中定義文字「行」：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            'mail.invoice.paid', ['invoice' => $this->invoice]
        );
    }

您可以透過將視圖名稱作為傳遞給 `view` 方法的陣列的第二個元素，來為郵件訊息指定一個純文字視圖：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->view(
            ['mail.invoice.paid', 'mail.invoice.paid-text'],
            ['invoice' => $this->invoice]
        );
    }

或者，如果您的訊息只有一個純文字視圖，您可以使用 `text` 方法：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)->text(
            'mail.invoice.paid-text', ['invoice' => $this->invoice]
        );
    }

<a name="customizing-the-sender"></a>
### 自訂寄件者

預設情況下，電子郵件的寄件者/寄件地址在 `config/mail.php` 設定檔中定義。不過，您可以使用 `from` 方法為特定通知指定寄件地址：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->from('barrett@example.com', 'Barrett Blair')
            ->line('...');
    }

<a name="customizing-the-recipient"></a>
### 自訂收件者

當透過 `mail` 通道傳送通知時，通知系統會自動在您的 notifiable 實體上尋找 `email` 屬性。您可以透過在 notifiable 實體上定義 `routeNotificationForMail` 方法，來自訂用於傳送通知的電子郵件地址：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * Route notifications for the mail channel.
         *
         * @return  array<string, string>|string
         */
        public function routeNotificationForMail(Notification $notification): array|string
        {
            // Return email address only...
            return $this->email_address;

            // Return email address and name...
            return [$this->email_address => $this->name];
        }
    }

<a name="customizing-the-subject"></a>
### 自訂主旨

預設情況下，電子郵件的主旨是通知的類別名稱，並格式化為「首字大寫」。因此，如果您的通知類別名為 `InvoicePaid`，電子郵件的主旨將是 `Invoice Paid`。如果您想為訊息指定不同的主旨，可以在建構訊息時呼叫 `subject` 方法：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('Notification Subject')
            ->line('...');
    }

<a name="customizing-the-mailer"></a>
### 自訂郵件服務

預設情況下，電子郵件通知將使用 `config/mail.php` 設定檔中定義的預設郵件服務傳送。然而，您可以在執行時，透過在建構訊息時呼叫 `mailer` 方法來指定不同的郵件服務：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->mailer('postmark')
            ->line('...');
    }

<a name="customizing-the-templates"></a>
### 自訂樣板

您可以透過發佈通知套件的資源來修改郵件通知所使用的 HTML 和純文字樣板。執行此指令後，郵件通知樣板將位於 `resources/views/vendor/notifications` 目錄中：

```shell
php artisan vendor:publish --tag=laravel-notifications
```

<a name="mail-attachments"></a>
### 附件

要為電子郵件通知添加附件，請在構建訊息時使用 `attach` 方法。`attach` 方法接受文件的絕對路徑作為其第一個參數：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->greeting('Hello!')
            ->attach('/path/to/file');
    }

> [!NOTE]
> 通知郵件訊息提供的 `attach` 方法也接受 [可附加物件](/docs/{{version}}/mail#attachable-objects)。請查閱完整的 [可附加物件文件](/docs/{{version}}/mail#attachable-objects) 以了解更多資訊。

當將文件附加到訊息時，您也可以透過將 `array` 作為第二個參數傳遞給 `attach` 方法來指定顯示名稱和/或 MIME 類型：

    /**
     * Get the mail representation of the notification.
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

與在 Mailable 物件中附加文件不同，您不能直接使用 `attachFromStorage` 從儲存磁碟附加文件。您應該使用 `attach` 方法並提供儲存磁碟上文件的絕對路徑。或者，您可以從 `toMail` 方法回傳一個 [Mailable 物件](/docs/{{version}}/mail#generating-mailables)：

    use App\Mail\InvoicePaid as InvoicePaidMailable;

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): Mailable
    {
        return (new InvoicePaidMailable($this->invoice))
            ->to($notifiable->email)
            ->attachFromStorage('/path/to/file');
    }

必要時，可以使用 `attachMany` 方法將多個文件附加到訊息：

    /**
     * Get the mail representation of the notification.
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

`attachData` 方法可用於將原始位元組字串作為附件附加。呼叫 `attachData` 方法時，您應該提供要指定給附件的文件名：

    /**
     * Get the mail representation of the notification.
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

一些第三方電子郵件服務商（例如 Mailgun 和 Postmark）支援訊息 "tags" 和 "metadata"，可用於對應用程式發送的電子郵件進行分組和追蹤。您可以透過 `tag` 和 `metadata` 方法將標籤和中繼資料添加到電子郵件訊息中：

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->greeting('Comment Upvoted!')
            ->tag('upvote')
            ->metadata('comment_id', $this->comment->id);
    }

如果您的應用程式使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件以了解更多關於 [標籤](https://documentation.mailgun.com/en/latest/user_manual.html#tagging-1) 和 [中繼資料](https://documentation.mailgun.com/en/latest/user_manual.html#attaching-data-to-messages) 的資訊。同樣地，也可以查閱 Postmark 文件以了解更多關於其對 [標籤](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 的支援。

如果您的應用程式使用 Amazon SES 來發送電子郵件，您應該使用 `metadata` 方法將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

`MailMessage` 類別的 `withSymfonyMessage` 方法允許您註冊一個閉包，該閉包將在發送訊息之前與 Symfony 訊息實例一起調用。這使您有機會在訊息傳送之前深度自訂訊息：

    use Symfony\Component\Mime\Email;

    /**
     * Get the mail representation of the notification.
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

如有需要，您可以從通知的 `toMail` 方法回傳一個完整的 [Mailable 物件](/docs/{{version}}/mail)。當回傳 `Mailable` 而不是 `MailMessage` 時，您需要使用 Mailable 物件的 `to` 方法來指定訊息收件人：

    use App\Mail\InvoicePaid as InvoicePaidMailable;
    use Illuminate\Mail\Mailable;

    /**
     * Get the mail representation of the notification.
     */
    public function toMail(object $notifiable): Mailable
    {
        return (new InvoicePaidMailable($this->invoice))
            ->to($notifiable->email);
    }

<a name="mailables-and-on-demand-notifications"></a>
#### Mailables 與即時通知

如果您正在傳送 [即時通知](#on-demand-notifications)，傳遞給 `toMail` 方法的 `$notifiable` 實例將是 `Illuminate\Notifications\AnonymousNotifiable` 的一個實例，它提供了一個 `routeNotificationFor` 方法，可用於檢索即時通知應發送到的電子郵件地址：

    use App\Mail\InvoicePaid as InvoicePaidMailable;
    use Illuminate\Notifications\AnonymousNotifiable;
    use Illuminate\Mail\Mailable;

    /**
     * Get the mail representation of the notification.
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

在設計郵件通知樣板時，可以在瀏覽器中像典型的 Blade 樣板一樣快速預覽渲染後的郵件訊息，這很方便。因此，Laravel 允許您直接從路由閉包或控制器回傳由郵件通知生成的任何郵件訊息。當回傳 `MailMessage` 時，它將在瀏覽器中渲染並顯示，讓您可以快速預覽其設計，而無需將其發送到實際的電子郵件地址：

    use App\Models\Invoice;
    use App\Notifications\InvoicePaid;

    Route::get('/notification', function () {
        $invoice = Invoice::find(1);

        return (new InvoicePaid($invoice))
            ->toMail($invoice->user);
    });

<a name="markdown-mail-notifications"></a>
## Markdown 郵件通知

Markdown 郵件通知讓您可以利用預先建置好的郵件通知樣板，同時賦予您更多自由撰寫更長、更客製化的訊息。由於訊息是以 Markdown 撰寫，Laravel 能夠為訊息渲染出美觀、響應式的 HTML 樣板，並同時自動生成純文字版本。

<a name="generating-the-message"></a>
### 生成訊息

若要生成帶有對應 Markdown 樣板的通知，您可以使用 `make:notification` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:notification InvoicePaid --markdown=mail.invoice.paid
```

與所有其他郵件通知一樣，使用 Markdown 樣板的通知應該在其通知類別上定義 `toMail` 方法。然而，您不需使用 `line` 和 `action` 方法來建構通知，而是使用 `markdown` 方法來指定應使用的 Markdown 樣板名稱。您希望提供給樣板的資料陣列可以作為該方法的第二個引數傳遞：

    /**
     * Get the mail representation of the notification.
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

Markdown 郵件通知結合了 Blade 元件和 Markdown 語法，讓您可以輕鬆建構通知，同時利用 Laravel 預先製作好的通知元件：

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

按鈕元件會渲染出一個置中的按鈕連結。該元件接受兩個引數：一個 `url` 和一個可選的 `color`。支援的顏色有 `primary`、`green` 和 `red`。您可以在一個通知中添加任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="green">
View Invoice
</x-mail::button>
```

<a name="panel-component"></a>
#### 面板元件

面板元件會在一個背景顏色與通知其餘部分略有不同的面板中渲染給定的文字區塊。這讓您可以將注意力吸引到特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### 表格元件

表格元件讓您可以將 Markdown 表格轉換為 HTML 表格。該元件接受 Markdown 表格作為其內容。表格欄位對齊支援使用預設的 Markdown 表格對齊語法：

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

您可以將所有 Markdown 通知元件匯出到您自己的應用程式中進行自訂。若要匯出這些元件，請使用 `vendor:publish` Artisan 指令來發布 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此指令會將 Markdown 郵件元件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含一個 `html` 和一個 `text` 目錄，每個目錄都包含所有可用元件的各自表示形式。您可以隨意自訂這些元件。

<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以在此檔案中自訂 CSS，您的樣式將自動內聯到 Markdown 通知產生的 HTML 中。

如果您想為 Laravel 的 Markdown 元件建構一個全新的主題，您可以將一個 CSS 檔案放置在 `html/themes` 目錄中。在命名並儲存您的 CSS 檔案後，更新 `mail` 設定檔的 `theme` 選項以符合您新主題的名稱。

若要自訂單一通知的主題，您可以在建構通知的郵件訊息時呼叫 `theme` 方法。`theme` 方法接受傳送通知時應使用的主題名稱：

    /**
     * Get the mail representation of the notification.
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
### 必要條件

`database` 通知通道會將通知資訊儲存到資料庫資料表中。此資料表會包含通知類型以及描述通知的 JSON 資料結構等資訊。

您可以查詢此資料表來在應用程式的使用者介面中顯示通知。但在此之前，您需要建立一個資料庫資料表來存放您的通知。您可以使用 `make:notifications-table` 命令來生成具有正確資料表結構的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan make:notifications-table

php artisan migrate
```

> [!NOTE]
> 如果您的可通知模型使用 [UUID 或 ULID 主鍵](/docs/{{version}}/eloquent#uuid-and-ulid-keys)，您應該在通知資料表遷移中將 `morphs` 方法替換為 [`uuidMorphs`](/docs/{{version}}/migrations#column-method-uuidMorphs) 或 [`ulidMorphs`](/docs/{{version}}/migrations#column-method-ulidMorphs)。

<a name="formatting-database-notifications"></a>
### 格式化資料庫通知

如果通知支援儲存在資料庫資料表中，您應該在通知類別上定義 `toDatabase` 或 `toArray` 方法。此方法會接收一個 `$notifiable` 實體，並應回傳一個純 PHP 陣列。回傳的陣列會被編碼為 JSON 並儲存在 `notifications` 資料表的 `data` 欄位中。讓我們看看 `toArray` 方法的範例：

    /**
     * Get the array representation of the notification.
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

當通知儲存在應用程式的資料庫中時，`type` 欄位預設會設定為通知的類別名稱，而 `read_at` 欄位會是 `null`。不過，您可以透過在通知類別中定義 `databaseType` 和 `initialDatabaseReadAtValue` 方法來客製化此行為：

    use Illuminate\Support\Carbon;

    /**
     * Get the notification's database type.
     */
    public function databaseType(object $notifiable): string
    {
        return 'invoice-paid';
    }

    /**
     * Get the initial value for the "read_at" column.
     */
    public function initialDatabaseReadAtValue(): ?Carbon
    {
        return null;
    }

<a name="todatabase-vs-toarray"></a>
#### `toDatabase` vs. `toArray`

`toArray` 方法也會被 `broadcast` 通道用來決定要廣播哪些資料到支援 JavaScript 的前端。如果您想為 `database` 和 `broadcast` 通道提供兩種不同的陣列表示，您應該定義 `toDatabase` 方法而不是 `toArray` 方法。

<a name="accessing-the-notifications"></a>
### 存取通知

一旦通知儲存在資料庫中，您需要一種方便的方式來從您的可通知實體存取它們。`Illuminate\Notifications\Notifiable` trait (特徵) 包含在 Laravel 預設的 `App\Models\User` 模型中，其中包含一個 `notifications` [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)，它會回傳該實體的通知。若要取得通知，您可以像存取任何其他 Eloquent 關聯一樣存取此方法。依預設，通知會按照 `created_at` 時間戳記進行排序，最新的通知會顯示在集合的最前方：

    $user = App\Models\User::find(1);

    foreach ($user->notifications as $notification) {
        echo $notification->type;
    }

如果您想只取得「未讀」通知，可以使用 `unreadNotifications` 關聯。同樣地，這些通知會按照 `created_at` 時間戳記進行排序，最新的通知會顯示在集合的最前方：

    $user = App\Models\User::find(1);

    foreach ($user->unreadNotifications as $notification) {
        echo $notification->type;
    }

> [!NOTE]
> 若要從您的 JavaScript 客戶端存取通知，您應該為應用程式定義一個通知控制器，該控制器會回傳某個可通知實體 (例如目前使用者) 的通知。然後，您可以從 JavaScript 客戶端向該控制器的 URL 發出 HTTP 請求。

<a name="marking-notifications-as-read"></a>
### 標記通知為已讀

通常，當使用者檢視通知時，您會希望將通知標記為「已讀」。`Illuminate\Notifications\Notifiable` trait 提供了一個 `markAsRead` 方法，該方法會更新通知在資料庫記錄中的 `read_at` 欄位：

    $user = App\Models\User::find(1);

    foreach ($user->unreadNotifications as $notification) {
        $notification->markAsRead();
    }

不過，您也可以直接在通知集合上使用 `markAsRead` 方法，而無需迴圈遍歷每個通知：

    $user->unreadNotifications->markAsRead();

您也可以使用批次更新查詢來將所有通知標記為已讀，而無需從資料庫中取回它們：

    $user = App\Models\User::find(1);

    $user->unreadNotifications()->update(['read_at' => now()]);

您可以 `delete` 通知來將它們從資料表中完全移除：

    $user->notifications()->delete();

<a name="broadcast-notifications"></a>
## 廣播通知

<a name="broadcast-prerequisites"></a>
### 必要條件

在廣播通知之前，您應該先設定並熟悉 Laravel 的 [事件廣播](/docs/{{version}}/broadcasting) 服務。事件廣播提供了一種從您由 JavaScript 驅動的前端對伺服器端 Laravel 事件做出反應的方式。

<a name="formatting-broadcast-notifications"></a>
### 格式化廣播通知

`broadcast` 通道使用 Laravel 的 [事件廣播](/docs/{{version}}/broadcasting) 服務來廣播通知，讓您由 JavaScript 驅動的前端能夠即時捕捉通知。如果通知支援廣播，您可以在通知類別上定義一個 `toBroadcast` 方法。這個方法會接收一個 `$notifiable` 實體，並應該回傳一個 `BroadcastMessage` 實例。如果 `toBroadcast` 方法不存在，則會使用 `toArray` 方法來收集應該廣播的資料。回傳的資料將會被編碼為 JSON 並廣播到您由 JavaScript 驅動的前端。讓我們來看一個 `toBroadcast` 方法的範例：

    use Illuminate\Notifications\Messages\BroadcastMessage;

    /**
     * Get the broadcastable representation of the notification.
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

所有廣播通知都會被佇列等待廣播。如果您想設定用於佇列廣播操作的佇列連線或佇列名稱，您可以使用 `BroadcastMessage` 的 `onConnection` 和 `onQueue` 方法：

    return (new BroadcastMessage($data))
        ->onConnection('sqs')
        ->onQueue('broadcasts');

<a name="customizing-the-notification-type"></a>
#### 自訂通知類型

除了您指定的資料外，所有廣播通知也都包含一個 `type` 欄位，其中包含通知的完整類別名稱。如果您想要自訂通知的 `type`，您可以在通知類別上定義一個 `broadcastType` 方法：

    /**
     * Get the type of the notification being broadcast.
     */
    public function broadcastType(): string
    {
        return 'broadcast.message';
    }

<a name="listening-for-notifications"></a>
### 監聽通知

通知將會透過一個使用 `{notifiable}.{id}` 慣例格式化的私有通道進行廣播。因此，如果您正在向 ID 為 `1` 的 `App\Models\User` 實例傳送通知，該通知將會透過 `App.Models.User.1` 私有通道進行廣播。當使用 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，您可以透過 `notification` 方法輕鬆監聽通道上的通知：

    Echo.private('App.Models.User.' + userId)
        .notification((notification) => {
            console.log(notification.type);
        });

<a name="customizing-the-notification-channel"></a>
#### 自訂通知通道

如果您想自訂實體的廣播通知在哪個通道上廣播，您可以在 notifiable 實體上定義一個 `receivesBroadcastNotificationsOn` 方法：

    <?php

    namespace App\Models;

    use Illuminate\Broadcasting\PrivateChannel;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * The channels the user receives notification broadcasts on.
         */
        public function receivesBroadcastNotificationsOn(): string
        {
            return 'users.'.$this->id;
        }
    }

<a name="sms-notifications"></a>
## 簡訊通知

<a name="sms-prerequisites"></a>
### 必要條件

Laravel 中傳送簡訊通知由 [Vonage](https://www.vonage.com/) (前身為 Nexmo) 提供支援。在透過 Vonage 傳送通知之前，您需要安裝 `laravel/vonage-notification-channel` 和 `guzzlehttp/guzzle` 套件：

    composer require laravel/vonage-notification-channel guzzlehttp/guzzle

該套件包含一個 [設定檔](https://github.com/laravel/vonage-notification-channel/blob/3.x/config/vonage.php)。但是，您不需要將這個設定檔匯出到您自己的應用程式中。您可以直接使用 `VONAGE_KEY` 和 `VONAGE_SECRET` 環境變數來定義您的 Vonage 公開和私密金鑰。

定義金鑰之後，您應該設定一個 `VONAGE_SMS_FROM` 環境變數，該變數定義了您的簡訊訊息預設應該從哪個電話號碼傳送。您可以在 Vonage 控制面板中生成這個電話號碼：

    VONAGE_SMS_FROM=15556666666

<a name="formatting-sms-notifications"></a>
### 格式化簡訊通知

如果通知支援以簡訊傳送，您應該在通知類別上定義一個 `toVonage` 方法。這個方法會接收一個 `$notifiable` 實體，並應該回傳一個 `Illuminate\Notifications\Messages\VonageMessage` 實例：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * Get the Vonage / SMS representation of the notification.
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->content('Your SMS message content');
    }

<a name="unicode-content"></a>
#### Unicode 內容

如果您的簡訊訊息將包含 Unicode 字元，您應該在建構 `VonageMessage` 實例時呼叫 `unicode` 方法：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * Get the Vonage / SMS representation of the notification.
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->content('Your unicode message')
            ->unicode();
    }

<a name="customizing-the-from-number"></a>
### 自訂「寄件者」號碼

如果您希望從一個不同於 `VONAGE_SMS_FROM` 環境變數指定的電話號碼傳送某些通知，您可以在 `VonageMessage` 實例上呼叫 `from` 方法：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * Get the Vonage / SMS representation of the notification.
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->content('Your SMS message content')
            ->from('15554443333');
    }

<a name="adding-a-client-reference"></a>
### 新增客戶參考

如果您想追蹤每個使用者、團隊或客戶的費用，您可以為通知新增一個「客戶參考」。Vonage 將允許您使用此客戶參考生成報告，以便您更好地了解特定客戶的簡訊使用情況。客戶參考可以是任何最長 40 個字元的字串：

    use Illuminate\Notifications\Messages\VonageMessage;

    /**
     * Get the Vonage / SMS representation of the notification.
     */
    public function toVonage(object $notifiable): VonageMessage
    {
        return (new VonageMessage)
            ->clientReference((string) $notifiable->id)
            ->content('Your SMS message content');
    }

<a name="routing-sms-notifications"></a>
### 簡訊通知路由

為了將 Vonage 通知路由到正確的電話號碼，請在您的可通知實體上定義 `routeNotificationForVonage` 方法：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * Route notifications for the Vonage channel.
         */
        public function routeNotificationForVonage(Notification $notification): string
        {
            return $this->phone_number;
        }
    }

<a name="slack-notifications"></a>
## Slack 通知

<a name="slack-prerequisites"></a>
### 必要條件

在傳送 Slack 通知之前，您應該透過 Composer 安裝 Slack 通知通道：

```shell
composer require laravel/slack-notification-channel
```

此外，您必須為您的 Slack 工作區建立一個 [Slack App](https://api.slack.com/apps?new_app=1)。

如果您只需要傳送通知到建立該 App 的 Slack 工作區，您應該確保您的 App 擁有 `chat:write`、`chat:write.public` 和 `chat:write.customize` 範圍。如果您想以您的 Slack App 名義傳送訊息，您應該確保您的 App 也擁有 `chat:write:bot` 範圍。這些範圍可以從 Slack 內部「OAuth & 權限」App 管理頁籤中新增。

接下來，複製 App 的「Bot User OAuth Token」，並將其放置於您應用程式的 `services.php` 設定檔中的 `slack` 設定陣列內。此 token 可以在 Slack 內部「OAuth & 權限」頁籤中找到：

    'slack' => [
        'notifications' => [
            'bot_user_oauth_token' => env('SLACK_BOT_USER_OAUTH_TOKEN'),
            'channel' => env('SLACK_BOT_USER_DEFAULT_CHANNEL'),
        ],
    ],

<a name="slack-app-distribution"></a>
#### App 分發

如果您的應用程式將向您應用程式使用者所擁有的外部 Slack 工作區傳送通知，您將需要透過 Slack「分發」您的 App。App 分發可以在 Slack 內部您 App 的「管理分發」頁籤中進行管理。一旦您的 App 已分發，您可以使用 [Socialite](/docs/{{version}}/socialite) 代表您應用程式的使用者[取得 Slack Bot token](/docs/{{version}}/socialite#slack-bot-scopes)。

<a name="formatting-slack-notifications"></a>
### 格式化 Slack 通知

如果通知支援作為 Slack 訊息傳送，您應該在通知類別上定義一個 `toSlack` 方法。此方法將接收一個 `$notifiable` 實例，並應回傳一個 `Illuminate\Notifications\Slack\SlackMessage` 實例。您可以使用 [Slack 的 Block Kit API](https://api.slack.com/block-kit) 來建構豐富的通知。以下範例可在 [Slack 的 Block Kit builder](https://app.slack.com/block-kit-builder/T01KWS6K23Z#%7B%22blocks%22:%5B%7B%22type%22:%22header%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Invoice%20Paid%22%7D%7D,%7B%22type%22:%22context%22,%22elements%22:%5B%7B%22type%22:%22plain_text%22,%22text%22:%22Customer%20%231234%22%7D%5D%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22An%20invoice%20has%20been%20paid.%22%7D,%22fields%22:%5B%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20No:*%5Cn1000%22%7D,%7B%22type%22:%22mrkdwn%22,%22text%22:%22*Invoice%20Recipient:*%5Cntaylor@laravel.com%22%7D%5D%7D,%7B%22type%22:%22divider%22%7D,%7B%22type%22:%22section%22,%22text%22:%7B%22type%22:%22plain_text%22,%22text%22:%22Congratulations!%22%7D%7D%5D%7D) 中預覽：

    use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
    use Illuminate\Notifications\Slack\BlockKit\Composites\ConfirmObject;
    use Illuminate\Notifications\Slack\SlackMessage;

    /**
     * Get the Slack representation of the notification.
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

<a name="using-slacks-block-kit-builder-template"></a>
#### 使用 Slack 的 Block Kit Builder 樣板

您可以提供由 Slack 的 Block Kit Builder 生成的原始 JSON payload 給 `usingBlockKitTemplate` 方法，而無需使用流暢的訊息建構方法來建構您的 Block Kit 訊息：

    use Illuminate\Notifications\Slack\SlackMessage;
    use Illuminate\Support\Str;

    /**
     * Get the Slack representation of the notification.
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

Slack 的 Block Kit 通知系統提供了強大功能來[處理使用者互動](https://api.slack.com/interactivity/handling)。為使用這些功能，您的 Slack App 應啟用「互動性 (Interactivity)」並設定一個「請求 URL (Request URL)」指向由您應用程式提供的 URL。這些設定可在 Slack 應用程式管理介面中「互動性與捷徑 (Interactivity & Shortcuts)」分頁進行管理。

在以下範例中，使用了 `actionsBlock` 方法，Slack 會向您的「請求 URL (Request URL)」發送一個 `POST` 請求，其 Payload 包含點擊按鈕的 Slack 使用者、點擊按鈕的 ID 等資訊。您的應用程式可根據此 Payload 決定要採取的動作。您也應[驗證此請求](https://api.slack.com/authentication/verifying-requests-from-slack)確實來自 Slack：

    use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
    use Illuminate\Notifications\Slack\SlackMessage;

    /**
     * Get the Slack representation of the notification.
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


<a name="slack-confirmation-modals"></a>
#### 確認彈出視窗

如果您希望使用者在執行動作前必須確認，您可以在定義按鈕時呼叫 `confirm` 方法。`confirm` 方法接受一個訊息與一個閉包，此閉包會接收一個 `ConfirmObject` 實例：

    use Illuminate\Notifications\Slack\BlockKit\Blocks\ActionsBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\ContextBlock;
    use Illuminate\Notifications\Slack\BlockKit\Blocks\SectionBlock;
    use Illuminate\Notifications\Slack\BlockKit\Composites\ConfirmObject;
    use Illuminate\Notifications\Slack\SlackMessage;

    /**
     * Get the Slack representation of the notification.
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

如果您想快速檢查您正在建立的區塊，您可以對 `SlackMessage` 實例呼叫 `dd` 方法。`dd` 方法會生成並輸出一個 URL 到 Slack 的 [Block Kit Builder](https://app.slack.com/block-kit-builder/)，在瀏覽器中顯示 Payload 與通知的預覽。您可以將 `true` 傳遞給 `dd` 方法以輸出原始 Payload：

    return (new SlackMessage)
            ->text('One of your invoices has been paid!')
            ->headerBlock('Invoice Paid')
            ->dd();


<a name="routing-slack-notifications"></a>
### Slack 通知路由

要將 Slack 通知導向至適當的 Slack 團隊與通道，請在您的 notifiable 模型上定義一個 `routeNotificationForSlack` 方法。此方法可回傳以下三種值之一：

- `null` - 這會將路由延遲到通知本身配置的通道。您可以在建構 `SlackMessage` 時使用 `to` 方法在通知內配置通道。
- 一個字串，指定要傳送通知的 Slack 通道，例如 `#support-channel`。
- 一個 `SlackRoute` 實例，允許您指定 OAuth token 與通道名稱，例如 `SlackRoute::make($this->slack_channel, $this->slack_token)`。此方法應用於向外部工作區傳送通知。

舉例來說，從 `routeNotificationForSlack` 方法回傳 `#support-channel` 會將通知傳送到與您應用程式 `services.php` 設定檔中 Bot User OAuth token 相關聯的工作區中的 `#support-channel` 通道：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Notifications\Notification;

    class User extends Authenticatable
    {
        use Notifiable;

        /**
         * Route notifications for the Slack channel.
         */
        public function routeNotificationForSlack(Notification $notification): mixed
        {
            return '#support-channel';
        }
    }


<a name="notifying-external-slack-workspaces"></a>
### 通知外部 Slack 工作區

> [!NOTE]
> 在向外部 Slack 工作區傳送通知之前，您的 Slack App 必須先[發佈](#slack-app-distribution)。

當然，您通常會希望將通知傳送到由應用程式使用者擁有的 Slack 工作區。為此，您首先需要為該使用者取得一個 Slack OAuth token。幸運的是，[Laravel Socialite](/docs/{{version}}/socialite) 包含一個 Slack 驅動，可讓您輕鬆地使用 Slack 驗證應用程式使用者並[取得 Bot token](/docs/{{version}}/socialite#slack-bot-scopes)。

一旦您取得 Bot token 並將其儲存在應用程式的資料庫中，您就可以利用 `SlackRoute::make` 方法將通知路由到使用者的工作區。此外，您的應用程式可能需要提供使用者指定通知應傳送到的通道的機會：

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
         * Route notifications for the Slack channel.
         */
        public function routeNotificationForSlack(Notification $notification): mixed
        {
            return SlackRoute::make($this->slack_channel, $this->slack_token);
        }
    }

<a name="localizing-notifications"></a>
## 通知本地化

Laravel 允許您以不同於 HTTP 請求當前語系的方式傳送通知，如果通知已排入佇列，它甚至會記住此語系。

為了實現此功能，`Illuminate\Notifications\Notification` 類別提供了 `locale` 方法來設定所需的語系。應用程式會在評估通知時切換到此語系，然後在評估完成後回復到先前的語系：

    $user->notify((new InvoicePaid($invoice))->locale('es'));

多個可通知實體的本地化也可以透過 `Notification` Facade 實現：

    Notification::locale('es')->send(
        $users, new InvoicePaid($invoice)
    );

<a name="user-preferred-locales"></a>
### 使用者偏好語系

有時，應用程式會儲存每個使用者偏好的語系。透過在您的可通知模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在傳送通知時使用此儲存的語系：

    use Illuminate\Contracts\Translation\HasLocalePreference;

    class User extends Model implements HasLocalePreference
    {
        /**
         * Get the user's preferred locale.
         */
        public function preferredLocale(): string
        {
            return $this->locale;
        }
    }

一旦您實作了該介面，Laravel 在向模型傳送通知和 Mailable 時，會自動使用偏好的語系。因此，使用此介面時，無需呼叫 `locale` 方法：

    $user->notify(new InvoicePaid($invoice));

<a name="testing"></a>
## 測試

您可以使用 `Notification` Facade 的 `fake` 方法來防止通知被傳送。通常，傳送通知與您實際測試的程式碼無關。最可能的情況是，僅斷言 Laravel 已被指示傳送某個通知就已足夠。

在呼叫 `Notification` Facade 的 `fake` 方法後，您可以斷言通知已被指示傳送給使用者，甚至檢查通知接收到的資料：

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

您可以將閉包傳遞給 `assertSentTo` 或 `assertNotSentTo` 方法，以斷言傳送的通知通過了指定的「真值測試」。如果至少有一個通知通過了給定的真值測試，則斷言會成功：

    Notification::assertSentTo(
        $user,
        function (OrderShipped $notification, array $channels) use ($order) {
            return $notification->order->id === $order->id;
        }
    );

<a name="on-demand-notifications"></a>
#### 即時通知

如果您正在測試的程式碼傳送 [即時通知](#on-demand-notifications)，您可以透過 `assertSentOnDemand` 方法測試該即時通知是否已傳送：

    Notification::assertSentOnDemand(OrderShipped::class);

透過將閉包作為第二個參數傳遞給 `assertSentOnDemand` 方法，您可以判斷即時通知是否已傳送至正確的「路由」位址：

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

當通知正在傳送時，通知系統會分派 `Illuminate\Notifications\Events\NotificationSending` 事件。此事件包含「可通知」實體與通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

    use Illuminate\Notifications\Events\NotificationSending;

    class CheckNotificationStatus
    {
        /**
         * Handle the given event.
         */
        public function handle(NotificationSending $event): void
        {
            // ...
        }
    }

如果 `NotificationSending` 事件的事件監聽器從其 `handle` 方法回傳 `false`，則通知將不會被傳送：

    /**
     * Handle the given event.
     */
    public function handle(NotificationSending $event): bool
    {
        return false;
    }

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification` 和 `channel` 屬性，以了解更多關於通知接收者或通知本身的資訊：

    /**
     * Handle the given event.
     */
    public function handle(NotificationSending $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
    }

<a name="notification-sent-event"></a>
#### 通知已傳送事件

當通知傳送完成時，通知系統會分派 `Illuminate\Notifications\Events\NotificationSent` [事件](/docs/{{version}}/events)。此事件包含「可通知」實體與通知實例本身。您可以在應用程式中為此事件建立 [事件監聽器](/docs/{{version}}/events)：

    use Illuminate\Notifications\Events\NotificationSent;

    class LogNotification
    {
        /**
         * Handle the given event.
         */
        public function handle(NotificationSent $event): void
        {
            // ...
        }
    }

在事件監聽器中，您可以存取事件上的 `notifiable`、`notification`、`channel` 和 `response` 屬性，以了解更多關於通知接收者或通知本身的資訊：

    /**
     * Handle the given event.
     */
    public function handle(NotificationSent $event): void
    {
        // $event->channel
        // $event->notifiable
        // $event->notification
        // $event->response
    }

<a name="custom-channels"></a>
## 自訂通道

Laravel 內建了一些通知通道，但你可能希望編寫自己的驅動來透過其他通道傳送通知。Laravel 讓這變得簡單。首先，定義一個包含 `send` 方法的類別。該方法應接收兩個引數：`$notifiable` 和 `$notification`。

在 `send` 方法中，你可以呼叫通知上的方法來取得你的通道能理解的訊息物件，然後以你希望的任何方式將通知傳送給 `$notifiable` 實例：

    <?php

    namespace App\Notifications;

    use Illuminate\Notifications\Notification;

    class VoiceChannel
    {
        /**
         * Send the given notification.
         */
        public function send(object $notifiable, Notification $notification): void
        {
            $message = $notification->toVoice($notifiable);

            // Send notification to the $notifiable instance...
        }
    }

定義了通知通道類別後，你就可以從任何通知的 `via` 方法回傳該類別名稱。在這個範例中，通知的 `toVoice` 方法可以回傳任何你選擇用來表示語音訊息的物件。例如，你可能會定義自己的 `VoiceMessage` 類別來表示這些訊息：

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
         * Get the notification channels.
         */
        public function via(object $notifiable): string
        {
            return VoiceChannel::class;
        }

        /**
         * Get the voice representation of the notification.
         */
        public function toVoice(object $notifiable): VoiceMessage
        {
            // ...
        }
    }