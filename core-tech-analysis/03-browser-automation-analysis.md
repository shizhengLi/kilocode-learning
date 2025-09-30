# KiloCode 浏览器自动化功能深度分析

## 技术架构概览

KiloCode 的浏览器自动化系统采用 Puppeteer Core 作为核心技术栈，提供了完整的网页交互、内容获取和自动化控制能力。系统设计支持本地和远程两种浏览器模式，具有良好的扩展性和稳定性。

### 核心技术栈
- **Puppeteer Core**: v23.4.0 - 主要浏览器自动化框架
- **Puppeteer Chromium Resolver**: v24.0.0 - 自动管理 Chromium 下载
- **Cheerio**: v1.0.0 - HTML 解析和清理
- **Turndown**: v7.2.0 - HTML 到 Markdown 转换

## 1. 浏览器会话管理系统

### 1.1 BrowserSession 核心实现

```typescript
// src/services/browser/BrowserSession.ts
export class BrowserSession {
    private browser: Browser | null = null
    private page: Page | null = null
    private activeTabUrl: string | null = null
    private isRemoteMode: boolean = false
    private remotePort: number | null = null
    private consoleLogs: ConsoleMessage[] = []
    private mousePosition: { x: number; y: number } = { x: 0, y: 0 }

    // 预定义视口尺寸
    static readonly VIEWPORT_PRESETS = {
        desktop: { width: 1280, height: 720 },
        tablet: { width: 768, height: 1024 },
        mobile: { width: 375, height: 812 }
    }

    async launchBrowser(options: BrowserLaunchOptions = {}): Promise<boolean> {
        try {
            const {
                mode = 'local',
                remotePort,
                headless = false,
                viewport = 'desktop',
                screenshotQuality = 80
            } = options

            this.isRemoteMode = mode === 'remote'
            this.remotePort = remotePort

            if (this.isRemoteMode && remotePort) {
                // 远程模式：连接到现有 Chrome 实例
                return await this.launchRemoteBrowser(remotePort, {
                    headless,
                    viewport,
                    screenshotQuality
                })
            } else {
                // 本地模式：启动隔离的 Chromium 实例
                return await this.launchLocalBrowser({
                    headless,
                    viewport,
                    screenshotQuality
                })
            }
        } catch (error) {
            console.error('Failed to launch browser:', error)
            await this.closeBrowser() // 清理资源
            return false
        }
    }

    private async launchLocalBrowser(options: LocalLaunchOptions): Promise<boolean> {
        try {
            // 使用 puppeteer-chromium-resolver 自动下载和管理 Chromium
            const browserFetcher = new BrowserFetcher()
            const revisionInfo = await browserFetcher.download()

            this.browser = await puppeteer.launch({
                executablePath: revisionInfo.executablePath,
                headless: options.headless,
                args: [
                    '--no-sandbox',
                    '--disable-setuid-sandbox',
                    '--disable-dev-shm-usage',
                    '--disable-gpu',
                    '--no-first-run',
                    '--no-default-browser-check',
                    '--disable-default-apps',
                    '--disable-extensions'
                ],
                defaultViewport: BrowserSession.VIEWPORT_PRESETS[options.viewport],
                ignoreDefaultArgs: ['--disable-extensions']
            })

            // 创建新页面
            this.page = await this.browser.newPage()

            // 设置截图质量
            await this.page.setDefaultTimeout(15000) // 15秒默认超时

            return true
        } catch (error) {
            console.error('Local browser launch failed:', error)
            return false
        }
    }

    private async launchRemoteBrowser(port: number, options: RemoteLaunchOptions): Promise<boolean> {
        try {
            // 连接到远程 Chrome 调试端口
            const browserURL = `http://localhost:${port}`
            this.browser = await puppeteer.connect({
                browserURL,
                defaultViewport: BrowserSession.VIEWPORT_PRESETS[options.viewport]
            })

            // 获取或创建新页面
            const pages = await this.browser.pages()
            this.page = pages[pages.length - 1] || await this.browser.newPage()

            return true
        } catch (error) {
            console.error(`Failed to connect to remote browser on port ${port}:`, error)
            return false
        }
    }

    async closeBrowser(): Promise<void> {
        try {
            // 清理页面
            if (this.page) {
                await this.page.close().catch(() => {})
                this.page = null
            }

            // 关闭浏览器
            if (this.browser) {
                if (this.isRemoteMode) {
                    await this.browser.disconnect()
                } else {
                    await this.browser.close()
                }
                this.browser = null
            }

            // 重置状态
            this.activeTabUrl = null
            this.consoleLogs = []
            this.mousePosition = { x: 0, y: 0 }
        } catch (error) {
            console.error('Error closing browser:', error)
        }
    }

    async navigateToUrl(url: string): Promise<boolean> {
        if (!this.page) {
            throw new Error('Browser not launched')
        }

        try {
            // 标准化 URL
            const normalizedUrl = this.normalizeUrl(url)

            // 检查是否可以复用当前标签页
            if (await this.canReuseCurrentTab(normalizedUrl)) {
                return await this.navigateInCurrentTab(normalizedUrl)
            } else {
                // 创建新标签页
                return await this.navigateInNewTab(normalizedUrl)
            }
        } catch (error) {
            console.error('Navigation failed:', error)
            return false
        }
    }

    private async canReuseCurrentTab(url: string): Promise<boolean> {
        if (!this.activeTabUrl) return false

        try {
            const currentDomain = new URL(this.activeTabUrl).hostname
            const targetDomain = new URL(url).hostname

            // 同域名的 URL 可以复用标签页
            return currentDomain === targetDomain
        } catch {
            return false
        }
    }

    private async navigateInCurrentTab(url: string): Promise<boolean> {
        if (!this.page) return false

        // 监听网络活动
        const networkListener = this.setupNetworkListener()

        try {
            await this.page.goto(url, {
                waitUntil: 'networkidle2',
                timeout: 15000
            })

            // 等待页面稳定
            await this.waitTillHTMLStable(this.page)

            this.activeTabUrl = url
            return true
        } catch (error) {
            // 降级到 domcontentloaded
            try {
                await this.page.goto(url, {
                    waitUntil: 'domcontentloaded',
                    timeout: 10000
                })
                this.activeTabUrl = url
                return true
            } catch (fallbackError) {
                console.error('Navigation fallback failed:', fallbackError)
                return false
            }
        } finally {
            networkListener.dispose()
        }
    }

    private async navigateInNewTab(url: string): Promise<boolean> {
        if (!this.browser) return false

        try {
            // 创建新标签页
            this.page = await this.browser.newPage()

            // 导航到 URL
            return await this.navigateInCurrentTab(url)
        } catch (error) {
            console.error('New tab navigation failed:', error)
            return false
        }
    }

    private setupNetworkListener(): NetworkListener {
        const requests: Set<string> = new Set()
        let hasNetworkActivity = false

        const requestListener = (request: Request) => {
            requests.add(request.url())
            hasNetworkActivity = true
        }

        const responseListener = (response: Response) => {
            requests.delete(response.url())
        }

        this.page?.on('request', requestListener)
        this.page?.on('response', responseListener)

        return {
            dispose: () => {
                this.page?.off('request', requestListener)
                this.page?.off('response', responseListener)
            },
            hasActivity: () => hasNetworkActivity,
            getActiveRequests: () => requests.size
        }
    }

    private async waitTillHTMLStable(page: Page, timeout = 5000): Promise<void> {
        const checkDurationMsecs = 500
        const maxChecks = timeout / checkDurationMsecs
        let lastHTMLSize = 0
        let countStableSizeIterations = 0
        const minStableSizeIterations = 3
        let checks = 0

        while (checks < maxChecks) {
            checks++
            try {
                const html = await page.content()
                const currentHTMLSize = html.length

                if (currentHTMLSize === lastHTMLSize) {
                    countStableSizeIterations++
                } else {
                    countStableSizeIterations = 0
                }

                if (countStableSizeIterations >= minStableSizeIterations) {
                    return // HTML 大小稳定，页面加载完成
                }

                lastHTMLSize = currentHTMLSize
                await new Promise(resolve => setTimeout(resolve, checkDurationMsecs))
            } catch (error) {
                console.warn('HTML stability check failed:', error)
                return
            }
        }
    }
}
```

### 1.2 智能页面交互系统

```typescript
// src/services/browser/BrowserSession.ts (continued)
export class BrowserSession {
    // 鼠标交互处理
    async handleMouseInteraction(action: MouseAction, coordinate: Coordinate): Promise<boolean> {
        if (!this.page) {
            throw new Error('Browser not launched')
        }

        try {
            // 更新鼠标位置
            this.mousePosition = { x: coordinate.x, y: coordinate.y }

            switch (action) {
                case 'click':
                    return await this.performClick(coordinate)
                case 'hover':
                    return await this.performHover(coordinate)
                case 'double_click':
                    return await this.performDoubleClick(coordinate)
                case 'right_click':
                    return await this.performRightClick(coordinate)
                default:
                    return false
            }
        } catch (error) {
            console.error(`Mouse ${action} failed:`, error)
            return false
        }
    }

    private async performClick(coordinate: Coordinate): Promise<boolean> {
        if (!this.page) return false

        // 设置网络监听器
        const networkListener = this.setupNetworkListener()

        try {
            // 执行点击
            await this.page.mouse.click(coordinate.x, coordinate.y, {
                delay: 100 // 点击延迟
            })

            // 等待可能的导航
            await this.waitForNavigationOrTimeout(networkListener)

            return true
        } catch (error) {
            console.error('Click failed:', error)
            return false
        } finally {
            networkListener.dispose()
        }
    }

    private async performHover(coordinate: Coordinate): Promise<boolean> {
        if (!this.page) return false

        try {
            await this.page.mouse.move(coordinate.x, coordinate.y)

            // 等待悬停效果
            await new Promise(resolve => setTimeout(resolve, 500))

            return true
        } catch (error) {
            console.error('Hover failed:', error)
            return false
        }
    }

    private async performDoubleClick(coordinate: Coordinate): Promise<boolean> {
        if (!this.page) return false

        try {
            await this.page.mouse.click(coordinate.x, coordinate.y, {
                clickCount: 2,
                delay: 100
            })

            return true
        } catch (error) {
            console.error('Double click failed:', error)
            return false
        }
    }

    private async performRightClick(coordinate: Coordinate): Promise<boolean> {
        if (!this.page) return false

        try {
            await this.page.mouse.click(coordinate.x, coordinate.y, {
                button: 'right',
                delay: 100
            })

            return true
        } catch (error) {
            console.error('Right click failed:', error)
            return false
        }
    }

    // 键盘输入
    async typeText(text: string): Promise<boolean> {
        if (!this.page) return false

        try {
            // 分页输入以处理长文本
            const chunks = this.splitTextIntoChunks(text, 1000)

            for (const chunk of chunks) {
                await this.page.keyboard.type(chunk, {
                    delay: 50 // 打字延迟
                })

                // 短暂停顿
                await new Promise(resolve => setTimeout(resolve, 100))
            }

            return true
        } catch (error) {
            console.error('Type failed:', error)
            return false
        }
    }

    private splitTextIntoChunks(text: string, maxLength: number): string[] {
        const chunks: string[] = []

        for (let i = 0; i < text.length; i += maxLength) {
            chunks.push(text.substring(i, i + maxLength))
        }

        return chunks
    }

    // 页面滚动
    async scrollPage(direction: 'up' | 'down', amount: number = 300): Promise<boolean> {
        if (!this.page) return false

        try {
            await this.page.evaluate(
                (direction, amount) => {
                    window.scrollBy({
                        top: direction === 'down' ? amount : -amount,
                        behavior: 'smooth'
                    })
                },
                direction,
                amount
            )

            // 等待滚动完成
            await new Promise(resolve => setTimeout(resolve, 500))

            return true
        } catch (error) {
            console.error('Scroll failed:', error)
            return false
        }
    }

    // 页面截图
    async takeScreenshot(options: ScreenshotOptions = {}): Promise<string> {
        if (!this.page) {
            throw new Error('Browser not launched')
        }

        try {
            const {
                format = 'webp',
                quality = 80,
                fullPage = false,
                captureMouse = true
            } = options

            // 如果需要显示鼠标位置，先添加鼠标指示器
            if (captureMouse) {
                await this.addMouseIndicator()
            }

            const screenshotOptions: ScreenshotOptions = {
                type: format,
                quality: quality,
                fullPage: fullPage,
                encoding: 'base64'
            }

            let screenshot = await this.page.screenshot(screenshotOptions)

            // 如果 WebP 格式失败，降级到 PNG
            if (format === 'webp' && typeof screenshot === 'string') {
                try {
                    // 验证 WebP 格式
                    const buffer = Buffer.from(screenshot, 'base64')
                    if (!this.isValidWebP(buffer)) {
                        throw new Error('Invalid WebP format')
                    }
                } catch {
                    // 降级到 PNG
                    screenshot = await this.page.screenshot({
                        ...screenshotOptions,
                        type: 'png'
                    })
                }
            }

            return `data:${format === 'png' ? 'image/png' : 'image/webp'};base64,${screenshot}`
        } catch (error) {
            console.error('Screenshot failed:', error)
            throw error
        }
    }

    private async addMouseIndicator(): Promise<void> {
        if (!this.page) return

        try {
            await this.page.evaluate(
                (x, y) => {
                    // 移除之前的指示器
                    const existing = document.getElementById('kilocode-mouse-indicator')
                    if (existing) {
                        existing.remove()
                    }

                    // 创建新的鼠标指示器
                    const indicator = document.createElement('div')
                    indicator.id = 'kilocode-mouse-indicator'
                    indicator.style.cssText = `
                        position: fixed;
                        left: ${x - 10}px;
                        top: ${y - 10}px;
                        width: 20px;
                        height: 20px;
                        border: 2px solid red;
                        border-radius: 50%;
                        pointer-events: none;
                        z-index: 999999;
                        background: rgba(255, 0, 0, 0.2);
                    `
                    document.body.appendChild(indicator)

                    // 3秒后自动移除
                    setTimeout(() => indicator.remove(), 3000)
                },
                this.mousePosition.x,
                this.mousePosition.y
            )
        } catch (error) {
            console.warn('Failed to add mouse indicator:', error)
        }
    }

    private isValidWebP(buffer: Buffer): boolean {
        // 简单的 WebP 格式验证
        return buffer.length > 12 &&
               buffer.toString('ascii', 0, 4) === 'RIFF' &&
               buffer.toString('ascii', 8, 12) === 'WEBP'
    }

    private async waitForNavigationOrTimeout(networkListener: NetworkListener): Promise<void> {
        const timeout = 10000 // 10秒超时
        const startTime = Date.now()

        return new Promise((resolve) => {
            const checkInterval = setInterval(() => {
                const elapsed = Date.now() - startTime

                // 检查是否有网络活动
                if (networkListener.hasActivity()) {
                    // 有网络活动，等待网络空闲
                    if (networkListener.getActiveRequests() === 0) {
                        clearInterval(checkInterval)
                        resolve()
                    }
                } else {
                    // 无网络活动，短时间后继续
                    if (elapsed > 2000) {
                        clearInterval(checkInterval)
                        resolve()
                    }
                }

                // 超时处理
                if (elapsed > timeout) {
                    clearInterval(checkInterval)
                    resolve()
                }
            }, 100)
        })
    }
}
```

## 2. URL 内容获取系统

### 2.1 UrlContentFetcher 实现

```typescript
// src/services/browser/UrlContentFetcher.ts
export class UrlContentFetcher {
    private browser: Browser | null = null
    private page: Page | null = null
    private retryCount: number = 3
    private timeout: number = 30000

    async fetchContent(url: string, options: FetchOptions = {}): Promise<FetchResult> {
        const {
            cleanHtml = true,
            convertToMarkdown = true,
            includeMetadata = true,
            timeout = this.timeout
        } = options

        let lastError: Error | null = null

        // 重试机制
        for (let attempt = 1; attempt <= this.retryCount; attempt++) {
            try {
                return await this.attemptFetch(url, {
                    cleanHtml,
                    convertToMarkdown,
                    includeMetadata,
                    timeout
                })
            } catch (error) {
                lastError = error as Error
                console.warn(`Fetch attempt ${attempt} failed for ${url}:`, error)

                if (attempt < this.retryCount) {
                    // 指数退避
                    const delay = Math.pow(2, attempt) * 1000
                    await new Promise(resolve => setTimeout(resolve, delay))
                }
            }
        }

        // 所有重试都失败
        return {
            success: false,
            url,
            error: lastError?.message || 'Unknown error',
            timestamp: new Date()
        }
    }

    private async attemptFetch(url: string, options: FetchOptions): Promise<FetchResult> {
        const startTime = Date.now()

        try {
            // 启动浏览器
            if (!this.browser) {
                this.browser = await puppeteer.launch({
                    headless: true,
                    args: ['--no-sandbox', '--disable-setuid-sandbox']
                })
            }

            // 创建新页面
            this.page = await this.browser.newPage()

            // 设置视口和超时
            await this.page.setViewport({ width: 1920, height: 1080 })
            await this.page.setDefaultTimeout(options.timeout)

            // 配置请求拦截
            await this.setupRequestInterception()

            // 导航到 URL
            const loadResult = await this.navigateToPage(url, options.timeout)

            if (!loadResult.success) {
                throw new Error(loadResult.error || 'Failed to load page')
            }

            // 获取页面内容
            const content = await this.extractPageContent(options)

            const duration = Date.now() - startTime

            return {
                success: true,
                url,
                content: content.content,
                metadata: {
                    ...content.metadata,
                    loadTime: loadResult.loadTime,
                    fetchDuration: duration,
                    fetchTime: new Date()
                },
                timestamp: new Date()
            }
        } catch (error) {
            throw error
        }
    }

    private async navigateToPage(url: string, timeout: number): Promise<LoadResult> {
        const startTime = Date.now()

        try {
            // 尝试完整的页面加载
            await this.page!.goto(url, {
                waitUntil: 'networkidle2',
                timeout: timeout
            })

            // 等待页面稳定
            await this.waitTillHTMLStable(this.page!)

            const loadTime = Date.now() - startTime

            return {
                success: true,
                loadTime
            }
        } catch (error) {
            console.warn('Full page load failed, trying domcontentloaded:', error)

            try {
                // 降级到 domcontentloaded
                await this.page!.goto(url, {
                    waitUntil: 'domcontentloaded',
                    timeout: Math.min(timeout, 10000)
                })

                const loadTime = Date.now() - startTime

                return {
                    success: true,
                    loadTime,
                    warning: 'Used domcontentloaded instead of networkidle2'
                }
            } catch (fallbackError) {
                return {
                    success: false,
                    error: fallbackError.message
                }
            }
        }
    }

    private async extractPageContent(options: FetchOptions): Promise<ContentResult> {
        if (!this.page) {
            throw new Error('Page not available')
        }

        // 获取页面 HTML
        const html = await this.page.content()
        const title = await this.page.title()

        // 清理 HTML
        let cleanedHtml = html
        if (options.cleanHtml) {
            cleanedHtml = this.cleanHtmlContent(html)
        }

        // 转换为 Markdown
        let content = cleanedHtml
        let format: 'html' | 'markdown' = 'html'

        if (options.convertToMarkdown) {
            const markdownResult = await this.convertToMarkdown(cleanedHtml)
            content = markdownResult.content
            format = 'markdown'
        }

        // 提取元数据
        const metadata = options.includeMetadata ? await this.extractMetadata() : {}

        return {
            content,
            format,
            metadata: {
                title,
                ...metadata
            }
        }
    }

    private cleanHtmlContent(html: string): string {
        const $ = cheerio.load(html)

        // 移除不需要的元素
        $('script, style, noscript, iframe, object, embed').remove()
        $('[style*="display:none"], [hidden]').remove()
        $('.ads, .advertisement, .sidebar, .footer, .header').remove()

        // 清理属性
        $('*').each((i, element) => {
            const attrs = $(element).attr()
            const keepAttrs = ['href', 'src', 'alt', 'title', 'class']

            Object.keys(attrs).forEach(attr => {
                if (!keepAttrs.includes(attr)) {
                    $(element).removeAttr(attr)
                }
            })
        })

        return $.html()
    }

    private async convertToMarkdown(html: string): Promise<{ content: string; warnings: string[] }> {
        try {
            const turndownService = new TurndownService({
                headingStyle: 'atx',
                codeBlockStyle: 'fenced',
                bulletListMarker: '-',
                emDelimiter: '*'
            })

            // 自定义规则
            turndownService.addRule('strikethrough', {
                filter: ['del', 's', 'strike'],
                replacement: (content) => `~~${content}~~`
            })

            turndownService.addRule('table', {
                filter: 'table',
                replacement: (content, node) => {
                    const table = node as cheerio.Cheerio
                    const rows = table.find('tr')

                    let markdown = '\n\n'

                    rows.each((_, row) => {
                        const cells = $(row).find('td, th')
                        const rowData = cells.map((_, cell) => $(cell).text().trim()).get()
                        markdown += `| ${rowData.join(' | ')} |\n`
                    })

                    return markdown + '\n'
                }
            })

            const markdown = turndownService.turndown(html)
            const warnings = []

            // 检查转换质量
            if (html.includes('<table>') && !markdown.includes('|')) {
                warnings.push('Table conversion may be incomplete')
            }

            if (html.includes('<code>') && !markdown.includes('`')) {
                warnings.push('Code block conversion may be incomplete')
            }

            return {
                content: markdown.trim(),
                warnings
            }
        } catch (error) {
            console.warn('Markdown conversion failed:', error)
            return {
                content: html, // 降级到 HTML
                warnings: ['Markdown conversion failed, returning HTML']
            }
        }
    }

    private async extractMetadata(): Promise<PageMetadata> {
        if (!this.page) {
            return {}
        }

        try {
            return await this.page.evaluate(() => {
                const metadata: PageMetadata = {}

                // 提取 meta 标签
                const metaTags = document.querySelectorAll('meta')
                metaTags.forEach(tag => {
                    const name = tag.getAttribute('name') || tag.getAttribute('property')
                    const content = tag.getAttribute('content')

                    if (name && content) {
                        metadata[name] = content
                    }
                })

                // 提取结构化数据
                const jsonLd = document.querySelector('script[type="application/ld+json"]')
                if (jsonLd) {
                    try {
                        metadata.structuredData = JSON.parse(jsonLd.textContent || '{}')
                    } catch {
                        // 忽略 JSON 解析错误
                    }
                }

                // 页面信息
                metadata.wordCount = document.body?.textContent?.split(/\s+/).length || 0
                metadata.linkCount = document.querySelectorAll('a[href]').length
                metadata.imageCount = document.querySelectorAll('img').length

                return metadata
            })
        } catch (error) {
            console.warn('Metadata extraction failed:', error)
            return {}
        }
    }

    private async setupRequestInterception(): Promise<void> {
        if (!this.page) return

        await this.page.setRequestInterception(true)

        this.page.on('request', (request) => {
            const resourceType = request.resourceType()

            // 阻止不必要的资源
            if (['image', 'stylesheet', 'font', 'media'].includes(resourceType)) {
                request.abort()
            } else {
                request.continue()
            }
        })
    }

    private async waitTillHTMLStable(page: Page, timeout = 5000): Promise<void> {
        // 实现与 BrowserSession 中相同的稳定检测算法
        const checkDurationMsecs = 500
        const maxChecks = timeout / checkDurationMsecs
        let lastHTMLSize = 0
        let countStableSizeIterations = 0
        const minStableSizeIterations = 3
        let checks = 0

        while (checks < maxChecks) {
            checks++
            try {
                const html = await page.content()
                const currentHTMLSize = html.length

                if (currentHTMLSize === lastHTMLSize) {
                    countStableSizeIterations++
                } else {
                    countStableSizeIterations = 0
                }

                if (countStableSizeIterations >= minStableSizeIterations) {
                    return
                }

                lastHTMLSize = currentHTMLSize
                await new Promise(resolve => setTimeout(resolve, checkDurationMsecs))
            } catch (error) {
                console.warn('HTML stability check failed:', error)
                return
            }
        }
    }

    async close(): Promise<void> {
        try {
            if (this.page) {
                await this.page.close()
                this.page = null
            }

            if (this.browser) {
                await this.browser.close()
                this.browser = null
            }
        } catch (error) {
            console.warn('Error closing UrlContentFetcher:', error)
        }
    }
}
```

## 3. 远程浏览器发现系统

### 3.1 浏览器发现和连接管理

```typescript
// src/services/browser/browserDiscovery.ts
export class BrowserDiscovery {
    private discoveredBrowsers: Map<number, BrowserInfo> = new Map()
    private scanInterval: NodeJS.Timeout | null = null
    private isScanning: boolean = false

    async discoverAvailableBrowsers(): Promise<BrowserInfo[]> {
        const browsers: BrowserInfo[] = []

        // 1. 扫描本地 Chrome 调试端口
        const localPorts = await this.scanLocalPorts()
        browsers.push(...localPorts)

        // 2. 扫描网络中的 Chrome 实例（可选）
        const networkBrowsers = await this.scanNetworkBrowsers()
        browsers.push(...networkBrowsers)

        // 3. 检查 Docker 环境
        const dockerBrowsers = await this.checkDockerBrowsers()
        browsers.push(...dockerBrowsers)

        // 缓存结果
        browsers.forEach(browser => {
            this.discoveredBrowsers.set(browser.port, browser)
        })

        return browsers.sort((a, b) => a.port - b.port)
    }

    private async scanLocalPorts(): Promise<BrowserInfo[]> {
        const commonPorts = [9222, 9223, 9229, 9333, 9444]
        const browsers: BrowserInfo[] = []

        // 并发检查端口
        const checks = commonPorts.map(port => this.checkBrowserPort(port))
        const results = await Promise.allSettled(checks)

        results.forEach((result, index) => {
            if (result.status === 'fulfilled' && result.value) {
                browsers.push(result.value!)
            }
        })

        return browsers
    }

    private async checkBrowserPort(port: number): Promise<BrowserInfo | null> {
        try {
            const response = await axios.get(`http://localhost:${port}/json/version`, {
                timeout: 2000
            })

            const data = response.data

            return {
                port,
                type: 'chrome',
                version: data['Browser'],
                protocolVersion: data['Protocol-Version'],
                userAgent: data['User-Agent'],
                webSocketDebuggerUrl: data['webSocketDebuggerUrl'],
                address: `http://localhost:${port}`,
                lastSeen: new Date(),
                status: 'available'
            }
        } catch (error) {
            return null
        }
    }

    private async scanNetworkBrowsers(): Promise<BrowserInfo[]> {
        // 扫描本地网络中的 Chrome 实例
        const browsers: BrowserInfo[] = []
        const networkRange = '192.168.1' // 可配置

        // 并发扫描常见的调试端口
        const portsToScan = [9222, 9223, 9229]
        const scanPromises: Promise<BrowserInfo[]>[] = []

        for (let i = 1; i <= 254; i++) {
            const host = `${networkRange}.${i}`

            for (const port of portsToScan) {
                scanPromises.push(this.checkNetworkBrowser(host, port))
            }
        }

        const results = await Promise.allSettled(scanPromises)

        results.forEach(result => {
            if (result.status === 'fulfilled' && result.value.length > 0) {
                browsers.push(...result.value)
            }
        })

        return browsers
    }

    private async checkNetworkBrowser(host: string, port: number): Promise<BrowserInfo[]> {
        try {
            const response = await axios.get(`http://${host}:${port}/json/version`, {
                timeout: 1000
            })

            const data = response.data

            return [{
                port,
                type: 'chrome',
                host,
                version: data['Browser'],
                protocolVersion: data['Protocol-Version'],
                userAgent: data['User-Agent'],
                webSocketDebuggerUrl: data['webSocketDebuggerUrl'],
                address: `http://${host}:${port}`,
                lastSeen: new Date(),
                status: 'available'
            }]
        } catch (error) {
            return []
        }
    }

    private async checkDockerBrowsers(): Promise<BrowserInfo[]> {
        // 检查 Docker 容器中的 Chrome
        const browsers: BrowserInfo[] = []

        try {
            // 检查 Docker 是否可用
            const { execSync } = require('child_process')

            // 查找运行中的 Chrome 容器
            const command = `docker ps --format "table {{.ID}}\\t{{.Image}}\\t{{.Ports}}" | grep chrome`
            const output = execSync(command, { encoding: 'utf8' })

            if (output) {
                const lines = output.split('\n').filter(line => line.trim())

                for (const line of lines) {
                    const parts = line.split('\t')
                    if (parts.length >= 3) {
                        const containerId = parts[0].trim()
                        const ports = parts[2].trim()

                        // 解析端口映射
                        const portMatches = ports.match(/:(\d+)->9222\/tcp/)
                        if (portMatches) {
                            const hostPort = parseInt(portMatches[1])
                            const browser = await this.checkBrowserPort(hostPort)

                            if (browser) {
                                browser.containerId = containerId
                                browser.type = 'docker-chrome'
                                browsers.push(browser)
                            }
                        }
                    }
                }
            }
        } catch (error) {
            // Docker 不可用，忽略
        }

        return browsers
    }

    async testBrowserConnection(browserInfo: BrowserInfo): Promise<ConnectionTestResult> {
        try {
            const startTime = Date.now()

            // 尝试连接
            const response = await axios.get(`${browserInfo.address}/json/version`, {
                timeout: 5000
            })

            const responseTime = Date.now() - startTime

            // 尝试获取页面列表
            const pagesResponse = await axios.get(`${browserInfo.address}/json`, {
                timeout: 3000
            })

            const pages = pagesResponse.data

            return {
                success: true,
                responseTime,
                availablePages: pages.length,
                browserInfo: {
                    ...browserInfo,
                    lastTested: new Date(),
                    status: 'connected'
                }
            }
        } catch (error) {
            return {
                success: false,
                error: error.message,
                browserInfo: {
                    ...browserInfo,
                    lastTested: new Date(),
                    status: 'error'
                }
            }
        }
    }

    startAutoDiscovery(intervalMs: number = 30000): void {
        if (this.scanInterval) {
            clearInterval(this.scanInterval)
        }

        this.scanInterval = setInterval(async () => {
            if (!this.isScanning) {
                this.isScanning = true
                try {
                    await this.discoverAvailableBrowsers()
                } catch (error) {
                    console.warn('Auto-discovery scan failed:', error)
                } finally {
                    this.isScanning = false
                }
            }
        }, intervalMs)
    }

    stopAutoDiscovery(): void {
        if (this.scanInterval) {
            clearInterval(this.scanInterval)
            this.scanInterval = null
        }
    }

    getAvailableBrowsers(): BrowserInfo[] {
        return Array.from(this.discoveredBrowsers.values())
            .filter(browser => browser.status === 'available')
            .sort((a, b) => (b.lastSeen?.getTime() || 0) - (a.lastSeen?.getTime() || 0))
    }
}
```

## 4. AI 助手集成系统

### 4.1 浏览器动作工具实现

```typescript
// src/core/tools/browserActionTool.ts
export async function browserActionTool(
    cline: Cline,
    block: ToolUse,
    addMessage: (message: Message) => Promise<void>,
    updateLastMessage: (message: Message) => Promise<void>
): Promise<boolean> {
    const { action, url, coordinate, text, browser_settings } = block.params

    try {
        // 验证参数
        if (!action || !url) {
            throw new Error('Missing required parameters: action and url are required')
        }

        // 获取浏览器会话
        const browserSession = BrowserSession.getInstance(cline.context)

        // 根据动作类型执行相应操作
        let result: BrowserActionResult

        switch (action) {
            case 'launch':
                result = await handleLaunchAction(browserSession, browser_settings)
                break
            case 'navigate':
                result = await handleNavigateAction(browserSession, url)
                break
            case 'click':
                result = await handleClickAction(browserSession, coordinate)
                break
            case 'type':
                result = await handleTypeAction(browserSession, text)
                break
            case 'hover':
                result = await handleHoverAction(browserSession, coordinate)
                break
            case 'scroll':
                result = await handleScrollAction(browserSession, coordinate)
                break
            case 'screenshot':
                result = await handleScreenshotAction(browserSession)
                break
            case 'close':
                result = await handleCloseAction(browserSession)
                break
            default:
                throw new Error(`Unknown browser action: ${action}`)
        }

        // 格式化输出
        const output = formatBrowserActionResult(result)

        // 添加消息
        await addMessage({
            role: 'tool',
            content: output,
            _type: 'browser_action_result',
            _source: 'browser_action',
            ts: Date.now()
        })

        return true
    } catch (error) {
        console.error('Browser action tool failed:', error)

        await addMessage({
            role: 'tool',
            content: `Browser action failed: ${error.message}`,
            _type: 'browser_action_error',
            _source: 'browser_action',
            ts: Date.now()
        })

        return false
    }
}

// 动作处理器函数
async function handleLaunchAction(
    browserSession: BrowserSession,
    settings?: BrowserSettings
): Promise<BrowserActionResult> {
    const success = await browserSession.launchBrowser(settings || {})

    return {
        action: 'launch',
        success,
        message: success ? 'Browser launched successfully' : 'Failed to launch browser',
        timestamp: new Date()
    }
}

async function handleNavigateAction(
    browserSession: BrowserSession,
    url: string
): Promise<BrowserActionResult> {
    const success = await browserSession.navigateToUrl(url)

    return {
        action: 'navigate',
        success,
        url,
        message: success ? `Navigated to ${url}` : `Failed to navigate to ${url}`,
        timestamp: new Date()
    }
}

async function handleClickAction(
    browserSession: BrowserSession,
    coordinate?: Coordinate
): Promise<BrowserActionResult> {
    if (!coordinate) {
        throw new Error('Coordinate is required for click action')
    }

    const success = await browserSession.handleMouseInteraction('click', coordinate)

    return {
        action: 'click',
        success,
        coordinate,
        message: success ? `Clicked at (${coordinate.x}, ${coordinate.y})` : 'Failed to click',
        timestamp: new Date()
    }
}

async function handleTypeAction(
    browserSession: BrowserSession,
    text?: string
): Promise<BrowserActionResult> {
    if (!text) {
        throw new Error('Text is required for type action')
    }

    const success = await browserSession.typeText(text)

    return {
        action: 'type',
        success,
        message: success ? 'Text typed successfully' : 'Failed to type text',
        timestamp: new Date()
    }
}

async function handleScreenshotAction(
    browserSession: BrowserSession
): Promise<BrowserActionResult> {
    try {
        const screenshot = await browserSession.takeScreenshot()

        return {
            action: 'screenshot',
            success: true,
            screenshot,
            message: 'Screenshot captured successfully',
            timestamp: new Date()
        }
    } catch (error) {
        return {
            action: 'screenshot',
            success: false,
            message: `Failed to capture screenshot: ${error.message}`,
            timestamp: new Date()
        }
    }
}

async function handleCloseAction(
    browserSession: BrowserSession
): Promise<BrowserActionResult> {
    await browserSession.closeBrowser()

    return {
        action: 'close',
        success: true,
        message: 'Browser closed successfully',
        timestamp: new Date()
    }
}

// 结果格式化函数
function formatBrowserActionResult(result: BrowserActionResult): string {
    let output = `## Browser Action: ${result.action}\n\n`

    output += `**Status**: ${result.success ? '✅ Success' : '❌ Failed'}\n`
    output += `**Time**: ${result.timestamp.toLocaleTimeString()}\n\n`

    if (result.message) {
        output += `**Message**: ${result.message}\n\n`
    }

    if (result.url) {
        output += `**URL**: ${result.url}\n\n`
    }

    if (result.coordinate) {
        output += `**Coordinate**: (${result.coordinate.x}, ${result.coordinate.y})\n\n`
    }

    if (result.screenshot) {
        output += `**Screenshot**:\n\n![Browser Screenshot](${result.screenshot})\n\n`
    }

    if (result.consoleLogs && result.consoleLogs.length > 0) {
        output += `**Console Logs**:\n\n\`\`\`\n`
        result.consoleLogs.forEach(log => {
            output += `[${log.type}] ${log.text}\n`
        })
        output += `\`\`\`\n\n`
    }

    return output
}
```

## 5. 错误处理和恢复机制

### 5.1 综合错误处理系统

```typescript
// src/services/browser/errorHandling/BrowserErrorHandler.ts
export class BrowserErrorHandler {
    private retryStrategies: Map<string, RetryStrategy> = new Map()
    private errorLog: ErrorLog[] = []
    private maxLogSize: number = 100

    constructor() {
        this.initializeRetryStrategies()
    }

    private initializeRetryStrategies(): void {
        // 网络错误重试策略
        this.retryStrategies.set('network', {
            maxAttempts: 3,
            delay: (attempt) => Math.pow(2, attempt) * 1000, // 指数退避
            condition: (error) => this.isNetworkError(error)
        })

        // 页面加载重试策略
        this.retryStrategies.set('navigation', {
            maxAttempts: 2,
            delay: (attempt) => 2000,
            condition: (error) => this.isNavigationError(error)
        })

        // 元素操作重试策略
        this.retryStrategies.set('interaction', {
            maxAttempts: 3,
            delay: (attempt) => 1000,
            condition: (error) => this.isInteractionError(error)
        })
    }

    async handleWithRetry<T>(
        operation: () => Promise<T>,
        strategyName: string,
        context: ErrorContext
    ): Promise<T> {
        const strategy = this.retryStrategies.get(strategyName)
        if (!strategy) {
            throw new Error(`Unknown retry strategy: ${strategyName}`)
        }

        let lastError: Error

        for (let attempt = 1; attempt <= strategy.maxAttempts; attempt++) {
            try {
                const result = await operation()
                this.logSuccess(context, attempt)
                return result
            } catch (error) {
                lastError = error as Error

                if (!strategy.condition(error) || attempt === strategy.maxAttempts) {
                    break
                }

                const delay = strategy.delay(attempt)
                this.logRetry(context, error as Error, attempt, delay)

                await new Promise(resolve => setTimeout(resolve, delay))
            }
        }

        // 所有重试都失败
        this.logFailure(context, lastError!)
        throw lastError!
    }

    private isNetworkError(error: Error): boolean {
        const networkErrors = [
            'ECONNREFUSED',
            'ECONNRESET',
            'ETIMEDOUT',
            'ENOTFOUND',
            'ERR_NETWORK',
            'TIMEOUT'
        ]

        return networkErrors.some(errCode => error.message.includes(errCode)) ||
               error.name === 'NetworkError' ||
               error.name === 'TimeoutError'
    }

    private isNavigationError(error: Error): boolean {
        return error.message.includes('net::ERR') ||
               error.message.includes('Navigation timeout') ||
               error.message.includes('Protocol error') ||
               error.name === 'NavigationError'
    }

    private isInteractionError(error: Error): boolean {
        return error.message.includes('Element not found') ||
               error.message.includes('Element is not visible') ||
               error.message.includes('Element is not clickable') ||
               error.name === 'InteractionError'
    }

    private logSuccess(context: ErrorContext, attempt: number): void {
        this.addLog({
            timestamp: new Date(),
            type: 'success',
            context,
            attempt,
            message: 'Operation completed successfully'
        })
    }

    private logRetry(context: ErrorContext, error: Error, attempt: number, delay: number): void {
        this.addLog({
            timestamp: new Date(),
            type: 'retry',
            context,
            attempt,
            error: error.message,
            message: `Retry attempt ${attempt} after ${delay}ms delay`,
            delay
        })
    }

    private logFailure(context: ErrorContext, error: Error): void {
        this.addLog({
            timestamp: new Date(),
            type: 'failure',
            context,
            error: error.message,
            message: 'Operation failed after all retry attempts'
        })
    }

    private addLog(log: ErrorLog): void {
        this.errorLog.push(log)

        // 保持日志大小限制
        if (this.errorLog.length > this.maxLogSize) {
            this.errorLog.shift()
        }
    }

    getErrorLogs(context?: ErrorContext): ErrorLog[] {
        if (!context) {
            return [...this.errorLog]
        }

        return this.errorLog.filter(log =>
            log.context.operation === context.operation &&
            (!context.url || log.context.url === context.url)
        )
    }

    clearLogs(): void {
        this.errorLog = []
    }
}

// 浏览器健康检查
export class BrowserHealthChecker {
    private session: BrowserSession
    private lastHealthCheck: Date = new Date(0)
    private checkInterval: number = 30000 // 30秒

    constructor(session: BrowserSession) {
        this.session = session
    }

    async checkHealth(): Promise<HealthStatus> {
        const now = new Date()

        // 避免频繁检查
        if (now.getTime() - this.lastHealthCheck.getTime() < this.checkInterval) {
            return { status: 'healthy', lastCheck: this.lastHealthCheck }
        }

        try {
            const checks = await Promise.allSettled([
                this.checkBrowserConnection(),
                this.checkPageResponsiveness(),
                this.checkMemoryUsage(),
                this.checkConsoleErrors()
            ])

            const results = checks.map(result =>
                result.status === 'fulfilled' ? result.value : null
            )

            const issues: string[] = []
            let isHealthy = true

            // 检查结果
            if (!results[0]?.connected) {
                issues.push('Browser connection lost')
                isHealthy = false
            }

            if (!results[1]?.responsive) {
                issues.push('Page not responsive')
                isHealthy = false
            }

            if (results[2]?.memoryUsage > 0.9) {
                issues.push('High memory usage')
                // 高内存使用不标记为不健康，但需要监控
            }

            if (results[3]?.errorCount > 10) {
                issues.push('Too many console errors')
                isHealthy = false
            }

            this.lastHealthCheck = now

            return {
                status: isHealthy ? 'healthy' : 'unhealthy',
                issues,
                lastCheck: now,
                details: {
                    connection: results[0],
                    responsiveness: results[1],
                    memory: results[2],
                    console: results[3]
                }
            }
        } catch (error) {
            return {
                status: 'unhealthy',
                issues: ['Health check failed'],
                lastCheck: now,
                error: error.message
            }
        }
    }

    private async checkBrowserConnection(): Promise<ConnectionHealth> {
        try {
            const isConnected = await this.session.isConnected()
            return {
                connected: isConnected,
                responseTime: isConnected ? await this.measureResponseTime() : undefined
            }
        } catch (error) {
            return { connected: false, error: error.message }
        }
    }

    private async checkPageResponsiveness(): Promise<ResponsivenessHealth> {
        try {
            const startTime = Date.now()
            await this.session.evaluate('document.readyState')
            const responseTime = Date.now() - startTime

            return {
                responsive: responseTime < 5000,
                responseTime
            }
        } catch (error) {
            return { responsive: false, error: error.message }
        }
    }

    private async checkMemoryUsage(): Promise<MemoryHealth> {
        try {
            const memoryInfo = await this.session.evaluate(() => {
                if ('memory' in performance) {
                    return (performance as any).memory
                }
                return null
            })

            if (memoryInfo) {
                const usageRatio = memoryInfo.usedJSHeapSize / memoryInfo.jsHeapSizeLimit
                return {
                    memoryUsage: usageRatio,
                    usedHeap: memoryInfo.usedJSHeapSize,
                    totalHeap: memoryInfo.jsHeapSizeLimit
                }
            }

            return { memoryUsage: 0 }
        } catch (error) {
            return { memoryUsage: 0, error: error.message }
        }
    }

    private async checkConsoleErrors(): Promise<ConsoleHealth> {
        try {
            const logs = await this.session.getConsoleLogs()
            const errorLogs = logs.filter(log => log.type === 'error')

            return {
                errorCount: errorLogs.length,
                recentErrors: errorLogs.slice(-5) // 最近5个错误
            }
        } catch (error) {
            return { errorCount: 0, error: error.message }
        }
    }

    private async measureResponseTime(): Promise<number> {
        const start = Date.now()
        await this.session.evaluate('1+1')
        return Date.now() - start
    }
}
```

## 总结

KiloCode 的浏览器自动化系统展现了以下技术特点：

### 1. **功能完整性**
- 支持本地和远程两种浏览器模式
- 完整的页面交互（点击、输入、滚动等）
- 智能截图和内容获取
- 健康检查和错误恢复

### 2. **架构设计优秀**
- 模块化的类设计，职责清晰
- 完善的错误处理和重试机制
- 灵活的配置和扩展性
- 与 AI 助手的无缝集成

### 3. **性能优化**
- 智能等待和同步机制
- 网络活动监控
- 标签页复用策略
- 资源使用优化

### 4. **用户体验**
- 直观的坐标系统
- 实时视觉反馈
- 详细的错误信息
- 健康状态监控

这个系统代表了现代浏览器自动化的最佳实践，为 KiloCode 提供了强大的 Web 自动化能力，使用户能够通过自然语言控制浏览器完成复杂的任务。