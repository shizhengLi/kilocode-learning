# 代码生成核心算法分析

> 深入解析 KiloCode 的提示词工程、上下文管理和代码质量保证机制

## 代码生成架构概览

KiloCode 的代码生成系统是一个复杂的智能工程，结合了先进的 AI 模型、精心设计的提示词策略和完善的上下文管理机制。这个系统旨在生成高质量、可执行、且符合最佳实践的代码。

### 核心设计原则

1. **智能上下文管理**：动态选择最相关的代码上下文
2. **分层提示词策略**：根据任务类型选择最适合的提示词模板
3. **质量保证机制**：多层验证确保生成代码的正确性
4. **性能优化**：平衡代码质量和生成速度

## 提示词工程系统

### 提示词模板架构

#### 1. 分层模板设计

```typescript
// src/core/prompts/PromptTemplate.ts
export interface PromptTemplate {
    id: string
    name: string
    description: string
    category: PromptCategory
    systemPrompt: string
    userPromptTemplate: string
    variables: PromptVariable[]
    expectedOutputType: OutputType
    codeStyle: CodeStyle
    complexity: ComplexityLevel
    tags: string[]
    metadata: PromptMetadata
}

export class PromptTemplateManager {
    private templates: Map<string, PromptTemplate> = new Map()
    private fallbackTemplate: PromptTemplate

    constructor() {
        this.initializeTemplates()
        this.fallbackTemplate = this.createFallbackTemplate()
    }

    private initializeTemplates(): void {
        // 代码生成模板
        this.templates.set("code-generation-basic", {
            id: "code-generation-basic",
            name: "基础代码生成",
            description: "生成简单函数和类",
            category: PromptCategory.CODE_GENERATION,
            systemPrompt: `你是一个专业的软件开发助手，擅长生成高质量、可维护的代码。

**代码风格要求：**
- 使用现代编程语言特性
- 遵循最佳实践和设计模式
- 添加适当的错误处理
- 保持代码简洁和可读性
- 添加必要的注释

**输出格式：**
- 只返回代码，不需要额外解释
- 使用 markdown 代码块格式
- 确保代码语法正确`,
            userPromptTemplate: `请生成一个{{language}}的{{codeType}}，功能要求：{{requirements}}

上下文信息：
{{context}}

代码风格要求：{{styleRequirements}}`,
            variables: [
                { name: "language", type: "string", required: true, description: "编程语言" },
                { name: "codeType", type: "string", required: true, description: "代码类型" },
                { name: "requirements", type: "string", required: true, description: "功能需求" },
                { name: "context", type: "string", required: false, description: "上下文信息" },
                { name: "styleRequirements", type: "string", required: false, description: "风格要求" }
            ],
            expectedOutputType: OutputType.CODE,
            codeStyle: CodeStyle.MODERN,
            complexity: ComplexityLevel.BASIC,
            tags: ["generation", "basic"],
            metadata: {
                version: "1.0.0",
                author: "KiloCode Team",
                lastUpdated: new Date(),
                usageCount: 0,
                successRate: 0.95
            }
        })

        // 算法实现模板
        this.templates.set("algorithm-implementation", {
            id: "algorithm-implementation",
            name: "算法实现",
            description: "实现复杂算法和数据结构",
            category: PromptCategory.ALGORITHM,
            systemPrompt: `你是一个算法专家，专门实现高效的算法和数据结构。

**算法要求：**
- 选择最优的时间复杂度和空间复杂度
- 考虑边界条件和异常情况
- 提供清晰的算法思路
- 包含时间复杂度分析

**实现标准：**
- 代码要经过充分测试
- 考虑性能优化
- 使用合适的数据结构
- 添加必要的注释说明`,
            userPromptTemplate: `请实现{{algorithmName}}算法。

**算法描述：**
{{algorithmDescription}}

**输入输出规格：**
- 输入：{{inputSpec}}
- 输出：{{outputSpec}}

**约束条件：**
{{constraints}}

**性能要求：**
{{performanceRequirements}}`,
            variables: [
                { name: "algorithmName", type: "string", required: true },
                { name: "algorithmDescription", type: "string", required: true },
                { name: "inputSpec", type: "string", required: true },
                { name: "outputSpec", type: "string", required: true },
                { name: "constraints", type: "string", required: false },
                { name: "performanceRequirements", type: "string", required: false }
            ],
            expectedOutputType: OutputType.CODE_WITH_EXPLANATION,
            codeStyle: CodeStyle.ALGORITHMIC,
            complexity: ComplexityLevel.ADVANCED,
            tags: ["algorithm", "implementation"],
            metadata: {
                version: "1.0.0",
                author: "KiloCode Team",
                lastUpdated: new Date(),
                usageCount: 0,
                successRate: 0.90
            }
        })

        // 代码重构模板
        this.templates.set("code-refactoring", {
            id: "code-refactoring",
            name: "代码重构",
            description: "改进现有代码结构和质量",
            category: PromptCategory.REFACTORING,
            systemPrompt: `你是一个代码重构专家，专门改善代码质量和可维护性。

**重构原则：**
- 保持功能不变的前提下改善代码结构
- 提高代码可读性和可维护性
- 应用设计模式和最佳实践
- 优化性能和资源使用

**重构重点：**
- 消除代码重复
- 简化复杂逻辑
- 改善命名和结构
- 增强错误处理`,
            userPromptTemplate: `请重构以下代码：

**原始代码：**
\`\`\`{{language}}
{{originalCode}}
\`\`\`

**重构要求：**
{{refactoringRequirements}}

**现有问题：**
{{currentIssues}}

**期望改进：**
{{desiredImprovements}}`,
            variables: [
                { name: "language", type: "string", required: true },
                { name: "originalCode", type: "string", required: true },
                { name: "refactoringRequirements", type: "string", required: false },
                { name: "currentIssues", type: "string", required: false },
                { name: "desiredImprovements", type: "string", required: false }
            ],
            expectedOutputType: OutputType.CODE_WITH_EXPLANATION,
            codeStyle: CodeStyle.CLEAN,
            complexity: ComplexityLevel.INTERMEDIATE,
            tags: ["refactoring", "improvement"],
            metadata: {
                version: "1.0.0",
                author: "KiloCode Team",
                lastUpdated: new Date(),
                usageCount: 0,
                successRate: 0.92
            }
        })
    }

    private createFallbackTemplate(): PromptTemplate {
        return {
            id: "fallback-template",
            name: "通用回退模板",
            description: "当没有找到合适模板时使用的通用模板",
            category: PromptCategory.GENERAL,
            systemPrompt: `你是一个专业的编程助手，请根据用户需求生成高质量的代码。

请确保：
1. 代码语法正确
2. 逻辑清晰
3. 包含必要的错误处理
4. 遵循最佳实践`,
            userPromptTemplate: "请完成以下任务：{{task}}",
            variables: [
                { name: "task", type: "string", required: true }
            ],
            expectedOutputType: OutputType.CODE,
            codeStyle: CodeStyle.GENERAL,
            complexity: ComplexityLevel.BASIC,
            tags: ["fallback", "general"],
            metadata: {
                version: "1.0.0",
                author: "KiloCode Team",
                lastUpdated: new Date(),
                usageCount: 0,
                successRate: 0.80
            }
        }
    }

    // 选择最适合的模板
    selectTemplate(
        taskType: TaskType,
        complexity: ComplexityLevel,
        language: string,
        tags: string[]
    ): PromptTemplate {
        // 基于任务类型筛选
        const categoryTemplates = Array.from(this.templates.values()).filter(
            template => this.isCategoryMatch(template.category, taskType)
        )

        // 基于复杂度筛选
        const complexityTemplates = categoryTemplates.filter(
            template => template.complexity <= complexity
        )

        // 基于标签匹配
        const tagScores = new Map<string, number>()
        for (const template of complexityTemplates) {
            const score = this.calculateTagScore(template.tags, tags)
            tagScores.set(template.id, score)
        }

        // 选择最高分的模板
        let bestTemplate: PromptTemplate | null = null
        let bestScore = -1

        for (const template of complexityTemplates) {
            const score = tagScores.get(template.id) || 0
            if (score > bestScore) {
                bestScore = score
                bestTemplate = template
            }
        }

        return bestTemplate || this.fallbackTemplate
    }

    private isCategoryMatch(category: PromptCategory, taskType: TaskType): boolean {
        const mapping = {
            [PromptCategory.CODE_GENERATION]: [TaskType.CODE_GENERATION],
            [PromptCategory.ALGORITHM]: [TaskType.CODE_GENERATION],
            [PromptCategory.REFACTORING]: [TaskType.CODE_REFACTORING],
            [PromptCategory.DEBUGGING]: [TaskType.DEBUGGING],
            [PromptCategory.TESTING]: [TaskType.TESTING],
            [PromptCategory.DOCUMENTATION]: [TaskType.DOCUMENTATION],
            [PromptCategory.GENERAL]: [TaskType.GENERAL]
        }

        return mapping[category]?.includes(taskType) || false
    }

    private calculateTagScore(templateTags: string[], requestTags: string[]): number {
        let score = 0
        for (const tag of requestTags) {
            if (templateTags.includes(tag)) {
                score += 1
            }
        }
        return score
    }

    // 渲染模板
    renderTemplate(template: PromptTemplate, variables: Record<string, string>): string {
        let rendered = template.userPromptTemplate

        // 替换变量
        for (const [key, value] of Object.entries(variables)) {
            const placeholder = `{{${key}}}`
            rendered = rendered.replace(new RegExp(placeholder, 'g'), value)
        }

        // 验证所有必需变量都已提供
        const missingVars = template.variables
            .filter(v => v.required && !variables[v.name])
            .map(v => v.name)

        if (missingVars.length > 0) {
            throw new Error(`Missing required variables: ${missingVars.join(', ')}`)
        }

        return rendered
    }

    getTemplate(id: string): PromptTemplate | undefined {
        return this.templates.get(id)
    }

    getAllTemplates(): PromptTemplate[] {
        return Array.from(this.templates.values())
    }

    updateTemplateUsage(id: string, success: boolean): void {
        const template = this.templates.get(id)
        if (template) {
            template.metadata.usageCount++
            if (success) {
                const successCount = template.metadata.successRate * (template.metadata.usageCount - 1)
                template.metadata.successRate = (successCount + 1) / template.metadata.usageCount
            } else {
                const successCount = template.metadata.successRate * (template.metadata.usageCount - 1)
                template.metadata.successRate = successCount / template.metadata.usageCount
            }
            template.metadata.lastUpdated = new Date()
        }
    }
}
```

#### 2. 动态提示词生成器

```typescript
// src/core/prompts/DynamicPromptGenerator.ts
export class DynamicPromptGenerator {
    private templateManager: PromptTemplateManager
    private contextAnalyzer: ContextAnalyzer
    private styleAdapter: CodeStyleAdapter

    constructor() {
        this.templateManager = new PromptTemplateManager()
        this.contextAnalyzer = new ContextAnalyzer()
        this.styleAdapter = new CodeStyleAdapter()
    }

    async generatePrompt(
        request: CodeGenerationRequest
    ): Promise<GeneratedPrompt> {
        const {
            description,
            taskType,
            context,
            language,
            style,
            complexity,
            constraints
        } = request

        // 1. 分析上下文
        const contextAnalysis = await this.contextAnalyzer.analyze(context)

        // 2. 适配代码风格
        const styleRequirements = await this.styleAdapter.adaptStyle(style, language)

        // 3. 选择提示词模板
        const template = this.templateManager.selectTemplate(
            taskType,
            complexity,
            language,
            this.extractTags(description, context)
        )

        // 4. 准备模板变量
        const variables = await this.prepareVariables({
            description,
            context,
            contextAnalysis,
            styleRequirements,
            language,
            constraints
        })

        // 5. 渲染提示词
        const userPrompt = this.templateManager.renderTemplate(template, variables)

        // 6. 生成最终提示词
        const finalPrompt = this.assembleFinalPrompt(template, userPrompt, context)

        return {
            systemPrompt: template.systemPrompt,
            userPrompt,
            finalPrompt,
            templateId: template.id,
            templateMetadata: template.metadata,
            contextTokens: contextAnalysis.estimatedTokens,
            variables
        }
    }

    private async prepareVariables(params: VariablePreparationParams): Promise<Record<string, string>> {
        const {
            description,
            context,
            contextAnalysis,
            styleRequirements,
            language,
            constraints
        } = params

        const variables: Record<string, string> = {}

        // 基础变量
        variables.language = language
        variables.requirements = description

        // 代码类型推断
        variables.codeType = this.inferCodeType(description, context)

        // 上下文格式化
        if (context && Object.keys(context).length > 0) {
            variables.context = this.formatContext(context, contextAnalysis)
        }

        // 风格要求
        variables.styleRequirements = this.formatStyleRequirements(styleRequirements)

        // 算法特定变量
        if (this.isAlgorithmTask(description)) {
            Object.assign(variables, await this.prepareAlgorithmVariables(description, context))
        }

        // 重构特定变量
        if (this.isRefactoringTask(description)) {
            Object.assign(variables, await this.prepareRefactoringVariables(description, context))
        }

        // 约束条件
        if (constraints) {
            variables.constraints = this.formatConstraints(constraints)
        }

        return variables
    }

    private inferCodeType(description: string, context?: TaskContext): string {
        const lowerDesc = description.toLowerCase()

        // 基于关键词推断
        if (lowerDesc.includes('function') || lowerDesc.includes('方法')) {
            return 'function'
        } else if (lowerDesc.includes('class') || lowerDesc.includes('类')) {
            return 'class'
        } else if (lowerDesc.includes('interface') || lowerDesc.includes('接口')) {
            return 'interface'
        } else if (lowerDesc.includes('algorithm') || lowerDesc.includes('算法')) {
            return 'algorithm'
        } else if (lowerDesc.includes('test') || lowerDesc.includes('测试')) {
            return 'test'
        } else if (lowerDesc.includes('component') || lowerDesc.includes('组件')) {
            return 'component'
        } else {
            return 'code'
        }
    }

    private formatContext(context: TaskContext, analysis: ContextAnalysis): string {
        const parts: string[] = []

        // 文件内容
        if (context.fileContents && Object.keys(context.fileContents).length > 0) {
            parts.push("**相关文件：**")
            for (const [filePath, content] of Object.entries(context.fileContents)) {
                parts.push(`\n文件: ${filePath}`)
                parts.push("```" + this.getFileExtension(filePath))
                parts.push(content)
                parts.push("```")
            }
        }

        // 项目结构
        if (context.projectStructure) {
            parts.push("\n**项目结构：**")
            parts.push("```\n" + context.projectStructure + "\n```")
        }

        // 依赖信息
        if (context.dependencies && context.dependencies.length > 0) {
            parts.push("\n**项目依赖：**")
            parts.push(context.dependencies.join(", "))
        }

        // 配置信息
        if (context.configurations && Object.keys(context.configurations).length > 0) {
            parts.push("\n**配置信息：**")
            for (const [key, value] of Object.entries(context.configurations)) {
                parts.push(`${key}: ${value}`)
            }
        }

        return parts.join("\n")
    }

    private formatStyleRequirements(style: CodeStyleRequirements): string {
        const requirements: string[] = []

        if (style.indentation) {
            requirements.push(`缩进：${style.indentation}`)
        }

        if (style.namingConvention) {
            requirements.push(`命名规范：${style.namingConvention}`)
        }

        if (style.documentation) {
            requirements.push(`文档要求：${style.documentation}`)
        }

        if (style.errorHandling) {
            requirements.push(`错误处理：${style.errorHandling}`)
        }

        if (style.performance) {
            requirements.push(`性能要求：${style.performance}`)
        }

        return requirements.join("；")
    }

    private isAlgorithmTask(description: string): boolean {
        const algorithmKeywords = [
            'algorithm', '算法', 'sort', '排序', 'search', '查找',
            'optimize', '优化', 'complexity', '复杂度',
            'data structure', '数据结构', 'graph', '图'
        ]
        return algorithmKeywords.some(keyword =>
            description.toLowerCase().includes(keyword)
        )
    }

    private isRefactoringTask(description: string): boolean {
        const refactoringKeywords = [
            'refactor', '重构', 'improve', '改进', 'optimize',
            'clean', '清理', 'restructure', '重组'
        ]
        return refactoringKeywords.some(keyword =>
            description.toLowerCase().includes(keyword)
        )
    }

    private async prepareAlgorithmVariables(
        description: string,
        context?: TaskContext
    ): Promise<Record<string, string>> {
        // 使用 AI 分析算法描述
        const analysis = await this.analyzeAlgorithmDescription(description)

        return {
            algorithmName: analysis.name || "未命名算法",
            algorithmDescription: analysis.description || description,
            inputSpec: analysis.inputSpec || "未指定",
            outputSpec: analysis.outputSpec || "未指定",
            constraints: analysis.constraints || "无特殊约束",
            performanceRequirements: analysis.performanceRequirements || "标准性能"
        }
    }

    private async prepareRefactoringVariables(
        description: string,
        context?: TaskContext
    ): Promise<Record<string, string>> {
        // 提取原始代码
        const originalCode = this.extractOriginalCode(description, context)

        return {
            language: this.detectLanguage(originalCode),
            originalCode: originalCode,
            refactoringRequirements: this.extractRefactoringRequirements(description),
            currentIssues: this.identifyCurrentIssues(originalCode),
            desiredImprovements: this.extractDesiredImprovements(description)
        }
    }

    private extractTags(description: string, context?: TaskContext): string[] {
        const tags: string[] = []
        const lowerDesc = description.toLowerCase()

        // 基于描述提取标签
        if (lowerDesc.includes('performance') || lowerDesc.includes('性能')) {
            tags.push('performance')
        }
        if (lowerDesc.includes('security') || lowerDesc.includes('安全')) {
            tags.push('security')
        }
        if (lowerDesc.includes('test') || lowerDesc.includes('测试')) {
            tags.push('testing')
        }
        if (lowerDesc.includes('api') || lowerDesc.includes('接口')) {
            tags.push('api')
        }

        // 基于上下文提取标签
        if (context?.fileExtensions) {
            if (context.fileExtensions.includes('.ts') || context.fileExtensions.includes('.js')) {
                tags.push('typescript', 'javascript')
            }
            if (context.fileExtensions.includes('.py')) {
                tags.push('python')
            }
            if (context.fileExtensions.includes('.java')) {
                tags.push('java')
            }
        }

        return tags
    }

    private assembleFinalPrompt(
        template: PromptTemplate,
        userPrompt: string,
        context?: TaskContext
    ): string {
        let finalPrompt = userPrompt

        // 添加额外上下文
        if (context?.additionalContext) {
            finalPrompt += "\n\n**额外上下文：**\n" + context.additionalContext
        }

        // 添加示例（如果有）
        if (context?.examples && context.examples.length > 0) {
            finalPrompt += "\n\n**参考示例：**\n"
            for (const example of context.examples) {
                finalPrompt += example + "\n\n"
            }
        }

        return finalPrompt
    }

    private getFileExtension(filePath: string): string {
        const match = filePath.match(/\.([^.]+)$/)
        return match ? match[1] : 'text'
    }

    private formatConstraints(constraints: CodeGenerationConstraints): string {
        const constraintList: string[] = []

        if (constraints.maxTokens) {
            constraintList.push(`最大令牌数：${constraints.maxTokens}`)
        }
        if (constraints.maxLines) {
            constraintList.push(`最大行数：${constraints.maxLines}`)
        }
        if (constraints.forbiddenPatterns) {
            constraintList.push(`禁止模式：${constraints.forbiddenPatterns.join(', ')}`)
        }
        if (constraints.requiredPatterns) {
            constraintList.push(`必需模式：${constraints.requiredPatterns.join(', ')}`)
        }

        return constraintList.join('；')
    }

    private async analyzeAlgorithmDescription(description: string): Promise<AlgorithmAnalysis> {
        // 简化的算法分析，实际应用中可以使用 AI 进行更深入的分析
        return {
            name: this.extractAlgorithmName(description),
            description: description,
            inputSpec: this.extractInputSpec(description),
            outputSpec: this.extractOutputSpec(description),
            constraints: this.extractConstraints(description),
            performanceRequirements: this.extractPerformanceRequirements(description)
        }
    }

    private extractAlgorithmName(description: string): string {
        // 简化的实现
        const patterns = [
            /实现\s*([^，\s。]+)/,
            /创建\s*([^，\s。]+)/,
            /开发\s*([^，\s。]+)/
        ]

        for (const pattern of patterns) {
            const match = description.match(pattern)
            if (match) {
                return match[1]
            }
        }

        return "未知算法"
    }

    private extractInputSpec(description: string): string {
        const inputPatterns = [
            /输入[：:]\s*([^，\n。]+)/,
            /input[：:]\s*([^，\n。]+)/i
        ]

        for (const pattern of inputPatterns) {
            const match = description.match(pattern)
            if (match) {
                return match[1]
            }
        }

        return "未明确指定"
    }

    private extractOutputSpec(description: string): string {
        const outputPatterns = [
            /输出[：:]\s*([^，\n。]+)/,
            /output[：:]\s*([^，\n。]+)/i
        ]

        for (const pattern of outputPatterns) {
            const match = description.match(pattern)
            if (match) {
                return match[1]
            }
        }

        return "未明确指定"
    }

    private extractConstraints(description: string): string {
        const constraintPatterns = [
            /约束[：:]\s*([^，\n。]+)/,
            /限制[：:]\s*([^，\n。]+)/,
            /要求[：:]\s*([^，\n。]+)/
        ]

        for (const pattern of constraintPatterns) {
            const match = description.match(pattern)
            if (match) {
                return match[1]
            }
        }

        return "无特殊约束"
    }

    private extractPerformanceRequirements(description: string): string {
        if (description.includes('O(1)') || description.includes('常数时间')) {
            return "常数时间复杂度 O(1)"
        }
        if (description.includes('O(n)') || description.includes('线性时间')) {
            return "线性时间复杂度 O(n)"
        }
        if (description.includes('O(log n)') || description.includes('对数时间')) {
            return "对数时间复杂度 O(log n)"
        }
        if (description.includes('O(n log n)')) {
            return "线性对数时间复杂度 O(n log n)"
        }
        if (description.includes('O(n²)') || description.includes('平方时间')) {
            return "平方时间复杂度 O(n²)"
        }

        return "标准性能要求"
    }

    private extractOriginalCode(description: string, context?: TaskContext): string {
        // 尝试从描述中提取代码块
        const codeBlockPattern = /```[\w]*\n([\s\S]*?)\n```/
        const match = description.match(codeBlockPattern)
        if (match) {
            return match[1]
        }

        // 尝试从上下文中获取代码
        if (context?.fileContents) {
            const firstFile = Object.values(context.fileContents)[0]
            if (firstFile) {
                return firstFile
            }
        }

        return "// 请提供需要重构的代码"
    }

    private detectLanguage(code: string): string {
        // 简单的语言检测
        if (code.includes('function') && code.includes('const')) return 'javascript'
        if (code.includes('def ') && code.includes(':')) return 'python'
        if (code.includes('public class') && code.includes('{')) return 'java'
        if (code.includes('#include') && code.includes('int main')) return 'cpp'
        if (code.includes('package main') && code.includes('func')) return 'go'
        if (code.includes('using System') && code.includes('class')) return 'csharp'

        return 'unknown'
    }

    private extractRefactoringRequirements(description: string): string {
        const patterns = [
            /重构要求[：:]\s*([^，\n。]+)/,
            /需要[：:]\s*([^，\n。]+)/,
            /目标[：:]\s*([^，\n。]+)/
        ]

        for (const pattern of patterns) {
            const match = description.match(pattern)
            if (match) {
                return match[1]
            }
        }

        return "改善代码质量和可维护性"
    }

    private identifyCurrentIssues(code: string): string {
        const issues: string[] = []

        if (code.includes('TODO') || code.includes('FIXME')) {
            issues.push("存在待办事项")
        }
        if (code.length > 1000) {
            issues.push("代码过长")
        }
        if (code.split('\n').length > 50) {
            issues.push("函数过长")
        }

        return issues.length > 0 ? issues.join(', ') : "无明显问题"
    }

    private extractDesiredImprovements(description: string): string {
        const patterns = [
            /期望[：:]\s*([^，\n。]+)/,
            /改进[：:]\s*([^，\n。]+)/,
            /优化[：:]\s*([^，\n。]+)/
        ]

        for (const pattern of patterns) {
            const match = description.match(pattern)
            if (match) {
                return match[1]
            }
        }

        return "提高代码质量和性能"
    }
}
```

## 上下文管理系统

### 上下文分析器

#### 1. 智能上下文选择

```typescript
// src/core/context/ContextAnalyzer.ts
export class ContextAnalyzer {
    private maxContextTokens: number = 128000
    private relevanceThreshold: number = 0.7
    private fileAnalyzer: FileAnalyzer

    constructor() {
        this.fileAnalyzer = new FileAnalyzer()
    }

    async analyze(context: TaskContext): Promise<ContextAnalysis> {
        const analysis: ContextAnalysis = {
            relevantFiles: [],
            estimatedTokens: 0,
            relevanceScores: new Map(),
            contextStructure: {},
            dependencies: [],
            importStatements: [],
            functionDefinitions: [],
            classDefinitions: [],
            typeDefinitions: []
        }

        // 1. 分析文件相关性
        if (context.fileContents) {
            analysis.relevantFiles = await this.analyzeFileRelevance(
                context.fileContents,
                context.description || ''
            )
        }

        // 2. 估算令牌数量
        analysis.estimatedTokens = await this.estimateContextTokens(analysis.relevantFiles)

        // 3. 提取代码结构
        if (analysis.relevantFiles.length > 0) {
            const codeStructure = await this.extractCodeStructure(analysis.relevantFiles)
            Object.assign(analysis, codeStructure)
        }

        // 4. 分析依赖关系
        analysis.dependencies = await this.analyzeDependencies(analysis.relevantFiles)

        return analysis
    }

    private async analyzeFileRelevance(
        fileContents: Record<string, string>,
        description: string
    ): Promise<RelevantFile[]> {
        const relevantFiles: RelevantFile[] = []
        const descEmbedding = await this.getEmbedding(description)

        for (const [filePath, content] of Object.entries(fileContents)) {
            // 计算文件相关性分数
            const relevanceScore = await this.calculateRelevanceScore(
                filePath,
                content,
                descEmbedding
            )

            if (relevanceScore >= this.relevanceThreshold) {
                relevantFiles.push({
                    path: filePath,
                    content,
                    relevanceScore,
                    language: this.detectLanguage(filePath, content),
                    type: this.classifyFileType(filePath),
                    size: content.length,
                    lastModified: await this.getLastModified(filePath)
                })
            }
        }

        // 按相关性排序
        relevantFiles.sort((a, b) => b.relevanceScore - a.relevanceScore)

        // 考虑上下文长度限制
        return this.selectFilesWithinTokenLimit(relevantFiles)
    }

    private async calculateRelevanceScore(
        filePath: string,
        content: string,
        descEmbedding: number[]
    ): Promise<number> {
        let score = 0

        // 1. 基于文件名相似性
        const filenameScore = this.calculateFilenameSimilarity(filePath, descEmbedding)
        score += filenameScore * 0.2

        // 2. 基于内容语义相似性
        const contentEmbedding = await this.getEmbedding(content.substring(0, 1000)) // 取前1000字符
        const semanticScore = this.cosineSimilarity(descEmbedding, contentEmbedding)
        score += semanticScore * 0.4

        // 3. 基于关键词匹配
        const keywordScore = this.calculateKeywordScore(content, descEmbedding)
        score += keywordScore * 0.2

        // 4. 基于代码结构匹配
        const structureScore = await this.calculateStructureScore(content)
        score += structureScore * 0.2

        return Math.min(1, score)
    }

    private calculateFilenameSimilarity(filePath: string, descEmbedding: number[]): number {
        const filename = path.basename(filePath, path.extname(filePath))
        const filenameEmbedding = await this.getEmbedding(filename)
        return this.cosineSimilarity(descEmbedding, filenameEmbedding)
    }

    private calculateKeywordScore(content: string, descEmbedding: number[]): number {
        const keywords = this.extractKeywords(content)
        let totalScore = 0

        for (const keyword of keywords) {
            const keywordEmbedding = await this.getEmbedding(keyword)
            const similarity = this.cosineSimilarity(descEmbedding, keywordEmbedding)
            totalScore += similarity
        }

        return keywords.length > 0 ? totalScore / keywords.length : 0
    }

    private async calculateStructureScore(content: string): Promise<number> {
        // 分析代码结构的相关性
        const structure = await this.analyzeCodeStructure(content)

        // 这里可以根据具体的代码结构特征进行评分
        let score = 0.5 // 基础分数

        // 函数定义加分
        if (structure.functions.length > 0) score += 0.1

        // 类定义加分
        if (structure.classes.length > 0) score += 0.1

        // 接口定义加分
        if (structure.interfaces.length > 0) score += 0.1

        // 类型定义加分
        if (structure.types.length > 0) score += 0.1

        return Math.min(1, score)
    }

    private async extractCodeStructure(relevantFiles: RelevantFile[]): Promise<CodeStructure> {
        const structure: CodeStructure = {
            functions: [],
            classes: [],
            interfaces: [],
            types: [],
            imports: [],
            exports: []
        }

        for (const file of relevantFiles) {
            const fileStructure = await this.fileAnalyzer.analyzeStructure(file.content)

            // 合并结构信息
            structure.functions.push(...fileStructure.functions.map(f => ({
                ...f,
                sourceFile: file.path
            })))

            structure.classes.push(...fileStructure.classes.map(c => ({
                ...c,
                sourceFile: file.path
            })))

            structure.interfaces.push(...fileStructure.interfaces.map(i => ({
                ...i,
                sourceFile: file.path
            })))

            structure.types.push(...fileStructure.types.map(t => ({
                ...t,
                sourceFile: file.path
            })))

            // 去重导入语句
            const uniqueImports = fileStructure.imports.filter(imp =>
                !structure.imports.some(existing => existing.path === imp.path)
            )
            structure.imports.push(...uniqueImports)

            // 去重导出语句
            const uniqueExports = fileStructure.exports.filter(exp =>
                !structure.exports.some(existing => existing.name === exp.name)
            )
            structure.exports.push(...uniqueExports)
        }

        return structure
    }

    private async analyzeDependencies(relevantFiles: RelevantFile[]): Promise<DependencyInfo[]> {
        const dependencies: DependencyInfo[] = []
        const dependencyMap = new Map<string, Set<string>>()

        // 分析每个文件的依赖
        for (const file of relevantFiles) {
            const fileDeps = await this.fileAnalyzer.extractDependencies(file.content)

            for (const dep of fileDeps) {
                if (!dependencyMap.has(dep.source)) {
                    dependencyMap.set(dep.source, new Set())
                }
                dependencyMap.get(dep.source)!.add(dep.target)
            }
        }

        // 转换为依赖信息
        for (const [source, targets] of dependencyMap.entries()) {
            for (const target of targets) {
                dependencies.push({
                    source,
                    target,
                    type: this.inferDependencyType(source, target),
                    strength: this.calculateDependencyStrength(source, target)
                })
            }
        }

        return dependencies
    }

    private selectFilesWithinTokenLimit(files: RelevantFile[]): RelevantFile[] {
        const selectedFiles: RelevantFile[] = []
        let totalTokens = 0

        for (const file of files) {
            const estimatedTokens = Math.ceil(file.content.length / 4) // 粗略估计

            if (totalTokens + estimatedTokens <= this.maxContextTokens) {
                selectedFiles.push(file)
                totalTokens += estimatedTokens
            } else {
                // 如果单个文件太大，尝试截取关键部分
                if (estimatedTokens > this.maxContextTokens) {
                    const truncatedFile = this.truncateFileContent(file, this.maxContextTokens - totalTokens)
                    if (truncatedFile) {
                        selectedFiles.push(truncatedFile)
                        totalTokens += Math.ceil(truncatedFile.content.length / 4)
                    }
                }
                break
            }
        }

        return selectedFiles
    }

    private truncateFileContent(file: RelevantFile, maxTokens: number): RelevantFile | null {
        const maxChars = maxTokens * 4
        const content = file.content

        // 尝试保留最相关的部分
        const importantParts = this.extractImportantParts(content, maxChars)

        if (importantParts.length > 0) {
            return {
                ...file,
                content: importantParts,
                truncated: true
            }
        }

        return null
    }

    private extractImportantParts(content: string, maxLength: number): string {
        // 提取导入语句
        const imports = this.extractImports(content)

        // 提取类和函数定义
        const definitions = this.extractDefinitions(content)

        let result = imports + '\n\n' + definitions

        // 如果还是太长，截取前部分
        if (result.length > maxLength) {
            result = result.substring(0, maxLength - 3) + '...'
        }

        return result
    }

    private extractImports(content: string): string {
        const importPatterns = [
            /import\s+.*?;?/g,
            /require\s*\(.*?\)/g,
            /#include\s*.*?$/gm,
            /using\s+.*?;?/g
        ]

        const imports: string[] = []

        for (const pattern of importPatterns) {
            const matches = content.match(pattern)
            if (matches) {
                imports.push(...matches)
            }
        }

        return imports.join('\n')
    }

    private extractDefinitions(content: string): string {
        const patterns = [
            /(class\s+\w+.*?{[^}]*})/gs,
            /(function\s+\w+.*?\{[^}]*})/gs,
            /(def\s+\w+.*?:[^]*?(?=\n\w|\n\n|$))/gs,
            /(interface\s+\w+.*?{[^}]*})/gs,
            /(type\s+\w+.*?=.*?;)/gs
        ]

        const definitions: string[] = []

        for (const pattern of patterns) {
            const matches = content.match(pattern)
            if (matches) {
                definitions.push(...matches)
            }
        }

        return definitions.join('\n\n')
    }

    private async estimateContextTokens(files: RelevantFile[]): Promise<number> {
        let totalTokens = 0

        for (const file of files) {
            totalTokens += Math.ceil(file.content.length / 4)
        }

        return totalTokens
    }

    private detectLanguage(filePath: string, content: string): string {
        const ext = path.extname(filePath).toLowerCase()

        const languageMap: Record<string, string> = {
            '.js': 'javascript',
            '.ts': 'typescript',
            '.py': 'python',
            '.java': 'java',
            '.cpp': 'cpp',
            '.c': 'c',
            '.go': 'go',
            '.rs': 'rust',
            '.php': 'php',
            '.rb': 'ruby',
            '.swift': 'swift',
            '.kt': 'kotlin',
            '.cs': 'csharp',
            '.html': 'html',
            '.css': 'css',
            '.json': 'json',
            '.xml': 'xml',
            '.yaml': 'yaml',
            '.yml': 'yaml',
            '.md': 'markdown'
        }

        return languageMap[ext] || 'unknown'
    }

    private classifyFileType(filePath: string): FileType {
        const filename = path.basename(filePath).toLowerCase()

        if (filename.includes('test') || filename.includes('spec')) {
            return FileType.TEST
        } else if (filename.includes('config') || filename.includes('setting')) {
            return FileType.CONFIG
        } else if (filename.includes('index') || filename.includes('main')) {
            return FileType.ENTRY_POINT
        } else if (filename.includes('interface') || filename.includes('type')) {
            return FileType.TYPE_DEFINITION
        } else if (filename.includes('util') || filename.includes('helper')) {
            return FileType.UTILITY
        } else if (filename.includes('model') || filename.includes('entity')) {
            return FileType.MODEL
        } else if (filename.includes('service') || filename.includes('controller')) {
            return FileType.SERVICE
        } else {
            return FileType.SOURCE
        }
    }

    private async getLastModified(filePath: string): Promise<Date> {
        try {
            const stats = await fs.stat(filePath)
            return stats.mtime
        } catch {
            return new Date()
        }
    }

    private extractKeywords(content: string): string[] {
        // 简化的关键词提取
        const words = content.toLowerCase()
            .replace(/[^\w\s]/g, ' ')
            .split(/\s+/)
            .filter(word => word.length > 2)

        // 过滤常见停用词
        const stopWords = new Set(['the', 'and', 'for', 'are', 'but', 'not', 'you', 'all', 'can', 'had', 'her', 'was', 'one', 'our', 'out', 'day', 'get', 'has', 'him', 'his', 'how', 'man', 'new', 'now', 'old', 'see', 'two', 'way', 'who', 'boy', 'did', 'its', 'let', 'put', 'say', 'she', 'too', 'use'])

        return words.filter(word => !stopWords.has(word))
    }

    private async analyzeCodeStructure(content: string): Promise<CodeStructure> {
        // 简化的代码结构分析
        const structure: CodeStructure = {
            functions: [],
            classes: [],
            interfaces: [],
            types: [],
            imports: [],
            exports: []
        }

        // 提取函数定义
        const functionPattern = /(?:function|def)\s+(\w+)/g
        let match
        while ((match = functionPattern.exec(content)) !== null) {
            structure.functions.push({
                name: match[1],
                parameters: [],
                returnType: 'unknown',
                accessibility: 'public'
            })
        }

        // 提取类定义
        const classPattern = /class\s+(\w+)/g
        while ((match = classPattern.exec(content)) !== null) {
            structure.classes.push({
                name: match[1],
                properties: [],
                methods: [],
                inheritance: [],
                accessibility: 'public'
            })
        }

        return structure
    }

    private inferDependencyType(source: string, target: string): DependencyType {
        if (target.startsWith('./') || target.startsWith('../')) {
            return DependencyType.LOCAL
        } else if (target.includes('node_modules') || !target.startsWith('.')) {
            return DependencyType.EXTERNAL
        } else {
            return DependencyType.INTERNAL
        }
    }

    private calculateDependencyStrength(source: string, target: string): number {
        // 简化的依赖强度计算
        if (target.includes('import') || target.includes('require')) {
            return 1.0
        } else if (target.includes('extends') || target.includes('implements')) {
            return 0.8
        } else {
            return 0.5
        }
    }

    private async getEmbedding(text: string): Promise<number[]> {
        // 简化的嵌入生成
        // 实际应用中应该调用真实的嵌入服务
        return Array.from({ length: 1536 }, () => Math.random())
    }

    private cosineSimilarity(a: number[], b: number[]): number {
        if (a.length !== b.length) return 0

        const dotProduct = a.reduce((sum, val, i) => sum + val * b[i], 0)
        const magnitudeA = Math.sqrt(a.reduce((sum, val) => sum + val * val, 0))
        const magnitudeB = Math.sqrt(b.reduce((sum, val) => sum + val * val, 0))

        return magnitudeA && magnitudeB ? dotProduct / (magnitudeA * magnitudeB) : 0
    }
}
```

#### 2. 上下文压缩器

```typescript
// src/core/context/ContextCompressor.ts
export class ContextCompressor {
    private maxTokens: number
    private importanceScorer: ImportanceScorer
    private redundancyDetector: RedundancyDetector

    constructor(maxTokens: number = 128000) {
        this.maxTokens = maxTokens
        this.importanceScorer = new ImportanceScorer()
        this.redundancyDetector = new RedundancyDetector()
    }

    async compressContext(
        context: TaskContext,
        description: string
    ): Promise<CompressedContext> {
        const compressionStart = Date.now()

        // 1. 检测和去除冗余
        const deduplicatedContext = await this.redundancyDetector.deduplicate(context)

        // 2. 评估重要性分数
        const importanceScores = await this.importanceScorer.score(deduplicatedContext, description)

        // 3. 选择最重要的内容
        const selectedContent = await this.selectImportantContent(
            deduplicatedContext,
            importanceScores
        )

        // 4. 智能压缩
        const compressedContent = await this.intelligentlyCompress(
            selectedContent,
            description
        )

        // 5. 验证压缩结果
        const validation = await this.validateCompression(compressedContent, context)

        return {
            originalContext: context,
            compressedContent,
            compressionRatio: this.calculateCompressionRatio(context, compressedContent),
            processingTime: Date.now() - compressionStart,
            estimatedTokens: this.estimateTokens(compressedContent),
            validation
        }
    }

    private async selectImportantContent(
        context: TaskContext,
        scores: ImportanceScores
    ): Promise<TaskContext> {
        const selectedContext: TaskContext = {
            fileContents: {},
            projectStructure: context.projectStructure,
            dependencies: context.dependencies,
            configurations: context.configurations
        }

        if (context.fileContents) {
            // 按重要性分数排序文件
            const sortedFiles = Object.entries(context.fileContents)
                .sort((a, b) => {
                    const scoreA = scores.fileScores.get(a[0]) || 0
                    const scoreB = scores.fileScores.get(b[0]) || 0
                    return scoreB - scoreA
                })

            // 选择最重要的文件
            let totalTokens = 0
            for (const [filePath, content] of sortedFiles) {
                const fileTokens = Math.ceil(content.length / 4)

                if (totalTokens + fileTokens <= this.maxTokens * 0.8) {
                    selectedContext.fileContents![filePath] = content
                    totalTokens += fileTokens
                } else {
                    // 如果文件太大，选择重要部分
                    const importantParts = await this.extractImportantParts(
                        filePath,
                        content,
                        this.maxTokens - totalTokens
                    )
                    if (importantParts) {
                        selectedContext.fileContents![filePath] = importantParts
                        totalTokens += Math.ceil(importantParts.length / 4)
                    }
                    break
                }
            }
        }

        // 选择重要的依赖信息
        if (context.dependencies) {
            selectedContext.dependencies = this.selectImportantDependencies(
                context.dependencies,
                scores.dependencyScores
            )
        }

        return selectedContext
    }

    private async intelligentlyCompress(
        context: TaskContext,
        description: string
    ): Promise<TaskContext> {
        const compressed: TaskContext = {
            fileContents: {},
            projectStructure: context.projectStructure,
            dependencies: context.dependencies,
            configurations: context.configurations
        }

        if (context.fileContents) {
            for (const [filePath, content] of Object.entries(context.fileContents)) {
                compressed.fileContents![filePath] = await this.compressFileContent(
                    filePath,
                    content,
                    description
                )
            }
        }

        return compressed
    }

    private async compressFileContent(
        filePath: string,
        content: string,
        description: string
    ): Promise<string> {
        const language = this.detectLanguage(filePath)

        // 基于语言的智能压缩策略
        switch (language) {
            case 'javascript':
            case 'typescript':
                return this.compressJavaScript(content, description)
            case 'python':
                return this.compressPython(content, description)
            case 'java':
                return this.compressJava(content, description)
            case 'cpp':
            case 'c':
                return this.compressCpp(content, description)
            default:
                return this.compressGeneric(content, description)
        }
    }

    private compressJavaScript(content: string, description: string): string {
        // 保留重要结构
        const patterns = [
            // 导入语句
            /(import\s+.*?from\s+['"][^'"]*['"][\s\S]*?;?\n)/g,
            // 函数定义
            /(function\s+\w+\s*\([^)]*\)\s*{[\s\S]*?})/g,
            /(const\s+\w+\s*=\s*\([^)]*\)\s*=>[\s\S]*?})/g,
            // 类定义
            /(class\s+\w+\s*{[\s\S]*?})/g,
            // 导出语句
            /(export\s+.*?;?\n)/g,
            // 类型定义
            /(interface\s+\w+\s*{[\s\S]*?})/g,
            /(type\s+\w+\s*=.*?;)/g
        ]

        const importantParts: string[] = []

        for (const pattern of patterns) {
            const matches = content.match(pattern)
            if (matches) {
                importantParts.push(...matches)
            }
        }

        // 去重
        const uniqueParts = [...new Set(importantParts)]

        // 按原始顺序排列
        const orderedParts = this.preserveOrder(content, uniqueParts)

        return orderedParts.join('\n\n')
    }

    private compressPython(content: string, description: string): string {
        const patterns = [
            // 导入语句
            /(import\s+.*?\n)/g,
            /(from\s+.*?\s+import\s+.*?\n)/g,
            // 函数定义
            /(def\s+\w+\s*\([^)]*\):\s*[\s\S]*?(?=\n\w|\n\n|$))/g,
            // 类定义
            /(class\s+\w+.*?:[\s\S]*?(?=\n\w|\n\n|$))/g,
            // 装饰器
            /(@\w+[\s\S]*?\n)/g
        ]

        const importantParts: string[] = []

        for (const pattern of patterns) {
            const matches = content.match(pattern)
            if (matches) {
                importantParts.push(...matches)
            }
        }

        const uniqueParts = [...new Set(importantParts)]
        const orderedParts = this.preserveOrder(content, uniqueParts)

        return orderedParts.join('\n\n')
    }

    private compressJava(content: string, description: string): string {
        const patterns = [
            // 包声明
            /(package\s+.*?;\n)/g,
            // 导入语句
            /(import\s+.*?;\n)/g,
            // 类定义
            /(class\s+\w+.*?{[\s\S]*?})/g,
            // 接口定义
            /(interface\s+\w+.*?{[\s\S]*?})/g,
            // 方法定义
            /(public\s+.*?\s+\w+\s*\([^)]*\)\s*{[\s\S]*?})/g
        ]

        const importantParts: string[] = []

        for (const pattern of patterns) {
            const matches = content.match(pattern)
            if (matches) {
                importantParts.push(...matches)
            }
        }

        const uniqueParts = [...new Set(importantParts)]
        const orderedParts = this.preserveOrder(content, uniqueParts)

        return orderedParts.join('\n\n')
    }

    private compressCpp(content: string, description: string): string {
        const patterns = [
            // 包含语句
            /(#include\s*.*?\n)/g,
            // 命名空间
            /(using\s+namespace\s+.*?;\n)/g,
            // 函数定义
            /(\w+\s+\w+\s*\([^)]*\)\s*{[\s\S]*?})/g,
            // 类定义
            /(class\s+\w+.*?{[\s\S]*?};)/g,
            // 结构体定义
            /(struct\s+\w+.*?{[\s\S]*?};)/g
        ]

        const importantParts: string[] = []

        for (const pattern of patterns) {
            const matches = content.match(pattern)
            if (matches) {
                importantParts.push(...matches)
            }
        }

        const uniqueParts = [...new Set(importantParts)]
        const orderedParts = this.preserveOrder(content, uniqueParts)

        return orderedParts.join('\n\n')
    }

    private compressGeneric(content: string, description: string): string {
        // 通用压缩策略：保留重要行
        const lines = content.split('\n')
        const importantLines: string[] = []

        for (const line of lines) {
            const trimmed = line.trim()

            // 保留非空行和非注释行
            if (trimmed && !trimmed.startsWith('//') && !trimmed.startsWith('#')) {
                importantLines.push(line)
            }
        }

        return importantLines.join('\n')
    }

    private preserveOrder(originalContent: string, parts: string[]): string[] {
        const ordered: string[] = []
        let lastIndex = 0

        // 按在原始内容中的出现顺序排列
        for (const part of parts) {
            const index = originalContent.indexOf(part, lastIndex)
            if (index !== -1) {
                ordered.push(part)
                lastIndex = index
            }
        }

        return ordered
    }

    private async extractImportantParts(
        filePath: string,
        content: string,
        maxTokens: number
    ): Promise<string | null> {
        const maxChars = maxTokens * 4
        const compressed = await this.compressFileContent(filePath, content, '')

        if (compressed.length <= maxChars) {
            return compressed
        }

        // 如果压缩后仍然太大，截取前部分
        return compressed.substring(0, maxChars - 3) + '...'
    }

    private selectImportantDependencies(
        dependencies: DependencyInfo[],
        scores: Map<string, number>
    ): DependencyInfo[] {
        // 按重要性分数排序
        return dependencies
            .sort((a, b) => {
                const scoreA = scores.get(a.source + ':' + a.target) || 0
                const scoreB = scores.get(b.source + ':' + b.target) || 0
                return scoreB - scoreA
            })
            .slice(0, 20) // 保留前20个最重要的依赖
    }

    private calculateCompressionRatio(
        original: TaskContext,
        compressed: TaskContext
    ): number {
        const originalSize = this.calculateContextSize(original)
        const compressedSize = this.calculateContextSize(compressed)

        return originalSize > 0 ? compressedSize / originalSize : 1
    }

    private calculateContextSize(context: TaskContext): number {
        let size = 0

        if (context.fileContents) {
            size += Object.values(context.fileContents)
                .reduce((sum, content) => sum + content.length, 0)
        }

        if (context.projectStructure) {
            size += context.projectStructure.length
        }

        if (context.dependencies) {
            size += JSON.stringify(context.dependencies).length
        }

        return size
    }

    private estimateTokens(context: TaskContext): number {
        return Math.ceil(this.calculateContextSize(context) / 4)
    }

    private async validateCompression(
        compressed: TaskContext,
        original: TaskContext
    ): Promise<CompressionValidation> {
        const validation: CompressionValidation = {
            isValid: true,
            warnings: [],
            errors: [],
            qualityScore: 1.0
        }

        // 检查是否保留了关键文件
        if (original.fileContents && Object.keys(original.fileContents).length > 0) {
            if (!compressed.fileContents || Object.keys(compressed.fileContents).length === 0) {
                validation.errors.push("No files retained after compression")
                validation.isValid = false
            } else {
                const retentionRatio = Object.keys(compressed.fileContents).length / Object.keys(original.fileContents).length
                if (retentionRatio < 0.1) {
                    validation.warnings.push(`Low file retention ratio: ${(retentionRatio * 100).toFixed(1)}%`)
                }
            }
        }

        // 检查令牌数量
        const estimatedTokens = this.estimateTokens(compressed)
        if (estimatedTokens > this.maxTokens) {
            validation.errors.push(`Compressed context exceeds token limit: ${estimatedTokens} > ${this.maxTokens}`)
            validation.isValid = false
        }

        // 计算质量分数
        validation.qualityScore = this.calculateQualityScore(compressed, original)

        return validation
    }

    private calculateQualityScore(compressed: TaskContext, original: TaskContext): number {
        let score = 1.0

        // 文件保留率
        if (original.fileContents && compressed.fileContents) {
            const fileRetention = Object.keys(compressed.fileContents).length / Object.keys(original.fileContents).length
            score *= Math.min(1, fileRetention * 2) // 允许一定程度的压缩
        }

        // 内容保留率
        const originalSize = this.calculateContextSize(original)
        const compressedSize = this.calculateContextSize(compressed)
        const contentRetention = compressedSize / originalSize
        score *= Math.max(0.1, contentRetention) // 最低保留10%

        return Math.max(0, Math.min(1, score))
    }

    private detectLanguage(filePath: string): string {
        const ext = path.extname(filePath).toLowerCase()

        const languageMap: Record<string, string> = {
            '.js': 'javascript',
            '.ts': 'typescript',
            '.py': 'python',
            '.java': 'java',
            '.cpp': 'cpp',
            '.c': 'c',
            '.go': 'go',
            '.rs': 'rust',
            '.cs': 'csharp'
        }

        return languageMap[ext] || 'generic'
    }
}
```

## 代码质量保证机制

### 代码验证器

#### 1. 多层次验证系统

```typescript
// src/core/quality/CodeValidator.ts
export class CodeValidator {
    private syntaxValidator: SyntaxValidator
    private semanticValidator: SemanticValidator
    private styleValidator: StyleValidator
    private securityValidator: SecurityValidator
    private performanceValidator: PerformanceValidator

    constructor() {
        this.syntaxValidator = new SyntaxValidator()
        this.semanticValidator = new SemanticValidator()
        this.styleValidator = new StyleValidator()
        this.securityValidator = new SecurityValidator()
        this.performanceValidator = new PerformanceValidator()
    }

    async validateCode(
        code: string,
        language: string,
        context?: ValidationContext
    ): Promise<CodeValidationResult> {
        const validationStart = Date.now()
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 1. 语法验证
        const syntaxResult = await this.syntaxValidator.validate(code, language)
        errors.push(...syntaxResult.errors)
        warnings.push(...syntaxResult.warnings)

        // 2. 语义验证
        const semanticResult = await this.semanticValidator.validate(code, language, context)
        errors.push(...semanticResult.errors)
        warnings.push(...semanticResult.warnings)
        suggestions.push(...semanticResult.suggestions)

        // 3. 代码风格验证
        const styleResult = await this.styleValidator.validate(code, language)
        warnings.push(...styleResult.warnings)
        suggestions.push(...styleResult.suggestions)

        // 4. 安全验证
        const securityResult = await this.securityValidator.validate(code, language)
        errors.push(...securityResult.errors)
        warnings.push(...securityResult.warnings)
        suggestions.push(...securityResult.suggestions)

        // 5. 性能验证
        const performanceResult = await this.performanceValidator.validate(code, language)
        warnings.push(...performanceResult.warnings)
        suggestions.push(...performanceResult.suggestions)

        // 计算综合质量分数
        const qualityScore = this.calculateQualityScore(errors, warnings, suggestions)

        // 生成验证报告
        const report: ValidationReport = {
            language,
            codeLength: code.length,
            validationTime: Date.now() - validationStart,
            totalErrors: errors.length,
            totalWarnings: warnings.length,
            totalSuggestions: suggestions.length,
            qualityScore,
            errors,
            warnings,
            suggestions,
            summary: this.generateSummary(errors, warnings, suggestions, qualityScore)
        }

        return {
            isValid: errors.length === 0,
            report,
            confidence: this.calculateConfidence(report)
        }
    }

    private calculateQualityScore(
        errors: ValidationError[],
        warnings: ValidationWarning[],
        suggestions: ValidationSuggestion[]
    ): number {
        let score = 1.0

        // 错误扣分
        score -= errors.length * 0.3
        score -= warnings.length * 0.1
        score -= suggestions.length * 0.05

        return Math.max(0, Math.min(1, score))
    }

    private calculateConfidence(report: ValidationReport): number {
        let confidence = 1.0

        // 基于验证时间调整置信度
        if (report.validationTime < 100) {
            confidence *= 0.8 // 快速验证置信度较低
        }

        // 基于代码长度调整置信度
        if (report.codeLength > 10000) {
            confidence *= 0.9 // 长代码验证置信度略低
        }

        // 基于错误数量调整置信度
        if (report.totalErrors > 5) {
            confidence *= 0.7
        }

        return Math.max(0, Math.min(1, confidence))
    }

    private generateSummary(
        errors: ValidationError[],
        warnings: ValidationWarning[],
        suggestions: ValidationSuggestion[],
        qualityScore: number
    ): string {
        const parts: string[] = []

        // 总体评价
        if (qualityScore >= 0.9) {
            parts.push("代码质量优秀")
        } else if (qualityScore >= 0.8) {
            parts.push("代码质量良好")
        } else if (qualityScore >= 0.6) {
            parts.push("代码质量一般")
        } else {
            parts.push("代码质量较差")
        }

        // 问题统计
        if (errors.length > 0) {
            parts.push(`发现 ${errors.length} 个错误`)
        }
        if (warnings.length > 0) {
            parts.push(`发现 ${warnings.length} 个警告`)
        }
        if (suggestions.length > 0) {
            parts.push(`提供 ${suggestions.length} 个建议`)
        }

        return parts.join("，")
    }
}

// 语法验证器
class SyntaxValidator {
    async validate(code: string, language: string): Promise<SyntaxValidationResult> {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        switch (language) {
            case 'javascript':
            case 'typescript':
                return this.validateJavaScript(code, language)
            case 'python':
                return this.validatePython(code)
            case 'java':
                return this.validateJava(code)
            case 'cpp':
            case 'c':
                return this.validateCpp(code)
            default:
                return this.validateGeneric(code)
        }
    }

    private validateJavaScript(code: string, language: string): SyntaxValidationResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        try {
            // 使用 TypeScript 编译器进行语法检查
            const ts = require('typescript')
            const result = ts.transpileModule(code, {
                compilerOptions: {
                    target: ts.ScriptTarget.ES2015,
                    module: ts.ModuleKind.CommonJS,
                    noEmit: true
                }
            })

            if (result.diagnostics) {
                for (const diagnostic of result.diagnostics) {
                    if (diagnostic.category === ts.DiagnosticCategory.Error) {
                        errors.push({
                            type: 'syntax',
                            message: diagnostic.messageText,
                            line: diagnostic.file?.getLineAndCharacterOfPosition(diagnostic.start!).line,
                            severity: 'error'
                        })
                    } else {
                        warnings.push({
                            type: 'syntax',
                            message: diagnostic.messageText,
                            line: diagnostic.file?.getLineAndCharacterOfPosition(diagnostic.start!).line,
                            severity: 'warning'
                        })
                    }
                }
            }
        } catch (error) {
            errors.push({
                type: 'syntax',
                message: `语法错误: ${error.message}`,
                line: 0,
                severity: 'error'
            })
        }

        return { errors, warnings }
    }

    private validatePython(code: string): SyntaxValidationResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        try {
            // 使用 Python 解析器进行语法检查
            // 这里简化实现，实际应该使用 Python 解析器
            const ast = this.parsePythonAST(code)
            if (!ast) {
                errors.push({
                    type: 'syntax',
                    message: 'Python 语法错误',
                    line: 0,
                    severity: 'error'
                })
            }
        } catch (error) {
            errors.push({
                type: 'syntax',
                message: `Python 语法错误: ${error.message}`,
                line: 0,
                severity: 'error'
            })
        }

        return { errors, warnings }
    }

    private parsePythonAST(code: string): any {
        // 简化的 Python AST 解析
        try {
            // 这里应该调用实际的 Python 解析器
            return { valid: true }
        } catch {
            return null
        }
    }

    private validateJava(code: string): SyntaxValidationResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 基础的 Java 语法检查
        const patterns = [
            { pattern: /class\s+\w+/, error: '缺少类定义' },
            { pattern: /public\s+static\s+void\s+main/, error: '缺少 main 方法' },
            { pattern: /;/g, error: '缺少分号' }
        ]

        for (const { pattern, error } of patterns) {
            if (!pattern.test(code)) {
                errors.push({
                    type: 'syntax',
                    message: error,
                    line: 0,
                    severity: 'error'
                })
            }
        }

        return { errors, warnings }
    }

    private validateCpp(code: string): SyntaxValidationResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 基础的 C++ 语法检查
        const patterns = [
            { pattern: /#include\s*<[^>]+>/, error: '缺少头文件包含' },
            { pattern: /int\s+main/, error: '缺少 main 函数' },
            { pattern: /{.*}/s, error: '括号不匹配' }
        ]

        for (const { pattern, error } of patterns) {
            if (!pattern.test(code)) {
                errors.push({
                    type: 'syntax',
                    message: error,
                    line: 0,
                    severity: 'error'
                })
            }
        }

        return { errors, warnings }
    }

    private validateGeneric(code: string): SyntaxValidationResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 通用语法检查
        const lines = code.split('\n')
        let braceCount = 0
        let parenCount = 0

        for (let i = 0; i < lines.length; i++) {
            const line = lines[i]

            // 检查括号匹配
            braceCount += (line.match(/{/g) || []).length
            braceCount -= (line.match(/}/g) || []).length
            parenCount += (line.match(/\(/g) || []).length
            parenCount -= (line.match(/\)/g) || []).length
        }

        if (braceCount !== 0) {
            errors.push({
                type: 'syntax',
                message: '大括号不匹配',
                line: 0,
                severity: 'error'
            })
        }

        if (parenCount !== 0) {
            errors.push({
                type: 'syntax',
                message: '括号不匹配',
                line: 0,
                severity: 'error'
            })
        }

        return { errors, warnings }
    }
}

// 语义验证器
class SemanticValidator {
    async validate(
        code: string,
        language: string,
        context?: ValidationContext
    ): Promise<SemanticValidationResult> {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 变量使用检查
        const variableIssues = this.checkVariableUsage(code, language)
        errors.push(...variableIssues.errors)
        warnings.push(...variableIssues.warnings)

        // 函数调用检查
        const functionIssues = this.checkFunctionCalls(code, language)
        errors.push(...functionIssues.errors)
        warnings.push(...functionIssues.warnings)

        // 类型检查
        const typeIssues = this.checkTypes(code, language)
        errors.push(...typeIssues.errors)
        warnings.push(...typeIssues.warnings)

        // 逻辑检查
        const logicIssues = this.checkLogic(code, language)
        warnings.push(...logicIssues.warnings)
        suggestions.push(...logicIssues.suggestions)

        return { errors, warnings, suggestions }
    }

    private checkVariableUsage(code: string, language: string): VariableCheckResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 提取变量定义和使用
        const definedVars = this.extractDefinedVariables(code, language)
        const usedVars = this.extractUsedVariables(code, language)

        // 检查未定义变量
        for (const usedVar of usedVars) {
            if (!definedVars.includes(usedVar)) {
                warnings.push({
                    type: 'semantic',
                    message: `变量 '${usedVar}' 可能未定义`,
                    line: this.findVariableLine(code, usedVar),
                    severity: 'warning'
                })
            }
        }

        // 检查未使用变量
        for (const definedVar of definedVars) {
            if (!usedVars.includes(definedVar)) {
                warnings.push({
                    type: 'semantic',
                    message: `变量 '${definedVar}' 已定义但未使用`,
                    line: this.findVariableLine(code, definedVar),
                    severity: 'warning'
                })
            }
        }

        return { errors, warnings }
    }

    private extractDefinedVariables(code: string, language: string): string[] {
        const variables: string[] = []

        const patterns = [
            // JavaScript/TypeScript
            /(?:const|let|var)\s+(\w+)/g,
            // Python
            /(\w+)\s*=/g,
            // Java
            /(?:int|String|double|boolean)\s+(\w+)/g,
            // C++
            /(?:int|float|double|char)\s+(\w+)/g
        ]

        for (const pattern of patterns) {
            let match
            while ((match = pattern.exec(code)) !== null) {
                variables.push(match[1])
            }
        }

        return [...new Set(variables)]
    }

    private extractUsedVariables(code: string, language: string): string[] {
        const variables: string[] = []

        // 匹配变量使用（排除关键字）
        const pattern = /\b([a-zA-Z_]\w*)\b/g
        const keywords = new Set([
            'if', 'else', 'for', 'while', 'function', 'return', 'class', 'import', 'export',
            'const', 'let', 'var', 'int', 'string', 'boolean', 'true', 'false', 'null'
        ])

        let match
        while ((match = pattern.exec(code)) !== null) {
            if (!keywords.has(match[1])) {
                variables.push(match[1])
            }
        }

        return [...new Set(variables)]
    }

    private findVariableLine(code: string, variable: string): number {
        const lines = code.split('\n')
        for (let i = 0; i < lines.length; i++) {
            if (lines[i].includes(variable)) {
                return i + 1
            }
        }
        return 0
    }

    private checkFunctionCalls(code: string, language: string): FunctionCallCheckResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 提取函数调用
        const functionCalls = this.extractFunctionCalls(code, language)

        // 检查函数定义
        const functionDefinitions = this.extractFunctionDefinitions(code, language)

        // 检查未定义函数
        for (const call of functionCalls) {
            if (!functionDefinitions.includes(call.name)) {
                warnings.push({
                    type: 'semantic',
                    message: `函数 '${call.name}' 可能未定义`,
                    line: call.line,
                    severity: 'warning'
                })
            }
        }

        return { errors, warnings }
    }

    private extractFunctionCalls(code: string, language: string): FunctionCall[] {
        const calls: FunctionCall[] = []

        const pattern = /(\w+)\s*\([^)]*\)/g
        let match
        const lines = code.split('\n')

        for (let i = 0; i < lines.length; i++) {
            const line = lines[i]
            while ((match = pattern.exec(line)) !== null) {
                calls.push({
                    name: match[1],
                    line: i + 1
                })
            }
        }

        return calls
    }

    private extractFunctionDefinitions(code: string, language: string): string[] {
        const functions: string[] = []

        const patterns = [
            /function\s+(\w+)/g,
            /def\s+(\w+)/g,
            /\w+\s+(\w+)\s*\([^)]*\)\s*{/g,
            /(\w+)\s*\([^)]*\)\s*=>/g
        ]

        for (const pattern of patterns) {
            let match
            while ((match = pattern.exec(code)) !== null) {
                functions.push(match[1])
            }
        }

        return [...new Set(functions)]
    }

    private checkTypes(code: string, language: string): TypeCheckResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 类型检查逻辑
        // 这里简化实现，实际应该使用类型检查器

        return { errors, warnings }
    }

    private checkLogic(code: string, language: string): LogicCheckResult {
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查潜在的逻辑问题
        const logicIssues = this.detectLogicIssues(code)
        warnings.push(...logicIssues.warnings)
        suggestions.push(...logicIssues.suggestions)

        return { warnings, suggestions }
    }

    private detectLogicIssues(code: string): LogicIssueResult {
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查无限循环
        if (this.detectPotentialInfiniteLoop(code)) {
            warnings.push({
                type: 'logic',
                message: '可能存在无限循环',
                line: 0,
                severity: 'warning'
            })
        }

        // 检查空指针解引用
        if (this.detectPotentialNullDereference(code)) {
            warnings.push({
                type: 'logic',
                message: '可能存在空指针解引用',
                line: 0,
                severity: 'warning'
            })
        }

        // 检查未处理的异常
        if (this.detectUnhandledExceptions(code)) {
            suggestions.push({
                type: 'logic',
                message: '建议添加异常处理',
                line: 0,
                severity: 'suggestion'
            })
        }

        return { warnings, suggestions }
    }

    private detectPotentialInfiniteLoop(code: string): boolean {
        // 简化的无限循环检测
        const loopPatterns = [
            /while\s*\(\s*true\s*\)/i,
            /for\s*\(\s*;;\s*\)/i
        ]

        return loopPatterns.some(pattern => pattern.test(code))
    }

    private detectPotentialNullDereference(code: string): boolean {
        // 简化的空指针检测
        const patterns = [
            /\.value/g,
            /\[\d+\]/g
        ]

        return patterns.some(pattern => pattern.test(code))
    }

    private detectUnhandledExceptions(code: string): boolean {
        // 简化的异常检测
        const riskyOperations = [
            /JSON\.parse/g,
            /fetch\(/g,
            /require\(/g
        ]

        const tryCatch = /try\s*{[\s\S]*?}\s*catch\s*\([^)]*\)\s*{[\s\S]*?}/g

        return riskyOperations.some(op => op.test(code)) && !tryCatch.test(code)
    }
}

// 代码风格验证器
class StyleValidator {
    async validate(code: string, language: string): Promise<StyleValidationResult> {
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查命名规范
        const namingIssues = this.checkNamingConventions(code, language)
        warnings.push(...namingIssues.warnings)

        // 检查代码格式
        const formattingIssues = this.checkCodeFormatting(code, language)
        warnings.push(...formattingIssues.warnings)

        // 检查注释质量
        const commentIssues = this.checkComments(code, language)
        suggestions.push(...commentIssues.suggestions)

        return { warnings, suggestions }
    }

    private checkNamingConventions(code: string, language: string): NamingCheckResult {
        const warnings: ValidationWarning[] = []

        // 检查变量命名
        const variablePattern = /(?:const|let|var|int|String|double|boolean)\s+(\w+)/g
        let match

        while ((match = variablePattern.exec(code)) !== null) {
            const variableName = match[1]

            // 检查驼峰命名
            if (!/^[a-z][a-zA-Z0-9]*$/.test(variableName)) {
                warnings.push({
                    type: 'style',
                    message: `变量 '${variableName}' 应使用驼峰命名法`,
                    line: this.getLineNumber(code, match.index),
                    severity: 'warning'
                })
            }
        }

        // 检查常量命名
        const constantPattern = /const\s+([A-Z_]+)\s*=/g
        while ((match = constantPattern.exec(code)) !== null) {
            const constantName = match[1]

            if (!/^[A-Z_]+$/.test(constantName)) {
                warnings.push({
                    type: 'style',
                    message: `常量 '${constantName}' 应使用大写下划线命名法`,
                    line: this.getLineNumber(code, match.index),
                    severity: 'warning'
                })
            }
        }

        return { warnings }
    }

    private checkCodeFormatting(code: string, language: string): FormattingCheckResult {
        const warnings: ValidationWarning[] = []

        // 检查缩进
        const indentIssues = this.checkIndentation(code)
        warnings.push(...indentIssues)

        // 检查行长度
        const lineLengthIssues = this.checkLineLength(code)
        warnings.push(...lineLengthIssues)

        // 检查空行
        const spacingIssues = this.checkSpacing(code)
        warnings.push(...spacingIssues)

        return { warnings }
    }

    private checkIndentation(code: string): ValidationWarning[] {
        const warnings: ValidationWarning[] = []
        const lines = code.split('\n')

        for (let i = 0; i < lines.length; i++) {
            const line = lines[i]
            if (line.trim()) {
                const indent = line.length - line.trimStart().length

                // 检查缩进一致性
                if (indent % 2 !== 0) {
                    warnings.push({
                        type: 'style',
                        message: '缩进应为2的倍数',
                        line: i + 1,
                        severity: 'warning'
                    })
                }
            }
        }

        return warnings
    }

    private checkLineLength(code: string): ValidationWarning[] {
        const warnings: ValidationWarning[] = []
        const lines = code.split('\n')

        for (let i = 0; i < lines.length; i++) {
            const line = lines[i]
            if (line.length > 120) {
                warnings.push({
                    type: 'style',
                    message: '行长度超过120个字符',
                    line: i + 1,
                    severity: 'warning'
                })
            }
        }

        return warnings
    }

    private checkSpacing(code: string): ValidationWarning[] = {
        const warnings: ValidationWarning[] = []
        const lines = code.split('\n')

        for (let i = 0; i < lines.length - 1; i++) {
            const currentLine = lines[i]
            const nextLine = lines[i + 1]

            // 检查连续空行
            if (!currentLine.trim() && !nextLine.trim()) {
                warnings.push({
                    type: 'style',
                    message: '避免连续空行',
                    line: i + 1,
                    severity: 'warning'
                })
            }
        }

        return warnings
    }

    private checkComments(code: string, language: string): CommentCheckResult {
        const suggestions: ValidationSuggestion[] = []

        // 检查注释覆盖率
        const commentCoverage = this.calculateCommentCoverage(code)
        if (commentCoverage < 0.1) {
            suggestions.push({
                type: 'style',
                message: '建议添加更多注释以提高代码可读性',
                line: 0,
                severity: 'suggestion'
            })
        }

        return { suggestions }
    }

    private calculateCommentCoverage(code: string): number {
        const lines = code.split('\n')
        const commentLines = lines.filter(line => line.trim().startsWith('//') || line.trim().startsWith('#'))

        return commentLines.length / lines.length
    }

    private getLineNumber(code: string, index: number): number {
        const lines = code.substring(0, index).split('\n')
        return lines.length
    }
}

// 安全验证器
class SecurityValidator {
    async validate(code: string, language: string): Promise<SecurityValidationResult> {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查SQL注入
        const sqlInjectionIssues = this.checkSQLInjection(code)
        errors.push(...sqlInjectionIssues.errors)
        warnings.push(...sqlInjectionIssues.warnings)

        // 检查XSS漏洞
        const xssIssues = this.checkXSS(code)
        errors.push(...xssIssues.errors)
        warnings.push(...xssIssues.warnings)

        // 检查敏感信息
        const sensitiveInfoIssues = this.checkSensitiveInformation(code)
        errors.push(...sensitiveInfoIssues.errors)
        suggestions.push(...sensitiveInfoIssues.suggestions)

        return { errors, warnings, suggestions }
    }

    private checkSQLInjection(code: string): SecurityCheckResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 检查字符串拼接的SQL查询
        const sqlPatterns = [
            /(?:SELECT|INSERT|UPDATE|DELETE)\s+.*?\+\s*\w+/gi,
            /(?:SELECT|INSERT|UPDATE|DELETE)\s+.*?\$\{.*?\}/gi
        ]

        for (const pattern of sqlPatterns) {
            if (pattern.test(code)) {
                errors.push({
                    type: 'security',
                    message: '检测到潜在的SQL注入漏洞',
                    line: 0,
                    severity: 'error'
                })
            }
        }

        return { errors, warnings }
    }

    private checkXSS(code: string): SecurityCheckResult {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 检查innerHTML使用
        if (code.includes('innerHTML')) {
            warnings.push({
                type: 'security',
                message: '使用innerHTML可能导致XSS攻击',
                line: 0,
                severity: 'warning'
            })
        }

        // 检查eval使用
        if (code.includes('eval(')) {
            errors.push({
                type: 'security',
                message: 'eval函数存在安全风险',
                line: 0,
                severity: 'error'
            })
        }

        return { errors, warnings }
    }

    private checkSensitiveInformation(code: string): SecurityCheckResult {
        const errors: ValidationError[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查硬编码的敏感信息
        const sensitivePatterns = [
            /password\s*=\s*['"][^'"]+['"]/gi,
            /api[_-]?key\s*=\s*['"][^'"]+['"]/gi,
            /secret\s*=\s*['"][^'"]+['"]/gi
        ]

        for (const pattern of sensitivePatterns) {
            if (pattern.test(code)) {
                errors.push({
                    type: 'security',
                    message: '检测到硬编码的敏感信息',
                    line: 0,
                    severity: 'error'
                })
            }
        }

        // 检查调试信息
        if (code.includes('console.log') || code.includes('console.debug')) {
            suggestions.push({
                type: 'security',
                message: '生产环境中应移除调试信息',
                line: 0,
                severity: 'suggestion'
            })
        }

        return { errors, suggestions }
    }
}

// 性能验证器
class PerformanceValidator {
    async validate(code: string, language: string): Promise<PerformanceValidationResult> {
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查循环性能
        const loopIssues = this.checkLoops(code)
        warnings.push(...loopIssues.warnings)
        suggestions.push(...loopIssues.suggestions)

        // 检查内存使用
        const memoryIssues = this.checkMemoryUsage(code)
        warnings.push(...memoryIssues.warnings)

        // 检查异步操作
        const asyncIssues = this.checkAsyncOperations(code)
        suggestions.push(...asyncIssues.suggestions)

        return { warnings, suggestions }
    }

    private checkLoops(code: string): PerformanceCheckResult {
        const warnings: ValidationWarning[] = []
        const suggestions: ValidationSuggestion[] = []

        // 检查嵌套循环
        const nestedLoops = (code.match(/for\s*\(/g) || []).length
        if (nestedLoops > 2) {
            warnings.push({
                type: 'performance',
                message: '检测到多层嵌套循环，可能影响性能',
                line: 0,
                severity: 'warning'
            })
        }

        // 检查循环内的函数调用
        if (/for\s*\([^)]+\)\s*{[\s\S]*?\w+\([^)]+\)/.test(code)) {
            suggestions.push({
                type: 'performance',
                message: '考虑将循环内的函数调用移出循环',
                line: 0,
                severity: 'suggestion'
            })
        }

        return { warnings, suggestions }
    }

    private checkMemoryUsage(code: string): PerformanceCheckResult {
        const warnings: ValidationWarning[] = []

        // 检查大数组创建
        if (/new\s+Array\(\d{4,}\)/.test(code)) {
            warnings.push({
                type: 'performance',
                message: '检测到大数组创建，注意内存使用',
                line: 0,
                severity: 'warning'
            })
        }

        return { warnings, suggestions: [] }
    }

    private checkAsyncOperations(code: string): PerformanceCheckResult {
        const suggestions: ValidationSuggestion[] = []

        // 检查同步操作
        if (code.includes('fs.readFileSync') || code.includes('child_process.execSync')) {
            suggestions.push({
                type: 'performance',
                message: '建议使用异步操作以提高性能',
                line: 0,
                severity: 'suggestion'
            })
        }

        return { warnings: [], suggestions }
    }
}
```

## 总结

KiloCode 的代码生成核心算法展现了以下技术特点：

### 1. **分层提示词架构**
- 智能模板选择系统
- 动态提示词生成
- 上下文感知的变量替换
- 多语言适配支持

### 2. **智能上下文管理**
- 基于相关性的文件选择
- 语义相似性计算
- 上下文压缩和优化
- 令牌数量控制

### 3. **多层质量保证**
- 语法验证
- 语义分析
- 代码风格检查
- 安全漏洞检测
- 性能问题识别

### 4. **自适应优化**
- 基于反馈的模板改进
- 动态上下文调整
- 智能错误恢复
- 性能监控和优化

### 5. **工程化实践**
- 模块化设计
- 可扩展架构
- 配置驱动
- 监控和分析

这种架构设计使得 KiloCode 能够生成高质量、可靠的代码，同时保持良好的用户体验和系统性能。对于构建 AI 编程助手，这种代码生成算法具有重要的参考价值。