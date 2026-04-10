# CSRF 防護

- [簡介](#csrf-introduction)
- [防止 CSRF 請求](#preventing-csrf-requests)
    - [來源驗證](#origin-verification)
    - [排除 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨站請求偽造 (Cross-site request forgeries) 是一種惡意利用方式，攻擊者會以已認證使用者的名義執行未經授權的指令。幸運的是，Laravel 讓您可以輕鬆地保護您的應用程式免受 [跨站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊。

<a name="csrf-explanation"></a>
#### 漏洞說明

如果您不熟悉跨站請求偽造，讓我們來討論一個此漏洞如何被利用的範例。假設您的應用程式有一個 `/user/email` 路由，該路由接受 `POST` 請求以更改已認證使用者的電子郵件地址。最有可能的情況是，此路由預期一個 `email` 輸入欄位包含使用者想要開始使用的電子郵件地址。

如果沒有 CSRF 防護，惡意網站可以建立一個指向您應用程式 `/user/email` 路由的 HTML 表單，並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

如果惡意網站在頁面載入時自動提交表單，惡意使用者只需要誘騙您應用程式中不覺的使用者訪問其網站，該使用者的電子郵件地址就會在您的應用程式中被更改。

為了防止此漏洞，我們需要檢查每個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，確認其是否包含一個惡意應用程式無法存取的秘密會話值。

<a name="preventing-csrf-requests"></a>
## 防止 CSRF 請求

`Illuminate\Foundation\Http\Middleware\PreventRequestForgery` [中介層](/docs/{{version}}/middleware)（預設包含在 `web` 中介層群組中）使用雙層方法來保護您的應用程式免受跨站請求偽造攻擊。

首先，該中介層會檢查瀏覽器的 `Sec-Fetch-Site` 標頭。現代瀏覽器會在每次請求時自動設置此標頭，用以指示請求是來自於同一來源 (same origin)、同一站台 (same site) 還是跨站來源 (cross-site source)。如果標頭指示請求來自同一來源，則請求將立即獲准，無需進行任何令牌驗證。

如果來源驗證未通過 —— 例如，因為請求來自不傳送 `Sec-Fetch-Site` 標頭的舊版瀏覽器，或者因為連線不安全 —— 則中介層會回退至傳統的 CSRF 令牌驗證。

Laravel 會為應用程式管理的每個啟動中的 [使用者會話](/docs/{{version}}/session) 自動生成一個 CSRF 「令牌 (token)」。此令牌用於驗證已認證的使用者確實是向應用程式發出請求的人。由於此令牌儲存在使用者的會話中，且每次會話重新產生時都會更改，因此惡意應用程式無法存取它。

目前會話的 CSRF 令牌可以透過請求的會話或透過 `csrf_token` 輔助函式來存取：

```php
use Illuminate\Http\Request;

Route::get('/token', function (Request $request) {
    $token = $request->session()->token();

    $token = csrf_token();

    // ...
});
```

每當您在應用程式中定義 「POST」、「PUT」、「PATCH」 或 「DELETE」 HTML 表單時，您應該在表單中包含一個隱藏的 CSRF `_token` 欄位，以便 CSRF 防護中介層可以驗證該請求。為了方便起見，您可以使用 `@csrf` Blade 指令來生成隱藏的令牌輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

<a name="csrf-tokens-and-spas"></a>
#### CSRF 令牌與 SPA

如果您正在構建一個將 Laravel 作為 API 後端的單頁應用程式 (SPA)，您應該參閱 [Laravel Sanctum 文件](/docs/{{version}}/sanctum) 以獲取有關 API 認證以及防止 CSRF 漏洞的資訊。

<a name="origin-verification"></a>
### 來源驗證

如上所述，Laravel 的請求偽造中介層首先會檢查 `Sec-Fetch-Site` 標頭，以確定請求是否來自同一來源。預設情況下，如果此檢查未通過，中介層會回退至 CSRF 令牌驗證。

然而，如果您希望完全依賴來源驗證並完全禁用 CSRF 令牌回退，可以使用應用程式 `bootstrap/app.php` 檔案中的 `preventRequestForgery` 方法來實現：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(originOnly: true);
})
```

在使用僅限來源模式時，未能通過來源驗證的請求將收到 `403` HTTP 回應，而非通常與 CSRF 令牌不匹配相關聯的 `419` 回應。

> [!WARNING]
> `Sec-Fetch-Site` 標頭僅由瀏覽器透過安全 (HTTPS) 連線發送。如果您的應用程式未透過 HTTPS 提供服務，則無法使用來源驗證，中介層將回退至 CSRF 令牌驗證。

如果您的應用程式需要接受來自子網域的請求（例如，`dashboard.example.com` 接受來自 `example.com` 的請求），除了同一來源請求外，您還可以允許同一站台請求：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(allowSameSite: true);
})
```

<a name="csrf-excluding-uris"></a>
### 排除 URI

有時您可能希望將一組 URI 從 CSRF 防護中排除。例如，如果您使用 [Stripe](https://stripe.com) 處理付款並利用其 Webhook 系統，您需要將 Stripe Webhook 處理路由從 CSRF 防護中排除，因為 Stripe 不知道要向您的路由發送哪個 CSRF 令牌。

通常，您應該將這類路由放置在 Laravel 應用於 `routes/web.php` 檔案中所有路由的 `web` 中介層群組之外。不過，您也可以透過在應用程式的 `bootstrap/app.php` 檔案中將其 URI 提供給 `preventRequestForgery` 方法來排除特定路由：

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
> 為了方便起見，在 [執行測試](/docs/{{version}}/testing) 時，所有路由的 CSRF 中介層都會自動禁用。

<a name="csrf-x-csrf-token"></a>
## X-CSRF-Token

除了將 CSRF 令牌作為 POST 參數檢查外，`PreventRequestForgery` 中介層還會檢查 `X-CSRF-TOKEN` 請求標頭。例如，您可以將令牌儲存在 HTML 的 `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，您可以指示像 jQuery 這樣的函式庫自動將令牌添加到所有請求標頭中。這為使用舊版 JavaScript 技術的 AJAX 應用程式提供了簡單且方便的 CSRF 防護：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```

<a name="csrf-x-xsrf-token"></a>
## X-XSRF-Token

Laravel 將目前的 CSRF 令牌儲存在加密的 `XSRF-TOKEN` Cookie 中，該 Cookie 包含在框架生成的每個回應中。您可以使用此 Cookie 值來設置 `X-XSRF-TOKEN` 請求標頭。

此 Cookie 主要是為了開發者的便利而發送的，因為某些 JavaScript 框架和函式庫（如 Angular 和 Axios）在同一來源請求時，會自動將其值放入 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]
> 預設情況下，`resources/js/bootstrap.js` 檔案包含 Axios HTTP 函式庫，它會自動為您發送 `X-XSRF-TOKEN` 標頭。