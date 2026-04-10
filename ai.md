# AI 輔助開發

 - [簡介](#introduction)
     - [為什麼選擇 Laravel 進行 AI 開發？](#why-laravel-for-ai-development)
 - [Laravel Boost](#laravel-boost)
     - [安裝](#installation)
     - [可用工具](#available-tools)
     - [AI 指南](#ai-guidelines)
     - [代理技能](#agent-skills)
     - [文件搜尋](#documentation-search)
     - [代理整合](#agents-integration)

<a name="introduction"></a>
## 簡介

Laravel 具有獨特的優勢，使其成為 AI 輔助與 AI 代理開發的最佳框架。隨著 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[OpenCode](https://opencode.ai)、[Cursor](https://cursor.com) 以及 [GitHub Copilot](https://github.com/features/copilot) 等 AI 程式碼代理的興起，開發者編寫程式碼的方式已發生轉變。這些工具能以空前速度生成完整功能、除錯複雜問題並重構程式碼 —— 但其成效很大程度上取決於它們對您程式碼庫的理解程度。


<a name="why-laravel-for-ai-development"></a>
### 為什麼選擇 Laravel 進行 AI 開發？

Laravel 既定的設計慣例與清晰的架構，使其成為 AI 輔助開發的理想框架。當您要求 AI 代理新增一個 controller 時，它能精確知道應該將其放置在何處；當您需要一個新的 migration 時，其命名慣例與檔案位置都是可預測的。這種一致性消除了 AI 工具在較靈活的框架中經常遇到的猜測問題。

除了檔案組織外，Laravel 具表現力的語法與詳盡的文件，為 AI 代理提供了生成準確且符合慣例之程式碼所需的上下文。像是 Eloquent 關係、表單請求(Form request) 以及中介層等功能，都遵循代理程式可以可靠理解並複製的模式。其結果是 AI 生成的程式碼看起來就像是由經驗豐富的 Laravel 開發者所撰寫，而非由泛用的 PHP 片段拼湊而成。


<a name="laravel-boost"></a>
## Laravel Boost

[Laravel Boost](https://github.com/laravel/boost) 填補了 AI 程式碼代理與您的 Laravel 應用程式之間的鴻溝。Boost 是一個 MCP (模型上下文協議) 伺服器，配備了 15 個以上的專用工具，能為 AI 代理提供對應用程式結構、資料庫、路由等深入的洞察。安裝 Boost 後，您的 AI 代理將從一個通用目的的程式碼助手，轉變為一名理解您特定應用程式的 Laravel 專家。

Boost 提供了三大核心能力：一套用於檢視與互動應用程式的 MCP 工具、專為 Laravel 生態系打造的可組合 AI 指南，以及一個包含超過 17,000 條 Laravel 專屬知識的強大文件 API。


<a name="installation"></a>
### 安裝

Boost 可安裝於執行 PHP 8.1 或更高版本的 Laravel 10, 11, 12, 以及 13 應用程式中。要開始使用，請將 Boost 安裝為開發依賴：

```shell
composer require laravel/boost --dev
```

安裝完成後，執行互動式安裝程式：

```shell
php artisan boost:install
```

安裝程式將自動偵測您的 IDE 與 AI 代理，讓您可以選擇適合您專案的整合選項。Boost 將生成必要的設定檔，例如適用於相容 MCP 編輯器的 `.mcp.json` 以及用於 AI 上下文的指南檔案。

> [!NOTE]
> 如果您希望每位開發者自行配置其環境，可以安全地將生成的設定檔（如 `.mcp.json`、`CLAUDE.md` 和 `boost.json`）加入到 `.gitignore` 中。


<a name="available-tools"></a>
### 可用工具

Boost 透過模型上下文協議向 AI 代理公開了一套全面的工具。這些工具允許代理深入理解並與您的 Laravel 應用程式互動：

<div class="content-list" markdown="1">

- **應用程式自我檢視** - 查詢您的 PHP 與 Laravel 版本、列出已安裝的套件，並檢視應用程式的設定與環境變數。
- **資料庫工具** - 檢視資料庫結構、執行唯讀查詢，且無需離開對話即可理解資料結構。
- **路由檢視** - 列出所有已註冊的路由及其對應的中介層、控制器與參數。
- **Artisan 指令** - 探索可用的 Artisan 指令及其引數，使代理能為您的任務建議並執行正確的指令。
- **日誌分析** - 讀取並分析應用程式的日誌檔案以協助除錯。
- **瀏覽器日誌** - 在使用 Laravel 前端工具開發時，存取瀏覽器主控台日誌與錯誤。
- **Tinker 整合** - 透過 Laravel Tinker 在應用程式上下文中執行 PHP 程式碼，允許代理測試假設並驗證行為。
- **文件搜尋** - 搜尋 Laravel 生態系文件，且結果會根據您安裝的套件版本量身打造。

</div>


<a name="ai-guidelines"></a>
### AI 指南

Boost 包含一套專為 Laravel 生態系精心設計的全面 AI 指南。這些指南教導 AI 代理如何撰寫符合 Laravel 慣例的程式碼、遵循框架慣例並避免常見陷阱。指南具有可組合性且具版本感知能力，這意味著代理會接收到適合您精確套件版本的指示。

指南可用於 Laravel 本身以及 Laravel 生態系中 16 個以上的套件，包括：

<div class="content-list" markdown="1">

- Livewire (2.x, 3.x, and 4.x)
- Inertia.js (React, Svelte, and Vue variants)
- Tailwind CSS (3.x and 4.x)
- Filament (3.x and 4.x)
- PHPUnit
- Pest PHP
- Laravel Pint
- 以及更多

</div>

當您執行 `boost:install` 時，Boost 會自動偵測您的應用程式使用了哪些套件，並將相關指南整合至您專案的 AI 上下文檔案中。


<a name="agent-skills"></a>
### 代理技能

[Agent Skills](https://agentskills.io/home) 是輕量且針對性的知識模組，代理在處理特定領域時可以根據需求啟動。與預先載入的指南不同，技能允許僅在相關時才載入詳細的模式與最佳實務，從而減少上下文膨脹並提高 AI 生成程式碼的相關性。

針對熱門的 Laravel 套件（如 Livewire, Inertia, Tailwind CSS, Pest 等）均提供相應的技能。當您執行 `boost:install` 並選擇技能功能時，系統會根據 `composer.json` 中偵測到的套件自動安裝相關技能。


<a name="documentation-search"></a>
### 文件搜尋

Boost 包含一個強大的文件 API，讓 AI 代理能存取超過 17,000 條 Laravel 生態系文件。與一般的網頁搜尋不同，這些文件經過索引、向量化並篩選，以匹配您的精確套件版本。

當代理需要理解某項功能如何運作時，它可以搜尋 Boost 的文件 API 並獲得準確且符合版本的資訊。這解決了 AI 代理經常建議已廢棄方法或舊版本框架語法常見問題。


<a name="agents-integration"></a>
### 代理整合

Boost 整合了支援模型上下文協議的熱門 IDE 與 AI 工具。有關 Cursor, Claude Code, Codex, Gemini CLI, GitHub Copilot 以及 Junie 的詳細設定指南，請參閱 Boost 文件的 [設定您的代理](/docs/{{version}}/boost#set-up-your-agents) 章節。