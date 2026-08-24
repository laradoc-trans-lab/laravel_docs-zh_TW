# Laravel Valet

- [簡介](#introduction)
- [安裝](#installation)
    - [升級 Valet](#upgrading-valet)
- [提供網站服務](#serving-sites)
    - [`park` 指令](#the-park-command)
    - [`link` 指令](#the-link-command)
    - [使用 TLS 加密網站](#securing-sites)
    - [設定預設網站](#serving-a-default-site)
    - [個別網站的 PHP 版本](#per-site-php-versions)
- [分享網站](#sharing-sites)
    - [在區域網路中分享網站](#sharing-sites-on-your-local-network)
- [特定網站的環境變數](#site-specific-environment-variables)
- [代理服務](#proxying-services)
- [自訂 Valet 驅動程式](#custom-valet-drivers)
    - [區域驅動程式](#local-drivers)
- [其他 Valet 指令](#other-valet-commands)
- [Valet 目錄與檔案](#valet-directories-and-files)
    - [磁碟存取權限](#disk-access)

<a name="introduction"></a>
## 簡介

> [!NOTE]
> 正在尋找在 macOS 或 Windows 上開發 Laravel 應用程式更簡單的方法嗎？不妨參考 [Laravel Herd](https://herd.laravel.com)。Herd 包含開始進行 Laravel 開發所需的一切，包括 Valet、PHP 和 Composer。

[Laravel Valet](https://github.com/laravel/valet) 是專為極簡主義者打造的 macOS 開發環境。Laravel Valet 會將你的 Mac 設定為在開機時始終於背景執行 [Nginx](https://www.nginx.com/)。接著，Valet 利用 [DnsMasq](https://en.wikipedia.org/wiki/Dnsmasq) 將所有對 `*.test` 網域的請求代理指向安裝在你本機電腦上的網站。

換句話說，Valet 是一個速度極快、僅需消耗約 7 MB 記憶體 (RAM) 的 Laravel 開發環境。Valet 並非要完全取代 [Sail](/docs/{{version}}/sail) 或 [Homestead](/docs/{{version}}/homestead)，但如果你偏好靈活基礎的設定、追求極致速度，或者是在記憶體有限的機器上工作，它提供了一個絕佳的替代方案。

開箱即用，Valet 支援的項目包含但不限於：

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

不過，你也可以使用自己的[自訂驅動程式](#custom-valet-drivers)來擴充 Valet。


<a name="installation"></a>
## 安裝

> [!WARNING]
> Valet 需要 macOS 與 [Homebrew](https://brew.sh/)。在安裝前，你應該確保沒有其他程式（例如 Apache 或 Nginx）綁定到本機電腦的 Port 80。

首先，你需要使用 `update` 指令確保 Homebrew 是最新版本：

```shell
brew update
```

接下來，你應該使用 Homebrew 來安裝 PHP：

```shell
brew install php
```

安裝 PHP 後，你就可以安裝 [Composer 套件包管理器](https://getcomposer.org)。此外，你應確保 `$HOME/.composer/vendor/bin` 目錄已包含在系統的「PATH」環境變數中。安裝 Composer 後，你可以將 Laravel Valet 安裝為全域的 Composer 套件：

```shell
composer global require laravel/valet
```

最後，你可以執行 Valet 的 `install` 指令。這將會設定並安裝 Valet 和 DnsMasq。此外，Valet 所依賴的背景常駐程序 (Daemon) 也會設定為在系統啟動時隨之啟動：

```shell
valet install
```

Valet 安裝完成後，請試著在終端機中使用類似 `ping foobar.test` 的指令 ping 任何 `*.test` 網域。如果 Valet 安裝無誤，你應該會看到該網域回應 `127.0.0.1`。

每次電腦開機時，Valet 都會自動啟動其所需的服務。


<a name="php-versions"></a>
#### PHP 版本

> [!NOTE]
> 除了修改全域的 PHP 版本外，你也可以透過 `isolate` [指令](#per-site-php-versions)指示 Valet 為個別網站使用不同的 PHP 版本。

Valet 允許你使用 `valet use php@version` 指令來切換 PHP 版本。如果尚未安裝該版本，Valet 會自動透過 Homebrew 安裝指定的 PHP 版本：

```shell
valet use php@8.2

valet use php
```

你也可以在專案根目錄中建立一個 `.valetrc` 檔案。`.valetrc` 檔案中應包含該網站應使用的 PHP 版本：

```shell
php=php@8.2
```

建立此檔案後，你只需執行 `valet use` 指令，該指令就會透過讀取該檔案來確定網站偏好的 PHP 版本。

> [!WARNING]
> 即使你安裝了多個 PHP 版本，Valet 一次也只會提供一個 PHP 版本的服務。


<a name="database"></a>
#### 資料庫

如果你的應用程式需要資料庫，可以參考 [DBngin](https://dbngin.com)，它提供了一個免費且多合一的資料庫管理工具，包含 MySQL、PostgreSQL 與 Redis。安裝 DBngin 後，你可以使用使用者名稱 `root` 與空字串密碼連接到 `127.0.0.1` 的資料庫。


<a name="resetting-your-installation"></a>
#### 重置你的安裝

如果你在讓 Valet 安裝順利運作時遇到困難，執行 `composer global require laravel/valet` 指令接著執行 `valet install` 將會重置你的安裝，並能解決各種問題。在極少數情況下，可能需要透過執行 `valet uninstall --force` 接著執行 `valet install` 來對 Valet 進行「強制重置 (Hard reset)」。


<a name="upgrading-valet"></a>
### 升級 Valet

你可以在終端機中執行 `composer global require laravel/valet` 指令來更新你的 Valet 安裝。升級後，良好的習慣是執行 `valet install` 指令，以便 Valet 在需要時能對你的設定檔進行額外升級。


<a name="upgrading-to-valet-4"></a>
#### 升級至 Valet 4

如果你要從 Valet 3 升級到 Valet 4，請採取以下步驟以正確升級你的 Valet 安裝：

<div class="content-list" markdown="1">

- 如果你曾建立 `.valetphprc` 檔案來自訂網站的 PHP 版本，請將每個 `.valetphprc` 檔案更名為 `.valetrc`。接著，在 `.valetrc` 檔案現有內容的前面加上 `php=`。
- 更新任何自訂驅動程式，以符合新驅動程式系統的命名空間 (Namespace)、擴充 (Extension)、型別提示 (Type-hints) 和回傳型別提示 (Return type-hints)。你可以參考 Valet 的 [SampleValetDriver](https://github.com/laravel/valet/blob/d7787c025e60abc24a5195dc7d4c5c6f2d984339/cli/stubs/SampleValetDriver.php) 作為範例。
- 如果你使用 PHP 7.1 - 7.4 來提供網站服務，請確保你仍透過 Homebrew 安裝了 8.0 或更高的 PHP 版本，因為即使該版本不是你主要連結的版本，Valet 也會使用它來執行其中的一些腳本。

</div>

<a name="serving-sites"></a>
## 提供網站服務

安裝好 Valet 後，您就可以開始為您的 Laravel 應用程式提供網站服務。Valet 提供兩個指令來幫助您提供應用程式服務：`park` 與 `link`。


<a name="the-park-command"></a>
### `park` 指令

`park` 指令會在您的電腦上註冊一個包含您應用程式的目錄。一旦該目錄透過 Valet 被「Park (停泊)」，該目錄下的所有目錄都可以透過 Web 瀏覽器在 `http://<directory-name>.test` 存取：

```shell
cd ~/Sites

valet park
```

就是這麼簡單。現在，您在「parked」目錄中建立的任何應用程式都將自動使用 `http://<directory-name>.test` 慣例來提供服務。因此，如果您的 parked 目錄包含一個名為「laravel」的目錄，則該目錄中的應用程式將可以透過 `http://laravel.test` 存取。此外，Valet 還自動允許您使用萬用字元子網域（`http://foo.laravel.test`）存取該網站。


<a name="the-link-command"></a>
### `link` 指令

`link` 指令也可以用來提供您的 Laravel 應用程式服務。如果您只想提供目錄中的單一網站服務，而不是整個目錄，則此指令非常實用：

```shell
cd ~/Sites/laravel

valet link
```

一旦應用程式使用 `link` 指令連結到 Valet，您就可以使用其目錄名稱存取該應用程式。因此，上例中連結的網站可以在 `http://laravel.test` 存取。此外，Valet 自動允許您使用萬用字元子網域（`http://foo.laravel.test`）存取該網站。

如果您想在不同的主機名稱上提供應用程式服務，可以將主機名稱傳遞給 `link` 指令。例如，您可以執行以下指令讓應用程式在 `http://application.test` 上可用：

```shell
cd ~/Sites/laravel

valet link application
```

當然，您也可以使用 `link` 指令在子網域上提供應用程式服務：

```shell
valet link api.application
```

您可以執行 `links` 指令來顯示所有已連結目錄的列表：

```shell
valet links
```

`unlink` 指令可用於刪除網站的符號連結（symbolic link）：

```shell
cd ~/Sites/laravel

valet unlink
```


<a name="securing-sites"></a>
### 使用 TLS 加密網站

預設情況下，Valet 透過 HTTP 提供網站服務。但是，如果您想透過使用 HTTP/2 的加密 TLS 提供網站服務，可以使用 `secure` 指令。例如，如果您的網站是由 Valet 在 `laravel.test` 網域上提供服務，您應該執行以下指令來保護它：

```shell
valet secure laravel
```

要對網站「解除安全設定」並恢復為透過純 HTTP 提供流量服務，請使用 `unsecure` 指令。與 `secure` 指令一樣，此指令接受您希望解除安全設定的主機名稱：

```shell
valet unsecure laravel
```


<a name="serving-a-default-site"></a>
### 設定預設網站

有時候，您可能希望將 Valet 設定為在存取未知的 `test` 網域時提供「預設」網站，而不是傳回 `404`。要實現此目的，您可以在 `~/.config/valet/config.json` 設定檔中新增 `default` 選項，其中包含應作為預設網站的網站路徑：

    "default": "/Users/Sally/Sites/example-site",


<a name="per-site-php-versions"></a>
### 個別網站的 PHP 版本

預設情況下，Valet 使用全域安裝的 PHP 來為您的網站提供服務。然而，如果您需要在不同網站之間支援多個 PHP 版本，可以使用 `isolate` 指令來指定特定網站應使用的 PHP 版本。`isolate` 指令會將 Valet 設定為對位於您目前工作目錄中的網站使用指定的 PHP 版本：

```shell
cd ~/Sites/example-site

valet isolate php@8.0
```

如果您的網站名稱與包含它的目錄名稱不符，可以使用 `--site` 選項來指定網站名稱：

```shell
valet isolate php@8.0 --site="site-name"
```

為方便起見，您可以使用 `valet php`、`composer` 與 `which-php` 指令，根據網站設定的 PHP 版本，將呼叫代理到相應的 PHP CLI 或工具：

```shell
valet php
valet composer
valet which-php
```

您可以執行 `isolated` 指令來顯示所有隔離網站及其 PHP 版本的列表：

```shell
valet isolated
```

要將網站恢復為 Valet 全域安裝的 PHP 版本，您可以在網站根目錄中執行 `unisolate` 指令：

```shell
valet unisolate
```


<a name="sharing-sites"></a>
## 分享網站

Valet 包含一個與全世界分享您本機網站的指令，提供了一種在行動裝置上測試網站或與團隊成員及客戶分享網站的簡單方法。

開箱即用，Valet 支援透過 ngrok 或 Expose 分享您的網站。在分享網站之前，您應該使用 `share-tool` 指令更新 Valet 設定，指定 `ngrok`、`expose` 或 `cloudflared`：

```shell
valet share-tool ngrok
```

如果您選擇了一個工具，但尚未透過 Homebrew（針對 ngrok 與 cloudflared）或 Composer（針對 Expose）安裝它，Valet 將會自動提示您進行安裝。當然，這兩個工具都需要您在開始分享網站之前先驗證您的 ngrok 或 Expose 帳號。

要分享網站，請在終端機中切換到該網站的目錄，然後執行 Valet 的 `share` 指令。一個可供公開存取的 URL 將會複製到您的剪貼簿中，隨時可以直接貼上到您的瀏覽器或與您的團隊分享：

```shell
cd ~/Sites/laravel

valet share
```

要停止分享您的網站，可以按下 `Control + C`。

> [!WARNING]
> 如果您使用自訂 DNS 伺服器（例如 `1.1.1.1`），ngrok 分享可能無法正常運作。如果您的電腦遇到這種情況，請開啟 Mac 的系統設定，前往「網路」設定，開啟「進階」設定，然後前往「DNS」標籤頁並將 `127.0.0.1` 新增為您的第一個 DNS 伺服器。


<a name="sharing-sites-via-ngrok"></a>
#### 透過 Ngrok 分享網站

使用 ngrok 分享網站需要您[建立一個 ngrok 帳號](https://dashboard.ngrok.com/signup)並[設定驗證令牌](https://dashboard.ngrok.com/get-started/your-authtoken)。擁有驗證令牌後，您可以透過該令牌更新您的 Valet 設定：

```shell
valet set-ngrok-token YOUR_TOKEN_HERE
```

> [!NOTE]
> 您可以向 share 指令傳遞額外的 ngrok 參數，例如 `valet share --region=eu`。更多資訊請參閱 [ngrok 說明文件](https://ngrok.com/docs)。


<a name="sharing-sites-via-expose"></a>
#### 透過 Expose 分享網站

使用 Expose 分享網站需要您[建立一個 Expose 帳號](https://expose.dev/register)並[透過您的驗證令牌進行 Expose 驗證](https://expose.dev/docs/getting-started/getting-your-token)。

您可以參閱 [Expose 說明文件](https://expose.dev/docs)以取得有關其支援的額外命令列參數的資訊。


<a name="sharing-sites-on-your-local-network"></a>
### 在區域網路中分享網站

Valet 預設將傳入流量限制在內部的 `127.0.0.1` 介面，因此您的開發電腦不會暴露於來自網際網路的安全風險中。

如果您希望允許區域網路上的其他裝置透過您電腦的 IP 位址（例如：`192.168.1.10/application.test`）存取您電腦上的 Valet 網站，您需要手動編輯該網站相應的 Nginx 設定檔，以解除對 `listen` 指令的限制。您應該刪除埠號 80 與 443 的 `listen` 指令上的 `127.0.0.1:` 前綴。

如果您尚未對專案執行 `valet secure`，可以透過編輯 `/usr/local/etc/nginx/valet/valet.conf` 檔案來開放所有非 HTTPS 網站的網路存取權限。但是，如果您透過 HTTPS 提供專案網站服務（您已對該網站執行 `valet secure`），則應該編輯 `~/.config/valet/Nginx/app-name.test` 檔案。

更新 Nginx 設定後，請執行 `valet restart` 指令以套用設定變更。

<a name="site-specific-environment-variables"></a>
## 特定網站的環境變數

某些使用其他框架的應用程式可能會依賴伺服器環境變數，但未提供在專案內部設定這些變數的方法。Valet 允許你透過在專案根目錄中新增 `.valet-env.php` 檔案來設定特定網站的環境變數。該檔案應回傳一個網站／環境變數配對的陣列，這些變數將會被新增至陣列中指定的每個網站的全域 `$_SERVER` 陣列中：

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

有時你可能希望將 Valet 網域代理至本機上的另一個服務。例如，你可能偶爾需要在執行 Valet 的同時於 Docker 中執行另一個獨立網站；然而，Valet 與 Docker 無法同時綁定到 Port 80。

為了妥善解決此問題，你可以使用 `proxy` 指令來產生代理。例如，你可以將來自 `http://elasticsearch.test` 的所有流量代理到 `http://127.0.0.1:9200`：

```shell
# Proxy over HTTP...
valet proxy elasticsearch http://127.0.0.1:9200

# Proxy over TLS + HTTP/2...
valet proxy elasticsearch http://127.0.0.1:9200 --secure
```

你可以使用 `unproxy` 指令移除代理：

```shell
valet unproxy elasticsearch
```

你可以使用 `proxies` 指令來列出所有已設定代理的網站設定：

```shell
valet proxies
```


<a name="custom-valet-drivers"></a>
## 自訂 Valet 驅動程式

你可以撰寫自己的 Valet「驅動程式」，以提供服務給執行於原生未被 Valet 支援的框架或 CMS 上的 PHP 應用程式。當你安裝 Valet 時，會建立一個 `~/.config/valet/Drivers` 目錄，其中包含一個 `SampleValetDriver.php` 檔案。此檔案包含示範驅動程式實作，說明如何撰寫自訂驅動程式。撰寫驅動程式只需要你實作三個方法：`serves`、`isStaticFile` 和 `frontControllerPath`。

這三個方法都會接收 `$sitePath`、`$siteName` 以及 `$uri` 數值作為其引數。`$sitePath` 是在你的電腦上提供服務之網站的完整路徑（Fully qualified path），例如 `/Users/Lisa/Sites/my-project`。`$siteName` 是網域的「主機 (Host)」／「網站名稱」部分（`my-project`）。`$uri` 則是傳入的請求 URI（`/foo/bar`）。

當你完成自訂 Valet 驅動程式後，請將其放置在 `~/.config/valet/Drivers` 目錄中，並遵循 `FrameworkValetDriver.php` 的命名慣例。例如，如果你正為 WordPress 撰寫自訂 valet 驅動程式，你的檔名應該是 `WordPressValetDriver.php`。

讓我們來看看你自訂的 Valet 驅動程式應該實作的各個方法之範例。


<a name="the-serves-method"></a>
#### `serves` 方法

若你的驅動程式應該處理傳入的請求，則 `serves` 方法應回傳 `true`。否則，該方法應回傳 `false`。因此，在此方法中，你應該嘗試判定給定的 `$sitePath` 是否包含你試圖提供服務之類型的專案。

例如，假設我們正在撰寫一個 `WordPressValetDriver`。我們的 `serves` 方法可能看起來像這樣：

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

`isStaticFile` 方法應判定傳入的請求是否為「靜態」檔案，例如圖片或樣式表。如果檔案是靜態的，該方法應回傳磁碟上該靜態檔案的完整路徑。若傳入的請求不是針對靜態檔案，該方法應回傳 `false`：

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
> 僅當傳入的請求使 `serves` 方法回傳 `true` 且請求 URI 不為 `/` 時，才會呼叫 `isStaticFile` 方法。


<a name="the-frontcontrollerpath-method"></a>
#### `frontControllerPath` 方法

`frontControllerPath` 方法應回傳應用程式「前端控制器 (Front Controller)」的完整路徑，通常為 "index.php" 檔案或同等檔案：

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
### 區域驅動程式

如果你想為單一應用程式定義自訂 Valet 驅動程式，請在應用程式的根目錄中建立 `LocalValetDriver.php` 檔案。你的自訂驅動程式可以繼承基底 `ValetDriver` 類別，或繼承現有特定應用程式的驅動程式，例如 `LaravelValetDriver`：

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
## 其他 Valet 指令

<div class="overflow-auto">

| 指令 | 說明 |
| --- | --- |
| `valet list` | 顯示所有 Valet 指令的清單。 |
| `valet diagnose` | 輸出診斷資訊以協助除錯 Valet。 |
| `valet directory-listing` | 設定目錄列表行為。預設為 "off"，當存取目錄時會渲染 404 頁面。 |
| `valet forget` | 從「停泊 (Parked)」目錄中執行此指令，以將其從停泊目錄清單中移除。 |
| `valet log` | 查看由 Valet 服務所寫入的日誌清單。 |
| `valet paths` | 查看你所有的「停泊」路徑。 |
| `valet restart` | 重啟 Valet 守護程序 (Daemons)。 |
| `valet start` | 啟動 Valet 守護程序 (Daemons)。 |
| `valet stop` | 停止 Valet 守護程序 (Daemons)。 |
| `valet trust` | 為 Brew 與 Valet 新增 sudoers 設定檔，以允許執行 Valet 指令時無需提示輸入密碼。 |
| `valet uninstall` | 解除安裝 Valet：顯示手動解除安裝的說明。傳入 `--force` 選項可強制刪除 Valet 的所有資源。 |

</div>

<a name="valet-directories-and-files"></a>
## Valet 目錄與檔案

在對你的 Valet 環境進行故障排除時，以下目錄與檔案資訊可能會有所幫助：


#### `~/.config/valet`

包含 Valet 的所有設定。你可能希望保留此目錄的備份。


#### `~/.config/valet/dnsmasq.d/`

此目錄包含 DnsMasq 的設定。


#### `~/.config/valet/Drivers/`

此目錄包含 Valet 的驅動程式。驅動程式決定了特定框架 / CMS 如何提供服務。


#### `~/.config/valet/Nginx/`

此目錄包含所有 Valet 的 Nginx 網站設定。這些檔案會在執行 `install` 和 `secure` 指令時重新建置。


#### `~/.config/valet/Sites/`

此目錄包含你[連結的專案](#the-link-command)的所有符號連結。


#### `~/.config/valet/config.json`

此檔案為 Valet 的主設定檔。


#### `~/.config/valet/valet.sock`

此檔案是 Valet 的 Nginx 安裝所使用的 PHP-FPM Socket。只有在 PHP 正常運作時才會存在。


#### `~/.config/valet/Log/fpm-php.www.log`

此檔案是 PHP 錯誤的使用者記錄檔。


#### `~/.config/valet/Log/nginx-error.log`

此檔案是 Nginx 錯誤的使用者記錄檔。


#### `/usr/local/var/log/php-fpm.log`

此檔案是 PHP-FPM 錯誤的系統記錄檔。


#### `/usr/local/var/log/nginx`

此目錄包含 Nginx 的存取與錯誤記錄檔。


#### `/usr/local/etc/php/X.X/conf.d`

此目錄包含各種 PHP 設定項目的 `*.ini` 檔案。


#### `/usr/local/etc/php/X.X/php-fpm.d/valet-fpm.conf`

此檔案是 PHP-FPM Pool 設定檔。


#### `~/.composer/vendor/laravel/valet/cli/stubs/secure.valet.conf`

此檔案是用於為你的網站建立 SSL 憑證時的預設 Nginx 設定。


<a name="disk-access"></a>
### 磁碟存取權限

自 macOS 10.14 起，[預設會限制對特定檔案與目錄的存取](https://manuals.info.apple.com/MANUALS/1000/MA1902/en_US/apple-platform-security-guide.pdf)。這些限制包括「桌面」、「文件」與「下載」目錄。此外，網路卷宗與可移除卷宗的存取也受到限制。因此，Valet 建議將你的網站資料夾放置在這些受保護的位置之外。

然而，如果你希望從這些位置之一提供網站服務，你需要授予 Nginx "Full Disk Access"（完全磁碟存取權限）。否則，你可能會遇到伺服器錯誤或 Nginx 的其他不可預測行為，特別是在提供靜態資源時。通常，macOS 會自動提示你授予 Nginx 對這些位置的完全存取權限。或者，你也可以透過 `System Preferences` > `Security & Privacy` > `Privacy` 並選擇 `Full Disk Access` 來手動進行設定。接著，在主視窗窗格中勾選任何 `nginx` 項目。