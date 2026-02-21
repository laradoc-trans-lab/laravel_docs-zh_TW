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
- [使用 Herd 進行安裝](#installation-using-herd)
    - [macOS 上的 Herd](#herd-on-macos)
    - [Windows 上的 Herd](#herd-on-windows)
- [IDE 支援](#ide-support)
- [Laravel 與 AI](#laravel-and-ai)
    - [安裝 Laravel Boost](#installing-laravel-boost)
- [後續步驟](#next-steps)
    - [Laravel 作為全端框架](#laravel-the-fullstack-framework)
    - [Laravel 作為 API 後端](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一個具有表現力豐富且語法優雅的 Web 應用程式框架。Web 框架為建立您的應用程式提供了結構和起點，讓您可以專注於創造令人驚嘆的作品，而我們則負責處理瑣碎的細節。

Laravel 致力於提供卓越的開發者體驗，同時提供強大的功能，如徹底的依賴注入 (Dependency Injection)、表現力豐富的資料庫抽象層、隊列 (Queues) 和排程任務、單元測試與整合測試等等。

無論您是 PHP Web 框架的新手，還是擁有多年經驗，Laravel 都是一個可以與您共同成長的框架。我們將協助您踏出作為 Web 開發者的第一步，或者在您將專業知識提升到更高境界時助您一臂之力。我們迫不及待想看到您的作品。


<a name="why-laravel"></a>
### 為什麼選擇 Laravel？

在建構 Web 應用程式時，有許多工具和框架可供選擇。然而，我們相信 Laravel 是建構現代化全端 Web 應用程式的最佳選擇。


#### 漸進式框架

我們喜歡稱 Laravel 為「漸進式」框架。這意味著 Laravel 會隨著您的成長而成長。如果您剛踏入 Web 開發領域，Laravel 龐大的文件庫、指南和[影片教學](https://laracasts.com)將協助您掌握基本技巧，而不會感到不知所措。

如果您是資深開發者，Laravel 為您提供了強大的工具，用於 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[隊列 (Queues)](/docs/{{version}}/queues)、[即時事件 (Real-time Events)](/docs/{{version}}/broadcasting) 等等。Laravel 經過精細調整，旨在建構專業的 Web 應用程式，並已準備好處理企業級的工作負載。


#### 具擴展性的框架

Laravel 具有驚人的可擴展性。歸功於 PHP 對擴展友好的特性，以及 Laravel 內建支援 Redis 等快速的分佈式快取系統，使用 Laravel 進行水平擴展（Horizontal Scaling）是輕而易舉的事。事實上，Laravel 應用程式已被輕鬆擴展以處理每月數億次的請求。

需要極致的擴展？像 [Laravel Cloud](https://cloud.laravel.com) 這樣的平台允許您以近乎無限的規模運行您的 Laravel 應用程式。


#### 支援代理程式的框架

Laravel 偏好約定優於配置（Opinionated Conventions）的特性以及定義良好的結構，使其成為使用 Cursor 和 Claude Code 等工具進行 [AI 輔助開發](/docs/{{version}}/ai) 的理想框架。當您要求 AI 代理程式新增一個控制器（Controller）時，它確切知道應該將其放在哪裡。當您需要一個新的遷移（Migration）時，命名慣例和檔案位置都是可預測的。這種一致性消除了在更靈活的框架中經常困擾 AI 工具的猜測工作。

除了檔案組織之外，Laravel 表現力豐富的語法和詳盡的文件為 AI 代理程式提供了生成準確且慣用程式碼所需的上下文。像是 Eloquent 關聯、表單請求（Form Requests）和中介層（Middleware）等功能都遵循著代理程式可以可靠理解並複製的模式。其結果是生成出的 AI 程式碼看起來就像是由資深的 Laravel 開發者編寫的，而不是由通用的 PHP 片段拼湊而成。

要深入了解為什麼 Laravel 是 AI 輔助開發的完美選擇，請參閱我們關於 [代理式開發 (Agentic Development)](/docs/{{version}}/ai) 的文件。


#### 社群框架

Laravel 結合了 PHP 生態系統中最好的套件，提供了最穩健且對開發者友善的框架。此外，來自世界各地數以千計的優秀開發者都已[為框架做出了貢獻](https://github.com/laravel/framework)。誰知道呢，也許您甚至會成為 Laravel 的貢獻者。


<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式


<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝器

在建立您的第一個 Laravel 應用程式之前，請確保您的本地電腦已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 以及 [Laravel 安裝器](https://github.com/laravel/installer)。此外，您應該安裝 [Node 和 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯應用程式的前端資源。

如果您本地電腦尚未安裝 PHP 和 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝器：

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

執行上述其中一個指令後，您應該重新啟動終端機連線。若要在透過 `php.new` 安裝後更新 PHP、Composer 和 Laravel 安裝器，您可以再次在終端機中執行該指令。

如果您已經安裝了 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝器：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 若要體驗功能齊全且具備圖形介面的 PHP 安裝與管理體驗，請參考 [Laravel Herd](#installation-using-herd)。


<a name="creating-an-application"></a>
### 建立應用程式

在您安裝完 PHP、Composer 和 Laravel 安裝器後，就可以準備建立新的 Laravel 應用程式了。Laravel 安裝器會引導您選擇偏好的測試框架、資料庫和入門套件（Starter Kit）：

```shell
laravel new example-app
```

應用程式建立完成後，您可以使用 `dev` Composer 腳本來啟動 Laravel 的本地開發伺服器、隊列工作者（Queue Worker）以及 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，您可以透過網頁瀏覽器訪問 [http://localhost:8000](http://localhost:8000) 進入您的應用程式。接著，您就可以準備 [開始探索 Laravel 生態系統的後續步驟](#next-steps)。當然，您可能也想要 [設定資料庫](#databases-and-migrations)。

> [!NOTE]
> 如果您希望在開發 Laravel 應用程式時能有個起點，請考慮使用我們的 [入門套件 (Starter Kits)](/docs/{{version}}/starter-kits)。Laravel 的入門套件為您的新 Laravel 應用程式提供了後端與前端的身分驗證鷹架 (Scaffolding)。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有完善的文件說明，因此請隨意瀏覽這些檔案，熟悉可用的選項。

Laravel 在預設情況下幾乎不需要額外的設定。您可以直接開始開發！不過，您可能希望查看 `config/app.php` 檔案及其文件。它包含了幾個您可能希望根據應用程式進行更改的選項，例如 `url` 和 `locale`。


<a name="environment-based-configuration"></a>
### 基於環境的設定

由於 Laravel 的許多設定選項值可能會根據您的應用程式是在本地機器還是正式環境伺服器上執行而有所不同，因此許多重要的設定值是使用應用程式根目錄下的 `.env` 檔案來定義的。

您的 `.env` 檔案不應該被提交到應用程式的版本控制系統中，因為每個使用您應用程式的開發者或伺服器可能需要不同的環境設定。此外，如果入侵者獲得了您的版本控制儲存庫的存取權限，這將會是一個安全風險，因為任何敏感的憑證都會暴露。

> [!NOTE]
> 有關 `.env` 檔案和基於環境設定的更多資訊，請查看完整的[設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="databases-and-migrations"></a>
### 資料庫與遷移

現在您已經建立了 Laravel 應用程式，您可能想要在資料庫中儲存一些資料。預設情況下，您的應用程式 `.env` 設定檔指定了 Laravel 將與 SQLite 資料庫進行互動。

在建立應用程式的過程中，Laravel 已為您建立了一個 `database/database.sqlite` 檔案，並執行了必要的遷移來建立應用程式的資料庫資料表。

如果您偏好使用其他資料庫驅動程式（如 MySQL 或 PostgreSQL），您可以更新您的 `.env` 設定檔以使用適當的資料庫。例如，如果您希望使用 MySQL，請如下更新 `.env` 設定檔的 `DB_*` 變數：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，則需要建立資料庫並執行應用程式的[資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]
> 如果您在 macOS 或 Windows 上開發，並且需要在本地安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。


<a name="directory-configuration"></a>
### 目錄設定

Laravel 應該始終從為網頁伺服器設定的「網頁目錄 (Web Directory)」根目錄提供服務。您不應嘗試從「網頁目錄」的子目錄中提供 Laravel 應用程式。這樣做可能會暴露應用程式中存在的敏感檔案。


<a name="installation-using-herd"></a>
## 使用 Herd 進行安裝

[Laravel Herd](https://herd.laravel.com) 是一個極速的、原生的 macOS 與 Windows 專用 Laravel 和 PHP 開發環境。Herd 包含了開始進行 Laravel 開發所需的一切，包括 PHP 和 Nginx。

一旦安裝了 Herd，您就可以開始使用 Laravel 進行開發。Herd 包含了 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 等命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 為 Herd 增加了額外的強大功能，例如建立與管理本地 MySQL、Postgres 和 Redis 資料庫的能力，以及本地郵件查看和日誌監控功能。


<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上開發，可以從 [Herd 官方網站](https://herd.laravel.com)下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 版的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「掛載 (Parked)」目錄。掛載目錄中的任何 Laravel 應用程式都會自動由 Herd 提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個掛載目錄，您可以使用其目錄名稱在 `.test` 網域存取該目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用 Herd 隨附的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您始終可以透過 Herd 的使用者介面 (UI) 管理您的掛載目錄和其他 PHP 設定，該介面可以從系統列的 Herd 選單中開啟。

您可以透過查看 [Herd 文件](https://herd.laravel.com/docs)來了解更多關於 Herd 的資訊。


<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以從 [Herd 官方網站](https://herd.laravel.com/windows)下載 Windows 版的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成引導程序，並首次存取 Herd UI。

透過左鍵點選 Herd 的系統列圖示即可開啟 Herd UI。點選右鍵則會開啟快速選單，可存取您日常所需的所有工具。

在安裝過程中，Herd 會在您的家目錄 `%USERPROFILE%\Herd` 中建立一個「掛載 (Parked)」目錄。掛載目錄中的任何 Laravel 應用程式都會自動由 Herd 提供服務，您可以使用其目錄名稱在 `.test` 網域存取該目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用 Herd 隨附的 Laravel CLI。要開始使用，請開啟 Powershell 並執行以下指令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以透過查看 [Windows 版 Herd 文件](https://herd.laravel.com/docs/windows)來了解更多關於 Herd 的資訊。


<a name="ide-support"></a>
## IDE 支援

在開發 Laravel 應用程式時，您可以自由使用任何您喜歡的程式碼編輯器。如果您正在尋找輕量級且可擴充的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 結合官方的 [Laravel VS Code 擴充功能](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel) 提供了出色的 Laravel 支援，具有語法突顯、程式碼片段 (Snippets)、Artisan 指令整合，以及針對 Eloquent 模型、路由、中介層、資產、設定和 Inertia.js 的智慧自動補全功能。

對於 Laravel 更廣泛且強大的支援，可以查看 [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)，這是一款 JetBrains IDE。PhpStorm 內建的 Laravel 框架支援包括 Blade 範本、針對 Eloquent 模型、路由、視圖、翻譯和元件的智慧自動補全，以及強大的程式碼生成和跨 Laravel 專案的導航功能。

對於尋求雲端開發體驗的使用者，[Firebase Studio](https://firebase.studio/) 讓您可以在瀏覽器中直接即時建立 Laravel 專案。無需任何設定，Firebase Studio 讓您可以輕鬆地從任何裝置開始建立 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個強大的工具，它搭起了 AI 編碼代理 (Agent) 與 Laravel 應用程式之間的橋樑。Boost 為 AI 代理提供了 Laravel 特定的上下文、工具與指南，使其能夠生成更準確、符合特定版本且遵循 Laravel 慣例的程式碼。

當你在 Laravel 應用程式中安裝 Boost 後，AI 代理即可存取超過 15 種專業工具，包括了解你正在使用的套件、查詢資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、生成測試以及透過 Tinker 執行程式碼的能力。

此外，Boost 還讓 AI 代理可以存取超過 17,000 份向量化的 Laravel 生態系統文件，且這些文件會根據你安裝的套件版本進行媒合。這意味著代理可以提供針對你專案所使用的確切版本之指引。

Boost 還包含由 Laravel 維護的 AI 指南，幫助代理遵循框架慣例、撰寫適當的測試，並在生成 Laravel 程式碼時避免常見的陷阱。


<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在運行 PHP 8.1 或更高版本的 Laravel 10, 11, 與 12 應用程式中。首先，將 Boost 安裝為開發依賴項目：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝器：

```shell
php artisan boost:install
```

安裝器會自動偵測你的 IDE 與 AI 代理，讓你選擇適合專案的功能。Boost 預設會尊重現有的專案慣例，且不會強制執行主觀的風格規則。

> [!NOTE]
> 若要進一步了解 Boost，請查看 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。


<a name="adding-custom-ai-guidelines"></a>
#### 新增自定義 AI 指南

若要使用你自己的自定義 AI 指南來擴充 Laravel Boost，請將 `.blade.php` 或 `.md` 檔案新增到應用程式的 `.ai/guidelines/*` 目錄中。當你執行 `boost:install` 時，這些檔案將會自動包含在 Laravel Boost 的指南中。


<a name="next-steps"></a>
## 後續步驟

既然你已經建立了 Laravel 應用程式，你可能會想接下來該學習什麼。首先，我們強烈建議透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期 (Request Lifecycle)](/docs/{{version}}/lifecycle)
- [設定 (Configuration)](/docs/{{version}}/configuration)
- [目錄結構 (Directory Structure)](/docs/{{version}}/structure)
- [前端 (Frontend)](/docs/{{version}}/frontend)
- [服務容器 (Service Container)](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

你想要如何使用 Laravel 也會決定你旅程的下一步。使用 Laravel 有多種方式，我們將在下方探索該框架的兩個主要使用場景。


<a name="laravel-the-fullstack-framework"></a>
### Laravel 作為全端框架

Laravel 可以作為全端 (Full Stack) 框架。所謂「全端」框架，是指你將使用 Laravel 來處理應用程式的路由請求，並透過 [Blade 模板](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染前端。這是使用 Laravel 框架最常見的方式，且在我們看來，也是使用 Laravel 最高效的方式。

如果你打算這樣使用 Laravel，你可能想查看關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖 (Views)](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，你可能也會對學習像 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 這樣的社群套件感興趣。這些套件讓你能在享受單頁 JavaScript 應用程式提供的許多 UI 優勢的同時，將 Laravel 作為全端框架使用。

如果你將 Laravel 作為全端框架使用，我們也強烈建議你學習如何使用 [Vite](/docs/{{version}}/vite) 來編譯應用程式的 CSS 與 JavaScript。

> [!NOTE]
> 如果你想在建立應用程式時領先一步，請查看我們官方的 [應用程式入門套件 (Starter Kits)](/docs/{{version}}/starter-kits)。


<a name="laravel-the-api-backend"></a>
### Laravel 作為 API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，你可以使用 Laravel 作為 [Next.js](https://nextjs.org) 應用程式的 API 後端。在這種情況下，你可以使用 Laravel 為應用程式提供 [驗證 (Authentication)](/docs/{{version}}/sanctum) 與資料儲存 / 檢索，同時還能利用 Laravel 強大的服務，如隊列 (Queues)、電子郵件、通知等。

如果你打算這樣使用 Laravel，你可能想查看關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 以及 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。