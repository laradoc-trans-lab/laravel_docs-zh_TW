# 版本說明

- [版本命名方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 13](#laravel-13)

<a name="versioning-scheme"></a>
## 版本命名方案

Laravel 及其其他第一方套件遵循 [Semantic Versioning](https://semver.org)。框架的主版本每年發布一次（約在第一季），而次要版本和修補版本則可能每週發布。次要版本和修補版本**絕不**應包含破壞性變更。

當在您的應用程式或套件中引用 Laravel 框架或其元件時，您應該始終使用版本限制（例如 `^13.0`），因為 Laravel 的主版本確實包含破壞性變更。然而，我們致力於確保您能在一天或更短的時間內更新到新的主版本。


<a name="named-arguments"></a>
#### 命名引數

[命名引數 (Named arguments)](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不在 Laravel 的向後相容性指南範圍內。為了改進 Laravel 的程式碼庫，我們可能會在必要時選擇重新命名函式引數。因此，在呼叫 Laravel 方法時使用命名引數應謹慎行之，並理解參數名稱在未來可能會發生變更。


<a name="support-policy"></a>
## 支援政策

對於所有 Laravel 版本，錯誤修正將提供 18 個月的支援，而安全性修正則提供 2 年。對於所有其他函式庫，只有最新的主版本會收到錯誤修正。此外，請查看 [Laravel 支援的](/docs/{{version}}/database#introduction) 資料庫版本。

<div class="overflow-auto">

| 版本 | PHP (*)   | 發布日期           | 錯誤修正至         | 安全性修正至       |
| ------- |-----------| ------------------- | ------------------- | -------------------- |
| 10      | 8.1 - 8.3 | February 14th, 2023 | August 6th, 2024    | February 4th, 2025   |
| 11      | 8.2 - 8.4 | March 12th, 2024    | September 3rd, 2025 | March 12th, 2026     |
| 12      | 8.2 - 8.5 | February 24th, 2025 | August 13th, 2026   | February 24th, 2027  |
| 13      | 8.3 - 8.5 | March 17th, 2026    | Q3 2027             | March 17th, 2028     |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>生命週期結束</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅提供安全性修正</div>
    </div>
</div>

(*) 支援的 PHP 版本

<a name="laravel-13"></a>
## Laravel 13

Laravel 13 延續了 Laravel 的年度發行節奏，重點在於 AI 原生工作流、更強大的預設值以及更具表達力的開發者 API。此版本包含了第一方的 AI 基礎元件、JSON:API 資源、語義 / 向量搜尋能力，以及在佇列、快取和安全性方面的漸進式改進。


<a name="minimal-breaking-changes"></a>
### Minimal Breaking Changes

在這次的發行週期中，我們很大一部分的重心在於最小化破壞性變更。相反地，我們致力於在一年之中持續交付不會破壞現有應用程式的品質生活 (quality-of-life) 改進。

因此，就工作量而言，Laravel 13 的版本升級相對較小，但仍提供了實質的新功能。基於此，大多數 Laravel 應用程式可以在不修改太多應用程式程式碼的情況下升級到 Laravel 13。


<a name="php-8"></a>
### PHP 8.3

Laravel 13.x 要求最低 PHP 版本為 8.3。


<a name="ai-sdk"></a>
### Laravel AI SDK

Laravel 13 引入了第一方的 [Laravel AI SDK](https://laravel.com/ai)，為文字生成、工具呼叫代理 (tool-calling agents)、嵌入 (embeddings)、音訊、圖像以及向量儲存整合提供統一的 API。

透過 AI SDK，您可以在保持一致且原生的 Laravel 開發體驗的同時，構建與供應商無關 (provider-agnostic) 的 AI 功能。

例如，可以使用單次呼叫來提示一個基礎代理：

```php
use App\Ai\Agents\SalesCoach;

$response = SalesCoach::make()->prompt('Analyze this sales transcript...');

return (string) $response;
```

Laravel AI SDK 也可以生成圖像、音訊和嵌入：

對於視覺生成案例，SDK 提供了一個簡潔的 API，可透過自然語言提示建立圖像：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')->generate();

$rawContent = (string) $image;
```

對於語音體驗，您可以將文字合成自然 sounding 的音訊，用於助手、旁白和無障礙功能：

```php
use Laravel\Ai\Audio;

$audio = Audio::of('I love coding with Laravel.')->generate();

$rawContent = (string) $audio;
```

而對於語義搜尋和檢索工作流，您可以直接從字串生成嵌入：

```php
use Illuminate\Support\Str;

$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings();
```


<a name="json-api"></a>
### JSON:API Resources

Laravel 現在包含了第一方的 [JSON:API 資源](/docs/{{version}}/eloquent-resources#jsonapi-resources)，讓回傳符合 JSON:API 規範的回應變得簡單直接。

JSON:API 資源處理資源物件的序列化、關聯包含 (relationship inclusion)、稀疏欄位集 (sparse fieldsets)、連結以及符合 JSON:API 規範的回應標頭。


<a name="request-forgery-protection"></a>
### Request Forgery Protection

為了安全性，Laravel 的 [請求偽造保護](/docs/{{version}}/csrf#preventing-csrf-requests) 中介層已得到增強，並正式命名為 `PreventRequestForgery`，在保留與基於令牌 (token-based) 的 CSRF 保護相容性的同時，增加了對來源感知 (origin-aware) 的請求驗證。


<a name="queue-routing"></a>
### Queue Routing

Laravel 13 透過 `Queue::route(...)` 增加了 [依類別定義的佇列路由](/docs/{{version}}/queues#queue-routing)，允許您在一個中心位置為特定作業 (jobs) 定義預設的佇列 / 連線路由規則：

```php
Queue::route(ProcessPodcast::class, connection: 'redis', queue: 'podcasts');
```


<a name="php-attributes"></a>
### Expanded PHP Attributes

Laravel 13 繼續擴展框架中對第一方 PHP 屬性 (PHP attribute) 的支援，使常見的配置和行為考量更具宣告性，且能與您的類別和方法共存 (colocated)。

值得注意的新增項目包括控制器和授權屬性，例如 [`#[Middleware]`](/docs/{{version}}/controllers#controller-middleware) 和 [`#[Authorize]`](/docs/{{version}}/controllers#authorization-attributes)，以及面向佇列的作業控制，例如 [`#[Tries]`](/docs/{{version}}/queues#max-job-attempts-and-timeout)、[`#[Backoff]`](/docs/{{version}}/queues#dealing-with-failed-jobs)、[`#[Timeout]`](/docs/{{version}}/queues#max-job-attempts-and-timeout) 以及 [`#[FailOnTimeout]`](/docs/{{version}}/queues#failing-on-timeout)。

例如，現在可以直接在類別和方法上宣告控制器中介層和策略 (policy) 檢查：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Comment;
use App\Models\Post;
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Illuminate\Routing\Attributes\Controllers\Middleware;

#[Middleware('auth')]
class CommentController
{
    #[Middleware('subscribed')]
    #[Authorize('create', [Comment::class, 'post'])]
    public function store(Post $post)
    {
        // ...
    }
}
```

Eloquent、事件、通知、驗證、測試以及資源序列化 API 中也引入了額外的屬性，在框架的更多領域為您提供一致的屬性優先 (attribute-first) 選項。


<a name="cache-touch"></a>
### Cache TTL Extension

Laravel 現在包含了 [`Cache::touch(...)`](/docs/{{version}}/cache)，讓您可以在不檢索並重新儲存值的情況下，延伸現有快取項目的 TTL。


<a name="semantic-search"></a>
### Semantic / Vector Search

Laravel 13 透過原生的向量查詢支援、嵌入工作流以及相關 API 深化了其語義搜尋的功能，相關文件請參閱 [搜尋](/docs/{{version}}/search#semantic-vector-search)、[查詢](/docs/{{version}}/queries#vector-similarity-clauses) 以及 [AI SDK](/docs/{{version}}/ai-sdk#embeddings)。

這些功能讓您能簡單地利用 PostgreSQL + `pgvector` 構建 AI 驅動的搜尋體驗，包括針對直接從字串生成的嵌入進行相似度搜尋。

例如，您可以直接從查詢構建器 (query builder) 執行語義相似度搜尋：

```php
$documents = DB::table('documents')
    ->whereVectorSimilarTo('embedding', 'Best wineries in Napa Valley')
    ->limit(10)
    ->get();
```