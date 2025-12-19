# Blade 模板

- [介紹](#introduction)
    - [使用 Livewire 強化 Blade](#supercharging-blade-with-livewire)
- [顯示資料](#displaying-data)
    - [HTML 實體編碼](#html-entity-encoding)
    - [Blade 與 JavaScript 框架](#blade-and-javascript-frameworks)
- [Blade 指令](#blade-directives)
    - [If 條件式](#if-statements)
    - [Switch 條件式](#switch-statements)
    - [迴圈](#loops)
    - [迴圈變數](#the-loop-variable)
    - [條件式 Class](#conditional-classes)
    - [額外屬性](#additional-attributes)
    - [載入子視圖](#including-subviews)
    - [`@once` 指令](#the-once-directive)
    - [原生 PHP](#raw-php)
    - [註解](#comments)
- [元件](#components)
    - [渲染元件](#rendering-components)
    - [索引元件](#index-components)
    - [傳遞資料至元件](#passing-data-to-components)
    - [元件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [插槽](#slots)
    - [行內元件視圖](#inline-component-views)
    - [動態元件](#dynamic-components)
    - [手動註冊元件](#manually-registering-components)
- [匿名元件](#anonymous-components)
    - [匿名索引元件](#anonymous-index-components)
    - [資料屬性 / 特性](#data-properties-attributes)
    - [存取父層資料](#accessing-parent-data)
    - [匿名元件路徑](#anonymous-component-paths)
- [建立佈局](#building-layouts)
    - [使用元件的佈局](#layouts-using-components)
    - [使用模板繼承的佈局](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [Method 欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [Stacks](#stacks)
- [服務注入](#service-injection)
- [渲染行內 Blade 模板](#rendering-inline-blade-templates)
- [渲染 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自訂 Echo 處理器](#custom-echo-handlers)
    - [自訂 If 條件式](#custom-if-statements)

<a name="introduction"></a>
## 介紹

Blade 是 Laravel 中一個簡單卻強大的模板引擎。與一些 PHP 模板引擎不同，Blade 不限制您在模板中使用原生 PHP 程式碼。事實上，所有 Blade 模板都會被編譯成原生 PHP 程式碼並快取起來，直到被修改為止，這意味著 Blade 幾乎不會為您的應用程式增加任何負擔。Blade 模板檔案使用 `.blade.php` 副檔名，通常儲存在 `resources/views` 目錄中。

Blade 視圖可以使用全域 `view` 輔助函式從路由或控制器中返回。當然，正如 [視圖](/docs/{{version}}/views) 文件中提到的，資料可以使用 `view` 輔助函式的第二個參數傳遞給 Blade 視圖：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'Finn']);
});
```

<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想要將您的 Blade 模板提升到一個新的層次並輕鬆建構動態介面嗎？請參考 [Laravel Livewire](https://livewire.laravel.com)。Livewire 允許您編寫 Blade 元件，這些元件透過動態功能進行增強，這些功能通常只能透過 React 或 Vue 等前端框架實現，為建構現代、響應式前端提供了一種很好的方法，同時避免了許多 JavaScript 框架的複雜性、用戶端渲染或建構步驟。

<a name="displaying-data"></a>
## 顯示資料

您可以透過將變數包在花括號中來顯示傳遞給 Blade 視圖的資料。例如，給定以下路由：

```php
Route::get('/', function () {
    return view('welcome', ['name' => 'Samantha']);
});
```

您可以像這樣顯示 `name` 變數的內容：

```blade
Hello, {{ $name }}.
```

> [!NOTE]
> Blade 的 `{{ }}` echo 語句會自動透過 PHP 的 `htmlspecialchars` 函式傳送，以防止 XSS 攻擊。

您不僅限於顯示傳遞給視圖的變數內容。您也可以輸出任何 PHP 函式的結果。事實上，您可以在 Blade echo 語句中放置任何您想要的 PHP 程式碼：

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade (以及 Laravel 的 `e` 函式) 會對 HTML 實體進行雙重編碼。如果您想停用雙重編碼，請在 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Blade;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Blade::withoutDoubleEncoding();
    }
}
```

<a name="displaying-unescaped-data"></a>
#### 顯示未逸出的資料

預設情況下，Blade `{{ }}` 語句會自動透過 PHP 的 `htmlspecialchars` 函式傳送，以防止 XSS 攻擊。如果您不希望您的資料被逸出，可以使用以下語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 在輸出應用程式使用者提供的內容時請務必小心。通常您應該使用逸出的雙花括號語法來防止在顯示使用者提供的資料時發生 XSS 攻擊。

<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「花括號」來表示應在瀏覽器中顯示的表達式，您可以使用 `@` 符號來告知 Blade 渲染引擎某個表達式應保持不變。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，`@` 符號會被 Blade 移除；但是，`{{ name }}` 表達式將保持不變，讓您的 JavaScript 框架進行渲染。

`@` 符號也可以用來逸出 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```

<a name="rendering-json"></a>
#### 渲染 JSON

有時您可能會將一個陣列傳遞給視圖，目的是將其渲染為 JSON 以初始化 JavaScript 變數。例如：

```php
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

然而，您可以使用 `Illuminate\Support\Js::from` 方法，而非手動呼叫 `json_encode`。`from` 方法接受與 PHP 的 `json_encode` 函式相同的參數；但是，它會確保產生的 JSON 已正確逸出以包含在 HTML 引號中。`from` 方法將返回一個 `JSON.parse` JavaScript 語句字串，該語句會將給定的物件或陣列轉換為有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新版本的 Laravel 應用程式骨架包含一個 `Js` Facade，它在您的 Blade 模板中提供了方便的此功能存取：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 您應該只使用 `Js::from` 方法將現有變數渲染為 JSON。Blade 模板是基於正規表達式，嘗試將複雜表達式傳遞給指令可能會導致意外失敗。

<a name="the-at-verbatim-directive"></a>
#### `@verbatim` 指令

如果您在模板的很大一部分中顯示 JavaScript 變數，您可以將 HTML 包裹在 `@verbatim` 指令中，這樣您就不必為每個 Blade echo 語句加上 `@` 符號前綴：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了模板繼承與顯示資料外，Blade 也為常見的 PHP 控制結構（例如條件式語句和迴圈）提供了方便的快捷方式。這些快捷方式提供了一種非常簡潔、精練的方式來處理 PHP 控制結構，同時也與它們的 PHP 對應項保持熟悉。

<a name="if-statements"></a>
### If 條件式

您可以使用 `@if`、`@elseif`、`@else` 和 `@endif` 指令來建構 `if` 條件式語句。這些指令的功能與它們的 PHP 對應項完全相同：

```blade
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif
```

為求方便，Blade 也提供了 `@unless` 指令：

```blade
@unless (Auth::check())
    You are not signed in.
@endunless
```

除了已經討論的條件式指令外，`@isset` 和 `@empty` 指令也可用作它們各自 PHP 函式的便捷快捷方式：

```blade
@isset($records)
    // $records is defined and is not null...
@endisset

@empty($records)
    // $records is "empty"...
@endempty
```

<a name="authentication-directives"></a>
#### 驗證指令

`@auth` 和 `@guest` 指令可用來快速判斷目前使用者是否已 [authenticated](/docs/{{version}}/authentication) 或為訪客：

```blade
@auth
    // The user is authenticated...
@endauth

@guest
    // The user is not authenticated...
@endguest
```

如有需要，在使用 `@auth` 和 `@guest` 指令時，您可以指定應該檢查的驗證守衛：

```blade
@auth('admin')
    // The user is authenticated...
@endauth

@guest('admin')
    // The user is not authenticated...
@endguest
```

<a name="environment-directives"></a>
#### 環境指令

您可以使用 `@production` 指令來檢查應用程式是否正在生產環境中運行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，您可以使用 `@env` 指令來判斷應用程式是否正在特定環境中運行：

```blade
@env('staging')
    // The application is running in "staging"...
@endenv

@env(['staging', 'production'])
    // The application is running in "staging" or "production"...
@endenv
```

<a name="section-directives"></a>
#### 區段指令

您可以使用 `@hasSection` 指令來判斷模板繼承區段是否有內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

您可以使用 `sectionMissing` 指令來判斷區段是否沒有內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```

<a name="session-directives"></a>
#### Session 指令

`@session` 指令可用來判斷 [session](/docs/{{version}}/session) 值是否存在。如果 session 值存在，則會評估 `@session` 和 `@endsession` 指令之間的模板內容。在 `@session` 指令的內容中，您可以輸出 `$value` 變數來顯示 session 值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```

<a name="context-directives"></a>
#### Context 指令

`@context` 指令可用來判斷 [context](/docs/{{version}}/context) 值是否存在。如果 context 值存在，則會評估 `@context` 和 `@endcontext` 指令之間的模板內容。在 `@context` 指令的內容中，您可以輸出 `$value` 變數來顯示 context 值：

```blade
@context('canonical')
    <link href="{{ $value }}" rel="canonical">
@endcontext
```

<a name="switch-statements"></a>
### Switch 條件式

可以使用 `@switch`、`@case`、`@break`、`@default` 和 `@endswitch` 指令來建構 Switch 條件式語句：

```blade
@switch($i)
    @case(1)
        First case...
        @break

    @case(2)
        Second case...
        @break

    @default
        Default case...
@endswitch
```

<a name="loops"></a>
### 迴圈

除了條件式語句，Blade 還提供了用於處理 PHP 迴圈結構的簡單指令。同樣地，每個指令的功能都與它們的 PHP 對應項完全相同：

```blade
@for ($i = 0; $i < 10; $i++)
    The current value is {{ $i }}
@endfor

@foreach ($users as $user)
    <p>This is user {{ $user->id }}</p>
@endforeach

@forelse ($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>No users</p>
@endforelse

@while (true)
    <p>I'm looping forever.</p>
@endwhile
```

> [!NOTE]
> 在遍歷 `foreach` 迴圈時，您可以使用 [迴圈變數](#the-loop-variable) 來獲取有關迴圈的寶貴資訊，例如您是否處於迴圈的第一次或最後一次迭代。

使用迴圈時，您還可以使用 `@continue` 和 `@break` 指令來跳過當前迭代或結束迴圈：

```blade
@foreach ($users as $user)
    @if ($user->type == 1)
        @continue
    @endif

    <li>{{ $user->name }}</li>

    @if ($user->number == 5)
        @break
    @endif
@endforeach
```

您也可以在指令宣告中包含繼續或中斷條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```

<a name="the-loop-variable"></a>
### 迴圈變數

在遍歷 `foreach` 迴圈時，一個 `$loop` 變數將在您的迴圈中可用。此變數提供了對一些有用資訊的存取，例如目前的迴圈索引以及這是否是迴圈的第一次或最後一次迭代：

```blade
@foreach ($users as $user)
    @if ($loop->first)
        This is the first iteration.
    @endif

    @if ($loop->last)
        This is the last iteration.
    @endif

    <p>This is user {{ $user->id }}</p>
@endforeach
```

如果您在巢狀迴圈中，可以透過 `parent` 屬性存取父層迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數還包含各種其他有用的屬性：

<div class="overflow-auto">

| Property           | Description                                            |
| ------------------ | ------------------------------------------------------ |
| `$loop->index`     | The index of the current loop iteration (starts at 0). |
| `$loop->iteration` | The current loop iteration (starts at 1).              |
| `$loop->remaining` | The iterations remaining in the loop.                  |
| `$loop->count`     | The total number of items in the array being iterated. |
| `$loop->first`     | Whether this is the first iteration through the loop.  |
| `$loop->last`      | Whether this is the last iteration through the loop.   |
| `$loop->even`      | Whether this is an even iteration through the loop.    |
| `$loop->odd`       | Whether this is an odd iteration through the loop.     |
| `$loop->depth`     | The nesting level of the current loop.                 |
| `$loop->parent`    | When in a nested loop, the parent's loop variable.     |

</div>

<a name="conditional-classes"></a>
### 條件式 Class

`@class` 指令會條件式地編譯一個 CSS class 字串。這個指令接受一個 class 陣列，其中陣列鍵包含您想要新增的 class 或多個 class，而值則是一個布林表達式。如果陣列元素有數字鍵，它將始終包含在渲染後的 class 列表中：

```blade
@php
    $isActive = false;
    $hasError = true;
@endphp

<span @class([
    'p-4',
    'font-bold' => $isActive,
    'text-gray-500' => ! $isActive,
    'bg-red' => $hasError,
])></span>

<span class="p-4 text-gray-500 bg-red"></span>
```

同樣地，`@style` 指令可用於條件式地將行內 CSS 樣式新增到 HTML 元素中：

```blade
@php
    $isActive = true;
@endphp

<span @style([
    'background-color: red',
    'font-weight: bold' => $isActive,
])></span>

<span style="background-color: red; font-weight: bold;"></span>
```

<a name="additional-attributes"></a>
### 額外屬性

為方便起見，您可以使用 `@checked` 指令輕鬆指出指定的 HTML 勾選框輸入是否為「已勾選」。如果提供的條件評估為 `true`，這個指令將會印出 `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，`@selected` 指令可用於指出指定的選取選項是否應該為「已選取」。

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，`@disabled` 指令可用於指出指定的元素是否應該為「已停用」。

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，`@readonly` 指令可用於指出指定的元素是否應該為「唯讀」。

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

此外，`@required` 指令可用於指出指定的元素是否應該為「必填」。

```blade
<input
    type="text"
    name="title"
    value="title"
    @required($user->isAdmin())
/>
```

<a name="including-subviews"></a>
### 載入子視圖

> [!NOTE]
> 雖然您可以自由使用 `@include` 指令，但 Blade [元件](#components) 提供了類似的功能，並比 `@include` 指令提供了多項優勢，例如資料和屬性綁定。

Blade 的 `@include` 指令允許您在一個視圖中載入另一個 Blade 視圖。所有在父視圖中可用的變數都將提供給被載入的視圖：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

儘管被載入的視圖將繼承父視圖中所有可用的資料，但您也可以傳遞一個額外資料陣列，以供被載入的視圖使用：

```blade
@include('view.name', ['status' => 'complete'])
```

如果您嘗試 `@include` 一個不存在的視圖，Laravel 將會拋出錯誤。如果您想載入一個可能存在也可能不存在的視圖，您應該使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

如果您想在指定的布林表達式評估為 `true` 或 `false` 時 `@include` 一個視圖，您可以使用 `@includeWhen` 和 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

若要從指定視圖陣列中載入第一個存在的視圖，您可以使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

> [!WARNING]
> 您應避免在 Blade 視圖中使用 `__DIR__` 和 `__FILE__` 常數，因為它們將指向快取、已編譯的視圖位置。

<a name="rendering-views-for-collections"></a>
#### 渲染集合的視圖

您可以使用 Blade 的 `@each` 指令將迴圈和載入合併為一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個參數是為陣列或集合中的每個元素渲染的視圖。第二個參數是您想要迭代的陣列或集合，而第三個參數是將在視圖中分配給當前迭代的變數名稱。因此，舉例來說，如果您正在迭代一個 `jobs` 陣列，通常您會希望在視圖中將每個 job 存取為 `job` 變數。當前迭代的陣列鍵將作為 `key` 變數在視圖中可用。

您也可以傳遞第四個參數給 `@each` 指令。這個參數決定了如果給定的陣列為空時將會渲染哪個視圖。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 `@each` 渲染的視圖不會繼承父視圖中的變數。如果子視圖需要這些變數，您應該改用 `@foreach` 和 `@include` 指令。

<a name="the-once-directive"></a>
### `@once` 指令

`@once` 指令允許您定義模板中在每個渲染週期中只會被評估一次的部分。這對於使用 [Stacks](#stacks) 將指定的 JavaScript 片段推送到頁面標頭可能很有用。舉例來說，如果您在迴圈中渲染指定的 [元件](#components)，您可能希望只在元件第一次渲染時將 JavaScript 推送到標頭：

```blade
@once
    @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush
@endonce
```

由於 `@once` 指令經常與 `@push` 或 `@prepend` 指令結合使用，因此為了您的方便，提供了 `@pushOnce` 和 `@prependOnce` 指令：

```blade
@pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
@endPushOnce
```

如果您從兩個獨立的 Blade 模板推送重複的內容，您應該為 `@pushOnce` 指令提供一個唯一的識別符作為第二個參數，以確保內容只渲染一次：

```blade
<!-- pie-chart.blade.php -->
@pushOnce('scripts', 'chart.js')
    <script src="/chart.js"></script>
@endPushOnce

<!-- line-chart.blade.php -->
@pushOnce('scripts', 'chart.js')
    <script src="/chart.js"></script>
@endPushOnce
```

<a name="raw-php"></a>
### 原生 PHP

在某些情況下，將 PHP 程式碼嵌入到您的視圖中會很有用。您可以使用 Blade 的 `@php` 指令在您的模板中執行一段原生 PHP 程式碼區塊：

```blade
@php
    $counter = 1;
@endphp
```

或者，如果您只需要使用 PHP 導入 Class，您可以使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

您可以為 `@use` 指令提供第二個參數，以別名化導入的 Class：

```blade
@use('App\Models\Flight', 'FlightModel')
```

如果您在同一個 Namespace 中有多個 Class，您可以將這些 Class 的導入分組：

```blade
@use('App\Models\{Flight, Airport}')
```

`@use` 指令也支援透過在導入路徑前加上 `function` 或 `const` 修飾符來導入 PHP 函式和常數：

```blade
@use(function App\Helpers\format_currency)
@use(const App\Constants\MAX_ATTEMPTS)
```

就像 Class 導入一樣，函式和常數也支援別名：

```blade
@use(function App\Helpers\format_currency, 'formatMoney')
@use(const App\Constants\MAX_ATTEMPTS, 'MAX_TRIES')
```

群組導入也支援 function 和 const 修飾符，允許您在一個指令中從同一個 Namespace 導入多個符號：

```blade
@use(function App\Helpers\{format_currency, format_date})
@use(const App\Constants\{MAX_ATTEMPTS, DEFAULT_TIMEOUT})
```

<a name="comments"></a>
### 註解

Blade 也允許您在視圖中定義註解。然而，與 HTML 註解不同的是，Blade 註解不會包含在您的應用程式所回傳的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件和插槽提供了類似於區塊、佈局和引入的優點；然而，有些人可能會覺得元件和插槽的心智模型更容易理解。撰寫元件有兩種方法：基於類別的元件和匿名元件。

若要建立基於類別的元件，您可以使用 `make:component` Artisan 指令。為了說明如何使用元件，我們將建立一個簡單的 `Alert` 元件。`make:component` 指令會將元件放置在 `app/View/Components` 目錄中：

```shell
php artisan make:component Alert
```

`make:component` 指令也會為元件建立一個視圖模板。該視圖將放置在 `resources/views/components` 目錄中。當為您自己的應用程式撰寫元件時，元件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現，因此通常不需要進一步的元件註冊。

您也可以在子目錄中建立元件：

```shell
php artisan make:component Forms/Input
```

上述指令將在 `app/View/Components/Forms` 目錄中建立一個 `Input` 元件，並且該視圖將放置在 `resources/views/components/forms` 目錄中。

<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

當為您自己的應用程式撰寫元件時，元件會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被發現。

然而，如果您正在建立一個使用 Blade 元件的套件，您將需要手動註冊您的元件類別及其 HTML 標籤別名。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::component('package-alert', Alert::class);
}
```

一旦您的元件被註冊，它就可以使用其標籤別名來渲染：

```blade
<x-package-alert/>
```

另外，您也可以使用 `componentNamespace` 方法透過慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能擁有 `Calendar` 和 `ColorPicker` 元件，這些元件位於 `Package\Views\Components` 命名空間中：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
}
```

這將允許使用 `package-name::` 語法，透過其供應商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過 Pascal-casing (帕斯卡命名法) 元件名稱來自動偵測與此元件連結的類別。子目錄也支援使用「點」符號。

<a name="rendering-components"></a>
### 渲染元件

若要顯示元件，您可以在您的 Blade 模板中使用 Blade 元件標籤。Blade 元件標籤以字串 `x-` 開頭，後接元件類別的 kebab-case (烤肉串命名法) 名稱：

```blade
<x-alert/>

<x-user-profile/>
```

如果元件類別巢狀更深地包含在 `app/View/Components` 目錄中，您可以使用 `.` 字元來表示目錄巢狀。例如，如果我們假設元件位於 `app/View/Components/Inputs/Button.php`，我們可以這樣渲染它：

```blade
<x-inputs.button/>
```

如果您想有條件地渲染您的元件，您可以在您的元件類別上定義一個 `shouldRender` 方法。如果 `shouldRender` 方法回傳 `false`，該元件將不會被渲染：

```php
use Illuminate\Support\Str;

/**
 * Whether the component should be rendered
 */
public function shouldRender(): bool
{
    return Str::length($this->message) > 0;
}
```

<a name="index-components"></a>
### 索引元件

有時候，元件是一個元件群組的一部分，您可能會希望將相關的元件分組到單一目錄中。例如，想像一個具有以下類別結構的「卡片」元件：

```text
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件巢狀於 `Card` 目錄中，您可能會預期需要透過 `<x-card.card>` 來渲染元件。然而，當元件的檔名與元件目錄的名稱相符時，Laravel 會自動假設該元件是「根」元件，並允許您在不重複目錄名稱的情況下渲染元件：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料至元件

您可以透過 HTML 屬性將資料傳遞給 Blade 元件。硬編碼的基本值可以透過簡單的 HTML 屬性字串傳遞給元件。PHP 運算式和變數應透過帶有 `:` 字元作為前綴的屬性傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

您應該在元件的類別建構子中定義所有元件的資料屬性。元件上的所有 Public 屬性都會自動提供給元件的視圖。沒有必要透過元件的 `render` 方法將資料傳遞給視圖：

```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;
use Illuminate\View\View;

class Alert extends Component
{
    /**
     * Create the component instance.
     */
    public function __construct(
        public string $type,
        public string $message,
    ) {}

    /**
     * Get the view / contents that represent the component.
     */
    public function render(): View
    {
        return view('components.alert');
    }
}
```

當您的元件被渲染時，您可以透過名稱回傳變數來顯示元件 Public 變數的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```


<a name="casing"></a>
#### 命名慣例

元件建構子參數應使用 `camelCase` 命名，而在 HTML 屬性中引用參數名稱時應使用 `kebab-case`。例如，給定以下元件建構子：

```php
/**
 * Create the component instance.
 */
public function __construct(
    public string $alertType,
) {}
```

`$alertType` 參數可以這樣提供給元件：

```blade
<x-alert alert-type="danger" />
```


<a name="short-attribute-syntax"></a>
#### 簡短屬性語法

將屬性傳遞給元件時，您也可以使用「簡短屬性」語法。這通常很方便，因為屬性名稱經常與其對應的變數名稱相符：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```


<a name="escaping-attribute-rendering"></a>
#### 跳脫屬性渲染

由於許多 JavaScript 框架（例如 Alpine.js）也使用冒號前綴的屬性，因此您可以使用雙冒號 (`::`) 作為前綴來告知 Blade 該屬性不是 PHP 運算式。例如，給定以下元件：

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

Blade 將渲染以下 HTML：

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```


<a name="component-methods"></a>
#### 元件方法

除了 Public 變數可供您的元件模板使用外，元件上的任何 Public 方法也都可以被呼叫。例如，想像一個具有 `isSelected` 方法的元件：

```php
/**
 * Determine if the given option is the currently selected option.
 */
public function isSelected(string $option): bool
{
    return $option === $this->selected;
}
```

您可以從元件模板中呼叫與方法名稱相符的變數來執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```


<a name="using-attributes-slots-within-component-class"></a>
#### 在元件類別中存取屬性與插槽

Blade 元件也允許您在類別的 render 方法中存取元件名稱、屬性與插槽。但是，為了存取這些資料，您應該從元件的 `render` 方法中回傳一個 Closure：

```php
use Closure;

/**
 * Get the view / contents that represent the component.
 */
public function render(): Closure
{
    return function () {
        return '<div {{ $attributes }}>Components content</div>';
    };
}
```

您的元件 `render` 方法回傳的 Closure 也可能接收一個 `$data` 陣列作為其唯一參數。此陣列將包含幾個提供元件資訊的元素：

```php
return function (array $data) {
    // $data['componentName'];
    // $data['attributes'];
    // $data['slot'];

    return '<div {{ $attributes }}>Components content</div>';
}
```

> [!WARNING]
> `$data` 陣列中的元素不應直接嵌入到您的 `render` 方法回傳的 Blade 字串中，因為這樣做可能透過惡意的屬性內容導致遠端程式碼執行。

`componentName` 等於 HTML 標籤中 `x-` 前綴後使用的名稱。所以 `<x-alert />` 的 `componentName` 將是 `alert`。`attributes` 元素將包含 HTML 標籤上存在的所有屬性。`slot` 元素是一個 `Illuminate\Support\HtmlString` 實例，其中包含元件插槽的內容。

此 Closure 應回傳一個字串。如果回傳的字串對應於現有的視圖，則該視圖將被渲染；否則，回傳的字串將作為行內 Blade 視圖進行評估。


<a name="additional-dependencies"></a>
#### 額外依賴

如果您的元件需要來自 Laravel 的 [服務容器](/docs/{{version}}/container) 的依賴項，您可以將它們列在元件的任何資料屬性之前，它們將由容器自動注入：

```php
use App\Services\AlertCreator;

/**
 * Create the component instance.
 */
public function __construct(
    public AlertCreator $creator,
    public string $type,
    public string $message,
) {}
```


<a name="hiding-attributes-and-methods"></a>
#### 隱藏屬性 / 方法

如果您想防止某些 Public 方法或屬性作為變數暴露給您的元件模板，您可以將它們添加到元件上的 `$except` 陣列屬性中：

```php
<?php

namespace App\View\Components;

use Illuminate\View\Component;

class Alert extends Component
{
    /**
     * The properties / methods that should not be exposed to the component template.
     *
     * @var array
     */
    protected $except = ['type'];

    /**
     * Create the component instance.
     */
    public function __construct(
        public string $type,
    ) {}
}
```

<a name="component-attributes"></a>
### 元件屬性

我們已經探討過如何將資料屬性傳遞給元件；然而，有時候您可能需要指定額外的 HTML 屬性，例如 `class`，這些屬性並非元件運作所需資料的一部分。通常，您會希望將這些額外屬性傳遞給元件模板的根元素。例如，假設我們想要渲染一個 `alert` 元件，如下所示：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於元件建構子的屬性都將自動添加到元件的「屬性包」中。這個屬性包會透過 `$attributes` 變數自動提供給元件使用。所有屬性都可以在元件內部透過印出此變數來渲染：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前不支援在元件標籤內使用 `@env` 等指令。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時您可能需要為屬性指定預設值，或將額外值合併到元件的某些屬性中。為此，您可以使用屬性包的 `merge` 方法。此方法對於定義一組應始終套用至元件的預設 CSS class 特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

如果我們假設此元件是這樣使用：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

最終渲染的元件 HTML 將會如下所示：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 條件式合併 Class

有時您可能希望在給定條件為 `true` 時合併 class。您可以透過 `class` 方法實現此功能，該方法接受一個 class 陣列，其中陣列的 key 包含您希望添加的 class，而 value 則是一個布林運算式。如果陣列元素具有數字 key，它將始終包含在渲染的 class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

如果您需要將其他屬性合併到元件上，您可以將 `merge` 方法鏈接到 `class` 方法之後：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 如果您需要在不應接收合併屬性的其他 HTML 元素上條件式地編譯 class，您可以使用 [@class 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 非 Class 屬性合併

合併非 `class` 屬性時，提供給 `merge` 方法的值將被視為屬性的「預設」值。然而，與 `class` 屬性不同，這些屬性不會與注入的屬性值合併，而是會被覆寫。例如，`button` 元件的實作可能如下所示：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要渲染帶有自訂 `type` 的 button 元件，可以在使用元件時指定。如果未指定任何 type，則會使用 `button` type：

```blade
<x-button type="submit">
    Submit
</x-button>
```

在此範例中，`button` 元件的渲染 HTML 將會是：

```blade
<button type="submit">
    Submit
</button>
```

如果您希望除了 `class` 之外的屬性也能將其預設值與注入值合併，您可以使用 `prepends` 方法。在此範例中，`data-controller` 屬性將始終以 `profile-controller` 開頭，並且任何額外注入的 `data-controller` 值將會放置在此預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 檢索與過濾屬性

您可以使用 `filter` 方法來過濾屬性。此方法接受一個閉包，如果希望保留屬性在屬性包中，該閉包應回傳 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為方便起見，您可以使用 `whereStartsWith` 方法來檢索所有鍵以給定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，`whereDoesntStartWith` 方法可用於排除所有鍵以給定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，您可以渲染給定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

如果您想檢查元件上是否存在某個屬性，您可以使用 `has` 方法。此方法接受屬性名稱作為其唯一參數，並回傳一個布林值，指示該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

如果將陣列傳遞給 `has` 方法，該方法將判斷元件上是否存在所有給定的屬性：

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

`hasAny` 方法可用於判斷元件上是否存在任何給定的屬性：

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

您可以使用 `get` 方法檢索特定屬性的值：

```blade
{{ $attributes->get('class') }}
```

`only` 方法可用於僅檢索具有給定鍵的屬性：

```blade
{{ $attributes->only(['class']) }}
```

`except` 方法可用於檢索除給定鍵之外的所有屬性：

```blade
{{ $attributes->except(['class']) }}
```

<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，某些關鍵字保留供 Blade 內部使用，以渲染元件。以下關鍵字不能在元件中定義為 public 屬性或方法名稱：

<div class="content-list" markdown="1">

- `data`
- `render`
- `resolve`
- `resolveView`
- `shouldRender`
- `view`
- `withAttributes`
- `withName`

</div>

<a name="slots"></a>
### 插槽

您通常需要透過「插槽」向元件傳遞額外內容。元件插槽透過輸出 `$slot` 變數來渲染。為了探討這個概念，讓我們想像一個 `alert` 元件具有以下標記：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過將內容注入元件來傳遞內容至該 `slot`：

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時元件可能需要在元件內的不同位置渲染多個不同的插槽。讓我們修改 alert 元件，以允許注入一個「title」插槽：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

您可以使用 `x-slot` 標籤來定義具名插槽的內容。任何不在明確 `x-slot` 標籤內的內容都將以 `$slot` 變數的形式傳遞給元件：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

您可以呼叫插槽的 `isEmpty` 方法來判斷該插槽是否包含內容：

```blade
<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    @if ($slot->isEmpty())
        This is default content if the slot is empty.
    @else
        {{ $slot }}
    @endif
</div>
```

此外，`hasActualContent` 方法可用於判斷該插槽是否包含任何不是 HTML 註解的「實際」內容：

```blade
@if ($slot->hasActualContent())
    The scope has non-comment content.
@endif
```


<a name="scoped-slots"></a>
#### 作用域插槽

如果您使用過像 Vue 這樣的 JavaScript 框架，您可能熟悉「作用域插槽」，它允許您在插槽內存取元件的資料或方法。您可以在 Laravel 中透過在元件上定義 public 方法或屬性，並透過 `$component` 變數在插槽內存取元件來實現類似的行為。在此範例中，我們假設 `x-alert` 元件在其元件類別上定義了一個 public `formatAlert` 方法：

```blade
<x-alert>
    <x-slot:title>
        {{ $component->formatAlert('Server Error') }}
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```


<a name="slot-attributes"></a>
#### 插槽屬性

與 Blade 元件一樣，您可以為插槽賦予額外[屬性](#component-attributes)，例如 CSS 類別名稱：

```xml
<x-card class="shadow-sm">
    <x-slot:heading class="font-bold">
        Heading
    </x-slot>

    Content

    <x-slot:footer class="text-sm">
        Footer
    </x-slot>
</x-card>
```

若要與插槽屬性互動，您可以存取插槽變數的 `attributes` 屬性。有關如何與屬性互動的更多資訊，請查閱[元件屬性](#component-attributes)文件：

```blade
@props([
    'heading',
    'footer',
])

<div {{ $attributes->class(['border']) }}>
    <h1 {{ $heading->attributes->class(['text-lg']) }}>
        {{ $heading }}
    </h1>

    {{ $slot }}

    <footer {{ $footer->attributes->class(['text-gray-700']) }}>
        {{ $footer }}
    </footer>
</div>
```


<a name="inline-component-views"></a>
### 行內元件視圖

對於非常小的元件，管理元件類別和元件的視圖模板可能會感到麻煩。因此，您可以直接從 `render` 方法傳回元件的標記：

```php
/**
 * Get the view / contents that represent the component.
 */
public function render(): string
{
    return <<<'blade'
        <div class="alert alert-danger">
            {{ $slot }}
        </div>
    blade;
}
```


<a name="generating-inline-view-components"></a>
#### 生成行內視圖元件

要建立渲染行內視圖的元件，您可以在執行 `make:component` 指令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```


<a name="dynamic-components"></a>
### 動態元件

有時您可能需要渲染一個元件，但在執行時期之前並不知道應該渲染哪個元件。在這種情況下，您可以使用 Laravel 內建的 `dynamic-component` 元件來根據執行時期值或變數渲染元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```


<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 以下有關手動註冊元件的文件主要適用於編寫包含視圖元件的 Laravel 套件的開發者。如果您不是在編寫套件，這部分元件文件可能與您不相關。

在為您自己的應用程式編寫元件時，元件會自動探索 `app/View/Components` 目錄和 `resources/views/components` 目錄。

但是，如果您正在建立一個使用 Blade 元件的套件，或者將元件放置在非慣例目錄中，您將需要手動註冊您的元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡找到元件。您通常應該在套件服務提供者的 `boot` 方法中註冊您的元件：

```php
use Illuminate\Support\Facades\Blade;
use VendorPackage\View\Components\AlertComponent;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::component('package-alert', AlertComponent::class);
}
```

一旦您的元件被註冊，就可以使用其標籤別名來渲染它：

```blade
<x-package-alert/>
```


#### 自動載入套件元件

或者，您可以使用 `componentNamespace` 方法按照慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能擁有位於 `Package\Views\Components` 命名空間下的 `Calendar` 和 `ColorPicker` 元件：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap your package's services.
 */
public function boot(): void
{
    Blade::componentNamespace('Nightshade\\Views\\Components', 'nightshade');
}
```

這將允許使用 `package-name::` 語法，透過其供應商命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 將透過將元件名稱轉為 Pascal 大小寫來自動偵測連結到此元件的類別。子目錄也支援使用「點」符號。

<a name="anonymous-components"></a>
## 匿名元件

與行內元件類似，匿名元件提供了一種透過單一檔案管理元件的機制。然而，匿名元件只使用單一視圖檔案，且沒有關聯的類別。要定義匿名元件，您只需要將 Blade 模板放置在 `resources/views/components` 目錄中。例如，假設您在 `resources/views/components/alert.blade.php` 定義了一個元件，您可以這樣簡單地渲染它：

```blade
<x-alert/>
```

您可以使用 `.` 字元來表示元件是否更深地巢狀在 `components` 目錄中。例如，假設元件定義在 `resources/views/components/inputs/button.blade.php`，您可以這樣渲染它：

```blade
<x-inputs.button/>
```

要透過 Artisan 建立匿名元件，您可以在呼叫 `make:component` 命令時使用 `--view` 旗標：

```shell
php artisan make:component forms.input --view
```

上述命令將在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，該檔案可以透過 `<x-forms.input />` 作為元件渲染。

<a name="anonymous-index-components"></a>
### 匿名索引元件

有時，當一個元件由許多 Blade 模板組成時，您可能會希望將給定元件的模板分組到單一目錄中。例如，想像一個「手風琴 (accordion)」元件，其目錄結構如下：

```text
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這種目錄結構允許您這樣渲染手風琴元件及其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

然而，為了透過 `x-accordion` 渲染手風琴元件，我們被迫將「索引」手風琴元件模板放置在 `resources/views/components` 目錄中，而不是將其與其他手風琴相關模板巢狀在 `accordion` 目錄中。

幸運的是，Blade 允許您將與元件目錄名稱相符的檔案放置在元件目錄本身中。當這個模板存在時，即使它巢狀在一個目錄中，它也可以作為元件的「根」元素來渲染。因此，我們可以繼續使用上述範例中給出的相同 Blade 語法；但是，我們將調整我們的目錄結構如下：

```text
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### 資料屬性 / 特性

由於匿名元件沒有任何關聯類別，您可能會想知道如何區分哪些資料應作為變數傳遞給元件，以及哪些屬性應放置在元件的[屬性包](#component-attributes)中。

您可以使用元件 Blade 模板頂部的 `@props` 指令來指定哪些屬性應被視為資料變數。元件上的所有其他屬性將透過元件的屬性包提供。如果您希望為資料變數提供預設值，您可以將變數名稱指定為陣列鍵，並將預設值指定為陣列值：

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

根據上述元件定義，我們可以這樣渲染元件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

<a name="accessing-parent-data"></a>
### 存取父層資料

有時您可能希望在子元件中存取父層元件的資料。在這些情況下，您可以使用 `@aware` 指令。例如，假設我們正在建立一個由父層 `<x-menu>` 和子層 `<x-menu.item>` 組成的複雜選單元件：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 元件的實現可能如下所示：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

由於 `color` 屬性只傳遞給父層 (`<x-menu>`)，它將不會在 `<x-menu.item>` 中可用。但是，如果我們使用 `@aware` 指令，我們也可以使其在 `<x-menu.item>` 中可用：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware` 指令無法存取未透過 HTML 屬性明確傳遞給父層元件的父層資料。未明確傳遞給父層元件的預設 `@props` 值無法被 `@aware` 指令存取。

<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前所述，匿名元件通常是透過將 Blade 模板放置在 `resources/views/components` 目錄中來定義的。但是，您偶爾可能希望除了預設路徑之外，還向 Laravel 註冊其他匿名元件路徑。

`anonymousComponentPath` 方法接受匿名元件位置的「路徑」作為其第一個參數，以及一個可選的「命名空間」，元件應放置在其下，作為其第二個參數。通常，此方法應在您的應用程式[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::anonymousComponentPath(__DIR__.'/../components');
}
```

當元件路徑註冊時未指定前綴（如上述範例），它們也可以在您的 Blade 元件中沒有對應前綴地渲染。例如，如果 `panel.blade.php` 元件存在於上述註冊的路徑中，它可以這樣渲染：

```blade
<x-panel />
```

可以將前綴「命名空間」作為 `anonymousComponentPath` 方法的第二個參數提供：

```php
Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');
```

當提供前綴時，該「命名空間」內的元件可以在渲染時透過將元件的命名空間作為前綴添加到元件名稱來渲染：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建立佈局


<a name="layouts-using-components"></a>
### 使用元件的佈局

大多數網頁應用程式在各個頁面之間維持相同的一般佈局。如果我們必須在建立的每個視圖中重複整個佈局 HTML，那麼維護我們的應用程式將會難以置信地繁瑣且難以維護。幸運的是，將此佈局定義為單一 [Blade 元件](#components) 並在整個應用程式中使用它會很方便。


<a name="defining-the-layout-component"></a>
#### 定義佈局元件

例如，假設我們正在建立一個「待辦事項」清單應用程式。我們可能會定義一個 `layout` 元件，其外觀如下：

```blade
<!-- resources/views/components/layout.blade.php -->

<html>
    <head>
        <title>{{ $title ?? 'Todo Manager' }}</title>
    </head>
    <body>
        <h1>Todos</h1>
        <hr/>
        {{ $slot }}
    </body>
</html>
```


<a name="applying-the-layout-component"></a>
#### 套用佈局元件

一旦 `layout` 元件被定義，我們就可以建立一個使用該元件的 Blade 視圖。在這個範例中，我們將定義一個簡單的視圖來顯示我們的任務清單：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入到元件中的內容將會提供給我們 `layout` 元件中預設的 `$slot` 變數。如您所見，我們的 `layout` 也會考慮 `$title` 插槽（如果有的話）；否則，會顯示預設標題。我們可以使用 [元件文件](#components) 中討論的標準插槽語法，從任務清單視圖注入自訂標題：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    <x-slot:title>
        Custom Title
    </x-slot>

    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

現在我們已經定義了佈局和任務清單視圖，我們只需要從路由返回 `task` 視圖：

```php
use App\Models\Task;

Route::get('/tasks', function () {
    return view('tasks', ['tasks' => Task::all()]);
});
```


<a name="layouts-using-template-inheritance"></a>
### 使用模板繼承的佈局


<a name="defining-a-layout"></a>
#### 定義佈局

佈局也可以透過「模板繼承」來建立。在 [元件](#components) 引入之前，這是建立應用程式的主要方式。

首先，讓我們看一個簡單的範例。我們將檢視頁面佈局。由於大多數網頁應用程式在各個頁面之間維持相同的一般佈局，因此將此佈局定義為單一 Blade 視圖是很方便的：

```blade
<!-- resources/views/layouts/app.blade.php -->

<html>
    <head>
        <title>App Name - @yield('title')</title>
    </head>
    <body>
        @section('sidebar')
            This is the master sidebar.
        @show

        <div class="container">
            @yield('content')
        </div>
    </body>
</html>
```

如您所見，此檔案包含典型的 HTML 標記。然而，請注意 `@section` 和 `@yield` 指令。`@section` 指令，顧名思義，定義了一個內容區塊，而 `@yield` 指令則用於顯示指定區塊的內容。

現在我們已經為應用程式定義了佈局，讓我們定義一個繼承該佈局的子頁面。


<a name="extending-a-layout"></a>
#### 擴充佈局

定義子視圖時，請使用 `@extends` Blade 指令來指定子視圖應該「繼承」哪個佈局。擴充 Blade 佈局的視圖可以使用 `@section` 指令將內容注入到佈局的區塊中。請記住，如上例所示，這些區塊的內容將使用 `@yield` 顯示在佈局中：

```blade
<!-- resources/views/child.blade.php -->

@extends('layouts.app')

@section('title', 'Page Title')

@section('sidebar')
    @@parent

    <p>This is appended to the master sidebar.</p>
@endsection

@section('content')
    <p>This is my body content.</p>
@endsection
```

在這個範例中，`sidebar` 區塊正在使用 `@@parent` 指令來附加（而非覆寫）內容到佈局的側邊欄。當視圖被渲染時，`@@parent` 指令將被佈局的內容替換。

> [!NOTE]
> 與前一個範例相反，這個 `sidebar` 區塊以 `@endsection` 結束，而不是 `@show`。`@endsection` 指令只會定義一個區塊，而 `@show` 將定義並**立即輸出**該區塊。

`@yield` 指令也接受一個預設值作為它的第二個參數。如果被輸出的區塊未定義，這個值將會被渲染：

```blade
@yield('content', 'Default content')
```


<a name="forms"></a>
## 表單


<a name="csrf-field"></a>
### CSRF 欄位

任何時候你在應用程式中定義一個 HTML 表單，你都應該在表單中包含一個隱藏的 CSRF token 欄位，以便 [CSRF 保護](/docs/{{version}}/csrf) 中介層可以驗證請求。你可以使用 `@csrf` Blade 指令來生成 token 欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```


<a name="method-field"></a>
### Method 欄位

由於 HTML 表單無法發出 `PUT`、`PATCH` 或 `DELETE` 請求，你將需要新增一個隱藏的 `_method` 欄位以偽造這些 HTTP 動詞。`@method` Blade 指令可以為你建立這個欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```


<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用來快速檢查是否存在指定屬性的 [驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令中，你可以輸出 `$message` 變數以顯示錯誤訊息：

```blade
<!-- /resources/views/post/create.blade.php -->

<label for="title">Post Title</label>

<input
    id="title"
    type="text"
    class="@error('title') is-invalid @enderror"
/>

@error('title')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

由於 `@error` 指令會被編譯為「if」條件式，因此你可以使用 `@else` 指令在屬性沒有錯誤時渲染內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror"
/>
```

你可以傳遞 [指定錯誤包的名稱](/docs/{{version}}/validation#named-error-bags) 作為 `@error` 指令的第二個參數，以取得包含多個表單的頁面上的驗證錯誤訊息：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email', 'login') is-invalid @enderror"
/>

@error('email', 'login')
    <div class="alert alert-danger">{{ $message }}</div>
@enderror
```

<a name="stacks"></a>
## Stacks

Blade 允許您將內容推入具名 Stacks 中，這些 Stacks 可以在其他視圖或佈局中的其他位置渲染。這對於指定子視圖所需的任何 JavaScript 函式庫特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

如果您想在給定布林表達式評估為 `true` 時 `@push` 內容，您可以使用 `@pushIf` 指令：

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

您可以根據需要多次推入 Stack。要渲染完整的 Stack 內容，請將 Stack 的名稱傳遞給 `@stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

如果您想將內容前置到 Stack 的開頭，您應該使用 `@prepend` 指令：

```blade
@push('scripts')
    This will be second...
@endpush

// Later...

@prepend('scripts')
    This will be first...
@endprepend
```

`@hasstack` 指令可用來判斷 Stack 是否為空：

```blade
@hasstack('list')
    <ul>
        @stack('list')
    </ul>
@endif
```

<a name="service-injection"></a>
## 服務注入

`@inject` 指令可用來從 Laravel [服務容器](/docs/{{version}}/container)中檢索服務。傳遞給 `@inject` 的第一個引數是服務將被放入的變數名稱，而第二個引數是您要解析的服務的類別或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```

<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 模板

有時您可能需要將原始 Blade 模板字串轉換為有效的 HTML。您可以使用 `Blade` Facade 提供的 `render` 方法來實現此目的。`render` 方法接受 Blade 模板字串和一個可選的資料陣列以提供給模板：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 會將行內 Blade 模板寫入 `storage/framework/views` 目錄進行渲染。如果您想讓 Laravel 在渲染 Blade 模板後刪除這些臨時檔案，您可以將 `deleteCachedView` 引數提供給該方法：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```

<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

使用前端框架，例如 [Turbo](https://turbo.hotwired.dev/) 和 [htmx](https://htmx.org/) 時，您可能偶爾只需要在 HTTP 回應中返回部分 Blade 模板。Blade「片段 (fragments)」允許您實現這一點。首先，將您的 Blade 模板的一部分放入 `@fragment` 和 `@endfragment` 指令中：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

然後，在渲染使用此模板的視圖時，您可以呼叫 `fragment` 方法來指定只有指定的片段應包含在傳出的 HTTP 回應中：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許您根據給定條件有條件地返回視圖片段。否則，將返回整個視圖：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 和 `fragmentsIf` 方法允許您在回應中返回多個視圖片段。這些片段將會被串聯在一起：

```php
view('dashboard', ['users' => $users])
    ->fragments(['user-list', 'comment-list']);

view('dashboard', ['users' => $users])
    ->fragmentsIf(
        $request->hasHeader('HX-Request'),
        ['user-list', 'comment-list']
    );
```

<a name="extending-blade"></a>
## 擴充 Blade

Blade 允許您使用 `directive` 方法定義自己的自訂指令。當 Blade 編譯器遇到自訂指令時，它將使用指令包含的表達式呼叫所提供的回呼函數。

以下範例建立了一個 `@datetime($var)` 指令，它會格式化一個給定的 `$var`，該變數應該是 `DateTime` 的實例：

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Blade;
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
        Blade::directive('datetime', function (string $expression) {
            return "<?php echo ($expression)->format('m/d/Y H:i'); ?>";
        });
    }
}
```

正如您所見，我們將 `format` 方法鏈接到傳入指令的任何表達式上。因此，在此範例中，此指令最終生成的 PHP 將是：

```php
<?php echo ($var)->format('m/d/Y H:i'); ?>
```

> [!WARNING]
> 更新 Blade 指令的邏輯後，您需要刪除所有快取的 Blade 視圖。可以使用 `view:clear` Artisan 命令來刪除快取的 Blade 視圖。

<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理器

如果您嘗試使用 Blade 「echo」一個物件，將會呼叫該物件的 `__toString` 方法。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您互動的類別屬於第三方函式庫時。

在這些情況下，Blade 允許您為該特定物件類型註冊自訂的 echo 處理器。為實現此目的，您應該呼叫 Blade 的 `stringable` 方法。`stringable` 方法接受一個閉包 (closure)。此閉包應類型提示它負責渲染的物件類型。通常，`stringable` 方法應在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
use Illuminate\Support\Facades\Blade;
use Money\Money;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::stringable(function (Money $money) {
        return $money->formatTo('en_GB');
    });
}
```

一旦定義了自訂的 echo 處理器，您就可以簡單地在 Blade 模板中 echo 物件：

```blade
Cost: {{ $money }}
```

<a name="custom-if-statements"></a>
### 自訂 If 條件式

在定義簡單的自訂條件式時，編寫自訂指令有時會比必要更複雜。因此，Blade 提供了一個 `Blade::if` 方法，允許您使用閉包快速定義自訂條件式指令。例如，讓我們定義一個自訂條件式來檢查應用程式設定的預設「disk」。我們可以在 `AppServiceProvider` 的 `boot` 方法中執行此操作：

```php
use Illuminate\Support\Facades\Blade;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::if('disk', function (string $value) {
        return config('filesystems.default') === $value;
    });
}
```

一旦定義了自訂條件式，您就可以在模板中使用它：

```blade
@disk('local')
    <!-- The application is using the local disk... -->
@elsedisk('s3')
    <!-- The application is using the s3 disk... -->
@else
    <!-- The application is using some other disk... -->
@enddisk

@unlessdisk('local')
    <!-- The application is not using the local disk... -->
@enddisk
```