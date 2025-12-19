# Concentration Tracking Design

**Issues:** #783, #792
**Date:** 2025-12-19
**Status:** Approved

## Summary

Add concentration tracking to the spells page so players can see which spell they're concentrating on and easily manage concentration during play.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Persistence | Cookie per character | Survives refresh, no backend needed |
| Location | Stats bar (next to Spell DC) | Contextual, follows existing layout |
| Setting concentration | Explicit button on SpellCard | Clear player control |
| Swapping behavior | Silent swap + visual warning | Awareness without friction |
| Display | Spell name links to detail modal | Quick reference to spell effect |
| Clearing | Both indicator X button and SpellCard toggle | Two natural paths |

## Data Model

```typescript
interface ConcentrationState {
  spellId: number      // CharacterSpell ID
  spellName: string    // Display name
  spellSlug: string    // For detail modal
}
```

**Cookie:** `character-{publicId}-concentration`

## Components

### ConcentrationIndicator.vue (New)

**Location:** `app/components/character/sheet/ConcentrationIndicator.vue`

```typescript
interface Props {
  concentration: ConcentrationState | null
  canEdit: boolean
}

defineEmits<{
  clear: []
  viewSpell: [slug: string]
}>()
```

**States:**
- Not concentrating: Hidden
- Concentrating (view mode): "Bless" as clickable link
- Concentrating (play mode): "Bless [X]" with clear button

### SpellCard.vue (Modifications)

**New props:**
```typescript
activeConcentration: ConcentrationState | null
canEdit: boolean
```

**New emit:**
```typescript
concentrate: [spell: ConcentrationState]
```

**Button states (concentration spells only, play mode only):**
- Not concentrating: "Concentrate" button
- Concentrating on this spell: "End Concentration" (highlighted)
- Concentrating on another: "Concentrate" + warning indicator

### characterPlayState.ts (Additions)

```typescript
// State
const activeConcentration = ref<ConcentrationState | null>(null)

// Cookie sync
const concentrationCookie = useCookie<ConcentrationState | null>(
  `character-${characterId}-concentration`,
  { default: () => null }
)

// Actions
function setConcentration(spell: ConcentrationState | null): void
function clearConcentration(): void
```

## Page Integration

In `spells.vue`, add ConcentrationIndicator to the stats bar section alongside Spell DC, Attack, and Ability displays.

## Test Plan

| Test file | Scenarios |
|-----------|-----------|
| `ConcentrationIndicator.test.ts` | Render states, click handlers, clear button |
| `SpellCard.test.ts` | Concentrate button visibility, warning state, toggle |
| `characterPlayState.test.ts` | Set/clear actions, cookie persistence |

**Key scenarios:**
1. Set concentration on a spell
2. Swap concentration (auto-clears old)
3. Clear via indicator button
4. Clear via SpellCard toggle
5. Cookie survives navigation
6. Non-concentration spells hide button
7. View mode hides interactive elements

## Files

| File | Action |
|------|--------|
| `app/stores/characterPlayState.ts` | Add concentration state + cookie |
| `app/components/character/sheet/ConcentrationIndicator.vue` | New |
| `app/components/character/sheet/SpellCard.vue` | Add concentrate button |
| `app/pages/characters/[publicId]/spells.vue` | Wire up indicator |
| `tests/stores/characterPlayState.test.ts` | Add tests |
| `tests/components/character/sheet/ConcentrationIndicator.test.ts` | New |
| `tests/components/character/sheet/SpellCard.test.ts` | Add tests |
