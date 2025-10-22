# 入門套件

- [介紹](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用的入門套件](#available-starter-kits)
    - [React](#react)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [入門套件客製化](#starter-kit-customization)
    - [React](#react-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [雙重驗證](#two-factor-authentication)
- [WorkOS AuthKit 驗證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 介紹

為了讓您能更快地開始建構新的 Laravel 應用程式，我們很高興能提供[應用程式入門套件](https://laravel.com/starter-kits)。這些入門套件能讓您搶先建構下一個 Laravel 應用程式，並包含註冊與驗證應用程式使用者所需的路由、控制器與視圖。這些入門套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 提供驗證功能。

雖然我們鼓勵您使用這些入門套件，但它們並非強制。您可以透過簡單地安裝全新版本的 Laravel，從零開始建構自己的應用程式。無論如何，我們相信您都將建構出色的成果！

<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

要使用我們的其中一個入門套件建立新的 Laravel 應用程式，您應該首先[安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝程式 CLI 工具：

```shell
composer global require laravel/installer
```

然後，使用 Laravel 安裝程式 CLI 建立新的 Laravel 應用程式。Laravel 安裝程式會提示您選擇偏好的入門套件：

```shell
laravel new my-app
```

建立您的 Laravel 應用程式後，您只需透過 NPM 安裝其前端依賴項並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

一旦您啟動了 Laravel 開發伺服器，您的應用程式將可在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。

<a name="available-starter-kits"></a>
## 可用的入門套件

<a name="react"></a>
### React

我們的 React 入門套件為使用 [Inertia](https://inertiajs.com) 搭配 React 前端建構 Laravel 應用程式提供了一個強大且現代的起點。

Inertia 允許您使用經典的伺服器端路由和控制器建構現代的單頁 React 應用程式。這讓您能同時享有 React 的前端強大功能、Laravel 卓越的後端生產力以及閃電般的 Vite 編譯速度。

React 入門套件採用 React 19、TypeScript、Tailwind 和 [shadcn/ui](https://ui.shadcn.com) 元件庫。

<a name="vue"></a>
### Vue

我們的 Vue 入門套件為使用 [Inertia](https://inertiajs.com) 搭配 Vue 前端建構 Laravel 應用程式提供了一個絕佳的起點。

Inertia 允許您使用經典的伺服器端路由和控制器建構現代的單頁 Vue 應用程式。這讓您能同時享有 Vue 的前端強大功能、Laravel 卓越的後端生產力以及閃電般的 Vite 編譯速度。

Vue 入門套件採用 Vue Composition API、TypeScript、Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。

<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件為使用 [Laravel Livewire](https://livewire.laravel.com) 前端建構 Laravel 應用程式提供了一個完美的起點。

Livewire 是一種強大的方式，僅使用 PHP 即可建構動態、響應式的前端 UI。它非常適合主要使用 Blade 模板的團隊，並正在尋找比 React 和 Vue 等 JavaScript 驅動的 SPA 框架更簡單的替代方案。

Livewire 入門套件採用 Livewire、Tailwind 和 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 入門套件客製化


<a name="react-customization"></a>
### React

我們的 React 入門套件是使用 Inertia 2, React 19, Tailwind 4 和 [shadcn/ui](https://ui.shadcn.com) 建構的。與我們所有的入門套件一樣，所有的後端和前端程式碼都存在於您的應用程式中，以便完全客製化。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼來客製化應用程式的外觀和行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn 元件，請先[找到您想要發佈的元件](https://ui.shadcn.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該指令會將 Switch 元件發佈到 `resources/js/components/ui/switch.tsx`。一旦元件發佈成功，您就可以在任何頁面中使用它：

```jsx
import { Switch } from "@/components/ui/switch"

const MyPage = () => {
  return (
    <div>
      <Switch />
    </div>
  );
};

export default MyPage;
```


<a name="react-available-layouts"></a>
#### 可用版面配置

React 入門套件包含兩種不同的主要版面配置供您選擇：「側邊欄」版面配置和「頁首」版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部匯入的版面配置來切換到頁首版面配置：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「浮動 (floating)」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 驗證頁面版面配置變體

React 入門套件中包含的驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的版面配置變體：「簡約 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的驗證版面配置，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部匯入的版面配置：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件是使用 Inertia 2, Vue 3 Composition API, Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 建構的。與我們所有的入門套件一樣，所有的後端和前端程式碼都存在於您的應用程式中，以便完全客製化。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼來客製化應用程式的外觀和行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn-vue 元件，請先[找到您想要發佈的元件](https://www.shadcn-vue.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn-vue@latest add switch
```

在此範例中，該指令會將 Switch 元件發佈到 `resources/js/components/ui/Switch.vue`。一旦元件發佈成功，您就可以在任何頁面中使用它：

```vue
<script setup lang="ts">
import { Switch } from '@/Components/ui/switch'
</script>

<template>
    <div>
        <Switch />
    </div>
</template>
```


<a name="vue-available-layouts"></a>
#### 可用版面配置

Vue 入門套件包含兩種不同的主要版面配置供您選擇：「側邊欄」版面配置和「頁首」版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂部匯入的版面配置來切換到頁首版面配置：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「浮動 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 驗證頁面版面配置變體

Vue 入門套件中包含的驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的版面配置變體：「簡約 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的驗證版面配置，請修改應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂部匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件是使用 Livewire 3, Tailwind 和 [Flux UI](https://fluxui.dev/) 建構的。與我們所有的入門套件一樣，所有的後端和前端程式碼都存在於您的應用程式中，以便完全客製化。


#### Livewire 和 Volt

大部分前端程式碼位於 `resources/views` 目錄中。您可以自由修改任何程式碼來客製化應用程式的外觀和行為：

```text
resources/views
├── components            # Reusable Livewire components
├── flux                  # Customized Flux components
├── livewire              # Livewire pages
├── partials              # Reusable Blade partials
├── dashboard.blade.php   # Authenticated user dashboard
├── welcome.blade.php     # Guest user welcome page
```


#### 傳統 Livewire 元件

前端程式碼位於 `resources/views` 目錄中，而 `app/Livewire` 目錄則包含 Livewire 元件對應的後端邏輯。


<a name="livewire-available-layouts"></a>
#### 可用版面配置

Livewire 入門套件包含兩種不同的主要版面配置供您選擇：「側邊欄」版面配置和「頁首」版面配置。側邊欄版面配置是預設值，但您可以透過修改應用程式 `resources/views/components/layouts/app.blade.php` 檔案所使用的版面配置來切換到頁首版面配置。此外，您應該為主 Flux 元件添加 `container` 屬性：

```blade
<x-layouts.app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts.app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 驗證頁面版面配置變體

Livewire 入門套件中包含的驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的版面配置變體：「簡約 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的驗證版面配置，請修改應用程式 `resources/views/components/layouts/auth.blade.php` 檔案所使用的版面配置：

```blade
<x-layouts.auth.split>
    {{ $slot }}
</x-layouts.auth.split>
```

<a name="two-factor-authentication"></a>
## 雙重驗證

所有入門套件都內建由 [Laravel Fortify](/docs/{{version}}/fortify#two-factor-authentication) 驅動的雙重驗證 (2FA)，為使用者帳戶增添一層額外的安全性。使用者可以使用任何支援基於時間的一次性密碼 (TOTP) 的驗證器應用程式來保護其帳戶。

雙重驗證預設啟用，並支援 [Fortify](/docs/{{version}}/fortify#two-factor-authentication) 提供的所有選項：

```php
Features::twoFactorAuthentication([
    'confirm' => true,
    'confirmPassword' => true,
]);
```

<a name="workos"></a>
## WorkOS AuthKit 驗證

預設情況下，React、Vue 和 Livewire 入門套件都利用 Laravel 內建的驗證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們還提供每個入門套件的 [WorkOS AuthKit](https://authkit.com) 驅動變體，該變體提供：

<div class="content-list" markdown="1">

- 社交驗證 (Google、Microsoft、GitHub 和 Apple)
- 通行金鑰驗證
- 基於電子郵件的「魔法驗證」(Magic Auth)
- SSO (單一登入)

</div>

使用 WorkOS 作為您的驗證提供者[需要一個 WorkOS 帳戶](https://workos.com)。WorkOS 為每月活躍用戶數達 100 萬的應用程式提供免費驗證。

要將 WorkOS AuthKit 用作您應用程式的驗證提供者，請透過 `laravel new` 建立新的入門套件驅動應用程式時，選擇 WorkOS 選項。

### 配置您的 WorkOS 入門套件

在使用 WorkOS 驅動的入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應與 WorkOS 儀表板中為您的應用程式提供的值相符：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 儀表板中配置應用程式首頁 URL。這是使用者登出應用程式後將被重新導向的 URL。

<a name="configuring-authkit-authentication-methods"></a>
#### 配置 AuthKit 驗證方法

當使用 WorkOS 驅動的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 配置設定中停用「電子郵件 + 密碼」驗證，讓使用者只能透過社交驗證提供者、通行金鑰、「魔法驗證」(Magic Auth) 和 SSO 進行驗證。這讓您的應用程式可以完全避免處理使用者密碼。

<a name="configuring-authkit-session-timeouts"></a>
#### 配置 AuthKit 會話逾時

此外，我們建議您配置 WorkOS AuthKit 會話不活動逾時，使其與您的 Laravel 應用程式配置的會話逾時閾值相符，該閾值通常為兩小時。

<a name="inertia-ssr"></a>
### Inertia SSR

React 和 Vue 入門套件與 Inertia 的[伺服器端渲染](https://inertiajs.com/server-side-rendering)功能相容。要為您的應用程式建構一個 Inertia SSR 相容的套件，請執行 `build:ssr` 命令：

```shell
npm run build:ssr
```

為了方便起見，還提供了 `composer dev:ssr` 命令。此命令會在為您的應用程式建構 SSR 相容套件後，啟動 Laravel 開發伺服器和 Inertia SSR 伺服器，讓您可以使用 Inertia 的伺服器端渲染引擎在本地測試您的應用程式：

```shell
composer dev:ssr
```

<a name="community-maintained-starter-kits"></a>
### 社群維護的入門套件

使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上任何社群維護的入門套件提供給 `--using` 旗標：

```shell
laravel new my-app --using=example/starter-kit
```

<a name="creating-starter-kits"></a>
#### 建立入門套件

為了確保您的入門套件可供他人使用，您需要將其發佈到 [Packagist](https://packagist.org)。您的入門套件應該在其 `.env.example` 檔案中定義所需的環境變數，並且任何必要的安裝後命令都應列在入門套件 `composer.json` 檔案的 `post-create-project-cmd` 陣列中。

<a name="faqs"></a>
### 常見問題

<a name="faq-upgrade"></a>
#### 如何升級？

每個入門套件都為您的下一個應用程式提供了一個堅實的起點。憑藉對程式碼的完全所有權，您可以完全按照您的設想調整、客製化和建構您的應用程式。然而，沒有必要更新入門套件本身。

<a name="faq-enable-email-verification"></a>
#### 如何啟用電子郵件驗證？

可以透過取消註解您的 `App/Models/User.php` 模型中的 `MustVerifyEmail` 匯入，並確保該模型實現 `MustVerifyEmail` 介面來添加電子郵件驗證：

```php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
// ...

class User extends Authenticatable implements MustVerifyEmail
{
    // ...
}
```

註冊後，使用者將收到一封驗證電子郵件。若要限制在使用者電子郵件地址驗證之前對某些路由的存取，請將 `verified` 中介層添加到這些路由中：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 當使用入門套件的 [WorkOS](#workos) 變體時，電子郵件驗證不是必需的。

<a name="faq-modify-email-template"></a>
#### 如何修改預設電子郵件範本？

您可能想要自訂預設電子郵件範本，使其更好地與您的應用程式品牌保持一致。若要修改此範本，您應該使用以下命令將電子郵件視圖發佈到您的應用程式中：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中生成多個檔案。您可以修改這些檔案以及 `resources/views/vendor/mail/themes/default.css` 檔案來更改預設電子郵件範本的外觀和樣式。