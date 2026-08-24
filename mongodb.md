# MongoDB

- [簡介](#introduction)
- [安裝](#installation)
    - [MongoDB 驅動程式](#mongodb-driver)
    - [啟動 MongoDB 伺服器](#starting-a-mongodb-server)
    - [安裝 Laravel MongoDB 套件](#install-the-laravel-mongodb-package)
- [設定](#configuration)
- [功能特色](#features)

<a name="introduction"></a>
## 簡介

[MongoDB](https://www.mongodb.com/resources/products/fundamentals/why-use-mongodb) 是最熱門的 NoSQL 文件導向資料庫之一，因其具備高寫入負載能力（適用於分析或物聯網）以及高可用性（可輕鬆設定具有自動故障轉移的副本集）而被廣泛使用。它還可以輕鬆地對資料庫進行分片（shard）以實現水平擴充，並擁有強大的查詢語言，可用於執行聚合、文字搜尋或地理空間查詢。

MongoDB 資料庫中的每筆紀錄不是像 SQL 資料庫那樣儲存在列或欄組成的資料表中，而是以 BSON 描述的文件（Document），這是一種資料的二進位表示法。應用程式接著可以 JSON 格式取得這些資訊。它支援多種資料類型，包括文件、陣列、內嵌文件與二進位資料。

在將 MongoDB 與 Laravel 一起使用之前，我們建議透過 Composer 安裝並使用 `mongodb/laravel-mongodb` 套件。`laravel-mongodb` 套件由 MongoDB 官方維護，雖然 PHP 透過 MongoDB 驅動程式原生支援 MongoDB，但 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 套件提供了與 Eloquent 及其他 Laravel 功能更豐富的整合：

```shell
composer require mongodb/laravel-mongodb
```


<a name="installation"></a>
## 安裝


<a name="mongodb-driver"></a>
### MongoDB 驅動程式

要連線至 MongoDB 資料庫，需要 `mongodb` PHP 擴充套件。若您使用 [Laravel Herd](https://herd.laravel.com) 進行本機開發，或者透過 `php.new` 安裝 PHP，您的系統中已經安裝了此擴充套件。然而，若您需要手動安裝該擴充套件，可以透過 PECL 進行安裝：

```shell
pecl install mongodb
```

關於安裝 MongoDB PHP 擴充套件的更多資訊，請參考 [MongoDB PHP 擴充套件安裝說明](https://www.php.net/manual/en/mongodb.installation.php)。


<a name="starting-a-mongodb-server"></a>
### 啟動 MongoDB 伺服器

MongoDB Community Server 可用於在本機執行 MongoDB，並支援安裝於 Windows、macOS、Linux 或作為 Docker 容器執行。若要瞭解如何安裝 MongoDB，請參考[官方 MongoDB Community 安裝指南](https://docs.mongodb.com/manual/administration/install-community/)。

可以在 `.env` 檔案中設定 MongoDB 伺服器的連線字串：

```ini
MONGODB_URI="mongodb://localhost:27017"
MONGODB_DATABASE="laravel_app"
```

若要在雲端託管 MongoDB，可以考慮使用 [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)。
若要從應用程式在本機存取 MongoDB Atlas 叢集，您需要[在叢集的網路設定中將您自己的 IP 位址新增](https://www.mongodb.com/docs/atlas/security/add-ip-address-to-list/)至專案的 IP 存取清單（IP Access List）中。

MongoDB Atlas 的連線字串也可以在 `.env` 檔案中設定：

```ini
MONGODB_URI="mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<dbname>?retryWrites=true&w=majority"
MONGODB_DATABASE="laravel_app"
```


<a name="install-the-laravel-mongodb-package"></a>
### 安裝 Laravel MongoDB 套件

最後，使用 Composer 安裝 Laravel MongoDB 套件：

```shell
composer require mongodb/laravel-mongodb
```

> [!NOTE]
> 若未安裝 `mongodb` PHP 擴充套件，此套件的安裝將會失敗。CLI 與 Web 伺服器之間的 PHP 設定可能有所不同，因此請確保在這兩種設定中皆已啟用該擴充套件。


<a name="configuration"></a>
## 設定

您可以透過應用程式的 `config/database.php` 設定檔來設定 MongoDB 連線。在此檔案中，新增一個使用 `mongodb` 驅動程式的 `mongodb` 連線：

```php
'connections' => [
    'mongodb' => [
        'driver' => 'mongodb',
        'dsn' => env('MONGODB_URI', 'mongodb://localhost:27017'),
        'database' => env('MONGODB_DATABASE', 'laravel_app'),
    ],
],
```


<a name="features"></a>
## 功能特色

完成設定後，您可以在應用程式中使用 `mongodb` 套件與資料庫連線，以充分利用各種強大的功能：

- [使用 Eloquent](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/eloquent-models/)，模型可以儲存在 MongoDB 集合（Collection）中。除了標準的 Eloquent 功能外，Laravel MongoDB 套件還提供了額外功能，例如內嵌關聯（Embedded relationships）。該套件還提供了對 MongoDB 驅動程式的直接存取，可用於執行原生查詢與聚合管道（Aggregation pipelines）等操作。
- 使用查詢生成器[撰寫複雜的查詢](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/query-builder/)。
- 使用向量嵌入與 `vectorSearch` Eloquent 方法進行[相似度 / 向量搜尋](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/fundamentals/vector-search/)。
- `mongodb` [快取驅動程式](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/)經過最佳化，可利用 TTL 索引等 MongoDB 功能自動清除過期的快取項目。
- 使用 `mongodb` 佇列驅動程式[分發與處理佇列任務](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/)。
- 透過 [Flysystem 的 GridFS 轉接器](https://flysystem.thephpleague.com/docs/adapter/gridfs/)[將檔案儲存至 GridFS 中](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/filesystems/)。
- 使用 `mongodb` Scout 引擎進行[全文檢索](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/scout/)。
- 大多數使用資料庫連線或 Eloquent 的第三方套件皆可與 MongoDB 一起使用。

若要繼續學習如何使用 MongoDB 與 Laravel，請參考 MongoDB 的[快速入門指南](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/quick-start/)。