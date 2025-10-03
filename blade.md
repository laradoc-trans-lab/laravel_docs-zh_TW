# Blade 樣板

- [簡介](#introduction)
    - [使用 Livewire 強化 Blade](#supercharging-blade-with-livewire)
- [顯示資料](#displaying-data)
    - [HTML 實體編碼](#html-entity-encoding)
    - [Blade 與 JavaScript 框架](#blade-and-javascript-frameworks)
- [Blade 指令](#blade-directives)
    - [If 陳述式](#if-statements)
    - [Switch 陳述式](#switch-statements)
    - [迴圈](#loops)
    - [迴圈變數](#the-loop-variable)
    - [條件式 Class](#conditional-classes)
    - [額外屬性](#additional-attributes)
    - [引入子 View](#including-subviews)
    - [`@once` 指令](#the-once-directive)
    - [原生 PHP](#raw-php)
    - [註解](#comments)
- [元件](#components)
    - [轉譯元件](#rendering-components)
    - [索引元件](#index-components)
    - [傳遞資料至元件](#passing-data-to-components)
    - [元件屬性](#component-attributes)
    - [保留關鍵字](#reserved-keywords)
    - [Slot](#slots)
    - [行內元件 View](#inline-component-views)
    - [動態元件](#dynamic-components)
    - [手動註冊元件](#manually-registering-components)
- [匿名元件](#anonymous-components)
    - [匿名索引元件](#anonymous-index-components)
    - [資料屬性 / 屬性](#data-properties-attributes)
    - [存取父層資料](#accessing-parent-data)
    - [匿名元件路徑](#anonymous-component-paths)
- [建構版面](#building-layouts)
    - [使用元件的版面](#layouts-using-components)
    - [使用樣板繼承的版面](#layouts-using-template-inheritance)
- [表單](#forms)
    - [CSRF 欄位](#csrf-field)
    - [方法欄位](#method-field)
    - [驗證錯誤](#validation-errors)
- [Stack](#stacks)
- [服務注入](#service-injection)
- [轉譯行內 Blade 樣板](#rendering-inline-blade-templates)
- [轉譯 Blade 片段](#rendering-blade-fragments)
- [擴充 Blade](#extending-blade)
    - [自訂 Echo 處理函式](#custom-echo-handlers)
    - [自訂 If 陳述式](#custom-if-statements)

<a name="introduction"></a>
## 簡介

Blade 是一個包含在 Laravel 中，既簡單又強大的樣板引擎。與其他 PHP 樣板引擎不同的是，Blade 並不會限制你在樣板中使用原生 PHP 程式碼。事實上，所有 Blade 樣板都會被編譯為原生 PHP 程式碼並快取起來，直到樣板被修改為止。這也代表 Blade 基本上不會為你的應用程式增加任何負擔。Blade 樣板檔案的副檔名為 `.blade.php`，且通常儲存在 `resources/views` 目錄中。

可以使用全域的 `view` 輔助函式從路由或控制器中回傳 Blade View。當然，如[有關 View 的說明文件](/docs/{{version}}/views)中所述，我們也可以使用 `view` 輔助函式的第二個引數來傳遞資料給 Blade View：

```php
Route::get('/', function () {
    return view('greeting', ['name' => 'Finn']);
});
```

<a name="supercharging-blade-with-livewire"></a>
### 使用 Livewire 強化 Blade

想讓你的 Blade 樣板更上一層樓，並輕鬆地建構動態介面嗎？請參考 [Laravel Livewire](https://livewire.laravel.com)。Livewire 可讓開發者撰寫 Blade 元件，並為其增加動態功能。一般來說，這些動態功能只有在 React 或 Vue 等前端框架中才能辦到。Livewire 提供了一種很好的方法來建構現代、響應式的前端，且不需要像許多 JavaScript 框架一樣處理複雜問題、客戶端轉譯、或建置步驟。

<a name="displaying-data"></a>
## 顯示資料

可以將傳遞至 Blade View 的資料變數用大括號包起來以顯示資料。舉例來說，假設有下列路由：

```php
Route::get('/', function () {
    return view('welcome', ['name' => 'Samantha']);
});
```

我們可以像這樣顯示 `name` 變數的內容：

```blade
Hello, {{ $name }}.
```

> [!NOTE]
> Blade 的 `{{ }}` Echo 陳述式會自動通過 PHP 的 `htmlspecialchars` 函式來處理，以避免 XSS 攻擊。

我們不僅限於顯示傳遞至 View 的變數內容。也可以 Echo (印出) 任何 PHP 函式的結果。實際上，我們可以在 Blade 的 Echo 陳述式中放入任何想要的 PHP 程式碼：

```blade
The current UNIX timestamp is {{ time() }}.
```

<a name="html-entity-encoding"></a>
### HTML 實體編碼

預設情況下，Blade (與 Laravel 的 `e` 函式) 會對 HTML 實體進行雙重編碼 (Double Encode)。若想停用雙重編碼，請在 `AppServiceProvider` 的 `boot` 方法中呼叫 `Blade::withoutDoubleEncoding` 方法：

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
#### 顯示未跳脫的資料

預設情況下，Blade 的 `{{ }}` 陳述式會自動通過 PHP 的 `htmlspecialchars` 函式來處理，以避免 XSS 攻擊。若不希望逸脫 (Escape) 資料，可以使用下列語法：

```blade
Hello, {!! $name !!}.
```

> [!WARNING]
> 在 Echo 使用者於應用程式中提供的內容時請務必小心。顯示使用者提供的資料時，應使用逸脫過的雙重大括號語法來避免 XSS 攻擊。

<a name="blade-and-javascript-frameworks"></a>
### Blade 與 JavaScript 框架

由於許多 JavaScript 框架也使用「大括號」來表示應在瀏覽器中顯示給定的運算式，因此，我們可以使用 `@` 符號來告知 Blade 轉譯引擎某個運算式應保持原樣不變。例如：

```blade
<h1>Laravel</h1>

Hello, @{{ name }}.
```

在此範例中，`@` 符號會被 Blade 移除；不過，`{{ name }}` 運算式會被 Blade 引擎保持原樣，讓其能被 JavaScript 框架轉譯。

`@` 符號也可用於逸脫 Blade 指令：

```blade
{{-- Blade template --}}
@@if()

<!-- HTML output -->
@if()
```

<a name="rendering-json"></a>
#### 轉譯 JSON

有時候，我們可能會想傳遞一個陣列到 View 中，並打算將其轉譯為 JSON 來初始化一個 JavaScript 變數。例如：

```php
<script>
    var app = <?php echo json_encode($array); ?>;
</script>
```

不過，除了手動呼叫 `json_encode` 外，我們也可以使用 `Illuminate\Support\Js::from` 方法指令。`from` 方法接受與 PHP 的 `json_encode` 函式相同的引數；不過，該方法會確保產生的 JSON 有被正確地逸脫，以便能包含在 HTML 引號中。`from` 方法會回傳一個 `JSON.parse` JavaScript 陳述式的字串，該陳述式會將給定的物件或陣列轉換為一個有效的 JavaScript 物件：

```blade
<script>
    var app = {{ Illuminate\Support\Js::from($array) }};
</script>
```

最新版的 Laravel 應用程式骨架包含一個 `Js` Facade，可用來在 Blade 樣板中輕鬆地存取此功能：

```blade
<script>
    var app = {{ Js::from($array) }};
</script>
```

> [!WARNING]
> 應只使用 `Js::from` 方法來將現有的變數轉譯為 JSON。Blade 樣板是基於正規表示式，若嘗試傳遞一個複雜的運算式給該指令可能會造成非預期的錯誤。

<a name="the-at-verbatim-directive"></a>
#### `@verbatim` 指令

若要在樣板中的一大部分顯示 JavaScript 變數，可以將 HTML 包在 `@verbatim` 指令中，這樣就不需要在每個 Blade Echo 陳述式前都加上 `@` 符號了：

```blade
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim
```

<a name="blade-directives"></a>
## Blade 指令

除了樣板繼承與顯示資料外，Blade 也為常見的 PHP 控制結構 (如條件陳述式與迴圈) 提供了方便的捷徑。這些捷徑提供了一種非常簡潔、扼要的方式來使用 PHP 控制結構，同時也與其對應的 PHP 寫法相似。

<a name="if-statements"></a>
### If 陳述式

我們可以使用 `@if`、`@elseif`、`@else`、與 `@endif` 指令來建構 `if` 陳述式。這些指令的功能與其對應的 PHP 寫法完全相同：

```blade
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif
```

為了方便，Blade 也提供了一個 `@unless` 指令：

```blade
@unless (Auth::check())
    You are not signed in.
@endunless
```

除了上面討論到的條件指令外，`@isset` 與 `@empty` 指令也可用來作為其對應 PHP 函式的方便捷徑：

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

可使用 `@auth` 與 `@guest` 指令來快速判斷目前的使用者是否為[已認證](/docs/{{version}}/authentication)使用者或是訪客：

```blade
@auth
    // The user is authenticated...
@endauth

@guest
    // The user is not authenticated...
@endguest
```

若有需要，在使用 `@auth` 與 `@guest` 指令時，我們也可以指定要檢查的認證 Guard：

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

我們可以使用 `@production` 指令來檢查應用程式是否正在生產環境中執行：

```blade
@production
    // Production specific content...
@endproduction
```

或者，也可以使用 `@env` 指令來判斷應用程式是否正在某個特定的環境中執行：

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

我們可以使用 `@hasSection` 指令來判斷某個樣板繼承的 Section 是否有內容：

```blade
@hasSection('navigation')
    <div class="pull-right">
        @yield('navigation')
    </div>

    <div class="clearfix"></div>
@endif
```

我們可以使用 `sectionMissing` 指令來判斷某個 Section 是否沒有內容：

```blade
@sectionMissing('navigation')
    <div class="pull-right">
        @include('default-navigation')
    </div>
@endif
```

<a name="session-directives"></a>
#### Session 指令

可使用 `@session` 指令來判斷某個 [Session](/docs/{{version}}/session) 值是否存在。若 Session 值存在，則會對 `@session` 與 `@endsession` 指令中的樣板內容進行求值。在 `@session` 指令的內容中，可以印出 `$value` 變數來顯示 Session 值：

```blade
@session('status')
    <div class="p-4 bg-green-100">
        {{ $value }}
    </div>
@endsession
```

<a name="context-directives"></a>
#### Context 指令

可使用 `@context` 指令來判斷某個 [Context](/docs/{{version}}/context) 值是否存在。若 Context 值存在，則會對 `@context` 與 `@endcontext` 指令中的樣板內容進行求值。在 `@context` 指令的內容中，可以印出 `$value` 變數來顯示 Context 值：

```blade
@context('canonical')
    <link href="{{ $value }}" rel="canonical">
@endcontext
```

<a name="switch-statements"></a>
### Switch 陳述式

Switch 陳述式可使用 `@switch`、`@case`、`@break`、`@default`、與 `@endswitch` 指令來建構：

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

除了條件陳述式外，Blade 也為 PHP 的迴圈結構提供了簡單的指令。同樣地，這些指令的功能也都與其對應的 PHP 寫法完全相同：

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
> 在 `foreach` 迴圈中進行迭代時，可以使用[迴圈變數](#the-loop-variable)來取得有關該迴圈的有用資訊，例如目前是否為第一次或最後一次迭代。

使用迴圈時，我們也可以使用 `@continue` 與 `@break` 指令來跳過目前的迭代或結束迴圈：

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

我們也可以在指令的宣告中直接包含 continue 或 break 的條件：

```blade
@foreach ($users as $user)
    @continue($user->type == 1)

    <li>{{ $user->name }}</li>

    @break($user->number == 5)
@endforeach
```

<a name="the-loop-variable"></a>
### 迴圈變數

在 `foreach` 迴圈中進行迭代時，迴圈內會有一個 `$loop` 變數可用。這個變數能讓我們存取一些有用的資訊，例如目前的迴圈索引、以及目前是否為第一次或最後一次迭代等：

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

若在巢狀迴圈中，可以透過 `parent` 屬性來存取父層迴圈的 `$loop` 變數：

```blade
@foreach ($users as $user)
    @foreach ($user->posts as $post)
        @if ($loop->parent->first)
            This is the first iteration of the parent loop.
        @endif
    @endforeach
@endforeach
```

`$loop` 變數也包含了其他各種有用的屬性：

<div class="overflow-auto">

| 屬性 | 說明 |
| --- | --- |
| `$loop->index` | 目前迴圈迭代的索引 (從 0 開始)。 |
| `$loop->iteration` | 目前的迴圈迭代 (從 1 開始)。 |
| `$loop->remaining` | 迴圈中剩餘的迭代次數。 |
| `$loop->count` | 正被迭代的陣列中的項目總數。 |
| `$loop->first` | 是否為迴圈的第一次迭代。 |
| `$loop->last` | 是否為迴圈的最後一次迭代。 |
| `$loop->even` | 是否為迴圈中的偶數次迭代。 |
| `$loop->odd` | 是否為迴圈中的奇數次迭代。 |
| `$loop->depth` | 目前迴圈的巢狀層級。 |
| `$loop->parent` | 在巢狀迴圈中時，父層的迴圈變數。 |

</div>

<a name="conditional-classes"></a>
### 條件式 Class

`@class` 指令會根據條件式地編譯出一個 CSS Class 字串。該指令接受一個 Class 陣列，其中，陣列的索引鍵為要新增的一或多個 Class，而陣列的值則為一個布林運算式。若陣列元素為數字索引鍵，則該元素會一直被包含在最終轉譯出的 Class 清單內：

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

同樣地，我們也可以使用 `@style` 指令來有條件地為 HTML 元素加上行內 (inline) CSS 樣式：

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

為了方便起見，我們可以使用 `@checked` 指令來輕鬆標示某個 HTML Checkbox 輸入方塊是否應為「已選取 (checked)」狀態。若提供的條件判斷為 `true`，該指令會 Echo `checked`：

```blade
<input
    type="checkbox"
    name="active"
    value="active"
    @checked(old('active', $user->active))
/>
```

同樣地，我們可以使用 `@selected` 指令來標示某個 Select 選項是否應為「已選取 (selected)」狀態：

```blade
<select name="version">
    @foreach ($product->versions as $version)
        <option value="{{ $version }}" @selected(old('version') == $version)>
            {{ $version }}
        </option>
    @endforeach
</select>
```

此外，我們也可使用 `@disabled` 指令來標示某個元素是否應為「已停用 (disabled)」狀態：

```blade
<button type="submit" @disabled($errors->isNotEmpty())>Submit</button>
```

此外，我們也可使用 `@readonly` 指令來標示某個元素是否應為「唯讀 (readonly)」狀態：

```blade
<input
    type="email"
    name="email"
    value="email@laravel.com"
    @readonly($user->isNotAdmin())
/>
```

此外，我們也可使用 `@required` 指令來標示某個元素是否應為「必填 (required)」狀態：

```blade
<input
    type="text"
    name="title"
    value="title"
    @required($user->isAdmin())
/>
```

<a name="including-subviews"></a>
### 引入子 View

> [!NOTE]
> 雖然可以自由使用 `@include` 指令，但 Blade [元件](#components) 提供了類似的功能，且相較於 `@include` 指令，元件還提供了資料與屬性綁定等數個優點。

Blade 的 `@include` 指令可讓你在某個 View 中引入另一個 Blade View。所有可用於父層 View 的變數也都會可用於被引入的 View：

```blade
<div>
    @include('shared.errors')

    <form>
        <!-- Form Contents -->
    </form>
</div>
```

雖然被引入的 View 會繼承父層 View 中所有可用的資料，但我們也可以傳入一個額外資料的陣列，讓這些資料可用於被引入的 View：

```blade
@include('view.name', ['status' => 'complete'])
```

若嘗試 `@include` 一個不存在的 View，Laravel 將會擲回一個錯誤。若想引入一個可能存在、也可能不存在的 View，則應使用 `@includeIf` 指令：

```blade
@includeIf('view.name', ['status' => 'complete'])
```

若想在某個布林表達式判斷為 `true` 或 `false` 時才 `@include` 某個 View，則可使用 `@includeWhen` 與 `@includeUnless` 指令：

```blade
@includeWhen($boolean, 'view.name', ['status' => 'complete'])

@includeUnless($boolean, 'view.name', ['status' => 'complete'])
```

若要從給定的 View 陣列中引入第一個存在的 View，則可使用 `includeFirst` 指令：

```blade
@includeFirst(['custom.admin', 'admin'], ['status' => 'complete'])
```

> [!WARNING]
> 應避免在 Blade View 中使用 `__DIR__` 與 `__FILE__` 常數，因為這些常數會指向快取、已編譯 View 的位置。

<a name="rendering-views-for-collections"></a>
#### 為 Collection 轉譯 View

我們可以使用 Blade 的 `@each` 指令來將迴圈與 Include 合併為一行：

```blade
@each('view.name', $jobs, 'job')
```

`@each` 指令的第一個引數為要為陣列或 Collection 中各個元素所轉譯的 View。第二個引數則為要迭代的陣列或 Collection。第三個引數為要在 View 中指派給目前迭代的變數名稱。舉例來說，若要迭代一個 `jobs` 陣列，我們通常會在 View 中將各個 Job 以 `job` 變數來存取。目前迭代的陣列索引鍵，可在 View 中以 `key` 變數存取。

我們也可以傳遞第四個引數給 `@each` 指令。此引數用來決定當給定陣列為空時要轉譯的 View。

```blade
@each('view.name', $jobs, 'job', 'view.empty')
```

> [!WARNING]
> 透過 `@each` 轉譯的 View 不會繼承父層 View 的變數。若子 View 需要這些變數，則應改用 `@foreach` 與 `@include` 指令。

<a name="the-once-directive"></a>
### `@once` 指令

`@once` 指令可用來定義樣板中只會被每個轉譯週期評估一次的部分。當要使用 [Stack](#stacks) 來將某段 JavaScript 推送到頁面的標頭 (Header) 時，這個指令特別好用。舉例來說，若要在迴圈中轉譯某個給定的[元件](#components)，我們可能只希望在元件第一次被轉譯時才將 JavaScript 推送到標頭：

```blade
@once
    @push('scripts')
        <script>
            // Your custom JavaScript...
        </script>
    @endpush
@endonce
```

由於 `@once` 指令通常會與 `@push` 或 `@prepend` 指令搭配使用，為了方便起見，我們也可以使用 `@pushOnce` 與 `@prependOnce` 指令：

```blade
@pushOnce('scripts')
    <script>
        // Your custom JavaScript...
    </script>
@endPushOnce
```

<a name="raw-php"></a>
### 原生 PHP

在某些情況下，在 View 中內嵌 PHP 程式碼會很有用。我們可以使用 Blade 的 `@php` 指令在樣板中執行一段原生 PHP：

```blade
@php
    $counter = 1;
@endphp
```

或者，若只需要使用 PHP 來匯入 (Import) Class，則可使用 `@use` 指令：

```blade
@use('App\Models\Flight')
```

我們可以提供第二個引數給 `@use` 指令來為匯入的 Class 設定別名：

```blade
@use('App\Models\Flight', 'FlightModel')
```

若有多個 Class 在同個命名空間下，可以將這些 Class 的匯入群組起來：

```blade
@use('App\Models\{Flight, Airport}')
```

`@use` 指令也支援匯入 PHP 函式與常數，只要在匯入路徑前加上 `function` 或 `const` 修飾詞即可：

```blade
@use(function App\Helpers\format_currency)
@use(const App\Constants\MAX_ATTEMPTS)
```

與匯入 Class 一樣，函式與常數也支援別名：

```blade
@use(function App\Helpers\format_currency, 'formatMoney')
@use(const App\Constants\MAX_ATTEMPTS, 'MAX_TRIES')
```

函式與常數修飾詞也都支援群組匯入，讓我們可以在單一指令內從同個命名空間中匯入多個符號：

```blade
@use(function App\Helpers\{format_currency, format_date})
@use(const App\Constants\{MAX_ATTEMPTS, DEFAULT_TIMEOUT})
```

<a name="comments"></a>
### 註解

Blade 也可讓你在 View 中定義註解。不過，與 HTML 註解不同，Blade 註解不會被包含在你的應用程式所回傳的 HTML 內：

```blade
{{-- This comment will not be present in the rendered HTML --}}
```

<a name="components"></a>
## 元件

元件與 Slot 提供的益處與 Section、版面、以及 Include 類似；不過，有些人可能會覺得元件與 Slot 的心智模型比較容易理解。有兩種方法可以撰寫元件：基於 Class 的元件與匿名元件。

若要建立一個基於 Class 的元件，可使用 `make:component` 這個 Artisan 指令。為了說明元件的用法，我們先來建立一個簡單的 `Alert` 元件。`make:component` 指令會將該元件放置於 `app/View/Components` 目錄下：

```shell
php artisan make:component Alert
```

`make:component` 指令也會為該元件建立一個 View 樣板。該 View 會被放置在 `resources/views/components` 目錄下。在為自己的專案撰寫元件時，元件會自動在 `app/View/Components` 與 `resources/views/components` 目錄下被找到，所以通常不需要進一步註冊元件。

也可以在子目錄下建立元件：

```shell
php artisan make:component Forms/Input
```

上述指令會在 `app/View/Components/Forms` 目錄下建立一個 `Input` 元件，且其 View 會被放置在 `resources/views/components/forms` 目錄下。

若想建立匿名元件 (一種只有 Blade 樣板而沒有 Class 的元件)，則可在執行 `make:component` 指令時使用 `--view` 旗標：

```shell
php artisan make:component forms.input --view
```

上述指令會在 `resources/views/components/forms/input.blade.php` 建立一個 Blade 檔，並可透過 `<x-forms.input />` 將其作為元件轉譯。

<a name="manually-registering-package-components"></a>
#### 手動註冊套件元件

在為自己的專案撰寫元件時，元件會自動在 `app/View/Components` 與 `resources/views/components` 目錄下被找到。

不過，若正在建構一個使用 Blade 元件的套件，則需要手動註冊元件的 Class 及其 HTML 標籤別名。一般來說，應在套件的 Service Provider 中的 `boot` 方法內註冊元件：

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

元件註冊好後，就可以使用其標籤別名來轉譯：

```blade
<x-package-alert/>
```

或者，也可以使用 `componentNamespace` 方法來依慣例自動載入元件的 Class。舉例來說，有個 `Nightshade` 套件，其中可能包含了 `Calendar` 與 `ColorPicker` 元件，都放在 `Package\Views\Components` 命名空間下：

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

這樣一來，就能使用 `package-name::` 語法來以 Vendor 命名空間使用套件的元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會自動將元件名稱轉為 PascalCase 來偵測連結到該元件的 Class。也支援使用「點 (.)」標記法來表示子目錄。

<a name="rendering-components"></a>
### 轉譯元件

若要顯示元件，可在 Blade 樣板中使用 Blade 元件標籤。Blade 元件標籤以 `x-` 字串開頭，並後接元件 Class 的 Kebab Case (小寫短線分隔命名) 名稱：

```blade
<x-alert/>

<x-user-profile/>
```

若元件的 Class 放在 `app/View/Components` 目錄下更深層的巢狀目錄內，則可使用 `.` 字元來表示目錄的巢狀結構。舉例來說，假設有個元件為於 `app/View/Components/Inputs/Button.php`，則可以這樣轉譯：

```blade
<x-inputs.button/>
```

若想有條件地轉譯元件，可在元件 Class 上定義一個 `shouldRender` 方法。若 `shouldRender` 方法回傳 `false`，則該元件將不會被轉譯：

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

有時候，元件會是一組元件群組的一部分，且我們可能會想將相關的元件放在同一個目錄下。舉例來說，假設有個「Card (卡片)」元件，其 Class 結構如下：

```text
App\Views\Components\Card\Card
App\Views\Components\Card\Header
App\Views\Components\Card\Body
```

由於根 `Card` 元件是巢狀地放在 `Card` 目錄下，你可能會以為需要透過 `<x-card.card>` 來轉譯該元件。不過，當元件的檔名與元件目錄的名稱相同時，Laravel 會自動假設該元件為「根」元件，並允許在轉譯該元件時不需要重複目錄名稱：

```blade
<x-card>
    <x-card.header>...</x-card.header>
    <x-card.body>...</x-card.body>
</x-card>
```

<a name="passing-data-to-components"></a>
### 傳遞資料至元件

我們可以使用 HTML 屬性來傳遞資料給 Blade 元件。寫死的(Hard-coded)、純量值可以使用簡單的 HTML 屬性字串來傳遞給元件。PHP 的運算式與變數則應使用 `:` 字元作為前置詞的屬性來傳遞給元件：

```blade
<x-alert type="error" :message="$message"/>
```

應在元件的類別建構函式中定義好所有元件的資料屬性。元件上所有的 Public 屬性都會自動提供給元件的 View 使用。不需要從元件的 `render` 方法中將資料傳遞給 View：

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

元件被轉譯時，只要用變數名稱 Echo (印出) 元件的 Public 變數，即可顯示其內容：

```blade
<div class="alert alert-{{ $type }}">
    {{ $message }}
</div>
```


<a name="casing"></a>
#### 大小寫

元件建構函式的引數應使用 `camelCase` (駝峰式大小寫) 來指定，而在 HTML 屬性中參照引數名稱時則應使用 `kebab-case` ( kebab 式命名法)。舉例來說，假設有下列這個元件建構函式：

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
#### 屬性簡寫語法

將屬性傳遞給元件時，也可以使用「屬性簡寫」語法。由於屬性名稱通常會與其對應的變數名稱相同，因此這種寫法通常很方便：

```blade
{{-- Short attribute syntax... --}}
<x-profile :$userId :$name />

{{-- Is equivalent to... --}}
<x-profile :user-id="$userId" :name="$name" />
```


<a name="escaping-attribute-rendering"></a>
#### 逸出屬性轉譯

由於 Alpine.js 等部分 JavaScript 框架也使用冒號前置詞的屬性，因此我們可以使用雙冒號 (`::`) 前置詞來告知 Blade 該屬性並非 PHP 運算式。舉例來說，假設有下列元件：

```blade
<x-button ::class="{ danger: isDeleting }">
    Submit
</x-button>
```

Blade 將會轉譯出下列 HTML：

```blade
<button :class="{ danger: isDeleting }">
    Submit
</button>
```


<a name="component-methods"></a>
#### 元件方法

除了 Public 變數外，元件樣板中也可以叫用元件上的任何 Public 方法。舉例來說，假設有個元件，其中包含一個 `isSelected` 方法：

```php
/**
 * Determine if the given option is the currently selected option.
 */
public function isSelected(string $option): bool
{
    return $option === $this->selected;
}
```

我們可以從元件樣板中，透過叫用與方法同名的變數來執行此方法：

```blade
<option {{ $isSelected($value) ? 'selected' : '' }} value="{{ $value }}">
    {{ $label }}
</option>
```


<a name="using-attributes-slots-within-component-class"></a>
#### 在元件類別中存取屬性與 Slot

Blade 元件也允許在類別的 Render 方法中存取元件名稱、屬性、以及 Slot。不過，為了存取這些資料，我們應在元件的 `render` 方法中回傳一個閉包 (Closure)：

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

從元件 `render` 方法回傳的閉包也可以接收一個 `$data` 陣列作為其唯一的引數。這個陣列會包含數個提供元件相關資訊的元素：

```php
return function (array $data) {
    // $data['componentName'];
    // $data['attributes'];
    // $data['slot'];

    return '<div {{ $attributes }}>Components content</div>';
}
```

> [!WARNING]
> `$data` 陣列中的元素絕不應直接嵌入到 `render` 方法回傳的 Blade 字串中，因為這麼做可能會因惡意的屬性內容而導致遠端程式碼執行。

`componentName` 等於 HTML 標籤中 `x-` 前置詞後的名稱。因此 `<x-alert />` 的 `componentName` 會是 `alert`。`attributes` 元素會包含所有出現在 HTML 標籤上的屬性。`slot` 元素則是一個 `Illuminate\Support\HtmlString` 的實體，其中包含了元件 Slot 的內容。

閉包應回傳一個字串。若回傳的字串對應到現有的 View，則會轉譯該 View；否則，回傳的字串會被當作行內的 Blade View 來執行。


<a name="additional-dependencies"></a>
#### 額外相依性

若元件需要使用到 Laravel [服務容器](/docs/{{version}}/container) 中的相依性，可以在元件的資料屬性前將這些相依性列出，這樣一來容器就會自動注入這些相依性：

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
#### 隱藏屬性／方法

若不想讓某些 Public 方法或屬性以變數的形式暴露給元件樣板，可以將它們加到元件的 `$except` 陣列屬性中：

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

我們已經看過如何傳遞資料屬性給元件了；不過，有時候我們可能需要指定一些額外的 HTML 屬性 (如 `class`)，而這些屬性並非元件運作所需的資料。一般來說，我們會想把這些額外的屬性傳遞到元件樣板的根元素上。舉例來說，假設我們想這樣轉譯一個 `alert` 元件：

```blade
<x-alert type="error" :message="$message" class="mt-4"/>
```

所有不在元件建構函式中的屬性，都會被自動加到元件的「屬性包 (Attribute Bag)」中。這個屬性包會透過 `$attributes` 變數自動提供給元件使用。只要 Echo (印出) 這個變數，就可以在元件中轉譯所有的屬性：

```blade
<div {{ $attributes }}>
    <!-- Component content -->
</div>
```

> [!WARNING]
> 目前尚不支援在元件標籤內使用 `@env` 這類的指令。例如，`<x-alert :live="@env('production')"/>` 將不會被編譯。

<a name="default-merged-attributes"></a>
#### 預設 / 合併屬性

有時候，我們可能需要為屬性指定預設值，或是將額外的值合併到元件的某些屬性中。為此，我們可以使用屬性包的 `merge` 方法。這個方法在定義一組應永遠套用至元件的預設 CSS Class 時特別有用：

```blade
<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

如果我們假設此元件的使用方式如下：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

該元件最終轉譯出來的 HTML 會像下面這樣：

```blade
<div class="alert alert-error mb-4">
    <!-- Contents of the $message variable -->
</div>
```

<a name="conditionally-merge-classes"></a>
#### 有條件地合併 Class

有時候，我們可能會想在某個條件為 `true` 時才合併 Class。我們可以透過 `class` 方法來達成。class 方法可接受一個 Class 陣列，其中陣列的索引鍵為要加入的一或多個 Class，而值則為布林運算式。若陣列元素為數字索引鍵，則該 Class 會一直被包含在轉譯出的 Class 列表中：

```blade
<div {{ $attributes->class(['p-4', 'bg-red' => $hasError]) }}>
    {{ $message }}
</div>
```

若需要在元件上合併其他屬性，可以將 `merge` 方法串在 `class` 方法後面：

```blade
<button {{ $attributes->class(['p-4'])->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

> [!NOTE]
> 若需要在其他不應收到合併屬性的 HTML 元素上有條件地編譯 Class，可以使用 [@class 指令](#conditional-classes)。

<a name="non-class-attribute-merging"></a>
#### 非 Class 屬性的合併

合併非 `class` 的屬性時，提供給 `merge` 方法的值會被視為該屬性的「預設」值。不過，與 `class` 屬性不同，這些屬性不會與傳入的屬性值合併，而是會被覆寫。舉例來說，某個 `button` 元件的實作可能像下面這樣：

```blade
<button {{ $attributes->merge(['type' => 'button']) }}>
    {{ $slot }}
</button>
```

若要以自訂的 `type` 來轉譯按鈕元件，可在使用該元件時指定。若未指定 type，則會使用 `button` 型別：

```blade
<x-button type="submit">
    Submit
</x-button>
```

在此範例中，`button` 元件轉譯出來的 HTML 會是：

```blade
<button type="submit">
    Submit
</button>
```

若想讓 `class` 以外的屬性，能將其預設值與傳入值串連在一起，可以使用 `prepends` 方法。在本範例中，`data-controller` 屬性將一律以 `profile-controller` 開頭，任何額外傳入的 `data-controller` 值都會被放在這個預設值之後：

```blade
<div {{ $attributes->merge(['data-controller' => $attributes->prepends('profile-controller')]) }}>
    {{ $slot }}
</div>
```

<a name="filtering-attributes"></a>
#### 擷取與過濾屬性

可以使用 `filter` 方法來過濾屬性。這個方法可接受一個閉包，若想在屬性包中保留該屬性，則閉包應回傳 `true`：

```blade
{{ $attributes->filter(fn (string $value, string $key) => $key == 'foo') }}
```

為了方便起見，可以使用 `whereStartsWith` 方法來取得所有索引鍵以特定字串開頭的屬性：

```blade
{{ $attributes->whereStartsWith('wire:model') }}
```

相反地，可使用 `whereDoesntStartWith` 方法來排除所有索引鍵以特定字串開頭的屬性：

```blade
{{ $attributes->whereDoesntStartWith('wire:model') }}
```

使用 `first` 方法，可以轉譯特定屬性包中的第一個屬性：

```blade
{{ $attributes->whereStartsWith('wire:model')->first() }}
```

若想檢查元件上是否有某個屬性，可以使用 `has` 方法。此方法只接受一個屬性名稱的引數，並回傳一個布林值，以表示該屬性是否存在：

```blade
@if ($attributes->has('class'))
    <div>Class attribute is present</div>
@endif
```

若將一個陣列傳給 `has` 方法，則該方法會判斷是否元件上**所有**給定的屬性都存在：

```blade
@if ($attributes->has(['name', 'class']))
    <div>All of the attributes are present</div>
@endif
```

可使用 `hasAny` 方法來判斷元件上是否存在**任何**一個給定的屬性：

```blade
@if ($attributes->hasAny(['href', ':href', 'v-bind:href']))
    <div>One of the attributes is present</div>
@endif
```

可以使用 `get` 方法來取得特定屬性的值：

```blade
{{ $attributes->get('class') }}
```

`only` 方法可用來取得只有特定索引鍵的屬性：

```blade
{{ $attributes->only(['class']) }}
```

`except` 方法可用來取得除了特定索引鍵以外的所有屬性：

```blade
{{ $attributes->except(['class']) }}
```

<a name="reserved-keywords"></a>
### 保留關鍵字

預設情況下，有些關鍵字會保留給 Blade 內部用來轉譯元件。下列關鍵字不能被定義為元件中的 public 屬性或方法名稱：

<div class="content-list" markdown="1">

- `data`
- `render`
- `resolveView`
- `shouldRender`
- `view`
- `withAttributes`
- `withName`

</div>

<a name="slots"></a>
### Slot

您通常會需要透過「Slot (插槽)」來將額外內容傳遞至元件。元件的 Slot 是透過 Echo `$slot` 變數來轉譯的。為了解說這個概念，我們來假設有個 `alert` 元件，其標記如下：

```blade
<!-- /resources/views/components/alert.blade.php -->

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

我們可以透過將內容注入至元件中，以將內容傳遞至 `slot`：

```blade
<x-alert>
    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

有時候，一個元件可能會需要在元件內的不同位置轉譯多個不同的 Slot。我們來修改一下 alert 元價，讓該元件可注入一個「title」Slot：

```blade
<!-- /resources/views/components/alert.blade.php -->

<span class="alert-title">{{ $title }}</span>

<div class="alert alert-danger">
    {{ $slot }}
</div>
```

可使用 `x-slot` 標籤來定義具名 Slot 的內容。任何不在 `x-slot` 標籤中的內容，都會被傳遞至元件的 `$slot` 變數中：

```xml
<x-alert>
    <x-slot:title>
        Server Error
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

我們可以叫用 Slot 的 `isEmpty` 方法來判斷該 Slot 是否有包含內容：

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

此外，`hasActualContent` 方法可用來判斷 Slot 是否包含任何不是 HTML 註解的「實際」內容：

```blade
@if ($slot->hasActualContent())
    The scope has non-comment content.
@endif
```

<a name="scoped-slots"></a>
#### 作用域 Slot

若曾使用過 Vue 之類的 JavaScript 框架，那讀者可能對「作用域 Slot (Scoped Slot)」不陌生。作用域 Slot 能讓你在 Slot 內存取來自元件的資料或方法。在 Laravel 中，可以透過在元件上定義公開(Public)方法或屬性，並在 Slot 內透過 `$component` 變數來存取該元件，以達成類似的行為。在本範例中，我們假設 `x-alert` 元件在其元件 Class 上有定義一個公開的 `formatAlert` 方法：

```blade
<x-alert>
    <x-slot:title>
        {{ $component->formatAlert('Server Error') }}
    </x-slot>

    <strong>Whoops!</strong> Something went wrong!
</x-alert>
```

<a name="slot-attributes"></a>
#### Slot 屬性

與 Blade 元件一樣，我們也可以為 Slot 指定如 CSS Class 名稱之類的額外[屬性](#component-attributes)：

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

若要與 Slot 屬性互動，可存取 Slot 變數的 `attributes` 屬性。更多有關如何與屬性互動的資訊，請參考[元件屬性](#component-attributes)的說明文件：

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
### 行內元件 View

對於很小的元件來說，同時管理元件 Class 與元件的 View 樣板可能會覺得很麻煩。因此，我們可以從 `render` 方法中直接回傳元件的標記：

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
#### 產生行內 View 元件

若要建立一個會轉譯行內 View 的元件，可在執行 `make:component` 指令時使用 `inline` 選項：

```shell
php artisan make:component Alert --inline
```

<a name="dynamic-components"></a>
### 動態元件

有時候，我們可能會需要在執行階段才知道要轉譯哪個元件。在這種情況下，我們可以使用 Laravel 內建的 `dynamic-component` 元件，來根據執行階段的值或變數來轉譯元件：

```blade
// $componentName = "secondary-button";

<x-dynamic-component :component="$componentName" class="mt-4" />
```

<a name="manually-registering-components"></a>
### 手動註冊元件

> [!WARNING]
> 接下來有關手動註冊元件的文件，主要是寫給要撰寫內含 View 元件的 Laravel 套件的讀者。若你沒有要寫套件，這部分的元件文件可能與你無關。

當為自己的專案撰寫元件時，元件會自動在 `app/View/Components` 與 `resources/views/components` 目錄下被找到。

不過，若你正在建立一個使用 Blade 元件的套件，或將元件放置於非常規的目錄中，則需要手動註冊元件的 Class 及其 HTML 標籤別名，好讓 Laravel 知道要去哪裡找該元件。一般來說，應在套件的 Service Provider 中的 `boot` 方法內註冊元件：

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

元件註冊好後，就可以使用其標籤別名來轉譯：

```blade
<x-package-alert/>
```

#### 自動載入套件元件

或者，我們也可以使用 `componentNamespace` 方法來按照慣例自動載入元件的 Class。舉例來說，有個 `Nightshade` 套件，其中可能包含 `Calendar` 與 `ColorPicker` 元件，且這些元件都為於 `Package\Views\Components` 命名空間下：

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

這樣一來，就可以使用 `package-name::` 語法，以 Vendor 命名空間來使用套件元件：

```blade
<x-nightshade::calendar />
<x-nightshade::color-picker />
```

Blade 會透過將元件名稱轉為 PascalCase 的方式來自動偵測連結至此元件的 Class。也支援使用「點 (.)」標記法來表示子目錄。

<a name="anonymous-components"></a>
## 匿名元件

與行內元件類似，匿名元件提供了一種透過單一檔案來管理元件的機制。不過，匿名元件只使用單一的 View 檔案，且沒有關聯的 Class。若要定義匿名元件，只需要在 `resources/views/components` 目錄內放置一個 Blade 樣板即可。舉例來說，假設我們已在 `resources/views/components/alert.blade.php` 定義了一個元件，則可以像這樣直接轉譯它：

```blade
<x-alert/>
```

若元件被巢狀地放在 `components` 目錄的更深層，則可以使用 `.` 字元來表示。舉例來說，假設元件被定義在 `resources/views/components/inputs/button.blade.php`，則可以像這樣轉譯它：

```blade
<x-inputs.button/>
```

<a name="anonymous-index-components"></a>
### 匿名索引元件

有時候，當一個元件由多個 Blade 樣板組成時，我們可能會想將該元件的樣板都放在同一個目錄下。舉例來說，假設有個「accordion (手風琴)」元件，其目錄結構如下：

```text
/resources/views/components/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

這樣的目錄結構能讓我們像這樣轉譯 accordion 元件與其項目：

```blade
<x-accordion>
    <x-accordion.item>
        ...
    </x-accordion.item>
</x-accordion>
```

不過，為了能透過 `x-accordion` 來轉譯 accordion 元件，我們被迫將「索引」accordion 元件樣板放在 `resources/views/components` 目錄下，而不是與其他 accordion 相關的樣板一起巢狀地放在 `accordion` 目錄下。

幸好，Blade 允許我們在元件的目錄內放置一個與該目錄同名的檔案。當這個樣板存在時，就算被巢狀地放在目錄內，也能被轉譯為該元件的「根」元素。因此，我們可以繼續使用上述範例中的 Blade 語法，但將目錄結構調整成這樣：

```text
/resources/views/components/accordion/accordion.blade.php
/resources/views/components/accordion/item.blade.php
```

<a name="data-properties-attributes"></a>
### 資料屬性 / 屬性

由於匿名元件沒有任何關聯的 Class，讀者可能會好奇要如何區分哪些資料該作為變數傳入元件，而哪些屬性又該被放在元件的[屬性包 (Attribute Bag)](#component-attributes) 中。

我們可以在元件的 Blade 樣板上方使用 `@props` 指令來指定哪些屬性該被視為資料變數。元件上所有其他的屬性都會被放在元件的屬性包中。若想為資料變數提供預設值，可將變數名稱指定為陣列的鍵、並將預設值指定為陣列的值：

```blade
<!-- /resources/views/components/alert.blade.php -->

@props(['type' => 'info', 'message'])

<div {{ $attributes->merge(['class' => 'alert alert-'.$type]) }}>
    {{ $message }}
</div>
```

像上面這樣定義好元件後，我們就可以像這樣轉譯該元件：

```blade
<x-alert type="error" :message="$message" class="mb-4"/>
```

<a name="accessing-parent-data"></a>
### 存取父層資料

有時候，我們可能會想在子元件中存取父元件的資料。在這些情況下，可以使用 `@aware` 指令。舉例來說，假設我們正在建構一個複雜的選單元件，其中包含父層的 `<x-menu>` 與子層的 `<x-menu.item>`：

```blade
<x-menu color="purple">
    <x-menu.item>...</x-menu.item>
    <x-menu.item>...</x-menu.item>
</x-menu>
```

`<x-menu>` 元件的實作可能如下：

```blade
<!-- /resources/views/components/menu/index.blade.php -->

@props(['color' => 'gray'])

<ul {{ $attributes->merge(['class' => 'bg-'.$color.'-200']) }}>
    {{ $slot }}
</ul>
```

因為 `color` Prop 只被傳入父層 (`<x-menu>`)，所以 `<x-menu.item>` 中無法使用該 Prop。不過，若使用 `@aware` 指令，就可以讓 `<x-menu.item>` 中也能使用該 Prop：

```blade
<!-- /resources/views/components/menu/item.blade.php -->

@aware(['color' => 'gray'])

<li {{ $attributes->merge(['class' => 'text-'.$color.'-800']) }}>
    {{ $slot }}
</li>
```

> [!WARNING]
> `@aware` 指令無法存取未透過 HTML 屬性明確傳遞給父元件的父層資料。未明確傳遞給父元件的預設 `@props` 值無法被 `@aware` 指令存取。

<a name="anonymous-component-paths"></a>
### 匿名元件路徑

如前述，匿名元件通常是透過在 `resources/views/components` 目錄中放置 Blade 樣板來定義的。不過，有時候除了預設路徑外，我們可能還想向 Laravel 註冊其他的匿名元件路徑。

`anonymousComponentPath` 方法的第一個引數是匿名元件位置的「路徑」，而第二個可選的引數則是元件應放置的「命名空間」。一般來說，這個方法應在某個應用程式的 [Service Provider](/docs/{{version}}/providers) 中的 `boot` 方法內呼叫：

```php
/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Blade::anonymousComponentPath(__DIR__.'/../components');
}
```

當元件路徑像上方程式碼中那樣，在沒有指定前置詞的情況下被註冊時，就可以在 Blade 元件中不加前置詞地轉譯。舉例來說，若有個 `panel.blade.php` 元件存在於上述註冊的路徑中，則可以像這樣轉譯它：

```blade
<x-panel />
```

前置詞「命名空間」可作為第二個引數傳給 `anonymousComponentPath` 方法：

```php
Blade::anonymousComponentPath(__DIR__.'/../components', 'dashboard');
```

當有提供前置詞時，該「命名空間」內的元件在轉譯時，就必須在元件名稱前加上元件的命名空間：

```blade
<x-dashboard::panel />
```

<a name="building-layouts"></a>
## 建構版面


<a name="layouts-using-components"></a>
### 使用元件的版面

大多數的 Web 應用程式都會在許多頁面中採用相同的通用版面。若我們必須在建立的每個 View 中都重複整個版面 HTML 的話，那會變得很麻煩、且難以維護。值得慶幸的是，我們可以很方便地將這個版面定義為單一的 [Blade 元件](#components)，並在整個應用程式中使用。


<a name="defining-the-layout-component"></a>
#### 定義版面元件

舉例來說，假設我們正在建置一個「待辦事項」清單應用程式。我們可以定義一個 `layout` 元件，如下所示：

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
#### 套用版面元件

一旦定義好 `layout` 元件後，我們就可以建立一個使用該元件的 Blade View。在這個範例中，我們會定義一個簡單的 View 來顯示我們的任務清單：

```blade
<!-- resources/views/tasks.blade.php -->

<x-layout>
    @foreach ($tasks as $task)
        <div>{{ $task }}</div>
    @endforeach
</x-layout>
```

請記得，注入至元件的內容會提供給我們 `layout` 元件中預設的 `$slot` 變數。你可能已經注意到了，若有提供 `$title` Slot 的話，我們的 `layout` 也會遵循該 Slot；否則，就會顯示預設的標題。我們可以使用[元件說明文件](#components)中討論到的標準 Slot 語法，來從任務清單 View 注入一個自訂的標題：

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

既然我們已經定義好了版面與任務清單 View，我們只需要從路由回傳 `task` View 即可：

```php
use App\Models\Task;

Route::get('/tasks', function () {
    return view('tasks', ['tasks' => Task::all()]);
});
```


<a name="layouts-using-template-inheritance"></a>
### 使用樣板繼承的版面


<a name="defining-a-layout"></a>
#### 定義版面

我們也可以透過「樣板繼承 (Template Inheritance)」來建立版面。這是在[元件](#components)出現前，建置應用程式的主要方法。

讓我們先來看一個簡單的範例。首先，我們要檢查一個頁面版面。由於大多數的 Web 應用程式都會在許多頁面中採用相同的通用版面，因此可以方便地將這個版面定義為單一的 Blade View：

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

如你所見，這個檔案包含典型的 HTML 標記。不過，請注意 `@section` 與 `@yield` 指令。`@section` 指令就如其名，是用來定義內容的區塊；而 `@yield` 指令則是用來顯示給定區塊的內容。

既然我們已經為應用程式定義好版面了，接著來定義一個繼承該版面的子頁面。


<a name="extending-a-layout"></a>
#### 擴充版面

在定義子 View 時，可使用 `@extends` Blade 指令來指定該子 View 要「繼承」的版面。擴充 Blade 版面的 View 可使用 `@section` 指令來將內容注入至版面的區塊中。請記得，如上方程式碼範例所示，這些區塊的內容會由版面中的 `@yield` 顯示：

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

在這個範例中，`sidebar` 區塊使用了 `@@parent` 指令來將內容附加 (Append) 至版面的 Sidebar 上，而不是覆寫 (Overwrite) 掉。`@@parent` 指令在 View 被轉譯時，會被取代為版面的內容。

> [!NOTE]
> 與前一個範例不同，這個 `sidebar` 區塊是以 `@endsection` 結尾，而不是 `@show`。`@endsection` 指令只會定義區塊，而 `@show` 則會定義**並立即 Yield** 該區塊。

`@yield` 指令也接受第二個參數作為預設值。若要 Yield 的區塊未定義，就會轉譯這個值：

```blade
@yield('content', 'Default content')
```


<a name="forms"></a>
## 表單


<a name="csrf-field"></a>
### CSRF 欄位

在應用程式中定義 HTML 表單時，都應在表單中包含一個隱藏的 CSRF Token 欄位，這樣[CSRF 保護](/docs/{{version}}/csrf)中介層才能驗證該請求。我們可以使用 `@csrf` Blade 指令來產生 Token 欄位：

```blade
<form method="POST" action="/profile">
    @csrf

    ...
</form>
```


<a name="method-field"></a>
### 方法欄位

由於 HTML 表單不能發出 `PUT`、`PATCH`、或 `DELETE` 請求，因此需要加上一個隱藏的 `_method` 欄位來偽造這些 HTTP 動詞。`@method` Blade 指令可以為我們建立這個欄位：

```blade
<form action="/foo/bar" method="POST">
    @method('PUT')

    ...
</form>
```


<a name="validation-errors"></a>
### 驗證錯誤

`@error` 指令可用來快速檢查給定屬性上是否存在[驗證錯誤訊息](/docs/{{version}}/validation#quick-displaying-the-validation-errors)。在 `@error` 指令中，可以 Echo `$message` 變數來顯示錯誤訊息：

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

由於 `@error` 指令會被編譯為「if」陳述式，因此我們可以在某個屬性沒有錯誤時使用 `@else` 指令來轉譯內容：

```blade
<!-- /resources/views/auth.blade.php -->

<label for="email">Email address</label>

<input
    id="email"
    type="email"
    class="@error('email') is-invalid @else is-valid @enderror"
/>
```

我們可以將[某個特定錯誤包 (Error Bag) 的名稱](/docs/{{version}}/validation#named-error-bags)作為第二個參數傳給 `@error` 指令，來從包含多個表單的頁面中擷取驗證錯誤訊息：

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
## Stack

Blade 允許我們將內容推送 (Push) 至具名的 Stack 上，並可在另一個 View 或版面中的其他地方轉譯。這對於指定子 View 所需的 JavaScript 函式庫特別有用：

```blade
@push('scripts')
    <script src="/example.js"></script>
@endpush
```

若想在給定布林表達式為 `true` 時才 `@push` 內容，可使用 `@pushIf` 指令：

```blade
@pushIf($shouldPush, 'scripts')
    <script src="/example.js"></script>
@endPushIf
```

我們可以視需要多次推送至 Stack 上。若要轉譯完整的 Stack 內容，請將 Stack 的名稱傳給 `@stack` 指令：

```blade
<head>
    <!-- Head Contents -->

    @stack('scripts')
</head>
```

若想將內容前置 (Prepend) 到 Stack 的開頭，應使用 `@prepend` 指令：

```blade
@push('scripts')
    This will be second...
@endpush

// Later...

@prepend('scripts')
    This will be first...
@endprepend
```

<a name="service-injection"></a>
## 服務注入

`@inject` 指令可用於從 Laravel [服務容器](/docs/{{version}}/container) 中取得服務。傳給 `@inject` 的第一個引數是服務要被存入的變數名稱，而第二個引數則是想解析的服務之 Class 或介面名稱：

```blade
@inject('metrics', 'App\Services\MetricsService')

<div>
    Monthly Revenue: {{ $metrics->monthlyRevenue() }}.
</div>
```


<a name="rendering-inline-blade-templates"></a>
## 轉譯行內 Blade 樣板

有時候，我們可能會需要將原始 Blade 樣板字串轉換為有效的 HTML。這可以透過 `Blade` Facade 提供的 `render` 方法來完成。`render` 方法可接受 Blade 樣板字串，並可選擇性地傳入一個陣列來為樣板提供資料：

```php
use Illuminate\Support\Facades\Blade;

return Blade::render('Hello, {{ $name }}', ['name' => 'Julian Bashir']);
```

Laravel 在轉譯行內 Blade 樣板時，會將其寫入 `storage/framework/views` 目錄。若想讓 Laravel 在轉譯完 Blade 樣板後移除這些暫存檔，則可以在呼叫該方法時提供 `deleteCachedView` 引數：

```php
return Blade::render(
    'Hello, {{ $name }}',
    ['name' => 'Julian Bashir'],
    deleteCachedView: true
);
```


<a name="rendering-blade-fragments"></a>
## 轉譯 Blade 片段

在使用 [Turbo](https://turbo.hotwired.dev/) 或 [htmx](https://htmx.org/) 等前端框架時，有時候可能會只需要在 HTTP 回應中回傳 Blade 樣板的一部分。Blade 的「片段 (Fragment)」功能就是為此而生的。若要開始使用，請將 Blade 樣板中的一部分放在 `@fragment` 與 `@endfragment` 指令之間：

```blade
@fragment('user-list')
    <ul>
        @foreach ($users as $user)
            <li>{{ $user->name }}</li>
        @endforeach
    </ul>
@endfragment
```

接著，在轉譯使用該樣板的 View 時，可以叫用 `fragment` 方法來指定只有指定的片段應被包含在送出的 HTTP 回應中：

```php
return view('dashboard', ['users' => $users])->fragment('user-list');
```

`fragmentIf` 方法可讓我們根據給定的條件來條件性地回傳 View 片段。若條件不符，則會回傳整個 View：

```php
return view('dashboard', ['users' => $users])
    ->fragmentIf($request->hasHeader('HX-Request'), 'user-list');
```

`fragments` 與 `fragmentsIf` 方法可讓我們在回應中回傳多個 View 片段。這些片段會被串接在一起：

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

Blade 允許我們使用 `directive` 方法來定義自己的自訂指令。當 Blade 編譯器遇到自訂指令時，就會呼叫給定的回呼函式，並傳入該指令包含的運算式。

下列範例會建立一個 `@datetime($var)` 指令，用來格式化給定的 `$var`， 應為 `DateTime` 的實體：

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

如你所見，我們將 `format` 方法串接到傳入指令的任何運算式上。所以，在這個範例中，該指令最終產生的 PHP 會是：

```php
<?php echo ($var)->format('m/d/Y H:i'); ?>
```

> [!WARNING]
> 在更新完 Blade 指令的邏輯後，需要刪除所有已快取的 Blade View。可使用 `view:clear` Artisan 指令來移除已快取的 Blade View。


<a name="custom-echo-handlers"></a>
### 自訂 Echo 處理函式

當嘗試使用 Blade 來「Echo」一個物件時，該物件的 `__toString` 方法會被叫用。[__toString](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) 方法是 PHP 的內建「魔術方法」之一。不過，有時候我們可能無法控制給定 Class 的 `__toString` 方法，例如當我們互動的 Class 來自第三方套件時。

在這些情況下，Blade 允許我們為特定類型的物件註冊一個自訂的 Echo 處理函式。若要這麼做，我們應叫用 Blade 的 `stringable` 方法。`stringable` 方法可接受一個閉包。這個閉包應型別提示其負責轉譯的物件類型。一般來說，應在應用程式的 `AppServiceProvider` Class 中 `boot` 方法內呼叫 `stringable` 方法：

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

定義好自訂 Echo 處理函式後，就可以直接在 Blade 樣板中 Echo 該物件：

```blade
Cost: {{ $money }}
```


<a name="custom-if-statements"></a>
### 自訂 If 陳述式

在定義簡單的自訂條件陳述式時，自己寫一個自訂指令有時候會比想像中複雜。因此，Blade 提供了 `Blade::if` 方法，可讓我們使用閉包來快速定義自訂的條件指令。舉例來說，我們來定義一個自訂條件來檢查應用程式設定的預設「磁碟 (disk)」。我們可以在 `AppServiceProvider` 的 `boot` 方法中定義：

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

定義好自訂條件後，就可以在樣板中使用：

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