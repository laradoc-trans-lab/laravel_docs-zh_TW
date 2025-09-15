# 檔案儲存

- [簡介](#introduction)
- [設定](#configuration)
    - [Local Driver](#the-local-driver)
    - [Public Disk](#the-public-disk)
    - [Driver 先決條件](#driver-prerequisites)
    - [Scoped 與唯讀檔案系統](#scoped-and-read-only-filesystems)
    - [Amazon S3 相容檔案系統](#amazon-s3-compatible-filesystems)
- [取得 Disk 實例](#obtaining-disk-instances)
    - [On-Demand Disks](#on-demand-disks)
- [擷取檔案](#retrieving-files)
    - [下載檔案](#downloading-files)
    - [檔案 URL](#file-urls)
    - [暫時性 URL](#temporary-urls)
    - [檔案 Metadata](#file-metadata)
- [儲存檔案](#storing-files)
    - [在檔案開頭與結尾寫入](#prepending-and-appending-to-files)
    - [複製與移動檔案](#copying-and-moving-files)
    - [自動串流](#automatic-streaming)
    - [檔案上傳](#file-uploads)
    - [檔案可見性](#file-visibility)
- [刪除檔案](#deleting-files)
- [目錄](#directories)
- [測試](#testing)
- [自訂檔案系統](#custom-filesystems)

<a name="introduction"></a>
## 簡介

Laravel 透過 Frank de Jonge 出色的 [Flysystem](https://github.com/thephpleague/flysystem) PHP 套件，提供了強大的檔案系統抽象層。Laravel 的 Flysystem 整合為本地檔案系統、SFTP 和 Amazon S3 提供了簡單的 driver。更棒的是，在本地開發機器和正式環境伺服器之間切換這些儲存選項非常簡單，因為每個系統的 API 都保持不變。

<a name="configuration"></a>
## 設定

Laravel 的檔案系統設定檔位於 `config/filesystems.php`。在此檔案中，您可以設定所有檔案系統的「disk」。每個 disk 都代表一個特定的儲存 driver 和儲存位置。設定檔中包含了每個支援 driver 的範例設定，因此您可以修改設定以反映您的儲存偏好和憑證。

`local` driver 與儲存在執行 Laravel 應用程式的伺服器上的檔案互動，而 `s3` driver 則用於寫入 Amazon 的 S3 雲端儲存服務。

> [!NOTE]
> 您可以設定任意數量的 disk，甚至可以有多個 disk 使用相同的 driver。

<a name="the-local-driver"></a>
### Local Driver

使用 `local` driver 時，所有檔案操作都相對於 `filesystems` 設定檔中定義的 `root` 目錄。預設情況下，此值設定為 `storage/app/private` 目錄。因此，以下方法會寫入 `storage/app/private/example.txt`：

    use Illuminate\Support\Facades\Storage;

    Storage::disk('local')->put('example.txt', 'Contents');

<a name="the-public-disk"></a>
### Public Disk

應用程式 `filesystems` 設定檔中包含的 `public` disk 旨在用於公開可存取的檔案。預設情況下，`public` disk 使用 `local` driver 並將其檔案儲存在 `storage/app/public` 中。

如果您的 `public` disk 使用 `local` driver，並且您希望這些檔案可以從網頁存取，您應該從來源目錄 `storage/app/public` 建立一個符號連結到目標目錄 `public/storage`：

要建立符號連結，您可以使用 `storage:link` Artisan 命令：

```shell
php artisan storage:link
```

一旦檔案儲存完成並建立符號連結後，您可以使用 `asset` 輔助函式建立檔案的 URL：

    echo asset('storage/file.txt');

您可以在 `filesystems` 設定檔中設定額外的符號連結。當您執行 `storage:link` 命令時，每個設定的連結都將被建立：

    'links' => [
        public_path('storage') => storage_path('app/public'),
        public_path('images') => storage_path('app/images'),
    ],

`storage:unlink` 命令可用於銷毀您設定的符號連結：

```shell
php artisan storage:unlink
```

<a name="driver-prerequisites"></a>
### Driver 先決條件

<a name="s3-driver-configuration"></a>
#### S3 Driver 設定

在使用 S3 driver 之前，您需要透過 Composer 套件管理器安裝 Flysystem S3 套件：

```shell
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

S3 disk 設定陣列位於您的 `config/filesystems.php` 設定檔中。通常，您應該使用以下環境變數來設定您的 S3 資訊和憑證，這些變數由 `config/filesystems.php` 設定檔引用：

```
AWS_ACCESS_KEY_ID=<your-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=<your-bucket-name>
AWS_USE_PATH_STYLE_ENDPOINT=false
```

為方便起見，這些環境變數與 AWS CLI 使用的命名慣例相符。

<a name="ftp-driver-configuration"></a>
#### FTP Driver 設定

在使用 FTP driver 之前，您需要透過 Composer 套件管理器安裝 Flysystem FTP 套件：

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
#### SFTP Driver 設定

在使用 SFTP driver 之前，您需要透過 Composer 套件管理器安裝 Flysystem SFTP 套件：

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
### Scoped 與唯讀檔案系統

Scoped disk 允許您定義一個檔案系統，其中所有路徑都會自動加上給定的路徑前綴。在建立 Scoped 檔案系統 disk 之前，您需要透過 Composer 套件管理器安裝一個額外的 Flysystem 套件：

```shell
composer require league/flysystem-path-prefixing "^3.0"
```

您可以透過定義一個使用 `scoped` driver 的 disk，來建立任何現有檔案系統 disk 的路徑 Scoped 實例。例如，您可以建立一個將現有 `s3` disk Scoped 到特定路徑前綴的 disk，然後使用您的 Scoped disk 的每個檔案操作都將使用指定的字首：

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

「唯讀」disk 允許您建立不允許寫入操作的檔案系統 disk。在使用 `read-only` 設定選項之前，您需要透過 Composer 套件管理器安裝一個額外的 Flysystem 套件：

```shell
composer require league/flysystem-read-only "^3.0"
```

接下來，您可以在一個或多個 disk 的設定陣列中包含 `read-only` 設定選項：

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```

<a name="amazon-s3-compatible-filesystems"></a>
### Amazon S3 相容檔案系統

預設情況下，應用程式的 `filesystems` 設定檔包含 `s3` disk 的設定。除了使用此 disk 與 [Amazon S3](https://aws.amazon.com/s3/) 互動之外，您還可以使用它與任何 S3 相容的檔案儲存服務互動，例如 [MinIO](https://github.com/minio/minio)、[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)、[Vultr Object Storage](https://www.vultr.com/products/object-storage/)、[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) 或 [Hetzner Cloud Storage](https://www.hetzner.com/storage/object-storage/)。

通常，在更新 disk 的憑證以符合您計劃使用的服務的憑證後，您只需要更新 `endpoint` 設定選項的值。此選項的值通常透過 `AWS_ENDPOINT` 環境變數定義：

    'endpoint' => env('AWS_ENDPOINT', 'https://minio:9000'),

<a name="minio"></a>
#### MinIO

為了讓 Laravel 的 Flysystem 整合在使用 MinIO 時產生正確的 URL，您應該定義 `AWS_URL` 環境變數，使其與應用程式的本地 URL 相符，並在 URL 路徑中包含 bucket 名稱：

```ini
AWS_URL=http://localhost:9000/local
```

> [!WARNING]
> 如果客戶端無法存取 `endpoint`，則在使用 MinIO 時，透過 `temporaryUrl` 方法產生暫時性儲存 URL 可能無法運作。

<a name="obtaining-disk-instances"></a>
## 取得 Disk 實例

`Storage` Facade 可用於與任何已設定的 disk 互動。例如，您可以使用 Facade 上的 `put` 方法將頭像儲存在預設 disk 上。如果您在未先呼叫 `disk` 方法的情況下呼叫 `Storage` Facade 上的方法，該方法將自動傳遞給預設 disk：

    use Illuminate\Support\Facades\Storage;

    Storage::put('avatars/1', $content);

如果您的應用程式與多個 disk 互動，您可以使用 `Storage` Facade 上的 `disk` 方法來處理特定 disk 上的檔案：

    Storage::disk('s3')->put('avatars/1', $content);

<a name="on-demand-disks"></a>
### On-Demand Disks

有時您可能希望在執行時使用給定設定建立 disk，而該設定實際上並不存在於應用程式的 `filesystems` 設定檔中。為此，您可以將設定陣列傳遞給 `Storage` Facade 的 `build` 方法：

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

`get` 方法可用於擷取檔案的內容。該方法將返回檔案的原始字串內容。請記住，所有檔案路徑都應相對於 disk 的「root」位置指定：

    $contents = Storage::get('file.jpg');

如果您擷取的檔案包含 JSON，您可以使用 `json` 方法擷取檔案並解碼其內容：

    $orders = Storage::json('orders.json');

`exists` 方法可用於判斷檔案是否存在於 disk 上：

    if (Storage::disk('s3')->exists('file.jpg')) {
        // ...
    }

`missing` 方法可用於判斷檔案是否從 disk 上遺失：

    if (Storage::disk('s3')->missing('file.jpg')) {
        // ...
    }

<a name="downloading-files"></a>
### 下載檔案

`download` 方法可用於產生一個回應，強制使用者的瀏覽器下載指定路徑的檔案。`download` 方法接受一個檔案名稱作為方法的第二個引數，該檔案名稱將決定使用者下載檔案時看到的檔案名稱。最後，您可以將 HTTP 標頭陣列作為方法的第三個引數傳遞：

    return Storage::download('file.jpg');

    return Storage::download('file.jpg', $name, $headers);

<a name="file-urls"></a>
### 檔案 URL

您可以使用 `url` 方法取得給定檔案的 URL。如果您使用 `local` driver，這通常只會將 `/storage` 前置到給定路徑並返回檔案的相對 URL。如果您使用 `s3` driver，則會返回完全限定的遠端 URL：

    use Illuminate\Support\Facades\Storage;

    $url = Storage::url('file.jpg');

使用 `local` driver 時，所有應該公開可存取的檔案都應放置在 `storage/app/public` 目錄中。此外，您應該在 `public/storage` 建立一個指向 `storage/app/public` 目錄的[符號連結](#the-public-disk)。

> [!WARNING]
> 使用 `local` driver 時，`url` 的回傳值未經 URL 編碼。因此，我們建議始終使用會建立有效 URL 的名稱來儲存檔案。

<a name="url-host-customization"></a>
#### URL 主機自訂

如果您想修改使用 `Storage` Facade 產生 URL 的主機，您可以在 disk 的設定陣列中新增或更改 `url` 選項：

    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
        'throw' => false,
    ],

<a name="temporary-urls"></a>
### 暫時性 URL

使用 `temporaryUrl` 方法，您可以為使用 `local` 和 `s3` driver 儲存的檔案建立暫時性 URL。此方法接受一個路徑和一個 `DateTime` 實例，指定 URL 何時過期：

    use Illuminate\Support\Facades\Storage;

    $url = Storage::temporaryUrl(
        'file.jpg', now()->addMinutes(5)
    );

<a name="enabling-local-temporary-urls"></a>
#### 啟用本地暫時性 URL

如果您在 `local` driver 引入暫時性 URL 支援之前開始開發應用程式，您可能需要啟用本地暫時性 URL。為此，請在 `config/filesystems.php` 設定檔中，將 `serve` 選項新增到 `local` disk 的設定陣列中：

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

如果您需要指定額外的 [S3 請求參數](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests)，您可以將請求參數陣列作為第三個引數傳遞給 `temporaryUrl` 方法：

    $url = Storage::temporaryUrl(
        'file.jpg',
        now()->addMinutes(5),
        [
            'ResponseContentType' => 'application/octet-stream',
            'ResponseContentDisposition' => 'attachment; filename=file2.jpg',
        ]
    );

<a name="customizing-temporary-urls"></a>
#### 自訂暫時性 URL

如果您需要自訂特定儲存 disk 的暫時性 URL 建立方式，您可以使用 `buildTemporaryUrlsUsing` 方法。例如，如果您有一個控制器允許您下載透過不支援暫時性 URL 的 disk 儲存的檔案，這會很有用。通常，此方法應從服務提供者的 `boot` 方法中呼叫：

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
#### 暫時性上傳 URL

> [!WARNING]
> 產生暫時性上傳 URL 的功能僅由 `s3` driver 支援。

如果您需要產生一個可用於直接從客戶端應用程式上傳檔案的暫時性 URL，您可以使用 `temporaryUploadUrl` 方法。此方法接受一個路徑和一個 `DateTime` 實例，指定 URL 何時過期。`temporaryUploadUrl` 方法返回一個關聯陣列，該陣列可以解構為上傳 URL 和應包含在上傳請求中的標頭：

    use Illuminate\Support\Facades\Storage;

    ['url' => $url, 'headers' => $headers] = Storage::temporaryUploadUrl(
        'file.jpg', now()->addMinutes(5)
    );

此方法主要用於需要客戶端應用程式直接將檔案上傳到雲端儲存系統（例如 Amazon S3）的無伺服器環境。

<a name="file-metadata"></a>
### 檔案 Metadata

除了讀取和寫入檔案之外，Laravel 還可以提供有關檔案本身的資訊。例如，`size` 方法可用於取得檔案的大小（以位元組為單位）：

    use Illuminate\Support\Facades\Storage;

    $size = Storage::size('file.jpg');

`lastModified` 方法返回檔案上次修改的 UNIX 時間戳記：

    $time = Storage::lastModified('file.jpg');

給定檔案的 MIME 類型可以透過 `mimeType` 方法取得：

    $mime = Storage::mimeType('file.jpg');

<a name="file-paths"></a>
#### 檔案路徑

您可以使用 `path` 方法取得給定檔案的路徑。如果您使用 `local` driver，這將返回檔案的絕對路徑。如果您使用 `s3` driver，此方法將返回 S3 bucket 中檔案的相對路徑：

    use Illuminate\Support\Facades\Storage;

    $path = Storage::path('file.jpg');

<a name="storing-files"></a>
## 儲存檔案

`put` 方法可用於將檔案內容儲存在 disk 上。您也可以將 PHP `resource` 傳遞給 `put` 方法，這將使用 Flysystem 的底層串流支援。請記住，所有檔案路徑都應相對於為 disk 設定的「root」位置指定：

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents);

    Storage::put('file.jpg', $resource);

<a name="failed-writes"></a>
#### 寫入失敗

如果 `put` 方法（或其他「寫入」操作）無法將檔案寫入 disk，則會返回 `false`：

    if (! Storage::put('file.jpg', $contents)) {
        // The file could not be written to disk...
    }

如果您願意，可以在檔案系統 disk 的設定陣列中定義 `throw` 選項。當此選項定義為 `true` 時，「寫入」方法（例如 `put`）在寫入操作失敗時將拋出 `League\Flysystem\UnableToWriteFile` 實例：

    'public' => [
        'driver' => 'local',
        // ...
        'throw' => true,
    ],

<a name="prepending-appending-to-files"></a>
### 在檔案開頭與結尾寫入

`prepend` 和 `append` 方法允許您在檔案的開頭或結尾寫入：

    Storage::prepend('file.log', 'Prepended Text');

    Storage::append('file.log', 'Appended Text');

<a name="copying-moving-files"></a>
### 複製與移動檔案

`copy` 方法可用於將現有檔案複製到 disk 上的新位置，而 `move` 方法可用於重新命名或將現有檔案移動到新位置：

    Storage::copy('old/file.jpg', 'new/file.jpg');

    Storage::move('old/file.jpg', 'new/file.jpg');

<a name="automatic-streaming"></a>
### 自動串流

將檔案串流到儲存空間可顯著減少記憶體使用量。如果您希望 Laravel 自動管理將給定檔案串流到您的儲存位置，您可以使用 `putFile` 或 `putFileAs` 方法。此方法接受 `Illuminate\Http\File` 或 `Illuminate\Http\UploadedFile` 實例，並會自動將檔案串流到您所需的位置：

    use Illuminate\Http\File;
    use Illuminate\Support\Facades\Storage;

    // Automatically generate a unique ID for filename...
    $path = Storage::putFile('photos', new File('/path/to/photo'));

    // Manually specify a filename...
    $path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');

關於 `putFile` 方法，有幾點需要注意。請注意，我們只指定了目錄名稱，而沒有指定檔案名稱。預設情況下，`putFile` 方法將產生一個唯一的 ID 作為檔案名稱。檔案的副檔名將透過檢查檔案的 MIME 類型來確定。`putFile` 方法將返回檔案的路徑，因此您可以將路徑（包括產生的檔案名稱）儲存在資料庫中。

`putFile` 和 `putFileAs` 方法也接受一個引數來指定儲存檔案的「可見性」。如果您將檔案儲存在雲端 disk（例如 Amazon S3）並希望透過產生的 URL 公開存取該檔案，這會特別有用：

    Storage::putFile('photos', new File('/path/to/photo'), 'public');

<a name="file-uploads"></a>
### 檔案上傳

在 Web 應用程式中，儲存檔案最常見的用例之一是儲存使用者上傳的檔案，例如照片和文件。Laravel 透過上傳檔案實例上的 `store` 方法，可以非常輕鬆地儲存上傳的檔案。呼叫 `store` 方法並傳入您希望儲存上傳檔案的路徑：

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

關於此範例，有幾點需要注意。請注意，我們只指定了目錄名稱，而沒有指定檔案名稱。預設情況下，`store` 方法將產生一個唯一的 ID 作為檔案名稱。檔案的副檔名將透過檢查檔案的 MIME 類型來確定。`store` 方法將返回檔案的路徑，因此您可以將路徑（包括產生的檔案名稱）儲存在資料庫中。

您也可以在 `Storage` Facade 上呼叫 `putFile` 方法，以執行與上述範例相同的檔案儲存操作：

    $path = Storage::putFile('avatars', $request->file('avatar'));

<a name="specifying-a-file-name"></a>
#### 指定檔案名稱

如果您不希望自動為儲存的檔案分配檔案名稱，您可以使用 `storeAs` 方法，該方法接收路徑、檔案名稱和（可選）disk 作為其引數：

    $path = $request->file('avatar')->storeAs(
        'avatars', $request->user()->id
    );

您也可以在 `Storage` Facade 上使用 `putFileAs` 方法，該方法將執行與上述範例相同的檔案儲存操作：

    $path = Storage::putFileAs(
        'avatars', $request->file('avatar'), $request->user()->id
    );

> [!WARNING]
> 不可列印和無效的 Unicode 字元將自動從檔案路徑中移除。因此，您可能希望在將檔案路徑傳遞給 Laravel 的檔案儲存方法之前對其進行清理。檔案路徑使用 `League\Flysystem\WhitespacePathNormalizer::normalizePath` 方法進行正規化。

<a name="specifying-a-disk"></a>
#### 指定 Disk

預設情況下，此上傳檔案的 `store` 方法將使用您的預設 disk。如果您想指定另一個 disk，請將 disk 名稱作為第二個引數傳遞給 `store` 方法：

    $path = $request->file('avatar')->store(
        'avatars/'.$request->user()->id, 's3'
    );

如果您使用 `storeAs` 方法，您可以將 disk 名稱作為第三個引數傳遞給該方法：

    $path = $request->file('avatar')->storeAs(
        'avatars',
        $request->user()->id,
        's3'
    );

<a name="other-uploaded-file-information"></a>
#### 其他上傳檔案資訊

如果您想取得上傳檔案的原始名稱和副檔名，您可以使用 `getClientOriginalName` 和 `getClientOriginalExtension` 方法：

    $file = $request->file('avatar');

    $name = $file->getClientOriginalName();
    $extension = $file->getClientOriginalExtension();

但是，請記住 `getClientOriginalName` 和 `getClientOriginalExtension` 方法被認為是不安全的，因為檔案名稱和副檔名可能會被惡意使用者篡改。因此，您通常應該優先使用 `hashName` 和 `extension` 方法來取得給定檔案上傳的名稱和副檔名：

    $file = $request->file('avatar');

    $name = $file->hashName(); // Generate a unique, random name...
    $extension = $file->extension(); // Determine the file's extension based on the file's MIME type...

<a name="file-visibility"></a>
### 檔案可見性

在 Laravel 的 Flysystem 整合中，「可見性」是跨多個平台的檔案權限抽象。檔案可以宣告為 `public` 或 `private`。當檔案宣告為 `public` 時，表示該檔案通常應該對其他人可存取。例如，使用 S3 driver 時，您可以擷取 `public` 檔案的 URL。

您可以在透過 `put` 方法寫入檔案時設定可見性：

    use Illuminate\Support\Facades\Storage;

    Storage::put('file.jpg', $contents, 'public');

如果檔案已經儲存，其可見性可以透過 `getVisibility` 和 `setVisibility` 方法擷取和設定：

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

使用 `local` driver 時，`public` [可見性](#file-visibility) 轉換為目錄的 `0755` 權限和檔案的 `0644` 權限。您可以在應用程式的 `filesystems` 設定檔中修改權限映射：

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

`delete` 方法接受單個檔案名稱或檔案陣列以進行刪除：

    use Illuminate\Support\Facades\Storage;

    Storage::delete('file.jpg');

    Storage::delete(['file.jpg', 'file2.jpg']);

如有必要，您可以指定應從哪個 disk 刪除檔案：

    use Illuminate\Support\Facades\Storage;

    Storage::disk('s3')->delete('path/file.jpg');

<a name="directories"></a>
## 目錄

<a name="get-all-files-within-a-directory"></a>
#### 取得目錄中的所有檔案

`files` 方法返回給定目錄中所有檔案的陣列。如果您想擷取給定目錄中所有檔案（包括所有子目錄）的列表，您可以使用 `allFiles` 方法：

    use Illuminate\Support\Facades\Storage;

    $files = Storage::files($directory);

    $files = Storage::allFiles($directory);

<a name="get-all-directories-within-a-directory"></a>
#### 取得目錄中的所有子目錄

`directories` 方法返回給定目錄中所有子目錄的陣列。此外，您可以使用 `allDirectories` 方法取得給定目錄及其所有子目錄中所有目錄的列表：

    $directories = Storage::directories($directory);

    $directories = Storage::allDirectories($directory);

<a name="create-a-directory"></a>
#### 建立目錄

`makeDirectory` 方法將建立給定的目錄，包括任何所需的子目錄：

    Storage::makeDirectory($directory);

<a name="delete-a-directory"></a>
#### 刪除目錄

最後，`deleteDirectory` 方法可用於移除目錄及其所有檔案：

    Storage::deleteDirectory($directory);

<a name="testing"></a>
## 測試

`Storage` Facade 的 `fake` 方法允許您輕鬆產生一個假的 disk，結合 `Illuminate\Http\UploadedFile` 類別的檔案產生工具，大大簡化了檔案上傳的測試。例如：

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

預設情況下，`fake` 方法將刪除其暫存目錄中的所有檔案。如果您想保留這些檔案，可以使用「persistentFake」方法。有關測試檔案上傳的更多資訊，您可以查閱 [HTTP 測試文件中有關檔案上傳的資訊](/docs/{{version}}/http-tests#testing-file-uploads)。

> [!WARNING]
> `image` 方法需要 [GD 擴充功能](https://www.php.net/manual/en/book.image.php)。

<a name="custom-filesystems"></a>
## 自訂檔案系統

Laravel 的 Flysystem 整合開箱即用支援多個「driver」；但是，Flysystem 不限於這些，並且有許多其他儲存系統的 Adapter。如果您想在 Laravel 應用程式中使用這些額外的 Adapter 之一，您可以建立一個自訂 driver。

為了定義自訂檔案系統，您需要一個 Flysystem Adapter。讓我們將一個社群維護的 Dropbox Adapter 新增到我們的專案中：

```shell
composer require spatie/flysystem-dropbox
```

接下來，您可以在應用程式的其中一個[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中註冊 driver。為此，您應該使用 `Storage` Facade 的 `extend` 方法：

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

`extend` 方法的第一個引數是 driver 的名稱，第二個是接收 `$app` 和 `$config` 變數的閉包。該閉包必須返回 `Illuminate\Filesystem\FilesystemAdapter` 的實例。`$config` 變數包含 `config/filesystems.php` 中為指定 disk 定義的值。

一旦您建立並註冊了擴充功能的服務提供者，您就可以在 `config/filesystems.php` 設定檔中使用 `dropbox` driver。

