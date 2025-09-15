# Laravel Fortify

- [簡介](#introduction)
    - [什麼是 Fortify？](#what-is-fortify)
    - [何時該使用 Fortify？](#when-should-i-use-fortify)
- [安裝](#installation)
    - [Fortify 功能](#fortify-features)
    - [停用視圖](#disabling-views)
- [認證](#authentication)
    - [自訂使用者認證](#customizing-user-authentication)
    - [自訂認證流程](#customizing-the-authentication-pipeline)
    - [自訂重新導向](#customizing-authentication-redirects)
- [雙重認證](#two-factor-authentication)
    - [啟用雙重認證](#enabling-two-factor-authentication)
    - [使用雙重認證進行認證](#authenticating-with-two-factor-authentication)
    - [停用雙重認證](#disabling-two-factor-authentication)
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

[Laravel Fortify](https://github.com/laravel/fortify) 是 Laravel 框架中一個與前端無關的認證後端實作。Fortify 註冊了實作 Laravel 所有認證功能所需的路由與控制器，包括登入、註冊、密碼重設、電子郵件驗證等。安裝 Fortify 後，您可以執行 `route:list` Artisan 命令來查看 Fortify 已註冊的路由。

由於 Fortify 不提供自己的使用者介面，它旨在與您自己的使用者介面搭配使用，由該介面向其註冊的路由發出請求。我們將在本文件其餘部分詳細討論如何向這些路由發出請求。

> [!NOTE]  
> 請記住，Fortify 是一個旨在讓您快速開始實作 Laravel 認證功能的套件。**您不一定要使用它。** 您隨時可以透過遵循 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 和 [電子郵件驗證](/docs/{{version}}/verification) 文件中提供的說明，手動與 Laravel 的認證服務互動。

<a name="what-is-fortify"></a>
### 什麼是 Fortify？

如前所述，Laravel Fortify 是 Laravel 框架中一個與前端無關的認證後端實作。Fortify 註冊了實作 Laravel 所有認證功能所需的路由與控制器，包括登入、註冊、密碼重設、電子郵件驗證等。

**您不一定要使用 Fortify 才能使用 Laravel 的認證功能。** 您隨時可以透過遵循 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 和 [電子郵件驗證](/docs/{{version}}/verification) 文件中提供的說明，手動與 Laravel 的認證服務互動。

如果您是 Laravel 的新手，在嘗試使用 Laravel Fortify 之前，您可能希望先探索 [Laravel Breeze](/docs/{{version}}/starter-kits) 應用程式入門套件。Laravel Breeze 為您的應用程式提供了認證骨架，其中包含使用 [Tailwind CSS](https://tailwindcss.com) 建構的使用者介面。與 Fortify 不同，Breeze 將其路由和控制器直接發佈到您的應用程式中。這讓您可以在讓 Laravel Fortify 為您實作這些功能之前，先研究並熟悉 Laravel 的認證功能。

Laravel Fortify 本質上是將 Laravel Breeze 的路由和控制器作為一個不包含使用者介面的套件提供。這讓您仍然可以快速地為應用程式的認證層建構後端實作，而無需受限於任何特定的前端意見。

<a name="when-should-i-use-fortify"></a>
### 何時該使用 Fortify？

您可能想知道何時適合使用 Laravel Fortify。首先，如果您正在使用 Laravel 的其中一個 [應用程式入門套件](/docs/{{version}}/starter-kits)，則無需安裝 Laravel Fortify，因為所有 Laravel 的應用程式入門套件都已提供完整的認證實作。

如果您沒有使用應用程式入門套件，並且您的應用程式需要認證功能，您有兩個選擇：手動實作應用程式的認證功能，或使用 Laravel Fortify 來提供這些功能的後端實作。

如果您選擇安裝 Fortify，您的使用者介面將向本文件中詳述的 Fortify 認證路由發出請求，以便認證和註冊使用者。

如果您選擇手動與 Laravel 的認證服務互動而不是使用 Fortify，您可以透過遵循 [認證](/docs/{{version}}/authentication)、[密碼重設](/docs/{{version}}/passwords) 和 [電子郵件驗證](/docs/{{version}}/verification) 文件中提供的說明來進行。

<a name="laravel-fortify-and-laravel-sanctum"></a>
#### Laravel Fortify 與 Laravel Sanctum

有些開發者對 [Laravel Sanctum](/docs/{{version}}/sanctum) 和 Laravel Fortify 之間的差異感到困惑。由於這兩個套件解決了兩個不同但相關的問題，Laravel Fortify 和 Laravel Sanctum 並非互斥或競爭的套件。

Laravel Sanctum 僅關注管理 API 權杖以及使用 Session Cookie 或權杖認證現有使用者。Sanctum 不提供任何處理使用者註冊、密碼重設等的路由。

如果您正在嘗試手動建構提供 API 或作為單頁應用程式後端的應用程式的認證層，那麼您很可能會同時使用 Laravel Fortify (用於使用者註冊、密碼重設等) 和 Laravel Sanctum (API 權杖管理、Session 認證)。

<a name="installation"></a>
## 安裝

首先，使用 Composer 套件管理器安裝 Fortify：

```shell
composer require laravel/fortify
```

接下來，使用 `fortify:install` Artisan 命令發佈 Fortify 的資源：

```shell
php artisan fortify:install
```

此命令會將 Fortify 的 Actions 發佈到您的 `app/Actions` 目錄中，如果該目錄不存在，則會建立它。此外，`FortifyServiceProvider`、設定檔和所有必要的資料庫遷移檔都將被發佈。

接下來，您應該遷移資料庫：

```shell
php artisan migrate
```

<a name="fortify-features"></a>
### Fortify 功能

`fortify` 設定檔包含一個 `features` 設定陣列。此陣列定義了 Fortify 預設會公開哪些後端路由/功能。如果您沒有將 Fortify 與 [Laravel Jetstream](https://jetstream.laravel.com) 結合使用，我們建議您只啟用以下功能，這些是大多數 Laravel 應用程式提供的基本認證功能：

```php
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
],
```

<a name="disabling-views"></a>
### 停用視圖

預設情況下，Fortify 定義了旨在返回視圖的路由，例如登入畫面或註冊畫面。但是，如果您正在建構一個由 JavaScript 驅動的單頁應用程式，您可能不需要這些路由。因此，您可以透過將應用程式 `config/fortify.php` 設定檔中的 `views` 設定值設為 `false` 來完全停用這些路由：

```php
'views' => false,
```

<a name="disabling-views-and-password-reset"></a>
#### 停用視圖與密碼重設

如果您選擇停用 Fortify 的視圖，並且您將為您的應用程式實作密碼重設功能，您仍然應該定義一個名為 `password.reset` 的路由，該路由負責顯示您應用程式的「重設密碼」視圖。這是必要的，因為 Laravel 的 `Illuminate\Auth\Notifications\ResetPassword` 通知將透過 `password.reset` 命名路由產生密碼重設 URL。

<a name="authentication"></a>
## 認證

首先，我們需要指示 Fortify 如何返回我們的「登入」視圖。請記住，Fortify 是一個無頭 (headless) 認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有認證視圖的渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法。Fortify 將負責定義返回此視圖的 `/login` 路由：

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

您的登入模板應包含一個向 `/login` 發出 POST 請求的表單。`/login` 端點需要一個字串 `email` / `username` 和一個 `password`。電子郵件/使用者名稱欄位的名稱應與 `config/fortify.php` 設定檔中的 `username` 值相符。此外，可以提供一個布林 `remember` 欄位，以指示使用者希望使用 Laravel 提供的「記住我」功能。

如果登入成功，Fortify 會將您重新導向到應用程式 `fortify` 設定檔中透過 `home` 設定選項配置的 URI。如果登入請求是 XHR 請求，則會返回 200 HTTP 回應。

如果請求不成功，使用者將被重新導向回登入畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應返回。

<a name="customizing-user-authentication"></a>
### 自訂使用者認證

Fortify 會根據提供的憑證和為您的應用程式配置的認證 Guard 自動擷取並認證使用者。但是，您有時可能希望完全自訂登入憑證的認證方式和使用者的擷取方式。幸運的是，Fortify 允許您使用 `Fortify::authenticateUsing` 方法輕鬆實現此目的。

此方法接受一個閉包，該閉包會接收傳入的 HTTP 請求。該閉包負責驗證附加到請求的登入憑證並返回相關的使用者實例。如果憑證無效或找不到使用者，閉包應返回 `null` 或 `false`。通常，此方法應從您的 `FortifyServiceProvider` 的 `boot` 方法中呼叫：

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

您可以在應用程式的 `fortify` 設定檔中自訂 Fortify 使用的認證 Guard。但是，您應該確保配置的 Guard 是 `Illuminate\Contracts\Auth\StatefulGuard` 的實作。如果您嘗試使用 Laravel Fortify 來認證 SPA，您應該將 Laravel 的預設 `web` Guard 與 [Laravel Sanctum](https://laravel.com/docs/sanctum) 結合使用。

<a name="customizing-the-authentication-pipeline"></a>
### 自訂認證流程

Laravel Fortify 透過一系列可呼叫類別的 Pipeline 來認證登入請求。如果您願意，您可以定義一個自訂的類別 Pipeline，登入請求將透過該 Pipeline 進行處理。每個類別都應該有一個 `__invoke` 方法，該方法接收傳入的 `Illuminate\Http\Request` 實例，並且像 [Middleware](/docs/{{version}}/middleware) 一樣，還有一個 `$next` 變數，該變數被呼叫以將請求傳遞給 Pipeline 中的下一個類別。

要定義您的自訂 Pipeline，您可以使用 `Fortify::authenticateThrough` 方法。此方法接受一個閉包，該閉包應返回要將登入請求透過其處理的類別陣列。通常，此方法應從您的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫。

以下範例包含預設的 Pipeline 定義，您可以在進行自己的修改時將其作為起點：

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

預設情況下，Fortify 將使用 `EnsureLoginIsNotThrottled` Middleware 來節流認證嘗試。此 Middleware 會節流使用者名稱和 IP 位址組合獨特的嘗試。

某些應用程式可能需要不同的方法來節流認證嘗試，例如僅按 IP 位址節流。因此，Fortify 允許您透過 `fortify.limiters.login` 設定選項指定您自己的 [速率限制器](/docs/{{version}}/routing#rate-limiting)。當然，此設定選項位於您應用程式的 `config/fortify.php` 設定檔中。

> [!NOTE]  
> 結合使用節流、[雙重認證](/docs/{{version}}/fortify#two-factor-authentication) 和外部 Web 應用程式防火牆 (WAF) 將為您的合法應用程式使用者提供最穩固的防禦。

<a name="customizing-authentication-redirects"></a>
### 自訂重新導向

如果登入成功，Fortify 會將您重新導向到應用程式 `fortify` 設定檔中透過 `home` 設定選項配置的 URI。如果登入請求是 XHR 請求，則會返回 200 HTTP 回應。使用者登出應用程式後，將被重新導向到 `/` URI。

如果您需要對此行為進行進階自訂，您可以將 `LoginResponse` 和 `LogoutResponse` 契約的實作綁定到 Laravel [服務容器](/docs/{{version}}/container) 中。通常，這應該在您應用程式 `App\Providers\FortifyServiceProvider` 類別的 `register` 方法中完成：

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

當 Fortify 的雙重認證功能啟用時，使用者在認證過程中需要輸入一個六位數的數字權杖。此權杖是使用基於時間的一次性密碼 (TOTP) 生成的，可以從任何相容 TOTP 的行動認證應用程式（例如 Google Authenticator）中擷取。

在開始之前，您應該首先確保應用程式的 `App\Models\User` 模型使用 `Laravel\Fortify\TwoFactorAuthenticatable` Trait：

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

接下來，您應該在應用程式中建構一個畫面，讓使用者可以管理他們的雙重認證設定。此畫面應允許使用者啟用和停用雙重認證，以及重新產生他們的雙重認證復原碼。

> 預設情況下，`fortify` 設定檔的 `features` 陣列指示 Fortify 的雙重認證設定在修改前需要密碼確認。因此，您的應用程式應在繼續之前實作 Fortify 的 [密碼確認](#password-confirmation) 功能。

<a name="enabling-two-factor-authentication"></a>
### 啟用雙重認證

要開始啟用雙重認證，您的應用程式應向 Fortify 定義的 `/user/two-factor-authentication` 端點發出 POST 請求。如果請求成功，使用者將被重新導向回先前的 URL，並且 `status` Session 變數將被設定為 `two-factor-authentication-enabled`。您可以在模板中偵測此 `status` Session 變數以顯示適當的成功訊息。如果請求是 XHR 請求，則會返回 `200` HTTP 回應。

選擇啟用雙重認證後，使用者仍必須透過提供有效的雙重認證碼來「確認」其雙重認證配置。因此，您的「成功」訊息應指示使用者仍需要進行雙重認證確認：

```html
 @if (session('status') == 'two-factor-authentication-enabled')
    <div class="mb-4 font-medium text-sm">
        請完成下方雙重認證的設定。
    </div>
 @endif
```

接下來，您應該顯示雙重認證 QR Code，供使用者掃描到他們的認證應用程式中。如果您使用 Blade 來渲染應用程式的前端，您可以使用使用者實例上可用的 `twoFactorQrCodeSvg` 方法來擷取 QR Code SVG：

```php
$request->user()->twoFactorQrCodeSvg();
```

如果您正在建構由 JavaScript 驅動的前端，您可以向 `/user/two-factor-qr-code` 端點發出 XHR GET 請求，以擷取使用者的雙重認證 QR Code。此端點將返回一個包含 `svg` 鍵的 JSON 物件。

<a name="confirming-two-factor-authentication"></a>
#### 確認雙重認證

除了顯示使用者的雙重認證 QR Code 之外，您還應該提供一個文字輸入框，讓使用者可以提供有效的認證碼來「確認」其雙重認證配置。此代碼應透過向 Fortify 定義的 `/user/confirmed-two-factor-authentication` 端點發出 POST 請求來提供給 Laravel 應用程式。

如果請求成功，使用者將被重新導向回先前的 URL，並且 `status` Session 變數將被設定為 `two-factor-authentication-confirmed`：

```html
 @if (session('status') == 'two-factor-authentication-confirmed')
    <div class="mb-4 font-medium text-sm">
        雙重認證已成功確認並啟用。
    </div>
 @endif
```

如果向雙重認證確認端點發出的請求是透過 XHR 請求進行的，則會返回 `200` HTTP 回應。

<a name="displaying-the-recovery-codes"></a>
#### 顯示復原碼

您還應該顯示使用者的雙重認證復原碼。這些復原碼允許使用者在失去行動裝置存取權時進行認證。如果您使用 Blade 來渲染應用程式的前端，您可以透過已認證的使用者實例存取復原碼：

```php
(array) $request->user()->recoveryCodes()
```

如果您正在建構由 JavaScript 驅動的前端，您可以向 `/user/two-factor-recovery-codes` 端點發出 XHR GET 請求。此端點將返回一個包含使用者復原碼的 JSON 陣列。

要重新產生使用者的復原碼，您的應用程式應向 `/user/two-factor-recovery-codes` 端點發出 POST 請求。

<a name="authenticating-with-two-factor-authentication"></a>
### 使用雙重認證進行認證

在認證過程中，Fortify 會自動將使用者重新導向到您應用程式的雙重認證挑戰畫面。但是，如果您的應用程式正在發出 XHR 登入請求，成功認證嘗試後返回的 JSON 回應將包含一個具有 `two_factor` 布林屬性的 JSON 物件。您應該檢查此值以判斷是否應重新導向到您應用程式的雙重認證挑戰畫面。

要開始實作雙重認證功能，我們需要指示 Fortify 如何返回我們的雙重認證挑戰視圖。所有 Fortify 的認證視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義返回此視圖的 `/two-factor-challenge` 路由。您的 `two-factor-challenge` 模板應包含一個向 `/two-factor-challenge` 端點發出 POST 請求的表單。`/two-factor-challenge` Action 需要一個包含有效 TOTP 權杖的 `code` 欄位，或一個包含使用者其中一個復原碼的 `recovery_code` 欄位。

如果登入成功，Fortify 會將使用者重新導向到應用程式 `fortify` 設定檔中透過 `home` 設定選項配置的 URI。如果登入請求是 XHR 請求，則會返回 204 HTTP 回應。

如果請求不成功，使用者將被重新導向回雙重認證挑戰畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應返回。

<a name="disabling-two-factor-authentication"></a>
### 停用雙重認證

要停用雙重認證，您的應用程式應向 `/user/two-factor-authentication` 端點發出 DELETE 請求。請記住，Fortify 的雙重認證端點在呼叫之前需要 [密碼確認](#password-confirmation)。

<a name="registration"></a>
## 註冊

要開始實作應用程式的註冊功能，我們需要指示 Fortify 如何返回我們的「註冊」視圖。請記住，Fortify 是一個無頭 (headless) 認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從您的 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義返回此視圖的 `/register` 路由。您的 `register` 模板應包含一個向 Fortify 定義的 `/register` 端點發出 POST 請求的表單。

`/register` 端點需要一個字串 `name`、字串電子郵件地址/使用者名稱、`password` 和 `password_confirmation` 欄位。電子郵件/使用者名稱欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `username` 設定值相符。

如果註冊成功，Fortify 會將使用者重新導向到應用程式 `fortify` 設定檔中透過 `home` 設定選項配置的 URI。如果請求是 XHR 請求，則會返回 201 HTTP 回應。

如果請求不成功，使用者將被重新導向回註冊畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應返回。

<a name="customizing-registration"></a>
### 自訂註冊

使用者驗證和建立過程可以透過修改您安裝 Laravel Fortify 時產生的 `App\Actions\Fortify\CreateNewUser` Action 來進行自訂。

<a name="password-reset"></a>
## 密碼重設

<a name="requesting-a-password-reset-link"></a>
### 請求密碼重設連結

要開始實作應用程式的密碼重設功能，我們需要指示 Fortify 如何返回我們的「忘記密碼」視圖。請記住，Fortify 是一個無頭 (headless) 認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義返回此視圖的 `/forgot-password` 端點。您的 `forgot-password` 模板應包含一個向 `/forgot-password` 端點發出 POST 請求的表單。

`/forgot-password` 端點需要一個字串 `email` 欄位。此欄位/資料庫欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `email` 設定值相符。

<a name="handling-the-password-reset-link-request-response"></a>
#### 處理密碼重設連結請求回應

如果密碼重設連結請求成功，Fortify 會將使用者重新導向回 `/forgot-password` 端點，並向使用者發送一封包含安全連結的電子郵件，他們可以使用該連結重設密碼。如果請求是 XHR 請求，則會返回 200 HTTP 回應。

成功請求後被重新導向回 `/forgot-password` 端點後，`status` Session 變數可用於顯示密碼重設連結請求嘗試的狀態。

`$status` Session 變數的值將與應用程式 `passwords` [語言檔](/docs/{{version}}/localization) 中定義的其中一個翻譯字串相符。如果您想自訂此值但尚未發佈 Laravel 的語言檔，您可以透過 `lang:publish` Artisan 命令來進行：

```html
 @if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
 @endif
```

如果請求不成功，使用者將被重新導向回請求密碼重設連結畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應返回。

<a name="resetting-the-password"></a>
### 重設密碼

要完成實作應用程式的密碼重設功能，我們需要指示 Fortify 如何返回我們的「重設密碼」視圖。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義顯示此視圖的路由。您的 `reset-password` 模板應包含一個向 `/reset-password` 發出 POST 請求的表單。

`/reset-password` 端點需要一個字串 `email` 欄位、一個 `password` 欄位、一個 `password_confirmation` 欄位，以及一個名為 `token` 的隱藏欄位，其中包含 `request()->route('token')` 的值。電子郵件欄位/資料庫欄位的名稱應與應用程式 `fortify` 設定檔中定義的 `email` 設定值相符。

<a name="handling-the-password-reset-response"></a>
#### 處理密碼重設回應

如果密碼重設請求成功，Fortify 會將使用者重新導向回 `/login` 路由，以便使用者可以使用他們的新密碼登入。此外，將設定一個 `status` Session 變數，以便您可以在登入畫面上顯示重設的成功狀態：

```blade
 @if (session('status'))
    <div class="mb-4 font-medium text-sm text-green-600">
        {{ session('status') }}
    </div>
 @endif
```

如果請求是 XHR 請求，則會返回 200 HTTP 回應。

如果請求不成功，使用者將被重新導向回重設密碼畫面，並且驗證錯誤將透過共用的 `$errors` [Blade 模板變數](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 提供給您。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應返回。

<a name="customizing-password-resets"></a>
### 自訂密碼重設

密碼重設過程可以透過修改您安裝 Laravel Fortify 時產生的 `App\Actions\ResetUserPassword` Action 來進行自訂。

<a name="email-verification"></a>
## 電子郵件驗證

註冊後，您可能希望使用者在繼續存取您的應用程式之前驗證他們的電子郵件地址。首先，請確保在您的 `fortify` 設定檔的 `features` 陣列中啟用 `emailVerification` 功能。接下來，您應該確保您的 `App\Models\User` 類別實作了 `Illuminate\Contracts\Auth\MustVerifyEmail` 介面。

完成這兩個設定步驟後，新註冊的使用者將自動收到一封電子郵件，提示他們驗證其電子郵件地址所有權。但是，我們需要通知 Fortify 如何顯示電子郵件驗證畫面，該畫面會告知使用者需要點擊電子郵件中的驗證連結。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

當使用者被 Laravel 內建的 `verified` Middleware 重新導向到 `/email/verify` 端點時，Fortify 將負責定義顯示此視圖的路由。

您的 `verify-email` 模板應包含一條資訊訊息，指示使用者點擊已發送到其電子郵件地址的電子郵件驗證連結。

<a name="resending-email-verification-links"></a>
#### 重新發送電子郵件驗證連結

如果您願意，您可以在應用程式的 `verify-email` 模板中添加一個按鈕，該按鈕會觸發向 `/email/verification-notification` 端點發出 POST 請求。當此端點收到請求時，新的驗證電子郵件連結將會發送給使用者，允許使用者在先前的連結不小心刪除或遺失時取得新的驗證連結。

如果重新發送驗證連結電子郵件的請求成功，Fortify 會將使用者重新導向回 `/email/verify` 端點，並帶有 `status` Session 變數，讓您可以向使用者顯示操作成功的資訊訊息。如果請求是 XHR 請求，則會返回 202 HTTP 回應：

```blade
 @if (session('status') == 'verification-link-sent')
    <div class="mb-4 font-medium text-sm text-green-600">
        新的電子郵件驗證連結已發送給您！
    </div>
 @endif
```

<a name="protecting-routes"></a>
### 保護路由

要指定路由或路由群組需要使用者已驗證其電子郵件地址，您應該將 Laravel 內建的 `verified` Middleware 附加到該路由。`verified` Middleware 別名由 Laravel 自動註冊，並作為 `Illuminate\Auth\Middleware\EnsureEmailIsVerified` Middleware 類別的別名：

```php
Route::get('/dashboard', function () {
    // ...
})->middleware(['verified']);
```

<a name="password-confirmation"></a>
## 密碼確認

在建構應用程式時，您偶爾可能會遇到需要使用者在執行操作前確認其密碼的動作。通常，這些路由會受到 Laravel 內建的 `password.confirm` Middleware 保護。

要開始實作密碼確認功能，我們需要指示 Fortify 如何返回應用程式的「密碼確認」視圖。請記住，Fortify 是一個無頭 (headless) 認證函式庫。如果您想要一個已為您完成的 Laravel 認證功能前端實作，您應該使用 [應用程式入門套件](/docs/{{version}}/starter-kits)。

所有 Fortify 的視圖渲染邏輯都可以使用 `Laravel\Fortify\Fortify` 類別中提供的適當方法進行自訂。通常，您應該從應用程式 `App\Providers\FortifyServiceProvider` 類別的 `boot` 方法中呼叫此方法：

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

Fortify 將負責定義返回此視圖的 `/user/confirm-password` 端點。您的 `confirm-password` 模板應包含一個向 `/user/confirm-password` 端點發出 POST 請求的表單。`/user/confirm-password` 端點需要一個包含使用者當前密碼的 `password` 欄位。

如果密碼與使用者當前密碼相符，Fortify 會將使用者重新導向到他們嘗試存取的路由。如果請求是 XHR 請求，則會返回 201 HTTP 回應。

如果請求不成功，使用者將被重新導向回確認密碼畫面，並且驗證錯誤將透過共用的 `$errors` Blade 模板變數提供給您。或者，如果是 XHR 請求，驗證錯誤將隨 422 HTTP 回應返回。
