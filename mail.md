# 郵件

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動器前置準備](#driver-prerequisites)
    - [故障轉移設定](#failover-configuration)
    - [輪詢 (Round Robin) 設定](#round-robin-configuration)
- [產生 Mailables](#generating-mailables)
- [撰寫 Mailables](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [行內附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與詮釋資料 (Metadata)](#tags-and-metadata)
    - [自定義 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailables](#markdown-mailables)
    - [產生 Markdown Mailables](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
    - [自定義元件](#customizing-the-components)
- [傳送郵件](#sending-mail)
    - [將郵件加入佇列](#queueing-mail)
- [渲染 Mailables](#rendering-mailables)
    - [在瀏覽器中預覽 Mailables](#previewing-mailables-in-the-browser)
- [本地化 Mailables](#localizing-mailables)
- [測試](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 傳送](#testing-mailable-sending)
- [郵件與本地開發](#mail-and-local-development)
- [事件](#events)
- [自定義傳送方式](#custom-transports)
    - [額外的 Symfony 傳送方式](#additional-symfony-transports)

<a name="introduction"></a>
## 簡介

傳送電子郵件不一定要很複雜。Laravel 提供了由熱門的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件所支援、乾淨且簡單的電子郵件 API。Laravel 與 Symfony Mailer 提供了多種驅動器，可用於透過 SMTP、Cloudflare、Mailgun、Postmark、Resend、Amazon SES 以及 `sendmail` 傳送郵件，讓你能夠快速地透過所選擇的本地或雲端服務開始傳送郵件。


<a name="configuration"></a>
### 設定

Laravel 的郵件服務可以透過應用程式的 `config/mail.php` 設定檔進行設定。在此檔案中設定的每個 mailer 都可以有其獨特的設定，甚至有其獨特的「傳送方式 (transport)」，這讓你的應用程式可以使用不同的郵件服務來傳送特定的郵件訊息。例如，你的應用程式可以使用 Postmark 來傳送交易型郵件，同時使用 Amazon SES 來傳送大量郵件。

在你的 `mail` 設定檔中，你會發現一個 `mailers` 設定陣列。此陣列包含了 Laravel 支援的每個主要郵件驅動器 / 傳送方式的範例設定項目，而 `default` 設定值則決定了當你的應用程式需要傳送郵件訊息時，預設會使用哪個 mailer。


<a name="driver-prerequisites"></a>
### 驅動器前置準備

基於 API 的驅動器（例如 Mailgun、Postmark 與 Resend）通常比透過 SMTP 伺服器傳送郵件更簡單且快速。只要情況允許，我們建議你使用這些驅動器中的其中之一。


<a name="cloudflare-driver"></a>
#### Cloudflare 驅動器

若要使用 Cloudflare 驅動器，請透過 Composer 安裝 Symfony 的 HTTP Client：

```shell
composer require symfony/http-client
```

接著，你需要對應用程式的 `config/mail.php` 設定檔進行兩項更改。首先，將預設 mailer 設定為 `cloudflare`：

```php
'default' => env('MAIL_MAILER', 'cloudflare'),
```

第二，在 `mailers` 陣列中加入以下設定陣列：

```php
'cloudflare' => [
    'transport' => 'cloudflare',
],
```

設定好應用程式的預設 mailer 後，請將以下選項加入到你的 `config/services.php` 設定檔中：

```php
'cloudflare' => [
    'account_id' => env('CLOUDFLARE_ACCOUNT_ID'),
    'key' => env('CLOUDFLARE_KEY'),
],
```


<a name="mailgun-driver"></a>
#### Mailgun 驅動器

若要使用 Mailgun 驅動器，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳送方式：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接著，你需要對應用程式的 `config/mail.php` 設定檔進行兩項更改。首先，將預設 mailer 設定為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

第二，在 `mailers` 陣列中加入以下設定陣列：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

設定好應用程式的預設 mailer 後，請將以下選項加入到你的 `config/services.php` 設定檔中：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

如果你使用的不是美國的 [Mailgun 區域 (Region)](https://documentation.mailgun.com/docs/mailgun/api-reference/#mailgun-regions)，你可以在 `services` 設定檔中定義該區域的端點 (Endpoint)：

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

若要使用 [Postmark](https://postmarkapp.com/) 驅動器，請透過 Composer 安裝 Symfony 的 Postmark Mailer 傳送方式：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接著，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設為 `postmark`。設定好應用程式的預設 mailer 後，請確保你的 `config/services.php` 設定檔包含以下選項：

```php
'postmark' => [
    'key' => env('POSTMARK_API_KEY'),
],
```

如果你想指定特定 mailer 應使用的 Postmark 訊息串流 (Message Stream)，你可以在 mailer 的設定陣列中加入 `message_stream_id` 設定選項。此設定陣列可以在應用程式的 `config/mail.php` 設定檔中找到：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

透過這種方式，你也能夠設定多個具有不同訊息串流的 Postmark mailer。


<a name="resend-driver"></a>
#### Resend 驅動器

若要使用 [Resend](https://resend.com/) 驅動器，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接著，將應用程式 `config/mail.php` 設定檔中的 `default` 選項設為 `resend`。設定好應用程式的預設 mailer 後，請確保你的 `config/services.php` 設定檔包含以下選項：

```php
'resend' => [
    'key' => env('RESEND_API_KEY'),
],
```


<a name="ses-driver"></a>
#### SES 驅動器

若要使用 Amazon SES 驅動器，你必須先安裝適用於 PHP 的 Amazon AWS SDK。你可以透過 Composer 套件管理器安裝此函式庫：

```shell
composer require aws/aws-sdk-php
```

接著，將 `config/mail.php` 設定檔中的 `default` 選項設為 `ses`，並確認你的 `config/services.php` 設定檔包含以下選項：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

若要透過工作階段令牌使用 AWS [暫時性憑證 (Temporary Credentials)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，你可以在應用程式的 SES 設定中加入 `token` 鍵：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

若要與 SES 的 [訂閱管理功能 (Subscription Management Features)](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html) 互動，你可以在郵件訊息的 [headers](#headers) 方法回傳的陣列中，回傳 `X-Ses-List-Management-Options` 標頭：

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

如果你想定義 Laravel 在傳送郵件時應傳遞給 AWS SDK `SendEmail` 方法的 [額外選項 (Additional Options)](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，你可以在 `ses` 設定中定義一個 `options` 陣列：

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

有時，你所設定用來傳送應用程式郵件的外部服務可能會發生故障。在這種情況下，定義一個或多個備援的郵件傳送設定會很有幫助，當主要傳送驅動器故障時，系統就會自動使用這些備援設定。

要達成此目的，你應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳送方式的郵件程式 (Mailer)。你的 `failover` 郵件程式設定陣列中應包含一個 `mailers` 陣列，用來參照已設定郵件程式的傳送順序：

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

設定好使用 `failover` 傳送方式的郵件程式後，你需要將應用程式 `.env` 檔案中的預設郵件程式設定為該故障轉移郵件程式，才能開始使用故障轉移功能：

```ini
MAIL_MAILER=failover
```

<a name="round-robin-configuration"></a>
### 輪詢 (Round Robin) 設定

`roundrobin` 傳送方式讓你可以將郵件傳送的工作負載分散到多個郵件程式。首先，在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` 傳送方式的郵件程式。你的 `roundrobin` 郵件程式設定陣列中應包含一個 `mailers` 陣列，用來參照哪些已設定的郵件程式應被用於傳送：

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

定義好輪詢郵件程式後，你應該在應用程式的 `mail` 設定檔中，將其名稱指定為 `default` 設定鍵的值，使其成為應用程式預設使用的郵件程式：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

輪詢傳送方式會從已設定的郵件程式列表中隨機選擇一個，然後在接下來的每封電子郵件中切換到下一個可用的郵件程式。與協助達成 *[高可用性 (High availability)](https://en.wikipedia.org/wiki/High_availability)* 的 `failover` 傳送方式相比，`roundrobin` 傳送方式提供的是 *[負載平衡 (Load balancing)](https://en.wikipedia.org/wiki/Load_balancing_(computing))*。

<a name="generating-mailables"></a>
## 產生 Mailables

在建構 Laravel 應用程式時，應用程式傳送的每一種郵件類型都由一個 「mailable」 類別來表示。這些類別儲存在 `app/Mail` 目錄中。如果您在應用程式中沒有看到此目錄也不必擔心，因為當您使用 `make:mail` Artisan 指令建立第一個 mailable 類別時，系統會自動為您產生：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailables

一旦你產生了 mailable 類別，請打開它，讓我們來探索其中的內容。Mailable 類別的設定是在幾個方法中完成的，包含 `envelope`、`content` 和 `attachments` 方法。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，用來定義主旨，有時也包含郵件的收件者。`content` 方法則回傳一個 `Illuminate\Mail\Mailables\Content` 物件，用來定義將用於產生郵件內容的 [Blade 範本](/docs/{{version}}/blade)。


<a name="configuring-the-sender"></a>
### 設定寄件者


<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，讓我們探索如何設定郵件的寄件者。或者換句話說，這封郵件是由「誰」寄出的。有兩種方式可以設定寄件者。第一種是在郵件的 envelope 中指定「from」位址：

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

如果你願意，你也可以指定一個 `replyTo` 位址：

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

然而，如果你的應用程式在所有郵件中都使用相同的「from」位址，那麼在每個產生的 mailable 類別中都添加它會變得很繁瑣。相反地，你可以在 `config/mail.php` 設定檔中指定一個全域的「from」位址。如果在 mailable 類別中沒有指定其他的「from」位址，則會使用此位址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，你也可以在 `config/mail.php` 設定檔中定義一個全域的「reply_to」位址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```


<a name="configuring-the-view"></a>
### 設定視圖

在 mailable 類別的 `content` 方法中，你可以定義 `view`，也就是渲染郵件內容時應使用的範本。由於每封郵件通常使用 [Blade 範本](/docs/{{version}}/blade) 來渲染內容，因此在建構郵件的 HTML 時，你可以使用 Blade 範本引擎完整且便利的功能：

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
> 你可能想要建立一個 `resources/views/mail` 目錄來存放所有的郵件範本；不過，你也可以自由地將它們放在 `resources/views` 目錄下的任何地方。


<a name="plain-text-emails"></a>
#### 純文字郵件

如果你想定義郵件的純文字版本，可以在建立郵件的 `Content` 定義時指定純文字範本。就像 `view` 參數一樣，`text` 參數應該是一個用於渲染郵件內容的範本名稱。你可以自由地為郵件定義 HTML 和純文字版本：

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

為了清晰起見，`html` 參數可以作為 `view` 參數的別名使用：

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

通常情況下，你會想要將一些資料傳遞到視圖中，以便在渲染郵件的 HTML 時使用。有兩種方式可以讓資料在視圖中使用。第一種，在 mailable 類別上定義的任何公開屬性都會自動在視圖中可用。因此，舉例來說，你可以將資料傳遞到 mailable 類別的建構子中，並將該資料設定為類別中定義的公開屬性：

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

一旦資料被設定為公開屬性，它就會自動在你的視圖中可用，因此你可以像存取 Blade 範本中的任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $order->price }}
</div>
```


<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果你想在將郵件資料發送到範本之前自定義其格式，可以透過 `Content` 定義的 `with` 參數手動將資料傳遞給視圖。通常你仍然會透過 mailable 類別的建構子傳遞資料；但你應該將此資料設定為 `protected` 或 `private` 屬性，這樣資料就不會自動在範本中可用：

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

一旦資料透過 `with` 參數傳遞，它就會自動在你的視圖中可用，因此你可以像存取 Blade 範本中的任何其他資料一樣存取它：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

若要為電子郵件加入附件，你可以將附件加入到訊息中 `attachments` 方法所回傳的陣列裡。首先，你可以透過 `Attachment` 類別提供的 `fromPath` 方法並提供檔案路徑來新增附件：

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

將檔案附加到訊息時，你還可以使用 `as` 與 `withMime` 方法為附件指定顯示名稱及 / 或 MIME 類型：

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

如果你已將檔案儲存在其中一個[檔案系統磁碟](/docs/{{version}}/filesystem)上，你可以使用 `fromStorage` 附件方法將其附加到電子郵件中：

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

當然，你也可以指定附件的名稱與 MIME 類型：

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

如果你需要指定預設磁碟以外的儲存磁碟，可以使用 `fromStorageDisk` 方法：

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

`fromData` 附件方法可用於將原始位元組字串作為附件。例如，如果你在記憶體中產生了 PDF，並希望在不將其寫入磁碟的情況下將其附加到電子郵件中，則可以使用此方法。`fromData` 方法接受一個用來解析原始資料位元組的閉包，以及應分配給附件的名稱：

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

在電子郵件中嵌入行內圖片通常很麻煩；不過，Laravel 提供了一種方便的方法來將圖片附加到電子郵件中。若要嵌入行內圖片，請在電子郵件範本中的 `$message` 變數上使用 `embed` 方法。Laravel 會自動讓 `$message` 變數在你的所有電子郵件範本中都可用，因此你不需要擔心手動傳遞它：

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

如果你已經有一個想要嵌入到電子郵件範本中的原始圖片資料字串，你可以在 `$message` 變數上呼叫 `embedData` 方法。呼叫 `embedData` 方法時，你需要提供應分配給嵌入圖片的檔名：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```

<a name="attachable-objects"></a>
### 可附加物件

雖然透過簡單的字串路徑將檔案附加到訊息中通常就足夠了，但在許多情況下，你的應用程式中的可附加實體是由類別代表的。例如，如果你的應用程式正將照片附加到訊息中，你的應用程式可能也有一個代表該照片的 `Photo` 模型。在這種情況下，直接將 `Photo` 模型傳遞給 `attach` 方法不是更方便嗎？可附加物件讓你能夠輕鬆達成這個目的。

首先，在要附加到訊息的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。此介面要求你的類別定義一個回傳 `Illuminate\Mail\Attachment` 實例的 `toMailAttachment` 方法：

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

定義好可附加物件後，你在建立電子郵件訊息時，可以從 `attachments` 方法回傳該物件的實例：

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

當然，附件資料可能儲存在遠端檔案儲存服務（如 Amazon S3）上。因此，Laravel 也允許你從儲存在應用程式其中一個[檔案系統磁碟](/docs/{{version}}/filesystem)上的資料來產生附件實例：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，你也可以透過記憶體中的資料建立附件實例。若要達成此目的，請提供一個閉包給 `fromData` 方法。該閉包應回傳代表附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 還提供了額外的方法供你自定義附件。例如，你可以使用 `as` 與 `withMime` 方法來自定義檔案的名稱與 MIME 類型：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```

<a name="headers"></a>
### 標頭

有時你可能需要為外寄訊息附加額外的標頭。例如，你可能需要設定自定義的 `Message-Id` 或其他任意文字標頭。

若要達成此目的，請在你的 mailable 中定義一個 `headers` 方法。`headers` 方法應回傳一個 `Illuminate\Mail\Mailables\Headers` 實例。此類別接受 `messageId`、`references` 與 `text` 參數。當然，你可以只提供特定訊息所需的參數：

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
### 標籤與詮釋資料 (Metadata)

一些第三方郵件供應商（例如 Mailgun 和 Postmark）支援郵件「標籤 (tags)」和「詮釋資料 (metadata)」，可用於對應用程式傳送的郵件進行分組和追蹤。您可以透過 `Envelope` 定義將標籤和詮釋資料加入到郵件訊息中：

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

如果您的應用程式使用 Mailgun 驅動器，您可以參考 Mailgun 的文件以取得更多關於 [標籤 (tags)](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags) 和 [詮釋資料 (metadata)](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages) 的資訊。同樣地，也可以參考 Postmark 的文件以取得更多關於其支援 [標籤 (tags)](https://postmarkapp.com/blog/tags-support-for-smtp) 和 [詮釋資料 (metadata)](https://postmarkapp.com/support/article/1125-custom-metadata-faq) 的資訊。

如果您的應用程式使用 Amazon SES 傳送郵件，您應該使用 `metadata` 方法將 [SES 「標籤 (tags)」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html) 附加到訊息中。


<a name="customizing-the-symfony-message"></a>
### 自定義 Symfony 訊息

Laravel 的郵件功能是由 Symfony Mailer 所驅動的。Laravel 允許您註冊自定義的回呼函式 (Callbacks)，這些回呼函式會在傳送訊息之前以 Symfony Message 實例作為引數被調用。這讓您有機會在訊息傳送前對其進行深度自定義。若要達成此目的，請在您的 `Envelope` 定義中定義一個 `using` 參數：

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

Markdown mailable 訊息讓您可以在 mailable 中利用 [郵件通知](/docs/{{version}}/notifications#mail-notifications) 內建的模板與元件。由於訊息是使用 Markdown 撰寫的，Laravel 能夠為訊息渲染出美觀且具備響應式設計的 HTML 模板，同時也會自動產生純文字版本。

<a name="generating-markdown-mailables"></a>
### 產生 Markdown Mailables

要產生一個帶有對應 Markdown 模板的 mailable，您可以使用 `make:mail` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

接著，在其 `content` 方法中設定 mailable 的 `Content` 定義時，請使用 `markdown` 參數而非 `view` 參數：

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

Markdown mailables 結合了 Blade 元件與 Markdown 語法，讓您可以輕鬆建構郵件訊息，同時利用 Laravel 預設的郵件 UI 元件：

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
> 在撰寫 Markdown 郵件時，請勿使用過多的縮排。根據 Markdown 標準，Markdown 解析器會將縮排內容渲染為程式碼區塊。

<a name="button-component"></a>
#### 按鈕元件 (Button Component)

按鈕元件會渲染一個置中的按鈕連結。此元件接受兩個引數：`url` 以及選填的 `color`。支援的顏色有 `primary`、`success` 與 `error`。您可以根據需求在訊息中加入任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```

<a name="panel-component"></a>
#### 面板元件 (Panel Component)

面板元件會將指定的文字區塊渲染在一個背景顏色與訊息其餘部分略有不同的面板中。這可以讓您吸引讀者注意特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```

<a name="table-component"></a>
#### 表格元件 (Table Component)

表格元件可以讓您將 Markdown 表格轉換為 HTML 表格。此元件接受 Markdown 表格作為其內容。支援使用預設的 Markdown 表格對齊語法來調整欄位對齊：

```blade
<x-mail::table>
| Laravel       | Table         | Example       |
| ------------- | :-----------: | ------------: |
| Col 2 is      | Centered      | $10           |
| Col 3 is      | Right-Aligned | $20           |
</x-mail::table>
```

<a name="customizing-the-components"></a>
### 自定義元件

您可以將所有的 Markdown 郵件元件匯出到您自己的應用程式中進行自定義。若要匯出元件，請使用 `vendor:publish` Artisan 指令來發布 `laravel-mail` 資源標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

此指令會將 Markdown 郵件元件發布至 `resources/views/vendor/mail` 目錄。`mail` 目錄會包含 `html` 與 `text` 目錄，各別存放每個可用元件的對應呈現方式。您可以隨意自定義這些元件。

<a name="customizing-the-css"></a>
#### 自定義 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄會包含一個 `default.css` 檔案。您可以自定義此檔案中的 CSS，您的樣式將會自動轉換為 Markdown 郵件訊息 HTML 版本中的行內 (Inline) CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建立一個全新的主題，可以在 `html/themes` 目錄中放置一個 CSS 檔案。命名並儲存 CSS 檔案後，請更新應用程式 `config/mail.php` 設定檔中的 `theme` 選項，使其與新主題的名稱一致。

若要為個別的 mailable 自定義主題，您可以將該 mailable 類別的 `$theme` 屬性設定為傳送該 mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 傳送郵件

若要傳送郵件，請使用 `Mail` [Facade](/docs/{{version}}/facades) 的 `to` 方法。`to` 方法接受電子郵件地址、使用者實例或使用者集合。如果你傳入一個物件或物件集合，mailer 將會自動在決定郵件收件者時使用它們的 `email` 與 `name` 屬性，因此請確保你的物件具有這些屬性。指定收件者後，你可以將 mailable 類別的實例傳遞給 `send` 方法：

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

傳送訊息時，你不僅限於指定「收件者 (to)」。你可以透過鏈結各自的方法來自由設定「收件者 (to)」、「副本 (cc)」與「密件副本 (bcc)」：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```


<a name="looping-over-recipients"></a>
#### 在收件者清單中進行迴圈

有時候，你可能需要透過對收件者/電子郵件地址陣列進行迭代，將 mailable 傳送給收件者列表。然而，由於 `to` 方法會將電子郵件地址附加到 mailable 的收件者列表中，因此迴圈的每一次迭代都會將另一封郵件傳送給之前的每一位收件者。因此，你應該總是為每一位收件者重新建立 mailable 實例：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```


<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過特定的 Mailer 傳送郵件

預設情況下，Laravel 會使用你的應用程式 `mail` 設定檔中設定為 `default` 的 mailer 來傳送電子郵件。不過，你可以使用 `mailer` 方法來使用特定的 mailer 設定傳送郵件：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```


<a name="queueing-mail"></a>
### 將郵件加入佇列


<a name="queueing-a-mail-message"></a>
#### 將郵件訊息加入佇列

由於傳送電子郵件訊息可能會對應用程式的響應時間產生負面影響，許多開發者選擇將電子郵件訊息加入佇列以在背景傳送。Laravel 使用其內建的[統一的佇列 API](/docs/{{version}}/queues) 讓這件事變得簡單。若要將郵件訊息加入佇列，請在指定郵件收件者後，使用 `Mail` Facade 的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法會自動負責將一個工作 (Job) 推送到佇列，以便在背景傳送訊息。在使用此功能之前，你需要[設定你的佇列](/docs/{{version}}/queues)。


<a name="delayed-message-queueing"></a>
#### 延遲郵件加入佇列

如果你希望延遲傳送已加入佇列的電子郵件，可以使用 `later` 方法。`later` 方法的第一個引數接受一個 `DateTime` 實例，用以指示何時傳送訊息：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->plus(minutes: 10), new OrderShipped($order));
```


<a name="pushing-to-specific-queues"></a>
#### 推送到特定的佇列

由於所有使用 `make:mail` 指令產生的 mailable 類別都使用了 `Illuminate\Bus\Queueable` Trait，因此你可以在任何 mailable 類別實例上呼叫 `onQueue` 與 `onConnection` 方法，讓你為該訊息指定連線與佇列名稱：

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
#### 預設加入佇列

如果你有想要總是加入佇列的 mailable 類別，可以在該類別上實作 `ShouldQueue` 契約(Contracts)。現在，即使你在寄信時呼叫的是 `send` 方法，該 mailable 仍然會被加入佇列，因為它實作了該契約：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```


<a name="queued-mailables-and-database-transactions"></a>
#### 已加入佇列的 Mailables 與資料庫交易

當在資料庫交易中分派已加入佇列的 mailable 時，它們可能會在資料庫交易提交之前就被佇列處理。發生這種情況時，你在資料庫交易期間對模型或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易中建立的任何模型或資料庫紀錄可能還不存在於資料庫中。如果你的 mailable 依賴這些模型，則在處理傳送已加入佇列 mailable 的工作時，可能會發生非預期的錯誤。

如果你的佇列連線之 `after_commit` 設定選項設為 `false`，你仍然可以透過在傳送郵件訊息時呼叫 `afterCommit` 方法，來指示特定的已加入佇列 mailable 應該在所有開啟的資料庫交易都提交後才被分派：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，你也可以從 mailable 的建構子中呼叫 `afterCommit` 方法：

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
> 若要進一步了解如何解決這些問題，請參閱有關[已加入佇列的工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="queued-email-failures"></a>
#### 已加入佇列的郵件失敗處理

當已加入佇列的郵件傳送失敗時，如果已加入佇列的 mailable 類別中定義了 `failed` 方法，則該方法將被叫用。導致郵件傳送失敗的 `Throwable` 實例將被傳遞給 `failed` 方法：

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

有時你可能希望在不傳送郵件的情況下取得 mailable 的 HTML 內容。若要達成此目的，你可以呼叫 mailable 的 `render` 方法。此方法會將渲染後的 mailable HTML 內容以字串形式回傳：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```


<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailables

在設計 mailable 的模板時，像預覽一般的 Blade 模板一樣，在瀏覽器中快速預覽渲染後的 mailable 是很方便的。因此，Laravel 允許你直接從路由閉包或控制器回傳任何 mailable。當回傳 mailable 時，它將被渲染並顯示在瀏覽器中，讓你可以快速預覽其設計，而無需將其傳送到實際的電子郵件地址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

<a name="localizing-mailables"></a>
## 本地化 Mailables

Laravel 允許您使用目前請求語系以外的語系來傳送 Mailables，甚至在郵件加入佇列時也會記住此語系。

為此，`Mail` facade 提供了一個 `locale` 方法來設定所需的語言。當 mailable 的樣板正在進行解析評估時，應用程式會切換到該語系，並在解析完成後恢復到先前的語系：

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```

<a name="user-preferred-locales"></a>
#### 使用者偏好的語系

有時，應用程式會儲存每個使用者偏好的語系。藉由在您的一個或多個 Model 中實作 `HasLocalePreference` 契約(Contracts)，您可以指示 Laravel 在傳送郵件時使用此儲存的語系：

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

一旦您實作了該介面，Laravel 在向該 Model 傳送 Mailables 和通知時將自動使用偏好的語系。因此，在使用此介面時不需要呼叫 `locale` 方法：

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試

<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了多種方法來檢查 Mailable 的結構。此外，Laravel 還提供了幾個方便的方法，用於測試 Mailable 是否包含您預期的內容：

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

如您所料，「HTML」斷言用於驗證 Mailable 的 HTML 版本是否包含指定的字串，而「text」斷言則用於驗證 Mailable 的純文字版本是否包含指定的字串。

<a name="testing-mailable-sending"></a>
### 測試 Mailable 傳送

我們建議將測試 Mailable 內容與斷言特定 Mailable 是否已「傳送」給特定使用者的測試分開。通常，Mailable 的內容與你正在測試的程式碼無關，只需要簡單地斷言 Laravel 已收到傳送特定 Mailable 的指令即可。

你可以使用 `Mail` Facade 的 `fake` 方法來防止郵件被實際傳送。呼叫 `Mail` Facade 的 `fake` 方法後，你就可以斷言 Mailable 已被要求傳送給使用者，甚至可以檢查 Mailable 接收到的資料：

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

如果你是將 Mailable 加入佇列以在背景傳送，則應使用 `assertQueued` 方法而不是 `assertSent`：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

你也可以使用 `assertOutgoingCount` 方法來斷言已傳送或已加入佇列的 Mailable 總數：

```php
Mail::assertOutgoingCount(3);
```

你可以將閉包傳遞給 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以斷言傳送的 Mailable 通過了特定的「真值測試 (truth test)」。如果至少有一個 Mailable 通過了給定的真值測試，則斷言成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

呼叫 `Mail` Facade 的斷言方法時，所提供的閉包所接收的 Mailable 執行個體會公開一些實用的方法，讓你檢查該 Mailable：

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

Mailable 執行個體還包含數個實用方法，用於檢查 Mailable 上的附件：

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

你可能已經注意到，有兩種方法可以用來斷言郵件未被傳送：`assertNotSent` 和 `assertNotQueued`。有時你可能想要斷言沒有任何郵件被傳送**或**加入佇列。為此，你可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本地開發

當開發會傳送電子郵件的應用程式時，你可能不希望真的將郵件寄到真實的電子郵件位址。Laravel 提供了幾種在本地開發期間「停用」實際傳送電子郵件的方法。


<a name="log-driver"></a>
#### Log 驅動器

`log` 郵件驅動器不會傳送你的電子郵件，而是會將所有郵件訊息寫入日誌檔中供你檢查。通常，此驅動器僅用於本地開發。有關根據環境設定應用程式的更多資訊，請參閱 [設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，你可以使用像 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 這樣的服務，並透過 `smtp` 驅動器將郵件訊息傳送到一個「虛擬」信箱，你可以在真實的郵件用戶端中查看它們。這種方法的優點是讓你可以實際在 Mailtrap 的訊息檢視器中檢查最終的郵件內容。

如果你正在使用 [Laravel Sail](/docs/{{version}}/sail)，你可以使用 [Mailpit](https://github.com/axllent/mailpit) 預覽你的訊息。當 Sail 運行時，你可以透過 `http://localhost:8025` 存取 Mailpit 介面。


<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 位址

最後，你可以透過呼叫 `Mail` Facade 提供的 `alwaysTo` 方法來指定一個全域的「收件者 (to)」位址。通常，應在應用程式其中一個服務提供者(Service Providers) 的 `boot` 方法中呼叫此方法：

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

使用 `alwaysTo` 方法時，郵件訊息中的任何其他「cc」或「bcc」位址都將被移除。


<a name="events"></a>
## 事件

Laravel 在傳送郵件訊息時會發送兩個事件。`MessageSending` 事件在訊息傳送前發送，而 `MessageSent` 事件在訊息傳送後發送。請記住，這些事件是在郵件「被傳送」時發送的，而不是在它進入佇列時。你可以在應用程式中為這些事件建立 [事件監聽器](/docs/{{version}}/events)：

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
## 自定義傳送方式

Laravel 包含多種郵件傳送方式；然而，你可能希望編寫自己的傳送方式，以便透過 Laravel 未內建支援的其他服務來遞送電子郵件。首先，定義一個繼承 `Symfony\Component\Mailer\Transport\AbstractTransport` 的類別。接著，在你的傳送方式中實作 `doSend` 與 `__toString` 方法：

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

定義好自定義傳送方式後，你可以透過 `Mail` Facade 提供的 `extend` 方法來註冊它。通常，這應該在應用程式 `AppServiceProvider` 的 `boot` 方法中完成。傳遞給 `extend` 方法的閉包會接收到一個 `$config` 引數。此引數將包含應用程式 `config/mail.php` 設定檔中為該郵件程式定義的設定陣列：

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

一旦定義並註冊了自定義傳送方式，你就可以在應用程式的 `config/mail.php` 設定檔中建立一個使用該新傳送方式的郵件程式定義：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```


<a name="additional-symfony-transports"></a>
### 額外的 Symfony 傳送方式

Laravel 包含對一些現有由 Symfony 維護的郵件傳送方式的支援，例如 Mailgun 和 Postmark。但是，你可能希望擴充 Laravel 以支援其他 Symfony 維護的傳送方式。你可以透過 Composer 安裝必要的 Symfony mailer 並將傳送方式註冊到 Laravel 中來實現。例如，你可以安裝並註冊「Brevo」(前身為「Sendinblue」) 的 Symfony mailer：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

安裝 Brevo mailer 套件後，你可以在應用程式的 `services` 設定檔中新增一條 Brevo API 憑證的項目：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接下來，你可以使用 `Mail` Facade 的 `extend` 方法將傳送方式註冊到 Laravel。通常，這應該在服務提供者(Service Providers) 的 `boot` 方法中完成：

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

註冊完傳送方式後，你就可以在應用程式的 `config/mail.php` 設定檔中建立一個使用該新傳送方式的郵件程式定義：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```