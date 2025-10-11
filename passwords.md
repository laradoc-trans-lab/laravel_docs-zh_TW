# 重設密碼

- [介紹](#introduction)
    - [模型準備](#model-preparation)
    - [資料庫準備](#database-preparation)
    - [設定信任主機](#configuring-trusted-hosts)
- [路由](#routing)
    - [請求密碼重設連結](#requesting-the-password-reset-link)
    - [重設密碼](#resetting-the-password)
- [刪除過期權杖](#deleting-expired-tokens)
- [客製化](#password-customization)

<a name="introduction"></a>
## 介紹

大多數 Web 應用程式都提供使用者重設遺忘密碼的方式。Laravel 無需您為每個建立的應用程式手動重新實作此功能，而是提供方便的服務來傳送密碼重設連結和安全重設密碼。

> [!NOTE]  
> 想要快速上手？在全新的 Laravel 應用程式中安裝 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits)。Laravel 的入門套件將負責為您架設整個驗證系統，包括重設遺忘的密碼。


<a name="model-preparation"></a>
### 模型準備

在使用 Laravel 的密碼重設功能之前，您的應用程式的 `App\Models\User` 模型必須使用 `Illuminate\Notifications\Notifiable` trait。通常，此 trait 已包含在新 Laravel 應用程式建立的預設 `App\Models\User` 模型中。

接下來，請驗證您的 `App\Models\User` 模型是否實作了 `Illuminate\Contracts\Auth\CanResetPassword` contract。框架中包含的 `App\Models\User` 模型已實作此介面，並使用 `Illuminate\Auth\Passwords\CanResetPassword` trait 來包含實作介面所需的方法。


<a name="database-preparation"></a>
### 資料庫準備

必須建立一個資料表來儲存您應用程式的密碼重設權杖。通常，這已包含在 Laravel 的預設 `0001_01_01_000000_create_users_table.php` 資料庫遷移檔中。


<a name="configuring-trusted-hosts"></a>
### 設定信任主機

預設情況下，無論 HTTP 請求的 `Host` 標頭內容為何，Laravel 都會回應其收到的所有請求。此外，在 Web 請求期間產生應用程式的絕對 URL 時，將會使用 `Host` 標頭的值。

通常，您應該設定您的 Web 伺服器，例如 Nginx 或 Apache，只向您的應用程式傳送與給定主機名稱相符的請求。但是，如果您無法直接自訂您的 Web 伺服器，並且需要指示 Laravel 只回應某些主機名稱，您可以透過在應用程式的 `bootstrap/app.php` 檔案中使用 `trustHosts` 中介層方法來實現。當您的應用程式提供密碼重設功能時，這尤其重要。

要了解有關此中介層方法的更多資訊，請查閱 [`TrustHosts` 中介層文件](/docs/{{version}}/requests#configuring-trusted-hosts)。

<a name="routing"></a>
## 路由

為了正確地支援使用者重設他們的密碼，我們需要定義幾個路由。首先，我們需要一對路由來處理使用者透過他們的電子郵件位址請求密碼重設連結。其次，我們需要一對路由來處理當使用者造訪發送到他們電子郵件中的密碼重設連結並完成密碼重設表單後，實際重設密碼的流程。

<a name="requesting-the-password-reset-link"></a>
### 請求密碼重設連結

<a name="the-password-reset-link-request-form"></a>
#### 密碼重設連結請求表單

首先，我們將定義請求密碼重設連結所需的路由。為了開始，我們將定義一個返回帶有密碼重設連結請求表單的視圖的路由：

    Route::get('/forgot-password', function () {
        return view('auth.forgot-password');
    })->middleware('guest')->name('password.request');

此路由返回的視圖應包含一個帶有 `email` 欄位的表單，該欄位將允許使用者為指定的電子郵件位址請求密碼重設連結。

<a name="password-reset-link-handling-the-form-submission"></a>
#### 處理表單提交

接下來，我們將定義一個路由來處理來自「忘記密碼」視圖的表單提交請求。此路由將負責驗證電子郵件位址並將密碼重設請求傳送給對應的使用者：

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

在繼續之前，讓我們更詳細地檢查這個路由。首先，請求的 `email` 屬性會被驗證。接著，我們將使用 Laravel 內建的「密碼中介」(透過 `Password` Facade) 向使用者傳送密碼重設連結。密碼中介將負責透過給定的欄位 (在此情況下為電子郵件位址) 檢索使用者，並透過 Laravel 內建的[通知系統](/docs/{{version}}/notifications)向使用者傳送密碼重設連結。

`sendResetLink` 方法會返回一個「狀態識別碼」。此狀態可以使用 Laravel 的[本地化](/docs/{{version}}/localization)輔助函數進行翻譯，以便向使用者顯示關於他們請求狀態的使用者友善訊息。密碼重設狀態的翻譯由您應用程式的 `lang/{lang}/passwords.php` 語言檔決定。該 `passwords` 語言檔中包含了狀態識別碼每個可能值的條目。

> [!NOTE]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果您想客製化 Laravel 的語言檔，您可以透過 `lang:publish` Artisan 指令來發佈它們。

您可能想知道當呼叫 `Password` Facade 的 `sendResetLink` 方法時，Laravel 如何知道從應用程式的資料庫中檢索使用者記錄。Laravel 密碼中介利用您認證系統的「使用者提供者」來檢索資料庫記錄。密碼中介使用的使用者提供者在 `config/auth.php` 設定檔的 `passwords` 設定陣列中設定。要了解更多關於編寫自訂使用者提供者的資訊，請查閱[認證文件](/docs/{{version}}/authentication#adding-custom-user-providers)。

> [!NOTE]  
> 當手動實作密碼重設時，您需要自行定義視圖和路由的內容。如果您想要包含所有必要的認證和驗證邏輯的骨架，請查看 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。

<a name="resetting-the-password"></a>
### 重設密碼

<a name="the-password-reset-form"></a>
#### 密碼重設表單

接下來，我們將定義必要的路由，以在使用者點擊電子郵件中寄送的密碼重設連結並提供新密碼後，實際重設密碼。首先，讓我們定義一個路由，用於顯示當使用者點擊密碼重設連結時出現的重設密碼表單。此路由將接收一個 `token` 參數，我們稍後將使用它來驗證密碼重設請求：

    Route::get('/reset-password/{token}', function (string $token) {
        return view('auth.reset-password', ['token' => $token]);
    })->middleware('guest')->name('password.reset');

此路由返回的視圖應顯示一個包含 `email` 欄位、`password` 欄位、`password_confirmation` 欄位以及一個隱藏的 `token` 欄位的表單，該隱藏欄位應包含我們路由接收到的秘密 `$token` 值。

<a name="password-reset-handling-the-form-submission"></a>
#### 處理表單提交

當然，我們需要定義一個路由來實際處理密碼重設表單的提交。此路由將負責驗證傳入的請求並更新資料庫中使用者的密碼：

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

在繼續之前，讓我們更詳細地檢查這個路由。首先，會驗證請求的 `token`、`email` 和 `password` 屬性。接著，我們將使用 Laravel 內建的「password broker」（透過 `Password` Facade）來驗證密碼重設請求憑證。

如果給予 password broker 的 token、電子郵件地址和密碼有效，則會調用傳遞給 `reset` 方法的閉包。在此閉包中，它會接收使用者實例和提供給密碼重設表單的明文密碼，我們可以在資料庫中更新使用者的密碼。

`reset` 方法會回傳一個「狀態」slug。這個狀態可以使用 Laravel 的 [localization](/docs/{{version}}/localization) 輔助函數進行翻譯，以便向使用者顯示關於其請求狀態的友善訊息。密碼重設狀態的翻譯由您應用程式的 `lang/{lang}/passwords.php` 語言檔案決定。每個可能狀態 slug 的條目都位於 `passwords` 語言檔案中。如果您的應用程式不包含 `lang` 目錄，您可以使用 `lang:publish` Artisan 命令來建立它。

在繼續之前，您可能想知道當呼叫 `Password` Facade 的 `reset` 方法時，Laravel 如何知道從應用程式的資料庫中取得使用者記錄。Laravel password broker 利用您認證系統的「使用者提供者」來取得資料庫記錄。password broker 所使用的使用者提供者在您 `config/auth.php` 設定檔的 `passwords` 設定陣列中進行配置。要了解更多關於編寫自訂使用者提供者的資訊，請查閱 [authentication documentation](/docs/{{version}}/authentication#adding-custom-user-providers)。

<a name="deleting-expired-tokens"></a>
## 刪除過期權杖

已過期的密碼重設權杖仍會存在於您的資料庫中。不過，您可以使用 `auth:clear-resets` Artisan 指令輕鬆刪除這些記錄：

```shell
php artisan auth:clear-resets
```

如果您希望自動化此過程，可以考慮將此指令新增到應用程式的 [排程器](/docs/{{version}}/scheduling) 中：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('auth:clear-resets')->everyFifteenMinutes();


<a name="password-customization"></a>
## 客製化


<a name="reset-link-customization"></a>
#### 重設連結客製化

您可以使用 `ResetPassword` 通知類別提供的 `createUrlUsing` 方法來自訂密碼重設連結的 URL。此方法接受一個閉包，該閉包會接收到正在接收通知的使用者實例以及密碼重設連結權杖。通常，您應該在 `App\Providers\AppServiceProvider` 服務提供者的 `boot` 方法中呼叫此方法：

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


<a name="reset-email-customization"></a>
#### 重設電子郵件客製化

您可以輕鬆修改用於向使用者寄送密碼重設連結的通知類別。首先，請覆寫 `App\Models\User` 模型上的 `sendPasswordResetNotification` 方法。在此方法中，您可以使用任何自訂的 [通知類別](/docs/{{version}}/notifications) 來寄送通知。密碼重設的 `$token` 是此方法接收的第一個引數。您可以使用這個 `$token` 來建構您選擇的密碼重設 URL，並向使用者寄送您的通知：

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