# 貢獻指南

- [錯誤回報](#bug-reports)
- [技術支援問題](#support-questions)
- [核心開發討論](#core-development-discussion)
- [要選擇哪個分支？](#which-branch)
- [編譯後的靜態資源](#compiled-assets)
- [AI 生成的貢獻](#ai-generated-contributions)
- [資安漏洞](#security-vulnerabilities)
- [程式碼風格](#coding-style)
    - [PHPDoc](#phpdoc)
    - [StyleCI](#styleci)
- [行為準則](#code-of-conduct)

<a name="bug-reports"></a>
## 錯誤回報

為了鼓勵積極合作，Laravel 強烈建議提交 Pull Request，而非僅僅是回報 Bug。Pull Request 只有在標示為「準備好進行審查 (ready for review)」（而非「草稿 (draft)」狀態）且新功能的所有測試皆通過時，才會被審查。若 Pull Request 長時間停留在「草稿」狀態且沒有積極更新，將會在幾天後被關閉。

然而，如果您提交了錯誤回報，您的 Issue 應包含標題以及對問題的清晰描述。您還應該包含盡可能多的相關資訊以及能夠重現該問題的範例程式碼。錯誤回報的目標是為了讓您自己與其他人能夠輕鬆重現該 Bug 並開發修復方案。

請記住，建立錯誤回報是希望其他遇到相同問題的人能夠與您一同合作解決問題。請不要期望錯誤回報會自動獲得處理，或是其他人會主動幫您修復。建立錯誤回報是幫助您與其他人踏出解決問題第一步的方法。如果您想盡一份心力，可以透過修復[我們 Issue 追蹤器中列出的任何 Bug](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel) 來提供協助。您必須登入 GitHub 驗證身份才能查看 Laravel 的所有 Issue。

在使用 Laravel 時，如果您注意到不正確的 DocBlock、PHPStan 或 IDE 警告，請勿建立 GitHub Issue。相反地，請直接提交 Pull Request 來修復該問題。

Laravel 的原始碼託管於 GitHub 上，且每個 Laravel 專案都有各自的儲存庫：

<div class="content-list" markdown="1">

- [Laravel AI SDK](https://github.com/laravel/ai)
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
- [Laravel Livewire 入門套件](https://github.com/laravel/livewire-starter-kit)
- [Laravel React 入門套件](https://github.com/laravel/react-starter-kit)
- [Laravel Svelte 入門套件](https://github.com/laravel/svelte-starter-kit)
- [Laravel Vue 入門套件](https://github.com/laravel/vue-starter-kit)

</div>


<a name="support-questions"></a>
## 技術支援問題

Laravel 的 GitHub Issue 追蹤器並非用於提供 Laravel 的協助或技術支援。請改為使用以下管道：

<div class="content-list" markdown="1">

- [GitHub Discussions](https://github.com/laravel/framework/discussions)
- [Laracasts 論壇](https://laracasts.com/discuss)
- [Laravel.io 論壇](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

</div>


<a name="core-development-discussion"></a>
## 核心開發討論

您可以在 Laravel 框架儲存庫的 [GitHub 討論區](https://github.com/laravel/framework/discussions) 中提議新功能或改進現有的 Laravel 行為。如果您提議新功能，請願意至少實作完成該功能所需的部分程式碼。

關於 Bug、新功能以及現有功能實作的非正式討論，會在 [Laravel Discord 伺服器](https://discord.gg/laravel) 的 `#internals` 頻道中進行。Laravel 的維護者 Taylor Otwell 通常會在工作日的上午 8 點至下午 5 點（UTC-06:00 或美中時間）出現在該頻道中，其他時間也會不定時出現。


<a name="which-branch"></a>
## 要選擇哪個分支？

**所有** Bug 修復都應該發送到支援修復 Bug 的最新版本（目前為 `13.x`）。除非修復的是僅存在於即將發布版本中的功能，否則修復 Bug 的 PR **絕不**應該發送到 `master` 分支。

**次要**功能若能**完全向下相容**現有版本，可以發送到最新的穩定分支（目前為 `13.x`）。

**主要**的新功能或包含破壞性變更的功能，則應始終發送到包含即將發布版本的 `master` 分支。


<a name="compiled-assets"></a>
## 編譯後的靜態資源

如果您提交的變更會影響編譯後的檔案，例如 `laravel/laravel` 儲存庫中 `resources/css` 或 `resources/js` 目錄下的多數檔案，請不要 commit 編譯後的檔案。由於這些檔案體積龐大，維護者實際上無法進行審查。這可能會被利用來作為向 Laravel 注入惡意程式碼的手法。為了進行防禦性預防，所有編譯後的檔案都將由 Laravel 的維護者統一生成並 commit。


<a name="ai-generated-contributions"></a>
## AI 生成的貢獻

我們感謝提交給 Laravel 的每一個 Pull Request。然而，主要由 AI 生成且未經過人類深思熟慮與審查的貢獻是不被接受的。

如果您選擇使用 AI 工具來協助您的貢獻，在提交之前，您**必須**親自對產出的程式碼進行深入的審查、測試與理解。

**絕不允許大量建立完全由 AI 生成的 Issue 或 Pull Request。** 此類 Pull Request 將會在未經審查的情況下直接關閉，且貢獻該內容的使用者可能會被封鎖於儲存庫之外。

我們鼓勵貢獻者熟悉現有的程式碼庫、與社群互動，並提交能夠反映自己對所解決問題的理解與仔細考量的 Pull Request。


<a name="security-vulnerabilities"></a>
## 資安漏洞

如果您在 Laravel 中發現資安漏洞，請發送電子郵件至資安團隊信箱 <a href="mailto:security@laravel.com">security@laravel.com</a>。所有資安漏洞都將會獲得及時處理。

<a name="coding-style"></a>
## 程式碼風格

Laravel 遵循 [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md) 程式碼風格標準以及 [PSR-4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md) 自動載入標準。


<a name="phpdoc"></a>
### PHPDoc

以下是一個有效的 Laravel 文件塊 (DocBlock) 範例。請注意，在 `@param` 屬性後面跟著兩個空格、引數型態、再兩個空格，最後才是變數名稱：

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

當 `@param` 或 `@return` 屬性因為使用了原生型別而顯得重複多餘時，可以將它們移除：

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

然而，當原生型別為泛型時，請透過使用 `@param` 或 `@return` 屬性來指定泛型型別：

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

如果您的程式碼風格不夠完美，請不用擔心！在 Pull Request 被合併後，[StyleCI](https://styleci.io/) 會自動將任何風格修復併入 Laravel 的儲存庫中。這能讓我們將精力專注於貢獻的內容本身，而不是程式碼風格。


<a name="code-of-conduct"></a>
## 行為準則

Laravel 的行為準則源自於 Ruby 的行為準則。任何違反行為準則的情況都可以向 Taylor Otwell (taylor@laravel.com) 回報：

<div class="content-list" markdown="1">

- 參與者應包容不同的觀點。
- 參與者必須確保其言語和行為沒有個人攻擊和貶低個人的言論。
- 在解讀他人的言語和行為時，參與者應始終假設對方出於善意。
- 任何合理情況下被視為騷擾的行為都將絕不被允許。

</div>