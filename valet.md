# Laravel Valet

- [介紹](#introduction)
- [安裝](#installation)
    - [升級 Valet](#upgrading-valet)
- [提供站台](#serving-sites)
    - [`park` 命令](#the-park-command)
    - [`link` 命令](#the-link-command)
    - [使用 TLS 保護站台](#securing-sites)
    - [提供預設站台](#serving-a-default-site)
    - [各站台 PHP 版本](#per-site-php-versions)
- [分享站台](#sharing-sites)
    - [在您的本機網路分享站台](#sharing-sites-on-your-local-network)
- [站台專用環境變數](#site-specific-environment-variables)
- [代理服務](#proxying-services)
- [客製化 Valet Drivers](#custom-valet-drivers)
    - [本機 Drivers](#local-drivers)
- [其他 Valet 命令](#other-valet-commands)
- [Valet 目錄與檔案](#valet-directories-and-files)
    - [磁碟存取](#disk-access)

<a name="introduction"></a>
## 介紹

> [!NOTE]
> 正在尋找在 macOS 或 Windows 上開發 Laravel 應用程式的更簡單方式嗎？請查看 [Laravel Herd](https://herd.laravel.com)。Herd 包含了開始 Laravel 開發所需的一切，包括 Valet、PHP 和 Composer。

[Laravel Valet](https://github.com/laravel/valet) 是一個適用於 macOS 極簡主義者的開發環境。Laravel Valet 會將您的 Mac 配置為在機器啟動時始終於背景執行 [Nginx](https://www.nginx.com/)。然後，透過使用 [DnsMasq](https://en.wikipedia.org/wiki/Dnsmasq)，Valet 會將 `*.test` 網域上的所有請求代理到安裝在本機上的站台。

換句話說，Valet 是一個極其快速的 Laravel 開發環境，僅使用約 7 MB 的記憶體。Valet 並非 [Sail](/docs/{{version}}/sail) 或 [Homestead](/docs/{{version}}/homestead) 的完全替代品，但如果您想要靈活的基礎、偏好極致的速度，或是在記憶體有限的機器上工作，它提供了一個絕佳的替代方案。

開箱即用，Valet 支援以下功能，但不限於此：

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

然而，您可以透過自己的[客製化 drivers](#custom-valet-drivers) 來擴充 Valet。

<a name="installation"></a>
## 安裝

> [!WARNING]
> Valet 需要 macOS 和 [Homebrew](https://brew.sh/)。在安裝之前，您應確保沒有其他程式 (例如 Apache 或 Nginx) 佔用您本機的 80 埠。

首先，您需要使用 `update` 命令來確保 Homebrew 是最新版本：

```shell
brew update
```

接著，您應該使用 Homebrew 來安裝 PHP：

```shell
brew install php
```

安裝 PHP 後，您就可以安裝 [Composer 套件管理器](https://getcomposer.org)。此外，您應確保 `$HOME/.composer/vendor/bin` 目錄位於您系統的「PATH」中。安裝 Composer 後，您可以將 Laravel Valet 作為全域 Composer 套件安裝：

```shell
composer global require laravel/valet
```

最後，您可以執行 Valet 的 `install` 命令。這將配置並安裝 Valet 和 DnsMasq。此外，Valet 依賴的守護行程 (daemons) 將被配置為在您的系統啟動時啟動：

```shell
valet install
```

Valet 安裝完成後，請嘗試在您的終端機上使用 `ping foobar.test` 等命令來 ping 任何 `*.test` 網域。如果 Valet 安裝正確，您應該會看到此網域在 `127.0.0.1` 上回應。

Valet 會在您的機器每次啟動時自動啟動其所需的服務。

<a name="php-versions"></a>
#### PHP 版本

> [!NOTE]
> 您可以指示 Valet 透過 `isolate` [命令](#per-site-php-versions) 來使用各站台的 PHP 版本，而不是修改您的全域 PHP 版本。

Valet 允許您使用 `valet use php@version` 命令來切換 PHP 版本。如果指定的 PHP 版本尚未安裝，Valet 將透過 Homebrew 安裝它：

```shell
valet use php@8.2

valet use php
```

您也可以在專案的根目錄中建立一個 `.valetrc` 檔案。`.valetrc` 檔案應包含站台應使用的 PHP 版本：

```shell
php=php@8.2
```

一旦建立此檔案，您只需執行 `valet use` 命令，該命令就會透過讀取檔案來判斷站台偏好的 PHP 版本。

> [!WARNING]
> Valet 一次只提供一個 PHP 版本，即使您安裝了多個 PHP 版本。

<a name="database"></a>
#### 資料庫

如果您的應用程式需要資料庫，請查看 [DBngin](https://dbngin.com)，它提供了一個免費、一體化的資料庫管理工具，包括 MySQL、PostgreSQL 和 Redis。安裝 DBngin 後，您可以使用 `root` 使用者名稱和空字串作為密碼，連線到 `127.0.0.1` 上的資料庫。

<a name="resetting-your-installation"></a>
#### 重設您的安裝

如果您的 Valet 安裝無法正常運行，執行 `composer global require laravel/valet` 命令，然後執行 `valet install` 將會重設您的安裝，並能解決各種問題。在極少數情況下，可能需要透過執行 `valet uninstall --force` 後再執行 `valet install` 來「硬重設」Valet。

<a name="upgrading-valet"></a>
### 升級 Valet

您可以在終端機中執行 `composer global require laravel/valet` 命令來更新您的 Valet 安裝。升級後，最好運行 `valet install` 命令，以便 Valet 可以在必要時對您的配置檔案進行額外升級。

<a name="upgrading-to-valet-4"></a>
#### 升級至 Valet 4

如果您正在從 Valet 3 升級到 Valet 4，請採取以下步驟來正確升級您的 Valet 安裝：

<div class="content-list" markdown="1">

- 如果您曾新增 `.valetphprc` 檔案來客製化站台的 PHP 版本，請將每個 `.valetphprc` 檔案重新命名為 `.valetrc`。然後，在 `.valetrc` 檔案的現有內容前加上 `php=`。
- 更新任何客製化 drivers，以符合新 driver 系統的命名空間 (namespace)、擴充功能 (extension)、型別提示 (type-hints) 和回傳型別提示 (return type-hints)。您可以參考 Valet 的 [SampleValetDriver](https://github.com/laravel/valet/blob/d7787c025e60abc24a5195dc7d4c5c6f2d984339/cli/stubs/SampleValetDriver.php) 作為範例。
- 如果您使用 PHP 7.1 - 7.4 來提供站台，請確保您仍使用 Homebrew 安裝 PHP 8.0 或更高版本，因為 Valet 將使用此版本 (即使它不是您主要連結的版本) 來運行其部分腳本。

</div>

<a name="serving-sites"></a>
## 提供站台

Valet 安裝完成後，您就可以開始提供您的 Laravel 應用程式。Valet 提供兩個命令來幫助您提供應用程式：`park` 和 `link`。


<a name="the-park-command"></a>
### `park` 命令

`park` 命令會在您的機器上註冊一個包含應用程式的目錄。一旦該目錄使用 Valet 被「parked」(停泊)，該目錄中的所有子目錄都將透過您的網路瀏覽器在 `http://<directory-name>.test` 存取：

```shell
cd ~/Sites

valet park
```

就這麼簡單。現在，您在「parked」(停泊)目錄中建立的任何應用程式都將自動使用 `http://<directory-name>.test` 的慣例來提供服務。因此，如果您的「parked」(停泊)目錄包含一個名為「laravel」的目錄，該目錄中的應用程式將可以透過 `http://laravel.test` 存取。此外，Valet 還會自動讓您使用萬用字元子網域 (`http://foo.laravel.test`) 來存取站台。


<a name="the-link-command"></a>
### `link` 命令

`link` 命令也可以用來提供您的 Laravel 應用程式。如果您想在一個目錄中提供單一站台而不是整個目錄，這個命令會很有用：

```shell
cd ~/Sites/laravel

valet link
```

一旦應用程式使用 `link` 命令連結到 Valet，您就可以使用其目錄名稱來存取該應用程式。因此，上述範例中連結的站台可以透過 `http://laravel.test` 存取。此外，Valet 還會自動讓您使用萬用字元子網域 (`http://foo.laravel.test`) 來存取站台。

如果您想在不同的主機名稱提供應用程式，可以將主機名稱傳遞給 `link` 命令。例如，您可以執行以下命令，讓應用程式可以透過 `http://application.test` 存取：

```shell
cd ~/Sites/laravel

valet link application
```

當然，您也可以使用 `link` 命令在子網域上提供應用程式：

```shell
valet link api.application
```

您可以執行 `links` 命令來顯示所有已連結目錄的列表：

```shell
valet links
```

`unlink` 命令可以用來銷毀站台的符號連結：

```shell
cd ~/Sites/laravel

valet unlink
```


<a name="securing-sites"></a>
### 使用 TLS 保護站台

預設情況下，Valet 透過 HTTP 提供站台。然而，如果您想透過使用 HTTP/2 的加密 TLS 來提供站台，您可以使用 `secure` 命令。例如，如果您的站台由 Valet 在 `laravel.test` 網域上提供服務，您應該執行以下命令來保護它：

```shell
valet secure laravel
```

若要「取消保護」站台並將其流量恢復為純 HTTP 提供，請使用 `unsecure` 命令。與 `secure` 命令一樣，此命令接受您希望取消保護的主機名稱：

```shell
valet unsecure laravel
```


<a name="serving-a-default-site"></a>
### 提供預設站台

有時，您可能希望設定 Valet，使其在訪問未知 `test` 網域時提供「預設」站台，而不是 `404` 錯誤。為此，您可以將 `default` 選項新增到您的 `~/.config/valet/config.json` 設定檔中，其中包含應作為預設站台的路徑：

    "default": "/Users/Sally/Sites/example-site",


<a name="per-site-php-versions"></a>
### 各站台 PHP 版本

預設情況下，Valet 使用您的全域 PHP 安裝來提供站台。然而，如果您需要在不同站台支援多個 PHP 版本，您可以使用 `isolate` 命令來指定特定站台應使用的 PHP 版本。`isolate` 命令會將 Valet 設定為對您目前工作目錄中的站台使用指定的 PHP 版本：

```shell
cd ~/Sites/example-site

valet isolate php@8.0
```

如果您的站台名稱與包含它的目錄名稱不符，您可以使用 `--site` 選項指定站台名稱：

```shell
valet isolate php@8.0 --site="site-name"
```

為了方便起見，您可以使用 `valet php`、`composer` 和 `which-php` 命令，根據站台設定的 PHP 版本將呼叫代理到適當的 PHP CLI 或工具：

```shell
valet php
valet composer
valet which-php
```

您可以執行 `isolated` 命令來顯示所有隔離站台及其 PHP 版本的列表：

```shell
valet isolated
```

要將站台恢復到 Valet 全域安裝的 PHP 版本，您可以在站台的根目錄中呼叫 `unisolate` 命令：

```shell
valet unisolate
```


<a name="sharing-sites"></a>
## 分享站台

Valet 包含一個命令，可將您的本機站台分享給世界，提供一種輕鬆的方式，可在行動裝置上測試您的站台或與團隊成員和客戶分享。

開箱即用，Valet 支援透過 ngrok 或 Expose 分享您的站台。在分享站台之前，您應該使用 `share-tool` 命令更新您的 Valet 設定，指定 `ngrok`、`expose` 或 `cloudflared`：

```shell
valet share-tool ngrok
```

如果您選擇一個工具，但尚未透過 Homebrew (適用於 ngrok 和 cloudflared) 或 Composer (適用於 Expose) 安裝它，Valet 將自動提示您安裝。當然，這兩種工具都要求您在開始分享站台之前驗證您的 ngrok 或 Expose 帳戶。

要分享站台，請在您的終端機中導航到該站台的目錄並執行 Valet 的 `share` 命令。一個可公開存取的 URL 將被複製到您的剪貼簿中，可以直接貼到瀏覽器或分享給您的團隊：

```shell
cd ~/Sites/laravel

valet share
```

若要停止分享您的站台，您可以按下 `Control + C`。

> [!WARNING]
> 如果您使用自訂 DNS 伺服器 (例如 `1.1.1.1`)，ngrok 分享可能無法正常運作。如果您的機器上發生這種情況，請打開 Mac 的系統設定，前往「網路」設定，打開「進階」設定，然後前往「DNS」標籤並將 `127.0.0.1` 新增為您的第一個 DNS 伺服器。


<a name="sharing-sites-via-ngrok"></a>
#### 透過 Ngrok 分享站台

透過 ngrok 分享您的站台需要您[建立一個 ngrok 帳戶](https://dashboard.ngrok.com/signup)並[設定一個驗證令牌](https://dashboard.ngrok.com/get-started/your-authtoken)。一旦您擁有驗證令牌，您就可以使用該令牌更新您的 `Valet` 設定：

```shell
valet set-ngrok-token YOUR_TOKEN_HERE
```

> [!NOTE]
> 您可以將額外的 ngrok 參數傳遞給 share 命令，例如 `valet share --region=eu`。欲了解更多資訊，請查閱 [ngrok 文件](https://ngrok.com/docs)。


<a name="sharing-sites-via-expose"></a>
#### 透過 Expose 分享站台

透過 Expose 分享您的站台需要您[建立一個 Expose 帳戶](https://expose.dev/register)並[透過您的驗證令牌進行 Expose 認證](https://expose.dev/docs/getting-started/getting-your-token)。

您可以查閱 [Expose 文件](https://expose.dev/docs)以了解其支援的額外命令列參數資訊。


<a name="sharing-sites-on-your-local-network"></a>
### 在您的本機網路分享站台

預設情況下，Valet 將傳入流量限制在內部 `127.0.0.1` 介面，以避免您的開發機器暴露於網際網路的安全風險。

如果您希望允許本機網路上的其他裝置透過您機器的 IP 位址 (例如：`192.168.1.10/application.test`) 存取您機器上的 Valet 站台，您將需要手動編輯該站台的相應 Nginx 設定檔，以移除對 `listen` 指令的限制。您應該移除 `listen` 指令中埠 80 和 443 上的 `127.0.0.1:` 前綴。

如果您尚未在專案上執行 `valet secure`，您可以透過編輯 `/usr/local/etc/nginx/valet/valet.conf` 檔案來開放所有非 HTTPS 站台的網路存取。然而，如果您是透過 HTTPS 提供專案站台 (您已為該站台執行過 `valet secure`)，那麼您應該編輯 `~/.config/valet/Nginx/app-name.test` 檔案。

更新 Nginx 設定後，執行 `valet restart` 命令以應用設定變更。

<a name="site-specific-environment-variables"></a>
## 站台專用環境變數

有些使用其他框架的應用程式可能依賴伺服器環境變數，但未提供在專案中配置這些變數的方法。Valet 允許您透過在專案的根目錄中加入一個 `.valet-env.php` 檔案來配置站台專屬的環境變數。此檔案應回傳一個站台 / 環境變數鍵值對陣列，這些鍵值對將被加入到全域 `$_SERVER` 陣列中，針對陣列中指定的每個站台：

```php
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
```


<a name="proxying-services"></a>
## 代理服務

有時您可能希望代理 (proxy) 一個 Valet 網域到本機上的另一個服務。例如，您可能偶爾需要執行 Valet，同時也在 Docker 中執行一個獨立的站台；然而，Valet 和 Docker 無法同時繫結到埠號 80。

為了解決這個問題，您可以使用 `proxy` 命令來建立代理。例如，您可以代理所有來自 `http://elasticsearch.test` 的流量到 `http://127.0.0.1:9200`：

```shell

```
# Proxy over HTTP...
valet proxy elasticsearch http://127.0.0.1:9200
```

```markdown
# Proxy over TLS + HTTP/2...
valet proxy elasticsearch http://127.0.0.1:9200 --secure
```

您可以使用 `unproxy` 命令來移除代理：

```shell
valet unproxy elasticsearch
```

您可以使用 `proxies` 命令列出所有已代理的站台設定：

```shell
valet proxies
```


<a name="custom-valet-drivers"></a>
## 客製化 Valet Drivers

您可以編寫自己的 Valet "driver" 來服務那些 Valet 不原生支援、運行於框架或 CMS 上的 PHP 應用程式。當您安裝 Valet 時，會建立一個 `~/.config/valet/Drivers` 目錄，其中包含一個 `SampleValetDriver.php` 檔案。此檔案包含一個範例 driver 實作，以展示如何編寫客製化 driver。編寫 driver 僅需要您實作三個方法：`serves`、`isStaticFile` 和 `frontControllerPath`。

所有這三個方法都會接收 `$sitePath`、`$siteName` 和 `$uri` 值作為參數。`$sitePath` 是您機器上所服務站台的完整路徑，例如 `/Users/Lisa/Sites/my-project`。`$siteName` 是網域的「host」/「site name」部分 (`my-project`)。`$uri` 是傳入的請求 URI (`/foo/bar`)。

完成客製化 Valet driver 後，請將其放置在 `~/.config/valet/Drivers` 目錄中，並使用 `FrameworkValetDriver.php` 命名慣例。例如，如果您正在為 WordPress 編寫客製化 valet driver，您的檔案名稱應該是 `WordPressValetDriver.php`。

讓我們看看您的客製化 Valet driver 應實作的每個方法的範例。


<a name="the-serves-method"></a>
#### `serves` 方法

如果您的 driver 應該處理傳入的請求，`serves` 方法應返回 `true`。否則，該方法應返回 `false`。因此，在這個方法中，您應該嘗試判斷給定的 `$sitePath` 是否包含您嘗試服務的專案類型。

例如，假設我們正在編寫一個 `WordPressValetDriver`。我們的 `serves` 方法可能看起來像這樣：

```php
/**
 * Determine if the driver serves the request.
 */
public function serves(string $sitePath, string $siteName, string $uri): bool
{
    return is_dir($sitePath.'/wp-admin');
}
```


<a name="the-isstaticfile-method"></a>
#### `isStaticFile` 方法

`isStaticFile` 應判斷傳入的請求是否針對「靜態」檔案，例如圖片或樣式表。如果檔案是靜態的，該方法應返回磁碟上靜態檔案的完整路徑。如果傳入的請求不是靜態檔案，該方法應返回 `false`：

```php
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
```

> [!WARNING]
> `isStaticFile` 方法只有在 `serves` 方法對傳入請求返回 `true` 且請求 URI 不是 `/` 時才會被呼叫。


<a name="the-frontcontrollerpath-method"></a>
#### `frontControllerPath` 方法

`frontControllerPath` 方法應返回您應用程式「前導控制器」的完整路徑，通常是「index.php」檔案或等效檔案：

```php
/**
 * Get the fully resolved path to the application's front controller.
 */
public function frontControllerPath(string $sitePath, string $siteName, string $uri): string
{
    return $sitePath.'/public/index.php';
}
```


<a name="local-drivers"></a>
### 本機 Drivers

如果您想為單一應用程式定義客製化 Valet driver，請在應用程式的根目錄中建立一個 `LocalValetDriver.php` 檔案。您的客製化 driver 可以擴展基礎的 `ValetDriver` 類別，或擴展現有的應用程式特定 driver，例如 `LaravelValetDriver`：

```php
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
```


<a name="other-valet-commands"></a>
## 其他 Valet 命令

<div class="overflow-auto">

| Command | Description |
| --- | --- |
| `valet list` | 顯示所有 Valet 命令的列表。 |
| `valet diagnose` | 輸出診斷資訊以協助偵錯 Valet。 |
| `valet directory-listing` | 決定目錄列表行為。預設為「off」，會為目錄渲染一個 404 頁面。 |
| `valet forget` | 在「parked」目錄中執行此命令，將其從已停泊的目錄列表中移除。 |
| `valet log` | 查看 Valet 服務所寫入的日誌列表。 |
| `valet paths` | 查看所有您「parked」的路徑。 |
| `valet restart` | 重新啟動 Valet 常駐程式 (daemons)。 |
| `valet start` | 啟動 Valet 常駐程式 (daemons)。 |
| `valet stop` | 停止 Valet 常駐程式 (daemons)。 |
| `valet trust` | 為 Brew 和 Valet 加入 sudoers 檔案，以允許 Valet 命令在不提示密碼的情況下運行。 |
| `valet uninstall` | 解除安裝 Valet：顯示手動解除安裝的說明。傳遞 `--force` 選項將強制刪除所有 Valet 資源。 |

</div>
```

<a name="valet-directories-and-files"></a>
## Valet 目錄與檔案

在解決 Valet 環境問題時，您可能會發現以下目錄和檔案資訊很有幫助：

#### `~/.config/valet`

包含 Valet 的所有設定。您可能希望備份此目錄。

#### `~/.config/valet/dnsmasq.d/`

此目錄包含 DNSMasq 的設定。

#### `~/.config/valet/Drivers/`

此目錄包含 Valet 的驅動程式 (drivers)。驅動程式決定了特定框架 / CMS 如何被提供服務。

#### `~/.config/valet/Nginx/`

此目錄包含 Valet 的所有 Nginx 站台設定。這些檔案在執行 `install` 和 `secure` 命令時會重建。

#### `~/.config/valet/Sites/`

此目錄包含您 [連結專案](#the-link-command) 的所有符號連結。

#### `~/.config/valet/config.json`

此檔案是 Valet 的主設定檔。

#### `~/.config/valet/valet.sock`

此檔案是 Valet 的 Nginx 安裝所使用的 PHP-FPM socket。僅在 PHP 正常運行時才會存在。

#### `~/.config/valet/Log/fpm-php.www.log`

此檔案是 PHP 錯誤的使用者日誌。

#### `~/.config/valet/Log/nginx-error.log`

此檔案是 Nginx 錯誤的使用者日誌。

#### `/usr/local/var/log/php-fpm.log`

此檔案是 PHP-FPM 錯誤的系統日誌。

#### `/usr/local/var/log/nginx`

此目錄包含 Nginx 的存取日誌和錯誤日誌。

#### `/usr/local/etc/php/X.X/conf.d`

此目錄包含各種 PHP 設定的 `*.ini` 檔案。

#### `/usr/local/etc/php/X.X/php-fpm.d/valet-fpm.conf`

此檔案是 PHP-FPM 處理程序池的設定檔。

#### `~/.composer/vendor/laravel/valet/cli/stubs/secure.valet.conf`

此檔案是建立您站台 SSL 憑證所使用的預設 Nginx 設定檔。

<a name="disk-access"></a>
### 磁碟存取

從 macOS 10.14 開始，[對某些檔案和目錄的存取預設受到限制](https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf)。這些限制包括桌面 (Desktop)、文件 (Documents) 和下載 (Downloads) 目錄。此外，網路磁碟區和可移除磁碟區的存取也受到限制。因此，Valet 建議您的站台資料夾位於這些受保護位置之外。

然而，如果您希望從這些位置之一提供站台服務，您將需要授予 Nginx 「完整磁碟存取權限」(Full Disk Access)。否則，您可能會遇到伺服器錯誤或 Nginx 的其他不可預測行為，尤其是在提供靜態資源時。通常，macOS 會自動提示您授予 Nginx 對這些位置的完整存取權限。或者，您可以透過 `System Preferences` > `Security & Privacy` > `Privacy` 手動執行此操作，並選擇 `Full Disk Access`。接著，在主視窗中啟用任何 `nginx` 項目。