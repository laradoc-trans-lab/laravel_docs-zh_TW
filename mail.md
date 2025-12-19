# 郵件

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動器前置準備](#driver-prerequisites)
    - [故障轉移設定](#failover-configuration)
    - [輪詢設定](#round-robin-configuration)
- [生成 Mailable](#generating-mailables)
- [撰寫 Mailable](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [內嵌附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與元資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailable](#markdown-mailables)
    - [生成 Markdown Mailable](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
    - [自訂元件](#customizing-the-components)
- [傳送郵件](#sending-mail)
    - [佇列郵件](#queueing-mail)
- [渲染 Mailable](#rendering-mailables)
    - [在瀏覽器中預覽 Mailable](#previewing-mailables-in-the-browser)
- [在地化 Mailable](#localizing-mailables)
- [測試](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 傳送](#testing-mailable-sending)
- [郵件與本地開發](#mail-and-local-development)
- [事件](#events)
- [自訂傳輸器](#custom-transports)
    - [額外的 Symfony 傳輸器](#additional-symfony-transports)

<a name="introduction"></a>
## 簡介

傳送電子郵件不一定非得這麼複雜。Laravel 提供了一個簡潔、簡單的郵件 API，由受歡迎的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件驅動。Laravel 和 Symfony Mailer 提供了透過 SMTP、Mailgun、Postmark、Resend、Amazon SES 和 `sendmail` 傳送郵件的驅動器，讓您可以快速開始透過您選擇的本地或雲端服務傳送郵件。


<a name="configuration"></a>
### 設定

Laravel 的郵件服務可以透過應用程式的 `config/mail.php` 設定檔進行設定。在此檔案中設定的每個郵件程式 (Mailer) 都可以有自己獨特的設定，甚至有自己獨特的「傳輸器 (Transport)」，讓您的應用程式能使用不同的郵件服務來傳送特定的郵件訊息。例如，您的應用程式可能會使用 Postmark 來傳送交易式郵件，而使用 Amazon SES 來傳送大量郵件。

在您的 `mail` 設定檔中，您會找到一個 `mailers` 設定陣列。此陣列包含了 Laravel 支援的每個主要郵件驅動器 / 傳輸器的範例設定項目，而 `default` 設定值則決定了當您的應用程式需要傳送郵件時，預設會使用哪個郵件程式。


<a name="driver-prerequisites"></a>
### 驅動器前置準備

基於 API 的驅動器（例如 Mailgun、Postmark 和 Resend）通常比透過 SMTP 伺服器傳送郵件更簡單、更快速。我們建議您盡可能使用其中一種驅動器。


<a name="mailgun-driver"></a>
#### Mailgun 驅動器

若要使用 Mailgun 驅動器，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳輸器：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接著，您需要在應用程式的 `config/mail.php` 設定檔中進行兩項更改。首先，將您的預設郵件程式設定為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

第二，將以下設定陣列新增到您的 `mailers` 陣列中：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

在設定好應用程式的預設郵件程式後，請將以下選項新增到您的 `config/services.php` 設定檔中：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

如果您使用的不是美國的 [Mailgun 區域](https://documentation.mailgun.com/docs/mailgun/api-reference/#mailgun-regions)，您可以在 `services` 設定檔中定義您區域的端點 (Endpoint)：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.eu.mailgun.net'),
    'scheme' => 'https',
],
```


<a name="postmark-driver"></a>
#### Postmark 驅動器

若要使用 [Postmark](https://postmarkapp.com/) 驅動器，請透過 Composer 安裝 Symfony 的 Postmark Mailer 傳輸器：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接著，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `postmark`。在設定好應用程式的預設郵件程式後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'postmark' => [
    'key' => env('POSTMARK_API_KEY'),
],
```

如果您想指定特定郵件程式應使用的 Postmark 訊息串流 (Message Stream)，您可以在郵件程式的設定陣列中加入 `message_stream_id` 設定選項。此設定陣列可以在您應用程式的 `config/mail.php` 設定檔中找到：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

透過這種方式，您還可以設定多個具有不同訊息串流的 Postmark 郵件程式。


<a name="resend-driver"></a>
#### Resend 驅動器

若要使用 [Resend](https://resend.com/) 驅動器，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接著，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `resend`。在設定好應用程式的預設郵件程式後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'resend' => [
    'key' => env('RESEND_API_KEY'),
],
```


<a name="ses-driver"></a>
#### SES 驅動器

若要使用 Amazon SES 驅動器，您必須先安裝適用於 PHP 的 Amazon AWS SDK。您可以透過 Composer 套件管理員安裝此函式庫：

```shell
composer require aws/aws-sdk-php
```

接著，將 `config/mail.php` 設定檔中的 `default` 選項設定為 `ses`，並確認您的 `config/services.php` 設定檔包含以下選項：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

若要透過工作階段權杖 (Session Token) 使用 AWS [暫時憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，您可以向應用程式的 SES 設定中加入 `token` 鍵：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

若要與 SES 的 [訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html) 互動，您可以在郵件訊息的 [headers](#headers) 方法回傳的陣列中，回傳 `X-Ses-List-Management-Options` 標頭：

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

如果您想定義 Laravel 在傳送電子郵件時應傳遞給 AWS SDK `SendEmail` 方法的 [額外選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，您可以在 `ses` 設定中定義一個 `options` 陣列：

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
### 故障轉移設定

有時，您設定用來傳送應用程式郵件的外部服務可能會發生故障。在這種情況下，定義一個或多個備用郵件遞送設定會很有幫助，這些設定將在您的主要遞送驅動器發生故障時被使用。

為了實現這一點，您應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳輸器的郵件程式。應用程式 `failover` 郵件程式的設定陣列應包含一個 `mailers` 陣列，該陣列參照了應選擇遞送的郵件程式順序：

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

一旦您設定了使用 `failover` 傳輸器的郵件程式，您將需要在應用程式的 `.env` 檔案中將故障轉移郵件程式設定為預設郵件程式，以使用故障轉移功能：

```ini
MAIL_MAILER=failover
```

<a name="round-robin-configuration"></a>
### 輪詢設定

`roundrobin` 傳輸器允許您將郵件傳送的工作量分配到多個郵件寄送器中。若要開始使用，請在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` 傳輸器的郵件寄送器。應用程式中 `roundrobin` 郵件寄送器的設定陣列應包含一個 `mailers` 陣列，用以引用哪些已設定的郵件寄送器應被用於傳送：

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

一旦定義了輪詢郵件寄送器，您應該將此郵件寄送器設定為應用程式使用的預設郵件寄送器，方法是在應用程式的 `mail` 設定檔中將其名稱指定為 `default` 設定鍵的值：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

輪詢傳輸器會從已設定的郵件寄送器清單中隨機選擇一個，然後為之後的每封電子郵件切換到下一個可用的郵件寄送器。與有助於實現 *[高可用性](https://en.wikipedia.org/wiki/High_availability)* 的 `failover` 傳輸器不同，`roundrobin` 傳輸器提供的是 *[負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))*。

<a name="generating-mailables"></a>
## 生成 Mailable

在開發 Laravel 應用程式時，應用程式傳送的每一種郵件類型都由一個「mailable」類別表示。這些類別存放在 `app/Mail` 目錄中。如果你在應用程式中沒看到這個目錄，請不用擔心，因為當你使用 `make:mail` Artisan 指令建立第一個 mailable 類別時，系統會自動為你生成該目錄：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailable

生成 Mailable 類別後，請將其開啟，讓我們來探索其內容。Mailable 類別的設定是透過幾個方法完成的，包含 `envelope`、`content` 和 `attachments` 方法。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，用來定義郵件的主旨，有時也包含收件者。`content` 方法則會回傳一個 `Illuminate\Mail\Mailables\Content` 物件，用來定義產生郵件內容時所使用的 [Blade 樣板](/docs/{{version}}/blade)。


<a name="configuring-the-sender"></a>
### 設定寄件者


<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，讓我們來探索如何設定郵件的寄件者。或者換句話說，也就是郵件要從（"from"）誰那裡寄出。設定寄件者有兩種方式。第一種，您可以在郵件的 envelope 中指定 "from" 地址：

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

如果您願意，也可以指定一個 `replyTo` 地址：

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
#### 使用全域 `from` 地址

然而，若您的應用程式在所有郵件中都使用相同的「寄件者 (from)」地址，那麼在每個生成的 Mailable 類別中都加上它會變得很繁瑣。相反地，您可以在 `config/mail.php` 設定檔中指定一個全域的「寄件者 (from)」地址。若 Mailable 類別中沒有指定其他「寄件者」地址，則會使用此地址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，您也可以在 `config/mail.php` 設定檔中定義一個全域的 "reply_to" 地址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```


<a name="configuring-the-view"></a>
### 設定視圖

在 Mailable 類別的 `content` 方法中，您可以定義 `view`，也就是在渲染郵件內容時應使用哪個樣板。由於每封郵件通常都使用 [Blade 樣板](/docs/{{version}}/blade) 來渲染其內容，因此在建構郵件的 HTML 時，您可以使用 Blade 樣板引擎提供的完整功能與便利性：

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
> 您可能希望建立一個 `resources/views/mail` 目錄來存放所有的郵件樣板；不過，您也可以自由地將它們放在 `resources/views` 目錄下的任何位置。


<a name="plain-text-emails"></a>
#### 純文字郵件

如果您想定義郵件的純文字版本，可以在建立郵件的 `Content` 定義時指定純文字樣板。與 `view` 參數類似，`text` 參數應該是將用於渲染郵件內容的樣板名稱。您可以自由地為郵件定義 HTML 和純文字版本：

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

為了清楚起見，`html` 參數可以作為 `view` 參數的別名：

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

通常，您會希望將一些資料傳遞給視圖，以便在渲染郵件的 HTML 時使用。有兩種方法可以讓資料在視圖中可用。首先，在 Mailable 類別上定義的任何公開屬性 (Public Property) 都會自動在視圖中可用。例如，您可以將資料傳入 Mailable 類別的建構子 (Constructor)，並將該資料設定為類別上定義的公開屬性：

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

一旦資料被設定為公開屬性後，它就會自動在您的視圖中可用，因此您可以像在 Blade 樣板中存取任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $order->price }}
</div>
```


<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果您想在將郵件資料傳送到樣板之前自訂其格式，可以透過 `Content` 定義的 `with` 參數手動將資料傳遞給視圖。通常，您仍然會透過 Mailable 類別的建構子傳遞資料；但是，您應該將這些資料設定為 `protected` 或 `private` 屬性，這樣資料才不會自動在樣板中可用：

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

一旦資料透過 `with` 參數傳遞後，它就會自動在您的視圖中可用，因此您可以像在 Blade 樣板中存取任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

若要為郵件新增附件，您可以在郵件的 `attachments` 方法回傳的陣列中加入附件。首先，您可以透過將檔案路徑提供給 `Attachment` 類別提供的 `fromPath` 方法來新增附件：

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

在將檔案附加到訊息時，您也可以使用 `as` 與 `withMime` 方法來指定附件的顯示名稱及／或 MIME 類型：

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

如果您已將檔案儲存在其中一個 [檔案系統磁碟](/docs/{{version}}/filesystem) 上，可以使用 `fromStorage` 附件方法將其附加到郵件中：

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

當然，您也可以指定附件的名稱與 MIME 類型：

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

如果您需要指定預設磁碟以外的儲存磁碟，可以使用 `fromStorageDisk` 方法：

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

`fromData` 附件方法可用於將原始位元組字串作為附件。例如，如果您在記憶體中生成了一個 PDF，並希望在不寫入磁碟的情況下將其附加到郵件中，就可以使用此方法。`fromData` 方法接受一個閉包 (Closure)，該閉包會解析出原始資料位元組以及應指派給該附件的名稱：

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

在郵件中嵌入內嵌圖片通常很麻煩；然而，Laravel 提供了一種便利的方式來將圖片附加到您的郵件。若要嵌入內嵌圖片，請在郵件範本中使用 `$message` 變數上的 `embed` 方法。Laravel 會自動讓 `$message` 變數在所有郵件範本中都可用，因此您不必擔心需要手動傳遞它：

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

如果您已經有一個想要嵌入到郵件範本中的原始圖片資料字串，可以呼叫 `$message` 變數上的 `embedData` 方法。呼叫 `embedData` 方法時，您需要提供應指派給嵌入圖片的檔名：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```


<a name="attachable-objects"></a>
### 可附加物件

雖然透過簡單的字串路徑將檔案附加到訊息通常就足夠了，但在許多情況下，應用程式中的可附加實體是由類別代表的。例如，如果您的應用程式正在將照片附加到訊息中，您的應用程式可能也有一個代表該照片的 `Photo` 模型。在這種情況下，直接將 `Photo` 模型傳遞給 `attach` 方法不是更方便嗎？可附加物件讓您可以做到這一點。

首先，在要附加到訊息的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。此介面規定您的類別必須定義一個 `toMailAttachment` 方法，該方法會回傳一個 `Illuminate\Mail\Attachment` 實例：

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

一旦定義了可附加物件，您就可以在建立郵件訊息時，從 `attachments` 方法回傳該物件的實例：

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

當然，附件資料可能會儲存在遠端檔案儲存服務（如 Amazon S3）上。因此，Laravel 也允許您從儲存在應用程式其中一個 [檔案系統磁碟](/docs/{{version}}/filesystem) 上的資料生成附件實例：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，您也可以透過記憶體中的資料建立附件實例。若要達成此目的，請提供一個閉包給 `fromData` 方法。該閉包應回傳代表附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 還提供了額外的方法讓您自訂附件。例如，您可以使用 `as` 與 `withMime` 方法來自訂檔案名稱與 MIME 類型：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```


<a name="headers"></a>
### 標頭

有時您可能需要為寄出的訊息附加額外的標頭。例如，您可能需要設定自訂的 `Message-Id` 或其他任意的文字標頭。

若要達成此目的，請在您的 mailable 中定義一個 `headers` 方法。`headers` 方法應回傳一個 `Illuminate\Mail\Mailables\Headers` 實例。此類別接受 `messageId`、`references` 與 `text` 參數。當然，您可以僅提供特定訊息所需的參數：

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
### 標籤與元資料

一些第三方郵件提供者（例如 Mailgun 和 Postmark）支援訊息「標籤 (tags)」和「元資料 (metadata)」，這可以用於對應用程式傳送的郵件進行分組和追蹤。您可以透過 `Envelope` 定義將標籤和元資料新增到郵件訊息中：

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

如果您的應用程式使用的是 Mailgun 驅動器，您可以參閱 Mailgun 的文件以獲取更多關於 [標籤 (tags)](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 與 [元資料 (metadata)](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的資訊。同樣地，也可以參閱 Postmark 的文件以獲取更多關於其對 [標籤 (tags)](https://postmarkapp.com/blog/tags-support-for-smtp) 與 [元資料 (metadata)](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 支援的資訊。

如果您的應用程式使用 Amazon SES 傳送郵件，您應該使用 `metadata` 方法將 [SES「標籤 (tags)」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。


<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能是由 Symfony Mailer 驅動的。Laravel 允許您註冊自訂的回呼 (callback)，這些回呼會在傳送訊息之前，與 Symfony Message 實例一起被呼叫。這讓您有機會在訊息傳送前對其進行深度自訂。若要實現此目的，請在您的 `Envelope` 定義中定義一個 `using` 參數：

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

Markdown mailable 訊息讓您可以在 Mailable 中利用 [郵件通知](/docs/{{version}}/notifications#mail-notifications) 預建的範本與元件。由於訊息是以 Markdown 撰寫的，Laravel 能夠為訊息渲染出美觀且回應式 (Responsive) 的 HTML 範本，同時也會自動生成純文字版本。


<a name="generating-markdown-mailables"></a>
### 生成 Markdown Mailable

若要生成帶有對應 Markdown 範本的 Mailable，您可以使用 `make:mail` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

接著，在 Mailable 類別的 `content` 方法中設定 `Content` 定義時，請使用 `markdown` 參數而非 `view` 參數：

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

Markdown Mailable 結合了 Blade 元件與 Markdown 語法，讓您可以輕鬆構建郵件訊息，同時利用 Laravel 預建的郵件 UI 元件：

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
> 撰寫 Markdown 郵件時請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染一個置中的按鈕連結。此元件接受兩個引數：`url` 以及選填的 `color`。支援的顏色有 `primary`、`success` 與 `error`。您可以根據需求在訊息中加入多個按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板元件

面板元件會將指定的文字區塊渲染在一個背景顏色與訊息其餘部分略有不同的面板中。這讓您可以吸引讀者對特定文字區塊的注意：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格元件

表格元件允許您將 Markdown 表格轉換為 HTML 表格。此元件接受 Markdown 表格作為其內容。支援使用預設的 Markdown 表格對齊語法來進行欄位對齊：

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

您可以將所有 Markdown 郵件元件匯出到您自己的應用程式中進行自訂。若要匯出這些元件，請使用 `vendor:publish` Artisan 指令來發佈 `laravel-mail` 資產標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此指令會將 Markdown 郵件元件發佈到 `resources/views/vendor/mail` 目錄中。`mail` 目錄將包含 `html` 和 `text` 目錄，分別包含每個可用元件的對應呈現方式。您可以隨意自訂這些元件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄會包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將會自動轉換為 Markdown 郵件訊息 HTML 版本中的行內 (Inline) CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建立一個全新的主題，可以在 `html/themes` 目錄中放置一個 CSS 檔案。命名並儲存 CSS 檔案後，請更新應用程式 `config/mail.php` 設定檔中的 `theme` 選項，使其與新主題的名稱一致。

若要為個別 Mailable 自訂主題，您可以將 Mailable 類別的 `$theme` 屬性設定為傳送該 Mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 傳送郵件

要傳送訊息，請使用 `Mail` [Facade](/docs/{{version}}/facades) 上的 `to` 方法。`to` 方法接受電子郵件地址、使用者執行個體或使用者集合。如果您傳遞一個物件或物件集合，Mailer 在確定郵件收件者時將自動使用其 `email` 與 `name` 屬性，因此請確保這些屬性在您的物件上是可用的。一旦指定了收件者，您就可以將 Mailable 類別的執行個體傳遞給 `send` 方法：

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

在傳送訊息時，您不限於僅指定 "to" 收件者。您可以透過將各自的方法鏈接在一起來自由設定 "to"、"cc" 與 "bcc" 收件者：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```


<a name="looping-over-recipients"></a>
#### 迭代收件者

有時，您可能需要透過迭代收件者 / 電子郵件地址陣列來將 Mailable 傳送給收件者列表。然而，由於 `to` 方法會將電子郵件地址附加到 Mailable 的收件者列表中，因此每次迴圈迭代都會將另一封電子郵件傳送給之前的每一位收件者。因此，您應該始終為每個收件者重新建立 Mailable 執行個體：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```


<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過指定的 Mailer 傳送郵件

預設情況下，Laravel 會使用應用程式 `mail` 設定檔中設定為 `default` 的 Mailer 來傳送電子郵件。不過，您可以使用 `mailer` 方法來透過特定的 Mailer 設定傳送訊息：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```


<a name="queueing-mail"></a>
### 佇列郵件


<a name="queueing-a-mail-message"></a>
#### 佇列郵件訊息

由於傳送電子郵件訊息可能會對應用程式的回應時間產生負面影響，因此許多開發人員選擇將電子郵件訊息放入佇列進行背景傳送。Laravel 使用其內建的[統一佇列 API](/docs/{{version}}/queues) 讓這件事變得簡單。要將郵件訊息放入佇列，請在指定訊息收件者後，使用 `Mail` Facade 上的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法會自動負責將任務推送到佇列中，以便在背景傳送訊息。在使用此功能之前，您需要先[設定您的佇列](/docs/{{version}}/queues)。


<a name="delayed-message-queueing"></a>
#### 延遲郵件佇列

如果您希望延遲佇列電子郵件訊息的傳遞，可以使用 `later` 方法。`later` 方法的第一個引數接受一個 `DateTime` 執行個體，用以指示訊息應該何時傳送：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->plus(minutes: 10), new OrderShipped($order));
```


<a name="pushing-to-specific-queues"></a>
#### 推送到特定佇列

由於所有使用 `make:mail` 指令生成的 Mailable 類別都使用了 `Illuminate\Bus\Queueable` Trait，因此您可以在任何 Mailable 類別執行個體上呼叫 `onQueue` 與 `onConnection` 方法，讓您能為訊息指定連線與佇列名稱：

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

如果您有希望始終被放入佇列的 Mailable 類別，可以在該類別上實作 `ShouldQueue` Contract。現在，即使您在郵寄時呼叫 `send` 方法，該 Mailable 仍會被放入佇列，因為它實作了該 Contract：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```


<a name="queued-mailables-and-database-transactions"></a>
#### 佇列 Mailable 與資料庫交易

當佇列的 Mailable 在資料庫交易中被分派時，它們可能會在資料庫交易提交之前就被佇列處理。發生這種情況時，您在資料庫交易期間對模型或資料庫記錄所做的任何更新可能尚未反映在資料庫中。此外，在交易內建立的任何模型或資料庫記錄可能尚不存在於資料庫中。如果您的 Mailable 依賴這些模型，則在處理傳送佇列 Mailable 的任務時可能會發生意外錯誤。

若您的佇列連線 `after_commit` 設定選項設為 `false`，您仍然可以透過在傳送郵件訊息時呼叫 `afterCommit` 方法，來指示特定的佇列 Mailable 應在所有開啟的資料庫交易提交後才分派：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，您可以從 Mailable 的建構子中呼叫 `afterCommit` 方法：

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
> 若要深入了解如何解決這些問題，請參閱有關[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="queued-email-failures"></a>
#### 佇列郵件失敗

當佇列電子郵件失敗時，如果佇列 Mailable 類別中定義了 `failed` 方法，則該方法將被叫用。導致佇列電子郵件失敗的 `Throwable` 執行個體將被傳遞給 `failed` 方法：

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

有時您可能希望獲取 Mailable 的 HTML 內容而不傳送它。要實現這一點，您可以呼叫 Mailable 的 `render` 方法。此方法將以字串形式回傳 Mailable 經評估後的 HTML 內容：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```


<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailable

在設計 Mailable 的樣板時，像預覽一般的 Blade 樣板一樣在瀏覽器中快速預覽渲染後的 Mailable 會非常方便。因此，Laravel 允許您直接從路由 Closure 或控制器回傳任何 Mailable。當 Mailable 被回傳時，它將被渲染並顯示在瀏覽器中，讓您可以快速預覽其設計，而無需將其傳送到實際的電子郵件地址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

<a name="localizing-mailables"></a>
## 在地化 Mailable

Laravel 允許您使用請求當前語系以外的語系來傳送 Mailable，甚至在郵件進入佇列時也會記住此語系。

為了達成此目的，`Mail` Facade 提供了 `locale` 方法來設定所需的語言。當 Mailable 的模板正在進行解析時，應用程式將切換到該語系，並在解析完成後還原回先前的語系：

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好語系

有時，應用程式會儲存每個使用者的偏好語系。透過在一個或多個模型中實作 `HasLocalePreference` 契約，您可以指示 Laravel 在傳送郵件時使用此儲存的語系：

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

一旦您實作了該介面，Laravel 在向該模型傳送 Mailable 和通知時，將自動使用其偏好語系。因此，在使用此介面時不需要呼叫 `locale` 方法：

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試


<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了多種方法來檢查 mailable 的結構。此外，Laravel 還提供了多種方便的方法來測試 mailable 是否包含您所預期的內容：

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

正如您所預期的，「HTML」斷言會確認 HTML 版本的 mailable 是否包含指定的字串，而「text」斷言則會確認純文字版本的 mailable 是否包含指定的字串。

<a name="testing-mailable-sending"></a>
### 測試 Mailable 傳送

我們建議將 Mailable 內容的測試，與斷言特定 Mailable 是否「傳送」給特定使用者的測試分開。通常，Mailable 的內容與您正在測試的程式碼無關，只需簡單地斷言 Laravel 已收到傳送特定 Mailable 的指令即可。

您可以使用 `Mail` facade 的 `fake` 方法來防止郵件被寄出。在呼叫 `Mail` facade 的 `fake` 方法後，您就可以斷言 Mailable 已被指示傳送給使用者，甚至檢查 Mailable 接收到的資料：

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

如果您正在將 Mailable 加入佇列以便在背景傳送，您應該使用 `assertQueued` 方法而不是 `assertSent`：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

您也可以使用 `assertOutgoingCount` 方法來斷言已傳送或已加入佇列的 Mailable 總數：

```php
Mail::assertOutgoingCount(3);
```

您可以傳遞一個閉包給 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以便斷言傳送的 Mailable 通過特定的「真實測試 (Truth Test)」。如果至少有一個 Mailable 通過給定的真實測試，則斷言將成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

當呼叫 `Mail` facade 的斷言方法時，提供給閉包的 Mailable 實例會公開一些有用的方法來檢查該 Mailable：

```php
Mail::assertSent(OrderShipped::class, function (OrderShipped $mail) use ($user) {
    return $mail->hasTo($user->email) &&
           $mail->hasCc('...') &&
           $mail->hasBcc('...') &&
           $mail->hasReplyTo('...') &&
           $mail->hasFrom('...') &&
           $mail->hasSubject('...') &&
           $mail->hasMetadata('order_id', $mail->order->id);
           $mail->usesMailer('ses');
});
```

Mailable 實例還包含幾個有用的方法來檢查 Mailable 上的附件：

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

您可能已經注意到有兩個方法可以用來斷言郵件未傳送：`assertNotSent` 和 `assertNotQueued`。有時您可能希望斷言沒有郵件被傳送**或**加入佇列。為了實現這一點，您可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本地開發

當開發一個會傳送電子郵件的應用程式時，你可能不希望真的將郵件傳送到真實的電子郵件地址。Laravel 提供了幾種在本地開發期間「停用」實際傳送郵件的方法。


<a name="log-driver"></a>
#### Log 驅動器

`log` 郵件驅動器不會傳送你的電子郵件，而是將所有郵件訊息寫入你的 Log 檔案中供你檢查。通常，此驅動器僅用於本地開發。關於根據環境設定應用程式的更多資訊，請參閱[設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，你可以使用像是 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 的服務以及 `smtp` 驅動器，將你的郵件訊息傳送到一個「虛擬」信箱，在那裡你可以透過真實的郵件用戶端查看它們。這種方法的好處是讓你可以實際在 Mailtrap 的訊息檢視器中檢查最終的郵件。

如果你正在使用 [Laravel Sail](/docs/{{version}}/sail)，你可以使用 [Mailpit](https://github.com/axllent/mailpit) 預覽你的訊息。當 Sail 運行時，你可以透過 `http://localhost:8025` 存取 Mailpit 介面。


<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 地址

最後，你可以透過呼叫 `Mail` Facade 提供的 `alwaysTo` 方法來指定一個全域的「收件者 (to)」地址。通常，這個方法應該在應用程式的其中一個服務提供者的 `boot` 方法中呼叫：

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

當使用 `alwaysTo` 方法時，郵件訊息上的任何其他「cc」或「bcc」地址都將被移除。


<a name="events"></a>
## 事件

Laravel 在傳送郵件訊息時會分派兩個事件。`MessageSending` 事件在訊息傳送之前分派，而 `MessageSent` 事件在訊息傳送之後分派。請記住，這些事件是在郵件正在「傳送」時分派的，而不是在它進入佇列時。你可以在應用程式中為這些事件建立[事件監聽器](/docs/{{version}}/events)：

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

Laravel 包含了多種郵件傳輸器；然而，你可能希望撰寫自己的傳輸器，以便透過 Laravel 原生不支援的其他服務來遞送電子郵件。首先，定義一個繼承 `Symfony\Component\Mailer\Transport\AbstractTransport` 類別的類別。接著，在你的傳輸器中實作 `doSend` 與 `__toString` 方法：

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

一旦你定義了自訂傳輸器，就可以透過 `Mail` Facade 提供的 `extend` 方法來註冊它。通常，這應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中完成。一個 `$config` 參數將會傳遞給 `extend` 方法提供的閉包。此參數將包含應用程式 `config/mail.php` 設定檔中為該郵件程式定義的設定陣列：

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

一旦你的自訂傳輸器定義並註冊完成，你就可以在應用程式的 `config/mail.php` 設定檔中建立一個使用該新傳輸器的郵件程式定義：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```


<a name="additional-symfony-transports"></a>
### 額外的 Symfony 傳輸器

Laravel 包含了對一些現有的 Symfony 維護郵件傳輸器的支援，例如 Mailgun 與 Postmark。然而，你可能希望擴充 Laravel 以支援額外的 Symfony 維護傳輸器。你可以透過 Composer 安裝必要的 Symfony mailer 並向 Laravel 註冊該傳輸器。例如，你可以安裝並註冊 「Brevo」（原名 「Sendinblue」）Symfony mailer：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

一旦 Brevo mailer 套件安裝完成，你可以在應用程式的 `services` 設定檔中為你的 Brevo API 憑證新增一個項目：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接下來，你可以使用 `Mail` Facade 的 `extend` 方法向 Laravel 註冊該傳輸器。通常，這應該在服務提供者的 `boot` 方法中完成：

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

註冊完傳輸器後，你可以在應用程式的 `config/mail.php` 設定檔中建立一個使用該新傳輸器的郵件程式定義：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```