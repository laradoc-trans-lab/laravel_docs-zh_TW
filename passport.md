# Laravel Passport

- [簡介](#introduction)
    - [Passport 還是 Sanctum？](#passport-or-sanctum)
- [安裝](#installation)
    - [部署 Passport](#deploying-passport)
    - [升級 Passport](#upgrading-passport)
- [設定](#configuration)
    - [客戶端密鑰雜湊](#client-secret-hashing)
    - [憑證存留期](#token-lifetimes)
    - [覆寫預設模型](#overriding-default-models)
    - [覆寫路由](#overriding-routes)
- [發行存取憑證](#issuing-access-tokens)
    - [管理客戶端](#managing-clients)
    - [請求憑證](#requesting-tokens)
    - [刷新憑證](#refreshing-tokens)
    - [撤銷憑證](#revoking-tokens)
    - [清除憑證](#purging-tokens)
- [帶有 PKCE 的授權碼授權類型](#code-grant-pkce)
    - [建立客戶端](#creating-a-auth-pkce-grant-client)
    - [請求憑證](#requesting-auth-pkce-grant-tokens)
- [密碼授權類型憑證](#password-grant-tokens)
    - [建立密碼授權類型客戶端](#creating-a-password-grant-client)
    - [請求憑證](#requesting-password-grant-tokens)
    - [請求所有 Scope](#requesting-all-scopes)
    - [自訂使用者 Provider](#customizing-the-user-provider)
    - [自訂使用者名稱欄位](#customizing-the-username-field)
    - [自訂密碼驗證](#customizing-the-password-validation)
- [隱含授權類型憑證](#implicit-grant-tokens)
- [客戶端憑證授權類型憑證](#client-credentials-grant-tokens)
- [個人存取憑證](#personal-access-tokens)
    - [建立個人存取客戶端](#creating-a-personal-access-client)
    - [管理個人存取憑證](#managing-personal-access-tokens)
- [保護路由](#protecting-routes)
    - [透過 Middleware](#via-middleware)
    - [傳遞存取憑證](#passing-the-access-token)
- [憑證 Scope](#token-scopes)
    - [定義 Scope](#defining-scopes)
    - [預設 Scope](#default-scope)
    - [將 Scope 分配給憑證](#assigning-scopes-to-tokens)
    - [檢查 Scope](#checking-scopes)
- [透過 JavaScript 呼叫您的 API](#consuming-your-api-with-javascript)
- [事件](#events)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

[Laravel Passport](https://github.com/laravel/passport) 在幾分鐘內為您的 Laravel 應用程式提供完整的 OAuth2 伺服器實作。Passport 是建立在由 Andy Millington 和 Simon Hamp 維護的 [League OAuth2 server](https://github.com/thephpleague.com/oauth2-server) 之上。

> [!WARNING]  
> 本文件假設您已熟悉 OAuth2。如果您對 OAuth2 一無所知，建議在繼續之前先熟悉 OAuth2 的一般 [術語](https://oauth2.thephpleague.com/terminology/) 和功能。


<a name="passport-or-sanctum"></a>
### Passport 還是 Sanctum？

在開始之前，您可能會希望判斷您的應用程式更適合使用 Laravel Passport 還是 [Laravel Sanctum](/docs/{{version}}/sanctum)。如果您的應用程式絕對需要支援 OAuth2，那麼您應該使用 Laravel Passport。

然而，如果您正在嘗試驗證單頁應用程式、行動應用程式或發行 API tokens，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。Laravel Sanctum 不支援 OAuth2；但是，它提供了更簡單的 API 驗證開發體驗。


<a name="installation"></a>
## 安裝

您可以使用 `install:api` Artisan 命令來安裝 Laravel Passport：

```shell
php artisan install:api --passport
```

此命令將發布並執行資料庫遷移，這些遷移對於建立您的應用程式儲存 OAuth2 客戶端和存取 tokens 所需的表格是必要的。該命令還會建立產生安全存取 tokens 所需的加密金鑰。

此外，此命令會詢問您是否願意使用 UUIDs 作為 Passport `Client` 模型的 Primary Key 值，而不是自動遞增的整數。

執行 `install:api` 命令後，將 `Laravel\Passport\HasApiTokens` trait 添加到您的 `App\Models\User` 模型中。此 trait 將為您的模型提供一些 Helper 方法，允許您檢查已驗證使用者的 token 和 scopes：

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

最後，在您的應用程式的 `config/auth.php` 設定檔中，您應該定義一個 `api` 驗證 guard 並將 `driver` 選項設定為 `passport`。這將指示您的應用程式在驗證傳入的 API 請求時使用 Passport 的 `TokenGuard`：

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

首次將 Passport 部署到您的應用程式伺服器時，您可能需要執行 `passport:keys` 命令。此命令會產生 Passport 產生存取 tokens 所需的加密金鑰。這些產生的金鑰通常不保留在原始碼控制中：

```shell
php artisan passport:keys
```

如有必要，您可以定義 Passport 金鑰應從何處載入的路徑。您可以使用 `Passport::loadKeysFrom` 方法來實現此目的。通常，此方法應在您的應用程式 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
    }


<a name="loading-keys-from-the-environment"></a>
#### 從環境載入金鑰

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

升級到 Passport 的新主要版本時，仔細審閱 [升級指南](https://github.com/laravel/passport/blob/master/UPGRADE.md) 非常重要。

<a name="configuration"></a>
## 設定

<a name="client-secret-hashing"></a>
### 客戶端密鑰雜湊

如果您希望客戶端密鑰在資料庫中儲存時進行雜湊處理，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `Passport::hashClientSecrets` 方法：

    use Laravel\Passport\Passport;

    Passport::hashClientSecrets();

啟用後，您所有的客戶端密鑰將只在建立後立即向使用者顯示。由於純文字的客戶端密鑰值從未儲存在資料庫中，因此如果密鑰遺失，將無法恢復其值。

<a name="token-lifetimes"></a>
### 憑證存留期

預設情況下，Passport 會發行有效期為一年的長期存取憑證。如果您想設定更長或更短的憑證存留期，可以使用 `tokensExpireIn`、`refreshTokensExpireIn` 和 `personalAccessTokensExpireIn` 方法。這些方法應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
> Passport 資料庫資料表中的 `expires_at` 欄位是唯讀的，僅供顯示之用。發行憑證時，Passport 會將過期資訊儲存在已簽署和加密的憑證中。如果您需要使憑證失效，應該[撤銷它](#revoking-tokens)。

<a name="overriding-default-models"></a>
### 覆寫預設模型

您可以透過定義自己的模型並擴充對應的 Passport 模型來自由地擴充 Passport 內部使用的模型：

    use Laravel\Passport\Client as PassportClient;

    class Client extends PassportClient
    {
        // ...
    }

定義模型後，您可以透過 `Laravel\Passport\Passport` 類別指示 Passport 使用您的自訂模型。通常，您應該在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中告知 Passport 您的自訂模型：

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

有時您可能希望自訂 Passport 所定義的路由。為此，您首先需要透過將 `Passport::ignoreRoutes` 加入到應用程式的 `AppServiceProvider` 的 `register` 方法中，來忽略 Passport 註冊的路由：

    use Laravel\Passport\Passport;

    /**
     * Register any application services.
     */
    public function register(): void
    {
        Passport::ignoreRoutes();
    }

然後，您可以將 Passport 在[其路由檔](https://github.com/laravel/passport/blob/11.x/routes/web.php)中定義的路由複製到應用程式的 `routes/web.php` 檔案中，並根據您的喜好進行修改：

    Route::group([
        'as' => 'passport.',
        'prefix' => config('passport.path', 'oauth'),
        'namespace' => '\Laravel\Passport\Http\Controllers',
    ], function () {
        // Passport routes...
    });

<a name="issuing-access-tokens"></a>
## 發行存取憑證

大多數開發者都熟悉透過授權碼 (authorization codes) 使用 OAuth2。使用授權碼時，客戶端應用程式會將使用者重新導向您的伺服器，使用者可以在此核准或拒絕向客戶端發行存取憑證的請求。

<a name="managing-clients"></a>
### 管理客戶端

首先，開發需要與您的應用程式 API 互動的應用程式時，需要透過建立「客戶端」來向您的應用程式註冊其應用程式。通常，這包括提供其應用程式的名稱以及一個 URL，您的應用程式在使用者核准其授權請求後可以重新導向到該 URL。

<a name="the-passportclient-command"></a>
#### `passport:client` 指令

建立客戶端最簡單的方式是使用 `passport:client` Artisan 指令。此指令可用於建立您自己的客戶端，以測試您的 OAuth2 功能。當您執行 `client` 指令時，Passport 會提示您輸入有關客戶端的更多資訊，並提供您客戶端 ID 和密鑰：

```shell
php artisan passport:client
```

**重新導向 URL**

如果您想為客戶端允許多個重新導向 URL，當 `passport:client` 指令提示您輸入 URL 時，您可以使用逗號分隔清單來指定它們。任何包含逗號的 URL 都應該進行 URL 編碼：

```shell
http://example.com/callback,http://examplefoo.com/callback
```

<a name="clients-json-api"></a>
#### JSON API

由於您的應用程式使用者無法利用 `client` 指令，Passport 提供了一個 JSON API，您可以用來建立客戶端。這省去了您手動編寫用於建立、更新和刪除客戶端的控制器。

然而，您需要將 Passport 的 JSON API 與您自己的前端配對，以提供一個儀表板供使用者管理他們的客戶端。下面，我們將回顧所有用於管理客戶端的 API 端點。為方便起見，我們將使用 [Axios](https://github.com/axios/axios) 來演示向這些端點發送 HTTP 請求。

JSON API 由 `web` 和 `auth` 中介層保護；因此，它只能從您自己的應用程式中呼叫。無法從外部來源呼叫。

<a name="get-oauthclients"></a>
#### `GET /oauth/clients`

此路由會回傳已驗證使用者所有客戶端。這主要用於列出使用者所有客戶端，以便他們可以編輯或刪除它們：

```js
axios.get('/oauth/clients')
    .then(response => {
        console.log(response.data);
    });
```

<a name="post-oauthclients"></a>
#### `POST /oauth/clients`

此路由用於建立新的客戶端。它需要兩部分資料：客戶端的 `name` 和 `redirect` URL。`redirect` URL 是使用者在核准或拒絕授權請求後將被重新導向的位置。

建立客戶端後，將會發行客戶端 ID 和客戶端密鑰。這些值將在從您的應用程式請求存取憑證時使用。客戶端建立路由將回傳新的客戶端實例：

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

此路由用於更新客戶端。它需要兩部分資料：客戶端的 `name` 和 `redirect` URL。`redirect` URL 是使用者在核准或拒絕授權請求後將被重新導向的位置。此路由將回傳更新後的客戶端實例：

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

此路由用於刪除客戶端：

```js
axios.delete('/oauth/clients/' + clientId)
    .then(response => {
        // ...
    });
```

<a name="requesting-tokens"></a>
### 請求憑證

<a name="requesting-tokens-redirecting-for-authorization"></a>
#### 重導向以進行授權

一旦客戶端建立完成，開發者即可使用其客戶端 ID 和密鑰向您的應用程式請求授權碼與存取憑證。首先，消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重導向請求，如下所示：

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

`prompt` 參數可用於指定 Passport 應用程式的認證行為。

如果 `prompt` 值為 `none`，若使用者尚未透過 Passport 應用程式認證，Passport 將始終拋出認證錯誤。如果值為 `consent`，Passport 將始終顯示授權核准畫面，即使所有 Scope 先前已授予消費應用程式。當值為 `login` 時，Passport 應用程式將始終提示使用者重新登入應用程式，即使他們已經有現有會話。

如果沒有提供 `prompt` 值，只有在使用者先前尚未授權存取消費應用程式所請求的 Scope 時，才會提示使用者進行授權。

> [!NOTE]  
> 請記住，`/oauth/authorize` 路由已由 Passport 定義。您無需手動定義此路由。

<a name="approving-the-request"></a>
#### 核准請求

當收到授權請求時，Passport 將根據 `prompt` 參數的值（如果存在）自動回應，並可能向使用者顯示一個模板，允許他們核准或拒絕授權請求。如果他們核准請求，他們將被重導向回消費應用程式指定的 `redirect_uri`。`redirect_uri` 必須與客戶端建立時指定的 `redirect` URL 相符。

如果您想自訂授權核准畫面，可以使用 `vendor:publish` Artisan 命令發佈 Passport 的視圖。發佈的視圖將會放置在 `resources/views/vendor/passport` 目錄中：

```shell
php artisan vendor:publish --tag=passport-views
```

有時您可能希望跳過授權提示，例如在授權第一方客戶端時。您可以透過 [擴展 `Client` 模型](#overriding-default-models) 並定義一個 `skipsAuthorization` 方法來實現。如果 `skipsAuthorization` 返回 `true`，客戶端將被核准，使用者將立即被重導向回 `redirect_uri`，除非消費應用程式在重導向以進行授權時明確設定了 `prompt` 參數：

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
#### 將授權碼轉換為存取憑證

如果使用者核准了授權請求，他們將被重導向回消費應用程式。消費方應首先驗證 `state` 參數是否與重導向前儲存的值一致。若 state 參數相符，則消費方應向您的應用程式發出 `POST` 請求以請求存取憑證。此請求應包含您的應用程式在使用者核准授權請求時發出的授權碼：

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

此 `/oauth/token` 路由將返回一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取憑證到期前的秒數。

> [!NOTE]  
> 與 `/oauth/authorize` 路由一樣，`/oauth/token` 路由已由 Passport 為您定義。無需手動定義此路由。

<a name="tokens-json-api"></a>
#### JSON API

Passport 也包含一個用於管理已授權存取憑證的 JSON API。您可以將其與您自己的前端配對，為使用者提供一個用於管理存取憑證的儀表板。為方便起見，我們將使用 [Axios](https://github.com/axios/axios) 來演示向端點發出 HTTP 請求。JSON API 受 `web` 和 `auth` 中介層保護；因此，它只能從您自己的應用程式呼叫。

<a name="get-oauthtokens"></a>
#### `GET /oauth/tokens`

此路由返回已認證使用者建立的所有已授權存取憑證。這主要用於列出使用者所有的憑證，以便他們可以撤銷這些憑證：

```js
axios.get('/oauth/tokens')
    .then(response => {
        console.log(response.data);
    });
```

<a name="delete-oauthtokenstoken-id"></a>
#### `DELETE /oauth/tokens/{token-id}`

此路由可用於撤銷已授權存取憑證及其相關的刷新憑證：

```js
axios.delete('/oauth/tokens/' + tokenId);
```

<a name="refreshing-tokens"></a>
### 刷新憑證

如果您的應用程式發行短期存取憑證，使用者將需要透過發行存取憑證時提供給他們的刷新憑證來刷新其存取憑證：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'refresh_token',
        'refresh_token' => 'the-refresh-token',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => '',
    ]);

    return $response->json();

此 `/oauth/token` 路由將返回一個 JSON 回應，其中包含 `access_token`、`refresh_token` 和 `expires_in` 屬性。`expires_in` 屬性包含存取憑證到期前的秒數。

<a name="revoking-tokens"></a>
### 撤銷憑證

您可以使用 `Laravel\Passport\TokenRepository` 上的 `revokeAccessToken` 方法來撤銷憑證。您可以使用 `Laravel\Passport\RefreshTokenRepository` 上的 `revokeRefreshTokensByAccessTokenId` 方法來撤銷憑證的刷新憑證。這些類別可以使用 Laravel 的 [服務容器](/docs/{{version}}/container) 來解析：

    use Laravel\Passport\TokenRepository;
    use Laravel\Passport\RefreshTokenRepository;

    $tokenRepository = app(TokenRepository::class);
    $refreshTokenRepository = app(RefreshTokenRepository::class);

    // Revoke an access token...
    $tokenRepository->revokeAccessToken($tokenId);

    // Revoke all of the token's refresh tokens...
    $refreshTokenRepository->revokeRefreshTokensByAccessTokenId($tokenId);

<a name="purging-tokens"></a>
### 清除憑證

當憑證被撤銷或過期時，您可能想將它們從資料庫中清除。Passport 內建的 `passport:purge` Artisan 指令可以為您完成這項工作：

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

您也可以在應用程式的 `routes/console.php` 檔案中設定一個 [排程工作](/docs/{{version}}/scheduling)，以排程方式自動清理您的憑證：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('passport:purge')->hourly();

<a name="code-grant-pkce"></a>
## 帶有 PKCE 的授權碼授權類型

帶有「程式碼交換證明鑰匙 (Proof Key for Code Exchange)」(PKCE) 的授權碼授權類型是單頁應用程式或原生應用程式安全地存取您 API 的一種方式。當您無法保證客戶端密鑰能被機密儲存，或者為了降低授權碼被攻擊者攔截的威脅時，應使用此授權類型。在交換授權碼以取得存取憑證時，結合了「程式碼驗證器 (code verifier)」與「程式碼挑戰 (code challenge)」來取代客戶端密鑰。

<a name="creating-a-auth-pkce-grant-client"></a>
### 建立客戶端

在您的應用程式能透過帶有 PKCE 的授權碼授權類型發行憑證之前，您需要建立一個啟用 PKCE 的客戶端。您可以使用帶有 `--public` 選項的 `passport:client` Artisan 命令來完成此操作：

```shell
php artisan passport:client --public
```

<a name="requesting-auth-pkce-grant-tokens"></a>
### 請求憑證

<a name="code-verifier-code-challenge"></a>
#### 程式碼驗證器與程式碼挑戰

由於此授權類型不提供客戶端密鑰，開發人員需要產生程式碼驗證器與程式碼挑戰的組合，以請求憑證。

程式碼驗證器應是一個介於 43 到 128 個字元的隨機字串，其中包含字母、數字，以及 `"-"`、`"."`、`"_"`、`"~"` 字元，如 [RFC 7636 規範](https://tools.ietf.org/html/rfc7636)中所定義。

程式碼挑戰應是一個帶有 URL 和檔案名稱安全字元的 Base64 編碼字串。應移除尾隨的 `'='` 字元，且不得有任何換行符號、空白或其他額外字元。

    $encoded = base64_encode(hash('sha256', $code_verifier, true));

    $codeChallenge = strtr(rtrim($encoded, '='), '+/', '-_');

<a name="code-grant-pkce-redirecting-for-authorization"></a>
#### 重新導向以進行授權

客戶端建立後，您可以使用客戶端 ID 以及產生的程式碼驗證器和程式碼挑戰，向您的應用程式請求授權碼和存取憑證。首先，消費應用程式應向您應用程式的 `/oauth/authorize` 路由發出重新導向請求：

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
#### 將授權碼轉換為存取憑證

如果使用者核准授權請求，他們將被重新導向回消費應用程式。消費方應根據重新導向前儲存的值來驗證 `state` 參數，如同標準授權碼授權類型一樣。

如果 state 參數相符，消費方應向您的應用程式發出 `POST` 請求以請求存取憑證。該請求應包含您的應用程式在使用者核准授權請求時發行的授權碼，以及最初產生的程式碼驗證器：

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
## 密碼授權類型憑證

> [!WARNING]  
> 我們不再建議使用 implicit grant tokens。相反地，您應該選擇 [OAuth2 Server 目前推薦的授權類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

OAuth2 password grant 允許您的其他第一方客戶端，例如行動應用程式，使用電子郵件地址 / 使用者名稱和密碼來獲取存取憑證。這讓您可以安全地向第一方客戶端發行存取憑證，而無需讓使用者經歷完整的 OAuth2 授權碼重新導向流程。

要啟用 password grant，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enablePasswordGrant` 方法：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }

<a name="creating-a-password-grant-client"></a>
### 建立密碼授權類型客戶端

在您的應用程式能透過 password grant 發行憑證之前，您需要建立一個 password grant 客戶端。您可以使用帶有 `--password` 選項的 `passport:client` Artisan 命令來完成此操作。**如果您已經執行過 `passport:install` 命令，則無需再次執行此命令：**

```shell
php artisan passport:client --password
```

<a name="requesting-password-grant-tokens"></a>
### 請求憑證

建立 password grant 客戶端後，您可以向 `/oauth/token` 路由發出 `POST` 請求，並提供使用者的電子郵件地址和密碼，以請求存取憑證。請記住，此路由已由 Passport 註冊，因此無需手動定義。如果請求成功，您將從伺服器收到包含 `access_token` 和 `refresh_token` 的 JSON 回應：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '',
    ]);

    return $response->json();

> [!NOTE]  
> 請記住，access tokens 預設是長期有效的。但是，如果需要，您可以自由地[設定您的最大 access token 存留期](#configuration)。

<a name="requesting-all-scopes"></a>
### 請求所有 Scope

當使用 password grant 或 client credentials grant 時，您可能希望為憑證授權應用程式支援的所有 scopes。您可以透過請求 `*` scope 來完成此操作。如果您請求 `*` scope，憑證實例上的 `can` 方法將始終返回 `true`。此 scope 只能分配給使用 `password` 或 `client_credentials` grant 發行的憑證：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'password',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'username' => 'taylor@laravel.com',
        'password' => 'my-password',
        'scope' => '*',
    ]);

<a name="customizing-the-user-provider"></a>
### 自訂使用者 Provider

如果您的應用程式使用多個[身份驗證使用者 provider](/docs/{{version}}/authentication#introduction)，您可以在透過 `artisan passport:client --password` 命令建立客戶端時，透過提供 `--provider` 選項來指定 password grant 客戶端使用哪個使用者 provider。提供的 provider 名稱應與應用程式 `config/auth.php` 設定檔中定義的有效 provider 相符。然後，您可以[使用 middleware 保護您的路由](#via-middleware)，以確保只有來自 guard 指定 provider 的使用者被授權。

<a name="customizing-the-username-field"></a>
### 自訂使用者名稱欄位

當使用 password grant 進行身份驗證時，Passport 將使用您可驗證模型的 `email` 屬性作為「使用者名稱」。但是，您可以透過在模型上定義 `findForPassport` 方法來自訂此行為：

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

當使用 password grant 進行身份驗證時，Passport 將使用您模型的 `password` 屬性來驗證提供的密碼。如果您的模型沒有 `password` 屬性，或者您希望自訂密碼驗證邏輯，您可以在模型上定義 `validateForPassportPasswordGrant` 方法：

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
## 隱含授權類型憑證

> [!WARNING]  
> 我們不再建議使用 implicit grant tokens。相反地，您應該選擇 [OAuth2 Server 目前推薦的授權類型](https://oauth2.thephpleague.com/authorization-server/which-grant/)。

implicit grant 類似於 authorization code grant；但是，憑證會直接返回給客戶端，而無需交換授權碼。此 grant 最常用於無法安全儲存客戶端憑證的 JavaScript 或行動應用程式。要啟用此 grant，請在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫 `enableImplicitGrant` 方法：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::enableImplicitGrant();
    }

啟用此 grant 後，開發人員可以使用其 client ID 從您的應用程式請求 access token。消費應用程式應向您的應用程式的 `/oauth/authorize` 路由發出重新導向請求，如下所示：

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
## 客戶端憑證授權類型憑證

客戶端憑證授權類型適用於機器對機器的認證。例如，您可能會在一個透過 API 執行維護任務的排程工作中使用此授權類型。

在您的應用程式能透過客戶端憑證授權類型發行憑證之前，您需要建立一個客戶端憑證授權客戶端。您可以使用 `passport:client` Artisan 命令的 `--client` 選項來執行此操作：

```shell
php artisan passport:client --client
```

接下來，要使用此授權類型，請為 `CheckClientCredentials` 中介層註冊一個中介層別名。您可以在應用程式的 `bootstrap/app.php` 檔案中定義中介層別名：

    use Laravel\Passport\Http\Middleware\CheckClientCredentials;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'client' => CheckClientCredentials::class
        ]);
    })

然後，將中介層附加到路由：

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client');

若要限制路由的特定 scopes，您可以在將 `client` 中介層附加到路由時，提供一個以逗號分隔的所需 scopes 列表：

    Route::get('/orders', function (Request $request) {
        ...
    })->middleware('client:check-status,your-scope');


<a name="retrieving-tokens"></a>
### 取得憑證

若要使用此授權類型取得憑證，請向 `oauth/token` 端點發出請求：

    use Illuminate\Support\Facades\Http;

    $response = Http::asForm()->post('http://passport-app.test/oauth/token', [
        'grant_type' => 'client_credentials',
        'client_id' => 'client-id',
        'client_secret' => 'client-secret',
        'scope' => 'your-scope',
    ]);

    return $response->json()['access_token'];


<a name="personal-access-tokens"></a>
## 個人存取憑證

有時，您的使用者可能希望在不經過典型授權碼重導向流程的情況下，自行發行存取憑證。允許使用者透過應用程式的 UI 自行發行憑證，這對於讓使用者試用您的 API 很有用，或者通常可以作為發行存取憑證的更簡單方法。

> [!NOTE]  
> 如果您的應用程式主要使用 Passport 發行個人存取憑證，請考慮使用 [Laravel Sanctum](/docs/{{version}}/sanctum)，這是 Laravel 用於發行 API 存取憑證的輕量級第一方函式庫。


<a name="creating-a-personal-access-client"></a>
### 建立個人存取客戶端

在您的應用程式能發行個人存取憑證之前，您需要建立一個個人存取客戶端。您可以透過執行帶有 `--personal` 選項的 `passport:client` Artisan 命令來執行此操作。如果您已經執行過 `passport:install` 命令，則不需要再執行此命令：

```shell
php artisan passport:client --personal
```

建立個人存取客戶端後，請將客戶端的 ID 和純文字密鑰值放在應用程式的 `.env` 檔案中：

```ini
PASSPORT_PERSONAL_ACCESS_CLIENT_ID="client-id-value"
PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET="unhashed-client-secret-value"
```


<a name="managing-personal-access-tokens"></a>
### 管理個人存取憑證

建立個人存取客戶端後，您可以使用 `App\Models\User` 模型實例上的 `createToken` 方法為給定的使用者發行憑證。`createToken` 方法接受憑證名稱作為第一個引數，以及一個可選的 [scopes](#token-scopes) 陣列作為第二個引數：

    use App\Models\User;

    $user = User::find(1);

    // Creating a token without scopes...
    $token = $user->createToken('Token Name')->accessToken;

    // Creating a token with scopes...
    $token = $user->createToken('My Token', ['place-orders'])->accessToken;


<a name="personal-access-tokens-json-api"></a>
#### JSON API

Passport 還包含一個用於管理個人存取憑證的 JSON API。您可以將其與自己的前端搭配使用，為使用者提供一個管理個人存取憑證的儀表板。下面，我們將回顧所有用於管理個人存取憑證的 API 端點。為方便起見，我們將使用 [Axios](https://github.com/axios/axios) 來演示如何向這些端點發出 HTTP 請求。

該 JSON API 受 `web` 和 `auth` 中介層保護；因此，它只能從您自己的應用程式中呼叫。它不能從外部來源呼叫。


<a name="get-oauthscopes"></a>
#### `GET /oauth/scopes`

此路由返回為您的應用程式定義的所有 [scopes](#token-scopes)。您可以使用此路由來列出使用者可以分配給個人存取憑證的 scopes：

```js
axios.get('/oauth/scopes')
    .then(response => {
        console.log(response.data);
    });
```


<a name="get-oauthpersonal-access-tokens"></a>
#### `GET /oauth/personal-access-tokens`

此路由返回已認證使用者建立的所有個人存取憑證。這主要用於列出使用者所有的憑證，以便他們可以編輯或撤銷它們：

```js
axios.get('/oauth/personal-access-tokens')
    .then(response => {
        console.log(response.data);
    });
```


<a name="post-oauthpersonal-access-tokens"></a>
#### `POST /oauth/personal-access-tokens`

此路由建立新的個人存取憑證。它需要兩項資料：憑證的 `name` 和應分配給憑證的 `scopes`：

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

此路由可用於撤銷個人存取憑證：

```js
axios.delete('/oauth/personal-access-tokens/' + tokenId);
```

<a name="protecting-routes"></a>
## 保護路由

<a name="via-middleware"></a>
### 透過 Middleware

Passport 包含一個 [認證 Guard](/docs/{{version}}/authentication#adding-custom-guards)，它將驗證傳入請求中的存取憑證。一旦您將 `api` Guard 設定為使用 `passport` 驅動，您只需在任何需要有效存取憑證的路由上指定 `auth:api` Middleware 即可：

    Route::get('/user', function () {
        // ...
    })->middleware('auth:api');

> [!WARNING]  
> 如果您正在使用[客戶端憑證授權類型](#client-credentials-grant-tokens)，您應該使用[`client` Middleware](#client-credentials-grant-tokens) 來保護您的路由，而不是 `auth:api` Middleware。

<a name="multiple-authentication-guards"></a>
#### 多個認證 Guard

如果您的應用程式驗證不同類型的使用者，他們可能使用完全不同的 Eloquent 模型，您可能需要在應用程式中為每種使用者 Provider 類型定義一個 Guard 設定。這讓您可以保護針對特定使用者 Provider 的請求。例如，假設 `config/auth.php` 設定檔中有以下 Guard 設定：

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],

    'api-customers' => [
        'driver' => 'passport',
        'provider' => 'customers',
    ],

以下路由將利用 `api-customers` Guard (它使用 `customers` 使用者 Provider) 來驗證傳入的請求：

    Route::get('/customer', function () {
        // ...
    })->middleware('auth:api-customers');

> [!NOTE]  
> 有關在 Passport 中使用多個使用者 Provider 的更多資訊，請查閱 [密碼授權類型文件](#customizing-the-user-provider)。

<a name="passing-the-access-token"></a>
### 傳遞存取憑證

在呼叫受 Passport 保護的路由時，應用程式的 API 客戶端應該在請求的 `Authorization` 標頭中將其存取憑證指定為 `Bearer` 憑證。例如，當使用 Guzzle HTTP 函式庫時：

    use Illuminate\Support\Facades\Http;

    $response = Http::withHeaders([
        'Accept' => 'application/json',
        'Authorization' => 'Bearer '.$accessToken,
    ])->get('https://passport-app.test/api/user');

    return $response->json();

<a name="token-scopes"></a>
## 憑證 Scope

Scope 允許您的 API 客戶端在請求授權存取帳戶時，請求一組特定的權限。例如，如果您正在建置一個電商應用程式，並非所有 API 客戶端都需要下訂單的能力。相反地，您可以允許客戶端僅請求授權來存取訂單運送狀態。換句話說，Scope 允許您的應用程式使用者限制第三方應用程式可以代表他們執行的動作。

<a name="defining-scopes"></a>
### 定義 Scope

您可以在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中，使用 `Passport::tokensCan` 方法定義您的 API Scope。`tokensCan` 方法接受一個 Scope 名稱和 Scope 描述的陣列。Scope 描述可以是您想要的任何內容，並將顯示在授權核准畫面上給使用者。

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

如果客戶端未請求任何特定 Scope，您可以設定 Passport 伺服器，使用 `setDefaultScope` 方法將預設 Scope 附加到憑證。通常，您應該從應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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
> Passport 的預設 Scope 不適用於使用者產生的個人存取憑證。

<a name="assigning-scopes-to-tokens"></a>
### 將 Scope 分配給憑證

<a name="when-requesting-authorization-codes"></a>
#### 請求授權碼時

當使用授權碼授權類型請求存取憑證時，客戶端應將其所需的 Scope 指定為 `scope` 查詢字串參數。`scope` 參數應為以空格分隔的 Scope 列表：

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
#### 發行個人存取憑證時

如果您使用 `App\Models\User` 模型的 `createToken` 方法發行個人存取憑證，您可以將所需 Scope 的陣列作為方法的第二個參數傳遞：

    $token = $user->createToken('My Token', ['place-orders'])->accessToken;

<a name="checking-scopes"></a>
### 檢查 Scope

Passport 包含兩個 Middleware，可用於驗證傳入請求是否已使用授予特定 Scope 的憑證進行認證。若要開始使用，請在應用程式的 `bootstrap/app.php` 檔案中定義以下 Middleware 別名：

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

可將 `scopes` Middleware 分配給路由，以驗證傳入請求的存取憑證是否包含所有列出的 Scope：

    Route::get('/orders', function () {
        // Access token has both "check-status" and "place-orders" scopes...
    })->middleware(['auth:api', 'scopes:check-status,place-orders']);

<a name="check-for-any-scopes"></a>
#### 檢查任何 Scope

可將 `scope` Middleware 分配給路由，以驗證傳入請求的存取憑證是否包含列出的 Scope 中的**至少一個**：

    Route::get('/orders', function () {
        // Access token has either "check-status" or "place-orders" scope...
    })->middleware(['auth:api', 'scope:check-status,place-orders']);

<a name="checking-scopes-on-a-token-instance"></a>
#### 檢查憑證實例上的 Scope

一旦存取憑證認證的請求進入您的應用程式，您仍然可以使用已認證的 `App\Models\User` 實例上的 `tokenCan` 方法檢查憑證是否具有特定的 Scope：

    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        if ($request->user()->tokenCan('place-orders')) {
            // ...
        }
    });

<a name="additional-scope-methods"></a>
#### 額外的 Scope 方法

`scopeIds` 方法將返回所有已定義 ID / 名稱的陣列：

    use Laravel\Passport\Passport;

    Passport::scopeIds();

`scopes` 方法將返回所有已定義 Scope 的陣列，作為 `Laravel\Passport\Scope` 的實例：

    Passport::scopes();

`scopesFor` 方法將返回與給定 ID / 名稱匹配的 `Laravel\Passport\Scope` 實例陣列：

    Passport::scopesFor(['place-orders', 'check-status']);

您可以使用 `hasScope` 方法判斷是否已定義特定的 Scope：

    Passport::hasScope('place-orders');

<a name="consuming-your-api-with-javascript"></a>
## 透過 JavaScript 呼叫您的 API

在建構 API 時，能夠從您的 JavaScript 應用程式呼叫您自己的 API 會非常有幫助。這種 API 開發方法允許您自己的應用程式呼叫您與世界共享的相同 API。相同的 API 可以被您的網路應用程式、行動應用程式、第三方應用程式以及您可能發佈在各種套件管理程式上的任何 SDK 呼叫。

通常，如果您想從 JavaScript 應用程式呼叫您的 API，您需要手動將存取憑證傳送到應用程式，並在每次請求時將其傳遞給您的應用程式。然而，Passport 包含一個中介層可以為您處理這項工作。您只需要將 `CreateFreshApiToken` 中介層附加到應用程式的 `bootstrap/app.php` 檔案中的 `web` 中介層群組：

    use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            CreateFreshApiToken::class,
        ]);
    })

> [!WARNING]  
> 您應該確保 `CreateFreshApiToken` 中介層是您的中介層堆疊中最後列出的中介層。

此中介層會將一個 `laravel_token` cookie 附加到您的傳出回應中。此 cookie 包含一個加密的 JWT，Passport 將使用它來驗證來自您的 JavaScript 應用程式的 API 請求。此 JWT 的存留期與您的 `session.lifetime` 設定值相同。現在，由於瀏覽器會自動將 cookie 與所有後續請求一起傳送，您可以向應用程式的 API 發出請求，而無需明確傳遞存取憑證：

    axios.get('/api/user')
        .then(response => {
            console.log(response.data);
        });


<a name="customizing-the-cookie-name"></a>
#### 自訂 Cookie 名稱

如果需要，您可以使用 `Passport::cookie` 方法自訂 `laravel_token` cookie 的名稱。通常，此方法應在應用程式的 `App\Providers\AppServiceProvider` 類別的 `boot` 方法中呼叫：

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Passport::cookie('custom_name');
    }


<a name="csrf-protection"></a>
#### CSRF 保護

當使用此驗證方法時，您需要確保您的請求中包含一個有效的 CSRF 權杖標頭。預設的 Laravel JavaScript 鷹架包含一個 Axios 實例，它會自動使用加密的 `XSRF-TOKEN` cookie 值在同源請求中傳送 `X-XSRF-TOKEN` 標頭。

> [!NOTE]  
> 如果您選擇傳送 `X-CSRF-TOKEN` 標頭而不是 `X-XSRF-TOKEN`，您需要使用 `csrf_token()` 提供的未加密權杖。


<a name="events"></a>
## 事件

Passport 在發行存取憑證和刷新憑證時會觸發事件。您可以[監聽這些事件](/docs/{{version}}/events)以清除或撤銷資料庫中的其他存取憑證：

<div class="overflow-auto">

| 事件名稱 |
| --- |
| `Laravel\Passport\Events\AccessTokenCreated` |
| `Laravel\Passport\Events\RefreshTokenCreated` |

</div>


<a name="testing"></a>
## 測試

Passport 的 `actingAs` 方法可用於指定當前已驗證的使用者及其 Scope。傳給 `actingAs` 方法的第一個參數是使用者實例，第二個是應授予使用者憑證的 Scope 陣列：

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

Passport 的 `actingAsClient` 方法可用於指定當前已驗證的客戶端及其 Scope。傳給 `actingAsClient` 方法的第一個參數是客戶端實例，第二個是應授予客戶端憑證的 Scope 陣列：

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