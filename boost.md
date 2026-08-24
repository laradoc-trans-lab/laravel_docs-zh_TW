# Laravel Boost

- [簡介](#introduction)
- [安裝](#installation)
    - [設定您的 AI 代理](#set-up-your-agents)
    - [保持 Boost 資源最新狀態](#keeping-boost-resources-updated)
- [MCP 伺服器](#mcp-server)
    - [可用的 MCP 工具](#available-mcp-tools)
    - [手動註冊 MCP 伺服器](#manually-registering-the-mcp-server)
- [AI 指南](#ai-guidelines)
    - [可用的 AI 指南](#available-ai-guidelines)
    - [新增自訂 AI 指南](#adding-custom-ai-guidelines)
    - [覆蓋 Boost AI 指南](#overriding-boost-ai-guidelines)
    - [第三方套件 AI 指南](#third-party-package-ai-guidelines)
- [AI 代理技能](#agent-skills)
    - [可用的技能](#available-skills)
    - [自訂技能](#custom-skills)
    - [覆蓋技能](#overriding-skills)
    - [第三方套件技能](#third-party-package-skills)
- [指南 vs. 技能](#guidelines-vs-skills)
- [專案規則](#project-rules)
    - [記錄規則](#recording-rules)
    - [推導您應用程式的慣例](#inferring-your-applications-conventions)
    - [停用專案規則](#disabling-project-rules)
- [說明文件 API](#documentation-api)
- [擴充 Boost](#extending-boost)
    - [新增支援其他 IDE / AI 代理](#adding-support-for-other-ides-ai-agents)

<a name="introduction"></a>
## 簡介

Laravel Boost 透過提供必要的指南與代理技能，加速 AI 輔助開發，幫助 AI 代理撰寫出符合 Laravel 最佳實踐的高品質 Laravel 應用程式。

Boost 還提供了一個強大的 Laravel 生態系統說明文件 API，它結合了內建的 MCP 工具與包含超過 17,000 筆 Laravel 專屬資訊的廣泛知識庫，全部透過 Embeddings 語意搜尋功能強化，以提供精準且具備上下文特性的結果。Boost 會指示像 Claude Code 與 Cursor 這樣的 AI 代理使用此 API 來學習最新版的 Laravel 功能與最佳實踐。


<a name="installation"></a>
## 安裝

Laravel Boost 可以透過 Composer 安裝：

```shell
composer require laravel/boost --dev
```

接下來，安裝 MCP 伺服器與程式碼撰寫指南：

```shell
php artisan boost:install
```

`boost:install` 命令會為您在安裝過程中所選擇的程式碼 AI 代理產生相關的代理指南與技能檔案。

安裝好 Laravel Boost 後，您就可以準備使用 Cursor、Claude Code 或您偏好的 AI 代理開始編寫程式碼了。

> [!NOTE]
> 您可以隨意將產生的 MCP 設定檔 (`.mcp.json`)、指南檔案 (`CLAUDE.md`、`AGENTS.md`、`junie/` 等) 以及 `boost.json` 設定檔加入到應用程式的 `.gitignore` 中，因為在執行 `boost:install` 和 `boost:update` 時，這些檔案都會自動重新產生。


<a name="set-up-your-agents"></a>
### 設定您的 AI 代理

```text tab=Cursor
1. Open the command palette (`Cmd+Shift+P` or `Ctrl+Shift+P`)
2. Press `enter` on "/open MCP Settings"
3. Turn the toggle on for `laravel-boost`
```

```text tab=Claude Code
Claude Code support is typically enabled automatically. If you find it isn't, open a shell in the project's directory and run the following command:

claude mcp add -s local -t stdio laravel-boost php artisan boost:mcp
```

```text tab=Codex
Codex support is typically enabled automatically. If you find it isn't, open a shell in the project's directory and run the following command:

codex mcp add laravel-boost -- php "artisan" "boost:mcp"
```

```text tab=Gemini CLI
Gemini CLI support is typically enabled automatically. If you find it isn't, open a shell in the project's directory and run the following command:

gemini mcp add -s project -t stdio laravel-boost php artisan boost:mcp
```

```text tab=GitHub Copilot (VS Code)
1. Open the command palette (`Cmd+Shift+P` or `Ctrl+Shift+P`)
2. Press `enter` on "MCP: List Servers"
3. Arrow to `laravel-boost` and press `enter`
4. Choose "Start server"
```

```text tab=Junie
1. Press `shift` twice to open the command palette
2. Search "MCP Settings" and press `enter`
3. Check the box next to `laravel-boost`
4. Click "Apply" at the bottom right
```


<a name="keeping-boost-resources-updated"></a>
### 保持 Boost 資源最新狀態

您可能需要定期更新本機的 Boost 資源 (AI 指南與技能)，以確保它們能反映您所安裝的 Laravel 生態系統套件之最新版本。若要這麼做，您可以使用 `boost:update` Artisan 命令。

```shell
php artisan boost:update
```

您也可以將其新增至 Composer 的 "post-update-cmd" 指令碼中，以自動化此流程：

```json
{
  "scripts": {
    "post-update-cmd": [
      "@php artisan boost:update --ansi"
    ]
  }
}
```

預設情況下，`boost:update` 命令僅會更新已經發布在您應用程式中的現有 Boost 資源。如果您希望 Boost 掃描您的應用程式以尋找任何新安裝的套件並提供發布其對應指南與技能的選項，您可以使用 `--discover` 選項：

```shell
php artisan boost:update --discover
```


<a name="mcp-server"></a>
## MCP 伺服器

Laravel Boost 提供了一個 MCP (Model Context Protocol) 伺服器，公開供 AI 代理與您的 Laravel 應用程式互動的工具。這些工具讓代理能夠檢視應用程式的架構、查詢資料庫、執行程式碼等等。


<a name="available-mcp-tools"></a>
### 可用的 MCP 工具

<div class="overflow-auto">

| 名稱                 | 備註                                                                                                       |
| -------------------- | ----------------------------------------------------------------------------------------------------------- |
| Application Info     | 讀取 PHP 與 Laravel 版本、資料庫引擎、附帶版本的生態系統套件清單以及 Eloquent 模型 |
| Browser Logs         | 讀取來自瀏覽器的紀錄與錯誤                                                                       |
| Database Connections | 檢視可用的資料庫連線，包含預設連線                                                                  |
| Database Query       | 對資料庫執行查詢                                                                        |
| Database Schema      | 讀取資料庫 Schema                                                                                    |
| Get Absolute URL     | 將相對路徑 URI 轉換為絕對路徑，以便代理產生有效的 URL                                        |
| Last Error           | 從應用程式的紀錄檔中讀取最後一個錯誤                                                        |
| Read Log Entries     | 讀取最後 N 筆紀錄項目                                                                                 |
| Record Rule          | 將持久的 [專案規則](#project-rules) 記錄至 `.ai/rules`，以便未來的代理可以繼承它                |
| Search Docs          | 查詢 Laravel 代管的說明文件 API 服務，以根據安裝的套件檢索說明文件    |

</div>


<a name="manually-registering-the-mcp-server"></a>
### 手動註冊 MCP 伺服器

有時候您可能需要手動在您選擇的編輯器中註冊 Laravel Boost MCP 伺服器。您應該使用以下詳細資訊註冊 MCP 伺服器：

<table>
<tr><td><strong>命令</strong></td><td><code>php</code></td></tr>
<tr><td><strong>引數</strong></td><td><code>artisan boost:mcp</code></td></tr>
</table>

JSON 範例：

```json
{
    "mcpServers": {
        "laravel-boost": {
            "command": "php",
            "args": ["artisan", "boost:mcp"]
        }
    }
}
```

<a name="ai-guidelines"></a>
## AI 指南

AI 指南是可組合的指令檔案，會預先載入，為 AI 代理提供有關 Laravel 生態系統套件的重要上下文。這些指南包含了核心慣例、最佳實踐以及特定框架的模式，能協助 AI 代理產生一致且高品質的程式碼。


<a name="available-ai-guidelines"></a>
### 可用的 AI 指南

Laravel Boost 包含了以下套件與框架的 AI 指南。`core` 指南為特定套件提供通用、泛化的 AI 建議，適用於所有版本。

<div class="overflow-auto">

| 套件 | 支援的版本 |
| ----------------- | ---------------------- |
| Core & Boost | core |
| Laravel Framework | core, 10.x, 11.x, 12.x, 13.x |
| Livewire | core, 2.x, 3.x, 4.x |
| Flux UI | core, free, pro |
| Folio | core |
| Herd | core |
| Inertia Laravel | core, 1.x, 2.x, 3.x |
| Inertia React | core, 1.x, 2.x, 3.x |
| Inertia Vue | core, 1.x, 2.x, 3.x |
| Inertia Svelte | core, 1.x, 2.x, 3.x |
| MCP | core |
| Pennant | core |
| Pest | core, 3.x, 4.x |
| PHPUnit | core |
| Pint | core |
| Sail | core |
| Tailwind CSS | core, 3.x, 4.x |
| Livewire Volt | core |
| Wayfinder | core |
| Enforce Tests | conditional |

</div>

> **注意：**若要保持您的 AI 指南處於最新狀態，請參閱[保持 Boost 資源最新狀態](#keeping-boost-resources-updated)章節。


<a name="adding-custom-ai-guidelines"></a>
### 新增自訂 AI 指南

若要使用您自訂的 AI 指南來擴充 Laravel Boost，請將 `.blade.php` 或 `.md` 檔案新增至您應用程式的 `.ai/guidelines/*` 目錄中。當您執行 `boost:install` 時，這些檔案將會自動包含在 Laravel Boost 的指南中。


<a name="overriding-boost-ai-guidelines"></a>
### 覆蓋 Boost AI 指南

您可以透過建立具備相同檔案路徑的自訂指南，來覆蓋 Boost 內建的 AI 指南。當您建立與現有 Boost 指南路徑相符的自訂指南時，Boost 會使用您的自訂版本而非內建版本。

例如，若要覆蓋 Boost 的「Inertia React v2 Form Guidance」指南，請在 `.ai/guidelines/inertia-react/2/forms.blade.php` 建立檔案。當您執行 `boost:install` 時，Boost 將會包含您的自訂指南，而非預設指南。


<a name="third-party-package-ai-guidelines"></a>
### 第三方套件 AI 指南

如果您維護了一個第三方套件，並希望 Boost 能包含該套件的 AI 指南，可以在您的套件中新增 `resources/boost/guidelines/core.blade.php` 檔案。當您的套件使用者執行 `php artisan boost:install` 時，Boost 便會自動載入您的指南。

AI 指南應該簡要概述您套件的功能、列出任何必要的檔案結構或慣例，並說明如何建立或使用其主要功能（附帶範例指令或程式碼片段）。請保持簡潔、具可操作性並專注於最佳實踐，以便 AI 能為您的使用者產生正確的程式碼。以下是一個範例：

```php
## Package Name

This package provides [brief description of functionality].

### Features

- Feature 1: [clear & short description].
- Feature 2: [clear & short description]. Example usage:

@verbatim
<code-snippet name="How to use Feature 2" lang="php">
$result = PackageName::featureTwo($param1, $param2);
</code-snippet>
@endverbatim
```


<a name="agent-skills"></a>
## AI 代理技能

[Agent Skills](https://agentskills.io/home)（AI 代理技能）是輕量級且具針對性的知識模組，AI 代理能在處理特定領域時按需啟用。與預先載入的指南不同，技能允許僅在相關時才載入詳細的模式與最佳實踐，從而減少上下文膨脹並提高 AI 所產生程式碼的相關性。

當您執行 `boost:install` 並選擇技能作為功能時，系統會根據在 `composer.json` 中偵測到的套件自動安裝技能。例如，如果您的專案包含 `livewire/livewire`，則會自動安裝 `livewire-development` 技能。而 Boost 內建的技能（例如 `infer-conventions`），無論您安裝了哪些套件都會被安裝。


<a name="available-skills"></a>
### 可用的技能

<div class="overflow-auto">

| 技能 | 套件 |
| -------------------------- | -------------- |
| fluxui-development | Flux UI |
| folio-routing | Folio |
| infer-conventions | Boost |
| inertia-react-development | Inertia React |
| inertia-svelte-development | Inertia Svelte |
| inertia-vue-development | Inertia Vue |
| livewire-development | Livewire |
| mcp-development | MCP |
| pennant-development | Pennant |
| pest-testing | Pest |
| tailwindcss-development | Tailwind CSS |
| volt-development | Volt |
| wayfinder-development | Wayfinder |

</div>

> **注意：**若要保持您的技能處於最新狀態，請參閱[保持 Boost 資源最新狀態](#keeping-boost-resources-updated)章節。


<a name="custom-skills"></a>
### 自訂技能

若要建立您自己的自訂技能，請在應用程式的 `.ai/skills/{skill-name}/` 目錄中新增一個 `SKILL.md` 檔案。當您執行 `boost:update` 時，您的自訂技能將會與 Boost 內建的技能一同安裝。

例如，要為應用程式的領域邏輯建立自訂技能：

```
.ai/skills/creating-invoices/SKILL.md
```


<a name="overriding-skills"></a>
### 覆蓋技能

您可以透過建立名稱相符的自訂技能來覆蓋 Boost 內建的技能。當您建立與現有 Boost 技能名稱相符的自訂技能時，Boost 會使用您的自訂版本而非內建版本。

例如，若要覆蓋 Boost 的 `livewire-development` 技能，請在 `.ai/skills/livewire-development/SKILL.md` 建立檔案。當您執行 `boost:update` 時，Boost 將會包含您的自訂技能，而非預設技能。


<a name="third-party-package-skills"></a>
### 第三方套件技能

如果您維護了一個第三方套件，並希望 Boost 能包含該套件的技能，可以在您的套件中新增 `resources/boost/skills/{skill-name}/SKILL.md` 檔案。當您的套件使用者執行 `php artisan boost:install` 時，Boost 便會根據使用者的偏好自動安裝您的技能。

Boost 技能支援 [Agent Skills 格式](https://agentskills.io/what-are-skills)，且結構應為包含 `SKILL.md` 檔案（帶有 YAML frontmatter 與 Markdown 指令）的資料夾。`SKILL.md` 檔案必須包含必要的 frontmatter（`name` 與 `description`），並可選擇性地包含腳本、範本與參考資料。

技能應該概述任何必要的檔案結構或慣例，並說明如何建立或使用其主要功能（附帶範例指令或程式碼片段）。請保持簡潔、具可操作性並專注於最佳實踐，以便 AI 能為您的使用者產生正確的程式碼：

```markdown
---
name: package-name-development
description: Build and work with PackageName features, including components and workflows.
---

# Package Name Development

## When to use this skill
Use this skill when working with PackageName features...

## Features

- Feature 1: [clear & short description].
- Feature 2: [clear & short description]. Example usage:

$result = PackageName::featureTwo($param1, $param2);
```

<a name="guidelines-vs-skills"></a>
## 指南 vs. 技能

Laravel Boost 提供兩種不同的方式來為 AI 代理提供有關您應用程式的上下文：**指南 (guidelines)** 與 **技能 (skills)**。

**指南**會在 AI 代理啟動時預先載入，提供關於 Laravel 慣例與最佳實踐的關鍵上下文，這些內容廣泛適用於您的程式碼庫。

**技能**則是在處理特定任務時按需啟動，包含特定領域的詳細模式（如 Livewire 元件或 Pest 測試）。僅在相關時載入技能可減少上下文膨脹並提高程式碼品質。

<div class="overflow-auto">

| 特點 | 指南 | 技能 |
| ----------- | --------------------------------- | -------------------------------- |
| **載入時機** | 預先載入，始終存在 | 按需載入，僅在相關時 |
| **適用範圍** | 廣泛且基礎 | 專注且針對特定任務 |
| **主要目的** | 核心慣例與最佳實踐 | 詳細的實作模式 |

</div>

指南與技能都是在描述 Laravel 生態系統。若要記錄您自己應用程式的慣例，您應該使用[專案規則](#project-rules)。


<a name="project-rules"></a>
## 專案規則

指南與技能是教導 AI 代理如何撰寫 Laravel，而專案規則則是教導它們如何撰寫您的應用程式。規則是指任何您需要在每個新會話中重新解釋的內容：

<div class="content-list" markdown="1">

- 您、您的 AI 代理或您的團隊成員在開發過程中做出的決策。
- 難以讓 AI 代理遵循的風格指南與偏好。
- 無法從周圍程式碼推導出來的陷阱與限制。

</div>

規則以 Markdown 檔案形式儲存在您應用程式的 `.ai/rules` 目錄中，且應該提交至版本控制系統。與 AI 代理個人且僅限於單次會話的記憶不同，您的規則會與您的團隊以及處理您應用程式的每個 AI 代理共享。

每個規則檔案都會在其 frontmatter 中宣告其適用的檔案比對模式 (globs)：

```markdown
---
paths:
  - app/Http/Controllers/**
---

# Http Controllers

## Extend BaseController for tenant scoping

All controllers must extend `App\Http\Controllers\BaseController`, which applies the
current tenant's query scope. Extending Laravel's base controller directly will leak
data across tenants.
```

此外，Boost 會維護一個 `.ai/rules/index.md` 檔案，該檔案將比對模式對應到其規則檔案。AI 代理會被指示在規劃或編輯任何檔案之前先查閱此索引，因此只有在相關時才會載入規則：

```markdown
# Project Rules Index

Before planning or editing, find the row whose globs match the file's path and read that rule file.

| Applies to | Rule file |
| --- | --- |
| app/Http/Controllers/** | .ai/rules/controllers.md |
| app/Models/** | .ai/rules/models.md |
```

> [!NOTE]
> 與 `.mcp.json` 及產生的指南檔案不同，`.ai/rules` 目錄應該提交至版本控制系統，以便與您的團隊共享這些規則。


<a name="recording-rules"></a>
### 記錄規則

要記錄一條規則，您只需要求您的 AI 代理記住它即可：

```text
Remember that all money values are stored as integer cents, never as floats.
```

AI 代理將會調用 Boost 的 `record-rule` MCP 工具，傳入 `glob`、簡短的 `title` 與 `note`。然後 Boost 會將該規則歸類在對應的區域下（必要時會建立規則檔案），並更新索引。

您應該始終使用 `record-rule` 工具來記錄規則，而不是手動建立規則檔案。Boost 會在記錄規則時重新產生 `.ai/rules/index.md`，而 AI 代理依賴該索引來找出哪些規則適用於它們正在處理的檔案。手動新增的規則檔案在下一次重新產生索引之前將不會被發現。


<a name="inferring-your-applications-conventions"></a>
### 推導您應用程式的慣例

逐一記錄規則對於未來的開發非常有效；然而，現有的應用程式已經包含多年的慣例累積。`infer-conventions` 技能可以從您已經撰寫的程式碼中引導並產生您的規則。要開始使用，請要求您的 AI 代理使用該技能：

```text
Use the infer-conventions skill
```

該技能將針對 Laravel 慣例的各個維度（包含驗證、控制器、授權、模型、架構、測試、前端、資料庫與主控台）全面掃描您的應用程式，接著對基礎類別 (base classes)、共用特徵 (shared traits) 和模組佈局等模式進行開放式的掃描。

該技能記錄的是您程式碼實際上的做法，而非應該怎麼做。它僅記錄具有充分支援且非預設的慣例，跳過框架預設值以及 Pint 或 Rector 已經強制的任何內容，並會回報真正混雜的模式而不是直接記錄它們。在寫入任何規則之前，該技能會呈現它所發現的每個慣例及其佐證，以供您確認。如果您希望該技能在未經確認的情況下記錄所有發現的慣例，您可以告訴它 "yolo"。


<a name="disabling-project-rules"></a>
### 停用專案規則

專案規則預設為啟用。要完全停用它們，請定義以下環境變數。這將移除 `record-rule` MCP 工具並停止 Boost 管理 `.ai/rules` 目錄：

```ini
BOOST_RULES_ENABLED=false
```


<a name="documentation-api"></a>
## 說明文件 API

Laravel Boost 包含一個說明文件 API，為 AI 代理提供存取包含超過 17,000 筆 Laravel 特定資訊的龐大知識庫。該 API 使用帶有嵌入向量 (embeddings) 的語意搜尋，以提供精準且感知上下文的結果。

`Search Docs` MCP 工具允許 AI 代理查詢 Laravel 託管的說明文件 API 服務，以根據您已安裝的套件檢索說明文件。Boost 的 AI 指南和技能會自動指示您的程式碼 AI 代理使用此 API。

<div class="overflow-auto">

| 套件           | 支援的版本 |
| ----------------- | ------------------ |
| Laravel Framework | 10.x, 11.x, 12.x, 13.x |
| Filament          | 2.x, 3.x, 4.x, 5.x |
| Flux UI           | 2.x Free, 2.x Pro  |
| Inertia           | 1.x, 2.x           |
| Livewire          | 1.x, 2.x, 3.x, 4.x |
| Nova              | 4.x, 5.x           |
| Pest              | 3.x, 4.x           |
| Tailwind CSS      | 3.x, 4.x           |

</div>


<a name="extending-boost"></a>
## 擴充 Boost

Boost 開箱即用，支援許多流行的 IDE 和 AI 代理。如果您的開發工具尚未獲得支援，您可以建立自己的 AI 代理並將其與 Boost 整合。


<a name="adding-support-for-other-ides-ai-agents"></a>
### 新增支援其他 IDE / AI 代理

要新增對新 IDE 或 AI 代理的支援，請建立一個繼承 `Laravel\Boost\Install\Agents\Agent` 的類別，並根據您的需求實作以下一個或多個契約 (Contracts)：

- `Laravel\Boost\Contracts\SupportsGuidelines` - 新增對 AI 指南的支援。
- `Laravel\Boost\Contracts\SupportsMcp` - 新增對 MCP 的支援。
- `Laravel\Boost\Contracts\SupportsSkills` - 新增對 AI 代理技能的支援。


<a name="writing-the-agent"></a>
#### 撰寫 AI 代理

```php
<?php

declare(strict_types=1);

namespace App;

use Laravel\Boost\Contracts\SupportsGuidelines;
use Laravel\Boost\Contracts\SupportsMcp;
use Laravel\Boost\Contracts\SupportsSkills;
use Laravel\Boost\Install\Agents\Agent;

class CustomAgent extends Agent implements SupportsGuidelines, SupportsMcp, SupportsSkills
{
    // Your implementation...
}
```

有關實作範例，請參閱 [ClaudeCode.php](https://github.com/laravel/boost/blob/main/src/Install/Agents/ClaudeCode.php)。


<a name="registering-the-agent"></a>
#### 註冊 AI 代理

在您應用程式的 `App\Providers\AppServiceProvider` 的 `boot` 方法中註冊您的自訂 AI 代理：

```php
use Laravel\Boost\Boost;

public function boot(): void
{
    Boost::registerAgent('customagent', CustomAgent::class);
}
```

註冊完成後，在執行 `php artisan boost:install` 時即可選擇您的 AI 代理。