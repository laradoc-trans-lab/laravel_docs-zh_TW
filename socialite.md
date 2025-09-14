# Laravel Socialite

- [簡介](#introduction)
- [安裝](#installation)
- [升級 Socialite](#upgrading-socialite)
- [設定](#configuration)
- [認證](#authentication)
    - [路由](#routing)
    - [認證與儲存](#authentication-and-storage)
    - [存取範圍 (Scopes)](#access-scopes)
    - [Slack Bot 範圍 (Scopes)](#slack-bot-scopes)
    - [選用參數](#optional-parameters)
- [取得使用者詳細資訊](#retrieving-user-details)

<a name="introduction"></a>
## 簡介

除了典型的表單式認證外，Laravel 也透過 [Laravel Socialite](https://github.com/laravel/socialite) 提供了一種簡單、方便的方式，讓您可以使用 OAuth 供應商進行認證。Socialite 目前支援透過 Facebook、X、LinkedIn、Google、GitHub、GitLab、Bitbucket 和 Slack 進行認證。

> [!NOTE]
> 其他平台的轉接器可透過社群驅動的 [Socialite Providers](https://socialiteproviders.com/) 網站取得。

<a name="installation"></a>
## 安裝

要開始使用 Socialite，請使用 Composer 套件管理器將套件新增至專案的依賴項：

```shell
composer require laravel/socialite
```

<a name="upgrading-socialite"></a>
## 升級 Socialite

當升級到 Socialite 的新主要版本時，請務必仔細查閱[升級指南](https://github.com/laravel/socialite/blob/master/UPGRADE.md)。

<a name="configuration"></a>
## 設定

在使用 Socialite 之前，您需要為應用程式使用的 OAuth 供應商新增憑證。通常，這些憑證可以透過在您將要認證的服務儀表板中建立「開發者應用程式」來取得。

這些憑證應放置在應用程式的 `config/services.php` 設定檔中，並應使用 `facebook`、`x`、`linkedin-openid`、`google`、`github`、`gitlab`、`bitbucket`、`slack` 或 `slack-openid` 作為鍵，具體取決於應用程式所需的供應商：

```php
'github' => [
    'client_id' => env('GITHUB_CLIENT_ID'),
    'client_secret' => env('GITHUB_CLIENT_SECRET'),
    'redirect' => 'http://example.com/callback-url',
],
```

> [!NOTE]
> 如果 `redirect` 選項包含相對路徑，它將自動解析為完整的 URL。

<a name="authentication"></a>
## 認證

<a name="routing"></a>
### 路由

要使用 OAuth 供應商認證使用者，您需要兩個路由：一個用於將使用者重新導向到 OAuth 供應商，另一個用於在認證後接收來自供應商的回呼。以下範例路由展示了這兩個路由的實作：

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/redirect', function () {
    return Socialite::driver('github')->redirect();
});

Route::get('/auth/callback', function () {
    $user = Socialite::driver('github')->user();

    // $user->token
});
```

`Socialite` Facade 提供的 `redirect` 方法負責將使用者重新導向到 OAuth 供應商，而 `user` 方法將檢查傳入的請求，並在使用者批准認證請求後從供應商取得使用者資訊。

<a name="authentication-and-storage"></a>
### 認證與儲存

一旦從 OAuth 供應商取得使用者，您可以判斷使用者是否存在於應用程式的資料庫中，並[認證使用者](/docs/{{version}}/authentication#authenticate-a-user-instance)。如果使用者不存在於應用程式的資料庫中，您通常會在資料庫中建立一個新記錄來代表該使用者：

```php
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
```

> [!NOTE]
> 有關特定 OAuth 供應商可用的使用者資訊的更多資訊，請查閱[取得使用者詳細資訊](#retrieving-user-details)的說明文件。

<a name="access-scopes"></a>
### 存取範圍 (Scopes)

在重新導向使用者之前，您可以使用 `scopes` 方法指定應包含在認證請求中的「範圍 (scopes)」。此方法會將所有先前指定的範圍與您指定的範圍合併：

```php
use Laravel\Socialite\Facades\Socialite;

return Socialite::driver('github')
    ->scopes(['read:user', 'public_repo'])
    ->redirect();
```

您可以使用 `setScopes` 方法覆寫認證請求上的所有現有範圍：

```php
return Socialite::driver('github')
    ->setScopes(['read:user', 'public_repo'])
    ->redirect();
```

<a name="slack-bot-scopes"></a>
### Slack Bot 範圍 (Scopes)

Slack 的 API 提供[不同類型的存取權杖](https://api.slack.com/authentication/token-types)，每種權杖都有自己的一組[權限範圍](https://api.slack.com/scopes)。Socialite 與以下兩種 Slack 存取權杖類型相容：

<div class="content-list" markdown="1">

- Bot (前綴為 `xoxb-`)
- User (前綴為 `xoxp-`)

</div>

預設情況下，`slack` 驅動程式將產生一個 `user` 權杖，並且呼叫驅動程式的 `user` 方法將返回使用者的詳細資訊。

Bot 權杖主要用於應用程式將通知傳送到由應用程式使用者擁有的外部 Slack 工作區。要產生 Bot 權杖，請在將使用者重新導向到 Slack 進行認證之前呼叫 `asBotUser` 方法：

```php
return Socialite::driver('slack')
    ->asBotUser()
    ->setScopes(['chat:write', 'chat:write.public', 'chat:write.customize'])
    ->redirect();
```

此外，在 Slack 將使用者重新導向回您的應用程式進行認證後，您必須在呼叫 `user` 方法之前呼叫 `asBotUser` 方法：

```php
$user = Socialite::driver('slack')->asBotUser()->user();
```

當產生 Bot 權杖時，`user` 方法仍將返回一個 `Laravel\Socialite\Two\User` 實例；但是，只有 `token` 屬性會被填充。此權杖可以儲存起來，以便[將通知傳送到已認證使用者的 Slack 工作區](/docs/{{version}}/notifications#notifying-external-slack-workspaces)。

<a name="optional-parameters"></a>
### 選用參數

許多 OAuth 供應商支援重新導向請求上的其他選用參數。要在請求中包含任何選用參數，請使用關聯陣列呼叫 `with` 方法：

```php
use Laravel\Socialite\Facades\Socialite;

return Socialite::driver('google')
    ->with(['hd' => 'example.com'])
    ->redirect();
```

> [!WARNING]
> 使用 `with` 方法時，請注意不要傳遞任何保留關鍵字，例如 `state` 或 `response_type`。

<a name="retrieving-user-details"></a>
## 取得使用者詳細資訊

在使用者重新導向回應用程式的認證回呼路由後，您可以使用 Socialite 的 `user` 方法取得使用者的詳細資訊。`user` 方法返回的使用者物件提供了各種屬性和方法，您可以用來將使用者資訊儲存到自己的資料庫中。

此物件上可用的屬性和方法可能因您用於認證的 OAuth 供應商是支援 OAuth 1.0 還是 OAuth 2.0 而異：

```php
use Laravel\Socialite\Facades\Socialite;

Route::get('/auth/callback', function () {
    $user = Socialite::driver('github')->user();

    // OAuth 2.0 供應商...
    $token = $user->token;
    $refreshToken = $user->refreshToken;
    $expiresIn = $user->expiresIn;

    // OAuth 1.0 供應商...
    $token = $user->token;
    $tokenSecret = $user->tokenSecret;

    // 所有供應商...
    $user->getId();
    $user->getNickname();
    $user->getName();
    $user->getEmail();
    $user->getAvatar();
});
```

<a name="retrieving-user-details-from-a-token-oauth2"></a>
#### 從權杖取得使用者詳細資訊

如果您已經擁有使用者的有效存取權杖，您可以使用 Socialite 的 `userFromToken` 方法取得其使用者詳細資訊：

```php
use Laravel\Socialite\Facades\Socialite;

$user = Socialite::driver('github')->userFromToken($token);
```

如果您透過 iOS 應用程式使用 Facebook Limited Login，Facebook 將返回 OIDC 權杖而不是存取權杖。與存取權杖一樣，OIDC 權杖可以提供給 `userFromToken` 方法以取得使用者詳細資訊。

<a name="stateless-authentication"></a>
#### 無狀態認證

`stateless` 方法可用於停用會話狀態驗證。這在將社交認證新增到不使用基於 Cookie 會話的無狀態 API 時非常有用：

```php
use Laravel\Socialite\Facades\Socialite;

return Socialite::driver('google')->stateless()->user();
```
