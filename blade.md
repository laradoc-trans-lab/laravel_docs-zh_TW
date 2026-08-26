# Blade 樣板

- [簡介](#introduction)
    - [使用 Livewire 強化 Blade](#supercharging-blade-with-livewire)
- [顯示資料](#displaying-data)
    - [HTML 實體編碼](#html-entity-encoding)
    - [Blade 與 JavaScript 框架](#blade-and-javascript-frameworks)
- [Blade 指令](#blade-directives)
    - [If 敘述](#if-statements)
    - [Switch 敘述](#switch-statements)
    - [迴圈](#loops)
    - [Loop 變數](#the-loop-variable)
    - [條件式 Class](#conditional-classes)
    - [額外屬性](#additional-attributes)
    - [引入子視圖](#including-subviews)
    - [`@once` 指令](#the-once-directive)
    - [原生 PHP](#raw-php)
    - [字型](#fonts)
    - [註解](#comments)
- [元件](#components)
    - [渲染元件](#rendering-components)
    - [Index 元件](#index-components)
    - [傳遞資料至元件](#passing-data-to-components)
    - [元件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [插槽](#slots)
    - [行內元件視圖](#inline-component-views)
    - [動態元件](#dynamic-components)
    - [手動註冊元件](#manually-registering-components)
- [匿名元件](#anonymous-components)
    - [匿名 Index 元件](#anonymous-index-components)
    - [資料屬性 / 屬性](#data-properties-attributes)
    - [存取父元件資料](#accessing-parent-data)
    - [匿名元件路徑](#anonymous-component-paths)
- [建立版面配置](#building-layouts)
    - [使用元件建立版面配置](#layouts-using-components)
    - [使用樣板繼承建立版面配置](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [Method 欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [堆疊](#stacks)
- [服務注入](#service-injection)
- [渲染行內 Blade 樣板](#rendering-inline-blade-templates)
- [渲染 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自訂 Echo 處理函式](#custom-echo-handlers)
    - [自訂 If 敘述](#custom-if-statements)

<a name="introduction"></a>
## 簡介

Blade 是 Laravel 附帶的簡單但強大的樣板引擎。與某些 PHP 樣板引擎不同，Blade 不會限制您在樣板中使用純 PHP 程式碼。事實上，所有 Blade 樣板都會被編譯成純 PHP 程式碼並進行快取，直到它們被修改為止，這意味著 Blade 基本上不會為您的應用程式增加任何額外負擔。Blade 樣板檔案使用 `.blade.php` 副檔名，通常儲存在 `resources/views` 目錄中。

Blade 視圖可以使用全域 `view` 輔助函式從路由或控制器回傳。當然，正如 [視圖](/docs/{{version}}/views) 文件中所述，可以使用 `view` 輔助函式的第二個引數將資料傳遞給 Blade 視圖：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'Finn']);
});
```

<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想將您的 Blade 樣板提升到全新境界，並輕鬆建置動態介面嗎？快來看看 [Laravel Livewire](https://livewire.laravel.com)。Livewire 允許您撰寫具備動態功能的 Blade 元件，這些功能通常只能透過 React、Svelte 或 Vue 等前端框架來實現。這提供了一種絕佳的方法來建置現代化的響應式前端，同時無需面對許多 JavaScript 框架的複雜性、用戶端渲染或建置步驟。

<a name="displaying-data"></a>
## 顯示資料

您可以透過將變數包裹在大括號中，來顯示傳遞給 Blade 視圖的資料。例如，給定以下路由：

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
> Blade 的 `{{ }}` echo 敘述句會自動通過 PHP 的 `htmlspecialchars` 函式處理，以防止 XSS 攻擊。

您不僅限於顯示傳遞給視圖的變數內容。您還可以印出任何 PHP 函式的結果。事實上，您可以將任何您想要的 PHP 程式碼放入 Blade 的 echo 敘述句中：

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade（以及 Laravel 的 `e` 函式）會對 HTML 實體進行雙重編碼（Double Encode）。如果您想停用雙重編碼，可以在 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

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
#### 顯示未轉義的資料

預設情況下，Blade 的 `{{ }}` 敘述句會自動通過 PHP 的 `htmlspecialchars` 函式處理，以防止 XSS 攻擊。如果您不希望資料被轉義，可以使用以下語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 印出由應用程式使用者所提供的內容時要非常小心。顯示使用者提供的資料時，通常應該使用轉義的雙大括號語法，以防止 XSS 攻擊。

<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「大括號」來表示給定的表達式應該顯示在瀏覽器中，因此您可以使用 `@` 符號來告知 Blade 渲染引擎該表達式應保持原樣。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，`@` 符號會被 Blade 移除；但是，`{{ name }}` 表達式將保持原樣不被 Blade 引擎處理，從而允許由您的 JavaScript 框架進行渲染。

`@` 符號也可用於轉義 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```

<a name="rendering-json"></a>
#### 渲染 JSON

有時候您可能會將陣列傳遞給視圖，目的是將其渲染為 JSON 以便初始化 JavaScript 變數。例如：

```php
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

然而，除了手動呼叫 `json_encode` 之外，您也可以使用 `Illuminate\Support\Js::from` 方法。`from` 方法接收與 PHP 的 `json_encode` 函式相同的引數；但是，它會確保產生的 JSON 已經過適當轉義，以便包含在 HTML 引號內。`from` 方法將回傳一個字串 `JSON.parse` JavaScript 敘述句，該敘述句會將給定的物件或陣列轉換為有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

Laravel 應用程式骨架的最新版本包含 `Js` Facade，它可以在 Blade 樣板中方便地使用此功能：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 您應該僅使用 `Js::from` 方法將現有變數渲染為 JSON。Blade 樣板是基於正規表示式進行處理的，嘗試將複雜表達式傳遞給指令可能會導致未預期的失敗。

<a name="the-at-verbatim-directive"></a>
#### `@verbatim` 指令

如果您要在樣板的大部分區域中顯示 JavaScript 變數，您可以將 HTML 包裹在 `@verbatim` 指令中，這樣就不必在每個 Blade echo 敘述句前加上 `@` 符號：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了樣板繼承與顯示資料外，Blade 還為常見的 PHP 控制結構（例如條件敘述與迴圈）提供了便利的快捷方式。這些快捷方式提供了一種非常簡潔優雅的方式來處理 PHP 控制結構，同時也保持與 PHP 對應語法相同的熟悉感。


<a name="if-statements"></a>
### If 敘述

你可以使用 `@if`、`@elseif`、`@else` 和 `@endif` 指令來建構 `if` 敘述。這些指令的功能與它們相對應的 PHP 語法完全相同：

```blade
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif
```

為了方便起見，Blade 還提供了 `@unless` 指令：

```blade
@unless (Auth::check())
    You are not signed in.
@endunless
```

除了已經討論過的條件指令外，`@isset` 和 `@empty` 指令還可以用作其對應 PHP 函式的便捷快捷方式：

```blade
@isset($records)
    // $records is defined and is not null...
@endisset

@empty($records)
    // $records is "empty"...
@endempty
```


<a name="authentication-directives"></a>
#### 認證指令

`@auth` 和 `@guest` 指令可用於快速判斷當前使用者是否已通過[認證](/docs/{{version}}/authentication)或是訪客：

```blade
@auth
    // The user is authenticated...
@endauth

@guest
    // The user is not authenticated...
@endguest
```

如果需要，你可以在使用 `@auth` 和 `@guest` 指令時指定應檢查的認證 Guard：

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

你可以使用 `@production` 指令來檢查應用程式是否正在正式環境 (Production) 中執行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，你可以使用 `@env` 指令來判斷應用程式是否正在特定環境中執行：

```blade
@env('staging')
    // The application is running in "staging"...
@endenv

@env(['staging', 'production'])
    // The application is running in "staging" or "production"...
@endenv
```


<a name="section-directives"></a>
#### Section 指令

你可以使用 `@hasSection` 指令來判斷樣板繼承的區塊 (Section) 是否含有內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

你可以使用 `sectionMissing` 指令來判斷某個區塊是否缺乏內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```


<a name="session-directives"></a>
#### Session 指令

`@session` 指令可用於判斷 [Session](/docs/{{version}}/session) 值是否存在。若該 Session 值存在，則會解析 `@session` 與 `@endsession` 指令之間的樣板內容。在 `@session` 指令的內容中，你可以印出 `$value` 變數來顯示該 Session 值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```


<a name="context-directives"></a>
#### Context 指令

`@context` 指令可用於判斷 [Context](/docs/{{version}}/context) 值是否存在。若該 Context 值存在，則會解析 `@context` 與 `@endcontext` 指令之間的樣板內容。在 `@context` 指令的內容中，你可以印出 `$value` 變數來顯示該 Context 值：

```blade
@context('canonical')
    <link href="{{ $value }}" rel="canonical">
@endcontext
```


<a name="switch-statements"></a>
### Switch 敘述

Switch 敘述可以使用 `@switch`、`@case`、`@break`、`@default` 與 `@endswitch` 指令來建構：

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

除了條件敘述外，Blade 還提供了簡單的指令來處理 PHP 的迴圈結構。同樣地，這些指令的功能與它們對應的 PHP 語法完全相同：

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
> 在對 `foreach` 迴圈進行疊代時，你可以使用 [Loop 變數](#the-loop-variable)來獲取有關迴圈的寶貴資訊，例如目前是否為迴圈的第一次或最後一次疊代。

在使用迴圈時，你也可以使用 `@continue` 和 `@break` 指令來跳過當前疊代或結束迴圈：

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

你也可以在指令宣告中包含跳過或結束的條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```


<a name="the-loop-variable"></a>
### Loop 變數

在對 `foreach` 迴圈進行疊代時，迴圈內部會提供一個 `$loop` 變數。這個變數讓你能夠存取一些有用的資訊，例如當前迴圈的索引以及這是否為迴圈的第一次或最後一次疊代：

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

如果你處於巢狀迴圈中，可以透過 `parent` 屬性存取父層迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數還包含許多其他實用的屬性：

<div class="overflow-auto">

| 屬性 | 說明 |
| ------------------ | ------------------------------------------------------ |
| `$loop->index` | 當前迴圈疊代的索引（從 0 開始）。 |
| `$loop->iteration` | 當前迴圈的疊代次數（從 1 開始）。 |
| `$loop->remaining` | 迴圈中剩餘的疊代次數。 |
| `$loop->count` | 正在疊代的陣列項目總數。 |
| `$loop->first` | 是否為迴圈的第一次疊代。 |
| `$loop->last` | 是否為迴圈的最後一次疊代。 |
| `$loop->even` | 是否為迴圈的偶數次疊代。 |
| `$loop->odd` | 是否為迴圈的奇數次疊代。 |
| `$loop->depth` | 當前迴圈的巢狀層級。 |
| `$loop->parent` | 處於巢狀迴圈時，父層迴圈的變數。 |

</div>

<a name="conditional-classes"></a>
### 條件式 Class

`@class` 指令能有條件地編譯 CSS class 字串。該指令接收一個 class 陣列，陣列的鍵（Key）包含您希望新增的一個或多個 class，而值則為布林運算式。如果陣列元素的鍵為數字，則該 class 永遠會被包含在渲染的 class 列表中：

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

同樣地，`@style` 指令可用於有條件地將行內 CSS 樣式新增至 HTML 元素：

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

為了方便起見，您可以使用 `@checked` 指令來輕鬆指示給定的 HTML 核取方塊（checkbox）輸入是否為「checked」。若提供的條件評估為 `true`，該指令將會印出 `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，`@selected` 指令可用於指示給定的下拉選單選項是否應該為「selected」：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，`@disabled` 指令可用於指示給定的元素是否應該為「disabled」：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

再者，`@readonly` 指令可用於指示給定的元素是否應該為「readonly」：

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

另外，`@required` 指令可用於指示給定的元素是否應該為「required」：

```blade
<input
    type="text"
    name="title"
    value="title"
    @required($user->isAdmin())
/>
```

<a name="including-subviews"></a>
### 引入子視圖

> [!NOTE]
> 雖然您可以自由使用 `@include` 指令，但 Blade [元件](#components) 提供了類似的功能，且相比 `@include` 指令擁有諸多優勢，例如資料與屬性綁定。

Blade 的 `@include` 指令允許您在一個視圖中引入另一個 Blade 視圖。父視圖中可用的所有變數都可以在被引入的視圖中使用：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

即使被引入的視圖會繼承父視圖中可用的所有資料，您也可以傳遞一個額外資料的陣列，使其在被引入的視圖中可用：

```blade
@include('view.name', ['status' => 'complete'])
```

如果您嘗試 `@include` 一個不存在的視圖，Laravel 將會拋出錯誤。如果您想引入一個可能存在也可能不存在的視圖，應該使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

如果您想在給定的布林運算式評估為 `true` 或 `false` 時才 `@include` 視圖，可以使用 `@includeWhen` 與 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

若要從給定的視圖陣列中引入第一個存在的視圖，您可以使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

如果您想引入視圖而不繼承父視圖的任何變數，可以使用 `@includeIsolated` 指令。被引入的視圖將只能存取您明確傳遞的變數：

```blade
@includeIsolated('view.name', ['user' => $user])
```

> [!WARNING]
> 您應該避免在 Blade 視圖中使用 `__DIR__` 與 `__FILE__` 常數，因為它們將會指向快取、編譯後的視圖位置。

<a name="rendering-views-for-collections"></a>
#### 為集合渲染視圖

您可以使用 Blade 的 `@each` 指令將迴圈與引入組合成一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個引數是用於為陣列或集合中的每個元素渲染的視圖。第二個引數是您想要迭代的陣列或集合，而第三個引數是在視圖中分配給當前迭代的變數名稱。因此，舉例來說，如果您正在迭代一個 `jobs` 陣列，通常您會想在視圖中以 `job` 變數來存取每個工作。當前迭代的陣列鍵（Key）將會在視圖中作為 `key` 變數使用。

您也可以向 `@each` 指令傳遞第四個引數。該引數指定了當給定的陣列為空時要渲染的視圖。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 `@each` 渲染的視圖不會繼承父視圖的變數。如果子視圖需要這些變數，您應該改為使用 `@foreach` 與 `@include` 指令。

<a name="the-once-directive"></a>
### `@once` 指令

`@once` 指令允許您定義樣板中在每個渲染週期內僅會被評估一次的部分。這對於使用[堆疊](#stacks)將給定的 JavaScript 片段推送至頁面的標頭（Header）特別有用。例如，如果您在迴圈中渲染一個給定的[元件](#components)，您可能只希望在首次渲染元件時將 JavaScript 推送到標頭：

```blade
@once
    @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush
@endonce
```

由於 `@once` 指令通常與 `@push` 或 `@prepend` 指令結合使用，因此為了方便起見，提供了 `@pushOnce` 與 `@prependOnce` 指令：

```blade
@pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
@endPushOnce
```

如果您從兩個獨立的 Blade 樣板推送重複的內容，應該向 `@pushOnce` 指令的第二個引數提供一個唯一的識別碼，以確保內容僅被渲染一次：

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

在某些情況下，將 PHP 程式碼嵌入到視圖中很有用。你可以使用 Blade 的 `@php` 指令在樣板中執行一段原生的 PHP 區塊：

```blade
@php
    $counter = 1;
@endphp
```

或者，若你只需要使用 PHP 來匯入類別，可以使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

可以為 `@use` 指令提供第二個引數，來為匯入的類別設定別名：

```blade
@use('App\Models\Flight', 'FlightModel')
```

若在同一個命名空間下有多個類別，你可以將這些類別的匯入進行群組化：

```blade
@use('App\Models\{Flight, Airport}')
```

`@use` 指令也支援透過在匯入路徑前加上 `function` 或 `const` 修飾詞來匯入 PHP 函式與常數：

```blade
@use(function App\Helpers\format_currency)
@use(const App\Constants\MAX_ATTEMPTS)
```

就像類別匯入一樣，函式與常數也同樣支援別名：

```blade
@use(function App\Helpers\format_currency, 'formatMoney')
@use(const App\Constants\MAX_ATTEMPTS, 'MAX_TRIES')
```

群組化匯入也同時支援 function 與 const 修飾詞，讓你可以在單一指令中從同一個命名空間匯入多個符號：

```blade
@use(function App\Helpers\{format_currency, format_date})
@use(const App\Constants\{MAX_ATTEMPTS, DEFAULT_TIMEOUT})
```


<a name="fonts"></a>
### 字型

使用 [Laravel 的 Vite 字型最佳化](/docs/{{version}}/vite#working-with-fonts) 時，你可以使用 `@fonts` 指令在應用程式的版面配置中渲染已設定的字型預載連結與行內字型 CSS：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    @fonts
    @vite('resources/js/app.js')
</head>
```

`@fonts` 指令會渲染在 `vite.config.js` 檔案中設定的所有字型家族。該指令通常應該放置在應用程式根版面配置的 `<head>` 中，且位於任何使用這些字型內容之前。

如果頁面只需要部分已設定的字型，你可以將一個或多個字型別名傳遞給該指令：

```blade
{{-- Load a single font alias... --}}
@fonts('sans')

{{-- Load multiple font aliases... --}}
@fonts(['sans', 'mono'])
```

字型別名是在 Vite 設定中定義字型時使用 `alias` 選項進行設定的。`@fonts` 指令會呼叫 `Vite` Facade 所提供的 `fonts` 方法，該方法也可以直接被調用：

```blade
{{ Vite::fonts(['sans', 'mono']) }}
```


<a name="comments"></a>
### 註解

Blade 也允許你在視圖中定義註解。然而，與 HTML 註解不同的是，Blade 註解不會包含在應用程式回傳的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件 (Components) 與插槽 (Slots) 提供與 Section、Layout 及 Include 類似的好處；然而，有些人可能覺得元件和插槽的心智模型更為易懂。撰寫元件有兩種方法：類別型元件 (class-based components) 與匿名元件 (anonymous components)。

若要建立類別型元件，您可以使用 `make:component` Artisan 指令。為了說明如何使用元件，我們將建立一個簡單的 `Alert` 元件。`make:component` 指令會將元件放置於 `app/View/Components` 目錄中：

```shell
php artisan make:component Alert
```

`make:component` 指令也會為該元件建立一個視圖樣板。該視圖將放置於 `resources/views/components` 目錄中。當為您自己的應用程式撰寫元件時，元件會自動在 `app/View/Components` 目錄與 `resources/views/components` 目錄中被自動尋找，因此通常不需要額外的元件註冊。

您也可以在子目錄中建立元件：

```shell
php artisan make:component Forms/Input
```

上述指令將會在 `app/View/Components/Forms` 目錄中建立一個 `Input` 元件，且視圖會放置於 `resources/views/components/forms` 目錄中。


<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

當為您自己的應用程式撰寫元件時，元件會在 `app/View/Components` 目錄與 `resources/views/components` 目錄中自動被尋找。

但是，若您正在建置一個使用 Blade 元件的套件，則需要手動註冊元件類別及其 HTML 標籤別名。您通常應該在套件服務提供者 (Service Provider) 的 `boot` 方法中註冊您的元件：

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

元件註冊完成後，即可使用其標籤別名進行渲染：

```blade
<x-package-alert/>
```

或者，您可以使用 `componentNamespace` 方法，依慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能具有位於 `Package\Views\Components` 命名空間下的 `Calendar` 和 `ColorPicker` 元件：

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

這將允許使用 `package-name::` 語法，透過套件的 Vendor 命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將元件名稱轉為 PascalCase (帕斯卡命名法) 來自動偵測連結到該元件的類別。也支援使用「點 (dot)」記法來指定子目錄。


<a name="rendering-components"></a>
### 渲染元件

若要顯示元件，可以在 Blade 樣板中使用 Blade 元件標籤。Blade 元件標籤以字串 `x-` 開頭，後跟元件類別的 kebab-case 名稱：

```blade
<x-alert/>

<x-user-profile/>
```

若元件類別嵌套在 `app/View/Components` 目錄更深處，您可以使用 `.` 字元來表示目錄嵌套。例如，假設元件位於 `app/View/Components/Inputs/Button.php`，我們可以這樣渲染它：

```blade
<x-inputs.button/>
```

如果您想根據條件渲染元件，可以在元件類別上定義 `shouldRender` 方法。若 `shouldRender` 方法傳回 `false`，則該元件不會被渲染：

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
### Index 元件

有時候元件是元件群組的一部份，您可能希望將相關元件整理在單一目錄中。例如，想像一個具有以下類別結構的「卡片 (card)」元件：

```text
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件嵌套在 `Card` 目錄中，您可能會認為需要透過 `<x-card.card>` 來渲染該元件。然而，當元件的檔案名稱與元件目錄名稱相同時，Laravel 會自動將該元件視為「根」元件，並允許您在渲染該元件時無需重複目錄名稱：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料至元件

您可以使用 HTML 屬性將資料傳遞給 Blade 元件。硬編碼（Hard-coded）的基本型態值可以使用簡單的 HTML 屬性字串傳遞給元件。PHP 運算式與變數則應透過前綴為 `:` 字元的屬性傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

您應該在元件類別的建構函式中定義所有的元件資料屬性。元件上的所有公開 (public) 屬性都會自動提供給元件的視圖使用。不需要從元件的 `render` 方法傳遞資料至視圖：

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

當渲染元件時，您可以透過印出與名稱相符的變數來顯示元件公開變數的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```

<a name="casing"></a>
#### 大小寫範式

元件建構函式的引數應使用 `camelCase`（小駝峰拼寫），而在 HTML 屬性中引用引數名稱時應使用 `kebab-case`（短條線連接）。例如，給定以下元件建構函式：

```php
/**
 * Create the component instance.
 */
public function __construct(
    public string $alertType,
) {}
```

可像這樣將 `$alertType` 引數提供給元件：

```blade
<x-alert alert-type="danger" />
```

<a name="short-attribute-syntax"></a>
#### 簡短屬性語法

將屬性傳遞給元件時，您也可以使用「簡短屬性」語法。這通常很方便，因為屬性名稱往往與它們所對應的變數名稱相同：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```

<a name="escaping-attribute-rendering"></a>
#### 轉義屬性渲染

由於某些 JavaScript 框架（例如 Alpine.js）也使用冒號開頭的屬性，您可以透過雙冒號 (`::`) 前綴告知 Blade 該屬性並非 PHP 運算式。例如，給定以下元件：

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

Blade 將會渲染出以下的 HTML：

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```

<a name="component-methods"></a>
#### 元件方法

除了在元件樣板中可使用的公開變數外，您也可以呼叫元件上的任何公開方法。例如，假設有一個具有 `isSelected` 方法的元件：

```php
/**
 * Determine if the given option is the currently selected option.
 */
public function isSelected(string $option): bool
{
    return $option === $this->selected;
}
```

您可以透過呼叫與方法名稱相符的變數，從元件樣板中執行該方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```

<a name="using-attributes-slots-within-component-class"></a>
#### 在元件類別內存取屬性與插槽

Blade 元件也允許您在類別的 render 方法內部存取元件名稱、屬性與插槽。然而，為了存取這些資料，您應該從元件的 `render` 方法回傳一個閉包 (Closure)：

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

您元件的 `render` 方法所回傳的閉包也可以接收一個 `$data` 陣列作為其唯一的引數。該陣列將包含數個提供元件相關資訊的元素：

```php
return function (array $data) {
    // $data['componentName'];
    // $data['attributes'];
    // $data['slot'];

    return '<div {{ $attributes }}>Components content</div>';
}
```

> [!WARNING]
> 絕對不要將 `$data` 陣列中的元素直接內嵌至 `render` 方法回傳的 Blade 字串中，因為這樣做可能會允許他人透過惡意的屬性內容進行遠端程式碼執行 (RCE)。

`componentName` 等於在 HTML 標籤中 `x-` 前綴後所使用的名稱。因此 `<x-alert />` 的 `componentName` 為 `alert`。`attributes` 元素包含 HTML 標籤上存在的所有屬性。`slot` 元素是一個包含元件插槽內容的 `Illuminate\Support\HtmlString` 實例。

該閉包應回傳一個字串。若回傳的字串對應至現有的視圖，則會渲染該視圖；否則，回傳的字串將作為行內 Blade 視圖來評估解析。

<a name="additional-dependencies"></a>
#### 額外依賴

若您的元件需要來自 Laravel [服務容器](/docs/{{version}}/container)的依賴項目，您可以將它們列在任何元件資料屬性之前，服務容器便會自動注入它們：

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

若您想防止某些公開方法或屬性作為變數暴露給元件樣板，您可以將它們新增至元件上的 `$except` 陣列屬性中：

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

我們已經探討過如何將資料屬性傳遞給元件；然而，有時您可能需要指定額外的 HTML 屬性（例如 `class`），這些屬性並非元件運作所需的資料的一部分。通常，您會希望將這些額外的屬性傳遞到元件樣板的根元素。例如，假設我們想要像這樣渲染一個 `alert` 元件：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於元件建構函式部分的屬性都會自動被加入到元件的「屬性包 (attribute bag)」中。這個屬性包會透過 `$attributes` 變數自動提供給元件使用。您可以在元件內印出此變數來渲染所有屬性：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前不支援在元件標籤中使用如 `@env` 之類的指令。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時您可能需要為屬性指定預設值，或將額外的值合併至元件的某些屬性中。要達到這個目的，您可以使用屬性包的 `merge` 方法。這個方法在定義一套應該總是套用到元件上的預設 CSS class 時特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

如果我們假設這個元件的使用方式如下：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

元件最終渲染出的 HTML 將如下所示：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 依條件合併 Class

有時您可能希望在給定條件為 `true` 時才合併 class。您可以透過 `class` 方法來達到這個目的，該方法接受一個 class 陣列，其中陣列鍵（Key）包含您想要新增的 class，而值（Value）為布林運算式。若陣列元素擁有數字鍵，則它將總是包含在渲染出的 class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

如果您需要將其他屬性合併至元件中，可以將 `merge` 方法鏈結（Chain）到 `class` 方法後面：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 如果您需要在其他不應接收合併屬性的 HTML 元素上依條件編譯 class，您可以使用 [@class 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 非 Class 屬性合併

當合併非 `class` 的屬性時，提供給 `merge` 方法的值將被視為該屬性的「預設」值。然而，與 `class` 屬性不同的是，這些屬性不會與注入的屬性值進行合併，而是會被覆寫。例如，`button` 元件的實作可能如下所示：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要以自訂的 `type` 渲染按鈕元件，可以在使用該元件時進行指定。如果未指定型態，將會使用 `button` 型態：

```blade
<x-button type="submit">
    Submit
</x-button>
```

此範例中 `button` 元件渲染後的 HTML 將為：

```blade
<button type="submit">
    Submit
</button>
```

如果您希望 `class` 以外的屬性能將其預設值與注入的值串接在一起，您可以使用 `prepends` 方法。在此範例中，`data-controller` 屬性將永遠以 `profile-controller` 開頭，而任何額外注入的 `data-controller` 值都將放在這個預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 取得與過濾屬性

您可以使用 `filter` 方法來過濾屬性。該方法接受一個閉包 (Closure)，若您希望將屬性保留在屬性包中，該閉包應傳回 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為方便起見，您可以使用 `whereStartsWith` 方法來取得所有 Key 以指定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，`whereDoesntStartWith` 方法可用於排除所有 Key 以指定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，您可以渲染指定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

如果您想檢查元件上是否存在某個屬性，可以使用 `has` 方法。該方法接受屬性名稱作為其唯一的引數，並傳回布林值表示該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

如果傳遞一個陣列給 `has` 方法，該方法將判斷所有指定的屬性是否都存在於元件上：

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

`hasAny` 方法可用於判斷指定的任何一個屬性是否存在於元件上：

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

您可以使用 `get` 方法取得特定屬性的值：

```blade
{{ $attributes->get('class') }}
```

`only` 方法可用於僅取得具有指定 Key 的屬性：

```blade
{{ $attributes->only(['class']) }}
```

`except` 方法可用於取得除了具有指定 Key 之外的所有屬性：

```blade
{{ $attributes->except(['class']) }}
```

<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，有些關鍵字保留給 Blade 內部用於渲染元件。以下關鍵字不能在您的元件中定義為公開屬性或方法名稱：

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

您經常需要透過「插槽 (slots)」將額外的內容傳遞給元件。元件插槽是透過輸出 `$slot` 變數來渲染的。為了深入瞭解這個概念，讓我們假設 `alert` 元件包含以下標記：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過將內容注入元件中，以將內容傳遞給 `slot`：

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時候，元件可能需要在元件內部的不同位置渲染多個不同的插槽。讓我們修改 alert 元件，以允許注入一個 "title" 插槽：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

您可以使用 `x-slot` 標籤來定義具名插槽的內容。任何不在明確的 `x-slot` 標籤內的內容都會在 `$slot` 變數中傳遞給元件：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

您可以呼叫插槽的 `isEmpty` 方法來判斷插槽是否包含內容：

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

此外，`hasActualContent` 方法可用於判斷插槽是否包含任何非 HTML 註解的「實際」內容：

```blade
@if ($slot->hasActualContent())
    The scope has non-comment content.
@endif
```


<a name="scoped-slots"></a>
#### 作用域插槽

如果您使用過諸如 Vue 之類的 JavaScript 框架，您可能熟悉「作用域插槽 (scoped slots)」，它允許您在插槽內存取來自元件的資料或方法。在 Laravel 中，您可以透過在元件上定義公用方法或屬性，並在插槽內透過 `$component` 變數存取該元件，以實現類似的行為。在這個範例中，我們假設 `x-alert` 元件在其元件類別上定義了一個公用的 `formatAlert` 方法：

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

如同 Blade 元件，您可以為插槽分配額外的[屬性](#component-attributes)，例如 CSS class 名稱：

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

要與插槽屬性進行互動，您可以存取插槽變數的 `attributes` 屬性。關於如何與屬性互動的更多資訊，請參閱[元件屬性](#component-attributes)的說明文件：

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

對於非常小的元件，同時管理元件類別和元件的視圖樣板可能會讓人覺得繁瑣。因此，您可以直接從 `render` 方法傳回元件的標記：

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
#### 產生行內視圖元件

要建立渲染行內視圖的元件，可以在執行 `make:component` 指令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```


<a name="dynamic-components"></a>
### 動態元件

有時候您可能需要渲染某個元件，但在執行階段前並不知道該渲染哪個元件。在這種情況下，您可以使用 Laravel 內建的 `dynamic-component` 元件，根據執行階段的值或變數來渲染元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```


<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 以下關於手動註冊元件的說明文件，主要適用於撰寫包含視圖元件的 Laravel 套件開發者。如果您不是在撰寫套件，本區塊的元件說明文件可能與您無關。

為您自己的應用程式撰寫元件時，元件會在 `app/View/Components` 目錄和 `resources/views/components` 目錄中被自動偵測。

但是，如果您正在建立使用 Blade 元件的套件，或將元件放置在非傳統目錄中，您將需要手動註冊元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡可以找到該元件。您通常應該在套件的服務提供者 (Service Providers) 的 `boot` 方法中註冊元件：

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

或者，您可以使用 `componentNamespace` 方法按照慣例自動載入元件類別。例如，`Nightshade` 套件可能具有位於 `Package\Views\Components` 命名空間內的 `Calendar` 與 `ColorPicker` 元件：

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

這將允許使用 `package-name::` 語法並透過 Vendor 命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將元件名稱轉為 PascalCase 來自動偵測連結到該元件的類別。使用「點」號表示法也支援子目錄。

<a name="anonymous-components"></a>
## 匿名元件

與行內元件類似，匿名元件提供了一種透過單一檔案管理元件的機制。然而，匿名元件使用單一視圖檔案，且沒有關聯的類別。要定義匿名元件，您只需要將 Blade 樣板放置在 `resources/views/components` 目錄中即可。例如，假設您在 `resources/views/components/alert.blade.php` 定義了一個元件，您可以像這樣直接渲染它：

```blade
<x-alert/>
```

您可以使用 `.` 字元來標示元件是否巢狀嵌入在 `components` 目錄更深層的位置。例如，假設該元件定義於 `resources/views/components/inputs/button.blade.php`，您可以像這樣渲染它：

```blade
<x-inputs.button/>
```

若要透過 Artisan 建立匿名元件，可以在執行 `make:component` 命令時使用 `--view` 標誌：

```shell
php artisan make:component forms.input --view
```

上述命令將會在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，並可透過 `<x-forms.input />` 將其作為元件來渲染。


<a name="anonymous-index-components"></a>
### 匿名 Index 元件

有時，當一個元件由多個 Blade 樣板組成時，您可能會希望將該元件的樣板群組化在單一目錄中。例如，想像一個具有以下目錄結構的「accordion」元件：

```text
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這樣的目錄結構讓您可以像這樣渲染 accordion 元件及其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

然而，為了透過 `x-accordion` 來渲染 accordion 元件，我們被迫將「index」accordion 元件樣板放在 `resources/views/components` 目錄中，而不是將其與其他 accordion 相關樣板一同巢狀放置在 `accordion` 目錄下。

幸運的是，Blade 允許您在元件目錄本身之中，放置一個與元件目錄名稱相同的檔案。當此樣板存在時，即使它巢狀放置於目錄內，也能作為該元件的「根」元素來渲染。因此，我們可以繼續使用上述範例中相同的 Blade 語法；不過，我們會將目錄結構調整如下：

```text
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```


<a name="data-properties-attributes"></a>
### 資料屬性 / 屬性

由於匿名元件沒有任何關聯的類別，您可能會好奇該如何區分哪些資料應該作為變數傳遞給元件，以及哪些屬性應該放入元件的[屬性包](#component-attributes)中。

您可以在元件 Blade 樣板的頂端使用 `@props` 指令，以指定哪些屬性應被視為資料變數。元件上的所有其他屬性將可透過元件的屬性包來取得。如果您想給予資料變數預設值，可以將變數名稱指定為陣列鍵名，並將預設值指定為陣列的值：

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

根據上述元件定義，我們可以像這樣渲染該元件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```


<a name="accessing-parent-data"></a>
### 存取父元件資料

有時您可能想要在子元件內部存取父元件的資料。在這些情況下，您可以使用 `@aware` 指令。例如，想像我們正在建立一個由父元件 `<x-menu>` 和子元件 `<x-menu.item>` 組成的複雜選單元件：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 元件的實作可能如下所示：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

因為 `color` 屬性僅傳遞至父元件 (`<x-menu>`)，所以在 `<x-menu.item>` 內部無法直接存取。但是，如果我們使用 `@aware` 指令，就可以讓它在 `<x-menu.item>` 內部也能被存取：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware` 指令無法存取未透過 HTML 屬性明確傳遞給父元件的父元件資料。未明確傳遞給父元件的預設 `@props` 值，無法由 `@aware` 指令存取。


<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前所述，匿名元件通常是透過將 Blade 樣板放置在 `resources/views/components` 目錄中來定義的。然而，除了預設路徑之外，您偶爾可能會想在 Laravel 中註冊其他的匿名元件路徑。

`anonymousComponentPath` 方法的第一個引數接收匿名元件位置的「路徑」，第二個選擇性引數則接收元件應歸屬的「命名空間」。通常，此方法應在應用程式的其中一個[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中呼叫：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::anonymousComponentPath(__DIR__.'/../components');
}
```

當註冊元件路徑時未指定前綴（如上例所示），它們也可以在您的 Blade 元件中渲染，而不必加上對應的前綴。例如，如果上述註冊的路徑中存在一個 `panel.blade.php` 元件，它可以像這樣渲染：

```blade
<x-panel />
```

前綴「命名空間」可以作為第二個引數提供給 `anonymousComponentPath` 方法：

```php
Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');
```

當提供前綴時，在渲染元件時，只要將元件的命名空間作為前綴附加在元件名稱前，即可渲染該「命名空間」內的元件：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建立版面配置


<a name="layouts-using-components"></a>
### 使用元件建立版面配置

大多數 Web 應用程式在各個頁面之間都保持相同的總體版面配置。如果我們必須在建立的每個視圖中重複撰寫整個版面配置 HTML，這將會非常麻煩且難以維護。幸好，將此版面配置定義為單一 [Blade 元件](#components)並在整個應用程式中使用它是相當方便的。


<a name="defining-the-layout-component"></a>
#### 定義版面配置元件

例如，假設我們正在建立一個「待辦事項 (todo)」清單應用程式。我們可能會定義一個如下所示的 `layout` 元件：

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
#### 套用版面配置元件

一旦定義了 `layout` 元件，我們就可以建立使用該元件的 Blade 視圖。在此範例中，我們將定義一個顯示任務清單的簡單視圖：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入到元件中的內容將會提供給我們 `layout` 元件內的預設 `$slot` 變數。您可能已經注意到，如果提供了 `$title` 插槽，我們的 `layout` 也會使用它；否則，會顯示預設標題。我們可以使用[元件文件](#components)中討論的標準插槽語法，從任務清單視圖注入自訂標題：

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

現在我們已經定義了版面配置與任務清單視圖，我們只需要從路由返回 `task` 視圖：

```php
use App\Models\Task;

Route::get('/tasks', function () {
    return view('tasks', ['tasks' => Task::all()]);
});
```


<a name="layouts-using-template-inheritance"></a>
### 使用樣板繼承建立版面配置


<a name="defining-a-layout"></a>
#### 定義版面配置

版面配置也可以透過「樣板繼承」來建立。在引入[元件](#components)之前，這是建構應用程式的主要方式。

首先，讓我們看一個簡單的範例。我們將檢視頁面版面配置。由於大多數 Web 應用程式在各個頁面之間都保持相同的總體版面配置，因此將此版面配置定義為單一 Blade 視圖會非常方便：

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

如您所見，此檔案包含常見的 HTML 標記。但是，請注意 `@section` 和 `@yield` 指令。顧名思義，`@section` 指令定義了一段內容區段，而 `@yield` 指令則用於顯示指定區段的內容。

現在我們已經為應用程式定義了版面配置，接著讓我們定義繼承該版面配置的子頁面。


<a name="extending-a-layout"></a>
#### 擴充版面配置

定義子視圖時，請使用 `@extends` Blade 指令來指定子視圖應該「繼承」哪一個版面配置。擴充 Blade 版面配置的視圖可以使用 `@section` 指令將內容注入到版面配置的區段中。請記住，如上例所示，這些區段的內容將會使用 `@yield` 顯示在版面配置中：

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

在此範例中，`sidebar` 區段利用 `@@parent` 指令將內容附加（而非覆寫）到版面配置的側邊欄。當渲染視圖時，`@@parent` 指令將會被版面配置的內容所替換。

> [!NOTE]
> 與先前的範例相反，此 `sidebar` 區段以 `@endsection` 結尾，而非 `@show`。`@endsection` 指令只會定義區段，而 `@show` 則會定義並**立即呈現 (yield)** 該區段。

`@yield` 指令還接受第二個參數作為預設值。如果要呈現的區段未定義，則會渲染此值：

```blade
@yield('content', 'Default content')
```


<a name="forms"></a>
## 表單


<a name="csrf-field"></a>
### CSRF 欄位

每當您在應用程式中定義 HTML 表單時，都應該在表單中包含一個隱藏的 CSRF 令牌欄位，以便 [CSRF 保護](/docs/{{version}}/csrf)中介層可以驗證請求。您可以使用 `@csrf` Blade 指令來產生令牌欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```


<a name="method-field"></a>
### Method 欄位

由於 HTML 表單無法發送 `PUT`、`PATCH` 或 `DELETE` 請求，因此您需要新增一個隱藏的 `_method` 欄位來模擬這些 HTTP 動詞。`@method` Blade 指令可以為您建立此欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```


<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用於快速檢查特定屬性是否存在[驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令內部，您可以印出 `$message` 變數來顯示錯誤訊息：

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

由於 `@error` 指令會編譯為「if」敘述，因此當屬性沒有錯誤時，您可以使用 `@else` 指令來渲染內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror"
/>
```

您可以將[特定錯誤包的名稱](/docs/{{version}}/validation#named-error-bags)作為第二個參數傳遞給 `@error` 指令，以取得包含多個表單的頁面上的驗證錯誤訊息：

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
## 堆疊

Blade 允許您推送到具名堆疊，這些堆疊可以在其他視圖或版面配置的任何地方進行渲染。這對於指定子視圖所需的任何 JavaScript 函式庫特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

若您希望在給定的布林運算式計算為 `true` 時才 `@push` 內容，可以使用 `@pushIf` 指令：

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

您可以根據需要多次推送到堆疊。若要渲染完整的堆疊內容，請將堆疊名稱傳遞給 `@stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

若您想將內容推送到堆疊的開頭，應該使用 `@prepend` 指令：

```blade
@push('scripts')
    This will be second...
@endpush

// Later...

@prepend('scripts')
    This will be first...
@endprepend
```

`@hasstack` 指令可以用來判斷堆疊是否包含內容：

```blade
@hasstack('list')
    <ul>
        @stack('list')
    </ul>
@endif
```


<a name="service-injection"></a>
## 服務注入

`@inject` 指令可以用來從 Laravel 的[服務容器](/docs/{{version}}/container)中取得服務。傳遞給 `@inject` 的第一個引數是服務將放置其中的變數名稱，而第二個引數是您希望解析的服務之類別名稱或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```


<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 樣板

有時您可能需要將原始的 Blade 樣板字串轉換為有效的 HTML。您可以使用 `Blade` Facade 提供的 `render` 方法來完成此操作。`render` 方法接受 Blade 樣板字串以及提供給樣板的選填資料陣列：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 會透過將行內 Blade 樣板寫入 `storage/framework/views` 目錄來進行渲染。若您希望 Laravel 在渲染 Blade 樣板後移除這些暫存檔案，可以向該方法提供 `deleteCachedView` 引數：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```


<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

當使用前端框架（例如 [Turbo](https://turbo.hotwired.dev/) 和 [htmx](https://htmx.org/)）時，您偶爾可能只需要在 HTTP 回應中傳回 Blade 樣板的一部分。Blade「片段 (Fragments)」允許您做到這一點。首先，將 Blade 樣板的一部分放置在 `@fragment` 和 `@endfragment` 指令之間：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

然後，在渲染使用此樣板的視圖時，您可以呼叫 `fragment` 方法，指定傳出的 HTTP 回應中應僅包含特定的片段：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許您根據給定條件有條件地傳回視圖的片段。否則，將傳回整個視圖：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 和 `fragmentsIf` 方法允許您在回應中傳回多個視圖片段。這些片段將會串接在一起：

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

Blade 允許您使用 `directive` 方法定義自訂指令。當 Blade 編譯器遇到自訂指令時，它會呼叫提供的回呼函式，並傳入該指令包含的運算式。

以下範例建立了一個 `@datetime($var)` 指令，該指令用於格式化給定的 `$var`（該變數應為 `DateTime` 的實例）：

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

如您所見，我們將 `format` 方法串接至傳入該指令的任何運算式上。因此在此範例中，此指令生成的最終 PHP 將會是：

```php
<?php echo ($var)->format('m/d/Y H:i'); ?>
```

> [!WARNING]
> 更新 Blade 指令的邏輯後，您需要刪除所有快取的 Blade 視圖。可以使用 `view:clear` Artisan 命令移除快取的 Blade 視圖。


<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理函式

若您嘗試使用 Blade 印出一個物件，該物件的 `__toString` 方法將會被呼叫。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您互動的類別屬於第三方函式庫時。

在這些情況下，Blade 允許您為該特定型態的物件註冊自訂 echo 處理函式。若要完成此操作，您應該呼叫 Blade 的 `stringable` 方法。`stringable` 方法接受一個閉包。此閉包應型態提示 (Type-hint) 其負責渲染的物件型態。通常，`stringable` 方法應在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

一旦定義了自訂的 echo 處理函式，您就可以在 Blade 樣板中直接印出該物件：

```blade
Cost: {{ $money }}
```


<a name="custom-if-statements"></a>
### 自訂 If 敘述

在定義簡單的自訂條件敘述時，編寫自訂指令有時比必要的更加複雜。因此，Blade 提供了 `Blade::if` 方法，讓您能使用閉包快速定義自訂條件指令。例如，讓我們定義一個自訂條件，用於檢查應用程式設定的預設「磁碟 (disk)」。我們可以在 `AppServiceProvider` 的 `boot` 方法中執行此操作：

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

定義好自訂條件後，您就可以在樣板中使用它：

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