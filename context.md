# 上下文

- [介紹](#introduction)
    - [運作方式](#how-it-works)
- [擷取上下文](#capturing-context)
    - [堆疊](#stacks)
- [檢索上下文](#retrieving-context)
    - [判斷項目是否存在](#determining-item-existence)
- [移除上下文](#removing-context)
- [隱藏上下文](#hidden-context)
- [事件](#events)
    - [脫水](#dehydrating)
    - [水合](#hydrated)

<a name="introduction"></a>
## 介紹

Laravel 的「上下文 (Context)」功能讓您能夠在應用程式中執行的請求 (requests)、工作 (jobs) 和指令 (commands) 中擷取、檢索並共用資訊。這些擷取的資訊也會包含在您的應用程式所寫入的日誌中，讓您更深入了解日誌項目寫入前發生的周邊程式碼執行歷史，並允許您追蹤分散式系統中的執行流程。

<a name="how-it-works"></a>
### 運作方式

了解 Laravel 的上下文 (Context) 功能的最佳方式是透過內建的日誌功能來實際操作。首先，您可以使用 `Context` Facade [將資訊新增到上下文中](#capturing-context)。在此範例中，我們將使用一個 [中介層 (middleware)](/docs/{{version}}/middleware) 在每個傳入的請求上，將請求 URL 和唯一的追蹤 ID 加入到上下文中：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class AddContext
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next): Response
    {
        Context::add('url', $request->url());
        Context::add('trace_id', Str::uuid()->toString());

        return $next($request);
    }
}
```

新增到上下文中的資訊會自動附加為中繼資料 (metadata) 到整個請求過程中寫入的任何 [日誌項目](/docs/{{version}}/logging)。將上下文作為中繼資料附加，可以區分傳遞給單個日誌項目的資訊與透過 `Context` 共用的資訊。例如，假設我們寫入以下日誌項目：

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

寫入的日誌將包含傳遞給日誌項目的 `auth_id`，但它也會包含上下文的 `url` 和 `trace_id` 作為中繼資料：

```
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

新增到上下文中的資訊也會提供給分派到佇列中的工作 (jobs)。例如，假設我們在將一些資訊新增到上下文後，分派一個 `ProcessPodcast` 工作 (job) 到佇列：

```php
// In our middleware...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// In our controller...
ProcessPodcast::dispatch($podcast);
```

當工作 (job) 被分派時，當前儲存在上下文中的任何資訊都會被擷取並與該工作 (job) 共用。然後將擷取的資訊重新注入到目前的上下文中，當工作 (job) 執行時。因此，如果我們工作 (job) 的 handle 方法要寫入日誌：

```php
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    // ...

    /**
     * Execute the job.
     */
    public function handle(): void
    {
        Log::info('Processing podcast.', [
            'podcast_id' => $this->podcast->id,
        ]);

        // ...
    }
}
```

產生的日誌項目將包含在最初分派該工作 (job) 的請求期間新增到上下文中的資訊：

```
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

儘管我們專注於 Laravel 上下文的內建日誌相關功能，但以下文件將說明上下文如何讓您在 HTTP 請求 / 佇列工作 (queued job) 邊界之間共用資訊，甚至是如何新增 [隱藏的上下文資料](#hidden-context)，這些資料不隨日誌項目寫入。

<a name="capturing-context"></a>
## 擷取上下文

您可以使用 `Context` Facade 的 `add` 方法將資訊儲存在目前的上下文中：

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

若要一次新增多個項目，您可以向 `add` 方法傳遞一個關聯陣列：

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

`add` 方法將覆寫任何具有相同鍵的現有值。如果您只希望在鍵不存在時才將資訊新增到上下文中，您可以使用 `addIf` 方法：

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

<a name="conditional-context"></a>
#### 條件式上下文

`when` 方法可用於根據給定條件將資料新增到上下文中。如果給定條件評估為 `true`，則會呼叫傳遞給 `when` 方法的第一個閉包，如果條件評估為 `false`，則會呼叫第二個閉包：

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Context;

Context::when(
    Auth::user()->isAdmin(),
    fn ($context) => $context->add('permissions', Auth::user()->permissions),
    fn ($context) => $context->add('permissions', []),
);
```

<a name="stacks"></a>
### 堆疊

Context 提供了建立「堆疊 (stacks)」的功能，這些堆疊是按新增順序儲存的資料列表。您可以透過呼叫 `push` 方法來將資訊新增到堆疊中：

```php
use Illuminate\Support\Facades\Context;

Context::push('breadcrumbs', 'first_value');

Context::push('breadcrumbs', 'second_value', 'third_value');

Context::get('breadcrumbs');
// [
//     'first_value',
//     'second_value',
//     'third_value',
// ]
```

堆疊可用於擷取請求的歷史資訊，例如在應用程式中發生的事件。例如，您可以建立一個事件監聽器，以便在每次執行查詢時將其推送到堆疊中，將查詢的 SQL 和持續時間作為一個元組 (tuple) 擷取：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

您可以使用 `stackContains` 和 `hiddenStackContains` 方法判斷一個值是否在堆疊中：

```php
if (Context::stackContains('breadcrumbs', 'first_value')) {
    //
}

if (Context::hiddenStackContains('secrets', 'first_value')) {
    //
}
```

`stackContains` 和 `hiddenStackContains` 方法也接受一個閉包作為它們的第二個引數，允許對值比較操作進行更多的控制：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;

return Context::stackContains('breadcrumbs', function ($value) {
    return Str::startsWith($value, 'query_');
});
```

<a name="retrieving-context"></a>
## 檢索上下文

您可以使用 `Context` Facade 的 `get` 方法從上下文中檢索資訊：

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only` 方法可用於檢索上下文中資訊的一個子集：

```php
$data = Context::only(['first_key', 'second_key']);
```

`pull` 方法可用於從上下文中檢索資訊並立即將其從上下文中移除：

```php
$value = Context::pull('key');
```

如果上下文資料儲存在 [堆疊](#stacks) 中，您可以使用 `pop` 方法從堆疊中彈出 (pop) 項目：

```php
Context::push('breadcrumbs', 'first_value', 'second_value');

Context::pop('breadcrumbs')
// second_value

Context::get('breadcrumbs');
// ['first_value'] 
```

如果您想檢索儲存在上下文中的所有資訊，您可以呼叫 `all` 方法：

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### 判斷項目是否存在

您可以使用 `has` 和 `missing` 方法判斷上下文是否為給定鍵儲存了任何值：

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}

if (Context::missing('key')) {
    // ...
}
```

`has` 方法將傳回 `true`，無論儲存的值為何。因此，例如，一個鍵值為 `null` 的鍵將被視為存在：

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## 移除上下文

`forget` 方法可用來從目前的上下文移除一個鍵及其值：

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

您可以透過向 `forget` 方法提供一個陣列，一次性移除多個鍵：

```php
Context::forget(['first_key', 'second_key']);
```


<a name="hidden-context"></a>
## 隱藏上下文

Context 提供了儲存「隱藏」資料的能力。這些隱藏資訊不會被附加到日誌中，也無法透過上述資料檢索方法存取。Context 提供了一組不同的方法來與隱藏上下文資訊互動：

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

這些「隱藏」方法的功能與上述非隱藏方法的功能相符：

```php
Context::addHidden(/* ... */);
Context::addHiddenIf(/* ... */);
Context::pushHidden(/* ... */);
Context::getHidden(/* ... */);
Context::pullHidden(/* ... */);
Context::popHidden(/* ... */);
Context::onlyHidden(/* ... */);
Context::allHidden(/* ... */);
Context::hasHidden(/* ... */);
Context::forgetHidden(/* ... */);
```


<a name="events"></a>
## 事件

Context 觸發兩個事件，讓您可以掛接到上下文的水合 (hydration) 與脫水 (dehydration) 過程。

為了說明這些事件的用途，想像在應用程式的一個中介層中，您根據傳入 HTTP 請求的 `Accept-Language` Header 設定 `app.locale` 設定值。Context 的事件讓您可以在請求期間擷取此值，並在佇列上還原它，確保在佇列上傳送的通知具有正確的 `app.locale` 值。我們可以使用 Context 的事件和 [隱藏上下文](#hidden-context) 資料來實現這一點，這將在以下文件中說明。


<a name="dehydrating"></a>
### 脫水

每當一個 job 被分派到佇列時，Context 中的資料會被「脫水」並與 job 的 payload 一同擷取。`Context::dehydrating` 方法允許您註冊一個閉包，該閉包將在脫水過程中被調用。在此閉包中，您可以修改將與排入佇列的 job 共享的資料。

通常，您應該在應用程式 `AppServiceProvider` 類的 `boot` 方法中註冊 `dehydrating` 回呼：

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Context::dehydrating(function (Repository $context) {
        $context->addHidden('locale', Config::get('app.locale'));
    });
}
```

> [!NOTE]  
> 您不應在 `dehydrating` 回呼中使用 `Context` Facade，因為這會改變目前處理程序的上下文。請確保您只對傳遞給回呼的 repository 進行更改。


<a name="hydrated"></a>
### 水合

每當一個排入佇列的 job 開始在佇列上執行時，任何與該 job 共享的上下文都將被「水合」回目前的上下文。`Context::hydrated` 方法允許您註冊一個閉包，該閉包將在水合過程中被調用。

通常，您應該在應用程式 `AppServiceProvider` 類的 `boot` 方法中註冊 `hydrated` 回呼：

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Context::hydrated(function (Repository $context) {
        if ($context->hasHidden('locale')) {
            Config::set('app.locale', $context->getHidden('locale'));
        }
    });
}
```

> [!NOTE]  
> 您不應在 `hydrated` 回呼中使用 `Context` Facade，而應確保您只對傳遞給回呼的 repository 進行更改。