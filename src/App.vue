<script setup lang="ts">
import { computed, ref } from 'vue'

type Mode = 'FREE' | 'PROMOTION'
const modes: { value: Mode; label: string }[] = [
  { value: 'FREE', label: 'Gratuito' },
  { value: 'PROMOTION', label: 'Promoción' },
]

const mode = ref<Mode>('PROMOTION')
const publishedMode = ref('FREE')
const publishedRevision = ref<number | ''>(1)
const noticeDate = ref('')
const effectiveDate = ref('')
const noticeEs = ref('')
const noticeEn = ref('')
const offlineDays = ref<number | ''>(30)
const message = ref('')
const copyDialog = ref<HTMLDialogElement>()
const jsonField = ref<HTMLTextAreaElement>()

const labels: Record<string, string> = {
  FREE: 'Acceso gratuito',
  PROMOTION: 'Promoción activa',
  PAID_REQUIRED: 'Licencia de pago requerida',
}
const minInterval = 30 * 24 * 60 * 60 * 1000
const toUtc = (date: string) => date ? `${date}T00:00:00Z` : ''
const error = computed(() => {
  const revision = publishedRevision.value
  if (typeof revision !== 'number' || revision < 1 || !Number.isSafeInteger(revision + 1)) {
    return 'Introduce una revisión publicada válida.'
  }
  const days = offlineDays.value
  if (typeof days !== 'number' || !Number.isInteger(days) || days < 1 || days > 30) {
    return 'Los días sin conexión deben estar entre 1 y 30.'
  }
  if (mode.value === 'FREE') return ''
  const notice = Date.parse(toUtc(noticeDate.value))
  const effective = Date.parse(toUtc(effectiveDate.value))
  if (!Number.isFinite(notice) || !Number.isFinite(effective)) {
    return 'Introduce las dos fechas para generar el JSON.'
  }
  return effective - notice < minInterval ? 'Deja al menos 30 días entre el aviso y la fecha anunciada.' : ''
})

const policyJson = computed(() => {
  if (error.value) return ''
  const promotion = mode.value === 'PROMOTION'
  return JSON.stringify({
    schema: 1,
    revision: Number(publishedRevision.value) + 1,
    mode: mode.value,
    noticeAtUtc: promotion ? toUtc(noticeDate.value) : null,
    effectiveAtUtc: promotion ? toUtc(effectiveDate.value) : null,
    offlineDays: offlineDays.value,
    noticeEs: promotion ? noticeEs.value : '',
    noticeEn: promotion ? noticeEn.value : '',
  }, null, 2)
})

const previewText = computed(() => {
  if (noticeEs.value.trim()) return noticeEs.value
  const date = new Date(toUtc(effectiveDate.value))
  return Number.isFinite(date.getTime())
    ? `Surkio sigue siendo gratis hasta el ${date.toLocaleDateString('es-ES', { dateStyle: 'long', timeZone: 'UTC' })}.`
    : 'Cuando indiques la fecha, aquí verás el aviso.'
})

function selectMode(next: Mode) {
  mode.value = next
  message.value = ''
}

async function preparePolicy() {
  if (!policyJson.value || !copyDialog.value) return
  message.value = 'Selecciona el JSON para copiarlo.'
  copyDialog.value.showModal()
  try {
    await navigator.clipboard.writeText(policyJson.value)
    message.value = 'JSON copiado al portapapeles.'
  } catch {
    jsonField.value?.focus()
    jsonField.value?.select()
  }
}
</script>

<template>
  <div class="shell">
    <aside class="sidebar">
      <a class="brand" href="#licencias" aria-label="Surkio, licencias">
        <img class="brand-icon" src="/surkio.svg" width="40" height="40" alt="">
        Surkio
      </a>
      <nav aria-label="Panel">
        <a class="nav-active" href="#licencias" aria-current="page"><span aria-hidden="true">◇</span> Licencias</a>
        <a href="https://console.firebase.google.com/project/surcos-dev/config"><span aria-hidden="true">◷</span> Historial en Firebase</a>
      </nav>
    </aside>

    <main id="licencias">
      <header class="page-header">
        <h1>Licencias</h1>
        <span class="environment"><i aria-hidden="true"></i> Entorno de pruebas</span>
      </header>

      <section class="card status-card" aria-label="Estado publicado">
        <h2>Estado publicado</h2>
        <div class="state">
          <span class="badge">{{ publishedMode }}</span>
          <span>{{ labels[publishedMode] }}</span>
        </div>
        <label class="revision">
          Revisión
          <input v-model.number="publishedRevision" type="number" min="1" step="1">
        </label>
        <details class="sync">
          <summary>Actualizar estado desde la consola</summary>
          <label>Modo publicado
            <select v-model="publishedMode">
              <option v-for="(label, value) in labels" :key="value" :value="value">{{ label }}</option>
            </select>
          </label>
          <p>Indica el modo y la revisión que ves en Remote Config. El panel usa esos datos para preparar la siguiente revisión.</p>
        </details>
      </section>

      <div class="columns">
        <form class="card editor" @submit.prevent="preparePolicy">
          <h2>Editar política</h2>
          <fieldset>
            <legend>Modo</legend>
            <div class="segments">
              <label v-for="option in modes" :key="option.value" :class="{ selected: mode === option.value }">
                <input type="radio" name="mode" :value="option.value" :checked="mode === option.value" @change="selectMode(option.value)">
                {{ option.label }}
              </label>
            </div>
          </fieldset>

          <template v-if="mode === 'PROMOTION'">
            <div class="dates">
              <label>Inicio del aviso <input v-model="noticeDate" type="date"></label>
              <label>Fecha anunciada de fin <input v-model="effectiveDate" type="date"></label>
            </div>
            <p class="hint">Fechas a las 00:00 UTC. El modo Promoción mantiene el acceso gratuito.</p>
            <label>Mensaje del aviso
              <textarea v-model="noticeEs" rows="3" maxlength="500" placeholder="Surkio sigue siendo gratis hasta…"></textarea>
            </label>
          </template>
          <p v-else class="hint">Con esta política, Surkio permite iniciar trabajos sin pedir licencia.</p>

          <details class="advanced">
            <summary>Opciones adicionales</summary>
            <label>Días sin conexión
              <input v-model.number="offlineDays" type="number" min="1" max="30" step="1">
            </label>
            <label v-if="mode === 'PROMOTION'">Aviso en inglés
              <textarea v-model="noticeEn" rows="2" maxlength="500"></textarea>
            </label>
          </details>
        </form>

        <section class="card preview">
          <h2>Vista previa del aviso</h2>
          <div v-if="mode === 'PROMOTION'" class="banner">
            <h3>Próximos cambios en la licencia</h3>
            <p>{{ previewText }}</p>
            <details>
              <summary>Ver detalles</summary>
              <p>El aviso informa del fin previsto de la promoción. En este modo puedes seguir utilizando Surkio gratis.</p>
            </details>
          </div>
          <div v-else class="banner">
            <h3>Acceso gratuito</h3>
            <p>Surkio permite iniciar nuevos trabajos sin mostrar el aviso de promoción.</p>
          </div>
        </section>
      </div>

      <footer class="actions">
        <button class="secondary" type="button" @click="selectMode('FREE')">Volver a gratuito</button>
        <div class="action-main">
          <p v-if="error" class="hint" role="status">{{ error }}</p>
          <button type="button" :disabled="!!error" @click="preparePolicy">Copiar JSON</button>
        </div>
      </footer>

      <dialog ref="copyDialog" aria-labelledby="dialog-title">
        <form method="dialog">
          <h2 id="dialog-title">Política lista para Firebase</h2>
          <p role="status">{{ message }}</p>
          <p>Pega el JSON en el valor predeterminado de <code>surcos_access_policy</code> y pulsa <strong>Publicar cambios</strong>.</p>
          <textarea ref="jsonField" :value="policyJson" aria-label="JSON generado" rows="12" readonly spellcheck="false"></textarea>
          <div class="dialog-actions">
            <a href="https://console.firebase.google.com/project/surcos-dev/config" target="_blank" rel="noopener noreferrer">Abrir Remote Config ↗</a>
            <button type="submit">Cerrar</button>
          </div>
        </form>
      </dialog>
    </main>
  </div>
</template>

<style>
:root {
  font: 16px/1.5 "Inter", "Segoe UI", system-ui, sans-serif;
  color: #172f25;
  background: #f5f6ef;
  font-synthesis: none;
  color-scheme: light;
}
*, *::before, *::after { box-sizing: border-box; }
body { display: block; min-width: 320px; margin: 0; }
#app { width: 100%; max-width: none; margin: 0; padding: 0; text-align: left; }
.shell { min-height: 100vh; display: grid; grid-template-columns: minmax(185px, 18%) minmax(0, 1fr); }
.sidebar { padding: 30px 16px; background: #123b2b; color: #f5fff6; }
.brand { display: flex; align-items: center; gap: 12px; margin: 0 8px 40px; font-size: 30px; font-weight: 750; color: inherit; text-decoration: none; }
.brand-icon { flex: 0 0 40px; width: 40px; height: 40px; }
nav { display: grid; gap: 8px; }
nav a { display: flex; align-items: center; gap: 10px; padding: 11px 12px; border-radius: 9px; color: inherit; text-decoration: none; }
nav a:hover, nav a.nav-active { background: #3a6e52; }
nav a span { font-size: 22px; line-height: 1; }
main { width: 100%; max-width: 1320px; margin: 0 auto; padding: clamp(20px, 3vw, 42px); }
.page-header, .actions, .dialog-actions { display: flex; align-items: center; justify-content: space-between; gap: 16px; }
.page-header { margin-bottom: 24px; }
h1, h2, h3, p { margin: 0; }
h1 { font-size: clamp(2rem, 3.8vw, 3.2rem); line-height: 1.15; letter-spacing: -.045em; }
h2 { font-size: clamp(1.25rem, 2.3vw, 1.8rem); letter-spacing: -.025em; line-height: 1.2; }
.environment { color: #355a46; font-size: .88rem; font-weight: 650; display: flex; align-items: center; gap: 9px; }
.environment i { width: 9px; height: 9px; background: #3e8b52; border-radius: 50%; }
.card { min-width: 0; padding: clamp(16px, 2vw, 25px); background: #fff; border: 1px solid #a3b8a7; border-radius: 14px; }
.status-card { display: flex; flex-wrap: wrap; align-items: center; gap: 10px 20px; margin-bottom: 22px; }
.status-card h2 { font-size: 1.13rem; }
.state { display: flex; align-items: center; gap: 10px; padding-left: 16px; border-left: 1px solid #d5dfd5; }
.badge { padding: 4px 10px; background: #267346; color: #fff; border-radius: 7px; font-size: .83rem; font-weight: 750; }
.revision { margin-left: auto; display: flex; align-items: center; gap: 8px; }
.revision input { width: 78px; }
.sync { flex-basis: 100%; font-size: .85rem; }
.sync label { display: block; max-width: 270px; margin: 8px 0; }
.columns { display: grid; grid-template-columns: minmax(0, 1.28fr) minmax(0, 1fr); gap: 18px; }
.editor, .preview { display: grid; align-content: start; gap: 17px; }
fieldset { min-width: 0; border: 0; padding: 0; margin: 0; }
legend, label { font-weight: 650; }
legend { margin-bottom: 5px; }
.segments { display: grid; grid-template-columns: 1fr 1fr; border: 1px solid #91aa96; border-radius: 9px; overflow: hidden; }
.segments label { cursor: pointer; display: flex; align-items: center; justify-content: center; gap: 6px; padding: 7px 4px; }
.segments .selected { color: #fff; background: #356b4c; }
input, textarea, select, button { font: inherit; }
.segments input { width: auto; margin: 0; accent-color: #174630; }
input:not([type="radio"]), select, textarea { width: 100%; min-width: 0; color: #172f25; background: #fbfcfa; border: 1px solid #92a999; border-radius: 8px; padding: 8px 10px; }
label > input:not([type="radio"]), label > textarea, label > select { display: block; margin-top: 5px; }
.revision input { width: 78px; margin-top: 0 !important; }
textarea { resize: vertical; font-weight: 400; }
.dates { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
summary { color: #315d44; cursor: pointer; font-weight: 650; }
.advanced label { display: block; margin-top: 10px; }
.advanced input { max-width: 130px; }
.hint { color: #4a6053; font-size: .85rem; }
.banner { min-height: 175px; padding: clamp(17px, 2vw, 24px); background: #e7f2dd; border-radius: 13px; overflow-wrap: anywhere; }
.banner h3 { font-size: clamp(1.3rem, 2.5vw, 1.85rem); line-height: 1.16; letter-spacing: -.025em; }
.banner p { margin: 12px 0 16px; }
.banner summary { width: fit-content; display: block; padding: 7px 12px; border: 1px solid #315d44; border-radius: 8px; }
.actions { align-items: flex-start; margin-top: 24px; }
.action-main { display: grid; justify-items: end; gap: 8px; text-align: right; }
button { padding: 10px 18px; border-radius: 9px; border: 1px solid #123f2c; color: #fff; background: #123f2c; font-weight: 700; cursor: pointer; }
button:hover:not(:disabled) { background: #285d43; color: #fff; }
button.secondary { background: transparent; color: #123f2c; }
button:disabled { opacity: .53; cursor: default; }
a:focus-visible, button:focus-visible, summary:focus-visible, input:focus-visible, select:focus-visible, textarea:focus-visible { outline: 3px solid #58a36b; outline-offset: 3px; }
dialog { width: min(94vw, 640px); max-height: 90vh; padding: 22px; border: 1px solid #a3b8a7; border-radius: 14px; color: #172f25; background: #fff; }
dialog::backdrop { background: #142920a0; }
dialog p, dialog textarea, .dialog-actions { margin-top: 12px; }
dialog textarea { font: .9rem/1.45 ui-monospace, monospace; }
.dialog-actions a { color: #21573b; font-weight: 700; }
@media (max-width: 940px) { .shell { grid-template-columns: 158px minmax(0, 1fr); } .sidebar { padding-inline: 9px; } .dates { grid-template-columns: 1fr; } }
@media (max-width: 720px) {
  .shell { display: block; }
  .sidebar { padding: 10px 16px; display: flex; flex-wrap: wrap; align-items: center; justify-content: space-between; gap: 9px; }
  .brand { margin: 0; font-size: 22px; }
  nav { display: flex; font-size: .83rem; }
  nav a { padding: 6px 8px; gap: 4px; }
  nav a span { font-size: 17px; }
  .columns { grid-template-columns: 1fr; }
  .dates { grid-template-columns: 1fr 1fr; }
}
@media (max-width: 440px) { .dates { grid-template-columns: 1fr; } .actions { flex-wrap: wrap; } .action-main { justify-items: start; text-align: left; } }
</style>
