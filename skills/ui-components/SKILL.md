---
name: ui-components
description: UI component patterns — pure presentation components, withViewModel boundary, ClientOnly, universal ActionButton, cva/cx styling, design tokens
license: MIT
compatibility: opencode
---

# UI Component Patterns

## Component Categories

### 1. Pure Presentation (no VM)
Simple stateless components using props only. **Most components should be this.**

```typescript
import type { HTMLAttributes } from 'react';
import { cx } from 'yummies/css';

type CardProps = HTMLAttributes<HTMLDivElement>;

export function Card({ className, ...props }: CardProps) {
  return (
    <div
      className={cx('rounded-2xl bg-surface p-4 shadow-sm', className)}
      {...props}
    />
  );
}
```

### 2. withViewModel Components (have own VM)
Only when component has significant business logic.

### 3. Client-Only Components
```typescript
export function ClientOnly({ children, fallback }: ClientOnlyProps) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => { setMounted(true); }, []);
  if (!mounted) return <>{fallback}</>;
  return <>{children}</>;
}
```

`ClientOnly` is a canonical acceptable case for `useState`/`useEffect` — pure UI timing in the presentation layer.
Other UI-only hook usage (focus management, local animation state, DOM measurement) is also acceptable as long as no business logic is involved.

## Universal Button Component Pattern

```typescript
// Single component for ALL interactive elements
<ActionButton
  action={handleClick}     // onClick handler
  icon={HeartIcon}         // Icon component or element
  iconClassName="size-4"   // Icon sizing/color overrides
  text="Label"             // Button text
  extraText="Sub-label"    // Secondary text line
  size="m"                 // Size variant
  look="solidBrand"        // Visual variant
  selected={true}          // Toggle/selected state
  loading={true}           // Skeleton loading state
  disabled={true}
  href="/link"             // Renders <a> instead of <button>
  ariaLabel="Description"
  className="extra-class"
>
  {/* Optional custom children override */}
</ActionButton>
```

### Common `look` Variants

| look | Use Case |
|------|----------|
| `solidBrand` | Primary CTA |
| `solidBrandBar` | Full-width bar button |
| `solidSuccess` | Success/checkout action |
| `softBrand` | Soft secondary action |
| `surface` | Surface-level action |
| `ghostNeutral` | Subtle neutral action |
| `ghostRisk` | Destructive action |
| `ghostMuted` | Muted ghost action |
| `outlinePill` | Filter/tag pill |
| `outlineMenu` | Dropdown menu item |
| `outlineCircle` | Circle icon button |
| `linkMuted` | Muted text link |
| `switch` | Toggle switch |
| `mediaRail` | Image/media thumbnail |
| `mediaSwatch` | Color/image swatch |
| `overlayIcon` | Icon overlay on image |
| `insetRow` | Full-width inset row |
| `segment` | Segmented control button |

## Styling Conventions

### Variant-based class composition with yummies/css

```typescript
import { cva, cx } from 'yummies/css';

const buttonCx = cva('base-classes', {
  variants: {
    size: { s: 'text-sm', m: 'text-base', l: 'text-lg' },
    tone: { accent: 'text-red', success: 'text-green' },
  },
  compoundVariants: [
    { size: 'l', tone: 'accent', class: 'text-xl font-bold' },
  ],
  defaultVariants: { size: 'm', tone: 'accent' },
});

// Usage
const className = cx(buttonCx({ size: 'l', tone: 'success' }), 'extra-class');
```

### Theme Design Tokens

Define custom design tokens via Tailwind CSS 4 `@theme`:

```css
@theme {
  --color-brand: #005bff;
  --color-surface: #ffffff;
  --color-bg-base: #f5f5f5;
  --color-text-primary: #1a1a1a;
}
```

### Naming Conventions
- kebab-case for custom CSS variables
- Component class composition via `cx()` from yummies/css
- No CSS modules — all Tailwind utility classes
