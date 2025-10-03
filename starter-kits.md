# 入門套件

- [簡介](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用入門套件](#available-starter-kits)
    - [React](#react)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [入門套件自訂](#starter-kit-customization)
    - [React](#react-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [雙因素驗證](#two-factor-authentication)
- [WorkOS AuthKit 驗證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為協助您更快地開始建構新的 Laravel 應用程式，我們很高興能提供[應用程式入門套件](https://laravel.com/starter-kits)。這些入門套件能讓您快速啟動下一個 Laravel 應用程式的建構，其中包含您需要用於註冊和驗證應用程式使用者的路由、控制器與視圖。

儘管我們歡迎您使用這些入門套件，但它們並非必需。您可以透過簡單地安裝全新 Laravel 副本，從頭開始建構自己的應用程式。無論哪種方式，我們都相信您將會建構出色的成果！

<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

要使用我們的入門套件之一建立新的 Laravel 應用程式，您應該首先[安裝 PHP 和 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 和 Composer，您可以透過 Composer 安裝 Laravel installer CLI 工具：

```shell
composer global require laravel/installer
```

然後，使用 Laravel installer CLI 建立新的 Laravel 應用程式。Laravel installer 將提示您選擇偏好的入門套件：

```shell
laravel new my-app
```

建立 Laravel 應用程式後，您只需透過 NPM 安裝其前端依賴項並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

一旦您啟動了 Laravel 開發伺服器，您的應用程式將可在您的網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 存取。

<a name="available-starter-kits"></a>
## 可用入門套件

<a name="react"></a>
### React

我們的 React 入門套件為使用 [Inertia](https://inertiajs.com) 搭配 React 前端建構 Laravel 應用程式，提供了一個穩健、現代的起點。

Inertia 讓您可以使用經典的伺服器端路由和控制器來建構現代的單頁面 React 應用程式。這讓您能享受 React 的前端能力，並結合 Laravel 令人驚嘆的後端生產力與閃電般的 Vite 編譯速度。

此 React 入門套件利用 React 19、TypeScript、Tailwind 和 [shadcn/ui](https://ui.shadcn.com) 元件庫。

<a name="vue"></a>
### Vue

我們的 Vue 入門套件為使用 [Inertia](https://inertiajs.com) 搭配 Vue 前端建構 Laravel 應用程式，提供了一個絕佳的起點。

Inertia 讓您可以使用經典的伺服器端路由和控制器來建構現代的單頁面 Vue 應用程式。這讓您能享受 Vue 的前端能力，並結合 Laravel 令人驚嘆的後端生產力與閃電般的 Vite 編譯速度。

此 Vue 入門套件利用 Vue Composition API、TypeScript、Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。

<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件為使用 [Laravel Livewire](https://livewire.laravel.com) 前端建構 Laravel 應用程式，提供了一個完美的起點。

Livewire 是一種強大的方式，僅使用 PHP 即可建構動態、響應式的前端 UI。對於主要使用 Blade 模板並尋求 JavaScript 驅動的 SPA 框架（如 React 和 Vue）的更簡單替代方案的團隊來說，Livewire 是個絕佳的選擇。

此 Livewire 入門套件利用 Livewire、Tailwind 和 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 入門套件自訂


<a name="react-customization"></a>
### React

我們的 React 入門套件基於 Inertia 2、React 19、Tailwind 4 和 [shadcn/ui](https://ui.shadcn.com) 建構。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完整的自訂。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn 元件，請先[找到您要發佈的元件](https://ui.shadcn.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn@latest add switch
```

在這個範例中，該指令會將 Switch 元件發佈到 `resources/js/components/ui/switch.tsx`。一旦元件發佈成功，您就可以在任何頁面中使用它：

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
#### 可用佈局

React 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部匯入的佈局來切換到標頭佈局：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「浮動 (floating)」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 驗證頁面佈局變體

React 入門套件中包含的驗證頁面，例如登入頁面與註冊頁面，也提供三種不同的佈局變體：「簡約 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的驗證佈局，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部匯入的佈局：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件基於 Inertia 2、Vue 3 Composition API、Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 建構。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完整的自訂。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發佈額外的 shadcn-vue 元件，請先[找到您要發佈的元件](https://www.shadcn-vue.com)。然後，使用 `npx` 發佈該元件：

```shell
npx shadcn-vue@latest add switch
```

在這個範例中，該指令會將 Switch 元件發佈到 `resources/js/components/ui/Switch.vue`。一旦元件發佈成功，您就可以在任何頁面中使用它：

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
#### 可用佈局

Vue 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂部匯入的佈局來切換到標頭佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```


<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌 (inset)」變體和「浮動 (floating)」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="vue-authentication-page-layout-variants"></a>
#### 驗證頁面佈局變體

Vue 入門套件中包含的驗證頁面，例如登入頁面與註冊頁面，也提供三種不同的佈局變體：「簡約 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的驗證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂部匯入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```


<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件基於 Livewire 3、Tailwind 和 [Flux UI](https://fluxui.dev/) 建構。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於您的應用程式中，以實現完整的自訂。


#### Livewire 和 Volt

大部分前端程式碼位於 `resources/views` 目錄中。您可以自由修改任何程式碼，以自訂應用程式的外觀與行為：

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

前端程式碼位於 `resources/views` 目錄中，而 `app/Livewire` 目錄則包含 Livewire 元件相對應的後端邏輯。


<a name="livewire-available-layouts"></a>
#### 可用佈局

Livewire 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設值，但您可以透過修改應用程式 `resources/views/components/layouts/app.blade.php` 檔案使用的佈局來切換到標頭佈局。此外，您應該將 `container` 屬性新增到主要的 Flux 元件：

```blade
<x-layouts.app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts.app.header>
```


<a name="livewire-authentication-page-layout-variants"></a>
#### 驗證頁面佈局變體

Livewire 入門套件中包含的驗證頁面，例如登入頁面與註冊頁面，也提供三種不同的佈局變體：「簡約 (simple)」、「卡片 (card)」和「分割 (split)」。

若要更改您的驗證佈局，請修改應用程式 `resources/views/components/layouts/auth.blade.php` 檔案使用的佈局：

```blade
<x-layouts.auth.split>
    {{ $slot }}
</x-layouts.auth.split>
```

<a name="two-factor-authentication"></a>
## 雙因素驗證

所有入門套件都包含由 [Laravel Fortify](/docs/{{version}}/fortify#two-factor-authentication) 驅動的內建雙因素驗證 (2FA)，為使用者帳戶增加一層額外的安全性。使用者可以使用任何支援基於時間的一次性密碼 (TOTP) 的驗證器應用程式來保護他們的帳戶。

雙因素驗證預設啟用，並支援 [Fortify](/docs/{{version}}/fortify#two-factor-authentication) 提供的所有選項：

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
- 基於電子郵件的「魔術驗證」
- SSO

</div>

使用 WorkOS 作為您的驗證提供者 [需要一個 WorkOS 帳戶](https://workkit.com)。WorkOS 為每月活躍使用者數不超過 100 萬的應用程式提供免費驗證服務。

要使用 WorkOS AuthKit 作為您應用程式的驗證提供者，請透過 `laravel new` 建立新的入門套件驅動應用程式時選擇 WorkOS 選項。


### 設定您的 WorkOS 入門套件

使用 WorkOS 驅動的入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應與您在 WorkOS 控制台中為您的應用程式提供的值相符：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 控制台中配置應用程式首頁 URL。此 URL 是使用者登出應用程式後將被重新導向的位置。


<a name="configuring-authkit-authentication-methods"></a>
#### 設定 AuthKit 驗證方法

當使用 WorkOS 驅動的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 設定中停用「電子郵件 + 密碼」驗證，允許使用者僅透過社交驗證提供者、通行金鑰、「魔術驗證」和 SSO 進行驗證。這讓您的應用程式可以完全避免處理使用者密碼。


<a name="configuring-authkit-session-timeouts"></a>
#### 設定 AuthKit 工作階段逾時

此外，我們建議您將 WorkOS AuthKit 工作階段閒置逾時設定為與您的 Laravel 應用程式配置的工作階段逾時門檻相符，通常是兩小時。


<a name="inertia-ssr"></a>
### Inertia SSR

React 和 Vue 入門套件與 Inertia 的 [伺服器端渲染](https://inertiajs.com/server-side-rendering) 功能相容。要為您的應用程式建置一個 Inertia SSR 相容的套件，請執行 `build:ssr` 命令：

```shell
npm run build:ssr
```

為方便起見，還提供了一個 `composer dev:ssr` 命令。此命令會在為您的應用程式建置 SSR 相容套件後啟動 Laravel 開發伺服器和 Inertia SSR 伺服器，讓您可以使用 Inertia 的伺服器端渲染引擎在本地測試您的應用程式：

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

為了確保您的入門套件可供其他人使用，您需要將其發佈到 [Packagist](https://packagist.org)。您的入門套件應在其 `.env.example` 檔案中定義其所需的環境變數，並且任何必要的安裝後指令都應列在入門套件 `composer.json` 檔案的 `post-create-project-cmd` 陣列中。


<a name="faqs"></a>
### 常見問題


<a name="faq-upgrade"></a>
#### 如何升級？

每個入門套件都為您的下一個應用程式提供了堅實的起點。由於您完全擁有程式碼，您可以根據您的設想調整、自訂和建置您的應用程式。然而，無需更新入門套件本身。


<a name="faq-enable-email-verification"></a>
#### 如何啟用電子郵件驗證？

可以透過取消註解您的 `App/Models/User.php` 模型中的 `MustVerifyEmail` 匯入，並確保該模型實作 `MustVerifyEmail` 介面來新增電子郵件驗證：

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

註冊後，使用者將會收到一封驗證電子郵件。為了限制在使用者電子郵件地址驗證之前對某些路由的存取，請將 `verified` 中介層新增到路由中：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 當使用入門套件的 [WorkOS](#workos) 變體時，不需要電子郵件驗證。


<a name="faq-modify-email-template"></a>
#### 如何修改預設電子郵件範本？

您可能希望自訂預設電子郵件範本，使其與您的應用程式品牌更好地保持一致。要修改此範本，您應該使用以下命令將電子郵件視圖發佈到您的應用程式：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中產生多個檔案。您可以修改這些檔案以及 `resources/views/vendor/mail/themes/default.css` 檔案來改變預設電子郵件範本的外觀。