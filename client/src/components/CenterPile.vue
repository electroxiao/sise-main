<template>
  <section class="center-pile" :class="{ active: isActive }">
    <div class="pile-row">
      <div class="deck-cluster" :title="`牌堆剩余 ${deckCount} 张`">
        <div class="deck-backs">
          <FourColorCard v-for="index in 5" :key="index" back size="sm" />
        </div>
        <strong>{{ deckCount }}</strong>
      </div>
      <div class="response-slot">
        <p>{{ responseLabel }}</p>
        <FourColorCard v-if="responseCard" :card="responseCard" size="lg" />
        <div v-else class="empty-card">待</div>
      </div>
    </div>
    <div class="turn-copy">
      <strong>{{ headline }}</strong>
      <span>{{ turnHint }}</span>
    </div>
  </section>
</template>

<script setup lang="ts">
import { computed } from "vue";
import type { Card } from "@/types/game";
import FourColorCard from "./FourColorCard.vue";

const props = withDefaults(
  defineProps<{
    deckCount: number;
    responseCard?: Card | null;
    responsePhase?: string;
    currentPlayerName?: string;
    turnHint?: string;
    canAct?: boolean;
  }>(),
  {
    responseCard: null,
    responsePhase: "",
    currentPlayerName: "-",
    turnHint: "",
    canAct: false,
  },
);

const isActive = computed(() => props.canAct);
const responseLabel = computed(() => (props.responseCard ? "待响牌" : "牌桌中央"));
const headline = computed(() => {
  if (props.canAct) {
    return "轮到你";
  }
  if (props.responsePhase === "collective") {
    return "全局待响";
  }
  if (props.responsePhase === "local_upper") {
    return "本家抉择";
  }
  if (props.responsePhase === "local_draw") {
    return "抓后处理";
  }
  return props.currentPlayerName ? `${props.currentPlayerName} 操作中` : "牌局进行中";
});
</script>

<style scoped>
.center-pile {
  position: absolute;
  left: 50%;
  top: 51%;
  z-index: 4;
  width: max-content;
  min-width: 0;
  transform: translate(-50%, -50%) perspective(900px) rotateX(9deg);
  display: grid;
  justify-content: center;
  justify-items: center;
  align-items: center;
  gap: 0.32rem;
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.center-pile.active {
  filter: drop-shadow(0 0 1.1rem rgba(247, 205, 106, 0.25));
}

.pile-row {
  display: grid;
  grid-template-columns: auto auto;
  align-items: end;
  justify-content: center;
  gap: clamp(1.05rem, 2.2vw, 1.7rem);
}

.deck-cluster {
  position: relative;
  width: 5.2rem;
  height: 3.6rem;
  display: grid;
  place-items: end center;
}

.deck-backs {
  height: 3.15rem;
  display: flex;
  align-items: end;
}

.deck-backs :deep(.fc-card) {
  margin-left: -0.85rem;
  transform: scale(0.86) rotate(var(--tilt, -1deg));
  transform-origin: bottom center;
}

.deck-backs :deep(.fc-card:first-child) {
  margin-left: 0;
}

.deck-cluster strong {
  position: relative;
  z-index: 2;
  min-width: 2.7rem;
  min-height: 1.55rem;
  margin-top: -0.76rem;
  border: 2px solid #28231e;
  border-radius: 0.45rem;
  display: grid;
  place-items: center;
  color: #2c251d;
  background: linear-gradient(180deg, #f8f1e4, #bfc5bc);
  font-size: 1.05rem;
  box-shadow: 0 0.35rem 0 rgba(0, 0, 0, 0.42);
}

.response-slot {
  display: grid;
  justify-items: center;
  gap: 0.18rem;
}

.response-slot p,
.turn-copy span {
  margin: 0;
  color: #d6c5a4;
  font-size: 0.78rem;
}

.empty-card {
  width: 1.85rem;
  height: 4.1rem;
  border: 2px dashed rgba(230, 215, 184, 0.42);
  border-radius: 0.4rem;
  display: grid;
  place-items: center;
  color: rgba(230, 215, 184, 0.5);
  font-family: "KaiTi", "STKaiti", serif;
  font-size: 1.35rem;
}

.turn-copy {
  min-width: 0;
  display: grid;
  gap: 0.12rem;
  justify-items: center;
  text-align: center;
}

.turn-copy strong {
  color: #fff2cf;
  font-family: "KaiTi", "STKaiti", serif;
  font-size: clamp(0.96rem, 2vh, 1.25rem);
  text-shadow: 0 2px 0 rgba(0, 0, 0, 0.42);
}

@media (max-width: 960px) {
  .center-pile {
    top: 49%;
  }

  .pile-row {
    gap: clamp(0.75rem, 2vw, 1.1rem);
  }
}
</style>
