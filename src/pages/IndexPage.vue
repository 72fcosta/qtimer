<template>
  <q-page class="q-pa-lg bg-grey-1">
    <div class="column items-center full-width q-gutter-lg">
      <q-card class="full-width" flat bordered>
        <q-card-section class="text-center">
          <div class="text-h5 text-weight-bold q-mb-xs">Timer de Ciclo</div>
          <div class="text-body2 text-grey-7">
            Ajuste a duração do ciclo, inicie o contador e receba um lembrete ao final.
          </div>
        </q-card-section>

        <q-separator />

        <q-card-section class="column items-center q-gutter-md">
          <q-circular-progress
            :value="progressValue"
            size="220px"
            :thickness="0.2"
            color="primary"
            track-color="grey-4"
            show-value
          >
            <div class="text-h3 text-weight-bolder">{{ formattedTime }}</div>
          </q-circular-progress>
          <div class="text-caption text-grey-7">
            Ciclo: {{ cycleMinutes }} min · Descanso: {{ restMinutes }} min
          </div>
          <div class="text-caption text-primary">
            {{ statusLabel }}
          </div>
        </q-card-section>

        <q-separator />

        <q-card-section class="q-gutter-md">
          <q-input
            v-model.number="cycleMinutes"
            type="number"
            label="Duração do ciclo (minutos)"
            outlined
            dense
            :disable="isRunning"
            :rules="durationRules"
            @blur="normalizeCycleDuration"
          >
            <template #prepend>
              <q-icon name="schedule" />
            </template>
            <template #append>
              <q-btn
                flat
                dense
                icon="restart_alt"
                :disable="isRunning || cycleMinutes === defaultCycleMinutes"
                @click="resetCycleDuration"
              >
                <q-tooltip>Voltar para {{ defaultCycleMinutes }} min</q-tooltip>
              </q-btn>
            </template>
          </q-input>

          <div class="row no-wrap items-center q-gutter-sm">
            <q-chip
              v-for="quick in quickCycleDurations"
              :key="quick"
              clickable
              outline
              color="primary"
              :disable="isRunning"
              @click="applyQuickCycleDuration(quick)"
            >
              {{ quick }} min
            </q-chip>
          </div>

          <q-input
            v-model.number="restMinutes"
            type="number"
            label="Duração do descanso (minutos)"
            outlined
            dense
            :disable="isRunning"
            :rules="durationRules"
            @blur="normalizeRestDuration"
          >
            <template #prepend>
              <q-icon name="self_improvement" />
            </template>
            <template #append>
              <q-btn
                flat
                dense
                icon="restart_alt"
                :disable="isRunning || restMinutes === defaultRestMinutes"
                @click="resetRestDuration"
              >
                <q-tooltip>Voltar para {{ defaultRestMinutes }} min</q-tooltip>
              </q-btn>
            </template>
          </q-input>

          <div class="row no-wrap items-center q-gutter-sm">
            <q-chip
              v-for="quick in quickRestDurations"
              :key="quick"
              clickable
              outline
              color="secondary"
              :disable="isRunning"
              @click="applyQuickRestDuration(quick)"
            >
              {{ quick }} min
            </q-chip>
          </div>
        </q-card-section>

        <q-separator />

        <q-card-actions align="center" class="q-gutter-sm">
          <q-btn
            color="primary"
            label="Iniciar ciclo"
            unelevated
            icon="play_arrow"
            :disable="isRunning || cycleMinutes < 1"
            @click="startCycleTimer"
          />
          <q-btn
            color="secondary"
            outline
            label="Pausar"
            icon="pause"
            :disable="!isRunning"
            @click="pauseTimer"
          />
          <q-btn
            flat
            color="primary"
            label="Resetar"
            icon="restart_alt"
            :disable="isRunning && remainingMs > 0"
            @click="resetTimer"
          />
          <q-btn
            dense
            flat
            color="dark"
            icon="widgets"
            :label="showWidget ? 'Ocultar widget' : 'Mostrar widget'"
            @click="toggleWidget"
          />
        </q-card-actions>

        <q-card-section class="q-gutter-sm">
          <q-banner v-if="!hasNotificationSupport" class="bg-grey-3 text-grey-9">
            <template #avatar>
              <q-icon name="notifications_off" color="grey-8" />
            </template>
            O navegador não suporta notificações do sistema. Usaremos apenas o alerta do app.
          </q-banner>

          <q-banner
            v-else-if="needsNotificationPermission"
            class="bg-grey-2 text-grey-9"
            inline-actions
          >
            <template #avatar>
              <q-icon name="notifications_active" color="primary" />
            </template>
            Ative notificações do sistema para receber o lembrete mesmo fora do app.
            <template #action>
              <q-btn
                flat
                dense
                color="primary"
                label="Permitir"
                @click="requestNotificationPermission"
              />
            </template>
          </q-banner>

      <q-banner v-else class="bg-positive text-white">
        <template #avatar>
          <q-icon name="notifications" />
        </template>
        Notificações ativas. Você receberá um lembrete ao fim de cada ciclo.
          </q-banner>
        </q-card-section>
      </q-card>
      <div v-if="showWidget" class="widget-panel" :style="widgetStyle" @mousedown.stop>
        <q-card flat bordered class="bg-white shadow-4">
          <q-card-section
            class="row items-center justify-between widget-drag-handle text-grey-7"
            @mousedown.prevent.stop="startDragging"
          >
            <div class="row items-center q-gutter-xs">
              <q-icon name="open_with" size="16px" />
              <span class="text-caption">Widget</span>
            </div>
            <q-btn flat dense round icon="close" size="sm" @click.stop="toggleWidget" />
          </q-card-section>
          <q-separator />
          <q-card-section class="q-pa-md">
            <div class="text-overline text-primary">{{ phaseLabel }}</div>
            <div class="text-h5 text-weight-bold">{{ formattedTime }}</div>
            <div class="text-caption text-grey-7">Estado: {{ widgetStateLabel }}</div>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script setup lang="ts">
import { useQuasar } from 'quasar';
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue';

type Phase = 'cycle' | 'rest';

const defaultCycleMinutes = 30;
const defaultRestMinutes = 5;
const cycleMinutes = ref(defaultCycleMinutes);
const restMinutes = ref(defaultRestMinutes);
const quickCycleDurations = [15, 25, 30, 45, 60];
const quickRestDurations = [5, 10, 15];
const phase = ref<Phase>('cycle');
const lastCompletedPhase = ref<Phase | null>(null);
const isRunning = ref(false);
const remainingMs = ref(defaultCycleMinutes * 60 * 1000);
const phaseEndAt = ref<number | null>(null);
const timerId = ref<number | null>(null);
const permissionStatus = ref<NotificationPermission>('default');
const reminderSuggestions = [
  'Levante-se por 2 minutos',
  'Alongue pescoço e ombros',
  'Beba um copo de água',
  'Olhe para longe para descansar os olhos',
  'Faça 10 respirações profundas',
  'Dê uma volta rápida pela casa',
];
const $q = useQuasar();
const showWidget = ref(false);
const widgetPosition = reactive({ x: 24, y: 24 });
const dragState = reactive({ active: false, offsetX: 0, offsetY: 0 });

const durationRules = [(val: number) => (val && val > 0) || 'Use pelo menos 1 minuto'];

const activeMinutes = computed(() =>
  phase.value === 'cycle' ? cycleMinutes.value : restMinutes.value,
);

const totalMs = computed(() => Math.max(1, activeMinutes.value * 60 * 1000));

const widgetStyle = computed(() => ({
  transform: `translate(${widgetPosition.x}px, ${widgetPosition.y}px)`,
}));

const progressValue = computed(() => {
  const remaining = Math.min(remainingMs.value, totalMs.value);
  const consumed = totalMs.value - remaining;
  const ratio = (consumed / totalMs.value) * 100;
  return Math.min(100, Math.max(0, ratio));
});

const formattedTime = computed(() => {
  const totalSeconds = Math.max(0, Math.round(remainingMs.value / 1000));
  const minutes = String(Math.floor(totalSeconds / 60)).padStart(2, '0');
  const seconds = String(totalSeconds % 60).padStart(2, '0');
  return `${minutes}:${seconds}`;
});

const statusLabel = computed(() => {
  if (isRunning.value) {
    return phase.value === 'cycle' ? 'Ciclo em andamento' : 'Descanso em andamento';
  }
  if (lastCompletedPhase.value === 'rest') {
    return 'Descanso concluído. Inicie um novo ciclo quando quiser.';
  }
  if (lastCompletedPhase.value === 'cycle') {
    return 'Ciclo concluído. Descanso iniciado automaticamente.';
  }
  return 'Pronto para iniciar um novo ciclo';
});

const widgetStateLabel = computed(() => {
  if (phase.value === 'rest' && isRunning.value) {
    return 'Descansando';
  }
  if (isRunning.value) {
    return 'Rodando';
  }
  return 'Pausado';
});

const phaseLabel = computed(() => (phase.value === 'cycle' ? 'Ciclo' : 'Descanso'));

const hasNotificationSupport = computed(() => typeof Notification !== 'undefined');
const needsNotificationPermission = computed(
  () => hasNotificationSupport.value && permissionStatus.value !== 'granted',
);

watch(
  cycleMinutes,
  (val, oldVal) => {
    const previousTotal = Math.max(1, oldVal * 60 * 1000);
    if (!isRunning.value && phase.value === 'cycle' && remainingMs.value === previousTotal) {
      remainingMs.value = Math.max(1, val * 60 * 1000);
      lastCompletedPhase.value = null;
    }
  },
  { flush: 'sync' },
);

watch(
  restMinutes,
  (val, oldVal) => {
    const previousTotal = Math.max(1, oldVal * 60 * 1000);
    if (!isRunning.value && phase.value === 'rest' && remainingMs.value === previousTotal) {
      remainingMs.value = Math.max(1, val * 60 * 1000);
      lastCompletedPhase.value = null;
    }
  },
  { flush: 'sync' },
);

onMounted(() => {
  if (hasNotificationSupport.value) {
    permissionStatus.value = Notification.permission;
  }

  window.addEventListener('mousemove', handleDrag);
  window.addEventListener('mouseup', stopDragging);
  setInitialWidgetPosition();
});

onBeforeUnmount(() => {
  stopTimer();
  window.removeEventListener('mousemove', handleDrag);
  window.removeEventListener('mouseup', stopDragging);
});

function normalizeCycleDuration() {
  if (!cycleMinutes.value || cycleMinutes.value < 1) {
    cycleMinutes.value = 1;
  }
}

function normalizeRestDuration() {
  if (!restMinutes.value || restMinutes.value < 1) {
    restMinutes.value = 1;
  }
}

function resetCycleDuration() {
  cycleMinutes.value = defaultCycleMinutes;
  if (!isRunning.value && phase.value === 'cycle') {
    remainingMs.value = defaultCycleMinutes * 60 * 1000;
    lastCompletedPhase.value = null;
  }
}

function resetRestDuration() {
  restMinutes.value = defaultRestMinutes;
  if (!isRunning.value && phase.value === 'rest') {
    remainingMs.value = defaultRestMinutes * 60 * 1000;
    lastCompletedPhase.value = null;
  }
}

function applyQuickCycleDuration(minutes: number) {
  cycleMinutes.value = minutes;
  if (!isRunning.value && phase.value === 'cycle') {
    remainingMs.value = minutes * 60 * 1000;
    lastCompletedPhase.value = null;
  }
}

function applyQuickRestDuration(minutes: number) {
  restMinutes.value = minutes;
  if (!isRunning.value && phase.value === 'rest') {
    remainingMs.value = minutes * 60 * 1000;
    lastCompletedPhase.value = null;
  }
}

function startCycleTimer() {
  if (isRunning.value && phase.value === 'cycle') {
    return;
  }

  const previousPhase = phase.value;
  phase.value = 'cycle';
  normalizeCycleDuration();

  const total = Math.max(1, cycleMinutes.value * 60 * 1000);
  if (previousPhase !== 'cycle' || remainingMs.value <= 0) {
    remainingMs.value = total;
  }

  lastCompletedPhase.value = null;
  isRunning.value = true;
  const now = Date.now();
  phaseEndAt.value = now + remainingMs.value;
  timerId.value = window.setInterval(tick, 200);
}

function pauseTimer() {
  if (!isRunning.value) {
    return;
  }
  updateRemainingFromNow();
  stopTimer();
}

function resetTimer() {
  stopTimer();
  phase.value = 'cycle';
  normalizeCycleDuration();
  remainingMs.value = Math.max(1, cycleMinutes.value * 60 * 1000);
  lastCompletedPhase.value = null;
}

function toggleWidget() {
  showWidget.value = !showWidget.value;
  if (showWidget.value) {
    setInitialWidgetPosition();
  }
}

function tick() {
  if (!phaseEndAt.value) {
    return;
  }

  const msLeft = phaseEndAt.value - Date.now();
  remainingMs.value = Math.max(0, msLeft);

  if (msLeft <= 0) {
    finishPhase();
  }
}

function updateRemainingFromNow() {
  if (!phaseEndAt.value) {
    return;
  }
  const msLeft = Math.max(0, phaseEndAt.value - Date.now());
  remainingMs.value = msLeft;
}

function stopTimer() {
  isRunning.value = false;
  phaseEndAt.value = null;
  if (timerId.value !== null) {
    clearInterval(timerId.value);
    timerId.value = null;
  }
}

function finishPhase() {
  stopTimer();
  remainingMs.value = 0;
  lastCompletedPhase.value = phase.value;

  if (phase.value === 'cycle') {
    void sendReminderThenStartRest();
  } else {
    handleRestFinished();
  }
}

async function sendReminderThenStartRest() {
  const suggestion = pickSuggestion();

  $q.notify({
    type: 'positive',
    message: 'Ciclo encerrado!',
    caption: `Sugestão rápida: ${suggestion}`,
    timeout: 6000,
    position: 'top-right',
    progress: true,
  });

  startRestTimer();

  if (hasNotificationSupport.value) {
    const permission =
      permissionStatus.value === 'default'
        ? await Notification.requestPermission()
        : permissionStatus.value;

    permissionStatus.value = permission;

    if (permission === 'granted') {
      new Notification('Tempo encerrado', {
        body: `Seu ciclo terminou. ${suggestion}`,
        tag: 'qtimer-cycle',
        silent: true,
      });
    }
  }
}

function startRestTimer() {
  if (isRunning.value && phase.value === 'rest') {
    return;
  }

  const previousPhase = phase.value;
  phase.value = 'rest';
  normalizeRestDuration();

  const total = Math.max(1, restMinutes.value * 60 * 1000);
  if (previousPhase !== 'rest' || remainingMs.value <= 0) {
    remainingMs.value = total;
  }

  isRunning.value = true;
  const now = Date.now();
  phaseEndAt.value = now + remainingMs.value;
  timerId.value = window.setInterval(tick, 200);
}

function handleRestFinished() {
  $q.notify({
    type: 'info',
    message: 'Descanso concluído',
    caption: 'Pronto para iniciar um novo ciclo.',
    timeout: 5000,
    position: 'top-right',
    progress: true,
  });

  phase.value = 'cycle';
  normalizeCycleDuration();
  remainingMs.value = Math.max(1, cycleMinutes.value * 60 * 1000);
}

function startDragging(event: MouseEvent) {
  if (!showWidget.value) {
    return;
  }
  dragState.active = true;
  dragState.offsetX = event.clientX - widgetPosition.x;
  dragState.offsetY = event.clientY - widgetPosition.y;
}

function handleDrag(event: MouseEvent) {
  if (!dragState.active) {
    return;
  }
  const newX = event.clientX - dragState.offsetX;
  const newY = event.clientY - dragState.offsetY;
  clampWidgetPosition(newX, newY);
}

function stopDragging() {
  dragState.active = false;
}

function clampWidgetPosition(x: number, y: number) {
  const padding = 8;
  const maxX = Math.max(padding, window.innerWidth - 240 - padding);
  const maxY = Math.max(padding, window.innerHeight - 160 - padding);
  widgetPosition.x = Math.min(Math.max(padding, x), maxX);
  widgetPosition.y = Math.min(Math.max(padding, y), maxY);
}

function setInitialWidgetPosition() {
  const padding = 16;
  const maxX = Math.max(padding, window.innerWidth - 240 - padding);
  const maxY = Math.max(padding, window.innerHeight - 180 - padding);
  widgetPosition.x = maxX;
  widgetPosition.y = maxY;
}

async function requestNotificationPermission() {
  if (!hasNotificationSupport.value) {
    $q.notify({
      type: 'warning',
      message: 'Seu navegador não suporta notificações do sistema.',
    });
    return;
  }

  const permission = await Notification.requestPermission();
  permissionStatus.value = permission;

  if (permission === 'granted') {
    $q.notify({
      type: 'positive',
      message: 'Notificações ativadas!',
      caption: 'Você receberá lembretes ao fim de cada ciclo.',
    });
  } else if (permission === 'denied') {
    $q.notify({
      type: 'warning',
      message: 'Notificações bloqueadas.',
      caption: 'Habilite-as no navegador para receber lembretes.',
    });
  }
}

function pickSuggestion() {
  const index = Math.floor(Math.random() * reminderSuggestions.length);
  return reminderSuggestions[index] || reminderSuggestions[0];
}
</script>

<style scoped>
.widget-panel {
  position: fixed;
  z-index: 2000;
  width: 220px;
  user-select: none;
}

.widget-drag-handle {
  cursor: move;
}
</style>
