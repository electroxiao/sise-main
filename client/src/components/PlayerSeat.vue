<template>
  <section
    class="player-seat"
    :class="[position, { active: isActive, dealer: isDealer, responder: isResponder }]"
    :data-player-id="player.clientId"
  >
    <div class="portrait" :class="avatarTone">
      <span v-if="isActive" class="turn-arrow" aria-hidden="true"></span>
      <span class="portrait-name">{{ portraitText }}</span>
      <i v-if="isDealer">庄</i>
    </div>
    <div class="hand-count">
      <div class="backs">
        <FourColorCard v-for="index in backCount" :key="index" back size="tiny" />
      </div>
      <b>{{ handCount }}</b>
    </div>
    <div class="seat-panel">
      <header>
        <strong>{{ displayName }}</strong>
        <span v-if="player.isBot">BOT</span>
        <span v-else-if="!player.connected">托管</span>
      </header>
      <div class="meld-row" :class="{ empty: !groups.length }" :data-meld-anchor="player.clientId">
        <template v-if="groups.length">
          <div
            v-for="group in groups"
            :key="group.id"
            class="meld fan-meld"
            :class="group.tone"
            :data-meld-group-anchor="player.clientId"
            :data-meld-card-ids="groupCardIds(group)"
            :style="fanGroupStyle(group.cards.length)"
          >
            <span
              v-for="(card, cardIndex) in group.cards"
              :key="card.id"
              class="fan-card"
              :style="fanCardStyle(cardIndex, group.cards.length)"
            >
              <FourColorCard :card="card" size="tiny" />
            </span>
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

function fanCardStyle(index: number, total: number): Record<string, string> {
  const center = (Math.max(total, 1) - 1) / 2;
  const offset = index - center;
  const maxSpread = total >= 5 ? 18.9 : total >= 4 ? 21.6 : 25.875;
  const rotation = offset * maxSpread;

  return {
    "--tilt": `${rotation.toFixed(2)}deg`,
    "--fan-z": `${index + 1}`,
  };
}

function fanGroupStyle(total: number): Record<string, string> {
  const count = Math.max(total, 1);
  const width = 1.35 + Math.max(count - 1, 0) * 0.34;

  return {
    "--fan-width": `${width.toFixed(2)}rem`,
  };
}

function groupCardIds(group: VisibleGroupBlock): string {
  return group.cards.map((card) => card.id).join("|");
}
</script>

<style scoped>
.player-seat {
  --opponent-avatar-w: clamp(6.4rem, min(10vw, 18vh), 9.2rem);
  position: absolute;
  z-index: 5;
  display: grid;
  grid-template-columns: auto minmax(10rem, 16rem);
  align-items: center;
  gap: 0.55rem;
}

.player-seat.active {
  z-index: 60;
}

.top {
  top: 2.1vh;
  left: 50%;
  transform: translateX(-50%);
  width: 100%;
  grid-template-columns: auto;
  justify-items: center;
  gap: 0;
  pointer-events: none;
}

.left {
  left: clamp(0.7rem, 1.4vw, 1.6rem);
  top: 37%;
  width: var(--opponent-avatar-w);
  transform: translateY(-50%);
  grid-template-columns: var(--opponent-avatar-w);
  justify-items: center;
  gap: 0;
}

.right {
  right: clamp(0.7rem, 1.4vw, 1.6rem);
  top: 37%;
  width: var(--opponent-avatar-w);
  transform: translateY(-50%);
  grid-template-columns: var(--opponent-avatar-w);
  justify-items: center;
  gap: 0;
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
    linear-gradient(180deg, #f4ead6, #d5b98c);
  box-shadow: 0 0.9rem 1.4rem rgba(0, 0, 0, 0.38);
  overflow: visible;
}

.portrait-name {
  position: absolute;
  left: -0.35rem;
  right: -0.35rem;
  bottom: 0;
  min-height: 2.1rem;
  display: grid;
  place-items: center;
  border: 2px solid rgba(23, 19, 15, 0.55);
  border-bottom: 0;
  border-radius: 0 0 0.38rem 0.38rem;
  border-top: 2px solid rgba(23, 19, 15, 0.55);
  background: rgba(255, 246, 226, 0.92);
  font-weight: 900;
  line-height: 1;
  overflow: visible;
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

.turn-arrow {
  position: absolute;
  z-index: 80;
  width: 4.7rem;
  height: 4.7rem;
  display: grid;
  place-items: center;
  color: #28d84d;
  font-family: "Arial Black", "Microsoft YaHei", sans-serif;
  font-size: 4.2rem;
  font-weight: 900;
  line-height: 1;
  text-shadow:
    -2px -2px 0 #111,
    0 -2px 0 #111,
    2px -2px 0 #111,
    -2px 0 0 #111,
    2px 0 0 #111,
    -2px 2px 0 #111,
    0 2px 0 #111,
    2px 2px 0 #111,
    0 0.22rem 0.14rem rgba(0, 0, 0, 0.42);
  -webkit-text-stroke: 2.4px #111;
  pointer-events: none;
}

.turn-arrow::before {
  display: block;
}

.top .turn-arrow {
  left: 50%;
  top: calc(100% + 0.46rem);
  animation: turn-arrow-top 0.72s ease-in-out infinite;
}

.top .turn-arrow::before {
  content: "⬆";
}

.left .turn-arrow {
  left: calc(100% + 0.54rem);
  top: 50%;
  animation: turn-arrow-left 0.72s ease-in-out infinite;
}

.left .turn-arrow::before {
  content: "⬅";
}

.right .turn-arrow {
  right: calc(100% + 0.54rem);
  top: 50%;
  animation: turn-arrow-right 0.72s ease-in-out infinite;
}

.right .turn-arrow::before {
  content: "➡";
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

.top .portrait,
.left .portrait,
.right .portrait {
  width: var(--opponent-avatar-w);
}

.top .portrait {
  z-index: 2;
  pointer-events: auto;
}

.top .seat-panel {
  position: absolute;
  left: calc(50% + var(--opponent-avatar-w) / 2 + clamp(0.5rem, 1vw, 0.9rem));
  top: calc(var(--opponent-avatar-w) * 0.08);
  width: clamp(19rem, 34vw, 31rem);
  max-height: clamp(5.2rem, 21vh, 9.2rem);
  overflow: auto;
  pointer-events: auto;
  scrollbar-width: none;
}

.left .seat-panel {
  position: absolute;
  left: 0;
  top: calc(var(--opponent-avatar-w) * 1.24);
  z-index: 3;
  width: clamp(10rem, calc(42vw - 6.25rem), 24.75rem);
  transform: none;
}

.right .seat-panel {
  position: absolute;
  right: 0;
  top: calc(var(--opponent-avatar-w) * 1.24);
  z-index: 3;
  width: clamp(10rem, calc(42vw - 6.25rem), 24.75rem);
  transform: none;
}

.top .portrait-name,
.left .portrait-name,
.right .portrait-name {
  min-height: 2.25rem;
  font-size: clamp(1.05rem, 1.8vw, 1.45rem);
}

.top .portrait-name,
.left .portrait-name,
.right .portrait-name {
  display: none;
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
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
  box-shadow: none;
}

.top .seat-panel {
  width: clamp(19rem, 34vw, 31rem);
  max-height: none;
  overflow: visible;
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
  position: relative;
  z-index: 4;
  width: max-content;
  display: flex;
  align-items: end;
  gap: 0.45rem;
  margin-top: 0.3rem;
}

.top .hand-count,
.left .hand-count,
.right .hand-count {
  position: absolute;
  width: max-content;
  justify-content: flex-start;
  margin-top: 0;
  gap: 0.38rem;
}

.top .hand-count {
  right: calc(50% + var(--opponent-avatar-w) / 2 + clamp(0.55rem, 1vw, 0.9rem));
  top: 0;
  margin-right: 0;
}

.left .hand-count {
  left: calc(var(--opponent-avatar-w) + 0.7rem);
  top: 0;
  margin-left: 0;
}

.right .hand-count {
  right: calc(var(--opponent-avatar-w) + 0.7rem);
  top: 0;
  margin-left: 0;
  flex-direction: row-reverse;
  justify-content: flex-start;
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
  border: 0;
  border-radius: 0;
  display: grid;
  place-items: center;
  background: transparent;
  color: #f7ead5;
  font-size: 1.1rem;
  text-shadow:
    0 2px 0 #17120d,
    0 0 0.4rem rgba(0, 0, 0, 0.82);
}

.top .hand-count b,
.left .hand-count b,
.right .hand-count b {
  background: transparent;
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
  margin-top: 0;
  overflow: visible;
}

.top .meld-row {
  max-width: min(90vw, 64rem);
}

.left .meld-row,
.right .meld-row {
  max-width: 100%;
}

.right .meld-row {
  justify-content: flex-end;
}

.top .meld-row,
.left .meld-row,
.right .meld-row {
  gap: 1.12rem;
}

.meld {
  position: relative;
  width: var(--fan-width, 1.35rem);
  height: 3.55rem;
  flex: 0 0 auto;
  padding: 0;
  border-radius: 0;
  background: transparent;
}

.fan-card {
  position: absolute;
  left: 50%;
  bottom: 0.18rem;
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

.active .portrait,
.responder .portrait {
  box-shadow:
    0 0.9rem 1.4rem rgba(0, 0, 0, 0.38),
    0 0 0 4px rgba(94, 211, 117, 0.72);
}

.dealer .seat-panel {
  border-color: #8d5e18;
}

@keyframes turn-arrow-top {
  0%,
  100% {
    transform: translateX(-50%) translateY(0);
  }

  50% {
    transform: translateX(-50%) translateY(0.38rem);
  }
}

@keyframes turn-arrow-left {
  0%,
  100% {
    transform: translateY(-50%) translateX(0);
  }

  50% {
    transform: translateY(-50%) translateX(0.38rem);
  }
}

@keyframes turn-arrow-right {
  0%,
  100% {
    transform: translateY(-50%) translateX(0);
  }

  50% {
    transform: translateY(-50%) translateX(-0.38rem);
  }
}

@media (max-width: 960px) {
  .player-seat {
    --opponent-avatar-w: clamp(5.8rem, min(12vw, 18vh), 7.5rem);
  }

  .player-seat {
    grid-template-columns: auto minmax(7rem, 12rem);
  }

  .top,
  .left,
  .right {
    grid-template-columns: var(--opponent-avatar-w);
  }

  .top {
    grid-template-columns: auto;
  }

  .left {
    grid-template-columns: var(--opponent-avatar-w);
  }

  .right {
    grid-template-columns: var(--opponent-avatar-w);
  }

  .top {
    top: 1vh;
  }

  .left,
  .right {
    top: 36%;
    width: var(--opponent-avatar-w);
  }

  .top .portrait,
  .left .portrait,
  .right .portrait {
    width: var(--opponent-avatar-w);
  }

  .top .seat-panel {
    left: calc(50% + var(--opponent-avatar-w) / 2 + 0.45rem);
    width: clamp(17rem, 38vw, 24rem);
    top: calc(var(--opponent-avatar-w) * 0.08);
    max-height: none;
  }

  .left .seat-panel {
    left: 0;
    top: calc(var(--opponent-avatar-w) * 1.24);
    width: clamp(8.5rem, calc(42vw - 6.25rem), 17.75rem);
  }

  .right .seat-panel {
    right: 0;
    top: calc(var(--opponent-avatar-w) * 1.24);
    width: clamp(8.5rem, calc(42vw - 6.25rem), 17.75rem);
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
