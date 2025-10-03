# 集合

- [簡介](#introduction)
    - [建立集合](#creating-collections)
    - [擴充集合](#extending-collections)
- [可用方法](#available-methods)
- [高階訊息](#higher-order-messages)
- [惰性集合](#lazy-collections)
    - [簡介](#lazy-collection-introduction)
    - [建立惰性集合](#creating-lazy-collections)
    - [Enumerable 契約](#the-enumerable-contract)
    - [惰性集合方法](#lazy-collection-methods)

<a name="introduction"></a>
## 簡介

`Illuminate\Support\Collection` 類別提供了一個流暢、便捷的包裝器，用於處理陣列資料。例如，請參考以下程式碼。我們將使用 `collect` 輔助函式從陣列中建立一個新的 Collection 實例，對每個元素執行 `strtoupper` 函式，然後移除所有空元素：

```php
$collection = collect(['Taylor', 'Abigail', null])->map(function (?string $name) {
    return strtoupper($name);
})->reject(function (string $name) {
    return empty($name);
});
```

如您所見，`Collection` 類別允許您鏈式呼叫其方法，以流暢地對底層陣列執行映射 (mapping) 與歸約 (reducing) 操作。一般而言，Collection 集合是不可變的，意即每個 `Collection` 方法都會回傳一個全新的 `Collection` 實例。

<a name="creating-collections"></a>
### 建立集合

如上所述，`collect` 輔助函式會為給定的陣列回傳一個新的 `Illuminate\Support\Collection` 實例。因此，建立 Collection 集合就像這樣簡單：

```php
$collection = collect([1, 2, 3]);
```

您也可以使用 [make](#method-make) 和 [fromJson](#method-fromjson) 方法來建立 Collection 集合。

> [!NOTE]
> [Eloquent](/docs/{{version}}/eloquent) 查詢的結果總是會以 `Collection` 實例的形式回傳。

<a name="extending-collections"></a>
### 擴充集合

Collection 集合是「可巨集化」(macroable) 的，這讓您可以在執行時期向 `Collection` 類別新增額外方法。`Illuminate\Support\Collection` 類別的 `macro` 方法接受一個閉包，該閉包將在您的巨集被呼叫時執行。此巨集閉包可以透過 `$this` 存取 Collection 的其他方法，就如同它是 Collection 類別的一個實際方法一樣。例如，以下程式碼為 `Collection` 類別新增了一個 `toUpper` 方法：

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

通常，您應該在 [Service Provider](/docs/{{version}}/providers) 的 `boot` 方法中宣告 Collection 巨集。

<a name="macro-arguments"></a>
#### 巨集引數

如有必要，您可以定義接受額外引數的巨集：

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
```

<a name="available-methods"></a>
## 可用方法

對於 Collection 集合文件的其餘大部分內容，我們將討論 `Collection` 類別中可用的每個方法。請記住，所有這些方法都可以鏈式呼叫，以流暢地操作底層陣列。此外，幾乎每個方法都會回傳一個新的 `Collection` 實例，這讓您可以在需要時保留 Collection 集合的原始副本：

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
[containsOneItem](#method-containsoneitem)
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
## 方法清單

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

`after` 方法傳回指定項目之後的項目。如果找不到指定項目或該項目是最後一個項目，則傳回 `null`：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->after(3);

// 4

$collection->after(5);

// null
```

此方法使用「鬆散」比較來搜尋指定項目，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比較，您可以向方法提供 `strict` 引數：

```php
collect([2, 4, 6, 8])->after('4', strict: true);

// null
```

或者，您可以提供自己的閉包來搜尋通過指定真值測試的第一個項目：

```php
collect([2, 4, 6, 8])->after(function (int $item, int $key) {
    return $item > 5;
});

// 8
```


<a name="method-all"></a>
#### `all()` {.collection-method}

`all` 方法傳回集合所代表的底層陣列：

```php
collect([1, 2, 3])->all();

// [1, 2, 3]
```


<a name="method-average"></a>
#### `average()` {.collection-method}

[avg](#method-avg) 方法的別名。


<a name="method-avg"></a>
#### `avg()` {.collection-method}

`avg` 方法傳回指定鍵的[平均值](https://en.wikipedia.org/wiki/Average)：

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

`before` 方法是 [after](#method-after) 方法的相反。它傳回指定項目之前的項目。如果找不到指定項目或該項目是第一個項目，則傳回 `null`：

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

`chunk` 方法將集合分成多個指定大小的較小集合：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7]);

$chunks = $collection->chunk(4);

$chunks->all();

// [[1, 2, 3, 4], [5, 6, 7]]
```

當與 [Bootstrap](https://getbootstrap.com/docs/5.3/layout/grid/) 等格線系統一起使用時，此方法在 [視圖](/docs/{{version}}/views) 中特別有用。例如，假設您有一個 [Eloquent](/docs/{{version}}/eloquent) 模型集合，您想將其顯示在格線中：

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

`chunkWhile` 方法根據指定回呼的評估結果，將集合分成多個較小的集合。傳遞給閉包的 `$chunk` 變數可用於檢查前一個元素：

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

`collapse` 方法將陣列或集合的集合折疊成單一的扁平集合：

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

`collapseWithKeys` 方法將陣列或集合的集合扁平化為單一集合，同時保持原始鍵不變。如果集合已扁平，則傳回空集合：

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

`collect` 方法傳回一個新的 `Collection` 實例，其中包含集合中目前的項目：

```php
$collectionA = collect([1, 2, 3]);

$collectionB = $collectionA->collect();

$collectionB->all();

// [1, 2, 3]
```

`collect` 方法主要用於將 [惰性集合](#lazy-collections) 轉換為標準的 `Collection` 實例：

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
> `collect` 方法在您擁有 `Enumerable` 實例並需要非惰性集合實例時特別有用。由於 `collect()` 是 `Enumerable` 契約的一部分，您可以安全地使用它來取得 `Collection` 實例。


<a name="method-combine"></a>
#### `combine()` {.collection-method}

`combine` 方法將集合的值作為鍵，與另一個陣列或集合的值組合：

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

`concat` 方法會對附加到原始集合的項目進行數字重新索引。若要在關聯式集合中保持鍵不變，請參閱 [merge](#method-merge) 方法。


<a name="method-contains"></a>
#### `contains()` {.collection-method}

`contains` 方法判斷集合是否包含指定項目。您可以向 `contains` 方法傳入一個閉包，以判斷集合中是否存在符合指定真值測試的元素：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->contains(function (int $value, int $key) {
    return $value > 5;
});

// false
```

或者，您可以向 `contains` 方法傳入一個字串，以判斷集合是否包含指定的項目值：

```php
$collection = collect(['name' => 'Desk', 'price' => 100]);

$collection->contains('Desk');

// true

$collection->contains('New York');

// false
```

您也可以向 `contains` 方法傳入一個鍵/值對，這將判斷指定鍵/值對是否存在於集合中：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
]);

$collection->contains('product', 'Bookcase');

// false
```

`contains` 方法在檢查項目值時使用「鬆散」比較，這表示包含整數值的字串將被視為與相同值的整數相等。請使用 [containsStrict](#method-containsstrict) 方法進行「嚴格」比較過濾。

有關 `contains` 的反向操作，請參閱 [doesntContain](#method-doesntcontain) 方法。


<a name="method-containsoneitem"></a>
#### `containsOneItem()` {.collection-method}

`containsOneItem` 方法判斷集合是否只包含一個項目：

```php
collect([])->containsOneItem();

// false

collect(['1'])->containsOneItem();

// true

collect(['1', '2'])->containsOneItem();

// false

collect([1, 2, 3])->containsOneItem(fn (int $item) => $item === 2);

// true
```

<a name="method-containsstrict"></a>
#### `containsStrict()` {.collection-method}

此方法與 [contains](#method-contains) 方法有相同的簽章；然而，所有值會使用「嚴格」比較進行比對。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-contains) 時，此方法的行為會有所不同。


<a name="method-count"></a>
#### `count()` {.collection-method}

`count` 方法會回傳集合中所有項目的總數：

```php
$collection = collect([1, 2, 3, 4]);

$collection->count();

// 4
```


<a name="method-countBy"></a>
#### `countBy()` {.collection-method}

`countBy` 方法會計算集合中值的出現次數。預設情況下，此方法會計算每個元素出現的次數，讓您可以計算集合中某些「類型」的元素：

```php
$collection = collect([1, 2, 2, 2, 3]);

$counted = $collection->countBy();

$counted->all();

// [1 => 1, 2 => 3, 3 => 1]
```

您可以向 `countBy` 方法傳入一個閉包，以根據自訂值來計算所有項目：

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

`crossJoin` 方法會將集合中的值與給定陣列或集合進行交叉合併，回傳一個包含所有可能排列的笛卡爾積：

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

`dd` 方法會傾印 (dump) 集合中的項目並終止指令碼的執行：

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

若您不想終止指令碼的執行，請改用 [dump](#method-dump) 方法。


<a name="method-diff"></a>
#### `diff()` {.collection-method}

`diff` 方法會依據值，將集合與另一個集合或純 PHP `array` 進行比較。此方法會回傳原始集合中，未存在於給定集合裡的值：

```php
$collection = collect([1, 2, 3, 4, 5]);

$diff = $collection->diff([2, 4, 6, 8]);

$diff->all();

// [1, 3, 5]
```

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-diff) 時，此方法的行為會有所不同。


<a name="method-diffassoc"></a>
#### `diffAssoc()` {.collection-method}

`diffAssoc` 方法會依據其鍵 (key) 與值，將集合與另一個集合或純 PHP `array` 進行比較。此方法會回傳原始集合中，未存在於給定集合裡的鍵值對 (key / value pair)：

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

與 `diffAssoc` 不同，`diffAssocUsing` 接受一個使用者提供的回呼函式來進行索引比較：

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

此回呼函式必須是一個比較函式，回傳一個小於、等於或大於零的整數。更多資訊請參閱 PHP 關於 [array_diff_uassoc](https://www.php.net/manual/en/function.array-diff-uassoc-parameters) 的文件，`diffAssocUsing` 方法內部就是使用這個 PHP 函式。


<a name="method-diffkeys"></a>
#### `diffKeys()` {.collection-method}

`diffKeys` 方法會依據其鍵 (key)，將集合與另一個集合或純 PHP `array` 進行比較。此方法會回傳原始集合中，未存在於給定集合裡的鍵值對 (key / value pair)：

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

`doesntContain` 方法會判斷集合是否不包含給定項目。您可以向 `doesntContain` 方法傳入一個閉包，來判斷集合中是否不存在與給定真值測試相符的元素：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->doesntContain(function (int $value, int $key) {
    return $value < 5;
});

// false
```

此外，您可以向 `doesntContain` 方法傳入一個字串，來判斷集合是否不包含給定項目值：

```php
$collection = collect(['name' => 'Desk', 'price' => 100]);

$collection->doesntContain('Table');

// true

$collection->doesntContain('Desk');

// false
```

您也可以向 `doesntContain` 方法傳入一個鍵值對 (key / value pair)，來判斷集合中是否不存在給定的鍵值對：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
]);

$collection->doesntContain('product', 'Bookcase');

// true
```

`doesntContain` 方法在檢查項目值時使用「寬鬆」比較，這表示一個包含整數值的字串將被視為與具有相同值的整數相等。


<a name="method-doesntcontainstrict"></a>
#### `doesntContainStrict()` {.collection-method}

此方法與 [doesntContain](#method-doesntcontain) 方法有相同的簽章；然而，所有值會使用「嚴格」比較進行比對。


<a name="method-dot"></a>
#### `dot()` {.collection-method}

`dot` 方法會將多維集合扁平化為使用「點」表示法指示深度的單層集合：

```php
$collection = collect(['products' => ['desk' => ['price' => 100]]]);

$flattened = $collection->dot();

$flattened->all();

// ['products.desk.price' => 100]
```


<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 方法會傾印 (dump) 集合中的項目：

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

若您想在傾印 (dump) 集合後終止指令碼的執行，請改用 [dd](#method-dd) 方法。


<a name="method-duplicates"></a>
#### `duplicates()` {.collection-method}

`duplicates` 方法會擷取並回傳集合中的重複值：

```php
$collection = collect(['a', 'b', 'a', 'c', 'b']);

$collection->duplicates();

// [2 => 'a', 4 => 'b']
```

若集合包含陣列或物件，您可以傳入屬性的鍵，以檢查重複值：

```php
$employees = collect([
    ['email' => 'abigail@example.com', 'position' => 'Developer'],
    ['email' => 'james@example.com', 'position' => 'Designer'],
    ['email' => 'victoria@example.com', 'position' => 'Developer'],
]);

$employees->duplicates('position');

// [2 => 'Developer']
```

#### `duplicatesStrict()` {.collection-method}

此方法與 [duplicates](#method-duplicates) 方法具有相同的簽名；然而，所有值都使用「嚴格」比較進行比對。

#### `each()` {.collection-method}

`each` 方法迭代集合中的項目，並將每個項目傳遞給閉包：

```php
$collection = collect([1, 2, 3, 4]);

$collection->each(function (int $item, int $key) {
    // ...
});
```

如果您想停止迭代項目，您可以從閉包中返回 `false`：

```php
$collection->each(function (int $item, int $key) {
    if (/* condition */) {
        return false;
    }
});
```

#### `eachSpread()` {.collection-method}

`eachSpread` 方法迭代集合中的項目，將每個巢狀項目值傳遞給給定的回呼：

```php
$collection = collect([['John Doe', 35], ['Jane Doe', 33]]);

$collection->eachSpread(function (string $name, int $age) {
    // ...
});
```

您可以從回呼中返回 `false` 來停止迭代項目：

```php
$collection->eachSpread(function (string $name, int $age) {
    return false;
});
```

#### `ensure()` {.collection-method}

`ensure` 方法可用來驗證集合中的所有元素是否屬於給定類型或類型列表。否則，將會拋出 `UnexpectedValueException` 異常：

```php
return $collection->ensure(User::class);

return $collection->ensure([User::class, Customer::class]);
```

也可以指定基本類型，例如 `string`、`int`、`float`、`bool` 和 `array`：

```php
return $collection->ensure('int');
```

> [!WARNING]
> `ensure` 方法不能保證不同類型的元素在稍後不會被新增到集合中。

#### `every()` {.collection-method}

`every` 方法可用來驗證集合中的所有元素是否都通過給定的真實性測試：

```php
collect([1, 2, 3, 4])->every(function (int $value, int $key) {
    return $value > 2;
});

// false
```

如果集合為空，`every` 方法將返回 true：

```php
$collection = collect([]);

$collection->every(function (int $value, int $key) {
    return $value > 2;
});

// true
```

#### `except()` {.collection-method}

`except` 方法返回集合中除指定鍵之外的所有項目：

```php
$collection = collect(['product_id' => 1, 'price' => 100, 'discount' => false]);

$filtered = $collection->except(['price', 'discount']);

$filtered->all();

// ['product_id' => 1]
```

有關 `except` 的反向操作，請參閱 [only](#method-only) 方法。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-except) 時，此方法的行為會有所不同。

#### `filter()` {.collection-method}

`filter` 方法使用給定的回呼過濾集合，只保留通過給定真實性測試的項目：

```php
$collection = collect([1, 2, 3, 4]);

$filtered = $collection->filter(function (int $value, int $key) {
    return $value > 2;
});

$filtered->all();

// [3, 4]
```

如果未提供回呼，集合中所有等同於 `false` 的項目都將被移除：

```php
$collection = collect([1, 2, 3, null, false, '', 0, []]);

$collection->filter()->all();

// [1, 2, 3]
```

有關 `filter` 的反向操作，請參閱 [reject](#method-reject) 方法。

#### `first()` {.collection-method}

`first` 方法返回集合中第一個通過給定真實性測試的元素：

```php
collect([1, 2, 3, 4])->first(function (int $value, int $key) {
    return $value > 2;
});

// 3
```

您也可以不帶參數呼叫 `first` 方法以取得集合中的第一個元素。如果集合為空，則返回 `null`：

```php
collect([1, 2, 3, 4])->first();

// 1
```

#### `firstOrFail()` {.collection-method}

`firstOrFail` 方法與 `first` 方法相同；然而，如果未找到結果，將會拋出 `Illuminate\Support\ItemNotFoundException` 異常：

```php
collect([1, 2, 3, 4])->firstOrFail(function (int $value, int $key) {
    return $value > 5;
});

// Throws ItemNotFoundException...
```

您也可以不帶參數呼叫 `firstOrFail` 方法以取得集合中的第一個元素。如果集合為空，將會拋出 `Illuminate\Support\ItemNotFoundException` 異常：

```php
collect([])->firstOrFail();

// Throws ItemNotFoundException...
```

#### `firstWhere()` {.collection-method}

`firstWhere` 方法返回集合中具有給定鍵/值對的第一個元素：

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

如同 [where](#method-where) 方法，您可以向 `firstWhere` 方法傳遞一個引數。在此情況下，`firstWhere` 方法將返回給定項目鍵的值為「truthy」的第一個項目：

```php
$collection->firstWhere('age');

// ['name' => 'Linda', 'age' => 14]
```

#### `flatMap()` {.collection-method}

`flatMap` 方法迭代集合，並將每個值傳遞給給定的閉包。閉包可以自由修改項目並返回，從而形成一個新的修改過的項目集合。然後，陣列會被扁平化一個層級：

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

#### `flatten()` {.collection-method}

`flatten` 方法將多維集合扁平化為單維：

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

如有必要，您可以向 `flatten` 方法傳遞一個「深度」引數：

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

在這個範例中，不提供深度呼叫 `flatten` 也會扁平化巢狀陣列，產生 `['iPhone 6S', 'Apple', 'Galaxy S7', 'Samsung']`。提供深度讓您可以指定巢狀陣列將被扁平化的層級數量。

#### `flip()` {.collection-method}

`flip` 方法將集合的鍵與其對應的值互換：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

$flipped = $collection->flip();

$flipped->all();

// ['Taylor' => 'name', 'Laravel' => 'framework']
```

<a name="method-forget"></a>
#### `forget()` {.collection-method}

`forget` 方法會依據鍵 (key) 從集合中移除一個項目：

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
> 與大多數其他集合方法不同，`forget` 並不會回傳一個新的修改過後的集合；它會修改並回傳其被呼叫的集合本身。


<a name="method-forpage"></a>
#### `forPage()` {.collection-method}

`forPage` 方法會回傳一個新的集合，其包含指定頁碼上的項目。此方法接受頁碼作為第一個引數，以及每頁要顯示的項目數量作為第二個引數：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9]);

$chunk = $collection->forPage(2, 3);

$chunk->all();

// [4, 5, 6]
```


<a name="method-fromjson"></a>
#### `fromJson()` {.collection-method}

靜態的 `fromJson` 方法會使用 `json_decode` PHP 函式解碼指定的 JSON 字串，以建立新的集合實例：

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

`get` 方法會回傳指定鍵的項目。如果該鍵不存在，則會回傳 `null`：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

$value = $collection->get('name');

// Taylor
```

您可以選擇傳遞預設值作為第二個引數：

```php
$collection = collect(['name' => 'Taylor', 'framework' => 'Laravel']);

$value = $collection->get('age', 34);

// 34
```

您甚至可以傳遞一個回呼函式作為方法的預設值。如果指定的鍵不存在，則會回傳該回呼函式的結果：

```php
$collection->get('email', function () {
    return 'taylor@example.com';
});

// taylor@example.com
```


<a name="method-groupby"></a>
#### `groupBy()` {.collection-method}

`groupBy` 方法會依據指定的鍵來分組集合中的項目：

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

除了傳遞字串 `key` 之外，您也可以傳遞一個回呼函式。此回呼函式應回傳您希望作為分組鍵的值：

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

可以傳遞多個分組條件作為陣列。每個陣列元素都會應用於多維陣列中對應的層級：

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

`has` 方法會判斷集合中是否存在指定的鍵：

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

`hasAny` 方法會判斷集合中是否存在任何一個指定的鍵：

```php
$collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

$collection->hasAny(['product', 'price']);

// true

$collection->hasAny(['name', 'price']);

// false
```


<a name="method-implode"></a>
#### `implode()` {.collection-method}

`implode` 方法會將集合中的項目連接起來。其引數取決於集合中項目的類型。如果集合包含陣列或物件，您應該傳遞要連接的屬性鍵，以及您希望放置在值之間的「連接字串」：

```php
$collection = collect([
    ['account_id' => 1, 'product' => 'Desk'],
    ['account_id' => 2, 'product' => 'Chair'],
]);

$collection->implode('product', ', ');

// 'Desk, Chair'
```

如果集合包含簡單的字串或數值，您應該只將「連接字串」作為方法唯一的引數傳遞：

```php
collect([1, 2, 3, 4, 5])->implode('-');

// '1-2-3-4-5'
```

如果您希望格式化要連接的值，可以傳遞一個回呼函式給 `implode` 方法：

```php
$collection->implode(function (array $item, int $key) {
    return strtoupper($item['product']);
}, ', ');

// 'DESK, CHAIR'
```


<a name="method-intersect"></a>
#### `intersect()` {.collection-method}

`intersect` 方法會從原始集合中移除任何未存在於指定陣列或集合中的值。結果集合將會保留原始集合的鍵：

```php
$collection = collect(['Desk', 'Sofa', 'Chair']);

$intersect = $collection->intersect(['Desk', 'Chair', 'Bookcase']);

$intersect->all();

// [0 => 'Desk', 2 => 'Chair']
```

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-intersect) 時，此方法的行為會被修改。


<a name="method-intersectusing"></a>
#### `intersectUsing()` {.collection-method}

`intersectUsing` 方法會從原始集合中移除任何未存在於指定陣列或集合中的值，並使用自訂回呼函式來比較這些值。結果集合將會保留原始集合的鍵：

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

`intersectAssoc` 方法會比較原始集合與另一個集合或陣列，並回傳存在於所有指定集合中的鍵/值對：

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

`intersectAssocUsing` 方法會將原始集合與另一個集合或陣列進行比較，並使用自訂比較回呼函數來判斷鍵與值的相等性，然後回傳兩者都存在的鍵 / 值對：

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

`intersectByKeys` 方法會從原始集合中移除給定陣列或集合中不存在的任何鍵及其對應的值：

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

`isEmpty` 方法如果集合為空，則回傳 `true`；否則回傳 `false`：

```php
collect([])->isEmpty();

// true
```


<a name="method-isnotempty"></a>
#### `isNotEmpty()` {.collection-method}

`isNotEmpty` 方法如果集合不為空，則回傳 `true`；否則回傳 `false`：

```php
collect([])->isNotEmpty();

// false
```


<a name="method-join"></a>
#### `join()` {.collection-method}

`join` 方法使用字串連接集合的值。您也可以使用此方法的第二個參數來指定最終元素應如何附加到字串中：

```php
collect(['a', 'b', 'c'])->join(', '); // 'a, b, c'
collect(['a', 'b', 'c'])->join(', ', ', and '); // 'a, b, and c'
collect(['a', 'b'])->join(', ', ' and '); // 'a and b'
collect(['a'])->join(', ', ' and '); // 'a'
collect([])->join(', ', ' and '); // ''
```


<a name="method-keyby"></a>
#### `keyBy()` {.collection-method}

`keyBy` 方法根據給定的鍵來索引集合。如果有多個項目具有相同的鍵，則只有最後一個會出現在新的集合中：

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

您也可以傳遞一個回呼函數給此方法。回呼函數應回傳用於索引集合的值：

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

`keys` 方法回傳集合中所有的鍵：

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

`last` 方法回傳集合中通過給定真值測試的最後一個元素：

```php
collect([1, 2, 3, 4])->last(function (int $value, int $key) {
    return $value < 3;
});

// 2
```

您也可以不帶任何引數呼叫 `last` 方法，以取得集合中的最後一個元素。如果集合為空，則回傳 `null`：

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

當您需要對包含許多項目的巨型 `Collection` 進行轉換時，這特別有用：

```php
$count = $hugeCollection
    ->lazy()
    ->where('country', 'FR')
    ->where('balance', '>', '100')
    ->count();
```

透過將集合轉換為 `LazyCollection`，我們可以避免分配大量額外記憶體。儘管原始集合仍將其值保留在記憶體中，但後續的篩選將不會。因此，在篩選集合結果時，幾乎不會分配額外記憶體。


<a name="method-macro"></a>
#### `macro()` {.collection-method}

靜態 `macro` 方法允許您在執行期間將方法新增至 `Collection` 類別。更多資訊，請參閱 [擴充集合](#extending-collections) 的文件。


<a name="method-make"></a>
#### `make()` {.collection-method}

靜態 `make` 方法會建立一個新的集合實例。請參閱 [建立集合](#creating-collections) 章節。

```php
use Illuminate\Support\Collection;

$collection = Collection::make([1, 2, 3]);
```


<a name="method-map"></a>
#### `map()` {.collection-method}

`map` 方法迭代集合並將每個值傳遞給指定的回呼函數。回呼函數可以自由修改項目並回傳，從而形成一個新的、經修改的項目集合：

```php
$collection = collect([1, 2, 3, 4, 5]);

$multiplied = $collection->map(function (int $item, int $key) {
    return $item * 2;
});

$multiplied->all();

// [2, 4, 6, 8, 10]
```

> [!WARNING]
> 與大多數其他集合方法一樣，`map` 會回傳一個新的集合實例；它不會修改被呼叫的集合。如果您想轉換原始集合，請使用 [transform](#method-transform) 方法。


<a name="method-mapinto"></a>
#### `mapInto()` {.collection-method}

`mapInto()` 方法迭代集合，透過將值傳遞給建構式來建立給定類別的新實例：

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

`mapSpread` 方法迭代集合中的項目，將每個巢狀項目值傳遞給指定的閉包函數。該閉包函數可以自由修改項目並回傳，從而形成一個新的、經修改的項目集合：

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

`mapToGroups` 方法根據給定的閉包函數將集合中的項目分組。該閉包函數應回傳包含單一鍵 / 值對的關聯式陣列，從而形成一個新的分組值集合：

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

`mapWithKeys` 方法會迭代遍歷集合並將每個值傳遞給給定的回呼。此回呼應該回傳一個包含單一鍵/值對的關聯陣列：

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

`max` 方法會回傳給定鍵的最大值：

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

`median` 方法會回傳給定鍵的 [中位數](https://en.wikipedia.org/wiki/Median)：

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

`merge` 方法會將給定的陣列或集合與原始集合合併。如果給定項目中的字串鍵與原始集合中的字串鍵匹配，給定項目中的值將會覆寫原始集合中的值：

```php
$collection = collect(['product_id' => 1, 'price' => 100]);

$merged = $collection->merge(['price' => 200, 'discount' => false]);

$merged->all();

// ['product_id' => 1, 'price' => 200, 'discount' => false]
```

如果給定項目的鍵是數字，這些值將被附加到集合的末端：

```php
$collection = collect(['Desk', 'Chair']);

$merged = $collection->merge(['Bookcase', 'Door']);

$merged->all();

// ['Desk', 'Chair', 'Bookcase', 'Door']
```


<a name="method-mergerecursive"></a>
#### `mergeRecursive()` {.collection-method}

`mergeRecursive` 方法會將給定的陣列或集合與原始集合遞迴地合併。如果給定項目中的字串鍵與原始集合中的字串鍵匹配，則這些鍵的值將被合併到一個陣列中，並遞迴執行此操作：

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

`min` 方法會回傳給定鍵的最小值：

```php
$min = collect([['foo' => 10], ['foo' => 20]])->min('foo');

// 10

$min = collect([1, 2, 3, 4, 5])->min();

// 1
```


<a name="method-mode"></a>
#### `mode()` {.collection-method}

`mode` 方法會回傳給定鍵的 [眾數](https://en.wikipedia.org/wiki/Mode_(statistics))：

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

`multiply` 方法會為集合中的所有項目建立指定數量的副本：

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

`nth` 方法會建立一個由每 n 個元素組成的新集合：

```php
$collection = collect(['a', 'b', 'c', 'd', 'e', 'f']);

$collection->nth(4);

// ['a', 'e']
```

您可以選擇傳遞一個起始偏移量作為第二個參數：

```php
$collection->nth(4, 1);

// ['b', 'f']
```


<a name="method-only"></a>
#### `only()` {.collection-method}

`only` 方法會回傳集合中具有指定鍵的項目：

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

若要取得 `only` 的相反結果，請參閱 [except](#method-except) 方法。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-only) 時，此方法的行為會有所不同。


<a name="method-pad"></a>
#### `pad()` {.collection-method}

`pad` 方法會用給定值填充陣列，直到陣列達到指定大小。此方法行為與 PHP 的 [array_pad](https://secure.php.net/manual/en/function.array-pad.php) 函式相似。

若要向左填充，您應該指定一個負數大小。如果給定大小的絕對值小於或等於陣列的長度，則不會進行填充：

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

`partition` 方法可以與 PHP 陣列解構結合使用，以將通過給定真值測試的元素與不通過的元素分開：

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
> 與 [Eloquent collections](/docs/{{version}}/eloquent-collections#method-partition) 互動時，此方法的行為會有所不同。


<a name="method-percentage"></a>
#### `percentage()` {.collection-method}

`percentage` 方法可用於快速判斷集合中通過給定真值測試的項目所佔的百分比：

```php
$collection = collect([1, 1, 2, 2, 2, 3]);

$percentage = $collection->percentage(fn (int $value) => $value === 1);

// 33.33
```

預設情況下，百分比將四捨五入到小數點後兩位。不過，您可以透過提供第二個參數來自訂此行為：

```php
$percentage = $collection->percentage(fn (int $value) => $value === 1, precision: 3);

// 33.333
```


<a name="method-pipe"></a>
#### `pipe()` {.collection-method}

`pipe` 方法會將集合傳遞給給定的閉包，並回傳已執行閉包的結果：

```php
$collection = collect([1, 2, 3]);

$piped = $collection->pipe(function (Collection $collection) {
    return $collection->sum();
});

// 6
```


<a name="method-pipeinto"></a>
#### `pipeInto()` {.collection-method}

`pipeInto` 方法會建立給定類別的新實例，並將集合傳遞給建構式：

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

`pipeThrough` 方法會將集合傳遞給給定的閉包陣列，並回傳這些閉包的執行結果：

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

`pluck` 方法會擷取指定鍵的所有值：

```php
$collection = collect([
    ['product_id' => 'prod-100', 'name' => 'Desk'],
    ['product_id' => 'prod-200', 'name' => 'Chair'],
]);

$plucked = $collection->pluck('name');

$plucked->all();

// ['Desk', 'Chair']
```

您也可以指定結果集合的鍵：

```php
$plucked = $collection->pluck('name', 'product_id');

$plucked->all();

// ['prod-100' => 'Desk', 'prod-200' => 'Chair']
```

`pluck` 方法也支援使用「點」符號來擷取巢狀值：

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

如果存在重複的鍵，最後一個符合的元素將會被插入到已擷取的集合中：

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

`pop` 方法會移除並回傳集合中的最後一個項目。如果集合是空的，則會回傳 `null`：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->pop();

// 5

$collection->all();

// [1, 2, 3, 4]
```

您可以向 `pop` 方法傳入一個整數，以從集合的末尾移除並回傳多個項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->pop(3);

// collect([5, 4, 3])

$collection->all();

// [1, 2]
```


<a name="method-prepend"></a>
#### `prepend()` {.collection-method}

`prepend` 方法會將一個項目加到集合的開頭：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->prepend(0);

$collection->all();

// [0, 1, 2, 3, 4, 5]
```

您也可以傳入第二個引數來指定插入項目 (prepended item) 的鍵：

```php
$collection = collect(['one' => 1, 'two' => 2]);

$collection->prepend(0, 'zero');

$collection->all();

// ['zero' => 0, 'one' => 1, 'two' => 2]
```


<a name="method-pull"></a>
#### `pull()` {.collection-method}

`pull` 方法會依據鍵來移除並回傳集合中的項目：

```php
$collection = collect(['product_id' => 'prod-100', 'name' => 'Desk']);

$collection->pull('name');

// 'Desk'

$collection->all();

// ['product_id' => 'prod-100']
```


<a name="method-push"></a>
#### `push()` {.collection-method}

`push` 方法會將一個項目附加到集合的末尾：

```php
$collection = collect([1, 2, 3, 4]);

$collection->push(5);

$collection->all();

// [1, 2, 3, 4, 5]
```

您也可以提供多個項目來附加到集合的末尾：

```php
$collection = collect([1, 2, 3, 4]);

$collection->push(5, 6, 7);
 
$collection->all();
 
// [1, 2, 3, 4, 5, 6, 7]
```


<a name="method-put"></a>
#### `put()` {.collection-method}

`put` 方法會在集合中設定給定的鍵和值：

```php
$collection = collect(['product_id' => 1, 'name' => 'Desk']);

$collection->put('price', 100);

$collection->all();

// ['product_id' => 1, 'name' => 'Desk', 'price' => 100]
```


<a name="method-random"></a>
#### `random()` {.collection-method}

`random` 方法會從集合中回傳一個隨機項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->random();

// 4 - (retrieved randomly)
```

您可以向 `random` 傳入一個整數，來指定您希望隨機擷取多少個項目。當明確傳入您希望收到的項目數量時，總是會回傳一個項目集合：

```php
$random = $collection->random(3);

$random->all();

// [2, 4, 5] - (retrieved randomly)
```

如果集合實例的項目少於請求的數量，`random` 方法將會拋出 `InvalidArgumentException`。

`random` 方法也接受一個閉包，該閉包將會接收當前的集合實例：

```php
use Illuminate\Support\Collection;

$random = $collection->random(fn (Collection $items) => min(10, count($items)));

$random->all();

// [1, 2, 3, 4, 5] - (retrieved randomly)
```


<a name="method-range"></a>
#### `range()` {.collection-method}

`range` 方法會回傳一個包含指定範圍內整數的集合：

```php
$collection = collect()->range(3, 6);

$collection->all();

// [3, 4, 5, 6]
```


<a name="method-reduce"></a>
#### `reduce()` {.collection-method}

`reduce` 方法會將集合縮減為單一值，並將每次迭代的結果傳遞給後續的迭代：

```php
$collection = collect([1, 2, 3]);

$total = $collection->reduce(function (?int $carry, int $item) {
    return $carry + $item;
});

// 6
```

在第一次迭代中，`$carry` 的值是 `null`；不過，您可以透過向 `reduce` 傳遞第二個引數來指定其初始值：

```php
$collection->reduce(function (int $carry, int $item) {
    return $carry + $item;
}, 4);

// 10
```

`reduce` 方法也會將陣列鍵傳遞給給定回呼：

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

`reduceSpread` 方法將集合縮減為值的陣列，並將每次迭代的結果傳遞給後續的迭代。此方法類似於 `reduce` 方法；不過，它可以接受多個初始值：

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

`reject` 方法使用給定的閉包來過濾集合。如果項目應從結果集合中移除，則閉包應回傳 `true`：

```php
$collection = collect([1, 2, 3, 4]);

$filtered = $collection->reject(function (int $value, int $key) {
    return $value > 2;
});

$filtered->all();

// [1, 2]
```

有關 `reject` 方法的反向操作，請參閱 [filter](#method-filter) 方法。


<a name="method-replace"></a>
#### `replace()` {.collection-method}

`replace` 方法的行為與 `merge` 類似；不過，除了覆寫具有字串鍵的符合項目外，`replace` 方法還會覆寫集合中具有符合數字鍵的項目：

```php
$collection = collect(['Taylor', 'Abigail', 'James']);

$replaced = $collection->replace([1 => 'Victoria', 3 => 'Finn']);

$replaced->all();

// ['Taylor', 'Victoria', 'James', 'Finn']
```

<a name="method-replacerecursive"></a>
#### `replaceRecursive()` {.collection-method}

`replaceRecursive` 方法的行為類似於 `replace`，但它會遞迴處理陣列，並對內部值套用相同的替換程序：

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

`reverse` 方法會反轉集合中項目的順序，同時保留原始鍵：

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

`search` 方法會在集合中搜尋給定的值，若找到則回傳其鍵。若找不到項目，則回傳 `false`：

```php
$collection = collect([2, 4, 6, 8]);

$collection->search(4);

// 1
```

搜尋使用「寬鬆」比較進行，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比較，您可以將 `true` 作為方法的第二個引數傳入：

```php
collect([2, 4, 6, 8])->search('4', strict: true);

// false
```

或者，您可以提供您自己的閉包來搜尋第一個通過給定真值測試的項目：

```php
collect([2, 4, 6, 8])->search(function (int $item, int $key) {
    return $item > 5;
});

// 2
```


<a name="method-select"></a>
#### `select()` {.collection-method}

`select` 方法會選取集合中指定的鍵，類似於 SQL 的 `SELECT` 陳述式：

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

`shift` 方法會從集合中移除並回傳第一個項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->shift();

// 1

$collection->all();

// [2, 3, 4, 5]
```

您可以將一個整數傳入 `shift` 方法，以從集合開頭移除並回傳多個項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->shift(3);

// collect([1, 2, 3])

$collection->all();

// [4, 5]
```


<a name="method-shuffle"></a>
#### `shuffle()` {.collection-method}

`shuffle` 方法會隨機打亂集合中的項目：

```php
$collection = collect([1, 2, 3, 4, 5]);

$shuffled = $collection->shuffle();

$shuffled->all();

// [3, 2, 5, 1, 4] - (generated randomly)
```


<a name="method-skip"></a>
#### `skip()` {.collection-method}

`skip` 方法會回傳一個新集合，其中從集合開頭移除了指定數量的元素：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

$collection = $collection->skip(4);

$collection->all();

// [5, 6, 7, 8, 9, 10]
```


<a name="method-skipuntil"></a>
#### `skipUntil()` {.collection-method}

`skipUntil` 方法會跳過集合中的項目，直到給定的回呼函式回傳 `true`。一旦回呼函式回傳 `true`，集合中所有剩餘項目都會作為新集合回傳：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->skipUntil(function (int $item) {
    return $item >= 3;
});

$subset->all();

// [3, 4]
```

您也可以將一個簡單的值傳入 `skipUntil` 方法，以跳過所有項目直到找到指定的值：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->skipUntil(3);

$subset->all();

// [3, 4]
```

> [!WARNING]
> 如果找不到給定的值，或回呼函式從未回傳 `true`，`skipUntil` 方法將回傳一個空集合。


<a name="method-skipwhile"></a>
#### `skipWhile()` {.collection-method}

`skipWhile` 方法會跳過集合中的項目，直到給定的回呼函式回傳 `false`。一旦回呼函式回傳 `false`，集合中所有剩餘項目都會作為新集合回傳：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->skipWhile(function (int $item) {
    return $item <= 3;
});

$subset->all();

// [4]
```

> [!WARNING]
> 如果回呼函式從未回傳 `false`，`skipWhile` 方法將回傳一個空集合。


<a name="method-slice"></a>
#### `slice()` {.collection-method}

`slice` 方法會回傳集合的一個切片，從給定的索引開始：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

$slice = $collection->slice(4);

$slice->all();

// [5, 6, 7, 8, 9, 10]
```

如果您希望限制回傳切片的大小，可以將所需大小作為第二個引數傳入該方法：

```php
$slice = $collection->slice(4, 2);

$slice->all();

// [5, 6]
```

回傳的切片預設會保留鍵。如果您不希望保留原始鍵，可以使用 [values](#method-values) 方法重新索引它們。


<a name="method-sliding"></a>
#### `sliding()` {.collection-method}

`sliding` 方法會回傳一個新的分塊集合，代表集合中項目的「滑動視窗」視圖：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunks = $collection->sliding(2);

$chunks->toArray();

// [[1, 2], [2, 3], [3, 4], [4, 5]]
```

這與 [eachSpread](#method-eachspread) 方法結合使用時特別有用：

```php
$transactions->sliding(2)->eachSpread(function (Collection $previous, Collection $current) {
    $current->total = $previous->total + $current->amount;
});
```

您可以選擇傳入第二個「步進」值，它決定每個分塊中第一個項目之間的距離：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunks = $collection->sliding(3, step: 2);

$chunks->toArray();

// [[1, 2, 3], [3, 4, 5]]
```


<a name="method-sole"></a>
#### `sole()` {.collection-method}

`sole` 方法會回傳集合中第一個通過給定真值測試的元素，但僅限於該真值測試只匹配到一個元素的情況：

```php
collect([1, 2, 3, 4])->sole(function (int $value, int $key) {
    return $value === 2;
});

// 2
```

您也可以將鍵/值對傳入 `sole` 方法，它會回傳集合中與給定鍵/值對匹配的第一個元素，但僅限於該鍵/值對只匹配到一個元素的情況：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
    ['product' => 'Chair', 'price' => 100],
]);

$collection->sole('product', 'Chair');

// ['product' => 'Chair', 'price' => 100]
```

或者，您也可以不傳入引數呼叫 `sole` 方法，若集合中只有一個元素，則取得該第一個元素：

```php
$collection = collect([
    ['product' => 'Desk', 'price' => 200],
]);

$collection->sole();

// ['product' => 'Desk', 'price' => 200]
```

如果集合中沒有應由 `sole` 方法回傳的元素，將會拋出 `\Illuminate\Collections\ItemNotFoundException` 異常。如果有多個元素應回傳，將會拋出 `\Illuminate\Collections\MultipleItemsFoundException` 異常。


<a name="method-some"></a>
#### `some()` {.collection-method}

[contains](#method-contains) 方法的別名。

<a name="method-sort"></a>
#### `sort()` {.collection-method}

`sort` 方法會對集合進行排序。排序後的集合會保留原始陣列的鍵，因此在以下範例中，我們將使用 [values](#method-values) 方法將鍵重設為連續的數字索引：

```php
$collection = collect([5, 3, 1, 2, 4]);

$sorted = $collection->sort();

$sorted->values()->all();

// [1, 2, 3, 4, 5]
```

如果您的排序需求更複雜，您可以傳遞一個回呼函式給 `sort` 並使用您自己的演算法。請參考 PHP 關於 [uasort](https://secure.php.net/manual/en/function.uasort.php#refsect1-function.uasort-parameters) 的文件，這是集合的 `sort` 方法內部使用的 PHP 函式。

> [!NOTE]
> 如果您需要排序巢狀陣列或物件的集合，請參閱 [sortBy](#method-sortby) 和 [sortByDesc](#method-sortbydesc) 方法。


<a name="method-sortby"></a>
#### `sortBy()` {.collection-method}

`sortBy` 方法會依據給定的鍵對集合進行排序。排序後的集合會保留原始陣列的鍵，因此在以下範例中，我們將使用 [values](#method-values) 方法將鍵重設為連續的數字索引：

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

`sortBy` 方法接受 [sort flags](https://www.php.net/manual/en/function.sort.php) 作為其第二個參數：

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

另外，您可以傳遞自己的閉包來決定如何排序集合的值：

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

如果您想依據多個屬性對集合進行排序，您可以傳遞一個排序操作陣列給 `sortBy` 方法。每個排序操作都應該是一個陣列，包含您希望排序的屬性以及期望的排序方向：

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

當依據多個屬性對集合進行排序時，您也可以提供定義每個排序操作的閉包：

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

此方法與 [sortBy](#method-sortby) 方法的簽章相同，但會以相反的順序排序集合。


<a name="method-sortdesc"></a>
#### `sortDesc()` {.collection-method}

此方法會以與 [sort](#method-sort) 方法相反的順序排序集合：

```php
$collection = collect([5, 3, 1, 2, 4]);

$sorted = $collection->sortDesc();

$sorted->values()->all();

// [5, 4, 3, 2, 1]
```

與 `sort` 不同，您不能將閉包傳遞給 `sortDesc`。相反，您應該使用 [sort](#method-sort) 方法並反轉您的比較。


<a name="method-sortkeys"></a>
#### `sortKeys()` {.collection-method}

`sortKeys` 方法會依據底層關聯陣列的鍵來排序集合：

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

此方法與 [sortKeys](#method-sortkeys) 方法的簽章相同，但會以相反的順序排序集合。


<a name="method-sortkeysusing"></a>
#### `sortKeysUsing()` {.collection-method}

`sortKeysUsing` 方法會使用回呼函式依據底層關聯陣列的鍵來排序集合：

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

回呼函式必須是一個比較函式，回傳小於、等於或大於零的整數。有關更多資訊，請參閱 PHP 關於 [uksort](https://www.php.net/manual/en/function.uksort.php#refsect1-function.uksort-parameters) 的文件，這是 `sortKeysUsing` 方法內部使用的 PHP 函式。


<a name="method-splice"></a>
#### `splice()` {.collection-method}

`splice` 方法會從指定索引開始移除並回傳一個項目切片：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunk = $collection->splice(2);

$chunk->all();

// [3, 4, 5]

$collection->all();

// [1, 2]
```

您可以傳遞第二個參數來限制結果集合的大小：

```php
$collection = collect([1, 2, 3, 4, 5]);

$chunk = $collection->splice(2, 1);

$chunk->all();

// [3]

$collection->all();

// [1, 2, 4, 5]
```

此外，您可以傳遞第三個參數，其中包含要替換從集合中移除項目的新項目：

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

`split` 方法會將集合分成給定數量的群組：

```php
$collection = collect([1, 2, 3, 4, 5]);

$groups = $collection->split(3);

$groups->all();

// [[1, 2], [3, 4], [5]]
```


<a name="method-splitin"></a>
#### `splitIn()` {.collection-method}

`splitIn` 方法會將集合分成給定數量的群組，在將剩餘項目分配給最終群組之前，會完全填滿非最終群組：

```php
$collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

$groups = $collection->splitIn(3);

$groups->all();

// [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]
```

<a name="method-sum"></a>
#### `sum()` {.collection-method}

`sum` 方法會回傳集合中所有項目的總和：

```php
collect([1, 2, 3, 4, 5])->sum();

// 15
```

如果集合包含巢狀陣列或物件，您應該傳遞一個鍵 (key)，用來指定要加總的值：

```php
$collection = collect([
    ['name' => 'JavaScript: The Good Parts', 'pages' => 176],
    ['name' => 'JavaScript: The Definitive Guide', 'pages' => 1096],
]);

$collection->sum('pages');

// 1272
```

此外，您可以傳遞自己的閉包 (closure) 來指定集合中哪些值要進行加總：

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

`take` 方法會回傳一個新的集合，其中包含指定數量的項目：

```php
$collection = collect([0, 1, 2, 3, 4, 5]);

$chunk = $collection->take(3);

$chunk->all();

// [0, 1, 2]
```

您也可以傳遞一個負整數來從集合的末端取得指定數量的項目：

```php
$collection = collect([0, 1, 2, 3, 4, 5]);

$chunk = $collection->take(-2);

$chunk->all();

// [4, 5]
```


<a name="method-takeuntil"></a>
#### `takeUntil()` {.collection-method}

`takeUntil` 方法會回傳集合中的項目，直到給定的回呼 (callback) 函數回傳 `true` 為止：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->takeUntil(function (int $item) {
    return $item >= 3;
});

$subset->all();

// [1, 2]
```

您也可以傳遞一個簡單的值給 `takeUntil` 方法來取得項目，直到找到指定的值：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->takeUntil(3);

$subset->all();

// [1, 2]
```

> [!WARNING]
> 如果找不到指定的值或回呼函數從未回傳 `true`，`takeUntil` 方法將會回傳集合中的所有項目。


<a name="method-takewhile"></a>
#### `takeWhile()` {.collection-method}

`takeWhile` 方法會回傳集合中的項目，直到給定的回呼函數回傳 `false` 為止：

```php
$collection = collect([1, 2, 3, 4]);

$subset = $collection->takeWhile(function (int $item) {
    return $item < 3;
});

$subset->all();

// [1, 2]
```

> [!WARNING]
> 如果回呼函數從未回傳 `false`，`takeWhile` 方法將會回傳集合中的所有項目。


<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 方法會將集合傳遞給給定的回呼，讓您可以在特定時間點「探查 (tap)」集合並對這些項目進行操作，同時不影響集合本身。然後，`tap` 方法會回傳這個集合：

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

靜態 `times` 方法會透過呼叫給定的閉包指定次數來建立一個新的集合：

```php
$collection = Collection::times(10, function (int $number) {
    return $number * 9;
});

$collection->all();

// [9, 18, 27, 36, 45, 54, 63, 72, 81, 90]
```


<a name="method-toarray"></a>
#### `toArray()` {.collection-method}

`toArray` 方法會將集合轉換為普通的 PHP `array`。如果集合中的值是 [Eloquent](/docs/{{version}}/eloquent) 模型，這些模型也將被轉換為陣列：

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
> `toArray` 也會將集合中所有 `Arrayable` 實例的巢狀物件轉換為陣列。如果您想取得集合底層的原始陣列，請改用 [all](#method-all) 方法。


<a name="method-tojson"></a>
#### `toJson()` {.collection-method}

`toJson` 方法會將集合轉換為 JSON 序列化字串：

```php
$collection = collect(['name' => 'Desk', 'price' => 200]);

$collection->toJson();

// '{"name":"Desk", "price":200}'
```


<a name="method-to-pretty-json"></a>
#### `toPrettyJson()` {.collection-method}

`toPrettyJson` 方法會使用 `JSON_PRETTY_PRINT` 選項將集合轉換為格式化的 JSON 字串：

```php
$collection = collect(['name' => 'Desk', 'price' => 200]);

$collection->toPrettyJson();
```


<a name="method-transform"></a>
#### `transform()` {.collection-method}

`transform` 方法會遍歷集合，並對集合中的每個項目呼叫給定的回呼函數。集合中的項目將被回呼函數回傳的值替換：

```php
$collection = collect([1, 2, 3, 4, 5]);

$collection->transform(function (int $item, int $key) {
    return $item * 2;
});

$collection->all();

// [2, 4, 6, 8, 10]
```

> [!WARNING]
> 與大多數其他集合方法不同，`transform` 會修改集合本身。如果您希望建立一個新的集合，請使用 [map](#method-map) 方法。


<a name="method-undot"></a>
#### `undot()` {.collection-method}

`undot` 方法會將使用「點 (dot)」符號的單維集合展開為多維集合：

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

`union` 方法會將給定的陣列新增到集合中。如果給定的陣列包含原始集合中已有的鍵 (key)，則原始集合的值將優先保留：

```php
$collection = collect([1 => ['a'], 2 => ['b']]);

$union = $collection->union([3 => ['c'], 1 => ['d']]);

$union->all();

// [1 => ['a'], 2 => ['b'], 3 => ['c']]
```

<a name="method-unique"></a>
#### `unique()` {.collection-method}

`unique` 方法會回傳集合中所有唯一的項目。回傳的集合會保留原始的陣列鍵，因此在以下範例中，我們將使用 [values](#method-values) 方法將鍵重設為連續的整數索引：

```php
$collection = collect([1, 1, 2, 2, 3, 4, 2]);

$unique = $collection->unique();

$unique->values()->all();

// [1, 2, 3, 4]
```

處理巢狀陣列或物件時，您可以指定用於判斷唯一性的鍵：

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

最後，您也可以向 `unique` 方法傳遞自己的閉包，以指定哪個值應該決定項目的唯一性：

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

`unique` 方法在檢查項目值時會使用「鬆散」比較，這表示包含整數值的字串會被視為與相同值的整數相等。若要使用「嚴格」比較進行篩選，請使用 [uniqueStrict](#method-uniquestrict) 方法。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-unique) 時，此方法的行為會有所不同。


<a name="method-uniquestrict"></a>
#### `uniqueStrict()` {.collection-method}

此方法與 [unique](#method-unique) 方法具有相同的簽名；但是，所有值都使用「嚴格」比較進行比較。


<a name="method-unless"></a>
#### `unless()` {.collection-method}

`unless` 方法會在傳入方法的第一個引數評估為 `true` 時，**不**執行給定的回呼。傳遞給 `unless` 方法的集合實例和第一個引數將提供給閉包：

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

可以向 `unless` 方法傳遞第二個回呼。當傳遞給 `unless` 方法的第一個引數評估為 `true` 時，將執行第二個回呼：

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

若要取得 `unless` 的反向操作，請參閱 [when](#method-when) 方法。


<a name="method-unlessempty"></a>
#### `unlessEmpty()` {.collection-method}

[whenNotEmpty](#method-whennotempty) 方法的別名。


<a name="method-unlessnotempty"></a>
#### `unlessNotEmpty()` {.collection-method}

[whenEmpty](#method-whenempty) 方法的別名。


<a name="method-unwrap"></a>
#### `unwrap()` {.collection-method}

靜態 `unwrap` 方法會在適用情況下，從給定值回傳集合的底層項目：

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

`value` 方法會從集合的第一個項目中擷取給定值：

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

`values` 方法會回傳一個新的集合，其鍵已重設為連續的整數：

```php
$collection = collect([
    10 => ['product' => 'Desk', 'price' => 200],
    11 => ['product' => 'Desk', 'price' => 200],
]);

$values = $collection->values();

$values->all();

/*
    [
        0 => ['product' => 'Desk', 'price' => 200],
        1 => ['product' => 'Desk', 'price' => 200],
    ]
*/
```


<a name="method-when"></a>
#### `when()` {.collection-method}

`when` 方法會在傳入方法的第一個引數評估為 `true` 時，執行給定的回呼。傳遞給 `when` 方法的集合實例和第一個引數將提供給閉包：

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

可以向 `when` 方法傳遞第二個回呼。當傳遞給 `when` 方法的第一個引數評估為 `false` 時，將執行第二個回呼：

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

若要取得 `when` 的反向操作，請參閱 [unless](#method-unless) 方法。


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

可以向 `whenEmpty` 方法傳遞第二個閉包，該閉包會在集合不為空時執行：

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

若要取得 `whenEmpty` 的反向操作，請參閱 [whenNotEmpty](#method-whennotempty) 方法。


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

可以向 `whenNotEmpty` 方法傳遞第二個閉包，該閉包會在集合為空時執行：

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

若要取得 `whenNotEmpty` 的反向操作，請參閱 [whenEmpty](#method-whenempty) 方法。

<a name="method-where"></a>
#### `where()` {.collection-method}

`where` 方法會根據給定的鍵/值對來篩選 Collection：

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

`where` 方法在檢查項目值時使用「鬆散」比對，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比對進行篩選，請使用 [whereStrict](#method-wherestrict) 方法；若要篩選 `null` 值，請使用 [whereNull](#method-wherenull) 和 [whereNotNull](#method-wherenotnull) 方法。

您也可以選擇將比較運算子作為第二個參數傳入。支援的運算子為：'===', '!==', '!=', '==', '=', '<>', '>', '<', '>=', 和 '<='：

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

此方法的簽名與 [where](#method-where) 方法相同；然而，所有值都使用「嚴格」比對進行比較。

<a name="method-wherebetween"></a>
#### `whereBetween()` {.collection-method}

`whereBetween` 方法透過判斷指定的項目值是否在給定範圍內來篩選 Collection：

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

`whereIn` 方法會移除 Collection 中指定項目值不在給定陣列內的元素：

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

`whereIn` 方法在檢查項目值時使用「鬆散」比對，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比對進行篩選，請使用 [whereInStrict](#method-whereinstrict) 方法。

<a name="method-whereinstrict"></a>
#### `whereInStrict()` {.collection-method}

此方法的簽名與 [whereIn](#method-wherein) 方法相同；然而，所有值都使用「嚴格」比對進行比較。

<a name="method-whereinstanceof"></a>
#### `whereInstanceOf()` {.collection-method}

`whereInstanceOf` 方法根據給定的類別型別篩選 Collection：

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

`whereNotBetween` 方法透過判斷指定的項目值是否在給定範圍外來篩選 Collection：

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

`whereNotIn` 方法會移除 Collection 中指定項目值包含在給定陣列內的元素：

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

`whereNotIn` 方法在檢查項目值時使用「鬆散」比對，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格」比對進行篩選，請使用 [whereNotInStrict](#method-wherenotinstrict) 方法。

<a name="method-wherenotinstrict"></a>
#### `whereNotInStrict()` {.collection-method}

此方法的簽名與 [whereNotIn](#method-wherenotin) 方法相同；然而，所有值都使用「嚴格」比對進行比較。

<a name="method-wherenotnull"></a>
#### `whereNotNull()` {.collection-method}

`whereNotNull` 方法會回傳 Collection 中給定鍵不是 `null` 的項目：

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

`whereNull` 方法會回傳 Collection 中給定鍵為 `null` 的項目：

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

靜態 `wrap` 方法在適用時將給定值包裝成 Collection：

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

`zip` 方法會將給定陣列的值與原始 Collection 中對應索引處的值合併：

```php
$collection = collect(['Chair', 'Desk']);

$zipped = $collection->zip([100, 200]);

$zipped->all();

// [['Chair', 100], ['Desk', 200]]
```

<a name="higher-order-messages"></a>
## 高階訊息

Collection 也支援「高階訊息 (higher order messages)」，這些是針對 Collection 執行常見操作的捷徑。提供高階訊息的 Collection 方法有：[average](#method-average)、[avg](#method-avg)、[contains](#method-contains)、[each](#method-each)、[every](#method-every)、[filter](#method-filter)、[first](#method-first)、[flatMap](#method-flatmap)、[groupBy](#method-groupby)、[keyBy](#method-keyby)、[map](#method-map)、[max](#method-max)、[min](#method-min)、[partition](#method-partition)、[reject](#method-reject)、[skipUntil](#method-skipuntil)、[skipWhile](#method-skipwhile)、[some](#method-some)、[sortBy](#method-sortby)、[sortByDesc](#method-sortbydesc)、[sum](#method-sum)、[takeUntil](#method-takeuntil)、[takeWhile](#method-takewhile) 和 [unique](#method-unique)。

每個高階訊息都可以作為 Collection 實例上的動態屬性來存取。例如，讓我們使用 `each` 高階訊息來呼叫 Collection 中每個物件上的方法：

```php
use App\Models\User;

$users = User::where('votes', '>', 500)->get();

$users->each->markAsVip();
```

同樣地，我們可以使用 `sum` 高階訊息來收集使用者 Collection 的所有「票數」總和：

```php
$users = User::where('group', 'Development')->get();

return $users->sum->votes;
```

<a name="lazy-collections"></a>
## 惰性集合


<a name="lazy-collection-introduction"></a>
### 簡介

> [!WARNING]
> 在深入了解 Laravel 的惰性集合之前，請花一些時間熟悉 [PHP 生成器](https://www.php.net/manual/en/language.generators.overview.php)。

為了補充已經強大的 `Collection` 類別，`LazyCollection` 類別利用 PHP 的 [生成器](https://www.php.net/manual/en/language.generators.overview.php) 來讓您處理非常龐大的資料集，同時保持記憶體使用率在低點。

例如，想像您的應用程式需要處理一個數 GB 大小的日誌檔，同時利用 Laravel 的集合方法來解析日誌。惰性集合可用來一次只在記憶體中保留一小部分檔案，而不是將整個檔案讀取到記憶體中：

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

或者，想像您需要迭代 10,000 個 Eloquent 模型。當使用傳統的 Laravel 集合時，所有 10,000 個 Eloquent 模型必須同時載入到記憶體中：

```php
use App\Models\User;

$users = User::all()->filter(function (User $user) {
    return $user->id > 500;
});
```

然而，查詢建構器的 `cursor` 方法會回傳一個 `LazyCollection` 實例。這讓您仍然只對資料庫執行單一查詢，但一次只在記憶體中保留一個 Eloquent 模型。在這個範例中，`filter` 回呼函式直到我們實際個別迭代每個使用者時才會執行，從而大幅減少記憶體使用量：

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
### 建立惰性集合

若要建立一個惰性集合實例，您應該將一個 PHP 生成器函式傳遞給集合的 `make` 方法：

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

幾乎所有 `Collection` 類別中可用的方法，在 `LazyCollection` 類別中也同樣可用。這兩個類別都實作了 `Illuminate\Support\Enumerable` 契約，該契約定義了以下方法：

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
> 會變動集合的方法 (例如 `shift`、`pop`、`prepend` 等) **不**適用於 `LazyCollection` 類別。

<a name="lazy-collection-methods"></a>
### 惰性集合方法

除了 `Enumerable` 契約中定義的方法外，`LazyCollection` 類別還包含以下方法：

<a name="method-takeUntilTimeout"></a>
#### `takeUntilTimeout()` {.collection-method}

`takeUntilTimeout` 方法會回傳一個新的惰性集合，該集合將列舉值直到指定時間。該時間過後，集合將停止列舉：

```php
$lazyCollection = LazyCollection::times(INF)
    ->takeUntilTimeout(now()->addMinute());

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

為了說明此方法的用法，想像一個應用程式使用 cursor 從資料庫提交發票。您可以定義一個 [排程任務](/docs/{{version}}/scheduling)，它每 15 分鐘執行一次，並且只處理最長 14 分鐘的發票：

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

雖然 `each` 方法會立即為集合中的每個項目呼叫給定的回呼，但 `tapEach` 方法只會在項目一個接一個地從列表中取出時才呼叫給定的回呼：

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

`throttle` 方法會限制惰性集合的頻率，使得每個值在指定的秒數後回傳。此方法在與對傳入請求進行頻率限制的外部 API 互動時特別有用：

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

`remember` 方法會回傳一個新的惰性集合，該集合將會記住所有已被列舉過的值，並且在後續的集合列舉中不會再次檢索它們：

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

`withHeartbeat` 方法允許您在惰性集合列舉期間，以固定時間間隔執行回呼。這對於需要定期維護任務 (例如延長鎖定或傳送進度更新) 的長時間執行操作特別有用：

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