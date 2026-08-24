# Laravel Socialite

- [簡介](#introduction)
- [安裝](#installation)
- [升級 Socialite](#upgrading-socialite)
- [設定](#configuration)
- [認證](#authentication)
    - [路由](#routing)
    - [認證與儲存](#authentication-and-storage)
    - [存取權限範圍](#access-scopes)
    - [Slack Bot 權限範圍](#slack-bot-scopes)
    - [可選參數](#optional-parameters)
- [取得使用者詳細資料](#retrieving-user-details)
- [測試](#testing)

<a name="introduction"></a>
## 簡介

除了傳統基於表單的認證方式外，Laravel 還透過 [Laravel Socialite](https://github.com/laravel/socialite) 提供了一種簡單且方便的方法來使用 OAuth 提供者進行認證。Socialite 目前支援透過 Facebook、X、LinkedIn、Google、GitHub、GitLab、Bitbucket 以及 Slack 進行認證。

> [!NOTE]
> 其他平台的轉接器 (Adapter) 可以透過社群驅動的 [Socialite Providers](https://socialiteproviders.com/) 網站取得。

<a name="installation"></a>
## 安裝

要開始使用 Socialite，請使用 Composer 套件管理器將該套件新增至專案的依賴項中：

```shell
composer require laravel/socialite
```

<a name="upgrading-socialite"></a>
## 升級 Socialite

當升級至 Socialite 的新主要版本時，請務必仔細審閱[升級指南](https://github.com/laravel/socialite/blob/master/UPGRADE.md)。

<a name="configuration"></a>
## 設定

在使用 Socialite 之前，您需要為應用程式所使用的 OAuth 提供者新增憑證。通常，這些憑證可以在您要進行認證服務的控制台 (Dashboard) 中透過建立「開發者應用程式」來取得。

這些憑證應放置在應用程式的 `config/services.php` 設定檔中，並應根據您應用程式需要的提供者使用鍵名 `facebook`、`x`、`linkedin-openid`、`google`、`github`、`gitlab`、`bitbucket`、`slack` 或 `slack-openid`：

```php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => 'http://example.com/callback-url',
],
```

> [!NOTE]
> 如果 `redirect` 選項包含相對路徑，它將會自動解析為完整的 URL。

<a name="authentication"></a>
## 認證

<a name="routing"></a>
### 路由

要使用 OAuth 提供者為使用者進行認證，您將需要兩個路由：一個用於將使用者重新導向至 OAuth 提供者，另一個用於在認證後接收來自提供者的回呼 (Callback)。以下範例路由展示了這兩個路由的實作方式：

```php
use Laravel\Socialite\Socialite;

Route::get('/auth/redirect', function () {
    return Socialite::driver('github')->redirect();
});

Route::get('/auth/callback', function () {
    $user = Socialite::driver('github')->user();

    // $user->token
});
```

由 `Socialite` Facade 提供的 `redirect` 方法會處理將使用者重新導向至 OAuth 提供者的工作，而 `user` 方法則會在使用者核准認證請求後，檢視傳入的請求並從提供者取得該使用者的資訊。

<a name="authentication-and-storage"></a>
### 認證與儲存

從 OAuth 提供者取得使用者資訊後，您可以檢查該使用者是否存在於應用程式的資料庫中，並[認證該使用者](/docs/{{version}}/authentication#authenticate-a-user-instance)。如果該使用者不存在於您應用程式的資料庫中，通常會在資料庫中建立一筆新紀錄來代表該使用者：

```php
use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Laravel\Socialite\Socialite;

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
```

> [!NOTE]
> 關於可從特定 OAuth 提供者取得哪些使用者資訊的更多細節，請參閱[取得使用者詳細資料](#retrieving-user-details)的說明文件。

<a name="access-scopes"></a>
### 存取權限範圍

在將使用者重新導向之前，您可以使用 `scopes` 方法來指定應包含在認證請求中的「權限範圍 (Scopes)」。該方法會將所有先前指定的權限範圍與您所指定的權限範圍進行合併：

```php
use Laravel\Socialite\Socialite;

return Socialite::driver('github')
    ->scopes(['read:user', 'public_repo'])
    ->redirect();
```

您可以使用 `setScopes` 方法覆寫認證請求上的所有現有權限範圍：

```php
return Socialite::driver('github')
    ->setScopes(['read:user', 'public_repo'])
    ->redirect();
```

<a name="slack-bot-scopes"></a>
### Slack Bot 權限範圍

Slack 的 API 提供[不同類型的存取令牌](https://api.slack.com/authentication/token-types)，每一種都有其專屬的[權限範圍](https://api.slack.com/scopes)集合。Socialite 相容於以下兩種 Slack 存取令牌類型：

<div class="content-list" markdown="1">

- Bot (前綴為 `xoxb-`)
- User (前綴為 `xoxp-`)

</div>

預設情況下，`slack` 驅動程式會產生一個 `user` 令牌，呼叫該驅動程式的 `user` 方法將會回傳使用者的詳細資料。

若您的應用程式需要傳送通知給使用者所擁有的外部 Slack 工作區，Bot 令牌會特別有用。要產生 Bot 令牌，請在將使用者重新導向至 Slack 進行認證前，呼叫 `asBotUser` 方法：

```php
return Socialite::driver('slack')
    ->asBotUser()
    ->setScopes(['chat:write', 'chat:write.public', 'chat:write.customize'])
    ->redirect();
```

此外，在 Slack 認證後將使用者重新導向回您的應用程式時，您必須在呼叫 `user` 方法之前先呼叫 `asBotUser` 方法：

```php
$user = Socialite::driver('slack')->asBotUser()->user();
```

當產生 Bot 令牌時，`user` 方法仍會回傳 `Laravel\Socialite\Two\User` 實例；然而，其中僅有 `token` 屬性會填入數值。儲存此令牌可用於[傳送通知至該已認證使用者的 Slack 工作區](/docs/{{version}}/notifications#notifying-external-slack-workspaces)。

<a name="optional-parameters"></a>
### 可選參數

許多 OAuth 提供者支援在重新導向請求中帶入其他可選參數。若要在請求中包含任何可選參數，請呼叫 `with` 方法並傳入關聯陣列：

```php
use Laravel\Socialite\Socialite;

return Socialite::driver('google')
    ->with(['hd' => 'example.com'])
    ->redirect();
```

> [!WARNING]
> 使用 `with` 方法時，請注意不要傳入任何保留關鍵字，例如 `state` 或 `response_type`。

<a name="retrieving-user-details"></a>
## 取得使用者詳細資料

當使用者被重新導向回應用程式的認證 Callback 路由後，您可以透過 Socialite 的 `user` 方法取得使用者的詳細資料。`user` 方法傳回的使用者物件提供了多種屬性和方法，讓您可以將使用者的資訊儲存到您自己的資料庫中。

根據您進行認證的 OAuth 提供者支援的是 OAuth 1.0 還是 OAuth 2.0，該物件可用的屬性和方法可能會有所不同：

```php
use Laravel\Socialite\Socialite;

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
```

<a name="retrieving-user-details-from-a-token-oauth2"></a>
#### 從 Token 取得使用者詳細資料

如果您已經擁有使用者的有效存取 token，您可以透過 Socialite 的 `userFromToken` 方法取得其使用者詳細資料：

```php
use Laravel\Socialite\Socialite;

$user = Socialite::driver('github')->userFromToken($token);
```

如果您透過 iOS 應用程式使用 Facebook Limited Login，Facebook 將會回傳 OIDC token 而非存取 token。與存取 token 一樣，您可以將 OIDC token 提供給 `userFromToken` 方法以取得使用者詳細資料。

<a name="stateless-authentication"></a>
#### 無狀態認證

`stateless` 方法可用於停用 Session 狀態驗證。這在將社群認證新增至不使用基於 Cookie 之 Session 的無狀態 API 時非常有用：

```php
use Laravel\Socialite\Socialite;

return Socialite::driver('google')->stateless()->user();
```

<a name="testing"></a>
## 測試

Laravel Socialite 提供了一種便捷的方式來測試 OAuth 認證流程，而不需要向 OAuth 提供者發送實際的請求。`fake` 方法允許您模擬 OAuth 提供者的行為，並定義應該回傳的使用者資料。

<a name="faking-the-redirect"></a>
#### 模擬重新導向

為了測試您的應用程式是否能正確地將使用者重新導向至 OAuth 提供者，您可以在發送請求至重新導向路由之前呼叫 `fake` 方法。這會讓 Socialite 傳回一個重新導向至偽造授權網址的回應，而不是重新導向到實際的 OAuth 提供者：

```php
use Laravel\Socialite\Socialite;

test('user is redirected to github', function () {
    Socialite::fake('github');

    $response = $this->get('/auth/github/redirect');

    $response->assertRedirect();
});
```

<a name="faking-the-callback"></a>
#### 模擬 Callback

為了測試您應用程式的 Callback 路由，您可以呼叫 `fake` 方法，並提供當您的應用程式向提供者請求使用者詳細資料時應該回傳的 `User` 實例。該 `User` 實例可以使用 `fake` 方法建立：

```php
use Laravel\Socialite\Socialite;
use Laravel\Socialite\Two\User;

test('user can login with github', function () {
    Socialite::fake('github', User::fake([
        'id' => 'github-123',
        'name' => 'Jason Beggs',
        'email' => 'jason@example.com',
    ]));

    $response = $this->get('/auth/github/callback');

    $response->assertRedirect('/dashboard');

    $this->assertDatabaseHas('users', [
        'name' => 'Jason Beggs',
        'email' => 'jason@example.com',
        'github_id' => 'github-123',
    ]);
});
```

預設情況下，`User` 實例會包含假的 OAuth token 值。若有需要，您可以透過向 `fake` 方法傳入額外的屬性來覆寫這些值：

```php
$fakeUser = User::fake([
    'id' => 'github-123',
    'name' => 'Jason Beggs',
    'email' => 'jason@example.com',
    'token' => 'fake-token',
    'refreshToken' => 'fake-refresh-token',
    'expiresIn' => 3600,
    'approvedScopes' => ['read', 'write'],
]);
```

OAuth 1 使用者可以使用 `Laravel\Socialite\One\User` 類別進行模擬。