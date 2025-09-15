# Laravel Passport

- [簡介](#introduction)
    - [Passport 還是 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [Client Secret 雜湊](#client-secret-hashing)
    - [Token 存留時間](#token-lifetimes)
    - [覆寫預設模型](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [發行 Access Token](#issuing-access-tokens)
    - [管理 Client](#managing-clients)
    - [請求 Token](#requesting-tokens)
    - [更新 Token](#refreshing-tokens)
    - [撤銷 Token](#revoking-tokens)
    - [清除 Token](#purging-tokens)
- [帶有 PKCE 的授權碼授權](#code-grant-pkce)
    - [建立 Client](#creating-a-auth-pkce-grant-client)
    - [請求 Token](#requesting-auth-pkce-grant-tokens)
- [密碼授權 Token](#password-grant-tokens)
    - [建立密碼授權 Client](#creating-a-password-grant-client)
    - [請求 Token](#requesting-password-grant-tokens)
    - [請求所有 Scope](#requesting-all-scopes)
    - [自訂使用者 Provider](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱含授權 Token](#implicit-grant-tokens)
- [Client Credentials 授權 Token](#client-credentials-grant-tokens)
- [個人 Access Token](#personal-access-tokens)
    - [建立個人 Access Client](#creating-a-personal-access-client)
    - [管理個人 Access Token](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過 Middleware](#via-middleware)
    - [傳遞 Access Token](#passing-the-access-token)
- [Token Scope](#token-scopes)
    - [定義 Scope](#defining-scopes)
    - [預設 Scope](#default-scope)
    - [為 Token 指派 Scope](#assigning-scopes-to-tokens)
    - [檢查 Scope](#checking-scopes)
- [使用 JavaScript 存取您的 API](#consuming-your-api-with-javascript)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 可以在幾分鐘內為您的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 建構於由 Andy Millington 和 Simon Hamp 維護的 [League OAuth2 伺服器](https://github.com/thephpleague/oauth2-server)之上。

> [!WARNING]
> 本文件假設您已熟悉 OAuth2。如果您對 OAuth2 一無所知，建議您在繼續之前先熟悉 OAuth2 的一般[術語](https://oauth2.thephpleague.com/terminology/)和功能。

<a name="passport-or-sanctum"></a>
### Passport 還是 Sanctum？

在開始之前，您可能需要判斷您的應用程式更適合使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum)。如果您的應用程式絕對需要支援 OAuth2，那麼您應該使用 Laravel Passport。

然而，如果您嘗試驗證單頁應用程式、行動應用程式或發行 API Token，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；但是，它提供了更簡單的 API 驗證開發體驗。

<a name="installation"></a>
## 安裝

您可以透過 `install:api` Artisan 命令安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此命令將發布並執行資料庫遷移，以建立您的應用程式儲存 OAuth2 Client 和 Access Token 所需的資料表。該命令還將建立產生安全 Access Token 所需的加密金鑰。

此外，此命令將詢問您是否要使用 UUID 作為 Passport `Client` 模型的主鍵值，而不是自動遞增的整數。

執行 `install:api` 命令後，將 `Laravel\Passport\HasApiTokens` Trait 加入到您的 `App\Models\User` 模型中。此 Trait 將為您的模型提供一些輔助方法，讓您可以檢查已驗證使用者的 Token 和 Scope：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Factories\HasFactory;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
    {
        use HasApiTokens, HasFactory, Notifiable;
    }

最後，在您的應用程式 `config/auth.php` 設定檔中，您應該定義一個 `api` 驗證 Guard，並將 `driver` 選項設定為 `passport`。這將指示您的應用程式在驗證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

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

<a name="deploying-passport"></a>
### 部署 Passport

首次將 Passport 部署到您的應用程式伺服器時，您可能需要執行 `passport:keys` 命令。此命令會產生 Passport 產生 Access Token 所需的加密金鑰。產生的金鑰通常不會保留在原始碼控制中：

```shell
php artisan passport:keys
```

如有必要，您可以定義 Passport 金鑰的載入路徑。您可以使用 `Passport::loadKeysFrom` 方法來完成此操作。通常，此方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
    }

<a name="loading-keys-from-the-environment"></a>
#### 從環境變數載入金鑰

或者，您可以使用 `vendor:publish` Artisan 命令發布 Passport 的設定檔：

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

升級到 Passport 的新主要版本時，務必仔細查閱[升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md)。

<a name="configuration"></a>
## 設定

<a name="client-secret-hashing"></a>
### Client Secret 雜湊

如果您希望 Client 的 Secret 在儲存到資料庫時進行雜湊處理，您應該在您的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `Passport::hashClientSecrets` 方法：

    use Laravel\Passport\Passport;

    Passport::hashClientSecrets();

啟用後，所有 Client Secret 將僅在建立後立即顯示給使用者。由於純文字的 Client Secret 值從未儲存在資料庫中，因此如果遺失，則無法復原 Secret 的值。

<a name="token-lifetimes"></a>
### Token 存留時間

預設情況下，Passport 會發行一年後過期的長效 Access Token。如果您想設定更長或更短的 Token 存留時間，您可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::tokensExpireIn(now()->addDays(15));
        Passport::refreshTokensExpireIn(now()->addDays(30));
        Passport::personalAccessTokensExpireIn(now()->addMonths(6));
    }

> [!WARNING]
> Passport 資料庫資料表上的 `expires_at` 欄位是唯讀的，僅供顯示之用。發行 Token 時，Passport 會將過期資訊儲存在已簽署和加密的 Token 中。如果您需要使 Token 失效，您應該[撤銷它](#revoking-tokens)。

<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以自由地透過定義自己的模型並擴展相應的 Passport 模型來擴展 Passport 內部使用的模型：

    use Laravel\Passport\Client as PassportClient;

    class Client extends PassportClient
    {
        // ...
    }

定義模型後，您可以透過 `Laravel\Passport\Passport` 類別指示 Passport 使用您的自訂模型。通常，您應該在您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中通知 Passport 您的自訂模型：

    use App\Models\Passport\AuthCode;
    use App\Models\Passport\Client;
    use App\Models\Passport\PersonalAccessClient;
    use App\Models\Passport\RefreshToken;
    use App\Models\Passport\Token;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::useTokenModel(Token::class);
        Passport::useRefreshTokenModel(RefreshToken::class);
        Passport::useAuthCodeModel(AuthCode::class);
        Passport::useClientModel(Client::class);
        Passport::usePersonalAccessClientModel(PersonalAccessClient::class);
    }

<a name="overriding-routes"></a>
### 覆寫路由

有時您可能希望自訂 Passport 定義的路由。為此，您首先需要透過將 `Passport::ignoreRoutes` 加入到您的應用程式 `AppServiceProvider` 的 `register` 方法中來忽略 Passport 註冊的路由：

    use Laravel\Passport\Passport;

    /**
     * Register any application services.
     */
    public function register(): void
    {
        Passport::ignoreRoutes();
    }

然後，您可以將 Passport 在[其路由檔案](https://github.com/laravel/passport/blob/11.x/routes/web.php)中定義的路由複製到您的應用程式 `routes/web.php` 檔案中，並根據您的喜好進行修改：

    Route::group([
        'as' => 'passport.',
        'prefix' => config('passport.path', 'oauth'),
        'namespace' => '\Laravel\Passport\Http\Controllers',
    ], function () {
        // Passport routes...
    });

<a name="issuing-access-tokens"></a>
## 發行 Access Token

透過授權碼使用 OAuth2 是大多數開發人員熟悉 OAuth2 的方式。使用授權碼時，Client 應用程式會將使用者重新導向到您的伺服器，使用者將在此處批准或拒絕向 Client 發行 Access Token 的請求。

<a name="managing-clients"></a>
### 管理 Client

首先，需要與您的應用程式 API 互動的開發人員需要透過建立「Client」來向您的應用程式註冊其應用程式。通常，這包括提供其應用程式的名稱以及使用者批准其授權請求後您的應用程式可以重新導向到的 URL。

<a name="the-passportclient-command"></a>
#### `passport:client` 命令

建立 Client 最簡單的方法是使用 `passport:client` Artisan 命令。此命令可用於建立您自己的 Client 以測試您的 OAuth2 功能。當您執行 `client` 命令時，Passport 將提示您輸入有關 Client 的更多資訊，並為您提供 Client ID 和 Secret：

```shell
php artisan passport:client
```

**重新導向 URL**

如果您想為 Client 允許多個重新導向 URL，您可以在 `passport:client` 命令提示輸入 URL 時，使用逗號分隔清單指定它們。任何包含逗號的 URL 都應進行 URL 編碼：

```shell
http://example.com/callback,http://examplefoo.com/callback
```

<a name="clients-json-api"></a>
#### JSON API

由於您的應用程式使用者無法使用 `client` 命令，Passport 提供了一個 JSON API，您可以使用它來建立 Client。這省去了您手動編寫用於建立、更新和刪除 Client 的 Controller 的麻煩。

但是，您需要將 Passport 的 JSON API 與您自己的前端配對，以提供一個儀表板供使用者管理其 Client。下面，我們將回顧所有用於管理 Client 的 API 端點。為方便起見，我們將使用 [Axios](https://github.com/axios/axios) 來演示向端點發出 HTTP 請求。

JSON API 受 `web` 和 `auth` Middleware 保護；因此，它只能從您自己的應用程式中呼叫。它無法從外部來源呼叫。

<a name="get-oauthclients"></a>
#### `GET /oauth/clients`

此路由會傳回已驗證使用者的所有 Client。這主要用於列出使用者的所有 Client，以便他們可以編輯或刪除它們：

```js
axios.get('/oauth/clients')
    .then(response => {
        console.log(response.data);
    });
```

<a name="post-oauthclients"></a>
#### `POST /oauth/clients`

此路由用於建立新的 Client。它需要兩部分資料：Client 的 `name` 和 `redirect` URL。`redirect` URL 是使用者在批准或拒絕授權請求後將被重新導向到的位置。

建立 Client 後，將發行 Client ID 和 Client Secret。這些值將在從您的應用程式請求 Access Token 時使用。Client 建立路由將傳回新的 Client 實例：

```js
const data = {
    name: 'Client Name',
    redirect: 'http://example.com/callback'
};

axios.post('/oauth/clients', data)
    .then(response => {
        console.log(response.data);
    })
    .catch (response => {
        // List errors on response...
    });
```

<a name="put-oauthclientsclient-id"></a>
#### `PUT /oauth/clients/{client-id}`

此路由用於更新 Client。它需要兩部分資料：Client 的 `name` 和 `redirect` URL。`redirect` URL 是使用者在批准或拒絕授權請求後將被重新導向到的位置。此路由將傳回更新後的 Client 實例：

```js
const data = {
    name: 'New Client Name',
    redirect: 'http://example.com/callback'
};

axios.put('/oauth/clients/' + clientId, data)
    .then(response => {
        console.log(response.data);
    })
    .catch (response => {
        // List errors on response...
    });
```

<a name="delete-oauthclientsclient-id"></a>
#### `DELETE /oauth/clients/{client-id}`

此路由用於刪除 Client：

```js
axios.delete('/oauth/clients/' + clientId)
    .then(response => {
        // ...
    });
```

<a name="requesting-tokens"></a>
### 請求 Token

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重新導向以進行授權

Client 建立後，開發人員可以使用其 Client ID 和 Secret 從您的應用程式請求授權碼和 Access Token。首先，消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'code',
            'scope' => '',
            'state' => $state,
            // 'prompt' => '', // "none", "consent", or "login"
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });

`prompt` 參數可用於指定 Passport 應用程式的驗證行為。

如果 `prompt` 值為 `none`，如果使用者尚未透過 Passport 應用程式進行驗證，Passport 將始終拋出驗證錯誤。如果值為 `consent`，Passport 將始終顯示授權批准畫面，即使所有 Scope 之前都已授予消費應用程式。當值為 `login` 時，Passport 應用程式將始終提示使用者重新登入應用程式，即使他們已經有現有的 Session。

如果未提供 `prompt` 值，則僅當使用者之前未授權存取消費應用程式的請求 Scope 時，才會提示使用者進行授權。

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="approving-the-request"></a>
#### 批准請求

收到授權請求時，Passport 將根據 `prompt` 參數的值（如果存在）自動回應，並可能向使用者顯示一個範本，允許他們批准或拒絕授權請求。如果他們批准請求，他們將被重新導向回消費應用程式指定的 `redirect_uri`。`redirect_uri` 必須與建立 Client 時指定的 `redirect` URL 相符。

如果您想自訂授權批准畫面，您可以使用 `vendor:publish` Artisan 命令發布 Passport 的視圖。發布的視圖將放置在 `resources/views/vendor/passport` 目錄中：

```shell
php artisan vendor:publish --tag=passport-views
```

有時您可能希望跳過授權提示，例如在授權第一方 Client 時。您可以透過[擴展 `Client` 模型](#overriding-default-models)並定義 `skipsAuthorization` 方法來實現此目的。如果 `skipsAuthorization` 傳回 `true`，Client 將被批准，使用者將立即被重新導向回 `redirect_uri`，除非消費應用程式在重新導向以進行授權時明確設定了 `prompt` 參數：

    <?php

    namespace App\Models\Passport;

    use Laravel\Passport\Client as BaseClient;

    class Client extends BaseClient
    {
        /**
         * Determine if the client should skip the authorization prompt.
         */
        public function skipsAuthorization(): bool
        {
            return $this->firstParty();
        }
    }

<a name="requesting-tokens-converting-authorization-codes-to-access-tokens"></a>
#### 將授權碼轉換為 Access Token

如果使用者批准了授權請求，他們將被重新導向回消費應用程式。消費者應首先根據重新導向之前儲存的值驗證 `state` 參數。如果 `state` 參數匹配，則消費者應向您的應用程式發出 `POST` 請求以請求 Access Token。請求應包含您的應用程式在使用者批准授權請求時發行的授權碼：

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Http;

    Route::get('/callback', function (Request $request) {
        $state = $request->session()->pull('state');

        throw_unless(
            strlen($state) > 0 && $state === $request->state,
            InvalidArgumentException::class,
            'Invalid state value.'
        );

        $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'client_secret' => 'client-secret',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'code' => $request->code,
        ]);

        return $response->json();
    });

此 `/oauth/token` 路由將傳回一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含 Access Token 過期前的秒數。

> [!NOTE]
> 與 `/oauth/authorize` 路由一樣，`/oauth/token` 路由由 Passport 為您定義。無需手動定義此路由。

<a name="tokens-json-api"></a>
#### JSON API

Passport 還包含一個用於管理已授權 Access Token 的 JSON API。您可以將其與您自己的前端配對，為您的使用者提供一個用於管理 Access Token 的儀表板。為方便起見，我們將使用 [Axios](https://github.com/axios/axios) 來演示向端點發出 HTTP 請求。JSON API 受 `web` 和 `auth` Middleware 保護；因此，它只能從您自己的應用程式中呼叫。

<a name="get-oauthtokens"></a>
#### `GET /oauth/tokens`

此路由會傳回已驗證使用者建立的所有已授權 Access Token。這主要用於列出使用者的所有 Token，以便他們可以撤銷它們：

```js
axios.get('/oauth/tokens')
    .then(response => {
        console.log(response.data);
    });
```

<a name="delete-oauthtokenstoken-id"></a>
#### `DELETE /oauth/tokens/{token-id}`

此路由可用於撤銷已授權 Access Token 及其相關的 Refresh Token：

```js
axios.delete('/oauth/tokens/' + tokenId);
```

<a name="refreshing-tokens"></a>
### 更新 Token

如果您的應用程式發行短效 Access Token，使用者將需要透過發行 Access Token 時提供給他們的 Refresh Token 來更新其 Access Token：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'refresh_token',
        'refresh_token' => 'the-refresh-token',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => '',
    ]);

    return $response->json();

此 `/oauth/token` 路由將傳回一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含 Access Token 過期前的秒數。

<a name="revoking-tokens"></a>
### 撤銷 Token

您可以使用 `Laravel\Passport\TokenRepository` 上的 `revokeAccessToken` 方法撤銷 Token。您可以使用 `Laravel\Passport\RefreshTokenRepository` 上的 `revokeRefreshTokensByAccessTokenId` 方法撤銷 Token 的 Refresh Token。這些類別可以使用 Laravel 的 [Service Container](/docs/{{version}}/container) 解析：

    use Laravel\Passport\TokenRepository;
    use Laravel\Passport\RefreshTokenRepository;

    $tokenRepository = app(TokenRepository::class);
    $refreshTokenRepository = app(RefreshTokenRepository::class);

    // Revoke an access token...
    $tokenRepository->revokeAccessToken($tokenId);

    // Revoke all of the token's refresh tokens...
    $refreshTokenRepository->revokeRefreshTokensByAccessTokenId($tokenId);

<a name="purging-tokens"></a>
### 清除 Token

當 Token 已被撤銷或過期時，您可能希望將它們從資料庫中清除。Passport 隨附的 `passport:purge` Artisan 命令可以為您完成此操作：

```shell
# Purge revoked and expired tokens and auth codes...
php artisan passport:purge

# Only purge tokens expired for more than 6 hours...
php artisan passport:purge --hours=6

# Only purge revoked tokens and auth codes...
php artisan passport:purge --revoked

# Only purge expired tokens and auth codes...
php artisan passport:purge --expired
```

您還可以設定應用程式 `routes/console.php` 檔案中的[排程任務](/docs/{{version}}/scheduling)，以按排程自動清除您的 Token：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('passport:purge')->hourly();

<a name="code-grant-pkce"></a>
## 帶有 PKCE 的授權碼授權

帶有「Proof Key for Code Exchange」(PKCE) 的授權碼授權是一種安全的方式，用於驗證單頁應用程式或原生應用程式以存取您的 API。當您無法保證 Client Secret 將被機密儲存，或為了減輕授權碼被攻擊者攔截的威脅時，應使用此授權。在將授權碼交換為 Access Token 時，「Code Verifier」和「Code Challenge」的組合取代了 Client Secret。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立 Client

在您的應用程式可以透過帶有 PKCE 的授權碼授權發行 Token 之前，您需要建立一個啟用 PKCE 的 Client。您可以使用帶有 `--public` 選項的 `passport:client` Artisan 命令來完成此操作：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求 Token

<a name="code-verifier-code-challenge"></a>
#### Code Verifier 和 Code Challenge

由於此授權不提供 Client Secret，開發人員需要產生 Code Verifier 和 Code Challenge 的組合才能請求 Token。

Code Verifier 應該是一個隨機字串，長度介於 43 到 128 個字元之間，包含字母、數字和 `"-"`、`"."`、`"_"`、`"~"` 字元，如 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636)中所定義。

Code Challenge 應該是一個 Base64 編碼的字串，帶有 URL 和檔案名安全的字元。應移除尾隨的 `'='` 字元，並且不應存在換行符、空白字元或其他額外字元。

    $encoded = base64_encode(hash('sha256', $code_verifier, true));

    $codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

Client 建立後，您可以使用 Client ID 和產生的 Code Verifier 和 Code Challenge 從您的應用程式請求授權碼和 Access Token。首先，消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求：

    use Illuminate\Http\Request;
    use Illuminate\Support\Str;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $request->session()->put(
            'code_verifier', $code_verifier = Str::random(128)
        );

        $codeChallenge = strtr(rtrim(
            base64_encode(hash('sha256', $code_verifier, true))
        , '='), '+/', '-_');

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'code',
            'scope' => '',
            'state' => $state,
            'code_challenge' => $codeChallenge,
            'code_challenge_method' => 'S256',
            // 'prompt' => '', // "none", "consent", or "login"
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });

<a name="code-grant-pkce-converting-authorization-codes-to-access-tokens"></a>
#### 將授權碼轉換為 Access Token

如果使用者批准了授權請求，他們將被重新導向回消費應用程式。消費者應根據重新導向之前儲存的值驗證 `state` 參數，如同標準授權碼授權一樣。

如果 `state` 參數匹配，消費者應向您的應用程式發出 `POST` 請求以請求 Access Token。請求應包含您的應用程式在使用者批准授權請求時發行的授權碼以及最初產生的 Code Verifier：

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Http;

    Route::get('/callback', function (Request $request) {
        $state = $request->session()->pull('state');

        $codeVerifier = $request->session()->pull('code_verifier');

        throw_unless(
            strlen($state) > 0 && $state === $request->state,
            InvalidArgumentException::class
        );

        $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
            'grant_type' => 'authorization_code',
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'code_verifier' => $codeVerifier,
            'code' => $request->code,
        ]);

        return $response->json();
    });

<a name="password-grant-tokens"></a>
## 密碼授權 Token

> [!WARNING]
> 我們不再建議使用密碼授權 Token。相反，您應該選擇 [OAuth2 伺服器目前推薦的授權類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 密碼授權允許您的其他第一方 Client (例如行動應用程式) 使用電子郵件地址/使用者名稱和密碼來取得 Access Token。這使您可以安全地向您的第一方 Client 發行 Access Token，而無需您的使用者經歷整個 OAuth2 授權碼重新導向流程。

要啟用密碼授權，請在您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }

<a name="creating-a-password-grant-client"></a>
### 建立密碼授權 Client

在您的應用程式可以透過密碼授權發行 Token 之前，您需要建立一個密碼授權 Client。您可以使用帶有 `--password` 選項的 `passport:client` Artisan 命令來完成此操作。**如果您已經執行了 `passport:install` 命令，則無需執行此命令：**

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>
### 請求 Token

建立密碼授權 Client 後，您可以透過向 `/oauth/token` 路由發出 `POST` 請求，並附帶使用者的電子郵件地址和密碼來請求 Access Token。請記住，此路由已由 Passport 註冊，因此無需手動定義。如果請求成功，您將從伺服器的 JSON 回應中收到 `access_token` 和 `refresh_token`：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor @laravel.com',
        'password' => 'my-password',
        'scope' => '',
    ]);

    return $response->json();

> [!NOTE]
> 請記住，Access Token 預設是長效的。但是，您可以根據需要[設定您的最大 Access Token 存留時間](#configuration)。

<a name="requesting-all-scopes"></a>
### 請求所有 Scope

當使用密碼授權或 Client Credentials 授權時，您可能希望為 Token 授權您的應用程式支援的所有 Scope。您可以透過請求 `*` Scope 來完成此操作。如果您請求 `*` Scope，Token 實例上的 `can` 方法將始終傳回 `true`。此 Scope 只能指派給使用 `password` 或 `client_credentials` 授權發行的 Token：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor @laravel.com',
        'password' => 'my-password',
        'scope' => '*',
    ]);

<a name="customizing-the-user-provider"></a>
### 自訂使用者 Provider

如果您的應用程式使用多個[驗證使用者 Provider](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --password` 命令建立 Client 時，透過提供 `--provider` 選項來指定密碼授權 Client 使用哪個使用者 Provider。給定的 Provider 名稱應與您的應用程式 `config/auth.php` 設定檔中定義的有效 Provider 相符。然後，您可以[使用 Middleware 保護您的路由](#via-middleware)，以確保只有來自 Guard 指定 Provider 的使用者才被授權。

<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

當使用密碼授權進行驗證時，Passport 將使用您的可驗證模型的 `email` 屬性作為「使用者名稱」。但是，您可以透過在模型上定義 `findForPassport` 方法來自訂此行為：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
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

<a name="customizing-the-password-validation"></a>
### 自訂密碼驗證

當使用密碼授權進行驗證時，Passport 將使用您模型的 `password` 屬性來驗證給定的密碼。如果您的模型沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，您可以在模型上定義 `validateForPassportPasswordGrant` 方法：

    <?php

    namespace App\Models;

    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    use Illuminate\Support\Facades\Hash;
    use Laravel\Passport\HasApiTokens;

    class User extends Authenticatable
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

<a name="implicit-grant-tokens"></a>
## 隱含授權 Token

> [!WARNING]
> 我們不再建議使用隱含授權 Token。相反，您應該選擇 [OAuth2 伺服器目前推薦的授權類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

隱含授權類似於授權碼授權；但是，Token 會直接傳回給 Client，而無需交換授權碼。此授權最常用於 Client Credentials 無法安全儲存的 JavaScript 或行動應用程式。要啟用此授權，請在您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::enableImplicitGrant();
    }

啟用授權後，開發人員可以使用其 Client ID 從您的應用程式請求 Access Token。消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

    use Illuminate\Http\Request;

    Route::get('/redirect', function (Request $request) {
        $request->session()->put('state', $state = Str::random(40));

        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://third-party-app.com/callback',
            'response_type' => 'token',
            'scope' => '',
            'state' => $state,
            // 'prompt' => '', // "none", "consent", or "login"
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });

> [!NOTE]
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="client-credentials-grant-tokens"></a>
## Client Credentials 授權 Token

Client Credentials 授權適用於機器對機器驗證。例如，您可以在執行 API 維護任務的排程任務中使用此授權。

在您的應用程式可以透過 Client Credentials 授權發行 Token 之前，您需要建立一個 Client Credentials 授權 Client。您可以使用 `passport:client` Artisan 命令的 `--client` 選項來完成此操作：

```shell
php artisan passport:client --client
```

接下來，要使用此授權類型，請為 `CheckClientCredentials` Middleware 註冊一個 Middleware 別名。您可以在應用程式的 `bootstrap/app.php` 檔案中定義 Middleware 別名：

    use Laravel\Passport\Http\Middleware\CheckClientCredentials;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'client' => CheckClientCredentials::class
        ]);
    })

然後，將 Middleware 附加到路由：

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client');

要將路由的存取限制為特定 Scope，您可以在將 `client` Middleware 附加到路由時，提供一個逗號分隔的所需 Scope 清單：

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client:check-status,your-scope');

<a name="retrieving-tokens"></a>
### 取得 Token

要使用此授權類型取得 Token，請向 `oauth/token` 端點發出請求：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'client_credentials',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => 'your-scope',
    ]);

    return $response->json()['access_token'];

<a name="personal-access-tokens"></a>
## 個人 Access Token

有時，您的使用者可能希望自行發行 Access Token，而無需經歷典型的授權碼重新導向流程。允許使用者透過您的應用程式 UI 自行發行 Token，對於允許使用者試驗您的 API 或作為發行 Access Token 的更簡單方法通常很有用。

> [!NOTE]
> 如果您的應用程式主要使用 Passport 發行個人 Access Token，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 輕量級的第一方函式庫，用於發行 API Access Token。

<a name="creating-a-personal-access-client"></a>
### 建立個人 Access Client

在您的應用程式可以發行個人 Access Token 之前，您需要建立一個個人 Access Client。您可以透過執行帶有 `--personal` 選項的 `passport:client` Artisan 命令來完成此操作。如果您已經執行了 `passport:install` 命令，則無需執行此命令：

```shell
php artisan passport:client --personal
```

建立個人 Access Client 後，將 Client 的 ID 和純文字 Secret 值放入您的應用程式 `.env` 檔案中：

```ini
PASSPORT_PERSONAL_ACCESS_CLIENT_ID="client-id-value"
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET="unhashed-client-secret-value"
```

<a name="managing-personal-access-tokens"></a>
### 管理個人 Access Token

建立個人 Access Client 後，您可以使用 `App\Models\User` 模型實例上的 `createToken` 方法為給定使用者發行 Token。`createToken` 方法接受 Token 的名稱作為第一個參數，以及一個可選的 [Scope](#token-scopes) 陣列作為第二個參數：

    use App\Models\User;

    $user = User::find(1);

    // Creating a token without scopes...
    $token = $user->createToken('Token Name')->accessToken;

    // Creating a token with scopes...
    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="personal-access-tokens-json-api"></a>
#### JSON API

Passport 還包含一個用於管理個人 Access Token 的 JSON API。您可以將其與您自己的前端配對，為您的使用者提供一個用於管理個人 Access Token 的儀表板。下面，我們將回顧所有用於管理個人 Access Token 的 API 端點。為方便起見，我們將使用 [Axios](https://github.com/axios/axios) 來演示向端點發出 HTTP 請求。

JSON API 受 `web` 和 `auth` Middleware 保護；因此，它只能從您自己的應用程式中呼叫。它無法從外部來源呼叫。

<a name="get-oauthscopes"></a>
#### `GET /oauth/scopes`

此路由會傳回為您的應用程式定義的所有 [Scope](#token-scopes)。您可以使用此路由列出使用者可以指派給個人 Access Token 的 Scope：

```js
axios.get('/oauth/scopes')
    .then(response => {
        console.log(response.data);
    });
```

<a name="get-oauthpersonal-access-tokens"></a>
#### `GET /oauth/personal-access-tokens`

此路由會傳回已驗證使用者建立的所有個人 Access Token。這主要用於列出使用者的所有 Token，以便他們可以編輯或撤銷它們：

```js
axios.get('/oauth/personal-access-tokens')
    .then(response => {
        console.log(response.data);
    });
```

<a name="post-oauthpersonal-access-tokens"></a>
#### `POST /oauth/personal-access-tokens`

此路由會建立新的個人 Access Token。它需要兩部分資料：Token 的 `name` 和應指派給 Token 的 `scopes`：

```js
const data = {
    name: 'Token Name',
    scopes: []
};

axios.post('/oauth/personal-access-tokens', data)
    .then(response => {
        console.log(response.data.accessToken);
    })
    .catch (response => {
        // List errors on response...
    });
```

<a name="delete-oauthpersonal-access-tokenstoken-id"></a>
#### `DELETE /oauth/personal-access-tokens/{token-id}`

此路由可用於撤銷個人 Access Token：

```js
axios.delete('/oauth/personal-access-tokens/' + tokenId);
```

<a name="protecting-routes"></a>
## 保護路由

<a name="via-middleware"></a>
### 透過 Middleware

Passport 包含一個[驗證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，它將驗證傳入請求上的 Access Token。一旦您將 `api` Guard 設定為使用 `passport` Driver，您只需在任何需要有效 Access Token 的路由上指定 `auth:api` Middleware：

    Route::get('/user', function () {
        // ...
    })->middleware('auth:api');

> [!WARNING]
> 如果您使用 [Client Credentials 授權](#client-credentials-grant-tokens)，您應該使用 [Client Middleware](#client-credentials-grant-tokens) 來保護您的路由，而不是 `auth:api` Middleware。

<a name="multiple-authentication-guards"></a>
#### 多個驗證 Guard

如果您的應用程式驗證不同類型的使用者，他們可能使用完全不同的 Eloquent 模型，您可能需要為應用程式中的每個使用者 Provider 類型定義一個 Guard 設定。這允許您保護針對特定使用者 Provider 的請求。例如，給定 `config/auth.php` 設定檔中的以下 Guard 設定：

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],

    'api-customers' => [
        'driver' => 'passport',
        'provider' => 'customers',
    ],

以下路由將使用 `api-customers` Guard (它使用 `customers` 使用者 Provider) 來驗證傳入請求：

    Route::get('/customer', function () {
        // ...
    })->middleware('auth:api-customers');

> [!NOTE]
> 有關將多個使用者 Provider 與 Passport 結合使用的更多資訊，請查閱[密碼授權文件](#customizing-the-user-provider)。

<a name="passing-the-access-token"></a>
### 傳遞 Access Token

當呼叫受 Passport 保護的路由時，您的應用程式的 API 消費者應將其 Access Token 指定為請求 `Authorization` Header 中的 `Bearer` Token。例如，當使用 Guzzle HTTP 函式庫時：

    use Illuminate\Support\Facades\Http;

    $response = Http::withHeaders([
        'Accept' => 'application/json',
        'Authorization' => 'Bearer '.$accessToken,
    ])->get('https://passport-app.test/api/user');

    return $response->json();

<a name="token-scopes"></a>
## Token Scope

Scope 允許您的 API Client 在請求授權以存取帳戶時請求一組特定的權限。例如，如果您正在建構一個電子商務應用程式，並非所有 API 消費者都需要下訂單的能力。相反，您可以允許消費者僅請求授權以存取訂單出貨狀態。換句話說，Scope 允許您的應用程式使用者限制第三方應用程式可以代表他們執行的操作。

<a name="defining-scopes"></a>
### 定義 Scope

您可以使用 `Passport::tokensCan` 方法在您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中定義您的 API 的 Scope。`tokensCan` 方法接受一個 Scope 名稱和 Scope 描述的陣列。Scope 描述可以是您想要的任何內容，並將顯示給授權批准畫面上的使用者：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::tokensCan([
            'place-orders' => 'Place orders',
            'check-status' => 'Check order status',
        ]);
    }

<a name="default-scope"></a>
### 預設 Scope

如果 Client 未請求任何特定 Scope，您可以設定您的 Passport 伺服器使用 `setDefaultScope` 方法將預設 Scope 附加到 Token。通常，您應該從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

    use Laravel\Passport\Passport;

    Passport::tokensCan([
        'place-orders' => 'Place orders',
        'check-status' => 'Check order status',
    ]);

    Passport::setDefaultScope([
        'check-status',
        'place-orders',
    ]);

> [!NOTE]
> Passport 的預設 Scope 不適用於使用者產生的個人 Access Token。

<a name="assigning-scopes-to-tokens"></a>
### 為 Token 指派 Scope

<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

當使用授權碼授權請求 Access Token 時，消費者應將其所需的 Scope 指定為 `scope` 查詢字串參數。`scope` 參數應為以空格分隔的 Scope 清單：

    Route::get('/redirect', function () {
        $query = http_build_query([
            'client_id' => 'client-id',
            'redirect_uri' => 'http://example.com/callback',
            'response_type' => 'code',
            'scope' => 'place-orders check-status',
        ]);

        return redirect('http://passport-app.test/oauth/authorize?'.$query);
    });

<a name="when-issuing-personal-access-tokens"></a>
#### 發行個人 Access Token 時

如果您使用 `App\Models\User` 模型的 `createToken` 方法發行個人 Access Token，您可以將所需的 Scope 陣列作為方法的第二個參數傳遞：

    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="checking-scopes"></a>
### 檢查 Scope

Passport 包含兩個 Middleware，可用於驗證傳入請求是否已使用已授予指定 Scope 的 Token 進行驗證。首先，在您的應用程式 `bootstrap/app.php` 檔案中定義以下 Middleware 別名：

    use Laravel\Passport\Http\Middleware\CheckForAnyScope;
    use Laravel\Passport\Http\Middleware\CheckScopes;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'scopes' => CheckScopes::class,
            'scope' => CheckForAnyScope::class,
        ]);
    })

<a name="check-for-all-scopes"></a>
#### 檢查所有 Scope

`scopes` Middleware 可以指派給路由，以驗證傳入請求的 Access Token 是否具有所有列出的 Scope：

    Route::get('/orders', function () {
        // Access token has both "check-status" and "place-orders" scopes...
    })->middleware(['auth:api', 'scopes:check-status,place-orders']);

<a name="check-for-any-scopes"></a>
#### 檢查任何 Scope

`scope` Middleware 可以指派給路由，以驗證傳入請求的 Access Token 是否具有*至少一個*列出的 Scope：

    Route::get('/orders', function () {
        // Access token has either "check-status" or "place-orders" scope...
    })->middleware(['auth:api', 'scope:check-status,place-orders']);

<a name="checking-scopes-on-a-token-instance"></a>
#### 檢查 Token 實例上的 Scope

一旦 Access Token 驗證的請求進入您的應用程式，您仍然可以使用已驗證 `App\Models\User` 實例上的 `tokenCan` 方法檢查 Token 是否具有給定的 Scope：

    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        if ($request->user()->tokenCan('place-orders')) {
            // ...
        }
    });

<a name="additional-scope-methods"></a>
#### 其他 Scope 方法

`scopeIds` 方法將傳回所有已定義 ID / 名稱的陣列：

    use Laravel\Passport\Passport;

    Passport::scopeIds();

`scopes` 方法將傳回所有已定義 Scope 的陣列，作為 `Laravel\Passport\Scope` 的實例：

    Passport::scopes();

`scopesFor` 方法將傳回與給定 ID / 名稱相符的 `Laravel\Passport\Scope` 實例陣列：

    Passport::scopesFor(['place-orders', 'check-status']);

您可以使用 `hasScope` 方法判斷是否已定義給定的 Scope：

    Passport::hasScope('place-orders');

<a name="consuming-your-api-with-javascript"></a>
## 使用 JavaScript 存取您的 API

在建構 API 時，能夠從您的 JavaScript 應用程式存取您自己的 API 非常有用。這種 API 開發方法允許您自己的應用程式存取您與世界共享的相同 API。相同的 API 可以由您的 Web 應用程式、行動應用程式、第三方應用程式以及您可能在各種套件管理器上發布的任何 SDK 存取。

通常，如果您想從您的 JavaScript 應用程式存取您的 API，您需要手動將 Access Token 傳送到應用程式，並在每次請求時將其傳遞給您的應用程式。但是，Passport 包含一個可以為您處理此問題的 Middleware。您只需將 `CreateFreshApiToken` Middleware 附加到您的應用程式 `bootstrap/app.php` 檔案中的 `web` Middleware 群組：

    use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            CreateFreshApiToken::class,
        ]);
    })

> [!WARNING]
> 您應該確保 `CreateFreshApiToken` Middleware 是您的 Middleware 堆疊中列出的最後一個 Middleware。

此 Middleware 會將 `laravel_token` Cookie 附加到您的傳出回應。此 Cookie 包含一個加密的 JWT，Passport 將使用它來驗證來自您的 JavaScript 應用程式的 API 請求。JWT 的存留時間等於您的 `session.lifetime` 設定值。現在，由於瀏覽器會自動將 Cookie 與所有後續請求一起傳送，因此您可以向您的應用程式 API 發出請求，而無需明確傳遞 Access Token：

    axios.get('/api/user')
        .then(response => {
            console.log(response.data);
        });

<a name="customizing-the-cookie-name"></a>
#### 自訂 Cookie 名稱

如果需要，您可以使用 `Passport::cookie` 方法自訂 `laravel_token` Cookie 的名稱。通常，此方法應從您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::cookie('custom_name');
    }

<a name="csrf-protection"></a>
#### CSRF 保護

當使用此驗證方法時，您需要確保請求中包含有效的 CSRF Token Header。預設的 Laravel JavaScript 腳手架包含一個 Axios 實例，它將自動使用加密的 `XSRF-TOKEN` Cookie 值在同源請求上傳送 `X-XSRF-TOKEN` Header。

> [!NOTE]
> 如果您選擇傳送 `X-CSRF-TOKEN` Header 而不是 `X-XSRF-TOKEN`，您將需要使用 `csrf_token()` 提供的未加密 Token。

<a name="events"></a>
## 事件

Passport 在發行 Access Token 和 Refresh Token 時會觸發事件。您可以[監聽這些事件](/docs/{{version}}/events)以清除或撤銷資料庫中的其他 Access Token：

<div class="overflow-auto">

| 事件名稱 |
| --- |
| `Laravel\Passport\Events\AccessTokenCreated` |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>

<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用於指定目前已驗證的使用者及其 Scope。傳遞給 `actingAs` 方法的第一個參數是使用者實例，第二個是應授予使用者 Token 的 Scope 陣列：

```php tab=Pest
use App\Models\User;
use Laravel\Passport\Passport;

test('servers can be created', function () {
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
});
```

```php tab=PHPUnit
use App\Models\User;
use Laravel\Passport\Passport;

public function test_servers_can_be_created(): void
{
    Passport::actingAs(
        User::factory()->create(),
        ['create-servers']
    );

    $response = $this->post('/api/create-server');

    $response->assertStatus(201);
}
```

Passport 的 `actingAsClient` 方法可用於指定目前已驗證的 Client 及其 Scope。傳遞給 `actingAsClient` 方法的第一個參數是 Client 實例，第二個是應授予 Client Token 的 Scope 陣列：

```php tab=Pest
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

test('orders can be retrieved', function () {
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
use Laravel\Passport\Client;
use Laravel\Passport\Passport;

public function test_orders_can_be_retrieved(): void
{
    Passport::actingAsClient(
        Client::factory()->create(),
        ['check-status']
    );

    $response = $this->get('/api/orders');

    $response->assertStatus(200);
}
```
