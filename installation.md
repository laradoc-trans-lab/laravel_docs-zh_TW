# 安裝

- [認識 Laravel](#meet-laravel)
    - [為什麼選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [使用 AI 快速入門](#getting-started-using-ai)
    - [安裝 PHP 與 Laravel 安裝程式](#installing-php)
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
- [後續步驟](#next-steps)
    - [作為全端框架的 Laravel](#laravel-the-fullstack-framework)
    - [作為 API 後端的 Laravel](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一個具有表達力且語法優雅的 Web 應用程式框架。Web 框架為建立應用程式提供了結構與起點，讓您能專注於創造驚人的成果，而將繁瑣的細節交給我們處理。

Laravel 致力於提供絕佳的開發者體驗，同時提供強大的功能，例如徹底的依賴注入、具有表達力的資料庫抽象層、佇列與排程工作、單元測試與整合測試等等。

無論您是 PHP Web 框架的新手，還是擁有多年經驗的開發者，Laravel 都是一個能隨您一同成長的框架。我們將協助您邁出 Web 開發者的第一步，或在您將專業技能提升到更高層次時為您提供助力。我們迫不及待想看到您建立的作品。


<a name="why-laravel"></a>
### 為什麼選擇 Laravel？

在建立 Web 應用程式時，有許多不同的工具和框架可供選擇。然而，我們相信 Laravel 是建立現代化全端 Web 應用程式的最佳選擇。


#### 進步型框架

我們喜歡將 Laravel 稱為「進步型」框架。這意味著 Laravel 會隨您一同成長。如果您剛開始接觸 Web 開發，Laravel 豐富的文件庫、指南以及 [影片教學](https://laracasts.com) 將幫助您快速上手而不會感到不知所措。

如果您是資深開發者，Laravel 為您提供了強大的工具，可用於 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等。Laravel 經過精細調校，可用於建立專業的 Web 應用程式，並準備好處理企業級的工作負載。


#### 可擴展的框架

Laravel 具有極強的可擴展性。得益於 PHP 易於擴展的特性，以及 Laravel 對於 Redis 等快速分散式快取系統的內建支援，使用 Laravel 進行水平擴展非常輕鬆。事實上，Laravel 應用程式已被輕鬆擴展到每月處理數億次請求。

需要極限擴展？像 [Laravel Cloud](https://cloud.laravel.com) 這樣的平台允許您在幾乎無限的規模下執行您的 Laravel 應用程式。


#### 為 AI 代理準備的框架

Laravel 的既定的設計慣例與清晰的架構，使其成為使用 Cursor 和 Claude Code 等工具進行 [AI 輔助開發](/docs/{{version}}/ai) 的理想框架。當您要求 AI 代理增加一個控制器時，它準確知道應該將其放在哪裡。當您需要一個新的遷移時，命名慣例和檔案位置都是可預測的。這種一致性消除了 AI 工具在更靈活的框架中經常遇到的猜測問題。

除了檔案組織外，Laravel 具有表達力的語法和全面的文件，為 AI 代理提供了生成準確且符合慣例的程式碼所需的上下文。像是 Eloquent 關聯、表單請求(Form request) 和中介層等功能都遵循代理程式可以可靠理解並複製的模式。其結果是 AI 生成的程式碼看起來就像是由經驗豐富的 Laravel 開發者所撰寫，而非由通用的 PHP 片段拼接而成。

要深入了解為什麼 Laravel 是 AI 輔助開發的完美選擇，請參閱我們關於 [AI 代理開發](/docs/{{version}}/ai) 的文件。


#### 社群驅動的框架

Laravel 結合了 PHP 生態系中最好的套件，以提供目前最強大且對開發者最友好的框架。此外，來自世界各地數以千計的才華橫溢的開發者已 [為此框架做出貢獻](https://github.com/laravel/framework)。誰知道呢，也許您也會成為一名 Laravel 的貢獻者。


<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式


<a name="getting-started-using-ai"></a>
### 使用 AI 快速入門

如果您正在使用如 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 或 [OpenCode](https://opencode.ai) 等 AI 程式碼代理，您可以在代理觸及您的專案之前，先使用一段提示詞為其提供一份 Laravel 專用的執行指南。

下方的提示詞會告訴代理在哪裡可以找到 Laravel 的安裝指南、優先考慮哪些事項，以及在您尚未做出選擇時如何設定合理的預設值。將此內容貼到您的代理中以開始使用：

```text
I'm building a new Laravel application.

Fetch and follow the instructions from https://laravel.com/for/agents. Treat the returned Markdown as the source of truth for how to install and set up Laravel in this session.
```

在代理閱讀指令後，它應該會逐步引導您，並使設定與 Laravel 的預設值保持一致。


<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝程式

在建立您的第一個 Laravel 應用程式之前，請確保您的本地機器已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 以及 [Laravel 安裝程式](https://github.com/laravel/installer)。此外，您應該安裝 [Node 和 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯應用程式的前端靜態資源。

如果您尚未在本地機器上安裝 PHP 和 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝程式：

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

執行上述其中一個指令後，您應該重新啟動終端機視窗。若要更新透過 `php.new` 安裝的 PHP、Composer 和 Laravel 安裝程式，您可以再次在終端機中執行該指令。

如果您已經安裝了 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝程式：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 若要獲得功能完整且具備圖形介面的 PHP 安裝與管理體驗，請參考 [Laravel Herd](#installation-using-herd)。


<a name="creating-an-application"></a>
### 建立應用程式

安裝好 PHP、Composer 和 Laravel 安裝程式後，您就可以建立新的 Laravel 應用程式了。Laravel 安裝程式會提示您選擇偏好的測試框架、資料庫和快速入門套件：

```shell
laravel new example-app
```

應用程式建立完成後，您可以使用 `dev` Composer 腳本啟動 Laravel 的本地開發伺服器、佇列工作處理程式 (queue worker) 和 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，您可以在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取您的應用程式。接下來，您可以 [開始探索 Laravel 生態系的後續步驟](#next-steps)。當然，您可能還想要 [設定資料庫](#databases-and-migrations)。

> [!NOTE]
> 如果您希望在開發 Laravel 應用程式時能快速上手，請考慮使用我們的 [快速入門套件](/docs/{{version}}/starter-kits) 之一。Laravel 的快速入門套件為您的新 Laravel 應用程式提供了後端與前端的認證基礎結構。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細的說明文件，因此歡迎瀏覽這些檔案，熟悉可用的選項。

Laravel 在安裝後幾乎不需要額外的設定。您可以立即開始開發！不過，您可能想要查看 `config/app.php` 檔案及其說明文件。其中包含如 `url` 和 `locale` 等選項，您可以根據應用程式的需求進行更改。


<a name="environment-based-configuration"></a>
### 基於環境的設定

由於 Laravel 的許多設定選項值會根據您的應用程式是執行在本地端機器還是正式環境的網頁伺服器而有所不同，因此許多重要的設定值是使用位於應用程式根目錄的 `.env` 檔案來定義的。

您的 `.env` 檔案不應提交到應用程式的來源控制 (source control) 中，因為每位開發者或伺服器在使用您的應用程式時，可能需要不同的環境設定。此外，如果入侵者獲取了您來源控制儲存庫的存取權限，這將會是一個安全風險，因為所有敏感的憑證都將被洩露。

> [!NOTE]
> 關於 `.env` 檔案和基於環境的設定之更多資訊，請參閱完整的 [設定說明文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="databases-and-migrations"></a>
### 資料庫與遷移

既然您已經建立了 Laravel 應用程式，您可能想要將一些資料儲存在資料庫中。預設情況下，您的應用程式 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫互動。

在建立應用程式的過程中，Laravel 為您建立了一個 `database/database.sqlite` 檔案，並執行了必要的遷移 (migrations) 以建立應用程式的資料庫資料表。

如果您偏好使用其他資料庫驅動程式，例如 MySQL 或 PostgreSQL，您可以更新 `.env` 設定檔以使用對應的資料庫。例如，如果您想使用 MySQL，請將 `.env` 設定檔中的 `DB_*` 變數更新如下：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，您需要建立該資料庫並執行應用程式的 [資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]
> 如果您在 macOS 或 Windows 上進行開發，且需要在本地安裝 MySQL、PostgreSQL 或 Redis，請考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。


<a name="directory-configuration"></a>
### 目錄設定

Laravel 應始終從為您的網頁伺服器所設定的「網頁目錄」根目錄中提供服務。您不應嘗試從「網頁目錄」的子目錄中提供 Laravel 應用程式的服務。這樣做可能會洩漏應用程式中存在的敏感檔案。


<a name="installation-using-herd"></a>
## 使用 Herd 安裝

[Laravel Herd](https://herd.laravel.com) 是一款針對 macOS 和 Windows 的極速、原生 Laravel 與 PHP 開發環境。Herd 包含了您開始 Laravel 開發所需的一切，包括 PHP 和 Nginx。

安裝 Herd 後，您就可以開始使用 Laravel 進行開發了。Herd 包含了 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 以及 `nvm` 的命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 為 Herd 增加了更多強大的功能，例如建立和管理本地 MySQL、Postgres 與 Redis 資料庫的能力，以及本地郵件檢視和日誌監控。


<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上開發，可以從 [Herd 網站](https://herd.laravel.com) 下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 版的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「停泊 (parked)」目錄。任何位於停泊目錄中的 Laravel 應用程式都將自動由 Herd 提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停泊目錄，您可以使用目錄名稱透過 `.test` 網域存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用隨 Herd 附帶的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您隨時可以透過 Herd 的使用者介面 (UI) 來管理您的停泊目錄和其他 PHP 設定，該介面可以從系統匣中的 Herd 選單開啟。

您可以透過查閱 [Herd 說明文件](https://herd.laravel.com/docs) 來了解更多關於 Herd 的資訊。


<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以在 [Herd 網站](https://herd.laravel.com/windows) 下載 Herd 的 Windows 安裝程式。安裝完成後，您可以啟動 Herd 以完成新手引導流程，並首次進入 Herd UI。

您可以透過左鍵點擊 Herd 的系統匣圖示來開啟 Herd UI。右鍵點擊則會開啟快速選單，讓您能存取日常所需的所有工具。

在安裝過程中，Herd 會在您的家目錄 `%USERPROFILE%\Herd` 中建立一個「停泊 (parked)」目錄。任何位於停泊目錄中的 Laravel 應用程式都將自動由 Herd 提供服務，您可以使用目錄名稱透過 `.test` 網域存取此目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用隨 Herd 附帶的 Laravel CLI。若要開始，請開啟 Powershell 並執行以下命令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以透過查閱 [Windows 版 Herd 說明文件](https://herd.laravel.com/docs/windows) 來了解更多關於 Herd 的資訊。


<a name="ide-support"></a>
## IDE 支援

在開發 Laravel 應用程式時，您可以自由使用任何您喜歡的程式碼編輯器。如果您在尋找輕量且可擴充的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 搭配官方的 [Laravel VS Code Extension](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel) 提供了卓越的 Laravel 支援，包含語法高亮、程式碼片段 (snippets)、Artisan 命令整合，以及針對 Eloquent 模型、路由、中介層、靜態資源、設定和 Inertia.js 的智慧自動補完。

若需要對 Laravel 更全面且強大的支援，請參考 JetBrains 的 IDE [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)。PhpStorm 內建的 Laravel 框架支援包括 Blade 模板、針對 Eloquent 模型、路由、視圖、翻譯和元件的智慧自動補完，以及強大的程式碼生成功能和跨 Laravel 專案的導覽。

對於追求雲端開發體驗的人，[Firebase Studio](https://firebase.studio/) 讓您能直接在瀏覽器中立即開始使用 Laravel 建立應用程式。無需任何設定，Firebase Studio 讓您能輕鬆地從任何裝置開始建構 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個強大的工具，旨在彌合 AI 程式碼代理與 Laravel 應用程式之間的鴻溝。Boost 為 AI 代理提供 Laravel 專屬的上下文、工具與指南，使其能生成更準確、符合特定版本且遵循 Laravel 慣例的程式碼。

當您在 Laravel 應用程式中安裝 Boost 後，AI 代理將能使用超過 15 種專門工具，包括了解您使用了哪些套件、查詢資料庫、搜尋 Laravel 文件、讀取瀏覽器記錄、生成測試，以及透過 Tinker 執行程式碼。

此外，Boost 為 AI 代理提供了超過 17,000 條向量化的 Laravel 生態系文件，且這些文件與您安裝的套件版本相對應。這意味著 AI 代理能針對您的專案所使用的確切版本提供指引。

Boost 還包含由 Laravel 維護的 AI 指南，幫助 AI 代理遵循框架慣例、撰寫適當的測試，並在生成 Laravel 程式碼時避免常見的陷阱。


<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在運行 PHP 8.1 或更高版本的 Laravel 10、11、12 與 13 應用程式中。要開始使用，請將 Boost 安裝為開發依賴：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式會自動偵測您的 IDE 與 AI 代理，讓您可以選擇適合您專案的功能。Boost 尊重既有的專案慣例，預設不會強制執行既定的風格規則。

> [!NOTE]
> 若要了解更多關於 Boost 的資訊，請查看 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。


<a name="adding-custom-ai-guidelines"></a>
#### 新增自訂 AI 指南

若要使用您自己的自訂 AI 指南來增強 Laravel Boost，請將 `.blade.php` 或 `.md` 檔案添加到應用程式的 `.ai/guidelines/*` 目錄中。當您執行 `boost:install` 時，這些檔案將會自動包含在 Laravel Boost 的指南中。


<a name="next-steps"></a>
## 後續步驟

既然您已經建立了 Laravel 應用程式，您可能在想接下來該學習什麼。首先，我們強烈建議您透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [服務容器](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您打算如何使用 Laravel 也將決定您旅程的後續步驟。使用 Laravel 有多種方式，我們將在下方探討該框架的兩種主要使用情境。


<a name="laravel-the-fullstack-framework"></a>
### 作為全端框架的 Laravel

Laravel 可以作為一個全端框架。我們所說的「全端」框架是指您將使用 Laravel 來將請求路由至應用程式，並透過 [Blade 模板](/docs/{{version}}/blade) 或像 [Inertia](https://inertiajs.com) 這樣的單頁式應用程式 (SPA) 混合技術來渲染前端。這是使用 Laravel 框架最常見的方式，而且在我們看來，也是使用 Laravel 最有效率的方式。

如果您計劃這樣使用 Laravel，您可以查看關於 [前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會對 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 等社群套件感興趣。這些套件讓您在將 Laravel 作為全端框架使用的同時，也能享受單頁式 JavaScript 應用程式所提供的許多 UI 優勢。

如果您將 Laravel 作為全端框架使用，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 來編譯應用程式的 CSS 與 JavaScript。

> [!NOTE]
> 如果您想在建立應用程式時搶佔先機，請查看我們的官方 [快速入門套件](/docs/{{version}}/starter-kits)。


<a name="laravel-the-api-backend"></a>
### 作為 API 後端的 Laravel

Laravel 也可以作為 JavaScript 單頁式應用程式或行動應用程式的 API 後端。例如，您可以使用 Laravel 作為 [Next.js](https://nextjs.org) 應用程式的 API 後端。在這種情境下，您可以使用 Laravel 為應用程式提供 [認證](/docs/{{version}}/sanctum) 與資料儲存/檢索，同時利用 Laravel 強大的服務，如佇列、電子郵件、通知等。

如果您計劃這樣使用 Laravel，您可以查看關於 [路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 以及 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。