# CSRF 保護

- [簡介](#csrf-introduction)
- [防止 CSRF 請求](#preventing-csrf-requests)
    - [排除特定 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨網站請求偽造 (Cross-site request forgeries) 是一種惡意攻擊，攻擊者會代表已通過身份驗證的使用者執行未經授權的指令。幸運的是，Laravel 讓保護您的應用程式免受 [跨網站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊變得輕而易舉。

<a name="csrf-explanation"></a>
#### 漏洞說明

如果您不熟悉跨網站請求偽造，讓我們討論一個此漏洞如何被利用的範例。想像您的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求來更改已通過身份驗證的使用者電子郵件地址。此路由很可能預期 `email` 輸入欄位包含使用者想要開始使用的電子郵件地址。

如果沒有 CSRF 保護，惡意網站可以建立一個指向您應用程式 `/user/email` 路由的 HTML 表單，並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交表單，惡意使用者只需誘騙您應用程式中毫無戒心的使用者造訪他們的網站，他們的電子郵件地址就會在您的應用程式中被更改。

為了防止此漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，以尋找惡意應用程式無法存取的秘密 Session 值。

<a name="preventing-csrf-requests"></a>
## 防止 CSRF 請求

Laravel 會為應用程式管理的每個活躍 [使用者 Session](/docs/{{version}}/session) 自動產生一個 CSRF「token」。此 token 用於驗證已通過身份驗證的使用者確實是向應用程式發出請求的人。由於此 token 儲存在使用者的 Session 中，並且每次 Session 重新產生時都會更改，因此惡意應用程式無法存取它。

目前 Session 的 CSRF token 可以透過請求的 Session 或 `csrf_token` 輔助函式來存取：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

每當您在應用程式中定義「POST」、「PUT」、「PATCH」或「DELETE」HTML 表單時，您都應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 保護 Middleware 可以驗證請求。為方便起見，您可以使用 ` @csrf` Blade 指令來產生隱藏的 token 輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [Middleware](/docs/{{version}}/middleware) 會自動驗證請求輸入中的 token 是否與 Session 中儲存的 token 相符。當這兩個 token 相符時，我們就知道是已通過身份驗證的使用者發起了請求。

<a name="csrf-tokens-and-spas"></a>
### CSRF Token 與 SPA

如果您正在建構一個使用 Laravel 作為 API 後端的 SPA，您應該查閱 [Laravel Sanctum 說明文件](/docs/{{version}}/sanctum)，以獲取有關使用您的 API 進行身份驗證和防範 CSRF 漏洞的資訊。

<a name="csrf-excluding-uris"></a>
### 從 CSRF 保護中排除 URI

有時您可能希望從 CSRF 保護中排除一組 URI。例如，如果您正在使用 [Stripe](https://stripe.com) 處理付款並利用他們的 webhook 系統，您將需要從 CSRF 保護中排除您的 Stripe webhook 處理程式路由，因為 Stripe 不會知道要向您的路由發送什麼 CSRF token。

通常，您應該將這些類型的路由放置在 Laravel 應用於 `routes/web.php` 檔案中所有路由的 `web` Middleware 群組之外。但是，您也可以透過在應用程式的 `bootstrap/app.php` 檔案中向 `validateCsrfTokens` 方法提供其 URI 來排除特定路由：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
        'http://example.com/foo/bar',
        'http://example.com/foo/*',
    ]);
})
```

> [!NOTE]
> 為方便起見，當 [執行測試](/docs/{{version}}/testing) 時，CSRF Middleware 會自動對所有路由停用。

<a name="csrf-x-csrf-token"></a>
## X-CSRF-TOKEN

除了檢查作為 POST 參數的 CSRF token 之外，預設包含在 `web` Middleware 群組中的 `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` Middleware 也會檢查 `X-CSRF-TOKEN` 請求標頭。例如，您可以將 token 儲存在 HTML `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，您可以指示像 jQuery 這樣的函式庫自動將 token 添加到所有請求標頭中。這為您使用傳統 JavaScript 技術的 AJAX 應用程式提供了簡單、方便的 CSRF 保護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

<a name="csrf-x-xsrf-token"></a>
## X-XSRF-TOKEN

Laravel 會將目前的 CSRF token 儲存在一個加密的 `XSRF-TOKEN` Cookie 中，該 Cookie 會隨框架產生的每個回應一起包含。您可以使用 Cookie 值來設定 `X-XSRF-TOKEN` 請求標頭。

此 Cookie 主要作為開發人員的便利性而發送，因為某些 JavaScript 框架和函式庫，例如 Angular 和 Axios，會自動將其值放置在同源請求的 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]
> 預設情況下，`resources/js/bootstrap.js` 檔案包含 Axios HTTP 函式庫，它會自動為您發送 `X-XSRF-TOKEN` 標頭。
