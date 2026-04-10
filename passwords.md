# 重設密碼

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動器先決條件](#driver-prerequisites)
    - [模型準備](#model-preparation)
    - [設定信任的主機](#configuring-trusted-hosts)
- [路由](#routing)
    - [請求密碼重設連結](#requesting-the-password-reset-link)
    - [重設密碼](#resetting-the-password)
- [刪除過期權杖](#deleting-expired-tokens)
- [客製化](#password-customization)

<a name="introduction"></a>
## 簡介

大多數 Web 應用程式都提供了一種讓使用者重設忘記密碼的方式。Laravel 無須您為每個建立的應用程式手動重新實作此功能，而是提供了便捷的服務來傳送密碼重設連結和安全地重設密碼。

> [!NOTE]
> 想要快速入門嗎？在全新的 Laravel 應用程式中安裝 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件將負責架構您的整個身份驗證系統，包括重設忘記的密碼。


<a name="configuration"></a>
### 設定

您應用程式的密碼重設設定檔儲存在 `config/auth.php`。請務必檢閱此檔案中可用的選項。預設情況下，Laravel 配置為使用 `database` 密碼重設驅動器。

密碼重設 `driver` 設定選項定義了密碼重設資料將儲存在何處。Laravel 包含兩個驅動器：

<div class="content-list" markdown="1">

- `database` - 密碼重設資料儲存在關聯式資料庫中。
- `cache` - 密碼重設資料儲存在您的其中一個基於快取的儲存中。

</div>


<a name="driver-prerequisites"></a>
### 驅動器先決條件


<a name="database"></a>
#### 資料庫

使用預設的 `database` 驅動器時，必須建立一個資料表來儲存您應用程式的密碼重設權杖。通常，這包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` 資料庫遷移中。


<a name="cache"></a>
#### 快取

還有一個快取驅動器可用於處理密碼重設，它不需要專用的資料庫資料表。條目以使用者的電子郵件地址作為鍵，因此請確保您沒有在應用程式的其他地方將電子郵件地址用作快取鍵：

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

為防止呼叫 `artisan cache:clear` 清除您的密碼重設資料，您可以選擇使用 `store` 設定鍵指定一個單獨的快取儲存。該值應對應於您的 `config/cache.php` 設定值中配置的儲存。


<a name="model-preparation"></a>
### 模型準備

在 Laravel 中使用密碼重設功能之前，您的應用程式的 `App\Models\User` 模型必須使用 `Illuminate\Notifications\Notifiable` 特性。通常，這個特性已經包含在新的 Laravel 應用程式建立時預設的 `App\Models\User` 模型中。

接下來，請驗證您的 `App\Models\User` 模型是否實作了 `Illuminate\Contracts\Auth\CanResetPassword` 契約。框架中包含的 `App\Models\User` 模型已經實作了此介面，並使用了 `Illuminate\Auth\Passwords\CanResetPassword` 特性來包含實作該介面所需的方法。


<a name="configuring-trusted-hosts"></a>
### 設定信任的主機

預設情況下，無論 HTTP 請求的 `Host` 標頭內容如何，Laravel 都會回應所有收到的請求。此外，在 Web 請求期間為您的應用程式產生絕對 URL 時，將使用 `Host` 標頭的值。

通常，您應該設定您的 Web 伺服器 (例如 Nginx 或 Apache)，使其只將與給定主機名稱匹配的請求傳送給您的應用程式。但是，如果您無法直接自訂 Web 伺服器，並且需要指示 Laravel 只回應某些主機名稱，您可以透過在應用程式的 `bootstrap/app.php` 檔案中使用 `trustHosts` 中介層方法來實現。當您的應用程式提供密碼重設功能時，這一點尤其重要。

要了解更多關於此中介層方法，請參閱 [TrustHosts 中介層文件](/docs/{{version}}/requests#configuring-trusted-hosts)。

<a name="routing"></a>
## 路由

為了正確實作支援使用者重設密碼的功能，我們需要定義多個路由。首先，我們需要一組路由來處理允許使用者透過他們的電子郵件地址請求密碼重設連結。其次，我們需要一組路由來處理在使用者造訪透過電子郵件傳送的密碼重設連結並填寫密碼重設表單後，實際重設密碼的流程。

<a name="requesting-the-password-reset-link"></a>
### 請求密碼重設連結

<a name="the-password-reset-link-request-form"></a>
#### 密碼重設連結請求表單

首先，我們將定義請求密碼重設連結所需的路由。一開始，我們將定義一個路由，它會回傳一個包含密碼重設連結請求表單的視圖：

```php
Route::get('/forgot-password', function () {
    return view('auth.forgot-password');
})->middleware('guest')->name('password.request');
```

此路由回傳的視圖應包含一個帶有 `email` 欄位的表單，該表單將允許使用者為指定的電子郵件地址請求密碼重設連結。

<a name="password-reset-link-handling-the-form-submission"></a>
#### 處理表單提交

接下來，我們將定義一個路由，用於處理來自「忘記密碼」視圖的表單提交請求。此路由將負責驗證電子郵件地址並將密碼重設請求傳送給對應的使用者：

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

在繼續之前，讓我們更詳細地檢查此路由。首先，請求的 `email` 屬性會被驗證。接著，我們將使用 Laravel 內建的「密碼代理 (password broker)」（透過 `Password` Facade）向使用者傳送密碼重設連結。密碼代理將負責根據給定欄位（在此情況下是電子郵件地址）擷取使用者，並透過 Laravel 內建的[通知系統](/docs/{{version}}/notifications)向使用者傳送密碼重設連結。

`sendResetLink` 方法會回傳一個「狀態」slug。這個狀態可以使用 Laravel 的[在地化](/docs/{{version}}/localization)輔助函式進行翻譯，以便向使用者顯示關於他們請求狀態的友善訊息。密碼重設狀態的翻譯是由應用程式的 `lang/{lang}/passwords.php` 語言檔案決定的。每個可能狀態 slug 的條目都位於 `passwords` 語言檔案中。

> [!NOTE]
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想客製化 Laravel 的語言檔案，可以使用 `lang:publish` Artisan 命令發佈它們。

您可能想知道，當呼叫 `Password` Facade 的 `sendResetLink` 方法時，Laravel 如何知道從應用程式的資料庫中擷取使用者記錄。Laravel 密碼代理利用您的認證系統的「使用者提供者」來擷取資料庫記錄。密碼代理使用的使用者提供者在應用程式的 `config/auth.php` 設定檔案中的 `passwords` 設定陣列中進行配置。要了解有關編寫客製化使用者提供者的更多資訊，請查閱[認證文件](/docs/{{version}}/authentication#adding-custom-user-providers)。

> [!NOTE]
> 當手動實作密碼重設時，您需要自行定義視圖和路由的內容。如果您想要包含所有必要的認證和驗證邏輯的骨架，請查閱 [Laravel 應用程式啟動套件](/docs/{{version}}/starter-kits)。

<a name="resetting-the-password"></a>
### 重設密碼

<a name="the-password-reset-form"></a>
#### 密碼重設表單

接下來，我們將定義必要的路由，以便在使用者點擊透過電子郵件傳送給他們的密碼重設連結並提供新密碼後，實際重設密碼。首先，讓我們定義一個路由，它將顯示使用者點擊密碼重設連結時出現的重設密碼表單。此路由將接收一個 `token` 參數，我們稍後將使用它來驗證密碼重設請求：

```php
Route::get('/reset-password/{token}', function (string $token) {
    return view('auth.reset-password', ['token' => $token]);
})->middleware('guest')->name('password.reset');
```

此路由回傳的視圖應顯示一個包含 `email` 欄位、`password` 欄位、`password_confirmation` 欄位以及一個隱藏的 `token` 欄位的表單，該欄位應包含我們的路由接收到的秘密 `$token` 值。

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

在繼續之前，讓我們更詳細地檢查此路由。首先，請求的 `token`、`email` 和 `password` 屬性會被驗證。接著，我們將使用 Laravel 內建的「密碼代理 (password broker)」（透過 `Password` Facade）來驗證密碼重設請求憑證。

如果提供給密碼代理的權杖、電子郵件地址和密碼是有效的，則傳遞給 `reset` 方法的閉包將被呼叫。在此閉包中，它接收使用者實例和提供給密碼重設表單的明文密碼，我們可以在資料庫中更新使用者的密碼。

`reset` 方法會回傳一個「狀態」slug。這個狀態可以使用 Laravel 的[在地化](/docs/{{version}}/localization)輔助函式進行翻譯，以便向使用者顯示關於他們請求狀態的友善訊息。密碼重設狀態的翻譯是由應用程式的 `lang/{lang}/passwords.php` 語言檔案決定的。每個可能狀態 slug 的條目都位於 `passwords` 語言檔案中。如果您的應用程式不包含 `lang` 目錄，您可以使用 `lang:publish` Artisan 命令建立它。

在繼續之前，您可能想知道，當呼叫 `Password` Facade 的 `reset` 方法時，Laravel 如何知道從應用程式的資料庫中擷取使用者記錄。Laravel 密碼代理利用您的認證系統的「使用者提供者」來擷取資料庫記錄。密碼代理使用的使用者提供者在應用程式的 `config/auth.php` 設定檔案中的 `passwords` 設定陣列中進行配置。要了解有關編寫客製化使用者提供者的更多資訊，請查閱[認證文件](/docs/{{version}}/authentication#adding-custom-user-providers)。

<a name="deleting-expired-tokens"></a>
## 刪除過期權杖

如果您正在使用 `database` 驅動器，過期的密碼重設權杖仍會存在於您的資料庫中。然而，您可以使用 `auth:clear-resets` Artisan 命令輕鬆刪除這些記錄：

```shell
php artisan auth:clear-resets
```

如果您想自動化此流程，請考慮將此命令新增到您應用程式的 [排程器](/docs/{{version}}/scheduling) 中：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('auth:clear-resets')->everyFifteenMinutes();
```


<a name="password-customization"></a>
## 客製化


<a name="reset-link-customization"></a>
#### 重設連結客製化

您可以使用 `ResetPassword` 通知類別提供的 `createUrlUsing` 方法來客製化密碼重設連結的 URL。此方法接受一個閉包，該閉包會接收正在接收通知的使用者實例以及密碼重設連結權杖。通常，您應該從應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫此方法：

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
#### 重設電子郵件客製化

您可以輕鬆修改用於向使用者傳送密碼重設連結的通知類別。首先，請覆寫 `App\Models\User` 模型上的 `sendPasswordResetNotification` 方法。在此方法中，您可以使用任何自訂的 [通知類別](/docs/{{version}}/notifications) 來傳送通知。密碼重設 `$token` 是該方法接收的第一個參數。您可以使用此 `$token` 來建構您選擇的密碼重設 URL，並將通知傳送給使用者：

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