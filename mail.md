# 郵件

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
    - [故障轉移設定](#failover-configuration)
    - [循環式設定](#round-robin-configuration)
- [產生 Mailables](#generating-mailables)
- [撰寫 Mailables](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [行內附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與中繼資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailables](#markdown-mailables)
    - [產生 Markdown Mailables](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
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

傳送電子郵件不必複雜。Laravel 提供了一個簡潔、簡單的電子郵件 API，由熱門的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件提供支援。Laravel 和 Symfony Mailer 提供了透過 SMTP、Mailgun、Postmark、Resend、Amazon SES 以及 `sendmail` 傳送電子郵件的驅動程式，讓您可以快速開始使用本地或雲端服務傳送郵件。


<a name="configuration"></a>
### 設定

Laravel 的電子郵件服務可以透過應用程式的 `config/mail.php` 設定檔進行設定。此檔案中設定的每個 mailer 都可以有自己獨特的設定，甚至有自己獨特的「傳輸器 (transport)」，讓您的應用程式可以使用不同的電子郵件服務來傳送特定的電子郵件訊息。例如，您的應用程式可能使用 Postmark 傳送交易性電子郵件，同時使用 Amazon SES 傳送大量電子郵件。

在您的 `mail` 設定檔中，您會找到一個 `mailers` 設定陣列。此陣列包含 Laravel 支援的每個主要郵件驅動程式 (drivers) / 傳輸器 (transports) 的範例設定項目，而 `default` 設定值則決定了當您的應用程式需要傳送電子郵件訊息時，預設將使用哪個 mailer。


<a name="driver-prerequisites"></a>
### 驅動程式先決條件

基於 API 的驅動程式，例如 Mailgun、Postmark、Resend 和 MailerSend，通常比透過 SMTP 伺服器傳送郵件更簡單、更快速。在可能的情況下，我們建議您使用這些驅動程式之一。


<a name="mailgun-driver"></a>
#### Mailgun 驅動程式

若要使用 Mailgun 驅動程式，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳輸器：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接著，您需要對應用程式的 `config/mail.php` 設定檔進行兩項變更。首先，將您的預設 mailer 設定為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

其次，將以下設定陣列新增至您的 `mailers` 陣列：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

設定應用程式的預設 mailer 後，將以下選項新增至您的 `config/services.php` 設定檔：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

如果您沒有使用美國 [Mailgun 區域](https://documentation.mailgun.com/docs/mailgun/api-reference/#mailgun-regions)，您可以在 `services` 設定檔中定義您所在區域的端點：

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

若要使用 [Postmark](https://postmarkapp.com/) 驅動程式，請透過 Composer 安裝 Symfony 的 Postmark Mailer 傳輸器：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接著，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `postmark`。設定應用程式的預設 mailer 後，請確認您的 `config/services.php` 設定檔包含以下選項：

```php
'postmark' => [
    'token' => env('POSTMARK_TOKEN'),
],
```

如果您想指定特定 mailer 應使用的 Postmark 訊息串流，您可以將 `message_stream_id` 設定選項新增到該 mailer 的設定陣列中。此設定陣列位於您的應用程式的 `config/mail.php` 設定檔中：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

透過這種方式，您還可以設定多個具有不同訊息串流的 Postmark mailer。


<a name="resend-driver"></a>
#### Resend 驅動程式

若要使用 [Resend](https://resend.com/) 驅動程式，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接著，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `resend`。設定應用程式的預設 mailer 後，請確認您的 `config/services.php` 設定檔包含以下選項：

```php
'resend' => [
    'key' => env('RESEND_KEY'),
],
```


<a name="ses-driver"></a>
#### SES 驅動程式

若要使用 Amazon SES 驅動程式，您必須先安裝 Amazon AWS SDK for PHP。您可以透過 Composer 套件管理器安裝此函式庫：

```shell
composer require aws/aws-sdk-php
```

接著，將 `config/mail.php` 設定檔中的 `default` 選項設定為 `ses`，並驗證您的 `config/services.php` 設定檔包含以下選項：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

若要透過會話權杖 (session token) 利用 AWS [臨時憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，您可以將 `token` 鍵新增到應用程式的 SES 設定中：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

若要與 SES 的[訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html)互動，您可以在郵件訊息的 [headers](#headers) 方法所回傳的陣列中，回傳 `X-Ses-List-Management-Options` 標頭：

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

如果您想要定義 Laravel 在傳送電子郵件時應傳遞給 AWS SDK 的 `SendEmail` 方法的[額外選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，您可以在 `ses` 設定中定義一個 `options` 陣列：

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

[MailerSend](https://www.mailersend.com/) 是一個交易性電子郵件和簡訊服務，他們為 Laravel 維護著自己的基於 API 的郵件驅動程式。包含該驅動程式的套件可以透過 Composer 套件管理器安裝：

```shell
composer require mailersend/laravel-driver
```

安裝該套件後，將 `MAILERSEND_API_KEY` 環境變數新增到應用程式的 `.env` 檔案中。此外，`MAIL_MAILER` 環境變數應定義為 `mailersend`：

```ini
MAIL_MAILER=mailersend
MAIL_FROM_ADDRESS=app@yourdomain.com
MAIL_FROM_NAME="App Name"

MAILERSEND_API_KEY=your-api-key
```

最後，將 MailerSend 新增到應用程式 `config/mail.php` 設定檔中的 `mailers` 陣列中：

```php
'mailersend' => [
    'transport' => 'mailersend',
],
```

要了解更多關於 MailerSend 的資訊，包括如何使用託管模板，請參閱 [MailerSend 驅動程式文件](https://github.com/mailersend/mailersend-laravel-driver#usage)。

<a name="failover-configuration"></a>
### 故障轉移設定

有時，您設定用於傳送應用程式郵件的外部服務可能會發生故障。在這些情況下，定義一個或多個備用郵件傳送設定會很有用，以防主要傳送驅動程式發生故障時使用。

為此，您應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳輸器的郵件驅動程式。應用程式 `failover` 郵件驅動程式的設定陣列應包含一個 `mailers` 陣列，該陣列引用了應選擇用於傳送的郵件驅動程式順序：

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

定義 failover 郵件驅動程式後，您應該將其設定為應用程式使用的預設郵件驅動程式，方法是在應用程式的 `mail` 設定檔中將其名稱指定為 `default` 設定鍵的值：

```php
'default' => env('MAIL_MAILER', 'failover'),
```

<a name="round-robin-configuration"></a>
### 循環式設定

`roundrobin` 傳輸器允許您將郵件工作負載分配給多個郵件驅動程式。首先，在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` 傳輸器的郵件驅動程式。應用程式 `roundrobin` 郵件驅動程式的設定陣列應包含一個 `mailers` 陣列，該陣列引用了應選擇用於傳送的郵件驅動程式：

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

定義 roundrobin 郵件驅動程式後，您應該將其設定為應用程式使用的預設郵件驅動程式，方法是在應用程式的 `mail` 設定檔中將其名稱指定為 `default` 設定鍵的值：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

roundrobin 傳輸器從已設定的郵件驅動程式列表中選擇一個隨機郵件驅動程式，然後在後續的每封郵件中切換到下一個可用的郵件驅動程式。與有助於實現[高可用性](https://en.wikipedia.org/wiki/High_availability)的 `failover` 傳輸器不同，`roundrobin` 傳輸器提供[負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))。

<a name="generating-mailables"></a>
## 產生 Mailables

在建構 Laravel 應用程式時，您的應用程式傳送的每種郵件類型都以「mailable」類別表示。這些類別儲存在 `app/Mail` 目錄中。如果您的應用程式中沒有看到此目錄，請不用擔心，因為當您使用 `make:mail` Artisan 命令建立第一個 mailable 類別時，它將會自動產生：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailables

當您產生 mailable class 後，請開啟它以便我們探索其內容。Mailable class 的設定是透過多個方法完成的，包括 `envelope`、`content` 和 `attachments` 方法。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，此物件定義了郵件的主旨，有時也定義收件者。`content` 方法則會回傳一個 `Illuminate\Mail\Mailables\Content` 物件，此物件定義了用於產生郵件內容的 [Blade 樣板](/docs/{{version}}/blade)。


<a name="configuring-the-sender"></a>
### 設定寄件者


<a name="using-the-envelope"></a>
#### 使用信封

首先，讓我們來探索如何設定電子郵件的寄件者。換句話說，也就是誰會是這封郵件的「寄件者 (from)」。設定寄件者有兩種方式。首先，您可以在郵件的 envelope 上指定「寄件者 (from)」位址：

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

如果您願意，您也可以指定 `replyTo` 位址：

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

然而，如果您的應用程式所有電子郵件都使用相同的「寄件者 (from)」位址，那麼在每個您產生的 mailable class 中都添加它可能會變得繁瑣。您可以改為在 `config/mail.php` 設定檔中指定一個全域的「寄件者 (from)」位址。如果 mailable class 中沒有指定其他「寄件者 (from)」位址，則會使用此位址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，您也可以在 `config/mail.php` 設定檔中定義一個全域的「回覆 (reply_to)」位址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```


<a name="configuring-the-view"></a>
### 設定視圖

在 mailable class 的 `content` 方法中，您可以定義 `view`，也就是用於渲染電子郵件內容的樣板。由於每封電子郵件通常會使用 [Blade 樣板](/docs/{{version}}/blade) 來渲染其內容，因此在建構電子郵件的 HTML 時，您將擁有 Blade 樣板引擎的完整功能和便利性：

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
> 您可能會希望建立一個 `resources/views/mail` 目錄來存放所有電子郵件樣板；不過，您也可以自由地將它們放置在 `resources/views` 目錄中的任何位置。


<a name="plain-text-emails"></a>
#### 純文字郵件

如果您想定義電子郵件的純文字版本，可以在建立郵件的 `Content` 定義時指定純文字樣板。如同 `view` 參數一樣，`text` 參數應該是一個樣板名稱，用於渲染電子郵件的內容。您可以自由地同時定義郵件的 HTML 和純文字版本：

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

為了清晰起見，`html` 參數可以作為 `view` 參數的別名來使用：

```php
return new Content(
    html: 'mail.orders.shipped',
    text: 'mail.orders.shipped-text'
);
```


<a name="view-data"></a>
### 視圖資料


<a name="via-public-properties"></a>
#### 透過公開屬性

通常，您會希望將一些資料傳遞給視圖，以便在渲染電子郵件的 HTML 時加以利用。您可以透過兩種方式讓資料在視圖中可用。首先，您 mailable class 上定義的任何 public 屬性都會自動在視圖中可用。因此，舉例來說，您可以將資料傳遞到 mailable class 的建構函式中，並將這些資料設定為 class 上定義的 public 屬性：

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

一旦資料被設定為 public 屬性，它就會自動在您的視圖中可用，因此您可以像存取 Blade 樣板中的任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $order->price }}
</div>
```


<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果您想在電子郵件資料傳送至樣板之前自訂其格式，可以透過 `Content` 定義的 `with` 參數手動將資料傳遞給視圖。通常，您仍然會透過 mailable class 的建構函式傳遞資料；但是，您應該將這些資料設定為 `protected` 或 `private` 屬性，這樣資料就不會自動在樣板中可用：

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

一旦資料透過 `with` 參數傳遞，它就會自動在您的視圖中可用，因此您可以像存取 Blade 樣板中的任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

要為電子郵件新增附件，您需要將附件新增到訊息的 `attachments` 方法所返回的陣列中。首先，您可以透過為 `Attachment` 類別提供的 `fromPath` 方法提供檔案路徑來新增附件：

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

為訊息附加檔案時，您也可以使用 `as` 和 `withMime` 方法指定附件的顯示名稱和/或 MIME 類型：

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

如果您已將檔案儲存在您的[檔案系統磁碟](/docs/{{version}}/filesystem)之一上，您可以使用 `fromStorage` 附件方法將其附加到電子郵件中：

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

如果您需要指定預設磁碟以外的儲存磁碟，則可以使用 `fromStorageDisk` 方法：

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

可以使用 `fromData` 附件方法將原始位元組字串作為附件附加。例如，如果您在記憶體中生成了 PDF，並希望將其附加到電子郵件而不安裝到磁碟，您可以使用此方法。 `fromData` 方法接受一個閉包，該閉包解析原始資料位元組以及應分配給附件的名稱：

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
### 行內附件

在電子郵件中嵌入行內圖片通常很繁瑣；然而，Laravel 提供了一種便捷的方式來將圖片附加到您的電子郵件中。要嵌入行內圖片，請在您的電子郵件範本中使用 `$message` 變數上的 `embed` 方法。Laravel 會自動將 `$message` 變數提供給您所有的電子郵件範本，因此您無需擔心手動傳遞它：

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]
> `$message` 變數在純文字訊息範本中不可用，因為純文字訊息不使用行內附件。


<a name="embedding-raw-data-attachments"></a>
#### 嵌入原始資料附件

如果您已經有一個原始圖片資料字串，希望嵌入到電子郵件範本中，您可以呼叫 `$message` 變數上的 `embedData` 方法。呼叫 `embedData` 方法時，您需要提供一個應分配給嵌入圖片的檔案名稱：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```


<a name="attachable-objects"></a>
### 可附加物件

雖然透過簡單的字串路徑將檔案附加到訊息通常已足夠，但在許多情況下，應用程式中可附加的實體是由類別表示的。例如，如果您的應用程式要將照片附加到訊息中，您的應用程式可能也有一個代表該照片的 `Photo` Model。在這種情況下，直接將 `Photo` Model 傳遞給 `attach` 方法會不會很方便？可附加物件正是為此而生。

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

定義好可附加物件後，在建構電子郵件訊息時，您可以從 `attachments` 方法返回該物件的一個實例：

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

當然，附件資料可以儲存在遠端檔案儲存服務上，例如 Amazon S3。因此，Laravel 也允許您從儲存在應用程式的[檔案系統磁碟](/docs/{{version}}/filesystem)上的資料生成附件實例：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，您還可以透過記憶體中的資料建立附件實例。為此，請向 `fromData` 方法提供一個閉包。該閉包應返回代表附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 也提供了其他方法，您可以用來自訂附件。例如，您可以使用 `as` 和 `withMime` 方法來自訂檔案的名稱和 MIME 類型：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```


<a name="headers"></a>
### 標頭

有時您可能需要為外寄訊息附加額外的標頭。例如，您可能需要設定自訂的 `Message-Id` 或其他任意文字標頭。

為此，請在您的 mailable 上定義一個 `headers` 方法。 `headers` 方法應返回一個 `Illuminate\Mail\Mailables\Headers` 實例。此類別接受 `messageId`、`references` 和 `text` 參數。當然，您可以只提供您的特定訊息所需的參數：

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

某些第三方電子郵件服務商，如 Mailgun 和 Postmark，支援訊息「標籤 (tags)」和「中繼資料 (metadata)」，可用於對應用程式傳送的電子郵件進行分組和追蹤。您可以透過 `Envelope` 定義，將標籤和中繼資料新增至電子郵件訊息中：

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

如果您的應用程式使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件，以了解更多關於[標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags)和[中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages)的資訊。同樣地，也可以查閱 Postmark 文件，以了解更多關於其對[標籤](https://postmarkapp.com/blog/tags-support-for-smtp)和[中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq)的支援。

如果您的應用程式使用 Amazon SES 傳送電子郵件，您應該使用 `metadata` 方法將 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)附加到訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能由 Symfony Mailer 提供支援。Laravel 允許您註冊自訂回呼 (callbacks)，這些回呼將在傳送訊息之前，以 Symfony Message 實例的形式被呼叫。這讓您有機會在訊息傳送之前對其進行深度自訂。為此，請在您的 `Envelope` 定義中定義一個 `using` 參數：

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
## Markdown Mailables

Markdown mailable 訊息讓您可以在 mailable 中利用 [郵件通知](/docs/{{version}}/notifications#mail-notifications) 的預建模板和元件。由於這些訊息是以 Markdown 編寫，Laravel 能夠為訊息渲染出美觀、響應式的 HTML 模板，同時也會自動產生純文字版本。


<a name="generating-markdown-mailables"></a>
### 產生 Markdown Mailables

若要產生帶有對應 Markdown 模板的 mailable，您可以使用 `make:mail` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

接著，在 mailable 的 `content` 方法中設定 `Content` 定義時，請使用 `markdown` 參數而非 `view` 參數：

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

Markdown mailables 使用 Blade 元件與 Markdown 語法的組合，讓您能輕鬆建構郵件訊息，同時利用 Laravel 預建的電子郵件 UI 元件：

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
> 撰寫 Markdown 電子郵件時，請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染出置中的按鈕連結。此元件接受兩個引數：一個 `url` 和一個可選的 `color`。支援的顏色有 `primary`、`success` 和 `error`。您可以在訊息中加入任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板元件

面板元件會將給定的文字區塊渲染在一個面板中，該面板的背景顏色與訊息其餘部分略有不同。這可讓您將注意力引導至特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格元件

表格元件允許您將 Markdown 表格轉換為 HTML 表格。此元件將 Markdown 表格作為其內容。表格欄位對齊方式支援使用預設的 Markdown 表格對齊語法：

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

您可以將所有 Markdown 郵件元件匯出到您自己的應用程式中進行自訂。若要匯出這些元件，請使用 `vendor:publish` Artisan 命令來發佈 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件元件發佈到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含 `html` 和 `text` 目錄，每個目錄都包含所有可用元件各自的表示形式。您可以隨意自訂這些元件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將會自動轉換為 Markdown 郵件訊息的 HTML 表示形式中的行內 CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建立一個全新的主題，您可以將 CSS 檔案放在 `html/themes` 目錄中。在命名並儲存您的 CSS 檔案後，請更新應用程式 `config/mail.php` 設定檔中的 `theme` 選項，以符合您新主題的名稱。

若要為個別的 mailable 自訂主題，您可以將 mailable 類別的 `$theme` 屬性設定為傳送該 mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 傳送郵件

若要傳送訊息，請使用 `Mail` [Facade](/docs/{{version}}/facades) 的 `to` 方法。`to` 方法接受電子郵件地址、使用者實例或使用者集合。若您傳遞物件或物件集合，郵件發送器將在判斷電子郵件收件人時，自動使用其 `email` 和 `name` 屬性，因此請確保這些屬性在您的物件上可用。指定收件人後，您可以將您的 Mailable 類別實例傳遞給 `send` 方法：

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

傳送訊息時，您不僅限於指定「To」收件人。您可以透過鏈接各自的方法來設定「To」、「Cc」和「Bcc」收件人：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```

<a name="looping-over-recipients"></a>
#### 循環處理收件人

有時，您可能需要透過迭代收件人/電子郵件地址陣列，將 Mailable 傳送給收件人列表。然而，由於 `to` 方法會將電子郵件地址附加到 Mailable 的收件人列表，循環中的每次迭代都會向每個先前的收件人再次傳送一封電子郵件。因此，您應始終為每個收件人重新建立 Mailable 實例：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```

<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過特定郵件發送器傳送郵件

預設情況下，Laravel 將使用您應用程式 `mail` 設定檔中設定為 `default` 的郵件發送器傳送電子郵件。然而，您可以使用 `mailer` 方法來透過特定的郵件發送器設定傳送訊息：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```

<a name="queueing-mail"></a>
### 佇列郵件

<a name="queueing-a-mail-message"></a>
#### 將郵件訊息加入佇列

由於傳送電子郵件訊息可能會對應用程式的響應時間產生負面影響，許多開發者選擇將電子郵件訊息加入佇列，以便在背景傳送。Laravel 透過其內建的 [統一佇列 API](/docs/{{version}}/queues) 簡化了這項工作。若要將郵件訊息加入佇列，請在指定訊息的收件人之後，使用 `Mail` Facade 的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法將自動負責將一個 Job 推送到佇列中，以便在背景傳送訊息。您需要在使用此功能之前[設定您的佇列](/docs/{{version}}/queues)。

<a name="delayed-message-queueing"></a>
#### 延遲訊息佇列

若您希望延遲佇列電子郵件訊息的傳送，您可以使用 `later` 方法。作為其第一個參數，`later` 方法接受一個 `DateTime` 實例，指示訊息應該何時傳送：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->addMinutes(10), new OrderShipped($order));
```

<a name="pushing-to-specific-queues"></a>
#### 推送到特定佇列

由於所有使用 `make:mail` 命令產生的 Mailable 類別都使用了 `Illuminate\Bus\Queueable` Trait，您可以在任何 Mailable 類別實例上呼叫 `onQueue` 和 `onConnection` 方法，讓您能夠為訊息指定連線和佇列名稱：

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

若您有希望始終加入佇列的 Mailable 類別，您可以在該類別上實作 `ShouldQueue` 契約。現在，即使您在寄送郵件時呼叫 `send` 方法，該 Mailable 仍會被加入佇列，因為它實作了該契約：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```

<a name="queued-mailables-and-database-transactions"></a>
#### 佇列式 Mailables 與資料庫交易

當佇列式 Mailables 在資料庫交易中被分派時，它們可能會在資料庫交易提交之前被佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。若您的 Mailable 依賴這些模型，當處理傳送佇列式 Mailable 的 Job 時，可能會發生意外錯誤。

若您的佇列連線的 `after_commit` 設定選項設為 `false`，您仍然可以在傳送郵件訊息時呼叫 `afterCommit` 方法，以指示特定的佇列式 Mailable 應在所有開放的資料庫交易提交後才分派：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，您可以從您的 Mailable 的建構函式中呼叫 `afterCommit` 方法：

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
> 要了解如何解決這些問題的更多資訊，請查閱關於[佇列式 Job 與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。

<a name="queued-email-failures"></a>
#### 佇列電子郵件失敗

當佇列電子郵件失敗時，佇列 Mailable 類別上的 `failed` 方法將會被呼叫 (若已定義)。導致佇列電子郵件失敗的 `Throwable` 實例將會被傳遞給 `failed` 方法：

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
## 渲染 Mailables

有時您可能希望擷取 Mailable 的 HTML 內容，而不實際傳送它。若要實現此目的，您可以呼叫 Mailable 的 `render` 方法。此方法將會以字串形式回傳 Mailable 評估後的 HTML 內容：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```

<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailables

在設計 Mailable 範本時，在瀏覽器中快速預覽渲染後的 Mailable 是很方便的，就像一般的 Blade 範本一樣。因此，Laravel 允許您直接從路由閉包或控制器回傳任何 Mailable。當 Mailable 被回傳時，它將會在瀏覽器中渲染並顯示，讓您能夠快速預覽其設計，而無需將其傳送至實際的電子郵件地址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

<a name="localizing-mailables"></a>
## 本地化 Mailables

Laravel 允許您以不同於請求目前語系 (locale) 的語系 (locale) 傳送 mailable，即使該郵件被佇列，它也會記住這個語系。

為達成此目的，`Mail` Facade 提供了 `locale` 方法來設定所需的語言。當 mailable 的模板被評估時，應用程式會切換到此語系，並在評估完成後回復到之前的語系：

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時候，應用程式會儲存每個使用者的偏好語系。透過在您的一個或多個模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在傳送郵件時使用這個已儲存的語系：

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

一旦您實作了這個介面，Laravel 將會在向模型傳送 mailables 和通知時自動使用偏好語系。因此，當使用此介面時，無需呼叫 `locale` 方法：

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試

<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了多種方法來檢查 mailable 的結構。此外，Laravel 也提供了幾種方便的方法來測試 mailable 是否包含您預期的內容：

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

如您所預期，「HTML」斷言會確認您 mailable 的 HTML 版本包含特定字串，而「text」斷言則會確認您 mailable 的純文字版本包含特定字串。

<a name="testing-mailable-sending"></a>
### 測試 Mailable 傳送

我們建議將 Mailable 的內容測試與斷言特定 Mailable 已「傳送」給特定使用者的測試分開進行。通常，Mailable 的內容與您正在測試的程式碼無關，因此只需斷言 Laravel 已被指示傳送特定的 Mailable 即可。

您可以使用 `Mail` Facade 的 `fake` 方法來阻止郵件被實際傳送。呼叫 `Mail` Facade 的 `fake` 方法後，您可以斷言 Mailable 已被指示傳送給使用者，甚至檢查 Mailable 收到的資料：

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

```php
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

如果您正在將 Mailable 加入佇列以在背景中傳送，您應該使用 `assertQueued` 方法而不是 `assertSent`：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

您可以將閉包傳遞給 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以斷言傳送的 Mailable 通過特定的「真實性測試」。如果至少有一個 Mailable 通過給定的真實性測試，則斷言將成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

當呼叫 `Mail` Facade 的斷言方法時，所提供的閉包接受的 Mailable 實例會公開有助於檢查 Mailable 的方法：

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

Mailable 實例還包含幾個有助於檢查 Mailable 附件的方法：

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

您可能已經注意到有兩種方法可以斷言郵件未傳送：`assertNotSent` 和 `assertNotQueued`。有時您可能希望斷言沒有郵件被傳送**或**加入佇列。為此，您可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本地開發

當開發傳送電子郵件的應用程式時，您可能不希望真的將電子郵件傳送到實際的電子郵件地址。Laravel 提供了幾種方法來「停用」在本地開發期間實際傳送電子郵件的功能。


<a name="log-driver"></a>
#### Log 驅動程式

`log` 郵件驅動程式不會傳送您的電子郵件，而是將所有電子郵件訊息寫入您的記錄檔中以供檢查。通常，此驅動程式僅在本地開發期間使用。有關如何根據環境設定應用程式的更多資訊，請查閱[設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，您可以使用 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 等服務以及 `smtp` 驅動程式，將您的電子郵件訊息傳送到「虛擬」信箱中，您可以在真正的電子郵件客戶端中查看它們。這種方法的好處是讓您可以在 Mailtrap 的訊息檢視器中實際檢查最終的電子郵件。

如果您使用 [Laravel Sail](/docs/{{version}}/sail)，您可以使用 [Mailpit](https://github.com/axllent/mailpit) 預覽您的訊息。當 Sail 執行時，您可以透過 `http://localhost:8025` 存取 Mailpit 介面。


<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 地址

最後，您可以透過呼叫 `Mail` Facade 提供的 `alwaysTo` 方法來指定一個全域的「to」地址。通常，這個方法應該在您應用程式的其中一個服務提供者的 `boot` 方法中被呼叫：

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

Laravel 在傳送郵件訊息時會發送兩個事件。`MessageSending` 事件在訊息傳送之前發送，而 `MessageSent` 事件在訊息傳送之後發送。請記住，這些事件是在郵件被 *傳送* 時發送，而不是在它被佇列時。您可以在應用程式中為這些事件建立[事件監聽器](/docs/{{version}}/events)：

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

Laravel 包含了多種郵件傳輸器；然而，您可能希望撰寫自己的傳輸器，以便透過 Laravel 未開箱即用支援的其他服務傳遞電子郵件。首先，定義一個繼承 `Symfony\Component\Mailer\Transport\AbstractTransport` 類別的類別。然後，在您的傳輸器上實作 `doSend` 和 `__toString` 方法：

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

定義好自訂傳輸器後，您可以透過 `Mail` Facade 提供的 `extend` 方法來註冊它。通常，這應該在您應用程式的 `AppServiceProvider` 的 `boot` 方法中完成。一個 `$config` 引數將會傳遞給 `extend` 方法提供的閉包。這個引數會包含為郵件程式在應用程式的 `config/mail.php` 設定檔中定義的設定陣列：

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

定義並註冊好自訂傳輸器後，您可以在應用程式的 `config/mail.php` 設定檔中建立一個使用這個新傳輸器的郵件程式定義：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```


<a name="additional-symfony-transports"></a>
### 額外 Symfony 傳輸器

Laravel 支援一些現有的 Symfony 維護的郵件傳輸器，如 Mailgun 和 Postmark。然而，您可能希望擴展 Laravel 以支援額外的 Symfony 維護的傳輸器。您可以透過 Composer 安裝必要的 Symfony 郵件程式並向 Laravel 註冊該傳輸器來做到這一點。例如，您可以安裝並註冊「Brevo」（以前稱為「Sendinblue」）Symfony 郵件程式：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

一旦 Brevo 郵件程式套件安裝完成，您可以將 Brevo API 憑證的條目新增到應用程式的 `services` 設定檔中：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接下來，您可以使用 `Mail` Facade 的 `extend` 方法來向 Laravel 註冊該傳輸器。通常，這應該在服務提供者的 `boot` 方法中完成：

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

一旦您的傳輸器註冊完成，您可以在應用程式的 `config/mail.php` 設定檔中建立一個使用這個新傳輸器的郵件程式定義：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```