# Laravel Fortify

- [介紹](#introduction)
    - [什麼是 Fortify？](#what-is-fortify)
    - [我應該何時使用 Fortify？](#when-should-i-use-fortify)
- [安裝](#installation)
    - [Fortify 功能](#fortify-features)
    - [停用視圖](#disabling-views)
- [認證](#authentication)
    - [自訂使用者認證](#customizing-user-authentication)
    - [自訂認證流程](#customizing-the-authentication-pipeline)
    - [自訂重新導向](#customizing-authentication-redirects)
- [兩步驟驗證](#two-factor-authentication)
    - [啟用兩步驟驗證](#enabling-two-factor-authentication)
    - [使用兩步驟驗證進行認證](#authenticating-with-two-factor-authentication)
    - [停用兩步驟驗證](#disabling-two-factor-authentication)
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
## 介紹

[Laravel Fortify](https://github.com/laravel/fortify) 是 Laravel 的一個前端無關的認證後端實作。Fortify 註冊了實作 Laravel 所有認證功能所需的路由與控制器，包括登入、註冊、密碼重設、電子郵件驗證等等。安裝 Fortify 後，您可以執行 `route:list` Artisan 命令來查看 Fortify 已註冊的路由。

由於 Fortify 不提供自己的使用者介面，因此它旨在與您自己的使用者介面搭配使用，由該介面向 Fortify 註冊的路由發出請求。在本文件的其餘部分，我們將詳細討論如何向這些路由發出請求。

> [!NOTE]
> 請記住，Fortify 是一個旨在讓您快速上手實作 Laravel 認證功能的套件。**您並非必須使用它。** 您始終可以透過遵循 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 和 [電子郵件驗證](/docs/{{version}}/verification) 文件中提供的說明，手動與 Laravel 的認證服務互動。


<a name="what-is-fortify"></a>
### 什麼是 Fortify？

如前所述，Laravel Fortify 是 Laravel 的一個前端無關的認證後端實作。Fortify 註冊了實作 Laravel 所有認證功能所需的路由與控制器，包括登入、註冊、密碼重設、電子郵件驗證等等。

**您並非必須使用 Fortify 才能使用 Laravel 的認證功能。** 您始終可以透過遵循 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 和 [電子郵件驗證](/docs/{{version}}/verification) 文件中提供的說明，手動與 Laravel 的認證服務互動。

如果您是 Laravel 的新手，建議您在使用 Laravel Fortify 之前，先探索 [我們的應用程式啟動套件](/docs/{{version}}/starter-kits)。我們的啟動套件為您的應用程式提供了一個認證骨架，其中包括一個使用 [Tailwind CSS](https://tailwindcss.tailwindcss.com) 建構的使用者介面。這讓您可以在 Laravel Fortify 為您實作這些功能之前，先學習並熟悉 Laravel 的認證功能。

Laravel Fortify 本質上是將我們應用程式啟動套件的路由與控制器，作為一個不包含使用者介面的套件提供。這讓您仍然可以快速為應用程式的認證層搭建後端實作，而無需受任何特定前端偏好的束縛。


<a name="when-should-i-use-fortify"></a>
### 我應該何時使用 Fortify？

您可能會想知道何時適合使用 Laravel Fortify。首先，如果您正在使用 Laravel 的某個 [應用程式啟動套件](/docs/{{version}}/starter-kits)，則無需安裝 Laravel Fortify，因為所有 Laravel 的應用程式啟動套件都已提供了完整的認證實作。

如果您沒有使用應用程式啟動套件，並且您的應用程式需要認證功能，您有兩種選擇：手動實作應用程式的認證功能，或使用 Laravel Fortify 來提供這些功能的後端實作。

如果您選擇安裝 Fortify，您的使用者介面將向本文件中詳述的 Fortify 認證路由發出請求，以認證及註冊使用者。

如果您選擇手動與 Laravel 的認證服務互動，而不是使用 Fortify，您可以透過遵循 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 和 [電子郵件驗證](/docs/{{version}}/verification) 文件中提供的說明來完成。


<a name="laravel-fortify-and-laravel-sanctum"></a>
#### Laravel Fortify 與 Laravel Sanctum

有些開發者會對 [Laravel Sanctum](/docs/{{version}}/sanctum) 和 Laravel Fortify 之間的差異感到困惑。由於這兩個套件解決的是兩個不同但相關的問題，因此 Laravel Fortify 和 Laravel Sanctum 並非互斥或競爭的套件。

Laravel Sanctum 僅關注管理 API token 以及使用 Session Cookie 或 token 認證現有使用者。Sanctum 不提供任何處理使用者註冊、密碼重設等的路由。

如果您正嘗試為提供 API 或作為單頁應用程式後端的應用程式手動建構認證層，那麼您很有可能同時利用 Laravel Fortify (用於使用者註冊、密碼重設等) 和 Laravel Sanctum (API token 管理、Session 認證)。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝 Fortify：

```shell
composer require laravel/fortify
```

接著，使用 `fortify:install` Artisan 命令發佈 Fortify 的資源：

```shell
php artisan fortify:install
```

此命令會將 Fortify 的 Actions 發佈到您的 `app/Actions` 目錄中，如果該目錄不存在則會建立。此外，`FortifyServiceProvider`、設定檔以及所有必要的資料庫遷移都會被發佈。

接下來，您應該遷移您的資料庫：

```shell
php artisan migrate
```


<a name="fortify-features"></a>
### Fortify 功能

`fortify` 設定檔包含一個 `features` 設定陣列。此陣列定義了 Fortify 預設會公開哪些後端路由 / 功能。我們建議您僅啟用以下功能，這些功能是大多數 Laravel 應用程式提供的基本認證功能：

```php
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
],
```


<a name="disabling-views"></a>
### 停用視圖

預設情況下，Fortify 會定義旨在返回視圖的路由，例如登入畫面或註冊畫面。但是，如果您正在建構由 JavaScript 驅動的單頁應用程式，您可能不需要這些路由。因此，您可以透過將應用程式 `config/fortify.php` 設定檔中的 `views` 設定值設為 `false` 來完全停用這些路由：

```php
'views' => false,
```


<a name="disabling-views-and-password-reset"></a>
#### 停用視圖與密碼重設

如果您選擇停用 Fortify 的視圖，並且將為您的應用程式實作密碼重設功能，您仍應定義一個名為 `password.reset` 的路由，負責顯示您應用程式的「重設密碼」視圖。這是必要的，因為 Laravel 的 `Illuminate\Auth\Notifications\ResetPassword` 通知將透過 `password.reset` 具名路由產生密碼重設 URL。

<a name="authentication"></a>
## 認證

首先，我們需要指示 Fortify 如何回傳我們的「登入」視圖。請記住，Fortify 是一個無前端的認證函式庫。如果您想要一個已完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有認證視圖的渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法。Fortify 將會處理定義回傳此視圖的 `/login` 路由：

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

您的登入範本應包含一個向 `/login` 發送 POST 請求的表單。`/login` 端點預期接收一個字串的 `email` / `username` 和一個 `password`。電子郵件 / 使用者名稱欄位的名稱應與應用程式 `config/fortify.php` 設定檔中的 `username` 值相符。此外，可以提供一個布林值 `remember` 欄位，以表示使用者希望使用 Laravel 提供的「記住我」功能。

如果登入成功，Fortify 將會把您重新導向至應用程式 `fortify` 設定檔中透過 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，則會回傳 200 HTTP 回應。

如果請求不成功，使用者將會被重新導向回登入畫面，且驗證錯誤將會透過共用的 `$errors` [Blade 範本變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將會隨著 422 HTTP 回應一同回傳。


<a name="customizing-user-authentication"></a>
### 自訂使用者認證

Fortify 將會根據提供的憑證以及為您的應用程式配置的認證守衛自動擷取並認證使用者。然而，您有時可能希望完全自訂登入憑證的認證方式和使用者的擷取方式。幸運的是，Fortify 允許您使用 `Fortify::authenticateUsing` 方法輕鬆實現這一點。

此方法接受一個閉包，該閉包會接收傳入的 HTTP 請求。該閉包負責驗證附加到請求的登入憑證，並回傳相關聯的使用者實例。如果憑證無效或找不到使用者，閉包應回傳 `null` 或 `false`。通常，此方法應在您的 `FortifyServiceProvider` 的 `boot` 方法中呼叫：

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
#### 認證守衛

您可以在應用程式的 `fortify` 設定檔中自訂 Fortify 使用的認證守衛。然而，您應該確保配置的守衛是 `Illuminate\Contracts\Auth\StatefulGuard` 的實作。如果您嘗試使用 Laravel Fortify 來認證 SPA，您應該將 Laravel 預設的 `web` 守衛與 [Laravel Sanctum](https://laravel.com/docs/sanctum) 結合使用。


<a name="customizing-the-authentication-pipeline"></a>
### 自訂認證流程

Laravel Fortify 透過可呼叫類別的流程來認證登入請求。如果您願意，您可以定義一個自訂的類別流程，登入請求應透過這些類別。每個類別都應該有一個 `__invoke` 方法，該方法接收傳入的 `Illuminate\Http\Request` 實例，並且，就像 [中介層](/docs/{{version}}/middleware) 一樣，一個 `$next` 變數，該變數被呼叫以將請求傳遞給流程中的下一個類別。

要定義您的自訂流程，您可以使用 `Fortify::authenticateThrough` 方法。此方法接受一個閉包，該閉包應回傳要將登入請求導向通過的類別陣列。通常，此方法應在您的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫。

下面的範例包含預設的流程定義，您可以在進行自己的修改時將其作為起點：

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

預設情況下，Fortify 將使用 `EnsureLoginIsNotThrottled` 中介層對認證嘗試進行節流。這個中介層會對使用者名稱和 IP 位址組合獨有的嘗試進行節流。

某些應用程式可能需要不同的方法來對認證嘗試進行節流，例如僅根據 IP 位址進行節流。因此，Fortify 允許您透過 `fortify.limiters.login` 設定選項指定自己的 [速率限制器](/docs/{{version}}/routing#rate-limiting)。當然，這個設定選項位於您應用程式的 `config/fortify.php` 設定檔中。

> [!NOTE]
> 結合節流、[兩步驟驗證](/docs/{{version}}/fortify#two-factor-authentication) 和外部網路應用程式防火牆 (WAF) 將為您的合法應用程式使用者提供最穩健的防禦。


<a name="customizing-authentication-redirects"></a>
### 自訂重新導向

如果登入嘗試成功，Fortify 將會把您重新導向至應用程式 `fortify` 設定檔中透過 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，則會回傳 200 HTTP 回應。使用者登出應用程式後，將會被重新導向至 `/` URI。

如果您需要此行為的高階自訂，您可以將 `LoginResponse` 和 `LogoutResponse` 契約的實作綁定到 Laravel [服務容器](/docs/{{version}}/container) 中。通常，這應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `register` 方法中完成：

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
## 兩步驟驗證

當 Fortify 的兩步驟驗證功能啟用時，使用者在認證過程中需要輸入一個六位數數字憑證。此憑證是使用基於時間的一次性密碼 (TOTP) 產生，可以從任何與 TOTP 相容的行動驗證器應用程式（例如 Google Authenticator）中取得。

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

接下來，您應該在應用程式中建立一個畫面，讓使用者可以管理其兩步驟驗證設定。此畫面應允許使用者啟用和停用兩步驟驗證，以及重新產生其兩步驟驗證復原碼。

> 依預設，`fortify` 設定檔中的 `features` 陣列會指示 Fortify 的兩步驟驗證設定在修改前需要密碼確認。因此，在繼續之前，您的應用程式應該實作 Fortify 的 [密碼確認](#password-confirmation) 功能。

<a name="enabling-two-factor-authentication"></a>
### 啟用兩步驟驗證

若要開始啟用兩步驟驗證，您的應用程式應該向 Fortify 定義的 `/user/two-factor-authentication` 端點發出 POST 請求。如果請求成功，使用者將會被重新導向回上一個 URL，並且 `status` Session 變數將會設為 `two-factor-authentication-enabled`。您可以在模板中偵測此 `status` Session 變數以顯示適當的成功訊息。如果請求是 XHR 請求，將會回傳 `200` HTTP 回應。

在選擇啟用兩步驟驗證後，使用者仍需提供有效的兩步驟驗證碼來「確認」其兩步驟驗證設定。因此，您的「成功」訊息應指示使用者仍需進行兩步驟驗證確認：

```html
@if (session('status') == 'two-factor-authentication-enabled')
    <div class="mb-4 font-medium text-sm">
        Please finish configuring two factor authentication below.
    </div>
@endif
```

接下來，您應該顯示兩步驟驗證的 QR 碼，供使用者掃描至其驗證器應用程式。如果您使用 Blade 渲染應用程式的前端，您可以透過使用者實例上的 `twoFactorQrCodeSvg` 方法取得 QR 碼的 SVG：

```php
$request->user()->twoFactorQrCodeSvg();
```

如果您正在建置由 JavaScript 驅動的前端，您可以向 `/user/two-factor-qr-code` 端點發出 XHR GET 請求，以取得使用者的兩步驟驗證 QR 碼。此端點將會回傳一個包含 `svg` 鍵的 JSON 物件。

<a name="confirming-two-factor-authentication"></a>
#### 確認兩步驟驗證

除了顯示使用者的兩步驟驗證 QR 碼之外，您應該提供一個文字輸入欄位，讓使用者可以提供有效的驗證碼以「確認」其兩步驟驗證設定。此憑證應透過向 Fortify 定義的 `/user/confirmed-two-factor-authentication` 端點發出 POST 請求來提供給 Laravel 應用程式。

如果請求成功，使用者將會被重新導向回上一個 URL，並且 `status` Session 變數將會設為 `two-factor-authentication-confirmed`：

```html
@if (session('status') == 'two-factor-authentication-confirmed')
    <div class="mb-4 font-medium text-sm">
        Two factor authentication confirmed and enabled successfully.
    </div>
@endif
```

如果向兩步驟驗證確認端點發出的請求是透過 XHR 請求進行的，將會回傳 `200` HTTP 回應。

<a name="displaying-the-recovery-codes"></a>
#### 顯示復原碼

您還應該顯示使用者的兩步驟復原碼。這些復原碼允許使用者在遺失行動裝置時也能進行認證。如果您使用 Blade 渲染應用程式的前端，您可以透過已認證使用者實例存取復原碼：

```php
(array) $request->user()->recoveryCodes()
```

如果您正在建置由 JavaScript 驅動的前端，您可以向 `/user/two-factor-recovery-codes` 端點發出 XHR GET 請求。此端點將會回傳一個包含使用者復原碼的 JSON 陣列。

若要重新產生使用者的復原碼，您的應用程式應該向 `/user/two-factor-recovery-codes` 端點發出 POST 請求。

<a name="authenticating-with-two-factor-authentication"></a>
### 使用兩步驟驗證進行認證

在認證過程中，Fortify 會自動將使用者重新導向至應用程式的兩步驟驗證挑戰畫面。然而，如果您的應用程式發出的是 XHR 登入請求，成功認證後回傳的 JSON 回應將會包含一個具有 `two_factor` 布林屬性的 JSON 物件。您應該檢查此值，以判斷是否應該重新導向至應用程式的兩步驟驗證挑戰畫面。

若要開始實作兩步驟驗證功能，我們需要指示 Fortify 如何回傳我們的兩步驟驗證挑戰視圖。所有 Fortify 的認證視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中可用的適當方法進行自訂。通常，您應該從應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將會負責定義回傳此視圖的 `/two-factor-challenge` 路由。您的 `two-factor-challenge` 模板應包含一個向 `/two-factor-challenge` 端點發出 POST 請求的表單。`/two-factor-challenge` 操作期望一個包含有效 TOTP 憑證的 `code` 欄位，或者一個包含使用者復原碼之一的 `recovery_code` 欄位。

如果登入嘗試成功，Fortify 將會將使用者重新導向至透過應用程式 `fortify` 設定檔中的 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，將會回傳 204 HTTP 回應。

如果請求不成功，使用者將會被重新導向回兩步驟挑戰畫面，並且驗證錯誤將會透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，在 XHR 請求的情況下，驗證錯誤將會隨著 422 HTTP 回應回傳。

<a name="disabling-two-factor-authentication"></a>
### 停用兩步驟驗證

若要停用兩步驟驗證，您的應用程式應該向 `/user/two-factor-authentication` 端點發出 DELETE 請求。請記住，Fortify 的兩步驟驗證端點在呼叫前需要 [密碼確認](#password-confirmation)。

<a name="registration"></a>
## 註冊

為了開始實作應用程式的註冊功能，我們需要指示 Fortify 如何回傳我們的「註冊」視圖。請記住，Fortify 是一個無頭的認證函式庫。如果您想要使用已完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法來自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義回傳此視圖的 `/register` 路由。您的 `register` 模板應該包含一個向 Fortify 定義的 `/register` 端點發出 POST 請求的表單。

`/register` 端點需要一個字串 `name`、字串電子郵件地址/使用者名稱、`password` 和 `password_confirmation` 欄位。電子郵件/使用者名稱欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `username` 設定值相符。

如果註冊成功，Fortify 會將使用者重新導向至應用程式 `fortify` 設定檔中透過 `home` 設定選項所配置的 URI。如果該請求是 XHR 請求，則會回傳 201 HTTP 回應。

如果請求不成功，使用者將被重新導向回註冊畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將與 422 HTTP 回應一同回傳。


<a name="customizing-registration"></a>
### 自訂註冊

使用者驗證和建立過程可以透過修改安裝 Laravel Fortify 時產生的 `App\Actions\Fortify\CreateNewUser` action 來進行自訂。


<a name="password-reset"></a>
## 密碼重設


<a name="requesting-a-password-reset-link"></a>
### 請求密碼重設連結

為了開始實作應用程式的密碼重設功能，我們需要指示 Fortify 如何回傳我們的「忘記密碼」視圖。請記住，Fortify 是一個無頭的認證函式庫。如果您想要使用已完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法來自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義回傳此視圖的 `/forgot-password` 端點。您的 `forgot-password` 模板應該包含一個向 `/forgot-password` 端點發出 POST 請求的表單。

`/forgot-password` 端點需要一個字串 `email` 欄位。此欄位/資料庫欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `email` 設定值相符。


<a name="handling-the-password-reset-link-request-response"></a>
#### 處理密碼重設連結請求的回應

如果密碼重設連結請求成功，Fortify 會將使用者重新導向回 `/forgot-password` 端點，並向使用者傳送一封電子郵件，其中包含一個可用於重設密碼的安全連結。如果該請求是 XHR 請求，則會回傳 200 HTTP 回應。

在成功請求後被重新導向回 `/forgot-password` 端點後，`status` session 變數可用來顯示密碼重設連結請求嘗試的狀態。

`$status` session 變數的值將與應用程式 `passwords` [語系檔](/docs/{{version}}/localization) 中定義的其中一個翻譯字串相符。如果您想自訂此值且尚未發布 Laravel 的語系檔，可以透過 `lang:publish` Artisan 命令進行發布：

```html
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果請求不成功，使用者將被重新導向回請求密碼重設連結畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將與 422 HTTP 回應一同回傳。


<a name="resetting-the-password"></a>
### 重設密碼

為了完成應用程式的密碼重設功能，我們需要指示 Fortify 如何回傳我們的「重設密碼」視圖。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法來自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義顯示此視圖的路由。您的 `reset-password` 模板應該包含一個向 `/reset-password` 發出 POST 請求的表單。

`/reset-password` 端點需要一個字串 `email` 欄位、一個 `password` 欄位、一個 `password_confirmation` 欄位，以及一個名為 `token` 的隱藏欄位，其包含 `request()->route('token')` 的值。「email」欄位/資料庫欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `email` 設定值相符。


<a name="handling-the-password-reset-response"></a>
#### 處理密碼重設的回應

如果密碼重設請求成功，Fortify 會將使用者重新導向回 `/login` 路由，以便使用者可以使用新密碼登入。此外，將設定一個 `status` session 變數，以便您可以在登入畫面顯示重設的成功狀態：

```blade
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果該請求是 XHR 請求，則會回傳 200 HTTP 回應。

如果請求不成功，使用者將被重新導向回重設密碼畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將與 422 HTTP 回應一同回傳。


<a name="customizing-password-resets"></a>
### 自訂密碼重設

密碼重設過程可以透過修改安裝 Laravel Fortify 時產生的 `App\Actions\ResetUserPassword` action 來進行自訂。

<a name="email-verification"></a>
## 電子郵件驗證

註冊後，您可能會希望使用者在繼續存取您的應用程式之前先驗證其電子郵件地址。首先，請確保在 `fortify` 設定檔的 `features` 陣列中啟用 `emailVerification` 功能。接著，您應該確保 `App\Models\User` 類別實作了 `Illuminate\Contracts\Auth\MustVerifyEmail` 介面。

完成這兩個設定步驟後，新註冊的使用者將會收到一封電子郵件，要求他們驗證其電子郵件地址的所有權。然而，我們需要告知 Fortify 如何顯示電子郵件驗證畫面，該畫面會通知使用者他們需要點擊電子郵件中的驗證連結。

Fortify 所有視圖的渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

當使用者透過 Laravel 內建的 `verified` 中介層被重新導向至 `/email/verify` 端點時，Fortify 將會負責定義顯示此視圖的路由。

您的 `verify-email` 模板應該包含一則資訊性訊息，指示使用者點擊寄送至其電子郵件地址的電子郵件驗證連結。

<a name="resending-email-verification-links"></a>
#### 重新寄送電子郵件驗證連結

如果您願意，可以在應用程式的 `verify-email` 模板中新增一個按鈕，觸發對 `/email/verification-notification` 端點的 POST 請求。當此端點收到請求時，將會向使用者電子郵件寄送一個新的驗證連結，讓使用者在先前的連結不小心刪除或遺失時，能夠取得新的驗證連結。

如果重新寄送驗證連結電子郵件的請求成功，Fortify 將會重新導向使用者回到 `/email/verify` 端點，並附帶一個 `status` Session 變數，讓您可以向使用者顯示操作成功的資訊性訊息。如果請求是 XHR 請求，則會傳回 202 HTTP 回應：

```blade
@if (session('status') == 'verification-link-sent')
    <div class="mb-4 font-medium text-sm text-green-600">
        A new email verification link has been emailed to you!
    </div>
@endif
```

<a name="protecting-routes"></a>
### 保護路由

若要指定路由或路由群組需要使用者已驗證其電子郵件地址，您應該將 Laravel 內建的 `verified` 中介層附加到該路由。`verified` 中介層別名由 Laravel 自動註冊，並作為 `Illuminate\Auth\Middleware\EnsureEmailIsVerified` 中介層的別名：

```php
Route::get('/dashboard', function () {
    // ...
})->middleware(['verified']);
```

<a name="password-confirmation"></a>
## 密碼確認

在建構應用程式時，您偶爾可能會遇到某些動作需要在執行前要求使用者確認密碼。通常，這些路由會受到 Laravel 內建的 `password.confirm` 中介層的保護。

若要開始實作密碼確認功能，我們需要告知 Fortify 如何傳回應用程式的「密碼確認」視圖。請記住，Fortify 是一個無頭 (headless) 認證函式庫。如果您想要一個已經為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

Fortify 所有視圖的渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將會負責定義傳回此視圖的 `/user/confirm-password` 端點。您的 `confirm-password` 模板應該包含一個表單，向 `/user/confirm-password` 端點發出 POST 請求。`/user/confirm-password` 端點預期一個包含使用者目前密碼的 `password` 欄位。

如果密碼與使用者目前的密碼相符，Fortify 將會將使用者重新導向到他們試圖存取的路由。如果請求是 XHR 請求，則會傳回 201 HTTP 回應。

如果請求未成功，使用者將會被重新導向回密碼確認畫面，並且驗證錯誤將透過共享的 `$errors` Blade 模板變數提供給您。或者，如果是 XHR 請求，驗證錯誤將會隨 422 HTTP 回應傳回。