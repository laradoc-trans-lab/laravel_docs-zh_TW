# 輔助函數

- [簡介](#introduction)
- [可用方法](#available-methods)
- [其他工具](#other-utilities)
    - [效能基準測試](#benchmarking)
    - [日期](#dates)
    - [延遲函數](#deferred-functions)
    - [抽獎](#lottery)
    - [管道](#pipeline)
    - [休眠](#sleep)
    - [時間箱](#timebox)

<a name="introduction"></a>
## 簡介

Laravel 包含多種全域的「輔助」PHP 函數。其中許多函數由框架本身使用；但是，如果您覺得方便，也可以在自己的應用程式中自由使用它們。


<a name="available-methods"></a>
## 可用方法

<style>
    .collection-method-list > p {
        columns: 10.8em 3; -moz-columns: 10.8em 3; -webkit-columns: 10.8em 3;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>


<a name="arrays-and-objects-method-list"></a>
### 陣列與物件

<div class="collection-method-list" markdown="1">

[Arr::accessible](#method-array-accessible)
[Arr::add](#method-array-add)
[Arr::collapse](#method-array-collapse)
[Arr::crossJoin](#method-array-crossjoin)
[Arr::divide](#method-array-divide)
[Arr::dot](#method-array-dot)
[Arr::except](#method-array-except)
[Arr::exists](#method-array-exists)
[Arr::first](#method-array-first)
[Arr::flatten](#method-array-flatten)
[Arr::forget](#method-array-forget)
[Arr::get](#method-array-get)
[Arr::has](#method-array-has)
[Arr::hasAny](#method-array-hasany)
[Arr::isAssoc](#method-array-isassoc)
[Arr::isList](#method-array-islist)
[Arr::join](#method-array-join)
[Arr::keyBy](#method-array-keyby)
[Arr::last](#method-array-last)
[Arr::map](#method-array-map)
[Arr::mapSpread](#method-array-map-spread)
[Arr::mapWithKeys](#method-array-map-with-keys)
[Arr::only](#method-array-only)
[Arr::pluck](#method-array-pluck)
[Arr::prepend](#method-array-prepend)
[Arr::prependKeysWith](#method-array-prependkeyswith)
[Arr::pull](#method-array-pull)
[Arr::query](#method-array-query)
[Arr::random](#method-array-random)
[Arr::reject](#method-array-reject)
[Arr::set](#method-array-set)
[Arr::shuffle](#method-array-shuffle)
[Arr::sort](#method-array-sort)
[Arr::sortDesc](#method-array-sort-desc)
[Arr::sortRecursive](#method-array-sort-recursive)
[Arr::take](#method-array-take)
[Arr::toCssClasses](#method-array-to-css-classes)
[Arr::toCssStyles](#method-array-to-css-styles)
[Arr::undot](#method-array-undot)
[Arr::where](#method-array-where)
[Arr::whereNotNull](#method-array-where-not-null)
[Arr::wrap](#method-array-wrap)
[data_fill](#method-data-fill)
[data_get](#method-data-get)
[data_set](#method-data-set)
[data_forget](#method-data-forget)
[head](#method-head)
[last](#method-last)
</div>


<a name="numbers-method-list"></a>
### 數字

<div class="collection-method-list" markdown="1">

[Number::abbreviate](#method-number-abbreviate)
[Number::clamp](#method-number-clamp)
[Number::currency](#method-number-currency)
[Number::defaultCurrency](#method-default-currency)
[Number::defaultLocale](#method-default-locale)
[Number::fileSize](#method-number-file-size)
[Number::forHumans](#method-number-for-humans)
[Number::format](#method-number-format)
[Number::ordinal](#method-number-ordinal)
[Number::pairs](#method-number-pairs)
[Number::percentage](#method-number-percentage)
[Number::spell](#method-number-spell)
[Number::trim](#method-number-trim)
[Number::useLocale](#method-number-use-locale)
[Number::withLocale](#method-number-with-locale)
[Number::useCurrency](#method-number-use-currency)
[Number::withCurrency](#method-number-with-currency)

</div>


<a name="paths-method-list"></a>
### 路徑

<div class="collection-method-list" markdown="1">

[app_path](#method-app-path)
[base_path](#method-base-path)
[config_path](#method-config-path)
[database_path](#method-database-path)
[lang_path](#method-lang-path)
[mix](#method-mix)
[public_path](#method-public-path)
[resource_path](#method-resource-path)
[storage_path](#method-storage-path)

</div>


<a name="urls-method-list"></a>
### 網址

<div class="collection-method-list" markdown="1">

[action](#method-action)
[asset](#method-asset)
[route](#method-route)
[secure_asset](#method-secure-asset)
[secure_url](#method-secure-url)
[to_route](#method-to-route)
[url](#method-url)

</div>


<a name="miscellaneous-method-list"></a>
### 其他

<div class="collection-method-list" markdown="1">

[abort](#method-abort)
[abort_if](#method-abort-if)
[abort_unless](#method-abort-unless)
[app](#method-app)
[auth](#method-auth)
[back](#method-back)
[bcrypt](#method-bcrypt)
[blank](#method-blank)
[broadcast](#method-broadcast)
[cache](#method-cache)
[class_uses_recursive](#method-class-uses-recursive)
[collect](#method-collect)
[config](#method-config)
[context](#method-context)
[cookie](#method-cookie)
[csrf_field](#method-csrf-field)
[csrf_token](#method-csrf-token)
[decrypt](#method-decrypt)
[dd](#method-dd)
[dispatch](#method-dispatch)
[dispatch_sync](#method-dispatch-sync)
[dump](#method-dump)
[encrypt](#method-encrypt)
[env](#method-env)
[event](#method-event)
[fake](#method-fake)
[filled](#method-filled)
[info](#method-info)
[literal](#method-literal)
[logger](#method-logger)
[method_field](#method-method-field)
[now](#method-now)
[old](#method-old)
[once](#method-once)
[optional](#method-optional)
[policy](#method-policy)
[redirect](#method-redirect)
[report](#method-report)
[report_if](#method-report-if)
[report_unless](#method-report-unless)
[request](#method-request)
[rescue](#method-rescue)
[resolve](#method-resolve)
[response](#method-response)
[retry](#method-retry)
[session](#method-session)
[tap](#method-tap)
[throw_if](#method-throw-if)
[throw_unless](#method-throw-unless)
[today](#method-today)
[trait_uses_recursive](#method-trait-uses-recursive)
[transform](#method-transform)
[validator](#method-validator)
[value](#method-value)
[view](#method-view)
[with](#method-with)
[when](#method-when)

</div>

<a name="arrays"></a>
## 陣列與物件


<a name="method-array-accessible"></a>
#### `Arr::accessible()` {.collection-method .first-collection-method}

`Arr::accessible` 方法會判斷給定的值是否為陣列可存取 (array accessible)：

    use Illuminate\Support\Arr;
    use Illuminate\Support\Collection;

    $isAccessible = Arr::accessible(['a' => 1, 'b' => 2]);

    // true

    $isAccessible = Arr::accessible(new Collection);

    // true

    $isAccessible = Arr::accessible('abc');

    // false

    $isAccessible = Arr::accessible(new stdClass);

    // false


<a name="method-array-add"></a>
#### `Arr::add()` {.collection-method}

`Arr::add` 方法會將給定的鍵／值對 (key / value pair) 新增至陣列中，但前提是該鍵在陣列中尚不存在或其值為 `null`：

    use Illuminate\Support\Arr;

    $array = Arr::add(['name' => 'Desk'], 'price', 100);

    // ['name' => 'Desk', 'price' => 100]

    $array = Arr::add(['name' => 'Desk', 'price' => null], 'price', 100);

    // ['name' => 'Desk', 'price' => 100]


<a name="method-array-collapse"></a>
#### `Arr::collapse()` {.collection-method}

`Arr::collapse` 方法會將多個陣列組成的陣列 (array of arrays) 折疊 (collapses) 成單一陣列：

    use Illuminate\Support\Arr;

    $array = Arr::collapse([[1, 2, 3], [4, 5, 6], [7, 8, 9]]);

    // [1, 2, 3, 4, 5, 6, 7, 8, 9]


<a name="method-array-crossjoin"></a>
#### `Arr::crossJoin()` {.collection-method}

`Arr::crossJoin` 方法會對給定的陣列進行笛卡兒積 (Cartesian product) 操作，返回所有可能的排列組合：

    use Illuminate\Support\Arr;

    $matrix = Arr::crossJoin([1, 2], ['a', 'b']);

    /*
        [
            [1, 'a'],
            [1, 'b'],
            [2, 'a'],
            [2, 'b'],
        ]
    */

    $matrix = Arr::crossJoin([1, 2], ['a', 'b'], ['I', 'II']);

    /*
        [
            [1, 'a', 'I'],
            [1, 'a', 'II'],
            [1, 'b', 'I'],
            [1, 'b', 'II'],
            [2, 'a', 'I'],
            [2, 'a', 'II'],
            [2, 'b', 'I'],
            [2, 'b', 'II'],
        ]
    */


<a name="method-array-divide"></a>
#### `Arr::divide()` {.collection-method}

`Arr::divide` 方法會返回兩個陣列：一個包含給定陣列的所有鍵，另一個包含所有值：

    use Illuminate\Support\Arr;

    [$keys, $values] = Arr::divide(['name' => 'Desk']);

    // $keys: ['name']

    // $values: ['Desk']


<a name="method-array-dot"></a>
#### `Arr::dot()` {.collection-method}

`Arr::dot` 方法會將多維陣列扁平化 (flattens) 為單層陣列，並使用「點」符號 (dot notation) 來表示層級深度：

    use Illuminate\Support\Arr;

    $array = ['products' => ['desk' => ['price' => 100]]];

    $flattened = Arr::dot($array);

    // ['products.desk.price' => 100]


<a name="method-array-except"></a>
#### `Arr::except()` {.collection-method}

`Arr::except` 方法會從陣列中移除給定的鍵／值對：

    use Illuminate\Support\Arr;

    $array = ['name' => 'Desk', 'price' => 100];

    $filtered = Arr::except($array, ['price']);

    // ['name' => 'Desk']


<a name="method-array-exists"></a>
#### `Arr::exists()` {.collection-method}

`Arr::exists` 方法會檢查給定的鍵是否存在於提供的陣列中：

    use Illuminate\Support\Arr;

    $array = ['name' => 'John Doe', 'age' => 17];

    $exists = Arr::exists($array, 'name');

    // true

    $exists = Arr::exists($array, 'salary');

    // false


<a name="method-array-first"></a>
#### `Arr::first()` {.collection-method}

`Arr::first` 方法會返回陣列中第一個通過給定真值測試的元素：

    use Illuminate\Support\Arr;

    $array = [100, 200, 300];

    $first = Arr::first($array, function (int $value, int $key) {
        return $value >= 150;
    });

    // 200

也可將預設值作為第三個參數傳入此方法。如果沒有任何值通過真值測試，此預設值將會被返回：

    use Illuminate\Support\Arr;

    $first = Arr::first($array, $callback, $default);


<a name="method-array-flatten"></a>
#### `Arr::flatten()` {.collection-method}

`Arr::flatten` 方法會將多維陣列扁平化為單層陣列：

    use Illuminate\Support\Arr;

    $array = ['name' => 'Joe', 'languages' => ['PHP', 'Ruby']];

    $flattened = Arr::flatten($array);

    // ['Joe', 'PHP', 'Ruby']


<a name="method-array-forget"></a>
#### `Arr::forget()` {.collection-method}

`Arr::forget` 方法會使用「點」符號，從深度巢狀陣列中移除給定的鍵／值對：

    use Illuminate\Support\Arr;

    $array = ['products' => ['desk' => ['price' => 100]]];

    Arr::forget($array, 'products.desk');

    // ['products' => []]


<a name="method-array-get"></a>
#### `Arr::get()` {.collection-method}

`Arr::get` 方法會使用「點」符號，從深度巢狀陣列中取回值：

    use Illuminate\Support\Arr;

    $array = ['products' => ['desk' => ['price' => 100]]];

    $price = Arr::get($array, 'products.desk.price');

    // 100

`Arr::get` 方法也接受一個預設值，如果陣列中不存在指定的鍵，則會返回此預設值：

    use Illuminate\Support\Arr;

    $discount = Arr::get($array, 'products.desk.discount', 0);

    // 0


<a name="method-array-has"></a>
#### `Arr::has()` {.collection-method}

`Arr::has` 方法會使用「點」符號，檢查給定的項目或多個項目是否存在於陣列中：

    use Illuminate\Support\Arr;

    $array = ['product' => ['name' => 'Desk', 'price' => 100]];

    $contains = Arr::has($array, 'product.name');

    // true

    $contains = Arr::has($array, ['product.price', 'product.discount']);

    // false


<a name="method-array-hasany"></a>
#### `Arr::hasAny()` {.collection-method}

`Arr::hasAny` 方法會使用「點」符號，檢查給定集合中的任一項目是否存在於陣列中：

    use Illuminate\Support\Arr;

    $array = ['product' => ['name' => 'Desk', 'price' => 100]];

    $contains = Arr::hasAny($array, 'product.name');

    // true

    $contains = Arr::hasAny($array, ['product.name', 'product.discount']);

    // true

    $contains = Arr::hasAny($array, ['category', 'product.discount']);

    // false


<a name="method-array-isassoc"></a>
#### `Arr::isAssoc()` {.collection-method}

`Arr::isAssoc` 方法會判斷給定的陣列是否為關聯陣列 (associative array)，如果是則返回 `true`。當陣列的鍵不是從零開始的連續數字時，則被視為「關聯」陣列：

    use Illuminate\Support\Arr;

    $isAssoc = Arr::isAssoc(['product' => ['name' => 'Desk', 'price' => 100]]);

    // true

    $isAssoc = Arr::isAssoc([1, 2, 3]);

    // false


<a name="method-array-islist"></a>
#### `Arr::isList()` {.collection-method}

`Arr::isList` 方法會判斷給定陣列的鍵是否為從零開始的連續整數，如果是則返回 `true`：

    use Illuminate\Support\Arr;

    $isList = Arr::isList(['foo', 'bar', 'baz']);

    // true

    $isList = Arr::isList(['product' => ['name' => 'Desk', 'price' => 100]]);

    // false


<a name="method-array-join"></a>
#### `Arr::join()` {.collection-method}

`Arr::join` 方法會將陣列元素以字串連接起來。使用此方法的第二個參數，您還可以為陣列的最後一個元素指定連接字串：

    use Illuminate\Support\Arr;

    $array = ['Tailwind', 'Alpine', 'Laravel', 'Livewire'];

    $joined = Arr::join($array, ', ');

    // Tailwind, Alpine, Laravel, Livewire

    $joined = Arr::join($array, ', ', ' and ');

    // Tailwind, Alpine, Laravel and Livewire


<a name="method-array-keyby"></a>
#### `Arr::keyBy()` {.collection-method}

`Arr::keyBy` 方法會根據給定的鍵來索引陣列。如果有多個項目具有相同的鍵，則只有最後一個項目會出現在新陣列中：

    use Illuminate\Support\Arr;

    $array = [
        ['product_id' => 'prod-100', 'name' => 'Desk'],
        ['product_id' => 'prod-200', 'name' => 'Chair'],
    ];

    $keyed = Arr::keyBy($array, 'product_id');

    /*
        [
            'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
            'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
        ]
    */

<a name="method-array-last"></a>
#### `Arr::last()` {.collection-method}

`Arr::last` 方法會回傳陣列中，通過指定真值測試的最後一個元素：

    use Illuminate\Support\Arr;

    $array = [100, 200, 300, 110];

    $last = Arr::last($array, function (int $value, int $key) {
        return $value >= 150;
    });

    // 300

該方法可選的第三個參數是預設值。若沒有任何值通過真值測試，則會回傳此預設值：

    use Illuminate\Support\Arr;

    $last = Arr::last($array, $callback, $default);


<a name="method-array-map"></a>
#### `Arr::map()` {.collection-method}

`Arr::map` 方法會迭代陣列，並將每個值與鍵傳遞給指定的 callback。陣列的值會被 callback 回傳的值取代：

    use Illuminate\Support\Arr;

    $array = ['first' => 'james', 'last' => 'kirk'];

    $mapped = Arr::map($array, function (string $value, string $key) {
        return ucfirst($value);
    });

    // ['first' => 'James', 'last' => 'Kirk']


<a name="method-array-map-spread"></a>
#### `Arr::mapSpread()` {.collection-method}

`Arr::mapSpread` 方法會迭代陣列，並將每個巢狀項目的值傳遞到指定的 closure。此 closure 可以自由地修改並回傳該項目，進而形成一個由修改過項目組成的新陣列：

    use Illuminate\Support\Arr;

    $array = [
        [0, 1],
        [2, 3],
        [4, 5],
        [6, 7],
        [8, 9],
    ];

    $mapped = Arr::mapSpread($array, function (int $even, int $odd) {
        return $even + $odd;
    });

    /*
        [1, 5, 9, 13, 17]
    */


<a name="method-array-map-with-keys"></a>
#### `Arr::mapWithKeys()` {.collection-method}

`Arr::mapWithKeys` 方法會迭代陣列，並將每個值傳遞到指定的 callback。此 callback 應該回傳一個包含單一鍵／值組合的關聯式陣列：

    use Illuminate\Support\Arr;

    $array = [
        [
            'name' => 'John',
            'department' => 'Sales',
            'email' => 'john@example.com',
        ],
        [
            'name' => 'Jane',
            'department' => 'Marketing',
            'email' => 'jane@example.com',
        ]
    ];

    $mapped = Arr::mapWithKeys($array, function (array $item, int $key) {
        return [$item['email'] => $item['name']];
    });

    /*
        [
            'john@example.com' => 'John',
            'jane@example.com' => 'Jane',
        ]
    */


<a name="method-array-only"></a>
#### `Arr::only()` {.collection-method}

`Arr::only` 方法會從給定陣列中，只回傳指定鍵／值組合：

    use Illuminate\Support\Arr;

    $array = ['name' => 'Desk', 'price' => 100, 'orders' => 10];

    $slice = Arr::only($array, ['name', 'price']);

    // ['name' => 'Desk', 'price' => 100]


<a name="method-array-pluck"></a>
#### `Arr::pluck()` {.collection-method}

`Arr::pluck` 方法會從陣列中取出指定鍵的所有值：

    use Illuminate\Support\Arr;

    $array = [
        ['developer' => ['id' => 1, 'name' => 'Taylor']],
        ['developer' => ['id' => 2, 'name' => 'Abigail']],
    ];

    $names = Arr::pluck($array, 'developer.name');

    // ['Taylor', 'Abigail']

你也可以指定結果列表應如何鍵控：

    use Illuminate\Support\Arr;

    $names = Arr::pluck($array, 'developer.name', 'developer.id');

    // [1 => 'Taylor', 2 => 'Abigail']


<a name="method-array-prepend"></a>
#### `Arr::prepend()` {.collection-method}

`Arr::prepend` 方法會將一個項目推送到陣列的開頭：

    use Illuminate\Support\Arr;

    $array = ['one', 'two', 'three', 'four'];

    $array = Arr::prepend($array, 'zero');

    // ['zero', 'one', 'two', 'three', 'four']

如有需要，你可以指定該值應該使用的鍵：

    use Illuminate\Support\Arr;

    $array = ['price' => 100];

    $array = Arr::prepend($array, 'Desk', 'name');

    // ['name' => 'Desk', 'price' => 100]


<a name="method-array-prependkeyswith"></a>
#### `Arr::prependKeysWith()` {.collection-method}

`Arr::prependKeysWith` 會將關聯式陣列的所有鍵名加上指定前綴：

    use Illuminate\Support\Arr;

    $array = [
        'name' => 'Desk',
        'price' => 100,
    ];

    $keyed = Arr::prependKeysWith($array, 'product.');

    /*
        [
            'product.name' => 'Desk',
            'product.price' => 100,
        ]
    */


<a name="method-array-pull"></a>
#### `Arr::pull()` {.collection-method}

`Arr::pull` 方法會回傳並從陣列中移除一個鍵／值組合：

    use Illuminate\Support\Arr;

    $array = ['name' => 'Desk', 'price' => 100];

    $name = Arr::pull($array, 'name');

    // $name: Desk

    // $array: ['price' => 100]

該方法可選的第三個參數是預設值。若鍵不存在，則會回傳此預設值：

    use Illuminate\Support\Arr;

    $value = Arr::pull($array, $key, $default);


<a name="method-array-query"></a>
#### `Arr::query()` {.collection-method}

`Arr::query` 方法會將陣列轉換為查詢字串：

    use Illuminate\Support\Arr;

    $array = [
        'name' => 'Taylor',
        'order' => [
            'column' => 'created_at',
            'direction' => 'desc'
        ]
    ];

    Arr::query($array);

    // name=Taylor&order[column]=created_at&order[direction]=desc


<a name="method-array-random"></a>
#### `Arr::random()` {.collection-method}

`Arr::random` 方法會從陣列中回傳一個隨機值：

    use Illuminate\Support\Arr;

    $array = [1, 2, 3, 4, 5];

    $random = Arr::random($array);

    // 4 - (retrieved randomly)

你也可以指定要回傳的項目數量作為第二個選用參數。請注意，即使只想要一個項目，提供此參數也會回傳一個陣列：

    use Illuminate\Support\Arr;

    $items = Arr::random($array, 2);

    // [2, 5] - (retrieved randomly)


<a name="method-array-reject"></a>
#### `Arr::reject()` {.collection-method}

`Arr::reject` 方法會使用指定 closure 從陣列中移除項目：

    use Illuminate\Support\Arr;

    $array = [100, '200', 300, '400', 500];

    $filtered = Arr::reject($array, function (string|int $value, int $key) {
        return is_string($value);
    });

    // [0 => 100, 2 => 300, 4 => 500]


<a name="method-array-set"></a>
#### `Arr::set()` {.collection-method}

`Arr::set` 方法會使用「點」表示法，設定巢狀陣列中的值：

    use Illuminate\Support\Arr;

    $array = ['products' => ['desk' => ['price' => 100]]];

    Arr::set($array, 'products.desk.price', 200);

    // ['products' => ['desk' => ['price' => 200]]]


<a name="method-array-shuffle"></a>
#### `Arr::shuffle()` {.collection-method}

`Arr::shuffle` 方法會隨機打亂陣列中的項目：

    use Illuminate\Support\Arr;

    $array = Arr::shuffle([1, 2, 3, 4, 5]);

    // [3, 2, 5, 1, 4] - (generated randomly)


<a name="method-array-sort"></a>
#### `Arr::sort()` {.collection-method}

`Arr::sort` 方法會依據陣列的值進行排序：

    use Illuminate\Support\Arr;

    $array = ['Desk', 'Table', 'Chair'];

    $sorted = Arr::sort($array);

    // ['Chair', 'Desk', 'Table']

你也可以根據指定 closure 的結果來排序陣列：

    use Illuminate\Support\Arr;

    $array = [
        ['name' => 'Desk'],
        ['name' => 'Table'],
        ['name' => 'Chair'],
    ];

    $sorted = array_values(Arr::sort($array, function (array $value) {
        return $value['name'];
    }));

    /*
        [
            ['name' => 'Chair'],
            ['name' => 'Desk'],
            ['name' => 'Table'],
        ]
    */

<a name="method-array-sort-desc"></a>
#### `Arr::sortDesc()` {.collection-method}

`Arr::sortDesc` 方法會依據陣列中的值，以遞減順序排序陣列：

    use Illuminate\Support\Arr;

    $array = ['Desk', 'Table', 'Chair'];

    $sorted = Arr::sortDesc($array);

    // ['Table', 'Desk', 'Chair']

您也可以透過給定的閉包結果來排序陣列：

    use Illuminate\Support\Arr;

    $array = [
        ['name' => 'Desk'],
        ['name' => 'Table'],
        ['name' => 'Chair'],
    ];

    $sorted = array_values(Arr::sortDesc($array, function (array $value) {
        return $value['name'];
    }));

    /*
        [
            ['name' => 'Table'],
            ['name' => 'Desk'],
            ['name' => 'Chair'],
        ]
    */


<a name="method-array-sort-recursive"></a>
#### `Arr::sortRecursive()` {.collection-method}

`Arr::sortRecursive` 方法會遞迴地排序陣列，其中數值索引子陣列使用 `sort` 函數，關聯式子陣列則使用 `ksort` 函數：

    use Illuminate\Support\Arr;

    $array = [
        ['Roman', 'Taylor', 'Li'],
        ['PHP', 'Ruby', 'JavaScript'],
        ['one' => 1, 'two' => 2, 'three' => 3],
    ];

    $sorted = Arr::sortRecursive($array);

    /*
        [
            ['JavaScript', 'PHP', 'Ruby'],
            ['one' => 1, 'three' => 3, 'two' => 2],
            ['Li', 'Roman', 'Taylor'],
        ]
    */

如果您希望結果以遞減順序排序，可以使用 `Arr::sortRecursiveDesc` 方法。

    $sorted = Arr::sortRecursiveDesc($array);


<a name="method-array-take"></a>
#### `Arr::take()` {.collection-method}

`Arr::take` 方法會回傳一個包含指定數量元素的新陣列：

    use Illuminate\Support\Arr;

    $array = [0, 1, 2, 3, 4, 5];

    $chunk = Arr::take($array, 3);

    // [0, 1, 2]

您也可以傳遞一個負整數，從陣列的末尾取出指定數量的元素：

    $array = [0, 1, 2, 3, 4, 5];

    $chunk = Arr::take($array, -2);

    // [4, 5]


<a name="method-array-to-css-classes"></a>
#### `Arr::toCssClasses()` {.collection-method}

`Arr::toCssClasses` 方法會條件式地編譯 CSS class 字串。該方法接受一個 class 陣列，其中陣列的鍵包含您希望新增的一個或多個 class，而值則是一個布林運算式。如果陣列元素具有數值鍵，它將始終包含在渲染的 class 列表中：

    use Illuminate\Support\Arr;

    $isActive = false;
    $hasError = true;

    $array = ['p-4', 'font-bold' => $isActive, 'bg-red' => $hasError];

    $classes = Arr::toCssClasses($array);

    /*
        'p-4 bg-red'
    */


<a name="method-array-to-css-styles"></a>
#### `Arr::toCssStyles()` {.collection-method}

`Arr::toCssStyles` 會條件式地編譯 CSS 樣式字串。該方法接受一個 class 陣列，其中陣列的鍵包含您希望新增的一個或多個 class，而值則是一個布林運算式。如果陣列元素具有數值鍵，它將始終包含在渲染的 class 列表中：

```php
use Illuminate\Support\Arr;

$hasColor = true;

$array = ['background-color: blue', 'color: blue' => $hasColor];

$classes = Arr::toCssStyles($array);

/*
    'background-color: blue; color: blue;'
*/
```

此方法支援 Laravel 的功能，允許 [將 class 合併至 Blade component 的屬性包](/docs/{{version}}/blade#conditionally-merge-classes)，以及 `@class` [Blade 指令](/docs/{{version}}/blade#conditional-classes)。


<a name="method-array-undot"></a>
#### `Arr::undot()` {.collection-method}

`Arr::undot` 方法會將使用「點」記號的單維陣列展開為多維陣列：

    use Illuminate\Support\Arr;

    $array = [
        'user.name' => 'Kevin Malone',
        'user.occupation' => 'Accountant',
    ];

    $array = Arr::undot($array);

    // ['user' => ['name' => 'Kevin Malone', 'occupation' => 'Accountant']]


<a name="method-array-where"></a>
#### `Arr::where()` {.collection-method}

`Arr::where` 方法會使用給定的閉包來篩選陣列：

    use Illuminate\Support\Arr;

    $array = [100, '200', 300, '400', 500];

    $filtered = Arr::where($array, function (string|int $value, int $key) {
        return is_string($value);
    });

    // [1 => '200', 3 => '400']


<a name="method-array-where-not-null"></a>
#### `Arr::whereNotNull()` {.collection-method}

`Arr::whereNotNull` 方法會從給定陣列中移除所有 `null` 值：

    use Illuminate\Support\Arr;

    $array = [0, null];

    $filtered = Arr::whereNotNull($array);

    // [0 => 0]


<a name="method-array-wrap"></a>
#### `Arr::wrap()` {.collection-method}

`Arr::wrap` 方法會將給定值包裝成陣列。如果給定值已經是陣列，它將不會被修改而直接回傳：

    use Illuminate\Support\Arr;

    $string = 'Laravel';

    $array = Arr::wrap($string);

    // ['Laravel']

如果給定值為 `null`，則會回傳一個空陣列：

    use Illuminate\Support\Arr;

    $array = Arr::wrap(null);

    // []


<a name="method-data-fill"></a>
#### `data_fill()` {.collection-method}

`data_fill` 函數會使用「點」記號在巢狀陣列或物件中設定遺失的值：

    $data = ['products' => ['desk' => ['price' => 100]]];

    data_fill($data, 'products.desk.price', 200);

    // ['products' => ['desk' => ['price' => 100]]]

    data_fill($data, 'products.desk.discount', 10);

    // ['products' => ['desk' => ['price' => 100, 'discount' => 10]]]

此函數也接受星號作為萬用字元，並會據此填入目標：

    $data = [
        'products' => [
            ['name' => 'Desk 1', 'price' => 100],
            ['name' => 'Desk 2'],
        ],
    ];

    data_fill($data, 'products.*.price', 200);

    /*
        [
            'products' => [
                ['name' => 'Desk 1', 'price' => 100],
                ['name' => 'Desk 2', 'price' => 200],
            ],
        ]
    */


<a name="method-data-get"></a>
#### `data_get()` {.collection-method}

`data_get` 函數會使用「點」記號從巢狀陣列或物件中擷取值：

    $data = ['products' => ['desk' => ['price' => 100]]];

    $price = data_get($data, 'products.desk.price');

    // 100

`data_get` 函數也接受一個預設值，如果找不到指定的鍵，則會回傳該預設值：

    $discount = data_get($data, 'products.desk.discount', 0);

    // 0

此函數也接受星號作為萬用字元，可以針對陣列或物件的任何鍵：

    $data = [
        'product-one' => ['name' => 'Desk 1', 'price' => 100],
        'product-two' => ['name' => 'Desk 2', 'price' => 150],
    ];

    data_get($data, '*.name');

    // ['Desk 1', 'Desk 2'];

`{first}` 和 `{last}` 預留位置可用於擷取陣列中的第一個或最後一個元素：

    $flight = [
        'segments' => [
            ['from' => 'LHR', 'departure' => '9:00', 'to' => 'IST', 'arrival' => '15:00'],
            ['from' => 'IST', 'departure' => '16:00', 'to' => 'PKX', 'arrival' => '20:00'],
        ],
    ];

    data_get($flight, 'segments.{first}.arrival');

    // 15:00

<a name="method-data-set"></a>
#### `data_set()` {.collection-method}

`data_set` 函數使用「點」符號將值設定到巢狀陣列或物件中：

    $data = ['products' => ['desk' => ['price' => 100]]];

    data_set($data, 'products.desk.price', 200);

    // ['products' => ['desk' => ['price' => 200]]]

此函數也接受使用星號的萬用字元，並會據此設定目標上的值：

    $data = [
        'products' => [
            ['name' => 'Desk 1', 'price' => 100],
            ['name' => 'Desk 2', 'price' => 150],
        ],
    ];

    data_set($data, 'products.*.price', 200);

    /*
        [
            'products' => [
                ['name' => 'Desk 1', 'price' => 200],
                ['name' => 'Desk 2', 'price' => 200],
            ],
        ]
    */

預設情況下，任何現有值都會被覆寫。如果您只想在值不存在時才設定，您可以將 `false` 作為第四個引數傳遞給此函數：

    $data = ['products' => ['desk' => ['price' => 100]]];

    data_set($data, 'products.desk.price', 200, overwrite: false);

    // ['products' => ['desk' => ['price' => 100]]]


<a name="method-data-forget"></a>
#### `data_forget()` {.collection-method}

`data_forget` 函數使用「點」符號從巢狀陣列或物件中移除值：

    $data = ['products' => ['desk' => ['price' => 100]]];

    data_forget($data, 'products.desk.price');

    // ['products' => ['desk' => []]]

此函數也接受使用星號的萬用字元，並會據此從目標中移除值：

    $data = [
        'products' => [
            ['name' => 'Desk 1', 'price' => 100],
            ['name' => 'Desk 2', 'price' => 150],
        ],
    ];

    data_forget($data, 'products.*.price');

    /*
        [
            'products' => [
                ['name' => 'Desk 1'],
                ['name' => 'Desk 2'],
            ],
        ]
    */


<a name="method-head"></a>
#### `head()` {.collection-method}

`head` 函數回傳給定陣列中的第一個元素：

    $array = [100, 200, 300];

    $first = head($array);

    // 100


<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 函數回傳給定陣列中的最後一個元素：

    $array = [100, 200, 300];

    $last = last($array);

    // 300

<a name="numbers"></a>
## Numbers


<a name="method-number-abbreviate"></a>
#### `Number::abbreviate()` {.collection-method}

`Number::abbreviate` 方法會回傳指定數值的易讀格式，並附帶單位縮寫：

    use Illuminate\Support\Number;

    $number = Number::abbreviate(1000);

    // 1K

    $number = Number::abbreviate(489939);

    // 490K

    $number = Number::abbreviate(1230000, precision: 2);

    // 1.23M


<a name="method-number-clamp"></a>
#### `Number::clamp()` {.collection-method}

`Number::clamp` 方法會確保給定數值維持在指定範圍內。若數值低於最小值，則回傳最小值。若數值高於最大值，則回傳最大值：

    use Illuminate\Support\Number;

    $number = Number::clamp(105, min: 10, max: 100);

    // 100

    $number = Number::clamp(5, min: 10, max: 100);

    // 10

    $number = Number::clamp(10, min: 10, max: 100);

    // 10

    $number = Number::clamp(20, min: 10, max: 100);

    // 20


<a name="method-number-currency"></a>
#### `Number::currency()` {.collection-method}

`Number::currency` 方法會將給定數值轉換為貨幣字串表示法：

    use Illuminate\Support\Number;

    $currency = Number::currency(1000);

    // $1,000.00

    $currency = Number::currency(1000, in: 'EUR');

    // €1,000.00

    $currency = Number::currency(1000, in: 'EUR', locale: 'de');

    // 1.000,00 €


<a name="method-default-currency"></a>
#### `Number::defaultCurrency()` {.collection-method}

`Number::defaultCurrency` 方法會回傳 `Number` class 所使用的預設貨幣：

    use Illuminate\Support\Number;

    $currency = Number::defaultCurrency();

    // USD


<a name="method-default-locale"></a>
#### `Number::defaultLocale()` {.collection-method}

`Number::defaultLocale` 方法會回傳 `Number` class 所使用的預設語系：

    use Illuminate\Support\Number;

    $locale = Number::defaultLocale();

    // en


<a name="method-number-file-size"></a>
#### `Number::fileSize()` {.collection-method}

`Number::fileSize` 方法會將給定位元組 (byte) 數值轉換為檔案大小字串表示法：

    use Illuminate\Support\Number;

    $size = Number::fileSize(1024);

    // 1 KB

    $size = Number::fileSize(1024 * 1024);

    // 1 MB

    $size = Number::fileSize(1024, precision: 2);

    // 1.00 KB


<a name="method-number-for-humans"></a>
#### `Number::forHumans()` {.collection-method}

`Number::forHumans` 方法會回傳指定數值的易讀格式：

    use Illuminate\Support\Number;

    $number = Number::forHumans(1000);

    // 1 thousand

    $number = Number::forHumans(489939);

    // 490 thousand

    $number = Number::forHumans(1230000, precision: 2);

    // 1.23 million


<a name="method-number-format"></a>
#### `Number::format()` {.collection-method}

`Number::format` 方法會將給定數值格式化為特定語系字串：

    use Illuminate\Support\Number;

    $number = Number::format(100000);

    // 100,000

    $number = Number::format(100000, precision: 2);

    // 100,000.00

    $number = Number::format(100000.123, maxPrecision: 2);

    // 100,000.12

    $number = Number::format(100000, locale: 'de');

    // 100.000


<a name="method-number-ordinal"></a>
#### `Number::ordinal()` {.collection-method}

`Number::ordinal` 方法會回傳數值的序數表示法：

    use Illuminate\Support\Number;

    $number = Number::ordinal(1);

    // 1st

    $number = Number::ordinal(2);

    // 2nd

    $number = Number::ordinal(21);

    // 21st


<a name="method-number-pairs"></a>
#### `Number::pairs()` {.collection-method}

`Number::pairs` 方法會根據指定的範圍和步進值，產生一個由數字對 (子範圍) 組成的陣列。此方法在將較大的數字範圍劃分為較小、易於管理的子範圍時非常有用，例如分頁或批次處理任務。`pairs` 方法會回傳一個陣列的陣列，其中每個內部陣列代表一對 (子範圍) 數字：

```php
use Illuminate\Support\Number;

$result = Number::pairs(25, 10);

// [[1, 10], [11, 20], [21, 25]]

$result = Number::pairs(25, 10, offset: 0);

// [[0, 10], [10, 20], [20, 25]]
```


<a name="method-number-percentage"></a>
#### `Number::percentage()` {.collection-method}

`Number::percentage` 方法會將給定數值轉換為百分比字串表示法：

    use Illuminate\Support\Number;

    $percentage = Number::percentage(10);

    // 10%

    $percentage = Number::percentage(10, precision: 2);

    // 10.00%

    $percentage = Number::percentage(10.123, maxPrecision: 2);

    // 10.12%

    $percentage = Number::percentage(10, precision: 2, locale: 'de');

    // 10,00%


<a name="method-number-spell"></a>
#### `Number::spell()` {.collection-method}

`Number::spell` 方法會將給定數值轉換為文字字串：

    use Illuminate\Support\Number;

    $number = Number::spell(102);

    // one hundred and two

    $number = Number::spell(88, locale: 'fr');

    // quatre-vingt-huit

`after` 引數可讓您指定一個值，在此值之後的所有數字都應以文字拼出：

    $number = Number::spell(10, after: 10);

    // 10

    $number = Number::spell(11, after: 10);

    // eleven

`until` 引數可讓您指定一個值，在此值之前的所有數字都應以文字拼出：

    $number = Number::spell(5, until: 10);

    // five

    $number = Number::spell(10, until: 10);

    // 10


<a name="method-number-trim"></a>
#### `Number::trim()` {.collection-method}

`Number::trim` 方法會移除給定數值小數點後的所有尾隨零位數：

    use Illuminate\Support\Number;

    $number = Number::trim(12.0);

    // 12

    $number = Number::trim(12.30);

    // 12.3


<a name="method-number-use-locale"></a>
#### `Number::useLocale()` {.collection-method}

`Number::useLocale` 方法會全域設定預設數字語系，這會影響 `Number` class 後續方法調用時，數字和貨幣的格式化方式：

    use Illuminate\Support\Number;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Number::useLocale('de');
    }


<a name="method-number-with-locale"></a>
#### `Number::withLocale()` {.collection-method}

`Number::withLocale` 方法會使用指定的語系執行給定的 Closure，然後在 Callback 執行後恢復原始語系：

    use Illuminate\Support\Number;

    $number = Number::withLocale('de', function () {
        return Number::format(1500);
    });


<a name="method-number-use-currency"></a>
#### `Number::useCurrency()` {.collection-method}

`Number::useCurrency` 方法會全域設定預設數字貨幣，這會影響 `Number` class 後續方法調用時，貨幣的格式化方式：

    use Illuminate\Support\Number;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Number::useCurrency('GBP');
    }


<a name="method-number-with-currency"></a>
#### `Number::withCurrency()` {.collection-method}

`Number::withCurrency` 方法會使用指定的貨幣執行給定的 Closure，然後在 Callback 執行後恢復原始貨幣：

    use Illuminate\Support\Number;

    $number = Number::withCurrency('GBP', function () {
        // ...
    });

<a name="paths"></a>
## 路徑


<a name="method-app-path"></a>
#### `app_path()` {.collection-method}

`app_path` 函數會回傳應用程式 `app` 目錄的完整路徑。你也可以使用 `app_path` 函數來產生相對於應用程式目錄的檔案完整路徑：

    $path = app_path();

    $path = app_path('Http/Controllers/Controller.php');


<a name="method-base-path"></a>
#### `base_path()` {.collection-method}

`base_path` 函數會回傳應用程式根目錄的完整路徑。你也可以使用 `base_path` 函數來產生相對於專案根目錄的檔案完整路徑：

    $path = base_path();

    $path = base_path('vendor/bin');


<a name="method-config-path"></a>
#### `config_path()` {.collection-method}

`config_path` 函數會回傳應用程式 `config` 目錄的完整路徑。你也可以使用 `config_path` 函數來產生應用程式設定目錄中指定檔案的完整路徑：

    $path = config_path();

    $path = config_path('app.php');


<a name="method-database-path"></a>
#### `database_path()` {.collection-method}

`database_path` 函數會回傳應用程式 `database` 目錄的完整路徑。你也可以使用 `database_path` 函數來產生資料庫目錄中指定檔案的完整路徑：

    $path = database_path();

    $path = database_path('factories/UserFactory.php');


<a name="method-lang-path"></a>
#### `lang_path()` {.collection-method}

`lang_path` 函數會回傳應用程式 `lang` 目錄的完整路徑。你也可以使用 `lang_path` 函數來產生該目錄中指定檔案的完整路徑：

    $path = lang_path();

    $path = lang_path('en/messages.php');

> [!NOTE]  
> 預設情況下，Laravel 應用程式骨架不包含 `lang` 目錄。如果你想客製化 Laravel 的語言檔案，可以透過 `lang:publish` Artisan 命令來發佈它們。


<a name="method-mix"></a>
#### `mix()` {.collection-method}

`mix` 函數會回傳 [版本化 Mix 檔案](/docs/{{version}}/mix) 的路徑：

    $path = mix('css/app.css');


<a name="method-public-path"></a>
#### `public_path()` {.collection-method}

`public_path` 函數會回傳應用程式 `public` 目錄的完整路徑。你也可以使用 `public_path` 函數來產生 public 目錄中指定檔案的完整路徑：

    $path = public_path();

    $path = public_path('css/app.css');


<a name="method-resource-path"></a>
#### `resource_path()` {.collection-method}

`resource_path` 函數會回傳應用程式 `resources` 目錄的完整路徑。你也可以使用 `resource_path` 函數來產生 resources 目錄中指定檔案的完整路徑：

    $path = resource_path();

    $path = resource_path('sass/app.scss');


<a name="method-storage-path"></a>
#### `storage_path()` {.collection-method}

`storage_path` 函數會回傳應用程式 `storage` 目錄的完整路徑。你也可以使用 `storage_path` 函數來產生 storage 目錄中指定檔案的完整路徑：

    $path = storage_path();

    $path = storage_path('app/file.txt');


<a name="urls"></a>
## 網址


<a name="method-action"></a>
#### `action()` {.collection-method}

`action` 函數會為給定的控制器動作產生一個 URL：

    use App\Http\Controllers\HomeController;

    $url = action([HomeController::class, 'index']);

如果方法接受路由參數，你可以將它們作為第二個參數傳遞給方法：

    $url = action([UserController::class, 'profile'], ['id' => 1]);


<a name="method-asset"></a>
#### `asset()` {.collection-method}

`asset` 函數會使用請求目前的協定 (HTTP 或 HTTPS) 為資產產生一個 URL：

    $url = asset('img/photo.jpg');

你可以透過在 `.env` 檔案中設定 `ASSET_URL` 變數來配置資產 URL 主機。這在你將資產託管在外部服務 (如 Amazon S3 或其他 CDN) 上時會很有用：

    // ASSET_URL=http://example.com/assets

    $url = asset('img/photo.jpg'); // http://example.com/assets/img/photo.jpg


<a name="method-route"></a>
#### `route()` {.collection-method}

`route` 函數會為給定的 [命名路由](/docs/{{version}}/routing#named-routes) 產生一個 URL：

    $url = route('route.name');

如果路由接受參數，你可以將它們作為第二個參數傳遞給函數：

    $url = route('route.name', ['id' => 1]);

預設情況下，`route` 函數會產生一個絕對 URL。如果你想產生一個相對 URL，你可以將 `false` 作為第三個參數傳遞給函數：

    $url = route('route.name', ['id' => 1], false);


<a name="method-secure-asset"></a>
#### `secure_asset()` {.collection-method}

`secure_asset` 函數會使用 HTTPS 為資產產生一個 URL：

    $url = secure_asset('img/photo.jpg');


<a name="method-secure-url"></a>
#### `secure_url()` {.collection-method}

`secure_url` 函數會為給定的路徑產生一個完整的 HTTPS URL。其他 URL 片段可以作為第二個參數傳遞給函數：

    $url = secure_url('user/profile');

    $url = secure_url('user/profile', [1]);


<a name="method-to-route"></a>
#### `to_route()` {.collection-method}

`to_route` 函數會為給定的 [命名路由](/docs/{{version}}/routing#named-routes) 產生一個 [重定向 HTTP 回應](/docs/{{version}}/responses#redirects)：

    return to_route('users.show', ['user' => 1]);

如果需要，你可以將 HTTP 狀態碼和任何額外的回應標頭作為第三和第四個參數傳遞給 `to_route` 方法：

    return to_route('users.show', ['user' => 1], 302, ['X-Framework' => 'Laravel']);


<a name="method-url"></a>
#### `url()` {.collection-method}

`url` 函數會為給定的路徑產生一個完整的 URL：

    $url = url('user/profile');

    $url = url('user/profile', [1]);

如果未提供路徑，則會回傳 `Illuminate\Routing\UrlGenerator` 實例：

    $current = url()->current();

    $full = url()->full();

    $previous = url()->previous();

<a name="miscellaneous"></a>
## 雜項

<a name="method-abort"></a>
#### `abort()` {.collection-method}

`abort` 函數會拋出 [HTTP 異常](/docs/{{version}}/errors#http-exceptions)，此異常將由 [例外處理器](/docs/{{version}}/errors#handling-exceptions) 渲染：

    abort(403);

您也可以提供異常訊息和應傳送至瀏覽器的自訂 HTTP 回應標頭：

    abort(403, 'Unauthorized.', $headers);

<a name="method-abort-if"></a>
#### `abort_if()` {.collection-method}

`abort_if` 函數會在給定的布林表達式評估為 `true` 時拋出 HTTP 異常：

    abort_if(! Auth::user()->isAdmin(), 403);

與 `abort` 方法類似，您也可以將異常的回應文字作為第三個參數，以及自訂回應標頭陣列作為第四個參數傳遞給此函數。

<a name="method-abort-unless"></a>
#### `abort_unless()` {.collection-method}

`abort_unless` 函數會在給定的布林表達式評估為 `false` 時拋出 HTTP 異常：

    abort_unless(Auth::user()->isAdmin(), 403);

與 `abort` 方法類似，您也可以將異常的回應文字作為第三個參數，以及自訂回應標頭陣列作為第四個參數傳遞給此函數。

<a name="method-app"></a>
#### `app()` {.collection-method}

`app` 函數會回傳 [服務容器](/docs/{{version}}/container) 實例：

    $container = app();

您可以傳遞類別或介面名稱以從容器中解析它：

    $api = app('HelpSpot\API');

<a name="method-auth"></a>
#### `auth()` {.collection-method}

`auth` 函數會回傳一個 [認證器](/docs/{{version}}/authentication) 實例。您可以使用它作為 `Auth` Facade 的替代方案：

    $user = auth()->user();

如有需要，您可以指定要存取的 Guard 實例：

    $user = auth('admin')->user();

<a name="method-back"></a>
#### `back()` {.collection-method}

`back` 函數會產生一個將使用者重新導向回先前位置的 [HTTP 重新導向回應](/docs/{{version}}/responses#redirects)：

    return back($status = 302, $headers = [], $fallback = '/');

    return back();

<a name="method-bcrypt"></a>
#### `bcrypt()` {.collection-method}

`bcrypt` 函數會使用 Bcrypt [雜湊](/docs/{{version}}/hashing) 給定的值。您可以使用此函數作為 `Hash` Facade 的替代方案：

    $password = bcrypt('my-secret-password');

<a name="method-blank"></a>
#### `blank()` {.collection-method}

`blank` 函數會判斷給定的值是否為「空白」：

    blank('');
    blank('   ');
    blank(null);
    blank(collect());

    // true

    blank(0);
    blank(true);
    blank(false);

    // false

若要取得 `blank` 的反向結果，請參閱 [`filled`](#method-filled) 方法。

<a name="method-broadcast"></a>
#### `broadcast()` {.collection-method}

`broadcast` 函數會將給定的 [事件](/docs/{{version}}/events) [廣播](/docs/{{version}}/broadcasting) 給其監聽者：

    broadcast(new UserRegistered($user));

    broadcast(new UserRegistered($user))->toOthers();

<a name="method-cache"></a>
#### `cache()` {.collection-method}

`cache` 函數可用來從 [快取](/docs/{{version}}/cache) 中取得值。如果給定的鍵在快取中不存在，則會回傳一個可選的預設值：

    $value = cache('key');

    $value = cache('key', 'default');

您可以透過傳遞鍵/值對的陣列來向快取中加入項目。您也應該傳遞快取值應被視為有效的秒數或持續時間：

    cache(['key' => 'value'], 300);

    cache(['key' => 'value'], now()->addSeconds(10));

<a name="method-class-uses-recursive"></a>
#### `class_uses_recursive()` {.collection-method}

`class_uses_recursive` 函數會回傳一個類別使用的所有特性，包括其所有父類別使用的特性：

    $traits = class_uses_recursive(App\Models\User::class);

<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 函數會從給定的值建立一個 [Collection](/docs/{{version}}/collections) 實例：

    $collection = collect(['taylor', 'abigail']);

<a name="method-config"></a>
#### `config()` {.collection-method}

`config` 函數會取得 [配置](/docs/{{version}}/configuration) 變數的值。配置值可以使用「點」語法存取，其中包含檔案名稱和您要存取的選項。如果配置選項不存在，可以指定一個預設值並回傳：

    $value = config('app.timezone');

    $value = config('app.timezone', $default);

您可以在執行時透過傳遞鍵/值對的陣列來設定配置變數。但是請注意，此函數只會影響目前請求的配置值，而不會更新您實際的配置值：

    config(['app.debug' => true]);

<a name="method-context"></a>
#### `context()` {.collection-method}

`context` 函數會從 [目前上下文](/docs/{{version}}/context) 取得值。如果上下文鍵不存在，可以指定一個預設值並回傳：

    $value = context('trace_id');

    $value = context('trace_id', $default);

您可以透過傳遞鍵/值對的陣列來設定上下文值：

    use Illuminate\Support\Str;

    context(['trace_id' => Str::uuid()->toString()]);

<a name="method-cookie"></a>
#### `cookie()` {.collection-method}

`cookie` 函數會建立一個新的 [cookie](/docs/{{version}}/requests#cookies) 實例：

    $cookie = cookie('name', 'value', $minutes);

<a name="method-csrf-field"></a>
#### `csrf_field()` {.collection-method}

`csrf_field` 函數會產生一個包含 CSRF token 值的 HTML `hidden` 輸入欄位。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

    {{ csrf_field() }}

<a name="method-csrf-token"></a>
#### `csrf_token()` {.collection-method}

`csrf_token` 函數會擷取目前 CSRF token 的值：

    $token = csrf_token();

<a name="method-decrypt"></a>
#### `decrypt()` {.collection-method}

`decrypt` 函數會 [解密](/docs/{{version}}/encryption) 給定的值。您可以使用此函數作為 `Crypt` Facade 的替代方案：

    $password = decrypt($value);

<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 函數會傾印給定的變數並結束腳本的執行：

    dd($value);

    dd($value1, $value2, $value3, ...);

如果您不希望停止腳本的執行，請改用 [`dump`](#method-dump) 函數。

<a name="method-dispatch"></a>
#### `dispatch()` {.collection-method}

`dispatch` 函數會將給定的 [Job](/docs/{{version}}/queues#creating-jobs) 推送到 Laravel 的 [Job 佇列](/docs/{{version}}/queues)：

    dispatch(new App\Jobs\SendEmails);

<a name="method-dispatch-sync"></a>
#### `dispatch_sync()` {.collection-method}

`dispatch_sync` 函數會將給定的 Job 推送到 [同步佇列](/docs/{{version}}/queues#synchronous-dispatching)，使其立即處理：

    dispatch_sync(new App\Jobs\SendEmails);

<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 函數會傾印給定的變數：

    dump($value);

    dump($value1, $value2, $value3, ...);

如果您想在傾印變數後停止執行腳本，請改用 [`dd`](#method-dd) 函數。

<a name="method-encrypt"></a>
#### `encrypt()` {.collection-method}

`encrypt` 函數會 [加密](/docs/{{version}}/encryption) 給定的值。您可以使用此函數作為 `Crypt` Facade 的替代方案：

    $secret = encrypt('my-secret-value');

<a name="method-env"></a>
#### `env()` {.collection-method}

`env` 函數會擷取 [環境變數](/docs/{{version}}/configuration#environment-configuration) 的值，或返回一個預設值：

    $env = env('APP_ENV');

    $env = env('APP_ENV', 'production');

> [!WARNING]  
> 若您在部署過程中執行 `config:cache` 命令，請務必確保您僅從設定檔中呼叫 `env` 函數。一旦設定被快取，`.env` 檔案將不會被載入，並且所有對 `env` 函數的呼叫都將返回 `null`。


<a name="method-event"></a>
#### `event()` {.collection-method}

`event` 函數會將給定的 [Event](/docs/{{version}}/events) 派發給其監聽器：

    event(new UserRegistered($user));


<a name="method-fake"></a>
#### `fake()` {.collection-method}

`fake` 函數從容器中解析 [Faker](https://github.com/FakerPHP/Faker) 單例，這在模型工廠、資料庫填充、測試和原型視圖中建立假資料時非常有用：

```blade
@for($i = 0; $i < 10; $i++)
    <dl>
        <dt>Name</dt>
        <dd>{{ fake()->name() }}</dd>

        <dt>Email</dt>
        <dd>{{ fake()->unique()->safeEmail() }}</dd>
    </dl>
@endfor
```

預設情況下，`fake` 函數會利用您 `config/app.php` 設定檔中的 `app.faker_locale` 設定選項。通常，此設定選項透過 `APP_FAKER_LOCALE` 環境變數進行設定。您也可以透過將語系傳遞給 `fake` 函數來指定語系。每個語系將解析為一個獨立的單例：

    fake('nl_NL')->name()


<a name="method-filled"></a>
#### `filled()` {.collection-method}

`filled` 函數判斷給定的值是否不為「空白」：

    filled(0);
    filled(true);
    filled(false);

    // true

    filled('');
    filled('   ');
    filled(null);
    filled(collect());

    // false

有關 `filled` 的相反函數，請參閱 [`blank`](#method-blank) 方法。


<a name="method-info"></a>
#### `info()` {.collection-method}

`info` 函數會將資訊寫入您應用程式的 [日誌](/docs/{{version}}/logging)：

    info('Some helpful information!');

上下文字資料的陣列也可以傳遞給該函數：

    info('User login attempt failed.', ['id' => $user->id]);


<a name="method-literal"></a>
#### `literal()` {.collection-method}

`literal` 函數會建立一個新的 [stdClass](https://www.php.net/manual/en/class.stdclass.php) 實例，並將給定的具名引數作為屬性：

    $obj = literal(
        name: 'Joe',
        languages: ['PHP', 'Ruby'],
    );

    $obj->name; // 'Joe'
    $obj->languages; // ['PHP', 'Ruby']


<a name="method-logger"></a>
#### `logger()` {.collection-method}

`logger` 函數可用於將 `debug` 等級訊息寫入 [日誌](/docs/{{version}}/logging)：

    logger('Debug message');

上下文字資料的陣列也可以傳遞給該函數：

    logger('User has logged in.', ['id' => $user->id]);

如果沒有將任何值傳遞給該函數，則會返回一個 [logger](/docs/{{version}}/logging) 實例：

    logger()->error('You are not allowed here.');


<a name="method-method-field"></a>
#### `method_field()` {.collection-method}

`method_field` 函數會產生一個 HTML `hidden` 輸入欄位，其中包含表單 HTTP 動詞的偽造值。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

    <form method="POST">
        {{ method_field('DELETE') }}
    </form>


<a name="method-now"></a>
#### `now()` {.collection-method}

`now` 函數會為當前時間建立一個新的 `Illuminate\Support\Carbon` 實例：

    $now = now();


<a name="method-old"></a>
#### `old()` {.collection-method}

`old` 函數會從 Session 中擷取快閃的 [舊輸入](/docs/{{version}}/requests#old-input) 值：

    $value = old('value');

    $value = old('value', 'default');

由於作為 `old` 函數第二個引數提供的「預設值」通常是 Eloquent 模型的一個屬性，Laravel 允許您直接將整個 Eloquent 模型作為 `old` 函數的第二個引數傳入。這樣做時，Laravel 會假定提供給 `old` 函數的第一個引數是應被視為「預設值」的 Eloquent 屬性名稱：

    {{ old('name', $user->name) }}

    // 這等同於...

    {{ old('name', $user) }}


<a name="method-once"></a>
#### `once()` {.collection-method}

`once` 函數會執行給定的回呼，並在請求期間將結果快取於記憶體中。任何後續對 `once` 函數使用相同回呼的呼叫都將返回先前快取的結果：

    function random(): int
    {
        return once(function () {
            return random_int(1, 1000);
        });
    }

    random(); // 123
    random(); // 123 (cached result)
    random(); // 123 (cached result)

當 `once` 函數從物件實例中執行時，快取結果將是該物件實例專屬的：

```php
<?php

class NumberService
{
    public function all(): array
    {
        return once(fn () => [1, 2, 3]);
    }
}

$service = new NumberService;

$service->all();
$service->all(); // (cached result)

$secondService = new NumberService;

$secondService->all();
$secondService->all(); // (cached result)
```

<a name="method-optional"></a>
#### `optional()` {.collection-method}

`optional` 函數接受任何引數，並允許您存取該物件的屬性或呼叫其方法。如果給定的物件是 `null`，屬性與方法將返回 `null` 而不會導致錯誤：

    return optional($user->address)->street;

    {!! old('name', optional($user)->name) !!}

`optional` 函數也接受一個閉包作為其第二個引數。如果作為第一個引數提供的值不為 `null`，則會呼叫該閉包：

    return optional(User::find($id), function (User $user) {
        return $user->name;
    });


<a name="method-policy"></a>
#### `policy()` {.collection-method}

`policy` 方法會擷取給定類別的 [Policy](/docs/{{version}}/authorization#creating-policies) 實例：

    $policy = policy(App\Models\User::class);


<a name="method-redirect"></a>
#### `redirect()` {.collection-method}

`redirect` 函數會返回一個 [重導向 HTTP 回應](/docs/{{version}}/responses#redirects)，如果沒有引數呼叫，則會返回重導向器實例：

    return redirect($to = null, $status = 302, $headers = [], $https = null);

    return redirect('/home');

    return redirect()->route('route.name');


<a name="method-report"></a>
#### `report()` {.collection-method}

`report` 函數會使用您的 [例外處理器](/docs/{{version}}/errors#handling-exceptions) 報告例外：

    report($e);

`report` 函數也接受一個字串作為引數。當給定字串時，該函數將建立一個以該字串作為訊息的例外：

    report('Something went wrong.');


<a name="method-report-if"></a>
#### `report_if()` {.collection-method}

如果給定條件為 `true`，`report_if` 函數會使用您的 [例外處理器](/docs/{{version}}/errors#handling-exceptions) 報告例外：

    report_if($shouldReport, $e);

    report_if($shouldReport, 'Something went wrong.');


<a name="method-report-unless"></a>
#### `report_unless()` {.collection-method}

如果給定條件為 `false`，`report_unless` 函數會使用您的 [例外處理器](/docs/{{version}}/errors#handling-exceptions) 報告例外：

    report_unless($reportingDisabled, $e);

    report_unless($reportingDisabled, 'Something went wrong.');


<a name="method-request"></a>
#### `request()` {.collection-method}

`request` 函數會返回當前的 [請求](/docs/{{version}}/requests) 實例，或從當前請求中取得輸入欄位的值：

    $request = request();

    $value = request('key', $default);

<a name="method-rescue"></a>
#### `rescue()` {.collection-method}

`rescue` 函數會執行給定的閉包，並捕獲在執行期間發生的任何異常。所有被捕獲的異常將會被傳送至您的[異常處理器](/docs/{{version}}/errors#handling-exceptions)；然而，請求會繼續處理：

    return rescue(function () {
        return $this->method();
    });

您也可以將第二個引數傳遞給 `rescue` 函數。如果閉包在執行期間發生異常，此引數將會是應該回傳的「預設」值：

    return rescue(function () {
        return $this->method();
    }, false);

    return rescue(function () {
        return $this->method();
    }, function () {
        return $this->failure();
    });

可以為 `rescue` 函數提供 `report` 引數，以決定是否應透過 `report` 函數報告異常：

    return rescue(function () {
        return $this->method();
    }, report: function (Throwable $throwable) {
        return $throwable instanceof InvalidArgumentException;
    });


<a name="method-resolve"></a>
#### `resolve()` {.collection-method}

`resolve` 函數使用[服務容器](/docs/{{version}}/container)將給定的類別或介面名稱解析為實例：

    $api = resolve('HelpSpot\API');


<a name="method-response"></a>
#### `response()` {.collection-method}

`response` 函數會建立一個[回應](/docs/{{version}}/responses)實例，或取得回應工廠的實例：

    return response('Hello World', 200, $headers);

    return response()->json(['foo' => 'bar'], 200, $headers);


<a name="method-retry"></a>
#### `retry()` {.collection-method}

`retry` 函數會嘗試執行給定的回呼，直到達到設定的最大嘗試次數。如果回呼沒有拋出異常，將會回傳其回傳值。如果回呼拋出異常，它將自動重試。如果超過最大嘗試次數，則會拋出異常：

    return retry(5, function () {
        // Attempt 5 times while resting 100ms between attempts...
    }, 100);

如果您想手動計算每次嘗試之間休眠的毫秒數，您可以將一個閉包作為第三個引數傳遞給 `retry` 函數：

    use Exception;

    return retry(5, function () {
        // ...
    }, function (int $attempt, Exception $exception) {
        return $attempt * 100;
    });

為方便起見，您可以將一個陣列作為第一個引數傳遞給 `retry` 函數。此陣列將用於確定後續嘗試之間休眠的毫秒數：

    return retry([100, 200], function () {
        // Sleep for 100ms on first retry, 200ms on second retry...
    });

若要在特定條件下才重試，您可以將一個閉包作為第四個引數傳遞給 `retry` 函數：

    use Exception;

    return retry(5, function () {
        // ...
    }, 100, function (Exception $exception) {
        return $exception instanceof RetryException;
    });


<a name="method-session"></a>
#### `session()` {.collection-method}

`session` 函數可用於取得或設定 [Session](/docs/{{version}}/session) 值：

    $value = session('key');

您可以透過將鍵/值對的陣列傳遞給函數來設定值：

    session(['chairs' => 7, 'instruments' => 3]);

如果沒有將任何值傳遞給函數，則會回傳 Session 儲存庫：

    $value = session()->get('key');

    session()->put('key', $value);


<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 函數接受兩個引數：一個任意的 `$value` 和一個閉包。`$value` 將會被傳遞給閉包，然後由 `tap` 函數回傳。閉包的回傳值是無關緊要的：

    $user = tap(User::first(), function (User $user) {
        $user->name = 'taylor';

        $user->save();
    });

如果沒有將閉包傳遞給 `tap` 函數，您可以在給定的 `$value` 上呼叫任何方法。您呼叫的方法的回傳值將始終是 `$value`，無論該方法在其定義中實際回傳什麼。例如，Eloquent 的 `update` 方法通常會回傳一個整數。但是，我們可以透過將 `update` 方法呼叫鏈式傳遞給 `tap` 函數，來強制該方法回傳模型本身：

    $user = tap($user)->update([
        'name' => $name,
        'email' => $email,
    ]);

若要將 `tap` 方法新增到類別，您可以將 `Illuminate\Support\Traits\Tappable` Trait 新增到該類別。此 Trait 的 `tap` 方法只接受一個閉包作為其唯一引數。物件實例本身將會被傳遞給閉包，然後由 `tap` 方法回傳：

    return $user->tap(function (User $user) {
        // ...
    });


<a name="method-throw-if"></a>
#### `throw_if()` {.collection-method}

如果給定的布林表達式評估為 `true`，`throw_if` 函數會拋出給定的異常：

    throw_if(! Auth::user()->isAdmin(), AuthorizationException::class);

    throw_if(
        ! Auth::user()->isAdmin(),
        AuthorizationException::class,
        '您不允許存取此頁面。'
    );


<a name="method-throw-unless"></a>
#### `throw_unless()` {.collection-method}

如果給定的布林表達式評估為 `false`，`throw_unless` 函數會拋出給定的異常：

    throw_unless(Auth::user()->isAdmin(), AuthorizationException::class);

    throw_unless(
        Auth::user()->isAdmin(),
        AuthorizationException::class,
        '您不允許存取此頁面。'
    );


<a name="method-today"></a>
#### `today()` {.collection-method}

`today` 函數會為當前日期建立一個新的 `Illuminate\Support\Carbon` 實例：

    $today = today();


<a name="method-trait-uses-recursive"></a>
#### `trait_uses_recursive()` {.collection-method}

`trait_uses_recursive` 函數會回傳 Trait 所使用的所有 Trait：

    $traits = trait_uses_recursive(\Illuminate\Notifications\Notifiable::class);


<a name="method-transform"></a>
#### `transform()` {.collection-method}

如果給定的值不是[空白](#method-blank)，`transform` 函數會對該值執行一個閉包，然後回傳閉包的回傳值：

    $callback = function (int $value) {
        return $value * 2;
    };

    $result = transform(5, $callback);

    // 10

可以將預設值或閉包作為第三個引數傳遞給函數。如果給定的值是空白，則會回傳此值：

    $result = transform(null, $callback, '該值為空白');

    // The value is blank


<a name="method-validator"></a>
#### `validator()` {.collection-method}

`validator` 函數會使用給定的引數建立一個新的[驗證器](/docs/{{version}}/validation)實例。您可以將其作為 `Validator` Facade 的替代方案：

    $validator = validator($data, $rules, $messages);


<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 函數會回傳給定的值。但是，如果將閉包傳遞給函數，則閉包將被執行並回傳其回傳值：

    $result = value(true);

    // true

    $result = value(function () {
        return false;
    });

    // false

可以將額外引數傳遞給 `value` 函數。如果第一個引數是閉包，則額外參數將作為引數傳遞給閉包，否則將被忽略：

    $result = value(function (string $name) {
        return $name;
    }, 'Taylor');

    // 'Taylor'


<a name="method-view"></a>
#### `view()` {.collection-method}

`view` 函數會取得一個[視圖](/docs/{{version}}/views)實例：

    return view('auth.login');

<a name="method-with"></a>
#### `with()` {.collection-method}

`with` 函數會回傳所給予的值。如果一個閉包作為第二個參數傳遞給函數，該閉包將被執行並回傳其返回值：

    $callback = function (mixed $value) {
        return is_numeric($value) ? $value * 2 : 0;
    };

    $result = with(5, $callback);

    // 10

    $result = with(null, $callback);

    // 0

    $result = with(5, null);

    // 5

<a name="method-when"></a>
#### `when()` {.collection-method}

`when` 函數在給定條件為 `true` 時，回傳所給予的值。否則，回傳 `null`。如果一個閉包作為第二個參數傳遞給函數，該閉包將被執行並回傳其返回值：

    $value = when(true, 'Hello World');

    $value = when(true, fn () => 'Hello World');

`when` 函數主要用於條件式地渲染 HTML 屬性：

```blade
<div {!! when($condition, 'wire:poll="calculate"') !!}>
    ...
</div>
```

<a name="other-utilities"></a>
## 其他工具

<a name="benchmarking"></a>
### 效能基準測試

有時您可能希望快速測試應用程式中特定部分的效能。在這些情況下，您可以使用 `Benchmark` 輔助類別來測量給定回呼完成所需的毫秒數：

    <?php

    use App\Models\User;
    use Illuminate\Support\Benchmark;

    Benchmark::dd(fn () => User::find(1)); // 0.1 ms

    Benchmark::dd([
        'Scenario 1' => fn () => User::count(), // 0.5 ms
        'Scenario 2' => fn () => User::all()->count(), // 20.0 ms
    ]);

依預設，給定的回呼將執行一次（一次迭代），其執行時間將顯示在瀏覽器或主控台中。

若要多次呼叫回呼，您可以將回呼應呼叫的迭代次數指定為方法的第二個引數。當多次執行回呼時，`Benchmark` 類別將傳回執行所有迭代的回呼所需的平均毫秒數：

    Benchmark::dd(fn () => User::count(), iterations: 10); // 0.5 ms

有時，您可能希望在取得回呼傳回的值的同時，對回呼的執行進行基準測試。`value` 方法將傳回一個陣列，其中包含回呼傳回的值以及執行回呼所需的毫秒數：

    [$count, $duration] = Benchmark::value(fn () => User::count());

<a name="dates"></a>
### 日期

Laravel 包含 [Carbon](https://carbon.nesbot.com/docs/)，一個功能強大的日期和時間處理函式庫。要建立新的 `Carbon` 實例，您可以呼叫 `now` 函數。此函數在您的 Laravel 應用程式中是全域可用的：

```php
$now = now();
```

或者，您可以使用 `Illuminate\Support\Carbon` 類別建立新的 `Carbon` 實例：

```php
use Illuminate\Support\Carbon;

$now = Carbon::now();
```

有關 Carbon 及其功能的詳細討論，請查閱 [官方 Carbon 文件](https://carbon.nesbot.com/docs/)。

<a name="deferred-functions"></a>
### 延遲函數

> [!WARNING]
> 延遲函數目前處於 Beta 版，我們正在收集社群回饋。

雖然 Laravel 的 [佇列任務](/docs/{{version}}/queues) 允許您將任務排入佇列進行背景處理，但有時您可能有一些簡單的任務，希望延遲執行，而無需設定或維護長時間運行的佇列工作者。

延遲函數允許您將閉包的執行延遲到 HTTP 回應發送給使用者之後，讓您的應用程式感覺快速且響應迅速。要延遲閉包的執行，只需將閉包傳遞給 `Illuminate\Support\defer` 函數即可：

```php
use App\Services\Metrics;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use function Illuminate\Support\defer;

Route::post('/orders', function (Request $request) {
    // Create order...

    defer(fn () => Metrics::reportOrder($order));

    return $order;
});
```

依預設，延遲函數只有在從中呼叫 `Illuminate\Support\defer` 的 HTTP 回應、Artisan 命令或佇列任務成功完成時才會執行。這表示如果請求導致 `4xx` 或 `5xx` HTTP 回應，延遲函數將不會執行。如果您希望延遲函數始終執行，您可以將 `always` 方法鏈接到您的延遲函數上：

```php
defer(fn () => Metrics::reportOrder($order))->always();
```

<a name="cancelling-deferred-functions"></a>
#### 取消延遲函數

如果您需要在延遲函數執行之前取消它，您可以使用 `forget` 方法透過其名稱取消該函數。要命名延遲函數，請向 `Illuminate\Support\defer` 函數提供第二個引數：

```php
defer(fn () => Metrics::report(), 'reportMetrics');

defer()->forget('reportMetrics');
```

<a name="deferred-function-compatibility"></a>
#### 延遲函數相容性

如果您將應用程式從 Laravel 10.x 升級到 Laravel 11.x，並且您的應用程式骨架仍然包含 `app/Http/Kernel.php` 檔案，您應該將 `InvokeDeferredCallbacks` 中介層新增到 Kernel 的 `$middleware` 屬性開頭：

```php
protected $middleware = [
    \Illuminate\Foundation\Http\Middleware\InvokeDeferredCallbacks::class, // [tl! add]
    \App\Http\Middleware\TrustProxies::class,
    // ...
];
```

<a name="disabling-deferred-functions-in-tests"></a>
#### 於測試中禁用延遲函數

在編寫測試時，禁用延遲函數可能很有用。您可以在測試中呼叫 `withoutDefer` 來指示 Laravel 立即呼叫所有延遲函數：

```php tab=Pest
test('without defer', function () {
    $this->withoutDefer();

    // ...
});
```

```php tab=PHPUnit
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_without_defer(): void
    {
        $this->withoutDefer();

        // ...
    }
}
```

如果您希望為測試案例中的所有測試禁用延遲函數，您可以在基礎 `TestCase` 類別的 `setUp` 方法中呼叫 `withoutDefer` 方法：

```php
<?php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUp(): void// [tl! add:start]
    {
        parent::setUp();

        $this->withoutDefer();
    }// [tl! add:end]
}
```

<a name="lottery"></a>
### 抽獎

Laravel 的 Lottery 類別可用於根據一組給定機率執行回呼。當您只想對一部分傳入請求執行程式碼時，這會特別有用：

    use Illuminate\Support\Lottery;

    Lottery::odds(1, 20)
        ->winner(fn () => $user->won())
        ->loser(fn () => $user->lost())
        ->choose();

您可以將 Laravel 的 Lottery 類別與其他 Laravel 功能結合使用。例如，您可能希望僅向您的異常處理器報告一小部分的慢速查詢。而且，由於 Lottery 類別是可呼叫的，我們可以將類別實例傳遞給任何接受可呼叫的方法：

    use Carbon\CarbonInterval;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Lottery;

    DB::whenQueryingForLongerThan(
        CarbonInterval::seconds(2),
        Lottery::odds(1, 100)->winner(fn () => report('Querying > 2 seconds.')),
    );

<a name="testing-lotteries"></a>
#### 測試抽獎

Laravel 提供了一些簡單的方法，讓您可以輕鬆測試應用程式的 Lottery 呼叫：

    // Lottery 將始終贏...
    Lottery::alwaysWin();

    // Lottery 將始終輸...
    Lottery::alwaysLose();

    // Lottery 將先贏後輸，最後恢復正常行為...
    Lottery::fix([true, false]);

    // Lottery 將恢復正常行為...
    Lottery::determineResultsNormally();

<a name="pipeline"></a>
### 管道

Laravel 的 `Pipeline` 外觀 (Facade) 提供了一種方便的方式，可以將給定的輸入透過一系列可呼叫的類別、閉包或可呼叫物進行「管道」處理，讓每個類別都有機會檢查或修改輸入，並呼叫管道中的下一個可呼叫物：

```php
use Closure;
use App\Models\User;
use Illuminate\Support\Facades\Pipeline;

$user = Pipeline::send($user)
    ->through([
        function (User $user, Closure $next) {
            // ...
    
            return $next($user);
        },
        function (User $user, Closure $next) {
            // ...
    
            return $next($user);
        },
    ])
    ->then(fn (User $user) => $user);
```

如您所見，管道中的每個可呼叫類別或閉包都會接收到輸入和一個 `$next` 閉包。呼叫 `$next` 閉包將會呼叫管道中的下一個可呼叫物。您可能已經注意到，這與 [中介層](/docs/{{version}}/middleware) 非常相似。

當管道中的最後一個可呼叫物呼叫 `$next` 閉包時，提供給 `then` 方法的可呼叫物將會被呼叫。通常，這個可呼叫物只會簡單地返回給定的輸入。

當然，如前所述，您不限於只向管道提供閉包。您也可以提供可呼叫類別。如果提供了類別名稱，該類別將透過 Laravel 的 [服務容器](/docs/{{version}}/container) 實例化，從而允許將依賴項注入到可呼叫類別中：

```php
$user = Pipeline::send($user)
    ->through([
        GenerateProfilePhoto::class,
        ActivateSubscription::class,
        SendWelcomeEmail::class,
    ])
    ->then(fn (User $user) => $user);
```

<a name="sleep"></a>
### 休眠

Laravel 的 `Sleep` 類別是對 PHP 原生 `sleep` 和 `usleep` 函數的輕量級封裝，提供了更高的可測試性，同時也為處理時間暴露了開發者友好的 API：

    use Illuminate\Support\Sleep;

    $waiting = true;

    while ($waiting) {
        Sleep::for(1)->second();

        $waiting = /* ... */;
    }

`Sleep` 類別提供了多種方法，讓您可以處理不同的時間單位：

    // Return a value after sleeping...
    $result = Sleep::for(1)->second()->then(fn () => 1 + 1);

    // Sleep while a given value is true...
    Sleep::for(1)->second()->while(fn () => shouldKeepSleeping());

    // Pause execution for 90 seconds...
    Sleep::for(1.5)->minutes();

    // Pause execution for 2 seconds...
    Sleep::for(2)->seconds();

    // Pause execution for 500 milliseconds...
    Sleep::for(500)->milliseconds();

    // Pause execution for 5,000 microseconds...
    Sleep::for(5000)->microseconds();

    // Pause execution until a given time...
    Sleep::until(now()->addMinute());

    // Alias of PHP's native "sleep" function...
    Sleep::sleep(2);

    // Alias of PHP's native "usleep" function...
    Sleep::usleep(5000);

為了方便地組合時間單位，您可以使用 `and` 方法：

    Sleep::for(1)->second()->and(10)->milliseconds();

<a name="testing-sleep"></a>
#### 測試休眠

測試利用 `Sleep` 類別或 PHP 原生 sleep 函數的程式碼時，您的測試將會暫停執行。正如您所料，這會使您的測試套件顯著變慢。例如，假設您正在測試以下程式碼：

    $waiting = /* ... */;

    $seconds = 1;

    while ($waiting) {
        Sleep::for($seconds++)->seconds();

        $waiting = /* ... */;
    }

通常，測試這段程式碼至少需要一秒鐘。幸運的是，`Sleep` 類別允許我們「偽造」休眠，從而讓我們的測試套件保持快速：

```php tab=Pest
it('waits until ready', function () {
    Sleep::fake();

    // ...
});
```

```php tab=PHPUnit
public function test_it_waits_until_ready()
{
    Sleep::fake();

    // ...
}
```

當偽造 `Sleep` 類別時，實際的執行暫停會被繞過，從而使測試顯著加快。

一旦 `Sleep` 類別被偽造，就可以對預期發生的「休眠」進行斷言。為了說明這一點，讓我們想像我們正在測試一段程式碼，它會暫停執行三次，每次暫停的時間增加一秒。使用 `assertSequence` 方法，我們可以斷言我們的程式碼在保持測試快速的同時「休眠」了適當的時間：

```php tab=Pest
it('checks if ready three times', function () {
    Sleep::fake();

    // ...

    Sleep::assertSequence([
        Sleep::for(1)->second(),
        Sleep::for(2)->seconds(),
        Sleep::for(3)->seconds(),
    ]);
}
```

```php tab=PHPUnit
public function test_it_checks_if_ready_three_times()
{
    Sleep::fake();

    // ...

    Sleep::assertSequence([
        Sleep::for(1)->second(),
        Sleep::for(2)->seconds(),
        Sleep::for(3)->seconds(),
    ]);
}
```

當然，`Sleep` 類別還提供了多種其他斷言，您可以在測試時使用：

    use Carbon\CarbonInterval as Duration;
    use Illuminate\Support\Sleep;

    // Assert that sleep was called 3 times...
    Sleep::assertSleptTimes(3);

    // Assert against the duration of sleep...
    Sleep::assertSlept(function (Duration $duration): bool {
        return /* ... */;
    }, times: 1);

    // Assert that the Sleep class was never invoked...
    Sleep::assertNeverSlept();

    // Assert that, even if Sleep was called, no execution paused occurred...
    Sleep::assertInsomniac();

有時，在應用程式碼中發生偽造休眠時執行某些操作可能會很有用。為此，您可以向 `whenFakingSleep` 方法提供一個回呼。在以下範例中，我們使用 Laravel 的 [時間操縱輔助函數](/docs/{{version}}/mocking#interacting-with-time) 來根據每次休眠的持續時間立即推進時間：

```php
use Carbon\CarbonInterval as Duration;

$this->freezeTime();

Sleep::fake();

Sleep::whenFakingSleep(function (Duration $duration) {
    // Progress time when faking sleep...
    $this->travel($duration->totalMilliseconds)->milliseconds();
});
```

由於推進時間是一個常見的需求，`fake` 方法接受 `syncWithCarbon` 參數，以便在測試中休眠時保持 Carbon 同步：

```php
Sleep::fake(syncWithCarbon: true);

$start = now();

Sleep::for(1)->second();

$start->diffForHumans(); // 1 second ago
```

Laravel 在內部暫停執行時會使用 `Sleep` 類別。例如，[`retry`](#method-retry) 輔助函數在休眠時使用 `Sleep` 類別，從而提高了使用該輔助函數時的可測試性。

<a name="timebox"></a>
### 時間箱

Laravel 的 `Timebox` 類別確保給定的回呼總是花費固定的時間執行，即使其實際執行完成得更快。這對於加密操作和使用者身份驗證檢查特別有用，因為攻擊者可能會利用執行時間的變化來推斷敏感資訊。

如果執行時間超過了固定持續時間，`Timebox` 則沒有效果。開發人員需要選擇足夠長的時間作為固定持續時間，以應對最壞情況。

call 方法接受一個閉包和一個以微秒為單位、時間限制，然後執行閉包並等待直到達到時間限制：

```php
use Illuminate\Support\Timebox;

(new Timebox)->call(function ($timebox) {
    // ...
}, microseconds: 10000);
```

如果在閉包中拋出異常，此類別將遵守定義的延遲，並在延遲之後重新拋出異常。