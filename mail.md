# 郵件

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
    - [故障轉移設定](#failover-configuration)
    - [循環負載設定](#round-robin-configuration)
- [生成 Mailables](#generating-mailables)
- [編寫 Mailables](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [行內附件](#inline-attachments)
    - [可附件物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與中繼資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailables](#markdown-mailables)
    - [生成 Markdown Mailables](#generating-markdown-mailables)
    - [編寫 Markdown 訊息](#writing-markdown-messages)
    - [自訂元件](#customizing-the-components)
- [傳送郵件](#sending-mail)
    - [佇列郵件](#queueing-mail)
- [渲染 Mailables](#rendering-mailables)
    - [在瀏覽器中預覽 Mailables](#previewing-mailables-in-the-browser)
- [本地化 Mailables](#localizing-mailables)
- [測試](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 傳送](#testing-mailable-sending)
- [郵件與本地開發](#mail-and-local-development)
- [事件](#events)
- [自訂傳輸器](#custom-transports)
    - [額外 Symfony 傳輸器](#additional-symfony-transports)

<a name="introduction"></a>
## 簡介

傳送電子郵件不必複雜。Laravel 提供一套乾淨簡潔的電子郵件 API，它由廣受歡迎的 [Symfony Mailer](https://symfony.com/doc/7.0/mailer.html) 元件所驅動。Laravel 和 Symfony Mailer 提供驅動程式，可透過 SMTP、Mailgun、Postmark、Resend、Amazon SES 和 `sendmail` 傳送電子郵件，讓您可以快速地開始透過本地或雲端服務傳送您選擇的郵件。

<a name="configuration"></a>
### 設定

Laravel 的郵件服務可以透過您應用程式的 `config/mail.php` 設定檔進行設定。在此檔案中設定的每個郵件傳送器都可以有其獨特的設定，甚至有其獨特的「傳輸器」，這讓您的應用程式可以使用不同的郵件服務來傳送特定的電子郵件訊息。例如，您的應用程式可能使用 Postmark 來傳送事務性電子郵件，同時使用 Amazon SES 來傳送大量電子郵件。

在您的 `mail` 設定檔中，您會找到一個 `mailers` 設定陣列。此陣列包含 Laravel 支援的每個主要郵件驅動程式/傳輸器的範例設定項目，而 `default` 設定值決定了當您的應用程式需要傳送電子郵件訊息時，將預設使用哪個郵件傳送器。

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

基於 API 的驅動程式，例如 Mailgun、Postmark、Resend 和 MailerSend，通常比透過 SMTP 伺服器傳送郵件更簡單、更快。在可能的情況下，我們建議您使用這些驅動程式之一。

<a name="mailgun-driver"></a>
#### Mailgun 驅動程式

要使用 Mailgun 驅動程式，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳輸器：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接下來，您需要在應用程式的 `config/mail.php` 設定檔中進行兩項變更。首先，將您的預設郵件傳送器設定為 `mailgun`：

    'default' => env('MAIL_MAILER', 'mailgun'),

其次，將以下設定陣列新增到您的 `mailers` 陣列中：

    'mailgun' => [
        'transport' => 'mailgun',
        // 'client' => [
        //     'timeout' => 5,
        // ],
    ],

設定應用程式的預設郵件傳送器後，將以下選項新增到您的 `config/services.php` 設定檔中：

    'mailgun' => [
        'domain' => env('MAILGUN_DOMAIN'),
        'secret' => env('MAILGUN_SECRET'),
        'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
        'scheme' => 'https',
    ],

如果您沒有使用美國 [Mailgun 區域](https://documentation.mailgun.com/en/latest/api-intro.html#mailgun-regions)，您可以在 `services` 設定檔中定義您所在區域的端點：

    'mailgun' => [
        'domain' => env('MAILGUN_DOMAIN'),
        'secret' => env('MAILGUN_SECRET'),
        'endpoint' => env('MAILGUN_ENDPOINT', 'api.eu.mailgun.net'),
        'scheme' => 'https',
    ],

<a name="postmark-driver"></a>
#### Postmark 驅動程式

要使用 [Postmark](https://postmarkapp.com/) 驅動程式，請透過 Composer 安裝 Symfony 的 Postmark Mailer 傳輸器：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接下來，在應用程式的 `config/mail.php` 設定檔中，將 `default` 選項設定為 `postmark`。設定應用程式的預設郵件傳送器後，請確保您的 `config/services.php` 設定檔包含以下選項：

    'postmark' => [
        'token' => env('POSTMARK_TOKEN'),
    ],

如果您想指定特定郵件傳送器應使用的 Postmark 訊息串流，您可以將 `message_stream_id` 設定選項新增到該郵件傳送器的設定陣列中。這個設定陣列位於您應用程式的 `config/mail.php` 設定檔中：

    'postmark' => [
        'transport' => 'postmark',
        'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
        // 'client' => [
        //     'timeout' => 5,
        // ],
    ],

透過這種方式，您還可以設定多個具有不同訊息串流的 Postmark 郵件傳送器。

<a name="resend-driver"></a>
#### Resend 驅動程式

要使用 [Resend](https://resend.com/) 驅動程式，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接下來，在應用程式的 `config/mail.php` 設定檔中，將 `default` 選項設定為 `resend`。設定應用程式的預設郵件傳送器後，請確保您的 `config/services.php` 設定檔包含以下選項：

    'resend' => [
        'key' => env('RESEND_KEY'),
    ],

<a name="ses-driver"></a>
#### SES 驅動程式

要使用 Amazon SES 驅動程式，您必須先安裝適用於 PHP 的 Amazon AWS SDK。您可以透過 Composer 套件管理器安裝此程式庫：

```shell
composer require aws/aws-sdk-php
```

接下來，在您的 `config/mail.php` 設定檔中將 `default` 選項設定為 `ses`，並確認您的 `config/services.php` 設定檔包含以下選項：

    'ses' => [
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    ],

若要透過會話令牌利用 AWS [臨時憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，您可以在應用程式的 SES 設定中新增一個 `token` 金鑰：

    'ses' => [
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        'token' => env('AWS_SESSION_TOKEN'),
    ],

若要與 SES 的 [訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html) 互動，您可以傳回郵件訊息 [`headers`](#headers) 方法所傳回陣列中的 `X-Ses-List-Management-Options` 標頭：

```php
/**
 * Get the message headers.
 */
public function headers(): Headers
{
    return new Headers(
        text: [
            'X-Ses-List-Management-Options' => 'contactListName=MyContactList;topicName=MyTopic',
        ],
    );
}
```

如果您想定義 [額外選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，讓 Laravel 在傳送電子郵件時傳遞給 AWS SDK 的 `SendEmail` 方法，您可以在 `ses` 設定中定義一個 `options` 陣列：

    'ses' => [
        'key' => env('AWS_ACCESS_KEY_ID'),
        'secret' => env('AWS_SECRET_ACCESS_KEY'),
        'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        'options' => [
            'ConfigurationSetName' => 'MyConfigurationSet',
            'EmailTags' => [
                ['Name' => 'foo', 'Value' => 'bar'],
            ],
        ],
    ],

<a name="mailersend-driver"></a>
#### MailerSend 驅動程式

[MailerSend](https://www.mailersend.com/) 是一項事務性電子郵件和簡訊服務，他們為 Laravel 維護著自己的基於 API 的郵件驅動程式。包含該驅動程式的套件可透過 Composer 套件管理器安裝：

```shell
composer require mailersend/laravel-driver
```

安裝套件後，將 `MAILERSEND_API_KEY` 環境變數新增到應用程式的 `.env` 檔案中。此外，`MAIL_MAILER` 環境變數應定義為 `mailersend`：

```ini
MAIL_MAILER=mailersend
MAIL_FROM_ADDRESS=app@yourdomain.com
MAIL_FROM_NAME="App Name"

MAILERSEND_API_KEY=your-api-key
```

最後，將 MailerSend 新增到應用程式 `config/mail.php` 設定檔中的 `mailers` 陣列：

```php
'mailersend' => [
    'transport' => 'mailersend',
],
```

要了解更多關於 MailerSend 的資訊，包括如何使用託管範本，請查閱 [MailerSend 驅動程式文件](https://github.com/mailersend/mailersend-laravel-driver#usage)。

<a name="failover-configuration"></a>
### 故障轉移設定

有時，您為應用程式配置來傳送郵件的外部服務可能會停機。在這些情況下，定義一個或多個備用郵件傳送設定會很有用，以便在主要傳送驅動程式停機時使用。

為此，您應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳輸器的郵件發送器 (mailer)。應用程式 `failover` 郵件發送器的設定陣列應包含一個 `mailers` 陣列，該陣列參照了應選擇配置好的郵件發送器的傳送順序：

    'mailers' => [
        'failover' => [
            'transport' => 'failover',
            'mailers' => [
                'postmark',
                'mailgun',
                'sendmail',
            ],
        ],

        // ...
    ],

一旦您的故障轉移郵件發送器已定義，您應該透過在其應用程式的 `mail` 設定檔中將其名稱指定為 `default` 設定鍵的值，來將此郵件發送器設定為應用程式使用的預設郵件發送器：

    'default' => env('MAIL_MAILER', 'failover'),

<a name="round-robin-configuration"></a>
### 循環負載設定

`roundrobin` 傳輸器允許您將郵件傳送工作負載分散到多個郵件發送器。首先，您需要在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` 傳輸器的郵件發送器。您的應用程式 `roundrobin` 郵件發送器的設定陣列應包含一個 `mailers` 陣列，該陣列參照了應使用哪些配置好的郵件發送器進行傳送：

    'mailers' => [
        'roundrobin' => [
            'transport' => 'roundrobin',
            'mailers' => [
                'ses',
                'postmark',
            ],
        ],

        // ...
    ],

一旦您的循環負載郵件發送器已定義，您應該透過在其應用程式的 `mail` 設定檔中將其名稱指定為 `default` 設定鍵的值，來將此郵件發送器設定為應用程式使用的預設郵件發送器：

    'default' => env('MAIL_MAILER', 'roundrobin'),

循環負載傳輸器會從配置好的郵件發送器列表中隨機選擇一個郵件發送器，然後為每封後續的電子郵件切換到下一個可用的郵件發送器。與有助於實現 *[高可用性](https://en.wikipedia.org/wiki/High_availability)* 的 `failover` 傳輸器相反，`roundrobin` 傳輸器提供了 *[負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))*.

<a name="generating-mailables"></a>
## 生成 Mailables

在建構 Laravel 應用程式時，您的應用程式所寄送的每種類型電子郵件都以一個「mailable」類別來表示。這些類別儲存在 `app/Mail` 目錄中。如果您的應用程式中沒有看到這個目錄，請別擔心，因為當您使用 `make:mail` Artisan 指令建立您的第一個 mailable 類別時，它會為您生成：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 編寫 Mailables

生成 mailable 類別後，請開啟它，以便我們探究其內容。Mailable 類別的設定是透過多個方法完成的，其中包括 `envelope`、`content` 和 `attachments` 方法。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，該物件定義了郵件主旨，有時也定義了收件者。`content` 方法會回傳一個 `Illuminate\Mail\Mailables\Content` 物件，該物件定義了用於生成郵件內容的 [Blade 模板](/docs/{{version}}/blade)。

<a name="configuring-the-sender"></a>
### 設定寄件者

<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，讓我們探討如何設定電子郵件的寄件者。換句話說，就是電子郵件的「寄件人」。有兩種方式可以設定寄件者。首先，您可以在郵件的 Envelope 中指定「寄件人」地址：

    use Illuminate\Mail\Mailables\Address;
    use Illuminate\Mail\Mailables\Envelope;

    /**
     * Get the message envelope.
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            from: new Address('jeffrey@example.com', 'Jeffrey Way'),
            subject: 'Order Shipped',
        );
    }

如果您願意，也可以指定 `replyTo` 地址：

    return new Envelope(
        from: new Address('jeffrey@example.com', 'Jeffrey Way'),
        replyTo: [
            new Address('taylor@example.com', 'Taylor Otwell'),
        ],
        subject: 'Order Shipped',
    );

<a name="using-a-global-from-address"></a>
#### 使用全局 `from` 位址

然而，如果您的應用程式對所有電子郵件都使用相同的「寄件人」地址，那麼在每個您生成的 mailable 類別中都添加它可能會變得麻煩。相反地，您可以在應用程式的 `config/mail.php` 設定檔中指定一個全局的 from 地址。如果在 mailable 類別中沒有指定其他 from 地址，則將使用此地址：

    'from' => [
        'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
        'name' => env('MAIL_FROM_NAME', 'Example'),
    ],

此外，您可以在應用程式的 `config/mail.php` 設定檔中定義一個全局的「回覆位址」。

    'reply_to' => ['address' => 'example@example.com', 'name' => 'App Name'],

<a name="configuring-the-view"></a>
### 設定視圖

在 mailable 類別的 `content` 方法中，您可以定義 `view`，也就是渲染電子郵件內容時應使用的模板。由於每封電子郵件通常使用 [Blade 模板](/docs/{{version}}/blade) 來渲染其內容，因此在建立電子郵件的 HTML 時，您擁有 Blade 模板引擎的全部功能和便利。

    /**
     * Get the message content definition.
     */
    public function content(): Content
    {
        return new Content(
            view: 'mail.orders.shipped',
        );
    }

> [!NOTE]  
> 您可能希望建立 `resources/views/emails` 目錄來存放所有電子郵件模板；然而，您可以將它們放置在 `resources/views` 目錄中的任何位置。

<a name="plain-text-emails"></a>
#### 純文字電子郵件

如果您想定義電子郵件的純文字版本，可以在建立郵件的 `Content` 定義時指定純文字模板。像 `view` 參數一樣，`text` 參數也應該是一個模板名稱，用於渲染電子郵件的內容。您可以自由定義郵件的 HTML 和純文字版本：

    /**
     * Get the message content definition.
     */
    public function content(): Content
    {
        return new Content(
            view: 'mail.orders.shipped',
            text: 'mail.orders.shipped-text'
        );
    }

為了清楚起見，`html` 參數可用作 `view` 參數的別名：

    return new Content(
        html: 'mail.orders.shipped',
        text: 'mail.orders.shipped-text'
    );

<a name="view-data"></a>
### 視圖資料

<a name="via-public-properties"></a>
#### 透過 Public 屬性

通常，您會希望向視圖傳遞一些資料，以便在渲染電子郵件的 HTML 時使用。有兩種方法可以讓您的視圖取得資料。首先，任何在 mailable 類別上定義的 public 屬性都將自動提供給視圖。因此，舉例來說，您可以將資料傳入 mailable 類別的建構子，並將該資料設定為類別上定義的 public 屬性：

    <?php

    namespace App\Mail;

    use App\Models\Order;
    use Illuminate\Bus\Queueable;
    use Illuminate\Mail\Mailable;
    use Illuminate\Mail\Mailables\Content;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped extends Mailable
    {
        use Queueable, SerializesModels;

        /**
         * Create a new message instance.
         */
        public function __construct(
            public Order $order,
        ) {}

        /**
         * Get the message content definition.
         */
        public function content(): Content
        {
            return new Content(
                view: 'mail.orders.shipped',
            );
        }
    }

一旦資料被設定為 public 屬性，它將自動在您的視圖中可用，因此您可以像在任何其他 Blade 模板中一樣存取它：

    <div>
        Price: {{ $order->price }}
    </div>

<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果您希望在資料傳送至模板之前自訂電子郵件資料的格式，您可以透過 `Content` 定義的 `with` 參數手動將資料傳遞給視圖。通常，您仍然會透過 mailable 類別的建構子傳遞資料；然而，您應該將這些資料設定為 `protected` 或 `private` 屬性，這樣資料就不會自動提供給模板。

    <?php

    namespace App\Mail;

    use App\Models\Order;
    use Illuminate\Bus\Queueable;
    use Illuminate\Mail\Mailable;
    use Illuminate\Mail\Mailables\Content;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped extends Mailable
    {
        use Queueable, SerializesModels;

        /**
         * Create a new message instance.
         */
        public function __construct(
            protected Order $order,
        ) {}

        /**
         * Get the message content definition.
         */
        public function content(): Content
        {
            return new Content(
                view: 'mail.orders.shipped',
                with: [
                    'orderName' => $this->order->name,
                    'orderPrice' => $this->order->price,
                ],
            );
        }
    }

一旦資料傳遞給 `with` 方法，它將自動在您的視圖中可用，因此您可以像在任何其他 Blade 模板中一樣存取它：

    <div>
        Price: {{ $orderPrice }}
    </div>

<a name="attachments"></a>
### 附件

要將附件新增到電子郵件，您會將附件新增到郵件 `attachments` 方法所回傳的陣列中。首先，您可以透過提供檔案路徑給 `Attachment` 類別所提供的 `fromPath` 方法來新增附件：

    use Illuminate\Mail\Mailables\Attachment;

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromPath('/path/to/file'),
        ];
    }

當附加檔案到郵件時，您也可以使用 `as` 和 `withMime` 方法來指定附件的顯示名稱和/或 MIME 類型：

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromPath('/path/to/file')
                ->as('name.pdf')
                ->withMime('application/pdf'),
        ];
    }


<a name="attaching-files-from-disk"></a>
#### 從磁碟附加檔案

如果您已將檔案儲存在其中一個[檔案系統磁碟](/docs/{{version}}/filesystem)上，您可以使用 `fromStorage` 附件方法將其附加到電子郵件中：

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorage('/path/to/file'),
        ];
    }

當然，您也可以指定附件的名稱和 MIME 類型：

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorage('/path/to/file')
                ->as('name.pdf')
                ->withMime('application/pdf'),
        ];
    }

如果您需要指定非預設磁碟的儲存磁碟，可以使用 `fromStorageDisk` 方法：

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorageDisk('s3', '/path/to/file')
                ->as('name.pdf')
                ->withMime('application/pdf'),
        ];
    }


<a name="raw-data-attachments"></a>
#### 原始資料附件

`fromData` 附件方法可用於將原始位元組字串附加為附件。例如，如果您已在記憶體中生成 PDF 並希望將其附加到電子郵件而不安裝到磁碟上，您可以使用此方法。`fromData` 方法接受一個閉包，該閉包解析原始資料位元組以及應為附件指定的名稱：

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromData(fn () => $this->pdf, 'Report.pdf')
                ->withMime('application/pdf'),
        ];
    }


<a name="inline-attachments"></a>
### 行內附件

在電子郵件中嵌入行內圖片通常很麻煩；然而，Laravel 提供了一種便捷的方式來將圖片附加到您的電子郵件中。要嵌入行內圖片，請在電子郵件範本中使用 `$message` 變數上的 `embed` 方法。Laravel 會自動讓 `$message` 變數在您的所有電子郵件範本中可用，因此您無需擔心手動傳遞它：

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]  
> 由於純文字訊息不使用行內附件，因此 `$message` 變數在純文字訊息範本中不可用。


<a name="embedding-raw-data-attachments"></a>
#### 嵌入原始資料附件

如果您已經有希望嵌入電子郵件範本的原始圖片資料字串，您可以呼叫 `$message` 變數上的 `embedData` 方法。呼叫 `embedData` 方法時，您需要提供一個應分配給嵌入圖片的檔案名稱：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```


<a name="attachable-objects"></a>
### 可附件物件

雖然透過簡單的字串路徑將檔案附加到訊息通常已足夠，但在許多情況下，應用程式中的可附件實體是由類別表示的。例如，如果您的應用程式將照片附加到訊息，您的應用程式可能也有一個 `Photo` 模型來表示該照片。在這種情況下，直接將 `Photo` 模型傳遞給 `attach` 方法會不會很方便？可附件物件正是為此而生。

首先，在將可附件到訊息的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。此介面規定您的類別定義一個 `toMailAttachment` 方法，該方法回傳一個 `Illuminate\Mail\Attachment` 實例：

    <?php

    namespace App\Models;

    use Illuminate\Contracts\Mail\Attachable;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Mail\Attachment;

    class Photo extends Model implements Attachable
    {
        /**
         * Get the attachable representation of the model.
         */
        public function toMailAttachment(): Attachment
        {
            return Attachment::fromPath('/path/to/file');
        }
    }

一旦您定義了可附件物件，您可以在建構電子郵件訊息時從 `attachments` 方法回傳該物件的實例：

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [$this->photo];
    }

當然，附件資料可以儲存在遠端檔案儲存服務上，例如 Amazon S3。因此，Laravel 也允許您從儲存在應用程式[檔案系統磁碟](/docs/{{version}}/filesystem)之一的資料中生成附件實例：

    // Create an attachment from a file on your default disk...
    return Attachment::fromStorage($this->path);

    // Create an attachment from a file on a specific disk...
    return Attachment::fromStorageDisk('backblaze', $this->path);

此外，您還可以透過記憶體中的資料來建立附件實例。為此，請為 `fromData` 方法提供一個閉包。該閉包應回傳表示附件的原始資料：

    return Attachment::fromData(fn () => $this->content, 'Photo Name');

Laravel 也提供其他方法，您可以使用這些方法來自訂您的附件。例如，您可以使用 `as` 和 `withMime` 方法來自訂檔案的名稱和 MIME 類型：

    return Attachment::fromPath('/path/to/file')
        ->as('Photo Name')
        ->withMime('image/jpeg');


<a name="headers"></a>
### 標頭

有時您可能需要為外寄訊息附加額外的標頭。例如，您可能需要設定自訂的 `Message-Id` 或其他任意文字標頭。

為此，請在您的 mailable 上定義一個 `headers` 方法。`headers` 方法應回傳一個 `Illuminate\Mail\Mailables\Headers` 實例。此類別接受 `messageId`、`references` 和 `text` 參數。當然，您可以僅提供特定訊息所需的參數：

    use Illuminate\Mail\Mailables\Headers;

    /**
     * Get the message headers.
     */
    public function headers(): Headers
    {
        return new Headers(
            messageId: 'custom-message-id@example.com',
            references: ['previous-message@example.com'],
            text: [
                'X-Custom-Header' => 'Custom Value',
            ],
        );
    }

<a name="tags-and-metadata"></a>
### 標籤與中繼資料

有些第三方郵件服務提供者，例如 Mailgun 和 Postmark，支援郵件「標籤 (tags)」與「中繼資料 (metadata)」，可用於對您的應用程式傳送的電子郵件進行分組和追蹤。您可以透過 `Envelope` 定義將標籤與中繼資料新增至電子郵件訊息中：

    use Illuminate\Mail\Mailables\Envelope;

    /**
     * Get the message envelope.
     *
     * @return \Illuminate\Mail\Mailables\Envelope
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Order Shipped',
            tags: ['shipment'],
            metadata: [
                'order_id' => $this->order->id,
            ],
        );
    }

如果您的應用程式使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件，以取得有關 [標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tagging) 和 [中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#attaching-data-to-messages) 的更多資訊。同樣地，也可以查閱 Postmark 文件，以取得有關其對 [標籤](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 支援的更多資訊。

如果您的應用程式使用 Amazon SES 傳送電子郵件，您應該使用 `metadata` 方法來將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。


<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能由 Symfony Mailer 提供支援。Laravel 允許您註冊自訂回呼，這些回呼將在傳送訊息之前與 Symfony Message 實例一起被呼叫。這讓您有機會在訊息傳送之前深度自訂訊息。為此，請在您的 `Envelope` 定義中定義 `using` 參數：

    use Illuminate\Mail\Mailables\Envelope;
    use Symfony\Component\Mime\Email;

    /**
     * Get the message envelope.
     */
    public function envelope(): Envelope
    {
        return new Envelope(
            subject: 'Order Shipped',
            using: [
                function (Email $message) {
                    // ...
                },
            ]
        );
    }

<a name="markdown-mailables"></a>
## Markdown Mailables

Markdown mailable 訊息讓您可以在您的 mailables 中利用 [郵件通知](/docs/{{version}}/notifications#mail-notifications) 的預建範本和元件。由於訊息以 Markdown 編寫，Laravel 能夠為這些訊息渲染出美觀、響應式的 HTML 範本，同時也會自動生成對應的純文字版本。


<a name="generating-markdown-mailables"></a>
### 生成 Markdown Mailables

若要生成一個帶有對應 Markdown 範本的 mailable，您可以使用 `make:mail` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

接著，當在 mailable 的 `content` 方法中設定 `Content` 定義時，請使用 `markdown` 參數而不是 `view` 參數：

    use Illuminate\Mail\Mailables\Content;

    /**
     * Get the message content definition.
     */
    public function content(): Content
    {
        return new Content(
            markdown: 'mail.orders.shipped',
            with: [
                'url' => $this->orderUrl,
            ],
        );
    }


<a name="writing-markdown-messages"></a>
### 編寫 Markdown 訊息

Markdown mailables 結合了 Blade 元件和 Markdown 語法，讓您可以輕鬆建構郵件訊息，同時利用 Laravel 預建的電子郵件 UI 元件：

```blade
<x-mail::message>
# Order Shipped

Your order has been shipped!

<x-mail::button :url="$url">
View Order
</x-mail::button>

Thanks,<br>
{{ config('app.name') }}
</x-mail::message>
```

> [!NOTE]  
> 編寫 Markdown 電子郵件時，請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染一個置中的按鈕連結。此元件接受兩個引數：一個 `url` 和一個可選的 `color`。支援的顏色有 `primary`、`success` 和 `error`。您可以在訊息中加入任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板元件

面板元件會將指定的文字區塊渲染在一個面板中，該面板的背景顏色與訊息的其餘部分略有不同。這讓您可以將注意力吸引到特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格元件

表格元件可讓您將 Markdown 表格轉換為 HTML 表格。此元件將 Markdown 表格作為其內容。表格欄位對齊方式支援使用預設的 Markdown 表格對齊語法：

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

您可以將所有 Markdown 郵件元件匯出到您的應用程式中進行自訂。若要匯出這些元件，請使用 `vendor:publish` Artisan 命令來發布 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件元件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含 `html` 和 `text` 目錄，每個目錄都包含所有可用元件的各自表示。您可以隨意自訂這些元件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將會自動轉換為 Markdown 郵件訊息 HTML 表示中的行內 CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建立一個全新的主題，您可以將一個 CSS 檔案放置在 `html/themes` 目錄中。在命名並儲存您的 CSS 檔案後，請更新您應用程式 `config/mail.php` 設定檔中的 `theme` 選項，以匹配您新主題的名稱。

若要為單獨的 mailable 自訂主題，您可以將 mailable 類別的 `$theme` 屬性設定為傳送該 mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 傳送郵件

要傳送訊息，請在 `Mail` [Facade](/docs/{{version}}/facades) 上使用 `to` 方法。`to` 方法接受電子郵件位址、使用者實例或使用者集合。如果您傳遞一個物件或物件集合，郵件發送器將在確定郵件收件者時自動使用其 `email` 和 `name` 屬性，因此請確保這些屬性在您的物件上可用。指定收件者後，您可以將您的 mailable 類別實例傳遞給 `send` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Mail\OrderShipped;
    use App\Models\Order;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Mail;

    class OrderShipmentController extends Controller
    {
        /**
         * Ship the given order.
         */
        public function store(Request $request): RedirectResponse
        {
            $order = Order::findOrFail($request->order_id);

            // Ship the order...

            Mail::to($request->user())->send(new OrderShipped($order));

            return redirect('/orders');
        }
    }

傳送訊息時，您不限於僅指定「to」收件者。您可以透過鏈接它們各自的方法來設定「to」、「cc」和「bcc」收件者：

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->send(new OrderShipped($order));


<a name="looping-over-recipients"></a>
#### 循環收件者

有時，您可能需要透過迭代收件者/電子郵件位址陣列，將 Mailable 傳送給收件者列表。然而，由於 `to` 方法會將電子郵件位址附加到 Mailable 的收件者列表，因此每次循環迭代都會向每個先前的收件者傳送另一封電子郵件。因此，您應該始終為每個收件者重新建立 Mailable 實例：

    foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
        Mail::to($recipient)->send(new OrderShipped($order));
    }


<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過特定郵件發送器傳送郵件

預設情況下，Laravel 將使用您應用程式 `mail` 設定檔中設定為 `default` 郵件發送器來傳送電子郵件。但是，您可以使用 `mailer` 方法來使用特定的郵件發送器設定來傳送訊息：

    Mail::mailer('postmark')
        ->to($request->user())
        ->send(new OrderShipped($order));


<a name="queueing-mail"></a>
### 佇列郵件


<a name="queueing-a-mail-message"></a>
#### 佇列郵件訊息

由於傳送電子郵件訊息可能會對應用程式的回應時間產生負面影響，因此許多開發人員選擇將電子郵件訊息排入佇列以進行背景傳送。Laravel 透過其內建的[統一佇列 API](/docs/{{version}}/queues) 輕鬆實現這一點。要將郵件訊息排入佇列，請在指定訊息收件者後，在 `Mail` facade 上使用 `queue` 方法：

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->queue(new OrderShipped($order));

此方法會自動將一個任務推送到佇列，以便在背景傳送訊息。在使用此功能之前，您需要[設定您的佇列](/docs/{{version}}/queues)。


<a name="delayed-message-queueing"></a>
#### 延遲佇列訊息

如果您希望延遲佇列電子郵件訊息的傳送，您可以使用 `later` 方法。作為其第一個引數，`later` 方法接受一個 `DateTime` 實例，指示應何時傳送訊息：

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->later(now()->addMinutes(10), new OrderShipped($order));


<a name="pushing-to-specific-queues"></a>
#### 推送到特定佇列

由於所有使用 `make:mail` 命令生成的 Mailable 類別都使用了 `Illuminate\Bus\Queueable` Trait，因此您可以在任何 Mailable 類別實例上呼叫 `onQueue` 和 `onConnection` 方法，讓您可以為訊息指定連線和佇列名稱：

    $message = (new OrderShipped($order))
        ->onConnection('sqs')
        ->onQueue('emails');

    Mail::to($request->user())
        ->cc($moreUsers)
        ->bcc($evenMoreUsers)
        ->queue($message);


<a name="queueing-by-default"></a>
#### 預設佇列

如果您有希望始終排入佇列的 Mailable 類別，則可以在該類別上實作 `ShouldQueue` 契約。現在，即使您在傳送郵件時呼叫 `send` 方法，由於它實作了該契約，Mailable 仍將排入佇列：

    use Illuminate\Contracts\Queue\ShouldQueue;

    class OrderShipped extends Mailable implements ShouldQueue
    {
        // ...
    }


<a name="queued-mailables-and-database-transactions"></a>
#### 佇列 Mailable 與資料庫事務

當佇列 Mailable 在資料庫事務中分派時，它們可能會在資料庫事務提交之前被佇列處理。發生這種情況時，您在資料庫事務期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在事務中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的 Mailable 依賴這些模型，則在處理傳送佇列 Mailable 的任務時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設定為 `false`，您仍然可以透過在傳送郵件訊息時呼叫 `afterCommit` 方法，指示特定的佇列 Mailable 應在所有開啟的資料庫事務提交後分派：

    Mail::to($request->user())->send(
        (new OrderShipped($order))->afterCommit()
    );

或者，您可以在 Mailable 的建構函式中呼叫 `afterCommit` 方法：

    <?php

    namespace App\Mail;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Mail\Mailable;
    use Illuminate\Queue\SerializesModels;

    class OrderShipped extends Mailable implements ShouldQueue
    {
        use Queueable, SerializesModels;

        /**
         * Create a new message instance.
         */
        public function __construct()
        {
            $this->afterCommit();
        }
    }

> [!NOTE]  
> 要了解如何解決這些問題的更多資訊，請查閱有關[佇列任務和資料庫事務](/docs/{{version}}/queues#jobs-and-database-transactions)的文檔。


<a name="rendering-mailables"></a>
## 渲染 Mailables

有時，您可能希望擷取 Mailable 的 HTML 內容而不傳送它。為此，您可以呼叫 Mailable 的 `render` 方法。此方法將以字串形式傳回 Mailable 的評估 HTML 內容：

    use App\Mail\InvoicePaid;
    use App\Models\Invoice;

    $invoice = Invoice::find(1);

    return (new InvoicePaid($invoice))->render();


<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailables

在設計 Mailable 範本時，在瀏覽器中像典型的 Blade 範本一樣快速預覽渲染的 Mailable 是很方便的。因此，Laravel 允許您直接從路由閉包或控制器傳回任何 Mailable。當傳回 Mailable 時，它將被渲染並顯示在瀏覽器中，讓您可以快速預覽其設計，而無需將其傳送給實際的電子郵件位址：

    Route::get('/mailable', function () {
        $invoice = App\Models\Invoice::find(1);

        return new App\Mail\InvoicePaid($invoice);
    });

<a name="localizing-mailables"></a>
## 本地化 Mailables

Laravel 允許您以不同於目前請求 (request) 的語系 (locale) 傳送 Mailable，即使郵件已佇列 (queued)，它也會記住這個語系。

為此，`Mail` Facade 提供 `locale` 方法來設定所需的語言。當 Mailable 的模板 (template) 正在被評估時，應用程式會切換到這個語系，並在評估完成後恢復到先前的語系：

    Mail::to($request->user())->locale('es')->send(
        new OrderShipped($order)
    );

<a name="user-preferred-locales"></a>
### 使用者偏好語系

有時，應用程式會儲存每個使用者偏好的語系。透過在您的一個或多個模型上實作 `HasLocalePreference` 契約 (contract)，您可以指示 Laravel 在傳送郵件時使用此儲存的語系：

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

一旦您實作了該介面 (interface)，Laravel 將在向模型傳送 Mailable 和通知 (notifications) 時自動使用偏好語系 (locale)。因此，使用此介面時，無需呼叫 `locale` 方法：

    Mail::to($request->user())->send(new OrderShipped($order));

<a name="testing-mailables"></a>
## 測試

<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了多種方法來檢查您的 Mailable 結構。此外，Laravel 還提供了幾種方便的方法來測試您的 Mailable 是否包含您預期的內容。這些方法包括：`assertSeeInHtml`、`assertDontSeeInHtml`、`assertSeeInOrderInHtml`、`assertSeeInText`、`assertDontSeeInText`、`assertSeeInOrderInText`、`assertHasAttachment`、`assertHasAttachedData`、`assertHasAttachmentFromStorage` 和 `assertHasAttachmentFromStorageDisk`。

正如您可能預期的，「HTML」斷言會判斷您的 Mailable HTML 版本是否包含給定的字串，而「text」斷言會判斷您的 Mailable 純文字版本是否包含給定的字串：

```php tab=Pest
use App\Mail\InvoicePaid;
use App\Models\User;

test('mailable content', function () {
    $user = User::factory()->create();

    $mailable = new InvoicePaid($user);

    $mailable->assertFrom('jeffrey@example.com');
    $mailable->assertTo('taylor@example.com');
    $mailable->assertHasCc('abigail@example.com');
    $mailable->assertHasBcc('victoria@example.com');
    $mailable->assertHasReplyTo('tyler@example.com');
    $mailable->assertHasSubject('Invoice Paid');
    $mailable->assertHasTag('example-tag');
    $mailable->assertHasMetadata('key', 'value');

    $mailable->assertSeeInHtml($user->email);
    $mailable->assertSeeInHtml('Invoice Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertSeeInOrderInText(['Invoice Paid', 'Thanks']);

    $mailable->assertHasAttachment('/path/to/file');
    $mailable->assertHasAttachment(Attachment::fromPath('/path/to/file'));
    $mailable->assertHasAttachedData($pdfData, 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorage('/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorageDisk('s3', '/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
});
```

```php tab=PHPUnit
use App\Mail\InvoicePaid;
use App\Models\User;

public function test_mailable_content(): void
{
    $user = User::factory()->create();

    $mailable = new InvoicePaid($user);

    $mailable->assertFrom('jeffrey@example.com');
    $mailable->assertTo('taylor@example.com');
    $mailable->assertHasCc('abigail@example.com');
    $mailable->assertHasBcc('victoria@example.com');
    $mailable->assertHasReplyTo('tyler@example.com');
    $mailable->assertHasSubject('Invoice Paid');
    $mailable->assertHasTag('example-tag');
    $mailable->assertHasMetadata('key', 'value');

    $mailable->assertSeeInHtml($user->email);
    $mailable->assertSeeInHtml('Invoice Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertSeeInOrderInText(['Invoice Paid', 'Thanks']);

    $mailable->assertHasAttachment('/path/to/file');
    $mailable->assertHasAttachment(Attachment::fromPath('/path/to/file'));
    $mailable->assertHasAttachedData($pdfData, 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorage('/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorageDisk('s3', '/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
}
```

<a name="testing-mailable-sending"></a>
### 測試 Mailable 傳送

我們建議測試 Mailable 的內容與斷言特定 Mailable 已「傳送」給特定使用者的測試分開進行。通常，Mailable 的內容與您正在測試的程式碼無關，並且僅斷言 Laravel 已收到傳送指定 Mailable 的指令就已足夠。

您可以將 `Mail` facade 的 `fake` 方法用來阻止郵件被實際傳送。呼叫 `Mail` facade 的 `fake` 方法後，您可以斷言已指示要傳送 Mailable 給使用者，甚至可以檢查 Mailable 接收到的資料：

```php tab=Pest
<?php

use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;

test('orders can be shipped', function () {
    Mail::fake();

    // Perform order shipping...

    // Assert that no mailables were sent...
    Mail::assertNothingSent();

    // Assert that a mailable was sent...
    Mail::assertSent(OrderShipped::class);

    // Assert a mailable was sent twice...
    Mail::assertSent(OrderShipped::class, 2);

    // Assert a mailable was sent to an email address...
    Mail::assertSent(OrderShipped::class, 'example@laravel.com');

    // Assert a mailable was sent to multiple email addresses...
    Mail::assertSent(OrderShipped::class, ['example@laravel.com', '...']);

    // Assert a mailable was not sent...
    Mail::assertNotSent(AnotherMailable::class);

    // Assert 3 total mailables were sent...
    Mail::assertSentCount(3);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_orders_can_be_shipped(): void
    {
        Mail::fake();

        // Perform order shipping...

        // Assert that no mailables were sent...
        Mail::assertNothingSent();

        // Assert that a mailable was sent...
        Mail::assertSent(OrderShipped::class);

        // Assert a mailable was sent twice...
        Mail::assertSent(OrderShipped::class, 2);

        // Assert a mailable was sent to an email address...
        Mail::assertSent(OrderShipped::class, 'example@laravel.com');

        // Assert a mailable was sent to multiple email addresses...
        Mail::assertSent(OrderShipped::class, ['example@laravel.com', '...']);

        // Assert a mailable was not sent...
        Mail::assertNotSent(AnotherMailable::class);

        // Assert 3 total mailables were sent...
        Mail::assertSentCount(3);
    }
}
```

如果您正在將 Mailable 排入佇列以在背景傳送，您應該使用 `assertQueued` 方法而不是 `assertSent`：

    Mail::assertQueued(OrderShipped::class);
    Mail::assertNotQueued(OrderShipped::class);
    Mail::assertNothingQueued();
    Mail::assertQueuedCount(3);

您可以將閉包傳遞給 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以斷言傳送的 Mailable 通過給定的「真值測試 (truth test)」。如果至少有一個通過給定真值測試的 Mailable 被傳送，則該斷言將會成功：

    Mail::assertSent(function (OrderShipped $mail) use ($order) {
        return $mail->order->id === $order->id;
    });

當呼叫 `Mail` facade 的斷言方法時，所提供閉包接受的 Mailable 實例會公開有助於檢查 Mailable 的方法：

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($user) {
        return $mail->hasTo($user->email) &&
               $mail->hasCc('...') &&
               $mail->hasBcc('...') &&
               $mail->hasReplyTo('...') &&
               $mail->hasFrom('...') &&
               $mail->hasSubject('...');
    });

Mailable 實例也包含幾個有助於檢查 Mailable 上附件的方法：

    use Illuminate\Mail\Mailables\Attachment;

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) {
        return $mail->hasAttachment(
            Attachment::fromPath('/path/to/file')
                ->as('name.pdf')
                ->withMime('application/pdf')
        );
    });

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) {
        return $mail->hasAttachment(
            Attachment::fromStorageDisk('s3', '/path/to/file')
        );
    });

    Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($pdfData) {
        return $mail->hasAttachment(
            Attachment::fromData(fn () => $pdfData, 'name.pdf')
        );
    });

您可能已經注意到，有兩種方法可以斷言郵件未傳送：`assertNotSent` 和 `assertNotQueued`。有時您可能希望斷言沒有郵件被傳送**或**排入佇列。為了實現這一點，您可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

    Mail::assertNothingOutgoing();

    Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
        return $mail->order->id === $order->id;
    });

<a name="mail-and-local-development"></a>
## 郵件與本地開發

當開發會傳送電子郵件的應用程式時，你可能不希望真的將電子郵件傳送給真實的電子郵件地址。Laravel 提供多種方法，可以在本地開發期間「停用」電子郵件的實際傳送。

<a name="log-driver"></a>
#### Log 驅動程式

`log` 郵件驅動程式不會傳送你的電子郵件，而是將所有電子郵件訊息寫入你的記錄檔以供檢查。通常，此驅動程式僅在本地開發期間使用。有關依環境設定應用程式的更多資訊，請查閱 [設定文件](/docs/{{version}}/configuration#environment-configuration)。

<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，你也可以使用 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 等服務以及 `smtp` 驅動程式，將電子郵件訊息傳送至「虛擬」信箱，你可以在真正的電子郵件用戶端中查看這些郵件。這種方法的好處是允許你實際檢查 Mailtrap 訊息檢視器中的最終電子郵件。

如果你正在使用 [Laravel Sail](/docs/{{version}}/sail)，你可以使用 [Mailpit](https://github.com/axllent/mailpit) 預覽你的訊息。當 Sail 運行時，你可以透過 `http://localhost:8025` 存取 Mailpit 介面。

<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 地址

最後，你可以透過呼叫 `Mail` Facade 提供的 `alwaysTo` 方法來指定一個全域「to」地址。通常，這個方法應該在應用程式的某個服務提供者的 `boot` 方法中呼叫：

    use Illuminate\Support\Facades\Mail;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        if ($this->app->environment('local')) {
            Mail::alwaysTo('taylor@example.com');
        }
    }

<a name="events"></a>
## 事件

Laravel 在傳送郵件訊息時會派發兩個事件。`MessageSending` 事件會在訊息傳送之前派發，而 `MessageSent` 事件則在訊息傳送之後派發。請記住，這些事件是在郵件被 *傳送* 時派發的，而不是在它被佇列時。你可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

    use Illuminate\Mail\Events\MessageSending;
    // use Illuminate\Mail\Events\MessageSent;

    class LogMessage
    {
        /**
         * Handle the given event.
         */
        public function handle(MessageSending $event): void
        {
            // ...
        }
    }

<a name="custom-transports"></a>
## 自訂傳輸器

Laravel 包含各種郵件傳輸器；但是，你可能希望編寫自己的傳輸器，透過 Laravel 不開箱即用的其他服務來傳遞電子郵件。首先，定義一個繼承 `Symfony\Component\Mailer\Transport\AbstractTransport` 類別的類別。然後，在你的傳輸器上實作 `doSend` 和 `__toString()` 方法：

    use MailchimpTransactional\ApiClient;
    use Symfony\Component\Mailer\SentMessage;
    use Symfony\Component\Mailer\Transport\AbstractTransport;
    use Symfony\Component\Mime\Address;
    use Symfony\Component\Mime\MessageConverter;

    class MailchimpTransport extends AbstractTransport
    {
        /**
         * Create a new Mailchimp transport instance.
         */
        public function __construct(
            protected ApiClient $client,
        ) {
            parent::__construct();
        }

        /**
         * {@inheritDoc}
         */
        protected function doSend(SentMessage $message): void
        {
            $email = MessageConverter::toEmail($message->getOriginalMessage());

            $this->client->messages->send(['message' => [
                'from_email' => $email->getFrom(),
                'to' => collect($email->getTo())->map(function (Address $email) {
                    return ['email' => $email->getAddress(), 'type' => 'to'];
                })->all(),
                'subject' => $email->getSubject(),
                'text' => $email->getTextBody(),
            ]]);
        }

        /**
         * Get the string representation of the transport.
         */
        public function __toString(): string
        {
            return 'mailchimp';
        }
    }

定義了自訂傳輸器後，你可以透過 `Mail` Facade 提供的 `extend` 方法來註冊它。通常，這應該在應用程式的 `AppServiceProvider` 服務提供者的 `boot` 方法中完成。一個 `$config` 引數將傳遞給提供給 `extend` 方法的閉包。這個引數將包含為應用程式 `config/mail.php` 設定檔中的郵件發送器定義的設定陣列：

    use App\Mail\MailchimpTransport;
    use Illuminate\Support\Facades\Mail;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Mail::extend('mailchimp', function (array $config = []) {
            return new MailchimpTransport(/* ... */);
        });
    }

一旦你的自訂傳輸器被定義並註冊後，你可以在應用程式的 `config/mail.php` 設定檔中建立一個使用新傳輸器的郵件發送器定義：

    'mailchimp' => [
        'transport' => 'mailchimp',
        // ...
    ],

<a name="additional-symfony-transports"></a>
### 額外 Symfony 傳輸器

Laravel 支援一些現有的 Symfony 維護的郵件傳輸器，如 Mailgun 和 Postmark。然而，你可能希望擴展 Laravel 以支援額外的 Symfony 維護的傳輸器。你可以透過 Composer 引入必要的 Symfony 郵件發送器，並向 Laravel 註冊該傳輸器。例如，你可以安裝並註冊「Brevo」（前稱「Sendinblue」）Symfony 郵件發送器：

```none
composer require symfony/brevo-mailer symfony/http-client
```

一旦 Brevo 郵件發送器套件安裝完成，你可以在應用程式的 `services` 設定檔中為你的 Brevo API 憑證新增一個項目：

    'brevo' => [
        'key' => 'your-api-key',
    ],

接下來，你可以使用 `Mail` Facade 的 `extend` 方法向 Laravel 註冊該傳輸器。通常，這應該在服務提供者的 `boot` 方法中完成：

    use Illuminate\Support\Facades\Mail;
    use Symfony\Component\Mailer\Bridge\Brevo\Transport\BrevoTransportFactory;
    use Symfony\Component\Mailer\Transport\Dsn;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Mail::extend('brevo', function () {
            return (new BrevoTransportFactory)->create(
                new Dsn(
                    'brevo+api',
                    'default',
                    config('services.brevo.key')
                )
            );
        });
    }

一旦你的傳輸器被註冊，你可以在應用程式的 config/mail.php 設定檔中建立一個使用新傳輸器的郵件發送器定義：

    'brevo' => [
        'transport' => 'brevo',
        // ...
    ],