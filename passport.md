# Laravel Passport

- [簡介](#introduction)
    - [Passport 還是 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [Token 生命週期](#token-lifetimes)
    - [覆寫預設 Model](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [授權碼模式 (Authorization Code Grant)](#authorization-code-grant)
    - [管理 Client](#managing-clients)
    - [請求 Token](#requesting-tokens)
    - [管理 Token](#managing-tokens)
    - [刷新 Token](#refreshing-tokens)
    - [撤銷 Token](#revoking-tokens)
    - [清除 Token](#purging-tokens)
- [帶有 PKCE 的授權碼模式](#code-grant-pkce)
    - [建立 Client](#creating-a-auth-pkce-grant-client)
    - [請求 Token](#requesting-auth-pkce-grant-tokens)
- [裝置授權模式 (Device Authorization Grant)](#device-authorization-grant)
    - [建立裝置授權模式 Client](#creating-a-device-authorization-grant-client)
    - [請求 Token](#requesting-device-authorization-grant-tokens)
- [密碼模式 (Password Grant)](#password-grant)
    - [建立密碼模式 Client](#creating-a-password-grant-client)
    - [請求 Token](#requesting-password-grant-tokens)
    - [請求所有 Scope](#requesting-all-scopes)
    - [自訂使用者提供者](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱式模式 (Implicit Grant)](#implicit-grant)
- [Client 憑證模式 (Client Credentials Grant)](#client-credentials-grant)
- [個人存取 Token (Personal Access Tokens)](#personal-access-tokens)
    - [建立個人存取 Client](#creating-a-personal-access-client)
    - [自訂使用者提供者](#customizing-the-user-provider-for-pat)
    - [管理個人存取 Token](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過中介層](#via-middleware)
    - [傳送 Access Token](#passing-the-access-token)
- [Token 範圍 (Token Scopes)](#token-scopes)
    - [定義 Scope](#defining-scopes)
    - [預設 Scope](#default-scope)
    - [指派 Scope 給 Token](#assigning-scopes-to-tokens)
    - [檢查 Scope](#checking-scopes)
- [SPA 認證](#spa-authentication)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 能在幾分鐘內為你的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 是建構於由 Andy Millington 與 Simon Hamp 維護的 [League OAuth2 伺服器](https://github.com/thephpleague/oauth2-server) 之上。

> [!NOTE]
> 本文件假設你已經熟悉 OAuth2。如果你對 OAuth2 完全不了解，在繼續之前請考慮先熟悉 OAuth2 的一般[專有名詞](https://oauth2.thephpleague.com/terminology/)與功能特色。


<a name="passport-or-sanctum"></a>
### Passport 還是 Sanctum？

在開始之前，你可能需要評估你的應用程式使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum) 會更合適。如果你的應用程式絕對需要支援 OAuth2，那麼你應該使用 Laravel Passport。

然而，如果你嘗試要對單頁應用程式（SPA）、行動應用程式進行認證，或是發行 API Token，你應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；然而，它提供了簡單許多的 API 認證開發體驗。


<a name="installation"></a>
## 安裝

你可以透過 `install:api` Artisan 指令安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此指令會發布並執行建立儲存 OAuth2 Client 與 Access Token 資料表所需的資料庫遷移。該指令還會建立產生安全 Access Token 所需的加密金鑰。

執行 `install:api` 指令後，將 `Laravel\Passport\HasApiTokens` Trait 以及 `Laravel\Passport\Contracts\OAuthenticatable` 介面新增至你的 `App\Models\User` Model 中。這個 Trait 將為你的 Model 提供一些輔助方法，讓你能夠檢視通過認證之使用者的 Token 與 Scope：

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

最後，在應用程式的 `config/auth.php` 設定檔中，你應該定義一個 `api` 認證 Guard，並將 `driver` 選項設定為 `passport`。這會指示你的應用程式在認證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

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

當首次將 Passport 部署至你的應用程式伺服器時，你可能需要執行 `passport:keys` 指令。此指令會產生 Passport 產生 Access Token 所需的加密金鑰。產生的金鑰通常不應包含在版本控制中：

```shell
php artisan passport:keys
```

如有需要，你可以定義載入 Passport 金鑰的路徑。你可以使用 `Passport::loadKeysFrom` 方法來達成此目的。通常，此方法應在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法內呼叫：

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

或者，你可以使用 `vendor:publish` Artisan 指令發布 Passport 的設定檔：

```shell
php artisan vendor:publish --tag=passport-config
```

設定檔發布後，你可以透過在環境變數中定義應用程式的加密金鑰來進行載入：

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

當升級至 Passport 的全新主要（Major）版本時，請務必仔細閱讀[升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。


<a name="configuration"></a>
## 設定


<a name="token-lifetimes"></a>
### Token 生命週期

預設情況下，Passport 會發行有效期為一年的長效 Access Token。如果你希望設定更長或更短的 Token 生命週期，可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 與 `personalAccessTokensExpireIn` 方法。這些方法應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> Passport 資料庫表中的 `expires_at` 欄位為唯讀，且僅供顯示使用。發行 Token 時，Passport 會將過期資訊儲存在經簽章與加密的 Token 內部。如果你需要讓 Token 失效，你應該[撤銷它](#revoking-tokens)。


<a name="overriding-default-models"></a>
### 覆寫預設 Model

你可以透過定義自己的 Model 並繼承對應的 Passport Model 來自由地擴充 Passport 內部使用的 Model：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定義好 Model 後，你可以透過 `Laravel\Passport\Passport` 類別來指示 Passport 使用你的自訂 Model。通常，你應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 關於你的自訂 Model：

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

有時你可能會想要自訂 Passport 所定義的路由。要達到這個目的，首先你需要透過在應用程式的 `AppServiceProvider` 的 `register` 方法中加入 `Passport::ignoreRoutes` 來忽略 Passport 註冊的路由：

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

接著，你可以將 Passport 在[其路由檔](https://github.com/laravel/passport/blob/master/routes/web.php)中定義的路由複製到應用程式的 `routes/web.php` 檔案中，並根據你的喜好進行修改：

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
## 授權碼模式 (Authorization Code Grant)

透過授權碼 (Authorization codes) 使用 OAuth2 是大多數開發人員最熟悉的方式。使用授權碼時，Client 應用程式會將使用者重新導向至您的伺服器，使用者可以在該處批准或拒絕向 Client 發行 access token 的請求。

首先，我們需要告知 Passport 如何回傳我們的「authorization」視圖。

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

Passport 會自動定義回傳此視圖的 `/oauth/authorize` 路由。您的 `auth.oauth.authorize` 模板應該包含一個向 `passport.authorizations.approve` 路由發送 POST 請求以批准授權的表單，以及一個向 `passport.authorizations.deny` 路由發送 DELETE 請求以拒絕授權的表單。`passport.authorizations.approve` 與 `passport.authorizations.deny` 路由需要 `state`、`client_id` 以及 `auth_token` 欄位。


<a name="managing-clients"></a>
### 管理 Client

開發需要與您的應用程式 API 互動的應用程式之開發人員，必須透過建立一個「Client」來向您的應用程式註冊其應用程式。通常，這包括提供其應用程式的名稱以及一個 URI，以便在使用者批准其授權請求後，您的應用程式可以重新導向至該 URI。


<a name="managing-first-party-clients"></a>
#### 第一方 Client

建立 Client 最簡單的方式是使用 `passport:client` Artisan 指令。此指令可用於建立第一方 Client 或測試您的 OAuth2 功能。當您執行 `passport:client` 指令時，Passport 會提示您輸入更多關於 Client 的資訊，並為您提供 Client ID 與 Secret：

```shell
php artisan passport:client
```

如果您想允許 Client 使用多個重新導向 URI，可以在 `passport:client` 指令提示輸入 URI 時，使用逗號分隔的列表來指定它們。任何包含逗號的 URI 都應該進行 URI 編碼：

```shell
https://third-party-app.com/callback,https://example.com/oauth/redirect
```


<a name="managing-third-party-clients"></a>
#### 第三方 Client

由於您的應用程式使用者無法使用 `passport:client` 指令，您可以透過 `Laravel\Passport\ClientRepository` 類別的 `createAuthorizationCodeGrantClient` 方法來為指定的使用者註冊 Client：

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

`createAuthorizationCodeGrantClient` 方法會回傳一個 `Laravel\Passport\Client` 的實例。您可以將 `$client->id` 作為 Client ID，並將 `$client->plainSecret` 作為 Client Secret 顯示給使用者。

<a name="requesting-tokens"></a>
### 請求 Token

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 轉址以進行授權

建立 Client 後，開發人員可以使用其 Client ID 與 Secret 向您的應用程式請求授權碼 (Authorization Code) 與 Access Token。首先，使用端應用程式應發送轉址請求至您應用程式的 `/oauth/authorize` 路由，如下所示：

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

`prompt` 參數可用於指定 Passport 應用程式的認證行為。

若 `prompt` 值為 `none`，當使用者尚未在 Passport 應用程式中通過認證時，Passport 將一律拋出認證錯誤。若值為 `consent`，即使先前已將所有 Scope 授權給該使用端應用程式，Passport 仍會一律顯示授權同意畫面。當值為 `login` 時，Passport 應用程式將一律提示使用者重新登入應用程式，即使他們已有現成的 Session 也是如此。

若未提供 `prompt` 值，則僅在使用者先前未就請求的 Scope 授權存取該使用端應用程式時，才會提示使用者進行授權。

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 事先定義。您不需要手動定義此路由。

<a name="approving-the-request"></a>
#### 同意請求

收到授權請求時，Passport 會根據 `prompt` 參數的值（如果存在）自動回應，並可能向使用者顯示一個頁面範本，允許他們同意或拒絕授權請求。如果他們同意請求，將被轉址回使用端應用程式所指定的 `redirect_uri`。`redirect_uri` 必須與建立 Client 時所指定的 `redirect` URL 相符。

有時您可能希望跳過授權提示，例如在授權第一方 (first-party) Client 時。您可以透過[擴充 `Client` Model](#overriding-default-models) 並定義 `skipsAuthorization` 方法來達成此目的。如果 `skipsAuthorization` 回傳 `true`，該 Client 將會自動被核准，且使用者會立即被轉址回 `redirect_uri`，除非使用端應用程式在轉址進行授權時顯式設定了 `prompt` 參數：

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
#### 將授權碼轉換為 Access Token

如果使用者同意授權請求，他們將會被轉址回使用端應用程式。使用端應首先比對 `state` 參數與轉址前儲存的值。若 State 參數吻合，使用端應向您的應用程式發送 `POST` 請求以請求 Access Token。該請求應包含使用者同意授權請求時由您應用程式發出的授權碼：

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

此 `/oauth/token` 路由將回傳包含 `access_token`、`refresh_token` 與 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含 Access Token 過期前剩餘的秒數。

> [!NOTE]
> 就像 `/oauth/authorize` 路由一樣，`/oauth/token` 路由也已由 Passport 為您定義好了。不需要手動定義此路由。

<a name="managing-tokens"></a>
### 管理 Token

您可以透過 `Laravel\Passport\HasApiTokens` Trait 的 `tokens` 方法來取得使用者已授權的 Token。例如，這可用於向您的使用者提供控制面板，以追蹤他們與第三方應用程式的連線：

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

若您的應用程式發行短效期的 Access Token，使用者將需要透過發行 Access Token 時所提供的 Refresh Token 來刷新其 Access Token：

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

此 `/oauth/token` 路由將回傳包含 `access_token`、`refresh_token` 與 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含 Access Token 過期前剩餘的秒數。

<a name="revoking-tokens"></a>
### 撤銷 Token

您可以使用 `Laravel\Passport\Token` Model 上的 `revoke` 方法來撤銷 Token。您可以使用 `Laravel\Passport\RefreshToken` Model 上的 `revoke` 方法來撤銷 Token 的 Refresh Token：

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

當 Token 已被撤銷或過期時，您可能希望將它們從資料庫中清除。Passport 隨附的 `passport:purge` Artisan 指令可以為您做到這點：

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

您也可以在應用程式的 `routes/console.php` 檔案中設定[排程任務](/docs/{{version}}/scheduling)，以定期自動清理 Token：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

<a name="code-grant-pkce"></a>
## 帶有 PKCE 的授權碼模式

帶有「證明金鑰程式碼交換 (Proof Key for Code Exchange, PKCE)」的授權碼模式，是一種為單頁應用程式 (SPA) 或行動應用程式進行認證以存取 API 的安全方式。當您無法保證 Client Secret 能被保密儲存，或者為了降低授權碼被攻擊者攔截的風險時，就應該使用此授權模式。在將授權碼交換為 Access Token 時，會結合「code verifier」與「code challenge」來取代 Client Secret。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立 Client

在您的應用程式可以透過帶有 PKCE 的授權碼模式核發 Token 之前，您需要建立一個啟用 PKCE 的 Client。您可以透過帶有 `--public` 選項的 `passport:client` Artisan 指令來完成：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求 Token

<a name="code-verifier-code-challenge"></a>
#### Code Verifier 與 Code Challenge

由於此授權模式不提供 Client Secret，開發人員需要產生 code verifier 與 code challenge 的組合才能請求 Token。

如同 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636) 中所定義，code verifier 應該是一個介於 43 到 128 個字元之間的隨機字串，包含英文字母、數字以及 `"-"`、`"."`、`"_"`、`"~"` 等字元。

code challenge 應該是一個經過 Base64 編碼且符合 URL 與檔名安全的字串。末尾的 `'='` 字元必須被移除，且不得包含換行符號、空白字元或其他額外字元。

```php
$encoded = base64_encode(hash('sha256', $codeVerifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重定向以進行授權

建立 Client 後，您可以使用 Client ID 以及產生的 code verifier 和 code challenge 向您的應用程式請求授權碼與 Access Token。首先，消費端應用程式應該發送重定向請求至您應用程式的 `/oauth/authorize` 路由：

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
#### 將授權碼轉換為 Access Token

如果使用者同意授權請求，他們將被重定向回消費端應用程式。如同標準授權碼模式一樣，消費端應該根據重定向前儲存的值來驗證 `state` 參數。

如果 state 參數相符，消費端應該向您的應用程式發送 `POST` 請求以請求 Access Token。該請求應包含使用者同意授權請求時由您應用程式核發的授權碼，以及最初產生的 code verifier：

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
## 裝置授權模式 (Device Authorization Grant)

OAuth2 裝置授權模式允許無瀏覽器或輸入有限的裝置（例如電視和遊戲主機）透過交換 "device code" 來取得 access token。使用裝置流程時，裝置 Client 會引導使用者使用次要裝置（例如電腦或智慧型手機）連線至您的伺服器，並在伺服器上輸入提供的 "user code"，以批准或拒絕該存取請求。

首先，我們需要指示 Passport 如何回傳我們的 "user code" 與 "authorization" 檢視。

所有授權檢視的渲染邏輯都可以透過 `Laravel\Passport\Passport` 類別中提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法內呼叫此方法：

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

Passport 會自動定義回傳這些檢視的路由。您的 `auth.oauth.device.user-code` 樣板應包含一個表單，向 `passport.device.authorizations.authorize` 路由發送 GET 請求。`passport.device.authorizations.authorize` 路由需要一個 `user_code` 查詢參數。

您的 `auth.oauth.device.authorize` 樣板應包含一個向 `passport.device.authorizations.approve` 路由發送 POST 請求以批准授權的表單，以及一個向 `passport.device.authorizations.deny` 路由發送 DELETE 請求以拒絕授權的表單。`passport.device.authorizations.approve` 與 `passport.device.authorizations.deny` 路由需要 `state`、`client_id` 以及 `auth_token` 欄位。


<a name="creating-a-device-authorization-grant-client"></a>
### 建立裝置授權模式 Client

在您的應用程式可以透過裝置授權模式核發 Token 之前，您需要建立一個已啟用裝置流程的 Client。您可以透過帶有 `--device` 選項的 `passport:client` Artisan 指令來完成此操作。此指令將建立一個第一方且已啟用裝置流程的 Client，並為您提供 Client ID 與 Secret：

```shell
php artisan passport:client --device
```

此外，您可以使用 `ClientRepository` 類別上的 `createDeviceAuthorizationGrantClient` 方法來註冊屬於指定使用者的第三方 Client：

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
#### 請求 Device Code

建立 Client 後，開發者可以使用其 Client ID 向您的應用程式請求 Device Code。首先，發起請求的裝置應向您應用程式的 `/oauth/device/code` 路由發送 `POST` 請求以取得 Device Code：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

這將回傳包含 `device_code`、`user_code`、`verification_uri`、`interval` 以及 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含 Device Code 過期前的秒數。`interval` 屬性包含發起請求的裝置輪詢 `/oauth/token` 路由時，每次請求之間應等待的秒數，以避免觸發速率限制錯誤。

> [!NOTE]
> 請記住，`/oauth/device/code` 路由已由 Passport 定義好。您不需要手動定義此路由。


<a name="user-code"></a>
#### 顯示 Verification URI 與 User Code

取得 Device Code 請求後，發起請求的裝置應指示使用者使用另一個裝置，並前往提供的 `verification_uri` 輸入 `user_code`，以批准授權請求。


<a name="polling-token-request"></a>
#### 輪詢 Token 請求

由於使用者將使用另一個獨立的裝置來同意（或拒絕）存取權限，因此發起請求的裝置應輪詢您的應用程式 `/oauth/token` 路由，以確認使用者何時回應了該請求。發起請求的裝置在請求 Device Code 時，應使用 JSON 回應中提供的最小輪詢 `interval` 秒數，以避免觸發速率限制錯誤：

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

如果使用者已批准授權請求，這將回傳包含 `access_token`、`refresh_token` 以及 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含 access token 過期前的秒數。

<a name="password-grant"></a>
## 密碼模式 (Password Grant)

> [!WARNING]
> 我們不再建議使用密碼模式 Token。相反地，您應該選擇 [OAuth2 Server 目前推薦的授權模式](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密碼模式允許您的其他第一方 Client（例如行動應用程式）使用 Email 地址 / 使用者名稱和密碼來取得 Access Token。這能讓您安全地向第一方 Client 發行 Access Token，而不需要使用者經歷整個 OAuth2 授權碼重導向流程。

若要啟用密碼模式，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

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
### 建立密碼模式 Client

在您的應用程式可以透過密碼模式發行 Token 之前，您需要建立一個密碼模式 Client。您可以使用帶有 `--password` 選項的 `passport:client` Artisan 指令來做到這一點。

```shell
php artisan passport:client --password
```


<a name="requesting-password-grant-tokens"></a>
### 請求 Token

當您啟用了該模式並建立了一個密碼模式 Client 後，您可以透過向 `/oauth/token` 路由發送一個帶有使用者 Email 地址和密碼的 `POST` 請求來請求 Access Token。請記住，此路由已經由 Passport 註冊，因此不需要手動定義它。如果請求成功，您將在伺服器的 JSON 回應中收到 `access_token` 和 `refresh_token`：

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
> 請記住，預設情況下 Access Token 是長效型的。不過，如果有需要，您可以自由地[設定 Access Token 的最大生命週期](#configuration)。


<a name="requesting-all-scopes"></a>
### 請求所有 Scope

當使用密碼模式或 Client 憑證模式時，您可能希望授權 Token 擁有您的應用程式支援的所有 Scope。您可以透過請求 `*` Scope 來做到這一點。如果您請求 `*` Scope，Token 實例上的 `can` 方法將永遠返回 `true`。此 Scope 只能指派給使用 `password` 或 `client_credentials` 模式所發行的 Token：

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

如果您的應用程式使用了不止一個[認證使用者提供者](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --password` 指令建立 Client 時，提供 `--provider` 選項來指定密碼模式 Client 所使用的使用者提供者。給定的提供者名稱應與您的應用程式 `config/auth.php` 設定檔中定義的有效提供者相符合。接著您就可以[使用中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自 Guard 所指定的提供者的使用者才能獲得授權。


<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

當使用密碼模式進行認證時，Passport 會使用可認證 Model 的 `email` 屬性作為 "username"。然而，您可以透過在 Model 上定義 `findForPassport` 方法來自訂此行為：

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

當使用密碼模式進行認證時，Passport 會使用 Model 的 `password` 屬性來驗證給定的密碼。如果您的 Model 沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，可以在 Model 上定義 `validateForPassportPasswordGrant` 方法：

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
## 隱式模式 (Implicit Grant)

> [!WARNING]
> 我們不再建議使用隱式模式 Token。相反地，您應該選擇 [OAuth2 Server 目前推薦的授權模式](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隱式模式與授權碼模式類似；然而，Token 會直接返回給 Client，而不需要交換授權碼。此模式最常用於無法安全儲存 Client 憑證的 JavaScript 或行動應用程式。若要啟用該模式，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

在您的應用程式可以透過隱式模式發行 Token 之前，您需要建立一個隱式模式 Client。您可以使用帶有 `--implicit` 選項的 `passport:client` Artisan 指令來做到這一點。

```shell
php artisan passport:client --implicit
```

一旦啟用了該模式並建立了隱式 Client，開發人員就可以使用他們的 Client ID 向您的應用程式請求 Access Token。消費端應用程式應該對您應用程式的 `/oauth/authorize` 路由發起重導向請求，如下所示：

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
> 請記住，`/oauth/authorize` 路由已經由 Passport 定義。您不需要手動定義此路由。

<a name="client-credentials-grant"></a>
## Client 憑證模式 (Client Credentials Grant)

Client 憑證模式適用於機器對機器 (Machine-to-Machine) 的認證。例如，您可能會在透過 API 執行維護任務的排程工作中發揮此授權模式的作用。

在您的應用程式可以透過 Client 憑證模式發行 Token 之前，您需要先建立一個 Client 憑證模式 Client。您可以使用 `passport:client` Artisan 命令的 `--client` 選項來執行此操作：

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

若要將路由的存取權限限制在特定 Scope，您可以傳遞所需的 Scope 列表給 `using` 方法：

```php
Route::get('/orders', function (Request $request) {
    // Access token is valid, the client is resource owner, and has both "servers:read" and "servers:create" scopes...
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```

> [!WARNING]
> [底層的 OAuth2 伺服器](https://oauth2.thephpleague.com/database-setup/#:~:text=Please%20note%20that,the%20bearer%20token.) 會針對 Client 憑證 Token，將 Token 的 `sub` 聲明 (Claim) 設定為 Client 的識別碼。預設情況下，Passport 在 Client 上使用 UUID，因此這不會與使用整數的主鍵使用者發生碰撞。但是，如果您將 `Passport::$clientUuids` 設定為 `false`，則 Client 憑證 Token 可能會不小心解析出 ID 與該 Client ID 相同的使用者。在這種情況下，使用此中介層無法保證傳入的 Token 確實是 Client 憑證 Token。


<a name="retrieving-tokens"></a>
### 取得 Token

若要使用此授權模式取得 Token，請向 `oauth/token` 端點發送請求：

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

有時，您的使用者可能希望自行發行存取 Token，而無需經過典型的授權碼重新導向流程。允許使用者透過您應用程式的 UI 自行發行 Token，對於讓使用者測試您的 API 非常有用，或者也可以作為一般發行存取 Token 的更簡單方法。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 來發行個人存取 Token，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 用於發行 API 存取 Token 的輕量級官方函式庫。


<a name="creating-a-personal-access-client"></a>
### 建立個人存取 Client

在您的應用程式可以發行個人存取 Token 之前，您需要先建立一個個人存取 Client。您可以透過執行帶有 `--personal` 選項的 `passport:client` Artisan 命令來做到這一點。如果您已經執行過 `passport:install` 命令，則不需要再執行此命令：

```shell
php artisan passport:client --personal
```


<a name="customizing-the-user-provider-for-pat"></a>
### 自訂使用者提供者

如果您的應用程式使用了不只一個 [認證使用者提供者](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --personal` 命令建立 Client 時，提供 `--provider` 選項來指定個人存取模式 Client 使用哪一個使用者提供者。給定的提供者名稱應符合您應用程式 `config/auth.php` 設定檔中定義的有效提供者。接著您可以[使用中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自該 Guard 指定之提供者的使用者被授權。


<a name="managing-personal-access-tokens"></a>
### 管理個人存取 Token

當您建立好個人存取 Client 後，就可以使用 `App\Models\User` Model 實例上的 `createToken` 方法為指定使用者發行 Token。`createToken` 方法接受 Token 名稱作為其第一個引數，並接受一個可選的 [Scope](#token-scopes) 陣列作為其第二個引數：

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

Passport 包含一個 [認證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，用來驗證傳入請求上的存取 Token。一旦您將 `api` Guard 設定為使用 `passport` 驅動器，您只需要在任何需要有效存取 Token 的路由上指定 `auth:api` 中介層即可：

```php
Route::get('/user', function () {
    // Only API authenticated users may access this route...
})->middleware('auth:api');
```

> [!WARNING]
> 如果您正在使用 [Client 憑證模式](#client-credentials-grant)，您應該使用 [`Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層](#client-credentials-grant) 來保護您的路由，而非 `auth:api` 中介層。


<a name="multiple-authentication-guards"></a>
#### 多重認證 Guard

如果您的應用程式需要對使用完全不同 Eloquent Model 的不同類型使用者進行認證，您可能需要為應用程式中的每個使用者提供者類型定義 Guard 設定。這允許您保護針對特定使用者提供者的請求。例如，假設 `config/auth.php` 設定檔中有以下 Guard 設定：

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

以下路由將利用使用 `customers` 使用者提供者的 `api-customers` Guard 來認證傳入的請求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> [!NOTE]
> 關於在 Passport 中使用多個使用者提供者的更多資訊，請參閱[個人存取 Token 文件](#customizing-the-user-provider-for-pat)和[密碼模式文件](#customizing-the-user-provider)。


<a name="passing-the-access-token"></a>
### 傳送 Access Token

當呼叫受 Passport 保護的路由時，您應用程式的 API 消費者應在其請求的 `Authorization` 標頭中將存取 Token 指定為 `Bearer` Token。例如，在使用 `Http` Facade 時：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>
## Token 範圍 (Token Scopes)

Scope 讓您的 API Client 能在請求存取帳號的授權時，指定一組特定的權限。例如，若您正在建立一個電子商務應用程式，並非所有 API 消費者都需要下單的能力。相反地，您可以允許消費者僅請求存取訂單發貨狀態的授權。換句話說，Scope 允許您應用程式的使用者限制第三方應用程式代表他們執行的動作。


<a name="defining-scopes"></a>
### 定義 Scope

您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中使用 `Passport::tokensCan` 方法來定義 API 的 Scope。`tokensCan` 方法接受一個包含 Scope 名稱與 Scope 描述的陣列。Scope 的描述可以是任何您希望的內容，並將會在授權核准畫面上顯示給使用者：

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

如果 Client 沒有請求任何特定的 Scope，您可以設定 Passport 伺服器，使用 `defaultScopes` 方法為 Token 附加預設的 Scope。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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
### 指派 Scope 給 Token


<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

當使用授權碼模式請求 Access Token 時，消費者應在 `scope` 查詢字串參數中指定其所需的 Scope。`scope` 參數應為以空白字元分隔的 Scope 列表：

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

如果您使用 `App\Models\User` Model 的 `createToken` 方法核發個人存取 Token，您可以將所需 Scope 的陣列作為第二個引數傳遞給該方法：

```php
$token = $user->createToken('My Token', ['orders:create'])->accessToken;
```


<a name="checking-scopes"></a>
### 檢查 Scope

Passport 包含兩個中介層，可用於驗證傳入的請求是否已透過被授予指定 Scope 的 Token 進行認證。


<a name="check-for-all-scopes"></a>
#### 檢查是否具備所有 Scope

可以將 `Laravel\Passport\Http\Middleware\CheckToken` 中介層指派給路由，以驗證傳入請求的 Access Token 是否具有所有列出的 Scope：

```php
use Laravel\Passport\Http\Middleware\CheckToken;

Route::get('/orders', function () {
    // Access token has both "orders:read" and "orders:create" scopes...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);
```


<a name="check-for-any-scopes"></a>
#### 檢查是否具備任一 Scope

可以將 `Laravel\Passport\Http\Middleware\CheckTokenForAnyScope` 中介層指派給路由，以驗證傳入請求的 Access Token 是否具有列出 Scope 中的*至少一個*：

```php
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

Route::get('/orders', function () {
    // Access token has either "orders:read" or "orders:create" scope...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```


<a name="scope-attributes"></a>
#### Scope Attribute

如果您的應用程式使用[控制器中介層 Attribute](/docs/{{version}}/controllers#middleware-attributes)，您可以使用 `Laravel\Passport\Attributes\AuthorizeToken` Attribute 作為 Passport 的 Scope 中介層的便利捷徑：

```php
<?php

namespace App\Http\Controllers;

use Laravel\Passport\Attributes\AuthorizeToken;

#[AuthorizeToken('orders:read')]
#[AuthorizeToken('orders:create', only: ['store'])]
class OrderController
{
    #[AuthorizeToken(['orders:read', 'orders:create'], anyScope: true)]
    public function index()
    {
        // Access token has either "orders:read" or "orders:create" scope...
    }

    public function store()
    {
        // Access token has both "orders:read" and "orders:create" scopes...
    }
}
```

預設情況下，`AuthorizeToken` Attribute 需要所有給定的 Scope。若您傳遞 `anyScope: true`，則當 Token 具有至少一個給定的 Scope 時，該請求就會獲得授權。


<a name="checking-scopes-on-a-token-instance"></a>
#### 在 Token 實例上檢查 Scope

一旦經過 Access Token 認證的請求進入您的應用程式，您仍可以在已認證的 `App\Models\User` 實例上使用 `tokenCan` 方法來檢查該 Token 是否具有指定的 Scope：

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

`scopeIds` 方法將傳回所有已定義 ID / 名稱的陣列：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法將傳回所有已定義 Scope 作為 `Laravel\Passport\Scope` 實例的陣列：

```php
Passport::scopes();
```

`scopesFor` 方法將傳回與給定 ID / 名稱相符的 `Laravel\Passport\Scope` 實例陣列：

```php
Passport::scopesFor(['user:read', 'orders:create']);
```

您可以使用 `hasScope` 方法來判斷指定的 Scope 是否已定義：

```php
Passport::hasScope('orders:create');
```

<a name="spa-authentication"></a>
## SPA 認證

在建立 API 時，如果能從你的 JavaScript 應用程式中取用自己的 API，會是非常有用的事。這種 API 開發方式允許你自己的應用程式與你對外公開的 API 使用同一個介面。同一個 API 可以被你的 Web 應用程式、行動應用程式、第三方應用程式以及你發布在各個套件管理工具上的任何 SDK 所使用。

通常，如果你想從 JavaScript 應用程式取用自己的 API，你需要手動將 access token 發送到該應用程式，並在每次向你的應用程式發送請求時隨附該 token。不過，Passport 包含了一個中介層可以為你處理這件事。你只需要將 `CreateFreshApiToken` 中介層附加到應用程式 `bootstrap/app.php` 檔案中的 `web` 中介層群組即可：

```php
use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        CreateFreshApiToken::class,
    ]);
})
```

> [!WARNING]
> 你應該確保 `CreateFreshApiToken` 中介層位於中介層堆疊中的最後一個。

這個中介層會在輸出的回應中附加一個 `laravel_token` cookie。此 cookie 包含一個加密的 JWT，Passport 將使用它來認證來自你的 JavaScript 應用程式的 API 請求。這個 JWT 的有效期限等於你設定檔中的 `session.lifetime` 數值。現在，由於瀏覽器會在後續的所有請求中自動傳送此 cookie，因此你可以直接向應用程式的 API 發送請求，而不必明確傳送 access token：

```js
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```


<a name="customizing-the-cookie-name"></a>
#### 自訂 Cookie 名稱

如果需要，你可以使用 `Passport::cookie` 方法來自訂 `laravel_token` cookie 的名稱。通常，這個方法應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
#### CSRF 防護

使用這種認證方式時，你需要確保請求中包含有效的 CSRF token 表頭 (Header)。骨架應用程式和所有入門套件隨附的預設 Laravel JavaScript 基底中都包含一個 [Axios](https://github.com/axios/axios) 實例，它會自動使用加密的 `XSRF-TOKEN` cookie 值，在同源請求上發送 `X-XSRF-TOKEN` 表頭。

> [!NOTE]
> 如果你選擇傳送 `X-CSRF-TOKEN` 表頭而非 `X-XSRF-TOKEN`，則需要使用由 `csrf_token()` 所提供的未加密 token。


<a name="events"></a>
## 事件

Passport 在簽發 access token 和 refresh token 時會觸發事件。你可以[監聽這些事件](/docs/{{version}}/events)來刪除或撤銷資料庫中的其他 access token：

<div class="overflow-auto">

| Event Name                                    |
| --------------------------------------------- |
| `Laravel\Passport\Events\AccessTokenCreated`  |
| `Laravel\Passport\Events\AccessTokenRevoked`  |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>


<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用於指定當前已認證的使用者及其 scope。傳給 `actingAs` 方法的第一個引數是使用者實例，第二個是應授予該使用者 token 的 scope 陣列：

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

Passport 的 `actingAsClient` 方法可用於指定當前已認證的 client 及其 scope。傳給 `actingAsClient` 方法的第一個引數是 client 實例，第二個是應授予該 client token 的 scope 陣列：

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