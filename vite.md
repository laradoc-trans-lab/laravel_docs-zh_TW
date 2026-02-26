# 資產打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel 外掛](#installing-vite-and-laravel-plugin)
  - [設定 Vite](#configuring-vite)
  - [載入你的腳本與樣式](#loading-your-scripts-and-styles)
- [執行 Vite](#running-vite)
- [使用 JavaScript](#working-with-scripts)
  - [別名](#aliases)
  - [Vue](#vue)
  - [React](#react)
  - [Svelte](#svelte)
  - [Inertia](#inertia)
  - [URL 處理](#url-processing)
- [使用樣式表](#working-with-stylesheets)
- [在 Blade 與路由中使用](#working-with-blade-and-routes)
  - [使用 Vite 處理靜態資產](#blade-processing-static-assets)
  - [儲存時自動重新整理](#blade-refreshing-on-save)
  - [別名](#blade-aliases)
- [資產預先抓取](#asset-prefetching)
- [自訂基礎 URL](#custom-base-urls)
- [環境變數](#environment-variables)
- [在測試中禁用 Vite](#disabling-vite-in-tests)
- [伺服器端渲染 (SSR)](#ssr)
- [Script 與 Style 標籤屬性](#script-and-style-attributes)
  - [內容安全政策 (CSP) Nonce](#content-security-policy-csp-nonce)
  - [子資源完整性 (SRI)](#subresource-integrity-sri)
  - [任意屬性](#arbitrary-attributes)
- [進階自訂](#advanced-customization)
  - [開發伺服器跨來源資源共享 (CORS)](#cors)
  - [修正開發伺服器 URL](#correcting-dev-server-urls)

<a name="introduction"></a>
## 簡介

[Vite](https://vitejs.dev) 是一個現代的前端建構工具，提供了極快的開發環境，並將您的程式碼打包成正式環境使用的資源。在使用 Laravel 建立應用程式時，您通常會使用 Vite 將應用程式的 CSS 和 JavaScript 檔案打包成準備好用於正式環境的資產。

Laravel 透過提供官方外掛和 Blade 指令來無縫整合 Vite，以便在開發和正式環境中載入您的資產。


<a name="installation"></a>
## 安裝與設定

> [!NOTE]
> 以下文件討論了如何手動安裝和設定 Laravel Vite 外掛。然而，Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了所有這些基本架構，是開始使用 Laravel 和 Vite 最快的方式。


<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 和 Laravel 外掛之前，您必須確保已安裝 Node.js (16+) 和 NPM：

```shell
node -v
npm -v
```

您可以從 [Node 官方網站](https://nodejs.org/en/download/)使用簡單的圖形安裝程式輕鬆安裝最新版本的 Node 和 NPM。或者，如果您使用的是 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，您可以透過 Sail 呼叫 Node 和 NPM：

```shell
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```


<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel 外掛

在全新的 Laravel 安裝中，您會在應用程式目錄結構的根目錄中找到 `package.json` 檔案。預設的 `package.json` 檔案已經包含了開始使用 Vite 和 Laravel 外掛所需的一切。您可以透過 NPM 安裝應用程式的前端依賴項目：

```shell
npm install
```


<a name="configuring-vite"></a>
### 設定 Vite

Vite 是透過專案根目錄下的 `vite.config.js` 檔案進行設定。您可以根據需要自由自訂此檔案，也可以安裝應用程式需要的任何其他外掛，例如 `@vitejs/plugin-react`、`@sveltejs/vite-plugin-svelte` 或 `@vitejs/plugin-vue`。

Laravel Vite 外掛需要您指定應用程式的進入點。這些可以是 JavaScript 或 CSS 檔案，並包含預處理語言，如 TypeScript、JSX、TSX 和 Sass。

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

如果您正在建立單頁應用程式 (SPA)，包括使用 Inertia 建立的應用程式，Vite 在沒有 CSS 進入點的情況下運作效果最佳：

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

相反地，您應該透過 JavaScript 導入您的 CSS。通常，這會在應用程式的 `resources/js/app.js` 檔案中完成：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel 外掛還支援多個進入點和進階設定選項，例如 [SSR 進入點](#ssr)。


<a name="working-with-a-secure-development-server"></a>
#### 使用安全的開發伺服器

如果您的本地開發網頁伺服器是透過 HTTPS 提供應用程式服務，您在連接到 Vite 開發伺服器時可能會遇到問題。

如果您使用的是 [Laravel Herd](https://herd.laravel.com) 且已對網站進行了安全保護，或者您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並對應用程式執行了 [secure 指令](/docs/{{version}}/valet#securing-sites)，Laravel Vite 外掛將自動為您偵測並使用產生的 TLS 憑證。

如果您使用與應用程式目錄名稱不符的主機來保護網站，您可以在應用程式的 `vite.config.js` 檔案中手動指定主機：

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

使用其他網頁伺服器時，您應該產生一個受信任的憑證，並手動設定 Vite 以使用產生的憑證：

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

如果您無法為系統產生受信任的憑證，您可以安裝並設定 [@vitejs/plugin-basic-ssl 外掛](https://github.com/vitejs/vite-plugin-basic-ssl)。使用不受信任的憑證時，您需要在執行 `npm run dev` 指令時，點擊控制台中的「Local」連結，在瀏覽器中接受 Vite 開發伺服器的憑證警告。


<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 上的 Sail 執行開發伺服器

當在 Windows Subsystem for Linux 2 (WSL2) 上的 [Laravel Sail](/docs/{{version}}/sail) 中執行 Vite 開發伺服器時，您應該將以下設定加入到您的 `vite.config.js` 檔案中，以確保瀏覽器可以與開發伺服器通訊：

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

如果在開發伺服器執行時，您的檔案更改沒有反映在瀏覽器中，您可能還需要設定 Vite 的 [server.watch.usePolling 選項](https://vitejs.dev/config/server-options.html#server-watch)。


<a name="loading-your-scripts-and-styles"></a>
### 載入你的腳本與樣式

設定好 Vite 進入點後，您現在可以在新增到應用程式根模板 `<head>` 中的 `@vite()` Blade 指令中引用它們：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

如果您是透過 JavaScript 導入 CSS，則只需包含 JavaScript 進入點：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` 指令會自動偵測 Vite 開發伺服器並注入 Vite 客戶端，以啟用熱模組替換 (Hot Module Replacement)。在建構模式下，該指令將載入您編譯後且具有版本號的資產，包括任何導入的 CSS。

如果需要，您也可以在呼叫 `@vite` 指令時指定編譯資產的建構路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```


<a name="inline-assets"></a>
#### 行內資產

有時可能需要包含資產的原始內容，而不是連結到資產的版本化 URL。例如，在將 HTML 內容傳遞給 PDF 產生器時，您可能需要直接在頁面中包含資產內容。您可以使用 `Vite` Facade 提供的 `content` 方法輸出 Vite 資產的內容：

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

執行 Vite 有兩種方式。您可以透過 `dev` 指令執行開發伺服器，這在本地開發時非常有用。開發伺服器會自動偵測檔案的變更，並立即反映在任何已開啟的瀏覽器視窗中。

或者，執行 `build` 指令會對應用程式的資產進行版本化並打包，為部署到正式環境做好準備：

```shell
# Run the Vite development server...
npm run dev

# Build and version the assets for production...
npm run build
```

如果您是在 [Sail](/docs/{{version}}/sail) 上的 WSL2 中執行開發伺服器，您可能需要一些[額外的設定](#configuring-hmr-in-sail-on-wsl2)選項。


<a name="working-with-scripts"></a>
## 使用 JavaScript


<a name="aliases"></a>
### 別名

預設情況下，Laravel 外掛提供了一個常用的別名，幫助您快速上手並方便地匯入應用程式的資產：

```js
{
    '@' => '/resources/js'
}
```

您可以透過在 `vite.config.js` 設定檔中新增自己的別名來覆寫 `'@'` 別名：

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

如果您想使用 [Vue](https://vuejs.org/) 框架建構前端，則還需要安裝 `@vitejs/plugin-vue` 外掛：

```shell
npm install --save-dev @vitejs/plugin-vue
```

接著，您可以將此外掛包含在 `vite.config.js` 設定檔中。在 Laravel 中使用 Vue 外掛時，您會需要一些額外的選項：

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
> Laravel 的 [starter kits](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Vue 與 Vite 設定。這些 starter kits 是開始使用 Laravel、Vue 與 Vite 的最快方式。


<a name="react"></a>
### React

如果您想使用 [React](https://reactjs.org/) 框架建構前端，則還需要安裝 `@vitejs/plugin-react` 外掛：

```shell
npm install --save-dev @vitejs/plugin-react
```

接著，您可以將此外掛包含在 `vite.config.js` 設定檔中：

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

您需要確保任何包含 JSX 的檔案都具有 `.jsx` 或 `.tsx` 副檔名，並記住視需要更新您的進入點，如[上方所示](#configuring-vite)。

您還需要在現有的 `@vite` 指令旁包含額外的 `@viteReactRefresh` Blade 指令。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` 指令必須在 `@vite` 指令之前呼叫。

> [!NOTE]
> Laravel 的 [starter kits](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、React 與 Vite 設定。這些 starter kits 是開始使用 Laravel、React 與 Vite 的最快方式。


<a name="svelte"></a>
### Svelte

如果您想使用 [Svelte](https://svelte.dev/) 框架建構前端，則還需要安裝 `@sveltejs/vite-plugin-svelte` 外掛：

```shell
npm install --save-dev @sveltejs/vite-plugin-svelte
```

接著，您可以將此外掛包含在 `vite.config.js` 設定檔中。

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
> Laravel 的 [starter kits](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Svelte 與 Vite 設定。這些 starter kits 是開始使用 Laravel、Svelte 與 Vite 的最快方式。


<a name="inertia"></a>
### Inertia

Laravel Vite 外掛提供了一個方便的 `resolvePageComponent` 函式，幫助您解析 Inertia 頁面元件。以下是該輔助函式與 Vue 3 搭配使用的範例；不過，您也可以在 React 或 Svelte 等其他框架中使用此函式：

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

如果您在 Inertia 中使用 Vite 的代碼分割 (Code Splitting) 功能，我們建議設定[資產預先抓取](#asset-prefetching)。

> [!NOTE]
> Laravel 的 [starter kits](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Inertia 與 Vite 設定。這些 starter kits 是開始使用 Laravel、Inertia 與 Vite 的最快方式。


<a name="url-processing"></a>
### URL 處理

在使用 Vite 並於應用程式的 HTML、CSS 或 JS 中引用資產時，有幾個注意事項需要考慮。首先，如果您使用絕對路徑引用資產，Vite 將不會在建構中包含該資產；因此，您應確保該資產在您的 public 目錄中可用。使用[獨立的 CSS 進入點](#configuring-vite)時應避免使用絕對路徑，因為在開發期間，瀏覽器會嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而不是從您的 public 目錄載入。

引用相對資產路徑時，請記住這些路徑是相對於引用它們的檔案。任何透過相對路徑引用的資產都將被 Vite 重寫、版本化並打包。

考慮以下專案結構：

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

以下範例示範了 Vite 如何處理相對與絕對 URL：

```html
<!-- This asset is not handled by Vite and will not be included in the build -->
<img src="/taylor.png">

<!-- This asset will be re-written, versioned, and bundled by Vite -->
<img src="../../images/abigail.png">
```

<a name="working-with-stylesheets"></a>
## 使用樣式表

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Tailwind 與 Vite 設定。或者，如果您想在不使用我們的入門套件的情況下使用 Tailwind 與 Laravel，請參考 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

所有的 Laravel 應用程式都已經包含了 Tailwind 與一個設定妥當的 `vite.config.js` 檔案。因此，您只需要啟動 Vite 開發伺服器或執行 `dev` Composer 命令，這將同時啟動 Laravel 與 Vite 開發伺服器：

```shell
composer run dev
```

您應用程式的 CSS 可以放置在 `resources/css/app.css` 檔案中。


<a name="working-with-blade-and-routes"></a>
## 在 Blade 與路由中使用


<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資產

在 JavaScript 或 CSS 中引用資產時，Vite 會自動處理它們並加上版本號。此外，在建置基於 Blade 的應用程式時，Vite 也可以處理並對您僅在 Blade 樣板中引用的靜態資產加上版本號。

然而，為了達成這一點，您需要透過將靜態資產匯入到應用程式的進入點，讓 Vite 知道這些資產的存在。例如，如果您想處理並對儲存在 `resources/images` 中的所有圖片和儲存在 `resources/fonts` 中的所有字體加上版本號，您應該在應用程式的 `resources/js/app.js` 進入點加入以下內容：

```js
import.meta.glob([
  '../images/**',
  '../fonts/**',
]);
```

現在，當執行 `npm run build` 時，這些資產將會被 Vite 處理。接著，您可以在 Blade 樣板中使用 `Vite::asset` 方法來引用這些資產，該方法將回傳指定資產的帶版本號 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```


<a name="blade-refreshing-on-save"></a>
### 儲存時自動重新整理

當您的應用程式是使用傳統的 Blade 伺服器端渲染建置時，Vite 可以透過在您修改應用程式中的視圖檔案時自動重新整理瀏覽器，來改善您的開發工作流程。要開始使用，您只需將 `refresh` 選項指定為 `true`。

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

當 `refresh` 選項為 `true` 時，在執行 `npm run dev` 期間，儲存以下目錄中的檔案將觸發瀏覽器執行全頁重新整理：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

如果您在應用程式的前端使用 [Ziggy](https://github.com/tighten/ziggy) 來產生路由連結，監控 `routes/**` 目錄會非常有用。

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

在底層，Laravel Vite 外掛使用了 [vite-plugin-full-reload](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，該套件提供了一些進階設定選項來微調此功能的行為。如果您需要這種程度的自訂，您可以提供一個 `config` 定義：

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

在 JavaScript 應用程式中，為經常引用的目錄[建立別名](#aliases)是很常見的。但是，您也可以透過在 `Illuminate\Support\Facades\Vite` 類別上使用 `macro` 方法，建立在 Blade 中使用的別名。通常，「macros (巨集)」應該定義在 [service provider](/docs/{{version}}/providers) 的 `boot` 方法中：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
}
```

一旦定義了巨集，就可以在您的樣板中呼叫它。例如，我們可以使用上面定義的 `image` 巨集來引用位於 `resources/images/logo.png` 的資產：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```


<a name="asset-prefetching"></a>
## 資產預先抓取

使用 Vite 的程式碼拆分 (Code Splitting) 功能建置 SPA 時，每當頁面導覽時都會抓取所需的資產。這種行為可能會導致 UI 渲染延遲。如果這對您選擇的前端框架造成問題，Laravel 提供了在初始頁面載入時積極預先抓取應用程式 JavaScript 與 CSS 資產的能力。

您可以透過在 [service provider](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `Vite::prefetch` 方法，來指示 Laravel 積極地預先抓取您的資產：

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

在上面的範例中，每次頁面載入時，資產將以最多 `3` 個同時下載的方式進行預先抓取。您可以根據應用程式的需求修改同時下載的數量，或者如果應用程式應該一次下載所有資產，則不指定限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

預設情況下，預先抓取將在 [頁面 _load_ 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event)觸發時開始。如果您想自訂預先抓取開始的時間，可以指定一個 Vite 將會監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上面的程式碼，預先抓取現在將在您手動於 `window` 物件上發送 `vite:prefetch` 事件時開始。例如，您可以讓預先抓取在頁面載入三秒後開始：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```


<a name="custom-base-urls"></a>
## 自訂基礎 URL

如果您由 Vite 編譯的資產部署到與應用程式不同的網域（例如透過 CDN），則必須在應用程式的 `.env` 檔案中指定 `ASSET_URL` 環境變數：

```env
ASSET_URL=https://cdn.example.com
```

設定資產 URL 後，所有指向您資產的重寫 URL 都將加上設定的值作為前綴：

```text
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記住，[絕對 URL 不會被 Vite 重寫](#url-processing)，因此它們不會被加上前綴。


<a name="environment-variables"></a>
## 環境變數

您可以透過在應用程式的 `.env` 檔案中為環境變數加上 `VITE_` 前綴，將它們注入到您的 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

您可以透過 `import.meta.env` 物件存取被注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

<a name="disabling-vite-in-tests"></a>
## 在測試中禁用 Vite

Laravel 的 Vite 整合會在執行測試時嘗試解析您的資產，這要求您必須執行 Vite 開發伺服器或建置您的資產。

如果您希望在測試期間模擬 Vite，可以呼叫 `withoutVite` 方法，該方法可用於任何繼承 Laravel 的 `TestCase` 類別的測試：

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

如果您想為所有測試禁用 Vite，可以在基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutVite` 方法：

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

Laravel Vite 外掛讓使用 Vite 設定伺服器端渲染變得非常簡單。要開始使用，請在 `resources/js/ssr.js` 建立一個 SSR 進入點，並透過將設定選項傳遞給 Laravel 外掛來指定該進入點：

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

為確保您不會忘記重新建置 SSR 進入點，我們建議擴展應用程式 `package.json` 中的 "build" 腳本以建立您的 SSR 建置：

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

如果您正在使用 [帶有 Inertia 的 SSR](https://inertiajs.com/server-side-rendering)，則可以使用 `inertia:start-ssr` Artisan 指令來啟動 SSR 伺服器：

```shell
php artisan inertia:start-ssr
```

> [!NOTE]
> Laravel 的 [入門套件](/docs/{{version}}/starter-kits) 已經包含了正確的 Laravel、Inertia SSR 與 Vite 設定。這些入門套件是開始使用 Laravel、Inertia SSR 與 Vite 的最快方式。


<a name="script-and-style-attributes"></a>
## Script 與 Style 標籤屬性


<a name="content-security-policy-csp-nonce"></a>
### 內容安全政策 (CSP) Nonce

如果您希望在 script 和 style 標籤中包含 [nonce 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce) 作為 [內容安全政策](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，可以在自訂 [中介層](/docs/{{version}}/middleware) 中使用 `useCspNonce` 方法來產生或指定 nonce：

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

呼叫 `useCspNonce` 方法後，Laravel 將自動在所有產生的 script 和 style 標籤上包含 `nonce` 屬性。

如果您需要在其他地方指定 nonce，包括 Laravel [入門套件](/docs/{{version}}/starter-kits) 中包含的 [Ziggy `@route` 指令](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，可以使用 `cspNonce` 方法取得它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個想要讓 Laravel 使用的 nonce，可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```


<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite manifest 包含了資產的 `integrity` 雜湊值，Laravel 將自動在它產生的任何 script 和 style 標籤上加入 `integrity` 屬性，以強制執行 [子資源完整性](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在其 manifest 中包含 `integrity` 雜湊值，但您可以透過安裝 [vite-plugin-manifest-sri](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 外掛來啟用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

接著您可以在 `vite.config.js` 檔案中啟用此外掛：

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

如果需要，您還可以自訂可在其中找到完整性雜湊值的 manifest 鍵名：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果您想完全禁用此自動偵測，可以將 `false` 傳遞給 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```


<a name="arbitrary-attributes"></a>
### 任意屬性

如果您需要在 script 和 style 標籤上包含額外的屬性，例如 [data-turbo-track](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，可以透過 `useScriptTagAttributes` 和 `useStyleTagAttributes` 方法來指定它們。通常，這些方法應該在 [服務提供者](/docs/{{version}}/providers) 中呼叫：

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

如果您需要有條件地加入屬性，可以傳遞一個回呼函式，該函式將接收資產來源路徑、其 URL、其 manifest 區塊 (chunk) 以及整個 manifest：

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
> 在 Vite 開發伺服器執行期間，`$chunk` 和 `$manifest` 參數將為 `null`。

<a name="advanced-customization"></a>
## 進階自訂

預設情況下，Laravel 的 Vite 外掛使用合理的慣例，應適用於大多數應用程式；然而，有時您可能需要自訂 Vite 的行為。為了啟用額外的自訂選項，我們提供了以下方法與選項，可以用來替代 `@vite` Blade 指令：

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

在 `vite.config.js` 檔案中，您也應該指定相同的設定：

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
### 開發伺服器跨來源資源共享 (CORS)

如果您在從 Vite 開發伺服器抓取資產時遇到瀏覽器的跨來源資源共享 (CORS) 問題，您可能需要授予您的自訂來源 (Origin) 存取開發伺服器的權限。Vite 結合 Laravel 外掛後，在不需要任何額外設定的情況下允許以下來源：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- 專案 `.env` 中的 `APP_URL`

為您的專案允許自訂來源的最簡單方法，是確保應用程式的 `APP_URL` 環境變數與您在瀏覽器中造訪的來源相符。例如，如果您造訪 `https://my-app.laravel`，您應該更新您的 `.env` 以使其相符：

```env
APP_URL=https://my-app.laravel
```

如果您需要對來源進行更細粒度的控制（例如支援多個來源），您應該利用 [Vite 全面且靈活的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。例如，您可以在專案的 `vite.config.js` 檔案中的 `server.cors.origin` 設定選項裡指定多個來源：

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

您也可以包含正規表示式 (Regex patterns)，如果您想允許給定頂級域名（例如 `*.laravel`）的所有來源，這會很有幫助：

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

Vite 生態系統中的某些外掛程式會假設以正斜線開頭的 URL 始終指向 Vite 開發伺服器。然而，由於 Laravel 整合的特性，情況並非如此。

例如，當 Vite 正在提供您的資產時，`vite-imagetools` 外掛會輸出如下的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` 外掛預期輸出的 URL 會被 Vite 攔截，然後外掛就可以處理所有以 `/@imagetools` 開頭的 URL。如果您使用的外掛程式預期這種行為，您將需要手動修正 URL。您可以透過使用 `transformOnServe` 選項在 `vite.config.js` 檔案中完成此操作。

在這個特定的範例中，我們將在產生的程式碼中，把開發伺服器 URL 加到所有出現 `/@imagetools` 的地方之前：

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

現在，當 Vite 在提供資產時，它將輸出指向 Vite 開發伺服器的 URL：

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! add] -->
```