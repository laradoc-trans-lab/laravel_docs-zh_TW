# 檔案儲存

- [簡介](#introduction)
- [設定](#configuration)
    - [本地驅動](#the-local-driver)
    - [Public 磁碟](#the-public-disk)
    - [驅動器先決條件](#driver-prerequisites)
    - [作用域與唯讀檔案系統](#scoped-and-read-only-filesystems)
    - [相容 Amazon S3 的檔案系統](#amazon-s3-compatible-filesystems)
- [取得磁碟實例](#obtaining-disk-instances)
    - [隨選磁碟](#on-demand-disks)
- [檔案取回](#retrieving-files)
    - [下載檔案](#downloading-files)
    - [檔案 URL](#file-urls)
    - [暫時 URL](#temporary-urls)
    - [檔案中繼資料](#file-metadata)
- [檔案儲存](#storing-files)
    - [檔案開頭與結尾附加內容](#prepending-appending-to-files)
    - [複製與移動檔案](#copying-moving-files)
    - [自動串流](#automatic-streaming)
    - [檔案上傳](#file-uploads)
    - [檔案可見性](#file-visibility)
- [刪除檔案](#deleting-files)
- [目錄](#directories)
- [測試](#testing)
- [自訂檔案系統](#custom-filesystems)

<a name="introduction"></a>
## 簡介

Laravel 透過 Frank de Jonge 開發的極佳 [Flysystem](https://github.com/thephpleague/flysystem) PHP 套件，提供強大的檔案系統抽象化功能。此 Laravel Flysystem 整合為本地檔案系統、SFTP 與 Amazon S3 提供了簡單的驅動器。更棒的是，由於每個系統的 API 都保持相同，因此在本地開發機與正式伺服器之間切換這些儲存選項變得異常簡單。

<a name="configuration"></a>
## 設定

Laravel 的檔案系統設定檔位於 `config/filesystems.php`。在此檔案中，您可以設定所有檔案系統的「磁碟」(disks)。每個磁碟都代表一個特定的儲存驅動器和儲存位置。設定檔中包含了每個支援驅動器的範例設定，您可以修改這些設定以反映您的儲存偏好和憑證。

`local` 驅動器與儲存在執行 Laravel 應用程式伺服器上的本地檔案互動，而 `s3` 驅動器則用於寫入 Amazon S3 雲端儲存服務。

> [!NOTE]  
> 您可以設定任意數量的磁碟，甚至可以有多個磁碟使用相同的驅動器。

<a name="the-local-driver"></a>
### 本地驅動

當使用 `local` 驅動器時，所有檔案操作都是相對於 `filesystems` 設定檔中定義的 `root` 目錄。預設情況下，此值設定為 `storage/app/private` 目錄。因此，以下方法會寫入 `storage/app/private/example.txt`：

    use Illuminate\Support\Facades\Storage;

    Storage::disk('local')->put('example.txt', 'Contents');

<a name="the-public-disk"></a>
### Public 磁碟

應用程式的 `filesystems` 設定檔中包含的 `public` 磁碟，旨在用於公開存取的檔案。預設情況下，`public` 磁碟使用 `local` 驅動器，並將檔案儲存在 `storage/app/public` 中。

如果您的 `public` 磁碟使用 `local` 驅動器，並且您希望這些檔案可以從網頁存取，則應從來源目錄 `storage/app/public` 建立一個符號連結到目標目錄 `public/storage`：

要建立符號連結，您可以使用 `storage:link` Artisan 命令：

```shell
php artisan storage:link
```

一旦檔案被儲存並建立了符號連結，您可以使用 `asset` 輔助函式建立檔案的 URL：

    echo asset('storage/file.txt');

您可以在 `filesystems` 設定檔中設定額外的符號連結。運行 `storage:link` 命令時，所有已設定的連結都將被建立：

    'links' => [
        public_path('storage') => storage_path('app/public'),
        public_path('images') => storage_path('app/images'),
    ],

`storage:unlink` 命令可用於銷毀已設定的符號連結：

```shell
php artisan storage:unlink
```

<a name="driver-prerequisites"></a>
### 驅動器先決條件

<a name="s3-driver-configuration"></a>
#### S3 驅動器設定

在使用 S3 驅動器之前，您需要透過 Composer 套件管理器安裝 Flysystem S3 套件：

```shell
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

S3 磁碟設定陣列位於您的 `config/filesystems.php` 設定檔中。通常，您應該使用以下環境變數設定您的 S3 資訊和憑證，這些變數由 `config/filesystems.php` 設定檔引用：

```
AWS_ACCESS_KEY_ID=<your-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=<your-bucket-name>
AWS_USE_PATH_STYLE_ENDPOINT=false
```

為方便起見，這些環境變數與 AWS CLI 使用的命名慣例一致。

<a name="ftp-driver-configuration"></a>
#### FTP 驅動器設定

在使用 FTP 驅動器之前，您需要透過 Composer 套件管理器安裝 Flysystem FTP 套件：

```shell
composer require league/flysystem-ftp "^3.0"
```

Laravel 的 Flysystem 整合與 FTP 配合得很好；但是，框架的預設 `config/filesystems.php` 設定檔中不包含範例設定。如果您需要設定 FTP 檔案系統，可以使用以下設定範例：

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

<a name="sftp-driver-configuration"></a>
#### SFTP 驅動器設定

在使用 SFTP 驅動器之前，您需要透過 Composer 套件管理器安裝 Flysystem SFTP 套件：

```shell
composer require league/flysystem-sftp-v3 "^3.0"
```

Laravel 的 Flysystem 整合與 SFTP 配合得很好；但是，框架的預設 `config/filesystems.php` 設定檔中不包含範例設定。如果您需要設定 SFTP 檔案系統，可以使用以下設定範例：

    'sftp' => [
        'driver' => 'sftp',
        'host' => env('SFTP_HOST'),

        // Settings for basic authentication...
        'username' => env('SFTP_USERNAME'),
        'password' => env('SFTP_PASSWORD'),

        // Settings for SSH key based authentication with encryption password...
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

<a name="scoped-and-read-only-filesystems"></a>
### 作用域與唯讀檔案系統

作用域磁碟允許您定義一個檔案系統，其中所有路徑都會自動加上指定的路徑前綴。在建立作用域檔案系統磁碟之前，您需要透過 Composer 套件管理器安裝額外的 Flysystem 套件：

```shell
composer require league/flysystem-path-prefixing "^3.0"
```

您可以透過定義使用 `scoped` 驅動器的磁碟來建立任何現有檔案系統磁碟的路徑作用域實例。例如，您可以建立一個磁碟，將您現有的 `s3` 磁碟作用域到一個特定的路徑前綴，然後使用作用域磁碟的每個檔案操作都將使用指定的前綴：

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

「唯讀」(Read-only) 磁碟允許您建立不允許寫入操作的檔案系統磁碟。在使用 `read-only` 設定選項之前，您需要透過 Composer 套件管理器安裝額外的 Flysystem 套件：

```shell
composer require league/flysystem-read-only "^3.0"
```

接下來，您可以在一個或多個磁碟的設定陣列中包含 `read-only` 設定選項：

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```

<a name="amazon-s3-compatible-filesystems"></a>
### 相容 Amazon S3 的檔案系統

預設情況下，應用程式的 `filesystems` 設定檔包含 `s3` 磁碟的設定。除了使用此磁碟與 [Amazon S3](https://aws.amazon.com/s3/) 互動外，您也可以使用它來與任何 S3 相容的檔案儲存服務互動，例如 [MinIO](https://github.com/minio/minio)、[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)、[Vultr Object Storage](https://www.vultr.com/products/object-storage/)、[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) 或 [Hetzner Cloud Storage](https://www.hetzner.com/storage/object-storage/)。

通常，在更新磁碟的憑證以符合您計劃使用的服務憑證後，您只需要更新 `endpoint` 設定選項的值。此選項的值通常透過 `AWS_ENDPOINT` 環境變數定義：

    'endpoint' => env('AWS_ENDPOINT', 'https://minio:9000'),


<a name="minio"></a>
#### MinIO

為了讓 Laravel 的 Flysystem 整合在使用 MinIO 時能產生正確的 URL，您應該定義 `AWS_URL` 環境變數，使其符合應用程式的本地 URL 並包含 URL 路徑中的 bucket 名稱：

```ini
AWS_URL=http://localhost:9000/local
```

> [!WARNING]  
> 透過 `temporaryUrl` 方法產生暫時儲存 URL 在使用 MinIO 時可能無法運作，如果 `endpoint` 無法被用戶端存取。

<a name="obtaining-disk-instances"></a>
## 取得磁碟實例

`Storage` Facade 可用於與任何已設定的磁碟互動。例如，您可以使用 Facade 上的 `put` 方法將頭像儲存到預設磁碟。如果您在未先呼叫 `disk` 方法的情況下呼叫 `Storage` Facade 上的方法，該方法將會自動傳遞給預設磁碟：

    use Illuminate\Support\Facades\Storage;

    Storage::put('avatars/1', $content);

如果您的應用程式與多個磁碟互動，您可以使用 `Storage` Facade 上的 `disk` 方法來處理特定磁碟上的檔案：

    Storage::disk('s3')->put('avatars/1', $content);

<a name="on-demand-disks"></a>
### 隨選磁碟

有時您可能希望在執行時使用給定設定建立一個磁碟，而無需該設定實際存在於您應用程式的 `filesystems` 設定檔中。為此，您可以將一個設定陣列傳遞給 `Storage` Facade 的 `build` 方法：

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);

$disk->put('image.jpg', $content);
```

<a name="retrieving-files"></a>
## 檔案取回

`get` 方法可用來取回檔案的內容。此方法會傳回檔案的原始字串內容。請記住，所有檔案路徑都應相對於磁碟的「根」位置指定：

    $contents = Storage::get('file.jpg');

如果取回的檔案包含 JSON，您可以使用 `json` 方法來取回檔案並解碼其內容：

    $orders = Storage::json('orders.json');

`exists` 方法可用來判斷檔案是否存在於磁碟上：

    if (Storage::disk('s3')->exists('file.jpg')) {
        // ...
    }

`missing` 方法可用來判斷檔案是否遺失於磁碟上：

    if (Storage::disk('s3')->missing('file.jpg')) {
        // ...
    }


<a name="downloading-files"></a>
### 下載檔案

`download` 方法可用來產生一個回應，該回應會強制使用者的瀏覽器下載指定路徑的檔案。`download` 方法接受一個檔名作為方法的第二個引數，這將決定使用者下載檔案時看到的檔名。最後，您可以傳遞一個 HTTP 標頭陣列作為方法的第三個引數：

    return Storage::download('file.jpg');

    return Storage::download('file.jpg', $name, $headers);


<a name="file-urls"></a>
### 檔案 URL

您可以使用 `url` 方法來取得指定檔案的 URL。如果您使用 `local` 驅動，這通常只會將 `/storage` 前置於指定路徑並傳回檔案的相對 URL。如果您使用 `s3` 驅動，則會傳回完整限定的遠端 URL：

    use Illuminate\Support\Facades\Storage;

    $url = Storage::url('file.jpg');

當使用 `local` 驅動時，所有應公開存取的檔案都應放在 `storage/app/public` 目錄中。此外，您應該在 `public/storage` 建立一個指向 `storage/app/public` 目錄的[符號連結](#the-public-disk)。

> [!WARNING]  
> 當使用 `local` 驅動時，`url` 的回傳值未經 URL 編碼。因此，我們建議您始終使用能建立有效 URL 的名稱來儲存檔案。


<a name="url-host-customization"></a>
#### URL 主機自訂

如果您想修改使用 `Storage` Facade 產生的 URL 的主機，您可以在磁碟的配置陣列中新增或更改 `url` 選項：

    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
        'throw' => false,
    ],


<a name="temporary-urls"></a>
### 暫時 URL

使用 `temporaryUrl` 方法，您可以為使用 `local` 和 `s3` 驅動程式儲存的檔案建立暫時 URL。此方法接受一個路徑和一個 `DateTime` 實例，指定 URL 何時到期：

    use Illuminate\Support\Facades\Storage;

    $url = Storage::temporaryUrl(
        'file.jpg', now()->addMinutes(5)
    );


<a name="enabling-local-temporary-urls"></a>
#### 啟用本地暫時 URL

如果您在 `local` 驅動程式引入暫時 URL 支援之前開始開發應用程式，您可能需要啟用本地暫時 URL。為此，請在您的 `local` 磁碟的配置陣列中，將 `serve` 選項新增至 `config/filesystems.php` 設定檔中：

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

如果您需要指定額外的 [S3 請求參數](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests)，您可以將請求參數陣列作為 `temporaryUrl` 方法的第三個引數傳遞：

    $url = Storage::temporaryUrl(
        'file.jpg',
        now()->addMinutes(5),
        [
            'ResponseContentType' => 'application/octet-stream',
            'ResponseContentDisposition' => 'attachment; filename=file2.jpg',
        ]
    );


<a name="customizing-temporary-urls"></a>
#### 自訂暫時 URL

如果您需要自訂特定儲存磁碟的暫時 URL 建立方式，您可以使用 `buildTemporaryUrlsUsing` 方法。舉例來說，如果您有一個控制器允許您下載透過通常不支援暫時 URL 的磁碟儲存的檔案，這會很有用。通常，此方法應從服務提供者的 `boot` 方法中呼叫：

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


<a name="temporary-upload-urls"></a>
#### 暫時上傳 URL

> [!WARNING]  
> 產生暫時上傳 URL 的功能僅支援 `s3` 驅動。

如果您需要產生一個可用於直接從客戶端應用程式上傳檔案的暫時 URL，您可以使用 `temporaryUploadUrl` 方法。此方法接受一個路徑和一個 `DateTime` 實例，指定 URL 何時到期。`temporaryUploadUrl` 方法會傳回一個關聯陣列，該陣列可解構為上傳 URL 和應包含在上傳請求中的標頭：

    use Illuminate\Support\Facades\Storage;

    ['url' => $url, 'headers' => $headers] = Storage::temporaryUploadUrl(
        'file.jpg', now()->addMinutes(5)
    );

此方法主要用於無伺服器環境，這些環境需要客戶端應用程式直接將檔案上傳到雲端儲存系統，例如 Amazon S3。


<a name="file-metadata"></a>
### 檔案中繼資料

除了讀取和寫入檔案，Laravel 還可以提供檔案本身的資訊。例如，`size` 方法可用於取得檔案大小 (以位元組為單位)：

    use Illuminate\Support\Facades\Storage;

    $size = Storage::size('file.jpg');

`lastModified` 方法傳回檔案上次修改的 UNIX 時間戳：

    $time = Storage::lastModified('file.jpg');

指定檔案的 MIME 類型可透過 `mimeType` 方法取得：

    $mime = Storage::mimeType('file.jpg');


<a name="file-paths"></a>
#### 檔案路徑

您可以使用 `path` 方法來取得指定檔案的路徑。如果您使用的是 `local` 驅動，這將傳回檔案的絕對路徑。如果您使用的是 `s3` 驅動，此方法將傳回 S3 Bucket 中檔案的相對路徑：

    use Illuminate\Support\Facades\Storage;

    $path = Storage::path('file.jpg');

<a name="storing-files"></a>
## 檔案儲存

`put` 方法可用來將檔案內容儲存到磁碟中。您也可以將 PHP `resource` 傳遞給 `put` 方法，這將使用 Flysystem 底層的串流支援。請記住，所有檔案路徑都應相對於磁碟配置的「根」位置指定：

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents);

    Storage::put('file.jpg', $resource);


<a name="failed-writes"></a>
#### 寫入失敗

如果 `put` 方法 (或其他「寫入」操作) 無法將檔案寫入磁碟，將會傳回 `false`：

    if (! Storage::put('file.jpg', $contents)) {
        // The file could not be written to disk...
    }

如果您願意，可以在檔案系統磁碟的配置陣列中定義 `throw` 選項。當此選項定義為 `true` 時，像是 `put` 這類的「寫入」方法，在寫入操作失敗時將會拋出 `League\Flysystem\UnableToWriteFile` 的實例：

    'public' => [
        'driver' => 'local',
        // ...
        'throw' => true,
    ],


<a name="prepending-appending-to-files"></a>
### 檔案開頭與結尾附加內容

`prepend` 和 `append` 方法允許您將內容寫入檔案的開頭或結尾：

    Storage::prepend('file.log', 'Prepended Text');

    Storage::append('file.log', 'Appended Text');


<a name="copying-moving-files"></a>
### 複製與移動檔案

`copy` 方法可用於將現有檔案複製到磁碟上的新位置，而 `move` 方法則可用於重新命名或移動現有檔案到新位置：

    Storage::copy('old/file.jpg', 'new/file.jpg');

    Storage::move('old/file.jpg', 'new/file.jpg');


<a name="automatic-streaming"></a>
### 自動串流

將檔案串流到儲存裝置可大幅降低記憶體使用量。如果您希望 Laravel 自動管理將指定檔案串流到您的儲存位置，可以使用 `putFile` 或 `putFileAs` 方法。此方法接受 `Illuminate\Http\File` 或 `Illuminate\Http\UploadedFile` 實例，並會自動將檔案串流到您指定的位置：

    use Illuminate\Http\File;
    use Illuminate\Support\Facades\Storage;

    // Automatically generate a unique ID for filename...
    $path = Storage::putFile('photos', new File('/path/to/photo'));

    // Manually specify a filename...
    $path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');

關於 `putFile` 方法，有幾點重要事項需要注意。請注意，我們只指定了目錄名稱，而沒有指定檔案名稱。預設情況下，`putFile` 方法會生成一個唯一 ID 作為檔案名稱。檔案的副檔名將透過檢查檔案的 MIME 類型來確定。`putFile` 方法會傳回檔案路徑，以便您可以將該路徑 (包含生成的檔案名稱) 儲存到您的資料庫中。

`putFile` 和 `putFileAs` 方法也接受一個參數來指定儲存檔案的「可見性」。如果您將檔案儲存在雲端磁碟 (例如 Amazon S3) 上，並且希望檔案能夠透過生成的 URL 公開存取，這會特別有用：

    Storage::putFile('photos', new File('/path/to/photo'), 'public');


<a name="file-uploads"></a>
### 檔案上傳

在網頁應用程式中，儲存檔案最常見的用途之一就是儲存使用者上傳的檔案，例如照片和文件。Laravel 透過上傳檔案實例上的 `store` 方法，讓儲存上傳的檔案變得非常容易。請呼叫 `store` 方法並傳入您希望儲存上傳檔案的路徑：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

關於這個範例，有幾點重要事項需要注意。請注意，我們只指定了目錄名稱，而沒有指定檔案名稱。預設情況下，`store` 方法會生成一個唯一 ID 作為檔案名稱。檔案的副檔名將透過檢查檔案的 MIME 類型來確定。`store` 方法會傳回檔案路徑，以便您可以將該路徑 (包含生成的檔案名稱) 儲存到您的資料庫中。

您也可以呼叫 `Storage` Facade 上的 `putFile` 方法來執行與上述範例相同的檔案儲存操作：

    $path = Storage::putFile('avatars', $request->file('avatar'));


<a name="specifying-a-file-name"></a>
#### 指定檔案名稱

如果您不希望自動指定檔案名稱給您儲存的檔案，可以使用 `storeAs` 方法，此方法會接收路徑、檔案名稱以及 (選填) 磁碟作為其參數：

    $path = $request->file('avatar')->storeAs(
        'avatars', $request->user()->id
    );

您也可以呼叫 `Storage` Facade 上的 `putFileAs` 方法，這將執行與上述範例相同的檔案儲存操作：

    $path = Storage::putFileAs(
        'avatars', $request->file('avatar'), $request->user()->id
    );

> [!WARNING]  
> 無法列印和無效的 Unicode 字元將會自動從檔案路徑中移除。因此，您可能希望在將檔案路徑傳遞給 Laravel 的檔案儲存方法之前，先清理您的檔案路徑。檔案路徑是使用 `League\Flysystem\WhitespacePathNormalizer::normalizePath` 方法進行正規化的。


<a name="specifying-a-disk"></a>
#### 指定磁碟

預設情況下，此上傳檔案的 `store` 方法將使用您的預設磁碟。如果您想指定另一個磁碟，請將磁碟名稱作為 `store` 方法的第二個參數傳遞：

    $path = $request->file('avatar')->store(
        'avatars/'.$request->user()->id, 's3'
    );

如果您正在使用 `storeAs` 方法，您可以將磁碟名稱作為方法的第三個參數傳遞：

    $path = $request->file('avatar')->storeAs(
        'avatars',
        $request->user()->id,
        's3'
    );


<a name="other-uploaded-file-information"></a>
#### 其他上傳檔案資訊

如果您想取得上傳檔案的原始名稱和副檔名，可以使用 `getClientOriginalName` 和 `getClientOriginalExtension` 方法：

    $file = $request->file('avatar');

    $name = $file->getClientOriginalName();
    $extension = $file->getClientOriginalExtension();

然而，請記住 `getClientOriginalName` 和 `getClientOriginalExtension` 方法被認為是不安全的，因為檔案名稱和副檔名可能被惡意使用者竄改。因此，通常您應該優先使用 `hashName` 和 `extension` 方法來取得指定檔案上傳的名稱和副檔名：

    $file = $request->file('avatar');

    $name = $file->hashName(); // Generate a unique, random name...
    $extension = $file->extension(); // Determine the file's extension based on the file's MIME type...

<a name="file-visibility"></a>
### 檔案可見性

在 Laravel 的 Flysystem 整合中，「可見性」是跨多個平台檔案權限的抽象概念。檔案可以被宣告為 `public` 或 `private`。當檔案被宣告為 `public` 時，表示該檔案通常可供其他人存取。例如，當使用 S3 驅動器時，您可以取回 `public` 檔案的 URL。

您可以在寫入檔案時，透過 `put` 方法設定可見性：

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents, 'public');

如果檔案已經儲存，其可見性可以透過 `getVisibility` 和 `setVisibility` 方法取回及設定：

    $visibility = Storage::getVisibility('file.jpg');

    Storage::setVisibility('file.jpg', 'public');

與上傳檔案互動時，您可以使用 `storePublicly` 和 `storePubliclyAs` 方法以 `public` 可見性儲存上傳的檔案：

    $path = $request->file('avatar')->storePublicly('avatars', 's3');

    $path = $request->file('avatar')->storePubliclyAs(
        'avatars',
        $request->user()->id,
        's3'
    );


<a name="local-files-and-visibility"></a>
#### 本地檔案與可見性

當使用 `local` 驅動器時，`public` [可見性](#file-visibility) 等同於目錄的 `0755` 權限和檔案的 `0644` 權限。您可以在應用程式的 `filesystems` 設定檔中修改權限映射：

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

<a name="deleting-files"></a>
## 刪除檔案

`delete` 方法接受單一檔案名稱或一個檔案陣列來進行刪除：

    use Illuminate\Support\Facades\Storage;

    Storage::delete('file.jpg');

    Storage::delete(['file.jpg', 'file2.jpg']);

如有需要，您可以指定要從哪個磁碟刪除檔案：

    use Illuminate\Support\Facades\Storage;

    Storage::disk('s3')->delete('path/file.jpg');


<a name="directories"></a>
## 目錄


<a name="get-all-files-within-a-directory"></a>
#### 取得目錄中的所有檔案

`files` 方法會回傳指定目錄中所有檔案的陣列。如果您想取得指定目錄（包含所有子目錄）中所有檔案的列表，可以使用 `allFiles` 方法：

    use Illuminate\Support\Facades\Storage;

    $files = Storage::files($directory);

    $files = Storage::allFiles($directory);


<a name="get-all-directories-within-a-directory"></a>
#### 取得目錄中的所有子目錄

`directories` 方法會回傳指定目錄中所有子目錄的陣列。此外，您可以使用 `allDirectories` 方法來取得指定目錄及其所有子目錄中所有目錄的列表：

    $directories = Storage::directories($directory);

    $directories = Storage::allDirectories($directory);


<a name="create-a-directory"></a>
#### 建立目錄

`makeDirectory` 方法會建立指定的目錄，包含任何必要的子目錄：

    Storage::makeDirectory($directory);


<a name="delete-a-directory"></a>
#### 刪除目錄

最後，`deleteDirectory` 方法可用於移除一個目錄及其所有檔案：

    Storage::deleteDirectory($directory);


<a name="testing"></a>
## 測試

`Storage` Facade 的 `fake` 方法讓您可以輕鬆地生成一個假磁碟，結合 `Illuminate\Http\UploadedFile` 類別的檔案生成工具，大大簡化了檔案上傳的測試。例如：

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

預設情況下，`fake` 方法會刪除其暫存目錄中的所有檔案。如果您想保留這些檔案，可以使用「persistentFake」方法。有關測試檔案上傳的更多資訊，您可以查閱 [HTTP 測試文件中有關檔案上傳的資訊](/docs/{{version}}/http-tests#testing-file-uploads)。

> [!WARNING]  
> `image` 方法需要 [GD 擴充功能](https://www.php.net/manual/en/book.image.php)。


<a name="custom-filesystems"></a>
## 自訂檔案系統

Laravel 的 Flysystem 整合提供了對多種「驅動器」的開箱即用支援；然而，Flysystem 並不限於這些驅動器，它還有許多其他儲存系統的轉接器 (adapter)。如果您想在 Laravel 應用程式中使用這些額外的轉接器之一，可以建立一個自訂驅動器。

為了定義自訂檔案系統，您需要一個 Flysystem 轉接器 (adapter)。讓我們為專案新增一個由社群維護的 Dropbox 轉接器：

```shell
composer require spatie/flysystem-dropbox
```

接下來，您可以在應用程式其中一個 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中註冊該驅動器。為此，您應該使用 `Storage` Facade 的 `extend` 方法：

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

`extend` 方法的第一個參數是驅動器的名稱，第二個是一個接收 `$app` 和 `$config` 變數的閉包。該閉包必須回傳 `Illuminate\Filesystem\FilesystemAdapter` 的實例。 `$config` 變數包含 `config/filesystems.php` 中為指定磁碟定義的值。

一旦您建立並註冊了擴充功能的服務提供者，您就可以在 `config/filesystems.php` 設定檔中使用 `dropbox` 驅動器了。