# 認證

- [簡介](#introduction)
    - [入門套件](#starter-kits)
    - [資料庫考量](#introduction-database-considerations)
    - [生態系統概述](#ecosystem-overview)
- [認證快速入門](#authentication-quickstart)
    - [安裝入門套件](#install-a-starter-kit)
    - [擷取已認證使用者](#retrieving-the-authenticated-user)
    - [保護路由](#protecting-routes)
    - [登入節流](#login-throttling)
- [手動認證使用者](#authenticating-users)
    - [記住使用者](#remembering-users)
    - [其他認證方法](#other-authentication-methods)
- [HTTP 基本認證](#http-basic-authentication)
    - [無狀態 HTTP 基本認證](#stateless-http-basic-authentication)
- [登出](#logging-out)
    - [讓其他裝置的 Session 失效](#invalidating-sessions-on-other-devices)
- [密碼確認](#password-confirmation)
    - [設定](#password-confirmation-configuration)
    - [路由](#password-confirmation-routing)
    - [保護路由](#password-confirmation-protecting-routes)
- [新增自訂 Guards](#adding-custom-guards)
    - [Closure 請求 Guards](#closure-request-guards)
- [新增自訂使用者提供者](#adding-custom-user-providers)
    - [使用者提供者契約](#the-user-provider-contract)
    - [Authenticatable 契約](#the-authenticatable-contract)
- [自動密碼再雜湊](#automatic-password-rehashing)
- [社群認證](/docs/{{version}}/socialite)
- [事件](#events)

<a name="introduction"></a>
## 簡介

許多 Web 應用程式都為其使用者提供了認證並「登入」應用程式的方式。在 Web 應用程式中實作此功能可能是一項複雜且潛在風險的任務。因此，Laravel 致力於為您提供所需的工具，以快速、安全、輕鬆地實作認證。

Laravel 的認證設施核心由「guards」和「提供者」組成。Guards 定義了每個請求中使用者如何被認證。舉例來說，Laravel 內建了一個 `session` guard，它使用 Session 儲存和 Cookie 來維護狀態。

提供者定義了如何從您的持久儲存中擷取使用者。Laravel 內建支援使用 [Eloquent](/docs/{{version}}/eloquent) 和資料庫查詢產生器來擷取使用者。但是，您可以根據應用程式的需要自由定義額外的提供者。

您應用程式的認證設定檔位於 `config/auth.php`。此檔案包含了多個文件完善的選項，用於調整 Laravel 認證服務的行為。

> [!NOTE]  
> Guards 和提供者不應與「角色」和「權限」混淆。要了解更多關於透過權限授權使用者操作的資訊，請參考[授權](/docs/{{version}}/authorization)文件。

<a name="starter-kits"></a>
### 入門套件

想快速入門嗎？在全新的 Laravel 應用程式中安裝 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。遷移資料庫後，在瀏覽器中導覽至 `/register` 或任何分配給您應用程式的其他 URL。這些入門套件將負責為您搭建整個認證系統！

**即使您選擇不在最終的 Laravel 應用程式中使用入門套件，安裝 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze) 入門套件也是一個學習如何在實際的 Laravel 專案中實作所有 Laravel 認證功能的絕佳機會。** 由於 Laravel Breeze 為您建立了認證控制器、路由和視圖，您可以檢視這些檔案中的程式碼，以了解如何實作 Laravel 的認證功能。

<a name="introduction-database-considerations"></a>
### 資料庫考量

預設情況下，Laravel 在您的 `app/Models` 目錄中包含一個 `App\Models\User` [Eloquent 模型](/docs/{{version}}/eloquent)。此模型可以與預設的 Eloquent 認證驅動一起使用。

如果您的應用程式未使用 Eloquent，您可以使用 `database` 認證提供者，它使用 Laravel 查詢產生器。如果您的應用程式正在使用 MongoDB，請查閱 MongoDB 官方的 [Laravel 使用者認證文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/user-authentication/)。

為 `App\Models\User` 模型建構資料庫綱要時，請確保密碼欄位的長度至少為 60 個字元。當然，新 Laravel 應用程式中包含的 `users` 資料表遷移檔案已建立一個超過此長度的欄位。

此外，您應該驗證您的 `users` (或等效) 資料表包含一個長度為 100 個字元的可為空、字串型別的 `remember_token` 欄位。此欄位將用於儲存勾選「記住我」選項登入應用程式的使用者的 token。同樣，新 Laravel 應用程式中包含的預設 `users` 資料表遷移檔案已經包含了此欄位。

<a name="ecosystem-overview"></a>
### 生態系統概述

Laravel 提供多個與認證相關的套件。在繼續之前，我們將回顧 Laravel 中的通用認證生態系統，並討論每個套件的預期用途。

首先，考量認證的運作方式。當使用網頁瀏覽器時，使用者將透過登入表單提供其使用者名稱與密碼。若這些憑證正確，應用程式將把已認證使用者的資訊儲存在使用者的 [session](/docs/{{version}}/session) 中。發給瀏覽器的 cookie 包含 session ID，以便後續對應用程式的請求可以將使用者與正確的 session 關聯起來。接收到 session cookie 後，應用程式將根據 session ID 擷取 session 資料，並注意認證資訊已儲存在 session 中，並將使用者視為「已認證」。

當遠端服務需要認證以存取 API 時，通常不使用 cookie 進行認證，因為沒有網頁瀏覽器。相反地，遠端服務會在每個請求中向 API 發送一個 API token。應用程式可能會根據一個有效 API token 表來驗證傳入的 token，並將該請求「認證」為由與該 API token 關聯的使用者執行。

<a name="laravels-built-in-browser-authentication-services"></a>
#### Laravel 內建的瀏覽器認證服務

Laravel 包含內建的認證和 session 服務，通常透過 `Auth` 和 `Session` Facades 存取。這些功能為從網頁瀏覽器發起的請求提供基於 cookie 的認證。它們提供方法，讓您可以驗證使用者的憑證並認證使用者。此外，這些服務將自動在使用者 session 中儲存適當的認證資料，並發放使用者的 session cookie。如何使用這些服務的討論包含在本文件中。

**應用程式入門套件**

如本文件所述，您可以手動與這些認證服務互動，以建立應用程式自己的認證層。然而，為了幫助您更快入門，我們已發布 [免費套件](/docs/{{version}}/starter-kits)，這些套件提供強大且現代化的整個認證層腳手架。這些套件是 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze)、[Laravel Jetstream](/docs/{{version}}/starter-kits#laravel-jetstream) 和 [Laravel Fortify](/docs/{{version}}/fortify)。

_Laravel Breeze_ 是 Laravel 所有認證功能的一個簡單、最小化的實作，包括登入、註冊、密碼重設、電子郵件驗證和密碼確認。Laravel Breeze 的視圖層由使用 [Tailwind CSS](https://tailwindcss.com) 設計的簡單 [Blade 模板](/docs/{{version}}/blade) 組成。若要開始使用，請查閱 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits) 的文件。

_Laravel Fortify_ 是 Laravel 的無頭 (headless) 認證後端，實作了本文件中發現的許多功能，包括基於 cookie 的認證以及雙因子認證和電子郵件驗證等其他功能。Fortify 提供 Laravel Jetstream 的認證後端，或可與 [Laravel Sanctum](/docs/{{version}}/sanctum) 結合獨立使用，為需要與 Laravel 進行認證的 SPA 提供認證。

_[Laravel Jetstream](https://jetstream.laravel.com)_ 是一個強大的應用程式入門套件，它使用 [Tailwind CSS](https://tailwindcss.com)、[Livewire](https://livewire.laravel.com) 和/或 [Inertia](https://inertiajs.com) 驅動的精美現代 UI 來消費和公開 Laravel Fortify 的認證服務。Laravel Jetstream 包含雙因子認證、團隊支援、瀏覽器 session 管理、個人資料管理以及與 [Laravel Sanctum](/docs/{{version}}/sanctum) 的內建整合等可選支援，以提供 API token 認證。Laravel 的 API 認證產品將在下方討論。

<a name="laravels-api-authentication-services"></a>
#### Laravel 的 API 認證服務

Laravel 提供兩個可選套件，協助您管理 API token 並認證使用 API token 發出的請求：[Passport](/docs/{{version}}/passport) 和 [Sanctum](/docs/{{version}}/sanctum)。請注意，這些函式庫與 Laravel 內建的基於 cookie 的認證函式庫並非互斥。這些函式庫主要專注於 API token 認證，而內建的認證服務則專注於基於 cookie 的瀏覽器認證。許多應用程式將同時使用 Laravel 內建的基於 cookie 的認證服務和其中一個 Laravel API 認證套件。

**Passport**

Passport 是一個 OAuth2 認證提供者，提供多種 OAuth2「授權類型 (grant types)」，允許您發行各種不同類型的 token。一般而言，這是一個功能強大且複雜的 API 認證套件。然而，大多數應用程式不需要 OAuth2 規範提供的複雜功能，這對使用者和開發者都可能造成困惑。此外，開發者歷來對於如何使用像 Passport 這樣的 OAuth2 認證提供者來認證 SPA 應用程式或行動應用程式感到困惑。

**Sanctum**

為了解決 OAuth2 的複雜性與開發者的困惑，我們著手建立一個更簡單、更精簡的認證套件，該套件能夠處理來自網頁瀏覽器的第一方 Web 請求和透過 token 發出的 API 請求。隨著 [Laravel Sanctum](/docs/{{version}}/sanctum) 的發布，這個目標得以實現，它應該被視為針對以下應用程式的首選和推薦認證套件：除了 API 之外還將提供第一方 Web UI 的應用程式，或者由與後端 Laravel 應用程式獨立存在的單頁應用程式 (SPA) 驅動的應用程式，或者提供行動用戶端的應用程式。

Laravel Sanctum 是一個混合式的 Web / API 認證套件，可以管理應用程式的整個認證過程。之所以能這樣做，是因為當基於 Sanctum 的應用程式收到請求時，Sanctum 會首先判斷請求是否包含引用已認證 session 的 session cookie。Sanctum 透過呼叫我們之前討論過的 Laravel 內建認證服務來實現這一點。如果請求不是透過 session cookie 進行認證，Sanctum 將檢查請求中是否有 API token。如果存在 API token，Sanctum 將使用該 token 認證請求。若要了解更多關於此過程的資訊，請參閱 Sanctum 的「[運作方式](/docs/{{version}}/sanctum#how-it-works)」文件。

Laravel Sanctum 是我們選擇納入 [Laravel Jetstream](https://jetstream.laravel.com) 應用程式入門套件的 API 套件，因為我們相信它最適合大多數 Web 應用程式的認證需求。

<a name="summary-choosing-your-stack"></a>
#### 總結與選擇您的堆疊

總之，如果您的應用程式將透過瀏覽器存取，且您正在建立單體式 (monolithic) Laravel 應用程式，您的應用程式將使用 Laravel 內建的認證服務。

接下來，如果您的應用程式提供將被第三方使用的 API，您將在 [Passport](/docs/{{version}}/passport) 或 [Sanctum](/docs/{{version}}/sanctum) 之間做出選擇，為您的應用程式提供 API token 認證。一般而言，盡可能應優先選擇 Sanctum，因為它是 API 認證、SPA 認證和行動認證的簡單、完整解決方案，包括對「scopes」或「abilities」的支援。

如果您正在建立一個由 Laravel 後端驅動的單頁應用程式 (SPA)，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。當使用 Sanctum 時，您將需要[手動實作自己的後端認證路由](#authenticating-users) 或利用 [Laravel Fortify](/docs/{{version}}/fortify) 作為無頭 (headless) 認證後端服務，該服務提供註冊、密碼重設、電子郵件驗證等功能的路由和控制器。

當您的應用程式絕對需要 OAuth2 規範提供的所有功能時，可以選擇 Passport。

而且，如果您想快速入門，我們很樂意推薦 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze) 作為快速啟動一個新 Laravel 應用程式的方式，該應用程式已使用我們首選的 Laravel 內建認證服務和 Laravel Sanctum 認證堆疊。

<a name="authentication-quickstart"></a>
## 認證快速入門

> [!WARNING]  
> 本文件此部分討論透過 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits) 認證使用者，其中包含 UI 鷹架，可協助您快速入門。如果您想直接整合 Laravel 的認證系統，請參閱 [手動認證使用者](#authenticating-users) 的文件。


<a name="install-a-starter-kit"></a>
### 安裝入門套件

首先，您應該 [安裝一個 Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。我們目前的入門套件，Laravel Breeze 和 Laravel Jetstream，為將認證功能整合到您的全新 Laravel 應用程式提供了精美設計的起點。

Laravel Breeze 是一個輕量且簡潔的實作，包含了 Laravel 所有認證功能，包括登入、註冊、密碼重設、電子郵件驗證以及密碼確認。Laravel Breeze 的視圖層由簡單的 [Blade 模板](/docs/{{version}}/blade) 組成，並使用 [Tailwind CSS](https://tailwindcss.com) 樣式。此外，Breeze 還提供了基於 [Livewire](https://livewire.laravel.com) 或 [Inertia](https://inertiajs.com) 的鷹架選項，可選擇使用 Vue 或 React 來進行基於 Inertia 的鷹架。

[Laravel Jetstream](https://jetstream.laravel.com) 是一個更強大的應用程式入門套件，支援使用 [Livewire](https://livewire.laravel.com) 或 [Inertia 和 Vue](https://inertiajs.com) 為您的應用程式提供鷹架。此外，Jetstream 還具有雙因子認證、團隊、個人檔案管理、瀏覽器 Session 管理、透過 [Laravel Sanctum](/docs/{{version}}/sanctum) 提供 API 支援、帳號刪除等可選功能。


<a name="retrieving-the-authenticated-user"></a>
### 擷取已認證使用者

安裝認證入門套件並允許使用者註冊並認證您的應用程式後，您通常需要與當前已認證的使用者互動。在處理傳入的請求時，您可以透過 `Auth` Facade 的 `user` 方法來存取已認證的使用者：

    use Illuminate\Support\Facades\Auth;

    // 擷取當前已認證的使用者...
    $user = Auth::user();

    // 擷取當前已認證使用者的 ID...
    $id = Auth::id();

或者，一旦使用者已認證，您可以透過 `Illuminate\Http\Request` 實例來存取已認證的使用者。請記住，型別提示的類別會自動被注入到您的控制器方法中。透過型別提示 `Illuminate\Http\Request` 物件，您可以方便地從應用程式中任何控制器方法透過請求的 `user` 方法存取已認證的使用者：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class FlightController extends Controller
    {
        /**
         * 更新現有航班的資訊。
         */
        public function update(Request $request): RedirectResponse
        {
            $user = $request->user();

            // ...

            return redirect('/flights');
        }
    }


<a name="determining-if-the-current-user-is-authenticated"></a>
#### 判斷當前使用者是否已認證

要判斷發出傳入 HTTP 請求的使用者是否已認證，您可以使用 `Auth` Facade 上的 `check` 方法。如果使用者已認證，此方法將回傳 `true`：

    use Illuminate\Support\Facades\Auth;

    if (Auth::check()) {
        // 使用者已登入...
    }

> [!NOTE]  
> 儘管可以使用 `check` 方法判斷使用者是否已認證，但您通常會使用中介層來驗證使用者是否已認證，然後才允許使用者存取特定的路由/控制器。要了解更多相關資訊，請參閱 [保護路由](/docs/{{version}}/authentication#protecting-routes) 的文件。


<a name="protecting-routes"></a>
### 保護路由

[路由中介層](/docs/{{version}}/middleware) 可以用來只允許已認證的使用者存取給定路由。Laravel 內建了一個 `auth` 中介層，它是 `Illuminate\Auth\Middleware\Authenticate` 類別的 [中介層別名](/docs/{{version}}/middleware#middleware-aliases)。由於此中介層已由 Laravel 內部建立別名，您只需將中介層附加到路由定義即可：

    Route::get('/flights', function () {
        // 只有已認證的使用者才能存取此路由...
    })->middleware('auth');


<a name="redirecting-unauthenticated-users"></a>
#### 重導未認證使用者

當 `auth` 中介層偵測到未認證使用者時，它會將使用者重導到 `login` [具名路由](/docs/{{version}}/routing#named-routes)。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `redirectGuestsTo` 方法來修改此行為：

    use Illuminate\Http\Request;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->redirectGuestsTo('/login');

        // 使用一個閉包...
        $middleware->redirectGuestsTo(fn (Request $request) => route('login'));
    })


<a name="specifying-a-guard"></a>
#### 指定 Guard

將 `auth` 中介層附加到路由時，您還可以指定應使用哪個「guard」來認證使用者。指定的 guard 應對應於 `auth.php` 設定檔中 `guards` 陣列的其中一個鍵：

    Route::get('/flights', function () {
        // 只有已認證的使用者才能存取此路由...
    })->middleware('auth:admin');


<a name="login-throttling"></a>
### 登入節流

如果您正在使用 Laravel Breeze 或 Laravel Jetstream [入門套件](/docs/{{version}}/starter-kits)，速率限制將會自動應用於登入嘗試。預設情況下，如果使用者多次嘗試後未能提供正確的憑證，將無法登入一分鐘。此節流機制對於使用者的使用者名稱/電子郵件地址及其 IP 位址是獨一無二的。

> [!NOTE]  
> 如果您想限制應用程式中其他路由的速率，請參閱 [速率限制文件](/docs/{{version}}/routing#rate-limiting)。

<a name="authenticating-users"></a>
## 手動認證使用者

您不一定要使用 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits)中包含的認證鷹架。如果您選擇不使用此鷹架，則需要直接使用 Laravel 認證類別來管理使用者認證。別擔心，這非常簡單！

我們將透過 `Auth` [Facade](/docs/{{version}}/facades) 存取 Laravel 的認證服務，因此我們需要確保在類別頂部引入 `Auth` Facade。接著，讓我們看看 `attempt` 方法。 `attempt` 方法通常用於處理您應用程式「登入」表單的認證嘗試。如果認證成功，您應該重新產生使用者的 [session](/docs/{{version}}/session) 以防止 [session 固定攻擊](https://en.wikipedia.org/wiki/Session_fixation)：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Support\Facades\Auth;

    class LoginController extends Controller
    {
        /**
         * Handle an authentication attempt.
         */
        public function authenticate(Request $request): RedirectResponse
        {
            $credentials = $request->validate([
                'email' => ['required', 'email'],
                'password' => ['required'],
            ]);

            if (Auth::attempt($credentials)) {
                $request->session()->regenerate();

                return redirect()->intended('dashboard');
            }

            return back()->withErrors([
                'email' => 'The provided credentials do not match our records.',
            ])->onlyInput('email');
        }
    }

`attempt` 方法接受鍵/值陣列作為其第一個引數。陣列中的值將用於在您的資料庫表格中尋找使用者。因此，在上面的範例中，使用者將透過 `email` 欄位的值進行擷取。如果找到使用者，資料庫中儲存的雜湊密碼將與透過陣列傳遞給方法的 `password` 值進行比較。您不應該雜湊傳入請求的 `password` 值，因為框架會自動雜湊該值，然後再將其與資料庫中的雜湊密碼進行比較。如果兩個雜湊密碼匹配，則將為使用者啟動一個已認證的 Session。

請記住，Laravel 的認證服務將根據您的認證 Guard 的「提供者」設定從資料庫中擷取使用者。在預設的 `config/auth.php` 設定檔中，指定了 Eloquent 使用者提供者，並指示它在擷取使用者時使用 `App\Models\User` Model。您可以根據應用程式的需求在設定檔中更改這些值。

如果認證成功，`attempt` 方法將傳回 `true`。否則，將傳回 `false`。

Laravel 重新導向器提供的 `intended` 方法會將使用者重新導向到他們在被認證中介層攔截之前嘗試存取的 URL。如果預期的目的地不可用，可以向此方法提供一個備用 URI。

<a name="specifying-additional-conditions"></a>
#### 指定額外條件

如果需要，您除了使用者的 email 和 password 之外，還可以向認證查詢添加額外的查詢條件。為此，我們只需將查詢條件添加到傳遞給 `attempt` 方法的陣列中。例如，我們可以驗證使用者是否被標記為「active」：

    if (Auth::attempt(['email' => $email, 'password' => $password, 'active' => 1])) {
        // Authentication was successful...
    }

對於複雜的查詢條件，您可以在憑證陣列中提供一個 Closure。此 Closure 將與查詢實例一起被調用，允許您根據應用程式的需求自訂查詢：

    use Illuminate\Database\Eloquent\Builder;

    if (Auth::attempt([
        'email' => $email,
        'password' => $password,
        fn (Builder $query) => $query->has('activeSubscription'),
    ])) {
        // Authentication was successful...
    }

> [!WARNING]  
> 在這些範例中，`email` 不是一個必要選項，它僅作為一個範例。您應該使用資料庫表格中與「username」相對應的任何欄位名稱。

`attemptWhen` 方法接受 Closure 作為其第二個引數，可用於在實際認證使用者之前對潛在使用者執行更廣泛的檢查。該 Closure 接收潛在使用者，並應傳回 `true` 或 `false` 以指示使用者是否可以被認證：

    if (Auth::attemptWhen([
        'email' => $email,
        'password' => $password,
    ], function (User $user) {
        return $user->isNotBanned();
    })) {
        // Authentication was successful...
    }

<a name="accessing-specific-guard-instances"></a>
#### 存取特定 Guard 實例

透過 `Auth` Facade 的 `guard` 方法，您可以指定在認證使用者時要使用的 Guard 實例。這允許您使用完全獨立的 Authenticatable Model 或使用者表格來管理應用程式不同部分的認證。

傳遞給 `guard` 方法的 Guard 名稱應與您 `auth.php` 設定檔中設定的 Guard 之一相對應：

    if (Auth::guard('admin')->attempt($credentials)) {
        // ...
    }

<a name="remembering-users"></a>
### 記住使用者

許多 Web 應用程式在登入表單上提供「記住我」核取方塊。如果您想在應用程式中提供「記住我」功能，可以將布林值作為第二個引數傳遞給 `attempt` 方法。

當此值為 `true` 時，Laravel 將無限期地保持使用者認證狀態，直到他們手動登出。您的 `users` 資料表必須包含字串 `remember_token` 欄位，該欄位將用於儲存「記住我」權杖。新 Laravel 應用程式中包含的 `users` 資料表遷移已包含此欄位：

    use Illuminate\Support\Facades\Auth;

    if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
        // The user is being remembered...
    }

如果您的應用程式提供「記住我」功能，您可以使用 `viaRemember` 方法來判斷當前已認證使用者是否透過「記住我」cookie 進行認證：

    use Illuminate\Support\Facades\Auth;

    if (Auth::viaRemember()) {
        // ...
    }

<a name="other-authentication-methods"></a>
### 其他認證方法

<a name="authenticate-a-user-instance"></a>
#### 認證一個使用者實例

如果您需要將一個現有的使用者實例設定為目前已認證的使用者，可以將該使用者實例傳遞給 `Auth` Facade 的 `login` 方法。指定的使用者實例必須是 `Illuminate\Contracts\Auth\Authenticatable` [契約](/docs/{{version}}/contracts) 的實作。Laravel 內建的 `App\Models\User` Model 已實作此介面。當您已經擁有一個有效的、如使用者剛註冊後的實例時，這種認證方法會很有用：

    use Illuminate\Support\Facades\Auth;

    Auth::login($user);

您可以將一個布林值作為第二個引數傳遞給 `login` 方法。此值表示是否希望已認證的 Session 啟用「記住我」功能。請注意，這代表該 Session 將會無限期地保持認證狀態，直到使用者手動登出應用程式為止：

    Auth::login($user, $remember = true);

如有需要，您可以在呼叫 `login` 方法之前指定認證 guard：

    Auth::guard('admin')->login($user);

<a name="authenticate-a-user-by-id"></a>
#### 透過 ID 認證使用者

要使用資料庫記錄的主鍵認證使用者，您可以使用 `loginUsingId` 方法。此方法接受您希望認證的使用者主鍵：

    Auth::loginUsingId(1);

您可以將一個布林值傳遞給 `loginUsingId` 方法的 `remember` 引數。此值表示是否希望已認證的 Session 啟用「記住我」功能。請注意，這代表該 Session 將會無限期地保持認證狀態，直到使用者手動登出應用程式為止：

    Auth::loginUsingId(1, remember: true);

<a name="authenticate-a-user-once"></a>
#### 一次性認證使用者

您可以使用 `once` 方法，在單一請求中認證使用者。呼叫此方法時，不會使用任何 Session 或 Cookie：

    if (Auth::once($credentials)) {
        // ...
    }

<a name="http-basic-authentication"></a>
## HTTP 基本認證

[HTTP 基本認證](https://en.wikipedia.org/wiki/Basic_access_authentication) 提供一種快速方式來認證應用程式的使用者，而無需設定專用的「登入」頁面。要開始使用，請將 `auth.basic` [中介層](/docs/{{version}}/middleware) 附加到路由。`auth.basic` 中介層已包含在 Laravel 框架中，因此您無需定義它：

    Route::get('/profile', function () {
        // Only authenticated users may access this route...
    })->middleware('auth.basic');

一旦中介層被附加到路由，當您在瀏覽器中存取該路由時，將自動提示您輸入憑證。預設情況下，`auth.basic` 中介層將假定 `users` 資料庫表格上的 `email` 欄位是使用者的「使用者名稱」。

<a name="a-note-on-fastcgi"></a>
#### 關於 FastCGI 的注意事項

如果您正在使用 PHP FastCGI 和 Apache 來服務您的 Laravel 應用程式，HTTP 基本認證可能無法正常運作。為了修正這些問題，您可以將以下行添加到應用程式的 `.htaccess` 檔案中：

```apache
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

<a name="stateless-http-basic-authentication"></a>
### 無狀態 HTTP 基本認證

您也可以在不設定使用者識別 cookie 的情況下使用 HTTP 基本認證。如果您選擇使用 HTTP 認證來驗證應用程式 API 的請求，這會特別有用。為此，請[定義一個中介層](/docs/{{version}}/middleware)來呼叫 `onceBasic` 方法。如果 `onceBasic` 方法沒有回傳任何回應，則請求可以進一步傳遞到應用程式中：

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Auth;
    use Symfony\Component\HttpFoundation\Response;

    class AuthenticateOnceWithBasicAuth
    {
        /**
         * Handle an incoming request.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            return Auth::onceBasic() ?: $next($request);
        }

    }

接著，將此中介層附加到路由：

    Route::get('/api/user', function () {
        // Only authenticated users may access this route...
    })->middleware(AuthenticateOnceWithBasicAuth::class);

<a name="logging-out"></a>
## 登出

要手動將使用者從您的應用程式登出，您可以使用 `Auth` facade 提供的 `logout` 方法。這將從使用者的 Session 中移除認證資訊，以便後續請求不會被認證。

除了呼叫 `logout` 方法之外，建議您使使用者的 Session 失效並重新生成其 [CSRF 權杖](/docs/{{version}}/csrf)。在將使用者登出後，您通常會將使用者重新導向到應用程式的根目錄：

    use Illuminate\Http\Request;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Support\Facades\Auth;

    /**
     * Log the user out of the application.
     */
    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();

        $request->session()->regenerateToken();

        return redirect('/');
    }

<a name="invalidating-sessions-on-other-devices"></a>
### 讓其他裝置的 Session 失效

Laravel 也提供一種機制，可以在不讓使用者目前裝置的 Session 失效的情況下，讓其在其他裝置上活動的 Session 失效並「登出」。此功能通常在使用者變更或更新密碼時使用，如果您想讓其他裝置的 Session 失效，同時保持目前裝置已認證的狀態。

在開始之前，您應該確保 `Illuminate\Session\Middleware\AuthenticateSession` 中介層已包含在應接收 Session 認證的路由上。通常，您應該將此中介層放在路由群組定義中，以便它可以應用於應用程式的大部分路由。預設情況下，`AuthenticateSession` 中介層可以使用 `auth.session` [中介層別名](/docs/{{version}}/middleware#middleware-aliases) 附加到路由：

    Route::middleware(['auth', 'auth.session'])->group(function () {
        Route::get('/', function () {
            // ...
        });
    });

然後，您可以使用 `Auth` facade 提供的 `logoutOtherDevices` 方法。此方法要求使用者確認其目前的密碼，您的應用程式應該透過輸入表單接受此密碼：

    use Illuminate\Support\Facades\Auth;

    Auth::logoutOtherDevices($currentPassword);

當 `logoutOtherDevices` 方法被呼叫時，使用者的其他 Session 將會完全失效，這表示他們將從所有先前認證過的 guards 中「登出」。

<a name="password-confirmation"></a>
## 密碼確認

當您建構應用程式時，有時會有一些操作需要使用者在執行該操作之前，或在使用者被重新導向到應用程式的敏感區域之前，先確認他們的密碼。Laravel 內建了中介層，讓這個過程變得輕而易舉。實作此功能將需要您定義兩個路由：一個路由用於顯示要求使用者確認密碼的視圖，另一個路由用於確認密碼是否有效並將使用者重新導向到其預期的目的地。

> [!NOTE]  
> 以下文件討論了如何直接整合 Laravel 的密碼確認功能；但是，如果您想更快地開始，[Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)包含了對此功能的支持！

<a name="password-confirmation-configuration"></a>
### 設定

使用者確認密碼後，三小時內不會再次被要求確認密碼。不過，您可以透過變更應用程式 `config/auth.php` 設定檔中的 `password_timeout` 設定值，來配置使用者再次被提示輸入密碼前的時間長度。

<a name="password-confirmation-routing"></a>
### 路由

<a name="the-password-confirmation-form"></a>
#### 密碼確認表單

首先，我們將定義一個路由，用來顯示要求使用者確認密碼的視圖：

    Route::get('/confirm-password', function () {
        return view('auth.confirm-password');
    })->middleware('auth')->name('password.confirm');

如您所料，此路由返回的視圖應包含一個帶有 `password` 欄位的表單。此外，您可以在視圖中加入文字，說明使用者正在進入應用程式的受保護區域，並且必須確認他們的密碼。

<a name="confirming-the-password"></a>
#### 確認密碼

接下來，我們將定義一個路由，用於處理「確認密碼」視圖中的表單請求。此路由將負責驗證密碼並將使用者重新導向到其預期的目的地：

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Facades\Redirect;

    Route::post('/confirm-password', function (Request $request) {
        if (! Hash::check($request->password, $request->user()->password)) {
            return back()->withErrors([
                'password' => ['The provided password does not match our records.']
            ]);
        }

        $request->session()->passwordConfirmed();

        return redirect()->intended();
    })->middleware(['auth', 'throttle:6,1']);

在繼續之前，讓我們先更詳細地檢視這個路由。首先，請求中的 `password` 欄位會被判斷是否與已認證使用者的密碼實際相符。如果密碼有效，我們需要通知 Laravel 的 Session，使用者已確認他們的密碼。`passwordConfirmed` 方法將在使用者 Session 中設定一個時間戳，Laravel 可以用它來判斷使用者上次確認密碼的時間。最後，我們可以將使用者重新導向到他們預期的目的地。

<a name="password-confirmation-protecting-routes"></a>
### 保護路由

您應該確保任何執行需要最近密碼確認的操作的路由都已指定 `password.confirm` 中介層。此中介層包含在 Laravel 的預設安裝中，它將自動將使用者預期的目的地儲存在 Session 中，以便使用者在確認密碼後可以被重新導向到該位置。在將使用者預期的目的地儲存在 Session 後，該中介層會將使用者重新導向到 `password.confirm` [命名路由](/docs/{{version}}/routing#named-routes)：

    Route::get('/settings', function () {
        // ...
    })->middleware(['password.confirm']);

    Route::post('/settings', function () {
        // ...
    })->middleware(['password.confirm']);

<a name="adding-custom-guards"></a>
## 新增自訂 Guards

您可以使用 `Auth` Facade 上的 `extend` 方法來定義您自己的認證 Guards。您應該將 `extend` 方法的呼叫放在 [服務提供者](/docs/{{version}}/providers) 中。由於 Laravel 已內建 `AppServiceProvider`，我們可以將程式碼放在該提供者中：

    <?php

    namespace App\Providers;

    use App\Services\Auth\JwtGuard;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Auth;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        // ...

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            Auth::extend('jwt', function (Application $app, string $name, array $config) {
                // Return an instance of Illuminate\Contracts\Auth\Guard...

                return new JwtGuard(Auth::createUserProvider($config['provider']));
            });
        }
    }

如同您在上方範例所見，傳遞給 `extend` 方法的回呼應該要回傳一個 `Illuminate\Contracts\Auth\Guard` 的實作。這個介面包含了一些您需要實作的方法，以定義一個自訂的 Guard。一旦您的自訂 Guard 被定義後，您就可以在 `auth.php` 設定檔的 `guards` 設定中引用該 Guard：

    'guards' => [
        'api' => [
            'driver' => 'jwt',
            'provider' => 'users',
        ],
    ],

<a name="closure-request-guards"></a>
### Closure 請求 Guards

實作自訂的 HTTP 請求式認證系統最簡單的方式是使用 `Auth::viaRequest` 方法。這個方法允許您使用單一的 Closure 快速定義您的認證流程。

首先，在您應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫 `Auth::viaRequest` 方法。`viaRequest` 方法接受一個認證驅動程式名稱作為其第一個引數。這個名稱可以是描述您自訂 Guard 的任何字串。傳遞給該方法的第二個引數應該是一個 Closure，它會接收傳入的 HTTP 請求，並返回一個使用者實例；如果認證失敗，則返回 `null`：

    use App\Models\User;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Auth;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Auth::viaRequest('custom-token', function (Request $request) {
            return User::where('token', (string) $request->token)->first();
        });
    }

一旦您的自訂認證驅動程式被定義，您可以在 `auth.php` 設定檔的 `guards` 設定中將其配置為驅動程式：

    'guards' => [
        'api' => [
            'driver' => 'custom-token',
        ],
    ],

最後，您可以在將認證中介層指定給路由時引用該 Guard：

    Route::middleware('auth:api')->group(function () {
        // ...
    });

<a name="adding-custom-user-providers"></a>
## 新增自訂使用者提供者

如果您沒有使用傳統的關聯式資料庫來儲存使用者，您將需要透過自己的認證使用者提供者來擴展 Laravel。我們將使用 `Auth` Facade 上的 `provider` 方法來定義一個自訂使用者提供者。此使用者提供者解析器應回傳一個 `Illuminate\Contracts\Auth\UserProvider` 實作：

    <?php

    namespace App\Providers;

    use App\Extensions\MongoUserProvider;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Auth;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        // ...

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            Auth::provider('mongo', function (Application $app, array $config) {
                // Return an instance of Illuminate\Contracts\Auth\UserProvider...

                return new MongoUserProvider($app->make('mongo.connection'));
            });
        }
    }

在您使用 `provider` 方法註冊提供者後，您可以在 `auth.php` 設定檔中切換到新的使用者提供者。首先，定義一個使用您新驅動的 `provider`：

    'providers' => [
        'users' => [
            'driver' => 'mongo',
        ],
    ],

最後，您可以在 `guards` 設定中參考此提供者：

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
    ],

<a name="the-user-provider-contract"></a>
### 使用者提供者契約

`Illuminate\Contracts\Auth\UserProvider` 的實作負責從持久性儲存系統（如 MySQL、MongoDB 等）中擷取 `Illuminate\Contracts\Auth\Authenticatable` 的實作。這兩個介面讓 Laravel 認證機制能夠持續運作，無論使用者資料如何儲存，或使用何種類別來表示已認證使用者：

讓我們來看看 `Illuminate\Contracts\Auth\UserProvider` 契約：

    <?php

    namespace Illuminate\Contracts\Auth;

    interface UserProvider
    {
        public function retrieveById($identifier);
        public function retrieveByToken($identifier, $token);
        public function updateRememberToken(Authenticatable $user, $token);
        public function retrieveByCredentials(array $credentials);
        public function validateCredentials(Authenticatable $user, array $credentials);
        public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
    }

`retrieveById` 函式通常會接收一個代表使用者的鍵，例如 MySQL 資料庫中的自動遞增 ID。應擷取與該 ID 相符的 `Authenticatable` 實作並由該方法回傳。

`retrieveByToken` 函式會透過其獨特的 `$identifier` 和「記住我」的 `$token` 來擷取使用者，這些通常儲存在像 `remember_token` 這樣的資料庫欄位中。與上一個方法一樣，此方法應回傳具有相符 token 值的 `Authenticatable` 實作。

`updateRememberToken` 方法會使用新的 `$token` 更新 `$user` 實例的 `remember_token`。在成功的「記住我」認證嘗試或使用者登出時，會為使用者指派一個新的 token。

`retrieveByCredentials` 方法接收一個 credentials 陣列，該陣列是在嘗試透過應用程式進行認證時傳遞給 `Auth::attempt` 方法的。該方法應隨後「查詢」底層持久性儲存中與這些 credentials 相符的使用者。通常，此方法會執行一個帶有「where」條件的查詢，該條件會搜尋一個使用者紀錄，其中「username」欄位的值與 `$credentials['username']` 的值相符。該方法應回傳 `Authenticatable` 的實作。**此方法不應嘗試執行任何密碼驗證或認證。**

`validateCredentials` 方法應將給定的 `$user` 與 `$credentials` 進行比較以認證使用者。例如，此方法通常會使用 `Hash::check` 方法來比較 `$user->getAuthPassword()` 的值與 `$credentials['password']` 的值。此方法應回傳 `true` 或 `false` 以指示密碼是否有效。

`rehashPasswordIfRequired` 方法應在需要且受支援的情況下，對給定的 `$user` 密碼進行再雜湊。例如，此方法通常會使用 `Hash::needsRehash` 方法來判斷 `$credentials['password']` 的值是否需要再雜湊。如果密碼需要再雜湊，該方法應使用 `Hash::make` 方法對密碼進行再雜湊，並更新底層持久性儲存中的使用者紀錄。

<a name="the-authenticatable-contract"></a>
### Authenticatable 契約

既然我們已經探討了 `UserProvider` 上的每個方法，接下來讓我們看看 `Authenticatable` 契約。請記住，使用者提供者應從 `retrieveById`、`retrieveByToken` 和 `retrieveByCredentials` 方法中回傳此介面的實作：

    <?php

    namespace Illuminate\Contracts\Auth;

    interface Authenticatable
    {
        public function getAuthIdentifierName();
        public function getAuthIdentifier();
        public function getAuthPasswordName();
        public function getAuthPassword();
        public function getRememberToken();
        public function setRememberToken($value);
        public function getRememberTokenName();
    }

這個介面很簡單。`getAuthIdentifierName` 方法應回傳使用者的「主鍵」欄位名稱，而 `getAuthIdentifier` 方法應回傳使用者的「主鍵」。在使用 MySQL 後端時，這很可能是指派給使用者紀錄的自動遞增主鍵。`getAuthPasswordName` 方法應回傳使用者密碼欄位的名稱。`getAuthPassword` 方法應回傳使用者的雜湊密碼。

此介面允許認證系統與任何「user」類別協同工作，無論您使用什麼 ORM 或儲存抽象層。預設情況下，Laravel 在 `app/Models` 目錄中包含一個實作此介面的 `App\Models\User` 類別。

<a name="automatic-password-rehashing"></a>
## 自動密碼再雜湊

Laravel 預設的密碼雜湊演算法是 bcrypt。bcrypt 雜湊的「工作因子」可以透過應用程式的 `config/hashing.php` 設定檔或 `BCRYPT_ROUNDS` 環境變數來調整。

通常，bcrypt 工作因子應隨著 CPU / GPU 處理能力的增加而隨時間增加。如果您增加了應用程式的 bcrypt 工作因子，Laravel 將在使用者透過 Laravel 入門套件或您透過 `attempt` 方法[手動認證使用者](#authenticating-users)時，優雅地自動對使用者密碼進行再雜湊。

通常，自動密碼再雜湊不應干擾您的應用程式；但是，您可以透過發布 `hashing` 設定檔來停用此行為：

```shell
php artisan config:publish hashing
```

一旦設定檔發布，您可以將 `rehash_on_login` 設定值設定為 `false`：

```php
'rehash_on_login' => false,
```

<a name="events"></a>
## 事件

Laravel 在認證過程中分派多種[事件](/docs/{{version}}/events)。您可以為以下任何事件[定義監聽器](/docs/{{version}}/events)：

<div class="overflow-auto">

| 事件名稱 |
| --- |
| `Illuminate\Auth\Events\Registered` |
| `Illuminate\Auth\Events\Attempting` |
| `Illuminate\Auth\Events\Authenticated` |
| `Illuminate\Auth\Events\Login` |
| `Illuminate\Auth\Events\Failed` |
| `Illuminate\Auth\Events\Validated` |
| `Illuminate\Auth\Events\Verified` |
| `Illuminate\Auth\Events\Logout` |
| `Illuminate\Auth\Events\CurrentDeviceLogout` |
| `Illuminate\Auth\Events\OtherDeviceLogout` |
| `Illuminate\Auth\Events\Lockout` |
| `Illuminate\Auth\Events\PasswordReset` |

</div>