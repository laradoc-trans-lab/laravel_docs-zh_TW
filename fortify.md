# Laravel Fortify

- [簡介](#introduction)
    - [什麼是 Fortify？](#what-is-fortify)
    - [何時該使用 Fortify？](#when-should-i-use-fortify)
- [安裝](#installation)
    - [Fortify 功能](#fortify-features)
    - [停用視圖](#disabling-views)
- [認證](#authentication)
    - [自訂使用者認證](#customizing-user-authentication)
    - [自訂認證流程 (Pipeline)](#customizing-the-authentication-pipeline)
    - [自訂重新導向](#customizing-authentication-redirects)
- [雙重認證](#two-factor-authentication)
    - [啟用雙重認證](#enabling-two-factor-authentication)
    - [透過雙重認證進行認證](#authenticating-with-two-factor-authentication)
    - [停用雙重認證](#disabling-two-factor-authentication)
- [註冊](#registration)
    - [自訂註冊](#customizing-registration)
- [密碼重設](#password-reset)
    - [請求密碼重設連結](#requesting-a-password-reset-link)
    - [重設密碼](#resetting-the-password)
    - [自訂密碼重設](#customizing-password-resets)
- [Email 驗證](#email-verification)
    - [保護路由](#protecting-routes)
- [密碼確認](#password-confirmation)

<a name="introduction"></a>
## 簡介

[Laravel Fortify](https://github.com/laravel/fortify) 是一個與前端框架無關的 Laravel 認證後端實作。Fortify 註冊了實作所有 Laravel 認證功能所需的路由與控制器，包含了登入、註冊、密碼重設、Email 驗證等等。安裝 Fortify 後，您可以執行 `route:list` Artisan 指令，查看 Fortify 註冊的所有路由。

由於 Fortify 並不提供其使用者介面，因此它旨在與您自己的使用者介面搭配使用，由該介面對其註冊的路由發出請求。本文件後續部分將確切討論如何對這些路由發出請求。

> [!NOTE]
> 請記住，Fortify 是一個旨在協助您快速開始實作 Laravel 認證功能的套件。**您並非必須使用它。** 您可以隨時透過查閱 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 以及 [Email 驗證](/docs/{{version}}/verification) 等文件，手動與 Laravel 的認證服務互動。


<a name="what-is-fortify"></a>
### 什麼是 Fortify？

如前所述，Laravel Fortify 是一個與前端框架無關的 Laravel 認證後端實作。Fortify 註冊了實作所有 Laravel 認證功能所需的路由與控制器，包含了登入、註冊、密碼重設、Email 驗證等等。

**您並非必須使用 Fortify 才能使用 Laravel 的認證功能。** 您可以隨時透過查閱 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 以及 [Email 驗證](/docs/{{version}}/verification) 等文件，手動與 Laravel 的認證服務互動。

如果您是 Laravel 新手，建議您可以探索 [我們的應用程式入門套件](/docs/{{version}}/starter-kits)。Laravel 的應用程式入門套件內部使用 Fortify 來為您的應用程式提供認證鷹架，其中包含了使用 [Tailwind CSS](https://tailwindcss.com) 建立的使用者介面。這讓您可以學習並熟悉 Laravel 的認證功能。

Laravel Fortify 本質上是將我們的應用程式入門套件中的路由與控制器提取出來，並將其作為一個不包含使用者介面的套件提供。這讓您仍能快速為應用程式的認證層構建後端實作，同時不受任何特定前端框架的限制。


<a name="when-should-i-use-fortify"></a>
### 何時該使用 Fortify？

您可能想知道何時適合使用 Laravel Fortify。首先，如果您正在使用 Laravel 的 [應用程式入門套件](/docs/{{version}}/starter-kits) 之一，則無需安裝 Laravel Fortify，因為所有 Laravel 的應用程式入門套件都已使用 Fortify 並提供了完整的認證實作。

如果您沒有使用應用程式入門套件，且您的應用程式需要認證功能，您有兩種選擇：手動實作應用程式的認證功能，或使用 Laravel Fortify 來提供這些功能的後端實作。

如果您選擇安裝 Fortify，您的使用者介面將對 Fortify 的認證路由發出請求，這些路由在本文件中詳細說明，用於認證與註冊使用者。

如果您選擇手動與 Laravel 的認證服務互動，而非使用 Fortify，您可以透過查閱 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 以及 [Email 驗證](/docs/{{version}}/verification) 等文件來實作。


<a name="laravel-fortify-and-laravel-sanctum"></a>
#### Laravel Fortify 與 Laravel Sanctum

有些開發者會對 [Laravel Sanctum](/docs/{{version}}/sanctum) 與 Laravel Fortify 之間的差異感到困惑。由於這兩個套件解決的是兩個不同但相關的問題，因此 Laravel Fortify 與 Laravel Sanctum 並非互斥或競爭的套件。

Laravel Sanctum 僅關注管理 API 權杖以及使用 Session Cookie 或權杖來認證現有使用者。Sanctum 不提供任何處理使用者註冊、密碼重設等功能的路由。

如果您嘗試為提供 API 或作為單頁應用程式後端的應用程式手動構建認證層，您極有可能同時使用 Laravel Fortify (用於使用者註冊、密碼重設等) 與 Laravel Sanctum (API 權杖管理、Session 認證)。


<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝 Fortify：

```shell
composer require laravel/fortify
```

接著，使用 `fortify:install` Artisan 指令發佈 Fortify 的資源：

```shell
php artisan fortify:install
```

此指令會將 Fortify 的 Actions 發佈到您的 `app/Actions` 目錄，如果該目錄不存在，將會被建立。此外，`FortifyServiceProvider`、設定檔與所有必要的資料庫遷移檔案也將會被發佈。

接著，您應該遷移您的資料庫：

```shell
php artisan migrate
```


<a name="fortify-features"></a>
### Fortify 功能

`fortify` 設定檔包含一個 `features` 設定陣列。此陣列定義了 Fortify 預設將公開的後端路由 / 功能。我們建議您僅啟用以下功能，這些功能是大多數 Laravel 應用程式提供的基本認證功能：

```php
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
],
```


<a name="disabling-views"></a>
### 停用視圖

預設情況下，Fortify 定義了旨在返回視圖的路由，例如登入畫面或註冊畫面。然而，如果您正在構建一個由 JavaScript 驅動的單頁應用程式，您可能不需要這些路由。因此，您可以透過將應用程式 `config/fortify.php` 設定檔中的 `views` 設定值設為 `false` 來完全停用這些路由：

```php
'views' => false,
```


<a name="disabling-views-and-password-reset"></a>
#### 停用視圖與密碼重設

如果您選擇停用 Fortify 的視圖，並且將為您的應用程式實作密碼重設功能，您仍應定義一個名為 `password.reset` 的路由，負責顯示應用程式的「重設密碼」視圖。這是必要的，因為 Laravel 的 `Illuminate\Auth\Notifications\ResetPassword` 通知將透過 `password.reset` 命名路由生成密碼重設 URL。

<a name="authentication"></a>
## 認證

首先，我們需要指示 Fortify 如何回傳我們的「登入」視圖。請記住，Fortify 是一個無前端的認證函式庫。如果你想要一個已經完成的 Laravel 認證功能前端實作，你應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有認證視圖的渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法來自訂。通常，你應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法。Fortify 將會負責定義回傳此視圖的 `/login` 路由：

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

你的登入模板應該包含一個向 `/login` 發送 POST 請求的表單。`/login` 端點預期一個字串 `email` / `username` 和一個 `password`。email / username 欄位的名稱應該與應用程式 `config/fortify.php` 配置檔中的 `username` 值相符。此外，可以提供一個布林 `remember` 欄位，表示使用者希望使用 Laravel 提供的「記住我」功能。

如果登入嘗試成功，Fortify 會將你重新導向到應用程式 `fortify` 配置檔中透過 `home` 配置選項設定的 URI。如果登入請求是 XHR 請求，則會回傳 200 HTTP 響應。

如果請求不成功，使用者將會被重新導向回登入畫面，並且驗證錯誤將會透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給你。或者，如果是 XHR 請求，驗證錯誤將會隨著 422 HTTP 響應一起回傳。

<a name="customizing-user-authentication"></a>
### 自訂使用者認證

Fortify 會根據提供的憑證和為你的應用程式配置的認證守衛自動擷取並認證使用者。但是，你可能偶爾希望對登入憑證如何被認證以及使用者如何被擷取擁有完全的自訂權。幸運的是，Fortify 允許你使用 `Fortify::authenticateUsing` 方法輕鬆實現這一點。

此方法接受一個閉包，該閉包接收傳入的 HTTP 請求。閉包負責驗證附加到請求的登入憑證並回傳相關的使用者實例。如果憑證無效或找不到使用者，閉包應該回傳 `null` 或 `false`。通常，這個方法應該在你的 `FortifyServiceProvider` 的 `boot` 方法中呼叫：

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

你可以在應用程式的 `fortify` 配置檔中自訂 Fortify 使用的認證守衛。但是，你應該確保配置的守衛是 `Illuminate\Contracts\Auth\StatefulGuard` 的實作。如果你嘗試使用 Laravel Fortify 來認證單頁應用程式，你應該結合 [Laravel Sanctum](https://laravel.com/docs/{{version}}/sanctum) 使用 Laravel 預設的 `web` 守衛。

<a name="customizing-the-authentication-pipeline"></a>
### 自訂認證流程 (Pipeline)

Laravel Fortify 透過一連串可呼叫的類別管道 (pipeline) 來認證登入請求。如果你願意，你可以定義一個自訂的類別管道，讓登入請求通過。每個類別都應該有一個 `__invoke` 方法，該方法接收傳入的 `Illuminate\Http\Request` 實例，並且像 [中介層](/docs/{{version}}/middleware) 一樣，有一個 `$next` 變數，用於將請求傳遞給管道中的下一個類別。

若要定義你的自訂管道，你可以使用 `Fortify::authenticateThrough` 方法。此方法接受一個閉包，該閉包應回傳用於管道登入請求的類別陣列。通常，此方法應在你的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫。

以下範例包含預設的管道定義，你可以將其作為進行修改的起點：

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

預設情況下，Fortify 會使用 `EnsureLoginIsNotThrottled` 中介層來限制認證嘗試。此中介層會限制針對使用者名稱和 IP 位址組合的唯一嘗試次數。

有些應用程式可能需要不同的方法來限制認證嘗試，例如僅依據 IP 位址進行限制。因此，Fortify 允許你透過 `fortify.limiters.login` 配置選項指定你自己的 [速率限制器](/docs/{{version}}/routing#rate-limiting)。當然，此配置選項位於應用程式的 `config/fortify.php` 配置檔中。

> [!NOTE]
> 結合使用節流、[雙重認證](/docs/{{version}}/fortify#two-factor-authentication) 和外部 Web 應用程式防火牆 (WAF) 將為你的合法應用程式使用者提供最穩固的防禦。

<a name="customizing-authentication-redirects"></a>
### 自訂重新導向

如果登入嘗試成功，Fortify 會將你重新導向到應用程式 `fortify` 配置檔中透過 `home` 配置選項設定的 URI。如果登入請求是 XHR 請求，則會回傳 200 HTTP 響應。使用者登出應用程式後，將會被重新導向到 `/` URI。

如果你需要對此行為進行進階自訂，你可以將 `LoginResponse` 和 `LogoutResponse` 契約的實作繫結到 Laravel [服務容器](/docs/{{version}}/container) 中。通常，這應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `register` 方法中完成：

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

當啟用 Fortify 的雙重認證功能時，使用者需要在認證過程中輸入一個六位數的數字權杖。這個權杖是使用基於時間的一次性密碼 (TOTP) 生成的，可以從任何與 TOTP 相容的行動認證應用程式（例如 Google Authenticator）中取得。

在開始之前，您應首先確保應用程式的 `App\Models\User` 模型使用了 `Laravel\Fortify\TwoFactorAuthenticatable` trait：

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

接下來，您應該在應用程式中建立一個畫面，讓使用者管理他們的雙重認證設定。這個畫面應該允許使用者啟用和停用雙重認證，以及重新產生他們的雙重認證復原碼。

> 依預設，`fortify` 設定檔的 `features` 陣列指示 Fortify 的雙重認證設定要求在修改前進行密碼確認。因此，您的應用程式應該在繼續之前實作 Fortify 的 [密碼確認](#password-confirmation) 功能。

<a name="enabling-two-factor-authentication"></a>
### 啟用雙重認證

要開始啟用雙重認證，您的應用程式應該向 Fortify 定義的 `/user/two-factor-authentication` 端點發出 POST 請求。如果請求成功，使用者將被重新導向回先前的 URL，並且 `status` session 變數將被設定為 `two-factor-authentication-enabled`。您可以在模板中偵測這個 `status` session 變數來顯示適當的成功訊息。如果請求是 XHR 請求，將會回傳 `200` HTTP 回應。

在選擇啟用雙重認證後，使用者仍必須透過提供有效的雙重認證碼來「確認」他們的雙重認證設定。因此，您的「成功」訊息應該指示使用者仍需要進行雙重認證確認：

```html
@if (session('status') == 'two-factor-authentication-enabled')
    <div class="mb-4 font-medium text-sm">
        Please finish configuring two-factor authentication below.
    </div>
@endif
```

接下來，您應該顯示雙重認證的 QR 碼，供使用者掃描至他們的認證應用程式中。如果您正在使用 Blade 渲染應用程式的前端，您可以透過使用者實例上的 `twoFactorQrCodeSvg` 方法來取得 QR 碼 SVG：

```php
$request->user()->twoFactorQrCodeSvg();
```

如果您正在建立由 JavaScript 驅動的前端，您可以向 `/user/two-factor-qr-code` 端點發出 XHR GET 請求，以取得使用者的雙重認證 QR 碼。此端點將回傳一個包含 `svg` 鍵的 JSON 物件。

<a name="confirming-two-factor-authentication"></a>
#### 確認雙重認證

除了顯示使用者的雙重認證 QR 碼外，您還應該提供一個文字輸入框，讓使用者輸入有效的認證碼來「確認」他們的雙重認證設定。此認證碼應透過向 Fortify 定義的 `/user/confirmed-two-factor-authentication` 端點發出 POST 請求，提供給 Laravel 應用程式。

如果請求成功，使用者將被重新導向回先前的 URL，並且 `status` session 變數將被設定為 `two-factor-authentication-confirmed`：

```html
@if (session('status') == 'two-factor-authentication-confirmed')
    <div class="mb-4 font-medium text-sm">
        Two-factor authentication confirmed and enabled successfully.
    </div>
@endif
```

如果向雙重認證確認端點發出的請求是透過 XHR 請求進行的，將會回傳 `200` HTTP 回應。

<a name="displaying-the-recovery-codes"></a>
#### 顯示復原碼

您也應該顯示使用者的雙重認證復原碼。這些復原碼允許使用者在失去行動裝置的存取權時進行認證。如果您正在使用 Blade 渲染應用程式的前端，您可以透過已認證的使用者實例來存取復原碼：

```php
(array) $request->user()->recoveryCodes()
```

如果您正在建立由 JavaScript 驅動的前端，您可以向 `/user/two-factor-recovery-codes` 端點發出 XHR GET 請求。此端點將回傳一個包含使用者復原碼的 JSON 陣列。

要重新產生使用者的復原碼，您的應用程式應該向 `/user/two-factor-recovery-codes` 端點發出 POST 請求。

<a name="authenticating-with-two-factor-authentication"></a>
### 透過雙重認證進行認證

在認證過程中，Fortify 會自動將使用者重新導向到應用程式的雙重認證挑戰畫面。然而，如果您的應用程式正在發出 XHR 登入請求，成功認證嘗試後回傳的 JSON 回應將包含一個具有 `two_factor` 布林屬性的 JSON 物件。您應該檢查此值以判斷是否需要重新導向到應用程式的雙重認證挑戰畫面。

要開始實作雙重認證功能，我們需要指示 Fortify 如何回傳我們的雙重認證挑戰視圖。Fortify 的所有認證視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中提供的方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 會負責定義回傳此視圖的 `/two-factor-challenge` 路由。您的 `two-factor-challenge` 模板應包含一個向 `/two-factor-challenge` 端點發出 POST 請求的表單。該 `/two-factor-challenge` 動作預期一個包含有效 TOTP 權杖的 `code` 欄位，或一個包含使用者其中一個復原碼的 `recovery_code` 欄位。

如果登入嘗試成功，Fortify 將把使用者重新導向到透過應用程式 `fortify` 設定檔中的 `home` 設定選項所配置的 URI。如果登入請求是 XHR 請求，將會回傳 204 HTTP 回應。

如果請求不成功，使用者將被重新導向回雙重認證挑戰畫面，且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將會隨著 422 HTTP 回應一同回傳。

<a name="disabling-two-factor-authentication"></a>
### 停用雙重認證

要停用雙重認證，您的應用程式應該向 `/user/two-factor-authentication` 端點發出 DELETE 請求。請記住，Fortify 的雙重認證端點在被呼叫之前需要 [密碼確認](#password-confirmation)。

<a name="registration"></a>
## 註冊

為了開始實作應用程式的註冊功能，我們需要指示 Fortify 如何回傳我們的「註冊」視圖。請記住，Fortify 是一個無頭 (headless) 的認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法進行自訂。通常，您應該從應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將會處理定義回傳此視圖的 `/register` 路由。您的 `register` 模板應包含一個向 Fortify 定義的 `/register` 端點發送 POST 請求的表單。

`/register` 端點期望一個字串 `name`、字串電子郵件地址 / 使用者名稱、`password` 和 `password_confirmation` 欄位。電子郵件 / 使用者名稱欄位的名稱應與應用程式的 `fortify` 設定檔中定義的 `username` 設定值相符。

如果註冊成功，Fortify 將會把使用者重新導向到透過應用程式 `fortify` 設定檔中的 `home` 設定選項所設定的 URI。如果請求是 XHR 請求，將會回傳 201 HTTP 回應。

如果請求未成功，使用者將會被重新導向回註冊畫面，並且您可以透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 取得驗證錯誤。或者，如果是 XHR 請求，驗證錯誤將會隨著 422 HTTP 回應一起回傳。

<a name="customizing-registration"></a>
### 自訂註冊

您可以透過修改在安裝 Laravel Fortify 時所產生的 `App\Actions\Fortify\CreateNewUser` 動作來客製化使用者驗證和建立的過程。

<a name="password-reset"></a>
## 密碼重設

<a name="requesting-a-password-reset-link"></a>
### 請求密碼重設連結

為了開始實作應用程式的密碼重設功能，我們需要指示 Fortify 如何回傳我們的「忘記密碼」視圖。請記住，Fortify 是一個無頭 (headless) 的認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法進行自訂。通常，您應該從應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將會處理定義回傳此視圖的 `/forgot-password` 端點。您的 `forgot-password` 模板應包含一個向 `/forgot-password` 端點發送 POST 請求的表單。

`/forgot-password` 端點期望一個字串 `email` 欄位。此欄位 / 資料庫欄位的名稱應與應用程式的 `fortify` 設定檔中定義的 `email` 設定值相符。

<a name="handling-the-password-reset-link-request-response"></a>
#### 處理密碼重設連結請求的回應

如果密碼重設連結請求成功，Fortify 將會把使用者重新導向回 `/forgot-password` 端點，並向使用者發送一封包含安全連結的電子郵件，供他們用來重設密碼。如果請求是 XHR 請求，將會回傳 200 HTTP 回應。

在成功請求後被重新導向回 `/forgot-password` 端點之後，`status` Session 變數可以用來顯示密碼重設連結請求嘗試的狀態。

`$status` Session 變數的值將會與應用程式 `passwords` [語言檔案](/docs/{{version}}/localization) 中定義的翻譯字串之一相符。如果您想要自訂此值但尚未發佈 Laravel 的語言檔案，您可以透過 `lang:publish` Artisan 命令來執行：

```html
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果請求未成功，使用者將會被重新導向回請求密碼重設連結畫面，並且您可以透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 取得驗證錯誤。或者，如果是 XHR 請求，驗證錯誤將會隨著 422 HTTP 回應一起回傳。

<a name="resetting-the-password"></a>
### 重設密碼

為了完成實作應用程式的密碼重設功能，我們需要指示 Fortify 如何回傳我們的「重設密碼」視圖。

所有 Fortify 的視圖渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別中適用的方法進行自訂。通常，您應該從應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將會處理定義顯示此視圖的路由。您的 `reset-password` 模板應包含一個向 `/reset-password` 發送 POST 請求的表單。

`/reset-password` 端點期望一個字串 `email` 欄位、一個 `password` 欄位、一個 `password_confirmation` 欄位，以及一個名為 `token` 的隱藏欄位，其包含 `request()->route('token')` 的值。「電子郵件」欄位 / 資料庫欄位的名稱應與應用程式的 `fortify` 設定檔中定義的 `email` 設定值相符。

<a name="handling-the-password-reset-response"></a>
#### 處理密碼重設回應

如果密碼重設請求成功，Fortify 將會重新導向回 `/login` 路由，以便使用者可以使用他們的新密碼登入。此外，將會設定一個 `status` Session 變數，以便您可以在登入畫面顯示重設成功的狀態：

```blade
@if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
@endif
```

如果請求是 XHR 請求，將會回傳 200 HTTP 回應。

如果請求未成功，使用者將會被重新導向回重設密碼畫面，並且您可以透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 取得驗證錯誤。或者，如果是 XHR 請求，驗證錯誤將會隨著 422 HTTP 回應一起回傳。

<a name="customizing-password-resets"></a>
### 自訂密碼重設

您可以透過修改在安裝 Laravel Fortify 時所產生的 `App\Actions\ResetUserPassword` 動作來客製化密碼重設的過程。

<a name="email-verification"></a>
## Email 驗證

註冊後，您可能希望使用者在繼續存取您的應用程式之前驗證他們的 Email 地址。首先，請確保您 `fortify` 配置檔的 `features` 陣列中已啟用 `emailVerification` 功能。接下來，您應確保您的 `App\Models\User` 類別實作了 `Illuminate\Contracts\Auth\MustVerifyEmail` 介面。

完成這兩個設定步驟後，新註冊的使用者將會收到一封電子郵件，提示他們驗證其 Email 地址的所有權。然而，我們需要告知 Fortify 如何顯示 Email 驗證畫面，該畫面會通知使用者他們需要點擊 Email 中的驗證連結。

所有 Fortify 視圖的渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

當使用者透過 Laravel 內建的 `verified` 中介層被重新導向到 `/email/verify` 端點時，Fortify 將會負責定義顯示此視圖的路由。

您的 `verify-email` 模板應包含一則訊息，指示使用者點擊發送到他們 Email 地址的 Email 驗證連結。

<a name="resending-email-verification-links"></a>
#### 重新發送 Email 驗證連結

如果您願意，您可以在應用程式的 `verify-email` 模板中添加一個按鈕，觸發向 `/email/verification-notification` 端點發送 POST 請求。當此端點收到請求時，一個新的驗證 Email 連結將會發送給使用者，讓使用者在先前的連結不小心刪除或遺失時，能夠取得新的驗證連結。

如果重新發送驗證連結 Email 的請求成功，Fortify 將會將使用者重新導向回 `/email/verify` 端點，並帶有 `status` session 變數，讓您可以向使用者顯示操作成功的訊息。如果該請求是 XHR 請求，則會回傳 202 HTTP 回應：

```blade
@if (session('status') == 'verification-link-sent')
    <div class="mb-4 font-medium text-sm text-green-600">
        A new email verification link has been emailed to you!
    </div>
@endif
```

<a name="protecting-routes"></a>
### 保護路由

若要指定一個路由或路由群組需要使用者驗證其 Email 地址，您應將 Laravel 內建的 `verified` 中介層附加到該路由。`verified` 中介層別名由 Laravel 自動註冊，並作為 `Illuminate\Auth\Middleware\EnsureEmailIsVerified` 中介層的別名：

```php
Route::get('/dashboard', function () {
    // ...
})->middleware(['verified']);
```

<a name="password-confirmation"></a>
## 密碼確認

在建構您的應用程式時，您偶爾會遇到某些動作在執行前需要使用者確認其密碼。通常，這些路由會受到 Laravel 內建的 `password.confirm` 中介層保護。

若要開始實作密碼確認功能，我們需要告知 Fortify 如何回傳我們應用程式的「密碼確認」視圖。請記住，Fortify 是一個無頭 (headless) 認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能的網頁前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 視圖的渲染邏輯都可以透過 `Laravel\Fortify\Fortify` 類別提供的適當方法進行自訂。通常，您應該在應用程式的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將會負責定義回傳此視圖的 `/user/confirm-password` 端點。您的 `confirm-password` 模板應包含一個表單，該表單會向 `/user/confirm-password` 端點發送 POST 請求。`/user/confirm-password` 端點預期有一個包含使用者目前密碼的 `password` 欄位。

如果密碼與使用者的目前密碼相符，Fortify 將會將使用者重新導向到他們原本嘗試存取的路由。如果該請求是 XHR 請求，則會回傳 201 HTTP 回應。

如果請求未成功，使用者將會被重新導向回密碼確認畫面，並且驗證錯誤將透過共用的 `$errors` Blade 模板變數供您使用。或者，如果是 XHR 請求，驗證錯誤將會隨 422 HTTP 回應一併回傳。