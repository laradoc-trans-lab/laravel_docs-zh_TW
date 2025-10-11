# Laravel Valet

- [簡介](#introduction)
- [安裝](#installation)
    - [升級 Valet](#upgrading-valet)
- [服務站台](#serving-sites)
    - [`park` 指令](#the-park-command)
    - [`link` 指令](#the-link-command)
    - [使用 TLS 保護站台](#securing-sites)
    - [服務預設站台](#serving-a-default-site)
    - [各站台專屬的 PHP 版本](#per-site-php-versions)
- [分享站台](#sharing-sites)
    - [在您的本機網路分享站台](#sharing-sites-on-your-local-network)
- [站台專屬環境變數](#site-specific-environment-variables)
- [代理服務](#proxying-services)
- [自訂 Valet Driver](#custom-valet-drivers)
    - [本機 Driver](#local-drivers)
- [其他 Valet 指令](#other-valet-commands)
- [Valet 目錄與檔案](#valet-directories-and-files)
    - [磁碟存取](#disk-access)

<a name="introduction"></a>
## 簡介

> [!NOTE]  
> 正在尋找一種在 macOS 或 Windows 上開發 Laravel 應用程式的更簡單方式嗎？請參考 [Laravel Herd](https://herd.laravel.com)。Herd 包含了開始 Laravel 開發所需的一切，包括 Valet、PHP 和 Composer。

[Laravel Valet](https://github.com/laravel/valet) 是一個為 macOS 極簡主義者設計的開發環境。Laravel Valet 會將您的 Mac 配置為在機器啟動時始終在背景執行 [Nginx](https://www.nginx.com/)。接著，Valet 會使用 [DnsMasq](https://en.wikipedia.org/wiki/Dnsmasq)，將所有對 `*.test` 網域的請求代理到安裝在您本機上的站台。

換句話說，Valet 是一個速度極快的 Laravel 開發環境，僅使用大約 7 MB 的 RAM。Valet 並非 [Sail](/docs/{{version}}/sail) 或 [Homestead](/docs/{{version}}/homestead) 的完整替代品，但如果您想要彈性的基礎、偏好極致的速度，或者是在 RAM 有限的機器上工作，它提供了一個很好的替代方案。

開箱即用，Valet 支援包括但不限於：

<style>
    #valet-support > ul {
        column-count: 3; -moz-column-count: 3; -webkit-column-count: 3;
        line-height: 1.9;
    }
</style>

<div id="valet-support" markdown="1">

- [Laravel](https://laravel.com)
- [Bedrock](https://roots.io/bedrock/)
- [CakePHP 3](https://cakephp.org)
- [ConcreteCMS](https://www.concretecms.com/)
- [Contao](https://contao.org/en/)
- [Craft](https://craftcms.com)
- [Drupal](https://www.drupal.org/)
- [ExpressionEngine](https://www.expressionengine.com/)
- [Jigsaw](https://jigsaw.tighten.co)
- [Joomla](https://www.joomla.org/)
- [Katana](https://github.com/themsaid/katana)
- [Kirby](https://getkirby.com/)
- [Magento](https://magento.com/)
- [OctoberCMS](https://octobercms.com/)
- [Sculpin](https://sculpin.io/)
- [Slim](https://www.slimframework.com)
- [Statamic](https://statamic.com)
- 靜態 HTML
- [Symfony](https://symfony.com)
- [WordPress](https://wordpress.org)
- [Zend](https://framework.zend.com)

</div>

不過，您可以使用自己的[自訂 Valet Driver](#custom-valet-drivers) 來擴展 Valet。

<a name="installation"></a>
## 安裝

> [!WARNING]  
> Valet 需要 macOS 和 [Homebrew](https://brew.sh/)。在安裝之前，您應該確保沒有其他程式，例如 Apache 或 Nginx，綁定到您本機的 80 埠。

要開始使用，您首先需要使用 `update` 指令確保 Homebrew 是最新的：

```shell
brew update
```

接著，您應該使用 Homebrew 安裝 PHP：

```shell
brew install php
```

安裝 PHP 後，您就可以安裝 [Composer 套件管理工具](https://getcomposer.org)了。此外，您應該確保 `$HOME/.composer/vendor/bin` 目錄位於您系統的「PATH」中。Composer 安裝完成後，您可以將 Laravel Valet 安裝為一個全域 Composer 套件：

```shell
composer global require laravel/valet
```

最後，您可以執行 Valet 的 `install` 指令。這將配置並安裝 Valet 和 DnsMasq。此外，Valet 所依賴的守護程式將被配置為在您的系統啟動時啟動：

```shell
valet install
```

Valet 安裝完成後，嘗試在您的終端機上使用 `ping foobar.test` 之類的指令來 ping 任何 `*.test` 網域。如果 Valet 正確安裝，您應該會看到此網域在 `127.0.0.1` 上回應。

Valet 會在您的機器每次啟動時自動啟動其所需的服務。

<a name="php-versions"></a>
#### PHP 版本

> [!NOTE]  
> 您可以指示 Valet 使用各站台專屬的 PHP 版本，而不是修改您的全域 PHP 版本，透過 `isolate` [指令](#per-site-php-versions)。

Valet 允許您使用 `valet use php@version` 指令切換 PHP 版本。如果指定的 PHP 版本尚未安裝，Valet 會透過 Homebrew 安裝它：

```shell
valet use php@8.2

valet use php
```

您也可以在專案的根目錄中建立一個 `.valetrc` 檔案。`.valetrc` 檔案應該包含站台應使用的 PHP 版本：

```shell
php=php@8.2
```

建立此檔案後，您只需執行 `valet use` 指令，該指令將透過讀取檔案來判斷站台偏好的 PHP 版本。

> [!WARNING]  
> Valet 一次只提供一個 PHP 版本，即使您安裝了多個 PHP 版本。

<a name="database"></a>
#### 資料庫

如果您的應用程式需要資料庫，請查看 [DBngin](https://dbngin.com)，它提供了一個免費、一體化的資料庫管理工具，包含 MySQL、PostgreSQL 和 Redis。DBngin 安裝完成後，您可以使用 `root` 使用者名稱和空字串密碼連接到 `127.0.0.1` 的資料庫。

<a name="resetting-your-installation"></a>
#### 重設您的安裝

如果您在讓 Valet 安裝正常執行時遇到問題，執行 `composer global require laravel/valet` 指令，接著執行 `valet install`，將會重設您的安裝並解決各種問題。在極少數情況下，可能需要「硬重設」Valet，透過執行 `valet uninstall --force` 接著執行 `valet install`。

<a name="upgrading-valet"></a>
### 升級 Valet

您可以透過在終端機中執行 `composer global require laravel/valet` 指令來更新您的 Valet 安裝。升級後，建議執行 `valet install` 指令，以便 Valet 在必要時對您的配置檔進行額外升級。

<a name="upgrading-to-valet-4"></a>
#### 升級至 Valet 4

如果您正在從 Valet 3 升級到 Valet 4，請採取以下步驟正確升級您的 Valet 安裝：

<div class="content-list" markdown="1">

- 如果您曾新增 `.valetphprc` 檔案來客製化站台的 PHP 版本，請將每個 `.valetphprc` 檔案重新命名為 `.valetrc`。接著，在 `.valetrc` 檔案的現有內容前加上 `php=`。
- 更新任何自訂 driver 以符合新 driver 系統的命名空間、擴展、型別提示和回傳型別提示。您可以參考 Valet 的 [SampleValetDriver](https://github.com/laravel/valet/blob/d7787c025e60abc24a5195dc7d4c5c6f2d984339/cli/stubs/SampleValetDriver.php) 作為範例。
- 如果您使用 PHP 7.1 - 7.4 來提供您的站台服務，請確保您仍使用 Homebrew 安裝 PHP 8.0 或更高版本，因為 Valet 將會使用此版本來執行其部分指令稿，即使它不是您主要連結的版本。

</div>

<a name="serving-sites"></a>
## 服務站台

一旦 Valet 安裝完成，您就可以開始服務您的 Laravel 應用程式。Valet 提供兩個指令來幫助您服務應用程式：`park` 和 `link`。

<a name="the-park-command"></a>
### `park` 指令

`park` 指令會在您的機器上註冊一個包含您應用程式的目錄。一旦該目錄被 Valet park 後，該目錄內的所有子目錄都將能透過網頁瀏覽器以 `http://<directory-name>.test` 網域存取：

```shell
cd ~/Sites

valet park
```

就這麼簡單。現在，您在 park 目錄中建立的任何應用程式都將自動以 `http://<directory-name>.test` 的慣例被服務。因此，如果您的停泊目錄包含一個名為 "laravel" 的目錄，該目錄中的應用程式將可透過 `http://laravel.test` 存取。此外，Valet 自動允許您使用萬用字元子網域 (`http://foo.laravel.test`) 存取站台。

<a name="the-link-command"></a>
### `link` 指令

`link` 指令也可以用來服務您的 Laravel 應用程式。如果您只想服務目錄中的單一站台而不是整個目錄，此指令會很有用：

```shell
cd ~/Sites/laravel

valet link
```

一旦應用程式使用 `link` 指令連結到 Valet，您就可以使用其目錄名稱來存取應用程式。因此，上述範例中連結的站台可以透過 `http://laravel.test` 存取。此外，Valet 自動允許您使用萬用字元子網域 (`http://foo.laravel.test`) 存取站台。

如果您想在不同的主機名稱下服務應用程式，您可以將主機名稱傳遞給 `link` 指令。例如，您可以執行以下指令來使應用程式在 `http://application.test` 可用：

```shell
cd ~/Sites/laravel

valet link application
```

當然，您也可以使用 `link` 指令在子網域上服務應用程式：

```shell
valet link api.application
```

您可以執行 `links` 指令以顯示所有已連結目錄的列表：

```shell
valet links
```

`unlink` 指令可以用來銷毀站台的符號連結：

```shell
cd ~/Sites/laravel

valet unlink
```

<a name="securing-sites"></a>
### 使用 TLS 保護站台

預設情況下，Valet 透過 HTTP 服務站台。但是，如果您希望透過使用 HTTP/2 的加密 TLS 服務站台，您可以使用 `secure` 指令。例如，如果您的站台由 Valet 在 `laravel.test` 網域上服務，您應該執行以下指令來保護它：

```shell
valet secure laravel
```

要「取消保護」站台並恢復透過純 HTTP 服務其流量，請使用 `unsecure` 指令。與 `secure` 指令一樣，此指令接受您希望取消保護的主機名稱：

```shell
valet unsecure laravel
```

<a name="serving-a-default-site"></a>
### 服務預設站台

有時候，您可能希望將 Valet 設定為在造訪未知 `test` 網域時，服務一個「預設」站台而不是 `404` 錯誤頁面。為了實現這一點，您可以在 `~/.config/valet/config.json` 設定檔中添加一個 `default` 選項，其中包含應作為預設站台的網站路徑：

    "default": "/Users/Sally/Sites/example-site",

<a name="per-site-php-versions"></a>
### 各站台專屬的 PHP 版本

預設情況下，Valet 使用您全域的 PHP 安裝來服務您的站台。但是，如果您需要在不同站台之間支援多個 PHP 版本，您可以使用 `isolate` 指令來指定特定站台應使用的 PHP 版本。`isolate` 指令會設定 Valet，使其在您目前工作目錄中的站台使用指定的 PHP 版本：

```shell
cd ~/Sites/example-site

valet isolate php@8.0
```

如果您的站台名稱與包含它的目錄名稱不符，您可以使用 `--site` 選項指定站台名稱：

```shell
valet isolate php@8.0 --site="site-name"
```

為了方便起見，您可以使用 `valet php`、`composer` 和 `which-php` 指令來根據站台配置的 PHP 版本，代理對適當的 PHP CLI 或工具的呼叫：

```shell
valet php
valet composer
valet which-php
```

您可以執行 `isolated` 指令以顯示所有獨立站台及其 PHP 版本的列表：

```shell
valet isolated
```

要將站台恢復為 Valet 全域安裝的 PHP 版本，您可以在站台的根目錄中呼叫 `unisolate` 指令：

```shell
valet unisolate
```

<a name="sharing-sites"></a>
## 分享站台

Valet 包含一個指令，可以將您的本地站台分享給全世界，提供一種簡單的方式來在行動裝置上測試您的站台，或與團隊成員和客戶分享。

Valet 預設支援透過 ngrok 或 Expose 分享您的站台。在分享站台之前，您應該使用 `share-tool` 指令更新您的 Valet 設定，指定 `ngrok` 或 `expose`：

```shell
valet share-tool ngrok
```

如果您選擇一個工具，但沒有透過 Homebrew (適用於 ngrok) 或 Composer (適用於 Expose) 安裝它，Valet 會自動提示您安裝。當然，這兩個工具都要求您在開始分享站台之前驗證您的 ngrok 或 Expose 帳戶。

要分享站台，請在終端機中導航到站台的目錄，然後執行 Valet 的 `share` 指令。一個公開可存取的 URL 將被複製到您的剪貼簿中，並準備好直接貼到您的瀏覽器中或與您的團隊分享：

```shell
cd ~/Sites/laravel

valet share
```

要停止分享您的站台，您可以按下 `Control + C`。

> [!WARNING]  
> 如果您使用自訂 DNS 伺服器 (例如 `1.1.1.1`)，ngrok 分享可能無法正常運作。如果您的機器上發生這種情況，請打開您的 Mac 系統設定，進入「網路」設定，打開「進階設定」，然後進入「DNS」分頁並將 `127.0.0.1` 添加為您的第一個 DNS 伺服器。

<a name="sharing-sites-via-ngrok"></a>
#### 透過 Ngrok 分享站台

使用 ngrok 分享您的站台需要您[建立一個 ngrok 帳戶](https://dashboard.ngrok.com/signup)並[設定一個認證 token](https://dashboard.ngrok.com/get-started/your-authtoken)。一旦您擁有認證 token，您可以使用該 token 更新您的 Valet 設定：

```shell
valet set-ngrok-token YOUR_TOKEN_HERE
```

> [!NOTE]  
> 您可以將額外的 ngrok 參數傳遞給 share 指令，例如 `valet share --region=eu`。有關更多資訊，請查閱 [ngrok 文件](https://ngrok.com/docs)。

<a name="sharing-sites-via-expose"></a>
#### 透過 Expose 分享站台

使用 Expose 分享您的站台需要您[建立一個 Expose 帳戶](https://expose.dev/register)並[透過您的認證 token 進行 Expose 驗證](https://expose.dev/docs/getting-started/getting-your-token)。

您可以查閱 [Expose 文件](https://expose.dev/docs)以獲取有關其支援的額外命令列參數的資訊。

<a name="sharing-sites-on-your-local-network"></a>
### 在您的本機網路分享站台

Valet 預設將傳入流量限制在內部 `127.0.0.1` 介面，這樣您的開發機器就不會暴露在來自網際網路的安全風險中。

如果您希望允許本機網路上的其他裝置透過您機器的 IP 位址 (例如：`192.168.1.10/application.test`) 存取您機器上的 Valet 站台，您將需要手動編輯該站台的相應 Nginx 設定檔，以移除 `listen` 指令上的限制。您應該移除連接埠 80 和 443 的 `listen` 指令上的 `127.0.0.1:` 前綴。

如果您尚未在專案上執行 `valet secure`，您可以透過編輯 `/usr/local/etc/nginx/valet/valet.conf` 檔案來開啟所有非 HTTPS 站台的網路存取。但是，如果您透過 HTTPS 服務專案站台 (您已為該站台執行了 `valet secure`)，那麼您應該編輯 `~/.config/valet/Nginx/app-name.test` 檔案。

更新 Nginx 設定後，執行 `valet restart` 指令以應用設定變更。

<a name="site-specific-environment-variables"></a>
## 站台專屬環境變數

有些使用其他框架的應用程式可能依賴伺服器環境變數，但沒有提供在專案中配置這些變數的方式。Valet 允許您透過在專案根目錄中新增一個 `.valet-env.php` 檔案來配置站台專屬的環境變數。這個檔案應該回傳一個站台/環境變數配對的陣列，這些變數將被新增到陣列中每個指定站台的全局 `$_SERVER` 陣列中：

    <?php

    return [
        // Set $_SERVER['key'] to "value" for the laravel.test site...
        'laravel' => [
            'key' => 'value',
        ],

        // Set $_SERVER['key'] to "value" for all sites...
        '*' => [
            'key' => 'value',
        ],
    ];


<a name="proxying-services"></a>
## 代理服務

有時您可能希望將 Valet 網域代理到本機上的另一個服務。例如，您可能偶爾需要在執行 Valet 的同時，也在 Docker 中執行另一個獨立的站台；但是，Valet 和 Docker 無法同時綁定到 Port 80。

為了解決這個問題，您可以使用 `proxy` 指令來產生一個代理。例如，您可以將所有從 `http://elasticsearch.test` 到 `http://127.0.0.1:9200` 的流量進行代理：

```shell
# Proxy over HTTP...
valet proxy elasticsearch http://127.0.0.1:9200

# Proxy over TLS + HTTP/2...
valet proxy elasticsearch http://127.0.0.1:9200 --secure
```

您可以使用 `unproxy` 指令來移除代理：

```shell
valet unproxy elasticsearch
```

您可以使用 `proxies` 指令來列出所有已代理的站台配置：

```shell
valet proxies
```


<a name="custom-valet-drivers"></a>
## 自訂 Valet Driver

您可以編寫自己的 Valet "driver" 來服務未被 Valet 原生支援的框架或 CMS 上執行的 PHP 應用程式。當您安裝 Valet 時，會建立一個 `~/.config/valet/Drivers` 目錄，其中包含一個 `SampleValetDriver.php` 檔案。這個檔案包含一個範例 driver 實作，以展示如何編寫自訂 driver。編寫 driver 只需實作三個方法：`serves`、`isStaticFile` 和 `frontControllerPath`。

所有這三個方法都會接收 `$sitePath`、`$siteName` 和 `$uri` 值作為參數。`$sitePath` 是機器上被服務站台的完整路徑，例如 `/Users/Lisa/Sites/my-project`。`$siteName` 是網域的「host」/「site name」部分 (`my-project`)。`$uri` 是傳入的請求 URI (`/foo/bar`)。

完成自訂 Valet driver 後，請使用 `FrameworkValetDriver.php` 命名慣例將其放置在 `~/.config/valet/Drivers` 目錄中。例如，如果您正在為 WordPress 編寫自訂 valet driver，您的檔案名稱應該是 `WordPressValetDriver.php`。

讓我們看看您的自訂 Valet driver 應該實作的每個方法的範例實作。


<a name="the-serves-method"></a>
#### `serves` 方法

如果您的 driver 應該處理傳入的請求，`serves` 方法應回傳 `true`。否則，該方法應回傳 `false`。因此，在此方法中，您應該嘗試判斷給定的 `$sitePath` 是否包含您正在嘗試服務的專案類型。

例如，假設我們正在編寫一個 `WordPressValetDriver`。我們的 `serves` 方法可能看起來像這樣：

    /**
     * Determine if the driver serves the request.
     */
    public function serves(string $sitePath, string $siteName, string $uri): bool
    {
        return is_dir($sitePath.'/wp-admin');
    }


<a name="the-isstaticfile-method"></a>
#### `isStaticFile` 方法

`isStaticFile` 應該判斷傳入的請求是否針對一個「靜態」檔案，例如圖片或樣式表。如果檔案是靜態的，該方法應該回傳磁碟上該靜態檔案的完整路徑。如果傳入的請求不是針對靜態檔案，該方法應回傳 `false`：

    /**
     * Determine if the incoming request is for a static file.
     *
     * @return string|false
     */
    public function isStaticFile(string $sitePath, string $siteName, string $uri)
    {
        if (file_exists($staticFilePath = $sitePath.'/public/'.$uri)) {
            return $staticFilePath;
        }

        return false;
    }

> [!WARNING]  
> `isStaticFile` 方法只有在傳入請求的 `serves` 方法回傳 `true` 且請求 URI 不是 `/` 時才會被呼叫。


<a name="the-frontcontrollerpath-method"></a>
#### `frontControllerPath` 方法

`frontControllerPath` 方法應該回傳您的應用程式「front controller」的完整路徑，這通常是一個「index.php」檔案或等效的檔案：

    /**
     * Get the fully resolved path to the application's front controller.
     */
    public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
    {
        return $sitePath.'/public/index.php';
    }


<a name="local-drivers"></a>
### 本機 Driver

如果您想為單一應用程式定義自訂 Valet driver，請在應用程式的根目錄中建立一個 `LocalValetDriver.php` 檔案。您的自訂 driver 可以擴充基礎的 `ValetDriver` 類別，或擴充現有的應用程式專屬 driver，例如 `LaravelValetDriver`：

    use Valet\Drivers\LaravelValetDriver;

    class LocalValetDriver extends LaravelValetDriver
    {
        /**
         * Determine if the driver serves the request.
         */
        public function serves(string $sitePath, string $siteName, string $uri): bool
        {
            return true;
        }

        /**
         * Get the fully resolved path to the application's front controller.
         */
        public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
        {
            return $sitePath.'/public_html/index.php';
        }
    }


<a name="other-valet-commands"></a>
## 其他 Valet 指令

<div class="overflow-auto">

| Command | 說明 |
| --- | --- |
| `valet list` | 顯示所有 Valet 指令的列表。 |
| `valet diagnose` | 輸出診斷資訊以協助偵錯 Valet。 |
| `valet directory-listing` | 決定目錄列表行為。預設為「off」，這會針對目錄渲染一個 404 頁面。 |
| `valet forget` | 從一個「parked」目錄中執行此指令，以將其從 parked 目錄列表中移除。 |
| `valet log` | 查看由 Valet 服務所寫入的日誌列表。 |
| `valet paths` | 查看所有您的「parked」路徑。 |
| `valet restart` | 重新啟動 Valet 守護程序。 |
| `valet start` | 啟動 Valet 守護程序。 |
| `valet stop` | 停止 Valet 守護程序。 |
| `valet trust` | 為 Brew 和 Valet 新增 sudoers 檔案，以允許 Valet 指令在不提示密碼的情況下執行。 |
| `valet uninstall` | 解除安裝 Valet：顯示手動解除安裝的說明。傳遞 `--force` 選項以強制刪除所有 Valet 的資源。 |

</div>

<a name="valet-directories-and-files"></a>
## Valet 目錄與檔案

當您排查 Valet 環境中的問題時，以下目錄與檔案資訊可能會對您有所幫助：


#### `~/.config/valet`

包含 Valet 的所有設定。您可能希望備份此目錄。


#### `~/.config/valet/dnsmasq.d/`

此目錄包含 DnsMasq 的設定。


#### `~/.config/valet/Drivers/`

此目錄包含 Valet 的 driver。Driver 決定了特定 framework / CMS 的服務方式。


#### `~/.config/valet/Nginx/`

此目錄包含 Valet 的所有 Nginx 站台設定。這些檔案在執行 `install` 和 `secure` 指令時會重新建立。


#### `~/.config/valet/Sites/`

此目錄包含所有 [已連結專案](#the-link-command) 的符號連結。


#### `~/.config/valet/config.json`

此檔案是 Valet 的主要設定檔。


#### `~/.config/valet/valet.sock`

此檔案是 Valet Nginx 安裝所使用的 PHP-FPM socket。此檔案僅在 PHP 正常運行時才會存在。


#### `~/.config/valet/Log/fpm-php.www.log`

此檔案是 PHP 錯誤的使用者日誌。


#### `~/.config/valet/Log/nginx-error.log`

此檔案是 Nginx 錯誤的使用者日誌。


#### `/usr/local/var/log/php-fpm.log`

此檔案是 PHP-FPM 錯誤的系統日誌。


#### `/usr/local/var/log/nginx`

此目錄包含 Nginx 的存取與錯誤日誌。


#### `/usr/local/etc/php/X.X/conf.d`

此目錄包含各種 PHP 設定的 `*.ini` 檔案。


#### `/usr/local/etc/php/X.X/php-fpm.d/valet-fpm.conf`

此檔案是 PHP-FPM pool 的設定檔。


#### `~/.composer/vendor/laravel/valet/cli/stubs/secure.valet.conf`

此檔案是建立您站台 SSL 憑證的預設 Nginx 設定。


<a name="disk-access"></a>
### 磁碟存取

自 macOS 10.14 起，[預設會限制對某些檔案與目錄的存取](https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf)。這些限制包括桌面、文件和下載目錄。此外，網路磁碟區和可移除磁碟區的存取也受到限制。因此，Valet 建議您的站台資料夾位於這些受保護的位置之外。

但是，如果您希望從這些位置之一服務站台，您需要授予 Nginx 「Full Disk Access (完全磁碟存取權限)」。否則，您可能會遇到伺服器錯誤或 Nginx 的其他不可預測行為，尤其是在服務靜態資源時。通常，macOS 會自動提示您授予 Nginx 對這些位置的完全存取權限。或者，您可以透過 `System Preferences` > `Security & Privacy` > `Privacy` 手動完成，並選擇 `Full Disk Access`。接下來，請在主視窗中啟用任何 `nginx` 條目。