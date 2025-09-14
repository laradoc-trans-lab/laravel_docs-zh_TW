# URL 產生

- [簡介](#introduction)
- [基本概念](#the-basics)
    - [產生 URL](#generating-urls)
    - [存取目前的 URL](#accessing-the-current-url)
- [具名路由的 URL](#urls-for-named-routes)
    - [簽章 URL](#signed-urls)
- [Controller Action 的 URL](#urls-for-controller-actions)
- [流暢的 URI 物件](#fluent-uri-objects)
- [預設值](#default-values)

<a name="introduction"></a>
## 簡介

Laravel 提供了多個輔助函式來協助您為應用程式產生 URL。這些輔助函式主要用於在您的樣板與 API 回應中建立連結，或在產生重新導向回應到應用程式的其他部分時。

<a name="the-basics"></a>
## 基本概念

<a name="generating-urls"></a>
### 產生 URL

`url` 輔助函式可用於為您的應用程式產生任意 URL。產生的 URL 將自動使用應用程式目前處理的請求中的協定 (HTTP 或 HTTPS) 與主機：

```php
$post = App\Models\Post::find(1);

echo url("/posts/{$post->id}");

// http://example.com/posts/1
```

若要產生帶有查詢字串參數的 URL，您可以使用 `query` 方法：

```php
echo url()->query('/posts', ['search' => 'Laravel']);

// https://example.com/posts?search=Laravel

echo url()->query('/posts?sort=latest', ['search' => 'Laravel']);

// http://example.com/posts?sort=latest&search=Laravel
```

提供路徑中已存在的查詢字串參數將會覆寫其現有值：

```php
echo url()->query('/posts?sort=latest', ['sort' => 'oldest']);

// http://example.com/posts?sort=oldest
```

值陣列也可以作為查詢參數傳遞。這些值將在產生的 URL 中正確地鍵入與編碼：

```php
echo $url = url()->query('/posts', ['columns' => ['title', 'body']]);

// http://example.com/posts?columns%5B0%5D=title&columns%5B1%5D=body

echo urldecode($url);

// http://example.com/posts?columns[0]=title&columns[1]=body
```

<a name="accessing-the-current-url"></a>
### 存取目前的 URL

如果沒有為 `url` 輔助函式提供路徑，則會回傳一個 `Illuminate\Routing\UrlGenerator` 實例，讓您可以存取有關目前 URL 的資訊：

```php
// 取得不含查詢字串的目前 URL...
echo url()->current();

// 取得包含查詢字串的目前 URL...
echo url()->full();

// 取得前一個請求的完整 URL...
echo url()->previous();

// 取得前一個請求的路徑...
echo url()->previousPath();
```

這些方法中的每一個也可以透過 `URL` [Facade](/docs/{{version}}/facades) 存取：

```php
use Illuminate\Support\Facades\URL;

echo URL::current();
```

<a name="urls-for-named-routes"></a>
## 具名路由的 URL

`route` 輔助函式可用於產生指向[具名路由](/docs/{{version}}/routing#named-routes)的 URL。具名路由讓您可以在不與路由上定義的實際 URL 耦合的情況下產生 URL。因此，如果路由的 URL 發生變化，則無需對您呼叫 `route` 函式的程式碼進行任何更改。例如，假設您的應用程式包含一個定義如下的路由：

```php
Route::get('/post/{post}', function (Post $post) {
    // ...
})->name('post.show');
```

若要產生指向此路由的 URL，您可以使用 `route` 輔助函式，如下所示：

```php
echo route('post.show', ['post' => 1]);

// http://example.com/post/1
```

當然，`route` 輔助函式也可以用於產生帶有多個參數的路由的 URL：

```php
Route::get('/post/{post}/comment/{comment}', function (Post $post, Comment $comment) {
    // ...
})->name('comment.show');

echo route('comment.show', ['post' => 1, 'comment' => 3]);

// http://example.com/post/1/comment/3
```

任何不對應於路由定義參數的額外陣列元素都將新增到 URL 的查詢字串中：

```php
echo route('post.show', ['post' => 1, 'search' => 'rocket']);

// http://example.com/post/1?search=rocket
```

<a name="eloquent-models"></a>
#### Eloquent Model

您通常會使用 [Eloquent Model](/docs/{{version}}/eloquent) 的路由鍵 (通常是主鍵) 來產生 URL。因此，您可以將 Eloquent Model 作為參數值傳遞。`route` 輔助函式將自動提取 Model 的路由鍵：

```php
echo route('post.show', ['post' => $post]);
```

<a name="signed-urls"></a>
### 簽章 URL

Laravel 讓您可以輕鬆地為具名路由建立「簽章」URL。這些 URL 的查詢字串中附加了一個「簽章」雜湊，這讓 Laravel 可以驗證 URL 自建立以來是否未被修改。簽章 URL 對於公開可存取但需要一層保護以防止 URL 操縱的路由特別有用。

例如，您可以使用簽章 URL 來實作一個公開的「取消訂閱」連結，該連結會透過電子郵件發送給您的客戶。若要為具名路由建立簽章 URL，請使用 `URL` Facade 的 `signedRoute` 方法：

```php
use Illuminate\Support\Facades\URL;

return URL::signedRoute('unsubscribe', ['user' => 1]);
```

您可以透過向 `signedRoute` 方法提供 `absolute` 引數來從簽章 URL 雜湊中排除網域：

```php
return URL::signedRoute('unsubscribe', ['user' => 1], absolute: false);
```

如果您想產生一個在指定時間後過期的臨時簽章路由 URL，您可以使用 `temporarySignedRoute` 方法。當 Laravel 驗證臨時簽章路由 URL 時，它將確保編碼到簽章 URL 中的過期時間戳記尚未過期：

```php
use Illuminate\Support\Facades\URL;

return URL::temporarySignedRoute(
    'unsubscribe', now()->addMinutes(30), ['user' => 1]
);
```

<a name="validating-signed-route-requests"></a>
#### 驗證簽章路由請求

若要驗證傳入請求是否具有有效的簽章，您應該在傳入的 `Illuminate\Http\Request` 實例上呼叫 `hasValidSignature` 方法：

```php
use Illuminate\Http\Request;

Route::get('/unsubscribe/{user}', function (Request $request) {
    if (! $request->hasValidSignature()) {
        abort(401);
    }

    // ...
})->name('unsubscribe');
```

有時，您可能需要允許應用程式的前端向簽章 URL 附加資料，例如在執行用戶端分頁時。因此，您可以使用 `hasValidSignatureWhileIgnoring` 方法指定在驗證簽章 URL 時應忽略的請求查詢參數。請記住，忽略參數允許任何人修改請求上的這些參數：

```php
if (! $request->hasValidSignatureWhileIgnoring(['page', 'order'])) {
    abort(401);
}
```

除了使用傳入的請求實例驗證簽章 URL 之外，您還可以將 `signed` (`Illuminate\Routing\Middleware\ValidateSignature`) [Middleware](/docs/{{version}}/middleware) 指派給路由。如果傳入的請求沒有有效的簽章，Middleware 將自動回傳 `403` HTTP 回應：

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed');
```

如果您的簽章 URL 不包含 URL 雜湊中的網域，您應該向 Middleware 提供 `relative` 引數：

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed:relative');
```

<a name="responding-to-invalid-signed-routes"></a>
#### 回應無效的簽章路由

當有人造訪已過期的簽章 URL 時，他們將收到 `403` HTTP 狀態碼的通用錯誤頁面。但是，您可以透過在應用程式的 `bootstrap/app.php` 檔案中為 `InvalidSignatureException` 異常定義一個自訂的「render」閉包來客製化此行為：

```php
use Illuminate\Routing\Exceptions\InvalidSignatureException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidSignatureException $e) {
        return response()->view('errors.link-expired', status: 403);
    });
})
```

<a name="urls-for-controller-actions"></a>
## Controller Action 的 URL

`action` 函式會為給定的 Controller Action 產生 URL：

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

如果 Controller 方法接受路由參數，您可以將路由參數的關聯陣列作為第二個引數傳遞給函式：

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```

<a name="fluent-uri-objects"></a>
## 流暢的 URI 物件

Laravel 的 `Uri` 類別提供了一個方便且流暢的介面，用於透過物件建立和操作 URI。此類別封裝了底層 League URI 套件提供的功能，並與 Laravel 的路由系統無縫整合。

您可以使用靜態方法輕鬆建立 `Uri` 實例：

```php
use App\Http\Controllers\UserController;
use App\Http\Controllers\InvokableController;
use Illuminate\Support\Uri;

// 從給定字串產生 URI 實例...
$uri = Uri::of('https://example.com/path');

// 產生指向路徑、具名路由或 Controller Action 的 URI 實例...
$uri = Uri::to('/dashboard');
$uri = Uri::route('users.show', ['user' => 1]);
$uri = Uri::signedRoute('users.show', ['user' => 1]);
$uri = Uri::temporarySignedRoute('user.index', now()->addMinutes(5));
$uri = Uri::action([UserController::class, 'index']);
$uri = Uri::action(InvokableController::class);

// 從目前請求 URL 產生 URI 實例...
$uri = $request->uri();
```

一旦您有了 URI 實例，您就可以流暢地修改它：

```php
$uri = Uri::of('https://example.com')
    ->withScheme('http')
    ->withHost('test.com')
    ->withPort(8000)
    ->withPath('/users')
    ->withQuery(['page' => 2])
    ->withFragment('section-1');
```

有關使用流暢 URI 物件的更多資訊，請參閱 [URI 說明文件](/docs/{{version}}/helpers#uri)。

<a name="default-values"></a>
## 預設值

對於某些應用程式，您可能希望為某些 URL 參數指定請求範圍的預設值。例如，想像您的許多路由都定義了一個 `{locale}` 參數：

```php
Route::get('/{locale}/posts', function () {
    // ...
})->name('post.index');
```

每次呼叫 `route` 輔助函式時都傳遞 `locale` 會很麻煩。因此，您可以使用 `URL::defaults` 方法為此參數定義一個預設值，該值將始終在目前請求期間應用。您可能希望從 [路由 Middleware](/docs/{{version}}/middleware#assigning-middleware-to-routes) 呼叫此方法，以便您可以存取目前的請求：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\URL;
use Symfony\Component\HttpFoundation\Response;

class SetDefaultLocaleForUrls
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        URL::defaults(['locale' => $request->user()->locale]);

        return $next($request);
    }
}
```

一旦設定了 `locale` 參數的預設值，您就不再需要在透過 `route` 輔助函式產生 URL 時傳遞其值。

<a name="url-defaults-middleware-priority"></a>
#### URL 預設值與 Middleware 優先順序

設定 URL 預設值可能會干擾 Laravel 處理隱式 Model 綁定。因此，您應該[優先處理您的 Middleware](/docs/{{version}}/middleware#sorting-middleware)，使其在 Laravel 自己的 `SubstituteBindings` Middleware 之前執行。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `priority` Middleware 方法來實現此目的：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->prependToPriorityList(
        before: \Illuminate\Routing\Middleware\SubstituteBindings::class,
        prepend: \App\Http\Middleware\SetDefaultLocaleForUrls::class,
    );
})
```
