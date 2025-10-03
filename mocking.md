# 模擬

- [簡介](#introduction)
- [模擬物件](#mocking-objects)
- [模擬 Facades](#mocking-facades)
    - [Facade Spies](#facade-spies)
- [與時間互動](#interacting-with-time)

<a name="introduction"></a>
## 簡介

在測試 Laravel 應用程式時，您可能希望「模擬」應用程式的某些面向，使其在給定的測試中不會實際執行。例如，當測試一個會派發事件的控制器時，您可能希望模擬事件監聽器，使其在測試期間不會實際執行。這讓您能夠只測試控制器的 HTTP 回應，而無需擔心事件監聽器的執行，因為事件監聽器可以在它們自己的測試案例中進行測試。

Laravel 開箱即用地提供了用於模擬事件、任務 (jobs) 和其他 Facades 的實用輔助方法。這些輔助方法主要為 Mockery 提供一個便利層，讓您不必手動進行複雜的 Mockery 方法呼叫。


<a name="mocking-objects"></a>
## 模擬物件

當您模擬一個將透過 Laravel 的 [服務容器](/docs/{{version}}/container) 注入到應用程式中的物件時，您需要將模擬的實例作為 `instance` 綁定繫結到容器中。這將指示容器使用您模擬的物件實例，而不是自行建構該物件：

```php tab=Pest
use App\Service;
use Mockery;
use Mockery\MockInterface;

test('something can be mocked', function () {
    $this->instance(
        Service::class,
        Mockery::mock(Service::class, function (MockInterface $mock) {
            $mock->expects('process');
        })
    );
});
```

```php tab=PHPUnit
use App\Service;
use Mockery;
use Mockery\MockInterface;

public function test_something_can_be_mocked(): void
{
    $this->instance(
        Service::class,
        Mockery::mock(Service::class, function (MockInterface $mock) {
            $mock->expects('process');
        })
    );
}
```

為了讓操作更為方便，您可以使用 Laravel 基礎測試案例類別提供的 `mock` 方法。例如，以下範例與上述範例等效：

```php
use App\Service;
use Mockery\MockInterface;

$mock = $this->mock(Service::class, function (MockInterface $mock) {
    $mock->expects('process');
});
```

當您只需要模擬物件的少數幾個方法時，可以使用 `partialMock` 方法。未被模擬的方法在呼叫時會正常執行：

```php
use App\Service;
use Mockery\MockInterface;

$mock = $this->partialMock(Service::class, function (MockInterface $mock) {
    $mock->expects('process');
});
```

同樣地，如果您想 [監控 (spy)](http://docs.mockery.io/en/latest/reference/spies.html) 一個物件，Laravel 的基礎測試案例類別提供了 `spy` 方法，作為 `Mockery::spy` 方法的便利包裝。Spies 類似於 mocks；然而，spies 會記錄 spy 與被測試程式碼之間的任何互動，讓您可以在程式碼執行後進行斷言：

```php
use App\Service;

$spy = $this->spy(Service::class);

// ...

$spy->shouldHaveReceived('process');
```


<a name="mocking-facades"></a>
## 模擬 Facades

與傳統的靜態方法呼叫不同，[Facades](/docs/{{version}}/facades) (包括 [即時 Facades](/docs/{{version}}/facades#real-time-facades)) 可以被模擬。這比傳統的靜態方法提供了巨大的優勢，並賦予您使用傳統依賴注入時相同的可測試性。在測試時，您可能經常需要模擬控制器中對 Laravel Facade 的呼叫。例如，考慮以下控制器動作：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\Cache;

class UserController extends Controller
{
    /**
     * Retrieve a list of all users of the application.
     */
    public function index(): array
    {
        $value = Cache::get('key');

        return [
            // ...
        ];
    }
}
```

我們可以透過使用 `expects` 方法來模擬對 `Cache` Facade 的呼叫，該方法將回傳一個 [Mockery](https://github.com/padraic/mockery) 模擬實例。由於 Facades 實際上是由 Laravel [服務容器](/docs/{{version}}/container) 解析和管理的，它們比典型的靜態類別具有更高的可測試性。例如，讓我們模擬對 `Cache` Facade 的 `get` 方法的呼叫：

```php tab=Pest
<?php

use Illuminate\Support\Facades\Cache;

test('get index', function () {
    Cache::expects('get')
        ->with('key')
        ->andReturn('value');

    $response = $this->get('/users');

    // ...
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Support\Facades\Cache;
use Tests\TestCase;

class UserControllerTest extends TestCase
{
    public function test_get_index(): void
    {
        Cache::expects('get')
            ->with('key')
            ->andReturn('value');

        $response = $this->get('/users');

        // ...
    }
}
```

> [!WARNING]
> 您不應該模擬 `Request` Facade。相反，在執行測試時，將您想要的輸入傳遞給 [HTTP 測試方法](/docs/{{version}}/http-tests)，例如 `get` 和 `post`。同樣地，與其模擬 `Config` Facade，不如在測試中呼叫 `Config::set` 方法。


<a name="facade-spies"></a>
### Facade Spies

如果您想 [監控 (spy)](http://docs.mockery.io/en/latest/reference/spies.html) 一個 Facade，您可以對應的 Facade 上呼叫 `spy` 方法。Spies 類似於 mocks；然而，spies 會記錄 spy 與被測試程式碼之間的任何互動，讓您可以在程式碼執行後進行斷言：

```php tab=Pest
<?php

use Illuminate\Support\Facades\Cache;

test('values are stored in cache', function () {
    Cache::spy();

    $response = $this->get('/');

    $response->assertStatus(200);

    Cache::shouldHaveReceived('put')->with('name', 'Taylor', 10);
});
```

```php tab=PHPUnit
use Illuminate\Support\Facades\Cache;

public function test_values_are_stored_in_cache(): void
{
    Cache::spy();

    $response = $this->get('/');

    $response->assertStatus(200);

    Cache::shouldHaveReceived('put')->with('name', 'Taylor', 10);
}
```

<a name="interacting-with-time"></a>
## 與時間互動

測試時，你偶爾會需要修改由 `now` 或 `Illuminate\Support\Carbon::now()` 等輔助函式回傳的時間。值得慶幸的是，Laravel 的基礎功能測試類別包含了能讓你操縱當前時間的輔助函式：

```php tab=Pest
test('time can be manipulated', function () {
    // Travel into the future...
    $this->travel(5)->milliseconds();
    $this->travel(5)->seconds();
    $this->travel(5)->minutes();
    $this->travel(5)->hours();
    $this->travel(5)->days();
    $this->travel(5)->weeks();
    $this->travel(5)->years();

    // Travel into the past...
    $this->travel(-5)->hours();

    // Travel to an explicit time...
    $this->travelTo(now()->subHours(6));

    // Return back to the present time...
    $this->travelBack();
});
```

```php tab=PHPUnit
public function test_time_can_be_manipulated(): void
{
    // Travel into the future...
    $this->travel(5)->milliseconds();
    $this->travel(5)->seconds();
    $this->travel(5)->minutes();
    $this->travel(5)->hours();
    $this->travel(5)->days();
    $this->travel(5)->weeks();
    $this->travel(5)->years();

    // Travel into the past...
    $this->travel(-5)->hours();

    // Travel to an explicit time...
    $this->travelTo(now()->subHours(6));

    // Return back to the present time...
    $this->travelBack();
}
```

你也可以為多種時間旅行方法提供一個閉包。閉包將會以指定時間凍結的狀態被呼叫。一旦閉包執行完畢，時間將會恢復正常：

```php
$this->travel(5)->days(function () {
    // Test something five days into the future...
});

$this->travelTo(now()->subDays(10), function () {
    // Test something during a given moment...
});
```

可以使用 `freezeTime` 方法來凍結當前時間。類似地，`freezeSecond` 方法會凍結當前時間，但會從當前秒數的開始點凍結：

```php
use Illuminate\Support\Carbon;

// Freeze time and resume normal time after executing closure...
$this->freezeTime(function (Carbon $time) {
    // ...
});

// Freeze time at the current second and resume normal time after executing closure...
$this->freezeSecond(function (Carbon $time) {
    // ...
})
```

如你所料，上述所有方法主要用於測試時間敏感的應用程式行為，例如鎖定討論區中的非活躍貼文：

```php tab=Pest
use App\Models\Thread;

test('forum threads lock after one week of inactivity', function () {
    $thread = Thread::factory()->create();

    $this->travel(1)->week();

    expect($thread->isLockedByInactivity())->toBeTrue();
});
```

```php tab=PHPUnit
use App\Models\Thread;

public function test_forum_threads_lock_after_one_week_of_inactivity()
{
    $thread = Thread::factory()->create();

    $this->travel(1)->week();

    $this->assertTrue($thread->isLockedByInactivity());
}
```