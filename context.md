# 內容

- [簡介](#introduction)
    - [運作方式](#how-it-works)
- [擷取內容](#capturing-context)
    - [堆疊](#stacks)
- [取得內容](#retrieving-context)
    - [判斷項目是否存在](#determining-item-existence)
- [移除內容](#removing-context)
- [隱藏內容](#hidden-context)
- [事件](#events)
    - [內容脫水化](#dehydrating)
    - [內容水合化](#hydrated)

<a name="introduction"></a>
## 簡介

Laravel 的 Context 功能讓您能夠在應用程式內執行的請求、任務 (jobs) 與命令中擷取、取得與分享資訊。這些擷取的資訊也會包含在您的應用程式所寫入的日誌中，讓您能更深入了解在日誌項目被寫入前所發生的周圍程式碼執行歷史，並讓您能夠追蹤分散式系統中的執行流程。

<a name="how-it-works"></a>
### 運作方式

了解 Laravel Context 功能的最佳方式是透過內建的日誌功能實際操作。首先，您可以使用 `Context` Facade [將資訊加入至內容中](#capturing-context)。在此範例中，我們將使用 [中介層](/docs/{{version}}/middleware) 來為每個傳入請求的內容加入請求 URL 與一個獨特的追蹤 ID：

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

加入到內容中的資訊將會自動以中繼資料的形式附加到整個請求期間寫入的任何 [日誌項目](/docs/{{version}}/logging) 中。將內容作為中繼資料附加，可以區分傳遞給單個日誌項目的資訊與透過 `Context` 共享的資訊。例如，假設我們寫入以下日誌項目：

```php
Log::info('User authenticated.', ['auth_id' => Auth::id()]);
```

寫入的日誌將包含傳遞給日誌項目的 `auth_id`，但它也會包含內容的 `url` 與 `trace_id` 作為中繼資料：

```text
User authenticated. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

加入到內容中的資訊也會提供給派遣到佇列的任務 (jobs)。例如，假設我們在將一些資訊加入到內容中後，將一個 `ProcessPodcast` 任務派遣到佇列：

```php
// In our middleware...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// In our controller...
ProcessPodcast::dispatch($podcast);
```

當任務被派遣時，任何目前儲存在內容中的資訊都會被擷取並與該任務共享。這些擷取的資訊隨後會在任務執行時被水合 (hydrated) 回目前的內容中。因此，如果我們的任務的 handle 方法要寫入日誌：

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

產生的日誌項目將包含在最初派遣該任務的請求期間加入到內容中的資訊：

```text
Processing podcast. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

儘管我們著重於 Laravel Context 內建的日誌相關功能，但以下的說明文件將闡述 Context 如何讓您能夠在 HTTP 請求／佇列任務邊界之間共享資訊，甚至是如何加入 [隱藏內容資料](#hidden-context) 這些不會隨日誌項目寫入的資訊。

<a name="capturing-context"></a>
## 擷取內容

您可以使用 `Context` Facade 的 `add` 方法，在目前的內容中儲存資訊：

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

若要一次新增多個項目，您可以將一個關聯陣列傳遞給 `add` 方法：

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

`add` 方法將會覆寫任何共享相同鍵的現有值。如果您只想在鍵不存在的情況下才將資訊新增至內容，則可以使用 `addIf` 方法：

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

Context 也提供了方便的方法，用於遞增或遞減給定的鍵。這兩種方法都至少接受一個引數：要追蹤的鍵。第二個引數可以用來指定鍵應遞增或遞減的量：

```php
Context::increment('records_added');
Context::increment('records_added', 5);

Context::decrement('records_added');
Context::decrement('records_added', 5);
```

<a name="conditional-context"></a>
#### 條件內容

`when` 方法可用於根據給定條件將資料新增至內容。如果給定條件評估為 `true`，則會呼叫提供給 `when` 方法的第一個閉包；如果條件評估為 `false`，則會呼叫第二個閉包：

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
#### 作用域內容

`scope` 方法提供了一種方式，可以在給定回呼執行期間暫時修改內容，並在回呼執行完成後將內容還原到其原始狀態。此外，您可以在閉包執行期間，傳遞應合併到內容中的額外資料（作為第二個和第三個引數）。

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
> 如果內容中的物件在作用域閉包內被修改，該變動將會反映在作用域之外。

<a name="stacks"></a>
### 堆疊

Context 提供了建立「堆疊」的能力，堆疊是按照資料新增順序儲存的清單。您可以透過呼叫 `push` 方法將資訊新增到堆疊：

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

堆疊可用於擷取請求的歷史資訊，例如應用程式中發生的事件。舉例來說，您可以建立一個事件監聽器，以便每次執行查詢時都將其推送到堆疊，並將查詢的 SQL 和持續時間作為元組擷取：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

// In AppServiceProvider.php...
DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

您可以使用 `stackContains` 和 `hiddenStackContains` 方法來判斷某個值是否在堆疊中：

```php
if (Context::stackContains('breadcrumbs', 'first_value')) {
    //
}

if (Context::hiddenStackContains('secrets', 'first_value')) {
    //
}
```

`stackContains` 和 `hiddenStackContains` 方法也接受一個閉包作為它們的第二個引數，從而允許對值比較操作有更多的控制：

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;

return Context::stackContains('breadcrumbs', function ($value) {
    return Str::startsWith($value, 'query_');
});
```

<a name="retrieving-context"></a>
## 取得內容

您可以使用 `Context` Facade 的 `get` 方法，從內容中取得資訊：

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

`only` 和 `except` 方法可用於取得內容中資訊的子集：

```php
$data = Context::only(['first_key', 'second_key']);

$data = Context::except(['first_key']);
```

`pull` 方法可用於從內容中取得資訊，並立即從內容中移除它：

```php
$value = Context::pull('key');
```

如果內容資料儲存在 [堆疊](#stacks) 中，您可以使用 `pop` 方法從堆疊中彈出項目：

```php
Context::push('breadcrumbs', 'first_value', 'second_value');

Context::pop('breadcrumbs');
// second_value

Context::get('breadcrumbs');
// ['first_value']
```

`remember` 和 `rememberHidden` 方法可用於從內容中取得資訊，如果請求的資訊不存在，則會將內容值設定為給定閉包回傳的值：

```php
$permissions = Context::remember(
    'user-permissions',
    fn () => $user->permissions,
);
```

如果您想取得內容中儲存的所有資訊，可以呼叫 `all` 方法：

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### 判斷項目是否存在

您可以使用 `has` 和 `missing` 方法來判斷內容是否為給定的鍵儲存了任何值：

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}

if (Context::missing('key')) {
    // ...
}
```

無論儲存的值為何，`has` 方法都會回傳 `true`。因此，舉例來說，一個值為 `null` 的鍵將會被視為存在：

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## 移除內容

`forget` 方法可用於從目前的內容中移除一個鍵及其值：

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
## 隱藏內容

Context 提供了儲存「隱藏」資料的能力。這些隱藏資訊不會附加到日誌中，也無法透過上述資料取得方法存取。Context 提供了一組不同的方法來與隱藏內容資訊互動：

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

這些「隱藏」方法反映了上述非隱藏方法的功能：

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

Context 會觸發兩個事件，讓您可以掛載到內容 (context) 的水合化與脫水化過程。

為了說明這些事件的用途，想像您的應用程式中，有一個中介層會根據傳入 HTTP 請求的 `Accept-Language` 標頭設定 `app.locale` 設定值。Context 的事件讓您可以在請求期間擷取這個值，並在佇列上還原，確保透過佇列傳送的通知擁有正確的 `app.locale` 值。我們可以利用 Context 的事件和 [隱藏](#hidden-context) 資料來實現此目的，接下來的文件將會說明。


<a name="dehydrating"></a>
### 內容脫水化

每當有任務被派送到佇列時，內容 (context) 中的資料會被「脫水化」，並與任務的 payload 一同被擷取。`Context::dehydrating` 方法讓您可以註冊一個閉包，該閉包會在脫水化過程中被呼叫。在這個閉包中，您可以修改將與佇列任務共用的資料。

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

> [!NOTE] 注意
> 您不應該在 `dehydrating` 回呼中使用 `Context` Facade，因為那會改變目前程序的內容。請確保您只對傳遞給回呼的 Repository 進行更改。


<a name="hydrated"></a>
### 內容水合化

每當一個佇列任務開始在佇列上執行時，任何與該任務共用的內容 (context) 都將會「水合化」回目前的內容中。`Context::hydrated` 方法讓您可以註冊一個閉包，該閉包會在水合化過程中被呼叫。

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

> [!NOTE] 注意
> 您不應該在 `hydrated` 回呼中使用 `Context` Facade，相反地，請確保您只對傳遞給回呼的 Repository 進行更改。