# 前端

- [簡介](#introduction)
- [使用 PHP](#using-php)
    - [PHP 與 Blade](#php-and-blade)
    - [Livewire](#livewire)
    - [入門套件](#php-starter-kits)
- [使用 React、Svelte 或 Vue](#using-react-svelte-or-vue)
    - [Inertia](#inertia)
    - [入門套件](#inertia-starter-kits)
- [靜態資源打包](#bundling-assets)

<a name="introduction"></a>
## 簡介

Laravel 是一個後端框架，提供了建構現代 Web 應用程式所需的所有功能，例如[路由](/docs/{{version}}/routing)、[驗證](/docs/{{version}}/validation)、[快取](/docs/{{version}}/cache)、[佇列](/docs/{{version}}/queues)、[檔案儲存](/docs/{{version}}/filesystem)等等。然而，我們認為為開發者提供美好的全端體驗也同樣重要，這包括了建構應用程式前端的強大方法。

在使用 Laravel 建構應用程式時，有兩種處理前端開發的主要方式，您選擇哪種方式取決於您是想利用 PHP 來建構前端，還是使用 JavaScript 框架（如 React、Svelte 和 Vue）。我們將在下方討論這兩種選項，以便您能針對應用程式的前端開發做出明智的決定。


<a name="using-php"></a>
## 使用 PHP


<a name="php-and-blade"></a>
### PHP 與 Blade

過去，大多數 PHP 應用程式使用簡單的 HTML 模板，並穿插著 PHP `echo` 語句來將 HTML 渲染到瀏覽器，這些語句會渲染在請求期間從資料庫中檢索到的資料：

```blade
<div>
    <?php foreach ($users as $user): ?>
        Hello, <?php echo $user->name; ?> <br />
    <?php endforeach; ?>
</div>
```

在 Laravel 中，這種渲染 HTML 的方法仍然可以透過 [views](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 來實現。Blade 是一種極其輕量級的模板語言，為顯示資料、迭代資料等提供了方便、簡潔的語法：

```blade
<div>
    @foreach ($users as $user)
        Hello, {{ $user->name }} <br />
    @endforeach
</div>
```

以這種方式建構應用程式時，表單提交和其他頁面互動通常會從伺服器接收一個全新的 HTML 文件，並由瀏覽器重新渲染整個頁面。即使在今天，許多應用程式可能仍然非常適合使用簡單的 Blade 模板以這種方式建構前端。


<a name="growing-expectations"></a>
#### 使用者期待的提升

然而，隨著使用者對 Web 應用程式的期望日益成熟，許多開發者發現需要建構更具動態性且互動感更精緻的前端。鑑於此，一些開發者選擇開始使用 JavaScript 框架（如 React、Svelte 和 Vue）來建構應用程式的前端。

其他人則偏好堅持使用他們熟悉的後端語言，開發了一些解決方案，讓他們在主要利用所選後端語言的同時，仍能建構現代化的 Web 應用程式 UI。例如在 [Rails](https://rubyonrails.org/) 生態系統中，這促使了如 [Turbo](https://turbo.hotwired.dev/)、[Hotwire](https://hotwired.dev/) 和 [Stimulus](https://stimulus.hotwired.dev/) 等函式庫的誕生。

在 Laravel 生態系統中，這種主要使用 PHP 來建立現代化、動態前端的需求，促成了 [Laravel Livewire](https://livewire.laravel.com) 和 [Alpine.js](https://alpinejs.dev/) 的產生。


<a name="livewire"></a>
### Livewire

[Laravel Livewire](https://livewire.laravel.com) 是一個用於建構由 Laravel 驅動的前端框架，其感覺就像使用 React、Svelte 和 Vue 等現代 JavaScript 框架建構的前端一樣，具有動態、現代且充滿活力的特性。

使用 Livewire 時，您將建立 Livewire 「元件」，這些元件會渲染 UI 的獨立部分，並公開可以從應用程式前端呼叫和互動的方法與資料。例如，一個簡單的「計數器 (Counter)」元件可能如下所示：

```php
<?php

use Livewire\Component;

new class extends Component
{
    public $count = 0;

    public function increment()
    {
        $this->count++;
    }
};
?>

<div>
    <button wire:click="increment">+</button>
    <h1>{{ $count }}</h1>
</div>

```

如您所見，Livewire 讓您可以撰寫新的 HTML 屬性，例如 `wire:click`，以連接 Laravel 應用程式的前端和後端。此外，您可以使用簡單的 Blade 表達式來渲染元件的當前狀態。

對許多人來說，Livewire 革命化了 Laravel 的前端開發，讓他們在建構現代化、動態 Web 應用程式的同時，能留在 Laravel 的舒適圈內。通常，使用 Livewire 的開發者也會利用 [Alpine.js](https://alpinejs.dev/) 在前端「點綴」所需的 JavaScript，例如用來渲染對話視窗。

如果您是 Laravel 新手，我們建議您先熟悉 [views](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 的基本用法。接著，請參閱官方的 [Laravel Livewire 說明文件](https://livewire.laravel.com/docs)，學習如何透過互動式的 Livewire 元件將您的應用程式提升到新的層次。


<a name="php-starter-kits"></a>
### 入門套件

如果您想使用 PHP 和 Livewire 建構前端，可以利用我們的 [Livewire 入門套件](/docs/{{version}}/starter-kits)來快速啟動您的應用程式開發。

<a name="using-react-svelte-or-vue"></a>
## 使用 React、Svelte 或 Vue

雖然可以使用 Laravel 和 Livewire 構建現代前端，但許多開發者仍然偏好利用 React、Svelte 或 Vue 等 JavaScript 框架的力量。這讓開發者能夠利用透過 NPM 提供的豐富 JavaScript 套件與工具生態系統。

然而，如果沒有額外的工具，將 Laravel 與 React、Svelte 或 Vue 搭配使用，會讓我們需要解決各種複雜的問題，例如客戶端路由 (client-side routing)、資料填充 (data hydration) 和認證 (authentication)。雖然使用像 [Next](https://nextjs.org/) 和 [Nuxt](https://nuxt.com/) 這樣具有既定設計慣例的 React / Svelte / Vue 框架通常能簡化客戶端路由；但是，當將 Laravel 這樣的後端框架與這些前端框架搭配時，資料填充和認證仍然是複雜且繁瑣的問題。

此外，開發者還必須維護兩個獨立的程式碼庫，通常需要協調這兩個儲存庫之間的維護、發布和部署。雖然這些問題並非無法解決，但我們不認為這是一種高效或令人愉快的應用程式開發方式。


<a name="inertia"></a>
### Inertia

幸好，Laravel 提供了兩全其美的方案。[Inertia](https://inertiajs.com) 橋接了您的 Laravel 應用程式與現代的 React、Svelte 或 Vue 前端，讓您能使用 React、Svelte 或 Vue 構建完整且現代的前端，同時利用 Laravel 的路由和控制器來處理路由、資料填充和認證 —— 且這一切都在單一程式碼庫中完成。透過這種方式，您可以享受 Laravel 和 React / Svelte / Vue 的強大功能，而不會削弱任何一種工具的能力。

在您的 Laravel 應用程式中安裝 Inertia 後，您將照常撰寫路由和控制器。但是，您不再是從控制器回傳 Blade 模板，而是回傳一個 Inertia 頁面：

```php
<?php

namespace App\Http\Controllers;

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
        return Inertia::render('users/show', [
            'user' => User::findOrFail($id)
        ]);
    }
}
```

一個 Inertia 頁面對應一個 React、Svelte 或 Vue 元件，通常存放在應用程式的 `resources/js/pages` 目錄中。透過 `Inertia::render` 方法傳遞給頁面的資料將被用於填充頁面元件的 "props"：

```jsx
import Layout from '@/layouts/authenticated';
import { Head } from '@inertiajs/react';

export default function Show({ user }) {
    return (
        <Layout>
            <Head title="Welcome" />
            <h1>Welcome</h1>
            <p>Hello {user.name}, welcome to Inertia.</p>
        </Layout>
    )
}
```

如您所見，Inertia 讓您在構建前端時能充分發揮 React、Svelte 或 Vue 的強大功能，同時在您的 Laravel 授權後端與 JavaScript 前端之間提供了一個輕量級的橋樑。


#### 伺服器端渲染 (Server-Side Rendering)

如果您因為應用程式需要伺服器端渲染而對投入 Inertia 有所顧慮，請不必擔心。Inertia 提供了 [伺服器端渲染支援](https://inertiajs.com/server-side-rendering)。而且，當透過 [Laravel Cloud](https://cloud.laravel.com) 或 [Laravel Forge](https://forge.laravel.com) 部署應用程式時，確保 Inertia 的伺服器端渲染程序持續執行變得非常簡單。


<a name="inertia-starter-kits"></a>
### 入門套件

如果您想使用 Inertia 和 React / Svelte / Vue 構建前端，可以利用我們的 [React、Svelte 或 Vue 應用程式入門套件](/docs/{{version}}/starter-kits) 來加速您的應用程式開發。所有這些入門套件都使用 Inertia、React / Svelte / Vue、[Tailwind](https://tailwindcss.com) 和 [Vite](https://vitejs.dev) 來搭建應用程式的後端和前端認證流程，讓您可以開始構建您的下一個偉大創意。


<a name="bundling-assets"></a>
## 靜態資源打包

無論您選擇使用 Blade 和 Livewire，還是 React / Svelte / Vue 和 Inertia 來開發前端，您可能都需要將應用程式的 CSS 打包成生產就緒的靜態資源。當然，如果您選擇使用 React、Svelte 或 Vue 構建應用程式的前端，您還需要將元件打包成瀏覽器就緒的 JavaScript 靜態資源。

預設情況下，Laravel 使用 [Vite](https://vitejs.dev) 來打包您的靜態資源。Vite 在本地開發期間提供極快的編譯速度和幾近即時的模組熱替換 (HMR)。在所有新的 Laravel 應用程式中（包括使用我們[入門套件](/docs/{{version}}/starter-kits)的應用程式），您都會找到一個 `vite.config.js` 檔案，該檔案載入了我們輕量級的 Laravel Vite 外掛，讓 Vite 在 Laravel 應用程式中變得非常好用。

開始使用 Laravel 和 Vite 最快的方法是使用[我們的應用程式入門套件](/docs/{{version}}/starter-kits)開始開發您的應用程式，這些套件透過提供前端和後端的認證架構來加速您的應用程式開發。

> [!NOTE]
> 有關在 Laravel 中使用 Vite 的更多詳細文件，請參閱我們關於[打包與編譯靜態資源的專門文件](/docs/{{version}}/vite)。