# 認證

- [簡介](#introduction)
    - [入門套件](#starter-kits)
    - [資料庫考量](#introduction-database-considerations)
    - [生態系概覽](#ecosystem-overview)
- [認證快速入門](#authentication-quickstart)
    - [安裝入門套件](#install-a-starter-kit)
    - [取得已認證使用者](#retrieving-the-authenticated-user)
    - [保護路由](#protecting-routes)
    - [登入節流](#login-throttling)
- [手動認證使用者](#authenticating-users)
    - [記住使用者](#remembering-users)
    - [其他認證方法](#other-authentication-methods)
- [HTTP 基本認證](#http-basic-authentication)
    - [無狀態 HTTP 基本認證](#stateless-http-basic-authentication)
- [登出](#logging-out)
    - [使其他裝置上的 Session 失效](#invalidating-sessions-on-other-devices)
- [密碼確認](#password-confirmation)
    - [設定](#password-confirmation-configuration)
    - [路由](#password-confirmation-routing)
    - [保護路由](#password-confirmation-protecting-routes)
- [新增自訂 Guard](#adding-custom-guards)
    - [Closure 請求 Guard](#closure-request-guards)
- [新增自訂使用者 Provider](#adding-custom-user-providers)
    - [User Provider 契約](#the-user-provider-contract)
    - [Authenticatable 契約](#the-authenticatable-contract)
- [自動密碼重新雜湊](#automatic-password-rehashing)
- [社群認證](/docs/{{version}}/socialite)
- [事件](#events)

<a name="introduction"></a>
## 簡介

許多 Web 應用程式都提供使用者認證並「登入」應用程式的方式。在 Web 應用程式中實作此功能可能是一項複雜且潛在風險的任務。因此，Laravel 致力於提供您所需的工具，以快速、安全且輕鬆地實作認證。

Laravel 的認證功能核心由「Guard」和「Provider」組成。Guard 定義了每個請求如何認證使用者。例如，Laravel 內建了一個 `session` Guard，它使用 Session 儲存和 Cookie 來維護狀態。

Provider 定義了如何從您的持久儲存中取得使用者。Laravel 內建支援使用 [Eloquent](/docs/{{version}}/eloquent) 和資料庫查詢產生器來取得使用者。但是，您可以根據應用程式的需求自由定義額外的 Provider。

您的應用程式認證設定檔位於 `config/auth.php`。此檔案包含幾個詳細說明如何調整 Laravel 認證服務行為的選項。

> [!NOTE]
> Guard 和 Provider 不應與「角色」和「權限」混淆。要了解更多關於透過權限授權使用者操作的資訊，請參閱 [授權](/docs/{{version}}/authorization) 說明文件。

<a name="starter-kits"></a>
### 入門套件

想要快速開始嗎？在全新的 Laravel 應用程式中安裝 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。遷移資料庫後，在瀏覽器中導覽至 `/register` 或任何其他分配給您應用程式的 URL。入門套件將負責建構您的整個認證系統！

**即使您選擇不在最終的 Laravel 應用程式中使用入門套件，安裝 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze) 入門套件也是一個絕佳的機會，可以學習如何在實際的 Laravel 專案中實作所有 Laravel 的認證功能。** 由於 Laravel Breeze 會為您建立認證控制器、路由和視圖，您可以檢查這些檔案中的程式碼，以了解 Laravel 的認證功能如何實作。

<a name="introduction-database-considerations"></a>
### 資料庫考量

預設情況下，Laravel 在您的 `app/Models` 目錄中包含一個 `App\Models\User` [Eloquent 模型](/docs/{{version}}/eloquent)。此模型可用於預設的 Eloquent 認證驅動。

如果您的應用程式未使用 Eloquent，您可以使用 `database` 認證 Provider，它使用 Laravel 查詢產生器。如果您的應用程式使用 MongoDB，請查看 MongoDB 官方的 [Laravel 使用者認證說明文件](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/user-authentication/)。

為 `App\Models\User` 模型建構資料庫 Schema 時，請確保密碼欄位長度至少為 60 個字元。當然，新 Laravel 應用程式中包含的 `users` 資料表遷移已經建立了一個超過此長度的欄位。

此外，您應該驗證您的 `users` (或等效) 資料表包含一個可為 null 的字串 `remember_token` 欄位，長度為 100 個字元。此欄位將用於儲存使用者在登入應用程式時選擇「記住我」選項的 Token。同樣，新 Laravel 應用程式中包含的預設 `users` 資料表遷移已經包含此欄位。

<a name="ecosystem-overview"></a>
### 生態系概覽

Laravel 提供了幾個與認證相關的套件。在繼續之前，我們將回顧 Laravel 中的一般認證生態系，並討論每個套件的預期用途。

首先，考慮認證如何運作。當使用 Web 瀏覽器時，使用者將透過登入表單提供其使用者名稱和密碼。如果這些憑證正確，應用程式將在使用者 [Session](/docs/{{version}}/session) 中儲存有關已認證使用者的資訊。發給瀏覽器的 Cookie 包含 Session ID，以便後續對應用程式的請求可以將使用者與正確的 Session 關聯起來。收到 Session Cookie 後，應用程式將根據 Session ID 取得 Session 資料，注意認證資訊已儲存在 Session 中，並將使用者視為「已認證」。

當遠端服務需要認證才能存取 API 時，通常不使用 Cookie 進行認證，因為沒有 Web 瀏覽器。相反，遠端服務在每個請求上向 API 發送一個 API Token。應用程式可以根據有效 API Token 的資料表驗證傳入的 Token，並「認證」該請求是由與該 API Token 關聯的使用者執行。

<a name="laravels-built-in-browser-authentication-services"></a>
#### Laravel 內建的瀏覽器認證服務

Laravel 包含內建的認證和 Session 服務，通常透過 `Auth` 和 `Session` Facade 存取。這些功能為從 Web 瀏覽器發起的請求提供基於 Cookie 的認證。它們提供允許您驗證使用者憑證和認證使用者的方法。此外，這些服務將自動在使用者 Session 中儲存適當的認證資料並發出使用者的 Session Cookie。本說明文件中包含如何使用這些服務的討論。

**應用程式入門套件**

如本說明文件所述，您可以手動與這些認證服務互動，以建構您應用程式自己的認證層。但是，為了幫助您更快地開始，我們發布了 [免費套件](/docs/{{version}}/starter-kits)，這些套件提供了整個認證層的強大、現代的建構。這些套件是 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze)、[Laravel Jetstream](/docs/{{version}}/starter-kits#laravel-jetstream) 和 [Laravel Fortify](/docs/{{version}}/fortify)。

_Laravel Breeze_ 是 Laravel 所有認證功能的簡單、最小實作，包括登入、註冊、密碼重設、電子郵件驗證和密碼確認。Laravel Breeze 的視圖層由簡單的 [Blade 模板](/docs/{{version}}/blade) 組成，並使用 [Tailwind CSS](https://tailwindcss.com) 樣式化。要開始使用，請查看 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits) 的說明文件。

_Laravel Fortify_ 是 Laravel 的無頭認證後端，它實作了本說明文件中發現的許多功能，包括基於 Cookie 的認證以及雙因素認證和電子郵件驗證等其他功能。Fortify 為 Laravel Jetstream 提供認證後端，或者可以獨立與 [Laravel Sanctum](/docs/{{version}}/sanctum) 結合使用，為需要與 Laravel 認證的 SPA 提供認證。

_[Laravel Jetstream](https://jetstream.laravel.com)_ 是一個強大的應用程式入門套件，它使用 [Tailwind CSS](https://tailwindcss.com)、[Livewire](https://livewire.laravel.com) 和/或 [Inertia](https://inertiajs.com) 提供美觀、現代的 UI，並消耗和公開 Laravel Fortify 的認證服務。Laravel Jetstream 包含雙因素認證、團隊支援、瀏覽器 Session 管理、個人資料管理的可選支援，以及與 [Laravel Sanctum](/docs/{{version}}/sanctum) 的內建整合，以提供 API Token 認證。Laravel 的 API 認證產品將在下面討論。

<a name="laravels-api-authentication-services"></a>
#### Laravel 的 API 認證服務

Laravel 提供了兩個可選套件來協助您管理 API Token 和認證使用 API Token 發出的請求：[Passport](/docs/{{version}}/passport) 和 [Sanctum](/docs/{{version}}/sanctum)。請注意，這些函式庫和 Laravel 內建的基於 Cookie 的認證函式庫並非互斥。這些函式庫主要專注於 API Token 認證，而內建的認證服務則專注於基於 Cookie 的瀏覽器認證。許多應用程式將同時使用 Laravel 內建的基於 Cookie 的認證服務和其中一個 Laravel API 認證套件。

**Passport**

Passport 是一個 OAuth2 認證 Provider，提供各種 OAuth2「授權類型」，允許您發行各種型別的 Token。總體而言，這是一個用於 API 認證的強大且複雜的套件。然而，大多數應用程式不需要 OAuth2 規範提供的複雜功能，這對使用者和開發人員來說都可能令人困惑。此外，開發人員歷來對如何使用 Passport 等 OAuth2 認證 Provider 認證 SPA 應用程式或行動應用程式感到困惑。

**Sanctum**

為了回應 OAuth2 的複雜性和開發人員的困惑，我們著手建立一個更簡單、更精簡的認證套件，可以處理來自 Web 瀏覽器的第一方 Web 請求和透過 Token 的 API 請求。這個目標隨著 [Laravel Sanctum](/docs/{{version}}/sanctum) 的發布而實現，對於將提供第一方 Web UI 以及 API 的應用程式，或者將由獨立於後端 Laravel 應用程式的單頁應用程式 (SPA) 提供支援的應用程式，或者提供行動用戶端的應用程式，Sanctum 應被視為首選和推薦的認證套件。

Laravel Sanctum 是一個混合 Web / API 認證套件，可以管理您應用程式的整個認證過程。這是可能的，因為當基於 Sanctum 的應用程式收到請求時，Sanctum 將首先判斷請求是否包含引用已認證 Session 的 Session Cookie。Sanctum 透過呼叫我們前面討論的 Laravel 內建認證服務來實現這一點。如果請求未透過 Session Cookie 認證，Sanctum 將檢查請求中是否存在 API Token。如果存在 API Token，Sanctum 將使用該 Token 認證請求。要了解有關此過程的更多資訊，請參閱 Sanctum 的「[運作方式](/docs/{{version}}/sanctum#how-it-works)」說明文件。

Laravel Sanctum 是我們選擇包含在 [Laravel Jetstream](https://jetstream.laravel.com) 應用程式入門套件中的 API 套件，因為我們相信它最適合大多數 Web 應用程式的認證需求。

<a name="summary-choosing-your-stack"></a>
#### 總結與選擇您的技術棧

總之，如果您的應用程式將透過瀏覽器存取，並且您正在建構一個單體 Laravel 應用程式，您的應用程式將使用 Laravel 內建的認證服務。

接下來，如果您的應用程式提供將由第三方使用的 API，您將在 [Passport](/docs/{{version}}/passport) 或 [Sanctum](/docs/{{version}}/sanctum) 之間進行選擇，為您的應用程式提供 API Token 認證。一般來說，應盡可能優先選擇 Sanctum，因為它是一個簡單、完整的 API 認證、SPA 認證和行動認證解決方案，包括對「Scope」或「Abilities」的支援。

如果您正在建構一個將由 Laravel 後端提供支援的單頁應用程式 (SPA)，您應該使用 [Laravel Sanctum](/docs/{{version}}/sanctum)。使用 Sanctum 時，您將需要 [手動實作您自己的後端認證路由](#authenticating-users) 或利用 [Laravel Fortify](/docs/{{version}}/fortify) 作為無頭認證後端服務，提供註冊、密碼重設、電子郵件驗證等功能的路由和控制器。

當您的應用程式絕對需要 OAuth2 規範提供的所有功能時，可以選擇 Passport。

而且，如果您想快速開始，我們很高興推薦 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze) 作為快速啟動新的 Laravel 應用程式的方法，該應用程式已經使用我們首選的 Laravel 內建認證服務和 Laravel Sanctum 認證技術棧。

<a name="authentication-quickstart"></a>
## 認證快速入門

> [!WARNING]
> 本說明文件部分討論透過 [Laravel 應用程式入門套件](/docs/{{version}}/starter-kits) 認證使用者，其中包含 UI 建構以幫助您快速開始。如果您想直接與 Laravel 的認證系統整合，請查看有關 [手動認證使用者](#authenticating-users) 的說明文件。

<a name="install-a-starter-kit"></a>
### 安裝入門套件

首先，您應該 [安裝一個 Laravel 應用程式入門套件](/docs/{{version}}/starter-kits)。我們目前的入門套件，Laravel Breeze 和 Laravel Jetstream，為將認證整合到您的全新 Laravel 應用程式中提供了精美設計的起點。

Laravel Breeze 是 Laravel 所有認證功能的最小、簡單實作，包括登入、註冊、密碼重設、電子郵件驗證和密碼確認。Laravel Breeze 的視圖層由簡單的 [Blade 模板](/docs/{{version}}/blade) 組成，並使用 [Tailwind CSS](https://tailwindcss.com) 樣式化。此外，Breeze 還提供基於 [Livewire](https://livewire.laravel.com) 或 [Inertia](https://inertiajs.com) 的建構選項，並可選擇使用 Vue 或 React 進行基於 Inertia 的建構。

[Laravel Jetstream](https://jetstream.laravel.com) 是一個更強大的應用程式入門套件，包含使用 [Livewire](https://livewire.laravel.com) 或 [Inertia 和 Vue](https://inertiajs.com) 建構應用程式的支援。此外，Jetstream 還具有雙因素認證、團隊、個人資料管理、瀏覽器 Session 管理、透過 [Laravel Sanctum](/docs/{{version}}/sanctum) 的 API 支援、帳戶刪除等可選支援。

<a name="retrieving-the-authenticated-user"></a>
### 取得已認證使用者

安裝認證入門套件並允許使用者註冊和認證您的應用程式後，您通常需要與目前已認證的使用者互動。在處理傳入請求時，您可以透過 `Auth` Facade 的 `user` 方法存取已認證的使用者：

    use Illuminate\Support\Facades\Auth;

    // 取得目前已認證的使用者...
    $user = Auth::user();

    // 取得目前已認證使用者的 ID...
    $id = Auth::id();

或者，一旦使用者經過認證，您可以透過 `Illuminate\Http\Request` 實例存取已認證的使用者。請記住，型別提示的類別將自動注入到您的控制器方法中。透過型別提示 `Illuminate\Http\Request` 物件，您可以透過請求的 `user` 方法方便地從應用程式中的任何控制器方法存取已認證的使用者：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class FlightController extends Controller
    {
        /**
         * 更新現有航班的航班資訊。
         */
        public function update(Request $request): RedirectResponse
        {
            $user = $request->user();

            // ...

            return redirect('/flights');
        }
    }

<a name="determining-if-the-current-user-is-authenticated"></a>
#### 判斷目前使用者是否已認證

要判斷發出傳入 HTTP 請求的使用者是否已認證，您可以使用 `Auth` Facade 上的 `check` 方法。如果使用者已認證，此方法將返回 `true`：

    use Illuminate\Support\Facades\Auth;

    if (Auth::check()) {
        // 使用者已登入...
    }

> [!NOTE]
> 儘管可以使用 `check` 方法判斷使用者是否已認證，但您通常會使用 Middleware 在允許使用者存取某些路由/控制器之前驗證使用者是否已認證。要了解更多資訊，請查看有關 [保護路由](#protecting-routes) 的說明文件。

<a name="protecting-routes"></a>
### 保護路由

[路由 Middleware](/docs/{{version}}/middleware) 可用於僅允許已認證的使用者存取給定路由。Laravel 內建了一個 `auth` Middleware，它是 `Illuminate\Auth\Middleware\Authenticate` 類別的 [Middleware 別名](/docs/{{version}}/middleware#middleware-aliases)。由於此 Middleware 已由 Laravel 內部別名，您只需將 Middleware 附加到路由定義即可：

    Route::get('/flights', function () {
        // 只有已認證的使用者才能存取此路由...
    })->middleware('auth');

<a name="redirecting-unauthenticated-users"></a>
#### 重新導向未認證使用者

當 `auth` Middleware 偵測到未認證的使用者時，它會將使用者重新導向到 `login` [具名路由](/docs/{{version}}/routing#named-routes)。您可以使用應用程式 `bootstrap/app.php` 檔案中的 `redirectGuestsTo` 方法修改此行為：

    use Illuminate\Http\Request;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->redirectGuestsTo('/login');

        // 使用 Closure...
        $middleware->redirectGuestsTo(fn (Request $request) => route('login'));
    })

<a name="specifying-a-guard"></a>
#### 指定 Guard

將 `auth` Middleware 附加到路由時，您還可以指定應使用哪個「Guard」來認證使用者。指定的 Guard 應與您 `auth.php` 設定檔中 `guards` 陣列中的其中一個鍵相對應：

    Route::get('/flights', function () {
        // 只有已認證的使用者才能存取此路由...
    })->middleware('auth:admin');

<a name="login-throttling"></a>
### 登入節流

如果您使用 Laravel Breeze 或 Laravel Jetstream [入門套件](/docs/{{version}}/starter-kits)，速率限制將自動應用於登入嘗試。預設情況下，如果使用者在多次嘗試後未能提供正確的憑證，他們將在一分鐘內無法登入。節流對於使用者的使用者名稱/電子郵件地址及其 IP 地址是唯一的。

> [!NOTE]
> 如果您想對應用程式中的其他路由進行速率限制，請查看 [速率限制說明文件](/docs/{{version}}/routing#rate-limiting)。

<a name="authenticating-users"></a>
## 手動認證使用者

您不需要使用 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits) 中包含的認證建構。如果您選擇不使用此建構，您將需要直接使用 Laravel 認證類別來管理使用者認證。別擔心，這很簡單！

我們將透過 `Auth` [Facade](/docs/{{version}}/facades) 存取 Laravel 的認證服務，因此我們需要確保在類別頂部匯入 `Auth` Facade。接下來，讓我們看看 `attempt` 方法。`attempt` 方法通常用於處理來自您應用程式「登入」表單的認證嘗試。如果認證成功，您應該重新產生使用者的 [Session](/docs/{{version}}/session) 以防止 [Session 劫持](https://en.wikipedia.org/wiki/Session_fixation)：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Support\Facades\Auth;

    class LoginController extends Controller
    {
        /**
         * 處理認證嘗試。
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
                'email' => '提供的憑證與我們的記錄不符。',
            ])->onlyInput('email');
        }
    }

`attempt` 方法接受一個鍵/值對陣列作為其第一個引數。陣列中的值將用於在您的資料庫表中尋找使用者。因此，在上面的範例中，使用者將透過 `email` 欄位的值取得。如果找到使用者，資料庫中儲存的雜湊密碼將與透過陣列傳遞給方法的 `password` 值進行比較。您不應該雜湊傳入請求的 `password` 值，因為框架會自動雜湊該值，然後再將其與資料庫中的雜湊密碼進行比較。如果兩個雜湊密碼匹配，將為使用者啟動一個已認證的 Session。

請記住，Laravel 的認證服務將根據您的認證 Guard 的「Provider」設定從您的資料庫中取得使用者。在預設的 `config/auth.php` 設定檔中，指定了 Eloquent 使用者 Provider，並指示它在取得使用者時使用 `App\Models\User` 模型。您可以根據應用程式的需求在設定檔中更改這些值。

如果認證成功，`attempt` 方法將返回 `true`。否則，將返回 `false`。

Laravel 重新導向器提供的 `intended` 方法會將使用者重新導向到他們在被認證 Middleware 攔截之前嘗試存取的 URL。如果預期目的地不可用，可以為此方法提供一個備用 URI。

<a name="specifying-additional-conditions"></a>
#### 指定額外條件

如果您願意，除了使用者的電子郵件和密碼之外，您還可以向認證查詢添加額外的查詢條件。為此，我們只需將查詢條件添加到傳遞給 `attempt` 方法的陣列中。例如，我們可以驗證使用者是否被標記為「active」：

    if (Auth::attempt(['email' => $email, 'password' => $password, 'active' => 1])) {
        // 認證成功...
    }

對於複雜的查詢條件，您可以在憑證陣列中提供一個 Closure。此 Closure 將使用查詢實例調用，允許您根據應用程式的需求自訂查詢：

    use Illuminate\Database\Eloquent\Builder;

    if (Auth::attempt([
        'email' => $email,
        'password' => $password,
        fn (Builder $query) => $query->has('activeSubscription'),
    ])) {
        // 認證成功...
    }

> [!WARNING]
> 在這些範例中，`email` 不是必需選項，它僅用作範例。您應該使用與資料庫表中「使用者名稱」相對應的任何欄位名稱。

`attemptWhen` 方法接受一個 Closure 作為其第二個引數，可用於在實際認證使用者之前對潛在使用者進行更廣泛的檢查。Closure 接收潛在使用者，並應返回 `true` 或 `false` 以指示使用者是否可以被認證：

    if (Auth::attemptWhen([
        'email' => $email,
        'password' => $password,
    ], function (User $user) {
        return $user->isNotBanned();
    })) {
        // 認證成功...
    }

<a name="accessing-specific-guard-instances"></a>
#### 存取特定 Guard 實例

透過 `Auth` Facade 的 `guard` 方法，您可以指定在認證使用者時要使用的 Guard 實例。這允許您使用完全獨立的 Authenticatable 模型或使用者資料表來管理應用程式不同部分的認證。

傳遞給 `guard` 方法的 Guard 名稱應與您 `auth.php` 設定檔中設定的其中一個 Guard 相對應：

    if (Auth::guard('admin')->attempt($credentials)) {
        // ...
    }

<a name="remembering-users"></a>
### 記住使用者

許多 Web 應用程式在其登入表單上提供「記住我」核取方塊。如果您想在應用程式中提供「記住我」功能，您可以將布林值作為第二個引數傳遞給 `attempt` 方法。

當此值為 `true` 時，Laravel 將無限期地保持使用者認證，直到他們手動登出。您的 `users` 資料表必須包含字串 `remember_token` 欄位，該欄位將用於儲存「記住我」Token。新 Laravel 應用程式中包含的 `users` 資料表遷移已經包含此欄位：

    use Illuminate\Support\Facades\Auth;

    if (Auth::attempt(['email' => $email, 'password' => $password], $remember)) {
        // 使用者被記住了...
    }

如果您的應用程式提供「記住我」功能，您可以使用 `viaRemember` 方法來判斷目前已認證的使用者是否使用「記住我」Cookie 進行認證：

    use Illuminate\Support\Facades\Auth;

    if (Auth::viaRemember()) {
        // ...
    }

<a name="other-authentication-methods"></a>
### 其他認證方法

<a name="authenticate-a-user-instance"></a>
#### 認證使用者實例

如果您需要將現有使用者實例設定為目前已認證的使用者，您可以將使用者實例傳遞給 `Auth` Facade 的 `login` 方法。給定的使用者實例必須是 `Illuminate\Contracts\Auth\Authenticatable` [契約](/docs/{{version}}/contracts) 的實作。Laravel 內建的 `App\Models\User` 模型已經實作此介面。當您已經擁有有效的使用者實例時，例如在使用者註冊應用程式後，此認證方法非常有用：

    use Illuminate\Support\Facades\Auth;

    Auth::login($user);

您可以將布林值作為第二個引數傳遞給 `login` 方法。此值指示是否需要為已認證的 Session 提供「記住我」功能。請記住，這表示 Session 將無限期地保持認證，直到使用者手動登出應用程式：

    Auth::login($user, $remember = true);

如果需要，您可以在呼叫 `login` 方法之前指定認證 Guard：

    Auth::guard('admin')->login($user);

<a name="authenticate-a-user-by-id"></a>
#### 透過 ID 認證使用者

要使用資料庫記錄的主鍵認證使用者，您可以使用 `loginUsingId` 方法。此方法接受您要認證的使用者的主鍵：

    Auth::loginUsingId(1);

您可以將布林值傳遞給 `loginUsingId` 方法的 `remember` 引數。此值指示是否需要為已認證的 Session 提供「記住我」功能。請記住，這表示 Session 將無限期地保持認證，直到使用者手動登出應用程式：

    Auth::loginUsingId(1, remember: true);

<a name="authenticate-a-user-once"></a>
#### 認證使用者一次

您可以使用 `once` 方法為單一請求認證應用程式中的使用者。呼叫此方法時不會使用 Session 或 Cookie：

    if (Auth::once($credentials)) {
        // ...
    }

<a name="http-basic-authentication"></a>
## HTTP 基本認證

[HTTP 基本認證](https://en.wikipedia.org/wiki/Basic_access_authentication) 提供了一種快速認證應用程式使用者的方法，而無需設定專用的「登入」頁面。要開始使用，請將 `auth.basic` [Middleware](/docs/{{version}}/middleware) 附加到路由。`auth.basic` Middleware 包含在 Laravel 框架中，因此您無需定義它：

    Route::get('/profile', function () {
        // 只有已認證的使用者才能存取此路由...
    })->middleware('auth.basic');

一旦 Middleware 附加到路由，當您在瀏覽器中存取路由時，系統將自動提示您輸入憑證。預設情況下，`auth.basic` Middleware 將假定您 `users` 資料庫表上的 `email` 欄位是使用者的「使用者名稱」。

<a name="a-note-on-fastcgi"></a>
#### 關於 FastCGI 的注意事項

如果您使用 PHP FastCGI 和 Apache 來服務您的 Laravel 應用程式，HTTP 基本認證可能無法正常運作。為了解決這些問題，可以將以下行添加到您應用程式的 `.htaccess` 檔案中：

```apache
RewriteCond %{HTTP:Authorization} ^(.+)$
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

<a name="stateless-http-basic-authentication"></a>
### 無狀態 HTTP 基本認證

您也可以使用 HTTP 基本認證，而無需在 Session 中設定使用者識別碼 Cookie。這主要在您選擇使用 HTTP 認證來認證對應用程式 API 的請求時很有用。為此，[定義一個 Middleware](/docs/{{version}}/middleware)，它呼叫 `onceBasic` 方法。如果 `onceBasic` 方法沒有返回回應，則請求可以進一步傳遞到應用程式中：

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Auth;
    use Symfony\Component\HttpFoundation\Response;

    class AuthenticateOnceWithBasicAuth
    {
        /**
         * 處理傳入請求。
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            return Auth::onceBasic() ?: $next($request);
        }

    }

接下來，將 Middleware 附加到路由：

    Route::get('/api/user', function () {
        // 只有已認證的使用者才能存取此路由...
    })->middleware(AuthenticateOnceWithBasicAuth::class);

<a name="logging-out"></a>
## 登出

要手動將使用者登出應用程式，您可以使用 `Auth` Facade 提供的 `logout` 方法。這將從使用者的 Session 中移除認證資訊，以便後續請求不會被認證。

除了呼叫 `logout` 方法之外，建議您使使用者的 Session 失效並重新產生其 [CSRF Token](/docs/{{version}}/csrf)。登出使用者後，您通常會將使用者重新導向到應用程式的根目錄：

    use Illuminate\Http\Request;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Support\Facades\Auth;

    /**
     * 將使用者登出應用程式。
     */
    public function logout(Request $request): RedirectResponse
    {
        Auth::logout();

        $request->session()->invalidate();

        $request->session()->regenerateToken();

        return redirect('/');
    }

<a name="invalidating-sessions-on-other-devices"></a>
### 使其他裝置上的 Session 失效

Laravel 還提供了一種機制，用於使使用者在其他裝置上活動的 Session 失效並「登出」，而不會使他們目前裝置上的 Session 失效。此功能通常在使用者更改或更新密碼時使用，並且您希望使其他裝置上的 Session 失效，同時保持目前裝置已認證。

在開始之前，您應該確保 `Illuminate\Session\Middleware\AuthenticateSession` Middleware 包含在應接收 Session 認證的路由上。通常，您應該將此 Middleware 放在路由群組定義上，以便它可以應用於應用程式的大部分路由。預設情況下，`AuthenticateSession` Middleware 可以使用 `auth.session` [Middleware 別名](/docs/{{version}}/middleware#middleware-aliases) 附加到路由：

    Route::middleware(['auth', 'auth.session'])->group(function () {
        Route::get('/', function () {
            // ...
        });
    });

然後，您可以使用 `Auth` Facade 提供的 `logoutOtherDevices` 方法。此方法要求使用者確認其目前密碼，您的應用程式應透過輸入表單接受此密碼：

    use Illuminate\Support\Facades\Auth;

    Auth::logoutOtherDevices($currentPassword);

當調用 `logoutOtherDevices` 方法時，使用者的其他 Session 將完全失效，這表示他們將從他們之前認證的所有 Guard 中「登出」。

<a name="password-confirmation"></a>
## 密碼確認

在建構應用程式時，您偶爾會遇到需要使用者在執行操作之前或在使用者重新導向到應用程式的敏感區域之前確認其密碼的操作。Laravel 包含內建 Middleware，使此過程變得輕而易舉。實作此功能將要求您定義兩個路由：一個路由用於顯示要求使用者確認其密碼的視圖，另一個路由用於確認密碼有效並將使用者重新導向到其預期目的地。

> [!NOTE]
> 以下說明文件討論如何直接與 Laravel 的密碼確認功能整合；但是，如果您想更快地開始，[Laravel 應用程式入門套件](/docs/{{version}}/starter-kits) 包含對此功能的支援！

<a name="password-confirmation-configuration"></a>
### 設定

確認密碼後，使用者在三小時內不會再次被要求確認密碼。但是，您可以透過更改應用程式 `config/auth.php` 設定檔中的 `password_timeout` 設定值來設定使用者重新提示密碼之前的時間長度。

<a name="password-confirmation-routing"></a>
### 路由

<a name="the-password-confirmation-form"></a>
#### 密碼確認表單

首先，我們將定義一個路由來顯示一個要求使用者確認其密碼的視圖：

    Route::get('/confirm-password', function () {
        return view('auth.confirm-password');
    })->middleware('auth')->name('password.confirm');

正如您所預期的，此路由返回的視圖應該有一個包含 `password` 欄位的表單。此外，請隨意在視圖中包含文字，解釋使用者正在進入應用程式的受保護區域，並且必須確認其密碼。

<a name="confirming-the-password"></a>
#### 確認密碼

接下來，我們將定義一個路由來處理來自「確認密碼」視圖的表單請求。此路由將負責驗證密碼並將使用者重新導向到其預期目的地：

    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Facades\Redirect;

    Route::post('/confirm-password', function (Request $request) {
        if (! Hash::check($request->password, $request->user()->password)) {
            return back()->withErrors([
                'password' => ['提供的密碼與我們的記錄不符。']
            ]);
        }

        $request->session()->passwordConfirmed();

        return redirect()->intended();
    })->middleware(['auth', 'throttle:6,1']);

在繼續之前，讓我們更詳細地檢查此路由。首先，確定請求的 `password` 欄位確實與已認證使用者的密碼匹配。如果密碼有效，我們需要通知 Laravel 的 Session 使用者已確認其密碼。`passwordConfirmed` 方法將在使用者 Session 中設定一個時間戳，Laravel 可以使用它來判斷使用者上次確認密碼的時間。最後，我們可以將使用者重新導向到其預期目的地。

<a name="password-confirmation-protecting-routes"></a>
### 保護路由

您應該確保任何執行需要最近密碼確認的操作的路由都分配了 `password.confirm` Middleware。此 Middleware 包含在 Laravel 的預設安裝中，並會自動將使用者的預期目的地儲存在 Session 中，以便使用者在確認密碼後可以重新導向到該位置。在 Session 中儲存使用者的預期目的地後，Middleware 會將使用者重新導向到 `password.confirm` [具名路由](/docs/{{version}}/routing#named-routes)：

    Route::get('/settings', function () {
        // ...
    })->middleware(['password.confirm']);

    Route::post('/settings', function () {
        // ...
    })->middleware(['password.confirm']);

<a name="adding-custom-guards"></a>
## 新增自訂 Guard

您可以使用 `Auth` Facade 上的 `extend` 方法定義自己的認證 Guard。您應該將對 `extend` 方法的呼叫放在 [服務 Provider](/docs/{{version}}/providers) 中。由於 Laravel 已經內建了一個 `AppServiceProvider`，我們可以將程式碼放在該 Provider 中：

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
         * 啟動任何應用程式服務。
         */
        public function boot(): void
        {
            Auth::extend('jwt', function (Application $app, string $name, array $config) {
                // 返回 Illuminate\Contracts\Auth\Guard 的實例...

                return new JwtGuard(Auth::createUserProvider($config['provider']));
            });
        }
    }

如您在上面的範例中看到的，傳遞給 `extend` 方法的回呼應該返回 `Illuminate\Contracts\Auth\Guard` 的實作。此介面包含您需要實作的幾個方法來定義自訂 Guard。一旦定義了自訂 Guard，您就可以在 `auth.php` 設定檔的 `guards` 設定中引用該 Guard：

    'guards' => [
        'api' => [
            'driver' => 'jwt',
            'provider' => 'users',
        ],
    ],

<a name="closure-request-guards"></a>
### Closure 請求 Guard

實作自訂、基於 HTTP 請求的認證系統最簡單的方法是使用 `Auth::viaRequest` 方法。此方法允許您使用單個 Closure 快速定義您的認證過程。

要開始使用，請在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫 `Auth::viaRequest` 方法。`viaRequest` 方法接受一個認證驅動名稱作為其第一個引數。此名稱可以是描述您自訂 Guard 的任何字串。傳遞給方法的第二個引數應該是一個 Closure，它接收傳入的 HTTP 請求並返回一個使用者實例，或者如果認證失敗，則返回 `null`：

    use App\Models\User;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Auth;

    /**
     * 啟動任何應用程式服務。
     */
    public function boot(): void
    {
        Auth::viaRequest('custom-token', function (Request $request) {
            return User::where('token', (string) $request->token)->first();
        });
    }

一旦定義了自訂認證驅動，您就可以在 `auth.php` 設定檔的 `guards` 設定中將其設定為驅動：

    'guards' => [
        'api' => [
            'driver' => 'custom-token',
        ],
    ],

最後，您可以在將認證 Middleware 分配給路由時引用 Guard：

    Route::middleware('auth:api')->group(function () {
        // ...
    });

<a name="adding-custom-user-providers"></a>
## 新增自訂使用者 Provider

如果您不使用傳統的關聯式資料庫來儲存使用者，您將需要使用自己的認證使用者 Provider 擴展 Laravel。我們將使用 `Auth` Facade 上的 `provider` 方法來定義自訂使用者 Provider。使用者 Provider 解析器應返回 `Illuminate\Contracts\Auth\UserProvider` 的實作：

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
         * 啟動任何應用程式服務。
         */
        public function boot(): void
        {
            Auth::provider('mongo', function (Application $app, array $config) {
                // 返回 Illuminate\Contracts\Auth\UserProvider 的實例...

                return new MongoUserProvider($app->make('mongo.connection'));
            });
        }
    }

使用 `provider` 方法註冊 Provider 後，您可以在 `auth.php` 設定檔中切換到新的使用者 Provider。首先，定義一個使用新驅動的 `provider`：

    'providers' => [
        'users' => [
            'driver' => 'mongo',
        ],
    ],

最後，您可以在 `guards` 設定中引用此 Provider：

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
    ],

<a name="the-user-provider-contract"></a>
### User Provider 契約

`Illuminate\Contracts\Auth\UserProvider` 實作負責從持久儲存系統 (例如 MySQL、MongoDB 等) 中取得 `Illuminate\Contracts\Auth\Authenticatable` 實作。這兩個介面允許 Laravel 認證機制繼續運作，無論使用者資料如何儲存或用於表示已認證使用者的類別型別：

讓我們看看 `Illuminate\Contracts\Auth\UserProvider` 契約：

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

`retrieveById` 函數通常接收一個代表使用者的鍵，例如來自 MySQL 資料庫的自動遞增 ID。應取得與 ID 匹配的 `Authenticatable` 實作並由該方法返回。

`retrieveByToken` 函數透過其唯一的 `$identifier` 和「記住我」`$token` 取得使用者，通常儲存在資料庫欄位中，例如 `remember_token`。與前一個方法一樣，應由該方法返回具有匹配 Token 值的 `Authenticatable` 實作。

`updateRememberToken` 方法使用新的 `$token` 更新 `$user` 實例的 `remember_token`。在成功的「記住我」認證嘗試或使用者登出時，會為使用者分配一個新的 Token。

`retrieveByCredentials` 方法接收傳遞給 `Auth::attempt` 方法的憑證陣列，當嘗試使用應用程式進行認證時。然後，該方法應「查詢」底層持久儲存以尋找與這些憑證匹配的使用者。通常，此方法將執行一個帶有「where」條件的查詢，該條件搜尋具有與 `$credentials['username']` 值匹配的「使用者名稱」的使用者記錄。該方法應返回 `Authenticatable` 的實作。**此方法不應嘗試進行任何密碼驗證或認證。**

`validateCredentials` 方法應將給定的 `$user` 與 `$credentials` 進行比較以認證使用者。例如，此方法通常會使用 `Hash::check` 方法將 `$user->getAuthPassword()` 的值與 `$credentials['password']` 的值進行比較。此方法應返回 `true` 或 `false`，指示密碼是否有效。

`rehashPasswordIfRequired` 方法應在需要和支援的情況下重新雜湊給定 `$user` 的密碼。例如，此方法通常會使用 `Hash::needsRehash` 方法來判斷 `$credentials['password']` 值是否需要重新雜湊。如果密碼需要重新雜湊，該方法應使用 `Hash::make` 方法重新雜湊密碼並更新底層持久儲存中的使用者記錄。

<a name="the-authenticatable-contract"></a>
### Authenticatable 契約

現在我們已經探討了 `UserProvider` 上的每個方法，讓我們看看 `Authenticatable` 契約。請記住，使用者 Provider 應從 `retrieveById`、`retrieveByToken` 和 `retrieveByCredentials` 方法返回此介面的實作：

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

此介面很簡單。`getAuthIdentifierName` 方法應返回使用者「主鍵」欄位的名稱，`getAuthIdentifier` 方法應返回使用者的「主鍵」。當使用 MySQL 後端時，這很可能是分配給使用者記錄的自動遞增主鍵。`getAuthPasswordName` 方法應返回使用者密碼欄位的名稱。`getAuthPassword` 方法應返回使用者的雜湊密碼。

此介面允許認證系統與任何「使用者」類別一起使用，無論您使用何種 ORM 或儲存抽象層。預設情況下，Laravel 在 `app/Models` 目錄中包含一個 `App\Models\User` 類別，它實作此介面。

<a name="automatic-password-rehashing"></a>
## 自動密碼重新雜湊

Laravel 的預設密碼雜湊演算法是 bcrypt。bcrypt 雜湊的「工作因子」可以透過應用程式的 `config/hashing.php` 設定檔或 `BCRYPT_ROUNDS` 環境變數進行調整。

通常，隨著 CPU/GPU 處理能力的增加，bcrypt 工作因子應隨時間增加。如果您增加應用程式的 bcrypt 工作因子，Laravel 將在使用者透過 Laravel 的入門套件認證應用程式時，或當您透過 `attempt` 方法 [手動認證使用者](#authenticating-users) 時，優雅且自動地重新雜湊使用者密碼。

通常，自動密碼重新雜湊不應中斷您的應用程式；但是，您可以透過發布 `hashing` 設定檔來禁用此行為：

```shell
php artisan config:publish hashing
```

一旦設定檔發布，您可以將 `rehash_on_login` 設定值設定為 `false`：

```php
'rehash_on_login' => false,
```

<a name="events"></a>
## 事件

Laravel 在認證過程中分派各種 [事件](/docs/{{version}}/events)。您可以為以下任何事件 [定義監聽器](/docs/{{version}}/events)：

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
