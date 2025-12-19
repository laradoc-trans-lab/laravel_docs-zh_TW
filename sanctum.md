# Laravel Sanctum

- [簡介](#introduction)
    - [運作原理](#how-it-works)
- [安裝](#installation)
- [設定](#configuration)
    - [覆寫預設模型](#overriding-default-models)
- [API Token 認證](#api-token-authentication)
    - [發行 API Token](#issuing-api-tokens)
    - [Token 能力](#token-abilities)
    - [保護路由](#protecting-routes)
    - [撤銷 Token](#revoking-tokens)
    - [Token 效期](#token-expiration)
- [SPA 認證](#spa-authentication)
    - [設定](#spa-configuration)
    - [認證](#spa-authenticating)
    - [保護路由](#protecting-spa-routes)
    - [授權私有廣播頻道](#authorizing-private-broadcast-channels)
- [行動應用程式認證](#mobile-application-authentication)
    - [發行 API Token](#issuing-mobile-api-tokens)
    - [保護路由](#protecting-mobile-api-routes)
    - [撤銷 Token](#revoking-mobile-api-tokens)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Sanctum](https://github.com/laravel/sanctum) 提供一個輕量級的認證系統，適用於 SPA (單頁應用程式)、行動應用程式以及簡單的、基於 Token 的 API。Sanctum 讓應用程式的每個使用者都能為其帳戶產生多個 API Token。這些 Token 可以被授予「abilities」(能力) / 「scopes」(權限範圍)，用以指定這些 Token 允許執行的動作。


<a name="how-it-works"></a>
### 運作原理

Laravel Sanctum 旨在解決兩個獨立的問題。讓我們在深入探討這個函式庫之前，先分別討論它們。


<a name="how-it-works-api-tokens"></a>
#### API Token

首先，Sanctum 是一個簡單的套件，您可以利用它向使用者發行 API Token，而無需 OAuth 的複雜性。此功能是受到 GitHub 及其他發行「個人存取 Token」的應用程式所啟發。例如，想像您的應用程式在「帳戶設定」中，有一個畫面可讓使用者為其帳戶產生一個 API Token。您可以使用 Sanctum 來產生和管理這些 Token。這些 Token 通常具有很長的有效期限 (數年)，但使用者可以隨時手動撤銷它們。

Laravel Sanctum 提供此功能的方式是將使用者 API Token 儲存在單一的資料庫表格中，並透過 `Authorization` 標頭來認證傳入的 HTTP 請求，該標頭應包含一個有效的 API Token。


<a name="how-it-works-spa-authentication"></a>
#### SPA 認證

其次，Sanctum 旨在提供一種簡單的方式來認證需要與 Laravel 驅動的 API 進行通訊的單頁應用程式 (SPA)。這些 SPA 可能存在於與您的 Laravel 應用程式相同的儲存庫中，或者可能是一個完全獨立的儲存庫，例如使用 Next.js 或 Nuxt 建立的 SPA。

對於此功能，Sanctum 不使用任何形式的 Token。相反地，Sanctum 使用 Laravel 內建基於 Cookie 的 Session 認證服務。通常，Sanctum 利用 Laravel 的 `web` 認證守衛來達成此目的。這提供了 CSRF 保護、Session 認證的好處，並能防止透過 XSS 洩露認證憑證。

Sanctum 只會在傳入請求源自於您自己的 SPA 前端時，才會嘗試使用 Cookie 進行認證。當 Sanctum 檢查傳入的 HTTP 請求時，它會首先檢查是否存在認證 Cookie，如果不存在，Sanctum 則會檢查 `Authorization` 標頭以尋找有效的 API Token。

> [!NOTE]
> 僅使用 Sanctum 進行 API Token 認證或僅進行 SPA 認證是完全沒問題的。僅僅因為您使用 Sanctum，並不表示您必須同時使用它提供的兩種功能。


<a name="installation"></a>
## 安裝

您可以透過 `install:api` Artisan 指令來安裝 Laravel Sanctum：

```shell
php artisan install:api
```

接下來，如果您打算利用 Sanctum 來認證 SPA，請參閱本文件中的 [SPA 認證](#spa-authentication) 章節。


<a name="configuration"></a>
## 設定


<a name="overriding-default-models"></a>
### 覆寫預設模型

儘管通常不需要，但您可以自由擴展 Sanctum 內部使用的 `PersonalAccessToken` 模型：

```php
use Laravel\Sanctum\PersonalAccessToken as SanctumPersonalAccessToken;

class PersonalAccessToken extends SanctumPersonalAccessToken
{
    // ...
}
```

接著，您可以透過 Sanctum 提供的 `usePersonalAccessTokenModel` 方法，指示 Sanctum 使用您自訂的模型。通常，您應該在應用程式的 `AppServiceProvider` 檔案中的 `boot` 方法中呼叫此方法：

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
> 您不應使用 API tokens 來認證您自己的第一方 SPA。請改用 Sanctum 內建的 [SPA 認證功能](#spa-authentication)。

<a name="issuing-api-tokens"></a>
### 發行 API Token

Sanctum 允許您發行 API tokens / 個人存取 tokens，這些 tokens 可用於認證您應用程式的 API 請求。當使用 API tokens 發出請求時，token 應包含在 `Authorization` 標頭中，作為 `Bearer` token。

要開始為使用者發行 tokens，您的 User 模型應使用 `Laravel\Sanctum\HasApiTokens` trait：

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

要發行 token，您可以使用 `createToken` 方法。`createToken` 方法會返回一個 `Laravel\Sanctum\NewAccessToken` 實例。API tokens 在儲存到資料庫之前會使用 SHA-256 雜湊處理，但您可以使用 `NewAccessToken` 實例的 `plainTextToken` 屬性來存取 token 的純文字值。您應該在 token 建立後立即將此值顯示給使用者：

```php
use Illuminate\Http\Request;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);

    return ['token' => $token->plainTextToken];
});
```

您可以使用 `HasApiTokens` trait 提供的 `tokens` Eloquent 關係來存取使用者所有的 tokens：

```php
foreach ($user->tokens as $token) {
    // ...
}
```

<a name="token-abilities"></a>
### Token 能力

Sanctum 允許您為 tokens 分配「能力 (abilities)」。能力的作用類似於 OAuth 的「scopes」。您可以將字串能力陣列作為第二個參數傳遞給 `createToken` 方法：

```php
return $user->createToken('token-name', ['server:update'])->plainTextToken;
```

當處理由 Sanctum 認證的傳入請求時，您可以使用 `tokenCan` 或 `tokenCant` 方法來判斷 token 是否具有特定能力：

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

Sanctum 還包含兩個中介層，可用於驗證傳入請求是否已使用具有特定能力的 token 進行認證。若要開始使用，請在您應用程式的 `bootstrap/app.php` 檔案中定義以下中介層別名：

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

`abilities` 中介層可以分配給路由，以驗證傳入請求的 token 是否具有所有列出的能力：

```php
Route::get('/orders', function () {
    // Token has both "check-status" and "place-orders" abilities...
})->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);
```

`ability` 中介層可以分配給路由，以驗證傳入請求的 token 是否具有*至少一個*列出的能力：

```php
Route::get('/orders', function () {
    // Token has the "check-status" or "place-orders" ability...
})->middleware(['auth:sanctum', 'ability:check-status,place-orders']);
```

<a name="first-party-ui-initiated-requests"></a>
#### 第一方 UI 發起的請求

為方便起見，如果傳入的認證請求是來自您的第一方 SPA 並且您正在使用 Sanctum 內建的 [SPA 認證](#spa-authentication)，則 `tokenCan` 方法將始終返回 `true`。

然而，這並不意味著您的應用程式必須允許使用者執行該動作。通常，您應用程式的[授權策略](/docs/{{version}}/authorization#creating-policies)將決定 token 是否已被授予執行能力的權限，並檢查使用者實例本身是否應被允許執行該動作。

例如，如果我們想像一個管理伺服器的應用程式，這可能意味著檢查 token 是否被授權更新伺服器**並且**該伺服器屬於該使用者：

```php
return $request->user()->id === $server->user_id &&
       $request->user()->tokenCan('server:update')
```

起初，允許 `tokenCan` 方法被呼叫並始終對第一方 UI 發起的請求返回 `true` 可能會感到奇怪；然而，能夠始終假設 API token 可用並可以透過 `tokenCan` 方法進行檢查是方便的。透過這種方法，您可以在應用程式的授權策略中始終呼叫 `tokenCan` 方法，而無需擔心請求是從應用程式的 UI 觸發還是由 API 的第三方消費者發起。

<a name="protecting-routes"></a>
### 保護路由

為了保護路由，使所有傳入請求都必須經過認證，您應該將 `sanctum` 認證守衛附加到您在 `routes/web.php` 和 `routes/api.php` 路由檔案中的受保護路由。此守衛將確保傳入請求被認證為有狀態的、基於 cookie 認證的請求，或者如果請求來自第三方，則包含有效的 API token 標頭。

您可能會想知道為什麼我們建議您使用 `sanctum` 守衛來認證應用程式 `routes/web.php` 檔案中的路由。請記住，Sanctum 會首先嘗試使用 Laravel 典型的基於 Session 的認證 cookie 來認證傳入請求。如果該 cookie 不存在，則 Sanctum 將嘗試使用請求的 `Authorization` 標頭中的 token 來認證請求。此外，使用 Sanctum 認證所有請求可確保我們始終可以在目前已認證的使用者實例上呼叫 `tokenCan` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

<a name="revoking-tokens"></a>
### 撤銷 Token

您可以透過使用 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關係，從資料庫中刪除 token 來「撤銷」它們：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke the token that was used to authenticate the current request...
$request->user()->currentAccessToken()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```

<a name="token-expiration"></a>
### Token 效期

預設情況下，Sanctum tokens 永不失效，只能透過[撤銷 token](#revoking-tokens) 來使其無效。然而，如果您想為應用程式的 API tokens 配置有效期限，您可以透過應用程式 `sanctum` 設定檔中定義的 `expiration` 設定選項來完成。此設定選項定義了發出的 token 在多少分鐘後將被視為過期：

```php
'expiration' => 525600,
```

如果您想獨立指定每個 token 的有效期限，您可以將有效期限作為第三個參數提供給 `createToken` 方法：

```php
return $user->createToken(
    'token-name', ['*'], now()->plus(weeks: 1)
)->plainTextToken;
```

如果您已為應用程式配置了 token 有效期限，您可能還希望[排程任務](/docs/{{version}}/scheduling)來清除應用程式中過期的 tokens。幸運的是，Sanctum 包含了一個 `sanctum:prune-expired` Artisan 命令，您可以使用它來完成此任務。例如，您可以配置一個排程任務，以刪除所有已過期至少 24 小時的 token 資料庫記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

<a name="spa-authentication"></a>
## SPA 認證

Sanctum 也提供一個簡單的方法來認證需要與 Laravel 驅動的 API 溝通的單頁應用程式 (SPAs)。這些 SPAs 可能與您的 Laravel 應用程式位在相同的儲存庫中，也可能是完全獨立的儲存庫。

針對此功能，Sanctum 不使用任何形式的 Token。相反地，Sanctum 使用 Laravel 內建基於 Cookie 的 Session 認證服務。這種認證方法提供了 CSRF 保護、Session 認證的好處，同時也防止認證憑證透過 XSS 洩漏。

> [!WARNING]
> 為了進行認證，您的 SPA 和 API 必須共用相同的頂級網域。但是，它們可以放在不同的子網域上。此外，您應該確保在請求中傳送 `Accept: application/json` 標頭以及 `Referer` 或 `Origin` 標頭。

<a name="spa-configuration"></a>
### 設定

<a name="configuring-your-first-party-domains"></a>
#### 設定您的第一方網域

首先，您應該設定您的 SPA 將從哪些網域發出請求。您可以使用 `sanctum` 設定檔中的 `stateful` 設定選項來設定這些網域。此設定決定了哪些網域在向您的 API 發出請求時，將使用 Laravel Session Cookie 來維持「有狀態的」認證。

為了幫助您設定第一方有狀態網域，Sanctum 提供了兩個輔助函式，您可以將其包含在設定中。首先，`Sanctum::currentApplicationUrlWithPort()` 將從 `APP_URL` 環境變數回傳目前的應用程式 URL，而 `Sanctum::currentRequestHost()` 將在有狀態網域列表中注入一個佔位符，該佔位符在執行時將被目前請求的主機取代，以便所有具有相同網域的請求都被視為有狀態的。

> [!WARNING]
> 如果您透過包含埠號的 URL (`127.0.0.1:8000`) 存取您的應用程式，您應該確保在網域中包含埠號。

<a name="sanctum-middleware"></a>
#### Sanctum 中介層

接下來，您應該指示 Laravel，來自您的 SPA 的傳入請求可以使用 Laravel 的 Session Cookie 進行認證，同時仍允許來自第三方或行動應用程式的請求使用 API Token 進行認證。這可以透過在應用程式的 `bootstrap/app.php` 檔案中調用 `statefulApi` 中介層方法輕鬆實現：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```

<a name="cors-and-cookies"></a>
#### CORS 與 Cookie

如果您在從不同子網域執行的 SPA 認證您的應用程式時遇到問題，您很可能錯誤設定了您的 CORS (Cross-Origin Resource Sharing) 或 Session Cookie 設定。

`config/cors.php` 設定檔預設不會發布。如果您需要自訂 Laravel 的 CORS 選項，您應該使用 `config:publish` Artisan 指令發布完整的 `cors` 設定檔：

```shell
php artisan config:publish cors
```

接下來，您應該確保您的應用程式的 CORS 設定回傳 `Access-Control-Allow-Credentials` 標頭，其值為 `True`。這可以透過將您應用程式的 `config/cors.php` 設定檔中的 `supports_credentials` 選項設定為 `true` 來實現。

此外，您應該在應用程式的全球 `axios` 實例上啟用 `withCredentials` 和 `withXSRFToken` 選項。通常，這應該在您的 `resources/js/bootstrap.js` 檔案中執行。如果您不使用 Axios 從前端發出 HTTP 請求，您應該在自己的 HTTP 客戶端上執行等效的設定：

```js
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

最後，您應該確保應用程式的 Session Cookie 網域設定支援您的根網域的任何子網域。您可以透過在應用程式的 `config/session.php` 設定檔中，在網域前加上一個開頭的 `.` 來實現此目的：

```php
'domain' => '.domain.com',
```

<a name="spa-authenticating"></a>
### 認證

<a name="csrf-protection"></a>
#### CSRF 保護

為了認證您的 SPA，您的 SPA 的「登入」頁面應首先向 `/sanctum/csrf-cookie` 端點發出請求，以初始化應用程式的 CSRF 保護：

```js
axios.get('/sanctum/csrf-cookie').then(response => {
    // Login...
});
```

在此請求期間，Laravel 將設定一個包含當前 CSRF Token 的 `XSRF-TOKEN` Cookie。然後，此 Token 應該被 URL 解碼，並在後續請求中透過 `X-XSRF-TOKEN` 標頭傳遞，某些 HTTP 客戶端函式庫 (如 Axios 和 Angular HttpClient) 會自動為您執行此操作。如果您的 JavaScript HTTP 函式庫未為您設定該值，您將需要手動將 `X-XSRF-TOKEN` 標頭設定為與此路由設定的 `XSRF-TOKEN` Cookie 的 URL 解碼值相符。

<a name="logging-in"></a>
#### 登入

一旦 CSRF 保護已初始化，您應該向您的 Laravel 應用程式的 `/login` 路由發出 `POST` 請求。此 `/login` 路由可以 [手動實作](/docs/{{version}}/authentication#authenticating-users) 或使用像 [Laravel Fortify](/docs/{{version}}/fortify) 這樣的無頭認證套件來實作。

如果登入請求成功，您將被認證，並且後續對您應用程式路由的請求將自動透過 Laravel 應用程式發給您客戶端的 Session Cookie 進行認證。此外，由於您的應用程式已經向 `/sanctum/csrf-cookie` 路由發出了請求，只要您的 JavaScript HTTP 客戶端在 `X-XSRF-TOKEN` 標頭中傳送 `XSRF-TOKEN` Cookie 的值，後續請求就應該自動獲得 CSRF 保護。

當然，如果由於長時間未活動導致您的使用者 Session 過期，對 Laravel 應用程式的後續請求可能會收到 401 或 419 HTTP 錯誤回應。在這種情況下，您應該將使用者重新導向到您的 SPA 的登入頁面。

> [!WARNING]
> 您可以自由編寫自己的 `/login` 端點；但是，您應該確保它使用 Laravel 提供的標準 [基於 Session 的認證服務](/docs/{{version}}/authentication#authenticating-users) 來認證使用者。通常，這意味著使用 `web` 認證 Guard。

<a name="protecting-spa-routes"></a>
### 保護路由

為了保護路由，使所有傳入請求都必須經過認證，您應該將 `sanctum` 認證 Guard 附加到 `routes/api.php` 檔案中的 API 路由。此 Guard 將確保傳入請求被認證為來自您的 SPA 的有狀態認證請求，或者如果請求來自第三方，則包含有效的 API Token 標頭：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

<a name="authorizing-private-broadcast-channels"></a>
### 授權私有廣播頻道

如果您的 SPA 需要透過 [私有 / 存在廣播頻道](/docs/{{version}}/broadcasting#authorizing-channels) 進行認證，您應該從應用程式的 `bootstrap/app.php` 檔案中，`withRouting` 方法裡的 `channels` 條目移除。相反地，您應該呼叫 `withBroadcasting` 方法，以便為您應用程式的廣播路由指定正確的中介層：

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

接著，為了讓 Pusher 的授權請求成功，您在初始化 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時，需要提供一個自訂的 Pusher `authorizer`。這讓您的應用程式可以設定 Pusher 使用 [已為跨網域請求正確設定](#cors-and-cookies) 的 `axios` 實例：

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

您也可以使用 Sanctum Token 來認證您的行動應用程式對 API 的請求。行動應用程式請求的認證過程與認證第三方 API 請求類似；不過，在發行 API Token 的方式上會有一些小差異。


<a name="issuing-mobile-api-tokens"></a>
### 發行 API Token

首先，建立一個路由，該路由接受使用者的 Email / 使用者名稱、密碼和設備名稱，然後將這些憑證換取一個新的 Sanctum Token。給予此端點的「設備名稱」僅供參考，可以是您想要的任何值。通常，設備名稱應該是使用者能夠辨識的名稱，例如「Nuno 的 iPhone 12」。

通常，您會從行動應用程式的「登入」畫面向 Token 端點發送請求。此端點將返回明文的 API Token，該 Token 可儲存在行動裝置上，並用於進行額外的 API 請求：

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

當行動應用程式使用此 Token 向您的應用程式發送 API 請求時，它應在 `Authorization` 標頭中以 `Bearer` Token 的形式傳遞該 Token。

> [!NOTE]
> 當為行動應用程式發行 Token 時，您也可以自由地指定 [Token 能力](#token-abilities)。


<a name="protecting-mobile-api-routes"></a>
### 保護路由

如前所述，您可以透過將 `sanctum` 認證守衛附加到路由上，來保護路由，使所有傳入的請求都必須經過認證：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-mobile-api-tokens"></a>
### 撤銷 Token

為了讓使用者撤銷發行給行動裝置的 API Token，您可以在網路應用程式 UI 的「帳號設定」部分，列出這些 Token 的名稱，並附上一個「撤銷 (Revoke)」按鈕。當使用者點擊「撤銷 (Revoke)」按鈕時，您可以從資料庫中刪除該 Token。請記住，您可以透過 `Laravel\Sanctum\HasApiTokens` Trait 提供的 `tokens` 關聯，存取使用者的 API Token：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```


<a name="testing"></a>
## 測試

在測試時，可以使用 `Sanctum::actingAs` 方法來認證使用者並指定應授予其 Token 的能力：

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

如果您想授予 Token 所有能力，則應在提供給 `actingAs` 方法的能力列表中包含 `*`：

```php
Sanctum::actingAs(
    User::factory()->create(),
    ['*']
);
```