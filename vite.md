# 靜態資源打包 (Vite)

- [簡介](#introduction)
- [安裝與設定](#installation)
  - [安裝 Node](#installing-node)
  - [安裝 Vite 與 Laravel 外掛](#installing-vite-and-laravel-plugin)
  - [設定 Vite](#configuring-vite)
  - [載入您的腳本與樣式](#loading-your-scripts-and-styles)
- [執行 Vite](#running-vite)
- [使用 JavaScript](#working-with-scripts)
  - [別名](#aliases)
  - [Vue](#vue)
  - [React](#react)
  - [Svelte](#svelte)
  - [Inertia](#inertia)
  - [URL 處理](#url-processing)
- [使用樣式表](#working-with-stylesheets)
- [使用字型](#working-with-fonts)
  - [字型提供者](#font-providers)
  - [本地字型](#local-fonts)
  - [字型選項](#font-options)
- [使用 Blade 與路由](#working-with-blade-and-routes)
  - [使用 Vite 處理靜態資源](#blade-processing-static-assets)
  - [儲存時自動重新整理](#blade-refreshing-on-save)
  - [別名](#blade-aliases)
- [靜態資源預取](#asset-prefetching)
- [自訂基底 URL](#custom-base-urls)
- [環境變數](#environment-variables)
- [在測試中停用 Vite](#disabling-vite-in-tests)
- [伺服器端渲染 (SSR)](#ssr)
- [Script 與 Style 標籤屬性](#script-and-style-attributes)
  - [內容安全策略 (CSP) Nonce](#content-security-policy-csp-nonce)
  - [子資源完整性 (SRI)](#subresource-integrity-sri)
  - [任意屬性](#arbitrary-attributes)
- [進階自訂](#advanced-customization)
  - [開發伺服器跨來源資源共享 (CORS)](#cors)
  - [修正開發伺服器 URL](#correcting-dev-server-urls)

<a name="introduction"></a>
## 簡介

[Vite](https://vitejs.dev) 是一個現代化的前端打包工具，提供極快速的開發環境，並可將您的程式碼打包用於正式環境。當使用 Laravel 建置應用程式時，您通常會使用 Vite 將應用程式的 CSS 與 JavaScript 檔案打包成適合生產環境的靜態資源。

Laravel 透過提供官方外掛與 Blade 指令，無縫整合 Vite，以在開發與正式環境中載入您的靜態資源。


<a name="installation"></a>
## 安裝與設定

> [!NOTE]
> 以下文件討論了如何手動安裝與設定 Laravel Vite 外掛。不過，Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含所有這些基底架構，是開始使用 Laravel 與 Vite 最快的方式。


<a name="installing-node"></a>
### 安裝 Node

在執行 Vite 與 Laravel 外掛之前，您必須確保已安裝 Node.js (16+) 以及 NPM：

```shell
node -v
npm -v
```

您可以透過 [Node 官方網站](https://nodejs.org/en/download/) 的圖形化安裝程式，輕鬆安裝最新版本的 Node 與 NPM。或者，若您正在使用 [Laravel Sail](https://laravel.com/docs/{{version}}/sail)，您可以透過 Sail 來呼叫 Node 與 NPM：

```shell
./vendor/bin/sail node -v
./vendor/bin/sail npm -v
```


<a name="installing-vite-and-laravel-plugin"></a>
### 安裝 Vite 與 Laravel 外掛

在全新安裝的 Laravel 中，您會在應用程式目錄結構的根目錄中找到 `package.json` 檔案。預設的 `package.json` 檔案已經包含了您開始使用 Vite 和 Laravel 外掛所需的一切。您可以透過 NPM 安裝應用程式的前端相依套件：

```shell
npm install
```


<a name="configuring-vite"></a>
### 設定 Vite

Vite 是透過專案根目錄中的 `vite.config.js` 檔案進行設定。您可以根據需求自由自訂此檔案，也可以安裝應用程式所需的任何其他外掛，例如 `@vitejs/plugin-react`、`@sveltejs/vite-plugin-svelte` 或 `@vitejs/plugin-vue`。

Laravel Vite 外掛需要您指定應用程式的進入點。這些進入點可以是 JavaScript 或 CSS 檔案，並包含經預處理的語言，如 TypeScript、JSX、TSX 與 Sass。

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

若您正在建置單頁應用程式 (SPA)，包含使用 Inertia 建置的應用程式，Vite 在不使用 CSS 進入點時效果最佳：

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

相反地，您應該透過 JavaScript 匯入您的 CSS。通常這會在您應用程式的 `resources/js/app.js` 檔案中進行：

```js
import './bootstrap';
import '../css/app.css'; // [tl! add]
```

Laravel 外掛也支援多個進入點以及進階的設定選項，例如 [SSR 進入點](#ssr)。


<a name="working-with-a-secure-development-server"></a>
#### 使用安全的開發伺服器

若您的本機開發 Web 伺服器是透過 HTTPS 提供應用程式服務，您可能會在連線到 Vite 開發伺服器時遇到問題。

若您正在使用 [Laravel Herd](https://herd.laravel.com) 且已啟用了安全站點，或是您正在使用 [Laravel Valet](/docs/{{version}}/valet) 並對您的應用程式執行了 [secure 指令](/docs/{{version}}/valet#securing-sites)，Laravel Vite 外掛會自動偵測並為您使用產生的 TLS 憑證。

若您使用與應用程式目錄名稱不相符的主機名稱保護該站點，您可以手動在應用程式的 `vite.config.js` 檔案中指定主機名稱：

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

使用其他 Web 伺服器時，您應該產生一個可信任的憑證，並手動設定 Vite 使用產生的憑證：

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

若您無法為系統產生受信任的憑證，您可以安裝並設定 [@vitejs/plugin-basic-ssl 外掛](https://github.com/vitejs/vite-plugin-basic-ssl)。當使用不受信任的憑證時，您需要在執行 `npm run dev` 指令後，在主控台中點擊「Local」連結並於瀏覽器中接受 Vite 開發伺服器的憑證警告。


<a name="configuring-hmr-in-sail-on-wsl2"></a>
#### 在 WSL2 上的 Sail 執行開發伺服器

當在 Windows Subsystem for Linux 2 (WSL2) 的 [Laravel Sail](/docs/{{version}}/sail) 中執行 Vite 開發伺服器時，您應該將以下設定新增至 `vite.config.js` 檔案中，以確保瀏覽器可以與開發伺服器進行通訊：

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

若在開發伺服器運作時，檔案變更沒有反映在瀏覽器中，您可能還需要設定 Vite 的 [server.watch.usePolling 選項](https://vitejs.dev/config/server-options.html#server-watch)。


<a name="loading-your-scripts-and-styles"></a>
### 載入您的腳本與樣式

設定好 Vite 進入點後，您現在可以在新增至應用程式根模板 `<head>` 中的 `@vite()` Blade 指令中引用它們：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
```

若您是透過 JavaScript 匯入 CSS，則只需要包含 JavaScript 進入點即可：

```blade
<!DOCTYPE html>
<head>
    {{-- ... --}}

    @vite('resources/js/app.js')
</head>
```

`@vite` 指令會自動偵測 Vite 開發伺服器並注入 Vite 客戶端以啟用熱模組替換 (Hot Module Replacement)。在打包模式下，該指令將載入編譯後並加上版本號的靜態資源，包括任何已匯入的 CSS。

如果需要，您也可以在呼叫 `@vite` 指令時指定已編譯靜態資源的打包路徑：

```blade
<!doctype html>
<head>
    {{-- Given build path is relative to public path. --}}

    @vite('resources/js/app.js', 'vendor/courier/build')
</head>
```


<a name="inline-assets"></a>
#### 行內靜態資源

有時可能需要包含靜態資源的原始內容，而不是連結到帶有版本號的靜態資源 URL。例如，當將 HTML 內容傳送到 PDF 產生器時，您可能需要將靜態資源內容直接包含在頁面中。您可以使用 `Vite` Facade 所提供的 `content` 方法來輸出 Vite 靜態資源的內容：

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

有兩種方式可以執行 Vite。您可以透過 `dev` 指令執行開發伺服器，這在本地端開發時非常有用。開發伺服器會自動偵測檔案的變更，並即時反應在任何已開啟的瀏覽器視窗中。

或者，執行 `build` 指令會為您應用程式的靜態資源加入版本號並進行打包，讓您可以準備部署到正式環境：

```shell
# Run the Vite development server...
npm run dev

# Build and version the assets for production...
npm run build
```

如果您是在 WSL2 上使用 [Sail](/docs/{{version}}/sail) 執行開發伺服器，可能需要一些[額外的設定](#configuring-hmr-in-sail-on-wsl2)選項。


<a name="working-with-scripts"></a>
## 使用 JavaScript


<a name="aliases"></a>
### 別名

預設情況下，Laravel 外掛提供了一個常用的別名，以協助您快速上手並便利地匯入應用程式的靜態資源：

```js
{
    '@' => '/resources/js'
}
```

您可以透過在 `vite.config.js` 設定檔中新增您自己的別名來覆寫 `'@'` 別名：

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

如果您想使用 [Vue](https://vuejs.org/) 框架來建立前端，則還需要安裝 `@vitejs/plugin-vue` 外掛：

```shell
npm install --save-dev @vitejs/plugin-vue
```

接著您可以在 `vite.config.js` 設定檔中包含該外掛。當在 Laravel 中搭配使用 Vue 外掛時，您會需要一些額外的選項：

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
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、Vue 與 Vite 設定。這些入門套件是開始使用 Laravel、Vue 與 Vite 的最快方式。


<a name="react"></a>
### React

如果您想使用 [React](https://reactjs.org/) 框架來建立前端，則還需要安裝 `@vitejs/plugin-react` 外掛：

```shell
npm install --save-dev @vitejs/plugin-react
```

接著您可以在 `vite.config.js` 設定檔中包含該外掛：

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

您必須確保任何包含 JSX 的檔案都有 `.jsx` 或 `.tsx` 副檔名，並記得視需要依[上述說明](#configuring-vite)更新您的入口點。

您還需要在使用現有的 `@vite` 指令之外，包含額外的 `@viteReactRefresh` Blade 指令。

```blade
@viteReactRefresh
@vite('resources/js/app.jsx')
```

`@viteReactRefresh` 指令必須在 `@vite` 指令之前被呼叫。

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、React 與 Vite 設定。這些入門套件是開始使用 Laravel、React 與 Vite 的最快方式。


<a name="svelte"></a>
### Svelte

如果您想使用 [Svelte](https://svelte.dev/) 框架來建立前端，則還需要安裝 `@sveltejs/vite-plugin-svelte` 外掛：

```shell
npm install --save-dev @sveltejs/vite-plugin-svelte
```

接著您可以在 `vite.config.js` 設定檔中包含該外掛。

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
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、Svelte 與 Vite 設定。這些入門套件是開始使用 Laravel、Svelte 與 Vite 的最快方式。


<a name="inertia"></a>
### Inertia

Laravel Vite 外掛提供了便利的 `resolvePageComponent` 函式，可幫助您解析 Inertia 頁面元件。以下是在 Vue 3 中使用此輔助函式的範例；不過，您也可以在 React 或 Svelte 等其他框架中利用此函式：

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

如果您搭配 Inertia 使用 Vite 的程式碼拆分 (Code Splitting) 功能，建議設定[靜態資源預取](#asset-prefetching)。

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了適當的 Laravel、Inertia 與 Vite 設定。這些入門套件是開始使用 Laravel、Inertia 與 Vite 的最快方式。


<a name="url-processing"></a>
### URL 處理

當使用 Vite 並在應用程式的 HTML、CSS 或 JS 中引用靜態資源時，需要注意幾個注意事項。首先，如果您使用絕對路徑引用靜態資源，Vite 將不會在打包時包含該靜態資源；因此，您應該確保該靜態資源存在於您的 public 目錄中。使用[獨立 CSS 入口點](#configuring-vite)時應避免使用絕對路徑，因為在開發期間，瀏覽器會嘗試從託管 CSS 的 Vite 開發伺服器載入這些路徑，而不是從您的 public 目錄載入。

當引用相對靜態資源路徑時，您應該記住這些路徑是相對於引用它們的檔案。任何透過相對路徑引用的靜態資源都將被 Vite 重寫、加入版本號並進行打包。

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

以下範例示範 Vite 如何處理相對與絕對 URL：

```html
<!-- This asset is not handled by Vite and will not be included in the build -->
<img src="/taylor.png">

<!-- This asset will be re-written, versioned, and bundled by Vite -->
<img src="../../images/abigail.png">
```

<a name="working-with-stylesheets"></a>
## 使用樣式表

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含了正確的 Tailwind 和 Vite 設定。或者，如果您想在不使用入門套件的情況下使用 Tailwind 與 Laravel，請參考 [Tailwind 的 Laravel 安裝指南](https://tailwindcss.com/docs/guides/laravel)。

所有的 Laravel 應用程式都已包含 Tailwind 及適當設定的 `vite.config.js` 檔案。因此，您只需要啟動 Vite 開發伺服器或執行 `dev` Composer 指令，該指令會同時啟動 Laravel 和 Vite 開發伺服器：

```shell
composer run dev
```

您應用程式的 CSS 可以放在 `resources/css/app.css` 檔案中。

<a name="working-with-fonts"></a>
## 使用字型

Laravel Vite 外掛可以為您的應用程式提供最佳化且自行代管（Self-hosted）的字型。當設定好字型後，外掛會解析請求的字型檔案、將其匯出為 Vite 靜態資源、產生字型 CSS，並寫入可供 Blade 的 [`@fonts` 指令](/docs/{{version}}/blade#fonts)讀取的字型資訊清單（Manifest）。

若要設定字型，請從 `laravel-vite-plugin/fonts` 匯入一個或多個提供者輔助函式，並將它們新增至 Laravel 外掛的 `fonts` 選項中：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { google } from 'laravel-vite-plugin/fonts';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            fonts: [
                google('Inter', {
                    alias: 'sans',
                    weights: [400, 500, 600, 700],
                    styles: ['normal', 'italic'],
                    subsets: ['latin'],
                    display: 'swap',
                    preload: [
                        { weight: 400 },
                        { weight: 700 },
                    ],
                    fallbacks: ['system-ui', 'sans-serif'],
                }),
            ],
        }),
    ],
});
```

在此範例中，`Inter` 字型將可透過 `sans` 別名使用。外掛會產生一個 `--font-sans` CSS 變數以及一個用來套用已產生字型組合（Font stack）的 `.font-sans` 工具類別（Utility class）。

<a name="font-providers"></a>
### 字型提供者

Laravel Vite 外掛包含適用於 Google Fonts、Bunny Fonts、Fontsource 及本地字型的提供者輔助函式：

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { bunny, fontsource, google, local } from 'laravel-vite-plugin/fonts';

export default defineConfig({
    plugins: [
        laravel({
            input: 'resources/js/app.js',
            fonts: [
                google('Inter', { alias: 'sans' }),
                bunny('Figtree', { alias: 'body' }),
                fontsource('JetBrains Mono', { alias: 'mono' }),
                local('Brand Sans', {
                    alias: 'brand',
                    src: 'resources/fonts/brand-sans',
                }),
            ],
        }),
    ],
});
```

`fontsource` 提供者會從已安裝的 Fontsource 套件中讀取字型。預設情況下，套件名稱是從字型家族名稱衍生而來，例如 `@fontsource/jetbrains-mono`。如果您的應用程式使用不同的套件名稱，可以使用 `package` 選項來指定。

<a name="local-fonts"></a>
### 本地字型

使用本地字型時，`src` 選項可以指向單一字型檔案、目錄或 Glob 樣式。外掛會自動偵測支援的字型檔案，並從其檔名推斷其字型粗細（Weight）與樣式（Style）：

```js
local('Brand Sans', {
    alias: 'brand',
    src: 'resources/fonts/brand-sans/*.woff2',
})
```

如果您需要完整控制可用的變體（Variants），可以使用 `variants` 選項來明確定義它們：

```js
local('Brand Sans', {
    alias: 'brand',
    variants: [
        { src: 'resources/fonts/BrandSans-Regular.woff2', weight: 400 },
        { src: 'resources/fonts/BrandSans-Italic.woff2', weight: 400, style: 'italic' },
        { src: ['resources/fonts/BrandSans-Bold.woff2', 'resources/fonts/BrandSans-Bold.ttf'], weight: 700 },
    ],
})
```

<a name="font-options"></a>
### 字型選項

根據提供者的不同，字型定義可接受多個選項，讓您可以自訂所產生的字型 CSS：

<div class="content-list" markdown="1">

- `alias` 定義 Blade `@fonts` 指令所使用的名稱，預設為字型家族名稱轉成的 Slug。
- `variable` 定義產生的 CSS 變數，預設為 `--font-{alias}`。
- `weights` 定義應該解析的遠端或 Fontsource 字型粗細，預設為 `[400]`。
- `styles` 定義應該解析的遠端或 Fontsource 字型樣式，預設為 `['normal']`。
- `subsets` 定義應該解析的遠端或 Fontsource 字型子集，預設為 `['latin']`。
- `display` 定義 `font-display` 的值，預設為 `swap`。
- `preload` 控制哪些 WOFF2 字型變體應該被預載。此選項可以是 `true`、`false` 或 `{ weight, style }` 選擇器的陣列。
- `fallbacks` 定義應附加到產生的字型組合末尾的額外後備字型。
- `optimizedFallbacks` 嘗試使用可選的 `fontaine` 套件來產生調整過度量參數（Metric-adjusted）的後備字型 Face，預設為 `true`。

</div>

最佳化後備字型需要 `fontaine` 套件，該套件預設未安裝。如果您希望 Laravel 產生調整過度量參數的後備字型 Face，您應該將 `fontaine` 安裝為開發相依套件：

```shell
npm install --save-dev fontaine
```

如果未安裝 `fontaine` 或無法讀取字型檔案，Laravel 將會跳過該字型最佳化的後備設定，並繼續使用任何透過 `fallbacks` 選項設定的字型。

本地字型是從上述的 `src` 或 `variants` 選項中解析，而不是使用 `weights`、`styles` 和 `subsets`。

<a name="working-with-blade-and-routes"></a>
## 使用 Blade 與路由


<a name="blade-processing-static-assets"></a>
### 使用 Vite 處理靜態資源

當在 JavaScript 或 CSS 中引用靜態資源時，Vite 會自動處理並為其加入版本號。此外，當構建基於 Blade 的應用程式時，Vite 也可以處理並為您僅在 Blade 模板中引用的靜態資源加入版本號。

然而，為了實現這一點，您需要透過在外掛的 `assets` 選項中指定靜態資源，讓 Vite 能夠識別它們。這個選項適用於您想要直接使用 `Vite::asset` 引用的靜態檔案。如果您希望 Laravel 產生字型 CSS 與預載連結，請改用 [`fonts` 選項](#working-with-fonts)。

例如，如果您想處理並為儲存在 `resources/images` 中的所有圖片以及儲存在 `resources/fonts` 中的所有字型加入版本號，您應該在 Vite 設定中新增以下內容：

```js
laravel({
    input: 'resources/js/app.js',
    assets: ['resources/images/**', 'resources/fonts/**'],
})
```

執行 `npm run build` 時，這些靜態資源現在將由 Vite 進行處理。接著，您可以透過 `Vite::asset` 方法在 Blade 模板中引用這些靜態資源，該方法將會傳回給定靜態資源帶有版本號的 URL：

```blade
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```

> [!NOTE]
> 在 Laravel Vite 外掛第 3 版之前，靜態資源必須使用 `import.meta.glob` 在應用程式的進入點中匯入。`assets` 選項是因應 Vite 8 的變更而引進的。


<a name="blade-refreshing-on-save"></a>
### 儲存時自動重新整理

當您的應用程式是使用傳統的 Blade 伺服器端渲染建構時，Vite 可以在您對應用程式中的視圖檔案進行變更時自動重新整理瀏覽器，從而改善您的開發工作流程。若要開始使用，您只需將 `refresh` 選項指定為 `true` 即可。

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

當 `refresh` 選項為 `true` 時，當您執行 `npm run dev` 並儲存以下目錄中的檔案時，將會觸發瀏覽器進行全頁重新整理：

- `app/Livewire/**`
- `app/View/Components/**`
- `lang/**`
- `resources/lang/**`
- `resources/views/**`
- `routes/**`

如果您利用 [Ziggy](https://github.com/tighten/ziggy) 在應用程式的前端產生路由連結，監視 `routes/**` 目錄會非常有用。

如果這些預設路徑不符合您的需求，您可以指定您自己要監視的路徑清單：

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

在底層，Laravel Vite 外掛使用了 [vite-plugin-full-reload](https://github.com/ElMassimo/vite-plugin-full-reload) 套件，它提供了一些進階設定選項來微調此功能的行為。如果您需要這種程度的自訂，可以提供 `config` 定義：

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

在 JavaScript 應用程式中，為經常引用的目錄[建立別名](#aliases)是很常見的做法。但是，您也可以透過使用 `Illuminate\Support\Facades\Vite` 類別上的 `macro` 方法來建立要在 Blade 中使用的別名。通常，「巨集 (macros)」應該要在[服務提供者 (service provider)](/docs/{{version}}/providers) 的 `boot` 方法中定義：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::macro('image', fn (string $asset) => $this->asset("resources/images/{$asset}"));
}
```

巨集定義完成後，就可以在您的模板中呼叫它。例如，我們可以使用上面定義的 `image` 巨集來引用位於 `resources/images/logo.png` 的靜態資源：

```blade
<img src="{{ Vite::image('logo.png') }}" alt="Laravel Logo">
```


<a name="asset-prefetching"></a>
## 靜態資源預取

當使用 Vite 的程式碼分割 (code splitting) 功能建構 SPA 時，每當進行頁面導覽，需要的靜態資源都會被抓取。這種行為可能會導致 UI 渲染延遲。如果您選擇的前端框架遇到這個問題，Laravel 提供了在初始頁面載入時積極預取應用程式的 JavaScript 與 CSS 靜態資源的功能。

您可以透過在[服務提供者 (service provider)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `Vite::prefetch` 方法，來指示 Laravel 積極預取您的靜態資源：

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

在上述範例中，每次頁面載入時，靜態資源將以最多 `3` 個同時下載進行預取。您可以修改併發數量以符合應用程式的需求，或者如果不限制併發數量並讓應用程式一次下載所有靜態資源，則不需指定限制：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch();
}
```

預設情況下，預取會在 [頁面 _load_ 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) 觸發時開始。如果您想自訂預取開始的時間，可以指定一個 Vite 監聽的事件：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Vite::prefetch(event: 'vite:prefetch');
}
```

根據上述程式碼，當您在 `window` 物件上手動發送 `vite:prefetch` 事件時，預取就會開始。例如，您可以讓預取在頁面載入後三秒開始：

```html
<script>
    addEventListener('load', () => setTimeout(() => {
        dispatchEvent(new Event('vite:prefetch'))
    }, 3000))
</script>
```


<a name="custom-base-urls"></a>
## 自訂基底 URL

如果您經由 Vite 編譯的靜態資源部署到與應用程式不同的網域（例如透過 CDN），則必須在應用程式的 `.env` 檔案中指定 `ASSET_URL` 環境變數：

```env
ASSET_URL=https://cdn.example.com
```

設定靜態資源 URL 後，所有改寫後的靜態資源 URL 都會加上所設定的值作為字首：

```text
https://cdn.example.com/build/assets/app.9dce8d17.js
```

請記住，[絕對路徑 URL 不會被 Vite 改寫](#url-processing)，因此它們不會加上字首。


<a name="environment-variables"></a>
## 環境變數

您可以透過在應用程式的 `.env` 檔案中為環境變數加上 `VITE_` 前綴，將其注入到 JavaScript 中：

```env
VITE_SENTRY_DSN_PUBLIC=http://example.com
```

您可以透過 `import.meta.env` 物件存取注入的環境變數：

```js
import.meta.env.VITE_SENTRY_DSN_PUBLIC
```

<a name="disabling-vite-in-tests"></a>
## 在測試中停用 Vite

Laravel 的 Vite 整合會在執行測試時嘗試解析您的靜態資源，這需要您執行 Vite 開發伺服器或建置您的靜態資源。

若您希望在測試期間模擬 (mock) Vite，可以呼叫 `withoutVite` 方法，該方法適用於任何繼承 Laravel `TestCase` 類別的測試：

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

如果您想為所有測試停用 Vite，可以在基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutVite` 方法：

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

Laravel Vite 外掛讓您能輕鬆設定 Vite 的伺服器端渲染 (SSR)。首先，在 `resources/js/ssr.js` 建立一個 SSR 入口點，並透過傳遞設定選項給 Laravel 外掛來指定該入口點：

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

為確保您不會忘記重新建置 SSR 入口點，建議擴充應用程式 `package.json` 中的 "build" 腳本來建立 SSR 建置檔：

```json
"scripts": {
     "dev": "vite",
     "build": "vite build" // [tl! remove]
     "build": "vite build && vite build --ssr" // [tl! add]
}
```

接著，若要建置並啟動 SSR 伺服器，您可以執行以下指令：

```shell
npm run build
node bootstrap/ssr/ssr.js
```

如果您正在搭配使用 [Inertia 的 SSR](https://inertiajs.com/server-side-rendering)，則可以使用 `inertia:start-ssr` Artisan 指令來啟動 SSR 伺服器：

```shell
php artisan inertia:start-ssr
```

> [!NOTE]
> Laravel 的[入門套件](/docs/{{version}}/starter-kits)已經包含適當的 Laravel、Inertia SSR 與 Vite 設定。這些入門套件是開始使用 Laravel、Inertia SSR 與 Vite 的最快方式。


<a name="script-and-style-attributes"></a>
## Script 與 Style 標籤屬性


<a name="content-security-policy-csp-nonce"></a>
### 內容安全策略 (CSP) Nonce

若您希望在 script 與 style 標籤上包含 [nonce 屬性](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/nonce)作為[內容安全策略 (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) 的一部分，可以在自訂的[中介層](/docs/{{version}}/middleware)中使用 `useCspNonce` 方法來產生或指定 nonce：

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

呼叫 `useCspNonce` 方法後，Laravel 將會自動在所有產生的 script 與 style 標籤上包含 `nonce` 屬性。

如果您需要在其他地方指定 nonce，包含 Laravel [入門套件](/docs/{{version}}/starter-kits)中隨附的 [Ziggy `@route` 指令](https://github.com/tighten/ziggy#using-routes-with-a-content-security-policy)，您可以使用 `cspNonce` 方法來取得它：

```blade
@routes(nonce: Vite::cspNonce())
```

如果您已經有一個想要指示 Laravel 使用的 nonce，可以將該 nonce 傳遞給 `useCspNonce` 方法：

```php
Vite::useCspNonce($nonce);
```


<a name="subresource-integrity-sri"></a>
### 子資源完整性 (SRI)

如果您的 Vite 清單 (manifest) 包含靜態資源的 `integrity` 雜湊值，Laravel 將會在產生的所有 script 與 style 標籤上自動新增 `integrity` 屬性，以強制執行[子資源完整性 (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)。預設情況下，Vite 不會在其清單中包含 `integrity` 雜湊值，但您可以透過安裝 [vite-plugin-manifest-sri](https://www.npmjs.com/package/vite-plugin-manifest-sri) NPM 外掛來啟用它：

```shell
npm install --save-dev vite-plugin-manifest-sri
```

接著，您可以在 `vite.config.js` 檔案中啟用此外掛：

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

如有需要，您還可以自訂用來尋找完整性雜湊值的清單鍵名 (manifest key)：

```php
use Illuminate\Support\Facades\Vite;

Vite::useIntegrityKey('custom-integrity-key');
```

如果您想完全停用此自動偵測功能，可以傳遞 `false` 給 `useIntegrityKey` 方法：

```php
Vite::useIntegrityKey(false);
```


<a name="arbitrary-attributes"></a>
### 任意屬性

如果需要在 script 與 style 標籤上包含額外的屬性，例如 [data-turbo-track](https://turbo.hotwired.dev/handbook/drive#reloading-when-assets-change) 屬性，可以透過 `useScriptTagAttributes` 與 `useStyleTagAttributes` 方法來指定它們。通常，這些方法應該在[服務提供者(Service Providers)](/docs/{{version}}/providers)中呼叫：

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

如果需要有條件地新增屬性，您可以傳遞一個回呼函式 (callback)，該函式將接收靜態資源的來源路徑、其 URL、其清單區塊 (manifest chunk) 以及整個清單：

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
> 在 Vite 開發伺服器執行期間，`$chunk` 與 `$manifest` 引數將會是 `null`。

<a name="advanced-customization"></a>
## 進階自訂

開箱即用，Laravel 的 Vite 外掛採用了適合大多數應用程式的合理慣例；不過，有時您可能需要自訂 Vite 的行為。為了支援額外自訂選項，我們提供以下方法與選項，可用來替代 `@vite` Blade 指令：

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

在 `vite.config.js` 檔案中，您接著應該指定相同的設定：

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

若您在從 Vite 開發伺服器取得靜態資源時於瀏覽器遇到跨來源資源共享 (CORS) 問題，您可能需要授權您的自訂來源存取開發伺服器。Vite 結合 Laravel 外掛後，在無需任何額外設定的情況下預設允許以下來源：

- `::1`
- `127.0.0.1`
- `localhost`
- `*.test`
- `*.localhost`
- 專案 `.env` 檔案中的 `APP_URL`

為專案允許自訂來源的最簡單方法，是確保您應用程式的 `APP_URL` 環境變數與您在瀏覽器中造訪的來源一致。例如，如果您造訪 `https://my-app.laravel`，您應該更新 `.env` 以配合該網址：

```env
APP_URL=https://my-app.laravel
```

如果您需要更細粒度地控制來源（例如支援多個來源），您應該使用 [Vite 完整且彈性的內建 CORS 伺服器設定](https://vite.dev/config/server-options.html#server-cors)。例如，您可以在專案的 `vite.config.js` 檔案中的 `server.cors.origin` 設定選項裡指定多個來源：

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

您也可以包含正規表示式（Regex）模式，這在您想要允許指定頂級網域（例如 `*.laravel`）的所有來源時非常有用：

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

Vite 生態系統中的某些外掛會假設以正斜線開頭的 URL 總是指向 Vite 開發伺服器。然而，由於 Laravel 整合的特性，實際情況並非總是如此。

例如，`vite-imagetools` 外掛在 Vite 提供靜態資源時會輸出如下的 URL：

```html
<img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520">
```

`vite-imagetools` 外掛預期輸出的 URL 會被 Vite 攔截，然後該外掛就能處理所有以 `/@imagetools` 開頭的 URL。如果您正使用的外掛預期此種行為，您將需要手動修正這些 URL。您可以在 `vite.config.js` 檔案中使用 `transformOnServe` 選項來達成此目的。

在此特定範例中，我們將在產生的程式碼內所有出現的 `/@imagetools` 前面加上開發伺服器 URL：

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

現在，當 Vite 提供靜態資源時，它將輸出指向 Vite 開發伺服器的 URL：

```html
- <img src="/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! remove] -->
+ <img src="http://[::1]:5173/@imagetools/f0b2f404b13f052c604e632f2fb60381bf61a520"><!-- [tl! add] -->
```