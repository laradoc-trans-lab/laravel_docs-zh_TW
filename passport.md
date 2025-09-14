# Laravel Passport

- [簡介](#introduction)
    - [Passport 還是 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [Token 存活時間](#token-lifetimes)
    - [覆寫預設模型](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [授權碼授權 (Authorization Code Grant)](#authorization-code-grant)
    - [管理 Client](#managing-clients)
    - [請求 Token](#requesting-tokens)
    - [管理 Token](#managing-tokens)
    - [更新 Token](#refreshing-tokens)
    - [撤銷 Token](#revoking-tokens)
    - [清除 Token](#purging-tokens)
- [附帶 PKCE 的授權碼授權 (Authorization Code Grant With PKCE)](#code-grant-pkce)
    - [建立 Client](#creating-a-auth-pkce-grant-client)
    - [請求 Token](#requesting-auth-pkce-grant-tokens)
- [裝置授權碼授權 (Device Authorization Grant)](#device-authorization-grant)
    - [建立裝置碼授權 Client](#creating-a-device-authorization-grant-client)
    - [請求 Token](#requesting-device-authorization-grant-tokens)
- [密碼授權 (Password Grant)](#password-grant)
    - [建立密碼授權 Client](#creating-a-password-grant-client)
    - [請求 Token](#requesting-password-grant-tokens)
    - [請求所有 Scope](#requesting-all-scopes)
    - [自訂使用者 Provider](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱含授權 (Implicit Grant)](#implicit-grant)
- [Client 憑證授權 (Client Credentials Grant)](#client-credentials-grant)
- [個人存取 Token (Personal Access Tokens)](#personal-access-tokens)
    - [建立個人存取 Client](#creating-a-personal-access-client)
    - [自訂使用者 Provider](#customizing-the-user-provider-for-pat)
    - [管理個人存取 Token](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過 Middleware](#via-middleware)
    - [傳遞存取 Token](#passing-the-access-token)
- [Token Scope](#token-scopes)
    - [定義 Scope](#defining-scopes)
    - [預設 Scope](#default-scope)
    - [為 Token 指派 Scope](#assigning-scopes-to-tokens)
    - [檢查 Scope](#checking-scopes)
- [SPA 驗證](#spa-authentication)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 可以在幾分鐘內為您的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 建構於由 Andy Millington 和 Simon Hamp 維護的 [League OAuth2 伺服器](https://github.com/thephpleague/oauth2-server)之上。

> [!NOTE]
> 本文件假設您已熟悉 OAuth2。如果您對 OAuth2 一無所知，建議您在繼續之前先熟悉 OAuth2 的一般[術語](https://oauth2.thephpleague.com/terminology/)和功能。

<a name="passport-or-sanctum"></a>
### Passport 還是 Sanctum？

在開始之前，您可能需要判斷您的應用程式更適合使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum)。如果您的應用程式絕對需要支援 OAuth2，那麼您應該使用 Laravel Passport。

然而，如果您嘗試驗證單頁應用程式 (SPA)、行動應用程式或發行 API Token，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；但是，它提供了更簡單的 API 驗證開發體驗。

<a name="installation"></a>
## 安裝

您可以透過 `install:api` Artisan 命令安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此命令將發布並執行資料庫遷移，以建立您的應用程式儲存 OAuth2 Client 和存取 Token 所需的資料表。該命令還將建立產生安全存取 Token 所需的加密金鑰。

執行 `install:api` 命令後，將 `Laravel\Passport\HasApiTokens` Trait 和 `Laravel\Passport\Contracts\OAuthenticatable` 介面新增到您的 `App\Models\User` 模型中。此 Trait 將為您的模型提供一些輔助方法，讓您可以檢查已驗證使用者的 Token 和 Scope：

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

最後，在您的應用程式的 `config/auth.php` 設定檔中，您應該定義一個 `api` 驗證 Guard 並將 `driver` 選項設定為 `passport`。這將指示您的應用程式在驗證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

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

首次將 Passport 部署到您的應用程式伺服器時，您可能需要執行 `passport:keys` 命令。此命令會產生 Passport 產生存取 Token 所需的加密金鑰。產生的金鑰通常不會保留在原始碼控制中：

```shell
php artisan passport:keys
```

如有必要，您可以定義 Passport 金鑰的載入路徑。您可以使用 `Passport::loadKeysFrom` 方法來完成此操作。通常，此方法應從您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
#### 從環境變數載入金鑰

或者，您可以使用 `vendor:publish` Artisan 命令發布 Passport 的設定檔：

```shell
php artisan vendor:publish --tag=passport-config
```

發布設定檔後，您可以透過將應用程式的加密金鑰定義為環境變數來載入它們：

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

升級到 Passport 的新主要版本時，務必仔細查閱[升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。

<a name="configuration"></a>
## 設定

<a name="token-lifetimes"></a>
### Token 存活時間

預設情況下，Passport 會發行一年後過期的長效存取 Token。如果您想設定更長/更短的 Token 存活時間，您可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應從您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> Passport 資料庫資料表上的 `expires_at` 欄位是唯讀的，僅供顯示之用。發行 Token 時，Passport 會將過期資訊儲存在已簽署和加密的 Token 中。如果您需要使 Token 失效，您應該[撤銷它](#revoking-tokens)。

<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以自由地透過定義自己的模型並擴展相應的 Passport 模型來擴展 Passport 內部使用的模型：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Passport\Passport` 類別指示 Passport 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 您的自訂模型：

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

有時您可能希望自訂 Passport 定義的路由。為此，您首先需要透過將 `Passport::ignoreRoutes` 新增到應用程式的 `AppServiceProvider` 的 `register` 方法中來忽略 Passport 註冊的路由：

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

然後，您可以將 Passport 在[其路由檔案](https://github.com/laravel/passport/blob/master/routes/web.php)中定義的路由複製到應用程式的 `routes/web.php` 檔案中，並根據您的喜好進行修改：

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
## 授權碼授權 (Authorization Code Grant)

透過授權碼使用 OAuth2 是大多數開發人員熟悉 OAuth2 的方式。使用授權碼時，Client 應用程式會將使用者重新導向到您的伺服器，使用者將在此處批准或拒絕向 Client 發行存取 Token 的請求。

首先，我們需要指示 Passport 如何回傳我們的「授權」視圖。

所有授權視圖的渲染邏輯都可以使用 `Laravel\Passport\Passport` 類別提供的適當方法進行自訂。通常，您應該從應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Passport 將自動定義 `/oauth/authorize` 路由，該路由會回傳此視圖。您的 `auth.oauth.authorize` 模板應包含一個表單，該表單向 `passport.authorizations.approve` 路由發出 POST 請求以批准授權，以及一個表單，該表單向 `passport.authorizations.deny` 路由發出 DELETE 請求以拒絕授權。`passport.authorizations.approve` 和 `passport.authorizations.deny` 路由需要 `state`、`client_id` 和 `auth_token` 欄位。

<a name="managing-clients"></a>
### 管理 Client

開發人員建立需要與您的應用程式 API 互動的應用程式時，需要透過建立「Client」來向您的應用程式註冊他們的應用程式。通常，這包括提供其應用程式的名稱以及一個 URI，您的應用程式可以在使用者批准其授權請求後重新導向到該 URI。

<a name="managing-first-party-clients"></a>
#### 第一方 Client

建立 Client 最簡單的方法是使用 `passport:client` Artisan 命令。此命令可用於建立第一方 Client 或測試您的 OAuth2 功能。當您執行 `passport:client` 命令時，Passport 將提示您輸入有關 Client 的更多資訊，並為您提供 Client ID 和 Secret：

```shell
php artisan passport:client
```

如果您想為您的 Client 允許多個重新導向 URI，您可以在 `passport:client` 命令提示您輸入 URI 時，使用逗號分隔列表指定它們。任何包含逗號的 URI 都應進行 URI 編碼：

```shell
https://third-party-app.com/callback,https://example.com/oauth/redirect
```

<a name="managing-third-party-clients"></a>
#### 第三方 Client

由於您的應用程式使用者無法使用 `passport:client` 命令，您可以使用 `Laravel\Passport\ClientRepository` 類別的 `createAuthorizationCodeGrantClient` 方法為給定使用者註冊 Client：

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

`createAuthorizationCodeGrantClient` 方法會回傳 `Laravel\Passport\Client` 的實例。您可以將 `$client->id` 顯示為 Client ID，將 `$client->plainSecret` 顯示為 Client Secret 給使用者。

<a name="requesting-tokens"></a>
### 請求 Token

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重新導向以進行授權

Client 建立後，開發人員可以使用其 Client ID 和 Secret 從您的應用程式請求授權碼和存取 Token。首先，消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

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

如果 `prompt` 值為 `none`，如果使用者尚未透過 Passport 應用程式進行驗證，Passport 將始終拋出驗證錯誤。如果值為 `consent`，Passport 將始終顯示授權批准畫面，即使所有 Scope 先前已授予消費應用程式。當值為 `login` 時，Passport 應用程式將始終提示使用者重新登入應用程式，即使他們已經有現有的 Session。

如果未提供 `prompt` 值，則僅當使用者先前未授權存取消費應用程式所請求的 Scope 時，才會提示使用者進行授權。

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="approving-the-request"></a>
#### 批准請求

收到授權請求時，Passport 將根據 `prompt` 參數的值 (如果存在) 自動回應，並可能向使用者顯示一個模板，允許他們批准或拒絕授權請求。如果他們批准請求，他們將被重新導向回消費應用程式指定的 `redirect_uri`。`redirect_uri` 必須與建立 Client 時指定的 `redirect` URL 相符。

有時您可能希望跳過授權提示，例如在授權第一方 Client 時。您可以透過[擴展 `Client` 模型](#overriding-default-models)並定義 `skipsAuthorization` 方法來實現此目的。如果 `skipsAuthorization` 回傳 `true`，Client 將被批准，並且使用者將立即被重新導向回 `redirect_uri`，除非消費應用程式在重新導向以進行授權時明確設定了 `prompt` 參數：

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

如果使用者批准授權請求，他們將被重新導向回消費應用程式。消費者應首先根據重新導向之前儲存的值驗證 `state` 參數。如果 `state` 參數匹配，則消費者應向您的應用程式發出 `POST` 請求以請求存取 Token。該請求應包含您的應用程式在使用者批准授權請求時發出的授權碼：

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

此 `/oauth/token` 路由將回傳一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取 Token 過期前的秒數。

> [!NOTE]
> 與 `/oauth/authorize` 路由一樣，`/oauth/token` 路由由 Passport 為您定義。無需手動定義此路由。

<a name="managing-tokens"></a>
### 管理 Token

您可以使用 `Laravel\Passport\HasApiTokens` Trait 的 `tokens` 方法來檢索使用者已授權的 Token。例如，這可用於為您的使用者提供一個儀表板，以追蹤他們與第三方應用程式的連接：

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
### 更新 Token

如果您的應用程式發行短效存取 Token，使用者將需要透過發行存取 Token 時提供給他們的 Refresh Token 來更新其存取 Token：

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

此 `/oauth/token` 路由將回傳一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取 Token 過期前的秒數。

<a name="revoking-tokens"></a>
### 撤銷 Token

您可以使用 `Laravel\Passport\Token` 模型上的 `revoke` 方法撤銷 Token。您可以使用 `Laravel\Passport\RefreshToken` 模型上的 `revoke` 方法撤銷 Token 的 Refresh Token：

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

當 Token 被撤銷或過期時，您可能希望將它們從資料庫中清除。Passport 包含的 `passport:purge` Artisan 命令可以為您完成此操作：

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

您也可以在應用程式的 `routes/console.php` 檔案中設定[排程任務](/docs/{{version}}/scheduling)，以定期自動清除您的 Token：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

<a name="code-grant-pkce"></a>
## 附帶 PKCE 的授權碼授權 (Authorization Code Grant With PKCE)

附帶「Proof Key for Code Exchange」(PKCE) 的授權碼授權是一種安全的方式，用於驗證單頁應用程式或行動應用程式以存取您的 API。當您無法保證 Client Secret 將被機密儲存，或者為了減輕授權碼被攻擊者攔截的威脅時，應使用此授權。當交換授權碼以獲取存取 Token 時，「Code Verifier」和「Code Challenge」的組合取代了 Client Secret。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立 Client

在您的應用程式可以透過附帶 PKCE 的授權碼授權發行 Token 之前，您需要建立一個啟用 PKCE 的 Client。您可以使用 `passport:client` Artisan 命令並帶有 `--public` 選項來完成此操作：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求 Token

<a name="code-verifier-code-challenge"></a>
#### Code Verifier 和 Code Challenge

由於此授權不提供 Client Secret，開發人員需要產生 Code Verifier 和 Code Challenge 的組合才能請求 Token。

Code Verifier 應該是一個隨機字串，長度介於 43 到 128 個字元之間，包含字母、數字以及 `"-"`、`"."`、`"_"`、`"~"` 字元，如 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636)中所定義。

Code Challenge 應該是一個 Base64 編碼的字串，帶有 URL 和檔案名安全的字元。應移除尾隨的 `'='` 字元，並且不應存在換行符、空白字元或其他額外字元。

```php
$encoded = base64_encode(hash('sha256', $codeVerifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

Client 建立後，您可以使用 Client ID 以及產生的 Code Verifier 和 Code Challenge 從您的應用程式請求授權碼和存取 Token。首先，消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求：

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

如果使用者批准授權請求，他們將被重新導向回消費應用程式。消費者應根據重新導向之前儲存的值驗證 `state` 參數，如同標準授權碼授權一樣。

如果 `state` 參數匹配，消費者應向您的應用程式發出 `POST` 請求以請求存取 Token。該請求應包含您的應用程式在使用者批准授權請求時發出的授權碼以及最初產生的 Code Verifier：

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
## 裝置授權碼授權 (Device Authorization Grant)

OAuth2 裝置授權碼授權允許無瀏覽器或輸入受限的裝置 (例如電視和遊戲機) 透過交換「裝置碼」來取得存取 Token。使用裝置流程時，裝置 Client 將指示使用者使用輔助裝置 (例如電腦或智慧型手機) 並連接到您的伺服器，使用者將在此處輸入提供的「使用者碼」並批准或拒絕存取請求。

首先，我們需要指示 Passport 如何回傳我們的「使用者碼」和「授權」視圖。

所有授權視圖的渲染邏輯都可以使用 `Laravel\Passport\Passport` 類別提供的適當方法進行自訂。通常，您應該從應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

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

Passport 將自動定義回傳這些視圖的路由。您的 `auth.oauth.device.user-code` 模板應包含一個表單，該表單向 `passport.device.authorizations.authorize` 路由發出 GET 請求。`passport.device.authorizations.authorize` 路由需要 `user_code` 查詢參數。

您的 `auth.oauth.device.authorize` 模板應包含一個表單，該表單向 `passport.device.authorizations.approve` 路由發出 POST 請求以批准授權，以及一個表單，該表單向 `passport.device.authorizations.deny` 路由發出 DELETE 請求以拒絕授權。`passport.device.authorizations.approve` 和 `passport.device.authorizations.deny` 路由需要 `state`、`client_id` 和 `auth_token` 欄位。

<a name="creating-a-device-authorization-grant-client"></a>
### 建立裝置授權碼授權 Client

在您的應用程式可以透過裝置授權碼授權發行 Token 之前，您需要建立一個啟用裝置流程的 Client。您可以使用 `passport:client` Artisan 命令並帶有 `--device` 選項來完成此操作。此命令將建立一個第一方啟用裝置流程的 Client，並為您提供 Client ID 和 Secret：

```shell
php artisan passport:client --device
```

此外，您可以使用 `ClientRepository` 類別上的 `createDeviceAuthorizationGrantClient` 方法來註冊屬於給定使用者的第三方 Client：

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

Client 建立後，開發人員可以使用其 Client ID 從您的應用程式請求裝置碼。首先，消費裝置應向您的應用程式的 `/oauth/device/code` 路由發出 `POST` 請求以請求裝置碼：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

這將回傳一個 JSON 回應，其中包含 `device_code`、`user_code`、`verification_uri`、`interval` 和 `expires_in` 屬性。`expires_in` 屬性包含裝置碼過期前的秒數。`interval` 屬性包含消費裝置在輪詢 `/oauth/token` 路由時應等待的秒數，以避免速率限制錯誤。

> [!NOTE]
> 請記住，`/oauth/device/code` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="user-code"></a>
#### 顯示驗證 URI 和使用者碼

取得裝置碼請求後，消費裝置應指示使用者使用另一個裝置並造訪提供的 `verification_uri` 並輸入 `user_code` 以批准授權請求。

<a name="polling-token-request"></a>
#### 輪詢 Token 請求

由於使用者將使用單獨的裝置來授予 (或拒絕) 存取權限，因此消費裝置應輪詢您的應用程式的 `/oauth/token` 路由，以確定使用者何時回應了請求。消費裝置應使用請求裝置碼時 JSON 回應中提供的最小輪詢 `interval`，以避免速率限制錯誤：

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

如果使用者已批准授權請求，這將回傳一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取 Token 過期前的秒數。

<a name="password-grant"></a>
## 密碼授權 (Password Grant)

> [!WARNING]
> 我們不再建議使用密碼授權 Token。相反，您應該選擇 [OAuth2 伺服器目前推薦的授權類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密碼授權允許您的其他第一方 Client (例如行動應用程式) 使用電子郵件地址/使用者名稱和密碼取得存取 Token。這允許您安全地向您的第一方 Client 發行存取 Token，而無需您的使用者經歷整個 OAuth2 授權碼重新導向流程。

要啟用密碼授權，請在您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

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
### 建立密碼授權 Client

在您的應用程式可以透過密碼授權發行 Token 之前，您需要建立一個密碼授權 Client。您可以使用 `passport:client` Artisan 命令並帶有 `--password` 選項來完成此操作。

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>
### 請求 Token

啟用授權並建立密碼授權 Client 後，您可以透過向 `/oauth/token` 路由發出 `POST` 請求並帶有使用者的電子郵件地址和密碼來請求存取 Token。請記住，此路由已由 Passport 註冊，因此無需手動定義。如果請求成功，您將在伺服器的 JSON 回應中收到 `access_token` 和 `refresh_token`：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret', // Required for confidential clients only...
    'username' => 'taylor @laravel.com',
    'password' => 'my-password',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

> [!NOTE]
> 請記住，存取 Token 預設是長效的。但是，您可以根據需要[設定您的最大存取 Token 存活時間](#configuration)。

<a name="requesting-all-scopes"></a>
### 請求所有 Scope

使用密碼授權或 Client 憑證授權時，您可能希望為 Token 授權您的應用程式支援的所有 Scope。您可以透過請求 `*` Scope 來完成此操作。如果您請求 `*` Scope，Token 實例上的 `can` 方法將始終回傳 `true`。此 Scope 只能分配給使用 `password` 或 `client_credentials` 授權發行的 Token：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret', // Required for confidential clients only...
    'username' => 'taylor @laravel.com',
    'password' => 'my-password',
    'scope' => '*',
]);
```

<a name="customizing-the-user-provider"></a>
### 自訂使用者 Provider

如果您的應用程式使用多個[驗證使用者 Provider](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --password` 命令建立 Client 時，透過提供 `--provider` 選項來指定密碼授權 Client 使用哪個使用者 Provider。給定的 Provider 名稱應與您的應用程式的 `config/auth.php` 設定檔中定義的有效 Provider 相符。然後，您可以[使用 Middleware 保護您的路由](#multiple-authentication-guards)，以確保只有來自 Guard 指定 Provider 的使用者才被授權。

<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

使用密碼授權進行驗證時，Passport 將使用您的可驗證模型的 `email` 屬性作為「使用者名稱」。但是，您可以透過在模型上定義 `findForPassport` 方法來自訂此行為：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens, Notifiable;

    /**
     * Find the user instance for the given username.
     */
    public function findForPassport(string $username): User
    {
        return $this->where('username', $username)->first();
    }
}
```

<a name="customizing-the-password-validation"></a>
### 自訂密碼驗證

使用密碼授權進行驗證時，Passport 將使用您模型的 `password` 屬性來驗證給定的密碼。如果您的模型沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，您可以在模型上定義 `validateForPassportPasswordGrant` 方法：

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
## 隱含授權 (Implicit Grant)

> [!WARNING]
> 我們不再建議使用隱含授權 Token。相反，您應該選擇 [OAuth2 伺服器目前推薦的授權類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隱含授權類似於授權碼授權；但是，Token 會在不交換授權碼的情況下回傳給 Client。此授權最常用於 Client 憑證無法安全儲存的 JavaScript 或行動應用程式。要啟用授權，請在您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

在您的應用程式可以透過隱含授權發行 Token 之前，您需要建立一個隱含授權 Client。您可以使用 `passport:client` Artisan 命令並帶有 `--implicit` 選項來完成此操作。

```shell
php artisan passport:client --implicit
```

啟用授權並建立隱含 Client 後，開發人員可以使用其 Client ID 從您的應用程式請求存取 Token。消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

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
## Client 憑證授權 (Client Credentials Grant)

Client 憑證授權適用於機器對機器驗證。例如，您可以在透過 API 執行維護任務的排程任務中使用此授權。

在您的應用程式可以透過 Client 憑證授權發行 Token 之前，您需要建立一個 Client 憑證授權 Client。您可以使用 `passport:client` Artisan 命令的 `--client` 選項來完成此操作：

```shell
php artisan passport:client --client
```

接下來，將 `Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` Middleware 分配給路由：

```php
use Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner;

Route::get('/orders', function (Request $request) {
    // Access token is valid and the client is resource owner...
})->middleware(EnsureClientIsResourceOwner::class);
```

要將路由的存取權限限制為特定 Scope，您可以向 `using` 方法提供所需 Scope 的列表：

```php
Route::get('/orders', function (Request $request) {
    // Access token is valid, the client is resource owner, and has both "servers:read" and "servers:create" scopes...
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```

<a name="retrieving-tokens"></a>
### 檢索 Token

要使用此授權類型檢索 Token，請向 `oauth/token` 端點發出請求：

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
## 個人存取 Token (Personal Access Tokens)

有時，您的使用者可能希望在不經過典型的授權碼重新導向流程的情況下，自行發行存取 Token。允許使用者透過您的應用程式 UI 自行發行 Token 對於允許使用者試驗您的 API 可能很有用，或者可以作為發行存取 Token 的更簡單方法。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 發行個人存取 Token，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 輕量級的第一方函式庫，用於發行 API 存取 Token。

<a name="creating-a-personal-access-client"></a>
### 建立個人存取 Client

在您的應用程式可以發行個人存取 Token 之前，您需要建立一個個人存取 Client。您可以透過執行 `passport:client` Artisan 命令並帶有 `--personal` 選項來完成此操作。如果您已經執行了 `passport:install` 命令，則無需執行此命令：

```shell
php artisan passport:client --personal
```

<a name="customizing-the-user-provider-for-pat"></a>
### 自訂使用者 Provider

如果您的應用程式使用多個[驗證使用者 Provider](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --personal` 命令建立 Client 時，透過提供 `--provider` 選項來指定個人存取授權 Client 使用哪個使用者 Provider。給定的 Provider 名稱應與您的應用程式的 `config/auth.php` 設定檔中定義的有效 Provider 相符。然後，您可以[使用 Middleware 保護您的路由](#multiple-authentication-guards)，以確保只有來自 Guard 指定 Provider 的使用者才被授權。

<a name="managing-personal-access-tokens"></a>
### 管理個人存取 Token

建立個人存取 Client 後，您可以使用 `App\Models\User` 模型實例上的 `createToken` 方法為給定使用者發行 Token。`createToken` 方法接受 Token 的名稱作為第一個參數，以及一個可選的 [Scope](#token-scopes) 陣列作為第二個參數：

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
### 透過 Middleware

Passport 包含一個[驗證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，它將驗證傳入請求上的存取 Token。一旦您將 `api` Guard 設定為使用 `passport` Driver，您只需在任何需要有效存取 Token 的路由上指定 `auth:api` Middleware：

```php
Route::get('/user', function () {
    // Only API authenticated users may access this route...
})->middleware('auth:api');
```

> [!WARNING]
> 如果您使用 [Client 憑證授權](#client-credentials-grant)，您應該使用 [ `Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` Middleware](#client-credentials-grant) 來保護您的路由，而不是 `auth:api` Middleware。

<a name="multiple-authentication-guards"></a>
#### 多個驗證 Guard

如果您的應用程式驗證不同類型的使用者，他們可能使用完全不同的 Eloquent 模型，您可能需要為應用程式中的每個使用者 Provider 類型定義一個 Guard 設定。這允許您保護針對特定使用者 Provider 的請求。例如，給定 `config/auth.php` 設定檔中的以下 Guard 設定：

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

以下路由將利用 `api-customers` Guard (它使用 `customers` 使用者 Provider) 來驗證傳入請求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> [!NOTE]
> 有關將多個使用者 Provider 與 Passport 結合使用的更多資訊，請查閱[個人存取 Token 文件](#customizing-the-user-provider-for-pat)和[密碼授權文件](#customizing-the-user-provider)。

<a name="passing-the-access-token"></a>
### 傳遞存取 Token

呼叫受 Passport 保護的路由時，您的應用程式的 API 消費者應將其存取 Token 指定為請求的 `Authorization` 標頭中的 `Bearer` Token。例如，使用 `Http` Facade 時：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>
## Token Scope

Scope 允許您的 API Client 在請求授權以存取帳戶時請求一組特定的權限。例如，如果您正在建立一個電子商務應用程式，並非所有 API 消費者都需要下訂單的能力。相反，您可以允許消費者僅請求授權以存取訂單出貨狀態。換句話說，Scope 允許您的應用程式使用者限制第三方應用程式可以代表他們執行的操作。

<a name="defining-scopes"></a>
### 定義 Scope

您可以使用 `Passport::tokensCan` 方法在您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義您的 API 的 Scope。`tokensCan` 方法接受一個 Scope 名稱和 Scope 描述的陣列。Scope 描述可以是您想要的任何內容，並將顯示給使用者在授權批准畫面上：

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
### 預設 Scope

如果 Client 未請求任何特定 Scope，您可以設定您的 Passport 伺服器使用 `defaultScopes` 方法將預設 Scope 附加到 Token。通常，您應該從您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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
### 為 Token 指派 Scope

<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

使用授權碼授權請求存取 Token 時，消費者應將其所需的 Scope 指定為 `scope` 查詢字串參數。`scope` 參數應為以空格分隔的 Scope 列表：

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
#### 發行個人存取 Token 時

如果您使用 `App\Models\User` 模型的 `createToken` 方法發行個人存取 Token，您可以將所需的 Scope 陣列作為方法的第二個參數傳遞：

```php
$token = $user->createToken('My Token', ['orders:create'])->accessToken;
```

<a name="checking-scopes"></a>
### 檢查 Scope

Passport 包含兩個 Middleware，可用於驗證傳入請求是否已使用已授予給定 Scope 的 Token 進行驗證。

<a name="check-for-all-scopes"></a>
#### 檢查所有 Scope

`Laravel\Passport\Http\Middleware\CheckToken` Middleware 可以分配給路由，以驗證傳入請求的存取 Token 是否具有所有列出的 Scope：

```php
use Laravel\Passport\Http\Middleware\CheckToken;

Route::get('/orders', function () {
    // Access token has both "orders:read" and "orders:create" scopes...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);
```

<a name="check-for-any-scopes"></a>
#### 檢查任何 Scope

`Laravel\Passport\Http\Middleware\CheckTokenForAnyScope` Middleware 可以分配給路由，以驗證傳入請求的存取 Token 是否具有*至少一個*列出的 Scope：

```php
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

Route::get('/orders', function () {
    // Access token has either "orders:read" or "orders:create" scope...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```

<a name="checking-scopes-on-a-token-instance"></a>
#### 檢查 Token 實例上的 Scope

一旦存取 Token 驗證的請求進入您的應用程式，您仍然可以使用已驗證的 `App\Models\User` 實例上的 `tokenCan` 方法檢查 Token 是否具有給定 Scope：

```php
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('orders:create')) {
        // ...
    }
});
```

<a name="additional-scope-methods"></a>
#### 其他 Scope 方法

`scopeIds` 方法將回傳所有已定義的 ID/名稱的陣列：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法將回傳所有已定義的 Scope 作為 `Laravel\Passport\Scope` 實例的陣列：

```php
Passport::scopes();
```

`scopesFor` 方法將回傳與給定 ID/名稱匹配的 `Laravel\Passport\Scope` 實例的陣列：

```php
Passport::scopesFor(['user:read', 'orders:create']);
```

您可以使用 `hasScope` 方法判斷是否已定義給定 Scope：

```php
Passport::hasScope('orders:create');
```

<a name="spa-authentication"></a>
## SPA 驗證

在建立 API 時，能夠從您的 JavaScript 應用程式消費您自己的 API 非常有用。這種 API 開發方法允許您自己的應用程式消費您與世界共享的相同 API。相同的 API 可以由您的 Web 應用程式、行動應用程式、第三方應用程式以及您可能在各種套件管理器上發布的任何 SDK 消費。

通常，如果您想從您的 JavaScript 應用程式消費您的 API，您需要手動將存取 Token 發送到應用程式，並在每次請求時將其傳遞給您的應用程式。但是，Passport 包含一個可以為您處理此問題的 Middleware。您只需將 `CreateFreshApiToken` Middleware 附加到您的應用程式的 `bootstrap/app.php` 檔案中的 `web` Middleware 群組：

```php
use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        CreateFreshApiToken::class,
    ]);
})
```

> [!WARNING]
> 您應該確保 `CreateFreshApiToken` Middleware 是您的 Middleware 堆疊中列出的最後一個 Middleware。

此 Middleware 會將 `laravel_token` Cookie 附加到您的傳出回應。此 Cookie 包含一個加密的 JWT，Passport 將使用它來驗證來自您的 JavaScript 應用程式的 API 請求。JWT 的存活時間等於您的 `session.lifetime` 設定值。現在，由於瀏覽器會自動將 Cookie 與所有後續請求一起發送，您可以向您的應用程式的 API 發出請求，而無需明確傳遞存取 Token：

```js
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

<a name="customizing-the-cookie-name"></a>
#### 自訂 Cookie 名稱

如有需要，您可以使用 `Passport::cookie` 方法自訂 `laravel_token` Cookie 的名稱。通常，此方法應從您的應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

使用此驗證方法時，您需要確保請求中包含有效的 CSRF Token 標頭。骨架應用程式和所有入門套件中包含的預設 Laravel JavaScript 腳手架包含一個 [Axios](https://github.com/axios/axios) 實例，它將自動使用加密的 `XSRF-TOKEN` Cookie 值在同源請求上發送 `X-XSRF-TOKEN` 標頭。

> [!NOTE]
> 如果您選擇發送 `X-CSRF-TOKEN` 標頭而不是 `X-XSRF-TOKEN`，您將需要使用 `csrf_token()` 提供的未加密 Token。

<a name="events"></a>
## 事件

Passport 在發行存取 Token 和 Refresh Token 時會觸發事件。您可以[監聽這些事件](/docs/{{version}}/events)以清除或撤銷資料庫中的其他存取 Token：

<div class="overflow-auto">

| 事件名稱                                    |
| --------------------------------------------- |
| `Laravel\Passport\Events\AccessTokenCreated`  |
| `Laravel\Passport\Events\AccessTokenRevoked`  |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>

<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用於指定當前已驗證的使用者及其 Scope。`actingAs` 方法的第一個參數是使用者實例，第二個是應授予使用者 Token 的 Scope 陣列：

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

Passport 的 `actingAsClient` 方法可用於指定當前已驗證的 Client 及其 Scope。`actingAsClient` 方法的第一個參數是 Client 實例，第二個是應授予 Client Token 的 Scope 陣列：

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
