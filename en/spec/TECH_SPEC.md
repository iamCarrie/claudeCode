# 技術規格文件（全局）

> 此文件定義專案的技術選型與預設規範。
> 所有功能 spec 預設繼承此設定，**無需重複填寫**。
> 若特定功能有例外，在該功能的 spec 內標注「覆蓋全局設定」即可。

---

## 1. 技術選型

| 項目 | 選擇 | 版本 |
|------|------|------|
| 框架 | Vue 3 | 3.x |
| 語言 | JavaScript | — |
| 樣式方案 | SCSS | — |
| UI Library | Vuetify | 3.x |
| 狀態管理 | Pinia | — |
| 表單處理 | vee-validate | — |
| HTTP Client | Axios | — |
| 套件管理 | pnpm | — |
| 開發伺服器 / 打包工具 | **Vite** | 5.x |
| 路由 | Vue Router | 4.x |

---

## 2. 啟動專案

```bash
# 安裝依賴
pnpm install

# 開發模式啟動（必須使用 Vite）
pnpm dev
# 等同於 vite --mode development

# 打包
pnpm build
# 等同於 vite build

# 預覽打包結果
pnpm preview
# 等同於 vite preview
```

> ⚠️ **禁止使用其他方式啟動**（例如 `node server.js`、`webpack serve` 等）。
> 一律透過 Vite 開發伺服器，確保 HMR、`import.meta.env` 及 path alias 正常運作。

### vite.config.js 基本設定

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  server: {
    port: 5173,       // 預設 port
    open: true,       // 啟動後自動開啟瀏覽器
    proxy: {
      '/api': {
        target: process.env.VITE_API_URL,
        changeOrigin: true,
      },
    },
  },
})
```

### HTTPS 設定

#### 開發環境（Vite dev server）

使用 `@vitejs/plugin-basic-ssl` 快速啟用自簽憑證：

```bash
pnpm add -D @vitejs/plugin-basic-ssl
```

```javascript
// vite.config.js
import basicSsl from '@vitejs/plugin-basic-ssl'

export default defineConfig({
  plugins: [
    vue(),
    basicSsl(),   // 自動產生自簽憑證，dev server 改為 https://
  ],
  server: {
    port: 5173,
    https: true,  // 搭配 basicSsl plugin 使用
    open: true,
  },
})
```

> ⚠️ 自簽憑證瀏覽器會顯示警告，點「進階 → 繼續前往」即可。
> 僅限開發使用，**不可用於 production**。

如果需要讓區網其他裝置也能 HTTPS 存取（例如手機測試），改用 mkcert 產生本機信任憑證：

```bash
# macOS
brew install mkcert
mkcert -install
mkcert localhost 127.0.0.1
# 產生 localhost+1.pem 和 localhost+1-key.pem
```

```javascript
// vite.config.js
import fs from 'node:fs'

export default defineConfig({
  server: {
    https: {
      cert: fs.readFileSync('./localhost+1.pem'),
      key:  fs.readFileSync('./localhost+1-key.pem'),
    },
    host: true,   // 對外開放，讓區網裝置可連
  },
})
```

> ⚠️ 憑證檔案（`*.pem`）加入 `.gitignore`，不要 commit 進 repo。

#### 正式環境（Production）

Production 的 HTTPS 不由 Vite 處理，**一律在反向代理層終止 TLS**：

```
使用者 → HTTPS → Nginx / Caddy（TLS 終止）→ HTTP → Vite preview / Node server
```

**Nginx 範例**：

```nginx
server {
  listen 443 ssl;
  server_name example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:4173;   # vite preview 預設 port
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

# 強制 HTTP → HTTPS
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

**Caddy 範例**（自動申請 Let's Encrypt，設定最簡單）：

```
# Caddyfile
example.com {
  reverse_proxy 127.0.0.1:4173
}
```

> 憑證申請推薦使用 **Let's Encrypt**（免費），搭配 Certbot 或 Caddy 自動續約。

---

## 2.5 組件自動引入（Auto Import）

使用 `unplugin-vue-components` 讓 `src/components/` 下的所有元件**自動按需引入**，無需在每個 SFC 手動 import。

### 安裝

```bash
pnpm add -D unplugin-vue-components
```

### vite.config.js 設定

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'
import Components from 'unplugin-vue-components/vite'
import { VuetifyResolver } from 'unplugin-vue-components/resolvers'

export default defineConfig({
  plugins: [
    vue(),
    Components({
      // 自動掃描的資料夾（預設已包含 src/components）
      dirs: ['src/components'],

      // 副檔名
      extensions: ['vue'],

      // 自動產生 TypeScript 型別宣告檔
      dts: 'src/components.d.ts',

      // Vuetify 元件同步自動引入（v-btn、v-text-field 等不需要 import）
      resolvers: [VuetifyResolver()],
    }),
  ],
  // ... 其餘設定
})
```

### 效果

設定完成後，`src/components/` 下的所有元件直接在 template 使用，**不需要 import 也不需要 components 註冊**：

```vue
<!-- ✅ 之前：每個頁面都要手動 import -->
<script setup>
import AppInput      from '@/components/base/AppInput.vue'
import AppSelect     from '@/components/base/AppSelect.vue'
import AppForm       from '@/components/base/AppForm.vue'
</script>

<!-- ✅ 之後：直接用，plugin 自動處理 -->
<script setup>
// 不需要任何 import
</script>

<template>
  <AppForm @submit="handleSubmit">
    <template #default="{ isSubmitting }">
      <AppInput  name="email" label="Email" rules="required|email" />
      <AppSelect name="role"  label="Role"  rules="required" :items="roles" />
      <v-btn type="submit" :loading="isSubmitting">送出</v-btn>
    </template>
  </AppForm>
</template>
```

### 注意事項

- 自動引入是**按需（on-demand）**的，不會把所有元件打包進 bundle，不影響 tree-shaking
- 元件檔名即為使用名稱（PascalCase），例如 `AppInput.vue` → `<AppInput />`
- 若有子資料夾（如 `base/AppInput.vue`），元件名稱仍為 `AppInput`（不含資料夾名）
- 新增元件後 dev server 會自動更新 `components.d.ts`，不需重啟
- `components.d.ts` 加入 `.gitignore` 或 commit 皆可，建議 commit 以確保 CI 型別正確

### .gitignore 建議

```
# 自動產生，可選擇 commit 或忽略
# src/components.d.ts
```



---

## 3. Vue Router 規範

### 3.1 基本設定

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      component: () => import('@/layouts/DefaultLayout.vue'),
      children: [
        {
          path: '',
          name: 'Home',
          component: () => import('@/views/HomeView.vue'),
        },
        {
          path: 'checkout',
          name: 'Checkout',
          component: () => import('@/views/CheckoutView.vue'),
          meta: { requiresAuth: true },
        },
      ],
    },
    {
      path: '/login',
      name: 'Login',
      component: () => import('@/views/LoginView.vue'),
    },
    {
      path: '/:pathMatch(.*)*',
      name: 'NotFound',
      component: () => import('@/views/NotFoundView.vue'),
    },
  ],
})

export default router
```

### 3.2 命名規範

| 對象 | 規則 | 範例 |
|------|------|------|
| Route `name` | PascalCase | `'CheckoutConfirm'` |
| Route `path` | kebab-case | `'/order-complete'` |
| View 元件檔名 | PascalCase + `View` 後綴 | `CheckoutView.vue` |

### 3.3 導航方式

```javascript
// ✅ 一律用 name 導航，不寫死 path 字串
const router = useRouter()
router.push({ name: 'Checkout' })
router.push({ name: 'OrderDetail', params: { id: '123' } })
router.push({ name: 'Search', query: { keyword: 'hello' } })

// ❌ 避免
router.push('/checkout')
```

### 3.4 Route Meta 與 Navigation Guard

```javascript
// router/index.js — 全局前置守衛
router.beforeEach((to, from) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isLoggedIn) {
    return { name: 'Login', query: { redirect: to.fullPath } }
  }
})

// meta 欄位定義（在 route 設定內使用）
// requiresAuth: true     → 需要登入
// layout: 'blank'        → 使用空白 layout（不含導覽列）
// title: '頁面標題'       → 用於更新 document.title
```

### 3.5 Lazy Loading

```javascript
// ✅ 所有 view 元件一律 lazy load（動態 import）
component: () => import('@/views/CheckoutView.vue')

// ❌ 避免直接 import（會打包進主 bundle）
import CheckoutView from '@/views/CheckoutView.vue'
```

---

## 4. 專案結構規範

```
src/
├── assets/               # 靜態資源（圖片、全局 SCSS）
│   └── styles/
│       ├── _variables.scss   # SCSS 變數
│       ├── _mixins.scss      # SCSS mixins
│       └── main.scss         # 全局樣式入口
├── components/
│   ├── base/             # 基礎 UI 元件（封裝 Vuetify）
│   └── [feature]/        # 功能型元件，依功能分資料夾
├── composables/          # 可複用的 Composition API 邏輯
├── lib/                  # 工具函數、API client 設定
│   └── api/              # 各資源 API 模組
├── router/               # Vue Router 設定
├── stores/               # Pinia stores
├── views/                # 頁面層級元件（對應路由）
└── docs/                 # spec 文件放這裡
    ├── TECH_SPEC.md      # 本文件
    └── spec-[feature].md # 各功能規格
```

---

## 3. 命名規範

| 對象 | 規則 | 範例 |
|------|------|------|
| 元件檔案 | PascalCase | `CheckoutForm.vue` |
| Composable 檔案 | camelCase，前綴 `use` | `useCartData.js` |
| 工具函數 | camelCase | `formatPrice.js` |
| SCSS 檔案 | kebab-case，partial 加 `_` 前綴 | `_card-styles.scss` |
| SCSS class | BEM / kebab-case | `.cart-summary__item` |
| 常數 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Pinia Store 檔案 | camelCase | `cartStore.js` |
| Pinia Store ID | camelCase | `useCartStore` |

---

## 4. 元件預設規範

```vue
<!-- ✅ 標準 SFC 結構，一律使用 <script setup> -->
<script setup>
// 1. import
// 2. props / emits 定義
// 3. store / composables
// 4. 響應式資料（ref / reactive / computed）
// 5. 方法
// 6. lifecycle hooks

const props = defineProps({
  itemId: {
    type: String,
    required: true,
  },
})

const emit = defineEmits(['update', 'delete'])
</script>

<template>
  <!-- 根元素只有一個 -->
</template>

<style lang="scss" scoped>
// 元件私有樣式，預設 scoped
</style>

// ✅ 一律使用 <script setup> 語法
// ✅ 超過 200 行考慮拆子元件
// ✅ 樣式預設 scoped，全局樣式放 assets/styles/
```

---

## 5. API 請求規範

### 5.1 Axios 設定

```javascript
// lib/axios.js
import axios from 'axios'

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
})

// Request 攔截器：自動帶入 token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// Response 攔截器：統一處理 401 / 500
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // 導向登入頁
    }
    return Promise.reject(error)
  }
)

export default apiClient
```

### 5.2 API 模組化

```javascript
// lib/api/cart.js — 每個資源一個檔案
import apiClient from '@/lib/axios'

export const cartApi = {
  getCart: () => apiClient.get('/cart'),
  updateItem: (itemId, payload) => apiClient.put(`/cart/items/${itemId}`, payload),
  deleteItem: (itemId) => apiClient.delete(`/cart/items/${itemId}`),
}
```

### 5.3 錯誤格式

```javascript
// 統一錯誤格式（後端約定）
// {
//   code: 'CART_NOT_FOUND',   // 業務錯誤碼
//   message: '購物車不存在',   // 使用者可讀訊息
//   details: ...              // 選填
// }

// 全局攔截器處理 401 / 500
// 個別元件只需處理業務邏輯錯誤（顯示 message）
```

---

## 6. Pinia Store 規範

```javascript
// stores/cartStore.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { cartApi } from '@/lib/api/cart'

export const useCartStore = defineStore('cart', () => {
  // state
  const items = ref([])
  const isLoading = ref(false)
  const error = ref(null)

  // getters
  const totalPrice = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  // actions
  async function fetchCart() {
    isLoading.value = true
    error.value = null
    try {
      const { data } = await cartApi.getCart()
      items.value = data.items
    } catch (err) {
      error.value = err.response?.data?.message ?? '載入失敗'
    } finally {
      isLoading.value = false
    }
  }

  return { items, isLoading, error, totalPrice, fetchCart }
})

// ✅ 使用 Setup Store 語法（比 Options Store 更靈活）
// ✅ isLoading / error 每個 store 標配
// ✅ actions 內統一 try/catch，錯誤存進 error state
```

---

## 8. 表單驗證規範

> 表單驗證規範已獨立為專屬文件，請參閱 `docs/spec-form-validation.md`。
> 涵蓋：統一 rules 管理（`src/plugins/veeValidate.js`）、各元件用法（Input、Textarea、Radio、Checkbox、Select、Autocomplete、File Upload）。

---

## 9. 樣式規範

```scss
// assets/styles/_variables.scss — 統一定義設計 token
$color-primary: #1976D2;
$color-error:   #D32F2F;
$spacing-base:  8px;

// assets/styles/_mixins.scss
@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: center;
}

@mixin respond-to($breakpoint) {
  @if $breakpoint == 'sm' {
    @media (min-width: 600px) { @content; }
  } @else if $breakpoint == 'md' {
    @media (min-width: 960px) { @content; }
  } @else if $breakpoint == 'lg' {
    @media (min-width: 1280px) { @content; }
  }
}
```

```vue
<style lang="scss" scoped>
@use '@/assets/styles/variables' as *;
@use '@/assets/styles/mixins' as *;

.cart-summary {
  padding: $spacing-base * 2;

  @include respond-to('md') {
    padding: $spacing-base * 3;
  }
}
</style>

// ✅ 顏色、間距使用 SCSS 變數，不寫死 hex
// ✅ 元件樣式預設 scoped
// ✅ 全局 / reset 樣式放 assets/styles/main.scss
// ✅ Vuetify 主題色在 plugins/vuetify.js 統一設定
```

---

## 10. 預設 States 處理方式

所有需要非同步資料的元件，預設要處理以下 states：

| State | 處理方式 |
|-------|---------|
| `isLoading` | `<v-skeleton-loader>` 或 `<v-progress-circular>`，版面與實際內容等高 |
| `error` | 顯示錯誤訊息 + 重試按鈕 |
| `empty` | 空狀態插圖 + 引導操作文字 |
| 正常 | 渲染實際內容 |

```vue
<template>
  <v-skeleton-loader v-if="store.isLoading" type="card" />

  <v-alert v-else-if="store.error" type="error" :text="store.error">
    <v-btn variant="text" @click="store.fetchData()">重試</v-btn>
  </v-alert>

  <EmptyState v-else-if="!store.items.length" />

  <ActualContent v-else :items="store.items" />
</template>
```

---

## 11. Toast / 通知規範

```javascript
// composables/useToast.js — 全局共用
import { ref } from 'vue'

const snackbar = ref({ show: false, message: '', color: 'success' })

export function useToast() {
  function show(message, color = 'success') {
    snackbar.value = { show: true, message, color }
  }
  return {
    snackbar,
    success: (msg) => show(msg, 'success'),
    error:   (msg) => show(msg, 'error'),
    info:    (msg) => show(msg, 'info'),
  }
}

// 在 App.vue 掛載 <v-snackbar> 監聽 snackbar state
// 規則：
// ✅ 成功操作 → success toast
// ✅ 使用者驗證錯誤 → 表單 inline error-messages，不用 toast
// ✅ 系統錯誤（API 失敗）→ error toast
```

---

## 12. Responsive 預設斷點

對齊 Vuetify 預設斷點：

| 裝置 | 斷點 | 設計稿寬度 |
|------|------|-----------|
| Mobile | `< 600px` | 375px |
| Tablet | `600px ~ 959px` | 768px |
| Desktop | `≥ 960px` | 1440px |

```vue
<!-- ✅ 優先使用 Vuetify Grid System -->
<v-row>
  <v-col cols="12" md="8">主內容</v-col>
  <v-col cols="12" md="4">側欄</v-col>
</v-row>

<!-- ✅ 需要自訂時，用 SCSS mixin respond-to() -->
```

> 開發優先順序：Mobile First

---

## 13. 環境變數

```bash
# .env.local 範例（Vite 專案）
VITE_API_URL=https://api.example.com
VITE_APP_ENV=development
```

> Vite 環境變數需加 `VITE_` 前綴才能在客戶端存取（`import.meta.env.VITE_API_URL`）。
