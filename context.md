# Context

- [簡介](#introduction)
    - [運作方式](#how-it-works)
- [擷取 Context](#capturing-context)
    - [堆疊 (Stacks)](#stacks)
- [取得 Context](#retrieving-context)
    - [判斷項目是否存在](#determining-item-existence)
- [移除 Context](#removing-context)
- [隱藏 Context](#hidden-context)
- [事件](#events)
    - [脫水 (Dehydrating)](#dehydrating)
    - [水合 (Hydrated)](#hydrated)

<a name="introduction"></a>
## 簡介

Laravel 的「context」功能讓您能夠在應用程式中執行的請求、任務 (job) 與命令之間擷取、取得並共享資訊。這些被擷取的資訊也會包含在應用程式寫入的日誌中，讓您能更深入瞭解寫入日誌項目之前發生的相關程式碼執行歷程，並允許您追蹤整個分散式系統中的執行流程。


<a name="how-it-works"></a>
### 運作方式

要理解 Laravel 的 context 功能，最好的方法就是透過內建的日誌功能來觀察其實際運作。若要開始使用，您可以透過 `Context` Facade [將資訊加入至 context](#capturing-context)。在這個範例中，我們將使用[中介層](/docs/{{version}}/middleware)在每個傳入的請求中將請求 URL 與唯一的 trace ID 加入至 context：

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

加入到 context 的資訊會自動作為中繼資料 (metadata) 附加到整個請求期間所寫入的任何[日誌項目](/docs/{{version}}/logging)中。將 context 作為中繼資料附加，可以區分傳遞給個別日誌項目的資訊與透過 `Context` 共享的資訊。例如，假設我們寫入以下日誌項目：

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

寫入的日誌將包含傳遞給日誌項目的 `auth_id`，但同時也會包含 context 的 `url` 與 `trace_id` 作為中繼資料：

```text
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

加入到 context 的資訊也可以供派送至佇列的任務使用。例如，假設我們在將某些資訊加入至 context 之後，將 `ProcessPodcast` 任務派送至佇列：

```php
// In our middleware...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// In our controller...
ProcessPodcast::dispatch($podcast);
```

當任務被派送時，目前儲存在 context 中的任何資訊都會被擷取並與該任務共享。當任務執行時，這些被擷取的資訊接著會被水合 (hydrated) 回當前的 context 中。因此，如果我們任務的 handle 方法要寫入日誌：

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

產生的日誌項目將包含當初派送該任務的請求期間加入到 context 中的資訊：

```text
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

雖然我們著重於 Laravel context 內建的日誌相關功能，但接下來的文件將說明 context 如何讓您跨越 HTTP 請求 / 佇列任務的邊界共享資訊，甚至是如何加入不會隨日誌項目寫入的[隱藏 context 資料](#hidden-context)。

<a name="capturing-context"></a>
## 擷取 Context

您可以使用 `Context` Facade 的 `add` 方法將資訊儲存在目前的 context 中：

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

若要一次新增多個項目，您可以傳遞一個關聯陣列給 `add` 方法：

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

`add` 方法會覆寫任何具有相同鍵名 (Key) 的現有值。如果您只希望在鍵名不存在時才將資訊加入 context，可以使用 `addIf` 方法：

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

Context 還提供了用於遞增或遞減指定鍵值的便捷方法。這兩種方法都至少接受一個引數：要追蹤的鍵名。您可以提供第二個引數來指定鍵值應遞增或遞減的幅度：

```php
Context::increment('records_added');
Context::increment('records_added', 5);

Context::decrement('records_added');
Context::decrement('records_added', 5);
```

<a name="conditional-context"></a>
#### 條件式 Context

`when` 方法可用於根據給定條件將資料新增至 context 中。當給定條件評估為 `true` 時，將會叫用傳遞給 `when` 方法的第一個閉包；若條件評估為 `false`，則會叫用第二個閉包：

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

`scope` 方法提供了一種在執行給定回呼期間暫時修改 context 的方式，並在回呼執行完畢後將 context 恢復為其原始狀態。此外，您可以在閉包執行時傳遞應合併到 context 中的額外資料（作為第二和第三個引數）。

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
> 如果在作用域閉包內修改了 context 中的物件，該變更將會反映在作用域之外。

<a name="stacks"></a>
### 堆疊 (Stacks)

Context 提供了建立「堆疊 (Stacks)」的功能，堆疊是按照新增順序儲存的資料列表。您可以透過叫用 `push` 方法將資訊加入堆疊中：

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

堆疊對於擷取請求的歷史資訊非常有用，例如整個應用程式中正在發生的事件。舉例來說，您可以建立一個事件監聽器，在每次執行查詢時推入堆疊，將查詢的 SQL 和執行時間作為元組 (Tuple) 擷取：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

// In AppServiceProvider.php...
DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

您可以使用 `stackContains` 與 `hiddenStackContains` 方法來判斷某個值是否存在於堆疊中：

```php
if (Context::stackContains('breadcrumbs', 'first_value')) {
    //
}

if (Context::hiddenStackContains('secrets', 'first_value')) {
    //
}
```

`stackContains` 與 `hiddenStackContains` 方法也接受一個閉包作為其第二個引數，以便對值比較操作進行更多控制：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;

return Context::stackContains('breadcrumbs', function ($value) {
    return Str::startsWith($value, 'query_');
});
```

<a name="retrieving-context"></a>
## 取得 Context

您可以使用 `Context` Facade 的 `get` 方法從 context 中取得資訊：

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only` 和 `except` 方法可用於取得 context 中的部分資訊子集：

```php
$data = Context::only(['first_key', 'second_key']);

$data = Context::except(['first_key']);
```

`pull` 方法可用於從 context 取得資訊並立即將其從 context 中移除：

```php
$value = Context::pull('key');
```

如果 context 資料儲存在[堆疊](#stacks)中，您可以使用 `pop` 方法從堆疊中彈出項目：

```php
Context::push('breadcrumbs', 'first_value', 'second_value');

Context::pop('breadcrumbs');
// second_value

Context::get('breadcrumbs');
// ['first_value']
```

`remember` 與 `rememberHidden` 方法可用於從 context 取得資訊，同時在請求的資訊不存在時，將 context 的值設定為給定閉包所回傳的值：

```php
$permissions = Context::remember(
    'user-permissions',
    fn () => $user->permissions,
);
```

如果您想取得儲存在 context 中的所有資訊，可以叫用 `all` 方法：

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### 判斷項目是否存在

您可以使用 `has` 和 `missing` 方法來判斷 context 中是否儲存了給定鍵名的任何值：

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}

if (Context::missing('key')) {
    // ...
}
```

無論儲存的值為何，`has` 方法都將回傳 `true`。因此，例如帶有 `null` 值的鍵名仍會被視為存在：

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## 移除 Context

`forget` 方法可用於從目前的 context 中移除鍵名及其值：

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

您可以透過向 `forget` 方法提供一個陣列來一次忘記多個鍵名：

```php
Context::forget(['first_key', 'second_key']);
```

<a name="hidden-context"></a>
## 隱藏 Context

Context 提供了儲存「隱藏 (Hidden)」資料的功能。這些隱藏資訊不會附加到日誌中，也無法透過上述記錄的資料取得方法來存取。Context 提供了一組不同的方法來與隱藏的 context 資訊進行互動：

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

這些「隱藏」方法對應了上述非隱藏方法的功能：

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

Context 會發送兩個事件，讓您可以掛鉤至 Context 的水合 (Hydration) 與脫水 (Dehydration) 流程中。

為了說明如何使用這些事件，請設想在應用程式的中介層中，您根據傳入 HTTP 請求的 `Accept-Language` 標頭設定了 `app.locale` 設定值。Context 的事件允許您在請求期間擷取此值，並在佇列中將其還原，確保在佇列發送的通知具有正確的 `app.locale` 值。我們可以使用 Context 的事件與[隱藏](#hidden-context)資料來實現這一點，接下來的文件將進行說明。


<a name="dehydrating"></a>
### 脫水 (Dehydrating)

每當任務被派發到佇列時，Context 中的資料就會被「脫水 (Dehydrated)」並與任務的負載 (Payload) 一起被擷取。`Context::dehydrating` 方法允許您註冊一個在脫水過程中被調用的閉包。在此閉包內，您可以對將與佇列任務共享的資料進行修改。

通常，您應該在應用程式的 `AppServiceProvider` 類別中的 `boot` 方法內註冊 `dehydrating` 回呼：

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
> 您不應該在 `dehydrating` 回呼中使用 `Context` Facade，因為那樣會修改目前行程 (Process) 的 Context。請確保僅對傳遞給回呼的儲存庫 (Repository) 進行修改。


<a name="hydrated"></a>
### 水合 (Hydrated)

每當佇列任務開始在佇列上執行時，與該任務共享的任何 Context 都將被「水合 (Hydrated)」回目前的 Context 中。`Context::hydrated` 方法允許您註冊一個將在水合過程中被調用的閉包。

通常，您應該在應用程式的 `AppServiceProvider` 類別中的 `boot` 方法內註冊 `hydrated` 回呼：

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
> 您不應該在 `hydrated` 回呼中使用 `Context` Facade，而是確保僅對傳遞給回呼的儲存庫進行修改。