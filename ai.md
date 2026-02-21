# AI 輔助開發

 - [介紹](#introduction)
     - [為什麼選擇 Laravel 進行 AI 開發？](#why-laravel-for-ai-development)
 - [Laravel Boost](#laravel-boost)
     - [安裝](#installation)
     - [可用工具](#available-tools)
     - [AI 指南](#ai-guidelines)
     - [Agent 技能](#agent-skills)
     - [文件搜尋](#documentation-search)
     - [Agents 整合](#agents-integration)

<a name="introduction"></a>
## 介紹

Laravel 處於獨特的優勢地位，是進行 AI 輔助與 Agent 開發的最佳框架。AI 編碼 Agent 如 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[OpenCode](https://opencode.ai)、[Cursor](https://cursor.com) 和 [GitHub Copilot](https://github.com/features/copilot) 的興起，改變了開發者編寫程式碼的方式。這些工具能以史無前例的速度生成完整功能、偵錯複雜問題並重構程式碼 —— 但其有效性在很大程度上取決於它們對您的程式碼庫的理解程度。


<a name="why-laravel-for-ai-development"></a>
### 為什麼選擇 Laravel 進行 AI 開發？

Laravel 具有主見的慣例與定義完善的結構，使其成為 AI 輔助開發的理想框架。當您要求 AI Agent 新增一個控制器時，它確切地知道該將其放在哪裡。當您需要一個新的遷移時，命名慣例和檔案位置都是可預測的。這種一致性消了在更靈活的框架中經常絆倒 AI 工具的猜測工作。

除了檔案組織之外，Laravel 富有表現力的語法和詳盡的文件為 AI Agent 提供了生成精確、道地的程式碼所需的上下文。像是 Eloquent 關聯、表單請求和中介層等功能，都遵循 Agent 可以可靠地理解並複製的模式。其結果是生成了看起來像是由資深的 Laravel 開發者編寫的 AI 程式碼，而非從通用的 PHP 片段拼湊而成。


<a name="laravel-boost"></a>
## Laravel Boost

[Laravel Boost](https://github.com/laravel/boost) 彌補了 AI 編碼 Agent 與您的 Laravel 應用程式之間的差距。Boost 是一個 MCP (Model Context Protocol) 伺服器，配備了超過 15 種專業工具，為 AI Agent 提供對您應用程式結構、資料庫、路由等內容的深入洞察。當您安裝 Boost 時，您的 AI Agent 會從一個通用型的程式碼助手轉變為了解您特定應用程式的 Laravel 專家。

Boost 提供了三大功能：一套用於檢查與互動應用程式的 MCP 工具、專為 Laravel 生態系統打造的可組合 AI 指南，以及一個包含超過 17,000 條 Laravel 專屬知識的強大文件 API。


<a name="installation"></a>
### 安裝

Boost 可安裝於執行 PHP 8.1 或更高版本的 Laravel 10、11 與 12 應用程式中。若要開始使用，請將 Boost 安裝為開發相依項目：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式將自動偵測您的 IDE 和 AI Agent，允許您選擇適合您專案的整合方案。Boost 將生成必要的設定檔，例如用於 MCP 相容編輯器的 `.mcp.json` 以及用於 AI 上下文的指南檔案。

> [!NOTE]
> 生成的設定檔如 `.mcp.json`、`CLAUDE.md` 和 `boost.json` 可以放心地加入到您的 `.gitignore` 中，如果您偏好每位開發者配置各自的環境。


<a name="available-tools"></a>
### 可用工具

Boost 透過 Model Context Protocol 向 AI Agent 公開了一套完整的工具。這些工具讓 Agent 能深入理解並與您的 Laravel 應用程式互動：

<div class="content-list" markdown="1">

- **應用程式內省 (Application Introspection)** - 查詢您的 PHP 與 Laravel 版本、列出已安裝的套件，並檢查應用程式的設定與環境變數。
- **資料庫工具** - 檢查您的資料庫結構、執行唯讀查詢，並在不離開對話的情況下了解您的資料結構。
- **路由檢查** - 列出所有已註冊的路由及其中介層、控制器和參數。
- **Artisan 指令** - 探索可用的 Artisan 指令及其參數，使 Agent 能夠針對您的任務建議並執行正確的指令。
- **記錄分析** - 讀取並分析應用程式的記錄檔，以協助偵錯問題。
- **瀏覽器記錄** - 在使用 Laravel 的前端工具開發時，存取瀏覽器主控台記錄與錯誤。
- **Tinker 整合** - 透過 Laravel Tinker 在應用程式的上下文中執行 PHP 程式碼，允許 Agent 測試假設並驗證行為。
- **文件搜尋** - 搜尋 Laravel 生態系統文件，並根據您安裝的套件版本提供量身定制的結果。

</div>


<a name="ai-guidelines"></a>
### AI 指南

Boost 包含一組完整的 AI 指南，專為 Laravel 生態系統打造。這些指南教導 AI Agent 如何編寫道地的 Laravel 程式碼、遵循框架慣例，並避免常見的陷阱。指南是可組合且具版本意識的，這意味著 Agent 會收到適用於您確切套件版本的指令。

這些指南適用於 Laravel 本身以及 Laravel 生態系統中的 16 個以上的套件，包括：

<div class="content-list" markdown="1">

- Livewire (2.x, 3.x, 與 4.x)
- Inertia.js (React 與 Vue 變體)
- Tailwind CSS (3.x 與 4.x)
- Filament (3.x 與 4.x)
- PHPUnit
- Pest PHP
- Laravel Pint
- 以及更多

</div>

當您執行 `boost:install` 時，Boost 會自動偵測您的應用程式使用的套件，並將相關指南組合到專案的 AI 上下文檔案中。


<a name="agent-skills"></a>
### Agent 技能

[Agent Skills](https://agentskills.io/home) 是輕量級、具針對性的知識模組，Agent 在處理特定領域時可以隨需啟用。與預先載入的指南不同，技能允許僅在相關時才載入詳細的模式和最佳實踐，從而減少上下文膨脹並提高 AI 生成程式碼的相關性。

這些技能適用於熱門的 Laravel 套件，如 Livewire、Inertia、Tailwind CSS、Pest 等。當您執行 `boost:install` 並選擇技能作為一項功能時，系統會根據從 `composer.json` 中偵測到的套件自動安裝技能。


<a name="documentation-search"></a>
### 文件搜尋

Boost 包含一個強大的文件 API，讓 AI Agent 可以存取超過 17,000 條 Laravel 生態系統文件。與通用的網路搜尋不同，這些文件經過索引、向量化和過濾，以符合您確切的套件版本。

當 Agent 需要了解某項功能如何運作時，它可以搜尋 Boost 的文件 API 並獲得準確且特定版本的資訊。這消除了 AI Agent 建議舊版框架中已棄用的方法或語法的常見問題。


<a name="agents-integration"></a>
### Agents 整合

Boost 與支援 Model Context Protocol 的熱門 IDE 和 AI 工具整合。有關 Cursor、Claude Code、Codex、Gemini CLI、GitHub Copilot 和 Junie 的詳細設定說明，請參閱 Boost 文件的 [設定您的 Agents](/docs/{{version}}/boost#set-up-your-agents) 章節。