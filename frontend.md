# 前端

- [簡介](#introduction)
- [使用 PHP](#using-php)
    - [PHP 與 Blade](#php-and-blade)
    - [Livewire](#livewire)
    - [入門套件](#php-starter-kits)
- [使用 Vue / React](#using-vue-react)
    - [Inertia](#inertia)
    - [入門套件](#inertia-starter-kits)
- [打包前端資源](#bundling-assets)

<a name="introduction"></a>
## 簡介

Laravel 是一個後端框架，提供了建構現代化網路應用程式所需的所有功能，例如 [路由](/docs/{{version}}/routing)、[驗證](/docs/{{version}}/validation)、[快取](/docs/{{version}}/cache)、[佇列](/docs/{{version}}/queues)、[檔案儲存](/docs/{{version}}/filesystem)等等。然而，我們相信為開發者提供一個優美的全端開發體驗至關重要，這包括建構應用程式前端的強大方法。

使用 Laravel 建構應用程式時，前端開發主要有兩種方式，而您選擇哪種方式取決於您是想利用 PHP 來建構前端，還是使用 Vue 和 React 等 JavaScript 框架。我們將在下方討論這兩種選項，以便您能針對應用程式的前端開發做出明智的決定。

<a name="using-php"></a>
## 使用 PHP

<a name="php-and-blade"></a>
### PHP 與 Blade

過去，大多數 PHP 應用程式會使用簡單的 HTML 模板，並穿插 PHP `echo` 語句來將請求期間從資料庫中擷取的資料渲染到瀏覽器：

```blade
<div>
    <?php foreach ($users as $user): ?>
        Hello, <?php echo $user->name; ?> <br />
    <?php endforeach; ?>
</div>
```

在 Laravel 中，這種渲染 HTML 的方法仍然可以透過 [視圖](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 來實現。Blade 是一種極其輕量級的模板語言，它提供了方便、簡潔的語法來顯示資料、迭代資料等等：

```blade
<div>
    @foreach ($users as $user)
        Hello, {{ $user->name }} <br />
    @endforeach
</div>
```

以這種方式建構應用程式時，表單提交和其他頁面互動通常會從伺服器接收一個全新的 HTML 文件，並且整個頁面會由瀏覽器重新渲染。即使在今天，許多應用程式可能仍然非常適合使用簡單的 Blade 模板來建構其前端。

<a name="growing-expectations"></a>
#### 日益增長的期望

然而，隨著使用者對網路應用程式的期望日趨成熟，許多開發者發現需要建構更具動態性、互動性更精緻的前端。有鑑於此，一些開發者選擇開始使用 Vue 和 React 等 JavaScript 框架來建構應用程式的前端。

另一些開發者則偏好繼續使用他們熟悉的後端語言，並開發出解決方案，讓他們能夠在主要使用所選後端語言的情況下，建構現代化的網路應用程式 UI。例如，在 [Rails](https://rubyonrails.org/) 生態系統中，這促使了 [Turbo](https://turbo.hotwired.dev/)、[Hotwire](https://hotwired.dev/) 和 [Stimulus](https://stimulus.hotwired.dev/) 等函式庫的誕生。

在 Laravel 生態系統中，主要透過 PHP 建立現代化、動態前端的需求，促成了 [Laravel Livewire](https://livewire.laravel.com) 和 [Alpine.js](https://alpinejs.dev/) 的誕生。

<a name="livewire"></a>
### Livewire

[Laravel Livewire](https://livewire.laravel.com) 是一個用於建構由 Laravel 驅動的前端框架，它能像使用 Vue 和 React 等現代 JavaScript 框架建構的前端一樣，提供動態、現代且生動的體驗。

使用 Livewire 時，您將建立 Livewire「元件」，這些元件會渲染 UI 的離散部分，並公開可從應用程式前端呼叫和互動的方法與資料。例如，一個簡單的「計數器」元件可能如下所示：

```php
<?php

namespace App\Http\Livewire;

use Livewire\Component;

class Counter extends Component
{
    public $count = 0;

    public function increment()
    {
        $this->count++;
    }

    public function render()
    {
        return view('livewire.counter');
    }
}
```

而計數器對應的模板將會這樣編寫：

```blade
<div>
    <button wire:click="increment">+</button>
    <h1>{{ $count }}</h1>
</div>
```

如您所見，Livewire 讓您能夠編寫新的 HTML 屬性，例如 `wire:click`，這些屬性將您的 Laravel 應用程式前端與後端連接起來。此外，您可以使用簡單的 Blade 表達式來渲染元件的當前狀態。

對許多人來說，Livewire 革新了 Laravel 的前端開發，讓他們在建構現代化、動態網路應用程式時，仍能安於 Laravel 的舒適圈。通常，使用 Livewire 的開發者也會利用 [Alpine.js](https://alpinejs.dev/)，僅在需要時才將 JavaScript「點綴」到他們的前端，例如為了渲染一個對話視窗。

如果您是 Laravel 的新手，我們建議您先熟悉 [視圖](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 的基本用法。然後，查閱官方的 [Laravel Livewire 說明文件](https://livewire.laravel.com/docs)，了解如何透過互動式 Livewire 元件將您的應用程式提升到新的層次。

<a name="php-starter-kits"></a>
### PHP 入門套件

如果您想使用 PHP 和 Livewire 來建構前端，可以利用我們的 Breeze 或 Jetstream [入門套件](/docs/{{version}}/starter-kits) 來快速啟動應用程式的開發。這兩個入門套件都使用 [Blade](/docs/{{version}}/blade) 和 [Tailwind](https://tailwindcss.com) 來搭建應用程式的後端和前端身份驗證流程，讓您能夠直接開始建構您的下一個偉大構想。

<a name="using-vue-react"></a>
## 使用 Vue / React

儘管使用 Laravel 和 Livewire 可以建構現代化的前端，但許多開發者仍然偏好利用 Vue 或 React 等 JavaScript 框架的強大功能。這讓開發者能夠利用透過 NPM 提供的豐富 JavaScript 套件和工具生態系統。

然而，如果沒有額外的工具，將 Laravel 與 Vue 或 React 配對將會讓我們需要解決各種複雜的問題，例如客戶端路由、資料水合 (data hydration) 和身份驗證。客戶端路由通常透過使用像 [Nuxt](https://nuxt.com/) 和 [Next](https://nextjs.org/) 這樣有主見的 Vue / React 框架來簡化；然而，當將 Laravel 這樣的後端框架與這些前端框架配對時，資料水合和身份驗證仍然是複雜且繁瑣的問題。

此外，開發者還需要維護兩個獨立的程式碼儲存庫，通常需要協調兩個儲存庫的維護、發布和部署。儘管這些問題並非無法克服，但我們不認為這是一種高效或愉快的應用程式開發方式。

<a name="inertia"></a>
### Inertia

幸運的是，Laravel 提供了兩全其美的方案。[Inertia](https://inertiajs.com) 彌合了您的 Laravel 應用程式與現代 Vue 或 React 前端之間的鴻溝，讓您能夠使用 Vue 或 React 建構功能齊全的現代前端，同時利用 Laravel 的路由和控制器進行路由、資料水合和身份驗證 — 所有這些都在一個程式碼儲存庫中完成。透過這種方法，您可以充分享受 Laravel 和 Vue / React 的強大功能，而不會削弱任何一個工具的能力。

在您的 Laravel 應用程式中安裝 Inertia 後，您將像往常一樣編寫路由和控制器。然而，您的控制器將不再返回 Blade 模板，而是返回一個 Inertia 頁面：

```php
<?php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Models\User;
use Inertia\Inertia;
use Inertia\Response;

class UserController extends Controller
{
    /**
     * Show the profile for a given user.
     */
    public function show(string $id): Response
    {
        return Inertia::render('Users/Profile', [
            'user' => User::findOrFail($id)
        ]);
    }
}
```

一個 Inertia 頁面對應一個 Vue 或 React 元件，通常儲存在應用程式的 `resources/js/Pages` 目錄中。透過 `Inertia::render` 方法提供給頁面的資料將用於水合頁面元件的「props」：

```vue
<script setup>
import Layout from ' @/Layouts/Authenticated.vue';
import { Head } from ' @inertiajs/vue3';

const props = defineProps(['user']);
</script>

<template>
    <Head title="User Profile" />

    <Layout>
        <template #header>
            <h2 class="font-semibold text-xl text-gray-800 leading-tight">
                Profile
            </h2>
        </template>

        <div class="py-12">
            Hello, {{ user.name }}
        </div>
    </Layout>
</template>
```

如您所見，Inertia 讓您在建構前端時能夠充分利用 Vue 或 React 的強大功能，同時在由 Laravel 驅動的後端與由 JavaScript 驅動的前端之間提供輕量級的橋樑。

#### 伺服器端渲染

如果您擔心因為應用程式需要伺服器端渲染而不敢深入 Inertia，請不用擔心。Inertia 提供了 [伺服器端渲染支援](https://inertiajs.com/server-side-rendering)。而且，當透過 [Laravel Forge](https://forge.laravel.com) 部署您的應用程式時，確保 Inertia 的伺服器端渲染程序始終運行是輕而易舉的事。

<a name="inertia-starter-kits"></a>
### 入門套件

如果您想使用 Inertia 和 Vue / React 來建構前端，可以利用我們的 Breeze 或 Jetstream [入門套件](/docs/{{version}}/starter-kits#breeze-and-inertia) 來快速啟動應用程式的開發。這兩個入門套件都使用 Inertia、Vue / React、[Tailwind](https://tailwindcss.com) 和 [Vite](https://vitejs.dev) 來搭建應用程式的後端和前端身份驗證流程，讓您能夠直接開始建構您的下一個偉大構想。

<a name="bundling-assets"></a>
## 打包前端資源

無論您選擇使用 Blade 和 Livewire，還是 Vue / React 和 Inertia 來開發前端，您都可能需要將應用程式的 CSS 打包成可供生產環境使用的資源。當然，如果您選擇使用 Vue 或 React 來建構應用程式的前端，您也需要將元件打包成瀏覽器可用的 JavaScript 資源。

預設情況下，Laravel 利用 [Vite](https://vitejs.dev) 來打包您的資源。Vite 提供了閃電般的建構時間和在本地開發期間近乎即時的 Hot Module Replacement (HMR)。在所有新的 Laravel 應用程式中，包括使用我們 [入門套件](/docs/{{version}}/starter-kits) 的應用程式，您都會找到一個 `vite.config.js` 檔案，該檔案載入了我們輕量級的 Laravel Vite plugin，讓 Vite 在 Laravel 應用程式中變得易於使用。

開始使用 Laravel 和 Vite 最快的方法是從 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze) 開始應用程式的開發，這是我們最簡單的入門套件，透過提供前端和後端身份驗證腳手架來快速啟動您的應用程式。

> [!NOTE]  
> 有關在 Laravel 中使用 Vite 的更詳細說明文件，請參閱我們 [關於打包和編譯資源的專門說明文件](/docs/{{version}}/vite)。
