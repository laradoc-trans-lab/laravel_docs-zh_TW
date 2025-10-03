# 貢獻指南

- [錯誤報告](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [選擇哪個分支？](#which-branch)
- [編譯資源](#compiled-assets)
- [安全漏洞](#security-vulnerabilities)
- [編碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤報告

為鼓勵積極協作，Laravel 強烈鼓勵提交拉取請求，而非僅僅是錯誤報告。拉取請求只有在標記為「準備審閱」（而非「草稿」狀態）且所有新功能的測試都通過時才會被審閱。懸而未決、不活躍的處於「草稿」狀態的拉取請求將在幾天後關閉。

然而，如果您提交錯誤報告，您的問題應包含一個標題和對該問題的清晰描述。您還應盡可能多地包含相關資訊和一個演示該問題的程式碼範例。錯誤報告的目標是讓您自己及他人更容易重現該錯誤並開發出修復程式。

請記住，建立錯誤報告是希望其他有相同問題的人能夠與您合作解決該問題。不要期望錯誤報告會自動獲得任何活動，或其他人會立刻修正它。建立錯誤報告是為了幫助您自己和他人開始解決問題。如果您想貢獻一份力，可以透過修復 [我們問題追蹤器中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel) 來提供幫助。您必須使用 GitHub 進行身份驗證才能查看所有 Laravel 的問題。

如果您在使用 Laravel 時發現不正確的 DocBlock、PHPStan 或 IDE 警告，請不要建立 GitHub 問題。相反地，請提交拉取請求來修正該問題。

Laravel 原始碼在 GitHub 上管理，每個 Laravel 專案都有其儲存庫：

<div class="content-list" markdown="1">

- [Laravel Application](https://github.com/laravel/laravel)
- [Laravel Art](https://github.com/laravel/art)
- [Laravel Documentation](https://github.com/laravel/docs)
- [Laravel Dusk](https://github.com/laravel/dusk)
- [Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Laravel Echo](https://github.com/laravel/echo)
- [Laravel Envoy](https://github.com/laravel/envoy)
- [Laravel Folio](https://github.com/laravel/folio)
- [Laravel Framework](https://github.com/laravel/framework)
- [Laravel Homestead](https://github.com/laravel/homestead) ([建置腳本](https://github.com/laravel/settler))
- [Laravel Horizon](https://github.com/laravel/horizon)
- [Laravel Passport](https://github.com/laravel/passport)
- [Laravel Pennant](https://github.com/laravel/pennant)
- [Laravel Pint](https://github.com/laravel/pint)
- [Laravel Prompts](https://github.com/laravel/prompts)
- [Laravel Reverb](https://github.com/laravel/reverb)
- [Laravel Sail](https://github.com/laravel/sail)
- [Laravel Sanctum](https://github.com/laravel/sanctum)
- [Laravel Scout](https://github.com/laravel/scout)
- [Laravel Socialite](https://github.com/laravel/socialite)
- [Laravel Telescope](https://github.com/laravel/telescope)
- [Laravel Livewire Starter Kit](https://github.com/laravel/livewire-starter-kit)
- [Laravel React Starter Kit](https://github.com/laravel/react-starter-kit)
- [Laravel Vue Starter Kit](https://github.com/laravel/vue-starter-kit)

</div>


<a name="support-questions"></a>
## 支援問題

Laravel 的 GitHub 問題追蹤器並非旨在提供 Laravel 協助或支援。相反地，請使用以下其中一個管道：

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/laravel/framework/discussions)
- [Laracasts Forums](https://laracasts.com/discuss)
- [Laravel.io Forums](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

</div>


<a name="core-development-discussion"></a>
## 核心開發討論

您可以在 Laravel framework 儲存庫的 [GitHub 討論區](https://github.com/laravel/framework/discussions) 提出新功能或改進現有 Laravel 行為。如果您提出新功能，請願意實作完成該功能所需的至少一部分程式碼。

關於錯誤、新功能和現有功能實作的非正式討論在 [Laravel Discord 伺服器](https://discord.gg/laravel) 的 `#internals` 頻道進行。Laravel 的維護者 Taylor Otwell 通常在工作日早上 8 點到下午 5 點（UTC-06:00 或 America/Chicago 時區）出現在頻道中，其他時間則偶爾出現。


<a name="which-branch"></a>
## 選擇哪個分支？

**所有**錯誤修復都應提交到支援錯誤修復的最新版本（目前為 `12.x`）。錯誤修復**絕不**應提交到 `master` 分支，除非它們修復了僅存在於即將發布版本中的功能。

與當前版本**完全向後兼容**的**次要**功能可以提交到最新的穩定分支（目前為 `12.x`）。

帶有破壞性變更的**主要**新功能或功能應始終提交到 `master` 分支，該分支包含即將發布的版本。


<a name="compiled-assets"></a>
## 編譯資源

如果您提交的變更將影響編譯檔案，例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 的大多數檔案，請不要提交編譯後的檔案。由於它們的龐大體積，維護者無法實際審閱。這可能被利用作為向 Laravel 注入惡意程式碼的方式。為了防禦性地防止這種情況，所有編譯後的檔案將由 Laravel 維護者產生並提交。


<a name="security-vulnerabilities"></a>
## 安全漏洞

如果您在 Laravel 中發現安全漏洞，請發送電子郵件至 Taylor Otwell (taylor@laravel.com)。所有安全漏洞都將迅速處理。


<a name="coding-style"></a>
## 編碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 編碼標準和 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 文件區塊範例。請注意，`@param` 屬性後面跟著兩個空格、參數類型，再兩個空格，最後是變數名稱：

```php
/**
 * Register a binding with the container.
 *
 * @param  string|array  $abstract
 * @param  \Closure|string|null  $concrete
 * @param  bool  $shared
 * @return void
 *
 * @throws \Exception
 */
public function bind($abstract, $concrete = null, $shared = false)
{
    // ...
}
```

當 `@param` 或 `@return` 屬性由於使用原生類型而多餘時，可以將其移除：

```php
/**
 * Execute the job.
 */
public function handle(AudioProcessor $processor): void
{
    //
}
```

然而，當原生類型是泛型時，請透過使用 `@param` 或 `@return` 屬性來指定泛型類型：

```php
/**
 * Get the attachments for the message.
 *
 * @return array<int, \Illuminate\Mail\Mailables\Attachment>
 */
public function attachments(): array
{
    return [
        Attachment::fromStorage('/path/to/file'),
    ];
}
```


<a name="styleci"></a>
### StyleCI

即使您的程式碼風格不完美也無需擔心！[StyleCI](https://styleci.io/) 會在拉取請求合併後，自動將任何風格修復合併到 Laravel 儲存庫中。這使我們能夠專注於貢獻的內容，而非程式碼風格。

<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則源自於 Ruby 的行為準則。任何違反行為準則的行為，可以向 Taylor Otwell (taylor@laravel.com) 舉報：

<div class="content-list" markdown="1">

- 參與者應包容不同意見。
- 參與者必須確保其言行舉止沒有人身攻擊或輕蔑的言論。
- 在解讀他人的言行時，參與者應始終假設其懷有善意。
- 任何可合理視為騷擾的行為將不被容忍。

</div>