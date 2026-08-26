# Laravel Passport

- [簡介](#introduction)
    - [Passport 還是 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [令牌生命週期](#token-lifetimes)
    - [覆寫預設 Model](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [授權碼 Grant](#authorization-code-grant)
    - [管理 Client](#managing-clients)
    - [請求令牌](#requesting-tokens)
    - [管理令牌](#managing-tokens)
    - [刷新令牌](#refreshing-tokens)
    - [撤銷令牌](#revoking-tokens)
    - [清除令牌](#purging-tokens)
- [搭配 PKCE 的授權碼 Grant](#code-grant-pkce)
    - [建立 Client](#creating-a-auth-pkce-grant-client)
    - [請求令牌](#requesting-auth-pkce-grant-tokens)
- [裝置授權 Grant](#device-authorization-grant)
    - [建立裝置碼 Grant Client](#creating-a-device-authorization-grant-client)
    - [請求令牌](#requesting-device-authorization-grant-tokens)
- [密碼 Grant](#password-grant)
    - [建立密碼 Grant Client](#creating-a-password-grant-client)
    - [請求令牌](#requesting-password-grant-tokens)
    - [請求所有 Scope](#requesting-all-scopes)
    - [自訂 User Provider](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱式 Grant](#implicit-grant)
- [客戶端憑證 Grant](#client-credentials-grant)
    - [取得令牌](#retrieving-tokens)
- [個人存取令牌 (Personal Access Tokens)](#personal-access-tokens)
    - [建立個人存取 Client](#creating-a-personal-access-client)
    - [自訂 User Provider](#customizing-the-user-provider-for-pat)
    - [管理個人存取令牌](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過中介層](#via-middleware)
    - [傳遞存取令牌](#passing-the-access-token)
- [令牌 Scope](#token-scopes)
    - [定義 Scope](#defining-scopes)
    - [預設 Scope](#default-scope)
    - [指派 Scope 給令牌](#assigning-scopes-to-tokens)
    - [檢查 Scope](#checking-scopes)
- [SPA 認證](#spa-authentication)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 在短短幾分鐘之內就能為你的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 是建構於 Andy Millington 和 Simon Hamp 所維護的 [League OAuth2 server](https://github.com/thephpleague/oauth2-server) 之上。

> [!NOTE]
> 本文件假設你已經熟悉 OAuth2。如果你對 OAuth2 完全不了解，請在繼續之前先熟悉 OAuth2 的一般[專有名詞](https://oauth2.thephpleague.com/terminology/)和功能特性。


<a name="passport-or-sanctum"></a>
### Passport 還是 Sanctum？

在開始之前，你可能需要先確定你的應用程式使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum) 會比較合適。如果你的應用程式絕對需要支援 OAuth2，那麼你應該使用 Laravel Passport。

然而，如果你只是想要對單頁應用程式（SPA）、行動裝置應用程式進行認證，或是發行 API 令牌，則應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 並不支援 OAuth2；不過，它提供了簡單許多的 API 認證開發體驗。


<a name="installation"></a>
## 安裝

你可以透過 `install:api` Artisan 指令來安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此指令會發布並執行資料庫遷移，以建立你的應用程式用來儲存 OAuth2 Client 和存取令牌所需的資料表。該指令還會建立產生安全存取令牌所需的加密金鑰。

執行 `install:api` 指令後，請將 `Laravel\Passport\HasApiTokens` trait 與 `Laravel\Passport\Contracts\OAuthenticatable` 介面新增至你的 `App\Models\User` Model。這個 trait 會為你的 Model 提供一些輔助方法，讓你能夠檢視通過認證的使用者令牌與 Scope：

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

最後，在應用程式的 `config/auth.php` 設定檔中，你應該定義一個 `api` 認證 Guard，並將 `driver` 選項設定為 `passport`。這會指示你的應用程式在對傳入的 API 請求進行認證時使用 Passport 的 `TokenGuard`：

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

當首次將 Passport 部署到應用程式伺服器時，你可能需要執行 `passport:keys` 指令。此指令會產生 Passport 生成存取令牌所需的加密金鑰。生成的金鑰通常不應保存在版本控制中：

```shell
php artisan passport:keys
```

如果有需要，你可以定義載入 Passport 金鑰的路徑。你可以使用 `Passport::loadKeysFrom` 方法來達成此目的。通常，此方法應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

在發布設定檔後，你可以透過將應用程式的加密金鑰定義為環境變數來進行載入：

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

升級至 Passport 的新主版本時，請務必仔細閱讀[升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。


<a name="configuration"></a>
## 設定


<a name="token-lifetimes"></a>
### 令牌生命週期

預設情況下，Passport 會發行有效期為一年的長效型存取令牌。如果你想設定更長或更短的令牌生命週期，可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> Passport 資料表中的 `expires_at` 欄位是唯讀的，僅用於顯示目的。在發行令牌時，Passport 將過期資訊儲存在經簽署且加密的令牌內部。如果你需要讓某個令牌失效，應該[撤銷它](#revoking-tokens)。


<a name="overriding-default-models"></a>
### 覆寫預設 Model

你可以透過定義自己的 Model 並繼承對應的 Passport Model，自由地擴充 Passport 內部所使用的 Model：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定義好 Model 之後，你可以透過 `Laravel\Passport\Passport` 類別來指示 Passport 使用你的自訂 Model。通常，你應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 關於你的自訂 Model：

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

有時你可能希望自訂由 Passport 定義的路由。為達成此目的，你首先需要在應用程式 `AppServiceProvider` 的 `register` 方法中加入 `Passport::ignoreRoutes`，藉此忽略 Passport 註冊的路由：

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
## 授權碼 Grant

使用授權碼的 OAuth2 是大多數開發者最熟悉的 OAuth2 使用方式。當使用授權碼時，Client 應用程式會將使用者重新導向至您的伺服器，使用者將在該處批准或拒絕向 Client 發行存取令牌的請求。

首先，我們需要指示 Passport 如何回傳我們的「授權」視圖。

授權視圖的所有渲染邏輯都可以透過 `Laravel\Passport\Passport` 類別所提供的適當方法來進行自訂。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Passport 會自動定義回傳此視圖的 `/oauth/authorize` 路由。您的 `auth.oauth.authorize` 範本應該包含一個向 `passport.authorizations.approve` 路由發送 POST 請求以批准授權的表單，以及一個向 `passport.authorizations.deny` 路由發送 DELETE 請求以拒絕授權的表單。`passport.authorizations.approve` 與 `passport.authorizations.deny` 路由需要 `state`、`client_id` 和 `auth_token` 欄位。


<a name="managing-clients"></a>
### 管理 Client

開發者若要建置需要與您的應用程式 API 進行互動的應用程式，必須透過建立「Client」來向您的應用程式進行註冊。通常，這包括提供其應用程式的名稱以及一個 URI，當使用者批准其授權請求後，您的應用程式可以重新導向至該 URI。


<a name="managing-first-party-clients"></a>
#### 第一方 Client

建立 Client 最簡單的方式是使用 `passport:client` Artisan 指令。此指令可用於建立第一方 Client 或測試您的 OAuth2 功能。當您執行 `passport:client` 指令時，Passport 會提示您輸入更多關於 Client 的資訊，並會為您提供 Client ID 和 Secret：

```shell
php artisan passport:client
```

如果您想為您的 Client 允許多個重導向 URI，可以在 `passport:client` 指令提示輸入 URI 時，使用逗號分隔列表來指定它們。任何包含逗號的 URI 都應該進行 URI 編碼：

```shell
https://third-party-app.com/callback,https://example.com/oauth/redirect
```


<a name="managing-third-party-clients"></a>
#### 第三方 Client

由於您應用程式的使用者將無法使用 `passport:client` 指令，您可以透過 `Laravel\Passport\ClientRepository` 類別的 `createAuthorizationCodeGrantClient` 方法來為指定的使用者註冊 Client：

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

`createAuthorizationCodeGrantClient` 方法會回傳一個 `Laravel\Passport\Client` 的實例。您可以向使用者顯示 `$client->id` 作為 Client ID，以及顯示 `$client->plainSecret` 作為 Client Secret。

<a name="requesting-tokens"></a>
### 請求令牌

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重導向以進行授權

一旦建立 Client 後，開發者便可以使用其 client ID 和 secret 向您的應用程式請求授權碼與存取令牌。首先，消費端應用程式（consuming application）應重導向請求至您應用程式的 `/oauth/authorize` 路由，如下所示：

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

若 `prompt` 的值為 `none`，當使用者尚未在 Passport 應用程式中通過認證時，Passport 永遠會拋出認證錯誤。若值為 `consent`，即使消費端應用程式先前已被授予所有 scope，Passport 永遠會顯示授權核准畫面。當值為 `login` 時，Passport 應用程式將永遠提示使用者重新登入應用程式，即使他們已經擁有現有的 Session。

若未提供 `prompt` 值，則僅在使用者先前未針對請求的 scope 授權存取給該消費端應用程式時，才會提示使用者進行授權。

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 自動定義。您不需要手動定義此路由。

<a name="approving-the-request"></a>
#### 核准請求

當收到授權請求時，Passport 會根據 `prompt` 參數的值（若存在）自動回應，並可能向使用者顯示一個模板，允許他們核准或拒絕授權請求。如果他們核准了請求，將會被重導向回消費端應用程式所指定的 `redirect_uri`。該 `redirect_uri` 必須與建立 Client 時所指定的 `redirect` URL 一致。

有時您可能希望跳過授權提示，例如在對第一方 Client 進行授權時。您可以透過[擴充 `Client` Model](#overriding-default-models) 並定義 `skipsAuthorization` 方法來實現。如果 `skipsAuthorization` 回傳 `true`，該 Client 將會自動被核准，且使用者會立即被重導向回 `redirect_uri`，除非消費端應用程式在重導向以進行授權時顯式設定了 `prompt` 參數：

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
#### 將授權碼轉換為存取令牌

如果使用者核准了授權請求，將會被重導向回消費端應用程式。消費端應首先將 `state` 參數與重導向前儲存的值進行驗證。如果 state 參數比對成功，消費端應向您的應用程式發送一個 `POST` 請求以請求存取令牌。該請求應包含使用者核准授權請求時由您應用程式所核發的授權碼：

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

此 `/oauth/token` 路由將回傳包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取令牌過期的秒數。

> [!NOTE]
> 與 `/oauth/authorize` 路由一樣，`/oauth/token` 路由已由 Passport 為您定義好了。不需要手動定義此路由。

<a name="managing-tokens"></a>
### 管理令牌

您可以使用 `Laravel\Passport\HasApiTokens` Trait 的 `tokens` 方法來取得使用者已授權的令牌。例如，這可以用於為您的使用者提供一個儀表板，以追蹤他們與第三方應用程式的連線：

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
### 刷新令牌

如果您的應用程式核發的是短效期存取令牌，使用者需要透過核發存取令牌時一併提供給他們的刷新令牌（refresh token）來刷新其存取令牌：

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

此 `/oauth/token` 路由將回傳包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取令牌過期的秒數。

<a name="revoking-tokens"></a>
### 撤銷令牌

您可以使用 `Laravel\Passport\Token` Model 上的 `revoke` 方法來撤銷令牌。您可以使用 `Laravel\Passport\RefreshToken` Model 上的 `revoke` 方法來撤銷令牌的刷新令牌：

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
### 清除令牌

當令牌已被撤銷或過期時，您可能希望從資料庫中清除它們。Passport 內建的 `passport:purge` Artisan 命令可以為您執行此操作：

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

您也可以在應用程式的 `routes/console.php` 檔案中設定[排程任務](/docs/{{version}}/scheduling)，以依照時程自動清除您的令牌：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

<a name="code-grant-pkce"></a>
## 搭配 PKCE 的授權碼 Grant

搭配「Proof Key for Code Exchange」(PKCE) 的授權碼 Grant 是一種用來驗證單頁應用程式 (SPA) 或行動應用程式存取 API 的安全方式。當您無法保證 Client 密鑰 (Client secret) 能被安全地保密時，或者為了減輕授權碼被攻擊者攔截的威脅，就應該使用此 Grant。在用授權碼交換存取令牌時，會使用「code verifier (程式碼驗證器)」與「code challenge (程式碼挑戰)」的組合來取代 Client 密鑰。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立 Client

在您的應用程式可以透過搭配 PKCE 的授權碼 Grant 發行令牌之前，您需要建立一個啟用了 PKCE 的 Client。您可以透過帶有 `--public` 選項的 `passport:client` Artisan 指令來完成此操作：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求令牌

<a name="code-verifier-code-challenge"></a>
#### Code Verifier 與 Code Challenge

由於此授權 Grant 不提供 Client 密鑰，開發人員需要產生 code verifier 與 code challenge 的組合，以便請求令牌。

根據 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636)所定義，code verifier 應該是一個長度介於 43 到 128 個字元之間的隨機字串，包含字母、數字以及 `"-"`、`"."`、`"_"`、`"~"` 字元。

code challenge 應該是經過 Base64 編碼且符合 URL 與檔名安全的字串。應移除末尾的 `'='` 字元，且不應包含換行符號、空白或其他額外字元。

```php
$encoded = base64_encode(hash('sha256', $codeVerifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

建立 Client 後，您可以使用 Client ID 以及產生的 code verifier 與 code challenge 來向您的應用程式請求授權碼與存取令牌。首先，消費端應用程式應該對您應用程式的 `/oauth/authorize` 路由發起重新導向請求：

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
#### 將授權碼轉換為存取令牌

如果使用者批准了授權請求，他們將被重新導向回消費端應用程式。如同標準的授權碼 Grant 一樣，消費端應先根據重新導向前儲存的值驗證 `state` 參數。

如果 state 參數匹配，消費端應該向您的應用程式發送 `POST` 請求以請求存取令牌。該請求應包含使用者批准授權請求時由您的應用程式發行的授權碼，以及最初產生的 code verifier：

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
## 裝置授權 Grant

OAuth2 裝置授權 Grant (Device Authorization Grant) 允許無瀏覽器或輸入受限的裝置（例如電視和遊戲主機），透過交換「裝置碼 (device code)」來取得存取令牌。使用裝置流程 (device flow) 時，裝置 Client 會指示使用者使用次要裝置（例如電腦或智慧型手機）並連線至您的伺服器，使用者將在伺服器輸入所提供的「使用者碼 (user code)」，並同意或拒絕存取請求。

開始之前，我們需要指示 Passport 如何回傳我們的「使用者碼」與「授權」檢視。

所有授權檢視的渲染邏輯都可以透過 `Laravel\Passport\Passport` 類別中適當的方法來進行自訂。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法裡呼叫此方法。

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

Passport 會自動定義回傳這些檢視的路由。您的 `auth.oauth.device.user-code` 樣板應包含一個表單，該表單會向 `passport.device.authorizations.authorize` 路由發送 GET 請求。`passport.device.authorizations.authorize` 路由預期需要一個 `user_code` 查詢參數。

您的 `auth.oauth.device.authorize` 樣板應包含一個發送 POST 請求至 `passport.device.authorizations.approve` 路由以核准授權的表單，以及一個發送 DELETE 請求至 `passport.device.authorizations.deny` 路由以拒絕授權的表單。`passport.device.authorizations.approve` 與 `passport.device.authorizations.deny` 路由預期需要 `state`、`client_id` 以及 `auth_token` 欄位。


<a name="creating-a-device-authorization-grant-client"></a>
### 建立裝置碼 Grant Client

在您的應用程式能夠透過裝置授權 Grant 發行令牌之前，您需要建立一個已啟用裝置流程的 Client。您可以使用帶有 `--device` 選項的 `passport:client` Artisan 指令來達成此目的。此指令將建立一個啟用了裝置流程的第一方 Client，並為您提供 Client ID 與 Secret：

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
### 請求令牌


<a name="device-code"></a>
#### 請求裝置碼

建立 Client 後，開發人員可以使用其 Client ID 向您的應用程式請求裝置碼。首先，消費端裝置應向您的應用程式的 `/oauth/device/code` 路由發送 `POST` 請求以請求裝置碼：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

這將回傳一個包含 `device_code`、`user_code`、`verification_uri`、`interval` 以及 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含裝置碼過期前的秒數。`interval` 屬性包含消費端裝置在輪詢 `/oauth/token` 路由時，各請求之間應等待的秒數，以避免發送過於頻繁導致速率限制錯誤。

> [!NOTE]
> 請記住，`/oauth/device/code` 路由已經由 Passport 定義好。您不需要手動定義此路由。


<a name="user-code"></a>
#### 顯示驗證 URI 與使用者碼

取得裝置碼請求後，消費端裝置應指示使用者使用另一個裝置造訪所提供的 `verification_uri` 並輸入 `user_code`，以核准該授權請求。


<a name="polling-token-request"></a>
#### 輪詢令牌請求

由於使用者將使用單獨的裝置來授權（或拒絕）存取，消費端裝置應輪詢您的應用程式的 `/oauth/token` 路由，以確認使用者何時對請求做出回應。消費端裝置在請求裝置碼時，應使用 JSON 回應中提供的最小輪詢 `interval`，以避免發生速率限制錯誤：

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

如果使用者已核准授權請求，這將回傳一個包含 `access_token`、`refresh_token` 及 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取令牌過期前的秒數。

<a name="password-grant"></a>
## 密碼 Grant

> [!WARNING]
> 我們不再建議使用密碼 Grant 令牌。相反地，您應該選擇 [OAuth2 Server 目前推薦的 Grant 類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密碼 Grant 允許您的其他第一方 Client（例如行動應用程式）使用 Email 地址 / 使用者名稱以及密碼來取得存取令牌。這讓您能夠安全地向第一方 Client 發行存取令牌，而不需要使用者經過完整的 OAuth2 授權碼轉址流程。

若要啟用密碼 Grant，請在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

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
### 建立密碼 Grant Client

在您的應用程式可以透過密碼 Grant 發行令牌之前，您需要先建立一個密碼 Grant Client。您可以使用帶有 `--password` 選項的 `passport:client` Artisan 指令來做到這一點。

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>
### 請求令牌

當您啟用了 Grant 並且建立了密碼 Grant Client 後，即可透過攜帶使用者的 Email 地址和密碼發送 `POST` 請求至 `/oauth/token` 路由來請求存取令牌。請記住，此路由已經由 Passport 自動註冊，因此不需要手動定義。如果請求成功，您將在來自伺服器的 JSON 回應中收到 `access_token` 和 `refresh_token`：

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
> 請記住，預設情況下存取令牌是長效型的。不過，如有需要，您可以自由地[設定您存取令牌的最長生命週期](#configuration)。

<a name="requesting-all-scopes"></a>
### 請求所有 Scope

使用密碼 Grant 或 Client Credentials Grant 時，您可能希望為應用程式支援的所有 Scope 來進行令牌授權。您可以透過請求 `*` Scope 來達成。如果您請求 `*` Scope，令牌實例上的 `can` 方法將會永遠回傳 `true`。此 Scope 僅能指派給使用 `password` 或 `client_credentials` Grant 發行的令牌：

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
### 自訂 User Provider

如果您的應用程式使用不只一個[認證 User Provider](/docs/{{version}}/authentication#introduction)，您可以透過 `artisan passport:client --password` 指令建立 Client 時提供 `--provider` 選項，來指定密碼 Grant Client 使用哪一個 User Provider。所提供的 Provider 名稱應符合您應用程式 `config/auth.php` 設定檔中定義的有效 Provider。接著您可以[使用中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自 Guard 指定 Provider 的使用者才能獲得授權。

<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

使用密碼 Grant 進行認證時，Passport 會使用可認證 Model 的 `email` 屬性作為「使用者名稱 (username)」。不過，您可以透過在 Model 上定義 `findForPassport` 方法來自訂此行為：

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

使用密碼 Grant 進行認證時，Passport 會使用 Model 的 `password` 屬性來驗證給定的密碼。如果您的 Model 沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，可以在 Model 上定義 `validateForPassportPasswordGrant` 方法：

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
## 隱式 Grant

> [!WARNING]
> 我們不再建議使用隱式 Grant 令牌。相反地，您應該選擇 [OAuth2 Server 目前推薦的 Grant 類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隱式 Grant 與授權碼 Grant 類似；但是，令牌會直接回傳給 Client，而不需要先兌換授權碼。此 Grant 最常用於無法安全儲存 Client 憑證的 JavaScript 或行動應用程式。若要啟用該 Grant，請在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

在您的應用程式可以透過隱式 Grant 發行令牌之前，您需要建立一個隱式 Grant Client。您可以使用帶有 `--implicit` 選項的 `passport:client` Artisan 指令來做到這一點。

```shell
php artisan passport:client --implicit
```

一旦啟用了 Grant 並且建立了隱式 Client，開發人員即可使用其 Client ID 向您的應用程式請求存取令牌。使用端應用程式應像這樣向您應用程式的 `/oauth/authorize` 路由發起轉址請求：

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
## 客戶端憑證 Grant

客戶端憑證 Grant 適用於機器對機器的認證。例如，您可能會在透過 API 執行維護任務的排程任務中使用此 Grant。

在您的應用程式可以透過客戶端憑證 Grant 發行令牌之前，您需要建立一個客戶端憑證 Grant 的 Client。您可以使用 `passport:client` Artisan 命令的 `--client` 選項來做到這點：

```shell
php artisan passport:client --client
```

接著，將 `Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層指派給路由：

```php
use Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner;

Route::get('/orders', function (Request $request) {
    // Access token is valid and the client is resource owner...
})->middleware(EnsureClientIsResourceOwner::class);
```

若要將路由的存取權限限制在特定的 Scope，您可以向 `using` 方法提供所需的 Scope 列表：

```php
Route::get('/orders', function (Request $request) {
    // Access token is valid, the client is resource owner, and has both "servers:read" and "servers:create" scopes...
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```

> [!WARNING]
> [底層的 OAuth2 伺服器](https://oauth2.thephpleague.com/database-setup/#:~:text=Please%20note%20that,the%20bearer%20token.)會將客戶端憑證令牌的 `sub` 聲明 (Claim) 設定為 Client 的識別碼。預設情況下，Passport 在 Client 上使用 UUID，因此這不會與使用整數主鍵的使用者產生衝突。然而，如果您將 `Passport::$clientUuids` 設定為 `false`，客戶端憑證令牌可能會不小心解析到 ID 與 Client ID 相符的使用者。在這種情況下，使用此中介層無法保證傳入的令牌是客戶端憑證令牌。


<a name="retrieving-tokens"></a>
### 取得令牌

若要使用此 Grant 類型取得令牌，請向 `oauth/token` 端點發送請求：

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
## 個人存取令牌 (Personal Access Tokens)

有時，您的使用者可能希望為自己發行存取令牌，而不需要經過典型的授權碼重新導向流程。允許使用者透過您的應用程式 UI 自行發行令牌，對於讓使用者測試您的 API 非常有用，或者作為一般發行存取令牌更簡單的方法。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 來發行個人存取令牌，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 用於發行 API 存取令牌的輕量級官方套件。


<a name="creating-a-personal-access-client"></a>
### 建立個人存取 Client

在您的應用程式可以發行個人存取令牌之前，您需要建立一個個人存取 Client。您可以透過執行帶有 `--personal` 選項的 `passport:client` Artisan 命令來做到這一點。如果您已經執行過 `passport:install` 命令，則不需要再執行此命令：

```shell
php artisan passport:client --personal
```


<a name="customizing-the-user-provider-for-pat"></a>
### 自訂 User Provider

如果您的應用程式使用多個[認證使用者提供者 (User Provider)](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --personal` 命令建立 Client 時提供 `--provider` 選項，來指定個人存取 Grant Client 所使用的 User Provider。給定的 Provider 名稱應與您應用程式的 `config/auth.php` 設定檔中定義的有效 Provider 相符。接著您可以使用[中介層來保護您的路由](#multiple-authentication-guards)，以確保只有來自 Guard 指定 Provider 的使用者才能獲得授權。


<a name="managing-personal-access-tokens"></a>
### 管理個人存取令牌

建立個人存取 Client 後，您可以使用 `App\Models\User` Model 實例上的 `createToken` 方法為特定使用者發行令牌。`createToken` 方法的第一個引數接收令牌的名稱，第二個引數接收可選的 [Scope](#token-scopes) 陣列：

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

Passport 包含一個[認證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，它會驗證傳入請求中的存取令牌。當您將 `api` Guard 設定為使用 `passport` 驅動程式後，您只需要在任何需要有效存取令牌的路由上指定 `auth:api` 中介層即可：

```php
Route::get('/user', function () {
    // Only API authenticated users may access this route...
})->middleware('auth:api');
```

> [!WARNING]
> 如果您正在使用[客戶端憑證 Grant](#client-credentials-grant)，您應該使用 [`Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層](#client-credentials-grant)來保護您的路由，而不是使用 `auth:api` 中介層。


<a name="multiple-authentication-guards"></a>
#### 多重認證 Guard

如果您的應用程式會驗證不同類型的使用者，而這些使用者可能使用完全不同的 Eloquent Model，您可能需要為應用程式中的每個 User Provider 類型定義一個 Guard 設定。這讓您可以保護針對特定 User Provider 的請求。例如，在 `config/auth.php` 設定檔中給定以下 Guard 設定：

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

以下路由將使用 `api-customers` Guard（該 Guard 使用 `customers` User Provider）來認證傳入的請求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> [!NOTE]
> 關於在 Passport 中使用多個 User Provider 的更多資訊，請參考[個人存取令牌文件](#customizing-the-user-provider-for-pat)以及[密碼 Grant 文件](#customizing-the-user-provider)。


<a name="passing-the-access-token"></a>
### 傳遞存取令牌

當呼叫受 Passport 保護的路由時，您應用程式的 API 消費者應該在其請求的 `Authorization` 標頭中將存取令牌指定為 `Bearer` 令牌。例如，使用 `Http` Facade 時：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>
## 令牌 Scope

Scope 允許您的 API Client 在請求存取帳號授權時，要求一組特定的權限。例如，若您正在建立一個電子商務應用程式，並非所有的 API 消費者都需要下單的能力。相反地，您可以允許消費者僅請求存取訂單出貨狀態的授權。換句話說，Scope 允許您應用程式的使用者限制第三方應用程式代表他們執行的操作。


<a name="defining-scopes"></a>
### 定義 Scope

您可以在應用程式的 `App\Providers\AppServiceProvider` 類別之 `boot` 方法中使用 `Passport::tokensCan` 方法來定義 API 的 Scope。`tokensCan` 方法接收一個包含 Scope 名稱與 Scope 描述的陣列。Scope 描述可以是您希望的任何內容，並將會在授權核准畫面上顯示給使用者：

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

如果 Client 沒有請求任何特定的 Scope，您可以設定 Passport 伺服器使用 `defaultScopes` 方法為令牌附加預設的 Scope。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法呼叫此方法：

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
### 指派 Scope 給令牌


<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

當使用授權碼 Grant 請求存取令牌時，消費者應將其所需的 Scope 指定為 `scope` 查詢字串參數。`scope` 參數應為以空格分隔的 Scope 列表：

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
#### 核發個人存取令牌時

如果您使用 `App\Models\User` Model 的 `createToken` 方法核發個人存取令牌，您可以將所需 Scope 的陣列作為第二個引數傳遞給該方法：

```php
$token = $user->createToken('My Token', ['orders:create'])->accessToken;
```


<a name="checking-scopes"></a>
### 檢查 Scope

Passport 包含兩個中介層，可用於驗證傳入的請求是否已透過被授予指定 Scope 的令牌進行認證。


<a name="check-for-all-scopes"></a>
#### 檢查所有 Scope

`Laravel\Passport\Http\Middleware\CheckToken` 中介層可以指派給路由，以驗證傳入請求的存取令牌是否具備所有列出的 Scope：

```php
use Laravel\Passport\Http\Middleware\CheckToken;

Route::get('/orders', function () {
    // Access token has both "orders:read" and "orders:create" scopes...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);
```


<a name="check-for-any-scopes"></a>
#### 檢查任一 Scope

`Laravel\Passport\Http\Middleware\CheckTokenForAnyScope` 中介層可以指派給路由，以驗證傳入請求的存取令牌是否具備列出的 Scope 中的*至少一個*：

```php
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

Route::get('/orders', function () {
    // Access token has either "orders:read" or "orders:create" scope...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```


<a name="scope-attributes"></a>
#### Scope Attribute

如果您的應用程式使用 [Controller 中介層 Attribute](/docs/{{version}}/controllers#middleware-attributes)，您可以使用 `Laravel\Passport\Attributes\AuthorizeToken` Attribute 作為 Passport 的 Scope 中介層的便利快捷方式：

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

預設情況下，`AuthorizeToken` Attribute 需要所有給定的 Scope。如果您傳入 `anyScope: true`，當令牌具備至少一個給定的 Scope 時，該請求就會被授權。


<a name="checking-scopes-on-a-token-instance"></a>
#### 在令牌實例上檢查 Scope

一旦經存取令牌認證的請求進入您的應用程式，您仍可以在已認證的 `App\Models\User` 實例上使用 `tokenCan` 方法來檢查該令牌是否具備給定的 Scope：

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

`scopeIds` 方法將回傳所有已定義的 ID / 名稱陣列：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法將回傳所有已定義 Scope 的陣列，其內容為 `Laravel\Passport\Scope` 的實例：

```php
Passport::scopes();
```

`scopesFor` 方法將回傳與給定 ID / 名稱相符的 `Laravel\Passport\Scope` 實例陣列：

```php
Passport::scopesFor(['user:read', 'orders:create']);
```

您可以使用 `hasScope` 方法來判斷給定的 Scope 是否已被定義：

```php
Passport::hasScope('orders:create');
```

<a name="spa-authentication"></a>
## SPA 認證

在建立 API 時，能夠從您的 JavaScript 應用程式呼叫您自己的 API 是非常有用的。這種 API 開發方式允許您自己的應用程式使用與您對外分享的相同 API。同一個 API 可以由您的 Web 應用程式、行動應用程式、第三方應用程式以及您在各種套件管理工具上發布的任何 SDK 所使用。

通常，如果您想從 JavaScript 應用程式呼叫 API，您需要手動將存取令牌發送到該應用程式，並在每次向您的應用程式發送請求時隨附該令牌。然而，Passport 包含了一個可以為您處理此問題的中介層。您只需要將 `CreateFreshApiToken` 中介層附加到應用程式 `bootstrap/app.php` 檔案中的 `web` 中介層群組即可：

```php
use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        CreateFreshApiToken::class,
    ]);
})
```

> [!WARNING]
> 您應該確保 `CreateFreshApiToken` 中介層是中介層堆疊中列出的最後一個中介層。

這個中介層會在發出的回應中附加一個 `laravel_token` Cookie。此 Cookie 包含一個加密的 JWT，Passport 會使用它來認證來自 JavaScript 應用程式的 API 請求。JWT 的生命週期等於您的 `session.lifetime` 設定值。現在，由於瀏覽器會自動在所有後續請求中傳送該 Cookie，因此您可以向應用程式的 API 發出請求，而無需明確傳遞存取令牌：

```js
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

<a name="customizing-the-cookie-name"></a>
#### 自訂 Cookie 名稱

如果需要，您可以使用 `Passport::cookie` 方法來自訂 `laravel_token` Cookie 的名稱。通常，此方法應該在應用程式的 `App\Providers\AppServiceProvider` 類別中的 `boot` 方法內呼叫：

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

當使用這種認證方式時，您需要確保請求中包含有效的 CSRF Token 標頭。基底應用程式和所有入門套件中內建的預設 Laravel JavaScript 腳手架都包含一個 [Axios](https://github.com/axios/axios) 實例，它會自動使用加密的 `XSRF-TOKEN` Cookie 值在同源請求上傳送 `X-XSRF-TOKEN` 標頭。

> [!NOTE]
> 如果您選擇傳送 `X-CSRF-TOKEN` 標頭而不是 `X-XSRF-TOKEN`，您需要使用 `csrf_token()` 提供的未加密令牌。

<a name="events"></a>
## 事件

Passport 在核發存取令牌和刷新令牌時會發起事件。您可以[監聽這些事件](/docs/{{version}}/events)以精簡或撤銷資料庫中的其他存取令牌：

<div class="overflow-auto">

| Event Name                                    |
| --------------------------------------------- |
| `Laravel\Passport\Events\AccessTokenCreated`  |
| `Laravel\Passport\Events\AccessTokenRevoked`  |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>

<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用於指定當前已認證的使用者及其 Scope。給予 `actingAs` 方法的第一個引數是使用者實例，第二個是應該授予該使用者令牌的 Scope 陣列：

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

Passport 的 `actingAsClient` 方法可用於指定當前已認證的 Client 及其 Scope。給予 `actingAsClient` 方法的第一個引數是 Client 實例，第二個是應該授予該 Client 令牌的 Scope 陣列：

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