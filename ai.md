# AI 輔助開發

 - [簡介](#introduction)
     - [為什麼選擇 Laravel 進行 AI 開發？](#why-laravel-for-ai-development)
 - [Laravel Boost](#laravel-boost)
     - [安裝](#installation)
     - [可用工具](#available-tools)
     - [AI 指南](#ai-guidelines)
     - [AI 代理技能(Agent Skills)](#agent-skills)
     - [文件搜尋](#documentation-search)
     - [AI 代理整合](#agents-integration)

<a name="introduction"></a>
## 簡介

Laravel 處於獨特的優勢地位，是進行 AI 輔助與 AI 代理開發的最佳框架。像是 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[OpenCode](https://opencode.ai)、[Cursor](https://cursor.com) 和 [GitHub Copilot](https://github.com/features/copilot) 等 AI 程式碼代理的崛起，徹底改變了開發者撰寫程式碼的方式。這些工具可以前所未有的速度生成完整功能、除錯複雜問題並重構程式碼——但它們的效果很大程度取決於對您程式碼庫的理解程度。


<a name="why-laravel-for-ai-development"></a>
### 為什麼選擇 Laravel 進行 AI 開發？

Laravel 的既定設計慣例與清晰架構，使其成為 AI 輔助開發的理想框架。當您要求 AI 代理新增控制器時，它精確知道該放在哪裡。當您需要新的遷移檔 (migration) 時，命名慣例和檔案位置都是可預測的。這種一致性消除了在更靈活的框架中經常困擾 AI 工具的盲目猜測。

除了檔案組織之外，Laravel 富有表現力的語法和完整的官方文件，為 AI 代理提供了生成準確且符合慣例的程式碼所需的上下文。像是 Eloquent 關聯、表單請求(Form request) 和中介層等功能，都遵循著代理可以可靠理解與複製的模式。其結果就是，AI 生成的程式碼看起來就像是由經驗豐富的 Laravel 開發者所撰寫，而非從通用的 PHP 程式碼片段拼湊而成。


<a name="laravel-boost"></a>
## Laravel Boost

[Laravel Boost](https://github.com/laravel/boost) 縮短了 AI 程式碼代理與您的 Laravel 應用程式之間的距離。Boost 是一個 MCP (Model Context Protocol) 伺服器，配備了 15 個以上的專用工具，能為 AI 代理提供深入了解您應用程式架構、資料庫、路由等資訊的能力。當您安裝 Boost 後，您的 AI 代理將從通用的程式碼助手轉變為了解您特定應用程式的 Laravel 專家。

Boost 提供三大核心能力：一套用於檢視與互動應用程式的 MCP 工具、專為 Laravel 生態系統打造的可組合 AI 指南，以及包含超過 17,000 條 Laravel 特定知識的強大文件 API。


<a name="installation"></a>
### 安裝

Boost 可以安裝在執行 PHP 8.1 或更高版本的 Laravel 10、11、12 與 13 應用程式中。要開始使用，請將 Boost 安裝為開發依賴項：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式將自動偵測您的 IDE 和 AI 代理，允許您選擇適合您專案的整合設定。Boost 會生成必要的設定檔，例如適用於支援 MCP 之編輯器的 `.mcp.json`，以及用於 AI 上下文的指南檔案。

> [!NOTE]
> 如果您希望每位開發者各自設定自己的環境，生成的設定檔（如 `.mcp.json`、`CLAUDE.md` 與 `boost.json`）可以安全地加入到您的 `.gitignore` 中。


<a name="available-tools"></a>
### 可用工具

Boost 透過模型上下文協議 (Model Context Protocol) 向 AI 代理提供一套完整的工具。這些工具允許代理深入了解您的 Laravel 應用程式並進行互動：

<div class="content-list" markdown="1">

- **應用程式自我檢視** - 查詢您的 PHP 與 Laravel 版本、列出已安裝的套件，並檢視應用程式的設定與環境變數。
- **資料庫工具** - 檢視您的資料庫結構 Schema、執行唯讀查詢，並在不離開對話的情況下理解您的資料結構。
- **路由檢視** - 列出所有已註冊的路由及其對應的中介層、控制器與參數。
- **Artisan 指令** - 探索可用的 Artisan 指令及其引數，使代理能夠為您的任務建議並執行正確的指令。
- **日誌分析** - 讀取並分析應用程式的日誌檔案，以協助排查除錯問題。
- **瀏覽器日誌** - 在使用 Laravel 的前端工具進行開發時，存取瀏覽器主控台日誌與錯誤。
- **Tinker 整合** - 透過 Laravel Tinker 在您的應用程式上下文中執行 PHP 程式碼，允許代理測試假設並驗證行為。
- **文件搜尋** - 搜尋 Laravel 生態系統文件，搜尋結果會根據您已安裝的套件版本量身打造。

</div>


<a name="ai-guidelines"></a>
### AI 指南

Boost 包含一套專為 Laravel 生態系統打造的完整 AI 指南。這些指南教導 AI 代理如何撰寫符合慣例的 Laravel 程式碼、遵循框架慣例並避免常見坑洞。指南具有可組合性且具備版本感知能力，這意味著代理會收到適合您確切套件版本的指示。

Laravel 本身以及 Laravel 生態系統中的 16 個以上套件皆提供指南，包含：

<div class="content-list" markdown="1">

- Livewire (2.x、3.x 與 4.x)
- Inertia.js (React、Svelte 與 Vue 變體)
- Tailwind CSS (3.x 與 4.x)
- Filament (3.x 與 4.x)
- PHPUnit
- Pest PHP
- Laravel Pint
- 以及更多套件

</div>

當您執行 `boost:install` 時，Boost 會自動偵測您的應用程式使用了哪些套件，並將相關指南組合到專案的 AI 上下文檔案中。


<a name="agent-skills"></a>
### AI 代理技能(Agent Skills)

[Agent Skills](https://agentskills.io/home) 是輕量級、具針對性的知識模組，代理可在處理特定領域時按需啟用。與預先載入的指南不同，技能允許僅在相關時才載入詳細的模式和最佳實踐，從而減少上下文膨脹並提升 AI 生成程式碼的相關性。

流行的 Laravel 套件（如 Livewire、Inertia、Tailwind CSS、Pest 等）皆提供技能支援。當您執行 `boost:install` 並選擇技能作為功能時，系統會根據 `composer.json` 中偵測到的套件自動安裝技能。


<a name="documentation-search"></a>
### 文件搜尋

Boost 包含一個強大的文件 API，讓 AI 代理能夠存取超過 17,000 條 Laravel 生態系統文件。與一般的網頁搜尋不同，這些文件經過索引、向量化和篩選，能精確符合您的套件版本。

當代理需要了解某個功能如何運作時，它可以搜尋 Boost 的文件 API 並獲得準確且特定版本的資訊。這消除了 AI 代理經常建議來自舊版框架之已棄用方法或語法的常見問題。


<a name="agents-integration"></a>
### AI 代理整合

Boost 與支援模型上下文協議 (Model Context Protocol) 的熱門 IDE 和 AI 工具整合。有關 Cursor、Claude Code、Codex、Gemini CLI、GitHub Copilot 與 Junie 的詳細設定說明，請參閱 Boost 文件的 [Set Up Your Agents](/docs/{{version}}/boost#set-up-your-agents) 章節。