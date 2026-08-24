# 入門套件

- [簡介](#introduction)
- [使用入門套件建立應用程式](#creating-an-application)
- [可用的入門套件](#available-starter-kits)
    - [React](#react)
    - [Svelte](#svelte)
    - [Vue](#vue)
    - [Livewire](#livewire)
- [客製化入門套件](#starter-kit-customization)
    - [React](#react-customization)
    - [Svelte](#svelte-customization)
    - [Vue](#vue-customization)
    - [Livewire](#livewire-customization)
- [認證](#authentication)
    - [啟用與停用功能](#enabling-and-disabling-features)
    - [客製化使用者建立與密碼重設](#customizing-actions)
    - [雙重認證](#two-factor-authentication)
    - [速率限制](#rate-limiting)
- [團隊](#teams)
- [WorkOS AuthKit 認證](#workos)
- [Inertia SSR](#inertia-ssr)
- [社群維護的入門套件](#community-maintained-starter-kits)
- [常見問題](#faqs)

<a name="introduction"></a>
## 簡介

為了讓您在建立全新的 Laravel 應用程式時能搶先一步，我們很榮幸能提供[應用程式入門套件](https://laravel.com/starter-kits)。這些入門套件能讓您快速展開下一個 Laravel 應用程式的開發，其中包含註冊與認證應用程式使用者所需的路由、控制器以及視圖。入門套件使用 [Laravel Fortify](/docs/{{version}}/fortify) 來提供認證功能。

雖然我們非常歡迎您使用這些入門套件，但這並非強制要求。您完全可以從頭開始建立自己的應用程式，只需要安裝一個乾淨的全新 Laravel 即可。無論選擇哪種方式，我們相信您都能打造出優秀的作品！


<a name="creating-an-application"></a>
## 使用入門套件建立應用程式

若要使用我們的入門套件建立新的 Laravel 應用程式，您應先[安裝 PHP 與 Laravel CLI 工具](/docs/{{version}}/installation#installing-php)。如果您已經安裝了 PHP 和 Composer，您可以透過 Composer 安裝 Laravel 安裝程式 CLI 工具：

```shell
composer global require laravel/installer
```

接著，使用 Laravel 安裝程式 CLI 來建立新的 Laravel 應用程式。Laravel 安裝程式會提示您選擇偏好的入門套件：

```shell
laravel new my-app
```

建立好 Laravel 應用程式後，您只需要透過 NPM 安裝前端相依套件並啟動 Laravel 開發伺服器：

```shell
cd my-app
npm install && npm run build
composer run dev
```

一旦啟動 Laravel 開發伺服器，您就可以在網頁瀏覽器中透過 [http://localhost:8000](http://localhost:8000) 來存取您的應用程式。


<a name="available-starter-kits"></a>
## 可用的入門套件


<a name="react"></a>
### React

我們的 React 入門套件為透過 [Inertia](https://inertiajs.com) 使用 React 前端建置 Laravel 應用程式提供了一個穩健且現代化的起點。

Inertia 允許您使用經典的伺服器端路由與控制器來建立現代化的單頁 React 應用程式。這讓您既能享受 React 的強大前端能力，又能結合 Laravel 驚人的後端生產力以及 Vite 極速的編譯體驗。

React 入門套件使用了 React 19、TypeScript、Tailwind 以及 [shadcn/ui](https://ui.shadcn.com) 元件庫。


<a name="svelte"></a>
### Svelte

我們的 Svelte 入門套件為透過 [Inertia](https://inertiajs.com) 使用 Svelte 前端建置 Laravel 應用程式提供了一個穩健且現代化的起點。

Inertia 允許您使用經典的伺服器端路由與控制器來建立現代化的單頁 Svelte 應用程式。這讓您既能享受 Svelte 的強大前端能力，又能結合 Laravel 驚人的後端生產力以及 Vite 極速的編譯體驗。

Svelte 入門套件使用了 Svelte 5、TypeScript、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 元件庫。


<a name="vue"></a>
### Vue

我們的 Vue 入門套件為透過 [Inertia](https://inertiajs.com) 使用 Vue 前端建置 Laravel 應用程式提供了一個絕佳的起點。

Inertia 允許您使用經典的伺服器端路由與控制器來建立現代化的單頁 Vue 應用程式。這讓您既能享受 Vue 的強大前端能力，又能結合 Laravel 驚人的後端生產力以及 Vite 極速的編譯體驗。

Vue 入門套件使用了 Vue Composition API、TypeScript、Tailwind 以及 [shadcn-vue](https://www.shadcn-vue.com/) 元件庫。


<a name="livewire"></a>
### Livewire

我們的 Livewire 入門套件為使用 [Laravel Livewire](https://livewire.laravel.com) 前端建置 Laravel 應用程式提供了完美的起點。

Livewire 是一種僅需使用 PHP 即可建置動態、響應式前端 UI 的強大方式。它非常適合主要使用 Blade 模板，並在尋找比 React、Svelte 和 Vue 等 JavaScript 驅動的 SPA 框架更簡單替代方案的團隊。

Livewire 入門套件使用了 Livewire、Tailwind 以及 [Flux UI](https://fluxui.dev) 元件庫。

<a name="starter-kit-customization"></a>
## 客製化入門套件


<a name="react-customization"></a>
### React

我們的 React 入門套件是使用 Inertia 3、React 19、Tailwind 4 以及 [shadcn/ui](https://ui.shadcn.com) 所打造。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於你的應用程式中，以實現完整的客製化。

大部分的前端程式碼都位於 `resources/js` 目錄中。你可以自由修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable React components
├── hooks/         # React hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn 元件，請先[尋找你想要發布的元件](https://ui.shadcn.com)。接著，使用 `npx` 來發布該元件：

```shell
npx shadcn@latest add switch
```

在此範例中，該指令會將 Switch 元件發布至 `resources/js/components/ui/switch.tsx`。元件發布後，你就可以在任何頁面中使用它：

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
#### 可用的版面配置

React 入門套件包含兩種不同的主要版面配置供你選擇："sidebar" 版面配置與 "header" 版面配置。側邊欄版面配置為預設值，但你可以透過修改應用程式 `resources/js/layouts/app-layout.tsx` 檔案頂端匯入的版面配置來切換至頁首版面配置：

```js
import AppLayoutTemplate from '@/layouts/app/app-sidebar-layout'; // [tl! remove]
import AppLayoutTemplate from '@/layouts/app/app-header-layout'; // [tl! add]
```


<a name="react-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設的側邊欄變體、"inset" 變體以及 "floating" 變體。你可以透過修改 `resources/js/components/app-sidebar.tsx` 元件來選擇最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="react-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

React 入門套件所包含的認證頁面（例如登入頁面與註冊頁面）同樣提供三種不同的版面配置變體："simple"、"card" 與 "split"。

若要變更你的認證版面配置，請修改應用程式 `resources/js/layouts/auth-layout.tsx` 檔案頂端匯入的版面配置：

```js
import AuthLayoutTemplate from '@/layouts/auth/auth-simple-layout'; // [tl! remove]
import AuthLayoutTemplate from '@/layouts/auth/auth-split-layout'; // [tl! add]
```


<a name="svelte-customization"></a>
### Svelte

我們的 Svelte 入門套件是使用 Inertia 3、Svelte 5、Tailwind 以及 [shadcn-svelte](https://www.shadcn-svelte.com/) 所打造。與我們所有的入門套件一樣，所有的後端與前端程式碼都存在於你的應用程式中，以實現完整的客製化。

大部分的前端程式碼都位於 `resources/js` 目錄中。你可以自由修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Svelte components
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration and Svelte rune modules
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布額外的 shadcn-svelte 元件，請先[尋找你想要發布的元件](https://www.shadcn-svelte.com)。接著，使用 `npx` 來發布該元件：

```shell
npx shadcn-svelte@latest add switch
```

在此範例中，該指令會將 Switch 元件發布至 `resources/js/components/ui/switch/switch.svelte`。元件發布後，你就可以在任何頁面中使用它：

```svelte
<script lang="ts">
    import { Switch } from '@/components/ui/switch'
</script>

<div>
    <Switch />
</div>
```


<a name="svelte-available-layouts"></a>
#### 可用的版面配置

Svelte 入門套件包含兩種不同的主要版面配置供你選擇："sidebar" 版面配置與 "header" 版面配置。側邊欄版面配置為預設值，但你可以透過修改應用程式 `resources/js/layouts/AppLayout.svelte` 檔案頂端匯入的版面配置來切換至頁首版面配置：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.svelte'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.svelte'; // [tl! add]
```


<a name="svelte-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面配置包含三種不同的變體：預設的側邊欄變體、"inset" 變體以及 "floating" 變體。你可以透過修改 `resources/js/components/AppSidebar.svelte` 元件來選擇最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```


<a name="svelte-authentication-page-layout-variants"></a>
#### 認證頁面版面配置變體

Svelte 入門套件所包含的認證頁面（例如登入頁面與註冊頁面）同樣提供三種不同的版面配置變體："simple"、"card" 與 "split"。

若要變更你的認證版面配置，請修改應用程式 `resources/js/layouts/AuthLayout.svelte` 檔案頂端匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.svelte'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.svelte'; // [tl! add]
```

<a name="vue-customization"></a>
### Vue

我們的 Vue 入門套件基於 Inertia 3、Vue 3 Composition API、Tailwind 與 [shadcn-vue](https://www.shadcn-vue.com/) 所打造。如同我們所有的入門套件一樣，所有後端與前端的程式碼都直接包含在您的應用程式中，以便您進行完全的客製化。

大部分的前端程式碼都位於 `resources/js` 目錄中。您可以隨意修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/js/
├── components/    # Reusable Vue components
├── composables/   # Vue composables / hooks
├── layouts/       # Application layouts
├── lib/           # Utility functions and configuration
├── pages/         # Page components
└── types/         # TypeScript definitions
```

若要發布更多的 shadcn-vue 元件，請先[尋找您想要發布的元件](https://www.shadcn-vue.com)。接著，使用 `npx` 來發布該元件：

```shell
npx shadcn-vue@latest add switch
```

在這個範例中，該指令會將 Switch 元件發布至 `resources/js/components/ui/Switch.vue`。元件發布完成後，您就可以在任何頁面中使用它：

```vue
<script setup lang="ts">
import { Switch } from '@/components/ui/switch'
</script>

<template>
    <div>
        <Switch />
    </div>
</template>
```

<a name="vue-available-layouts"></a>
#### 可用的版面配置

Vue 入門套件包含兩種不同的主要版面配置供您選擇："sidebar"（側邊欄）版面與 "header"（頁首）版面。側邊欄版面為預設值，但您可以透過修改應用程式 `resources/js/layouts/AppLayout.vue` 檔案頂端所匯入的版面配置來切換至頁首版面：

```js
import AppLayout from '@/layouts/app/AppSidebarLayout.vue'; // [tl! remove]
import AppLayout from '@/layouts/app/AppHeaderLayout.vue'; // [tl! add]
```

<a name="vue-sidebar-variants"></a>
#### 側邊欄變體

側邊欄版面包含三種不同的變體：預設側邊欄變體、"inset" 變體以及 "floating" 變體。您可以透過修改 `resources/js/components/AppSidebar.vue` 元件來選擇最喜歡的變體：

```text
<Sidebar collapsible="icon" variant="sidebar"> [tl! remove]
<Sidebar collapsible="icon" variant="inset"> [tl! add]
```

<a name="vue-authentication-page-layout-variants"></a>
#### 認證頁面版面變體

Vue 入門套件包含的認證頁面（例如登入頁面與註冊頁面）也提供了三種不同的版面配置變體："simple"、"card" 與 "split"。

若要變更您的認證版面配置，請修改您應用程式 `resources/js/layouts/AuthLayout.vue` 檔案頂端匯入的版面配置：

```js
import AuthLayout from '@/layouts/auth/AuthSimpleLayout.vue'; // [tl! remove]
import AuthLayout from '@/layouts/auth/AuthSplitLayout.vue'; // [tl! add]
```

<a name="livewire-customization"></a>
### Livewire

我們的 Livewire 入門套件基於 Livewire 4、Tailwind 與 [Flux UI](https://fluxui.dev/) 所打造。如同我們所有的入門套件一樣，所有後端與前端的程式碼都直接包含在您的應用程式中，以便您進行完全的客製化。

大部分的前端程式碼都位於 `resources/views` 目錄中。您可以隨意修改任何程式碼，以客製化應用程式的外觀與行為：

```text
resources/views
├── components            # Reusable components
├── flux                  # Customized Flux components
├── layouts               # Application layouts
├── pages                 # Livewire pages
├── partials              # Reusable Blade partials
├── dashboard.blade.php   # Authenticated user dashboard
├── welcome.blade.php     # Guest user welcome page
```

<a name="livewire-available-layouts"></a>
#### 可用的版面配置

Livewire 入門套件包含兩種不同的主要版面配置供您選擇："sidebar"（側邊欄）版面與 "header"（頁首）版面。側邊欄版面為預設值，但您可以透過修改應用程式 `resources/views/layouts/app.blade.php` 檔案所使用的版面配置來切換至頁首版面。此外，您應該在主要 Flux 元件上新增 `container` 屬性：

```blade
<x-layouts::app.header>
    <flux:main container>
        {{ $slot }}
    </flux:main>
</x-layouts::app.header>
```

<a name="livewire-authentication-page-layout-variants"></a>
#### 認證頁面版面變體

Livewire 入門套件包含的認證頁面（例如登入頁面與註冊頁面）也提供了三種不同的版面配置變體："simple"、"card" 與 "split"。

若要變更您的認證版面配置，請修改您應用程式 `resources/views/layouts/auth.blade.php` 檔案所使用的版面配置：

```blade
<x-layouts::auth.split>
    {{ $slot }}
</x-layouts::auth.split>
```

<a name="authentication"></a>
## 認證

所有入門套件皆使用 [Laravel Fortify](/docs/{{version}}/fortify) 來處理認證。Fortify 提供登入、註冊、重設密碼、驗證 Email 等功能的路由、控制器與邏輯。

Fortify 會根據您應用程式的 `config/fortify.php` 設定檔中所啟用的功能，自動註冊以下認證路由：

<div class="overflow-auto">

| 路由 | 方法 | 說明 |
| ---------------------------------- | ------ | ----------------------------------- |
| `/login`                           | `GET`    | 顯示登入表單 |
| `/login`                           | `POST`   | 進行使用者認證 |
| `/logout`                          | `POST`   | 將使用者登出 |
| `/register`                        | `GET`    | 顯示註冊表單 |
| `/register`                        | `POST`   | 建立新使用者 |
| `/forgot-password`                 | `GET`    | 顯示密碼重設申請表單 |
| `/forgot-password`                 | `POST`   | 傳送密碼重設連結 |
| `/reset-password/{token}`          | `GET`    | 顯示密碼重設表單 |
| `/reset-password`                  | `POST`   | 更新密碼 |
| `/email/verify`                    | `GET`    | 顯示 Email 驗證提示 |
| `/email/verify/{id}/{hash}`        | `GET`    | 驗證 Email 地址 |
| `/email/verification-notification` | `POST`   | 重新傳送驗證 Email |
| `/user/confirm-password`           | `GET`    | 顯示密碼確認表單 |
| `/user/confirm-password`           | `POST`   | 確認密碼 |
| `/two-factor-challenge`            | `GET`    | 顯示雙重認證 (2FA) 驗證表單 |
| `/two-factor-challenge`            | `POST`   | 驗證雙重認證 (2FA) 驗證碼 |

</div>

可以使用 `php artisan route:list` Artisan 指令來列出您應用程式中的所有路由。


<a name="enabling-and-disabling-features"></a>
### 啟用與停用功能

您可以在應用程式的 `config/fortify.php` 設定檔中，控制要啟用哪些 Fortify 功能：

```php
use Laravel\Fortify\Features;

'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
],
```

若要停用某項功能，只需註解掉或從 `features` 陣列中移除該功能項目即可。例如，移除 `Features::registration()` 可停用公開註冊功能。

當使用 [React](#react)、[Svelte](#svelte) 或 [Vue](#vue) 入門套件時，您還需要移除前端程式碼中所有對已停用功能之路由的引用。例如，如果您停用了 Email 驗證，就應該在 React、Svelte 或 Vue 元件中移除 `verification` 路由的匯入與引用。這是因為這些入門套件使用 Wayfinder 進行型別安全的路由處理，而 Wayfinder 會在建置時產生路由定義。如果您引用了不再存在的路由，應用程式將無法完成建置。


<a name="customizing-actions"></a>
### 客製化使用者建立與密碼重設

當使用者註冊或重設密碼時，Fortify 會呼叫位於應用程式 `app/Actions/Fortify` 目錄中的 Action 類別：

<div class="overflow-auto">

| 檔案 | 說明 |
| ----------------------------- | ------------------------------------- |
| `CreateNewUser.php`           | 驗證並建立新使用者 |
| `ResetUserPassword.php`       | 驗證並更新使用者密碼 |
| `PasswordValidationRules.php` | 定義密碼驗證規則 |

</div>

例如，若要客製化您應用程式的註冊邏輯，您應該編輯 `CreateNewUser` Action：

```php
public function create(array $input): User
{
    Validator::make($input, [
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'email', 'max:255', 'unique:users'],
        'phone' => ['required', 'string', 'max:20'], // [tl! add]
        'password' => $this->passwordRules(),
    ])->validate();

    return User::create([
        'name' => $input['name'],
        'email' => $input['email'],
        'phone' => $input['phone'], // [tl! add]
        'password' => Hash::make($input['password']),
    ]);
}
```


<a name="two-factor-authentication"></a>
### 雙重認證

入門套件內建雙重認證 (2FA) 功能，允許使用者使用任何支援 TOTP 的驗證器 App 來保護其帳號安全。預設情況下，透過您應用程式 `config/fortify.php` 設定檔中的 `Features::twoFactorAuthentication()` 已啟用 2FA。

`confirm` 選項要求使用者在完全啟用 2FA 之前先驗證一組驗證碼，而 `confirmPassword` 則要求在啟用或停用 2FA 之前先進行密碼確認。更多詳細資訊請參閱 [Fortify 的雙重認證文件](/docs/{{version}}/fortify#two-factor-authentication)。


<a name="rate-limiting"></a>
### 速率限制

速率限制可以防止暴力破解和重複的登入嘗試癱瘓您的認證端點。您可以在應用程式的 `FortifyServiceProvider` 中客製化 Fortify 的速率限制行為：

```php
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email.$request->ip());
});
```


<a name="teams"></a>
## 團隊

React、Svelte、Vue 和 Livewire 入門套件在建立時也可以選擇包含團隊支援。當啟用團隊功能時，每個使用者可以屬於一個或多個團隊，並擁有一個「目前團隊」。在註冊時，系統會自動為新使用者建立一個個人團隊。入門套件還包含了團隊管理畫面，用於建立團隊、切換團隊、邀請成員以及更新團隊詳細資訊。

當某個路由作用域限定於目前團隊時，URL 中會包含目前團隊的 slug。例如，儀表板路由會變為 `/{current_team}/dashboard`，而團隊管理頁面則使用像是 `settings/teams/{team}` 的路由。當使用 `{current_team}` 和 `{team}` 路由參數時，入門套件會自動確保已認證的使用者屬於所請求的團隊，然後才允許存取該路由。

為了更方便生成感知團隊脈絡的 URL，入門套件為已認證使用者的目前團隊註冊了 URL 預設值。這使得呼叫如 `route('dashboard')` 之類的輔助函式時，會自動包含目前團隊的 slug。當使用者登入、註冊或切換團隊時，入門套件會更新目前團隊並重新整理這些 URL 預設值，確保產生的連結繼續使用正確的團隊上下文。

在建立或重新命名團隊時，入門套件也會防止使用者選擇保留名稱，這些保留名稱可能會產生不安全或衝突的路由片段。例如，會與 `settings`、`login` 或 `dashboard` 等路由前綴發生衝突的名稱均不可使用。

<a name="workos"></a>
## WorkOS AuthKit 認證

預設情況下，React、Svelte、Vue 與 Livewire 入門套件均採用 Laravel 內建的認證系統，提供登入、註冊、重設密碼、Email 驗證等功能。此外，我們也為每個入門套件提供了由 [WorkOS AuthKit](https://authkit.com) 驅動的變體，其提供：

<div class="content-list" markdown="1">

- 社群帳號認證 (Google, Microsoft, GitHub, 及 Apple)
- Passkey 認證
- 基於 Email 的「Magic Auth」
- SSO

</div>

使用 WorkOS 作為您的認證提供者[需要 WorkOS 帳號](https://workos.com)。WorkOS 為每月活躍使用者數達 100 萬以內的應用程式提供免費認證服務。

若要使用 WorkOS AuthKit 作為您應用程式的認證提供者，請在透過 `laravel new` 建立新的入門套件應用程式時選擇 WorkOS 選項。


### 設定您的 WorkOS 入門套件

使用由 WorkOS 驅動的入門套件建立新應用程式後，您應該在應用程式的 `.env` 檔案中設定 `WORKOS_CLIENT_ID`、`WORKOS_API_KEY` 與 `WORKOS_REDIRECT_URL` 環境變數。這些變數應該與 WorkOS 主控台針對您的應用程式所提供的值一致：

```ini
WORKOS_CLIENT_ID=your-client-id
WORKOS_API_KEY=your-api-key
WORKOS_REDIRECT_URL="${APP_URL}/authenticate"
```

此外，您還應該在 WorkOS 主控台中設定應用程式的首頁 URL。此 URL 是使用者從您的應用程式登出後會被重新導向的位置。


<a name="configuring-authkit-authentication-methods"></a>
#### 設定 AuthKit 認證方式

當使用由 WorkOS 驅動的入門套件時，我們建議您在應用程式的 WorkOS AuthKit 設定中停用「Email + 密碼」認證，讓使用者只能透過社群認證提供者、Passkey、「Magic Auth」與 SSO 進行認證。這可以讓您的應用程式完全免於處理使用者密碼。


<a name="configuring-authkit-session-timeouts"></a>
#### 設定 AuthKit 工作階段逾時

此外，我們建議您將 WorkOS AuthKit 的工作階段閒置逾時時間，設定為與 Laravel 應用程式所設定的工作階段逾時門檻一致（通常為兩小時）。


<a name="inertia-ssr"></a>
### Inertia SSR

React、Svelte 與 Vue 入門套件皆相容於 Inertia 的[伺服器端渲染 (server-side rendering)](https://inertiajs.com/server-side-rendering) 功能。若要為您的應用程式建置相容於 Inertia SSR 的打包檔案 (bundle)，請執行 `build:ssr` 指令：

```shell
npm run build:ssr
```

為了方便起見，同時也提供了 `composer dev:ssr` 指令。此指令會在為您的應用程式建置 SSR 相容打包檔案後，啟動 Laravel 開發伺服器與 Inertia SSR 伺服器，讓您可以利用 Inertia 的伺服器端渲染引擎在本機測試應用程式：

```shell
composer dev:ssr
```


<a name="community-maintained-starter-kits"></a>
### 社群維護的入門套件

使用 Laravel 安裝程式建立新的 Laravel 應用程式時，您可以將 Packagist 上提供的任何社群維護入門套件傳遞給 `--using` 旗標：

```shell
laravel new my-app --using=example/starter-kit
```


<a name="creating-starter-kits"></a>
#### 建立入門套件

若要確保其他人可以使用您的入門套件，您需要將其發布至 [Packagist](https://packagist.org)。您的入門套件應在其 `.env.example` 檔案中定義所需的環境變數，並且任何必要的安裝後指令都應列在入門套件 `composer.json` 檔案的 `post-create-project-cmd` 陣列中。


<a name="faqs"></a>
### 常見問題


<a name="faq-upgrade"></a>
#### 如何升級？

每個入門套件都為您的下一個應用程式提供了堅實的起點。擁有了程式碼的全權所有權，您可以完全按照自己的構想去微調、客製化與建構應用程式。因此，並沒有更新入門套件本身的必要。


<a name="faq-enable-email-verification"></a>
#### 如何啟用 Email 驗證？

只要取消註解 `App/Models/User.php` 模型中的 `MustVerifyEmail` 引用，並確保該模型實作了 `MustVerifyEmail` 介面，即可新增 Email 驗證：

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

註冊後，使用者將會收到一封驗證 Email。若要限制某些路由在使用者的 Email 地址驗證通過前無法存取，請將 `verified` 中介層新增至這些路由：

```php
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', function () {
        return Inertia::render('dashboard');
    })->name('dashboard');
});
```

> [!NOTE]
> 使用入門套件的 [WorkOS](#workos) 變體時不需要進行 Email 驗證。


<a name="faq-modify-email-template"></a>
#### 如何修改預設的 Email 範本？

您可能希望客製化預設的 Email 範本，使其更符合您應用程式的品牌形象。若要修改此範本，您應該使用以下指令將 Email 視圖發布至您的應用程式：

```
php artisan vendor:publish --tag=laravel-mail
```

這將會在 `resources/views/vendor/mail` 中產生數個檔案。您可以修改這些檔案中的任何一個以及 `resources/views/vendor/mail/themes/default.css` 檔案，來更改預設 Email 範本的外觀與樣式。