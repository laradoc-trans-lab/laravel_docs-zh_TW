# 安裝

- [認識 Laravel](#meet-laravel)
    - [為什麼選擇 Laravel？](#why-laravel)
- [建立 Laravel 應用程式](#creating-a-laravel-project)
    - [使用 AI 開始入手](#getting-started-using-ai)
    - [安裝 PHP 與 Laravel 安裝程式](#installing-php)
    - [建立應用程式](#creating-an-application)
- [初始設定](#initial-configuration)
    - [基於環境的設定](#environment-based-configuration)
    - [資料庫與 Migration](#databases-and-migrations)
    - [目錄設定](#directory-configuration)
- [使用 Herd 安裝](#installation-using-herd)
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

Laravel 是一個具有展現力、優雅語法的 Web 應用程式框架。Web 框架為建立應用程式提供了一個架構與起點，讓你能專注於創造驚豔的作品，而細節則交由我們來處理。

Laravel 致力於提供極佳的開發者體驗，同時提供強大的功能，例如完善的依賴注入、具展現力的資料庫抽象層、佇列與排程任務、單元與整合測試等等。

無論你是剛接觸 PHP Web 框架的新手，還是擁有多年經驗的專家，Laravel 都是一個能陪伴你一同成長的框架。我們會協助你邁出成為 Web 開發者的第一步，或是在你將專業技能提升到全新境界時給予助力。我們迫不及待想看到你所開發出的成果。


<a name="why-laravel"></a>
### 為什麼選擇 Laravel？

在建立 Web 應用程式時，你有許多工具和框架可以選擇。然而，我們相信 Laravel 是建立現代化全端 Web 應用程式的最佳選擇。


#### 漸進式框架

我們喜歡稱 Laravel 為「漸進式（progressive）」框架。意思是說，Laravel 會跟著你一起成長。如果你才剛踏入 Web 開發領域，Laravel 豐富的文件庫、指南和[教學影片](https://laracasts.com)將會幫助你快速上手，而不會感到無所適從。

如果你是資深開發者，Laravel 為你提供了強大的工具，用於[依賴注入](/docs/{{version}}/container)、[單元測試](/docs/{{version}}/testing)、[佇列](/docs/{{version}}/queues)、[即時事件](/docs/{{version}}/broadcasting)等。Laravel 經過精細調校，專門用於建立專業的 Web 應用程式，並已準備好處理企業級的工作負載。


#### 可擴充的框架

Laravel 具有極佳的擴充性。得益於 PHP 易於擴充的特性，以及 Laravel 對 Redis 等快速分散式快取系統的內建支援，使用 Laravel 進行水平擴充輕而易舉。事實上，Laravel 應用程式已經能輕鬆擴充以處理每月數億次的請求。

需要極致的擴充能力嗎？像 [Laravel Cloud](https://cloud.laravel.com) 這樣的平台允許你以幾乎無限的規模運行你的 Laravel 應用程式。


#### 支援 AI 代理的框架

Laravel 既定的設定慣例與清晰架構，使其成為使用 Cursor 和 Claude Code 等工具進行 [AI 輔助開發](/docs/{{version}}/ai) 的理想框架。當你要求 AI 代理新增一個控制器時，它能精確知道該放在哪裡。當你需要新的 Migration 時，命名慣例與檔案位置都是可預測的。這種一致性消除了在更靈活的框架中經常使 AI 工具受阻的猜測過程。

除了檔案組織結構外，Laravel 具展現力的語法和完整的文件，也為 AI 代理提供了生成準確且符合慣例程式碼所需的上下文。像是 Eloquent 關聯、表單請求 (Form request) 和中介層等功能，都遵循著 AI 代理能可靠理解並複製的模式。其結果就是生成出來的程式碼就像是由經驗豐富的 Laravel 開發者所撰寫，而非從通用的 PHP 程式碼片段拼湊而成。

若要進一步瞭解為什麼 Laravel 是 AI 輔助開發的完美選擇，請參考我們關於 [AI 代理開發](/docs/{{version}}/ai) 的文件。


#### 社群導向的框架

Laravel 結合了 PHP 生態系統中最優質的套件，提供最穩健且對開發者友善的框架。此外，來自世界各地數以千計的優秀開發者已經[為此框架做出了貢獻](https://github.com/laravel/framework)。誰知道呢，也許有一天你也會成為 Laravel 的貢獻者。


<a name="creating-a-laravel-project"></a>
## 建立 Laravel 應用程式


<a name="getting-started-using-ai"></a>
### 使用 AI 開始入手

如果你正在使用像是 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 或 [OpenCode](https://opencode.ai) 這類的 AI 程式碼代理，你可以在它碰觸你的專案之前，使用提示詞為該代理提供一份針對 Laravel 的操作指南。

下方提示詞會告訴代理在哪裡可以找到 Laravel 的安裝指南、優先考慮事項，以及當你尚未做出選擇時如何設定合理的預設值。將此內容貼入你的代理中即可開始：

```text
I'm building a new Laravel application.

Fetch and follow the instructions from https://laravel.com/for/agents. Treat the returned Markdown as the source of truth for how to install and set up Laravel in this session.
```

在代理讀取說明之後，它應該會一步步引導你，並確保設定與 Laravel 的預設值保持一致。


<a name="installing-php"></a>
### 安裝 PHP 與 Laravel 安裝程式

在建立你的第一個 Laravel 應用程式之前，請確保你的本機電腦已安裝 [PHP](https://php.net)、[Composer](https://getcomposer.org) 與 [Laravel 安裝程式](https://github.com/laravel/installer)。此外，你還應該安裝 [Node 與 NPM](https://nodejs.org) 或 [Bun](https://bun.sh/)，以便編譯應用程式的前端靜態資源。

如果你的本機電腦尚未安裝 PHP 和 Composer，以下指令可在 macOS、Windows 或 Linux 上安裝 PHP、Composer 及 Laravel 安裝程式：

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

執行上述其中一個指令後，你應該重啟終端機工作階段。透過 `php.new` 安裝後，若要更新 PHP、Composer 與 Laravel 安裝程式，你可以重新在終端機中執行該指令。

如果你已經安裝了 PHP 和 Composer，則可以透過 Composer 來安裝 Laravel 安裝程式：

```shell
composer global require laravel/installer
```

> [!NOTE]
> 若想體驗功能完整且具備圖形介面的 PHP 安裝與管理體驗，請參考 [Laravel Herd](#installation-using-herd)。


<a name="creating-an-application"></a>
### 建立應用程式

安裝好 PHP、Composer 與 Laravel 安裝程式後，你就準備好建立新的 Laravel 應用程式了：

```shell
laravel new example-app
```

應用程式建立完成後，你可以使用 `dev` Composer 腳本來啟動 Laravel 的本機開發伺服器、佇列工作行程 (queue worker) 以及 Vite 開發伺服器：

```shell
cd example-app
npm install && npm run build
composer run dev
```

啟動開發伺服器後，你可以透過網頁瀏覽器造訪 [http://localhost:8000](http://localhost:8000) 來存取你的應用程式。接下來，你就可以準備[踏出進入 Laravel 生態系統的後續步驟](#next-steps)。當然，你可能也想[設定資料庫](#databases-and-migrations)並執行必要的 Migration。

> [!NOTE]
> 如果希望在開發 Laravel 應用程式時能有一個良好的開端，可以考慮使用我們的 [入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件為新的 Laravel 應用程式提供了後端與前端的認證鷹架。

<a name="initial-configuration"></a>
## 初始設定

Laravel 框架的所有設定檔都儲存在 `config` 目錄中。每個選項都有詳細說明，因此你可以隨意瀏覽這些檔案並熟悉可用的選項。

Laravel 開箱即用，幾乎不需要額外設定。你可以直接開始開發！不過，你可能想要檢視 `config/app.php` 檔案及其文件說明。它包含了一些你可能想根據應用程式需求進行修改的選項，例如 `url` 和 `locale`。


<a name="environment-based-configuration"></a>
### 基於環境的設定

由於 Laravel 的許多設定選項值可能會根據你的應用程式是運行在本地電腦還是正式營運的 Web 伺服器上而有所不同，因此許多重要的設定值都是透過位於應用程式根目錄下的 `.env` 檔案來定義的。

你的 `.env` 檔案不應該提交到應用程式的版本控制中，因為使用你應用程式的每個開發人員或伺服器都可能需要不同的環境設定。此外，如果入侵者取得你的版本控制儲存庫存取權限，這將會帶來安全風險，因為任何敏感的憑證都會遭到洩露。

> [!NOTE]
> 有關 `.env` 檔案和基於環境設定的更多資訊，請參閱完整的[設定文件](/docs/{{version}}/configuration#environment-configuration)。


<a name="databases-and-migrations"></a>
### 資料庫與 Migration

現在你已經建立了 Laravel 應用程式，可能想要將一些資料儲存到資料庫中。預設情況下，應用程式的 `.env` 設定檔指定 Laravel 將與 SQLite 資料庫進行互動。

在建立應用程式的過程中，Laravel 已為你建立了 `database/database.sqlite` 檔案，並執行了必要的 Migration 來建立應用程式的資料庫資料表。

如果你傾向使用其他資料庫驅動程式（例如 MySQL 或 PostgreSQL），你可以更新 `.env` 設定檔以使用相應的資料庫。例如，如果你想使用 MySQL，請像這樣更新 `.env` 設定檔中的 `DB_*` 變數：

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

如果你選擇使用 SQLite 以外的資料庫，你需要建立該資料庫並執行應用程式的[資料庫 Migration](/docs/{{version}}/migrations)：

```shell
php artisan migrate
```

> [!NOTE]
> 如果你在 macOS 或 Windows 上進行開發，且需要於本地安裝 MySQL、PostgreSQL 或 Redis，可以考慮使用 [Herd Pro](https://herd.laravel.com/#plans) 或 [DBngin](https://dbngin.com/)。


<a name="directory-configuration"></a>
### 目錄設定

Laravel 應該始終從為你的 Web 伺服器所設定的「Web 目錄」根目錄提供服務。你不應該嘗試從「Web 目錄」的子目錄提供 Laravel 應用程式服務。嘗試這麼做可能會曝光應用程式內部存在的敏感檔案。


<a name="installation-using-herd"></a>
## 使用 Herd 安裝

[Laravel Herd](https://herd.laravel.com) 是一個專為 macOS 與 Windows 設計且極速的原生 Laravel 和 PHP 開發環境。Herd 包含了開始進行 Laravel 開發所需的一切，包含 PHP 和 Nginx。

一旦安裝好 Herd，你就準備好可以使用 Laravel 開始開發了。Herd 包含了適用於 `php`、`composer`、`laravel`、`expose`、`node`、`npm` 和 `nvm` 的命令列工具。

> [!NOTE]
> [Herd Pro](https://herd.laravel.com/#plans) 為 Herd 增添了額外的強大功能，例如建立與管理本地 MySQL、Postgres 與 Redis 資料庫的能力，以及本地郵件檢視與日誌監控。


<a name="herd-on-macos"></a>
### macOS 上的 Herd

如果你在 macOS 上開發，可以從 [Herd 網站](https://herd.laravel.com) 下載 Herd 安裝程式。安裝程式會自動下載最新版本的 PHP，並將你的 Mac 設定為始終在背景執行 [Nginx](https://www.nginx.com/)。

macOS 版的 Herd 使用 [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) 來支援「Parked (停泊)」目錄。任何位於 Parked 目錄中的 Laravel 應用程式都將由 Herd 自動提供服務。預設情況下，Herd 會在 `~/Herd` 建立一個 Parked 目錄，你可以透過資料庫目錄名稱並加上 `.test` 網域來存取該目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立全新 Laravel 應用程式最快的方式是使用 Herd 隨附的 Laravel CLI：

```shell
cd ~/Herd
laravel new my-app
cd my-app
herd open
```

當然，你隨時可以透過 Herd 的 UI 管理你的 Parked 目錄與其他 PHP 設定，你可以從系統列中的 Herd 選單開啟它。

你可以透過參閱 [Herd 文件](https://herd.laravel.com/docs)瞭解更多關於 Herd 的資訊。


<a name="herd-on-windows"></a>
### Windows 上的 Herd

你可以在 [Herd 網站](https://herd.laravel.com/windows) 下載適用於 Windows 的 Herd 安裝程式。安裝完成後，你可以啟動 Herd 以完成引導程序，並首次存取 Herd UI。

只要左鍵點擊 Herd 的系統列圖示即可存取 Herd UI。按右鍵則會開啟快速選單，方便存取你日常所需的所有工具。

在安裝過程中，Herd 會在你的家目錄 `%USERPROFILE%\Herd` 建立一個「Parked」目錄。任何位於 Parked 目錄中的 Laravel 應用程式都將由 Herd 自動提供服務，你可以透過其目錄名稱在 `.test` 網域存取該目錄中的任何 Laravel 應用程式。

安裝 Herd 後，建立全新 Laravel 應用程式最快的方式是使用 Herd 隨附的 Laravel CLI。若要開始，請開啟 PowerShell 並執行以下命令：

```shell
cd ~\Herd
laravel new my-app
cd my-app
herd open
```

你可以透過參閱 [Windows 版 Herd 文件](https://herd.laravel.com/docs/windows)瞭解更多關於 Herd 的資訊。


<a name="ide-support"></a>
## IDE 支援

開發 Laravel 應用程式時，你可以自由選擇使用任何程式碼編輯器。[Laravel LSP](https://github.com/laravel/lsp) 提供可感知框架的編輯器支援，包括自動補全、懸停資訊、診斷、文件連結、跳轉至定義以及對 Laravel 和 Blade 程式碼的快速修正。

要安裝 Laravel LSP，請透過 Composer 進行全域安裝。請確保 Composer 的全域 vendor bin 目錄已包含在你的 `PATH` 中：

```shell
composer global require laravel/lsp
```

如果你正在尋找輕量且具可擴充性的編輯器，[VS Code](https://code.visualstudio.com) 或 [Cursor](https://cursor.com) 結合官方的 [Laravel VS Code 擴充套件](https://marketplace.visualstudio.com/items?itemName=laravel.vscode-laravel)，可提供語法高亮、程式碼片段 (Snippets)、Artisan 命令整合以及自動 Laravel LSP 支援。官方 Laravel 擴充套件也適用於 [Sublime Text](https://github.com/laravel/sublime-extension) 和 [Zed](https://github.com/laravel/zed-extension)。關於其他相容語言伺服器 (Language Server) 的編輯器（包括 Neovim 和 OpenCode）的設定說明，請參閱 [Laravel LSP 儲存庫](https://github.com/laravel/lsp)。

對於全面且強大的 Laravel 支援，可以參考 JetBrains 開發的 IDE [PhpStorm](https://www.jetbrains.com/phpstorm/laravel/?utm_source=laravel.com&utm_medium=link&utm_campaign=laravel-2025&utm_content=partner&ref=laravel-2025)。PhpStorm 內建的 Laravel 框架支援包含 Blade 模板、針對 Eloquent Model、路由、View、翻譯與組件的智慧自動完成，以及強大的程式碼生成與跨 Laravel 專案導覽功能。

對於尋求雲端開發體驗的人來說，[Firebase Studio](https://firebase.studio/) 讓你能在瀏覽器中直接即時體驗建置 Laravel。完全無需設定，Firebase Studio 讓你輕鬆從任何裝置開始建置 Laravel 應用程式。

<a name="laravel-and-ai"></a>
## Laravel 與 AI

[Laravel Boost](https://github.com/laravel/boost) 是一款強大的工具，能架起 AI 程式代理與 Laravel 應用程式之間的橋樑。Boost 為 AI 代理提供了特定於 Laravel 的上下文、工具與準則，使其能夠產生更精確、符合特定版本且遵循 Laravel 慣例的程式碼。

當您在 Laravel 應用程式中安裝 Boost 後，AI 代理將能使用超過 15 種專用工具，包含了解您正在使用的套件、查詢您的資料庫、搜尋 Laravel 文件、讀取瀏覽器日誌、產生測試以及透過 Tinker 執行程式碼的能力。

此外，Boost 還為 AI 代理提供了超過 17,000 篇向量化的 Laravel 生態系統文件，並且專門針對您所安裝的套件版本。這意味著 AI 代理可以針對您專案所使用的確切版本提供目標明確的指導。

Boost 還包含由 Laravel 維護的 AI 準則，有助於 AI 代理遵循框架慣例、撰寫適當的測試，並在產生 Laravel 程式碼時避免常見的陷阱。


<a name="installing-laravel-boost"></a>
### 安裝 Laravel Boost

Boost 可以安裝在執行 PHP 8.1 或更高版本的 Laravel 10、11、12 與 13 應用程式中。首先，請將 Boost 安裝為開發依賴項：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式會自動偵測您的 IDE 與 AI 代理，讓您可以選擇啟用適合您專案的功能。Boost 會尊重現有的專案慣例，預設情況下不會強制實施既定風格規則。

> [!NOTE]
> 若要進一步了解 Boost，請參閱 [GitHub 上的 Laravel Boost 儲存庫](https://github.com/laravel/boost)。


<a name="adding-custom-ai-guidelines"></a>
#### 新增自訂 AI 準則

若要使用您自己的自訂 AI 準則來擴充 Laravel Boost，請將 `.blade.php` 或 `.md` 檔案新增至應用程式的 `.ai/guidelines/*` 目錄中。當您執行 `boost:install` 時，這些檔案將會自動包含在 Laravel Boost 的準則中。


<a name="next-steps"></a>
## 後續步驟

現在您已經建立了 Laravel 應用程式，您可能會想知道接下來該學習什麼。首先，我們強烈建議透過閱讀以下文件來熟悉 Laravel 的運作方式：

<div class="content-list" markdown="1">

- [請求生命週期](/docs/{{version}}/lifecycle)
- [設定](/docs/{{version}}/configuration)
- [目錄結構](/docs/{{version}}/structure)
- [前端](/docs/{{version}}/frontend)
- [服務容器](/docs/{{version}}/container)
- [Facades](/docs/{{version}}/facades)

</div>

您打算如何使用 Laravel 也將決定您旅程中的下一步。使用 Laravel 的方式有多種，我們將在下方探討該框架的兩個主要使用場景。


<a name="laravel-the-fullstack-framework"></a>
### Laravel 作為全端框架

Laravel 可以作為全端框架使用。所謂「全端」框架，意思是您將使用 Laravel 來將請求路由到您的應用程式，並透過 [Blade 模板](/docs/{{version}}/blade) 或類似 [Inertia](https://inertiajs.com) 的單頁應用程式混合技術來渲染前端。這是使用 Laravel 框架最常見的方式，也是我們認為最具生產力的 Laravel 使用方式。

如果您打算以此方式使用 Laravel，可以參考我們關於[前端開發](/docs/{{version}}/frontend)、[路由](/docs/{{version}}/routing)、[視圖](/docs/{{version}}/views) 或 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。此外，您可能也會對了解社群套件如 [Livewire](https://livewire.laravel.com) 與 [Inertia](https://inertiajs.com) 感興趣。這些套件允許您將 Laravel 作為全端框架使用的同時，也能享有單頁 JavaScript 應用程式所提供的許多 UI 優勢。

如果您將 Laravel 作為全端框架使用，我們也強烈建議您學習如何使用 [Vite](/docs/{{version}}/vite) 來編譯應用程式的 CSS 與 JavaScript。

> [!NOTE]
> 如果您想在建置應用程式時搶占先機，請參考我們官方的[入門套件](/docs/{{version}}/starter-kits)。


<a name="laravel-the-api-backend"></a>
### Laravel 作為 API 後端

Laravel 也可以作為 JavaScript 單頁應用程式或行動應用程式的 API 後端。例如，您可以將 Laravel 作為 [Next.js](https://nextjs.org) 應用程式的 API 後端。在這個情境下，您可以使用 Laravel 為應用程式提供 [認證](/docs/{{version}}/sanctum) 以及資料儲存／讀取，同時利用 Laravel 強大的服務，例如佇列、郵件、通知等。

如果您打算以此方式使用 Laravel，可以參考我們關於[路由](/docs/{{version}}/routing)、[Laravel Sanctum](/docs/{{version}}/sanctum) 以及 [Eloquent ORM](/docs/{{version}}/eloquent) 的文件。