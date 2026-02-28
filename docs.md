# ReaderCore Design Document

**Date:** 2026-02-28
**Status:** Approved
**Project:** reader-core (independent project, separate from openreader)

---

## 1. Overview

ReaderCore is an independent, React-based document reader core library that provides:

- Multi-format document rendering (EPUB, PDF, HTML, TXT, MD)
- Middleware-based plugin system for extensible rendering
- Built-in light/dark theme with CSS variable customization
- State interface without built-in persistence

**Dependencies:**
- `react` (peer dependency, ^18 || ^19)
- `pdfjs-dist` (PDF parsing & rendering)
- `epubjs` (EPUB parsing)
- No Next.js dependency

**Key design decisions:**
- Dual-track rendering: PDF uses native pdfjs rendering, EPUB/HTML/TXT/MD use unified AST-based rendering (Flow Track)
- Plugin system uses middleware/pipeline pattern with a unified Facade to abstract track differences
- TTS is NOT built-in; it should be implemented as a plugin
- State interface only; persistence is the consumer's responsibility

---

## 2. Architecture: Dual-Track + Plugin Facade

```
             +-- PDF Track ------ pdfjs-dist native render --+
File -> Router                                                 -> PluginFacade -> Plugins
             +-- Flow Track -- AST -- Middleware Pipeline -- +
```

### PDF Track
- Uses `pdfjs-dist` directly for Canvas/SVG rendering
- TextLayer for text selection and highlighting
- Plugin UI injected via overlay layer on top of PDF pages
- Plugins have limited capabilities (no render middleware, approximate paragraph positions)

### Flow Track
- All non-PDF formats parsed into a unified Document AST
- AST rendered through a middleware pipeline to React elements
- Plugins have full rendering control via middleware
- Supports EPUB, HTML, TXT, MD

---

## 3. Project Structure

```
reader-core/
src/
  core/                    # Core layer
    types.ts               # Core type definitions
    ReaderCore.ts           # Core engine (document, plugins, state)
    EventBus.ts             # Event bus
    PluginRegistry.ts       # Plugin lifecycle management

  document/                # Document abstraction layer
    DocumentModel.ts        # Unified document model (AST)
    parsers/               # Format parsers
      ParserInterface.ts
      EpubParser.ts
      HtmlParser.ts
      MarkdownParser.ts
      TxtParser.ts
    PdfDocumentProxy.ts     # PDF document proxy (content access without AST)

  rendering/               # Rendering layer
    flow/                  # Flow Track (stream content rendering)
      FlowRenderer.tsx      # AST -> React renderer
      BlockRenderer.tsx
      InlineRenderer.tsx
    pdf/                   # PDF Track (native rendering)
      PdfRenderer.tsx       # pdfjs-dist wrapper
      PdfOverlay.tsx        # Plugin overlay layer on PDF
    middleware/             # Render middleware (Flow Track only)
      MiddlewarePipeline.ts
      types.ts

  plugin/                  # Plugin system
    PluginFacade.ts         # Unified plugin facade interface
    FlowFacade.ts           # Flow Track facade implementation
    PdfFacade.ts            # PDF Track facade implementation
    types.ts                # Plugin type definitions

  state/                   # State management
    ReaderState.ts          # Reader state interface
    StateStore.ts           # Simple state container (no persistence)

  theme/                   # Theme system
    ThemeProvider.tsx
    themes.ts               # Preset light/dark
    types.ts

  components/              # Public React components
    ReaderProvider.tsx       # Top-level Provider
    ReaderView.tsx           # Main render view (auto-selects Track)
    PluginSlot.tsx           # Plugin UI slots
    Toolbar.tsx              # Extensible toolbar

package.json
tsconfig.json
vitest.config.ts
vite.config.ts              # Vite library build
```

---

## 4. Core Data Model (Document AST)

### Document

```typescript
interface Document {
  id: string;
  metadata: DocumentMetadata;
  content: DocumentContent | null; // null for PDF Track
  format: DocumentFormat;
}

type DocumentFormat = 'epub' | 'pdf' | 'html' | 'txt' | 'md';

interface DocumentMetadata {
  title: string;
  author?: string;
  language?: string;
  toc?: TocEntry[];
  totalPages?: number; // PDF only
  [key: string]: unknown;
}

interface TocEntry {
  title: string;
  href: string;
  level: number;
  children?: TocEntry[];
}
```

### Document Content (Flow Track AST)

```typescript
interface DocumentContent {
  chapters: Chapter[];
}

interface Chapter {
  id: string;
  title: string;
  blocks: Block[];
}

type Block =
  | ParagraphBlock
  | HeadingBlock
  | ImageBlock
  | ListBlock
  | BlockquoteBlock
  | CodeBlock
  | TableBlock
  | HorizontalRuleBlock;

interface BlockBase {
  id: string; // Stable block ID for plugin reference
  type: string;
  metadata?: Record<string, unknown>;
}

interface ParagraphBlock extends BlockBase {
  type: 'paragraph';
  children: InlineNode[];
}

interface HeadingBlock extends BlockBase {
  type: 'heading';
  level: 1 | 2 | 3 | 4 | 5 | 6;
  children: InlineNode[];
}

interface ImageBlock extends BlockBase {
  type: 'image';
  src: string;
  alt?: string;
  title?: string;
}

interface ListBlock extends BlockBase {
  type: 'list';
  ordered: boolean;
  items: ListItem[];
}

interface ListItem {
  children: InlineNode[];
  subList?: ListBlock;
}

interface BlockquoteBlock extends BlockBase {
  type: 'blockquote';
  children: Block[];
}

interface CodeBlock extends BlockBase {
  type: 'code';
  language?: string;
  value: string;
}

interface TableBlock extends BlockBase {
  type: 'table';
  headers: InlineNode[][];
  rows: InlineNode[][][];
}

interface HorizontalRuleBlock extends BlockBase {
  type: 'hr';
}

// Inline nodes
type InlineNode =
  | TextNode
  | BoldNode
  | ItalicNode
  | LinkNode
  | CodeSpanNode
  | BreakNode;

interface TextNode {
  type: 'text';
  value: string;
}

interface BoldNode {
  type: 'bold';
  children: InlineNode[];
}

interface ItalicNode {
  type: 'italic';
  children: InlineNode[];
}

interface LinkNode {
  type: 'link';
  href: string;
  title?: string;
  children: InlineNode[];
}

interface CodeSpanNode {
  type: 'code-span';
  value: string;
}

interface BreakNode {
  type: 'break';
}
```

### State

```typescript
interface ReaderState {
  document: Document | null;
  currentLocation: Location;
  theme: ThemeConfig;
  zoom: number;
  viewMode: 'single' | 'dual' | 'scroll';
  selection: SelectionInfo | null;
}

type Location =
  | { type: 'flow'; chapterId: string; blockId: string; offset?: number }
  | { type: 'pdf'; page: number };

interface SelectionInfo {
  text: string;
  range: LocationRange;
  paragraphId?: string;
}
```

---

## 5. Format Parsers

### Parser Interface

```typescript
interface DocumentParser {
  parse(data: ArrayBuffer | string, options?: ParseOptions): Promise<Document>;
}

interface ParseOptions {
  id: string;
  metadata?: Partial<DocumentMetadata>;
}
```

### EPUB Parser

Uses `epubjs` to parse EPUB spine items, extracts HTML content from each section, converts to AST blocks via DOM traversal.

```typescript
class EpubParser implements DocumentParser {
  async parse(data: ArrayBuffer): Promise<Document> {
    const book = ePub(data);
    await book.ready;

    const chapters: Chapter[] = [];
    for (const section of book.spine.items) {
      const doc = await section.load(book.load.bind(book));
      const blocks = htmlToBlocks(doc.innerHTML);
      chapters.push({ id: section.href, title: section.label, blocks });
    }

    return {
      content: { chapters },
      metadata: extractMetadata(book),
      format: 'epub',
    };
  }
}
```

### HTML Parser

Parses HTML string via DOMParser, recursively walks the DOM tree to generate Block[].

### Markdown Parser

Parses markdown to AST (using markdown-it or remark), converts to Block[].

### TXT Parser

Splits plain text by double newlines into paragraphs, each becomes a ParagraphBlock.

### PDF Document Proxy

PDF is NOT parsed into AST. Instead, `PdfDocumentProxy` wraps `pdfjs-dist` to provide:
- Page rendering to Canvas
- TextContent extraction for text selection and search
- Approximate paragraph detection from TextContent coordinates

---

## 6. Plugin System

### Plugin Interface

```typescript
interface ReaderPlugin {
  name: string;
  dependencies?: string[];
  setup(facade: PluginFacade): PluginCleanup | void;
  onDocumentReady?(facade: PluginFacade): void;
  destroy?(): void;
}

type PluginCleanup = () => void;
```

### Plugin Lifecycle

```
Register -> Initialize (setup) -> Document Ready (onDocumentReady) -> Running -> Destroy
```

### PluginFacade (Unified Interface)

```typescript
interface PluginFacade {
  // === Content Access ===
  getDocument(): Document;
  getVisibleParagraphs(): ParagraphInfo[];
  getParagraphText(paragraphId: string): string;
  getSelection(): SelectionInfo | null;
  getLocation(): Location;

  // === Events ===
  on<E extends keyof ReaderEventMap>(event: E, handler: ReaderEventMap[E]): Unsubscribe;
  once<E extends keyof ReaderEventMap>(event: E, handler: ReaderEventMap[E]): Unsubscribe;
  emit(event: string, data?: unknown): void;

  // === UI Injection ===
  registerSlot(slot: SlotName, component: React.ComponentType<SlotProps>): Unsubscribe;
  injectAroundBlock(
    blockId: string,
    position: 'before' | 'after' | 'wrap',
    component: React.ComponentType<BlockInjectionProps>
  ): Unsubscribe;
  injectFloating(component: React.ComponentType<FloatingProps>): Unsubscribe;

  // === Render Middleware (Flow Track only) ===
  useMiddleware(middleware: RenderMiddleware): Unsubscribe;

  // === State ===
  getStore<T>(initialState: T): PluginStore<T>;
  getReaderState(): ReaderState;

  // === Capability Query ===
  isFlowTrack(): boolean;
  isPdfTrack(): boolean;
  getCapabilities(): TrackCapabilities;
}

interface TrackCapabilities {
  renderMiddleware: boolean;
  preciseBlockPosition: boolean;
  inlineHighlight: boolean;
  textSelection: boolean;
}
```

### Event System

```typescript
interface ReaderEventMap {
  // Document events
  'document:loaded': (doc: Document) => void;
  'document:unloaded': () => void;

  // Navigation events
  'location:change': (location: Location) => void;
  'chapter:change': (chapter: Chapter) => void;

  // Interaction events
  'text:select': (selection: SelectionInfo) => void;
  'text:deselect': () => void;
  'block:click': (blockId: string, event: MouseEvent) => void;
  'block:hover': (blockId: string | null) => void;

  // View events
  'view:scroll': (scrollInfo: ScrollInfo) => void;
  'view:resize': (size: { width: number; height: number }) => void;
  'theme:change': (theme: ThemeConfig) => void;

  // Visibility events
  'blocks:visible': (blockIds: string[]) => void;

  // PDF-specific
  'pdf:page:change': (page: number) => void;
  'pdf:page:rendered': (page: number, canvas: HTMLCanvasElement) => void;
}
```

### Render Middleware (Flow Track Only)

```typescript
interface RenderMiddleware {
  name: string;
  priority?: number; // Lower = earlier. Default: 100

  transformBlock?(
    block: Block,
    context: MiddlewareContext,
    next: () => Block
  ): Block;

  wrapBlockElement?(
    element: React.ReactElement,
    block: Block,
    context: MiddlewareContext,
    next: () => React.ReactElement
  ): React.ReactElement;

  renderInline?(
    node: InlineNode,
    context: MiddlewareContext
  ): React.ReactElement | null;
}

interface MiddlewareContext {
  chapter: Chapter;
  blockIndex: number;
  facade: PluginFacade;
}
```

### UI Slots

```typescript
type SlotName =
  | 'toolbar.left'
  | 'toolbar.center'
  | 'toolbar.right'
  | 'sidebar.left'
  | 'sidebar.right'
  | 'footer'
  | 'overlay'
  | 'context-menu';
```

### PDF Track Plugin Behavior

On PDF Track, the PluginFacade implementation (PdfFacade) provides:
- **Full support**: events, slots, state, floating UI, text selection
- **Limited support**: getVisibleParagraphs (approximate via TextContent coordinates)
- **Not supported**: useMiddleware (warns and returns no-op), precise block positioning

---

## 7. React Component API

### ReaderProvider

```typescript
interface ReaderProviderProps {
  document: Document | null;
  plugins?: ReaderPlugin[];
  theme?: 'light' | 'dark' | ThemeConfig;
  initialLocation?: Location;
  viewMode?: 'single' | 'dual' | 'scroll';
  onLocationChange?: (location: Location) => void;
  onStateChange?: (state: ReaderState) => void;
  errorFallback?: React.ComponentType<{ error: ReaderError; retry: () => void }>;
  onError?: (error: ReaderError) => void;
  children: React.ReactNode;
}
```

### ReaderView

```typescript
interface ReaderViewProps {
  className?: string;
  style?: React.CSSProperties;
}
```

Automatically selects PDF Track or Flow Track based on document format.

### ReaderToolbar

```typescript
interface ReaderToolbarProps {
  hide?: ('toc' | 'theme' | 'zoom')[];
  className?: string;
}
```

Extensible via plugin slot registration (toolbar.left, toolbar.center, toolbar.right).

### useReader Hook

```typescript
interface ReaderAPI {
  document: Document | null;
  location: Location;
  theme: ThemeConfig;
  goToLocation(location: Location): void;
  goToNextPage(): void;
  goToPrevPage(): void;
  goToChapter(chapterId: string): void;
  toc: TocEntry[];
  setTheme(theme: 'light' | 'dark' | ThemeConfig): void;
  zoom: number;
  setZoom(zoom: number): void;
  viewMode: 'single' | 'dual' | 'scroll';
  setViewMode(mode: 'single' | 'dual' | 'scroll'): void;
  selection: SelectionInfo | null;
  trackType: 'flow' | 'pdf';
}
```

### Consumer Usage Example

```tsx
import {
  ReaderProvider,
  ReaderView,
  ReaderToolbar,
  useReader,
  createEpubDocument,
  createPdfDocument,
  createHtmlDocument,
} from 'reader-core';

function App() {
  const [doc, setDoc] = useState(null);

  async function handleFileOpen(file: File) {
    if (file.name.endsWith('.epub')) {
      setDoc(await createEpubDocument(await file.arrayBuffer(), { id: file.name }));
    } else if (file.name.endsWith('.pdf')) {
      setDoc(await createPdfDocument(await file.arrayBuffer(), { id: file.name }));
    } else if (file.name.endsWith('.md')) {
      setDoc(createHtmlDocument(await file.text(), { id: file.name, format: 'md' }));
    }
  }

  return (
    <ReaderProvider
      document={doc}
      plugins={[translationPlugin, notesPlugin]}
      theme="light"
      onLocationChange={(loc) => saveProgress(loc)}
    >
      <ReaderToolbar />
      <ReaderView />
    </ReaderProvider>
  );
}
```

---

## 8. Theme System

```typescript
interface ThemeConfig {
  name: string;
  colors: {
    background: string;
    foreground: string;
    primary: string;
    secondary: string;
    border: string;
    selection: string;
    highlight: string;
  };
  typography: {
    fontFamily: string;
    fontSize: number;
    lineHeight: number;
    paragraphSpacing: number;
  };
  spacing: {
    contentPadding: number;
    pageMargin: number;
  };
}
```

Themes are applied via CSS custom properties (`--reader-bg`, `--reader-fg`, `--reader-primary`, etc.), accessible to both plugins and consumers.

Built-in themes: `light` and `dark`.

---

## 9. Error Handling

### Error Types

```typescript
class ReaderError extends Error {
  constructor(
    message: string,
    public code: ReaderErrorCode,
    public cause?: unknown
  ) { super(message); }
}

enum ReaderErrorCode {
  DOCUMENT_PARSE_FAILED = 'DOCUMENT_PARSE_FAILED',
  DOCUMENT_FORMAT_UNSUPPORTED = 'DOCUMENT_FORMAT_UNSUPPORTED',
  DOCUMENT_CORRUPTED = 'DOCUMENT_CORRUPTED',
  RENDER_FAILED = 'RENDER_FAILED',
  PDF_PAGE_RENDER_FAILED = 'PDF_PAGE_RENDER_FAILED',
  PLUGIN_SETUP_FAILED = 'PLUGIN_SETUP_FAILED',
  PLUGIN_MIDDLEWARE_ERROR = 'PLUGIN_MIDDLEWARE_ERROR',
  PLUGIN_DEPENDENCY_MISSING = 'PLUGIN_DEPENDENCY_MISSING',
}
```

### Plugin Error Isolation

- Plugin errors are caught and logged but do not crash the reader
- Middleware errors cause the middleware to be skipped; default rendering continues
- Plugin errors are emitted via `plugin:error` event for monitoring

---

## 10. Testing Strategy

```
Unit Tests
  - Parser tests (each format -> AST correctness)
  - Middleware pipeline tests (execution order, error isolation)
  - EventBus tests (subscribe, publish, unsubscribe)
  - StateStore tests (read/write)
  - Theme tests (CSS variable generation)

Component Tests (React Testing Library)
  - ReaderProvider mount/unmount
  - ReaderView auto-selects correct Track
  - FlowRenderer renders AST nodes
  - Slot system (register, unregister, multi-plugin)
  - useReader hook return values

Integration Tests
  - EPUB end-to-end: load -> parse -> render -> navigate
  - PDF end-to-end: load -> render -> text selection
  - Plugin integration: register -> activate -> middleware effect -> destroy
  - Multi-plugin cooperation: two middlewares execute by priority
```

---

## 11. Build & Distribution

- Build tool: Vite (library mode)
- Output: ESM + CJS + TypeScript declarations
- Test framework: Vitest
- `peerDependencies`: `react ^18 || ^19`
- `dependencies`: `pdfjs-dist`, `epubjs`

---

## 12. Plugin Example: Translation Plugin

```typescript
const translationPlugin: ReaderPlugin = {
  name: 'translation',

  setup(facade) {
    const store = facade.getStore<{
      translations: Map<string, string>;
      isActive: boolean;
    }>({ translations: new Map(), isActive: false });

    const unregToolbar = facade.registerSlot('toolbar.right', () => (
      <button onClick={() => store.update(s => ({ ...s, isActive: !s.isActive }))}>
        Translate
      </button>
    ));

    const unregMiddleware = facade.useMiddleware({
      name: 'translation-render',
      priority: 200,
      wrapBlockElement(element, block, ctx, next) {
        const wrapped = next();
        if (block.type !== 'paragraph' || !store.get().isActive) return wrapped;
        const translation = store.get().translations.get(block.id);
        return (
          <>
            {wrapped}
            {translation && <div className="translation">{translation}</div>}
          </>
        );
      }
    });

    const unregSelect = facade.on('text:select', async (selection) => {
      if (!store.get().isActive) return;
      const translated = await translateAPI(selection.text);
      if (selection.paragraphId) {
        store.update(s => {
          const translations = new Map(s.translations);
          translations.set(selection.paragraphId!, translated);
          return { ...s, translations };
        });
      }
    });

    return () => { unregToolbar(); unregMiddleware(); unregSelect(); };
  }
};
```

---

## Data Flow Diagram

```
+-----------------------------------------------------------+
|  ReaderProvider                                            |
|  +----------------+  +----------------+  +--------------+ |
|  |  ReaderCore     |  | PluginRegistry |  |  StateStore  | |
|  |  (engine)       |  | (lifecycle)    |  |  (state)     | |
|  +-------+--------+  +-------+--------+  +------+-------+ |
|          |                    |                   |         |
|    +-----v--------------------v-------------------v-----+  |
|    |                  EventBus                          |  |
|    +-----+--------------------+-------------------+-----+  |
|          |                    |                   |         |
|  +-------v--------+  +-------v--------+  +-------v------+ |
|  |  ReaderView     |  |  Toolbar       |  | PluginSlots  | |
|  |  +-----------+  |  |                |  | (sidebar,    | |
|  |  |PDF Track  |  |  |                |  |  overlay...) | |
|  |  |or         |  |  |                |  |              | |
|  |  |Flow Track |  |  |                |  |              | |
|  |  +-----------+  |  |                |  |              | |
|  +-----------------+  +----------------+  +--------------+ |
+-----------------------------------------------------------+
```
