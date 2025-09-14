# 入門套件

- [簡介](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用的入門套件](#available-starter-kits)
    - [React](#react)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [入門套件客製化](#starter-kit-customization)
    - [React](#react-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [WorkOS AuthKit 身份驗證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建構新的 Laravel 應用程式時能有個好的開始，我們很高興能提供[應用程式入門套件](https://laravel.com/starter-kits)。這些入門套件讓您在建構下一個 Laravel 應用程式時能有個好的開始，並包含您註冊與驗證應用程式使用者所需的路由、控制器與視圖。

雖然歡迎您使用這些入門套件，但它們並非強制。您可以透過簡單地安裝全新的 Laravel，從頭開始建構自己的應用程式。無論哪種方式，我們相信您都能建構出色的應用程式！

<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

若要使用我們的入門套件之一建立新的 Laravel 應用程式，您應首先[安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 與 Composer，您可以透過 Composer 安裝 Laravel 安裝程式 CLI 工具：

```shell
composer global require laravel/installer
```

然後，使用 Laravel 安裝程式 CLI 建立新的 Laravel 應用程式。Laravel 安裝程式將提示您選擇偏好的入門套件：

```shell
laravel new my-app
```

建立 Laravel 應用程式後，您只需透過 NPM 安裝其前端依賴項並啟動 Laravel 開發伺服器：

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

我們的 React 入門套件提供了一個強大、現代的起點，用於使用 [Inertia](https://inertiajs.com) 建構帶有 React 前端的 Laravel 應用程式。

Inertia 允許您使用傳統的伺服器端路由與控制器來建構現代的單頁 React 應用程式。這讓您可以享受 React 的前端能力，結合 Laravel 令人難以置信的後端生產力以及閃電般的 Vite 編譯速度。

React 入門套件使用 React 19、TypeScript、Tailwind 和 [shadcn/ui](https://ui.shadcn.com) 元件庫。

<a name="vue"></a>
### Vue

我們的 Vue 入門套件提供了一個很好的起點，用於使用 [Inertia](https://inertiajs.com) 建構帶有 Vue 前端的 Laravel 應用程式。

Inertia 允許您使用傳統的伺服器端路由與控制器來建構現代的單頁 Vue 應用程式。這讓您可以享受 Vue 的前端能力，結合 Laravel 令人難以置信的後端生產力以及閃電般的 Vite 編譯速度。

Vue 入門套件使用 Vue Composition API、TypeScript、Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。

<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件提供了一個完美的起點，用於建構帶有 [Laravel Livewire](https://livewire.laravel.com) 前端的 Laravel 應用程式。

Livewire 是一種強大的方式，僅使用 PHP 即可建構動態、響應式的前端 UI。對於主要使用 Blade 模板並尋求比 JavaScript 驅動的 SPA 框架（如 React 和 Vue）更簡單替代方案的團隊來說，它非常適合。

Livewire 入門套件使用 Livewire、Tailwind 和 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 入門套件客製化

<a name="react-customization"></a>
### React

我們的 React 入門套件是使用 Inertia 2、React 19、Tailwind 4 和 [shadcn/ui](https://ui.shadcn.com) 建構的。與我們所有的入門套件一樣，所有後端和前端程式碼都存在於您的應用程式中，以允許完全客製化。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼以客製化應用程式的外觀和行為：

```text
resources/js/
├── components/    # 可重複使用的 React 元件
├── hooks/         # React hooks
├── layouts/       # 應用程式佈局
├── lib/           # 工具函數和配置
├── pages/         # 頁面元件
└── types/         # TypeScript 定義
```

若要發布額外的 shadcn 元件，請先[找到您要發布的元件](https://ui.shadcn.com)。然後，使用 `npx` 發布元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該命令會將 Switch 元件發布到 `resources/js/components/ui/switch.tsx`。一旦元件發布，您就可以在任何頁面中使用它：

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
#### 可用的佈局

React 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設的，但您可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂部匯入的佈局來切換到標頭佈局：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```

<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌」變體和「浮動」變體。您可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```

<a name="react-authentication-page-layout-variants"></a>
#### 身份驗證頁面佈局變體

React 入門套件中包含的身份驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的佈局變體：「簡約」、「卡片」和「分割」。

若要更改您的身份驗證佈局，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂部匯入的佈局：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```

<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件是使用 Inertia 2、Vue 3 Composition API、Tailwind 和 [shadcn-vue](https://www.shadcn-vue.com/) 建構的。與我們所有的入門套件一樣，所有後端和前端程式碼都存在於您的應用程式中，以允許完全客製化。

大部分前端程式碼位於 `resources/js` 目錄中。您可以自由修改任何程式碼以客製化應用程式的外觀和行為：

```text
resources/js/
├── components/    # 可重複使用的 Vue 元件
├── composables/   # Vue composables / hooks
├── layouts/       # 應用程式佈局
├── lib/           # 工具函數和配置
├── pages/         # 頁面元件
└── types/         # TypeScript 定義
```

若要發布額外的 shadcn-vue 元件，請先[找到您要發布的元件](https://www.shadcn-vue.com)。然後，使用 `npx` 發布元件：

```shell
npx shadcn-vue@latest add switch
```

在此範例中，該命令會將 Switch 元件發布到 `resources/js/components/ui/Switch.vue`。一旦元件發布，您就可以在任何頁面中使用它：

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
#### 可用的佈局

Vue 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設的，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂部匯入的佈局來切換到標頭佈局：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```

<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄佈局包含三種不同的變體：預設側邊欄變體、「內嵌」變體和「浮動」變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇您最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```

<a name="vue-authentication-page-layout-variants"></a>
#### 身份驗證頁面佈局變體

Vue 入門套件中包含的身份驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的佈局變體：「簡約」、「卡片」和「分割」。

若要更改您的身份驗證佈局，請修改應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂部匯入的佈局：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```

<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件是使用 Livewire 3、Tailwind 和 [Flux UI](https://fluxui.dev/) 建構的。與我們所有的入門套件一樣，所有後端和前端程式碼都存在於您的應用程式中，以允許完全客製化。

#### Livewire 和 Volt

大部分前端程式碼位於 `resources/views` 目錄中。您可以自由修改任何程式碼以客製化應用程式的外觀和行為：

```text
resources/views
├── components            # 可重複使用的 Livewire 元件
├── flux                  # 客製化的 Flux 元件
├── livewire              # Livewire 頁面
├── partials              # 可重複使用的 Blade 部分
├── dashboard.blade.php   # 已驗證使用者的儀表板
├── welcome.blade.php     # 訪客歡迎頁面
```

#### 傳統 Livewire 元件

前端程式碼位於 `resources/views` 目錄中，而 `app/Livewire` 目錄包含 Livewire 元件的相應後端邏輯。

<a name="livewire-available-layouts"></a>
#### 可用的佈局

Livewire 入門套件包含兩種不同的主要佈局供您選擇：「側邊欄」佈局和「標頭」佈局。側邊欄佈局是預設的，但您可以透過修改應用程式 `resources/views/components/layouts/app.blade.php` 檔案使用的佈局來切換到標頭佈局。此外，您應該將 `container` 屬性新增到主要的 Flux 元件：

```blade
<x-layouts.app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts.app.header>
```

<a name="livewire-authentication-page-layout-variants"></a>
#### 身份驗證頁面佈局變體

Livewire 入門套件中包含的身份驗證頁面，例如登入頁面和註冊頁面，也提供三種不同的佈局變體：「簡約」、「卡片」和「分割」。

若要更改您的身份驗證佈局，請修改應用程式 `resources/views/components/layouts/auth.blade.php` 檔案使用的佈局：

```blade
<x-layouts.auth.split>
    {{ $slot }}
</x-layouts.auth.split>
```

<a name="workos"></a>
## WorkOS AuthKit 身份驗證

預設情況下，React、Vue 和 Livewire 入門套件都使用 Laravel 內建的身份驗證系統來提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們還提供每個入門套件的 [WorkOS AuthKit](https://authkit.com) 支援變體，提供：

<div class="content-list" markdown="1">

- 社群身份驗證 (Google、Microsoft、GitHub 和 Apple)
- Passkey 身份驗證
- 基於電子郵件的「Magic Auth」
- SSO

</div>

使用 WorkOS 作為您的身份驗證提供者[需要一個 WorkOS 帳戶](https://workos.com)。WorkOS 為每月活躍使用者高達 100 萬的應用程式提供免費身份驗證。

若要使用 WorkOS AuthKit 作為應用程式的身份驗證提供者，請在透過 `laravel new` 建立新的入門套件支援應用程式時選擇 WorkOS 選項。

### 配置您的 WorkOS 入門套件

使用 WorkOS 支援的入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 和 `WORKOS_REDIRECT_URL` 環境變數。這些變數應與 WorkOS 儀表板中為您的應用程式提供的值相符：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您應該在 WorkOS 儀表板中配置應用程式首頁 URL。此 URL 是使用者登出應用程式後將被重新導向的位置。

<a name="configuring-authkit-authentication-methods"></a>
#### 配置 AuthKit 身份驗證方法

使用 WorkOS 支援的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 配置設定中停用「電子郵件 + 密碼」身份驗證，允許使用者僅透過社群身份驗證提供者、Passkey、「Magic Auth」和 SSO 進行身份驗證。這讓您的應用程式可以完全避免處理使用者密碼。

<a name="configuring-authkit-session-timeouts"></a>
#### 配置 AuthKit 工作階段逾時

此外，我們建議您將 WorkOS AuthKit 工作階段非活動逾時配置為與您的 Laravel 應用程式配置的工作階段逾時閾值相符，通常為兩小時。

<a name="inertia-ssr"></a>
### Inertia SSR

React 和 Vue 入門套件與 Inertia 的[伺服器端渲染](https://inertiajs.com/server-side-rendering)功能相容。若要為您的應用程式建構 Inertia SSR 相容的套件，請執行 `build:ssr` 命令：

```shell
npm run build:ssr
```

為方便起見，也提供了 `composer dev:ssr` 命令。此命令將在為您的應用程式建構 SSR 相容套件後啟動 Laravel 開發伺服器和 Inertia SSR 伺服器，讓您可以使用 Inertia 的伺服器端渲染引擎在本地測試您的應用程式：

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

為確保您的入門套件可供他人使用，您需要將其發布到 [Packagist](https://packagist.org)。您的入門套件應在其 `.env.example` 檔案中定義其所需的環境變數，並且任何必要的安裝後命令應列在入門套件 `composer.json` 檔案的 `post-create-project-cmd` 陣列中。

<a name="faqs"></a>
### 常見問題

<a name="faq-upgrade"></a>
#### 如何升級？

每個入門套件都為您的下一個應用程式提供了堅實的起點。透過對程式碼的完全所有權，您可以根據自己的設想調整、客製化和建構您的應用程式。但是，無需更新入門套件本身。

<a name="faq-enable-email-verification"></a>
#### 如何啟用電子郵件驗證？

可以透過取消註解 `App/Models/User.php` 模型中的 `MustVerifyEmail` 匯入並確保模型實作 `MustVerifyEmail` 介面來新增電子郵件驗證：

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

註冊後，使用者將收到驗證電子郵件。若要限制對某些路由的存取，直到使用者的電子郵件地址經過驗證，請將 `verified` Middleware 新增到路由：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 使用入門套件的 [WorkOS](#workos) 變體時，不需要電子郵件驗證。

<a name="faq-modify-email-template"></a>
#### 如何修改預設電子郵件模板？

您可能希望客製化預設電子郵件模板，使其更好地符合應用程式的品牌。若要修改此模板，您應該使用以下命令將電子郵件視圖發布到您的應用程式：

```
php artisan vendor:publish --tag=laravel-mail
```

這將在 `resources/views/vendor/mail` 中產生多個檔案。您可以修改這些檔案以及 `resources/views/vendor/mail/themes/default.css` 檔案，以更改預設電子郵件模板的外觀。
