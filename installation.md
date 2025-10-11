# 安裝

- [認識 Laravel](#meet-laravel)
    - [為何選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [安裝 PHP 與 Laravel 安裝器](#installing-php)
    - [建立應用程式](#creating-an-application)
- [初始設定](#initial-configuration)
    - [基於環境的設定](#environment-based-configuration)
    - [資料庫與遷移](#databases-and-migrations)
    - [目錄設定](#directory-configuration)
- [使用 Herd 進行本地安裝](#local-installation-using-herd)
    - [macOS 上的 Herd](#herd-on-macos)
    - [Windows 上的 Herd](#herd-on-windows)
- [使用 Sail 進行 Docker 安裝](#docker-installation-using-sail)
    - [macOS 上的 Sail](#sail-on-macos)
    - [Windows 上的 Sail](#sail-on-windows)
    - [Linux 上的 Sail](#sail-on-linux)
    - [選擇您的 Sail 服務](#choosing-your-sail-services)
- [IDE 支援](#ide-support)
- [Laravel 與 AI](#laravel-and-ai)
    - [安裝 Laravel Boost](#installing-laravel-boost)
- [下一步](#next-steps)
    - [作為全端框架的 Laravel](#laravel-the-fullstack-framework)
    - [作為 API 後端的 Laravel](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一個具有表達力、優雅語法的 Web 應用程式框架。Web 框架提供建立應用程式的結構和起點，讓您專注於創造令人驚豔的事物，而我們則負責處理細節。

Laravel 致力於提供卓越的開發者體驗，同時提供強大的功能，例如完善的依賴注入、富有表達力的資料庫抽象層、佇列與排程作業、單元與整合測試等等。

無論您是 PHP Web 框架的新手，或是擁有多年經驗，Laravel 都是一個可以與您共同成長的框架。我們將幫助您踏出成為 Web 開發者的第一步，或是在您將專業知識提升到更高層次時助您一臂之力。我們迫不及待地想看到您所創造的一切。

> [!NOTE]  
> Laravel 新手？請查看 [Laravel Bootcamp](https://bootcamp.laravel.com)，透過實作方式導覽框架，我們將引導您建立您的第一個 Laravel 應用程式。

<a name="why-laravel"></a>
### 為何選擇 Laravel？

在建立 Web 應用程式時，您可以使用多種工具和框架。然而，我們相信 Laravel 是建立現代化、全端 Web 應用程式的最佳選擇。

#### 漸進式框架

我們喜歡將 Laravel 稱為「漸進式」框架。我們的意思是，Laravel 會與您一同成長。如果您剛踏入 Web 開發領域，Laravel 龐大的文件庫、指南和 [影片教學](https://laracasts.com) 將幫助您輕鬆入門，而不會感到不知所措。

如果您是資深開發人員，Laravel 提供您強大的工具，用於 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等等。Laravel 經過精心調整，適用於建立專業的 Web 應用程式，並能處理企業級工作負載。

#### 可擴展框架

Laravel 具有令人難以置信的可擴展性。由於 PHP 友善於擴展的特性，以及 Laravel 內建支援 Redis 等快速分散式快取系統，使用 Laravel 進行水平擴展輕而易舉。事實上，Laravel 應用程式已能輕鬆擴展，每月處理數億個請求。

需要極致擴展？像 [Laravel Vapor](https://vapor.laravel.com) 這樣的平台，讓您可以在 AWS 最新的無伺服器技術上，以近乎無限的規模運行您的 Laravel 應用程式。

#### 社群框架

Laravel 結合 PHP 生態系統中最佳的套件，提供最穩健且開發者友善的框架。此外，全球數千名才華洋溢的開發者已 [對此框架做出貢獻](https://github.com/laravel/framework)。說不定，您也將成為 Laravel 的貢獻者。

<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式

<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝器

在建立您的第一個 Laravel 應用程式之前，請確保您的本地機器已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 和 [Laravel 安裝器](https://github.com/laravel/installer)。此外，您應該安裝 [Node 和 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯您應用程式的前端資產。

如果您的本地機器尚未安裝 PHP 和 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝器：

```shell tab=macOS
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.4)"
```

```shell tab=Windows PowerShell
# Run as administrator...
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.4'))
```

```shell tab=Linux
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.4)"
```

執行上述任一指令後，您應該重新啟動終端機連線。若要在透過 `php.new` 安裝後更新 PHP、Composer 和 Laravel 安裝器，您可以在終端機中重新執行該指令。

如果您已經安裝 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝器：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 如需功能齊全的圖形化 PHP 安裝和管理體驗，請參閱 [Laravel Herd](#local-installation-using-herd)。

<a name="creating-an-application"></a>
### 建立應用程式

在您安裝了 PHP、Composer 和 Laravel 安裝器之後，您已準備好建立一個新的 Laravel 應用程式。Laravel 安裝器將提示您選擇偏好的測試框架、資料庫和入門套件：

```nothing
laravel new example-app
```

應用程式建立後，您可以使用 `dev` Composer 腳本來啟動 Laravel 的本地開發伺服器、佇列工作者和 Vite 開發伺服器：

```nothing
cd example-app
npm install && npm run build
composer run dev
```

一旦您啟動開發伺服器，您的應用程式將可在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。接下來，您已準備好 [邁向 Laravel 生態系統的下一步](#next-steps)。當然，您可能也想 [設定資料庫](#databases-and-migrations)。

> [!NOTE]  
> 如果您想在開發 Laravel 應用程式時搶得先機，可以考慮使用我們的其中一個 [入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件為您的新 Laravel 應用程式提供後端和前端身份驗證骨架。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細說明，您可以隨意查閱這些檔案，熟悉可用的選項。

Laravel 開箱即用，幾乎不需要額外設定。您可以自由開始開發！然而，您可能希望查閱 `config/app.php` 檔案及其文件。它包含 `url` 和 `locale` 等多個選項，您可以根據您的應用程式進行修改。

<a name="environment-based-configuration"></a>
### 基於環境的設定

由於許多 Laravel 的設定選項值可能因應用程式是在本地機器還是生產環境網頁伺服器上執行而異，因此許多重要的設定值是透過應用程式根目錄下的 `.env` 檔案來定義的。

您的 `.env` 檔案不應提交到應用程式的原始碼控制中，因為每個使用您應用程式的開發者/伺服器可能需要不同的環境設定。此外，如果入侵者取得您原始碼儲存庫的存取權，這將會是個安全風險，因為任何敏感的憑證都將會暴露。

> [!NOTE]  
> 有關 `.env` 檔案和基於環境的設定的更多資訊，請查閱完整的[設定文件](/docs/{{version}}/configuration#environment-configuration)。

<a name="databases-and-migrations"></a>
### 資料庫與遷移

既然您已經建立 Laravel 應用程式，您可能想要將一些資料儲存在資料庫中。預設情況下，您的應用程式的 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫互動。

在建立應用程式期間，Laravel 為您建立了一個 `database/database.sqlite` 檔案，並執行了必要的遷移來建立應用程式的資料庫表格。

如果您偏好使用其他資料庫驅動程式，例如 MySQL 或 PostgreSQL，您可以更新您的 `.env` 設定檔以使用適當的資料庫。例如，如果您希望使用 MySQL，請像這樣更新您的 `.env` 設定檔中的 `DB_*` 變數：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，您將需要建立資料庫並執行應用程式的[資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]  
> 如果您在 macOS 或 Windows 上進行開發，並且需要本地安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans)。

<a name="directory-configuration"></a>
### 目錄設定

Laravel 應用程式應始終從您的網頁伺服器所設定的「網頁目錄」根目錄提供服務。您不應嘗試從「網頁目錄」的子目錄中提供 Laravel 應用程式。這樣做可能會暴露應用程式中存在的敏感檔案。

<a name="local-installation-using-herd"></a>
## 使用 Herd 進行本地安裝

[Laravel Herd](https://herd.laravel.com) 是一個用於 macOS 和 Windows 的極速原生 Laravel 與 PHP 開發環境。Herd 包含您開始 Laravel 開發所需的一切，包括 PHP 和 Nginx。

安裝 Herd 後，您就可以開始使用 Laravel 進行開發了。Herd 包含 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 的命令列工具。

> [!NOTE]  
> [Herd Pro](https://herd.laravel.com/#plans) 透過額外的強大功能來增強 Herd，例如建立和管理本地 MySQL、Postgres 和 Redis 資料庫的能力，以及本地郵件查看和日誌監控。

<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上進行開發，可以從 [Herd 網站](https://herd.laravel.com)下載 Herd 安裝程式。該安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

適用於 macOS 的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「停放」目錄。任何在停放目錄中的 Laravel 應用程式都將由 Herd 自動提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停放目錄，您可以使用其目錄名稱，在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用隨 Herd 捆綁的 Laravel CLI：

```nothing
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您也可以隨時透過 Herd 的使用者介面管理您的停放目錄和其他 PHP 設定，該介面可以從系統匣中的 Herd 選單開啟。

您可以透過查閱 [Herd 文件](https://herd.laravel.com/docs)了解更多關於 Herd 的資訊。

<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以在 [Herd 網站](https://herd.laravel.com/windows)上下載適用於 Windows 的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成入門流程並首次存取 Herd 使用者介面。

透過左鍵點擊 Herd 的系統匣圖示即可存取 Herd 使用者介面。右鍵點擊則會開啟快速選單，可存取您日常所需的所有工具。

在安裝期間，Herd 會在您的家目錄 `%USERPROFILE%\Herd` 中建立一個「停放」目錄。任何在停放目錄中的 Laravel 應用程式都將由 Herd 自動提供服務，您可以使用其目錄名稱，在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用隨 Herd 捆綁的 Laravel CLI。要開始使用，請開啟 Powershell 並執行以下命令：

```nothing
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以透過查閱[適用於 Windows 的 Herd 文件](https://herd.laravel.com/docs/windows)了解更多關於 Herd 的資訊。

<a name="docker-installation-using-sail"></a>
## 使用 Sail 進行 Docker 安裝

我們希望無論您偏好哪種作業系統，都能盡可能輕鬆地開始使用 Laravel。因此，您可以在本機上開發和運行 Laravel 應用程式的方式有很多種。雖然您可能希望日後再探索這些選項，但 Laravel 提供了 [Sail](/docs/{{version}}/sail)，這是一個用於使用 [Docker](https://www.docker.com) 運行 Laravel 應用程式的內建解決方案。

Docker 是一個用於在小型、輕量級「容器」中運行應用程式和服務的工具，這些容器不會干擾您本機上已安裝的軟體或配置。這意味著您無需擔心在本機上配置或設定複雜的開發工具，例如網頁伺服器和資料庫。要開始使用，您只需安裝 [Docker Desktop](https://www.docker.com/products/docker-desktop)。

Laravel Sail 是一個輕量級命令列介面，用於與 Laravel 預設的 Docker 配置互動。Sail 提供了一個絕佳的起點，讓您無需具備 Docker 經驗，即可使用 PHP、MySQL 和 Redis 來建置 Laravel 應用程式。

> [!NOTE]  
> 已經是 Docker 專家了？別擔心！關於 Sail 的一切都可以透過 Laravel 隨附的 `docker-compose.yml` 檔案進行客製化。

<a name="sail-on-macos"></a>
### macOS 上的 Sail

如果您在 Mac 上開發，並且已經安裝了 [Docker Desktop](https://www.docker.com/products/docker-desktop)，您可以使用一個簡單的終端機指令來建立新的 Laravel 應用程式。例如，要在一個名為「example-app」的目錄中建立新的 Laravel 應用程式，您可以在終端機中運行以下指令：

```shell
curl -s "https://laravel.build/example-app" | bash
```

當然，您可以將此 URL 中的「example-app」更改為您喜歡的任何名稱——只需確保應用程式名稱只包含英數字元、破折號和底線即可。Laravel 應用程式的目錄將在您執行指令的目錄中建立。

Sail 安裝可能需要幾分鐘的時間，因為 Sail 的應用程式容器會在您的本機上建置。

應用程式建立完成後，您可以導航到應用程式目錄並啟動 Laravel Sail。Laravel Sail 提供一個簡單的命令列介面，用於與 Laravel 預設的 Docker 配置互動：

```shell
cd example-app

./vendor/bin/sail up
```

一旦應用程式的 Docker 容器啟動，您應該運行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
./vendor/bin/sail artisan migrate
```

最後，您可以在網頁瀏覽器中存取應用程式：http://localhost。

> [!NOTE]  
> 要繼續了解更多關於 Laravel Sail 的資訊，請查閱其[完整文件](/docs/{{version}}/sail)。

<a name="sail-on-windows"></a>
### Windows 上的 Sail

在您的 Windows 機器上建立新的 Laravel 應用程式之前，請務必安裝 [Docker Desktop](https://www.docker.com/products/docker-desktop)。接著，您應該確保 Windows Subsystem for Linux 2 (WSL2) 已安裝並啟用。WSL 允許您在 Windows 10 上原生運行 Linux 二進位可執行檔。有關如何安裝和啟用 WSL2 的資訊，可在 Microsoft 的[開發者環境文件](https://docs.microsoft.com/en-us/windows/wsl/install-win10)中找到。

> [!NOTE]  
> 安裝並啟用 WSL2 後，您應該確保 Docker Desktop 已[配置為使用 WSL2 後端](https://docs.docker.com/docker-for-windows/wsl/)。

接下來，您就可以建立您的第一個 Laravel 應用程式了。啟動 [Windows Terminal](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701?rtc=1&activetab=pivot:overviewtab) 並為您的 WSL2 Linux 作業系統開始一個新的終端機會話。接著，您可以使用一個簡單的終端機指令來建立新的 Laravel 應用程式。例如，要在一個名為「example-app」的目錄中建立新的 Laravel 應用程式，您可以在終端機中運行以下指令：

```shell
curl -s https://laravel.build/example-app | bash
```

當然，您可以將此 URL 中的「example-app」更改為您喜歡的任何名稱——只需確保應用程式名稱只包含英數字元、破折號和底線即可。Laravel 應用程式的目錄將在您執行指令的目錄中建立。

Sail 安裝可能需要幾分鐘的時間，因為 Sail 的應用程式容器會在您的本機上建置。

應用程式建立完成後，您可以導航到應用程式目錄並啟動 Laravel Sail。Laravel Sail 提供一個簡單的命令列介面，用於與 Laravel 預設的 Docker 配置互動：

```shell
cd example-app

./vendor/bin/sail up
```

一旦應用程式的 Docker 容器啟動，您應該運行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
./vendor/bin/sail artisan migrate
```

最後，您可以在網頁瀏覽器中存取應用程式：http://localhost。

> [!NOTE]  
> 要繼續了解更多關於 Laravel Sail 的資訊，請查閱其[完整文件](/docs/{{version}}/sail)。

#### 在 WSL2 中開發

當然，您需要能夠修改在您的 WSL2 安裝中建立的 Laravel 應用程式檔案。為此，我們建議使用 Microsoft 的 [Visual Studio Code](https://code.visualstudio.com) 編輯器及其第一方擴充功能 [Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)。

安裝這些工具後，您可以使用 Windows Terminal 在應用程式的根目錄中執行 `code .` 指令來開啟任何 Laravel 應用程式。

<a name="sail-on-linux"></a>
### Linux 上的 Sail

如果您在 Linux 上開發，並且已經安裝了 [Docker Compose](https://docs.docker.com/compose/install/)，您可以使用一個簡單的終端機指令來建立新的 Laravel 應用程式。

首先，如果您正在使用 Docker Desktop for Linux，您應該執行以下指令。如果您沒有使用 Docker Desktop for Linux，則可以跳過此步驟：

```shell
docker context use default
```

然後，要在一個名為「example-app」的目錄中建立新的 Laravel 應用程式，您可以在終端機中運行以下指令：

```shell
curl -s https://laravel.build/example-app | bash
```

當然，您可以將此 URL 中的「example-app」更改為您喜歡的任何名稱——只需確保應用程式名稱只包含英數字元、破折號和底線即可。Laravel 應用程式的目錄將在您執行指令的目錄中建立。

Sail 安裝可能需要幾分鐘的時間，因為 Sail 的應用程式容器會在您的本機上建置。

應用程式建立完成後，您可以導航到應用程式目錄並啟動 Laravel Sail。Laravel Sail 提供一個簡單的命令列介面，用於與 Laravel 預設的 Docker 配置互動：

```shell
cd example-app

./vendor/bin/sail up
```

一旦應用程式的 Docker 容器啟動，您應該運行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
./vendor/bin/sail artisan migrate
```

最後，您可以在網頁瀏覽器中存取應用程式：http://localhost。

> [!NOTE]  
> 要繼續了解更多關於 Laravel Sail 的資訊，請查閱其[完整文件](/docs/{{version}}/sail)。

<a name="choosing-your-sail-services"></a>
### 選擇您的 Sail 服務

透過 Sail 建立新的 Laravel 應用程式時，您可以使用 `with` 查詢字串變數來選擇應設定哪些服務在新應用程式的 `docker-compose.yml` 檔案中。可用的服務包括 `mysql`、`pgsql`、`mariadb`、`redis`、`valkey`、`memcached`、`meilisearch`、`typesense`、`minio`、`selenium` 和 `mailpit`：

```shell
curl -s "https://laravel.build/example-app?with=mysql,redis" | bash
```

如果您沒有指定想要設定哪些服務，則會設定預設的 `mysql`、`redis`、`meilisearch`、`mailpit` 和 `selenium` 服務堆疊。

您可以透過將 `devcontainer` 參數新增至 URL 來指示 Sail 安裝預設的 [Devcontainer](/docs/{{version}}/sail#using-devcontainers)：

```shell
curl -s "https://laravel.build/example-app?with=mysql,redis&devcontainer" | bash
```

<a name="ide-support"></a>
## IDE 支援

在開發 Laravel 應用程式時，您可以自由使用任何您喜歡的程式碼編輯器；然而，[PhpStorm](https://www.jetbrains.com/phpstorm/laravel/) 為 Laravel 及其生態系提供了廣泛的支援，其中包括 [Laravel Pint](https://www.jetbrains.com/help/phpstorm/using-laravel-pint.html)。

此外，社群維護的 [Laravel Idea](https://laravel-idea.com/) PhpStorm 外掛程式提供了多種實用的 IDE 增強功能，包括程式碼生成、Eloquent 語法自動完成、驗證規則自動完成等等。


<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個強大的工具，彌合了 AI 編碼代理與 Laravel 應用程式之間的鴻溝。Boost 為 AI 代理提供了 Laravel 專屬的上下文、工具和指南，使它們能夠生成更準確、特定於版本的程式碼，並遵循 Laravel 慣例。

當您在 Laravel 應用程式中安裝 Boost 時，AI 代理可以存取超過 15 種專用工具，包括了解您正在使用哪些套件、查詢您的資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、生成測試，以及透過 Tinker 執行程式碼。

此外，Boost 還為 AI 代理提供了超過 17,000 份向量化的 Laravel 生態系文件，這些文件針對您已安裝的套件版本。這意味著代理可以提供針對您專案所使用的確切版本量身定制的指導。

Boost 還包括由 Laravel 維護的 AI 指南，這些指南會引導代理遵循框架慣例、編寫適當的測試，並避免在生成 Laravel 程式碼時的常見陷阱。


<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在運行 PHP 8.1 或更高版本的 Laravel 10、11 和 12 應用程式中。要開始使用，請將 Boost 作為開發依賴項安裝：

```shell
composer require laravel/boost --dev
```

安裝完成後，運行互動式安裝器：

```shell
php artisan boost:install
```

安裝器將自動偵測您的 IDE 和 AI 代理，讓您可以選擇啟用您專案所需的功能。Boost 尊重現有的專案慣例，並且預設不會強制執行有偏見的風格規則。

> [!NOTE]
> 要了解更多關於 Boost 的資訊，請查閱 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。


<a name="next-steps"></a>
## 下一步

現在您已經建立了您的 Laravel 應用程式，您可能想知道接下來要學習什麼。首先，我們強烈建議您透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [服務容器](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您打算如何使用 Laravel 也將決定您旅程中的下一步。使用 Laravel 的方式有多種，我們將在下面探討該框架的兩個主要使用案例。

> [!NOTE]  
> Laravel 新手？請參考 [Laravel Bootcamp](https://bootcamp.laravel.com)，我們將引導您建立第一個 Laravel 應用程式，讓您親身體驗這個框架。


<a name="laravel-the-fullstack-framework"></a>
### 作為全端框架的 Laravel

Laravel 可以作為一個全端框架。所謂「全端」框架，是指您將使用 Laravel 來路由對應用程式的請求，並透過 [Blade 範本](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染您的前端。這是使用 Laravel 框架最常見，也是我們認為最具生產力的方式。

如果您打算這樣使用 Laravel，您可能會想查閱我們關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會對了解像 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 這樣的社群套件感興趣。這些套件讓您能夠將 Laravel 作為全端框架使用，同時享受單頁 JavaScript 應用程式所提供的許多 UI 優勢。

如果您正在將 Laravel 作為全端框架使用，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 編譯應用程式的 CSS 和 JavaScript。

> [!NOTE]  
> 如果您想在建立應用程式時搶得先機，請參考我們的官方 [應用程式入門套件](/docs/{{version}}/starter-kits) 之一。


<a name="laravel-the-api-backend"></a>
### 作為 API 後端的 Laravel

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可能會將 Laravel 作為 [Next.js](https://nextjs.org) 應用程式的 API 後端。在此情境下，您可以使用 Laravel 為您的應用程式提供 [認證](/docs/{{version}}/sanctum) 和資料儲存/檢索，同時也利用 Laravel 強大的服務，例如佇列 (queues)、電子郵件、通知等等。

如果您打算這樣使用 Laravel，您可能會想查閱我們關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。

> [!NOTE]  
> 需要搶先建構您的 Laravel 後端和 Next.js 前端嗎？Laravel Breeze 提供了 [API 堆疊](/docs/{{version}}/starter-kits#breeze-and-next) 以及 [Next.js 前端實作](https://github.com/laravel/breeze-next)，讓您可以在幾分鐘內開始。