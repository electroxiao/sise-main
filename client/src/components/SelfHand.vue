<template>
  <section class="self-hand-shell">
    <div class="self-portrait" :class="{ active: isActive, dealer: isDealer }">
      <span>{{ portraitText }}</span>
    </div>
    <div class="self-play">
      <div class="self-melds" :class="{ empty: !groups.length }">
        <div v-for="group in groups" :key="group.id" class="self-meld">
          <em v-if="group.badge">{{ group.badge }}</em>
          <FourColorCard v-for="card in group.cards" :key="card.id" :card="card" size="sm" />
        </div>
      </div>
      <div class="hand-row">
        <button
          v-for="(card, index) in privateHand"
          :key="card.id"
          :data-testid="`hand-card-${card.id}`"
          class="hand-button"
          :style="{ '--tilt': `${(index % 5) - 2}deg` }"
          :disabled="!canDiscardCard(card)"
          @click="emit('discardCard', card.id)"
        >
          <span v-if="candidateBadgeText(card.id)" class="candidate-badge">{{ candidateBadgeText(card.id) }}</span>
          <FourColorCard
            :card="card"
            size="xl"
            :playable="canDiscardCard(card)"
            :disabled="canDiscard && !canDiscardCard(card)"
            :selected="isSelectedCandidateCard(card.id)"
          />
        </button>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
import { computed } from "vue";
import type { ActionCandidate, Card, PlayerState } from "@/types/game";
import FourColorCard from "./FourColorCard.vue";
import type { VisibleGroupBlock } from "./PlayerSeat.vue";

const props = withDefaults(
  defineProps<{
    player?: PlayerState | null;
    privateHand: Card[];
    groups: VisibleGroupBlock[];
    canDiscard?: boolean;
    isActive?: boolean;
    isDealer?: boolean;
    activeCandidates?: ActionCandidate[];
    selectedCandidateId?: string | null;
  }>(),
  {
    player: null,
    canDiscard: false,
    isActive: false,
    isDealer: false,
    activeCandidates: () => [],
    selectedCandidateId: null,
  },
);

const emit = defineEmits<{
  discardCard: [cardId: string];
}>();

const portraitText = computed(() => (props.player?.name ?? "自己").replace(/\s*\[BOT\]\s*$/, "").slice(0, 2) || "己");
const selectedCandidate = computed(
  () => props.activeCandidates.find((candidate) => candidate.id === props.selectedCandidateId) ?? null,
);

function canDiscardCard(card: Card): boolean {
  return Boolean(props.canDiscard) && card.type !== "jiang" && card.color !== "gold";
}

function candidateBadgeText(cardId: string): string {
  const indexes = props.activeCandidates
    .map((candidate, index) => (candidate.cardIds.includes(cardId) ? index + 1 : 0))
    .filter(Boolean);
  return indexes.length ? indexes.join("/") : "";
}

function isSelectedCandidateCard(cardId: string): boolean {
  return Boolean(selectedCandidate.value?.cardIds.includes(cardId));
}
</script>

<style scoped>
.self-hand-shell {
  position: absolute;
  left: clamp(0.8rem, 2vw, 1.6rem);
  right: clamp(0.8rem, 2vw, 1.6rem);
  bottom: clamp(0.45rem, 1.2vh, 0.95rem);
  z-index: 8;
  display: grid;
  grid-template-columns: clamp(5rem, 10vw, 8rem) minmax(0, 1fr);
  align-items: end;
  gap: clamp(0.65rem, 1.2vw, 1rem);
}

.self-portrait {
  position: relative;
  aspect-ratio: 0.9;
  border: 3px solid #16110e;
  border-radius: 0.55rem;
  display: grid;
  place-items: end center;
  color: #142516;
  background:
    radial-gradient(circle at 50% 30%, #f0c7a9 0 22%, transparent 23%),
    linear-gradient(180deg, #e9f4e5 0%, #bcd9ad 56%, #5a8b57 57%);
  box-shadow: 0 0.5rem 0 #3c2c1b, 0 0.9rem 1.5rem rgba(0, 0, 0, 0.42);
  overflow: hidden;
}

.self-portrait::before {
  content: "";
  position: absolute;
  left: 29%;
  top: 15%;
  width: 42%;
  height: 26%;
  border-radius: 44% 44% 35% 35%;
  background: #1e1b17;
}

.self-portrait span {
  position: relative;
  z-index: 1;
  width: 100%;
  min-height: 1.9rem;
  display: grid;
  place-items: center;
  border-top: 2px solid rgba(22, 17, 14, 0.48);
  background: rgba(255, 246, 226, 0.94);
  font-weight: 900;
}

.self-portrait.active {
  box-shadow:
    0 0.5rem 0 #3c2c1b,
    0 0.9rem 1.5rem rgba(0, 0, 0, 0.42),
    0 0 0 4px rgba(94, 211, 117, 0.76);
}

.self-portrait.dealer::after {
  content: "庄";
  position: absolute;
  right: 0.28rem;
  top: 0.28rem;
  width: 1.7rem;
  height: 1.7rem;
  border-radius: 999px;
  display: grid;
  place-items: center;
  color: #ffe8a3;
  background: #c93622;
  font-weight: 900;
  box-shadow: 0 0 0 2px #2d160f;
}

.self-play {
  min-width: 0;
  display: grid;
  gap: 0.5rem;
}

.self-melds {
  min-height: 3.8rem;
  display: flex;
  align-items: end;
  gap: 0.5rem;
  padding: 0.18rem 0.65rem;
  border: 0;
  border-radius: 0;
  background: transparent;
  overflow-x: auto;
}

.self-melds.empty {
  pointer-events: none;
}

.self-meld {
  position: relative;
  flex: 0 0 auto;
  display: flex;
  align-items: end;
  gap: 0.14rem;
  padding: 0.25rem 0.32rem;
  border-radius: 0.42rem;
  background: rgba(255, 255, 255, 0.1);
}

.self-meld em,
.candidate-badge {
  position: absolute;
  z-index: 2;
  min-width: 1.4rem;
  min-height: 1.4rem;
  border-radius: 999px;
  display: grid;
  place-items: center;
  background: #1f2933;
  color: #fff3cc;
  font-size: 0.68rem;
  font-style: normal;
  font-weight: 900;
}

.self-meld em {
  left: -0.38rem;
  top: -0.38rem;
}

.hand-row {
  min-height: clamp(5.8rem, 11vw, 7.5rem);
  display: flex;
  align-items: end;
  gap: clamp(0.12rem, 0.45vw, 0.32rem);
  padding: 0.75rem 0.8rem 0.35rem;
  border: 2px solid rgba(20, 18, 15, 0.72);
  border-radius: 0.75rem;
  background:
    linear-gradient(180deg, rgba(52, 55, 51, 0.72), rgba(25, 26, 24, 0.82)),
    repeating-linear-gradient(90deg, rgba(255, 255, 255, 0.04) 0 2px, transparent 2px 7px);
  overflow-x: auto;
  box-shadow: inset 0 0 0 2px rgba(255, 255, 255, 0.04), 0 0.7rem 1.3rem rgba(0, 0, 0, 0.25);
}

.hand-button {
  position: relative;
  flex: 0 0 auto;
  border: 0;
  padding: 0;
  background: transparent;
}

.hand-button:disabled {
  cursor: not-allowed;
}

.candidate-badge {
  right: -0.26rem;
  top: -0.3rem;
}

@media (max-width: 960px) {
  .self-hand-shell {
    grid-template-columns: clamp(4.2rem, 12vw, 5.4rem) minmax(0, 1fr);
  }

  .self-melds {
    min-height: 3.2rem;
    padding: 0.32rem;
  }

  .hand-row {
    min-height: 5rem;
    padding: 0.55rem 0.45rem 0.3rem;
  }
}
</style>
