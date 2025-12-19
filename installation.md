# 安裝

- [認識 Laravel](#meet-laravel)
    - [為什麼選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [安裝 PHP 與 Laravel 安裝器](#installing-php)
    - [建立應用程式](#creating-an-application)
- [初始設定](#initial-configuration)
    - [基於環境的設定](#environment-based-configuration)
    - [資料庫與遷移](#databases-and-migrations)
    - [目錄設定](#directory-configuration)
- [使用 Herd 安裝](#installation-using-herd)
    - [macOS 上的 Herd](#herd-on-macos)
    - [Windows 上的 Herd](#herd-on-windows)
- [IDE 支援](#ide-support)
- [Laravel 與 AI](#laravel-and-ai)
    - [安裝 Laravel Boost](#installing-laravel-boost)
- [接下來的步驟](#next-steps)
    - [Laravel：全端框架](#laravel-the-fullstack-framework)
    - [Laravel：API 後端](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一個具有表達力、語法優雅的網頁應用程式框架。網頁框架為建立應用程式提供結構和起點，讓您可以專注於創造令人驚豔的事物，而我們則負責處理細節。

Laravel 致力於提供卓越的開發者體驗，同時提供強大的功能，例如完善的依賴注入、富有表達力的資料庫抽象層、佇列與排程任務、單元與整合測試等等。

無論您是剛接觸 PHP 網頁框架的新手，還是擁有多年經驗的開發者，Laravel 都是一個可以與您一同成長的框架。我們將協助您踏出網頁開發的第一步，或在您提升專業知識的道路上助您一臂之力。我們迫不及待想看到您的創作。

<a name="why-laravel"></a>
### 為什麼選擇 Laravel？

在建立網頁應用程式時，您可以使用多種工具和框架。然而，我們相信 Laravel 是建構現代全端網頁應用程式的最佳選擇。

#### 漸進式框架

我們喜歡稱 Laravel 為「漸進式」框架。這意味著 Laravel 會與您一同成長。如果您剛踏入網頁開發領域，Laravel 龐大的文件庫、指南和 [影片教學課程](https://laracasts.com) 將幫助您輕鬆入門，而不會感到不知所措。

如果您是資深開發者，Laravel 則提供強大的工具，用於 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等。Laravel 經過精心調整，適用於建立專業的網頁應用程式，並準備好處理企業級工作負載。

#### 可擴展的框架

Laravel 具有令人難以置信的可擴展性。得益於 PHP 對於擴展的友好特性，以及 Laravel 內建對 Redis 等快速分散式快取系統的支援，使用 Laravel 進行橫向擴展輕而易舉。事實上，Laravel 應用程式已能輕鬆擴展，每月處理數億次請求。

需要極致擴展嗎？像 [Laravel Cloud](https://cloud.laravel.com) 這樣的平台，讓您的 Laravel 應用程式能以近乎無限的規模運行。

#### 社群框架

Laravel 結合了 PHP 生態系統中最佳的套件，提供最穩健且開發者友善的框架。此外，全球成千上萬的優秀開發者也 [為此框架貢獻](https://github.com/laravel/framework) 己力。說不定您將來也會成為 Laravel 的貢獻者。

<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式

<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝器

在建立您的第一個 Laravel 應用程式之前，請確保您的本機已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 和 [Laravel 安裝器](https://github.com/laravel/installer)。此外，您應該安裝 [Node 和 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯您應用程式的前端資產。

如果您的本機沒有安裝 PHP 和 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝器：

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

執行上述任一指令後，您應該重新啟動您的終端機工作階段。若要在透過 `php.new` 安裝 PHP、Composer 和 Laravel 安裝器後進行更新，您可以在終端機中重新執行該指令。

如果您已經安裝 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝器：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 如需功能齊全、圖形化的 PHP 安裝與管理體驗，請查看 [Laravel Herd](#installation-using-herd)。

<a name="creating-an-application"></a>
### 建立應用程式

在您安裝 PHP、Composer 和 Laravel 安裝器之後，您就可以建立新的 Laravel 應用程式了。Laravel 安裝器會提示您選擇偏好的測試框架、資料庫和入門套件：

```shell
laravel new example-app
```

應用程式建立後，您可以使用 `dev` Composer 指令稿啟動 Laravel 的本機開發伺服器、佇列工作者和 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

一旦您啟動開發伺服器，您的應用程式將可在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。接下來，您已準備好 [開始深入 Laravel 生態系統](#next-steps)。當然，您可能也想 [設定資料庫](#databases-and-migrations)。

> [!NOTE]
> 如果您希望在開發 Laravel 應用程式時搶得先機，可以考慮使用我們的 [入門套件](/docs/{{version}}/starter-kits) 之一。Laravel 的入門套件為您的新 Laravel 應用程式提供後端和前端身份驗證骨架。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細說明，您可以隨意查閱這些檔案，熟悉可用的選項。

Laravel 開箱即用，幾乎不需要額外設定。您可以自由地開始開發！不過，您可能會希望檢閱 `config/app.php` 檔案及其文件。它包含多個選項，例如 `url` 和 `locale`，您可以根據應用程式的需求進行修改。

<a name="environment-based-configuration"></a>
### 基於環境的設定

由於 Laravel 的許多設定選項值可能會因應用程式是在本機還是生產網路伺服器上執行而異，因此許多重要的設定值都是透過應用程式根目錄下的 `.env` 檔案定義的。

您的 `.env` 檔案不應提交到應用程式的原始碼管理中，因為每個開發人員或伺服器使用您的應用程式時可能需要不同的環境設定。此外，若入侵者取得您原始碼儲存庫的存取權，這將構成安全風險，因為所有敏感憑證都將暴露無遺。

> [!NOTE]
> 如需更多關於 `.env` 檔案和基於環境的設定資訊，請查閱完整的 [設定文件](/docs/{{version}}/configuration#environment-configuration)。

<a name="databases-and-migrations"></a>
### 資料庫與遷移

建立 Laravel 應用程式後，您可能希望將一些資料儲存到資料庫中。預設情況下，您的應用程式 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫互動。

在應用程式建立期間，Laravel 會為您建立一個 `database/database.sqlite` 檔案，並執行必要的遷移 (migrations) 來建立應用程式的資料庫資料表。

如果您偏好使用其他資料庫驅動程式，例如 MySQL 或 PostgreSQL，您可以更新您的 `.env` 設定檔以使用適當的資料庫。例如，如果您希望使用 MySQL，請更新 `.env` 設定檔中的 `DB_*` 變數如下：

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
> 如果您正在 macOS 或 Windows 上開發，並且需要在本機安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。

<a name="directory-configuration"></a>
### 目錄設定

Laravel 應用程式應始終從為您的網路伺服器設定的「網路目錄」根目錄提供服務。您不應嘗試從「網路目錄」的子目錄提供 Laravel 應用程式。這樣做可能會暴露應用程式中的敏感檔案。

<a name="installation-using-herd"></a>
## 使用 Herd 安裝

[Laravel Herd](https://herd.laravel.com) 是一個用於 macOS 和 Windows 的極速原生 Laravel 和 PHP 開發環境。Herd 包含了您開始 Laravel 開發所需的一切，包括 PHP 和 Nginx。

一旦安裝 Herd，您就可以開始使用 Laravel 進行開發。Herd 包含 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 的命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 透過額外的強大功能強化了 Herd，例如建立和管理本機 MySQL、Postgres 和 Redis 資料庫的能力，以及本機郵件檢視和日誌監控。

<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上開發，可以從 [Herd 網站](https://herd.laravel.com) 下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 版的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「停放 (parked)」目錄。任何位於停放目錄中的 Laravel 應用程式都將自動由 Herd 提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停放目錄，您可以使用其目錄名稱透過 `.test` 網域存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用 Herd 隨附的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您也可以透過 Herd 的 UI 管理您的停放目錄和其他 PHP 設定，該 UI 可以從系統匣中的 Herd 選單開啟。

您可以查閱 [Herd 文件](https://herd.laravel.com/docs) 以了解更多關於 Herd 的資訊。

<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以從 [Herd 網站](https://herd.laravel.com/windows) 下載 Windows 版的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成入門流程並首次存取 Herd UI。

透過左鍵點擊 Herd 的系統匣圖示即可存取 Herd UI。右鍵點擊會開啟快速選單，其中包含您日常所需的所有工具。

在安裝期間，Herd 會在您的家目錄中 `%USERPROFILE%\Herd` 建立一個「停放 (parked)」目錄。任何位於停放目錄中的 Laravel 應用程式都將自動由 Herd 提供服務，您可以使用其目錄名稱透過 `.test` 網域存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新的 Laravel 應用程式最快的方式是使用 Herd 隨附的 Laravel CLI。若要開始使用，請開啟 Powershell 並執行以下指令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以查閱 [Windows 版 Herd 文件](https://herd.laravel.com/docs/windows) 以了解更多關於 Herd 的資訊。

<a name="ide-support"></a>
## IDE 支援

您在開發 Laravel 應用程式時可以自由使用任何程式碼編輯器。如果您正在尋找輕量級且可擴展的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 結合官方的 [Laravel VS Code Extension](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel) 提供了出色的 Laravel 支援，具有語法高亮、程式碼片段、Artisan 指令整合，以及針對 Eloquent 模型、路由、中介層 (middleware)、資源 (assets)、設定 (config) 和 Inertia.js 的智能自動完成等功能。

若要獲得 Laravel 廣泛且穩固的支援，請參考 JetBrains IDE —— [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)。PhpStorm 內建的 Laravel 框架支援包括 Blade 模板、針對 Eloquent 模型、路由、視圖、翻譯和元件的智能自動完成功能，以及強大的程式碼生成和跨 Laravel 專案的導航。

對於那些尋求雲端開發體驗的人來說，[Firebase Studio](https://firebase.studio/) 提供了直接在瀏覽器中建構 Laravel 應用程式的即時存取。無需任何設定，Firebase Studio 讓您可以輕鬆地從任何裝置開始建構 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個功能強大的工具，它彌合了 AI 程式碼代理與 Laravel 應用程式之間的鴻溝。Boost 為 AI 代理提供了 Laravel 特定的上下文、工具和指南，使它們能夠生成更準確、特定版本且遵循 Laravel 慣例的程式碼。

當您在 Laravel 應用程式中安裝 Boost 時，AI 代理可以存取超過 15 種專業工具，包括：了解您正在使用的套件、查詢您的資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、生成測試以及透過 Tinker 執行程式碼。

此外，Boost 還為 AI 代理提供了超過 17,000 份向量化的 Laravel 生態系統文件，這些文件針對您已安裝的套件版本。這意味著代理可以提供針對您的專案所使用的確切版本的指導。

Boost 還包含了 Laravel 維護的 AI 指南，這些指南可幫助代理遵循框架慣例、編寫適當的測試，並在生成 Laravel 程式碼時避免常見的陷阱。

<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在運行 PHP 8.1 或更高版本的 Laravel 10、11 和 12 應用程式中。要開始使用，請將 Boost 作為開發依賴項安裝：

```shell
composer require laravel/boost --dev
```

安裝後，運行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式將自動偵測您的 IDE 和 AI 代理，讓您選擇加入對您的專案有意義的功能。Boost 尊重現有的專案慣例，預設不強制執行主觀的風格規則。

> [!NOTE]
> 要了解更多關於 Boost 的資訊，請查閱 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。

<a name="next-steps"></a>
## 接下來的步驟

既然您已經建立了 Laravel 應用程式，您可能會想知道接下來要學習什麼。首先，我們強烈建議您透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [服務容器](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您希望如何使用 Laravel 也將決定您旅程中的下一步。使用 Laravel 的方式有很多種，我們將在下面探討該框架的兩種主要使用案例。

<a name="laravel-the-fullstack-framework"></a>
### Laravel：全端框架

Laravel 可以作為一個全端框架。所謂「全端」框架，我們指的是您將使用 Laravel 來將請求路由到您的應用程式，並透過 [Blade 範本](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染前端。這是使用 Laravel 框架最常見的方式，也是我們認為最有效率的使用方式。

如果您打算這樣使用 Laravel，您可能會想查看我們關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會對學習 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 等社群套件感興趣。這些套件讓您可以使用 Laravel 作為全端框架，同時享受單頁 JavaScript 應用程式提供的許多 UI 優勢。

如果您正在使用 Laravel 作為全端框架，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 編譯應用程式的 CSS 和 JavaScript。

> [!NOTE]
> 如果您想在建構應用程式時搶占先機，請查看我們的官方 [應用程式入門套件](/docs/{{version}}/starter-kits)。

<a name="laravel-the-api-backend"></a>
### Laravel：API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可以將 Laravel 用作 [Next.js](https://nextjs.org) 應用程式的 API 後端。在此情境中，您可以使用 Laravel 為應用程式提供 [認證](/docs/{{version}}/sanctum) 和資料儲存/擷取，同時還能利用 Laravel 強大的服務，例如隊列、電子郵件、通知等等。

如果您打算這樣使用 Laravel，您可能會想查看我們關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。