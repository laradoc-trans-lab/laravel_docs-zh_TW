# 設定

- [簡介](#introduction)
- [環境設定](#environment-configuration)
    - [環境變數類型](#environment-variable-types)
    - [取得環境設定](#retrieving-environment-configuration)
    - [判斷目前環境](#determining-the-current-environment)
    - [加密環境檔案](#encrypting-environment-files)
- [讀取設定值](#accessing-configuration-values)
- [設定快取](#configuration-caching)
- [發布設定](#configuration-publishing)
- [除錯模式](#debug-mode)
- [維護模式](#maintenance-mode)

<a name="introduction"></a>
## 簡介

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有文件說明，因此請隨意查看這些檔案，並熟悉可用的選項。

這些設定檔可讓您設定資料庫連線資訊、郵件伺服器資訊，以及各種其他核心設定值，例如應用程式 URL 和加密金鑰。


<a name="the-about-command"></a>
#### `about` 指令

Laravel 可以透過 `about` Artisan 指令顯示應用程式的設定、驅動程式和環境的概覽。

```shell
php artisan about
```

如果您只對應用程式概覽輸出的特定區塊感興趣，可以使用 `--only` 選項來過濾該區塊：

```shell
php artisan about --only=environment
```

或者，若要詳細查看特定設定檔的值，可以使用 `config:show` Artisan 指令：

```shell
php artisan config:show database
```

<a name="environment-configuration"></a>
## 環境設定

根據應用程式運行的環境提供不同的設定值通常很有幫助。例如，您可能希望在本地環境使用與正式伺服器不同的快取驅動。

為了讓這件事變得簡單，Laravel 使用了 [DotEnv](https://github.com/vlucas/phpdotenv) PHP 函式庫。在全新的 Laravel 安裝中，您的應用程式根目錄會包含一個 `.env.example` 檔案，其中定義了許多常見的環境變數。在 Laravel 安裝過程中，此檔案會自動被複製為 `.env`。

Laravel 預設的 `.env` 檔案包含了一些常見的設定值，這些值可能會根據您的應用程式是在本地運行還是正式網頁伺服器運行而有所不同。接著，`config` 目錄中的設定檔會使用 Laravel 的 `env` 函式讀取這些值。

如果您是與團隊協作開發，您可能希望在應用程式中繼續包含並更新 `.env.example` 檔案。透過在範例設定檔中放置佔位值，團隊中的其他開發者可以清楚地看到執行您的應用程式需要哪些環境變數。

> [!NOTE]
> `.env` 檔案中的任何變數都可以被外部環境變數（如伺服器級別或系統級別的環境變數）覆蓋。


<a name="environment-file-security"></a>
#### 環境檔案安全性

您的 `.env` 檔案不應該被提交到應用程式的版本控制系統中，因為每個使用您應用程式的開發者或伺服器可能需要不同的環境設定。此外，如果入侵者獲得了您的版本控制儲存庫存取權限，這將會是一個安全風險，因為任何敏感的憑證都會暴露。

然而，可以使用 Laravel 內建的[環境加密](#encrypting-environment-files)功能來加密您的環境檔案。加密後的環境檔案可以安全地放置在版本控制系統中。


<a name="additional-environment-files"></a>
#### 額外的環境檔案

在載入應用程式環境變數之前，Laravel 會判斷是否已從外部提供 `APP_ENV` 環境變數，或者是否指定了 `--env` CLI 參數。如果是，Laravel 會嘗試載入 `.env.[APP_ENV]` 檔案（如果存在）。如果不存在，則會載入預設的 `.env` 檔案。


<a name="environment-variable-types"></a>
### 環境變數類型

`.env` 檔案中的所有變數通常都被解析為字串，因此建立了一些保留值，讓您可以從 `env()` 函式回傳更廣泛的類型：

<div class="overflow-auto">

| `.env` 值 | `env()` 值 |
| ------------ | ------------- |
| true         | (bool) true   |
| (true)       | (bool) true   |
| false        | (bool) false  |
| (false)      | (bool) false  |
| empty        | (string) ''   |
| (empty)      | (string) ''   |
| null         | (null) null   |
| (null)       | (null) null   |

</div>

如果您需要定義一個包含空白字元的環境變數值，可以透過將該值包含在雙引號中來實現：

```ini
APP_NAME="My Application"
```


<a name="retrieving-environment-configuration"></a>
### 取得環境設定

當您的應用程式收到請求時，`.env` 檔案中列出的所有變數都會載入到 `$_ENV` PHP 超全域變數中。然而，您可以在設定檔中使用 `env` 函式從這些變數中取得值。事實上，如果您查看 Laravel 的設定檔，您會發現許多選項已經在使用此函式：

```php
'debug' => (bool) env('APP_DEBUG', false),
```

傳遞給 `env` 函式的第二個值是「預設值」。如果指定的鍵不存在環境變數，則會回傳此值。


<a name="determining-the-current-environment"></a>
### 判斷目前環境

目前的應用程式環境是透過 `.env` 檔案中的 `APP_ENV` 變數來決定的。您可以透過 `App` [facade](/docs/{{version}}/facades) 上的 `environment` 方法來存取此值：

```php
use Illuminate\Support\Facades\App;

$environment = App::environment();
```

您也可以將參數傳遞給 `environment` 方法，以判斷環境是否符合特定值。如果環境符合任何給定的值，該方法將回傳 `true`：

```php
if (App::environment('local')) {
    // The environment is local
}

if (App::environment(['local', 'staging'])) {
    // The environment is either local OR staging...
}
```

> [!NOTE]
> 目前應用程式環境的偵測可以透過定義伺服器級別的 `APP_ENV` 環境變數來覆寫。

<a name="encrypting-environment-files"></a>
### 加密環境檔案

未加密的環境檔案不應該存放在版本控制中。然而，Laravel 允許您加密環境檔案，以便讓它們能與應用程式的其餘部分一起安全地加入版本控制。

<a name="encryption"></a>
#### 加密

若要加密環境檔案，您可以使用 `env:encrypt` 指令：

```shell
php artisan env:encrypt
```

執行 `env:encrypt` 指令會加密您的 `.env` 檔案，並將加密後的內容放在 `.env.encrypted` 檔案中。解密金鑰會顯示在指令的輸出中，應將其儲存在安全的密碼管理員中。如果您想要提供自己的加密金鑰，可以在調用指令時使用 `--key` 選項：

```shell
php artisan env:encrypt --key=3UVsEgGVK36XN82KKeyLFMhvosbZN1aF
```

> [!NOTE]
> 提供的金鑰長度應與所使用的加密演算法所需的金鑰長度相符。預設情況下，Laravel 將使用 `AES-256-CBC` 演算法，這需要 32 個字元的金鑰。您可以透過在調用指令時傳遞 `--cipher` 選項，自由地使用任何 Laravel [encrypter](/docs/{{version}}/encryption) 支援的演算法。

如果您的應用程式有多個環境檔案，例如 `.env` 和 `.env.staging`，您可以透過 `--env` 選項提供環境名稱來指定應加密的環境檔案：

```shell
php artisan env:encrypt --env=staging
```

<a name="readable-variable-names"></a>
#### 可讀的變數名稱

在加密環境檔案時，您可以使用 `--readable` 選項，在加密變數值的同時保留可見的變數名稱：

```shell
php artisan env:encrypt --readable
```

這將產生一個具有以下格式的加密檔案：

```ini
APP_NAME=eyJpdiI6...
APP_ENV=eyJpdiI6...
APP_KEY=eyJpdiI6...
APP_DEBUG=eyJpdiI6...
APP_URL=eyJpdiI6...
```

使用可讀格式讓您可以在不洩露敏感資料的情況下查看存在哪些環境變數。這也使得審查 Pull Request 變得更加容易，因為您無需解密檔案即可查看哪些變數被新增、刪除或重新命名。

當解密環境檔案時，Laravel 會自動偵測使用了哪種格式，因此 `env:decrypt` 指令不需要額外的選項。

> [!NOTE]
> 當使用 `--readable` 選項時，原始環境檔案中的註解和空白行不會包含在加密後的輸出中。

<a name="decryption"></a>
#### 解密

若要解密環境檔案，您可以使用 `env:decrypt` 指令。此指令需要解密金鑰，Laravel 將從 `LARAVEL_ENV_ENCRYPTION_KEY` 環境變數中取得該金鑰：

```shell
php artisan env:decrypt
```

或者，可以透過 `--key` 選項直接向指令提供金鑰：

```shell
php artisan env:decrypt --key=3UVsEgGVK36XN82KKeyLFMhvosbZN1aF
```

當調用 `env:decrypt` 指令時，Laravel 會解密 `.env.encrypted` 檔案的內容，並將解密後的內容放在 `.env` 檔案中。

可以向 `env:decrypt` 指令提供 `--cipher` 選項，以便使用自訂的加密演算法：

```shell
php artisan env:decrypt --key=qUWuNRdfuImXcKxZ --cipher=AES-128-CBC
```

如果您的應用程式有多個環境檔案，例如 `.env` 和 `.env.staging`，您可以透過 `--env` 選項提供環境名稱來指定應解密的環境檔案：

```shell
php artisan env:decrypt --env=staging
```

為了覆寫現有的環境檔案，您可以向 `env:decrypt` 指令提供 `--force` 選項：

```shell
php artisan env:decrypt --force
```

<a name="accessing-configuration-values"></a>
## 讀取設定值

您可以在應用程式中的任何位置，透過 `Config` facade 或全域的 `config` 函式輕鬆讀取設定值。可以使用「點 (dot)」語法來讀取設定值，其中包含檔案名稱以及您希望讀取的選項名稱。您也可以指定一個預設值，若該設定選項不存在時則會回傳該預設值：

```php
use Illuminate\Support\Facades\Config;

$value = Config::get('app.timezone');

$value = config('app.timezone');

// Retrieve a default value if the configuration value does not exist...
$value = config('app.timezone', 'Asia/Seoul');
```

要在執行階段設定設定值，您可以呼叫 `Config` facade 的 `set` 方法，或是將一個陣列傳遞給 `config` 函式：

```php
Config::set('app.timezone', 'America/Chicago');

config(['app.timezone' => 'America/Chicago']);
```

為了輔助靜態分析，`Config` facade 還提供了有型別的設定讀取方法。如果取得的設定值與預期型別不符，將會拋出異常：

```php
Config::string('config-key');
Config::integer('config-key');
Config::float('config-key');
Config::boolean('config-key');
Config::array('config-key');
Config::collection('config-key');
```


<a name="configuration-caching"></a>
## 設定快取

為了提升應用程式的速度，您應該使用 `config:cache` Artisan 指令將所有的設定檔快取到單一檔案中。這會將應用程式的所有設定選項合併為一個檔案，讓框架可以快速載入。

通常您應該將 `php artisan config:cache` 指令作為正式環境部署程序的一部分執行。而在本地開發期間不應該執行此指令，因為在開發過程中會經常需要更改設定選項。

一旦設定被快取後，應用程式的 `.env` 檔案將不會在請求或 Artisan 指令期間被框架載入；因此，`env` 函式將只能回傳外部的系統級環境變數。

基於這個原因，您應該確保只在應用程式的設定 (`config`) 檔案中呼叫 `env` 函式。您可以透過查看 Laravel 的預設設定檔來看到許多這類的範例。在應用程式的任何地方，都可以使用[上述](#accessing-configuration-values)提到的 `config` 函式來讀取設定值。

可以使用 `config:clear` 指令來清除快取的設定：

```shell
php artisan config:clear
```

> [!WARNING]
> 如果您在部署程序中執行 `config:cache` 指令，請務必確保您只在設定檔中呼叫 `env` 函式。一旦設定被快取，`.env` 檔案將不會被載入；因此，`env` 函式將僅回傳外部的系統級環境變數。


<a name="configuration-publishing"></a>
## 發布設定

Laravel 的大多數設定檔都已經發布在應用程式的 `config` 目錄中；然而，某些設定檔如 `cors.php` 和 `view.php` 預設情況下並未發布，因為大多數應用程式永遠不需要修改它們。

不過，您可以使用 `config:publish` Artisan 指令來發布任何預設未發布的設定檔：

```shell
php artisan config:publish

php artisan config:publish --all
```


<a name="debug-mode"></a>
## 除錯模式

`config/app.php` 設定檔中的 `debug` 選項決定了實際向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循儲存在 `.env` 檔案中的 `APP_DEBUG` 環境變數之值。

> [!WARNING]
> 對於本地開發，您應該將 `APP_DEBUG` 環境變數設定為 `true`。**在正式環境中，此值應始終為 `false`。如果在正式環境中將變數設定為 `true`，您將面臨向應用程式終端使用者洩漏敏感設定值的風險。**

<a name="maintenance-mode"></a>
## 維護模式

當您的應用程式處於維護模式時，所有進入應用程式的請求都會顯示一個自訂視圖。這讓您在更新或進行維護時可以輕鬆地「停用」應用程式。您的應用程式預設的中介層堆疊中包含了一個維護模式檢查。如果應用程式處於維護模式，則會拋出一個狀態碼為 503 的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例。

若要啟用維護模式，請執行 `down` Artisan 指令：

```shell
php artisan down
```

如果您希望所有維護模式的回應都傳送 `Refresh` HTTP 標頭，您可以在呼叫 `down` 指令時提供 `refresh` 選項。`Refresh` 標頭會指示瀏覽器在指定的秒數後自動重新整理頁面：

```shell
php artisan down --refresh=15
```

您也可以為 `down` 指令提供 `retry` 選項，這將被設置為 `Retry-After` HTTP 標頭的值，儘管瀏覽器通常會忽略此標頭：

```shell
php artisan down --retry=60
```


<a name="bypassing-maintenance-mode"></a>
#### 跳過維護模式

若要允許使用秘密權杖跳過維護模式，您可以使用 `secret` 選項來指定維護模式跳過權杖：

```shell
php artisan down --secret="1630542a-246b-4b66-afa1-dd72a4c43515"
```

將應用程式設為維護模式後，您可以導航到與此權杖相匹配的應用程式 URL，Laravel 將向您的瀏覽器發送一個維護模式跳過 Cookie：

```shell
https://example.com/1630542a-246b-4b66-afa1-dd72a4c43515
```

如果您希望 Laravel 為您產生秘密權杖，可以使用 `with-secret` 選項。一旦應用程式進入維護模式，秘密權杖將會顯示給您：

```shell
php artisan down --with-secret
```

存取此隱藏路由後，您將被重新導向到應用程式的 `/` 路由。一旦 Cookie 發送至您的瀏覽器，您就可以像未處於維護模式一樣正常瀏覽應用程式。

> [!NOTE]
> 您的維護模式秘密權杖通常應由英數字元組成，並可選擇包含連字號。您應該避免使用在 URL 中具有特殊意義的字元，例如 `?` 或 `&`。


<a name="maintenance-mode-on-multiple-servers"></a>
#### 多台伺服器上的維護模式

預設情況下，Laravel 使用基於檔案的系統來判斷您的應用程式是否處於維護模式。這意味著要啟動維護模式，必須在託管應用程式的每台伺服器上執行 `php artisan down` 指令。

或者，Laravel 提供了一種基於快取的方法來處理維護模式。此方法只需要在其中一台伺服器上執行 `php artisan down` 指令。若要使用此方法，請修改應用程式 `.env` 檔案中的維護模式變數。您應該選擇一個所有伺服器都可以存取的快取 `store`。這可確保每台伺服器的維護模式狀態保持一致：

```ini
APP_MAINTENANCE_DRIVER=cache
APP_MAINTENANCE_STORE=database
```


<a name="pre-rendering-the-maintenance-mode-view"></a>
#### 預先渲染維護模式視圖

如果您在部署期間使用 `php artisan down` 指令，當您的 Composer 依賴項目或其他基礎架構組件正在更新時，使用者可能偶爾仍會遇到錯誤。這是因為 Laravel 框架的大部分必須啟動才能判斷您的應用程式處於維護模式，並使用範本引擎渲染維護模式視圖。

因此，Laravel 允許您預先渲染一個維護模式視圖，該視圖將在請求週期的最開始時回傳。此視圖在載入任何應用程式依賴項目之前就會渲染。您可以使用 `down` 指令的 `render` 選項預先渲染您選擇的範本：

```shell
php artisan down --render="errors::503"
```


<a name="redirecting-maintenance-mode-requests"></a>
#### 重新導向維護模式請求

在維護模式下，Laravel 會為使用者嘗試存取的所有應用程式 URL 顯示維護模式視圖。如果您願意，可以指示 Laravel 將所有請求重新導向到特定的 URL。這可以透過使用 `redirect` 選項來達成。例如，您可能希望將所有請求重新導向到 `/` URI：

```shell
php artisan down --redirect=/
```


<a name="disabling-maintenance-mode"></a>
#### 停用維護模式

若要停用維護模式，請使用 `up` 指令：

```shell
php artisan up
```

> [!NOTE]
> 您可以透過在 `resources/views/errors/503.blade.php` 定義自己的範本來客製化預設的維護模式範本。


<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於維護模式時，將不會處理任何[佇列任務](/docs/{{version}}/queues)。一旦應用程式結束維護模式，任務將照常繼續處理。


<a name="alternatives-to-maintenance-mode"></a>
#### 維護模式的替代方案

由於維護模式需要您的應用程式停機數秒，請考慮在像 [Laravel Cloud](https://cloud.laravel.com) 這樣的全託管平台上執行您的應用程式，以實現 Laravel 的零停機部署。