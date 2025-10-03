# URL 產生

- [介紹](#introduction)
- [基本概念](#the-basics)
    - [產生 URL](#generating-urls)
    - [存取目前 URL](#accessing-the-current-url)
- [具名路由的 URL](#urls-for-named-routes)
    - [簽章式 URL](#signed-urls)
- [控制器行動的 URL](#urls-for-controller-actions)
- [流暢型 URI 物件](#fluent-uri-objects)
- [預設值](#default-values)

<a name="introduction"></a>
## 介紹

Laravel 提供了多個輔助方法來協助您為應用程式產生 URL。這些輔助方法主要用於在您的範本和 API 回應中建立連結，或在產生重新導向回應到應用程式的另一個部分時非常有用。

<a name="the-basics"></a>
## 基本概念

<a name="generating-urls"></a>
### 產生 URL

`url` 輔助方法可用於為您的應用程式產生任意 URL。產生的 URL 將自動使用應用程式正在處理的目前請求中的 scheme (HTTP 或 HTTPS) 和 host：

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

值陣列也可以作為查詢參數傳遞。這些值將在產生的 URL 中正確地鍵控並編碼：

```php
echo $url = url()->query('/posts', ['columns' => ['title', 'body']]);

// http://example.com/posts?columns%5B0%5D=title&columns%5B1%5D=body

echo urldecode($url);

// http://example.com/posts?columns[0]=title&columns[1]=body
```

<a name="accessing-the-current-url"></a>
### 存取目前 URL

如果沒有向 `url` 輔助方法提供路徑，則會返回一個 `Illuminate\Routing\UrlGenerator` 實例，允許您存取有關目前 URL 的資訊：

```php
// Get the current URL without the query string...
echo url()->current();

// Get the current URL including the query string...
echo url()->full();

// Get the full URL for the previous request...
echo url()->previous();

// Get the path for the previous request...
echo url()->previousPath();
```

每個這些方法也可以透過 `URL` [Facade](/docs/{{version}}/facades) 存取：

```php
use Illuminate\Support\Facades\URL;

echo URL::current();
```

<a name="urls-for-named-routes"></a>
## 具名路由的 URL

`route` 輔助方法可用於為 [具名路由](/docs/{{version}}/routing#named-routes) 產生 URL。具名路由允許您產生 URL，而無需與路由上定義的實際 URL 綁定。因此，如果路由的 URL 發生變化，您的 `route` 函式呼叫無需進行任何修改。例如，假設您的應用程式包含一個定義如下的路由：

```php
Route::get('/post/{post}', function (Post $post) {
    // ...
})->name('post.show');
```

若要產生此路由的 URL，您可以使用 `route` 輔助方法，如下所示：

```php
echo route('post.show', ['post' => 1]);

// http://example.com/post/1
```

當然，`route` 輔助方法也可以用於產生帶有多個參數的路由的 URL：

```php
Route::get('/post/{post}/comment/{comment}', function (Post $post, Comment $comment) {
    // ...
})->name('comment.show');

echo route('comment.show', ['post' => 1, 'comment' => 3]);

// http://example.com/post/1/comment/3
```

任何不符合路由定義參數的額外陣列元素將被新增到 URL 的查詢字串中：

```php
echo route('post.show', ['post' => 1, 'search' => 'rocket']);

// http://example.com/post/1?search=rocket
```

<a name="eloquent-models"></a>
#### Eloquent 模型

您通常會使用 [Eloquent 模型](/docs/{{version}}/eloquent) 的路由鍵 (通常是主鍵) 來產生 URL。因此，您可以將 Eloquent 模型作為參數值傳遞。`route` 輔助方法將自動提取模型的路由鍵：

```php
echo route('post.show', ['post' => $post]);
```

<a name="signed-urls"></a>
### 簽章式 URL

Laravel 允許您輕鬆地為具名路由建立「簽章式」URL。這些 URL 的查詢字串中附加了一個「簽章」雜湊，這使得 Laravel 可以驗證 URL 自建立以來未被修改。簽章式 URL 對於可公開存取但需要一層保護以防止 URL 操縱的路由特別有用。

例如，您可以使用簽章式 URL 實作發送給客戶的公開「取消訂閱」連結。若要為具名路由建立簽章式 URL，請使用 `URL` Facade 的 `signedRoute` 方法：

```php
use Illuminate\Support\Facades\URL;

return URL::signedRoute('unsubscribe', ['user' => 1]);
```

您可以透過向 `signedRoute` 方法提供 `absolute` 引數來從簽章式 URL 雜湊中排除網域：

```php
return URL::signedRoute('unsubscribe', ['user' => 1], absolute: false);
```

如果您想產生一個在指定時間後過期的暫時簽章式路由 URL，您可以使用 `temporarySignedRoute` 方法。當 Laravel 驗證暫時簽章式路由 URL 時，它將確保編碼到簽章式 URL 中的過期時間戳記尚未經過：

```php
use Illuminate\Support\Facades\URL;

return URL::temporarySignedRoute(
    'unsubscribe', now()->addMinutes(30), ['user' => 1]
);
```

<a name="validating-signed-route-requests"></a>
#### 驗證簽章式路由請求

為了驗證傳入請求是否具有有效簽章，您應該在傳入的 `Illuminate\Http\Request` 實例上呼叫 `hasValidSignature` 方法：

```php
use Illuminate\Http\Request;

Route::get('/unsubscribe/{user}', function (Request $request) {
    if (! $request->hasValidSignature()) {
        abort(401);
    }

    // ...
})->name('unsubscribe');
```

有時，您可能需要允許應用程式的前端向簽章式 URL 附加資料，例如在執行客戶端分頁時。因此，您可以使用 `hasValidSignatureWhileIgnoring` 方法指定在驗證簽章式 URL 時應忽略的請求查詢參數。請記住，忽略參數允許任何人修改請求上的這些參數：

```php
if (! $request->hasValidSignatureWhileIgnoring(['page', 'order'])) {
    abort(401);
}
```

您可以將 `signed` (`Illuminate\Routing\Middleware\ValidateSignature`) [中介層](/docs/{{version}}/middleware) 分配給路由，而不是使用傳入請求實例來驗證簽章式 URL。如果傳入請求沒有有效簽章，中介層將自動返回 `403` HTTP 回應：

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed');
```

如果您的簽章式 URL 未在 URL 雜湊中包含網域，您應該向中介層提供 `relative` 引數：

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed:relative');
```

<a name="responding-to-invalid-signed-routes"></a>
#### 回應無效的簽章式路由

當有人訪問已過期的簽章式 URL 時，他們將收到 `403` HTTP 狀態碼的通用錯誤頁面。但是，您可以透過在應用程式的 `bootstrap/app.php` 檔案中為 `InvalidSignatureException` 異常定義一個自訂的「render」閉包來客製化此行為：

```php
use Illuminate\Routing\Exceptions\InvalidSignatureException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidSignatureException $e) {
        return response()->view('errors.link-expired', status: 403);
    });
})
```

<a name="urls-for-controller-actions"></a>
## 控制器行動的 URL

`action` 函式為給定的控制器行動產生 URL：

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

如果控制器方法接受路由參數，您可以將路由參數的關聯陣列作為第二個引數傳遞給該函式：

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```

<a name="fluent-uri-objects"></a>
## 流暢型 URI 物件

Laravel 的 `Uri` 類別提供了一個方便且流暢的介面，可用於透過物件建立和操作 URI。這個類別包裝了底層 League URI 函式庫所提供的功能，並與 Laravel 的路由系統無縫整合。

您可以使用靜態方法輕鬆建立 `Uri` 實例：

```php
use App\Http\Controllers\UserController;
use App\Http\Controllers\InvokableController;
use Illuminate\Support\Uri;

// Generate a URI instance from the given string...
$uri = Uri::of('https://example.com/path');

// Generate URI instances to paths, named routes, or controller actions...
$uri = Uri::to('/dashboard');
$uri = Uri::route('users.show', ['user' => 1]);
$uri = Uri::signedRoute('users.show', ['user' => 1]);
$uri = Uri::temporarySignedRoute('user.index', now()->addMinutes(5));
$uri = Uri::action([UserController::class, 'index']);
$uri = Uri::action(InvokableController::class);

// Generate a URI instance from the current request URL...
$uri = $request->uri();
```

一旦您擁有 Uri 實例，就可以流暢地修改它：

```php
$uri = Uri::of('https://example.com')
    ->withScheme('http')
    ->withHost('test.com')
    ->withPort(8000)
    ->withPath('/users')
    ->withQuery(['page' => 2])
    ->withFragment('section-1');
```

有關使用流暢型 URI 物件的更多資訊，請參考 [URI 文件](/docs/{{version}}/helpers#uri)。


<a name="default-values"></a>
## 預設值

對於某些應用程式，您可能希望為特定的 URL 參數指定請求範圍的預設值。例如，想像您的許多路由都定義了一個 `{locale}` 參數：

```php
Route::get('/{locale}/posts', function () {
    // ...
})->name('post.index');
```

每次呼叫 `route` 輔助函式時都傳遞 `locale` 是很麻煩的。因此，您可以使用 `URL::defaults` 方法來定義此參數的預設值，該值將始終應用於當前請求。您可能希望從 [路由中介層](/docs/{{version}}/middleware#assigning-middleware-to-routes) 中呼叫此方法，以便您可以存取當前請求：

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

一旦為 `locale` 參數設定了預設值，當透過 `route` 輔助函式產生 URL 時，您就不再需要傳遞其值。


<a name="url-defaults-middleware-priority"></a>
#### URL 預設值與中介層優先級

設定 URL 預設值可能會干擾 Laravel 處理隱式模型綁定的方式。因此，您應該 [優先執行您的中介層](/docs/{{version}}/middleware#sorting-middleware)，讓其在 Laravel 自己的 `SubstituteBindings` 中介層之前執行。您可以透過應用程式的 `bootstrap/app.php` 檔案中的 `priority` 中介層方法來實現這一點：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->prependToPriorityList(
        before: \Illuminate\Routing\Middleware\SubstituteBindings::class,
        prepend: \App\Http\Middleware\SetDefaultLocaleForUrls::class,
    );
})
```