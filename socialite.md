# Laravel Socialite

- [簡介](#introduction)
- [安裝](#installation)
- [升級 Socialite](#upgrading-socialite)
- [設定](#configuration)
- [認證](#authentication)
    - [路由](#routing)
    - [認證與儲存](#authentication-and-storage)
    - [存取範圍](#access-scopes)
    - [Slack Bot 存取範圍](#slack-bot-scopes)
    - [選用參數](#optional-parameters)
- [擷取使用者資訊](#retrieving-user-details)

<a name="introduction"></a>
## 簡介

除了常見的表單認證之外，Laravel 也透過使用 [Laravel Socialite](https://github.com/laravel/socialite) 提供了一個簡單便捷的方式來使用 OAuth 供應商進行認證。Socialite 目前支援透過 Facebook、X、LinkedIn、Google、GitHub、GitLab、Bitbucket 與 Slack 進行認證。

> [!NOTE]  
> 其他平台的適配器可透過社群驅動的 [Socialite Providers](https://socialiteproviders.com/) 網站取得。


<a name="installation"></a>
## 安裝

要開始使用 Socialite，請使用 Composer 套件管理工具將此套件新增到您的專案依賴中：

```shell
composer require laravel/socialite
```


<a name="upgrading-socialite"></a>
## 升級 Socialite

升級到 Socialite 的主要新版本時，請務必仔細查閱[升級指南](https://github.com/laravel/socialite/blob/master/UPGRADE.md)。


<a name="configuration"></a>
## 設定

在使用 Socialite 之前，您需要為應用程式使用的 OAuth 供應商新增憑證。通常，這些憑證可以透過在您將進行認證的服務的儀表板中建立一個「開發者應用程式」來取得。

這些憑證應放置在應用程式的 `config/services.php` 設定檔中，並應根據您應用程式所需的供應商，使用 `facebook`、`x`、`linkedin-openid`、`google`、`github`、`gitlab`、`bitbucket`、`slack` 或 `slack-openid` 作為鍵名：

    'github' => [
        'client_id' => env('GITHUB_CLIENT_ID'),
        'client_secret' => env('GITHUB_CLIENT_SECRET'),
        'redirect' => 'http://example.com/callback-url',
    ],

> [!NOTE]  
> 如果 `redirect` 選項包含相對路徑，它將自動解析為完整的 URL。


<a name="authentication"></a>
## 認證


<a name="routing"></a>
### 路由

要使用 OAuth 供應商認證使用者，您需要兩個路由：一個用於將使用者重定向到 OAuth 供應商，另一個用於在認證後從供應商接收回呼。以下範例路由展示了這兩個路由的實作方式：

    use Laravel\Socialite\Facades\Socialite;

    Route::get('/auth/redirect', function () {
        return Socialite::driver('github')->redirect();
    });

    Route::get('/auth/callback', function () {
        $user = Socialite::driver('github')->user();

        // $user->token
    });

由 `Socialite` facade 提供的 `redirect` 方法負責將使用者重定向到 OAuth 供應商，而 `user` 方法將檢查傳入的請求，並在使用者批准認證請求後從供應商擷取使用者資訊。


<a name="authentication-and-storage"></a>
### 認證與儲存

從 OAuth 供應商擷取使用者後，您可以判斷使用者是否存在於您的應用程式資料庫中，並[認證該使用者](/docs/{{version}}/authentication#authenticate-a-user-instance)。如果使用者不存在於您的應用程式資料庫中，您通常會在資料庫中建立一個新紀錄來代表該使用者：

    use App\Models\User;
    use Illuminate\Support\Facades\Auth;
    use Laravel\Socialite\Facades\Socialite;

    Route::get('/auth/callback', function () {
        $githubUser = Socialite::driver('github')->user();

        $user = User::updateOrCreate([
            'github_id' => $githubUser->id,
        ], [
            'name' => $githubUser->name,
            'email' => $githubUser->email,
            'github_token' => $githubUser->token,
            'github_refresh_token' => $githubUser->refreshToken,
        ]);

        Auth::login($user);

        return redirect('/dashboard');
    });

> [!NOTE]  
> 有關從特定 OAuth 供應商可取得哪些使用者資訊的更多資訊，請查閱[擷取使用者資訊](#retrieving-user-details)文件。


<a name="access-scopes"></a>
### 存取範圍

在重定向使用者之前，您可以使用 `scopes` 方法來指定應包含在認證請求中的「存取範圍」。此方法會將所有先前指定的存取範圍與您指定的存取範圍合併：

    use Laravel\Socialite\Facades\Socialite;

    return Socialite::driver('github')
        ->scopes(['read:user', 'public_repo'])
        ->redirect();

您可以使用 `setScopes` 方法來覆寫認證請求上所有現有的存取範圍：

    return Socialite::driver('github')
        ->setScopes(['read:user', 'public_repo'])
        ->redirect();


<a name="slack-bot-scopes"></a>
### Slack Bot 存取範圍

Slack 的 API 提供了[不同類型的存取權杖](https://api.slack.com/authentication/token-types)，每個都有自己的一組[權限存取範圍](https://api.slack.com/scopes)。Socialite 與以下兩種 Slack 存取權杖類型相容：

<div class="content-list" markdown="1">

- Bot (前綴為 `xoxb-`)
- User (前綴為 `xoxp-`)

</div>

預設情況下，`slack` 驅動器會生成一個 `user` 權杖，並呼叫該驅動器的 `user` 方法將傳回使用者的詳細資訊。

Bot 權杖主要用於您的應用程式向由應用程式使用者擁有的外部 Slack 工作區發送通知的情況。要生成 bot 權杖，請在將使用者重定向到 Slack 進行認證之前，呼叫 `asBotUser` 方法：

    return Socialite::driver('slack')
        ->asBotUser()
        ->setScopes(['chat:write', 'chat:write.public', 'chat:write.customize'])
        ->redirect();

此外，在 Slack 將使用者重定向回您的應用程式進行認證後，您必須在呼叫 `user` 方法之前呼叫 `asBotUser` 方法：

    $user = Socialite::driver('slack')->asBotUser()->user();

當生成 bot 權杖時，`user` 方法仍然會傳回一個 `Laravel\Socialite\Two\User` 實例；然而，只有 `token` 屬性會被填充。此權杖可以儲存起來，以便[向已認證使用者的 Slack 工作區發送通知](/docs/{{version}}/notifications#notifying-external-slack-workspaces)。


<a name="optional-parameters"></a>
### 選用參數

許多 OAuth 供應商支援在重定向請求中加入其他選用參數。要在請求中包含任何選用參數，請使用關聯陣列呼叫 `with` 方法：

    use Laravel\Socialite\Facades\Socialite;

    return Socialite::driver('google')
        ->with(['hd' => 'example.com'])
        ->redirect();

> [!WARNING]  
> 使用 `with` 方法時，請注意不要傳遞任何保留關鍵字，例如 `state` 或 `response_type`。

<a name="retrieving-user-details"></a>
## 擷取使用者資訊

當使用者被重新導向回您的應用程式的認證回呼路由後，您可以使用 Socialite 的 `user` 方法擷取使用者的詳細資訊。由 `user` 方法回傳的使用者物件提供了多種屬性與方法，您可以用來在您自己的資料庫中儲存使用者的資訊。

根據您正在認證的 OAuth 服務提供者是否支援 OAuth 1.0 或 OAuth 2.0，此物件上可能會提供不同的屬性與方法：

    use Laravel\Socialite\Facades\Socialite;

    Route::get('/auth/callback', function () {
        $user = Socialite::driver('github')->user();

        // OAuth 2.0 providers...
        $token = $user->token;
        $refreshToken = $user->refreshToken;
        $expiresIn = $user->expiresIn;

        // OAuth 1.0 providers...
        $token = $user->token;
        $tokenSecret = $user->tokenSecret;

        // All providers...
        $user->getId();
        $user->getNickname();
        $user->getName();
        $user->getEmail();
        $user->getAvatar();
    });


<a name="retrieving-user-details-from-a-token-oauth2"></a>
#### 從 Token 擷取使用者資訊

如果您已經擁有使用者有效的存取 Token，您可以使用 Socialite 的 `userFromToken` 方法擷取他們的使用者詳細資訊：

    use Laravel\Socialite\Facades\Socialite;

    $user = Socialite::driver('github')->userFromToken($token);

如果您是透過 iOS 應用程式使用 Facebook 有限登入，Facebook 將會回傳 OIDC token 而非存取 Token。如同存取 Token，OIDC token 也可以提供給 `userFromToken` 方法以擷取使用者詳細資訊。


<a name="stateless-authentication"></a>
#### 無狀態認證

`stateless` 方法可用來停用 Session 狀態驗證。這在將社群認證功能新增到不使用基於 Cookie 的 Session 的無狀態 API 時非常有用：

    use Laravel\Socialite\Facades\Socialite;

    return Socialite::driver('google')->stateless()->user();