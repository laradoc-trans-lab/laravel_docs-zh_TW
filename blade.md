# Blade 模板

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
    - [傳送資料至元件](#passing-data-to-components)
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
    - [使用模板繼承建立版面配置](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [Method 欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [堆疊](#stacks)
- [服務注入](#service-injection)
- [渲染行內 Blade 模板](#rendering-inline-blade-templates)
- [渲染 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自訂 Echo 處理程序](#custom-echo-handlers)
    - [自訂 If 敘述](#custom-if-statements)

<a name="introduction"></a>
## 簡介

Blade 是 Laravel 附帶的簡單但強大的模板引擎。與某些 PHP 模板引擎不同，Blade 不會限制您在模板中使用原生的 PHP 程式碼。事實上，所有 Blade 模板都會被編譯成原生的 PHP 程式碼並進行快取，直到它們被修改為止，這意味著 Blade 基本上不會為您的應用程式帶來任何額外負擔。Blade 模板檔案使用 `.blade.php` 檔案副檔名，通常儲存在 `resources/views` 目錄中。

Blade 視圖可以透過全域 `view` 輔助函式從路由或控制器回傳。當然，如同[視圖](/docs/{{version}}/views)文件所述，可以使用 `view` 輔助函式的第二個引數將資料傳遞給 Blade 視圖：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'Finn']);
});
```

<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想讓您的 Blade 模板更上一層樓，並輕鬆建立動態介面嗎？快來看看 [Laravel Livewire](https://livewire.laravel.com)。Livewire 讓您能夠撰寫具備動態功能的 Blade 元件，而這些功能通常只能透過像 React、Svelte 或 Vue 這類前端框架來實現。它提供了一種絕佳的方法來構建現代化、反應式的前端，而且無需承受許多 JavaScript 框架的複雜性、用戶端渲染或建置步驟。

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
> Blade 的 `{{ }}` 印出敘述會自動傳遞給 PHP 的 `htmlspecialchars` 函式處理，以防止 XSS 攻擊。

您不限於顯示傳遞給視圖的變數內容。您也可以印出任何 PHP 函式的結果。事實上，您可以將您希望的任何 PHP 程式碼放進 Blade 印出敘述中：

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade（以及 Laravel 的 `e` 函式）會對 HTML 實體進行雙重編碼。如果您想要停用雙重編碼，請從 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

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
#### 顯示未轉義資料

預設情況下，Blade `{{ }}` 敘述會自動傳遞給 PHP 的 `htmlspecialchars` 函式處理，以防止 XSS 攻擊。如果您不希望資料被轉義，可以使用以下語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 印出應用程式使用者提供的內容時要非常小心。在顯示使用者提供的資料時，通常應該使用會自動轉義的雙花括號語法，以防止 XSS 攻擊。

<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「花括號」來表示應該在瀏覽器中顯示給定的運算式，因此您可以使用 `@` 符號來告知 Blade 渲染引擎該運算式應保持原樣不變。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，`@` 符號會被 Blade 移除；然而，`{{ name }}` 運算式會保持原樣而不被 Blade 引擎處理，從而允許它由您的 JavaScript 框架進行渲染。

`@` 符號也可以用來轉義 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```

<a name="rendering-json"></a>
#### 渲染 JSON

有時您可能會將陣列傳遞給視圖，目的是將其渲染為 JSON 以便初始化 JavaScript 變數。例如：

```php
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

然而，您可以改用 `Illuminate\Support\Js::from` 方法，而不是手動呼叫 `json_encode`。`from` 方法接受與 PHP 的 `json_encode` 函式相同的引數；但是，它會確保產生的 JSON 已被適當轉義，以便包含在 HTML 引號內。`from` 方法將回傳一個字串 `JSON.parse` JavaScript 敘述，該敘述會將給定的物件或陣列轉換為有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新版本的 Laravel 應用程式骨架包含一個 `Js` Facade，它可以在您的 Blade 模板中方便地存取此功能：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 您應該只使用 `Js::from` 方法將既有變數渲染為 JSON。Blade 模板是基於正規表示式進行運作的，嘗試將複雜的運算式傳遞給該指令可能會導致未預期的失敗。

<a name="the-at-verbatim-directive"></a>
#### `@verbatim` 指令

如果您要在模板的大部分區域中顯示 JavaScript 變數，可以將 HTML 包裹在 `@verbatim` 指令中，這樣您就不必在每個 Blade 印出敘述前加上 `@` 符號：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了模板繼承與顯示資料外，Blade 還為常見的 PHP 控制結構（例如條件式敘述與迴圈）提供了方便的快捷方式。這些快捷方式提供了一種非常乾淨、簡潔的方式來使用 PHP 控制結構，同時又能保持與 PHP 對應語法相同的熟悉感。


<a name="if-statements"></a>
### If 敘述

你可以使用 `@if`、`@elseif`、`@else` 和 `@endif` 指令來建構 `if` 敘述。這些指令的功能與它們對應的 PHP 敘述完全相同：

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

除了前面提到的條件指令外，`@isset` 與 `@empty` 指令也可以作為對應 PHP 函數的方便快捷方式：

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

`@auth` 與 `@guest` 指令可用於快速判斷當前使用者是否已通過[認證](/docs/{{version}}/authentication)或是訪客：

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

你可以使用 `@production` 指令來檢查應用程式是否正在生產（Production）環境中運行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，你可以使用 `@env` 指令來判斷應用程式是否正在特定的環境中運行：

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

你可以使用 `@hasSection` 指令來判斷模板繼承的 Section 是否包含內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

你可以使用 `sectionMissing` 指令來判斷 Section 是否沒有內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```


<a name="session-directives"></a>
#### Session 指令

`@session` 指令可用於判斷是否存在 [Session](/docs/{{version}}/session) 值。若 Session 值存在，將會解析 `@session` 與 `@endsession` 指令之間的模板內容。在 `@session` 指令的內容中，你可以輸出 `$value` 變數來顯示 Session 的值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```


<a name="context-directives"></a>
#### Context 指令

`@context` 指令可用於判斷是否存在 [Context](/docs/{{version}}/context) 值。若 Context 值存在，將會解析 `@context` 與 `@endcontext` 指令之間的模板內容。在 `@context` 指令的內容中，你可以輸出 `$value` 變數來顯示 Context 的值：

```blade
@context('canonical')
    <link href="{{ $value }}" rel="canonical">
@endcontext
```


<a name="switch-statements"></a>
### Switch 敘述

可以使用 `@switch`、`@case`、`@break`、`@default` 與 `@endswitch` 指令來建構 Switch 敘述：

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

除了條件式敘述外，Blade 還提供了用於處理 PHP 迴圈結構的簡單指令。同樣地，這些指令的功能與它們對應的 PHP 敘述完全相同：

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
> 在 `foreach` 迴圈迭代時，你可以使用 [loop 變數](#the-loop-variable) 來取得有關迴圈的寶貴資訊，例如目前是否為迴圈的第一次或最後一次迭代。

在使用迴圈時，你也可以使用 `@continue` 與 `@break` 指令來跳過當前迭代或終止迴圈：

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

你也可以在指令宣告中直接包含跳過或終止的條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```


<a name="the-loop-variable"></a>
### Loop 變數

在 `foreach` 迴圈迭代期間，迴圈內部會提供一個 `$loop` 變數。此變數可讓你存取一些有用的資訊，例如目前的迴圈索引，以及這是否為迴圈的第一次或最後一次迭代：

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

如果處於巢狀迴圈中，你可以透過 `parent` 屬性存取父層迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數還包含其他各種有用的屬性：

<div class="overflow-auto">

| 屬性 | 說明 |
| ------------------ | ------------------------------------------------------ |
| `$loop->index` | 當前迴圈迭代的索引（從 0 開始）。 |
| `$loop->iteration` | 當前的迴圈迭代次數（從 1 開始）。 |
| `$loop->remaining` | 迴圈中剩餘的迭代次數。 |
| `$loop->count` | 被迭代的陣列中的項目總數。 |
| `$loop->first` | 是否為迴圈的第一次迭代。 |
| `$loop->last` | 是否為迴圈的最後一次迭代。 |
| `$loop->even` | 是否為迴圈的偶數次迭代。 |
| `$loop->odd` | 是否為迴圈的奇數次迭代。 |
| `$loop->depth` | 當前迴圈的巢狀深度。 |
| `$loop->parent` | 處於巢狀迴圈時，父層迴圈的變數。 |

</div>

<a name="conditional-classes"></a>
### 條件式 Class

`@class` 指令可用於條件式地編譯 CSS class 字串。該指令接受一個 class 陣列，其中的陣列鍵包含您想要新增的一個或多個 class，而陣列值則為布林運算式。如果陣列元素具有數字鍵，則它將總是包含在渲染後的 class 列表中：

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

同樣地，`@style` 指令可用於條件式地向 HTML 元素新增行內 CSS 樣式：

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

為了方便起見，您可以使用 `@checked` 指令來輕鬆指示給定的 HTML 核取方塊（checkbox）輸入是否為 "checked"。如果提供的條件評估為 `true`，此指令將印出 `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，`@selected` 指令可以用於指示給定的 select 選項是否應該被 "selected"：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，`@disabled` 指令可以用於指示給定的元素是否應該被 "disabled"：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，`@readonly` 指令可以用於指示給定的元素是否應該為 "readonly"：

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

另外，`@required` 指令可以用於指示給定的元素是否應該為 "required"：

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
> 雖然您可以自由使用 `@include` 指令，但 Blade [元件](#components) 提供了類似的功能，且比起 `@include` 指令具有更多優點，例如資料與屬性綁定。

Blade 的 `@include` 指令允許您從另一個視圖中引入 Blade 視圖。父視圖中可用的所有變數都將在被引入的視圖中可用：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

即使被引入的視圖會繼承父視圖中的所有可用資料，您也可以傳遞一個額外資料的陣列，使其在被引入的視圖中可用：

```blade
@include('view.name', ['status' => 'complete'])
```

如果您嘗試 `@include` 一個不存在的視圖，Laravel 將會拋出錯誤。如果您想要引入可能存在也可能不存在的視圖，應該使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

如果您想在給定的布林運算式評估為 `true` 或 `false` 時 `@include` 視圖，可以使用 `@includeWhen` 與 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

若要引入給定視圖陣列中第一個存在的視圖，可以使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

如果您想要引入一個視圖且不繼承父視圖的任何變數，可以使用 `@includeIsolated` 指令。被引入的視圖將只能存取您明確傳遞的變數：

```blade
@includeIsolated('view.name', ['user' => $user])
```

> [!WARNING]
> 您應該避免在 Blade 視圖中使用 `__DIR__` 和 `__FILE__` 常數，因為它們會指向已快取、編譯後的視圖位置。


<a name="rendering-views-for-collections"></a>
#### 為集合渲染視圖

您可以透過 Blade 的 `@each` 指令將迴圈與引入合併為一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個引數是陣列或集合中每個元素要渲染的視圖。第二個引數是您想要迭代的陣列或集合，而第三個引數是分配給視圖內當前迭代的變數名稱。因此，舉例來說，如果您正在迭代一個 `jobs` 陣列，通常您會希望在視圖中將每個工作當作 `job` 變數來存取。當前迭代的陣列鍵可在視圖中作為 `key` 變數使用。

您也可以向 `@each` 指令傳遞第四個引數。這個引數決定了當給定陣列為空時將被渲染的視圖。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 `@each` 渲染的視圖不會繼承父視圖的變數。如果子視圖需要這些變數，您應該改用 `@foreach` 和 `@include` 指令。


<a name="the-once-directive"></a>
### `@once` 指令

`@once` 指令允許您定義模板中每個渲染週期只會評估一次的部分。這對於使用[堆疊](#stacks)將給定的 JavaScript 推送到頁面的標頭（header）非常有用。例如，如果您在迴圈中渲染給定的[元件](#components)，您可能希望僅在第一次渲染該元件時將 JavaScript 推送到標頭：

```blade
@once
    @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush
@endonce
```

由於 `@once` 指令經常與 `@push` 或 `@prepend` 指令搭配使用，因此提供 `@pushOnce` 與 `@prependOnce` 指令以方便您使用：

```blade
@pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
@endPushOnce
```

如果您從兩個單獨的 Blade 模板中推送重複的內容，則應提供一個唯一的識別碼作為 `@pushOnce` 指令的第二個引數，以確保內容僅被渲染一次：

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

在某些情況下，在視圖中嵌入 PHP 程式碼非常有用。你可以使用 Blade 的 `@php` 指令在模板中執行一段原生的 PHP 程式碼：

```blade
@php
    $counter = 1;
@endphp
```

或者，如果你只需要使用 PHP 來匯入類別，可以使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

可以為 `@use` 指令提供第二個引數，為匯入的類別指定別名：

```blade
@use('App\Models\Flight', 'FlightModel')
```

如果你在同一個命名空間下有多個類別，可以將這些類別的匯入進行分組：

```blade
@use('App\Models\{Flight, Airport}')
```

`@use` 指令也支援透過在匯入路徑加上 `function` 或 `const` 修飾符來匯入 PHP 函式和常數：

```blade
@use(function App\Helpers\format_currency)
@use(const App\Constants\MAX_ATTEMPTS)
```

就像匯入類別一樣，函式與常數同樣支援別名：

```blade
@use(function App\Helpers\format_currency, 'formatMoney')
@use(const App\Constants\MAX_ATTEMPTS, 'MAX_TRIES')
```

函式與 const 修飾符同樣支援分組匯入，讓你能夠在單一指令中匯入來自同一命名空間的多個符號：

```blade
@use(function App\Helpers\{format_currency, format_date})
@use(const App\Constants\{MAX_ATTEMPTS, DEFAULT_TIMEOUT})
```

<a name="fonts"></a>
### 字型

當使用 [Laravel 的 Vite 字型最佳化](/docs/{{version}}/vite#working-with-fonts) 時，你可以使用 `@fonts` 指令在應用程式的版面配置中渲染已設定好的字型預載連結與行內字型 CSS：

```blade
<!doctype html>
<head>
    {{-- ... --}}

    @fonts
    @vite('resources/js/app.js')
</head>
```

`@fonts` 指令會渲染在 `vite.config.js` 檔案中設定的所有字型家族。該指令通常應放置在應用程式根版面配置的 `<head>` 之中，且位於任何使用這些字型的內容之前。

如果頁面只需要部分已設定的字型，你可以傳遞一個或多個字型別名給該指令：

```blade
{{-- Load a single font alias... --}}
@fonts('sans')

{{-- Load multiple font aliases... --}}
@fonts(['sans', 'mono'])
```

字型別名是在 Vite 設定中定義字型時使用 `alias` 選項進行設定的。`@fonts` 指令會呼叫 `Vite` Facade 所提供的 `fonts` 方法，該方法也可以直接呼叫：

```blade
{{ Vite::fonts(['sans', 'mono']) }}
```

<a name="comments"></a>
### 註解

Blade 還允許你在視圖中定義註解。然而，與 HTML 註解不同的是，Blade 註解不會包含在應用程式回傳的 HTML 中：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件與插槽（slots）帶來的效益與區塊（sections）、版面配置（layouts）以及引入（includes）類似；然而，有些人會覺得元件與插槽的心智模型更容易理解。撰寫元件有兩種方式：類別型元件（class-based components）與匿名元件（anonymous components）。

要建立類別型元件，您可以使用 `make:component` Artisan 命令。為了說明如何使用元件，我們將建立一個簡單的 `Alert` 元件。`make:component` 命令會將元件放置在 `app/View/Components` 目錄中：

```shell
php artisan make:component Alert
```

`make:component` 命令也會為該元件建立一個視圖模板。該視圖將放置在 `resources/views/components` 目錄中。在為自己的應用程式撰寫元件時，系統會自動搜尋 `app/View/Components` 目錄與 `resources/views/components` 目錄中的元件，因此通常不需要進行額外的元件註冊。

您也可以在子目錄中建立元件：

```shell
php artisan make:component Forms/Input
```

上述命令將在 `app/View/Components/Forms` 目錄中建立一個 `Input` 元件，且視圖會放置在 `resources/views/components/forms` 目錄中。


<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

在為自己的應用程式撰寫元件時，系統會自動搜尋 `app/View/Components` 目錄與 `resources/views/components` 目錄中的元件。

然而，如果您正在開發一個使用 Blade 元件的套件，則需要手動註冊您的元件類別及其 HTML 標籤別名。您通常應該在套件的服務提供者(Service Providers)的 `boot` 方法中註冊您的元件：

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

一旦您的元件被註冊，就可以使用其標籤別名來渲染它：

```blade
<x-package-alert/>
```

或者，您可以使用 `componentNamespace` 方法按慣例自動載入元件類別。例如，一個 `Nightshade` 套件可能擁有位於 `Package\Views\Components` 命名空間下的 `Calendar` 和 `ColorPicker` 元件：

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

這將允許透過其 Vendor 命名空間，使用 `package-name::` 語法來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會自動透過帕斯卡命名法（Pascal-casing）將元件名稱轉換，以自動偵測與此元件連結的類別。使用「點」號（dot）標記法也支援子目錄。


<a name="rendering-components"></a>
### 渲染元件

要顯示元件，您可以在 Blade 模板中使用 Blade 元件標籤。Blade 元件標籤以字串 `x-` 開頭，後跟元件類別的串接命名法（kebab-case）名稱：

```blade
<x-alert/>

<x-user-profile/>
```

如果元件類別嵌套在 `app/View/Components` 目錄的更深層，您可以使用 `.` 字元來表示目錄嵌套。例如，假設一個元件位於 `app/View/Components/Inputs/Button.php`，我們可以像這樣渲染它：

```blade
<x-inputs.button/>
```

如果您想依條件渲染元件，您可以在元件類別上定義一個 `shouldRender` 方法。如果 `shouldRender` 方法傳回 `false`，該元件將不會被渲染：

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

有時候元件屬於元件群組的一部分，您可能希望將相關的元件分組在單一目錄中。例如，想像一個具有以下類別結構的 "card" 元件：

```text
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件嵌套在 `Card` 目錄內，您可能會以為需要透過 `<x-card.card>` 來渲染該元件。然而，當元件的檔案名稱與元件的目錄名稱相符時，Laravel 會自動假設該元件是「根」元件，並允許您渲染該元件時無需重複目錄名稱：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳送資料至元件

你可以使用 HTML 屬性將資料傳送到 Blade 元件。寫死的純量值（primitive values）可以使用簡單的 HTML 屬性字串傳遞給元件。PHP 運算式與變數則應透過使用 `:` 字元作為前綴的屬性傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

你應該在元件的類別建構子中定義所有元件的資料屬性。元件上的所有公開（public）屬性都會自動提供給元件的視圖使用。不需要在元件的 `render` 方法中將資料傳遞給視圖：

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

當渲染你的元件時，你可以透過印出變數名稱來顯示元件公開變數的內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```


<a name="casing"></a>
#### 大小寫格式

元件建構子的引數應使用 `camelCase`（駝峰式大小寫）指定，而在 HTML 屬性中引用引數名稱時則應使用 `kebab-case`（蛇形/短橫線大小寫）。例如，給定以下元件建構子：

```php
/**
 * Create the component instance.
 */
public function __construct(
    public string $alertType,
) {}
```

可以像這樣將 `$alertType` 引數提供給元件：

```blade
<x-alert alert-type="danger" />
```


<a name="short-attribute-syntax"></a>
#### 簡短屬性語法

傳送屬性給元件時，你也可以使用「簡短屬性」語法。這通常很方便，因為屬性名稱通常會與它們所對應的變數名稱相符：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```


<a name="escaping-attribute-rendering"></a>
#### 跳脫屬性渲染

由於某些 JavaScript 框架（例如 Alpine.js）也使用冒號前綴的屬性，你可以使用雙冒號（`::`）前綴來告知 Blade 該屬性不是 PHP 運算式。例如，給定以下元件：

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

除了可以在元件模板中使用公開變數外，也可以呼叫元件上的任何公開方法。例如，假設有一個包含 `isSelected` 方法的元件：

```php
/**
 * Determine if the given option is the currently selected option.
 */
public function isSelected(string $option): bool
{
    return $option === $this->selected;
}
```

你可以透過呼叫與方法名稱相符的變數，從元件模板中執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```


<a name="using-attributes-slots-within-component-class"></a>
#### 在元件類別中存取屬性與插槽

Blade 元件也允許你在類別的 render 方法內部存取元件名稱、屬性與插槽。然而，為了存取這些資料，你應該從元件的 `render` 方法回傳一個 Closure：

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

元件 `render` 方法所回傳的 Closure 也可以接收一個 `$data` 陣列作為其唯一的引數。該陣列將包含多個提供元件相關資訊的元素：

```php
return function (array $data) {
    // $data['componentName'];
    // $data['attributes'];
    // $data['slot'];

    return '<div {{ $attributes }}>Components content</div>';
}
```

> [!WARNING]
> `$data` 陣列中的元素絕不應該直接嵌入到 `render` 方法回傳的 Blade 字串中，因為這樣做可能會透過惡意的屬性內容造成遠端程式碼執行。

`componentName` 等於 HTML 標籤中在 `x-` 前綴之後使用的名稱。因此 `<x-alert />` 的 `componentName` 將會是 `alert`。`attributes` 元素將會包含 HTML 標籤上的所有屬性。`slot` 元素是一個包含元件插槽內容的 `Illuminate\Support\HtmlString` 實例。

Closure 應該回傳一個字串。若回傳的字串對應至現有的視圖，則會渲染該視圖；否則，回傳的字串將會被當作行內 Blade 視圖進行評估。


<a name="additional-dependencies"></a>
#### 額外依賴

如果你的元件需要來自 Laravel [服務容器](/docs/{{version}}/container)的依賴，你可以在元件的任何資料屬性之前列出它們，它們將會由容器自動注入：

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

如果你想防止某些公開方法或屬性作為變數暴露給元件模板，可以將它們新增至元件上的 `$except` 陣列屬性中：

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

我們已經探討過如何將資料屬性傳遞給元件；然而，有時您可能需要指定額外的 HTML 屬性（例如 `class`），這些屬性並不屬於元件運作所需的資料。通常，您會希望將這些額外的屬性往下傳遞給元件模板的根元素。例如，假設我們想要像這樣渲染一個 `alert` 元件：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不屬於元件建構函式一部分的屬性，都會自動被新增至元件的「屬性包（Attribute Bag）」中。該屬性包會自動透過 `$attributes` 變數提供給元件。只要輸出這個變數，就能在元件內渲染所有的屬性：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前不支援在元件標籤內使用 `@env` 等指令。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時您可能需要為屬性指定預設值，或是將額外的值合併至元件的某些屬性中。要達到這個目的，您可以使用屬性包的 `merge` 方法。這個方法在定義一組應該永遠套用到元件的預設 CSS Class 時特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

假設這個元件的使用方式如下：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

最終渲染出的元件 HTML 會像下面這樣：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 依條件合併 Class

有時您可能希望在特定條件為 `true` 時才合併 Class。您可以透過 `class` 方法來做到這一點，該方法接受一個 Class 陣列，陣列的鍵（Key）包含您想要新增的一個或多個 Class，而值（Value）則是一個布林運算式。如果陣列元素使用的是數值鍵，它將會永遠被包含在渲染的 Class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

如果您需要將其他屬性合併至您的元件，您可以將 `merge` 方法串接在 `class` 方法之後：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 如果您需要在不需要接收合併屬性的其他 HTML 元素上依條件編譯 Class，可以使用 [@class 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 非 Class 屬性合併

當合併非 `class` 屬性時，提供給 `merge` 方法的值會被視為該屬性的「預設」值。然而，與 `class` 屬性不同的是，這些屬性不會與注入的屬性值合併在一起。相反地，它們將會被覆寫。例如，一個 `button` 元件的實作可能如下所示：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要使用自訂的 `type` 來渲染按鈕元件，可以在使用該元件時指定它。如果沒有指定類型，則會使用 `button` 類型：

```blade
<x-button type="submit">
    Submit
</x-button>
```

在此範例中，渲染出的 `button` 元件 HTML 為：

```blade
<button type="submit">
    Submit
</button>
```

如果您希望 `class` 以外的屬性能將其預設值與注入的值串接在一起，您可以使用 `prepends` 方法。在這個範例中，`data-controller` 屬性將永遠以 `profile-controller` 開頭，任何額外注入的 `data-controller` 值都會放在這個預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 取得與篩選屬性

您可以使用 `filter` 方法來篩選屬性。此方法接受一個閉包（Closure），若您希望將屬性保留在屬性包中，該閉包應傳回 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為了方便起見，您可以使用 `whereStartsWith` 方法來取得所有 Key 以指定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，`whereDoesntStartWith` 方法可以用來排除所有 Key 以指定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，您可以渲染指定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

如果您想檢查元件上是否存在某個屬性，您可以使用 `has` 方法。此方法接受屬性名稱作為其唯一的引數，並傳回一個布林值，用以指出該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

如果將陣列傳遞給 `has` 方法，該方法將會判斷所有指定的屬性是否皆存在於元件上：

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

您可以使用 `get` 方法來取得特定屬性值：

```blade
{{ $attributes->get('class') }}
```

`only` 方法可用於僅取得具有指定 Key 的屬性：

```blade
{{ $attributes->only(['class']) }}
```

`except` 方法可用於取得除了指定 Key 之外的所有屬性：

```blade
{{ $attributes->except(['class']) }}
```

<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，為了解析與渲染元件，某些關鍵字被保留給 Blade 內部使用。以下關鍵字無法在您的元件中被定義為公開屬性或方法名稱：

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

您經常會需要透過「插槽 (Slots)」將額外的內容傳遞給元件。元件插槽是透過輸出 `$slot` 變數來渲染的。為了瞭解這個概念，讓我們假設有一個 `alert` 元件包含以下標記：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過將內容注入元件中，以傳遞內容至 `slot`：

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時候元件可能需要在元件內的不同位置渲染多個不同的插槽。讓我們修改 alert 元件，以允許注入 "title" 插槽：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

您可以透過 `x-slot` 標籤來定義具名插槽的內容。任何不在明確的 `x-slot` 標籤內的內容，都會被傳遞給元件中的 `$slot` 變數：

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

此外，`hasActualContent` 方法可用於判斷插槽是否包含任何不是 HTML 註解的「實際」內容：

```blade
@if ($slot->hasActualContent())
    The scope has non-comment content.
@endif
```


<a name="scoped-slots"></a>
#### 作用域插槽

如果您曾經使用過像 Vue 這樣的 JavaScript 框架，您可能很熟悉「作用域插槽 (Scoped Slots)」，它允許您在插槽內存取來自該元件的資料或方法。您可以在 Laravel 中達成類似的行為：透過在元件中定義公開方法或屬性，並在插槽內經由 `$component` 變數存取該元件。在這個範例中，我們假設 `x-alert` 元件在其元件類別中定義了一個公開的 `formatAlert` 方法：

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

就像 Blade 元件一樣，您可以為插槽分配額外的[屬性](#component-attributes)，例如 CSS class 名稱：

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

若要與插槽屬性進行互動，您可以存取插槽變數的 `attributes` 屬性。關於如何與屬性互動的更多資訊，請參閱 [元件屬性](#component-attributes) 的相關文件：

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

對於非常小的元件來說，同時管理元件類別與元件的視圖模板可能會讓人覺得繁瑣。因此，您可以直接從 `render` 方法中回傳元件的標記：

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

若要建立渲染行內視圖的元件，可以在執行 `make:component` 指令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```


<a name="dynamic-components"></a>
### 動態元件

有時候您可能需要渲染元件，但在執行時期之前並不知道應該渲染哪一個元件。在這種情況下，您可以使用 Laravel 內建的 `dynamic-component` 元件，根據執行時期的數值或變數來渲染元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```


<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 以下關於手動註冊元件的文件，主要適用於撰寫包含視圖元件之 Laravel 套件的開發人員。如果您並不是在撰寫套件，這部分的元件文件可能對您並不適用。

為您自己的應用程式編寫元件時，系統會自動在 `app/View/Components` 目錄和 `resources/views/components` 目錄中自動偵測元件。

但是，如果您正在建置使用 Blade 元件的套件，或是將元件放置在非慣例的目錄中，您將需要手動註冊元件類別及其 HTML 標籤別名，以便 Laravel 知道在哪裡可以找到該元件。您通常應該在套件的服務提供者(Service Providers)中的 `boot` 方法內註冊元件：

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

當您的元件註冊完成後，就可以使用其標籤別名來進行渲染：

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

這將允許使用 `package-name::` 語法，透過 Vendor 命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會自動將元件名稱轉為帕斯卡命名法 (Pascal-case) 來檢測與該元件連結的類別。也支援使用「點」號表示法來指定子目錄。

<a name="anonymous-components"></a>
## 匿名元件

與行內元件類似，匿名元件提供了一種透過單一檔案管理元件的機制。然而，匿名元件使用的是單一視圖檔案，並且沒有與之關聯的類別。要定義匿名元件，您只需要將 Blade 模板放置在 `resources/views/components` 目錄中。例如，假設您在 `resources/views/components/alert.blade.php` 定義了一個元件，您可以像這樣簡單地渲染它：

```blade
<x-alert/>
```

您可以使用 `.` 字元來表示元件是否巢狀於 `components` 目錄深處。例如，假設元件定義在 `resources/views/components/inputs/button.blade.php`，您可以像這樣渲染它：

```blade
<x-inputs.button/>
```

要透過 Artisan 建立匿名元件，您可以在執行 `make:component` 指令時使用 `--view` 標誌：

```shell
php artisan make:component forms.input --view
```

上述指令將在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔案，該檔案可以透過 `<x-forms.input />` 來作為元件渲染。

<a name="anonymous-index-components"></a>
### 匿名 Index 元件

有時候，當一個元件是由許多 Blade 模板組成時，您可能會希望將該元件的模板集中在單一目錄中。例如，想像一個「折疊選單 (accordion)」元件具有以下目錄結構：

```text
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這種目錄結構允許您像這樣渲染折疊選單元件及其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

然而，為了透過 `x-accordion` 渲染折疊選單元件，我們被迫將「index」折疊選單元件模板放在 `resources/views/components` 目錄中，而不是將其與其他折疊選單相關的模板一起巢狀放在 `accordion` 目錄中。

值得慶幸的是，Blade 允許您在元件本身的目錄中放置一個名稱與該元件目錄名稱相同的檔案。當此模板存在時，即使它巢狀於目錄內，也可以作為元件的「根」元素來渲染。因此，我們可以繼續使用上述範例中給出的相同 Blade 語法；不過，我們會將目錄結構調整如下：

```text
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### 資料屬性 / 屬性

由於匿名元件沒有任何關聯的類別，您可能會想知道要如何區分哪些資料應該作為變數傳遞給元件，以及哪些屬性應該存放在元件的[屬性包](#component-attributes)中。

您可以在元件 Blade 模板的頂端使用 `@props` 指令來指定哪些屬性應被視為資料變數。元件上的所有其他屬性將可透過元件的屬性包來使用。如果您希望給予資料變數一個預設值，您可以將變數名稱指定為陣列鍵值，並將預設值指定為陣列的值：

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

有時候，您可能希望在子元件內部存取父元件的資料。在這些情況下，您可以使用 `@aware` 指令。例如，想像我們正在建立一個複雜的選單元件，包含父元件 `<x-menu>` 和子元件 `<x-menu.item>`：

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

由於 `color` 屬性 (prop) 僅傳遞給父元件 (`<x-menu>`)，因此它在 `<x-menu.item>` 內部是無法使用的。但是，如果我們使用 `@aware` 指令，我們也可以讓它在 `<x-menu.item>` 內部使用：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware` 指令無法存取未透過 HTML 屬性明確傳遞給父元件的父元件資料。未明確傳遞給父元件的預設 `@props` 值無法由 `@aware` 指令存取。

<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前所述，匿名元件通常是透過將 Blade 模板放置在 `resources/views/components` 目錄中來定義的。然而，除了預設路徑之外，您有時可能還想向 Laravel 註冊其他的匿名元件路徑。

`anonymousComponentPath` 方法的第一個引數接受匿名元件位置的「路徑 (path)」，第二個選填引數則接受元件應放置於其下的「命名空間 (namespace)」。通常，此方法應該在應用程式的其中一個[服務提供者(Service Providers)](/docs/{{version}}/providers)的 `boot` 方法中呼叫：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::anonymousComponentPath(__DIR__.'/../components');
}
```

當註冊元件路徑時未指定字首（如上例所示）時，它們也可以在 Blade 元件中渲染而不需要對應的字首。例如，如果上面註冊的路徑中存在一個 `panel.blade.php` 元件，它可以像這樣渲染：

```blade
<x-panel />
```

字首「命名空間」可以作為第二個引數提供給 `anonymousComponentPath` 方法：

```php
Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');
```

提供字首時，該「命名空間」內的元件在渲染時，可以透過將元件的命名空間加上字首來渲染元件：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建立版面配置

<a name="layouts-using-components"></a>
### 使用元件建立版面配置

大多數的 Web 應用程式在不同的頁面上都會保持相同的通用版面配置。如果我們必須在建立的每一個視圖中重複撰寫整個版面配置 HTML，那將會非常繁瑣且難以維護應用程式。幸運的是，將此版面配置定義為單一 [Blade 元件](#components)，並在整個應用程式中重複使用它非常方便。

<a name="defining-the-layout-component"></a>
#### 定義版面配置元件

舉例來說，假設我們正在建立一個「待辦事項」應用程式。我們可以定義一個如下所示的 `layout` 元件：

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

定義好 `layout` 元件後，我們就可以建立一個使用該元件的 Blade 視圖。在此範例中，我們將定義一個顯示任務列表的簡單視圖：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記住，注入到元件中的內容將會傳遞至我們 `layout` 元件內的預設 `$slot` 變數。正如您可能已經注意到的，如果提供了一個 `$title` 插槽，我們的 `layout` 也會尊重並使用它；否則，會顯示預設標題。我們可以使用[元件文件](#components)中討論的標準插槽語法，從任務列表視圖注入自訂標題：

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

現在我們已經定義了版面配置與任務列表視圖，我們只需要從路由回傳 `task` 視圖：

```php
use App\Models\Task;

Route::get('/tasks', function () {
    return view('tasks', ['tasks' => Task::all()]);
});
```

<a name="layouts-using-template-inheritance"></a>
### 使用模板繼承建立版面配置

<a name="defining-a-layout"></a>
#### 定義版面配置

版面配置也可以透過「模板繼承」來建立。在推出[元件](#components)之前，這是建構應用程式的主要方式。

首先，讓我們看一個簡單的範例。我們將先檢視一個頁面版面配置。由於大多數 Web 應用程式在不同頁面上保持相同的通用版面配置，因此將此版面配置定義為單一 Blade 視圖會非常方便：

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

如您所見，此檔案包含常見的 HTML 標籤。不過，請注意 `@section` 與 `@yield` 指令。顧名思義，`@section` 指令定義了一個內容區塊，而 `@yield` 指令則用於顯示指定區塊的內容。

現在我們已經為應用程式定義了版面配置，接著讓我們定義一個繼承該版面配置的子頁面。

<a name="extending-a-layout"></a>
#### 擴充版面配置

定義子視圖時，請使用 `@extends` Blade 指令來指定子視圖應該「繼承」哪一個版面配置。繼承 Blade 版面配置的視圖可以使用 `@section` 指令將內容注入版面配置的區塊中。請記住，如上例所示，這些區塊的內容將在版面配置中透過 `@yield` 來顯示：

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

在這個範例中，`sidebar` 區塊使用了 `@@parent` 指令來將內容附加（而非覆寫）至版面配置的側邊欄。當渲染視圖時，`@@parent` 指令將會被版面配置中的內容所替換。

> [!NOTE]
> 與前一個範例相反，這個 `sidebar` 區塊是以 `@endsection` 結尾，而不是 `@show`。`@endsection` 指令僅會定義區塊，而 `@show` 會定義並**立即呈報**該區塊。

`@yield` 指令也接受第二個參數作為預設值。若欲呈報的區塊未被定義，則會渲染此值：

```blade
@yield('content', 'Default content')
```

<a name="forms"></a>
## 表單

<a name="csrf-field"></a>
### CSRF 欄位

每當您在應用程式中定義 HTML 表單時，都應該在表單中包含一個隱藏的 CSRF 令牌欄位，以便 [CSRF 保護](/docs/{{version}}/csrf)中介層能夠驗證請求。您可以使用 `@csrf` Blade 指令來產生此令牌欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```

<a name="method-field"></a>
### Method 欄位

由於 HTML 表單無法發送 `PUT`、`PATCH` 或 `DELETE` 請求，您需要新增一個隱藏的 `_method` 欄位來偽造這些 HTTP 動作。`@method` Blade 指令可以為您建立此欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```

<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用於快速檢查給定屬性是否存在[驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令內部，您可以印出 `$message` 變數來顯示錯誤訊息：

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

您可以將[特定錯誤包的名稱](/docs/{{version}}/validation#named-error-bags)作為第二個參數傳遞給 `@error` 指令，以在包含多個表單的頁面上取得驗證錯誤訊息：

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

Blade 允許您推送至具名的堆疊，這些堆疊可以在其他視圖或版面配置中的某處進行渲染。這在指定子視圖所需的任何 JavaScript 函式庫時特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

如果您想在給定的布林運算式評估為 `true` 時才 `@push` 內容，您可以使用 `@pushIf` 指令：

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

如果您想將內容附加到堆疊的開頭，您應該使用 `@prepend` 指令：

```blade
@push('scripts')
    This will be second...
@endpush

// Later...

@prepend('scripts')
    This will be first...
@endprepend
```

`@hasstack` 指令可用於確定堆疊是否為空：

```blade
@hasstack('list')
    <ul>
        @stack('list')
    </ul>
@endif
```


<a name="service-injection"></a>
## 服務注入

`@inject` 指令可用於從 Laravel 的[服務容器](/docs/{{version}}/container)中取得服務。傳遞給 `@inject` 的第一個引數是放置該服務的變數名稱，而第二個引數則是要解析的服務類別或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```


<a name="rendering-inline-blade-templates"></a>
## 渲染行內 Blade 模板

有時候您可能需要將原生的 Blade 模板字串轉換為有效的 HTML。您可以使用 `Blade` Facade 提供的 `render` 方法來達成此目的。`render` 方法接收 Blade 模板字串以及要提供給模板的可選資料陣列：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 透過將行內 Blade 模板寫入 `storage/framework/views` 目錄來進行渲染。如果您希望 Laravel 在渲染 Blade 模板後刪除這些暫存檔，您可以為該方法提供 `deleteCachedView` 引數：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```


<a name="rendering-blade-fragments"></a>
## 渲染 Blade 片段

當使用如 [Turbo](https://turbo.hotwired.dev/) 及 [htmx](https://htmx.org/) 等前端框架時，您有時可能只需要在 HTTP 回應中傳回 Blade 模板的一部分。Blade 的「片段 (fragment)」功能恰好滿足這個需求。首先，請將 Blade 模板的一部分放置在 `@fragment` 和 `@endfragment` 指令中：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

然後，在渲染使用此模板的視圖時，您可以呼叫 `fragment` 方法，以指定發送的 HTTP 回應中僅應包含該特定的片段：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法允許您根據給定的條件判斷是否僅傳回視圖的某個片段。否則，將傳回整個視圖：

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

Blade 允許您使用 `directive` 方法定義自訂指令。當 Blade 編譯器遇到該自訂指令時，會使用該指令所包含的運算式呼叫所提供的回呼函式 (callback)。

以下範例建立了一個 `@datetime($var)` 指令，用來格式化給定的 `$var`（該變數應為 `DateTime` 的實例）：

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

如您所見，我們將 `format` 方法串接在傳入該指令的任何運算式後面。因此，在此範例中，該指令最終產生的 PHP 碼將為：

```php
<?php echo ($var)->format('m/d/Y H:i'); ?>
```

> [!WARNING]
> 更新 Blade 指令的邏輯後，您必須刪除所有快取的 Blade 視圖。可以使用 `view:clear` Artisan 指令清除快取的 Blade 視圖。


<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理程序

如果您嘗試使用 Blade 來「印出 (echo)」一個物件，該物件的 `__toString` 方法將會被調用。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 內建的「魔術方法 (magic methods)」之一。然而，有時您可能無法控制給定類別的 `__toString` 方法，例如當您互動的類別屬於第三方函式庫時。

在這些情況下，Blade 允許您為該特定類型的物件註冊自訂 Echo 處理程序。為此，您應該呼叫 Blade 的 `stringable` 方法。`stringable` 方法接收一個閉包 (closure)。此閉包應型別提示 (type-hint) 其負責渲染的物件類型。通常，`stringable` 方法應在應用程式的 `AppServiceProvider` 類別中的 `boot` 方法內呼叫：

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

一旦定義了自訂 Echo 處理程序，您就可以直接在 Blade 模板中印出該物件：

```blade
Cost: {{ $money }}
```


<a name="custom-if-statements"></a>
### 自訂 If 敘述

在定義簡單的自訂條件敘述時，撰寫自訂指令有時過於複雜。因此，Blade 提供了一個 `Blade::if` 方法，讓您能使用閉包快速定義自訂的條件指令。例如，讓我們定義一個檢查應用程式設定的預設「磁碟 (disk)」的自訂條件。我們可以在 `AppServiceProvider` 的 `boot` 方法中這樣做：

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

定義好自訂條件後，您就可以在模板中使用它：

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