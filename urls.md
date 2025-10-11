# URL 生成

- [簡介](#introduction)
- [基礎概念](#the-basics)
    - [生成 URL](#generating-urls)
    - [存取目前的 URL](#accessing-the-current-url)
- [命名路由的 URL](#urls-for-named-routes)
    - [簽章 URL](#signed-urls)
- [控制器動作的 URL](#urls-for-controller-actions)
- [預設值](#default-values)

<a name="introduction"></a>
## 簡介

Laravel 提供了多個輔助函式來幫助你為應用程式生成 URL。這些輔助函式主要在你的模板和 API 回應中建立連結，或在生成重新導向回應到應用程式的另一個部分時非常有用。

<a name="the-basics"></a>
## 基礎概念

<a name="generating-urls"></a>
### 生成 URL

`url` 輔助函式可用於為你的應用程式生成任意 URL。生成的 URL 將會自動使用應用程式正在處理的當前請求中的協定 (HTTP 或 HTTPS) 和主機：

    $post = App\Models\Post::find(1);

    echo url("/posts/{$post->id}");

    // http://example.com/posts/1

要生成帶有查詢字串參數的 URL，你可以使用 `query` 方法：

    echo url()->query('/posts', ['search' => 'Laravel']);

    // https://example.com/posts?search=Laravel

    echo url()->query('/posts?sort=latest', ['search' => 'Laravel']);

    // http://example.com/posts?sort=latest&search=Laravel

提供已存在於路徑中的查詢字串參數將會覆寫它們的現有值：

    echo url()->query('/posts?sort=latest', ['sort' => 'oldest']);

    // http://example.com/posts?sort=oldest

值陣列也可以作為查詢參數傳遞。這些值將會被正確地鍵控並編碼到生成的 URL 中：

    echo $url = url()->query('/posts', ['columns' => ['title', 'body']]);

    // http://example.com/posts?columns%5B0%5D=title&columns%5B1%5D=body

    echo urldecode($url);

    // http://example.com/posts?columns[0]=title&columns[1]=body

<a name="accessing-the-current-url"></a>
### 存取目前的 URL

如果未提供路徑給 `url` 輔助函式，則會回傳一個 `Illuminate\Routing\UrlGenerator` 實例，允許你存取有關目前 URL 的資訊：

    // Get the current URL without the query string...
    echo url()->current();

    // Get the current URL including the query string...
    echo url()->full();

    // Get the full URL for the previous request...
    echo url()->previous();

    // Get the path for the previous request...
    echo url()->previousPath();

這些方法中的每一個也可以透過 `URL` [Facade](/docs/{{version}}/facades) 存取：

    use Illuminate\Support\Facades\URL;

    echo URL::current();

<a name="urls-for-named-routes"></a>
## 命名路由的 URL

`route` 輔助函式可用於生成指向[命名路由](/docs/{{version}}/routing#named-routes)的 URL。命名路由允許你在生成 URL 時，無須與路由上定義的實際 URL 綁定。因此，如果路由的 URL 變更，無須修改你對 `route` 函式的呼叫。例如，想像你的應用程式包含以下定義的路由：

    Route::get('/post/{post}', function (Post $post) {
        // ...
    })->name('post.show');

要生成此路由的 URL，你可以這樣使用 `route` 輔助函式：

    echo route('post.show', ['post' => 1]);

    // http://example.com/post/1

當然，`route` 輔助函式也可以用於生成具有多個參數的路由的 URL：

    Route::get('/post/{post}/comment/{comment}', function (Post $post, Comment $comment) {
        // ...
    })->name('comment.show');

    echo route('comment.show', ['post' => 1, 'comment' => 3]);

    // http://example.com/post/1/comment/3

任何額外的陣列元素，若不對應於路由的定義參數，將會被新增到 URL 的查詢字串：

    echo route('post.show', ['post' => 1, 'search' => 'rocket']);

    // http://example.com/post/1?search=rocket

<a name="eloquent-models"></a>
#### Eloquent 模型

你通常會使用 [Eloquent 模型](/docs/{{version}}/eloquent)的路由鍵 (通常是主鍵) 來生成 URL。為此，你可以將 Eloquent 模型作為參數值傳遞。`route` 輔助函式將會自動提取模型的路由鍵：

    echo route('post.show', ['post' => $post]);

<a name="signed-urls"></a>
### 簽章 URL

Laravel 允許你輕鬆建立指向命名路由的「簽章」URL。這些 URL 的查詢字串中會附加一個「簽章」雜湊值，這允許 Laravel 驗證 URL 自建立以來未被修改。簽章 URL 對於公開可存取但需要一層保護以防止 URL 遭到操縱的路由特別有用。

例如，你可以使用簽章 URL 來實作一個公開的「取消訂閱」連結，該連結會透過電子郵件寄送給你的客戶。要建立一個指向命名路由的簽章 URL，請使用 `URL` Facade 的 `signedRoute` 方法：

    use Illuminate\Support\Facades\URL;

    return URL::signedRoute('unsubscribe', ['user' => 1]);

你可以透過向 `signedRoute` 方法提供 `absolute` 引數來從簽章 URL 雜湊值中排除網域：

    return URL::signedRoute('unsubscribe', ['user' => 1], absolute: false);

如果你想生成一個在指定時間後過期的暫時簽章路由 URL，你可以使用 `temporarySignedRoute` 方法。當 Laravel 驗證一個暫時簽章路由 URL 時，它將會確保編碼到簽章 URL 中的過期時間戳記尚未過期：

    use Illuminate\Support\Facades\URL;

    return URL::temporarySignedRoute(
        'unsubscribe', now()->addMinutes(30), ['user' => 1]
    );

<a name="validating-signed-route-requests"></a>
#### 驗證簽章路由請求

要驗證傳入請求是否具有有效的簽章，你應該在傳入的 `Illuminate\Http\Request` 實例上呼叫 `hasValidSignature` 方法：

    use Illuminate\Http\Request;

    Route::get('/unsubscribe/{user}', function (Request $request) {
        if (! $request->hasValidSignature()) {
            abort(401);
        }

        // ...
    })->name('unsubscribe');

有時，你可能需要允許應用程式的前端將資料附加到簽章 URL，例如在執行客戶端分頁時。因此，你可以使用 `hasValidSignatureWhileIgnoring` 方法指定在驗證簽章 URL 時應該被忽略的請求查詢參數。請記住，忽略參數允許任何人修改請求上的這些參數：

    if (! $request->hasValidSignatureWhileIgnoring(['page', 'order'])) {
        abort(401);
    }

除了使用傳入請求實例來驗證簽章 URL 之外，你也可以將 `signed` (`Illuminate\Routing\Middleware\ValidateSignature`) [中介層](/docs/{{version}}/middleware)指定給路由。如果傳入請求沒有有效的簽章，中介層將會自動回傳 `403` HTTP 回應：

    Route::post('/unsubscribe/{user}', function (Request $request) {
        // ...
    })->name('unsubscribe')->middleware('signed');

如果你的簽章 URL 不包含 URL 雜湊中的網域，你應該向中介層提供 `relative` 引數：

    Route::post('/unsubscribe/{user}', function (Request $request) {
        // ...
    })->name('unsubscribe')->middleware('signed:relative');

<a name="responding-to-invalid-signed-routes"></a>
#### 回應無效的簽章路由

當有人造訪已過期的簽章 URL 時，他們將會收到一個 `403` HTTP 狀態碼的通用錯誤頁面。但是，你可以透過在應用程式的 `bootstrap/app.php` 檔案中為 `InvalidSignatureException` 異常定義一個自訂的「渲染」閉包來客製化此行為：

    use Illuminate\Routing\Exceptions\InvalidSignatureException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (InvalidSignatureException $e) {
            return response()->view('errors.link-expired', status: 403);
        });
    })

<a name="urls-for-controller-actions"></a>
## 控制器動作的 URL

`action` 函式會為指定的控制器動作生成 URL：

    use App\Http\Controllers\HomeController;

    $url = action([HomeController::class, 'index']);

如果控制器方法接受路由參數，您可以將路由參數的關聯陣列作為第二個引數傳遞給該函式：

    $url = action([UserController::class, 'profile'], ['id' => 1]);


<a name="default-values"></a>
## 預設值

對於某些應用程式，您可能希望為特定的 URL 參數指定全請求範圍的預設值。例如，假設您的許多路由定義了一個 `{locale}` 參數：

    Route::get('/{locale}/posts', function () {
        // ...
    })->name('post.index');

每次呼叫 `route` 輔助函式時都傳遞 `locale` 會很麻煩。因此，您可以使用 `URL::defaults` 方法來定義此參數的預設值，該預設值將始終在目前請求期間套用。您可能希望從 [路由中介層](/docs/{{version}}/middleware#assigning-middleware-to-routes) 呼叫此方法，以便您可以存取目前的請求：

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

設定 `locale` 參數的預設值後，您在使用 `route` 輔助函式生成 URL 時就不再需要傳遞其值。


<a name="url-defaults-middleware-priority"></a>
#### URL 預設值與中介層優先順序

設定 URL 預設值可能會干擾 Laravel 處理隱式模型綁定。因此，您應該 [優先處理您的中介層](/docs/{{version}}/middleware#sorting-middleware)，讓它們在 Laravel 自身的 `SubstituteBindings` 中介層之前執行。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `priority` 中介層方法來實現此目的：

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->prependToPriorityList(
        before: \Illuminate\Routing\Middleware\SubstituteBindings::class,
        prepend: \App\Http\Middleware\SetDefaultLocaleForUrls::class,
    );
})
```