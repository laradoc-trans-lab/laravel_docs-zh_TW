# HTTP 測試

- [簡介](#introduction)
- [發送請求](#making-requests)
    - [自定義請求標頭](#customizing-request-headers)
    - [Cookies](#cookies)
    - [Session / 身份驗證](#session-and-authentication)
    - [除錯回應](#debugging-responses)
    - [例外處理](#exception-handling)
- [測試 JSON APIs](#testing-json-apis)
    - [流式 JSON 測試](#fluent-json-testing)
- [測試檔案上傳](#testing-file-uploads)
- [測試視圖](#testing-views)
    - [渲染 Blade 與元件](#rendering-blade-and-components)
- [快取路由](#caching-routes)
- [可用的斷言](#available-assertions)
    - [回應斷言](#response-assertions)
    - [身份驗證斷言](#authentication-assertions)
    - [驗證斷言](#validation-assertions)

<a name="introduction"></a>
## 簡介

Laravel 提供了一個非常流暢的 API，用於向您的應用程式發送 HTTP 請求並檢查其回應。例如，請參考下方定義的功能測試：

```php tab=Pest
<?php

test('the application returns a successful response', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_the_application_returns_a_successful_response(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

`get` 方法會向應用程式發送一個 `GET` 請求，而 `assertStatus` 方法則會斷言回傳的回應應該具有指定的 HTTP 狀態碼。除了這個簡單的斷言之外，Laravel 還包含各種斷言，用於檢查回應標頭、內容、JSON 結構等。

<a name="making-requests"></a>
## 發送請求

要向您的應用程式發送請求，您可以在測試中調用 `get`、`post`、`put`、`patch` 或 `delete` 方法。這些方法實際上並未對您的應用程式發出「真實的」HTTP 請求。相反地，整個網路請求是在內部模擬的。

測試請求方法不會回傳 `Illuminate\Http\Response` 實例，而是回傳 `Illuminate\Testing\TestResponse` 的實例，它提供了[多種實用的斷言](#available-assertions)，讓您可以檢查應用程式的回應：

```php tab=Pest
<?php

test('basic request', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_a_basic_request(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

一般而言，您的每個測試應該只對應用程式發送一次請求。如果在單個測試方法中執行多個請求，可能會發生非預期的行為。

> [!NOTE]
> 為了方便起見，執行測試時會自動停用 CSRF 中介層。


<a name="customizing-request-headers"></a>
### 自定義請求標頭

您可以使用 `withHeaders` 方法在請求發送到應用程式之前自定義請求標頭。此方法允許您向請求添加任何您想要的自定義標頭：

```php tab=Pest
<?php

test('interacting with headers', function () {
    $response = $this->withHeaders([
        'X-Header' => 'Value',
    ])->post('/user', ['name' => 'Sally']);

    $response->assertStatus(201);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_interacting_with_headers(): void
    {
        $response = $this->withHeaders([
            'X-Header' => 'Value',
        ])->post('/user', ['name' => 'Sally']);

        $response->assertStatus(201);
    }
}
```


<a name="cookies"></a>
### Cookies

在發送請求之前，您可以使用 `withCookie` 或 `withCookies` 方法來設定 Cookie 值。`withCookie` 方法接受 Cookie 名稱和值作為其兩個參數，而 `withCookies` 方法接受一個名稱 / 值對的陣列：

```php tab=Pest
<?php

test('interacting with cookies', function () {
    $response = $this->withCookie('color', 'blue')->get('/');

    $response = $this->withCookies([
        'color' => 'blue',
        'name' => 'Taylor',
    ])->get('/');

    //
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_interacting_with_cookies(): void
    {
        $response = $this->withCookie('color', 'blue')->get('/');

        $response = $this->withCookies([
            'color' => 'blue',
            'name' => 'Taylor',
        ])->get('/');

        //
    }
}
```


<a name="session-and-authentication"></a>
### Session / 身份驗證

Laravel 提供了多個輔助函式，用於在 HTTP 測試期間與 Session 進行互動。首先，您可以使用 `withSession` 方法將 Session 資料設定為指定的陣列。這在發出請求到應用程式之前預先載入 Session 資料非常有用：

```php tab=Pest
<?php

test('interacting with the session', function () {
    $response = $this->withSession(['banned' => false])->get('/');

    //
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_interacting_with_the_session(): void
    {
        $response = $this->withSession(['banned' => false])->get('/');

        //
    }
}
```

Laravel 的 Session 通常用於維護目前已通過身份驗證的使用者的狀態。因此，`actingAs` 輔助方法提供了一種簡單的方法，將指定的使用者驗證為目前使用者。例如，我們可以使用 [model factory](/docs/{{version}}/eloquent-factories) 來產生並驗證一個使用者：

```php tab=Pest
<?php

use App\Models\User;

test('an action that requires authentication', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
        ->withSession(['banned' => false])
        ->get('/');

    //
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Models\User;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_an_action_that_requires_authentication(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->withSession(['banned' => false])
            ->get('/');

        //
    }
}
```

您也可以透過將 Guard 名稱作為 `actingAs` 方法的第二個參數，來指定用於驗證該使用者的 Guard。提供給 `actingAs` 方法的 Guard 也將在測試期間成為預設 Guard：

```php
$this->actingAs($user, 'web');
```

如果您想確保請求未通過身份驗證，可以使用 `actingAsGuest` 方法：

```php
$this->actingAsGuest();
```


<a name="debugging-responses"></a>
### 除錯回應

在向應用程式發送測試請求後，可以使用 `dump`、`dumpHeaders` 和 `dumpSession` 方法來檢查和除錯回應內容：

```php tab=Pest
<?php

test('basic test', function () {
    $response = $this->get('/');

    $response->dump();
    $response->dumpHeaders();
    $response->dumpSession();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_basic_test(): void
    {
        $response = $this->get('/');

        $response->dump();
        $response->dumpHeaders();
        $response->dumpSession();
    }
}
```

此外，您可以使用 `dd`、`ddHeaders`、`ddBody`、`ddJson` 和 `ddSession` 方法來傾印 (Dump) 關於回應的資訊並停止執行：

```php tab=Pest
<?php

test('basic test', function () {
    $response = $this->get('/');

    $response->dd();
    $response->ddHeaders();
    $response->ddBody();
    $response->ddJson();
    $response->ddSession();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_basic_test(): void
    {
        $response = $this->get('/');

        $response->dd();
        $response->ddHeaders();
        $response->ddBody();
        $response->ddJson();
        $response->ddSession();
    }
}
```

<a name="exception-handling"></a>
### 例外處理

有時您可能需要測試應用程式是否拋出了特定的例外。為了實現這一點，您可以透過 `Exceptions` Facade 夾帶「模擬 (fake)」例外處理程序。一旦例外處理程序被模擬，您就可以利用 `assertReported` 和 `assertNotReported` 方法對請求期間拋出的例外進行斷言：

```php tab=Pest
<?php

use App\Exceptions\InvalidOrderException;
use Illuminate\Support\Facades\Exceptions;

test('exception is thrown', function () {
    Exceptions::fake();

    $response = $this->get('/order/1');

    // Assert an exception was thrown...
    Exceptions::assertReported(InvalidOrderException::class);

    // Assert against the exception...
    Exceptions::assertReported(function (InvalidOrderException $e) {
        return $e->getMessage() === 'The order was invalid.';
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Exceptions\InvalidOrderException;
use Illuminate\Support\Facades\Exceptions;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_exception_is_thrown(): void
    {
        Exceptions::fake();

        $response = $this->get('/');

        // Assert an exception was thrown...
        Exceptions::assertReported(InvalidOrderException::class);

        // Assert against the exception...
        Exceptions::assertReported(function (InvalidOrderException $e) {
            return $e->getMessage() === 'The order was invalid.';
        });
    }
}
```

`assertNotReported` 和 `assertNothingReported` 方法可用於斷言在請求期間未拋出指定的例外，或未拋出任何例外：

```php
Exceptions::assertNotReported(InvalidOrderException::class);

Exceptions::assertNothingReported();
```

您可以在發送請求之前調用 `withoutExceptionHandling` 方法，完全停用特定請求的例外處理：

```php
$response = $this->withoutExceptionHandling()->get('/');
```

此外，如果您想確保您的應用程式沒有使用 PHP 語言或應用程式使用的函式庫已棄用的功能，您可以在發送請求之前調用 `withoutDeprecationHandling` 方法。當停用棄用處理時，棄用警告將會轉換為例外，從而導致您的測試失敗：

```php
$response = $this->withoutDeprecationHandling()->get('/');
```

`assertThrows` 方法可用於斷言指定 Closure 內的程式碼拋出了指定類型的例外：

```php
$this->assertThrows(
    fn () => (new ProcessOrder)->execute(),
    OrderInvalid::class
);
```

如果您想檢查並針對拋出的例外進行斷言，您可以提供一個 Closure 作為 `assertThrows` 方法的第二個參數：

```php
$this->assertThrows(
    fn () => (new ProcessOrder)->execute(),
    fn (OrderInvalid $e) => $e->orderId() === 123;
);
```

`assertDoesntThrow` 方法可用於斷言指定 Closure 內的程式碼不會拋出任何例外：

```php
$this->assertDoesntThrow(fn () => (new ProcessOrder)->execute());
```

<a name="testing-json-apis"></a>
## 測試 JSON APIs

Laravel 也提供了多種輔助函式來測試 JSON APIs 及其回應。例如，可以使用 `json`、`getJson`、`postJson`、`putJson`、`patchJson`、`deleteJson` 以及 `optionsJson` 方法來發送帶有各種 HTTP 動詞的 JSON 請求。您也可以輕鬆地將資料與標頭傳遞給這些方法。首先，讓我們編寫一個測試，向 `/api/user` 發送 `POST` 請求，並斷言傳回了預期的 JSON 資料：

```php tab=Pest
<?php

test('making an api request', function () {
    $response = $this->postJson('/api/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertJson([
            'created' => true,
        ]);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_making_an_api_request(): void
    {
        $response = $this->postJson('/api/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertJson([
                'created' => true,
            ]);
    }
}
```

此外，JSON 回應資料可以像存取陣列變數一樣在回應上進行存取，這讓您能夠方便地檢查 JSON 回應中傳回的個別值：

```php tab=Pest
expect($response['created'])->toBeTrue();
```

```php tab=PHPUnit
$this->assertTrue($response['created']);
```

> [!NOTE]
> `assertJson` 方法會將回應轉換為陣列，以驗證指定的陣列是否存在於應用程式傳回的 JSON 回應中。因此，只要指定的片段存在，即使 JSON 回應中還有其他屬性，此測試仍會通過。


<a name="verifying-exact-match"></a>
#### 斷言 JSON 完全匹配

如前所述，`assertJson` 方法可用於斷言 JSON 片段存在於 JSON 回應中。如果您想驗證指定的陣列是否與應用程式傳回的 JSON **完全匹配**，則應使用 `assertExactJson` 方法：

```php tab=Pest
<?php

test('asserting an exact json match', function () {
    $response = $this->postJson('/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertExactJson([
            'created' => true,
        ]);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_asserting_an_exact_json_match(): void
    {
        $response = $this->postJson('/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertExactJson([
                'created' => true,
            ]);
    }
}
```


<a name="verifying-json-paths"></a>
#### 斷言 JSON 路徑

如果您想驗證 JSON 回應在指定路徑中是否包含指定的資料，則應使用 `assertJsonPath` 方法：

```php tab=Pest
<?php

test('asserting a json path value', function () {
    $response = $this->postJson('/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertJsonPath('team.owner.name', 'Darian');
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_asserting_a_json_paths_value(): void
    {
        $response = $this->postJson('/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertJsonPath('team.owner.name', 'Darian');
    }
}
```

`assertJsonPath` 方法也接受一個閉包，可用於動態判斷該斷言是否應通過：

```php
$response->assertJsonPath('team.owner.name', fn (string $name) => strlen($name) >= 3);
```

<a name="fluent-json-testing"></a>
### 流式 JSON 測試

Laravel 還提供了一種優雅的方式來流暢地測試應用程式的 JSON 回應。首先，傳遞一個閉包給 `assertJson` 方法。此閉包將會接收一個 `Illuminate\Testing\Fluent\AssertableJson` 實例，該實例可用於對應用程式回傳的 JSON 進行斷言。`where` 方法可用於對 JSON 的特定屬性進行斷言，而 `missing` 方法則可用於斷言 JSON 中缺少特定屬性：

```php tab=Pest
use Illuminate\Testing\Fluent\AssertableJson;

test('fluent json', function () {
    $response = $this->getJson('/users/1');

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->where('id', 1)
                ->where('name', 'Victoria Faith')
                ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                ->whereNot('status', 'pending')
                ->missing('password')
                ->etc()
        );
});
```

```php tab=PHPUnit
use Illuminate\Testing\Fluent\AssertableJson;

/**
 * A basic functional test example.
 */
public function test_fluent_json(): void
{
    $response = $this->getJson('/users/1');

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->where('id', 1)
                ->where('name', 'Victoria Faith')
                ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                ->whereNot('status', 'pending')
                ->missing('password')
                ->etc()
        );
}
```


#### 理解 `etc` 方法

在上述範例中，您可能已經注意到我們在斷言鏈的末端調用了 `etc` 方法。此方法告知 Laravel，JSON 物件中可能還存在其他屬性。如果未使用 `etc` 方法，當 JSON 物件中存在您未進行斷言的其他屬性時，測試將會失敗。

此行為的目的是為了防止您無意中在 JSON 回應中洩露敏感資訊，方法是強制您必須明確對屬性進行斷言，或者透過 `etc` 方法明確允許額外的屬性。

然而，您應該注意，在斷言鏈中不包含 `etc` 方法並不能確保不會在 JSON 物件的嵌套陣列中加入額外屬性。`etc` 方法僅確保在調用 `etc` 方法的該嵌套層級中不存在額外屬性。


<a name="asserting-json-attribute-presence-and-absence"></a>
#### 斷言屬性存在 / 缺失

若要斷言屬性是否存在，可以使用 `has` 和 `missing` 方法：

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->has('data')
        ->missing('message')
);
```

此外，`hasAll` 和 `missingAll` 方法允許同時斷言多個屬性的存在與否：

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->hasAll(['status', 'data'])
        ->missingAll(['message', 'code'])
);
```

您可以使用 `hasAny` 方法來判斷給定屬性列表中是否至少存在一個屬性：

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->has('status')
        ->hasAny('data', 'message', 'code')
);
```


<a name="asserting-against-json-collections"></a>
#### 對 JSON 集合進行斷言

通常，您的路由會回傳一個包含多個項目的 JSON 回應，例如多個使用者：

```php
Route::get('/users', function () {
    return User::all();
});
```

在這些情況下，我們可以使用流式 JSON 物件的 `has` 方法來對回應中包含的使用者進行斷言。例如，讓我們斷言 JSON 回應包含三個使用者。接著，我們將使用 `first` 方法對集合中的第一個使用者進行一些斷言。`first` 方法接受一個閉包，該閉包會接收另一個可斷言的 JSON 字串，我們可以用它來對 JSON 集合中的第一個物件進行斷言：

```php
$response
    ->assertJson(fn (AssertableJson $json) =>
        $json->has(3)
            ->first(fn (AssertableJson $json) =>
                $json->where('id', 1)
                    ->where('name', 'Victoria Faith')
                    ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                    ->missing('password')
                    ->etc()
            )
    );
```


<a name="scoping-json-collection-assertions"></a>
#### 限定 JSON 集合斷言範圍

有時，您的應用程式路由會回傳被分配了命名鍵值的 JSON 集合：

```php
Route::get('/users', function () {
    return [
        'meta' => [...],
        'users' => User::all(),
    ];
})
```

在測試這些路由時，您可以使用 `has` 方法來斷言集合中的項目數量。此外，您可以使用 `has` 方法來限定斷言鏈的範圍：

```php
$response
    ->assertJson(fn (AssertableJson $json) =>
        $json->has('meta')
            ->has('users', 3)
            ->has('users.0', fn (AssertableJson $json) =>
                $json->where('id', 1)
                    ->where('name', 'Victoria Faith')
                    ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                    ->missing('password')
                    ->etc()
            )
    );
```

然而，與其兩次分別調用 `has` 方法來對 `users` 集合進行斷言，您可以只調用一次並提供一個閉包作為其第三個參數。這樣做時，該閉包將會自動被調用，並將範圍限定在集合中的第一個項目：

```php
$response
    ->assertJson(fn (AssertableJson $json) =>
        $json->has('meta')
            ->has('users', 3, fn (AssertableJson $json) =>
                $json->where('id', 1)
                    ->where('name', 'Victoria Faith')
                    ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                    ->missing('password')
                    ->etc()
            )
    );
```


<a name="asserting-json-types"></a>
#### 斷言 JSON 類型

您可能只想斷言 JSON 回應中的屬性是否為特定類型。`Illuminate\Testing\Fluent\AssertableJson` 類別提供了 `whereType` 和 `whereAllType` 方法來實現此目的：

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->whereType('id', 'integer')
        ->whereAllType([
            'users.0.name' => 'string',
            'meta' => 'array'
        ])
);
```

您可以使用 `|` 字元指定多個類型，或者將類型陣列作為第二個參數傳遞給 `whereType` 方法。如果回應的值是所列類型中的任何一種，斷言就會成功：

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->whereType('name', 'string|null')
        ->whereType('id', ['string', 'integer'])
);
```

`whereType` 和 `whereAllType` 方法識別以下類型：`string`、`integer`、`double`、`boolean`、`array` 和 `null`。

<a name="testing-file-uploads"></a>
## 測試檔案上傳

`Illuminate\Http\UploadedFile` 類別提供了一個 `fake` 方法，可用於產生測試用的虛擬檔案或圖片。這與 `Storage` Facade 的 `fake` 方法結合使用，大幅簡化了檔案上傳的測試。例如，您可以結合這兩個功能來輕鬆測試頭像上傳表單：

```php tab=Pest
<?php

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

test('avatars can be uploaded', function () {
    Storage::fake('avatars');

    $file = UploadedFile::fake()->image('avatar.jpg');

    $response = $this->post('/avatar', [
        'avatar' => $file,
    ]);

    Storage::disk('avatars')->assertExists($file->hashName());
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_avatars_can_be_uploaded(): void
    {
        Storage::fake('avatars');

        $file = UploadedFile::fake()->image('avatar.jpg');

        $response = $this->post('/avatar', [
            'avatar' => $file,
        ]);

        Storage::disk('avatars')->assertExists($file->hashName());
    }
}
```

如果您想斷言某個檔案不存在，可以使用 `Storage` Facade 提供的 `assertMissing` 方法：

```php
Storage::fake('avatars');

// ...

Storage::disk('avatars')->assertMissing('missing.jpg');
```

<a name="fake-file-customization"></a>
#### 偽造檔案自定義

使用 `UploadedFile` 類別提供的 `fake` 方法建立檔案時，您可以指定圖片的寬度、高度和大小（以 KB 為單位），以便更好地測試應用程式的驗證規則：

```php
UploadedFile::fake()->image('avatar.jpg', $width, $height)->size(100);
```

除了建立圖片，您還可以使用 `create` 方法建立任何其他類型的檔案：

```php
UploadedFile::fake()->create('document.pdf', $sizeInKilobytes);
```

如果需要，您可以向該方法傳遞 `$mimeType` 參數，以明確定義檔案應回傳的 MIME 類型：

```php
UploadedFile::fake()->create(
    'document.pdf', $sizeInKilobytes, 'application/pdf'
);
```

<a name="testing-views"></a>
## 測試視圖

Laravel 還允許您在不發送模擬 HTTP 請求的情況下渲染視圖。要做到這一點，您可以在測試中呼叫 `view` 方法。`view` 方法接受視圖名稱和一個選用的資料陣列。該方法回傳一個 `Illuminate\Testing\TestView` 實例，它提供了多種方法來方便地對視圖內容進行斷言：

```php tab=Pest
<?php

test('a welcome view can be rendered', function () {
    $view = $this->view('welcome', ['name' => 'Taylor']);

    $view->assertSee('Taylor');
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_a_welcome_view_can_be_rendered(): void
    {
        $view = $this->view('welcome', ['name' => 'Taylor']);

        $view->assertSee('Taylor');
    }
}
```

`TestView` 類別提供了以下斷言方法：`assertSee`、`assertSeeInOrder`、`assertSeeText`、`assertSeeTextInOrder`、`assertDontSee` 和 `assertDontSeeText`。

如果需要，您可以透過將 `TestView` 實例轉換為字串來獲取原始的、已渲染的視圖內容：

```php
$contents = (string) $this->view('welcome');
```

<a name="sharing-errors"></a>
#### 共享錯誤

某些視圖可能依賴於 [Laravel 提供的全域錯誤包 (Error Bag)](/docs/{{version}}/validation#quick-displaying-the-validation-errors) 中共享的錯誤。要使用錯誤訊息填充錯誤包，您可以使用 `withViewErrors` 方法：

```php
$view = $this->withViewErrors([
    'name' => ['Please provide a valid name.']
])->view('form');

$view->assertSee('Please provide a valid name.');
```

<a name="rendering-blade-and-components"></a>
### 渲染 Blade 與元件

如有必要，您可以使用 `blade` 方法來解析並渲染原始的 [Blade](/docs/{{version}}/blade) 字串。與 `view` 方法一樣，`blade` 方法回傳 `Illuminate\Testing\TestView` 實例：

```php
$view = $this->blade(
    '<x-component :name="$name" />',
    ['name' => 'Taylor']
);

$view->assertSee('Taylor');
```

您可以使用 `component` 方法來解析並渲染 [Blade 元件](/docs/{{version}}/blade#components)。`component` 方法回傳一個 `Illuminate\Testing\TestComponent` 實例：

```php
$view = $this->component(Profile::class, ['name' => 'Taylor']);

$view->assertSee('Taylor');
```

<a name="caching-routes"></a>
## 快取路由

在測試運行之前，Laravel 會啟動一個全新的應用程式實例，包括收集所有定義的路由。如果您的應用程式有許多路由檔案，您可能希望將 `Illuminate\Foundation\Testing\WithCachedRoutes` Trait 加入您的測試案例中。在使用了這個 Trait 的測試中，路由只會建立一次並儲存在記憶體中，這意味著路由收集過程在整個測試套件中僅運行一次：

```php tab=Pest
<?php

use App\Http\Controllers\UserController;
use Illuminate\Foundation\Testing\WithCachedRoutes;

pest()->use(WithCachedRoutes::class);

test('basic example', function () {
    $this->get(action([UserController::class, 'index']));

    // ...
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Http\Controllers\UserController;
use Illuminate\Foundation\Testing\WithCachedRoutes;
use Tests\TestCase;

class BasicTest extends TestCase
{
    use WithCachedRoutes;

    /**
     * A basic functional test example.
     */
    public function test_basic_example(): void
    {
        $response = $this->get(action([UserController::class, 'index']));

        // ...
    }
}
```

<a name="available-assertions"></a>
## 可用的斷言

<a name="response-assertions"></a>
### 回應斷言

Laravel 的 `Illuminate\Testing\TestResponse` 類別提供多種自定義斷言方法，供您在測試應用程式時使用。這些斷言可以從 `json`、`get`、`post`、`put` 與 `delete` 測試方法回傳的回應中存取：

<style>
    .collection-method-list > p {
        columns: 14.4em 2; -moz-columns: 14.4em 2; -webkit-columns: 14.4em 2;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>

<div class="collection-method-list" markdown="1">

[assertAccepted](#assert-accepted)
[assertBadRequest](#assert-bad-request)
[assertClientError](#assert-client-error)
[assertConflict](#assert-conflict)
[assertCookie](#assert-cookie)
[assertCookieExpired](#assert-cookie-expired)
[assertCookieNotExpired](#assert-cookie-not-expired)
[assertCookieMissing](#assert-cookie-missing)
[assertCreated](#assert-created)
[assertDontSee](#assert-dont-see)
[assertDontSeeText](#assert-dont-see-text)
[assertDownload](#assert-download)
[assertExactJson](#assert-exact-json)
[assertExactJsonStructure](#assert-exact-json-structure)
[assertForbidden](#assert-forbidden)
[assertFound](#assert-found)
[assertGone](#assert-gone)
[assertHeader](#assert-header)
[assertHeaderContains](#assert-header-contains)
[assertHeaderMissing](#assert-header-missing)
[assertInternalServerError](#assert-internal-server-error)
[assertJson](#assert-json)
[assertJsonCount](#assert-json-count)
[assertJsonFragment](#assert-json-fragment)
[assertJsonIsArray](#assert-json-is-array)
[assertJsonIsObject](#assert-json-is-object)
[assertJsonMissing](#assert-json-missing)
[assertJsonMissingExact](#assert-json-missing-exact)
[assertJsonMissingValidationErrors](#assert-json-missing-validation-errors)
[assertJsonPath](#assert-json-path)
[assertJsonMissingPath](#assert-json-missing-path)
[assertJsonStructure](#assert-json-structure)
[assertJsonValidationErrors](#assert-json-validation-errors)
[assertJsonValidationErrorFor](#assert-json-validation-error-for)
[assertLocation](#assert-location)
[assertMethodNotAllowed](#assert-method-not-allowed)
[assertMovedPermanently](#assert-moved-permanently)
[assertContent](#assert-content)
[assertNoContent](#assert-no-content)
[assertStreamed](#assert-streamed)
[assertStreamedContent](#assert-streamed-content)
[assertNotFound](#assert-not-found)
[assertOk](#assert-ok)
[assertPaymentRequired](#assert-payment-required)
[assertPlainCookie](#assert-plain-cookie)
[assertRedirect](#assert-redirect)
[assertRedirectBack](#assert-redirect-back)
[assertRedirectBackWithErrors](#assert-redirect-back-with-errors)
[assertRedirectBackWithoutErrors](#assert-redirect-back-without-errors)
[assertRedirectContains](#assert-redirect-contains)
[assertRedirectToRoute](#assert-redirect-to-route)
[assertRedirectToSignedRoute](#assert-redirect-to-signed-route)
[assertRequestTimeout](#assert-request-timeout)
[assertSee](#assert-see)
[assertSeeInOrder](#assert-see-in-order)
[assertSeeText](#assert-see-text)
[assertSeeTextInOrder](#assert-see-text-in-order)
[assertServerError](#assert-server-error)
[assertServiceUnavailable](#assert-service-unavailable)
[assertSessionHas](#assert-session-has)
[assertSessionHasInput](#assert-session-has-input)
[assertSessionHasAll](#assert-session-has-all)
[assertSessionHasErrors](#assert-session-has-errors)
[assertSessionHasErrorsIn](#assert-session-has-errors-in)
[assertSessionHasNoErrors](#assert-session-has-no-errors)
[assertSessionDoesntHaveErrors](#assert-session-doesnt-have-errors)
[assertSessionMissing](#assert-session-missing)
[assertStatus](#assert-status)
[assertSuccessful](#assert-successful)
[assertTooManyRequests](#assert-too-many-requests)
[assertUnauthorized](#assert-unauthorized)
[assertUnprocessable](#assert-unprocessable)
[assertUnsupportedMediaType](#assert-unsupported-media-type)
[assertValid](#assert-valid)
[assertInvalid](#assert-invalid)
[assertViewHas](#assert-view-has)
[assertViewHasAll](#assert-view-has-all)
[assertViewIs](#assert-view-is)
[assertViewMissing](#assert-view-missing)

</div>

<a name="assert-accepted"></a>
#### assertAccepted

斷言回應具有已接受 (202) 的 HTTP 狀態碼：

```php
$response->assertAccepted();
```

<a name="assert-bad-request"></a>
#### assertBadRequest

斷言回應具有錯誤請求 (400) 的 HTTP 狀態碼：

```php
$response->assertBadRequest();
```

<a name="assert-client-error"></a>
#### assertClientError

斷言回應具有用戶端錯誤 (>= 400, < 500) 的 HTTP 狀態碼：

```php
$response->assertClientError();
```

<a name="assert-conflict"></a>
#### assertConflict

斷言回應具有衝突 (409) 的 HTTP 狀態碼：

```php
$response->assertConflict();
```

<a name="assert-cookie"></a>
#### assertCookie

斷言回應包含給定的 cookie：

```php
$response->assertCookie($cookieName, $value = null);
```

<a name="assert-cookie-expired"></a>
#### assertCookieExpired

斷言回應包含給定的 cookie 且該 cookie 已過期：

```php
$response->assertCookieExpired($cookieName);
```

<a name="assert-cookie-not-expired"></a>
#### assertCookieNotExpired

斷言回應包含給定的 cookie 且該 cookie 尚未過期：

```php
$response->assertCookieNotExpired($cookieName);
```

<a name="assert-cookie-missing"></a>
#### assertCookieMissing

斷言回應不包含給定的 cookie：

```php
$response->assertCookieMissing($cookieName);
```

<a name="assert-created"></a>
#### assertCreated

斷言回應具有 201 HTTP 狀態碼：

```php
$response->assertCreated();
```

<a name="assert-dont-see"></a>
#### assertDontSee

斷言給定的字串不包含在應用程式回傳的回應中。除非您將第二個參數傳入 `false`，否則此斷言會自動對給定字串進行跳脫處理：

```php
$response->assertDontSee($value, $escape = true);
```

<a name="assert-dont-see-text"></a>
#### assertDontSeeText

斷言給定的字串不包含在回應文字中。除非您將第二個參數傳入 `false`，否則此斷言會自動對給定字串進行跳脫處理。此方法在進行斷言之前，會先將回應內容傳入 PHP 的 `strip_tags` 函式：

```php
$response->assertDontSeeText($value, $escape = true);
```

<a name="assert-download"></a>
#### assertDownload

斷言回應是一個「下載」。通常，這意味著回傳回應的被呼叫路由回傳了 `Response::download` 回應、`BinaryFileResponse` 或 `Storage::download` 回應：

```php
$response->assertDownload();
```

如果需要，您可以斷言下載的檔案被指派了特定的檔名：

```php
$response->assertDownload('image.jpg');
```

<a name="assert-exact-json"></a>
#### assertExactJson

斷言回應與給定的 JSON 資料完全相符：

```php
$response->assertExactJson(array $data);
```

<a name="assert-exact-json-structure"></a>
#### assertExactJsonStructure

斷言回應與給定的 JSON 結構完全相符：

```php
$response->assertExactJsonStructure(array $data);
```

此方法是 [assertJsonStructure](#assert-json-structure) 的更嚴格變體。與 `assertJsonStructure` 相比，如果回應中包含任何未明確包含在預期 JSON 結構中的鍵，此方法將會失敗。

<a name="assert-forbidden"></a>
#### assertForbidden

斷言回應具有禁止 (403) 的 HTTP 狀態碼：

```php
$response->assertForbidden();
```

<a name="assert-found"></a>
#### assertFound

斷言回應具有找到 (302) 的 HTTP 狀態碼：

```php
$response->assertFound();
```

<a name="assert-gone"></a>
#### assertGone

斷言回應具有已消失 (410) 的 HTTP 狀態碼：

```php
$response->assertGone();
```

<a name="assert-header"></a>
#### assertHeader

斷言回應中存在給定的標頭與值：

```php
$response->assertHeader($headerName, $value = null);
```

<a name="assert-header-contains"></a>
#### assertHeaderContains

斷言給定的標頭包含特定的子字串值：

```php
$response->assertHeaderContains($headerName, $value);
```

<a name="assert-header-missing"></a>
#### assertHeaderMissing

斷言回應中不存在給定的標頭：

```php
$response->assertHeaderMissing($headerName);
```

<a name="assert-internal-server-error"></a>
#### assertInternalServerError

斷言回應具有「內部伺服器錯誤」(500) 的 HTTP 狀態碼：

```php
$response->assertInternalServerError();
```

<a name="assert-json"></a>
#### assertJson

斷言回應包含給定的 JSON 資料：

```php
$response->assertJson(array $data, $strict = false);
```

`assertJson` 方法會將回應轉換為陣列，以驗證給定的陣列是否存在於應用程式回傳的 JSON 回應中。因此，即使 JSON 回應中還有其他屬性，只要給定的片段存在，此測試仍會通過。

<a name="assert-json-count"></a>
#### assertJsonCount

斷言回應的 JSON 在給定鍵路徑下，具有包含預期項目數量的陣列：

```php
$response->assertJsonCount($count, $key = null);
```

<a name="assert-json-fragment"></a>
#### assertJsonFragment

斷言回應中的任何位置包含給定的 JSON 資料：

```php
Route::get('/users', function () {
    return [
        'users' => [
            [
                'name' => 'Taylor Otwell',
            ],
        ],
    ];
});

$response->assertJsonFragment(['name' => 'Taylor Otwell']);
```

<a name="assert-json-is-array"></a>
#### assertJsonIsArray

斷言回應的 JSON 是一個陣列：

```php
$response->assertJsonIsArray();
```

<a name="assert-json-is-object"></a>
#### assertJsonIsObject

斷言回應的 JSON 是一個物件：

```php
$response->assertJsonIsObject();
```

<a name="assert-json-missing"></a>
#### assertJsonMissing

斷言回應不包含給定的 JSON 資料：

```php
$response->assertJsonMissing(array $data);
```

<a name="assert-json-missing-exact"></a>
#### assertJsonMissingExact

斷言回應不包含完全相符的 JSON 資料：

```php
$response->assertJsonMissingExact(array $data);
```

<a name="assert-json-missing-validation-errors"></a>
#### assertJsonMissingValidationErrors

斷言回應對於給定的鍵沒有 JSON 驗證錯誤：

```php
$response->assertJsonMissingValidationErrors($keys);
```

> [!NOTE]
> 更通用的 [assertValid](#assert-valid) 方法可用於斷言回應沒有以 JSON 形式回傳的驗證錯誤，**且**沒有錯誤被閃存 (Flash) 到 Session 儲存空間。

<a name="assert-json-path"></a>
#### assertJsonPath

斷言回應在指定的路徑包含給定的資料：

```php
$response->assertJsonPath($path, $expectedValue);
```

例如，如果您的應用程式回傳以下 JSON 回應：

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

您可以像這樣斷言 `user` 物件的 `name` 屬性與給定的值相符：

```php
$response->assertJsonPath('user.name', 'Steve Schoger');
```

<a name="assert-json-missing-path"></a>
#### assertJsonMissingPath

斷言回應不包含給定的路徑：

```php
$response->assertJsonMissingPath($path);
```

例如，如果您的應用程式回傳以下 JSON 回應：

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

您可以斷言它不包含 `user` 物件的 `email` 屬性：

```php
$response->assertJsonMissingPath('user.email');
```

<a name="assert-json-structure"></a>
#### assertJsonStructure

斷言回應具有給定的 JSON 結構：

```php
$response->assertJsonStructure(array $structure);
```

例如，如果您的應用程式回傳的 JSON 回應包含以下資料：

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

您可以像這樣斷言 JSON 結構符合您的預期：

```php
$response->assertJsonStructure([
    'user' => [
        'name',
    ]
]);
```

有時，應用程式回傳的 JSON 回應可能包含物件陣列：

```json
{
    "user": [
        {
            "name": "Steve Schoger",
            "age": 55,
            "location": "Earth"
        },
        {
            "name": "Mary Schoger",
            "age": 60,
            "location": "Earth"
        }
    ]
}
```

在這種情況下，您可以使用 `*` 字元來對陣列中所有物件的結構進行斷言：

```php
$response->assertJsonStructure([
    'user' => [
        '*' => [
             'name',
             'age',
             'location'
        ]
    ]
]);
```

<a name="assert-json-validation-errors"></a>
#### assertJsonValidationErrors

斷言回應對於給定的鍵具有給定的 JSON 驗證錯誤。當斷言驗證錯誤是以 JSON 結構回傳，而非閃存到 Session 的回應時，應使用此方法：

```php
$response->assertJsonValidationErrors(array $data, $responseKey = 'errors');
```

> [!NOTE]
> 更通用的 [assertInvalid](#assert-invalid) 方法可用於斷言回應具有以 JSON 形式回傳的驗證錯誤，**或**錯誤被閃存到 Session 儲存空間。

<a name="assert-json-validation-error-for"></a>
#### assertJsonValidationErrorFor

斷言回應對於給定的鍵具有任何 JSON 驗證錯誤：

```php
$response->assertJsonValidationErrorFor(string $key, $responseKey = 'errors');
```

<a name="assert-method-not-allowed"></a>
#### assertMethodNotAllowed

斷言回應具有方法不允許 (405) 的 HTTP 狀態碼：

```php
$response->assertMethodNotAllowed();
```

<a name="assert-moved-permanently"></a>
#### assertMovedPermanently

斷言回應具有永久移動 (301) 的 HTTP 狀態碼：

```php
$response->assertMovedPermanently();
```

<a name="assert-location"></a>
#### assertLocation

斷言回應的 `Location` 標頭中具有給定的 URI 值：

```php
$response->assertLocation($uri);
```

<a name="assert-content"></a>
#### assertContent

斷言給定的字串與回應內容相符：

```php
$response->assertContent($value);
```

<a name="assert-no-content"></a>
#### assertNoContent

斷言回應具有給定的 HTTP 狀態碼且無內容：

```php
$response->assertNoContent($status = 204);
```

<a name="assert-streamed"></a>
#### assertStreamed

斷言回應是一個串流回應：

    $response->assertStreamed();

<a name="assert-streamed-content"></a>
#### assertStreamedContent

斷言給定的字串與串流回應內容相符：

```php
$response->assertStreamedContent($value);
```

<a name="assert-not-found"></a>
#### assertNotFound

斷言回應具有找不到 (404) 的 HTTP 狀態碼：

```php
$response->assertNotFound();
```

<a name="assert-ok"></a>
#### assertOk

斷言回應具有 200 HTTP 狀態碼：

```php
$response->assertOk();
```

<a name="assert-payment-required"></a>
#### assertPaymentRequired

斷言回應具有需要付款 (402) 的 HTTP 狀態碼：

```php
$response->assertPaymentRequired();
```

<a name="assert-plain-cookie"></a>
#### assertPlainCookie

斷言回應包含給定的未加密 cookie：

```php
$response->assertPlainCookie($cookieName, $value = null);
```

<a name="assert-redirect"></a>
#### assertRedirect

斷言回應是重導向至給定的 URI：

```php
$response->assertRedirect($uri = null);
```

<a name="assert-redirect-back"></a>
#### assertRedirectBack

斷言回應是否正在重導回前一個頁面：

```php
$response->assertRedirectBack();
```

<a name="assert-redirect-back-with-errors"></a>
#### assertRedirectBackWithErrors

斷言回應是否正在重導回前一個頁面，且 [Session 具有給定的錯誤](#assert-session-has-errors)：

```php
$response->assertRedirectBackWithErrors(
    array $keys = [], $format = null, $errorBag = 'default'
);
```

<a name="assert-redirect-back-without-errors"></a>
#### assertRedirectBackWithoutErrors

斷言回應是否正在重導回前一個頁面，且 Session 不包含任何錯誤訊息：

```php
$response->assertRedirectBackWithoutErrors();
```

<a name="assert-redirect-contains"></a>
#### assertRedirectContains

斷言回應是否正在重導至包含給定字串的 URI：

```php
$response->assertRedirectContains($string);
```

<a name="assert-redirect-to-route"></a>
#### assertRedirectToRoute

斷言回應是重導向至給定的 [命名路由](/docs/{{version}}/routing#named-routes)：

```php
$response->assertRedirectToRoute($name, $parameters = []);
```

<a name="assert-redirect-to-signed-route"></a>
#### assertRedirectToSignedRoute

斷言回應是重導向至給定的 [簽名路由](/docs/{{version}}/urls#signed-urls)：

```php
$response->assertRedirectToSignedRoute($name = null, $parameters = []);
```

<a name="assert-request-timeout"></a>
#### assertRequestTimeout

斷言回應具有請求逾時 (408) 的 HTTP 狀態碼：

```php
$response->assertRequestTimeout();
```

<a name="assert-see"></a>
#### assertSee

斷言回應中包含給定的字串。除非您將第二個參數傳入 `false`，否則此斷言會自動對給定字串進行跳脫處理：

```php
$response->assertSee($value, $escape = true);
```

<a name="assert-see-in-order"></a>
#### assertSeeInOrder

斷言回應中按順序包含給定的字串。除非您將第二個參數傳入 `false`，否則此斷言會自動對給定字串進行跳脫處理：

```php
$response->assertSeeInOrder(array $values, $escape = true);
```

<a name="assert-see-text"></a>
#### assertSeeText

斷言回應文字中包含給定的字串。除非您將第二個參數傳入 `false`，否則此斷言會自動對給定字串進行跳脫處理。在進行斷言之前，會先將回應內容傳入 PHP 的 `strip_tags` 函式：

```php
$response->assertSeeText($value, $escape = true);
```

<a name="assert-see-text-in-order"></a>
#### assertSeeTextInOrder

斷言回應文字中按順序包含給定的字串。除非您將第二個參數傳入 `false`，否則此斷言會自動對給定字串進行跳脫處理。在進行斷言之前，會先將回應內容傳入 PHP 的 `strip_tags` 函式：

```php
$response->assertSeeTextInOrder(array $values, $escape = true);
```

<a name="assert-server-error"></a>
#### assertServerError

斷言回應具有伺服器錯誤 (>= 500, < 600) 的 HTTP 狀態碼：

```php
$response->assertServerError();
```

<a name="assert-service-unavailable"></a>
#### assertServiceUnavailable

斷言回應具有「服務無法使用」(503) 的 HTTP 狀態碼：

```php
$response->assertServiceUnavailable();
```

<a name="assert-session-has"></a>
#### assertSessionHas

斷言 Session 包含給定的資料片段：

```php
$response->assertSessionHas($key, $value = null);
```

如果需要，可以將閉包作為第二個參數提供給 `assertSessionHas` 方法。如果閉包回傳 `true`，則斷言將通過：

```php
$response->assertSessionHas($key, function (User $value) {
    return $value->name === 'Taylor Otwell';
});
```

<a name="assert-session-has-input"></a>
#### assertSessionHasInput

斷言 Session 在 [閃存輸入陣列](/docs/{{version}}/responses#redirecting-with-flashed-session-data) 中具有給定的值：

```php
$response->assertSessionHasInput($key, $value = null);
```

如果需要，可以將閉包作為第二個參數提供給 `assertSessionHasInput` 方法。如果閉包回傳 `true`，則斷言將通過：

```php
use Illuminate\Support\Facades\Crypt;

$response->assertSessionHasInput($key, function (string $value) {
    return Crypt::decryptString($value) === 'secret';
});
```

<a name="assert-session-has-all"></a>
#### assertSessionHasAll

斷言 Session 包含給定的鍵值對陣列：

```php
$response->assertSessionHasAll(array $data);
```

例如，如果您的應用程式 Session 包含 `name` 與 `status` 鍵，您可以像這樣斷言兩者皆存在且具有指定的值：

```php
$response->assertSessionHasAll([
    'name' => 'Taylor Otwell',
    'status' => 'active',
]);
```

<a name="assert-session-has-errors"></a>
#### assertSessionHasErrors

斷言 Session 包含給定 `$keys` 的錯誤。如果 `$keys` 是一個關聯陣列，則斷言 Session 為每個欄位 (鍵) 包含特定的錯誤訊息 (值)。在測試將驗證錯誤閃存到 Session 而非以 JSON 結構回傳的路由時，應使用此方法：

```php
$response->assertSessionHasErrors(
    array $keys = [], $format = null, $errorBag = 'default'
);
```

例如，要斷言 `name` 與 `email` 欄位具有閃存到 Session 的驗證錯誤訊息，您可以像這樣呼叫 `assertSessionHasErrors` 方法：

```php
$response->assertSessionHasErrors(['name', 'email']);
```

或者，您可以斷言給定欄位具有特定的驗證錯誤訊息：

```php
$response->assertSessionHasErrors([
    'name' => 'The given name was invalid.'
]);
```

> [!NOTE]
> 更通用的 [assertInvalid](#assert-invalid) 方法可用於斷言回應具有以 JSON 形式回傳的驗證錯誤，**或**錯誤被閃存到 Session 儲存空間。

<a name="assert-session-has-errors-in"></a>
#### assertSessionHasErrorsIn

斷言 Session 在特定的 [錯誤包 (Error Bag)](/docs/{{version}}/validation#named-error-bags) 中包含給定 `$keys` 的錯誤。如果 `$keys` 是一個關聯陣列，則斷言 Session 為該錯誤包內的每個欄位 (鍵) 包含特定的錯誤訊息 (值)：

```php
$response->assertSessionHasErrorsIn($errorBag, $keys = [], $format = null);
```

<a name="assert-session-has-no-errors"></a>
#### assertSessionHasNoErrors

斷言 Session 沒有驗證錯誤：

```php
$response->assertSessionHasNoErrors();
```

<a name="assert-session-doesnt-have-errors"></a>
#### assertSessionDoesntHaveErrors

斷言 Session 對於給定的鍵沒有驗證錯誤：

```php
$response->assertSessionDoesntHaveErrors($keys = [], $format = null, $errorBag = 'default');
```

> [!NOTE]
> 更通用的 [assertValid](#assert-valid) 方法可用於斷言回應沒有以 JSON 形式回傳的驗證錯誤，**且**沒有錯誤被閃存到 Session 儲存空間。

<a name="assert-session-missing"></a>
#### assertSessionMissing

斷言 Session 不包含給定的鍵：

```php
$response->assertSessionMissing($key);
```

<a name="assert-status"></a>
#### assertStatus

斷言回應具有給定的 HTTP 狀態碼：

```php
$response->assertStatus($code);
```

<a name="assert-successful"></a>
#### assertSuccessful

斷言回應具有成功 (>= 200 且 < 300) 的 HTTP 狀態碼：

```php
$response->assertSuccessful();
```

<a name="assert-too-many-requests"></a>
#### assertTooManyRequests

斷言回應具有請求過多 (429) 的 HTTP 狀態碼：

```php
$response->assertTooManyRequests();
```

<a name="assert-unauthorized"></a>
#### assertUnauthorized

斷言回應具有未經授權 (401) 的 HTTP 狀態碼：

```php
$response->assertUnauthorized();
```

<a name="assert-unprocessable"></a>
#### assertUnprocessable

斷言回應具有無法處理的實體 (422) 的 HTTP 狀態碼：

```php
$response->assertUnprocessable();
```

<a name="assert-unsupported-media-type"></a>
#### assertUnsupportedMediaType

斷言回應具有不支援的媒體類型 (415) 的 HTTP 狀態碼：

```php
$response->assertUnsupportedMediaType();
```

<a name="assert-valid"></a>
#### assertValid

斷言回應對於給定的鍵沒有驗證錯誤。此方法可用於斷言驗證錯誤是以 JSON 結構回傳，或是驗證錯誤已被閃存到 Session 的回應：

```php
// Assert that no validation errors are present...
$response->assertValid();

// Assert that the given keys do not have validation errors...
$response->assertValid(['name', 'email']);
```

<a name="assert-invalid"></a>
#### assertInvalid

斷言回應對於給定的鍵具有驗證錯誤。此方法可用於斷言驗證錯誤是以 JSON 結構回傳，或是驗證錯誤已被閃存到 Session 的回應：

```php
$response->assertInvalid(['name', 'email']);
```

您也可以斷言給定鍵具有特定的驗證錯誤訊息。執行此操作時，您可以提供完整訊息或僅提供訊息的一小部分：

```php
$response->assertInvalid([
    'name' => 'The name field is required.',
    'email' => 'valid email address',
]);
```

如果您想斷言給定欄位是僅有的具有驗證錯誤的欄位，您可以使用 `assertOnlyInvalid` 方法：

```php
$response->assertOnlyInvalid(['name', 'email']);
```

<a name="assert-view-has"></a>
#### assertViewHas

斷言回應視圖包含給定的資料片段：

```php
$response->assertViewHas($key, $value = null);
```

將閉包作為第二個參數傳遞給 `assertViewHas` 方法，將允許您檢查特定的視圖資料並對其進行斷言：

```php
$response->assertViewHas('user', function (User $user) {
    return $user->name === 'Taylor';
});
```

此外，視圖資料可以作為回應上的陣列變數來存取，讓您可以方便地檢查它：

```php tab=Pest
expect($response['name'])->toBe('Taylor');
```

```php tab=PHPUnit
$this->assertEquals('Taylor', $response['name']);
```

<a name="assert-view-has-all"></a>
#### assertViewHasAll

斷言回應視圖具有給定的資料列表：

```php
$response->assertViewHasAll(array $data);
```

此方法可用於斷言視圖僅包含與給定鍵相符的資料：

```php
$response->assertViewHasAll([
    'name',
    'email',
]);
```

或者，您可以斷言視圖資料存在且具有特定值：

```php
$response->assertViewHasAll([
    'name' => 'Taylor Otwell',
    'email' => 'taylor@example.com,',
]);
```

<a name="assert-view-is"></a>
#### assertViewIs

斷言路由回傳了給定的視圖：

```php
$response->assertViewIs($value);
```

<a name="assert-view-missing"></a>
#### assertViewMissing

斷言給定的資料鍵未提供給應用程式回應中回傳的視圖：

```php
$response->assertViewMissing($key);
```

<a name="authentication-assertions"></a>
### 身份驗證斷言

Laravel 也提供了各種與身份驗證相關的斷言，供您在應用程式的功能測試中利用。請注意，這些方法是在測試類別本身呼叫的，而不是由 `get` 和 `post` 等方法回傳的 `Illuminate\Testing\TestResponse` 實例。


<a name="assert-authenticated"></a>
#### assertAuthenticated

斷言使用者已通過身份驗證：

```php
$this->assertAuthenticated($guard = null);
```


<a name="assert-guest"></a>
#### assertGuest

斷言使用者未通過身份驗證：

```php
$this->assertGuest($guard = null);
```


<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

斷言特定的使用者已通過身份驗證：

```php
$this->assertAuthenticatedAs($user, $guard = null);
```

<a name="validation-assertions"></a>
## 驗證斷言

Laravel 提供了兩個主要的驗證相關斷言，供您用來確保請求中提供的資料是有效或無效的。

<a name="validation-assert-valid"></a>
#### assertValid

斷言回應中給定的鍵 (keys) 沒有驗證錯誤。此方法可用於針對驗證錯誤以 JSON 結構回傳，或驗證錯誤已被快閃 (flashed) 至 Session 的回應進行斷言：

```php
// Assert that no validation errors are present...
$response->assertValid();

// Assert that the given keys do not have validation errors...
$response->assertValid(['name', 'email']);
```

<a name="validation-assert-invalid"></a>
#### assertInvalid

斷言回應中給定的鍵具有驗證錯誤。此方法可用於針對驗證錯誤以 JSON 結構回傳，或驗證錯誤已被快閃至 Session 的回應進行斷言：

```php
$response->assertInvalid(['name', 'email']);
```

您也可以斷言給定的鍵具有特定的驗證錯誤訊息。執行此操作時，您可以提供完整的訊息，或僅提供訊息的一小部分：

```php
$response->assertInvalid([
    'name' => 'The name field is required.',
    'email' => 'valid email address',
]);
```