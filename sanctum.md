# Laravel Sanctum

- [簡介](#introduction)
    - [運作方式](#how-it-works)
- [安裝](#installation)
- [配置](#configuration)
    - [覆寫預設模型](#overriding-default-models)
- [API 權杖驗證](#api-token-authentication)
    - [發行 API 權杖](#issuing-api-tokens)
    - [權杖能力](#token-abilities)
    - [保護路由](#protecting-routes)
    - [撤銷權杖](#revoking-tokens)
    - [權杖過期](#token-expiration)
- [SPA 驗證](#spa-authentication)
    - [配置](#spa-configuration)
    - [驗證](#spa-authenticating)
    - [保護路由](#protecting-spa-routes)
    - [授權私有廣播頻道](#authorizing-private-broadcast-channels)
- [行動應用程式驗證](#mobile-application-authentication)
    - [發行 API 權杖](#issuing-mobile-api-tokens)
    - [保護路由](#protecting-mobile-api-routes)
    - [撤銷權杖](#revoking-mobile-api-tokens)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Sanctum](https://github.com/laravel/sanctum) 為 SPA (單頁式應用程式)、行動應用程式以及簡單的、基於權杖的 API 提供了一個輕量級驗證系統。Sanctum 允許您應用程式的每個使用者為其帳號產生多個 API 權杖。這些權杖可以被授予能力 / 作用域 (scopes)，以指定權杖被允許執行哪些操作。

<a name="how-it-works"></a>
### 運作方式

Laravel Sanctum 旨在解決兩個獨立的問題。在深入探討這個函式庫之前，讓我們先討論每個問題。

<a name="how-it-works-api-tokens"></a>
#### API 權杖

首先，Sanctum 是一個簡單的套件，您可以用它來向使用者發行 API 權杖，而無需 OAuth 的複雜性。此功能受 GitHub 及其他發行「個人存取權杖」的應用程式所啟發。例如，想像您應用程式的「帳號設定」中有一個畫面，使用者可以在其中為其帳號產生 API 權杖。您可以使用 Sanctum 來產生並管理這些權杖。這些權杖通常具有非常長的過期時間 (數年)，但使用者可以隨時手動撤銷。

Laravel Sanctum 提供此功能的方式是將使用者 API 權杖儲存在單一資料庫表格中，並透過 `Authorization` 標頭驗證傳入的 HTTP 請求，該標頭應包含有效的 API 權杖。

<a name="how-it-works-spa-authentication"></a>
#### SPA 驗證

其次，Sanctum 旨在提供一種簡單的方式來驗證需要與基於 Laravel 的 API 進行通訊的單頁式應用程式 (SPA)。這些 SPA 可能與您的 Laravel 應用程式存在於同一個儲存庫中，或者可能是一個完全獨立的儲存庫，例如使用 Next.js 或 Nuxt 建立的 SPA。

對於此功能，Sanctum 不使用任何形式的權杖。相反地，Sanctum 使用 Laravel 內建的基於 Cookie 的 Session 驗證服務。通常，Sanctum 會利用 Laravel 的 `web` 驗證守衛 (guard) 來達成此目的。這提供了 CSRF 保護、Session 驗證的好處，並防止透過 XSS 洩露驗證憑證。

Sanctum 僅在傳入請求源自於您自己的 SPA 前端時，才會嘗試使用 Cookie 進行驗證。當 Sanctum 檢查傳入的 HTTP 請求時，它會首先檢查驗證 Cookie，如果沒有，則 Sanctum 會檢查 `Authorization` 標頭以尋找有效的 API 權杖。

> [!NOTE]  
> 完全可以只將 Sanctum 用於 API 權杖驗證或只用於 SPA 驗證。僅僅因為您使用 Sanctum，並不意味著您必須使用它提供的所有功能。

<a name="installation"></a>
## 安裝

您可以透過 `install:api` Artisan 命令安裝 Laravel Sanctum：

```shell
php artisan install:api
```

接下來，如果您打算利用 Sanctum 來驗證 SPA，請參閱本文檔的 [SPA 驗證](#spa-authentication) 部分。

<a name="configuration"></a>
## 配置

<a name="overriding-default-models"></a>
### 覆寫預設模型

雖然通常不需要，但您可以自由擴展 Sanctum 內部使用的 `PersonalAccessToken` 模型：

    use Laravel\Sanctum\PersonalAccessToken as SanctumPersonalAccessToken;

    class PersonalAccessToken extends SanctumPersonalAccessToken
    {
        // ...
    }

然後，您可以透過 Sanctum 提供的 `usePersonalAccessTokenModel` 方法指示 Sanctum 使用您的自訂模型。通常，您應該在應用程式的 `AppServiceProvider` 檔案的 `boot` 方法中呼叫此方法：

    use App\Models\Sanctum\PersonalAccessToken;
    use Laravel\Sanctum\Sanctum;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Sanctum::usePersonalAccessTokenModel(PersonalAccessToken::class);
    }

<a name="api-token-authentication"></a>
## API 權杖驗證

> [!NOTE]  
> 您不應該使用 API 權杖來驗證您自己的第一方 SPA。相反地，請使用 Sanctum 內建的 [SPA 驗證功能](#spa-authentication)。

<a name="issuing-api-tokens"></a>
### 發行 API 權杖

Sanctum 允許您發行 API 權杖 / 個人存取權杖，這些權杖可用於驗證對您應用程式的 API 請求。當使用 API 權杖發出請求時，該權杖應包含在 `Authorization` 標頭中，作為 `Bearer` 權杖。

要開始為使用者發行權杖，您的 User 模型應使用 `Laravel\Sanctum\HasApiTokens` trait：

    use Laravel\Sanctum\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, HasFactory, Notifiable;
    }

要發行權杖，您可以使用 `createToken` 方法。`createToken` 方法會回傳一個 `Laravel\Sanctum\NewAccessToken` 實例。API 權杖在儲存到資料庫之前會使用 SHA-256 雜湊處理，但您可以使用 `NewAccessToken` 實例的 `plainTextToken` 屬性來存取權杖的純文字值。您應在權杖建立後立即將此值顯示給使用者：

    use Illuminate\Http\Request;

    Route::post('/tokens/create', function (Request $request) {
        $token = $request->user()->createToken($request->token_name);

        return ['token' => $token->plainTextToken];
    });

您可以使用 `HasApiTokens` trait 提供的 `tokens` Eloquent 關聯來存取使用者所有的權杖：

    foreach ($user->tokens as $token) {
        // ...
    }

<a name="token-abilities"></a>
### 權杖能力

Sanctum 允許您為權杖分配「能力」(abilities)。能力與 OAuth 的「範圍」(scopes) 有相似的目的。您可以將一個字串能力陣列作為 `createToken` 方法的第二個參數傳遞：

    return $user->createToken('token-name', ['server:update'])->plainTextToken;

當處理經 Sanctum 驗證的傳入請求時，您可以使用 `tokenCan` 或 `tokenCant` 方法來判斷權杖是否具有指定能力：

    if ($user->tokenCan('server:update')) {
        // ...
    }

    if ($user->tokenCant('server:update')) {
        // ...
    }

<a name="token-ability-middleware"></a>
#### 權杖能力中介層

Sanctum 還包含兩個中介層，可用於驗證傳入請求是否已使用授予特定能力的權杖進行驗證。首先，在您應用程式的 `bootstrap/app.php` 檔案中定義以下中介層別名：

    use Laravel\Sanctum\Http\Middleware\CheckAbilities;
    use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'abilities' => CheckAbilities::class,
            'ability' => CheckForAnyAbility::class,
        ]);
    })

`abilities` 中介層可以被指派給路由，以驗證傳入請求的權杖是否具有所有列出的能力：

    Route::get('/orders', function () {
        // Token has both "check-status" and "place-orders" abilities...
    })->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);

`ability` 中介層可以被指派給路由，以驗證傳入請求的權杖是否具有*至少其中一項*列出的能力：

    Route::get('/orders', function () {
        // Token has the "check-status" or "place-orders" ability...
    })->middleware(['auth:sanctum', 'ability:check-status,place-orders']);

<a name="first-party-ui-initiated-requests"></a>
#### 第一方 UI 發起的請求

為方便起見，如果傳入的驗證請求來自您的第一方 SPA 並且您正在使用 Sanctum 內建的 [SPA 驗證](#spa-authentication) 功能，則 `tokenCan` 方法將始終回傳 `true`。

然而，這並不意味著您的應用程式必須允許使用者執行該動作。通常，您應用程式的 [授權策略](/docs/{{version}}/authorization#creating-policies) 將會決定權杖是否已被授予執行能力的權限，並檢查使用者實例本身是否應被允許執行該動作。

例如，如果我們想像一個管理伺服器的應用程式，這可能意味著檢查權杖是否被授權更新伺服器 **並且** 該伺服器屬於該使用者：

```php
return $request->user()->id === $server->user_id &&
       $request->user()->tokenCan('server:update')
```

起初，允許 `tokenCan` 方法被呼叫並始終對第一方 UI 發起的請求回傳 `true` 可能看起來很奇怪；然而，能夠始終假設 API 權杖可用並可透過 `tokenCan` 方法進行檢查是很方便的。透過這種方法，您可以在應用程式的授權策略中始終呼叫 `tokenCan` 方法，而無需擔心請求是由您的應用程式 UI 觸發，還是由您的 API 第三方消費者發起。

<a name="protecting-routes"></a>
### 保護路由

為了保護路由以使所有傳入請求都必須經過驗證，您應該將 `sanctum` 驗證守衛附加到您在 `routes/web.php` 和 `routes/api.php` 路由檔案中的受保護路由。此守衛將確保傳入請求被驗證為有狀態、經 cookie 驗證的請求，或者如果請求來自第三方，則包含有效的 API 權杖標頭。

您可能想知道為什麼我們建議您使用 `sanctum` 守衛來驗證應用程式 `routes/web.php` 檔案中的路由。請記住，Sanctum 將首先嘗試使用 Laravel 典型的會話驗證 cookie 來驗證傳入請求。如果該 cookie 不存在，Sanctum 將嘗試使用請求的 `Authorization` 標頭中的權杖來驗證請求。此外，使用 Sanctum 驗證所有請求可確保我們始終可以對當前驗證的使用者實例呼叫 `tokenCan` 方法：

    use Illuminate\Http\Request;

    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('auth:sanctum');

<a name="revoking-tokens"></a>
### 撤銷權杖

您可以透過使用 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯，從資料庫中刪除權杖來「撤銷」它們：

    // Revoke all tokens...
    $user->tokens()->delete();

    // Revoke the token that was used to authenticate the current request...
    $request->user()->currentAccessToken()->delete();

    // Revoke a specific token...
    $user->tokens()->where('id', $tokenId)->delete();

<a name="token-expiration"></a>
### 權杖過期

預設情況下，Sanctum 權杖永不失效，並且只能透過 [撤銷權杖](#revoking-tokens) 來使其失效。然而，如果您想為應用程式的 API 權杖配置過期時間，您可以透過應用程式 `sanctum` 配置檔中定義的 `expiration` 配置選項來實現。此配置選項定義了發行的權杖在多少分鐘後會被視為過期：

```php
'expiration' => 525600,
```

如果您想獨立指定每個權杖的過期時間，您可以將過期時間作為 `createToken` 方法的第三個參數提供：

```php
return $user->createToken(
    'token-name', ['*'], now()->addWeek()
)->plainTextToken;
```

如果您已為應用程式配置了權杖過期時間，您可能還希望 [排程任務](/docs/{{version}}/scheduling) 來清除應用程式中已過期的權杖。幸運的是，Sanctum 包含了一個 `sanctum:prune-expired` Artisan 命令，您可以使用它來完成此任務。例如，您可以配置一個排程任務來刪除所有已過期至少 24 小時的過期權杖資料庫記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

<a name="spa-authentication"></a>
## SPA 驗證

Sanctum 的存在也為了提供一個簡單的方法，用於驗證需要與由 Laravel 驅動的 API 進行通訊的單頁應用程式 (SPA)。這些 SPA 可能與您的 Laravel 應用程式存在於同一個儲存庫中，或者是一個完全獨立的儲存庫。

對於此功能，Sanctum 不使用任何形式的權杖。相反地，Sanctum 使用 Laravel 內建基於 Cookie 的 Session 驗證服務。這種驗證方式提供了 CSRF 保護、Session 驗證的優點，並能防止驗證憑證透過 XSS 洩漏。

> [!WARNING]  
> 為了進行驗證，您的 SPA 和 API 必須共享相同的頂級網域。然而，它們可以位於不同的子網域上。此外，您應該確保在請求中發送 `Accept: application/json` 標頭以及 `Referer` 或 `Origin` 標頭。

<a name="spa-configuration"></a>
### 配置

<a name="configuring-your-first-party-domains"></a>
#### 配置您的第一方網域

首先，您應該配置您的 SPA 將從哪些網域發出請求。您可以使用 `sanctum` 配置檔中的 `stateful` 配置選項來配置這些網域。此配置設定決定了哪些網域在向您的 API 發出請求時，將使用 Laravel Session Cookie 維護「有狀態的」驗證。

> [!WARNING]  
> 如果您是透過包含連接埠 (`127.0.0.1:8000`) 的 URL 存取您的應用程式，您應該確保在網域中包含連接埠號碼。

<a name="sanctum-middleware"></a>
#### Sanctum 中介層

接下來，您應該指示 Laravel，來自您的 SPA 的傳入請求可以使用 Laravel Session Cookie 進行驗證，同時仍允許來自第三方或行動應用程式的請求使用 API 權杖進行驗證。這可以透過在應用程式的 `bootstrap/app.php` 檔案中呼叫 `statefulApi` 中介層方法輕鬆實現：

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->statefulApi();
    })

<a name="cors-and-cookies"></a>
#### CORS 與 Cookie

如果您在從運行於獨立子網域的 SPA 驗證您的應用程式時遇到問題，您很可能配置錯誤了 CORS (跨來源資源共享) 或 Session Cookie 設定。

`config/cors.php` 配置檔預設不會發布。如果您需要自訂 Laravel 的 CORS 選項，您應該使用 `config:publish` Artisan 指令發布完整的 `cors` 配置檔：

```bash
php artisan config:publish cors
```

接下來，您應該確保應用程式的 CORS 配置回傳 `Access-Control-Allow-Credentials` 標頭，其值為 `True`。這可以透過將應用程式 `config/cors.php` 配置檔中的 `supports_credentials` 選項設定為 `true` 來實現。

此外，您應該在應用程式的全域 `axios` 實例上啟用 `withCredentials` 和 `withXSRFToken` 選項。通常，這應該在您的 `resources/js/bootstrap.js` 檔案中執行。如果您不是使用 Axios 從前端發出 HTTP 請求，您應該在您自己的 HTTP 用戶端上執行等效的配置：

```js
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

最後，您應該確保應用程式的 Session Cookie 網域配置支援您的根網域的任何子網域。這可以透過在應用程式的 `config/session.php` 配置檔中，將網域加上前導 `.` 來實現：

    'domain' => '.domain.com',

<a name="spa-authenticating"></a>
### 驗證

<a name="csrf-protection"></a>
#### CSRF 保護

為了驗證您的 SPA，您的 SPA「登入」頁面應首先向 `/sanctum/csrf-cookie` 端點發出請求，以初始化應用程式的 CSRF 保護：

```js
axios.get('/sanctum/csrf-cookie').then(response => {
    // Login...
});
```

在此請求期間，Laravel 將設定一個包含目前 CSRF 權杖的 `XSRF-TOKEN` Cookie。然後，此權杖應進行 URL 解碼，並在後續請求中透過 `X-XSRF-TOKEN` 標頭傳遞，某些 HTTP 用戶端函式庫 (例如 Axios 和 Angular HttpClient) 會自動為您執行此操作。如果您的 JavaScript HTTP 函式庫沒有為您設定該值，您將需要手動設定 `X-XSRF-TOKEN` 標頭，使其符合由此路由設定的 `XSRF-TOKEN` Cookie 的 URL 解碼值。

<a name="logging-in"></a>
#### 登入

一旦 CSRF 保護已初始化，您應該向您的 Laravel 應用程式的 `/login` 路由發出 `POST` 請求。這個 `/login` 路由可以[手動實作](/docs/{{version}}/authentication#authenticating-users)，或者使用像 [Laravel Fortify](/docs/{{version}}/fortify) 這樣的無頭驗證套件。

如果登入請求成功，您將被驗證，並且對應用程式路由的後續請求將自動透過 Laravel 應用程式發給您的用戶端的 Session Cookie 進行驗證。此外，由於您的應用程式已向 `/sanctum/csrf-cookie` 路由發出請求，只要您的 JavaScript HTTP 用戶端將 `XSRF-TOKEN` Cookie 的值發送到 `X-XSRF-TOKEN` 標頭中，後續請求就應該自動獲得 CSRF 保護。

當然，如果您的用戶 Session 因不活動而過期，對 Laravel 應用程式的後續請求可能會收到 401 或 419 HTTP 錯誤回應。在這種情況下，您應該將用戶重新導向到您的 SPA 登入頁面。

> [!WARNING]  
> 您可以自由編寫自己的 `/login` 端點；但是，您應該確保它使用 Laravel 提供的標準[基於 Session 的驗證服務](/docs/{{version}}/authentication#authenticating-users)來驗證用戶。通常，這意味著使用 `web` 驗證守衛。

<a name="protecting-spa-routes"></a>
### 保護路由

為了保護路由，使所有傳入請求都必須經過驗證，您應該將 `sanctum` 驗證守衛附加到您 `routes/api.php` 檔案中的 API 路由。這個守衛將確保傳入請求，無論是來自您的 SPA 的有狀態驗證請求，還是來自第三方並包含有效 API 權杖標頭的請求，都能被正確驗證：

    use Illuminate\Http\Request;

    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('auth:sanctum');

<a name="authorizing-private-broadcast-channels"></a>
### 授權私有廣播頻道

如果您的 SPA 需要驗證 [私有 / 存在廣播頻道](/docs/{{version}}/broadcasting#authorizing-channels)，您應該從應用程式的 `bootstrap/app.php` 檔案中的 `withRouting` 方法移除 `channels` 條目。相反地，您應該呼叫 `withBroadcasting` 方法，以便為應用程式的廣播路由指定正確的中介層：

    return Application::configure(basePath: dirname(__DIR__))
        ->withRouting(
            web: __DIR__.'/../routes/web.php',
            // ...
        )
        ->withBroadcasting(
            __DIR__.'/../routes/channels.php',
            ['prefix' => 'api', 'middleware' => ['api', 'auth:sanctum']],
        )

接下來，為了使 Pusher 的授權請求成功，您在初始化 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時需要提供一個自訂的 Pusher `authorizer`。這讓您的應用程式可以配置 Pusher 使用已 [針對跨網域請求正確配置](#cors-and-cookies) 的 `axios` 實例：

```js
window.Echo = new Echo({
    broadcaster: "pusher",
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    encrypted: true,
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    authorizer: (channel, options) => {
        return {
            authorize: (socketId, callback) => {
                axios.post('/api/broadcasting/auth', {
                    socket_id: socketId,
                    channel_name: channel.name
                })
                .then(response => {
                    callback(false, response.data);
                })
                .catch(error => {
                    callback(true, error);
                });
            }
        };
    },
})
```

<a name="mobile-application-authentication"></a>
## 行動應用程式驗證

您也可以使用 Sanctum 權杖來驗證您的行動應用程式對 API 的請求。行動應用程式請求的驗證過程與第三方 API 請求的驗證類似；但是，在您發行 API 權杖的方式上有一些微小的差異。


<a name="issuing-mobile-api-tokens"></a>
### 發行 API 權杖

首先，請建立一個路由，該路由接受使用者的電子郵件/使用者名稱、密碼和裝置名稱，然後將這些憑證交換為新的 Sanctum 權杖。提供給此端點的「裝置名稱」僅供參考，可以是您希望的任何值。通常，裝置名稱應為使用者可識別的名稱，例如「Nuno 的 iPhone 12」。

通常，您會從行動應用程式的「登入」畫面向權杖端點發出請求。該端點將傳回純文字的 API 權杖，然後您可以將其儲存在行動裝置上，並用於發出額外的 API 請求：

    use App\Models\User;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Validation\ValidationException;

    Route::post('/sanctum/token', function (Request $request) {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
            'device_name' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        return $user->createToken($request->device_name)->plainTextToken;
    });

當行動應用程式使用權杖向您的應用程式發出 API 請求時，應將權杖作為 `Bearer` 權杖包含在 `Authorization` 標頭中。

> [!NOTE]  
> 在為行動應用程式發行權杖時，您也可以自由指定 [權杖能力](#token-abilities)。


<a name="protecting-mobile-api-routes"></a>
### 保護路由

如前所述，您可以透過將 `sanctum` 驗證守衛附加到路由上，來保護路由，使所有傳入的請求都必須經過驗證：

    Route::get('/user', function (Request $request) {
        return $request->user();
    })->middleware('auth:sanctum');


<a name="revoking-mobile-api-tokens"></a>
### 撤銷權杖

若要允許使用者撤銷發行給行動裝置的 API 權杖，您可以在網路應用程式 UI 的「帳戶設定」部分中，列出這些權杖的名稱，並附帶一個「撤銷」按鈕。當使用者點擊「撤銷」按鈕時，您可以從資料庫中刪除該權杖。請記住，您可以透過 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯來存取使用者的 API 權杖：

    // Revoke all tokens...
    $user->tokens()->delete();

    // Revoke a specific token...
    $user->tokens()->where('id', $tokenId)->delete();


<a name="testing"></a>
## 測試

在測試期間，可以使用 `Sanctum::actingAs` 方法來驗證使用者並指定應授予其權杖哪些能力：

```php tab=Pest
use App\Models\User;
use Laravel\Sanctum\Sanctum;

test('task list can be retrieved', function () {
    Sanctum::actingAs(
        User::factory()->create(),
        ['view-tasks']
    );

    $response = $this->get('/api/task');

    $response->assertOk();
});
```

```php tab=PHPUnit
use App\Models\User;
use Laravel\Sanctum\Sanctum;

public function test_task_list_can_be_retrieved(): void
{
    Sanctum::actingAs(
        User::factory()->create(),
        ['view-tasks']
    );

    $response = $this->get('/api/task');

    $response->assertOk();
}
```

如果您想授予權杖所有能力，應在提供給 `actingAs` 方法的能力列表中包含 `*`：

    Sanctum::actingAs(
        User::factory()->create(),
        ['*']
    );