# 郵件

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
    - [容錯移轉設定](#failover-configuration)
    - [循環設定](#round-robin-configuration)
- [產生 Mailable](#generating-mailables)
- [撰寫 Mailable](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [內嵌附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與中繼資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailable](#markdown-mailables)
    - [產生 Markdown Mailable](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
    - [自訂元件](#customizing-the-components)
- [寄送郵件](#sending-mail)
    - [郵件佇列](#queueing-mail)
- [渲染 Mailable](#rendering-mailables)
    - [在瀏覽器中預覽 Mailable](#previewing-mailables-in-the-browser)
- [Mailable 本地化](#localizing-mailables)
- [測試](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 寄送](#testing-mailable-sending)
- [郵件與本地開發](#mail-and-local-development)
- [事件](#events)
- [自訂傳輸器](#custom-transports)
    - [額外的 Symfony 傳輸器](#additional-symfony-transports)

<a name="introduction"></a>
## 簡介

寄送電子郵件不應該是件複雜的事。Laravel 提供了一個簡潔、簡單的電子郵件 API，由廣受歡迎的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件提供支援。Laravel 和 Symfony Mailer 提供了透過 SMTP、Mailgun、Postmark、Resend、Amazon SES 和 `sendmail` 寄送電子郵件的驅動程式，讓您可以快速開始透過您選擇的本地或雲端服務寄送郵件。

<a name="configuration"></a>
### 設定

Laravel 的電子郵件服務可以透過應用程式的 `config/mail.php` 設定檔進行設定。此檔案中設定的每個郵件發送器都可以有自己獨特的設定，甚至有自己獨特的「傳輸器 (transport)」，讓您的應用程式可以使用不同的電子郵件服務來寄送特定的電子郵件訊息。例如，您的應用程式可能使用 Postmark 寄送交易郵件，同時使用 Amazon SES 寄送大量郵件。

在您的 `mail` 設定檔中，您會找到一個 `mailers` 設定陣列。此陣列包含 Laravel 支援的每個主要郵件驅動程式/傳輸器的範例設定項目，而 `default` 設定值則決定了當您的應用程式需要寄送電子郵件訊息時，預設將使用哪個郵件發送器。

<a name="driver-prerequisites"></a>
### 驅動程式/傳輸器先決條件

基於 API 的驅動程式，例如 Mailgun、Postmark、Resend 和 MailerSend，通常比透過 SMTP 伺服器寄送郵件更簡單、更快。只要有可能，我們建議您使用這些驅動程式之一。

<a name="mailgun-driver"></a>
#### Mailgun 驅動程式

要使用 Mailgun 驅動程式，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳輸器：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接下來，您需要在應用程式的 `config/mail.php` 設定檔中進行兩項變更。首先，將您的預設郵件發送器設定為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

其次，將以下設定陣列新增到您的 `mailers` 陣列中：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

設定應用程式的預設郵件發送器後，將以下選項新增到您的 `config/services.php` 設定檔中：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

如果您沒有使用美國 [Mailgun 區域](https://documentation.mailgun.com/docs/mailgun/api-reference/#mailgun-regions)，您可以在 `services` 設定檔中定義您區域的端點：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.eu.mailgun.net'),
    'scheme' => 'https',
],
```

<a name="postmark-driver"></a>
#### Postmark 驅動程式

要使用 [Postmark](https://postmarkapp.com/) 驅動程式，請透過 Composer 安裝 Symfony 的 Postmark Mailer 傳輸器：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接下來，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `postmark`。設定應用程式的預設郵件發送器後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'postmark' => [
    'token' => env('POSTMARK_TOKEN'),
],
```

如果您想指定給定郵件發送器應使用的 Postmark 訊息串流，您可以將 `message_stream_id` 設定選項新增到郵件發送器的設定陣列中。此設定陣列可以在應用程式的 `config/mail.php` 設定檔中找到：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

這樣您也可以設定多個具有不同訊息串流的 Postmark 郵件發送器。

<a name="resend-driver"></a>
#### Resend 驅動程式

要使用 [Resend](https://resend.com/) 驅動程式，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接下來，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `resend`。設定應用程式的預設郵件發送器後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'resend' => [
    'key' => env('RESEND_KEY'),
],
```

<a name="ses-driver"></a>
#### SES 驅動程式

要使用 Amazon SES 驅動程式，您必須先安裝適用於 PHP 的 Amazon AWS SDK。您可以透過 Composer 套件管理器安裝此函式庫：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `config/mail.php` 設定檔中的 `default` 選項設定為 `ses`，並驗證您的 `config/services.php` 設定檔包含以下選項：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

要透過工作階段權杖使用 AWS [臨時憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，您可以將 `token` 鍵新增到應用程式的 SES 設定中：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

要與 SES 的 [訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html) 互動，您可以透過郵件訊息的 [headers](#headers) 方法返回的陣列中返回 `X-Ses-List-Management-Options` 標頭：

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

如果您想定義 Laravel 在寄送電子郵件時應傳遞給 AWS SDK 的 `SendEmail` 方法的 [其他選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，您可以在 SES 設定中定義一個 `options` 陣列：

```php
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
```

<a name="mailersend-driver"></a>
#### MailerSend 驅動程式

[MailerSend](https://www.mailersend.com/) 是一個交易電子郵件和 SMS 服務，它維護自己的基於 API 的 Laravel 郵件驅動程式。包含該驅動程式的套件可以透過 Composer 套件管理器安裝：

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

要了解有關 MailerSend 的更多資訊，包括如何使用託管範本，請參閱 [MailerSend 驅動程式文件](https://github.com/mailersend/mailersend-laravel-driver#usage)。

<a name="failover-configuration"></a>
### 容錯移轉設定

有時，您設定用於寄送應用程式郵件的外部服務可能會中斷。在這些情況下，定義一個或多個備用郵件傳遞設定會很有用，以便在主要傳遞驅動程式中斷時使用。

為此，您應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳輸器的郵件發送器。應用程式 `failover` 郵件發送器的設定陣列應包含一個 `mailers` 陣列，該陣列引用了應選擇用於傳遞的已設定郵件發送器的順序：

```php
'mailers' => [
    'failover' => [
        'transport' => 'failover',
        'mailers' => [
            'postmark',
            'mailgun',
            'sendmail',
        ],
        'retry_after' => 60,
    ],

    // ...
],
```

定義容錯移轉郵件發送器後，您應該將此郵件發送器設定為應用程式使用的預設郵件發送器，方法是將其名稱指定為應用程式 `mail` 設定檔中 `default` 設定鍵的值：

```php
'default' => env('MAIL_MAILER', 'failover'),
```

<a name="round-robin-configuration"></a>
### 循環設定

`roundrobin` 傳輸器允許您將郵件工作負載分發到多個郵件發送器。首先，在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` 傳輸器的郵件發送器。應用程式 `roundrobin` 郵件發送器的設定陣列應包含一個 `mailers` 陣列，該陣列引用了應選擇用於傳遞的已設定郵件發送器：

```php
'mailers' => [
    'roundrobin' => [
        'transport' => 'roundrobin',
        'mailers' => [
            'ses',
            'postmark',
        ],
        'retry_after' => 60,
    ],

    // ...
],
```

定義循環郵件發送器後，您應該將此郵件發送器設定為應用程式使用的預設郵件發送器，方法是將其名稱指定為應用程式 `mail` 設定檔中 `default` 設定鍵的值：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

循環傳輸器從已設定的郵件發送器列表中隨機選擇一個郵件發送器，然後為每個後續電子郵件切換到下一個可用的郵件發送器。與有助於實現 [高可用性](https://en.wikipedia.org/wiki/High_availability) 的 `failover` 傳輸器不同，`roundrobin` 傳輸器提供 [負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))。

<a name="generating-mailables"></a>
## 產生 Mailable

在建構 Laravel 應用程式時，應用程式寄送的每種電子郵件類型都表示為一個「mailable」類別。這些類別儲存在 `app/Mail` 目錄中。如果您在應用程式中沒有看到此目錄，請不要擔心，因為當您使用 `make:mail` Artisan 命令建立第一個 mailable 類別時，它將為您產生：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailable

產生 mailable 類別後，打開它以便我們探索其內容。Mailable 類別設定在多個方法中完成，包括 `envelope`、`content` 和 `attachments` 方法。

`envelope` 方法返回一個 `Illuminate\Mail\Mailables\Envelope` 物件，該物件定義了訊息的主旨，有時還定義了收件人。`content` 方法返回一個 `Illuminate\Mail\Mailables\Content` 物件，該物件定義了將用於產生訊息內容的 [Blade 範本](/docs/{{version}}/blade)。

<a name="configuring-the-sender"></a>
### 設定寄件者

<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，讓我們探索如何設定電子郵件的寄件者。換句話說，電子郵件將「來自」誰。有兩種方法可以設定寄件者。首先，您可以在訊息的 envelope 上指定「寄件者」位址：

```php
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
```

如果您願意，您也可以指定一個 `replyTo` 位址：

```php
return new Envelope(
    from: new Address('jeffrey@example.com', 'Jeffrey Way'),
    replyTo: [
        new Address('taylor@example.com', 'Taylor Otwell'),
    ],
    subject: 'Order Shipped',
);
```

<a name="using-a-global-from-address"></a>
#### 使用全域 `from` 位址

然而，如果您的應用程式對所有電子郵件都使用相同的「寄件者」位址，那麼將其新增到您產生的每個 mailable 類別中可能會很麻煩。相反，您可以在 `config/mail.php` 設定檔中指定一個全域「寄件者」位址。如果 mailable 類別中沒有指定其他「寄件者」位址，則將使用此位址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，您可以在 `config/mail.php` 設定檔中定義一個全域「reply_to」位址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```

<a name="configuring-the-view"></a>
### 設定視圖

在 mailable 類別的 `content` 方法中，您可以定義 `view`，或者在渲染電子郵件內容時應使用哪個範本。由於每封電子郵件通常使用 [Blade 範本](/docs/{{version}}/blade) 來渲染其內容，因此在建構電子郵件的 HTML 時，您擁有 Blade 範本引擎的全部功能和便利性：

```php
/**
 * Get the message content definition.
 */
public function content(): Content
{
    return new Content(
        view: 'mail.orders.shipped',
    );
}
```

> [!NOTE]
> 您可能希望建立一個 `resources/views/mail` 目錄來存放所有電子郵件範本；但是，您可以將它們放置在 `resources/views` 目錄中的任何位置。

<a name="plain-text-emails"></a>
#### 純文字電子郵件

如果您想定義電子郵件的純文字版本，您可以在建立訊息的 `Content` 定義時指定純文字範本。與 `view` 參數一樣，`text` 參數應該是一個範本名稱，該名稱將用於渲染電子郵件的內容。您可以自由定義訊息的 HTML 和純文字版本：

```php
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
```

為了清晰起見，`html` 參數可以用作 `view` 參數的別名：

```php
return new Content(
    html: 'mail.orders.shipped',
    text: 'mail.orders.shipped-text'
);
```

<a name="view-data"></a>
### 視圖資料

<a name="via-public-properties"></a>
#### 透過 Public 屬性

通常，您會希望將一些資料傳遞給您的視圖，以便在渲染電子郵件的 HTML 時使用。有兩種方法可以使資料可供您的視圖使用。首先，在您的 mailable 類別上定義的任何 public 屬性都將自動可供視圖使用。因此，例如，您可以將資料傳遞到 mailable 類別的建構函式中，並將該資料設定為類別上定義的 public 屬性：

```php
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
```

一旦資料設定為 public 屬性，它將自動在您的視圖中可用，因此您可以像存取 Blade 範本中的任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $order->price }}
</div>
```

<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果您想在電子郵件資料傳遞到範本之前自訂其格式，您可以透過 `Content` 定義的 `with` 參數手動將資料傳遞給視圖。通常，您仍然會透過 mailable 類別的建構函式傳遞資料；但是，您應該將此資料設定為 `protected` 或 `private` 屬性，以便資料不會自動可供範本使用：

```php
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
```

一旦資料透過 `with` 參數傳遞，它將自動在您的視圖中可用，因此您可以像存取 Blade 範本中的任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

要將附件新增到電子郵件中，您需要將附件新增到訊息 `attachments` 方法返回的陣列中。首先，您可以透過向 `Attachment` 類別提供的 `fromPath` 方法提供檔案路徑來新增附件：

```php
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
```

將檔案附加到訊息時，您還可以使用 `as` 和 `withMime` 方法指定附件的顯示名稱和/或 MIME 類型：

```php
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
```

<a name="attaching-files-from-disk"></a>
#### 從磁碟附加檔案

如果您已將檔案儲存在其中一個 [檔案系統磁碟](/docs/{{version}}/filesystem) 上，則可以使用 `fromStorage` 附件方法將其附加到電子郵件中：

```php
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
```

當然，您也可以指定附件的名稱和 MIME 類型：

```php
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
```

如果您需要指定除預設磁碟之外的儲存磁碟，可以使用 `fromStorageDisk` 方法：

```php
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
```

<a name="raw-data-attachments"></a>
#### 原始資料附件

`fromData` 附件方法可用於將原始位元組字串作為附件附加。例如，如果您在記憶體中產生了 PDF 並希望將其附加到電子郵件中而無需將其寫入磁碟，則可以使用此方法。`fromData` 方法接受一個閉包，該閉包解析原始資料位元組以及應分配給附件的名稱：

```php
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
```

<a name="inline-attachments"></a>
### 內嵌附件

將內嵌圖片嵌入電子郵件通常很麻煩；然而，Laravel 提供了一種方便的方法來將圖片附加到您的電子郵件中。要嵌入內嵌圖片，請在電子郵件範本中使用 `$message` 變數上的 `embed` 方法。Laravel 會自動將 `$message` 變數提供給所有電子郵件範本，因此您無需擔心手動傳遞它：

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]
> `$message` 變數在純文字訊息範本中不可用，因為純文字訊息不使用內嵌附件。

<a name="embedding-raw-data-attachments"></a>
#### 嵌入原始資料附件

如果您已經有一個要嵌入到電子郵件範本中的原始圖片資料字串，您可以呼叫 `$message` 變數上的 `embedData` 方法。呼叫 `embedData` 方法時，您需要提供一個應分配給嵌入圖片的檔案名稱：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```

<a name="attachable-objects"></a>
### 可附加物件

雖然透過簡單的字串路徑將檔案附加到訊息通常就足夠了，但在許多情況下，應用程式中可附加的實體是由類別表示的。例如，如果您的應用程式正在將照片附加到訊息中，您的應用程式可能還有一個表示該照片的 `Photo` 模型。在這種情況下，如果能簡單地將 `Photo` 模型傳遞給 `attach` 方法，那不是很方便嗎？可附加物件允許您做到這一點。

首先，在將可附加到訊息的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。此介面規定您的類別定義一個 `toMailAttachment` 方法，該方法返回一個 `Illuminate\Mail\Attachment` 實例：

```php
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
```

定義可附加物件後，您可以在建構電子郵件訊息時從 `attachments` 方法返回該物件的實例：

```php
/**
 * Get the attachments for the message.
 *
 * @return array<int, \Illuminate\Mail\Mailables\Attachment>
 */
public function attachments(): array
{
    return [$this->photo];
}
```

當然，附件資料可以儲存在遠端檔案儲存服務上，例如 Amazon S3。因此，Laravel 還允許您從儲存在應用程式 [檔案系統磁碟](/docs/{{version}}/filesystem) 上的資料產生附件實例：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，您可以透過記憶體中的資料建立附件實例。為此，請向 `fromData` 方法提供一個閉包。該閉包應返回表示附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 還提供了其他方法，您可以用來自訂附件。例如，您可以使用 `as` 和 `withMime` 方法來自訂檔案的名稱和 MIME 類型：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```

<a name="headers"></a>
### 標頭

有時您可能需要將額外的標頭附加到外寄訊息。例如，您可能需要設定自訂的 `Message-Id` 或其他任意文字標頭。

為此，請在您的 mailable 上定義一個 `headers` 方法。`headers` 方法應返回一個 `Illuminate\Mail\Mailables\Headers` 實例。此類別接受 `messageId`、`references` 和 `text` 參數。當然，您可以只提供您特定訊息所需的參數：

```php
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
```

<a name="tags-and-metadata"></a>
### 標籤與中繼資料

一些第三方電子郵件提供商，例如 Mailgun 和 Postmark，支援訊息「標籤」和「中繼資料」，可用於對應用程式寄送的電子郵件進行分組和追蹤。您可以透過 `Envelope` 定義將標籤和中繼資料新增到電子郵件訊息中：

```php
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
```

如果您的應用程式正在使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件以獲取有關 [標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 和 [中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的更多資訊。同樣，也可以查閱 Postmark 文件以獲取有關其對 [標籤](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 支援的更多資訊。

如果您的應用程式正在使用 Amazon SES 寄送電子郵件，您應該使用 `metadata` 方法將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能由 Symfony Mailer 提供支援。Laravel 允許您註冊自訂回呼，這些回呼將在寄送訊息之前與 Symfony Message 實例一起呼叫。這讓您有機會在寄送訊息之前深度自訂訊息。為此，請在您的 `Envelope` 定義上定義一個 `using` 參數：

```php
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
```

<a name="markdown-mailables"></a>
## Markdown Mailable

Markdown mailable 訊息允許您利用 [郵件通知](/docs/{{version}}/notifications#mail-notifications) 的預建範本和元件。由於訊息是用 Markdown 撰寫的，Laravel 能夠為訊息渲染美觀、響應式的 HTML 範本，同時還會自動產生純文字對應項。

<a name="generating-markdown-mailables"></a>
### 產生 Markdown Mailable

要產生具有相應 Markdown 範本的 mailable，您可以使用 `make:mail` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

然後，在 mailable 的 `content` 方法中設定 mailable `Content` 定義時，使用 `markdown` 參數而不是 `view` 參數：

```php
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
```

<a name="writing-markdown-messages"></a>
### 撰寫 Markdown 訊息

Markdown mailable 使用 Blade 元件和 Markdown 語法的組合，讓您可以輕鬆建構郵件訊息，同時利用 Laravel 的預建電子郵件 UI 元件：

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
> 撰寫 Markdown 電子郵件時請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排內容渲染為程式碼區塊。

<a name="button-component"></a>
#### 按鈕元件

按鈕元件渲染一個置中的按鈕連結。該元件接受兩個參數，一個 `url` 和一個可選的 `color`。支援的顏色有 `primary`、`success` 和 `error`。您可以根據需要向訊息中新增任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```

<a name="panel-component"></a>
#### 面板元件

面板元件將給定的文字區塊渲染在一個面板中，該面板的背景顏色與訊息的其餘部分略有不同。這允許您將注意力吸引到給定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### 表格元件

表格元件允許您將 Markdown 表格轉換為 HTML 表格。該元件接受 Markdown 表格作為其內容。表格欄位對齊支援使用預設的 Markdown 表格對齊語法：

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

您可以將所有 Markdown 郵件元件匯出到自己的應用程式中進行自訂。要匯出元件，請使用 `vendor:publish` Artisan 命令發布 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件元件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含一個 `html` 和一個 `text` 目錄，每個目錄都包含每個可用元件的各自表示。您可以自由地自訂這些元件。

<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將自動轉換為 Markdown 郵件訊息的 HTML 表示中的內嵌 CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建構一個全新的主題，您可以將 CSS 檔案放置在 `html/themes` 目錄中。命名並儲存 CSS 檔案後，更新應用程式 `config/mail.php` 設定檔的 `theme` 選項以符合新主題的名稱。

要為單個 mailable 自訂主題，您可以將 mailable 類別的 `$theme` 屬性設定為寄送該 mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 寄送郵件

要寄送訊息，請使用 `Mail` [Facade](/docs/{{version}}/facades) 上的 `to` 方法。`to` 方法接受電子郵件位址、使用者實例或使用者集合。如果您傳遞物件或物件集合，郵件發送器將在確定電子郵件收件人時自動使用其 `email` 和 `name` 屬性，因此請確保這些屬性在您的物件上可用。指定收件人後，您可以將 mailable 類別的實例傳遞給 `send` 方法：

```php
<?php

namespace App\Http\Controllers;

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
```

您不僅限於在寄送訊息時指定「收件人」。您可以透過將其各自的方法鏈接在一起來自由設定「收件人」、「副本」和「密件副本」收件人：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```

<a name="looping-over-recipients"></a>
#### 循環處理收件人

有時，您可能需要透過迭代收件人/電子郵件位址陣列來將 mailable 寄送給收件人列表。然而，由於 `to` 方法將電子郵件位址附加到 mailable 的收件人列表，因此每次循環迭代都會向每個先前的收件人寄送另一封電子郵件。因此，您應該始終為每個收件人重新建立 mailable 實例：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```

<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過特定郵件發送器寄送郵件

預設情況下，Laravel 將使用應用程式 `mail` 設定檔中設定為 `default` 郵件發送器的郵件發送器寄送電子郵件。但是，您可以使用 `mailer` 方法使用特定的郵件發送器設定寄送訊息：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```

<a name="queueing-mail"></a>
### 郵件佇列

<a name="queueing-a-mail-message"></a>
#### 將郵件訊息加入佇列

由於寄送電子郵件訊息可能會對應用程式的響應時間產生負面影響，因此許多開發人員選擇將電子郵件訊息加入佇列以進行背景寄送。Laravel 使用其內建的 [統一佇列 API](/docs/{{version}}/queues) 輕鬆實現了這一點。要將郵件訊息加入佇列，請在指定訊息收件人後使用 `Mail` Facade 上的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法將自動負責將任務推送到佇列中，以便在背景寄送訊息。您需要在使用此功能之前 [設定您的佇列](/docs/{{version}}/queues)。

<a name="delayed-message-queueing"></a>
#### 延遲訊息佇列

如果您希望延遲佇列電子郵件訊息的傳遞，您可以使用 `later` 方法。作為其第一個參數，`later` 方法接受一個 `DateTime` 實例，指示應何時寄送訊息：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->addMinutes(10), new OrderShipped($order));
```

<a name="pushing-to-specific-queues"></a>
#### 推送到特定佇列

由於使用 `make:mail` 命令產生的所有 mailable 類別都使用了 `Illuminate\Bus\Queueable` Trait，因此您可以在任何 mailable 類別實例上呼叫 `onQueue` 和 `onConnection` 方法，允許您為訊息指定連線和佇列名稱：

```php
$message = (new OrderShipped($order))
    ->onConnection('sqs')
    ->onQueue('emails');

Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue($message);
```

<a name="queueing-by-default"></a>
#### 預設佇列

如果您有希望始終加入佇列的 mailable 類別，您可以在類別上實作 `ShouldQueue` 契約。現在，即使您在郵寄時呼叫 `send` 方法，mailable 仍將加入佇列，因為它實作了該契約：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```

<a name="queued-mailables-and-database-transactions"></a>
#### 佇列 Mailable 與資料庫交易

當佇列 mailable 在資料庫交易中分派時，它們可能會在資料庫交易提交之前由佇列處理。發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的 mailable 依賴於這些模型，則在處理寄送佇列 mailable 的任務時可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 設定選項設定為 `false`，您仍然可以透過在寄送郵件訊息時呼叫 `afterCommit` 方法來指示特定佇列 mailable 應在所有開啟的資料庫交易提交後分派：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，您可以從 mailable 的建構函式中呼叫 `afterCommit` 方法：

```php
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
```

> [!NOTE]
> 要了解有關解決這些問題的更多資訊，請查閱有關 [佇列任務和資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions) 的文件。

<a name="queued-email-failures"></a>
#### 佇列電子郵件失敗

當佇列電子郵件失敗時，如果已定義，將呼叫佇列 mailable 類別上的 `failed` 方法。導致佇列電子郵件失敗的 `Throwable` 實例將傳遞給 `failed` 方法：

```php
<?php

namespace App\Mail;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;
use Throwable;

class OrderDelayed extends Mailable implements ShouldQueue
{
    use SerializesModels;

    /**
     * Handle a queued email's failure.
     */
    public function failed(Throwable $exception): void
    {
        // ...
    }
}
```

<a name="rendering-mailables"></a>
## 渲染 Mailable

有時您可能希望在不寄送的情況下擷取 mailable 的 HTML 內容。為此，您可以呼叫 mailable 的 `render` 方法。此方法將以字串形式返回 mailable 的評估 HTML 內容：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```

<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailable

在設計 mailable 的範本時，在瀏覽器中像典型的 Blade 範本一樣快速預覽渲染的 mailable 會很方便。因此，Laravel 允許您直接從路由閉包或控制器返回任何 mailable。當返回 mailable 時，它將在瀏覽器中渲染並顯示，讓您可以快速預覽其設計而無需將其寄送給實際的電子郵件位址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

<a name="localizing-mailables"></a>
## Mailable 本地化

Laravel 允許您以請求當前語系以外的語系寄送 mailable，即使郵件已加入佇列，它也會記住此語系。

為此，`Mail` Facade 提供了 `locale` 方法來設定所需的語言。當 mailable 的範本正在評估時，應用程式將切換到此語系，然後在評估完成後恢復到先前的語系：

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時，應用程式會儲存每個使用者偏好的語系。透過在一個或多個模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在寄送郵件時使用此儲存的語系：

```php
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
```

一旦您實作了介面，Laravel 將在向模型寄送 mailable 和通知時自動使用偏好的語系。因此，在使用此介面時無需呼叫 `locale` 方法：

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試

<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了多種檢查 mailable 結構的方法。此外，Laravel 還提供了多種方便的方法來測試您的 mailable 是否包含您期望的內容：

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
    $mailable->assertDontSeeInHtml('Invoice Not Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertDontSeeInText('Invoice Not Paid');
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
    $mailable->assertDontSeeInHtml('Invoice Not Paid');
    $mailable->assertSeeInOrderInHtml(['Invoice Paid', 'Thanks']);

    $mailable->assertSeeInText($user->email);
    $mailable->assertDontSeeInText('Invoice Not Paid');
    $mailable->assertSeeInOrderInText(['Invoice Paid', 'Thanks']);

    $mailable->assertHasAttachment('/path/to/file');
    $mailable->assertHasAttachment(Attachment::fromPath('/path/to/file'));
    $mailable->assertHasAttachedData($pdfData, 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorage('/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
    $mailable->assertHasAttachmentFromStorageDisk('s3', '/path/to/file', 'name.pdf', ['mime' => 'application/pdf']);
}
```

正如您所預期的，「HTML」斷言斷言您的 mailable 的 HTML 版本包含給定字串，而「text」斷言斷言您的 mailable 的純文字版本包含給定字串。

<a name="testing-mailable-sending"></a>
### 測試 Mailable 寄送

我們建議將 mailable 內容的測試與斷言給定 mailable 已「寄送」給特定使用者的測試分開。通常，mailable 的內容與您正在測試的程式碼無關，並且僅斷言 Laravel 已被指示寄送給定 mailable 就足夠了。

您可以使用 `Mail` Facade 的 `fake` 方法來阻止郵件寄送。呼叫 `Mail` Facade 的 `fake` 方法後，您可以斷言 mailable 已被指示寄送給使用者，甚至檢查 mailable 接收到的資料：

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

    // Assert a mailable was sent twice...
    Mail::assertSentTimes(OrderShipped::class, 2);

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

        // Assert a mailable was sent twice...
        Mail::assertSentTimes(OrderShipped::class, 2);

        // Assert 3 total mailables were sent...
        Mail::assertSentCount(3);
    }
}
```

如果您正在將 mailable 加入佇列以進行背景傳遞，則應使用 `assertQueued` 方法而不是 `assertSent`：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

您可以將閉包傳遞給 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以斷言已寄送通過給定「真實性測試」的 mailable。如果至少有一個 mailable 通過給定的真實性測試，則斷言將成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

呼叫 `Mail` Facade 的斷言方法時，所提供閉包接受的 mailable 實例會公開有助於檢查 mailable 的方法：

```php
Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($user) {
    return $mail->hasTo($user->email) &&
           $mail->hasCc('...') &&
           $mail->hasBcc('...') &&
           $mail->hasReplyTo('...') &&
           $mail->hasFrom('...') &&
           $mail->hasSubject('...') &&
           $mail->usesMailer('ses');
});
```

mailable 實例還包含多個有助於檢查 mailable 附件的方法：

```php
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
```

您可能已經注意到有兩種方法可以斷言郵件未寄送：`assertNotSent` 和 `assertNotQueued`。有時您可能希望斷言沒有郵件被寄送**或**加入佇列。為此，您可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本地開發

在開發寄送電子郵件的應用程式時，您可能不希望實際將電子郵件寄送給真實的電子郵件位址。Laravel 提供了多種方法來「禁用」在本地開發期間實際寄送電子郵件。

<a name="log-driver"></a>
#### Log 驅動程式

`log` 郵件驅動程式不會寄送您的電子郵件，而是將所有電子郵件訊息寫入您的日誌檔案以供檢查。通常，此驅動程式僅在本地開發期間使用。有關按環境設定應用程式的更多資訊，請查閱 [設定文件](/docs/{{version}}/configuration#environment-configuration)。

<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，您可以使用 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 等服務和 `smtp` 驅動程式將您的電子郵件訊息寄送給「虛擬」郵箱，您可以在真實的電子郵件客戶端中查看它們。這種方法的好處是允許您在 Mailtrap 的訊息檢視器中實際檢查最終的電子郵件。

如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，您可以使用 [Mailpit](https://github.com/axllent/mailpit) 預覽您的訊息。當 Sail 運行時，您可以透過以下位址存取 Mailpit 介面：`http://localhost:8025`。

<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 位址

最後，您可以透過呼叫 `Mail` Facade 提供的 `alwaysTo` 方法來指定一個全域「收件人」位址。通常，此方法應從應用程式服務提供者之一的 `boot` 方法中呼叫：

```php
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
```

<a name="events"></a>
## 事件

Laravel 在寄送郵件訊息時會分派兩個事件。`MessageSending` 事件在訊息寄送之前分派，而 `MessageSent` 事件在訊息寄送之後分派。請記住，這些事件是在郵件正在**寄送**時分派的，而不是在郵件加入佇列時分派的。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

```php
use Illuminate\Mail\Events\MessageSending;
// use Illuminate\Mail\Events\MessageSent;

class LogMessage
{
    /**
     * Handle the event.
     */
    public function handle(MessageSending $event): void
    {
        // ...
    }
}
```

<a name="custom-transports"></a>
## 自訂傳輸器

Laravel 包含了多種郵件傳輸器；然而，您可能希望編寫自己的傳輸器，透過 Laravel 不支援的其他服務傳遞電子郵件。首先，定義一個擴展 `Symfony\Component\Mailer\Transport\AbstractTransport` 類別的類別。然後，在您的傳輸器上實作 `doSend` 和 `__toString` 方法：

```php
<?php

namespace App\Mail;

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
```

定義自訂傳輸器後，您可以透過 `Mail` Facade 提供的 `extend` 方法註冊它。通常，這應該在應用程式 `AppServiceProvider` 的 `boot` 方法中完成。一個 `$config` 參數將傳遞給提供給 `extend` 方法的閉包。此參數將包含在應用程式 `config/mail.php` 設定檔中為郵件發送器定義的設定陣列：

```php
use App\Mail\MailchimpTransport;
use Illuminate\Support\Facades\Mail;
use MailchimpTransactional\ApiClient;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Mail::extend('mailchimp', function (array $config = []) {
        $client = new ApiClient;

        $client->setApiKey($config['key']);

        return new MailchimpTransport($client);
    });
}
```

一旦您的自訂傳輸器已定義並註冊，您可以在應用程式 `config/mail.php` 設定檔中建立一個使用新傳輸器的郵件發送器定義：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```

<a name="additional-symfony-transports"></a>
### 額外的 Symfony 傳輸器

Laravel 包含了對一些現有 Symfony 維護的郵件傳輸器（如 Mailgun 和 Postmark）的支援。然而，您可能希望透過支援額外的 Symfony 維護的傳輸器來擴展 Laravel。您可以透過 Composer 引入必要的 Symfony 郵件發送器並向 Laravel 註冊傳輸器來實現。例如，您可以安裝並註冊「Brevo」（以前稱為「Sendinblue」）Symfony 郵件發送器：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

安裝 Brevo 郵件發送器套件後，您可以將 Brevo API 憑證的條目新增到應用程式的 `services` 設定檔中：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接下來，您可以使用 `Mail` Facade 的 `extend` 方法向 Laravel 註冊傳輸器。通常，這應該在服務提供者的 `boot` 方法中完成：

```php
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
```

一旦您的傳輸器已註冊，您可以在應用程式 `config/mail.php` 設定檔中建立一個使用新傳輸器的郵件發送器定義：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```
