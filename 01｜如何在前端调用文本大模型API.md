# 01｜如何在前端调用文本大模型API
你好，我是月影。

这一节课，我们先来了解如何通过调用API的方式来使用最基础的文本大模型。

由于几乎所有开放给用户使用的大模型服务都提供HTTP协议的API，所以对于前端工程师来说，使用大模型最简单的入门方式其实就是直接调用由服务平台提供的API。

由于各种不同类型的大模型和不同服务平台的API略有差异，我们也无法穷尽，所以这节课我们主要了解Deepseek和Coze这两个通用文本大模型服务的前端使用方法，以及不同的调用方式。

因为是课程的第一部分，所以内容不会太过于深入细节，但是在这一讲中，我们将通过实战操作，来熟悉大模型的API调用方法、数据协议和格式，为后续深入学习打下基础。

## 使用DeepSeek Platform

要说最近国内哪个大模型得到最多人关注，那毫无疑问应该是DeepSeek，它不仅以极低的成本实现了和行业巨头相媲美的推理能力和性能，而且最近发布的深度推理 R1 模型在性能和效果上都表现极其优越，它的团队还将模型训练中的技术创新全部公开，促进了技术社区之间的深入交流与协同创新。

实际上在Deepseek v3和R1推出的近一年之前，我的业务中就在部分使用它，因为它极低廉的价格和不错的推理能力，也因为它提供了非常好用的官方API平台，所以它对我们这些AI应用创业者非常友好。

好，让我们言归正传，来看看作为前端工程师，应当如何使用Deepseek官方API。

DeepSeek官方API平台是 [https://platform.deepseek.com/](https://platform.deepseek.com/) ，我们可以用手机号注册或者微信登录。登录后，就能进入API后台：

![图片](images/868006/92a04cd74ac5fdc1e97d74584cc304b5.jpg)

在这里，我们需要注意两个信息，一个是主页面上的余额信息，它决定了还有多少token剩余量。根据文档，Deepseek目前的价格如下：

![图片](images/868006/024bf73db0f81cba81b9a809f3d983c0.png)

如果你是第一次接触大模型API调用，需要了解一下token的概念。token是模型用来表示自然语言文本的基本单位，通常一个中文词语或一个英文单词、数字或符号计为一个token。在每次API调用成功后，我们可以通过返回结果的usage得到token的消耗量。

除了余额，第二个需要关注的信息是API Keys，它是我们的应用调用API的许可凭证。我们可以点击左侧菜单来创建、查看和管理它。

![图片](images/868006/37f21446fb88aa303cc2daafbc03c876.png)

接下来我们就从最简单的做起，从浏览器端调用Deepseek API。因为它支持HTTPS协议的接口，API调用URL是 `[https://api.deepseek.com/chat/completions](https://api.deepseek.com/chat/completions)`。

让我们用 Trae 创建一个项目，写一个简单的页面。

![图片](images/868006/fef70136bd945f38e036e2f21f104eb4.png)

Trae 的 AI Builder 自动创建的项目结构如下：

![图片](images/868006/67e903370b489f157ef1284884ff619b.png)

项目创建完毕后，别忘了分别修改项目目录下的 index.html 文件以及 script.js 文件：

> index.html

```javascript
<body>
    <h1>Hello Deepseek!</h1>
    <div id="reply">thinking...</div>
    <script type="module" src="script.js"></script>
</body>

```

> script.js

```javascript
const endpoint = 'https://api.deepseek.com/chat/completions';
const headers = {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${import.meta.env.VITE_DEEPSEEK_API_KEY}`
};

const payload = {
    model: 'deepseek-chat',
    messages: [
        {role: "system", content: "You are a helpful assistant."},
        {role: "user", content: "你好 Deepseek"}
    ],
    stream: false,
};

const response = await fetch(endpoint, {
    method: 'POST',
    headers: headers,
    body: JSON.stringify(payload)
});

const data = await response.json();
document.getElementById('reply').textContent = data.choices[0].message.content;

```

同时在项目目录下创建 .env.local 文件，内容如下，其中sk-xxxxxxxx为你在Deepseek Platform中创建的API Key。

```javascript
VITE_DEEPSEEK_API_KEY=sk-xxxxxxxxx

```

运行代码，等待几秒，你大概会得到如下输出：

![图片](images/868006/da0dc5e2b0ccb3411e812b7da06ca607.png)

在这里，主要看一下 `script.js` 这个文件。

首先我们声明endpoint变量为 `https://api.deepseek.com/chat/completions` 。前面说过，这是Deepseek的API调用URL。接着，我们通过声明headers变量来设置HTTP Headers。

这里需要注意的是，根据API鉴权规范， **我们需要将Deepseek Platform申请的API Key放在Authorization请求头字段中传给服务器以完成权限验证**。由于请求是通过HTTPS加密传输的，所以不用担心在这个过程中我们的API Key被第三方窃取。

接着，我们设置要发送给API服务的body内容，它是一份JSON格式的文本数据：

```json
{
    "model": "deepseek-chat",
    "messages": [
        {role: "system", content: "You are a helpful assistant."},
        {role: "user", content: "你好 Deepseek"}
    ],
    "stream": false,
}

```

数据中，model字段指定了要调用的模型类型，Deepseek Platform支持两个模型。其中deepseek-chat是基础模型，目前版本是v3，deepseek-reasoner是深度思考模型，目前版本号是r1。与基础模型相比，深度思考模型的推理能力更强，相应的响应速度要慢一些，价格也要贵不少。这里我们先使用基础模型。

messages字段是要发送给大模型的具体消息，一条消息本身也是JSON格式的，role字段是一个枚举字段，可选的值分别是system、user和assistant，依次表示该条消息是系统消息（也就是我们一般俗称的提示词）、用户消息和AI应答消息。其中user和assistant消息是必须成对的，以表示聊天上下文，且最后一条消息必须是user消息，而system消息的条数和位置一般没有限制。消息体的content则是具体的消息文本内容。

stream字段为false，表示它是以标准的HTTP方式传输而非流式（streaming）传输（关于流式传输的内容，我们下节课再详细展开）。

最终，我们将内容发送给大模型服务，并获得返回值，这一过程是一个异步请求过程。

```javascript
const response = await fetch(endpoint, {
    method: 'POST',
    headers: headers,
    body: JSON.stringify(payload)
});

const data = await response.json();

```

当数据从大模型API服务返回后，如果你打开控制台，你可以看到完整的API响应数据，大致如下：

```json
{
    "id": "91e192ff-5adf-4955-92bb-5ac8fe9d3c22",
    "object": "chat.completion",
    "created": 1739870580,
    "model": "deepseek-chat",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "你好！有什么我可以帮助你的吗？"
            },
            "logprobs": null,
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 12,
        "completion_tokens": 8,
        "total_tokens": 20,
        "prompt_tokens_details": {
            "cached_tokens": 0
        },
        "prompt_cache_hit_tokens": 0,
        "prompt_cache_miss_tokens": 12
    },
    "system_fingerprint": "fp_3a5770e1b4"
}

```

其中我们关注的数据主要是choices字段返回的内容，choices.message.content就是AI返回的文本响应内容，我们将它在网页上显示出来。

```javascript
document.getElementById('reply').textContent
  = data.choices[0].message.content;

```

此外，如果需要的话，我们还可以通过usage字段得到消耗的token数量，从而计算出我们这一次调用的费用消耗，在这里就不展开细说了。

这样我们就完成了一次最简单的Deepseek大模型API调用。是不是很简单呢？

## 使用Coze

接下来，我们介绍另一个平台——Coze。

Coze（扣子）是由字节跳动推出的一款AI机器人和智能体创建平台，旨在帮助用户快速构建、调试和优化AI聊天机器人应用程序。

严格来说，Coze不同于Deepseek Platform，因为它不仅提供API，更重要的是集成了创建智能体和AI应用机器人的能力。

API是底层，智能体和AI机器人可以理解为上层应用，在后续的课程里，我们会详细了解它们的区别。但是在这一节课中，我们先不用管这些区别，重点来看一下如何使用Coze的API来生成文本。

首先我们在 [https://www.coze.cn/](https://www.coze.cn/) 完成注册，进入Coze控制台，选择“工作空间>个人空间>项目开发”，然后点击创建，选择创建智能体。

![图片](images/868006/ab5931564386c2cdb89ca628906e71d7.png)

智能体名称中，我们输入“通用智能体 for API”，点击确认按钮完成创建。

![图片](images/868006/83319fc0663a008037432c9744edb21d.png)

Coze智能体的系统提示词支持模板变量，我们在人设与回复逻辑中只输入一个变量{{prompt}}，模型选择豆包.1.5 Pro 32k。

![图片](images/868006/b865214958b9c59d1b366f44ebcae0d6.png)

接着我们点发布按钮，选择发布平台只勾选发布为API。

![图片](images/868006/8bc55934a551c4f9d400b8d7d232b0c7.png)

继续点发布按钮完成发布。

发布完成后，你可以从浏览器地址栏中获取到bot\_id，此时浏览器地址栏类似于 [https://www.coze.cn/space/7472697029454872613/bot/7473317995184865307](https://www.coze.cn/space/7472697029454872613/bot/7473317995184865307)，这里的7473317995184865307这串数字就是bot\_id，接下来我们在API调用中会用到它。

接着我们选择左侧菜单中的“扣子 API”，展开后选择授权，切换到“个人访问令牌”，点击添加新令牌，创建个人访问令牌，之后我们就可以通过个人访问令牌来进行API调用的鉴权了。

![图片](images/868006/85ec255b99ed8ef0b07ce6ce1f79b174.png)

接下来我们还是在 Trae 中创建一个新的Web项目Coze API。

项目创建后，添加.env.local文件如下：

```typescript
VITE_API_KEY=pat_kTWhBZtBNYhE2xdGshu2Ukeq7z71V**********
VITE_BOT_ID=7473317995184865307

```

然后修改index.html和script.js。

> index.html

```xml
<!DOCTYPE html>
<html>
  <head>
    <title>Coze API</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width">
    <link rel="stylesheet" type="text/css" href="style.css">
  </head>
  <body>
    <h1>Hello Coze!</h1>
    <div id="reply">thinking...</div>
    <script type="module" src="script.js"></script>
  </body>
</html>

```

> script.js

```javascript
const endpoint = 'https://api.coze.cn/open_api/v2/chat';

const payload = {
    bot_id: import.meta.env.VITE_BOT_ID,
    user: 'yvo',
    query: '你好',
    chat_history: [],
    stram: false,
    custom_variables: {
        prompt: "你是一个AI助手"
    }
};

const response = await fetch(endpoint, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${import.meta.env.VITE_API_KEY}`,
    },
    body: JSON.stringify(payload),
});

const data = await response.json();
document.getElementById('reply').textContent = data.messages[0].content;

```

从上面的代码，我们看到，Coze API和Deepseek Platform的API调用逻辑基本类似，只是参数和返回结果有所不同。

Coze的API调用需要传bot\_id和user参数，其中bot\_id就是我们创建的通用智能体的ID，前面我们说过它可以从浏览器地址栏的URL中获得，user可以是一个任意字符串，只是用来标识。

此外，Coze API的query和chat\_history是分开的，当前输入内容以query字段传入，格式是一个字符串，而历史消息以chat\_history字段传入，格式和前面Deepseek的messages差不多，具体可以看Coze的官方文档。如果不需要历史消息，只需要传入一个空的数组。

由于Coze没有system消息，它的提示词是在人设与回复逻辑中设置，我们在前面创建时，已经定义了一个叫prompt的模板变量，所以在这里我们可以通过custom\_variables参数将prompt变量的具体值传入。

这样，我们就完成了Coze智能体的API调用，点击运行代码，等待几秒钟，就可以在网页上看到推理结果。

![图片](images/868006/a9fe25f019930b31faa41a0906bf6679.png)

## 要点总结

今天，我们以Deepseek Platform和Coze为例，详细介绍了如何在浏览器端调用文本大模型API。

实际上，除了Deepseek和Coze外，包括月之暗面、智谱清言在内的大部分文本大模型API都是兼容OpenAI的API参数格式的，因此基本上都可以用同样的方式对它们进行调用，只需要变更API key和endpoint即可。有兴趣的同学，可以在课后尝试修改例子，调用其他平台的大模型。这样也可以体验一下不同模型的输出效率和推理能力的差别。

## 课后练习

我们前面了解了Deepseek API的基本用法，它还有一些配置参数，会影响大模型内容的生成，比较常用的参数比如temperature，它的设定会影响内容输出的随机性，你可以在Deepseek Platform中详细浏览一下文档，并通过修改上面的例子动手实践，学习这些参数的作用和效果，这对你将来在项目中具体应用会很有帮助。

常用的文本大模型，除了Deepseek和Coze外，还有很多其他的选择，它们各有特点和长处，比如月之暗面的大模型Moonshot，在多模态（支持视觉模型）和处理长文本方面表现比较不错。月之暗面也提供了Moonshot AI的开放平台 [https://platform.moonshot.cn/](https://platform.moonshot.cn/)，你可以在这个平台上完成注册，并根据文档练习如何调用Moonshot API，体验它和Deepseek有哪些不同，将你的体验结果分享到评论中。