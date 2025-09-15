# Starter Kits

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

為了讓您在建構新的 Laravel 應用程式時能有個好的開始，我們很高興能提供身份驗證與應用程式的 Starter Kits。這些 Kits 會自動為您的應用程式建立所需的路由、Controller 和 View，以便註冊與驗證應用程式的使用者。

雖然我們歡迎您使用這些 Starter Kits，但它們並非強制。您可以透過簡單地安裝全新的 Laravel 應用程式，從頭開始建構自己的應用程式。無論哪種方式，我們相信您都能建構出色的作品！

<a name="laravel-breeze"></a>
## Laravel Breeze

[Laravel Breeze](https://github.com/laravel/breeze) 是 Laravel 所有[身份驗證功能](/docs/{{version}}/authentication)的最小、簡單實作，包括登入、註冊、密碼重設、電子郵件驗證和密碼確認。此外，Breeze 還包含一個簡單的「個人資料」頁面，使用者可以在其中更新其姓名、電子郵件地址和密碼。

Laravel Breeze 的預設 View 層由簡單的 [Blade templates](/docs/{{version}}/blade) 組成，並使用 [Tailwind CSS](https://tailwindcss.com) 進行樣式設定。此外，Breeze 還提供基於 [Livewire](https://livewire.laravel.com) 或 [Inertia](https://inertiajs.com) 的 Scaffold 選項，其中基於 Inertia 的 Scaffold 可選擇使用 Vue 或 React。

<img src="https://laravel.com/img/docs/breeze-register.png">

#### Laravel Bootcamp

如果您是 Laravel 的新手，歡迎跳到 [Laravel Bootcamp](https://bootcamp.laravel.com)。Laravel Bootcamp 將引導您使用 Breeze 建構您的第一個 Laravel 應用程式。這是了解 Laravel 和 Breeze 所提供的一切的絕佳方式。

<a name="laravel-breeze-installation"></a>
### 安裝

首先，您應該[建立一個新的 Laravel 應用程式](/docs/{{version}}/installation)。如果您使用 [Laravel installer](/docs/{{version}}/installation#creating-a-laravel-project) 建立應用程式，系統將在安裝過程中提示您安裝 Laravel Breeze。否則，您需要遵循下面的手動安裝說明。

如果您已經建立了一個新的 Laravel 應用程式但沒有 Starter Kit，您可以手動使用 Composer 安裝 Laravel Breeze：

```shell
composer require laravel/breeze --dev
```

在 Composer 安裝完 Laravel Breeze 套件後，您應該執行 `breeze:install` Artisan command。此命令會將身份驗證的 View、路由、Controller 和其他資源發佈到您的應用程式。Laravel Breeze 會將其所有程式碼發佈到您的應用程式中，以便您完全控制和了解其功能和實作。

`breeze:install` command 將提示您選擇偏好的前端 Stack 和測試框架：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

<a name="breeze-and-blade"></a>
### Breeze 與 Blade

預設的 Breeze 「Stack」是 Blade Stack，它利用簡單的 [Blade templates](/docs/{{version}}/blade) 來渲染您應用程式的前端。Blade Stack 可以透過在執行 `breeze:install` command 時不帶任何額外參數並選擇 Blade 前端 Stack 來安裝。安裝 Breeze 的 Scaffold 後，您還應該編譯應用程式的前端資產：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

接下來，您可以在網頁瀏覽器中導航到應用程式的 `/login` 或 `/register` URL。所有 Breeze 的路由都定義在 `routes/auth.php` 檔案中。

> [!NOTE]  
> 要了解更多關於編譯應用程式的 CSS 和 JavaScript，請查閱 Laravel 的 [Vite 說明文件](/docs/{{version}}/vite#running-vite)。

<a name="breeze-and-livewire"></a>
### Breeze 與 Livewire

Laravel Breeze 也提供 [Livewire](https://livewire.laravel.com) Scaffold。Livewire 是一種僅使用 PHP 建構動態、響應式前端 UI 的強大方式。

Livewire 非常適合主要使用 Blade templates 並正在尋找 Vue 和 React 等 JavaScript 驅動的 SPA 框架的更簡單替代方案的團隊。

要使用 Livewire Stack，您可以在執行 `breeze:install` Artisan command 時選擇 Livewire 前端 Stack。安裝 Breeze 的 Scaffold 後，您應該執行資料庫遷移：

```shell
php artisan breeze:install

php artisan migrate
```

<a name="breeze-and-inertia"></a>
### Breeze 與 React / Vue

Laravel Breeze 也透過 [Inertia](https://inertiajs.com) 前端實作提供 React 和 Vue Scaffold。Inertia 允許您使用經典的伺服器端路由和 Controller 建構現代的單頁 React 和 Vue 應用程式。

Inertia 讓您享受 React 和 Vue 的前端強大功能，結合 Laravel 令人難以置信的後端生產力以及閃電般的 [Vite](https://vitejs.dev) 編譯速度。要使用 Inertia Stack，您可以在執行 `breeze:install` Artisan command 時選擇 Vue 或 React 前端 Stack。

當選擇 Vue 或 React 前端 Stack 時，Breeze 安裝程式還會提示您決定是否需要 [Inertia SSR](https://inertiajs.com/server-side-rendering) 或 TypeScript 支援。安裝 Breeze 的 Scaffold 後，您還應該編譯應用程式的前端資產：

```shell
php artisan breeze:install

php artisan migrate
npm install
npm run dev
```

接下來，您可以在網頁瀏覽器中導航到應用程式的 `/login` 或 `/register` URL。所有 Breeze 的路由都定義在 `routes/auth.php` 檔案中。

<a name="breeze-and-next"></a>
### Breeze 與 Next.js / API

Laravel Breeze 還可以 Scaffold 一個身份驗證 API，該 API 準備好驗證由 [Next](https://nextjs.org)、[Nuxt](https://nuxt.com) 等現代 JavaScript 應用程式提供支援的應用程式。要開始使用，請在執行 `breeze:install` Artisan command 時選擇 API Stack 作為您所需的 Stack：

```shell
php artisan breeze:install

php artisan migrate
```

在安裝過程中，Breeze 會在您應用程式的 `.env` 檔案中新增一個 `FRONTEND_URL` 環境變數。此 URL 應該是您的 JavaScript 應用程式的 URL。在本地開發期間，這通常是 `http://localhost:3000`。此外，您應該確保您的 `APP_URL` 設定為 `http://localhost:8000`，這是 `serve` Artisan command 使用的預設 URL。

<a name="next-reference-implementation"></a>
#### Next.js 參考實作

最後，您已準備好將此後端與您選擇的前端配對。Breeze 前端的 Next 參考實作[可在 GitHub 上取得](https://github.com/laravel/breeze-next)。此前端由 Laravel 維護，並包含與 Breeze 提供的傳統 Blade 和 Inertia Stack 相同的使用者介面。

<a name="laravel-jetstream"></a>
## Laravel Jetstream

雖然 Laravel Breeze 為建構 Laravel 應用程式提供了一個簡單且最小的起點，但 Jetstream 透過更強大的功能和額外的前端技術 Stack 增強了該功能。**對於剛接觸 Laravel 的人，我們建議在升級到 Laravel Jetstream 之前，先學習 Laravel Breeze 的基礎知識。**

Jetstream 為 Laravel 提供了一個設計精美的應用程式 Scaffold，並包含登入、註冊、電子郵件驗證、雙因素身份驗證、Session 管理、透過 Laravel Sanctum 提供的 API 支援以及可選的團隊管理。Jetstream 使用 [Tailwind CSS](https://tailwindcss.com) 設計，並提供您選擇的 [Livewire](https://livewire.laravel.com) 或 [Inertia](https://inertiajs.com) 驅動的前端 Scaffold。

有關安裝 Laravel Jetstream 的完整說明文件可在[官方 Jetstream 說明文件](https://jetstream.laravel.com)中找到。
