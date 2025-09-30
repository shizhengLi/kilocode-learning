# KiloCode 代码库检索机制深度分析

## 系统架构概览

KiloCode 的代码库检索系统是一个多层次的智能搜索架构，结合了传统文件系统遍历、现代代码解析技术和语义向量搜索。系统采用分层设计，确保高性能和可扩展性。

### 核心组件关系图

```
用户查询
    ↓
搜索服务 (CodeIndexSearchService)
    ↓
嵌入器 (Embedder) → 向量化
    ↓
向量存储 (QdrantVectorStore) → 语义搜索
    ↓
代码解析器 (CodeParser) ← 文件系统
    ↓
结果处理和格式化
```

## 1. 代码解析与索引系统

### 1.1 Tree-sitter 集成架构

```typescript
// src/services/tree-sitter/index.ts
export class TreeSitterManager {
    private parsers: Map<string, Parser> = new Map()
    private languages: Map<string, Language> = new Map()
    private queries: Map<string, Query> = new Map()

    async initialize(): Promise<void> {
        // 初始化支持的编程语言
        await this.initializeLanguages()
    }

    private async initializeLanguages(): Promise<void> {
        const languageConfigs = [
            { name: 'javascript', module: TreeSitterJavascript },
            { name: 'typescript', module: TreeSitterTypescript },
            { name: 'python', module: TreeSitterPython },
            { name: 'rust', module: TreeSitterRust },
            { name: 'go', module: TreeSitterGo },
            // ... 更多语言
        ]

        for (const config of languageConfigs) {
            try {
                const Parser = await import('web-tree-sitter')
                const parser = new Parser()
                const language = await config.module

                await parser.setLanguage(language)

                this.parsers.set(config.name, parser)
                this.languages.set(config.name, language)

                // 加载语言特定的查询
                await this.loadLanguageQueries(config.name, language)
            } catch (error) {
                console.warn(`Failed to initialize ${config.name}:`, error)
            }
        }
    }

    private async loadLanguageQueries(languageName: string, language: Language): Promise<void> {
        const queryPatterns = {
            javascript: `
                (function_declaration) @function
                (class_declaration) @class
                (method_definition) @method
                (variable_declaration) @variable
                (import_statement) @import
                (export_statement) @export
            `,
            python: `
                (function_definition) @function
                (class_definition) @class
                (import_statement) @import
                (assignment) @assignment
            `,
            // ... 更多语言的查询模式
        }

        const pattern = queryPatterns[languageName as keyof typeof queryPatterns]
        if (pattern) {
            try {
                const query = language.query(pattern)
                this.queries.set(languageName, query)
            } catch (error) {
                console.warn(`Failed to create query for ${languageName}:`, error)
            }
        }
    }

    parseCode(code: string, languageName: string): ParseResult {
        const parser = this.parsers.get(languageName)
        const query = this.queries.get(languageName)

        if (!parser) {
            // 降级到行分割
            return this.fallbackParse(code)
        }

        try {
            const tree = parser.parse(code)
            const captures = query?.captures(tree.rootNode) || []

            return {
                success: true,
                tree,
                captures,
                language: languageName
            }
        } catch (error) {
            console.warn(`Parse failed for ${languageName}:`, error)
            return this.fallbackParse(code)
        }
    }

    private fallbackParse(code: string): ParseResult {
        // 基于行的简单分割作为降级方案
        const lines = code.split('\n')
        const chunks: CodeChunk[] = []

        for (let i = 0; i < lines.length; i += 20) { // 每20行一个块
            const chunk = lines.slice(i, i + 20).join('\n')
            chunks.push({
                content: chunk,
                startLine: i + 1,
                endLine: Math.min(i + 20, lines.length)
            })
        }

        return {
            success: false,
            chunks,
            language: 'text'
        }
    }
}
```

### 1.2 智能代码分块器

```typescript
// src/services/code-index/processors/parser.ts
export class CodeParser {
    private treeSitter: TreeSitterManager
    private languageDetector: LanguageDetector

    async parseFile(fileInfo: FileInfo): Promise<CodeChunk[]> {
        const content = await fs.readFile(fileInfo.filePath, 'utf-8')
        const language = this.languageDetector.detect(fileInfo.filePath, content)

        // 基于语言选择解析策略
        switch (language) {
            case 'markdown':
                return this.parseMarkdown(content, fileInfo)
            case 'javascript':
            case 'typescript':
            case 'python':
            case 'java':
            case 'go':
            case 'rust':
            case 'c':
            case 'cpp':
            case 'csharp':
            case 'php':
                return this.parseProgrammingLanguage(content, language, fileInfo)
            default:
                return this.parseGeneric(content, fileInfo)
        }
    }

    private async parseProgrammingLanguage(
        content: string,
        language: string,
        fileInfo: FileInfo
    ): Promise<CodeChunk[]> {
        const parseResult = this.treeSitter.parseCode(content, language)

        if (parseResult.success && parseResult.captures) {
            return this.createSemanticChunks(parseResult, fileInfo)
        } else {
            // 降级到智能行分割
            return this.createIntelligentLineChunks(content, fileInfo)
        }
    }

    private createSemanticChunks(parseResult: ParseResult, fileInfo: FileInfo): CodeChunk[] {
        const chunks: CodeChunk[] = []
        const captures = parseResult.captures

        // 按语法节点分组
        const nodesByType = this.groupNodesByType(captures)

        // 为每种类型的节点创建代码块
        for (const [type, nodes] of Object.entries(nodesByType)) {
            for (const node of nodes) {
                const chunk = this.createChunkFromNode(node, type, fileInfo)
                if (chunk && this.isValidChunk(chunk)) {
                    chunks.push(chunk)
                }
            }
        }

        // 如果语义分块失败，使用智能行分割
        if (chunks.length === 0) {
            return this.createIntelligentLineChunks(
                parseResult.tree?.rootNode.text || '',
                fileInfo
            )
        }

        return chunks
    }

    private createChunkFromNode(node: any, type: string, fileInfo: FileInfo): CodeChunk | null {
        const startLine = node.startPosition.row + 1
        const endLine = node.endPosition.row + 1
        const content = node.text

        // 验证块大小
        if (content.length < 50 || content.length > 1000) {
            return null
        }

        return {
            content: content.trim(),
            startLine,
            endLine,
            filePath: fileInfo.filePath,
            language: fileInfo.language,
            metadata: {
                nodeType: type,
                identifier: this.extractIdentifier(node, type),
                parentContext: this.getParentContext(node),
                complexity: this.calculateComplexity(node)
            }
        }
    }

    private createIntelligentLineChunks(content: string, fileInfo: FileInfo): CodeChunk[] {
        const lines = content.split('\n')
        const chunks: CodeChunk[] = []
        let currentChunk: string[] = []
        let startLine = 1

        for (let i = 0; i < lines.length; i++) {
            const line = lines[i].trim()

            // 空行分隔逻辑块
            if (line === '' && currentChunk.length > 0) {
                const chunk = this.createLineChunk(currentChunk, startLine, fileInfo)
                if (chunk) {
                    chunks.push(chunk)
                }
                currentChunk = []
                startLine = i + 2 // 跳过空行
            } else if (line) {
                currentChunk.push(line)

                // 检查块大小
                const chunkSize = currentChunk.join('\n').length
                if (chunkSize > 800) { // 接近最大限制
                    const chunk = this.createLineChunk(currentChunk, startLine, fileInfo)
                    if (chunk) {
                        chunks.push(chunk)
                    }
                    currentChunk = []
                    startLine = i + 1
                }
            }
        }

        // 处理最后一个块
        if (currentChunk.length > 0) {
            const chunk = this.createLineChunk(currentChunk, startLine, fileInfo)
            if (chunk) {
                chunks.push(chunk)
            }
        }

        return chunks
    }

    private createLineChunk(lines: string[], startLine: number, fileInfo: FileInfo): CodeChunk | null {
        const content = lines.join('\n').trim()

        if (content.length < 50 || content.length > 1000) {
            return null
        }

        return {
            content,
            startLine,
            endLine: startLine + lines.length - 1,
            filePath: fileInfo.filePath,
            language: fileInfo.language,
            metadata: {
                chunkType: 'line-based',
                lineCount: lines.length,
                estimatedComplexity: this.estimateComplexity(lines)
            }
        }
    }

    private parseMarkdown(content: string, fileInfo: FileInfo): CodeChunk[] {
        const chunks: CodeChunk[] = []

        // 按标题分割
        const sections = content.split(/^#{1,6}\s+/gm)

        for (let i = 0; i < sections.length; i++) {
            const section = sections[i].trim()
            if (section.length > 50 && section.length < 1000) {
                chunks.push({
                    content: section,
                    startLine: this.getLineNumber(content, section),
                    endLine: this.getLineNumber(content, section) + section.split('\n').length - 1,
                    filePath: fileInfo.filePath,
                    language: 'markdown',
                    metadata: {
                        sectionType: i === 0 ? 'header' : 'content',
                        sectionIndex: i
                    }
                })
            }
        }

        return chunks
    }
}
```

### 1.3 高性能文件扫描器

```typescript
// src/services/code-index/processors/scanner.ts
export class DirectoryScanner {
    private concurrencyLimit: number = 10
    private batchSize: number = 60
    private fileCache: Map<string, FileCacheEntry> = new Map()

    async scanDirectory(directoryPath: string): Promise<ScanResult> {
        const startTime = Date.now()

        try {
            // 1. 递归获取所有文件
            const allFiles = await this.listFiles(directoryPath)

            // 2. 过滤支持的文件类型
            const supportedFiles = this.filterSupportedFiles(allFiles)

            // 3. 并发处理文件
            const processedFiles = await this.processFilesConcurrently(supportedFiles)

            // 4. 生成统计信息
            const stats = this.generateStatistics(processedFiles, startTime)

            return {
                success: true,
                files: processedFiles,
                stats,
                duration: Date.now() - startTime
            }
        } catch (error) {
            console.error('Directory scan failed:', error)
            return {
                success: false,
                error: error.message,
                files: [],
                stats: this.generateEmptyStats(),
                duration: Date.now() - startTime
            }
        }
    }

    private async listFiles(directoryPath: string): Promise<FileInfo[]> {
        const { globby } = await import('globby')

        const patterns = [
            '**/*',
            '!**/node_modules/**',
            '!**/.git/**',
            '!**/.vscode/**',
            '!**/dist/**',
            '!**/build/**',
            '!**/*.min.js',
            '!**/*.map'
        ]

        const filePaths = await globby(patterns, {
            cwd: directoryPath,
            absolute: true,
            gitignore: true,
            ignoreFiles: ['.rooignore']
        })

        // 获取文件信息
        const fileInfos: FileInfo[] = []
        for (const filePath of filePaths) {
            try {
                const stats = await fs.stat(filePath)
                fileInfos.push({
                    filePath,
                    size: stats.size,
                    lastModified: stats.mtime,
                    language: this.detectLanguage(filePath)
                })
            } catch (error) {
                console.warn(`Failed to stat ${filePath}:`, error)
            }
        }

        return fileInfos
    }

    private async processFilesConcurrently(files: FileInfo[]): Promise<ProcessedFile[]> {
        const results: ProcessedFile[] = []
        const semaphore = new Semaphore(this.concurrencyLimit)

        const tasks = files.map(async (file) => {
            const release = await semaphore.acquire()
            try {
                const result = await this.processSingleFile(file)
                return result
            } finally {
                release()
            }
        })

        const batchResults = await Promise.all(tasks)
        return batchResults.filter(result => result !== null) as ProcessedFile[]
    }

    private async processSingleFile(fileInfo: FileInfo): Promise<ProcessedFile | null> {
        try {
            // 检查文件大小限制
            if (fileInfo.size > 1024 * 1024) { // 1MB 限制
                return null
            }

            // 检查文件缓存
            const cacheKey = this.getCacheKey(fileInfo)
            const cached = this.fileCache.get(cacheKey)

            if (cached && cached.hash === await this.calculateFileHash(fileInfo.filePath)) {
                return cached.data
            }

            // 解析文件
            const chunks = await this.parser.parseFile(fileInfo)

            if (chunks.length === 0) {
                return null
            }

            const processedFile: ProcessedFile = {
                ...fileInfo,
                chunks,
                chunkCount: chunks.length,
                totalSize: chunks.reduce((sum, chunk) => sum + chunk.content.length, 0)
            }

            // 更新缓存
            this.fileCache.set(cacheKey, {
                hash: await this.calculateFileHash(fileInfo.filePath),
                data: processedFile,
                timestamp: Date.now()
            })

            return processedFile
        } catch (error) {
            console.warn(`Failed to process ${fileInfo.filePath}:`, error)
            return null
        }
    }

    private detectLanguage(filePath: string): string {
        const ext = path.extname(filePath).toLowerCase()
        const languageMap: Record<string, string> = {
            '.js': 'javascript',
            '.ts': 'typescript',
            '.jsx': 'javascript',
            '.tsx': 'typescript',
            '.py': 'python',
            '.java': 'java',
            '.go': 'go',
            '.rs': 'rust',
            '.c': 'c',
            '.cpp': 'cpp',
            '.cs': 'csharp',
            '.php': 'php',
            '.rb': 'ruby',
            '.swift': 'swift',
            '.kt': 'kotlin',
            '.scala': 'scala',
            '.md': 'markdown',
            '.json': 'json',
            '.yaml': 'yaml',
            '.yml': 'yaml',
            '.xml': 'xml',
            '.html': 'html',
            '.css': 'css',
            '.scss': 'scss',
            '.less': 'less',
            '.sql': 'sql',
            '.sh': 'bash',
            '.dockerfile': 'docker'
        }

        return languageMap[ext] || 'text'
    }

    private generateStatistics(files: ProcessedFile[], startTime: number): ScanStatistics {
        const totalFiles = files.length
        const totalChunks = files.reduce((sum, file) => sum + file.chunkCount, 0)
        const totalSize = files.reduce((sum, file) => sum + file.totalSize, 0)
        const avgChunksPerFile = totalFiles > 0 ? totalChunks / totalFiles : 0
        const avgSizePerChunk = totalChunks > 0 ? totalSize / totalChunks : 0

        const languageStats = files.reduce((stats, file) => {
            stats[file.language] = (stats[file.language] || 0) + 1
            return stats
        }, {} as Record<string, number>)

        return {
            totalFiles,
            totalChunks,
            totalSize,
            avgChunksPerFile,
            avgSizePerChunk,
            languageDistribution: languageStats,
            duration: Date.now() - startTime,
            filesProcessed: files.length
        }
    }
}
```

## 2. 搜索算法与排序机制

### 2.1 多策略搜索实现

```typescript
// src/services/code-index/search-service.ts
export class CodeIndexSearchService {
    private vectorStore: IVectorStore
    private embedder: IEmbedder
    private textSearcher: TextSearchEngine
    private hybridScorer: HybridScorer

    async search(query: string, options: SearchOptions = {}): Promise<SearchResult[]> {
        const {
            directoryPrefix,
            maxResults = 10,
            minScore = 0.4,
            searchType = 'hybrid'
        } = options

        // 并行执行不同类型的搜索
        const [semanticResults, textResults] = await Promise.all([
            this.semanticSearch(query, { directoryPrefix, maxResults: maxResults * 2 }),
            this.textSearch(query, { directoryPrefix, maxResults: maxResults * 2 })
        ])

        // 融合搜索结果
        const fusedResults = this.fuseResults(semanticResults, textResults, searchType)

        // 最终排序和过滤
        const finalResults = this.rankAndFilterResults(fusedResults, {
            maxResults,
            minScore
        })

        return finalResults
    }

    private async semanticSearch(query: string, options: SemanticSearchOptions): Promise<SearchResult[]> {
        try {
            // 生成查询嵌入
            const embeddingResponse = await this.embedder.createEmbeddings([query])

            if (!embeddingResponse || !embeddingResponse.embeddings[0]) {
                return []
            }

            const queryVector = embeddingResponse.embeddings[0]

            // 执行向量搜索
            const vectorResults = await this.vectorStore.search(
                queryVector,
                options.directoryPrefix,
                0.1, // 较低的阈值，后面再过滤
                options.maxResults
            )

            return vectorResults.map(result => ({
                id: result.id,
                score: result.score,
                type: 'semantic',
                payload: result.payload,
                relevance: this.calculateSemanticRelevance(result, query)
            }))
        } catch (error) {
            console.warn('Semantic search failed:', error)
            return []
        }
    }

    private async textSearch(query: string, options: TextSearchOptions): Promise<SearchResult[]> {
        try {
            // 使用 ripgrep 进行快速文本搜索
            const textResults = await this.textSearcher.search(query, {
                directoryPrefix: options.directoryPrefix,
                maxResults: options.maxResults
            })

            return textResults.map(result => ({
                id: this.generateTextSearchId(result),
                score: this.calculateTextScore(result, query),
                type: 'text',
                payload: result,
                relevance: this.calculateTextRelevance(result, query)
            }))
        } catch (error) {
            console.warn('Text search failed:', error)
            return []
        }
    }

    private fuseResults(
        semanticResults: SearchResult[],
        textResults: SearchResult[],
        searchType: string
    ): SearchResult[] {
        switch (searchType) {
            case 'semantic':
                return semanticResults
            case 'text':
                return textResults
            case 'hybrid':
            default:
                return this.hybridScorer.fuse(semanticResults, textResults)
        }
    }

    private rankAndFilterResults(
        results: SearchResult[],
        options: RankOptions
    ): SearchResult[] {
        // 1. 按相关性排序
        results.sort((a, b) => b.relevance - a.relevance)

        // 2. 过滤低分结果
        const filteredResults = results.filter(result =>
            result.relevance >= options.minScore
        )

        // 3. 限制结果数量
        return filteredResults.slice(0, options.maxResults)
    }

    private calculateSemanticRelevance(result: VectorStoreSearchResult, query: string): number {
        // 基础向量相似度
        let relevance = result.score

        // 文件名匹配加分
        const fileName = path.basename(result.payload.filePath)
        if (fileName.toLowerCase().includes(query.toLowerCase())) {
            relevance += 0.2
        }

        // 代码长度权重
        const codeLength = result.payload.codeChunk.length
        if (codeLength > 100 && codeLength < 500) {
            relevance += 0.1 // 中等长度的代码块更相关
        }

        // 语言匹配（如果查询包含语言关键词）
        if (this.hasLanguageKeyword(query, result.payload.language)) {
            relevance += 0.15
        }

        return Math.min(relevance, 1.0)
    }

    private calculateTextRelevance(result: any, query: string): number {
        let relevance = result.score

        // 精确匹配加分
        if (result.exactMatches > 0) {
            relevance += result.exactMatches * 0.1
        }

        // 上下文匹配加分
        if (result.contextMatches > 0) {
            relevance += result.contextMatches * 0.05
        }

        return Math.min(relevance, 1.0)
    }
}
```

### 2.2 混合搜索评分器

```typescript
// src/services/code-index/scoring/hybrid-scorer.ts
export class HybridScorer {
    private weights: ScoringWeights = {
        semantic: 0.7,
        text: 0.2,
        recency: 0.05,
        popularity: 0.05
    }

    fuse(semanticResults: SearchResult[], textResults: SearchResult[]): SearchResult[] {
        const fusedMap = new Map<string, FusedResult>()

        // 处理语义搜索结果
        for (const result of semanticResults) {
            const key = this.getResultKey(result)
            if (!fusedMap.has(key)) {
                fusedMap.set(key, {
                    key,
                    semanticScore: result.score,
                    textScore: 0,
                    semanticResult: result,
                    textResult: null
                })
            }
        }

        // 处理文本搜索结果
        for (const result of textResults) {
            const key = this.getResultKey(result)
            const existing = fusedMap.get(key)

            if (existing) {
                existing.textScore = result.score
                existing.textResult = result
            } else {
                fusedMap.set(key, {
                    key,
                    semanticScore: 0,
                    textScore: result.score,
                    semanticResult: null,
                    textResult: result
                })
            }
        }

        // 计算融合分数
        const fusedResults = Array.from(fusedMap.values()).map(fused => {
            const combinedScore = this.calculateCombinedScore(fused)
            return {
                ...this.getBestResult(fused),
                score: combinedScore,
                fusionType: this.getFusionType(fused)
            }
        })

        return fusedResults
    }

    private calculateCombinedScore(fused: FusedResult): number {
        const { semantic, text, recency, popularity } = this.weights

        let score = 0

        // 语义搜索权重
        if (fused.semanticScore > 0) {
            score += fused.semanticScore * semantic
        }

        // 文本搜索权重
        if (fused.textScore > 0) {
            score += fused.textScore * text
        }

        // 新鲜度加分
        const recencyBonus = this.calculateRecencyBonus(fused)
        score += recencyBonus * recency

        // 流行度加分
        const popularityBonus = this.calculatePopularityBonus(fused)
        score += popularityBonus * popularity

        return Math.min(score, 1.0)
    }

    private calculateRecencyBonus(fused: FusedResult): number {
        const result = this.getBestResult(fused)
        const payload = result.payload

        if (!payload.lastModified) {
            return 0
        }

        const now = Date.now()
        const age = now - new Date(payload.lastModified).getTime()
        const maxAge = 30 * 24 * 60 * 60 * 1000 // 30天

        // 新文件有加分，越新加分越多
        return Math.max(0, 1 - (age / maxAge))
    }

    private calculatePopularityBonus(fused: FusedResult): number {
        const result = this.getBestResult(fused)
        const payload = result.payload

        // 基于文件的引用次数、大小等计算流行度
        let popularity = 0

        // 较大的文件可能更重要
        if (payload.fileSize > 1000) {
            popularity += 0.1
        }

        // 有较多导入/导出的文件可能更核心
        if (payload.imports > 5 || payload.exports > 3) {
            popularity += 0.15
        }

        return Math.min(popularity, 0.3)
    }

    private getResultKey(result: SearchResult): string {
        const payload = result.payload
        return `${payload.filePath}:${payload.startLine}-${payload.endLine}`
    }

    private getBestResult(fused: FusedResult): SearchResult {
        return fused.semanticResult || fused.textResult!
    }

    private getFusionType(fused: FusedResult): string {
        if (fused.semanticResult && fused.textResult) {
            return 'hybrid'
        } else if (fused.semanticResult) {
            return 'semantic'
        } else {
            return 'text'
        }
    }
}
```

## 3. 上下文管理系统

### 3.1 搜索上下文构建

```typescript
// src/services/code-index/context/ContextBuilder.ts
export class SearchContextBuilder {
    private contextCache: Map<string, ContextCache> = new Map()
    private maxContextSize: number = 5000 // 5KB 上下文限制

    async buildSearchContext(
        query: string,
        searchResults: SearchResult[],
        options: ContextOptions = {}
    ): Promise<SearchContext> {
        const { includeCode = true, includeMetadata = true, maxFiles = 5 } = options

        // 1. 分析查询意图
        const intent = this.analyzeQueryIntent(query)

        // 2. 选择最相关的结果
        const relevantResults = this.selectRelevantResults(searchResults, intent, maxFiles)

        // 3. 构建文件上下文
        const fileContexts = await this.buildFileContexts(relevantResults, {
            includeCode,
            includeMetadata
        })

        // 4. 构建项目级上下文
        const projectContext = await this.buildProjectContext(relevantResults)

        // 5. 优化上下文大小
        const optimizedContext = this.optimizeContextSize({
            query,
            intent,
            fileContexts,
            projectContext
        })

        return optimizedContext
    }

    private analyzeQueryIntent(query: string): QueryIntent {
        const intent: QueryIntent = {
            type: 'general',
            entities: [],
            keywords: [],
            language: undefined,
            specificity: 0.5
        }

        // 检测编程语言
        const languageKeywords = {
            javascript: ['javascript', 'js', 'node', 'react', 'vue', 'angular'],
            python: ['python', 'py', 'django', 'flask', 'numpy'],
            java: ['java', 'spring', 'maven', 'gradle'],
            // ... 更多语言
        }

        for (const [lang, keywords] of Object.entries(languageKeywords)) {
            if (keywords.some(keyword => query.toLowerCase().includes(keyword))) {
                intent.language = lang
                intent.specificity += 0.2
                break
            }
        }

        // 检测特定模式
        const patterns = {
            function: /function|func|def|method/i,
            class: /class|interface|struct/i,
            variable: /variable|var|let|const|val/i,
            import: /import|require|include/i,
            error: /error|exception|bug|fix/i,
            performance: /performance|optimize|slow|fast/i
        }

        for (const [type, pattern] of Object.entries(patterns)) {
            if (pattern.test(query)) {
                intent.entities.push(type)
                intent.specificity += 0.1
            }
        }

        // 提取关键词
        const words = query.toLowerCase().split(/\s+/)
        intent.keywords = words.filter(word => word.length > 2)

        return intent
    }

    private selectRelevantResults(
        results: SearchResult[],
        intent: QueryIntent,
        maxFiles: number
    ): SearchResult[] {
        // 按相关性排序
        results.sort((a, b) => b.relevance - a.relevance)

        // 按意图过滤和重排序
        const scoredResults = results.map(result => ({
            result,
            score: this.calculateRelevanceScore(result, intent)
        }))

        scoredResults.sort((a, b) => b.score - a.score)

        // 限制文件数量
        const selectedFiles = new Set<string>()
        const finalResults: SearchResult[] = []

        for (const { result } of scoredResults) {
            const filePath = result.payload.filePath

            if (!selectedFiles.has(filePath) && selectedFiles.size < maxFiles) {
                selectedFiles.add(filePath)
                finalResults.push(result)
            }
        }

        return finalResults
    }

    private calculateRelevanceScore(result: SearchResult, intent: QueryIntent): number {
        let score = result.relevance

        // 语言匹配加分
        if (intent.language && result.payload.language === intent.language) {
            score += 0.3
        }

        // 实体匹配加分
        const codeContent = result.payload.codeChunk.toLowerCase()
        for (const entity of intent.entities) {
            if (codeContent.includes(entity)) {
                score += 0.2
            }
        }

        // 关键词匹配加分
        for (const keyword of intent.keywords) {
            if (codeContent.includes(keyword)) {
                score += 0.1
            }
        }

        return Math.min(score, 1.0)
    }

    private async buildFileContexts(
        results: SearchResult[],
        options: BuildFileOptions
    ): Promise<FileContext[]> {
        const fileContexts: FileContext[] = []

        for (const result of results) {
            const fileContext = await this.buildSingleFileContext(result, options)
            if (fileContext) {
                fileContexts.push(fileContext)
            }
        }

        return fileContexts
    }

    private async buildSingleFileContext(
        result: SearchResult,
        options: BuildFileOptions
    ): Promise<FileContext | null> {
        const { filePath, startLine, endLine, language } = result.payload

        try {
            // 获取文件完整内容
            const content = await fs.readFile(filePath, 'utf-8')
            const lines = content.split('\n')

            // 获取上下文范围（当前代码块前后几行）
            const contextStart = Math.max(0, startLine - 5)
            const contextEnd = Math.min(lines.length, endLine + 5)

            const contextLines = lines.slice(contextStart, contextEnd)
            const contextContent = contextLines.join('\n')

            // 提取文件元数据
            const metadata = options.includeMetadata ? await this.extractFileMetadata(filePath) : {}

            return {
                filePath,
                language,
                mainCode: result.payload.codeChunk,
                contextCode: contextContent,
                startLine: contextStart + 1,
                endLine: contextEnd,
                metadata
            }
        } catch (error) {
            console.warn(`Failed to build context for ${filePath}:`, error)
            return null
        }
    }

    private async buildProjectContext(results: SearchResult[]): Promise<ProjectContext> {
        const filePaths = results.map(r => r.payload.filePath)
        const projectRoot = this.detectProjectRoot(filePaths)

        return {
            projectRoot,
            structure: await this.analyzeProjectStructure(projectRoot),
            dependencies: await this.extractDependencies(projectRoot),
            configuration: await this.loadProjectConfig(projectRoot)
        }
    }

    private optimizeContextSize(context: RawSearchContext): SearchContext {
        let totalSize = JSON.stringify(context).length

        if (totalSize <= this.maxContextSize) {
            return context as SearchContext
        }

        // 优先级缩减策略
        let optimizedContext = { ...context }

        // 1. 减少文件上下文
        while (totalSize > this.maxContextSize && optimizedContext.fileContexts.length > 1) {
            optimizedContext.fileContexts.pop()
            totalSize = JSON.stringify(optimizedContext).length
        }

        // 2. 减少每个文件的上下文
        if (totalSize > this.maxContextSize) {
            optimizedContext.fileContexts = optimizedContext.fileContexts.map(ctx => ({
                ...ctx,
                contextCode: this.truncateCode(ctx.contextCode, 1000), // 1KB 限制
                mainCode: this.truncateCode(ctx.mainCode, 500) // 500B 限制
            }))
            totalSize = JSON.stringify(optimizedContext).length
        }

        // 3. 简化项目上下文
        if (totalSize > this.maxContextSize) {
            optimizedContext.projectContext = {
                ...optimizedContext.projectContext,
                structure: this.simplifyStructure(optimizedContext.projectContext.structure)
            }
        }

        return optimizedContext
    }

    private truncateCode(code: string, maxLength: number): string {
        if (code.length <= maxLength) {
            return code
        }

        const lines = code.split('\n')
        const maxLines = Math.floor(maxLength / 50) // 假设平均每行50字符

        if (lines.length <= maxLines) {
            return code.substring(0, maxLength)
        }

        return lines.slice(0, maxLines).join('\n') + '\n... [truncated]'
    }
}
```

## 4. 性能优化与缓存

### 4.1 多级缓存系统

```typescript
// src/services/code-index/cache/CacheManager.ts
export class CacheManager {
    private memoryCache: Map<string, CacheEntry> = new Map()
    private diskCache: DiskCache
    private config: CacheConfig

    constructor(config: CacheConfig) {
        this.config = config
        this.diskCache = new DiskCache(config.diskCachePath)

        // 定期清理缓存
        setInterval(() => this.cleanup(), this.config.cleanupInterval)
    }

    async get<T>(key: string): Promise<T | null> {
        // 1. 检查内存缓存
        const memoryEntry = this.memoryCache.get(key)
        if (memoryEntry && !this.isExpired(memoryEntry)) {
            memoryEntry.accessCount++
            return memoryEntry.data
        }

        // 2. 检查磁盘缓存
        const diskEntry = await this.diskCache.get(key)
        if (diskEntry && !this.isExpired(diskEntry)) {
            // 提升到内存缓存
            this.memoryCache.set(key, {
                ...diskEntry,
                accessCount: 1
            })
            return diskEntry.data
        }

        return null
    }

    async set<T>(key: string, data: T, ttl: number = this.config.defaultTTL): Promise<void> {
        const entry: CacheEntry = {
            data,
            timestamp: Date.now(),
            ttl,
            accessCount: 1,
            size: this.calculateSize(data)
        }

        // 检查内存缓存大小
        if (this.getMemoryCacheSize() + entry.size > this.config.maxMemorySize) {
            this.evictFromMemory()
        }

        // 存储到内存缓存
        this.memoryCache.set(key, entry)

        // 异步存储到磁盘缓存
        this.diskCache.set(key, entry).catch(error => {
            console.warn('Failed to write to disk cache:', error)
        })
    }

    private async cleanup(): Promise<void> {
        // 清理内存缓存
        this.cleanupMemoryCache()

        // 清理磁盘缓存
        await this.cleanupDiskCache()
    }

    private cleanupMemoryCache(): void {
        const now = Date.now()
        const toDelete: string[] = []

        for (const [key, entry] of this.memoryCache.entries()) {
            if (this.isExpired(entry) || this.shouldEvict(entry)) {
                toDelete.push(key)
            }
        }

        toDelete.forEach(key => this.memoryCache.delete(key))
    }

    private async cleanupDiskCache(): Promise<void> {
        const now = Date.now()
        const expiredKeys = await this.diskCache.findExpiredKeys(now)

        for (const key of expiredKeys) {
            await this.diskCache.delete(key)
        }
    }

    private evictFromMemory(): void {
        // LRU (Least Recently Used) 策略
        let lruKey = ''
        let lruCount = Infinity
        let lruTime = Infinity

        for (const [key, entry] of this.memoryCache.entries()) {
            if (entry.accessCount < lruCount ||
                (entry.accessCount === lruCount && entry.timestamp < lruTime)) {
                lruKey = key
                lruCount = entry.accessCount
                lruTime = entry.timestamp
            }
        }

        if (lruKey) {
            this.memoryCache.delete(lruKey)
        }
    }

    private getMemoryCacheSize(): number {
        let totalSize = 0
        for (const entry of this.memoryCache.values()) {
            totalSize += entry.size
        }
        return totalSize
    }

    private isExpired(entry: CacheEntry): boolean {
        return Date.now() - entry.timestamp > entry.ttl
    }

    private shouldEvict(entry: CacheEntry): boolean {
        // 访问次数很少且占用空间大的条目优先被清理
        const age = Date.now() - entry.timestamp
        const accessRate = entry.accessCount / (age / 1000) // 每秒访问次数

        return accessRate < 0.01 && entry.size > 1024 // 1KB
    }

    private calculateSize(data: any): number {
        return JSON.stringify(data).length * 2 // 粗略估算
    }
}
```

## 5. 集成与工具支持

### 5.1 AI助手工具集成

```typescript
// src/core/tools/CodebaseSearchTool.ts
export class CodebaseSearchTool {
    private searchService: CodeIndexSearchService
    private contextBuilder: SearchContextBuilder

    async execute(
        query: string,
        options: SearchToolOptions = {}
    ): Promise<ToolExecutionResult> {
        try {
            // 1. 执行搜索
            const searchResults = await this.searchService.search(query, {
                directoryPrefix: options.directory,
                maxResults: options.maxResults || 10,
                minScore: options.minScore || 0.4,
                searchType: options.searchType || 'hybrid'
            })

            // 2. 构建搜索上下文
            const context = await this.contextBuilder.buildSearchContext(
                query,
                searchResults,
                {
                    includeCode: true,
                    includeMetadata: options.includeMetadata,
                    maxFiles: options.maxFiles || 5
                }
            )

            // 3. 格式化输出
            const formattedOutput = this.formatOutput(context, searchResults)

            return {
                success: true,
                output: formattedOutput,
                metadata: {
                    resultCount: searchResults.length,
                    contextSize: JSON.stringify(context).length,
                    searchTime: Date.now()
                }
            }
        } catch (error) {
            return {
                success: false,
                error: error.message,
                output: `搜索失败: ${error.message}`
            }
        }
    }

    private formatOutput(context: SearchContext, results: SearchResult[]): string {
        let output = `## 搜索结果: "${context.query}"\n\n`

        // 添加搜索意图分析
        if (context.intent) {
            output += `**搜索意图**: ${context.intent.type}\n`
            if (context.intent.language) {
                output += `**语言**: ${context.intent.language}\n`
            }
            if (context.intent.entities.length > 0) {
                output += `**相关实体**: ${context.intent.entities.join(', ')}\n`
            }
            output += '\n'
        }

        // 添加相关文件
        output += `### 相关文件 (${context.fileContexts.length})\n\n`

        for (const fileContext of context.fileContexts) {
            output += `#### ${path.basename(fileContext.filePath)}\n`
            output += `**路径**: \`${fileContext.filePath}\`\n`
            output += `**语言**: ${fileContext.language}\n`
            output += `**行数**: ${fileContext.startLine}-${fileContext.endLine}\n\n`

            if (fileContext.mainCode) {
                output += '```' + this.getLanguageAlias(fileContext.language) + '\n'
                output += fileContext.mainCode + '\n'
                output += '```\n\n'
            }
        }

        // 添加项目上下文
        if (context.projectContext && context.projectContext.dependencies) {
            output += '### 项目依赖\n\n'
            output += '```\n'
            output += JSON.stringify(context.projectContext.dependencies, null, 2)
            output += '\n```\n\n'
        }

        return output
    }

    private getLanguageAlias(language: string): string {
        const aliases: Record<string, string> = {
            'javascript': 'javascript',
            'typescript': 'typescript',
            'python': 'python',
            'java': 'java',
            'go': 'go',
            'rust': 'rust',
            'c': 'c',
            'cpp': 'cpp',
            'markdown': 'markdown'
        }

        return aliases[language] || 'text'
    }
}
```

## 总结

KiloCode 的代码库检索系统展现了以下技术特点：

### 1. **多模态搜索**
- 结合语义搜索和文本搜索
- 智能评分和结果融合
- 上下文感知的搜索体验

### 2. **智能代码理解**
- 基于 Tree-sitter 的语法分析
- 30+ 编程语言支持
- 语义分块和上下文提取

### 3. **高性能设计**
- 多级缓存系统
- 并发处理和批优化
- 智能增量更新

### 4. **用户体验优化**
- 搜索意图识别
- 结果相关性排序
- 上下文构建和优化

这个系统代表了现代代码检索的最佳实践，为 AI 代码助手提供了强大的代码理解和检索能力，是 KiloCode 核心竞争力的关键技术支撑。