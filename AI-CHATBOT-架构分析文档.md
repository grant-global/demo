# AI Chatbot 项目架构分析与原理文档

## 项目概述

这是一个基于 Next.js 14 和 AI SDK 构建的现代化 AI 聊天机器人应用，具备完整的用户认证系统、实时聊天功能、AI 工具调用、文档生成与编辑等核心功能。项目采用全栈架构，使用 TypeScript 开发，遵循严格的代码质量标准。

### 技术栈

- **前端框架**: Next.js 15.3.0-canary.31 with App Router
- **开发语言**: TypeScript 5.6.3
- **UI 框架**: React 19 RC + Radix UI + Tailwind CSS 4.1.13
- **AI 集成**: AI SDK 5.0.26 + Vercel AI Gateway
- **数据库**: PostgreSQL + Drizzle ORM 0.34.0
- **认证系统**: NextAuth.js 5.0.0-beta.25
- **代码质量**: Biome 2.2.2 + Ultracite 5.3.9
- **状态管理**: SWR 2.2.5 + React Hooks
- **部署平台**: Vercel

## 整体架构

### 1. 项目结构分析

```
ai-chatbot/
├── app/                          # Next.js App Router 应用目录
│   ├── (auth)/                   # 认证相关路由组
│   │   ├── actions.ts           # 认证相关 Server Actions
│   │   ├── api/auth/            # 认证 API 路由
│   │   ├── auth.config.ts       # NextAuth 配置
│   │   ├── auth.ts             # NextAuth 核心配置
│   │   ├── login/              # 登录页面
│   │   └── register/           # 注册页面
│   ├── (chat)/                  # 聊天相关路由组
│   │   ├── actions.ts          # 聊天相关 Server Actions
│   │   ├── api/                # 聊天 API 路由
│   │   │   ├── chat/           # 聊天消息 API
│   │   │   ├── document/       # 文档管理 API
│   │   │   ├── files/          # 文件上传 API
│   │   │   ├── history/        # 聊天历史 API
│   │   │   ├── suggestions/    # 建议 API
│   │   │   └── vote/           # 消息投票 API
│   │   ├── chat/[id]/          # 动态聊天页面
│   │   ├── layout.tsx          # 聊天布局组件
│   │   └── page.tsx           # 聊天首页
│   ├── globals.css            # 全局样式
│   └── layout.tsx            # 根布局组件
├── artifacts/                 # Artifacts 功能模块
│   ├── actions.ts            # Artifacts 相关 Server Actions
│   ├── code/                # 代码类型 Artifact
│   ├── image/               # 图片类型 Artifact
│   ├── sheet/               # 表格类型 Artifact
│   └── text/                # 文本类型 Artifact
├── components/               # 可复用组件库
│   ├── ui/                  # 基础 UI 组件 (shadcn/ui)
│   ├── elements/            # 复合元素组件
│   ├── chat.tsx            # 主聊天组件
│   ├── messages.tsx        # 消息列表组件
│   ├── artifact.tsx        # Artifacts 容器组件
│   └── ...                 # 其他业务组件
├── hooks/                   # 自定义 React Hooks
├── lib/                     # 核心业务逻辑库
│   ├── ai/                 # AI 集成模块
│   │   ├── models.ts       # AI 模型配置
│   │   ├── providers.ts    # AI 提供商配置
│   │   ├── prompts.ts      # 系统提示词
│   │   └── tools/          # AI 工具函数
│   ├── artifacts/          # Artifacts 服务端逻辑
│   ├── db/                 # 数据库相关
│   │   ├── schema.ts       # 数据库模式定义
│   │   ├── queries.ts      # 数据库查询函数
│   │   └── migrations/     # 数据库迁移文件
│   ├── editor/             # 编辑器相关
│   └── utils.ts           # 工具函数
└── tests/                  # 测试文件
```

### 2. 核心模块架构

## 认证系统 (Authentication System)

### 认证架构设计

项目使用 NextAuth.js v5 实现认证系统，支持多种登录方式：

#### 1. 认证提供商配置

```typescript
// app/(auth)/auth.ts
export const {
  handlers: { GET, POST },
  auth,
  signIn,
  signOut,
} = NextAuth({
  providers: [
    // 常规用户凭证登录
    Credentials({
      credentials: {},
      async authorize({ email, password }: any) {
        const users = await getUser(email);
        // 密码验证逻辑
        const passwordsMatch = await compare(password, user.password);
        return passwordsMatch ? { ...user, type: "regular" } : null;
      },
    }),
    // 访客登录
    Credentials({
      id: "guest",
      credentials: {},
      async authorize() {
        const [guestUser] = await createGuestUser();
        return { ...guestUser, type: "guest" };
      },
    }),
  ],
  // JWT 和 Session 回调配置
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.id = user.id as string;
        token.type = user.type;
      }
      return token;
    },
    session({ session, token }) {
      if (session.user) {
        session.user.id = token.id;
        session.user.type = token.type;
      }
      return session;
    },
  },
});
```

#### 2. 用户类型系统

```typescript
export type UserType = "guest" | "regular";

// 用户权限配置
const entitlementsByUserType = {
  guest: { maxMessagesPerDay: 10 },
  regular: { maxMessagesPerDay: 100 }
};
```

#### 3. 中间件保护

```typescript
// middleware.ts
export async function middleware(request: NextRequest) {
  const token = await getToken({ req: request });
  
  if (!token) {
    // 未认证用户重定向到访客登录
    return NextResponse.redirect(
      new URL(`/api/auth/guest?redirectUrl=${encodeURIComponent(request.url)}`, request.url)
    );
  }
  
  const isGuest = guestRegex.test(token?.email ?? "");
  
  // 已登录用户访问登录页面时重定向到首页
  if (token && !isGuest && ["/login", "/register"].includes(pathname)) {
    return NextResponse.redirect(new URL("/", request.url));
  }
  
  return NextResponse.next();
}
```

## AI 集成系统 (AI Integration)

### 1. AI 模型配置架构

#### 模型定义与提供商

```typescript
// lib/ai/models.ts
export type ChatModel = {
  id: string;
  name: string;
  description: string;
};

export const chatModels: ChatModel[] = [
  {
    id: "chat-model",
    name: "Grok Vision",
    description: "Advanced multimodal model with vision and text capabilities",
  },
  {
    id: "chat-model-reasoning",
    name: "Grok Reasoning", 
    description: "Uses advanced chain-of-thought reasoning for complex problems",
  },
];
```

#### AI 提供商配置

```typescript
// lib/ai/providers.ts
export const myProvider = customProvider({
  languageModels: {
    "chat-model": gateway.languageModel("xai/grok-2-vision-1212"),
    "chat-model-reasoning": wrapLanguageModel({
      model: gateway.languageModel("xai/grok-3-mini"),
      middleware: extractReasoningMiddleware({ tagName: "think" }),
    }),
    "title-model": gateway.languageModel("xai/grok-2-1212"),
    "artifact-model": gateway.languageModel("xai/grok-2-1212"),
  },
});
```

### 2. AI 工具系统 (Tools)

项目实现了强大的 AI 工具调用系统，支持多种工具：

#### 文档创建工具

```typescript
// lib/ai/tools/create-document.ts
export const createDocument = ({ session, dataStream }: CreateDocumentProps) =>
  tool({
    description: "Create a document for writing or content creation activities",
    inputSchema: z.object({
      title: z.string(),
      kind: z.enum(artifactKinds),
    }),
    execute: async ({ title, kind }) => {
      const id = generateUUID();
      
      // 流式输出数据到客户端
      dataStream.write({ type: "data-kind", data: kind, transient: true });
      dataStream.write({ type: "data-id", data: id, transient: true });
      dataStream.write({ type: "data-title", data: title, transient: true });
      
      // 调用对应类型的文档处理器
      const documentHandler = documentHandlersByArtifactKind.find(
        (handler) => handler.kind === kind
      );
      
      await documentHandler.onCreateDocument({
        id, title, dataStream, session,
      });
      
      return { id, title, kind, content: "Document created successfully" };
    },
  });
```

#### 天气查询工具

```typescript
// lib/ai/tools/get-weather.ts
export const getWeather = tool({
  description: "Get weather information for a location",
  inputSchema: z.object({
    location: z.string(),
  }),
  execute: async ({ location }) => {
    // 调用天气 API 获取数据
    // 返回结构化天气信息
  },
});
```

### 3. 系统提示词 (System Prompts)

```typescript
// lib/ai/prompts.ts
export const systemPrompt = ({
  selectedChatModel,
  requestHints,
}: {
  selectedChatModel: string;
  requestHints: RequestHints;
}) => {
  const requestPrompt = getRequestPromptFromHints(requestHints);

  if (selectedChatModel === "chat-model-reasoning") {
    return `${regularPrompt}\n\n${requestPrompt}`;
  }

  return `${regularPrompt}\n\n${requestPrompt}\n\n${artifactsPrompt}`;
};

// Artifacts 相关提示词
export const artifactsPrompt = `
Artifacts is a special user interface mode that helps users with writing, editing, and other content creation tasks. When artifact is open, it is on the right side of the screen, while the conversation is on the left side.

**When to use \`createDocument\`:**
- For substantial content (>10 lines) or code
- For content users will likely save/reuse (emails, code, essays, etc.)
- When explicitly requested to create a document

**When NOT to use \`createDocument\`:**
- For informational/explanatory content
- For conversational responses
- When asked to keep it in chat

**Using \`updateDocument\`:**
- Default to full document rewrites for major changes
- Use targeted updates only for specific, isolated changes
- Follow user instructions for which parts to modify
`;
```

## 数据库架构 (Database Schema)

### 1. 数据模型设计

项目使用 PostgreSQL 数据库，通过 Drizzle ORM 进行数据访问：

#### 核心数据表

```typescript
// lib/db/schema.ts

// 用户表
export const user = pgTable("User", {
  id: uuid("id").primaryKey().notNull().defaultRandom(),
  email: varchar("email", { length: 64 }).notNull(),
  password: varchar("password", { length: 64 }),
});

// 聊天表
export const chat = pgTable("Chat", {
  id: uuid("id").primaryKey().notNull().defaultRandom(),
  createdAt: timestamp("createdAt").notNull(),
  title: text("title").notNull(),
  userId: uuid("userId").notNull().references(() => user.id),
  visibility: varchar("visibility", { enum: ["public", "private"] })
    .notNull()
    .default("private"),
  lastContext: jsonb("lastContext").$type<AppUsage | null>(),
});

// 消息表 (新版本)
export const message = pgTable("Message_v2", {
  id: uuid("id").primaryKey().notNull().defaultRandom(),
  chatId: uuid("chatId").notNull().references(() => chat.id),
  role: varchar("role").notNull(),
  parts: json("parts").notNull(), // 支持多模态内容
  attachments: json("attachments").notNull(),
  createdAt: timestamp("createdAt").notNull(),
});

// 文档表 (Artifacts)
export const document = pgTable("Document", {
  id: uuid("id").notNull().defaultRandom(),
  createdAt: timestamp("createdAt").notNull(),
  title: text("title").notNull(),
  content: text("content"),
  kind: varchar("text", { enum: ["text", "code", "image", "sheet"] })
    .notNull()
    .default("text"),
  userId: uuid("userId").notNull().references(() => user.id),
});

// 投票表
export const vote = pgTable("Vote_v2", {
  chatId: uuid("chatId").notNull().references(() => chat.id),
  messageId: uuid("messageId").notNull().references(() => message.id),
  isUpvoted: boolean("isUpvoted").notNull(),
});

// 建议表 (用于文档编辑建议)
export const suggestion = pgTable("Suggestion", {
  id: uuid("id").notNull().defaultRandom(),
  documentId: uuid("documentId").notNull(),
  documentCreatedAt: timestamp("documentCreatedAt").notNull(),
  originalText: text("originalText").notNull(),
  suggestedText: text("suggestedText").notNull(),
  description: text("description"),
  isResolved: boolean("isResolved").notNull().default(false),
  userId: uuid("userId").notNull().references(() => user.id),
  createdAt: timestamp("createdAt").notNull(),
});
```

### 2. 数据访问层

```typescript
// lib/db/queries.ts

// 保存聊天记录
export async function saveChat({
  id, userId, title, visibility,
}: {
  id: string;
  userId: string;
  title: string;
  visibility: VisibilityType;
}) {
  return await db.insert(chat).values({
    id, createdAt: new Date(), userId, title, visibility,
  });
}

// 获取用户聊天历史
export async function getChatsByUserId({
  id, limit, startingAfter, endingBefore,
}: {
  id: string;
  limit: number;
  startingAfter: string | null;
  endingBefore: string | null;
}) {
  // 实现分页查询逻辑
  // 支持基于游标的分页
}

// 保存消息
export async function saveMessages({ messages }: { messages: DBMessage[] }) {
  return await db.insert(message).values(messages);
}

// 消息投票
export async function voteMessage({
  chatId, messageId, type,
}: {
  chatId: string;
  messageId: string;
  type: "up" | "down";
}) {
  // 检查是否已存在投票
  const [existingVote] = await db
    .select()
    .from(vote)
    .where(and(eq(vote.messageId, messageId)));

  if (existingVote) {
    // 更新现有投票
    return await db
      .update(vote)
      .set({ isUpvoted: type === "up" })
      .where(and(eq(vote.messageId, messageId), eq(vote.chatId, chatId)));
  }
  
  // 创建新投票
  return await db.insert(vote).values({
    chatId, messageId, isUpvoted: type === "up",
  });
}
```

## 聊天功能系统 (Chat System)

### 1. 聊天组件架构

#### 主聊天组件

```typescript
// components/chat.tsx
export function Chat({
  id, initialMessages, initialChatModel, 
  initialVisibilityType, isReadonly, autoResume,
}: ChatProps) {
  // 使用 useChat hook 管理聊天状态
  const {
    messages, setMessages, sendMessage, 
    status, stop, regenerate, resumeStream,
  } = useChat<ChatMessage>({
    id,
    messages: initialMessages,
    experimental_throttle: 100,
    generateId: generateUUID,
    transport: new DefaultChatTransport({
      api: "/api/chat",
      fetch: fetchWithErrorHandlers,
      prepareSendMessagesRequest(request) {
        return {
          body: {
            id: request.id,
            message: request.messages.at(-1),
            selectedChatModel: currentModelIdRef.current,
            selectedVisibilityType: visibilityType,
            ...request.body,
          },
        };
      },
    }),
    onData: (dataPart) => {
      // 处理流式数据
      setDataStream((ds) => (ds ? [...ds, dataPart] : []));
      if (dataPart.type === "data-usage") {
        setUsage(dataPart.data);
      }
    },
    onFinish: () => {
      // 聊天完成后更新历史记录
      mutate(unstable_serialize(getChatHistoryPaginationKey));
    },
    onError: (error) => {
      // 错误处理
      if (error instanceof ChatSDKError) {
        toast({ type: "error", description: error.message });
      }
    },
  });

  return (
    <>
      <div className="flex h-dvh min-w-0 flex-col bg-background">
        <ChatHeader chatId={id} isReadonly={isReadonly} />
        
        <Messages
          chatId={id}
          messages={messages}
          regenerate={regenerate}
          setMessages={setMessages}
          status={status}
          votes={votes}
        />
        
        <MultimodalInput
          chatId={id}
          input={input}
          messages={messages}
          sendMessage={sendMessage}
          setInput={setInput}
          status={status}
          stop={stop}
        />
      </div>
      
      <Artifact
        chatId={id}
        messages={messages}
        sendMessage={sendMessage}
        // ... other props
      />
    </>
  );
}
```

### 2. 消息处理系统

#### 多模态输入组件

```typescript
// components/multimodal-input.tsx
export function MultimodalInput({
  chatId, input, setInput, status, 
  sendMessage, attachments, setAttachments,
}: MultimodalInputProps) {
  
  const submitForm = useCallback(() => {
    sendMessage({
      role: "user",
      parts: [
        // 附件处理
        ...attachments.map((attachment) => ({
          type: "file" as const,
          url: attachment.url,
          name: attachment.name,
          mediaType: attachment.contentType,
        })),
        // 文本内容
        { type: "text", text: input },
      ],
    });
    
    // 清理状态
    setAttachments([]);
    setInput("");
  }, [input, attachments, sendMessage]);

  const uploadFile = useCallback(async (file: File) => {
    const formData = new FormData();
    formData.append("file", file);

    const response = await fetch("/api/files/upload", {
      method: "POST",
      body: formData,
    });

    if (response.ok) {
      const data = await response.json();
      return {
        url: data.url,
        name: data.pathname,
        contentType: data.contentType,
      };
    }
  }, []);

  return (
    <PromptInput onSubmit={submitForm}>
      {/* 附件预览 */}
      {attachments.length > 0 && (
        <div className="flex flex-row gap-2">
          {attachments.map((attachment) => (
            <PreviewAttachment
              key={attachment.url}
              attachment={attachment}
              onRemove={() => {/* 移除附件逻辑 */}}
            />
          ))}
        </div>
      )}
      
      {/* 文本输入区域 */}
      <PromptInputTextarea
        value={input}
        onChange={handleInput}
        placeholder="Send a message..."
      />
      
      {/* 工具栏 */}
      <PromptInputToolbar>
        <AttachmentsButton fileInputRef={fileInputRef} />
        <ModelSelector
          selectedModelId={selectedModelId}
          onModelChange={onModelChange}
        />
        <SubmitButton disabled={!input.trim()} />
      </PromptInputToolbar>
    </PromptInput>
  );
}
```

### 3. 聊天 API 路由

```typescript
// app/(chat)/api/chat/route.ts
export async function POST(request: Request) {
  const requestBody = postRequestBodySchema.parse(await request.json());
  const { id, message, selectedChatModel, selectedVisibilityType } = requestBody;
  
  // 认证检查
  const session = await auth();
  if (!session?.user) {
    return new ChatSDKError("unauthorized:chat").toResponse();
  }
  
  // 速率限制检查
  const messageCount = await getMessageCountByUserId({
    id: session.user.id,
    differenceInHours: 24,
  });
  
  if (messageCount > entitlementsByUserType[session.user.type].maxMessagesPerDay) {
    return new ChatSDKError("rate_limit:chat").toResponse();
  }
  
  // 保存用户消息
  await saveMessages([{
    chatId: id,
    id: message.id,
    role: "user",
    parts: message.parts,
    attachments: [],
    createdAt: new Date(),
  }]);
  
  // 创建 AI 响应流
  const stream = createUIMessageStream({
    execute: ({ writer: dataStream }) => {
      const result = streamText({
        model: myProvider.languageModel(selectedChatModel),
        system: systemPrompt({ selectedChatModel, requestHints }),
        messages: convertToModelMessages(uiMessages),
        stopWhen: stepCountIs(5),
        experimental_activeTools: selectedChatModel === "chat-model-reasoning" 
          ? []
          : ["getWeather", "createDocument", "updateDocument", "requestSuggestions"],
        experimental_transform: smoothStream({ chunking: "word" }),
        tools: {
          getWeather,
          createDocument: createDocument({ session, dataStream }),
          updateDocument: updateDocument({ session, dataStream }),
          requestSuggestions: requestSuggestions({ session, dataStream }),
        },
        onFinish: async ({ usage }) => {
          // 使用 TokenLens 丰富使用统计信息
          const providers = await getTokenlensCatalog();
          const modelId = myProvider.languageModel(selectedChatModel).modelId;
          const summary = getUsage({ modelId, usage, providers });
          const finalUsage = { ...usage, ...summary, modelId } as AppUsage;
          
          dataStream.write({ type: "data-usage", data: finalUsage });
        },
      });
      
      // 合并结果流
      dataStream.merge(result.toUIMessageStream({ sendReasoning: true }));
    },
    onFinish: async ({ messages }) => {
      // 保存 AI 响应消息
      await saveMessages(messages.map((msg) => ({
        id: msg.id,
        role: msg.role,
        parts: msg.parts,
        createdAt: new Date(),
        attachments: [],
        chatId: id,
      })));
    },
  });
  
  return new Response(stream.pipeThrough(new JsonToSseTransformStream()));
}
```

## Artifacts 系统 (Document Generation)

### 1. Artifacts 架构概述

Artifacts 是项目的核心特色功能，允许 AI 在聊天界面右侧创建和编辑各种类型的文档。

#### 支持的 Artifact 类型

```typescript
// components/artifact.tsx
export const artifactDefinitions = [
  textArtifact,    // 文本文档
  codeArtifact,    // 代码文档
  imageArtifact,   // 图像生成
  sheetArtifact,   // 电子表格
];

export type ArtifactKind = (typeof artifactDefinitions)[number]["kind"];

export type UIArtifact = {
  title: string;
  documentId: string;
  kind: ArtifactKind;
  content: string;
  isVisible: boolean;
  status: "streaming" | "idle";
  boundingBox: {
    top: number;
    left: number;
    width: number;
    height: number;
  };
};
```

### 2. 文档处理器架构

```typescript
// lib/artifacts/server.ts
export type DocumentHandler<T = ArtifactKind> = {
  kind: T;
  onCreateDocument: (args: CreateDocumentCallbackProps) => Promise<void>;
  onUpdateDocument: (args: UpdateDocumentCallbackProps) => Promise<void>;
};

export function createDocumentHandler<T extends ArtifactKind>(config: {
  kind: T;
  onCreateDocument: (params: CreateDocumentCallbackProps) => Promise<string>;
  onUpdateDocument: (params: UpdateDocumentCallbackProps) => Promise<string>;
}): DocumentHandler<T> {
  return {
    kind: config.kind,
    onCreateDocument: async (args) => {
      // 生成文档内容
      const draftContent = await config.onCreateDocument(args);
      
      // 保存到数据库
      if (args.session?.user?.id) {
        await saveDocument({
          id: args.id,
          title: args.title,
          content: draftContent,
          kind: config.kind,
          userId: args.session.user.id,
        });
      }
    },
    onUpdateDocument: async (args) => {
      // 更新文档内容
      const draftContent = await config.onUpdateDocument(args);
      
      // 保存新版本
      if (args.session?.user?.id) {
        await saveDocument({
          id: args.document.id,
          title: args.document.title,
          content: draftContent,
          kind: config.kind,
          userId: args.session.user.id,
        });
      }
    },
  };
}
```

### 3. 代码 Artifact 实现

```typescript
// artifacts/code/server.ts
export const codeDocumentHandler = createDocumentHandler({
  kind: "code",
  onCreateDocument: async ({ title, dataStream, session }) => {
    const result = await streamText({
      model: myProvider.languageModel("artifact-model"),
      system: codePrompt,
      prompt: title,
    });

    let draftContent = "";

    result.textStream
      .pipeThrough(
        new TransformStream({
          transform(chunk, controller) {
            draftContent += chunk;
            controller.enqueue(chunk);
          },
        })
      )
      .pipeThrough(new TextEncoderStream())
      .pipeThrough(
        new TransformStream({
          transform(chunk, controller) {
            dataStream.write({
              type: "data-stream",
              data: new TextDecoder().decode(chunk),
            });
            controller.enqueue(chunk);
          },
        })
      );

    await result.textStream.pipeTo(new WritableStream());
    return draftContent;
  },
  onUpdateDocument: async ({ document, description, dataStream, session }) => {
    const result = await streamText({
      model: myProvider.languageModel("artifact-model"),
      system: updateDocumentPrompt(document.content, "code"),
      prompt: description,
    });

    // 类似的流式更新逻辑
    // ...
    
    return updatedContent;
  },
});
```

### 4. Artifact UI 组件

```typescript
// components/artifact.tsx
export function Artifact({
  chatId, messages, sendMessage, 
  selectedModelId, isReadonly,
}: ArtifactProps) {
  const { artifact, setArtifact } = useArtifact();
  
  // 获取文档历史版本
  const { data: documents, mutate: mutateDocuments } = useSWR<Document[]>(
    artifact.documentId !== "init" 
      ? `/api/document?id=${artifact.documentId}` 
      : null,
    fetcher
  );
  
  // 内容变更处理
  const handleContentChange = useCallback((updatedContent: string) => {
    if (!artifact) return;
    
    mutate<Document[]>(
      `/api/document?id=${artifact.documentId}`,
      async (currentDocuments) => {
        const currentDocument = currentDocuments?.at(-1);
        
        if (currentDocument?.content !== updatedContent) {
          // 保存更新
          await fetch(`/api/document?id=${artifact.documentId}`, {
            method: "POST",
            body: JSON.stringify({
              title: artifact.title,
              content: updatedContent,
              kind: artifact.kind,
            }),
          });
          
          // 创建新版本记录
          const newDocument = {
            ...currentDocument,
            content: updatedContent,
            createdAt: new Date(),
          };
          
          return [...(currentDocuments || []), newDocument];
        }
        
        return currentDocuments || [];
      },
      { revalidate: false }
    );
  }, [artifact, mutate]);

  // 防抖处理内容保存
  const debouncedHandleContentChange = useDebounceCallback(
    handleContentChange, 2000
  );

  const artifactDefinition = artifactDefinitions.find(
    (definition) => definition.kind === artifact.kind
  );

  return (
    <AnimatePresence>
      {artifact.isVisible && (
        <motion.div className="fixed top-0 left-0 z-50 flex h-dvh w-dvw">
          {/* 左侧聊天面板 */}
          <motion.div className="h-dvh w-[400px] bg-muted">
            <ArtifactMessages
              messages={messages}
              regenerate={regenerate}
              // ... other props
            />
            
            <MultimodalInput
              chatId={chatId}
              sendMessage={sendMessage}
              // ... other props
            />
          </motion.div>
          
          {/* 右侧文档编辑面板 */}
          <motion.div className="flex-1 flex flex-col">
            <div className="flex justify-between p-4">
              <div>
                <div className="font-medium">{artifact.title}</div>
                {document && (
                  <div className="text-sm text-muted-foreground">
                    Updated {formatDistance(new Date(document.createdAt), new Date(), {
                      addSuffix: true,
                    })}
                  </div>
                )}
              </div>
              
              <ArtifactActions
                artifact={artifact}
                currentVersionIndex={currentVersionIndex}
                handleVersionChange={handleVersionChange}
                // ... other props
              />
            </div>
            
            {/* 文档内容渲染 */}
            <div className="flex-1 overflow-y-auto">
              <artifactDefinition.content
                content={artifact.content}
                onSaveContent={debouncedHandleContentChange}
                isCurrentVersion={isCurrentVersion}
                mode={mode}
                // ... other props
              />
            </div>
          </motion.div>
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

## UI 组件架构 (UI Components)

### 1. 设计系统

项目采用现代化的设计系统，基于以下技术栈构建：

- **基础组件**: Radix UI primitives
- **样式系统**: Tailwind CSS 4.1.13 
- **组件库**: shadcn/ui
- **动画库**: Framer Motion 11.3.19
- **图标库**: Lucide React 0.446.0

#### 设计主题

```typescript
// components/theme-provider.tsx
export function ThemeProvider({ children, ...props }: ThemeProviderProps) {
  return (
    <NextThemesProvider
      attribute="class"
      defaultTheme="system"
      disableTransitionOnChange
      enableSystem
      {...props}
    >
      {children}
    </NextThemesProvider>
  );
}
```

### 2. 组件分层架构

```
components/
├── ui/                    # 基础 UI 组件 (shadcn/ui)
│   ├── button.tsx        # 按钮组件
│   ├── input.tsx         # 输入框组件
│   ├── avatar.tsx        # 头像组件
│   ├── sidebar.tsx       # 侧边栏组件
│   └── ...               # 其他基础组件
├── elements/             # 复合元素组件
│   ├── conversation.tsx  # 对话容器
│   ├── message.tsx       # 消息组件
│   ├── prompt-input.tsx  # 提示输入组件
│   └── ...               # 其他复合组件
├── 业务组件/              # 业务特定组件
│   ├── chat.tsx          # 主聊天组件
│   ├── messages.tsx      # 消息列表
│   ├── artifact.tsx      # Artifacts 容器
│   ├── multimodal-input.tsx # 多模态输入
│   └── ...               # 其他业务组件
└── 布局组件/              # 布局相关组件
    ├── app-sidebar.tsx   # 应用侧边栏
    ├── chat-header.tsx   # 聊天头部
    └── ...               # 其他布局组件
```

### 3. 核心 UI 组件实现

#### 消息组件

```typescript
// components/elements/message.tsx
export function Message({
  message, isLoading, vote, 
  regenerate, chatId,
}: MessageProps) {
  const [mode, setMode] = useState<"view" | "edit">("view");
  
  return (
    <motion.div
      initial={{ opacity: 0, y: 5 }}
      animate={{ opacity: 1, y: 0 }}
      className="group relative mx-auto flex w-full max-w-3xl gap-4 px-4"
    >
      {/* 用户头像 */}
      <div className="flex size-8 shrink-0 select-none items-center justify-center rounded-full">
        {message.role === "user" ? (
          <div className="size-8 rounded-full bg-background border" />
        ) : (
          <div className="flex size-8 items-center justify-center rounded-full bg-primary text-primary-foreground">
            AI
          </div>
        )}
      </div>
      
      {/* 消息内容 */}
      <div className="flex min-h-8 flex-1 flex-col">
        {message.parts.map((part, index) => (
          <div key={index}>
            {part.type === "text" && (
              <div className="prose prose-sm max-w-none">
                <Markdown>{part.text}</Markdown>
              </div>
            )}
            {part.type === "file" && (
              <PreviewAttachment 
                attachment={{
                  url: part.url,
                  name: part.name || "",
                  contentType: part.mediaType || "",
                }}
              />
            )}
          </div>
        ))}
        
        {/* 消息操作 */}
        {message.role === "assistant" && (
          <MessageActions
            message={message}
            vote={vote}
            isLoading={isLoading}
            regenerate={regenerate}
          />
        )}
      </div>
    </motion.div>
  );
}
```

#### 侧边栏组件

```typescript
// components/app-sidebar.tsx
export function AppSidebar({ user }: { user: User | undefined }) {
  return (
    <Sidebar className="border-sidebar-border bg-sidebar">
      <SidebarHeader>
        <SidebarUserNav user={user} />
      </SidebarHeader>
      
      <SidebarContent>
        <SidebarHistory user={user} />
      </SidebarContent>
      
      <SidebarFooter>
        <VersionFooter />
      </SidebarFooter>
    </Sidebar>
  );
}

// 聊天历史组件
export function SidebarHistory({ user }: { user: User | undefined }) {
  const { data, mutate, size, setSize, isLoading } = useSWRInfinite(
    getChatHistoryPaginationKey,
    fetcher,
    { 
      revalidateFirstPage: false,
      revalidateOnFocus: false,
    }
  );
  
  const [searchQuery, setSearchQuery] = useState("");
  
  // 聊天记录搜索过滤
  const filteredChats = useMemo(() => {
    if (!searchQuery.trim()) return chats;
    
    return chats?.filter((chat) =>
      chat.title.toLowerCase().includes(searchQuery.toLowerCase())
    );
  }, [chats, searchQuery]);
  
  return (
    <SidebarGroup>
      {/* 搜索框 */}
      <div className="px-2 pb-2">
        <Input
          placeholder="Search chats..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          className="h-8"
        />
      </div>
      
      {/* 聊天记录列表 */}
      <SidebarGroupContent>
        {isLoading ? (
          // 骨架屏加载状态
          Array.from({ length: 5 }).map((_, index) => (
            <div key={index} className="flex items-center gap-3 px-2 py-1">
              <div className="h-4 w-4 rounded bg-muted animate-pulse" />
              <div className="h-4 flex-1 rounded bg-muted animate-pulse" />
            </div>
          ))
        ) : (
          filteredChats?.map((chat) => (
            <SidebarHistoryItem
              key={chat.id}
              chat={chat}
              onDelete={() => {
                // 删除聊天逻辑
                mutate();
              }}
            />
          ))
        )}
      </SidebarGroupContent>
      
      {/* 加载更多 */}
      {hasMore && (
        <div className="px-2 pt-2">
          <Button
            variant="ghost"
            size="sm"
            onClick={() => setSize(size + 1)}
            disabled={isLoading}
            className="w-full"
          >
            {isLoading ? "Loading..." : "Load More"}
          </Button>
        </div>
      )}
    </SidebarGroup>
  );
}
```

### 4. 响应式设计

项目采用移动优先的响应式设计策略：

```typescript
// hooks/use-mobile.ts
export function useMobile() {
  const [isMobile, setIsMobile] = useState(false);
  
  useEffect(() => {
    const checkMobile = () => {
      setIsMobile(window.innerWidth < 768);
    };
    
    checkMobile();
    window.addEventListener("resize", checkMobile);
    
    return () => window.removeEventListener("resize", checkMobile);
  }, []);
  
  return isMobile;
}

// 在组件中使用
export function ResponsiveLayout({ children }: { children: React.ReactNode }) {
  const isMobile = useMobile();
  
  return (
    <div className={cn(
      "flex h-dvh",
      isMobile ? "flex-col" : "flex-row"
    )}>
      {children}
    </div>
  );
}
```

## 自定义 Hooks (Custom Hooks)

### 1. 聊天相关 Hooks

```typescript
// hooks/use-messages.tsx
export function useMessages({ status }: { status: string }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const endRef = useRef<HTMLDivElement>(null);
  const [isAtBottom, setIsAtBottom] = useState(true);
  
  const scrollToBottom = useCallback((behavior: "auto" | "smooth" = "auto") => {
    endRef.current?.scrollIntoView({ behavior });
  }, []);
  
  // 监听滚动位置
  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;
    
    const handleScroll = () => {
      const { scrollTop, scrollHeight, clientHeight } = container;
      const isAtBottom = scrollTop + clientHeight >= scrollHeight - 100;
      setIsAtBottom(isAtBottom);
    };
    
    container.addEventListener("scroll", handleScroll);
    return () => container.removeEventListener("scroll", handleScroll);
  }, []);
  
  // 自动滚动到底部
  useEffect(() => {
    if (status === "streaming" && isAtBottom) {
      scrollToBottom("smooth");
    }
  }, [status, isAtBottom, scrollToBottom]);
  
  return {
    containerRef,
    endRef,
    isAtBottom,
    scrollToBottom,
  };
}
```

### 2. Artifact 管理 Hook

```typescript
// hooks/use-artifact.ts
interface ArtifactState {
  artifact: UIArtifact;
  metadata: Record<string, any>;
  setArtifact: (artifact: UIArtifact | ((prev: UIArtifact) => UIArtifact)) => void;
  setMetadata: (metadata: Record<string, any>) => void;
}

export function useArtifact(): ArtifactState {
  const [artifact, setArtifact] = useState<UIArtifact>({
    title: "",
    documentId: "init",
    kind: "text",
    content: "",
    isVisible: false,
    status: "idle",
    boundingBox: { top: 0, left: 0, width: 0, height: 0 },
  });
  
  const [metadata, setMetadata] = useState<Record<string, any>>({});
  
  return { artifact, metadata, setArtifact, setMetadata };
}
```

### 3. 数据流 Hook

```typescript
// hooks/use-auto-resume.ts
export function useAutoResume({
  autoResume,
  initialMessages,
  resumeStream,
  setMessages,
}: {
  autoResume: boolean;
  initialMessages: ChatMessage[];
  resumeStream: () => void;
  setMessages: (messages: ChatMessage[]) => void;
}) {
  useEffect(() => {
    if (!autoResume || initialMessages.length === 0) return;
    
    const lastMessage = initialMessages[initialMessages.length - 1];
    
    // 检查最后一条消息是否是未完成的 AI 响应
    if (
      lastMessage.role === "assistant" &&
      lastMessage.parts.length > 0 &&
      lastMessage.parts.some(part => 
        part.type === "text" && part.text.endsWith("...")
      )
    ) {
      // 恢复流式传输
      resumeStream();
    }
  }, [autoResume, initialMessages, resumeStream]);
}
```

## 性能优化 (Performance Optimization)

### 1. 组件优化

#### React.memo 使用

```typescript
// components/messages.tsx
export const Messages = memo(PureMessages, (prevProps, nextProps) => {
  // 当 Artifact 可见时跳过不必要的重渲染
  if (prevProps.isArtifactVisible && nextProps.isArtifactVisible) {
    return true;
  }
  
  // 精确比较需要重渲染的条件
  if (prevProps.status !== nextProps.status) return false;
  if (prevProps.messages.length !== nextProps.messages.length) return false;
  if (!equal(prevProps.messages, nextProps.messages)) return false;
  if (!equal(prevProps.votes, nextProps.votes)) return false;
  
  return false; // 默认重渲染
});
```

#### 虚拟化长列表

```typescript
// components/sidebar-history.tsx
export function SidebarHistory({ user }: { user: User | undefined }) {
  // 使用 SWR 无限滚动
  const { data, mutate, size, setSize, isLoading } = useSWRInfinite(
    getChatHistoryPaginationKey,
    fetcher,
    { 
      revalidateFirstPage: false,
      revalidateOnFocus: false,
      revalidateOnReconnect: false,
    }
  );
  
  // 聊天记录展平
  const chats = useMemo(() => {
    return data ? data.flatMap(page => page.chats) : [];
  }, [data]);
  
  // 实现虚拟滚动以处理大量聊天记录
  const virtualizer = useVirtualizer({
    count: chats.length,
    getScrollElement: () => scrollElementRef.current,
    estimateSize: () => 48, // 每个聊天项目的估计高度
    overscan: 10, // 预渲染额外项目数量
  });
  
  return (
    <div ref={scrollElementRef} className="flex-1 overflow-auto">
      <div style={{ height: `${virtualizer.getTotalSize()}px` }}>
        {virtualizer.getVirtualItems().map((virtualItem) => {
          const chat = chats[virtualItem.index];
          
          return (
            <div
              key={chat.id}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                height: `${virtualItem.size}px`,
                transform: `translateY(${virtualItem.start}px)`,
              }}
            >
              <SidebarHistoryItem chat={chat} />
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### 2. 数据获取优化

#### SWR 配置

```typescript
// lib/utils.ts
export const fetcher = async (url: string) => {
  const response = await fetch(url);
  
  if (!response.ok) {
    const error = new Error('Network response was not ok');
    error.status = response.status;
    throw error;
  }
  
  return response.json();
};

// SWR 全局配置
export const swrConfig = {
  fetcher,
  revalidateOnFocus: false,
  revalidateOnReconnect: true,
  errorRetryCount: 3,
  errorRetryInterval: 5000,
  dedupingInterval: 2000,
};
```

#### 缓存策略

```typescript
// app/(chat)/api/chat/route.ts
import { unstable_cache as cache } from "next/cache";

// TokenLens 目录缓存 (24小时)
const getTokenlensCatalog = cache(
  async (): Promise<ModelCatalog | undefined> => {
    try {
      return await fetchModels();
    } catch (err) {
      console.warn("TokenLens: catalog fetch failed", err);
      return;
    }
  },
  ["tokenlens-catalog"],
  { revalidate: 24 * 60 * 60 }
);
```

### 3. 流式响应优化

```typescript
// 流式文本处理
export function smoothStream({ chunking = "word" }: { chunking?: "word" | "sentence" }) {
  return new TransformStream({
    transform(chunk, controller) {
      if (chunking === "word") {
        // 按词分割以提供更平滑的流式体验
        const words = chunk.split(/(\s+)/);
        for (const word of words) {
          if (word.trim()) {
            controller.enqueue(word);
          }
        }
      } else {
        controller.enqueue(chunk);
      }
    },
  });
}
```

## 错误处理 (Error Handling)

### 1. 自定义错误类

```typescript
// lib/errors.ts
export class ChatSDKError extends Error {
  public readonly code: string;
  public readonly status: number;
  
  constructor(code: string, message?: string) {
    super(message || getDefaultMessage(code));
    this.code = code;
    this.status = getStatusCode(code);
    this.name = "ChatSDKError";
  }
  
  toResponse(): Response {
    return new Response(
      JSON.stringify({ 
        error: { code: this.code, message: this.message } 
      }),
      { 
        status: this.status,
        headers: { "Content-Type": "application/json" },
      }
    );
  }
}

function getStatusCode(code: string): number {
  if (code.startsWith("unauthorized")) return 401;
  if (code.startsWith("forbidden")) return 403;
  if (code.startsWith("not_found")) return 404;
  if (code.startsWith("rate_limit")) return 429;
  if (code.startsWith("bad_request")) return 400;
  return 500;
}

function getDefaultMessage(code: string): string {
  const messages = {
    "unauthorized:chat": "You must be logged in to chat",
    "rate_limit:chat": "You've reached your daily message limit",
    "bad_request:activate_gateway": "AI Gateway requires a valid credit card on file",
    "offline:chat": "Chat service is currently unavailable",
  };
  
  return messages[code] || "An unknown error occurred";
}
```

### 2. 全局错误边界

```typescript
// components/error-boundary.tsx
export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error?: Error }
> {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }
  
  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error("Error boundary caught error:", error, errorInfo);
    
    // 发送错误报告到监控服务
    if (typeof window !== "undefined") {
      // Sentry, LogRocket 等错误监控服务
    }
  }
  
  render() {
    if (this.state.hasError) {
      return (
        <div className="flex flex-col items-center justify-center min-h-screen">
          <h2 className="text-2xl font-bold text-red-600 mb-4">
            Something went wrong
          </h2>
          <p className="text-muted-foreground mb-4">
            We're sorry, but something unexpected happened.
          </p>
          <Button
            onClick={() => this.setState({ hasError: false, error: undefined })}
          >
            Try again
          </Button>
        </div>
      );
    }
    
    return this.props.children;
  }
}
```

### 3. API 错误处理

```typescript
// lib/utils.ts
export async function fetchWithErrorHandlers(
  url: string,
  options?: RequestInit
): Promise<Response> {
  try {
    const response = await fetch(url, options);
    
    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      
      if (errorData.error?.code) {
        throw new ChatSDKError(errorData.error.code, errorData.error.message);
      }
      
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    return response;
  } catch (error) {
    if (error instanceof ChatSDKError) {
      throw error;
    }
    
    // 网络错误或其他未知错误
    console.error("Fetch error:", error);
    throw new ChatSDKError("offline:network", "Network request failed");
  }
}
```

## 测试策略 (Testing Strategy)

### 1. 端到端测试

```typescript
// tests/e2e/chat.test.ts
import { expect, test } from "@playwright/test";

test.describe("Chat functionality", () => {
  test.beforeEach(async ({ page }) => {
    await page.goto("/");
  });
  
  test("should send and receive messages", async ({ page }) => {
    // 等待聊天界面加载
    await expect(page.getByTestId("multimodal-input")).toBeVisible();
    
    // 输入消息
    const input = page.getByTestId("multimodal-input");
    await input.fill("Hello, this is a test message");
    
    // 发送消息
    await page.getByTestId("submit-button").click();
    
    // 验证用户消息出现
    await expect(page.getByText("Hello, this is a test message")).toBeVisible();
    
    // 等待 AI 响应
    await expect(page.getByTestId("thinking-message")).toBeVisible();
    
    // 验证 AI 响应出现
    await expect(page.locator("[data-testid^='message-assistant']")).toBeVisible({
      timeout: 30000,
    });
  });
  
  test("should create and display artifacts", async ({ page }) => {
    const input = page.getByTestId("multimodal-input");
    await input.fill("Create a Python script that calculates fibonacci numbers");
    await page.getByTestId("submit-button").click();
    
    // 等待 Artifact 创建
    await expect(page.getByTestId("artifact")).toBeVisible({ timeout: 30000 });
    
    // 验证 Artifact 内容
    await expect(page.getByTestId("code-editor")).toBeVisible();
    await expect(page.getByText("fibonacci")).toBeVisible();
  });
  
  test("should handle file uploads", async ({ page }) => {
    // 创建测试文件
    const fileContent = "This is a test file content";
    const buffer = Buffer.from(fileContent);
    
    // 上传文件
    await page.getByTestId("attachments-button").click();
    await page.setInputFiles('input[type="file"]', {
      name: "test.txt",
      mimeType: "text/plain",
      buffer,
    });
    
    // 验证附件预览出现
    await expect(page.getByTestId("attachments-preview")).toBeVisible();
    await expect(page.getByText("test.txt")).toBeVisible();
    
    // 发送带附件的消息
    await page.getByTestId("multimodal-input").fill("Please analyze this file");
    await page.getByTestId("submit-button").click();
    
    // 验证消息发送成功
    await expect(page.getByText("Please analyze this file")).toBeVisible();
  });
});
```

### 2. 组件测试

```typescript
// tests/components/multimodal-input.test.tsx
import { render, screen, fireEvent, waitFor } from "@testing-library/react";
import { MultimodalInput } from "@/components/multimodal-input";

describe("MultimodalInput", () => {
  const mockProps = {
    chatId: "test-chat",
    input: "",
    setInput: vi.fn(),
    status: "ready" as const,
    stop: vi.fn(),
    attachments: [],
    setAttachments: vi.fn(),
    messages: [],
    setMessages: vi.fn(),
    sendMessage: vi.fn(),
    selectedVisibilityType: "private" as const,
    selectedModelId: "chat-model",
  };
  
  test("should render input field", () => {
    render(<MultimodalInput {...mockProps} />);
    expect(screen.getByTestId("multimodal-input")).toBeInTheDocument();
  });
  
  test("should handle text input", async () => {
    render(<MultimodalInput {...mockProps} />);
    
    const input = screen.getByTestId("multimodal-input");
    fireEvent.change(input, { target: { value: "Test message" } });
    
    await waitFor(() => {
      expect(mockProps.setInput).toHaveBeenCalledWith("Test message");
    });
  });
  
  test("should disable submit when input is empty", () => {
    render(<MultimodalInput {...mockProps} />);
    
    const submitButton = screen.getByTestId("submit-button");
    expect(submitButton).toBeDisabled();
  });
  
  test("should enable submit when input has content", () => {
    render(<MultimodalInput {...mockProps} input="Test message" />);
    
    const submitButton = screen.getByTestId("submit-button");
    expect(submitButton).toBeEnabled();
  });
});
```

## 部署与配置 (Deployment & Configuration)

### 1. 环境配置

```env
# .env.example
# Database
POSTGRES_URL=postgresql://user:password@localhost:5432/chatbot

# Authentication
AUTH_SECRET=your-auth-secret
NEXTAUTH_URL=http://localhost:3000

# AI Models
AI_GATEWAY_API_KEY=your-ai-gateway-key

# File Storage
BLOB_READ_WRITE_TOKEN=your-vercel-blob-token

# Redis (Optional - for resumable streams)
REDIS_URL=redis://localhost:6379

# Monitoring
VERCEL_ANALYTICS_ID=your-analytics-id
```

### 2. Next.js 配置

```typescript
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    ppr: true, // Partial Prerendering
    serverComponentsExternalPackages: [
      "@vercel/postgres",
      "postgres",
    ],
  },
  
  // 图片优化配置
  images: {
    domains: ["vercel.com", "blob.vercel-storage.com"],
    formats: ["image/webp", "image/avif"],
  },
  
  // Webpack 配置
  webpack: (config, { isServer }) => {
    if (!isServer) {
      // 客户端特定配置
      config.resolve.fallback = {
        ...config.resolve.fallback,
        fs: false,
      };
    }
    
    return config;
  },
  
  // 头部配置
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          {
            key: "X-Frame-Options",
            value: "DENY",
          },
          {
            key: "X-Content-Type-Options", 
            value: "nosniff",
          },
          {
            key: "Referrer-Policy",
            value: "origin-when-cross-origin",
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

### 3. 数据库迁移

```typescript
// lib/db/migrate.ts
import { drizzle } from "drizzle-orm/postgres-js";
import { migrate } from "drizzle-orm/postgres-js/migrator";
import postgres from "postgres";

async function runMigrations() {
  const connection = postgres(process.env.POSTGRES_URL!, { max: 1 });
  const db = drizzle(connection);
  
  console.log("Running migrations...");
  
  await migrate(db, { migrationsFolder: "./lib/db/migrations" });
  
  console.log("Migrations completed successfully!");
  
  await connection.end();
}

if (require.main === module) {
  runMigrations().catch((error) => {
    console.error("Migration failed:", error);
    process.exit(1);
  });
}
```

### 4. 生产环境优化

```typescript
// 生产环境配置
const isProduction = process.env.NODE_ENV === "production";

// 条件性导入生产优化模块
if (isProduction) {
  // 启用 OpenTelemetry 遥测
  import("./instrumentation");
  
  // 配置错误监控
  import("@vercel/analytics/react").then(({ Analytics }) => {
    // 分析配置
  });
}

// 性能监控
export const performanceConfig = {
  // Core Web Vitals 阈值
  thresholds: {
    FCP: 1800, // First Contentful Paint
    LCP: 2500, // Largest Contentful Paint  
    FID: 100,  // First Input Delay
    CLS: 0.1,  // Cumulative Layout Shift
  },
  
  // 监控配置
  monitoring: {
    enabled: isProduction,
    sampleRate: 0.1, // 10% 采样率
  },
};
```

## 总结

### 核心特性

1. **现代化技术栈**: Next.js 15 + React 19 + TypeScript，提供类型安全的开发体验
2. **AI 驱动**: 集成 Vercel AI Gateway，支持多种 AI 模型和工具调用
3. **实时交互**: 流式响应和实时 UI 更新，提供流畅的用户体验
4. **Artifacts 系统**: 强大的文档生成和编辑功能，支持代码、文本、图片、表格等多种格式
5. **多模态支持**: 支持文本、图片、文件等多种输入格式
6. **用户认证**: 完整的认证系统，支持常规用户和访客模式
7. **数据持久化**: PostgreSQL + Drizzle ORM，可靠的数据存储
8. **响应式设计**: 移动优先的设计，适配各种设备

### 技术亮点

1. **类型安全**: 全面的 TypeScript 类型定义，减少运行时错误
2. **性能优化**: React.memo、虚拟化、缓存策略等多重优化
3. **错误处理**: 完善的错误处理机制和用户友好的错误提示
4. **测试覆盖**: 端到端测试和单元测试，保证代码质量
5. **代码质量**: Biome + Ultracite 严格的代码规范和质量控制
6. **可扩展性**: 模块化架构，易于扩展新功能

### 最佳实践

1. **组件设计**: 单一职责原则，可复用的组件设计
2. **状态管理**: 合理使用 React 状态和 SWR 数据获取
3. **性能优化**: 避免不必要的重渲染，优化大列表性能
4. **用户体验**: 流畅的交互动画，及时的加载反馈
5. **安全性**: 输入验证、认证鉴权、XSS 防护等安全措施

这个 AI 聊天机器人项目展现了现代 Web 应用开发的最佳实践，是学习和参考全栈开发的优秀案例。通过深入理解其架构和实现原理，可以为构建类似的 AI 应用提供宝贵的经验和指导。

