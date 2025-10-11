# 電子郵件驗證

- [簡介](#introduction)
    - [模型準備](#model-preparation)
    - [資料庫準備](#database-preparation)
- [路由](#verification-routing)
    - [電子郵件驗證通知](#the-email-verification-notice)
    - [電子郵件驗證處理器](#the-email-verification-handler)
    - [重新傳送驗證電子郵件](#resending-the-verification-email)
    - [保護路由](#protecting-routes)
- [客製化](#customization)
- [事件](#events)

<a name="introduction"></a>
## 簡介

許多網路應用程式要求使用者在使用前驗證他們的電子郵件地址。Laravel 提供方便的內建服務，用於傳送與驗證電子郵件驗證請求，而不是強迫您為每個建立的應用程式手動重新實作此功能。

> [!NOTE]  
> 想要快速開始？在全新的 Laravel 應用程式中安裝其中一個 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。這些入門套件將負責為您的整個認證系統進行鷹架，包括電子郵件驗證支援。


<a name="model-preparation"></a>
### 模型準備

在開始之前，請確認您的 `App\Models\User` 模型實作了 `Illuminate\Contracts\Auth\MustVerifyEmail` 契約：

    <?php

    namespace App\Models;

    use Illuminate\Contracts\Auth\MustVerifyEmail;
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;

    class User extends Authenticatable implements MustVerifyEmail
    {
        use Notifiable;

        // ...
    }

一旦此介面被加入到您的模型中，新註冊的使用者將會自動收到一封包含電子郵件驗證連結的電子郵件。這之所以能無縫地發生，是因為 Laravel 會自動為 `Illuminate\Auth\Events\Registered` 事件註冊 `Illuminate\Auth\Listeners\SendEmailVerificationNotification` [監聽器](/docs/{{version}}/events)。

如果您是手動在應用程式中實作註冊，而不是使用 [入門套件](/docs/{{version}}/starter-kits)，您應該確保在使用者成功註冊後分派 `Illuminate\Auth\Events\Registered` 事件：

    use Illuminate\Auth\Events\Registered;

    event(new Registered($user));


<a name="database-preparation"></a>
### 資料庫準備

接下來，您的 `users` 資料表必須包含一個 `email_verified_at` 欄位，用來儲存使用者電子郵件地址被驗證的日期與時間。通常，這會包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` 資料庫遷移中。


<a name="verification-routing"></a>
## 路由

為了正確實作電子郵件驗證，需要定義三個路由。首先，需要一個路由來向使用者顯示通知，告知他們應點擊 Laravel 在註冊後寄送給他們的驗證電子郵件中的電子郵件驗證連結。

其次，需要一個路由來處理使用者點擊電子郵件中的電子郵件驗證連結時產生的請求。

第三，如果使用者不小心遺失了第一個驗證連結，則需要一個路由來重新傳送驗證連結。


<a name="the-email-verification-notice"></a>
### 電子郵件驗證通知

如前所述，應該定義一個路由，該路由將回傳一個視圖，指示使用者點擊 Laravel 在註冊後寄送給他們的電子郵件驗證連結。當使用者嘗試存取應用程式的其他部分而未先驗證其電子郵件地址時，將向他們顯示此視圖。請記住，只要您的 `App\Models\User` 模型實作了 `MustVerifyEmail` 介面，該連結就會自動透過電子郵件傳送給使用者：

    Route::get('/email/verify', function () {
        return view('auth.verify-email');
    })->middleware('auth')->name('verification.notice');

回傳電子郵件驗證通知的路由應命名為 `verification.notice`。為此路由分配此確切名稱非常重要，因為 Laravel [內建的](#protecting-routes) `verified` 中介層會自動重新導向到此路由名稱，如果使用者尚未驗證其電子郵件地址。

> [!NOTE]  
> 當手動實作電子郵件驗證時，您需要自行定義驗證通知視圖的內容。如果您想要包含所有必要認證和驗證視圖的鷹架，請查看 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。


<a name="the-email-verification-handler"></a>
### 電子郵件驗證處理器

接下來，我們需要定義一個路由，用於處理使用者點擊寄送給他們的電子郵件驗證連結時產生的請求。此路由應命名為 `verification.verify`，並被分配 `auth` 和 `signed` 中介層：

    use Illuminate\Foundation\Auth\EmailVerificationRequest;

    Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
        $request->fulfill();

        return redirect('/home');
    })->middleware(['auth', 'signed'])->name('verification.verify');

在繼續之前，讓我們仔細看看這個路由。首先，您會注意到我們使用的是 `EmailVerificationRequest` 請求類型，而不是典型的 `Illuminate\Http\Request` 實例。`EmailVerificationRequest` 是 Laravel 內建的 [表單請求](/docs/{{version}}/validation#form-request-validation)。此請求將自動處理驗證請求的 `id` 和 `hash` 參數。

接下來，我們可以直接呼叫請求上的 `fulfill` 方法。此方法將在已認證使用者上呼叫 `markEmailAsVerified` 方法，並分派 `Illuminate\Auth\Events\Verified` 事件。`markEmailAsVerified` 方法透過 `Illuminate\Foundation\Auth\User` 基底類別，在預設的 `App\Models\User` 模型中可用。一旦使用者的電子郵件地址被驗證，您便可以將他們重新導向到您想要的任何位置。


<a name="resending-the-verification-email"></a>
### 重新傳送驗證電子郵件

有時使用者可能會遺失或不小心刪除電子郵件地址驗證電子郵件。為了因應這種情況，您可能希望定義一個路由，允許使用者請求重新傳送驗證電子郵件。然後，您可以在 [驗證通知視圖](#the-email-verification-notice) 中放置一個簡單的表單提交按鈕，向此路由發出請求：

    use Illuminate\Http\Request;

    Route::post('/email/verification-notification', function (Request $request) {
        $request->user()->sendEmailVerificationNotification();

        return back()->with('message', 'Verification link sent!');
    })->middleware(['auth', 'throttle:6,1'])->name('verification.send');


<a name="protecting-routes"></a>
### 保護路由

[路由中介層](/docs/{{version}}/middleware) 可用於僅允許已驗證使用者存取給定路由。Laravel 包含一個 `verified` [中介層別名](/docs/{{version}}/middleware#middleware-aliases)，它是 `Illuminate\Auth\Middleware\EnsureEmailIsVerified` 中介層類別的別名。由於此別名已由 Laravel 自動註冊，因此您所需要做的就是將 `verified` 中介層附加到路由定義。通常，此中介層會與 `auth` 中介層搭配使用：

    Route::get('/profile', function () {
        // Only verified users may access this route...
    })->middleware(['auth', 'verified']);

如果未驗證的使用者嘗試存取已分配此中介層的路由，他們將會自動重新導向到 `verification.notice` [具名路由](/docs/{{version}}/routing#named-routes)。

<a name="customization"></a>
## 客製化

<a name="verification-email-customization"></a>
#### 驗證電子郵件客製化

儘管預設的電子郵件驗證通知應能滿足大多數應用程式的需求，但 Laravel 允許您客製化電子郵件驗證的郵件訊息如何建構。

若要開始客製化，請將一個閉包傳遞給由 `Illuminate\Auth\Notifications\VerifyEmail` 通知所提供的 `toMailUsing` 方法。該閉包將會收到正在接收通知的「可通知的模型實例 (notifiable model instance)」，以及使用者必須造訪才能驗證其電子郵件位址的「簽章過的電子郵件驗證 URL」。該閉包應回傳一個 `Illuminate\Notifications\Messages\MailMessage` 實例。通常，您應該從應用程式的 `AppServiceProvider` 類別的 `boot` 方法中呼叫 `toMailUsing` 方法：

    use Illuminate\Auth\Notifications\VerifyEmail;
    use Illuminate\Notifications\Messages\MailMessage;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        // ...

        VerifyEmail::toMailUsing(function (object $notifiable, string $url) {
            return (new MailMessage)
                ->subject('Verify Email Address')
                ->line('Click the button below to verify your email address.')
                ->action('Verify Email Address', $url);
        });
    }

> [!NOTE]  
> 欲瞭解更多關於郵件通知的資訊，請查閱[郵件通知文件](/docs/{{version}}/notifications#mail-notifications)。

<a name="events"></a>
## 事件

當使用 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)時，Laravel 會在電子郵件驗證過程中分派一個 `Illuminate\Auth\Events\Verified` [事件](/docs/{{version}}/events)。如果您正在手動處理應用程式的電子郵件驗證，您可能會希望在驗證完成後手動分派這些事件。