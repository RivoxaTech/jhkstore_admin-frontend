<script setup>
import { computed, reactive, ref } from 'vue'
import { RouterLink } from 'vue-router'
import { ShieldCheck } from '@lucide/vue'
import { api } from '@/api/client.js'
import { useAuthStore } from '@/stores/auth.js'
import { useUiStore } from '@/stores/ui.js'
import PasswordInput from '@/components/PasswordInput.vue'

// The signed-in person's own page: who they are, and the change of their password (the current one is asked for).
const auth = useAuthStore()
const ui = useUiStore()
const form = reactive({ current_password: '', password: '', password_confirmation: '' })
const errors = ref({})
const message = ref('')
const busy = ref(false)
const role = computed(() => ({ developer: 'Developer', admin: 'Admin' })[auth.role] || auth.role)
const err = (k) => errors.value[k]?.[0] || ''

// the rules are the server's (it says how many characters it needs: 12 on a live server), so the page never accepts what the server will refuse
const min = computed(() => auth.admin?.password_min || 8)
const rules = computed(() => [
  { id: 'length', text: `At least ${min.value} characters`, ok: form.password.length >= min.value },
  { id: 'username', text: 'Not your username', ok: form.password !== '' && form.password.toLowerCase() !== String(auth.admin?.username || '').toLowerCase() },
  { id: 'old', text: 'Different from your current password', ok: form.password !== '' && form.password !== form.current_password },
  { id: 'same', text: 'The two new passwords are the same', ok: form.password !== '' && form.password === form.password_confirmation },
])
const ready = computed(() => !!form.current_password && rules.value.every((r) => r.ok))

async function save() {
  errors.value = {}
  message.value = ''
  if (!form.current_password) errors.value.current_password = ['Enter your current password.']
  if (form.password.length < min.value) errors.value.password = [`The new password needs at least ${min.value} characters.`]
  else if (form.password.toLowerCase() === String(auth.admin?.username || '').toLowerCase()) errors.value.password = ['The password cannot be your username.']
  else if (form.password === form.current_password) errors.value.password = ['Choose a password you have not used just now.']
  else if (form.password !== form.password_confirmation) errors.value.password_confirmation = ['The two new passwords are not the same.']
  if (Object.keys(errors.value).length) return
  busy.value = true
  try {
    await api.put('admin/auth/password', { ...form })
    form.current_password = form.password = form.password_confirmation = ''
    message.value = 'Your password was changed. Use it the next time you sign in.'
    ui.toast('Password changed.')
  } catch (e) {
    if (e.status === 422) errors.value = e.errors
    else if (e.status === 429) errors.value = { current_password: ['Too many tries. Wait a minute and try again.'] }
    else errors.value = { current_password: [e.message] }
  } finally {
    busy.value = false
  }
}
</script>

<template>
  <div class="page-head">
    <div>
      <h1>Profile</h1>
      <div class="crumbs"><RouterLink :to="{ name: 'dashboard' }">Dashboard</RouterLink> › <span>Profile</span></div>
    </div>
  </div>

  <div class="profile-grid">
    <section class="card">
      <div class="card__body profile-me">
        <span class="avatar avatar--xl">{{ auth.initials }}</span>
        <div>
          <h2>{{ auth.admin?.name || auth.admin?.username }}</h2>
          <p class="muted">@{{ auth.admin?.username }} · {{ role }}</p>
          <p v-if="auth.admin?.email" class="muted">{{ auth.admin.email }}</p>
        </div>
      </div>
    </section>

    <section class="card">
      <div class="card__head"><h2><ShieldCheck :size="15" /> Change password</h2></div>
      <form class="card__body profile-form" novalidate data-testid="password-form" @submit.prevent="save">
        <div v-if="message" class="alert alert--success" role="status" data-testid="password-done">{{ message }}</div>
        <div class="field">
          <label for="pw-current">Current password</label>
          <PasswordInput id="pw-current" v-model="form.current_password" autocomplete="current-password" :invalid="!!err('current_password')" />
          <p v-if="err('current_password')" class="err">{{ err('current_password') }}</p>
        </div>
        <div class="field">
          <label for="pw-new">New password</label>
          <PasswordInput id="pw-new" v-model="form.password" autocomplete="new-password" :invalid="!!err('password')" />
          <p v-if="err('password')" class="err">{{ err('password') }}</p>
        </div>
        <div class="field">
          <label for="pw-again">New password again</label>
          <PasswordInput id="pw-again" v-model="form.password_confirmation" autocomplete="new-password" :invalid="!!err('password_confirmation')" />
          <p v-if="err('password_confirmation')" class="err">{{ err('password_confirmation') }}</p>
        </div>
        <ul class="rules" aria-label="Password rules" data-testid="password-rules">
          <li v-for="r in rules" :key="r.id" :class="{ 'is-ok': r.ok }" :data-rule="r.id" :data-ok="r.ok"><span class="rules__dot" aria-hidden="true">{{ r.ok ? '✓' : '' }}</span>{{ r.text }}</li>
        </ul>
        <div class="profile-actions"><button class="btn btn-primary" type="submit" :disabled="busy || !ready" data-testid="password-save"><span v-if="busy" class="spinner" /> Save new password</button></div>
      </form>
    </section>
  </div>
</template>

<style scoped>
.profile-grid { display: grid; gap: 12px; grid-template-columns: minmax(0, 1fr) minmax(0, 1.4fr); align-items: start; max-width: 980px; }
.profile-me { display: flex; align-items: center; gap: 16px; }
.profile-me h2 { font-size: 17px; }
.avatar--xl { width: 64px; height: 64px; font-size: 22px; }
.profile-form { display: grid; gap: 16px; max-width: 440px; }
.profile-form .field { display: grid; gap: 6px; }
.profile-form .field label { font-weight: 600; font-size: 13px; }
.rules { list-style: none; display: grid; gap: 6px; padding: 12px 14px; background: var(--muted-soft); border-radius: 10px; font-size: 13px; color: var(--text-3); }
.rules li { display: flex; align-items: center; gap: 10px; }
.rules__dot { flex: none; width: 18px; height: 18px; border-radius: 50%; border: 1.5px solid var(--border, #cbd5e1); display: grid; place-items: center; font-size: 11px; font-weight: 700; line-height: 1; }
.rules li.is-ok { color: var(--text); }
.rules li.is-ok .rules__dot { background: var(--success, #16a34a); border-color: var(--success, #16a34a); color: #fff; }
.profile-actions .btn { width: 100%; justify-content: center; }
@media (max-width: 900px) { .profile-grid { grid-template-columns: minmax(0, 1fr); } }
</style>
