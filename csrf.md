# CSRF 保護

- [簡介](#csrf-introduction)
- [預防 CSRF 請求](#preventing-csrf-requests)
    - [排除 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨站請求偽造 (Cross-site request forgeries) 是一種惡意攻擊類型，透過它，未經授權的指令會代表已驗證使用者執行。幸運地，Laravel 讓保護你的應用程式免受 [跨站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊變得容易。

<a name="csrf-explanation"></a>
#### 漏洞解釋

如果你不熟悉跨站請求偽造，讓我們討論一個此漏洞如何被利用的範例。想像你的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求以更改已驗證使用者的電子郵件地址。很可能，這個路由預期一個 `email` 輸入欄位包含使用者想要開始使用的電子郵件地址。

如果沒有 CSRF 保護，一個惡意網站可以建立一個 HTML 表單，指向你的應用程式的 `/user/email` 路由，並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交表單，惡意使用者只需要誘騙一個毫無戒心的應用程式使用者造訪他們的網站，他們的電子郵件地址就會在你的應用程式中被更改。

為了預防這個漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求中是否存在一個惡意應用程式無法存取的秘密 Session 值。

<a name="preventing-csrf-requests"></a>
## 預防 CSRF 請求

Laravel 會自動為每個應用程式管理的活躍 [使用者 Session](/docs/{{version}}/session) 生成一個 CSRF「權杖」。這個權杖用於驗證已驗證使用者是實際向應用程式發出請求的人。由於這個權杖儲存在使用者的 Session 中，並且每次 Session 重新生成時都會改變，因此惡意應用程式無法存取它。

當前 Session 的 CSRF 權杖可以透過請求的 Session 或 `csrf_token` 輔助函式存取：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

任何時候你在應用程式中定義「POST」、「PUT」、「PATCH」或「DELETE」HTML 表單時，都應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 保護中介層可以驗證請求。為了方便起見，你可以使用 `@csrf` Blade 指令來生成隱藏的權杖輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [中介層](/docs/{{version}}/middleware)，它預設包含在 `web` 中介層群組中，會自動驗證請求輸入中的權杖是否與 Session 中儲存的權杖匹配。當這兩個權杖匹配時，我們就知道已驗證使用者是發起該請求的人。

<a name="csrf-tokens-and-spas"></a>
### CSRF 權杖與 SPA

如果你正在建立一個利用 Laravel 作為 API 後端的 SPA (單頁應用程式)，你應該查閱 [Laravel Sanctum 文件](/docs/{{version}}/sanctum)，以獲取關於如何透過 API 進行驗證以及如何保護免受 CSRF 漏洞攻擊的資訊。

<a name="csrf-excluding-uris"></a>
### 排除 URI

有時候你可能希望將某些 URI 排除在 CSRF 保護之外。例如，如果你正在使用 [Stripe](https://stripe.com) 處理支付，並利用他們的 Webhook 系統，你需要將你的 Stripe Webhook 處理路由排除在 CSRF 保護之外，因為 Stripe 不會知道要向你的路由發送什麼 CSRF 權杖。

通常，你應該將這類路由放置在 Laravel 套用到 `routes/web.php` 檔案中所有路由的 `web` 中介層群組之外。然而，你也可以透過將其 URI 提供給應用程式 `bootstrap/app.php` 檔案中的 `validateCsrfTokens` 方法來排除特定路由：

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
> 為了方便，當 [執行測試](/docs/{{version}}/testing) 時，CSRF 中介層會自動停用所有路由。

<a name="csrf-x-csrf-token"></a>
## X-CSRF-Token

除了檢查作為 POST 參數的 CSRF 權杖之外，`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` 中介層（預設包含在 `web` 中介層群組中）也會檢查 `X-CSRF-TOKEN` 請求標頭。例如，你可以將權杖儲存在 HTML `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，你可以指示像 jQuery 這樣的函式庫自動將權杖加入到所有請求標頭中。這為使用傳統 JavaScript 技術的 AJAX 應用程式提供了簡單、方便的 CSRF 保護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

<a name="csrf-x-xsrf-token"></a>
## X-XSRF-Token

Laravel 會將當前的 CSRF 權杖儲存在一個加密的 `XSRF-TOKEN` cookie 中，該 cookie 會隨框架生成的每個回應一起傳送。你可以使用該 cookie 值來設定 `X-XSRF-TOKEN` 請求標頭。

這個 cookie 主要作為開發者便利性而發送，因為一些 JavaScript 框架和函式庫，例如 Angular 和 Axios，會自動將其值放在同源請求的 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]
> 預設情況下，`resources/js/bootstrap.js` 檔案會包含 Axios HTTP 函式庫，它會自動為你發送 `X-XSRF-TOKEN` 標頭。