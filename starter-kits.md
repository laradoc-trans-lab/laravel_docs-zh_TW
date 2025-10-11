# 入門套件

- [簡介](#introduction)
- [Laravel Breeze](#laravel-breeze)
    - [安裝](#laravel-breeze-installation)
    - [Breeze 與 Blade](#breeze-and-blade)
    - [Breeze 與 Livewire](#breeze-and-livewire)
    - [Breeze 與 React / Vue](#breeze-and-inertia)
    - [Breeze 與 Next.js / API](#breeze-and-next)
- [Laravel Jetstream](#laravel-jetstream)

<a name="introduction"></a>
## 簡介

為了讓您在建立新的 Laravel 應用程式時能有個好的開始，我們很高興能提供身分驗證及應用程式入門套件。這些套件會自動為您的應用程式建構好路由 (routes)、控制器 (controllers) 及視圖 (views)，您會需要這些來註冊 (register) 及驗證 (authenticate) 應用程式的使用者。

雖然歡迎您使用這些入門套件，但它們並非必要。您可以透過安裝一份全新的 Laravel，從零開始建構自己的應用程式。無論哪種方式，我們相信您都會創造出很棒的應用程式！

<a name="laravel-breeze"></a>
## Laravel Breeze

[Laravel Breeze](https://github.com/laravel/breeze) 是一個簡約且簡單的實作，包含了 Laravel 所有[身分驗證功能](/docs/{{version}}/authentication)，像是登入 (login)、註冊 (registration)、密碼重設 (password reset)、電子郵件驗證 (email verification) 及密碼確認 (password confirmation)。此外，Breeze 也包含一個簡單的「個人資料 (profile)」頁面，讓使用者可以更新姓名、電子郵件位址及密碼。

Laravel Breeze 預設的視圖層 (view layer) 由簡單的 [Blade 模板](/docs/{{version}}/blade)組成，並使用 [Tailwind CSS](https://tailwindcss.com) 進行樣式設計。此外，Breeze 也提供基於 [Livewire](https://livewire.laravel.com) 或 [Inertia](https://inertiajs.com) 的 Scaffold 選項，Inertia 的 Scaffold 可以選擇使用 Vue 或 React。

<img src="https://laravel.com/img/docs/breeze-register.png">

#### Laravel Bootcamp

如果您是 Laravel 新手，歡迎跳到 [Laravel Bootcamp](https://bootcamp.laravel.com)。Laravel Bootcamp 將會引導您使用 Breeze 建立您的第一個 Laravel 應用程式。這是一個全面了解 Laravel 與 Breeze 所能提供的一切的好方法。

<a name="laravel-breeze-installation"></a>
### 安裝

首先，您應該[建立一個新的 Laravel 應用程式](/docs/{{version}}/installation)。如果您使用 [Laravel 安裝程式](/docs/{{version}}/installation#creating-a-laravel-project)建立應用程式，在安裝過程中會提示您安裝 Laravel Breeze。否則，您將需要遵循下面的手動安裝說明。

如果您已經建立了一個新的 Laravel 應用程式但沒有入門套件，您可以使用 Composer 手動安裝 Laravel Breeze：

```shell
composer require laravel/breeze --dev
```

在 Composer 安裝好 Laravel Breeze 套件後，您應該執行 `breeze:install` Artisan 指令。此指令會將身分驗證的視圖 (views)、路由 (routes)、控制器 (controllers) 及其他資源發佈到您的應用程式。Laravel Breeze 會將所有程式碼發佈到您的應用程式，以便您對其功能及實作擁有完整的控制權與可見性。

`breeze:install` 指令將會提示您選擇偏好的前端技術棧 (frontend stack) 及測試框架 (testing framework)：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

<a name="breeze-and-blade"></a>
### Breeze 與 Blade

Breeze 預設的「技術棧 (stack)」是 Blade 技術棧，它利用簡單的 [Blade 模板](/docs/{{version}}/blade)來渲染您的應用程式前端。Blade 技術棧可以透過不帶任何額外引數執行 `breeze:install` 指令，並選擇 Blade 前端技術棧來安裝。Breeze 的 Scaffold 安裝完成後，您也應該編譯應用程式的前端資產 (frontend assets)：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

接著，您可以在網頁瀏覽器中導覽至應用程式的 `/login` 或 `/register` URL。Breeze 的所有路由 (routes) 都定義在 `routes/auth.php` 檔案中。

> [!NOTE]  
> 若要了解更多關於編譯應用程式的 CSS 與 JavaScript 的資訊，請查閱 Laravel 的 [Vite 文件](/docs/{{version}}/vite#running-vite)。

<a name="breeze-and-livewire"></a>
### Breeze 與 Livewire

Laravel Breeze 也提供 [Livewire](https://livewire.laravel.com) 的 Scaffold。Livewire 是一種強大的方式，可以僅使用 PHP 來建構動態、響應式的前端 UI。

Livewire 非常適合主要使用 Blade 模板，並正在尋找比 JavaScript 驅動的 SPA 框架 (如 Vue 與 React) 更簡單替代方案的團隊。

若要使用 Livewire 技術棧，您可以在執行 `breeze:install` Artisan 指令時選擇 Livewire 前端技術棧。Breeze 的 Scaffold 安裝完成後，您應該執行資料庫遷移 (database migrations)：

```shell
php artisan breeze:install

php artisan migrate
```

<a name="breeze-and-inertia"></a>
### Breeze 與 React / Vue

Laravel Breeze 也透過 [Inertia](https://inertiajs.com) 前端實作，提供了 React 與 Vue 的 Scaffold。Inertia 讓您可以使用經典的伺服器端路由 (server-side routing) 與控制器 (controllers) 來建構現代化的單頁 (single-page) React 與 Vue 應用程式。

Inertia 讓您可以同時享受 React 與 Vue 的前端強大功能，以及 Laravel 驚人的後端生產力與閃電般的 [Vite](https://vitejs.dev) 編譯速度。若要使用 Inertia 技術棧，您可以在執行 `breeze:install` Artisan 指令時選擇 Vue 或 React 前端技術棧。

當選擇 Vue 或 React 前端技術棧時，Breeze 安裝程式也會提示您是否需要 [Inertia SSR](https://inertiajs.com/server-side-rendering) 或 TypeScript 支援。Breeze 的 Scaffold 安裝完成後，您也應該編譯應用程式的前端資產 (frontend assets)：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

接著，您可以在網頁瀏覽器中導覽至應用程式的 `/login` 或 `/register` URL。Breeze 的所有路由 (routes) 都定義在 `routes/auth.php` 檔案中。

<a name="breeze-and-next"></a>
### Breeze 與 Next.js / API

Laravel Breeze 也可以為身分驗證 API 建立 Scaffold，此 API 能夠驗證現代 JavaScript 應用程式，例如由 [Next](https://nextjs.org)、[Nuxt](https://nuxt.com) 及其他框架驅動的應用程式。若要開始，請在執行 `breeze:install` Artisan 指令時選擇 API 技術棧作為您想要的技術棧：

```shell
php artisan breeze:install

php artisan migrate
```

在安裝過程中，Breeze 會將 `FRONTEND_URL` 環境變數新增到應用程式的 `.env` 檔案中。這個 URL 應該是您的 JavaScript 應用程式的 URL。在本地開發期間，這通常會是 `http://localhost:3000`。此外，您應該確保您的 `APP_URL` 設定為 `http://localhost:8000`，這是 `serve` Artisan 指令所使用的預設 URL。

<a name="next-reference-implementation"></a>
#### Next.js 參考實作

最後，您已準備好將此後端與您選擇的前端配對。Breeze 前端的一個 Next 參考實作[可在 GitHub 上取得](https://github.com/laravel/breeze-next)。此前端由 Laravel 維護，並包含與 Breeze 提供的傳統 Blade 和 Inertia 技術棧相同的用戶介面。

<a name="laravel-jetstream"></a>
## Laravel Jetstream

雖然 Laravel Breeze 為建立 Laravel 應用程式提供了一個簡單而最小的起點，但 Jetstream 透過更強大的功能和額外的前端技術棧來增強該功能。**對於 Laravel 新手，我們建議先學習 Laravel Breeze 的基礎知識，然後再進階到 Laravel Jetstream。**

Jetstream 為 Laravel 提供了一個設計精美的應用程式 Scaffold，並包含登入 (login)、註冊 (registration)、電子郵件驗證 (email verification)、雙因子身分驗證 (two-factor authentication)、Session 管理、透過 Laravel Sanctum 提供的 API 支援以及可選的團隊管理功能。Jetstream 是使用 [Tailwind CSS](https://tailwindcss.com) 設計的，並提供您選擇 [Livewire](https://livewire.laravel.com) 或 [Inertia](https://inertiajs.com) 驅動的前端 Scaffold。

有關安裝 Laravel Jetstream 的完整文件，請參閱[官方 Jetstream 文件](https://jetstream.laravel.com)。