<template>
  <div v-if="canAct && normalized.length" class="action-dock">
    <button
      v-for="item in normalized"
      :key="item.key"
      :data-testid="`action-${item.key}`"
      class="action-btn"
      :class="{ selected: selectionMode === item.action }"
      :disabled="busy || !isClickable(item)"
      @click="onClick(item)"
    >
      {{ text(item) }}
    </button>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from "vue";
import type { ActionRequest, ActionType, AvailableAction } from "@/types/game";

type SelectionMode = "kai" | "peng" | "chi" | null;
type PanelAction = {
  key: string;
  action: ActionType;
  enabled: boolean;
  deferred?: boolean;
  deferredKind?: "pass";
  candidates: AvailableAction["candidates"];
};

const props = withDefaults(
  defineProps<{
    actions: AvailableAction[];
    canAct?: boolean;
    responsePhase?: string;
    selectionMode?: SelectionMode;
  }>(),
  {
    canAct: false,
    responsePhase: "",
    selectionMode: null,
  },
);

const emit = defineEmits<{
  submit: [request: ActionRequest];
  selectionChange: [payload: { mode: SelectionMode; selectedCandidateId: string | null }];
}>();

const busy = ref(false);
const defaultOrder: ActionType[] = ["hu", "kai", "peng", "chi", "pass"];
const selectionMode = computed<SelectionMode>(() => props.selectionMode ?? null);

const normalized = computed<PanelAction[]>(() => {
  const map = new Map(props.actions.map((x) => [x.action, x]));
  const isCollective = props.responsePhase === "collective";
  const pass = map.get("pass");
  const isPendingForMe = isCollective && Boolean(pass?.enabled && pass.deferred);
  const ordered: PanelAction[] = defaultOrder
    .map((action) => {
      const item = map.get(action);
      return {
        key: action,
        action,
        enabled: Boolean(item?.enabled),
        deferred: Boolean(item?.deferred),
        candidates: item?.candidates ?? [],
      };
    })
    .filter((item) => !(item.action === "chi" && isCollective && !item.deferred))
    .filter((item) => !(item.action === "pass" && isPendingForMe));

  if (isPendingForMe) {
    ordered.push({
      key: "deferred-pass",
      action: "pass",
      enabled: false,
      deferred: true,
      deferredKind: "pass",
      candidates: [],
    });
  }
  return ordered.filter((item) => isClickable(item));
});

function isMeldAction(action: ActionType): action is Exclude<SelectionMode, null> {
  return action === "kai" || action === "peng" || action === "chi";
}

function isClickable(item: { enabled: boolean; deferred?: boolean }): boolean {
  return item.enabled || Boolean(item.deferred);
}

function actionText(action: ActionType): string {
  if (action === "pass" && props.responsePhase === "local_upper") {
    return "抓";
  }
  const map: Record<ActionType, string> = {
    hu: "胡",
    kai: "开",
    peng: "碰",
    chi: "吃",
    pass: "过",
  };
  return map[action];
}

function text(item: PanelAction): string {
  if (item.deferredKind === "pass") {
    return "抓";
  }
  return actionText(item.action);
}

function pulseUnlock(): void {
  window.setTimeout(() => {
    busy.value = false;
  }, 220);
}

function onClick(item: PanelAction): void {
  if (busy.value) {
    return;
  }
  if (item.deferredKind === "pass") {
    busy.value = true;
    emit("selectionChange", { mode: null, selectedCandidateId: null });
    emit("submit", { action: "pass", deferred: true });
    pulseUnlock();
    return;
  }

  if (isMeldAction(item.action)) {
    if ((item.candidates?.length ?? 0) === 1) {
      busy.value = true;
      const candidateId = item.candidates?.[0]?.id ?? "";
      emit("selectionChange", { mode: null, selectedCandidateId: candidateId || null });
      emit("submit", { action: item.action, candidateId });
      pulseUnlock();
      return;
    }
    emit("selectionChange", {
      mode: selectionMode.value === item.action ? null : item.action,
      selectedCandidateId: null,
    });
    return;
  }

  busy.value = true;
  emit("selectionChange", { mode: null, selectedCandidateId: null });
  emit("submit", item.action);
  pulseUnlock();
}
</script>

<style scoped>
.action-dock {
  pointer-events: auto;
  min-width: clamp(5.7rem, 8vw, 7.4rem);
  display: flex;
  justify-content: center;
  gap: clamp(0.45rem, 1vw, 0.8rem);
  padding: clamp(0.55rem, 1.15vh, 0.8rem) clamp(0.7rem, 1.5vw, 1.05rem);
  border: 2px solid rgba(20, 18, 15, 0.84);
  border-radius: 0.82rem;
  background:
    linear-gradient(180deg, rgba(255, 246, 224, 0.9), rgba(179, 165, 139, 0.86)),
    repeating-linear-gradient(90deg, rgba(30, 25, 18, 0.08) 0 3px, transparent 3px 9px);
  box-shadow: 0 0.8rem 1.6rem rgba(0, 0, 0, 0.4), inset 0 0 0 2px rgba(255, 255, 255, 0.32);
}

.action-btn {
  min-width: clamp(3.2rem, 6vw, 5.6rem);
  min-height: clamp(2.7rem, 5.8vh, 3.45rem);
  border: 2px solid #1d1711;
  border-radius: 0.65rem;
  color: #f8efe0;
  background:
    linear-gradient(180deg, #2f6fa1, #173e61 52%, #0f2b45),
    radial-gradient(circle at 30% 20%, rgba(255, 255, 255, 0.24), transparent 36%);
  font-family: "KaiTi", "STKaiti", "DFKai-SB", serif;
  font-size: clamp(1.25rem, 2.7vh, 1.8rem);
  font-weight: 900;
  cursor: pointer;
  text-shadow: 0 2px 0 rgba(0, 0, 0, 0.55);
  box-shadow: 0 0.34rem 0 #0a1827, 0 0.65rem 1rem rgba(0, 0, 0, 0.34);
  transition: transform 0.16s ease, filter 0.16s ease;
}

.action-btn:hover:not(:disabled) {
  transform: translateY(-0.18rem);
  filter: brightness(1.12);
}

.action-btn:active:not(:disabled) {
  transform: translateY(0.12rem);
  box-shadow: 0 0.14rem 0 #0a1827, 0 0.35rem 0.7rem rgba(0, 0, 0, 0.3);
}

.action-btn.selected {
  background:
    linear-gradient(180deg, #c1842f, #89511c 52%, #5c3416),
    radial-gradient(circle at 30% 20%, rgba(255, 255, 255, 0.24), transparent 36%);
}

.action-btn:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}
</style>
