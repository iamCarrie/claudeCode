# Tech Spec (Global)

> This document defines the project's technology choices and default conventions.
> All feature specs inherit these settings by default — **no need to repeat them**.
> If a specific feature overrides any global setting, note it as "Override global setting" in that feature's spec.

---

## 1. Tech Stack

| Item | Choice | Version |
|------|--------|---------|
| Framework | Vue 3 | 3.x |
| Language | JavaScript | — |
| Styling | SCSS | — |
| UI Library | Vuetify | 3.x |
| State Management | Pinia | — |
| Form Handling | vee-validate | — |
| HTTP Client | Axios | — |
| Package Manager | pnpm | — |
| Dev Server / Bundler | **Vite** | 5.x |
| Routing | Vue Router | 4.x |

---

## 2. Running the Project

```bash
# Install dependencies
pnpm install

# Start dev server (must use Vite)
pnpm dev
# Equivalent to: vite --mode development

# Build for production
pnpm build
# Equivalent to: vite build

# Preview production build
pnpm preview
# Equivalent to: vite preview
```

> ⚠️ **Do not start the project any other way** (e.g. `node server.js`, `webpack serve`, etc.).
> Always use the Vite dev server to ensure HMR, `import.meta.env`, and path aliases work correctly.

### 2.1 vite.config.js Base Config

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
    port: 5173,       // default port
    open: true,       // auto-open browser on start
    proxy: {
      '/api': {
        target: process.env.VITE_API_URL,
        changeOrigin: true,
      },
    },
  },
})
```

### 2.2 HTTPS Configuration

#### Development (Vite dev server)

Use `@vitejs/plugin-basic-ssl` for a quick self-signed certificate:

```bash
pnpm add -D @vitejs/plugin-basic-ssl
```

```javascript
// vite.config.js
import basicSsl from '@vitejs/plugin-basic-ssl'

export default defineConfig({
  plugins: [
    vue(),
    basicSsl(),   // auto-generates a self-signed cert, dev server switches to https://
  ],
  server: {
    port: 5173,
    https: true,  // used together with the basicSsl plugin
    open: true,
  },
})
```

> ⚠️ The browser will show a warning for self-signed certificates — click "Advanced → Proceed" to continue.
> For development only. **Never use in production.**

If you need HTTPS access from other devices on the local network (e.g. mobile testing), use mkcert to generate a locally-trusted certificate instead:

```bash
# macOS
brew install mkcert
mkcert -install
mkcert localhost 127.0.0.1
# Generates localhost+1.pem and localhost+1-key.pem
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
    host: true,   // expose to local network
  },
})
```

> ⚠️ Add certificate files (`*.pem`) to `.gitignore` — do not commit them to the repo.

#### Production

HTTPS in production is **not handled by Vite**. TLS should always be terminated at the reverse proxy layer:

```
Client → HTTPS → Nginx / Caddy (TLS termination) → HTTP → Vite preview / Node server
```

**Nginx example**:

```nginx
server {
  listen 443 ssl;
  server_name example.com;

  ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:4173;   # default vite preview port
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

# Force HTTP → HTTPS
server {
  listen 80;
  server_name example.com;
  return 301 https://$host$request_uri;
}
```

**Caddy example** (auto-provisions Let's Encrypt — simplest setup):

```
# Caddyfile
example.com {
  reverse_proxy 127.0.0.1:4173
}
```

> Recommended: use **Let's Encrypt** (free) with Certbot or Caddy for automatic certificate renewal.

---

## 3. Component Auto Import

Use `unplugin-vue-components` so every component under `src/components/` is **automatically imported on demand** — no manual imports needed in any SFC.

### 3.1 Installation

```bash
pnpm add -D unplugin-vue-components
```

### 3.2 vite.config.js Setup

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
      // Folders to scan (src/components is included by default)
      dirs: ['src/components'],

      // File extensions to include
      extensions: ['vue'],

      // Auto-generate TypeScript declaration file
      dts: 'src/components.d.ts',

      // Also auto-import Vuetify components (v-btn, v-text-field, etc.)
      resolvers: [VuetifyResolver()],
    }),
  ],
  // ... rest of config
})
```

### 3.3 Result

Once configured, every component under `src/components/` can be used directly in templates — **no import statement, no components registration required**:

```vue
<!-- ✅ Before: manual imports on every page -->
<script setup>
import AppInput  from '@/components/base/AppInput.vue'
import AppSelect from '@/components/base/AppSelect.vue'
import AppForm   from '@/components/base/AppForm.vue'
</script>

<!-- ✅ After: use directly, the plugin handles it -->
<script setup>
// no imports needed
</script>

<template>
  <AppForm @submit="handleSubmit">
    <template #default="{ isSubmitting }">
      <AppInput  name="email" label="Email" rules="required|email" />
      <AppSelect name="role"  label="Role"  rules="required" :items="roles" />
      <v-btn type="submit" :loading="isSubmitting">Submit</v-btn>
    </template>
  </AppForm>
</template>
```

### 3.4 Notes

- Auto import is **on-demand** — unused components are never bundled, tree-shaking is preserved
- Component filename = component tag name (PascalCase): `AppInput.vue` → `<AppInput />`
- Components in subdirectories (e.g. `base/AppInput.vue`) are registered by filename only — `<AppInput />`, not `<BaseAppInput />`
- Adding a new component auto-updates `components.d.ts` without restarting the dev server
- Committing `components.d.ts` is recommended to keep CI type-checking accurate

### 3.5 .gitignore Note

```
# Auto-generated — commit or ignore, your choice
# src/components.d.ts
```



---

## 4. Vue Router Conventions

### 4.1 Basic Setup

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

### 4.2 Naming Conventions

| Target | Rule | Example |
|--------|------|---------|
| Route `name` | PascalCase | `'CheckoutConfirm'` |
| Route `path` | kebab-case | `'/order-complete'` |
| View component filename | PascalCase + `View` suffix | `CheckoutView.vue` |

### 4.3 Navigation

```javascript
// ✅ Always navigate by name — never hardcode path strings
const router = useRouter()
router.push({ name: 'Checkout' })
router.push({ name: 'OrderDetail', params: { id: '123' } })
router.push({ name: 'Search', query: { keyword: 'hello' } })

// ❌ Avoid
router.push('/checkout')
```

### 4.4 Route Meta & Navigation Guards

```javascript
// router/index.js — global beforeEach guard
router.beforeEach((to, from) => {
  const authStore = useAuthStore()

  if (to.meta.requiresAuth && !authStore.isLoggedIn) {
    return { name: 'Login', query: { redirect: to.fullPath } }
  }
})

// Available meta fields (set per route)
// requiresAuth: true   → requires authentication
// layout: 'blank'      → use blank layout (no navbar)
// title: 'Page Title'  → used to update document.title
```

### 4.5 Lazy Loading

```javascript
// ✅ All view components must use lazy loading (dynamic import)
component: () => import('@/views/CheckoutView.vue')

// ❌ Avoid static imports (gets bundled into the main chunk)
import CheckoutView from '@/views/CheckoutView.vue'
```

---

## 5. Project Structure

```
src/
├── assets/                   # Static assets (images, global SCSS)
│   └── styles/
│       ├── _variables.scss   # SCSS variables
│       ├── _mixins.scss      # SCSS mixins
│       └── main.scss         # Global styles entry point
├── components/
│   ├── base/                 # Base UI components (wrapping Vuetify)
│   └── [feature]/            # Feature components, grouped by feature
├── composables/              # Reusable Composition API logic
├── lib/                      # Low-level utilities (no business logic)
│   ├── axios.js              # Axios instance + interceptors
│   └── api/                  # HTTP layer: pure API calls
│       ├── cart.js
│       ├── product.js
│       └── order.js
├── services/                 # Business logic: API composition, data transformation, decisions
│   ├── cartService.js
│   ├── orderService.js
│   └── authService.js
├── stores/                   # Pinia stores: state management
│   ├── cartStore.js
│   └── orderStore.js
├── router/                   # Vue Router config
├── plugins/                  # Vue plugin setup (vuetify, veeValidate, etc.)
│   ├── vuetify.js
│   └── veeValidate.js
├── views/                    # Page-level components (mapped to routes)
└── docs/                     # Spec documents
    ├── TECH_SPEC.md          # This file
    └── spec-[feature].md     # Per-feature specs
```

---

## 6. Naming Conventions

| Target | Rule | Example |
|--------|------|---------|
| Component files | PascalCase | `CheckoutForm.vue` |
| Composable files | camelCase, prefixed with `use` | `useCartData.js` |
| Utility functions | camelCase | `formatPrice.js` |
| SCSS files | kebab-case, partials prefixed with `_` | `_card-styles.scss` |
| SCSS classes | BEM / kebab-case | `.cart-summary__item` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Pinia store files | camelCase | `cartStore.js` |
| Pinia store ID | camelCase | `useCartStore` |

---

## 7. Component Conventions

```vue
<!-- ✅ Standard SFC structure — always use <script setup> -->
<script setup>
// 1. imports
// 2. props / emits
// 3. stores / composables
// 4. reactive data (ref / reactive / computed)
// 5. methods
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
  <!-- Single root element -->
</template>

<style lang="scss" scoped>
// Component-scoped styles
</style>

// ✅ Always use <script setup> syntax
// ✅ Consider splitting into sub-components if over 200 lines
// ✅ Styles are scoped by default; global styles go in assets/styles/
```

---

## 8. API Conventions

Three layers with clear responsibilities:

```
Component / Pinia Store
        ↓
src/services/        ← Business logic (data composition, transformation, decisions)
        ↓
src/lib/api/         ← HTTP layer (pure API calls, no business logic)
        ↓
src/lib/axios.js     ← Axios config (interceptors, base URL)
```

---

### 8.1 Axios Setup

```javascript
// lib/axios.js
import axios from 'axios'

const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
})

// Request interceptor: auto-attach token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// Response interceptor: handle 401 / 500 globally
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // redirect to login
    }
    return Promise.reject(error)
  }
)

export default apiClient
```

---

### 8.2 API Layer (`src/lib/api/`)

**Responsibility: pure HTTP calls mapping to backend endpoints. No business logic.**

```javascript
// lib/api/cart.js
import apiClient from '@/lib/axios'

export const cartApi = {
  getCart:     ()               => apiClient.get('/cart'),
  addItem:     (payload)        => apiClient.post('/cart/items', payload),
  updateItem:  (itemId, payload)=> apiClient.put(`/cart/items/${itemId}`, payload),
  deleteItem:  (itemId)         => apiClient.delete(`/cart/items/${itemId}`),
  applyCoupon: (code)           => apiClient.post('/cart/coupon', { code }),
}
```

```javascript
// lib/api/product.js
import apiClient from '@/lib/axios'

export const productApi = {
  getList:  (params) => apiClient.get('/products', { params }),
  getById:  (id)     => apiClient.get(`/products/${id}`),
  getStock: (id)     => apiClient.get(`/products/${id}/stock`),
}
```

---

### 8.3 Service Layer (`src/services/`)

**Responsibility: business logic, data composition, transformation, and decisions. Calls one or more API modules and returns data that components/stores can use directly.**

Rules:
- One service file per business domain (`cart`, `order`, `auth`, etc.)
- Only call `lib/api/` modules inside services — never use `apiClient` directly
- Components and stores only call services, **never the API layer directly**
- Services `throw` errors; stores catch them in `try/catch`

```javascript
// services/cartService.js
import { cartApi }    from '@/lib/api/cart'
import { productApi } from '@/lib/api/product'

export const cartService = {

  // Fetch cart and enrich each item with live stock status (combines two APIs)
  async getCartWithStock() {
    const { data: cart } = await cartApi.getCart()

    const stockResults = await Promise.all(
      cart.items.map(item => productApi.getStock(item.productId))
    )

    return {
      ...cart,
      items: cart.items.map((item, i) => ({
        ...item,
        inStock: stockResults[i].data.quantity > 0,
      })),
    }
  },

  // Check stock before adding to cart; throw a business error if insufficient
  async addItem(productId, quantity) {
    const { data: stock } = await productApi.getStock(productId)

    if (stock.quantity < quantity) {
      throw new Error('Not enough stock to add this item to the cart')
    }

    return cartApi.addItem({ productId, quantity })
  },

  // Apply a coupon and return a formatted discount summary
  async applyCoupon(code) {
    const { data } = await cartApi.applyCoupon(code)
    return {
      discountRate:   data.discount_rate,
      discountAmount: data.original_total * data.discount_rate,
      finalTotal:     data.final_total,
    }
  },
}
```

---

### 8.4 Store Calls Service

Stores handle state only — all business logic is delegated to services:

```javascript
// stores/cartStore.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { cartService } from '@/services/cartService'

export const useCartStore = defineStore('cart', () => {
  const items     = ref([])
  const isLoading = ref(false)
  const error     = ref(null)

  const totalPrice = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  async function fetchCart() {
    isLoading.value = true
    error.value = null
    try {
      const cart = await cartService.getCartWithStock()  // ← call service, not API
      items.value = cart.items
    } catch (err) {
      error.value = err.message ?? 'Failed to load cart'
    } finally {
      isLoading.value = false
    }
  }

  async function addItem(productId, quantity) {
    isLoading.value = true
    error.value = null
    try {
      await cartService.addItem(productId, quantity)     // ← stock check is inside service
      await fetchCart()
    } catch (err) {
      error.value = err.message ?? 'Failed to add item'
    } finally {
      isLoading.value = false
    }
  }

  return { items, isLoading, error, totalPrice, fetchCart, addItem }
})
```

---

### 8.5 Error Format

```javascript
// Agreed error shape from the backend:
// {
//   code: 'CART_NOT_FOUND',     // business error code
//   message: 'Cart not found',  // user-readable message
//   details: ...                // optional
// }

// Global interceptors → handle 401 / 500
// Service layer       → handle business logic errors, throw to store
// Store               → catch and write to error state
// Component           → display store.error, never handles API errors directly
```

---

### 8.6 Layer Responsibilities

| Layer | Path | Responsibility | Can call |
|-------|------|----------------|----------|
| HTTP | `lib/api/` | Pure API calls, maps to endpoints | `lib/axios.js` |
| Service | `services/` | Business logic, data composition & transformation | `lib/api/` |
| Store | `stores/` | State management, triggers services | `services/` |
| Component | `components/` `views/` | UI rendering | `stores/` `composables/` |

> ⚠️ **No cross-layer calls**: components must not call `lib/api/` directly; stores must not call `lib/api/` directly; services must not import stores.

## 9. Pinia Store Conventions

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
      error.value = err.response?.data?.message ?? 'Failed to load'
    } finally {
      isLoading.value = false
    }
  }

  return { items, isLoading, error, totalPrice, fetchCart }
})

// ✅ Use Setup Store syntax (more flexible than Options Store)
// ✅ isLoading and error are required in every store
// ✅ Always use try/catch in actions; store errors in error state
```

---

## 10. Form Validation Conventions

> Form validation conventions have been extracted into a dedicated spec.
> See `docs/spec-form-validation.md` for the full reference.
> Covers: centralized rules management (`src/plugins/veeValidate.js`) and usage for all form components (Input, Textarea, Radio, Checkbox, Select, Autocomplete, File Upload).

---

## 11. Styling Conventions

```scss
// assets/styles/_variables.scss — centralized design tokens
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

// ✅ Use SCSS variables for colors and spacing — no hardcoded hex values
// ✅ Component styles are scoped by default
// ✅ Global / reset styles go in assets/styles/main.scss
// ✅ Vuetify theme colors are configured in plugins/vuetify.js
```

---

## 12. Default State Handling

All components that fetch async data must handle these states:

| State | Behavior |
|-------|----------|
| `isLoading` | Show `<v-skeleton-loader>` or `<v-progress-circular>`, matching the height of the actual content |
| `error` | Show error message + retry button |
| `empty` | Show empty state illustration + call-to-action text |
| success | Render actual content |

```vue
<template>
  <v-skeleton-loader v-if="store.isLoading" type="card" />

  <v-alert v-else-if="store.error" type="error" :text="store.error">
    <v-btn variant="text" @click="store.fetchData()">Retry</v-btn>
  </v-alert>

  <EmptyState v-else-if="!store.items.length" />

  <ActualContent v-else :items="store.items" />
</template>
```

---

## 13. Toast / Notification Conventions

```javascript
// composables/useToast.js — shared globally
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

// Mount <v-snackbar> in App.vue listening to snackbar state
// Rules:
// ✅ Successful action → success toast
// ✅ User validation error → inline form error-messages, not toast
// ✅ System error (API failure) → error toast
```

---

## 14. Default Responsive Breakpoints

Aligned with Vuetify's default breakpoints:

| Device | Breakpoint | Design width |
|--------|------------|--------------|
| Mobile | `< 600px` | 375px |
| Tablet | `600px ~ 959px` | 768px |
| Desktop | `≥ 960px` | 1440px |

```vue
<!-- ✅ Prefer Vuetify Grid System -->
<v-row>
  <v-col cols="12" md="8">Main content</v-col>
  <v-col cols="12" md="4">Sidebar</v-col>
</v-row>

<!-- ✅ For custom needs, use the SCSS respond-to() mixin -->
```

> Development priority: **Mobile First**

---

## 15. Environment Variables

```bash
# .env.local example (Vite project)
VITE_API_URL=https://api.example.com
VITE_APP_ENV=development
```

> Vite environment variables must be prefixed with `VITE_` to be accessible on the client side via `import.meta.env.VITE_API_URL`.
