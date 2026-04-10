# 靜態資源打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel 插件](#installing-vite-and-laravel-plugin)
  - [設定 Vite](#configuring-vite)
  - [載入您的指令碼與樣式](#loading-your-scripts-and-styles)
- [執行 Vite](#running-vite)
- [使用 JavaScript](#working-with-scripts)
  - [別名](#aliases)
  - [Vue](#vue)
  - [React](#react)
  - [Svelte](#svelte)
  - [Inertia](#inertia)
  - [URL 處理](#url-processing)
- [使用樣式表](#working-with-stylesheets)
- [使用 Blade 與路由](#working-with-blade-and-routes)
  - [使用 Vite 處理靜態資源](#blade-processing-static-assets)
  - [儲存時自動重新整理](#blade-refreshing-on-save)
  - [別名](#blade-aliases)
- [資源預取](#asset-prefetching)
- [自定義基礎 URL](#custom-base-urls)
- [環境變數](#environment-variables)
- [在測試中禁用 Vite](#disabling-vite-in-tests)
- [伺服器端渲染 (SSR)](#ssr)
- [指令碼與樣式標籤屬性](#script-and-style-attributes)
  - [內容安全性原則 (CSP) Nonce](#content-security-policy-csp-nonce)
  - [子資源完整性 (SRI)](#subresource-integrity-sri)
  - [任意屬性](#arbitrary-attributes)
- [進階自定義](#advanced-customization)
  - [開發伺服器跨來源資源共用 (CORS)](#cors)
  - [修正開發伺服器 URL](#correcting-dev-server-urls)

<a name="introduction"></a>
## 簡介

[Vite](https://vitejs.dev) 是一個現代的前端建構工具，能提供極快的開發環境並為正式環境打包您的程式碼。在建構 Laravel 應用程式時，您通常會使用 Vite 將應用程式的 CSS 與 JavaScript 檔案打包成可用於正式環境的資源。

Laravel 透過提供官方插件與 Blade 指令來無縫整合 Vite，以便在開發與正式環境中載入您的資源。


<a name="installation"></a>
## 安裝與設定

> [!NOTE]
> 以下文件將討論如何手動安裝與設定 Laravel Vite 插件。不過，Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了所有這些腳手架，是開始使用 Laravel 與 Vite 最快的方式。


<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 與 Laravel 插件之前，您必須確保已安裝 Node.js (16+) 與 NPM：

```shell
node -v
npm -v
```

您可以使用 [Node 官方網站](https://nodejs.org/en/download/) 提供的簡單圖形安裝程式輕鬆安裝最新版本的 Node 與 NPM。或者，如果您使用的是 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，您可以透過 Sail 呼叫 Node 與 NPM：

```shell
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```


<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel 插件

在剛安裝好的 Laravel 專案中，您會在應用程式目錄結構的根目錄下找到 `package.json` 檔案。預設的 `package.json` 檔案已經包含了使用 Vite 與 Laravel 插件所需的所有內容。您可以使用 NPM 安裝應用程式的前端依賴項目：

```shell
npm install
```


<a name="configuring-vite"></a>
### 設定 Vite

Vite 透過專案根目錄中的 `vite.config.js` 檔案進行設定。您可以根據需求自定義此檔案，也可以安裝應用程式所需的任何其他插件，例如 `@vitejs/plugin-react`、`@sveltejs/vite-plugin-svelte` 或 `@vitejs/plugin-vue`。

Laravel Vite 插件要求您指定應用程式的進入點 (entry points)。這些可以是 JavaScript 或 CSS 檔案，並包含預處理語言，例如 TypeScript、JSX、TSX 和 Sass。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css',
            'resources/js/app.js',
        ]),
    ],
});
```

如果您正在構建單頁應用程式 (SPA)，包括使用 Inertia 構建的應用程式，Vite 在沒有 CSS 進入點的情況下運作效果最好：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css', // [tl! remove]
            'resources/js/app.js',
        ]),
    ],
});
```

相反地，您應該透過 JavaScript 匯入 CSS。通常這會在應用程式的 `resources/js/app.js` 檔案中完成：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel 插件還支援多個進入點以及進階設定選項，例如 [SSR 進入點](#ssr)。


<a name="working-with-a-secure-development-server"></a>
#### 使用安全開發伺服器

如果您的本地開發 Web 伺服器透過 HTTPS 提供應用程式服務，您在連接 Vite 開發伺服器時可能會遇到問題。

如果您使用 [Laravel Herd](https://herd.laravel.com) 並已將網站設為安全，或者您使用 [Laravel Valet](/docs/{{version}}/valet) 並對應用程式執行了 [secure 指令](/docs/{{version}}/valet#securing-sites)，Laravel Vite 插件將會自動偵測並為您使用生成的 TLS 憑證。

如果您使用與應用程式目錄名稱不符的主機來確保網站安全，您可以在應用程式的 `vite.config.js` 檔案中手動指定主機：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            detectTls: 'my-app.test', // [tl! add]
        }),
    ],
});
```

當使用其他 Web 伺服器時，您應該生成一個受信任的憑證，並手動設定 Vite 以使用該生成的憑證：

```js
// ...
import fs from 'fs'; // [tl! add]

const host = 'my-app.test'; // [tl! add]

export default defineConfig({
    // ...
    server: { // [tl! add]
        host, // [tl! add]
        hmr: { host }, // [tl! add]
        https: { // [tl! add]
            key: fs.readFileSync(`/path/to/${host}.key`), // [tl! add]
            cert: fs.readFileSync(`/path/to/${host}.crt`), // [tl! add]
        }, // [tl! add]
    }, // [tl! add]
});
```

如果您無法為您的系統生成受信任的憑證，您可以安裝並設定 [@vitejs/plugin-basic-ssl 插件](https://github.com/vitejs/vite-plugin-basic-ssl)。使用不受信任的憑證時，您需要在瀏覽器中接受 Vite 開發伺服器的憑證警告，方法是在執行 `npm run dev` 指令時，點擊主控台中的 「Local」 連結。


<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 的 Sail 中執行開發伺服器

在 Windows Subsystem for Linux 2 (WSL2) 的 [Laravel Sail](/docs/{{version}}/sail) 中執行 Vite 開發伺服器時，您應該在 `vite.config.js` 檔案中加入以下設定，以確保瀏覽器可以與開發伺服器通訊：

```js
// ...

export default defineConfig({
    // ...
    server: { // [tl! add:start]
        hmr: {
            host: 'localhost',
        },
    }, // [tl! add:end]
});
```

如果開發伺服器執行期間，檔案變更未反映在瀏覽器中，您可能還需要設定 Vite 的 [server.watch.usePolling 選項](https://vitejs.dev/config/server-options.html#server-watch)。


<a name="loading-your-scripts-and-styles"></a>
### 載入您的指令碼與樣式

設定好 Vite 進入點後，您現在可以在 `@vite()` Blade 指令中引用它們，並將其添加到應用程式根模板的 `<head>` 中：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果您是透過 JavaScript 匯入 CSS，則只需要包含 JavaScript 進入點：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` 指令會自動偵測 Vite 開發伺服器並注入 Vite 用戶端以啟用熱模組替換 (Hot Module Replacement)。在建構模式下，該指令會載入您編譯且版本化的資源，包括任何匯入的 CSS。

如果需要，您在呼叫 `@vite` 指令時也可以指定編譯後資源的建構路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```


<a name="inline-assets"></a>
#### 內嵌資源

有時可能需要包含資源的原始內容，而不是連結到資源的版本化 URL。例如，當將 HTML 內容傳遞給 PDF 生成器時，您可能需要將資源內容直接包含在頁面中。您可以使用 `Vite` Facade 提供的 `content` 方法來輸出 Vite 資源的內容：

```blade
@use('Illuminate\Support\Facades\Vite')

<!doctype html>
<head>
    {{-- ... --}}

    <style>
        {!! Vite::content('resources/css/app.css') !!}
    </style>
    <script>
        {!! Vite::content('resources/js/app.js') !!}
    </script>
</head>
```

<a name="running-vite"></a>
## 執行 Vite

您可以使用兩種方式來執行 Vite。您可以使用 `dev` 指令來執行開發伺服器，這在本地開發時非常有用。開發伺服器會自動偵測您的檔案變更，並立即反映在任何開啟的瀏覽器視窗中。

或者，執行 `build` 指令將會為您的應用程式資源加上版本號並進行打包，使其準備好部署到生產環境：

```shell
# Run the Vite development server...
npm run dev

# Build and version the assets for production...
npm run build
```

如果您是在 WSL2 上的 [Sail](/docs/{{version}}/sail) 中執行開發伺服器，您可能需要一些 [額外設定](#configuring-hmr-in-sail-on-wsl2) 選項。


<a name="working-with-scripts"></a>
## 使用 JavaScript


<a name="aliases"></a>
### 別名

預設情況下，Laravel 插件提供了一個常用的別名，以幫助您快速上手並方便地匯入應用程式的資源：

```js
{
    '@' => '/resources/js'
}
```

您可以在 `vite.config.js` 設定檔中加入自己的別名來覆寫 `'@'` 別名：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel(['resources/ts/app.tsx']),
    ],
    resolve: {
        alias: {
            '@': '/resources/ts',
        },
    },
});
```


<a name="vue"></a>
### Vue

如果您想使用 [Vue](https://vuejs.org/) 框架來建構前端，則還需要安裝 `@vitejs/plugin-vue` 插件：

```shell
npm install --save-dev @vitejs/plugin-vue
```

接著您可以將該插件包含在 `vite.config.js` 設定檔中。在使用 Vue 插件搭配 Laravel 時，您會需要幾個額外選項：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.js']),
        vue({
            template: {
                transformAssetUrls: {
                    // The Vue plugin will re-write asset URLs, when referenced
                    // in Single File Components, to point to the Laravel web
                    // server. Setting this to `null` allows the Laravel plugin
                    // to instead re-write asset URLs to point to the Vite
                    // server instead.
                    base: null,

                    // The Vue plugin will parse absolute URLs and treat them
                    // as absolute paths to files on disk. Setting this to
                    // `false` will leave absolute URLs un-touched so they can
                    // reference assets in the public directory as expected.
                    includeAbsolute: false,
                },
            },
        }),
    ],
});
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Vue 與 Vite 設定。這些入門套件是開始使用 Laravel、Vue 與 Vite 最快的方式。


<a name="react"></a>
### React

如果您想使用 [React](https://reactjs.org/) 框架來建構前端，則還需要安裝 `@vitejs/plugin-react` 插件：

```shell
npm install --save-dev @vitejs/plugin-react
```

接著您可以將該插件包含在 `vite.config.js` 設定檔中：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import react from '@vitejs/plugin-react';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.jsx']),
        react(),
    ],
});
```

您需要確保任何包含 JSX 的檔案都具有 `.jsx` 或 `.tsx` 副檔名，並記得在需要時更新您的入口點，如 [上述](#configuring-vite) 所示。

您還需要在現有的 `@vite` 指令旁邊加入額外的 `@viteReactRefresh` Blade 指令。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` 指令必須在 `@vite` 指令之前呼叫。

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、React 與 Vite 設定。這些入門套件是開始使用 Laravel、React 與 Vite 最快的方式。


<a name="svelte"></a>
### Svelte

如果您想使用 [Svelte](https://svelte.dev/) 框架來建構前端，則還需要安裝 `@sveltejs/vite-plugin-svelte` 插件：

```shell
npm install --save-dev @sveltejs/vite-plugin-svelte
```

接著您可以將該插件包含在 `vite.config.js` 設定檔中。

```js
import { svelte } from '@sveltejs/vite-plugin-svelte';
import laravel from 'laravel-vite-plugin';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [
    laravel({
      input: ['resources/js/app.ts'],
      ssr: 'resources/js/ssr.ts',
      refresh: true,
    }),
    svelte(),
  ],
});
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Svelte 與 Vite 設定。這些入門套件是開始使用 Laravel、Svelte 與 Vite 最快的方式。


<a name="inertia"></a>
### Inertia

Laravel Vite 插件提供了一個方便的 `resolvePageComponent` 函式，幫助您解析 Inertia 頁面元件。以下是在 Vue 3 中使用該輔助函式的範例；不過，您也可以在 React 或 Svelte 等其他框架中使用此函式：

```js
import { createApp, h } from 'vue';
import { createInertiaApp } from '@inertiajs/vue3';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
  resolve: (name) => resolvePageComponent(`./Pages/${name}.vue`, import.meta.glob('./Pages/**/*.vue')),
  setup({ el, App, props, plugin }) {
    createApp({ render: () => h(App, props) })
      .use(plugin)
      .mount(el)
  },
});
```

如果您在 Inertia 中使用 Vite 的程式碼分割 (code splitting) 功能，我們建議設定 [資源預取](#asset-prefetching)。

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Inertia 與 Vite 設定。這些入門套件是開始使用 Laravel、Inertia 與 Vite 最快的方式。


<a name="url-processing"></a>
### URL 處理

在使用 Vite 並在應用程式的 HTML、CSS 或 JS 中引用資源時，有幾件事需要注意。首先，如果您使用絕對路徑引用資源，Vite 將不會將該資源包含在構建中；因此，您應該確保該資源存在於您的 public 目錄中。在使用 [專用的 CSS 入口點](#configuring-vite) 時，應避免使用絕對路徑，因為在開發過程中，瀏覽器會嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而不是從您的 public 目錄載入。

在引用相對資源路徑時，請記得路徑是相對於引用該資源的檔案。任何透過相對路徑引用的資源都將被 Vite 重新編寫、加上版本號並打包。

請參考以下專案結構：

```text
public/
  taylor.png
resources/
  js/
    Pages/
      Welcome.vue
  images/
    abigail.png
```

以下範例展示了 Vite 如何處理相對與絕對 URL：

```html
<!-- This asset is not handled by Vite and will not be included in the build -->
<img src="/taylor.png">

<!-- This asset will be re-written, versioned, and bundled by Vite -->
<img src="../../images/abigail.png">
```

<a name="working-with-stylesheets"></a>
## 使用樣式表

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Tailwind 與 Vite 設定。或者，如果您想在不使用入門套件的情況下使用 Tailwind 與 Laravel，請參閱 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

所有的 Laravel 應用程式都已經包含了 Tailwind 以及設定妥當的 `vite.config.js` 檔案。因此，您只需要啟動 Vite 開發伺服器，或是執行 `dev` Composer 指令，這將會同時啟動 Laravel 和 Vite 的開發伺服器：

```shell
composer run dev
```

您的應用程式 CSS 可以放置在 `resources/css/app.css` 檔案中。


<a name="working-with-blade-and-routes"></a>
## 使用 Blade 與路由


<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資源

當您在 JavaScript 或 CSS 中引用資源時，Vite 會自動處理並為其版本化。此外，在構建基於 Blade 的應用程式時，Vite 也可以處理並為那些僅在 Blade 模板中引用的靜態資源進行版本化。

然而，為了達成此目的，您需要透過在插件的 `assets` 選項中指定資源，讓 Vite 知道這些資源的存在。例如，如果您想要處理並版本化儲存在 `resources/images` 中的所有圖片以及儲存在 `resources/fonts` 中的所有字體，您應該在 Vite 設定中加入以下內容：

```js
laravel({
    input: 'resources/js/app.js',
    assets: ['resources/images/**', 'resources/fonts/**'],
})
```

現在，當執行 `npm run build` 時，這些資源將由 Vite 處理。接著您可以在 Blade 模板中使用 `Vite::asset` 方法來引用這些資源，該方法會回傳指定資源的版本化 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```

> [!NOTE]
> 在 Laravel Vite 插件第 3 版之前，靜態資源必須使用 `import.meta.glob` 在應用程式的進入點中匯入。`assets` 選項是因為 Vite 8 的變更而引入的。


<a name="blade-refreshing-on-save"></a>
### 儲存時自動重新整理

當您的應用程式使用傳統的 Blade 伺服器端渲染構建時，Vite 可以透過在您修改應用程式中的視圖檔案時自動重新整理瀏覽器，來改善您的開發工作流程。要開始使用，您只需將 `refresh` 選項指定為 `true`。

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: true,
        }),
    ],
});
```

當 `refresh` 選項為 `true` 時，在執行 `npm run dev` 期間，儲存以下目錄中的檔案將會觸發瀏覽器執行全頁重新整理：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

如果您在應用程式的前端利用 [Ziggy](https://github.com/tighten/ziggy) 來產生路由連結，那麼監控 `routes/**` 目錄會非常有用。

如果這些預設路徑不符合您的需求，您可以指定自己的監控路徑列表：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: ['resources/views/**'],
        }),
    ],
});
```

在底層，Laravel Vite 插件使用了 [vite-plugin-full-reload](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，該套件提供了一些進階設定選項來微調此功能的行為。如果您需要此等級的自定義，您可以提供一個 `config` 定義：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            refresh: [{
                paths: ['path/to/watch/**'],
                config: { delay: 300 }
            }],
        }),
    ],
});
```


<a name="blade-aliases"></a>
### 別名

在 JavaScript 應用程式中，為經常引用的目錄建立[別名](#aliases)是很常見的做法。但是，您也可以透過在 `Illuminate\Support\Facades\Vite` 類別上使用 `macro` 方法，建立可用於 Blade 的別名。通常，「巨集 (macros)」應該定義在[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
}
```

一旦定義了巨集，就可以在模板中呼叫它。例如，我們可以使用上面定義的 `image` 巨來引用位於 `resources/images/logo.png` 的資源：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```


<a name="asset-prefetching"></a>
## 資源預取

當使用 Vite 的程式碼分割 (code splitting) 功能構建 SPA 時，每次頁面導覽都會擷取所需的資源。這種行為可能會導致 UI 渲染延遲。如果這對您選擇的前端框架造成困擾，Laravel 提供了在初始頁面載入時，預先積極預取 (eagerly prefetch) 應用程式 JavaScript 與 CSS 資源的能力。

您可以在[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中呼叫 `Vite::prefetch` 方法，來指示 Laravel 積極預取您的資源：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Vite;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Vite::prefetch(concurrency: 3);
    }
}
```

在上述範例中，每次頁面載入時，資源將以最多 `3` 個同時下載的數量進行預取。您可以根據應用程式的需求修改同時執行數量，或者如果您希望應用程式一次下載所有資源，則不指定同時執行限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

預設情況下，預取將在[頁面 _load_ 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event)觸發時開始。如果您想自定義預取開始的時間，可以指定一個 Vite 將會監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上述程式碼，現在當您在 `window` 物件上手動發送 `vite:prefetch` 事件時，預取將會開始。例如，您可以設定在頁面載入三秒後開始預取：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```


<a name="custom-base-urls"></a>
## 自定義基礎 URL

如果您的 Vite 編譯資源被部署到與應用程式不同的網域（例如透過 CDN），您必須在應用程式的 `.env` 檔案中指定 `ASSET_URL` 環境變數：

```env
ASSET_URL=https://cdn.example.com
```

設定資源 URL 後，所有重新寫入的資源 URL 都會加上設定的值作為前綴：

```text
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記得，[絕對 URL 不會被 Vite 重新寫入](#url-processing)，因此它們不會被加上前綴。


<a name="environment-variables"></a>
## 環境變數

您可以在應用程式的 `.env` 檔案中，透過在變數前加上 `VITE_` 前綴，將環境變數注入到您的 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

您可以透過 `import.meta.env` 物件來存取注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

<a name="disabling-vite-in-tests"></a>
## 在測試中禁用 Vite

Laravel 的 Vite 整合在執行測試時會嘗試解析您的資源，這需要您執行 Vite 開發伺服器或建置您的資源。

如果您希望在測試期間模擬 (mock) Vite，可以呼叫 `withoutVite` 方法，該方法適用於任何繼承 Laravel `TestCase` 類別的測試：

```php tab=Pest
test('without vite example', function () {
    $this->withoutVite();

    // ...
});
```

```php tab=PHPUnit
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_without_vite_example(): void
    {
        $this->withoutVite();

        // ...
    }
}
```

如果您想要為所有測試禁用 Vite，可以從基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutVite` 方法：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void// [tl! add:start]
    {
        parent::setUp();

        $this->withoutVite();
    }// [tl! add:end]
}
```


<a name="ssr"></a>
## 伺服器端渲染 (SSR)

Laravel Vite 插件讓設定使用 Vite 的伺服器端渲染變得非常簡單。要開始使用，請在 `resources/js/ssr.js` 建立一個 SSR 入口點，並透過向 Laravel 插件傳遞設定選項來指定該入口點：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            ssr: 'resources/js/ssr.js',
        }),
    ],
});
```

為了確保您不會忘記重新建置 SSR 入口點，我們建議擴充應用程式 `package.json` 中的 "build" 腳本以建立您的 SSR 建置版本：

```json
"scripts": {
     "dev": "vite",
     "build": "vite build" // [tl! remove]
     "build": "vite build && vite build --ssr" // [tl! add]
}
```

接著，要建置並啟動 SSR 伺服器，您可以執行以下指令：

```shell
npm run build
node bootstrap/ssr/ssr.js
```

如果您使用 [搭配 Inertia 的 SSR](https://inertiajs.com/server-side-rendering)，則可以使用 `inertia:start-ssr` Artisan 指令來啟動 SSR 伺服器：

```shell
php artisan inertia:start-ssr
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Inertia SSR 與 Vite 設定。這些入門套件是開始使用 Laravel、Inertia SSR 與 Vite 最快的方式。


<a name="script-and-style-attributes"></a>
## 指令碼與樣式標籤屬性


<a name="content-security-policy-csp-nonce"></a>
### 內容安全性原則 (CSP) Nonce

如果您希望在指令碼和樣式標籤上包含 [nonce 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) 作為 [內容安全性原則 (Content Security Policy)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，您可以使用自定義 [中介層](/docs/{{version}}/middleware) 中的 `useCspNonce` 方法來產生或指定 nonce：

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Vite;
use Symfony\Component\HttpFoundation\Response;

class AddContentSecurityPolicyHeaders
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        Vite::useCspNonce();

        return $next($request)->withHeaders([
            'Content-Security-Policy' => "script-src 'nonce-".Vite::cspNonce()."'",
        ]);
    }
}
```

在呼叫 `useCspNonce` 方法後，Laravel 將自動在所有產生的指令碼和樣式標籤上包含 `nonce` 屬性。

如果您需要在其他地方指定 nonce，包括 Laravel [入門套件](/docs/{{version}}/starter-kits) 中包含的 [Ziggy `@route` 指令](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，您可以使用 `cspNonce` 方法來獲取它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個想要讓 Laravel 使用的 nonce，您可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```


<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite 資訊清單 (manifest) 為您的資源包含了 `integrity` 雜湊值，Laravel 會自動在產生的任何指令碼和樣式標籤上添加 `integrity` 屬性，以強制執行 [子資源完整性 (Subresource Integrity)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在資訊清單中包含 `integrity` 雜湊值，但您可以透過安裝 [vite-plugin-manifest-sri](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 插件來啟用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

接著您可以在 `vite.config.js` 檔案中啟用此插件：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import manifestSRI from 'vite-plugin-manifest-sri';// [tl! add]

export default defineConfig({
    plugins: [
        laravel({
            // ...
        }),
        manifestSRI(),// [tl! add]
    ],
});
```

如果需要，您也可以自定義存放完整性雜湊值的資訊清單鍵 (manifest key)：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果您想要完全禁用此自動偵測，可以將 `false` 傳遞給 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```


<a name="arbitrary-attributes"></a>
### 任意屬性

如果您需要在指令碼和樣式標籤上包含額外的屬性，例如 [data-turbo-track](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，您可以使用 `useScriptTagAttributes` 和 `useStyleTagAttributes` 方法來指定。通常，這些方法應該從 [服務提供者(Service Providers)](/docs/{{version}}/providers) 中呼叫：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes([
    'data-turbo-track' => 'reload', // Specify a value for the attribute...
    'async' => true, // Specify an attribute without a value...
    'integrity' => false, // Exclude an attribute that would otherwise be included...
]);

Vite::useStyleTagAttributes([
    'data-turbo-track' => 'reload',
]);
```

如果您需要有條件地添加屬性，可以傳遞一個回呼函數 (callback)，該函數將接收資源來源路徑、其 URL、其資訊清單區塊 (manifest chunk) 以及整個資訊清單：

```php
use Illuminate\Support\Facades\Vite;

Vite::useScriptTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $src === 'resources/js/app.js' ? 'reload' : false,
]);

Vite::useStyleTagAttributes(fn (string $src, string $url, array|null $chunk, array|null $manifest) => [
    'data-turbo-track' => $chunk && $chunk['isEntry'] ? 'reload' : false,
]);
```

> [!WARNING]
> 當 Vite 開發伺服器執行時，`$chunk` 與 `$manifest` 引數將會是 `null`。

<a name="advanced-customization"></a>
## 進階自定義

Laravel 的 Vite 插件開箱即用，使用了適合大多數應用程式的既定設計慣例；然而，有時您可能需要自定義 Vite 的行為。為了啟用額外的自定義選項，我們提供了以下方法與選項，可用於替代 `@vite` Blade 指令：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    {{
        Vite::useHotFile(storage_path('vite.hot')) // Customize the "hot" file...
            ->useBuildDirectory('bundle') // Customize the build directory...
            ->useManifestFilename('assets.json') // Customize the manifest filename...
            ->withEntryPoints(['resources/js/app.js']) // Specify the entry points...
            ->createAssetPathsUsing(function (string $path, ?bool $secure) { // Customize the backend path generation for built assets...
                return "https://cdn.example.com/{$path}";
            })
    }}
</head>
```

接著，您應該在 `vite.config.js` 檔案中指定相同的設定：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            hotFile: 'storage/vite.hot', // Customize the "hot" file...
            buildDirectory: 'bundle', // Customize the build directory...
            input: ['resources/js/app.js'], // Specify the entry points...
        }),
    ],
    build: {
      manifest: 'assets.json', // Customize the manifest filename...
    },
});
```


<a name="cors"></a>
### 開發伺服器跨來源資源共用 (CORS)

如果您在從 Vite 開發伺服器擷取資源時，在瀏覽器中遇到跨來源資源共用 (CORS) 問題，您可能需要授予自定義來源存取開發伺服器的權限。Vite 結合 Laravel 插件在無需任何額外設定的情況下，允許以下來源：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- 專案 `.env` 中的 `APP_URL`

為您的專案允許自定義來源最簡單的方法，是確保應用程式的 `APP_URL` 環境變數與您在瀏覽器中訪問的來源一致。例如，如果您訪問的是 `https://my-app.laravel`，您應該更新您的 `.env` 以使其匹配：

```env
APP_URL=https://my-app.laravel
```

如果您需要對來源進行更精細的控制（例如支持多個來源），則應利用 [Vite 全面且靈活的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。例如，您可以在專案的 `vite.config.js` 檔案中的 `server.cors.origin` 設定選項中指定多個來源：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            refresh: true,
        }),
    ],
    server: {  // [tl! add]
        cors: {  // [tl! add]
            origin: [  // [tl! add]
                'https://backend.laravel',  // [tl! add]
                'http://admin.laravel:8566',  // [tl! add]
            ],  // [tl! add]
        },  // [tl! add]
    },  // [tl! add]
});
```

您也可以包含正規表達式 (regex) 模式，如果您想允許特定頂級域名（例如 `*.laravel`）的所有來源，這將會很有幫助：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            refresh: true,
        }),
    ],
    server: {  // [tl! add]
        cors: {  // [tl! add]
            origin: [ // [tl! add]
                // Supports: SCHEME://DOMAIN.laravel[:PORT] [tl! add]
                /^https?:\/\/.*\.laravel(:\d+)?$/, //[tl! add]
            ], // [tl! add]
        }, // [tl! add]
    }, // [tl! add]
});
```


<a name="correcting-dev-server-urls"></a>
### 修正開發伺服器 URL

Vite 生態系統中的某些插件假設以正斜線 (forward-slash) 開頭的 URL 總是會指向 Vite 開發伺服器。然而，由於 Laravel 整合的特性，情況並非如此。

例如，當 Vite 正在提供您的資源時，`vite-imagetools` 插件會輸出如下的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` 插件預期輸出 URL 會被 Vite 攔截，然後插件可以處理所有以 `/@imagetools` 開頭的 URL。如果您使用的插件預期有這種行為，您需要手動修正這些 URL。您可以在 `vite.config.js` 檔案中使用 `transformOnServe` 選項來完成此操作。

在這個特定範例中，我們將在產生的程式碼中，將開發伺服器 URL 附加到所有 `/@imagetools` 的前面：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { imagetools } from 'vite-imagetools';

export default defineConfig({
    plugins: [
        laravel({
            // ...
            transformOnServe: (code, devServerUrl) => code.replaceAll('/@imagetools', devServerUrl+'/@imagetools'),
        }),
        imagetools(),
    ],
});
```

現在，當 Vite 正在提供資源時，它將輸出指向 Vite 開發伺服器的 URL：

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! add] -->
```