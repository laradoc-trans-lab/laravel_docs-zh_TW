# 郵件

- [介紹](#introduction)
    - [設定](#configuration)
    - [Driver 前置需求](#driver-prerequisites)
    - [容錯移轉設定](#failover-configuration)
    - [循環輪替設定](#round-robin-configuration)
- [產生 Mailable](#generating-mailables)
- [撰寫 Mailable](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定 View](#configuring-the-view)
    - [View 資料](#view-data)
    - [附件](#attachments)
    - [行內附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與中繼資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailable](#markdown-mailables)
    - [產生 Markdown Mailable](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
    - [自訂元件](#customizing-the-components)
- [寄送郵件](#sending-mail)
    - [佇列郵件](#queueing-mail)
- [渲染 Mailable](#rendering-mailables)
    - [在瀏覽器中預覽 Mailable](#previewing-mailables-in-the-browser)
- [本地化 Mailable](#localizing-mailables)
- [測試](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 寄送](#testing-mailable-sending)
- [郵件與本機開發](#mail-and-local-development)
- [事件](#events)
- [自訂 Transport](#custom-transports)
    - [其他 Symfony Transport](#additional-symfony-transports)

<a name="introduction"></a>
## 介紹

寄送郵件不一定要很複雜。Laravel 提供了一套簡潔、簡單的郵件 API，其底層由熱門的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件驅動。Laravel 與 Symfony Mailer 提供了多種 Driver，可透過 SMTP、Mailgun、Postmark、Resend、Amazon SES、以及 `sendmail` 來寄送郵件，讓你能快速地開始透過所選的本機或雲端服務來寄送郵件。


<a name="configuration"></a>
### 設定

Laravel 的郵件服務可透過專案的 `config/mail.php` 設定檔來設定。在這個檔案中設定的每個 Mailer 都可以有自己獨特的設定，甚至可以有自己獨特的「transport」，讓你的應用程式能為不同的郵件訊息使用不同的郵件服務。舉例來說，你的應用程式可能會用 Postmark 來寄送交易相關郵件，並用 Amazon SES 來寄送大量郵件。

在 `mail` 設定檔中，可以看到一個 `mailers` 設定陣列。這個陣列為 Laravel 支援的每個主要郵件 Driver / transport 都提供了一個範例設定項目。此外，`default` 設定值則是用來決定當應用程式需要寄送郵件訊息時，預設要使用哪個 Mailer。


<a name="driver-prerequisites"></a>
### Driver 前置需求

比起透過 SMTP 伺服器寄送郵件，使用像 Mailgun、Postmark、Resend、MailerSend 等基於 API 的 Driver 通常會更簡單也更快速。只要情況允許，我們都建議使用其中一種 Driver。


<a name="mailgun-driver"></a>
#### Mailgun Driver

若要使用 Mailgun Driver，請先透過 Composer 來安裝 Symfony 的 Mailgun Mailer transport：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接著，需要在專案的 `config/mail.php` 設定檔中進行兩項變更。首先，將預設 Mailer 設為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

第二，將下列設定陣列加到 `mailers` 陣列中：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

設定好應用程式的預設 Mailer 後，請將下列選項加到 `config/services.php` 設定檔中：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

若未使用美國的 [Mailgun 地區](https://documentation.mailgun.com/docs/mailgun/api-reference/#mailgun-regions)，則可在 `services` 設定檔中定義該地區的 Endpoint：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.eu.mailgun.net'),
    'scheme' => 'https',
],
```


<a name="postmark-driver"></a>
#### Postmark Driver

若要使用 [Postmark](https://postmarkapp.com/) Driver，請先透過 Composer 來安裝 Symfony 的 Postmark Mailer transport：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接著，將專案 `config/mail.php` 設定檔中的 `default` 選項設為 `postmark`。設定好專案的預設 Mailer 後，請確保 `config/services.php` 設定檔中包含下列選項：

```php
'postmark' => [
    'token' => env('POSTMARK_TOKEN'),
],
```

若想為給定的 Mailer 指定要使用的 Postmark Message Stream，可將 `message_stream_id` 設定選項加到該 Mailer 的設定陣列中。這個設定陣列可以在專案的 `config/mail.php` 設定檔中找到：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

這樣一來，就可以設定多個使用不同 Message Stream 的 Postmark Mailer。


<a name="resend-driver"></a>
#### Resend Driver

若要使用 [Resend](https://resend.com/) Driver，請先透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接著，請將專案 `config/mail.php` 設定檔中的 `default` 選項設為 `resend`。設定好專案的預設 Mailer 後，請確保 `config/services.php` 設定檔中包含下列選項：

```php
'resend' => [
    'key' => env('RESEND_KEY'),
],
```


<a name="ses-driver"></a>
#### SES Driver

若要使用 Amazon SES Driver，必須先安裝適用於 PHP 的 Amazon AWS SDK。可透過 Composer 套件管理員來安裝此函式庫：

```shell
composer require aws/aws-sdk-php
```

接著，將 `config/mail.php` 設定檔中的 `default` 選項設為 `ses`，並確認 `config/services.php` 設定檔中包含下列選項：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

若要透過 Session Token 來使用 AWS [暫時憑證](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，可在專案的 SES 設定中加上 `token` 機碼：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

若要與 SES 的[訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html)互動，可在郵件訊息的 [headers](#headers) 方法所回傳的陣列中回傳 `X-Ses-List-Management-Options` 標頭：

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

若想定義讓 Laravel 在寄送郵件時要傳給 AWS SDK `SendEmail` 方法的[其他選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，可在 `ses` 設定中定義一個 `options` 陣列：

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
#### MailerSend Driver

[MailerSend](https://www.mailersend.com/) 是一個交易型郵件與 SMS 服務，該服務為 Laravel 維護了自己基於 API 的郵件 Driver。包含該 Driver 的套件可透過 Composer 套件管理員來安裝：

```shell
composer require mailersend/laravel-driver
```

安裝好套件後，請將 `MAILERSEND_API_KEY` 環境變數加到專案的 `.env` 檔中。此外，也應將 `MAIL_MAILER` 環境變數定義為 `mailersend`：

```ini
MAIL_MAILER=mailersend
MAIL_FROM_ADDRESS=app@yourdomain.com
MAIL_FROM_NAME="App Name"

MAILERSEND_API_KEY=your-api-key
```

最後，在專案的 `config/mail.php` 設定檔中，將 MailerSend 加到 `mailers` 陣列裡：

```php
'mailersend' => [
    'transport' => 'mailersend',
],
```

想瞭解更多有關 MailerSend 的資訊 (包含如何使用 Hosted Template)，請參考 [MailerSend Driver 文件](https://github.com/mailersend/mailersend-laravel-driver#usage)。

<a name="failover-configuration"></a>
### 容錯移轉設定

有時候，用來寄送應用程式郵件的外部服務可能會停機。在這種情況下，若能定義一或多個備用郵件傳送設定，以便在主要傳送 Driver 停機時使用，將會很有用。

為了達成這個目的，請在應用程式的 `mail` 設定檔中定義一個使用 `failover` transport 的 mailer。應用程式 `failover` mailer 的設定陣列應包含一個 `mailers` 陣列，該陣列參照了要用來傳送郵件的 mailer 設定順序：

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

一旦定義了容錯移轉 mailer，就應在應用程式的 `mail` 設定檔中，將其名稱指定為 `default` 設定鍵的值，以將其設為應用程式使用的預設 mailer：

```php
'default' => env('MAIL_MAILER', 'failover'),
```


<a name="round-robin-configuration"></a>
### 循環輪替設定

`roundrobin` transport 可讓您將郵件工作負載分散到多個 mailer。首先，在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` transport 的 mailer。應用程式 `roundrobin` mailer 的設定陣列應包含一個 `mailers` 陣列，該陣列參照了要用來傳送郵件的 mailer 設定：

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

一旦定義了循環輪替 mailer，就應在應用程式的 `mail` 設定檔中，將其名稱指定為 `default` 設定鍵的值，以將其設為應用程式使用的預設 mailer：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

循環輪替 transport 會從設定的 mailer 列表中隨機選擇一個 mailer，然後對於後續的每封郵件，切換到下一個可用的 mailer。與有助於實現 *[高可用性](https://en.wikipedia.org/wiki/High_availability)* 的 `failover` transport 不同，`roundrobin` transport 提供了 *[負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))*.

<a name="generating-mailables"></a>
## 產生 Mailable

在建立 Laravel 應用程式時，應用程式所寄送的各種類型的郵件，都會以一個「Mailable」類別來表示。這些類別儲存在 `app/Mail` 目錄中。若在應用程式中找不到這個目錄也別擔心，因為當我們使用 `make:mail` 這個 Artisan 指令來建立第一個 Mailable 類別時，這個目錄就會被產生出來：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailable

產生 Mailable 類別後，打開該檔案來看看其內容。Mailable 類別的設定是透過好幾個方法來完成的，包含 `envelope`、`content`、以及 `attachments` 方法。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，用來定義主旨，有時候也會用來定義訊息的收件者。`content` 方法則會回傳一個 `Illuminate\Mail\Mailables\Content` 物件，用來定義要用來產生訊息內容的 [Blade 樣板](/docs/{{version}}/blade)。

<a name="configuring-the-sender"></a>
### 設定寄件者

<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，我們先來看看如何設定郵件的寄件者。或者，換句話說，這封郵件要由誰「寄出」。有兩種方式可以設定寄件者。第一種，可以在訊息的 Envelope 上指定「from」位址：

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

如果想的話，也可以指定一個 `replyTo` 位址：

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

不過，若應用程式中所有的郵件都使用同一個「from」位址，則要在每個產生的 Mailable 類別中都加上這個位址就會變得很麻煩。另一種做法是，可以在 `config/mail.php` 設定檔中指定一個全域的「from」位址。若 Mailable 類別中為指定其他的「from」位址，則會使用這個位址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，也可以在 `config/mail.php` 設定檔中定義一個全域的「reply_to」位址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```

<a name="configuring-the-view"></a>
### 設定 View

在 Mailable 類別的 `content` 方法中，可以定義 `view`，也就是在渲染郵件內容時要使用哪個樣板。由於每封郵件通常都會使用 [Blade 樣板](/docs/{{version}}/blade) 來渲染其內容，因此在建立郵件的 HTML 時，就可以使用 Blade 樣板引擎的完整功能與便利性：

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
> 建議在 `resources/views/mail` 目錄下建立並管理所有的郵件樣板；不過，也可以將這些樣板放在 `resources/views` 目錄下的任何地方。

<a name="plain-text-emails"></a>
#### 純文字郵件

若想為郵件定義純文字版本，可以在建立訊息的 `Content` 定義時指定純文字樣板。與 `view` 參數類似，`text` 參數應為樣板的名稱，用來渲染郵件的內容。我們可以自由地為訊息定義 HTML 與純文字兩種版本：

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

為了清楚起見，`html` 參數可作為 `view` 參數的別名：

```php
return new Content(
    html: 'mail.orders.shipped',
    text: 'mail.orders.shipped-text'
);
```

<a name="view-data"></a>
### View 資料

<a name="via-public-properties"></a>
#### 透過 Public 屬性

一般來說，我們會想傳遞一些資料給 View，以便在渲染郵件的 HTML 時使用。有兩種方式可以將資料提供給 View。第一種，在 Mailable 類別上定義的任何 Public (公開) 屬性都會自動提供給 View。舉例來說，可以將資料傳入 Mailable 類別的建構函式，並將該資料設定給類別上定義的 Public 屬性：

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

資料被設為 Public 屬性後，就會自動在 View 中可用，因此可以像存取 Blade 樣板中的其他資料一樣存取該資料：

```blade
<div>
    Price: {{ $order->price }}
</div>
```

<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

若想在將郵件資料送至樣板前來自訂其格式，可以透過 `Content` 定義的 `with` 參數來手動將資料傳給 View。一般來說，還是會透過 Mailable 類別的建構函式來傳入資料；不過，應將這些資料設為 `protected` 或 `private` 屬性，這樣資料才不會自動提供給樣板：

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

資料透過 `with` 參數傳入後，就會自動在 View 中可用，因此可以像存取 Blade 樣板中的其他資料一樣存取該資料：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

若要為 E-mail 加上附件，請在訊息的 `attachments` 方法所回傳的陣列中加上附件。首先，我們可以透過將檔案路徑提供給 `Attachment` 類別的 `fromPath` 方法來加上附件：

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

在為訊息附加檔案時，也可以使用 `as` 與 `withMime` 方法來指定附件的顯示名稱與／或 MIME 類型：

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

若已將檔案儲存於其中一個[檔案系統磁碟](/docs/{{version}}/filesystem)上，則可使用 `fromStorage` 附件方法來將該檔案附加到 E-mail 上：

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

當然，我們也可以指定附件的名稱與 MIME 類型：

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

若需要指定預設磁碟以外的儲存磁碟，則可使用 `fromStorageDisk` 方法：

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

`fromData` 附件方法可用來將原始位元組字串作為附件來附加。舉例來說，若已在記憶體中產生了一份 PDF 且想將其附加到 E-mail 中而不寫入磁碟，就可以使用這個方法。`fromData` 方法可接受一個閉包 (Closure)，該閉包會解析出原始資料位元組，以及該附件應被指派的名稱：

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

將行內圖片內嵌至 E-mail 中通常很麻煩；不過，Laravel 提供了一個方便的方法來為 E-mail 附加圖片。若要內嵌行內圖片，請在 E-mail 範本中使用 `$message` 變數上的 `embed` 方法。Laravel 會自動讓 `$message` 變數可用於所有的 E-mail 範本，所以不需要擔心要手動傳入：

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]
> `$message` 變數無法在純文字訊息範本中使用，因為純文字訊息不使用行內附件。

<a name="embedding-raw-data-attachments"></a>
#### 內嵌原始資料附件

若已有想內嵌至 E-mail 範本中的原始圖片資料字串，則可呼叫 `$message` 變數上的 `embedData` 方法。在呼叫 `embedData` 方法時，需要提供一個檔名來指派給該內嵌圖片：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```

<a name="attachable-objects"></a>
### 可附加物件

雖然透過簡單的字串路徑來為訊息附加檔案通常已足夠，但在許多情況下，應用程式中的可附加實體 (Entity) 會由類別來表示。舉例來說，若應用程式正在為訊息附加一張相片，則應用程式中可能也會有一個用來表示該相片的 `Photo` Model。在這種情況下，若能直接將 `Photo` Model 傳給 `attach` 方法，不是就很方便嗎？可附加物件 (Attachable Object) 就能辦到這點。

首先，請在會附加到訊息中的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。該介面會要求類別中必須定義一個 `toMailAttachment` 方法，且該方法會回傳一個 `Illuminate\Mail\Attachment` 實體：

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

定義好可附加物件後，就可以在建立 E-mail 訊息時從 `attachments` 方法回傳該物件的實體：

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

當然，附件資料也可以儲存在如 Amazon S3 等遠端檔案儲存服務上。因此，Laravel 也允許我們從儲存在應用程式的其中一個[檔案系統磁碟](/docs/{{version}}/filesystem)上的資料來產生附件實體：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，我們也可以透過記憶體中的資料來建立附件實體。若要這麼做，請提供一個閉包給 `fromData` 方法。該閉包應回傳代表附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 還提供了其他方法來讓我們自訂附件。舉例來說，我們可以使用 `as` 與 `withMime` 方法來自訂檔案的名稱與 MIME 類型：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```

<a name="headers"></a>
### 標頭

有時候，我們可能需要為要寄出的訊息附加額外的標頭 (Header)。例如，我們可能需要設定自訂的 `Message-Id` 或其他任意的文字標頭。

若要這麼做，請在 Mailable 上定義一個 `headers` 方法。`headers` 方法應回傳一個 `Illuminate\Mail\Mailables\Headers` 實體。這個類別可接受 `messageId`、`references`、與 `text` 參數。當然，我們只需要為特定的訊息提供需要的參數即可：

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

有些第三方郵件供應商 (如 Mailgun 與 Postmark) 支援訊息「標籤」與「中繼資料」，可用來將應用程式寄出的郵件分組與追蹤。可透過 `Envelope` 定義來為郵件訊息加上標籤與中繼資料：

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

若應用程式正在使用 Mailgun Driver，可參閱 Mailgun 的說明文件來進一步了解其[標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags)與[中繼資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages)。同樣地，也可以參閱 Postmark 的說明文件來進一步了解其對於[標籤](https://postmarkapp.com/blog/tags-support-for-smtp)與[中繼資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq)的支援。

若應用程式是使用 Amazon SES 來寄送郵件，則應使用 `metadata` 方法來為訊息附加 [SES「標籤」](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)。


<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能是由 Symfony Mailer 所驅動。Laravel 允許我們註冊自訂的回呼函式。這些回呼函式會在訊息寄出前，以 Symfony Message 的實體為引數叫用。這樣一來，我們就有機會在訊息寄出前對其進行深度自訂。若要這麼做，請在 `Envelope` 定義中定義一個 `using` 參數：

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

Markdown Mailable 訊息可讓你在 Mailable 中利用[郵件通知](/docs/{{version}}/notifications#mail-notifications)預先建立好的樣板與元件。由於訊息是以 Markdown 撰寫，Laravel 能為訊息渲染出漂亮、響應式的 HTML 樣板，同時也會自動產生一份純文字版的訊息。

<a name="generating-markdown-mailables"></a>
### 產生 Markdown Mailable

若要產生帶有對應 Markdown 樣板的 Mailable，可使用 `make:mail` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

接著，在 Mailable 的 `content` 方法中設定 `Content` 定義時，請使用 `markdown` 參數，而不是 `view` 參數：

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

Markdown Mailable 結合了 Blade 元件與 Markdown 語法，讓我們能利用 Laravel 預先建立好的郵件 UI 元件來輕鬆地建構郵件訊息：

```blade
<x-mail::message>

# 訂單已出貨

您的訂單已出貨！

<x-mail::button :url="$url">
檢視訂單
</x-mail::button>

謝謝,<br>
{{ config('app.name') }}
</x-mail::message>
```

> [!NOTE]
> 撰寫 Markdown 郵件時，請勿使用多餘的縮排。根據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。


<a name="button-component"></a>
#### 按鈕元件

按鈕元件會渲染一個置中的按鈕連結。該元件接受兩個引數：`url` 以及選用的 `color`。支援的顏色有 `primary`、`success`、`error`。可在訊息中加入任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```


<a name="panel-component"></a>
#### 面板元件

面板元件會將給定的文字區塊渲染在一個面板中，該面板的背景顏色會與訊息的其他部分稍有不同。這樣就能將讀者的注意力吸引到給定的文字區塊上：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### 表格元件

表格元件可將 Markdown 表格轉換為 HTML 表格。該元件接受 Markdown 表格為其內容。可使用預設的 Markdown 表格對齊語法來支援表格欄位對齊：

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

可以將所有的 Markdown 郵件元件匯出至自己的專案中來自訂。若要匯出元件，可使用 `vendor:publish` Artisan 指令來發布 `laravel-mail` 這個 Asset Tag：

```shell
php artisan vendor:publish --tag=laravel-mail
```

這個指令會將 Markdown 郵件元件發布到 `resources/views/vendor/mail` 目錄。`mail` 目錄中會有 `html` 與 `text` 目錄，其中分別包含了每個可用元件的對應版本。可隨意自訂這些元件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄中會有一個 `default.css` 檔。可自訂此檔案中的 CSS，然後這些樣式就會在 Markdown 郵件訊息的 HTML 版面中自動被轉換為行內 (inline) CSS 樣式。

若想為 Laravel 的 Markdown 元件建立一個全新的主題，可將一個 CSS 檔放到 `html/themes` 目錄中。命名並儲存好 CSS 檔後，請更新專案 `config/mail.php` 設定檔中的 `theme` 選項，使其符合新主題的名稱。

若要為個別的 Mailable 自訂主題，可設定該 Mailable Class 的 `$theme` 屬性，為寄送該 Mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 寄送郵件

若要寄送訊息，請使用 `Mail` [Facade](/docs/{{version}}/facades) 上的 `to` 方法。`to` 方法可接受一個 E-mail 地址、一個使用者實體、或一個使用者集合。若傳入一個物件或物件集合，Mailer 會自動使用其 `email` 與 `name` 屬性來判斷郵件收件者，所以請確定物件上有這些屬性。指定好收件者後，就可以傳入 Mailable 類別的實體給 `send` 方法：

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

寄送訊息時，不限於只能指定「to」收件者。我們可以自由地將「to」、「cc」、「bcc」等收件者的方法串連在一起設定：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```

<a name="looping-over-recipients"></a>
#### 遍歷收件者

有時候，我們可能會需要透過遍歷收件者陣列或 E-mail 地址陣列來將 Mailable 寄給一串收件者列表。不過，由於 `to` 方法會將 E-mail 地址附加到該 Mailable 的收件者列表中，所以每次迴圈迭代時，都會再將郵件寄給所有先前的收件者。因此，我們應為每個收件者都重新建立一個 Mailable 實體：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```

<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過指定的 Mailer 寄送郵件

預設情況下，Laravel 會使用應用程式的 `mail` 設定檔中設定為 `default` 的 Mailer 來寄送郵件。不過，我們也可以使用 `mailer` 方法來透過指定的 Mailer 設定寄送郵件：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```

<a name="queueing-mail"></a>
### 佇列郵件

<a name="queueing-a-mail-message"></a>
#### 將郵件訊息加入佇列

由於寄送郵件訊息可能會對應用程式的回應時間造成負面影響，許多開發者會選擇將郵件訊息加入佇列並在背景寄送。Laravel 使用其內建的[統一佇列 API](/docs/{{version}}/queues) 讓這件事變得很簡單。若要將郵件訊息加入佇列，請在指定好訊息的收件者後，使用 `Mail` Facade 上的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法會自動處理將工作推送到佇列上，讓訊息能在背景寄送。在使用此功能前，需要先[設定好佇列](/docs/{{version}}/queues)。

<a name="delayed-message-queueing"></a>
#### 延遲訊息佇列

若想延遲寄送已佇列的郵件訊息，可使用 `later` 方法。`later` 方法的第一個引數可接受一個 `DateTime` 實體，用來指出該則訊息應於何時寄送：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->addMinutes(10), new OrderShipped($order));
```

<a name="pushing-to-specific-queues"></a>
#### 推送到指定的佇列

由於所有使用 `make:mail` 指令產生的 Mailable 類別都有使用 `Illuminate\Bus\Queueable` Trait，因此我們可以在任何 Mailable 類別實體上呼叫 `onQueue` 與 `onConnection` 方法，以指定該訊息的連線與佇列名稱：

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

若有些 Mailable 類別想讓其總是加入佇列，則可以在該類別上實作 `ShouldQueue` Contract。現在，即使在寄送郵件時呼叫了 `send` 方法，該 Mailable 也還是會被加入佇列，因為它實作了這個 Contract：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```

<a name="queued-mailables-and-database-transactions"></a>
#### 已佇列的 Mailable 與資料庫交易

當已佇列的 Mailable 在資料庫交易中被分派時，它們可能會在資料庫交易提交前就被佇列處理。發生這種情況時，我們在資料庫交易中所做的任何 Model 或資料庫記錄更新可能都還沒反映到資料庫中。此外，在交易中建立的任何 Model 或資料庫記錄可能都還不存在於資料庫中。若 Mailable 相依于這些 Model，則在處理寄送已佇列 Mailable 的工作時，就可能發生非預期的錯誤。

若佇列連線的 `after_commit` 設定選項為 `false`，我們還是可以在寄送郵件訊息時呼叫 `afterCommit` 方法，以表示某個特定的已佇列 Mailable 應在所有開啟的資料庫交易都提交後才分派：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，也可以在 Mailable 的建構函式中呼叫 `afterCommit` 方法：

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
> 了解更多有關如何處理這些問題的資訊，請參考有關[已佇列工作與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的文件。

<a name="queued-email-failures"></a>
#### 已佇列郵件的失敗

當已佇列的郵件失敗時，若已佇列的 Mailable 類別上有定義 `failed` 方法，則會叫用該方法。造成已佇列郵件失敗的 `Throwable` 實體會被傳入 `failed` 方法：

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

有時候，我們可能會想在不寄送的情況下擷取 Mailable 的 HTML 內容。為此，我們可以呼叫 Mailable 的 `render` 方法。該方法會以字串形式回傳 Mailable 求值後的 HTML 內容：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```

<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailable

在設計 Mailable 的範本時，能像一般的 Blade 範本一樣在瀏覽器中快速預覽渲染好的 Mailable 是很方便的。因此，Laravel 允許我們直接從路由閉包或 Controller 中回傳任何 Mailable。當 Mailable 被回傳時，它會被渲染並顯示在瀏覽器中，讓我們能快速預覽其設計，而不需要實際將其寄到 E-mail 地址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```

<a name="localizing-mailables"></a>
## 本地化 Mailable

Laravel 允許我們以請求當前語系以外的語系來寄送 Mailable。若郵件有被佇列，Laravel 還會記住這個語系。

若要這麼做，`Mail` Facade 提供了 `locale` 方法來設定想要的語言。在對 Mailable 的樣板進行求值時，應用程式會切換為該語系，並在求值完成後切換回先前的語系：

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```


<a name="user-preferred-locales"></a>
#### 使用者偏好的語系

有時候，應用程式會儲存每個使用者偏好的語系。只要在一個或多個 Model 上實作 `HasLocalePreference` Contract，就可以讓 Laravel 在寄送郵件時使用這個已儲存的語系：

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

實作完該介面後，Laravel 就會在寄送 Mailable 與通知給該 Model 時自動使用偏好的語系。因此，在使用此介面時，就不需要再呼叫 `locale` 方法了：

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試


<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供了許多方法來檢查 Mailable 的結構。此外，Laravel 還提供了一些方便的方法來測試 Mailable 是否包含預期的內容：

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

如你所想，「HTML」相關的斷言 (Assertion) 是用來斷言 Mailable 的 HTML 版本中包含給定的字串，而「text」相關的斷言則是用來斷言 Mailable 的純文字版本中包含給定的字串。


<a name="testing-mailable-sending"></a>
### 測試 Mailable 寄送

我們建議將 Mailable 內容的測試與斷言某個 Mailable 是否已「寄給」特定使用者的測試分開。一般來說，Mailable 的內容與正在測試的程式碼無關，只要能簡單地斷言 Laravel 已收到寄送某個 Mailable 的指令就夠了。

我們可以使用 `Mail` Facade 的 `fake` 方法來防止郵件被寄出。在呼叫 `Mail` Facade 的 `fake` 方法後，就可以斷言 Mailable 是否已收到寄給使用者的指令，甚至還可以檢查 Mailable 收到的資料：

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

若要在背景佇列 Mailable 來寄送，則應使用 `assertQueued` 方法而不是 `assertSent`：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

我們可以傳遞一個閉包 (Closure) 到 `assertSent`、`assertNotSent`、`assertQueued`、或 `assertNotQueued` 等方法中，來斷言寄出的 Mailable 是否通過給定的「真實性測試」。只要至少有一個寄出的 Mailable 通過給定的真實性測試，該斷言就會成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

在呼叫 `Mail` Facade 的斷言方法時，傳入的閉包所接受的 Mailable 實體提供了一些有用的方法來檢查該 Mailable：

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

該 Mailable 實體也包含數個有用的方法來檢查 Mailable 上的附件：

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

你可能已經注意到，有兩個方法可用來斷言郵件未被寄出：`assertNotSent` 與 `assertNotQueued`。有時候，我們可能會想斷言郵件**既沒有**被寄出，**也沒有**被佇列。為此，我們可以使用 `assertNothingOutgoing` 與 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本機開發

在開發會寄送郵件的應用程式時，你可能不會想真的把郵件寄送到真實的電子郵件信箱。Laravel 提供了數種方法，能在本機開發時「停用」實際的郵件寄送功能。


<a name="log-driver"></a>
#### Log Driver

`log` 郵件 Driver 不會真的寄出郵件，而是會將所有的郵件訊息都寫入 Log 檔中以供檢查。一般來說，這個 Driver 只會在本機開發時使用。更多有關各環境設定檔的資訊，請參考[設定檔文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，我們也可以使用像 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 這類的服務，並搭配 `smtp` Driver 來將郵件訊息寄送到一個「虛設的 (Dummy)」信箱，然後再到真正的郵件客戶端中檢視。這個方法的好處是，我們可以在 Mailtrap 的訊息檢視器中實際檢查最終寄出的郵件。

若正在使用 [Laravel Sail](/docs/{{version}}/sail)，則可以使用 [Mailpit](https://github.com/axllent/mailpit) 來預覽郵件。當 Sail 在執行時，可以到 `http://localhost:8025` 來存取 Mailpit 的介面。


<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 位址

最後，我們也可以叫用 `Mail` Facade 提供的 `alwaysTo` 方法來指定一個全域的「收件人 (to)」信箱。一般來說，這個方法應在某個應用程式 Service Provider 的 `boot` 方法中呼叫：

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

Laravel 在寄送郵件訊息時會分派 (Dispatch) 兩個事件。`MessageSending` 事件會在訊息寄出前分派，而 `MessageSent` 事件則會在訊息寄出後分派。請記得，這些事件是在郵件被 *寄出 (sent)* 時分派的，而不是在郵件被佇列 (Queued) 起來時。我們可以在應用程式中為這些事件建立[事件監聽器](/docs/{{version}}/events)：

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
## 自訂 Transport

Laravel 內建了多種郵件 Transport；不過，有時候我們可能會想自行撰寫 Transport 來使用 Laravel 未內建支援的其他服務來寄送郵件。首先，先定義一個繼承 `Symfony\Component\Mailer\Transport\AbstractTransport` Class 的類別。然後，在你的 Transport 上實作 `doSend` 與 `__toString` 方法：

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

定義好自訂的 Transport 後，就可以通過 `Mail` Facade 提供的 `extend` 方法來註冊。一般來說，應在應用程式的 `AppServiceProvider` 的 `boot` 方法中進行。傳給 `extend` 方法的閉包會被傳入一個 `$config` 引數。這個引數會包含在 `config/mail.php` 設定檔中為該 Mailer 定義的設定陣列：

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

定義並註冊好自訂的 Transport 後，就可以在 `config/mail.php` 設定檔中建立一個使用該新 Transport 的 Mailer 定義：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```


<a name="additional-symfony-transports"></a>
### 其他 Symfony Transport

Laravel 內建支援了一些現有的、由 Symfony 維護的郵件 Transport，如 Mailgun 與 Postmark。不過，我們有時候可能會想擴充 Laravel 來支援其他由 Symfony 維護的 Transport。我們可以通過 Composer 來 Require (引用) 必要的 Symfony Mailer，並將該 Transport 註冊到 Laravel 中。舉例來說，我們可以安裝並註冊「Brevo」(原「Sendinblue」) Symfony Mailer：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

安裝好 Brevo Mailer 套件後，就可以在 `services` 設定檔中為你的 Brevo API 憑證新增一個項目：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接著，我們可以使用 `Mail` Facade 的 `extend` 方法來將該 Transport 註冊到 Laravel 中。一般來說，應在某個 Service Provider 的 `boot` 方法中進行：

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

註冊好 Transport 後，就可以在 `config/mail.php` 設定檔中建立一個使用該新 Transport 的 Mailer 定義：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```