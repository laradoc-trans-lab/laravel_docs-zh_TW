# 安裝

- [認識 Laravel](#meet-laravel)
    - [為什麼選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [使用 AI 開始上手](#getting-started-using-ai)
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
    - [Laravel 全端框架](#laravel-the-fullstack-framework)
    - [Laravel API 後端](#laravel-the-api-backend)

<a name="meet-laravel"></a>
## 認識 Laravel

Laravel 是一款擁有表現力與優雅語法的 Web 應用程式框架。Web 框架為建立您的應用程式提供了架構與起步點，讓您可以專注於創造令人驚嘆的作品，而細節則由我們來為您打點。

Laravel 致力於提供極佳的開發者體驗，同時提供強大的功能，例如徹底的依賴注入、具表現力的資料庫抽象層、佇列與排程任務、單元與整合測試等等。

無論您是剛接觸 PHP Web 框架的新手，還是擁有多年經驗的專家，Laravel 都是一款能與您一同成長的框架。我們將協助您踏出作為 Web 開發者的第一步，或在您將專業知識提升到下一個層次時為您提供助力。我們迫不及待想看看您建立的作品了。


<a name="why-laravel"></a>
### 為什麼選擇 Laravel？

在建立 Web 應用程式時，有許多工具和框架可供您選擇。然而，我們相信 Laravel 是建立現代化、全端 Web 應用程式的最佳選擇。


#### 漸進式框架

我們喜歡將 Laravel 稱為「漸進式」框架。意思是，Laravel 會與您一同成長。如果您剛踏入 Web 開發領域，Laravel 豐富的文件庫、指南和 [影片教學](https://laracasts.com) 將能協助您掌握基本要領，而不會感到不知所措。

如果您是資深開發者，Laravel 為您提供了強大的工具，用於 [依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting) 等等。Laravel 經過精細調整，非常適合建立專業的 Web 應用程式，並已準備好處理企業級的工作負載。


#### 具擴充性的框架

Laravel 具有極佳的擴充性。得益於 PHP 易於擴充的特性，以及 Laravel 對 Redis 等快速、分散式快取系統的內建支援，使用 Laravel 進行水平擴充變得輕而易舉。事實上，Laravel 應用程式已被輕易擴充以處理每月數億次的請求。

需要極致的擴充性嗎？像 [Laravel Cloud](https://cloud.laravel.com) 這樣的平台可以讓您在近乎無限的規模下執行您的 Laravel 應用程式。


#### 支援 AI 代理的框架

Laravel 既定的設計慣例與清晰的架構，使其成為使用 Cursor 和 Claude Code 等工具進行 [AI 輔助開發](/docs/{{version}}/ai) 的理想框架。當您要求 AI 代理新增一個控制器（Controller）時，它能確切地知道應該將其放在哪裡。當您需要新的遷移（Migration）時，命名慣例和檔案位置都是可以預測的。這種一致性消除了在更靈活的框架中經常使 AI 工具感到困惑的猜測。

除了檔案組織之外，Laravel 具表現力的語法和詳盡的文件，也為 AI 代理提供了生成準確、道地程式碼所需的上下文。像是 Eloquent 關聯、表單請求(Form request) 以及中介層等功能都遵循著代理可以可靠理解並複製的模式。其結果是，AI 生成的程式碼看起來就像是由經驗豐富的 Laravel 開發者所撰寫，而不是從通用的 PHP 片段拼接而成。

要深入了解為什麼 Laravel 是 AI 輔助開發的完美選擇，請參閱我們關於 [AI 代理開發](/docs/{{version}}/ai) 的文件。


#### 社群框架

Laravel 結合了 PHP 生態系統中最好的套件，提供了最健全且對開發者友善的框架。此外，來自全球數以千計的優秀開發者已 [為此框架做出貢獻](https://github.com/laravel/framework)。說不定，您甚至也會成為 Laravel 的貢獻者。


<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式


<a name="getting-started-using-ai"></a>
### 使用 AI 開始上手

如果您正在使用像是 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 或 [OpenCode](https://opencode.ai) 這樣的 AI 編碼代理，您可以在它接觸您的專案之前，先輸入一個提供該代理 Laravel 專用指南的提示詞。

下方的提示詞會告訴代理在哪裡可以找到 Laravel 的安裝指南、優先考慮的事項，以及在您尚未做出選擇時如何制定合理的預設值。請將此內容複製並貼上到您的代理中以開始：

```text
I'm building a new Laravel application.

Fetch and follow the instructions from https://laravel.com/for/agents. Treat the returned Markdown as the source of truth for how to install and set up Laravel in this session.
```

在代理閱讀完說明後，它應該會引導您一步步進行，並使設定與 Laravel 的預設值保持一致。


<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝器

在建立您的第一個 Laravel 應用程式之前，請確保您的本機電腦已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 以及 [the Laravel installer](https://github.com/laravel/installer)。此外，您應該安裝 [Node and NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯您應用程式的前端靜態資源。

如果您的本機電腦上尚未安裝 PHP 和 Composer，以下指令將在 macOS、Windows 或 Linux 上安裝 PHP、Composer 和 Laravel 安裝器：

```shell tab=macOS
/bin/bash -c "$(curl -fsSL https://php.new/install/mac/8.5)"
```

```shell tab=Windows PowerShell
# Run as administrator...
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://php.new/install/windows/8.5'))
```

```shell tab=Linux
/bin/bash -c "$(curl -fsSL https://php.new/install/linux/8.5)"
```

執行上述其中一個指令後，您應該重新啟動您的終端機工作階段。若要在透過 `php.new` 安裝後更新 PHP、Composer 以及 Laravel 安裝器，您可以在終端機中重新執行該指令。

如果您已經安裝了 PHP 和 Composer，您可以使用 Composer 來安裝 Laravel 安裝器：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 如需功能齊全、圖形介面的 PHP 安裝與管理體驗，請參考 [Laravel Herd](#installation-using-herd)。


<a name="creating-an-application"></a>
### 建立應用程式

在您安裝好 PHP、Composer 和 Laravel 安裝器之後，就可以準備建立新的 Laravel 應用程式了。Laravel 安裝器會提示您選擇您偏好的測試框架、資料庫以及入門套件：

```shell
laravel new example-app
```

應用程式建立完成後，您可以使用 `dev` Composer 腳本來啟動 Laravel 的本機開發伺服器、佇列工作程式以及 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，您就可以在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取您的應用程式。接下來，您就可以準備 [開始邁出進入 Laravel 生態系統的後續步驟](#next-steps)。當然，您可能也會想要 [設定資料庫](#databases-and-migrations)。

> [!NOTE]
> 如果您想在開發 Laravel 應用程式時搶得先機，可以考慮使用我們的 [入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件為您的新 Laravel 應用程式提供了後端與前端的認證鷹架。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細的說明文件，因此歡迎隨時瀏覽這些檔案，熟悉可用的設定選項。

Laravel 基本上不需要任何額外的設定即可直接使用。您可以直接開始開發！不過，您可能想查看 `config/app.php` 檔案及其文件。它包含了幾個您可能想根據應用程式進行修改的選項，例如 `url` 與 `locale`。


<a name="environment-based-configuration"></a>
### 基於環境的設定

由於 Laravel 的許多設定選項值可能會根據您的應用程式是在本機電腦還是生產環境網頁伺服器上執行而有所不同，因此許多重要的設定值都是使用存在於應用程式根目錄下的 `.env` 檔案來定義的。

您的 `.env` 檔案不應該提交到應用程式的版本控制系統中，因為每個使用您應用程式的開發者或伺服器可能都需要不同的環境設定。此外，這也會帶來安全風險，萬一入侵者取得您版本控制存放庫的存取權，任何敏感的憑證都會外洩。

> [!NOTE]
> 有關 `.env` 檔案與基於環境設定的更多資訊，請查看完整的[設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="databases-and-migrations"></a>
### 資料庫與遷移

現在您已經建立了 Laravel 應用程式，您可能想在資料庫中儲存一些資料。預設情況下，您應用程式的 `.env` 設定檔指定了 Laravel 將與 SQLite 資料庫進行互動。

在建立應用程式的過程中，Laravel 已為您建立了一個 `database/database.sqlite` 檔案，並執行了必要的遷移以建立應用程式的資料庫資料表。

如果您傾向使用其他資料庫驅動程式（例如 MySQL 或 PostgreSQL），您可以更新 `.env` 設定檔以使用相應的資料庫。例如，如果您想使用 MySQL，請像這樣更新 `.env` 設定檔的 `DB_*` 變數：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果您選擇使用 SQLite 以外的資料庫，您將需要建立該資料庫並執行應用程式的[資料庫遷移](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]
> 如果您在 macOS 或 Windows 上進行開發，且需要在本機安裝 MySQL、PostgreSQL 或 Redis，可以考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。


<a name="directory-configuration"></a>
### 目錄設定

Laravel 應始終由為您的網頁伺服器設定的 "web directory"（網頁目錄）根目錄來提供服務。您不應該嘗試從 "web directory" 的子目錄來提供 Laravel 應用程式的服務。這樣做可能會暴露您應用程式中存在的敏感檔案。


<a name="installation-using-herd"></a>
## 使用 Herd 進行安裝

[Laravel Herd](https://herd.laravel.com) 是一款適用於 macOS 和 Windows 的極速、原生 Laravel 與 PHP 開發環境。Herd 包含了您開始進行 Laravel 開發所需的一切，包括 PHP 和 Nginx。

一旦安裝了 Herd，您就可以開始使用 Laravel 進行開發。Herd 包含了 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 等命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 為 Herd 增添了更多強大的功能，例如建立與管理本機 MySQL、Postgres 和 Redis 資料庫的能力，以及本機郵件檢視和日誌監控。


<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果您在 macOS 上進行開發，您可以從 [Herd 官方網站](https://herd.laravel.com)下載 Herd 安裝程式。該安裝程式會自動下載最新版本的 PHP，並將您的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 版的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援 "parked"（停放）目錄。任何位於停放目錄中的 Laravel 應用程式都將自動由 Herd 提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個停放目錄，您可以使用目錄名稱，在 `.test` 網域上存取該目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用 Herd 隨附的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，您隨時可以透過 Herd 的 UI 來管理您的停放目錄和其他 PHP 設定，該 UI 可以從系統工具列中的 Herd 選單開啟。

您可以透過查看 [Herd 說明文件](https://herd.laravel.com/docs)來了解更多關於 Herd 的資訊。


<a name="herd-on-windows"></a>
### Windows 上的 Herd

您可以在 [Herd 官方網站](https://herd.laravel.com/windows)下載 Windows 版的 Herd 安裝程式。安裝完成後，您可以啟動 Herd 以完成引導流程，並首次存取 Herd UI。

在 Herd 的系統工具列圖示上按一下左鍵，即可存取 Herd UI。按一下右鍵則可開啟快速選單，其中包含您日常所需的所有工具。

在安裝過程中，Herd 會在您的個人家目錄中建立一個 "parked"（停放）目錄，路徑為 `%USERPROFILE%\Herd`。任何位於停放目錄中的 Laravel 應用程式都將自動由 Herd 提供服務，您可以使用目錄名稱，在 `.test` 網域上存取該目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立新 Laravel 應用程式最快的方法是使用 Herd 隨附的 Laravel CLI。若要開始，請開啟 PowerShell 並執行以下命令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

您可以透過查看 [Windows 版 Herd 說明文件](https://herd.laravel.com/docs/windows)來了解更多關於 Herd 的資訊。


<a name="ide-support"></a>
## IDE 支援

在開發 Laravel 應用程式時，您可以自由使用任何您喜歡的程式碼編輯器。如果您正在尋找輕量且具擴充性的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 結合官方的 [Laravel VS Code 擴充套件](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel)提供了極佳的 Laravel 支援，具有語法突顯、程式碼片段、Artisan 指令整合，以及針對 Eloquent 模型、路由、中介層、靜態資源、設定與 Inertia.js 的智慧自動補全功能。

若需要更全面且強大的 Laravel 支援，可以看看 JetBrains 旗下的 IDE [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)。PhpStorm 內建對 Laravel 框架的支援，包括 Blade 模板，針對 Eloquent 模型、路由、視圖、翻譯與元件的智慧自動補全，以及在整個 Laravel 專案中強大的程式碼生成與導覽功能。

對於追求雲端開發體驗的人來說，[Firebase Studio](https://firebase.studio/) 提供了直接在瀏覽器中建置 Laravel 的即時管道。無需任何設定，Firebase Studio 讓您能輕鬆地在任何裝置上開始建立 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一個強大的工具，橋接了 AI 開發代理 (AI coding agents) 與 Laravel 應用程式之間的鴻溝。Boost 為 AI 代理提供了 Laravel 專屬的上下文、工具和指南，使它們能夠生成更精確、符合特定版本且遵循 Laravel 慣例的程式碼。

當您在 Laravel 應用程式中安裝 Boost 後，AI 代理即可使用超過 15 種專屬工具，包括了解您正在使用的套件、查詢您的資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、生成測試以及透過 Tinker 執行程式碼。

此外，Boost 還能讓 AI 代理存取超過 17,000 篇向量化的 Laravel 生態系統文件，且這些文件會與您安裝的套件版本保持一致。這意味著 AI 代理可以針對您專案所使用的確切版本來提供指導。

Boost 還包含了由 Laravel 官方維護的 AI 指南，幫助 AI 代理遵循框架慣例、撰寫適當的測試，並在生成 Laravel 程式碼時避免常見的陷阱。


<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在運行 PHP 8.1 或更高版本的 Laravel 10、11、12 和 13 應用程式中。要開始使用，請將 Boost 安裝為開發依賴項：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式會自動偵測您的 IDE 和 AI 代理，讓您選擇啟用適合您專案的功能。Boost 尊重現有的專案慣例，預設情況下不會強加主觀的樣式規則。

> [!NOTE]
> 欲深入了解 Boost，請參閱 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。


<a name="adding-custom-ai-guidelines"></a>
#### 新增自訂 AI 指南

若要使用您自己自訂的 AI 指南來擴充 Laravel Boost，請將 `.blade.php` 或 `.md` 檔案新增至您應用程式的 `.ai/guidelines/*` 目錄中。當您執行 `boost:install` 時，這些檔案將會自動包含在 Laravel Boost 的指南中。


<a name="next-steps"></a>
## 後續步驟

現在您已經建立了您的 Laravel 應用程式，您可能會想知道接下來該學習什麼。首先，我們強烈建議您閱讀以下文件，以熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期 (Request Lifecycle)](/docs/{{version}}/lifecycle)
- [設定 (Configuration)](/docs/{{version}}/configuration)
- [目錄結構 (Directory Structure)](/docs/{{version}}/structure)
- [前端 (Frontend)](/docs/{{version}}/frontend)
- [服務容器 (Service Container)](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您想如何使用 Laravel 也將決定您旅程的下一步。使用 Laravel 的方式有很多種，我們將在下方探討該框架的兩個主要使用場景。


<a name="laravel-the-fullstack-framework"></a>
### Laravel 全端框架

Laravel 可以作為全端框架使用。我們所說的「全端」框架，是指您將使用 Laravel 來為您的應用程式路由請求，並透過 [Blade 範本](/docs/{{version}}/blade)或像 [Inertia](https://inertiajs.com) 這樣的單頁應用程式混合技術來渲染前端。這是使用 Laravel 框架最常見的方式，且在我們看來，也是使用 Laravel 最有效率的方式。

如果您計劃以這種方式使用 Laravel，您可能會想查看我們關於[前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views)或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會有興趣了解 [Livewire](https://livewire.laravel.com) 和 [Inertia](https://inertiajs.com) 等社群套件。這些套件允許您將 Laravel 作為全端框架使用，同時還能享受單頁 JavaScript 應用程式所帶來的許多 UI 優勢。

如果您將 Laravel 用作全端框架，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 來編譯應用程式的 CSS 和 JavaScript。

> [!NOTE]
> 如果您想在建立應用程式時搶得先機，請參考我們官方的[應用程式入門套件](/docs/{{version}}/starter-kits)。


<a name="laravel-the-api-backend"></a>
### Laravel API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可以將 Laravel 用作 [Next.js](https://nextjs.org) 應用程式的 API 後端。在這種情境下，您可以使用 Laravel 為您的應用程式提供[認證](/docs/{{version}}/sanctum)以及資料儲存與檢索，同時還能利用 Laravel 的強大服務，例如佇列 (queues)、電子郵件、通知等。

如果您計劃以這種方式使用 Laravel，您可能會想查看我們關於[路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。