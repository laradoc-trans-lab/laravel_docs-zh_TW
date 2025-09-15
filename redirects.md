# HTTP 重新導向

- [建立重新導向](#creating-redirects)
- [重新導向至具名路由](#redirecting-named-routes)
- [重新導向至控制器動作](#redirecting-controller-actions)
- [重新導向並附帶快閃 Session 資料](#redirecting-with-flashed-session-data)

<a name="creating-redirects"></a>
## 建立重新導向

重新導向回應是 `Illuminate\Http\RedirectResponse` 類別的實例，並包含將使用者重新導向至另一個 URL 所需的正確標頭。有幾種方法可以產生 `RedirectResponse` 實例。最簡單的方法是使用全域 `redirect` 輔助函式：

    Route::get('/dashboard', function () {
        return redirect('/home/dashboard');
    });

有時您可能希望將使用者重新導向至他們先前的位置，例如當提交的表單無效時。您可以透過使用全域 `back` 輔助函式來實現。由於此功能利用了 [session](/docs/{{version}}/session)，請確保呼叫 `back` 函式的路由正在使用 `web` Middleware 群組或已套用所有 session Middleware：

    Route::post('/user/profile', function () {
        // Validate the request...

        return back()->withInput();
    });

<a name="redirecting-named-routes"></a>
## 重新導向至具名路由

當您呼叫不帶參數的 `redirect` 輔助函式時，會傳回 `Illuminate\Routing\Redirector` 的實例，讓您可以呼叫 `Redirector` 實例上的任何方法。例如，要產生重新導向至具名路由的 `RedirectResponse`，您可以使用 `route` 方法：

    return redirect()->route('login');

如果您的路由有參數，您可以將它們作為第二個引數傳遞給 `route` 方法：

    // For a route with the following URI: profile/{id}

    return redirect()->route('profile', ['id' => 1]);

為方便起見，Laravel 也提供了全域 `to_route` 函式：

    return to_route('profile', ['id' => 1]);

<a name="populating-parameters-via-eloquent-models"></a>
#### 透過 Eloquent 模型填充參數

如果您要重新導向至具有從 Eloquent 模型填充的「ID」參數的路由，您可以傳遞模型本身。ID 將會自動擷取：

    // For a route with the following URI: profile/{id}

    return redirect()->route('profile', [$user]);

如果您想自訂放置在路由參數中的值，您應該覆寫 Eloquent 模型上的 `getRouteKey` 方法：

    /**
     * Get the value of the model's route key.
     */
    public function getRouteKey(): mixed
    {
        return $this->slug;
    }

<a name="redirecting-controller-actions"></a>
## 重新導向至控制器動作

您也可以產生重新導向至[控制器動作](/docs/{{version}}/controllers)。為此，請將控制器和動作名稱傳遞給 `action` 方法：

    use App\Http\Controllers\HomeController;

    return redirect()->action([HomeController::class, 'index']);

如果您的控制器路由需要參數，您可以將它們作為第二個引數傳遞給 `action` 方法：

    return redirect()->action(
        [UserController::class, 'profile'], ['id' => 1]
    );

<a name="redirecting-with-flashed-session-data"></a>
## 重新導向並附帶快閃 Session 資料

重新導向至新 URL 並[將資料快閃至 session](/docs/{{version}}/session#flash-data) 通常是同時進行的。通常，這是在成功執行動作後完成的，此時您會將成功訊息快閃至 session。為方便起見，您可以建立 `RedirectResponse` 實例並透過單一、流暢的方法鏈將資料快閃至 session：

    Route::post('/user/profile', function () {
        // Update the user's profile...

        return redirect('/dashboard')->with('status', 'Profile updated!');
    });

您可以使用 `RedirectResponse` 實例提供的 `withInput` 方法，在將使用者重新導向至新位置之前，將目前請求的輸入資料快閃至 session。一旦輸入已快閃至 session，您可以在下一個請求期間輕鬆[擷取它](/docs/{{version}}/requests#retrieving-old-input)：

    return back()->withInput();

使用者重新導向後，您可以從 [session](/docs/{{version}}/session) 顯示快閃訊息。例如，使用 [Blade 語法](/docs/{{version}}/blade)：

    @if (session('status'))
        <div class="alert alert-success">
            {{ session('status') }}
        </div>
    @endif
