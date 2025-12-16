# Modal State Standardization Design

**Issue:** #699
**Date:** 2025-12-16
**Status:** Approved

## Problem

Modal components use inconsistent state management patterns:
- 22 modals use `:open` + `@update:open` emit
- 7 picker modals use `:open` + `@close` emit
- 2 modals already use `defineModel`

## Solution

Standardize all 31 modals to use Vue 3.4+ `defineModel` pattern.

## Standard Pattern

```typescript
// 1. Open state via defineModel
const open = defineModel<boolean>('open', { default: false })

// 2. Other props (data the modal needs)
const props = defineProps<{
  characterId: string
  currentHp: number
}>()

// 3. Action emits only (no 'update:open' or 'close')
const emit = defineEmits<{
  apply: [value: number]
}>()

// 4. Close helper
function close() {
  open.value = false
}

// 5. Action handlers close after success
function handleApply() {
  emit('apply', result)
  close()
}
```

**Template:**
```vue
<UModal v-model:open="open">
  <!-- content -->
</UModal>
```

**Parent usage:**
```vue
<MyModal v-model:open="showModal" @apply="handleApply" />
```

## Migration Scope

### Group A: `update:open` → `defineModel` (22 modals)

Same external API, cleaner internals.

**sheet/ (9 files):**
- HpEditModal.vue
- TempHpModal.vue
- EditModal.vue
- NoteEditModal.vue
- NoteDeleteModal.vue
- CurrencyEditModal.vue
- AddConditionModal.vue
- DeadlyExhaustionConfirmModal.vue
- LongRestConfirmModal.vue

**inventory/ (6 files):**
- ShopModal.vue
- SellModal.vue
- AddLootModal.vue
- EditQuantityModal.vue
- DropConfirmModal.vue
- EquipSlotPickerModal.vue

**wizard/ (1 file):**
- WizardChangeConfirmationModal.vue

**party/ (2 files):**
- CreateModal.vue
- AddCharacterModal.vue

**dm-screen/ (3 files):**
- AddMonsterModal.vue
- SavePresetModal.vue
- LoadPresetModal.vue

**character/ (1 file):**
- ImportModal.vue

### Group B: `@close` → `defineModel` (8 modals)

API change - parents need update from `@close` to `v-model:open`.

**picker/ (8 files):**
- EntityDetailModal.vue (wrapper - remove computed bridge)
- SpellDetailModal.vue
- FeatDetailModal.vue
- RaceDetailModal.vue
- ClassDetailModal.vue
- BackgroundDetailModal.vue
- SubclassDetailModal.vue
- SubraceDetailModal.vue

### Already Done (2 modals)
- LevelUpConfirmModal.vue
- ItemDetailModal.vue

## Testing Strategy

- Update existing tests to use `v-model:open` pattern
- For picker modals: update parent component tests using `@close`
- No new test files needed

## Documentation

Add modal pattern to `.claude/rules/patterns.md`.

## Implementation Order

1. Group A (22 modals) - no API changes, lower risk
2. Group B (8 modals) - API changes require parent updates
3. Update documentation
