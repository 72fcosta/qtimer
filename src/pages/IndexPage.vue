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
            thickness="0.2"
            color="primary"
            track-color="grey-4"
            show-value
          >
            <div class="text-h3 text-weight-bolder">{{ formattedTime }}</div>
          </q-circular-progress>
          <div class="text-caption text-grey-7">
            Duração configurada: {{ durationMinutes }} min
          </div>
          <div class="text-caption text-primary">
            {{ statusLabel }}
          </div>
        </q-card-section>

        <q-separator />

        <q-card-section class="q-gutter-md">
          <q-input
            v-model.number="durationMinutes"
            type="number"
            label="Duração do ciclo (minutos)"
            outlined
            dense
            :disable="isRunning"
            :rules="durationRules"
            @blur="normalizeDuration"
          >
            <template #prepend>
              <q-icon name="schedule" />
            </template>
            <template #append>
              <q-btn
                flat
                dense
                icon="restart_alt"
                :disable="isRunning || durationMinutes === defaultMinutes"
                @click="resetDuration"
              >
                <q-tooltip>Voltar para {{ defaultMinutes }} min</q-tooltip>
              </q-btn>
            </template>
          </q-input>

          <div class="row no-wrap items-center q-gutter-sm">
            <q-chip
              v-for="quick in quickDurations"
              :key="quick"
              clickable
              outline
              color="primary"
              :disable="isRunning"
              @click="applyQuickDuration(quick)"
            >
              {{ quick }} min
            </q-chip>
          </div>
        </q-card-section>

        <q-separator />

        <q-card-actions align="center" class="q-gutter-sm">
          <q-btn
            color="primary"
            label="Iniciar"
            unelevated
            icon="play_arrow"
            :disable="isRunning || durationMinutes < 1"
            @click="startTimer"
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
        </q-card-actions>

        <q-card-section class="q-gutter-sm">
          <q-banner v-if="!hasNotificationSupport" class="bg-grey-3 text-grey-9">
            <template #avatar>
              <q-icon name="notifications_off" color="grey-8" />
            </template>
            O navegador não suporta notificações do sistema. Usaremos apenas o alerta do app.
          </q-banner>

          <q-banner v-else-if="needsNotificationPermission" class="bg-grey-2 text-grey-9" inline-actions>
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
    </div>
  </q-page>
</template>

<script setup lang="ts">
import { useQuasar } from 'quasar';
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue';

const defaultMinutes = 30;
const durationMinutes = ref(defaultMinutes);
const quickDurations = [15, 25, 30, 45, 60];
const isRunning = ref(false);
const hasFinished = ref(false);
const remainingMs = ref(defaultMinutes * 60 * 1000);
const cycleEndAt = ref<number | null>(null);
const timerId = ref<number | null>(null);
const permissionStatus = ref<NotificationPermission>('default');
const $q = useQuasar();

const durationRules = [
  (val: number) => (val && val > 0) || 'Use pelo menos 1 minuto',
];

const totalMs = computed(() => Math.max(1, durationMinutes.value * 60 * 1000));

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
    return 'Contando ciclo em andamento';
  }
  if (hasFinished.value) {
    return 'Ciclo concluído. Ajuste e inicie o próximo quando quiser.';
  }
  return 'Pronto para iniciar um novo ciclo';
});

const hasNotificationSupport = computed(() => typeof Notification !== 'undefined');
const needsNotificationPermission = computed(
  () => hasNotificationSupport.value && permissionStatus.value !== 'granted'
);

watch(
  durationMinutes,
  (val, oldVal) => {
    const previousTotal = Math.max(1, oldVal * 60 * 1000);
    if (!isRunning.value && remainingMs.value === previousTotal) {
      remainingMs.value = Math.max(1, val * 60 * 1000);
    }
  },
  { flush: 'sync' }
);

onMounted(() => {
  if (hasNotificationSupport.value) {
    permissionStatus.value = Notification.permission;
  }
});

onBeforeUnmount(stopTimer);

function normalizeDuration() {
  if (!durationMinutes.value || durationMinutes.value < 1) {
    durationMinutes.value = 1;
  }
}

function resetDuration() {
  durationMinutes.value = defaultMinutes;
  if (!isRunning.value) {
    remainingMs.value = defaultMinutes * 60 * 1000;
    hasFinished.value = false;
  }
}

function applyQuickDuration(minutes: number) {
  durationMinutes.value = minutes;
  if (!isRunning.value) {
    remainingMs.value = minutes * 60 * 1000;
    hasFinished.value = false;
  }
}

function startTimer() {
  if (isRunning.value) {
    return;
  }

  normalizeDuration();

  if (remainingMs.value <= 0 || hasFinished.value) {
    remainingMs.value = totalMs.value;
    hasFinished.value = false;
  }

  isRunning.value = true;
  const now = Date.now();
  cycleEndAt.value = now + remainingMs.value;
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
  remainingMs.value = totalMs.value;
  hasFinished.value = false;
}

function tick() {
  if (!cycleEndAt.value) {
    return;
  }

  const msLeft = cycleEndAt.value - Date.now();
  remainingMs.value = Math.max(0, msLeft);

  if (msLeft <= 0) {
    finishCycle();
  }
}

function updateRemainingFromNow() {
  if (!cycleEndAt.value) {
    return;
  }
  const msLeft = Math.max(0, cycleEndAt.value - Date.now());
  remainingMs.value = msLeft;
}

function stopTimer() {
  isRunning.value = false;
  cycleEndAt.value = null;
  if (timerId.value !== null) {
    clearInterval(timerId.value);
    timerId.value = null;
  }
}

function finishCycle() {
  stopTimer();
  remainingMs.value = 0;
  hasFinished.value = true;
  void sendReminder();
}

async function sendReminder() {
  $q.notify({
    type: 'positive',
    message: 'Ciclo encerrado! Hora do lembrete.',
    caption: 'Você pode iniciar um novo ciclo quando quiser.',
    timeout: 4000,
  });

  if (!hasNotificationSupport.value) {
    return;
  }

  const permission =
    permissionStatus.value === 'default'
      ? await Notification.requestPermission()
      : permissionStatus.value;

  permissionStatus.value = permission;

  if (permission === 'granted') {
    new Notification('Tempo encerrado', {
      body: 'Seu ciclo configurado terminou. Hora do próximo passo!',
      tag: 'qtimer-cycle',
    });
  }
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
</script>
