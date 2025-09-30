# AI 模型集成与管理系统

> 深入解析 KiloCode 的多模型适配层、智能路由算法和成本优化策略

## AI 模型集成架构概览

KiloCode 支持超过 400 种 AI 模型，包括 OpenAI、Anthropic、Google Gemini 等主流模型提供商。这种多模型支持需要一个强大而灵活的集成架构。

### 架构设计目标

1. **统一接口**：所有模型使用相同的 API 接口
2. **智能路由**：根据任务类型选择最适合的模型
3. **成本优化**：平衡性能和成本
4. **容错处理**：模型失败时的自动切换
5. **性能监控**：实时跟踪模型性能

## 多模型适配层

### 基础提供者架构

#### 1. 抽象基类设计

```typescript
// src/api/providers/base-provider.ts
export abstract class BaseProvider implements ApiHandler {
    abstract createMessage(
        systemPrompt: string,
        messages: Anthropic.Messages.MessageParam[],
        metadata?: ApiHandlerCreateMessageMetadata,
    ): ApiStream

    abstract getModel(): { id: string; info: ModelInfo }

    /**
     * 默认 Token 计数实现使用 tiktoken
     * 提供商可以重写此方法以使用其原生 Token 计数端点
     */
    async countTokens(content: Anthropic.Messages.ContentBlockParam[]): Promise<number> {
        if (content.length === 0) {
            return 0
        }
        return countTokens(content, { useWorker: true })
    }
}
```

#### 2. API 处理器接口

```typescript
// src/api/index.ts
export interface ApiHandler {
    createMessage(
        systemPrompt: string,
        messages: Anthropic.Messages.MessageParam[],
        metadata?: ApiHandlerCreateMessageMetadata,
    ): ApiStream

    getModel(): { id: string; info: ModelInfo }

    /**
     * 统计内容块的 Token 数量
     * 所有提供者都扩展了 BaseProvider，提供默认的 tiktoken 实现
     * 但可以重写此方法以使用其原生 Token 计数端点
     */
    countTokens(content: Array<Anthropic.Messages.ContentBlockParam>): Promise<number>
}

export interface SingleCompletionHandler {
    completePrompt(prompt: string): Promise<string>
}
```

### 具体模型实现

#### 1. Anthropic 模型适配器

```typescript
// src/api/providers/anthropic.ts
export class AnthropicHandler extends BaseProvider implements SingleCompletionHandler {
    private options: ApiHandlerOptions
    private client: Anthropic

    constructor(options: ApiHandlerOptions) {
        super()
        this.options = options

        const apiKeyFieldName =
            this.options.anthropicBaseUrl && this.options.anthropicUseAuthToken
                ? "authToken"
                : "apiKey"

        this.client = new Anthropic({
            baseURL: this.options.anthropicBaseUrl || undefined,
            [apiKeyFieldName]: this.options.apiKey,
        })
    }

    async *createMessage(
        systemPrompt: string,
        messages: Anthropic.Messages.MessageParam[],
        metadata?: ApiHandlerCreateMessageMetadata,
    ): ApiStream {
        let stream: AnthropicStream<Anthropic.Messages.RawMessageStreamEvent>
        const cacheControl: CacheControlEphemeral = { type: "ephemeral" }
        let { id: modelId, betas = [], maxTokens, temperature, reasoning: thinking } = this.getModel()

        // 为支持的模型添加 1M 上下文 Beta 标志
        if (
            (modelId === "claude-sonnet-4-20250514" || modelId === "claude-4.5-sonnet") &&
            this.options.anthropicBeta1MContext
        ) {
            betas.push("context-1m-2025-08-07")
        }

        // 处理不同 Claude 模型的缓存策略
        switch (modelId) {
            case "claude-4.5-sonnet":
            case "claude-sonnet-4-20250514":
            case "claude-opus-4-1-20250805":
            case "claude-opus-4-20250514":
            case "claude-3-7-sonnet-20250219":
            case "claude-3-5-sonnet-20241022":
            case "claude-3-5-haiku-20241022":
            case "claude-3-opus-20240229":
            case "claude-3-haiku-20240307": {
                // 智能缓存控制
                const userMsgIndices = messages.reduce(
                    (acc, msg, index) => (msg.role === "user" ? [...acc, index] : acc),
                    [] as number[],
                )

                const lastUserMsgIndex = userMsgIndices[userMsgIndices.length - 1] ?? -1
                const secondLastMsgUserIndex = userMsgIndices[userMsgIndices.length - 2] ?? -1

                stream = await this.client.messages.create({
                    model: modelId,
                    max_tokens: maxTokens ?? ANTHROPIC_DEFAULT_MAX_TOKENS,
                    temperature,
                    thinking,
                    system: [{ text: systemPrompt, type: "text", cache_control: cacheControl }],
                    messages: messages.map((message, index) => {
                        if (index === lastUserMsgIndex || index === secondLastMsgUserIndex) {
                            return {
                                ...message,
                                content: typeof message.content === "string"
                                    ? [{ type: "text", text: message.content, cache_control: cacheControl }]
                                    : message.content.map((content, contentIndex) =>
                                        contentIndex === message.content.length - 1
                                            ? { ...content, cache_control: cacheControl }
                                            : content
                                    ),
                            }
                        }
                        return message
                    }),
                    betas,
                    stream: true,
                })
                break
            }
            default: {
                stream = await this.client.messages.create({
                    model: modelId,
                    max_tokens: maxTokens ?? ANTHROPIC_DEFAULT_MAX_TOKENS,
                    temperature,
                    thinking,
                    system: [{ text: systemPrompt, type: "text" }],
                    messages,
                    betas,
                    stream: true,
                })
            }
        }

        // 流式处理响应
        for await (const chunk of stream) {
            switch (chunk.type) {
                case "message_start":
                    yield {
                        type: "message_start",
                        message: chunk.message,
                    }
                    break
                case "content_block_start":
                    yield {
                        type: "content_block_start",
                        index: chunk.index,
                        content_block: chunk.content_block,
                    }
                    break
                case "content_block_delta":
                    yield {
                        type: "content_block_delta",
                        index: chunk.index,
                        delta: chunk.delta,
                    }
                    break
                case "content_block_stop":
                    yield {
                        type: "content_block_stop",
                        index: chunk.index,
                    }
                    break
                case "message_delta":
                    yield {
                        type: "message_delta",
                        delta: chunk.delta,
                        usage: chunk.usage,
                    }
                    break
                case "message_stop":
                    yield {
                        type: "message_stop",
                    }
                    break
            }
        }
    }

    getModel(): { id: string; info: ModelInfo } {
        const modelId = this.options.apiModelId || anthropicDefaultModelId
        const modelInfo = anthropicModels[modelId]

        if (!modelInfo) {
            throw new Error(`Unknown model: ${modelId}`)
        }

        return {
            id: modelId,
            info: modelInfo,
        }
    }

    async completePrompt(prompt: string): Promise<string> {
        const response = await this.client.messages.create({
            model: this.getModel().id,
            max_tokens: 1000,
            messages: [{ role: "user", content: prompt }],
        })

        return response.content[0].text
    }
}
```

#### 2. 路由提供者基类

```typescript
// src/api/providers/router-provider.ts
export abstract class RouterProvider extends BaseProvider {
    protected readonly options: ApiHandlerOptions
    protected readonly name: RouterName
    protected models: ModelRecord = {}
    protected readonly modelId?: string
    protected readonly defaultModelId: string
    protected readonly defaultModelInfo: ModelInfo
    protected readonly client: OpenAI

    constructor({
        options,
        name,
        baseURL,
        apiKey = "not-provided",
        modelId,
        defaultModelId,
        defaultModelInfo,
    }: RouterProviderOptions) {
        super()

        this.options = options
        this.name = name
        this.modelId = modelId
        this.defaultModelId = defaultModelId
        this.defaultModelInfo = defaultModelInfo

        this.client = new OpenAI({
            baseURL,
            apiKey,
            defaultHeaders: {
                ...DEFAULT_HEADERS,
                ...(options.openAiHeaders || {}),
            },
        })
    }

    // 动态获取模型列表
    public async fetchModel() {
        this.models = await getModels({
            provider: this.name,
            apiKey: this.client.apiKey,
            baseUrl: this.client.baseURL
        })
        return this.getModel()
    }

    override getModel(): { id: string; info: ModelInfo } {
        const id = this.modelId ?? this.defaultModelId

        return this.models[id]
            ? { id, info: this.models[id] }
            : { id: this.defaultModelId, info: this.defaultModelInfo }
    }

    // 检查模型是否支持温度参数
    protected supportsTemperature(modelId: string): boolean {
        return !TEMPERATURE_UNSUPPORTED_PREFIXES.some((prefix) =>
            modelId.startsWith(prefix)
        )
    }
}
```

### 模型工厂

#### 1. 模型构建器

```typescript
// src/api/index.ts
export function buildApiHandler(configuration: ProviderSettings): ApiHandler {
    const { apiProvider, ...options } = configuration

    switch (apiProvider) {
        case "anthropic":
            return new AnthropicHandler(options)
        case "openai":
            return new OpenAiHandler(options)
        case "gemini":
            return new GeminiHandler(options)
        case "vertex":
            return new VertexHandler(options)
        case "bedrock":
            return new AwsBedrockHandler(options)
        case "openrouter":
            return new OpenRouterHandler(options)
        case "lmstudio":
            return new LmStudioHandler(options)
        case "ollama":
            return new OllamaHandler(options)
        case "groq":
            return new GroqHandler(options)
        case "huggingface":
            return new HuggingFaceHandler(options)
        case "together":
            return new TogetherHandler(options)
        case "mistral":
            return new MistralHandler(options)
        case "cerebras":
            return new CerebrasHandler(options)
        case "deepseek":
            return new DeepSeekHandler(options)
        case "perplexity":
            return new PerplexityHandler(options)
        case "codeium":
            return new CodeiumHandler(options)
        case "codestral":
            return new CodestralHandler(options)
        case "xai":
            return new XAIHandler(options)
        case "nvidia":
            return new NvidiaHandler(options)
        case "openrouter":
            return new OpenRouterHandler(options)
        case "fireworks":
            return new FireworksHandler(options)
        case "moonshot":
            return new MoonshotHandler(options)
        case "zhipuai":
            return new ZhipuaiHandler(options)
        case "minimax":
            return new MinimaxHandler(options)
        case "01ai":
            return new ZeroOneAIHandler(options)
        case "stepfun":
            return new StepFunHandler(options)
        case "hyperbolic":
            return new HyperbolicHandler(options)
        case "novita":
            return new NovitaHandler(options)
        case "replicate":
            return new ReplicateHandler(options)
        case "anyscale":
            return new AnyScaleHandler(options)
        case "vllm":
            return new VLLMHandler(options)
        case "kilocode-openrouter": // KiloCode 特有
            return new KilocodeOpenrouterHandler(options)
        case "virtual-quota-fallback":
            return new VirtualQuotaFallbackHandler(options)
        case "gemini-cli":
            return new GeminiCliHandler(options)
        case "claude-code":
            return new ClaudeCodeHandler(options)
        case "qwen-code":
            return new QwenCodeHandler(options)
        case "human-relay":
            return new HumanRelayHandler(options)
        case "fake-ai":
            return new FakeAIHandler(options)
        default:
            throw new Error(`Unknown API provider: ${apiProvider}`)
    }
}
```

## 智能路由算法

### 任务类型分析

#### 1. 任务分类器

```typescript
// src/core/routing/TaskClassifier.ts
export class TaskClassifier {
    private keywords: Map<TaskType, string[]> = new Map([
        [TaskType.CODE_GENERATION, [
            "generate", "create", "write", "implement", "develop", "code", "function", "class"
        ]],
        [TaskType.CODE_REFACTORING, [
            "refactor", "improve", "optimize", "restructure", "clean", "modularize"
        ]],
        [TaskType.DEBUGGING, [
            "debug", "fix", "error", "bug", "issue", "problem", "troubleshoot"
        ]],
        [TaskType.DOCUMENTATION, [
            "document", "explain", "describe", "comment", "readme", "guide"
        ]],
        [TaskType.TESTING, [
            "test", "unit test", "integration test", "spec", "assert", "verify"
        ]],
        [TaskType.REVIEW, [
            "review", "analyze", "critique", "suggest", "improvement", "feedback"
        ]]
    ])

    classifyTask(description: string, context?: TaskContext): TaskClassification {
        const lowerDesc = description.toLowerCase()
        let maxScore = 0
        let bestType = TaskType.GENERAL

        // 基于关键词分类
        for (const [type, keywords] of this.keywords.entries()) {
            const score = keywords.reduce((sum, keyword) => {
                return sum + (lowerDesc.includes(keyword) ? 1 : 0)
            }, 0)

            if (score > maxScore) {
                maxScore = score
                bestType = type
            }
        }

        // 基于上下文调整
        if (context) {
            const contextScore = this.calculateContextScore(context, bestType)
            if (contextScore > maxScore * 0.5) {
                bestType = this.adjustTypeBasedOnContext(bestType, context)
            }
        }

        return {
            type: bestType,
            confidence: maxScore / Math.max(...this.keywords.values().map(k => k.length)),
            factors: this.getDecisionFactors(description, context)
        }
    }

    private calculateContextScore(context: TaskContext, type: TaskType): number {
        let score = 0

        // 文件类型影响
        if (context.fileExtensions) {
            if (type === TaskType.CODE_GENERATION && context.fileExtensions.some(ext =>
                ['.ts', '.js', '.py', '.java', '.cpp'].includes(ext))) {
                score += 2
            }
        }

        // 代码复杂度影响
        if (context.complexity === 'high' && type === TaskType.CODE_REFACTORING) {
            score += 2
        }

        return score
    }

    private adjustTypeBasedOnContext(type: TaskType, context: TaskContext): TaskType {
        // 根据上下文调整任务类型
        if (type === TaskType.GENERAL && context.fileExtensions?.includes('.md')) {
            return TaskType.DOCUMENTATION
        }

        return type
    }

    private getDecisionFactors(description: string, context?: TaskContext): string[] {
        const factors: string[] = []

        // 添加关键词因素
        for (const [type, keywords] of this.keywords.entries()) {
            const matches = keywords.filter(keyword =>
                description.toLowerCase().includes(keyword)
            )
            if (matches.length > 0) {
                factors.push(`${type}: ${matches.join(', ')}`)
            }
        }

        // 添加上下文因素
        if (context) {
            if (context.fileExtensions) {
                factors.push(`Files: ${context.fileExtensions.join(', ')}`)
            }
            if (context.complexity) {
                factors.push(`Complexity: ${context.complexity}`)
            }
        }

        return factors
    }
}
```

#### 2. 模型能力矩阵

```typescript
// src/core/routing/ModelCapabilityMatrix.ts
export interface ModelCapability {
    codeGeneration: number        // 0-1 分
    codeRefactoring: number       // 0-1 分
    debugging: number            // 0-1 分
    documentation: number        // 0-1 分
    testing: number              // 0-1 分
    review: number               // 0-1 分
    contextLength: number       // 最大上下文长度
    speed: number               // 0-1 分 (响应速度)
    cost: number                // 0-1 分 (成本效益)
    reliability: number         // 0-1 分 (可靠性)
}

export class ModelCapabilityMatrix {
    private capabilities: Map<string, ModelCapability> = new Map([
        ["claude-3-5-sonnet-20241022", {
            codeGeneration: 0.95,
            codeRefactoring: 0.92,
            debugging: 0.90,
            documentation: 0.93,
            testing: 0.88,
            review: 0.94,
            contextLength: 200000,
            speed: 0.75,
            cost: 0.60,
            reliability: 0.92
        }],
        ["gpt-4", {
            codeGeneration: 0.93,
            codeRefactoring: 0.90,
            debugging: 0.88,
            documentation: 0.91,
            testing: 0.85,
            review: 0.92,
            contextLength: 128000,
            speed: 0.70,
            cost: 0.50,
            reliability: 0.90
        }],
        ["gemini-pro", {
            codeGeneration: 0.88,
            codeRefactoring: 0.85,
            debugging: 0.82,
            documentation: 0.87,
            testing: 0.80,
            review: 0.86,
            contextLength: 32000,
            speed: 0.85,
            cost: 0.75,
            reliability: 0.85
        }]
    ])

    getCapability(modelId: string): ModelCapability | undefined {
        return this.capabilities.get(modelId)
    }

    addCapability(modelId: string, capability: ModelCapability): void {
        this.capabilities.set(modelId, capability)
    }

    // 获取最适合的模型
    getBestModelForTask(taskType: TaskType, availableModels: string[]): string | null {
        if (availableModels.length === 0) return null

        const taskKey = this.getTaskKey(taskType)
        let bestModel = availableModels[0]
        let bestScore = 0

        for (const modelId of availableModels) {
            const capability = this.capabilities.get(modelId)
            if (!capability) continue

            const score = this.calculateScore(capability, taskKey)
            if (score > bestScore) {
                bestScore = score
                bestModel = modelId
            }
        }

        return bestModel
    }

    private getTaskKey(taskType: TaskType): keyof ModelCapability {
        switch (taskType) {
            case TaskType.CODE_GENERATION: return "codeGeneration"
            case TaskType.CODE_REFACTORING: return "codeRefactoring"
            case TaskType.DEBUGGING: return "debugging"
            case TaskType.DOCUMENTATION: return "documentation"
            case TaskType.TESTING: return "testing"
            case TaskType.REVIEW: return "review"
            default: return "codeGeneration"
        }
    }

    private calculateScore(capability: ModelCapability, taskKey: keyof ModelCapability): number {
        // 综合评分：主能力 + 速度 + 可靠性 - 成本
        const primaryScore = capability[taskKey]
        const speedBonus = capability.speed * 0.2
        const reliabilityBonus = capability.reliability * 0.3
        const costPenalty = capability.cost * 0.1

        return primaryScore + speedBonus + reliabilityBonus - costPenalty
    }
}
```

### 路由决策引擎

#### 1. 智能路由器

```typescript
// src/core/routing/IntelligentRouter.ts
export class IntelligentRouter {
    private classifier: TaskClassifier
    private capabilityMatrix: ModelCapabilityMatrix
    private performanceTracker: ModelPerformanceTracker
    private costCalculator: CostCalculator

    constructor() {
        this.classifier = new TaskClassifier()
        this.capabilityMatrix = new ModelCapabilityMatrix()
        this.performanceTracker = new ModelPerformanceTracker()
        this.costCalculator = new CostCalculator()
    }

    async selectModel(
        request: ModelSelectionRequest
    ): Promise<ModelSelectionResult> {
        const {
            description,
            context,
            availableModels,
            budget,
            performanceRequirements,
            fallbackModels = []
        } = request

        // 1. 任务分类
        const classification = this.classifier.classifyTask(description, context)

        // 2. 模型筛选
        const candidateModels = await this.filterCandidates(
            availableModels,
            context,
            budget,
            performanceRequirements
        )

        // 3. 性能评估
        const performanceScores = await this.performanceTracker.getScores(candidateModels)

        // 4. 成本评估
        const costEstimates = await this.costCalculator.estimateCosts(
            candidateModels,
            description,
            context
        )

        // 5. 综合评分
        const rankings = this.calculateRankings(
            candidateModels,
            classification,
            performanceScores,
            costEstimates
        )

        // 6. 选择最佳模型
        const selectedModel = this.selectBestModel(rankings, fallbackModels)

        return {
            selectedModel,
            alternatives: rankings.slice(1, 4).map(r => r.modelId),
            reasoning: this.generateReasoning(selectedModel, classification, rankings),
            estimatedCost: costEstimates.get(selectedModel) || 0,
            confidence: classification.confidence
        }
    }

    private async filterCandidates(
        models: string[],
        context?: TaskContext,
        budget?: number,
        performance?: PerformanceRequirements
    ): Promise<string[]> {
        let candidates = [...models]

        // 上下文长度筛选
        if (context?.estimatedTokens) {
            candidates = candidates.filter(modelId => {
                const capability = this.capabilityMatrix.getCapability(modelId)
                return capability && capability.contextLength >= context.estimatedTokens!
            })
        }

        // 预算筛选
        if (budget) {
            const costEstimates = await this.costCalculator.estimateCosts(candidates, "", context)
            candidates = candidates.filter(modelId =>
                (costEstimates.get(modelId) || 0) <= budget
            )
        }

        // 性能要求筛选
        if (performance) {
            const scores = await this.performanceTracker.getScores(candidates)
            candidates = candidates.filter(modelId => {
                const score = scores.get(modelId) || 0
                return score >= performance.minScore
            })
        }

        return candidates.length > 0 ? candidates : models // 至少返回原始模型
    }

    private calculateRankings(
        models: string[],
        classification: TaskClassification,
        performanceScores: Map<string, number>,
        costEstimates: Map<string, number>
    ): ModelRanking[] {
        const rankings: ModelRanking[] = []

        for (const modelId of models) {
            const capability = this.capabilityMatrix.getCapability(modelId)
            if (!capability) continue

            const taskKey = this.getTaskKey(classification.type)
            const capabilityScore = capability[taskKey] * classification.confidence
            const performanceScore = performanceScores.get(modelId) || 0.5
            const costScore = 1 - Math.min(costEstimates.get(modelId) || 0, 1) // 成本越低分数越高

            const totalScore = (
                capabilityScore * 0.5 +      // 50% 权重给能力
                performanceScore * 0.3 +     // 30% 权重给性能
                costScore * 0.2             // 20% 权重给成本
            )

            rankings.push({
                modelId,
                score: totalScore,
                capabilityScore,
                performanceScore,
                costScore,
                factors: {
                    taskFit: capabilityScore,
                    reliability: capability.reliability,
                    speed: capability.speed,
                    costEffectiveness: costScore
                }
            })
        }

        return rankings.sort((a, b) => b.score - a.score)
    }

    private selectBestModel(rankings: ModelRanking[], fallbackModels: string[]): string {
        if (rankings.length === 0) {
            return fallbackModels[0] || "claude-3-5-sonnet-20241022" // 默认模型
        }

        const best = rankings[0]

        // 如果最佳模型分数太低，考虑使用备用模型
        if (best.score < 0.6 && fallbackModels.length > 0) {
            const fallbackRankings = rankings.filter(r =>
                fallbackModels.includes(r.modelId)
            )
            return fallbackRankings[0]?.modelId || best.modelId
        }

        return best.modelId
    }

    private generateReasoning(
        selectedModel: string,
        classification: TaskClassification,
        rankings: ModelRanking[]
    ): RoutingReasoning {
        const selected = rankings.find(r => r.modelId === selectedModel)
        if (!selected) {
            return { primary: "Default model selected", factors: [] }
        }

        const factors: string[] = []

        // 任务适配性
        if (selected.capabilityScore > 0.8) {
            factors.push(`Excellent fit for ${classification.type} tasks`)
        } else if (selected.capabilityScore > 0.6) {
            factors.push(`Good fit for ${classification.type} tasks`)
        }

        // 性能因素
        if (selected.performanceScore > 0.8) {
            factors.push("High performance and reliability")
        } else if (selected.performanceScore < 0.5) {
            factors.push("Performance could be improved")
        }

        // 成本因素
        if (selected.costScore > 0.7) {
            factors.push("Cost-effective choice")
        } else if (selected.costScore < 0.3) {
            factors.push("Higher cost option")
        }

        return {
            primary: `Selected ${selectedModel} based on overall score of ${(selected.score * 100).toFixed(1)}%`,
            factors,
            confidence: classification.confidence
        }
    }

    private getTaskKey(taskType: TaskType): keyof ModelCapability {
        switch (taskType) {
            case TaskType.CODE_GENERATION: return "codeGeneration"
            case TaskType.CODE_REFACTORING: return "codeRefactoring"
            case TaskType.DEBUGGING: return "debugging"
            case TaskType.DOCUMENTATION: return "documentation"
            case TaskType.TESTING: return "testing"
            case TaskType.REVIEW: return "review"
            default: return "codeGeneration"
        }
    }
}
```

## 成本优化策略

### 成本计算器

#### 1. 令牌成本计算

```typescript
// src/core/cost/CostCalculator.ts
export class CostCalculator {
    private pricing: Map<string, ModelPricing> = new Map([
        ["claude-3-5-sonnet-20241022", {
            inputTokenPrice: 0.000003,  // $3 per 1M tokens
            outputTokenPrice: 0.000015, // $15 per 1M tokens
            currency: "USD"
        }],
        ["gpt-4", {
            inputTokenPrice: 0.00003,
            outputTokenPrice: 0.00006,
            currency: "USD"
        }],
        ["gemini-pro", {
            inputTokenPrice: 0.000001,
            outputTokenPrice: 0.000002,
            currency: "USD"
        }]
    ])

    async estimateCosts(
        models: string[],
        description: string,
        context?: TaskContext
    ): Promise<Map<string, number>> {
        const costs = new Map<string, number>()

        for (const modelId of models) {
            const pricing = this.pricing.get(modelId)
            if (!pricing) {
                costs.set(modelId, 0) // 未知模型成本为0
                continue
            }

            // 估算输入和输出令牌数
            const estimatedInputTokens = await this.estimateInputTokens(description, context)
            const estimatedOutputTokens = await this.estimateOutputTokens(description, context)

            const inputCost = estimatedInputTokens * pricing.inputTokenPrice
            const outputCost = estimatedOutputTokens * pricing.outputTokenPrice

            costs.set(modelId, inputCost + outputCost)
        }

        return costs
    }

    private async estimateInputTokens(description: string, context?: TaskContext): Promise<number> {
        let totalContent = description

        // 添加上下文内容
        if (context?.fileContents) {
            totalContent += "\n" + Object.values(context.fileContents).join("\n")
        }

        // 粗略估算：4字符 ≈ 1 token
        return Math.ceil(totalContent.length / 4)
    }

    private async estimateOutputTokens(description: string, context?: TaskContext): Promise<number> {
        // 基于任务类型估算输出长度
        const taskType = new TaskClassifier().classifyTask(description, context).type

        switch (taskType) {
            case TaskType.CODE_GENERATION:
                return 1000 // 通常生成较长的代码
            case TaskType.CODE_REFACTORING:
                return 800 // 中等长度输出
            case TaskType.DEBUGGING:
                return 600 // 相对较短的解决方案
            case TaskType.DOCUMENTATION:
                return 1200 // 详细文档
            case TaskType.TESTING:
                return 900 // 测试代码
            case TaskType.REVIEW:
                return 700 // 评审意见
            default:
                return 500 // 默认输出长度
        }
    }

    // 实际成本计算（基于真实令牌使用）
    calculateActualCost(
        modelId: string,
        inputTokens: number,
        outputTokens: number
    ): number {
        const pricing = this.pricing.get(modelId)
        if (!pricing) return 0

        return (
            inputTokens * pricing.inputTokenPrice +
            outputTokens * pricing.outputTokenPrice
        )
    }

    // 添加或更新模型定价
    updatePricing(modelId: string, pricing: ModelPricing): void {
        this.pricing.set(modelId, pricing)
    }

    // 获取模型定价信息
    getPricing(modelId: string): ModelPricing | undefined {
        return this.pricing.get(modelId)
    }
}
```

### 预算管理器

#### 1. 预算控制系统

```typescript
// src/core/cost/BudgetManager.ts
export class BudgetManager {
    private dailyBudget: number
    private monthlyBudget: number
    private dailySpent: number = 0
    private monthlySpent: number = 0
    private lastReset: Date = new Date()
    private costCalculator: CostCalculator
    private alerts: BudgetAlert[] = []

    constructor(dailyBudget: number, monthlyBudget: number) {
        this.dailyBudget = dailyBudget
        this.monthlyBudget = monthlyBudget
        this.costCalculator = new CostCalculator()
    }

    async checkBudgetAllowance(
        modelId: string,
        description: string,
        context?: TaskContext
    ): Promise<BudgetCheckResult> {
        // 重置每日预算
        this.resetDailyIfNeeded()

        // 估算成本
        const estimatedCosts = await this.costCalculator.estimateCosts(
            [modelId],
            description,
            context
        )
        const estimatedCost = estimatedCosts.get(modelId) || 0

        // 检查预算限制
        const dailyRemaining = this.dailyBudget - this.dailySpent
        const monthlyRemaining = this.monthlyBudget - this.monthlySpent

        const dailyAllowed = estimatedCost <= dailyRemaining
        const monthlyAllowed = estimatedCost <= monthlyRemaining

        const allowed = dailyAllowed && monthlyAllowed

        // 生成警告
        const warnings: string[] = []
        if (!dailyAllowed) {
            warnings.push(`Daily budget exceeded. Required: $${estimatedCost.toFixed(4)}, Available: $${dailyRemaining.toFixed(4)}`)
        }
        if (!monthlyAllowed) {
            warnings.push(`Monthly budget exceeded. Required: $${estimatedCost.toFixed(4)}, Available: $${monthlyRemaining.toFixed(4)}`)
        }

        // 检查预算使用率警告
        const dailyUsagePercent = (this.dailySpent / this.dailyBudget) * 100
        const monthlyUsagePercent = (this.monthlySpent / this.monthlyBudget) * 100

        if (dailyUsagePercent > 80 && dailyUsagePercent < 100) {
            warnings.push(`Daily budget ${dailyUsagePercent.toFixed(1)}% used`)
        }
        if (monthlyUsagePercent > 80 && monthlyUsagePercent < 100) {
            warnings.push(`Monthly budget ${monthlyUsagePercent.toFixed(1)}% used`)
        }

        return {
            allowed,
            estimatedCost,
            dailyRemaining,
            monthlyRemaining,
            warnings,
            dailyUsagePercent,
            monthlyUsagePercent
        }
    }

    recordExpense(cost: number): void {
        this.dailySpent += cost
        this.monthlySpent += cost

        // 检查并触发警报
        this.checkAlerts()
    }

    private resetDailyIfNeeded(): void {
        const now = new Date()
        const isNewDay = now.getDate() !== this.lastReset.getDate()

        if (isNewDay) {
            this.dailySpent = 0
            this.lastReset = now
        }
    }

    private checkAlerts(): void {
        // 每日预算警报
        if (this.dailySpent > this.dailyBudget * 0.8) {
            this.addAlert({
                type: "daily_budget_warning",
                threshold: 80,
                current: this.dailySpent,
                budget: this.dailyBudget,
                timestamp: new Date()
            })
        }

        if (this.dailySpent > this.dailyBudget) {
            this.addAlert({
                type: "daily_budget_exceeded",
                threshold: 100,
                current: this.dailySpent,
                budget: this.dailyBudget,
                timestamp: new Date()
            })
        }

        // 每月预算警报
        if (this.monthlySpent > this.monthlyBudget * 0.8) {
            this.addAlert({
                type: "monthly_budget_warning",
                threshold: 80,
                current: this.monthlySpent,
                budget: this.monthlyBudget,
                timestamp: new Date()
            })
        }

        if (this.monthlySpent > this.monthlyBudget) {
            this.addAlert({
                type: "monthly_budget_exceeded",
                threshold: 100,
                current: this.monthlySpent,
                budget: this.monthlyBudget,
                timestamp: new Date()
            })
        }
    }

    private addAlert(alert: BudgetAlert): void {
        // 避免重复警报
        const recentAlert = this.alerts.find(a =>
            a.type === alert.type &&
            Date.now() - a.timestamp.getTime() < 3600000 // 1小时内不重复
        )

        if (!recentAlert) {
            this.alerts.push(alert)
            // 这里可以添加通知逻辑
        }
    }

    // 获取预算使用情况
    getBudgetStatus(): BudgetStatus {
        this.resetDailyIfNeeded()

        return {
            daily: {
                budget: this.dailyBudget,
                spent: this.dailySpent,
                remaining: this.dailyBudget - this.dailySpent,
                usagePercent: (this.dailySpent / this.dailyBudget) * 100
            },
            monthly: {
                budget: this.monthlyBudget,
                spent: this.monthlySpent,
                remaining: this.monthlyBudget - this.monthlySpent,
                usagePercent: (this.monthlySpent / this.monthlyBudget) * 100
            },
            alerts: this.alerts.filter(a => Date.now() - a.timestamp.getTime() < 86400000) // 最近24小时的警报
        }
    }

    // 设置预算
    setBudgets(daily: number, monthly: number): void {
        this.dailyBudget = daily
        this.monthlyBudget = monthly
    }

    // 重置预算
    resetBudgets(): void {
        this.dailySpent = 0
        this.monthlySpent = 0
        this.alerts = []
    }
}
```

### 缓存策略

#### 1. 响应缓存系统

```typescript
// src/core/cache/ResponseCache.ts
export class ResponseCache {
    private cache: Map<string, CacheEntry> = new Map()
    private maxSize: number
    private ttl: number // Time to live in milliseconds

    constructor(maxSize: number = 1000, ttl: number = 3600000) { // 1小时TTL
        this.maxSize = maxSize
        this.ttl = ttl
    }

    // 生成缓存键
    private generateKey(
        systemPrompt: string,
        messages: Anthropic.Messages.MessageParam[],
        modelId: string
    ): string {
        const content = JSON.stringify({
            systemPrompt,
            messages: messages.map(m => ({
                role: m.role,
                content: typeof m.content === 'string' ? m.content : m.content
            })),
            modelId
        })

        // 使用简单的哈希算法
        let hash = 0
        for (let i = 0; i < content.length; i++) {
            const char = content.charCodeAt(i)
            hash = ((hash << 5) - hash) + char
            hash = hash & hash // 转换为32位整数
        }

        return `${modelId}:${Math.abs(hash).toString(36)}`
    }

    // 获取缓存响应
    get(
        systemPrompt: string,
        messages: Anthropic.Messages.MessageParam[],
        modelId: string
    ): CacheResponse | null {
        const key = this.generateKey(systemPrompt, messages, modelId)
        const entry = this.cache.get(key)

        if (!entry) return null

        // 检查是否过期
        if (Date.now() - entry.timestamp > this.ttl) {
            this.cache.delete(key)
            return null
        }

        // 更新访问时间（LRU策略）
        entry.lastAccessed = Date.now()

        return {
            response: entry.response,
            tokens: entry.tokens,
            cost: entry.cost,
            cached: true
        }
    }

    // 存储缓存响应
    set(
        systemPrompt: string,
        messages: Anthropic.Messages.MessageParam[],
        modelId: string,
        response: string,
        tokens: { input: number; output: number },
        cost: number
    ): void {
        const key = this.generateKey(systemPrompt, messages, modelId)

        // 检查缓存大小限制
        if (this.cache.size >= this.maxSize) {
            this.evictLRU()
        }

        this.cache.set(key, {
            response,
            tokens,
            cost,
            timestamp: Date.now(),
            lastAccessed: Date.now()
        })
    }

    // LRU淘汰策略
    private evictLRU(): void {
        let oldestKey: string | null = null
        let oldestTime = Date.now()

        for (const [key, entry] of this.cache.entries()) {
            if (entry.lastAccessed < oldestTime) {
                oldestTime = entry.lastAccessed
                oldestKey = key
            }
        }

        if (oldestKey) {
            this.cache.delete(oldestKey)
        }
    }

    // 清理过期缓存
    cleanup(): void {
        const now = Date.now()
        for (const [key, entry] of this.cache.entries()) {
            if (now - entry.timestamp > this.ttl) {
                this.cache.delete(key)
            }
        }
    }

    // 获取缓存统计
    getStats(): CacheStats {
        let totalSize = 0
        let hitRate = 0

        for (const entry of this.cache.values()) {
            totalSize += entry.response.length
        }

        return {
            size: this.cache.size,
            maxSize: this.maxSize,
            totalMemoryBytes: totalSize,
            hitRate
        }
    }

    // 清空缓存
    clear(): void {
        this.cache.clear()
    }
}
```

## 性能监控

### 模型性能追踪

#### 1. 性能监控器

```typescript
// src/core/monitoring/ModelPerformanceTracker.ts
export class ModelPerformanceTracker {
    private metrics: Map<string, ModelMetrics> = new Map()
    private recentRequests: Array<RequestRecord> = []
    private maxRecentRequests = 1000

    recordRequest(
        modelId: string,
        request: RequestRecord
    ): void {
        // 更新模型指标
        if (!this.metrics.has(modelId)) {
            this.metrics.set(modelId, {
                totalRequests: 0,
                successfulRequests: 0,
                failedRequests: 0,
                averageLatency: 0,
                averageTokensPerSecond: 0,
                totalTokens: 0,
                totalCost: 0,
                uptime: 1,
                lastUsed: Date.now()
            })
        }

        const metrics = this.metrics.get(modelId)!
        metrics.totalRequests++
        metrics.lastUsed = Date.now()

        if (request.success) {
            metrics.successfulRequests++
        } else {
            metrics.failedRequests++
        }

        // 更新延迟指标
        if (request.latency) {
            const currentAvg = metrics.averageLatency
            const totalRequests = metrics.totalRequests
            metrics.averageLatency = (currentAvg * (totalRequests - 1) + request.latency) / totalRequests
        }

        // 更新令牌处理速度
        if (request.tokens && request.duration) {
            const tokensPerSecond = request.tokens / (request.duration / 1000)
            const currentAvg = metrics.averageTokensPerSecond
            const totalRequests = metrics.totalRequests
            metrics.averageTokensPerSecond = (currentAvg * (totalRequests - 1) + tokensPerSecond) / totalRequests
        }

        // 更新总令牌数和成本
        if (request.tokens) {
            metrics.totalTokens += request.tokens
        }
        if (request.cost) {
            metrics.totalCost += request.cost
        }

        // 记录最近的请求
        this.recentRequests.push({
            ...request,
            modelId,
            timestamp: Date.now()
        })

        // 保持最近请求列表大小
        if (this.recentRequests.length > this.maxRecentRequests) {
            this.recentRequests.shift()
        }
    }

    getModelMetrics(modelId: string): ModelMetrics | undefined {
        return this.metrics.get(modelId)
    }

    getAvailableModels(): string[] {
        return Array.from(this.metrics.keys())
    }

    getScores(models: string[]): Map<string, number> {
        const scores = new Map<string, number>()

        for (const modelId of models) {
            const metrics = this.metrics.get(modelId)
            if (!metrics) {
                scores.set(modelId, 0.5) // 默认分数
                continue
            }

            // 计算综合性能分数
            const successRate = metrics.successfulRequests / metrics.totalRequests
            const latencyScore = Math.max(0, 1 - (metrics.averageLatency / 30000)) // 30秒为基准
            const speedScore = Math.min(1, metrics.averageTokensPerSecond / 100) // 100 tokens/秒为基准

            const overallScore = (
                successRate * 0.5 +      // 50% 权重给成功率
                latencyScore * 0.3 +     // 30% 权重给延迟
                speedScore * 0.2         // 20% 权重给速度
            )

            scores.set(modelId, overallScore)
        }

        return scores
    }

    getRecentRequests(limit: number = 100): Array<RequestRecord & { modelId: string }> {
        return this.recentRequests.slice(-limit)
    }

    getHealthStatus(): HealthStatus {
        const totalModels = this.metrics.size
        let healthyModels = 0
        let degradedModels = 0
        let unhealthyModels = 0

        for (const metrics of this.metrics.values()) {
            const successRate = metrics.successfulRequests / metrics.totalRequests
            const avgLatency = metrics.averageLatency

            if (successRate >= 0.95 && avgLatency < 10000) { // 95%成功率，延迟<10秒
                healthyModels++
            } else if (successRate >= 0.8 && avgLatency < 30000) { // 80%成功率，延迟<30秒
                degradedModels++
            } else {
                unhealthyModels++
            }
        }

        return {
            totalModels,
            healthyModels,
            degradedModels,
            unhealthyModels,
            overallHealth: totalModels === 0 ? "unknown" :
                unhealthyModels / totalModels > 0.2 ? "unhealthy" :
                degradedModels / totalModels > 0.3 ? "degraded" : "healthy"
        }
    }

    resetMetrics(modelId?: string): void {
        if (modelId) {
            this.metrics.delete(modelId)
        } else {
            this.metrics.clear()
        }
    }
}
```

### 质量评估器

#### 1. 响应质量评估

```typescript
// src/core/quality/ResponseQualityEvaluator.ts
export class ResponseQualityEvaluator {
    private evaluationCriteria: EvaluationCriteria[] = [
        {
            name: "relevance",
            weight: 0.3,
            evaluate: this.evaluateRelevance.bind(this)
        },
        {
            name: "completeness",
            weight: 0.25,
            evaluate: this.evaluateCompleteness.bind(this)
        },
        {
            name: "accuracy",
            weight: 0.25,
            evaluate: this.evaluateAccuracy.bind(this)
        },
        {
            name: "clarity",
            weight: 0.2,
            evaluate: this.evaluateClarity.bind(this)
        }
    ]

    async evaluateResponse(
        request: EvaluationRequest
    ): Promise<EvaluationResult> {
        const {
            originalPrompt,
            response,
            modelId,
            context,
            expectedOutput
        } = request

        const scores: Record<string, number> = {}
        let totalScore = 0

        // 执行各项评估
        for (const criterion of this.evaluationCriteria) {
            const score = await criterion.evaluate(
                originalPrompt,
                response,
                context,
                expectedOutput
            )
            scores[criterion.name] = score
            totalScore += score * criterion.weight
        }

        // 计算置信度
        const confidence = this.calculateConfidence(scores)

        // 生成改进建议
        const suggestions = await this.generateSuggestions(scores, response)

        return {
            modelId,
            totalScore,
            scores,
            confidence,
            suggestions,
            timestamp: Date.now()
        }
    }

    private async evaluateRelevance(
        prompt: string,
        response: string,
        context?: TaskContext
    ): Promise<number> {
        // 使用嵌入向量计算相关性
        const promptEmbedding = await this.getEmbedding(prompt)
        const responseEmbedding = await this.getEmbedding(response)

        const similarity = this.cosineSimilarity(promptEmbedding, responseEmbedding)

        // 考虑上下文相关性
        let contextBonus = 0
        if (context?.fileContents) {
            const contextRelevance = await this.calculateContextRelevance(response, context.fileContents)
            contextBonus = contextRelevance * 0.2
        }

        return Math.min(1, similarity + contextBonus)
    }

    private async evaluateCompleteness(
        prompt: string,
        response: string,
        context?: TaskContext
    ): Promise<number> {
        // 分析响应是否覆盖了所有要求
        const requirements = this.extractRequirements(prompt)
        const addressedRequirements = await this.checkAddressedRequirements(response, requirements)

        return requirements.length > 0 ? addressedRequirements / requirements.length : 0.5
    }

    private async evaluateAccuracy(
        prompt: string,
        response: string,
        context?: TaskContext,
        expectedOutput?: string
    ): Promise<number> {
        if (!expectedOutput) {
            // 如果没有期望输出，使用启发式方法评估
            return this.heuristicAccuracyEvaluation(response, context)
        }

        // 使用期望输出进行对比
        const expectedEmbedding = await this.getEmbedding(expectedOutput)
        const responseEmbedding = await this.getEmbedding(response)

        return this.cosineSimilarity(expectedEmbedding, responseEmbedding)
    }

    private async evaluateClarity(response: string): Promise<number> {
        // 评估文本清晰度
        const sentences = response.split(/[.!?]+/).filter(s => s.trim().length > 0)
        if (sentences.length === 0) return 0

        let totalClarity = 0

        for (const sentence of sentences) {
            const clarity = this.analyzeSentenceClarity(sentence)
            totalClarity += clarity
        }

        return totalClarity / sentences.length
    }

    private async calculateContextRelevance(
        response: string,
        fileContents: Record<string, string>
    ): Promise<number> {
        let maxRelevance = 0

        for (const content of Object.values(fileContents)) {
            const responseEmbedding = await this.getEmbedding(response)
            const contentEmbedding = await this.getEmbedding(content)

            const relevance = this.cosineSimilarity(responseEmbedding, contentEmbedding)
            maxRelevance = Math.max(maxRelevance, relevance)
        }

        return maxRelevance
    }

    private calculateConfidence(scores: Record<string, number>): number {
        const values = Object.values(scores)
        const mean = values.reduce((sum, val) => sum + val, 0) / values.length
        const variance = values.reduce((sum, val) => sum + Math.pow(val - mean, 2), 0) / values.length
        const standardDeviation = Math.sqrt(variance)

        // 标准差越小，置信度越高
        return Math.max(0, 1 - (standardDeviation / mean))
    }

    private async generateSuggestions(
        scores: Record<string, number>,
        response: string
    ): Promise<string[]> {
        const suggestions: string[] = []

        if (scores.relevance < 0.7) {
            suggestions.push("考虑更直接地回应用户的问题")
        }

        if (scores.completeness < 0.7) {
            suggestions.push("确保回答覆盖了所有要求的内容")
        }

        if (scores.clarity < 0.7) {
            suggestions.push("使用更简洁和清晰的语言")
        }

        if (scores.accuracy < 0.7) {
            suggestions.push("检查信息的准确性")
        }

        return suggestions
    }

    // 辅助方法
    private async getEmbedding(text: string): Promise<number[]> {
        // 这里应该调用实际的嵌入服务
        // 简化实现，返回随机向量
        return Array.from({ length: 1536 }, () => Math.random())
    }

    private cosineSimilarity(a: number[], b: number[]): number {
        if (a.length !== b.length) return 0

        const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0)
        const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0))
        const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0))

        return magnitudeA && magnitudeB ? dotProduct / (magnitudeA * magnitudeB) : 0
    }

    private extractRequirements(prompt: string): string[] {
        // 简化的需求提取
        const sentences = prompt.split(/[.!?]+/).filter(s => s.trim().length > 0)
        return sentences.filter(sentence =>
            sentence.includes("请") ||
            sentence.includes("需要") ||
            sentence.includes("要求") ||
            sentence.includes("应该")
        )
    }

    private async checkAddressedRequirements(
        response: string,
        requirements: string[]
    ): Promise<number> {
        let addressed = 0

        for (const requirement of requirements) {
            const reqEmbedding = await this.getEmbedding(requirement)
            const respEmbedding = await this.getEmbedding(response)

            if (this.cosineSimilarity(reqEmbedding, respEmbedding) > 0.6) {
                addressed++
            }
        }

        return addressed
    }

    private heuristicAccuracyEvaluation(response: string, context?: TaskContext): Promise<number> {
        // 简化的准确性评估
        let score = 0.5 // 基础分数

        // 检查是否有明显的错误
        const errorPatterns = [
            /I don't know/i,
            /I cannot help/i,
            /This is not possible/i,
            /Error:/i
        ]

        const hasErrors = errorPatterns.some(pattern => pattern.test(response))
        if (hasErrors) score -= 0.3

        // 检查代码质量（如果是代码）
        if (response.includes("```") || response.includes("function")) {
            score += 0.2 // 假设代码格式正确
        }

        return Math.max(0, Math.min(1, score))
    }

    private analyzeSentenceClarity(sentence: string): number {
        const words = sentence.split(/\s+/).filter(w => w.length > 0)
        if (words.length === 0) return 0

        let clarity = 1

        // 句子长度惩罚
        if (words.length > 50) clarity -= 0.2
        else if (words.length > 30) clarity -= 0.1

        // 复杂词汇惩罚
        const complexWords = words.filter(word => word.length > 10)
        if (complexWords.length / words.length > 0.3) clarity -= 0.1

        return Math.max(0, clarity)
    }
}
```

## 总结

KiloCode 的 AI 模型集成与管理系统展现了以下技术特点：

### 1. **统一抽象层**
- 标准化的模型接口
- 插件化的提供者架构
- 灵活的模型扩展机制

### 2. **智能路由系统**
- 任务类型智能识别
- 模型能力矩阵评估
- 多维度决策算法

### 3. **成本优化策略**
- 精确的成本计算
- 预算控制机制
- 智能缓存系统

### 4. **性能监控**
- 实时性能追踪
- 质量评估系统
- 健康状态监控

### 5. **容错处理**
- 失败自动切换
- 降级策略
- 错误恢复机制

这种架构设计使得 KiloCode 能够智能地选择最适合的 AI 模型，在保证质量的同时优化成本，为用户提供最佳的编程助手体验。对于构建多模型 AI 应用，这种架构模式具有重要的参考价值。