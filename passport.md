# Laravel Passport

- [簡介](#introduction)
    - [Passport 或 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [Token 生命周期](#token-lifetimes)
    - [覆寫預設 Model](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [授權碼授權](#authorization-code-grant)
    - [管理 Client](#managing-clients)
    - [請求 Token](#requesting-tokens)
    - [管理 Token](#managing-tokens)
    - [刷新 Token](#refreshing-tokens)
    - [撤銷 Token](#revoking-tokens)
    - [清除 Token](#purging-tokens)
- [帶 PKCE 的授權碼授權](#code-grant-pkce)
    - [建立 Client](#creating-a-auth-pkce-grant-client)
    - [請求 Token](#requesting-auth-pkce-grant-tokens)
- [裝置授權碼授權](#device-authorization-grant)
    - [建立裝置碼授權 Client](#creating-a-device-authorization-grant-client)
    - [請求 Token](#requesting-device-authorization-grant-tokens)
- [密碼授權](#password-grant)
    - [建立密碼授權 Client](#creating-a-password-grant-client)
    - [請求 Token](#requesting-password-grant-tokens)
    - [請求所有 Scopes](#requesting-all-scopes)
    - [自訂使用者 Provider](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱含授權](#implicit-grant)
- [Client 憑證授權](#client-credentials-grant)
- [個人存取 Token](#personal-access-tokens)
    - [建立個人存取 Client](#creating-a-personal-access-client)
    - [自訂使用者 Provider](#customizing-the-user-provider-for-pat)
    - [管理個人存取 Token](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過中介層](#via-middleware)
    - [傳遞存取 Token](#passing-the-access-token)
- [Token Scopes](#token-scopes)
    - [定義 Scopes](#defining-scopes)
    - [預設 Scope](#default-scope)
    - [指派 Scopes 給 Token](#assigning-scopes-to-tokens)
    - [檢查 Scopes](#checking-scopes)
- [SPA 身份驗證](#spa-authentication)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 在幾分鐘內即可為您的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 建立在由 Andy Millington 和 Simon Hamp 維護的 [League OAuth2 伺服器](https://github.com/thephpleague/oauth2-server) 之上。

> [!NOTE]
> 本文件假設您已熟悉 OAuth2。如果您對 OAuth2 一無所知，建議您在繼續之前先熟悉 OAuth2 的一般[術語](https://oauth2.thephpleague.com/terminology/)和功能。

<a name="passport-or-sanctum"></a>
### Passport 或 Sanctum？

在開始之前，您可能需要判斷您的應用程式更適合使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum)。如果您的應用程式絕對需要支援 OAuth2，那麼您應該使用 Laravel Passport。

然而，如果您正在嘗試驗證單頁應用程式、行動應用程式或發放 API Token，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；然而，它提供了更簡單的 API 身份驗證開發體驗。

<a name="installation"></a>
## 安裝

您可以透過 `install:api` Artisan 指令安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此指令將會發布並執行資料庫遷移，以建立您的應用程式儲存 OAuth2 Client 和存取 Token 所需的資料表。此指令還會建立產生安全存取 Token 所需的加密金鑰。

執行 `install:api` 指令後，將 `Laravel\Passport\HasApiTokens` Trait 和 `Laravel\Passport\Contracts\OAuthenticatable` 介面新增到您的 `App\Models\User` 模型。此 Trait 將為您的模型提供一些輔助方法，讓您能夠檢查已驗證使用者的 Token 和 Scopes：

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

最後，在您的應用程式的 `config/auth.php` 設定檔中，您應該定義一個 `api` 身份驗證守衛，並將 `driver` 選項設定為 `passport`。這會指示您的應用程式在驗證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

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

首次將 Passport 部署到您的應用程式伺服器時，您可能需要執行 `passport:keys` 指令。此指令會產生 Passport 產生存取 Token 所需的加密金鑰。產生的金鑰通常不保留在版本控制中：

```shell
php artisan passport:keys
```

如有必要，您可以定義載入 Passport 金鑰的路徑。您可以使用 `Passport::loadKeysFrom` 方法來實現此目的。通常，此方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
#### 從環境載入 Keys

或者，您可以使用 `vendor:publish` Artisan 指令發布 Passport 的設定檔：

```shell
php artisan vendor:publish --tag=passport-config
```

設定檔發布後，您可以透過將應用程式的加密金鑰定義為環境變數來載入它們：

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

升級到 Passport 的新主要版本時，請務必仔細查閱[升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。

<a name="configuration"></a>
## 設定

<a name="token-lifetimes"></a>
### Token 生命周期

預設情況下，Passport 會發放有效期為一年的長效存取 Token。如果您想設定更長或更短的 Token 生命周期，可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> Passport 資料庫資料表上的 `expires_at` 欄位是唯讀的，僅供顯示用途。發放 Token 時，Passport 會將過期資訊儲存在已簽章和加密的 Token 中。如果您需要使 Token 失效，您應該[撤銷它](#revoking-tokens)。

<a name="overriding-default-models"></a>
### 覆寫預設 Model

您可以透過定義自己的模型並擴充對應的 Passport 模型來自由擴充 Passport 內部使用的模型：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定義您的模型後，您可以透過 `Laravel\Passport\Passport` 類別指示 Passport 使用您的自訂模型。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 您的自訂模型：

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

有時您可能希望自訂 Passport 定義的路由。為此，您首先需要透過將 `Passport::ignoreRoutes` 新增到應用程式 `AppServiceProvider` 的 `register` 方法來忽略 Passport 註冊的路由：

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
## 授權碼授權

大多數開發者都熟悉透過授權碼來使用 OAuth2。當使用授權碼時，Client 應用程式會將使用者重新導向到您的伺服器，使用者可在伺服器上核准或拒絕向 Client 發行存取 Token 的請求。

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

Passport 將自動定義 `/oauth/authorize` 路由來回傳此視圖。您的 `auth.oauth.authorize` 範本應包含一個向 `passport.authorizations.approve` 路由發送 POST 請求以核准授權的表單，以及一個向 `passport.authorizations.deny` 路由發送 DELETE 請求以拒絕授權的表單。`passport.authorizations.approve` 和 `passport.authorizations.deny` 路由預期 `state`、`client_id` 和 `auth_token` 欄位。

<a name="managing-clients"></a>
### 管理 Client

建立需要與您應用程式的 API 互動的應用程式的開發者，將需要透過建立一個「Client」來向您的應用程式註冊其應用程式。通常，這包括提供應用程式的名稱以及一個 URI，您的應用程式在使用者核准其授權請求後可將其重新導向至該 URI。

<a name="managing-first-party-clients"></a>
#### 第一方 Client

建立 Client 最簡單的方式是使用 `passport:client` Artisan 指令。此指令可用於建立第一方 Client 或測試您的 OAuth2 功能。當您執行 `passport:client` 指令時，Passport 會提示您輸入更多關於 Client 的資訊，並提供您 Client ID 和 Secret：

```shell
php artisan passport:client
```

如果您想為您的 Client 允許多個重新導向 URI，您可以在 `passport:client` 指令提示輸入 URI 時，使用逗號分隔的列表來指定它們。任何包含逗號的 URI 都應進行 URI 編碼：

```shell
https://third-party-app.com/callback,https://example.com/oauth/redirect
```

<a name="managing-third-party-clients"></a>
#### 第三方 Client

由於您的應用程式使用者將無法利用 `passport:client` 指令，您可以使用 `Laravel\Passport\ClientRepository` 類別的 `createAuthorizationCodeGrantClient` 方法為指定使用者註冊 Client：

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

`createAuthorizationCodeGrantClient` 方法會回傳 `Laravel\Passport\Client` 的實例。您可以將 `$client->id` 作為 Client ID，並將 `$client->plainSecret` 作為 Client Secret 顯示給使用者。

<a name="requesting-tokens"></a>
### 請求 Token

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重新導向進行授權

Client 建立後，開發者可以使用其 Client ID 和 secret 向您的應用程式請求授權碼與存取 Token。首先，消耗端應用程式應對您應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

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

`prompt` 參數可用來指定 Passport 應用程式的身份驗證行為。

若 `prompt` 值為 `none`，若使用者尚未透過 Passport 應用程式進行身份驗證，Passport 將始終拋出身份驗證錯誤。若值為 `consent`，Passport 將始終顯示授權核准畫面，即使所有 Scopes 先前已授予消耗端應用程式。若值為 `login`，Passport 應用程式將始終提示使用者重新登入應用程式，即使他們已有現有 Session。

若未提供 `prompt` 值，則僅當使用者先前未授權消耗端應用程式存取請求的 Scopes 時，才會提示使用者進行授權。

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您不需要手動定義此路由。

<a name="approving-the-request"></a>
#### 核准請求

當收到授權請求時，Passport 將根據 `prompt` 參數的值 (若存在) 自動回應，並可能會向使用者顯示一個模板，允許他們核准或拒絕授權請求。若他們核准請求，他們將被重新導向回消耗端應用程式指定的 `redirect_uri`。`redirect_uri` 必須與建立 Client 時指定的 `redirect` URL 相符。

有時您可能希望跳過授權提示，例如在授權第一方 Client 時。您可以透過[擴展 `Client` Model](#overriding-default-models) 並定義 `skipsAuthorization` 方法來實現此目的。如果 `skipsAuthorization` 返回 `true`，Client 將被核准，使用者將立即被重新導向回 `redirect_uri`，除非消耗端應用程式在重新導向進行授權時明確設定了 `prompt` 參數：

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

如果使用者核准了授權請求，他們將被重新導向回消耗端應用程式。消耗端應首先根據重新導向之前儲存的值驗證 `state` 參數。如果 state 參數匹配，則消耗端應向您的應用程式發出 `POST` 請求以請求存取 Token。該請求應包含您的應用程式在使用者核准授權請求時發出的授權碼：

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

此 `/oauth/token` 路由將返回一個包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取 Token 到期前的秒數。

> [!NOTE]
> 與 `/oauth/authorize` 路由一樣，`/oauth/token` 路由已由 Passport 為您定義。無需手動定義此路由。

<a name="managing-tokens"></a>
### 管理 Token

您可以使用 `Laravel\Passport\HasApiTokens` trait 的 `tokens` 方法來檢索使用者已授權的 Token。例如，這可用於為您的使用者提供一個儀表板，以追蹤他們與第三方應用程式的連接：

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

如果您的應用程式發行短生命週期的存取 Token，使用者將需要透過發行存取 Token 時提供給他們的 refresh token 來刷新他們的存取 Token：

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

此 `/oauth/token` 路由將返回一個包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取 Token 到期前的秒數。

<a name="revoking-tokens"></a>
### 撤銷 Token

您可以使用 `Laravel\Passport\Token` Model 上的 `revoke` 方法來撤銷 Token。您可以使用 `Laravel\Passport\RefreshToken` Model 上的 `revoke` 方法來撤銷 Token 的 refresh token：

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

當 Token 已被撤銷或過期時，您可能想要將其從資料庫中清除。Passport 內建的 `passport:purge` Artisan 指令可以為您完成此項工作：

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

您也可以在應用程式的 `routes/console.php` 檔案中設定 [排程作業](/docs/{{version}}/scheduling)，以定期自動清除您的 Token：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

<a name="code-grant-pkce"></a>
## 帶 PKCE 的授權碼授權

帶有 PKCE (Proof Key for Code Exchange) 的授權碼授權是驗證單頁應用程式或行動應用程式以存取你的 API 的一種安全方式。當你無法保證 client secret 能被保密儲存，或者為了減輕授權碼被攻擊者攔截的威脅時，就應該使用此授權類型。在將授權碼換取存取 token 時，「code verifier」與「code challenge」的組合將取代 client secret。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立 Client

在你的應用程式可以透過帶 PKCE 的授權碼授權來發行 token 之前，你需要建立一個啟用 PKCE 的 client。你可以使用帶有 `--public` 選項的 `passport:client` Artisan 指令來執行此操作：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求 Token

<a name="code-verifier-code-challenge"></a>
#### Code Verifier 與 Code Challenge

由於此授權授權不提供 client secret，開發者將需要產生 code verifier 與 code challenge 的組合，以便請求 token。

code verifier 應為介於 43 到 128 個字元的隨機字串，包含字母、數字以及 `"-"`、`"."`、`"_"`、`"~"` 字元，如 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636)中所定義。

code challenge 應為 Base64 編碼字串，且包含 URL 和檔名安全的字元。尾部的 `'='` 字元應被移除，且不應包含換行符、空白字元或其他額外字元。

```php
$encoded = base64_encode(hash('sha256', $codeVerifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

一旦 client 建立完成，你可以使用 client ID 以及產生的 code verifier 與 code challenge 來向你的應用程式請求授權碼和存取 token。首先，消費應用程式應向你的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

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

如果使用者核准授權請求，他們將被重新導向回消費應用程式。消費方應驗證 `state` 參數與重新導向前儲存的值是否一致，如同標準授權碼授權一樣。

如果 state 參數匹配，消費方應向你的應用程式發出 `POST` 請求以請求存取 token。該請求應包含你的應用程式在使用者核准授權請求時發出的授權碼，以及最初產生的 code verifier：

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
## 裝置授權碼授權

OAuth2 裝置授權碼授權允許無瀏覽器或輸入能力有限的裝置，例如電視與遊戲主機，透過交換「裝置碼」來取得存取 Token。使用裝置流程時，裝置 Client 會指示使用者使用次要裝置（例如電腦或智慧型手機）連線至您的伺服器，然後輸入提供的「使用者碼」並批准或拒絕存取請求。

首先，我們需要指示 Passport 如何回傳我們的「使用者碼」與「授權」視圖。

所有授權視圖的渲染邏輯都可以使用 `Laravel\Passport\Passport` 類別中適用的方法來自訂。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

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

Passport 會自動定義回傳這些視圖的路由。您的 `auth.oauth.device.user-code` 模板應包含一個表單，該表單向 `passport.device.authorizations.authorize` 路由發出 GET 請求。`passport.device.authorizations.authorize` 路由期望一個 `user_code` 查詢參數。

您的 `auth.oauth.device.authorize` 模板應包含一個表單，該表單向 `passport.device.authorizations.approve` 路由發出 POST 請求以批准授權，以及一個表單，該表單向 `passport.device.authorizations.deny` 路由發出 DELETE 請求以拒絕授權。`passport.device.authorizations.approve` 與 `passport.device.authorizations.deny` 路由期望 `state`、`client_id` 與 `auth_token` 欄位。

<a name="creating-a-device-authorization-grant-client"></a>
### 建立裝置碼授權 Client

在您的應用程式可以透過裝置授權碼授權發行 Token 之前，您需要建立一個已啟用裝置流程的 Client。您可以使用 `passport:client` Artisan 指令並帶上 `--device` 選項來執行此操作。此指令將建立一個第一方已啟用裝置流程的 Client，並提供您 Client ID 與密鑰：

```shell
php artisan passport:client --device
```

此外，您可以使用 `ClientRepository` 類別上的 `createDeviceAuthorizationGrantClient` 方法來註冊一個屬於指定使用者的第三方 Client：

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

Client 建立後，開發者即可使用其 Client ID 從您的應用程式請求裝置碼。首先，消費端裝置應向您應用程式的 `/oauth/device/code` 路由發出 `POST` 請求以請求裝置碼：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

這將回傳一個 JSON 回應，其中包含 `device_code`、`user_code`、`verification_uri`、`interval` 與 `expires_in` 屬性。`expires_in` 屬性包含裝置碼過期前的秒數。`interval` 屬性包含消費端裝置在輪詢 `/oauth/token` 路由時，應在請求之間等待的秒數，以避免速率限制錯誤。

> [!NOTE]
> 請記住，`/oauth/device/code` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="user-code"></a>
#### 顯示驗證 URI 與使用者碼

一旦裝置碼請求已取得，消費端裝置應指示使用者使用另一個裝置並造訪提供的 `verification_uri`，然後輸入 `user_code` 以批准授權請求。

<a name="polling-token-request"></a>
#### 輪詢 Token 請求

由於使用者將使用單獨的裝置來授予（或拒絕）存取，消費端裝置應輪詢您應用程式的 `/oauth/token` 路由以判斷使用者何時回應請求。消費端裝置應使用 JSON 回應中提供的最小輪詢 `interval`，在請求裝置碼時以避免速率限制錯誤：

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

如果使用者已批准授權請求，這將回傳一個 JSON 回應，其中包含 `access_token`、`refresh_token` 與 `expires_in` 屬性。`expires_in` 屬性包含存取 Token 過期前的秒數。

<a name="password-grant"></a>
## 密碼授權

> [!WARNING]
> We no longer recommend using password grant tokens. Instead, you should choose [a grant type that is currently recommended by OAuth2 Server](https://oauth2.thephpleague.com/authorization-server/which-grant/).

OAuth2 密碼授權 (password grant) 允許您的其他第一方 Client (例如行動應用程式) 使用電子郵件地址/使用者名稱和密碼來取得存取 Token。這使您能夠安全地向您的第一方 Client 頒發存取 Token，而無需您的使用者經歷完整的 OAuth2 授權碼重新導向流程。

若要啟用密碼授權，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

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

在您的應用程式能夠透過密碼授權頒發 Token 之前，您需要建立一個密碼授權 Client。您可以使用帶有 `--password` 選項的 `passport:client` Artisan 指令來執行此操作。

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>
### 請求 Token

一旦您啟用了授權並建立了一個密碼授權 Client，您就可以透過向 `/oauth/token` 路由發送一個 `POST` 請求，並附帶使用者的電子郵件地址和密碼來請求存取 Token。請記住，此路由已由 Passport 註冊，因此無需手動定義。如果請求成功，您將從伺服器收到的 JSON 回應中獲得 `access_token` 和 `refresh_token`：

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
> 請記住，存取 Token 預設是長期有效的。但是，您可以在需要時自由地[設定您的最大存取 Token 生命周期](#configuration)。

<a name="requesting-all-scopes"></a>
### 請求所有 Scopes

當使用密碼授權 (password grant) 或 Client 憑證授權 (client credentials grant) 時，您可能希望授權 Token 擁有您的應用程式支援的所有 Scopes。您可以透過請求 `*` Scope 來實現這一點。如果您請求 `*` Scope，Token 實例上的 `can` 方法將始終返回 `true`。此 Scope 只能分配給使用 `password` 或 `client_credentials` 授權發出的 Token：

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
### 自訂使用者 Provider

如果您的應用程式使用多個[身份驗證使用者 Provider](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --password` 指令建立 Client 時，提供 `--provider` 選項來指定密碼授權 Client 使用哪個使用者 Provider。給定的 Provider 名稱應與您的應用程式 `config/auth.php` 設定檔中定義的有效 Provider 相符。然後，您可以[使用中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自該守衛指定 Provider 的使用者才能被授權。

<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

當使用密碼授權 (password grant) 進行身份驗證時，Passport 將使用您的可驗證 Model 的 `email` 屬性作為「使用者名稱」。但是，您可以透過在 Model 上定義 `findForPassport` 方法來自訂此行為：

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

當使用密碼授權 (password grant) 進行身份驗證時，Passport 將使用您的 Model 的 `password` 屬性來驗證給定的密碼。如果您的 Model 沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，您可以在您的 Model 上定義一個 `validateForPassportPasswordGrant` 方法：

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
## 隱含授權

> [!WARNING]
> We no longer recommend using implicit grant tokens. Instead, you should choose [a grant type that is currently recommended by OAuth2 Server](https://oauth2.thephpleague.com/authorization-server/which-grant/).

隱含授權 (implicit grant) 與授權碼授權 (authorization code grant) 相似；但是，Token 會直接返回給 Client，而無需交換授權碼。此授權最常應用於 Client 憑證無法安全儲存的 JavaScript 或行動應用程式。若要啟用此授權，請在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

在您的應用程式能夠透過隱含授權頒發 Token 之前，您需要建立一個隱含授權 Client。您可以使用帶有 `--implicit` 選項的 `passport:client` Artisan 指令來執行此操作。

```shell
php artisan passport:client --implicit
```

一旦授權已啟用且隱含 Client 已建立，開發人員即可使用其 Client ID 向您的應用程式請求存取 Token。消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

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
## Client 憑證授權

Client 憑證授權適用於機器對機器驗證。例如，您可能會在一個排程任務中使用此授權類型，該任務透過 API 執行維護任務。

在您的應用程式可以透過 Client 憑證授權發行 Token 之前，您需要建立一個 Client 憑證授權 Client。您可以使用 `passport:client` Artisan 指令的 `--client` 選項來完成此操作：

```shell
php artisan passport:client --client
```

接下來，將 `Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層指派給路由：

```php
use Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner;

Route::get('/orders', function (Request $request) {
    // Access token is valid and the client is resource owner...
})->middleware(EnsureClientIsResourceOwner::class);
```

若要將路由的存取權限限制為特定 Scopes，您可以將所需 Scopes 列表提供給 `using` 方法：

```php
Route::get('/orders', function (Request $request) {
    // Access token is valid, the client is resource owner, and has both "servers:read" and "servers:create" scopes...
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```


<a name="retrieving-tokens"></a>
### 取得 Token

若要使用此授權類型取得 Token，請向 `oauth/token` 端點發出請求：

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

有時，您的使用者可能希望在不經過典型的授權碼重新導向流程的情況下，發行存取 Token 給自己。允許使用者透過應用程式的 UI 發行 Token 給自己，對於讓使用者實驗您的 API 會很有用，或者可以作為發行存取 Token 的更簡單方式。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 發行個人存取 Token，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 用於發行 API 存取 Token 的輕量級第一方函式庫。


<a name="creating-a-personal-access-client"></a>
### 建立個人存取 Client

在您的應用程式可以發行個人存取 Token 之前，您需要建立一個個人存取 Client。您可以透過執行 `passport:client` Artisan 指令並帶上 `--personal` 選項來完成此操作。如果您已執行 `passport:install` 指令，則無需再次執行此指令：

```shell
php artisan passport:client --personal
```


<a name="customizing-the-user-provider-for-pat"></a>
### 自訂使用者 Provider

如果您的應用程式使用多個[身份驗證使用者 Provider](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --personal` 指令建立 Client 時，透過提供 `--provider` 選項來指定個人存取授權 Client 使用哪個使用者 Provider。提供的 Provider 名稱應與您應用程式 `config/auth.php` 設定檔中定義的有效 Provider 相符。然後您可以[透過中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自該 Guard 指定 Provider 的使用者獲得授權。


<a name="managing-personal-access-tokens"></a>
### 管理個人存取 Token

一旦您建立了個人存取 Client，您可以使用 `App\Models\User` Model 實例上的 `createToken` 方法為指定使用者發行 Token。 `createToken` 方法將 Token 的名稱作為第一個參數，並將一個可選的 [scopes](#token-scopes) 陣列作為第二個參數：

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

Passport 包含一個[身份驗證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，它將驗證傳入請求上的存取 Token。一旦您將 `api` Guard 設定為使用 `passport` Driver，您只需在任何需要有效存取 Token 的路由上指定 `auth:api` 中介層：

```php
Route::get('/user', function () {
    // Only API authenticated users may access this route...
})->middleware('auth:api');
```

> [!WARNING]
> 如果您正在使用[Client 憑證授權](#client-credentials-grant)，您應該使用 [`Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層](#client-credentials-grant)來保護您的路由，而非 `auth:api` 中介層。


<a name="multiple-authentication-guards"></a>
#### 多個身份驗證 Guard

如果您的應用程式驗證不同類型的使用者，這些使用者可能使用完全不同的 Eloquent Model，您可能需要為應用程式中的每個使用者 Provider 類型定義一個 Guard 設定。這允許您保護專用於特定使用者 Provider 的請求。例如，考慮 `config/auth.php` 設定檔中的以下 Guard 設定：

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

以下路由將利用 `api-customers` Guard (使用 `customers` 使用者 Provider) 來驗證傳入的請求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> [!NOTE]
> 有關搭配 Passport 使用多個使用者 Provider 的更多資訊，請查閱[個人存取 Token 文件](#customizing-the-user-provider-for-pat)和[密碼授權文件](#customizing-the-user-provider)。


<a name="passing-the-access-token"></a>
### 傳遞存取 Token

當呼叫受 Passport 保護的路由時，應用程式的 API 消費者應在請求的 `Authorization` 標頭中將其存取 Token 指定為 `Bearer` Token。例如，當使用 `Http` Facade 時：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>
## Token Scopes

Scopes 允許您的 API 客戶端在請求存取帳戶的授權時，請求一組特定的權限。舉例來說，如果您正在建構一個電子商務應用程式，並非所有 API 消費者都需要下訂單的能力。相反地，您可以允許消費者僅請求存取訂單運送狀態的授權。換句話說，Scopes 讓您的應用程式使用者能夠限制第三方應用程式代表他們執行的動作。

<a name="defining-scopes"></a>
### 定義 Scopes

您可以在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中，使用 `Passport::tokensCan` 方法定義您的 API Scopes。`tokensCan` 方法接受一個包含 Scope 名稱與 Scope 描述的陣列。Scope 描述可以是您想要的任何內容，並會顯示在授權核准畫面上給使用者：

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

如果 Client 未請求任何特定 Scopes，您可以設定您的 Passport 伺服器，使用 `defaultScopes` 方法將預設 Scopes 附加到 Token。通常，您應在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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
### 指派 Scopes 給 Token

<a name="when-requesting-authorization-codes"></a>
#### When Requesting Authorization Codes

當使用授權碼授權請求 Access Token 時，消費者應將其所需的 Scopes 指定為 `scope` 查詢字串參數。`scope` 參數應為以空格分隔的 Scopes 列表：

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
#### When Issuing Personal Access Tokens

如果您使用 `App\Models\User` 模型上的 `createToken` 方法發行個人存取 Token，您可以將所需 Scopes 的陣列作為該方法的第二個參數傳遞：

```php
$token = $user->createToken('My Token', ['orders:create'])->accessToken;
```

<a name="checking-scopes"></a>
### 檢查 Scopes

Passport 包含兩個中介層，可用於驗證傳入請求是否已透過獲得指定 Scope 的 Token 進行身份驗證。

<a name="check-for-all-scopes"></a>
#### Check For All Scopes

`Laravel\Passport\Http\Middleware\CheckToken` 中介層可以指派給路由，以驗證傳入請求的 Access Token 具有所有列出的 Scopes：

```php
use Laravel\Passport\Http\Middleware\CheckToken;

Route::get('/orders', function () {
    // Access token has both "orders:read" and "orders:create" scopes...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);
```

<a name="check-for-any-scopes"></a>
#### Check for Any Scopes

`Laravel\Passport\Http\Middleware\CheckTokenForAnyScope` 中介層可以指派給路由，以驗證傳入請求的 Access Token 至少具有其中一個列出的 Scopes：

```php
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

Route::get('/orders', function () {
    // Access token has either "orders:read" or "orders:create" scope...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```

<a name="checking-scopes-on-a-token-instance"></a>
#### Checking Scopes on a Token Instance

一旦經過 Access Token 身份驗證的請求進入您的應用程式，您仍然可以使用已驗證的 `App\Models\User` 實例上的 `tokenCan` 方法檢查該 Token 是否具有指定的 Scope：

```php
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('orders:create')) {
        // ...
    }
});
```

<a name="additional-scope-methods"></a>
#### Additional Scope Methods

`scopeIds` 方法將返回所有已定義 ID / 名稱的陣列：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法將返回所有已定義的 Scopes 陣列，作為 `Laravel\Passport\Scope` 實例：

```php
Passport::scopes();
```

`scopesFor` 方法將返回與指定 ID / 名稱匹配的 `Laravel\Passport\Scope` 實例陣列：

```php
Passport::scopesFor(['user:read', 'orders:create']);
```

您可以使用 `hasScope` 方法判斷指定的 Scope 是否已被定義：

```php
Passport::hasScope('orders:create');
```

<a name="spa-authentication"></a>
## SPA 身份驗證

在建構 API 時，能夠從您的 JavaScript 應用程式中消費您自己的 API 是極其有用的。這種 API 開發方法允許您自己的應用程式消費與您分享給世界的相同 API。相同的 API 可由您的 Web 應用程式、行動應用程式、第三方應用程式以及您可能在各種套件管理工具上發布的任何 SDK 消費。

通常，如果您想從 JavaScript 應用程式中消費您的 API，您需要手動將 Access Token 傳送給應用程式，並在每次請求時傳遞它。然而，Passport 包含一個可以為您處理此事的中介層。您所需要做的就是將 `CreateFreshApiToken` 中介層附加到應用程式 `bootstrap/app.php` 檔案中的 `web` 中介層群組：

```php
use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        CreateFreshApiToken::class,
    ]);
})
```

> [!WARNING]
> 您應確保 `CreateFreshApiToken` 中介層是您的中介層堆疊中最後列出的中介層。

此中介層將在您的傳出回應中附加一個 `laravel_token` cookie。此 cookie 包含一個加密的 JWT，Passport 將使用它來驗證來自您的 JavaScript 應用程式的 API 請求。此 JWT 的生命週期等於您的 `session.lifetime` 設定值。現在，由於瀏覽器會自動隨所有後續請求傳送此 cookie，您可以向應用程式的 API 發出請求，而無需明確傳遞 Access Token：

```js
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

<a name="customizing-the-cookie-name"></a>
#### Customizing the Cookie Name

如果需要，您可以使用 `Passport::cookie` 方法自訂 `laravel_token` cookie 的名稱。通常，您應在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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
#### CSRF Protection

當使用這種身份驗證方法時，您需要確保請求中包含有效的 CSRF Token 標頭。骨架應用程式和所有入門套件中包含的預設 Laravel JavaScript 腳手架包含一個 [Axios](https://github.com/axios/axios) 實例，該實例會自動使用加密的 `XSRF-TOKEN` cookie 值，在同源請求中傳送 `X-XSRF-TOKEN` 標頭。

> [!NOTE]
> 如果您選擇傳送 `X-CSRF-TOKEN` 標頭而不是 `X-XSRF-TOKEN`，您將需要使用由 `csrf_token()` 提供、未加密的 Token。

<a name="events"></a>
## 事件

Passport 在發行存取 Token 和刷新 Token 時會觸發事件。您可以[監聽這些事件](/docs/{{version}}/events)來清除或撤銷資料庫中的其他存取 Token：

<div class="overflow-auto">

| 事件名稱                                    |
| --------------------------------------------- |
| `Laravel\Passport\Events\AccessTokenCreated`  |
| `Laravel\Passport\Events\AccessTokenRevoked`  |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>


<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用來指定目前通過身份驗證的使用者及其 Scopes。傳給 `actingAs` 方法的第一個參數是使用者實例，第二個是應授予使用者 Token 的 Scopes 陣列：

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

Passport 的 `actingAsClient` 方法可用來指定目前通過身份驗證的 Client 及其 Scopes。傳給 `actingAsClient` 方法的第一個參數是 Client 實例，第二個是應授予 Client Token 的 Scopes 陣列：

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