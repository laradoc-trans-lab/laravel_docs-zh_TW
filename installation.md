# 安裝

- [認識 Laravel](#meet-laravel)
    - [為何選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [安裝 PHP 與 Laravel Installer](#installing-php)
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
- [後續步驟](#next-steps)
    - [Laravel：全端框架](#laravel-the-fullstack-framework)
    - [Laravel：API 後端](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一個具有表達性、優雅語法的 Web 應用程式框架。Web 框架為建立應用程式提供了結構與起點，讓您可以專注於創造令人驚豔的事物，而我們則處理細節。

Laravel 致力於提供卓越的開發者體驗，同時提供強大的功能，例如完善的依賴注入、表達性的資料庫抽象層、佇列與排程任務、單元與整合測試等等。

無論您是 PHP Web 框架的新手，還是擁有多年經驗的開發者，Laravel 都是一個可以與您一同成長的框架。我們將協助您踏出 Web 開發的第一步，或在您將專業知識提升到更高層次時助您一臂之力。我們迫不及待想看到您將會建立什麼。

> [!NOTE]  
> Laravel 新手？請查看 [Laravel Bootcamp](https://bootcamp.laravel.com)，透過實作帶您建立第一個 Laravel 應用程式，親身體驗框架。

<a name="why-laravel"></a>
### 為何選擇 Laravel？

在建立 Web 應用程式時，有各種工具與框架可供選擇。然而，我們相信 Laravel 是建立現代化、全端 Web 應用程式的最佳選擇。

#### 漸進式框架

我們喜歡稱 Laravel 為「漸進式」框架。這表示 Laravel 會與您一同成長。如果您剛踏入 Web 開發領域，Laravel 豐富的說明文件、指南與 [影片教學](https://laracasts.com) 將協助您學習基礎知識，而不會感到不知所措。

如果您是資深開發者，Laravel 則提供強大的工具，例如 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等等。Laravel 經過精心調整，可用於建立專業的 Web 應用程式，並準備好處理企業級工作負載。

#### 可擴展的框架

Laravel 具有令人難以置信的可擴展性。由於 PHP 具備易於擴展的特性，加上 Laravel 內建支援 Redis 等快速、分散式快取系統，使用 Laravel 進行水平擴展輕而易舉。事實上，Laravel 應用程式已能輕鬆擴展以處理每月數億次的請求。

需要極致的擴展性？[Laravel Vapor](https://vapor.laravel.com) 等平台讓您可以在 AWS 最新的無伺服器技術上，以近乎無限的規模執行您的 Laravel 應用程式。

#### 社群框架

Laravel 結合了 PHP 生態系統中最好的套件，以提供最穩健且對開發者最友善的框架。此外，來自世界各地數千名才華洋溢的開發者也為 [框架做出了貢獻](https://github.com/laravel/framework)。誰知道呢，也許您也會成為 Laravel 的貢獻者。

<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式

<a name="installing-php"></a>
### 安裝 PHP 與 Laravel Installer

在建立您的第一個 Laravel 應用程式之前，請確保您的本機電腦已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 與 [Laravel installer](https://github.com/laravel/installer)。此外，您應該安裝 [Node 與 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯應用程式的前端資源。

如果您的本機電腦尚未安裝 PHP 與 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 與 Laravel installer：

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

執行上述其中一個指令後，您應該重新啟動您的終端機工作階段。若要在透過 `php.new` 安裝 PHP、Composer 與 Laravel installer 後更新它們，您可以在終端機中重新執行該指令。

如果您已安裝 PHP 與 Composer，則可以透過 Composer 安裝 Laravel installer：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 若要獲得功能齊全、圖形化的 PHP 安裝與管理體驗，請查看 [Laravel Herd](#local-installation-using-herd)。

<a name="creating-an-application"></a>
### 建立應用程式

安裝 PHP、Composer 與 Laravel installer 後，您就可以建立新的 Laravel 應用程式了。Laravel installer 將提示您選擇偏好的測試框架、資料庫與 starter kit：

```nothing
laravel new example-app
```

應用程式建立後，您可以使用 `dev` Composer script 啟動 Laravel 的本地開發伺服器、佇列 worker 與 Vite 開發伺服器：

```nothing
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，您的應用程式將可在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。接下來，您已準備好 [開始進入 Laravel 生態系統的下一步](#next-steps)。當然，您可能也想 [設定資料庫](#databases-and-migrations)。

> [!NOTE]  
> 如果您希望在開發 Laravel 應用程式時搶先一步，請考慮使用我們的 [starter kits](/docs/{{version}}/starter-kits) 之一。Laravel 的 starter kits 為您的新 Laravel 應用程式提供後端與前端的身份驗證骨架。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有說明文件，因此請隨意瀏覽這些檔案，熟悉可用的選項。

Laravel 開箱即用，幾乎不需要額外的設定。您可以自由地開始開發！但是，您可能希望檢閱 `config/app.php` 檔案及其說明文件。它包含一些選項，例如 `url` 和 `locale`，您可能希望根據您的應用程式進行更改。

<a name="environment-based-configuration"></a>
### 基於環境的設定

由於許多 Laravel 的設定選項值可能會因您的應用程式是在本機電腦上執行還是在生產 Web 伺服器上執行而異，因此許多重要的設定值都是透過應用程式根目錄中的 `.env` 檔案來定義的。

您的 `.env` 檔案不應提交到應用程式的原始碼控制中，因為每個使用您應用程式的開發者/伺服器可能需要不同的環境設定。此外，如果入侵者取得您原始碼控制儲存庫的存取權，這將會是一個安全風險，因為任何敏感憑證都將會暴露。

> [!NOTE]  
> 有關 `.env` 檔案與基於環境的設定的更多資訊，請查看完整的 [設定說明文件](/docs/{{version}}/configuration#environment-configuration)。

<a name="databases-and-migrations"></a>
### 資料庫與遷移

現在您已經建立了 Laravel 應用程式，您可能希望將一些資料儲存在資料庫中。預設情況下，您的應用程式的 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫互動。

在建立應用程式期間，Laravel 為您建立了一個 `database/database.sqlite` 檔案，並執行了必要的遷移以建立應用程式的資料庫表格。

如果您偏好使用其他資料庫驅動程式，例如 MySQL 或 PostgreSQL，您可以更新您的 `.env` 設定檔以使用適當的資料庫。例如，如果您希望使用 MySQL，請像這樣更新您的 `.env` 設定檔的 `DB_*` 變數：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，您將需要建立資料庫並執行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]  
> 如果您在 macOS 或 Windows 上開發，並且需要本機安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans)。

<a name="directory-configuration"></a>
### 目錄設定

Laravel 應始終從為您的 Web 伺服器設定的「Web 目錄」根目錄提供服務。您不應嘗試從「Web 目錄」的子目錄提供 Laravel 應用程式。這樣做可能會暴露應用程式中存在的敏感檔案。

<a name="local-installation-using-herd"></a>
## 使用 Herd 進行本地安裝

[Laravel Herd](https://herd.laravel.com) 是一個適用於 macOS 和 Windows 的極速、原生的 Laravel 和 PHP 開發環境。Herd 包含了開始 Laravel 開發所需的一切，包括 PHP 和 Nginx。

安裝 Herd 後，您就可以開始使用 Laravel 進行開發了。Herd 包含了 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 的命令列工具。

> [!NOTE]  
> [Herd Pro](https://herd.laravel.com/#plans) 透過額外的強大功能增強了 Herd，例如建立和管理本機 MySQL、Postgres 和 Redis 資料庫的能力，以及本機郵件檢視和日誌監控。

<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上開發，可以從 [Herd 網站](https://herd.laravel.com) 下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 版的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「停放」目錄。任何停放目錄中的 Laravel 應用程式都將自動由 Herd 提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停放目錄，您可以使用其目錄名稱在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用 Herd 隨附的 Laravel CLI：

```nothing
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您始終可以透過 Herd 的 UI 管理您的停放目錄和其他 PHP 設定，該 UI 可以從系統匣中的 Herd 選單開啟。

您可以透過查看 [Herd 說明文件](https://herd.laravel.com/docs) 了解更多關於 Herd 的資訊。

<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以從 [Herd 網站](https://herd.laravel.com/windows) 下載 Windows 版的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成上線流程並首次存取 Herd UI。

Herd UI 可透過左鍵點擊 Herd 的系統匣圖示來存取。右鍵點擊則會開啟快速選單，可存取您日常所需的所有工具。

在安裝期間，Herd 會在您的家目錄 `%USERPROFILE%\Herd` 中建立一個「停放」目錄。任何停放目錄中的 Laravel 應用程式都將自動由 Herd 提供服務，您可以使用其目錄名稱在 `.test` 網域上存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用 Herd 隨附的 Laravel CLI。若要開始，請開啟 Powershell 並執行以下指令：

```nothing
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以透過查看 [Windows 版的 Herd 說明文件](https://herd.laravel.com/docs/windows) 了解更多關於 Herd 的資訊。

<a name="docker-installation-using-sail"></a>
## 使用 Sail 進行 Docker 安裝

我們希望無論您偏好的作業系統為何，都能盡可能輕鬆地開始使用 Laravel。因此，有各種選項可以在您的本機電腦上開發和執行 Laravel 應用程式。雖然您可能希望稍後探索這些選項，但 Laravel 提供了 [Sail](/docs/{{version}}/sail)，這是一個內建的解決方案，用於使用 [Docker](https://www.docker.com) 執行您的 Laravel 應用程式。

Docker 是一種用於在小型、輕量級「容器」中執行應用程式和服務的工具，這些容器不會干擾您的本機電腦已安裝的軟體或設定。這表示您無需擔心在本機電腦上設定或建立複雜的開發工具，例如 Web 伺服器和資料庫。若要開始，您只需安裝 [Docker Desktop](https://www.docker.com/products/docker-desktop)。

Laravel Sail 是一個輕量級的命令列介面，用於與 Laravel 的預設 Docker 設定互動。Sail 為使用 PHP、MySQL 和 Redis 建立 Laravel 應用程式提供了一個很好的起點，而無需先前的 Docker 經驗。

> [!NOTE]  
> 已經是 Docker 專家了？別擔心！Sail 的所有內容都可以使用 Laravel 隨附的 `docker-compose.yml` 檔案進行自訂。

<a name="sail-on-macos"></a>
### macOS 上的 Sail

如果您在 Mac 上開發，並且已安裝 [Docker Desktop](https://www.docker.com/products/docker-desktop)，您可以使用簡單的終端機指令來建立新的 Laravel 應用程式。例如，若要在名為「example-app」的目錄中建立新的 Laravel 應用程式，您可以在終端機中執行以下指令：

```shell
curl -s "https://laravel.build/example-app" | bash
```

當然，您可以將此 URL 中的「example-app」更改為您喜歡的任何名稱，但請確保應用程式名稱只包含英數字元、連字號和底線。Laravel 應用程式的目錄將在您執行指令的目錄中建立。

Sail 安裝可能需要幾分鐘，因為 Sail 的應用程式容器會在您的本機電腦上建立。

應用程式建立後，您可以導航到應用程式目錄並啟動 Laravel Sail。Laravel Sail 提供了一個簡單的命令列介面，用於與 Laravel 的預設 Docker 設定互動：

```shell
cd example-app

./vendor/bin/sail up
```

一旦應用程式的 Docker 容器啟動，您應該執行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
./vendor/bin/sail artisan migrate
```

最後，您可以在網頁瀏覽器中透過以下網址存取應用程式：http://localhost。

> [!NOTE]  
> 若要繼續了解更多關於 Laravel Sail 的資訊，請查閱其 [完整說明文件](/docs/{{version}}/sail)。

<a name="sail-on-windows"></a>
### Windows 上的 Sail

在您的 Windows 機器上建立新的 Laravel 應用程式之前，請務必安裝 [Docker Desktop](https://www.docker.com/products/docker-desktop)。接下來，您應該確保已安裝並啟用 Windows Subsystem for Linux 2 (WSL2)。WSL 允許您在 Windows 10 上原生執行 Linux 二進位可執行檔。有關如何安裝和啟用 WSL2 的資訊可以在 Microsoft 的 [開發者環境說明文件](https://docs.microsoft.com/en-us/windows/wsl/install-win10) 中找到。

> [!NOTE]  
> 安裝並啟用 WSL2 後，您應該確保 Docker Desktop 已 [設定為使用 WSL2 後端](https://docs.docker.com/docker-for-windows/wsl/)。

接下來，您已準備好建立您的第一個 Laravel 應用程式。啟動 [Windows Terminal](https://www.microsoft.com/en-us/p/windows-terminal/9n0dx20hk701?rtc=1&activetab=pivot:overviewtab) 並為您的 WSL2 Linux 作業系統開始新的終端機工作階段。接下來，您可以使用簡單的終端機指令來建立新的 Laravel 應用程式。例如，若要在名為「example-app」的目錄中建立新的 Laravel 應用程式，您可以在終端機中執行以下指令：

```shell
curl -s https://laravel.build/example-app | bash
```

當然，您可以將此 URL 中的「example-app」更改為您喜歡的任何名稱，但請確保應用程式名稱只包含英數字元、連字號和底線。Laravel 應用程式的目錄將在您執行指令的目錄中建立。

Sail 安裝可能需要幾分鐘，因為 Sail 的應用程式容器會在您的本機電腦上建立。

應用程式建立後，您可以導航到應用程式目錄並啟動 Laravel Sail。Laravel Sail 提供了一個簡單的命令列介面，用於與 Laravel 的預設 Docker 設定互動：

```shell
cd example-app

./vendor/bin/sail up
```

一旦應用程式的 Docker 容器啟動，您應該執行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
./vendor/bin/sail artisan migrate
```

最後，您可以在網頁瀏覽器中透過以下網址存取應用程式：http://localhost。

> [!NOTE]  
> 若要繼續了解更多關於 Laravel Sail 的資訊，請查閱其 [完整說明文件](/docs/{{version}}/sail)。

#### 在 WSL2 中開發

當然，您將需要能夠修改在您的 WSL2 安裝中建立的 Laravel 應用程式檔案。為此，我們建議使用 Microsoft 的 [Visual Studio Code](https://code.visualstudio.com) 編輯器及其 [Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack) 第一方擴充功能。

安裝這些工具後，您可以透過使用 Windows Terminal 從應用程式的根目錄執行 `code .` 指令來開啟任何 Laravel 應用程式。

<a name="sail-on-linux"></a>
### Linux 上的 Sail

如果您在 Linux 上開發，並且已安裝 [Docker Compose](https://docs.docker.com/compose/install/)，您可以使用簡單的終端機指令來建立新的 Laravel 應用程式。

首先，如果您正在使用 Docker Desktop for Linux，您應該執行以下指令。如果您沒有使用 Docker Desktop for Linux，您可以跳過此步驟：

```shell
docker context use default
```

然後，若要在名為「example-app」的目錄中建立新的 Laravel 應用程式，您可以在終端機中執行以下指令：

```shell
curl -s https://laravel.build/example-app | bash
```

當然，您可以將此 URL 中的「example-app」更改為您喜歡的任何名稱，但請確保應用程式名稱只包含英數字元、連字號和底線。Laravel 應用程式的目錄將在您執行指令的目錄中建立。

Sail 安裝可能需要幾分鐘，因為 Sail 的應用程式容器會在您的本機電腦上建立。

應用程式建立後，您可以導航到應用程式目錄並啟動 Laravel Sail。Laravel Sail 提供了一個簡單的命令列介面，用於與 Laravel 的預設 Docker 設定互動：

```shell
cd example-app

./vendor/bin/sail up
```

一旦應用程式的 Docker 容器啟動，您應該執行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
./vendor/bin/sail artisan migrate
```

最後，您可以在網頁瀏覽器中透過以下網址存取應用程式：http://localhost。

> [!NOTE]  
> 若要繼續了解更多關於 Laravel Sail 的資訊，請查閱其 [完整說明文件](/docs/{{version}}/sail)。

<a name="choosing-your-sail-services"></a>
### 選擇您的 Sail 服務

透過 Sail 建立新的 Laravel 應用程式時，您可以使用 `with` 查詢字串變數來選擇應在您的新應用程式的 `docker-compose.yml` 檔案中設定哪些服務。可用的服務包括 `mysql`、`pgsql`、`mariadb`、`redis`、`valkey`、`memcached`、`meilisearch`、`typesense`、`minio`、`selenium` 和 `mailpit`：

```shell
curl -s "https://laravel.build/example-app?with=mysql,redis" | bash
```

如果您未指定要設定哪些服務，則將設定 `mysql`、`redis`、`meilisearch`、`mailpit` 和 `selenium` 的預設堆疊。

您可以透過將 `devcontainer` 參數新增至 URL，指示 Sail 安裝預設的 [Devcontainer](/docs/{{version}}/sail#using-devcontainers)：

```shell
curl -s "https://laravel.build/example-app?with=mysql,redis&devcontainer" | bash
```

<a name="ide-support"></a>
## IDE 支援

您可以自由使用任何您喜歡的程式碼編輯器來開發 Laravel 應用程式；然而，[PhpStorm](https://www.jetbrains.com/phpstorm/laravel/) 為 Laravel 及其生態系統提供了廣泛的支援，包括 [Laravel Pint](https://www.jetbrains.com/help/phpstorm/using-laravel-pint.html)。

此外，社群維護的 [Laravel Idea](https://laravel-idea.com/) PhpStorm 外掛程式提供了各種有用的 IDE 增強功能，包括程式碼生成、Eloquent 語法補全、驗證規則補全等等。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個強大的工具，彌合了 AI 程式碼代理與 Laravel 應用程式之間的差距。Boost 為 AI 代理提供了 Laravel 特定的上下文、工具和指南，因此它們可以生成更準確、版本特定的程式碼，並遵循 Laravel 慣例。

當您在 Laravel 應用程式中安裝 Boost 時，AI 代理可以存取超過 15 種專門工具，包括了解您正在使用哪些套件、查詢您的資料庫、搜尋 Laravel 說明文件、讀取瀏覽器日誌、生成測試以及透過 Tinker 執行程式碼的能力。

此外，Boost 還為 AI 代理提供了超過 17,000 條向量化的 Laravel 生態系統說明文件，這些文件專門針對您已安裝的套件版本。這表示代理可以提供針對您專案所使用的確切版本的指導。

Boost 還包括 Laravel 維護的 AI 指南，這些指南會引導代理遵循框架慣例、編寫適當的測試，並避免在生成 Laravel 程式碼時常見的陷阱。

<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在執行 PHP 8.1 或更高版本的 Laravel 10、11 和 12 應用程式中。若要開始，請將 Boost 安裝為開發依賴項：

```shell
composer require laravel/boost --dev
```

安裝後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式將自動偵測您的 IDE 和 AI 代理，讓您可以選擇加入對您的專案有意義的功能。Boost 尊重現有的專案慣例，並且預設不強制執行主觀的樣式規則。

> [!NOTE]
> 若要了解更多關於 Boost 的資訊，請查看 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。

<a name="next-steps"></a>
## 後續步驟

現在您已經建立了 Laravel 應用程式，您可能想知道接下來要學習什麼。首先，我們強烈建議您透過閱讀以下說明文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [Service Container](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您希望如何使用 Laravel 也將決定您旅程的下一步。使用 Laravel 的方式有很多種，我們將在下面探討框架的兩種主要用例。

> [!NOTE]  
> Laravel 新手？請查看 [Laravel Bootcamp](https://bootcamp.laravel.com) 透過實作帶您建立第一個 Laravel 應用程式，親身體驗框架。

<a name="laravel-the-fullstack-framework"></a>
### Laravel：全端框架

Laravel 可以作為一個全端框架。所謂「全端」框架，是指您將使用 Laravel 來路由應用程式的請求，並透過 [Blade 模板](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染您的前端。這是使用 Laravel 框架最常見的方式，也是我們認為使用 Laravel 最有效率的方式。

如果您打算這樣使用 Laravel，您可能需要查看我們關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的說明文件。此外，您可能對學習像 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 這樣的社群套件感興趣。這些套件允許您將 Laravel 作為全端框架使用，同時享受單頁 JavaScript 應用程式提供的許多 UI 優勢。

如果您將 Laravel 作為全端框架使用，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 編譯應用程式的 CSS 和 JavaScript。

> [!NOTE]  
> 如果您想在建立應用程式時搶先一步，請查看我們官方的 [應用程式 starter kits](/docs/{{version}}/starter-kits) 之一。

<a name="laravel-the-api-backend"></a>
### Laravel：API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可以使用 Laravel 作為 [Next.js](https://nextjs.org) 應用程式的 API 後端。在此情境下，您可以使用 Laravel 為您的應用程式提供 [身份驗證](/docs/{{version}}/sanctum) 和資料儲存/檢索，同時也利用 Laravel 強大的服務，例如佇列、電子郵件、通知等等。

如果您打算這樣使用 Laravel，您可能需要查看我們關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的說明文件。

> [!NOTE]  
> 需要快速建立您的 Laravel 後端和 Next.js 前端嗎？Laravel Breeze 提供了 [API 堆疊](/docs/{{version}}/starter-kits#breeze-and-next) 以及 [Next.js 前端實作](https://github.com/laravel/breeze-next)，讓您可以在幾分鐘內開始。
