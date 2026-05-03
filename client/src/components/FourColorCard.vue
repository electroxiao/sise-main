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
        'stretched-face': stretchedFace,
        response: card?.isResponseCard,
      },
    ]"
    :title="card ? getCardLabelText(card) : '牌背'"
  >
    <template v-if="back"></template>
    <template v-else-if="card">
      <span class="face top">{{ face }}</span>
      <span class="face bottom">{{ face }}</span>
    </template>
  </div>
</template>

<script setup lang="ts">
import { computed } from "vue";
import type { Card } from "@/types/game";
import { getCardFaceText, getCardLabelText } from "@/utils/cardText";

const props = withDefaults(
  defineProps<{
    card?: Card | null;
    back?: boolean;
    size?: "tiny" | "sm" | "md" | "lg" | "xl";
    playable?: boolean;
    selected?: boolean;
    disabled?: boolean;
    stretchedFace?: boolean;
  }>(),
  {
    card: null,
    back: false,
    size: "md",
    playable: false,
    selected: false,
    disabled: false,
    stretchedFace: false,
  },
);

const face = computed(() => (props.card ? getCardFaceText(props.card) : ""));
const colorClass = computed(() => `tone-${props.card?.color ?? "white"}`);
</script>

<style scoped>
.fc-card {
  --card-ratio: 0.43 / 1;
  --card-w: 2.1rem;
  position: relative;
  width: var(--card-w);
  aspect-ratio: var(--card-ratio);
  border: 2px solid #14110d;
  border-radius: 0.34rem;
  display: grid;
  grid-template-rows: 1fr 1fr;
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
  font-size: 0.82rem;
}

.size-sm {
  --card-w: 1.65rem;
  font-size: 1rem;
}

.size-md {
  --card-w: 2rem;
  font-size: 1.18rem;
}

.size-lg {
  --card-w: 2.35rem;
  font-size: 1.38rem;
}

.size-xl {
  --card-w: clamp(1.65rem, 3.3vw, 2.35rem);
  font-size: clamp(1.32rem, 2.5vw, 1.82rem);
}

.face {
  writing-mode: vertical-rl;
  font-family: "KaiTi", "STKaiti", "DFKai-SB", serif;
  font-weight: 900;
  line-height: 0.9;
  letter-spacing: 0;
  text-shadow: 1px 1px 0 rgba(255, 255, 255, 0.55);
}

.top {
  align-self: center;
  transform: translateY(-0.22em);
}

.bottom {
  align-self: center;
  transform: translateY(0.22em) rotate(180deg);
}

.stretched-face .top {
  transform: translateY(-0.22em) scaleY(1.25);
}

.stretched-face .bottom {
  transform: translateY(0.22em) rotate(180deg) scaleY(1.25);
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

.is-back {
  overflow: hidden;
  border-color: #1f2a1b;
  background:
    radial-gradient(circle at 35% 20%, rgba(255, 245, 198, 0.18), transparent 28%),
    repeating-linear-gradient(115deg, rgba(255, 255, 255, 0.12) 0 3px, transparent 3px 9px),
    linear-gradient(180deg, #6ea04f, #385f32);
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
