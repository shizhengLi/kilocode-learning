# 性能优化与用户体验设计

> 深入分析 KiloCode 的性能瓶颈分析、响应速度优化和用户界面设计

## 性能优化概述

KiloCode 作为一款复杂的 VS Code AI 助手扩展，面临着多个性能挑战：多模型 AI 集成的延迟问题、大量并发请求的处理、复杂工作流的执行效率，以及 WebView UI 的响应速度。本章节将深入探讨 KiloCode 在性能优化方面的技术实践。

### 性能优化目标

1. **响应速度**：用户交互响应时间 < 100ms
2. **吞吐量**：支持 100+ 并发任务处理
3. **资源利用**：内存占用 < 200MB，CPU 使用率 < 30%
4. **稳定性**：99.9% 的任务成功率

## 性能瓶颈分析

### 1. AI 模型调用瓶颈

#### 网络延迟分析
```typescript
// src/performance/ai/ModelLatencyProfiler.ts
export class ModelLatencyProfiler {
    private latencyMetrics: Map<string, LatencyMetric[]> = new Map()
    private connectionPool: ConnectionPool

    async measureLatency(modelName: string): Promise<LatencyMetric> {
        const startTime = performance.now()

        try {
            // 模拟网络请求
            await this.connectionPool.execute(modelName, {
                type: "test",
                payload: { test: true }
            })

            const endTime = performance.now()
            const latency = endTime - startTime

            const metric: LatencyMetric = {
                model: modelName,
                latency,
                timestamp: new Date(),
                success: true
            }

            this.recordMetric(metric)
            return metric
        } catch (error) {
            const metric: LatencyMetric = {
                model: modelName,
                latency: performance.now() - startTime,
                timestamp: new Date(),
                success: false,
                error: error.message
            }

            this.recordMetric(metric)
            return metric
        }
    }

    private recordMetric(metric: LatencyMetric) {
        if (!this.latencyMetrics.has(metric.model)) {
            this.latencyMetrics.set(metric.model, [])
        }

        const metrics = this.latencyMetrics.get(metric.model)!
        metrics.push(metric)

        // 保留最近 1000 条记录
        if (metrics.length > 1000) {
            metrics.shift()
        }
    }

    getAverageLatency(modelName: string): number {
        const metrics = this.latencyMetrics.get(modelName) || []
        const successfulMetrics = metrics.filter(m => m.success)

        if (successfulMetrics.length === 0) {
            return Infinity
        }

        const totalLatency = successfulMetrics.reduce((sum, m) => sum + m.latency, 0)
        return totalLatency / successfulMetrics.length
    }

    getPerformanceReport(): PerformanceReport {
        const report: PerformanceReport = {
            timestamp: new Date(),
            models: {}
        }

        for (const [modelName, metrics] of this.latencyMetrics.entries()) {
            const successfulMetrics = metrics.filter(m => m.success)
            const failedMetrics = metrics.filter(m => !m.success)

            report.models[modelName] = {
                averageLatency: this.getAverageLatency(modelName),
                successRate: successfulMetrics.length / metrics.length,
                totalRequests: metrics.length,
                failureCount: failedMetrics.length,
                p95Latency: this.calculatePercentile(successfulMetrics, 95),
                p99Latency: this.calculatePercentile(successfulMetrics, 99)
            }
        }

        return report
    }

    private calculatePercentile(metrics: LatencyMetric[], percentile: number): number {
        if (metrics.length === 0) return 0

        const sortedMetrics = metrics.sort((a, b) => a.latency - b.latency)
        const index = Math.ceil((percentile / 100) * sortedMetrics.length) - 1

        return sortedMetrics[index]?.latency || 0
    }
}

interface LatencyMetric {
    model: string
    latency: number
    timestamp: Date
    success: boolean
    error?: string
}

interface PerformanceReport {
    timestamp: Date
    models: Record<string, ModelPerformance>
}

interface ModelPerformance {
    averageLatency: number
    successRate: number
    totalRequests: number
    failureCount: number
    p95Latency: number
    p99Latency: number
}
```

#### 模型响应时间优化
```typescript
// src/performance/ai/ModelResponseOptimizer.ts
export class ModelResponseOptimizer {
    private cache: LRUCache<string, ModelResponse>
    private prefetchQueue: PrefetchQueue
    private compressionEnabled: boolean = true

    constructor() {
        this.cache = new LRUCache({
            max: 1000,
            ttl: 1000 * 60 * 5 // 5分钟缓存
        })

        this.prefetchQueue = new PrefetchQueue({
            maxConcurrent: 3,
            retryAttempts: 2
        })
    }

    async optimizeResponse(
        request: ModelRequest,
        modelService: ModelService
    ): Promise<ModelResponse> {
        // 1. 缓存检查
        const cacheKey = this.generateCacheKey(request)
        const cachedResponse = this.cache.get(cacheKey)

        if (cachedResponse) {
            return cachedResponse
        }

        // 2. 请求压缩
        const optimizedRequest = this.compressRequest(request)

        // 3. 并行预取
        this.prefetchRelatedRequests(request, modelService)

        // 4. 执行请求
        const response = await modelService.execute(optimizedRequest)

        // 5. 响应优化
        const optimizedResponse = await this.optimizeResponseStructure(response)

        // 6. 缓存存储
        this.cache.set(cacheKey, optimizedResponse)

        return optimizedResponse
    }

    private compressRequest(request: ModelRequest): ModelRequest {
        if (!this.compressionEnabled) {
            return request
        }

        // 1. 上下文压缩
        const compressedContext = this.compressContext(request.context)

        // 2. 提示词优化
        const optimizedPrompt = this.optimizePrompt(request.prompt)

        // 3. 参数简化
        const optimizedParameters = this.simplifyParameters(request.parameters)

        return {
            ...request,
            context: compressedContext,
            prompt: optimizedPrompt,
            parameters: optimizedParameters
        }
    }

    private compressContext(context: ModelContext): ModelContext {
        // 移除不必要的元数据
        const filteredContext = {
            messages: context.messages.slice(-20), // 保留最近20条消息
            variables: context.variables,
            metadata: {
                ...context.metadata,
                // 只保留必要的元数据
                sessionId: context.metadata.sessionId,
                userId: context.metadata.userId
            }
        }

        // JSON 压缩
        return JSON.parse(JSON.stringify(filteredContext))
    }

    private optimizePrompt(prompt: string): string {
        // 移除多余的空格和换行
        let optimized = prompt.trim().replace(/\s+/g, ' ')

        // 如果提示词过长，进行截断
        if (optimized.length > 4000) {
            optimized = optimized.substring(0, 4000) + "..."
        }

        return optimized
    }

    private simplifyParameters(parameters: Record<string, any>): Record<string, any> {
        const simplified: Record<string, any> = {}

        for (const [key, value] of Object.entries(parameters)) {
            // 移除复杂的对象和数组
            if (typeof value === 'string' || typeof value === 'number' || typeof value === 'boolean') {
                simplified[key] = value
            }
        }

        return simplified
    }

    private async optimizeResponseStructure(response: ModelResponse): Promise<ModelResponse> {
        // 1. 流式响应处理
        if (response.stream) {
            return await this.processStreamResponse(response)
        }

        // 2. 结构化数据优化
        return this.optimizeStructuredResponse(response)
    }

    private async processStreamResponse(response: ModelResponse): Promise<ModelResponse> {
        const chunks: string[] = []

        for await (const chunk of response.stream) {
            chunks.push(chunk)

            // 实时反馈
            if (chunks.length % 10 === 0) {
                this.emitProgress(chunks.length, response.estimatedTotal)
            }
        }

        return {
            ...response,
            content: chunks.join(''),
            stream: null
        }
    }

    private optimizeStructuredResponse(response: ModelResponse): ModelResponse {
        // 移除重复内容
        const dedupedContent = this.deduplicateContent(response.content)

        // 格式化输出
        const formattedContent = this.formatOutput(dedupedContent)

        return {
            ...response,
            content: formattedContent
        }
    }

    private prefetchRelatedRequests(request: ModelRequest, modelService: ModelService) {
        // 基于当前请求预测可能的后续请求
        const relatedRequests = this.predictRelatedRequests(request)

        for (const relatedRequest of relatedRequests) {
            this.prefetchQueue.enqueue(async () => {
                const response = await modelService.execute(relatedRequest)
                const cacheKey = this.generateCacheKey(relatedRequest)
                this.cache.set(cacheKey, response)
            })
        }
    }

    private predictRelatedRequests(request: ModelRequest): ModelRequest[] {
        const relatedRequests: ModelRequest[] = []

        // 基于请求类型预测相关请求
        if (request.type === 'code-generation') {
            relatedRequests.push({
                ...request,
                type: 'code-review'
            })
        }

        if (request.type === 'explanation') {
            relatedRequests.push({
                ...request,
                type: 'example-generation'
            })
        }

        return relatedRequests
    }

    private generateCacheKey(request: ModelRequest): string {
        const key = {
            type: request.type,
            prompt: request.prompt,
            model: request.model,
            timestamp: Math.floor(Date.now() / 1000 / 60) // 按分钟缓存
        }

        return Buffer.from(JSON.stringify(key)).toString('base64')
    }
}
```

### 2. 内存管理优化

#### 内存泄漏检测
```typescript
// src/performance/memory/MemoryLeakDetector.ts
export class MemoryLeakDetector {
    private allocations: Map<string, AllocationInfo> = new Map()
    private thresholds: MemoryThresholds
    private gc: GarbageCollector

    constructor(thresholds: MemoryThresholds) {
        this.thresholds = thresholds
        this.gc = new GarbageCollector()

        // 定期检查内存使用情况
        setInterval(() => this.checkMemoryLeaks(), 5000)
    }

    trackAllocation(object: any, context: AllocationContext): void {
        const allocationId = this.generateAllocationId(object)
        const allocation: AllocationInfo = {
            id: allocationId,
            object,
            context,
            timestamp: Date.now(),
            size: this.calculateObjectSize(object),
            stackTrace: this.captureStackTrace()
        }

        this.allocations.set(allocationId, allocation)

        // 设置弱引用以便垃圾回收
        this.setupWeakReference(allocationId, object)
    }

    private checkMemoryLeaks(): void {
        const memoryUsage = process.memoryUsage()
        const { heapUsed, heapTotal, external, rss } = memoryUsage

        // 检查内存使用是否超过阈值
        if (heapUsed > this.thresholds.heapUsed) {
            this.handleMemoryOverflow(memoryUsage)
        }

        // 检查长期存在的对象
        const now = Date.now()
        const staleAllocations: AllocationInfo[] = []

        for (const allocation of this.allocations.values()) {
            if (now - allocation.timestamp > this.thresholds.maxAge) {
                staleAllocations.push(allocation)
            }
        }

        if (staleAllocations.length > this.thresholds.maxStaleObjects) {
            this.handleStaleObjects(staleAllocations)
        }
    }

    private handleMemoryOverflow(memoryUsage: NodeJS.MemoryUsage): void {
        // 1. 记录内存溢出事件
        this.logMemoryOverflow(memoryUsage)

        // 2. 触发垃圾回收
        this.gc.forceCollection()

        // 3. 清理缓存
        this.clearCaches()

        // 4. 释放大对象
        this.releaseLargeObjects()

        // 5. 检查是否需要警告用户
        if (memoryUsage.heapUsed > this.thresholds.criticalHeapUsed) {
            this.warnUser(memoryUsage)
        }
    }

    private handleStaleObjects(staleAllocations: AllocationInfo[]): void {
        // 分析长期存在的对象
        const analysis = this.analyzeStaleObjects(staleAllocations)

        // 记录潜在内存泄漏
        this.logPotentialLeaks(analysis)

        // 尝试清理可以安全释放的对象
        this.cleanupSafeObjects(staleAllocations)
    }

    private analyzeStaleObjects(staleAllocations: AllocationInfo[]): MemoryLeakAnalysis {
        const byType: Record<string, AllocationInfo[]> = {}
        const byContext: Record<string, AllocationInfo[]> = {}

        for (const allocation of staleAllocations) {
            const type = allocation.object.constructor.name
            if (!byType[type]) byType[type] = []
            byType[type].push(allocation)

            const context = allocation.context.component
            if (!byContext[context]) byContext[context] = []
            byContext[context].push(allocation)
        }

        return {
            totalCount: staleAllocations.length,
            byType,
            byContext,
            averageAge: staleAllocations.reduce((sum, a) => sum + (Date.now() - a.timestamp), 0) / staleAllocations.length,
            totalSize: staleAllocations.reduce((sum, a) => sum + a.size, 0)
        }
    }

    private clearCaches(): void {
        // 清理各种缓存
        this.clearLRUCaches()
        this.clearWeakMaps()
        this.clearEventListeners()
    }

    private releaseLargeObjects(): void {
        const largeObjects: AllocationInfo[] = []

        for (const allocation of this.allocations.values()) {
            if (allocation.size > this.thresholds.largeObjectSize) {
                largeObjects.push(allocation)
            }
        }

        // 按大小排序，优先释放最大的对象
        largeObjects.sort((a, b) => b.size - a.size)

        for (const allocation of largeObjects.slice(0, 10)) {
            this.allocations.delete(allocation.id)
            this.logObjectRelease(allocation)
        }
    }

    private calculateObjectSize(object: any): number {
        // 简化的对象大小计算
        const json = JSON.stringify(object)
        return json.length * 2 // 粗略估算
    }

    private generateAllocationId(object: any): string {
        return `${object.constructor.name}_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
    }

    private captureStackTrace(): string {
        const stack = new Error().stack
        return stack ? stack.split('\n').slice(3, 8).join('\n') : ''
    }

    private setupWeakReference(allocationId: string, object: any): void {
        // 使用 WeakRef 来跟踪对象而不阻止垃圾回收
        const weakRef = new WeakRef(object)

        // 定期检查对象是否已被回收
        setTimeout(() => {
            if (!weakRef.deref()) {
                this.allocations.delete(allocationId)
            }
        }, 60000) // 1分钟后检查
    }
}

interface AllocationInfo {
    id: string
    object: any
    context: AllocationContext
    timestamp: number
    size: number
    stackTrace: string
}

interface AllocationContext {
    component: string
    operation: string
    reason: string
}

interface MemoryThresholds {
    heapUsed: number
    criticalHeapUsed: number
    maxAge: number
    maxStaleObjects: number
    largeObjectSize: number
}

interface MemoryLeakAnalysis {
    totalCount: number
    byType: Record<string, AllocationInfo[]>
    byContext: Record<string, AllocationInfo[]>
    averageAge: number
    totalSize: number
}
```

### 3. 并发性能优化

#### 连接池管理
```typescript
// src/performance/network/ConnectionPool.ts
export class ConnectionPool {
    private pools: Map<string, ModelConnection[]> = new Map()
    private activeConnections: Map<string, number> = new Map()
    private config: ConnectionPoolConfig

    constructor(config: ConnectionPoolConfig) {
        this.config = config
        this.initializePools()
    }

    async acquireConnection(modelName: string): Promise<ModelConnection> {
        const pool = this.pools.get(modelName) || []

        // 1. 查找可用连接
        const availableConnection = pool.find(conn => conn.isAvailable())

        if (availableConnection) {
            availableConnection.acquire()
            return availableConnection
        }

        // 2. 检查是否可以创建新连接
        const activeCount = this.activeConnections.get(modelName) || 0

        if (activeCount < this.config.maxConnectionsPerModel) {
            const newConnection = await this.createConnection(modelName)
            newConnection.acquire()

            this.activeConnections.set(modelName, activeCount + 1)
            pool.push(newConnection)

            return newConnection
        }

        // 3. 等待连接释放
        return await this.waitForConnection(modelName)
    }

    releaseConnection(connection: ModelConnection): void {
        connection.release()

        // 如果连接过期，关闭它
        if (connection.isExpired()) {
            this.closeConnection(connection)
        }
    }

    private async waitForConnection(modelName: string): Promise<ModelConnection> {
        return new Promise((resolve, reject) => {
            const timeout = setTimeout(() => {
                reject(new Error(`Connection timeout for model: ${modelName}`))
            }, this.config.connectionTimeout)

            const checkInterval = setInterval(() => {
                const pool = this.pools.get(modelName) || []
                const availableConnection = pool.find(conn => conn.isAvailable())

                if (availableConnection) {
                    clearTimeout(timeout)
                    clearInterval(checkInterval)
                    availableConnection.acquire()
                    resolve(availableConnection)
                }
            }, 100)
        })
    }

    private async createConnection(modelName: string): Promise<ModelConnection> {
        const connection = new ModelConnection({
            modelName,
            timeout: this.config.connectionTimeout,
            keepAlive: this.config.keepAlive,
            maxRequests: this.config.maxRequestsPerConnection
        })

        await connection.connect()
        return connection
    }

    private closeConnection(connection: ModelConnection): void {
        connection.close()

        const pool = this.pools.get(connection.modelName) || []
        const index = pool.indexOf(connection)

        if (index > -1) {
            pool.splice(index, 1)
        }

        const activeCount = this.activeConnections.get(connection.modelName) || 0
        this.activeConnections.set(connection.modelName, Math.max(0, activeCount - 1))
    }

    private initializePools(): void {
        // 定期清理过期连接
        setInterval(() => this.cleanupExpiredConnections(), 30000)

        // 定期调整连接池大小
        setInterval(() => this.adjustPoolSize(), 60000)
    }

    private cleanupExpiredConnections(): void {
        for (const [modelName, pool] of this.pools.entries()) {
            const expiredConnections = pool.filter(conn => conn.isExpired())

            for (const connection of expiredConnections) {
                this.closeConnection(connection)
            }
        }
    }

    private adjustPoolSize(): void {
        for (const [modelName, pool] of this.pools.entries()) {
            const activeConnections = pool.filter(conn => !conn.isAvailable()).length
            const availableConnections = pool.filter(conn => conn.isAvailable()).length

            // 根据使用情况调整连接池大小
            if (availableConnections > this.config.maxIdleConnections) {
                // 关闭多余的空闲连接
                const connectionsToClose = availableConnections - this.config.maxIdleConnections
                const idleConnections = pool.filter(conn => conn.isAvailable())

                for (let i = 0; i < connectionsToClose && i < idleConnections.length; i++) {
                    this.closeConnection(idleConnections[i])
                }
            }
        }
    }

    getPoolStatus(): PoolStatus {
        const status: PoolStatus = {
            totalConnections: 0,
            activeConnections: 0,
            idleConnections: 0,
            byModel: {}
        }

        for (const [modelName, pool] of this.pools.entries()) {
            const activeCount = pool.filter(conn => !conn.isAvailable()).length
            const idleCount = pool.filter(conn => conn.isAvailable()).length

            status.totalConnections += pool.length
            status.activeConnections += activeCount
            status.idleConnections += idleCount

            status.byModel[modelName] = {
                total: pool.length,
                active: activeCount,
                idle: idleCount
            }
        }

        return status
    }
}

export class ModelConnection {
    public readonly modelName: string
    private acquired: boolean = false
    private createdAt: number = Date.now()
    private lastUsed: number = Date.now()
    private requestCount: number = 0
    private socket?: any

    constructor(private config: ModelConnectionConfig) {
        this.modelName = config.modelName
    }

    async connect(): Promise<void> {
        // 建立网络连接
        this.socket = await this.createSocket()
    }

    async execute(request: ModelRequest): Promise<ModelResponse> {
        if (!this.acquired) {
            throw new Error('Connection not acquired')
        }

        this.lastUsed = Date.now()
        this.requestCount++

        // 检查是否超过最大请求数
        if (this.requestCount >= this.config.maxRequests) {
            this.markForClose()
        }

        return await this.sendRequest(request)
    }

    isAvailable(): boolean {
        return !this.acquired && !this.isExpired()
    }

    isExpired(): boolean {
        const age = Date.now() - this.createdAt
        const idleTime = Date.now() - this.lastUsed

        return age > this.config.maxAge || idleTime > this.config.maxIdleTime
    }

    acquire(): void {
        this.acquired = true
    }

    release(): void {
        this.acquired = false
    }

    close(): void {
        if (this.socket) {
            this.socket.destroy()
            this.socket = undefined
        }
    }

    private markForClose(): void {
        // 标记连接为待关闭状态
        this.acquired = false
    }

    private async createSocket(): Promise<any> {
        // 创建网络套接字连接
        return new Promise((resolve, reject) => {
            const socket = require('net').createConnection({
                host: this.config.host,
                port: this.config.port
            })

            socket.on('connect', () => resolve(socket))
            socket.on('error', reject)
        })
    }

    private async sendRequest(request: ModelRequest): Promise<ModelResponse> {
        return new Promise((resolve, reject) => {
            const timeout = setTimeout(() => {
                reject(new Error('Request timeout'))
            }, this.config.timeout)

            this.socket.write(JSON.stringify(request))

            this.socket.once('data', (data: Buffer) => {
                clearTimeout(timeout)
                try {
                    const response = JSON.parse(data.toString())
                    resolve(response)
                } catch (error) {
                    reject(error)
                }
            })
        })
    }
}

interface ConnectionPoolConfig {
    maxConnectionsPerModel: number
    maxRequestsPerConnection: number
    maxIdleConnections: number
    connectionTimeout: number
    keepAlive: boolean
}

interface ModelConnectionConfig {
    modelName: string
    host: string
    port: number
    timeout: number
    keepAlive: boolean
    maxRequests: number
    maxAge: number
    maxIdleTime: number
}

interface PoolStatus {
    totalConnections: number
    activeConnections: number
    idleConnections: number
    byModel: Record<string, ModelPoolStatus>
}

interface ModelPoolStatus {
    total: number
    active: number
    idle: number
}
```

## 响应速度优化

### 1. 前端性能优化

#### 虚拟滚动优化
```typescript
// src/frontend/performance/VirtualScrollOptimizer.tsx
export function VirtualizedConversationList({ messages, onMessageClick }: VirtualizedConversationListProps) {
    const containerRef = useRef<HTMLDivElement>(null)
    const [visibleRange, setVisibleRange] = useState({ start: 0, end: 20 })
    const [scrollPosition, setScrollPosition] = useState(0)
    const itemHeight = 60 // 每个消息项的预估高度

    // 计算可见区域
    useEffect(() => {
        const container = containerRef.current
        if (!container) return

        const handleScroll = () => {
            const scrollTop = container.scrollTop
            const containerHeight = container.clientHeight

            const startIndex = Math.floor(scrollTop / itemHeight)
            const endIndex = Math.min(
                startIndex + Math.ceil(containerHeight / itemHeight) + 5,
                messages.length - 1
            )

            setScrollPosition(scrollTop)
            setVisibleRange({ start: startIndex, end: endIndex })
        }

        container.addEventListener('scroll', handleScroll)
        return () => container.removeEventListener('scroll', handleScroll)
    }, [messages.length])

    // 计算总高度
    const totalHeight = messages.length * itemHeight

    // 渲染可见项目
    const visibleMessages = messages.slice(visibleRange.start, visibleRange.end + 1)

    return (
        <div
            ref={containerRef}
            className="conversation-list-container"
            style={{ height: '600px', overflow: 'auto' }}
        >
            <div style={{ height: totalHeight, position: 'relative' }}>
                {visibleMessages.map((message, index) => {
                    const actualIndex = visibleRange.start + index
                    const top = actualIndex * itemHeight

                    return (
                        <div
                            key={message.id}
                            className="conversation-item"
                            style={{
                                position: 'absolute',
                                top: `${top}px`,
                                width: '100%',
                                height: `${itemHeight}px`
                            }}
                            onClick={() => onMessageClick(message)}
                        >
                            <MessageItem message={message} />
                        </div>
                    )
                })}
            </div>
        </div>
    )
}

// 优化的消息项组件
export function MessageItem({ message }: MessageItemProps) {
    const [isExpanded, setIsExpanded] = useState(false)
    const previewRef = useRef<HTMLDivElement>(null)

    // 使用 useMemo 优化内容计算
    const content = useMemo(() => {
        if (!isExpanded && message.content.length > 100) {
            return message.content.substring(0, 100) + '...'
        }
        return message.content
    }, [message.content, isExpanded])

    // 使用 useCallback 优化事件处理
    const handleClick = useCallback(() => {
        setIsExpanded(!isExpanded)
    }, [isExpanded])

    return (
        <div className="message-item" onClick={handleClick}>
            <div className="message-header">
                <span className="message-author">{message.author}</span>
                <span className="message-time">
                    {new Date(message.timestamp).toLocaleTimeString()}
                </span>
            </div>
            <div ref={previewRef} className="message-content">
                {content}
            </div>
            {isExpanded && (
                <div className="message-actions">
                    <button onClick={(e) => {
                        e.stopPropagation()
                        copyToClipboard(message.content)
                    }}>
                        复制
                    </button>
                </div>
            )}
        </div>
    )
}

// 代码高亮优化组件
export function CodeBlock({ code, language }: CodeBlockProps) {
    const [isCopied, setIsCopied] = useState(false)
    const [isHighlighted, setIsHighlighted] = useState(false)
    const codeRef = useRef<HTMLElement>(null)

    // 使用 requestAnimationFrame 优化高亮渲染
    useEffect(() => {
        if (!isHighlighted && codeRef.current) {
            requestAnimationFrame(() => {
                if (codeRef.current) {
                    // 执行代码高亮
                    highlightCode(codeRef.current, code, language)
                    setIsHighlighted(true)
                }
            })
        }
    }, [code, language, isHighlighted])

    const copyCode = useCallback(async () => {
        try {
            await navigator.clipboard.writeText(code)
            setIsCopied(true)
            setTimeout(() => setIsCopied(false), 2000)
        } catch (error) {
            console.error('复制失败:', error)
        }
    }, [code])

    return (
        <div className="code-block-container">
            <div className="code-header">
                <span className="code-language">{language}</span>
                <button
                    className={`copy-button ${isCopied ? 'copied' : ''}`}
                    onClick={copyCode}
                >
                    {isCopied ? '已复制' : '复制'}
                </button>
            </div>
            <pre className="code-content">
                <code ref={codeRef} className={`language-${language}`}>
                    {code}
                </code>
            </pre>
        </div>
    )
}
```

#### 图片懒加载优化
```typescript
// src/frontend/performance/ImageLazyLoader.tsx
export function LazyImage({ src, alt, placeholder, ...props }: LazyImageProps) {
    const [isLoaded, setIsLoaded] = useState(false)
    const [isInView, setIsInView] = useState(false)
    const [hasError, setHasError] = useState(false)
    const imgRef = useRef<HTMLImageElement>(null)

    // 使用 Intersection Observer 监听可见性
    useEffect(() => {
        const observer = new IntersectionObserver(
            ([entry]) => {
                if (entry.isIntersecting) {
                    setIsInView(true)
                    observer.disconnect()
                }
            },
            {
                rootMargin: '50px',
                threshold: 0.1
            }
        )

        if (imgRef.current) {
            observer.observe(imgRef.current)
        }

        return () => observer.disconnect()
    }, [])

    // 预加载图片
    useEffect(() => {
        if (isInView && !isLoaded && !hasError) {
            const img = new Image()

            img.onload = () => {
                setIsLoaded(true)
            }

            img.onerror = () => {
                setHasError(true)
            }

            img.src = src
        }
    }, [isInView, isLoaded, hasError, src])

    return (
        <div className="lazy-image-container" ref={imgRef}>
            {!isLoaded && !hasError && (
                <div className="image-placeholder">
                    {placeholder || <ImagePlaceholder />}
                </div>
            )}

            {hasError && (
                <div className="image-error">
                    <ImageErrorIcon />
                    <span>图片加载失败</span>
                </div>
            )}

            {isInView && (
                <img
                    src={src}
                    alt={alt}
                    className={`lazy-image ${isLoaded ? 'loaded' : ''}`}
                    style={{ opacity: isLoaded ? 1 : 0 }}
                    {...props}
                />
            )}
        </div>
    )
}

// 优化的 SVG 图标组件
export function OptimizedIcon({ name, size = 24, className = '' }: IconProps) {
    const [iconPath, setIconPath] = useState<string>('')
    const [isLoading, setIsLoading] = useState(true)

    useEffect(() => {
        // 从缓存或异步加载图标
        loadIcon(name).then(path => {
            setIconPath(path)
            setIsLoading(false)
        }).catch(() => {
            setIsLoading(false)
        })
    }, [name])

    if (isLoading) {
        return (
            <div
                className={`icon-loading ${className}`}
                style={{ width: size, height: size }}
            >
                <Spinner size={size * 0.6} />
            </div>
        )
    }

    if (!iconPath) {
        return (
            <div
                className={`icon-placeholder ${className}`}
                style={{ width: size, height: size }}
            >
                ?
            </div>
        )
    }

    return (
        <svg
            width={size}
            height={size}
            viewBox="0 0 24 24"
            className={`optimized-icon ${className}`}
            dangerouslySetInnerHTML={{ __html: iconPath }}
        />
    )
}
```

### 2. 网络请求优化

#### 请求去重和批处理
```typescript
// src/frontend/network/RequestOptimizer.ts
export class RequestOptimizer {
    private pendingRequests: Map<string, Promise<any>> = new Map()
    private batchQueue: BatchRequest[] = []
    private batchTimer: NodeJS.Timeout | null = null
    private cache: Map<string, CachedResponse> = new Map()

    constructor(private config: RequestOptimizerConfig) {
        this.initializeBatchProcessor()
    }

    async request<T>(url: string, options: RequestOptions = {}): Promise<T> {
        const cacheKey = this.generateCacheKey(url, options)

        // 1. 检查缓存
        const cachedResponse = this.cache.get(cacheKey)
        if (cachedResponse && !this.isCacheExpired(cachedResponse)) {
            return cachedResponse.data
        }

        // 2. 检查是否有相同的请求正在进行
        const pendingRequest = this.pendingRequests.get(cacheKey)
        if (pendingRequest) {
            return pendingRequest
        }

        // 3. 创建新请求
        const requestPromise = this.executeRequest<T>(url, options, cacheKey)
        this.pendingRequests.set(cacheKey, requestPromise)

        try {
            const result = await requestPromise
            return result
        } finally {
            this.pendingRequests.delete(cacheKey)
        }
    }

    private async executeRequest<T>(
        url: string,
        options: RequestOptions,
        cacheKey: string
    ): Promise<T> {
        try {
            const response = await fetch(url, options)
            const data = await response.json()

            // 缓存响应
            if (response.ok) {
                this.cache.set(cacheKey, {
                    data,
                    timestamp: Date.now(),
                    ttl: this.calculateTTL(response)
                })
            }

            return data
        } catch (error) {
            throw error
        }
    }

    addToBatch(request: BatchRequest): Promise<any> {
        return new Promise((resolve, reject) => {
            this.batchQueue.push({
                ...request,
                resolve,
                reject
            })

            if (!this.batchTimer) {
                this.batchTimer = setTimeout(() => {
                    this.processBatch()
                }, this.config.batchDelay)
            }
        })
    }

    private async processBatch(): Promise<void> {
        if (this.batchQueue.length === 0) return

        const batch = [...this.batchQueue]
        this.batchQueue = []
        this.batchTimer = null

        try {
            // 按URL分组批处理请求
            const groupedRequests = this.groupRequestsByType(batch)

            for (const [type, requests] of groupedRequests.entries()) {
                switch (type) {
                    case 'parallel':
                        await this.processParallelRequests(requests)
                        break
                    case 'sequential':
                        await this.processSequentialRequests(requests)
                        break
                    case 'merged':
                        await this.processMergedRequests(requests)
                        break
                }
            }
        } catch (error) {
            // 批处理失败时，回退到单个请求处理
            for (const request of batch) {
                try {
                    const result = await this.executeSingleRequest(request)
                    request.resolve(result)
                } catch (error) {
                    request.reject(error)
                }
            }
        }
    }

    private groupRequestsByType(requests: BatchRequest[]): Map<string, BatchRequest[]> {
        const groups = new Map<string, BatchRequest[]>()

        for (const request of requests) {
            let type = 'parallel'

            if (request.config?.batchType) {
                type = request.config.batchType
            } else if (request.method !== 'GET') {
                type = 'sequential'
            }

            if (!groups.has(type)) {
                groups.set(type, [])
            }

            groups.get(type)!.push(request)
        }

        return groups
    }

    private async processParallelRequests(requests: BatchRequest[]): Promise<void> {
        const promises = requests.map(async (request) => {
            try {
                const result = await this.executeSingleRequest(request)
                request.resolve(result)
            } catch (error) {
                request.reject(error)
            }
        })

        await Promise.all(promises)
    }

    private async processSequentialRequests(requests: BatchRequest[]): Promise<void> {
        for (const request of requests) {
            try {
                const result = await this.executeSingleRequest(request)
                request.resolve(result)
            } catch (error) {
                request.reject(error)
            }
        }
    }

    private async processMergedRequests(requests: BatchRequest[]): Promise<void> {
        // 合并相似的请求
        const mergedRequests = this.mergeSimilarRequests(requests)

        for (const mergedRequest of mergedRequests) {
            try {
                const result = await this.executeMergedRequest(mergedRequest)

                // 分发结果给原始请求
                for (const originalRequest of mergedRequest.originalRequests) {
                    originalRequest.resolve(this.extractRelevantData(result, originalRequest))
                }
            } catch (error) {
                for (const originalRequest of mergedRequest.originalRequests) {
                    originalRequest.reject(error)
                }
            }
        }
    }

    private mergeSimilarRequests(requests: BatchRequest[]): MergedRequest[] {
        const merged: MergedRequest[] = []
        const groups = new Map<string, BatchRequest[]>()

        // 按URL分组
        for (const request of requests) {
            const key = this.getMergingKey(request)
            if (!groups.has(key)) {
                groups.set(key, [])
            }
            groups.get(key)!.push(request)
        }

        // 合并每组请求
        for (const [key, groupRequests] of groups.entries()) {
            merged.push({
                url: key,
                method: 'GET',
                originalRequests: groupRequests,
                mergedParams: this.mergeParams(groupRequests)
            })
        }

        return merged
    }

    private getMergingKey(request: BatchRequest): string {
        return request.url.split('?')[0]
    }

    private mergeParams(requests: BatchRequest[]): Record<string, any> {
        const merged: Record<string, any> = {}

        for (const request of requests) {
            const params = new URLSearchParams(request.url.split('?')[1])
            for (const [key, value] of params.entries()) {
                merged[key] = value
            }
        }

        return merged
    }

    private async executeMergedRequest(mergedRequest: MergedRequest): Promise<any> {
        const queryString = new URLSearchParams(mergedRequest.mergedParams).toString()
        const url = `${mergedRequest.url}${queryString ? `?${queryString}` : ''}`

        const response = await fetch(url, { method: mergedRequest.method })
        return response.json()
    }

    private extractRelevantData(result: any, request: BatchRequest): any {
        // 根据原始请求从合并结果中提取相关数据
        return result
    }

    private async executeSingleRequest(request: BatchRequest): Promise<any> {
        const response = await fetch(request.url, {
            method: request.method,
            headers: request.headers,
            body: request.body
        })
        return response.json()
    }

    private generateCacheKey(url: string, options: RequestOptions): string {
        const key = {
            url,
            method: options.method || 'GET',
            headers: options.headers,
            body: options.body
        }
        return JSON.stringify(key)
    }

    private isCacheExpired(cachedResponse: CachedResponse): boolean {
        return Date.now() - cachedResponse.timestamp > cachedResponse.ttl
    }

    private calculateTTL(response: Response): number {
        const cacheControl = response.headers.get('Cache-Control')
        if (cacheControl) {
            const match = cacheControl.match(/max-age=(\d+)/)
            if (match) {
                return parseInt(match[1]) * 1000
            }
        }
        return this.config.defaultTTL
    }

    private initializeBatchProcessor(): void {
        // 定期清理缓存
        setInterval(() => {
            this.cleanupCache()
        }, this.config.cacheCleanupInterval)
    }

    private cleanupCache(): void {
        const now = Date.now()

        for (const [key, cachedResponse] of this.cache.entries()) {
            if (now - cachedResponse.timestamp > cachedResponse.ttl) {
                this.cache.delete(key)
            }
        }
    }
}

interface RequestOptimizerConfig {
    batchDelay: number
    defaultTTL: number
    cacheCleanupInterval: number
    maxBatchSize: number
}

interface BatchRequest {
    url: string
    method: string
    headers?: Record<string, string>
    body?: string
    config?: {
        batchType?: 'parallel' | 'sequential' | 'merged'
    }
    resolve: (value: any) => void
    reject: (error: any) => void
}

interface MergedRequest {
    url: string
    method: string
    originalRequests: BatchRequest[]
    mergedParams: Record<string, any>
}

interface CachedResponse {
    data: any
    timestamp: number
    ttl: number
}
```

## 用户界面设计

### 1. 交互设计优化

#### 智能提示系统
```typescript
// src/frontend/ui/SmartTooltip.tsx
export function SmartTooltip({
    children,
    content,
    position = 'top',
    trigger = 'hover',
    delay = 300
}: SmartTooltipProps) {
    const [isVisible, setIsVisible] = useState(false)
    const [showTimer, setShowTimer] = useState<NodeJS.Timeout | null>(null)
    const [hideTimer, setHideTimer] = useState<NodeJS.Timeout | null>(null)
    const tooltipRef = useRef<HTMLDivElement>(null)
    const triggerRef = useRef<HTMLDivElement>(null)

    const showTooltip = useCallback(() => {
        if (hideTimer) {
            clearTimeout(hideTimer)
            setHideTimer(null)
        }

        if (!showTimer) {
            const timer = setTimeout(() => {
                setIsVisible(true)
            }, delay)
            setShowTimer(timer)
        }
    }, [delay, hideTimer, showTimer])

    const hideTooltip = useCallback(() => {
        if (showTimer) {
            clearTimeout(showTimer)
            setShowTimer(null)
        }

        if (!hideTimer) {
            const timer = setTimeout(() => {
                setIsVisible(false)
            }, 100)
            setHideTimer(timer)
        }
    }, [showTimer, hideTimer])

    const handleClickOutside = useCallback((event: MouseEvent) => {
        if (
            tooltipRef.current &&
            !tooltipRef.current.contains(event.target as Node) &&
            triggerRef.current &&
            !triggerRef.current.contains(event.target as Node)
        ) {
            hideTooltip()
        }
    }, [hideTooltip])

    useEffect(() => {
        if (isVisible) {
            document.addEventListener('mousedown', handleClickOutside)
            return () => {
                document.removeEventListener('mousedown', handleClickOutside)
            }
        }
    }, [isVisible, handleClickOutside])

    const triggerProps = {
        ref: triggerRef,
        onMouseEnter: trigger === 'hover' ? showTooltip : undefined,
        onMouseLeave: trigger === 'hover' ? hideTooltip : undefined,
        onClick: trigger === 'click' ? showTooltip : undefined,
        onFocus: trigger === 'focus' ? showTooltip : undefined,
        onBlur: trigger === 'focus' ? hideTooltip : undefined,
    }

    return (
        <>
            <div {...triggerProps}>
                {children}
            </div>

            {isVisible && (
                <div
                    ref={tooltipRef}
                    className={`smart-tooltip smart-tooltip-${position}`}
                    role="tooltip"
                    aria-live="polite"
                >
                    <div className="tooltip-content">
                        {typeof content === 'string' ? (
                            <Markdown content={content} />
                        ) : (
                            content
                        )}
                    </div>
                    <div className="tooltip-arrow" />
                </div>
            )}
        </>
    )
}

// 优化的输入框组件
export function SmartInput({
    value,
    onChange,
    placeholder,
    suggestions = [],
    onSuggestionSelect,
    loading = false,
    error = null,
    ...props
}: SmartInputProps) {
    const [showSuggestions, setShowSuggestions] = useState(false)
    const [highlightedIndex, setHighlightedIndex] = useState(-1)
    const [inputValue, setInputValue] = useState(value)
    const inputRef = useRef<HTMLInputElement>(null)
    const suggestionsRef = useRef<HTMLDivElement>(null)

    const debouncedOnChange = useMemo(
        () => debounce(onChange, 300),
        [onChange]
    )

    const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        const newValue = e.target.value
        setInputValue(newValue)
        debouncedOnChange(newValue)

        if (newValue.length > 0) {
            setShowSuggestions(true)
        } else {
            setShowSuggestions(false)
        }
    }

    const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
        if (!showSuggestions || suggestions.length === 0) return

        switch (e.key) {
            case 'ArrowDown':
                e.preventDefault()
                setHighlightedIndex(prev =>
                    prev < suggestions.length - 1 ? prev + 1 : prev
                )
                break
            case 'ArrowUp':
                e.preventDefault()
                setHighlightedIndex(prev => prev > 0 ? prev - 1 : -1)
                break
            case 'Enter':
                e.preventDefault()
                if (highlightedIndex >= 0 && highlightedIndex < suggestions.length) {
                    selectSuggestion(suggestions[highlightedIndex])
                }
                break
            case 'Escape':
                setShowSuggestions(false)
                setHighlightedIndex(-1)
                break
        }
    }

    const selectSuggestion = (suggestion: Suggestion) => {
        setInputValue(suggestion.value)
        onChange(suggestion.value)
        setShowSuggestions(false)
        setHighlightedIndex(-1)
        onSuggestionSelect?.(suggestion)
    }

    const handleClickOutside = useCallback((event: MouseEvent) => {
        if (
            inputRef.current &&
            !inputRef.current.contains(event.target as Node) &&
            suggestionsRef.current &&
            !suggestionsRef.current.contains(event.target as Node)
        ) {
            setShowSuggestions(false)
            setHighlightedIndex(-1)
        }
    }, [])

    useEffect(() => {
        document.addEventListener('mousedown', handleClickOutside)
        return () => {
            document.removeEventListener('mousedown', handleClickOutside)
        }
    }, [handleClickOutside])

    useEffect(() => {
        setInputValue(value)
    }, [value])

    return (
        <div className="smart-input-container">
            <div className="input-wrapper">
                <input
                    ref={inputRef}
                    type="text"
                    value={inputValue}
                    onChange={handleInputChange}
                    onKeyDown={handleKeyDown}
                    placeholder={placeholder}
                    className={`smart-input ${error ? 'error' : ''} ${loading ? 'loading' : ''}`}
                    {...props}
                />

                {loading && (
                    <div className="input-loading">
                        <Spinner size={16} />
                    </div>
                )}

                {error && (
                    <div className="input-error">
                        <ErrorIcon size={16} />
                        <span>{error}</span>
                    </div>
                )}
            </div>

            {showSuggestions && suggestions.length > 0 && (
                <div ref={suggestionsRef} className="suggestions-list">
                    {suggestions.map((suggestion, index) => (
                        <div
                            key={suggestion.id}
                            className={`suggestion-item ${
                                index === highlightedIndex ? 'highlighted' : ''
                            }`}
                            onClick={() => selectSuggestion(suggestion)}
                        >
                            <div className="suggestion-content">
                                <span className="suggestion-value">
                                    {suggestion.value}
                                </span>
                                {suggestion.description && (
                                    <span className="suggestion-description">
                                        {suggestion.description}
                                    </span>
                                )}
                            </div>
                            {suggestion.type && (
                                <div className="suggestion-type">
                                    {suggestion.type}
                                </div>
                            )}
                        </div>
                    ))}
                </div>
            )}
        </div>
    )
}

// 键盘快捷键系统
export function KeyboardShortcutProvider({ children }: { children: React.ReactNode }) {
    const [shortcuts, setShortcuts] = useState<KeyboardShortcut[]>([])
    const [helpVisible, setHelpVisible] = useState(false)

    const registerShortcut = useCallback((shortcut: KeyboardShortcut) => {
        setShortcuts(prev => [...prev, shortcut])
    }, [])

    const unregisterShortcut = useCallback((id: string) => {
        setShortcuts(prev => prev.filter(s => s.id !== id))
    }, [])

    const handleKeyDown = useCallback((event: KeyboardEvent) => {
        // 忽略在输入框中的快捷键
        if (
            event.target instanceof HTMLInputElement ||
            event.target instanceof HTMLTextAreaElement ||
            event.target instanceof HTMLSelectElement
        ) {
            return
        }

        for (const shortcut of shortcuts) {
            if (isShortcutMatch(event, shortcut)) {
                event.preventDefault()
                shortcut.callback()
                break
            }
        }
    }, [shortcuts])

    useEffect(() => {
        document.addEventListener('keydown', handleKeyDown)
        return () => {
            document.removeEventListener('keydown', handleKeyDown)
        }
    }, [handleKeyDown])

    const contextValue = useMemo(() => ({
        registerShortcut,
        unregisterShortcut,
        showHelp: () => setHelpVisible(true),
        hideHelp: () => setHelpVisible(false)
    }), [registerShortcut, unregisterShortcut])

    return (
        <KeyboardShortcutContext.Provider value={contextValue}>
            {children}

            {helpVisible && (
                <ShortcutHelpModal
                    shortcuts={shortcuts}
                    onClose={() => setHelpVisible(false)}
                />
            )}
        </KeyboardShortcutContext.Provider>
    )
}

function isShortcutMatch(event: KeyboardEvent, shortcut: KeyboardShortcut): boolean {
    const modifiers = {
        ctrl: event.ctrlKey,
        alt: event.altKey,
        shift: event.shiftKey,
        meta: event.metaKey
    }

    const shortcutModifiers = {
        ctrl: shortcut.key.includes('Ctrl+'),
        alt: shortcut.key.includes('Alt+'),
        shift: shortcut.key.includes('Shift+'),
        meta: shortcut.key.includes('Cmd+') || shortcut.key.includes('Meta+')
    }

    const key = shortcut.key
        .replace('Ctrl+', '')
        .replace('Alt+', '')
        .replace('Shift+', '')
        .replace('Cmd+', '')
        .replace('Meta+', '')

    return (
        event.key.toLowerCase() === key.toLowerCase() &&
        modifiers.ctrl === shortcutModifiers.ctrl &&
        modifiers.alt === shortcutModifiers.alt &&
        modifiers.shift === shortcutModifiers.shift &&
        modifiers.meta === shortcutModifiers.meta
    )
}
```

### 2. 主题与样式系统

#### CSS-in-JS 优化
```typescript
// src/frontend/styles/ThemeProvider.tsx
export function ThemeProvider({ children, theme = 'light' }: ThemeProviderProps) {
    const [currentTheme, setCurrentTheme] = useState<Theme>(themes[theme])
    const [customThemes, setCustomThemes] = useState<CustomTheme[]>([])

    // 应用主题到文档根元素
    useEffect(() => {
        const root = document.documentElement
        Object.entries(currentTheme.colors).forEach(([key, value]) => {
            root.style.setProperty(`--color-${key}`, value)
        })

        Object.entries(currentTheme.spacing).forEach(([key, value]) => {
            root.style.setProperty(`--spacing-${key}`, value)
        })

        Object.entries(currentTheme.typography).forEach(([key, value]) => {
            root.style.setProperty(`--font-${key}`, value.fontFamily)
            root.style.setProperty(`--font-size-${key}`, value.fontSize)
            root.style.setProperty(`--font-weight-${key}`, value.fontWeight)
            root.style.setProperty(`--line-height-${key}`, value.lineHeight)
        })
    }, [currentTheme])

    const contextValue = useMemo(() => ({
        theme: currentTheme,
        setTheme: (newTheme: string) => {
            setCurrentTheme(themes[newTheme] || themes.light)
        },
        addCustomTheme: (theme: CustomTheme) => {
            setCustomThemes(prev => [...prev, theme])
        },
        removeCustomTheme: (id: string) => {
            setCustomThemes(prev => prev.filter(t => t.id !== id))
        },
        getColor: (colorName: string) => currentTheme.colors[colorName] || colorName,
        getSpacing: (spacingName: string) => currentTheme.spacing[spacingName] || spacingName,
        getTypography: (typographyName: string) => currentTheme.typography[typographyName]
    }), [currentTheme])

    return (
        <ThemeContext.Provider value={contextValue}>
            {children}
        </ThemeContext.Provider>
    )
}

// 优化的样式函数
export const styled = <P extends object>(
    component: React.ComponentType<P>
) => {
    return (styles: TemplateStringsArray | StyleFunction<P>) => {
        const StyledComponent = React.forwardRef<any, P>((props, ref) => {
            const { theme } = useTheme()

            const className = useMemo(() => {
                let computedStyles: React.CSSProperties

                if (typeof styles === 'function') {
                    computedStyles = styles({ ...props, theme })
                } else {
                    // 处理模板字符串样式
                    computedStyles = parseTemplateStyles(styles, props, theme)
                }

                return generateClassName(computedStyles)
            }, [props, theme, styles])

            return React.createElement(component, {
                ...props,
                ref,
                className: `${props.className || ''} ${className}`
            })
        })

        StyledComponent.displayName = `Styled(${getDisplayName(component)})`
        return StyledComponent
    }
}

// 响应式设计系统
export function useResponsive() {
    const [breakpoint, setBreakpoint] = useState<Breakpoint>('desktop')
    const [screenSize, setScreenSize] = useState({
        width: typeof window !== 'undefined' ? window.innerWidth : 0,
        height: typeof window !== 'undefined' ? window.innerHeight : 0
    })

    useEffect(() => {
        const handleResize = () => {
            const width = window.innerWidth
            const height = window.innerHeight

            setScreenSize({ width, height })

            // 确定断点
            if (width < 640) {
                setBreakpoint('mobile')
            } else if (width < 1024) {
                setBreakpoint('tablet')
            } else {
                setBreakpoint('desktop')
            }
        }

        handleResize()
        window.addEventListener('resize', handleResize)

        return () => window.removeEventListener('resize', handleResize)
    }, [])

    return {
        breakpoint,
        screenSize,
        isMobile: breakpoint === 'mobile',
        isTablet: breakpoint === 'tablet',
        isDesktop: breakpoint === 'desktop',
        isSmallScreen: screenSize.width < 640,
        isMediumScreen: screenSize.width >= 640 && screenSize.width < 1024,
        isLargeScreen: screenSize.width >= 1024
    }
}

// 动画优化
export function useAnimation(
    animation: AnimationConfig,
    deps: any[] = []
): { ref: React.RefObject<any>; isAnimating: boolean } {
    const ref = useRef<any>(null)
    const [isAnimating, setIsAnimating] = useState(false)
    const animationRef = useRef<Animation | null>(null)

    useEffect(() => {
        if (!ref.current) return

        const element = ref.current

        // 清理之前的动画
        if (animationRef.current) {
            animationRef.current.cancel()
        }

        // 创建新动画
        const animationInstance = element.animate(
            animation.keyframes,
            animation.options
        )

        animationRef.current = animationInstance

        const handleStart = () => setIsAnimating(true)
        const handleFinish = () => setIsAnimating(false)
        const handleCancel = () => setIsAnimating(false)

        animationInstance.addEventListener('begin', handleStart)
        animationInstance.addEventListener('finish', handleFinish)
        animationInstance.addEventListener('cancel', handleCancel)

        return () => {
            animationInstance.removeEventListener('begin', handleStart)
            animationInstance.removeEventListener('finish', handleFinish)
            animationInstance.removeEventListener('cancel', handleCancel)
            animationInstance.cancel()
        }
    }, deps)

    return { ref, isAnimating }
}

// 优化的拖拽系统
export function useDragDrop<T = any>(
    options: DragDropOptions<T> = {}
): {
    isDragging: boolean
    dragProps: React.HTMLAttributes<any>
    dropProps: React.HTMLAttributes<any>
} {
    const [isDragging, setIsDragging] = useState(false)
    const dragDataRef = useRef<T | null>(null)

    const handleDragStart = useCallback((e: React.DragEvent, data: T) => {
        setIsDragging(true)
        dragDataRef.current = data

        if (options.dragImage) {
            const img = new Image()
            img.src = options.dragImage
            e.dataTransfer.setDragImage(img, 0, 0)
        }

        e.dataTransfer.effectAllowed = options.effectAllowed || 'move'

        if (options.onDragStart) {
            options.onDragStart(e, data)
        }
    }, [options])

    const handleDragOver = useCallback((e: React.DragEvent) => {
        e.preventDefault()
        e.dataTransfer.dropEffect = options.dropEffect || 'move'

        if (options.onDragOver) {
            options.onDragOver(e)
        }
    }, [options])

    const handleDrop = useCallback((e: React.DragEvent) => {
        e.preventDefault()
        setIsDragging(false)

        if (dragDataRef.current && options.onDrop) {
            options.onDrop(e, dragDataRef.current)
        }

        dragDataRef.current = null
    }, [options])

    const handleDragEnd = useCallback((e: React.DragEvent) => {
        setIsDragging(false)
        dragDataRef.current = null

        if (options.onDragEnd) {
            options.onDragEnd(e)
        }
    }, [options])

    const dragProps = {
        draggable: true,
        onDragStart: (e: React.DragEvent) => handleDragStart(e, options.data!),
        onDragEnd: handleDragEnd
    }

    const dropProps = {
        onDragOver: handleDragOver,
        onDrop: handleDrop
    }

    return { isDragging, dragProps, dropProps }
}
```

### 3. 用户体验优化

#### 加载状态管理
```typescript
// src/frontend/ui/LoadingStates.tsx
export function LoadingManager({ children }: { children: React.ReactNode }) {
    const [loadingStates, setLoadingStates] = useState<Map<string, LoadingState>>(new Map())

    const startLoading = useCallback((id: string, options: LoadingOptions = {}) => {
        setLoadingStates(prev => {
            const newState = new Map(prev)
            newState.set(id, {
                id,
                status: 'loading',
                startTime: Date.now(),
                message: options.message || 'Loading...',
                type: options.type || 'spinner',
                progress: options.progress || 0
            })
            return newState
        })
    }, [])

    const updateLoading = useCallback((id: string, updates: Partial<LoadingState>) => {
        setLoadingStates(prev => {
            const newState = new Map(prev)
            const currentState = newState.get(id)

            if (currentState) {
                newState.set(id, { ...currentState, ...updates })
            }

            return newState
        })
    }, [])

    const completeLoading = useCallback((id: string, success: boolean = true) => {
        setLoadingStates(prev => {
            const newState = new Map(prev)
            const currentState = newState.get(id)

            if (currentState) {
                newState.set(id, {
                    ...currentState,
                    status: success ? 'completed' : 'error',
                    endTime: Date.now(),
                    duration: Date.now() - currentState.startTime
                })

                // 3秒后移除加载状态
                setTimeout(() => {
                    setLoadingStates(prev => {
                        const newState = new Map(prev)
                        newState.delete(id)
                        return newState
                    })
                }, 3000)
            }

            return newState
        })
    }, [])

    const cancelLoading = useCallback((id: string) => {
        setLoadingStates(prev => {
            const newState = new Map(prev)
            newState.delete(id)
            return newState
        })
    }, [])

    const contextValue = useMemo(() => ({
        loadingStates: Array.from(loadingStates.values()),
        startLoading,
        updateLoading,
        completeLoading,
        cancelLoading
    }), [loadingStates, startLoading, updateLoading, completeLoading, cancelLoading])

    return (
        <LoadingContext.Provider value={contextValue}>
            {children}
        </LoadingContext.Provider>
    )
}

// 优化的骨架屏组件
export function SkeletonLoader({
    type = 'text',
    lines = 3,
    width = '100%',
    height = '20px'
}: SkeletonLoaderProps) {
    return (
        <div className="skeleton-loader">
            {type === 'text' && (
                <>
                    {Array.from({ length: lines }).map((_, index) => (
                        <div
                            key={index}
                            className="skeleton-line"
                            style={{
                                width: index === lines - 1 ? '60%' : width,
                                height,
                                animationDelay: `${index * 0.1}s`
                            }}
                        />
                    ))}
                </>
            )}

            {type === 'card' && (
                <div className="skeleton-card">
                    <div className="skeleton-avatar" />
                    <div className="skeleton-content">
                        <div className="skeleton-line" style={{ width: '80%' }} />
                        <div className="skeleton-line" style={{ width: '60%' }} />
                        <div className="skeleton-line" style={{ width: '40%' }} />
                    </div>
                </div>
            )}

            {type === 'table' && (
                <div className="skeleton-table">
                    {Array.from({ length: 5 }).map((_, rowIndex) => (
                        <div key={rowIndex} className="skeleton-table-row">
                            {Array.from({ length: 4 }).map((_, colIndex) => (
                                <div
                                    key={colIndex}
                                    className="skeleton-table-cell"
                                    style={{
                                        width: `${Math.random() * 40 + 60}%`,
                                        animationDelay: `${(rowIndex * 4 + colIndex) * 0.05}s`
                                    }}
                                />
                            ))}
                        </div>
                    ))}
                </div>
            )}
        </div>
    )
}

// 智能错误边界
export class SmartErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
    constructor(props: ErrorBoundaryProps) {
        super(props)
        this.state = { hasError: false, error: null, errorInfo: null }
    }

    static getDerivedStateFromError(error: Error): ErrorBoundaryState {
        return { hasError: true, error, errorInfo: null }
    }

    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
        this.setState({ error, errorInfo })

        // 发送错误报告
        if (this.props.onError) {
            this.props.onError(error, errorInfo)
        }
    }

    render() {
        if (this.state.hasError) {
            return (
                <div className="error-boundary">
                    <div className="error-content">
                        <h2>出现了一个错误</h2>
                        <p>我们遇到了一个意外错误，请稍后再试。</p>

                        {this.props.fallback ? (
                            this.props.fallback(this.state.error!)
                        ) : (
                            <div className="error-actions">
                                <button
                                    onClick={() => this.setState({ hasError: false })}
                                    className="retry-button"
                                >
                                    重试
                                </button>
                                <button
                                    onClick={() => window.location.reload()}
                                    className="refresh-button"
                                >
                                    刷新页面
                                </button>
                            </div>
                        )}

                        {process.env.NODE_ENV === 'development' && this.state.error && (
                            <details className="error-details">
                                <summary>错误详情</summary>
                                <pre>{this.state.error.toString()}</pre>
                                <pre>{this.state.errorInfo?.componentStack}</pre>
                            </details>
                        )}
                    </div>
                </div>
            )
        }

        return this.props.children
    }
}
```

## 性能监控与调优

### 1. 性能指标收集

```typescript
// src/performance/monitoring/PerformanceMonitor.ts
export class PerformanceMonitor {
    private metrics: Map<string, PerformanceMetric[]> = new Map()
    private observers: PerformanceObserver[] = []
    private config: PerformanceMonitorConfig

    constructor(config: PerformanceMonitorConfig) {
        this.config = config
        this.initializeObservers()
    }

    private initializeObservers(): void {
        // 1. 导航性能观察者
        const navigationObserver = new PerformanceObserver((list) => {
            for (const entry of list.getEntries()) {
                if (entry instanceof PerformanceNavigationTiming) {
                    this.recordMetric('navigation', {
                        name: 'page-load',
                        value: entry.loadEventEnd - entry.loadEventStart,
                        timestamp: entry.loadEventEnd,
                        metadata: {
                            domComplete: entry.domComplete,
                            domInteractive: entry.domInteractive,
                            firstPaint: entry.responseStart,
                            resources: entry.transferSize
                        }
                    })
                }
            }
        })

        navigationObserver.observe({ entryTypes: ['navigation'] })
        this.observers.push(navigationObserver)

        // 2. 资源加载观察者
        const resourceObserver = new PerformanceObserver((list) => {
            for (const entry of list.getEntries()) {
                if (entry instanceof PerformanceResourceTiming) {
                    this.recordMetric('resources', {
                        name: entry.name,
                        value: entry.duration,
                        timestamp: entry.startTime,
                        metadata: {
                            size: entry.transferSize,
                            type: entry.initiatorType,
                            cached: entry.transferSize === 0
                        }
                    })
                }
            }
        })

        resourceObserver.observe({ entryTypes: ['resource'] })
        this.observers.push(resourceObserver)

        // 3. 长任务观察者
        const longTaskObserver = new PerformanceObserver((list) => {
            for (const entry of list.getEntries()) {
                if (entry instanceof PerformanceLongTaskTiming) {
                    this.recordMetric('long-tasks', {
                        name: 'long-task',
                        value: entry.duration,
                        timestamp: entry.startTime,
                        metadata: {
                            type: entry.name,
                            source: entry.attribution
                        }
                    })
                }
            }
        })

        longTaskObserver.observe({ entryTypes: ['longtask'] })
        this.observers.push(longTaskObserver)

        // 4. 用户交互观察者
        if ('EventTiming' in window) {
            const eventTimingObserver = new PerformanceObserver((list) => {
                for (const entry of list.getEntries()) {
                    if (entry instanceof PerformanceEventTiming) {
                        this.recordMetric('interaction', {
                            name: entry.name,
                            value: entry.duration,
                            timestamp: entry.startTime,
                            metadata: {
                                processingStart: entry.processingStart,
                                processingEnd: entry.processingEnd,
                                target: entry.target?.tagName
                            }
                        })
                    }
                }
            })

            eventTimingObserver.observe({ entryTypes: ['first-input', 'event'] })
            this.observers.push(eventTimingObserver)
        }
    }

    recordMetric(category: string, metric: PerformanceMetric): void {
        if (!this.metrics.has(category)) {
            this.metrics.set(category, [])
        }

        const metrics = this.metrics.get(category)!
        metrics.push(metric)

        // 限制指标数量
        if (metrics.length > this.config.maxMetricsPerCategory) {
            metrics.shift()
        }

        // 触发告警
        this.checkThresholds(category, metric)
    }

    private checkThresholds(category: string, metric: PerformanceMetric): void {
        const thresholds = this.config.thresholds[category]

        if (!thresholds) return

        if (metric.value > thresholds.warning) {
            this.warn(category, metric, thresholds.warning)
        }

        if (metric.value > thresholds.critical) {
            this.critical(category, metric, thresholds.critical)
        }
    }

    private warn(category: string, metric: PerformanceMetric, threshold: number): void {
        console.warn(`Performance warning: ${category} - ${metric.name} = ${metric.value}ms (> ${threshold}ms)`)

        // 发送告警
        if (this.config.onWarning) {
            this.config.onWarning(category, metric, threshold)
        }
    }

    private critical(category: string, metric: PerformanceMetric, threshold: number): void {
        console.error(`Performance critical: ${category} - ${metric.name} = ${metric.value}ms (> ${threshold}ms)`)

        // 发送严重告警
        if (this.config.onCritical) {
            this.config.onCritical(category, metric, threshold)
        }
    }

    getPerformanceReport(): PerformanceReport {
        const report: PerformanceReport = {
            timestamp: new Date(),
            categories: {}
        }

        for (const [category, metrics] of this.metrics.entries()) {
            report.categories[category] = {
                count: metrics.length,
                average: metrics.reduce((sum, m) => sum + m.value, 0) / metrics.length,
                min: Math.min(...metrics.map(m => m.value)),
                max: Math.max(...metrics.map(m => m.value)),
                p95: this.calculatePercentile(metrics, 95),
                p99: this.calculatePercentile(metrics, 99),
                recent: metrics.slice(-10)
            }
        }

        return report
    }

    private calculatePercentile(metrics: PerformanceMetric[], percentile: number): number {
        if (metrics.length === 0) return 0

        const sortedMetrics = metrics.sort((a, b) => a.value - b.value)
        const index = Math.ceil((percentile / 100) * sortedMetrics.length) - 1

        return sortedMetrics[index]?.value || 0
    }

    startMeasurement(name: string): () => void {
        const startTime = performance.now()

        return () => {
            const duration = performance.now() - startTime
            this.recordMetric('custom', {
                name,
                value: duration,
                timestamp: performance.now()
            })
        }
    }

    cleanup(): void {
        this.observers.forEach(observer => observer.disconnect())
        this.observers = []
        this.metrics.clear()
    }
}

interface PerformanceMonitorConfig {
    maxMetricsPerCategory: number
    thresholds: Record<string, { warning: number; critical: number }>
    onWarning?: (category: string, metric: PerformanceMetric, threshold: number) => void
    onCritical?: (category: string, metric: PerformanceMetric, threshold: number) => void
}

interface PerformanceMetric {
    name: string
    value: number
    timestamp: number
    metadata?: Record<string, any>
}

interface PerformanceReport {
    timestamp: Date
    categories: Record<string, CategoryReport>
}

interface CategoryReport {
    count: number
    average: number
    min: number
    max: number
    p95: number
    p99: number
    recent: PerformanceMetric[]
}
```

### 2. 自动化性能调优

```typescript
// src/performance/auto-tuner/AutoPerformanceTuner.ts
export class AutoPerformanceTuner {
    private monitor: PerformanceMonitor
    private tuner: PerformanceTuner
    private config: AutoTunerConfig

    constructor(config: AutoTunerConfig) {
        this.config = config
        this.monitor = new PerformanceMonitor(config.monitorConfig)
        this.tuner = new PerformanceTuner(config.tunerConfig)

        this.initializeAutoTuning()
    }

    private initializeAutoTuning(): void {
        // 定期检查性能并自动调优
        setInterval(() => {
            this.autoTune()
        }, this.config.tuningInterval)
    }

    private async autoTune(): Promise<void> {
        const report = this.monitor.getPerformanceReport()

        for (const [category, categoryReport] of Object.entries(report.categories)) {
            // 检查是否需要调优
            if (this.needsTuning(categoryReport)) {
                await this.tuneCategory(category, categoryReport)
            }
        }
    }

    private needsTuning(categoryReport: CategoryReport): boolean {
        const thresholds = this.config.tuningThresholds[categoryReport.count > 0 ? 'active' : 'idle']

        return (
            categoryReport.average > thresholds.average ||
            categoryReport.p95 > thresholds.p95 ||
            categoryReport.p99 > thresholds.p99
        )
    }

    private async tuneCategory(category: string, report: CategoryReport): Promise<void> {
        const tuningStrategies = this.config.tuningStrategies[category]

        if (!tuningStrategies) return

        for (const strategy of tuningStrategies) {
            try {
                const result = await this.executeTuningStrategy(strategy, report)

                if (result.success) {
                    console.log(`Applied tuning strategy: ${strategy.name} for ${category}`)
                    break
                }
            } catch (error) {
                console.error(`Failed to apply tuning strategy: ${strategy.name}`, error)
            }
        }
    }

    private async executeTuningStrategy(
        strategy: TuningStrategy,
        report: CategoryReport
    ): Promise<TuningResult> {
        switch (strategy.type) {
            case 'cache':
                return await this.tuneCache(strategy, report)
            case 'connection':
                return await this.tuneConnections(strategy, report)
            case 'thread':
                return await this.tuneThreads(strategy, report)
            case 'memory':
                return await this.tuneMemory(strategy, report)
            case 'algorithm':
                return await this.tuneAlgorithm(strategy, report)
            default:
                return { success: false, reason: 'Unknown strategy type' }
        }
    }

    private async tuneCache(strategy: TuningStrategy, report: CategoryReport): Promise<TuningResult> {
        const cacheConfig = strategy.config as CacheTuningConfig

        // 根据性能指标调整缓存
        if (report.average > cacheConfig.threshold) {
            await this.tuner.adjustCache({
                size: Math.min(cacheConfig.maxSize, cacheConfig.size * 1.5),
                ttl: Math.max(cacheConfig.minTTL, cacheConfig.ttl * 0.8),
                strategy: cacheConfig.strategy
            })

            return { success: true }
        }

        return { success: false, reason: 'No tuning needed' }
    }

    private async tuneConnections(strategy: TuningStrategy, report: CategoryReport): Promise<TuningResult> {
        const connectionConfig = strategy.config as ConnectionTuningConfig

        if (report.average > connectionConfig.threshold) {
            await this.tuner.adjustConnections({
                maxConnections: Math.min(connectionConfig.maxConnections, connectionConfig.maxConnections * 1.2),
                keepAlive: true,
                timeout: Math.max(connectionConfig.minTimeout, connectionConfig.timeout * 0.9)
            })

            return { success: true }
        }

        return { success: false, reason: 'No tuning needed' }
    }

    private async tuneThreads(strategy: TuningStrategy, report: CategoryReport): Promise<TuningResult> {
        const threadConfig = strategy.config as ThreadTuningConfig

        if (report.p95 > threadConfig.threshold) {
            await this.tuner.adjustThreads({
                workerCount: Math.min(threadConfig.maxWorkers, threadConfig.workerCount + 1),
                taskQueueSize: threadConfig.queueSize * 1.2,
                priority: threadConfig.priority
            })

            return { success: true }
        }

        return { success: false, reason: 'No tuning needed' }
    }

    private async tuneMemory(strategy: TuningStrategy, report: CategoryReport): Promise<TuningResult> {
        const memoryConfig = strategy.config as MemoryTuningConfig

        const memoryUsage = process.memoryUsage()

        if (memoryUsage.heapUsed > memoryConfig.threshold) {
            await this.tuner.adjustMemory({
                gcThreshold: memoryUsage.heapUsed * 0.8,
                maxHeapSize: memoryConfig.maxHeapSize,
                gcStrategy: memoryConfig.gcStrategy
            })

            return { success: true }
        }

        return { success: false, reason: 'No tuning needed' }
    }

    private async tuneAlgorithm(strategy: TuningStrategy, report: CategoryReport): Promise<TuningResult> {
        const algorithmConfig = strategy.config as AlgorithmTuningConfig

        if (report.average > algorithmConfig.threshold) {
            await this.tuner.adjustAlgorithm({
                complexity: algorithmConfig.complexity,
                batchSize: algorithmConfig.batchSize * 1.2,
                parallelism: Math.min(algorithmConfig.maxParallelism, algorithmConfig.parallelism + 1)
            })

            return { success: true }
        }

        return { success: false, reason: 'No tuning needed' }
    }
}

interface AutoTunerConfig {
    tuningInterval: number
    monitorConfig: PerformanceMonitorConfig
    tunerConfig: PerformanceTunerConfig
    tuningThresholds: Record<string, PerformanceThresholds>
    tuningStrategies: Record<string, TuningStrategy[]>
}

interface PerformanceThresholds {
    average: number
    p95: number
    p99: number
}

interface TuningStrategy {
    name: string
    type: 'cache' | 'connection' | 'thread' | 'memory' | 'algorithm'
    config: any
    priority: number
}

interface TuningResult {
    success: boolean
    reason?: string
}
```

## 总结

KiloCode 的性能优化与用户体验设计展现了以下特点：

### 1. **性能优化策略**
- **多层次优化**：从 AI 模型调用到前端渲染的全方位性能优化
- **智能缓存**：多级缓存策略和智能预取机制
- **连接池管理**：高效的连接复用和动态调整
- **内存管理**：自动内存泄漏检测和垃圾回收优化

### 2. **响应速度提升**
- **前端优化**：虚拟滚动、懒加载、批处理等技术
- **网络优化**：请求去重、合并、压缩和预加载
- **渲染优化**：使用 requestAnimationFrame 和 Web Workers
- **资源优化**：图片懒加载、代码分割和预加载

### 3. **用户体验设计**
- **智能交互**：键盘快捷键、拖拽支持、智能提示
- **主题系统**：灵活的主题切换和样式管理
- **错误处理**：优雅的错误边界和用户友好的错误提示
- **加载状态**：骨架屏、进度指示和状态管理

### 4. **自动化监控**
- **实时监控**：全面的性能指标收集和分析
- **自动调优**：基于性能指标的自动参数调整
- **告警系统**：智能的性能异常检测和告警
- **报告生成**：详细的性能报告和趋势分析

这种全面的性能优化和用户体验设计使得 KiloCode 能够在高负载情况下保持良好的响应速度，同时为用户提供流畅、直观的交互体验。通过自动化监控和调优机制，系统能够持续优化性能，确保用户体验的一致性和可靠性。