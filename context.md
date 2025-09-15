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

Laravel 的「Context」功能，讓您能夠在應用程式的整個請求、任務和命令執行過程中，擷取、取得並共用資訊。這些被擷取的資訊也會包含在應用程式寫入的日誌中，讓您能更深入地了解寫入日誌項目之前的程式碼執行脈絡，並得以在分散式系統中追蹤執行的流程。

<a name="how-it-works"></a>
### 運作方式

了解 Laravel Context 功能的最佳方式，是透過內建的日誌功能來實際操作。首先，您可以使用 `Context` Facade [將資訊新增到 Context](#capturing-context)。在這個範例中，我們將使用 [Middleware](/docs/{{version}}/middleware) 在每個傳入的請求中，將請求 URL 和唯一的追蹤 ID 新增到 Context：

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

新增到 Context 的資訊會自動作為中繼資料，附加到整個請求期間寫入的任何 [日誌項目](/docs/{{version}}/logging) 中。將 Context 作為中繼資料附加，可以區分傳遞給個別日誌項目的資訊與透過 `Context` 共用的資訊。例如，假設我們寫入以下日誌項目：

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

寫入的日誌將包含傳遞給日誌項目的 `auth_id`，但也會包含 Context 的 `url` 和 `trace_id` 作為中繼資料：

```
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

新增到 Context 的資訊也可用於分派到 Queue 的任務。例如，假設我們在將一些資訊新增到 Context 後，分派一個 `ProcessPodcast` 任務到 Queue：

```php
// 在我們的 Middleware 中...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// 在我們的 Controller 中...
ProcessPodcast::dispatch($podcast);
```

當任務被分派時，目前儲存在 Context 中的任何資訊都會被擷取並與任務共用。然後，在任務執行期間，擷取的資訊會被水合 (hydrated) 回目前的 Context 中。因此，如果我們的任務的 `handle` 方法要寫入日誌：

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

產生的日誌項目將包含在最初分派任務的請求期間新增到 Context 的資訊：

```
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

儘管我們專注於 Laravel Context 內建的日誌相關功能，但以下文件將說明 Context 如何讓您在 HTTP 請求/佇列任務邊界之間共用資訊，甚至是如何新增 [隱藏的 Context 資料](#hidden-context) 而不寫入日誌項目。

<a name="capturing-context"></a>
## 擷取 Context

您可以使用 `Context` Facade 的 `add` 方法將資訊儲存在目前的 Context 中：

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

`add` 方法將會覆寫任何具有相同 Key 的現有值。如果您只希望在 Key 不存在時才將資訊新增到 Context，您可以使用 `addIf` 方法：

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

<a name="conditional-context"></a>
#### 條件式 Context

`when` 方法可用於根據給定條件將資料新增到 Context。如果給定條件評估為 `true`，則會呼叫傳遞給 `when` 方法的第一個閉包；如果條件評估為 `false`，則會呼叫第二個閉包：

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
### 堆疊 (Stacks)

Context 提供了建立「堆疊」的功能，堆疊是按照新增順序儲存的資料列表。您可以透過呼叫 `push` 方法將資訊新增到堆疊中：

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

堆疊對於擷取請求的歷史資訊很有用，例如應用程式中發生的事件。例如，您可以建立一個事件監聽器，在每次執行查詢時將查詢 SQL 和持續時間作為 Tuple 推送到堆疊中：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

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
## 取得 Context

您可以使用 `Context` Facade 的 `get` 方法從 Context 中取得資訊：

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only` 方法可用於取得 Context 中資訊的子集：

```php
$data = Context::only(['first_key', 'second_key']);
```

`pull` 方法可用於從 Context 中取得資訊並立即將其從 Context 中移除：

```php
$value = Context::pull('key');
```

如果 Context 資料儲存在 [堆疊](#stacks) 中，您可以使用 `pop` 方法從堆疊中彈出項目：

```php
Context::push('breadcrumbs', 'first_value', 'second_value');

Context::pop('breadcrumbs')
// second_value

Context::get('breadcrumbs');
// ['first_value']
```

如果您想取得儲存在 Context 中的所有資訊，您可以呼叫 `all` 方法：

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### 判斷項目是否存在

您可以使用 `has` 和 `missing` 方法判斷 Context 是否為給定 Key 儲存了任何值：

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}

if (Context::missing('key')) {
    // ...
}
```

`has` 方法將返回 `true`，無論儲存的值為何。因此，例如，具有 `null` 值的 Key 將被視為存在：

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## 移除 Context

`forget` 方法可用於從目前的 Context 中移除 Key 及其值：

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

您可以透過向 `forget` 方法提供陣列來一次忘記多個 Key：

```php
Context::forget(['first_key', 'second_key']);
```

<a name="hidden-context"></a>
## 隱藏 Context

Context 提供了儲存「隱藏」資料的功能。這些隱藏資訊不會附加到日誌中，也無法透過上述資料取得方法存取。Context 提供了一組不同的方法來與隱藏的 Context 資訊互動：

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
Context::allHidden(/* ... */);
Context::hasHidden(/* ... */);
Context::forgetHidden(/* ... */);
```

<a name="events"></a>
## 事件

Context 會分派兩個事件，讓您可以掛鉤到 Context 的水合 (hydration) 和脫水 (dehydration) 過程。

為了說明這些事件如何使用，想像在您應用程式的 Middleware 中，您根據傳入 HTTP 請求的 `Accept-Language` 標頭設定 `app.locale` 設定值。Context 的事件允許您在請求期間擷取此值並在 Queue 上還原它，確保在 Queue 上傳送的通知具有正確的 `app.locale` 值。我們可以使用 Context 的事件和 [隱藏](#hidden-context) 資料來實現這一點，以下文件將說明。

<a name="dehydrating"></a>
### 脫水 (Dehydrating)

每當任務被分派到 Queue 時，Context 中的資料都會被「脫水」並與任務的 Payload 一起擷取。`Context::dehydrating` 方法允許您註冊一個閉包，該閉包將在脫水過程中被呼叫。在此閉包中，您可以更改將與佇列任務共用的資料。

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
> 您不應在 `dehydrating` 回呼中使用 `Context` Facade，因為這會改變目前程序的 Context。請確保您只對傳遞給回呼的 Repository 進行更改。

<a name="hydrated"></a>
### 水合 (Hydrated)

每當佇列任務開始在 Queue 上執行時，與任務共用的任何 Context 都會被「水合」回目前的 Context 中。`Context::hydrated` 方法允許您註冊一個閉包，該閉包將在水合過程中被呼叫。

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
> 您不應在 `hydrated` 回呼中使用 `Context` Facade，而應確保您只對傳遞給回呼的 Repository 進行更改。
