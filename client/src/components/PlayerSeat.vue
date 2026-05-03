<template>
  <section class="player-seat" :class="[position, { active: isActive, dealer: isDealer, responder: isResponder }]">
    <div class="portrait" :class="avatarTone">
      <span>{{ portraitText }}</span>
      <i v-if="isDealer">庄</i>
    </div>
    <div class="seat-panel">
      <header>
        <strong>{{ displayName }}</strong>
        <span v-if="player.isBot">BOT</span>
        <span v-else-if="!player.connected">托管</span>
      </header>
      <div class="hand-count">
        <div class="backs">
          <FourColorCard v-for="index in backCount" :key="index" back size="tiny" />
        </div>
        <b>{{ handCount }}</b>
      </div>
      <div class="meld-row" :class="{ empty: !groups.length }">
        <template v-if="groups.length">
          <div v-for="group in groups" :key="group.id" class="meld" :class="group.tone">
            <em v-if="group.badge">{{ group.badge }}</em>
            <FourColorCard v-for="card in group.cards" :key="card.id" :card="card" size="tiny" />
          </div>
        </template>
      </div>
    </div>
  </section>
</template>

<script setup lang="ts">
import { computed } from "vue";
import type { Card, PlayerState } from "@/types/game";
import FourColorCard from "./FourColorCard.vue";

export type VisibleGroupBlock = {
  id: string;
  cards: Card[];
  badge?: string;
  tone: "meld" | "fish" | "public";
};

const props = defineProps<{
  player: PlayerState;
  position: "top" | "left" | "right";
  handCount: number;
  groups: VisibleGroupBlock[];
  isDealer: boolean;
  isActive: boolean;
  isResponder: boolean;
}>();

const displayName = computed(() => props.player.name.replace(/\s*\[BOT\]\s*$/, ""));
const portraitText = computed(() => displayName.value.slice(0, 2) || "客");
const backCount = computed(() => Math.max(2, Math.min(5, Math.ceil(props.handCount / 5))));
const avatarTone = computed(() => {
  const code = [...props.player.clientId].reduce((sum, ch) => sum + ch.charCodeAt(0), 0);
  return `tone-${(code % 4) + 1}`;
});
</script>

<style scoped>
.player-seat {
  position: absolute;
  z-index: 5;
  display: grid;
  grid-template-columns: auto minmax(10rem, 16rem);
  align-items: center;
  gap: 0.55rem;
}

.top {
  top: 2.1vh;
  left: 50%;
  transform: translateX(-50%);
  grid-template-columns: auto minmax(18rem, 28rem);
  justify-items: start;
  gap: 0.9rem;
}

.left {
  left: 3vw;
  top: 36%;
  transform: translateY(-50%);
  grid-template-columns: auto minmax(15rem, 23rem);
  justify-items: center;
  gap: 0.5rem;
}

.right {
  right: 3vw;
  top: 36%;
  transform: translateY(-50%);
  grid-template-columns: minmax(15rem, 23rem) auto;
  justify-items: center;
  gap: 0.5rem;
}

.right .portrait {
  order: 2;
}

.portrait {
  position: relative;
  width: clamp(4.6rem, 8vw, 6.8rem);
  aspect-ratio: 0.86;
  border: 3px solid #17130f;
  border-radius: 0.5rem;
  display: grid;
  place-items: center;
  color: #21160f;
  background:
    radial-gradient(circle at 50% 28%, rgba(255, 255, 255, 0.7), transparent 19%),
    linear-gradient(180deg, #f4ead6, #d5b98c);
  box-shadow: 0 0.5rem 0 #3c2c1b, 0 0.9rem 1.4rem rgba(0, 0, 0, 0.38);
  overflow: hidden;
}

.portrait::before {
  content: "";
  position: absolute;
  inset: 0.55rem 0.75rem 2.1rem;
  border-radius: 46% 46% 42% 42%;
  background: rgba(38, 33, 28, 0.88);
  box-shadow: 0 1.55rem 0 0 #f1c7a8;
}

.portrait span {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  min-height: 1.7rem;
  display: grid;
  place-items: center;
  border-top: 2px solid rgba(23, 19, 15, 0.55);
  background: rgba(255, 246, 226, 0.92);
  font-weight: 900;
}

.portrait i {
  position: absolute;
  right: 0.2rem;
  top: 0.2rem;
  width: 1.6rem;
  height: 1.6rem;
  border-radius: 999px;
  display: grid;
  place-items: center;
  background: #c93622;
  color: #ffe8a3;
  font-style: normal;
  font-weight: 900;
  box-shadow: 0 0 0 2px #2d160f;
}

.tone-2::before {
  background: #5e3d25;
}

.tone-3::before {
  background: #1e3227;
}

.tone-4::before {
  background: #263850;
}

.left .portrait,
.right .portrait {
  width: clamp(7.6rem, 14vw, 11.6rem);
}

.left .portrait::before,
.right .portrait::before {
  inset: 0.8rem 1rem 2.75rem;
  box-shadow: 0 2.15rem 0 0 #f1c7a8;
}

.left .portrait span,
.right .portrait span {
  min-height: 2.25rem;
  font-size: clamp(1.05rem, 1.8vw, 1.45rem);
}

.seat-panel {
  min-width: 0;
  padding: 0.55rem 0.65rem;
  border: 2px solid rgba(21, 17, 13, 0.86);
  border-radius: 0.6rem;
  background:
    linear-gradient(180deg, rgba(254, 247, 230, 0.94), rgba(190, 176, 148, 0.86)),
    repeating-linear-gradient(0deg, rgba(0, 0, 0, 0.05) 0 2px, transparent 2px 7px);
  box-shadow: 0 0.5rem 1rem rgba(0, 0, 0, 0.28);
}

.top .seat-panel,
.left .seat-panel,
.right .seat-panel {
  width: 100%;
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 0.5rem;
}

header strong {
  overflow: hidden;
  color: #241a12;
  font-size: clamp(0.88rem, 1.4vw, 1.08rem);
  white-space: nowrap;
  text-overflow: ellipsis;
}

header span {
  flex: 0 0 auto;
  padding: 0.08rem 0.36rem;
  border-radius: 999px;
  color: #f7eddb;
  background: #27313b;
  font-size: 0.68rem;
  font-weight: 900;
}

.top header,
.left header,
.right header {
  display: none;
}

.hand-count {
  display: flex;
  align-items: end;
  gap: 0.45rem;
  margin-top: 0.3rem;
}

.top .hand-count,
.left .hand-count,
.right .hand-count {
  justify-content: flex-start;
  margin-top: 0.2rem;
  gap: 0.38rem;
}

.right .hand-count {
  justify-content: flex-end;
}

.backs {
  height: 3.4rem;
  display: flex;
  align-items: end;
}

.backs :deep(.fc-card) {
  margin-left: -0.75rem;
}

.backs :deep(.fc-card:first-child) {
  margin-left: 0;
}

.hand-count b {
  min-width: 2.1rem;
  min-height: 1.7rem;
  border: 2px solid #2b251f;
  border-radius: 0.35rem;
  display: grid;
  place-items: center;
  background: #f8efe0;
  color: #271d14;
  font-size: 1.1rem;
}

.top .hand-count b,
.left .hand-count b,
.right .hand-count b {
  background: #f4ebdc;
}

.meld-row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.4rem;
  min-height: 3.35rem;
  margin-top: 0.45rem;
}

.meld-row.empty {
  min-height: 0;
}

.top .meld-row,
.left .meld-row,
.right .meld-row {
  justify-content: flex-start;
  align-content: flex-start;
  min-height: 0;
  max-width: 100%;
  margin-top: 0.42rem;
  overflow: visible;
}

.right .meld-row {
  justify-content: flex-end;
}

.meld {
  position: relative;
  display: flex;
  align-items: end;
  gap: 0.12rem;
  padding: 0.25rem;
  border-radius: 0.35rem;
  background: rgba(37, 31, 25, 0.18);
}

.meld em {
  position: absolute;
  left: -0.35rem;
  top: -0.35rem;
  z-index: 2;
  min-width: 1.35rem;
  min-height: 1.35rem;
  border-radius: 999px;
  display: grid;
  place-items: center;
  color: #fff3cc;
  background: #1f2933;
  font-size: 0.68rem;
  font-style: normal;
  font-weight: 900;
}

.active .portrait,
.responder .portrait {
  box-shadow:
    0 0.5rem 0 #3c2c1b,
    0 0.9rem 1.4rem rgba(0, 0, 0, 0.38),
    0 0 0 4px rgba(94, 211, 117, 0.72);
}

.dealer .seat-panel {
  border-color: #8d5e18;
}

@media (max-width: 960px) {
  .player-seat {
    grid-template-columns: auto minmax(7rem, 12rem);
  }

  .top,
  .left,
  .right {
    grid-template-columns: auto minmax(10rem, 1fr);
  }

  .right {
    grid-template-columns: minmax(10rem, 1fr) auto;
  }

  .top {
    top: 1vh;
  }

  .left,
  .right {
    top: 34%;
  }

  .left .portrait,
  .right .portrait {
    width: clamp(6.2rem, 15vw, 8rem);
  }

  .seat-panel {
    padding: 0.42rem;
  }

  .top .seat-panel,
  .left .seat-panel,
  .right .seat-panel {
    padding: 0;
  }

  .meld-row {
    max-height: none;
    overflow: visible;
  }
}
</style>
