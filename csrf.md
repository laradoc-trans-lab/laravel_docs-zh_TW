# CSRF 保護

- [簡介](#csrf-introduction)
- [防範 CSRF 請求](#preventing-csrf-requests)
    - [排除 URI](#csrf-excluding-uris)
- [X-CSRF-Token](#csrf-x-csrf-token)
- [X-XSRF-Token](#csrf-x-xsrf-token)

<a name="csrf-introduction"></a>
## 簡介

跨網站請求偽造是一種惡意攻擊類型，透過它，未經授權的指令會代表已驗證使用者執行。幸運的是，Laravel 讓保護你的應用程式免受 [跨網站請求偽造](https://en.wikipedia.org/wiki/Cross-site_request_forgery) (CSRF) 攻擊變得輕而易舉。


<a name="csrf-explanation"></a>
#### 漏洞說明

假如你不熟悉跨網站請求偽造，讓我們討論一個這個漏洞如何被利用的範例。假設你的應用程式有一個 `/user/email` 路由，它接受 `POST` 請求來更改已驗證使用者的電子郵件地址。最有可能的是，這個路由預期會有一個 `email` 輸入欄位，其中包含使用者想要開始使用的電子郵件地址。

沒有 CSRF 保護的情況下，惡意網站可以建立一個 HTML 表單，指向你應用程式的 `/user/email` 路由並提交惡意使用者自己的電子郵件地址：

```blade
<form action="https://your-application.com/user/email" method="POST">
    <input type="email" value="malicious-email@example.com">
</form>

<script>
    document.forms[0].submit();
</script>
```

假如惡意網站在頁面載入時自動提交表單，惡意使用者只需要誘騙你應用程式中不知情的用戶訪問他們的網站，然後他們的電子郵件地址就會在你的應用程式中被更改。

為了防止這個漏洞，我們需要檢查每一個傳入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 請求，以尋找一個惡意應用程式無法存取的秘密 Session 值。


<a name="preventing-csrf-requests"></a>
## 防範 CSRF 請求

Laravel 會自動為每個由應用程式管理的活躍 [使用者 Session](/docs/{{version}}/session) 產生一個 CSRF「權杖」。這個權杖是用來驗證已驗證使用者確實是向應用程式發出請求的人。由於這個權杖儲存在使用者的 Session 中，並且每次 Session 重新產生時都會改變，惡意應用程式無法存取它。

目前 Session 的 CSRF 權杖可以透過請求的 Session 或 `csrf_token` 輔助函式存取：

    use Illuminate\Http\Request;

    Route::get('/token', function (Request $request) {
        $token = $request->session()->token();

        $token = csrf_token();

        // ...
    });

任何時候當你在應用程式中定義「POST」、「PUT」、「PATCH」或「DELETE」HTML 表單時，你應該在表單中包含一個隱藏的 CSRF `_token` 欄位，讓 CSRF 保護中介層可以驗證請求。為了方便，你可以使用 `@csrf` Blade 指令來產生隱藏的權杖輸入欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    <!-- Equivalent to... -->
    <input type="hidden" name="_token" value="{{ csrf_token() }}" />
</form>
```

`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` [中介層](/docs/{{version}}/middleware) 預設包含在 `web` 中介層群組中，會自動驗證請求輸入中的權杖是否與儲存在 Session 中的權杖相符。當這兩個權杖相符時，我們就知道已驗證使用者才是發起請求的人。


<a name="csrf-tokens-and-spas"></a>
### CSRF 權杖與 SPA

如果你正在建構一個以 Laravel 作為 API 後端的 SPA，你應該查閱 [Laravel Sanctum 文件](/docs/{{version}}/sanctum)，以獲取有關你的 API 認證以及如何防範 CSRF 漏洞的資訊。


<a name="csrf-excluding-uris"></a>
### 排除 URI

有時你可能希望將一組 URI 從 CSRF 保護中排除。例如，如果你正在使用 [Stripe](https://stripe.com) 處理付款並利用其 webhook 系統，你將需要將你的 Stripe webhook 處理路由從 CSRF 保護中排除，因為 Stripe 不會知道要向你的路由發送哪個 CSRF 權杖。

通常，你應該將這些類型的路由放置在 Laravel 應用於 `routes/web.php` 檔案中所有路由的 `web` 中介層群組之外。然而，你也可以透過在應用程式的 `bootstrap/app.php` 檔案中，將其 URI 提供給 `validateCsrfTokens` 方法來排除特定的路由：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->validateCsrfTokens(except: [
            'stripe/*',
            'http://example.com/foo/bar',
            'http://example.com/foo/*',
        ]);
    })

> [!NOTE]  
> 為了方便，當 [執行測試](/docs/{{version}}/testing) 時，CSRF 中介層會自動對所有路由停用。


<a name="csrf-x-csrf-token"></a>
## X-CSRF-Token

除了檢查作為 POST 參數的 CSRF 權杖之外，`Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` 中介層預設包含在 `web` 中介層群組中，也會檢查 `X-CSRF-TOKEN` 請求標頭。舉例來說，你可以將權杖儲存在 HTML `meta` 標籤中：

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

然後，你可以指示像是 jQuery 這樣的函式庫自動將權杖添加到所有請求標頭。這為你的 AJAX 應用程式提供了簡單方便的 CSRF 保護，這些應用程式使用傳統 JavaScript 技術：

```js
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});
```


<a name="csrf-x-xsrf-token"></a>
## X-XSRF-Token

Laravel 會將目前的 CSRF 權杖儲存在加密的 `XSRF-TOKEN` Cookie 中，這個 Cookie 會隨框架產生的每個回應一起發送。你可以使用這個 Cookie 的值來設定 `X-XSRF-TOKEN` 請求標頭。

這個 Cookie 主要作為開發人員的便利措施發送，因為一些 JavaScript 框架和函式庫，例如 Angular 和 Axios，會在同源請求上自動將其值放置在 `X-XSRF-TOKEN` 標頭中。

> [!NOTE]  
> 預設情況下，`resources/js/bootstrap.js` 檔案中包含了 Axios HTTP 函式庫，它會自動為你發送 `X-XSRF-TOKEN` 標頭。