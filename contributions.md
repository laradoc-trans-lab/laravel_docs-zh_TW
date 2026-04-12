# 貢獻指南

- [錯誤報告](#bug-reports)
- [支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [該選擇哪個分支？](#which-branch)
- [編譯後的資源](#compiled-assets)
- [AI 生成的貢獻](#ai-generated-contributions)
- [安全性漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤報告

為了鼓勵積極的協作，Laravel 強烈建議提交 Pull Request，而不僅僅是錯誤報告。只有在標記為 「ready for review」（而非處於 「draft」 狀態）且新功能的所有測試都通過時，Pull Request 才會被審查。長時間處於 「draft」 狀態且沒有活動的 Pull Request 將在幾天後被關閉。

然而，如果您提交錯誤報告，您的 Issue 應該包含一個標題以及對問題的清晰描述。您還應該盡可能提供相關資訊以及一個能演示該問題的程式碼範例。錯誤報告的目標是讓您自己以及他人能輕鬆地重現錯誤並開發修復方案。

請記住，建立錯誤報告是希望其他遇到相同問題的人能夠與您共同協作解決。不要期待錯誤報告會自動獲得關注，或有人會立刻跳出來修復它。建立錯誤報告是為了幫助您自己和他人開始解決問題。如果您想貢獻一份力量，可以透過修復 [我們 Issue 追蹤器中列出的任何錯誤](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel) 來提供幫助。您必須通過 GitHub 認證才能查看 Laravel 的所有 Issue。

如果您在使用 Laravel 時發現不正確的 DocBlock、PHPStan 或 IDE 警告，請不要建立 GitHub Issue，請直接提交 Pull Request 來修復該問題。

Laravel 的原始碼在 GitHub 上管理，每個 Laravel 專案都有各自的儲存庫：

<div class="content-list" markdown="1">

- [Laravel Application](https://github.com/laravel/laravel)
- [Laravel Art](https://github.com/laravel/art)
- [Laravel Boost](https://github.com/laravel/boost)
- [Laravel Documentation](https://github.com/laravel/docs)
- [Laravel Dusk](https://github.com/laravel/dusk)
- [Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Laravel Echo](https://github.com/laravel/echo)
- [Laravel Envoy](https://github.com/laravel/envoy)
- [Laravel Folio](https://github.com/laravel/folio)
- [Laravel Framework](https://github.com/laravel/framework)
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
- [Laravel Svelte Starter Kit](https://github.com/laravel/svelte-starter-kit)
- [Laravel Vue Starter Kit](https://github.com/laravel/vue-starter-kit)

</div>


<a name="support-questions"></a>
## 支援問題

Laravel 的 GitHub Issue 追蹤器並非旨在提供 Laravel 的協助或支援。請改用以下其中一個管道：

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

您可以在 Laravel 框架儲存庫的 [GitHub 討論區](https://github.com/laravel/framework/discussions) 提出新功能建議或對現有 Laravel 行為的改進。如果您提出了新功能，請願意實作完成該功能所需的部分程式碼。

關於錯誤、新功能以及現有功能實作的非正式討論，會在 [Laravel Discord 伺服器](https://discord.gg/laravel) 的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常在工作日的早上 8 點到下午 5 點（UTC-06:00 或美國/芝加哥時間）出現在該頻道，其他時間則會零星出現。


<a name="which-branch"></a>
## 該選擇哪個分支？

**所有**錯誤修復都應提交至支援錯誤修復的最新版本（目前為 `13.x`）。除非修復的是僅存在於即將發布版本中的功能，否則錯誤修復**絕對不要**提交至 `master` 分支。

與目前版本**完全向下相容**的**次要**功能，可以提交至最新的穩定分支（目前為 `13.x`）。

**重大**新功能或包含破壞性變更 (breaking changes) 的功能，應始終提交至包含即將發布版本的 `master` 分支。


<a name="compiled-assets"></a>
## 編譯後的資源

如果您提交的變更會影響編譯後的檔案（例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 內的大部分檔案），請不要提交編譯後的檔案。由於檔案體積較大，維護者無法實際對其進行審查。這可能會被利用來將惡意程式碼注入到 Laravel 中。為了防禦性地防止此情況，所有編譯後的檔案都將由 Laravel 維護者生成並提交。


<a name="ai-generated-contributions"></a>
## AI 生成的貢獻

我們感謝提交給 Laravel 的每一個 Pull Request。然而，主要由 AI 生成且缺乏深思熟慮的人工審查與考量的貢獻是不被接受的。

如果您選擇使用 AI 工具來輔助您的貢獻，在提交之前，您**必須**對生成的程式碼進行徹底的審查、測試並完全理解其內容。

**我們不會容忍大量開啟完全由 AI 生成的 Issue 或 Pull Request。** 這樣的 Pull Request 將在不經審查的情況下被關閉，且貢獻者可能會被封鎖於儲存庫之外。

我們鼓勵貢獻者熟悉現有的程式碼庫、參與社群，並提交反映自身理解且經過深思熟慮解決問題的 Pull Request。


<a name="security-vulnerabilities"></a>
## 安全性漏洞

如果您在 Laravel 中發現安全性漏洞，請發送電子郵件給 Taylor Otwell (<a href="mailto:taylor@laravel.com">taylor@laravel.com</a>)。所有安全性漏洞都將被迅速處理。

<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 程式碼標準與 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 文件區塊範例。請注意，`@param` 屬性後方接兩個空格、引數型別、再接兩個空格，最後才是變數名稱：

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

當由於使用原生型別而導致 `@param` 或 `@return` 屬性變得多餘時，可以將其移除：

```php
/**
 * Execute the job.
 * [tl! remove]
 * @return void [tl! remove]
 */
public function handle(AudioProcessor $processor): void
{
    // ...
}
```

然而，當原生型別為泛型時，請透過 `@param` 或 `@return` 屬性來指定泛型型別：

```php
/**
 * Get the attachments for the message.
 * [tl! add]
 * @return array<int, \Illuminate\Mail\Mailables\Attachment> [tl! add]
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

不必擔心您的程式碼風格不夠完美！[StyleCI](https://styleci.io/) 會在 pull requests 合併後，自動將任何風格修正合併到 Laravel 儲存庫中。這讓我們能將重心放在貢獻的內容而非程式碼風格上。


<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則衍生自 Ruby 的行為準則。任何違反行為準則的行為都可以回報給 Taylor Otwell (taylor@laravel.com)：

<div class="content-list" markdown="1">

- 參與者應對對立觀點保持包容。
- 參與者必須確保其語言和行為不包含人身攻擊或貶低他人的評論。
- 在解讀他人的言行時，參與者應始終假設對方抱持善意。
- 任何被合理認定為騷擾的行為都將不被容忍。

</div>