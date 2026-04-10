# Collections

- [入門](#introduction)
    - [建立集合](#creating-collections)
    - [擴充集合](#extending-collections)
- [可用方法](#available-methods)
- [高階訊息 (Higher Order Messages)](#higher-order-messages)
- [Lazy 集合](#lazy-collections)
    - [入門](#lazy-collection-introduction)
    - [建立 Lazy 集合](#creating-lazy-collections)
    - [Enumerable 契約](#the-enumerable-contract)
    - [Lazy 集合方法](#lazy-collection-methods)

<a name="introduction"></a>
## 入門

`Illuminate\Support\Collection` 類別為處理資料陣列提供了一個流暢且方便的封裝。例如，請參考以下程式碼。我們將使用 `collect` 輔助函式從陣列建立一個新的集合實例，在每個元素上執行 `strtoupper` 函式，然後移除所有空元素：

```php
$collection = collect(['Taylor', 'Abigail', null])->map(function (?string $name) {
    return strtoupper($name);
})->reject(function (string $name) {
    return empty($name);
});
```

如您所見，`Collection` 類別允許您鏈結其方法，以對底層陣列執行流暢的映射與簡約。一般來說，集合是不可變的 (immutable)，這意味著每個 `Collection` 方法都會回傳一個全新的 `Collection` 實例。


<a name="creating-collections"></a>
### 建立集合

如上所述，`collect` 輔助函式會為給定的陣列回傳一個新的 `Illuminate\Support\Collection` 實例。因此，建立集合非常簡單：

```php
$collection = collect([1, 2, 3]);
```

您也可以使用 [make](#method-make) 和 [fromJson](#method-fromjson) 方法來建立集合。

> [!NOTE]
> [Eloquent](/docs/{{version}}/eloquent) 查詢的結果總是會以 `Collection` 實例形式回傳。


<a name="extending-collections"></a>
### 擴充集合

集合是「可巨集擴充的 (macroable)」，這允許您在執行時期向 `Collection` 類別新增額外的方法。`Illuminate\Support\Collection` 類別的 `macro` 方法接受一個閉包，該閉包將在呼叫巨集時執行。巨集閉包可以透過 `$this` 存取集合的其他方法，就像它是集合類別的真實方法一樣。例如，以下程式碼向 `Collection` 類別新增了一個 `toUpper` 方法：

```php
use Illuminate\Support\Collection;
use Illuminate\Support\Str;

Collection::macro('toUpper', function () {
    return $this->map(function (string $value) {
        return Str::upper($value);
    });
});

$collection = collect(['first', 'second']);

$upper = $collection->toUpper();

// ['FIRST', 'SECOND']
```

通常情況下，您應該在[服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中宣告集合巨集。


<a name="macro-arguments"></a>
#### 巨集參數

如有必要，您可以定義接受額外參數的巨集：

```php
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Lang;

Collection::macro('toLocale', function (string $locale) {
    return $this->map(function (string $value) use ($locale) {
        return Lang::get($value, [], $locale);
    });
});

$collection = collect(['first', 'second']);

$translated = $collection->toLocale('es');

// ['primero', 'segundo'];
```


<a name="available-methods"></a>
## 可用方法

在剩餘的大部分集合文件中，我們將討論 `Collection` 類別中可用的每個方法。請記住，所有這些方法都可以鏈結在一起，以流暢地操作底層陣列。此外，幾乎每個方法都會回傳一個新的 `Collection` 實例，這允許您在必要時保留集合的原始副本：

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

<div class="collection-method-list" markdown="1">

[after](#method-after)
[all](#method-all)
[average](#method-average)
[avg](#method-avg)
[before](#method-before)
[chunk](#method-chunk)
[chunkWhile](#method-chunkwhile)
[collapse](#method-collapse)
[collapseWithKeys](#method-collapsewithkeys)
[collect](#method-collect)
[combine](#method-combine)
[concat](#method-concat)
[contains](#method-contains)
[containsStrict](#method-containsstrict)
[count](#method-count)
[countBy](#method-countBy)
[crossJoin](#method-crossjoin)
[dd](#method-dd)
[diff](#method-diff)
[diffAssoc](#method-diffassoc)
[diffAssocUsing](#method-diffassocusing)
[diffKeys](#method-diffkeys)
[doesntContain](#method-doesntcontain)
[doesntContainStrict](#method-doesntcontainstrict)
[dot](#method-dot)
[dump](#method-dump)
[duplicates](#method-duplicates)
[duplicatesStrict](#method-duplicatesstrict)
[each](#method-each)
[eachSpread](#method-eachspread)
[ensure](#method-ensure)
[every](#method-every)
[except](#method-except)
[filter](#method-filter)
[first](#method-first)
[firstOrFail](#method-first-or-fail)
[firstWhere](#method-first-where)
[flatMap](#method-flatmap)
[flatten](#method-flatten)
[flip](#method-flip)
[forget](#method-forget)
[forPage](#method-forpage)
[fromJson](#method-fromjson)
[get](#method-get)
[groupBy](#method-groupby)
[has](#method-has)
[hasAny](#method-hasany)
[hasMany](#method-hasmany)
[hasSole](#method-hassole)
[implode](#method-implode)
[intersect](#method-intersect)
[intersectUsing](#method-intersectusing)
[intersectAssoc](#method-intersectAssoc)
[intersectAssocUsing](#method-intersectassocusing)
[intersectByKeys](#method-intersectbykeys)
[isEmpty](#method-isempty)
[isNotEmpty](#method-isnotempty)
[join](#method-join)
[keyBy](#method-keyby)
[keys](#method-keys)
[last](#method-last)
[lazy](#method-lazy)
[macro](#method-macro)
[make](#method-make)
[map](#method-map)
[mapInto](#method-mapinto)
[mapSpread](#method-mapspread)
[mapToGroups](#method-maptogroups)
[mapWithKeys](#method-mapwithkeys)
[max](#method-max)
[median](#method-median)
[merge](#method-merge)
[mergeRecursive](#method-mergerecursive)
[min](#method-min)
[mode](#method-mode)
[multiply](#method-multiply)
[nth](#method-nth)
[only](#method-only)
[pad](#method-pad)
[partition](#method-partition)
[percentage](#method-percentage)
[pipe](#method-pipe)
[pipeInto](#method-pipeinto)
[pipeThrough](#method-pipethrough)
[pluck](#method-pluck)
[pop](#method-pop)
[prepend](#method-prepend)
[pull](#method-pull)
[push](#method-push)
[put](#method-put)
[random](#method-random)
[range](#method-range)
[reduce](#method-reduce)
[reduceSpread](#method-reduce-spread)
[reject](#method-reject)
[replace](#method-replace)
[replaceRecursive](#method-replacerecursive)
[reverse](#method-reverse)
[search](#method-search)
[select](#method-select)
[shift](#method-shift)
[shuffle](#method-shuffle)
[skip](#method-skip)
[skipUntil](#method-skipuntil)
[skipWhile](#method-skipwhile)
[slice](#method-slice)
[sliding](#method-sliding)
[sole](#method-sole)
[some](#method-some)
[sort](#method-sort)
[sortBy](#method-sortby)
[sortByDesc](#method-sortbydesc)
[sortDesc](#method-sortdesc)
[sortKeys](#method-sortkeys)
[sortKeysDesc](#method-sortkeysdesc)
[sortKeysUsing](#method-sortkeysusing)
[splice](#method-splice)
[split](#method-split)
[splitIn](#method-splitin)
[sum](#method-sum)
[take](#method-take)
[takeUntil](#method-takeuntil)
[takeWhile](#method-takewhile)
[tap](#method-tap)
[times](#method-times)
[toArray](#method-toarray)
[toJson](#method-tojson)
[toPrettyJson](#method-to-pretty-json)
[transform](#method-transform)
[undot](#method-undot)
[union](#method-union)
[unique](#method-unique)
[uniqueStrict](#method-uniquestrict)
[unless](#method-unless)
[unlessEmpty](#method-unlessempty)
[unlessNotEmpty](#method-unlessnotempty)
[unwrap](#method-unwrap)
[value](#method-value)
[values](#method-values)
[when](#method-when)
[whenEmpty](#method-whenempty)
[whenNotEmpty](#method-whennotempty)
[where](#method-where)
[whereStrict](#method-wherestrict)
[whereBetween](#method-wherebetween)
[whereIn](#method-wherein)
[whereInStrict](#method-whereinstrict)
[whereInstanceOf](#method-whereinstanceof)
[whereNotBetween](#method-wherenotbetween)
[whereNotIn](#method-wherenotin)
[whereNotInStrict](#method-wherenotinstrict)
[whereNotNull](#method-wherenotnull)
[whereNull](#method-wherenull)
[wrap](#method-wrap)
[zip](#method-zip)

</div>

<a name="method-listing"></a>
## 方法列表

<style>
    .collection-method code {
        font-size: 14px;
    }

    .collection-method:not(.first-collection-method) {
        margin-top: 50px;
    }
</style>


<a name="method-after"></a>
#### `after()` {.collection-method .first-collection-method}

`after` 方法回傳指定項目之後的項目。如果找不到指定項目或是該項目為最後一個項目，則回傳 `null`：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->after(3);

// 4

$collection->after(5);

// null
```

此方法使用「寬鬆」比較來搜尋指定項目，這代表包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比較，可以為該方法提供 `strict` 參數：

```php
collect([2, 4, 6, 8])->after('4', strict: true);

// null
```

或者，你也可以提供自己的閉包來搜尋第一個通過指定真相測試 (Truth Test) 的項目：

```php
collect([2, 4, 6, 8])->after(function (int $item, int $key) {
    return $item > 5;
});

// 8
```


<a name="method-all"></a>
#### `all()` {.collection-method}

`all` 方法回傳集合所代表的底層陣列：

```php
collect([1, 2, 3])->all();

// [1, 2, 3]
```


<a name="method-average"></a>
#### `average()` {.collection-method}

[avg](#method-avg) 方法的別名。


<a name="method-avg"></a>
#### `avg()` {.collection-method}

`avg` 方法回傳指定鍵的[平均值](https://en.wikipedia.org/wiki/Average)：

```php
$average = collect([
    ['foo' => 10],
    ['foo' => 10],
    ['foo' => 20],
    ['foo' => 40]
])->avg('foo');

// 20

$average = collect([1, 1, 2, 4])->avg();

// 2
```


<a name="method-before"></a>
#### `before()` {.collection-method}

`before` 方法與 [after](#method-after) 方法相反。它會回傳指定項目之前的項目。如果找不到指定項目或是該項目為第一個項目，則回傳 `null`：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->before(3);

// 2

$collection->before(1);

// null

collect([2, 4, 6, 8])->before('4', strict: true);

// null

collect([2, 4, 6, 8])->before(function (int $item, int $key) {
    return $item > 5;
});

// 4
```


<a name="method-chunk"></a>
#### `chunk()` {.collection-method}

`chunk` 方法將集合分割成多個指定大小的較小集合：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7]);

$chunks = $collection->chunk(4);

$chunks->all();

// [[1, 2, 3, 4], [5, 6, 7]]
```

此方法在 [views](/docs/{{version}}/views) 中搭配網格系統（如 [Bootstrap](https://getbootstrap.com/docs/5.3/layout/grid/)）時特別有用。例如，想像你有一個想要在網格中顯示的 Eloquent 模型集合：

```blade
@foreach ($products->chunk(3) as $chunk)
    <div class="row">
        @foreach ($chunk as $product)
            <div class="col-xs-4">{{ $product->name }}</div>
        @endforeach
    </div>
@endforeach
```


<a name="method-chunkwhile"></a>
#### `chunkWhile()` {.collection-method}

`chunkWhile` 方法根據指定回呼函式的評估結果將集合分割成多個較小的集合。傳遞給閉包的 `$chunk` 變數可用於檢查前一個元素：

```php
$collection = collect(str_split('AABBCCCD'));

$chunks = $collection->chunkWhile(function (string $value, int $key, Collection $chunk) {
    return $value === $chunk->last();
});

$chunks->all();

// [['A', 'A'], ['B', 'B'], ['C', 'C', 'C'], ['D']]
```


<a name="method-collapse"></a>
#### `collapse()` {.collection-method}

`collapse` 方法將陣列集合或集合的集合收摺成單一且扁平的集合：

```php
$collection = collect([
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
]);

$collapsed = $collection->collapse();

$collapsed->all();

// [1, 2, 3, 4, 5, 6, 7, 8, 9]
```


<a name="method-collapsewithkeys"></a>
#### `collapseWithKeys()` {.collection-method}

`collapseWithKeys` 方法將陣列集合或集合的集合扁平化為單一集合，並保留原始的鍵。如果集合已經是扁平的，它將回傳一個空集合：

```php
$collection = collect([
    ['first'  => collect([1, 2, 3])],
    ['second' => [4, 5, 6]],
    ['third'  => collect([7, 8, 9])]
]);

$collapsed = $collection->collapseWithKeys();

$collapsed->all();

// [
//     'first'  => [1, 2, 3],
//     'second' => [4, 5, 6],
//     'third'  => [7, 8, 9],
// ]
```


<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 方法會回傳一個包含目前集合項目的新 `Collection` 實例：

```php
$collectionA = collect([1, 2, 3]);

$collectionB = $collectionA->collect();

$collectionB->all();

// [1, 2, 3]
```

`collect` 方法主要用於將 [Lazy 集合](#lazy-collections) 轉換為標準的 `Collection` 實例：

```php
$lazyCollection = LazyCollection::make(function () {
    yield 1;
    yield 2;
    yield 3;
});

$collection = $lazyCollection->collect();

$collection::class;

// 'Illuminate\Support\Collection'

$collection->all();

// [1, 2, 3]
```

> [!NOTE]
> 當你有一個 `Enumerable` 實例且需要一個非 Lazy 的集合實例時，`collect` 方法特別有用。由於 `collect()` 是 `Enumerable` 契約的一部分，你可以放心地使用它來取得 `Collection` 實例。


<a name="method-combine"></a>
#### `combine()` {.collection-method}

`combine` 方法將集合的值作為鍵，與另一個陣列或集合的值進行組合：

```php
$collection = collect(['name', 'age']);

$combined = $collection->combine(['George', 29]);

$combined->all();

// ['name' => 'George', 'age' => 29]
```


<a name="method-concat"></a>
#### `concat()` {.collection-method}

`concat` 方法將指定陣列或集合的值附加到另一個集合的末尾：

```php
$collection = collect(['John Doe']);

$concatenated = $collection->concat(['Jane Doe'])->concat(['name' => 'Johnny Doe']);

$concatenated->all();

// ['John Doe', 'Jane Doe', 'Johnny Doe']
```

`concat` 方法會為串接到原始集合的項目重新建立數值索引。若要保留關聯集合中的鍵，請參閱 [merge](#method-merge) 方法。


<a name="method-contains"></a>
#### `contains()` {.collection-method}

`contains` 方法用來判斷集合是否包含指定的項目。你可以傳遞一個閉包給 `contains` 方法，以判斷集合中是否存在符合指定真相測試的元素：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->contains(function (int $value, int $key) {
    return $value > 5;
});

// false
```

或者，你也可以傳遞一個字串給 `contains` 方法，以判斷集合是否包含指定的項目值：

```php
$collection = collect(['name' => 'Desk', 'price' => 100]);

$collection->contains('Desk');

// true

$collection->contains('New York');

// false
```

你也可以傳遞一組鍵值對給 `contains` 方法，這將判斷集合中是否存在指定的配對：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
]);

$collection->contains('product', 'Bookcase');

// false
```

`contains` 方法在檢查項目值時使用「寬鬆」比較，這代表包含整數值的字串將被視為與相同值的整數相等。請使用 [containsStrict](#method-containsstrict) 方法來透過「嚴格」比較進行過濾。

關於 `contains` 的反向操作，請參閱 [doesntContain](#method-doesntcontain) 方法。


<a name="method-containsstrict"></a>
#### `containsStrict()` {.collection-method}

此方法的簽署與 [contains](#method-contains) 方法相同；然而，所有的值都會使用「嚴格」比較。

> [!NOTE]
> 當使用 [Eloquent 集合](/docs/{{version}}/eloquent-collections#method-contains) 時，此方法的行為會有所不同。

<a name="method-count"></a>
#### `count()` {.collection-method}

`count` 方法回傳集合中的項目總數：

```php
$collection = collect([1, 2, 3, 4]);

$collection->count();

// 4
```


<a name="method-countBy"></a>
#### `countBy()` {.collection-method}

`countBy` 方法計算集合中各個值出現的次數。根據預設，該方法會計算每個元素出現的次數，這讓您可以統計集合中某些「類型」的元素：

```php
$collection = collect([1, 2, 2, 2, 3]);

$counted = $collection->countBy();

$counted->all();

// [1 => 1, 2 => 3, 3 => 1]
```

您可以向 `countBy` 方法傳遞一個閉包，以根據自定義的值來統計所有項目：

```php
$collection = collect(['alice@gmail.com', 'bob@yahoo.com', 'carlos@gmail.com']);

$counted = $collection->countBy(function (string $email) {
    return substr(strrchr($email, '@'), 1);
});

$counted->all();

// ['gmail.com' => 2, 'yahoo.com' => 1]
```


<a name="method-crossjoin"></a>
#### `crossJoin()` {.collection-method}

`crossJoin` 方法將集合的值與給定的陣列或集合進行交叉連結，並回傳包含所有可能排列組合的笛卡兒積 (Cartesian product)：

```php
$collection = collect([1, 2]);

$matrix = $collection->crossJoin(['a', 'b']);

$matrix->all();

/*
    [
        [1, 'a'],
        [1, 'b'],
        [2, 'a'],
        [2, 'b'],
    ]
*/

$collection = collect([1, 2]);

$matrix = $collection->crossJoin(['a', 'b'], ['I', 'II']);

$matrix->all();

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
```


<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 方法傾印集合的項目並結束指令碼的執行：

```php
$collection = collect(['John Doe', 'Jane Doe']);

$collection->dd();

/*
    array:2 [
        0 => "John Doe"
        1 => "Jane Doe"
    ]
*/
```

如果您不想停止執行指令碼，請改用 [dump](#method-dump) 方法。


<a name="method-diff"></a>
#### `diff()` {.collection-method}

`diff` 方法根據值將集合與另一個集合或純 PHP `array` 進行比較。此方法將回傳原集合中存在但給定集合中不存在的值：

```php
$collection = collect([1, 2, 3, 4, 5]);

$diff = $collection->diff([2, 4, 6, 8]);

$diff->all();

// [1, 3, 5]
```

> [!NOTE]
> 此方法的行為在操作 [Eloquent 集合](/docs/{{version}}/eloquent-collections#method-diff)時會有所不同。


<a name="method-diffassoc"></a>
#### `diffAssoc()` {.collection-method}

`diffAssoc` 方法根據鍵和值將集合與另一個集合或純 PHP `array` 進行比較。此方法將回傳原集合中存在但給定集合中不存在的鍵 / 值對：

```php
$collection = collect([
    'color' => 'orange',
    'type' => 'fruit',
    'remain' => 6,
]);

$diff = $collection->diffAssoc([
    'color' => 'yellow',
    'type' => 'fruit',
    'remain' => 3,
    'used' => 6,
]);

$diff->all();

// ['color' => 'orange', 'remain' => 6]
```


<a name="method-diffassocusing"></a>
#### `diffAssocUsing()` {.collection-method}

與 `diffAssoc` 不同，`diffAssocUsing` 接受一個使用者提供的回呼函數來進行索引的比較：

```php
$collection = collect([
    'color' => 'orange',
    'type' => 'fruit',
    'remain' => 6,
]);

$diff = $collection->diffAssocUsing([
    'Color' => 'yellow',
    'Type' => 'fruit',
    'Remain' => 3,
], 'strnatcasecmp');

$diff->all();

// ['color' => 'orange', 'remain' => 6]
```

該回呼必須是一個比較函數，並回傳小於、等於或大於零的整數。更多資訊請參考 PHP 官方文件中的 [array_diff_uassoc](https://www.php.net/array_diff_uassoc#refsect1-function.array-diff-uassoc-parameters)，這是 `diffAssocUsing` 方法內部使用的 PHP 函數。


<a name="method-diffkeys"></a>
#### `diffKeys()` {.collection-method}

`diffKeys` 方法根據鍵將集合與另一個集合或純 PHP `array` 進行比較。此方法將回傳原集合中存在但給定集合中不存在的鍵 / 值對：

```php
$collection = collect([
    'one' => 10,
    'two' => 20,
    'three' => 30,
    'four' => 40,
    'five' => 50,
]);

$diff = $collection->diffKeys([
    'two' => 2,
    'four' => 4,
    'six' => 6,
    'eight' => 8,
]);

$diff->all();

// ['one' => 10, 'three' => 30, 'five' => 50]
```


<a name="method-doesntcontain"></a>
#### `doesntContain()` {.collection-method}

`doesntContain` 方法判斷集合中是否不包含給定的項目。您可以向 `doesntContain` 方法傳遞一個閉包，以判斷集合中是否不存在符合給定真值測試的元素：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->doesntContain(function (int $value, int $key) {
    return $value < 5;
});

// false
```

或者，您可以向 `doesntContain` 方法傳遞一個字串，以判斷集合中是否不包含給定的項目值：

```php
$collection = collect(['name' => 'Desk', 'price' => 100]);

$collection->doesntContain('Table');

// true

$collection->doesntContain('Desk');

// false
```

您也可以向 `doesntContain` 方法傳遞一個鍵 / 值對，這將判斷給定的配對是否不存在於集合中：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
]);

$collection->doesntContain('product', 'Bookcase');

// true
```

`doesntContain` 方法在檢查項目值時使用「寬鬆 (loose)」比較，這意味著包含整數值的字串將被視為等於相同值的整數。


<a name="method-doesntcontainstrict"></a>
#### `doesntContainStrict()` {.collection-method}

此方法具有與 [doesntContain](#method-doesntcontain) 方法相同的定義；但是，所有值都會使用「嚴格 (strict)」比較。


<a name="method-dot"></a>
#### `dot()` {.collection-method}

`dot` 方法將多維集合扁平化為單層集合，並使用「點 (dot)」標記法來表示深度：

```php
$collection = collect(['products' => ['desk' => ['price' => 100]]]);

$flattened = $collection->dot();

$flattened->all();

// ['products.desk.price' => 100]
```


<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 方法傾印集合的項目：

```php
$collection = collect(['John Doe', 'Jane Doe']);

$collection->dump();

/*
    array:2 [
        0 => "John Doe"
        1 => "Jane Doe"
    ]
*/
```

如果您想在傾印集合後停止執行指令碼，請改用 [dd](#method-dd) 方法。


<a name="method-duplicates"></a>
#### `duplicates()` {.collection-method}

`duplicates` 方法從集合中檢索並回傳重複的值：

```php
$collection = collect(['a', 'b', 'a', 'c', 'b']);

$collection->duplicates();

// [2 => 'a', 4 => 'b']
```

如果集合包含陣列或物件，您可以傳遞想要檢查重複值的屬性鍵名：

```php
$employees = collect([
    ['email' => 'abigail@example.com', 'position' => 'Developer'],
    ['email' => 'james@example.com', 'position' => 'Designer'],
    ['email' => 'victoria@example.com', 'position' => 'Developer'],
]);

$employees->duplicates('position');

// [2 => 'Developer']
```


<a name="method-duplicatesstrict"></a>
#### `duplicatesStrict()` {.collection-method}

此方法具有與 [duplicates](#method-duplicates) 方法相同的定義；但是，所有值都會使用「嚴格 (strict)」比較。

<a name="method-each"></a>
#### `each()` {.collection-method}

`each` 方法會迭代集合中的項目，並將每個項目傳遞給閉包：

```php
$collection = collect([1, 2, 3, 4]);

$collection->each(function (int $item, int $key) {
    // ...
});
```

如果您想停止迭代項目，可以從閉包中回傳 `false`：

```php
$collection->each(function (int $item, int $key) {
    if (/* condition */) {
        return false;
    }
});
```


<a name="method-eachspread"></a>
#### `eachSpread()` {.collection-method}

`eachSpread` 方法會迭代集合的項目，並將每個巢狀項目值傳遞給給定的回呼：

```php
$collection = collect([['John Doe', 35], ['Jane Doe', 33]]);

$collection->eachSpread(function (string $name, int $age) {
    // ...
});
```

您可以透過從回呼中回傳 `false` 來停止迭代項目：

```php
$collection->eachSpread(function (string $name, int $age) {
    return false;
});
```


<a name="method-ensure"></a>
#### `ensure()` {.collection-method}

`ensure` 方法可以用來驗證集合的所有元素是否為給定的型別或型別清單。否則，將會拋出 `UnexpectedValueException`：

```php
return $collection->ensure(User::class);

return $collection->ensure([User::class, Customer::class]);
```

也可以指定原始型別，例如 `string`、`int`、`float`、`bool` 和 `array`：

```php
return $collection->ensure('int');
```

> [!WARNING]
> `ensure` 方法並不保證之後不會有不同型別的元素被加入到集合中。


<a name="method-every"></a>
#### `every()` {.collection-method}

`every` 方法可以用來驗證集合的所有元素是否都通過給定的真值測試：

```php
collect([1, 2, 3, 4])->every(function (int $value, int $key) {
    return $value > 2;
});

// false
```

如果集合是空的，`every` 方法將回傳 true：

```php
$collection = collect([]);

$collection->every(function (int $value, int $key) {
    return $value > 2;
});

// true
```


<a name="method-except"></a>
#### `except()` {.collection-method}

`except` 方法會回傳集合中除了指定鍵以外的所有項目：

```php
$collection = collect(['product_id' => 1, 'price' => 100, 'discount' => false]);

$filtered = $collection->except(['price', 'discount']);

$filtered->all();

// ['product_id' => 1]
```

關於 `except` 的相反方法，請參考 [only](#method-only) 方法。

> [!NOTE]
> 當使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-except) 時，此方法的行為會被修改。


<a name="method-filter"></a>
#### `filter()` {.collection-method}

`filter` 方法使用給定的回呼過濾集合，僅保留那些通過給定真值測試的項目：

```php
$collection = collect([1, 2, 3, 4]);

$filtered = $collection->filter(function (int $value, int $key) {
    return $value > 2;
});

$filtered->all();

// [3, 4]
```

如果沒有提供回呼，集合中所有等同於 `false` 的項目都將被移除：

```php
$collection = collect([1, 2, 3, null, false, '', 0, []]);

$collection->filter()->all();

// [1, 2, 3]
```

關於 `filter` 的相反方法，請參考 [reject](#method-reject) 方法。


<a name="method-first"></a>
#### `first()` {.collection-method}

`first` 方法回傳集合中第一個通過給定真值測試的元素：

```php
collect([1, 2, 3, 4])->first(function (int $value, int $key) {
    return $value > 2;
});

// 3
```

您也可以在不傳遞參數的情況下呼叫 `first` 方法來取得集合中的第一個元素。如果集合是空的，則回傳 `null`：

```php
collect([1, 2, 3, 4])->first();

// 1
```


<a name="method-first-or-fail"></a>
#### `firstOrFail()` {.collection-method}

`firstOrFail` 方法與 `first` 方法相同；然而，如果找不到結果，將會拋出 `Illuminate\Support\ItemNotFoundException` 例外：

```php
collect([1, 2, 3, 4])->firstOrFail(function (int $value, int $key) {
    return $value > 5;
});

// Throws ItemNotFoundException...
```

您也可以在不傳遞參數的情況下呼叫 `firstOrFail` 方法來取得集合中的第一個元素。如果集合是空的，將會拋出 `Illuminate\Support\ItemNotFoundException` 例外：

```php
collect([])->firstOrFail();

// Throws ItemNotFoundException...
```


<a name="method-first-where"></a>
#### `firstWhere()` {.collection-method}

`firstWhere` 方法回傳集合中第一個具有給定鍵值對的元素：

```php
$collection = collect([
    ['name' => 'Regena', 'age' => null],
    ['name' => 'Linda', 'age' => 14],
    ['name' => 'Diego', 'age' => 23],
    ['name' => 'Linda', 'age' => 84],
]);

$collection->firstWhere('name', 'Linda');

// ['name' => 'Linda', 'age' => 14]
```

您也可以使用比較運算子呼叫 `firstWhere` 方法：

```php
$collection->firstWhere('age', '>=', 18);

// ['name' => 'Diego', 'age' => 23]
```

就像 [where](#method-where) 方法一樣，您可以向 `firstWhere` 方法傳遞一個參數。在這種情況下，`firstWhere` 方法將回傳第一個指定項目鍵的值為「真值 (truthy)」的項目：

```php
$collection->firstWhere('age');

// ['name' => 'Linda', 'age' => 14]
```


<a name="method-flatmap"></a>
#### `flatMap()` {.collection-method}

`flatMap` 方法迭代集合並將每個值傳遞給給定的閉包。閉包可以自由地修改項目並將其回傳，從而形成一個新的已修改項目集合。接著，陣列會被打平一級：

```php
$collection = collect([
    ['name' => 'Sally'],
    ['school' => 'Arkansas'],
    ['age' => 28]
]);

$flattened = $collection->flatMap(function (array $values) {
    return array_map('strtoupper', $values);
});

$flattened->all();

// ['name' => 'SALLY', 'school' => 'ARKANSAS', 'age' => '28'];
```


<a name="method-flatten"></a>
#### `flatten()` {.collection-method}

`flatten` 方法將多維集合打平為一維：

```php
$collection = collect([
    'name' => 'Taylor',
    'languages' => [
        'PHP', 'JavaScript'
    ]
]);

$flattened = $collection->flatten();

$flattened->all();

// ['Taylor', 'PHP', 'JavaScript'];
```

如有必要，您可以向 `flatten` 方法傳遞一個「深度 (depth)」參數：

```php
$collection = collect([
    'Apple' => [
        [
            'name' => 'iPhone 6S',
            'brand' => 'Apple'
        ],
    ],
    'Samsung' => [
        [
            'name' => 'Galaxy S7',
            'brand' => 'Samsung'
        ],
    ],
]);

$products = $collection->flatten(1);

$products->values()->all();

/*
    [
        ['name' => 'iPhone 6S', 'brand' => 'Apple'],
        ['name' => 'Galaxy S7', 'brand' => 'Samsung'],
    ]
*/
```

在這個範例中，若在不提供深度的情況下呼叫 `flatten` 也會打平巢狀陣列，結果將會是 `['iPhone 6S', 'Apple', 'Galaxy S7', 'Samsung']`。提供深度允許您指定巢狀陣列將被打平的層級數。


<a name="method-flip"></a>
#### `flip()` {.collection-method}

`flip` 方法將集合的鍵與其對應的值交換：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

$flipped = $collection->flip();

$flipped->all();

// ['Taylor' => 'name', 'Laravel' => 'framework']
```


<a name="method-forget"></a>
#### `forget()` {.collection-method}

`forget` 方法透過鍵從集合中移除一個項目：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

// Forget a single key...
$collection->forget('name');

// ['framework' => 'Laravel']

// Forget multiple keys...
$collection->forget(['name', 'framework']);

// []
```

> [!WARNING]
> 與大多數其他集合方法不同，`forget` 不會回傳一個新的已修改集合；它會直接修改並回傳呼叫它的集合本身。

<a name="method-forpage"></a>
#### `forPage()` {.collection-method}

`forPage` 方法回傳一個包含指定頁碼中項目的新集合。該方法接受頁碼作為第一個參數，並將每頁顯示的項目數量作為第二個參數：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9]);

$chunk = $collection->forPage(2, 3);

$chunk->all();

// [4, 5, 6]
```


<a name="method-fromjson"></a>
#### `fromJson()` {.collection-method}

靜態 `fromJson` 方法透過使用 PHP 的 `json_decode` 函式解碼給定的 JSON 字串來建立一個新的集合實例：

```php
use Illuminate\Support\Collection;

$json = json_encode([
    'name' => 'Taylor Otwell',
    'role' => 'Developer',
    'status' => 'Active',
]);

$collection = Collection::fromJson($json);
```


<a name="method-get"></a>
#### `get()` {.collection-method}

`get` 方法回傳指定鍵名 (Key) 的項目。如果該鍵名不存在，則回傳 `null`：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

$value = $collection->get('name');

// Taylor
```

您可以選擇性地傳遞一個預設值作為第二個參數：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

$value = $collection->get('age', 34);

// 34
```

您甚至可以傳遞一個回呼 (Callback) 作為該方法的預設值。如果指定的鍵名不存在，將回傳該回呼的結果：

```php
$collection->get('email', function () {
    return 'taylor@example.com';
});

// taylor@example.com
```


<a name="method-groupby"></a>
#### `groupBy()` {.collection-method}

`groupBy` 方法根據給定的鍵名對集合項目進行分組：

```php
$collection = collect([
    ['account_id' => 'account-x10', 'product' => 'Chair'],
    ['account_id' => 'account-x10', 'product' => 'Bookcase'],
    ['account_id' => 'account-x11', 'product' => 'Desk'],
]);

$grouped = $collection->groupBy('account_id');

$grouped->all();

/*
    [
        'account-x10' => [
            ['account_id' => 'account-x10', 'product' => 'Chair'],
            ['account_id' => 'account-x10', 'product' => 'Bookcase'],
        ],
        'account-x11' => [
            ['account_id' => 'account-x11', 'product' => 'Desk'],
        ],
    ]
*/
```

除了傳遞字串 `key` 之外，您也可以傳遞一個回呼。該回呼應回傳您希望作為分組鍵名的值：

```php
$grouped = $collection->groupBy(function (array $item, int $key) {
    return substr($item['account_id'], -3);
});

$grouped->all();

/*
    [
        'x10' => [
            ['account_id' => 'account-x10', 'product' => 'Chair'],
            ['account_id' => 'account-x10', 'product' => 'Bookcase'],
        ],
        'x11' => [
            ['account_id' => 'account-x11', 'product' => 'Desk'],
        ],
    ]
*/
```

可以將多個分組標準作為陣列傳遞。每個陣列元素將套用於多維陣列中對應的層級：

```php
$data = new Collection([
    10 => ['user' => 1, 'skill' => 1, 'roles' => ['Role_1', 'Role_3']],
    20 => ['user' => 2, 'skill' => 1, 'roles' => ['Role_1', 'Role_2']],
    30 => ['user' => 3, 'skill' => 2, 'roles' => ['Role_1']],
    40 => ['user' => 4, 'skill' => 2, 'roles' => ['Role_2']],
]);

$result = $data->groupBy(['skill', function (array $item) {
    return $item['roles'];
}], preserveKeys: true);

/*
[
    1 => [
        'Role_1' => [
            10 => ['user' => 1, 'skill' => 1, 'roles' => ['Role_1', 'Role_3']],
            20 => ['user' => 2, 'skill' => 1, 'roles' => ['Role_1', 'Role_2']],
        ],
        'Role_2' => [
            20 => ['user' => 2, 'skill' => 1, 'roles' => ['Role_1', 'Role_2']],
        ],
        'Role_3' => [
            10 => ['user' => 1, 'skill' => 1, 'roles' => ['Role_1', 'Role_3']],
        ],
    ],
    2 => [
        'Role_1' => [
            30 => ['user' => 3, 'skill' => 2, 'roles' => ['Role_1']],
        ],
        'Role_2' => [
            40 => ['user' => 4, 'skill' => 2, 'roles' => ['Role_2']],
        ],
    ],
];
*/
```


<a name="method-has"></a>
#### `has()` {.collection-method}

`has` 方法判斷集合中是否存在給定的鍵名：

```php
$collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

$collection->has('product');

// true

$collection->has(['product', 'amount']);

// true

$collection->has(['amount', 'price']);

// false
```


<a name="method-hasany"></a>
#### `hasAny()` {.collection-method}

`hasAny` 方法判斷集合中是否存在任何給定的鍵名：

```php
$collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

$collection->hasAny(['product', 'price']);

// true

$collection->hasAny(['name', 'price']);

// false
```


<a name="method-hasmany"></a>
#### `hasMany()` {.collection-method}

`hasMany` 方法判斷集合是否包含多個項目：

```php
collect([])->hasMany();

// false

collect(['1'])->hasMany();

// false

collect([1, 2, 3])->hasMany();

// true

collect([
    ['age' => 2],
    ['age' => 3],
])->hasMany(fn ($item) => $item['age'] === 2)

// false
```


<a name="method-hassole"></a>
#### `hasSole()` {.collection-method}

`hasSole` 方法判斷集合是否僅包含單一項目，並可選擇是否符合給定的標準：

```php
collect([])->hasSole();

// false

collect(['1'])->hasSole();

// true

collect([1, 2, 3])->hasSole(fn (int $item) => $item === 2);

// true
```


<a name="method-implode"></a>
#### `implode()` {.collection-method}

`implode` 方法結合集合中的項目。其參數取決於集合中項目的類型。如果集合包含陣列或物件，您應該傳遞您希望結合的屬性鍵名，以及您希望放在值之間的「膠合 (Glue)」字串：

```php
$collection = collect([
    ['account_id' => 1, 'product' => 'Desk'],
    ['account_id' => 2, 'product' => 'Chair'],
]);

$collection->implode('product', ', ');

// 'Desk, Chair'
```

如果集合包含簡單的字串或數值，您應該傳遞「膠合」字串作為該方法的唯一參數：

```php
collect([1, 2, 3, 4, 5])->implode('-');

// '1-2-3-4-5'
```

如果您想格式化要結合的值，可以傳遞一個閉包 (Closure) 給 `implode` 方法：

```php
$collection->implode(function (array $item, int $key) {
    return strtoupper($item['product']);
}, ', ');

// 'DESK, CHAIR'
```


<a name="method-intersect"></a>
#### `intersect()` {.collection-method}

`intersect` 方法從原始集合中移除不在給定陣列或集合中的任何值。產生的集合將保留原始集合的鍵名：

```php
$collection = collect(['Desk', 'Sofa', 'Chair']);

$intersect = $collection->intersect(['Desk', 'Chair', 'Bookcase']);

$intersect->all();

// [0 => 'Desk', 2 => 'Chair']
```

> [!NOTE]
> 當使用 [Eloquent 集合](/docs/{{version}}/eloquent-collections#method-intersect) 時，此方法的行為會有所不同。


<a name="method-intersectusing"></a>
#### `intersectUsing()` {.collection-method}

`intersectUsing` 方法使用自定義回呼來比較值，並從原始集合中移除不在給定陣列或集合中的任何值。產生的集合將保留原始集合的鍵名：

```php
$collection = collect(['Desk', 'Sofa', 'Chair']);

$intersect = $collection->intersectUsing(['desk', 'chair', 'bookcase'], function (string $a, string $b) {
    return strcasecmp($a, $b);
});

$intersect->all();

// [0 => 'Desk', 2 => 'Chair']
```

<a name="method-intersectAssoc"></a>
#### `intersectAssoc()` {.collection-method}

`intersectAssoc` 方法會比較原始集合與另一個集合或陣列，回傳存在於所有指定集合中的鍵值對 (Key / Value Pairs)：

```php
$collection = collect([
    'color' => 'red',
    'size' => 'M',
    'material' => 'cotton'
]);

$intersect = $collection->intersectAssoc([
    'color' => 'blue',
    'size' => 'M',
    'material' => 'polyester'
]);

$intersect->all();

// ['size' => 'M']
```


<a name="method-intersectassocusing"></a>
#### `intersectAssocUsing()` {.collection-method}

`intersectAssocUsing` 方法會比較原始集合與另一個集合或陣列，回傳兩者皆有的鍵值對，並使用自定義的比較回呼函式 (Callback) 來判定鍵與值是否相等：

```php
$collection = collect([
    'color' => 'red',
    'Size' => 'M',
    'material' => 'cotton',
]);

$intersect = $collection->intersectAssocUsing([
    'color' => 'blue',
    'size' => 'M',
    'material' => 'polyester',
], function (string $a, string $b) {
    return strcasecmp($a, $b);
});

$intersect->all();

// ['Size' => 'M']
```


<a name="method-intersectbykeys"></a>
#### `intersectByKeys()` {.collection-method}

`intersectByKeys` 方法會從原始集合中移除任何不存在於指定陣列或集合中的鍵及其對應的值：

```php
$collection = collect([
    'serial' => 'UX301', 'type' => 'screen', 'year' => 2009,
]);

$intersect = $collection->intersectByKeys([
    'reference' => 'UX404', 'type' => 'tab', 'year' => 2011,
]);

$intersect->all();

// ['type' => 'screen', 'year' => 2009]
```


<a name="method-isempty"></a>
#### `isEmpty()` {.collection-method}

如果集合為空，`isEmpty` 方法會回傳 `true`；否則回傳 `false`：

```php
collect([])->isEmpty();

// true
```


<a name="method-isnotempty"></a>
#### `isNotEmpty()` {.collection-method}

如果集合不為空，`isNotEmpty` 方法會回傳 `true`；否則回傳 `false`：

```php
collect([])->isNotEmpty();

// false
```


<a name="method-join"></a>
#### `join()` {.collection-method}

`join` 方法會將集合的值連接為字串。使用此方法的第二個參數，您還可以指定如何將最後一個元素附加到字串中：

```php
collect(['a', 'b', 'c'])->join(', '); // 'a, b, c'
collect(['a', 'b', 'c'])->join(', ', ', and '); // 'a, b, and c'
collect(['a', 'b'])->join(', ', ' and '); // 'a and b'
collect(['a'])->join(', ', ' and '); // 'a'
collect([])->join(', ', ' and '); // ''
```


<a name="method-keyby"></a>
#### `keyBy()` {.collection-method}

`keyBy` 方法以指定的鍵作為集合的鍵。如果多個項目具有相同的鍵，則只有最後一個項目會出現在新集合中：

```php
$collection = collect([
    ['product_id' => 'prod-100', 'name' => 'Desk'],
    ['product_id' => 'prod-200', 'name' => 'Chair'],
]);

$keyed = $collection->keyBy('product_id');

$keyed->all();

/*
    [
        'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
        'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]
*/
```

您也可以傳遞一個回呼函式給此方法。該回呼函式應回傳作為集合鍵的值：

```php
$keyed = $collection->keyBy(function (array $item, int $key) {
    return strtoupper($item['product_id']);
});

$keyed->all();

/*
    [
        'PROD-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
        'PROD-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]
*/
```


<a name="method-keys"></a>
#### `keys()` {.collection-method}

`keys` 方法回傳集合所有的鍵：

```php
$collection = collect([
    'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
    'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
]);

$keys = $collection->keys();

$keys->all();

// ['prod-100', 'prod-200']
```


<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 方法回傳集合中通過指定真值測試 (Truth Test) 的最後一個元素：

```php
collect([1, 2, 3, 4])->last(function (int $value, int $key) {
    return $value < 3;
});

// 2
```

您也可以在不帶參數的情況下呼叫 `last` 方法，以取得集合中的最後一個元素。如果集合為空，則回傳 `null`：

```php
collect([1, 2, 3, 4])->last();

// 4
```


<a name="method-lazy"></a>
#### `lazy()` {.collection-method}

`lazy` 方法從底層項目陣列回傳一個新的 [LazyCollection](#lazy-collections) 實例：

```php
$lazyCollection = collect([1, 2, 3, 4])->lazy();

$lazyCollection::class;

// Illuminate\Support\LazyCollection

$lazyCollection->all();

// [1, 2, 3, 4]
```

當您需要對包含許多項目的龐大 `Collection` 進行轉換時，這特別有用：

```php
$count = $hugeCollection
    ->lazy()
    ->where('country', 'FR')
    ->where('balance', '>', '100')
    ->count();
```

透過將集合轉換為 `LazyCollection`，我們可以避免分配大量的額外記憶體。雖然原始集合仍將其值保留在記憶體中，但後續的篩選將不會。因此，在篩選集合結果時，幾乎不會分配額外的記憶體。


<a name="method-macro"></a>
#### `macro()` {.collection-method}

靜態 `macro` 方法允許您在執行期間為 `Collection` 類別增加方法。請參閱[擴充集合](#extending-collections)的說明文件以取得更多資訊。


<a name="method-make"></a>
#### `make()` {.collection-method}

靜態 `make` 方法會建立一個新的集合實例。請參閱[建立集合](#creating-collections)章節。

```php
use Illuminate\Support\Collection;

$collection = Collection::make([1, 2, 3]);
```


<a name="method-map"></a>
#### `map()` {.collection-method}

`map` 方法會遍歷集合，並將每個值傳遞給指定的回呼函式。回呼函式可以自由地修改項目並將其回傳，從而形成一個由修改後的項目組成的新集合：

```php
$collection = collect([1, 2, 3, 4, 5]);

$multiplied = $collection->map(function (int $item, int $key) {
    return $item * 2;
});

$multiplied->all();

// [2, 4, 6, 8, 10]
```

> [!WARNING]
> 與大多數其他集合方法不同，`map` 會回傳一個新的集合實例；它不會修改被呼叫的原始集合。如果您想轉換原始集合，請使用 [transform](#method-transform) 方法。


<a name="method-mapinto"></a>
#### `mapInto()` {.collection-method}

`mapInto()` 方法會遍歷集合，透過將值傳遞給建構函式來建立指定類別的新實例：

```php
class Currency
{
    /**
     * Create a new currency instance.
     */
    function __construct(
        public string $code,
    ) {}
}

$collection = collect(['USD', 'EUR', 'GBP']);

$currencies = $collection->mapInto(Currency::class);

$currencies->all();

// [Currency('USD'), Currency('EUR'), Currency('GBP')]
```


<a name="method-mapspread"></a>
#### `mapSpread()` {.collection-method}

`mapSpread` 方法會遍歷集合項目，並將每個巢狀項目的值傳遞給給定的閉包 (Closure)。閉包可以自由地修改項目並將其回傳，從而形成一個由修改後的項目組成的新集合：

```php
$collection = collect([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

$chunks = $collection->chunk(2);

$sequence = $chunks->mapSpread(function (int $even, int $odd) {
    return $even + $odd;
});

$sequence->all();

// [1, 5, 9, 13, 17]
```

<a name="method-maptogroups"></a>
#### `mapToGroups()` {.collection-method}

`mapToGroups` 方法透過給定的閉包對集合項目進行分組。該閉包應回傳一個包含單一鍵值對 (key / value pair) 的關聯陣列，從而形成一個分組後的新集合：

```php
$collection = collect([
    [
        'name' => 'John Doe',
        'department' => 'Sales',
    ],
    [
        'name' => 'Jane Doe',
        'department' => 'Sales',
    ],
    [
        'name' => 'Johnny Doe',
        'department' => 'Marketing',
    ]
]);

$grouped = $collection->mapToGroups(function (array $item, int $key) {
    return [$item['department'] => $item['name']];
});

$grouped->all();

/*
    [
        'Sales' => ['John Doe', 'Jane Doe'],
        'Marketing' => ['Johnny Doe'],
    ]
*/

$grouped->get('Sales')->all();

// ['John Doe', 'Jane Doe']
```


<a name="method-mapwithkeys"></a>
#### `mapWithKeys()` {.collection-method}

`mapWithKeys` 方法遍歷集合並將每個值傳遞給給定的回呼函式。回呼函式應回傳一個包含單一鍵值對的關聯陣列：

```php
$collection = collect([
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
]);

$keyed = $collection->mapWithKeys(function (array $item, int $key) {
    return [$item['email'] => $item['name']];
});

$keyed->all();

/*
    [
        'john@example.com' => 'John',
        'jane@example.com' => 'Jane',
    ]
*/
```


<a name="method-max"></a>
#### `max()` {.collection-method}

`max` 方法回傳給定鍵的最大值：

```php
$max = collect([
    ['foo' => 10],
    ['foo' => 20]
])->max('foo');

// 20

$max = collect([1, 2, 3, 4, 5])->max();

// 5
```


<a name="method-median"></a>
#### `median()` {.collection-method}

`median` 方法回傳指定鍵的[中位數 (median value)](https://en.wikipedia.org/wiki/Median)：

```php
$median = collect([
    ['foo' => 10],
    ['foo' => 10],
    ['foo' => 20],
    ['foo' => 40]
])->median('foo');

// 15

$median = collect([1, 1, 2, 4])->median();

// 1.5
```


<a name="method-merge"></a>
#### `merge()` {.collection-method}

`merge` 方法將給定的陣列或集合與原始集合合併。如果給定項目中的字串鍵與原始集合中的字串鍵相符，則給定項目的值將覆蓋原始集合中的值：

```php
$collection = collect(['product_id' => 1, 'price' => 100]);

$merged = $collection->merge(['price' => 200, 'discount' => false]);

$merged->all();

// ['product_id' => 1, 'price' => 200, 'discount' => false]
```

如果給定項目的鍵是數字，則值將附加到集合的末尾：

```php
$collection = collect(['Desk', 'Chair']);

$merged = $collection->merge(['Bookcase', 'Door']);

$merged->all();

// ['Desk', 'Chair', 'Bookcase', 'Door']
```


<a name="method-mergerecursive"></a>
#### `mergeRecursive()` {.collection-method}

`mergeRecursive` 方法遞迴地將給定的陣列或集合與原始集合合併。如果給定項目中的字串鍵與原始集合中的字串鍵相符，則這些鍵的值將合併到一個陣列中，且這是遞迴進行的：

```php
$collection = collect(['product_id' => 1, 'price' => 100]);

$merged = $collection->mergeRecursive([
    'product_id' => 2,
    'price' => 200,
    'discount' => false
]);

$merged->all();

// ['product_id' => [1, 2], 'price' => [100, 200], 'discount' => false]
```


<a name="method-min"></a>
#### `min()` {.collection-method}

`min` 方法回傳給定鍵的最小值：

```php
$min = collect([
    ['foo' => 10],
    ['foo' => 20]
])->min('foo');

// 10

$min = collect([1, 2, 3, 4, 5])->min();

// 1
```


<a name="method-mode"></a>
#### `mode()` {.collection-method}

`mode` 方法回傳指定鍵的[眾數 (mode value)](https://en.wikipedia.org/wiki/Mode_(statistics))：

```php
$mode = collect([
    ['foo' => 10],
    ['foo' => 10],
    ['foo' => 20],
    ['foo' => 40]
])->mode('foo');

// [10]

$mode = collect([1, 1, 2, 4])->mode();

// [1]

$mode = collect([1, 1, 2, 2])->mode();

// [1, 2]
```


<a name="method-multiply"></a>
#### `multiply()` {.collection-method}

`multiply` 方法建立集合中所有項目的指定數量副本：

```php
$users = collect([
    ['name' => 'User #1', 'email' => 'user1@example.com'],
    ['name' => 'User #2', 'email' => 'user2@example.com'],
])->multiply(3);

/*
    [
        ['name' => 'User #1', 'email' => 'user1@example.com'],
        ['name' => 'User #2', 'email' => 'user2@example.com'],
        ['name' => 'User #1', 'email' => 'user1@example.com'],
        ['name' => 'User #2', 'email' => 'user2@example.com'],
        ['name' => 'User #1', 'email' => 'user1@example.com'],
        ['name' => 'User #2', 'email' => 'user2@example.com'],
    ]
*/
```


<a name="method-nth"></a>
#### `nth()` {.collection-method}

`nth` 方法建立一個由每隔 n 個元素組成的新集合：

```php
$collection = collect(['a', 'b', 'c', 'd', 'e', 'f']);

$collection->nth(4);

// ['a', 'e']
```

您可以選擇性地傳遞一個起始偏移量作為第二個參數：

```php
$collection->nth(4, 1);

// ['b', 'f']
```


<a name="method-only"></a>
#### `only()` {.collection-method}

`only` 方法回傳集合中具有指定鍵的項目：

```php
$collection = collect([
    'product_id' => 1,
    'name' => 'Desk',
    'price' => 100,
    'discount' => false
]);

$filtered = $collection->only(['product_id', 'name']);

$filtered->all();

// ['product_id' => 1, 'name' => 'Desk']
```

有關 `only` 的反向操作，請參閱 [except](#method-except) 方法。

> [!NOTE]
> 當使用 [Eloquent 集合](/docs/{{version}}/eloquent-collections#method-only)時，此方法的行為會有所不同。


<a name="method-pad"></a>
#### `pad()` {.collection-method}

`pad` 方法將使用給定的值填充陣列，直到陣列達到指定的大小。此方法的行為類似於 PHP 的 [array_pad](https://secure.php.net/manual/en/function.array-pad.php) 函式。

若要向左填充，您應該指定負值的大小。如果給定大小的絕對值小於或等於陣列的長度，則不會進行填充：

```php
$collection = collect(['A', 'B', 'C']);

$filtered = $collection->pad(5, 0);

$filtered->all();

// ['A', 'B', 'C', 0, 0]

$filtered = $collection->pad(-5, 0);

$filtered->all();

// [0, 0, 'A', 'B', 'C']
```


<a name="method-partition"></a>
#### `partition()` {.collection-method}

`partition` 方法可以與 PHP 的陣列解構 (array destructuring) 結合使用，將通過給定真值測試 (truth test) 的元素與未通過的元素分開：

```php
$collection = collect([1, 2, 3, 4, 5, 6]);

[$underThree, $equalOrAboveThree] = $collection->partition(function (int $i) {
    return $i < 3;
});

$underThree->all();

// [1, 2]

$equalOrAboveThree->all();

// [3, 4, 5, 6]
```

> [!NOTE]
> 當與 [Eloquent 集合](/docs/{{version}}/eloquent-collections#method-partition)互動時，此方法的行為會有所不同。


<a name="method-percentage"></a>
#### `percentage()` {.collection-method}

`percentage` 方法可用於快速確定集合中通過給定真值測試的項目百分比：

```php
$collection = collect([1, 1, 2, 2, 2, 3]);

$percentage = $collection->percentage(fn (int $value) => $value === 1);

// 33.33
```

預設情況下，百分比將四捨五入到小數點後兩位。但是，您可以透過為該方法提供第二個參數來自訂此行為：

```php
$percentage = $collection->percentage(fn (int $value) => $value === 1, precision: 3);

// 33.333
```

<a name="method-pipe"></a>
#### `pipe()` {.collection-method}

`pipe` 方法將集合傳遞給給定的閉包，並回傳該閉包執行的結果：

```php
$collection = collect([1, 2, 3]);

$piped = $collection->pipe(function (Collection $collection) {
    return $collection->sum();
});

// 6
```


<a name="method-pipeinto"></a>
#### `pipeInto()` {.collection-method}

`pipeInto` 方法會建立給定類別的新實例，並將集合傳入建構函式：

```php
class ResourceCollection
{
    /**
     * Create a new ResourceCollection instance.
     */
    public function __construct(
        public Collection $collection,
    ) {}
}

$collection = collect([1, 2, 3]);

$resource = $collection->pipeInto(ResourceCollection::class);

$resource->collection->all();

// [1, 2, 3]
```


<a name="method-pipethrough"></a>
#### `pipeThrough()` {.collection-method}

`pipeThrough` 方法將集合傳遞給給定的閉包陣列，並回傳執行這些閉包後的結果：

```php
use Illuminate\Support\Collection;

$collection = collect([1, 2, 3]);

$result = $collection->pipeThrough([
    function (Collection $collection) {
        return $collection->merge([4, 5]);
    },
    function (Collection $collection) {
        return $collection->sum();
    },
]);

// 15
```


<a name="method-pluck"></a>
#### `pluck()` {.collection-method}

`pluck` 方法會獲取給定鍵的所有值：

```php
$collection = collect([
    ['product_id' => 'prod-100', 'name' => 'Desk'],
    ['product_id' => 'prod-200', 'name' => 'Chair'],
]);

$plucked = $collection->pluck('name');

$plucked->all();

// ['Desk', 'Chair']
```

您也可以指定您希望如何設定結果集合的鍵：

```php
$plucked = $collection->pluck('name', 'product_id');

$plucked->all();

// ['prod-100' => 'Desk', 'prod-200' => 'Chair']
```

`pluck` 方法也支援使用「點」記法 (Dot Notation) 獲取巢狀值：

```php
$collection = collect([
    [
        'name' => 'Laracon',
        'speakers' => [
            'first_day' => ['Rosa', 'Judith'],
        ],
    ],
    [
        'name' => 'VueConf',
        'speakers' => [
            'first_day' => ['Abigail', 'Joey'],
        ],
    ],
]);

$plucked = $collection->pluck('speakers.first_day');

$plucked->all();

// [['Rosa', 'Judith'], ['Abigail', 'Joey']]
```

如果存在重複的鍵，最後一個相符的元素將被插入到取得的集合中：

```php
$collection = collect([
    ['brand' => 'Tesla',  'color' => 'red'],
    ['brand' => 'Pagani', 'color' => 'white'],
    ['brand' => 'Tesla',  'color' => 'black'],
    ['brand' => 'Pagani', 'color' => 'orange'],
]);

$plucked = $collection->pluck('color', 'brand');

$plucked->all();

// ['Tesla' => 'black', 'Pagani' => 'orange']
```


<a name="method-pop"></a>
#### `pop()` {.collection-method}

`pop` 方法會移除並回傳集合中的最後一個項目。如果集合為空，則回傳 `null`：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->pop();

// 5

$collection->all();

// [1, 2, 3, 4]
```

您可以將一個整數傳遞給 `pop` 方法，以從集合末尾移除並回傳多個項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->pop(3);

// collect([5, 4, 3])

$collection->all();

// [1, 2]
```


<a name="method-prepend"></a>
#### `prepend()` {.collection-method}

`prepend` 方法將一個項目添加到集合的開頭：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->prepend(0);

$collection->all();

// [0, 1, 2, 3, 4, 5]
```

您也可以傳遞第二個參數來指定被添加項目的鍵：

```php
$collection = collect(['one' => 1, 'two' => 2]);

$collection->prepend(0, 'zero');

$collection->all();

// ['zero' => 0, 'one' => 1, 'two' => 2]
```


<a name="method-pull"></a>
#### `pull()` {.collection-method}

`pull` 方法根據鍵從集合中移除並回傳該項目：

```php
$collection = collect(['product_id' => 'prod-100', 'name' => 'Desk']);

$collection->pull('name');

// 'Desk'

$collection->all();

// ['product_id' => 'prod-100']
```


<a name="method-push"></a>
#### `push()` {.collection-method}

`push` 方法將一個項目附加到集合的末尾：

```php
$collection = collect([1, 2, 3, 4]);

$collection->push(5);

$collection->all();

// [1, 2, 3, 4, 5]
```

您也可以提供多個項目附加到集合的末尾：

```php
$collection = collect([1, 2, 3, 4]);

$collection->push(5, 6, 7);
 
$collection->all();
 
// [1, 2, 3, 4, 5, 6, 7]
```


<a name="method-put"></a>
#### `put()` {.collection-method}

`put` 方法在集合中設定給定的鍵與值：

```php
$collection = collect(['product_id' => 1, 'name' => 'Desk']);

$collection->put('price', 100);

$collection->all();

// ['product_id' => 1, 'name' => 'Desk', 'price' => 100]
```


<a name="method-random"></a>
#### `random()` {.collection-method}

`random` 方法從集合中回傳一個隨機項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->random();

// 4 - (retrieved randomly)
```

您可以傳遞一個整數給 `random`，以指定您想要隨機獲取多少個項目。當明確傳遞您希望收到的項目數量時，總是會回傳一個項目集合：

```php
$random = $collection->random(3);

$random->all();

// [2, 4, 5] - (retrieved randomly)
```

如果集合實例的項目少於要求的數量，`random` 方法將會拋出 `InvalidArgumentException`。

`random` 方法也接受一個閉包，該閉包將接收當前的集合實例：

```php
use Illuminate\Support\Collection;

$random = $collection->random(fn (Collection $items) => min(10, count($items)));

$random->all();

// [1, 2, 3, 4, 5] - (retrieved randomly)
```


<a name="method-range"></a>
#### `range()` {.collection-method}

`range` 方法回傳一個包含指定範圍內整數的集合：

```php
$collection = collect()->range(3, 6);

$collection->all();

// [3, 4, 5, 6]
```


<a name="method-reduce"></a>
#### `reduce()` {.collection-method}

`reduce` 方法將集合歸納為單一值，並將每次迭代的結果傳遞給下一次迭代：

```php
$collection = collect([1, 2, 3]);

$total = $collection->reduce(function (?int $carry, int $item) {
    return $carry + $item;
});

// 6
```

第一次迭代時 `$carry` 的值為 `null`；但是，您可以透過向 `reduce` 傳遞第二個參數來指定其初始值：

```php
$collection->reduce(function (int $carry, int $item) {
    return $carry + $item;
}, 4);

// 10
```

`reduce` 方法也會將陣列的鍵傳遞給給定的閉包：

```php
$collection = collect([
    'usd' => 1400,
    'gbp' => 1200,
    'eur' => 1000,
]);

$ratio = [
    'usd' => 1,
    'gbp' => 1.37,
    'eur' => 1.22,
];

$collection->reduce(function (int $carry, int $value, string $key) use ($ratio) {
    return $carry + ($value * $ratio[$key]);
}, 0);

// 4264
```


<a name="method-reduce-spread"></a>
#### `reduceSpread()` {.collection-method}

`reduceSpread` 方法將集合歸納為一個值的陣列，並將每次迭代的結果傳遞給下一次迭代。此方法與 `reduce` 方法類似；但它可以接受多個初始值：

```php
[$creditsRemaining, $batch] = Image::where('status', 'unprocessed')
    ->get()
    ->reduceSpread(function (int $creditsRemaining, Collection $batch, Image $image) {
        if ($creditsRemaining >= $image->creditsRequired()) {
            $batch->push($image);

            $creditsRemaining -= $image->creditsRequired();
        }

        return [$creditsRemaining, $batch];
    }, $creditsAvailable, collect());
```

<a name="method-reject"></a>
#### `reject()` {.collection-method}

`reject` 方法使用給定的閉包過濾集合。如果該項目應該從結果集合中移除，則閉包應回傳 `true`：

```php
$collection = collect([1, 2, 3, 4]);

$filtered = $collection->reject(function (int $value, int $key) {
    return $value > 2;
});

$filtered->all();

// [1, 2]
```

關於 `reject` 方法的反向操作，請參閱 [filter](#method-filter) 方法。


<a name="method-replace"></a>
#### `replace()` {.collection-method}

`replace` 方法的行為與 `merge` 類似；然而，除了覆寫具有相符字串鍵的項目外，`replace` 方法還會覆寫集合中具有相符數值鍵的項目：

```php
$collection = collect(['Taylor', 'Abigail', 'James']);

$replaced = $collection->replace([1 => 'Victoria', 3 => 'Finn']);

$replaced->all();

// ['Taylor', 'Victoria', 'James', 'Finn']
```


<a name="method-replacerecursive"></a>
#### `replaceRecursive()` {.collection-method}

`replaceRecursive` 方法的行為與 `replace` 類似，但它會遞迴進入陣列，並對內部的值應用相同的替換過程：

```php
$collection = collect([
    'Taylor',
    'Abigail',
    [
        'James',
        'Victoria',
        'Finn'
    ]
]);

$replaced = $collection->replaceRecursive([
    'Charlie',
    2 => [1 => 'King']
]);

$replaced->all();

// ['Charlie', 'Abigail', ['James', 'King', 'Finn']]
```


<a name="method-reverse"></a>
#### `reverse()` {.collection-method}

`reverse` 方法反轉集合項目的順序，並保留原始的鍵：

```php
$collection = collect(['a', 'b', 'c', 'd', 'e']);

$reversed = $collection->reverse();

$reversed->all();

/*
    [
        4 => 'e',
        3 => 'd',
        2 => 'c',
        1 => 'b',
        0 => 'a',
    ]
*/
```


<a name="method-search"></a>
#### `search()` {.collection-method}

`search` 方法在集合中搜尋給定的值，如果找到則回傳其鍵。如果找不到該項目，則回傳 `false`：

```php
$collection = collect([2, 4, 6, 8]);

$collection->search(4);

// 1
```

搜尋是使用「寬鬆」比較進行的，這意味著包含整數值的字串將被視為等於相同值的整數。要使用「嚴格」比較，可以將 `true` 作為該方法的第二個參數傳遞：

```php
collect([2, 4, 6, 8])->search('4', strict: true);

// false
```

或者，您可以提供自己的閉包來搜尋第一個通過給定真值測試的項目：

```php
collect([2, 4, 6, 8])->search(function (int $item, int $key) {
    return $item > 5;
});

// 2
```


<a name="method-select"></a>
#### `select()` {.collection-method}

`select` 方法從集合中選取給定的鍵，類似於 SQL 的 `SELECT` 語法：

```php
$users = collect([
    ['name' => 'Taylor Otwell', 'role' => 'Developer', 'status' => 'active'],
    ['name' => 'Victoria Faith', 'role' => 'Researcher', 'status' => 'active'],
]);

$users->select(['name', 'role']);

/*
    [
        ['name' => 'Taylor Otwell', 'role' => 'Developer'],
        ['name' => 'Victoria Faith', 'role' => 'Researcher'],
    ],
*/
```


<a name="method-shift"></a>
#### `shift()` {.collection-method}

`shift` 方法移除並回傳集合中的第一個項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->shift();

// 1

$collection->all();

// [2, 3, 4, 5]
```

您可以向 `shift` 方法傳遞一個整數，以從集合的開頭移除並回傳多個項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->shift(3);

// collect([1, 2, 3])

$collection->all();

// [4, 5]
```


<a name="method-shuffle"></a>
#### `shuffle()` {.collection-method}

`shuffle` 方法隨機洗牌集合中的項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$shuffled = $collection->shuffle();

$shuffled->all();

// [3, 2, 5, 1, 4] - (generated randomly)
```


<a name="method-skip"></a>
#### `skip()` {.collection-method}

`skip` 方法回傳一個新集合，並從集合開頭移除給定數量的元素：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

$collection = $collection->skip(4);

$collection->all();

// [5, 6, 7, 8, 9, 10]
```


<a name="method-skipuntil"></a>
#### `skipUntil()` {.collection-method}

`skipUntil` 方法會跳過集合中的項目，直到給定的回呼回傳 `false` 為止。一旦回呼回傳 `true`，集合中剩餘的所有項目將作為一個新集合回傳：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->skipUntil(function (int $item) {
    return $item >= 3;
});

$subset->all();

// [3, 4]
```

您也可以向 `skipUntil` 方法傳遞一個簡單的值，以跳過所有項目直到找到給定的值為止：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->skipUntil(3);

$subset->all();

// [3, 4]
```

> [!WARNING]
> 如果找不到給定的值，或者回呼從未回傳 `true`，則 `skipUntil` 方法將回傳一個空集合。


<a name="method-skipwhile"></a>
#### `skipWhile()` {.collection-method}

`skipWhile` 方法會跳過集合中的項目，只要給定的回呼回傳 `true`。一旦回呼回傳 `false`，集合中剩餘的所有項目將作為一個新集合回傳：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->skipWhile(function (int $item) {
    return $item <= 3;
});

$subset->all();

// [4]
```

> [!WARNING]
> 如果回呼從未回傳 `false`，則 `skipWhile` 方法將回傳一個空集合。


<a name="method-slice"></a>
#### `slice()` {.collection-method}

`slice` 方法回傳集合中從給定索引開始的切片：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

$slice = $collection->slice(4);

$slice->all();

// [5, 6, 7, 8, 9, 10]
```

如果您想限制回傳切片的大小，請將所需的大小作為第二個參數傳遞給該方法：

```php
$slice = $collection->slice(4, 2);

$slice->all();

// [5, 6]
```

回傳的切片預設會保留鍵。如果您不想保留原始的鍵，可以使用 [values](#method-values) 方法重新索引它們。


<a name="method-sliding"></a>
#### `sliding()` {.collection-method}

`sliding` 方法回傳一個由區塊組成的新集合，代表集合中項目的「滑動視窗 (sliding window)」視圖：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunks = $collection->sliding(2);

$chunks->toArray();

// [[1, 2], [2, 3], [3, 4], [4, 5]]
```

這在與 [eachSpread](#method-eachspread) 方法結合使用時特別有用：

```php
$transactions->sliding(2)->eachSpread(function (Collection $previous, Collection $current) {
    $current->total = $previous->total + $current->amount;
});
```

您可以選擇性地傳遞第二個「步長 (step)」值，這決定了每個區塊的第一個項目之間的距離：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunks = $collection->sliding(3, step: 2);

$chunks->toArray();

// [[1, 2, 3], [3, 4, 5]]
```

<a name="method-sole"></a>
#### `sole()` {.collection-method}

`sole` 方法返回集合中通過給定真值測試的第一個元素，但前提是該真值測試僅匹配到一個元素：

```php
collect([1, 2, 3, 4])->sole(function (int $value, int $key) {
    return $value === 2;
});

// 2
```

您也可以將鍵值對傳遞給 `sole` 方法，這將返回集合中符合該鍵值對的第一個元素，但前提是必須剛好只有一個元素符合：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
]);

$collection->sole('product', 'Chair');

// ['product' => 'Chair', 'price' => 100]
```

或者，如果集合中只有一個元素，您也可以在不帶參數的情況下呼叫 `sole` 方法來取得集合中的第一個元素：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
]);

$collection->sole();

// ['product' => 'Desk', 'price' => 200]
```

如果集合中沒有應由 `sole` 方法返回的元素，則會拋出 `\Illuminate\Collections\ItemNotFoundException` 異常。如果找到多個應返回的元素，則會拋出 `\Illuminate\Collections\MultipleItemsFoundException`。


<a name="method-some"></a>
#### `some()` {.collection-method}

[contains](#method-contains) 方法的別名。


<a name="method-sort"></a>
#### `sort()` {.collection-method}

`sort` 方法對集合進行排序。排序後的集合會保留原始的陣列鍵名，因此在下面的範例中，我們將使用 [values](#method-values) 方法將鍵名重置為連續編號的索引：

```php
$collection = collect([5, 3, 1, 2, 4]);

$sorted = $collection->sort();

$sorted->values()->all();

// [1, 2, 3, 4, 5]
```

如果您有更進階的排序需求，可以向 `sort` 傳遞一個帶有自定義演算法的回呼。請參閱有關 [uasort](https://secure.php.net/manual/en/function.uasort.php#refsect1-function.uasort-parameters) 的 PHP 文件，這就是集合的 `sort` 方法內部所使用的函式。

> [!NOTE]
> 如果您需要對巢狀陣列或物件的集合進行排序，請參閱 [sortBy](#method-sortby) 和 [sortByDesc](#method-sortbydesc) 方法。


<a name="method-sortby"></a>
#### `sortBy()` {.collection-method}

`sortBy` 方法根據給定的鍵對集合進行排序。排序後的集合會保留原始的陣列鍵名，因此在下面的範例中，我們將使用 [values](#method-values) 方法將鍵名重置為連續編號的索引：

```php
$collection = collect([
    ['name' => 'Desk', 'price' => 200],
    ['name' => 'Chair', 'price' => 100],
    ['name' => 'Bookcase', 'price' => 150],
]);

$sorted = $collection->sortBy('price');

$sorted->values()->all();

/*
    [
        ['name' => 'Chair', 'price' => 100],
        ['name' => 'Bookcase', 'price' => 150],
        ['name' => 'Desk', 'price' => 200],
    ]
*/
```

`sortBy` 方法接受 [排序標誌 (sort flags)](https://www.php.net/manual/en/function.sort.php) 作為其第二個參數：

```php
$collection = collect([
    ['title' => 'Item 1'],
    ['title' => 'Item 12'],
    ['title' => 'Item 3'],
]);

$sorted = $collection->sortBy('title', SORT_NATURAL);

$sorted->values()->all();

/*
    [
        ['title' => 'Item 1'],
        ['title' => 'Item 3'],
        ['title' => 'Item 12'],
    ]
*/
```

或者，您可以傳遞自己的閉包來決定如何對集合的值進行排序：

```php
$collection = collect([
    ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
    ['name' => 'Chair', 'colors' => ['Black']],
    ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
]);

$sorted = $collection->sortBy(function (array $product, int $key) {
    return count($product['colors']);
});

$sorted->values()->all();

/*
    [
        ['name' => 'Chair', 'colors' => ['Black']],
        ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
        ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
    ]
*/
```

如果您想根據多個屬性對集合進行排序，可以向 `sortBy` 方法傳遞一個排序操作陣列。每個排序操作都應該是一個陣列，由您希望排序的屬性以及排序的方向組成：

```php
$collection = collect([
    ['name' => 'Taylor Otwell', 'age' => 34],
    ['name' => 'Abigail Otwell', 'age' => 30],
    ['name' => 'Taylor Otwell', 'age' => 36],
    ['name' => 'Abigail Otwell', 'age' => 32],
]);

$sorted = $collection->sortBy([
    ['name', 'asc'],
    ['age', 'desc'],
]);

$sorted->values()->all();

/*
    [
        ['name' => 'Abigail Otwell', 'age' => 32],
        ['name' => 'Abigail Otwell', 'age' => 30],
        ['name' => 'Taylor Otwell', 'age' => 36],
        ['name' => 'Taylor Otwell', 'age' => 34],
    ]
*/
```

當根據多個屬性對集合進行排序時，您也可以提供定義每個排序操作的閉包：

```php
$collection = collect([
    ['name' => 'Taylor Otwell', 'age' => 34],
    ['name' => 'Abigail Otwell', 'age' => 30],
    ['name' => 'Taylor Otwell', 'age' => 36],
    ['name' => 'Abigail Otwell', 'age' => 32],
]);

$sorted = $collection->sortBy([
    fn (array $a, array $b) => $a['name'] <=> $b['name'],
    fn (array $a, array $b) => $b['age'] <=> $a['age'],
]);

$sorted->values()->all();

/*
    [
        ['name' => 'Abigail Otwell', 'age' => 32],
        ['name' => 'Abigail Otwell', 'age' => 30],
        ['name' => 'Taylor Otwell', 'age' => 36],
        ['name' => 'Taylor Otwell', 'age' => 34],
    ]
*/
```


<a name="method-sortbydesc"></a>
#### `sortByDesc()` {.collection-method}

此方法的簽章與 [sortBy](#method-sortby) 方法相同，但會以相反的順序對集合進行排序。


<a name="method-sortdesc"></a>
#### `sortDesc()` {.collection-method}

此方法將以與 [sort](#method-sort) 方法相反的順序對集合進行排序：

```php
$collection = collect([5, 3, 1, 2, 4]);

$sorted = $collection->sortDesc();

$sorted->values()->all();

// [5, 4, 3, 2, 1]
```

與 `sort` 不同，您不能向 `sortDesc` 傳遞閉包。相反地，您應該使用 [sort](#method-sort) 方法並反轉您的比較邏輯。


<a name="method-sortkeys"></a>
#### `sortKeys()` {.collection-method}

`sortKeys` 方法根據底層關聯陣列的鍵對集合進行排序：

```php
$collection = collect([
    'id' => 22345,
    'first' => 'John',
    'last' => 'Doe',
]);

$sorted = $collection->sortKeys();

$sorted->all();

/*
    [
        'first' => 'John',
        'id' => 22345,
        'last' => 'Doe',
    ]
*/
```


<a name="method-sortkeysdesc"></a>
#### `sortKeysDesc()` {.collection-method}

此方法的簽章與 [sortKeys](#method-sortkeys) 方法相同，但會以相反的順序對集合進行排序。


<a name="method-sortkeysusing"></a>
#### `sortKeysUsing()` {.collection-method}

`sortKeysUsing` 方法使用回呼根據底層關聯陣列的鍵對集合進行排序：

```php
$collection = collect([
    'ID' => 22345,
    'first' => 'John',
    'last' => 'Doe',
]);

$sorted = $collection->sortKeysUsing('strnatcasecmp');

$sorted->all();

/*
    [
        'first' => 'John',
        'ID' => 22345,
        'last' => 'Doe',
    ]
*/
```

回呼必須是一個比較函式，返回小於、等於或大於零的整數。欲瞭解更多資訊，請參閱有關 [uksort](https://www.php.net/manual/en/function.uksort.php#refsect1-function.uksort-parameters) 的 PHP 文件，這就是 `sortKeysUsing` 方法內部所使用的 PHP 函式。

<a name="method-splice"></a>
#### `splice()` {.collection-method}

`splice` 方法從指定的索引處開始移除並回傳項目切片：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunk = $collection->splice(2);

$chunk->all();

// [3, 4, 5]

$collection->all();

// [1, 2]
```

您可以傳入第二個參數來限制所產生的集合大小：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunk = $collection->splice(2, 1);

$chunk->all();

// [3]

$collection->all();

// [1, 2, 4, 5]
```

此外，您可以傳入第三個參數，包含要取代從集合中移除項目的新項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunk = $collection->splice(2, 1, [10, 11]);

$chunk->all();

// [3]

$collection->all();

// [1, 2, 10, 11, 4, 5]
```


<a name="method-split"></a>
#### `split()` {.collection-method}

`split` 方法將集合拆分成指定數量的群組：

```php
$collection = collect([1, 2, 3, 4, 5]);

$groups = $collection->split(3);

$groups->all();

// [[1, 2], [3, 4], [5]]
```


<a name="method-splitin"></a>
#### `splitIn()` {.collection-method}

`splitIn` 方法將集合拆分成指定數量的群組，在將餘數分配給最後一個群組之前，會先完全填滿非末端的群組：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

$groups = $collection->splitIn(3);

$groups->all();

// [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]
```


<a name="method-sum"></a>
#### `sum()` {.collection-method}

`sum` 方法回傳集合中所有項目的總和：

```php
collect([1, 2, 3, 4, 5])->sum();

// 15
```

如果集合包含巢狀陣列或物件，您應該傳入一個鍵 (key)，用來決定要加總哪些值：

```php
$collection = collect([
    ['name' => 'JavaScript: The Good Parts', 'pages' => 176],
    ['name' => 'JavaScript: The Definitive Guide', 'pages' => 1096],
]);

$collection->sum('pages');

// 1272
```

此外，您可以傳入自己的閉包來決定要加總集合中的哪些值：

```php
$collection = collect([
    ['name' => 'Chair', 'colors' => ['Black']],
    ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
    ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
]);

$collection->sum(function (array $product) {
    return count($product['colors']);
});

// 6
```


<a name="method-take"></a>
#### `take()` {.collection-method}

`take` 方法回傳包含指定項目數量的新集合：

```php
$collection = collect([0, 1, 2, 3, 4, 5]);

$chunk = $collection->take(3);

$chunk->all();

// [0, 1, 2]
```

您也可以傳入負整數，從集合末端開始取得指定數量的項目：

```php
$collection = collect([0, 1, 2, 3, 4, 5]);

$chunk = $collection->take(-2);

$chunk->all();

// [4, 5]
```


<a name="method-takeuntil"></a>
#### `takeUntil()` {.collection-method}

`takeUntil` 方法會回傳集合中的項目，直到指定的回呼回傳 `true` 為止：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->takeUntil(function (int $item) {
    return $item >= 3;
});

$subset->all();

// [1, 2]
```

您也可以傳入一個簡單的值給 `takeUntil` 方法，以取得直到找到該值之前的項目：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->takeUntil(3);

$subset->all();

// [1, 2]
```

> [!WARNING]
> 如果找不到指定的值，或回呼從未回傳 `true`，`takeUntil` 方法將回傳集合中的所有項目。


<a name="method-takewhile"></a>
#### `takeWhile()` {.collection-method}

`takeWhile` 方法會回傳集合中的項目，直到指定的回呼回傳 `false` 為止：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->takeWhile(function (int $item) {
    return $item < 3;
});

$subset->all();

// [1, 2]
```

> [!WARNING]
> 如果回呼從未回傳 `false`，`takeWhile` 方法將回傳集合中的所有項目。


<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 方法將集合傳遞給指定的回呼，允許您在特定時間點「介入 (tap)」集合，對項目進行某些操作，同時不影響集合本身。接著 `tap` 方法會回傳該集合：

```php
collect([2, 4, 3, 1, 5])
    ->sort()
    ->tap(function (Collection $collection) {
        Log::debug('Values after sorting', $collection->values()->all());
    })
    ->shift();

// 1
```


<a name="method-times"></a>
#### `times()` {.collection-method}

靜態 `times` 方法藉由呼叫指定次數的閉包來建立新的集合：

```php
$collection = Collection::times(10, function (int $number) {
    return $number * 9;
});

$collection->all();

// [9, 18, 27, 36, 45, 54, 63, 72, 81, 90]
```


<a name="method-toarray"></a>
#### `toArray()` {.collection-method}

`toArray` 方法將集合轉換為一般的 PHP `array`。如果集合的值是 Eloquent 模型，則模型也會被轉換為陣列：

```php
$collection = collect(['name' => 'Desk', 'price' => 200]);

$collection->toArray();

/*
    [
        ['name' => 'Desk', 'price' => 200],
    ]
*/
```

> [!WARNING]
> `toArray` 也會將集合中所有實作 `Arrayable` 介面的巢狀物件轉換為陣列。如果您想取得集合底層的原始陣列，請改用 [all](#method-all) 方法。


<a name="method-tojson"></a>
#### `toJson()` {.collection-method}

`toJson` 方法將集合轉換為 JSON 序列化字串：

```php
$collection = collect(['name' => 'Desk', 'price' => 200]);

$collection->toJson();

// '{"name":"Desk", "price":200}'
```


<a name="method-to-pretty-json"></a>
#### `toPrettyJson()` {.collection-method}

`toPrettyJson` 方法使用 `JSON_PRETTY_PRINT` 選項將集合轉換為格式化的 JSON 字串：

```php
$collection = collect(['name' => 'Desk', 'price' => 200]);

$collection->toPrettyJson();
```


<a name="method-transform"></a>
#### `transform()` {.collection-method}

`transform` 方法會走訪集合，並對集合中的每個項目呼叫指定的回呼。集合中的項目將被回呼回傳的值取代：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->transform(function (int $item, int $key) {
    return $item * 2;
});

$collection->all();

// [2, 4, 6, 8, 10]
```

> [!WARNING]
> 與大多數其他的集合方法不同，`transform` 會修改集合本身。如果您希望改為建立新集合，請使用 [map](#method-map) 方法。


<a name="method-undot"></a>
#### `undot()` {.collection-method}

`undot` 方法將使用「點 (dot)」記法的單維集合展開為多維集合：

```php
$person = collect([
    'name.first_name' => 'Marie',
    'name.last_name' => 'Valentine',
    'address.line_1' => '2992 Eagle Drive',
    'address.line_2' => '',
    'address.suburb' => 'Detroit',
    'address.state' => 'MI',
    'address.postcode' => '48219'
]);

$person = $person->undot();

$person->toArray();

/*
    [
        "name" => [
            "first_name" => "Marie",
            "last_name" => "Valentine",
        ],
        "address" => [
            "line_1" => "2992 Eagle Drive",
            "line_2" => "",
            "suburb" => "Detroit",
            "state" => "MI",
            "postcode" => "48219",
        ],
    ]
*/
```


<a name="method-union"></a>
#### `union()` {.collection-method}

`union` 方法將指定的陣列加入到集合中。如果指定的陣列中包含已存在於原始集合中的鍵 (key)，則會保留原始集合的值：

```php
$collection = collect([1 => ['a'], 2 => ['b']]);

$union = $collection->union([3 => ['c'], 1 => ['d']]);

$union->all();

// [1 => ['a'], 2 => ['b'], 3 => ['c']]
```

<a name="method-unique"></a>
#### `unique()` {.collection-method}

`unique` 方法回傳集合中所有不重複的項目。回傳的集合會保留原始的陣列鍵名，因此在下方的範例中，我們將使用 [values](#method-values) 方法將鍵名重置為連續數字的索引：

```php
$collection = collect([1, 1, 2, 2, 3, 4, 2]);

$unique = $collection->unique();

$unique->values()->all();

// [1, 2, 3, 4]
```

當處理巢狀陣列或物件時，您可以指定用於判斷唯一性的鍵名：

```php
$collection = collect([
    ['name' => 'iPhone 6', 'brand' => 'Apple', 'type' => 'phone'],
    ['name' => 'iPhone 5', 'brand' => 'Apple', 'type' => 'phone'],
    ['name' => 'Apple Watch', 'brand' => 'Apple', 'type' => 'watch'],
    ['name' => 'Galaxy S6', 'brand' => 'Samsung', 'type' => 'phone'],
    ['name' => 'Galaxy Gear', 'brand' => 'Samsung', 'type' => 'watch'],
]);

$unique = $collection->unique('brand');

$unique->values()->all();

/*
    [
        ['name' => 'iPhone 6', 'brand' => 'Apple', 'type' => 'phone'],
        ['name' => 'Galaxy S6', 'brand' => 'Samsung', 'type' => 'phone'],
    ]
*/
```

最後，您也可以傳遞自己的閉包給 `unique` 方法，以指定哪個值應該用來判斷項目的唯一性：

```php
$unique = $collection->unique(function (array $item) {
    return $item['brand'].$item['type'];
});

$unique->values()->all();

/*
    [
        ['name' => 'iPhone 6', 'brand' => 'Apple', 'type' => 'phone'],
        ['name' => 'Apple Watch', 'brand' => 'Apple', 'type' => 'watch'],
        ['name' => 'Galaxy S6', 'brand' => 'Samsung', 'type' => 'phone'],
        ['name' => 'Galaxy Gear', 'brand' => 'Samsung', 'type' => 'watch'],
    ]
*/
```

`unique` 方法在檢查項目值時使用「鬆散 (loose)」比較，這意味著包含整數值的字串將被視為與相同值的整數相等。請使用 [uniqueStrict](#method-uniquestrict) 方法來改用「嚴格 (strict)」比較進行篩選。

> [!NOTE]
> 當使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-unique) 時，此方法的行為會有所不同。


<a name="method-uniquestrict"></a>
#### `uniqueStrict()` {.collection-method}

此方法與 [unique](#method-unique) 方法具有相同的簽署 (Signature)；但是，所有值都使用「嚴格 (strict)」比較。


<a name="method-unless"></a>
#### `unless()` {.collection-method}

`unless` 方法將執行給定的回呼，除非傳遞給方法的第一個參數評估為 `true`。集合實例和傳遞給 `unless` 方法的第一個參數將被提供給該閉包：

```php
$collection = collect([1, 2, 3]);

$collection->unless(true, function (Collection $collection, bool $value) {
    return $collection->push(4);
});

$collection->unless(false, function (Collection $collection, bool $value) {
    return $collection->push(5);
});

$collection->all();

// [1, 2, 3, 5]
```

您可以傳遞第二個回呼給 `unless` 方法。當傳遞給 `unless` 方法的第一個參數評估為 `true` 時，將執行第二個回呼：

```php
$collection = collect([1, 2, 3]);

$collection->unless(true, function (Collection $collection, bool $value) {
    return $collection->push(4);
}, function (Collection $collection, bool $value) {
    return $collection->push(5);
});

$collection->all();

// [1, 2, 3, 5]
```

關於 `unless` 的反向操作，請參閱 [when](#method-when) 方法。


<a name="method-unlessempty"></a>
#### `unlessEmpty()` {.collection-method}

[whenNotEmpty](#method-whennotempty) 方法的別名。


<a name="method-unlessnotempty"></a>
#### `unlessNotEmpty()` {.collection-method}

[whenEmpty](#method-whenempty) 方法的別名。


<a name="method-unwrap"></a>
#### `unwrap()` {.collection-method}

靜態 `unwrap` 方法會在適用時，從給定值回傳集合底層的項目：

```php
Collection::unwrap(collect('John Doe'));

// ['John Doe']

Collection::unwrap(['John Doe']);

// ['John Doe']

Collection::unwrap('John Doe');

// 'John Doe'
```


<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 方法從集合的第一個元素中取得給定的值：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Speaker', 'price' => 400],
]);

$value = $collection->value('price');

// 200
```


<a name="method-values"></a>
#### `values()` {.collection-method}

`values` 方法回傳一個新集合，並將鍵名重置為連續整數：

```php
$collection = collect([
    10 => ['product' => 'Desk', 'price' => 200],
    11 => ['product' => 'Speaker', 'price' => 400],
]);

$values = $collection->values();

$values->all();

/*
    [
        0 => ['product' => 'Desk', 'price' => 200],
        1 => ['product' => 'Speaker', 'price' => 400],
    ]
*/
```


<a name="method-when"></a>
#### `when()` {.collection-method}

`when` 方法會在傳遞給方法的第一個參數評估為 `true` 時執行給定的回呼。集合實例和傳遞給 `when` 方法的第一個參數將被提供給該閉包：

```php
$collection = collect([1, 2, 3]);

$collection->when(true, function (Collection $collection, bool $value) {
    return $collection->push(4);
});

$collection->when(false, function (Collection $collection, bool $value) {
    return $collection->push(5);
});

$collection->all();

// [1, 2, 3, 4]
```

您可以傳遞第二個回呼給 `when` 方法。當傳遞給 `when` 方法的第一個參數評估為 `false` 時，將執行第二個回呼：

```php
$collection = collect([1, 2, 3]);

$collection->when(false, function (Collection $collection, bool $value) {
    return $collection->push(4);
}, function (Collection $collection, bool $value) {
    return $collection->push(5);
});

$collection->all();

// [1, 2, 3, 5]
```

關於 `when` 的反向操作，請參閱 [unless](#method-unless) 方法。


<a name="method-whenempty"></a>
#### `whenEmpty()` {.collection-method}

`whenEmpty` 方法會在集合為空時執行給定的回呼：

```php
$collection = collect(['Michael', 'Tom']);

$collection->whenEmpty(function (Collection $collection) {
    return $collection->push('Adam');
});

$collection->all();

// ['Michael', 'Tom']

$collection = collect();

$collection->whenEmpty(function (Collection $collection) {
    return $collection->push('Adam');
});

$collection->all();

// ['Adam']
```

您可以傳遞第二個閉包給 `whenEmpty` 方法，該閉包將在集合不為空時執行：

```php
$collection = collect(['Michael', 'Tom']);

$collection->whenEmpty(function (Collection $collection) {
    return $collection->push('Adam');
}, function (Collection $collection) {
    return $collection->push('Taylor');
});

$collection->all();

// ['Michael', 'Tom', 'Taylor']
```

關於 `whenEmpty` 的反向操作，請參閱 [whenNotEmpty](#method-whennotempty) 方法。


<a name="method-whennotempty"></a>
#### `whenNotEmpty()` {.collection-method}

`whenNotEmpty` 方法會在集合不為空時執行給定的回呼：

```php
$collection = collect(['Michael', 'Tom']);

$collection->whenNotEmpty(function (Collection $collection) {
    return $collection->push('Adam');
});

$collection->all();

// ['Michael', 'Tom', 'Adam']

$collection = collect();

$collection->whenNotEmpty(function (Collection $collection) {
    return $collection->push('Adam');
});

$collection->all();

// []
```

您可以傳遞第二個閉包給 `whenNotEmpty` 方法，該閉包將在集合為空時執行：

```php
$collection = collect();

$collection->whenNotEmpty(function (Collection $collection) {
    return $collection->push('Adam');
}, function (Collection $collection) {
    return $collection->push('Taylor');
});

$collection->all();

// ['Taylor']
```

關於 `whenNotEmpty` 的反向操作，請參閱 [whenEmpty](#method-whenempty) 方法。

<a name="method-where"></a>
#### `where()` {.collection-method}

`where` 方法透過指定的鍵 / 值配對過濾集合：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
    ['product' => 'Bookcase', 'price' => 150],
    ['product' => 'Door', 'price' => 100],
]);

$filtered = $collection->where('price', 100);

$filtered->all();

/*
    [
        ['product' => 'Chair', 'price' => 100],
        ['product' => 'Door', 'price' => 100],
    ]
*/
```

`where` 方法在檢查項目值時使用「寬鬆」比較，這意味著包含整數值的字串將被視為等於相同數值的整數。使用 [whereStrict](#method-wherestrict) 方法可以使用「嚴格」比較進行過濾，或者使用 [whereNull](#method-wherenull) 和 [whereNotNull](#method-wherenotnull) 方法來過濾 `null` 值。

您可以選擇性地將比較運算子作為第二個參數傳遞。支援的運算子有：'===', '!==', '!=', '==', '=', '<>', '>', '<', '>=', 和 '<='：

```php
$collection = collect([
    ['name' => 'Jim', 'platform' => 'Mac'],
    ['name' => 'Sally', 'platform' => 'Mac'],
    ['name' => 'Sue', 'platform' => 'Linux'],
]);

$filtered = $collection->where('platform', '!=', 'Linux');

$filtered->all();

/*
    [
        ['name' => 'Jim', 'platform' => 'Mac'],
        ['name' => 'Sally', 'platform' => 'Mac'],
    ]
*/
```


<a name="method-wherestrict"></a>
#### `whereStrict()` {.collection-method}

此方法的簽署與 [where](#method-where) 方法相同；但是，所有值都使用「嚴格」比較。


<a name="method-wherebetween"></a>
#### `whereBetween()` {.collection-method}

`whereBetween` 方法藉由判斷指定的項目值是否在給定的範圍內來過濾集合：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 80],
    ['product' => 'Bookcase', 'price' => 150],
    ['product' => 'Pencil', 'price' => 30],
    ['product' => 'Door', 'price' => 100],
]);

$filtered = $collection->whereBetween('price', [100, 200]);

$filtered->all();

/*
    [
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Bookcase', 'price' => 150],
        ['product' => 'Door', 'price' => 100],
    ]
*/
```


<a name="method-wherein"></a>
#### `whereIn()` {.collection-method}

`whereIn` 方法會從集合中移除不包含在給定陣列中指定項目值的元素：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
    ['product' => 'Bookcase', 'price' => 150],
    ['product' => 'Door', 'price' => 100],
]);

$filtered = $collection->whereIn('price', [150, 200]);

$filtered->all();

/*
    [
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Bookcase', 'price' => 150],
    ]
*/
```

`whereIn` 方法在檢查項目值時使用「寬鬆」比較，這意味著包含整數值的字串將被視為等於相同數值的整數。使用 [whereInStrict](#method-whereinstrict) 方法可以使用「嚴格」比較進行過濾。


<a name="method-whereinstrict"></a>
#### `whereInStrict()` {.collection-method}

此方法的簽署與 [whereIn](#method-wherein) 方法相同；但是，所有值都使用「嚴格」比較。


<a name="method-whereinstanceof"></a>
#### `whereInstanceOf()` {.collection-method}

`whereInstanceOf` 方法按指定的類別型別過濾集合：

```php
use App\Models\User;
use App\Models\Post;

$collection = collect([
    new User,
    new User,
    new Post,
]);

$filtered = $collection->whereInstanceOf(User::class);

$filtered->all();

// [App\Models\User, App\Models\User]
```


<a name="method-wherenotbetween"></a>
#### `whereNotBetween()` {.collection-method}

`whereNotBetween` 方法藉由判斷指定的項目值是否在給定的範圍之外來過濾集合：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 80],
    ['product' => 'Bookcase', 'price' => 150],
    ['product' => 'Pencil', 'price' => 30],
    ['product' => 'Door', 'price' => 100],
]);

$filtered = $collection->whereNotBetween('price', [100, 200]);

$filtered->all();

/*
    [
        ['product' => 'Chair', 'price' => 80],
        ['product' => 'Pencil', 'price' => 30],
    ]
*/
```


<a name="method-wherenotin"></a>
#### `whereNotIn()` {.collection-method}

`whereNotIn` 方法會從集合中移除包含在給定陣列中指定項目值的元素：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
    ['product' => 'Bookcase', 'price' => 150],
    ['product' => 'Door', 'price' => 100],
]);

$filtered = $collection->whereNotIn('price', [150, 200]);

$filtered->all();

/*
    [
        ['product' => 'Chair', 'price' => 100],
        ['product' => 'Door', 'price' => 100],
    ]
*/
```

`whereNotIn` 方法在檢查項目值時使用「寬鬆」比較，這意味著包含整數值的字串將被視為等於相同數值的整數。使用 [whereNotInStrict](#method-wherenotinstrict) 方法可以使用「嚴格」比較進行過濾。


<a name="method-wherenotinstrict"></a>
#### `whereNotInStrict()` {.collection-method}

此方法的簽署與 [whereNotIn](#method-wherenotin) 方法相同；但是，所有值都使用「嚴格」比較。


<a name="method-wherenotnull"></a>
#### `whereNotNull()` {.collection-method}

`whereNotNull` 方法從集合中回傳指定鍵不為 `null` 的項目：

```php
$collection = collect([
    ['name' => 'Desk'],
    ['name' => null],
    ['name' => 'Bookcase'],
    ['name' => 0],
    ['name' => ''],
]);

$filtered = $collection->whereNotNull('name');

$filtered->all();

/*
    [
        ['name' => 'Desk'],
        ['name' => 'Bookcase'],
        ['name' => 0],
        ['name' => ''],
    ]
*/
```


<a name="method-wherenull"></a>
#### `whereNull()` {.collection-method}

`whereNull` 方法從集合中回傳指定鍵為 `null` 的項目：

```php
$collection = collect([
    ['name' => 'Desk'],
    ['name' => null],
    ['name' => 'Bookcase'],
    ['name' => 0],
    ['name' => ''],
]);

$filtered = $collection->whereNull('name');

$filtered->all();

/*
    [
        ['name' => null],
    ]
*/
```


<a name="method-wrap"></a>
#### `wrap()` {.collection-method}

靜態 `wrap` 方法在適用時將給定的值包裝在集合中：

```php
use Illuminate\Support\Collection;

$collection = Collection::wrap('John Doe');

$collection->all();

// ['John Doe']

$collection = Collection::wrap(['John Doe']);

$collection->all();

// ['John Doe']

$collection = Collection::wrap(collect('John Doe'));

$collection->all();

// ['John Doe']
```


<a name="method-zip"></a>
#### `zip()` {.collection-method}

`zip` 方法將給定陣列的值與原始集合中對應索引的值合併在一起：

```php
$collection = collect(['Chair', 'Desk']);

$zipped = $collection->zip([100, 200]);

$zipped->all();

// [['Chair', 100], ['Desk', 200]]
```

<a name="higher-order-messages"></a>
## 高階訊息 (Higher Order Messages)

Collections 也支援「高階訊息 (higher order messages)」，這是對集合執行常見操作的捷徑。提供高階訊息的集合方法有：[average](#method-average)、[avg](#method-avg)、[contains](#method-contains)、[each](#method-each)、[every](#method-every)、[filter](#method-filter)、[first](#method-first)、[flatMap](#method-flatmap)、[groupBy](#method-groupby)、[keyBy](#method-keyby)、[map](#method-map)、[max](#method-max)、[min](#method-min)、[partition](#method-partition)、[reject](#method-reject)、[skipUntil](#method-skipuntil)、[skipWhile](#method-skipwhile)、[some](#method-some)、[sortBy](#method-sortby)、[sortByDesc](#method-sortbydesc)、[sum](#method-sum)、[takeUntil](#method-takeuntil)、[takeWhile](#method-takewhile) 以及 [unique](#method-unique)。

每個高階訊息都可以透過集合實例上的動態屬性來存取。例如，讓我們使用 `each` 高階訊息來呼叫集合中每個物件的方法：

```php
use App\Models\User;

$users = User::where('votes', '>', 500)->get();

$users->each->markAsVip();
```

同樣地，我們可以使用 `sum` 高階訊息來加總集合中所有使用者的「votes」總數：

```php
$users = User::where('group', 'Development')->get();

return $users->sum->votes;
```

<a name="lazy-collections"></a>
## Lazy 集合


<a name="lazy-collection-introduction"></a>
### 入門

> [!WARNING]
> 在進一步了解 Laravel 的 Lazy 集合之前，請先花點時間熟悉 [PHP 產生器 (Generators)](https://www.php.net/manual/en/language.generators.overview.php)。

為了補充功能已經相當強大的 `Collection` 類別，`LazyCollection` 類別利用了 PHP 的 [產生器 (Generators)](https://www.php.net/manual/en/language.generators.overview.php)，讓您在處理大型資料集時能同時保持較低的記憶體使用量。

例如，想像您的應用程式需要處理一個數 GB 大小的日誌檔案，同時又想利用 Laravel 的集合方法來解析這些日誌。與其一次將整個檔案讀入記憶體，不如使用 Lazy 集合，在給定的時間內只將檔案的一小部分保留在記憶體中：

```php
use App\Models\LogEntry;
use Illuminate\Support\LazyCollection;

LazyCollection::make(function () {
    $handle = fopen('log.txt', 'r');

    while (($line = fgets($handle)) !== false) {
        yield $line;
    }

    fclose($handle);
})->chunk(4)->map(function (array $lines) {
    return LogEntry::fromLines($lines);
})->each(function (LogEntry $logEntry) {
    // Process the log entry...
});
```

或者，想像您需要跑過 10,000 個 Eloquent 模型。當使用傳統的 Laravel 集合時，所有 10,000 個 Eloquent 模型都必須同時載入記憶體：

```php
use App\Models\User;

$users = User::all()->filter(function (User $user) {
    return $user->id > 500;
});
```

然而，查詢產生器的 `cursor` 方法會回傳一個 `LazyCollection` 實例。這讓您仍然可以只對資料庫執行單次查詢，但每次在記憶體中僅保留一個 Eloquent 模型。在這個範例中，直到我們實際對每個使用者進行迭代之前，`filter` 回呼都不會執行，這能大幅減少記憶體使用量：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```


<a name="creating-lazy-collections"></a>
### 建立 Lazy 集合

若要建立一個 Lazy 集合實例，您應該傳遞一個 PHP 產生器函式給集合的 `make` 方法：

```php
use Illuminate\Support\LazyCollection;

LazyCollection::make(function () {
    $handle = fopen('log.txt', 'r');

    while (($line = fgets($handle)) !== false) {
        yield $line;
    }

    fclose($handle);
});
```


<a name="the-enumerable-contract"></a>
### Enumerable 契約

幾乎所有在 `Collection` 類別中可用的方法，在 `LazyCollection` 類別中也同樣可用。這兩個類別都實作了 `Illuminate\Support\Enumerable` 契約，該契約定義了以下方法：

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

<div class="collection-method-list" markdown="1">

[all](#method-all)
[average](#method-average)
[avg](#method-avg)
[chunk](#method-chunk)
[chunkWhile](#method-chunkwhile)
[collapse](#method-collapse)
[collect](#method-collect)
[combine](#method-combine)
[concat](#method-concat)
[contains](#method-contains)
[containsStrict](#method-containsstrict)
[count](#method-count)
[countBy](#method-countBy)
[crossJoin](#method-crossjoin)
[dd](#method-dd)
[diff](#method-diff)
[diffAssoc](#method-diffassoc)
[diffKeys](#method-diffkeys)
[dump](#method-dump)
[duplicates](#method-duplicates)
[duplicatesStrict](#method-duplicatesstrict)
[each](#method-each)
[eachSpread](#method-eachspread)
[every](#method-every)
[except](#method-except)
[filter](#method-filter)
[first](#method-first)
[firstOrFail](#method-first-or-fail)
[firstWhere](#method-first-where)
[flatMap](#method-flatmap)
[flatten](#method-flatten)
[flip](#method-flip)
[forPage](#method-forpage)
[get](#method-get)
[groupBy](#method-groupby)
[has](#method-has)
[implode](#method-implode)
[intersect](#method-intersect)
[intersectAssoc](#method-intersectAssoc)
[intersectByKeys](#method-intersectbykeys)
[isEmpty](#method-isempty)
[isNotEmpty](#method-isnotempty)
[join](#method-join)
[keyBy](#method-keyby)
[keys](#method-keys)
[last](#method-last)
[macro](#method-macro)
[make](#method-make)
[map](#method-map)
[mapInto](#method-mapinto)
[mapSpread](#method-mapspread)
[mapToGroups](#method-maptogroups)
[mapWithKeys](#method-mapwithkeys)
[max](#method-max)
[median](#method-median)
[merge](#method-merge)
[mergeRecursive](#method-mergerecursive)
[min](#method-min)
[mode](#method-mode)
[nth](#method-nth)
[only](#method-only)
[pad](#method-pad)
[partition](#method-partition)
[pipe](#method-pipe)
[pluck](#method-pluck)
[random](#method-random)
[reduce](#method-reduce)
[reject](#method-reject)
[replace](#method-replace)
[replaceRecursive](#method-replacerecursive)
[reverse](#method-reverse)
[search](#method-search)
[shuffle](#method-shuffle)
[skip](#method-skip)
[slice](#method-slice)
[sole](#method-sole)
[some](#method-some)
[sort](#method-sort)
[sortBy](#method-sortby)
[sortByDesc](#method-sortbydesc)
[sortKeys](#method-sortkeys)
[sortKeysDesc](#method-sortkeysdesc)
[split](#method-split)
[sum](#method-sum)
[take](#method-take)
[tap](#method-tap)
[times](#method-times)
[toArray](#method-toarray)
[toJson](#method-tojson)
[union](#method-union)
[unique](#method-unique)
[uniqueStrict](#method-uniquestrict)
[unless](#method-unless)
[unlessEmpty](#method-unlessempty)
[unlessNotEmpty](#method-unlessnotempty)
[unwrap](#method-unwrap)
[values](#method-values)
[when](#method-when)
[whenEmpty](#method-whenempty)
[whenNotEmpty](#method-whennotempty)
[where](#method-where)
[whereStrict](#method-wherestrict)
[whereBetween](#method-wherebetween)
[whereIn](#method-wherein)
[whereInStrict](#method-whereinstrict)
[whereInstanceOf](#method-whereinstanceof)
[whereNotBetween](#method-wherenotbetween)
[whereNotIn](#method-wherenotin)
[whereNotInStrict](#method-wherenotinstrict)
[wrap](#method-wrap)
[zip](#method-zip)

</div>

> [!WARNING]
> 會變動集合的方法（例如 `shift`、`pop`、`prepend` 等）在 `LazyCollection` 類別中**不**可用。

<a name="lazy-collection-methods"></a>
### Lazy 集合方法

除了 `Enumerable` 契約中定義的方法外，`LazyCollection` 類別還包含以下方法：


<a name="method-takeUntilTimeout"></a>
#### `takeUntilTimeout()` {.collection-method}

`takeUntilTimeout` 方法會回傳一個新的 Lazy 集合，該集合會持續列舉數值直到指定的時間為止。超過該時間後，集合將停止列舉：

```php
$lazyCollection = LazyCollection::times(INF)
    ->takeUntilTimeout(now()->plus(minutes: 1));

$lazyCollection->each(function (int $number) {
    dump($number);

    sleep(1);
});

// 1
// 2
// ...
// 58
// 59
```

為了說明此方法的用法，請想像一個使用指標 (Cursor) 從資料庫送出發票的應用程式。您可以定義一個每 15 分鐘執行一次的[排程工作](/docs/{{version}}/scheduling)，並限制發票處理時間最多為 14 分鐘：

```php
use App\Models\Invoice;
use Illuminate\Support\Carbon;

Invoice::pending()->cursor()
    ->takeUntilTimeout(
        Carbon::createFromTimestamp(LARAVEL_START)->add(14, 'minutes')
    )
    ->each(fn (Invoice $invoice) => $invoice->submit());
```


<a name="method-tapEach"></a>
#### `tapEach()` {.collection-method}

雖然 `each` 方法會立即為集合中的每個項目呼叫給定的回呼，但 `tapEach` 方法只會在項目被逐一取出時才呼叫給定的回呼：

```php
// Nothing has been dumped so far...
$lazyCollection = LazyCollection::times(INF)->tapEach(function (int $value) {
    dump($value);
});

// Three items are dumped...
$array = $lazyCollection->take(3)->all();

// 1
// 2
// 3
```


<a name="method-throttle"></a>
#### `throttle()` {.collection-method}

`throttle` 方法會對 Lazy 集合進行節流，使得每個數值都在指定的秒數後才回傳。此方法對於與有速率限制 (Rate Limit) 的外部 API 進行互動的情況特別有用：

```php
use App\Models\User;

User::where('vip', true)
    ->cursor()
    ->throttle(seconds: 1)
    ->each(function (User $user) {
        // Call external API...
    });
```


<a name="method-remember"></a>
#### `remember()` {.collection-method}

`remember` 方法會回傳一個新的 Lazy 集合，該集合會記住任何已經列舉過的數值，並且在後續列舉集合時不會再次重新取得它們：

```php
// No query has been executed yet...
$users = User::cursor()->remember();

// The query is executed...
// The first 5 users are hydrated from the database...
$users->take(5)->all();

// First 5 users come from the collection's cache...
// The rest are hydrated from the database...
$users->take(20)->all();
```


<a name="method-with-heartbeat"></a>
#### `withHeartbeat()` {.collection-method}

`withHeartbeat` 方法允許您在列舉 Lazy 集合時，以固定的時間間隔執行回呼。這對於需要定期維護任務（例如延長鎖定或傳送進度更新）的長時間執行操作特別有用：

```php
use Carbon\CarbonInterval;
use Illuminate\Support\Facades\Cache;

$lock = Cache::lock('generate-reports', seconds: 60 * 5);

if ($lock->get()) {
    try {
        Report::where('status', 'pending')
            ->lazy()
            ->withHeartbeat(
                CarbonInterval::minutes(4),
                fn () => $lock->extend(CarbonInterval::minutes(5))
            )
            ->each(fn ($report) => $report->process());
    } finally {
        $lock->release();
    }
}
```