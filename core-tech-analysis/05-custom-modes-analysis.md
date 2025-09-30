# KiloCode 自定义模式实现机制深度分析

## 系统架构概览

KiloCode 的自定义模式系统是一个功能强大的 AI 助手定制框架，允许用户创建、管理和分享专门化的 AI 模式。该系统采用分层配置管理、严格的权限控制和灵活的扩展机制，为用户提供了完整的 AI 行为定制能力。

## 1. 自定义模式核心架构

### 1.1 自定义模式数据模型

```typescript
// packages/types/src/mode.ts (扩展)
export interface CustomMode extends Mode {
    // 自定义模式特有字段
    source: 'global' | 'project'
    filePath?: string
    createdAt: Date
    updatedAt: Date
    createdBy?: string
    version?: string
    dependencies?: string[]
    templates?: ModeTemplate[]
    hooks?: ModeHooks
    validation?: ModeValidation
    marketplace?: MarketplaceInfo
}

export interface ModeTemplate {
    id: string
    name: string
    description: string
    variables: TemplateVariable[]
    conditions: TemplateCondition[]
    actions: TemplateAction[]
}

export interface ModeHooks {
    onBeforeModeSwitch?: () => Promise<void>
    onAfterModeSwitch?: () => Promise<void>
    onBeforeToolUse?: (toolName: string, params: any) => Promise<boolean>
    onAfterToolUse?: (toolName: string, params: any, result: any) => Promise<void>
    onFileAccess?: (filePath: string, action: 'read' | 'write') => Promise<boolean>
}

export interface ModeValidation {
    requiredTools?: string[]
    requiredVariables?: string[]
    filePatternRules?: FilePatternRule[]
    customValidators?: CustomValidator[]
}

export interface FilePatternRule {
    pattern: string
    required?: boolean
    description?: string
    errorMessage?: string
}

export interface CustomValidator {
    name: string
    validate: (context: ValidationContext) => Promise<ValidationResult>
    errorMessage?: string
}

export interface MarketplaceInfo {
    isPublished?: boolean
    marketplaceId?: string
    downloadCount?: number
    rating?: number
    reviews?: MarketplaceReview[]
    tags?: string[]
    category?: string
}
```

### 1.2 自定义模式管理器扩展

```typescript
// src/core/config/CustomModesManager.ts (扩展)
export class CustomModesManager {
    // ... 前面的基础代码 ...

    private templateManager: ModeTemplateManager
    private validatorManager: ModeValidatorManager
    private marketplaceClient: MarketplaceClient
    private hooksManager: ModeHooksManager

    constructor(context: vscode.ExtensionContext) {
        super(context)
        this.templateManager = new ModeTemplateManager(context)
        this.validatorManager = new ModeValidatorManager(context)
        this.marketplaceClient = new MarketplaceClient(context)
        this.hooksManager = new ModeHooksManager(context)
    }

    // 高级自定义模式创建
    async createAdvancedCustomMode(
        config: AdvancedModeConfig
    ): Promise<CustomMode> {
        // 1. 验证配置
        const validatedConfig = await this.validateAdvancedConfig(config)

        // 2. 处理模板
        const processedConfig = await this.templateManager.processTemplates(validatedConfig)

        // 3. 解析依赖
        await this.resolveDependencies(processedConfig)

        // 4. 设置钩子
        await this.setupModeHooks(processedConfig)

        // 5. 创建模式
        const mode = await this.createModeObject(processedConfig)

        // 6. 验证模式
        await this.validatorManager.validateMode(mode)

        // 7. 保存模式
        await this.saveCustomMode(mode)

        return mode
    }

    // 模式克隆和继承
    async cloneMode(
        sourceModeSlug: string,
        newConfig: Partial<CustomMode>
    ): Promise<CustomMode> {
        const sourceMode = await this.findModeBySlug(sourceModeSlug)
        if (!sourceMode) {
            throw new Error(`Source mode not found: ${sourceModeSlug}`)
        }

        const clonedMode: CustomMode = {
            ...sourceMode,
            ...newConfig,
            slug: newConfig.slug || this.generateSlugFromName(newConfig.name || `${sourceMode.name} Clone`),
            name: newConfig.name || `${sourceMode.name} Clone`,
            source: newConfig.source || 'global',
            createdAt: new Date(),
            updatedAt: new Date(),
            version: newConfig.version || this.incrementVersion(sourceMode.version),
            dependencies: newConfig.dependencies || sourceMode.dependencies,
            createdBy: newConfig.createdBy || sourceMode.createdBy
        }

        return await this.createAdvancedCustomMode(clonedMode)
    }

    // 模式版本管理
    async createModeVersion(
        modeSlug: string,
        changes: Partial<CustomMode>,
        changelog?: string
    ): Promise<CustomMode> {
        const existingMode = await this.findModeBySlug(modeSlug)
        if (!existingMode) {
            throw new Error(`Mode not found: ${modeSlug}`)
        }

        const newVersion = this.incrementVersion(existingMode.version)
        const versionedMode: CustomMode = {
            ...existingMode,
            ...changes,
            version: newVersion,
            updatedAt: new Date()
        }

        // 保存版本历史
        await this.saveModeVersionHistory(existingMode, versionedMode, changelog)

        return await this.updateCustomMode(modeSlug, versionedMode)
    }

    // 模式依赖解析
    private async resolveDependencies(mode: CustomMode): Promise<void> {
        if (!mode.dependencies || mode.dependencies.length === 0) {
            return
        }

        const resolvedDependencies: CustomMode[] = []

        for (const depSlug of mode.dependencies) {
            try {
                const dependency = await this.findModeBySlug(depSlug)
                if (dependency) {
                    resolvedDependencies.push(dependency)
                } else {
                    console.warn(`Dependency not found: ${depSlug}`)
                }
            } catch (error) {
                console.warn(`Failed to resolve dependency ${depSlug}:`, error)
            }
        }

        // 应用依赖继承
        await this.applyDependencyInheritance(mode, resolvedDependencies)
    }

    private async applyDependencyInheritance(
        mode: CustomMode,
        dependencies: CustomMode[]
    ): Promise<void> {
        for (const dependency of dependencies) {
            // 继承工具权限（合并）
            mode.tools = Array.from(new Set([...mode.tools, ...dependency.tools]))

            // 继承变量（深度合并）
            if (dependency.variables) {
                mode.variables = {
                    ...dependency.variables,
                    ...mode.variables
                }
            }

            // 继承文件路径规则（合并）
            if (dependency.allowedFilePaths) {
                mode.allowedFilePaths = [
                    ...(mode.allowedFilePaths || []),
                    ...(dependency.allowedFilePaths || [])
                ]
            }

            if (dependency.disallowedFilePaths) {
                mode.disallowedFilePaths = [
                    ...(mode.disallowedFilePaths || []),
                    ...(dependency.disallowedFilePaths || [])
                ]
            }
        }
    }

    // 模式模板处理
    async createModeFromTemplate(
        templateId: string,
        variables: Record<string, any>
    ): Promise<CustomMode> {
        const template = await this.templateManager.getTemplate(templateId)
        if (!template) {
            throw new Error(`Template not found: ${templateId}`)
        }

        // 处理模板变量
        const processedConfig = await this.templateManager.processTemplateVariables(
            template,
            variables
        )

        // 验证模板条件
        await this.templateManager.validateTemplateConditions(template, variables)

        // 创建模式
        return await this.createAdvancedCustomMode(processedConfig)
    }

    // 模式导出和分享
    async exportMode(modeSlug: string, format: 'json' | 'package' = 'json'): Promise<ExportResult> {
        const mode = await this.findModeBySlug(modeSlug)
        if (!mode) {
            throw new Error(`Mode not found: ${modeSlug}`)
        }

        switch (format) {
            case 'json':
                return await this.exportModeAsJson(mode)
            case 'package':
                return await this.exportModeAsPackage(mode)
            default:
                throw new Error(`Unsupported export format: ${format}`)
        }
    }

    private async exportModeAsJson(mode: CustomMode): Promise<ExportResult> {
        const exportData = {
            mode,
            metadata: {
                exportedAt: new Date(),
                version: '1.0',
                exporter: 'KiloCode Custom Modes'
            },
            dependencies: await this.resolveModeDependencies(mode)
        }

        const content = JSON.stringify(exportData, null, 2)
        const fileName = `${mode.slug}-export.json`

        return {
            format: 'json',
            content,
            fileName,
            size: content.length
        }
    }

    private async exportModeAsPackage(mode: CustomMode): Promise<ExportResult> {
        // 创建压缩包
        const archive = await this.createModePackage(mode)

        return {
            format: 'package',
            content: archive,
            fileName: `${mode.slug}-package.zip`,
            size: archive.length
        }
    }

    // 模式导入
    async importMode(
        importData: string | Buffer,
        format: 'json' | 'package' = 'json'
    ): Promise<CustomMode> {
        switch (format) {
            case 'json':
                return await this.importModeFromJson(importData as string)
            case 'package':
                return await this.importModeFromPackage(importData as Buffer)
            default:
                throw new Error(`Unsupported import format: ${format}`)
        }
    }

    private async importModeFromJson(jsonData: string): Promise<CustomMode> {
        try {
            const importData = JSON.parse(jsonData)

            // 验证导入数据
            if (!importData.mode) {
                throw new Error('Invalid import data: missing mode object')
            }

            // 检查冲突
            const existingMode = await this.findModeBySlug(importData.mode.slug)
            if (existingMode) {
                throw new Error(`Mode with slug "${importData.mode.slug}" already exists`)
            }

            // 解析依赖
            if (importData.dependencies) {
                await this.resolveImportDependencies(importData.dependencies)
            }

            // 创建模式
            const mode: CustomMode = {
                ...importData.mode,
                source: 'global',
                createdAt: new Date(),
                updatedAt: new Date(),
                importedAt: new Date()
            }

            return await this.createAdvancedCustomMode(mode)
        } catch (error) {
            throw new Error(`Failed to import mode: ${error.message}`)
        }
    }

    // 模式市场集成
    async publishToMarketplace(
        modeSlug: string,
        marketplaceConfig: MarketplaceConfig
    ): Promise<MarketplaceResult> {
        const mode = await this.findModeBySlug(modeSlug)
        if (!mode) {
            throw new Error(`Mode not found: ${modeSlug}`)
        }

        // 验证市场发布要求
        await this.validateMarketplaceRequirements(mode)

        // 准备市场数据
        const marketplaceData = await this.prepareMarketplaceData(mode, marketplaceConfig)

        // 发布到市场
        const result = await this.marketplaceClient.publishMode(marketplaceData)

        // 更新模式信息
        mode.marketplace = {
            ...mode.marketplace,
            isPublished: true,
            marketplaceId: result.marketplaceId,
            publishedAt: new Date()
        }

        await this.updateCustomMode(modeSlug, mode)

        return result
    }

    async installFromMarketplace(marketplaceId: string): Promise<CustomMode> {
        // 从市场获取模式信息
        const marketplaceInfo = await this.marketplaceClient.getModeInfo(marketplaceId)

        // 下载模式包
        const modePackage = await this.marketplaceClient.downloadMode(marketplaceId)

        // 安装模式
        const mode = await this.importMode(modePackage, 'package')

        // 设置市场信息
        mode.marketplace = {
            ...marketplaceInfo,
            isPublished: true,
            marketplaceId,
            installedAt: new Date()
        }

        return await this.updateCustomMode(mode.slug, mode)
    }
}
```

## 2. 模式验证系统

### 2.1 验证管理器实现

```typescript
// src/core/config/ModeValidatorManager.ts
export class ModeValidatorManager {
    private customValidators: Map<string, CustomValidator> = new Map()
    private validationRules: ValidationRule[] = []

    constructor(context: vscode.ExtensionContext) {
        this.initializeDefaultValidators()
        this.loadCustomValidators()
    }

    async validateMode(mode: CustomMode): Promise<ValidationResult> {
        const errors: ValidationError[] = []

        // 1. 基础架构验证
        const schemaValidation = this.validateModeSchema(mode)
        errors.push(...schemaValidation.errors)

        // 2. 工具权限验证
        const toolsValidation = await this.validateModeTools(mode)
        errors.push(...toolsValidation.errors)

        // 3. 文件路径验证
        const filePathsValidation = await this.validateFilePaths(mode)
        errors.push(...filePathsValidation.errors)

        // 4. 自定义验证器
        const customValidation = await this.runCustomValidators(mode)
        errors.push(...customValidation.errors)

        // 5. 依赖验证
        const dependencyValidation = await this.validateDependencies(mode)
        errors.push(...dependencyValidation.errors)

        return {
            isValid: errors.length === 0,
            errors,
            warnings: this.generateValidationWarnings(mode)
        }
    }

    private validateModeSchema(mode: CustomMode): ValidationResult {
        const errors: ValidationError[] = []

        // 必需字段验证
        const requiredFields = ['slug', 'name', 'role', 'description', 'tools']
        for (const field of requiredFields) {
            if (!mode[field as keyof CustomMode]) {
                errors.push({
                    field,
                    message: `${field} is required`,
                    severity: 'error'
                })
            }
        }

        // Slug 格式验证
        if (mode.slug && !/^[a-z0-9-]+$/.test(mode.slug)) {
            errors.push({
                field: 'slug',
                message: 'Slug must contain only lowercase letters, numbers, and hyphens',
                severity: 'error'
            })
        }

        // 工具权限验证
        if (mode.tools && mode.tools.length > 0) {
            const validTools = ['read', 'edit', 'browser', 'command', 'mcp']
            const invalidTools = mode.tools.filter(tool => !validTools.includes(tool))

            if (invalidTools.length > 0) {
                errors.push({
                    field: 'tools',
                    message: `Invalid tools: ${invalidTools.join(', ')}`,
                    severity: 'error'
                })
            }
        }

        return { isValid: errors.length === 0, errors }
    }

    private async validateModeTools(mode: CustomMode): Promise<ValidationResult> {
        const errors: ValidationError[] = []

        // 检查工具组合的合理性
        if (mode.tools.includes('edit') && !mode.tools.includes('read')) {
            errors.push({
                field: 'tools',
                message: 'Edit tool requires read tool',
                severity: 'warning'
            })
        }

        // 检查危险工具组合
        const dangerousCombinations = [
            { tools: ['command', 'edit'], reason: 'Command and edit tools combination may be dangerous' },
            { tools: ['browser', 'command'], reason: 'Browser and command tools combination may be dangerous' }
        ]

        for (const combo of dangerousCombinations) {
            if (combo.tools.every(tool => mode.tools.includes(tool))) {
                errors.push({
                    field: 'tools',
                    message: combo.reason,
                    severity: 'warning'
                })
            }
        }

        return { isValid: errors.length === 0, errors }
    }

    private async validateFilePaths(mode: CustomMode): Promise<ValidationResult> {
        const errors: ValidationError[] = []

        // 验证允许的文件路径
        if (mode.allowedFilePaths) {
            for (const pattern of mode.allowedFilePaths) {
                try {
                    new RegExp(pattern)
                } catch (error) {
                    errors.push({
                        field: 'allowedFilePaths',
                        message: `Invalid regex pattern: ${pattern}`,
                        severity: 'error'
                    })
                }
            }
        }

        // 验证禁止的文件路径
        if (mode.disallowedFilePaths) {
            for (const pattern of mode.disallowedFilePaths) {
                try {
                    new RegExp(pattern)
                } catch (error) {
                    errors.push({
                        field: 'disallowedFilePaths',
                        message: `Invalid regex pattern: ${pattern}`,
                        severity: 'error'
                    })
                }
            }
        }

        // 检查冲突
        if (mode.allowedFilePaths && mode.disallowedFilePaths) {
            // 这里可以添加更复杂的冲突检测逻辑
        }

        return { isValid: errors.length === 0, errors }
    }

    private async runCustomValidators(mode: CustomMode): Promise<ValidationResult> {
        const errors: ValidationError[] = []

        for (const [name, validator] of this.customValidators) {
            try {
                const result = await validator.validate({
                    mode,
                    context: this.createValidationContext(mode)
                })

                if (!result.isValid) {
                    errors.push({
                        field: 'custom',
                        message: result.message || `Custom validator ${name} failed`,
                        severity: 'error',
                        validator: name
                    })
                }
            } catch (error) {
                errors.push({
                    field: 'custom',
                    message: `Custom validator ${name} failed: ${error.message}`,
                    severity: 'error'
                })
            }
        }

        return { isValid: errors.length === 0, errors }
    }

    private async validateDependencies(mode: CustomMode): Promise<ValidationResult> {
        const errors: ValidationError[] = []

        if (!mode.dependencies || mode.dependencies.length === 0) {
            return { isValid: true, errors: [] }
        }

        for (const depSlug of mode.dependencies) {
            try {
                const dependency = await this.findModeBySlug(depSlug)
                if (!dependency) {
                    errors.push({
                        field: 'dependencies',
                        message: `Dependency not found: ${depSlug}`,
                        severity: 'error'
                    })
                }

                // 检查循环依赖
                if (await this.hasCircularDependency(mode.slug, depSlug)) {
                    errors.push({
                        field: 'dependencies',
                        message: `Circular dependency detected: ${mode.slug} -> ${depSlug}`,
                        severity: 'error'
                    })
                }
            } catch (error) {
                errors.push({
                    field: 'dependencies',
                    message: `Failed to validate dependency ${depSlug}: ${error.message}`,
                    severity: 'error'
                })
            }
        }

        return { isValid: errors.length === 0, errors }
    }

    private async hasCircularDependency(modeSlug: string, depSlug: string): Promise<boolean> {
        // 简化的循环依赖检测
        const visited = new Set<string>()
        const stack = new Set<string>()

        const hasCycle = async (current: string): Promise<boolean> => {
            if (stack.has(current)) return true
            if (visited.has(current)) return false

            visited.add(current)
            stack.add(current)

            const mode = await this.findModeBySlug(current)
            if (mode && mode.dependencies) {
                for (const dep of mode.dependencies) {
                    if (await hasCycle(dep)) return true
                }
            }

            stack.delete(current)
            return false
        }

        return hasCycle(depSlug)
    }

    // 注册自定义验证器
    registerCustomValidator(name: string, validator: CustomValidator): void {
        this.customValidators.set(name, validator)
    }

    // 卸载自定义验证器
    unregisterCustomValidator(name: string): void {
        this.customValidators.delete(name)
    }
}
```

## 3. 模板系统

### 3.1 模板管理器实现

```typescript
// src/core/config/ModeTemplateManager.ts
export class ModeTemplateManager {
    private templates: Map<string, ModeTemplate> = new Map()
    private templateCategories: Map<string, string[]> = new Map()

    constructor(context: vscode.ExtensionContext) {
        this.initializeDefaultTemplates()
        this.loadCustomTemplates()
    }

    private initializeDefaultTemplates(): void {
        // Web 开发模式模板
        this.registerTemplate({
            id: 'web-developer',
            name: 'Web Developer',
            description: 'Specialized mode for web development tasks',
            category: 'development',
            variables: [
                {
                    name: 'frontendFramework',
                    type: 'select',
                    options: ['react', 'vue', 'angular', 'svelte'],
                    default: 'react',
                    required: true,
                    description: 'Primary frontend framework'
                },
                {
                    name: 'backendLanguage',
                    type: 'select',
                    options: ['javascript', 'typescript', 'python', 'java'],
                    default: 'typescript',
                    required: true,
                    description: 'Backend programming language'
                }
            ],
            conditions: [
                {
                    type: 'file_exists',
                    pattern: 'package\\.json'
                }
            ],
            actions: [
                {
                    type: 'set_tools',
                    tools: ['read', 'edit', 'browser', 'command']
                },
                {
                    type: 'set_instructions',
                    instructions: `You are a web development specialist focusing on {{frontendFramework}} and {{backendLanguage}}.
                    Your expertise includes:
                    - {{frontendFramework}} component development
                    - API design and implementation
                    - Database integration
                    - Testing and deployment
                    - Performance optimization`
                }
            ]
        })

        // 数据科学模式模板
        this.registerTemplate({
            id: 'data-scientist',
            name: 'Data Scientist',
            description: 'Specialized mode for data science and machine learning',
            category: 'data-science',
            variables: [
                {
                    name: 'primaryLanguage',
                    type: 'select',
                    options: ['python', 'r', 'julia'],
                    default: 'python',
                    required: true
                },
                {
                    name: 'specialization',
                    type: 'select',
                    options: ['machine-learning', 'data-analysis', 'visualization', 'deep-learning'],
                    default: 'machine-learning'
                }
            ],
            actions: [
                {
                    type: 'set_tools',
                    tools: ['read', 'edit', 'browser', 'command']
                },
                {
                    type: 'set_file_patterns',
                    allowed: ['.*\\.(py|ipynb|r|jl|csv|json)$']
                }
            ]
        })

        // DevOps 工程师模板
        this.registerTemplate({
            id: 'devops-engineer',
            name: 'DevOps Engineer',
            description: 'Specialized mode for DevOps and infrastructure tasks',
            category: 'devops',
            variables: [
                {
                    name: 'orchestration',
                    type: 'select',
                    options: ['kubernetes', 'docker', 'aws', 'azure'],
                    default: 'docker'
                },
                {
                    name: 'ciTool',
                    type: 'select',
                    options: ['github-actions', 'jenkins', 'gitlab-ci', 'circleci'],
                    default: 'github-actions'
                }
            ],
            actions: [
                {
                    type: 'set_tools',
                    tools: ['read', 'edit', 'browser', 'command']
                },
                {
                    type: 'set_file_patterns',
                    allowed: ['.*\\.(yaml|yml|dockerfile|sh|tf|json)$']
                }
            ]
        })
    }

    async processTemplates(config: AdvancedModeConfig): Promise<AdvancedModeConfig> {
        let processedConfig = { ...config }

        if (config.templateId) {
            const template = await this.getTemplate(config.templateId)
            if (template) {
                processedConfig = await this.applyTemplate(template, config)
            }
        }

        return processedConfig
    }

    private async applyTemplate(
        template: ModeTemplate,
        config: AdvancedModeConfig
    ): Promise<AdvancedModeConfig> {
        const processedConfig = { ...config }

        // 处理模板变量
        for (const variable of template.variables) {
            const value = config.variables?.[variable.name] || variable.default

            if (variable.required && !value) {
                throw new Error(`Required variable missing: ${variable.name}`)
            }

            // 应用变量替换
            processedConfig = this.applyVariableReplacement(processedConfig, variable.name, value)
        }

        // 执行模板动作
        for (const action of template.actions) {
            processedConfig = await this.executeTemplateAction(action, processedConfig)
        }

        return processedConfig
    }

    private applyVariableReplacement(
        config: AdvancedModeConfig,
        variableName: string,
        value: any
    ): AdvancedModeConfig {
        const replacement = `{{${variableName}}}`
        const replacementValue = String(value)

        // 在字符串字段中替换变量
        const replaceInObject = (obj: any): any => {
            if (typeof obj === 'string') {
                return obj.replace(new RegExp(replacement, 'g'), replacementValue)
            } else if (Array.isArray(obj)) {
                return obj.map(item => replaceInObject(item))
            } else if (obj && typeof obj === 'object') {
                const result: any = {}
                for (const [key, value] of Object.entries(obj)) {
                    result[key] = replaceInObject(value)
                }
                return result
            }
            return obj
        }

        return replaceInObject(config)
    }

    private async executeTemplateAction(
        action: TemplateAction,
        config: AdvancedModeConfig
    ): Promise<AdvancedModeConfig> {
        switch (action.type) {
            case 'set_tools':
                return {
                    ...config,
                    tools: action.tools
                }
            case 'set_instructions':
                return {
                    ...config,
                    customInstructions: action.instructions
                }
            case 'set_file_patterns':
                return {
                    ...config,
                    allowedFilePaths: action.allowed,
                    disallowedFilePaths: action.disallowed
                }
            case 'set_api_config':
                return {
                    ...config,
                    apiBase: action.apiBase,
                    apiModel: action.apiModel,
                    apiProvider: action.apiProvider
                }
            default:
                return config
        }
    }

    async validateTemplateConditions(
        template: ModeTemplate,
        variables: Record<string, any>
    ): Promise<void> {
        for (const condition of template.conditions) {
            await this.validateCondition(condition, variables)
        }
    }

    private async validateCondition(
        condition: TemplateCondition,
        variables: Record<string, any>
    ): Promise<void> {
        switch (condition.type) {
            case 'file_exists':
                const workspaceFolders = vscode.workspace.workspaceFolders
                if (workspaceFolders && workspaceFolders.length > 0) {
                    const files = await vscode.workspace.findFiles(condition.pattern)
                    if (files.length === 0) {
                        throw new Error(`Required file pattern not found: ${condition.pattern}`)
                    }
                }
                break

            case 'variable_equals':
                const value = variables[condition.variable]
                if (value !== condition.expected) {
                    throw new Error(`Variable ${condition.variable} must be ${condition.expected}`)
                }
                break

            case 'tool_available':
                // 检查工具是否可用
                break
        }
    }

    registerTemplate(template: ModeTemplate): void {
        this.templates.set(template.id, template)

        // 分类管理
        if (!this.templateCategories.has(template.category)) {
            this.templateCategories.set(template.category, [])
        }
        this.templateCategories.get(template.category)!.push(template.id)
    }

    async getTemplate(templateId: string): Promise<ModeTemplate | undefined> {
        return this.templates.get(templateId)
    }

    async getTemplatesByCategory(category: string): Promise<ModeTemplate[]> {
        const templateIds = this.templateCategories.get(category) || []
        const templates: ModeTemplate[] = []

        for (const id of templateIds) {
            const template = await this.getTemplate(id)
            if (template) {
                templates.push(template)
            }
        }

        return templates
    }

    async getAllTemplates(): Promise<ModeTemplate[]> {
        return Array.from(this.templates.values())
    }
}
```

## 4. 钩子系统

### 4.1 模式钩子管理器

```typescript
// src/core/config/ModeHooksManager.ts
export class ModeHooksManager {
    private hooks: Map<string, ModeHook[]> = new Map()
    private activeHooks: Map<string, HookSubscription[]> = new Map()

    constructor(context: vscode.ExtensionContext) {
        this.initializeDefaultHooks()
    }

    private initializeDefaultHooks(): void {
        // 文件访问安全钩子
        this.registerHook('file_access', {
            name: 'security_check',
            priority: 100,
            async execute(context: HookContext): Promise<HookResult> {
                const { filePath, action } = context.params

                // 检查敏感文件
                const sensitivePatterns = [
                    /.*\/\.env$/,
                    /.*\/config\/.*\.json$/,
                    /.*\/secrets\..*$/,
                    /.*\/private_key.*/
                ]

                for (const pattern of sensitivePatterns) {
                    if (pattern.test(filePath)) {
                        return {
                            success: false,
                            message: `Access to sensitive file denied: ${filePath}`
                        }
                    }
                }

                return { success: true }
            }
        })

        // 工具使用前验证
        this.registerHook('before_tool_use', {
            name: 'tool_validation',
            priority: 90,
            async execute(context: HookContext): Promise<HookResult> {
                const { toolName, params } = context.params

                // 特殊工具验证
                if (toolName === 'command') {
                    const command = params.command
                    if (this.isDangerousCommand(command)) {
                        return {
                            success: false,
                            message: `Dangerous command blocked: ${command}`,
                            requiresApproval: true
                        }
                    }
                }

                return { success: true }
            }
        })

        // 模式切换前清理
        this.registerHook('before_mode_switch', {
            name: 'cleanup_check',
            priority: 80,
            async execute(context: HookContext): Promise<HookResult> {
                const { fromMode, toMode } = context.params

                // 检查是否有未保存的更改
                const hasUnsavedChanges = await this.checkUnsavedChanges()
                if (hasUnsavedChanges) {
                    return {
                        success: false,
                        message: 'Cannot switch mode: unsaved changes detected',
                        requiresConfirmation: true
                    }
                }

                return { success: true }
            }
        })
    }

    async registerHook(
        event: HookEvent,
        hook: ModeHook
    ): Promise<void> {
        if (!this.hooks.has(event)) {
            this.hooks.set(event, [])
        }

        this.hooks.get(event)!.push(hook)

        // 按优先级排序
        this.hooks.get(event)!.sort((a, b) => b.priority - a.priority)
    }

    async executeHooks(
        event: HookEvent,
        context: HookContext
    ): Promise<HookExecutionResult> {
        const hooks = this.hooks.get(event) || []
        const results: HookResult[] = []

        for (const hook of hooks) {
            try {
                const result = await hook.execute(context)
                results.push(result)

                // 如果钩子失败且不继续执行
                if (!result.success && !result.continueOnFailure) {
                    return {
                        success: false,
                        results,
                        failedHook: hook.name,
                        message: result.message
                    }
                }

                // 如果需要用户确认
                if (result.requiresApproval || result.requiresConfirmation) {
                    return {
                        success: false,
                        results,
                        requiresApproval: result.requiresApproval,
                        requiresConfirmation: result.requiresConfirmation,
                        message: result.message
                    }
                }
            } catch (error) {
                results.push({
                    success: false,
                    message: `Hook execution failed: ${error.message}`
                })

                // 如果钩子抛出异常，默认停止执行
                return {
                    success: false,
                    results,
                    failedHook: hook.name,
                    message: `Hook ${hook.name} failed: ${error.message}`
                }
            }
        }

        return {
            success: true,
            results
        }
    }

    async executeModeHooks(
        mode: CustomMode,
        event: HookEvent,
        context: HookContext
    ): Promise<HookExecutionResult> {
        // 执行全局钩子
        const globalResult = await this.executeHooks(event, context)
        if (!globalResult.success) {
            return globalResult
        }

        // 执行模式特定钩子
        if (mode.hooks) {
            const modeHooks = this.getModeHooks(mode, event)
            for (const hook of modeHooks) {
                try {
                    const result = await hook.execute(context)
                    if (!result.success && !result.continueOnFailure) {
                        return {
                            success: false,
                            message: result.message
                        }
                    }
                } catch (error) {
                    return {
                        success: false,
                        message: `Mode hook failed: ${error.message}`
                    }
                }
            }
        }

        return { success: true }
    }

    private getModeHooks(mode: CustomMode, event: HookEvent): ModeHook[] {
        if (!mode.hooks) return []

        const hookMethods: Record<HookEvent, keyof ModeHooks> = {
            before_mode_switch: 'onBeforeModeSwitch',
            after_mode_switch: 'onAfterModeSwitch',
            before_tool_use: 'onBeforeToolUse',
            after_tool_use: 'onAfterToolUse',
            file_access: 'onFileAccess'
        }

        const methodName = hookMethods[event]
        if (!methodName || !mode.hooks[methodName]) {
            return []
        }

        // 将模式钩子转换为标准钩子格式
        return [
            {
                name: `${mode.slug}_${methodName}`,
                priority: 50,
                execute: mode.hooks[methodName]!
            }
        ]
    }

    private isDangerousCommand(command: string): boolean {
        const dangerousPatterns = [
            /^rm\s+-rf/,
            /^sudo\s+rm/,
            /^format\s+[a-z]:/,
            /^del\s+\/[sfq]/,
            /^shutdown/,
            /^reboot/,
            /^passwd/,
            /^userdel/,
            /^groupdel/
        ]

        return dangerousPatterns.some(pattern => pattern.test(command))
    }

    private async checkUnsavedChanges(): Promise<boolean> {
        // 检查 VS Code 中是否有未保存的文件
        const workspace = vscode.workspace
        const textDocuments = workspace.textDocuments

        return textDocuments.some(doc => doc.isDirty)
    }
}
```

## 5. 市场集成系统

### 5.1 市场客户端实现

```typescript
// src/core/config/MarketplaceClient.ts
export class MarketplaceClient {
    private apiBase: string
    private apiKey?: string
    private cache: Map<string, MarketplaceCacheEntry> = new Map()

    constructor(context: vscode.ExtensionContext) {
        this.apiBase = 'https://marketplace.kilocode.ai/api/v1'
        this.loadConfiguration()
    }

    private async loadConfiguration(): Promise<void> {
        const config = vscode.workspace.getConfiguration('kilocode')
        this.apiKey = config.get('marketplace.apiKey')
    }

    async publishMode(modeData: MarketplaceModeData): Promise<MarketplacePublishResult> {
        try {
            const response = await fetch(`${this.apiBase}/modes`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${this.apiKey}`,
                    'User-Agent': 'KiloCode Extension v1.0'
                },
                body: JSON.stringify(modeData)
            })

            if (!response.ok) {
                const error = await response.json()
                throw new Error(error.message || 'Failed to publish mode')
            }

            const result = await response.json()
            return {
                success: true,
                marketplaceId: result.id,
                publishedAt: new Date(),
                url: `${this.apiBase}/modes/${result.id}`
            }
        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    async getModeInfo(marketplaceId: string): Promise<MarketplaceModeInfo> {
        // 检查缓存
        const cached = this.cache.get(`mode_info:${marketplaceId}`)
        if (cached && !this.isCacheExpired(cached)) {
            return cached.data
        }

        try {
            const response = await fetch(`${this.apiBase}/modes/${marketplaceId}`, {
                headers: {
                    'User-Agent': 'KiloCode Extension v1.0'
                }
            })

            if (!response.ok) {
                throw new Error(`Mode not found: ${marketplaceId}`)
            }

            const modeInfo = await response.json()

            // 缓存结果
            this.cache.set(`mode_info:${marketplaceId}`, {
                data: modeInfo,
                timestamp: Date.now(),
                ttl: 3600000 // 1小时缓存
            })

            return modeInfo
        } catch (error) {
            throw new Error(`Failed to get mode info: ${error.message}`)
        }
    }

    async searchModes(query: MarketplaceSearchQuery): Promise<MarketplaceSearchResult> {
        const params = new URLSearchParams()

        if (query.q) params.append('q', query.q)
        if (query.category) params.append('category', query.category)
        if (query.tags) params.append('tags', query.tags.join(','))
        if (query.sort) params.append('sort', query.sort)
        if (query.limit) params.append('limit', query.limit.toString())
        if (query.offset) params.append('offset', query.offset.toString())

        try {
            const response = await fetch(`${this.apiBase}/modes/search?${params}`, {
                headers: {
                    'User-Agent': 'KiloCode Extension v1.0'
                }
            })

            if (!response.ok) {
                throw new Error('Search failed')
            }

            return await response.json()
        } catch (error) {
            throw new Error(`Failed to search modes: ${error.message}`)
        }
    }

    async downloadMode(marketplaceId: string): Promise<Buffer> {
        try {
            const response = await fetch(`${this.apiBase}/modes/${marketplaceId}/download`, {
                headers: {
                    'Authorization': `Bearer ${this.apiKey}`,
                    'User-Agent': 'KiloCode Extension v1.0'
                }
            })

            if (!response.ok) {
                throw new Error(`Failed to download mode: ${marketplaceId}`)
            }

            return Buffer.from(await response.arrayBuffer())
        } catch (error) {
            throw new Error(`Failed to download mode: ${error.message}`)
        }
    }

    async getPopularModes(limit: number = 10): Promise<MarketplaceModeInfo[]> {
        try {
            const response = await fetch(`${this.apiBase}/modes/popular?limit=${limit}`, {
                headers: {
                    'User-Agent': 'KiloCode Extension v1.0'
                }
            })

            if (!response.ok) {
                throw new Error('Failed to get popular modes')
            }

            return await response.json()
        } catch (error) {
            throw new Error(`Failed to get popular modes: ${error.message}`)
        }
    }

    async getUserModes(): Promise<MarketplaceModeInfo[]> {
        if (!this.apiKey) {
            throw new Error('API key required for user modes')
        }

        try {
            const response = await fetch(`${this.apiBase}/user/modes`, {
                headers: {
                    'Authorization': `Bearer ${this.apiKey}`,
                    'User-Agent': 'KiloCode Extension v1.0'
                }
            })

            if (!response.ok) {
                throw new Error('Failed to get user modes')
            }

            return await response.json()
        } catch (error) {
            throw new Error(`Failed to get user modes: ${error.message}`)
        }
    }

    async installMode(marketplaceId: string): Promise<InstallResult> {
        try {
            // 下载模式包
            const modePackage = await this.downloadMode(marketplaceId)

            // 解析模式包
            const modeData = await this.parseModePackage(modePackage)

            // 安装模式
            const modesManager = new CustomModesManager(this.context)
            const installedMode = await modesManager.importMode(modePackage, 'package')

            return {
                success: true,
                mode: installedMode,
                marketplaceId
            }
        } catch (error) {
            return {
                success: false,
                error: error.message
            }
        }
    }

    private async parseModePackage(packageData: Buffer): Promise<any> {
        // 这里应该实现包解析逻辑
        // 可能是 ZIP 格式或其他压缩格式
        throw new Error('Package parsing not implemented')
    }

    private isCacheExpired(entry: MarketplaceCacheEntry): boolean {
        return Date.now() - entry.timestamp > entry.ttl
    }
}
```

## 6. 实际应用示例

### 6.1 创建专业开发模式

```typescript
// 示例：创建一个专业的 React 开发模式
async function createReactDeveloperMode(): Promise<CustomMode> {
    const modesManager = new CustomModesManager(context)

    const reactModeConfig: AdvancedModeConfig = {
        slug: 'react-specialist',
        name: 'React Specialist',
        role: `You are a React development expert with deep knowledge of:
- React hooks and component architecture
- State management (Redux, Zustand, Context API)
- Performance optimization and code splitting
- Testing with Jest and React Testing Library
- Build tools and deployment strategies`,
        description: 'Specialized mode for React application development',
        customInstructions: `As a React Specialist, focus on:
1. Writing clean, reusable React components
2. Implementing efficient state management
3. Optimizing performance and bundle size
4. Following React best practices and patterns
5. Ensuring accessibility and responsive design`,
        tools: ['read', 'edit', 'browser', 'command'],
        allowedFilePaths: [
            '.*\\.(js|jsx|ts|tsx)$',
            '.*\\.(css|scss|sass)$',
            '.*\\.(json)$'
        ],
        disallowedFilePaths: [
            '.*\\.(py|java|cpp|c)$',
            'node_modules.*',
            'dist.*',
            'build.*'
        ],
        variables: {
            preferredStateManagement: 'zustand',
            testingFramework: 'jest',
            stylingApproach: 'css-modules'
        },
        hooks: {
            onBeforeModeSwitch: async () => {
                // 检查是否有 React 项目
                const packageJson = await getPackageJson()
                if (!packageJson?.dependencies?.react) {
                    throw new Error('React Specialist mode requires a React project')
                }
            },
            onBeforeToolUse: async (toolName: string, params: any) => {
                if (toolName === 'command' && params.command.includes('npm run')) {
                    // 检查 package.json 中是否存在对应的脚本
                    const packageJson = await getPackageJson()
                    const scriptName = params.command.replace('npm run ', '')
                    if (!packageJson?.scripts?.[scriptName]) {
                        throw new Error(`Script "${scriptName}" not found in package.json`)
                    }
                }
            }
        },
        validation: {
            requiredTools: ['read', 'edit'],
            filePatternRules: [
                {
                    pattern: 'package\\.json',
                    required: true,
                    description: 'React projects require package.json'
                }
            ]
        }
    }

    return await modesManager.createAdvancedCustomMode(reactModeConfig)
}
```

### 6.2 创建团队协作模式

```typescript
// 示例：创建团队协作的代码审查模式
async function createCodeReviewMode(): Promise<CustomMode> {
    const modesManager = new CustomModesManager(context)

    const reviewModeConfig: AdvancedModeConfig = {
        slug: 'code-reviewer',
        name: 'Code Reviewer',
        role: `You are an expert code reviewer focused on:
- Code quality and best practices
- Security vulnerabilities and potential issues
- Performance optimizations
- Maintainability and readability
- Testing coverage and quality`,
        description: 'Specialized mode for thorough code review and analysis',
        customInstructions: `As a Code Reviewer:
1. Provide detailed, constructive feedback
2. Identify potential bugs and security issues
3. Suggest performance improvements
4. Ensure code follows project standards
5. Check for proper error handling and edge cases`,
        tools: ['read', 'browser'], // 只读权限，防止意外修改
        allowedFilePaths: [
            '.*\\.(js|jsx|ts|tsx|py|java|cpp|c|go|rs)$',
            '.*\\.(md|txt)$'
        ],
        templateId: 'code-review-template',
        hooks: {
            onFileAccess: async (filePath: string, action: string) => {
                if (action === 'write') {
                    throw new Error('Code Reviewer mode is read-only')
                }
                return true
            }
        }
    }

    return await modesManager.createAdvancedCustomMode(reviewModeConfig)
}
```

## 总结

KiloCode 的自定义模式系统展现了一个功能丰富、架构完善的 AI 助手定制平台：

### 1. **强大的定制能力**
- 完整的模式生命周期管理
- 灵活的模板系统和变量替换
- 细粒度的权限控制和文件路径限制
- 丰富的钩子系统实现业务逻辑

### 2. **企业级功能**
- 模式版本管理和依赖解析
- 完整的验证和安全检查机制
- 市场集成和分享功能
- 导入导出和团队协作支持

### 3. **优秀的扩展性**
- 插件化的验证器系统
- 可配置的钩子和事件处理
- 模块化的架构设计
- 开放的 API 和扩展接口

### 4. **用户友好**
- 直观的配置界面
- 详细的错误信息和验证反馈
- 丰富的模板和预设
- 完整的文档和示例

这个系统不仅满足了基础的 AI 助手定制需求，更提供了一个完整的 AI 模式生态系统，让用户可以创建、分享和使用各种专业的 AI 助手模式，极大地扩展了 KiloCode 的应用场景和价值。