# 發行說明

- [版本控制方案](#versioning-scheme)
- [支援政策](#support-policy)
- [Laravel 12](#laravel-12)

<a name="versioning-scheme"></a>
## 版本控制方案

Laravel 及其其他第一方套件遵循 [語義化版本控制](https://semver.org)。框架主要發行版每年（約第一季度）發布，而次要及補丁發行版則可能頻繁至每週發布。次要及補丁發行版**絕不**應包含破壞性變更。

當從您的應用程式或套件中參考 Laravel 框架或其組件時，您應始終使用諸如 `^12.0` 的版本約束，因為 Laravel 的主要發行版確實包含破壞性變更。然而，我們致力於確保您可以在一天或更短的時間內更新到新的主要發行版。

<a name="named-arguments"></a>
#### 具名引數

[具名引數](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments) 不在 Laravel 的向後相容性準則範圍內。我們可能會選擇在必要時重新命名函數引數，以改進 Laravel 程式碼庫。因此，在呼叫 Laravel 方法時使用具名引數應謹慎進行，並了解參數名稱未來可能會改變。

<a name="support-policy"></a>
## 支援政策

對於所有 Laravel 發行版，提供 18 個月的錯誤修復和 2 年的安全性修復。對於所有額外的函式庫，只有最新的主要發行版會收到錯誤修復。此外，請查閱 [Laravel 支援的資料庫版本](/docs/{{version}}/database#introduction)。

<div class="overflow-auto">

| 版本 | PHP (*)   | 發行日期            | 錯誤修復期限      | 安全性修復期限     |
| ---- | --------- | ------------------- | ------------------- | -------------------- |
| 10   | 8.1 - 8.3 | February 14th, 2023 | August 6th, 2024    | February 4th, 2025   |
| 11   | 8.2 - 8.4 | March 12th, 2024    | September 3rd, 2025 | March 12th, 2026     |
| 12   | 8.2 - 8.4 | February 24th, 2025 | August 13th, 2026   | February 24th, 2027  |
| 13   | 8.3 - 8.4 | Q1 2026             | Q3 2027             | Q1 2028              |

</div>

<div class="version-colors">
    <div class="end-of-life">
        <div class="color-box"></div>
        <div>生命週期結束</div>
    </div>
    <div class="security-fixes">
        <div class="color-box"></div>
        <div>僅安全性修復</div>
    </div>
</div>

(*) 支援的 PHP 版本

<a name="laravel-12"></a>
## Laravel 12

Laravel 12 延續了 Laravel 11.x 中的改進，透過更新上游依賴項，並引入了用於 React、Vue 和 Livewire 的全新入門套件，其中包括使用 [WorkOS AuthKit](https://authkit.com) 進行使用者身份驗證的選項。我們的入門套件的 WorkOS 變體提供社交身份驗證、通行金鑰和 SSO 支援。

<a name="minimal-breaking-changes"></a>
### 最少破壞性變更

我們在此發行週期中的大部分重點一直在盡量減少破壞性變更。相反地，我們致力於全年發布持續的生活品質改進，這些改進不會破壞現有應用程式。

因此，Laravel 12 發行版是一個相對次要的「維護發行版」，目的是為了升級現有依賴項。鑑於此，大多數 Laravel 應用程式可以在不更改任何應用程式程式碼的情況下升級到 Laravel 12。

<a name="new-application-starter-kits"></a>
### 新應用程式入門套件

Laravel 12 引入了用於 React、Vue 和 Livewire 的全新 [應用程式入門套件](/docs/{{version}}/starter-kits)。React 和 Vue 入門套件利用 Inertia 2、TypeScript、[shadcn/ui](https://ui.shadcn.com) 和 Tailwind，而 Livewire 入門套件則利用基於 Tailwind 的 [Flux UI](https://fluxui.dev) 組件函式庫和 Laravel Volt。

React、Vue 和 Livewire 入門套件都利用了 Laravel 內建的身份驗證系統，以提供登入、註冊、密碼重設、電子郵件驗證等功能。此外，我們正在引入每個入門套件的 [由 WorkOS AuthKit 提供支援的](https://authkit.com) 變體，提供社交身份驗證、通行金鑰和 SSO 支援。WorkOS 為每月活躍用戶高達 100 萬的應用程式提供免費身份驗證。

隨著我們新的應用程式入門套件的引入，Laravel Breeze 和 Laravel Jetstream 將不再收到額外更新。

若要開始使用我們的新入門套件，請查閱 [入門套件文件](/docs/{{version}}/starter-kits)。