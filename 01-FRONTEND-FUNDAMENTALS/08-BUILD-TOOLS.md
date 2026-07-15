# BUILD TOOLS & BUNDLERS
## Exam-Style Study Notes

---

## TOPIC: Modern Build Tools for Frontend Development

### WHY? (Problem Statement)

**Before Build Tools:**
- Multiple `<script>` tags, manual dependency ordering
- No module system (IIFE, globals pollution)
- No transpilation (write ES5 only)
- No optimization (huge bundles, no tree-shaking)
- No dev server (manual refresh, no HMR)

**What Build Tools Solve:**
| Problem | Solution |
|---------|----------|
| Module system | ES Modules (import/export) |
| Browser compatibility | Transpilation (Babel, SWC) |
| Performance | Bundling, minification, tree-shaking, code splitting |
| Developer experience | Dev server, HMR, TypeScript support |
| Production optimization | Hashing, compression, asset optimization |

**Real-World Analogy:**
- No build tool = Hand-delivering individual letters
- Build tool = Logistics company (sorts, packages, optimizes routes, tracks delivery)

---

### HOW? (Internal Mechanism)

#### 1. **Bundler Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                      BUNDLING PIPELINE                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ENTRY POINTS                                              │
│  ├── src/main.ts                                           │
│  ├── src/styles.css                                        │
│  └── src/app.tsx                                           │
│       │                                                    │
│       ▼                                                    │
│  DEPENDENCY GRAPH (AST parsing)                            │
│  ├── import './utils'                                      │
│  ├── import 'react'                                        │
│  └── import './Component.vue'                              │
│       │                                                    │
│       ▼                                                    │
│  TRANSFORMATION                                            │
│  ├── TypeScript → JavaScript                               │
│  ├── JSX → React.createElement                             │
│  ├── Vue/Svelte → Render functions                         │
│  ├── CSS → JS modules / Extract                            │
│  └── Assets → Hash + Copy                                  │
│       │                                                    │
│       ▼                                                    │
│  OPTIMIZATION                                              │
│  ├── Tree-shaking (dead code elimination)                  │
│  ├── Code splitting (dynamic import)                       │
│   ├── Minification (Terser, esbuild)                       │
│   └── Scope hoisting                                       │
│       │                                                    │
│       ▼                                                    │
│  OUTPUT                                                    │
│  ├── dist/main.[hash].js                                   │
│  ├── dist/vendor.[hash].js                                 │
│  ├── dist/styles.[hash].css                                │
│  └── dist/index.html                                       │
└─────────────────────────────────────────────────────────────┘
```

#### 2. **Module Resolution**

```
import './local-file'           → Relative path
import 'package-name'          → node_modules/package-name
import '@/components/Button'   → Path alias (tsconfig paths)
import 'pkg/subpath'           → package.json exports field
import 'pkg'                   → package.json main/module/browser
```

---

### WHAT? (Key Tools & Config)

#### 1. **Vite (Modern Standard)**

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
    },
  },
  
  build: {
    target: 'es2020',
    outDir: 'dist',
    sourcemap: true,
    minify: 'esbuild', // or 'terser'
    cssCodeSplit: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          ui: ['@headlessui/react', '@heroicons/react'],
        },
      },
    },
  },
  
  server: {
    port: 3000,
    open: true,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true,
      },
    },
  },
  
  // Optimize deps
  optimizeDeps: {
    include: ['react', 'react-dom'],
    exclude: ['some-heavy-dep'],
  },
});
```

**Why Vite?**
- Native ES Modules in dev (no bundling!)
- esbuild for transforms (10-100x faster than Webpack)
- Rollup for production builds
- Instant HMR regardless of app size

#### 2. **Rollup (Library Bundler)**

```javascript
// rollup.config.js
import typescript from '@rollup/plugin-typescript';
import { nodeResolve } from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import json from '@rollup/plugin-json';
import { terser } from 'rollup-plugin-terser';
import dts from 'rollup-plugin-dts';

const packageJson = require('./package.json');

export default [
  // ESM + CJS builds
  {
    input: 'src/index.ts',
    output: [
      { file: packageJson.main, format: 'cjs', sourcemap: true },
      { file: packageJson.module, format: 'esm', sourcemap: true },
    ],
    plugins: [
      nodeResolve({ browser: true }),
      commonjs(),
      json(),
      typescript({ tsconfig: './tsconfig.json' }),
      terser(),
    ],
    external: ['react', 'react-dom'],
  },
  // Type declarations
  {
    input: 'src/index.ts',
    output: { file: 'dist/index.d.ts', format: 'esm' },
    plugins: [dts()],
  },
];
```

#### 3. **esbuild (Speed Demon)**

```javascript
// build.js
const esbuild = require('esbuild');

await esbuild.build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  outfile: 'dist/bundle.js',
  platform: 'browser',
  format: 'esm',
  target: 'es2020',
  sourcemap: true,
  minify: true,
  treeShaking: true,
  splitting: true,
  external: ['react', 'react-dom'],
  loader: { '.png': 'dataurl', '.svg': 'text' },
  define: {
    'process.env.NODE_ENV': '"production"',
  },
});

// Watch mode
const ctx = await esbuild.context({ /* same options */ });
await ctx.watch();
await ctx.serve({ port: 3000, servedir: 'dist' });
```

#### 4. **Webpack (Enterprise Standard)**

```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');

module.exports = (env, argv) => {
  const isProd = argv.mode === 'production';
  
  return {
    entry: './src/index.tsx',
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: isProd ? '[name].[contenthash].js' : '[name].js',
      chunkFilename: isProd ? '[name].[contenthash].chunk.js' : '[name].chunk.js',
      clean: true,
      publicPath: '/',
    },
    
    resolve: {
      extensions: ['.tsx', '.ts', '.js', '.jsx'],
      alias: { '@': path.resolve(__dirname, 'src') },
    },
    
    module: {
      rules: [
        {
          test: /\.(ts|tsx)$/,
          use: [
            { loader: 'babel-loader', options: { presets: ['@babel/preset-react', '@babel/preset-typescript'] } },
            // OR: { loader: 'swc-loader' } // Much faster
            // OR: { loader: 'esbuild-loader' } // Fastest
          ],
          exclude: /node_modules/,
        },
        {
          test: /\.css$/,
          use: [isProd ? MiniCssExtractPlugin.loader : 'style-loader', 'css-loader', 'postcss-loader'],
        },
        {
          test: /\.(png|svg|jpg|webp)$/,
          type: 'asset',
          parser: { dataUrlCondition: { maxSize: 8 * 1024 } },
        },
      ],
    },
    
    plugins: [
      new HtmlWebpackPlugin({ template: './public/index.html' }),
      new MiniCssExtractPlugin({ filename: isProd ? '[name].[contenthash].css' : '[name].css' }),
    ],
    
    optimization: {
      minimize: isProd,
      minimizer: [
        new TerserPlugin({ extractComments: false, parallel: true }),
        new CssMinimizerPlugin(),
      ],
      splitChunks: {
        chunks: 'all',
        cacheGroups: {
          vendor: { test: /[\\/]node_modules[\\/]/, name: 'vendors', priority: 10 },
          common: { minChunks: 2, priority: 5, reuseExistingChunk: true },
        },
      },
      runtimeChunk: 'single',
    },
    
    devServer: {
      port: 3000,
      hot: true,
      historyApiFallback: true,
      proxy: { '/api': 'http://localhost:5000' },
    },
  };
};
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | Anti-Pattern | Why Avoid |
|---------|--------------|-----------|
| **Vite for apps** | Webpack for new projects | Vite = faster dev, simpler config |
| **Rollup for libraries** | Vite for libs | Rollup = better ESM/CJS, smaller |
| **esbuild for transforms** | Babel for everything | esbuild = 100x faster |
| **Code splitting** | Single bundle | Load only what's needed |
| **Content hashes** | No hashing | Cache busting broken |
| **Tree shaking** | Side effects in modules | Dead code stays |
| **Path aliases** | Relative imports `../../../` | Refactoring breaks imports |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "How does tree-shaking work?"

**Answer:**
```
Tree-shaking = Dead Code Elimination via Static Analysis

REQUIREMENTS:
1. ES Modules (import/export) - static structure
2. No side effects - or declare in package.json
3. Minifier that removes unused code (Terser, esbuild)

PROCESS:
1. Build dependency graph from entry points
2. Mark all exports as "used" initially
3. Traverse graph - only keep reachable exports
4. Remove unused exports from modules
5. Minifier removes dead code from bundles

PACKAGE.JSON SIDE EFFECTS:
{
  "sideEffects": false          // All files pure
  // OR
  "sideEffects": ["./polyfills.js", "*.css"]  // Specific files
}
```

#### 2. **Code**: "Configure code splitting for React routes"

```javascript
// Vite / Rollup / Webpack all support dynamic import
// React.lazy + Suspense

import { Suspense, lazy } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Profile = lazy(() => import('./pages/Profile'));

// Webpack magic comment for chunk naming
const HeavyChart = lazy(() => import(/* webpackChunkName: "charts" */ './HeavyChart'));

// Vite equivalent
const HeavyChart = lazy(() => import('./HeavyChart?chunkName=charts'));

// Router with Suspense
function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}

// Preload on hover (performance)
<Link 
  to="/dashboard" 
  onMouseEnter={() => import('./pages/Dashboard')}
/>
```

#### 3. **Performance**: "Build takes too long - optimize"

```javascript
// 1. Use esbuild/SWC instead of Babel
// vite.config.ts
export default defineConfig({
  esbuild: {
    // Already used by default for TS transpilation
  },
  // For React: use @vitejs/plugin-react-swc instead of plugin-react
});

// 2. Exclude heavy deps from optimization
optimizeDeps: {
  exclude: ['@myorg/heavy-ui-lib'],
}

// 3. Cache in CI
// GitHub Actions
- uses: actions/cache@v3
  with:
    path: |
      node_modules
      ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

// 4. Parallelize
// Webpack: thread-loader, TerserPlugin parallel: true
// Vite: native parallel (esbuild, Rollup)

// 5. Reduce type checking in dev
// tsconfig.json
{ "compilerOptions": { "skipLibCheck": true } }
// CI: separate type-check job

// 6. Persistent cache
// Webpack 5: cache: { type: 'filesystem' }
// Vite: cacheDir: node_modules/.vite
```

#### 4. **Architecture**: "Monorepo build setup"

```javascript
// pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'

// turbo.json (Turborepo)
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"],
      "cache": true
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "lint": {},
    "dev": { "cache": false, "persistent": true }
  }
}

// Package.json (app)
{
  "scripts": {
    "build": "vite build",
    "dev": "vite",
    "test": "vitest run"
  }
}

// Package.json (shared lib)
{
  "name": "@myorg/ui",
  "main": "dist/index.cjs.js",
  "module": "dist/index.esm.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.esm.js",
      "require": "./dist/index.cjs.js",
      "types": "./dist/index.d.ts"
    },
    "./Button": { /* ... */ }
  },
  "scripts": {
    "build": "rollup -c",
    "dev": "rollup -c --watch"
  }
}
```

#### 5. **Debugging**: "Production build works locally but fails on server"

```
CHECKLIST:
☐ Node version matches (check .nvmrc / engines in package.json)
☐ Environment variables (API URLs, secrets)
☐ Case-sensitive imports (Linux vs Mac/Windows)
☐ Missing .env.production file
☐ Base path / publicPath mismatch (subdirectory deploy)
☐ Chunk loading fails (CSP, CSP hash, network)
☐ Source maps exposed (disable in prod!)
☐ Bundle size limits (CDN, function size limits)
☐ Dynamic import paths (must be static for analysis)

COMMON FIX:
vite.config.ts:
build: {
  assetsDir: 'assets',
  rollupOptions: {
    output: {
      entryFileNames: 'assets/[name].[hash].js',
      chunkFileNames: 'assets/[name].[hash].js',
      assetFileNames: 'assets/[name].[hash].[ext]',
    }
  }
}
```

---

### GOTCHAS & EDGE CASES

- [ ] **CommonJS vs ESM**: `require()` vs `import` - interop issues
- [ ] **`sideEffects: false`** breaks CSS imports, polyfills - list exceptions
- [ ] **Dynamic imports** must have static string for analysis: `import('./pages/' + page)` ❌
- [ ] **Circular dependencies** - Webpack handles, Rollup warns, esbuild errors
- [ ] **TypeScript `isolatedModules`** - required for esbuild/SWC (no cross-file emit)
- [ ] **CSS order** - import order matters for cascade
- [ ] **Asset inlining** - small images as data URLs, large as files
- [ ] **Node polyfills** - `process`, `Buffer`, `global` need manual polyfill in browser
- [ ] **Dev vs Prod parity** - HMR, source maps, define values differ

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete Vite Config for React + TS**
```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react-swc'; // or @vitejs/plugin-react
import path from 'path';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    mode === 'analyze' && visualizer({ open: true, gzipSize: true }),
  ].filter(Boolean),
  
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
  
  build: {
    target: 'es2020',
    sourcemap: true,
    minify: 'esbuild',
    cssCodeSplit: true,
    reportCompressedSize: true,
    chunkSizeWarningLimit: 1000,
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
          'utils': ['date-fns', 'zod', 'axios'],
        },
        chunkFileNames: 'assets/js/[name]-[hash].js',
        entryFileNames: 'assets/js/[name]-[hash].js',
        assetFileNames: 'assets/[ext]/[name]-[hash].[ext]',
      },
    },
  },
  
  server: {
    port: 3000,
    host: true,
    hmr: { overlay: true },
  },
  
  optimizeDeps: {
    include: ['react', 'react-dom'],
    force: true,
  },
  
  define: {
    'import.meta.env.BUILD_TIME': JSON.stringify(new Date().toISOString()),
  },
}));
```

#### 2. **Bundle Analysis**
```bash
# Vite
npm run build -- --mode analyze

# Webpack
npx webpack-bundle-analyzer dist/stats.json

# Rollup
npx rollup-plugin-visualizer dist/stats.html

# All: Look for:
# - Large dependencies (moment.js → date-fns)
# - Duplicate code (lodash in multiple chunks)
# - Missing tree-shaking (sideEffects issue)
# - Large chunks (>500KB gzipped)
```

#### 3. **Environment Variables**
```typescript
// .env (committed - defaults)
VITE_API_URL=http://localhost:5000
VITE_APP_TITLE=My App

// .env.local (gitignored - local overrides)
VITE_API_URL=https://api.staging.example.com

// .env.production (committed - prod values)
VITE_API_URL=https://api.example.com

// Usage in code
console.log(import.meta.env.VITE_API_URL);
console.log(import.meta.env.PROD); // boolean
console.log(import.meta.env.DEV);  // boolean

// TypeScript types
// vite-env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
}
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

### PRACTICE PROBLEMS

1. **Setup** a Vite + React + TS project from scratch with: path aliases, code splitting, bundle analysis
2. **Configure** a monorepo with Turborepo: 2 apps, 3 shared packages
3. **Optimize** a slow build: profile, identify bottlenecks, apply fixes
4. **Migrate** a Webpack project to Vite - document challenges
5. **Debug** a production-only bug: sourcemap analysis, chunk loading issues

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Tree-shaking requirements | ES Modules, no side effects, minifier |
| Vite dev server | Native ES Modules (no bundle), esbuild transforms |
| Code splitting syntax | `const Comp = lazy(() => import('./Comp'))` |
| Rollup vs Vite | Rollup: libraries. Vite: applications |
| esbuild vs Babel | esbuild: 100x faster, Go-based, no plugins |
| `sideEffects: false` | Enables aggressive tree-shaking, list exceptions |
| Dynamic import chunk name | `import(/* webpackChunkName: "name" */ './file')` |
| Bundle analysis | rollup-plugin-visualizer, webpack-bundle-analyzer |
| `isolatedModules` TS flag | Required for esbuild/SWC (single-file emit) |
| Vite optimizeDeps | Pre-bundles deps for faster dev server startup |

---

## NEXT TOPIC: `09-TESTING.md`

> **Study Tip**: Create a Vite project, add bundle analyzer, experiment with manualChunks. Measure gzip size before/after.