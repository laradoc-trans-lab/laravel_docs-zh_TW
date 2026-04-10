# 前端

- [介紹](#introduction)
- [使用 PHP](#using-php)
    - [PHP 與 Blade](#php-and-blade)
    - [Livewire](#livewire)
    - [入門套件](#php-starter-kits)
- [使用 React, Svelte, 或 Vue](#using-react-svelte-or-vue)
    - [Inertia](#inertia)
    - [入門套件](#inertia-starter-kits)
- [打包資源](#bundling-assets)

<a name="introduction"></a>
## 介紹

Laravel 是一個後端框架，提供了構建現代 Web 應用程式所需的所有功能，例如[路由](/docs/{{version}}/routing)、[驗證](/docs/{{version}}/validation)、[快取](/docs/{{version}}/cache)、[隊列](/docs/{{version}}/queues)、[檔案儲存](/docs/{{version}}/filesystem)等等。然而，我們認為為開發者提供優質的全端體驗非常重要，這包括用於構建應用程式前端的強大方法。

在使用 Laravel 構建應用程式時，主要有兩種處理前端開發的方法，而你選擇哪種方法取決於你希望透過利用 PHP 還是使用 React、Svelte 和 Vue 等 JavaScript 框架來構建前端。我們將在下方討論這兩個選項，以便你可以針對應用程式的前端開發做出明智的決定。


<a name="using-php"></a>
## 使用 PHP


<a name="php-and-blade"></a>
### PHP 與 Blade

過去，大多數 PHP 應用程式使用簡單的 HTML 模板，並穿插著 PHP `echo` 語句來向瀏覽器渲染 HTML，這些語句會印出在請求期間從資料庫檢索到的資料：

```blade
<div>
    <?php foreach ($users as $user): ?>
        Hello, <?php echo $user->name; ?> <br />
    <?php endforeach; ?>
</div>
```

在 Laravel 中，仍然可以使用 [views](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 來實現這種渲染 HTML 的方法。Blade 是一種極其輕量級的模板語言，為顯示資料、迭代資料等提供了方便、簡短的語法：

```blade
<div>
    @foreach ($users as $user)
        Hello, {{ $user->name }} <br />
    @endforeach
</div>
```

以這種方式構建應用程式時，表單提交和其他頁面互動通常會從伺服器接收一個全新的 HTML 文件，並且整個頁面由瀏覽器重新渲染。即使在今天，許多應用程式可能仍然非常適合使用簡單的 Blade 模板以這種方式構建其前端。


<a name="growing-expectations"></a>
#### 日益增長的期望

然而，隨著使用者對 Web 應用程式的期望日益成熟，許多開發者發現需要構建更具動態性且互動感更精緻的前端。有鑑於此，一些開發者選擇開始使用 React、Svelte 和 Vue 等 JavaScript 框架來構建其應用程式的前端。

其他人則傾向於堅持使用他們熟悉的後端語言，開發了一些解決方案，允許在主要仍使用其首選後端語言的同時，構建現代 Web 應用程式的使用者介面。例如，在 [Rails](https://rubyonrails.org/) 生態系統中，這促使了 [Turbo](https://turbo.hotwired.dev/) [Hotwire](https://hotwired.dev/) 和 [Stimulus](https://stimulus.hotwired.dev/) 等函式庫的誕生。

在 Laravel 生態系統中，主要透過使用 PHP 建立現代、動態前端的需求導致了 [Laravel Livewire](https://livewire.laravel.com) 和 [Alpine.js](https://alpinejs.dev/) 的誕生。


<a name="livewire"></a>
### Livewire

[Laravel Livewire](https://livewire.laravel.com) 是一個用於構建由 Laravel 驅動的前端框架，其感覺就像使用 React、Svelte 和 Vue 等現代 JavaScript 框架構建的前端一樣，具有動態、現代且充滿活力的特點。

使用 Livewire 時，你將建立 Livewire 「組件 (components)」，這些組件負責渲染 UI 的離散部分，並公開可以從應用程式前端調用和互動的方法與資料。例如，一個簡單的「計數器 (Counter)」組件可能如下所示：

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

如你所見，Livewire 讓你可以編寫新的 HTML 屬性（如 `wire:click`），將 Laravel 應用程式的前端和後端連接起來。此外，你可以使用簡單的 Blade 表達式來渲染組件的當前狀態。

對許多人來說，Livewire 徹底改變了 Laravel 的前端開發，讓他們在構建現代、動態的 Web 應用程式時能夠保持在 Laravel 的舒適圈內。通常，使用 Livewire 的開發者還會利用 [Alpine.js](https://alpinejs.dev/) 將 JavaScript 「灑」在僅有需要的前端部分，例如為了渲染一個對話框窗口。

如果你是 Laravel 的新手，我們建議先熟悉 [views](/docs/{{version}}/views) 和 [Blade](/docs/{{version}}/blade) 的基本用法。然後，請參考官方 [Laravel Livewire documentation](https://livewire.laravel.com/docs) 以瞭解如何使用互動式 Livewire 組件將你的應用程式提升到下一個境界。


<a name="php-starter-kits"></a>
### 入門套件

如果你希望使用 PHP 和 Livewire 構建前端，可以利用我們的 [Livewire 入門套件](/docs/{{version}}/starter-kits) 來啟動你的應用程式開發。

<a name="using-react-svelte-or-vue"></a>
## 使用 React, Svelte, 或 Vue

雖然可以使用 Laravel 和 Livewire 構建現代化的前端，但許多開發者仍偏好利用 React、Svelte 或 Vue 等 JavaScript 框架的力量。這讓開發者能夠利用經由 NPM 提供的豐富 JavaScript 套件與工具生態系統。

然而，在沒有額外工具的情況下，將 Laravel 與 React、Svelte 或 Vue 配對會讓我們需要解決各種複雜的問題，例如用戶端路由、資料填充 (Data Hydration) 和身分驗證。用戶端路由通常透過使用像是 [Next](https://nextjs.org/) 和 [Nuxt](https://nuxt.com/) 等具有特定規範的 React / Svelte / Vue 框架來簡化；然而，當將 Laravel 這樣的後端框架與這些前端框架配對時，資料填充和身分驗證仍然是複雜且繁瑣的問題。

此外，開發者必須維護兩個獨立的程式碼庫，通常需要跨這兩個程式碼庫協調維護、發佈和部署。雖然這些問題並非無法克服，但我們不認為這是一種高效或令人愉快的開發應用程式的方式。


<a name="inertia"></a>
### Inertia

慶幸的是，Laravel 提供了兩全其美的方案。[Inertia](https://inertiajs.com) 彌合了您的 Laravel 應用程式與現代 React、Svelte 或 Vue 前端之間的差距，讓您可以使用 React、Svelte 或 Vue 構建完整且現代的前端，同時利用 Laravel 的路由和控制器來處理路由、資料填充和身分驗證 —— 且這一切都在單一的程式碼庫中完成。透過這種方式，您可以同時享受 Laravel 和 React / Svelte / Vue 的強大功能，而不會削弱任何一種工具的能力。

在您的 Laravel 應用程式中安裝 Inertia 後，您可以像往常一樣撰寫路由和控制器。但是，您不再是從控制器回傳 Blade 樣板，而是回傳一個 Inertia 頁面：

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

一個 Inertia 頁面對應到一個 React、Svelte 或 Vue 元件，通常存放在應用程式的 `resources/js/pages` 目錄中。透過 `Inertia::render` 方法提供給頁面的資料將用於填充頁面元件的 "props"：

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

如您所見，Inertia 讓您在構建前端時能充分利用 React、Svelte 或 Vue 的強大功能，同時在 Laravel 驅動的後端和 JavaScript 驅動的前端之間提供了一個輕量級的橋樑。


#### 伺服器端渲染

如果您因為應用程式需要伺服器端渲染 (Server-Side Rendering) 而對投入 Inertia 有所顧慮，請不必擔心。Inertia 提供了 [伺服器端渲染支援](https://inertiajs.com/server-side-rendering)。而且，當透過 [Laravel Cloud](https://cloud.laravel.com) 或 [Laravel Forge](https://forge.laravel.com) 部署應用程式時，要確保 Inertia 的伺服器端渲染程序始終運行是非常容易的事。


<a name="inertia-starter-kits"></a>
### 入門套件

如果您想使用 Inertia 和 React / Svelte / Vue 構建前端，您可以利用我們的 [React、Svelte 或 Vue 應用程式入門套件](/docs/{{version}}/starter-kits) 來快速啟動應用程式的開發。這兩個入門套件都使用 Inertia、React / Svelte / Vue、[Tailwind](https://tailwindcss.com) 和 [Vite](https://vitejs.dev) 建立了應用程式後端和前端的身分驗證流程，讓您可以開始實踐您的下一個偉大創意。


<a name="bundling-assets"></a>
## 打包資源

無論您選擇使用 Blade 和 Livewire，還是 React / Svelte / Vue 和 Inertia 來開發前端，您可能都需要將應用程式的 CSS 打包成正式生產環境使用的資源。當然，如果您選擇使用 React、Svelte 或 Vue 構建應用程式的前端，您還需要將元件打包成瀏覽器可用的 JavaScript 資源。

預設情況下，Laravel 使用 [Vite](https://vitejs.dev) 來打包您的資源。Vite 在本地開發期間提供了極快的編譯時間和幾乎瞬時的熱模組替換 (HMR)。在所有新的 Laravel 應用程式中，包括使用我們 [入門套件](/docs/{{version}}/starter-kits) 的應用程式，您都會找到一個 `vite.config.js` 檔案，它載入了我們輕量級的 Laravel Vite 外掛，讓 Vite 在 Laravel 應用程式中極其好用。

開始使用 Laravel 和 Vite 的最快方法是從使用 [我們的應用程式入門套件](/docs/{{version}}/starter-kits) 開始開發應用程式，它透過提供前端和後端的身分驗證樣板來讓您的應用程式快速起步。

> [!NOTE]
> 有關在 Laravel 中使用 Vite 的更多詳細文件，請參閱我們關於 [打包與編譯資源的專屬文件](/docs/{{version}}/vite)。