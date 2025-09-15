# 設定

- [前言](#introduction)
- [環境設定](#environment-configuration)
    - [環境變數類型](#environment-variable-types)
    - [取得環境設定](#retrieving-environment-configuration)
    - [判斷目前環境](#determining-the-current-environment)
    - [加密環境檔案](#encrypting-environment-files)
- [存取設定值](#accessing-configuration-values)
- [設定快取](#configuration-caching)
- [發佈設定](#configuration-publishing)
- [除錯模式](#debug-mode)
- [維護模式](#maintenance-mode)

<a name="introduction"></a>
## 前言

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細說明，您可以隨意瀏覽這些檔案，熟悉可用的選項。

這些設定檔讓您可以設定資料庫連線資訊、郵件伺服器資訊，以及其他各種核心設定值，例如應用程式的 URL 和加密金鑰。

<a name="the-about-command"></a>
#### `about` 命令

Laravel 可以透過 `about` Artisan 命令顯示應用程式的設定、驅動程式和環境的概覽。

```shell
php artisan about
```

如果您只對應用程式概覽輸出的特定部分感興趣，可以使用 `--only` 選項進行篩選：

```shell
php artisan about --only=environment
```

或者，若要詳細探索特定設定檔的值，可以使用 `config:show` Artisan 命令：

```shell
php artisan config:show database
```

<a name="environment-configuration"></a>
## 環境設定

根據應用程式執行的環境，擁有不同的設定值通常很有幫助。例如，您可能希望在本地端使用與正式伺服器上不同的快取驅動程式。

為了讓這變得輕而易舉，Laravel 利用了 [DotEnv](https://github.com/vlucas/phpdotenv) PHP 函式庫。在全新的 Laravel 安裝中，應用程式的根目錄會包含一個 `.env.example` 檔案，其中定義了許多常見的環境變數。在 Laravel 安裝過程中，此檔案會自動複製到 `.env`。

Laravel 的預設 `.env` 檔案包含一些常見的設定值，這些值可能會因應用程式是在本地端執行還是部署在正式 Web 伺服器上而有所不同。然後，這些值會透過 Laravel 的 `env` 函式由 `config` 目錄中的設定檔讀取。

如果您是團隊開發，您可能希望繼續在應用程式中包含並更新 `.env.example` 檔案。透過在範例設定檔中放置預留位置值，團隊中的其他開發者可以清楚地看到執行應用程式所需的環境變數。

> [!NOTE]
> 您 `.env` 檔案中的任何變數都可以被外部環境變數覆寫，例如伺服器層級或系統層級的環境變數。

<a name="environment-file-security"></a>
#### 環境檔案安全性

您的 `.env` 檔案不應提交到應用程式的原始碼控制中，因為每個使用您應用程式的開發者/伺服器可能需要不同的環境設定。此外，如果入侵者取得您原始碼控制儲存庫的存取權，這將會是一個安全風險，因為任何敏感憑證都將會暴露。

然而，可以使用 Laravel 內建的[環境加密](#encrypting-environment-files)來加密您的環境檔案。加密後的環境檔案可以安全地放置在原始碼控制中。

<a name="additional-environment-files"></a>
#### 其他環境檔案

在載入應用程式的環境變數之前，Laravel 會判斷是否已從外部提供了 `APP_ENV` 環境變數，或者是否已指定了 `--env` CLI 引數。如果是，Laravel 將嘗試載入 `.env.[APP_ENV]` 檔案（如果存在）。如果不存在，則會載入預設的 `.env` 檔案。

<a name="environment-variable-types"></a>
### 環境變數類型

您 `.env` 檔案中的所有變數通常都會被解析為字串，因此已建立一些保留值，讓您可以從 `env()` 函式傳回更廣泛的類型：

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

如果您需要定義一個包含空格值的環境變數，可以將該值用雙引號括起來：

```ini
APP_NAME="My Application"
```

<a name="retrieving-environment-configuration"></a>
### 取得環境設定

當您的應用程式收到請求時，`.env` 檔案中列出的所有變數都將載入到 `$_ENV` PHP 超級全域變數中。但是，您可以在設定檔中使用 `env` 函式來取得這些變數的值。事實上，如果您檢閱 Laravel 設定檔，您會注意到許多選項已經在使用此函式：

    'debug' => env('APP_DEBUG', false),

傳遞給 `env` 函式的第二個值是「預設值」。如果給定鍵沒有環境變數存在，則會傳回此值。

<a name="determining-the-current-environment"></a>
### 判斷目前環境

目前的應用程式環境是透過您 `.env` 檔案中的 `APP_ENV` 變數來判斷的。您可以透過 `App` [Facade](/docs/{{version}}/facades) 上的 `environment` 方法來存取此值：

    use Illuminate\Support\Facades\App;

    $environment = App::environment();

您也可以將引數傳遞給 `environment` 方法，以判斷環境是否符合給定值。如果環境符合任何給定值，該方法將傳回 `true`：

    if (App::environment('local')) {
        // The environment is local
    }

    if (App::environment(['local', 'staging'])) {
        // The environment is either local OR staging...
    }

> [!NOTE]
> 目前應用程式環境的偵測可以透過定義伺服器層級的 `APP_ENV` 環境變數來覆寫。

<a name="encrypting-environment-files"></a>
### 加密環境檔案

未加密的環境檔案絕不應儲存在原始碼控制中。然而，Laravel 允許您加密環境檔案，以便它們可以安全地與應用程式的其餘部分一起新增到原始碼控制中。

<a name="encryption"></a>
#### 加密

若要加密環境檔案，您可以使用 `env:encrypt` 命令：

```shell
php artisan env:encrypt
```

執行 `env:encrypt` 命令將會加密您的 `.env` 檔案，並將加密後的內容放置在 `.env.encrypted` 檔案中。解密金鑰會顯示在命令的輸出中，應儲存在安全的密碼管理器中。如果您想提供自己的加密金鑰，可以在呼叫命令時使用 `--key` 選項：

```shell
php artisan env:encrypt --key=3UVsEgGVK36XN82KKeyLFMhvosbZN1aF
```

> [!NOTE]
> 所提供的金鑰長度應與所使用的加密演算法所需的金鑰長度相符。預設情況下，Laravel 將使用 `AES-256-CBC` 演算法，該演算法需要 32 個字元的金鑰。您可以透過在呼叫命令時傳遞 `--cipher` 選項，自由使用 Laravel [加密器](/docs/{{version}}/encryption)支援的任何演算法。

如果您的應用程式有多個環境檔案，例如 `.env` 和 `.env.staging`，您可以透過 `--env` 選項提供環境名稱來指定應加密的環境檔案：

```shell
php artisan env:encrypt --env=staging
```

<a name="decryption"></a>
#### 解密

若要解密環境檔案，您可以使用 `env:decrypt` 命令。此命令需要一個解密金鑰，Laravel 將從 `LARAVEL_ENV_ENCRYPTION_KEY` 環境變數中取得該金鑰：

```shell
php artisan env:decrypt
```

或者，金鑰可以直接透過 `--key` 選項提供給命令：

```shell
php artisan env:decrypt --key=3UVsEgGVK36XN82KKeyLFMhvosbZN1aF
```

當呼叫 `env:decrypt` 命令時，Laravel 將解密 `.env.encrypted` 檔案的內容，並將解密後的內容放置在 `.env` 檔案中。

可以向 `env:decrypt` 命令提供 `--cipher` 選項，以使用自訂加密演算法：

```shell
php artisan env:decrypt --key=qUWuNRdfuImXcKxZ --cipher=AES-128-CBC
```

如果您的應用程式有多個環境檔案，例如 `.env` 和 `.env.staging`，您可以透過 `--env` 選項提供環境名稱來指定應解密的環境檔案：

```shell
php artisan env:decrypt --env=staging
```

為了覆寫現有的環境檔案，您可以向 `env:decrypt` 命令提供 `--force` 選項：

```shell
php artisan env:decrypt --force
```

<a name="accessing-configuration-values"></a>
## 存取設定值

您可以使用 `Config` Facade 或全域 `config` 函式，從應用程式的任何地方輕鬆存取您的設定值。設定值可以使用「點」語法存取，其中包含您要存取的檔案名稱和選項。也可以指定預設值，如果設定選項不存在，則會傳回該預設值：

    use Illuminate\Support\Facades\Config;

    $value = Config::get('app.timezone');

    $value = config('app.timezone');

    // 如果設定值不存在，則取得預設值...
    $value = config('app.timezone', 'Asia/Seoul');

若要在執行時設定設定值，您可以呼叫 `Config` Facade 的 `set` 方法，或將陣列傳遞給 `config` 函式：

    Config::set('app.timezone', 'America/Chicago');

    config(['app.timezone' => 'America/Chicago']);

為了協助靜態分析，`Config` Facade 也提供了型別化的設定取得方法。如果取得的設定值與預期型別不符，將會拋出例外：

    Config::string('config-key');
    Config::integer('config-key');
    Config::float('config-key');
    Config::boolean('config-key');
    Config::array('config-key');

<a name="configuration-caching"></a>
## 設定快取

為了提高應用程式的速度，您應該使用 `config:cache` Artisan 命令將所有設定檔快取到單一檔案中。這會將應用程式的所有設定選項合併到一個檔案中，框架可以快速載入該檔案。

您通常應該在正式部署過程中執行 `php artisan config:cache` 命令。在本地開發期間不應執行此命令，因為在應用程式開發過程中，設定選項會經常需要更改。

一旦設定被快取，您的應用程式的 `.env` 檔案將不會在請求或 Artisan 命令期間被框架載入；因此，`env` 函式將只傳回外部、系統層級的環境變數。

因此，您應該確保只從應用程式的設定 (`config`) 檔案中呼叫 `env` 函式。您可以透過檢查 Laravel 的預設設定檔來查看許多範例。設定值可以使用[上述](#accessing-configuration-values)的 `config` 函式從應用程式的任何地方存取。

`config:clear` 命令可用於清除快取的設定：

```shell
php artisan config:clear
```

> [!WARNING]
> 如果您在部署過程中執行 `config:cache` 命令，您應該確保只從設定檔中呼叫 `env` 函式。一旦設定被快取，`.env` 檔案將不會被載入；因此，`env` 函式將只傳回外部、系統層級的環境變數。

<a name="configuration-publishing"></a>
## 發佈設定

Laravel 的大多數設定檔都已發佈在您應用程式的 `config` 目錄中；然而，某些設定檔，例如 `cors.php` 和 `view.php`，預設不會發佈，因為大多數應用程式永遠不需要修改它們。

但是，您可以使用 `config:publish` Artisan 命令來發佈任何預設未發佈的設定檔：

```shell
php artisan config:publish

php artisan config:publish --all
```

<a name="debug-mode"></a>
## 除錯模式

您 `config/app.php` 設定檔中的 `debug` 選項決定了實際向使用者顯示多少錯誤資訊。預設情況下，此選項設定為遵循 `APP_DEBUG` 環境變數的值，該變數儲存在您的 `.env` 檔案中。

> [!WARNING]
> 對於本地開發，您應該將 `APP_DEBUG` 環境變數設定為 `true`。**在您的正式環境中，此值應始終為 `false`。如果該變數在正式環境中設定為 `true`，您將面臨向應用程式終端使用者暴露敏感設定值的風險。**

<a name="maintenance-mode"></a>
## 維護模式

當您的應用程式處於維護模式時，所有對應用程式的請求都將顯示一個自訂視圖。這使得在應用程式更新或執行維護時，可以輕鬆地「停用」應用程式。維護模式檢查包含在應用程式的預設 Middleware 堆疊中。如果應用程式處於維護模式，將會拋出一個狀態碼為 503 的 `Symfony\Component\HttpKernel\Exception\HttpException` 實例。

若要啟用維護模式，請執行 `down` Artisan 命令：

```shell
php artisan down
```

如果您希望所有維護模式回應都傳送 `Refresh` HTTP 標頭，您可以在呼叫 `down` 命令時提供 `refresh` 選項。`Refresh` 標頭將指示瀏覽器在指定秒數後自動重新整理頁面：

```shell
php artisan down --refresh=15
```

您也可以向 `down` 命令提供 `retry` 選項，該選項將設定為 `Retry-After` HTTP 標頭的值，儘管瀏覽器通常會忽略此標頭：

```shell
php artisan down --retry=60
```

<a name="bypassing-maintenance-mode"></a>
#### 繞過維護模式

若要允許使用密碼權杖繞過維護模式，您可以使用 `secret` 選項來指定維護模式繞過權杖：

```shell
php artisan down --secret="1630542a-246b-4b66-afa1-dd72a4c43515"
```

將應用程式置於維護模式後，您可以導航到與此權杖匹配的應用程式 URL，Laravel 將向您的瀏覽器發出維護模式繞過 Cookie：

```shell
https://example.com/1630542a-246b-4b66-afa1-dd72a4c43515
```

如果您希望 Laravel 為您產生密碼權杖，您可以使用 `with-secret` 選項。一旦應用程式進入維護模式，密碼將會顯示給您：

```shell
php artisan down --with-secret
```

當存取此隱藏路由時，您將被重新導向到應用程式的 `/` 路由。一旦 Cookie 已發行到您的瀏覽器，您將能夠像應用程式未處於維護模式一樣正常瀏覽應用程式。

> [!NOTE]
> 您的維護模式密碼通常應由英數字元組成，並可選地包含破折號。您應避免使用在 URL 中具有特殊含義的字元，例如 `?` 或 `&`。

<a name="maintenance-mode-on-multiple-servers"></a>
#### 多伺服器上的維護模式

預設情況下，Laravel 使用基於檔案的系統來判斷您的應用程式是否處於維護模式。這表示要啟用維護模式，必須在託管您應用程式的每個伺服器上執行 `php artisan down` 命令。

或者，Laravel 提供了一種基於快取的方法來處理維護模式。此方法只需要在一台伺服器上執行 `php artisan down` 命令。若要使用此方法，請修改應用程式 `.env` 檔案中的維護模式變數。您應該選擇一個所有伺服器都可以存取的快取 `store`。這確保了維護模式狀態在每個伺服器上都保持一致：

```ini
APP_MAINTENANCE_DRIVER=cache
APP_MAINTENANCE_STORE=database
```

<a name="pre-rendering-the-maintenance-mode-view"></a>
#### 預先渲染維護模式視圖

如果您在部署期間使用 `php artisan down` 命令，如果使用者在 Composer 依賴項或其他基礎設施元件更新時存取應用程式，他們可能仍會偶爾遇到錯誤。這是因為 Laravel 框架的很大一部分必須啟動才能判斷您的應用程式是否處於維護模式，並使用模板引擎渲染維護模式視圖。

因此，Laravel 允許您預先渲染一個維護模式視圖，該視圖將在請求週期的最開始返回。此視圖在載入任何應用程式依賴項之前渲染。您可以使用 `down` 命令的 `render` 選項預先渲染您選擇的模板：

```shell
php artisan down --render="errors::503"
```

<a name="redirecting-maintenance-mode-requests"></a>
#### 重新導向維護模式請求

在維護模式下，Laravel 將為使用者嘗試存取的所有應用程式 URL 顯示維護模式視圖。如果您願意，您可以指示 Laravel 將所有請求重新導向到特定的 URL。這可以透過使用 `redirect` 選項來完成。例如，您可能希望將所有請求重新導向到 `/` URI：

```shell
php artisan down --redirect=/
```

<a name="disabling-maintenance-mode"></a>
#### 停用維護模式

若要停用維護模式，請使用 `up` 命令：

```shell
php artisan up
```

> [!NOTE]
> 您可以透過在 `resources/views/errors/503.blade.php` 定義自己的模板來自訂預設的維護模式模板。

<a name="maintenance-mode-queues"></a>
#### 維護模式與佇列

當您的應用程式處於維護模式時，不會處理任何 [佇列任務](/docs/{{version}}/queues)。一旦應用程式退出維護模式，任務將照常處理。

<a name="alternatives-to-maintenance-mode"></a>
#### 維護模式的替代方案

由於維護模式需要您的應用程式有幾秒鐘的停機時間，請考慮使用 [Laravel Vapor](https://vapor.laravel.com) 和 [Envoyer](https://envoyer.io) 等替代方案，以實現 Laravel 的零停機部署。
