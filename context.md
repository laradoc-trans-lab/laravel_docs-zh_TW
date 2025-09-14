# Context

- [簡介](#introduction)
    - [運作方式](#how-it-works)
- [擷取 Context](#capturing-context)
    - [堆疊](#stacks)
- [擷取 Context](#retrieving-context)
    - [判斷項目是否存在](#determining-item-existence)
- [移除 Context](#removing-context)
- [隱藏 Context](#hidden-context)
- [事件](#events)
    - [脫水 (Dehydrating)](#dehydrating)
    - [水合 (Hydrated)](#hydrated)

<a name="introduction"></a>
## 簡介

Laravel 的「context」功能讓您能夠在應用程式中執行的請求、任務 (jobs) 和命令中擷取、擷取和共享資訊。這些擷取的資訊也會包含在應用程式寫入的日誌中，讓您更深入地了解日誌條目寫入之前發生的周圍程式碼執行歷史，並允許您追蹤分散式系統中的執行流程。

<a name="how-it-works"></a>
### 運作方式

了解 Laravel context 功能的最佳方式是透過內建的日誌功能來實際操作。首先，您可以使用 `Context` facade [將資訊新增到 context](#capturing-context)。在此範例中，我們將使用 [middleware](/docs/{{version}}/middleware) 在每個傳入請求中將請求 URL 和唯一的追蹤 ID 新增到 context：

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

新增到 context 的資訊會自動作為中繼資料附加到在整個請求中寫入的任何 [日誌條目](/docs/{{version}}/logging)。將 context 作為中繼資料附加允許將傳遞給個別日誌條目的資訊與透過 `Context` 共享的資訊區分開來。例如，假設我們寫入以下日誌條目：

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

寫入的日誌將包含傳遞給日誌條目的 `auth_id`，但它也將包含 context 的 `url` 和 `trace_id` 作為中繼資料：

```text
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

新增到 context 的資訊也可用於分派到佇列的任務。例如，假設我們在將一些資訊新增到 context 後分派一個 `ProcessPodcast` 任務到佇列：

```php
// 在我們的 middleware 中...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// 在我們的 controller 中...
ProcessPodcast::dispatch($podcast);
```

當任務被分派時，目前儲存在 context 中的任何資訊都會被擷取並與任務共享。然後，在任務執行時，擷取的資訊會被水合回目前的 context。因此，如果我們的任務的 handle 方法要寫入日誌：

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

產生的日誌條目將包含在最初分派任務的請求期間新增到 context 的資訊：

```text
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

儘管我們專注於 Laravel context 的內建日誌相關功能，但以下文件將說明 context 如何讓您在 HTTP 請求/佇列任務邊界之間共享資訊，甚至如何新增 [隱藏的 context 資料](#hidden-context) 不會隨日誌條目寫入。

<a name="capturing-context"></a>
## 擷取 Context

您可以使用 `Context` facade 的 `add` 方法將資訊儲存在目前的 context 中：

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

若要一次新增多個項目，您可以將關聯陣列傳遞給 `add` 方法：

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

`add` 方法將覆寫任何共享相同鍵的現有值。如果您只想在鍵尚不存在時將資訊新增到 context，您可以使用 `addIf` 方法：

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

Context 還提供了方便的方法來遞增或遞減給定鍵。這兩種方法都至少接受一個參數：要追蹤的鍵。可以提供第二個參數來指定鍵應該遞增或遞減的數量：

```php
Context::increment('records_added');
Context::increment('records_added', 5);

Context::decrement('records_added');
Context::decrement('records_added', 5);
```

<a name="conditional-context"></a>
#### 條件式 Context

`when` 方法可用於根據給定條件將資料新增到 context。提供給 `when` 方法的第一個閉包將在給定條件評估為 `true` 時被調用，而第二個閉包將在條件評估為 `false` 時被調用：

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Context;

Context::when(
    Auth::user()->isAdmin(),
    fn ($context) => $context->add('permissions', Auth::user()->permissions),
    fn ($context) => $context->add('permissions', []),
);
```

<a name="scoped-context"></a>
#### 作用域 Context

`scope` 方法提供了一種在給定回呼執行期間暫時修改 context 的方法，並在回呼執行完成時將 context 恢復到其原始狀態。此外，您可以在閉包執行時傳遞應合併到 context 中的額外資料（作為第二個和第三個參數）。

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\Log;

Context::add('trace_id', 'abc-999');
Context::addHidden('user_id', 123);

Context::scope(
    function () {
        Context::add('action', 'adding_friend');

        $userId = Context::getHidden('user_id');

        Log::debug("Adding user [{$userId}] to friends list.");
        // Adding user [987] to friends list.  {"trace_id":"abc-999","user_name":"taylor_otwell","action":"adding_friend"}
    },
    data: ['user_name' => 'taylor_otwell'],
    hidden: ['user_id' => 987],
);

Context::all();
// [
//     'trace_id' => 'abc-999',
// ]

Context::allHidden();
// [
//     'user_id' => 123,
// ]
```

> [!WARNING]
> 如果 context 中的物件在作用域閉包內被修改，該變動將反映在作用域之外。

<a name="stacks"></a>
### 堆疊

Context 提供了建立「堆疊」的能力，堆疊是按新增順序儲存的資料列表。您可以透過調用 `push` 方法將資訊新增到堆疊中：

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

堆疊對於擷取請求的歷史資訊很有用，例如應用程式中發生的事件。例如，您可以建立一個事件監聽器，以便在每次執行查詢時推送到堆疊，將查詢 SQL 和持續時間作為元組擷取：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

// 在 AppServiceProvider.php 中...
DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

您可以使用 `stackContains` 和 `hiddenStackContains` 方法判斷值是否在堆疊中：

```php
if (Context::stackContains('breadcrumbs', 'first_value')) {
    //
}

if (Context::hiddenStackContains('secrets', 'first_value')) {
    //
}
```

`stackContains` 和 `hiddenStackContains` 方法也接受閉包作為它們的第二個參數，允許對值比較操作進行更多控制：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;

return Context::stackContains('breadcrumbs', function ($value) {
    return Str::startsWith($value, 'query_');
});
```

<a name="retrieving-context"></a>
## 擷取 Context

您可以使用 `Context` facade 的 `get` 方法從 context 中擷取資訊：

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only` 和 `except` 方法可用於擷取 context 中資訊的子集：

```php
$data = Context::only(['first_key', 'second_key']);

$data = Context::except(['first_key']);
```

`pull` 方法可用於從 context 中擷取資訊並立即從 context 中移除它：

```php
$value = Context::pull('key');
```

如果 context 資料儲存在 [堆疊](#stacks) 中，您可以使用 `pop` 方法從堆疊中彈出項目：

```php
Context::push('breadcrumbs', 'first_value', 'second_value');

Context::pop('breadcrumbs');
// second_value

Context::get('breadcrumbs');
// ['first_value']
```

`remember` 和 `rememberHidden` 方法可用於從 context 中擷取資訊，同時如果請求的資訊不存在，則將 context 值設定為給定閉包返回的值：

```php
$permissions = Context::remember(
    'user-permissions',
    fn () => $user->permissions,
);
```

如果您想擷取儲存在 context 中的所有資訊，您可以調用 `all` 方法：

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### 判斷項目是否存在

您可以使用 `has` 和 `missing` 方法來判斷 context 是否為給定鍵儲存了任何值：

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}

if (Context::missing('key')) {
    // ...
}
```

`has` 方法將返回 `true`，無論儲存的值是什麼。因此，例如，一個值為 `null` 的鍵將被視為存在：

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## 移除 Context

`forget` 方法可用於從目前的 context 中移除鍵及其值：

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

您可以透過向 `forget` 方法提供陣列來一次忘記多個鍵：

```php
Context::forget(['first_key', 'second_key']);
```

<a name="hidden-context"></a>
## 隱藏 Context

Context 提供了儲存「隱藏」資料的能力。這些隱藏資訊不會附加到日誌中，也無法透過上述資料擷取方法存取。Context 提供了不同的方法集來與隱藏 context 資訊互動：

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

「隱藏」方法反映了上述非隱藏方法的功能：

```php
Context::addHidden(/* ... */);
Context::addHiddenIf(/* ... */);
Context::pushHidden(/* ... */);
Context::getHidden(/* ... */);
Context::pullHidden(/* ... */);
Context::popHidden(/* ... */);
Context::onlyHidden(/* ... */);
Context::exceptHidden(/* ... */);
Context::allHidden(/* ... */);
Context::hasHidden(/* ... */);
Context::missingHidden(/* ... */);
Context::forgetHidden(/* ... */);
```

<a name="events"></a>
## 事件

Context 分派兩個事件，讓您可以掛接到 context 的水合和脫水過程。

為了說明這些事件如何使用，假設在應用程式的 middleware 中，您根據傳入 HTTP 請求的 `Accept-Language` 標頭設定 `app.locale` 設定值。Context 的事件允許您在請求期間擷取此值並在佇列上恢復它，確保在佇列上傳送的通知具有正確的 `app.locale` 值。我們可以使用 context 的事件和 [隱藏](#hidden-context) 資料來實現這一點，以下文件將說明這一點。

<a name="dehydrating"></a>
### 脫水 (Dehydrating)

每當任務分派到佇列時，context 中的資料會被「脫水」並與任務的 payload 一起擷取。`Context::dehydrating` 方法允許您註冊一個閉包，該閉包將在脫水過程中被調用。在此閉包中，您可以更改將與佇列任務共享的資料。

通常，您應該在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊 `dehydrating` 回呼：

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
> 您不應該在 `dehydrating` 回呼中使用 `Context` facade，因為那會改變目前程序的 context。請確保您只更改傳遞給回呼的 repository。

<a name="hydrated"></a>
### 水合 (Hydrated)

每當佇列任務開始在佇列上執行時，與任務共享的任何 context 都會被「水合」回目前的 context。`Context::hydrated` 方法允許您註冊一個閉包，該閉包將在水合過程中被調用。

通常，您應該在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊 `hydrated` 回呼：

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
> 您不應該在 `hydrated` 回呼中使用 `Context` facade，而應該確保您只更改傳遞給回呼的 repository。

