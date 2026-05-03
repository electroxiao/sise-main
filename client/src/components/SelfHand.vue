<template>
  <section class="self-hand-shell">
    <div class="self-portrait" :class="{ active: isActive, dealer: isDealer }">
    </div>
    <div class="self-play">
      <div class="self-melds" :class="{ empty: !groups.length }">
        <div
          v-for="group in groups"
          :key="group.id"
          class="self-meld fan-meld"
          :style="fanGroupStyle(group.cards.length)"
        >
          <span
            v-for="(card, cardIndex) in group.cards"
            :key="card.id"
            class="fan-card"
            :style="fanCardStyle(cardIndex, group.cards.length)"
          >
            <FourColorCard :card="card" size="sm" />
          </span>
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
            stretched-face
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

function fanCardStyle(index: number, total: number): Record<string, string> {
  const center = (Math.max(total, 1) - 1) / 2;
  const offset = index - center;
  const maxSpread = total >= 5 ? 17.1 : total >= 4 ? 19.8 : 23.625;
  const rotation = offset * maxSpread;

  return {
    "--tilt": `${rotation.toFixed(2)}deg`,
    "--fan-z": `${index + 1}`,
  };
}

function fanGroupStyle(total: number): Record<string, string> {
  const count = Math.max(total, 1);
  const width = 1.65 + Math.max(count - 1, 0) * 0.42;

  return {
    "--fan-width": `${width.toFixed(2)}rem`,
  };
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
    linear-gradient(180deg, #e9f4e5 0%, #bcd9ad 56%, #5a8b57 57%);
  box-shadow: 0 0.5rem 0 #3c2c1b, 0 0.9rem 1.5rem rgba(0, 0, 0, 0.42);
  overflow: hidden;
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
  gap: 2rem;
  padding: 0.2rem 0.65rem 0.24rem 1.35rem;
  border: 0;
  border-radius: 0;
  background: transparent;
  overflow: visible;
}

.self-melds.empty {
  pointer-events: none;
}

.self-meld {
  position: relative;
  flex: 0 0 auto;
  width: var(--fan-width, 1.65rem);
  height: 4.25rem;
  padding: 0.34rem 0.36rem 0.2rem;
  border-radius: 0;
  background: transparent;
}

.fan-card {
  position: absolute;
  left: 50%;
  bottom: 0.2rem;
  z-index: var(--fan-z);
  display: block;
  transform: translateX(-50%);
  transform-origin: 50% 100%;
}

.fan-card :deep(.fc-card) {
  transform-origin: 50% 100%;
}

.fan-card :deep(.fc-card.response) {
  box-shadow:
    0 0.32rem 0 rgba(20, 14, 10, 0.86),
    0 0.55rem 1rem rgba(0, 0, 0, 0.34),
    inset 0 0 0 2px rgba(255, 255, 255, 0.28);
}

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

.hand-row {
  min-height: clamp(5.8rem, 11vw, 7.5rem);
  display: flex;
  align-items: end;
  gap: clamp(0.12rem, 0.45vw, 0.32rem);
  padding: 0.75rem 0.8rem 0.35rem;
  border: 2px solid rgba(20, 18, 15, 0.72);
  border-radius: 0.75rem;
  background:
    linear-gradient(180deg, rgba(52, 55, 51, 0.72), rgba(25, 26, 24, 0.82));
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
    padding: 0.32rem 0.32rem 0.32rem 1.1rem;
  }

  .hand-row {
    min-height: 5rem;
    padding: 0.55rem 0.45rem 0.3rem;
  }
}
</style>
