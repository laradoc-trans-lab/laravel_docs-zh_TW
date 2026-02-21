# 檔案儲存

- [介紹](#introduction)
- [配置](#configuration)
    - [本地驅動](#the-local-driver)
    - [公共磁碟](#the-public-disk)
    - [驅動需求](#driver-prerequisites)
    - [限定範圍與唯讀檔案系統](#scoped-and-read-only-filesystems)
    - [與 Amazon S3 相容的檔案系統](#amazon-s3-compatible-filesystems)
- [取得磁碟執行個體](#obtaining-disk-instances)
    - [隨選磁碟](#on-demand-disks)
- [擷取檔案](#retrieving-files)
    - [下載檔案](#downloading-files)
    - [檔案 URL](#file-urls)
    - [暫時性 URL](#temporary-urls)
    - [檔案中繼資料](#file-metadata)
- [儲存檔案](#storing-files)
    - [在檔案開頭與結尾附加內容](#prepending-appending-to-files)
    - [複製與移動檔案](#copying-moving-files)
    - [自動串流](#automatic-streaming)
    - [檔案上傳](#file-uploads)
    - [檔案可見度](#file-visibility)
- [刪除檔案](#deleting-files)
- [目錄](#directories)
- [測試](#testing)
- [自定義檔案系統](#custom-filesystems)

<a name="introduction"></a>
## 介紹

Laravel 藉由 Frank de Jonge 所開發的出色 [Flysystem](https://github.com/thephpleague/flysystem) PHP 套件，提供了一個強大的檔案系統抽象層。Laravel 與 Flysystem 的整合提供了簡單的驅動程式，以便與本地檔案系統、SFTP 及 Amazon S3 搭配使用。更棒的是，由於每個系統的 API 均保持一致，在本地開發環境與正式環境伺服器之間切換這些儲存選項變得極其簡單。

<a name="configuration"></a>
## 配置

Laravel 的檔案系統配置檔案位於 `config/filesystems.php`。在此檔案中，您可以配置所有的檔案系統「磁碟 (disks)」。每個磁碟都代表一個特定的儲存驅動與儲存位置。配置檔案中包含了每個支援驅動的範例配置，因此您可以修改配置來反映您的儲存偏好與憑證。

`local` 驅動與儲存在運行 Laravel 應用程式的伺服器上的本地檔案進行互動，而 `sftp` 儲存驅動則用於基於 SSH 金鑰的 FTP。`s3` 驅動則用於寫入 Amazon 的 S3 雲端儲存服務。

> [!NOTE]
> 您可以配置任意數量的磁碟，甚至可以有多個使用相同驅動的磁碟。


<a name="the-local-driver"></a>
### 本地驅動

當使用 `local` 驅動時，所有的檔案操作都是相對於 `filesystems` 配置檔案中定義的 `root` 目錄。預設情況下，此值設定為 `storage/app/private` 目錄。因此，以下方法將會寫入到 `storage/app/private/example.txt`：

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('example.txt', 'Contents');
```


<a name="the-public-disk"></a>
### 公共磁碟

包含在您應用程式 `filesystems` 配置檔案中的 `public` 磁碟是用於存放將被公開存取的檔案。預設情況下，`public` 磁碟使用 `local` 驅動，並將其檔案儲存在 `storage/app/public`。

如果您的 `public` 磁碟使用 `local` 驅動，且您希望讓這些檔案可以透過網頁存取，則應建立一個從來源目錄 `storage/app/public` 到目標目錄 `public/storage` 的符號連結：

若要建立符號連結，您可以使用 `storage:link` Artisan 指令：

```shell
php artisan storage:link
```

一旦檔案被儲存且符號連結已建立，您就可以使用 `asset` 輔助函式來建立指向檔案的 URL：

```php
echo asset('storage/file.txt');
```

您可以在 `filesystems` 配置檔案中配置額外的符號連結。當您執行 `storage:link` 指令時，將會建立每個配置的連結：

```php
'links' => [
    public_path('storage') => storage_path('app/public'),
    public_path('images') => storage_path('app/images'),
],
```

`storage:unlink` 指令可用於銷毀您配置的符號連結：

```shell
php artisan storage:unlink
```


<a name="driver-prerequisites"></a>
### 驅動需求


<a name="s3-driver-configuration"></a>
#### S3 驅動配置

在使用 S3 驅動之前，您需要透過 Composer 套件管理器安裝 Flysystem S3 套件：

```shell
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

S3 磁碟配置陣列位於您的 `config/filesystems.php` 配置檔案中。通常，您應該使用以下環境變數來配置您的 S3 資訊與憑證，這些變數會被 `config/filesystems.php` 配置檔案所引用：

```ini
AWS_ACCESS_KEY_ID=<your-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=<your-bucket-name>
AWS_USE_PATH_STYLE_ENDPOINT=false
```

為了方便起見，這些環境變數符合 AWS CLI 所使用的命名慣例。


<a name="ftp-driver-configuration"></a>
#### FTP 驅動配置

在使用 FTP 驅動之前，您需要透過 Composer 套件管理器安裝 Flysystem FTP 套件：

```shell
composer require league/flysystem-ftp "^3.0"
```

Laravel 的 Flysystem 整合與 FTP 配合得非常好；然而，框架預設的 `config/filesystems.php` 配置檔案中並未包含範例配置。如果您需要配置 FTP 檔案系統，可以使用下方的配置範例：

```php
'ftp' => [
    'driver' => 'ftp',
    'host' => env('FTP_HOST'),
    'username' => env('FTP_USERNAME'),
    'password' => env('FTP_PASSWORD'),

    // Optional FTP Settings...
    // 'port' => env('FTP_PORT', 21),
    // 'root' => env('FTP_ROOT'),
    // 'passive' => true,
    // 'ssl' => true,
    // 'timeout' => 30,
],
```


<a name="sftp-driver-configuration"></a>
#### SFTP 驅動配置

在使用 SFTP 驅動之前，您需要透過 Composer 套件管理器安裝 Flysystem SFTP 套件：

```shell
composer require league/flysystem-sftp-v3 "^3.0"
```

Laravel 的 Flysystem 整合與 SFTP 配合得非常好；然而，框架預設的 `config/filesystems.php` 配置檔案中並未包含範例配置。如果您需要配置 SFTP 檔案系統，可以使用下方的配置範例：

```php
'sftp' => [
    'driver' => 'sftp',
    'host' => env('SFTP_HOST'),

    // Settings for basic authentication...
    'username' => env('SFTP_USERNAME'),
    'password' => env('SFTP_PASSWORD'),

    // Settings for SSH key-based authentication with encryption password...
    'privateKey' => env('SFTP_PRIVATE_KEY'),
    'passphrase' => env('SFTP_PASSPHRASE'),

    // Settings for file / directory permissions...
    'visibility' => 'private', // `private` = 0600, `public` = 0644
    'directory_visibility' => 'private', // `private` = 0700, `public` = 0755

    // Optional SFTP Settings...
    // 'hostFingerprint' => env('SFTP_HOST_FINGERPRINT'),
    // 'maxTries' => 4,
    // 'passphrase' => env('SFTP_PASSPHRASE'),
    // 'port' => env('SFTP_PORT', 22),
    // 'root' => env('SFTP_ROOT', ''),
    // 'timeout' => 30,
    // 'useAgent' => true,
],
```


<a name="scoped-and-read-only-filesystems"></a>
### 限定範圍與唯讀檔案系統

範圍限定磁碟允許您定義一個檔案系統，其中所有的路徑都會自動加上給定的路徑前綴。在建立範圍限定的檔案系統磁碟之前，您需要透過 Composer 套件管理器安裝額外的 Flysystem 套件：

```shell
composer require league/flysystem-path-prefixing "^3.0"
```

您可以藉由定義一個使用 `scoped` 驅動的磁碟，來為任何現有的檔案系統磁碟建立一個路徑範圍限定的執行個體。例如，您可以建立一個磁碟，將現有的 `s3` 磁碟限定在特定的路徑前綴，接著使用該範圍限定磁碟進行的每個檔案操作都會使用該指定的前綴：

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

「唯讀」磁碟允許您建立不允許寫入操作的檔案系統磁碟。在使用 `read-only` 配置選項之前，您需要透過 Composer 套件管理器安裝額外的 Flysystem 套件：

```shell
composer require league/flysystem-read-only "^3.0"
```

接著，您可以在一個或多個磁碟配置陣列中加入 `read-only` 配置選項：

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```


<a name="amazon-s3-compatible-filesystems"></a>
### 與 Amazon S3 相容的檔案系統

預設情況下，您應用程式的 `filesystems` 配置檔案包含一個 `s3` 磁碟的配置。除了使用此磁碟與 [Amazon S3](https://aws.amazon.com/s3/) 互動外，您還可以使用它與任何相容於 S3 的檔案儲存服務互動，例如 [RustFS](https://github.com/rustfs/rustfs)、[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)、[Vultr Object Storage](https://www.vultr.com/products/object-storage/)、[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) 或 [Hetzner Cloud Storage](https://www.hetzner.com/storage/object-storage/)。

通常，在更新磁碟憑證以符合您計劃使用的服務憑證後，您只需要更新 `endpoint` 配置選項的值。此選項的值通常透過 `AWS_ENDPOINT` 環境變數來定義：

```php
'endpoint' => env('AWS_ENDPOINT', 'https://rustfs:9000'),
```

<a name="obtaining-disk-instances"></a>
## 取得磁碟執行個體

`Storage` Facade 可用於與任何已設定的磁碟進行互動。例如，您可以使用 Facade 上的 `put` 方法將大頭貼儲存到預設磁碟。如果您在呼叫 `Storage` Facade 的方法時沒有先呼叫 `disk` 方法，該方法將會自動被傳遞給預設磁碟：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('avatars/1', $content);
```

如果您的應用程式與多個磁碟互動，您可以使用 `Storage` Facade 上的 `disk` 方法來處理特定磁碟上的檔案：

```php
Storage::disk('s3')->put('avatars/1', $content);
```

<a name="on-demand-disks"></a>
### 隨選磁碟

有時您可能希望在執行階段使用指定的設定來建立磁碟，而該設定實際上並未出現在應用程式的 `filesystems` 設定檔中。若要達成此目的，您可以將設定陣列傳遞給 `Storage` Facade 的 `build` 方法：

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);

$disk->put('image.jpg', $content);
```

<a name="retrieving-files"></a>
## 擷取檔案

`get` 方法可用於擷取檔案內容。該方法將回傳檔案的原始字串內容。請記住，所有檔案路徑都應相對於磁碟的「根 (root)」位置指定：

```php
$contents = Storage::get('file.jpg');
```

如果您擷取的檔案包含 JSON，您可以使用 `json` 方法來擷取檔案並對其內容進行解碼：

```php
$orders = Storage::json('orders.json');
```

`exists` 方法可用於判斷磁碟上是否存在某個檔案：

```php
if (Storage::disk('s3')->exists('file.jpg')) {
    // ...
}
```

`missing` 方法可用於判斷磁碟上是否缺少某個檔案：

```php
if (Storage::disk('s3')->missing('file.jpg')) {
    // ...
}
```


<a name="downloading-files"></a>
### 下載檔案

`download` 方法可用於產生一個回應，強制使用者的瀏覽器下載指定路徑的檔案。`download` 方法接受檔名作為其第二個參數，這將決定下載檔案的使用者所看到的檔名。最後，您可以將 HTTP 標頭陣列作為其第三個參數進行傳遞：

```php
return Storage::download('file.jpg');

return Storage::download('file.jpg', $name, $headers);
```


<a name="file-urls"></a>
### 檔案 URL

您可以使用 `url` 方法來取得指定檔案的 URL。如果您使用的是 `local` 驅動，這通常只是在指定路徑前加上 `/storage` 並回傳該檔案的相對 URL。如果您使用的是 `s3` 驅動，則會回傳完整的遠端 URL：

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::url('file.jpg');
```

使用 `local` 驅動時，所有應公開存取的檔案都應放置在 `storage/app/public` 目錄中。此外，您應該在 `public/storage` [建立符號連結](#the-public-disk)，指向 `storage/app/public` 目錄。

> [!WARNING]
> 使用 `local` 驅動時，`url` 的回傳值未經過 URL 編碼。因此，我們建議始終使用能產生有效 URL 的名稱來儲存檔案。


<a name="url-host-customization"></a>
#### URL 主機自定義

如果您想修改使用 `Storage` Facade 產生的 URL 的主機，可以在磁碟的配置陣列中新增或更改 `url` 選項：

```php
'public' => [
    'driver' => 'local',
    'root' => storage_path('app/public'),
    'url' => env('APP_URL').'/storage',
    'visibility' => 'public',
    'throw' => false,
],
```


<a name="temporary-urls"></a>
### 暫時性 URL

使用 `temporaryUrl` 方法，您可以為使用 `local` 和 `s3` 驅動儲存的檔案建立暫時性 URL。此方法接受路徑與一個指定 URL 何時過期的 `DateTime` 執行個體：

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::temporaryUrl(
    'file.jpg', now()->plus(minutes: 5)
);
```


<a name="enabling-local-temporary-urls"></a>
#### 啟用本地暫時性 URL

如果您在 `local` 驅動引入暫時性 URL 支援之前就開始開發應用程式，您可能需要啟用本地暫時性 URL。若要執行此操作，請在 `config/filesystems.php` 配置檔案中，將 `serve` 選項新增到您的 `local` 磁碟配置陣列中：

```php
'local' => [
    'driver' => 'local',
    'root' => storage_path('app/private'),
    'serve' => true, // [tl! add]
    'throw' => false,
],
```


<a name="s3-request-parameters"></a>
#### S3 請求參數

如果您需要指定額外的 [S3 請求參數](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests)，可以將請求參數陣列作為第三個參數傳遞給 `temporaryUrl` 方法：

```php
$url = Storage::temporaryUrl(
    'file.jpg',
    now()->plus(minutes: 5),
    [
        'ResponseContentType' => 'application/octet-stream',
        'ResponseContentDisposition' => 'attachment; filename=file2.jpg',
    ]
);
```


<a name="customizing-temporary-urls"></a>
#### 自定義暫時性 URL

如果您需要自定義特定儲存磁碟建立暫時性 URL 的方式，可以使用 `buildTemporaryUrlsUsing` 方法。例如，如果您有一個控制器允許下載透過通常不支援暫時性 URL 的磁碟儲存的檔案，這將非常有用。通常，應從服務提供者的 `boot` 方法中呼叫此方法：

```php
<?php

namespace App\Providers;

use DateTime;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\URL;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Storage::disk('local')->buildTemporaryUrlsUsing(
            function (string $path, DateTime $expiration, array $options) {
                return URL::temporarySignedRoute(
                    'files.download',
                    $expiration,
                    array_merge($options, ['path' => $path])
                );
            }
        );
    }
}
```


<a name="temporary-upload-urls"></a>
#### 暫時性上傳 URL

> [!WARNING]
> 產生暫時性上傳 URL 的功能僅由 `s3` 驅動支援。

如果您需要產生一個可用於直接從用戶端應用程式上傳檔案的暫時性 URL，可以使用 `temporaryUploadUrl` 方法。此方法接受路徑與一個指定 URL 何時過期的 `DateTime` 執行個體。`temporaryUploadUrl` 方法會回傳一個關聯陣列，該陣列可以被解構為上傳 URL 以及應包含在該上傳請求中的標頭：

```php
use Illuminate\Support\Facades\Storage;

['url' => $url, 'headers' => $headers] = Storage::temporaryUploadUrl(
    'file.jpg', now()->plus(minutes: 5)
);
```

此方法主要用於無伺服器 (serverless) 環境中，這類環境要求用戶端應用程式直接將檔案上傳到雲端儲存系統，例如 Amazon S3。


<a name="file-metadata"></a>
### 檔案中繼資料

除了讀取和寫入檔案外，Laravel 還可以提供檔案本身的資訊。例如，`size` 方法可用於取得檔案的大小（以位元組為單位）：

```php
use Illuminate\Support\Facades\Storage;

$size = Storage::size('file.jpg');
```

`lastModified` 方法回傳檔案最後一次修改時的 UNIX 時間戳記：

```php
$time = Storage::lastModified('file.jpg');
```

指定檔案的 MIME 類型可以透過 `mimeType` 方法取得：

```php
$mime = Storage::mimeType('file.jpg');
```


<a name="file-paths"></a>
#### 檔案路徑

您可以使用 `path` 方法來取得指定檔案的路徑。如果您使用的是 `local` 驅動，這將回傳檔案的絕對路徑。如果您使用的是 `s3` 驅動，此方法將回傳 S3 bucket 中檔案的相對路徑：

```php
use Illuminate\Support\Facades\Storage;

$path = Storage::path('file.jpg');
```

<a name="storing-files"></a>
## 儲存檔案

`put` 方法可用於在磁碟上儲存檔案內容。您也可以將 PHP `resource` 傳遞給 `put` 方法，這將使用 Flysystem 底層的串流支援。請記住，所有檔案路徑都應相對於為磁碟配置的「root」位置：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents);

Storage::put('file.jpg', $resource);
```

<a name="failed-writes"></a>
#### 寫入失敗

如果 `put` 方法 (或其他「寫入」操作) 無法將檔案寫入磁碟，將會回傳 `false`：

```php
if (! Storage::put('file.jpg', $contents)) {
    // The file could not be written to disk...
}
```

如果您願意，可以在檔案系統磁碟的配置陣列中定義 `throw` 選項。當此選項定義為 `true` 時，當寫入操作失敗時，`put` 等「寫入」方法將拋出 `League\Flysystem\UnableToWriteFile` 的執行個體：

```php
'public' => [
    'driver' => 'local',
    // ...
    'throw' => true,
],
```

<a name="prepending-appending-to-files"></a>
### 在檔案開頭與結尾附加內容

`prepend` 與 `append` 方法允許您在檔案的開頭或結尾寫入內容：

```php
Storage::prepend('file.log', 'Prepended Text');

Storage::append('file.log', 'Appended Text');
```

<a name="copying-moving-files"></a>
### 複製與移動檔案

`copy` 方法可用於將現有檔案複製到磁碟上的新位置，而 `move` 方法可用於重新命名現有檔案或將其移動到新位置：

```php
Storage::copy('old/file.jpg', 'new/file.jpg');

Storage::move('old/file.jpg', 'new/file.jpg');
```

<a name="automatic-streaming"></a>
### 自動串流

將檔案串流到儲存空間可以顯著減少記憶體使用量。如果您希望 Laravel 自動管理將指定檔案串流到您的儲存位置，可以使用 `putFile` 或 `putFileAs` 方法。此方法接受 `Illuminate\Http\File` 或 `Illuminate\Http\UploadedFile` 執行個體，並會自動將檔案串流到您所需的位置：

```php
use Illuminate\Http\File;
use Illuminate\Support\Facades\Storage;

// Automatically generate a unique ID for filename...
$path = Storage::putFile('photos', new File('/path/to/photo'));

// Manually specify a filename...
$path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');
```

關於 `putFile` 方法，有幾點重要事項需要注意。請注意，我們只指定了目錄名稱而不是檔案名稱。預設情況下，`putFile` 方法將產生一個唯一的 ID 作為檔案名稱。檔案的副檔名將透過檢查檔案的 MIME 類型來確定。`putFile` 方法將回傳檔案路徑，因此您可以將包含產生的檔案名稱的路徑儲存在資料庫中。

`putFile` 和 `putFileAs` 方法也接受一個參數來指定儲存檔案的「可見度 (visibility)」。如果您將檔案儲存在 Amazon S3 等雲端磁碟上，並希望檔案可以透過產生的 URL 公開存取，這會非常有用：

```php
Storage::putFile('photos', new File('/path/to/photo'), 'public');
```

<a name="file-uploads"></a>
### 檔案上傳

在 Web 應用程式中，儲存檔案最常見的情境之一是儲存使用者上傳的檔案，例如照片和文件。Laravel 透過在上傳檔案執行個體上使用 `store` 方法，可以非常輕鬆地儲存上傳的檔案。請使用您希望儲存上傳檔案的路徑來呼叫 `store` 方法：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class UserAvatarController extends Controller
{
    /**
     * Update the avatar for the user.
     */
    public function update(Request $request): string
    {
        $path = $request->file('avatar')->store('avatars');

        return $path;
    }
}
```

關於這個範例，有幾點重要事項需要注意。請注意，我們只指定了目錄名稱，而不是檔案名稱。預設情況下，`store` 方法將產生一個唯一的 ID 作為檔案名稱。檔案的副檔名將透過檢查檔案的 MIME 類型來確定。`store` 方法將回傳檔案路徑，因此您可以將包含產生的檔案名稱的路徑儲存在資料庫中。

您也可以在 `Storage` facade 上呼叫 `putFile` 方法，來執行與上述範例相同的檔案儲存操作：

```php
$path = Storage::putFile('avatars', $request->file('avatar'));
```

<a name="specifying-a-file-name"></a>
#### 指定檔案名稱

如果您不希望系統自動為儲存的檔案分配檔案名稱，可以使用 `storeAs` 方法，該方法接收路徑、檔案名稱和 (選填的) 磁碟作為參數：

```php
$path = $request->file('avatar')->storeAs(
    'avatars', $request->user()->id
);
```

您也可以在 `Storage` facade 上使用 `putFileAs` 方法，這將執行與上述範例相同的檔案儲存操作：

```php
$path = Storage::putFileAs(
    'avatars', $request->file('avatar'), $request->user()->id
);
```

> [!WARNING]
> 不可列印與無效的 Unicode 字元將自動從檔案路徑中移除。因此，在將檔案路徑傳遞給 Laravel 的檔案儲存方法之前，您可能需要先過濾您的檔案路徑。檔案路徑使用 `League\Flysystem\WhitespacePathNormalizer::normalizePath` 方法進行標準化。

<a name="specifying-a-disk"></a>
#### 指定磁碟

預設情況下，上傳檔案的 `store` 方法會使用預設磁碟。如果您想指定另一個磁碟，請將磁碟名稱作為 `store` 方法的第二個參數傳遞：

```php
$path = $request->file('avatar')->store(
    'avatars/'.$request->user()->id, 's3'
);
```

如果您使用的是 `storeAs` 方法，可以將磁碟名稱作為方法的第三個參數傳遞：

```php
$path = $request->file('avatar')->storeAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="other-uploaded-file-information"></a>
#### 其他上傳檔案資訊

如果您想取得上傳檔案的原始名稱與副檔名，可以使用 `getClientOriginalName` 與 `getClientOriginalExtension` 方法：

```php
$file = $request->file('avatar');

$name = $file->getClientOriginalName();
$extension = $file->getClientOriginalExtension();
```

然而，請記住 `getClientOriginalName` 與 `getClientOriginalExtension` 方法被認為是不安全的，因為檔案名稱和副檔名可能會被惡意使用者篡改。基於這個原因，您通常應該優先使用 `hashName` 與 `extension` 方法來取得指定上傳檔案的名稱與副檔名：

```php
$file = $request->file('avatar');

$name = $file->hashName(); // Generate a unique, random name...
$extension = $file->extension(); // Determine the file's extension based on the file's MIME type...
```

<a name="file-visibility"></a>
### 檔案可見度

在 Laravel 的 Flysystem 整合中，「可見度 (Visibility)」是跨多個平台的檔案權限抽象化。檔案可以被宣告為 `public` 或 `private`。當檔案被宣告為 `public` 時，表示該檔案通常應該讓其他人可以存取。例如，在使用 S3 驅動時，您可以取得 `public` 檔案的 URL。

您可以在透過 `put` 方法寫入檔案時設定可見度：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents, 'public');
```

如果檔案已經被儲存，可以透過 `getVisibility` 與 `setVisibility` 方法取得與設定其可見度：

```php
$visibility = Storage::getVisibility('file.jpg');

Storage::setVisibility('file.jpg', 'public');
```

當處理上傳的檔案時，您可以使用 `storePublicly` 與 `storePubliclyAs` 方法，以 `public` 可見度來儲存上傳的檔案：

```php
$path = $request->file('avatar')->storePublicly('avatars', 's3');

$path = $request->file('avatar')->storePubliclyAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="local-files-and-visibility"></a>
#### 本地檔案與可見度

當使用 `local` 驅動時，`public` [可見度](#file-visibility) 會轉換為目錄的 `0755` 權限與檔案的 `0644` 權限。您可以在應用程式的 `filesystems` 設定檔中修改權限映射 (Mapping)：

```php
'local' => [
    'driver' => 'local',
    'root' => storage_path('app'),
    'permissions' => [
        'file' => [
            'public' => 0644,
            'private' => 0600,
        ],
        'dir' => [
            'public' => 0755,
            'private' => 0700,
        ],
    ],
    'throw' => false,
],
```

<a name="deleting-files"></a>
## 刪除檔案

`delete` 方法接受單個檔名或一個包含多個檔名的陣列來進行刪除：

```php
use Illuminate\Support\Facades\Storage;

Storage::delete('file.jpg');

Storage::delete(['file.jpg', 'file2.jpg']);
```

如有必要，您可以指定要從哪個磁碟刪除檔案：

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('s3')->delete('path/file.jpg');
```


<a name="directories"></a>
## 目錄


<a name="get-all-files-within-a-directory"></a>
#### 取得目錄內的所有檔案

`files` 方法會回傳一個包含指定目錄內所有檔案的陣列。如果您想要取得指定目錄內的所有檔案（包含子目錄），可以使用 `allFiles` 方法：

```php
use Illuminate\Support\Facades\Storage;

$files = Storage::files($directory);

$files = Storage::allFiles($directory);
```


<a name="get-all-directories-within-a-directory"></a>
#### 取得目錄內的所有目錄

`directories` 方法會回傳一個包含指定目錄內所有目錄的陣列。如果您想要取得指定目錄內的所有目錄（包含子目錄），可以使用 `allDirectories` 方法：

```php
$directories = Storage::directories($directory);

$directories = Storage::allDirectories($directory);
```


<a name="create-a-directory"></a>
#### 建立目錄

`makeDirectory` 方法會建立指定的目錄，包含任何所需的子目錄：

```php
Storage::makeDirectory($directory);
```


<a name="delete-a-directory"></a>
#### 刪除目錄

最後，`deleteDirectory` 方法可用於移除目錄及其所有檔案：

```php
Storage::deleteDirectory($directory);
```


<a name="testing"></a>
## 測試

`Storage` Facade 的 `fake` 方法讓您可以輕鬆地生成一個虛擬磁碟，結合 `Illuminate\Http\UploadedFile` 類別的檔案生成工具，大大簡化了檔案上傳的測試。例如：

```php tab=Pest
<?php

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

test('albums can be uploaded', function () {
    Storage::fake('photos');

    $response = $this->json('POST', '/photos', [
        UploadedFile::fake()->image('photo1.jpg'),
        UploadedFile::fake()->image('photo2.jpg')
    ]);

    // Assert one or more files were stored...
    Storage::disk('photos')->assertExists('photo1.jpg');
    Storage::disk('photos')->assertExists(['photo1.jpg', 'photo2.jpg']);

    // Assert one or more files were not stored...
    Storage::disk('photos')->assertMissing('missing.jpg');
    Storage::disk('photos')->assertMissing(['missing.jpg', 'non-existing.jpg']);

    // Assert that the number of files in a given directory matches the expected count...
    Storage::disk('photos')->assertCount('/wallpapers', 2);

    // Assert that a given directory is empty...
    Storage::disk('photos')->assertDirectoryEmpty('/wallpapers');
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_albums_can_be_uploaded(): void
    {
        Storage::fake('photos');

        $response = $this->json('POST', '/photos', [
            UploadedFile::fake()->image('photo1.jpg'),
            UploadedFile::fake()->image('photo2.jpg')
        ]);

        // Assert one or more files were stored...
        Storage::disk('photos')->assertExists('photo1.jpg');
        Storage::disk('photos')->assertExists(['photo1.jpg', 'photo2.jpg']);

        // Assert one or more files were not stored...
        Storage::disk('photos')->assertMissing('missing.jpg');
        Storage::disk('photos')->assertMissing(['missing.jpg', 'non-existing.jpg']);

        // Assert that the number of files in a given directory matches the expected count...
        Storage::disk('photos')->assertCount('/wallpapers', 2);

        // Assert that a given directory is empty...
        Storage::disk('photos')->assertDirectoryEmpty('/wallpapers');
    }
}
```

預設情況下，`fake` 方法會刪除其暫存目錄中的所有檔案。如果您想保留這些檔案，可以改用 "persistentFake" 方法。有關測試檔案上傳的更多資訊，可以參考 [HTTP 測試文件的檔案上傳資訊](/docs/{{version}}/http-tests#testing-file-uploads)。

> [!WARNING]
> `image` 方法需要 [GD 擴充功能](https://www.php.net/manual/en/book.image.php)。


<a name="custom-filesystems"></a>
## 自定義檔案系統

Laravel 的 Flysystem 整合內建支援多種「驅動」；然而，Flysystem 並不限於這些，它還有許多其他儲存系統的轉接器 (Adapter)。如果您想在 Laravel 應用程式中使用這些額外的轉接器，可以建立自定義驅動。

為了定義自定義檔案系統，您需要一個 Flysystem 轉接器。讓我們將社群維護的 Dropbox 轉接器新增到專案中：

```shell
composer require spatie/flysystem-dropbox
```

接下來，您可以在應用程式的其中一個 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中註冊該驅動。要完成此操作，您應該使用 `Storage` Facade 的 `extend` 方法：

```php
<?php

namespace App\Providers;

use Illuminate\Contracts\Foundation\Application;
use Illuminate\Filesystem\FilesystemAdapter;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\ServiceProvider;
use League\Flysystem\Filesystem;
use Spatie\Dropbox\Client as DropboxClient;
use Spatie\FlysystemDropbox\DropboxAdapter;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Storage::extend('dropbox', function (Application $app, array $config) {
            $adapter = new DropboxAdapter(new DropboxClient(
                $config['authorization_token']
            ));

            return new FilesystemAdapter(
                new Filesystem($adapter, $config),
                $adapter,
                $config
            );
        });
    }
}
```

`extend` 方法的第一個參數是驅動的名稱，第二個參數是一個接收 `$app` 與 `$config` 變數的 Closure。該 Closure 必須回傳一個 `Illuminate\Filesystem\FilesystemAdapter` 的執行個體。`$config` 變數包含在 `config/filesystems.php` 中為指定磁碟定義的值。

一旦您建立並註冊了擴充功能的服務提供者，就可以在 `config/filesystems.php` 設定檔中使用 `dropbox` 驅動。