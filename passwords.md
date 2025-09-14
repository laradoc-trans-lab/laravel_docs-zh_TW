# 重設密碼

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
    - [模型準備](#model-preparation)
    - [設定信任的主機](#configuring-trusted-hosts)
- [路由](#routing)
    - [請求密碼重設連結](#requesting-the-password-reset-link)
    - [重設密碼](#resetting-the-password)
- [刪除過期權杖](#deleting-expired-tokens)
- [自訂](#password-customization)

<a name="introduction"></a>
## 簡介

大多數的網路應用程式都提供使用者重設忘記密碼的方式。Laravel 無需您為每個應用程式手動重新實作此功能，而是提供了方便的服務，用於傳送密碼重設連結和安全地重設密碼。

> [!NOTE]
> 想快速上手？在全新的 Laravel 應用程式中安裝 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件將負責建構整個身份驗證系統，包括重設忘記的密碼。

<a name="configuration"></a>
### 設定

您應用程式的密碼重設設定檔儲存在 `config/auth.php`。請務必檢閱此檔案中可用的選項。預設情況下，Laravel 設定為使用 `database` 密碼重設驅動程式。

密碼重設 `driver` 設定選項定義了密碼重設資料將儲存在何處。Laravel 包含兩個驅動程式：

<div class="content-list" markdown="1">

- `database` - 密碼重設資料儲存在關聯式資料庫中。
- `cache` - 密碼重設資料儲存在您的其中一個基於快取的儲存區中。

</div>

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

<a name="database"></a>
#### Database

使用預設的 `database` 驅動程式時，必須建立一個資料表來儲存您應用程式的密碼重設權杖。通常，這會包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` 資料庫遷移中。

<a name="cache"></a>
#### Cache

還有一個快取驅動程式可用於處理密碼重設，它不需要專用的資料庫資料表。條目以使用者的電子郵件地址作為鍵，因此請確保您沒有在應用程式的其他地方使用電子郵件地址作為快取鍵：

```php
'passwords' => [
    'users' => [
        'driver' => 'cache',
        'provider' => 'users',
        'store' => 'passwords', // Optional...
        'expire' => 60,
        'throttle' => 60,
    ],
],
```

為了防止呼叫 `artisan cache:clear` 清除您的密碼重設資料，您可以選擇使用 `store` 設定鍵指定一個單獨的快取儲存區。該值應與您 `config/cache.php` 設定值中設定的儲存區相對應。

<a name="model-preparation"></a>
### 模型準備

在使用 Laravel 的密碼重設功能之前，您應用程式的 `App\Models\User` 模型必須使用 `Illuminate\Notifications\Notifiable` Trait。通常，此 Trait 已包含在隨新的 Laravel 應用程式建立的預設 `App\Models\User` 模型中。

接下來，驗證您的 `App\Models\User` 模型是否實作了 `Illuminate\Contracts\Auth\CanResetPassword` 契約。框架中包含的 `App\Models\User` 模型已經實作了此介面，並使用 `Illuminate\Auth\Passwords\CanResetPassword` Trait 來包含實作介面所需的方法。

<a name="configuring-trusted-hosts"></a>
### 設定信任的主機

預設情況下，Laravel 將回應它收到的所有請求，無論 HTTP 請求的 `Host` 標頭內容為何。此外，在 Web 請求期間產生應用程式的絕對 URL 時，將使用 `Host` 標頭的值。

通常，您應該設定您的 Web 伺服器，例如 Nginx 或 Apache，以僅將符合給定主機名稱的請求傳送到您的應用程式。但是，如果您無法直接自訂 Web 伺服器，並且需要指示 Laravel 僅回應某些主機名稱，您可以使用應用程式 `bootstrap/app.php` 檔案中的 `trustHosts` Middleware 方法來執行此操作。當您的應用程式提供密碼重設功能時，這尤其重要。

要了解有關此 Middleware 方法的更多資訊，請參閱 [TrustHosts Middleware 說明文件](/docs/{{version}}/requests#configuring-trusted-hosts)。

<a name="routing"></a>
## 路由

為了正確實作允許使用者重設密碼的支援，我們需要定義幾個路由。首先，我們需要一對路由來處理允許使用者透過其電子郵件地址請求密碼重設連結。其次，我們需要一對路由來處理在使用者造訪透過電子郵件傳送給他們的密碼重設連結並完成密碼重設表單後實際重設密碼。

<a name="requesting-the-password-reset-link"></a>
### 請求密碼重設連結

<a name="the-password-reset-link-request-form"></a>
#### 密碼重設連結請求表單

首先，我們將定義請求密碼重設連結所需的路由。首先，我們將定義一個路由，該路由會回傳一個包含密碼重設連結請求表單的視圖：

```php
Route::get('/forgot-password', function () {
    return view('auth.forgot-password');
})->middleware('guest')->name('password.request');
```

此路由回傳的視圖應包含一個帶有 `email` 欄位的表單，該欄位將允許使用者為給定的電子郵件地址請求密碼重設連結。

<a name="password-reset-link-handling-the-form-submission"></a>
#### 處理表單提交

接下來，我們將定義一個路由來處理來自「忘記密碼」視圖的表單提交請求。此路由將負責驗證電子郵件地址並將密碼重設請求傳送給相應的使用者：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Password;

Route::post('/forgot-password', function (Request $request) {
    $request->validate(['email' => 'required|email']);

    $status = Password::sendResetLink(
        $request->only('email')
    );

    return $status === Password::ResetLinkSent
        ? back()->with(['status' => __($status)])
        : back()->withErrors(['email' => __($status)]);
})->middleware('guest')->name('password.email');
```

在繼續之前，讓我們更詳細地檢查此路由。首先，驗證請求的 `email` 屬性。接下來，我們將使用 Laravel 內建的「密碼代理」(透過 `Password` Facade) 向使用者傳送密碼重設連結。密碼代理將負責透過給定的欄位 (在此情況下為電子郵件地址) 擷取使用者，並透過 Laravel 內建的[通知系統](/docs/{{version}}/notifications)向使用者傳送密碼重設連結。

`sendResetLink` 方法回傳一個「狀態」Slug。此狀態可以使用 Laravel 的[本地化](/docs/{{version}}/localization)輔助函數進行翻譯，以便向使用者顯示有關其請求狀態的使用者友善訊息。密碼重設狀態的翻譯由您應用程式的 `lang/{lang}/passwords.php` 語言檔案決定。每個可能狀態 Slug 值的一個條目都位於 `passwords` 語言檔案中。

> [!NOTE]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想自訂 Laravel 的語言檔案，您可以透過 `lang:publish` Artisan 命令發布它們。

您可能想知道當呼叫 `Password` Facade 的 `sendResetLink` 方法時，Laravel 如何知道如何從應用程式的資料庫中擷取使用者記錄。Laravel 密碼代理利用您身份驗證系統的「使用者提供者」來擷取資料庫記錄。密碼代理使用的使用者提供者在您 `config/auth.php` 設定檔的 `passwords` 設定陣列中設定。要了解有關編寫自訂使用者提供者的更多資訊，請參閱[身份驗證說明文件](/docs/{{version}}/authentication#adding-custom-user-providers)。

> [!NOTE]
> 手動實作密碼重設時，您需要自行定義視圖和路由的內容。如果您想要包含所有必要身份驗證和驗證邏輯的建構，請查看 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。

<a name="resetting-the-password"></a>
### 重設密碼

<a name="the-password-reset-form"></a>
#### 密碼重設表單

接下來，我們將定義實際重設密碼所需的路由，一旦使用者點擊透過電子郵件傳送給他們的密碼重設連結並提供新密碼。首先，讓我們定義一個路由，該路由將顯示當使用者點擊重設密碼連結時顯示的重設密碼表單。此路由將接收一個 `token` 參數，我們稍後將使用它來驗證密碼重設請求：

```php
Route::get('/reset-password/{token}', function (string $token) {
    return view('auth.reset-password', ['token' => $token]);
})->middleware('guest')->name('password.reset');
```

此路由回傳的視圖應顯示一個包含 `email` 欄位、`password` 欄位、`password_confirmation` 欄位和一個隱藏的 `token` 欄位的表單，該欄位應包含我們的路由接收到的秘密 `$token` 的值。

<a name="password-reset-handling-the-form-submission"></a>
#### 處理表單提交

當然，我們需要定義一個路由來實際處理密碼重設表單提交。此路由將負責驗證傳入請求並更新資料庫中的使用者密碼：

```php
use App\Models\User;
use Illuminate\Auth\Events\PasswordReset;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Password;
use Illuminate\Support\Str;

Route::post('/reset-password', function (Request $request) {
    $request->validate([
        'token' => 'required',
        'email' => 'required|email',
        'password' => 'required|min:8|confirmed',
    ]);

    $status = Password::reset(
        $request->only('email', 'password', 'password_confirmation', 'token'),
        function (User $user, string $password) {
            $user->forceFill([
                'password' => Hash::make($password)
            ])->setRememberToken(Str::random(60));

            $user->save();

            event(new PasswordReset($user));
        }
    );

    return $status === Password::PasswordReset
        ? redirect()->route('login')->with('status', __($status))
        : back()->withErrors(['email' => [__($status)]]);
})->middleware('guest')->name('password.update');
```

在繼續之前，讓我們更詳細地檢查此路由。首先，驗證請求的 `token`、`email` 和 `password` 屬性。接下來，我們將使用 Laravel 內建的「密碼代理」(透過 `Password` Facade) 來驗證密碼重設請求憑證。

如果提供給密碼代理的權杖、電子郵件地址和密碼有效，則將呼叫傳遞給 `reset` 方法的閉包。在此閉包中，它接收使用者實例和提供給密碼重設表單的純文字密碼，我們可以在資料庫中更新使用者的密碼。

`reset` 方法回傳一個「狀態」Slug。此狀態可以使用 Laravel 的[本地化](/docs/{{version}}/localization)輔助函數進行翻譯，以便向使用者顯示有關其請求狀態的使用者友善訊息。密碼重設狀態的翻譯由您應用程式的 `lang/{lang}/passwords.php` 語言檔案決定。每個可能狀態 Slug 值的一個條目都位於 `passwords` 語言檔案中。如果您的應用程式不包含 `lang` 目錄，您可以使用 `lang:publish` Artisan 命令建立它。

在繼續之前，您可能想知道當呼叫 `Password` Facade 的 `reset` 方法時，Laravel 如何知道如何從應用程式的資料庫中擷取使用者記錄。Laravel 密碼代理利用您身份驗證系統的「使用者提供者」來擷取資料庫記錄。密碼代理使用的使用者提供者在您 `config/auth.php` 設定檔的 `passwords` 設定陣列中設定。要了解有關編寫自訂使用者提供者的更多資訊，請參閱[身份驗證說明文件](/docs/{{version}}/authentication#adding-custom-user-providers)。

<a name="deleting-expired-tokens"></a>
## 刪除過期權杖

如果您使用 `database` 驅動程式，過期的密碼重設權杖仍將存在於您的資料庫中。但是，您可以使用 `auth:clear-resets` Artisan 命令輕鬆刪除這些記錄：

```shell
php artisan auth:clear-resets
```

如果您想自動化此過程，請考慮將該命令新增到應用程式的[排程器](/docs/{{version}}/scheduling)中：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('auth:clear-resets')->everyFifteenMinutes();
```

<a name="password-customization"></a>
## 自訂

<a name="reset-link-customization"></a>
#### 重設連結自訂

您可以使用 `ResetPassword` 通知類別提供的 `createUrlUsing` 方法自訂密碼重設連結 URL。此方法接受一個閉包，該閉包接收正在接收通知的使用者實例以及密碼重設連結權杖。通常，您應該從應用程式 `AppServiceProvider` 的 `boot` 方法呼叫此方法：

```php
use App\Models\User;
use Illuminate\Auth\Notifications\ResetPassword;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    ResetPassword::createUrlUsing(function (User $user, string $token) {
        return 'https://example.com/reset-password?token='.$token;
    });
}
```

<a name="reset-email-customization"></a>
#### 重設電子郵件自訂

您可以輕鬆修改用於向使用者傳送密碼重設連結的通知類別。首先，覆寫 `App\Models\User` 模型上的 `sendPasswordResetNotification` 方法。在此方法中，您可以使用您自己建立的任何[通知類別](/docs/{{version}}/notifications)傳送通知。密碼重設 `$token` 是該方法接收的第一個參數。您可以使用此 `$token` 建立您選擇的密碼重設 URL 並將通知傳送給使用者：

```php
use App\Notifications\ResetPasswordNotification;

/**
 * Send a password reset notification to the user.
 *
 * @param  string  $token
 */
public function sendPasswordResetNotification($token): void
{
    $url = 'https://example.com/reset-password?token='.$token;

    $this->notify(new ResetPasswordNotification($url));
}
```
