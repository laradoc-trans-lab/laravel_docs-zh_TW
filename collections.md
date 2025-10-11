# Collections

- [簡介](#introduction)
    - [建立 Collections](#creating-collections)
    - [擴充 Collections](#extending-collections)
- [可用方法](#available-methods)
- [高階訊息](#higher-order-messages)
- [Lazy Collections](#lazy-collections)
    - [簡介](#lazy-collection-introduction)
    - [建立 Lazy Collections](#creating-lazy-collections)
    - [Enumerable 契約](#the-enumerable-contract)
    - [Lazy Collection 方法](#lazy-collection-methods)

<a name="introduction"></a>
## 簡介

`Illuminate\Support\Collection` 類別提供了一個流暢且便利的封裝，用於處理陣列資料。例如，請看以下程式碼。我們將使用 `collect` 輔助函式從陣列建立一個新的 Collection 實例，對每個元素執行 `strtoupper` 函式，然後移除所有空元素：

    $collection = collect(['taylor', 'abigail', null])->map(function (?string $name) {
        return strtoupper($name);
    })->reject(function (string $name) {
        return empty($name);
    });

如您所見，`Collection` 類別允許您鏈接其方法，以流暢地對底層陣列執行映射 (mapping) 和歸約 (reducing) 操作。一般而言，Collections 是不可變的，這意味著每個 `Collection` 方法都會回傳一個全新的 `Collection` 實例。

<a name="creating-collections"></a>
### 建立 Collections

如上所述，`collect` 輔助函式會為給定的陣列回傳一個新的 `Illuminate\Support\Collection` 實例。因此，建立 Collection 就像這樣簡單：

    $collection = collect([1, 2, 3]);

> [!NOTE]  
> [Eloquent](/docs/{{version}}/eloquent) 查詢的結果總是作為 `Collection` 實例回傳。

<a name="extending-collections"></a>
### 擴充 Collections

Collections 具有「巨集化 (macroable)」特性，這允許您在執行期間為 `Collection` 類別添加額外的方法。`Illuminate\Support\Collection` 類別的 `macro` 方法接受一個閉包 (closure)，該閉包將在呼叫您的巨集時執行。巨集閉包可以透過 `$this` 存取 Collection 的其他方法，就像它是一個真正的 Collection 類別方法一樣。例如，以下程式碼為 `Collection` 類別添加了一個 `toUpper` 方法：

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

通常，您應該在[服務提供者 (service provider)](/docs/{{version}}/providers) 的 `boot` 方法中宣告 Collection 巨集。

<a name="macro-arguments"></a>
#### 巨集引數

如有必要，您可以定義接受額外引數的巨集：

    use Illuminate\Support\Collection;
    use Illuminate\Support\Facades\Lang;

    Collection::macro('toLocale', function (string $locale) {
        return $this->map(function (string $value) use ($locale) {
            return Lang::get($value, [], $locale);
        });
    });

    $collection = collect(['first', 'second']);

    $translated = $collection->toLocale('es');

<a name="available-methods"></a>
## 可用方法

在 Collection 文件剩餘的大部分內容中，我們將討論 `Collection` 類別上的每個可用方法。請記住，所有這些方法都可以鏈接起來，以流暢地操作底層陣列。此外，幾乎每個方法都回傳一個新的 `Collection` 實例，允許您在必要時保留 Collection 的原始副本：

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

`after` 方法會回傳給定項目之後的項目。如果找不到給定項目或是它是最後一個項目，則會回傳 `null`：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->after(3);

    // 4

    $collection->after(5);

    // null

此方法使用「寬鬆 (loose)」比較來搜尋給定項目，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格 (strict)」比較，您可以為此方法提供 `strict` 引數：

    collect([2, 4, 6, 8])->after('4', strict: true);

    // null

或者，您可以提供自己的閉包 (closure) 來搜尋第一個通過指定真實性測試的項目：

    collect([2, 4, 6, 8])->after(function (int $item, int $key) {
        return $item > 5;
    });

    // 8


<a name="method-all"></a>
#### `all()` {.collection-method}

`all` 方法會回傳 Collection 所代表的底層陣列：

    collect([1, 2, 3])->all();

    // [1, 2, 3]


<a name="method-average"></a>
#### `average()` {.collection-method}

[`avg`](#method-avg) 方法的別名。


<a name="method-avg"></a>
#### `avg()` {.collection-method}

`avg` 方法會回傳給定鍵的[平均值](https://en.wikipedia.org/wiki/Average)：

    $average = collect([
        ['foo' => 10],
        ['foo' => 10],
        ['foo' => 20],
        ['foo' => 40]
    ])->avg('foo');

    // 20

    $average = collect([1, 1, 2, 4])->avg();

    // 2


<a name="method-before"></a>
#### `before()` {.collection-method}

`before` 方法與 [`after`](#method-after) 方法相反。它會回傳給定項目之前的項目。如果找不到給定項目或是它是第一個項目，則會回傳 `null`：

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


<a name="method-chunk"></a>
#### `chunk()` {.collection-method}

`chunk` 方法會將 Collection 分割成多個較小、指定大小的 Collection：

    $collection = collect([1, 2, 3, 4, 5, 6, 7]);

    $chunks = $collection->chunk(4);

    $chunks->all();

    // [[1, 2, 3, 4], [5, 6, 7]]

此方法在 [視圖](/docs/{{version}}/views) 中，搭配像 [Bootstrap](https://getbootstrap.com/docs/5.3/layout/grid/) 這類格線系統使用時特別有用。例如，想像您有一個 [Eloquent](/docs/{{version}}/eloquent) 模型 Collection，想在格線中顯示：

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

`chunkWhile` 方法會根據給定回呼的評估，將 Collection 分割成多個較小的 Collection。傳遞給閉包的 `$chunk` 變數可用來檢查前一個元素：

    $collection = collect(str_split('AABBCCCD'));

    $chunks = $collection->chunkWhile(function (string $value, int $key, Collection $chunk) {
        return $value === $chunk->last();
    });

    $chunks->all();

    // [['A', 'A'], ['B', 'B'], ['C', 'C', 'C'], ['D']]


<a name="method-collapse"></a>
#### `collapse()` {.collection-method}

`collapse` 方法會將陣列 Collection 壓縮成單一、扁平化的 Collection：

    $collection = collect([
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ]);

    $collapsed = $collection->collapse();

    $collapsed->all();

    // [1, 2, 3, 4, 5, 6, 7, 8, 9]


<a name="method-collapsewithkeys"></a>
#### `collapseWithKeys()` {.collection-method}

`collapseWithKeys` 方法會將陣列或 Collection 的 Collection 扁平化為單一 Collection，同時保留原始鍵：

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


<a name="method-collect"></a>
#### `collect()` {.collection-method}

`collect` 方法會回傳一個新的 `Collection` 實例，其中包含目前 Collection 中的項目：

    $collectionA = collect([1, 2, 3]);

    $collectionB = $collectionA->collect();

    $collectionB->all();

    // [1, 2, 3]

`collect` 方法主要用於將 [lazy collections](#lazy-collections) 轉換為標準 `Collection` 實例：

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

> [!NOTE]  
> `collect` 方法在您擁有 `Enumerable` 實例並需要一個非 Lazy Collection 實例時特別有用。由於 `collect()` 是 `Enumerable` 契約的一部分，您可以安全地使用它來取得 `Collection` 實例。


<a name="method-combine"></a>
#### `combine()` {.collection-method}

`combine` 方法會將 Collection 的值作為鍵，與另一個陣列或 Collection 的值組合起來：

    $collection = collect(['name', 'age']);

    $combined = $collection->combine(['George', 29]);

    $combined->all();

    // ['name' => 'George', 'age' => 29]


<a name="method-concat"></a>
#### `concat()` {.collection-method}

`concat` 方法會將給定的 `array` 或 Collection 的值附加到另一個 Collection 的末尾：

    $collection = collect(['John Doe']);

    $concatenated = $collection->concat(['Jane Doe'])->concat(['name' => 'Johnny Doe']);

    $concatenated->all();

    // ['John Doe', 'Jane Doe', 'Johnny Doe']

`concat` 方法會對附加到原始 Collection 的項目進行數值重新索引。若要保留關聯式 Collection 中的鍵，請參閱 [merge](#method-merge) 方法。


<a name="method-contains"></a>
#### `contains()` {.collection-method}

`contains` 方法會判斷 Collection 是否包含給定項目。您可以傳遞一個閉包給 `contains` 方法，以判斷 Collection 中是否存在符合指定真實性測試的元素：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->contains(function (int $value, int $key) {
        return $value > 5;
    });

    // false

或者，您可以傳遞一個字串給 `contains` 方法，以判斷 Collection 是否包含給定的項目值：

    $collection = collect(['name' => 'Desk', 'price' => 100]);

    $collection->contains('Desk');

    // true

    $collection->contains('New York');

    // false

您也可以傳遞一個鍵/值對給 `contains` 方法，它將判斷給定的鍵/值對是否存在於 Collection 中：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
    ]);

    $collection->contains('product', 'Bookcase');

    // false

`contains` 方法在檢查項目值時使用「寬鬆 (loose)」比較，這表示包含整數值的字串將被視為與相同值的整數相等。若要使用「嚴格 (strict)」比較進行篩選，請使用 [`containsStrict`](#method-containsstrict) 方法。

`contains` 的反向操作，請參閱 [doesntContain](#method-doesntcontain) 方法。

<a name="method-containsoneitem"></a>
#### `containsOneItem()` {.collection-method}

`containsOneItem` 方法判斷 collection 是否包含單一項目：

    collect([])->containsOneItem();

    // false

    collect(['1'])->containsOneItem();

    // true

    collect(['1', '2'])->containsOneItem();

    // false


<a name="method-containsstrict"></a>
#### `containsStrict()` {.collection-method}

此方法與 `[`contains`](#method-contains)` 方法簽名相同；然而，所有值都使用「嚴格」比較進行比較。

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-contains) 時，此方法的行為會被修改。


<a name="method-count"></a>
#### `count()` {.collection-method}

`count` 方法會回傳 collection 中項目的總數：

    $collection = collect([1, 2, 3, 4]);

    $collection->count();

    // 4


<a name="method-countBy"></a>
#### `countBy()` {.collection-method}

`countBy` 方法會計算 collection 中值的出現次數。預設情況下，此方法會計算每個元素的出現次數，讓您能夠計算 collection 中某些「類型」的元素：

    $collection = collect([1, 2, 2, 2, 3]);

    $counted = $collection->countBy();

    $counted->all();

    // [1 => 1, 2 => 3, 3 => 1]

您可以傳遞一個閉包 (closure) 給 `countBy` 方法，以透過自訂值來計算所有項目：

    $collection = collect(['alice@gmail.com', 'bob@yahoo.com', 'carlos@gmail.com']);

    $counted = $collection->countBy(function (string $email) {
        return substr(strrchr($email, "@"), 1);
    });

    $counted->all();

    // ['gmail.com' => 2, 'yahoo.com' => 1]


<a name="method-crossjoin"></a>
#### `crossJoin()` {.collection-method}

`crossJoin` 方法會將 collection 的值與給定陣列或 collections 進行交叉結合，回傳包含所有可能排列的笛卡兒積：

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


<a name="method-dd"></a>
#### `dd()` {.collection-method}

`dd` 方法會傾印 (dump) collection 的項目並結束腳本執行：

    $collection = collect(['John Doe', 'Jane Doe']);

    $collection->dd();

    /*
        Collection {
            #items: array:2 [
                0 => "John Doe"
                1 => "Jane Doe"
            ]
        }
    */

如果您不想停止腳本執行，請改用 `[`dump`](#method-dump)` 方法。


<a name="method-diff"></a>
#### `diff()` {.collection-method}

`diff` 方法會根據值來比較 collection 與另一個 collection 或純 PHP `array`。此方法會回傳原始 collection 中不存在於給定 collection 的值：

    $collection = collect([1, 2, 3, 4, 5]);

    $diff = $collection->diff([2, 4, 6, 8]);

    $diff->all();

    // [1, 3, 5]

> [!NOTE]
> 使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-diff) 時，此方法的行為會被修改。


<a name="method-diffassoc"></a>
#### `diffAssoc()` {.collection-method}

`diffAssoc` 方法會根據鍵和值來比較 collection 與另一個 collection 或純 PHP `array`。此方法會回傳原始 collection 中不存在於給定 collection 的鍵值對：

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


<a name="method-diffassocusing"></a>
#### `diffAssocUsing()` {.collection-method}

與 `diffAssoc` 不同，`diffAssocUsing` 接受使用者提供的回呼函數 (callback function) 用於索引比較：

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

回呼函數必須是一個比較函數，回傳值小於、等於或大於零的整數。更多資訊請參考 PHP 關於 `[`array_diff_uassoc`](https://www.php.net/array_diff_uassoc#refsect1-function.array-diff-uassoc-parameters)` 的文件，它是 `diffAssocUsing` 方法內部使用的 PHP 函數。


<a name="method-diffkeys"></a>
#### `diffKeys()` {.collection-method}

`diffKeys` 方法會根據鍵來比較 collection 與另一個 collection 或純 PHP `array`。此方法會回傳原始 collection 中不存在於給定 collection 的鍵值對：

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


<a name="method-doesntcontain"></a>
#### `doesntContain()` {.collection-method}

`doesntContain` 方法判斷 collection 是否不包含給定項目。您可以傳遞一個閉包 (closure) 給 `doesntContain` 方法，以判斷 collection 中是否存在不符合給定真值測試 (truth test) 的元素：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->doesntContain(function (int $value, int $key) {
        return $value < 5;
    });

    // false

或者，您可以傳遞一個字串給 `doesntContain` 方法，以判斷 collection 是否不包含給定項目值：

    $collection = collect(['name' => 'Desk', 'price' => 100]);

    $collection->doesntContain('Table');

    // true

    $collection->doesntContain('Desk');

    // false

您也可以傳遞一個鍵值對給 `doesntContain` 方法，它將判斷給定鍵值對是否存在於 collection 中：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
    ]);

    $collection->doesntContain('product', 'Bookcase');

    // true

`doesntContain` 方法在檢查項目值時使用「寬鬆」比較，這表示包含整數值的字串將被視為與具有相同值的整數相等。


<a name="method-dot"></a>
#### `dot()` {.collection-method}

`dot` 方法會將多維度的 collection 扁平化為單層 collection，並使用「點」表示法來指示深度：

    $collection = collect(['products' => ['desk' => ['price' => 100]]]);

    $flattened = $collection->dot();

    $flattened->all();

    // ['products.desk.price' => 100]


<a name="method-dump"></a>
#### `dump()` {.collection-method}

`dump` 方法會傾印 (dump) collection 的項目：

    $collection = collect(['John Doe', 'Jane Doe']);

    $collection->dump();

    /*
        Collection {
            #items: array:2 [
                0 => "John Doe"
                1 => "Jane Doe"
            ]
        }
    */

如果您想在傾印 (dump) collection 後停止執行腳本，請改用 `[`dd`](#method-dd)` 方法。

<a name="method-duplicates"></a>
#### `duplicates()` {.collection-method}

`duplicates` 方法會從 Collection 中擷取並回傳重複的值：

    $collection = collect(['a', 'b', 'a', 'c', 'b']);

    $collection->duplicates();

    // [2 => 'a', 4 => 'b']

若 Collection 包含陣列或物件，你可以傳入屬性的鍵 (key) 來檢查是否有重複的值：

    $employees = collect([
        ['email' => 'abigail@example.com', 'position' => 'Developer'],
        ['email' => 'james@example.com', 'position' => 'Designer'],
        ['email' => 'victoria@example.com', 'position' => 'Developer'],
    ]);

    $employees->duplicates('position');

    // [2 => 'Developer']


<a name="method-duplicatesstrict"></a>
#### `duplicatesStrict()` {.collection-method}

此方法與 [`duplicates`](#method-duplicates) 方法具有相同的簽名 (signature)；然而，所有的值都是使用「嚴格」比較來進行比較。


<a name="method-each"></a>
#### `each()` {.collection-method}

`each` 方法會迭代 Collection 中的項目，並將每個項目傳遞給一個閉包：

    $collection = collect([1, 2, 3, 4]);

    $collection->each(function (int $item, int $key) {
        // ...
    });

如果你想停止迭代項目，你可以從閉包中回傳 `false`：

    $collection->each(function (int $item, int $key) {
        if (/* condition */) {
            return false;
        }
    });


<a name="method-eachspread"></a>
#### `eachSpread()` {.collection-method}

`eachSpread` 方法會迭代 Collection 中的項目，並將每個巢狀項目值傳遞到給定的回呼 (callback) 中：

    $collection = collect([['John Doe', 35], ['Jane Doe', 33]]);

    $collection->eachSpread(function (string $name, int $age) {
        // ...
    });

你可以從回呼中回傳 `false` 來停止迭代項目：

    $collection->eachSpread(function (string $name, int $age) {
        return false;
    });


<a name="method-ensure"></a>
#### `ensure()` {.collection-method}

`ensure` 方法可用於驗證 Collection 的所有元素是否為給定的型別或型別列表。否則，將拋出 `UnexpectedValueException` 異常：

    return $collection->ensure(User::class);

    return $collection->ensure([User::class, Customer::class]);

也可以指定 `string`、`int`、`float`、`bool` 和 `array` 等基本型別：

    return $collection->ensure('int');

> [!WARNING]  
> `ensure` 方法不能保證日後不會將不同型別的元素添加到 Collection 中。


<a name="method-every"></a>
#### `every()` {.collection-method}

`every` 方法可用於驗證 Collection 的所有元素是否通過給定的真值測試：

    collect([1, 2, 3, 4])->every(function (int $value, int $key) {
        return $value > 2;
    });

    // false

若 Collection 為空，`every` 方法將回傳 true：

    $collection = collect([]);

    $collection->every(function (int $value, int $key) {
        return $value > 2;
    });

    // true


<a name="method-except"></a>
#### `except()` {.collection-method}

`except` 方法會回傳 Collection 中除指定鍵之外的所有項目：

    $collection = collect(['product_id' => 1, 'price' => 100, 'discount' => false]);

    $filtered = $collection->except(['price', 'discount']);

    $filtered->all();

    // ['product_id' => 1]

關於 `except` 的反向操作，請參閱 [only](#method-only) 方法。

> [!NOTE]  
> 當使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-except) 時，此方法的行為會被修改。


<a name="method-filter"></a>
#### `filter()` {.collection-method}

`filter` 方法使用給定的回呼過濾 Collection，只保留通過給定真值測試的項目：

    $collection = collect([1, 2, 3, 4]);

    $filtered = $collection->filter(function (int $value, int $key) {
        return $value > 2;
    });

    $filtered->all();

    // [3, 4]

如果沒有提供回呼，所有等同於 `false` 的 Collection 條目將會被移除：

    $collection = collect([1, 2, 3, null, false, '', 0, []]);

    $collection->filter()->all();

    // [1, 2, 3]

關於 `filter` 的反向操作，請參閱 [reject](#method-reject) 方法。


<a name="method-first"></a>
#### `first()` {.collection-method}

`first` 方法會回傳 Collection 中第一個通過給定真值測試的元素：

    collect([1, 2, 3, 4])->first(function (int $value, int $key) {
        return $value > 2;
    });

    // 3

你也可以不帶任何參數地呼叫 `first` 方法來取得 Collection 中的第一個元素。若 Collection 為空，則回傳 `null`：

    collect([1, 2, 3, 4])->first();

    // 1


<a name="method-first-or-fail"></a>
#### `firstOrFail()` {.collection-method}

`firstOrFail` 方法與 `first` 方法相同；然而，若沒有找到任何結果，將拋出 `Illuminate\Support\ItemNotFoundException` 異常：

    collect([1, 2, 3, 4])->firstOrFail(function (int $value, int $key) {
        return $value > 5;
    });

    // Throws ItemNotFoundException...

你也可以不帶任何參數地呼叫 `firstOrFail` 方法來取得 Collection 中的第一個元素。若 Collection 為空，將拋出 `Illuminate\Support\ItemNotFoundException` 異常：

    collect([])->firstOrFail();

    // Throws ItemNotFoundException...


<a name="method-first-where"></a>
#### `firstWhere()` {.collection-method}

`firstWhere` 方法會回傳 Collection 中第一個具有給定鍵 / 值配對的元素：

    $collection = collect([
        ['name' => 'Regena', 'age' => null],
        ['name' => 'Linda', 'age' => 14],
        ['name' => 'Diego', 'age' => 23],
        ['name' => 'Linda', 'age' => 84],
    ]);

    $collection->firstWhere('name', 'Linda');

    // ['name' => 'Linda', 'age' => 14]

你也可以使用比較運算子呼叫 `firstWhere` 方法：

    $collection->firstWhere('age', '>=', 18);

    // ['name' => 'Diego', 'age' => 23]

如同 [where](#method-where) 方法，你可以傳遞一個參數給 `firstWhere` 方法。在此情境下，`firstWhere` 方法將回傳給定項目鍵的值為「真值 (truthy)」的第一個項目：

    $collection->firstWhere('age');

    // ['name' => 'Linda', 'age' => 14]


<a name="method-flatmap"></a>
#### `flatMap()` {.collection-method}

`flatMap` 方法會迭代 Collection，並將每個值傳遞給給定的閉包。閉包可以自由修改項目並回傳，從而形成一個新的、經修改的項目 Collection。然後，該陣列會被扁平化一層：

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

<a name="method-flatten"></a>
#### `flatten()` {.collection-method}

`flatten` 方法將多維度的 Collection 扁平化為單一維度：

    $collection = collect([
        'name' => 'taylor',
        'languages' => [
            'php', 'javascript'
        ]
    ]);

    $flattened = $collection->flatten();

    $flattened->all();

    // ['taylor', 'php', 'javascript'];

如有需要，您可以將「深度 (depth)」引數傳遞給 `flatten` 方法：

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

在此範例中，不提供深度而呼叫 `flatten` 將會連同巢狀陣列一併扁平化，產生 `['iPhone 6S', 'Apple', 'Galaxy S7', 'Samsung']`。提供深度允許您指定將巢狀陣列扁平化的層級數量。


<a name="method-flip"></a>
#### `flip()` {.collection-method}

`flip` 方法將 Collection 的鍵與其對應的值交換：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    $flipped = $collection->flip();

    $flipped->all();

    // ['taylor' => 'name', 'laravel' => 'framework']


<a name="method-forget"></a>
#### `forget()` {.collection-method}

`forget` 方法依其鍵從 Collection 中移除項目：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    // Forget a single key...
    $collection->forget('name');

    // ['framework' => 'laravel']

    // Forget multiple keys...
    $collection->forget(['name', 'framework']);

    // []

> [!WARNING]  
> 與大多數其他 Collection 方法不同，`forget` 不會回傳一個新的、已修改的 Collection；它會修改並回傳被呼叫的 Collection 本身。


<a name="method-forpage"></a>
#### `forPage()` {.collection-method}

`forPage` 方法回傳一個新的 Collection，其中包含指定頁碼的項目。該方法接受頁碼作為第一個引數，以及每頁顯示的項目數量作為第二個引數：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9]);

    $chunk = $collection->forPage(2, 3);

    $chunk->all();

    // [4, 5, 6]


<a name="method-get"></a>
#### `get()` {.collection-method}

`get` 方法回傳指定鍵的項目。如果該鍵不存在，則回傳 `null`：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    $value = $collection->get('name');

    // taylor

您可以選擇性地傳遞一個預設值作為第二個引數：

    $collection = collect(['name' => 'taylor', 'framework' => 'laravel']);

    $value = $collection->get('age', 34);

    // 34

您甚至可以傳遞一個回呼作為方法的預設值。如果指定鍵不存在，則回傳該回呼的結果：

    $collection->get('email', function () {
        return 'taylor@example.com';
    });

    // taylor@example.com


<a name="method-groupby"></a>
#### `groupBy()` {.collection-method}

`groupBy` 方法依據給定鍵將 Collection 中的項目分組：

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

您可以傳遞一個回呼而不是字串 `key`。該回呼應回傳您希望作為分組鍵的值：

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

可以將多個分組條件以陣列形式傳遞。每個陣列元素將應用於多維陣列中對應的層級：

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


<a name="method-has"></a>
#### `has()` {.collection-method}

`has` 方法判斷 Collection 中是否存在給定鍵：

    $collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

    $collection->has('product');

    // true

    $collection->has(['product', 'amount']);

    // true

    $collection->has(['amount', 'price']);

    // false


<a name="method-hasany"></a>
#### `hasAny()` {.collection-method}

`hasAny` 方法判斷 Collection 中是否存在任一個給定鍵：

    $collection = collect(['account_id' => 1, 'product' => 'Desk', 'amount' => 5]);

    $collection->hasAny(['product', 'price']);

    // true

    $collection->hasAny(['name', 'price']);

    // false


<a name="method-implode"></a>
#### `implode()` {.collection-method}

`implode` 方法連接 Collection 中的項目。它的引數取決於 Collection 中項目的類型。如果 Collection 包含陣列或物件，您應該傳遞您希望連接的屬性鍵，以及您希望放置在值之間的「連接字串」：

    $collection = collect([
        ['account_id' => 1, 'product' => 'Desk'],
        ['account_id' => 2, 'product' => 'Chair'],
    ]);

    $collection->implode('product', ', ');

    // Desk, Chair

如果 Collection 包含簡單的字串或數值，您應該將「連接字串」作為唯一引數傳遞給該方法：

    collect([1, 2, 3, 4, 5])->implode('-');

    // '1-2-3-4-5'

如果您希望格式化要連接的值，您可以傳遞一個閉包給 `implode` 方法：

    $collection->implode(function (array $item, int $key) {
        return strtoupper($item['product']);
    }, ', ');

    // DESK, CHAIR

<a name="method-intersect"></a>
#### `intersect()` {.collection-method}

`intersect` 方法會移除原始 Collection 中不存在於給定 `array` 或 Collection 的任何值。結果的 Collection 將會保留原始 Collection 的鍵：

    $collection = collect(['Desk', 'Sofa', 'Chair']);

    $intersect = $collection->intersect(['Desk', 'Chair', 'Bookcase']);

    $intersect->all();

    // [0 => 'Desk', 2 => 'Chair']

> [!NOTE]
> 此方法的行為在使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-intersect) 時會有所修改。

<a name="method-intersectusing"></a>
#### `intersectUsing()` {.collection-method}

`intersectUsing` 方法會移除原始 Collection 中不存在於給定 `array` 或 Collection 的任何值，並使用自訂回呼函式來比較這些值。結果的 Collection 將會保留原始 Collection 的鍵：

    $collection = collect(['Desk', 'Sofa', 'Chair']);

    $intersect = $collection->intersectUsing(['desk', 'chair', 'bookcase'], function ($a, $b) {
        return strcasecmp($a, $b);
    });

    $intersect->all();

    // [0 => 'Desk', 2 => 'Chair']

<a name="method-intersectAssoc"></a>
#### `intersectAssoc()` {.collection-method}

`intersectAssoc` 方法會比較原始 Collection 與另一個 Collection 或 `array`，回傳所有給定 Collection 中都存在的鍵值對：

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

<a name="method-intersectassocusing"></a>
#### `intersectAssocUsing()` {.collection-method}

`intersectAssocUsing` 方法會比較原始 Collection 與另一個 Collection 或 `array`，回傳兩者中都存在的鍵值對，並使用自訂比較回呼函式來判斷鍵與值是否相等：

    $collection = collect([
        'color' => 'red',
        'Size' => 'M',
        'material' => 'cotton',
    ]);

    $intersect = $collection->intersectAssocUsing([
        'color' => 'blue',
        'size' => 'M',
        'material' => 'polyester',
    ], function ($a, $b) {
        return strcasecmp($a, $b);
    });

    $intersect->all();

    // ['Size' => 'M']

<a name="method-intersectbykeys"></a>
#### `intersectByKeys()` {.collection-method}

`intersectByKeys` 方法會移除原始 Collection 中不存在於給定 `array` 或 Collection 的任何鍵及其對應的值：

    $collection = collect([
        'serial' => 'UX301', 'type' => 'screen', 'year' => 2009,
    ]);

    $intersect = $collection->intersectByKeys([
        'reference' => 'UX404', 'type' => 'tab', 'year' => 2011,
    ]);

    $intersect->all();

    // ['type' => 'screen', 'year' => 2009]

<a name="method-isempty"></a>
#### `isEmpty()` {.collection-method}

`isEmpty` 方法會判斷 Collection 是否為空。如果為空，則回傳 `true`；否則回傳 `false`：

    collect([])->isEmpty();

    // true

<a name="method-isnotempty"></a>
#### `isNotEmpty()` {.collection-method}

`isNotEmpty` 方法會判斷 Collection 是否不為空。如果不為空，則回傳 `true`；否則回傳 `false`：

    collect([])->isNotEmpty();

    // false

<a name="method-join"></a>
#### `join()` {.collection-method}

`join` 方法會將 Collection 中的值以字串連接。使用此方法的第二個參數，您也可以指定最後一個元素應如何附加到字串中：

    collect(['a', 'b', 'c'])->join(', '); // 'a, b, c'
    collect(['a', 'b', 'c'])->join(', ', ', and '); // 'a, b, and c'
    collect(['a', 'b'])->join(', ', ' and '); // 'a and b'
    collect(['a'])->join(', ', ' and '); // 'a'
    collect([])->join(', ', ' and '); // ''

<a name="method-keyby"></a>
#### `keyBy()` {.collection-method}

`keyBy` 方法會根據給定的鍵來為 Collection 建立鍵。如果多個項目具有相同的鍵，則只有最後一個會出現在新的 Collection 中：

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

您也可以向方法傳遞一個回呼函式。此回呼函式應該回傳用於為 Collection 建立鍵的值：

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

<a name="method-keys"></a>
#### `keys()` {.collection-method}

`keys` 方法會回傳 Collection 的所有鍵：

    $collection = collect([
        'prod-100' => ['product_id' => 'prod-100', 'name' => 'Desk'],
        'prod-200' => ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]);

    $keys = $collection->keys();

    $keys->all();

    // ['prod-100', 'prod-200']

<a name="method-last"></a>
#### `last()` {.collection-method}

`last` 方法會回傳 Collection 中通過給定真值測試的最後一個元素：

    collect([1, 2, 3, 4])->last(function (int $value, int $key) {
        return $value < 3;
    });

    // 2

您也可以呼叫不帶參數的 `last` 方法，以取得 Collection 中的最後一個元素。如果 Collection 為空，則回傳 `null`：

    collect([1, 2, 3, 4])->last();

    // 4

<a name="method-lazy"></a>
#### `lazy()` {.collection-method}

`lazy` 方法會從底層的項目陣列中回傳一個新的 [`LazyCollection`](#lazy-collections) 實體：

    $lazyCollection = collect([1, 2, 3, 4])->lazy();

    $lazyCollection::class;

    // Illuminate\Support\LazyCollection

    $lazyCollection->all();

    // [1, 2, 3, 4]

這在您需要對包含大量項目的龐大 `Collection` 執行轉換時特別有用：

    $count = $hugeCollection
        ->lazy()
        ->where('country', 'FR')
        ->where('balance', '>', '100')
        ->count();

透過將 Collection 轉換為 `LazyCollection`，我們避免了分配大量額外的記憶體。儘管原始 Collection 仍將其值保留在記憶體中，但後續的篩選不會。因此，在篩選 Collection 結果時，幾乎不會分配額外的記憶體。

<a name="method-macro"></a>
#### `macro()` {.collection-method}

靜態 `macro` 方法允許您在執行時向 `Collection` 類別添加方法。有關更多資訊，請參閱 [擴充 Collections](#extending-collections) 的文件。

<a name="method-make"></a>
#### `make()` {.collection-method}

靜態 `make` 方法會建立一個新的 Collection 實體。請參閱 [建立 Collections](#creating-collections) 章節。

<a name="method-map"></a>
#### `map()` {.collection-method}

`map` 方法會遍歷 Collection，並將每個值傳遞給指定的回呼函式。回呼函式可以自由地修改項目並回傳它，從而形成一個新的已修改項目 Collection：

    $collection = collect([1, 2, 3, 4, 5]);

    $multiplied = $collection->map(function (int $item, int $key) {
        return $item * 2;
    });

    $multiplied->all();

    // [2, 4, 6, 8, 10]

> [!WARNING]
> 與大多數其他 Collection 方法一樣，`map` 會回傳一個新的 Collection 實體；它不會修改呼叫它的 Collection。如果您想要轉換原始 Collection，請使用 [`transform`](#method-transform) 方法。

<a name="method-mapinto"></a>
#### `mapInto()` {.collection-method}

`mapInto()` 方法會迭代 collection，透過將值傳入建構子來建立指定類別的新實例：

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


<a name="method-mapspread"></a>
#### `mapSpread()` {.collection-method}

`mapSpread` 方法會迭代 collection 中的項目，將每個巢狀項目值傳遞給指定的閉包。該閉包可以自由修改項目並回傳，藉此形成一個新的、經修改的項目 collection：

    $collection = collect([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

    $chunks = $collection->chunk(2);

    $sequence = $chunks->mapSpread(function (int $even, int $odd) {
        return $even + $odd;
    });

    $sequence->all();

    // [1, 5, 9, 13, 17]


<a name="method-maptogroups"></a>
#### `mapToGroups()` {.collection-method}

`mapToGroups` 方法會根據指定的閉包將 collection 的項目進行分組。該閉包應回傳一個包含單一鍵/值對的關聯陣列，藉此形成一個新的分組值 collection：

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


<a name="method-mapwithkeys"></a>
#### `mapWithKeys()` {.collection-method}

`mapWithKeys` 方法會迭代 collection 並將每個值傳遞給指定的 callback。該 callback 應回傳一個包含單一鍵/值對的關聯陣列：

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


<a name="method-max"></a>
#### `max()` {.collection-method}

`max` 方法會回傳指定鍵的最大值：

    $max = collect([
        ['foo' => 10],
        ['foo' => 20]
    ])->max('foo');

    // 20

    $max = collect([1, 2, 3, 4, 5])->max();

    // 5


<a name="method-median"></a>
#### `median()` {.collection-method}

`median` 方法會回傳指定鍵的 [中位數](https://en.wikipedia.org/wiki/Median)：

    $median = collect([
        ['foo' => 10],
        ['foo' => 10],
        ['foo' => 20],
        ['foo' => 40]
    ])->median('foo');

    // 15

    $median = collect([1, 1, 2, 4])->median();

    // 1.5


<a name="method-merge"></a>
#### `merge()` {.collection-method}

`merge` 方法會將指定的陣列或 collection 與原始 collection 合併。如果指定的項目中的字串鍵與原始 collection 中的字串鍵相符，則指定項目的值將會覆寫原始 collection 中的值：

    $collection = collect(['product_id' => 1, 'price' => 100]);

    $merged = $collection->merge(['price' => 200, 'discount' => false]);

    $merged->all();

    // ['product_id' => 1, 'price' => 200, 'discount' => false]

如果指定項目的鍵是數值，這些值將會被附加到 collection 的末尾：

    $collection = collect(['Desk', 'Chair']);

    $merged = $collection->merge(['Bookcase', 'Door']);

    $merged->all();

    // ['Desk', 'Chair', 'Bookcase', 'Door']


<a name="method-mergerecursive"></a>
#### `mergeRecursive()` {.collection-method}

`mergeRecursive` 方法會將指定的陣列或 collection 與原始 collection 進行遞迴合併。如果指定項目中的字串鍵與原始 collection 中的字串鍵相符，這些鍵的值將會合併成一個陣列，並遞迴執行此操作：

    $collection = collect(['product_id' => 1, 'price' => 100]);

    $merged = $collection->mergeRecursive([
        'product_id' => 2,
        'price' => 200,
        'discount' => false
    ]);

    $merged->all();

    // ['product_id' => [1, 2], 'price' => [100, 200], 'discount' => false]


<a name="method-min"></a>
#### `min()` {.collection-method}

`min` 方法會回傳指定鍵的最小值：

    $min = collect([['foo' => 10], ['foo' => 20]])->min('foo');

    // 10

    $min = collect([1, 2, 3, 4, 5])->min();

    // 1


<a name="method-mode"></a>
#### `mode()` {.collection-method}

`mode` 方法會回傳指定鍵的 [眾數](https://en.wikipedia.org/wiki/Mode_(statistics))：

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


<a name="method-multiply"></a>
#### `multiply()` {.collection-method}

`multiply` 方法會建立 collection 中所有指定數量項目的副本：

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

`nth` 方法會建立一個新的 collection，由每第 n 個元素組成：

    $collection = collect(['a', 'b', 'c', 'd', 'e', 'f']);

    $collection->nth(4);

    // ['a', 'e']

你可以選擇傳遞一個起始 offset 作為第二個參數：

    $collection->nth(4, 1);

    // ['b', 'f']


<a name="method-only"></a>
#### `only()` {.collection-method}

`only` 方法會回傳 collection 中具有指定鍵的項目：

    $collection = collect([
        'product_id' => 1,
        'name' => 'Desk',
        'price' => 100,
        'discount' => false
    ]);

    $filtered = $collection->only(['product_id', 'name']);

    $filtered->all();

    // ['product_id' => 1, 'name' => 'Desk']

對於 `only` 的反向操作，請參閱 [except](#method-except) 方法。

> [!NOTE]  
> 此方法的行為在使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-only) 時會有所不同。


<a name="method-pad"></a>
#### `pad()` {.collection-method}

`pad` 方法將使用指定的值填充陣列，直到陣列達到指定的大小。此方法行為類似於 PHP 的 [array_pad](https://secure.php.net/manual/en/function.array-pad.php) 函式。

若要向左填充，你應該指定一個負數大小。如果指定大小的絕對值小於或等於陣列的長度，將不會進行填充：

    $collection = collect(['A', 'B', 'C']);

    $filtered = $collection->pad(5, 0);

    $filtered->all();

    // ['A', 'B', 'C', 0, 0]

    $filtered = $collection->pad(-5, 0);

    $filtered->all();

    // [0, 0, 'A', 'B', 'C']

<a name="method-partition"></a>
#### `partition()` {.collection-method}

`partition` 方法可以與 PHP 陣列解構 (array destructuring) 結合使用，以將通過給定真值測試的元素與未通過的元素分開：

    $collection = collect([1, 2, 3, 4, 5, 6]);

    [$underThree, $equalOrAboveThree] = $collection->partition(function (int $i) {
        return $i < 3;
    });

    $underThree->all();

    // [1, 2]

    $equalOrAboveThree->all();

    // [3, 4, 5, 6]


<a name="method-percentage"></a>
#### `percentage()` {.collection-method}

`percentage` 方法可以用來快速判斷集合中通過給定真值測試的項目所佔的百分比：

```php
$collection = collect([1, 1, 2, 2, 2, 3]);

$percentage = $collection->percentage(fn ($value) => $value === 1);

// 33.33
```

預設情況下，百分比將四捨五入到小數點後兩位。不過，您可以透過為該方法提供第二個引數來客製化此行為：

```php
$percentage = $collection->percentage(fn ($value) => $value === 1, precision: 3);

// 33.333
```


<a name="method-pipe"></a>
#### `pipe()` {.collection-method}

`pipe` 方法將集合傳遞給給定的閉包，並返回執行閉包的結果：

    $collection = collect([1, 2, 3]);

    $piped = $collection->pipe(function (Collection $collection) {
        return $collection->sum();
    });

    // 6


<a name="method-pipeinto"></a>
#### `pipeInto()` {.collection-method}

`pipeInto` 方法會建立給定類別的新實例，並將集合傳遞給建構式：

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


<a name="method-pipethrough"></a>
#### `pipeThrough()` {.collection-method}

`pipeThrough` 方法將集合傳遞給給定的閉包陣列，並返回執行這些閉包的結果：

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


<a name="method-pluck"></a>
#### `pluck()` {.collection-method}

`pluck` 方法擷取給定鍵的所有值：

    $collection = collect([
        ['product_id' => 'prod-100', 'name' => 'Desk'],
        ['product_id' => 'prod-200', 'name' => 'Chair'],
    ]);

    $plucked = $collection->pluck('name');

    $plucked->all();

    // ['Desk', 'Chair']

您也可以指定希望結果集合如何被設為鍵：

    $plucked = $collection->pluck('name', 'product_id');

    $plucked->all();

    // ['prod-100' => 'Desk', 'prod-200' => 'Chair']

`pluck` 方法也支援使用「點」標記法來擷取巢狀值：

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

如果存在重複的鍵，最後一個匹配的元素將會被插入到擷取的集合中：

    $collection = collect([
        ['brand' => 'Tesla',  'color' => 'red'],
        ['brand' => 'Pagani', 'color' => 'white'],
        ['brand' => 'Tesla',  'color' => 'black'],
        ['brand' => 'Pagani', 'color' => 'orange'],
    ]);

    $plucked = $collection->pluck('color', 'brand');

    $plucked->all();

    // ['Tesla' => 'black', 'Pagani' => 'orange']


<a name="method-pop"></a>
#### `pop()` {.collection-method}

`pop` 方法會移除並回傳集合中的最後一個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->pop();

    // 5

    $collection->all();

    // [1, 2, 3, 4]

您可以傳遞一個整數給 `pop` 方法，以從集合的尾端移除並回傳多個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->pop(3);

    // collect([5, 4, 3])

    $collection->all();

    // [1, 2]


<a name="method-prepend"></a>
#### `prepend()` {.collection-method}

`prepend` 方法會在集合的開頭新增一個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->prepend(0);

    $collection->all();

    // [0, 1, 2, 3, 4, 5]

您也可以傳遞第二個引數來指定新增項目的鍵：

    $collection = collect(['one' => 1, 'two' => 2]);

    $collection->prepend(0, 'zero');

    $collection->all();

    // ['zero' => 0, 'one' => 1, 'two' => 2]


<a name="method-pull"></a>
#### `pull()` {.collection-method}

`pull` 方法會依據鍵值從集合中移除並回傳一個項目：

    $collection = collect(['product_id' => 'prod-100', 'name' => 'Desk']);

    $collection->pull('name');

    // 'Desk'

    $collection->all();

    // ['product_id' => 'prod-100']


<a name="method-push"></a>
#### `push()` {.collection-method}

`push` 方法會將一個項目附加到集合的尾端：

    $collection = collect([1, 2, 3, 4]);

    $collection->push(5);

    $collection->all();

    // [1, 2, 3, 4, 5]


<a name="method-put"></a>
#### `put()` {.collection-method}

`put` 方法會在集合中設定給定的鍵和值：

    $collection = collect(['product_id' => 1, 'name' => 'Desk']);

    $collection->put('price', 100);

    $collection->all();

    // ['product_id' => 1, 'name' => 'Desk', 'price' => 100]


<a name="method-random"></a>
#### `random()` {.collection-method}

`random` 方法會從集合中回傳一個隨機項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->random();

    // 4 - (retrieved randomly)

您可以傳遞一個整數給 `random` 方法，以指定要隨機擷取的項目數量。當明確傳遞您希望接收的項目數量時，總是會回傳一個項目的 Collection：

    $random = $collection->random(3);

    $random->all();

    // [2, 4, 5] - (retrieved randomly)

如果集合實例的項目數量少於請求的數量，`random` 方法將會拋出 `InvalidArgumentException`。

`random` 方法也接受一個閉包，該閉包將接收當前的集合實例：

    use Illuminate\Support\Collection;

    $random = $collection->random(fn (Collection $items) => min(10, count($items)));

    $random->all();

    // [1, 2, 3, 4, 5] - (retrieved randomly)


<a name="method-range"></a>
#### `range()` {.collection-method}

`range` 方法會回傳一個包含指定範圍內整數的集合：

    $collection = collect()->range(3, 6);

    $collection->all();

    // [3, 4, 5, 6]

<a name="method-reduce"></a>
#### `reduce()` {.collection-method}

`reduce` 方法將集合縮減為單一值，並將每次迭代的結果傳遞給下一次迭代：

    $collection = collect([1, 2, 3]);

    $total = $collection->reduce(function (?int $carry, int $item) {
        return $carry + $item;
    });

    // 6

`$carry` 在第一次迭代時的值為 `null`；不過，您可以透過向 `reduce` 傳遞第二個引數來指定其初始值：

    $collection->reduce(function (int $carry, int $item) {
        return $carry + $item;
    }, 4);

    // 10

`reduce` 方法也會將關聯式集合中的陣列鍵傳遞給指定的回呼：

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

    $collection->reduce(function (int $carry, int $value, int $key) use ($ratio) {
        return $carry + ($value * $ratio[$key]);
    });

    // 4264


<a name="method-reduce-spread"></a>
#### `reduceSpread()` {.collection-method}

`reduceSpread` 方法將集合縮減為一個值陣列，並將每次迭代的結果傳遞給下一次迭代。此方法與 `reduce` 方法相似；不過，它可以接受多個初始值：

    [$creditsRemaining, $batch] = Image::where('status', 'unprocessed')
        ->get()
        ->reduceSpread(function (int $creditsRemaining, Collection $batch, Image $image) {
            if ($creditsRemaining >= $image->creditsRequired()) {
                $batch->push($image);

                $creditsRemaining -= $image->creditsRequired();
            }

            return [$creditsRemaining, $batch];
        }, $creditsAvailable, collect());


<a name="method-reject"></a>
#### `reject()` {.collection-method}

`reject` 方法使用指定閉包來篩選集合。如果項目應從結果集合中移除，則閉包應回傳 `true`：

    $collection = collect([1, 2, 3, 4]);

    $filtered = $collection->reject(function (int $value, int $key) {
        return $value > 2;
    });

    $filtered->all();

    // [1, 2]

關於 `reject` 方法的反向操作，請參閱 [`filter`](#method-filter) 方法。


<a name="method-replace"></a>
#### `replace()` {.collection-method}

`replace` 方法的行為與 `merge` 相似；不過，除了覆寫具有字串鍵的相符項目外，`replace` 方法還會覆寫集合中具有相符數字鍵的項目：

    $collection = collect(['Taylor', 'Abigail', 'James']);

    $replaced = $collection->replace([1 => 'Victoria', 3 => 'Finn']);

    $replaced->all();

    // ['Taylor', 'Victoria', 'James', 'Finn']


<a name="method-replacerecursive"></a>
#### `replaceRecursive()` {.collection-method}

此方法的運作方式與 `replace` 相似，但它會遞迴到陣列中並將相同的替換程序應用於內部值：

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


<a name="method-reverse"></a>
#### `reverse()` {.collection-method}

`reverse` 方法會反轉集合中項目的順序，同時保留原始鍵：

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


<a name="method-search"></a>
#### `search()` {.collection-method}

`search` 方法會在集合中搜尋指定的值，如果找到，則回傳其鍵。如果找不到該項目，則回傳 `false`：

    $collection = collect([2, 4, 6, 8]);

    $collection->search(4);

    // 1

搜尋是使用「寬鬆」比較執行的，這表示包含整數值的字串將被視為與具有相同值的整數相等。若要使用「嚴格」比較，請將 `true` 作為第二個引數傳遞給該方法：

    collect([2, 4, 6, 8])->search('4', strict: true);

    // false

此外，您也可以提供自己的閉包來搜尋第一個通過指定真實性測試的項目：

    collect([2, 4, 6, 8])->search(function (int $item, int $key) {
        return $item > 5;
    });

    // 2


<a name="method-select"></a>
#### `select()` {.collection-method}

`select` 方法從集合中選取指定的鍵，類似於 SQL `SELECT` 語句：

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

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->shift();

    // 1

    $collection->all();

    // [2, 3, 4, 5]

您可以向 `shift` 方法傳遞一個整數，以從集合的開頭移除並回傳多個項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->shift(3);

    // collect([1, 2, 3])

    $collection->all();

    // [4, 5]


<a name="method-shuffle"></a>
#### `shuffle()` {.collection-method}

`shuffle` 方法會隨機打亂集合中的項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $shuffled = $collection->shuffle();

    $shuffled->all();

    // [3, 2, 5, 1, 4] - (generated randomly)


<a name="method-skip"></a>
#### `skip()` {.collection-method}

`skip` 方法會回傳一個新集合，其中移除了集合開頭指定數量的元素：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

    $collection = $collection->skip(4);

    $collection->all();

    // [5, 6, 7, 8, 9, 10]


<a name="method-skipuntil"></a>
#### `skipUntil()` {.collection-method}

`skipUntil` 方法會跳過集合中的項目，直到指定的回呼回傳 `false` 為止。一旦回呼回傳 `true`，集合中所有剩餘的項目都將作為一個新集合回傳：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->skipUntil(function (int $item) {
        return $item >= 3;
    });

    $subset->all();

    // [3, 4]

您也可以向 `skipUntil` 方法傳遞一個簡單的值，以跳過所有項目直到找到指定值：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->skipUntil(3);

    $subset->all();

    // [3, 4]

> [!WARNING]  
> 如果找不到指定的值或回呼從未回傳 `true`，則 `skipUntil` 方法將回傳一個空集合。


<a name="method-skipwhile"></a>
#### `skipWhile()` {.collection-method}

`skipWhile` 方法會跳過集合中的項目，直到指定的回呼回傳 `true` 為止。一旦回呼回傳 `false`，集合中所有剩餘的項目都將作為一個新集合回傳：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->skipWhile(function (int $item) {
        return $item <= 3;
    });

    $subset->all();

    // [4]

> [!WARNING]  
> 如果回呼從未回傳 `false`，則 `skipWhile` 方法將回傳一個空集合。

<a name="method-slice"></a>
#### `slice()` {.collection-method}

`slice` 方法會從指定索引處回傳 Collection 的切片：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

    $slice = $collection->slice(4);

    $slice->all();

    // [5, 6, 7, 8, 9, 10]

若要限制回傳切片的大小，可將所需大小作為第二個參數傳遞給方法：

    $slice = $collection->slice(4, 2);

    $slice->all();

    // [5, 6]

回傳的切片預設會保留鍵。若您不想保留原始鍵，可使用 [`values`](#method-values) 方法重新索引。

<a name="method-sliding"></a>
#### `sliding()` {.collection-method}

`sliding` 方法會回傳一個新的 Collection，其中包含代表 Collection 中項目「滑動視窗」檢視的區塊：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunks = $collection->sliding(2);

    $chunks->toArray();

    // [[1, 2], [2, 3], [3, 4], [4, 5]]

這在與 [`eachSpread`](#method-eachspread) 方法結合使用時特別有用：

    $transactions->sliding(2)->eachSpread(function (Collection $previous, Collection $current) {
        $current->total = $previous->total + $current->amount;
    });

您可以選擇傳遞第二個「step」值，該值決定每個區塊第一個項目之間的距離：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunks = $collection->sliding(3, step: 2);

    $chunks->toArray();

    // [[1, 2, 3], [3, 4, 5]]

<a name="method-sole"></a>
#### `sole()` {.collection-method}

`sole` 方法會回傳 Collection 中第一個通過給定真實性測試的元素，但僅限於該真實性測試只符合一個元素的情況：

    collect([1, 2, 3, 4])->sole(function (int $value, int $key) {
        return $value === 2;
    });

    // 2

您也可以傳遞鍵/值對到 `sole` 方法，這將回傳 Collection 中第一個符合給定對的元素，但僅限於它只符合一個元素的情況：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Chair', 'price' => 100],
    ]);

    $collection->sole('product', 'Chair');

    // ['product' => 'Chair', 'price' => 100]

或者，您也可以不帶參數呼叫 `sole` 方法，以在 Collection 中只有一個元素時取得第一個元素：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
    ]);

    $collection->sole();

    // ['product' => 'Desk', 'price' => 200]

如果 Collection 中沒有應由 `sole` 方法回傳的元素，則會拋出 `\Illuminate\Collections\ItemNotFoundException` 異常。如果有多個元素應被回傳，則會拋出 `\Illuminate\Collections\MultipleItemsFoundException`。

<a name="method-some"></a>
#### `some()` {.collection-method}

[`contains`](#method-contains) 方法的別名。

<a name="method-sort"></a>
#### `sort()` {.collection-method}

`sort` 方法會對 Collection 進行排序。排序後的 Collection 會保留原始的陣列鍵，因此在以下範例中，我們將使用 [`values`](#method-values) 方法將鍵重設為連續編號的索引：

    $collection = collect([5, 3, 1, 2, 4]);

    $sorted = $collection->sort();

    $sorted->values()->all();

    // [1, 2, 3, 4, 5]

如果您的排序需求更複雜，您可以傳遞一個帶有自己演算法的回呼函式給 `sort`。請參閱 PHP 關於 [`uasort`](https://secure.php.net/manual/en/function.uasort.php#refsect1-function.uasort-parameters) 的文件，這是 Collection 的 `sort` 方法內部所使用的 PHP 函式。

> [!NOTE]  
> 若您需要對巢狀陣列或物件的 Collection 進行排序，請參閱 [`sortBy`](#method-sortby) 與 [`sortByDesc`](#method-sortbydesc) 方法。

<a name="method-sortby"></a>
#### `sortBy()` {.collection-method}

`sortBy` 方法會根據給定的鍵對 Collection 進行排序。排序後的 Collection 會保留原始的陣列鍵，因此在以下範例中，我們將使用 [`values`](#method-values) 方法將鍵重設為連續編號的索引：

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

`sortBy` 方法接受 [sort flags](https://www.php.net/manual/en/function.sort.php) 作為其第二個參數：

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

或者，您可以傳遞自己的閉包來決定如何排序 Collection 的值：

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

如果您想依據多個屬性對 Collection 進行排序，您可以傳遞一個排序操作陣列給 `sortBy` 方法。每個排序操作都應是一個陣列，其中包含您想排序的屬性以及所需的排序方向：

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

當依據多個屬性對 Collection 進行排序時，您也可以提供定義每個排序操作的閉包：

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

<a name="method-sortbydesc"></a>
#### `sortByDesc()` {.collection-method}

此方法的簽章與 [`sortBy`](#method-sortby) 方法相同，但會以相反順序排序 Collection。

<a name="method-sortdesc"></a>
#### `sortDesc()` {.collection-method}

此方法會以與 [`sort`](#method-sort) 方法相反的順序排序 Collection：

    $collection = collect([5, 3, 1, 2, 4]);

    $sorted = $collection->sortDesc();

    $sorted->values()->all();

    // [5, 4, 3, 2, 1]

與 `sort` 不同，您不能傳遞閉包給 `sortDesc`。相反，您應該使用 [`sort`](#method-sort) 方法並反轉您的比較。

<a name="method-sortkeys"></a>
#### `sortKeys()` {.collection-method}

`sortKeys` 方法會依據底層關聯陣列的鍵來排序 collection：

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


<a name="method-sortkeysdesc"></a>
#### `sortKeysDesc()` {.collection-method}

這個方法與 [`sortKeys`](#method-sortkeys) 方法具有相同的簽名，但會以相反的順序來排序 collection。


<a name="method-sortkeysusing"></a>
#### `sortKeysUsing()` {.collection-method}

`sortKeysUsing` 方法會使用回呼函式，根據底層關聯陣列的鍵來排序 collection：

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

該回呼函式必須是比較函式，回傳小於、等於或大於零的整數。更多資訊，請參考 PHP 關於 [`uksort`](https://www.php.net/manual/en/function.uksort.php#refsect1-function.uksort-parameters) 的文件，`sortKeysUsing` 方法內部就是使用這個 PHP 函式。


<a name="method-splice"></a>
#### `splice()` {.collection-method}

`splice` 方法會移除並回傳從指定索引開始的一段項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunk = $collection->splice(2);

    $chunk->all();

    // [3, 4, 5]

    $collection->all();

    // [1, 2]

您可以傳遞第二個引數來限制結果 collection 的大小：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunk = $collection->splice(2, 1);

    $chunk->all();

    // [3]

    $collection->all();

    // [1, 2, 4, 5]

此外，您可以傳遞第三個引數，其中包含新項目，用來替換從 collection 中移除的項目：

    $collection = collect([1, 2, 3, 4, 5]);

    $chunk = $collection->splice(2, 1, [10, 11]);

    $chunk->all();

    // [3]

    $collection->all();

    // [1, 2, 10, 11, 4, 5]


<a name="method-split"></a>
#### `split()` {.collection-method}

`split` 方法會將 collection 分割成指定數量的群組：

    $collection = collect([1, 2, 3, 4, 5]);

    $groups = $collection->split(3);

    $groups->all();

    // [[1, 2], [3, 4], [5]]


<a name="method-splitin"></a>
#### `splitIn()` {.collection-method}

`splitIn` 方法會將 collection 分割成指定數量的群組，先完全填滿非最終群組，然後再將剩餘的項目分配給最終群組：

    $collection = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

    $groups = $collection->splitIn(3);

    $groups->all();

    // [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10]]


<a name="method-sum"></a>
#### `sum()` {.collection-method}

`sum` 方法會回傳 collection 中所有項目的總和：

    collect([1, 2, 3, 4, 5])->sum();

    // 15

如果 collection 包含巢狀陣列或物件，您應該傳遞一個鍵，用於決定要加總哪些值：

    $collection = collect([
        ['name' => 'JavaScript: The Good Parts', 'pages' => 176],
        ['name' => 'JavaScript: The Definitive Guide', 'pages' => 1096],
    ]);

    $collection->sum('pages');

    // 1272

此外，您可以傳遞自己的 closure 來決定要加總 collection 中的哪些值：

    $collection = collect([
        ['name' => 'Chair', 'colors' => ['Black']],
        ['name' => 'Desk', 'colors' => ['Black', 'Mahogany']],
        ['name' => 'Bookcase', 'colors' => ['Red', 'Beige', 'Brown']],
    ]);

    $collection->sum(function (array $product) {
        return count($product['colors']);
    });

    // 6


<a name="method-take"></a>
#### `take()` {.collection-method}

`take` 方法會回傳一個包含指定數量項目新 collection：

    $collection = collect([0, 1, 2, 3, 4, 5]);

    $chunk = $collection->take(3);

    $chunk->all();

    // [0, 1, 2]

您也可以傳遞一個負整數，從 collection 的尾部取出指定數量的項目：

    $collection = collect([0, 1, 2, 3, 4, 5]);

    $chunk = $collection->take(-2);

    $chunk->all();

    // [4, 5]


<a name="method-takeuntil"></a>
#### `takeUntil()` {.collection-method}

`takeUntil` 方法會回傳 collection 中的項目，直到給定的回呼函式回傳 `true`：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->takeUntil(function (int $item) {
        return $item >= 3;
    });

    $subset->all();

    // [1, 2]

您也可以傳遞一個簡單的值給 `takeUntil` 方法，以取得直到找到該值為止的項目：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->takeUntil(3);

    $subset->all();

    // [1, 2]

> [!WARNING]  
> 如果找不到給定的值，或回呼函式從未回傳 `true`，`takeUntil` 方法將回傳 collection 中的所有項目。


<a name="method-takewhile"></a>
#### `takeWhile()` {.collection-method}

`takeWhile` 方法會回傳 collection 中的項目，直到給定的回呼函式回傳 `false`：

    $collection = collect([1, 2, 3, 4]);

    $subset = $collection->takeWhile(function (int $item) {
        return $item < 3;
    });

    $subset->all();

    // [1, 2]

> [!WARNING]  
> 如果回呼函式從未回傳 `false`，`takeWhile` 方法將回傳 collection 中的所有項目。


<a name="method-tap"></a>
#### `tap()` {.collection-method}

`tap` 方法會將 collection 傳遞給指定的回呼函式，讓您可以「輕觸」collection 的特定點並對項目進行操作，同時不影響 collection 本身。然後 `tap` 方法會回傳 collection：

    collect([2, 4, 3, 1, 5])
        ->sort()
        ->tap(function (Collection $collection) {
            Log::debug('Values after sorting', $collection->values()->all());
        })
        ->shift();

    // 1


<a name="method-times"></a>
#### `times()` {.collection-method}

靜態 `times` 方法會透過呼叫給定的 closure 指定次數來建立一個新 collection：

    $collection = Collection::times(10, function (int $number) {
        return $number * 9;
    });

    $collection->all();

    // [9, 18, 27, 36, 45, 54, 63, 72, 81, 90]


<a name="method-toarray"></a>
#### `toArray()` {.collection-method}

`toArray` 方法會將 collection 轉換為一個普通的 PHP `array`。如果 collection 的值是 [Eloquent](/docs/{{version}}/eloquent) models，這些 models 也會被轉換為陣列：

    $collection = collect(['name' => 'Desk', 'price' => 200]);

    $collection->toArray();

    /*
        [
            ['name' => 'Desk', 'price' => 200],
        ]
    */

> [!WARNING]  
> `toArray` 也會將 collection 中所有實例化 `Arrayable` 的巢狀物件轉換為陣列。如果您想取得 collection 底層的原始陣列，請改用 [`all`](#method-all) 方法。


<a name="method-tojson"></a>
#### `toJson()` {.collection-method}

`toJson` 方法會將 collection 轉換為 JSON 序列化的字串：

    $collection = collect(['name' => 'Desk', 'price' => 200]);

    $collection->toJson();

    // '{"name":"Desk", "price":200}'


<a name="method-transform"></a>
#### `transform()` {.collection-method}

`transform` 方法會迭代 collection，並對 collection 中的每個項目呼叫給定的回呼函式。collection 中的項目將會被回呼函式回傳的值取代：

    $collection = collect([1, 2, 3, 4, 5]);

    $collection->transform(function (int $item, int $key) {
        return $item * 2;
    });

    $collection->all();

    // [2, 4, 6, 8, 10]

> [!WARNING]  
> 與大多數其他 collection 方法不同，`transform` 會修改 collection 本身。如果您希望建立一個新 collection，請使用 [`map`](#method-map) 方法。

<a name="method-undot"></a>
#### `undot()` {.collection-method}

`undot` 方法會將使用「點」表示法的單維度 Collection 展開成多維度的 Collection：

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

<a name="method-union"></a>
#### `union()` {.collection-method}

`union` 方法會將給定的陣列新增到 Collection 中。如果給定陣列中的鍵 (key) 已存在於原始 Collection 中，將優先使用原始 Collection 的值：

    $collection = collect([1 => ['a'], 2 => ['b']]);

    $union = $collection->union([3 => ['c'], 1 => ['d']]);

    $union->all();

    // [1 => ['a'], 2 => ['b'], 3 => ['c']]

<a name="method-unique"></a>
#### `unique()` {.collection-method}

`unique` 方法會回傳 Collection 中所有唯一的項目。回傳的 Collection 會保留原始陣列的鍵 (key)，因此在以下範例中，我們將使用 [`values`](#method-values) 方法將鍵重設為連續的數字索引：

    $collection = collect([1, 1, 2, 2, 3, 4, 2]);

    $unique = $collection->unique();

    $unique->values()->all();

    // [1, 2, 3, 4]

當處理巢狀陣列或物件時，您可以指定用於判斷唯一性的鍵：

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

最後，您也可以傳入自訂的閉包 (closure) 給 `unique` 方法，以指定哪個值應該判斷某個項目的唯一性：

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

`unique` 方法在檢查項目值時使用「寬鬆」比較 (loose comparison)，這表示具有整數值的字串會被視為與具有相同值的整數相等。若要使用「嚴格」比較 (strict comparison) 進行過濾，請使用 [`uniqueStrict`](#method-uniquestrict) 方法。

> [!NOTE]  
> 此方法的行為在使用 [Eloquent Collections](/docs/{{version}}/eloquent-collections#method-unique) 時會被修改。

<a name="method-uniquestrict"></a>
#### `uniqueStrict()` {.collection-method}

此方法與 [`unique`](#method-unique) 方法具有相同的簽章 (signature)；但是，所有值都使用「嚴格」比較 (strict comparison)。

<a name="method-unless"></a>
#### `unless()` {.collection-method}

`unless` 方法會在傳入方法的第一個引數評估為 `true` **除非** 執行給定的回呼 (callback)：

    $collection = collect([1, 2, 3]);

    $collection->unless(true, function (Collection $collection) {
        return $collection->push(4);
    });

    $collection->unless(false, function (Collection $collection) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 5]

可以傳入第二個回呼給 `unless` 方法。當傳入 `unless` 方法的第一個引數評估為 `true` 時，將會執行第二個回呼：

    $collection = collect([1, 2, 3]);

    $collection->unless(true, function (Collection $collection) {
        return $collection->push(4);
    }, function (Collection $collection) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 5]

關於 `unless` 的反向操作，請參閱 [`when`](#method-when) 方法。

<a name="method-unlessempty"></a>
#### `unlessEmpty()` {.collection-method}

[`whenNotEmpty`](#method-whennotempty) 方法的別名。

<a name="method-unlessnotempty"></a>
#### `unlessNotEmpty()` {.collection-method}

[`whenEmpty`](#method-whenempty) 方法的別名。

<a name="method-unwrap"></a>
#### `unwrap()` {.collection-method}

靜態 `unwrap` 方法在適用時會從給定值中回傳 Collection 的底層項目：

    Collection::unwrap(collect('John Doe'));

    // ['John Doe']

    Collection::unwrap(['John Doe']);

    // ['John Doe']

    Collection::unwrap('John Doe');

    // 'John Doe'

<a name="method-value"></a>
#### `value()` {.collection-method}

`value` 方法會從 Collection 的第一個元素中取得給定值：

    $collection = collect([
        ['product' => 'Desk', 'price' => 200],
        ['product' => 'Speaker', 'price' => 400],
    ]);

    $value = $collection->value('price');

    // 200

<a name="method-values"></a>
#### `values()` {.collection-method}

`values` 方法會回傳一個新的 Collection，且將鍵 (keys) 重設為連續的整數：

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

<a name="method-when"></a>
#### `when()` {.collection-method}

`when` 方法會在傳入方法的第一個引數評估為 `true` 時執行給定的回呼 (callback)。Collection 實例和傳入 `when` 方法的第一個引數將會傳給閉包 (closure)：

    $collection = collect([1, 2, 3]);

    $collection->when(true, function (Collection $collection, int $value) {
        return $collection->push(4);
    });

    $collection->when(false, function (Collection $collection, int $value) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 4]

可以傳入第二個回呼給 `when` 方法。當傳入 `when` 方法的第一個引數評估為 `false` 時，將會執行第二個回呼：

    $collection = collect([1, 2, 3]);

    $collection->when(false, function (Collection $collection, int $value) {
        return $collection->push(4);
    }, function (Collection $collection) {
        return $collection->push(5);
    });

    $collection->all();

    // [1, 2, 3, 5]

關於 `when` 的反向操作，請參閱 [`unless`](#method-unless) 方法。

<a name="method-whenempty"></a>
#### `whenEmpty()` {.collection-method}

`whenEmpty` 方法會在 Collection 為空時執行指定的回呼函式：

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

第二個閉包可以傳給 `whenEmpty` 方法，當 Collection 不為空時會執行該閉包：

    $collection = collect(['Michael', 'Tom']);

    $collection->whenEmpty(function (Collection $collection) {
        return $collection->push('Adam');
    }, function (Collection $collection) {
        return $collection->push('Taylor');
    });

    $collection->all();

    // ['Michael', 'Tom', 'Taylor']

關於 `whenEmpty` 的反向操作，請參考 [`whenNotEmpty`](#method-whennotempty) 方法。


<a name="method-whennotempty"></a>
#### `whenNotEmpty()` {.collection-method}

`whenNotEmpty` 方法會在 Collection 不為空時執行指定的回呼函式：

    $collection = collect(['michael', 'tom']);

    $collection->whenNotEmpty(function (Collection $collection) {
        return $collection->push('adam');
    });

    $collection->all();

    // ['michael', 'tom', 'adam']


    $collection = collect();

    $collection->whenNotEmpty(function (Collection $collection) {
        return $collection->push('adam');
    });

    $collection->all();

    // []

第二個閉包可以傳給 `whenNotEmpty` 方法，當 Collection 為空時會執行該閉包：

    $collection = collect();

    $collection->whenNotEmpty(function (Collection $collection) {
        return $collection->push('adam');
    }, function (Collection $collection) {
        return $collection->push('taylor');
    });

    $collection->all();

    // ['taylor']

關於 `whenNotEmpty` 的反向操作，請參考 [`whenEmpty`](#method-whenempty) 方法。


<a name="method-where"></a>
#### `where()` {.collection-method}

`where` 方法透過指定的鍵 / 值配對來篩選 Collection：

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

`where` 方法在檢查項目值時使用「寬鬆」比較，表示一個包含整數值的字串會被視為與相同值的整數相等。若要使用「嚴格」比較，請使用 [`whereStrict`](#method-wherestrict) 方法。

您可以選擇傳遞比較運算子作為第二個參數。支援的運算子有：'===', '!==', '!=', '==', '=', '<>', '>', '<', '>=', 和 '<='：

    $collection = collect([
        ['name' => 'Jim', 'deleted_at' => '2019-01-01 00:00:00'],
        ['name' => 'Sally', 'deleted_at' => '2019-01-02 00:00:00'],
        ['name' => 'Sue', 'deleted_at' => null],
    ]);

    $filtered = $collection->where('deleted_at', '!=', null);

    $filtered->all();

    /*
        [
            ['name' => 'Jim', 'deleted_at' => '2019-01-01 00:00:00'],
            ['name' => 'Sally', 'deleted_at' => '2019-01-02 00:00:00'],
        ]
    */


<a name="method-wherestrict"></a>
#### `whereStrict()` {.collection-method}

此方法與 [`where`](#method-where) 方法具有相同的簽名；然而，所有值都使用「嚴格」比較進行比較。


<a name="method-wherebetween"></a>
#### `whereBetween()` {.collection-method}

`whereBetween` 方法透過判斷指定項目值是否在給定範圍內來篩選 Collection：

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


<a name="method-wherein"></a>
#### `whereIn()` {.collection-method}

`whereIn` 方法從 Collection 中移除不具有包含在給定陣列中的指定項目值的元素：

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

`whereIn` 方法在檢查項目值時使用「寬鬆」比較，表示一個包含整數值的字串會被視為與相同值的整數相等。若要使用「嚴格」比較，請使用 [`whereInStrict`](#method-whereinstrict) 方法。


<a name="method-whereinstrict"></a>
#### `whereInStrict()` {.collection-method}

此方法與 [`whereIn`](#method-wherein) 方法具有相同的簽名；然而，所有值都使用「嚴格」比較進行比較。


<a name="method-whereinstanceof"></a>
#### `whereInstanceOf()` {.collection-method}

`whereInstanceOf` 方法透過給定的類別型別來篩選 Collection：

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


<a name="method-wherenotbetween"></a>
#### `whereNotBetween()` {.collection-method}

`whereNotBetween` 方法透過判斷指定項目值是否在給定範圍外來篩選 Collection：

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


<a name="method-wherenotin"></a>
#### `whereNotIn()` {.collection-method}

`whereNotIn` 方法從 Collection 中移除具有包含在給定陣列中的指定項目值的元素：

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

`whereNotIn` 方法在檢查項目值時使用「寬鬆」比較，表示一個包含整數值的字串會被視為與相同值的整數相等。若要使用「嚴格」比較，請使用 [`whereNotInStrict`](#method-wherenotinstrict) 方法。


<a name="method-wherenotinstrict"></a>
#### `whereNotInStrict()` {.collection-method}

此方法與 [`whereNotIn`](#method-wherenotin) 方法具有相同的簽名；然而，所有值都使用「嚴格」比較進行比較。

<a name="method-wherenotnull"></a>
#### `whereNotNull()` {.collection-method}

`whereNotNull` 方法會回傳 Collection 中指定鍵的值不為 `null` 的項目：

    $collection = collect([
        ['name' => 'Desk'],
        ['name' => null],
        ['name' => 'Bookcase'],
    ]);

    $filtered = $collection->whereNotNull('name');

    $filtered->all();

    /*
        [
            ['name' => 'Desk'],
            ['name' => 'Bookcase'],
        ]
    */


<a name="method-wherenull"></a>
#### `whereNull()` {.collection-method}

`whereNull` 方法會回傳 Collection 中指定鍵的值為 `null` 的項目：

    $collection = collect([
        ['name' => 'Desk'],
        ['name' => null],
        ['name' => 'Bookcase'],
    ]);

    $filtered = $collection->whereNull('name');

    $filtered->all();

    /*
        [
            ['name' => null],
        ]
    */


<a name="method-wrap"></a>
#### `wrap()` {.collection-method}

靜態 `wrap` 方法會將給定的值在適用情況下包裝為 Collection：

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


<a name="method-zip"></a>
#### `zip()` {.collection-method}

`zip` 方法會將給定陣列的值與原始 Collection 的值，依照它們的對應索引合併在一起：

    $collection = collect(['Chair', 'Desk']);

    $zipped = $collection->zip([100, 200]);

    $zipped->all();

    // [['Chair', 100], ['Desk', 200]]

<a name="higher-order-messages"></a>
## 高階訊息

Collection 也支援「高階訊息」，這是對 Collection 執行常見操作的捷徑。提供高階訊息的 Collection 方法包括：[`average`](#method-average)、[`avg`](#method-avg)、[`contains`](#method-contains)、[`each`](#method-each)、[`every`](#method-every)、[`filter`](#method-filter)、[`first`](#method-first)、[`flatMap`](#method-flatmap)、[`groupBy`](#method-groupby)、[`keyBy`](#method-keyby)、[`map`](#method-map)、[`max`](#method-max)、[`min`](#method-min)、[`partition`](#method-partition)、[`reject`](#method-reject)、[`skipUntil`](#method-skipuntil)、[`skipWhile`](#method-skipwhile)、[`some`](#method-some)、[`sortBy`](#method-sortby)、[`sortByDesc`](#method-sortbydesc)、[`sum`](#method-sum)、[`takeUntil`](#method-takeuntil)、[`takeWhile`](#method-takewhile) 和 [`unique`](#method-unique)。

每個高階訊息都可以作為 Collection 實例上的動態屬性來存取。例如，我們可以使用 `each` 高階訊息來呼叫 Collection 內的每個物件的方法：

    use App\Models\User;

    $users = User::where('votes', '>', 500)->get();

    $users->each->markAsVip();

同樣地，我們可以使用 `sum` 高階訊息來收集使用者的 Collection 的總「票數」：

    $users = User::where('group', 'Development')->get();

    return $users->sum->votes;

<a name="lazy-collections"></a>
## Lazy Collections

<a name="lazy-collection-introduction"></a>
### 簡介

> [!WARNING]  
> 在學習 Laravel 的 Lazy Collections 之前，請先花點時間熟悉 [PHP 生成器 (generators)](https://www.php.net/manual/en/language.generators.overview.php)。

為補充已然強大的 `Collection` 類別，`LazyCollection` 類別利用 PHP 的 [生成器 (generators)](https://www.php.net/manual/en/language.generators.overview.php)，讓您可以在處理非常大的資料集時保持低記憶體使用率。

舉例來說，假設您的應用程式需要處理一個數 GB 大小的日誌檔案，同時利用 Laravel 的 Collection 方法來解析日誌。Lazy Collections 可以用於每次只在記憶體中保留檔案的一小部分，而不是一次將整個檔案讀入記憶體：

    use App\Models\LogEntry;
    use Illuminate\Support\LazyCollection;

    LazyCollection::make(function () {
        $handle = fopen('log.txt', 'r');

        while (($line = fgets($handle)) !== false) {
            yield $line;
        }
    })->chunk(4)->map(function (array $lines) {
        return LogEntry::fromLines($lines);
    })->each(function (LogEntry $logEntry) {
        // Process the log entry...
    });

或者，想像您需要迭代 10,000 個 Eloquent 模型。當使用傳統的 Laravel Collections 時，所有 10,000 個 Eloquent 模型必須同時載入到記憶體中：

    use App\Models\User;

    $users = User::all()->filter(function (User $user) {
        return $user->id > 500;
    });

然而，查詢產生器 (query builder) 的 `cursor` 方法會回傳一個 `LazyCollection` 實例。這讓您仍然可以只對資料庫執行單一查詢，但每次只在記憶體中保留一個 Eloquent 模型。在這個範例中，`filter` 回呼函式 (callback) 在我們實際單獨迭代每個使用者之前不會執行，從而大幅減少記憶體使用量：

    use App\Models\User;

    $users = User::cursor()->filter(function (User $user) {
        return $user->id > 500;
    });

    foreach ($users as $user) {
        echo $user->id;
    }

<a name="creating-lazy-collections"></a>
### 建立 Lazy Collections

要建立 Lazy Collection 實例，您應該將 PHP 生成器函式傳遞給 Collection 的 `make` 方法：

    use Illuminate\Support\LazyCollection;

    LazyCollection::make(function () {
        $handle = fopen('log.txt', 'r');

        while (($line = fgets($handle)) !== false) {
            yield $line;
        }
    });

<a name="the-enumerable-contract"></a>
### Enumerable 契約

`Collection` 類別中幾乎所有可用的方法，在 `LazyCollection` 類別中也都可用。這兩個類別都實作了 `Illuminate\Support\Enumerable` 契約，該契約定義了以下方法：

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
> 會改變 Collection 的方法 (例如 `shift`、`pop`、`prepend` 等) **不**適用於 `LazyCollection` 類別。

<a name="lazy-collection-methods"></a>
### Lazy Collection 方法

除了 `Enumerable` 契約中定義的方法外，`LazyCollection` 類別還包含以下方法：

<a name="method-takeUntilTimeout"></a>
#### `takeUntilTimeout()` {.collection-method}

`takeUntilTimeout` 方法會回傳一個新的 Lazy Collection，它會列舉值直到指定時間為止。超過該時間後，Collection 將停止列舉：

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

為了說明此方法的用法，想像一個應用程式使用 cursor 從資料庫提交發票。您可以定義一個 [排程任務](/docs/{{version}}/scheduling)，每 15 分鐘執行一次，並且最多只處理 14 分鐘的發票：

    use App\Models\Invoice;
    use Illuminate\Support\Carbon;

    Invoice::pending()->cursor()
        ->takeUntilTimeout(
            Carbon::createFromTimestamp(LARAVEL_START)->add(14, 'minutes')
        )
        ->each(fn (Invoice $invoice) => $invoice->submit());

<a name="method-tapEach"></a>
#### `tapEach()` {.collection-method}

雖然 `each` 方法會立即為 Collection 中的每個項目呼叫給定的回呼，但 `tapEach` 方法僅在項目從列表中逐一取出時才呼叫給定的回呼：

    // Nothing has been dumped so far...
    $lazyCollection = LazyCollection::times(INF)->tapEach(function (int $value) {
        dump($value);
    });

    // Three items are dumped...
    $array = $lazyCollection->take(3)->all();

    // 1
    // 2
    // 3

<a name="method-throttle"></a>
#### `throttle()` {.collection-method}

`throttle` 方法將限制 Lazy Collection，使其在指定的秒數後才回傳每個值。此方法對於您可能與限制傳入請求速率的外部 API 互動的情況特別有用：

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

`remember` 方法會回傳一個新的 Lazy Collection，它將記住所有已列舉的值，並且在後續的 Collection 列舉中不會再次檢索它們：

    // No query has been executed yet...
    $users = User::cursor()->remember();

    // The query is executed...
    // The first 5 users are hydrated from the database...
    $users->take(5)->all();

    // First 5 users come from the collection's cache...
    // The rest are hydrated from the database...
    $users->take(20)->all();