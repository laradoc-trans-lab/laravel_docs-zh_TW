# 郵件

- [簡介](#introduction)
    - [配置](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
    - [故障轉移配置](#failover-configuration)
    - [循環配置](#round-robin-configuration)
- [生成 Mailables](#generating-mailables)
- [撰寫 Mailables](#writing-mailables)
    - [配置寄件者](#configuring-the-sender)
    - [配置視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [行內附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與中繼資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailables](#markdown-mailables)
    - [生成 Markdown Mailables](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
    - [自訂元件](#customizing-the-components)
- [寄送郵件](#sending-mail)
    - [郵件佇列](#queueing-mail)
- [渲染 Mailables](#rendering-mailables)
    - [在瀏覽器中預覽 Mailables](#previewing-mailables-in-the-browser)
- [本地化 Mailables](#localizing-mailables)
- [測試 Mailables](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 寄送](#testing-mailable-sending)
- [郵件與本地開發](#mail-and-local-development)
- [事件](#events)
- [自訂傳輸器](#custom-transports)
    - [額外 Symfony 傳輸器](#additional-symfony-transports)

<a name="introduction"></a>
## 簡介

寄送電子郵件不必複雜。Laravel 提供了由廣受歡迎的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件支援的簡潔、簡單的電子郵件 API。Laravel 與 Symfony Mailer 提供了透過 SMTP、Mailgun、Postmark、Resend、Amazon SES 和 `sendmail` 寄送電子郵件的驅動程式，讓您能夠快速地透過您選擇的本地或雲端服務開始寄送郵件。

<a name="configuration"></a>
### 配置

Laravel 的電子郵件服務可以透過應用程式的 `config/mail.php` 設定檔進行配置。此檔案中配置的每個郵件發送器都可以擁有自己獨特的配置，甚至擁有自己獨特的「傳輸器」，這使得您的應用程式可以使用不同的電子郵件服務來寄送特定的電子郵件訊息。例如，您的應用程式可能使用 Postmark 來寄送交易性電子郵件，同時使用 Amazon SES 來寄送大量電子郵件。

在您的 `mail` 設定檔中，您會找到一個 `mailers` 配置陣列。此陣列包含了 Laravel 支援的每個主要郵件驅動程式／傳輸器的範例配置項目，而 `default` 配置值則決定了當您的應用程式需要寄送電子郵件訊息時，預設將使用哪個郵件發送器。

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

基於 API 的驅動程式，例如 Mailgun、Postmark 和 Resend，通常比透過 SMTP 伺服器寄送郵件更簡單、更快速。在可能的情況下，我們建議您使用這些驅動程式之一。

<a name="mailgun-driver"></a>
#### Mailgun 驅動程式

若要使用 Mailgun 驅動程式，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳輸器：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接著，您需要在應用程式的 `config/mail.php` 設定檔中進行兩項變更。首先，將您的預設郵件發送器設定為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

其次，將以下配置陣列新增到您的 `mailers` 陣列中：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

配置應用程式的預設郵件發送器後，將以下選項新增到您的 `config/services.php` 設定檔：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

如果您未使用美國 [Mailgun 地區](https://documentation.mailgun.com/docs/mailgun/api-reference/#mailgun-regions)，您可以在 `services` 設定檔中定義您地區的端點：

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

接著，將您應用程式的 `config/mail.php` 設定檔中的 `default` 選項設定為 `postmark`。配置好應用程式的預設郵件發送器後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'postmark' => [
    'token' => env('POSTMARK_TOKEN'),
],
```

如果您想要指定特定郵件發送器應使用的 Postmark 訊息串流，您可以將 `message_stream_id` 配置選項新增到該郵件發送器的配置陣列中。此配置陣列位於您應用程式的 `config/mail.php` 設定檔：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

透過這種方式，您還可以設定多個 Postmark 郵件發送器，每個發送器使用不同的訊息串流。

<a name="resend-driver"></a>
#### Resend 驅動程式

若要使用 [Resend](https://resend.com/) 驅動程式，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接著，將您應用程式的 `config/mail.php` 設定檔中的 `default` 選項設定為 `resend`。配置好應用程式的預設郵件發送器後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'resend' => [
    'key' => env('RESEND_KEY'),
],
```

<a name="ses-driver"></a>
#### SES 驅動程式

若要使用 Amazon SES 驅動程式，您必須先安裝 Amazon AWS SDK for PHP。您可以透過 Composer 套件管理器安裝此程式庫：

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

若要透過工作階段憑證使用 AWS [臨時憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，您可以將 `token` 鍵新增到應用程式的 SES 配置中：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

若要與 SES 的 [訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html) 互動，您可以將 `X-Ses-List-Management-Options` 標頭回傳在郵件訊息的 [headers](#headers) 方法所回傳的陣列中：

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

如果您想定義 Laravel 在寄送電子郵件時應傳遞給 AWS SDK 的 `SendEmail` 方法的[額外選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，您可以在 `ses` 配置中定義一個 `options` 陣列：

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

<a name="failover-configuration"></a>
### 故障轉移配置

有時，您配置來寄送應用程式郵件的外部服務可能會中斷。在這些情況下，定義一個或多個備用郵件寄送配置會很有用，這些配置將在主要寄送驅動程式中斷時使用。

為了實現這一點，您應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳輸器的郵件發送器。應用程式的 `failover` 郵件發送器的配置陣列應該包含一個 `mailers` 陣列，其中引用了應選擇配置的郵件發送器進行寄送的順序：

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

一旦定義了故障轉移郵件發送器，您就應該透過在應用程式的 `mail` 設定檔中將其名稱指定為 `default` 配置鍵的值，來將此郵件發送器設定為應用程式使用的預設郵件發送器：

```php
'default' => env('MAIL_MAILER', 'failover'),
```

<a name="round-robin-configuration"></a>
### 循環配置

`roundrobin` 傳輸器允許您將郵件工作負載分散到多個郵件發送器。首先，在應用程式的 `mail` 配置檔中定義一個使用 `roundrobin` 傳輸器的郵件發送器。應用程式的 `roundrobin` 郵件發送器的配置陣列應包含一個 `mailers` 陣列，用來參考應使用哪些已配置的郵件發送器進行遞送：

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

定義好循環郵件發送器後，您應該將其名稱指定為應用程式 `mail` 配置檔中 `default` 配置鍵的值，將此郵件發送器設定為應用程式使用的預設郵件發送器：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

循環傳輸器會從已配置的郵件發送器列表中隨機選擇一個郵件發送器，然後為每個後續電子郵件切換到下一個可用的郵件發送器。與有助於實現 *[高可用性](https://en.wikipedia.org/wiki/High_availability)* 的 `failover` 傳輸器不同，`roundrobin` 傳輸器則提供 *[負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))*.

<a name="generating-mailables"></a>
## 生成 Mailables

在建立 Laravel 應用程式時，應用程式發送的每種類型電子郵件都會以一個「mailable」類別來表示。這些類別儲存於 `app/Mail` 目錄中。如果您在應用程式中沒有看到這個目錄，請不用擔心，因為當您使用 `make:mail` Artisan 指令建立第一個 mailable 類別時，它會為您生成：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailables

生成 Mailable 類別後，開啟它，以便我們探索其內容。Mailable 類別的配置是透過多個方法完成的，這些方法包括 `envelope`、`content` 和 `attachments`。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，此物件定義了訊息的主旨，有時也定義了收件者。`content` 方法會回傳一個 `Illuminate\Mail\Mailables\Content` 物件，此物件定義了將用於生成訊息內容的 [Blade 模板](/docs/{{version}}/blade)。

<a name="configuring-the-sender"></a>
### 配置寄件者

<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，讓我們探索如何配置電子郵件的寄件者。換句話說，就是電子郵件的「發件者 (from)」是誰。有兩種方式可以配置寄件者。首先，您可以透過訊息的 envelope 指定「發件者 (from)」位址：

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

如果您願意，也可以指定一個 `replyTo` 位址：

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

然而，如果您的應用程式的所有電子郵件都使用相同的「發件者 (from)」位址，將其新增到您生成的每個 Mailable 類別中可能會變得繁瑣。您可以改為在 `config/mail.php` 配置檔案中指定一個全域的「發件者 (from)」位址。如果 Mailable 類別中沒有指定其他「發件者 (from)」位址，則會使用此位址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，您可以在 `config/mail.php` 配置檔案中定義一個全域的「回覆 (reply_to)」位址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```

<a name="configuring-the-view"></a>
### 配置視圖

在 Mailable 類別的 `content` 方法內，您可以定義 `view` (視圖)，即渲染電子郵件內容時應使用的模板。由於每封電子郵件通常會使用 [Blade 模板](/docs/{{version}}/blade) 來渲染其內容，因此在構建電子郵件的 HTML 時，您可以使用 Blade 模板引擎的全部功能與便利性：

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
> 您可能會希望建立一個 `resources/views/mail` 目錄來存放所有電子郵件模板；然而，您可以自由地將它們放置在 `resources/views` 目錄內的任何位置。

<a name="plain-text-emails"></a>
#### 純文字電子郵件

如果您想定義電子郵件的純文字版本，可以在建立訊息的 `Content` 定義時指定純文字模板。如同 `view` 參數一樣，`text` 參數也應該是一個將用於渲染電子郵件內容的模板名稱。您可以自由地定義訊息的 HTML 和純文字版本：

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

為了清晰起見，`html` 參數可用作 `view` 參數的別名：

```php
return new Content(
    html: 'mail.orders.shipped',
    text: 'mail.orders.shipped-text'
);
```

<a name="view-data"></a>
### 視圖資料

<a name="via-public-properties"></a>
#### 透過公共屬性

通常，您會希望將一些資料傳遞到您的視圖中，以便在渲染電子郵件的 HTML 時利用這些資料。有兩種方式可以讓資料在您的視圖中可用。首先，Mailable 類別上定義的任何公共屬性都將自動提供給視圖。因此，舉例來說，您可以將資料傳遞到 Mailable 類別的建構函式中，並將該資料設定為類別上定義的公共屬性：

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

一旦資料已設定為公共屬性，它將自動在您的視圖中可用，因此您可以像您在 Blade 模板中存取任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $order->price }}
</div>
```

<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果您想在電子郵件資料傳送到模板之前自訂其格式，可以透過 `Content` 定義的 `with` 參數手動將資料傳遞到視圖。通常，您仍然會透過 Mailable 類別的建構函式傳遞資料；然而，您應該將這些資料設定為 `protected` 或 `private` 屬性，以便資料不會自動提供給模板：

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

一旦資料已透過 `with` 參數傳遞，它將自動在您的視圖中可用，因此您可以像您在 Blade 模板中存取任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

要為郵件添加附件，您可以將附件添加到訊息的 `attachments` 方法所回傳的陣列中。首先，您可以透過提供檔案路徑給 `Attachment` 類別的 `fromPath` 方法來添加附件：

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

當附加檔案到訊息時，您也可以使用 `as` 和 `withMime` 方法來指定附件的顯示名稱和/或 MIME 類型：

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

如果您已將檔案儲存在您的 [檔案系統磁碟](/docs/{{version}}/filesystem) 之一上，您可以使用 `fromStorage` 附件方法將其附加到電子郵件中：

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

如果您需要指定除了預設磁碟以外的儲存磁碟，可以使用 `fromStorageDisk` 方法：

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

`fromData` 附件方法可用於將原始位元組字串作為附件附加。例如，如果您在記憶體中生成了 PDF 並希望將其附加到電子郵件而無需寫入磁碟，則可以使用此方法。`fromData` 方法接受一個閉包，該閉包解析原始資料位元組以及附件應被賦予的名稱：

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

在電子郵件中嵌入行內圖片通常很麻煩；然而，Laravel 提供了一種方便的方式來將圖片附加到您的電子郵件中。要嵌入行內圖片，請在電子郵件模板中使用 `$message` 變數上的 `embed` 方法。Laravel 會自動將 `$message` 變數提供給您所有的電子郵件模板，因此您無需擔心手動傳遞它：

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]
> `$message` 變數在純文字訊息模板中不可用，因為純文字訊息不使用行內附件。


<a name="embedding-raw-data-attachments"></a>
#### 嵌入原始資料附件

如果您已經有一個想要嵌入到電子郵件模板中的原始圖片資料字串，您可以在 `$message` 變數上呼叫 `embedData` 方法。呼叫 `embedData` 方法時，您需要提供一個應分配給嵌入圖片的檔案名稱：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```


<a name="attachable-objects"></a>
### 可附加物件

儘管透過簡單的字串路徑將檔案附加到訊息通常已足夠，但在許多情況下，應用程式中可附加的實體是由類別表示的。例如，如果您的應用程式將照片附加到訊息中，您的應用程式可能還有一個表示該照片的 `Photo` 模型。在這種情況下，如果能夠直接將 `Photo` 模型傳遞給 `attach` 方法，豈不是很方便？可附加物件讓您能夠做到這一點。

首先，在將附加到訊息的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。此介面要求您的類別定義一個 `toMailAttachment` 方法，該方法回傳一個 `Illuminate\Mail\Attachment` 實例：

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

定義好您的可附加物件後，您可以在建構電子郵件訊息時，從 `attachments` 方法回傳該物件的一個實例：

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

當然，附件資料可以儲存在遠端檔案儲存服務上，例如 Amazon S3。因此，Laravel 也允許您從儲存在應用程式 [檔案系統磁碟](/docs/{{version}}/filesystem) 之一上的資料生成附件實例：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，您還可以透過記憶體中的資料來建立附件實例。要實現這一點，請向 `fromData` 方法提供一個閉包。該閉包應回傳代表附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 還提供了您可以用來自訂附件的其他方法。例如，您可以使用 `as` 和 `withMime` 方法來自訂檔案的名稱和 MIME 類型：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```


<a name="headers"></a>
### 標頭

有時您可能需要為外寄訊息附加額外的標頭。例如，您可能需要設定自訂的 `Message-Id` 或其他任意的文字標頭。

為實現此目的，請在您的 mailable 上定義一個 `headers` 方法。`headers` 方法應回傳一個 `Illuminate\Mail\Mailables\Headers` 實例。此類別接受 `messageId`、`references` 和 `text` 參數。當然，您可以只提供您的特定訊息所需的參數：

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

某些第三方電子郵件服務商，例如 Mailgun 和 Postmark，支援訊息「標籤 (tags)」和「中繼資料 (metadata)」，可用於對應用程式發送的電子郵件進行分組和追蹤。您可以透過 `Envelope` 定義為電子郵件訊息添加標籤和中繼資料：

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

如果您的應用程式正在使用 Mailgun 驅動程式，您可以查閱 Mailgun 的文件，以獲取有關[標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags)和[中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages)的更多資訊。同樣地，Postmark 文件也提供了有關其[標籤](https://postmarkapp.com/blog/tags-support-for-smtp)和[中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq)支援的更多資訊。

如果您的應用程式正在使用 Amazon SES 發送電子郵件，您應該使用 `metadata` 方法為訊息附加 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能由 Symfony Mailer 提供支援。Laravel 允許您註冊自訂的回呼，這些回呼將在發送訊息之前與 Symfony Message 實例一起被調用。這讓您有機會在發送訊息之前深度自訂訊息。為此，您可以在 `Envelope` 定義上定義一個 `using` 參數：

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

Markdown mailable 訊息讓您可以在 mailable 中利用 [郵件通知](/docs/{{version}}/notifications#mail-notifications) 的預建模板和元件。由於訊息是以 Markdown 撰寫，Laravel 能夠為訊息渲染出美觀、響應式的 HTML 模板，同時也會自動生成純文字版本。


<a name="generating-markdown-mailables"></a>
### 生成 Markdown Mailables

要生成帶有對應 Markdown 模板的 mailable，您可以使用 `make:mail` Artisan 命令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

然後，在 mailable 的 `content` 方法中配置其 `Content` 定義時，請使用 `markdown` 參數而不是 `view` 參數：

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

Markdown mailables 結合了 Blade 元件和 Markdown 語法，讓您能夠輕鬆構建郵件訊息，同時利用 Laravel 預建的電子郵件 UI 元件：

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
> 在撰寫 Markdown 電子郵件時，請勿使用過度縮排。根據 Markdown 標準，Markdown 解析器會將縮排內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染一個置中按鈕連結。此元件接受兩個參數，一個是 `url`，另一個是可選的 `color`。支援的顏色有 `primary`、`success` 和 `error`。您可以根據需要添加任意數量的按鈕元件到訊息中：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板元件

面板元件會將給定的文字區塊渲染在背景顏色與訊息其餘部分略有不同的面板中。這讓您可以引起對特定文字區塊的注意：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格元件

表格元件允許您將 Markdown 表格轉換為 HTML 表格。此元件接受 Markdown 表格作為其內容。表格欄位對齊支援使用預設的 Markdown 表格對齊語法：

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

您可以將所有 Markdown 郵件元件匯出到自己的應用程式中進行自訂。要匯出這些元件，請使用 `vendor:publish` Artisan 命令來發佈 `laravel-mail` 資源標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此命令會將 Markdown 郵件元件發佈到 `resources/views/vendor/mail` 目錄。`mail` 目錄將包含 `html` 和 `text` 目錄，每個目錄都包含所有可用元件的各自表示。您可以隨意自訂這些元件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄將包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將自動轉換為 Markdown 郵件訊息的 HTML 表示中的行內 CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建立一個全新的主題，您可以將 CSS 檔案放置在 `html/themes` 目錄中。命名並儲存您的 CSS 檔案後，請更新應用程式 `config/mail.php` 設定檔中的 `theme` 選項，以匹配您的新主題名稱。

要為單個 mailable 自訂主題，您可以將 mailable 類別的 `$theme` 屬性設定為寄送該 mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 寄送郵件

要寄送訊息，請使用 `Mail` [Facade](/docs/{{version}}/facades) 上的 `to` 方法。`to` 方法接受一個電子郵件地址、一個使用者實例或一個使用者集合。如果您傳遞一個物件或物件集合，郵件傳送器會自動使用它們的 `email` 和 `name` 屬性來決定電子郵件的收件者，因此請確保這些屬性在您的物件上是可用的。一旦您指定了收件者，您就可以將 Mailable 類別的實例傳遞給 `send` 方法：

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

您不限於只指定「to」收件者當寄送訊息時。您可以自由設定「to」、「cc」和「bcc」收件者，透過鏈式呼叫它們各自的方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```

<a name="looping-over-recipients"></a>
#### 遍歷收件者

有時，您可能需要透過遍歷一個收件者 / 電子郵件地址陣列來將 Mailable 傳送給收件者列表。然而，由於 `to` 方法會將電子郵件地址附加到 Mailable 的收件者列表，每次迴圈迭代都會向所有先前的收件者傳送另一封電子郵件。因此，您應該始終為每個收件者重新建立 Mailable 實例：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```

<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過特定郵件傳送器寄送郵件

預設情況下，Laravel 會使用您應用程式 `mail` 設定檔中配置為 `default` 郵件傳送器來寄送電子郵件。然而，您可以使用 `mailer` 方法來寄送訊息，透過特定的郵件傳送器設定：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```

<a name="queueing-mail"></a>
### 郵件佇列

<a name="queueing-a-mail-message"></a>
#### 將郵件訊息加入佇列

由於寄送電子郵件訊息可能會對您應用程式的回應時間產生負面影響，許多開發者選擇將電子郵件訊息加入佇列以進行背景寄送。Laravel 透過其內建的[統一佇列 API](/docs/{{version}}/queues) 讓這件事變得容易。要將郵件訊息加入佇列，請在指定訊息的收件者後使用 `Mail` Facade 上的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法會自動將一個任務推送到佇列中，以便訊息在背景中寄送。在使用此功能之前，您需要[設定您的佇列](/docs/{{version}}/queues)。

<a name="delayed-message-queueing"></a>
#### 延遲訊息佇列

如果您希望延遲佇列電子郵件訊息的傳送，您可以使用 `later` 方法。作為其第一個參數，`later` 方法接受一個 `DateTime` 實例，用來指示訊息應該何時寄送：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->addMinutes(10), new OrderShipped($order));
```

<a name="pushing-to-specific-queues"></a>
#### 推送到特定佇列

由於所有使用 `make:mail` 命令生成的 Mailable 類別都使用了 `Illuminate\Bus\Queueable` Trait，您可以在任何 Mailable 類別實例上呼叫 `onQueue` 和 `onConnection` 方法，讓您能夠為訊息指定連線和佇列名稱：

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
#### 預設情況下加入佇列

如果您有一些 Mailable 類別希望總是自動加入佇列，您可以在該類別上實作 `ShouldQueue` 契約。現在，即使您在寄送郵件時呼叫 `send` 方法，該 Mailable 仍會被加入佇列，因為它實作了該契約：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```

<a name="queued-mailables-and-database-transactions"></a>
#### 佇列式 Mailables 與資料庫交易

當佇列式 Mailables 在資料庫交易中分派時，它們可能會在資料庫交易提交之前被佇列處理。當這種情況發生時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫記錄可能不存在於資料庫中。如果您的 Mailable 依賴這些模型，當處理傳送佇列式 Mailable 的任務時，可能會發生意外錯誤。

如果您的佇列連線的 `after_commit` 配置選項設為 `false`，您仍然可以指示某個佇列式 Mailable 應在所有開放的資料庫交易提交後分派，透過在傳送郵件訊息時呼叫 `afterCommit` 方法：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，您可以在 Mailable 的建構子中呼叫 `afterCommit` 方法：

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
> 要了解更多關於解決這些問題的方法，請查閱有關[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="queued-email-failures"></a>
#### 佇列郵件失敗

當一個佇列郵件失敗時，佇列式 Mailable 類別上的 `failed` 方法將會被呼叫，如果它已定義。導致佇列郵件失敗的 `Throwable` 實例將會傳遞給 `failed` 方法：

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

有時您可能希望擷取一個 Mailable 的 HTML 內容而不實際寄送它。為了實現這一點，您可以呼叫 Mailable 的 `render` 方法。此方法會將 Mailable 評估後的 HTML 內容作為字串回傳：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```

<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailables

當設計 Mailable 的模板時，方便在瀏覽器中快速預覽渲染後的 Mailable，就像一般的 Blade 模板一樣。基於這個原因，Laravel 允許您直接從路由閉包或控制器中回傳任何 Mailable。當 Mailable 被回傳時，它將在瀏覽器中被渲染並顯示，讓您能夠快速預覽其設計而無需將其寄送到實際的電子郵件地址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

<a name="localizing-mailables"></a>
## 本地化 Mailables

Laravel 允許您以不同於請求當前語系的語系寄送 Mailable，即使郵件進入佇列，也會記住此語系。

為此，`Mail` facade 提供了 `locale` 方法來設定所需的語言。當 Mailable 的模板正在評估時，應用程式將切換到此語系，並在評估完成後恢復到先前的語系。

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```


<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時，應用程式會儲存每個使用者的偏好語系。透過在一個或多個模型上實作 `HasLocalePreference` 契約，您可以指示 Laravel 在寄送郵件時使用此儲存的語系。

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

一旦您實作了此介面，Laravel 將在向模型寄送 Mailable 和通知時自動使用偏好語系。因此，當使用此介面時，無需呼叫 `locale` 方法。

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試 Mailables


<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了多種方法來檢查您的 mailable 結構。此外，Laravel 還提供了幾個方便的方法，用於測試您的 mailable 是否包含您預期的內容：

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

如您所料，「HTML」斷言用於斷言您 mailable 的 HTML 版本包含給定字串，而「文字」斷言則用於斷言您 mailable 的純文字版本包含給定字串。

<a name="testing-mailable-sending"></a>
### 測試 Mailable 寄送

我們建議將測試 Mailable 內容與測試特定 Mailable 是否「寄送」給特定使用者的測試分開進行。通常，Mailable 的內容與您正在測試的程式碼無關，因此只需斷言 Laravel 已指示寄送特定 Mailable 即可。

您可以使用 `Mail` 外觀的 `fake` 方法來防止郵件被寄送。在呼叫 `Mail` 外觀的 `fake` 方法後，您可以斷言 Mailable 已被指示寄送給使用者，甚至檢查 Mailable 收到的資料：

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

如果您正在將 Mailable 加入佇列以進行背景傳送，您應該使用 `assertQueued` 方法而不是 `assertSent` 方法：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

您可以將閉包傳遞給 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以斷言寄送的 Mailable 通過了特定的「真實性測試」。如果至少有一個 Mailable 通過了給定的真實性測試，則該斷言將會成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

當呼叫 `Mail` 外觀的斷言方法時，所提供的閉包接受的 Mailable 實例會公開有用的方法，用於檢查該 Mailable：

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

Mailable 實例還包含幾個有用的方法，用於檢查 Mailable 上的附件：

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

您可能已經注意到，有兩種斷言郵件未被寄送的方法：`assertNotSent` 和 `assertNotQueued`。有時您可能希望斷言沒有郵件被寄送**或**加入佇列。為此，您可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本地開發

在開發寄送電子郵件的應用程式時，您可能不想實際將電子郵件寄送到真實的電子郵件地址。Laravel 提供了幾種方法來「停用」在本地開發期間實際寄送電子郵件的功能。

<a name="log-driver"></a>
#### 日誌驅動程式

`log` 郵件驅動程式不會寄送您的電子郵件，而是會將所有電子郵件訊息寫入您的日誌檔案以供檢查。通常，此驅動程式僅在本地開發期間使用。有關每個環境的應用程式配置的更多資訊，請查閱 [配置文件](/docs/{{version}}/configuration#environment-configuration)。

<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

另外，您可以使用諸如 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 之類的服務以及 `smtp` 驅動程式，將您的電子郵件訊息寄送到「虛擬」信箱，您可以在其中以真實的電子郵件客戶端查看它們。這種方法的好處是讓您可以實際檢查 Mailtrap 訊息檢視器中的最終電子郵件。

如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，您可以使用 [Mailpit](https://github.com/axllent/mailpit) 預覽您的訊息。當 Sail 運行時，您可以透過 `http://localhost:8025` 訪問 Mailpit 界面。

<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 地址

最後，您可以透過呼叫 `Mail` facade 提供的 `alwaysTo` 方法來指定一個全域「to」地址。通常，此方法應在您的應用程式服務提供者之一的 `boot` 方法中呼叫：

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

當使用 `alwaysTo` 方法時，郵件訊息中任何額外的「cc」或「bcc」地址將會被移除。

<a name="events"></a>
## 事件

Laravel 在寄送郵件訊息時會觸發兩個事件。`MessageSending` 事件在訊息寄出之前觸發，而 `MessageSent` 事件在訊息寄出之後觸發。請記住，這些事件是在郵件被 *寄送* 時觸發，而不是在排入佇列時觸發。您可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

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

Laravel 包含了各種郵件傳輸器；然而，您可能希望撰寫自己的傳輸器，以透過 Laravel 不支援的服務來寄送電子郵件。首先，定義一個擴充了 `Symfony\Component\Mailer\Transport\AbstractTransport` 類別的類別。然後，在您的傳輸器中實作 `doSend` 和 `__toString` 方法：

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

一旦您定義了自訂傳輸器，即可透過 `Mail` facade 提供的 `extend` 方法註冊它。通常，這應該在您的應用程式 `AppServiceProvider` 的 `boot` 方法中完成。一個 `$config` 參數將傳遞給提供給 `extend` 方法的閉包。此參數將包含為應用程式 `config/mail.php` 配置檔案中的郵件發送器定義的配置陣列：

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

一旦您的自訂傳輸器已定義並註冊，您可以在應用程式的 `config/mail.php` 配置檔案中建立一個郵件發送器定義，該定義使用新的傳輸器：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```

<a name="additional-symfony-transports"></a>
### 額外 Symfony 傳輸器

Laravel 包含了對一些現有的 Symfony 維護的郵件傳輸器的支援，例如 Mailgun 和 Postmark。然而，您可能希望擴展 Laravel 以支援額外的 Symfony 維護的傳輸器。您可以透過 Composer 引入必要的 Symfony 郵件發送器並向 Laravel 註冊該傳輸器來實現此目的。例如，您可以安裝並註冊「Brevo」（前身為「Sendinblue」）Symfony 郵件發送器：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

一旦 Brevo 郵件發送器套件安裝完成，您可以在應用程式的 `services` 配置檔案中為您的 Brevo API 憑證新增一個條目：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接下來，您可以使用 `Mail` facade 的 `extend` 方法向 Laravel 註冊該傳輸器。通常，這應該在服務提供者的 `boot` 方法中完成：

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

一旦您的傳輸器已註冊，您可以在應用程式的 `config/mail.php` 配置檔案中建立一個郵件發送器定義，該定義使用新的傳輸器：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```