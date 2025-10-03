# Laravel Sanctum

- [簡介](#introduction)
    - [運作方式](#how-it-works)
- [安裝](#installation)
- [設定](#configuration)
    - [覆寫預設的模型](#overriding-default-models)
- [API Token 驗證](#api-token-authentication)
    - [發行 API Token](#issuing-api-tokens)
    - [Token 能力](#token-abilities)
    - [保護路由](#protecting-routes)
    - [撤銷 Token](#revoking-tokens)
    - [Token 過期時間](#token-expiration)
- [SPA 驗證](#spa-authentication)
    - [設定](#spa-configuration)
    - [驗證](#spa-authenticating)
    - [保護路由](#protecting-spa-routes)
    - [授權私人廣播頻道](#authorizing-private-broadcast-channels)
- [行動應用程式驗證](#mobile-application-authentication)
    - [發行 API Token](#issuing-mobile-api-tokens)
    - [保護路由](#protecting-mobile-api-routes)
    - [撤銷 Token](#revoking-mobile-api-tokens)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Sanctum](https://github.com/laravel/sanctum) 提供輕量級的驗證系統，用於 SPA (單頁應用程式)、行動應用程式，以及簡單的基於 Token 的 API。Sanctum 允許應用程式中的每個使用者為其帳戶產生多個 API Token。這些 Token 可被授予「能力 (abilities)」/「範圍 (scopes)」，以指定這些 Token 允許執行的動作。

<a name="how-it-works"></a>
### 運作方式

Laravel Sanctum 旨在解決兩個獨立的問題。在深入探討此函式庫之前，讓我們先討論每個問題。

<a name="how-it-works-api-tokens"></a>
#### API Token

首先，Sanctum 是一個簡單的套件，可用於向使用者發行 API Token，而無需 OAuth 的複雜性。此功能靈感來自於 GitHub 及其他發行「個人存取 Token」的應用程式。例如，想像您的應用程式「帳戶設定」中，有一個畫面允許使用者為其帳戶產生一個 API Token。您可以使用 Sanctum 來產生和管理這些 Token。這些 Token 通常具有非常長的過期時間（數年），但使用者可以隨時手動撤銷。

Laravel Sanctum 提供此功能，透過將使用者 API Token 儲存在單一資料庫表格中，並透過 `Authorization` 標頭驗證傳入的 HTTP 請求，該標頭應包含有效的 API Token。

<a name="how-it-works-spa-authentication"></a>
#### SPA 驗證

其次，Sanctum 提供了一種簡單的方法來驗證需要與 Laravel 支援的 API 通訊的單頁應用程式 (SPA)。這些 SPA 可能存在於與您的 Laravel 應用程式相同的儲存庫中，也可能是一個完全獨立的儲存庫，例如使用 Next.js 或 Nuxt 建立的 SPA。

對於此功能，Sanctum 不使用任何種類的 Token。相反地，Sanctum 使用 Laravel 內建基於 Cookie 的 Session 驗證服務。通常，Sanctum 利用 Laravel 的 `web` 驗證 guard 來達成此目的。這提供了 CSRF 保護、Session 驗證的好處，同時也防止透過 XSS 洩漏驗證憑證。

Sanctum 只會在傳入請求源自於您自己的 SPA 前端時，才會嘗試使用 Cookie 進行驗證。當 Sanctum 檢查傳入的 HTTP 請求時，它會首先檢查是否存在驗證 Cookie，如果不存在，Sanctum 便會檢查 `Authorization` 標頭以尋找有效的 API Token。

> [!NOTE]
> 僅將 Sanctum 用於 API Token 驗證或僅用於 SPA 驗證是完全可以的。即使您使用了 Sanctum，也無需同時使用它提供的兩種功能。

<a name="installation"></a>
## 安裝

您可以使用 `install:api` Artisan 命令來安裝 Laravel Sanctum：

```shell
php artisan install:api
```

接著，如果您計畫利用 Sanctum 來驗證 SPA，請參閱本文件中的[SPA 驗證](#spa-authentication)章節。

<a name="configuration"></a>
## 設定

<a name="overriding-default-models"></a>
### 覆寫預設的模型

儘管通常不需要，但您可以自由地擴展 Sanctum 內部使用的 `PersonalAccessToken` 模型：

```php
use Laravel\Sanctum\PersonalAccessToken as SanctumPersonalAccessToken;

class PersonalAccessToken extends SanctumPersonalAccessToken
{
    // ...
}
```

接著，您可以透過 Sanctum 提供的 `usePersonalAccessTokenModel` 方法，指示 Sanctum 使用您的自訂模型。通常，您應該在應用程式 `AppServiceProvider` 檔案的 `boot` 方法中呼叫此方法：

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
## API Token 驗證

> [!NOTE]
> 您不應使用 API Token 來驗證您自己的第一方 SPA。請改用 Sanctum 內建的 [SPA 驗證功能](#spa-authentication)。


<a name="issuing-api-tokens"></a>
### 發行 API Token

Sanctum 允許您發行 API Token / 個人存取 Token，可用於驗證對應用程式的 API 請求。當使用 API Token 發出請求時，該 Token 應包含在 `Authorization` 標頭中，作為 `Bearer` Token。

要開始為使用者發行 Token，您的 User 模型應使用 `Laravel\Sanctum\HasApiTokens` trait：

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

要發行 Token，您可以使用 `createToken` 方法。 `createToken` 方法會回傳一個 `Laravel\Sanctum\NewAccessToken` 實例。API Token 在儲存到資料庫之前會使用 SHA-256 雜湊進行處理，但您可以透過 `NewAccessToken` 實例的 `plainTextToken` 屬性存取 Token 的明文值。您應在 Token 建立後立即將此值顯示給使用者：

```php
use Illuminate\Http\Request;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);

    return ['token' => $token->plainTextToken];
});
```

您可以使用 `HasApiTokens` trait 提供的 `tokens` Eloquent 關聯來存取使用者所有的 Token：

```php
foreach ($user->tokens as $token) {
    // ...
}
```


<a name="token-abilities"></a>
### Token 能力

Sanctum 允許您為 Token 指派「能力」。能力的作用類似於 OAuth 的「Scope」。您可以將字串能力陣列作為第二個參數傳遞給 `createToken` 方法：

```php
return $user->createToken('token-name', ['server:update'])->plainTextToken;
```

當處理由 Sanctum 驗證的傳入請求時，您可以使用 `tokenCan` 或 `tokenCant` 方法來判斷 Token 是否具有指定的能力：

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

Sanctum 還包含兩個中介層，可用於驗證傳入請求是否透過已獲授權指定能力的 Token 進行驗證。要開始使用，請在應用程式的 `bootstrap/app.php` 檔案中定義以下中介層別名：

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

`abilities` 中介層可以指派給路由，以驗證傳入請求的 Token 具有所有列出的能力：

```php
Route::get('/orders', function () {
    // Token has both "check-status" and "place-orders" abilities...
})->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);
```

`ability` 中介層可以指派給路由，以驗證傳入請求的 Token 具有 *至少一項* 列出的能力：

```php
Route::get('/orders', function () {
    // Token has the "check-status" or "place-orders" ability...
})->middleware(['auth:sanctum', 'ability:check-status,place-orders']);
```


<a name="first-party-ui-initiated-requests"></a>
#### 第一方 UI 發起的請求

為了方便起見，如果傳入的驗證請求來自您的第一方 SPA 並且您正在使用 Sanctum 內建的 [SPA 驗證](#spa-authentication)，則 `tokenCan` 方法將始終回傳 `true`。

然而，這不一定表示您的應用程式必須允許使用者執行該動作。通常，應用程式的 [授權 Policies](/docs/{{version}}/authorization#creating-policies) 將會決定 Token 是否已被授予執行能力的權限，並檢查使用者實例本身是否應被允許執行該動作。

例如，如果我們想像一個管理伺服器的應用程式，這可能意味著檢查 Token 是否被授權更新伺服器，**並且**該伺服器屬於該使用者：

```php
return $request->user()->id === $server->user_id &&
       $request->user()->tokenCan('server:update')
```

起初，允許呼叫 `tokenCan` 方法並始終針對第一方 UI 發起的請求回傳 `true` 可能看起來很奇怪；然而，能夠始終假設 API Token 可用並可以透過 `tokenCan` 方法進行檢查是很方便的。透過這種方法，您可以在應用程式的授權 Policies 中始終呼叫 `tokenCan` 方法，而無需擔心請求是由應用程式的 UI 觸發，還是由 API 的第三方消費者發起。


<a name="protecting-routes"></a>
### 保護路由

為了保護路由，使所有傳入請求都必須經過驗證，您應該在應用程式的 `routes/web.php` 和 `routes/api.php` 路由檔案中，將 `sanctum` 驗證 guard 附加到受保護的路由。此 guard 將確保傳入請求以有狀態的 Cookie 驗證請求進行驗證，或在請求來自第三方時，包含有效的 API Token 標頭。

您可能想知道為什麼我們建議您使用 `sanctum` guard 來驗證應用程式 `routes/web.php` 檔案中的路由。請記住，Sanctum 將首先嘗試使用 Laravel 典型的 Session 驗證 Cookie 來驗證傳入請求。如果該 Cookie 不存在，Sanctum 將嘗試使用請求 `Authorization` 標頭中的 Token 來驗證請求。此外，使用 Sanctum 驗證所有請求可確保我們始終可以在目前已驗證的使用者實例上呼叫 `tokenCan` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-tokens"></a>
### 撤銷 Token

您可以使用 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯，從資料庫中刪除 Token，從而「撤銷」它們：

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

預設情況下，Sanctum Token 永不過期，只能透過 [撤銷 Token](#revoking-tokens) 來使其失效。然而，如果您想為應用程式的 API Token 設定過期時間，您可以透過應用程式 `sanctum` 設定檔中定義的 `expiration` 設定選項來完成。此設定選項定義了已發行 Token 在多少分鐘後將被視為過期：

```php
'expiration' => 525600,
```

如果您想獨立指定每個 Token 的過期時間，您可以將過期時間作為第三個參數提供給 `createToken` 方法：

```php
return $user->createToken(
    'token-name', ['*'], now()->addWeek()
)->plainTextToken;
```

如果您已為應用程式設定了 Token 過期時間，您可能還希望 [排程任務](/docs/{{version}}/scheduling) 來清理應用程式的過期 Token。幸運的是，Sanctum 包含一個 `sanctum:prune-expired` Artisan 命令，您可以使用它來完成此任務。例如，您可以配置一個排程任務，以每天刪除所有已過期至少 24 小時的 Token 資料庫記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

<a name="spa-authentication"></a>
## SPA 驗證

Sanctum 也提供了一種簡單的方式來驗證需要與 Laravel API 通訊的單頁應用程式 (SPA)。這些 SPA 可能存在於您的 Laravel 應用程式所在的相同儲存庫中，也可能是完全獨立的儲存庫。

對於此功能，Sanctum 不使用任何形式的 Token。相反地，Sanctum 使用 Laravel 內建的基於 Cookie 的 Session 驗證服務。這種驗證方法提供了 CSRF 保護、Session 驗證的好處，同時也防止了透過 XSS 洩露驗證憑證。

> [!WARNING]
> 為了進行驗證，您的 SPA 和 API 必須共享相同的頂級網域。然而，它們可以位於不同的子網域上。此外，您應該確保在您的請求中傳送 `Accept: application/json` 標頭以及 `Referer` 或 `Origin` 標頭。

<a name="spa-configuration"></a>
### 設定

<a name="configuring-your-first-party-domains"></a>
#### 設定您的第一方網域

首先，您應該設定您的 SPA 將從哪些網域發出請求。您可以使用 `sanctum` 設定檔中的 `stateful` 設定選項來設定這些網域。此設定決定了哪些網域在向您的 API 發出請求時，將使用 Laravel Session Cookie 來維護「有狀態的」驗證。

為了協助您設定第一方有狀態網域，Sanctum 提供了兩個輔助函式，您可以將其包含在設定中。首先，`Sanctum::currentApplicationUrlWithPort()` 將從 `APP_URL` 環境變數中回傳當前應用程式的 URL，而 `Sanctum::currentRequestHost()` 將在有狀態網域列表中注入一個佔位符，該佔位符在執行時將被當前請求的主機替換，以便所有具有相同網域的請求都被視為有狀態的。

> [!WARNING]
> 如果您透過包含連接埠的 URL (`127.0.0.1:8000`) 存取您的應用程式，您應該確保在網域中包含連接埠號。

<a name="sanctum-middleware"></a>
#### Sanctum 中介層

接下來，您應該指示 Laravel 允許來自您的 SPA 的傳入請求使用 Laravel 的 Session Cookie 進行驗證，同時仍允許來自第三方或行動應用程式的請求使用 API Token 進行驗證。這可以透過在應用程式的 `bootstrap/app.php` 檔案中呼叫 `statefulApi` 中介層方法來輕鬆完成：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```

<a name="cors-and-cookies"></a>
#### CORS 與 Cookie

如果您在從執行於不同子網域的 SPA 驗證您的應用程式時遇到問題，您可能錯誤配置了您的 CORS (跨來源資源共享) 或 Session Cookie 設定。

預設情況下，`config/cors.php` 設定檔並未發佈。如果您需要自訂 Laravel 的 CORS 選項，您應該使用 `config:publish` Artisan 命令發佈完整的 `cors` 設定檔：

```shell
php artisan config:publish cors
```

接下來，您應該確保您的應用程式的 CORS 設定回傳了值為 `True` 的 `Access-Control-Allow-Credentials` 標頭。這可以透過將應用程式 `config/cors.php` 設定檔中的 `supports_credentials` 選項設定為 `true` 來實現。

此外，您應該在應用程式的全局 `axios` 實例上啟用 `withCredentials` 和 `withXSRFToken` 選項。通常，這應該在您的 `resources/js/bootstrap.js` 檔案中執行。如果您沒有使用 Axios 從前端發出 HTTP 請求，您應該在您自己的 HTTP 客戶端上執行等效的設定：

```js
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

最後，您應該確保您的應用程式的 Session Cookie 網域設定支援您的根網域的任何子網域。這可以透過在您的應用程式 `config/session.php` 設定檔中，在網域前加上一個前導 `.` 來實現：

```php
'domain' => '.domain.com',
```

<a name="spa-authenticating"></a>
### 驗證

<a name="csrf-protection"></a>
#### CSRF 保護

為了驗證您的 SPA，您的 SPA 的「登入」頁面應首先向 `/sanctum/csrf-cookie` 端點發出請求，以初始化應用程式的 CSRF 保護：

```js
axios.get('/sanctum/csrf-cookie').then(response => {
    // Login...
});
```

在此請求期間，Laravel 將設定一個包含當前 CSRF Token 的 `XSRF-TOKEN` Cookie。然後，此 Token 應進行 URL 解碼，並在後續請求中透過 `X-XSRF-TOKEN` 標頭傳遞，某些 HTTP 客戶端函式庫 (例如 Axios 和 Angular HttpClient) 會自動為您完成此操作。如果您的 JavaScript HTTP 函式庫未為您設定該值，您將需要手動設定 `X-XSRF-TOKEN` 標頭，使其與此路由設定的 `XSRF-TOKEN` Cookie 的 URL 解碼值相符。

<a name="logging-in"></a>
#### 登入

一旦 CSRF 保護初始化完成，您應該向 Laravel 應用程式的 `/login` 路由發出 `POST` 請求。這個 `/login` 路由可以 [手動實作](/docs/{{version}}/authentication#authenticating-users) 或使用像 [Laravel Fortify](/docs/{{version}}/fortify) 這樣的無頭 (headless) 驗證套件。

如果登入請求成功，您將被驗證，並且對應用程式路由的後續請求將透過 Laravel 應用程式發給您的客戶端的 Session Cookie 自動進行驗證。此外，由於您的應用程式已向 `/sanctum/csrf-cookie` 路由發出請求，只要您的 JavaScript HTTP 客戶端在 `X-XSRF-TOKEN` 標頭中傳送 `XSRF-TOKEN` Cookie 的值，後續請求就應自動獲得 CSRF 保護。

當然，如果您的使用者 Session 因缺乏活動而過期，對 Laravel 應用程式的後續請求可能會收到 401 或 419 HTTP 錯誤回應。在這種情況下，您應該將使用者重新導向到您 SPA 的登入頁面。

> [!WARNING]
> 您可以自由編寫自己的 `/login` 端點；但是，您應該確保它使用 Laravel 提供的標準 [基於 Session 的驗證服務](/docs/{{version}}/authentication#authenticating-users) 來驗證使用者。通常，這意味著使用 `web` 驗證 guard。

<a name="protecting-spa-routes"></a>
### 保護路由

為了保護路由，使所有傳入請求都必須經過驗證，您應該將 `sanctum` 驗證 guard 附加到您 `routes/api.php` 檔案中的 API 路由。這個 guard 將確保傳入請求要嘛是來自您的 SPA 的有狀態驗證請求，要嘛是包含有效 API Token 標頭的請求 (如果請求來自第三方)：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

<a name="authorizing-private-broadcast-channels"></a>
### 授權私人廣播頻道

如果您的 SPA 需要與 [私人 / 存在廣播頻道](/docs/{{version}}/broadcasting#authorizing-channels) 進行驗證，您應該從應用程式的 `bootstrap/app.php` 檔案中 `withRouting` 方法裡的 `channels` 條目移除。相反地，您應該呼叫 `withBroadcasting` 方法，以便為應用程式的廣播路由指定正確的中介層：

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

接下來，為了讓 Pusher 的授權請求成功，您需要在初始化 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時提供自訂的 Pusher `authorizer`。這允許您的應用程式設定 Pusher 使用 [已為跨網域請求正確配置](#cors-and-cookies) 的 `axios` 實例：

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

您也可以使用 Sanctum Token 來驗證您的行動應用程式對 API 發出的請求。行動應用程式請求的驗證過程與驗證第三方 API 請求相似；然而，在發行 API Token 的方式上存在一些細微差異。


<a name="issuing-mobile-api-tokens"></a>
### 發行 API Token

首先，請建立一個路由，該路由接受使用者的電子郵件/使用者名稱、密碼和裝置名稱，然後將這些憑證換取一個新的 Sanctum Token。提供給此端點的「裝置名稱」僅供資訊用途，您可以設定任何您希望的值。一般來說，裝置名稱應該是一個使用者能辨識的名稱，例如「Nuno 的 iPhone 12」。

通常，您會從行動應用程式的「登入」畫面向 token 端點發出請求。該端點將返回純文字的 API Token，該 Token 隨後可以儲存在行動裝置上，並用於發出額外的 API 請求：

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

當行動應用程式使用 Token 向您的應用程式發出 API 請求時，它應該在 `Authorization` 標頭中以 `Bearer` Token 的形式傳遞該 Token。

> [!NOTE]
> 為行動應用程式發行 Token 時，您也可以自由指定 [token 能力](#token-abilities)。


<a name="protecting-mobile-api-routes"></a>
### 保護路由

如先前文件中所說明，您可以保護路由，讓所有傳入的請求都必須經過驗證，方法是將 `sanctum` 驗證 guard 附加到這些路由：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-mobile-api-tokens"></a>
### 撤銷 Token

為了讓使用者能夠撤銷發行給行動裝置的 API Token，您可以在網站應用程式 UI 的「帳號設定」部分，將這些 Token 依名稱列出，並附上一個「撤銷」按鈕。當使用者點擊「撤銷」按鈕時，您可以從資料庫中刪除該 Token。請記住，您可以透過 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯來存取使用者的 API Token：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```


<a name="testing"></a>
## 測試

在測試時，`Sanctum::actingAs` 方法可用於驗證使用者，並指定應授予其 Token 的能力：

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

如果您想授予 Token 所有能力，您應該在提供給 `actingAs` 方法的能力列表中包含 `*`：

```php
Sanctum::actingAs(
    User::factory()->create(),
    ['*']
);
```