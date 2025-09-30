# KiloCode Multi Mode 架构深度分析

## 系统架构概览

KiloCode 的 Multi Mode 系统是一个创新的 AI 助手角色架构，通过不同的专业模式和角色分工，为用户提供更精准、更高效的 AI 辅助体验。系统采用分层设计，支持内置模式、自定义模式以及复杂的模式协作机制。

## 1. 核心架构设计

### 1.1 模式定义和类型系统

```typescript
// packages/types/src/mode.ts
export interface Mode {
    slug: string
    name: string
    role: string
    groupName?: string
    description: string
    customInstructions?: string
    tools: ToolGroup[]
    icon?: string
    isCustom?: boolean
    apiBase?: string
    apiModel?: string
    apiProvider?: string
    systemPrompt?: string
    highlightRange?: [number, number]
    experimentalFeatures?: ExperimentalFeatures
    // 自定义模式扩展字段
    allowedFilePaths?: string[]
    disallowedFilePaths?: string[]
    customRules?: string[]
    variables?: Record<string, any>
}

export interface ToolGroup {
    name: string
    tools: ToolName[]
}

export type ToolName =
    | "read"
    | "edit"
    | "browser"
    | "command"
    | "mcp"
    | "modes" // 模式切换工具总是可用

export interface ExperimentalFeatures {
    enableExperimentalFeatures?: boolean
    enableCodeAnalysis?: boolean
    enableAdvancedMode?: boolean
    enableAutoMode?: boolean
    enableOrchestration?: boolean
}
```

### 1.2 五大内置模式定义

```typescript
// src/shared/modes.ts
export const BUILTIN_MODES: Mode[] = [
    {
        slug: "architect",
        name: "Architect",
        role: "You are a technical leader focused on planning and design",
        description: "Focuses on planning, designing, and strategizing before implementation",
        customInstructions: `As the Architect, your primary responsibilities include:
1. Understanding requirements and analyzing constraints
2. Creating comprehensive plans and designs
3. Making strategic technical decisions
4. Coordinating implementation phases
5. Ensuring scalability and maintainability`,
        tools: ["read", "browser", "mcp"],
        icon: "codicon-type-hierarchy-sub",
        highlightRange: [0, 1],
        experimentalFeatures: {
            enableExperimentalFeatures: true,
            enableAdvancedMode: true,
            enableOrchestration: true
        }
    },
    {
        slug: "code",
        name: "Code",
        role: "You are a software engineer with extensive programming knowledge",
        description: "Focuses on writing, modifying, and refactoring code",
        customInstructions: `As the Code specialist, your primary responsibilities include:
1. Writing clean, efficient, and well-documented code
2. Debugging and fixing issues
3. Refactoring and optimizing existing code
4. Following best practices and coding standards
5. Ensuring code quality and test coverage`,
        tools: ["read", "edit", "browser", "command", "mcp"],
        icon: "codicon-code",
        highlightRange: [2, 3],
        experimentalFeatures: {
            enableCodeAnalysis: true,
            enableAutoMode: true
        }
    },
    {
        slug: "ask",
        name: "Ask",
        role: "You are a technical assistant for questions and information",
        description: "Focuses on explanations, documentation, and technical Q&A",
        customInstructions: `As the Ask specialist, your primary responsibilities include:
1. Providing clear and accurate technical explanations
2. Creating comprehensive documentation
3. Answering technical questions thoroughly
4. Researching and synthesizing information
5. Making complex concepts accessible`,
        tools: ["read", "browser", "mcp"],
        icon: "codicon-question",
        highlightRange: [4, 5]
    },
    {
        slug: "debug",
        name: "Debug",
        role: "You are an expert software debugger",
        description: "Focuses on systematic problem diagnosis and resolution",
        customInstructions: `As the Debug specialist, your primary responsibilities include:
1. Analyzing error messages and stack traces
2. Identifying root causes of issues
3. Developing systematic debugging strategies
4. Implementing and testing fixes
5. Preventing future recurrence of similar issues`,
        tools: ["read", "edit", "browser", "command", "mcp"],
        icon: "codicon-bug",
        highlightRange: [6, 7],
        experimentalFeatures: {
            enableCodeAnalysis: true,
            enableAdvancedMode: true
        }
    },
    {
        slug: "orchestrator",
        name: "Orchestrator",
        role: "You are a strategic workflow coordinator",
        description: "Coordinates complex tasks across multiple specialized modes",
        customInstructions: `As the Orchestrator, your primary responsibilities include:
1. Analyzing complex requirements and breaking them into manageable tasks
2. Delegating specialized tasks to appropriate expert modes
3. Coordinating the workflow between different specialists
4. Synthesizing results and ensuring overall coherence
5. Managing timelines and dependencies`,
        tools: [],
        icon: "codicon-run-all",
        highlightRange: [8, 9],
        experimentalFeatures: {
            enableExperimentalFeatures: true,
            enableAdvancedMode: true,
            enableOrchestration: true
        }
    }
]
```

## 2. 模式管理系统

### 2.1 模式管理器核心实现

```typescript
// src/core/config/CustomModesManager.ts
export class CustomModesManager {
    private context: vscode.ExtensionContext
    private globalStateKey: string = "kilocode.customModes"
    private projectModeFile: string = ".kilocodemodes"
    private fileWatcher: vscode.FileSystemWatcher | null = null
    private modeCache: Map<string, Mode[]> = new Map()
    private schemaValidator: z.ZodSchema<Mode>

    constructor(context: vscode.ExtensionContext) {
        this.context = context
        this.initializeSchemaValidator()
        this.initializeFileWatcher()
    }

    private initializeSchemaValidator(): void {
        this.schemaValidator = z.object({
            slug: z.string().min(1).max(50),
            name: z.string().min(1).max(100),
            role: z.string().min(1).max(1000),
            description: z.string().min(1).max(500),
            customInstructions: z.string().optional(),
            tools: z.array(z.enum(["read", "edit", "browser", "command", "mcp"])),
            icon: z.string().optional(),
            isCustom: z.boolean().default(true),
            apiBase: z.string().url().optional(),
            apiModel: z.string().optional(),
            apiProvider: z.string().optional(),
            systemPrompt: z.string().optional(),
            allowedFilePaths: z.array(z.string()).optional(),
            disallowedFilePaths: z.array(z.string()).optional(),
            customRules: z.array(z.string()).optional(),
            variables: z.record(z.any()).optional()
        })
    }

    async getAllModes(): Promise<Mode[]> {
        // 1. 获取内置模式
        const builtinModes = [...BUILTIN_MODES]

        // 2. 获取全局自定义模式
        const globalCustomModes = await this.getGlobalCustomModes()

        // 3. 获取项目级自定义模式
        const projectCustomModes = await this.getProjectCustomModes()

        // 4. 合并并去重
        const allModes = [...builtinModes, ...globalCustomModes, ...projectCustomModes]
        return this.deduplicateModes(allModes)
    }

    async getProjectCustomModes(): Promise<Mode[]> {
        const workspaceFolders = vscode.workspace.workspaceFolders
        if (!workspaceFolders || workspaceFolders.length === 0) {
            return []
        }

        const projectRoot = workspaceFolders[0].uri.fsPath
        const modeFilePath = path.join(projectRoot, this.projectModeFile)
        const cacheKey = `project:${projectRoot}`

        // 检查缓存
        if (this.modeCache.has(cacheKey)) {
            return this.modeCache.get(cacheKey)!
        }

        try {
            const content = await fs.readFile(modeFilePath, 'utf-8')
            const modes = await this.parseModesFile(content, modeFilePath)
            this.modeCache.set(cacheKey, modes)
            return modes
        } catch (error) {
            if (error.code !== 'ENOENT') {
                console.warn(`Failed to load project modes: ${error.message}`)
            }
            return []
        }
    }

    async getGlobalCustomModes(): Promise<Mode[]> {
        const cacheKey = 'global'

        // 检查缓存
        if (this.modeCache.has(cacheKey)) {
            return this.modeCache.get(cacheKey)!
        }

        try {
            const globalModesData = this.context.globalState.get<Mode[]>(this.globalStateKey, [])
            this.modeCache.set(cacheKey, globalModesData)
            return globalModesData
        } catch (error) {
            console.warn(`Failed to load global modes: ${error.message}`)
            return []
        }
    }

    private async parseModesFile(content: string, filePath: string): Promise<Mode[]> {
        try {
            const data = JSON.parse(content)

            if (!Array.isArray(data)) {
                throw new Error('Modes file must contain an array of mode definitions')
            }

            const modes: Mode[] = []

            for (const modeData of data) {
                try {
                    // 验证模式数据
                    const validatedMode = this.schemaValidator.parse(modeData)

                    // 添加额外字段
                    const mode: Mode = {
                        ...validatedMode,
                        isCustom: true,
                        source: 'project',
                        filePath,
                        createdAt: new Date(),
                        updatedAt: new Date()
                    }

                    modes.push(mode)
                } catch (validationError) {
                    console.warn(`Invalid mode definition in ${filePath}: ${validationError.message}`)
                }
            }

            return modes
        } catch (error) {
            throw new Error(`Failed to parse modes file: ${error.message}`)
        }
    }

    async createCustomMode(mode: Omit<Mode, 'isCustom' | 'createdAt' | 'updatedAt'>): Promise<Mode> {
        // 验证模式数据
        const validatedMode = this.schemaValidator.parse(mode)

        const newMode: Mode = {
            ...validatedMode,
            isCustom: true,
            createdAt: new Date(),
            updatedAt: new Date()
        }

        if (mode.source === 'project') {
            await this.addProjectMode(newMode)
        } else {
            await this.addGlobalMode(newMode)
        }

        return newMode
    }

    async updateCustomMode(modeSlug: string, updates: Partial<Mode>): Promise<Mode> {
        const existingMode = await this.findModeBySlug(modeSlug)
        if (!existingMode) {
            throw new Error(`Mode not found: ${modeSlug}`)
        }

        const updatedMode: Mode = {
            ...existingMode,
            ...updates,
            updatedAt: new Date()
        }

        // 验证更新后的模式
        this.schemaValidator.parse(updatedMode)

        if (existingMode.source === 'project') {
            await this.updateProjectMode(updatedMode)
        } else {
            await this.updateGlobalMode(updatedMode)
        }

        return updatedMode
    }

    async deleteCustomMode(modeSlug: string): Promise<void> {
        const mode = await this.findModeBySlug(modeSlug)
        if (!mode) {
            throw new Error(`Mode not found: ${modeSlug}`)
        }

        if (mode.source === 'project') {
            await this.deleteProjectMode(modeSlug)
        } else {
            await this.deleteGlobalMode(modeSlug)
        }
    }

    private async findModeBySlug(slug: string): Promise<Mode | undefined> {
        const allModes = await this.getAllModes()
        return allModes.find(mode => mode.slug === slug)
    }

    private async addProjectMode(mode: Mode): Promise<void> {
        const projectModes = await this.getProjectCustomModes()
        projectModes.push(mode)
        await this.saveProjectModes(projectModes)
    }

    private async addGlobalMode(mode: Mode): Promise<void> {
        const globalModes = await this.getGlobalCustomModes()
        globalModes.push(mode)
        await this.saveGlobalModes(globalModes)
    }

    private async saveProjectModes(modes: Mode[]): Promise<void> {
        const workspaceFolders = vscode.workspace.workspaceFolders
        if (!workspaceFolders || workspaceFolders.length === 0) {
            throw new Error('No workspace folder found')
        }

        const projectRoot = workspaceFolders[0].uri.fsPath
        const modeFilePath = path.join(projectRoot, this.projectModeFile)

        // 序列化并保存
        const content = JSON.stringify(modes, null, 2)
        await fs.writeFile(modeFilePath, content, 'utf-8')

        // 更新缓存
        const cacheKey = `project:${projectRoot}`
        this.modeCache.set(cacheKey, modes)
    }

    private async saveGlobalModes(modes: Mode[]): Promise<void> {
        await this.context.globalState.update(this.globalStateKey, modes)

        // 更新缓存
        const cacheKey = 'global'
        this.modeCache.set(cacheKey, modes)
    }

    private initializeFileWatcher(): void {
        const workspaceFolders = vscode.workspace.workspaceFolders
        if (!workspaceFolders || workspaceFolders.length === 0) {
            return
        }

        const projectRoot = workspaceFolders[0].uri.fsPath
        const modeFilePath = path.join(projectRoot, this.projectModeFile)

        this.fileWatcher = vscode.workspace.createFileSystemWatcher(
            new vscode.RelativePattern(workspaceFolders[0], '.kilocodemodes')
        )

        this.fileWatcher.onDidChange(() => {
            this.invalidateProjectCache(projectRoot)
        })

        this.fileWatcher.onDidCreate(() => {
            this.invalidateProjectCache(projectRoot)
        })

        this.fileWatcher.onDidDelete(() => {
            this.invalidateProjectCache(projectRoot)
        })
    }

    private invalidateProjectCache(projectRoot: string): void {
        const cacheKey = `project:${projectRoot}`
        this.modeCache.delete(cacheKey)
    }

    private deduplicateModes(modes: Mode[]): Mode[] {
        const seen = new Set<string>()
        const result: Mode[] = []

        for (const mode of modes) {
            if (!seen.has(mode.slug)) {
                seen.add(mode.slug)
                result.push(mode)
            } else {
                // 项目级模式优先于全局模式
                const existingIndex = result.findIndex(m => m.slug === mode.slug)
                if (existingIndex >= 0 && mode.source === 'project') {
                    result[existingIndex] = mode
                }
            }
        }

        return result
    }
}
```

## 3. 模式切换系统

### 3.1 模式切换工具实现

```typescript
// src/core/tools/switchModeTool.ts
export async function switchModeTool(
    cline: Cline,
    block: ToolUse,
    addMessage: (message: Message) => Promise<void>,
    updateLastMessage: (message: Message) => Promise<void>
): Promise<boolean> {
    const { mode: targetModeSlug } = block.params

    try {
        // 1. 验证目标模式
        const targetMode = await validateTargetMode(targetModeSlug)

        // 2. 检查切换权限
        await validateModeSwitchPermission(cline, targetMode)

        // 3. 获取用户确认
        const userConfirmed = await requestModeSwitchConfirmation(
            cline,
            targetMode,
            addMessage
        )

        if (!userConfirmed) {
            await addMessage({
                role: "tool",
                content: `Mode switch to "${targetMode.name}" cancelled by user.`,
                _type: "mode_switch_cancelled",
                _source: "switch_mode",
                ts: Date.now()
            })
            return false
        }

        // 4. 执行模式切换
        const switchResult = await executeModeSwitch(cline, targetMode)

        // 5. 更新系统提示
        await updateSystemPromptForMode(cline, targetMode)

        // 6. 添加切换成功消息
        await addMessage({
            role: "tool",
            content: `Successfully switched to ${targetMode.name} mode.\n\n${targetMode.description}`,
            _type: "mode_switch_success",
            _source: "switch_mode",
            ts: Date.now()
        })

        return true
    } catch (error) {
        console.error("Mode switch failed:", error)

        await addMessage({
            role: "tool",
            content: `Failed to switch mode: ${error.message}`,
            _type: "mode_switch_error",
            _source: "switch_mode",
            ts: Date.now()
        })

        return false
    }
}

async function validateTargetMode(modeSlug: string): Promise<Mode> {
    const modesManager = new CustomModesManager(cline.context)
    const allModes = await modesManager.getAllModes()

    const targetMode = allModes.find(mode => mode.slug === modeSlug)
    if (!targetMode) {
        throw new Error(`Mode not found: ${modeSlug}`)
    }

    return targetMode
}

async function validateModeSwitchPermission(cline: Cline, targetMode: Mode): Promise<void> {
    const currentMode = cline.currentMode

    // 检查是否已经在目标模式
    if (currentMode && currentMode.slug === targetMode.slug) {
        throw new Error(`Already in ${targetMode.name} mode`)
    }

    // 检查文件路径限制（自定义模式）
    if (targetMode.allowedFilePaths && targetMode.allowedFilePaths.length > 0) {
        const workspaceFolders = vscode.workspace.workspaceFolders
        if (!workspaceFolders || workspaceFolders.length === 0) {
            throw new Error("No workspace folder available for file path validation")
        }

        const currentFile = await getCurrentActiveFile()
        if (currentFile) {
            const isAllowed = targetMode.allowedFilePaths.some(pattern => {
                const regex = new RegExp(pattern)
                return regex.test(currentFile)
            })

            if (!isAllowed) {
                throw new Error(`Current file is not allowed in ${targetMode.name} mode`)
            }
        }
    }

    // 检查禁止文件路径
    if (targetMode.disallowedFilePaths && targetMode.disallowedFilePaths.length > 0) {
        const currentFile = await getCurrentActiveFile()
        if (currentFile) {
            const isDisallowed = targetMode.disallowedFilePaths.some(pattern => {
                const regex = new RegExp(pattern)
                return regex.test(currentFile)
            })

            if (isDisallowed) {
                throw new Error(`Current file is not allowed in ${targetMode.name} mode`)
            }
        }
    }
}

async function requestModeSwitchConfirmation(
    cline: Cline,
    targetMode: Mode,
    addMessage: (message: Message) => Promise<void>
): Promise<boolean> {
    return new Promise((resolve) => {
        // 创建模式切换确认消息
        const confirmationMessage: Message = {
            role: "user",
            content: `🔄 **Mode Switch Request**

**Target Mode**: ${targetMode.name}
**Role**: ${targetMode.role}
**Description**: ${targetMode.description}

**Available Tools**: ${targetMode.tools.join(', ')}

${targetMode.customInstructions ? `**Special Instructions**: ${targetMode.customInstructions}` : ''}

Do you want to switch to this mode?`,
            _type: "mode_switch_confirmation",
            _source: "switch_mode",
            ts: Date.now()
        }

        addMessage(confirmationMessage).then(() => {
            // 设置确认监听器
            const listener = (message: Message) => {
                if (message._type === "mode_switch_response") {
                    const response = (message as any).response
                    listener.dispose() // 清理监听器
                    resolve(response === "confirm")
                }
            }

            // 添加到消息监听器
            cline.addMessageListener(listener)
        })
    })
}

async function executeModeSwitch(cline: Cline, targetMode: Mode): Promise<ModeSwitchResult> {
    const startTime = Date.now()

    try {
        // 保存当前模式状态
        const previousMode = cline.currentMode
        const modeContext = await captureModeContext(cline)

        // 切换模式
        cline.currentMode = targetMode

        // 应用模式特定的配置
        await applyModeConfiguration(cline, targetMode)

        // 更新模式历史
        await updateModeHistory(cline, {
            from: previousMode?.slug,
            to: targetMode.slug,
            timestamp: new Date(),
            context: modeContext
        })

        // 触发模式切换事件
        cline.emit('modeChanged', {
            previousMode,
            currentMode: targetMode,
            duration: Date.now() - startTime
        })

        return {
            success: true,
            previousMode,
            currentMode: targetMode,
            duration: Date.now() - startTime
        }
    } catch (error) {
        return {
            success: false,
            error: error.message,
            duration: Date.now() - startTime
        }
    }
}

async function applyModeConfiguration(cline: Cline, mode: Mode): Promise<void> {
    // 更新工具权限
    cline.availableTools = new Set(mode.tools)

    // 应用自定义 API 配置
    if (mode.apiBase && mode.apiModel) {
        await cline.updateApiConfiguration({
            baseURL: mode.apiBase,
            model: mode.apiModel,
            provider: mode.apiProvider
        })
    }

    // 应用模式特定的变量
    if (mode.variables) {
        cline.modeVariables = { ...cline.modeVariables, ...mode.variables }
    }

    // 应用实验性功能
    if (mode.experimentalFeatures) {
        cline.experimentalFeatures = {
            ...cline.experimentalFeatures,
            ...mode.experimentalFeatures
        }
    }
}

async function updateSystemPromptForMode(cline: Cline, mode: Mode): Promise<void> {
    // 构建模式特定的系统提示
    const modeSpecificPrompt = buildModeSpecificPrompt(mode)

    // 更新系统提示
    await cline.updateSystemPrompt(modeSpecificPrompt)
}

function buildModeSpecificPrompt(mode: Mode): string {
    let prompt = `You are now in ${mode.name} mode.\n\n`

    prompt += `## Your Role\n${mode.role}\n\n`

    prompt += `## Mode Description\n${mode.description}\n\n`

    if (mode.customInstructions) {
        prompt += `## Special Instructions\n${mode.customInstructions}\n\n`
    }

    prompt += `## Available Tools\n`
    prompt += `You have access to the following tools: ${mode.tools.join(', ')}.\n\n`

    if (mode.systemPrompt) {
        prompt += `## Additional System Instructions\n${mode.systemPrompt}\n\n`
    }

    return prompt
}
```

## 4. Orchestrator 模式实现

### 4.1 任务编排和委托系统

```typescript
// src/core/tools/newTaskTool.ts
export async function newTaskTool(
    cline: Cline,
    block: ToolUse,
    addMessage: (message: Message) => Promise<void>,
    updateLastMessage: (message: Message) => Promise<void>
): Promise<boolean> {
    const {
        description,
        mode: targetModeSlug,
        parentTaskId,
        priority = 'medium',
        context: taskContext
    } = block.params

    try {
        // 1. 验证目标模式（仅在 Orchestrator 模式下允许任务委托）
        if (cline.currentMode?.slug !== 'orchestrator') {
            throw new Error('Only Orchestrator mode can create delegated tasks')
        }

        // 2. 验证目标模式
        const targetMode = await validateTargetMode(targetModeSlug)

        // 3. 创建任务
        const task = await createDelegatedTask({
            description,
            targetMode,
            parentTaskId,
            priority,
            context: taskContext,
            orchestrator: cline.currentMode,
            workspace: cline.workspace
        })

        // 4. 执行任务委托
        const delegationResult = await executeTaskDelegation(cline, task, targetMode)

        // 5. 添加任务创建消息
        await addMessage({
            role: "tool",
            content: `Task delegated to ${targetMode.name} mode:

**Task ID**: ${task.id}
**Description**: ${task.description}
**Priority**: ${task.priority}
**Target Mode**: ${targetMode.name}

**Status**: ${delegationResult.status}
${delegationResult.estimatedDuration ? `**Estimated Duration**: ${delegationResult.estimatedDuration}` : ''}`,
            _type: "task_delegated",
            _source: "new_task",
            ts: Date.now()
        })

        return true
    } catch (error) {
        console.error("Task delegation failed:", error)

        await addMessage({
            role: "tool",
            content: `Failed to delegate task: ${error.message}`,
            _type: "task_delegation_error",
            _source: "new_task",
            ts: Date.now()
        })

        return false
    }
}

interface DelegatedTask {
    id: string
    description: string
    targetMode: Mode
    parentTaskId?: string
    priority: 'low' | 'medium' | 'high' | 'urgent'
    context: any
    orchestrator: Mode
    workspace: string
    status: 'pending' | 'executing' | 'completed' | 'failed'
    createdAt: Date
    startedAt?: Date
    completedAt?: Date
    result?: any
    error?: string
}

async function createDelegatedTask(options: {
    description: string
    targetMode: Mode
    parentTaskId?: string
    priority: string
    context: any
    orchestrator: Mode
    workspace: string
}): Promise<DelegatedTask> {
    return {
        id: generateTaskId(),
        description: options.description,
        targetMode: options.targetMode,
        parentTaskId: options.parentTaskId,
        priority: options.priority,
        context: options.context,
        orchestrator: options.orchestrator,
        workspace: options.workspace,
        status: 'pending',
        createdAt: new Date()
    }
}

async function executeTaskDelegation(
    cline: Cline,
    task: DelegatedTask,
    targetMode: Mode
): Promise<DelegationResult> {
    const startTime = Date.now()

    try {
        // 1. 创建任务上下文
        const taskContext = await createTaskContext(cline, task)

        // 2. 切换到目标模式
        await switchToTaskMode(cline, targetMode, taskContext)

        // 3. 执行任务
        const executionResult = await executeTaskInMode(cline, task)

        // 4. 返回到 Orchestrator 模式
        await returnToOrchestratorMode(cline)

        return {
            status: executionResult.success ? 'completed' : 'failed',
            duration: Date.now() - startTime,
            result: executionResult.result,
            estimatedDuration: estimateTaskDuration(task)
        }
    } catch (error) {
        return {
            status: 'failed',
            error: error.message,
            duration: Date.now() - startTime
        }
    }
}

async function createTaskContext(cline: Cline, task: DelegatedTask): Promise<TaskContext> {
    return {
        taskId: task.id,
        parentTaskId: task.parentTaskId,
        description: task.description,
        priority: task.priority,
        orchestratorContext: cloneContext(cline.context),
        sharedVariables: {},
        resultBuffer: [],
        progress: {
            completed: 0,
            total: 100,
            steps: []
        }
    }
}

async function switchToTaskMode(
    cline: Cline,
    targetMode: Mode,
    taskContext: TaskContext
): Promise<void> {
    // 保存 Orchestrator 上下文
    cline.orchestratorContext = cloneContext(cline.context)

    // 设置任务上下文
    cline.taskContext = taskContext

    // 切换模式
    cline.currentMode = targetMode

    // 应用模式配置
    await applyModeConfiguration(cline, targetMode)

    // 构建任务特定的系统提示
    const taskPrompt = buildTaskSpecificPrompt(targetMode, taskContext)
    await cline.updateSystemPrompt(taskPrompt)
}

async function executeTaskInMode(cline: Cline, task: DelegatedTask): Promise<TaskExecutionResult> {
    try {
        // 更新任务状态
        task.status = 'executing'
        task.startedAt = new Date()

        // 这里会触发 AI 在目标模式下执行任务
        // 具体的执行逻辑依赖于 AI 的响应和工具调用
        const result = await processTaskExecution(cline, task)

        task.status = 'completed'
        task.completedAt = new Date()
        task.result = result

        return {
            success: true,
            result
        }
    } catch (error) {
        task.status = 'failed'
        task.completedAt = new Date()
        task.error = error.message

        return {
            success: false,
            error: error.message
        }
    }
}

function buildTaskSpecificPrompt(mode: Mode, taskContext: TaskContext): string {
    let prompt = `You are executing a delegated task in ${mode.name} mode.\n\n`

    prompt += `## Task Information\n`
    prompt += `- **Task ID**: ${taskContext.taskId}\n`
    prompt += `- **Description**: ${taskContext.description}\n`
    prompt += `- **Priority**: ${taskContext.priority}\n\n`

    prompt += `## Your Role\n${mode.role}\n\n`

    prompt += `## Available Tools\nYou have access to: ${mode.tools.join(', ')}\n\n`

    if (mode.customInstructions) {
        prompt += `## Special Instructions\n${mode.customInstructions}\n\n`
    }

    prompt += `## Task Context\n`
    prompt += `You have been delegated this task by the Orchestrator mode. `
    prompt += `Focus on completing this specific task efficiently. `
    prompt += `Share your results and any important findings with the Orchestrator.\n\n`

    return prompt
}

async function returnToOrchestratorMode(cline: Cline): Promise<void> {
    const orchestratorMode = BUILTIN_MODES.find(m => m.slug === 'orchestrator')
    if (!orchestratorMode) {
        throw new Error('Orchestrator mode not found')
    }

    // 恢复 Orchestrator 上下文
    cline.context = cline.orchestratorContext || cloneContext(cline.context)

    // 清理任务上下文
    cline.taskContext = null

    // 切换回 Orchestrator 模式
    cline.currentMode = orchestratorMode
    await applyModeConfiguration(cline, orchestratorMode)

    // 恢复原始系统提示
    await cline.updateSystemPrompt(buildModeSpecificPrompt(orchestratorMode))
}
```

## 5. 工具权限控制系统

### 5.1 工具验证和权限管理

```typescript
// src/core/tools/validateToolUse.ts
export async function validateToolUse(
    cline: Cline,
    block: ToolUse,
    addMessage: (message: Message) => Promise<void>
): Promise<ToolValidationResult> {
    const { name: toolName } = block

    try {
        // 1. 检查工具是否可用
        const isAvailable = await isToolAvailable(cline, toolName)
        if (!isAvailable) {
            return {
                valid: false,
                reason: `Tool "${toolName}" is not available in current mode`,
                blocked: true
            }
        }

        // 2. 验证工具参数
        const paramValidation = await validateToolParameters(toolName, block.params)
        if (!paramValidation.valid) {
            return {
                valid: false,
                reason: `Invalid parameters for tool "${toolName}": ${paramValidation.reason}`,
                blocked: true
            }
        }

        // 3. 检查工具特定的权限
        const permissionCheck = await checkToolPermissions(cline, toolName, block.params)
        if (!permissionCheck.granted) {
            return {
                valid: false,
                reason: permissionCheck.reason,
                blocked: true,
                requiresApproval: permissionCheck.requiresApproval
            }
        }

        // 4. 检查文件路径限制
        if (toolName === 'read' || toolName === 'edit') {
            const fileCheck = await checkFilePathPermissions(cline, block.params)
            if (!fileCheck.allowed) {
                return {
                    valid: false,
                    reason: fileCheck.reason,
                    blocked: true
                }
            }
        }

        // 5. 检查资源使用限制
        const resourceCheck = await checkResourceLimits(cline, toolName)
        if (!resourceCheck.allowed) {
            return {
                valid: false,
                reason: resourceCheck.reason,
                blocked: true
            }
        }

        return {
            valid: true,
            granted: true
        }
    } catch (error) {
        return {
            valid: false,
            reason: `Tool validation error: ${error.message}`,
            blocked: true
        }
    }
}

async function isToolAvailable(cline: Cline, toolName: string): Promise<boolean> {
    const currentMode = cline.currentMode
    if (!currentMode) {
        return false
    }

    // 模式切换工具总是可用
    if (toolName === 'modes') {
        return true
    }

    // 检查当前模式的工具权限
    return currentMode.tools.includes(toolName as ToolName)
}

async function checkToolPermissions(
    cline: Cline,
    toolName: string,
    params: any
): Promise<PermissionCheck> {
    const currentMode = cline.currentMode

    switch (toolName) {
        case 'command':
            return checkCommandPermissions(cline, params)
        case 'browser':
            return checkBrowserPermissions(cline, params)
        case 'edit':
            return checkEditPermissions(cline, params)
        case 'read':
            return checkReadPermissions(cline, params)
        default:
            return { granted: true }
    }
}

async function checkCommandPermissions(cline: Cline, params: any): Promise<PermissionCheck> {
    const { command } = params

    // 检查危险命令
    const dangerousCommands = [
        /^rm\s+-rf/,
        /^sudo\s+/,
        /^format/,
        /^del/,
        /^rmdir\/s/,
        /^shutdown/,
        /^reboot/
    ]

    for (const pattern of dangerousCommands) {
        if (pattern.test(command)) {
            return {
                granted: false,
                reason: `Command "${command}" requires special approval due to potential security risks`,
                requiresApproval: true
            }
        }
    }

    // 检查命令复杂度和影响范围
    const complexity = analyzeCommandComplexity(command)
    if (complexity > 0.8) {
        return {
            granted: false,
            reason: `Complex command "${command}" requires approval`,
            requiresApproval: true
        }
    }

    return { granted: true }
}

async function checkBrowserPermissions(cline: Cline, params: any): Promise<PermissionCheck> {
    const { url } = params

    // 检查 URL 安全性
    if (!isSafeUrl(url)) {
        return {
            granted: false,
            reason: `URL "${url}" is not allowed for security reasons`,
            requiresApproval: false
        }
    }

    // 检查网络访问权限
    if (!await hasNetworkAccessPermission(cline)) {
        return {
            granted: false,
            reason: 'Network access is not available in current environment',
            requiresApproval: false
        }
    }

    return { granted: true }
}

async function checkFilePathPermissions(cline: Cline, params: any): Promise<FilePermissionCheck> {
    const currentMode = cline.currentMode
    if (!currentMode) {
        return { allowed: true }
    }

    let filePath = ''
    if (params.path) {
        filePath = params.path
    } else if (params.relPath) {
        filePath = path.resolve(cline.workspace, params.relPath)
    }

    if (!filePath) {
        return { allowed: true }
    }

    // 检查允许的文件路径
    if (currentMode.allowedFilePaths && currentMode.allowedFilePaths.length > 0) {
        const isAllowed = currentMode.allowedFilePaths.some(pattern => {
            const regex = new RegExp(pattern)
            return regex.test(filePath)
        })

        if (!isAllowed) {
            return {
                allowed: false,
                reason: `File "${filePath}" is not allowed in ${currentMode.name} mode`
            }
        }
    }

    // 检查禁止的文件路径
    if (currentMode.disallowedFilePaths && currentMode.disallowedFilePaths.length > 0) {
        const isDisallowed = currentMode.disallowedFilePaths.some(pattern => {
            const regex = new RegExp(pattern)
            return regex.test(filePath)
        })

        if (isDisallowed) {
            return {
                allowed: false,
                reason: `File "${filePath}" is not allowed in ${currentMode.name} mode`
            }
        }
    }

    return { allowed: true }
}

function isSafeUrl(url: string): boolean {
    try {
        const urlObj = new URL(url)

        // 检查协议
        const allowedProtocols = ['http:', 'https:']
        if (!allowedProtocols.includes(urlObj.protocol)) {
            return false
        }

        // 检查本地网络地址
        if (urlObj.hostname === 'localhost' || urlObj.hostname === '127.0.0.1') {
            return false
        }

        // 检查私有网络
        const privateNetworks = [
            /^10\./,
            /^192\.168\./,
            /^172\.(1[6-9]|2[0-9]|3[0-1])\./
        ]

        for (const network of privateNetworks) {
            if (network.test(urlObj.hostname)) {
                return false
            }
        }

        return true
    } catch {
        return false
    }
}

function analyzeCommandComplexity(command: string): number {
    let complexity = 0

    // 管道操作增加复杂度
    if (command.includes('|')) {
        complexity += 0.3
    }

    // 重定向增加复杂度
    if (command.includes('>') || command.includes('<')) {
        complexity += 0.2
    }

    // 环境变量使用增加复杂度
    if (command.includes('$')) {
        complexity += 0.2
    }

    // 循环和条件语句增加复杂度
    if (command.includes('for') || command.includes('while') || command.includes('if')) {
        complexity += 0.3
    }

    // 长命令增加复杂度
    if (command.length > 100) {
        complexity += 0.1
    }

    return Math.min(complexity, 1.0)
}
```

## 6. UI 界面和用户交互

### 6.1 模式管理界面

```typescript
// webview-ui/src/components/modes/ModesView.tsx
export function ModesView() {
    const [modes, setModes] = useState<Mode[]>([])
    const [selectedMode, setSelectedMode] = useState<Mode | null>(null)
    const [isEditing, setIsEditing] = useState(false)
    const [isCreating, setIsCreating] = useState(false)
    const [searchTerm, setSearchTerm] = useState('')

    const { modesManager, currentMode, switchMode } = useModes()

    useEffect(() => {
        loadModes()
    }, [])

    const loadModes = async () => {
        try {
            const allModes = await modesManager.getAllModes()
            setModes(allModes)
        } catch (error) {
            console.error('Failed to load modes:', error)
        }
    }

    const filteredModes = modes.filter(mode =>
        mode.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
        mode.description.toLowerCase().includes(searchTerm.toLowerCase()) ||
        mode.slug.toLowerCase().includes(searchTerm.toLowerCase())
    )

    const handleModeSwitch = async (mode: Mode) => {
        try {
            await switchMode(mode.slug)
        } catch (error) {
            console.error('Failed to switch mode:', error)
        }
    }

    const handleCreateMode = () => {
        setIsCreating(true)
        setSelectedMode(null)
    }

    const handleEditMode = (mode: Mode) => {
        setSelectedMode(mode)
        setIsEditing(true)
    }

    const handleDeleteMode = async (mode: Mode) => {
        if (window.confirm(`Are you sure you want to delete the "${mode.name}" mode?`)) {
            try {
                await modesManager.deleteCustomMode(mode.slug)
                await loadModes()
            } catch (error) {
                console.error('Failed to delete mode:', error)
                alert(`Failed to delete mode: ${error.message}`)
            }
        }
    }

    const handleSaveMode = async (modeData: Partial<Mode>) => {
        try {
            if (selectedMode) {
                // 更新现有模式
                await modesManager.updateCustomMode(selectedMode.slug, modeData)
            } else {
                // 创建新模式
                await modesManager.createCustomMode(modeData as Omit<Mode, 'isCustom'>)
            }

            await loadModes()
            setIsEditing(false)
            setIsCreating(false)
            setSelectedMode(null)
        } catch (error) {
            console.error('Failed to save mode:', error)
            alert(`Failed to save mode: ${error.message}`)
        }
    }

    return (
        <div className="modes-view">
            <div className="modes-header">
                <h2>Modes</h2>
                <div className="modes-actions">
                    <SearchInput
                        placeholder="Search modes..."
                        value={searchTerm}
                        onChange={setSearchTerm}
                    />
                    <Button
                        icon="plus"
                        onClick={handleCreateMode}
                        tooltip="Create new mode"
                    >
                        New Mode
                    </Button>
                </div>
            </div>

            <div className="modes-grid">
                {filteredModes.map(mode => (
                    <ModeCard
                        key={mode.slug}
                        mode={mode}
                        isActive={currentMode?.slug === mode.slug}
                        onSelect={handleModeSwitch}
                        onEdit={mode.isCustom ? handleEditMode : undefined}
                        onDelete={mode.isCustom ? handleDeleteMode : undefined}
                    />
                ))}
            </div>

            {(isEditing || isCreating) && (
                <ModeEditor
                    mode={selectedMode}
                    onSave={handleSaveMode}
                    onCancel={() => {
                        setIsEditing(false)
                        setIsCreating(false)
                        setSelectedMode(null)
                    }}
                />
            )}
        </div>
    )
}

function ModeCard({ mode, isActive, onSelect, onEdit, onDelete }: ModeCardProps) {
    return (
        <div className={`mode-card ${isActive ? 'active' : ''} ${mode.isCustom ? 'custom' : 'builtin'}`}>
            <div className="mode-header">
                <div className="mode-icon">
                    <VSCodeIcon name={mode.icon} />
                </div>
                <div className="mode-info">
                    <h3>{mode.name}</h3>
                    <p>{mode.description}</p>
                </div>
                <div className="mode-status">
                    {isActive && <span className="active-badge">Active</span>}
                    {mode.isCustom && <span className="custom-badge">Custom</span>}
                </div>
            </div>

            <div className="mode-content">
                <div className="mode-role">
                    <strong>Role:</strong> {mode.role.substring(0, 100)}...
                </div>
                <div className="mode-tools">
                    <strong>Tools:</strong> {mode.tools.join(', ')}
                </div>
            </div>

            <div className="mode-actions">
                <Button
                    variant={isActive ? 'secondary' : 'primary'}
                    onClick={() => onSelect(mode)}
                    disabled={isActive}
                >
                    {isActive ? 'Active' : 'Switch'}
                </Button>
                {onEdit && (
                    <Button
                        variant="ghost"
                        icon="edit"
                        onClick={() => onEdit(mode)}
                        tooltip="Edit mode"
                    />
                )}
                {onDelete && (
                    <Button
                        variant="ghost"
                        icon="trash"
                        onClick={() => onDelete(mode)}
                        tooltip="Delete mode"
                    />
                )}
            </div>
        </div>
    )
}

function ModeEditor({ mode, onSave, onCancel }: ModeEditorProps) {
    const [formData, setFormData] = useState<Partial<Mode>>({
        slug: mode?.slug || '',
        name: mode?.name || '',
        role: mode?.role || '',
        description: mode?.description || '',
        customInstructions: mode?.customInstructions || '',
        tools: mode?.tools || [],
        icon: mode?.icon || '',
        allowedFilePaths: mode?.allowedFilePaths || [],
        disallowedFilePaths: mode?.disallowedFilePaths || []
    })

    const [errors, setErrors] = useState<Record<string, string>>({})

    const validateForm = (): boolean => {
        const newErrors: Record<string, string> = {}

        if (!formData.slug) {
            newErrors.slug = 'Slug is required'
        } else if (!/^[a-z0-9-]+$/.test(formData.slug)) {
            newErrors.slug = 'Slug must contain only lowercase letters, numbers, and hyphens'
        }

        if (!formData.name) {
            newErrors.name = 'Name is required'
        }

        if (!formData.role) {
            newErrors.role = 'Role is required'
        }

        if (!formData.description) {
            newErrors.description = 'Description is required'
        }

        setErrors(newErrors)
        return Object.keys(newErrors).length === 0
    }

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault()
        if (validateForm()) {
            onSave(formData)
        }
    }

    return (
        <div className="mode-editor-overlay">
            <div className="mode-editor">
                <div className="mode-editor-header">
                    <h3>{mode ? 'Edit Mode' : 'Create New Mode'}</h3>
                    <Button variant="ghost" icon="close" onClick={onCancel} />
                </div>

                <form onSubmit={handleSubmit} className="mode-editor-form">
                    <FormField
                        label="Slug"
                        name="slug"
                        value={formData.slug}
                        onChange={(value) => setFormData({ ...formData, slug: value })}
                        error={errors.slug}
                        description="Unique identifier for the mode"
                        disabled={!!mode} // Slug cannot be changed after creation
                    />

                    <FormField
                        label="Name"
                        name="name"
                        value={formData.name}
                        onChange={(value) => setFormData({ ...formData, name: value })}
                        error={errors.name}
                        description="Display name for the mode"
                    />

                    <FormField
                        label="Role"
                        name="role"
                        type="textarea"
                        value={formData.role}
                        onChange={(value) => setFormData({ ...formData, role: value })}
                        error={errors.role}
                        description="AI role description"
                        rows={3}
                    />

                    <FormField
                        label="Description"
                        name="description"
                        type="textarea"
                        value={formData.description}
                        onChange={(value) => setFormData({ ...formData, description: value })}
                        error={errors.description}
                        description="Brief description of the mode's purpose"
                        rows={2}
                    />

                    <FormField
                        label="Custom Instructions"
                        name="customInstructions"
                        type="textarea"
                        value={formData.customInstructions}
                        onChange={(value) => setFormData({ ...formData, customInstructions: value })}
                        description="Additional instructions for this mode"
                        rows={4}
                    />

                    <FormField
                        label="Tools"
                        name="tools"
                        type="multiselect"
                        value={formData.tools}
                        onChange={(value) => setFormData({ ...formData, tools: value })}
                        options={[
                            { value: 'read', label: 'Read Files' },
                            { value: 'edit', label: 'Edit Files' },
                            { value: 'browser', label: 'Browser' },
                            { value: 'command', label: 'Command' },
                            { value: 'mcp', label: 'MCP Tools' }
                        ]}
                        description="Tools available in this mode"
                    />

                    <FormField
                        label="Icon"
                        name="icon"
                        value={formData.icon}
                        onChange={(value) => setFormData({ ...formData, icon: value })}
                        description="VS Code icon name (optional)"
                    />

                    <FormField
                        label="Allowed File Paths"
                        name="allowedFilePaths"
                        type="tags"
                        value={formData.allowedFilePaths}
                        onChange={(value) => setFormData({ ...formData, allowedFilePaths: value })}
                        description="Regex patterns for allowed file paths (optional)"
                    />

                    <FormField
                        label="Disallowed File Paths"
                        name="disallowedFilePaths"
                        type="tags"
                        value={formData.disallowedFilePaths}
                        onChange={(value) => setFormData({ ...formData, disallowedFilePaths: value })}
                        description="Regex patterns for disallowed file paths (optional)"
                    />

                    <div className="mode-editor-actions">
                        <Button type="button" variant="secondary" onClick={onCancel}>
                            Cancel
                        </Button>
                        <Button type="submit" variant="primary">
                            {mode ? 'Update' : 'Create'} Mode
                        </Button>
                    </div>
                </form>
            </div>
        </div>
    )
}
```

## 总结

KiloCode 的 Multi Mode 系统代表了 AI 助手架构的一次创新性突破，展现了以下技术特点：

### 1. **创新的架构设计**
- 五大内置模式：Architect、Code、Ask、Debug、Orchestrator
- 完整的自定义模式系统
- 灵活的工具权限控制
- 智能的任务编排机制

### 2. **强大的扩展能力**
- 模式级别的配置和定制
- 文件路径限制和权限管理
- 实验性功能的渐进式启用
- 完整的导入导出功能

### 3. **智能的协作机制**
- Orchestrator 模式的任务委托
- 跨模式的上下文共享
- 任务执行的状态跟踪
- 结果合成和报告生成

### 4. **优秀的用户体验**
- 直观的模式切换界面
- 实时的权限验证
- 详细的模式信息展示
- 流畅的交互反馈

这个系统不仅提供了基础的 AI 助手功能，更建立了一个完整的 AI 角色生态系统，让用户可以根据不同的需求创建和使用专门的 AI 助手，极大地扩展了 AI 代码助手的应用场景和实用性。