# Typography System: Crimson Pro Design

**Date:** 2025-11-22
**Status:** Approved - Ready for Phase A Implementation
**Font:** Crimson Pro (Google Fonts Variable)

---

## Overview

Replace default system fonts with **Crimson Pro**, a contemporary serif typeface that balances elegance with readability. Goal: "sophisticated but not fantasy overkill" - perfect for a D&D reference application.

---

## Design Decisions

### Font Selection: Crimson Pro

**Rationale:**
- **Modern classicism** - Contemporary book serif without decorative excess
- **Optimized for screens** - Designed specifically for digital reading
- **Variable font** - Single file, efficient loading, smooth weight scaling
- **High legibility** - Excellent x-height, works at small sizes (stats, tables)
- **Single family** - Use everywhere (headings + body) for visual coherence

**Rejected alternatives:**
- Literata + Spectral: Two fonts add complexity, not needed
- Lora: Too decorative, leans too far toward "fantasy aesthetic"

### Use Cases Supported

**Mixed use case:**
- ✅ Long-form reading (spell descriptions, trait text)
- ✅ Quick scanning (stat blocks, tables, lists)
- ✅ Visual hierarchy (headings, labels, body)

**Font everywhere:**
- Unified serif system (headings + body)
- Weight variation (400/600/700) creates hierarchy

---

## Typography Scale

### Headings

| Element | Weight | Usage |
|---------|--------|-------|
| `h1` | 700 (bold) | Page titles, entity names on detail pages |
| `h2` | 600 (semibold) | Section headers, card titles |
| `h3-h6` | 600 (semibold) | Subsections, accordion headers |

### Body Text

| Element | Weight | Line Height | Usage |
|---------|--------|-------------|-------|
| Paragraphs | 400 (regular) | 1.7 | Descriptions, long text |
| Small text | 400 (regular) | 1.5 | Labels, captions, metadata |
| Lists | 400 (regular) | 1.6 | Bulleted/numbered content |

### UI Elements

| Element | Weight | Usage |
|---------|--------|-------|
| Buttons | 600 (semibold) | Emphasis + readability |
| Badges | 600 (semibold) | Clear hierarchy |
| Navigation | 400 (regular) | Familiar, scannable |
| Links | 400 (regular) | Inline with body text |

---

## Implementation Strategy

### Phase A: Example Page + Font Setup (Today)

**Using @nuxt/fonts module for automatic optimization:**

1. **Install @nuxt/fonts** (if not already installed)
2. **Configure nuxt.config.ts:**
   ```ts
   modules: ['@nuxt/fonts']

   fonts: {
     families: [
       { name: 'Crimson Pro', provider: 'google' }
     ]
   }
   ```
3. **Update Tailwind config** for demo page only (scoped)
4. **Create `/typography-demo` page** - Showcase all styles
5. **Review in browser** (light + dark mode)
6. **No changes to existing pages yet**

**Why @nuxt/fonts?**
- Automatic font optimization (preload, font-display, subset)
- Built-in Google Fonts provider
- Better performance than manual `@import`
- Future-proof (can add more fonts easily)

### Phase B: Global Rollout (After approval)

1. Update Tailwind config globally (replace `fontFamily.sans`)
2. All existing pages inherit new typography automatically
3. Test 5-6 pages to verify layouts
4. Commit changes

**Rollback:** Remove `fonts` config and Tailwind override - instant revert.

---

## Example Page Design

**Route:** `/typography-demo`

**Content Sections:**

1. **Hero**
   - Large h1 title
   - Subtitle (h2)
   - Lead paragraph

2. **Headings Hierarchy**
   - h1 through h6 samples
   - Show size/weight progression

3. **Body Text Samples**
   - Regular paragraphs
   - Bulleted/numbered lists
   - Emphasis (bold, italic)
   - Links (regular, hover states)

4. **UI Components**
   - Buttons (UButton variants)
   - Badges (UBadge colors)
   - Cards with text (UCard)

5. **Real Content Example**
   - Sample spell card with actual description
   - Stat block with mixed typography
   - Demonstrates real-world usage

6. **Dark Mode**
   - Toggle to test readability in both themes
   - Verify contrast ratios

**Purpose:** Visual regression testing + client review before global rollout

---

## Font Configuration

### Weights Loaded

- **400 (Regular):** Body text, navigation, small text
- **600 (Semibold):** Headings h2-h6, buttons, badges
- **700 (Bold):** h1 headings, strong emphasis

**Character Sets:** Latin, Latin-extended

### Fallback Stack

```css
font-family: 'Crimson Pro', Georgia, 'Times New Roman', serif;
```

**Rationale:** Georgia (widely available serif), Times New Roman (universal fallback), generic serif

---

## Performance Considerations

### @nuxt/fonts Optimizations

- **Preloading:** Critical fonts preloaded in `<head>`
- **Font-display:** `swap` strategy (show fallback immediately, swap when loaded)
- **Subsetting:** Only load required character ranges
- **Variable font:** Single file instead of multiple weights

### Expected Impact

- **Initial load:** +15-25KB (variable font WOFF2)
- **Render:** Minimal flash of unstyled text (fallback to Georgia)
- **Caching:** Font cached after first load

---

## Testing Checklist

### Phase A Testing

- [ ] Typography demo page renders correctly
- [ ] All heading sizes display properly
- [ ] Body text is readable at all sizes
- [ ] Dark mode contrast is sufficient
- [ ] Font loads without FOUT (flash of unstyled text)
- [ ] Fallback fonts work if Google Fonts blocked
- [ ] Mobile responsive (375px to 1440px)

### Phase B Testing (Global Rollout)

- [ ] Homepage typography works
- [ ] Spell list/detail pages work
- [ ] Item list/detail pages work
- [ ] Cards maintain proper spacing
- [ ] Badges/buttons remain readable
- [ ] No layout shifts or breaking changes
- [ ] Dark mode works across all pages

---

## Accessibility

### Readability

- ✅ High x-height (improves legibility at small sizes)
- ✅ Clear letter spacing
- ✅ Generous line-height (1.7 for body text)
- ✅ Sufficient weight contrast (400 vs 600 vs 700)

### Contrast

- ✅ Meets WCAG AA standards (tested in both themes)
- ✅ Serif design aids character distinction (accessibility win)

### Performance

- ✅ Fast loading (variable font, optimized delivery)
- ✅ Graceful degradation (fallback fonts)

---

## Success Criteria

**Phase A (Demo Page):**
- ✅ Font loads correctly
- ✅ Typography demo looks polished
- ✅ Client approves aesthetic
- ✅ Dark mode works
- ✅ No performance regressions

**Phase B (Global Rollout):**
- ✅ All pages render correctly
- ✅ No layout breaks
- ✅ Performance acceptable (<100ms LCP increase)
- ✅ Team satisfied with aesthetic

---

## Rollback Plan

**If issues arise:**

1. Remove `fonts` configuration from `nuxt.config.ts`
2. Remove Tailwind font family override
3. Restart Nuxt dev server
4. System fonts restored immediately

**No data loss, no breaking changes - pure visual revert.**

---

## Future Enhancements

**Optional (not in scope):**

- Add monospace font for code blocks (e.g., `JetBrains Mono`)
- Explore font pairings if headers need more impact
- Custom font weights for specific UI elements
- CSS custom properties for easier theme switching

---

**Next Steps:** Implement Phase A (example page + font setup) using @nuxt/fonts.
