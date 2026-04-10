# Laravel Sanctum

- [介紹](#introduction)
    - [運作原理](#how-it-works)
- [安裝](#installation)
- [設定](#configuration)
    - [覆蓋預設模型](#overriding-default-models)
- [API 令牌認證](#api-token-authentication)
    - [核發 API 令牌](#issuing-api-tokens)
    - [令牌權限 (Token Abilities)](#token-abilities)
    - [保護路由](#protecting-routes)
    - [撤銷令牌](#revoking-tokens)
    - [令牌過期](#token-expiration)
- [SPA 認證](#spa-authentication)
    - [設定](#spa-configuration)
    - [進行認證](#spa-authenticating)
    - [保護路由](#protecting-spa-routes)
    - [授權私有廣播頻道](#authorizing-private-broadcast-channels)
- [行動應用程式認證](#mobile-application-authentication)
    - [核發 API 令牌](#issuing-mobile-api-tokens)
    - [保護路由](#protecting-mobile-api-routes)
    - [撤銷令牌](#revoking-mobile-api-tokens)
- [測試](#testing)

<a name="introduction"></a>
## 介紹

[Laravel Sanctum](https://github.com/laravel/sanctum) 為 SPA (單頁應用程式)、行動應用程式以及簡單的令牌型 API 提供了一個輕量級的認證系統。Sanctum 允許您應用程式的每個使用者為其帳戶產生多個 API 令牌。這些令牌可以被授予權限 (abilities) / 範圍 (scopes)，用以指定該令牌被允許執行的操作。


<a name="how-it-works"></a>
### 運作原理

Laravel Sanctum 的存在是為了解決兩個獨立的問題。在深入研究此函式庫之前，讓我們分別討論這兩個問題。


<a name="how-it-works-api-tokens"></a>
#### API 令牌

首先，Sanctum 是一個簡單的套件，您可以使用它向使用者核發 API 令牌，而無需處理 OAuth 的複雜性。此功能靈感來自 GitHub 和其他核發「個人存取令牌 (personal access tokens)」的應用程式。例如，想像您的應用程式在「帳戶設定」中有一個畫面，使用者可以在其中為其帳戶產生一個 API 令牌。您可以使用 Sanctum 來產生並管理這些令牌。這些令牌通常具有非常長的過期時間（數年），但使用者可以隨時手動撤銷。

Laravel Sanctum 透過將使用者 API 令牌儲存在單一資料庫資料表中，並透過 `Authorization` 標頭（應包含有效的 API 令牌）來認證傳入的 HTTP 請求，來提供此功能。


<a name="how-it-works-spa-authentication"></a>
#### SPA 認證

其次，Sanctum 旨在為需要與 Laravel 驅動的 API 通訊的單頁應用程式 (SPA) 提供一種簡單的認證方式。這些 SPA 可能與您的 Laravel 應用程式存在於同一個儲存庫中，或者可能是一個完全獨立的儲存庫，例如使用 Next.js 或 Nuxt 建立的 SPA。

對於此功能，Sanctum 不使用任何形式的令牌。相反地，Sanctum 使用 Laravel 內建的基於 Cookie 的工作階段認證服務。通常，Sanctum 利用 Laravel 的 `web` 認證守衛 (guard) 來達成此目的。這提供了 CSRF 保護、工作階段認證的優勢，並能防止認證憑據透過 XSS 洩漏。

僅當傳入的請求來自您自己的 SPA 前端時，Sanctum 才會嘗試使用 Cookie 進行認證。當 Sanctum 檢查傳入的 HTTP 請求時，它會首先檢查認證 Cookie，如果不存在，Sanctum 接著會檢查 `Authorization` 標頭是否包含有效的 API 令牌。

> [!NOTE]
> 僅將 Sanctum 用於 API 令牌認證或僅用於 SPA 認證都是完全可以的。僅僅因為您使用了 Sanctum，並不意味著您必須使用它提供的這兩種功能。


<a name="installation"></a>
## 安裝

您可以使用 `install:api` Artisan 命令來安裝 Laravel Sanctum：

```shell
php artisan install:api
```

接下來，如果您計劃利用 Sanctum 來認證 SPA，請參考本文件的 [SPA 認證](#spa-authentication) 章節。


<a name="configuration"></a>
## 設定


<a name="overriding-default-models"></a>
### 覆蓋預設模型

雖然通常不需要，但您可以自由地擴充 Sanctum 內部使用的 `PersonalAccessToken` 模型：

```php
use Laravel\Sanctum\PersonalAccessToken as SanctumPersonalAccessToken;

class PersonalAccessToken extends SanctumPersonalAccessToken
{
    // ...
}
```

接著，您可以使用 Sanctum 提供的 `usePersonalAccessTokenModel` 方法指示 Sanctum 使用您的自定義模型。通常，您應該在應用程式 `AppServiceProvider` 文件的 `boot` 方法中呼叫此方法：

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
## API 令牌認證

> [!NOTE]
> 您不應該使用 API 令牌來認證您自己的第一方 SPA。請改為使用 Sanctum 內建的 [SPA 認證功能](#spa-authentication)。


<a name="issuing-api-tokens"></a>
### 核發 API 令牌

Sanctum 允許您核發 API 令牌 / 個人存取令牌 (personal access tokens)，可用於認證對您應用程式的 API 請求。在使用 API 令牌發出請求時，令牌應作為 `Bearer` 令牌包含在 `Authorization` 標頭中。

要開始為使用者核發令牌，您的 User 模型應使用 `Laravel\Sanctum\HasApiTokens` trait：

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

要核發令牌，您可以使用 `createToken` 方法。`createToken` 方法會返回一個 `Laravel\Sanctum\NewAccessToken` 實例。API 令牌在儲存到資料庫之前會使用 SHA-256 雜湊演算法進行雜湊，但您可以使用 `NewAccessToken` 實例的 `plainTextToken` 屬性來存取令牌的明文值。您應該在令牌建立後立即將此值顯示給使用者：

```php
use Illuminate\Http\Request;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);

    return ['token' => $token->plainTextToken];
});
```

您可以使用 `HasApiTokens` trait 提供的 `tokens` Eloquent 關聯來存取該使用者所有的令牌：

```php
foreach ($user->tokens as $token) {
    // ...
}
```


<a name="token-abilities"></a>
### 令牌權限 (Token Abilities)

Sanctum 允許您為令牌分配「權限 (abilities)」。權限的功能與 OAuth 的「範圍 (scopes)」類似。您可以將字串權限陣列作為 `createToken` 方法的第二個引數傳入：

```php
return $user->createToken('token-name', ['server:update'])->plainTextToken;
```

在處理由 Sanctum 認證的傳入請求時，您可以使用 `tokenCan` 或 `tokenCant` 方法來判斷令牌是否具有特定權限：

```php
if ($user->tokenCan('server:update')) {
    // ...
}

if ($user->tokenCant('server:update')) {
    // ...
}
```


<a name="token-ability-middleware"></a>
#### 令牌權限中介層

Sanctum 還包含兩個中介層，可用於驗證傳入的請求是否使用已授予特定權限的令牌進行認證。要開始使用，請在應用程式的 `bootstrap/app.php` 檔案中定義以下中介層別名：

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

`abilities` 中介層可以分配給路由，用以驗證傳入請求的令牌是否具有列出的所有權限：

```php
Route::get('/orders', function () {
    // Token has both "check-status" and "place-orders" abilities...
})->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);
```

`ability` 中介層可以分配給路由，用以驗證傳入請求的令牌是否具有列出權限中的 *至少其中一個*：

```php
Route::get('/orders', function () {
    // Token has the "check-status" or "place-orders" ability...
})->middleware(['auth:sanctum', 'ability:check-status,place-orders']);
```


<a name="first-party-ui-initiated-requests"></a>
#### 由第一方 UI 發起的請求

為了方便起見，如果傳入的已認證請求來自您的第一方 SPA，且您使用的是 Sanctum 內建的 [SPA 認證](#spa-authentication)，則 `tokenCan` 方法將始終返回 `true`。

然而，這並不一定意味著您的應用程式必須允許使用者執行該動作。通常，您應用程式的 [授權原則 (authorization policies)](/docs/{{version}}/authorization#creating-policies) 將決定該令牌是否被授予執行該權限的許可，並檢查使用者實例本身是否被允許執行該動作。

例如，如果我們想像一個管理伺服器的應用程式，這可能意味著需要檢查令牌是否獲准更新伺服器，**且** 該伺服器屬於該使用者：

```php
return $request->user()->id === $server->user_id &&
       $request->user()->tokenCan('server:update')
```

乍看之下，允許 `tokenCan` 方法在由第一方 UI 發起的請求中被呼叫並始終返回 `true` 可能很奇怪；然而，能夠始終假設 API 令牌可用並可透過 `tokenCan` 方法進行檢查是非常方便的。透過這種方法，您可以在應用程式的授權原則中隨時呼叫 `tokenCan` 方法，而無需擔心請求是由應用程式的 UI 觸發，還是由 API 的第三方消費者發起的。


<a name="protecting-routes"></a>
### 保護路由

為了保護路由以確保所有傳入請求都必須經過認證，您應該將 `sanctum` 認證守衛 (authentication guard) 附加到 `routes/web.php` 和 `routes/api.php` 路由檔案中的受保護路由上。此守衛將確保傳入請求 either 是有狀態的、以 Cookie 認證的請求，或者是來自第三方的包含有效 API 令牌標頭的請求。

您可能會好奇為什麼我們建議在應用程式的 `routes/web.php` 檔案中使用 `sanctum` 守衛來認證路由。請記住，Sanctum 會首先嘗試使用 Laravel 典型的會話認證 Cookie 來認證傳入請求。如果該 Cookie 不存在，Sanctum 則會嘗試使用請求 `Authorization` 標頭中的令牌來認證請求。此外，使用 Sanctum 認證所有請求可確保我們始終可以在目前已認證的使用者實例上呼叫 `tokenCan` 方法：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-tokens"></a>
### 撤銷令牌

您可以使用 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯，透過將令牌從資料庫中刪除來「撤銷」令牌：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke the token that was used to authenticate the current request...
$request->user()->currentAccessToken()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```


<a name="token-expiration"></a>
### 令牌過期

預設情況下，Sanctum 令牌永不過期，只能透過 [撤銷令牌](#revoking-tokens) 來使其失效。然而，如果您想為應用程式的 API 令牌設定過期時間，可以透過應用程式 `sanctum` 設定檔案中定義的 `expiration` 設定選項來達成。此設定選項定義了核發令牌後多少分鐘會被視為過期：

```php
'expiration' => 525600,
```

如果您想獨立指定每個令牌的過期時間，可以在呼叫 `createToken` 方法時，將過期時間作為第三個引數傳入：

```php
return $user->createToken(
    'token-name', ['*'], now()->plus(weeks: 1)
)->plainTextToken;
```

如果您為應用程式設定了令牌過期時間，您可能還希望 [排定一個任務](/docs/{{version}}/scheduling) 來清理應用程式中已過期的令牌。幸運的是，Sanctum 包含一個 `sanctum:prune-expired` Artisan 命令可用於達成此目的。例如，您可以設定一個排定任務，刪除所有已過期至少 24 小時的令牌資料庫記錄：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

<a name="spa-authentication"></a>
## SPA 認證

Sanctum 同樣提供了一種簡單的方法，為需要與 Laravel 驅動的 API 進行通訊的單頁應用程式 (SPAs) 進行認證。這些 SPAs 可能與您的 Laravel 應用程式位於同一個儲存庫中，也可能位於完全獨立的儲存庫中。

針對此功能，Sanctum 不使用任何形式的令牌。相反地，Sanctum 使用 Laravel 內建的基於 Cookie 的會話認證服務。這種認證方式提供了 CSRF 保護、會話認證的優勢，並能防止認證憑據透過 XSS 洩露。

> [!WARNING]
> 為了能夠進行認證，您的 SPA 與 API 必須共享相同的頂級網域。不過，它們可以被放置在不同的子網域上。此外，您應該確保在請求中發送 `Accept: application/json` 標頭，以及 `Referer` 或 `Origin` 標頭。


<a name="spa-configuration"></a>
### 設定


<a name="configuring-your-first-party-domains"></a>
#### 設定您的第一方網域

首先，您應該設定您的 SPA 將從哪些網域發出請求。您可以使用 `sanctum` 設定檔中的 `stateful` 設定選項來配置這些網域。此設定決定了哪些網域在向您的 API 發出請求時，將使用 Laravel 會話 Cookie 維持「有狀態 (stateful)」認證。

為了協助您設定第一方有狀態網域，Sanctum 提供了兩個您可以包含在設定中的輔助函式。首先，`Sanctum::currentApplicationUrlWithPort()` 將從 `APP_URL` 環境變數回傳目前的應用程式 URL；而 `Sanctum::currentRequestHost()` 則會在有狀態網域列表中注入一個佔位符，在執行時，該佔位符將被目前請求的主機 (host) 取代，使得所有來自相同網域的請求都被視為有狀態的。

> [!WARNING]
> 如果您是透過包含連接埠的 URL（例如 `127.0.0.1:8000`）訪問您的應用程式，請確保在網域中包含連接埠號碼。


<a name="sanctum-middleware"></a>
#### Sanctum 中介層

接下來，您應該告知 Laravel，來自您 SPA 的請求可以使用 Laravel 的會話 Cookie 進行認證，同時仍允許來自第三方或行動應用程式的請求使用 API 令牌進行認證。這可以透過在應用程式的 `bootstrap/app.php` 檔案中呼叫 `statefulApi` 中介層方法來輕鬆實現：

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```


<a name="cors-and-cookies"></a>
#### CORS 與 Cookie

如果您在從執行於獨立子網域的 SPA 認證應用程式時遇到問題，很可能是您的 CORS (跨來源資源共用) 或會話 Cookie 設定錯誤。

`config/cors.php` 設定檔預設不會被發布。如果您需要自訂 Laravel 的 CORS 選項，您應該使用 `config:publish` Artisan 指令發布完整的 `cors` 設定檔：

```shell
php artisan config:publish cors
```

接著，您應該確保應用程式的 CORS 設定回傳 `Access-Control-Allow-Credentials` 標頭且其值為 `True`。這可以透過將應用程式 `config/cors.php` 設定檔中的 `supports_credentials` 選項設定為 `true` 來達成。

此外，您應該在應用程式的全域 `axios` 實例上啟用 `withCredentials` 和 `withXSRFToken` 選項。通常這應該在 `resources/js/bootstrap.js` 檔案中執行。如果您不是使用 Axios 從前端發出 HTTP 請求，您應該在自己的 HTTP 用戶端上進行相同的設定：

```js
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

最後，您應該確保應用程式的會話 Cookie 網域設定支援您根網域的任何子網域。您可以在應用程式的 `config/session.php` 設定檔中，在網域前加上一個點 `.` 來實現此功能：

```php
'domain' => '.domain.com',
```


<a name="spa-authenticating"></a>
### 進行認證


<a name="csrf-protection"></a>
#### CSRF 保護

要為您的 SPA 進行認證，您的 SPA 「登入」頁面應該首先向 `/sanctum/csrf-cookie` 端點發出請求，以為應用程式初始化 CSRF 保護：

```js
axios.get('/sanctum/csrf-cookie').then(response => {
    // Login...
});
```

在此請求期間，Laravel 將設定一個包含目前 CSRF 令牌的 `XSRF-TOKEN` Cookie。此令牌隨後應被 URL 解碼，並在後續請求中透過 `X-XSRF-TOKEN` 標頭傳遞，某些 HTTP 用戶端函式庫（如 Axios 和 Angular HttpClient）會自動為您完成此操作。如果您的 JavaScript HTTP 函式庫沒有為您設定此值，您將需要手動設定 `X-XSRF-TOKEN` 標頭，使其與此路由設定的 `XSRF-TOKEN` Cookie 的 URL 解碼值相符。


<a name="logging-in"></a>
#### 登入

一旦 CSRF 保護初始化完成，您應該向 Laravel 應用程式的 `/login` 路由發出 `POST` 請求。這個 `/login` 路由可以[手動實現](/docs/{{version}}/authentication#authenticating-users)，或者使用像 [Laravel Fortify](/docs/{{version}}/fortify) 這樣的無頭認證套件。

如果登入請求成功，您將被認證，且後續向應用程式路由發出的請求將自動透過 Laravel 應用程式核發給客戶端的會話 Cookie 進行認證。此外，由於您的應用程式已經向 `/sanctum/csrf-cookie` 路由發出了請求，只要您的 JavaScript HTTP 用戶端在 `X-XSRF-TOKEN` 標頭中發送 `XSRF-TOKEN` Cookie 的值，後續請求應能自動獲得 CSRF 保護。

當然，如果使用者的會話因缺乏活動而過期，後續向 Laravel 應用程式發出的請求可能會收到 401 或 419 HTTP 錯誤回應。在這種情況下，您應該將使用者重新導向至 SPA 的登入頁面。

> [!WARNING]
> 您可以自由撰寫自己的 `/login` 端點；但您應該確保它使用 Laravel 提供的標準[基於會話的認證服務](/docs/{{version}}/authentication#authenticating-users)來對使用者進行認證。通常這意味著使用 `web` 認證守衛。


<a name="protecting-spa-routes"></a>
### 保護路由

要保護路由以確保所有進入的請求都必須經過認證，您應該將 `sanctum` 認證守衛附加到 `routes/api.php` 檔案中的 API 路由上。此守衛將確保進入的請求被認證為來自您 SPA 的有狀態認證請求，或者如果請求來自第三方，則包含有效的 API 令牌標頭：

```php
use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

<a name="authorizing-private-broadcast-channels"></a>
### 授權私有廣播頻道

如果您的 SPA 需要對 [私有 / 存在 (presence) 廣播頻道](/docs/{{version}}/broadcasting#authorizing-channels) 進行認證，您應該從應用程式 `bootstrap/app.php` 檔案中的 `withRouting` 方法中移除 `channels` 項目。相反地，您應該呼叫 `withBroadcasting` 方法，以便為您的應用程式廣播路由指定正確的中介層：

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

接下來，為了讓 Pusher 的授權請求能夠成功，您在初始化 [Laravel Echo](/docs/{{version}}/broadcasting#client-side-installation) 時需要提供一個自定義的 Pusher `authorizer`。這讓您的應用程式能夠設定 Pusher 使用[已正確設定跨網域請求](#cors-and-cookies)的 `axios` 實例：

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

您也可以使用 Sanctum 令牌來認證行動應用程式對 API 的請求。認證行動應用程式請求的流程與認證第三方 API 請求類似，但在核發 API 令牌的方式上有些微差異。


<a name="issuing-mobile-api-tokens"></a>
### 核發 API 令牌

首先，請建立一個可接收使用者電子郵件/使用者名稱、密碼以及裝置名稱的路由，然後將這些憑據交換為新的 Sanctum 令牌。提供給此端點的「裝置名稱」僅用於資訊記錄目的，可以是任何您想要的值。一般而言，裝置名稱應為使用者可識別的名稱，例如「Nuno 的 iPhone 17」。

通常，您會從行動應用程式的「登入」畫面向令牌端點發送請求。該端點將回傳明文 API 令牌，隨後可將其儲存在行動裝置上，並用於發送其他的 API 請求：

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

當行動應用程式使用令牌向您的應用程式發送 API 請求時，應將令牌作為 `Bearer` 令牌放入 `Authorization` 標頭中。

> [!NOTE]
> 為行動應用程式核發令牌時，您也可以自由指定 [令牌權限 (token abilities)](#token-abilities)。


<a name="protecting-mobile-api-routes"></a>
### 保護路由

如先前所述，您可以通过將 `sanctum` 認證守衛 (guard) 附加到路由，來保護路由以確保所有傳入的請求都必須經過認證：

```php
Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```


<a name="revoking-mobile-api-tokens"></a>
### 撤銷令牌

若要允許使用者撤銷核發給行動裝置的 API 令牌，您可以在 Web 應用程式 UI 的「帳戶設定」部分中，按名稱列出這些令牌並加上「撤銷」按鈕。當使用者點擊「撤銷」按鈕時，您可以從資料庫中刪除該令牌。請記得，您可以使用 `Laravel\Sanctum\HasApiTokens` trait 提供的 `tokens` 關聯來存取使用者的 API 令牌：

```php
// Revoke all tokens...
$user->tokens()->delete();

// Revoke a specific token...
$user->tokens()->where('id', $tokenId)->delete();
```


<a name="testing"></a>
## 測試

在測試時，可以使用 `Sanctum::actingAs` 方法來認證使用者，並指定應授予其令牌的權限：

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

如果您想授予令牌所有權限，應在提供給 `actingAs` 方法的權限列表中包含 `*`：

```php
Sanctum::actingAs(
    User::factory()->create(),
    ['*']
);
```