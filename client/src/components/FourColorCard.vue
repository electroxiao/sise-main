<template>
  <div
    class="fc-card"
    :class="[
      back ? 'is-back' : colorClass,
      `size-${size}`,
      {
        playable,
        selected,
        disabled,
        response: card?.isResponseCard,
      },
    ]"
    :title="card ? getCardLabelText(card) : '牌背'"
  >
    <template v-if="back">
      <span class="back-mark"></span>
    </template>
    <template v-else-if="card">
      <span class="face top">{{ face }}</span>
      <span class="stripe">{{ colorText }}</span>
      <span class="face bottom">{{ face }}</span>
    </template>
  </div>
</template>

<script setup lang="ts">
import { computed } from "vue";
import type { Card } from "@/types/game";
import { getCardColorText, getCardFaceText, getCardLabelText } from "@/utils/cardText";

const props = withDefaults(
  defineProps<{
    card?: Card | null;
    back?: boolean;
    size?: "tiny" | "sm" | "md" | "lg" | "xl";
    playable?: boolean;
    selected?: boolean;
    disabled?: boolean;
  }>(),
  {
    card: null,
    back: false,
    size: "md",
    playable: false,
    selected: false,
    disabled: false,
  },
);

const face = computed(() => (props.card ? getCardFaceText(props.card) : ""));
const colorText = computed(() => (props.card ? getCardColorText(props.card) || "金" : ""));
const colorClass = computed(() => `tone-${props.card?.color ?? "white"}`);
</script>

<style scoped>
.fc-card {
  --card-w: 2.1rem;
  --card-h: 4.9rem;
  position: relative;
  width: var(--card-w);
  height: var(--card-h);
  border: 2px solid #14110d;
  border-radius: 0.34rem;
  display: grid;
  grid-template-rows: 1fr auto 1fr;
  place-items: center;
  color: #1d120a;
  background:
    linear-gradient(90deg, rgba(255, 255, 255, 0.62), transparent 32%, rgba(0, 0, 0, 0.04)),
    #f7ead6;
  box-shadow:
    0 0.32rem 0 rgba(20, 14, 10, 0.86),
    0 0.55rem 1rem rgba(0, 0, 0, 0.34),
    inset 0 0 0 2px rgba(255, 255, 255, 0.28);
  transform: rotate(var(--tilt, -1deg));
  transition: transform 0.18s ease, filter 0.18s ease, box-shadow 0.18s ease;
  user-select: none;
}

.size-tiny {
  --card-w: 1.35rem;
  --card-h: 3.05rem;
  font-size: 0.7rem;
}

.size-sm {
  --card-w: 1.65rem;
  --card-h: 3.7rem;
  font-size: 0.82rem;
}

.size-md {
  --card-w: 2rem;
  --card-h: 4.55rem;
  font-size: 1rem;
}

.size-lg {
  --card-w: 2.35rem;
  --card-h: 5.35rem;
  font-size: 1.16rem;
}

.size-xl {
  --card-w: clamp(2rem, 4.1vw, 2.9rem);
  --card-h: clamp(4.75rem, 9.5vw, 6.55rem);
  font-size: clamp(1rem, 2vw, 1.38rem);
}

.face {
  writing-mode: vertical-rl;
  font-family: "KaiTi", "STKaiti", "DFKai-SB", serif;
  font-weight: 900;
  line-height: 0.95;
  letter-spacing: 0;
  text-shadow: 1px 1px 0 rgba(255, 255, 255, 0.55);
}

.top {
  align-self: end;
}

.bottom {
  align-self: start;
  transform: rotate(180deg);
}

.stripe {
  width: 78%;
  min-height: 1.1em;
  border-radius: 999px;
  display: grid;
  place-items: center;
  font-size: 0.58em;
  font-weight: 900;
  color: rgba(255, 250, 235, 0.92);
  background: rgba(17, 24, 39, 0.78);
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.16);
}

.tone-red {
  background:
    linear-gradient(90deg, rgba(255, 255, 255, 0.4), transparent 32%, rgba(30, 0, 0, 0.1)),
    #e8673f;
}

.tone-yellow {
  background:
    linear-gradient(90deg, rgba(255, 255, 255, 0.46), transparent 32%, rgba(30, 0, 0, 0.08)),
    #f0c84f;
}

.tone-green {
  background:
    linear-gradient(90deg, rgba(255, 255, 255, 0.38), transparent 32%, rgba(0, 20, 0, 0.13)),
    #49b96f;
}

.tone-white {
  background:
    linear-gradient(90deg, rgba(255, 255, 255, 0.8), transparent 36%, rgba(0, 0, 0, 0.04)),
    #fff6e8;
}

.tone-gold {
  background:
    linear-gradient(90deg, rgba(255, 255, 255, 0.36), transparent 32%, rgba(40, 0, 0, 0.16)),
    #d94b31;
}

.tone-gold .stripe {
  background: #f6cf55;
  color: #4a170d;
}

.is-back {
  overflow: hidden;
  border-color: #1f2a1b;
  background:
    radial-gradient(circle at 35% 20%, rgba(255, 245, 198, 0.18), transparent 28%),
    repeating-linear-gradient(115deg, rgba(255, 255, 255, 0.12) 0 3px, transparent 3px 9px),
    linear-gradient(180deg, #6ea04f, #385f32);
}

.back-mark {
  width: 58%;
  height: 72%;
  border: 1px solid rgba(255, 245, 210, 0.34);
  border-radius: 0.25rem;
  box-shadow: inset 0 0 0 2px rgba(20, 40, 19, 0.22);
}

.playable {
  cursor: pointer;
}

.playable:hover {
  transform: translateY(-0.45rem) rotate(var(--tilt, -1deg));
  box-shadow:
    0 0.34rem 0 rgba(20, 14, 10, 0.9),
    0 0.7rem 1.25rem rgba(0, 0, 0, 0.42),
    0 0 0 3px rgba(251, 204, 91, 0.52);
}

.selected,
.response {
  box-shadow:
    0 0.32rem 0 rgba(20, 14, 10, 0.86),
    0 0.55rem 1rem rgba(0, 0, 0, 0.34),
    0 0 0 3px rgba(248, 210, 88, 0.72);
}

.disabled {
  filter: grayscale(0.35) brightness(0.82);
  opacity: 0.62;
}
</style>
