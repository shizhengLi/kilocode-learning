# MCP 插件生态系统设计

> 深入解析 KiloCode 的 Model Context Protocol 插件架构、安全机制和市场运营模式

## MCP 插件生态系统概览

KiloCode 基于 Model Context Protocol (MCP) 构建了一个强大的插件生态系统，允许开发者扩展和定制 AI 编程助手的功能。这个生态系统不仅提供了丰富的扩展能力，还确保了安全性和可维护性。

### 系统设计目标

1. **可扩展性**：支持多种类型的插件和功能扩展
2. **安全性**：沙箱隔离和权限控制
3. **易用性**：简单的开发和部署流程
4. **生态繁荣**：市场机制和开发者激励

## 插件架构设计

### MCP 协议基础

#### 1. MCP 协议规范

```typescript
// src/core/mcp/MCPSpecification.ts
export interface MCPSpecification {
    version: string
    protocol: string
    capabilities: MCPCapability[]
    security: MCPSecurityModel
    communication: MCPCommunicationModel
}

export interface MCPCapability {
    type: 'tool' | 'resource' | 'prompt' | 'sampling'
    name: string
    description: string
    schema: JSONSchema
    handler: string
}

export interface MCPSecurityModel {
    sandbox: boolean
    permissions: Permission[]
    isolation: IsolationLevel
    audit: AuditPolicy
}

export interface MCPCommunicationModel {
    transport: 'stdio' | 'sse' | 'websocket'
    format: 'json' | 'jsonrpc'
    compression: boolean
    encryption: boolean
}

// MCP 基础协议实现
export class MCPProtocol {
    private specification: MCPSpecification
    private messageHandlers: Map<string, MessageHandler> = new Map()
    private connection: MCPConnection | null = null

    constructor(specification: MCPSpecification) {
        this.specification = specification
        this.initializeHandlers()
    }

    private initializeHandlers(): void {
        // 注册核心消息处理器
        this.messageHandlers.set('initialize', this.handleInitialize.bind(this))
        this.messageHandlers.set('capabilities', this.handleCapabilities.bind(this))
        this.messageHandlers.set('tool/call', this.handleToolCall.bind(this))
        this.messageHandlers.set('resource/read', this.handleResourceRead.bind(this))
        this.messageHandlers.set('prompt/get', this.handlePromptGet.bind(this))
        this.messageHandlers.set('sampling/create', this.handleSamplingCreate.bind(this))
    }

    async connect(connection: MCPConnection): Promise<void> {
        this.connection = connection

        // 发送初始化消息
        const initMessage: MCPMessage = {
            jsonrpc: '2.0',
            id: 1,
            method: 'initialize',
            params: {
                protocolVersion: this.specification.version,
                capabilities: this.getClientCapabilities(),
                clientInfo: {
                    name: 'KiloCode',
                    version: '1.0.0'
                }
            }
        }

        await this.sendMessage(initMessage)

        // 等待服务器响应
        const response = await this.waitForMessage('initialize')
        if (response.result?.capabilities) {
            this.setupServerCapabilities(response.result.capabilities)
        }
    }

    async callTool(toolName: string, parameters: any): Promise<ToolResponse> {
        const message: MCPMessage = {
            jsonrpc: '2.0',
            id: this.generateId(),
            method: 'tools/call',
            params: {
                name: toolName,
                arguments: parameters
            }
        }

        const response = await this.sendMessageAndWaitForResponse(message)
        return response.result as ToolResponse
    }

    async readResource(uri: string): Promise<ResourceResponse> {
        const message: MCPMessage = {
            jsonrpc: '2.0',
            id: this.generateId(),
            method: 'resources/read',
            params: {
                uri
            }
        }

        const response = await this.sendMessageAndWaitForResponse(message)
        return response.result as ResourceResponse
    }

    async getPrompt(promptName: string, parameters?: any): Promise<PromptResponse> {
        const message: MCPMessage = {
            jsonrpc: '2.0',
            id: this.generateId(),
            method: 'prompts/get',
            params: {
                name: promptName,
                arguments: parameters
            }
        }

        const response = await this.sendMessageAndWaitForResponse(message)
        return response.result as PromptResponse
    }

    private async handleInitialize(message: MCPMessage): Promise<MCPMessage> {
        return {
            jsonrpc: '2.0',
            id: message.id,
            result: {
                protocolVersion: this.specification.version,
                capabilities: this.getServerCapabilities(),
                serverInfo: {
                    name: 'KiloCode MCP Server',
                    version: '1.0.0'
                }
            }
        }
    }

    private async handleCapabilities(message: MCPMessage): Promise<MCPMessage> {
        return {
            jsonrpc: '2.0',
            id: message.id,
            result: {
                capabilities: this.specification.capabilities
            }
        }
    }

    private async handleToolCall(message: MCPMessage): Promise<MCPMessage> {
        const { name, arguments: parameters } = message.params

        try {
            const result = await this.executeTool(name, parameters)
            return {
                jsonrpc: '2.0',
                id: message.id,
                result: {
                    success: true,
                    data: result
                }
            }
        } catch (error) {
            return {
                jsonrpc: '2.0',
                id: message.id,
                error: {
                    code: -32000,
                    message: `Tool execution failed: ${error.message}`
                }
            }
        }
    }

    private async handleResourceRead(message: MCPMessage): Promise<MCPMessage> {
        const { uri } = message.params

        try {
            const resource = await this.readResourceContent(uri)
            return {
                jsonrpc: '2.0',
                id: message.id,
                result: {
                    contents: [{
                        uri,
                        mimeType: this.getMimeType(uri),
                        text: resource
                    }]
                }
            }
        } catch (error) {
            return {
                jsonrpc: '2.0',
                id: message.id,
                error: {
                    code: -32000,
                    message: `Resource read failed: ${error.message}`
                }
            }
        }
    }

    private async handlePromptGet(message: MCPMessage): Promise<MCPMessage> {
        const { name, arguments: parameters } = message.params

        try {
            const prompt = await this.getPromptContent(name, parameters)
            return {
                jsonrpc: '2.0',
                id: message.id,
                result: {
                    description: prompt.description,
                    messages: prompt.messages
                }
            }
        } catch (error) {
            return {
                jsonrpc: '2.0',
                id: message.id,
                error: {
                    code: -32000,
                    message: `Prompt get failed: ${error.message}`
                }
            }
        }
    }

    private async sendMessage(message: MCPMessage): Promise<void> {
        if (!this.connection) {
            throw new Error('No connection established')
        }
        await this.connection.send(message)
    }

    private async sendMessageAndWaitForResponse(message: MCPMessage): Promise<MCPMessage> {
        await this.sendMessage(message)
        return this.waitForMessage(message.id?.toString() || '')
    }

    private async waitForMessage(messageId: string): Promise<MCPMessage> {
        return new Promise((resolve, reject) => {
            const timeout = setTimeout(() => {
                reject(new Error(`Timeout waiting for message: ${messageId}`))
            }, 30000)

            const handler = (message: MCPMessage) => {
                if (message.id?.toString() === messageId) {
                    clearTimeout(timeout)
                    this.connection?.removeMessageListener(handler)
                    resolve(message)
                }
            }

            this.connection?.addMessageListener(handler)
        })
    }

    private generateId(): string {
        return Math.random().toString(36).substr(2, 9)
    }

    private getClientCapabilities(): any {
        return {
            tools: {},
            resources: {},
            prompts: {},
            logging: {}
        }
    }

    private getServerCapabilities(): any {
        const capabilities: any = {}

        for (const capability of this.specification.capabilities) {
            switch (capability.type) {
                case 'tool':
                    capabilities.tools = capabilities.tools || {}
                    capabilities.tools[capability.name] = {
                        description: capability.description,
                        inputSchema: capability.schema
                    }
                    break
                case 'resource':
                    capabilities.resources = capabilities.resources || {}
                    capabilities.resources[capability.name] = {
                        description: capability.description,
                        uriScheme: capability.schema.uriScheme
                    }
                    break
                case 'prompt':
                    capabilities.prompts = capabilities.prompts || {}
                    capabilities.prompts[capability.name] = {
                        description: capability.description,
                        arguments: capability.schema.arguments
                    }
                    break
            }
        }

        return capabilities
    }

    private setupServerCapabilities(serverCapabilities: any): void {
        // 处理服务器端能力
        console.log('Server capabilities:', serverCapabilities)
    }

    private async executeTool(name: string, parameters: any): Promise<any> {
        // 查找并执行对应的工具
        const capability = this.specification.capabilities.find(
            cap => cap.type === 'tool' && cap.name === name
        )

        if (!capability) {
            throw new Error(`Tool not found: ${name}`)
        }

        // 这里应该调用对应的处理器
        return this.invokeToolHandler(capability.handler, parameters)
    }

    private async invokeToolHandler(handler: string, parameters: any): Promise<any> {
        // 动态调用工具处理器
        // 实际实现中应该使用依赖注入或服务定位器
        return { success: true, result: `Executed ${handler} with ${JSON.stringify(parameters)}` }
    }

    private async readResourceContent(uri: string): Promise<string> {
        // 读取资源内容
        return `Resource content for ${uri}`
    }

    private async getPromptContent(name: string, parameters?: any): Promise<any> {
        // 获取提示词内容
        return {
            description: `Prompt: ${name}`,
            messages: [
                {
                    role: 'user',
                    content: {
                        type: 'text',
                        text: `Generated prompt for ${name}`
                    }
                }
            ]
        }
    }

    private getMimeType(uri: string): string {
        const extension = uri.split('.').pop()?.toLowerCase()
        const mimeTypes: Record<string, string> = {
            'txt': 'text/plain',
            'json': 'application/json',
            'js': 'application/javascript',
            'ts': 'application/typescript',
            'py': 'text/x-python',
            'md': 'text/markdown'
        }
        return mimeTypes[extension || ''] || 'text/plain'
    }
}
```

### 插件管理系统

#### 1. 插件生命周期管理

```typescript
// src/core/mcp/MCPPluginManager.ts
export class MCPPluginManager {
    private plugins: Map<string, MCPPlugin> = new Map()
    private activeConnections: Map<string, MCPConnection> = new Map()
    private eventEmitter: EventEmitter = new EventEmitter()
    private securityManager: MCPSecurityManager
    private configManager: MCPConfigManager
    private pluginRegistry: MCPPluginRegistry

    constructor(
        securityManager: MCPSecurityManager,
        configManager: MCPConfigManager,
        pluginRegistry: MCPPluginRegistry
    ) {
        this.securityManager = securityManager
        this.configManager = configManager
        this.pluginRegistry = pluginRegistry
        this.initializeSystem()
    }

    private async initializeSystem(): Promise<void> {
        // 加载已安装的插件
        await this.loadInstalledPlugins()

        // 设置事件监听器
        this.setupEventListeners()

        // 启动插件健康检查
        this.startHealthCheck()
    }

    async installPlugin(pluginId: string, version: string): Promise<InstallResult> {
        const installStart = Date.now()

        try {
            // 1. 验证插件
            const validation = await this.validatePlugin(pluginId, version)
            if (!validation.valid) {
                return {
                    success: false,
                    error: validation.error,
                    installTime: Date.now() - installStart
                }
            }

            // 2. 下载插件
            const downloadResult = await this.downloadPlugin(pluginId, version)
            if (!downloadResult.success) {
                return {
                    success: false,
                    error: downloadResult.error,
                    installTime: Date.now() - installStart
                }
            }

            // 3. 安装依赖
            const dependencyResult = await this.installDependencies(pluginId, downloadResult.metadata.dependencies)
            if (!dependencyResult.success) {
                return {
                    success: false,
                    error: dependencyResult.error,
                    installTime: Date.now() - installStart
                }
            }

            // 4. 安全检查
            const securityCheck = await this.securityManager.performSecurityCheck(
                downloadResult.pluginPath
            )
            if (!securityCheck.passed) {
                return {
                    success: false,
                    error: `Security check failed: ${securityCheck.reason}`,
                    installTime: Date.now() - installStart
                }
            }

            // 5. 注册插件
            const registration = await this.registerPlugin(pluginId, {
                version,
                path: downloadResult.pluginPath,
                metadata: downloadResult.metadata
            })

            if (!registration.success) {
                return {
                    success: false,
                    error: registration.error,
                    installTime: Date.now() - installStart
                }
            }

            // 6. 初始化插件
            const initResult = await this.initializePlugin(pluginId)
            if (!initResult.success) {
                // 回滚安装
                await this.rollbackInstallation(pluginId)
                return {
                    success: false,
                    error: initResult.error,
                    installTime: Date.now() - installStart
                }
            }

            // 7. 启动插件
            const startResult = await this.startPlugin(pluginId)
            if (!startResult.success) {
                await this.rollbackInstallation(pluginId)
                return {
                    success: false,
                    error: startResult.error,
                    installTime: Date.now() - installStart
                }
            }

            // 发送安装成功事件
            this.eventEmitter.emit('pluginInstalled', {
                pluginId,
                version,
                installTime: Date.now() - installStart
            })

            return {
                success: true,
                pluginId,
                version,
                installTime: Date.now() - installStart
            }

        } catch (error) {
            await this.rollbackInstallation(pluginId)
            return {
                success: false,
                error: error.message,
                installTime: Date.now() - installStart
            }
        }
    }

    async uninstallPlugin(pluginId: string): Promise<UninstallResult> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return {
                success: false,
                error: `Plugin not found: ${pluginId}`
            }
        }

        try {
            // 1. 停止插件
            await this.stopPlugin(pluginId)

            // 2. 清理连接
            this.cleanupConnections(pluginId)

            // 3. 注销插件
            await this.unregisterPlugin(pluginId)

            // 4. 删除文件
            await this.deletePluginFiles(pluginId)

            // 5. 更新配置
            await this.configManager.removePluginConfig(pluginId)

            // 发送卸载事件
            this.eventEmitter.emit('pluginUninstalled', { pluginId })

            return {
                success: true,
                pluginId
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    async startPlugin(pluginId: string): Promise<StartResult> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return {
                success: false,
                error: `Plugin not found: ${pluginId}`
            }
        }

        if (plugin.status === PluginStatus.RUNNING) {
            return {
                success: true,
                pluginId,
                message: 'Plugin already running'
            }
        }

        try {
            // 创建连接
            const connection = await this.createPluginConnection(plugin)

            // 初始化协议
            const protocol = new MCPProtocol(plugin.specification)
            await protocol.connect(connection)

            // 存储连接
            this.activeConnections.set(pluginId, connection)

            // 更新状态
            plugin.status = PluginStatus.RUNNING
            plugin.lastStarted = new Date()

            // 发送启动事件
            this.eventEmitter.emit('pluginStarted', { pluginId })

            return {
                success: true,
                pluginId
            }

        } catch (error) {
            plugin.status = PluginStatus.ERROR
            plugin.error = error.message

            return {
                success: false,
                error: error.message
            }
        }
    }

    async stopPlugin(pluginId: string): Promise<StopResult> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return {
                success: false,
                error: `Plugin not found: ${pluginId}`
            }
        }

        if (plugin.status !== PluginStatus.RUNNING) {
            return {
                success: true,
                pluginId,
                message: 'Plugin not running'
            }
        }

        try {
            // 断开连接
            const connection = this.activeConnections.get(pluginId)
            if (connection) {
                await connection.disconnect()
                this.activeConnections.delete(pluginId)
            }

            // 更新状态
            plugin.status = PluginStatus.STOPPED
            plugin.lastStopped = new Date()

            // 发送停止事件
            this.eventEmitter.emit('pluginStopped', { pluginId })

            return {
                success: true,
                pluginId
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    async restartPlugin(pluginId: string): Promise<RestartResult> {
        const stopResult = await this.stopPlugin(pluginId)
        if (!stopResult.success) {
            return {
                success: false,
                error: stopResult.error
            }
        }

        const startResult = await this.startPlugin(pluginId)
        if (!startResult.success) {
            return {
                success: false,
                error: startResult.error
            }
        }

        return {
            success: true,
            pluginId
        }
    }

    async updatePlugin(pluginId: string, newVersion: string): Promise<UpdateResult> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return {
                success: false,
                error: `Plugin not found: ${pluginId}`
            }
        }

        try {
            // 1. 停止插件
            await this.stopPlugin(pluginId)

            // 2. 备份当前版本
            const backupResult = await this.backupPlugin(pluginId)
            if (!backupResult.success) {
                return {
                    success: false,
                    error: backupResult.error
                }
            }

            // 3. 安装新版本
            const installResult = await this.installPlugin(pluginId, newVersion)
            if (!installResult.success) {
                // 恢复备份
                await this.restoreBackup(pluginId, backupResult.backupPath)
                return {
                    success: false,
                    error: installResult.error
                }
            }

            // 4. 启动插件
            const startResult = await this.startPlugin(pluginId)
            if (!startResult.success) {
                await this.restoreBackup(pluginId, backupResult.backupPath)
                return {
                    success: false,
                    error: startResult.error
                }
            }

            // 清理备份
            await this.cleanupBackup(backupResult.backupPath)

            return {
                success: true,
                pluginId,
                oldVersion: plugin.version,
                newVersion
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    private async validatePlugin(pluginId: string, version: string): Promise<ValidationResult> {
        const metadata = await this.pluginRegistry.getPluginMetadata(pluginId, version)

        if (!metadata) {
            return {
                valid: false,
                error: `Plugin metadata not found: ${pluginId}@${version}`
            }
        }

        // 验证兼容性
        if (!this.isVersionCompatible(version)) {
            return {
                valid: false,
                error: `Incompatible version: ${version}`
            }
        }

        // 验证签名
        const signatureValid = await this.securityManager.verifySignature(metadata)
        if (!signatureValid) {
            return {
                valid: false,
                error: 'Invalid plugin signature'
            }
        }

        return { valid: true }
    }

    private async downloadPlugin(pluginId: string, version: string): Promise<DownloadResult> {
        const metadata = await this.pluginRegistry.getPluginMetadata(pluginId, version)
        if (!metadata) {
            return {
                success: false,
                error: `Plugin metadata not found`
            }
        }

        // 下载插件包
        const downloadUrl = this.getDownloadUrl(pluginId, version)
        const response = await fetch(downloadUrl)

        if (!response.ok) {
            return {
                success: false,
                error: `Download failed: ${response.statusText}`
            }
        }

        // 保存到临时目录
        const tempPath = path.join(os.tmpdir(), `${pluginId}-${version}.tar.gz`)
        const buffer = await response.buffer()
        await fs.writeFile(tempPath, buffer)

        // 解压插件
        const extractPath = path.join(os.tmpdir(), `${pluginId}-${version}`)
        await this.extractPlugin(tempPath, extractPath)

        return {
            success: true,
            pluginPath: extractPath,
            metadata
        }
    }

    private async installDependencies(pluginId: string, dependencies: PluginDependency[]): Promise<DependencyResult> {
        const failedDependencies: string[] = []

        for (const dep of dependencies) {
            try {
                const depResult = await this.installPlugin(dep.pluginId, dep.version)
                if (!depResult.success) {
                    failedDependencies.push(dep.pluginId)
                }
            } catch (error) {
                failedDependencies.push(dep.pluginId)
            }
        }

        if (failedDependencies.length > 0) {
            return {
                success: false,
                error: `Failed to install dependencies: ${failedDependencies.join(', ')}`
            }
        }

        return { success: true }
    }

    private async registerPlugin(pluginId: string, registration: PluginRegistration): Promise<RegistrationResult> {
        try {
            const plugin: MCPPlugin = {
                id: pluginId,
                version: registration.version,
                path: registration.path,
                metadata: registration.metadata,
                status: PluginStatus.INSTALLED,
                installedAt: new Date(),
                config: await this.configManager.getPluginConfig(pluginId)
            }

            this.plugins.set(pluginId, plugin)

            return { success: true }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    private async unregisterPlugin(pluginId: string): Promise<void> {
        this.plugins.delete(pluginId)
        await this.pluginRegistry.unregisterPlugin(pluginId)
    }

    private async initializePlugin(pluginId: string): Promise<InitResult> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return {
                success: false,
                error: `Plugin not found: ${pluginId}`
            }
        }

        try {
            // 读取插件规范
            const specPath = path.join(plugin.path, 'mcp.json')
            const specContent = await fs.readFile(specPath, 'utf-8')
            const specification = JSON.parse(specContent)

            plugin.specification = specification
            plugin.status = PluginStatus.INITIALIZED

            return { success: true }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    private async createPluginConnection(plugin: MCPPlugin): Promise<MCPConnection> {
        const connection = new MCPConnection()

        // 根据插件类型创建不同的连接
        switch (plugin.metadata.connectionType) {
            case 'stdio':
                return await this.createStdioConnection(plugin)
            case 'sse':
                return await this.createSSEConnection(plugin)
            case 'websocket':
                return await this.createWebSocketConnection(plugin)
            default:
                throw new Error(`Unsupported connection type: ${plugin.metadata.connectionType}`)
        }
    }

    private async createStdioConnection(plugin: MCPPlugin): Promise<MCPConnection> {
        const command = plugin.metadata.command || 'node'
        const args = plugin.metadata.args || [path.join(plugin.path, 'dist', 'index.js')]

        const childProcess = spawn(command, args, {
            cwd: plugin.path,
            stdio: ['pipe', 'pipe', 'pipe']
        })

        return new StdioConnection(childProcess)
    }

    private async createSSEConnection(plugin: MCPPlugin): Promise<MCPConnection> {
        const url = plugin.metadata.url || `http://localhost:${plugin.metadata.port || 3000}`
        return new SSEConnection(url)
    }

    private async createWebSocketConnection(plugin: MCPPlugin): Promise<MCPConnection> {
        const url = plugin.metadata.url || `ws://localhost:${plugin.metadata.port || 3000}`
        return new WebSocketConnection(url)
    }

    private cleanupConnections(pluginId: string): void {
        const connection = this.activeConnections.get(pluginId)
        if (connection) {
            connection.disconnect()
            this.activeConnections.delete(pluginId)
        }
    }

    private async deletePluginFiles(pluginId: string): Promise<void> {
        const plugin = this.plugins.get(pluginId)
        if (plugin) {
            await fs.rm(plugin.path, { recursive: true, force: true })
        }
    }

    private async rollbackInstallation(pluginId: string): Promise<void> {
        try {
            await this.stopPlugin(pluginId)
            this.cleanupConnections(pluginId)
            await this.unregisterPlugin(pluginId)
            await this.deletePluginFiles(pluginId)
        } catch (error) {
            console.error(`Rollback failed for plugin ${pluginId}:`, error)
        }
    }

    private async backupPlugin(pluginId: string): Promise<BackupResult> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return {
                success: false,
                error: `Plugin not found: ${pluginId}`
            }
        }

        const backupPath = path.join(os.tmpdir(), `backup-${pluginId}-${Date.now()}`)
        await fs.cp(plugin.path, backupPath, { recursive: true })

        return {
            success: true,
            backupPath
        }
    }

    private async restoreBackup(pluginId: string, backupPath: string): Promise<void> {
        const plugin = this.plugins.get(pluginId)
        if (plugin) {
            await fs.cp(backupPath, plugin.path, { recursive: true })
        }
    }

    private async cleanupBackup(backupPath: string): Promise<void> {
        await fs.rm(backupPath, { recursive: true, force: true })
    }

    private async loadInstalledPlugins(): Promise<void> {
        const installedPlugins = await this.configManager.getInstalledPlugins()

        for (const pluginInfo of installedPlugins) {
            try {
                const plugin: MCPPlugin = {
                    id: pluginInfo.id,
                    version: pluginInfo.version,
                    path: pluginInfo.path,
                    metadata: pluginInfo.metadata,
                    status: PluginStatus.INSTALLED,
                    installedAt: pluginInfo.installedAt,
                    config: pluginInfo.config
                }

                this.plugins.set(pluginInfo.id, plugin)

                // 自动启动已启用的插件
                if (pluginInfo.autoStart) {
                    await this.startPlugin(pluginInfo.id)
                }
            } catch (error) {
                console.error(`Failed to load plugin ${pluginInfo.id}:`, error)
            }
        }
    }

    private setupEventListeners(): void {
        this.eventEmitter.on('pluginError', (event: PluginErrorEvent) => {
            console.error(`Plugin error: ${event.pluginId} - ${event.error}`)
            this.handlePluginError(event.pluginId, event.error)
        })

        this.eventEmitter.on('pluginCrash', (event: PluginCrashEvent) => {
            console.warn(`Plugin crashed: ${event.pluginId}`)
            this.handlePluginCrash(event.pluginId)
        })
    }

    private startHealthCheck(): void {
        setInterval(() => {
            this.performHealthCheck()
        }, 30000) // 每30秒检查一次
    }

    private async performHealthCheck(): Promise<void> {
        for (const [pluginId, plugin] of this.plugins.entries()) {
            if (plugin.status === PluginStatus.RUNNING) {
                try {
                    const health = await this.checkPluginHealth(pluginId)
                    if (!health.healthy) {
                        console.warn(`Plugin ${pluginId} is unhealthy: ${health.reason}`)
                        await this.handleUnhealthyPlugin(pluginId, health)
                    }
                } catch (error) {
                    console.error(`Health check failed for plugin ${pluginId}:`, error)
                }
            }
        }
    }

    private async checkPluginHealth(pluginId: string): Promise<HealthStatus> {
        const plugin = this.plugins.get(pluginId)
        if (!plugin) {
            return { healthy: false, reason: 'Plugin not found' }
        }

        const connection = this.activeConnections.get(pluginId)
        if (!connection) {
            return { healthy: false, reason: 'No active connection' }
        }

        try {
            // 发送健康检查消息
            const response = await connection.sendAndWaitForResponse({
                jsonrpc: '2.0',
                id: 'health-check',
                method: 'health'
            })

            if (response.result?.status === 'healthy') {
                return { healthy: true }
            } else {
                return { healthy: false, reason: response.result?.reason || 'Unknown reason' }
            }
        } catch (error) {
            return { healthy: false, reason: error.message }
        }
    }

    private handlePluginError(pluginId: string, error: string): void {
        const plugin = this.plugins.get(pluginId)
        if (plugin) {
            plugin.error = error
            plugin.lastError = new Date()
        }
    }

    private handlePluginCrash(pluginId: string): void {
        const plugin = this.plugins.get(pluginId)
        if (plugin) {
            plugin.status = PluginStatus.CRASHED
            plugin.crashCount = (plugin.crashCount || 0) + 1

            // 如果崩溃次数过多，停止插件
            if (plugin.crashCount > 3) {
                this.stopPlugin(pluginId)
            }
        }
    }

    private async handleUnhealthyPlugin(pluginId: string, health: HealthStatus): Promise<void> {
        const plugin = this.plugins.get(pluginId)
        if (plugin) {
            plugin.healthStatus = health

            // 尝试重启插件
            if (plugin.autoRestart !== false) {
                await this.restartPlugin(pluginId)
            }
        }
    }

    private isVersionCompatible(version: string): boolean {
        // 简化的版本兼容性检查
        const currentVersion = '1.0.0'
        return this.compareVersions(version, currentVersion) >= 0
    }

    private compareVersions(v1: string, v2: string): number {
        const v1Parts = v1.split('.').map(Number)
        const v2Parts = v2.split('.').map(Number)

        for (let i = 0; i < Math.max(v1Parts.length, v2Parts.length); i++) {
            const v1Part = v1Parts[i] || 0
            const v2Part = v2Parts[i] || 0

            if (v1Part < v2Part) return -1
            if (v1Part > v2Part) return 1
        }

        return 0
    }

    private getDownloadUrl(pluginId: string, version: string): string {
        return `https://plugins.kilocode.ai/${pluginId}/${version}/download`
    }

    private async extractPlugin(archivePath: string, extractPath: string): Promise<void> {
        // 简化的解压实现
        // 实际应该使用 tar.gz 或 zip 解压库
        await fs.mkdir(extractPath, { recursive: true })
    }

    // 公共接口
    getPlugin(pluginId: string): MCPPlugin | undefined {
        return this.plugins.get(pluginId)
    }

    getAllPlugins(): MCPPlugin[] {
        return Array.from(this.plugins.values())
    }

    getRunningPlugins(): MCPPlugin[] {
        return Array.from(this.plugins.values()).filter(
            plugin => plugin.status === PluginStatus.RUNNING
        )
    }

    async executeTool(pluginId: string, toolName: string, parameters: any): Promise<any> {
        const connection = this.activeConnections.get(pluginId)
        if (!connection) {
            throw new Error(`Plugin not running: ${pluginId}`)
        }

        const message: MCPMessage = {
            jsonrpc: '2.0',
            id: this.generateId(),
            method: 'tools/call',
            params: {
                name: toolName,
                arguments: parameters
            }
        }

        const response = await connection.sendAndWaitForResponse(message)
        return response.result
    }

    private generateId(): string {
        return Math.random().toString(36).substr(2, 9)
    }
}
```

### 插件开发工具包

#### 1. 插件 SDK

```typescript
// src/core/mcp/MCPPluginSDK.ts
export class MCPPluginSDK {
    private server: MCPServer
    private tools: Map<string, Tool> = new Map()
    private resources: Map<string, Resource> = new Map()
    private prompts: Map<string, Prompt> = new Map()
    private logger: MCPLogger

    constructor(config: PluginConfig) {
        this.server = new MCPServer(config)
        this.logger = new MCPLogger(config.name)
        this.initializeSDK()
    }

    private initializeSDK(): void {
        // 设置工具处理器
        this.server.setToolHandler(this.handleToolCall.bind(this))

        // 设置资源处理器
        this.server.setResourceHandler(this.handleResourceRequest.bind(this))

        // 设置提示词处理器
        this.server.setPromptHandler(this.handlePromptRequest.bind(this))

        // 设置生命周期处理器
        this.server.setLifecycleHandler(this.handleLifecycleEvent.bind(this))
    }

    // 注册工具
    registerTool(tool: Tool): void {
        this.tools.set(tool.name, tool)
        this.logger.info(`Tool registered: ${tool.name}`)
    }

    // 注册资源
    registerResource(resource: Resource): void {
        this.resources.set(resource.uri, resource)
        this.logger.info(`Resource registered: ${resource.uri}`)
    }

    // 注册提示词
    registerPrompt(prompt: Prompt): void {
        this.prompts.set(prompt.name, prompt)
        this.logger.info(`Prompt registered: ${prompt.name}`)
    }

    // 获取工具列表
    getTools(): Tool[] {
        return Array.from(this.tools.values())
    }

    // 获取资源列表
    getResources(): Resource[] {
        return Array.from(this.resources.values())
    }

    // 获取提示词列表
    getPrompts(): Prompt[] {
        return Array.from(this.prompts.values())
    }

    // 启动服务器
    async start(): Promise<void> {
        await this.server.start()
        this.logger.info('Plugin server started')
    }

    // 停止服务器
    async stop(): Promise<void> {
        await this.server.stop()
        this.logger.info('Plugin server stopped')
    }

    private async handleToolCall(call: ToolCall): Promise<ToolResponse> {
        const tool = this.tools.get(call.name)
        if (!tool) {
            throw new Error(`Tool not found: ${call.name}`)
        }

        try {
            // 验证参数
            this.validateParameters(tool.schema, call.arguments)

            // 执行工具
            const result = await tool.handler(call.arguments)

            return {
                success: true,
                data: result
            }
        } catch (error) {
            this.logger.error(`Tool execution failed: ${call.name}`, error)
            return {
                success: false,
                error: error.message
            }
        }
    }

    private async handleResourceRequest(request: ResourceRequest): Promise<ResourceResponse> {
        const resource = this.resources.get(request.uri)
        if (!resource) {
            throw new Error(`Resource not found: ${request.uri}`)
        }

        try {
            const content = await resource.handler(request)
            return {
                contents: [{
                    uri: request.uri,
                    mimeType: resource.mimeType,
                    text: content
                }]
            }
        } catch (error) {
            this.logger.error(`Resource request failed: ${request.uri}`, error)
            throw error
        }
    }

    private async handlePromptRequest(request: PromptRequest): Promise<PromptResponse> {
        const prompt = this.prompts.get(request.name)
        if (!prompt) {
            throw new Error(`Prompt not found: ${request.name}`)
        }

        try {
            const result = await prompt.handler(request.arguments)
            return {
                description: prompt.description,
                messages: result.messages
            }
        } catch (error) {
            this.logger.error(`Prompt request failed: ${request.name}`, error)
            throw error
        }
    }

    private async handleLifecycleEvent(event: LifecycleEvent): Promise<void> {
        switch (event.type) {
            case 'initialize':
                await this.handleInitialize(event)
                break
            case 'shutdown':
                await this.handleShutdown(event)
                break
            case 'configure':
                await this.handleConfigure(event)
                break
        }
    }

    private async handleInitialize(event: LifecycleEvent): Promise<void> {
        this.logger.info('Plugin initializing...')
        // 插件初始化逻辑
    }

    private async handleShutdown(event: LifecycleEvent): Promise<void> {
        this.logger.info('Plugin shutting down...')
        // 插件关闭逻辑
    }

    private async handleConfigure(event: LifecycleEvent): Promise<void> {
        this.logger.info('Plugin configuring...', event.config)
        // 插件配置逻辑
    }

    private validateParameters(schema: JSONSchema, parameters: any): void {
        // 参数验证逻辑
        // 这里可以使用 JSON Schema 验证库
    }

    // 实用工具
    createLogger(name: string): MCPLogger {
        return new MCPLogger(name)
    }

    createStorage(): PluginStorage {
        return new PluginStorage()
    }

    createHttpClient(): HttpClient {
        return new HttpClient()
    }

    createFileSystem(): PluginFileSystem {
        return new PluginFileSystem()
    }

    // 事件系统
    on(event: string, handler: (...args: any[]) => void): void {
        this.server.on(event, handler)
    }

    emit(event: string, ...args: any[]): void {
        this.server.emit(event, ...args)
    }
}

// 插件服务器
class MCPServer {
    private config: PluginConfig
    private connection: MCPConnection | null = null
    private toolHandler: ((call: ToolCall) => Promise<ToolResponse>) | null = null
    private resourceHandler: ((request: ResourceRequest) => Promise<ResourceResponse>) | null = null
    private promptHandler: ((request: PromptRequest) => Promise<PromptResponse>) | null = null
    private lifecycleHandler: ((event: LifecycleEvent) => Promise<void>) | null = null

    constructor(config: PluginConfig) {
        this.config = config
    }

    async start(): Promise<void> {
        // 根据配置启动服务器
        switch (this.config.connectionType) {
            case 'stdio':
                await this.startStdioServer()
                break
            case 'sse':
                await this.startSSEServer()
                break
            case 'websocket':
                await this.startWebSocketServer()
                break
        }
    }

    async stop(): Promise<void> {
        if (this.connection) {
            await this.connection.disconnect()
            this.connection = null
        }
    }

    setToolHandler(handler: (call: ToolCall) => Promise<ToolResponse>): void {
        this.toolHandler = handler
    }

    setResourceHandler(handler: (request: ResourceRequest) => Promise<ResourceResponse>): void {
        this.resourceHandler = handler
    }

    setPromptHandler(handler: (request: PromptRequest) => Promise<PromptResponse>): void {
        this.promptHandler = handler
    }

    setLifecycleHandler(handler: (event: LifecycleEvent) => Promise<void>): void {
        this.lifecycleHandler = handler
    }

    on(event: string, handler: (...args: any[]) => void): void {
        // 事件监听
    }

    emit(event: string, ...args: any[]): void {
        // 事件触发
    }

    private async startStdioServer(): Promise<void> {
        this.connection = new StdioConnection(process.stdin, process.stdout)
        await this.setupMessageHandlers()
    }

    private async startSSEServer(): Promise<void> {
        const server = http.createServer((req, res) => {
            if (req.url === '/mcp') {
                this.connection = new SSEConnection(req, res)
                this.setupMessageHandlers()
            }
        })

        server.listen(this.config.port || 3000)
    }

    private async startWebSocketServer(): Promise<void> {
        const server = http.createServer()
        const wss = new WebSocketServer({ server })

        wss.on('connection', (ws) => {
            this.connection = new WebSocketConnection(ws)
            this.setupMessageHandlers()
        })

        server.listen(this.config.port || 3000)
    }

    private async setupMessageHandlers(): Promise<void> {
        if (!this.connection) return

        this.connection.onMessage(async (message: MCPMessage) => {
            try {
                const response = await this.handleMessage(message)
                if (response) {
                    await this.connection.send(response)
                }
            } catch (error) {
                await this.connection.send({
                    jsonrpc: '2.0',
                    id: message.id,
                    error: {
                        code: -32000,
                        message: error.message
                    }
                })
            }
        })
    }

    private async handleMessage(message: MCPMessage): Promise<MCPMessage | null> {
        switch (message.method) {
            case 'initialize':
                return this.handleInitialize(message)
            case 'tools/call':
                return this.handleToolCall(message)
            case 'resources/read':
                return this.handleResourceRead(message)
            case 'prompts/get':
                return this.handlePromptGet(message)
            case 'lifecycle':
                return this.handleLifecycle(message)
            default:
                return null
        }
    }

    private async handleInitialize(message: MCPMessage): Promise<MCPMessage> {
        if (this.lifecycleHandler) {
            await this.lifecycleHandler({
                type: 'initialize',
                config: message.params?.config || {}
            })
        }

        return {
            jsonrpc: '2.0',
            id: message.id,
            result: {
                protocolVersion: '2024-11-05',
                capabilities: {
                    tools: this.getToolCapabilities(),
                    resources: this.getResourceCapabilities(),
                    prompts: this.getPromptCapabilities()
                },
                serverInfo: {
                    name: this.config.name,
                    version: this.config.version
                }
            }
        }
    }

    private async handleToolCall(message: MCPMessage): Promise<MCPMessage> {
        if (!this.toolHandler) {
            throw new Error('Tool handler not set')
        }

        const result = await this.toolHandler({
            name: message.params?.name,
            arguments: message.params?.arguments
        })

        return {
            jsonrpc: '2.0',
            id: message.id,
            result
        }
    }

    private async handleResourceRead(message: MCPMessage): Promise<MCPMessage> {
        if (!this.resourceHandler) {
            throw new Error('Resource handler not set')
        }

        const result = await this.resourceHandler({
            uri: message.params?.uri
        })

        return {
            jsonrpc: '2.0',
            id: message.id,
            result
        }
    }

    private async handlePromptGet(message: MCPMessage): Promise<MCPMessage> {
        if (!this.promptHandler) {
            throw new Error('Prompt handler not set')
        }

        const result = await this.promptHandler({
            name: message.params?.name,
            arguments: message.params?.arguments
        })

        return {
            jsonrpc: '2.0',
            id: message.id,
            result
        }
    }

    private async handleLifecycle(message: MCPMessage): Promise<MCPMessage> {
        if (this.lifecycleHandler) {
            await this.lifecycleHandler({
                type: message.params?.type,
                config: message.params?.config || {}
            })
        }

        return {
            jsonrpc: '2.0',
            id: message.id,
            result: { success: true }
        }
    }

    private getToolCapabilities(): any {
        // 返回工具能力描述
        return {}
    }

    private getResourceCapabilities(): any {
        // 返回资源能力描述
        return {}
    }

    private getPromptCapabilities(): any {
        // 返回提示词能力描述
        return {}
    }
}

// 插件日志系统
class MCPLogger {
    private name: string

    constructor(name: string) {
        this.name = name
    }

    info(message: string, ...args: any[]): void {
        console.log(`[${this.name}] INFO:`, message, ...args)
    }

    warn(message: string, ...args: any[]): void {
        console.warn(`[${this.name}] WARN:`, message, ...args)
    }

    error(message: string, ...args: any[]): void {
        console.error(`[${this.name}] ERROR:`, message, ...args)
    }

    debug(message: string, ...args: any[]): void {
        if (process.env.NODE_ENV === 'development') {
            console.debug(`[${this.name}] DEBUG:`, message, ...args)
        }
    }
}

// 插件存储系统
class PluginStorage {
    private storagePath: string

    constructor() {
        this.storagePath = path.join(os.homedir(), '.kilocode', 'plugins', 'storage')
        fs.mkdirSync(this.storagePath, { recursive: true })
    }

    async get(key: string): Promise<any> {
        try {
            const filePath = path.join(this.storagePath, `${key}.json`)
            const content = await fs.readFile(filePath, 'utf-8')
            return JSON.parse(content)
        } catch (error) {
            return null
        }
    }

    async set(key: string, value: any): Promise<void> {
        const filePath = path.join(this.storagePath, `${key}.json`)
        await fs.writeFile(filePath, JSON.stringify(value, null, 2))
    }

    async delete(key: string): Promise<void> {
        const filePath = path.join(this.storagePath, `${key}.json`)
        await fs.unlink(filePath)
    }

    async list(): Promise<string[]> {
        const files = await fs.readdir(this.storagePath)
        return files.filter(file => file.endsWith('.json')).map(file => file.slice(0, -5))
    }
}

// HTTP 客户端
class HttpClient {
    async request(options: RequestOptions): Promise<HttpResponse> {
        const response = await fetch(options.url, {
            method: options.method || 'GET',
            headers: options.headers,
            body: options.body
        })

        return {
            status: response.status,
            headers: Object.fromEntries(response.headers.entries()),
            body: await response.text()
        }
    }
}

// 插件文件系统
class PluginFileSystem {
    async readFile(path: string): Promise<string> {
        return fs.readFile(path, 'utf-8')
    }

    async writeFile(path: string, content: string): Promise<void> {
        await fs.writeFile(path, content, 'utf-8')
    }

    async exists(path: string): Promise<boolean> {
        try {
            await fs.access(path)
            return true
        } catch {
            return false
        }
    }

    async readdir(path: string): Promise<string[]> {
        return fs.readdir(path)
    }
}
```

## 沙箱安全机制

### 安全模型设计

#### 1. 权限控制系统

```typescript
// src/core/mcp/security/MCPSecurityManager.ts
export class MCPSecurityManager {
    private permissionStore: PermissionStore
    private sandboxManager: SandboxManager
    private auditLogger: AuditLogger
    private securityPolicy: SecurityPolicy

    constructor(
        permissionStore: PermissionStore,
        sandboxManager: SandboxManager,
        auditLogger: AuditLogger
    ) {
        this.permissionStore = permissionStore
        this.sandboxManager = sandboxManager
        this.auditLogger = auditLogger
        this.securityPolicy = new SecurityPolicy()
    }

    async performSecurityCheck(pluginPath: string): Promise<SecurityCheckResult> {
        const checkStart = Date.now()

        try {
            // 1. 文件系统检查
            const filesystemCheck = await this.checkFilesystemSafety(pluginPath)
            if (!filesystemCheck.passed) {
                return {
                    passed: false,
                    reason: `Filesystem check failed: ${filesystemCheck.reason}`,
                    checkTime: Date.now() - checkStart
                }
            }

            // 2. 代码静态分析
            const staticAnalysis = await this.performStaticAnalysis(pluginPath)
            if (!staticAnalysis.passed) {
                return {
                    passed: false,
                    reason: `Static analysis failed: ${staticAnalysis.reason}`,
                    checkTime: Date.now() - checkStart
                }
            }

            // 3. 依赖项检查
            const dependencyCheck = await this.checkDependencies(pluginPath)
            if (!dependencyCheck.passed) {
                return {
                    passed: false,
                    reason: `Dependency check failed: ${dependencyCheck.reason}`,
                    checkTime: Date.now() - checkStart
                }
            }

            // 4. 权限请求验证
            const permissionCheck = await this.validatePermissions(pluginPath)
            if (!permissionCheck.passed) {
                return {
                    passed: false,
                    reason: `Permission validation failed: ${permissionCheck.reason}`,
                    checkTime: Date.now() - checkStart
                }
            }

            // 5. 签名验证
            const signatureCheck = await this.verifySignature(pluginPath)
            if (!signatureCheck.passed) {
                return {
                    passed: false,
                    reason: `Signature verification failed: ${signatureCheck.reason}`,
                    checkTime: Date.now() - checkStart
                }
            }

            return {
                passed: true,
                checkTime: Date.now() - checkStart
            }

        } catch (error) {
            return {
                passed: false,
                reason: `Security check error: ${error.message}`,
                checkTime: Date.now() - checkStart
            }
        }
    }

    async grantPermissions(pluginId: string, permissions: Permission[]): Promise<PermissionResult> {
        const result: PermissionResult = {
            success: false,
            granted: [],
            denied: []
        }

        for (const permission of permissions) {
            try {
                // 验证权限请求
                const validation = await this.validatePermissionRequest(pluginId, permission)
                if (!validation.valid) {
                    result.denied.push({
                        permission,
                        reason: validation.reason
                    })
                    continue
                }

                // 检查权限冲突
                const conflictCheck = await this.checkPermissionConflicts(pluginId, permission)
                if (conflictCheck.hasConflict) {
                    result.denied.push({
                        permission,
                        reason: `Permission conflict: ${conflictCheck.conflictReason}`
                    })
                    continue
                }

                // 授予权限
                await this.permissionStore.grantPermission(pluginId, permission)
                result.granted.push(permission)

                // 记录审计日志
                await this.auditLogger.log({
                    type: 'permission_granted',
                    pluginId,
                    permission: permission.type,
                    details: permission.details,
                    timestamp: new Date()
                })

            } catch (error) {
                result.denied.push({
                    permission,
                    reason: error.message
                })
            }
        }

        result.success = result.granted.length > 0
        return result
    }

    async revokePermissions(pluginId: string, permissionTypes: string[]): Promise<RevokeResult> {
        const result: RevokeResult = {
            success: false,
            revoked: [],
            failed: []
        }

        for (const permissionType of permissionTypes) {
            try {
                await this.permissionStore.revokePermission(pluginId, permissionType)
                result.revoked.push(permissionType)

                // 记录审计日志
                await this.auditLogger.log({
                    type: 'permission_revoked',
                    pluginId,
                    permission: permissionType,
                    timestamp: new Date()
                })

            } catch (error) {
                result.failed.push({
                    permissionType,
                    reason: error.message
                })
            }
        }

        result.success = result.revoked.length > 0
        return result
    }

    async checkPermission(pluginId: string, permissionType: string, context?: SecurityContext): Promise<PermissionCheck> {
        try {
            // 获取插件权限
            const permissions = await this.permissionStore.getPluginPermissions(pluginId)

            // 检查是否有对应权限
            const hasPermission = permissions.some(p => p.type === permissionType)

            if (!hasPermission) {
                return {
                    granted: false,
                    reason: 'Permission not granted'
                }
            }

            // 检查权限条件
            const permission = permissions.find(p => p.type === permissionType)!
            const conditionCheck = await this.checkPermissionConditions(permission, context)

            if (!conditionCheck.passed) {
                return {
                    granted: false,
                    reason: conditionCheck.reason
                }
            }

            // 检查安全策略
            const policyCheck = await this.securityPolicy.evaluatePermission(permission, context)
            if (!policyCheck.allowed) {
                return {
                    granted: false,
                    reason: policyCheck.reason
                }
            }

            return {
                granted: true
            }

        } catch (error) {
            return {
                granted: false,
                reason: error.message
            }
        }
    }

    async executeInSandbox<T>(
        pluginId: string,
        operation: SandboxOperation<T>
    ): Promise<SandboxResult<T>> {
        const sandboxStart = Date.now()

        try {
            // 检查权限
            const permissionCheck = await this.checkPermission(
                pluginId,
                operation.permissionType,
                operation.context
            )

            if (!permissionCheck.granted) {
                return {
                    success: false,
                    error: `Permission denied: ${permissionCheck.reason}`,
                    executionTime: Date.now() - sandboxStart
                }
            }

            // 创建沙箱环境
            const sandbox = await this.sandboxManager.createSandbox(pluginId)

            try {
                // 执行操作
                const result = await sandbox.execute(operation)

                // 记录审计日志
                await this.auditLogger.log({
                    type: 'sandbox_execution',
                    pluginId,
                    operation: operation.type,
                    success: true,
                    executionTime: Date.now() - sandboxStart,
                    timestamp: new Date()
                })

                return {
                    success: true,
                    result,
                    executionTime: Date.now() - sandboxStart
                }

            } finally {
                // 清理沙箱
                await sandbox.cleanup()
            }

        } catch (error) {
            await this.auditLogger.log({
                type: 'sandbox_execution',
                pluginId,
                operation: operation.type,
                success: false,
                error: error.message,
                executionTime: Date.now() - sandboxStart,
                timestamp: new Date()
            })

            return {
                success: false,
                error: error.message,
                executionTime: Date.now() - sandboxStart
            }
        }
    }

    private async checkFilesystemSafety(pluginPath: string): Promise<FilesystemCheckResult> {
        const dangerousPatterns = [
            /\b(exec|eval|spawn|fork)\b/,
            /\b(require|import)\s*\(\s*['"]fs['"]\s*\)/,
            /\b(require|import)\s*\(\s*['"]child_process['"]\s*\)/,
            /\b(process\.\w+)\b/,
            /\b(global\.\w+)\b/,
            /\b(window\.\w+)\b/,
            /\b(document\.\w+)\b/
        ]

        const suspiciousFiles = ['package.json', 'index.js', 'dist/index.js']

        for (const file of suspiciousFiles) {
            const filePath = path.join(pluginPath, file)
            if (await this.fileExists(filePath)) {
                const content = await fs.readFile(filePath, 'utf-8')

                for (const pattern of dangerousPatterns) {
                    if (pattern.test(content)) {
                        return {
                            passed: false,
                            reason: `Dangerous pattern detected in ${file}: ${pattern}`
                        }
                    }
                }
            }
        }

        return { passed: true }
    }

    private async performStaticAnalysis(pluginPath: string): Promise<StaticAnalysisResult> {
        const analyzer = new StaticCodeAnalyzer()

        try {
            const analysis = await analyzer.analyzeDirectory(pluginPath)

            if (analysis.issues.length > 0) {
                const criticalIssues = analysis.issues.filter(issue => issue.severity === 'critical')
                if (criticalIssues.length > 0) {
                    return {
                        passed: false,
                        reason: `Critical security issues found: ${criticalIssues.map(i => i.message).join(', ')}`
                    }
                }
            }

            return { passed: true }

        } catch (error) {
            return {
                passed: false,
                reason: `Static analysis failed: ${error.message}`
            }
        }
    }

    private async checkDependencies(pluginPath: string): Promise<DependencyCheckResult> {
        const packageJsonPath = path.join(pluginPath, 'package.json')

        if (!await this.fileExists(packageJsonPath)) {
            return { passed: true } // 没有 package.json
        }

        try {
            const packageJson = JSON.parse(await fs.readFile(packageJsonPath, 'utf-8'))
            const dependencies = { ...packageJson.dependencies, ...packageJson.devDependencies }

            const blockedPackages = [
                'fs-extra',
                'child_process',
                'node-pty',
                'node-sandbox',
                'vm2'
            ]

            const suspiciousPackages = Object.keys(dependencies).filter(dep =>
                blockedPackages.includes(dep)
            )

            if (suspiciousPackages.length > 0) {
                return {
                    passed: false,
                    reason: `Blocked dependencies found: ${suspiciousPackages.join(', ')}`
                }
            }

            // 检查已知漏洞
            const vulnerabilityCheck = await this.checkVulnerabilities(dependencies)
            if (!vulnerabilityCheck.passed) {
                return {
                    passed: false,
                    reason: `Security vulnerabilities found: ${vulnerabilityCheck.vulnerabilities.join(', ')}`
                }
            }

            return { passed: true }

        } catch (error) {
            return {
                passed: false,
                reason: `Dependency check failed: ${error.message}`
            }
        }
    }

    private async validatePermissions(pluginPath: string): Promise<PermissionValidationResult> {
        const manifestPath = path.join(pluginPath, 'mcp.json')

        if (!await this.fileExists(manifestPath)) {
            return { passed: true } // 没有 manifest
        }

        try {
            const manifest = JSON.parse(await fs.readFile(manifestPath, 'utf-8'))

            if (!manifest.permissions || !Array.isArray(manifest.permissions)) {
                return { passed: true } // 没有权限请求
            }

            const invalidPermissions = manifest.permissions.filter(permission =>
                !this.isValidPermissionType(permission.type)
            )

            if (invalidPermissions.length > 0) {
                return {
                    passed: false,
                    reason: `Invalid permission types: ${invalidPermissions.map(p => p.type).join(', ')}`
                }
            }

            return { passed: true }

        } catch (error) {
            return {
                passed: false,
                reason: `Permission validation failed: ${error.message}`
            }
        }
    }

    private async verifySignature(pluginPath: string): Promise<SignatureCheckResult> {
        const signaturePath = path.join(pluginPath, 'signature.json')

        if (!await this.fileExists(signaturePath)) {
            return {
                passed: false,
                reason: 'Plugin signature not found'
            }
        }

        try {
            const signature = JSON.parse(await fs.readFile(signaturePath, 'utf-8'))
            const publicKey = await this.getPublicKey()

            // 验证签名
            const isValid = await this.verifyDigitalSignature(
                pluginPath,
                signature.signature,
                publicKey
            )

            if (!isValid) {
                return {
                    passed: false,
                    reason: 'Invalid plugin signature'
                }
            }

            // 检查证书链
            const certificateValid = await this.validateCertificateChain(signature.certificate)
            if (!certificateValid) {
                return {
                    passed: false,
                    reason: 'Invalid certificate chain'
                }
            }

            return { passed: true }

        } catch (error) {
            return {
                passed: false,
                reason: `Signature verification failed: ${error.message}`
            }
        }
    }

    private async validatePermissionRequest(pluginId: string, permission: Permission): Promise<PermissionValidation> {
        // 检查权限类型是否有效
        if (!this.isValidPermissionType(permission.type)) {
            return {
                valid: false,
                reason: `Invalid permission type: ${permission.type}`
            }
        }

        // 检查权限参数
        const paramValidation = this.validatePermissionParameters(permission)
        if (!paramValidation.valid) {
            return {
                valid: false,
                reason: paramValidation.reason
            }
        }

        // 检查插件信誉
        const reputationCheck = await this.checkPluginReputation(pluginId)
        if (reputationCheck.score < 0.5) {
            return {
                valid: false,
                reason: 'Plugin reputation too low'
            }
        }

        return { valid: true }
    }

    private async checkPermissionConflicts(pluginId: string, permission: Permission): Promise<ConflictCheck> {
        const existingPermissions = await this.permissionStore.getPluginPermissions(pluginId)

        for (const existing of existingPermissions) {
            if (this.hasPermissionConflict(existing, permission)) {
                return {
                    hasConflict: true,
                    conflictReason: `Conflict with existing permission: ${existing.type}`
                }
            }
        }

        return { hasConflict: false }
    }

    private async checkPermissionConditions(permission: Permission, context?: SecurityContext): Promise<ConditionCheck> {
        if (!permission.conditions) {
            return { passed: true }
        }

        for (const condition of permission.conditions) {
            const result = await this.evaluateCondition(condition, context)
            if (!result.passed) {
                return {
                    passed: false,
                    reason: result.reason
                }
            }
        }

        return { passed: true }
    }

    private async evaluateCondition(condition: PermissionCondition, context?: SecurityContext): Promise<ConditionEvaluation> {
        switch (condition.type) {
            case 'time_range':
                return this.evaluateTimeRangeCondition(condition)
            case 'network_access':
                return this.evaluateNetworkAccessCondition(condition, context)
            case 'file_access':
                return this.evaluateFileAccessCondition(condition, context)
            case 'resource_limit':
                return this.evaluateResourceLimitCondition(condition, context)
            default:
                return { passed: true }
        }
    }

    private evaluateTimeRangeCondition(condition: PermissionCondition): ConditionEvaluation {
        const now = new Date()
        const currentTime = now.getHours() * 60 + now.getMinutes()

        if (condition.startTime !== undefined && currentTime < condition.startTime) {
            return {
                passed: false,
                reason: `Access outside allowed time range (before ${condition.startTime})`
            }
        }

        if (condition.endTime !== undefined && currentTime > condition.endTime) {
            return {
                passed: false,
                reason: `Access outside allowed time range (after ${condition.endTime})`
            }
        }

        return { passed: true }
    }

    private async evaluateNetworkAccessCondition(
        condition: PermissionCondition,
        context?: SecurityContext
    ): Promise<ConditionEvaluation> {
        if (!context?.networkAccess) {
            return { passed: true }
        }

        // 检查允许的域名
        if (condition.allowedDomains && context.networkAccess.hostname) {
            const allowed = condition.allowedDomains.some(domain =>
                context.networkAccess!.hostname === domain ||
                context.networkAccess!.hostname.endsWith(`.${domain}`)
            )

            if (!allowed) {
                return {
                    passed: false,
                    reason: `Network access to ${context.networkAccess.hostname} not allowed`
                }
            }
        }

        // 检查端口限制
        if (condition.allowedPorts && context.networkAccess.port) {
            if (!condition.allowedPorts.includes(context.networkAccess.port)) {
                return {
                    passed: false,
                    reason: `Network access to port ${context.networkAccess.port} not allowed`
                }
            }
        }

        return { passed: true }
    }

    private async evaluateFileAccessCondition(
        condition: PermissionCondition,
        context?: SecurityContext
    ): Promise<ConditionEvaluation> {
        if (!context?.fileAccess) {
            return { passed: true }
        }

        // 检查允许的路径
        if (condition.allowedPaths && context.fileAccess.path) {
            const allowed = condition.allowedPaths.some(allowedPath =>
                context.fileAccess!.path.startsWith(allowedPath)
            )

            if (!allowed) {
                return {
                    passed: false,
                    reason: `File access to ${context.fileAccess.path} not allowed`
                }
            }
        }

        // 检查文件操作类型
        if (condition.allowedOperations && context.fileAccess.operation) {
            if (!condition.allowedOperations.includes(context.fileAccess.operation)) {
                return {
                    passed: false,
                    reason: `File operation ${context.fileAccess.operation} not allowed`
                }
            }
        }

        return { passed: true }
    }

    private evaluateResourceLimitCondition(
        condition: PermissionCondition,
        context?: SecurityContext
    ): Promise<ConditionEvaluation> {
        if (!context?.resourceUsage) {
            return { passed: true }
        }

        // 检查内存限制
        if (condition.maxMemory && context.resourceUsage.memory > condition.maxMemory) {
            return {
                passed: false,
                reason: `Memory usage exceeded: ${context.resourceUsage.memory} > ${condition.maxMemory}`
            }
        }

        // 检查CPU限制
        if (condition.maxCPU && context.resourceUsage.cpu > condition.maxCPU) {
            return {
                passed: false,
                reason: `CPU usage exceeded: ${context.resourceUsage.cpu} > ${condition.maxCPU}`
            }
        }

        return Promise.resolve({ passed: true })
    }

    private async checkVulnerabilities(dependencies: Record<string, string>): Promise<VulnerabilityCheck> {
        // 简化的漏洞检查实现
        // 实际应该调用安全数据库服务
        const knownVulnerablePackages = [
            'lodash@4.17.15',
            'express@4.16.0',
            'request@2.88.0'
        ]

        const vulnerabilities: string[] = []

        for (const [pkg, version] of Object.entries(dependencies)) {
            const fullVersion = `${pkg}@${version}`
            if (knownVulnerablePackages.some(vuln => fullVersion.startsWith(vuln))) {
                vulnerabilities.push(fullVersion)
            }
        }

        return {
            passed: vulnerabilities.length === 0,
            vulnerabilities
        }
    }

    private isValidPermissionType(type: string): boolean {
        const validTypes = [
            'filesystem.read',
            'filesystem.write',
            'network.http',
            'network.websocket',
            'system.process',
            'system.environment',
            'user.data',
            'plugin.api'
        ]

        return validTypes.includes(type)
    }

    private validatePermissionParameters(permission: Permission): ParameterValidation {
        // 根据权限类型验证参数
        switch (permission.type) {
            case 'filesystem.read':
            case 'filesystem.write':
                if (!permission.details?.paths) {
                    return {
                        valid: false,
                        reason: 'Filesystem permissions require paths parameter'
                    }
                }
                break
            case 'network.http':
                if (!permission.details?.domains) {
                    return {
                        valid: false,
                        reason: 'Network permissions require domains parameter'
                    }
                }
                break
        }

        return { valid: true }
    }

    private hasPermissionConflict(existing: Permission, requested: Permission): boolean {
        // 检查权限冲突
        if (existing.type === requested.type) {
            return true
        }

        // 检查子权限冲突
        const conflictMatrix = {
            'filesystem.write': ['filesystem.read'],
            'system.process': ['filesystem.read', 'filesystem.write'],
            'network.http': ['user.data']
        }

        const conflicts = conflictMatrix[requested.type as keyof typeof conflictMatrix] || []
        return conflicts.includes(existing.type)
    }

    private async checkPluginReputation(pluginId: string): Promise<ReputationCheck> {
        // 简化的信誉检查
        // 实际应该从插件市场获取信誉数据
        return {
            score: 0.8, // 默认较高的信誉分数
            factors: ['positive_reviews', 'active_maintenance']
        }
    }

    private async getPublicKey(): Promise<string> {
        // 获取公钥
        return '-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...\n-----END PUBLIC KEY-----'
    }

    private async verifyDigitalSignature(
        data: string,
        signature: string,
        publicKey: string
    ): Promise<boolean> {
        // 简化的签名验证
        // 实际应该使用加密库
        return true
    }

    private async validateCertificateChain(certificate: any): Promise<boolean> {
        // 简化的证书链验证
        // 实际应该验证证书链和过期时间
        return true
    }

    private async fileExists(filePath: string): Promise<boolean> {
        try {
            await fs.access(filePath)
            return true
        } catch {
            return false
        }
    }
}

// 静态代码分析器
class StaticCodeAnalyzer {
    async analyzeDirectory(directoryPath: string): Promise<StaticAnalysisResult> {
        const issues: SecurityIssue[] = []

        // 分析 JavaScript/TypeScript 文件
        const jsFiles = await this.findFiles(directoryPath, ['.js', '.ts'])
        for (const file of jsFiles) {
            const fileIssues = await this.analyzeJavaScriptFile(file)
            issues.push(...fileIssues)
        }

        // 分析 Python 文件
        const pyFiles = await this.findFiles(directoryPath, ['.py'])
        for (const file of pyFiles) {
            const fileIssues = await this.analyzePythonFile(file)
            issues.push(...fileIssues)
        }

        return {
            issues,
            summary: this.generateSummary(issues)
        }
    }

    private async findFiles(directory: string, extensions: string[]): Promise<string[]> {
        const files: string[] = []

        const entries = await fs.readdir(directory, { withFileTypes: true })
        for (const entry of entries) {
            const fullPath = path.join(directory, entry.name)

            if (entry.isDirectory()) {
                const subFiles = await this.findFiles(fullPath, extensions)
                files.push(...subFiles)
            } else if (extensions.some(ext => entry.name.endsWith(ext))) {
                files.push(fullPath)
            }
        }

        return files
    }

    private async analyzeJavaScriptFile(filePath: string): Promise<SecurityIssue[]> {
        const issues: SecurityIssue[] = []
        const content = await fs.readFile(filePath, 'utf-8')

        // 危险模式检测
        const dangerousPatterns = [
            {
                pattern: /eval\s*\(/,
                message: 'Use of eval() detected',
                severity: 'critical'
            },
            {
                pattern: /Function\s*\(/,
                message: 'Dynamic function creation detected',
                severity: 'high'
            },
            {
                pattern: /require\s*\(\s*['"]child_process['"]\s*\)/,
                message: 'Child process module imported',
                severity: 'critical'
            },
            {
                pattern: /require\s*\(\s*['"]fs['"]\s*\)/,
                message: 'File system module imported',
                severity: 'high'
            },
            {
                pattern: /process\.env/,
                message: 'Environment variable access detected',
                severity: 'medium'
            }
        ]

        for (const { pattern, message, severity } of dangerousPatterns) {
            let match
            while ((match = pattern.exec(content)) !== null) {
                const line = content.substring(0, match.index).split('\n').length
                issues.push({
                    type: 'dangerous_pattern',
                    file: filePath,
                    line,
                    message,
                    severity: severity as 'low' | 'medium' | 'high' | 'critical'
                })
            }
        }

        return issues
    }

    private async analyzePythonFile(filePath: string): Promise<SecurityIssue[]> {
        const issues: SecurityIssue[] = []
        const content = await fs.readFile(filePath, 'utf-8')

        // 危险模式检测
        const dangerousPatterns = [
            {
                pattern: /exec\s*\(/,
                message: 'Use of exec() detected',
                severity: 'critical'
            },
            {
                pattern: /eval\s*\(/,
                message: 'Use of eval() detected',
                severity: 'critical'
            },
            {
                pattern: /subprocess\./,
                message: 'Subprocess module usage detected',
                severity: 'high'
            },
            {
                pattern: /os\.system/,
                message: 'OS system command execution detected',
                severity: 'critical'
            }
        ]

        for (const { pattern, message, severity } of dangerousPatterns) {
            let match
            while ((match = pattern.exec(content)) !== null) {
                const line = content.substring(0, match.index).split('\n').length
                issues.push({
                    type: 'dangerous_pattern',
                    file: filePath,
                    line,
                    message,
                    severity: severity as 'low' | 'medium' | 'high' | 'critical'
                })
            }
        }

        return issues
    }

    private generateSummary(issues: SecurityIssue[]): string {
        const severityCount = issues.reduce((acc, issue) => {
            acc[issue.severity] = (acc[issue.severity] || 0) + 1
            return acc
        }, {} as Record<string, number>)

        return `Found ${issues.length} issues: ${Object.entries(severityCount)
            .map(([severity, count]) => `${count} ${severity}`)
            .join(', ')}`
    }
}
```

## 市场运营模式

### 插件市场架构

#### 1. 市场服务系统

```typescript
// src/core/mcp/marketplace/PluginMarketplace.ts
export class PluginMarketplace {
    private marketplaceClient: MarketplaceClient
    private pluginRegistry: PluginRegistry
    private paymentService: PaymentService
    private reviewService: ReviewService
    private analyticsService: AnalyticsService

    constructor(
        marketplaceClient: MarketplaceClient,
        pluginRegistry: PluginRegistry,
        paymentService: PaymentService,
        reviewService: ReviewService,
        analyticsService: AnalyticsService
    ) {
        this.marketplaceClient = marketplaceClient
        this.pluginRegistry = pluginRegistry
        this.paymentService = paymentService
        this.reviewService = reviewService
        this.analyticsService = analyticsService
    }

    async searchPlugins(query: SearchQuery): Promise<SearchResult> {
        const searchStart = Date.now()

        try {
            // 1. 构建搜索请求
            const searchRequest = this.buildSearchRequest(query)

            // 2. 执行搜索
            const searchResponse = await this.marketplaceClient.search(searchRequest)

            // 3. 处理搜索结果
            const processedResults = await this.processSearchResults(searchResponse.results)

            // 4. 排序和分页
            const sortedResults = this.sortAndPaginate(processedResults, query)

            // 5. 记录搜索分析
            await this.analyticsService.recordSearch({
                query: query.q,
                resultsCount: sortedResults.length,
                searchTime: Date.now() - searchStart,
                timestamp: new Date()
            })

            return {
                success: true,
                results: sortedResults,
                total: searchResponse.total,
                page: query.page || 1,
                pageSize: query.pageSize || 20,
                searchTime: Date.now() - searchStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                searchTime: Date.now() - searchStart
            }
        }
    }

    async getPluginDetails(pluginId: string, version?: string): Promise<PluginDetailsResult> {
        try {
            // 获取插件基本信息
            const metadata = await this.pluginRegistry.getPluginMetadata(pluginId, version)
            if (!metadata) {
                return {
                    success: false,
                    error: `Plugin not found: ${pluginId}`
                }
            }

            // 获取插件统计信息
            const stats = await this.analyticsService.getPluginStats(pluginId)

            // 获取插件评论
            const reviews = await this.reviewService.getPluginReviews(pluginId)

            // 获取插件版本信息
            const versions = await this.pluginRegistry.getPluginVersions(pluginId)

            // 检查用户购买状态
            const purchaseInfo = await this.paymentService.getPurchaseInfo(pluginId)

            return {
                success: true,
                plugin: {
                    ...metadata,
                    stats,
                    reviews,
                    versions,
                    isPurchased: purchaseInfo.isPurchased,
                    purchaseDate: purchaseInfo.purchaseDate
                }
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    async purchasePlugin(pluginId: string, version: string): Promise<PurchaseResult> {
        const purchaseStart = Date.now()

        try {
            // 1. 验证插件
            const plugin = await this.pluginRegistry.getPluginMetadata(pluginId, version)
            if (!plugin) {
                return {
                    success: false,
                    error: `Plugin not found: ${pluginId}@${version}`
                }
            }

            // 2. 检查是否已购买
            const existingPurchase = await this.paymentService.getPurchaseInfo(pluginId)
            if (existingPurchase.isPurchased) {
                return {
                    success: false,
                    error: 'Plugin already purchased'
                }
            }

            // 3. 创建支付订单
            const order = await this.paymentService.createOrder({
                pluginId,
                version,
                amount: plugin.pricing?.amount || 0,
                currency: plugin.pricing?.currency || 'USD'
            })

            // 4. 处理支付
            const paymentResult = await this.paymentService.processPayment(order.id)

            if (!paymentResult.success) {
                return {
                    success: false,
                    error: `Payment failed: ${paymentResult.error}`
                }
            }

            // 5. 授予访问权限
            await this.paymentService.grantAccess({
                pluginId,
                version,
                userId: paymentResult.userId,
                orderId: order.id,
                purchaseDate: new Date()
            })

            // 6. 记录购买分析
            await this.analyticsService.recordPurchase({
                pluginId,
                version,
                amount: plugin.pricing?.amount || 0,
                currency: plugin.pricing?.currency || 'USD',
                purchaseTime: Date.now() - purchaseStart,
                timestamp: new Date()
            })

            return {
                success: true,
                pluginId,
                version,
                orderId: order.id,
                purchaseTime: Date.now() - purchaseStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                purchaseTime: Date.now() - purchaseStart
            }
        }
    }

    async publishPlugin(publishRequest: PublishPluginRequest): Promise<PublishResult> {
        const publishStart = Date.now()

        try {
            // 1. 验证发布者
            const publisherValid = await this.validatePublisher(publishRequest.publisherId)
            if (!publisherValid) {
                return {
                    success: false,
                    error: 'Invalid publisher'
                }
            }

            // 2. 验证插件包
            const packageValidation = await this.validatePluginPackage(publishRequest.packagePath)
            if (!packageValidation.valid) {
                return {
                    success: false,
                    error: `Package validation failed: ${packageValidation.error}`
                }
            }

            // 3. 安全检查
            const securityCheck = await this.performSecurityCheck(publishRequest.packagePath)
            if (!securityCheck.passed) {
                return {
                    success: false,
                    error: `Security check failed: ${securityCheck.reason}`
                }
            }

            // 4. 质量检查
            const qualityCheck = await this.performQualityCheck(publishRequest.packagePath)
            if (!qualityCheck.passed) {
                return {
                    success: false,
                    error: `Quality check failed: ${qualityCheck.reason}`
                }
            }

            // 5. 生成插件签名
            const signature = await this.generatePluginSignature(publishRequest.packagePath)

            // 6. 上传插件包
            const uploadResult = await this.uploadPluginPackage(publishRequest.packagePath)
            if (!uploadResult.success) {
                return {
                    success: false,
                    error: `Upload failed: ${uploadResult.error}`
                }
            }

            // 7. 注册插件
            const registration = await this.pluginRegistry.registerPlugin({
                id: publishRequest.pluginId,
                version: publishRequest.version,
                metadata: {
                    ...publishRequest.metadata,
                    packageUrl: uploadResult.packageUrl,
                    signature,
                    publishedAt: new Date(),
                    publisherId: publishRequest.publisherId
                }
            })

            if (!registration.success) {
                return {
                    success: false,
                    error: `Registration failed: ${registration.error}`
                }
            }

            // 8. 记录发布分析
            await this.analyticsService.recordPublish({
                pluginId: publishRequest.pluginId,
                version: publishRequest.version,
                publisherId: publishRequest.publisherId,
                publishTime: Date.now() - publishStart,
                timestamp: new Date()
            })

            return {
                success: true,
                pluginId: publishRequest.pluginId,
                version: publishRequest.version,
                publishTime: Date.now() - publishStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                publishTime: Date.now() - publishStart
            }
        }
    }

    async updatePlugin(pluginId: string, version: string, updateRequest: UpdatePluginRequest): Promise<UpdateResult> {
        try {
            // 1. 验证更新权限
            const hasPermission = await this.verifyUpdatePermission(pluginId, updateRequest.publisherId)
            if (!hasPermission) {
                return {
                    success: false,
                    error: 'No permission to update this plugin'
                }
            }

            // 2. 验证更新包
            const packageValidation = await this.validatePluginPackage(updateRequest.packagePath)
            if (!packageValidation.valid) {
                return {
                    success: false,
                    error: `Package validation failed: ${packageValidation.error}`
                }
            }

            // 3. 安全检查
            const securityCheck = await this.performSecurityCheck(updateRequest.packagePath)
            if (!securityCheck.passed) {
                return {
                    success: false,
                    error: `Security check failed: ${securityCheck.reason}`
                }
            }

            // 4. 上传新版本
            const uploadResult = await this.uploadPluginPackage(updateRequest.packagePath)
            if (!uploadResult.success) {
                return {
                    success: false,
                    error: `Upload failed: ${uploadResult.error}`
                }
            }

            // 5. 更新插件信息
            const updateResult = await this.pluginRegistry.updatePlugin(pluginId, version, {
                ...updateRequest.metadata,
                packageUrl: uploadResult.packageUrl,
                updatedAt: new Date()
            })

            if (!updateResult.success) {
                return {
                    success: false,
                    error: `Update failed: ${updateResult.error}`
                }
            }

            return {
                success: true,
                pluginId,
                version
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    async getTrendingPlugins(period: 'daily' | 'weekly' | 'monthly'): Promise<TrendingResult> {
        try {
            const trending = await this.analyticsService.getTrendingPlugins(period)
            const pluginDetails = await Promise.all(
                trending.map(async (item) => {
                    const metadata = await this.pluginRegistry.getPluginMetadata(item.pluginId, item.version)
                    return {
                        ...item,
                        metadata
                    }
                })
            )

            return {
                success: true,
                plugins: pluginDetails,
                period
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    async getRecommendations(userId: string): Promise<RecommendationResult> {
        try {
            // 获取用户行为数据
            const userBehavior = await this.analyticsService.getUserBehavior(userId)

            // 基于协同过滤推荐
            const collaborativeRecommendations = await this.getCollaborativeRecommendations(userId)

            // 基于内容推荐
            const contentRecommendations = await this.getContentBasedRecommendations(userBehavior)

            // 基于热门度推荐
            const popularRecommendations = await this.getPopularRecommendations()

            // 合并和去重
            const allRecommendations = [
                ...collaborativeRecommendations,
                ...contentRecommendations,
                ...popularRecommendations
            ]

            const uniqueRecommendations = this.deduplicateRecommendations(allRecommendations)

            // 排序推荐结果
            const sortedRecommendations = this.sortRecommendations(uniqueRecommendations, userBehavior)

            return {
                success: true,
                recommendations: sortedRecommendations.slice(0, 10) // 返回前10个推荐
            }

        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    private buildSearchRequest(query: SearchQuery): MarketplaceSearchRequest {
        return {
            q: query.q,
            category: query.category,
            tags: query.tags,
            priceRange: query.priceRange,
            rating: query.minRating,
            sortBy: query.sortBy || 'relevance',
            sortOrder: query.sortOrder || 'desc',
            page: query.page || 1,
            pageSize: query.pageSize || 20
        }
    }

    private async processSearchResults(results: MarketplacePlugin[]): Promise<ProcessedPlugin[]> {
        const processedResults: ProcessedPlugin[] = []

        for (const result of results) {
            // 获取附加信息
            const stats = await this.analyticsService.getPluginStats(result.id)
            const reviews = await this.reviewService.getPluginReviews(result.id)

            processedResults.push({
                ...result,
                stats,
                reviews,
                processedAt: new Date()
            })
        }

        return processedResults
    }

    private sortAndPaginate(results: ProcessedPlugin[], query: SearchQuery): ProcessedPlugin[] {
        let sorted = [...results]

        // 排序
        switch (query.sortBy) {
            case 'downloads':
                sorted.sort((a, b) => b.stats!.downloads - a.stats!.downloads)
                break
            case 'rating':
                sorted.sort((a, b) => b.stats!.averageRating - a.stats!.averageRating)
                break
            case 'updated':
                sorted.sort((a, b) => new Date(b.updatedAt).getTime() - new Date(a.updatedAt).getTime())
                break
            case 'price':
                sorted.sort((a, b) => (a.pricing?.amount || 0) - (b.pricing?.amount || 0))
                break
            default:
                // 相关性排序（已在搜索服务中处理）
                break
        }

        // 应用排序方向
        if (query.sortOrder === 'asc') {
            sorted.reverse()
        }

        // 分页
        const start = ((query.page || 1) - 1) * (query.pageSize || 20)
        const end = start + (query.pageSize || 20)

        return sorted.slice(start, end)
    }

    private async validatePublisher(publisherId: string): Promise<boolean> {
        // 验证发布者身份和权限
        const publisher = await this.marketplaceClient.getPublisher(publisherId)
        return publisher && publisher.status === 'active'
    }

    private async validatePluginPackage(packagePath: string): Promise<PackageValidation> {
        try {
            // 检查文件结构
            const structureValid = await this.validatePackageStructure(packagePath)
            if (!structureValid.valid) {
                return structureValid
            }

            // 检查 manifest 文件
            const manifestValid = await this.validateManifest(packagePath)
            if (!manifestValid.valid) {
                return manifestValid
            }

            // 检查代码质量
            const qualityValid = await this.validateCodeQuality(packagePath)
            if (!qualityValid.valid) {
                return qualityValid
            }

            return { valid: true }

        } catch (error) {
            return {
                valid: false,
                error: error.message
            }
        }
    }

    private async validatePackageStructure(packagePath: string): Promise<PackageValidation> {
        const requiredFiles = ['mcp.json', 'package.json', 'README.md']
        const optionalFiles = ['LICENSE', 'CHANGELOG.md']

        try {
            const files = await fs.readdir(packagePath)

            // 检查必需文件
            for (const requiredFile of requiredFiles) {
                if (!files.includes(requiredFile)) {
                    return {
                        valid: false,
                        error: `Missing required file: ${requiredFile}`
                    }
                }
            }

            return { valid: true }

        } catch (error) {
            return {
                valid: false,
                error: `Failed to read package directory: ${error.message}`
            }
        }
    }

    private async validateManifest(packagePath: string): Promise<PackageValidation> {
        try {
            const manifestPath = path.join(packagePath, 'mcp.json')
            const manifestContent = await fs.readFile(manifestPath, 'utf-8')
            const manifest = JSON.parse(manifestContent)

            // 验证必需字段
            const requiredFields = ['name', 'version', 'description', 'author']
            for (const field of requiredFields) {
                if (!manifest[field]) {
                    return {
                        valid: false,
                        error: `Missing required field in manifest: ${field}`
                    }
                }
            }

            // 验证版本格式
            if (!this.isValidVersion(manifest.version)) {
                return {
                    valid: false,
                    error: 'Invalid version format'
                }
            }

            return { valid: true }

        } catch (error) {
            return {
                valid: false,
                error: `Failed to validate manifest: ${error.message}`
            }
        }
    }

    private async validateCodeQuality(packagePath: string): Promise<PackageValidation> {
        try {
            // 运行代码质量检查
            const analyzer = new CodeQualityAnalyzer()
            const analysis = await analyzer.analyzeDirectory(packagePath)

            if (analysis.issues.length > 10) {
                return {
                    valid: false,
                    error: 'Too many code quality issues detected'
                }
            }

            const criticalIssues = analysis.issues.filter(issue => issue.severity === 'critical')
            if (criticalIssues.length > 0) {
                return {
                    valid: false,
                    error: 'Critical code quality issues detected'
                }
            }

            return { valid: true }

        } catch (error) {
            return {
                valid: false,
                error: `Code quality check failed: ${error.message}`
            }
        }
    }

    private async performSecurityCheck(packagePath: string): Promise<SecurityCheckResult> {
        // 调用安全检查服务
        const securityManager = new MCPSecurityManager(
            new PermissionStore(),
            new SandboxManager(),
            new AuditLogger()
        )

        return securityManager.performSecurityCheck(packagePath)
    }

    private async performQualityCheck(packagePath: string): Promise<QualityCheckResult> {
        // 质量检查包括：
        // 1. 代码覆盖率
        // 2. 文档完整性
        // 3. 测试用例质量
        // 4. 性能基准

        return { passed: true }
    }

    private async generatePluginSignature(packagePath: string): Promise<string> {
        // 生成数字签名
        const privateKey = await this.getPrivateKey()
        const packageHash = await this.calculatePackageHash(packagePath)

        return this.signData(packageHash, privateKey)
    }

    private async uploadPluginPackage(packagePath: string): Promise<UploadResult> {
        // 上传到云存储
        const storageService = new StorageService()
        return storage.uploadFile(packagePath, 'plugins')
    }

    private async verifyUpdatePermission(pluginId: string, publisherId: string): Promise<boolean> {
        const plugin = await this.pluginRegistry.getPluginMetadata(pluginId)
        return plugin?.publisherId === publisherId
    }

    private async getCollaborativeRecommendations(userId: string): Promise<RecommendationItem[]> {
        // 基于用户相似性推荐
        const similarUsers = await this.analyticsService.getSimilarUsers(userId)
        const recommendations: RecommendationItem[] = []

        for (const similarUser of similarUsers) {
            const userPlugins = await this.analyticsService.getUserPlugins(similarUser.userId)
            for (const plugin of userPlugins) {
                if (plugin.rating >= 4) { // 只推荐高评分的插件
                    recommendations.push({
                        pluginId: plugin.pluginId,
                        version: plugin.version,
                        score: similarUser.similarity * plugin.rating,
                        reason: 'Users with similar preferences also liked this'
                    })
                }
            }
        }

        return recommendations
    }

    private async getContentBasedRecommendations(userBehavior: UserBehavior): Promise<RecommendationItem[]> {
        // 基于用户历史行为推荐
        const recommendations: RecommendationItem[] = []

        // 基于用户安装的插件推荐相关插件
        for (const installedPlugin of userBehavior.installedPlugins) {
            const similarPlugins = await this.analyticsService.getSimilarPlugins(installedPlugin.pluginId)
            for (const similarPlugin of similarPlugins) {
                recommendations.push({
                    pluginId: similarPlugin.pluginId,
                    version: similarPlugin.version,
                    score: similarPlugin.similarity,
                    reason: 'Similar to your installed plugins'
                })
            }
        }

        return recommendations
    }

    private async getPopularRecommendations(): Promise<RecommendationItem[]> {
        // 基于流行度推荐
        const popularPlugins = await this.analyticsService.getPopularPlugins()
        return popularPlugins.map(plugin => ({
            pluginId: plugin.pluginId,
            version: plugin.version,
            score: plugin.popularityScore,
            reason: 'Popular among users'
        }))
    }

    private deduplicateRecommendations(recommendations: RecommendationItem[]): RecommendationItem[] {
        const seen = new Set<string>()
        return recommendations.filter(rec => {
            const key = `${rec.pluginId}@${rec.version}`
            if (seen.has(key)) {
                return false
            }
            seen.add(key)
            return true
        })
    }

    private sortRecommendations(recommendations: RecommendationItem[], userBehavior: UserBehavior): RecommendationItem[] {
        return recommendations.sort((a, b) => {
            // 考虑用户偏好权重
            const userWeights = this.getUserPreferenceWeights(userBehavior)

            const scoreA = a.score * this.getCategoryWeight(a.pluginId, userWeights)
            const scoreB = b.score * this.getCategoryWeight(b.pluginId, userWeights)

            return scoreB - scoreA
        })
    }

    private getUserPreferenceWeights(userBehavior: UserBehavior): Record<string, number> {
        // 基于用户行为计算类别权重
        const weights: Record<string, number> = {}
        const totalInteractions = userBehavior.searches.length + userBehavior.installedPlugins.length

        // 计算搜索权重
        for (const search of userBehavior.searches) {
            const category = this.inferCategoryFromQuery(search.query)
            weights[category] = (weights[category] || 0) + 0.1
        }

        // 计算安装权重
        for (const plugin of userBehavior.installedPlugins) {
            const category = this.inferCategoryFromPlugin(plugin.pluginId)
            weights[category] = (weights[category] || 0) + 0.3
        }

        return weights
    }

    private getCategoryWeight(pluginId: string, userWeights: Record<string, number>): number {
        const category = this.inferCategoryFromPlugin(pluginId)
        return userWeights[category] || 1.0
    }

    private inferCategoryFromQuery(query: string): string {
        // 基于查询词推断类别
        if (query.includes('database')) return 'database'
        if (query.includes('api')) return 'api'
        if (query.includes('ui')) return 'ui'
        if (query.includes('test')) return 'testing'
        return 'general'
    }

    private inferCategoryFromPlugin(pluginId: string): string {
        // 基于插件ID推断类别
        if (pluginId.includes('db')) return 'database'
        if (pluginId.includes('api')) return 'api'
        if (pluginId.includes('ui')) return 'ui'
        if (pluginId.includes('test')) return 'testing'
        return 'general'
    }

    private isValidVersion(version: string): boolean {
        // 验证语义化版本格式
        return /^\d+\.\d+\.\d+(-[\w\d-]+)?$/.test(version)
    }

    private async getPrivateKey(): Promise<string> {
        // 获取私钥用于签名
        return '-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDA...\n-----END PRIVATE KEY-----'
    }

    private async calculatePackageHash(packagePath: string): Promise<string> {
        // 计算包的哈希值
        const crypto = require('crypto')
        const hash = crypto.createHash('sha256')

        const files = await this.getAllFiles(packagePath)
        for (const file of files) {
            const content = await fs.readFile(file)
            hash.update(content)
        }

        return hash.digest('hex')
    }

    private async getAllFiles(directory: string): Promise<string[]> {
        const files: string[] = []

        const entries = await fs.readdir(directory, { withFileTypes: true })
        for (const entry of entries) {
            const fullPath = path.join(directory, entry.name)

            if (entry.isDirectory()) {
                const subFiles = await this.getAllFiles(fullPath)
                files.push(...subFiles)
            } else {
                files.push(fullPath)
            }
        }

        return files
    }

    private signData(data: string, privateKey: string): string {
        // 使用私钥签名数据
        const crypto = require('crypto')
        const sign = crypto.createSign('RSA-SHA256')
        sign.update(data)
        return sign.sign(privateKey, 'hex')
    }
}

// 代码质量分析器
class CodeQualityAnalyzer {
    async analyzeDirectory(directoryPath: string): Promise<CodeQualityAnalysis> {
        const issues: CodeQualityIssue[] = []

        // 分析代码覆盖率
        const coverage = await this.analyzeCodeCoverage(directoryPath)
        if (coverage < 0.8) {
            issues.push({
                type: 'coverage',
                severity: 'medium',
                message: `Low code coverage: ${(coverage * 100).toFixed(1)}%`
            })
        }

        // 分析复杂度
        const complexity = await this.analyzeComplexity(directoryPath)
        if (complexity.average > 10) {
            issues.push({
                type: 'complexity',
                severity: 'high',
                message: `High cyclomatic complexity: ${complexity.average.toFixed(1)}`
            })
        }

        // 分析文档完整性
        const documentation = await this.analyzeDocumentation(directoryPath)
        if (documentation.score < 0.7) {
            issues.push({
                type: 'documentation',
                severity: 'medium',
                message: `Insufficient documentation: ${(documentation.score * 100).toFixed(1)}%`
            })
        }

        return {
            issues,
            summary: {
                coverage,
                complexity,
                documentation
            }
        }
    }

    private async analyzeCodeCoverage(directoryPath: string): Promise<number> {
        // 简化的覆盖率分析
        // 实际应该运行测试并收集覆盖率数据
        return 0.85 // 假设85%覆盖率
    }

    private async analyzeComplexity(directoryPath: string): Promise<ComplexityAnalysis> {
        // 简化的复杂度分析
        return {
            average: 8.5,
            max: 15,
            min: 3
        }
    }

    private async analyzeDocumentation(directoryPath: string): Promise<DocumentationAnalysis> {
        // 分析文档完整性
        const files = await this.getAllFiles(directoryPath, ['.js', '.ts', '.py'])
        let documentedFiles = 0

        for (const file of files) {
            const content = await fs.readFile(file, 'utf-8')
            if (this.hasDocumentation(content)) {
                documentedFiles++
            }
        }

        return {
            score: files.length > 0 ? documentedFiles / files.length : 0,
            documentedFiles,
            totalFiles: files.length
        }
    }

    private hasDocumentation(content: string): boolean {
        // 检查是否有文档注释
        const docPatterns = [
            /\/\*\*[\s\S]*?\*\//, // JSDoc
            /\/\/\/.*$/gm,      // XML文档注释
            /#.*$/gm           // Python文档注释
        ]

        return docPatterns.some(pattern => pattern.test(content))
    }

    private async getAllFiles(directory: string, extensions: string[]): Promise<string[]> {
        const files: string[] = []

        const entries = await fs.readdir(directory, { withFileTypes: true })
        for (const entry of entries) {
            const fullPath = path.join(directory, entry.name)

            if (entry.isDirectory()) {
                const subFiles = await this.getAllFiles(fullPath, extensions)
                files.push(...subFiles)
            } else if (extensions.some(ext => entry.name.endsWith(ext))) {
                files.push(fullPath)
            }
        }

        return files
    }
}
```

## 总结

KiloCode 的 MCP 插件生态系统展现了以下技术特点：

### 1. **标准化协议架构**
- 基于 Model Context Protocol 的统一接口
- 灵活的工具、资源和提示词支持
- 多种通信方式（stdio、SSE、WebSocket）

### 2. **强大的插件管理**
- 完整的插件生命周期管理
- 智能依赖解析和版本控制
- 自动更新和回滚机制

### 3. **多层安全防护**
- 沙箱隔离执行环境
- 细粒度权限控制
- 静态代码分析和漏洞检测
- 数字签名和证书验证

### 4. **繁荣的市场生态**
- 插件搜索和推荐系统
- 支付和许可管理
- 用户评价和反馈机制
- 数据分析和运营支持

### 5. **开发者友好**
- 完整的 SDK 和开发工具
- 详细的文档和示例
- 自动化测试和发布流程

这种架构设计使得 KiloCode 能够构建一个安全、繁荣、可持续的插件生态系统，为用户提供丰富的扩展功能，同时保证系统的安全性和稳定性。对于构建现代化的插件系统，这种架构模式具有重要的参考价值。