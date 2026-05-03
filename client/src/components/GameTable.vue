<template>
  <div class="game-table" data-testid="game-board">
    <div class="ink-backdrop"></div>
    <div class="table-slab">
      <div class="table-grain"></div>
      <PlayerSeat
        v-if="topPlayer"
        :player="topPlayer"
        position="top"
        :hand-count="playerHandCount(topPlayer)"
        :groups="groupBlocks(topPlayer, 'top')"
        :is-dealer="isDealer(topPlayer.clientId)"
        :is-active="isCurrentTurn(topPlayer.clientId)"
        :is-responder="isCollectiveResponder(topPlayer.clientId)"
      />
      <PlayerSeat
        v-if="leftPlayer"
        :player="leftPlayer"
        position="left"
        :hand-count="playerHandCount(leftPlayer)"
        :groups="groupBlocks(leftPlayer, 'left')"
        :is-dealer="isDealer(leftPlayer.clientId)"
        :is-active="isCurrentTurn(leftPlayer.clientId)"
        :is-responder="isCollectiveResponder(leftPlayer.clientId)"
      />
      <PlayerSeat
        v-if="rightPlayer"
        :player="rightPlayer"
        position="right"
        :hand-count="playerHandCount(rightPlayer)"
        :groups="groupBlocks(rightPlayer, 'right')"
        :is-dealer="isDealer(rightPlayer.clientId)"
        :is-active="isCurrentTurn(rightPlayer.clientId)"
        :is-responder="isCollectiveResponder(rightPlayer.clientId)"
      />
      <CenterPile
        :deck-count="Number(state?.deckCount ?? 0)"
        :response-card="responseCard"
        :response-phase="responsePhase"
        :current-player-name="currentPlayerName"
        :turn-hint="turnHint"
        :can-act="canAct"
      />
      <div class="action-float">
        <ActionDock
          :actions="actions"
          :can-act="canAct"
          :response-phase="responsePhase"
          :selection-mode="selectionMode"
          @submit="(request) => emit('submitAction', request)"
          @selection-change="(payload) => emit('selectionChange', payload)"
        />
      </div>
      <SelfHand
        v-if="selfPlayer"
        :player="selfPlayer"
        :private-hand="privateHand"
        :groups="groupBlocks(selfPlayer, 'self')"
        :can-discard="canDiscard"
        :is-active="isCurrentTurn(selfPlayer.clientId)"
        :is-dealer="isDealer(selfPlayer.clientId)"
        :active-candidates="activeCandidates"
        :selected-candidate-id="selectedCandidateId"
        @discard-card="(cardId) => emit('discardCard', cardId)"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from "vue";
import type { ActionCandidate, ActionRequest, AvailableAction, Card, PlayerState } from "@/types/game";
import ActionDock from "./ActionDock.vue";
import CenterPile from "./CenterPile.vue";
import PlayerSeat, { type VisibleGroupBlock } from "./PlayerSeat.vue";
import SelfHand from "./SelfHand.vue";

type SelectionMode = "kai" | "peng" | "chi" | null;
type ExposedGroup = {
  id: string;
  cards: Card[];
};

const props = withDefaults(
  defineProps<{
    state: any;
    players: PlayerState[];
    privateHand: Card[];
    mySeatId: string;
    canDiscard?: boolean;
    actions?: AvailableAction[];
    canAct?: boolean;
    responsePhase?: string;
    currentPlayerName?: string;
    turnHint?: string;
    selectionMode?: SelectionMode;
    selectedCandidateId?: string | null;
    activeCandidates?: ActionCandidate[];
  }>(),
  {
    canDiscard: false,
    actions: () => [],
    canAct: false,
    responsePhase: "",
    currentPlayerName: "-",
    turnHint: "",
    selectionMode: null,
    selectedCandidateId: null,
    activeCandidates: () => [],
  },
);

const emit = defineEmits<{
  discardCard: [cardId: string];
  submitAction: [request: ActionRequest];
  selectionChange: [payload: { mode: SelectionMode; selectedCandidateId: string | null }];
}>();

const orderedPlayers = computed(() => {
  if (!props.players.length) {
    return [];
  }
  const selfIndex = props.players.findIndex((player) => player.clientId === props.mySeatId);
  if (selfIndex < 0) {
    return props.players;
  }
  return [...props.players.slice(selfIndex), ...props.players.slice(0, selfIndex)];
});

const selfPlayer = computed(() => orderedPlayers.value[0] ?? props.players.find((x) => x.clientId === props.mySeatId) ?? null);
const rightPlayer = computed(() => orderedPlayers.value[1] ?? null);
const topPlayer = computed(() => orderedPlayers.value[2] ?? null);
const leftPlayer = computed(() => orderedPlayers.value[3] ?? null);

const responseCard = computed<Card | null>(() => {
  const directResponse = props.state?.responseCard;
  if (directResponse?.id) {
    return directResponse as Card;
  }
  const directTarget = props.state?.targetCard;
  if (directTarget?.id) {
    return directTarget as Card;
  }
  const publicCount = props.state?.publicDiscardPile?.length ?? 0;
  const publicTop = publicCount > 0 ? props.state?.publicDiscardPile?.[publicCount - 1] : null;
  return publicTop?.id ? ({ ...(publicTop as Card), source: "upper" } as Card) : null;
});

function splitGroups(cards: Card[], sizes: number[], prefix: string): ExposedGroup[] {
  const groups: ExposedGroup[] = [];
  let cursor = 0;
  sizes.forEach((rawSize, index) => {
    const size = Math.max(1, Number(rawSize) || 1);
    const chunk = cards.slice(cursor, cursor + size);
    cursor += size;
    if (chunk.length) {
      groups.push({ id: `${prefix}-${index}-${chunk.map((card) => card.id).join("-")}`, cards: chunk });
    }
  });
  if (cursor < cards.length) {
    cards.slice(cursor).forEach((card, index) => {
      groups.push({ id: `${prefix}-tail-${index}-${card.id}`, cards: [card] });
    });
  }
  return groups;
}

function splitFish(cards: Card[], prefix: string): ExposedGroup[] {
  const byFace = new Map<string, Card[]>();
  cards.forEach((card) => {
    const key = card.color === "gold" ? "gold" : `${card.color}:${card.type}`;
    byFace.set(key, [...(byFace.get(key) ?? []), card]);
  });
  const groups: ExposedGroup[] = [];
  let index = 0;
  byFace.forEach((items, key) => {
    for (let cursor = 0; cursor < items.length; cursor += 5) {
      const chunk = items.slice(cursor, cursor + 5);
      groups.push({ id: `${prefix}-fish-${key}-${index}`, cards: chunk });
      index += 1;
    }
  });
  return groups;
}

function groupBlocks(player: PlayerState, prefix: string): VisibleGroupBlock[] {
  const fish = splitFish(player.fishArea ?? [], prefix).map((group) => ({
    ...group,
    badge: "鱼",
    tone: "fish" as const,
  }));
  const exposed = splitGroups(player.exposedArea ?? [], player.exposedGroupSizes ?? [], `${prefix}-exp`).map((group, index) => ({
    ...group,
    badge: (player.exposedGroupKinds ?? [])[index] ? groupKindLabel((player.exposedGroupKinds ?? [])[index]) : undefined,
    tone: "meld" as const,
  }));
  const generals = (player.generalArea ?? []).map((card, index) => ({
    id: `${prefix}-general-${index}-${card.id}`,
    cards: [card],
    badge: "收",
    tone: "public" as const,
  }));
  return [...fish, ...exposed, ...generals];
}

function groupKindLabel(kind: string): string {
  const map: Record<string, string> = {
    chi: "吃",
    peng: "碰",
    kai: "开",
    fish: "鱼",
  };
  return map[kind] ?? "";
}

function playerHandCount(player: PlayerState): number {
  if (player.clientId === props.mySeatId) {
    return props.privateHand.length;
  }
  return Number(player.handCount ?? 0);
}

function isDealer(playerId: string): boolean {
  return Boolean(playerId) && props.state?.dealerId === playerId;
}

function isCurrentTurn(playerId: string): boolean {
  const displayId = props.state?.currentTurnPlayerId || props.state?.currentPlayerId || props.state?.activeResponderId;
  return Boolean(playerId) && displayId === playerId;
}

function isCollectiveResponder(playerId: string): boolean {
  return props.responsePhase === "collective" && Boolean(playerId) && props.state?.activeResponderId === playerId;
}
</script>

<style scoped>
.game-table {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 0;
  overflow: hidden;
  isolation: isolate;
  color: #f5ead5;
  background:
    radial-gradient(circle at 50% 18%, rgba(187, 132, 73, 0.24), transparent 28%),
    linear-gradient(180deg, #2a2726 0%, #161514 62%, #0d0d0c 100%);
}

.ink-backdrop {
  position: absolute;
  inset: 0;
  z-index: 0;
  background:
    radial-gradient(circle at 15% 20%, rgba(101, 82, 61, 0.48), transparent 22%),
    radial-gradient(circle at 84% 24%, rgba(81, 96, 58, 0.34), transparent 21%),
    repeating-radial-gradient(circle at 50% 30%, rgba(255, 255, 255, 0.035) 0 1px, transparent 1px 5px);
  filter: contrast(1.08);
}

.ink-backdrop::after {
  content: "";
  position: absolute;
  inset: 0;
  background:
    linear-gradient(110deg, transparent 0 43%, rgba(0, 0, 0, 0.22) 43% 45%, transparent 45%),
    linear-gradient(84deg, transparent 0 64%, rgba(0, 0, 0, 0.18) 64% 65%, transparent 65%);
  opacity: 0.42;
}

.table-slab {
  position: absolute;
  left: 50%;
  top: 50%;
  z-index: 1;
  width: min(96vw, 118rem);
  height: min(88vh, 57rem);
  min-height: 34rem;
  transform: translate(-50%, -50%) perspective(1100px) rotateX(9deg);
  border: 3px solid #171412;
  border-radius: 2.1rem 2.1rem 1.15rem 1.15rem;
  background:
    radial-gradient(ellipse at 50% 24%, rgba(88, 89, 82, 0.7), transparent 42%),
    linear-gradient(180deg, #3d403b 0%, #30342f 56%, #242723 100%);
  box-shadow:
    0 1.3rem 0 #141210,
    0 2rem 3rem rgba(0, 0, 0, 0.55),
    inset 0 0 0 4px rgba(255, 255, 255, 0.04);
  overflow: hidden;
}

.table-slab::before {
  content: "";
  position: absolute;
  inset: 4.5% 3.2% 9%;
  border: 2px solid rgba(9, 9, 8, 0.68);
  border-radius: 2.2rem 2.2rem 1.2rem 1.2rem;
  box-shadow: inset 0 0 0 2px rgba(255, 255, 255, 0.035);
}

.table-grain {
  position: absolute;
  inset: 0;
  z-index: 0;
  background:
    repeating-linear-gradient(95deg, rgba(255, 255, 255, 0.03) 0 2px, transparent 2px 12px),
    radial-gradient(circle at 38% 54%, rgba(255, 255, 255, 0.08), transparent 5%),
    radial-gradient(circle at 63% 42%, rgba(255, 255, 255, 0.07), transparent 6%),
    radial-gradient(circle at 28% 72%, rgba(255, 255, 255, 0.06), transparent 5%);
  opacity: 0.9;
}

.action-float {
  position: absolute;
  left: 50%;
  bottom: clamp(9.2rem, 20vh, 13.4rem);
  z-index: 12;
  transform: translateX(-50%) perspective(900px) rotateX(-9deg);
}

@media (max-width: 960px) {
  .table-slab {
    width: 100vw;
    height: 100%;
    min-height: 28rem;
    border-radius: 0;
  }

  .action-float {
    bottom: clamp(7.5rem, 24vh, 10rem);
  }
}
</style>
