# Laravel AI SDK

- [簡介](#introduction)
- [安裝](#installation)
    - [設定](#configuration)
    - [自訂 Base URL](#custom-base-urls)
    - [支援的提供者](#provider-support)
- [AI 代理](#agents)
    - [進行提示](#prompting)
    - [對話上下文](#conversation-context)
    - [結構化輸出](#structured-output)
    - [附件](#attachments)
    - [串流](#streaming)
    - [廣播](#broadcasting)
    - [佇列](#queueing)
    - [工具](#tools)
    - [提供者工具](#provider-tools)
    - [子代理](#sub-agents)
    - [中介層](#middleware)
    - [匿名代理](#anonymous-agents)
    - [AI 代理設定](#agent-configuration)
    - [提供者選項](#provider-options)
- [圖片](#images)
- [語音 (TTS)](#audio)
- [語音轉文字 (STT)](#transcription)
- [嵌入向量](#embeddings)
    - [查詢嵌入向量](#querying-embeddings)
    - [快取嵌入向量](#caching-embeddings)
- [重新排序 (Reranking)](#reranking)
- [檔案](#files)
- [向量儲存庫](#vector-stores)
    - [將檔案新增至儲存庫](#adding-files-to-stores)
- [容錯移轉](#failover)
- [測試](#testing)
    - [AI 代理](#testing-agents)
    - [圖片](#testing-images)
    - [語音](#testing-audio)
    - [語音轉文字](#testing-transcriptions)
    - [嵌入向量](#testing-embeddings)
    - [重新排序](#testing-reranking)
    - [檔案](#testing-files)
    - [向量儲存庫](#testing-vector-stores)
- [事件](#events)

<a name="introduction"></a>
## 簡介

[Laravel AI SDK](https://github.com/laravel/ai) 提供了一個統一且具表達力的 API，用於與 OpenAI、Anthropic、Gemini 等 AI 提供者進行互動。透過 AI SDK，您可以使用一致且對 Laravel 友善的介面來建立具備工具與結構化輸出的智慧型 AI 代理、產生圖片、合成與轉錄語音、建立向量嵌入，以及更多功能。


<a name="installation"></a>
## 安裝

您可以使用 Composer 安裝 Laravel AI SDK：

```shell
composer require laravel/ai
```

接著，您應該使用 `vendor:publish` Artisan 指令發布 AI SDK 的設定檔與遷移檔：

```shell
php artisan vendor:publish --provider="Laravel\Ai\AiServiceProvider"
```

最後，您應該執行應用程式的資料庫遷移。這將會建立 `agent_conversations` 與 `agent_conversation_messages` 資料表，AI SDK 會使用這些資料表來支援其對話儲存功能：

```shell
php artisan migrate
```


<a name="configuration"></a>
### 設定

您可以在應用程式的 `config/ai.php` 設定檔中定義您的 AI 提供者憑證，或者在應用程式的 `.env` 檔案中將其定義為環境變數：

```ini
ANTHROPIC_API_KEY=
AZURE_OPENAI_API_KEY=
COHERE_API_KEY=
DEEPSEEK_API_KEY=
ELEVENLABS_API_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
MISTRAL_API_KEY=
OLLAMA_API_KEY=
OPENAI_API_KEY=
OPENROUTER_API_KEY=
JINA_API_KEY=
VOYAGEAI_API_KEY=
XAI_API_KEY=
```

用於文字、圖片、語音、語音轉文字和嵌入向量的預設模型，也可以在應用程式的 `config/ai.php` 設定檔中進行設定。


<a name="custom-base-urls"></a>
### 自訂 Base URL

預設情況下，Laravel AI SDK 會直接連線到每個提供者的公開 API 端點。然而，您可能需要透過不同的端點來路由請求——例如，當使用代理服務來集中管理 API 金鑰、實作速率限制，或是透過企業網關來路由流量時。

您可以透過在提供者設定中新增 `url` 參數來設定自訂的 Base URL：

```php
'providers' => [
    'openai' => [
        'driver' => 'openai',
        'key' => env('OPENAI_API_KEY'),
        'url' => env('OPENAI_BASE_URL'),
    ],

    'anthropic' => [
        'driver' => 'anthropic',
        'key' => env('ANTHROPIC_API_KEY'),
        'url' => env('ANTHROPIC_BASE_URL'),
    ],
],
```

這在透過代理服務（例如 LiteLLM 或 Azure OpenAI Gateway）路由請求或使用其他替代端點時非常有用。

以下提供者支援自訂 Base URL：OpenAI、Anthropic、Gemini、Groq、Cohere、DeepSeek、xAI 以及 OpenRouter。


<a name="provider-support"></a>
### 支援的提供者

AI SDK 的各項功能支援多種不同的提供者。下表總結了每個功能可使用的提供者：

| 功能 | 提供者 |
|---|---|
| 文字 | OpenAI, Anthropic, Gemini, Azure, Bedrock, Groq, xAI, DeepSeek, Mistral, Ollama, OpenRouter |
| 圖片 | OpenAI, Gemini, xAI, Azure, Bedrock, OpenRouter |
| TTS | OpenAI, ElevenLabs, Gemini |
| STT | OpenAI, ElevenLabs, Mistral, Gemini |
| 嵌入向量 | OpenAI, Gemini, Azure, Bedrock, Cohere, Mistral, Jina, VoyageAI, Ollama, OpenRouter |
| 重新排序 | Cohere, Jina, VoyageAI |
| 檔案 | OpenAI, Anthropic, Gemini |

您可以使用 `Laravel\Ai\Enums\Lab` 列舉在程式碼中引用提供者，而不需要使用純字串：

```php
use Laravel\Ai\Enums\Lab;

Lab::Anthropic;
Lab::OpenAI;
Lab::Gemini;
// ...
```

<a name="agents"></a>
## AI 代理

「AI 代理 (Agents)」是 Laravel AI SDK 中與 AI 提供者進行互動的基本建構單元。每個 AI 代理都是一個專屬的 PHP 類別，它封裝了與大型語言模型互動所需的指令、對話上下文、工具和輸出綱要 (Schema)。您可以將 AI 代理視為一個專門的助手——例如銷售教練、文件分析師或客服機器人——您只需設定一次，便能在整個應用程式中根據需要向其發送提示 (Prompt)。

您可以透過 `make:agent` Artisan 指令來建立 AI 代理：

```shell
php artisan make:agent SalesCoach

php artisan make:agent SalesCoach --structured
```

在產生的 AI 代理類別中，您可以定義系統提示詞 / 指令、訊息上下文、可用工具以及輸出綱要 (若適用)：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Tools\RetrievePreviousTranscripts;
use App\Models\History;
use App\Models\User;
use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\Conversational;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Messages\Message;
use Laravel\Ai\Promptable;
use Stringable;

class SalesCoach implements Agent, Conversational, HasTools, HasStructuredOutput
{
    use Promptable;

    public function __construct(public User $user) {}

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): Stringable|string
    {
        return 'You are a sales coach, analyzing transcripts and providing feedback and an overall sales strength score.';
    }

    /**
     * Get the list of messages comprising the conversation so far.
     */
    public function messages(): iterable
    {
        return History::where('user_id', $this->user->id)
            ->latest()
            ->limit(50)
            ->get()
            ->reverse()
            ->map(function ($message) {
                return new Message($message->role, $message->content);
            })->all();
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new RetrievePreviousTranscripts,
        ];
    }

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'feedback' => $schema->string()->required(),
            'score' => $schema->integer()->min(1)->max(10)->required(),
        ];
    }
}
```

<a name="prompting"></a>
### 進行提示

要向 AI 代理發送提示，首先使用 `make` 方法或標準的實例化方式建立實例，然後呼叫 `prompt`：

```php
$response = (new SalesCoach)
    ->prompt('Analyze this sales transcript...');

return (string) $response;
```

`make` 方法會自容器中解析您的 AI 代理，從而支援自動的依賴注入。您也可以將引數傳遞給 AI 代理的建構子：

```php
$agent = SalesCoach::make(user: $user);
```

透過傳遞額外的引數給 `prompt` 方法，您可以在提示時覆寫預設的提供者、模型或 HTTP 逾時時間：

```php
$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: Lab::Anthropic,
    model: 'claude-haiku-4-5-20251001',
    timeout: 120,
);
```

<a name="conversation-context"></a>
### 對話上下文

如果您的 AI 代理實作了 `Conversational` 介面，您可以使用 `messages` 方法來傳回先前的對話上下文 (若適用)：

```php
use App\Models\History;
use Laravel\Ai\Messages\Message;

/**
 * Get the list of messages comprising the conversation so far.
 */
public function messages(): iterable
{
    return History::where('user_id', $this->user->id)
        ->latest()
        ->limit(50)
        ->get()
        ->reverse()
        ->map(function ($message) {
            return new Message($message->role, $message->content);
        })->all();
}
```

<a name="remembering-conversations"></a>
#### 記住對話

> **注意：** 在使用 `RemembersConversations` trait 之前，您應該使用 `vendor:publish` Artisan 指令發布並執行 AI SDK 的資料庫遷移。這些資料庫遷移將會建立用來儲存對話所需的資料庫資料表。

如果您希望 Laravel 自動為您的 AI 代理儲存與讀取對話歷史紀錄，您可以使用 `RemembersConversations` trait。此 trait 提供了一種簡單的方法來將對話訊息持久化儲存到資料庫中，而不需要手動實作 `Conversational` 介面：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Concerns\RemembersConversations;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\Conversational;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, Conversational
{
    use Promptable, RemembersConversations;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You are a sales coach...';
    }
}
```

要為使用者開始一段新的對話，請在提示之前呼叫 `forUser` 方法：

```php
$response = (new SalesCoach)->forUser($user)->prompt('Hello!');

$conversationId = $response->conversationId;
```

對話 ID 會包含在回應中傳回，您可以將其儲存起來以便日後參考。如果您想使用 Eloquent 取得某個使用者的所有對話，可以將 `HasConversations` trait 新增到您的使用者模型中：

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Ai\Concerns\HasConversations;

class User extends Authenticatable
{
    use HasConversations;
}
```

將該 trait 新增至您的模型後，您便可以透過 `conversations` 關聯來取得並查詢該使用者的對話歷史紀錄：

```php
$conversations = $user->conversations()
    ->latest('updated_at')
    ->paginate(20);
```

要繼續進行現有的對話，請使用 `continue` 方法：

```php
$response = (new SalesCoach)
    ->continue($conversationId, as: $user)
    ->prompt('Tell me more about that.');
```

當使用 `RemembersConversations` trait 時，系統會在提示時自動載入先前的訊息並將其納入對話上下文。每次互動後，新的訊息 (包括使用者和助理) 都會被自動儲存。

<a name="structured-output"></a>
### 結構化輸出

如果您希望您的 AI 代理返回結構化輸出，請實作 `HasStructuredOutput` 介面，這需要您的 AI 代理定義一個 `schema` 方法：

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasStructuredOutput
{
    use Promptable;

    // ...

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'score' => $schema->integer()->required(),
        ];
    }
}
```

當對返回結構化輸出的 AI 代理進行提示時，您可以像存取陣列一樣存取傳回的 `StructuredAgentResponse`：

```php
$response = (new SalesCoach)->prompt('Analyze this sales transcript...');

return $response['score'];
```


<a name="structured-output-nested-objects"></a>
#### 巢狀物件

若要定義巢狀的結構化輸出，請使用 `object` 方法並傳入一個閉包 (Closure)：

```php
<?php

namespace App\Ai\Agents;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasStructuredOutput;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasStructuredOutput
{
    use Promptable;

    // ...

    /**
     * Get the agent's structured output schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'score' => $schema->integer()->required(),
            'metadata' => $schema->object(fn ($schema) => [
                'confidence' => $schema->string()->enum(['low', 'medium', 'high'])->required(),
                'language' => $schema->string()->required(),
            ])->required(),
        ];
    }
}
```


<a name="structured-output-arrays-of-objects"></a>
#### 物件陣列

如果您的 AI 代理應該返回一個結構化項目的列表，請結合使用 `array` 與 `object` 方法：

```php
public function schema(JsonSchema $schema): array
{
    return [
        'feedback' => $schema->array()
            ->items(
                $schema->object(fn ($schema) => [
                    'comment' => $schema->string()->required(),
                    'score' => $schema->integer()->required(),
                ])
            )
            ->required(),
    ];
}
```


<a name="attachments"></a>
### 附件

在進行提示時，您也可以在提示詞中傳入附件，以便讓模型檢查圖片和文件：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Files;

$response = (new SalesCoach)->prompt(
    'Analyze the attached sales transcript...',
    attachments: [
        Files\Document::fromStorage('transcript.pdf') // Attach a document from a filesystem disk...
        Files\Document::fromPath('/home/laravel/transcript.md') // Attach a document from a local path...
        $request->file('transcript'), // Attach an uploaded file...
    ]
);
```

同樣地，`Laravel\Ai\Files\Image` 類別也可用於將圖片附加到提示詞中：

```php
use App\Ai\Agents\ImageAnalyzer;
use Laravel\Ai\Files;

$response = (new ImageAnalyzer)->prompt(
    'What is in this image?',
    attachments: [
        Files\Image::fromStorage('photo.jpg') // Attach an image from a filesystem disk...
        Files\Image::fromPath('/home/laravel/photo.jpg') // Attach an image from a local path...
        $request->file('photo'), // Attach an uploaded file...
    ]
);
```


<a name="streaming"></a>
### 串流

您可以透過呼叫 `stream` 方法來對 AI 代理的處理結果進行串流。傳回的 `StreamableAgentResponse` 可以直接從路由中返回，以自動向客戶端發送伺服器傳送事件 (SSE) 的串流回應：

```php
use App\Ai\Agents\SalesCoach;

Route::get('/coach', function () {
    return (new SalesCoach)->stream('Analyze this sales transcript...');
});
```

可以使用 `then` 方法來提供一個閉包，該閉包將在整個回應成功串流至客戶端後被調用：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Responses\StreamedAgentResponse;

Route::get('/coach', function () {
    return (new SalesCoach)
        ->stream('Analyze this sales transcript...')
        ->then(function (StreamedAgentResponse $response) {
            // $response->text, $response->events, $response->usage...
        });
});
```

或者，您也可以手動逐一讀取串流事件：

```php
$stream = (new SalesCoach)->stream('Analyze this sales transcript...');

foreach ($stream as $event) {
    // ...
}
```


<a name="streaming-using-the-vercel-ai-sdk-protocol"></a>
#### 使用 Vercel AI SDK 協定進行串流

您可以透過在可串流回應上呼叫 `usingVercelDataProtocol` 方法，來使用 [Vercel AI SDK 串流協定](https://ai-sdk.dev/docs/ai-sdk-ui/stream-protocol) 進行事件串流：

```php
use App\Ai\Agents\SalesCoach;

Route::get('/coach', function () {
    return (new SalesCoach)
        ->stream('Analyze this sales transcript...')
        ->usingVercelDataProtocol();
});
```


<a name="broadcasting"></a>
### 廣播

您可以用幾種不同的方式來廣播串流事件。第一種，您可以簡單地在串流事件上呼叫 `broadcast` 或 `broadcastNow` 方法：

```php
use App\Ai\Agents\SalesCoach;
use Illuminate\Broadcasting\Channel;

$stream = (new SalesCoach)->stream('Analyze this sales transcript...');

foreach ($stream as $event) {
    $event->broadcast(new Channel('channel-name'));
}
```

或者，您您可以呼叫 AI 代理的 `broadcastOnQueue` 方法，將 AI 代理的運作放入佇列中，並在串流事件可用時進行廣播：

```php
(new SalesCoach)->broadcastOnQueue(
    'Analyze this sales transcript...'
    new Channel('channel-name'),
);
```


<a name="queueing"></a>
### 佇列

透過使用 AI 代理的 `queue` 方法，您可以對該 AI 代理進行提示，但允許它在背景處理回應，使您的應用程式保持流暢與即時回應。可以使用 `then` 和 `catch` 方法來註冊閉包，這些閉包將在回應可用或發生異常時被調用：

```php
use Illuminate\Http\Request;
use Laravel\Ai\Responses\AgentResponse;
use Throwable;

Route::post('/coach', function (Request $request) {
    (new SalesCoach)
        ->queue($request->input('transcript'))
        ->then(function (AgentResponse $response) {
            // ...
        })
        ->catch(function (Throwable $e) {
            // ...
        });

    return back();
});
```

<a name="tools"></a>
### 工具

工具可以用來賦予 AI 代理額外的功能，讓它們在回應提示詞時可以使用。可以使用 `make:tool` Artisan 指令來建立工具：

```shell
php artisan make:tool RandomNumberGenerator
```

產生的工具會被放置在應用程式的 `app/Ai/Tools` 目錄中。每個工具都包含一個 `handle` 方法，當 AI 代理需要使用該工具時，就會呼叫此方法：

```php
<?php

namespace App\Ai\Tools;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Tool;
use Laravel\Ai\Tools\Request;
use Stringable;

class RandomNumberGenerator implements Tool
{
    /**
     * Get the description of the tool's purpose.
     */
    public function description(): Stringable|string
    {
        return 'This tool may be used to generate cryptographically secure random numbers.';
    }

    /**
     * Execute the tool.
     */
    public function handle(Request $request): Stringable|string
    {
        return (string) random_int($request['min'], $request['max']);
    }

    /**
     * Get the tool's schema definition.
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'min' => $schema->integer()->min(0)->required(),
            'max' => $schema->integer()->required(),
        ];
    }
}
```

定義好工具後，您可以在任何 AI 代理的 `tools` 方法中將其回傳：

```php
use App\Ai\Tools\RandomNumberGenerator;

/**
 * Get the tools available to the agent.
 *
 * @return Tool[]
 */
public function tools(): iterable
{
    return [
        new RandomNumberGenerator,
    ];
}
```

<a name="similarity-search"></a>
#### 相似度搜尋

`SimilaritySearch` 工具允許 AI 代理使用儲存在資料庫中的向量嵌入 (Vector Embeddings)，來搜尋與給定查詢相似的文件。當您想讓 AI 代理存取並搜尋應用程式的資料時，這對於檢索增強生成 (RAG) 非常有用。

建立相似度搜尋工具最簡單的方法，是對擁有向量嵌入的 Eloquent 模型使用 `usingModel` 方法：

```php
use App\Models\Document;
use Laravel\Ai\Tools\SimilaritySearch;

public function tools(): iterable
{
    return [
        SimilaritySearch::usingModel(Document::class, 'embedding'),
    ];
}
```

第一個引數是 Eloquent 模型的類別，第二個引數則是包含向量嵌入的欄位。

您也可以提供一個介於 `0.0` 與 `1.0` 之間的最小相似度門檻，以及一個用來自訂查詢的閉包：

```php
SimilaritySearch::usingModel(
    model: Document::class,
    column: 'embedding',
    minSimilarity: 0.7,
    limit: 10,
    query: fn ($query) => $query->where('published', true),
),
```

若需要更多控制，您可以使用回傳搜尋結果的自訂閉包來建立相似度搜尋工具：

```php
use App\Models\Document;
use Laravel\Ai\Tools\SimilaritySearch;

public function tools(): iterable
{
    return [
        new SimilaritySearch(using: function (string $query) {
            return Document::query()
                ->where('user_id', $this->user->id)
                ->whereVectorSimilarTo('embedding', $query)
                ->limit(10)
                ->get();
        }),
    ];
}
```

您可以使用 `withDescription` 方法來建立自訂的工具描述：

```php
SimilaritySearch::usingModel(Document::class, 'embedding')
    ->withDescription('Search the knowledge base for relevant articles.'),
```

<a name="provider-tools"></a>
### 提供者工具

提供者工具是由 AI 提供者原生實現的特殊工具，提供網頁搜尋、URL 擷取和檔案搜尋等功能。與一般工具不同，提供者工具是由提供者本身執行，而非您的應用程式。

提供者工具可以由 AI 代理的 `tools` 方法回傳。

<a name="web-search"></a>
#### 網頁搜尋

`WebSearch` 提供者工具允許 AI 代理在網路上搜尋即時資訊。這對於回答有關時事、最新數據或自模型訓練截止以來可能已發生變化的主題非常有用。

**支援的提供者：** Anthropic, OpenAI, Gemini

```php
use Laravel\Ai\Providers\Tools\WebSearch;

public function tools(): iterable
{
    return [
        new WebSearch,
    ];
}
```

您可以設定網頁搜尋工具，以限制搜尋次數或將結果限制在特定網域：

```php
(new WebSearch)->max(5)->allow(['laravel.com', 'php.net']),
```

若要根據使用者位置來精確篩選搜尋結果，請使用 `location` 方法：

```php
(new WebSearch)->location(
    city: 'New York',
    region: 'NY',
    country: 'US'
);
```

<a name="web-fetch"></a>
#### 網頁擷取

`WebFetch` 提供者工具允許 AI 代理擷取並讀取網頁內容。當您需要 AI 代理分析特定的 URL 或從已知網頁中擷取詳細資訊時，這非常有用。

**支援的提供者：** Anthropic, Gemini

```php
use Laravel\Ai\Providers\Tools\WebFetch;

public function tools(): iterable
{
    return [
        new WebFetch,
    ];
}
```

您可以設定網頁擷取工具以限制擷取次數，或限制在特定網域：

```php
(new WebFetch)->max(3)->allow(['docs.laravel.com']),
```

<a name="file-search"></a>
#### 檔案搜尋

`FileSearch` 提供者工具允許 AI 代理搜尋儲存在[向量儲存庫](#vector-stores)中的[檔案](#files)。這能讓 AI 代理在您上傳的文件中搜尋相關資訊，從而實現檢索增強生成 (RAG)。

**支援的提供者：** OpenAI, Gemini

```php
use Laravel\Ai\Providers\Tools\FileSearch;

public function tools(): iterable
{
    return [
        new FileSearch(stores: ['store_id']),
    ];
}
```

您可以提供多個向量儲存庫 ID，以跨多個儲存庫進行搜尋：

```php
new FileSearch(stores: ['store_1', 'store_2']);
```

如果您的檔案有[中繼資料](#adding-files-to-stores)，您可以透過提供 `where` 引數來過濾搜尋結果。對於簡單的等值過濾，請傳入一個陣列：

```php
new FileSearch(stores: ['store_id'], where: [
    'author' => 'Taylor Otwell',
    'year' => 2026,
]);
```

對於更複雜的過濾，您可以傳入一個接收 `FileSearchQuery` 實例的閉包：

```php
use Laravel\Ai\Providers\Tools\FileSearchQuery;

new FileSearch(stores: ['store_id'], where: fn (FileSearchQuery $query) =>
    $query->where('author', 'Taylor Otwell')
        ->whereNot('status', 'draft')
        ->whereIn('category', ['news', 'updates'])
);
```

<a name="sub-agents"></a>
### 子代理

AI 代理也可以從另一個 AI 代理的 `tools` 方法中被回傳。當一個 AI 代理被作為工具回傳時，父代理可以將特定任務委派給子代理，並在回答原始提示詞時使用該子代理的回應。當通用型 AI 代理需要存取擁有專屬指令、工具、模型設定或提供者偏好設定的特化 AI 代理時，這非常有用。

例如，客戶支援 AI 代理可以將退款資格問題委派給專屬的退款 AI 代理：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Promptable;

class CustomerSupportAgent implements Agent, HasTools
{
    use Promptable;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You help customers with account, order, and billing questions. Delegate refund policy questions to the refunds specialist.';
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new RefundsAgent,
        ];
    }
}
```

若要自訂子代理如何公開給父代理，請在子代理上實作 `CanActAsTool` 介面，並定義面向工具的名稱與描述：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Tools\LookupOrder;
use Laravel\Ai\Attributes\Provider;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\CanActAsTool;
use Laravel\Ai\Contracts\HasTools;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

#[Provider(Lab::Anthropic)]
class RefundsAgent implements Agent, CanActAsTool, HasTools
{
    use Promptable;

    /**
     * Get the instructions that the agent should follow.
     */
    public function instructions(): string
    {
        return 'You are a refunds specialist. Use order details and the refund policy to give concise eligibility guidance.';
    }

    /**
     * Get the agent's tool name.
     */
    public function name(): string
    {
        return 'refunds_specialist';
    }

    /**
     * Get the agent's tool description.
     */
    public function description(): string
    {
        return 'Determine whether an order is eligible for a refund and explain the next step.';
    }

    /**
     * Get the tools available to the agent.
     *
     * @return Tool[]
     */
    public function tools(): iterable
    {
        return [
            new LookupOrder,
        ];
    }
}
```

如果子代理未實作 `CanActAsTool`，Laravel 將使用該代理的類別基礎名稱（basename）作為工具名稱，並使用一個通用描述，要求父代理傳遞清晰、獨立的任務描述。每次子代理的呼叫都是獨立執行的，且不會接收父代理的對話歷史紀錄。


<a name="middleware"></a>
### 中介層

AI 代理支援中介層，讓您能夠在提示詞發送到提供者之前進行攔截與修改。您可以使用 `make:agent-middleware` Artisan 指令來建立中介層：

```shell
php artisan make:agent-middleware LogPrompts
```

產生的中介層將會被放置在應用程式的 `app/Ai/Middleware` 目錄中。若要為 AI 代理新增中介層，請實作 `HasMiddleware` 介面，並定義一個回傳中介層類別陣列的 `middleware` 方法：

```php
<?php

namespace App\Ai\Agents;

use App\Ai\Middleware\LogPrompts;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasMiddleware;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasMiddleware
{
    use Promptable;

    // ...

    /**
     * Get the agent's middleware.
     */
    public function middleware(): array
    {
        return [
            new LogPrompts,
        ];
    }
}
```

每個中介層類別都應定義一個 `handle` 方法，該方法接收 `AgentPrompt` 以及一個用於將提示詞傳遞給下一個中介層的 `Closure`：

```php
<?php

namespace App\Ai\Middleware;

use Closure;
use Laravel\Ai\Prompts\AgentPrompt;

class LogPrompts
{
    /**
     * Handle the incoming prompt.
     */
    public function handle(AgentPrompt $prompt, Closure $next)
    {
        Log::info('Prompting agent', ['prompt' => $prompt->prompt]);

        return $next($prompt);
    }
}
```

您可以在回應上使用 `then` 方法，以便在 AI 代理完成處理後執行程式碼。這適用於同步與串流回應：

```php
public function handle(AgentPrompt $prompt, Closure $next)
{
    return $next($prompt)->then(function (AgentResponse $response) {
        Log::info('Agent responded', ['text' => $response->text]);
    });
}
```


<a name="anonymous-agents"></a>
### 匿名代理

有時您可能想在不建立專屬 AI 代理類別的情況下，快速與模型進行互動。您可以使用 `agent` 函式來建立一個即時的匿名代理：

```php
use function Laravel\Ai\{agent};

$response = agent(
    instructions: 'You are an expert at software development.',
    messages: [],
    tools: [],
)->prompt('Tell me about Laravel')
```

匿名代理也可以產生結構化輸出：

```php
use Illuminate\Contracts\JsonSchema\JsonSchema;

use function Laravel\Ai\{agent};

$response = agent(
    schema: fn (JsonSchema $schema) => [
        'number' => $schema->integer()->required(),
    ],
)->prompt('Generate a random number less than 100')
```


<a name="agent-configuration"></a>
### AI 代理設定

您可以使用 PHP 屬性（Attributes）來設定 AI 代理的文字產生選項。以下是可用的屬性：

- `MaxSteps`：AI 代理在使用工具時可以執行的最大步數。
- `MaxTokens`：模型可以產生的最大 token 數量。
- `Model`：AI 代理應使用的模型。
- `Provider`：AI 代理要使用的 AI 提供者（或用於容錯移轉的多個提供者）。
- `Temperature`：用於產生的採樣溫度（0.0 至 1.0）。
- `Timeout`：代理請求的 HTTP 逾時時間，以秒為單位（預設：60）。
- `TopP`：用於產生的核心採樣機率（0.0 至 1.0）。
- `UseCheapestModel`：使用提供者最便宜的文字模型，以進行成本優化。
- `UseSmartestModel`：針對複雜任務使用提供者最強大的文字模型。

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Attributes\MaxSteps;
use Laravel\Ai\Attributes\MaxTokens;
use Laravel\Ai\Attributes\Model;
use Laravel\Ai\Attributes\Provider;
use Laravel\Ai\Attributes\Temperature;
use Laravel\Ai\Attributes\Timeout;
use Laravel\Ai\Attributes\TopP;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

#[Provider(Lab::Anthropic)]
#[Model('claude-haiku-4-5-20251001')]
#[MaxSteps(10)]
#[MaxTokens(4096)]
#[Temperature(0.7)]
#[Timeout(120)]
#[TopP(0.9)]
class SalesCoach implements Agent
{
    use Promptable;

    // ...
}
```

`UseCheapestModel` 與 `UseSmartestModel` 屬性讓您能自動為給定的提供者選擇最符合成本效益或最強大的模型，而無需指定模型名稱。當您想要在不同的提供者之間針對成本或能力進行優化時，這非常有用：

```php
use Laravel\Ai\Attributes\UseCheapestModel;
use Laravel\Ai\Attributes\UseSmartestModel;
use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Promptable;

#[UseCheapestModel]
class SimpleSummarizer implements Agent
{
    use Promptable;

    // Will use the cheapest model (e.g., Haiku)...
}

#[UseSmartestModel]
class ComplexReasoner implements Agent
{
    use Promptable;

    // Will use the most capable model (e.g., Opus)...
}
```

> [!NOTE]
> 隨著提供者發布新模型，`UseCheapestModel` 與 `UseSmartestModel` 所選擇的底層模型可能會在 Laravel AI SDK 的不同版本之間發生變化。切換模型可能會引入行為改變、遭棄用的參數以及顯著的成本差異。如果您需要穩定、可預測的模型與定價，請使用 `Model` 屬性明確指定模型。

<a name="provider-options"></a>
### 提供者選項

如果您的 AI 代理需要傳遞特定提供者的選項（例如 OpenAI 的推理強度或懲罰設定），請實作 `HasProviderOptions` 契約(Contracts)並定義 `providerOptions` 方法：

```php
<?php

namespace App\Ai\Agents;

use Laravel\Ai\Contracts\Agent;
use Laravel\Ai\Contracts\HasProviderOptions;
use Laravel\Ai\Enums\Lab;
use Laravel\Ai\Promptable;

class SalesCoach implements Agent, HasProviderOptions
{
    use Promptable;

    // ...

    /**
     * Get provider-specific generation options.
     */
    public function providerOptions(Lab|string $provider): array
    {
        return match ($provider) {
            Lab::OpenAI => [
                'reasoning' => ['effort' => 'low'],
                'frequency_penalty' => 0.5,
                'presence_penalty' => 0.3,
            ],
            Lab::Anthropic => [
                'thinking' => ['budget_tokens' => 1024],
                'cache_control' => ['type' => 'ephemeral'],
            ],
            default => [],
        };
    }
}
```

`providerOptions` 方法會接收當前正在使用的提供者（`Lab` 列舉或字串），讓您可以針對每個提供者傳回不同的選項。這在搭配使用[容錯移轉](#failover)時特別有用，因為每個備用的提供者都可以接收各自的設定。

上面的 Anthropic 範例也透過 `cache_control` 啟用了[提示詞快取 (Prompt Caching)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)。

<a name="images"></a>
## 圖片

可以使用 `Laravel\Ai\Image` 類別來透過 `openai`、`gemini` 或 `xai` 提供者產生圖片：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')->generate();

$rawContent = (string) $image;
```

`square`、`portrait` 與 `landscape` 方法可用於控制圖片的長寬比，而 `quality` 方法可用於指引模型最終的圖片品質（`high`、`medium`、`low`）。`timeout` 方法則可用於指定以秒為單位的 HTTP 逾時時間：

```php
use Laravel\Ai\Image;

$image = Image::of('A donut sitting on the kitchen counter')
    ->quality('high')
    ->landscape()
    ->timeout(120)
    ->generate();
```

您可以使用 `attachments` 方法來附加參考圖片：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Image;

$image = Image::of('Update this photo of me to be in the style of an impressionist painting.')
    ->attachments([
        Files\Image::fromStorage('photo.jpg'),
        // Files\Image::fromPath('/home/laravel/photo.jpg'),
        // Files\Image::fromUrl('https://example.com/photo.jpg'),
        // $request->file('photo'),
    ])
    ->landscape()
    ->generate();
```

產生的圖片可以輕鬆地儲存到您應用程式的 `config/filesystems.php` 設定檔中所配置的預設磁碟中：

```php
$image = Image::of('A donut sitting on the kitchen counter');

$path = $image->store();
$path = $image->storeAs('image.jpg');
$path = $image->storePublicly();
$path = $image->storePubliclyAs('image.jpg');
```

圖片產生也可以放入佇列中執行：

```php
use Laravel\Ai\Image;
use Laravel\Ai\Responses\ImageResponse;

Image::of('A donut sitting on the kitchen counter')
    ->portrait()
    ->queue()
    ->then(function (ImageResponse $image) {
        $path = $image->store();

        // ...
    });
```


<a name="audio"></a>
## 語音 (TTS)

可以使用 `Laravel\Ai\Audio` 類別從給定的文字產生語音：

```php
use Laravel\Ai\Audio;

$audio = Audio::of('I love coding with Laravel.')->generate();

$rawContent = (string) $audio;
```

您也可以使用 Laravel 的 `Stringable` 類別所提供的 `toAudio` 方法，直接從字串產生語音：

```php
use Illuminate\Support\Str;

$audio = Str::of('I love coding with Laravel.')->toAudio();
```

`male`、`female` 與 `voice` 方法可用於決定所產生語音的聲音：

```php
$audio = Audio::of('I love coding with Laravel.')
    ->female()
    ->generate();

$audio = Audio::of('I love coding with Laravel.')
    ->voice('voice-id-or-name')
    ->generate();
```

同樣地，`instructions` 方法可用於動態指導模型所產生的語音聽起來應該如何：

```php
$audio = Audio::of('I love coding with Laravel.')
    ->female()
    ->instructions('Said like a pirate')
    ->generate();
```

產生的語音可以輕鬆地儲存到您應用程式的 `config/filesystems.php` 設定檔中所配置的預設磁碟中：

```php
$audio = Audio::of('I love coding with Laravel.')->generate();

$path = $audio->store();
$path = $audio->storeAs('audio.mp3');
$path = $audio->storePublicly();
$path = $audio->storePubliclyAs('audio.mp3');
```

語音產生也可以放入佇列中執行：

```php
use Laravel\Ai\Audio;
use Laravel\Ai\Responses\AudioResponse;

Audio::of('I love coding with Laravel.')
    ->queue()
    ->then(function (AudioResponse $audio) {
        $path = $audio->store();

        // ...
    });
```


<a name="transcription"></a>
## 語音轉文字 (STT)

可以使用 `Laravel\Ai\Transcription` 類別來產生給定語音的文字稿：

```php
use Laravel\Ai\Transcription;

$transcript = Transcription::fromPath('/home/laravel/audio.mp3')->generate();
$transcript = Transcription::fromStorage('audio.mp3')->generate();
$transcript = Transcription::fromUpload($request->file('audio'))->generate();

return (string) $transcript;
```

`diarize` 方法可用於表示除了原始文字稿之外，您還希望回應包含講者辨識 (Diarized) 的文字稿，讓您可以依說話者來存取分段的文字稿：

```php
$transcript = Transcription::fromStorage('audio.mp3')
    ->diarize()
    ->generate();
```

語音轉文字的產生也可以放入佇列中執行：

```php
use Laravel\Ai\Transcription;
use Laravel\Ai\Responses\TranscriptionResponse;

Transcription::fromStorage('audio.mp3')
    ->queue()
    ->then(function (TranscriptionResponse $transcript) {
        // ...
    });
```

<a name="embeddings"></a>
## 嵌入向量

您可以使用 Laravel 的 `Stringable` 類別所提供的全新 `toEmbeddings` 方法，輕鬆地為任何指定的字串產生向量嵌入：

```php
use Illuminate\Support\Str;

$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings();
```

或者，您可以使用 `Embeddings` 類別來同時為多個輸入值產生嵌入向量：

```php
use Laravel\Ai\Embeddings;

$response = Embeddings::for([
    'Napa Valley has great wine.',
    'Laravel is a PHP framework.',
])->generate();

$response->embeddings; // [[0.123, 0.456, ...], [0.789, 0.012, ...]]
```

您可以指定嵌入向量的維度與提供者：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->dimensions(1536)
    ->generate(Lab::OpenAI, 'text-embedding-3-small');
```

<a name="querying-embeddings"></a>
### 查詢嵌入向量

產生嵌入向量後，您通常會將它們儲存於資料庫的 `vector` 欄位中，以便日後進行查詢。Laravel 透過 `pgvector` 擴充功能，為 PostgreSQL 的向量欄位提供了原生支援。首先，請在您的遷移中定義一個 `vector` 欄位，並指定維度數量：

```php
Schema::ensureVectorExtensionExists();

Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->vector('embedding', dimensions: 1536);
    $table->timestamps();
});
```

您也可以新增向量索引以加速相似度搜尋。當您在向量欄位上呼叫 `index` 時，Laravel 將會自動建立一個使用餘弦距離的 HNSW 索引：

```php
$table->vector('embedding', dimensions: 1536)->index();
```

在您的 Eloquent 模型上，您應該將此向量欄位轉換為 `array`：

```php
protected function casts(): array
{
    return [
        'embedding' => 'array',
    ];
}
```

若要查詢相似的紀錄，請使用 `whereVectorSimilarTo` 方法。此方法會透過最小餘弦相似度（介於 `0.0` 與 `1.0` 之間，其中 `1.0` 為完全相同）來篩選結果，並依相似度進行排序：

```php
use App\Models\Document;

$documents = Document::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

其中的 `$queryEmbedding` 可以是浮點數陣列或純字串。若傳入字串，Laravel 將會自動為其產生嵌入向量：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', 'best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

如果您需要更細緻的控制，也可以單獨使用較低階的 `whereVectorDistanceLessThan`、`selectVectorDistance` 以及 `orderByVectorDistance` 方法：

```php
$documents = Document::query()
    ->select('*')
    ->selectVectorDistance('embedding', $queryEmbedding, as: 'distance')
    ->whereVectorDistanceLessThan('embedding', $queryEmbedding, maxDistance: 0.3)
    ->orderByVectorDistance('embedding', $queryEmbedding)
    ->limit(10)
    ->get();
```

如果您希望讓 AI 代理擁有執行相似度搜尋的能力以作為一項工具，請參閱 [相似度搜尋](#similarity-search) 工具文件。

> [!NOTE]
> 向量查詢目前僅在搭配 `pgvector` 擴充功能的 PostgreSQL 連線中受支援。

<a name="caching-embeddings"></a>
### 快取嵌入向量

嵌入向量的產生可以進行快取，以避免針對相同輸入進行重複的 API 呼叫。若要啟用快取，請將 `ai.caching.embeddings.cache` 設定選項設為 `true`：

```php
'caching' => [
    'embeddings' => [
        'cache' => true,
        'store' => env('CACHE_STORE', 'database'),
        // ...
    ],
],
```

啟用快取時，嵌入向量會被快取 30 天。快取鍵值是根據提供者、模型、維度以及輸入內容所產生，以確保相同的請求會傳回快取的結果，而不同的設定則會產生新的嵌入向量。

即使全域快取已被停用，您也可以使用 `cache` 方法來針對特定請求啟用快取：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache()
    ->generate();
```

您可以指定自訂的快取持續時間（以秒為單位）：

```php
$response = Embeddings::for(['Napa Valley has great wine.'])
    ->cache(seconds: 3600) // Cache for 1 hour
    ->generate();
```

`toEmbeddings` 這個 Stringable 方法也接受 `cache` 引數：

```php
// Cache with default duration...
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: true);

// Cache for a specific duration...
$embeddings = Str::of('Napa Valley has great wine.')->toEmbeddings(cache: 3600);
```

<a name="reranking"></a>
## 重新排序 (Reranking)

重新排序（Reranking）允許您根據文件與指定查詢的相關性，重新排列文件列表。這對於利用語意理解來改善搜尋結果非常有用：

您可以使用 `Laravel\Ai\Reranking` 類別來重新排序文件：

```php
use Laravel\Ai\Reranking;

$response = Reranking::of([
    'Django is a Python web framework.',
    'Laravel is a PHP web application framework.',
    'React is a JavaScript library for building user interfaces.',
])->rerank('PHP frameworks');

// Access the top result...
$response->first()->document; // "Laravel is a PHP web application framework."
$response->first()->score;    // 0.95
$response->first()->index;    // 1 (original position)
```

您可以使用 `limit` 方法來限制回傳的結果數量：

```php
$response = Reranking::of($documents)
    ->limit(5)
    ->rerank('search query');
```

<a name="reranking-collections"></a>
### 重新排序集合

為了方便起見，您可以使用 `rerank` 巨集對 Laravel 集合（Collections）進行重新排序。第一個引數指定用於重新排序的欄位，第二個引數則是查詢內容：

```php
// Rerank by a single field...
$posts = Post::all()
    ->rerank('body', 'Laravel tutorials');

// Rerank by multiple fields (sent as JSON)...
$reranked = $posts->rerank(['title', 'body'], 'Laravel tutorials');

// Rerank using a closure to build the document...
$reranked = $posts->rerank(
    fn ($post) => $post->title.': '.$post->body,
    'Laravel tutorials'
);
```

您也可以限制結果的數量並指定提供者：

```php
$reranked = $posts->rerank(
    by: 'content',
    query: 'Laravel tutorials',
    limit: 10,
    provider: Lab::Cohere
);
```

<a name="files"></a>
## 檔案

可以使用 `Laravel\Ai\Files` 類別或個別的檔案類別將檔案儲存到您的 AI 提供者，以便稍後在對話中使用。這對於大型文件或您想要多次引用而不想重複上傳的檔案非常有用：

```php
use Laravel\Ai\Files\Document;
use Laravel\Ai\Files\Image;

// Store a file from a local path...
$response = Document::fromPath('/home/laravel/document.pdf')->put();
$response = Image::fromPath('/home/laravel/photo.jpg')->put();

// Store a file that is stored on a filesystem disk...
$response = Document::fromStorage('document.pdf', disk: 'local')->put();
$response = Image::fromStorage('photo.jpg', disk: 'local')->put();

// Store a file that is stored on a remote URL...
$response = Document::fromUrl('https://example.com/document.pdf')->put();
$response = Image::fromUrl('https://example.com/photo.jpg')->put();

return $response->id;
```

您也可以儲存原始內容或上傳的檔案：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Files\Document;

// Store raw content...
$stored = Document::fromString('Hello, World!', 'text/plain')->put();

// Store an uploaded file...
$stored = Document::fromUpload($request->file('document'))->put();
```

檔案儲存後，您可以在透過 AI 代理生成文字時引用該檔案，而不需要重新上傳：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Files;

$response = (new SalesCoach)->prompt(
    'Analyze the attached sales transcript...'
    attachments: [
        Files\Document::fromId('file-id') // Attach a stored document...
    ]
);
```

若要取得先前儲存的檔案，請在檔案實例上使用 `get` 方法：

```php
use Laravel\Ai\Files\Document;

$file = Document::fromId('file-id')->get();

$file->id;
$file->mimeType();
```

若要從提供者刪除檔案，請使用 `delete` 方法：

```php
Document::fromId('file-id')->delete();
```

預設情況下，`Files` 類別會使用您應用程式 `config/ai.php` 設定檔中設定的預設 AI 提供者。對於大多數操作，您可以使用 `provider` 引數指定不同的提供者：

```php
$response = Document::fromPath(
    '/home/laravel/document.pdf'
)->put(provider: Lab::Anthropic);
```


<a name="using-stored-files-in-conversations"></a>
### 在對話中使用儲存的檔案

檔案儲存到提供者後，您可以使用 `Document` 或 `Image` 類別上的 `fromId` 方法，在 AI 代理對話中引用它：

```php
use App\Ai\Agents\DocumentAnalyzer;
use Laravel\Ai\Files;
use Laravel\Ai\Files\Document;

$stored = Document::fromPath('/path/to/report.pdf')->put();

$response = (new DocumentAnalyzer)->prompt(
    'Summarize this document.',
    attachments: [
        Document::fromId($stored->id),
    ],
);
```

同樣地，儲存的圖片也可以使用 `Image` 類別來引用：

```php
use Laravel\Ai\Files;
use Laravel\Ai\Files\Image;

$stored = Image::fromPath('/path/to/photo.jpg')->put();

$response = (new ImageAnalyzer)->prompt(
    'What is in this image?',
    attachments: [
        Image::fromId($stored->id),
    ],
);
```


<a name="vector-stores"></a>
## 向量儲存庫

向量儲存庫允許您建立可搜尋的檔案集合，用於檢索增強生成 (RAG)。`Laravel\Ai\Stores` 類別提供了建立、取得和刪除向量儲存庫的方法：

```php
use Laravel\Ai\Stores;

// Create a new vector store...
$store = Stores::create('Knowledge Base');

// Create a store with additional options...
$store = Stores::create(
    name: 'Knowledge Base',
    description: 'Documentation and reference materials.',
    expiresWhenIdleFor: days(30),
);

return $store->id;
```

若要透過 ID 取得現有的向量儲存庫，請使用 `get` 方法：

```php
use Laravel\Ai\Stores;

$store = Stores::get('store_id');

$store->id;
$store->name;
$store->fileCounts;
$store->ready;
```

若要刪除向量儲存庫，請在 `Stores` 類別或儲存庫實例上使用 `delete` 方法：

```php
use Laravel\Ai\Stores;

// Delete by ID...
Stores::delete('store_id');

// Or delete via a store instance...
$store = Stores::get('store_id');

$store->delete();
```


<a name="adding-files-to-stores"></a>
### 將檔案新增至儲存庫

一旦您有了向量儲存庫，您就可以使用 `add` 方法將 [檔案](#files) 新增到其中。新增到儲存庫的檔案會自動建立索引，以便使用 [檔案搜尋提供者工具](#file-search) 進行語意搜尋：

```php
use Laravel\Ai\Files\Document;
use Laravel\Ai\Stores;

$store = Stores::get('store_id');

// Add a file that has already been stored with the provider...
$document = $store->add('file_id');
$document = $store->add(Document::fromId('file_id'));

// Or, store and add a file in one step...
$document = $store->add(Document::fromPath('/path/to/document.pdf'));
$document = $store->add(Document::fromStorage('manual.pdf'));
$document = $store->add($request->file('document'));

$document->id;
$document->fileId;
```

> **注意：** 通常，當您將先前儲存的檔案新增到向量儲存庫時，傳回的文件 ID 會與該檔案先前被指派的 ID 相同；但是，某些向量儲存提供者可能會傳回一個全新的、不同的「文件 ID」。因此，建議您始終在資料庫中儲存這兩個 ID，以備將來參考。

在將檔案新增至儲存庫時，您可以為其附加中繼資料(metadata)。此中繼資料稍後可在使用 [檔案搜尋提供者工具](#file-search) 時用來篩選搜尋結果：

```php
$store->add(Document::fromPath('/path/to/document.pdf'), metadata: [
    'author' => 'Taylor Otwell',
    'department' => 'Engineering',
    'year' => 2026,
]);
```

若要從儲存庫中移除檔案，請使用 `remove` 方法：

```php
$store->remove('file_id');
```

從向量儲存庫中移除檔案並不會將其自提供者的 [檔案儲存空間](#files) 中刪除。若要從向量儲存庫中移除該檔案，並將其從檔案儲存空間中永久刪除，請使用 `deleteFile` 引數：

```php
$store->remove('file_abc123', deleteFile: true);
```


<a name="failover"></a>
## 容錯移轉

在進行提示或生成其他媒體時，您可以提供一組提供者 / 模型的陣列。這樣一來，如果在主要提供者上遇到服務中斷或達到速率限制時，系統會自動容錯移轉至備用的提供者 / 模型：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Image;

$response = (new SalesCoach)->prompt(
    'Analyze this sales transcript...',
    provider: [Lab::OpenAI, Lab::Anthropic],
);

$image = Image::of('A donut sitting on the kitchen counter')
    ->generate(provider: [Lab::Gemini, Lab::xAI]);
```

<a name="testing"></a>
## 測試


<a name="testing-agents"></a>
### AI 代理

若要在測試期間模擬 AI 代理的回應，可以在 AI 代理類別上呼叫 `fake` 方法。您也可以選擇提供一個回應陣列或一個閉包 (Closure)：

```php
use App\Ai\Agents\SalesCoach;
use Laravel\Ai\Prompts\AgentPrompt;

// Automatically generate a fixed response for every prompt...
SalesCoach::fake();

// Provide a list of prompt responses...
SalesCoach::fake([
    'First response',
    'Second response',
]);

// Dynamically handle prompt responses based on the incoming prompt...
SalesCoach::fake(function (AgentPrompt $prompt) {
    return 'Response for: '.$prompt->prompt;
});
```

> **注意：** 當在回傳結構化輸出的 AI 代理上呼叫 `Agent::fake()` 時，Laravel 將會自動產生符合該 AI 代理定義輸出綱要 (Schema) 的模擬資料。

在對 AI 代理進行提示之後，您可以針對接收到的提示詞進行斷言：

```php
use Laravel\Ai\Prompts\AgentPrompt;

SalesCoach::assertPrompted('Analyze this...');

SalesCoach::assertPrompted(function (AgentPrompt $prompt) {
    return $prompt->contains('Analyze');
});

SalesCoach::assertNotPrompted('Missing prompt');

SalesCoach::assertNeverPrompted();
```

對於加入佇列的 AI 代理呼叫，請使用佇列斷言方法：

```php
use Laravel\Ai\QueuedAgentPrompt;

SalesCoach::assertQueued('Analyze this...');

SalesCoach::assertQueued(function (QueuedAgentPrompt $prompt) {
    return $prompt->contains('Analyze');
});

SalesCoach::assertNotQueued('Missing prompt');

SalesCoach::assertNeverQueued();
```

若要確保所有的 AI 代理呼叫都有對應的模擬回應，您可以使用 `preventStrayPrompts`。如果呼叫了 AI 代理卻沒有定義對應的模擬回應，系統將會拋出異常 (Exception)：

```php
SalesCoach::fake()->preventStrayPrompts();
```


<a name="testing-images"></a>
### 圖片

您可以透過呼叫 `Image` 類別上的 `fake` 方法來模擬圖片生成。一旦圖片生成被模擬後，便可以針對記錄的圖片生成提示詞進行各種斷言：

```php
use Laravel\Ai\Image;
use Laravel\Ai\Prompts\ImagePrompt;
use Laravel\Ai\Prompts\QueuedImagePrompt;

// Automatically generate a fixed response for every prompt...
Image::fake();

// Provide a list of prompt responses...
Image::fake([
    base64_encode($firstImage),
    base64_encode($secondImage),
]);

// Dynamically handle prompt responses based on the incoming prompt...
Image::fake(function (ImagePrompt $prompt) {
    return base64_encode('...');
});
```

生成圖片後，您可以針對接收到的提示詞進行斷言：

```php
Image::assertGenerated(function (ImagePrompt $prompt) {
    return $prompt->contains('sunset') && $prompt->isLandscape();
});

Image::assertNotGenerated('Missing prompt');

Image::assertNothingGenerated();
```

對於加入佇列的圖片生成，請使用佇列斷言方法：

```php
Image::assertQueued(
    fn (QueuedImagePrompt $prompt) => $prompt->contains('sunset')
);

Image::assertNotQueued('Missing prompt');

Image::assertNothingQueued();
```

若要確保所有圖片生成都有對應的模擬回應，您可以使用 `preventStrayImages`。如果生成了圖片卻沒有定義對應的模擬回應，系統將會拋出異常 (Exception)：

```php
Image::fake()->preventStrayImages();
```


<a name="testing-audio"></a>
### 語音

您可以透過呼叫 `Audio` 類別上的 `fake` 方法來模擬語音生成。一旦語音生成被模擬後，便可以針對記錄的語音生成提示詞進行各種斷言：

```php
use Laravel\Ai\Audio;
use Laravel\Ai\Prompts\AudioPrompt;
use Laravel\Ai\Prompts\QueuedAudioPrompt;

// Automatically generate a fixed response for every prompt...
Audio::fake();

// Provide a list of prompt responses...
Audio::fake([
    base64_encode($firstAudio),
    base64_encode($secondAudio),
]);

// Dynamically handle prompt responses based on the incoming prompt...
Audio::fake(function (AudioPrompt $prompt) {
    return base64_encode('...');
});
```

生成語音後，您可以針對接收到的提示詞進行斷言：

```php
Audio::assertGenerated(function (AudioPrompt $prompt) {
    return $prompt->contains('Hello') && $prompt->isFemale();
});

Audio::assertNotGenerated('Missing prompt');

Audio::assertNothingGenerated();
```

對於加入佇列的語音生成，請使用佇列斷言方法：

```php
Audio::assertQueued(
    fn (QueuedAudioPrompt $prompt) => $prompt->contains('Hello')
);

Audio::assertNotQueued('Missing prompt');

Audio::assertNothingQueued();
```

若要確保所有語音生成都有對應的模擬回應，您可以使用 `preventStrayAudio`。如果生成了語音卻沒有定義對應的模擬回應，系統將會拋出異常 (Exception)：

```php
Audio::fake()->preventStrayAudio();
```


<a name="testing-transcriptions"></a>
### 語音轉文字

您可以透過呼叫 `Transcription` 類別上的 `fake` 方法來模擬語音轉文字生成。一旦語音轉文字被模擬後，便可以針對記錄的語音轉文字生成提示詞進行各種斷言：

```php
use Laravel\Ai\Transcription;
use Laravel\Ai\Prompts\TranscriptionPrompt;
use Laravel\Ai\Prompts\QueuedTranscriptionPrompt;

// Automatically generate a fixed response for every prompt...
Transcription::fake();

// Provide a list of prompt responses...
Transcription::fake([
    'First transcription text.',
    'Second transcription text.',
]);

// Dynamically handle prompt responses based on the incoming prompt...
Transcription::fake(function (TranscriptionPrompt $prompt) {
    return 'Transcribed text...';
});
```

生成語音轉文字後，您可以針對接收到的提示詞進行斷言：

```php
Transcription::assertGenerated(function (TranscriptionPrompt $prompt) {
    return $prompt->language === 'en' && $prompt->isDiarized();
});

Transcription::assertNotGenerated(
    fn (TranscriptionPrompt $prompt) => $prompt->language === 'fr'
);

Transcription::assertNothingGenerated();
```

對於加入佇列的語音轉文字生成，請使用佇列斷言方法：

```php
Transcription::assertQueued(
    fn (QueuedTranscriptionPrompt $prompt) => $prompt->isDiarized()
);

Transcription::assertNotQueued(
    fn (QueuedTranscriptionPrompt $prompt) => $prompt->language === 'fr'
);

Transcription::assertNothingQueued();
```

若要確保所有語音轉文字生成都有對應的模擬回應，您可以使用 `preventStrayTranscriptions`。如果生成了語音轉文字卻沒有定義對應的模擬回應，系統將會拋出異常 (Exception)：

```php
Transcription::fake()->preventStrayTranscriptions();
```

<a name="testing-embeddings"></a>
### 嵌入向量

您可以透過呼叫 `Embeddings` 類別上的 `fake` 方法來模擬嵌入向量的生成。一旦模擬了嵌入向量，就可以針對記錄到的嵌入向量生成提示詞進行各種斷言：

```php
use Laravel\Ai\Embeddings;
use Laravel\Ai\Prompts\EmbeddingsPrompt;
use Laravel\Ai\Prompts\QueuedEmbeddingsPrompt;

// Automatically generate fake embeddings of the proper dimensions for every prompt...
Embeddings::fake();

// Provide a list of prompt responses...
Embeddings::fake([
    [$firstEmbeddingVector],
    [$secondEmbeddingVector],
]);

// Dynamically handle prompt responses based on the incoming prompt...
Embeddings::fake(function (EmbeddingsPrompt $prompt) {
    return array_map(
        fn () => Embeddings::fakeEmbedding($prompt->dimensions),
        $prompt->inputs
    );
});
```

生成嵌入向量後，您可以對收到的提示詞進行斷言：

```php
Embeddings::assertGenerated(function (EmbeddingsPrompt $prompt) {
    return $prompt->contains('Laravel') && $prompt->dimensions === 1536;
});

Embeddings::assertNotGenerated(
    fn (EmbeddingsPrompt $prompt) => $prompt->contains('Other')
);

Embeddings::assertNothingGenerated();
```

對於已排入佇列的嵌入向量生成，請使用佇列斷言方法：

```php
Embeddings::assertQueued(
    fn (QueuedEmbeddingsPrompt $prompt) => $prompt->contains('Laravel')
);

Embeddings::assertNotQueued(
    fn (QueuedEmbeddingsPrompt $prompt) => $prompt->contains('Other')
);

Embeddings::assertNothingQueued();
```

為了確保所有嵌入向量的生成都有對應的模擬回應，您可以使用 `preventStrayEmbeddings`。如果生成嵌入向量時沒有定義模擬回應，將會拋出例外狀況：

```php
Embeddings::fake()->preventStrayEmbeddings();
```


<a name="testing-reranking"></a>
### 重新排序

重新排序操作可以透過呼叫 `Reranking` 類別上的 `fake` 方法來模擬：

```php
use Laravel\Ai\Reranking;
use Laravel\Ai\Prompts\RerankingPrompt;
use Laravel\Ai\Responses\Data\RankedDocument;

// Automatically generate a fake reranked responses...
Reranking::fake();

// Provide custom responses...
Reranking::fake([
    [
        new RankedDocument(index: 0, document: 'First', score: 0.95),
        new RankedDocument(index: 1, document: 'Second', score: 0.80),
    ],
]);
```

重新排序後，您可以對執行的操作進行斷言：

```php
Reranking::assertReranked(function (RerankingPrompt $prompt) {
    return $prompt->contains('Laravel') && $prompt->limit === 5;
});

Reranking::assertNotReranked(
    fn (RerankingPrompt $prompt) => $prompt->contains('Django')
);

Reranking::assertNothingReranked();
```


<a name="testing-files"></a>
### 檔案

檔案操作可以透過呼叫 `Files` 類別上的 `fake` 方法來模擬：

```php
use Laravel\Ai\Files;

Files::fake();
```

一旦模擬了檔案操作，您就可以對發生的上傳和刪除進行斷言：

```php
use Laravel\Ai\Contracts\Files\StorableFile;
use Laravel\Ai\Files\Document;

// Store files...
Document::fromString('Hello, Laravel!', mimeType: 'text/plain')
    ->as('hello.txt')
    ->put();

// Make assertions...
Files::assertStored(fn (StorableFile $file) =>
    (string) $file === 'Hello, Laravel!' &&
        $file->mimeType() === 'text/plain';
);

Files::assertNotStored(fn (StorableFile $file) =>
    (string) $file === 'Hello, World!'
);

Files::assertNothingStored();
```

若要對刪除檔案進行斷言，您可以傳遞檔案 ID：

```php
Files::assertDeleted('file-id');
Files::assertNotDeleted('file-id');
Files::assertNothingDeleted();
```


<a name="testing-vector-stores"></a>
### 向量儲存庫

向量儲存庫的操作可以透過呼叫 `Stores` 類別上的 `fake` 方法來模擬。模擬儲存庫也會自動模擬[檔案操作](#files)：

```php
use Laravel\Ai\Stores;

Stores::fake();
```

一旦模擬了儲存庫操作，您就可以對已建立或刪除的儲存庫進行斷言：

```php
use Laravel\Ai\Stores;

// Create store...
$store = Stores::create('Knowledge Base');

// Make assertions...
Stores::assertCreated('Knowledge Base');

Stores::assertCreated(fn (string $name, ?string $description) =>
    $name === 'Knowledge Base'
);

Stores::assertNotCreated('Other Store');

Stores::assertNothingCreated();
```

若要對刪除儲存庫進行斷言，您可以提供儲存庫 ID：

```php
Stores::assertDeleted('store_id');
Stores::assertNotDeleted('other_store_id');
Stores::assertNothingDeleted();
```

若要斷言檔案已被新增至儲存庫或自儲存庫中移除，請使用給定 `Store` 執行個體上的斷言方法：

```php
Stores::fake();

$store = Stores::get('store_id');

// Add / remove files...
$store->add('added_id');
$store->remove('removed_id');

// Make assertions...
$store->assertAdded('added_id');
$store->assertRemoved('removed_id');

$store->assertNotAdded('other_file_id');
$store->assertNotRemoved('other_file_id');
```

如果檔案在同一個請求中被儲存到提供者的[檔案儲存空間](#files)並新增到向量儲存庫，您可能不知道該檔案在提供者端的 ID。在這種情況下，您可以傳遞閉包到 `assertAdded` 方法，以針對新增檔案的內容進行斷言：

```php
use Laravel\Ai\Contracts\Files\StorableFile;
use Laravel\Ai\Files\Document;

$store->add(Document::fromString('Hello, World!', 'text/plain')->as('hello.txt'));

$store->assertAdded(fn (StorableFile $file) => $file->name() === 'hello.txt');
$store->assertAdded(fn (StorableFile $file) => $file->content() === 'Hello, World!');
```

<a name="events"></a>
## 事件

Laravel AI SDK 會發送多種[事件](/docs/{{version}}/events)，包括：

- `AddingFileToStore`
- `AgentPrompted`
- `AgentStreamed`
- `AudioGenerated`
- `CreatingStore`
- `EmbeddingsGenerated`
- `FileAddedToStore`
- `FileDeleted`
- `FileRemovedFromStore`
- `FileStored`
- `GeneratingAudio`
- `GeneratingEmbeddings`
- `GeneratingImage`
- `GeneratingTranscription`
- `ImageGenerated`
- `InvokingTool`
- `PromptingAgent`
- `RemovingFileFromStore`
- `Reranked`
- `Reranking`
- `StoreCreated`
- `StoringFile`
- `StreamingAgent`
- `ToolInvoked`
- `TranscriptionGenerated`

您可以監聽任何這些事件，以記錄或儲存 AI SDK 的使用資訊。