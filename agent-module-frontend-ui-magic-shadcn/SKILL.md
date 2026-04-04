---
name: agent-module-frontend-ui-magic-shadcn
description: "Module: Frontend UI Implementation with Shadcn and Magic UI"
---
# Module: Frontend UI Implementation with Shadcn and Magic UI

## 1. Module Metadata
- Module ID: `agent.module.frontend-ui-magic-shadcn`
- Version: `1.0.0`
- Maturity: `production`
- Scope: Guides agents on generating, installing, configuring, and using Shadcn and Magic UI components natively in Next.js + Tailwind web applications.
- Primary outcomes:
  - Standardized setup of Tailwind CSS, Shadcn architecture, and Magic UI tokens.
  - Reproducible premium component generation using CLI and MCP native tools.
  - Consistent and robust aesthetic across UI components.

## 2. Mission and Applicability
Use this module when generating rich, highly animated, and premium frontend interfaces in Next.js. 

Apply when:
- Creating new visual components, dashboards, or landing pages.
- Adding complex micro-animations or modern glassmorphism designs.
- Interacting with UI design tools (e.g., Shadcn CLI, Magic UI MCP Context).

Do not apply when:
- Working on backend node APIs or pure data orchestration.
- The user requests a basic vanilla HTML/CSS implementation without a build step.

## 3. Architecture Pattern
- Pattern: `Atomic Component Architecture (Next.js + Tailwind + Framer Motion + Shadcn + Magic UI)`
- Core design rules:
  - **Component Isolation**: All UI primitives should reside within `components/ui/`.
  - **Utility Consolidation**: Use `cn()` helper (clsx + tailwind-merge) for dynamic styling rules (located in `lib/utils` or `lib/utils.ts`).
  - **Theming & Variables**: Maintain design tokens (colors, animations, border radii) as native CSS variables within `app/globals.css`. Do not hardcode absolute magic colors.

## 4. Implementation Workflow

### Phase A: Environment Setup and Initialization
1. Ensure project has Next.js and Tailwind configurations.
2. Initialize Shadcn UI: Run `npx shadcn@latest init` to setup `components.json` and utility files.
3. Integrate Magic UI Configuration:
   - Expand `tailwind.config.js` to include Magic UI keyframes and animations (e.g., `orbit`, `marquee`, `pulse`, `shine`, `aurora`, `shimmer-slide`).
   - Append Magic UI color token variables to `app/globals.css` (e.g., `--color-1` to `--color-5` for rainbow accents).
   - Ensure `framer-motion` and `lucide-react` are installed. Run `npm install framer-motion clsx tailwind-merge` if required.

Exit criteria:
- `components.json`, `tailwind.config.js`, and `app/globals.css` cleanly unify both libraries without compilation errors.
- `/lib/utils.ts` (or equivalent) exports a working `cn` function.

### Phase B: Component Sourcing and Installation
1. Requesting Shadcn Components:
   - Use the terminal/CLI tool to add standard UI primitives.
   - Command: `npx shadcn@latest add <component_name>` (e.g., button, dialog, card, input).
2. Requesting Magic UI Components (Agentic Workflow via MCP):
   - **DO NOT hallucinate Magic UI source code.** 
   - Step 1: Use `mcp_magic-ui_searchRegistryItems` to search and find the correct component identifier.
   - Step 2: Use `mcp_magic-ui_getRegistryItem` with `includeSource: true` to reliably retrieve the exact canonical implementation of the requested Magic UI component from your MCP connection.
   - Step 3: Extract the code from the MCP response and write the retrieved content into `components/ui/<component>.tsx`.
   - Step 4: Add any component-specific dependencies or Tailwind configurations returned by the registry item payload.

Exit criteria:
- The component is physically present in `components/ui/` and is exported cleanly.
- Peer dependencies mapped by the MCP are resolved.

### Phase C: Assembling Views and Animations
1. Import primitives into the feature layout/page inside `app/` or `components/`.
2. Construct advanced premium aesthetic by composing `magic-card`, `bento-grid`, `animated-list`, etc., wrapped around solid Shadcn form elements.
3. Validate responsive behavior across Tailwind modifiers (`sm:`, `md:`, `lg:`).
4. Utilize `glass-effect` class names, `text-gradient`, or other `globals.css` utility layers.

Exit criteria:
- The interface fulfills the user requirements and delivers a wow-factor presentation with smooth micro-animations.

## 5. Decision Framework
| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Core Primitives (Buttons, Forms, Inputs) | Shadcn UI | Custom Tailwind | Standardize accessible interaction elements using Shadcn primitives. |
| Hero Sections, Animations, Eye-candy | Magic UI | Framer Motion Custom | Use Magic UI elements for premium, vibrant layouts. Build custom with Framer Motion only if unsupported. |
| Magic UI Component Retrieval | MCP Tools (`magic-ui`) | Web Scraping/Browsing | Native MCP retrieval guarantees up-to-date accurate source code directly from the Magic UI registry. |

## 6. Validation Strategy
### Functional Validation
- Ensure Framer Motion `<motion.div>` hooks have no hydration mismatches in Next.js (prepend `"use client";` directive at the top of client-side animation components).
- Ensure Shadcn form controls successfully connect with standard react context or hook forms.

### Failure Validation
- Verify that standard dark mode toggling (using `next-themes`) correctly flips `oklch()` or `hsl()` variables inside `app/globals.css`. Fallback gracefully if conditional rendering disrupts Framer Motion layouts.

### Contract/Security/Performance Validation
- Keep excessive animated node counts minimized (e.g., `particles`, `confetti`) to guarantee ~60fps scrolling on lower-end devices.

## 7. Benchmarks and SLO Targets
- Design Fidelity: `Components must align exactly with the design language (fonts, shadow softness, glass effects) established in globals.css.`
- Reusability: `UI components should act purely as visual layers and not contain business logic.`
- MCP Context Accuracy: `Always query the registry before writing component code to prevent hallucinations.`

## 8. Risks and Controls
- Risk: Hallucinating complex Magic UI component code leading to syntax or un-handled runtime errors.
  - Control: Utilize the explicitly configured `magic-ui` MCP connection tools (`mcp_magic-ui_getRegistryItem`) to pull official pristine source files perfectly.
- Risk: Clashing Tailwind configurations or missed dependencies.
  - Control: Rigorously audit `tailwind.config.js` extension maps (keyframes, colors) against the required metadata returned by the Magic UI registry.

## 9. Agent Execution Checklist
- [ ] Determine required visual interactive elements for the page.
- [ ] Auto-generate standard Shadcn components using `npx shadcn@latest add ...` via terminal.
- [ ] For premium animations, search Magic UI registry via MCP: `call:mcp_magic-ui_searchRegistryItems`.
- [ ] Pull specific Magic UI component source code via MCP: `call:mcp_magic-ui_getRegistryItem{name: "<item>", includeSource: true}`.
- [ ] Merge returned Tailwind constraints (`extend` objects, `keyframes`, etc.) into `tailwind.config.js`.
- [ ] Verify imported variables align seamlessly with core themes mapped inside `app/globals.css`.
- [ ] Incorporate components into the UI layout with fluid, responsive structures.

## 10. Reuse Notes
Adapt only:
- Base color tokens in `globals.css` (e.g. overriding `--color-1`) when implementing customized branding.
- The `animation` object inside `tailwind.config.js` to change loop speeds and durations.
