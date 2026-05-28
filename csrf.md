# CSRF 防護

- [介紹](#csrf-introduction)
- [防止 CSRF 請求](#preventing-csrf-requests)
    - [來源驗證](#origin-verification)
    - [排除 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 介紹

跨站請求偽造（Cross-site request forgery）是一種惡意攻擊手法，藉此代表已認證的使用者執行未授權的指令。幸好，Laravel 讓你能輕鬆保護應用程式免受 [跨站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊。


<a name="csrf-explanation"></a>
#### 漏洞原理解析

如果你還不熟悉跨站請求偽造，讓我們來看一個如何利用此漏洞的範例。假設你的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求來修改已認證使用者的 E-mail 地址。這個路由通常會預期有一個 `email` 輸入欄位，裡面包含使用者想要開始使用的新 E-mail 地址。

在沒有 CSRF 防護的情況下，惡意網站可以建立一個指向你應用程式 `/user/email` 路由的 HTML 表單，並提交惡意使用者自己的 E-mail 地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交該表單，惡意使用者只需要誘騙你應用程式的無辜使用者訪問他們的網站，該使用者的 E-mail 地址就會在你的應用程式中被修改。

為了防止這種漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，確認其中是否包含惡意應用程式無法存取的私密 Session 值。


<a name="preventing-csrf-requests"></a>
## 防止 CSRF 請求

預設包含在 `web` 中介層群組中的 `Illuminate\Foundation\Http\Middleware\PreventRequestForgery` [中介層](/docs/{{version}}/middleware)，採用了雙層防禦機制來保護你的應用程式免受跨站請求偽造。

第一步，中介層會檢查瀏覽器的 `Sec-Fetch-Site` 標頭。現代瀏覽器會在每個請求上自動設定此標頭，用以標示該請求是來自同源（Same-origin）、同站（Same-site）還是跨來源（Cross-site）。如果標頭顯示請求來自同源，則該請求會立即被允許，無需進行任何 CSRF Token 驗證。

如果來源驗證未通過——例如，因為請求來自不發送 `Sec-Fetch-Site` 標頭的舊版瀏覽器，或者因為連線不安全——中介層將退回使用傳統的 CSRF Token 驗證。

Laravel 會為應用程式管理的每個活動 [使用者 Session](/docs/{{version}}/session) 自動產生一個 CSRF 「token」。此 token 用於驗證已認證的使用者是否為實際向應用程式發送請求的本人。由於此 token 儲存在使用者的 Session 中，且每次 Session 重建時都會變更，因此惡意應用程式無法存取它。

目前 Session 的 CSRF Token 點可以透過請求的 Session 或透過 `csrf_token` 輔助函式來存取：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

每當你在應用程式中定義 "POST"、"PUT"、"PATCH" 或 "DELETE" 的 HTML 表單時，都應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 防護中介層能夠驗證該請求。為了方便起見，你可以使用 `@csrf` Blade 指令來產生隱藏的 token 輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```


<a name="csrf-tokens-and-spas"></a>
#### CSRF Token 與 SPA

如果你正在建置以 Laravel 作為 API 後端的 SPA，你應該參考 [Laravel Sanctum 文件](/docs/{{version}}/sanctum)，以了解如何與 API 進行認證並防止 CSRF 漏洞。


<a name="origin-verification"></a>
### 來源驗證

如上所述，Laravel 的請求偽造中介層首先會檢查 `Sec-Fetch-Site` 標頭，以判斷請求是否來自同源。預設情況下，如果此檢查未通過，中介層將退回使用 CSRF Token 驗證。

然而，如果你希望完全依賴來源驗證並徹底停用 CSRF Token 的退回機制，可以在應用程式的 `bootstrap/app.php` 檔案中使用 `preventRequestForgery` 方法：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(originOnly: true);
})
```

使用僅限來源（Origin-only）模式時，未通過來源驗證的請求將會收到 `403` HTTP 回應，而不是通常與 CSRF Token 不符相關的 `419` 回應。

> [!WARNING]
> `Sec-Fetch-Site` 標頭僅由瀏覽器透過安全（HTTPS）連線發送。如果你的應用程式未使用 HTTPS 服務，來源驗證將無法運作，中介層將退回使用 CSRF Token 驗證。

如果你的應用程式需要接受來自子網域的請求（例如：`dashboard.example.com` 接受來自 `example.com` 的請求），除了同源請求之外，你也可以允許同站（Same-site）請求：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(allowSameSite: true);
})
```


<a name="csrf-excluding-uris"></a>
### 排除 URI

有時你可能希望將一組 URI 排除在 CSRF 防護之外。例如，如果你使用 [Stripe](https://stripe.com) 來處理付款並使用他們的 Webhook 系統，你將需要把 Stripe Webhook 處理程式路由排除在 CSRF 防護之外，因為 Stripe 不會知道要發送什麼 CSRF Token 給你的路由。

通常，你應該將此類路由放在 Laravel 套用到 `routes/web.php` 檔案中所有路由的 `web` 中介層群組之外。然而，你也可以透過在應用程式的 `bootstrap/app.php` 檔案中將特定路由的 URI 傳遞給 `preventRequestForgery` 方法來排除它們：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(except: [
        'stripe/*',
        'http://example.com/foo/bar',
        'http://example.com/foo/*',
    ]);
})
```

> [!NOTE]
> 為了方便起見，在 [執行測試](/docs/{{version}}/testing) 時，所有路由的 CSRF 中介層都會自動停用。


<a name="csrf-x-csrf-token"></a>
## X-CSRF-Token

除了將 CSRF Token 作為 POST 參數進行檢查之外，`PreventRequestForgery` 中介層還會檢查 `X-CSRF-TOKEN` 請求標頭。例如，你可以將 Token 儲存在 HTML 的 `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，你可以指示像 jQuery 這樣的函式庫自動將 Token 新增到所有請求標頭中。這為使用舊版 JavaScript 技術的 AJAX 應用程式提供了簡單且便利的 CSRF 防護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```


<a name="csrf-x-xsrf-token"></a>
## X-XSRF-Token

Laravel 將目前的 CSRF Token 儲存在加密的 `XSRF-TOKEN` Cookie 中，該 Cookie 會包含在框架產生的每個回應中。你可以使用此 Cookie 值來設定 `X-XSRF-TOKEN` 請求標頭。

發送此 Cookie 主要是在開發上提供便利，因為某些 JavaScript 框架與函式庫（如 Angular 和 Axios）會自動在同源請求的 `X-XSRF-TOKEN` 標頭中帶入其值。