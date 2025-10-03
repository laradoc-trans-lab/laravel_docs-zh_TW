# 檔案儲存

- [簡介](#introduction)
- [設定](#configuration)
    - [Local 驅動](#the-local-driver)
    - [Public 磁碟](#the-public-disk)
    - [驅動的必要條件](#driver-prerequisites)
    - [限定範圍與唯讀的檔案系統](#scoped-and-read-only-filesystems)
    - [與 Amazon S3 相容的檔案系統](#amazon-s3-compatible-filesystems)
- [取得磁碟實體](#obtaining-disk-instances)
    - [On-Demand 磁碟](#on-demand-disks)
- [取回檔案](#retrieving-files)
    - [下載檔案](#downloading-files)
    - [檔案 URL](#file-urls)
    - [暫時 URL](#temporary-urls)
    - [檔案元資料](#file-metadata)
- [儲存檔案](#storing-files)
    - [在檔案開頭與結尾寫入](#prepending-appending-to-files)
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

由於有 Frank de Jonge 所開發的優秀 [Flysystem](https://github.com/thephpleague/flysystem) PHP 套件，Laravel 得以提供一個強大的檔案系統抽象層。Laravel 的 Flysystem 整合提供了簡單的驅動，可用於操作本機檔案系統、SFTP、以及 Amazon S3。更棒的是，由於各個系統的 API 都維持不變，因此在本機開發環境與正式伺服器之間切換這些儲存選項時也變得非常簡單。

<a name="configuration"></a>
## 設定

Laravel 的檔案系統設定檔位於 `config/filesystems.php`。在這個檔案中，可以設定所有的檔案系統「磁碟 (disk)」。每個磁碟都代表一種特定的儲存驅動與儲存位置。設定檔中有包含各個支援的驅動的範例設定，因此可以修改設定來反映你的儲存偏好設定與憑證。

`local` 驅動會與儲存在執行 Laravel 應用程式的伺服器上的本機檔案互動，而 `sftp` 儲存驅動則用於基於 SSH 金鑰的 FTP。`s3` 驅動則用於寫入 Amazon S3 雲端儲存服務。

> [!NOTE]
> 你可以設定任意數量的磁碟，甚至可以有多個磁碟使用同一個驅動。

<a name="the-local-driver"></a>
### Local 驅動

使用 `local` 驅動時，所有的檔案操作都會相對於 `filesystems` 設定檔中定義的 `root` 目錄。預設情況下，這個值被設為 `storage/app/private` 目錄。因此，下列方法會寫入 `storage/app/private/example.txt`：

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('example.txt', 'Contents');
```

<a name="the-public-disk"></a>
### Public 磁碟

應用程式的 `filesystems` 設定檔中有包含一個 `public` 磁碟，這個磁碟用於存放要公開存取的檔案。預設情況下，`public` 磁碟使用 `local` 驅動，並將其檔案存放在 `storage/app/public` 中。

如果你的 `public` 磁碟使用 `local` 驅動且想讓這些檔案可以從網路上存取，則應建立一個符號連結 (Symbolic Link)，從來源目錄 `storage/app/public` 指向目標目錄 `public/storage`：

若要建立符號連結，可使用 `storage:link` 這個 Artisan 命令：

```shell
php artisan storage:link
```

檔案儲存好且符號連結建立好之後，就可以使用 `asset` 輔助函式來為檔案建立 URL：

```php
echo asset('storage/file.txt');
```

可以在 `filesystems` 設定檔中設定額外的符號連結。當執行 `storage:link` 命令時，每個設定好的連結都會被建立：

```php
'links' => [
    public_path('storage') => storage_path('app/public'),
    public_path('images') => storage_path('app/images'),
],
```

`storage:unlink` 命令可用於銷毀已設定的符號連結：

```shell
php artisan storage:unlink
```

<a name="driver-prerequisites"></a>
### 驅動的必要條件

<a name="s3-driver-configuration"></a>
#### S3 驅動設定

在使用 S3 驅動前，需要先透過 Composer 套件管理員來安裝 Flysystem S3 套件：

```shell
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

S3 磁碟的設定陣列位於 `config/filesystems.php` 設定檔中。一般來說，應使用下列環境變數來設定 S3 的資訊與憑證。這些環境變數有在 `config/filesystems.php` 設定檔中被參考：

```ini
AWS_ACCESS_KEY_ID=<your-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=<your-bucket-name>
AWS_USE_PATH_STYLE_ENDPOINT=false
```

為了方便，這些環境變數的命名慣例與 AWS CLI 所使用的相同。

<a name="ftp-driver-configuration"></a>
#### FTP 驅動設定

在使用 FTP 驅動前，需要先透過 Composer 套件管理員來安裝 Flysystem FTP 套件：

```shell
composer require league/flysystem-ftp "^3.0"
```

Laravel 的 Flysystem 整合與 FTP 搭配得很好；不過，框架預設的 `config/filesystems.php` 設定檔中並未包含範例設定。若需要設定 FTP 檔案系統，可使用下列範例設定：

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
#### SFTP 驅動設定

在使用 SFTP 驅動前，需要先透過 Composer 套件管理員來安裝 Flysystem SFTP 套件：

```shell
composer require league/flysystem-sftp-v3 "^3.0"
```

Laravel 的 Flysystem 整合與 SFTP 搭配得很好；不過，框架預設的 `config/filesystems.php` 設定檔中並未包含範例設定。若需要設定 SFTP 檔案系統，可使用下列範例設定：

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
### 限定範圍與唯讀的檔案系統

「限定範圍 (Scoped)」磁碟可讓你在定義檔案系統時，讓所有路徑都自動加上給定的路徑前綴。在建立限定範圍的檔案系統磁碟前，需要先透過 Composer 套件管理員安裝一個額外的 Flysystem 套件：

```shell
composer require league/flysystem-path-prefixing "^3.0"
```

透過定義一個使用 `scoped` 驅動的磁碟，就可以為任何現有的檔案系統磁碟建立一個限定路徑範圍的實體。舉例來說，可以建立一個磁碟，將現有的 `s3` 磁碟限定在某個特定的路徑前綴下，之後所有使用這個限定範圍磁碟的檔案操作都會使用指定的前綴：

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

「唯讀 (Read-only)」磁碟可讓你建立不允許寫入操作的檔案系統磁碟。在使用 `read-only` 設定選項前，需要先透過 Composer 套件管理員安裝一個額外的 Flysystem 套件：

```shell
composer require league/flysystem-read-only "^3.0"
```

接著，就可以在一個或多個磁碟的設定陣列中包含 `read-only` 設定選項：

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```

<a name="amazon-s3-compatible-filesystems"></a>
### 與 Amazon S3 相容的檔案系統

預設情況下，應用程式的 `filesystems` 設定檔中包含了一個 `s3` 磁碟的設定。除了用這個磁碟來與 [Amazon S3](https://aws.amazon.com/s3/) 互動外，你也可以用它來與任何 S3 相容的檔案儲存服務互動，例如 [MinIO](https://github.com/minio/minio)、[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)、[Vultr Object Storage](https://www.vultr.com/products/object-storage/)、[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/)、或 [Hetzner Cloud Storage](https://www.hetzner.com/storage/object-storage/)。

通常，在更新磁碟憑證以符合要使用的服務憑證後，只需要更新 `endpoint` 設定選項的值。此選項的值通常是透過 `AWS_ENDPOINT` 環境變數來定義的：

```php
'endpoint' => env('AWS_ENDPOINT', 'https://minio:9000'),
```


<a name="minio"></a>
#### MinIO

為讓 Laravel 的 Flysystem 整合在使用 MinIO 時能產生正確的 URL，你應定義 `AWS_URL` 環境變數，使其符合應用程式的本機 URL 並在 URL 路徑中包含 Bucket 名稱：

```ini
AWS_URL=http://localhost:9000/local
```

> [!WARNING]
> 若用戶端無法存取 `endpoint`，則在使用 MinIO 時，透過 `temporaryUrl` 方法產生的暫時儲存 URL 可能會無法運作。

<a name="obtaining-disk-instances"></a>
## 取得磁碟實體

`Storage` facade 可用來與任何已設定的磁碟互動。例如，我們可以在這個 Facade 上使用 `put` 方法來將頭像儲存在預設磁碟上。若在呼叫 `Storage` facade 上的方法前未先呼叫 `disk` 方法，則該方法呼叫會被自動傳給預設磁碟：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('avatars/1', $content);
```

若應用程式會與多個磁碟互動，我們可以在 `Storage` facade 上使用 `disk` 方法來處理特定磁碟上的檔案：

```php
Storage::disk('s3')->put('avatars/1', $content);
```


<a name="on-demand-disks"></a>
### On-Demand 磁碟

有時候，我們可能會想在執行階段 (Runtime) 使用給定的設定來建立磁碟，而不需要將該設定實際寫在應用程式的 `filesystems` 設定檔中。若要這麼做，可以傳遞一個設定陣列到 `Storage` facade 的 `build` 方法：

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);

$disk->put('image.jpg', $content);
```

<a name="retrieving-files"></a>
## 取回檔案

`get` 方法可用於取回檔案的內容。該方法會回傳檔案的原始字串內容。請記得，所有的檔案路徑都應相對於該磁碟的「根 (root)」位置：

```php
$contents = Storage::get('file.jpg');
```

若要取回的檔案包含 JSON，可使用 `json` 方法來取回檔案並解碼其內容：

```php
$orders = Storage::json('orders.json');
```

`exists` 方法可用來判斷檔案是否存在於磁碟上：

```php
if (Storage::disk('s3')->exists('file.jpg')) {
    // ...
}
```

`missing` 方法可用來判斷檔案是否不存在於磁碟上：

```php
if (Storage::disk('s3')->missing('file.jpg')) {
    // ...
}
```


<a name="downloading-files"></a>
### 下載檔案

`download` 方法可用來產生一個會強制使用者瀏覽器下載指定路徑檔案的回應。`download` 方法的第二個引數可傳入一個檔名，用來決定使用者下載檔案時會看到的檔名。最後，還可以傳入一組 HTTP 標頭陣列作為該方法的第三個引數：

```php
return Storage::download('file.jpg');

return Storage::download('file.jpg', $name, $headers);
```


<a name="file-urls"></a>
### 檔案 URL

可使用 `url` 方法來取得指定檔案的 URL。若使用 `local` 驅動，則通常只會在給定的路徑前加上 `/storage`，並回傳該檔案的相對 URL。若使用 `s3` 驅動，則會回傳完整的遠端 URL：

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::url('file.jpg');
```

使用 `local` 驅動時，所有應可公開存取的檔案都應放在 `storage/app/public` 目錄下。此外，還應在 `public/storage` [建立一個指向 `storage/app/public` 目錄的符號連結 (Symbolic Link)](#the-public-disk)。

> [!WARNING]
> 使用 `local` 驅動時，`url` 的回傳值不會被 URL 編碼。因此，建議一律使用能建立有效 URL 的名稱來儲存檔案。


<a name="url-host-customization"></a>
#### URL 主機自訂

若想修改由 `Storage` Facade 產生的 URL 的主機，可在磁碟的設定陣列中新增或修改 `url` 選項：

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
### 暫時 URL

使用 `temporaryUrl` 方法，可以為使用 `local` 與 `s3` 驅動儲存的檔案建立暫時 URL。此方法可傳入一個路徑與一個用來指定 URL 到期時間的 `DateTime` 實體：

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::temporaryUrl(
    'file.jpg', now()->addMinutes(5)
);
```


<a name="enabling-local-temporary-urls"></a>
#### 啟用 Local 暫時 URL

如果在 `local` 驅動引入暫時 URL 支援前就已開始開發專案，則可能需要手動啟用 Local 的暫時 URL。為此，請在 `config/filesystems.php` 設定檔中，將 `serve` 選項加到 `local` 磁碟的設定陣列內：

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

若需要指定額外的 [S3 請求參數](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests)，可在 `temporaryUrl` 方法的第三個引數中傳入請求參數陣列：

```php
$url = Storage::temporaryUrl(
    'file.jpg',
    now()->addMinutes(5),
    [
        'ResponseContentType' => 'application/octet-stream',
        'ResponseContentDisposition' => 'attachment; filename=file2.jpg',
    ]
);
```


<a name="customizing-temporary-urls"></a>
#### 自訂暫時 URL

若需要為特定儲存磁碟自訂暫時 URL 的建立方式，可使用 `buildTemporaryUrlsUsing` 方法。舉例來說，若有一個 Controller 讓你能下載由不支援暫時 URL 的磁碟所儲存的檔案時，這個方法就很有用。一般來說，應在 Service Provider 的 `boot` 方法中呼叫此方法：

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
#### 暫時上傳 URL

> [!WARNING]
> 產生暫時上傳 URL 的功能僅 `s3` 驅動支援。

若需要產生一個暫時 URL，用來讓 Client 端的應用程式直接上傳檔案，可使用 `temporaryUploadUrl` 方法。此方法可傳入一個路徑與一個用來指定 URL 到期時間的 `DateTime` 實體。`temporaryUploadUrl` 方法會回傳一個關聯陣列，可將其解構為上傳 URL 與應包含在上傳請求中的標頭：

```php
use Illuminate\Support\Facades\Storage;

['url' => $url, 'headers' => $headers] = Storage::temporaryUploadUrl(
    'file.jpg', now()->addMinutes(5)
);
```

此方法主要適用於需要 Client 端應用程式直接將檔案上傳到雲端儲存系統 (如 Amazon S3) 的無伺服器 (Serverless) 環境中。


<a name="file-metadata"></a>
### 檔案元資料

除了讀寫檔案外，Laravel 也可以提供檔案本身的資訊。舉例來說，`size` 方法可用來取得檔案的大小 (單位為 Bytes)：

```php
use Illuminate\Support\Facades\Storage;

$size = Storage::size('file.jpg');
```

`lastModified` 方法會回傳該檔案最後修改時間的 UNIX 時間戳記：

```php
$time = Storage::lastModified('file.jpg');
```

可透過 `mimeType` 方法來取得指定檔案的 MIME 類型：

```php
$mime = Storage::mimeType('file.jpg');
```


<a name="file-paths"></a>
#### 檔案路徑

可使用 `path` 方法來取得指定檔案的路徑。若使用 `local` 驅動，則會回傳該檔案的絕對路徑。若使用 `s3` 驅動，則此方法會回傳該檔案在 S3 儲存桶中的相對路徑：

```php
use Illuminate\Support\Facades\Storage;

$path = Storage::path('file.jpg');
```

<a name="storing-files"></a>
## 儲存檔案

可使用 `put` 方法來將檔案內容儲存到磁碟上。也可以傳遞一個 PHP 的 `resource` 給 `put` 方法，這樣就會使用 Flysystem 底層的串流支援。請記得，所有的檔案路徑都應相對於該磁碟上設定的「根 (root)」位置：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents);

Storage::put('file.jpg', $resource);
```

<a name="failed-writes"></a>
#### 寫入失敗

若 `put` 方法 (或其他「寫入」操作) 無法將檔案寫入磁碟，則會回傳 `false`：

```php
if (! Storage::put('file.jpg', $contents)) {
    // The file could not be written to disk...
}
```

若有需要，也可以在檔案系統磁碟的設定陣列中定義 `throw` 選項。當該選項設為 `true` 時，像 `put` 這樣的「寫入」方法就會在寫入操作失敗時擲回一個 `League\Flysystem\UnableToWriteFile` 的實體：

```php
'public' => [
    'driver' => 'local',
    // ...
    'throw' => true,
],
```

<a name="prepending-appending-to-files"></a>
### 在檔案開頭與結尾寫入

`prepend` 與 `append` 方法可讓開發者在檔案的開頭或結尾寫入內容：

```php
Storage::prepend('file.log', 'Prepended Text');

Storage::append('file.log', 'Appended Text');
```

<a name="copying-moving-files"></a>
### 複製與移動檔案

`copy` 方法可用於將現有檔案複製到磁碟上的新位置，而 `move` 方法則可用於將現有檔案重新命名或移動到新位置：

```php
Storage::copy('old/file.jpg', 'new/file.jpg');

Storage::move('old/file.jpg', 'new/file.jpg');
```

<a name="automatic-streaming"></a>
### 自動串流

將檔案以串流的方式存到儲存空間能大幅降低記憶體用量。若想讓 Laravel 自動管理將特定檔案串流到儲存位置，可使用 `putFile` 或 `putFileAs` 方法。此方法接受 `Illuminate\Http\File` 或 `Illuminate\Http\UploadedFile` 的實體，並會自動將該檔案串流到想要的位置：

```php
use Illuminate\Http\File;
use Illuminate\Support\Facades\Storage;

// Automatically generate a unique ID for filename...
$path = Storage::putFile('photos', new File('/path/to/photo'));

// Manually specify a filename...
$path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');
```

關於 `putFile` 方法，有幾點很重要。請注意，我們只指定了目錄名稱，而沒有檔名。預設情況下，`putFile` 方法會產生一個唯一的 ID 來當作檔名。檔案的副檔名會透過檢查檔案的 MIME 類型來判斷。`putFile` 方法會回傳檔案的路徑，這樣就能將包含產生的檔名在內的路徑儲存到資料庫中。

`putFile` 與 `putFileAs` 方法也接受一個引數來指定儲存檔案的「可見性 (visibility)」。若要將檔案儲存到像 Amazon S3 這樣的雲端磁碟上，並希望能透過產生的 URL 公開存取該檔案，這個功能就特別有用：

```php
Storage::putFile('photos', new File('/path/to/photo'), 'public');
```

<a name="file-uploads"></a>
### 檔案上傳

在 Web 應用程式中，最常見的檔案儲存使用情境之一就是儲存使用者上傳的檔案，如相片和文件。Laravel 中，只要使用上傳檔案實體上的 `store` 方法，就能輕鬆儲存上傳的檔案。呼叫 `store` 方法時，傳入要儲存該上傳檔案的路徑即可：

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

關於這個範例，有幾點很重要。請注意，我們只指定了目錄名稱，而沒有檔名。預設情況下，`store` 方法會產生一個唯一的 ID 來當作檔名。檔案的副檔名會透過檢查檔案的 MIME 類型來判斷。`store` 方法會回傳檔案的路徑，這樣就能將包含產生的檔名在內的路徑儲存到資料庫中。

也可以在 `Storage` Facade 上呼叫 `putFile` 方法來執行與上例相同的檔案儲存操作：

```php
$path = Storage::putFile('avatars', $request->file('avatar'));
```

<a name="specifying-a-file-name"></a>
#### 指定檔名

若不想要讓儲存的檔案被自動指派檔名，可使用 `storeAs` 方法。該方法會接收路徑、檔名、與 (可選的) 磁碟作為其引數：

```php
$path = $request->file('avatar')->storeAs(
    'avatars', $request->user()->id
);
```

也可以在 `Storage` Facade 上使用 `putFileAs` 方法，該方法會執行與上例相同的檔案儲存操作：

```php
$path = Storage::putFileAs(
    'avatars', $request->file('avatar'), $request->user()->id
);
```

> [!WARNING]
> 檔案路徑中，不可列印與無效的 Unicode 字元會被自動移除。因此，建議在將檔案路徑傳給 Laravel 的檔案儲存方法前先過濾 (Sanitize) 過這些路徑。檔案路徑是使用 `League\Flysystem\WhitespacePathNormalizer::normalizePath` 方法來正規化的。

<a name="specifying-a-disk"></a>
#### 指定磁碟

預設情況下，上傳檔案的 `store` 方法會使用預設磁碟。若想指定另一個磁碟，可將磁碟名稱作為第二個引數傳給 `store` 方法：

```php
$path = $request->file('avatar')->store(
    'avatars/'.$request->user()->id, 's3'
);
```

若使用 `storeAs` 方法，則可將磁碟名稱作為第三個引數傳給該方法：

```php
$path = $request->file('avatar')->storeAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="other-uploaded-file-information"></a>
#### 其他上傳檔案資訊

若想取得上傳檔案的原始檔名與副檔名，可使用 `getClientOriginalName` 與 `getClientOriginalExtension` 方法：

```php
$file = $request->file('avatar');

$name = $file->getClientOriginalName();
$extension = $file->getClientOriginalExtension();
```

不過請注意，`getClientOriginalName` 與 `getClientOriginalExtension` 方法被認為是不安全的，因為檔名與副檔名可能會被惡意使用者竄改。因此，通常應優先使用 `hashName` 與 `extension` 方法來取得該上傳檔案的檔名與副檔名：

```php
$file = $request->file('avatar');

$name = $file->hashName(); // Generate a unique, random name...
$extension = $file->extension(); // Determine the file's extension based on the file's MIME type...
```

<a name="file-visibility"></a>
### 檔案可見性

在 Laravel 的 Flysystem 整合中，「可見性 (Visibility)」是一個跨多個平台對檔案權限的抽象化。檔案可被宣告為 `public` 或 `private`。當檔案被宣告為 `public` 時，代表該檔案通常應能讓其他人存取。舉例來說，在使用 S3 驅動時，就可以取回 `public` 檔案的 URL。

我們可以在寫入檔案時，通過 `put` 方法來設定可見性：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents, 'public');
```

若檔案已儲存，則可通過 `getVisibility` 與 `setVisibility` 方法來取回並設定其可見性：

```php
$visibility = Storage::getVisibility('file.jpg');

Storage::setVisibility('file.jpg', 'public');
```

與上傳檔案互動時，可使用 `storePublicly` 與 `storePubliclyAs` 方法來以上傳 `public` 可見性的檔案：

```php
$path = $request->file('avatar')->storePublicly('avatars', 's3');

$path = $request->file('avatar')->storePubliclyAs(
    'avatars',
    $request->user()->id,
    's3'
);
```


<a name="local-files-and-visibility"></a>
#### Local 檔案與可見性

使用 `local` 驅動時，`public` [可見性](#file-visibility) 會被轉為目錄的 `0755` 權限與檔案的 `0644` 權限。我們可以在專案的 `filesystems` 設定檔中修改權限的對應關係：

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

`delete` 方法可接受單一檔名或一組檔名陣列來刪除檔案：

```php
use Illuminate\Support\Facades\Storage;

Storage::delete('file.jpg');

Storage::delete(['file.jpg', 'file2.jpg']);
```

若有需要，也可以指定要從哪個磁碟刪除檔案：

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('s3')->delete('path/file.jpg');
```


<a name="directories"></a>
## 目錄


<a name="get-all-files-within-a-directory"></a>
#### 取得目錄中的所有檔案

`files` 方法會回傳給定目錄中所有檔案的陣列。若想取回給定目錄中包含子目錄在內的所有檔案清單，可改用 `allFiles` 方法：

```php
use Illuminate\Support\Facades\Storage;

$files = Storage::files($directory);

$files = Storage::allFiles($directory);
```


<a name="get-all-directories-within-a-directory"></a>
#### 取得目錄中的所有目錄

`directories` 方法會回傳給定目錄中所有目錄的陣列。若想取回給定目錄中包含子目錄在內的所有目錄清單，可改用 `allDirectories` 方法：

```php
$directories = Storage::directories($directory);

$directories = Storage::allDirectories($directory);
```


<a name="create-a-directory"></a>
#### 建立目錄

`makeDirectory` 方法會建立給定的目錄，其中也包含所有需要的子目錄：

```php
Storage::makeDirectory($directory);
```


<a name="delete-a-directory"></a>
#### 刪除目錄

最後，`deleteDirectory` 方法可用來移除一個目錄與其中所有的檔案：

```php
Storage::deleteDirectory($directory);
```


<a name="testing"></a>
## 測試

`Storage` Facade 的 `fake` 方法能讓我們輕鬆地產生假的磁碟。搭配 `Illuminate\Http\UploadedFile` 類別的檔案產生工具函式，就能大幅簡化檔案上傳的測試。例如：

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

預設情況下，`fake` 方法會刪除其暫存目錄中的所有檔案。若想保留這些檔案，可改用「persistentFake」方法。更多有關測試檔案上傳的資訊，請參考 [HTTP 測試文件中關於檔案上傳的說明](/docs/{{version}}/http-tests#testing-file-uploads)。

> [!WARNING]
> `image` 方法需要啟用 [GD 擴充套件](https://www.php.net/manual/en/book.image.php)。


<a name="custom-filesystems"></a>
## 自訂檔案系統

Laravel 的 Flysystem 整合提供了好幾種內建的「驅動(Driver)」。不過，Flysystem 不僅限於這些，還提供了許多其他儲存系統的轉接器(Adapter)。若想在 Laravel 應用程式中使用這些額外的轉接器，可以建立一個自訂驅動。

為了定義自訂檔案系統，我們會需要一個 Flysystem 轉接器。讓我們先在專案中加入一個社群維護的 Dropbox 轉接器：

```shell
composer require spatie/flysystem-dropbox
```

接著，我們可以在應用程式中其中一個 [Service Provider](/docs/{{version}}/providers) 的 `boot` 方法內註冊這個驅動。為此，我們應使用 `Storage` Facade 的 `extend` 方法：

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

`extend` 方法的第一個引數是驅動的名稱，而第二個引數是個會收到 `$app` 與 `$config` 變數的閉包 (Closure)。這個閉包必須回傳一個 `Illuminate\Filesystem\FilesystemAdapter` 的實體。`$config` 變數包含了在 `config/filesystems.php` 內為指定磁碟所定義的值。

在建立並註冊完擴充套件的 Service Provider 後，就可以在 `config/filesystems.php` 設定檔中使用 `dropbox` 驅動了。