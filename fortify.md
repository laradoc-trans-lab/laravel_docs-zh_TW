# Laravel Fortify

- [簡介](#introduction)
    - [什麼是 Fortify？](#what-is-fortify)
    - [我應該在什麼時候使用 Fortify？](#when-should-i-use-fortify)
- [安裝](#installation)
    - [Fortify 功能](#fortify-features)
    - [停用視圖](#disabling-views)
- [認證](#authentication)
    - [自訂使用者認證](#customizing-user-authentication)
    - [自訂認證管道](#customizing-the-authentication-pipeline)
    - [自訂重導向](#customizing-authentication-redirects)
- [雙重認證](#two-factor-authentication)
    - [啟用雙重認證](#enabling-two-factor-authentication)
    - [使用雙重認證進行認證](#authenticating-with-two-factor-authentication)
    - [停用雙重認證](#disabling-two-factor-authentication)
- [通行密鑰 (Passkeys)](#passkeys)
    - [啟用通行密鑰](#enabling-passkeys)
    - [JavaScript 用戶端](#passkeys-javascript-client)
    - [使用通行密鑰進行認證](#authenticating-with-passkeys)
    - [使用通行密鑰確認密碼](#confirming-password-with-passkeys)
    - [註冊通行密鑰](#registering-passkeys)
    - [刪除通行密鑰](#deleting-passkeys)
- [註冊](#registration)
    - [自訂註冊](#customizing-registration)
- [密碼重設](#password-reset)
    - [請求密碼重設連結](#requesting-a-password-reset-link)
    - [重設密碼](#resetting-the-password)
    - [自訂密碼重設](#customizing-password-resets)
- [電子郵件驗證](#email-verification)
    - [保護路由](#protecting-routes)
- [密碼確認](#password-confirmation)

<a name="introduction"></a>
## 簡介

[Laravel Fortify](https://github.com/laravel/fortify) 是一個不限定前端 (frontend agnostic) 的 Laravel 認證後端實作。Fortify 註冊了實作所有 Laravel 認證功能所需的路由與控制器，包括登入、註冊、密碼重設、電子郵件驗證等。安裝 Fortify 後，你可以執行 `route:list` Artisan 指令來查看 Fortify 註冊的路由。

由於 Fortify 不提供自己的使用者介面，因此它旨在與你自己的使用者介面搭配使用，由你的介面向其註冊的路由發送請求。我們將在本文後續內容中詳細討論如何向這些路由發送請求。

> [!NOTE]
> 請記住，Fortify 是一個旨在讓你快速開始實作 Laravel 認證功能的套件。**你並非一定要使用它。** 你隨時可以參考 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 以及 [電子郵件驗證](/docs/{{version}}/verification) 的說明文件，手動與 Laravel 的認證服務進行互動。


<a name="what-is-fortify"></a>
### 什麼是 Fortify？

如前所述，Laravel Fortify 是一個不限定前端的 Laravel 認證後端實作。Fortify 註冊了實作所有 Laravel 認證功能所需的路由與控制器，包括登入、註冊、密碼重設、電子郵件驗證等。

**為了使用 Laravel 的認證功能，你並非一定要使用 Fortify。** 你隨時可以參考 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 以及 [電子郵件驗證](/docs/{{version}}/verification) 的說明文件，手動與 Laravel 的認證服務進行互動。

如果你是 Laravel 的新手，你可能想先探索[我們的應用程式入門套件](/docs/{{version}}/starter-kits)。Laravel 的應用程式入門套件在內部使用 Fortify，為你的應用程式提供包含 [Tailwind CSS](https://tailwindcss.com) 構建的使用者介面在內的認證腳手架。這可以讓你學習並熟悉 Laravel 的認證功能。

Laravel Fortify 本質上是將應用程式入門套件中的路由與控制器提取出來，並以不包含使用者介面的套件形式提供。這讓你可以快速建構應用程式認證層的後端實作，而不會被任何特定的前端既定設計慣例所束縛。


<a name="when-should-i-use-fortify"></a>
### 我應該在什麼時候使用 Fortify？

你可能會想知道什麼時候適合使用 Laravel Fortify。首先，如果你正在使用 Laravel 的[應用程式入門套件](/docs/{{version}}/starter-kits)之一，則不需要安裝 Laravel Fortify，因為所有的 Laravel 應用程式入門套件都已經使用了 Fortify 並提供了完整的認證實作。

如果你沒有使用入門套件，且你的應用程式需要認證功能，你有兩個選擇：手動實作應用程式的認證功能，或者使用 Laravel Fortify 來提供這些功能的後端實作。

如果你選擇安裝 Fortify，你的使用者介面將向本文件中詳細說明的 Fortify 認證路由發送請求，以便對使用者進行認證與註冊。

如果你選擇手動與 Laravel 的認證服務互動而不使用 Fortify，你可以參考 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 以及 [電子郵件驗證](/docs/{{version}}/verification) 的說明文件來進行。


<a name="laravel-fortify-and-laravel-sanctum"></a>
#### Laravel Fortify 與 Laravel Sanctum

有些開發者會對 [Laravel Sanctum](/docs/{{version}}/sanctum) 與 Laravel Fortify 之間的區別感到困惑。因為這兩個套件解決的是兩個不同但相關的問題，所以 Laravel Fortify 與 Laravel Sanctum 並不是互斥或競爭的套件。

Laravel Sanctum 僅關注於管理 API 令牌 (API tokens)，以及使用 Session Cookie 或令牌對現有使用者進行認證。Sanctum 不提供任何處理使用者註冊、密碼重設等功能的路由。

如果你正嘗試手動為提供 API 或作為單頁面應用程式 (SPA) 後端的應用程式建構認證層，你很有可能同時使用 Laravel Fortify（用於使用者註冊、密碼重設等）與 Laravel Sanctum（API 令牌管理、Session 認證）。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理員安裝 Fortify：

```shell
composer require laravel/fortify
```

接著，使用 `fortify:install` Artisan 指令發布 Fortify 的資源：

```shell
php artisan fortify:install
```

此指令會將 Fortify 的 Action 發布到你的 `app/Actions` 目錄中（如果該目錄不存在則會自動建立）。此外，還會發布 `FortifyServiceProvider`、設定檔以及所有必要的資料庫遷移。

接著，你應該執行資料庫遷移：

```shell
php artisan migrate
```


<a name="fortify-features"></a>
### Fortify 功能

`fortify` 設定檔包含一個 `features` 設定陣列。此陣列定義了 Fortify 預設會公開哪些後端路由與功能。我們建議你僅啟用以下功能，這些是大多數 Laravel 應用程式提供的基礎認證功能：

```php
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
],
```


<a name="disabling-views"></a>
### 停用視圖

預設情況下，Fortify 定義了旨在回傳視圖的路由，例如登入畫面或註冊畫面。但是，如果你正在建構一個由 JavaScript 驅動的單頁面應用程式，你可能不需要這些路由。因此，你可以透過將應用程式 `config/fortify.php` 設定檔中的 `views` 設定值設為 `false` 來完全停用這些路由：

```php
'views' => false,
```


<a name="disabling-views-and-password-reset"></a>
#### 停用視圖與密碼重設

如果你選擇停用 Fortify 的視圖，且你將為應用程式實作密碼重設功能，你仍然應該定義一個名為 `password.reset` 的路由，該路由負責顯示應用程式的「重設密碼」視圖。這是必要的，因為 Laravel 的 `Illuminate\Auth\Notifications\ResetPassword` 通知將透過 `password.reset` 命名路由來產生密碼重設 URL。

<a name="authentication"></a>
## 認證

首先，我們需要指示 Fortify 如何回傳我們的「登入」視圖。請記住，Fortify 是一個無介面 (Headless) 的認證函式庫。如果您想要一份已經為您完成的 Laravel 認證功能前端實作，您應該使用[入門套件 (Starter Kits)](/docs/{{version}}/starter-kits)。

所有的認證視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的對應方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別中的 `boot` 方法呼叫此方法。Fortify 會負責定義回傳此視圖的 `/login` 路由：

```php
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::loginView(function () {
        return view('auth.login');
    });

    // ...
}
```

您的登入模板應包含一個向 `/login` 發送 POST 請求的表單。`/login` 端點預期接收字串類型的 `email` / `username` 以及 `password`。email / username 欄位的名稱應與 `config/fortify.php` 設定檔中的 `username` 值相符。此外，還可以提供一個布林類型的 `remember` 欄位，用以表示使用者希望使用 Laravel 提供的「記住我」功能。

如果登入嘗試成功，Fortify 會將您重導向至應用程式 `fortify` 設定檔中 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，則會回傳 200 HTTP 回應。

如果請求失敗，使用者將被重導向回登入畫面，且驗證錯誤將透過共享的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors)提供給您。或者在 XHR 請求的情況下，驗證錯誤將隨 422 HTTP 回應一起回傳。


<a name="customizing-user-authentication"></a>
### 自訂使用者認證

Fortify 會根據提供的憑據和為您的應用程式配置的認證 Guard 自動取得並認證使用者。然而，有時您可能希望完全自訂登入憑據的認證方式以及使用者的取得方式。幸運的是，Fortify 讓您可以使用 `Fortify::authenticateUsing` 方法輕鬆實現這一點。

此方法接受一個接收傳入 HTTP 請求的閉包 (Closure)。該閉包負責驗證請求中附帶的登入憑據並回傳關聯的使用者實例。如果憑據無效或找不到使用者，閉包應回傳 `null` 或 `false`。通常，此方法應在 `FortifyServiceProvider` 的 `boot` 方法中呼叫：

```php
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::authenticateUsing(function (Request $request) {
        $user = User::where('email', $request->email)->first();

        if ($user &&
            Hash::check($request->password, $user->password)) {
            return $user;
        }
    });

    // ...
}
```


<a name="authentication-guard"></a>
#### 認證 Guard

您可以在應用程式的 `fortify` 設定檔中自訂 Fortify 使用的認證 Guard。但是，您應確保配置的 Guard 是 `Illuminate\Contracts\Auth\StatefulGuard` 的實作。如果您嘗試使用 Laravel Fortify 來認證 SPA，則應將 Laravel 預設的 `web` Guard 與 [Laravel Sanctum](https://laravel.com/docs/sanctum) 結合使用。


<a name="customizing-the-authentication-pipeline"></a>
### 自訂認證管道

Laravel Fortify 透過一個由可呼叫類別組成的管道來認證登入請求。如果您願意，可以定義一個自訂類別管道，讓登入請求流經該管道。每個類別都應該有一個 `__invoke` 方法，該方法接收傳入的 `Illuminate\Http\Request` 實例，並且像[中介層](/docs/{{version}}/middleware)一樣，接收一個 `$next` 變數，呼叫該變數以將請求傳遞給管道中的下一個類別。

要定義自訂管道，您可以使用 `Fortify::authenticateThrough` 方法。此方法接受一個閉包，該閉包應回傳登入請求要流經的類別陣列。通常，此方法應在應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫。

下方的範例包含預設的管道定義，您可以在進行自訂修改時將其作為起點：

```php
use Laravel\Fortify\Actions\AttemptToAuthenticate;
use Laravel\Fortify\Actions\CanonicalizeUsername;
use Laravel\Fortify\Actions\EnsureLoginIsNotThrottled;
use Laravel\Fortify\Actions\PrepareAuthenticatedSession;
use Laravel\Fortify\Actions\RedirectIfTwoFactorAuthenticatable;
use Laravel\Fortify\Features;
use Laravel\Fortify\Fortify;
use Illuminate\Http\Request;

Fortify::authenticateThrough(function (Request $request) {
    return array_filter([
            config('fortify.limiters.login') ? null : EnsureLoginIsNotThrottled::class,
            config('fortify.lowercase_usernames') ? CanonicalizeUsername::class : null,
            Features::enabled(Features::twoFactorAuthentication()) ? RedirectIfTwoFactorAuthenticatable::class : null,
            AttemptToAuthenticate::class,
            PrepareAuthenticatedSession::class,
    ]);
});
```


#### 認證節流

預設情況下，Fortify 會使用 `EnsureLoginIsNotThrottled` 中介層對認證嘗試進行節流 (Throttling)。此中介層會針對使用者名稱與 IP 位址組合唯一的嘗試進行節流。

某些應用程式可能需要不同的認證嘗試節流方式，例如僅按 IP 位址進行節流。因此，Fortify 允許您透過 `fortify.limiters.login` 設定選項指定自己的[速率限制器 (Rate Limiter)](/docs/{{version}}/routing#rate-limiting)。當然，此設定選項位於應用程式的 `config/fortify.php` 設定檔中。

> [!NOTE]
> 結合使用節流、[雙重認證](/docs/{{version}}/fortify#two-factor-authentication)以及外部 Web 應用程式防火牆 (WAF)，將為您的合法應用程式使用者提供最強大的防禦。


<a name="customizing-authentication-redirects"></a>
### 自訂重導向

如果登入嘗試成功，Fortify 會將您重導向至應用程式 `fortify` 設定檔中 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，則會回傳 200 HTTP 回應。在使用者登出應用程式後，使用者將被重導向至 `/` URI。

如果您需要對此行為進行進階自訂，可以將 `LoginResponse` 和 `LogoutResponse` 契約 (Contracts) 的實作綁定到 Laravel [服務容器(service container)](/docs/{{version}}/container) 中。通常，這應該在應用程式 `App\Providers\FortifyServiceProvider` 類別的 `register` 方法中完成：

```php
use Laravel\Fortify\Contracts\LogoutResponse;

/**
 * Register any application services.
 */
public function register(): void
{
    $this->app->instance(LogoutResponse::class, new class implements LogoutResponse {
        public function toResponse($request)
        {
            return redirect('/');
        }
    });
}
```

<a name="two-factor-authentication"></a>
## 雙重認證

當 Fortify 的雙重認證功能啟用時，使用者在認證過程中會被要求輸入一個六位數的數位令牌。這個令牌是使用基於時間的一次性密碼 (TOTP) 生成的，可以從任何與 TOTP 相容的行動裝置認證應用程式（如 Google Authenticator）中取得。

在開始之前，您應該先確保應用程式的 `App\Models\User` 模型使用了 `Laravel\Fortify\TwoFactorAuthenticatable` trait：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\TwoFactorAuthenticatable;

class User extends Authenticatable
{
    use Notifiable, TwoFactorAuthenticatable;
}
```

接著，您應該在應用程式中建立一個畫面，讓使用者可以管理他們的雙重認證設定。這個畫面應該允許使用者啟用與停用雙重認證，以及重新產生他們的雙重認證復原碼。

> 預設情況下，`fortify` 設定檔中的 `features` 陣列會指示 Fortify 的雙重認證設定在修改前需要進行密碼確認。因此，您的應用程式應該在繼續之前先實作 Fortify 的 [密碼確認](#password-confirmation) 功能。


<a name="enabling-two-factor-authentication"></a>
### 啟用雙重認證

要開始啟用雙重認證，您的應用程式應該向 Fortify 定義的 `/user/two-factor-authentication` 端點發送一個 POST 請求。如果請求成功，使用者將被重導向回上一個 URL，且 `status` 區段 (session) 變數將被設置為 `two-factor-authentication-enabled`。您可以在模板中偵測此 `status` 變數以顯示相應的成功訊息。如果請求是 XHR 請求，則會回傳 `200` HTTP 回應。

在選擇啟用雙重認證後，使用者仍必須透過提供一個有效的雙重認證碼來「確認」他們的雙重認證設定。因此，您的「成功」訊息應該指示使用者仍需要進行雙重認證確認：

```html
@if (session('status') == 'two-factor-authentication-enabled')
    <div class="mb-4 font-medium text-sm">
        Please finish configuring two-factor authentication below.
    </div>
@endif
```

接下來，您應該顯示雙重認證 QR code，供使用者掃描至其認證應用程式中。如果您使用 Blade 來渲染應用程式的前端，可以使用使用者實例上的 `twoFactorQrCodeSvg` 方法來取得 QR code 的 SVG：

```php
$request->user()->twoFactorQrCodeSvg();
```

如果您正在開發以 JavaScript 驅動的前端，可以向 `/user/two-factor-qr-code` 端點發送 XHR GET 請求，以取得使用者的雙重認證 QR code。該端點將回傳一個包含 `svg` 鍵值的 JSON 物件。


<a name="confirming-two-factor-authentication"></a>
#### 確認雙重認證

除了顯示使用者的雙重認證 QR code 之外，您還應該提供一個文字輸入框，讓使用者輸入有效的認證碼以「確認」他們的雙重認證設定。此代碼應透過 POST 請求傳送到 Fortify 定義的 `/user/confirmed-two-factor-authentication` 端點。

如果請求成功，使用者將被重導向回上一個 URL，且 `status` 區段變數將被設置為 `two-factor-authentication-confirmed`：

```html
@if (session('status') == 'two-factor-authentication-confirmed')
    <div class="mb-4 font-medium text-sm">
        Two-factor authentication confirmed and enabled successfully.
    </div>
@endif
```

如果向雙重認證確認端點發送的是 XHR 請求，則會回傳 `200` HTTP 回應。


<a name="displaying-the-recovery-codes"></a>
#### 顯示復原碼

您也應該顯示使用者的雙重認證復原碼。這些復原碼允許使用者在失去行動裝置存取權限時進行認證。如果您使用 Blade 來渲染應用程式的前端，可以透過已認證的使用者實例存取復原碼：

```php
(array) $request->user()->recoveryCodes()
```

如果您正在開發以 JavaScript 驅動的前端，可以向 `/user/two-factor-recovery-codes` 端點發送 XHR GET 請求。該端點將回傳一個包含使用者復原碼的 JSON 陣列。

若要重新產生使用者的復原碼，您的應用程式應該向 `/user/two-factor-recovery-codes` 端點發送 POST 請求。


<a name="authenticating-with-two-factor-authentication"></a>
### 使用雙重認證進行認證

在認證過程中，Fortify 會自動將使用者重導向到應用程式的雙重認證挑戰畫面。然而，如果您的應用程式是發送 XHR 登入請求，則在成功認證後回傳的 JSON 回應將包含一個具有 `two_factor` 布林屬性的 JSON 物件。您應該檢查此值，以判斷是否需要重導向到應用程式的雙重認證挑戰畫面。

要開始實作雙重認證功能，我們需要指示 Fortify 如何回傳我們的雙重認證挑戰視圖。所有 Fortify 的認證視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::twoFactorChallengeView(function () {
        return view('auth.two-factor-challenge');
    });

    // ...
}
```

Fortify 會負責定義回傳此視圖的 `/two-factor-challenge` 路由。您的 `two-factor-challenge` 模板應包含一個向 `/two-factor-challenge` 端點發送 POST 請求的表單。`/two-factor-challenge` 動作預期接收一個包含有效 TOTP 令牌的 `code` 欄位，或者一個包含使用者復原碼之一的 `recovery_code` 欄位。

如果登入嘗試成功，Fortify 會將使用者重導向到應用程式 `fortify` 設定檔中 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，則會回傳 204 HTTP 回應。

如果請求不成功，使用者將被重導向回雙重認證挑戰畫面，您可以透過共享的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 取得驗證錯誤。或者，在 XHR 請求的情況下，驗證錯誤將隨 422 HTTP 回應一起回傳。


<a name="disabling-two-factor-authentication"></a>
### 停用雙重認證

要停用雙重認證，您的應用程式應該向 `/user/two-factor-authentication` 端點發送一個 DELETE 請求。請記住，Fortify 的雙重認證端點在呼叫前需要進行 [密碼確認](#password-confirmation)。

<a name="passkeys"></a>
## 通行密鑰 (Passkeys)

Fortify 支援使用 WebAuthn 的通行密鑰 (passkey) 認證。通行密鑰讓使用者可以使用平台驗證器（如 Face ID、Touch ID、Windows Hello 或硬體安全金鑰）在不使用密碼的情況下進行認證。

<a name="enabling-passkeys"></a>
### 啟用通行密鑰

要開始使用，請確保在應用程式的 `fortify` 設定檔中啟用了 `passkeys` 功能：

```php
use Laravel\Fortify\Features;

'features' => [
    // ...
    Features::passkeys([
        'confirmPassword' => true,
    ]),
],
```

`confirmPassword` 選項決定了 Fortify 是否在註冊或刪除通行密鑰之前要求[密碼確認](#password-confirmation)。

接著，確保應用程式的 `App\Models\User` 模型實作了 `Laravel\Fortify\Contracts\PasskeyUser` 並使用了 `Laravel\Fortify\PasskeyAuthenticatable` trait：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\Contracts\PasskeyUser;
use Laravel\Fortify\PasskeyAuthenticatable;

class User extends Authenticatable implements PasskeyUser
{
    use Notifiable, PasskeyAuthenticatable;
}
```

Fortify 的通行密鑰設定選項可以透過應用程式 `config/fortify.php` 檔案中的 `passkeys` 設定陣列進行自訂：

```php
'passkeys' => [
    'relying_party_id' => parse_url(config('app.url'), PHP_URL_HOST),
    'allowed_origins' => [config('app.url')],
    'user_handle_secret' => config('app.key'),
    'timeout' => 60000,
],
```

> [!NOTE]
> Fortify 封裝了 `laravel/passkeys` Composer 套件並為您完成設定。如果您正在使用 Fortify 的通行密鑰功能，您應該使用應用程式的 `config/fortify.php` 檔案來設定通行密鑰。您不需要發布 `laravel/passkeys` 的設定檔，在那裡定義的任何值都將被 Fortify 覆蓋。

`relying_party_id` 應與您的應用程式網域相符。`allowed_origins` 陣列列出了可以完成通行密鑰註冊和認證的瀏覽器來源。`user_handle_secret` 用於推導不透明的使用者識別碼，確保在不同的通行密鑰註冊中識別出同一使用者。`timeout` 選項控制通行密鑰註冊和認證操作可以保持啟用的時間。

Fortify 對其通行密鑰登入、確認和註冊路由套用了專用的通行密鑰速率限制器。如果需要，您可以使用 `fortify.limiters.passkeys` 設定選項和對應的 `RateLimiter::for(...)` 定義來進行自訂。

<a name="passkeys-javascript-client"></a>
### JavaScript 用戶端

如果您正在建構自訂的前端，包括帶有瀏覽器端腳本的 Blade 應用程式，您可以使用官方的 [`@laravel/passkeys`](https://www.npmjs.com/package/@laravel/passkeys) 套件。此套件處理瀏覽器的 WebAuthn 儀式並向 Fortify 的通行密鑰端點發送請求。

透過 npm 安裝套件：

```shell
npm install @laravel/passkeys
```

接著，您可以從前端啟動通行密鑰註冊與驗證：

```js
import { Passkeys } from "@laravel/passkeys";

await Passkeys.register({ name: "MacBook Pro" });
await Passkeys.verify();
```

如果您的應用程式使用自訂的通行密鑰端點 URI，您可以在每次呼叫時覆寫路由：

```js
await Passkeys.verify({
    routes: {
        options: "/passkeys/confirm/options",
        submit: "/passkeys/confirm",
    },
});

await Passkeys.register({
    name: "MacBook Pro",
    routes: {
        options: "/user/passkeys/options",
        submit: "/user/passkeys",
    },
});
```

該套件還透過 `@laravel/passkeys/react`、`@laravel/passkeys/vue` 和 `@laravel/passkeys/svelte` 提供 React、Vue 和 Svelte 的輔助函式。

<a name="authenticating-with-passkeys"></a>
### 使用通行密鑰進行認證

若要使用通行密鑰認證使用者，您的應用程式應首先向 `/passkeys/login/options` 端點發送一個 GET 請求。此端點會回傳 WebAuthn 挑戰選項，您的前端應將其傳遞給 `navigator.credentials.get(...)`。

瀏覽器回傳憑證後，您的應用程式應向 `/passkeys/login` 發送一個包含憑證負載的 POST 請求。您也可以包含一個布林值 `remember` 欄位。

如果請求成功，Fortify 將把使用者登入到設定的 guard 並回傳：

<div class="content-list" markdown="1">

- 針對標準請求，重導向回應至您的預期目的地。
- 針對 XHR 請求，回傳包含 `redirect` 鍵的 JSON 負載之 `200` HTTP 回應。

</div>

<a name="confirming-password-with-passkeys"></a>
### 使用通行密鑰確認密碼

對於已認證的對談，Fortify 提供了通行密鑰確認端點，以滿足 Laravel 對目前對談的密碼確認要求。

若要使用通行密鑰進行確認，您的應用程式應首先向 `/passkeys/confirm/options` 發送一個 GET 請求。此端點會回傳 WebAuthn 挑戰選項，您的前端應將其傳遞給 `navigator.credentials.get(...)`。

瀏覽器回傳憑證後，您的應用程式應向 `/passkeys/confirm` 發送一個包含憑證負載的 POST 請求。

如果請求成功，Fortify 將目前對談標記為密碼已確認，並回傳：

<div class="content-list" markdown="1">

- 針對標準請求，重導向回應至您的預期目的地。
- 針對 XHR 請求，回傳包含 `redirect` 鍵的 JSON 負載之 `200` HTTP 回應。

</div>

<a name="registering-passkeys"></a>
### 註冊通行密鑰

要為已認證的使用者註冊通行密鑰，您的應用程式應首先向 `/user/passkeys/options` 發送一個 GET 請求。此端點會回傳 WebAuthn 建立選項，您的前端應將其傳遞給 `navigator.credentials.create(...)`。

瀏覽器回傳憑證後，您的應用程式應向 `/user/passkeys` 發送一個 POST 請求，其中包含一個 `name` 欄位和一個 `credential` 欄位，該欄位包含由 `navigator.credentials.create(...)` 回傳的序列化 [`PublicKeyCredential`](https://developer.mozilla.org/en-US/docs/Web/API/PublicKeyCredential) 物件。

如果請求成功，Fortify 將回傳：

<div class="content-list" markdown="1">

- 針對標準請求，回傳一個重導向回應，並在 session 中帶有 `passkey-registered` 狀態。
- 針對 XHR 請求，回傳一個包含 `status` 鍵的 `200` HTTP 回應，以及新註冊通行密鑰的 `id` 和 `name`。

</div>

<a name="deleting-passkeys"></a>
### 刪除通行密鑰

若要刪除通行密鑰，您的應用程式應向 `/user/passkeys/{passkey}` 發送一個 DELETE 請求。

如果請求成功，Fortify 將回傳：

<div class="content-list" markdown="1">

- 針對標準請求，回傳一個重導向回應，並在 session 中帶有 `passkey-deleted` 狀態。
- 針對 XHR 請求，回傳一個包含 `status` 鍵的 `200` HTTP 回應。

</div>

<a name="registration"></a>
## 註冊

要開始實作應用程式的註冊功能，我們需要指示 Fortify 如何回傳我們的 "register" 視圖。請記住，Fortify 是一個無介面 (headless) 的認證函式庫。如果您想要一個已經為您完成的 Laravel 認證功能前端實作，您應該使用[入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::registerView(function () {
        return view('auth.register');
    });

    // ...
}
```

Fortify 會負責定義回傳此視圖的 `/register` 路由。您的 `register` 樣板應該包含一個表單，並向 Fortify 定義的 `/register` 端點發送 POST 請求。

`/register` 端點預期會有字串類型的 `name`、字串類型的電子郵件地址 / 使用者名稱、`password` 以及 `password_confirmation` 欄位。電子郵件 / 使用者名稱欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `username` 設定值相符。

如果註冊嘗試成功，Fortify 將把使用者重導向至應用程式 `fortify` 設定檔中透過 `home` 設定選項所配置的 URI。如果該請求是 XHR 請求，則會回傳 201 HTTP 回應。

如果請求失敗，使用者將被重導回註冊畫面，而驗證錯誤可以透過共享的 `$errors` [Blade 樣板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors)取得。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應一起回傳。


<a name="customizing-registration"></a>
### 自訂註冊

使用者驗證與建立流程可以透過修改安裝 Laravel Fortify 時產生的 `App\Actions\Fortify\CreateNewUser` Action 進行自訂。


<a name="password-reset"></a>
## 密碼重設


<a name="requesting-a-password-reset-link"></a>
### 請求密碼重設連結

要開始實作應用程式的密碼重設功能，我們需要指示 Fortify 如何回傳我們的 "forgot password" 視圖。請記住，Fortify 是一個無介面 (headless) 的認證函式庫。如果您想要一個已經為您完成的 Laravel 認證功能前端實作，您應該使用[入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::requestPasswordResetLinkView(function () {
        return view('auth.forgot-password');
    });

    // ...
}
```

Fortify 會負責定義回傳此視圖的 `/forgot-password` 端點。您的 `forgot-password` 樣板應該包含一個表單，並向 `/forgot-password` 端點發送 POST 請求。

`/forgot-password` 端點預期會有一個字串類型的 `email` 欄位。此欄位 / 資料庫欄位的名稱應與應用程式 `fortify` 設定檔中的 `email` 設定值相符。


<a name="handling-the-password-reset-link-request-response"></a>
#### 處理密碼重設連結請求回應

如果密碼重設連結請求成功，Fortify 將把使用者重導回 `/forgot-password` 端點，並向使用者發送一封包含可用於重設密碼的安全連結電子郵件。如果該請求是 XHR 請求，則會回傳 200 HTTP 回應。

在請求成功並被重導回 `/forgot-password` 端點後，可以使用 `status` Session 變數來顯示密碼重設連結請求嘗試的狀態。

`$status` Session 變數的值將與應用程式 `passwords` [語言檔](/docs/{{version}}/localization)中定義的翻譯字串之一相符。如果您想自訂此值且尚未發布 Laravel 的語言檔，可以透過 `lang:publish` Artisan 指令來完成：

```html
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果請求失敗，使用者將被重導回請求密碼重設連結的畫面，而驗證錯誤可以透過共享的 `$errors` [Blade 樣板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors)取得。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應一起回傳。


<a name="resetting-the-password"></a>
### 重設密碼

為了完成應用程式密碼重設功能的實作，我們需要指示 Fortify 如何回傳我們的 "reset password" 視圖。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Fortify\Fortify;
use Illuminate\Http\Request;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::resetPasswordView(function (Request $request) {
        return view('auth.reset-password', ['request' => $request]);
    });

    // ...
}
```

Fortify 會負責定義顯示此視圖的路由。您的 `reset-password` 樣板應該包含一個表單，並向 `/reset-password` 發送 POST 請求。

`/reset-password` 端點預期會有一個字串類型的 `email` 欄位、一個 `password` 欄位、一個 `password_confirmation` 欄位，以及一個名為 `token` 的隱藏欄位，其中包含 `request()->route('token')` 的值。「email」欄位 / 資料庫欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `email` 設定值相符。


<a name="handling-the-password-reset-response"></a>
#### 處理密碼重設回應

如果密碼重設請求成功，Fortify 將重導回 `/login` 路由，以便使用者可以使用新密碼登入。此外，還會設定一個 `status` Session 變數，以便您在登入畫面上顯示重設成功的狀態：

```blade
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果該請求是 XHR 請求，則會回傳 200 HTTP 回應。

如果請求失敗，使用者將被重導回復重設密碼畫面，而驗證錯誤可以透過共享的 `$errors` [Blade 樣板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors)取得。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應一起回傳。


<a name="customizing-password-resets"></a>
### 自訂密碼重設

密碼重設流程可以透過修改安裝 Laravel Fortify 時產生的 `App\Actions\ResetUserPassword` Action 進行自訂。

<a name="email-verification"></a>
## 電子郵件驗證

註冊後，您可能希望使用者在繼續存取您的應用程式之前驗證其電子郵件地址。首先，請確保在 `fortify` 設定檔的 `features` 陣列中啟用了 `emailVerification` 功能。接下來，您應確保您的 `App\Models\User` 類別實作了 `Illuminate\Contracts\Auth\MustVerifyEmail` 介面。

完成這兩個設定步驟後，新註冊的使用者將收到一封電子郵件，提示他們驗證其電子郵件地址的所有權。然而，我們需要告知 Fortify 如何顯示電子郵件驗證畫面，該畫面會告知使用者他們需要點擊郵件中的驗證連結。

所有的 Fortify 視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::verifyEmailView(function () {
        return view('auth.verify-email');
    });

    // ...
}
```

當使用者被 Laravel 內建的 `verified` 中介層重導向到 `/email/verify` 端點時，Fortify 將負責定義顯示此視圖的路由。

您的 `verify-email` 範本應包含一條資訊訊息，指示使用者點擊發送到其電子郵件地址的驗證連結。

<a name="resending-email-verification-links"></a>
#### 重新發送電子郵件驗證連結

如果您願意，可以在應用程式的 `verify-email` 範本中添加一個按鈕，該按鈕會觸發對 `/email/verification-notification` 端點的 POST 請求。當此端點收到請求時，將向使用者發送一封新的驗證郵件連結，以便使用者在之前的連結被意外刪除或遺失時可以獲得新的連結。

如果重新發送驗證連結郵件的請求成功，Fortify 將把使用者重導向回 `/email/verify` 端點，並附帶一個 `status` 工作階段 (Session) 變數，讓您可以向使用者顯示一條資訊訊息，告知他們操作成功。如果該請求是 XHR 請求，則會傳回 202 HTTP 回應：

```blade
@if (session('status') == 'verification-link-sent')
    <div class="mb-4 font-medium text-sm text-green-600">
        A new email verification link has been emailed to you!
    </div>
@endif
```

<a name="protecting-routes"></a>
### 保護路由

若要指定路由或路由群組需要使用者已驗證其電子郵件地址，您應該將 Laravel 內建的 `verified` 中介層附加到該路由上。`verified` 中介層別名由 Laravel 自動註冊，並作為 `Illuminate\Auth\Middleware\EnsureEmailIsVerified` 中介層的別名：

```php
Route::get('/dashboard', function () {
    // ...
})->middleware(['verified']);
```

<a name="password-confirmation"></a>
## 密碼確認

在開發應用程式時，您偶爾會有一些需要在執行操作之前要求使用者確認密碼的操作。通常，這些路由受到 Laravel 內建的 `password.confirm` 中介層保護。

為了開始實作密碼確認功能，我們需要指示 Fortify 如何傳回應用程式的「密碼確認」視圖。請記住，Fortify 是一個無前端 (headless) 的認證函式庫。如果您想要一個已經為您完成的 Laravel 認證功能的前端實作，您應該使用[入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

```php
use Laravel\Fortify\Fortify;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Fortify::confirmPasswordView(function () {
        return view('auth.confirm-password');
    });

    // ...
}
```

Fortify 將負責定義傳回此視圖的 `/user/confirm-password` 端點。您的 `confirm-password` 範本應包含一個向 `/user/confirm-password` 端點發送 POST 請求的表單。`/user/confirm-password` 端點預期有一個包含使用者目前密碼的 `password` 欄位。

如果密碼與使用者目前的密碼相符，Fortify 將把使用者重導向到他們原本嘗試存取的路由。如果該請求是 XHR 請求，則會傳回 201 HTTP 回應。

如果請求失敗，使用者將被重導向回密碼確認畫面，並且您可以透過共用的 `$errors` Blade 範本變數取得驗證錯誤。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應一起傳回。