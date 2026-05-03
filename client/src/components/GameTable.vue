<template>
  <div class="game-table" data-testid="game-board">
    <div class="ink-backdrop"></div>
    <div ref="tableRef" class="table-slab">
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
      <div class="discard-routes" aria-hidden="true">
        <div
          v-for="player in orderedPlayers"
          :key="`discard-${player.clientId}`"
          class="discard-strip"
          :class="relativePositionClass(player.clientId)"
          :data-discard-anchor="player.clientId"
          data-testid="discard-flow-zone"
        >
          <span v-if="hiddenDiscardCount(player) > 0" class="discard-more">+{{ hiddenDiscardCount(player) }}</span>
          <span
            v-for="card in visibleDiscardCards(player)"
            :key="`discard-${player.clientId}-${card.id}`"
            class="discard-chip"
            :class="{ pending: isPendingDiscard(player.clientId, card) }"
          >
            <FourColorCard :card="card" size="tiny" />
          </span>
        </div>
      </div>
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
        @discard-card="handleDiscardCard"
      />
    </div>
    <div class="flight-layer" aria-hidden="true">
      <div
        v-for="flight in flights"
        :key="`flight-${flight.id}`"
        class="flight-card"
        :class="flight.mode"
        :style="flightStyle(flight)"
      >
        <FourColorCard v-if="flight.card" :card="flight.card" size="md" />
        <FourColorCard v-else back size="md" />
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onUnmounted, ref, watch } from "vue";
import type { ActionCandidate, ActionRequest, AvailableAction, Card, PlayerState } from "@/types/game";
import ActionDock from "./ActionDock.vue";
import CenterPile from "./CenterPile.vue";
import FourColorCard from "./FourColorCard.vue";
import PlayerSeat, { type VisibleGroupBlock } from "./PlayerSeat.vue";
import SelfHand from "./SelfHand.vue";

type SelectionMode = "kai" | "peng" | "chi" | null;
type ExposedGroup = {
  id: string;
  cards: Card[];
};
type SeatPosition = "self" | "right" | "top" | "left";
type FlightMode = "discard" | "draw" | "meld" | "settle";
type Point = {
  x: number;
  y: number;
};
type CardFlight = {
  id: number;
  mode: FlightMode;
  card?: Card;
  sx: number;
  sy: number;
  ex: number;
  ey: number;
  duration: number;
  delay: number;
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

const tableRef = ref<HTMLElement | null>(null);
const flights = ref<CardFlight[]>([]);
const recentResponseCard = ref<Card | null>(null);
const manualDiscardStart = ref<{ cardId: string; card: Card; point: Point } | null>(null);
const flightTimers: ReturnType<typeof setTimeout>[] = [];
let flightSeq = 0;
let lastAnimationFingerprint = "";

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
const positionByPlayerId = computed(() => {
  const map = new Map<string, SeatPosition>();
  orderedPlayers.value.forEach((player, index) => {
    const positions: SeatPosition[] = ["self", "right", "top", "left"];
    map.set(player.clientId, positions[index] ?? "top");
  });
  return map;
});

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

watch(
  responseCard,
  (card) => {
    if (card?.id) {
      recentResponseCard.value = { ...card };
    }
  },
  { immediate: true },
);

watch(
  () =>
    [
      props.state?.lastAction ?? "",
      props.state?.responsePhase ?? "",
      props.state?.responseCard?.id ?? "",
      props.state?.targetCard?.id ?? "",
      props.state?.deckCount ?? "",
      props.state?.pollOriginPlayerId ?? "",
      props.players
        .map(
          (player) =>
            `${player.clientId}:${player.discardPile?.map((card) => card.id).join(",")}:${player.exposedArea?.map((card) => card.id).join(",")}`,
        )
        .join("|"),
    ].join("::"),
  async (fingerprint, previous) => {
    if (!previous || fingerprint === lastAnimationFingerprint) {
      return;
    }
    lastAnimationFingerprint = fingerprint;
    await nextTick();
    triggerActionAnimation(String(props.state?.lastAction ?? ""));
  },
  { flush: "post" },
);

onUnmounted(() => {
  flightTimers.forEach((timer) => clearTimeout(timer));
  flightTimers.length = 0;
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

function handleDiscardCard(cardId: string): void {
  const card = props.privateHand.find((item) => item.id === cardId);
  const element = findCardElement(cardId);
  const point = element ? pointFromElement(element) : null;
  if (card && point) {
    manualDiscardStart.value = { cardId, card: { ...card }, point };
  }
  emit("discardCard", cardId);
}

function visibleDiscardCards(player: PlayerState): Card[] {
  const pendingId = responseCard.value?.id ?? "";
  const cards = (player.discardPile ?? []).filter((card) => card.id !== pendingId);
  return cards.slice(Math.max(0, cards.length - 7));
}

function hiddenDiscardCount(player: PlayerState): number {
  const pendingId = responseCard.value?.id ?? "";
  return Math.max(0, (player.discardPile ?? []).filter((card) => card.id !== pendingId).length - 7);
}

function relativePositionClass(playerId: string): string {
  return `pos-${positionByPlayerId.value.get(playerId) ?? "top"}`;
}

function isPendingDiscard(playerId: string, card: Card): boolean {
  const pending = responseCard.value;
  if (!pending?.id || pending.source !== "upper" || pending.id !== card.id) {
    return false;
  }
  const originId = String(props.state?.pollOriginPlayerId || props.state?.previousPlayerId || "");
  if (originId) {
    return originId === playerId;
  }
  const match = String(props.state?.lastAction ?? "").match(/^(\S+)\s+DISCARD$/);
  return match?.[1] === playerId;
}

function triggerActionAnimation(lastAction: string): void {
  const match = lastAction.trim().match(/^(\S+)\s+([A-Z_]+)$/);
  if (!match) {
    return;
  }
  const actorId = match[1];
  const keyword = match[2];
  if (keyword === "DISCARD") {
    triggerDiscardAnimation(actorId);
    return;
  }
  if (keyword === "ZHUA" || keyword === "TURN_DRAW" || keyword === "KONG_DRAW") {
    triggerDrawAnimation();
    return;
  }
  if (keyword === "CHI" || keyword === "PENG" || keyword === "KAI") {
    triggerMeldAnimation(actorId, keyword);
    return;
  }
  if (keyword === "PASS") {
    triggerSettleAnimation(actorId);
  }
}

function triggerDiscardAnimation(actorId: string): void {
  const card = responseCard.value ?? recentResponseCard.value ?? latestPlayerDiscard(actorId);
  const target = pointForSelector('[data-flight-anchor="response"]');
  const manual = manualDiscardStart.value?.cardId === card?.id ? manualDiscardStart.value : null;
  const source = manual?.point ?? pointForPlayer(actorId);
  if (!card || !source || !target) {
    manualDiscardStart.value = null;
    return;
  }
  spawnFlight({
    mode: "discard",
    card: manual?.card ?? card,
    sx: source.x,
    sy: source.y,
    ex: target.x,
    ey: target.y,
    duration: 827,
    delay: 0,
  });
  manualDiscardStart.value = null;
}

function triggerDrawAnimation(): void {
  const card = responseCard.value ?? recentResponseCard.value;
  const source = pointForSelector('[data-flight-anchor="deck"]');
  const target = pointForSelector('[data-flight-anchor="response"]');
  if (!card || !source || !target) {
    return;
  }
  spawnFlight({
    mode: "draw",
    card,
    sx: source.x,
    sy: source.y,
    ex: target.x,
    ey: target.y,
    duration: 907,
    delay: 0,
  });
}

function triggerMeldAnimation(actorId: string, keyword: string): void {
  void keyword;
  const card = recentResponseCard.value ?? responseCard.value ?? latestExposedCard(actorId);
  const source = pointForSelector('[data-flight-anchor="response"]') ?? pointForDiscardOrigin();
  const target = pointForMeldGroup(actorId, card) ?? pointForMeld(actorId);
  if (!card || !source || !target) {
    return;
  }
  spawnFlight({
    mode: "meld",
    card,
    sx: source.x,
    sy: source.y,
    ex: target.x,
    ey: target.y,
    duration: 880,
    delay: 0,
  });
}

function triggerSettleAnimation(actorId: string): void {
  const card = recentResponseCard.value ?? responseCard.value ?? latestPlayerDiscard(actorId);
  const source = pointForSelector('[data-flight-anchor="response"]');
  const target = pointForDiscard(actorId);
  if (!card || !source || !target) {
    return;
  }
  spawnFlight({
    mode: "settle",
    card,
    sx: source.x,
    sy: source.y,
    ex: target.x,
    ey: target.y,
    duration: 747,
    delay: 0,
  });
}

function spawnFlight(flight: Omit<CardFlight, "id">): void {
  const id = ++flightSeq;
  flights.value.push({ id, ...flight });
  const timer = setTimeout(() => {
    flights.value = flights.value.filter((item) => item.id !== id);
  }, flight.duration + flight.delay + 120);
  flightTimers.push(timer);
}

function flightStyle(flight: CardFlight): Record<string, string> {
  return {
    "--sx": `${flight.sx}px`,
    "--sy": `${flight.sy}px`,
    "--ex": `${flight.ex}px`,
    "--ey": `${flight.ey}px`,
    "--dur": `${flight.duration}ms`,
    "--delay": `${flight.delay}ms`,
  };
}

function pointForSelector(selector: string): Point | null {
  const root = tableRef.value;
  const element = root?.querySelector(selector);
  return element instanceof HTMLElement ? pointFromElement(element) : null;
}

function pointForPlayer(playerId: string): Point | null {
  const root = tableRef.value;
  const element = [...(root?.querySelectorAll("[data-player-id]") ?? [])].find(
    (item) => item instanceof HTMLElement && item.dataset.playerId === playerId,
  );
  return element instanceof HTMLElement ? pointFromElement(element) : null;
}

function pointForMeld(playerId: string): Point | null {
  const root = tableRef.value;
  const element = [...(root?.querySelectorAll("[data-meld-anchor]") ?? [])].find(
    (item) => item instanceof HTMLElement && item.dataset.meldAnchor === playerId,
  );
  return element instanceof HTMLElement ? pointFromElement(element) : pointForPlayer(playerId);
}

function pointForMeldGroup(playerId: string, card?: Card | null): Point | null {
  if (!card?.id) {
    return null;
  }
  const root = tableRef.value;
  const element = [...(root?.querySelectorAll("[data-meld-group-anchor]") ?? [])].find((item) => {
    if (!(item instanceof HTMLElement) || item.dataset.meldGroupAnchor !== playerId) {
      return false;
    }
    return (item.dataset.meldCardIds ?? "").split("|").includes(card.id);
  });
  return element instanceof HTMLElement ? pointFromElement(element) : null;
}

function pointForDiscard(playerId: string): Point | null {
  const root = tableRef.value;
  const element = [...(root?.querySelectorAll("[data-discard-anchor]") ?? [])].find(
    (item) => item instanceof HTMLElement && item.dataset.discardAnchor === playerId,
  );
  return element instanceof HTMLElement ? pointFromElement(element) : pointForPlayer(playerId);
}

function pointForDiscardOrigin(): Point | null {
  const originId = String(props.state?.pollOriginPlayerId || props.state?.previousPlayerId || "");
  return originId ? pointForDiscard(originId) : null;
}

function pointFromElement(element: Element): Point | null {
  const rect = element.getBoundingClientRect();
  return {
    x: rect.left + rect.width / 2,
    y: rect.top + rect.height / 2,
  };
}

function findCardElement(cardId: string): HTMLElement | null {
  const root = tableRef.value;
  if (!root) {
    return null;
  }
  return (
    ([...root.querySelectorAll("[data-card-id]")].find(
      (item) => item instanceof HTMLElement && item.dataset.cardId === cardId,
    ) as HTMLElement | undefined) ?? null
  );
}

function latestPlayerDiscard(playerId: string): Card | null {
  const player = props.players.find((item) => item.clientId === playerId);
  const cards = player?.discardPile ?? [];
  return cards.length ? cards[cards.length - 1] : null;
}

function latestExposedCard(playerId: string): Card | null {
  const player = props.players.find((item) => item.clientId === playerId);
  const cards = player?.exposedArea ?? [];
  return cards.length ? cards[cards.length - 1] : null;
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
    radial-gradient(circle at 84% 24%, rgba(81, 96, 58, 0.34), transparent 21%);
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
  --opponent-avatar-w: clamp(6.4rem, min(10vw, 18vh), 9.2rem);
  --opponent-seat-inline: clamp(0.7rem, 1.4vw, 1.6rem);
  --opponent-hand-gap: 0.7rem;
  --opponent-hand-h: 3.4rem;
  --opponent-discard-gap: 0.58rem;
  --side-seat-top: 37%;
  --side-seat-visual-top: calc(var(--side-seat-top) - (var(--opponent-avatar-w) / 0.86 / 2));
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

.discard-routes {
  position: absolute;
  inset: 0;
  z-index: 6;
  pointer-events: none;
}

.discard-strip {
  position: absolute;
  min-width: 4.4rem;
  min-height: 2.3rem;
  display: flex;
  align-items: center;
  gap: 0.18rem;
  padding: 0.24rem 0.32rem;
  border: 1px solid rgba(18, 15, 12, 0.58);
  border-radius: 0.45rem;
  background: rgba(24, 23, 20, 0.38);
  box-shadow: inset 0 0 0 1px rgba(255, 238, 194, 0.05);
  opacity: 0.86;
}

.discard-strip.pos-self {
  left: calc(clamp(0.8rem, 2vw, 1.6rem) + clamp(2.5rem, 5vw, 4rem));
  bottom: calc(clamp(0.45rem, 1.2vh, 0.95rem) + clamp(5.6rem, 11.2vw, 8.9rem) + 0.45rem);
  transform: translateX(-50%);
}

.discard-strip.pos-right {
  right: calc(var(--opponent-seat-inline) + var(--opponent-avatar-w) + var(--opponent-hand-gap) + 1.72rem);
  top: calc(var(--side-seat-visual-top) + var(--opponent-hand-h) + var(--opponent-discard-gap));
  transform: translateX(50%);
  flex-direction: row-reverse;
}

.discard-strip.pos-top {
  right: calc(50% + var(--opponent-avatar-w) / 2 + clamp(0.55rem, 1vw, 0.9rem) + 4.05rem);
  top: calc(2.1vh + var(--opponent-hand-h) + var(--opponent-discard-gap));
  transform: translateX(50%);
}

.discard-strip.pos-left {
  left: calc(var(--opponent-seat-inline) + var(--opponent-avatar-w) + var(--opponent-hand-gap) + 1.72rem);
  top: calc(var(--side-seat-visual-top) + var(--opponent-hand-h) + var(--opponent-discard-gap));
  transform: translateX(-50%);
}

.discard-chip {
  position: relative;
  display: block;
  opacity: 0.66;
  transform: translateY(0.08rem);
}

.discard-chip :deep(.fc-card) {
  box-shadow:
    0 0.18rem 0 rgba(20, 14, 10, 0.72),
    0 0.32rem 0.55rem rgba(0, 0, 0, 0.28);
}

.discard-chip.pending {
  z-index: 2;
  opacity: 1;
  filter: drop-shadow(0 0 0.45rem rgba(248, 210, 88, 0.62));
}

.discard-more {
  min-width: 1.55rem;
  height: 1.45rem;
  display: grid;
  place-items: center;
  color: #e9d9b7;
  font-size: 0.68rem;
  font-weight: 900;
  text-shadow: 0 1px 0 rgba(0, 0, 0, 0.7);
}

.flight-layer {
  position: fixed;
  inset: 0;
  z-index: 32;
  overflow: hidden;
  pointer-events: none;
}

.flight-card {
  position: absolute;
  left: 0;
  top: 0;
  display: block;
  transform: translate(var(--sx), var(--sy)) translate(-50%, -50%);
  animation: card-flight var(--dur) cubic-bezier(0.35, 0, 0.25, 1) var(--delay) both;
  filter: drop-shadow(0 0.75rem 0.75rem rgba(0, 0, 0, 0.38));
  will-change: transform, opacity, filter;
}

.flight-card.draw {
  animation-timing-function: cubic-bezier(0.35, 0, 0.25, 1);
}

.flight-card.meld {
  filter: drop-shadow(0 0.7rem 0.65rem rgba(0, 0, 0, 0.42)) drop-shadow(0 0 0.35rem rgba(248, 210, 88, 0.34));
}

.flight-card.settle {
  animation-duration: var(--dur);
  opacity: 0.94;
}

@keyframes card-flight {
  0% {
    opacity: 0;
    transform: translate(var(--sx), var(--sy)) translate(-50%, -50%) scale(1) rotate(0deg);
  }

  12% {
    opacity: 1;
  }

  100% {
    opacity: 0.96;
    transform: translate(var(--ex), var(--ey)) translate(-50%, -50%) scale(1) rotate(0deg);
  }
}

@media (max-width: 960px) {
  .table-slab {
    --opponent-avatar-w: clamp(5.8rem, min(12vw, 18vh), 7.5rem);
    --opponent-discard-gap: 0.42rem;
    --side-seat-top: 36%;
    width: 100vw;
    height: 100%;
    min-height: 28rem;
    border-radius: 0;
  }

  .action-float {
    bottom: clamp(7.5rem, 24vh, 10rem);
  }

  .discard-strip {
    max-width: 10.5rem;
    padding: 0.16rem 0.22rem;
    gap: 0.08rem;
  }

  .discard-strip.pos-self {
    left: calc(clamp(0.8rem, 2vw, 1.6rem) + clamp(2.1rem, 6vw, 2.7rem));
    bottom: calc(clamp(0.45rem, 1.2vh, 0.95rem) + clamp(4.7rem, 13.4vw, 6rem) + 0.3rem);
  }

  .discard-strip.pos-top {
    right: calc(50% + var(--opponent-avatar-w) / 2 + 0.45rem + 3.7rem);
    top: calc(1vh + var(--opponent-hand-h) + var(--opponent-discard-gap));
  }

  .discard-strip.pos-left {
    left: calc(var(--opponent-seat-inline) + var(--opponent-avatar-w) + var(--opponent-hand-gap) + 1.6rem);
    top: calc(var(--side-seat-visual-top) + var(--opponent-hand-h) + var(--opponent-discard-gap));
  }

  .discard-strip.pos-right {
    right: calc(var(--opponent-seat-inline) + var(--opponent-avatar-w) + var(--opponent-hand-gap) + 1.6rem);
    top: calc(var(--side-seat-visual-top) + var(--opponent-hand-h) + var(--opponent-discard-gap));
  }
}
</style>
