# Laravel Passport

- [簡介](#introduction)
    - [Passport 還是 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [令牌生命週期](#token-lifetimes)
    - [覆寫預設模型](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [授權碼授予 (Authorization Code Grant)](#authorization-code-grant)
    - [管理客戶端](#managing-clients)
    - [請求令牌](#requesting-tokens)
    - [管理令牌](#managing-tokens)
    - [重新整理令牌](#refreshing-tokens)
    - [撤銷令牌](#revoking-tokens)
    - [清除令牌](#purging-tokens)
- [搭配 PKCE 的授權碼授予 (Authorization Code Grant With PKCE)](#code-grant-pkce)
    - [建立客戶端](#creating-a-auth-pkce-grant-client)
    - [請求令牌](#requesting-auth-pkce-grant-tokens)
- [裝置授權授予 (Device Authorization Grant)](#device-authorization-grant)
    - [建立裝置碼授予客戶端](#creating-a-device-authorization-grant-client)
    - [請求令牌](#requesting-device-authorization-grant-tokens)
- [密碼授予 (Password Grant)](#password-grant)
    - [建立密碼授予客戶端](#creating-a-password-grant-client)
    - [請求令牌](#requesting-password-grant-tokens)
    - [請求所有範圍](#requesting-all-scopes)
    - [自定義使用者提供者](#customizing-the-user-provider)
    - [自定義使用者名稱欄位](#customizing-the-username-field)
    - [自定義密碼驗證](#customizing-the-password-validation)
- [隱含授予 (Implicit Grant)](#implicit-grant)
- [客戶端憑證授予 (Client Credentials Grant)](#client-credentials-grant)
- [個人存取令牌 (Personal Access Tokens)](#personal-access-tokens)
    - [建立個人存取客戶端](#creating-a-personal-access-client)
    - [自定義使用者提供者](#customizing-the-user-provider-for-pat)
    - [管理個人存取令牌](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過中介層](#via-middleware)
    - [傳遞存取令牌](#passing-the-access-token)
- [令牌範圍 (Token Scopes)](#token-scopes)
    - [定義範圍](#defining-scopes)
    - [預設範圍](#default-scope)
    - [將範圍分配給令牌](#assigning-scopes-to-tokens)
    - [檢查範圍](#checking-scopes)
- [SPA 認證](#spa-authentication)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 讓您能在短短幾分鐘內為您的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 是基於由 Andy Millington 和 Simon Hamp 維護的 [League OAuth2 server](https://github.com/thephpleague/oauth2-server) 建構而成。

> [!NOTE]
> 本文件假設您已經熟悉 OAuth2。如果您對 OAuth2 一毫不知情，請在繼續之前先熟悉 OAuth2 的一般 [術語](https://oauth2.thephpleague.com/terminology/) 與功能。


<a name="passport-or-sanctum"></a>
### Passport 還是 Sanctum？

在開始之前，您可能想確定您的應用程式更適合使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum)。如果您的應用程式絕對需要支援 OAuth2，那麼您應該使用 Laravel Passport。

然而，如果您是嘗試對單頁應用程式 (SPA)、行動應用程式進行認證，或發行 API 令牌，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；但它提供了簡單得多的 API 認證開發體驗。


<a name="installation"></a>
## 安裝

您可以使用 `install:api` Artisan 命令來安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此命令將發布並執行必要的資料庫遷移，以建立應用程式儲存 OAuth2 客戶端與存取令牌所需的資料表。該命令還會建立用來產生安全存取令牌的加密金鑰。

執行 `install:api` 命令後，請將 `Laravel\Passport\HasApiTokens` trait 和 `Laravel\Passport\Contracts\OAuthenticatable` 介面新增至您的 `App\Models\User` 模型。此 trait 將為您的模型提供一些輔助方法，讓您可以檢查已認證使用者的令牌與範圍 (scopes)：

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

最後，在您應用程式的 `config/auth.php` 設定檔中，您應該定義一個 `api` 認證守衛 (authentication guard) 並將 `driver` 選項設定為 `passport`。這將指示您的應用程式在認證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

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

首次將 Passport 部署到應用程式伺服器時，您可能需要執行 `passport:keys` 命令。此命令會產生 Passport 產生存取令牌所需的加密金鑰。產生的金鑰通常不會保存在版本控制中：

```shell
php artisan passport:keys
```

如有必要，您可以定義 Passport 金鑰應載入的路徑。您可以使用 `Passport::loadKeysFrom` 方法來達成此目的。通常，此方法應在您應用程式的 `App\Providers\AppServiceProvider` 類別之 `boot` 方法中呼叫：

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

或者，您可以使用 `vendor:publish` Artisan 命令來發布 Passport 的設定檔：

```shell
php artisan vendor:publish --tag=passport-config
```

設定檔發布後，您可以透過將加密金鑰定義為環境變數來載入應用程式的加密金鑰：

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

升級到 Passport 的新主版本時，請務必仔細閱讀 [升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。


<a name="configuration"></a>
## 設定


<a name="token-lifetimes"></a>
### 令牌生命週期

預設情況下，Passport 會發行在一年後過期的長效存取令牌。如果您想設定更長或更短的令牌生命週期，可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應在您應用程式的 `App\Providers\AppServiceProvider` 類別之 `boot` 方法中呼叫：

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
> Passport 資料表上的 `expires_at` 欄位是唯讀的，僅用於顯示目的。在發行令牌時，Passport 會將過期資訊儲存在經過簽署與加密的令牌中。如果您需要使令牌失效，應該 [撤銷它](#revoking-tokens)。


<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以透過定義自己的模型並繼承對應的 Passport 模型，來擴展 Passport 內部使用的模型：

```php
use Laravel\Passport\Client as PassportClient;

class Client extends PassportClient
{
    // ...
}
```

定義模型後，您可以透過 `Laravel\Passport\Passport` 類別指示 Passport 使用您的自定義模型。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 您的自定義模型：

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

有時您可能希望自定義 Passport 定義的路由。為了實現這一點，您首先需要透過在應用程式 `AppServiceProvider` 的 `register` 方法中加入 `Passport::ignoreRoutes` 來忽略 Passport 註冊的路由：

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

接著，您可以將 Passport 在 [其路由檔案](https://github.com/laravel/passport/blob/master/routes/web.php) 中定義的路由複製到您應用程式的 `routes/web.php` 檔案中，並根據您的需求進行修改：

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
## 授權碼授予 (Authorization Code Grant)

大多數開發者對 OAuth2 最熟悉的便是透過授權碼來使用。當使用授權碼時，客戶端應用程式會將使用者重新導向至您的伺服器，使用者在那裡可以選擇批准或拒絕向該客戶端核發存取令牌的請求。

在開始之前，我們需要告知 Passport 如何回傳我們的「授權」視圖。

所有的授權視圖渲染邏輯都可以透過 `Laravel\Passport\Passport` 類別中提供的對應方法來自定義。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Passport 會自動定義一個回傳此視圖的 `/oauth/authorize` 路由。您的 `auth.oauth.authorize` 模板應包含一個向 `passport.authorizations.approve` 路由發送 POST 請求以批准授權的表單，以及一個向 `passport.authorizations.deny` 路由發送 DELETE 請求以拒絕授權的表單。`passport.authorizations.approve` 與 `passport.authorizations.deny` 路由預期接收 `state`、`client_id` 以及 `auth_token` 欄位。


<a name="managing-clients"></a>
### 管理客戶端

開發需要與您的應用程式 API 互動之應用程式的開發者，需要透過建立「客戶端」來將其應用程式在您的系統中進行註冊。通常，這包括提供其應用程式的名稱，以及一個在使用者批准授權請求後，您的應用程式可以重新導向至的 URI。


<a name="managing-first-party-clients"></a>
#### 第一方客戶端

建立客戶端最簡單的方法是使用 `passport:client` Artisan 命令。此命令可用於建立第一方客戶端或測試您的 OAuth2 功能。當您執行 `passport:client` 命令時，Passport 會提示您輸入客戶端的詳細資訊，並提供客戶端 ID 與私鑰 (secret)：

```shell
php artisan passport:client
```

如果您希望為客戶端允許多個重新導向 URI，可以在 `passport:client` 命令提示輸入 URI 時，使用逗號分隔的清單來指定。任何包含逗號的 URI 都應經過 URI 編碼：

```shell
https://third-party-app.com/callback,https://example.com/oauth/redirect
```


<a name="managing-third-party-clients"></a>
#### 第三方客戶端

由於您的應用程式使用者無法使用 `passport:client` 命令，您可以使用 `Laravel\Passport\ClientRepository` 類別的 `createAuthorizationCodeGrantClient` 方法為特定使用者註冊客戶端：

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

`createAuthorizationCodeGrantClient` 方法會回傳一個 `Laravel\Passport\Client` 實例。您可以將 `$client->id` 作為客戶端 ID，將 `$client->plainSecret` 作為客戶端私鑰顯示給使用者。

<a name="requesting-tokens"></a>
### 請求令牌


<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重新導向進行授權

一旦建立了客戶端，開發者可以使用其客戶端 ID 和私密金鑰 (secret) 向您的應用程式請求授權碼和存取令牌。首先，消費端應用程式應像這樣向您應用程式的 `/oauth/authorize` 路由發送重新導向請求：

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

可以使用 `prompt` 參數來指定 Passport 應用程式的認證行為。

如果 `prompt` 的值為 `none`，當使用者尚未在 Passport 應用程式中通過認證時，Passport 將始終拋出認證錯誤。如果值為 `consent`，即使先前已授予消費端應用程式所有範圍，Passport 仍會始終顯示授權批准畫面。當值為 `login` 時，Passport 應用程式將始終提示使用者重新登入應用程式，即使他們已經擁有現有的工作階段。

如果未提供 `prompt` 值，則僅在使用者先前未針對請求的範圍授予消費端應用程式存取權限時，才會提示其進行授權。

> [!NOTE]
> 請記得，`/oauth/authorize` 路由已由 Passport 定義。您不需要手動定義此路由。


<a name="approving-the-request"></a>
#### 批准請求

在接收授權請求時，Passport 會根據 `prompt` 參數的值（如果存在）自動回應，並可能向使用者顯示一個模板，允許他們批准或拒絕授權請求。如果他們批准了請求，他們將被重新導向回消費端應用程式所指定的 `redirect_uri`。`redirect_uri` 必須與建立客戶端時指定的 `redirect` URL 一致。

有時您可能希望跳過授權提示，例如在為第一方客戶端授權時。您可以透過 [擴充 `Client` 模型](#overriding-default-models) 並定義 `skipsAuthorization` 方法來實現此功能。如果 `skipsAuthorization` 回傳 `true`，該客戶端將被批准，且使用者將立即被重新導向回 `redirect_uri`，除非消費端應用程式在重新導向進行授權時明確設定了 `prompt` 參數：

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

如果使用者批准了授權請求，他們將被重新導向回消費端應用程式。消費端應首先根據重新導向前儲存的值驗證 `state` 參數。如果 state 參數匹配，則消費端應向您的應用程式發送 `POST` 請求以請求存取令牌。該請求應包含使用者批准授權請求時由您的應用程式發出的授權碼：

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

此 `/oauth/token` 路由將回傳一個包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取令牌過期前剩餘的秒數。

> [!NOTE]
> 如同 `/oauth/authorize` 路由，`/oauth/token` 路由也由 Passport 為您定義。無需手動定義此路由。


<a name="managing-tokens"></a>
### 管理令牌

您可以使用 `Laravel\Passport\HasApiTokens` trait 的 `tokens` 方法來獲取使用者的已授權令牌。例如，這可用於為您的使用者提供一個儀表板，以追蹤他們與第三方應用程式的連接情況：

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
### 重新整理令牌

如果您的應用程式發行短期存取令牌，使用者需要透過發行存取令牌時提供的重新整理令牌 (refresh token) 來重新整理其存取令牌：

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

此 `/oauth/token` 路由將回傳一個包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取令牌過期前剩餘的秒數。


<a name="revoking-tokens"></a>
### 撤銷令牌

您可以使用 `Laravel\Passport\Token` 模型上的 `revoke` 方法來撤銷令牌。您可以使用 `Laravel\Passport\RefreshToken` 模型上的 `revoke` 方法來撤銷令牌的重新整理令牌：

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

當令牌被撤銷或過期時，您可能想要將它們從資料庫中清除。Passport 內建的 `passport:purge` Artisan 指令可以為您完成此操作：

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

您也可以在應用程式的 `routes/console.php` 檔案中設定 [排程工作](/docs/{{version}}/scheduling)，以定期自動清除令牌：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

<a name="code-grant-pkce"></a>
## 搭配 PKCE 的授權碼授予 (Authorization Code Grant With PKCE)

搭配「程式碼交換證明金鑰」(Proof Key for Code Exchange, PKCE) 的授權碼授予是一種安全的認證方式，可用於單頁應用程式或行動應用程式以存取您的 API。當您無法保證客戶端秘密 (client secret) 能被機密地儲存，或是為了降低授權碼被攻擊者攔截的風險時，應使用此授予方式。在將授權碼兌換為存取令牌時，會使用「程式碼驗證碼」(code verifier) 與「程式碼挑戰碼」(code challenge) 的組合來取代客戶端秘密。


<a name="creating-a-auth-pkce-grant-client"></a>
### 建立客戶端

在您的應用程式能透過搭配 PKCE 的授權碼授予來發行令牌之前，您需要建立一個啟用 PKCE 的客戶端。您可以使用 `passport:client` Artisan 主控台命令並加上 `--public` 選項來完成此操作：

```shell
php artisan passport:client --public
```


<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求令牌


<a name="code-verifier-code-challenge"></a>
#### 程式碼驗證碼與程式碼挑戰碼

由於此授權授予不提供客戶端秘密，開發者需要產生一組程式碼驗證碼與程式碼挑戰碼的組合才能請求令牌。

根據 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636) 的定義，程式碼驗證碼應該是一個 43 到 128 個字元之間的隨機字串，包含字母、數字以及 `"-"`、`"."`、`"_"`、`"~"` 字元。

程式碼挑戰碼應該是一個經 Base64 編碼的字串，且包含適用於 URL 和檔名的安全字元。應移除末尾的 `'='` 字元，且不得包含換行符號、空白或其他額外字元。

```php
$encoded = base64_encode(hash('sha256', $codeVerifier, true));

$codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');
```


<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

客戶端建立完成後，您可以使用客戶端 ID 以及產生的程式碼驗證碼與程式碼挑戰碼，向您的應用程式請求授權碼和存取令牌。首先，消費端應用程式應向您的應用程式 `/oauth/authorize` 路由發出重新導向請求：

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

如果使用者核准了授權請求，他們將被重新導向回消費端應用程式。消費端應像在標準的授權碼授予中一樣，將 `state` 參數與重新導向前儲存的值進行驗證。

如果 state 參數匹配，消費端應向您的應用程式發出 `POST` 請求以請求存取令牌。該請求應包含使用者核准授權請求時由您的應用程式所發行的授權碼，以及最初產生的程式碼驗證碼：

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
## 裝置授權授予 (Device Authorization Grant)

OAuth2 裝置授權授予允許沒有瀏覽器或輸入受限的裝置（例如電視和遊戲主機），透過交換「裝置碼 (device code)」來獲取存取令牌。當使用裝置流程 (device flow) 時，裝置客戶端會指示使用者使用第二台裝置（例如電腦或智慧型手機）連接到您的伺服器，並在該處輸入提供的「使用者碼 (user code)」以核准或拒絕存取請求。

在開始之前，我們需要指示 Passport 如何回傳我們的「使用者碼」與「授權」視圖。

所有授權視圖的渲染邏輯都可以使用 `Laravel\Passport\Passport` 類別中提供的對應方法來自定義。通常，您應該在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法。

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

Passport 會自動定義回傳這些視圖的路由。您的 `auth.oauth.device.user-code` 模板應包含一個表單，該表單會向 `passport.device.authorizations.authorize` 路由發送 GET 請求。`passport.device.authorizations.authorize` 路由需要一個 `user_code` 查詢參數。

您的 `auth.oauth.device.authorize` 模板應包含一個向 `passport.device.authorizations.approve` 路由發送 POST 請求以核准授權的表單，以及一個向 `passport.device.authorizations.deny` 路由發送 DELETE 請求以拒絕授權的表單。`passport.device.authorizations.approve` 與 `passport.device.authorizations.deny` 路由需要 `state`、`client_id` 和 `auth_token` 欄位。


<a name="creating-a-device-authorization-grant-client"></a>
### 建立裝置碼授予客戶端

在您的應用程式能透過裝置授權授予發行令牌之前，您需要建立一個啟用了裝置流程的客戶端。您可以使用 `passport:client` Artisan 指令搭配 `--device` 選項來完成此操作。此指令將建立一個啟用了裝置流程的第一方客戶端，並提供客戶端 ID 與私鑰 (secret)：

```shell
php artisan passport:client --device
```

此外，您可以使用 `ClientRepository` 類別的 `createDeviceAuthorizationGrantClient` 方法來為指定的使用者註冊第三方客戶端：

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

客戶端建立後，開發者可以使用其客戶端 ID 向您的應用程式請求裝置碼。首先，使用端裝置應向您的應用程式 `/oauth/device/code` 路由發送 `POST` 請求以請求裝置碼：

```php
use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

這將回傳一個包含 `device_code`、`user_code`、`verification_uri`、`interval` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含裝置碼過期前的秒數。`interval` 屬性包含使用端裝置在輪詢 `/oauth/token` 路由時，兩次請求之間應等待的秒數，以避免觸發速率限制錯誤。

> [!NOTE]
> 記得，`/oauth/device/code` 路由已經由 Passport 定義，您不需要手動定義此路由。


<a name="user-code"></a>
#### 顯示驗證 URI 與使用者碼

獲取裝置碼請求後，使用端裝置應指示使用者使用另一台裝置訪問提供的 `verification_uri` 並輸入 `user_code` 以核准授權請求。


<a name="polling-token-request"></a>
#### 輪詢令牌請求

由於使用者將使用另一台裝置來授予（或拒絕）存取權限，使用端裝置應輪詢您的應用程式 `/oauth/token` 路由，以確定使用者何時對請求做出回應。使用端裝置應使用請求裝置碼時 JSON 回應中提供的最小輪詢 `interval`，以避免觸發速率限制錯誤：

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

如果使用者核准了授權請求，這將回傳一個包含 `access_token`、`refresh_token` 和 `expires_in` 屬性的 JSON 回應。`expires_in` 屬性包含存取令牌過期前的秒數。

<a name="password-grant"></a>
## 密碼授予 (Password Grant)

> [!WARNING]
> 我們不再建議使用密碼授予令牌。相反地，您應該選擇 [目前由 OAuth2 Server 建議的授予類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密碼授予允許您的其他第一方客戶端（例如行動應用程式），使用電子郵件地址/使用者名稱和密碼來獲取存取令牌。這讓您可以安全地向第一方客戶端發行存取令牌，而不需要使用者經過完整的 OAuth2 授權碼重新導向流程。

要啟用密碼授予，請在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

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

在您的應用程式能透過密碼授予發行令牌之前，您需要建立一個密碼授予客戶端。您可以使用 `passport:client` Artisan 指令並加上 `--password` 選項來完成。

```shell
php artisan passport:client --password
```


<a name="requesting-password-grant-tokens"></a>
### 請求令牌

一旦您啟用了授予並建立了密碼授予客戶端，您就可以透過向 `/oauth/token` 路由發送 `POST` 請求，並提供使用者的電子郵件地址和密碼來請求存取令牌。請記住，此路由已由 Passport 註冊，因此不需要手動定義。如果請求成功，您將在伺服器的 JSON 回應中收到 `access_token` 和 `refresh_token`：

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
> 請記住，存取令牌預設為長效的。不過，如果需要，您可以自由地 [設定您的最大存取令牌生命週期](#configuration)。


<a name="requesting-all-scopes"></a>
### 請求所有範圍

當使用密碼授予或客戶端憑證授予時，您可能希望為令牌授權應用程式所支援的所有範圍。您可以透過請求 `*` 範圍來達成。如果您請求 `*` 範圍，令牌實例上的 `can` 方法將始終回傳 `true`。此範圍只能分配給使用 `password` 或 `client_credentials` 授予而發行的令牌：

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
### 自定義使用者提供者

如果您的應用程式使用了超過一個 [認證使用者提供者](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --password` 指令建立客戶端時提供 `--provider` 選項，來指定密碼授予客戶端要使用的使用者提供者。提供的提供者名稱應與應用程式 `config/auth.php` 設定檔中定義的有效提供者相符。接著您可以 [使用中介層保護您的路由](#multiple-authentication-guards)，以確保只有來自該保護機制指定提供者的使用者被授權。


<a name="customizing-the-username-field"></a>
### 自定義使用者名稱欄位

在使用密碼授予進行認證時，Passport 會將認證模型 (authenticatable model) 的 `email` 屬性作為「使用者名稱」。不過，您可以在模型上定義 `findForPassport` 方法來自定義此行為：

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
### 自定義密碼驗證

在使用密碼授予進行認證時，Passport 會使用模型的 `password` 屬性來驗證給定的密碼。如果您的模型沒有 `password` 屬性，或者您希望自定義密碼驗證邏輯，您可以在模型上定義 `validateForPassportPasswordGrant` 方法：

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
## 隱含授予 (Implicit Grant)

> [!WARNING]
> 我們不再建議使用隱含授予令牌。相反地，您應該選擇 [目前由 OAuth2 Server 建議的授予類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隱含授予與授權碼授予類似；但令牌會直接回傳給客戶端，而不需要交換授權碼。此授予最常用於 JavaScript 或行動應用程式，因為這些客戶端的憑證無法被安全地儲存。要啟用此授予，請在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

在您的應用程式能透過隱含授予發行令牌之前，您需要建立一個隱含授予客戶端。您可以使用 `passport:client` Artisan 指令並加上 `--implicit` 選項來完成。

```shell
php artisan passport:client --implicit
```

一旦啟用了授予並建立了隱含客戶端，開發者可以使用其客戶端 ID 向您的應用程式請求存取令牌。使用端應用程式應向您的應用程式 `/oauth/authorize` 路由發送重新導向請求，如下所示：

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
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您不需要手動定義此路由。

<a name="client-credentials-grant"></a>
## 客戶端憑證授予 (Client Credentials Grant)

客戶端憑證授予適用於機器對機器的認證。例如，您可以在透過 API 執行維護任務的排程作業中使用此授予方式。

在您的應用程式能透過客戶端憑證授予來發行令牌之前，您需要先建立一個客戶端憑證授予客戶端。您可以使用 `passport:client` Artisan 命令的 `--client` 選項來完成此操作：

```shell
php artisan passport:client --client
```

接下來，將 `Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層分配給路由：

```php
use Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner;

Route::get('/orders', function (Request $request) {
    // Access token is valid and the client is resource owner...
})->middleware(EnsureClientIsResourceOwner::class);
```

若要將路由的存取限制在特定的範圍內，您可以將所需範圍的列表提供給 `using` 方法：

```php
Route::get('/orders', function (Request $request) {
    // Access token is valid, the client is resource owner, and has both "servers:read" and "servers:create" scopes...
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```

> [!WARNING]
> 底層的 [OAuth2 伺服器](https://oauth2.thephpleague.com/database-setup/#:~:text=Please%20note%20that,the%20bearer%20token.) 將客戶端憑證令牌的 `sub` 宣告 (claim) 設定為客戶端的識別碼。預設情況下，Passport 為客戶端使用 UUID，因此這不會與使用整數主鍵的使用者發生衝突。然而，如果您將 `Passport::$clientUuids` 設定為 `false`，客戶端憑證令牌可能會不小心解析到 ID 與客戶端 ID 相同的使用者。在這種情況下，使用此中介層無法保證傳入的令牌是客戶端憑證令牌。


<a name="retrieving-tokens"></a>
### 請求令牌

若要使用此授予類型來取得令牌，請向 `oauth/token` 端點發送請求：

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

有時候，您的使用者可能希望在不經過典型的授權碼重新導向流程的情況下，為自己發行存取令牌。允許使用者透過您應用程式的使用者介面 (UI) 為自己發行令牌，可用於讓使用者嘗試您的 API，或者作為一種更簡單的發行存取令牌的方法。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 來發行個人存取令牌，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 提供的一個輕量級第一方庫，用於發行 API 存取令牌。


<a name="creating-a-personal-access-client"></a>
### 建立個人存取客戶端

在您的應用程式能發行個人存取令牌之前，您需要先建立一個個人存取客戶端。您可以使用 `passport:client` Artisan 命令並加上 `--personal` 選項來完成此操作。如果您已經執行過 `passport:install` 命令，則不需要執行此命令：

```shell
php artisan passport:client --personal
```


<a name="customizing-the-user-provider-for-pat"></a>
### 自定義使用者提供者

如果您的應用程式使用了多個 [認證使用者提供者](/docs/{{version}}/authentication#introduction)，您可以在使用 `artisan passport:client --personal` 命令建立客戶端時，透過提供 `--provider` 選項來指定個人存取授予客戶端要使用的使用者提供者。所提供的提供者名稱必須與您應用程式的 `config/auth.php` 設定檔中定義的有效提供者相符。接著您可以 [使用中介層來保護路由](#multiple-authentication-guards)，以確保只有來自該守衛所指定提供者的使用者才能獲得授權。


<a name="managing-personal-access-tokens"></a>
### 管理個人存取令牌

建立個人存取客戶端後，您可以使用 `App\Models\User` 模型實例上的 `createToken` 方法為特定使用者發行令牌。`createToken` 方法的第一個引數是令牌的名稱，第二個引數是一個可選的 [範圍](#token-scopes) 陣列：

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

Passport 包含了一個 [認證守衛](/docs/{{version}}/authentication#adding-custom-guards)，可用於驗證傳入請求中的存取令牌。一旦您將 `api` 守衛設定為使用 `passport` 驅動程式，您只需要在任何需要有效存取令牌的路由上指定 `auth:api` 中介層即可：

```php
Route::get('/user', function () {
    // Only API authenticated users may access this route...
})->middleware('auth:api');
```

> [!WARNING]
> 如果您使用的是 [客戶端憑證授予](#client-credentials-grant)，您應該使用 [`Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner` 中介層](#client-credentials-grant) 來保護您的路由，而非使用 `auth:api` 中介層。


<a name="multiple-authentication-guards"></a>
#### 多重認證守衛

如果您的應用程式需要對不同類型的使用者進行認證，而這些使用者可能使用完全不同的 Eloquent 模型，您可能需要在應用程式中為每種使用者提供者類型定義守衛設定。這讓您能夠保護針對特定使用者提供者的請求。例如，在 `config/auth.php` 設定檔中使用以下守衛設定：

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

以下路由將利用 `api-customers` 守衛（該守衛使用 `customers` 使用者提供者）來認證傳入的請求：

```php
Route::get('/customer', function () {
    // ...
})->middleware('auth:api-customers');
```

> [!NOTE]
> 如需更多關於在 Passport 中使用多個使用者提供者的資訊，請參閱 [個人存取令牌文件](#customizing-the-user-provider-for-pat) 與 [密碼授予文件](#customizing-the-user-provider)。


<a name="passing-the-access-token"></a>
### 傳遞存取令牌

在呼叫受 Passport 保護的路由時，您應用程式的 API 使用者應在請求的 `Authorization` 標頭中將其存取令牌指定為 `Bearer` 令牌。例如，使用 `Http` Facade 時：

```php
use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

<a name="token-scopes"></a>
## 令牌範圍 (Token Scopes)

範圍 (Scopes) 允許您的 API 客戶端在請求存取帳戶的授權時，請求一組特定的權限。例如，如果您正在建立一個電子商務應用程式，並非所有 API 消費者都需要下單的能力。相反地，您可以允許消費者僅請求存取訂單出貨狀態的授權。換句話說，範圍允許您的應用程式使用者限制第三方應用程式代表他們執行的動作。


<a name="defining-scopes"></a>
### 定義範圍

您可以使用 `App\Providers\AppServiceProvider` 類別之 `boot` 方法中的 `Passport::tokensCan` 方法來定義 API 的範圍。`tokensCan` 方法接收一個包含範圍名稱與範圍描述的陣列。範圍描述可以是任何您想要的內容，並將在授權核准畫面中顯示給使用者：

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

如果客戶端沒有請求任何特定的範圍，您可以使用 `defaultScopes` 方法將您的 Passport 伺服器設定為將預設範圍附加到令牌中。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別之 `boot` 方法中呼叫此方法：

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
### 將範圍分配給令牌


<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

在使用授權碼授予 (authorization code grant) 請求存取令牌時，消費者應將所需的範圍指定為 `scope` 查詢字串參數。`scope` 參數應為以空格分隔的範圍列表：

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
#### 發行個人存取令牌時

如果您使用 `App\Models\User` 模型的 `createToken` 方法來發行個人存取令牌，您可以將所需範圍的陣列作為該方法的第二個引數傳入：

```php
$token = $user->createToken('My Token', ['orders:create'])->accessToken;
```


<a name="checking-scopes"></a>
### 檢查範圍

Passport 包含兩個中介層，可用於驗證傳入的請求是否使用已授予特定範圍的令牌進行認證。


<a name="check-for-all-scopes"></a>
#### 檢查所有範圍

`Laravel\Passport\Http\Middleware\CheckToken` 中介層可以分配給路由，以驗證傳入請求的存取令牌是否具有所有列出的範圍：

```php
use Laravel\Passport\Http\Middleware\CheckToken;

Route::get('/orders', function () {
    // Access token has both "orders:read" and "orders:create" scopes...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);
```


<a name="check-for-any-scopes"></a>
#### 檢查任一範圍

`Laravel\Passport\Http\Middleware\CheckTokenForAnyScope` 中介層可以分配給路由，以驗證傳入請求的存取令牌是否具有列出範圍中的*至少一個*：

```php
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

Route::get('/orders', function () {
    // Access token has either "orders:read" or "orders:create" scope...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```


<a name="scope-attributes"></a>
#### 範圍屬性

如果您的應用程式使用 [控制器中介層屬性](/docs/{{version}}/controllers#middleware-attributes)，您可以使用 `Laravel\Passport\Attributes\AuthorizeToken` 屬性作為 Passport 範圍中介層的便捷快捷方式：

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

預設情況下，`AuthorizeToken` 屬性要求所有給定的範圍。如果您傳遞 `anyScope: true`，則當令牌具有給定範圍中的至少一個時，請求將被授權。


<a name="checking-scopes-on-a-token-instance"></a>
#### 在令牌實例上檢查範圍

一旦經過存取令牌認證的請求進入您的應用程式，您仍然可以使用已認證的 `App\Models\User` 實例上的 `tokenCan` 方法來檢查令牌是否具有給定範圍：

```php
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('orders:create')) {
        // ...
    }
});
```


<a name="additional-scope-methods"></a>
#### 其他範圍方法

`scopeIds` 方法將回傳所有定義的 ID/名稱陣列：

```php
use Laravel\Passport\Passport;

Passport::scopeIds();
```

`scopes` 方法將回傳所有定義的範圍陣列，且每個元素皆為 `Laravel\Passport\Scope` 實例：

```php
Passport::scopes();
```

`scopesFor` 方法將回傳與給定 ID/名稱相匹配的 `Laravel\Passport\Scope` 實例陣列：

```php
Passport::scopesFor(['user:read', 'orders:create']);
```

您可以使用 `hasScope` 方法來判斷特定的範圍是否已定義：

```php
Passport::hasScope('orders:create');
```

<a name="spa-authentication"></a>
## SPA 認證

在建立 API 時，能夠從您的 JavaScript 應用程式中使用您自己的 API 會非常有用。這種 API 開發方法允許您自己的應用程式使用與您分享給世界的相同 API。同樣的 API 可以被您的網頁應用程式、行動應用程式、第三方應用程式以及您在各種套件管理工具上發布的任何 SDK 所使用。

通常，如果您想從 JavaScript 應用程式中使用您的 API，您需要手動將存取令牌發送到應用程式，並在每次向應用程式發出請求時傳遞該令牌。然而，Passport 包含了一個可以為您處理此問題的中介層。您只需要將 `CreateFreshApiToken` 中介層附加到應用程式 `bootstrap/app.php` 檔案中的 `web` 中介層群組即可：

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

此中介層會在您的回傳回應中附加一個 `laravel_token` Cookie。此 Cookie 包含一個加密的 JWT，Passport 將使用它來認證來自您 JavaScript 應用程式的 API 請求。該 JWT 的生命週期與您的 `session.lifetime` 設定值相同。現在，由於瀏覽器會自動在所有後續請求中發送此 Cookie，因此您可以向應用程式的 API 發出請求，而無需明確傳遞存取令牌：

```js
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```


<a name="customizing-the-cookie-name"></a>
#### 自定義 Cookie 名稱

如果需要，您可以使用 `Passport::cookie` 方法來自定義 `laravel_token` Cookie 的名稱。通常，此方法應在應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

當使用這種認證方法時，您需要確保請求中包含有效的 CSRF 令牌標頭。骨架應用程式和所有入門套件中包含的預設 Laravel JavaScript 基礎結構包含了一個 [Axios](https://github.com/axios/axios) 實例，它會自動使用加密的 `XSRF-TOKEN` Cookie 值在同源請求中發送 `X-XSRF-TOKEN` 標頭。

> [!NOTE]
> 如果您選擇發送 `X-CSRF-TOKEN` 標頭而不是 `X-XSRF-TOKEN`，您需要使用由 `csrf_token()` 提供的未加密令牌。


<a name="events"></a>
## 事件

Passport 在發行存取令牌和重新整理令牌時會觸發事件。您可以[監聽這些事件](/docs/{{version}}/events)以修剪或撤銷資料庫中的其他存取令牌：

<div class="overflow-auto">

| 事件名稱                                    |
| --------------------------------------------- |
| `Laravel\Passport\Events\AccessTokenCreated`  |
| `Laravel\Passport\Events\AccessTokenRevoked`  |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>


<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用於指定目前經過認證的使用者及其範圍。傳遞給 `actingAs` 方法的第一個引數是使用者實例，第二個是應授予使用者令牌的範圍陣列：

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

Passport 的 `actingAsClient` 方法可用於指定目前經過認證的客戶端及其範圍。傳遞給 `actingAsClient` 方法的第一個引數是客戶端實例，第二個是應授予客戶端令牌的範圍陣列：

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