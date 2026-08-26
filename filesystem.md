# 檔案儲存

- [簡介](#introduction)
- [設定](#configuration)
    - [Local 驅動器](#the-local-driver)
    - [Public 磁碟](#the-public-disk)
    - [驅動器前置需求](#driver-prerequisites)
    - [作用域、唯讀與透讀檔案系統](#scoped-and-read-only-filesystems)
    - [相容 Amazon S3 的檔案系統](#amazon-s3-compatible-filesystems)
- [取得磁碟實例](#obtaining-disk-instances)
    - [隨需磁碟](#on-demand-disks)
- [取得檔案](#retrieving-files)
    - [下載檔案](#downloading-files)
    - [檔案 URL](#file-urls)
    - [臨時 URL](#temporary-urls)
    - [檔案 Metadata](#file-metadata)
- [儲存檔案](#storing-files)
    - [在檔案開頭與結尾附加內容](#prepending-appending-to-files)
    - [複製與移動檔案](#copying-moving-files)
    - [自動串流](#automatic-streaming)
    - [檔案上傳](#file-uploads)
    - [檔案能見度](#file-visibility)
    - [圖片處理](#image-manipulation)
- [刪除檔案](#deleting-files)
- [目錄](#directories)
- [測試](#testing)
- [自訂檔案系統](#custom-filesystems)

<a name="introduction"></a>
## 簡介

感謝 Frank de Jonge 所開發的優秀 [Flysystem](https://github.com/thephpleague/flysystem) PHP 套件，Laravel 提供了強大的檔案系統抽象層。Laravel 的 Flysystem 整合提供了簡單的驅動器來操作本地檔案系統、SFTP 以及 Amazon S3。更棒的是，由於每個系統的 API 都保持一致，因此在本地開發機器與正式伺服器之間切換這些儲存選項變得極為簡單。

<a name="configuration"></a>
## 設定

Laravel 的檔案系統設定檔位於 `config/filesystems.php`。在此檔案中，你可以設定所有的檔案系統「磁碟 (disk)」。每個磁碟代表特定的儲存驅動器與儲存位置。設定檔中包含了每個支援驅動器的範例設定，因此你可以修改設定以反映你的儲存偏好與憑證。

`local` 驅動器用於處理儲存於執行 Laravel 應用程式伺服器本機上的檔案，而 `sftp` 儲存驅動器則用於基於 SSH 金鑰的 FTP。`s3` 驅動器則是用來寫入 Amazon 的 S3 雲端儲存服務。

> [!NOTE]
> 你可以根據需求設定任意數量的磁碟，甚至可以有多個使用相同驅動器的磁碟。


<a name="the-local-driver"></a>
### Local 驅動器

使用 `local` 驅動器時，所有檔案操作都是相對於 `filesystems` 設定檔中定義的 `root` 目錄。預設情況下，此值設定為 `storage/app/private` 目錄。因此，以下方法將會寫入至 `storage/app/private/example.txt`：

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('local')->put('example.txt', 'Contents');
```


<a name="the-public-disk"></a>
### Public 磁碟

包含在應用程式 `filesystems` 設定檔中的 `public` 磁碟是用於可以公開存取的檔案。預設情況下，`public` 磁碟使用 `local` 驅動器，並將其檔案儲存在 `storage/app/public`。

若你的 `public` 磁碟使用 `local` 驅動器，且你想讓這些檔案可以透過網路公開存取，則應該建立一個從來源目錄 `storage/app/public` 指向目標目錄 `public/storage` 的符號連結：

要建立符號連結，你可以使用 `storage:link` Artisan 指令：

```shell
php artisan storage:link
```

檔案儲存且符號連結建立完成後，你可以使用 `asset` 輔助函式來建立檔案的 URL：

```php
echo asset('storage/file.txt');
```

你可以在 `filesystems` 設定檔中設定額外的符號連結。當你執行 `storage:link` 指令時，每個設定好的連結都會被建立：

```php
'links' => [
    public_path('storage') => storage_path('app/public'),
    public_path('images') => storage_path('app/images'),
],
```

`storage:unlink` 指令可用於刪除你所設定的符號連結：

```shell
php artisan storage:unlink
```


<a name="driver-prerequisites"></a>
### 驅動器前置需求


<a name="s3-driver-configuration"></a>
#### S3 驅動器設定

在使用 S3 驅動器之前，你需要透過 Composer 套件包管理器安裝 Flysystem S3 套件：

```shell
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

S3 磁碟設定陣列位於你的 `config/filesystems.php` 設定檔中。通常，你應該使用以下由 `config/filesystems.php` 設定檔所引用的環境變數來設定你的 S3 資訊與憑證：

```ini
AWS_ACCESS_KEY_ID=<your-key-id>
AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=<your-bucket-name>
AWS_USE_PATH_STYLE_ENDPOINT=false
```

為求方便，這些環境變數符合 AWS CLI 所使用的命名慣例。


<a name="ftp-driver-configuration"></a>
#### FTP 驅動器設定

在使用 FTP 驅動器之前，你需要透過 Composer 套件包管理器安裝 Flysystem FTP 套件：

```shell
composer require league/flysystem-ftp "^3.0"
```

Laravel 的 Flysystem 整合非常適合與 FTP 搭配使用；然而，框架預設的 `config/filesystems.php` 設定檔中並未包含範例設定。如果你需要設定 FTP 檔案系統，可以使用下方的範例設定：

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
#### SFTP 驅動器設定

在使用 SFTP 驅動器之前，你需要透過 Composer 套件包管理器安裝 Flysystem SFTP 套件：

```shell
composer require league/flysystem-sftp-v3 "^3.0"
```

Laravel 的 Flysystem 整合非常適合與 SFTP 搭配使用；然而，框架預設的 `config/filesystems.php` 設定檔中並未包含範例設定。如果你需要設定 SFTP 檔案系統，可以使用下方的範例設定：

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
### 作用域、唯讀與透讀檔案系統

作用域 (Scoped) 磁碟允許你定義一個檔案系統，其中所有的路徑都會自動加上指定的路徑前綴。在建立作用域檔案系統磁碟之前，你需要透過 Composer 套件包管理器安裝額外的 Flysystem 套件：

```shell
composer require league/flysystem-path-prefixing "^3.0"
```

你可以透過定義一個使用 `scoped` 驅動器的磁碟，來為任何現有的檔案系統磁碟建立一個路徑作用域實例。例如，你可以建立一個磁碟，將現有的 `s3` 磁碟限制在特定的路徑前綴下，隨後使用該作用域磁碟進行的每個檔案操作都將使用指定的字頭：

```php
's3-videos' => [
    'driver' => 'scoped',
    'disk' => 's3',
    'prefix' => 'path/to/videos',
],
```

「唯讀 (Read-only)」磁碟允許你建立不允許寫入操作的檔案系統磁碟。在使用 `read-only` 設定選項之前，你需要透過 Composer 套件包管理器安裝額外的 Flysystem 套件：

```shell
composer require league/flysystem-read-only "^3.0"
```

接著，你可以在一個或多個磁碟的設定陣列中包含 `read-only` 設定選項：

```php
's3-videos' => [
    'driver' => 's3',
    // ...
    'read-only' => true,
],
```

透讀 (Read-through) 磁碟允許你在不停機的情況下於磁碟之間遷移檔案。當讀取檔案時，Laravel 會先檢查主要磁碟。如果檔案僅存在於備用 (fallback) 磁碟上，Laravel 會從備用磁碟讀取該檔案，並將其複製到主要磁碟，供未來的請求使用：

```php
'assets' => [
    'driver' => 'read-through',
    'primary' => 's3',
    'fallback' => 'legacy-s3',
],
```

寫入和目錄列表操作以主要磁碟為目標。檔案是否存在以及 Metadata 的檢查會使用任一磁碟，且不會將檔案複製到主要磁碟。如果將備用檔案複製到主要磁碟失敗，預設情況下讀取操作仍然會成功。若要改為拋出例外，請將 `throw_on_promotion_failure` 設定選項設為 `true`。

<a name="amazon-s3-compatible-filesystems"></a>
### 相容 Amazon S3 的檔案系統

預設情況下，您應用程式的 `filesystems` 設定檔包含了一個針對 `s3` 磁碟的設定。除了使用此磁碟與 [Amazon S3](https://aws.amazon.com/s3/) 互動外，您也可以使用它來與任何相容於 S3 的檔案儲存服務進行互動，例如 [RustFS](https://github.com/rustfs/rustfs)、[DigitalOcean Spaces](https://www.digitalocean.com/products/spaces/)、[Vultr Object Storage](https://www.vultr.com/products/object-storage/)、[Cloudflare R2](https://www.cloudflare.com/developer-platform/products/r2/) 或 [Hetzner Cloud Storage](https://www.hetzner.com/storage/object-storage/)。

通常，在更新磁碟的憑證以符合您打算使用的服務憑證後，您只需要更新 `endpoint` 設定選項的值即可。該選項的值通常是透過 `AWS_ENDPOINT` 環境變數來定義：

```php
'endpoint' => env('AWS_ENDPOINT', 'https://rustfs:9000'),
```

<a name="obtaining-disk-instances"></a>
## 取得磁碟實例

`Storage` Facade 可用於與任何已設定的磁碟進行互動。例如，你可以使用 Facade 上的 `put` 方法將頭像儲存至預設磁碟。如果你在呼叫 `Storage` Facade 的方法時沒有先呼叫 `disk` 方法，該方法將會自動傳送給預設磁碟：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('avatars/1', $content);
```

如果你的應用程式會與多個磁碟互動，你可以使用 `Storage` Facade 上的 `disk` 方法來對特定磁碟上的檔案進行操作：

```php
Storage::disk('s3')->put('avatars/1', $content);
```

<a name="on-demand-disks"></a>
### 隨需磁碟

有時候你可能希望在執行階段使用給定的設定建立磁碟，而無須將該設定實際寫入應用程式的 `filesystems` 設定檔中。若要達到這個目的，你可以將設定陣列傳遞給 `Storage` Facade 的 `build` 方法：

```php
use Illuminate\Support\Facades\Storage;

$disk = Storage::build([
    'driver' => 'local',
    'root' => '/path/to/root',
]);

$disk->put('image.jpg', $content);
```

<a name="retrieving-files"></a>
## 取得檔案

可以使用 `get` 方法來取得檔案的內容。該方法將會回傳檔案的原生字串內容。請記住，所有檔案路徑都應該相對於磁碟設定的「root」位置來指定：

```php
$contents = Storage::get('file.jpg');
```

若您要取得的檔案包含 JSON，可以使用 `json` 方法來取得檔案並解碼其內容：

```php
$orders = Storage::json('orders.json');
```

可以使用 `exists` 方法來確認檔案是否存在於磁碟上：

```php
if (Storage::disk('s3')->exists('file.jpg')) {
    // ...
}
```

可以使用 `missing` 方法來確認檔案是否不存在於磁碟上：

```php
if (Storage::disk('s3')->missing('file.jpg')) {
    // ...
}
```


<a name="downloading-files"></a>
### 下載檔案

可以使用 `download` 方法來產生一個回應，強制使用者的瀏覽器下載指定路徑的檔案。`download` 方法接受檔名作為其第二個引數，這將決定下載該檔案的使用者所看到的檔名。最後，您可以傳入一個 HTTP 標頭陣列作為該方法的第三個引數：

```php
return Storage::download('file.jpg');

return Storage::download('file.jpg', $name, $headers);
```


<a name="file-urls"></a>
### 檔案 URL

您可以使用 `url` 方法來取得指定檔案的 URL。若您使用的是 `local` 驅動器，這通常只會在給定的路徑前加上 `/storage`，並回傳檔案的相對 URL。若您使用的是 `s3` 驅動器，則會回傳完整的遠端 URL：

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::url('file.jpg');
```

使用 `local` 驅動器時，所有應該能公開存取的檔案都應放在 `storage/app/public` 目錄中。此外，您應該在 `public/storage` [建立符號連結](#the-public-disk)，並指向 `storage/app/public` 目錄。

> [!WARNING]
> 使用 `local` 驅動器時，`url` 的回傳值不會經過 URL 編碼。因此，我們建議您始終使用能產生有效 URL 的名稱來儲存檔案。


<a name="url-host-customization"></a>
#### URL 主機自訂

若您想修改使用 `Storage` Facade 產生的 URL 主機，可以在磁碟的設定陣列中新增或修改 `url` 選項：

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
### 臨時 URL

使用 `temporaryUrl` 方法，您可以為透過 `local` 與 `s3` 驅動器儲存的檔案建立臨時 URL。此方法接受一個路徑和一個指定 URL 何時過期的 `DateTime` 實例：

```php
use Illuminate\Support\Facades\Storage;

$url = Storage::temporaryUrl(
    'file.jpg', now()->plus(minutes: 5)
);
```


<a name="enabling-local-temporary-urls"></a>
#### 啟用本地臨時 URL

若您在 `local` 驅動器支援臨時 URL 之前就已開始開發應用程式，則可能需要啟用本地臨時 URL。為此，請在 `config/filesystems.php` 設定檔中的 `local` 磁碟設定陣列加入 `serve` 選項：

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

若您需要指定額外的 [S3 請求參數](https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectGET.html#RESTObjectGET-requests)，可以將請求參數陣列作為第三個引數傳給 `temporaryUrl` 方法：

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
#### 自訂臨時 URL

若您需要自訂特定儲存磁碟建立臨時 URL 的方式，可以使用 `buildTemporaryUrlsUsing` 方法。例如，當您有一個控制器允許下載透過通常不支援臨時 URL 的磁碟所儲存的檔案時，這個功能就非常有用。通常，此方法應該在服務提供者(Service Providers)的 `boot` 方法中呼叫：

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
#### 臨時上傳 URL

> [!WARNING]
> 產生臨時上傳 URL 的功能僅受 `s3` 與 `local` 驅動器支援。

若您需要產生一個可用於直接從用戶端應用程式上傳檔案的臨時 URL，可以使用 `temporaryUploadUrl` 方法。該方法接受一個路徑與一個指定 URL 何時過期的 `DateTime` 實例。`temporaryUploadUrl` 方法會回傳一個關聯陣列，您可以解構該陣列以取得上傳 URL 以及上傳請求中應包含的標頭：

```php
use Illuminate\Support\Facades\Storage;

['url' => $url, 'headers' => $headers] = Storage::temporaryUploadUrl(
    'file.jpg', now()->plus(minutes: 5)
);
```

這個方法主要用於無伺服器環境中，此類環境需要用戶端應用程式直接將檔案上傳到 Amazon S3 等雲端儲存系統。


<a name="file-metadata"></a>
### 檔案 Metadata

除了讀取和寫入檔案外，Laravel 還能提供有關檔案本身的資訊。例如，可以使用 `size` 方法來取得檔案的大小（以位元組為單位）：

```php
use Illuminate\Support\Facades\Storage;

$size = Storage::size('file.jpg');
```

`lastModified` 方法回傳檔案上次被修改時的 UNIX 時間戳記：

```php
$time = Storage::lastModified('file.jpg');
```

給定檔案的 MIME 類型可以透過 `mimeType` 方法取得：

```php
$mime = Storage::mimeType('file.jpg');
```


<a name="file-paths"></a>
#### 檔案路徑

您可以使用 `path` 方法來取得指定檔案的路徑。若您使用的是 `local` 驅動器，這將會回傳檔案的絕對路徑。若您使用的是 `s3` 驅動器，此方法將會回傳檔案在 S3 Bucket 中的相對路徑：

```php
use Illuminate\Support\Facades\Storage;

$path = Storage::path('file.jpg');
```

<a name="storing-files"></a>
## 儲存檔案

`put` 方法可以用來將檔案內容儲存到磁碟上。您也可以傳遞 PHP `resource` 給 `put` 方法，這會使用 Flysystem 底層的串流支援。請記住，所有檔案路徑都應該相對於該磁碟設定的「root」位置：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents);

Storage::put('file.jpg', $resource);
```

<a name="failed-writes"></a>
#### 寫入失敗

如果 `put` 方法（或其他「寫入」操作）無法將檔案寫入磁碟，將會回傳 `false`：

```php
if (! Storage::put('file.jpg', $contents)) {
    // The file could not be written to disk...
}
```

若您有需要，可以在檔案系統磁碟的設定陣列中定義 `throw` 選項。當此選項定義為 `true` 時，每當寫入操作失敗，像 `put` 這類的「寫入」方法就會拋出 `League\Flysystem\UnableToWriteFile` 的實例：

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

`copy` 方法可用於將現有檔案複製到磁碟上的新位置，而 `move` 方法則可用於將現有檔案重新命名或移動到新位置：

```php
Storage::copy('old/file.jpg', 'new/file.jpg');

Storage::move('old/file.jpg', 'new/file.jpg');
```

<a name="automatic-streaming"></a>
### 自動串流

將檔案以串流方式傳輸到儲存空間可以顯著減少記憶體使用量。如果您希望 Laravel 自動管理將指定檔案串流傳輸至儲存位置，可以使用 `putFile` 或 `putFileAs` 方法。此方法接收 `Illuminate\Http\File` 或 `Illuminate\Http\UploadedFile` 實例，並會自動將檔案串流傳輸至您期望的位置：

```php
use Illuminate\Http\File;
use Illuminate\Support\Facades\Storage;

// Automatically generate a unique ID for filename...
$path = Storage::putFile('photos', new File('/path/to/photo'));

// Manually specify a filename...
$path = Storage::putFileAs('photos', new File('/path/to/photo'), 'photo.jpg');
```

關於 `putFile` 方法，有幾個注意事項需要提醒。請注意，我們只指定了目錄名稱而非檔名。預設情況下，`putFile` 方法會產生一個唯一的 ID 來作為檔名。檔案的副檔名將透過檢視檔案的 MIME 類型來決定。`putFile` 方法會回傳該檔案的路徑，因此您可以將包含產生的檔名在內的路徑儲存至資料庫中。

`putFile` 與 `putFileAs` 方法也接受一個用於指定已儲存檔案「能見度 (Visibility)」的引數。當您將檔案儲存在 Amazon S3 等雲端磁碟，並希望該檔案能透過產生的 URL 公開存取時，這特別有用：

```php
Storage::putFile('photos', new File('/path/to/photo'), 'public');
```

<a name="file-uploads"></a>
### 檔案上傳

在 Web 應用程式中，儲存檔案最常見的使用情境之一就是儲存使用者上傳的檔案，例如照片與文件。Laravel 透過上傳檔案實例上的 `store` 方法，讓您能非常輕鬆地儲存上傳的檔案。請呼叫 `store` 方法並傳入您希望儲存上傳檔案的路徑：

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

關於這個範例，有幾個注意事項需要提醒。請注意，我們只指定了目錄名稱，而不是檔名。預設情況下，`store` 方法會產生一個唯一的 ID 來作為檔名。檔案的副檔名將透過檢視檔案的 MIME 類型來決定。`store` 方法會回傳該檔案的路徑，因此您可以將包含產生的檔名在內的路徑儲存至資料庫中。

您也可以呼叫 `Storage` Facade 上的 `putFile` 方法來執行與上述範例相同的檔案儲存操作：

```php
$path = Storage::putFile('avatars', $request->file('avatar'));
```

<a name="specifying-a-file-name"></a>
#### 指定檔名

如果您不希望系統自動為儲存的檔案分配檔名，您可以使用 `storeAs` 方法，該方法接收路徑、檔名以及（可選的）磁碟作為其引數：

```php
$path = $request->file('avatar')->storeAs(
    'avatars', $request->user()->id
);
```

您也可以使用 `Storage` Facade 上的 `putFileAs` 方法，這會執行與上述範例相同的檔案儲存操作：

```php
$path = Storage::putFileAs(
    'avatars', $request->file('avatar'), $request->user()->id
);
```

> [!WARNING]
> 無法列印及無效的 Unicode 字元會自動從檔案路徑中移除。因此，您可能需要在將檔案路徑傳遞給 Laravel 的檔案儲存方法之前進行淨化 (Sanitize)。檔案路徑是使用 `League\Flysystem\WhitespacePathNormalizer::normalizePath` 方法進行標準化 (Normalize) 的。

<a name="specifying-a-disk"></a>
#### 指定磁碟

預設情況下，上傳檔案的 `store` 方法會使用您的預設磁碟。如果您想要指定其他磁碟，請將磁碟名稱作為第二個引數傳給 `store` 方法：

```php
$path = $request->file('avatar')->store(
    'avatars/'.$request->user()->id, 's3'
);
```

如果您使用的是 `storeAs` 方法，可以將磁碟名稱作為第三個引數傳給該方法：

```php
$path = $request->file('avatar')->storeAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="other-uploaded-file-information"></a>
#### 其他上傳檔案資訊

如果您想要取得上傳檔案的原始名稱與副檔名，可以使用 `getClientOriginalName` 與 `getClientOriginalExtension` 方法：

```php
$file = $request->file('avatar');

$name = $file->getClientOriginalName();
$extension = $file->getClientOriginalExtension();
```

不過請記住，`getClientOriginalName` 和 `getClientOriginalExtension` 方法被認為是不安全的，因為檔名和副檔名可能會被惡意使用者篡改。基於這個原因，您通常應該優先使用 `hashName` 和 `extension` 方法來取得該上傳檔案的名稱與副檔名：

```php
$file = $request->file('avatar');

$name = $file->hashName(); // Generate a unique, random name...
$extension = $file->extension(); // Determine the file's extension based on the file's MIME type...
```

<a name="file-visibility"></a>
### 檔案能見度

在 Laravel 的 Flysystem 整合中，「能見度 (Visibility)」是跨多個平台的檔案權限抽象化。檔案可以宣告為 `public` 或 `private`。當檔案宣告為 `public` 時，代表您指示該檔案通常可以供其他人存取。例如，當使用 S3 驅動器時，您可以為 `public` 檔案取得 URL。

您可以在透過 `put` 方法寫入檔案時設定能見度：

```php
use Illuminate\Support\Facades\Storage;

Storage::put('file.jpg', $contents, 'public');
```

如果檔案已經儲存，可以透過 `getVisibility` 與 `setVisibility` 方法來取得與設定其能見度：

```php
$visibility = Storage::getVisibility('file.jpg');

Storage::setVisibility('file.jpg', 'public');
```

在處理上傳檔案時，您可以使用 `storePublicly` 與 `storePubliclyAs` 方法將上傳的檔案儲存為 `public` 能見度：

```php
$path = $request->file('avatar')->storePublicly('avatars', 's3');

$path = $request->file('avatar')->storePubliclyAs(
    'avatars',
    $request->user()->id,
    's3'
);
```

<a name="image-manipulation"></a>
### 圖片處理

如果您需要在儲存上傳的圖片之前對其進行縮放、裁切或轉換，您可以使用 Laravel 的 [圖片處理功能](/docs/{{version}}/images)：

```php
$path = $request->image('avatar')
    ->cover(400, 400)
    ->toWebp()
    ->storePublicly('avatars', 'public');
```

您也可以從已經儲存在某個檔案系統磁碟上的檔案來建立圖片實例：

```php
$image = Storage::disk('public')->image('avatars/photo.jpg');
```


<a name="local-files-and-visibility"></a>
#### 本地檔案與能見度

當使用 `local` 驅動器時，`public` [能見度](#file-visibility) 會轉換為目錄的 `0755` 權限與檔案的 `0644` 權限。您可以在應用程式的 `filesystems` 設定檔中修改這些權限對映：

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

`delete` 方法接受單一檔名或包含多個檔案的陣列來進行刪除：

```php
use Illuminate\Support\Facades\Storage;

Storage::delete('file.jpg');

Storage::delete(['file.jpg', 'file2.jpg']);
```

必要時，您可以指定要從哪個磁碟刪除檔案：

```php
use Illuminate\Support\Facades\Storage;

Storage::disk('s3')->delete('path/file.jpg');
```


<a name="directories"></a>
## 目錄


<a name="get-all-files-within-a-directory"></a>
#### 取得目錄內的所有檔案

`files` 方法會回傳給定目錄內所有檔案的陣列。如果您想取得包含子目錄在內的所有檔案清單，可以使用 `allFiles` 方法：

```php
use Illuminate\Support\Facades\Storage;

$files = Storage::files($directory);

$files = Storage::allFiles($directory);
```


<a name="get-all-directories-within-a-directory"></a>
#### 取得目錄內的所有目錄

`directories` 方法會回傳給定目錄內所有目錄的陣列。如果您想取得包含子目錄在內的所有目錄清單，可以使用 `allDirectories` 方法：

```php
$directories = Storage::directories($directory);

$directories = Storage::allDirectories($directory);
```


<a name="create-a-directory"></a>
#### 建立目錄

`makeDirectory` 方法會建立指定的目錄，包含所有需要的子目錄：

```php
Storage::makeDirectory($directory);
```


<a name="delete-a-directory"></a>
#### 刪除目錄

最後，`deleteDirectory` 方法可用於移除目錄及其包含的所有檔案：

```php
Storage::deleteDirectory($directory);
```


<a name="testing"></a>
## 測試

`Storage` Facade 的 `fake` 方法讓您可以輕鬆生成一個虛擬磁碟，搭配 `Illuminate\Http\UploadedFile` 類別的檔案生成公用工具，能極大地簡化檔案上傳的測試。例如：

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

    // Assert that the disk contains no files...
    Storage::disk('photos')->assertEmpty();
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

        // Assert that the disk contains no files...
        Storage::disk('photos')->assertEmpty();
    }
}
```

預設情況下，`fake` 方法會刪除其暫存目錄中的所有檔案。如果您想保留這些檔案，可以改用 "persistentFake" 方法。關於測試檔案上傳的更多資訊，您可以參考 [HTTP 測試文件的檔案上傳說明](/docs/{{version}}/http-tests#testing-file-uploads)。

> [!WARNING]
> `image` 方法需要安裝 [GD 擴充套件](https://www.php.net/manual/en/book.image.php)。


<a name="custom-filesystems"></a>
## 自訂檔案系統

Laravel 的 Flysystem 整合開箱即支援多種「驅動器」；不過，Flysystem 不僅限於這些，還擁有許多其他儲存系統的轉接器 (Adapter)。如果您想在 Laravel 應用程式中使用這些額外的轉接器，可以建立自訂驅動器。

為了定義自訂檔案系統，您需要一個 Flysystem 轉接器。讓我們將社群維護的 Dropbox 轉接器新增到專案中：

```shell
composer require spatie/flysystem-dropbox
```

接下來，您可以在應用程式的 [服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中註冊該驅動器。為此，您應該使用 `Storage` Facade 的 `extend` 方法：

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

`extend` 方法的第一個引數是驅動器的名稱，第二個則是接收 `$app` 和 `$config` 變數的 Closure。該 Closure 必須回傳 `Illuminate\Filesystem\FilesystemAdapter` 的實例。`$config` 變數包含了在 `config/filesystems.php` 中為指定磁碟所定義的值。

一旦建立並註冊了擴充套件的服務提供者，您就可以在 `config/filesystems.php` 設定檔中使用 `dropbox` 驅動器。