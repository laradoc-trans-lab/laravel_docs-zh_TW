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

Laravel 是一個後端框架，提供了建構現代網路應用程式所需的所有功能，例如[路由](/docs/{{version}}/routing)、[驗證](/docs/{{version}}/validation)、[快取](/docs/{{version}}/cache)、[佇列](/docs/{{version}}/queues)、[檔案儲存](/docs/{{version}}/filesystem)等等。然而，我們認為為開發人員提供優美的全端體驗非常重要，這包括建構應用程式前端的強大方法。

在使用 Laravel 建構應用程式時，處理前端開發有兩種主要方式，您選擇哪種方式取決於您是希望透過利用 PHP 建構前端，還是使用 Vue 和 React 等 JavaScript 框架。我們將在下方討論這兩種選項，以便您可以就應用程式前端開發的最佳方法做出明智的決定。

<a name="using-php"></a>
## 使用 PHP

<a name="php-and-blade"></a>
### PHP 與 Blade

過去，大多數 PHP 應用程式透過簡單的 HTML 模板，並穿插著 PHP `echo` 語句，將 HTML 渲染到瀏覽器，這些語句會印出在請求期間從資料庫中檢索到的資料：

```blade
<div>
    <?php foreach ($users as $user): ?>
        Hello, <?php echo $user->name; ?> <br />
    <?php endforeach; ?>
</div>
```

在 Laravel 中，這種渲染 HTML 的方式仍然可以使用[視圖](/docs/{{version}}/views)和 [Blade](/docs/{{version}}/blade) 來實現。Blade 是一種極其輕量的模板語言，它提供了方便、簡潔的語法來顯示資料、迭代資料等等：

```blade
<div>
    @foreach ($users as $user)
        Hello, {{ $user->name }} <br />
    @endforeach
</div>
```

以這種方式建構應用程式時，表單提交和其他頁面互動通常會從伺服器接收一份全新的 HTML 文件，並且整個頁面會由瀏覽器重新渲染。即使在今天，許多應用程式仍然非常適合使用簡單的 Blade 模板來建構他們的前端。

<a name="growing-expectations"></a>
#### 不斷增長的期望

然而，隨著使用者對網路應用程式的期望日趨成熟，許多開發人員發現需要建構更具動態性且感覺更精緻的互動式前端。有鑑於此，一些開發人員選擇開始使用 Vue 和 React 等 JavaScript 框架來建構他們的應用程式前端。

其他人，更傾向於堅持使用他們熟悉的後端語言，則開發了解決方案，這些解決方案允許建構現代網路應用程式的 UI，同時仍然主要利用他們選擇的後端語言。例如，在 [Rails](https://rubyonrails.org/) 生態系中，這促進了 [Turbo](https://turbo.hotwired.dev/) [Hotwire](https://hotwired.dev/) 和 [Stimulus](https://stimulus.hotwired.dev/) 等函式庫的誕生。

在 Laravel 生態系中，主要使用 PHP 建立現代、動態前端的需求，促成了 [Laravel Livewire](https://livewire.laravel.com) 和 [Alpine.js](https://alpinejs.dev/) 的誕生。

<a name="livewire"></a>
### Livewire

[Laravel Livewire](https://livewire.laravel.com) 是一個用於建構由 Laravel 驅動的前端的框架，這些前端感覺動態、現代且生動，就像使用 Vue 和 React 等現代 JavaScript 框架建構的前端一樣。

使用 Livewire 時，您將建立 Livewire「組件」，這些組件會渲染 UI 的獨立部分，並公開可以從應用程式前端調用和互動的方法和資料。例如，一個簡單的「計數器」組件可能如下所示：

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

而且，對應的計數器模板會這樣撰寫：

```blade
<div>
    <button wire:click="increment">+</button>
    <h1>{{ $count }}</h1>
</div>
```

如您所見，Livewire 讓您可以撰寫新的 HTML 屬性，例如 `wire:click`，這些屬性將您的 Laravel 應用程式前端和後端連接起來。此外，您可以使用簡單的 Blade 表達式來渲染組件的當前狀態。

對許多人來說，Livewire 徹底改變了 Laravel 的前端開發，讓他們能夠在 Laravel 的舒適區內，建構現代、動態的網路應用程式。通常，使用 Livewire 的開發人員也會利用 [Alpine.js](https://alpinejs.dev/) 在前端「點綴」JavaScript，僅在需要時使用，例如為了渲染對話視窗。

如果您是剛接觸 Laravel，我們建議您熟悉 [視圖](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 的基本用法。然後，查閱官方的 [Laravel Livewire 文件](https://livewire.laravel.com/docs) 以學習如何透過互動式 Livewire 組件讓您的應用程式更上一層樓。

<a name="php-starter-kits"></a>
### 入門套件

如果您想使用 PHP 和 Livewire 建構前端，您可以利用我們的 Breeze 或 Jetstream [入門套件](/docs/{{version}}/starter-kits) 來快速啟動您的應用程式開發。這兩個入門套件都使用 [Blade](/docs/{{version}}/blade) 和 [Tailwind](https://tailwindcss.com) 為您的應用程式後端和前端身份驗證流程搭建骨架，這樣您就可以輕鬆開始建構您的下一個偉大構想。

<a name="using-vue-react"></a>
## 使用 Vue / React

儘管可以使用 Laravel 和 Livewire 建立現代前端，但許多開發者仍偏好利用 Vue 或 React 等 JavaScript 框架的強大功能。這讓開發者可以善用透過 NPM 取得的 JavaScript 套件與工具的豐富生態系。

然而，若沒有額外工具，將 Laravel 與 Vue 或 React 搭配使用會使我們需要解決各種複雜的問題，例如客戶端路由、資料水合以及身份驗證。客戶端路由通常可以透過使用像 [Nuxt](https://nuxt.com/) 和 [Next](https://nextjs.org/) 這樣具主見的 Vue / React 框架來簡化；然而，當將 Laravel 等後端框架與這些前端框架搭配使用時，資料水合和身份驗證仍然是複雜且繁瑣的問題。

此外，開發者還需要維護兩個獨立的程式碼儲存庫，通常需要協調兩個儲存庫之間的維護、發布和部署。雖然這些問題並非不可克服，但我們認為這並不是一種高效或愉快的應用程式開發方式。

<a name="inertia"></a>
### Inertia

幸運的是，Laravel 提供了兩全其美的解決方案。[Inertia](https://inertiajs.com) 彌合了你的 Laravel 應用程式與現代 Vue 或 React 前端之間的差距，讓你能夠使用 Vue 或 React 建立功能齊全的現代前端，同時利用 Laravel 路由和控制器進行路由、資料水合以及身份驗證 — 所有這些都在單一程式碼儲存庫中完成。透過這種方法，你可以在不削弱任何工具功能的情況下，同時享受 Laravel 和 Vue / React 的全部強大功能。

在將 Inertia 安裝到你的 Laravel 應用程式後，你將一如往常地編寫路由和控制器。然而，你將不再從控制器中回傳 Blade 範本，而是回傳一個 Inertia 頁面：

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

Inertia 頁面對應於一個 Vue 或 React 元件，通常儲存於應用程式的 `resources/js/Pages` 目錄中。透過 `Inertia::render` 方法傳遞給頁面的資料將用於水合頁面組件的「props」：

```vue
<script setup>
import Layout from '@/Layouts/Authenticated.vue';
import { Head } from '@inertiajs/vue3';

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

如你所見，Inertia 讓你在建立前端時能夠利用 Vue 或 React 的全部強大功能，同時也為你由 Laravel 驅動的後端與由 JavaScript 驅動的前端之間提供了輕量級的橋樑。

#### 伺服器端渲染

如果你擔心由於應用程式需要伺服器端渲染而猶豫是否深入 Inertia，請不用擔心。Inertia 提供 [伺服器端渲染支援](https://inertiajs.com/server-side-rendering)。而且，當透過 [Laravel Forge](https://forge.laravel.com) 部署你的應用程式時，確保 Inertia 的伺服器端渲染程序始終運行輕而易舉。

<a name="inertia-starter-kits"></a>
### 入門套件

如果你想使用 Inertia 和 Vue / React 建立前端，你可以利用我們的 Breeze 或 Jetstream [入門套件](/docs/{{version}}/starter-kits#breeze-and-inertia) 來加速你應用程式的開發。這兩個入門套件都使用 Inertia、Vue / React、[Tailwind](https://tailwindcss.com) 和 [Vite](https://vitejs.dev) 為你應用程式的後端和前端身份驗證流程搭建骨架，讓你能夠開始構建下一個偉大創意。

<a name="bundling-assets"></a>
## 打包前端資源

無論你選擇使用 Blade 和 Livewire 還是 Vue / React 和 Inertia 來開發前端，你都可能需要將應用程式的 CSS 打包成可供生產使用的資源。當然，如果你選擇使用 Vue 或 React 建立應用程式的前端，你也將需要把你的組件打包成可供瀏覽器使用的 JavaScript 資源。

預設情況下，Laravel 利用 [Vite](https://vitejs.dev) 來打包你的資源。Vite 提供極速的建置時間，並在本地開發期間提供近乎即時的熱模組替換 (HMR)。在所有新的 Laravel 應用程式中，包括那些使用我們的 [入門套件](/docs/{{version}}/starter-kits) 的應用程式，你都會找到一個 `vite.config.js` 檔案，它載入我們輕量級的 Laravel Vite 插件，讓 Vite 與 Laravel 應用程式搭配使用起來輕而易舉。

開始使用 Laravel 和 Vite 的最快方式是使用 [Laravel Breeze](/docs/{{version}}/starter-kits#laravel-breeze) 來啟動你的應用程式開發，這是我們最簡單的入門套件，它透過提供前端和後端身份驗證骨架來加速你應用程式的啟動。

> [!NOTE]  
> 如需更詳細關於利用 Vite 搭配 Laravel 的文件，請參閱我們關於 [打包和編譯資源的專屬文件](/docs/{{version}}/vite)。