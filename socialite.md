# Laravel Socialite

- [介紹](#introduction)
- [安裝](#installation)
- [升級 Socialite](#upgrading-socialite)
- [設定](#configuration)
- [身分驗證](#authentication)
    - [路由](#routing)
    - [身分驗證與儲存](#authentication-and-storage)
    - [存取範圍](#access-scopes)
    - [Slack Bot 範圍](#slack-bot-scopes)
    - [選用參數](#optional-parameters)
- [取得使用者詳情](#retrieving-user-details)
- [測試](#testing)

<a name="introduction"></a>
## 介紹

除了典型的表單身分驗證外，Laravel 還提供了一種簡單、方便的方法，使用 [Laravel Socialite](https://github.com/laravel/socialite) 與 OAuth 供應商進行身分驗證。Socialite 目前支援透過 Facebook、X、LinkedIn、Google、GitHub、GitLab、Bitbucket 以及 Slack 進行身分驗證。

> [!NOTE]
> 其他平台的適配器可以透過社群驅動的 [Socialite Providers](https://socialiteproviders.com/) 網站取得。


<a name="installation"></a>
## 安裝

要開始使用 Socialite，請使用 Composer 套件管理員將該套件新增至專案的依賴中：

```shell
composer require laravel/socialite
```


<a name="upgrading-socialite"></a>
## 升級 Socialite

當升級到 Socialite 的新主版本時，請務必仔細閱讀 [升級指南](https://github.com/laravel/socialite/blob/master/UPGRADE.md)。


<a name="configuration"></a>
## 設定

在開始使用 Socialite 之前，您需要為應用程式所使用的 OAuth 供應商新增憑證。通常，這些憑證可以透過在您要進行身分驗證的服務儀表板中建立「開發者應用程式」來取得。

這些憑證應放在應用程式的 `config/services.php` 設定檔中，並根據應用程式所需的供應商使用金鑰 `facebook`、`x`、`linkedin-openid`、`google`、`github`、`gitlab`、`bitbucket`、`slack` 或 `slack-openid`：

```php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => 'http://example.com/callback-url',
],
```

> [!NOTE]
> 如果 `redirect` 選項包含相對路徑，它將自動被解析為完整的 URL。


<a name="authentication"></a>
## 身分驗證


<a name="routing"></a>
### 路由

要使用 OAuth 供應商對使用者進行身分驗證，您需要兩個路由：一個用於將使用者重新導向到 OAuth 供應商，另一個用於在驗證後接收來自供應商的回呼。下方的範例路由展示了這兩個路由的實作：

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

由 `Socialite` Facade 提供的 `redirect` 方法負責將使用者重新導向到 OAuth 供應商，而 `user` 方法將會檢查傳入的請求，並在使用者核准身分驗證請求後從供應商取得使用者的資訊。


<a name="authentication-and-storage"></a>
### 身分驗證與儲存

從 OAuth 供應商取得使用者後，您可以判斷該使用者是否存在於應用程式的資料庫中，並[驗證該使用者](/docs/{{version}}/authentication#authenticate-a-user-instance)。如果該使用者在應用程式的資料庫中不存在，您通常會在資料庫中建立一筆新記錄來代表該使用者：

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
> 有關特定的 OAuth 供應商可提供哪些使用者資訊的更多詳細資訊，請參閱[取得使用者詳情](#retrieving-user-details)的說明文件。


<a name="access-scopes"></a>
### 存取範圍

在重新導向使用者之前，您可以使用 `scopes` 方法指定應包含在身分驗證請求中的「範圍 (Scopes)」。此方法會將所有先前指定的範圍與您指定的範圍進行合併：

```php
use Laravel\Socialite\Socialite;

return Socialite::driver('github')
    ->scopes(['read:user', 'public_repo'])
    ->redirect();
```

您可以使用 `setScopes` 方法覆寫身分驗證請求上所有現有的範圍：

```php
return Socialite::driver('github')
    ->setScopes(['read:user', 'public_repo'])
    ->redirect();
```


<a name="slack-bot-scopes"></a>
### Slack Bot 範圍

Slack 的 API 提供[不同類型的存取權杖](https://api.slack.com/authentication/token-types)，每種權杖都有自己的[權限範圍](https://api.slack.com/scopes)集。Socialite 相容於以下兩種 Slack 存取權杖類型：

<div class="content-list" markdown="1">

- Bot (字首為 `xoxb-`)
- User (字首為 `xoxp-`)

</div>

預設情況下，`slack` 驅動程式會產生 `user` 權杖，並在調用驅動程式的 `user` 方法時回傳使用者的詳細資訊。

如果您的應用程式要向使用者所擁有的外部 Slack 工作區發送通知，則 Bot 權杖主要非常有用。要產生 Bot 權杖，請在將使用者重新導向到 Slack 進行身分驗證之前調用 `asBotUser` 方法：

```php
return Socialite::driver('slack')
    ->asBotUser()
    ->setScopes(['chat:write', 'chat:write.public', 'chat:write.customize'])
    ->redirect();
```

此外，在 Slack 將使用者重新導向回您的應用程式進行身分驗證後，您必須在調用 `user` 方法之前調用 `asBotUser` 方法：

```php
$user = Socialite::driver('slack')->asBotUser()->user();
```

產生 Bot 權杖時，`user` 方法仍會回傳一個 `Laravel\Socialite\Two\User` 實例；然而，只有 `token` 屬性會被填入數據。此權杖可以被儲存，以便[向已驗證使用者的 Slack 工作區發送通知](/docs/{{version}}/notifications#notifying-external-slack-workspaces)。


<a name="optional-parameters"></a>
### 選用參數

許多 OAuth 供應商在重新導向請求中支援其他選用參數。要在請求中包含任何選用參數，請調用 `with` 方法並傳入一個關聯陣列：

```php
use Laravel\Socialite\Socialite;

return Socialite::driver('google')
    ->with(['hd' => 'example.com'])
    ->redirect();
```

> [!WARNING]
> 使用 `with` 方法時，請注意不要傳入任何保留關鍵字，例如 `state` 或 `response_type`。

<a name="retrieving-user-details"></a>
## 取得使用者詳情

在使用者被重新導向回您的應用程式身分驗證回呼路由後，您可以使用 Socialite 的 `user` 方法取得使用者的詳情。由 `user` 方法回傳的使用者物件提供了多種屬性與方法，您可以用來將使用者資訊儲存到您自己的資料庫中。

根據您進行驗證的 OAuth 提供者是支援 OAuth 1.0 還是 OAuth 2.0，此物件上可用的屬性與方法可能會有所不同：

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
#### 從權杖取得使用者詳情

如果您已經擁有使用者的有效存取權杖 (Access Token)，您可以使用 Socialite 的 `userFromToken` 方法來取得其使用者詳情：

```php
use Laravel\Socialite\Socialite;

$user = Socialite::driver('github')->userFromToken($token);
```

如果您是透過 iOS 應用程式使用 Facebook Limited Login，Facebook 將回傳 OIDC 權杖而非存取權杖。與存取權杖一樣，OIDC 權杖也可以提供給 `userFromToken` 方法以取得使用者詳情。


<a name="stateless-authentication"></a>
#### 無狀態身分驗證

`stateless` 方法可用於停用 Session 狀態驗證。這在將社群身分驗證加入到不使用基於 Cookie 之 Session 的無狀態 API 時非常有用：

```php
use Laravel\Socialite\Socialite;

return Socialite::driver('google')->stateless()->user();
```


<a name="testing"></a>
## 測試

Laravel Socialite 提供了一種便捷的方式來測試 OAuth 身分驗證流程，而無需對 OAuth 提供者發出實際請求。`fake` 方法允許您模擬 OAuth 提供者的行為，並定義應回傳的使用者資料。


<a name="faking-the-redirect"></a>
#### 模擬重新導向

為了測試您的應用程式是否能正確地將使用者重新導向至 OAuth 提供者，您可以在向重新導向路由發出請求之前呼叫 `fake` 方法。這將使 Socialite 回傳一個重新導向至虛構授權網址的結果，而非重新導向至實際的 OAuth 提供者：

```php
use Laravel\Socialite\Socialite;

test('user is redirected to github', function () {
    Socialite::fake('github');

    $response = $this->get('/auth/github/redirect');

    $response->assertRedirect();
});
```


<a name="faking-the-callback"></a>
#### 模擬回呼

為了測試應用程式的回呼路由，您可以呼叫 `fake` 方法並提供一個 `User` 實例，該實例應在應用程式向提供者請求使用者詳情時回傳。`User` 實例可以使用 `map` 方法建立：

```php
use Laravel\Socialite\Socialite;
use Laravel\Socialite\Two\User;

test('user can login with github', function () {
    Socialite::fake('github', (new User)->map([
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

預設情況下，`User` 實例還會包含一個 `token` 屬性。如果需要，您可以手動為 `User` 實例指定額外的屬性：

```php
$fakeUser = (new User)->map([
    'id' => 'github-123',
    'name' => 'Jason Beggs',
    'email' => 'jason@example.com',
])->setToken('fake-token')
  ->setRefreshToken('fake-refresh-token')
  ->setExpiresIn(3600)
  ->setApprovedScopes(['read', 'write'])
```