# Mocking

- [簡介](#introduction)
- [Mocking 物件](#mocking-objects)
- [Mocking Facades](#mocking-facades)
    - [Facade Spies](#facade-spies)
- [與時間互動](#interacting-with-time)

<a name="introduction"></a>
## 簡介

測試 Laravel 應用程式時，你可能希望「mock」應用程式的某些部分，以便在特定的測試期間不實際執行它們。例如，在測試派發事件的控制器時，你可能希望 mock 事件監聽器，使它們在測試期間不會實際執行。這讓你只需測試控制器的 HTTP 回應，而無需擔心事件監聽器的執行，因為事件監聽器可以在它們自己的測試案例中進行測試。

Laravel 內建了用於 mock 事件、任務 (jobs) 和其他 facades 的便利方法。這些輔助方法主要是在 Mockery 之上提供一個便利層，因此你不需要手動編寫複雜的 Mockery 方法呼叫。


<a name="mocking-objects"></a>
## Mocking 物件

當要 mock 一個將透過 Laravel 的 [服務容器 (service container)](/docs/{{version}}/container) 注入到應用程式中的物件時，你需要在容器中將 mock 的實例綁定為 `instance` 綁定。這會指示容器使用該物件的 mock 實例，而不是自行建構物件：

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

為了更加方便，你可以使用 Laravel 基礎測試類別提供的 `mock` 方法。例如，以下範例等同於上述範例：

```php
use App\Service;
use Mockery\MockInterface;

$mock = $this->mock(Service::class, function (MockInterface $mock) {
    $mock->expects('process');
});
```

當你只需要 mock 物件的少數幾個方法時，可以使用 `partialMock` 方法。未被 mock 的方法在被呼叫時將正常執行：

```php
use App\Service;
use Mockery\MockInterface;

$mock = $this->partialMock(Service::class, function (MockInterface $mock) {
    $mock->expects('process');
});
```

同樣地，如果你想對物件進行 [spy](http://docs.mockery.io/en/latest/reference/spies.html)，Laravel 的基礎測試類別提供了一個 `spy` 方法，作為 `Mockery::spy` 方法的便利封裝。Spy 與 mock 類似；然而，spy 會記錄 spy 與被測試程式碼之間的所有互動，讓你在程式碼執行後進行斷言 (assertions)：

```php
use App\Service;

$spy = $this->spy(Service::class);

// ...

$spy->shouldHaveReceived('process');
```


<a name="mocking-facades"></a>
## Mocking Facades

與傳統的靜態方法呼叫不同，[facades](/docs/{{version}}/facades) (包含 [即時 facades](/docs/{{version}}/facades#real-time-facades)) 可以被 mock。這與傳統的靜態方法相比具有很大的優勢，並賦予你與使用傳統相依注入時相同的可測試性。在測試時，你可能經常想要 mock 控制器中對 Laravel facade 的呼叫。例如，考慮以下控制器動作：

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

我們可以使用 `expects` 方法來 mock 對 `Cache` facade 的呼叫，該方法將回傳一個 [Mockery](https://github.com/padraic/mockery) mock 的實例。由於 facades 實際上是由 Laravel [服務容器 (service container)](/docs/{{version}}/container) 解析和管理的，因此它們比典型的靜態類別具有更高的可測試性。例如，讓我們 mock 對 `Cache` facade 的 `get` 方法呼叫：

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
> 你不應該 mock `Request` facade。相反地，在執行測試時，請將你想要的輸入傳遞到 [HTTP 測試方法](/docs/{{version}}/http-tests)（如 `get` 和 `post`）中。同樣地，不要 mock `Config` facade，而是在測試中呼叫 `Config::set` 方法。


<a name="facade-spies"></a>
### Facade Spies

如果你想要對 facade 進行 [spy](http://docs.mockery.io/en/latest/reference/spies.html)，可以對相應的 facade 呼叫 `spy` 方法。Spy 與 mock 類似；然而，spy 會記錄 spy 與被測試程式碼之間的所有互動，讓你在程式碼執行後進行斷言：

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

在測試時，您偶爾可能需要修改由 `now` 或 `Illuminate\Support\Carbon::now()` 等輔助函式回傳的時間。幸好，Laravel 的基礎功能測試類別包含了可以讓您操控目前時間的輔助方法：

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
    $this->travelTo(now()->minus(hours: 6));

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
    $this->travelTo(now()->minus(hours: 6));

    // Return back to the present time...
    $this->travelBack();
}
```

您也可以為各種時間旅行方法提供一個閉包 (Closure)。該閉包將會在時間凍結於指定時間的狀態下被調用。一旦閉包執行完畢，時間將恢復正常：

```php
$this->travel(5)->days(function () {
    // Test something five days into the future...
});

$this->travelTo(now()->mins(days: 10), function () {
    // Test something during a given moment...
});
```

`freezeTime` 方法可以用來凍結目前時間。同樣地，`freezeSecond` 方法會凍結目前時間，但會設定在目前秒數的起始處：

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

如您所料，上述討論的所有方法主要用於測試對時間敏感的應用程式行為，例如鎖定討論論壇中不活躍的討論串：

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