# Design: Show Skill Advantages in SkillsList (#433)

## Summary

Display an advantage indicator (amber bolt icon) on skills where the character has unconditional advantage. Conditional advantages (like Stonecunning's "related to stonework") are out of scope for this issue.

## Data Source

The `/characters/{id}/stats` endpoint returns:

```json
{
  "skill_advantages": [
    {
      "skill": "Deception",
      "skill_slug": "deception",
      "condition": null,
      "source": "Actor"
    }
  ]
}
```

## Scope

- **In scope:** Unconditional advantages (`condition: null`)
- **Out of scope:** Conditional advantages — belong in DefensesPanel or future "Situational Bonuses" section

## UI Design

**Layout:** Icon appears after skill name
```
[●] +5 Deception ⚡ CHA
```

**Visual treatment:**
- Icon: `i-heroicons-bolt`
- Color: `text-warning-500` (amber)
- Size: `w-4 h-4`

**Tooltip:** `"Advantage (Actor)"` — shows source

**Accessibility:** `aria-label="Has advantage from Actor"`

## Data Flow

```
API: /characters/{id}/stats
  └── skill_advantages: [{ skill_slug, condition, source }]
        │
        ▼
useCharacterSheet.ts
  └── skillAdvantages: ComputedRef<SkillAdvantage[]>
        │
        ▼
SkillsList.vue
  └── Props: skills + skillAdvantages
  └── Matches skill.slug → skillAdvantage.skill_slug
  └── Renders: bolt icon with tooltip
```

## Implementation

### 1. types/character.ts

Add type (manual, not in OpenAPI spec):

```typescript
export interface SkillAdvantage {
  skill: string
  skill_slug: string
  condition: string | null
  source: string
}
```

### 2. composables/useCharacterSheet.ts

- Add `skillAdvantages` computed from `stats.value?.skill_advantages`
- Filter to unconditional only (`condition === null`)
- Add to return object and interface

### 3. components/character/sheet/SkillsList.vue

- Add `skillAdvantages` prop of type `SkillAdvantage[]`
- Create helper: `getAdvantage(slug: string): SkillAdvantage | undefined`
- Render bolt icon with UTooltip when advantage exists

### 4. pages/characters/[publicId]/index.vue

- Pass `skillAdvantages` prop to `<CharacterSheetSkillsList>`

### 5. Tests

Update `tests/components/character/sheet/SkillsList.spec.ts`:
- Test advantage icon renders when skill has advantage
- Test tooltip shows correct source
- Test no icon when no advantage
- Test conditional advantages are excluded

## Test Character

Use `dark-thorn-QjeD` (Variant Human + Actor feat) to verify:
- Deception should show advantage indicator
- Performance should show advantage indicator
- Other skills should not show indicator
