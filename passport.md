# Laravel Passport

- [簡介](#introduction)
    - [Passport 或 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [Token 壽命](#token-lifetimes)
    - [覆寫預設模型](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [授權碼授予](#authorization-code-grant)
    - [管理客戶端](#managing-clients)
    - [請求 Token](#requesting-tokens)
    - [管理 Token](#managing-tokens)
    - [刷新 Token](#refreshing-tokens)
    - [撤銷 Token](#revoking-tokens)
    - [清除 Token](#purging-tokens)
- [帶有 PKCE 的授權碼授予](#code-grant-pkce)
    - [建立客戶端](#creating-a-auth-pkce-grant-client)
    - [請求 Token](#requesting-auth-pkce-grant-tokens)
- [裝置授權授予](#device-authorization-grant)
    - [建立裝置碼授予客戶端](#creating-a-device-authorization-grant-client)
    - [請求 Token](#requesting-device-authorization-grant-tokens)
- [密碼授予](#password-grant)
    - [建立密碼授予客戶端](#creating-a-password-grant-client)
    - [請求 Token](#requesting-password-grant-tokens)
    - [請求所有範圍](#requesting-all-scopes)
    - [自訂使用者提供者](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱含授予](#implicit-grant)
- [客戶端憑證授予](#client-credentials-grant)
- [個人存取 Token](#personal-access-tokens)
    - [建立個人存取客戶端](#creating-a-personal-access-client)
    - [自訂使用者提供者](#customizing-the-user-provider-for-pat)
    - [管理個人存取 Token](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過中介層](#via-middleware)
    - [傳遞存取 Token](#passing-the-access-token)
- [Token 範圍](#token-scopes)
    - [定義範圍](#defining-scopes)
    - [預設範圍](#default-scope)
    - [為 Token 分配範圍](#assigning-scopes-to-tokens)
    - [檢查範圍](#checking-scopes)
- [SPA 驗證](#spa-authentication)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 在幾分鐘內就能為您的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 建立在由 Andy Millington 和 Simon Hamp 維護的 [League OAuth2 server](https://github.com/thephpleague/oauth2-server) 之上。

> [!NOTE]
> 本文件假設您已熟悉 OAuth2。如果您對 OAuth2 一無所知，請考慮在繼續之前熟悉 OAuth2 的一般 [術語](https://oauth2.thephpleague.com/terminology/) 和功能。

<a name="passport-or-sanctum"></a>
### Passport 或 Sanctum？

在開始之前，您可能需要確定您的應用程式是更適合使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum)。如果您的應用程式絕對需要支援 OAuth2，那麼您應該使用 Laravel Passport。

然而，如果您嘗試驗證單頁應用程式、行動應用程式或發布 API tokens，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；然而，它提供了更簡單的 API 驗證開發體驗。

<a name="installation"></a>
## 安裝

您可以透過 `install:api` Artisan 命令安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此命令將發布並執行資料庫遷移，以建立您的應用程式儲存 OAuth2 客戶端和存取 tokens 所需的表格。此命令還會建立生成安全存取 tokens 所需的加密金鑰。

執行 `install:api` 命令後，將 `Laravel\Passport\HasApiTokens` trait 和 `Laravel\Passport\Contracts\OAuthenticatable` interface 加入您的 `App\Models\User` 模型。此 trait 將為您的模型提供一些輔助方法，讓您可以檢查已驗證使用者的 token 和 scopes：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

最後，在您的應用程式 `config/auth.php` 設定檔中，您應該定義一個 `api` 驗證 guard 並將 `driver` 選項設定為 `passport`。這將指示您的應用程式在驗證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

<a name="deploying-passport"></a>
### 部署 Passport

首次將 Passport 部署到您的應用程式伺服器時，您可能需要執行 `passport:keys` 命令。此命令會生成 Passport 生成存取 tokens 所需的加密金鑰。生成的金鑰通常不會保留在原始碼控制中：

```shell
php artisan passport:keys
```

如有必要，您可以定義 Passport 金鑰應從何處載入的路徑。您可以使用 `Passport::loadKeysFrom` 方法來實現此目的。通常，此方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
}
```

<a name="loading-keys-from-the-environment"></a>
#### 從環境載入金鑰

或者，您可以使用 `vendor:publish` Artisan 命令發布 Passport 的設定檔：

```shell
php artisan vendor:publish --tag=passport-config
```

設定檔發布後，您可以透過將它們定義為環境變數來載入應用程式的加密金鑰：

```ini
PASSPORT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
<private key here>
-----END RSA PRIVATE KEY-----"

PASSPORT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<public key here>
-----END PUBLIC KEY-----"
```

<a name="upgrading-passport"></a>
### 升級 Passport

升級到新主要版本的 Passport 時，務必仔細審閱[升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。

<a name="configuration"></a>
## 設定

<a name="token-lifetimes"></a>
### Token 壽命

預設情況下，Passport 會發布一年後過期的長效存取 tokens。如果您想設定更長或更短的 token 壽命，您可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
use Carbon\CarbonInterval;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::tokensExpireIn(CarbonInterval::days(15));
    Passport::refreshTokensExpireIn(CarbonInterval::days(30));
    Passport::personalAccessTokensExpireIn(CarbonInterval::months(6));
}
```

> [!WARNING]
> Passport 資料庫表格中的 `expires_at` 欄位是唯讀的，僅供顯示之用。發布 tokens 時，Passport 會將過期資訊儲存在簽名和加密的 tokens 中。如果您需要使 token 失效，您應該[撤銷它](#revoking-tokens)。

<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以透過定義自己的模型並擴充對應的 Passport 模型來自由地擴充 Passport 內部使用的模型：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Passport\Passport` 類別指示 Passport 使用您的自訂模型。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 您的自訂模型：

```php
use App\Models\Passport\AuthCode;
use App\Models\Passport\Client;
use App\Models\Passport\DeviceCode;
use App\Models\Passport\RefreshToken;
use App\Models\Passport\Token;
use Laravel\Passport\Passport;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::useTokenModel(Token::class);
    Passport::useRefreshTokenModel(RefreshToken::class);
    Passport::useAuthCodeModel(AuthCode::class);
    Passport::useClientModel(Client::class);
    Passport::useDeviceCodeModel(DeviceCode::class);
}
```

<a name="overriding-routes"></a>
### 覆寫路由

有時您可能希望自訂 Passport 定義的路由。為此，您首先需要透過將 `Passport::ignoreRoutes` 加入到應用程式 `AppServiceProvider` 的 `register` 方法中來忽略 Passport 註冊的路由：

```php
use Laravel\Passport\Passport;

/**
 * Register any application services.
 */
public function register(): void
{
    Passport::ignoreRoutes();
}
```

然後，您可以將 Passport 在[其路由檔](https://github.com/laravel/passport/blob/master/routes/web.php)中定義的路由複製到應用程式的 `routes/web.php` 檔中，並根據您的喜好進行修改：

```php
Route::group([
    'as' => 'passport.',
    'prefix' => config('passport.path', 'oauth'),
    'namespace' => '\Laravel\Passport\Http\Controllers',
], function () {
    // Passport routes...
});
```

<a name="authorization-code-grant"></a>
## 授權碼授予

大多數開發者都熟悉透過授權碼使用 OAuth2。當使用授權碼時，客戶端應用程式會將使用者重新導向到您的伺服器，使用者可以在此處批准或拒絕向客戶端發行存取 Token 的請求。

首先，我們需要指示 Passport 如何回傳我們的「授權」視圖。

所有授權視圖的渲染邏輯都可以透過 `Laravel\Passport\Passport` 類別中提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Inertia\Inertia;
use Laravel\Passport\Passport;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    // By providing a view name...
    Passport::authorizationView('auth.oauth.authorize');

    // By providing a closure...
    Passport::authorizationView(
        fn ($parameters) => Inertia::render('Auth/OAuth/Authorize', [
            'request' => $parameters['request'],
            'authToken' => $parameters['authToken'],
            'client' => $parameters['client'],
            'user' => $parameters['user'],
            'scopes' => $parameters['scopes'],
        ])
    );
}
```

Passport 會自動定義回傳此視圖的 `/oauth/authorize` 路由。您的 `auth.oauth.authorize` 範本應包含一個表單，該表單向 `passport.authorizations.approve` 路由發出 POST 請求以批准授權，以及一個向 `passport.authorizations.deny` 路由發出 DELETE 請求以拒絕授權的表單。`passport.authorizations.approve` 和 `passport.authorizations.deny` 路由預期會有 `state`、`client_id` 和 `auth_token` 欄位。

<a name="managing-clients"></a>
### 管理客戶端

建立需要與您應用程式 API 互動的應用程式的開發人員，需要透過建立一個「客戶端」來向您的應用程式註冊其應用程式。通常，這包括提供其應用程式的名稱以及一個 URI，您的應用程式可以在使用者批准其授權請求後重新導向到該 URI。

<a name="managing-first-party-clients"></a>
#### 第一方客戶端

建立客戶端最簡單的方法是使用 `passport:client` Artisan 命令。此命令可用於建立第一方客戶端或測試您的 OAuth2 功能。當您執行 `passport:client` 命令時，Passport 會提示您輸入更多關於客戶端的資訊，並提供您客戶端 ID 和 Secret：

```shell
php artisan passport:client
```

如果您希望為客戶端允許多個重新導向 URI，您可以在 `passport:client` 命令提示您輸入 URI 時，使用逗號分隔列表來指定它們。任何包含逗號的 URI 都應該進行 URI 編碼：

```shell
https://third-party-app.com/callback,https://example.com/oauth/redirect
```

<a name="managing-third-party-clients"></a>
#### 第三方客戶端

由於您應用程式的使用者將無法使用 `passport:client` 命令，您可以利用 `Laravel\Passport\ClientRepository` 類別的 `createAuthorizationCodeGrantClient` 方法為指定使用者註冊客戶端：

```php
use App\Models\User;
use Laravel\Passport\ClientRepository;

$user = User::find($userId);

// Creating an OAuth app client that belongs to the given user...
$client = app(ClientRepository::class)->createAuthorizationCodeGrantClient(
    user: $user,
    name: 'Example App',
    redirectUris: ['https://third-party-app.com/callback'],
    confidential: false,
    enableDeviceFlow: true
);

// Retrieving all the OAuth app clients that belong to the user...
$clients = $user->oauthApps()->get();
```

`createAuthorizationCodeGrantClient` 方法會回傳一個 `Laravel\Passport\Client` 的實例。您可以將 `$client->id` 作為客戶端 ID，並將 `$client->plainSecret` 作為客戶端 Secret 顯示給使用者。

<a name="requesting-tokens"></a>
### 請求 Token

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重新導向以進行授權

建立客戶端後，開發者可以使用他們的客戶端 ID 和密鑰向您的應用程式請求授權碼和存取 Token。首先，消費應用程式應向您應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => 'user:read orders:create',
        'state' => $state,
        // 'prompt' => '', // "none", "consent", or "login"
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

`prompt` 參數可用於指定 Passport 應用程式的驗證行為。

如果 `prompt` 值為 `none`，若使用者尚未透過 Passport 應用程式驗證，Passport 將始終拋出驗證錯誤。如果值為 `consent`，Passport 將始終顯示授權核准畫面，即使所有範圍先前都已授予消費應用程式。當值為 `login` 時，Passport 應用程式將始終提示使用者重新登入應用程式，即使他們已經有現有的會話。

如果沒有提供 `prompt` 值，使用者只會在他們之前未授權存取消費應用程式所請求的範圍時才提示授權。

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="approving-the-request"></a>
#### 核准請求

當收到授權請求時，Passport 將根據 `prompt` 參數的值（如果存在）自動回應，並可能向使用者顯示一個模板，讓他們核准或拒絕授權請求。如果他們核准請求，他們將被重新導向回消費應用程式指定的 `redirect_uri`。`redirect_uri` 必須與建立客戶端時指定的 `redirect` URL 相符。

有時您可能希望跳過授權提示，例如在授權第一方客戶端時。您可以透過[擴展 `Client` 模型](#overriding-default-models)並定義 `skipsAuthorization` 方法來實現這一點。如果 `skipsAuthorization` 返回 `true`，客戶端將被核准，使用者將立即被重新導向回 `redirect_uri`，除非消費應用程式在重新導向以進行授權時明確設定了 `prompt` 參數：

```php
<?php

namespace App\Models\Passport;

use Illuminate\Contracts\Auth\Authenticatable;
use Laravel\Passport\Client as BaseClient;

class Client extends BaseClient
{
    /**
     * Determine if the client should skip the authorization prompt.
     *
     * @param  \Laravel\Passport\Scope[]  $scopes
     */
    public function skipsAuthorization(Authenticatable $user, array $scopes): bool
    {
        return $this->firstParty();
    }
}
```

<a name="requesting-tokens-converting-authorization-codes-to-access-tokens"></a>
#### 將授權碼轉換為存取 Token

如果使用者核准了授權請求，他們將被重新導向回消費應用程式。消費者應首先驗證 `state` 參數是否與重新導向之前儲存的值一致。如果 state 參數相符，則消費者應向您的應用程式發出 `POST` 請求以請求存取 Token。該請求應包含您的應用程式在使用者核准授權請求時發出的授權碼：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class,
        'Invalid state value.'
    );

    $response = Http::asForm()->post('https://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'your-client-id',
        'client_secret' => 'your-client-secret',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'code' => $request->code,
    ]);

    return $response->json();
});
```

此 `/oauth/token` 路由將返回一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取 Token 到期前的秒數。

> [!NOTE]
> 與 `/oauth/authorize` 路由一樣，`/oauth/token` 路由已由 Passport 為您定義。無需手動定義此路由。

<a name="managing-tokens"></a>
### 管理 Token

您可以使用 `Laravel\Passport\HasApiTokens` Trait 的 `tokens` 方法來檢索使用者已授權的 Token。例如，這可用於向您的使用者提供一個儀表板，以追蹤他們與第三方應用程式的連接：

```php
use App\Models\User;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\Facades\Date;
use Laravel\Passport\Token;

$user = User::find($userId);

// Retrieving all of the valid tokens for the user...
$tokens = $user->tokens()
    ->where('revoked', false)
    ->where('expires_at', '>', Date::now())
    ->get();

// Retrieving all the user's connections to third-party OAuth app clients...
$connections = $tokens->load('client')
    ->reject(fn (Token $token) => $token->client->firstParty())
    ->groupBy('client_id')
    ->map(fn (Collection $tokens) => [
        'client' => $tokens->first()->client,
        'scopes' => $tokens->pluck('scopes')->flatten()->unique()->values()->all(),
        'tokens_count' => $tokens->count(),
    ])
    ->values();
```

<a name="refreshing-tokens"></a>
### 刷新 Token

如果您的應用程式發行了短期的存取 Token，使用者將需要透過發行存取 Token 時提供給他們的刷新 Token 來刷新他們的存取 Token：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'refresh_token',
    'refresh_token' => 'the-refresh-token',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret', // Required for confidential clients only...
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

此 `/oauth/token` 路由將返回一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取 Token 到期前的秒數。

<a name="revoking-tokens"></a>
### 撤銷 Token

您可以透過使用 `Laravel\Passport\Token` 模型上的 `revoke` 方法來撤銷 Token。您可以使用 `Laravel\Passport\RefreshToken` 模型上的 `revoke` 方法來撤銷 Token 的刷新 Token：

```php
use Laravel\Passport\Passport;
use Laravel\Passport\Token;

$token = Passport::token()->find($tokenId);

// Revoke an access token...
$token->revoke();

// Revoke the token's refresh token...
$token->refreshToken?->revoke();

// Revoke all of the user's tokens...
User::find($userId)->tokens()->each(function (Token $token) {
    $token->revoke();
    $token->refreshToken?->revoke();
});
```

<a name="purging-tokens"></a>
### 清除 Token

當 Token 已被撤銷或過期時，您可能會想要將它們從資料庫中清除。Passport 內建的 `passport:purge` Artisan 指令可以為您完成這項任務：

```shell
# Purge revoked and expired tokens, auth codes, and device codes...
php artisan passport:purge

# Only purge tokens expired for more than 6 hours...
php artisan passport:purge --hours=6

# Only purge revoked tokens, auth codes, and device codes...
php artisan passport:purge --revoked

# Only purge expired tokens, auth codes, and device codes...
php artisan passport:purge --expired
```

您也可以在應用程式的 `routes/console.php` 檔案中設定一個 [排程工作](/docs/{{version}}/scheduling)，以便按時自動清除您的 Token：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

<a name="code-grant-pkce"></a>
## 帶有 PKCE 的授權碼授予

帶有「代碼交換證明密鑰」(PKCE) 的授權碼授予是驗證單頁應用程式或行動應用程式以存取您的 API 的安全方式。當您無法保證客戶端密鑰能被保密儲存，或者為了減輕授權碼被攻擊者攔截的威脅時，應使用此授予。結合「代碼驗證器」(code verifier) 和「代碼挑戰碼」(code challenge) 會在交換授權碼以獲取存取 Token 時取代客戶端密鑰。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立客戶端

在您的應用程式能夠透過帶有 PKCE 的授權碼授予來發出 Token 之前，您需要建立一個啟用 PKCE 的客戶端。您可以使用 `passport:client` Artisan 指令搭配 `--public` 選項來完成此操作：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求 Token

<a name="code-verifier-code-challenge"></a>
#### 代碼驗證器與代碼挑戰碼

由於此授權授予不提供客戶端密鑰，開發者需要生成代碼驗證器和代碼挑戰碼的組合才能請求 Token。

代碼驗證器應該是一個隨機字串，長度在 43 到 128 個字元之間，包含字母、數字以及 `"-"`、`"."`、`"_"`、`"~"` 等字元，如 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636)中所定義。

代碼挑戰碼應該是一個 Base64 編碼的字串，包含 URL 和檔名安全的字元。結尾的 `'='` 字元應被移除，且不應包含換行符、空白字元或其他額外字元。

```php
$encoded = base64_encode(hash('sha256', $codeVerifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

客戶端建立後，您可以使用客戶端 ID 以及生成的代碼驗證器和代碼挑戰碼從您的應用程式請求授權碼和存取 Token。首先，消費應用程式應向您應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $request->session()->put(
        'code_verifier', $codeVerifier = Str::random(128)
    );

    $codeChallenge = strtr(rtrim(
        base64_encode(hash('sha256', $codeVerifier, true))
    , '='), '+/', '-_');

    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => 'user:read orders:create',
        'state' => $state,
        'code_challenge' => $codeChallenge,
        'code_challenge_method' => 'S256',
        // 'prompt' => '', // "none", "consent", or "login"
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

<a name="code-grant-pkce-converting-authorization-codes-to-access-tokens"></a>
#### 將授權碼轉換為存取 Token

如果使用者批准了授權請求，他們將被重新導向回消費應用程式。消費者應將 `state` 參數與重新導向前儲存的值進行驗證，如同標準授權碼授予一樣。

如果 state 參數匹配，消費者應向您的應用程式發出 `POST` 請求以請求存取 Token。該請求應包含您的應用程式在使用者批准授權請求時發出的授權碼以及最初生成的代碼驗證器：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    $codeVerifier = $request->session()->pull('code_verifier');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class
    );

    $response = Http::asForm()->post('https://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'code_verifier' => $codeVerifier,
        'code' => $request->code,
    ]);

    return $response->json();
});
```

<a name="device-authorization-grant"></a>
## 裝置授權授予

OAuth2 裝置授權授予允許無瀏覽器或輸入受限的裝置，例如電視和遊戲主機，透過交換「裝置碼」來取得存取 Token。使用裝置流程時，裝置客戶端會引導使用者使用第二個裝置，例如電腦或智慧型手機，連接到您的伺服器並輸入提供的「使用者碼」，然後批准或拒絕存取請求。

首先，我們需要告知 Passport 如何回傳我們的「使用者碼」與「授權」視圖。

所有授權視圖的渲染邏輯都可以透過 `Laravel\Passport\Passport` 類別中提供的方法來自訂。通常，您應該從應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

```php
use Inertia\Inertia;
use Laravel\Passport\Passport;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    // By providing a view name...
    Passport::deviceUserCodeView('auth.oauth.device.user-code');
    Passport::deviceAuthorizationView('auth.oauth.device.authorize');

    // By providing a closure...
    Passport::deviceUserCodeView(
        fn ($parameters) => Inertia::render('Auth/OAuth/Device/UserCode')
    );

    Passport::deviceAuthorizationView(
        fn ($parameters) => Inertia::render('Auth/OAuth/Device/Authorize', [
            'request' => $parameters['request'],
            'authToken' => $parameters['authToken'],
            'client' => $parameters['client'],
            'user' => $parameters['user'],
            'scopes' => $parameters['scopes'],
        ])
    );

    // ...
}
```

Passport 會自動定義回傳這些視圖的路由。您的 `auth.oauth.device.user-code` 模板應包含一個表單，向 `passport.device.authorizations.authorize` 路由發出 GET 請求。`passport.device.authorizations.authorize` 路由預期有一個 `user_code` 查詢參數。

您的 `auth.oauth.device.authorize` 模板應包含一個表單，向 `passport.device.authorizations.approve` 路由發出 POST 請求以批准授權，以及一個向 `passport.device.authorizations.deny` 路由發出 DELETE 請求以拒絕授權的表單。`passport.device.authorizations.approve` 和 `passport.device.authorizations.deny` 路由預期有 `state`、`client_id` 和 `auth_token` 欄位。

<a name="creating-a-device-authorization-grant-client"></a>
### 建立裝置碼授予客戶端

在您的應用程式能透過裝置授權授予來發行 Token 之前，您需要建立一個啟用裝置流程的客戶端。您可以透過 `passport:client` Artisan 指令並搭配 `--device` 選項來完成。此指令將會建立一個第一方啟用裝置流程的客戶端，並提供您客戶端 ID 和密鑰：

```shell
php artisan passport:client --device
```

此外，您可以使用 `ClientRepository` 類別上的 `createDeviceAuthorizationGrantClient` 方法，為指定使用者註冊一個屬於他們的第三方客戶端：

```php
use App\Models\User;
use Laravel\Passport\ClientRepository;

$user = User::find($userId);

$client = app(ClientRepository::class)->createDeviceAuthorizationGrantClient(
    user: $user,
    name: 'Example Device',
    confidential: false,
);
```

<a name="requesting-device-authorization-grant-tokens"></a>
### 請求 Token

<a name="device-code"></a>
#### 請求裝置碼

建立客戶端後，開發者可以使用他們的客戶端 ID 從您的應用程式請求裝置碼。首先，消費裝置應向您的應用程式的 `/oauth/device/code` 路由發出 `POST` 請求以請求裝置碼：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

這將回傳一個包含 `device_code`、`user_code`、`verification_uri`、`interval` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含裝置碼過期前的秒數。`interval` 屬性包含消費裝置在輪詢 `/oauth/token` 路由時，為避免速率限制錯誤，應在請求之間等待的秒數。

> [!NOTE]
> 請記住，`/oauth/device/code` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="user-code"></a>
#### 顯示驗證 URI 與使用者碼

取得裝置碼請求後，消費裝置應引導使用者使用另一個裝置，並造訪提供的 `verification_uri` 和輸入 `user_code`，以批准授權請求。

<a name="polling-token-request"></a>
#### 輪詢 Token 請求

由於使用者將使用單獨的裝置來授予（或拒絕）存取權，消費裝置應輪詢您的應用程式的 `/oauth/token` 路由，以確定使用者何時回應了請求。消費裝置應在請求裝置碼時使用 JSON 回應中提供的最小輪詢 `interval`，以避免速率限制錯誤：

```php
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Sleep;

$interval = 5;

do {
    Sleep::for($interval)->seconds();

    $response = Http::asForm()->post('https://passport-app.test/oauth/token', [
        'grant_type' => 'urn:ietf:params:oauth:grant-type:device_code',
        'client_id' => 'your-client-id',
        'client_secret' => 'your-client-secret', // Required for confidential clients only...
        'device_code' => 'the-device-code',
    ]);

    if ($response->json('error') === 'slow_down') {
        $interval += 5;
    }
} while (in_array($response->json('error'), ['authorization_pending', 'slow_down']));

return $response->json();
```

如果使用者已批准授權請求，這將回傳一個包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取 Token 過期前的秒數。

<a name="password-grant"></a>
## 密碼授予

> [!WARNING]
> 我們不再建議使用密碼授予 Token。相反地，您應該選擇 [目前 OAuth2 Server 推薦的授予類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密碼授予允許您的其他第一方客戶端（例如行動應用程式）使用電子郵件地址/使用者名稱和密碼來取得存取 Token。這使您能夠安全地向第一方客戶端發放存取 Token，而無需使用者經歷完整的 OAuth2 授權碼重新導向流程。

要啟用密碼授予，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enablePasswordGrant();
}
```


<a name="creating-a-password-grant-client"></a>
### 建立密碼授予客戶端

在您的應用程式能夠透過密碼授予發放 Token 之前，您需要建立一個密碼授予客戶端。您可以透過 `passport:client` Artisan 指令搭配 `--password` 選項來完成此操作。

```shell
php artisan passport:client --password
```


<a name="requesting-password-grant-tokens"></a>
### 請求 Token

一旦您啟用授予並建立了密碼授予客戶端，您可以透過向 `/oauth/token` 路由發出 `POST` 請求，並提供使用者的電子郵件地址和密碼來請求存取 Token。請記住，此路由已由 Passport 註冊，因此無需手動定義。如果請求成功，您將在伺服器傳回的 JSON 回應中收到 `access_token` 和 `refresh_token`：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret', // Required for confidential clients only...
    'username' => 'taylor@laravel.com',
    'password' => 'my-password',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

> [!NOTE]
> 請記住，存取 Token 預設是長效期的。但是，如果需要，您可以自由地[設定您的最大存取 Token 壽命](#configuration)。


<a name="requesting-all-scopes"></a>
### 請求所有範圍

當使用密碼授予或客戶端憑證授予時，您可能希望為您的應用程式支援的所有範圍授權 Token。您可以透過請求 `*` 範圍來做到這一點。如果您請求 `*` 範圍，Token 實例上的 `can` 方法將始終傳回 `true`。此範圍只能分配給使用 `password` 或 `client_credentials` 授予發放的 Token：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret', // Required for confidential clients only...
    'username' => 'taylor@laravel.com',
    'password' => 'my-password',
    'scope' => '*',
]);
```


<a name="customizing-the-user-provider"></a>
### 自訂使用者提供者

如果您的應用程式使用多個[身份驗證使用者提供者](/docs/{{version}}/authentication#introduction)，您可以透過在透過 `artisan passport:client --password` 命令建立客戶端時提供 `--provider` 選項來指定密碼授予客戶端使用的使用者提供者。給定的提供者名稱應與應用程式的 `config/auth.php` 設定檔中定義的有效提供者相符。然後，您可以[使用中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自該守衛指定提供者的使用者被授權。


<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

當使用密碼授予進行身份驗證時，Passport 會將您可驗證模型 (authenticatable model) 的 `email` 屬性用作「使用者名稱」。但是，您可以透過在模型上定義 `findForPassport` 方法來自訂此行為：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\Bridge\Client;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * Find the user instance for the given username.
     */
    public function findForPassport(string $username, Client $client): User
    {
        return $this->where('username', $username)->first();
    }
}
```


<a name="customizing-the-password-validation"></a>
### 自訂密碼驗證

當使用密碼授予進行身份驗證時，Passport 會使用您模型的 `password` 屬性來驗證給定的密碼。如果您的模型沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，您可以在模型上定義 `validateForPassportPasswordGrant` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Illuminate\Support\Facades\Hash;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * Validate the password of the user for the Passport password grant.
     */
    public function validateForPassportPasswordGrant(string $password): bool
    {
        return Hash::check($password, $this->password);
    }
}
```


<a name="implicit-grant"></a>
## 隱含授予

> [!WARNING]
> 我們不再建議使用隱含授予 Token。相反地，您應該選擇 [目前 OAuth2 Server 推薦的授予類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隱含授予與授權碼授予相似；然而，Token 會直接返回給客戶端，而無需交換授權碼。此授予最常用於無法安全儲存客戶端憑證的 JavaScript 或行動應用程式。要啟用此授予，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

在您的應用程式能夠透過隱含授予發放 Token 之前，您需要建立一個隱含授予客戶端。您可以透過 `passport:client` Artisan 指令搭配 `--implicit` 選項來完成此操作。

```shell
php artisan passport:client --implicit
```

一旦授予已啟用並建立了隱含客戶端，開發人員可以使用其客戶端 ID 向您的應用程式請求存取 Token。消費應用程式應向您應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

```php
use Illuminate\Http\Request;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'token',
        'scope' => 'user:read orders:create',
        'state' => $state,
        // 'prompt' => '', // "none", "consent", or "login"
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="client-credentials-grant"></a>
## 客戶端憑證授予

客戶端憑證授予適用於機器對機器驗證。例如，您可能會在執行 API 維護任務的排程工作中使用此授予。

在您的應用程式可以透過客戶端憑證授予發行 Token 之前，您需要建立一個客戶端憑證授予客戶端。您可以使用 `passport:client` Artisan 指令的 `--client` 選項來完成此操作：

```shell
php artisan passport:client --client
```

接著，將 `Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層分配給一個路由：

```php
use Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner;

Route::get('/orders', function (Request $request) {
    // Access token is valid and the client is resource owner...
})->middleware(EnsureClientIsResourceOwner::class);
```

為了限制路由的存取到特定範圍，您可以向 `using` 方法提供所需範圍的列表：

```php
Route::get('/orders', function (Request $request) {
    // Access token is valid, the client is resource owner, and has both "servers:read" and "servers:create" scopes...
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```

<a name="retrieving-tokens"></a>
### 檢索 Token

要使用此授予類型檢索 Token，請向 `oauth/token` 端點發出請求：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'client_credentials',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret',
    'scope' => 'servers:read servers:create',
]);

return $response->json()['access_token'];
```

<a name="personal-access-tokens"></a>
## 個人存取 Token

有時，您的使用者可能希望在不經過典型授權碼重新導向流程的情況下，自行發行存取 Token。允許使用者透過您應用程式的 UI 自行發行 Token，對於讓使用者實驗您的 API 會很有用，或者通常可作為發行存取 Token 的一種更簡單的方法。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 來發行個人存取 Token，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 用於發行 API 存取 Token 的輕量級第一方函式庫。

<a name="creating-a-personal-access-client"></a>
### 建立個人存取客戶端

在您的應用程式可以發行個人存取 Token 之前，您需要建立一個個人存取客戶端。您可以透過執行帶有 `--personal` 選項的 `passport:client` Artisan 指令來完成此操作。如果您已經執行過 `passport:install` 指令，則無需再次執行此指令：

```shell
php artisan passport:client --personal
```

<a name="customizing-the-user-provider-for-pat"></a>
### 自訂使用者提供者

如果您的應用程式使用多個 [驗證使用者提供者](/docs/{{version}}/authentication#introduction)，您可以透過在 `artisan passport:client --personal` 指令建立客戶端時提供 `--provider` 選項來指定個人存取授予客戶端使用的使用者提供者。給定的提供者名稱應與您的應用程式 `config/auth.php` 設定檔中定義的有效提供者相符。然後，您可以 [使用中介層保護您的路由](#multiple-authentication-guards) 以確保只有來自該 Guard 指定提供者的使用者被授權。

<a name="managing-personal-access-tokens"></a>
### 管理個人存取 Token

建立個人存取客戶端後，您可以使用 `App\Models\User` 模型實例上的 `createToken` 方法為指定使用者發行 Token。`createToken` 方法接受 Token 名稱為第一個參數，以及可選的 [scopes](#token-scopes) 陣列作為第二個參數：

```php
use App\Models\User;
use Illuminate\Support\Facades\Date;
use Laravel\Passport\Token;

$user = User::find($userId);

// Creating a token without scopes...
$token = $user->createToken('My Token')->accessToken;

// Creating a token with scopes...
$token = $user->createToken('My Token', ['user:read', 'orders:create'])->accessToken;

// Creating a token with all scopes...
$token = $user->createToken('My Token', ['*'])->accessToken;

// Retrieving all the valid personal access tokens that belong to the user...
$tokens = $user->tokens()
    ->with('client')
    ->where('revoked', false)
    ->where('expires_at', '>', Date::now())
    ->get()
    ->filter(fn (Token $token) => $token->client->hasGrantType('personal_access'));
```

<a name="protecting-routes"></a>
## 保護路由

<a name="via-middleware"></a>
### 透過中介層

Passport 包含一個 [驗證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，它將驗證傳入請求中的存取 Token。一旦您將 `api` Guard 設定為使用 `passport` Driver，您只需在任何需要有效存取 Token 的路由上指定 `auth:api` 中介層即可：

```php
Route::get('/user', function () {
    // Only API authenticated users may access this route...
})->middleware('auth:api');
```

> [!WARNING]
> 如果您正在使用 [客戶端憑證授予](#client-credentials-grant)，您應該使用 [`Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層](#client-credentials-grant) 來保護您的路由，而不是 `auth:api` 中介層。

<a name="multiple-authentication-guards"></a>
#### 多個驗證 Guard

如果您的應用程式驗證不同類型的使用者，他們可能使用完全不同的 Eloquent 模型，您可能需要在應用程式中為每種使用者提供者類型定義一個 Guard 設定。這使您可以保護針對特定使用者提供者的請求。例如，假設 `config/auth.php` 設定檔中有以下 Guard 設定：

```php
'guards' => [
    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],

    'api-customers' => [
        'driver' => 'passport',
        'provider' => 'customers',
    ],
],
```

以下路由將利用 `api-customers` Guard (使用 `customers` 使用者提供者) 來驗證傳入的請求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> [!NOTE]
> 有關將多個使用者提供者與 Passport 搭配使用的更多資訊，請查閱 [個人存取 Token 文件](#customizing-the-user-provider-for-pat) 和 [密碼授予文件](#customizing-the-user-provider)。

<a name="passing-the-access-token"></a>
### 傳遞存取 Token

當呼叫受 Passport 保護的路由時，您應用程式的 API 消費者應在其請求的 `Authorization` 標頭中將其存取 Token 指定為 `Bearer` Token。例如，使用 `Http` Facade 時：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>
## Token 範圍

範圍讓您的 API 客戶端在請求帳戶存取授權時，可以請求一組特定的權限。例如，如果您正在建立一個電子商務應用程式，並非所有 API 消費者都需要下訂單的能力。相反地，您可以允許消費者僅請求存取訂單運送狀態的授權。換句話說，範圍允許您應用程式的使用者限制第三方應用程式代表他們執行操作。

<a name="defining-scopes"></a>
### 定義範圍

您可以使用 `Passport::tokensCan` 方法在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義 API 的範圍。`tokensCan` 方法接受一個範圍名稱和範圍描述的陣列。範圍描述可以是您想要的任何內容，並會顯示給使用者在授權核准畫面上：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::tokensCan([
        'user:read' => 'Retrieve the user info',
        'orders:create' => 'Place orders',
        'orders:read:status' => 'Check order status',
    ]);
}
```

<a name="default-scope"></a>
### 預設範圍

如果客戶端沒有請求任何特定範圍，您可以設定您的 Passport 伺服器，使用 `defaultScopes` 方法將預設範圍附加到 Token。通常，您應該從應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Passport\Passport;

Passport::tokensCan([
    'user:read' => 'Retrieve the user info',
    'orders:create' => 'Place orders',
    'orders:read:status' => 'Check order status',
]);

Passport::defaultScopes([
    'user:read',
    'orders:create',
]);
```

<a name="assigning-scopes-to-tokens"></a>
### 為 Token 分配範圍

<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

當使用授權碼授予請求存取 Token 時，消費者應將其所需的範圍指定為 `scope` 查詢字串參數。`scope` 參數應為一個以空格分隔的範圍列表：

```php
Route::get('/redirect', function () {
    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => 'user:read orders:create',
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

<a name="when-issuing-personal-access-tokens"></a>
#### 核發個人存取 Token 時

如果您使用 `App\Models\User` 模型中的 `createToken` 方法核發個人存取 Token，您可以將所需範圍的陣列作為該方法的第二個參數傳遞：

```php
$token = $user->createToken('My Token', ['orders:create'])->accessToken;
```

<a name="checking-scopes"></a>
### 檢查範圍

Passport 包含兩個中介層，可用來驗證傳入請求是否已使用授予特定範圍的 Token 進行驗證。

<a name="check-for-all-scopes"></a>
#### 檢查所有範圍

`Laravel\Passport\Http\Middleware\CheckToken` 中介層可以指派給路由，以驗證傳入請求的存取 Token 是否具有所有列出的範圍：

```php
use Laravel\Passport\Http\Middleware\CheckToken;

Route::get('/orders', function () {
    // Access token has both "orders:read" and "orders:create" scopes...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);
```

<a name="check-for-any-scopes"></a>
#### 檢查任一範圍

`Laravel\Passport\Http\Middleware\CheckTokenForAnyScope` 中介層可以指派給路由，以驗證傳入請求的存取 Token 是否具有列出的範圍中的「至少一個」：

```php
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

Route::get('/orders', function () {
    // Access token has either "orders:read" or "orders:create" scope...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```

<a name="checking-scopes-on-a-token-instance"></a>
#### 檢查 Token 實例上的範圍

一旦存取 Token 驗證請求進入您的應用程式，您仍然可以使用已驗證的 `App\Models\User` 實例上的 `tokenCan` 方法檢查 Token 是否具有特定範圍：

```php
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('orders:create')) {
        // ...
    }
});
```

<a name="additional-scope-methods"></a>
#### 額外的範圍方法

`scopeIds` 方法會傳回所有已定義 ID / 名稱的陣列：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法會傳回所有已定義範圍的陣列，這些範圍是 `Laravel\Passport\Scope` 的實例：

```php
Passport::scopes();
```

`scopesFor` 方法會傳回與指定 ID / 名稱相符的 `Laravel\Passport\Scope` 實例陣列：

```php
Passport::scopesFor(['user:read', 'orders:create']);
```

您可以使用 `hasScope` 方法判斷是否已定義特定範圍：

```php
Passport::hasScope('orders:create');
```

<a name="spa-authentication"></a>
## SPA 驗證

在建立 API 時，能夠從您的 JavaScript 應用程式中使用自己的 API 會非常有幫助。這種 API 開發方法允許您的應用程式使用您與世界共享的相同 API。同一個 API 可以被您的網路應用程式、行動應用程式、第三方應用程式以及您可能在各種套件管理工具上發布的任何 SDK 使用。

通常，如果您想從 JavaScript 應用程式中使用您的 API，您需要手動將存取 Token 傳送給應用程式，並在每次請求時傳遞它給您的應用程式。然而，Passport 包含一個可以為您處理此事的中介層。您所需要做的，就是將 `CreateFreshApiToken` 中介層附加到應用程式 `bootstrap/app.php` 檔案中的 `web` 中介層群組：

```php
use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        CreateFreshApiToken::class,
    ]);
})
```

> [!WARNING]
> 您應該確保 `CreateFreshApiToken` 中介層是您中介層堆疊中列出的最後一個中介層。

這個中介層會將 `laravel_token` cookie 附加到您的傳出回應中。這個 cookie 包含一個加密的 JWT，Passport 將用它來驗證來自您的 JavaScript 應用程式的 API 請求。JWT 的壽命等於您 `session.lifetime` 的設定值。現在，由於瀏覽器會自動將 cookie 與所有後續請求一起傳送，您可以向應用程式的 API 發出請求，而無需明確傳遞存取 Token：

```js
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

<a name="customizing-the-cookie-name"></a>
#### 自訂 Cookie 名稱

如有需要，您可以使用 `Passport::cookie` 方法自訂 `laravel_token` cookie 的名稱。通常，您應該從應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::cookie('custom_name');
}
```

<a name="csrf-protection"></a>
#### CSRF 保護

當使用此驗證方法時，您需要確保請求中包含有效的 CSRF Token 標頭。骨架應用程式和所有入門套件中包含的預設 Laravel JavaScript 鷹架包含了 [Axios](https://github.com/axios/axios) 實例，它會自動使用加密的 `XSRF-TOKEN` cookie 值在同源請求中傳送 `X-XSRF-TOKEN` 標頭。

> [!NOTE]
> 如果您選擇傳送 `X-CSRF-TOKEN` 標頭而不是 `X-XSRF-TOKEN`，您將需要使用 `csrf_token()` 提供的未加密 Token。

<a name="events"></a>
## 事件

Passport 在發行存取 Token 和刷新 Token 時會觸發事件。您可以[監聽這些事件](/docs/{{version}}/events)來修剪或撤銷資料庫中的其他存取 Token：

<div class="overflow-auto">

| 事件名稱                                      |
| --------------------------------------------- |
| `Laravel\Passport\Events\AccessTokenCreated`  |
| `Laravel\Passport\Events\AccessTokenRevoked`  |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>


<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用來指定當前經過驗證的使用者及其範圍。傳給 `actingAs` 方法的第一個參數是使用者實例，第二個則是應授予使用者 Token 的範圍陣列：

```php tab=Pest
use App\Models\User;
use Laravel\Passport\Passport;

test('orders can be created', function () {
    Passport::actingAs(
        User::factory()->create(),
        ['orders:create']
    );

    $response = $this->post('/api/orders');

    $response->assertStatus(201);
});
```

```php tab=PHPUnit
use App\Models\User;
use Laravel\Passport\Passport;

public function test_orders_can_be_created(): void
{
    Passport::actingAs(
        User::factory()->create(),
        ['orders:create']
    );

    $response = $this->post('/api/orders');

    $response->assertStatus(201);
}
```

Passport 的 `actingAsClient` 方法可用來指定當前經過驗證的客戶端及其範圍。傳給 `actingAsClient` 方法的第一個參數是客戶端實例，第二個則是應授予客戶端 Token 的範圍陣列：

```php tab=Pest
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

test('servers can be retrieved', function () {
    Passport::actingAsClient(
        Client::factory()->create(),
        ['servers:read']
    );

    $response = $this->get('/api/servers');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

public function test_servers_can_be_retrieved(): void
{
    Passport::actingAsClient(
        Client::factory()->create(),
        ['servers:read']
    );

    $response = $this->get('/api/servers');

    $response->assertStatus(200);
}
```