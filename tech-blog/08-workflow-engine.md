# 任务自动化工作流引擎

> 深入解析 KiloCode 的工作流定义、任务调度算法和状态同步机制

## 工作流引擎概览

KiloCode 的任务自动化工作流引擎是一个强大的系统，能够将复杂的编程任务分解为可管理的步骤，并通过智能调度和状态管理实现高效的任务执行。这个引擎支持多种工作流模式，包括线性、并行、条件分支和循环等。

### 系统设计目标

1. **灵活性**：支持多种工作流模式
2. **可靠性**：确保任务执行的一致性
3. **可观测性**：完整的执行过程追踪
4. **可扩展性**：支持自定义任务类型

## 工作流定义系统

### 工作流规范

#### 1. 工作流定义语言

```typescript
// src/core/workflow/WorkflowDefinition.ts
export interface WorkflowDefinition {
    id: string
    name: string
    description: string
    version: string
    author: string
    tags: string[]

    // 工作流结构
    steps: WorkflowStep[]
    parameters: WorkflowParameter[]
    outputs: WorkflowOutput[]

    // 执行配置
    executionConfig: ExecutionConfig
    retryConfig: RetryConfig
    timeoutConfig: TimeoutConfig

    // 元数据
    metadata: WorkflowMetadata
    createdAt: Date
    updatedAt: Date
}

export interface WorkflowStep {
    id: string
    name: string
    type: StepType
    description: string

    // 依赖关系
    dependsOn: string[]

    // 输入输出
    inputs: StepInput[]
    outputs: StepOutput[]

    // 执行配置
    executor: string
    parameters: Record<string, any>

    // 条件逻辑
    condition?: StepCondition
    loopConfig?: LoopConfig

    // 错误处理
    errorHandling: ErrorHandlingConfig

    // 元数据
    metadata: StepMetadata
}

export enum StepType {
    TASK = 'task',
    PARALLEL = 'parallel',
    CONDITIONAL = 'conditional',
    LOOP = 'loop',
    SUB_WORKFLOW = 'sub_workflow',
    WAIT = 'wait',
    MANUAL = 'manual'
}

export interface StepCondition {
    type: 'expression' | 'rule'
    expression?: string
    rules?: ConditionRule[]
}

export interface ConditionRule {
    field: string
    operator: 'equals' | 'not_equals' | 'greater_than' | 'less_than' | 'contains'
    value: any
}

export interface LoopConfig {
    type: 'for' | 'while' | 'foreach'
    iterable?: string
    condition?: string
    maxIterations?: number
}

export interface ErrorHandlingConfig {
    retry: boolean
    maxRetries: number
    backoffStrategy: 'linear' | 'exponential'
    onTimeout: 'continue' | 'fail' | 'retry'
    onError: 'continue' | 'fail' | 'retry'
}

export interface ExecutionConfig {
    maxConcurrentSteps: number
    priority: 'low' | 'medium' | 'high'
    queueTimeout: number
    stepTimeout: number
}

export interface RetryConfig {
    maxRetries: number
    retryDelay: number
    backoffMultiplier: number
    maxRetryDelay: number
}

export interface TimeoutConfig {
    stepTimeout: number
    workflowTimeout: number
    idleTimeout: number
}

// 工作流验证器
export class WorkflowValidator {
    private stepRegistry: StepRegistry
    private expressionEvaluator: ExpressionEvaluator

    constructor(stepRegistry: StepRegistry, expressionEvaluator: ExpressionEvaluator) {
        this.stepRegistry = stepRegistry
        this.expressionEvaluator = expressionEvaluator
    }

    async validateDefinition(definition: WorkflowDefinition): Promise<ValidationResult> {
        const errors: ValidationError[] = []
        const warnings: ValidationWarning[] = []

        // 1. 基础信息验证
        this.validateBasicInfo(definition, errors, warnings)

        // 2. 步骤验证
        await this.validateSteps(definition, errors, warnings)

        // 3. 依赖关系验证
        this.validateDependencies(definition, errors, warnings)

        // 4. 输入输出验证
        this.validateInputsOutputs(definition, errors, warnings)

        // 5. 条件表达式验证
        await this.validateConditions(definition, errors, warnings)

        // 6. 循环配置验证
        this.validateLoops(definition, errors, warnings)

        // 7. 错误处理验证
        this.validateErrorHandling(definition, errors, warnings)

        return {
            isValid: errors.length === 0,
            errors,
            warnings
        }
    }

    private validateBasicInfo(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): void {
        if (!definition.id || definition.id.trim() === '') {
            errors.push({
                type: 'validation',
                message: 'Workflow ID is required',
                field: 'id'
            })
        }

        if (!definition.name || definition.name.trim() === '') {
            errors.push({
                type: 'validation',
                message: 'Workflow name is required',
                field: 'name'
            })
        }

        if (!definition.steps || definition.steps.length === 0) {
            errors.push({
                type: 'validation',
                message: 'Workflow must have at least one step',
                field: 'steps'
            })
        }

        // 检查ID唯一性
        const stepIds = new Set<string>()
        for (const step of definition.steps) {
            if (stepIds.has(step.id)) {
                errors.push({
                    type: 'validation',
                    message: `Duplicate step ID: ${step.id}`,
                    field: 'steps'
                })
            }
            stepIds.add(step.id)
        }
    }

    private async validateSteps(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): Promise<void> {
        for (const step of definition.steps) {
            // 验证步骤类型
            if (!Object.values(StepType).includes(step.type)) {
                errors.push({
                    type: 'validation',
                    message: `Invalid step type: ${step.type}`,
                    field: `steps.${step.id}.type`
                })
            }

            // 验证执行器
            if (!step.executor) {
                errors.push({
                    type: 'validation',
                    message: 'Step executor is required',
                    field: `steps.${step.id}.executor`
                })
            } else if (!this.stepRegistry.hasExecutor(step.executor)) {
                warnings.push({
                    type: 'validation',
                    message: `Unknown executor: ${step.executor}`,
                    field: `steps.${step.id}.executor`
                })
            }

            // 验证参数
            if (step.parameters) {
                const paramValidation = await this.stepRegistry.validateParameters(step.executor, step.parameters)
                if (!paramValidation.valid) {
                    errors.push({
                        type: 'validation',
                        message: `Invalid parameters for step ${step.id}: ${paramValidation.message}`,
                        field: `steps.${step.id}.parameters`
                    })
                }
            }

            // 特定类型验证
            switch (step.type) {
                case StepType.CONDITIONAL:
                    this.validateConditionalStep(step, errors, warnings)
                    break
                case StepType.LOOP:
                    this.validateLoopStep(step, errors, warnings)
                    break
                case StepType.PARALLEL:
                    this.validateParallelStep(step, errors, warnings)
                    break
                case StepType.SUB_WORKFLOW:
                    this.validateSubWorkflowStep(step, errors, warnings)
                    break
            }
        }
    }

    private validateDependencies(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): void {
        const stepIds = new Set(definition.steps.map(s => s.id))

        for (const step of definition.steps) {
            // 验证依赖步骤存在
            for (const depId of step.dependsOn) {
                if (!stepIds.has(depId)) {
                    errors.push({
                        type: 'validation',
                        message: `Dependency step not found: ${depId}`,
                        field: `steps.${step.id}.dependsOn`
                    })
                }
            }

            // 检查循环依赖
            if (this.hasCircularDependency(step, definition.steps)) {
                errors.push({
                    type: 'validation',
                    message: 'Circular dependency detected',
                    field: `steps.${step.id}.dependsOn`
                })
            }
        }

        // 检查孤立步骤
        this.validateIsolatedSteps(definition.steps, warnings)
    }

    private validateInputsOutputs(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): void {
        // 验证工作流参数
        const paramNames = new Set(definition.parameters?.map(p => p.name) || [])

        for (const step of definition.steps) {
            // 验证步骤输入
            for (const input of step.inputs || []) {
                if (input.source === 'parameter' && !paramNames.has(input.name)) {
                    errors.push({
                        type: 'validation',
                        message: `Parameter not found: ${input.name}`,
                        field: `steps.${step.id}.inputs`
                    })
                }

                if (input.source === 'step') {
                    const sourceStep = definition.steps.find(s => s.id === input.fromStep)
                    if (!sourceStep) {
                        errors.push({
                            type: 'validation',
                            message: `Source step not found: ${input.fromStep}`,
                            field: `steps.${step.id}.inputs`
                        })
                    } else {
                        const sourceOutput = sourceStep.outputs?.find(o => o.name === input.name)
                        if (!sourceOutput) {
                            errors.push({
                                type: 'validation',
                                message: `Output not found in source step: ${input.name}`,
                                field: `steps.${step.id}.inputs`
                            })
                        }
                    }
                }
            }
        }
    }

    private async validateConditions(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): Promise<void> {
        for (const step of definition.steps) {
            if (step.condition) {
                try {
                    if (step.condition.type === 'expression' && step.condition.expression) {
                        await this.expressionEvaluator.validate(step.condition.expression)
                    }
                } catch (error) {
                    errors.push({
                        type: 'validation',
                        message: `Invalid condition expression: ${error.message}`,
                        field: `steps.${step.id}.condition`
                    })
                }
            }
        }
    }

    private validateLoops(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): void {
        for (const step of definition.steps) {
            if (step.loopConfig) {
                if (step.loopConfig.maxIterations && step.loopConfig.maxIterations <= 0) {
                    errors.push({
                        type: 'validation',
                        message: 'Max iterations must be greater than 0',
                        field: `steps.${step.id}.loopConfig.maxIterations`
                    })
                }

                if (step.loopConfig.type === 'while' && !step.loopConfig.condition) {
                    errors.push({
                        type: 'validation',
                        message: 'While loop requires condition',
                        field: `steps.${step.id}.loopConfig`
                    })
                }

                if (step.loopConfig.type === 'foreach' && !step.loopConfig.iterable) {
                    errors.push({
                        type: 'validation',
                        message: 'Foreach loop requires iterable',
                        field: `steps.${step.id}.loopConfig`
                    })
                }
            }
        }
    }

    private validateErrorHandling(definition: WorkflowDefinition, errors: ValidationError[], warnings: ValidationWarning[]): void {
        for (const step of definition.steps) {
            const errorHandling = step.errorHandling

            if (errorHandling.retry && errorHandling.maxRetries < 0) {
                errors.push({
                    type: 'validation',
                    message: 'Max retries must be non-negative',
                    field: `steps.${step.id}.errorHandling.maxRetries`
                })
            }

            if (errorHandling.retry && errorHandling.retryDelay < 0) {
                errors.push({
                    type: 'validation',
                    message: 'Retry delay must be non-negative',
                    field: `steps.${step.id}.errorHandling.retryDelay`
                })
            }
        }
    }

    private validateConditionalStep(step: WorkflowStep, errors: ValidationError[], warnings: ValidationWarning[]): void {
        if (!step.condition) {
            errors.push({
                type: 'validation',
                message: 'Conditional step requires condition',
                field: `steps.${step.id}.condition`
            })
        }
    }

    private validateLoopStep(step: WorkflowStep, errors: ValidationError[], warnings: ValidationWarning[]): void {
        if (!step.loopConfig) {
            errors.push({
                type: 'validation',
                message: 'Loop step requires loopConfig',
                field: `steps.${step.id}.loopConfig`
            })
        }
    }

    private validateParallelStep(step: WorkflowStep, errors: ValidationError[], warnings: ValidationWarning[]): void {
        // 并行步骤的特殊验证
        if (step.dependsOn.length === 0) {
            warnings.push({
                type: 'validation',
                message: 'Parallel step with no dependencies will execute immediately',
                field: `steps.${step.id}.dependsOn`
            })
        }
    }

    private validateSubWorkflowStep(step: WorkflowStep, errors: ValidationError[], warnings: ValidationWarning[]): void {
        // 子工作流步骤的特殊验证
        if (!step.parameters?.workflowId) {
            errors.push({
                type: 'validation',
                message: 'Sub workflow step requires workflowId parameter',
                field: `steps.${step.id}.parameters.workflowId`
            })
        }
    }

    private hasCircularDependency(step: WorkflowStep, allSteps: WorkflowStep[]): boolean {
        const visited = new Set<string>()
        const recursionStack = new Set<string>()

        const hasCycle = (stepId: string): boolean => {
            if (recursionStack.has(stepId)) {
                return true
            }

            if (visited.has(stepId)) {
                return false
            }

            visited.add(stepId)
            recursionStack.add(stepId)

            const step = allSteps.find(s => s.id === stepId)
            if (step) {
                for (const depId of step.dependsOn) {
                    if (hasCycle(depId)) {
                        return true
                    }
                }
            }

            recursionStack.delete(stepId)
            return false
        }

        return hasCycle(step.id)
    }

    private validateIsolatedSteps(steps: WorkflowStep[], warnings: ValidationWarning[]): void {
        const reachableSteps = new Set<string>()

        // 从没有依赖的步骤开始遍历
        const startSteps = steps.filter(s => s.dependsOn.length === 0)
        const queue = [...startSteps.map(s => s.id)]

        while (queue.length > 0) {
            const stepId = queue.shift()!
            if (reachableSteps.has(stepId)) continue

            reachableSteps.add(stepId)

            // 找到依赖当前步骤的所有步骤
            const dependentSteps = steps.filter(s => s.dependsOn.includes(stepId))
            queue.push(...dependentSteps.map(s => s.id))
        }

        // 检查是否有不可达的步骤
        for (const step of steps) {
            if (!reachableSteps.has(step.id)) {
                warnings.push({
                    type: 'validation',
                    message: `Step is unreachable: ${step.id}`,
                    field: 'steps'
                })
            }
        }
    }
}

// 工作流构建器
export class WorkflowBuilder {
    private definition: Partial<WorkflowDefinition> = {
        steps: [],
        parameters: [],
        outputs: [],
        tags: []
    }

    constructor(id: string, name: string) {
        this.definition.id = id
        this.definition.name = name
        this.definition.version = '1.0.0'
        this.definition.createdAt = new Date()
    }

    description(description: string): WorkflowBuilder {
        this.definition.description = description
        return this
    }

    author(author: string): WorkflowBuilder {
        this.definition.author = author
        return this
    }

    version(version: string): WorkflowBuilder {
        this.definition.version = version
        return this
    }

    tags(tags: string[]): WorkflowBuilder {
        this.definition.tags = tags
        return this
    }

    parameter(name: string, type: string, required: boolean = true, defaultValue?: any): WorkflowBuilder {
        if (!this.definition.parameters) {
            this.definition.parameters = []
        }

        this.definition.parameters.push({
            name,
            type,
            required,
            defaultValue
        })

        return this
    }

    step(step: WorkflowStep): WorkflowBuilder {
        if (!this.definition.steps) {
            this.definition.steps = []
        }

        this.definition.steps.push(step)
        return this
    }

    taskStep(id: string, name: string, executor: string): WorkflowStepBuilder {
        const step: WorkflowStep = {
            id,
            name,
            type: StepType.TASK,
            description: '',
            dependsOn: [],
            inputs: [],
            outputs: [],
            executor,
            parameters: {},
            errorHandling: {
                retry: false,
                maxRetries: 0,
                backoffStrategy: 'linear',
                onTimeout: 'fail',
                onError: 'fail'
            },
            metadata: {}
        }

        return new WorkflowStepBuilder(this, step)
    }

    parallelStep(id: string, name: string, steps: WorkflowStep[]): WorkflowStepBuilder {
        const step: WorkflowStep = {
            id,
            name,
            type: StepType.PARALLEL,
            description: '',
            dependsOn: [],
            inputs: [],
            outputs: [],
            executor: 'parallel',
            parameters: { steps },
            errorHandling: {
                retry: false,
                maxRetries: 0,
                backoffStrategy: 'linear',
                onTimeout: 'fail',
                onError: 'fail'
            },
            metadata: {}
        }

        return new WorkflowStepBuilder(this, step)
    }

    conditionalStep(id: string, name: string, condition: StepCondition): WorkflowStepBuilder {
        const step: WorkflowStep = {
            id,
            name,
            type: StepType.CONDITIONAL,
            description: '',
            dependsOn: [],
            inputs: [],
            outputs: [],
            executor: 'conditional',
            parameters: {},
            condition,
            errorHandling: {
                retry: false,
                maxRetries: 0,
                backoffStrategy: 'linear',
                onTimeout: 'fail',
                onError: 'fail'
            },
            metadata: {}
        }

        return new WorkflowStepBuilder(this, step)
    }

    loopStep(id: string, name: string, loopConfig: LoopConfig): WorkflowStepBuilder {
        const step: WorkflowStep = {
            id,
            name,
            type: StepType.LOOP,
            description: '',
            dependsOn: [],
            inputs: [],
            outputs: [],
            executor: 'loop',
            parameters: {},
            loopConfig,
            errorHandling: {
                retry: false,
                maxRetries: 0,
                backoffStrategy: 'linear',
                onTimeout: 'fail',
                onError: 'fail'
            },
            metadata: {}
        }

        return new WorkflowStepBuilder(this, step)
    }

    executionConfig(config: Partial<ExecutionConfig>): WorkflowBuilder {
        this.definition.executionConfig = { ...this.definition.executionConfig, ...config }
        return this
    }

    retryConfig(config: Partial<RetryConfig>): WorkflowBuilder {
        this.definition.retryConfig = { ...this.definition.retryConfig, ...config }
        return this
    }

    timeoutConfig(config: Partial<TimeoutConfig>): WorkflowBuilder {
        this.definition.timeoutConfig = { ...this.definition.timeoutConfig, ...config }
        return this
    }

    build(): WorkflowDefinition {
        if (!this.definition.id) {
            throw new Error('Workflow ID is required')
        }

        if (!this.definition.name) {
            throw new Error('Workflow name is required')
        }

        if (!this.definition.steps || this.definition.steps.length === 0) {
            throw new Error('Workflow must have at least one step')
        }

        this.definition.updatedAt = new Date()

        return this.definition as WorkflowDefinition
    }
}

// 工作流步骤构建器
export class WorkflowStepBuilder {
    private builder: WorkflowBuilder
    private step: WorkflowStep

    constructor(builder: WorkflowBuilder, step: WorkflowStep) {
        this.builder = builder
        this.step = step
    }

    description(description: string): WorkflowStepBuilder {
        this.step.description = description
        return this
    }

    dependsOn(stepIds: string[]): WorkflowStepBuilder {
        this.step.dependsOn = stepIds
        return this
    }

    input(name: string, source: 'parameter' | 'step', fromStep?: string, fromOutput?: string): WorkflowStepBuilder {
        if (!this.step.inputs) {
            this.step.inputs = []
        }

        this.step.inputs.push({
            name,
            source,
            fromStep,
            fromOutput
        })

        return this
    }

    output(name: string, type: string): WorkflowStepBuilder {
        if (!this.step.outputs) {
            this.step.outputs = []
        }

        this.step.outputs.push({
            name,
            type
        })

        return this
    }

    parameter(name: string, value: any): WorkflowStepBuilder {
        if (!this.step.parameters) {
            this.step.parameters = {}
        }

        this.step.parameters[name] = value
        return this
    }

    parameters(params: Record<string, any>): WorkflowStepBuilder {
        this.step.parameters = { ...this.step.parameters, ...params }
        return this
    }

    condition(condition: StepCondition): WorkflowStepBuilder {
        this.step.condition = condition
        return this
    }

    loopConfig(config: LoopConfig): WorkflowStepBuilder {
        this.step.loopConfig = config
        return this
    }

    retry(maxRetries: number, delay: number = 1000, backoff: 'linear' | 'exponential' = 'linear'): WorkflowStepBuilder {
        this.step.errorHandling.retry = true
        this.step.errorHandling.maxRetries = maxRetries
        this.step.errorHandling.retryDelay = delay
        this.step.errorHandling.backoffStrategy = backoff
        return this
    }

    timeout(stepTimeout: number): WorkflowStepBuilder {
        this.step.errorHandling.onTimeout = 'fail'
        // 这里应该设置步骤超时配置
        return this
    }

    onError(action: 'continue' | 'fail' | 'retry'): WorkflowStepBuilder {
        this.step.errorHandling.onError = action
        return this
    }

    metadata(metadata: StepMetadata): WorkflowStepBuilder {
        this.step.metadata = { ...this.step.metadata, ...metadata }
        return this
    }

    add(): WorkflowBuilder {
        this.builder.step(this.step)
        return this.builder
    }

    buildStep(): WorkflowStep {
        return this.step
    }
}
```

### 工作流执行引擎

#### 1. 执行引擎核心

```typescript
// src/core/workflow/WorkflowEngine.ts
export class WorkflowEngine {
    private workflowStore: WorkflowStore
    private executorRegistry: ExecutorRegistry
    private scheduler: TaskScheduler
    private stateManager: StateManager
    private eventBus: EventBus
    private metrics: WorkflowMetrics

    constructor(
        workflowStore: WorkflowStore,
        executorRegistry: ExecutorRegistry,
        scheduler: TaskScheduler,
        stateManager: StateManager,
        eventBus: EventBus,
        metrics: WorkflowMetrics
    ) {
        this.workflowStore = workflowStore
        this.executorRegistry = executorRegistry
        this.scheduler = scheduler
        this.stateManager = stateManager
        this.eventBus = eventBus
        this.metrics = metrics
    }

    async executeWorkflow(
        workflowId: string,
        parameters: Record<string, any> = {},
        options: ExecutionOptions = {}
    ): Promise<ExecutionResult> {
        const executionStart = Date.now()

        try {
            // 1. 获取工作流定义
            const workflow = await this.workflowStore.getWorkflow(workflowId)
            if (!workflow) {
                throw new Error(`Workflow not found: ${workflowId}`)
            }

            // 2. 创建执行实例
            const execution = await this.createExecution(workflow, parameters, options)

            // 3. 验证输入参数
            this.validateInputs(workflow, parameters)

            // 4. 开始执行
            await this.startExecution(execution)

            // 5. 等待执行完成或监听进度
            const result = await this.monitorExecution(execution)

            // 6. 记录执行指标
            await this.metrics.recordExecution({
                workflowId,
                executionId: execution.id,
                status: result.status,
                duration: Date.now() - executionStart,
                stepsCompleted: result.stepsCompleted,
                stepsFailed: result.stepsFailed,
                timestamp: new Date()
            })

            return result

        } catch (error) {
            await this.metrics.recordExecution({
                workflowId,
                executionId: 'unknown',
                status: 'failed',
                duration: Date.now() - executionStart,
                error: error.message,
                timestamp: new Date()
            })

            throw error
        }
    }

    private async createExecution(
        workflow: WorkflowDefinition,
        parameters: Record<string, any>,
        options: ExecutionOptions
    ): Promise<WorkflowExecution> {
        const execution: WorkflowExecution = {
            id: this.generateExecutionId(),
            workflowId: workflow.id,
            workflowVersion: workflow.version,
            status: ExecutionStatus.CREATED,
            parameters,
            context: this.createInitialContext(workflow, parameters),
            steps: this.createStepExecutions(workflow),
            createdAt: new Date(),
            startedAt: null,
            completedAt: null,
            options
        }

        // 保存执行实例
        await this.stateManager.saveExecution(execution)

        // 发布创建事件
        await this.eventBus.publish('workflow.created', {
            executionId: execution.id,
            workflowId: workflow.id,
            parameters
        })

        return execution
    }

    private createInitialContext(
        workflow: WorkflowDefinition,
        parameters: Record<string, any>
    ): ExecutionContext {
        const context: ExecutionContext = {
            variables: {},
            stepResults: {},
            errors: [],
            metadata: {
                executionId: this.generateExecutionId(),
                workflowId: workflow.id,
                startTime: new Date(),
                retryCount: 0
            }
        }

        // 初始化参数变量
        for (const param of workflow.parameters || []) {
            if (parameters[param.name] !== undefined) {
                context.variables[param.name] = parameters[param.name]
            } else if (param.defaultValue !== undefined) {
                context.variables[param.name] = param.defaultValue
            } else if (param.required) {
                throw new Error(`Required parameter missing: ${param.name}`)
            }
        }

        return context
    }

    private createStepExecutions(workflow: WorkflowDefinition): StepExecution[] {
        return workflow.steps.map(step => ({
            id: this.generateStepExecutionId(),
            stepId: step.id,
            status: StepStatus.PENDING,
            startTime: null,
            endTime: null,
            retryCount: 0,
            output: null,
            error: null,
            parameters: {}
        }))
    }

    private async startExecution(execution: WorkflowExecution): Promise<void> {
        execution.status = ExecutionStatus.RUNNING
        execution.startedAt = new Date()

        await this.stateManager.updateExecution(execution)

        // 发布开始事件
        await this.eventBus.publish('workflow.started', {
            executionId: execution.id,
            workflowId: execution.workflowId
        })

        // 开始执行步骤
        await this.executeReadySteps(execution)
    }

    private async executeReadySteps(execution: WorkflowExecution): Promise<void> {
        // 获取可以执行的步骤
        const readySteps = this.getReadySteps(execution)

        if (readySteps.length === 0) {
            // 检查是否所有步骤都已完成
            if (this.isExecutionComplete(execution)) {
                await this.completeExecution(execution)
            }
            return
        }

        // 并行执行就绪步骤
        const executionPromises = readySteps.map(step => this.executeStep(execution, step))

        try {
            await Promise.all(executionPromises)

            // 继续执行下一个步骤
            await this.executeReadySteps(execution)
        } catch (error) {
            // 处理执行错误
            await this.handleExecutionError(execution, error)
        }
    }

    private getReadySteps(execution: WorkflowExecution): StepExecution[] {
        const workflow = execution.workflow!
        const readySteps: StepExecution[] = []

        for (const stepExecution of execution.steps) {
            if (stepExecution.status !== StepStatus.PENDING) {
                continue
            }

            const stepDefinition = workflow.steps.find(s => s.id === stepExecution.stepId)
            if (!stepDefinition) {
                continue
            }

            // 检查依赖是否已完成
            const dependenciesMet = stepDefinition.dependsOn.every(depId => {
                const depStep = execution.steps.find(s => s.stepId === depId)
                return depStep && depStep.status === StepStatus.COMPLETED
            })

            if (dependenciesMet) {
                readySteps.push(stepExecution)
            }
        }

        return readySteps
    }

    private async executeStep(execution: WorkflowExecution, stepExecution: StepExecution): Promise<void> {
        const stepDefinition = execution.workflow!.steps.find(s => s.id === stepExecution.stepId)!
        stepExecution.status = StepStatus.RUNNING
        stepExecution.startTime = new Date()

        await this.stateManager.updateStepExecution(execution.id, stepExecution)

        try {
            // 准备步骤参数
            const stepParameters = await this.prepareStepParameters(execution, stepDefinition)

            // 检查条件（如果是条件步骤）
            if (stepDefinition.type === StepType.CONDITIONAL && stepDefinition.condition) {
                const conditionResult = await this.evaluateCondition(
                    stepDefinition.condition,
                    execution.context
                )

                if (!conditionResult) {
                    stepExecution.status = StepStatus.SKIPPED
                    stepExecution.endTime = new Date()
                    await this.stateManager.updateStepExecution(execution.id, stepExecution)
                    return
                }
            }

            // 执行步骤
            const result = await this.executeStepWithRetry(
                execution,
                stepDefinition,
                stepParameters
            )

            // 处理循环（如果是循环步骤）
            if (stepDefinition.type === StepType.LOOP && stepDefinition.loopConfig) {
                const loopResult = await this.executeLoopStep(
                    execution,
                    stepDefinition,
                    stepParameters
                )
                stepExecution.output = loopResult
            } else {
                stepExecution.output = result
            }

            stepExecution.status = StepStatus.COMPLETED
            stepExecution.endTime = new Date()

            // 更新上下文
            execution.context.stepResults[stepExecution.stepId] = result

            // 发布步骤完成事件
            await this.eventBus.publish('step.completed', {
                executionId: execution.id,
                stepId: stepExecution.stepId,
                result
            })

        } catch (error) {
            stepExecution.status = StepStatus.FAILED
            stepExecution.error = error.message
            stepExecution.endTime = new Date()

            // 添加错误到上下文
            execution.context.errors.push({
                stepId: stepExecution.stepId,
                error: error.message,
                timestamp: new Date()
            })

            // 发布步骤失败事件
            await this.eventBus.publish('step.failed', {
                executionId: execution.id,
                stepId: stepExecution.stepId,
                error: error.message
            })

            // 检查是否应该停止执行
            if (stepDefinition.errorHandling.onError === 'fail') {
                throw new Error(`Step ${stepExecution.stepId} failed: ${error.message}`)
            }
        }

        await this.stateManager.updateStepExecution(execution.id, stepExecution)
    }

    private async executeStepWithRetry(
        execution: WorkflowExecution,
        stepDefinition: WorkflowStep,
        parameters: Record<string, any>
    ): Promise<any> {
        let lastError: Error | null = null
        const maxRetries = stepDefinition.errorHandling.maxRetries || 0

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return await this.executeStepOnce(execution, stepDefinition, parameters)
            } catch (error) {
                lastError = error

                if (attempt < maxRetries) {
                    const delay = this.calculateRetryDelay(
                        stepDefinition.errorHandling.retryDelay || 1000,
                        stepDefinition.errorHandling.backoffStrategy,
                        attempt
                    )

                    await this.sleep(delay)

                    // 发布重试事件
                    await this.eventBus.publish('step.retried', {
                        executionId: execution.id,
                        stepId: stepDefinition.id,
                        attempt: attempt + 1,
                        error: error.message
                    })
                }
            }
        }

        throw lastError || new Error('Step execution failed')
    }

    private async executeStepOnce(
        execution: WorkflowExecution,
        stepDefinition: WorkflowStep,
        parameters: Record<string, any>
    ): Promise<any> {
        const executor = this.executorRegistry.getExecutor(stepDefinition.executor)
        if (!executor) {
            throw new Error(`Executor not found: ${stepDefinition.executor}`)
        }

        const executionContext: ExecutionContext = {
            ...execution.context,
            stepId: stepDefinition.id,
            stepParameters: parameters,
            workflowDefinition: execution.workflow
        }

        return executor.execute(executionContext)
    }

    private async executeLoopStep(
        execution: WorkflowExecution,
        stepDefinition: WorkflowStep,
        parameters: Record<string, any>
    ): Promise<any> {
        if (!stepDefinition.loopConfig) {
            throw new Error('Loop configuration is required for loop steps')
        }

        const loopConfig = stepDefinition.loopConfig
        const results: any[] = []

        switch (loopConfig.type) {
            case 'for':
                for (let i = 0; i < (loopConfig.maxIterations || 10); i++) {
                    const result = await this.executeStepOnce(execution, stepDefinition, {
                        ...parameters,
                        iteration: i
                    })
                    results.push(result)
                }
                break

            case 'foreach':
                const iterable = this.resolveIterable(loopConfig.iterable!, execution.context)
                for (const item of iterable) {
                    const result = await this.executeStepOnce(execution, stepDefinition, {
                        ...parameters,
                        item
                    })
                    results.push(result)
                }
                break

            case 'while':
                let iteration = 0
                while (await this.evaluateCondition(
                    { type: 'expression', expression: loopConfig.condition! },
                    execution.context
                ) && iteration < (loopConfig.maxIterations || 100)) {
                    const result = await this.executeStepOnce(execution, stepDefinition, {
                        ...parameters,
                        iteration
                    })
                    results.push(result)
                    iteration++
                }
                break
        }

        return results
    }

    private async evaluateCondition(
        condition: StepCondition,
        context: ExecutionContext
    ): Promise<boolean> {
        if (condition.type === 'expression' && condition.expression) {
            const evaluator = new ExpressionEvaluator()
            return evaluator.evaluate(condition.expression, context)
        } else if (condition.type === 'rule' && condition.rules) {
            return this.evaluateRules(condition.rules, context)
        }

        return true
    }

    private evaluateRules(rules: ConditionRule[], context: ExecutionContext): boolean {
        for (const rule of rules) {
            const value = this.resolveFieldValue(rule.field, context)

            switch (rule.operator) {
                case 'equals':
                    if (value !== rule.value) return false
                    break
                case 'not_equals':
                    if (value === rule.value) return false
                    break
                case 'greater_than':
                    if (!(value > rule.value)) return false
                    break
                case 'less_than':
                    if (!(value < rule.value)) return false
                    break
                case 'contains':
                    if (!Array.isArray(value) || !value.includes(rule.value)) return false
                    break
            }
        }

        return true
    }

    private resolveFieldValue(field: string, context: ExecutionContext): any {
        // 解析字段路径，如 stepResults.stepId.output
        const parts = field.split('.')
        let value: any = context

        for (const part of parts) {
            if (value && typeof value === 'object') {
                value = value[part]
            } else {
                return undefined
            }
        }

        return value
    }

    private resolveIterable(iterablePath: string, context: ExecutionContext): any[] {
        const iterable = this.resolveFieldValue(iterablePath, context)
        return Array.isArray(iterable) ? iterable : []
    }

    private async prepareStepParameters(
        execution: WorkflowExecution,
        stepDefinition: WorkflowStep
    ): Promise<Record<string, any>> {
        const parameters: Record<string, any> = {}

        // 合并默认参数
        Object.assign(parameters, stepDefinition.parameters)

        // 处理输入映射
        for (const input of stepDefinition.inputs || []) {
            let value: any

            if (input.source === 'parameter') {
                value = execution.context.variables[input.name]
            } else if (input.source === 'step') {
                const stepResult = execution.context.stepResults[input.fromStep!]
                if (stepResult && typeof stepResult === 'object') {
                    value = input.fromOutput ? stepResult[input.fromOutput] : stepResult
                }
            }

            if (value !== undefined) {
                parameters[input.name] = value
            }
        }

        return parameters
    }

    private calculateRetryDelay(
        baseDelay: number,
        strategy: 'linear' | 'exponential',
        attempt: number
    ): number {
        if (strategy === 'exponential') {
            return baseDelay * Math.pow(2, attempt)
        } else {
            return baseDelay * (attempt + 1)
        }
    }

    private async handleExecutionError(execution: WorkflowExecution, error: Error): Promise<void> {
        execution.status = ExecutionStatus.FAILED
        execution.completedAt = new Date()

        await this.stateManager.updateExecution(execution)

        // 发布执行失败事件
        await this.eventBus.publish('workflow.failed', {
            executionId: execution.id,
            workflowId: execution.workflowId,
            error: error.message
        })
    }

    private async completeExecution(execution: WorkflowExecution): Promise<void> {
        execution.status = ExecutionStatus.COMPLETED
        execution.completedAt = new Date()

        await this.stateManager.updateExecution(execution)

        // 发布执行完成事件
        await this.eventBus.publish('workflow.completed', {
            executionId: execution.id,
            workflowId: execution.workflowId,
            result: this.generateExecutionResult(execution)
        })
    }

    private isExecutionComplete(execution: WorkflowExecution): boolean {
        // 检查是否所有步骤都已完成或跳过
        return execution.steps.every(step =>
            step.status === StepStatus.COMPLETED ||
            step.status === StepStatus.SKIPPED
        )
    }

    private generateExecutionResult(execution: WorkflowExecution): ExecutionResult {
        const stepsCompleted = execution.steps.filter(s => s.status === StepStatus.COMPLETED).length
        const stepsFailed = execution.steps.filter(s => s.status === StepStatus.FAILED).length
        const stepsSkipped = execution.steps.filter(s => s.status === StepStatus.SKIPPED).length

        return {
            executionId: execution.id,
            workflowId: execution.workflowId,
            status: execution.status,
            stepsCompleted,
            stepsFailed,
            stepsSkipped,
            duration: execution.completedAt ? execution.completedAt.getTime() - execution.startedAt!.getTime() : 0,
            output: this.generateWorkflowOutput(execution),
            errors: execution.context.errors,
            startedAt: execution.startedAt!,
            completedAt: execution.completedAt
        }
    }

    private generateWorkflowOutput(execution: WorkflowExecution): Record<string, any> {
        const output: Record<string, any> = {}

        // 收集工作流定义的输出
        if (execution.workflow!.outputs) {
            for (const workflowOutput of execution.workflow!.outputs) {
                if (workflowOutput.source === 'step') {
                    const stepResult = execution.context.stepResults[workflowOutput.fromStep!]
                    if (stepResult) {
                        output[workflowOutput.name] = workflowOutput.fromOutput
                            ? stepResult[workflowOutput.fromOutput]
                            : stepResult
                    }
                }
            }
        }

        return output
    }

    private async monitorExecution(execution: WorkflowExecution): Promise<ExecutionResult> {
        return new Promise((resolve, reject) => {
            const timeout = setTimeout(() => {
                listener.dispose()
                reject(new Error('Execution timeout'))
            }, execution.options.timeout || 300000) // 5分钟默认超时

            const listener = this.eventBus.subscribe('workflow.*', async (event: WorkflowEvent) => {
                if (event.data.executionId === execution.id) {
                    switch (event.type) {
                        case 'workflow.completed':
                            clearTimeout(timeout)
                            listener.dispose()
                            resolve(event.data.result)
                            break
                        case 'workflow.failed':
                            clearTimeout(timeout)
                            listener.dispose()
                            reject(new Error(event.data.error))
                            break
                    }
                }
            })
        })
    }

    private generateExecutionId(): string {
        return `exec_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
    }

    private generateStepExecutionId(): string {
        return `step_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
    }

    private sleep(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms))
    }

    private validateInputs(workflow: WorkflowDefinition, parameters: Record<string, any>): void {
        for (const param of workflow.parameters || []) {
            if (param.required && parameters[param.name] === undefined) {
                throw new Error(`Required parameter missing: ${param.name}`)
            }
        }
    }

    // 公共方法
    async getExecution(executionId: string): Promise<WorkflowExecution | null> {
        return this.stateManager.getExecution(executionId)
    }

    async getExecutionHistory(workflowId: string): Promise<WorkflowExecution[]> {
        return this.stateManager.getExecutionHistory(workflowId)
    }

    async cancelExecution(executionId: string): Promise<void> {
        const execution = await this.stateManager.getExecution(executionId)
        if (!execution) {
            throw new Error(`Execution not found: ${executionId}`)
        }

        if (execution.status === ExecutionStatus.RUNNING) {
            execution.status = ExecutionStatus.CANCELLED
            execution.completedAt = new Date()
            await this.stateManager.updateExecution(execution)

            await this.eventBus.publish('workflow.cancelled', {
                executionId,
                workflowId: execution.workflowId
            })
        }
    }

    async retryExecution(executionId: string): Promise<ExecutionResult> {
        const execution = await this.stateManager.getExecution(executionId)
        if (!execution) {
            throw new Error(`Execution not found: ${executionId}`)
        }

        if (execution.status !== ExecutionStatus.FAILED) {
            throw new Error('Only failed executions can be retried')
        }

        // 重置步骤状态
        execution.steps.forEach(step => {
            if (step.status === StepStatus.FAILED) {
                step.status = StepStatus.PENDING
                step.startTime = null
                step.endTime = null
                step.retryCount++
                step.error = null
            }
        })

        execution.status = ExecutionStatus.RUNNING
        execution.startedAt = new Date()
        execution.completedAt = null
        execution.context.retryCount++

        await this.stateManager.updateExecution(execution)

        // 重新开始执行
        return this.monitorExecution(execution)
    }
}

// 状态管理器
class StateManager {
    private executionStore: ExecutionStore
    private stepExecutionStore: StepExecutionStore

    constructor(executionStore: ExecutionStore, stepExecutionStore: StepExecutionStore) {
        this.executionStore = executionStore
        this.stepExecutionStore = stepExecutionStore
    }

    async saveExecution(execution: WorkflowExecution): Promise<void> {
        await this.executionStore.save(execution)
    }

    async updateExecution(execution: WorkflowExecution): Promise<void> {
        await this.executionStore.update(execution)
    }

    async getExecution(executionId: string): Promise<WorkflowExecution | null> {
        return this.executionStore.get(executionId)
    }

    async getExecutionHistory(workflowId: string): Promise<WorkflowExecution[]> {
        return this.executionStore.getByWorkflowId(workflowId)
    }

    async updateStepExecution(executionId: string, stepExecution: StepExecution): Promise<void> {
        await this.stepExecutionStore.update(executionId, stepExecution)
    }

    async getStepExecution(executionId: string, stepId: string): Promise<StepExecution | null> {
        return this.stepExecutionStore.get(executionId, stepId)
    }
}

// 表达式求值器
class ExpressionEvaluator {
    async evaluate(expression: string, context: ExecutionContext): Promise<boolean> {
        // 安全的表达式求值
        // 这里应该使用沙箱环境执行表达式
        try {
            // 创建安全的求值上下文
            const evalContext = this.createSafeContext(context)

            // 使用 Function 构造函数创建安全的求值环境
            const evalFunction = new Function(
                'context',
                `with(context) {
                    return ${expression};
                }`
            )

            return evalFunction(evalContext)
        } catch (error) {
            throw new Error(`Expression evaluation failed: ${error.message}`)
        }
    }

    async validate(expression: string): Promise<void> {
        // 验证表达式语法
        try {
            new Function(`return ${expression}`)
        } catch (error) {
            throw new Error(`Invalid expression: ${error.message}`)
        }

        // 检查表达式安全性
        const dangerousPatterns = [
            /import\s+/,
            /require\s*\(/,
            /eval\s*\(/,
            /Function\s*\(/,
            /process\./,
            /global\./,
            /window\./,
            /document\./
        ]

        for (const pattern of dangerousPatterns) {
            if (pattern.test(expression)) {
                throw new Error('Expression contains dangerous patterns')
            }
        }
    }

    private createSafeContext(context: ExecutionContext): any {
        return {
            // 只暴露安全的上下文属性
            variables: context.variables,
            stepResults: context.stepResults,
            metadata: context.metadata,

            // 提供安全的工具函数
            equals: (a: any, b: any) => a === b,
            notEquals: (a: any, b: any) => a !== b,
            greaterThan: (a: number, b: number) => a > b,
            lessThan: (a: number, b: number) => a < b,
            contains: (arr: any[], item: any) => Array.isArray(arr) && arr.includes(item),
            length: (arr: any[]) => Array.isArray(arr) ? arr.length : 0,
            now: () => new Date(),
            parseInt: (str: string) => parseInt(str, 10),
            parseFloat: (str: string) => parseFloat(str)
        }
    }
}
```

## 任务调度算法

### 智能调度器

#### 1. 任务调度策略

```typescript
// src/core/workflow/scheduler/TaskScheduler.ts
export class TaskScheduler {
    private queueManager: QueueManager
    private executorRegistry: ExecutorRegistry
    private loadBalancer: LoadBalancer
    private priorityManager: PriorityManager
    private resourceManager: ResourceManager
    private metrics: SchedulerMetrics

    constructor(
        queueManager: QueueManager,
        executorRegistry: ExecutorRegistry,
        loadBalancer: LoadBalancer,
        priorityManager: PriorityManager,
        resourceManager: ResourceManager,
        metrics: SchedulerMetrics
    ) {
        this.queueManager = queueManager
        this.executorRegistry = executorRegistry
        this.loadBalancer = loadBalancer
        this.priorityManager = priorityManager
        this.resourceManager = resourceManager
        this.metrics = metrics
    }

    async scheduleTask(task: Task): Promise<ScheduleResult> {
        const scheduleStart = Date.now()

        try {
            // 1. 任务验证
            const validation = await this.validateTask(task)
            if (!validation.valid) {
                return {
                    success: false,
                    error: validation.error,
                    scheduleTime: Date.now() - scheduleStart
                }
            }

            // 2. 资源评估
            const resourceAssessment = await this.assessResources(task)
            if (!resourceAssessment.sufficient) {
                return {
                    success: false,
                    error: `Insufficient resources: ${resourceAssessment.reason}`,
                    scheduleTime: Date.now() - scheduleStart
                }
            }

            // 3. 优先级计算
            const priority = await this.calculatePriority(task)

            // 4. 队列选择
            const queue = await this.selectQueue(task, priority)

            // 5. 调度策略应用
            const scheduleStrategy = this.selectScheduleStrategy(task)

            // 6. 执行器分配
            const executor = await this.allocateExecutor(task, scheduleStrategy)

            // 7. 任务排队
            const queuedTask = await this.queueTask(task, queue, executor, priority)

            // 8. 记录调度指标
            await this.metrics.recordSchedule({
                taskId: task.id,
                priority,
                queue: queue.name,
                executor: executor.id,
                scheduleTime: Date.now() - scheduleStart,
                strategy: scheduleStrategy.type,
                timestamp: new Date()
            })

            return {
                success: true,
                taskId: task.id,
                queueId: queue.id,
                executorId: executor.id,
                estimatedStartTime: this.calculateEstimatedStartTime(queuedTask),
                scheduleTime: Date.now() - scheduleStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                scheduleTime: Date.now() - scheduleStart
            }
        }
    }

    async scheduleBatch(tasks: Task[]): Promise<BatchScheduleResult> {
        const batchStart = Date.now()

        try {
            const results: ScheduleResult[] = []
            const successfulTasks: string[] = []
            const failedTasks: string[] = []

            // 批量验证
            const validationResults = await Promise.all(
                tasks.map(task => this.validateTask(task))
            )

            const validTasks = tasks.filter((_, index) => validationResults[index].valid)

            if (validTasks.length === 0) {
                return {
                    success: false,
                    error: 'No valid tasks to schedule',
                    scheduleTime: Date.now() - batchStart
                }
            }

            // 批量资源评估
            const resourceAssessment = await this.assessBatchResources(validTasks)
            if (!resourceAssessment.sufficient) {
                return {
                    success: false,
                    error: `Insufficient resources for batch: ${resourceAssessment.reason}`,
                    scheduleTime: Date.now() - batchStart
                }
            }

            // 批量优先级计算
            const priorities = await Promise.all(
                validTasks.map(task => this.calculatePriority(task))
            )

            // 分组调度
            const taskGroups = this.groupTasksByStrategy(validTasks, priorities)

            // 并行调度各组任务
            const groupPromises = taskGroups.map(group =>
                this.scheduleTaskGroup(group)
            )

            const groupResults = await Promise.all(groupPromises)

            // 合并结果
            for (const groupResult of groupResults) {
                results.push(...groupResult.results)
                successfulTasks.push(...groupResult.successfulTasks)
                failedTasks.push(...groupResult.failedTasks)
            }

            return {
                success: failedTasks.length === 0,
                results,
                successfulTasks,
                failedTasks,
                totalTasks: tasks.length,
                scheduleTime: Date.now() - batchStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                scheduleTime: Date.now() - batchStart
            }
        }
    }

    private async validateTask(task: Task): Promise<TaskValidation> {
        // 验证任务必需字段
        if (!task.id) {
            return {
                valid: false,
                error: 'Task ID is required'
            }
        }

        if (!task.type) {
            return {
                valid: false,
                error: 'Task type is required'
            }
        }

        if (!task.executor) {
            return {
                valid: false,
                error: 'Task executor is required'
            }
        }

        // 验证执行器存在
        const executor = this.executorRegistry.getExecutor(task.executor)
        if (!executor) {
            return {
                valid: false,
                error: `Executor not found: ${task.executor}`
            }
        }

        // 验证任务参数
        try {
            await executor.validateParameters(task.parameters || {})
        } catch (error) {
            return {
                valid: false,
                error: `Invalid parameters: ${error.message}`
            }
        }

        return { valid: true }
    }

    private async assessResources(task: Task): Promise<ResourceAssessment> {
        const requiredResources = this.calculateRequiredResources(task)
        const availableResources = await this.resourceManager.getAvailableResources()

        // 检查CPU
        if (requiredResources.cpu > availableResources.cpu) {
            return {
                sufficient: false,
                reason: `Insufficient CPU: required ${requiredResources.cpu}, available ${availableResources.cpu}`
            }
        }

        // 检查内存
        if (requiredResources.memory > availableResources.memory) {
            return {
                sufficient: false,
                reason: `Insufficient memory: required ${requiredResources.memory}, available ${availableResources.memory}`
            }
        }

        // 检查磁盘空间
        if (requiredResources.disk > availableResources.disk) {
            return {
                sufficient: false,
                reason: `Insufficient disk space: required ${requiredResources.disk}, available ${availableResources.disk}`
            }
        }

        // 检查网络带宽
        if (requiredResources.network > availableResources.network) {
            return {
                sufficient: false,
                reason: `Insufficient network bandwidth: required ${requiredResources.network}, available ${availableResources.network}`
            }
        }

        return { sufficient: true }
    }

    private async assessBatchResources(tasks: Task[]): Promise<BatchResourceAssessment> {
        const totalRequired = tasks.reduce((acc, task) => {
            const required = this.calculateRequiredResources(task)
            return {
                cpu: acc.cpu + required.cpu,
                memory: acc.memory + required.memory,
                disk: acc.disk + required.disk,
                network: acc.network + required.network
            }
        }, { cpu: 0, memory: 0, disk: 0, network: 0 })

        const available = await this.resourceManager.getAvailableResources()

        const sufficient =
            totalRequired.cpu <= available.cpu &&
            totalRequired.memory <= available.memory &&
            totalRequired.disk <= available.disk &&
            totalRequired.network <= available.network

        return {
            sufficient,
            totalRequired,
            available,
            reason: sufficient ? '' : 'Insufficient resources for batch execution'
        }
    }

    private calculateRequiredResources(task: Task): ResourceRequirements {
        // 基于任务类型和参数估算资源需求
        const baseRequirements = this.getBaseResourceRequirements(task.type)

        // 考虑任务参数的影响
        const multiplier = this.calculateParameterMultiplier(task.parameters || {})

        return {
            cpu: baseRequirements.cpu * multiplier,
            memory: baseRequirements.memory * multiplier,
            disk: baseRequirements.disk * multiplier,
            network: baseRequirements.network * multiplier
        }
    }

    private getBaseResourceRequirements(taskType: string): ResourceRequirements {
        // 不同类型任务的基础资源需求
        const requirements: Record<string, ResourceRequirements> = {
            'code_generation': { cpu: 2, memory: 1024, disk: 100, network: 10 },
            'file_processing': { cpu: 1, memory: 512, disk: 500, network: 5 },
            'network_request': { cpu: 0.5, memory: 256, disk: 10, network: 100 },
            'data_analysis': { cpu: 4, memory: 2048, disk: 1000, network: 20 },
            'compilation': { cpu: 3, memory: 1536, disk: 200, network: 15 }
        }

        return requirements[taskType] || { cpu: 1, memory: 512, disk: 100, network: 10 }
    }

    private calculateParameterMultiplier(parameters: Record<string, any>): number {
        // 基于任务参数调整资源需求
        let multiplier = 1.0

        // 文件大小影响
        if (parameters.fileSize) {
            multiplier *= Math.max(1, parameters.fileSize / (1024 * 1024)) // 基于1MB的倍数
        }

        // 数据集大小影响
        if (parameters.datasetSize) {
            multiplier *= Math.max(1, parameters.datasetSize / 1000) // 基于1000条记录的倍数
        }

        // 复杂度影响
        if (parameters.complexity) {
            multiplier *= parameters.complexity
        }

        return Math.min(multiplier, 10) // 最大限制为10倍
    }

    private async calculatePriority(task: Task): Promise<number> {
        // 计算任务优先级（0-100，越高优先级越高）
        let priority = 50 // 基础优先级

        // 用户定义优先级
        if (task.priority) {
            switch (task.priority) {
                case 'high':
                    priority += 30
                    break
                case 'medium':
                    priority += 10
                    break
                case 'low':
                    priority -= 20
                    break
            }
        }

        // 紧急度调整
        if (task.urgency) {
            priority += task.urgency * 20
        }

        // 等待时间调整
        if (task.createdAt) {
            const waitTime = Date.now() - task.createdAt.getTime()
            priority += Math.min(waitTime / (1000 * 60), 20) // 每分钟增加1点，最多20点
        }

        // 用户重要性调整
        if (task.userImportance) {
            priority += task.userImportance * 10
        }

        // 业务价值调整
        if (task.businessValue) {
            priority += task.businessValue * 15
        }

        return Math.max(0, Math.min(100, priority))
    }

    private async selectQueue(task: Task, priority: number): Promise<TaskQueue> {
        const queues = await this.queueManager.getAllQueues()

        // 基于任务类型和优先级选择队列
        const candidateQueues = queues.filter(queue => {
            return queue.acceptedTypes.includes(task.type) &&
                   priority >= queue.minPriority &&
                   priority <= queue.maxPriority
        })

        if (candidateQueues.length === 0) {
            // 如果没有匹配的队列，使用默认队列
            const defaultQueue = queues.find(q => q.isDefault)
            if (!defaultQueue) {
                throw new Error('No suitable queue found for task')
            }
            return defaultQueue
        }

        // 选择负载最轻的候选队列
        return candidateQueues.reduce((lightest, current) => {
            return current.currentLoad < lightest.currentLoad ? current : lightest
        })
    }

    private selectScheduleStrategy(task: Task): ScheduleStrategy {
        // 基于任务特性选择调度策略
        if (task.requirements?.realTime) {
            return { type: 'immediate', priority: 100 }
        }

        if (task.requirements?.highAvailability) {
            return { type: 'high_availability', priority: 80 }
        }

        if (task.type === 'batch_processing') {
            return { type: 'batch', priority: 30 }
        }

        if (task.priority === 'high') {
            return { type: 'priority', priority: 70 }
        }

        return { type: 'fair_share', priority: 50 }
    }

    private async allocateExecutor(task: Task, strategy: ScheduleStrategy): Promise<Executor> {
        const availableExecutors = await this.executorRegistry.getAvailableExecutors(task.type)

        if (availableExecutors.length === 0) {
            throw new Error('No available executors for task type: ' + task.type)
        }

        switch (strategy.type) {
            case 'immediate':
                // 选择负载最轻的执行器
                return this.selectLeastLoadedExecutor(availableExecutors)

            case 'high_availability':
                // 选择高可用性执行器
                return this.selectHighAvailabilityExecutor(availableExecutors)

            case 'priority':
                // 选择最高性能执行器
                return this.selectHighestPerformanceExecutor(availableExecutors)

            case 'batch':
                // 选择成本效益最高的执行器
                return this.selectCostEffectiveExecutor(availableExecutors)

            case 'fair_share':
            default:
                // 使用负载均衡
                return this.loadBalancer.selectExecutor(availableExecutors, task)
        }
    }

    private selectLeastLoadedExecutor(executors: Executor[]): Executor {
        return executors.reduce((leastLoaded, current) => {
            const currentLoad = current.getCurrentLoad()
            const leastLoad = leastLoaded.getCurrentLoad()
            return currentLoad < leastLoad ? current : leastLoaded
        })
    }

    private selectHighAvailabilityExecutor(executors: Executor[]): Executor {
        // 选择高可用性执行器（考虑备用资源、故障恢复能力等）
        return executors.reduce((best, current) => {
            const currentScore = this.calculateAvailabilityScore(current)
            const bestScore = this.calculateAvailabilityScore(best)
            return currentScore > bestScore ? current : best
        })
    }

    private selectHighestPerformanceExecutor(executors: Executor[]): Executor {
        // 选择性能最高的执行器
        return executors.reduce((best, current) => {
            const currentScore = this.calculatePerformanceScore(current)
            const bestScore = this.calculatePerformanceScore(best)
            return currentScore > bestScore ? current : best
        })
    }

    private selectCostEffectiveExecutor(executors: Executor[]): Executor {
        // 选择成本效益最高的执行器
        return executors.reduce((best, current) => {
            const currentScore = this.calculateCostEffectivenessScore(current)
            const bestScore = this.calculateCostEffectivenessScore(best)
            return currentScore > bestScore ? current : best
        })
    }

    private calculateAvailabilityScore(executor: Executor): number {
        // 计算可用性分数
        const uptime = executor.getUptime() / 100 // 转换为百分比
        const failover = executor.hasFailover() ? 20 : 0
        const backup = executor.hasBackup() ? 10 : 0

        return uptime + failover + backup
    }

    private calculatePerformanceScore(executor: Executor): number {
        // 计算性能分数
        const cpuScore = executor.getCPUPerformance() * 0.3
        const memoryScore = executor.getMemoryPerformance() * 0.3
        const ioScore = executor.getIOPerformance() * 0.2
        const networkScore = executor.getNetworkPerformance() * 0.2

        return cpuScore + memoryScore + ioScore + networkScore
    }

    private calculateCostEffectivenessScore(executor: Executor): number {
        // 计算成本效益分数
        const performance = this.calculatePerformanceScore(executor)
        const cost = executor.getHourlyCost()

        return cost > 0 ? performance / cost : performance
    }

    private async queueTask(task: Task, queue: TaskQueue, executor: Executor, priority: number): Promise<QueuedTask> {
        const queuedTask: QueuedTask = {
            id: this.generateQueuedTaskId(),
            taskId: task.id,
            queueId: queue.id,
            executorId: executor.id,
            priority,
            status: 'queued',
            queuedAt: new Date(),
            estimatedStartTime: this.calculateEstimatedStartTimeForQueue(queue, priority),
            resourceRequirements: this.calculateRequiredResources(task)
        }

        await this.queueManager.enqueue(queue.id, queuedTask)

        return queuedTask
    }

    private calculateEstimatedStartTime(queuedTask: QueuedTask): Date {
        // 估算任务开始时间
        const queue = this.queueManager.getQueue(queuedTask.queueId)
        const aheadTasks = queue.tasks.filter(t =>
            t.status === 'queued' && t.priority > queuedTask.priority
        ).length

        const averageExecutionTime = queue.averageExecutionTime || 30000 // 30秒默认
        const estimatedWaitTime = aheadTasks * averageExecutionTime

        return new Date(Date.now() + estimatedWaitTime)
    }

    private calculateEstimatedStartTimeForQueue(queue: TaskQueue, priority: number): Date {
        const aheadTasks = queue.tasks.filter(t =>
            t.status === 'queued' && t.priority > priority
        ).length

        const averageExecutionTime = queue.averageExecutionTime || 30000
        const estimatedWaitTime = aheadTasks * averageExecutionTime

        return new Date(Date.now() + estimatedWaitTime)
    }

    private groupTasksByStrategy(tasks: Task[], priorities: number[]): TaskGroup[] {
        const groups: Map<string, TaskGroup> = new Map()

        tasks.forEach((task, index) => {
            const priority = priorities[index]
            const strategy = this.selectScheduleStrategy(task)
            const groupKey = `${strategy.type}_${Math.floor(priority / 20) * 20}` // 按策略和优先级分组

            if (!groups.has(groupKey)) {
                groups.set(groupKey, {
                    strategy,
                    priority: priority,
                    tasks: []
                })
            }

            groups.get(groupKey)!.tasks.push(task)
        })

        return Array.from(groups.values())
    }

    private async scheduleTaskGroup(group: TaskGroup): Promise<GroupScheduleResult> {
        const results: ScheduleResult[] = []
        const successfulTasks: string[] = []
        const failedTasks: string[] = []

        // 为组内的任务分配资源
        const resourceAssessment = await this.assessBatchResources(group.tasks)
        if (!resourceAssessment.sufficient) {
            // 如果资源不足，逐个调度
            for (const task of group.tasks) {
                try {
                    const result = await this.scheduleTask(task)
                    results.push(result)

                    if (result.success) {
                        successfulTasks.push(task.id)
                    } else {
                        failedTasks.push(task.id)
                    }
                } catch (error) {
                    failedTasks.push(task.id)
                    results.push({
                        success: false,
                        error: error.message,
                        scheduleTime: 0
                    })
                }
            }
        } else {
            // 资源充足，并行调度
            const taskPromises = group.tasks.map(task => this.scheduleTask(task))
            const taskResults = await Promise.all(taskPromises)

            results.push(...taskResults)

            taskResults.forEach((result, index) => {
                if (result.success) {
                    successfulTasks.push(group.tasks[index].id)
                } else {
                    failedTasks.push(group.tasks[index].id)
                }
            })
        }

        return {
            results,
            successfulTasks,
            failedTasks,
            strategy: group.strategy.type
        }
    }

    private generateQueuedTaskId(): string {
        return `queued_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
    }

    // 公共方法
    async getQueueStatus(queueId: string): Promise<QueueStatus> {
        return this.queueManager.getQueueStatus(queueId)
    }

    async getAllQueueStatuses(): Promise<QueueStatus[]> {
        return this.queueManager.getAllQueueStatuses()
    }

    async getExecutorStatus(executorId: string): Promise<ExecutorStatus> {
        const executor = this.executorRegistry.getExecutor(executorId)
        if (!executor) {
            throw new Error(`Executor not found: ${executorId}`)
        }

        return executor.getStatus()
    }

    async getAllExecutorStatuses(): Promise<ExecutorStatus[]> {
        const executors = await this.executorRegistry.getAllExecutors()
        return Promise.all(executors.map(executor => executor.getStatus()))
    }

    async rebalanceQueues(): Promise<void> {
        // 重新平衡队列负载
        const queues = await this.queueManager.getAllQueues()
        const avgLoad = queues.reduce((sum, q) => sum + q.currentLoad, 0) / queues.length

        for (const queue of queues) {
            if (queue.currentLoad > avgLoad * 1.5) {
                // 队列负载过高，重新分配任务
                await this.rebalanceQueue(queue, avgLoad)
            }
        }
    }

    private async rebalanceQueue(overloadedQueue: TaskQueue, targetLoad: number): Promise<void> {
        const tasksToMove = overloadedQueue.tasks
            .filter(t => t.status === 'queued')
            .slice(0, Math.floor(overloadedQueue.currentLoad - targetLoad))

        for (const task of tasksToMove) {
            // 重新调度任务到其他队列
            const newQueue = await this.selectAlternativeQueue(task, overloadedQueue.id)
            if (newQueue) {
                await this.queueManager.dequeue(overloadedQueue.id, task.id)
                await this.queueManager.enqueue(newQueue.id, task)
            }
        }
    }

    private async selectAlternativeQueue(task: Task, excludeQueueId: string): Promise<TaskQueue | null> {
        const queues = await this.queueManager.getAllQueues()
        const candidateQueues = queues.filter(q =>
            q.id !== excludeQueueId &&
            q.acceptedTypes.includes(task.type) &&
            q.currentLoad < q.capacity * 0.8 // 只选择负载低于80%的队列
        )

        if (candidateQueues.length === 0) {
            return null
        }

        // 选择负载最轻的队列
        return candidateQueues.reduce((lightest, current) =>
            current.currentLoad < lightest.currentLoad ? current : lightest
        )
    }
}

// 负载均衡器
class LoadBalancer {
    private strategies: Map<string, LoadBalancingStrategy> = new Map()
    private currentStrategy: string = 'round_robin'

    constructor() {
        this.initializeStrategies()
    }

    private initializeStrategies(): void {
        // 轮询策略
        this.strategies.set('round_robin', {
            name: 'round_robin',
            selectExecutor: (executors: Executor[], task: Task) => {
                const index = this.roundRobinIndex % executors.length
                this.roundRobinIndex++
                return executors[index]
            }
        })

        // 最少连接策略
        this.strategies.set('least_connections', {
            name: 'least_connections',
            selectExecutor: (executors: Executor[], task: Task) => {
                return executors.reduce((least, current) =>
                    current.getActiveConnections() < least.getActiveConnections() ? current : least
                )
            }
        })

        // 加权轮询策略
        this.strategies.set('weighted_round_robin', {
            name: 'weighted_round_robin',
            selectExecutor: (executors: Executor[], task: Task) => {
                const totalWeight = executors.reduce((sum, e) => sum + e.getWeight(), 0)
                let random = Math.random() * totalWeight

                for (const executor of executors) {
                    random -= executor.getWeight()
                    if (random <= 0) {
                        return executor
                    }
                }

                return executors[0]
            }
        })

        // 响应时间策略
        this.strategies.set('response_time', {
            name: 'response_time',
            selectExecutor: (executors: Executor[], task: Task) => {
                return executors.reduce((fastest, current) =>
                    current.getAverageResponseTime() < fastest.getAverageResponseTime() ? current : fastest
                )
            }
        })
    }

    private roundRobinIndex: number = 0

    selectExecutor(executors: Executor[], task: Task): Executor {
        const strategy = this.strategies.get(this.currentStrategy)
        if (!strategy) {
            throw new Error(`Load balancing strategy not found: ${this.currentStrategy}`)
        }

        return strategy.selectExecutor(executors, task)
    }

    setStrategy(strategyName: string): void {
        if (!this.strategies.has(strategyName)) {
            throw new Error(`Unknown load balancing strategy: ${strategyName}`)
        }
        this.currentStrategy = strategyName
    }

    getCurrentStrategy(): string {
        return this.currentStrategy
    }

    getAvailableStrategies(): string[] {
        return Array.from(this.strategies.keys())
    }
}

// 资源管理器
class ResourceManager {
    private resources: SystemResources = {
        cpu: { total: 100, used: 0, available: 100 },
        memory: { total: 8192, used: 0, available: 8192 }, // MB
        disk: { total: 100000, used: 0, available: 100000 }, // MB
        network: { total: 1000, used: 0, available: 1000 } // Mbps
    }

    private resourceAllocations: Map<string, ResourceAllocation> = new Map()

    async getAvailableResources(): Promise<AvailableResources> {
        // 实时获取系统资源信息
        await this.updateResourceUsage()

        return {
            cpu: this.resources.cpu.available,
            memory: this.resources.memory.available,
            disk: this.resources.disk.available,
            network: this.resources.network.available
        }
    }

    async allocateResources(taskId: string, requirements: ResourceRequirements): Promise<boolean> {
        await this.updateResourceUsage()

        // 检查资源是否足够
        if (
            requirements.cpu > this.resources.cpu.available ||
            requirements.memory > this.resources.memory.available ||
            requirements.disk > this.resources.disk.available ||
            requirements.network > this.resources.network.available
        ) {
            return false
        }

        // 分配资源
        this.resources.cpu.used += requirements.cpu
        this.resources.memory.used += requirements.memory
        this.resources.disk.used += requirements.disk
        this.resources.network.used += requirements.network

        this.resources.cpu.available = this.resources.cpu.total - this.resources.cpu.used
        this.resources.memory.available = this.resources.memory.total - this.resources.memory.used
        this.resources.disk.available = this.resources.disk.total - this.resources.disk.used
        this.resources.network.available = this.resources.network.total - this.resources.network.used

        // 记录分配
        this.resourceAllocations.set(taskId, {
            taskId,
            requirements,
            allocatedAt: new Date()
        })

        return true
    }

    async releaseResources(taskId: string): Promise<void> {
        const allocation = this.resourceAllocations.get(taskId)
        if (allocation) {
            // 释放资源
            this.resources.cpu.used -= allocation.requirements.cpu
            this.resources.memory.used -= allocation.requirements.memory
            this.resources.disk.used -= allocation.requirements.disk
            this.resources.network.used -= allocation.requirements.network

            this.resources.cpu.available = this.resources.cpu.total - this.resources.cpu.used
            this.resources.memory.available = this.resources.memory.total - this.resources.memory.used
            this.resources.disk.available = this.resources.disk.total - this.resources.disk.used
            this.resources.network.available = this.resources.network.total - this.resources.network.used

            // 确保不会出现负数
            this.resources.cpu.used = Math.max(0, this.resources.cpu.used)
            this.resources.memory.used = Math.max(0, this.resources.memory.used)
            this.resources.disk.used = Math.max(0, this.resources.disk.used)
            this.resources.network.used = Math.max(0, this.resources.network.used)

            this.resourceAllocations.delete(taskId)
        }
    }

    private async updateResourceUsage(): Promise<void> {
        // 更新系统资源使用情况
        // 这里应该调用系统监控API
        // 简化实现，使用模拟数据
    }

    getResourceUtilization(): ResourceUtilization {
        return {
            cpu: (this.resources.cpu.used / this.resources.cpu.total) * 100,
            memory: (this.resources.memory.used / this.resources.memory.total) * 100,
            disk: (this.resources.disk.used / this.resources.disk.total) * 100,
            network: (this.resources.network.used / this.resources.network.total) * 100
        }
    }
}
```

## 状态同步机制

### 状态管理系统

#### 1. 状态同步架构

```typescript
// src/core/workflow/state/StateManager.ts
export class StateManager {
    private stateStore: StateStore
    private eventBus: EventBus
    private conflictResolver: ConflictResolver
    private snapshotManager: SnapshotManager
    private stateValidator: StateValidator
    private metrics: StateMetrics

    constructor(
        stateStore: StateStore,
        eventBus: EventBus,
        conflictResolver: ConflictResolver,
        snapshotManager: SnapshotManager,
        stateValidator: StateValidator,
        metrics: StateMetrics
    ) {
        this.stateStore = stateStore
        this.eventBus = eventBus
        this.conflictResolver = conflictResolver
        this.snapshotManager = snapshotManager
        this.stateValidator = stateValidator
        this.metrics = metrics
    }

    async updateExecutionState(
        executionId: string,
        stateUpdate: StateUpdate
    ): Promise<StateUpdateResult> {
        const updateStart = Date.now()

        try {
            // 1. 获取当前状态
            const currentState = await this.stateStore.getExecutionState(executionId)
            const newState = this.applyStateUpdate(currentState, stateUpdate)

            // 2. 验证状态转换
            const validation = await this.stateValidator.validateTransition(currentState, newState)
            if (!validation.valid) {
                return {
                    success: false,
                    error: validation.error,
                    update_time: Date.now() - updateStart
                }
            }

            // 3. 检查冲突
            const conflictCheck = await this.checkForConflicts(executionId, stateUpdate)
            if (conflictCheck.hasConflict) {
                const resolvedState = await this.conflictResolver.resolve(
                    currentState,
                    newState,
                    conflictCheck.conflictingUpdate
                )

                if (resolvedState) {
                    await this.applyResolvedState(executionId, resolvedState, stateUpdate)
                } else {
                    return {
                        success: false,
                        error: 'State conflict could not be resolved',
                        update_time: Date.now() - updateStart
                    }
                }
            } else {
                // 4. 应用状态更新
                await this.applyStateUpdateToStore(executionId, newState, stateUpdate)
            }

            // 5. 创建状态快照（如果需要）
            if (this.shouldCreateSnapshot(newState)) {
                await this.snapshotManager.createSnapshot(executionId, newState)
            }

            // 6. 发布状态变更事件
            await this.eventBus.publish('execution.state_updated', {
                executionId,
                oldState: currentState,
                newState,
                update: stateUpdate,
                timestamp: new Date()
            })

            // 7. 记录指标
            await this.metrics.recordStateUpdate({
                executionId,
                updateType: stateUpdate.type,
                success: true,
                update_time: Date.now() - updateStart,
                timestamp: new Date()
            })

            return {
                success: true,
                executionId,
                newState,
                update_time: Date.now() - updateStart
            }

        } catch (error) {
            await this.metrics.recordStateUpdate({
                executionId,
                updateType: stateUpdate.type,
                success: false,
                error: error.message,
                update_time: Date.now() - updateStart,
                timestamp: new Date()
            })

            return {
                success: false,
                error: error.message,
                update_time: Date.now() - updateStart
            }
        }
    }

    async getExecutionState(executionId: string, version?: number): Promise<ExecutionState | null> {
        if (version) {
            // 获取特定版本的状态
            return this.stateStore.getExecutionStateVersion(executionId, version)
        } else {
            // 获取最新状态
            return this.stateStore.getExecutionState(executionId)
        }
    }

    async getStateHistory(executionId: string): Promise<StateHistory[]> {
        return this.stateStore.getStateHistory(executionId)
    }

    async getStateDiff(executionId: string, fromVersion: number, toVersion: number): Promise<StateDiff> {
        const fromState = await this.stateStore.getExecutionStateVersion(executionId, fromVersion)
        const toState = await this.stateStore.getExecutionStateVersion(executionId, toVersion)

        if (!fromState || !toState) {
            throw new Error('State version not found')
        }

        return this.calculateStateDiff(fromState, toState)
    }

    async createExecutionCheckpoint(executionId: string): Promise<Checkpoint> {
        const currentState = await this.stateStore.getExecutionState(executionId)
        if (!currentState) {
            throw new Error(`Execution not found: ${executionId}`)
        }

        const checkpoint: Checkpoint = {
            id: this.generateCheckpointId(),
            executionId,
            state: currentState,
            version: currentState.version,
            createdAt: new Date(),
            metadata: {
                totalSteps: currentState.steps.length,
                completedSteps: currentState.steps.filter(s => s.status === 'completed').length,
                failedSteps: currentState.steps.filter(s => s.status === 'failed').length
            }
        }

        await this.stateStore.saveCheckpoint(checkpoint)

        return checkpoint
    }

    async restoreFromCheckpoint(checkpointId: string): Promise<RestoreResult> {
        const restoreStart = Date.now()

        try {
            const checkpoint = await this.stateStore.getCheckpoint(checkpointId)
            if (!checkpoint) {
                throw new Error(`Checkpoint not found: ${checkpointId}`)
            }

            // 验证检查点有效性
            const validation = await this.validateCheckpoint(checkpoint)
            if (!validation.valid) {
                return {
                    success: false,
                    error: validation.error,
                    restore_time: Date.now() - restoreStart
                }
            }

            // 恢复状态
            await this.stateStore.restoreExecutionState(
                checkpoint.executionId,
                checkpoint.state
            )

            // 发布恢复事件
            await this.eventBus.publish('execution.restored', {
                executionId: checkpoint.executionId,
                checkpointId,
                timestamp: new Date()
            })

            return {
                success: true,
                executionId: checkpoint.executionId,
                checkpointId,
                restore_time: Date.now() - restoreStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                restore_time: Date.now() - restoreStart
            }
        }
    }

    private applyStateUpdate(currentState: ExecutionState, update: StateUpdate): ExecutionState {
        const newState: ExecutionState = JSON.parse(JSON.stringify(currentState)) // 深拷贝

        switch (update.type) {
            case 'step_started':
                this.applyStepStarted(newState, update)
                break
            case 'step_completed':
                this.applyStepCompleted(newState, update)
                break
            case 'step_failed':
                this.applyStepFailed(newState, update)
                break
            case 'step_retried':
                this.applyStepRetried(newState, update)
                break
            case 'execution_resumed':
                this.applyExecutionResumed(newState, update)
                break
            case 'execution_paused':
                this.applyExecutionPaused(newState, update)
                break
            case 'variables_updated':
                this.applyVariablesUpdated(newState, update)
                break
            case 'error_logged':
                this.applyErrorLogged(newState, update)
                break
            default:
                throw new Error(`Unknown state update type: ${update.type}`)
        }

        // 更新版本号
        newState.version = (currentState.version || 0) + 1
        newState.updatedAt = new Date()

        return newState
    }

    private applyStepStarted(state: ExecutionState, update: StateUpdate): void {
        const stepState = state.steps.find(s => s.stepId === update.stepId)
        if (stepState) {
            stepState.status = 'running'
            stepState.startTime = update.timestamp
            stepState.retryCount = update.retryCount || 0
        }
    }

    private applyStepCompleted(state: ExecutionState, update: StateUpdate): void {
        const stepState = state.steps.find(s => s.stepId === update.stepId)
        if (stepState) {
            stepState.status = 'completed'
            stepState.endTime = update.timestamp
            stepState.output = update.output
        }

        // 更新上下文
        if (update.output !== undefined) {
            state.context.stepResults[update.stepId] = update.output
        }
    }

    private applyStepFailed(state: ExecutionState, update: StateUpdate): void {
        const stepState = state.steps.find(s => s.stepId === update.stepId)
        if (stepState) {
            stepState.status = 'failed'
            stepState.endTime = update.timestamp
            stepState.error = update.error
        }

        // 记录错误
        state.context.errors.push({
            stepId: update.stepId,
            error: update.error,
            timestamp: update.timestamp
        })
    }

    private applyStepRetried(state: ExecutionState, update: StateUpdate): void {
        const stepState = state.steps.find(s => s.stepId === update.stepId)
        if (stepState) {
            stepState.status = 'pending'
            stepState.retryCount = (stepState.retryCount || 0) + 1
            stepState.startTime = null
            stepState.endTime = null
            stepState.error = null
        }
    }

    private applyExecutionResumed(state: ExecutionState, update: StateUpdate): void {
        state.status = 'running'
        state.resumedAt = update.timestamp
    }

    private applyExecutionPaused(state: ExecutionState, update: StateUpdate): void {
        state.status = 'paused'
        state.pausedAt = update.timestamp
    }

    private applyVariablesUpdated(state: ExecutionState, update: StateUpdate): void {
        if (update.variables) {
            Object.assign(state.context.variables, update.variables)
        }
    }

    private applyErrorLogged(state: ExecutionState, update: StateUpdate): void {
        if (update.error) {
            state.context.errors.push({
                stepId: update.stepId || 'system',
                error: update.error,
                timestamp: update.timestamp
            })
        }
    }

    private async checkForConflicts(
        executionId: string,
        update: StateUpdate
    ): Promise<ConflictCheck> {
        const currentState = await this.stateStore.getExecutionState(executionId)
        if (!currentState) {
            return { hasConflict: false }
        }

        // 检查是否有并发的状态更新
        const concurrentUpdates = await this.stateStore.getConcurrentUpdates(
            executionId,
            currentState.version
        )

        if (concurrentUpdates.length > 0) {
            return {
                hasConflict: true,
                conflictingUpdate: concurrentUpdates[0]
            }
        }

        return { hasConflict: false }
    }

    private async applyStateUpdateToStore(
        executionId: string,
        newState: ExecutionState,
        update: StateUpdate
    ): Promise<void> {
        await this.stateStore.updateExecutionState(executionId, newState)

        // 记录状态历史
        await this.stateStore.recordStateHistory({
            executionId,
            version: newState.version,
            update,
            timestamp: new Date()
        })
    }

    private async applyResolvedState(
        executionId: string,
        resolvedState: ExecutionState,
        originalUpdate: StateUpdate
    ): Promise<void> {
        await this.stateStore.updateExecutionState(executionId, resolvedState)

        // 记录冲突解决
        await this.stateStore.recordConflictResolution({
            executionId,
            originalUpdate,
            resolvedState,
            resolvedAt: new Date()
        })

        // 发布冲突解决事件
        await this.eventBus.publish('state.conflict_resolved', {
            executionId,
            resolvedAt: new Date()
        })
    }

    private shouldCreateSnapshot(state: ExecutionState): boolean {
        // 基于状态变化决定是否创建快照
        const completedSteps = state.steps.filter(s => s.status === 'completed').length
        const totalSteps = state.steps.length

        // 每25%的步骤完成时创建快照
        return completedSteps > 0 && completedSteps % Math.ceil(totalSteps / 4) === 0
    }

    private calculateStateDiff(fromState: ExecutionState, toState: ExecutionState): StateDiff {
        const changes: StateChange[] = []

        // 比较状态差异
        if (fromState.status !== toState.status) {
            changes.push({
                field: 'status',
                from: fromState.status,
                to: toState.status,
                type: 'updated'
            })
        }

        // 比较步骤状态
        fromState.steps.forEach((fromStep, index) => {
            const toStep = toState.steps[index]
            if (fromStep.status !== toStep.status) {
                changes.push({
                    field: `steps.${index}.status`,
                    from: fromStep.status,
                    to: toStep.status,
                    type: 'updated'
                })
            }
        })

        // 比较变量
        const variableChanges = this.calculateObjectDiff(
            fromState.context.variables,
            toState.context.variables,
            'context.variables'
        )
        changes.push(...variableChanges)

        return {
            fromVersion: fromState.version,
            toVersion: toState.version,
            changes,
            timestamp: new Date()
        }
    }

    private calculateObjectDiff(from: any, to: any, path: string): StateChange[] {
        const changes: StateChange[] = []

        const allKeys = new Set([...Object.keys(from), ...Object.keys(to)])

        for (const key of allKeys) {
            const currentPath = `${path}.${key}`
            const fromValue = from[key]
            const toValue = to[key]

            if (JSON.stringify(fromValue) !== JSON.stringify(toValue)) {
                changes.push({
                    field: currentPath,
                    from: fromValue,
                    to: toValue,
                    type: 'updated'
                })
            }
        }

        return changes
    }

    private async validateCheckpoint(checkpoint: Checkpoint): Promise<CheckpointValidation> {
        // 验证检查点是否仍然有效
        const currentState = await this.stateStore.getExecutionState(checkpoint.executionId)
        if (!currentState) {
            return {
                valid: false,
                error: 'Execution not found'
            }
        }

        // 检查版本兼容性
        if (checkpoint.version > currentState.version) {
            return {
                valid: false,
                error: 'Checkpoint version is newer than current state'
            }
        }

        // 检查时间有效性（检查点不能太旧）
        const maxAge = 24 * 60 * 60 * 1000 // 24小时
        if (Date.now() - checkpoint.createdAt.getTime() > maxAge) {
            return {
                valid: false,
                error: 'Checkpoint is too old'
            }
        }

        return { valid: true }
    }

    private generateCheckpointId(): string {
        return `checkpoint_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
    }

    // 状态同步API
    async syncExecutionState(executionId: string): Promise<SyncResult> {
        const syncStart = Date.now()

        try {
            // 获取远程状态
            const remoteState = await this.fetchRemoteState(executionId)

            // 获取本地状态
            const localState = await this.stateStore.getExecutionState(executionId)

            if (!remoteState) {
                return {
                    success: false,
                    error: 'Remote state not found',
                    sync_time: Date.now() - syncStart
                }
            }

            if (!localState) {
                // 本地没有状态，使用远程状态
                await this.stateStore.updateExecutionState(executionId, remoteState)
                return {
                    success: true,
                    executionId,
                    action: 'initialized',
                    sync_time: Date.now() - syncStart
                }
            }

            // 比较状态版本
            if (remoteState.version > localState.version) {
                // 远程状态更新，应用更新
                const diff = this.calculateStateDiff(localState, remoteState)
                await this.stateStore.updateExecutionState(executionId, remoteState)

                return {
                    success: true,
                    executionId,
                    action: 'updated',
                    diff,
                    sync_time: Date.now() - syncStart
                }
            } else if (localState.version > remoteState.version) {
                // 本地状态更新，推送到远程
                await this.pushStateToRemote(executionId, localState)

                return {
                    success: true,
                    executionId,
                    action: 'pushed',
                    sync_time: Date.now() - syncStart
                }
            } else {
                // 状态同步
                return {
                    success: true,
                    executionId,
                    action: 'synchronized',
                    sync_time: Date.now() - syncStart
                }
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                sync_time: Date.now() - syncStart
            }
        }
    }

    private async fetchRemoteState(executionId: string): Promise<ExecutionState | null> {
        // 从远程存储获取状态
        // 简化实现
        return null
    }

    private async pushStateToRemote(executionId: string, state: ExecutionState): Promise<void> {
        // 推送状态到远程存储
        // 简化实现
    }

    // 健康检查
    async checkStateConsistency(executionId: string): Promise<ConsistencyCheck> {
        try {
            const localState = await this.stateStore.getExecutionState(executionId)
            const remoteState = await this.fetchRemoteState(executionId)

            if (!localState && !remoteState) {
                return {
                    consistent: true,
                    message: 'No state exists'
                }
            }

            if (!localState || !remoteState) {
                return {
                    consistent: false,
                    message: 'State exists only on one side',
                    localExists: !!localState,
                    remoteExists: !!remoteState
                }
            }

            if (localState.version !== remoteState.version) {
                return {
                    consistent: false,
                    message: 'Version mismatch',
                    localVersion: localState.version,
                    remoteVersion: remoteState.version
                }
            }

            // 深度比较状态内容
            const contentMatch = this.deepCompareStates(localState, remoteState)
            if (!contentMatch) {
                return {
                    consistent: false,
                    message: 'Content mismatch'
                }
            }

            return {
                consistent: true,
                message: 'States are consistent'
            }

        } catch (error) {
            return {
                consistent: false,
                message: `Consistency check failed: ${error.message}`
            }
        }
    }

    private deepCompareStates(state1: ExecutionState, state2: ExecutionState): boolean {
        // 深度比较两个状态是否相同
        return JSON.stringify(state1) === JSON.stringify(state2)
    }

    // 状态清理
    async cleanupOldStates(maxAge: number = 7 * 24 * 60 * 60 * 1000): Promise<CleanupResult> {
        const cleanupStart = Date.now()

        try {
            const cutoffDate = new Date(Date.now() - maxAge)

            const deletedStates = await this.stateStore.deleteOldStates(cutoffDate)
            const deletedHistory = await this.stateStore.deleteOldHistory(cutoffDate)
            const deletedSnapshots = await this.snapshotManager.deleteOldSnapshots(cutoffDate)

            await this.metrics.recordCleanup({
                deletedStates,
                deletedHistory,
                deletedSnapshots,
                cleanup_time: Date.now() - cleanupStart,
                timestamp: new Date()
            })

            return {
                success: true,
                deletedStates,
                deletedHistory,
                deletedSnapshots,
                cleanup_time: Date.now() - cleanupStart
            }

        } catch (error) {
            return {
                success: false,
                error: error.message,
                cleanup_time: Date.now() - cleanupStart
            }
        }
    }
}

// 冲突解决器
class ConflictResolver {
    private strategies: Map<string, ConflictResolutionStrategy> = new Map()

    constructor() {
        this.initializeStrategies()
    }

    private initializeStrategies(): void {
        // 最后写入获胜策略
        this.strategies.set('last_write_wins', {
            name: 'last_write_wins',
            resolve: (currentState, newState, conflictingUpdate) => {
                return newState
            }
        })

        // 手动合并策略
        this.strategies.set('manual_merge', {
            name: 'manual_merge',
            resolve: (currentState, newState, conflictingUpdate) => {
                // 这里应该触发手动合并流程
                // 简化实现，返回null表示需要人工干预
                return null
            }
        })

        // 字段级别合并策略
        this.strategies.set('field_merge', {
            name: 'field_merge',
            resolve: (currentState, newState, conflictingUpdate) => {
                return this.mergeFieldByField(currentState, newState, conflictingUpdate)
            }
        })
    }

    async resolve(
        currentState: ExecutionState,
        newState: ExecutionState,
        conflictingUpdate: StateUpdate
    ): Promise<ExecutionState | null> {
        const strategy = this.strategies.get('field_merge') // 默认使用字段合并策略
        if (!strategy) {
            throw new Error('Conflict resolution strategy not found')
        }

        return strategy.resolve(currentState, newState, conflictingUpdate)
    }

    private mergeFieldByField(
        currentState: ExecutionState,
        newState: ExecutionState,
        conflictingUpdate: StateUpdate
    ): ExecutionState {
        const mergedState: ExecutionState = JSON.parse(JSON.stringify(currentState))

        // 基于更新类型进行智能合并
        switch (conflictingUpdate.type) {
            case 'step_completed':
                // 步骤完成通常是幂等的，可以使用新状态
                return newState

            case 'variables_updated':
                // 变量更新可以合并
                if (conflictingUpdate.variables) {
                    Object.assign(mergedState.context.variables, conflictingUpdate.variables)
                }
                break

            case 'error_logged':
                // 错误日志可以合并
                if (conflictingUpdate.error) {
                    mergedState.context.errors.push({
                        stepId: conflictingUpdate.stepId || 'system',
                        error: conflictingUpdate.error,
                        timestamp: conflictingUpdate.timestamp
                    })
                }
                break

            default:
                // 对于其他类型的更新，使用时间戳较新的
                if (newState.updatedAt > currentState.updatedAt) {
                    return newState
                }
                break
        }

        mergedState.version = Math.max(currentState.version, newState.version) + 1
        mergedState.updatedAt = new Date()

        return mergedState
    }
}

// 状态验证器
class StateValidator {
    async validateTransition(
        fromState: ExecutionState | null,
        toState: ExecutionState
    ): Promise<StateValidation> {
        if (!fromState) {
            // 初始状态验证
            return this.validateInitialState(toState)
        }

        // 状态转换验证
        const validTransitions = this.getValidStateTransitions()
        const fromStatus = fromState.status
        const toStatus = toState.status

        if (!validTransitions[fromStatus]?.includes(toStatus)) {
            return {
                valid: false,
                error: `Invalid state transition from ${fromStatus} to ${toStatus}`
            }
        }

        // 步骤状态验证
        const stepValidation = this.validateStepStates(fromState, toState)
        if (!stepValidation.valid) {
            return stepValidation
        }

        // 数据完整性验证
        const integrityValidation = this.validateDataIntegrity(toState)
        if (!integrityValidation.valid) {
            return integrityValidation
        }

        return { valid: true }
    }

    private validateInitialState(state: ExecutionState): StateValidation {
        if (state.status !== 'created' && state.status !== 'running') {
            return {
                valid: false,
                error: `Initial state must be 'created' or 'running', got '${state.status}'`
            }
        }

        if (!state.steps || state.steps.length === 0) {
            return {
                valid: false,
                error: 'Initial state must have at least one step'
            }
        }

        return { valid: true }
    }

    private getValidStateTransitions(): Record<string, string[]> {
        return {
            'created': ['running'],
            'running': ['paused', 'completed', 'failed'],
            'paused': ['running', 'completed', 'failed'],
            'completed': [],
            'failed': ['running'] // 允许重试
        }
    }

    private validateStepStates(fromState: ExecutionState, toState: ExecutionState): StateValidation {
        for (let i = 0; i < toState.steps.length; i++) {
            const fromStep = fromState.steps[i]
            const toStep = toState.steps[i]

            if (fromStep && toStep) {
                // 验证步骤状态转换的有效性
                const validStepTransitions = this.getValidStepStateTransitions()
                const fromStatus = fromStep.status
                const toStatus = toStep.status

                if (!validStepTransitions[fromStatus]?.includes(toStatus)) {
                    return {
                        valid: false,
                        error: `Invalid step state transition for step ${i}: ${fromStatus} -> ${toStatus}`
                    }
                }
            }
        }

        return { valid: true }
    }

    private getValidStepStateTransitions(): Record<string, string[]> {
        return {
            'pending': ['running'],
            'running': ['completed', 'failed'],
            'completed': [],
            'failed': ['pending'], // 允许重试
            'skipped': []
        }
    }

    private validateDataIntegrity(state: ExecutionState): StateValidation {
        // 验证必需字段
        if (!state.executionId) {
            return {
                valid: false,
                error: 'Execution ID is required'
            }
        }

        if (!state.workflowId) {
            return {
                valid: false,
                error: 'Workflow ID is required'
            }
        }

        if (!state.status) {
            return {
                valid: false,
                error: 'Status is required'
            }
        }

        // 验证步骤数据完整性
        if (!state.steps || state.steps.length === 0) {
            return {
                valid: false,
                error: 'At least one step is required'
            }
        }

        for (const step of state.steps) {
            if (!step.stepId) {
                return {
                    valid: false,
                    error: 'Step ID is required'
                }
            }

            if (!step.status) {
                return {
                    valid: false,
                    error: 'Step status is required'
                }
            }
        }

        return { valid: true }
    }
}
```

## 总结

KiloCode 的任务自动化工作流引擎展现了以下技术特点：

### 1. **灵活的工作流定义**
- 支持多种步骤类型（任务、并行、条件、循环）
- 强类型的工作流定义语言
- 完整的验证和错误处理机制

### 2. **智能任务调度**
- 多种调度策略（轮询、最少连接、加权轮询）
- 动态资源管理和负载均衡
- 批量任务优化处理

### 3. **可靠的状态同步**
- 版本化的状态管理
- 冲突检测和自动解决
- 状态快照和恢复机制

### 4. **高可用性设计**
- 任务重试和错误恢复
- 执行器健康检查
- 状态一致性保证

### 5. **可观测性支持**
- 完整的执行过程追踪
- 性能指标监控
- 详细的日志和审计

这种架构设计使得 KiloCode 能够处理复杂的任务自动化场景，确保任务执行的可靠性和一致性。对于构建企业级的工作流引擎，这种架构模式具有重要的参考价值。