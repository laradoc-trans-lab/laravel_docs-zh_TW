# Laravel Sanctum

- [簡介](#introduction)
    - [運作原理](#how-it-works)
- [安裝](#installation)
- [設定](#configuration)
    - [覆寫預設 Model](#overriding-default-models)
- [API Token 認證](#api-token-authentication)
    - [核發 API Token](#issuing-api-tokens)
    - [Token 能力](#token-abilities)
    - [保護路由](#protecting-routes)
    - [撤銷 Token](#revoking-tokens)
    - [Token 過期時間](#token-expiration)
- [SPA 認證](#spa-authentication)
    - [設定](#spa-configuration)
    - [進行認證](#spa-authenticating)
    - [保護路由](#protecting-spa-routes)
    - [授權私有廣播頻道](#authorizing-private-broadcast-channels)
- [行動應用程式認證](#mobile-application-authentication)
    - [核發 API Token](#issuing-mobile-api-tokens)
    - [保護路由](#protecting-mobile-api-routes)
    - [撤銷 Token](#revoking-mobile-api-tokens)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Sanctum](https://github.com/laravel/sanctum) 為 SPA (單頁應用程式)、行動應用程式以及簡單的、基於 Token 的 API 提供了一個羽量級的認證系統。Sanctum 允許您的應用程式中的每個使用者為其帳號生成多個 API Token。這些 Token 可以被授予「能力 (Abilities) / 範圍 (Scopes)」，用以指定該 Token 被允許執行哪些動作。


<a name="how-it-works"></a>
### 運作原理

Laravel Sanctum 的存在是為了解決兩個獨立的問題。在深入探討此套件之前，讓我們分別討論這兩個問題。


<a name="how-it-works-api-tokens"></a>
#### API Token

第一，Sanctum 是一個簡單的套件，您可以用它來向使用者核發 API Token，而不需要處理複雜的 OAuth。此功能靈感來自 GitHub 及其他會核發「個人存取 Token (Personal Access Tokens)」的應用程式。例如，想像您的應用程式在「帳號設定」中有一個畫面，使用者可以在那裡為自己的帳號生成一個 API Token。您可以使用 Sanctum 來生成與管理這些 Token。這些 Token 通常具有非常長的過期時間（數年），但使用者可以隨時手動撤銷它們。

Laravel Sanctum 透過將使用者的 API Token 儲存在單一資料庫資料表中，並透過應包含有效 API Token 的 `Authorization` 標頭來認證傳入的 HTTP 請求，從而提供此功能。


<a name="how-it-works-spa-authentication"></a>
#### SPA 認證

第二，Sanctum 的存在是為了提供一種簡單的方法來認證需要與 Laravel 驅動的 API 進行通訊的單頁應用程式 (SPA)。這些 SPA 可能與您的 Laravel 應用程式存在於同一個存放庫 (Repository) 中，也可能是一個完全獨立的存放庫，例如使用 Next.js 或 Nuxt 建立的 SPA。

對於此功能，Sanctum 不會使用任何種類的 Token。相反地，Sanctum 使用 Laravel 內建基於 Cookie 的 Session 認證服務。通常，Sanctum 會利用 Laravel 的 `web` 認證 Guard 來實現這一點。這提供了 CSRF 保護、Session 認證的好處，同時還能防止認證憑證透過 XSS 洩露。

只有當傳入的請求來自您自己的 SPA 前端時，Sanctum 才會嘗試使用 Cookie 進行認證。當 Sanctum 檢查傳入的 HTTP 請求時，它會先檢查是否存在認證 Cookie，如果不存在，Sanctum 接著會檢查 `Authorization` 標頭中是否有有效的 API Token。

> [!NOTE]
> 僅將 Sanctum 用於 API Token 認證或僅用於 SPA 認證是完全沒有問題的。僅僅因為您使用了 Sanctum，並不代表您必須同時使用它所提供的這兩種功能。


<a name="installation"></a>
## 安裝

您可以使用 `install:api` Artisan 指令來安裝 Laravel Sanctum：

```shell
php artisan install:api
```

接下來，如果您計畫利用 Sanctum 來認證 SPA，請參閱本文件的 [SPA 認證](#spa-authentication)章節。


<a name="configuration"></a>
## 設定


<a name="overriding-default-models"></a>
### 覆寫預設 Model

雖然通常不需要，但您可以自由擴充 Sanctum 內部使用的 `PersonalAccessToken` Model：

```php
use Laravel\Sanctum\PersonalAccessToken as SanctumPersonalAccessToken;

class PersonalAccessToken extends SanctumPersonalAccessToken
{
    // ...
}
```

然後，您可以透過 Sanctum 提供的 `usePersonalAccessTokenModel` 方法來指示 Sanctum 使用您的自訂 Model。通常，您應該在應用程式的 `AppServiceProvider` 檔案中的 `boot` 方法內呼叫此方法：

```php
use App\Models\Sanctum\PersonalAccessToken;
use Laravel\Sanctum\Sanctum;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Sanctum::usePersonalAccessTokenModel(PersonalAccessToken::class);
}
```

<a name="api-token-authentication"></a>
## API Token 認證

> [!NOTE]
> 您不應該使用 API token 來認證您自己的第一方 SPA。相反地，請使用 Sanctum 內建的 [SPA 認證功能](#spa-authentication)。


<a name="issuing-api-tokens"></a>
### 核發 API Token

Sanctum 允許您核發 API token / 個人存取權杖 (personal access tokens)，可用於認證至您應用程式的 API 請求。當使用 API token 發送請求時，該 token 應作為 `Bearer` token 包含在 `Authorization` 標頭中。

若要開始為使用者核發 token，您的 User Model 應使用 `Laravel\Sanctum\HasApiTokens` trait：

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

若要核發 token，您可以使用 `createToken` 方法。`createToken` 方法會返回一個 `Laravel\Sanctum\NewAccessToken` 實例。API token 在存入資料庫前會使用 SHA-256 進行雜湊，但您可以使用 `NewAccessToken` 實例的 `plainTextToken` 屬性來取得該 token 的純文字值。您應該在 token 建立後立即將此值顯示給使用者：

```php
use Illuminate\Http\Request;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);

    return ['token' => $token->plainTextToken];
});
```

您可以使用 `HasApiTokens` trait 提供的 `tokens` Eloquent 關聯來存取該使用者的所有 token：

```php
foreach ($user->tokens as $token) {
    // ...
}
```


<a name="token-abilities"></a>
### Token 能力

Sanctum 允許您為 token 指派「能力 (abilities)」。能力的作用與 OAuth 的「範圍 (scopes)」類似。您可以將字串能力的陣列作為第二個引數傳遞給 `createToken` 方法：

```php
return $user->createToken('token-name', ['server:update'])->plainTextToken;
```

在處理經由 Sanctum 認證的傳入請求時，您可以使用 `tokenCan` 或 `tokenCant` 方法來判定該 token 是否具有特定的能力：

```php
if ($user->tokenCan('server:update')) {
    // ...
}

if ($user->tokenCant('server:update')) {
    // ...
}
```


<a name="token-ability-middleware"></a>
#### Token 能力中介層

Sanctum 還包含了兩個中介層，可用於驗證傳入的請求是否已透過被授予特定能力的 token 進行認證。若要開始使用，請在應用程式的 `bootstrap/app.php` 檔案中定義以下中介層別名：

```php
use Laravel\Sanctum\Http\Middleware\CheckAbilities;
use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'abilities' => CheckAbilities::class,
        'ability' => CheckForAnyAbility::class,
    ]);
})
```

可以將 `abilities` 中介層指派給路由，以驗證傳入請求的 token 是否具備所有列出的能力：

```php
Route::get('/orders', function () {
    // Token has both "check-status" and "place-orders" abilities...
})->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);
```

可以將 `ability` 中介層指派給路由，以驗證傳入請求的 token 是否具備列出能力中的 *至少一個*：

```php
Route::get('/orders', function () {
    // Token has the "check-status" or "place-orders" ability...
})->middleware(['auth:sanctum', 'ability:check-status,place-orders']);
```


<a name="first-party-ui-initiated-requests"></a>
#### 第一方 UI 發起的請求

為求便利，如果傳入的已認證請求來自您的第一方 SPA，且您使用的是 Sanctum 內建的 [SPA 認證](#spa-authentication)，則 `tokenCan` 方法將一律返回 `true`。

然而，這並不一定代表您的應用程式必須允許使用者執行該操作。通常，您應用程式的[授權原則 (authorization policies)](/docs/{{version}}/authorization#creating-policies) 將會判定 token 是否已被授予執行這些能力的權限，並檢查使用者實例本身是否應被允許執行該操作。

例如，假設有一個管理伺服器的應用程式，這可能意味著要檢查 token 是否被授權更新伺服器，**並且**該伺服器屬於該使用者：

```php
return $request->user()->id === $server->user_id &&
       $request->user()->tokenCan('server:update')
```

起初，允許對第一方 UI 發起的請求呼叫 `tokenCan` 方法並始終返回 `true` 看起來可能極為奇怪；然而，能夠始終假設 API token 存在並可透過 `tokenCan` 方法進行檢視是相當便利的。透過這種做法，您可以在應用程式的授權原則中隨時呼叫 `tokenCan` 方法，而無需擔心該請求是從應用程式的 UI 觸發，還是由您 API 的第三方消費者發起。


<a name="protecting-routes"></a>
### 保護路由

為了保護路由以確保所有傳入的請求都必須經過認證，您應該在 `routes/web.php` 和 `routes/api.php` 路由檔案中，將 `sanctum` 認證 guard 附加到您要保護的路由上。此 guard 將確保傳入的請求不論是作為有狀態、使用 Cookie 認證的請求，抑或是來自第三方的請求（需包含有效的 API token 標頭），都能通過認證。

您可能會好奇，為什麼我們建議您在應用程式的 `routes/web.php` 檔案中使用 `sanctum` guard 來認證路由。請記住，Sanctum 首先會嘗試使用 Laravel 典型的 Session 認證 Cookie 來認證傳入的請求。如果該 Cookie 不存在，Sanctum 就會嘗試使用請求中 `Authorization` 標頭的 token 來認證請求。此外，使用 Sanctum 認證所有請求可確保我們始終可以在當前已認證的使用者實例上呼叫 `tokenCan` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-tokens"></a>
### 撤銷 Token

您可以透過使用 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯，從資料庫中刪除 token 來「撤銷」它們：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke the token that was used to authenticate the current request...
$request->user()->currentAccessToken()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```


<a name="token-expiration"></a>
### Token 過期時間

預設情況下，Sanctum token 永不過期，且只能透過[撤銷 Token](#revoking-tokens) 來使其失效。然而，如果您想為應用程式的 API token 設定過期時間，您可以透過應用程式的 `sanctum` 設定檔中定義的 `expiration` 設定選項來進行。此設定選項定義了已核發的 token 被視為過期前的分鐘數：

```php
'expiration' => 525600,
```

如果您想單獨指定每個 token 的過期時間，可以將過期時間作為第三個引數傳遞給 `createToken` 方法：

```php
return $user->createToken(
    'token-name', ['*'], now()->plus(weeks: 1)
)->plainTextToken;
```

如果您為應用程式設定了 token 過期時間，您可能也希望[排程任務](/docs/{{version}}/scheduling)來修剪應用程式中已過期的 token。幸運的是，Sanctum 包含了一個 `sanctum:prune-expired` Artisan 指令，您可以使用它來完成此操作。例如，您可以設定一個排程任務，刪除所有已過期至少 24 小時的過期 token 資料庫紀錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

<a name="spa-authentication"></a>
## SPA 認證

Sanctum 也提供了一種簡單的方法來認證需要與 Laravel API 進行通訊的單頁應用程式 (SPA)。這些 SPA 可以與您的 Laravel 應用程式存在於同一個專案存放庫 (Repository) 中，也可以是完全獨立的存放庫。

對於此功能，Sanctum 不會使用任何類型的 Token。相反地，Sanctum 使用 Laravel 內建基於 Cookie 的 Session 認證服務。這種認證方式帶來了 CSRF 保護、Session 認證的好處，同時也能防止認證憑證透過 XSS 外洩。

> [!WARNING]
> 為了進行認證，您的 SPA 和 API 必須共享相同的頂級網域 (Top-level domain)。不過，它們可以部署在不同的子網域上。此外，您必須確保在請求中傳送 `Accept: application/json` 標頭以及 `Referer` 或 `Origin` 標頭。


<a name="spa-configuration"></a>
### 設定


<a name="configuring-your-first-party-domains"></a>
#### 設定您的第一方網域

首先，您應該設定您的 SPA 將從哪些網域發送請求。您可以使用 `sanctum` 設定檔中的 `stateful` 設定選項來設定這些網域。此設定決定了在向您的 API 發送請求時，哪些網域將使用 Laravel 的 Session Cookie 來維持「具狀態 (Stateful)」的認證。

為了解決您在設定第一方具狀態網域時的困擾，Sanctum 提供了兩個您可以在設定中使用的輔助函式。首先，`Sanctum::currentApplicationUrlWithPort()` 會從 `APP_URL` 環境變數中傳回目前的應用程式 URL，而 `Sanctum::currentRequestHost()` 則會在具狀態網域清單中注入一個預留位置，該預留位置在執行時期會被目前請求的主機 (Host) 替換，以便將來自同一網域的所有請求都視為具狀態。

> [!WARNING]
> 如果您是透過包含連接埠的 URL (`127.0.0.1:8000`) 存取應用程式，請確保您在網域中也包含了連接埠號碼。


<a name="sanctum-middleware"></a>
#### Sanctum 中介層

接下來，您應該指示 Laravel，允許來自您 SPA 的傳入請求使用 Laravel 的 Session Cookie 進行認證，同時仍允許來自第三方或行動應用程式的請求使用 API Token 進行認證。這可以透過在應用程式的 `bootstrap/app.php` 檔案中調用 `statefulApi` 中介層方法來輕鬆完成：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```


<a name="cors-and-cookies"></a>
#### CORS 與 Cookie

如果您在從執行於獨立子網域的 SPA 對應用程式進行認證時遇到問題，您可能設定錯了 CORS (跨來源資源共享) 或 Session Cookie 設定。

`config/cors.php` 設定檔預設並未發布。如果您需要自訂 Laravel 的 CORS 選項，您應該使用 `config:publish` Artisan 指令發布完整的 `cors` 設定檔：

```shell
php artisan config:publish cors
```

Next, you should ensure that your application's CORS configuration is returning the `Access-Control-Allow-Credentials` header with a value of `True`. This may be accomplished by setting the `supports_credentials` option within your application's `config/cors.php` configuration file to `true`.

In addition, you should enable the `withCredentials` and `withXSRFToken` options on your application's global `axios` instance. This can be performed in your `resources/js/app.js` file. If you are not using Axios to make HTTP requests from your frontend, you should perform the equivalent configuration on your own HTTP client:

```js
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

最後，您應該確保應用程式的 Session Cookie 網域設定支援根網域的任何子網域。您可以在應用程式的 `config/session.php` 設定檔中，透過在網域前加上一個開頭的 `.` 來實現此目的：

```php
'domain' => '.domain.com',
```


<a name="spa-authenticating"></a>
### 進行認證


<a name="csrf-protection"></a>
#### CSRF 保護

要認證您的 SPA，您的 SPA「登入」頁面應該先向 `/sanctum/csrf-cookie` 端點發送請求，以初始化應用程式的 CSRF 保護：

```js
axios.get('/sanctum/csrf-cookie').then(response => {
    // Login...
});
```

在此請求期間，Laravel 將會設定一個包含目前 CSRF 權杖 (Token) 的 `XSRF-TOKEN` Cookie。接著，此 Token 應經由 URL 解碼，並在後續的請求中放入 `X-XSRF-TOKEN` 標頭中傳送，某些 HTTP 用戶端函式庫（例如 Axios 和 Angular HttpClient）會自動為您處理此操作。如果您的 JavaScript HTTP 函式庫沒有自動為您設定該值，您將需要手動設定 `X-XSRF-TOKEN` 標頭，使其與該路由所設定之 `XSRF-TOKEN` Cookie 解碼後的值相符。


<a name="logging-in"></a>
#### 登入

一旦 CSRF 保護初始化完成，您應該向 Laravel 應用程式的 `/login` 路由發送一個 `POST` 請求。此 `/login` 路由可以[手動實現](/docs/{{version}}/authentication#authenticating-users)，或是使用像 [Laravel Fortify](/docs/{{version}}/fortify) 這樣的無介面 (Headless) 認證套件。

如果登入請求成功，您將通過認證，並且後續對應用程式路由的請求將自動透過 Laravel 應用程式發行給您用戶端的 Session Cookie 進行認證。此外，由於您的應用程式已經向 `/sanctum/csrf-cookie` 路由發送了請求，只要您的 JavaScript HTTP 用戶端在 `X-XSRF-TOKEN` 標頭中傳送了 `XSRF-TOKEN` Cookie 的值，後續的請求就應該會自動獲得 CSRF 保護。

當然，如果您的使用者 Session 因閒置而過期，後續對 Laravel 應用程式的請求可能會收到 401 或 419 的 HTTP 錯誤回應。在這種狀況下，您應該將使用者導向至您的 SPA 登入頁面。

由於這種 SPA 認證方式是基於 Session 的，因此您可以使用 Laravel 的標準認證服務，包括 [「記住我」](/docs/{{version}}/authentication#remembering-users) 功能。

> [!WARNING]
> 您可以自由編寫自己的 `/login` 端點；但是，您應該確保它使用 Laravel 提供的標準[基於 Session 的認證服務](/docs/{{version}}/authentication#authenticating-users)來認證使用者。這通常意味著要使用 `web` 認證 Guard。


<a name="protecting-spa-routes"></a>
### 保護路由

若要保護路由以確保所有傳入的請求都必須通過認證，您應該在 `routes/api.php` 檔案中將 `sanctum` 認證 Guard 套用到您的 API 路由。此 Guard 將確保傳入的請求不是來自您 SPA 的具狀態認證請求，就是包含有效 API Token 標頭的第三方請求：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

<a name="authorizing-private-broadcast-channels"></a>
### 授權私有廣播頻道

如果您的 SPA 需要認證[私有 / 存在廣播頻道](/docs/{{version}}/broadcasting#authorizing-channels)，您應該從應用程式的 `bootstrap/app.php` 檔案中的 `withRouting` 方法裡移除 `channels` 項目。相反地，您應該呼叫 `withBroadcasting` 方法，以便為您的應用程式廣播路由指定正確的中介層：

```php
return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        // ...
    )
    ->withBroadcasting(
        __DIR__.'/../routes/channels.php',
        ['prefix' => 'api', 'middleware' => ['api', 'auth:sanctum']],
    )
```

接下來，為了讓 Pusher 的授權請求能夠成功，您在初始化 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時需要提供一個自訂的 Pusher `authorizer`。這能讓您的應用程式設定 Pusher 使用[已針對跨網域請求進行正確設定](#cors-and-cookies)的 `axios` 實例：

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
## 行動應用程式認證

您也可以使用 Sanctum token 來認證行動應用程式發送給 API 的請求。認證行動應用程式請求的流程與認證第三方 API 請求類似，不過在如何核發 API token 上有一些小差異。


<a name="issuing-mobile-api-tokens"></a>
### 核發 API Token

首先，請建立一個接收使用者 Email / 帳號、密碼及裝置名稱的路由，並用這些憑證來交換一個新的 Sanctum token。給予該端點的「裝置名稱」僅供識別使用，可以是任何您想要的名稱。一般來說，裝置名稱應該要是使用者認得出來的名稱，例如「Nuno's iPhone 17」。

通常，您會從行動應用程式的「登入」畫面發送請求到 token 端點。該端點會回傳純文字的 API token，接著便可以將其儲存在行動裝置上，並用於發送後續的 API 請求：

```php
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
```

當行動應用程式使用 token 向您的應用程式發送 API 請求時，應將 token 放在 `Authorization` 標頭中作為 `Bearer` token 傳送。

> [!NOTE]
> 為行動應用程式核發 token 時，您也可以自由地指定 [token 能力](#token-abilities)。


<a name="protecting-mobile-api-routes"></a>
### 保護路由

如同先前所述，您可以藉由為路由套用 `sanctum` 認證防衛 (guard) 來保護路由，確保所有傳入的請求都必須經過認證：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-mobile-api-tokens"></a>
### 撤銷 Token

為了讓使用者能撤銷核發給行動裝置的 API token，您可以在網頁應用程式 UI 的「帳號設定」區塊中，列出這些 token 的名稱並加上「撤銷」按鈕。當使用者點選「撤銷」按鈕時，您就可以從資料庫中刪除該 token。請記住，您可以透過 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯來取得使用者的 API token：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```


<a name="testing"></a>
## 測試

在測試時，可以使用 `Sanctum::actingAs` 方法來認證使用者，並指定該使用者的 token 應被授予哪些能力：

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

如果您想要將所有能力都授予該 token，應在傳入 `actingAs` 方法的能力清單中包含 `*`：

```php
Sanctum::actingAs(
    User::factory()->create(),
    ['*']
);
```