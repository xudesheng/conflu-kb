# LLM provider, `AIAgent`, and FileRepositories

## LLM provider setup

### Select the right Thing Template

The first step is to define an LLM provider. We already have 5 `ThingTemplate` entities defined in the extension.

<img src="./__images__//image-20260530171838949.png" alt="image-20260530171838949" style="zoom:50%;" />

They are:

* `AzureOpenAIChatV4Provider`: supporting Azure OpenAI 4.x model with or without `mini`. The typical models are: `gpt-4.1`, `gpt-4.1-mini`. It uses the OpenAI Chat Completions API.
* `AzureOpenAIChatV5Provider`: supporting Azure OpenAI 5.x & O-series models with or without `mini`. The typical models are: `gpt-5.4`,`gpt-5.4-mini`,`gpt-5.5`,`gpt-5.5-mini`,`o3-pro` etc. It uses the OpenAI Chat Completions API (reasoning models, `max_completion_tokens`).
* `OpenAIChatV5Provider`: similar to `AzureOpenAIChatV5Provider` but it is provided by `OpenAI` directly.
* `OpenAIChatV4Provider`: similar to `AzureOpenAIChatV4Provider` but it is provided by `OpenAI` directly.
* `AnthropicMessagesProvider`: This template supports all models from `Anthropic`.

Besides the 5 provider templates above, we plan to add:

* A `Google Gemini` provider template.
* A `Mistral` provider template.



### Create your LLM provider Thing based on a selected Thing Template

Let's create the first LLM provider Thing. In this example, we will use a deployment of `gpt-5.4`:

<img src="./__images__//image-20260531141248150.png" alt="image-20260531141248150" style="zoom:50%;" />

Please make sure to choose the template: `AzureOpenAIChatV5Provider`.

### LLM provider configuration

On the configuration page, please pay attention to the following fields:

* endpoint
* apiKey
* deployment
* `maxCompletionTokens`: This number includes the reasoning tokens and final output tokens. We recommend a number between 2048 and 8192.
* `rateControlMode`: it must be `enforce`. Otherwise, you may encounter 429 errors too often.
* `tokensPerMinuteLimit`: when you obtain your API key, usually this limit has been set on the Azure side. Please check what it is. Please be aware that `Parler` has no way to manage the limit if the API key is used by multiple users across different ThingWorx instances.
* `maxConcurrentRequests`: We only allow **1** to be used right now.
* `maxLocalWaitMs`: please use `120000` during your test.

<img src="./__images__//image-20260530214748710.png" alt="image-20260530214748710" style="zoom:50%;" />



### Specific configuration for Anthropic Sonnet 4.6 and Opus 4.8 on Azure Foundry

When you use the Anthropic models on Azure Foundry, please change the base url into the following pattern.

<img src="./__images__//image-20260611093606894.png" alt="image-20260611093606894" style="zoom:50%;" />

Anthropic has deprecated the `temperature` parameter completely in `Opus 4.8` model. Please use value `omit` for `samplingParametersMode` configuration item. This applies to both Azure Foundry deployment and Anthropic deployment.

<img src="./__images__//image-20260607221919660.png" alt="image-20260607221919660" style="zoom:50%;" />

### Test your connection to LLM

Please go to the `Services` tab and select `TestConnection` to run.

<img src="./__images__//image-20260530180804660.png" alt="image-20260530180804660" style="zoom:50%;" />

If everything has been configured properly, the result should be true.

<img src="./__images__//image-20260530181606376.png" alt="image-20260530181606376" style="zoom:50%;" />



If the result is false, please check your Application log and look for LLM_HTTP_FAILURE. If the message looks like below, that means the IP from your ThingWorx server to the LLM provider is **blocked**.

<img src="./__images__//image-20260530181430360.png" alt="image-20260530181430360" style="zoom:50%;" />



You have to find out the outgoing IP on your ThingWorx instance and ask your admin to put your IP address on the Azure side.

**`Caution`**: The `Public IP Address` on the PTC cloud instance desktop may be wrong, you have to use the following command to ensure you have the right outgoing public IP:

```
Invoke-RestMethod -Uri "https://api64.ipify.org"
```


<img src="./__images__//image-20260602001413754.png" alt="image-20260602001413754" style="zoom:50%;" />

<img src="./__images__//image-20260602001554124.png" alt="image-20260602001554124" style="zoom:50%;" />



## Two FileRepositories (recommended for class labs)

You should have created two repositories `AIDocRepository` & `ConfigurationRepository` in the last chapter. 

Parler separates **configuration files** from **exported artifacts**:

| Setting on `AIAgent`          | Purpose                                                      |
| ----------------------------- | ------------------------------------------------------------ |
| **`configurationRepository`** | FileRepository holding **`/taxonomies`**, **`/skills`**, **`/playbooks`**, **`/tools/extended_tools.json`**, **`/policies`**, etc. Maintainer details live in **`docs/agent/configuration-repository.md`** in the **parler** monorepo; the workshop chapters inline the paths you need. |
| **`exportFileRepository`**    | FileRepository for **table exports** and similar outputs (`AgentSettings.exportFileRepository` — **`THINGNAME`** to a **FileRepository**). |

**Exercise:** create **two** FileRepository Things—one **config**, one **export**—bind both on your **`AIAgent`** to be created in the next step, and confirm Composer aspects resolve. **(screenshot)** repository bindings.



## Create a dedicated `AIAgent` Thing

### Create `AIAgent` Thing

After you have created your LLM provider, you can now create your `AIAgent` Thing.

<img src="./__images__//image-20260531141935107.png" alt="image-20260531141935107" style="zoom:50%;" />

### Configure and set up the LLM provider

You have to go to the configuration tab and set the `LLM API Provider` to the Thing you created in the last step.

<img src="./__images__//image-20260530185237735.png" alt="image-20260530185237735" style="zoom:50%;" />





### Test your Agent Thing

You can use the `TestConnection` on the `AIAgent` Thing to test the connection too.

<img src="./__images__//image-20260530185348115.png" alt="image-20260530185348115" style="zoom:50%;" />

The result should be true if your LLM provider can connect without issues. If the result is false, please check the error the same way as in the last step.

<img src="./__images__//image-20260530185414301.png" alt="image-20260530185414301" style="zoom:50%;" />

## Two refresh services

There are two `refresh` services on your `AIAgent` Thing. We will use them very often in the next steps.

<img src="./__images__//image-20260530185520610.png" alt="image-20260530185520610" style="zoom:50%;" />
