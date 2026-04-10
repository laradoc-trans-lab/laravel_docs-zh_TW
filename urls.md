# URL 生成

- [簡介](#introduction)
- [基礎](#the-basics)
    - [生成 URL](#generating-urls)
    - [存取當前 URL](#accessing-the-current-url)
- [具名路由的 URL](#urls-for-named-routes)
    - [簽章式 URL](#signed-urls)
- [控制器動作的 URL](#urls-for-controller-actions)
- [流暢的 URI 物件](#fluent-uri-objects)
- [預設值](#default-values)

<a name="introduction"></a>
## 簡介

Laravel 提供了數個輔助函數，協助您為應用程式生成 URL。這些輔助函數主要在您的模板和 API 回應中建立連結時很有幫助，或者在生成重新導向回應到應用程式的其他部分時。


<a name="the-basics"></a>
## 基礎


<a name="generating-urls"></a>
### 生成 URL

`url` 輔助函數可用於為您的應用程式生成任意 URL。生成的 URL 將自動使用應用程式處理的當前請求中的協定 (HTTP 或 HTTPS) 和主機：

```php
$post = App\Models\Post::find(1);

echo url("/posts/{$post->id}");

// http://example.com/posts/1
```

若要生成帶有查詢字串參數的 URL，您可以使用 `query` 方法：

```php
echo url()->query('/posts', ['search' => 'Laravel']);

// https://example.com/posts?search=Laravel

echo url()->query('/posts?sort=latest', ['search' => 'Laravel']);

// http://example.com/posts?sort=latest&search=Laravel
```

提供路徑中已存在的查詢字串參數，將會覆寫其現有值：

```php
echo url()->query('/posts?sort=latest', ['sort' => 'oldest']);

// http://example.com/posts?sort=oldest
```

值的陣列也可以作為查詢參數傳遞。這些值將在生成的 URL 中正確地進行鍵值設定和編碼：

```php
echo $url = url()->query('/posts', ['columns' => ['title', 'body']]);

// http://example.com/posts?columns%5B0%5D=title&columns%5B1%5D=body

echo urldecode($url);

// http://example.com/posts?columns[0]=title&columns[1]=body
```


<a name="accessing-the-current-url"></a>
### 存取當前 URL

如果沒有向 `url` 輔助函數提供路徑，則會回傳一個 `Illuminate\Routing\UrlGenerator` 實例，允許您存取有關當前 URL 的資訊：

```php
// Get the current URL without the query string...
echo url()->current();

// Get the current URL including the query string...
echo url()->full();
```

這些方法也可以透過 `URL` [Facade](/docs/{{version}}/facades) 存取：

```php
use Illuminate\Support\Facades\URL;

echo URL::current();
```


<a name="accessing-the-previous-url"></a>
#### 存取先前的 URL

有時，了解使用者從哪個先前的 URL 訪問會很有幫助。您可以透過 `url` 輔助函數的 `previous` 和 `previousPath` 方法來存取先前的 URL：

```php
// Get the full URL for the previous request...
echo url()->previous();

// Get the path for the previous request...
echo url()->previousPath();
```

或者，透過 [session](/docs/{{version}}/session)，您可以將先前的 URL 作為 [流暢的 URI 物件](#fluent-uri-objects) 實例來存取：

```php
use Illuminate\Http\Request;

Route::post('/users', function (Request $request) {
    $previousUri = $request->session()->previousUri();

    // ...
});
```

也可以透過 session 檢索先前訪問的 URL 的路由名稱：

```php
$previousRoute = $request->session()->previousRoute();
```

<a name="urls-for-named-routes"></a>
## 具名路由的 URL

`route` 輔助函式可用於生成 [具名路由](/docs/{{version}}/routing#named-routes) 的 URL。具名路由允許您生成 URL，而無需與路由上定義的實際 URL 綁定。因此，如果路由的 URL 發生變更，您對 `route` 函式的呼叫無需進行任何修改。例如，假設您的應用程式包含如下定義的路由：

```php
Route::get('/post/{post}', function (Post $post) {
    // ...
})->name('post.show');
```

要為此路由生成 URL，您可以這樣使用 `route` 輔助函式：

```php
echo route('post.show', ['post' => 1]);

// http://example.com/post/1
```

當然，`route` 輔助函式也可用於生成帶有多個參數的路由 URL：

```php
Route::get('/post/{post}/comment/{comment}', function (Post $post, Comment $comment) {
    // ...
})->name('comment.show');

echo route('comment.show', ['post' => 1, 'comment' => 3]);

// http://example.com/post/1/comment/3
```

任何不屬於路由定義參數的額外陣列元素將會被新增到 URL 的查詢字串中：

```php
echo route('post.show', ['post' => 1, 'search' => 'rocket']);

// http://example.com/post/1?search=rocket
```

<a name="eloquent-models"></a>
#### Eloquent 模型

您經常會使用 [Eloquent 模型](/docs/{{version}}/eloquent) 的路由鍵 (通常是主鍵) 來生成 URL。因此，您可以將 Eloquent 模型作為參數值傳入。`route` 輔助函式將會自動提取模型的路由鍵：

```php
echo route('post.show', ['post' => $post]);
```

<a name="signed-urls"></a>
### 簽章式 URL

Laravel 允許您輕鬆地為具名路由建立「簽章式」URL。這些 URL 的查詢字串中附加了一個「簽章」雜湊值，允許 Laravel 驗證該 URL 自建立以來是否已被修改。簽章式 URL 對於那些公開可存取但需要一層保護以防止 URL 遭到操縱的路由特別有用。

例如，您可以使用簽章式 URL 來實現一個公開的「取消訂閱」連結，並透過電子郵件發送給您的客戶。若要為具名路由建立簽章式 URL，請使用 `URL` Facade 的 `signedRoute` 方法：

```php
use Illuminate\Support\Facades\URL;

return URL::signedRoute('unsubscribe', ['user' => 1]);
```

您可以透過向 `signedRoute` 方法提供 `absolute` 參數，將網域從簽章式 URL 雜湊中排除：

```php
return URL::signedRoute('unsubscribe', ['user' => 1], absolute: false);
```

如果您想生成一個在指定時間後過期的臨時簽章式路由 URL，您可以使用 `temporarySignedRoute` 方法。當 Laravel 驗證臨時簽章式路由 URL 時，它會確保編碼到簽章式 URL 中的過期時間戳尚未經過：

```php
use Illuminate\Support\Facades\URL;

return URL::temporarySignedRoute(
    'unsubscribe', now()->plus(minutes: 30), ['user' => 1]
);
```

<a name="validating-signed-route-requests"></a>
#### 驗證簽章式路由請求

若要驗證傳入的請求是否具有有效的簽章，您應該在傳入的 `Illuminate\Http\Request` 實例上呼叫 `hasValidSignature` 方法：

```php
use Illuminate\Http\Request;

Route::get('/unsubscribe/{user}', function (Request $request) {
    if (! $request->hasValidSignature()) {
        abort(401);
    }

    // ...
})->name('unsubscribe');
```

有時，您可能需要允許應用程式的前端向簽章式 URL 附加資料，例如在執行客戶端分頁時。因此，您可以使用 `hasValidSignatureWhileIgnoring` 方法來指定在驗證簽章式 URL 時應忽略的請求查詢參數。請記住，忽略參數會允許任何人在請求中修改這些參數：

```php
if (! $request->hasValidSignatureWhileIgnoring(['page', 'order'])) {
    abort(401);
}
```

除了使用傳入的請求實例驗證簽章式 URL 之外，您還可以將 `signed` (`Illuminate\Routing\Middleware\ValidateSignature`) [中介層](/docs/{{version}}/middleware) 指派給路由。如果傳入的請求沒有有效的簽章，該中介層將會自動返回 `403` HTTP 回應：

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed');
```

如果您的簽章式 URL 不包含 URL 雜湊中的網域，您應該向中介層提供 `relative` 參數：

```php
Route::post('/unsubscribe/{user}', function (Request $request) {
    // ...
})->name('unsubscribe')->middleware('signed:relative');
```

<a name="responding-to-invalid-signed-routes"></a>
#### 回應無效的簽章式路由

當有人造訪已過期的簽章式 URL 時，他們將會收到 `403` HTTP 狀態碼的通用錯誤頁面。不過，您可以透過在應用程式的 `bootstrap/app.php` 檔案中為 `InvalidSignatureException` 例外定義一個自訂的「render」閉包來客製化此行為：

```php
use Illuminate\Routing\Exceptions\InvalidSignatureException;

->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (InvalidSignatureException $e) {
        return response()->view('errors.link-expired', status: 403);
    });
})
```

<a name="urls-for-controller-actions"></a>
## 控制器動作的 URL

`action` 函式會為指定的控制器動作生成 URL：

```php
use App\Http\Controllers\HomeController;

$url = action([HomeController::class, 'index']);
```

如果控制器方法接受路由參數，您可以將一個關聯陣列的路由參數作為第二個參數傳遞給該函式：

```php
$url = action([UserController::class, 'profile'], ['id' => 1]);
```

<a name="fluent-uri-objects"></a>
## 流暢的 URI 物件

Laravel 的 `Uri` 類別提供了一個方便且流暢的介面，可用於透過物件建立和操作 URI。這個類別封裝了底層 League URI 套件提供的功能，並與 Laravel 的路由系統無縫整合。

您可以輕鬆地使用靜態方法建立 `Uri` 實例：

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
$uri = Uri::temporarySignedRoute('user.index', now()->plus(minutes: 5));
$uri = Uri::action([UserController::class, 'index']);
$uri = Uri::action(InvokableController::class);

// Generate a URI instance from the current request URL...
$uri = $request->uri();

// Generate a URI instance from the previous request URL...
$uri = $request->session()->previousUri();
```

一旦您有了 URI 實例，您可以流暢地修改它：

```php
$uri = Uri::of('https://example.com')
    ->withScheme('http')
    ->withHost('test.com')
    ->withPort(8000)
    ->withPath('/users')
    ->withQuery(['page' => 2])
    ->withFragment('section-1');
```

有關使用流暢 URI 物件的更多資訊，請查閱 [URI 文件](/docs/{{version}}/helpers#uri)。

<a name="default-values"></a>
## 預設值

對於某些應用程式，您可能希望為特定的 URL 參數指定請求範圍的預設值。例如，假設您的許多路由都定義了一個 `{locale}` 參數：

```php
Route::get('/{locale}/posts', function () {
    // ...
})->name('post.index');
```

每次呼叫 `route` 輔助函數時都傳遞 `locale` 會很繁瑣。因此，您可以使用 `URL::defaults` 方法來定義此參數的預設值，該值將始終在當前請求期間應用。您可能希望從一個[路由中介層](/docs/{{version}}/middleware#assigning-middleware-to-routes)中呼叫此方法，以便您可以存取當前請求：

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

一旦 `locale` 參數的預設值設定完成，您就不再需要透過 `route` 輔助函數生成 URL 時傳遞其值。

<a name="url-defaults-middleware-priority"></a>
#### URL 預設值與中介層優先級

設定 URL 預設值可能會干擾 Laravel 處理隱式模型綁定。因此，您應該[優先處理您的中介層](/docs/{{version}}/middleware#sorting-middleware)，使其在 Laravel 自己的 `SubstituteBindings` 中介層之前執行。您可以在應用程式的 `bootstrap/app.php` 檔案中使用 `priority` 中介層方法來實現此目的：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->prependToPriorityList(
        before: \Illuminate\Routing\Middleware\SubstituteBindings::class,
        prepend: \App\Http\Middleware\SetDefaultLocaleForUrls::class,
    );
})
```