# UI 8-State Lifecycle Specification (Universal Edition)

In world-class digital products (Awwwards / Webby / FWA tier), the essence of superior craftsmanship lies in **meticulous state design and fluid, continuous feedback**. Any interactive element or asynchronous data container must design for and implement the complete 8-state lifecycle closed loop.

---

## 1. Default State (Initial Polished Baseline)
- **Primary Objective**: Establish immediate visual harmony, structured hierarchy, and comfortable breathing room.
- **Implementation Rules**:
  - Containers strictly utilize Design Token surface backgrounds (e.g., `--color-bg-surface`) paired with translucent borders (`--color-border-subtle`).
  - Typography scales and contrast ratios must strictly satisfy accessibility guidelines, distinctly separating headings, body copy, and metadata.
  - Saturated pure colors are prohibited in idle default states.

---

## 2. Hover State (Subtle Elevation & Ambient Glow)
- **Primary Objective**: Provide subtle, organic physical responsiveness upon pointer hover.
- **Implementation Rules**:
  - **GPU Compositing Layer**: Restrict motion to `transform: translateY(-2px) scale(1.01)`. Never animate `top`, `margin`, or box-model properties.
  - **Ambient Diffusion**: Elevate border highlight opacity (`--color-border-highlight`) and introduce subtle accent glow (`--shadow-glow`).
  - **Calibrated Easing**: Bind transitions to `cubic-bezier(0.16, 1, 0.3, 1)` (`--ease-out-expo`) over `200~280ms` to prevent jarring snap-cuts.

---

## 3. Active / Press State (Tactile Micro-Depression)
- **Primary Objective**: Deliver tactile, mechanical key-switch feedback during pointer interaction.
- **Implementation Rules**:
  - Upon pointerdown/click, apply instant micro-contraction: `transform: translateY(0px) scale(0.98)`.
  - Shorten transition duration to `100~150ms` for immediate, crisp tactile responsiveness.

---

## 4. Focus-Visible State (Accessible Keyboard Navigation)
- **Primary Objective**: Combine WCAG 2.1 AA keyboard accessibility with high-end modern aesthetics.
- **Implementation Rules**:
  - Strictly use the `:focus-visible` pseudo-class (preventing intrusive black borders upon normal pointer clicks).
  - Render a dual-ring ambient halo: `outline: 2px solid var(--color-accent-primary); outline-offset: 2px;`.

---

## 5. Skeleton / Loading State (Fluid Shimmer Skeleton)
- **Primary Objective**: Eliminate waiting friction during asynchronous requests, preserving layout geometry and preventing Cumulative Layout Shifts (CLS). Never use standalone crude spinners.
- **Implementation Rules**:
  - Skeleton geometry must replicate final content dimensions (text lines, avatars, media cards) with a 1:1 footprint.
  - Implement fluid gradient shimmer animations:
    ```css
    @keyframes shimmer {
      0% { transform: translateX(-100%); }
      100% { transform: translateX(100%); }
    }
    ```
  - Animate using hardware-accelerated pseudo-elements (`::after`) over the base surface, tuned to a `1.5s` cycle.

---

## 6. Empty State (Tasteful Blank Canvas & Call-To-Action)
- **Primary Objective**: Eliminate generic "No Data Available" dead ends; convert empty moments into engaging user journeys.
- **Implementation Rules**:
  - Feature refined vector line illustrations or abstract geometric ambient lights.
  - Provide concise, encouraging copywriting paired with an actionable Call-To-Action (CTA button, e.g., "Create First Project", "Reset Filters").

---

## 7. Error / Fallback State (Empathetic Self-Healing & Retry)
- **Primary Objective**: When network timeouts, 500 errors, or image 404s occur, offer empathetic feedback and a self-healing pathway.
- **Implementation Rules**:
  - Never expose raw stack traces, database errors, or broken red icons directly to end users.
  - Present an embedded error card with plain-language troubleshooting guidance.
  - Always include an **in-place "Retry" button** equipped with in-flight debounce protection against rapid consecutive clicks.

---

## 8. Disabled State (Polished Inactive Presentation)
- **Primary Objective**: Communicate unreachability clearly while preserving visual depth and grid stability.
- **Implementation Rules**:
  - Gracefully reduce opacity (e.g., `opacity: 0.45;`) and assign `cursor: not-allowed;`.
  - Suppress pointer events on the target element: `pointer-events: none;` (wrap inside a parent container if tooltip explanations are required).
  - Retain subtle border contours so the component does not vanish and destabilize layout rhythm.

---

## 9. State Icon Engineering (Zero-Emoji Standardization)
- **Zero Native Emojis**: Never use native emojis (❌, ⚠️, 📦, 💡, ⏳) as state indicators or functional icons. Native emojis render inconsistently across operating systems and degrade aesthetic credibility.
- **Cross-Platform Vector Rendering**: Use standard inline SVGs or structured vector icon font families (Alibaba Iconfont, Remix Icon, FontAwesome) with `currentColor` inheritance to ensure razor-sharp rendering on all high-DPI displays.
