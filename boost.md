# Laravel Boost

- [簡介](#introduction)
- [安裝](#installation)
    - [設定您的 AI 代理](#set-up-your-agents)
    - [保持 Boost 資源更新](#keeping-boost-resources-updated)
- [MCP 伺服器](#mcp-server)
    - [可用的 MCP 工具](#available-mcp-tools)
    - [手動註冊 MCP 伺服器](#manually-registering-the-mcp-server)
- [AI 指南](#ai-guidelines)
    - [可用的 AI 指南](#available-ai-guidelines)
    - [新增自定義 AI 指南](#adding-custom-ai-guidelines)
    - [覆寫 Boost AI 指南](#overriding-boost-ai-guidelines)
    - [第三方套件 AI 指南](#third-party-package-ai-guidelines)
- [AI 代理技能(Agent Skills)](#agent-skills)
    - [可用的技能](#available-skills)
    - [自定義技能](#custom-skills)
    - [覆寫技能](#overriding-skills)
    - [第三方套件技能](#third-party-package-skills)
- [指南 vs. 技能](#guidelines-vs-skills)
- [文件 API](#documentation-api)
- [擴展 Boost](#extending-boost)
    - [新增對其他 IDE / AI 代理的支援](#adding-support-for-other-ides-ai-agents)

<a name="introduction"></a>
## 簡介

Laravel Boost 透過提供必要的指南與 AI 代理技能(Agent Skills)，協助 AI 代理撰寫遵循 Laravel 最佳實踐的高品質 Laravel 應用程式，從而加速 AI 輔助開發。

Boost 還提供了一個強大的 Laravel 生態系文件 API，該 API 將內建的 MCP 工具與包含超過 17,000 條 Laravel 特定資訊的龐大知識庫相結合，並透過使用嵌入 (embeddings) 的語義搜尋能力來提供精確且感知上下文的結果。Boost 會指示像是 Claude Code 和 Cursor 等 AI 代理使用此 API 來學習最新的 Laravel 功能與最佳實踐。


<a name="installation"></a>
## 安裝

Laravel Boost 可以透過 Composer 安裝：

```shell
composer require laravel/boost --dev
```

接下來，安裝 MCP 伺服器與編碼指南：

```shell
php artisan boost:install
```

`boost:install` 指令將為您在安裝過程中選擇的編碼代理產生相關的代理指南與技能檔案。

一旦安裝好 Laravel Boost，您就可以開始使用 Cursor、Claude Code 或您選擇的 AI 代理進行開發。

> [!NOTE]
> 您可以將產生的 MCP 設定檔 (`.mcp.json`)、指南檔案 (`CLAUDE.md`、`AGENTS.md`、`junie/` 等) 以及 `boost.json` 設定檔加入到應用程式的 `.gitignore` 中，因為這些檔案在執行 `boost:install` 與 `boost:update` 時會自動重新產生。


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
### 保持 Boost 資源更新

您可能希望定期更新您的本地 Boost 資源 (AI 指南與技能)，以確保它們反映了您安裝的 Laravel 生態系套件的最新版本。若要執行此操作，您可以使用 `boost:update` Artisan 指令。

```shell
php artisan boost:update
```

您也可以將其加入到 Composer 的 "post-update-cmd" 腳本中來自動化此過程：

```json
{
  "scripts": {
    "post-update-cmd": [
      "@php artisan boost:update --ansi"
    ]
  }
}
```

預設情況下，`boost:update` 指令僅會更新已在應用程式中發布的現有 Boost 資源。如果您希望 Boost 掃描應用程式中是否有任何新安裝的套件，並提議發布其對應的指南與技能，您可以使用 `--discover` 選項：

```shell
php artisan boost:update --discover
```


<a name="mcp-server"></a>
## MCP 伺服器

Laravel Boost 提供了一個 MCP (Model Context Protocol) 伺服器，提供讓 AI 代理與您的 Laravel 應用程式互動的工具。這些工具讓 AI 代理能夠檢查應用程式結構、查詢資料庫、執行程式碼等。


<a name="available-mcp-tools"></a>
### 可用的 MCP 工具

| 名稱 | 備註 |
| -------------------- | ----------------------------------------------------------------------------------------------------------- |
| Application Info | 讀取 PHP 與 Laravel 版本、資料庫引擎、包含版本的生態系套件列表以及 Eloquent 模型 |
| Browser Logs | 讀取來自瀏覽器的日誌與錯誤 |
| Database Connections | 檢查可用的資料庫連線，包括預設連線 |
| Database Query | 對資料庫執行查詢 |
| Database Schema | 讀取資料庫結構 |
| Get Absolute URL | 將相對路徑 URI 轉換為絕對路徑，以便 AI 代理產生有效的 URL |
| Last Error | 從應用程式的日誌檔案中讀取最後一個錯誤 |
| Read Log Entries | 讀取最後 N 條日誌條目 |
| Search Docs | 查詢 Laravel 託管的文件 API 服務，根據已安裝的套件獲取文件 |


<a name="manually-registering-the-mcp-server"></a>
### 手動註冊 MCP 伺服器

有時您可能需要手動將 Laravel Boost MCP 伺服器註冊到您選擇的編輯器中。您應該使用以下詳細資訊來註冊 MCP 伺服器：

<table>
<tr><td><strong>指令</strong></td><td><code>php</code></td></tr>
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

AI 指南是可組合的指令檔案，會在啟動時預先載入，以提供 AI 代理關於 Laravel 生態系套件的必要上下文。這些指南包含核心慣例、最佳實踐以及框架特定的模式，能幫助 AI 代理產生一致且高品質的程式碼。


<a name="available-ai-guidelines"></a>
### 可用的 AI 指南

Laravel Boost 為以下套件和框架提供了 AI 指南。`core` 指南為給定套件提供通用的建議，適用於所有版本。

| 套件           | 支援版本     |
| ----------------- | ---------------------- |
| Core & Boost      | core                   |
| Laravel Framework | core, 10.x, 11.x, 12.x, 13.x |
| Livewire          | core, 2.x, 3.x, 4.x    |
| Flux UI           | core, free, pro        |
| Folio             | core                   |
| Herd              | core                   |
| Inertia Laravel   | core, 1.x, 2.x, 3.x    |
| Inertia React     | core, 1.x, 2.x, 3.x    |
| Inertia Vue       | core, 1.x, 2.x, 3.x    |
| Inertia Svelte    | core, 1.x, 2.x, 3.x    |
| MCP               | core                   |
| Pennant           | core                   |
| Pest              | core, 3.x, 4.x         |
| PHPUnit           | core                   |
| Pint              | core                   |
| Sail              | core                   |
| Tailwind CSS      | core, 3.x, 4.x         |
| Livewire Volt     | core                   |
| Wayfinder         | core                   |
| Enforce Tests     | conditional            |

> **注意：** 若要保持您的 AI 指南為最新狀態，請參閱 [保持 Boost 資源更新](#keeping-boost-resources-updated) 區塊。


<a name="adding-custom-ai-guidelines"></a>
### 新增自定義 AI 指南

若要使用自定義的 AI 指南來增強 Laravel Boost，請將 `.blade.php` 或 `.md` 檔案新增至應用程式的 `.ai/guidelines/*` 目錄中。當您執行 `boost:install` 時，這些檔案將會自動包含在 Laravel Boost 的指南中。


<a name="overriding-boost-ai-guidelines"></a>
### 覆寫 Boost AI 指南

您可以透過建立路徑相同的自定義指南來覆寫 Boost 內建的 AI 指南。當您建立的自定義指南路徑與現有的 Boost 指南路徑相符時，Boost 將使用您的自定義版本而非內建版本。

例如，若要覆寫 Boost 的 "Inertia React v2 Form Guidance" 指南，請在 `.ai/guidelines/inertia-react/2/forms.blade.php` 建立檔案。當您執行 `boost:install` 時，Boost 將包含您的自定義指南而非預設指南。


<a name="third-party-package-ai-guidelines"></a>
### 第三方套件 AI 指南

如果您維護一個第三方套件並希望 Boost 為其提供 AI 指南，您可以透過在套件中新增 `resources/boost/guidelines/core.blade.php` 檔案來達成。當您的套件使用者執行 `php artisan boost:install` 時，Boost 將會自動載入您的指南。

AI 指南應簡要概述您的套件功能、列出所需的檔案結構或慣例，並說明如何建立或使用其主要功能（附上範例指令或程式碼片段）。請保持內容簡潔、具可操作性並專注於最佳實踐，以便 AI 能為您的使用者產生正確的程式碼。以下是一個範例：

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
## AI 代理技能(Agent Skills)

[AI 代理技能(Agent Skills)](https://agentskills.io/home) 是輕量且具針對性的知識模組，AI 代理在處理特定領域時可以根據需求啟用。與預先載入的指南不同，技能允許僅在相關時才載入詳細的模式和最佳實踐，從而減少上下文膨脹並提高 AI 產生程式碼的相關性。

當您執行 `boost:install` 並選擇將技能作為功能時，系統會根據 `composer.json` 中偵測到的套件自動安裝技能。例如，如果您的專案包含 `livewire/livewire`，則會自動安裝 `livewire-development` 技能。


<a name="available-skills"></a>
### 可用的技能

| 技能                      | 套件        |
| -------------------------- | -------------- |
| fluxui-development         | Flux UI        |
| folio-routing              | Folio          |
| inertia-react-development  | Inertia React  |
| inertia-svelte-development | Inertia Svelte |
| inertia-vue-development    | Inertia Vue    |
| livewire-development       | Livewire       |
| mcp-development            | MCP            |
| pennant-development        | Pennant        |
| pest-testing               | Pest           |
| tailwindcss-development    | Tailwind CSS   |
| volt-development           | Volt           |
| wayfinder-development      | Wayfinder      |

> **注意：** 若要保持您的技能為最新狀態，請參閱 [保持 Boost 資源更新](#keeping-boost-resources-updated) 區塊。


<a name="custom-skills"></a>
### 自定義技能

若要建立自定義技能，請在應用程式的 `.ai/skills/{skill-name}/` 目錄中新增 `SKILL.md` 檔案。當您執行 `boost:update` 時，您的自定義技能將與 Boost 的內建技能一同安裝。

例如，若要為應用程式的領域邏輯建立自定義技能：

```
.ai/skills/creating-invoices/SKILL.md
```


<a name="overriding-skills"></a>
### 覆寫技能

您可以透過建立名稱相同的自定義技能來覆寫 Boost 的內建技能。當您建立的自定義技能名稱與現有的 Boost 技能名稱相符時，Boost 將使用您的自定義版本而非內建版本。

例如，若要覆寫 Boost 的 `livewire-development` 技能，請在 `.ai/skills/livewire-development/SKILL.md` 建立檔案。當您執行 `boost:update` 時，Boost 將包含您的自定義技能而非預設技能。


<a name="third-party-package-skills"></a>
### 第三方套件技能

如果您維護一個第三方套件並希望 Boost 為其提供技能，您可以透過在套件中新增 `resources/boost/skills/{skill-name}/SKILL.md` 檔案來達成。當您的套件使用者執行 `php artisan boost:install` 時，Boost 將根據使用者偏好自動安裝您的技能。

Boost 技能支援 [Agent Skills 格式](https://agentskills.io/what-are-skills)，且結構應為一個包含 `SKILL.md` 檔案的資料夾，該檔案需包含 YAML 前置資料 (frontmatter) 和 Markdown 指令。`SKILL.md` 檔案必須包含必要的前置資料 (`name` 和 `description`)，並可選擇性地包含指令碼、範本和參考資料。

技能應概述任何所需的檔案結構或慣例，並說明如何建立或使用其主要功能（附上範例指令或程式碼片段）。請保持內容簡潔、具可操作性並專注於最佳實踐，以便 AI 能為您的使用者產生正確的程式碼：

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

Laravel Boost 提供了兩種截然不同的方式來為 AI 代理提供關於您應用程式的上下文：**指南 (guidelines)** 與 **技能 (skills)**。

**指南** 在 AI 代理啟動時會先行載入，提供關於 Laravel 慣例與最佳實務的關鍵上下文，這些內容廣泛適用於您的整個程式碼庫。

**技能** 則是在執行特定任務時根據需求啟動，包含針對特定領域（例如 Livewire 元件或 Pest 測試）的詳細模式。僅在相關時載入技能可減少上下文膨脹並提高程式碼品質。

| 項目      | 指南                        | 技能                           |
| ----------- | --------------------------------- | -------------------------------- |
| **載入時機**  | 先行載入，始終存在           | 按需載入，相關時啟動         |
| **範圍**   | 廣泛且基礎               | 集中且針對特定任務           |
| **目的** | 核心慣例與最佳實務 | 詳細的實作模式 |


<a name="documentation-api"></a>
## 文件 API

Laravel Boost 包含一個文件 API，可讓 AI 代理存取一個包含超過 17,000 條 Laravel 特定資訊的龐大知識庫。該 API 使用帶有嵌入 (embeddings) 的語義搜尋，以提供精確且感知上下文的結果。

`Search Docs` MCP 工具允許 AI 代理查詢 Laravel 託管的文件 API 服務，以根據您安裝的套件檢索文件。Boost 的 AI 指南與技能會自動指示您的開發代理使用此 API。

| 套件           | 支援版本 |
| ----------------- | ------------------ |
| Laravel Framework | 10.x, 11.x, 12.x, 13.x |
| Filament          | 2.x, 3.x, 4.x, 5.x |
| Flux UI           | 2.x Free, 2.x Pro  |
| Inertia           | 1.x, 2.x           |
| Livewire          | 1.x, 2.x, 3.x, 4.x |
| Nova              | 4.x, 5.x           |
| Pest              | 3.x, 4.x           |
| Tailwind CSS      | 3.x, 4.x           |


<a name="extending-boost"></a>
## 擴展 Boost

Boost 開箱即用，支援許多熱門的 IDE 與 AI 代理。如果您的開發工具尚未獲得支援，您可以建立自己的代理並將其與 Boost 整合。


<a name="adding-support-for-other-ides-ai-agents"></a>
### 新增對其他 IDE / AI 代理的支援

若要新增對新 IDE 或 AI 代理的支援，請建立一個繼承 `Laravel\Boost\Install\Agents\Agent` 的類別，並根據您的需求實作以下一個或多個契約(Contracts)：

- `Laravel\Boost\Contracts\SupportsGuidelines` - 新增對 AI 指南的支援。
- `Laravel\Boost\Contracts\SupportsMcp` - 新增對 MCP 的支援。
- `Laravel\Boost\Contracts\SupportsSkills` - 新增對 AI 代理技能(Agent Skills)的支援。


<a name="writing-the-agent"></a>
#### 撰寫代理

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

如需實作範例，請參閱 [ClaudeCode.php](https://github.com/laravel/boost/blob/main/src/Install/Agents/ClaudeCode.php)。


<a name="registering-the-agent"></a>
#### 註冊代理

在您應用程式的 `App\Providers\AppServiceProvider` 的 `boot` 方法中註冊您的自定義代理：

```php
use Laravel\Boost\Boost;

public function boot(): void
{
    Boost::registerAgent('customagent', CustomAgent::class);
}
```

註冊完成後，在執行 `php artisan boost:install` 時，您就可以選擇該代理。