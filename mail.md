# Mail

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動器事前準備](#driver-prerequisites)
    - [備援轉移設定](#failover-configuration)
    - [輪詢設定](#round-robin-configuration)
- [建立 Mailable](#generating-mailables)
- [撰寫 Mailable](#writing-mailables)
    - [設定寄件者](#configuring-the-sender)
    - [設定視圖](#configuring-the-view)
    - [視圖資料](#view-data)
    - [附件](#attachments)
    - [行內附件](#inline-attachments)
    - [可附加物件](#attachable-objects)
    - [標頭](#headers)
    - [標籤與元資料](#tags-and-metadata)
    - [自訂 Symfony 訊息](#customizing-the-symfony-message)
- [Markdown Mailable](#markdown-mailables)
    - [建立 Markdown Mailable](#generating-markdown-mailables)
    - [撰寫 Markdown 訊息](#writing-markdown-messages)
    - [自訂元件](#customizing-the-components)
- [寄送郵件](#sending-mail)
    - [佇列郵件](#queueing-mail)
- [渲染 Mailable](#rendering-mailables)
    - [在瀏覽器中預覽 Mailable](#previewing-mailables-in-the-browser)
- [Mailable 在地化](#localizing-mailables)
- [測試](#testing-mailables)
    - [測試 Mailable 內容](#testing-mailable-content)
    - [測試 Mailable 寄送](#testing-mailable-sending)
- [郵件與本機開發](#mail-and-local-development)
- [事件](#events)
- [自訂傳輸器](#custom-transports)
    - [額外的 Symfony 傳輸器](#additional-symfony-transports)

<a name="introduction"></a>
## 簡介

發送電子郵件不必很複雜。Laravel 提供了由熱門的 [Symfony Mailer](https://symfony.com/doc/current/mailer.html) 元件所強化的乾淨、簡單的電子郵件 API。Laravel 與 Symfony Mailer 提供了透過 SMTP、Cloudflare、Mailgun、Postmark、Resend、Amazon SES 和 `sendmail` 發送電子郵件的驅動器，讓您能夠快速開始透過您選擇的本機或雲端服務發送郵件。


<a name="configuration"></a>
### 設定

Laravel 的電子郵件服務可以透過您應用程式的 `config/mail.php` 設定檔進行設定。在此檔案中設定的每個 mailer 都可以有自己獨特的設定，甚至是自己獨特的「傳輸器 (transport)」，讓您的應用程式能使用不同的電子郵件服務來發送特定的郵件。例如，您的應用程式可能會使用 Postmark 來發送交易型郵件，同時使用 Amazon SES 來發送大量郵件。

在您的 `mail` 設定檔中，您會找到一個 `mailers` 設定陣列。這個陣列包含 Laravel 支援的每個主要郵件驅動器 / 傳輸器的範例設定項目，而 `default` 設定值則決定了當您的應用程式需要發送電子郵件時，預設將使用哪一個 mailer。


<a name="driver-prerequisites"></a>
### 驅動器事前準備

基於 API 的驅動器（例如 Mailgun、Postmark 與 Resend）通常比透過 SMTP 伺服器發送郵件更簡單且更快速。只要可能，我們建議您使用這些驅動器的其中之一。


<a name="cloudflare-driver"></a>
#### Cloudflare 驅動器

要使用 Cloudflare 驅動器，請透過 Composer 安裝 Symfony 的 HTTP Client：

```shell
composer require symfony/http-client
```

接下來，您需要在應用程式的 `config/mail.php` 設定檔中進行兩項修改。首先，將您的預設 mailer 設定為 `cloudflare`：

```php
'default' => env('MAIL_MAILER', 'cloudflare'),
```

第二，將以下設定陣列新增至您的 `mailers` 陣列中：

```php
'cloudflare' => [
    'transport' => 'cloudflare',
],
```

設定完應用程式的預設 mailer 後，請將以下選項新增至您的 `config/services.php` 設定檔中：

```php
'cloudflare' => [
    'account_id' => env('CLOUDFLARE_ACCOUNT_ID'),
    'key' => env('CLOUDFLARE_KEY'),
],
```


<a name="mailgun-driver"></a>
#### Mailgun 驅動器

要使用 Mailgun 驅動器，請透過 Composer 安裝 Symfony 的 Mailgun Mailer 傳輸器：

```shell
composer require symfony/mailgun-mailer symfony/http-client
```

接下來，您需要在應用程式的 `config/mail.php` 設定檔中進行兩項修改。首先，將您的預設 mailer 設定為 `mailgun`：

```php
'default' => env('MAIL_MAILER', 'mailgun'),
```

第二，將以下設定陣列新增至您的 `mailers` 陣列中：

```php
'mailgun' => [
    'transport' => 'mailgun',
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

設定完應用程式的預設 mailer 後，請將以下選項新增至您的 `config/services.php` 設定檔中：

```php
'mailgun' => [
    'domain' => env('MAILGUN_DOMAIN'),
    'secret' => env('MAILGUN_SECRET'),
    'endpoint' => env('MAILGUN_ENDPOINT', 'api.mailgun.net'),
    'scheme' => 'https',
],
```

如果您不是使用美國的 [Mailgun 地區](https://documentation.mailgun.com/docs/mailgun/api-reference/api-overview#mailgun-regions)，可以在 `services` 設定檔中定義您所在地區的端點：

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

要使用 [Postmark](https://postmarkapp.com/) 驅動器，請透過 Composer 安裝 Symfony 的 Postmark Mailer 傳輸器：

```shell
composer require symfony/postmark-mailer symfony/http-client
```

接下來，將您應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `postmark`。設定完應用程式的預設 mailer 後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'postmark' => [
    'key' => env('POSTMARK_API_KEY'),
],
```

如果您想要指定特定 mailer 所應使用的 Postmark 訊息串流 (message stream)，可以在該 mailer 的設定陣列中新增 `message_stream_id` 設定選項。此設定陣列可以在您應用程式的 `config/mail.php` 設定檔中找到：

```php
'postmark' => [
    'transport' => 'postmark',
    'message_stream_id' => env('POSTMARK_MESSAGE_STREAM_ID'),
    // 'client' => [
    //     'timeout' => 5,
    // ],
],
```

如此一來，您就能設定多個使用不同訊息串流的 Postmark mailer。


<a name="resend-driver"></a>
#### Resend 驅動器

要使用 [Resend](https://resend.com/) 驅動器，請透過 Composer 安裝 Resend 的 PHP SDK：

```shell
composer require resend/resend-php
```

接下來，將您應用程式 `config/mail.php` 設定檔中的 `default` 選項設定為 `resend`。設定完應用程式的預設 mailer 後，請確保您的 `config/services.php` 設定檔包含以下選項：

```php
'resend' => [
    'key' => env('RESEND_API_KEY'),
],
```


<a name="ses-driver"></a>
#### SES 驅動器

要使用 Amazon SES 驅動器，您必須先安裝適用於 PHP 的 Amazon AWS SDK。您可以透過 Composer 套件管理工具安裝此函式庫：

```shell
composer require aws/aws-sdk-php
```

接下來，將 `config/mail.php` 設定檔中的 `default` 選項設定為 `ses`，並確認您的 `config/services.php` 設定檔包含以下選項：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
],
```

若要透過 Session Token 使用 AWS [臨時憑證 (temporary credentials)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)，您可以在應用程式的 SES 設定中新增一個 `token` 鍵值：

```php
'ses' => [
    'key' => env('AWS_ACCESS_KEY_ID'),
    'secret' => env('AWS_SECRET_ACCESS_KEY'),
    'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
    'token' => env('AWS_SESSION_TOKEN'),
],
```

若要與 SES 的[訂閱管理功能](https://docs.aws.amazon.com/ses/latest/dg/sending-email-subscription-management.html)進行互動，您可以在郵件訊息的 [headers](#headers) 方法回傳的陣列中，回傳 `X-Ses-List-Management-Options` 標頭：

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

若要透過 SES [租戶 (tenant)](https://docs.aws.amazon.com/ses/latest/dg/tenants.html) 發送電子郵件，您可以從 `headers` 方法回傳 `X-Ses-Tenant-Name` 標頭。發送訊息時，Laravel 會將標頭值作為 `TenantName` 選項傳遞給 SES：

```php
public function headers(): Headers
{
    return new Headers(
        text: [
            'X-Ses-Tenant-Name' => 'tenant-id',
        ],
    );
}
```

如果您想要定義發送電子郵件時 Laravel 應該傳遞給 AWS SDK `SendEmail` 方法的[額外選項](https://docs.aws.amazon.com/aws-sdk-php/v3/api/api-sesv2-2019-09-27.html#sendemail)，可以在 `ses` 設定中定義一個 `options` 陣列：

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
### 備援轉移設定

有時候，您所設定用來寄送應用程式郵件的外部服務可能會發生宕機。在這些情況下，定義一個或多個備份郵件遞送設定會非常有用，當主要遞送驅動器停止服務時，便能使用這些備份設定。

若要實現這一點，您應該在應用程式的 `mail` 設定檔中定義一個使用 `failover` 傳輸器的 Mailer。應用程式中 `failover` Mailer 的設定陣列應該包含一個 `mailers` 陣列，用來指定選擇已設定之 Mailer 進行遞送的順序：

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

一旦您設定了使用 `failover` 傳輸器的 Mailer，您需要在應用程式的 `.env` 檔案中將此 Failover Mailer 設定為預設 Mailer，才能使用備援轉移功能：

```ini
MAIL_MAILER=failover
```


<a name="round-robin-configuration"></a>
### 輪詢設定

`roundrobin` 傳輸器允許您將郵件寄送工作負載分散到多個 Mailer 上。若要開始使用，請在應用程式的 `mail` 設定檔中定義一個使用 `roundrobin` 傳輸器的 Mailer。您應用程式中 `roundrobin` Mailer 的設定陣列應包含一個 `mailers` 陣列，用來指定有哪些已設定的 Mailer 應該被用於遞送：

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

一旦定義了輪詢 Mailer，您應該透過在應用程式的 `mail` 設定檔中將其名稱指定為 `default` 設定鍵的值，來將此 Mailer 設定為應用程式使用的預設 Mailer：

```php
'default' => env('MAIL_MAILER', 'roundrobin'),
```

輪詢傳輸器會從已設定的 Mailer 列表中隨機選擇一個 Mailer，然後在後續的每封電子郵件中切換到下一個可用的 Mailer。與有助於實現 *[高可用性](https://en.wikipedia.org/wiki/High_availability)* 的 `failover` 傳輸器不同，`roundrobin` 傳輸器提供的是 *[負載平衡](https://en.wikipedia.org/wiki/Load_balancing_(computing))*。

<a name="generating-mailables"></a>
## 建立 Mailable

在建構 Laravel 應用程式時，應用程式寄出的每一種電子郵件類型都由一個 "mailable" 類別來代表。這些類別儲存在 `app/Mail` 目錄中。如果在您的應用程式中沒看到這個目錄也不用擔心，因為當您使用 `make:mail` Artisan 指令建立第一個 mailable 類別時，系統就會自動為您產生這個目錄：

```shell
php artisan make:mail OrderShipped
```

<a name="writing-mailables"></a>
## 撰寫 Mailable

當您建立 Mailable 類別後，打開它以便我們探索其內容。Mailable 類別的設定是透過幾個方法來完成的，包含 `envelope`、`content` 和 `attachments` 方法。

`envelope` 方法會回傳一個 `Illuminate\Mail\Mailables\Envelope` 物件，用來定義郵件的主旨，以及有時包含的收件者。`content` 方法則會回傳一個 `Illuminate\Mail\Mailables\Content` 物件，用來定義用於產生郵件內容的 [Blade 模板](/docs/{{version}}/blade)。


<a name="configuring-the-sender"></a>
### 設定寄件者


<a name="using-the-envelope"></a>
#### 使用 Envelope

首先，讓我們來探討如何設定郵件的寄件者。或者換句話說，也就是這封郵件是由誰「寄出 (from)」。有兩種方式可以設定寄件者。第一種，您可以在郵件的 Envelope 上指定 "from" 地址：

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

如果您願意，也可以指定 `replyTo` 地址：

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

然而，如果您的應用程式對所有郵件都使用相同的 "from" 地址，每次建立 Mailable 類別都要重複設定可能會非常繁瑣。相對地，您可以在 `config/mail.php` 設定檔中指定一個全域的 "from" 地址。若 Mailable 類別中未指定其他的 "from" 地址，系統就會使用這個全域地址：

```php
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
```

此外，您也可以在 `config/mail.php` 設定檔中定義全域的 "reply_to" 地址：

```php
'reply_to' => [
    'address' => 'example@example.com',
    'name' => 'App Name',
],
```


<a name="configuring-the-view"></a>
### 設定視圖

在 Mailable 類別的 `content` 方法中，您可以定義 `view`，也就是渲染郵件內容時應該使用的模板。由於每封郵件通常都是使用 [Blade 模板](/docs/{{version}}/blade) 來渲染內容，因此在建構郵件的 HTML 時，您能充分利用 Blade 模板引擎的強大功能與便利性：

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
> 您可能希望建立 `resources/views/mail` 目錄來存放所有郵件模板；不過，您也可以自由地將它們放置在 `resources/views` 目錄下的任何位置。


<a name="plain-text-emails"></a>
#### 純文字郵件

如果您想定義郵件的純文字版本，可以在建立郵件的 `Content` 定義時指定純文字模板。如同 `view` 參數，`text` 參數應該是一個用於渲染郵件內容的模板名稱。您可以自由地同時定義郵件的 HTML 版本與純文字版本：

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

為了更清晰起見，`html` 參數可以作為 `view` 參數的別名使用：

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

通常，您會希望傳送一些資料給視圖，以便在渲染郵件的 HTML 時使用。有兩種方法可以讓視圖取得資料。第一種，在 Mailable 類別上定義的任何公開屬性均會自動提供給視圖使用。因此，例如您可以將資料傳入 Mailable 類別的建構子，並將該資料設定給類別上定義的公開屬性：

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

一旦資料設定給公開屬性後，它就會自動在視圖中可用，因此您可以像在 Blade 模板中存取任何其他資料一樣來存取它：

```blade
<div>
    Price: {{ $order->price }}
</div>
```


<a name="via-the-with-parameter"></a>
#### 透過 `with` 參數：

如果您想在郵件資料發送到模板之前自訂其格式，可以透過 `Content` 定義的 `with` 參數手動將資料傳遞給視圖。通常，您仍然會透過 Mailable 類別的建構子傳遞資料；但是，您應該將此資料設定為 `protected` 或 `private` 屬性，這樣資料就不會自動提供給模板：

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

當資料透過 `with` 參數傳遞後，它會自動在您的視圖中可用，因此您可以像在 Blade 模板中存取任何其他資料一樣來存取它：

```blade
<div>
    Price: {{ $orderPrice }}
</div>
```

<a name="attachments"></a>
### 附件

若要將附件新增至電子郵件，您可以在郵件的 `attachments` 方法所回傳的陣列中加入附件。首先，您可以透過將檔案路徑提供給 `Attachment` 類別所提供的 `fromPath` 方法來新增附件：

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

將檔案附加至郵件時，您也可以使用 `as` 與 `withMime` 方法來指定附件的顯示名稱及／或 MIME 型態：

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

如果您已將檔案儲存於 [檔案系統磁碟](/docs/{{version}}/filesystem) 上，可以使用 `fromStorage` 附件方法將其附加至電子郵件：

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

當然，您也可以指定附件的名稱與 MIME 型態：

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

`fromData` 附件方法可用於將原始位元組字串作為附件附加。例如，如果您在記憶體中產生了 PDF 且不想寫入磁碟就將其附加到電子郵件，您可能就會使用此方法。`fromData` 方法接受一個閉包，該閉包解析原始資料位元組以及應指派給該附件的名稱：

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

將行內圖片嵌入到電子郵件中通常很繁瑣；然而，Laravel 提供了一種便利的方式來將圖片附加至您的電子郵件中。若要嵌入行內圖片，請在電子郵件範本內對 `$message` 變數使用 `embed` 方法。Laravel 會自動讓 `$message` 變數在所有的電子郵件範本中皆可用，因此您不需要擔心要手動傳入它：

```blade
<body>
    Here is an image:

    <img src="{{ $message->embed($pathToImage) }}">
</body>
```

> [!WARNING]
> `$message` 變數在純文字郵件範本中不可用，因為純文字郵件無法使用行內附件。


<a name="embedding-raw-data-attachments"></a>
#### 嵌入原始資料附件

如果您已經有一個想要嵌入到電子郵件範本中的原始圖片資料字串，可以在 `$message` 變數上呼叫 `embedData` 方法。呼叫 `embedData` 方法時，您需要提供一個應指派給該嵌入圖片的檔名：

```blade
<body>
    Here is an image from raw data:

    <img src="{{ $message->embedData($data, 'example-image.jpg') }}">
</body>
```


<a name="attachable-objects"></a>
### 可附加物件

雖然透過簡單的字串路徑將檔案附加至郵件通常就足夠了，但在許多情況下，應用程式中可附加的實體是由類別所代表的。例如，如果您的應用程式要將相片附加至郵件，您的應用程式可能也有一個代表該相片的 `Photo` Model。在這種情況下，如果能直接將 `Photo` Model 傳遞給 `attach` 方法不是非常方便嗎？可附加物件允許您做到這一點。

要開始使用，請在可附加至郵件的物件上實作 `Illuminate\Contracts\Mail\Attachable` 介面。該介面規定您的類別必須定義一個回傳 `Illuminate\Mail\Attachment` 實例的 `toMailAttachment` 方法：

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

當您定義好可附加物件後，可以在建構電子郵件訊息時，從 `attachments` 方法回傳該物件的實例：

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

當然，附件資料也可以儲存在遠端檔案儲存服務上（例如 Amazon S3）。因此，Laravel 也允許您從儲存在應用程式 [檔案系統磁碟](/docs/{{version}}/filesystem) 上的資料來產生附件實例：

```php
// Create an attachment from a file on your default disk...
return Attachment::fromStorage($this->path);

// Create an attachment from a file on a specific disk...
return Attachment::fromStorageDisk('backblaze', $this->path);
```

此外，您也可以透過記憶體中的資料來建立附件實例。若要做到這一點，請提供一個閉包給 `fromData` 方法。該閉包應回傳代表該附件的原始資料：

```php
return Attachment::fromData(fn () => $this->content, 'Photo Name');
```

Laravel 還提供了其他方法供您自訂附件。例如，您可以使用 `as` 與 `withMime` 方法來自訂檔案的名稱與 MIME 型態：

```php
return Attachment::fromPath('/path/to/file')
    ->as('Photo Name')
    ->withMime('image/jpeg');
```


<a name="headers"></a>
### 標頭

有時您可能需要為發出的郵件附加額外的標頭。例如，您可能需要設定自訂的 `Message-Id` 或其他任意的文字標頭。

若要做到這一點，請在您的 mailable 上定義一個 `headers` 方法。`headers` 方法應回傳一個 `Illuminate\Mail\Mailables\Headers` 實例。該類別接受 `messageId`、`references` 與 `text` 參數。當然，您可以僅提供特定郵件所需的參數：

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

某些第三方電子郵件提供者（例如 Mailgun 和 Postmark）支援訊息「標籤 (tags)」與「元資料 (metadata)」，這些可以用來對您應用程式寄出的郵件進行分組與追蹤。您可以透過 `Envelope` 定義將標籤與元資料新增至電子郵件訊息中：

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

若您的應用程式正在使用 Mailgun 驅動器，可以參考 Mailgun 的文件以取得更多關於[標籤](https://documentation.mailgun.com/docs/mailgun/user-manual/tracking-messages/#tags)與[元資料](https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/#attaching-metadata-to-messages)的資訊。同樣地，也可以參考 Postmark 文件瞭解更多關於其對[標籤](https://postmarkapp.com/blog/tags-support-for-smtp)與[元資料](https://postmarkapp.com/support/article/1125-custom-metadata-faq)支援的資訊。

若您的應用程式使用 Amazon SES 來寄送電子郵件，您應該使用 `metadata` 方法將 [SES "標籤"](https://docs.aws.amazon.com/ses/latest/APIReference/API_MessageTag.html)附加至訊息中。

<a name="customizing-the-symfony-message"></a>
### 自訂 Symfony 訊息

Laravel 的郵件功能由 Symfony Mailer 提供支援。Laravel 允許您註冊自訂的回呼函式，這些回呼函式會在寄送訊息之前，帶著 Symfony Message 實例被呼叫。這讓您有機會在訊息寄出前對其進行深度自訂。要實現此目的，請在您的 `Envelope` 定義上定義 `using` 參數：

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

Markdown mailable 訊息讓您可以在 mailable 中善用[郵件通知](/docs/{{version}}/notifications#mail-notifications)預先建置好的範本與元件。因為這些訊息是以 Markdown 撰寫的，所以 Laravel 能夠為這些訊息渲染出美觀且具響應式的 HTML 範本，同時也能自動產生相對應的純文字版本。


<a name="generating-markdown-mailables"></a>
### 建立 Markdown Mailable

若要建立包含對應 Markdown 範本的 mailable，您可以使用 `make:mail` Artisan 指令的 `--markdown` 選項：

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

然後，在其 `content` 方法中設定 mailable 的 `Content` 定義時，改用 `markdown` 參數而非 `view` 參數：

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

Markdown mailable 結合使用了 Blade 元件與 Markdown 語法，讓您可以在利用 Laravel 預建的郵件 UI 元件之餘，輕鬆建構郵件訊息：

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
> 撰寫 Markdown 郵件時，請勿使用過多的縮排。依據 Markdown 標準，Markdown 解析器會將縮排的內容渲染為程式碼區塊。


<a name="button-component"></a>
#### Button 元件

Button 元件會渲染出一個置中的按鈕連結。該元件接受兩個引數：`url` 與可選的 `color`。支援的顏色有 `primary`、`success` 及 `error`。您可以根據需求在訊息中新增任意數量的按鈕元件：

```blade
<x-mail::button :url="$url" color="success">
View Order
</x-mail::button>
```


<a name="panel-component"></a>
#### Panel 元件

Panel 元件會在背景顏色與訊息其餘部分略有不同的面板中渲染指定的文字區塊。這能讓您吸引讀者注意到特定的文字區塊：

```blade
<x-mail::panel>
This is the panel content.
</x-mail::panel>
```


<a name="table-component"></a>
#### Table 元件

Table 元件允許您將 Markdown 表格轉換為 HTML 表格。該元件接收 Markdown 表格作為其內容。支援使用預設的 Markdown 表格對齊語法來對齊表格欄位：

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

您可以將所有的 Markdown 郵件元件匯出至自己的應用程式中進行自訂。若要匯出元件，請使用 `vendor:publish` Artisan 指令來發布 `laravel-mail` 靜態資源標籤：

```shell
php artisan vendor:publish --tag=laravel-mail
```

這個指令會將 Markdown 郵件元件發布至 `resources/views/vendor/mail` 目錄。`mail` 目錄下會包含 `html` 和 `text` 目錄，分別代表每個可用元件對應的形式。您可以隨意自訂這些元件。


<a name="customizing-the-css"></a>
#### 自訂 CSS

匯出元件後，`resources/views/vendor/mail/html/themes` 目錄中會包含一個 `default.css` 檔案。您可以自訂此檔案中的 CSS，您的樣式將會自動轉換為 Markdown 郵件訊息 HTML 版本中的行內 CSS 樣式。

如果您想為 Laravel 的 Markdown 元件建立全新的主題，可以在 `html/themes` 目錄中放置一個 CSS 檔案。為 CSS 檔案命名並儲存後，更新應用程式 `config/mail.php` 設定檔中的 `theme` 選項，使其與新主題的名稱一致。

若要為單一 mailable 自訂主題，可以在該 mailable 類別中設定 `$theme` 屬性為寄送該 mailable 時應使用的主題名稱。

<a name="sending-mail"></a>
## 寄送郵件

要寄送訊息，請使用 `Mail` [Facade](/docs/{{version}}/facades) 上的 `to` 方法。`to` 方法接受電子郵件地址、使用者實例或使用者集合。如果您傳入一個物件或物件集合，mailer 將會自動在決定郵件收件者時使用其 `email` 與 `name` 屬性，因此請確保這些屬性存在於您的物件上。指定收件者後，您可以將 mailable 類別的實例傳遞給 `send` 方法：

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

寄送訊息時，您並不局限於僅指定 "to"（收件者）。您可以自由地透過將各自對應的方法鏈結在一起，來設定 "to"、"cc"（副本）與 "bcc"（密件副本）收件者：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->send(new OrderShipped($order));
```


<a name="looping-over-recipients"></a>
#### 迴圈處理收件者

有時，您可能需要透過巡覽收件者 / 電子郵件地址陣列，將 mailable 寄送給收件者清單。然而，由於 `to` 方法會將電子郵件地址附加到 mailable 的收件者清單中，因此迴圈中的每次反覆運算都會再傳送另一封電子郵件給之前的每一個收件者。因此，您應該始終為每個收件者重新建立 mailable 實例：

```php
foreach (['taylor@example.com', 'dries@example.com'] as $recipient) {
    Mail::to($recipient)->send(new OrderShipped($order));
}
```


<a name="sending-mail-via-a-specific-mailer"></a>
#### 透過特定 Mailer 寄送郵件

預設情況下，Laravel 會使用應用程式的 `mail` 設定檔中設定為 `default` 的 mailer 來寄送電子郵件。不過，您可以使用 `mailer` 方法來使用特定的 mailer 設定寄送訊息：

```php
Mail::mailer('postmark')
    ->to($request->user())
    ->send(new OrderShipped($order));
```


<a name="queueing-mail"></a>
### 佇列郵件


<a name="queueing-a-mail-message"></a>
#### 將郵件訊息加入佇列

由於寄送電子郵件訊息可能會對應用程式的回應時間產生負面影響，許多開發者選擇將電子郵件訊息加入佇列以在背景寄送。Laravel 使用其內建的[統一佇列 API](/docs/{{version}}/queues) 讓這件事變得簡單。若要將郵件訊息加入佇列，請在指定訊息的收件者後使用 `Mail` Facade 上的 `queue` 方法：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue(new OrderShipped($order));
```

此方法會自動處理將任務推送到佇列中，以便在背景寄送訊息。在使用此功能之前，您需要先[設定您的佇列](/docs/{{version}}/queues)。


<a name="delayed-message-queueing"></a>
#### 延遲郵件佇列

如果您希望延遲寄送佇列中的電子郵件訊息，您可以使用 `later` 方法。`later` 方法接受一個 `DateTime` 實例作為其第一個引數，用以指示應該何時寄送訊息：

```php
Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->later(now()->plus(minutes: 10), new OrderShipped($order));
```


<a name="pushing-to-specific-queues"></a>
#### 推送到特定佇列

由於所有使用 `make:mail` 命令建立的 mailable 類別都使用了 `Illuminate\Bus\Queueable` trait，您可以在任何 mailable 類別實例上呼叫 `onQueue` 與 `onConnection` 方法，這允許您為訊息指定連線與佇列名稱：

```php
$message = (new OrderShipped($order))
    ->onConnection('sqs')
    ->onQueue('emails');

Mail::to($request->user())
    ->cc($moreUsers)
    ->bcc($evenMoreUsers)
    ->queue($message);
```

或者，您可以在 mailable 類別上使用 `Connection` 與 `Queue` Attribute（屬性）來指定連線與佇列：

```php
use Illuminate\Queue\Attributes\Connection;
use Illuminate\Queue\Attributes\Queue;

#[Connection('sqs')]
#[Queue('emails')]
class OrderShipped extends Mailable
{
    // ...
}
```


<a name="queueing-by-default"></a>
#### 預設加入佇列

如果您有希望始終加入佇列的 mailable 類別，您可以在該類別上實作 `ShouldQueue` 契約(Contracts)。如此一來，即使您在寄件時呼叫 `send` 方法，由於該 mailable 實作了該契約，它依然會被加入佇列：

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class OrderShipped extends Mailable implements ShouldQueue
{
    // ...
}
```


<a name="queued-mailables-and-database-transactions"></a>
#### 佇列 Mailable 與資料庫交易

當佇列 mailable 在資料庫交易內被分發時，它們可能會在資料庫交易提交之前就被佇列處理。發生這種情況時，您在資料庫交易期間對 Model 或資料庫紀錄所做的任何更新可能尚未反映在資料庫中。此外，在交易內建立的任何 Model 或資料庫紀錄可能還不存在於資料庫中。如果您的 mailable 依賴這些 Model，則在處理寄送佇列 mailable 的任務時可能會發生意外錯誤。

如果佇列連線的 `after_commit` 設定選項設定為 `false`，您仍可以在寄送郵件訊息時透過呼叫 `afterCommit` 方法，來指定特定的佇列 mailable 應在所有開啟的資料庫交易都提交後才被分發：

```php
Mail::to($request->user())->send(
    (new OrderShipped($order))->afterCommit()
);
```

或者，您可以從 mailable 的建構子中呼叫 `afterCommit` 方法：

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
> 若要瞭解更多關於如何避開這些問題的資訊，請參閱有關[佇列任務與資料庫交易](/docs/{{version}}/queues#jobs-and-database-transactions)的說明文件。


<a name="queued-email-failures"></a>
#### 佇列電子郵件失敗處理

當佇列電子郵件失敗時，如果佇列 mailable 類別中定義了 `failed` 方法，該方法將會被呼叫。導致佇列電子郵件失敗的 `Throwable` 實例將會被傳遞給 `failed` 方法：

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

有時您可能希望在不發送郵件的情況下擷取 mailable 的 HTML 內容。若要做到這一點，您可以呼叫 mailable 的 `render` 方法。此方法會將解析後的 mailable HTML 內容作為字串傳回：

```php
use App\Mail\InvoicePaid;
use App\Models\Invoice;

$invoice = Invoice::find(1);

return (new InvoicePaid($invoice))->render();
```


<a name="previewing-mailables-in-the-browser"></a>
### 在瀏覽器中預覽 Mailable

當在設計 mailable 的模板時，像一般的 Blade 模板一樣在瀏覽器中快速預覽渲染後的 mailable 會非常方便。因此，Laravel 允許您直接從路由閉包或控制器傳回任何 mailable。當傳回 mailable 時，它將會被渲染並顯示在瀏覽器中，讓您能快速預覽其設計，而無需將其發送到真實的電子郵件地址：

```php
Route::get('/mailable', function () {
    $invoice = App\Models\Invoice::find(1);

    return new App\Mail\InvoicePaid($invoice);
});
```


<a name="localizing-mailables"></a>
## Mailable 在地化

Laravel 允許您以當前請求以外的語系發送 mailable，而且如果郵件進入佇列，它甚至會記住此語系。

為了達到這個目的，`Mail` Facade 提供了一個 `locale` 方法來設定所需的語言。當解析 mailable 的模板時，應用程式會切換到此語系，並在解析完成後還原回先前的語系：

```php
Mail::to($request->user())->locale('es')->send(
    new OrderShipped($order)
);
```


<a name="user-preferred-locales"></a>
#### 使用者偏好的語系

有時候，應用程式會儲存每個使用者偏好的語系。藉由在一或多個 Model 上實作 `HasLocalePreference` 契約(Contracts)，您可以指示 Laravel 在發送郵件時使用此儲存的語系：

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

一旦您實作了該介面，Laravel 在向該 Model 發送 mailable 及通知時，將會自動使用偏好的語系。因此，使用此介面時無需呼叫 `locale` 方法：

```php
Mail::to($request->user())->send(new OrderShipped($order));
```

<a name="testing-mailables"></a>
## 測試

<a name="testing-mailable-content"></a>
### 測試 Mailable 內容

Laravel 提供各種方法來檢查 mailable 的結構。此外，Laravel 還提供數個便利的方法，用來測試您的 mailable 是否包含您所預期的內容：

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

如同您所預期的，「HTML」斷言會驗證 mailable 的 HTML 版本是否包含指定的字串，而「text」斷言則會驗證 mailable 的純文字版本是否包含指定的字串。

<a name="testing-mailable-sending"></a>
### 測試 Mailable 寄送

我們建議將 Mailable 的內容測試與斷言特定的 Mailable 已「寄送」給特定使用者的測試分開進行。通常，Mailable 的內容與您要測試的程式碼無關，只需單純斷言 Laravel 已接獲指示寄送特定的 Mailable 即可。

您可以使用 `Mail` Facade 的 `fake` 方法來防止郵件被實際寄出。呼叫 `Mail` Facade 的 `fake` 方法後，即可斷言已指示將哪些 Mailable 寄送給使用者，甚至可以檢查 Mailable 所接收到的資料：

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

如果您將 Mailable 加入佇列並於背景傳送，您應該使用 `assertQueued` 方法來替代 `assertSent`：

```php
Mail::assertQueued(OrderShipped::class);
Mail::assertNotQueued(OrderShipped::class);
Mail::assertNothingQueued();
Mail::assertQueuedCount(3);
```

您也可以使用 `assertOutgoingCount` 方法來斷言已寄出或佇列中的 Mailable 總數量：

```php
Mail::assertOutgoingCount(3);
```

您可以傳遞閉包至 `assertSent`、`assertNotSent`、`assertQueued` 或 `assertNotQueued` 方法，以斷言寄出的 Mailable 是否通過指定的「真值測試 (truth test)」。若至少有一個寄出的 Mailable 通過指定的真值測試，則斷言將會成功：

```php
Mail::assertSent(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

呼叫 `Mail` Facade 的斷言方法時，傳入閉包接收到的 Mailable 實例會提供一些實用的方法來檢驗該 Mailable：

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

Mailable 實例還包含幾個用來檢驗 Mailable 上附件的實用方法：

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

您可能已經注意到有兩種方法可用於斷言郵件未被寄出：`assertNotSent` 和 `assertNotQueued`。有時您可能希望斷言沒有任何郵件被寄出**或**加入佇列。為達此目的，您可以使用 `assertNothingOutgoing` 和 `assertNotOutgoing` 方法：

```php
Mail::assertNothingOutgoing();

Mail::assertNotOutgoing(function (OrderShipped $mail) use ($order) {
    return $mail->order->id === $order->id;
});
```

<a name="mail-and-local-development"></a>
## 郵件與本機開發

開發會寄送電子郵件的應用程式時，你可能不會希望真的將郵件寄給實際的電子郵件地址。Laravel 提供多種方法能在本機開發期間「停用」真實郵件的寄送。


<a name="log-driver"></a>
#### Log 驅動器

`log` 郵件驅動器不會真實寄出郵件，而是將所有電子郵件訊息寫入日誌檔中供你檢查。通常，這個驅動器僅用於本機開發。關於按環境設定應用程式的更多資訊，請參考[設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="mailtrap"></a>
#### HELO / Mailtrap / Mailpit

或者，你也可以使用像 [HELO](https://usehelo.com) 或 [Mailtrap](https://mailtrap.io) 這類服務配合 `smtp` 驅動器，將電子郵件訊息寄送到「虛擬」信箱中，並能在真實的郵件用戶端中查看它們。這種做法的好處是能讓你直接在 Mailtrap 的訊息檢視器中檢查最終產生的電子郵件。

如果你使用的是 [Laravel Sail](/docs/{{version}}/sail)，則可以使用 [Mailpit](https://github.com/axllent/mailpit) 來預覽訊息。當 Sail 正在運行時，你可以透過 `http://localhost:8025` 存取 Mailpit 介面。


<a name="using-a-global-to-address"></a>
#### 使用全域 `to` 地址

最後，你可以透過呼叫 `Mail` Facade 提供的 `alwaysTo` 方法來指定全域的 "to" 地址。通常，這個方法應該在應用程式的某個服務提供者(Service Providers)中的 `boot` 方法內呼叫：

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

當使用 `alwaysTo` 方法時，郵件訊息中的任何額外 "cc" 或 "bcc" 地址都將被移除。


<a name="events"></a>
## 事件

Laravel 在寄送郵件訊息時會發送兩個事件。`MessageSending` 事件會在訊息寄出前發送，而 `MessageSent` 事件則會在訊息寄出後發送。請記住，這些事件是在郵件被*寄送*時發送，而不是在被放進佇列時發送。你可以在應用程式中為這些事件建立[事件監聽器](/docs/{{version}}/events)：

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

Laravel 內建了多種郵件傳輸器；然而，你可能希望撰寫自己的傳輸器，以便透過 Laravel 開箱即用尚未支援的其他服務來傳遞電子郵件。若要開始，請定義一個繼承 `Symfony\Component\Mailer\Transport\AbstractTransport` 類別的類別。然後，在你的傳輸器上實作 `doSend` 與 `__toString` 方法：

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

當你定義好自訂傳輸器後，可以透過 `Mail` Facade 提供的 `extend` 方法來註冊它。通常，這應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中進行。傳給 `extend` 方法的閉包會接收一個 `$config` 引數。這個引數將包含應用程式 `config/mail.php` 設定檔中為該郵件寄送者所定義的設定陣列：

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

當你的自訂傳輸器被定義並註冊後，你可以在應用程式的 `config/mail.php` 設定檔中建立一個使用該新傳輸器的 mailer 定義：

```php
'mailchimp' => [
    'transport' => 'mailchimp',
    'key' => env('MAILCHIMP_API_KEY'),
    // ...
],
```


<a name="additional-symfony-transports"></a>
### 額外的 Symfony 傳輸器

Laravel 支援部分現有由 Symfony 維護的郵件傳輸器，例如 Mailgun 和 Postmark。然而，你可能希望擴充 Laravel 以支援其他 Symfony 維護的傳輸器。你可以透過 Composer 載入必要的 Symfony mailer 套件，並將該傳輸器註冊至 Laravel。例如，你可以安裝並註冊 "Brevo"（前身為 "Sendinblue"）Symfony mailer：

```shell
composer require symfony/brevo-mailer symfony/http-client
```

安裝 Brevo mailer 套件後，你可以在應用程式的 `services` 設定檔中為你的 Brevo API 憑證新增一個項目：

```php
'brevo' => [
    'key' => env('BREVO_API_KEY'),
],
```

接下來，你可以使用 `Mail` Facade 的 `extend` 方法將傳輸器註冊到 Laravel 中。通常，這應該在服務提供者(Service Providers)的 `boot` 方法中進行：

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

當你的傳輸器註冊完成後，你可以在應用程式的 `config/mail.php` 設定檔中建立一個使用該新傳輸器的 mailer 定義：

```php
'brevo' => [
    'transport' => 'brevo',
    // ...
],
```