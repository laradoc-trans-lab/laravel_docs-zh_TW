# Eloquent: 序列化

- [簡介](#introduction)
- [序列化模型與集合](#serializing-models-and-collections)
    - [序列化為陣列](#serializing-to-arrays)
    - [序列化為 JSON](#serializing-to-json)
- [在 JSON 中隱藏屬性](#hiding-attributes-from-json)
- [附加數值到 JSON](#appending-values-to-json)
- [日期序列化](#date-serialization)

<a name="introduction"></a>
## 簡介

使用 Laravel 建立 API 時，你經常需要將模型與關聯轉換為陣列或 JSON。Eloquent 包含了執行這些轉換的便利方法，並能控制哪些屬性應包含在模型的序列化表示層中。

> [!NOTE]
> 若要以更穩健的方式處理 Eloquent 模型與集合的 JSON 序列化，請參閱 [Eloquent API 資源](/docs/{{version}}/eloquent-resources) 的文件。


<a name="serializing-models-and-collections"></a>
## 序列化模型與集合


<a name="serializing-to-arrays"></a>
### 序列化為陣列

要將模型及其已載入的 [關聯](/docs/{{version}}/eloquent-relationships) 轉換為陣列，你應該使用 `toArray` 方法。此方法是遞迴的，因此所有屬性和所有關聯（包含關聯的關聯）都會被轉換為陣列：

```php
use App\Models\User;

$user = User::with('roles')->first();

return $user->toArray();
```

`attributesToArray` 方法可用於將模型的屬性轉換為陣列，但不包含其關聯：

```php
$user = User::first();

return $user->attributesToArray();
```

你也可以透過在集合實例上呼叫 `toArray` 方法，將整個模型的 [集合](/docs/{{version}}/eloquent-collections) 轉換為陣列：

```php
$users = User::all();

return $users->toArray();
```


<a name="serializing-to-json"></a>
### 序列化為 JSON

要將模型轉換為 JSON，你應該使用 `toJson` 方法。與 `toArray` 一樣，`toJson` 方法也是遞迴的，因此所有屬性和關聯都會被轉換為 JSON。你也可以指定任何 [PHP 支援](https://secure.php.net/manual/en/function.json-encode.php) 的 JSON 編碼選項：

```php
use App\Models\User;

$user = User::find(1);

return $user->toJson();

return $user->toJson(JSON_PRETTY_PRINT);
```

或者，你可以將模型或集合轉型為字串，這會自動呼叫模型或集合上的 `toJson` 方法：

```php
return (string) User::find(1);
```

由於模型和集合在轉型為字串時會轉換為 JSON，因此你可以直接從應用程式的路由或控制器回傳 Eloquent 物件。當從路由或控制器回傳時，Laravel 會自動將你的 Eloquent 模型與集合序列化為 JSON：

```php
Route::get('/users', function () {
    return User::all();
});
```


<a name="relationships"></a>
#### 關聯

當 Eloquent 模型被轉換為 JSON 時，其已載入的關聯將自動作為屬性包含在 JSON 物件中。此外，雖然 Eloquent 關聯方法是使用「駝峰式 (camel case)」方法名稱定義的，但關聯的 JSON 屬性將會是「蛇型命名 (snake case)」。


<a name="hiding-attributes-from-json"></a>
## 在 JSON 中隱藏屬性

有時你可能希望限制包含在模型陣列或 JSON 表示層中的屬性，例如密碼。為此，請在你的模型中加入 `$hidden` 屬性。列在 `$hidden` 屬性陣列中的屬性將不會包含在模型的序列化表示層中：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The attributes that should be hidden for serialization.
     *
     * @var array<string>
     */
    protected $hidden = ['password'];
}
```

> [!NOTE]
> 若要隱藏關聯，請將關聯的方法名稱加入到 Eloquent 模型的 `$hidden` 屬性中。

或者，你可以使用 `visible` 屬性來定義應包含在模型陣列與 JSON 表示層中的「允許清單」。當模型轉換為陣列或 JSON 時，所有不在 `$visible` 陣列中的屬性都將被隱藏：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The attributes that should be visible in arrays.
     *
     * @var array
     */
    protected $visible = ['first_name', 'last_name'];
}
```


<a name="temporarily-modifying-attribute-visibility"></a>
#### 暫時修改屬性可見性

如果你想在給定的模型實例上讓某些通常被隱藏的屬性變為可見，你可以使用 `makeVisible` 或 `mergeVisible` 方法。`makeVisible` 方法會回傳該模型實例：

```php
return $user->makeVisible('attribute')->toArray();

return $user->mergeVisible(['name', 'email'])->toArray();
```

同樣地，如果你想隱藏某些通常可見的屬性，你可以使用 `makeHidden` 或 `mergeHidden` 方法：

```php
return $user->makeHidden('attribute')->toArray();

return $user->mergeHidden(['name', 'email'])->toArray();
```

如果你希望暫時覆蓋所有的可見或隱藏屬性，可以分別使用 `setVisible` 和 `setHidden` 方法：

```php
return $user->setVisible(['id', 'name'])->toArray();

return $user->setHidden(['email', 'password', 'remember_token'])->toArray();
```


<a name="appending-values-to-json"></a>
## 附加數值到 JSON

有時，在將模型轉換為陣列或 JSON 時，你可能希望新增資料庫中沒有對應欄位的屬性。為此，請先為該值定義一個 [存取器 (accessor)](/docs/{{version}}/eloquent-mutators)：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Determine if the user is an administrator.
     */
    protected function isAdmin(): Attribute
    {
        return new Attribute(
            get: fn () => 'yes',
        );
    }
}
```

如果你希望存取器始終附加到模型的陣列和 JSON 表示層中，可以將屬性名稱加入到模型的 `appends` 屬性中。請注意，屬性名稱通常使用其「蛇型命名 (snake case)」的序列化表示來進行引用，即使存取器的 PHP 方法是使用「駝峰式 (camel case)」定義的：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The accessors to append to the model's array form.
     *
     * @var array
     */
    protected $appends = ['is_admin'];
}
```

一旦屬性被加入到 `appends` 清單中，它將同時包含在模型的陣列與 JSON 表示層中。`appends` 陣列中的屬性也將遵循模型上配置的 `visible` 與 `hidden` 設定。


<a name="appending-at-run-time"></a>
#### 執行時附加

在執行時，你可以使用 `append` 或 `mergeAppends` 方法指示模型實例附加額外的屬性。或者，你可以使用 `setAppends` 方法為給定的模型實例覆蓋整個附加屬性的陣列：

```php
return $user->append('is_admin')->toArray();

return $user->mergeAppends(['is_admin', 'status'])->toArray();

return $user->setAppends(['is_admin'])->toArray();
```

同樣地，如果你想從模型中移除所有附加屬性，可以使用 `withoutAppends` 方法：

```php
return $user->withoutAppends()->toArray();
```

<a name="date-serialization"></a>
## 日期序列化


<a name="customizing-the-default-date-format"></a>
#### 自定義預設日期格式

您可以透過覆寫 `serializeDate` 方法來定義預設的序列化格式。此方法不會影響日期在資料庫中儲存的格式：

```php
/**
 * Prepare a date for array / JSON serialization.
 */
protected function serializeDate(DateTimeInterface $date): string
{
    return $date->format('Y-m-d');
}
```


<a name="customizing-the-date-format-per-attribute"></a>
#### 自定義個別屬性的日期格式

您可以透過在模型的 [型別轉換宣告](/docs/{{version}}/eloquent-mutators#attribute-casting) 中指定日期格式，來定義個別 Eloquent 日期屬性的序列化格式：

```php
protected function casts(): array
{
    return [
        'birthday' => 'date:Y-m-d',
        'joined_at' => 'datetime:Y-m-d H:00',
    ];
}
```