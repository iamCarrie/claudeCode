# Form Validation Spec

> **Related:** `docs/TECH_SPEC.md` — global tech conventions
> **Shared plugin:** `src/plugins/veeValidate.js` — global vee-validate setup + rule registration
> **Stack:** vee-validate v4 + @vee-validate/rules + Vuetify

---

## 1. Overview

```
src/plugins/veeValidate.js
  ├── configure() — global settings
  ├── defineRule() — registers all rules globally via @vee-validate/rules
  └── defineRule() — registers any custom rules (phone, etc.)
            ↓ (imported once in main.js)
Form component  ← AppForm wraps <Form> at the outermost layer
            ↓
Field components ← accept :rules prop as string or object, use useField() internally
```

**Core principles:**
- `AppForm` (wrapping vee-validate's `<Form>`) is always placed at the **outermost layer** of the form
- Rules are registered globally in `veeValidate.js` via `defineRule()` — never re-imported per component
- Each field component accepts a `:rules` prop — passed in as a **string** (`"required|min:2"`) or **object** (`{ required: true, min: 2 }`) from the parent
- Each field component uses `useField()` internally to connect to the parent form context
- Error messages are passed directly to Vuetify's `:error-messages` prop

---

## 2. Shared Plugin — `src/plugins/veeValidate.js`

Registers all rules globally. Import once in `main.js` — all forms across the app have access automatically.

```javascript
// src/plugins/veeValidate.js
import { defineRule, configure } from 'vee-validate'
import { all } from '@vee-validate/rules'
import { localize, setLocale } from '@vee-validate/i18n'
import en from '@vee-validate/i18n/dist/locale/en.json'

// ─── Register all @vee-validate/rules ─────────────────────────
// This registers: required, email, min, max, min_value, max_value,
// numeric, regex, confirmed, size, mimes, image, url, between, etc.
Object.entries(all).forEach(([name, rule]) => {
  defineRule(name, rule)
})

// ─── Custom rules (not in @vee-validate/rules) ────────────────
// Validator must return true (pass) or a string (error message)
defineRule('phone', (value) => {
  if (!value || !value.length) return true   // empty → pass (use required separately)
  if (!/^09\d{8}$/.test(value)) return 'Please enter a valid phone number (09xxxxxxxx)'
  return true
})

// ─── Global configuration ──────────────────────────────────────
configure({
  generateMessage: localize({ en }),   // use @vee-validate/i18n for built-in rule messages
  validateOnInput: true,               // validate on every keystroke, not just on blur
})

setLocale('en')
```

**Install dependencies:**

```bash
pnpm add vee-validate @vee-validate/rules @vee-validate/i18n
```

**Register in `main.js`:**

```javascript
// main.js
import '@/plugins/veeValidate'   // must be imported before app.mount()

const app = createApp(App)
app.mount('#app')
```

---

## 3. How to Pass Rules from Parent

Rules are always passed from the parent form as a **string** or **object**. Both formats are valid.

```vue
<!-- String format (Laravel-style) — recommended for simple rules -->
<AppInput name="email" label="Email" rules="required|email" />
<AppInput name="name"  label="Name"  rules="required|min:2|max:50" />
<AppInput name="phone" label="Phone" rules="required|phone" />
<AppInput name="age"   label="Age"   rules="required|numeric|min_value:18|max_value:99" />

<!-- Object format — recommended for dynamic rules or readability -->
<AppInput name="email" label="Email" :rules="{ required: true, email: true }" />
<AppInput name="name"  label="Name"  :rules="{ required: true, min: 2, max: 50 }" />
<AppInput name="file"  label="File"  :rules="{ required: true, size: 5120, mimes: ['image/jpeg', 'image/png'] }" />

<!-- Cross-field: confirmed uses @ prefix to reference another field -->
<AppInput name="password"         label="Password"         rules="required|min:8" />
<AppInput name="password_confirm" label="Confirm Password" rules="required|confirmed:@password" />
```

**Available built-in rules (from `@vee-validate/rules`):**

| Rule | String usage | Object usage |
|------|-------------|--------------|
| required | `required` | `{ required: true }` |
| email | `email` | `{ email: true }` |
| min (length) | `min:2` | `{ min: 2 }` |
| max (length) | `max:50` | `{ max: 50 }` |
| min_value | `min_value:0` | `{ min_value: 0 }` |
| max_value | `max_value:100` | `{ max_value: 100 }` |
| numeric | `numeric` | `{ numeric: true }` |
| integer | `integer` | `{ integer: true }` |
| between | `between:1,10` | `{ between: [1, 10] }` |
| confirmed | `confirmed:@password` | — |
| size (KB) | `size:5120` | `{ size: 5120 }` |
| mimes | `mimes:image/jpeg,image/png` | `{ mimes: ['image/jpeg'] }` |
| image | `image` | `{ image: true }` |
| ext | `ext:jpg,png` | `{ ext: ['jpg', 'png'] }` |
| url | `url` | `{ url: true }` |
| regex | — | `{ regex: /^[A-Z]+$/ }` |
| **phone** (custom) | `phone` | `{ phone: true }` |

---

## 4. Component Specs

All field components share this `:rules` prop:

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `name` | String | required | Field name key, connects to parent form context |
| `label` | String | `''` | Display label |
| `rules` | String / Object | `''` | Rules in string `"required|min:2"` or object `{ required: true }` format |
| `disabled` | Boolean | `false` | Disables the field |

---

### 4.1 Input / Textarea

**File:** `src/components/base/AppInput.vue`

```vue
<script setup>
import { useField } from 'vee-validate'

const props = defineProps({
  name:        { type: String,           required: true },
  label:       { type: String,           default: '' },
  rules:       { type: [String, Object], default: '' },   // string or object format
  type:        { type: String,           default: 'text' },
  placeholder: { type: String,           default: '' },
  multiline:   { type: Boolean,          default: false }, // true → renders <v-textarea>
  rows:        { type: Number,           default: 4 },
  disabled:    { type: Boolean,          default: false },
})

const { value, errorMessage } = useField(() => props.name, props.rules)
</script>

<template>
  <v-textarea
    v-if="multiline"
    v-model="value"
    :label="label"
    :rows="rows"
    :placeholder="placeholder"
    :disabled="disabled"
    :error-messages="errorMessage"
    variant="outlined"
  />
  <v-text-field
    v-else
    v-model="value"
    :label="label"
    :type="type"
    :placeholder="placeholder"
    :disabled="disabled"
    :error-messages="errorMessage"
    variant="outlined"
  />
</template>
```

**Usage:**

```vue
<AppInput name="username" label="Username" rules="required|min:2|max:20" />
<AppInput name="email"    label="Email"    rules="required|email" type="email" />
<AppInput name="password" label="Password" rules="required|min:8" type="password" />
<AppInput name="bio"      label="Bio"      rules="max:500" :multiline="true" :rows="5" />
```

---

### 4.2 Radio

**File:** `src/components/base/AppRadio.vue`

```vue
<script setup>
import { useField } from 'vee-validate'

const props = defineProps({
  name:    { type: String,           required: true },
  label:   { type: String,           default: '' },
  rules:   { type: [String, Object], default: '' },
  options: { type: Array,            required: true },   // [{ label, value }]
  inline:  { type: Boolean,          default: false },
})

const { value, errorMessage } = useField(() => props.name, props.rules)
</script>

<template>
  <v-radio-group
    v-model="value"
    :label="label"
    :inline="inline"
    :error-messages="errorMessage"
  >
    <v-radio
      v-for="option in options"
      :key="option.value"
      :label="option.label"
      :value="option.value"
    />
  </v-radio-group>
</template>
```

**Usage:**

```vue
<AppRadio
  name="gender"
  label="Gender"
  rules="required"
  :options="[
    { label: 'Male',   value: 'male' },
    { label: 'Female', value: 'female' },
  ]"
  :inline="true"
/>
```

---

### 4.3 Checkbox

**File:** `src/components/base/AppCheckbox.vue`

Supports **single** (boolean toggle) and **multiple** (array of values) modes.

```vue
<script setup>
import { useField } from 'vee-validate'

const props = defineProps({
  name:       { type: String,           required: true },
  label:      { type: String,           default: '' },
  rules:      { type: [String, Object], default: '' },
  options:    { type: Array,            default: null }, // null → single; array → multiple
  trueValue:  { default: true },
  falseValue: { default: false },
})

const { value, errorMessage } = useField(() => props.name, props.rules)
</script>

<template>
  <!-- Multiple checkboxes -->
  <div v-if="options">
    <p class="text-subtitle-2 mb-1">{{ label }}</p>
    <v-checkbox
      v-for="option in options"
      :key="option.value"
      v-model="value"
      :label="option.label"
      :value="option.value"
      hide-details
    />
    <p v-if="errorMessage" class="text-error text-caption mt-1">{{ errorMessage }}</p>
  </div>

  <!-- Single checkbox -->
  <v-checkbox
    v-else
    v-model="value"
    :label="label"
    :true-value="trueValue"
    :false-value="falseValue"
    :error-messages="errorMessage"
  />
</template>
```

**Usage:**

```vue
<!-- Single: use is rule to enforce true -->
<AppCheckbox
  name="agreeTerms"
  label="I agree to the Terms & Conditions"
  :rules="{ required: true, is: true }"
/>

<!-- Multiple -->
<AppCheckbox
  name="interests"
  label="Interests"
  rules="required"
  :options="[
    { label: 'Design',      value: 'design' },
    { label: 'Engineering', value: 'engineering' },
  ]"
/>
```

---

### 4.4 Select

**File:** `src/components/base/AppSelect.vue`

```vue
<script setup>
import { useField } from 'vee-validate'

const props = defineProps({
  name:        { type: String,           required: true },
  label:       { type: String,           default: '' },
  rules:       { type: [String, Object], default: '' },
  items:       { type: Array,            required: true }, // [{ title, value }] or strings
  placeholder: { type: String,           default: 'Please select' },
  multiple:    { type: Boolean,          default: false },
  clearable:   { type: Boolean,          default: false },
  disabled:    { type: Boolean,          default: false },
})

const { value, errorMessage } = useField(() => props.name, props.rules)
</script>

<template>
  <v-select
    v-model="value"
    :label="label"
    :items="items"
    :placeholder="placeholder"
    :multiple="multiple"
    :clearable="clearable"
    :disabled="disabled"
    :error-messages="errorMessage"
    variant="outlined"
  />
</template>
```

**Usage:**

```vue
<AppSelect
  name="country"
  label="Country"
  rules="required"
  :items="[
    { title: 'Taiwan', value: 'TW' },
    { title: 'Japan',  value: 'JP' },
  ]"
/>

<!-- Multi-select -->
<AppSelect
  name="tags"
  label="Tags"
  rules="required"
  :items="tagOptions"
  :multiple="true"
  :clearable="true"
/>
```

---

### 4.5 Autocomplete

**File:** `src/components/base/AppAutocomplete.vue`

```vue
<script setup>
import { useField } from 'vee-validate'

const props = defineProps({
  name:        { type: String,           required: true },
  label:       { type: String,           default: '' },
  rules:       { type: [String, Object], default: '' },
  items:       { type: Array,            required: true },
  placeholder: { type: String,           default: 'Type to search' },
  multiple:    { type: Boolean,          default: false },
  clearable:   { type: Boolean,          default: true },
  loading:     { type: Boolean,          default: false },
  disabled:    { type: Boolean,          default: false },
})

const emit = defineEmits(['search'])

const { value, errorMessage } = useField(() => props.name, props.rules)
</script>

<template>
  <v-autocomplete
    v-model="value"
    :label="label"
    :items="items"
    :placeholder="placeholder"
    :multiple="multiple"
    :clearable="clearable"
    :loading="loading"
    :disabled="disabled"
    :error-messages="errorMessage"
    variant="outlined"
    @update:search="emit('search', $event)"
  />
</template>
```

**Usage:**

```vue
<AppAutocomplete
  name="city"
  label="City"
  rules="required"
  :items="cityList"
/>

<!-- Async search -->
<AppAutocomplete
  name="user"
  label="Assign to"
  rules="required"
  :items="searchResults"
  :loading="isSearching"
  @search="handleSearch"
/>
```

---

### 4.6 File Upload

**File:** `src/components/base/AppFileUpload.vue`

```vue
<script setup>
import { useField } from 'vee-validate'

const props = defineProps({
  name:     { type: String,           required: true },
  label:    { type: String,           default: 'Upload File' },
  rules:    { type: [String, Object], default: '' },
  accept:   { type: String,           default: '*' },
  multiple: { type: Boolean,          default: false },
  disabled: { type: Boolean,          default: false },
})

const { value, errorMessage } = useField(() => props.name, props.rules)
</script>

<template>
  <v-file-input
    v-model="value"
    :label="label"
    :accept="accept"
    :multiple="multiple"
    :disabled="disabled"
    :error-messages="errorMessage"
    variant="outlined"
    prepend-icon="mdi-paperclip"
  />
</template>
```

**Usage:**

```vue
<!-- size unit is KB: 5120 = 5MB -->
<AppFileUpload
  name="avatar"
  label="Profile Photo"
  accept="image/*"
  rules="required|image|size:5120"
/>

<!-- specific file types via ext -->
<AppFileUpload
  name="document"
  label="Document"
  accept=".pdf,.docx"
  :rules="{ required: true, ext: ['pdf', 'docx'], size: 20480 }"
/>
```

---

## 5. Complete Form Example

> Prefer using `AppForm` to wrap your form (see Section 7).
> Only fall back to `useForm()` directly when you need full control (multi-step forms, dynamic fields, conditional validation, etc.).

```vue
<script setup>
// With unplugin-vue-components, no component imports are needed.
// All App* components are auto-imported from src/components/.

const initialValues = {
  name: '', email: '', phone: '', password: '', password_confirm: '',
  gender: null, interests: [], country: null, city: null,
  avatar: null, agreeTerms: false,
}

async function handleSubmit(values, { resetForm, setErrors }) {
  try {
    await submitApi(values)
    resetForm()
  } catch (err) {
    // Write API errors back to fields
    if (err.code === 'EMAIL_TAKEN') setErrors({ email: 'This email is already taken' })
  }
}
</script>

<template>
  <AppForm :initial-values="initialValues" @submit="handleSubmit">
    <template #default="{ isSubmitting, resetForm }">
      <AppInput name="name"             label="Full Name"        rules="required|min:2|max:50" />
      <AppInput name="email"            label="Email"            rules="required|email" type="email" />
      <AppInput name="phone"            label="Phone"            rules="required|phone" />
      <AppInput name="password"         label="Password"         rules="required|min:8" type="password" />
      <AppInput name="password_confirm" label="Confirm Password" rules="required|confirmed:@password" type="password" />
      <AppInput name="bio"              label="Bio"              rules="max:500" :multiline="true" />

      <AppRadio
        name="gender" label="Gender" rules="required" :inline="true"
        :options="[{ label: 'Male', value: 'male' }, { label: 'Female', value: 'female' }]"
      />
      <AppCheckbox
        name="interests" label="Interests" rules="required"
        :options="[{ label: 'Design', value: 'design' }, { label: 'Engineering', value: 'engineering' }]"
      />
      <AppCheckbox
        name="agreeTerms" label="I agree to the Terms & Conditions"
        :rules="{ required: true, is: true }"
      />

      <AppSelect       name="country" label="Country" rules="required" :items="countryList" />
      <AppAutocomplete name="city"    label="City"    rules="required" :items="cityList" />

      <AppFileUpload name="avatar" label="Profile Photo" accept="image/*" rules="required|image|size:5120" />

      <v-btn type="submit" color="primary" :loading="isSubmitting">Submit</v-btn>
      <v-btn variant="text" @click="resetForm()">Reset</v-btn>
    </template>
  </AppForm>
</template>
```

---

## 6. Custom Rule Pattern

When you need a rule not available in `@vee-validate/rules`, add it to `veeValidate.js`.

The validator function must return `true` (pass) or a `string` (error message):

```javascript
// src/plugins/veeValidate.js

defineRule('phone', (value) => {
  if (!value || !value.length) return true   // empty → pass (combine with required separately)
  if (!/^09\d{8}$/.test(value)) return 'Please enter a valid phone number (09xxxxxxxx)'
  return true
})

// Rule with arguments — destructure from the second parameter array
defineRule('minAge', (value, [minAge]) => {
  if (!value) return true
  const age = new Date().getFullYear() - new Date(value).getFullYear()
  if (age < minAge) return `Must be at least ${minAge} years old`
  return true
})
```

**Usage:**

```vue
<AppInput name="phone"    rules="required|phone" />
<AppInput name="birthday" rules="required|minAge:18" type="date" />
```

---

## 7. AppForm — Wrapper Component

**File:** `src/components/base/AppForm.vue`

Wraps vee-validate's `<Form>` component so each page no longer needs to write `useForm()`, `handleSubmit`, or `resetForm` boilerplate.

### How it works

vee-validate's `<Form>` component has `useForm()` built in under the hood. It exposes form state via slot props. `AppForm` wraps it and provides a unified interface for:
- `@submit` / `@invalid` events
- `isSubmitting` loading state via slot
- API error writeback via `setErrors`

```vue
<!-- src/components/base/AppForm.vue -->
<script setup>
import { Form } from 'vee-validate'

const props = defineProps({
  validationSchema: { type: Object, default: undefined }, // optional schema object
  initialValues:    { type: Object, default: undefined }, // optional initial field values
  id:               { type: String, default: undefined }, // optional form id
})

const emit = defineEmits([
  'submit',  // fires when validation passes — payload: (values, { resetForm, setErrors })
  'invalid', // fires when validation fails (optional use)
])

function onSubmit(values, actions) {
  emit('submit', values, actions)
}

function onInvalid(errors) {
  emit('invalid', errors)
}
</script>

<template>
  <Form
    v-slot="{ isSubmitting, resetForm, setErrors, meta }"
    :validation-schema="validationSchema"
    :initial-values="initialValues"
    @submit="onSubmit"
    @invalid-submit="({ errors }) => onInvalid(errors)"
  >
    <slot
      :is-submitting="isSubmitting"
      :reset-form="resetForm"
      :set-errors="setErrors"
      :meta="meta"
    />
  </Form>
</template>
```

### Usage

```vue
<!-- ✅ Basic usage — no useForm() needed -->
<script setup>
async function handleSubmit(values, { resetForm, setErrors }) {
  try {
    await submitApi(values)
    resetForm()
  } catch (err) {
    // Write API errors back to specific fields
    setErrors({ email: 'This email is already in use' })
  }
}
</script>

<template>
  <AppForm @submit="handleSubmit">
    <template #default="{ isSubmitting, resetForm }">
      <AppInput name="name"  label="Name"  rules="required|min:2" />
      <AppInput name="email" label="Email" rules="required|email" type="email" />
      <AppInput name="phone" label="Phone" rules="required|phone" />

      <v-btn type="submit" color="primary" :loading="isSubmitting">Submit</v-btn>
      <v-btn variant="text" @click="resetForm()">Reset</v-btn>
    </template>
  </AppForm>
</template>
```

```vue
<!-- ✅ With initialValues — for edit pages -->
<AppForm :initial-values="user" @submit="handleUpdate">
  <template #default="{ isSubmitting }">
    <AppInput name="name"  label="Name"  rules="required" />
    <AppInput name="email" label="Email" rules="required|email" />
    <v-btn type="submit" :loading="isSubmitting">Update</v-btn>
  </template>
</AppForm>
```

```vue
<!-- ✅ API error writeback -->
<AppForm @submit="handleSubmit">
  <template #default="{ isSubmitting }">
    <AppInput name="email" label="Email" rules="required|email" />
    <v-btn type="submit" :loading="isSubmitting">Log in</v-btn>
  </template>
</AppForm>

<script setup>
async function handleSubmit(values, { setErrors }) {
  const res = await loginApi(values)
  if (res.error?.code === 'EMAIL_NOT_FOUND') {
    setErrors({ email: 'No account found with this email' })
  }
}
</script>
```

### AppForm vs useForm()

| | `AppForm` | `useForm()` |
|--|-----------|-------------|
| Needs to import useForm | ❌ No | ✅ Yes |
| Needs to write handleSubmit | ❌ No | ✅ Yes |
| Access isSubmitting | ✅ via slot props | ✅ via destructure |
| Access setErrors | ✅ via slot props | ✅ via destructure |
| Best for | Standard create / edit forms | Complex forms needing full control |

> ⚠️ For complex form logic (multi-step, dynamic fields, conditional validation), use `useForm()` directly for full control.
