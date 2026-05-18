# Agent 365 MCP App — Issue List (Prioritized)

Pre-public-release cleanup checklist. Estimated remaining effort: **~2–4 hours** for P1, **+2–3 hours** for P2.

**Resolved:** sample-template placeholders, `env/.env.dev` gitignored, lifecycle yaml uses variable, widget debug logs removed, verbose `ontoolresult` logs removed, `test-graph.ts` deleted, "180 agents" → "50 agents", Graph endpoints corrected, TODO comment removed, OBO/auth section fully rewritten, widget lifecycle docs added. **All P0 blockers cleared.**

---

## P1 — Quality issues (should fix to look like a "reference implementation")

The sample works without these, but they reduce its value as something others copy from.

### P1a — Widget duplication (biggest single win: ~250 LoC removed)

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 12 | "Assign Owner" people-picker panel duplicated verbatim | [AgentRegistry.tsx:737-823](src/widgets/agent-registry/AgentRegistry.tsx#L737-L823), [AgentMap.tsx:581-638](src/widgets/agent-map/AgentMap.tsx#L581-L638) | Extract `shared/AssignOwnerPanel.tsx`. Includes ~90 LoC of UI + ~7 pieces of state + debounce logic. |
| 13 | `StatCard` + stat styles duplicated | [AgentRegistry.tsx:96-302](src/widgets/agent-registry/AgentRegistry.tsx#L96-L302), [RiskyAgents.tsx:64-203](src/widgets/risky-agents/RiskyAgents.tsx#L64-L203) | Extract `shared/StatCard.tsx` and `shared/gridStyles.ts`. |
| 14 | Pagination footer duplicated | [AgentRegistry.tsx:690-731](src/widgets/agent-registry/AgentRegistry.tsx#L690-L731), [RiskyAgents.tsx:439-480](src/widgets/risky-agents/RiskyAgents.tsx#L439-L480) | Extract `shared/Pagination.tsx`. |
| 15 | MessageBar notification pattern duplicated | AgentRegistry + AgentMap | Extract `shared/useNotification.ts` hook. |

### P1b — Dead code

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 16 | Unused shared icon (registry defines its own) | [shared/Agent365Icon.tsx](src/widgets/shared/Agent365Icon.tsx) | Delete the shared file, or replace the local definition in [AgentRegistry.tsx:70-72](src/widgets/agent-registry/AgentRegistry.tsx#L70-L72) with the shared one. |
| 17 | `getThemeColors` imported but never called | [AgentRegistry.tsx:64](src/widgets/agent-registry/AgentRegistry.tsx#L64), [shared/theme.ts](src/widgets/shared/theme.ts) | Remove the import; delete the helper if unused everywhere. |
| 18 | Unused Fluent imports | [AgentRegistry.tsx:13,51-55](src/widgets/agent-registry/AgentRegistry.tsx#L13) | Remove `ShieldCheckmarkRegular`, `PeopleRegular`, `WarningRegular`, `ProhibitedRegular`, `Card`. |
| 19 | Empty toolbar slot | [AgentRegistry.tsx:632](src/widgets/agent-registry/AgentRegistry.tsx#L632) | Delete `<div className={styles.toolbarLeft} />`. |
| 20 | Bug: `onBlock` callback ignores its `id` parameter | [AgentRegistry.tsx:381](src/widgets/agent-registry/AgentRegistry.tsx#L381) | Change to `onBlock={() => handleBlock(selectedAgent)}` — the `(id) =>` form is misleading since `id` is unused. |

### P1c — Server-side

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 21 | `load_agent_info` tool missing from plugin manifest | [appPackage/agent365-plugin.json](appPackage/agent365-plugin.json) | Add `load_agent_info` to `functions` and `run_for_functions`, matching the registration in [server.ts:152](server.ts#L152). |
| 22 | Noisy emoji logs throughout server | [main.ts:20,28,60,67](main.ts), [src/graph/packages.ts](src/graph/packages.ts), [src/tools/get-agent-detail.ts](src/tools/get-agent-detail.ts) | Gate behind `DEBUG` env var, or strip emojis. Production samples shouldn't ship `🚀`/`🔐`/`📨`. |
| 23 | Auth token length leaked to logs | [main.ts:28](main.ts#L28) | Drop the `(${incomingToken.length} chars)` detail — log "present" / "missing" only. |
| 24 | N+1 Graph calls in `getAgentDetailsMap` | [src/graph/packages.ts:189-210](src/graph/packages.ts#L189-L210) | `load_agent_info` already exists for lazy loading. Remove the eager preload, or document the tradeoff with a comment. Bad pattern to demonstrate in a sample. |
| 25 | Duplicate `elementDetails` parsing | [src/graph/packages.ts:150-183](src/graph/packages.ts#L150-L183), [src/types.ts:261-298](src/types.ts#L261-L298) | Extract a single `parseElementDetails(pkg)` helper into types.ts. |
| 26 | Cross-request module-level cache | [src/graph/packages.ts:10](src/graph/packages.ts#L10) | Add a comment noting the single-tenant assumption, or key by tenant from the JWT. |
| 27 | Mock `reassignAgent` returns generic "New Owner" | [src/mock/packages.ts](src/mock/packages.ts) (in `reassignAgent` helper) | Look up the user by `newOwnerId` in `MOCK_USERS` for a more realistic demo. |

### P1d — Widget styling

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 28 | Hardcoded hex colors break dark mode | [AgentRegistry.tsx:182-186,226,240,552](src/widgets/agent-registry/AgentRegistry.tsx), [AgentMap.tsx:261,544](src/widgets/agent-map/AgentMap.tsx) | Replace `#E0E0E0`, `#FFFFFF`, `#242424`, `#fafafa` with Fluent `tokens.colorNeutralStroke2` / `tokens.colorNeutralBackground1` / etc. |
| 29 | Dozens of inline `style={{}}` blobs | [AgentRegistry.tsx:552,559-563,593,744,812](src/widgets/agent-registry/AgentRegistry.tsx) and similar | Move repeated layout patterns into `makeStyles`. Bad look for a sample showcasing Fluent. |
| 30 | `any` casts in tool-result parsing (6 sites) | [AgentRegistry.tsx:346,403,674](src/widgets/agent-registry/AgentRegistry.tsx), [AgentMap.tsx:149,190,369](src/widgets/agent-map/AgentMap.tsx) | Add a typed `parseToolResult<T>(result): T \| null` helper to [McpAppProvider.tsx](src/widgets/shared/McpAppProvider.tsx) — exactly the kind of utility a sample should demonstrate. |
| 31 | Inconsistent error handling | [AgentMap.tsx:192,372](src/widgets/agent-map/AgentMap.tsx) swallow with `catch {}`; AgentRegistry surfaces via notifications | Pick one pattern. Prefer surfacing via the shared `useNotification` from #15. |

---

## P2 — Polish (nice-to-haves)

### P2a — Oversized widget files

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 32 | `AgentMap.tsx` ~1300 lines | [src/widgets/agent-map/AgentMap.tsx](src/widgets/agent-map/AgentMap.tsx) | Split into `agent-map/icons.ts` (the embedded SVG strings on lines 50,53,56,59,62), `agent-map/renderMap.ts` (pure D3 function taking `(svg, hierarchy, agents, callbacks)`), and the React component. The single `useEffect` at lines 227–457 is 225 LoC on its own. |
| 33 | `AgentRegistry.tsx` ~840 lines | [src/widgets/agent-registry/AgentRegistry.tsx](src/widgets/agent-registry/AgentRegistry.tsx) | After P1a extractions, also pull the inline list view block at lines 516–587 into `RegistryInlineView.tsx`. Should shrink the main file to ~300 LoC. |

### P2b — A11y

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 34 | AgentMap SVG missing `role="img"` and `aria-label` | [AgentMap.tsx](src/widgets/agent-map/AgentMap.tsx) | Add ARIA attributes; provide a text alternative. |
| 35 | Zoom/pan is mouse-only | [AgentMap.tsx](src/widgets/agent-map/AgentMap.tsx) | Add keyboard handlers for zoom (`+`/`-`) and pan (arrow keys). |
| 36 | DataGrid rows have `onClick` but no keyboard handler | [AgentRegistry.tsx:663-683](src/widgets/agent-registry/AgentRegistry.tsx#L663-L683) | Add `role="button"`, `tabIndex={0}`, and `onKeyDown` handler for Enter/Space. |

### P2c — Performance

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 37 | Column definitions rebuilt every render | [AgentRegistry.tsx:436](src/widgets/agent-registry/AgentRegistry.tsx#L436), [RiskyAgents.tsx:236](src/widgets/risky-agents/RiskyAgents.tsx#L236) | Wrap in `useMemo`. The closure over `hoveredRowId` is the reason — consider moving the icon-swap to CSS `:hover` instead. |
| 38 | D3 cleanup gap on unmount | [AgentMap.tsx:227-457](src/widgets/agent-map/AgentMap.tsx#L227-L457) | Return `() => { d3.select(svg).on(".zoom", null); }` from the effect. |

### P2d — Misc

| # | Issue | Location | Action |
|---|-------|----------|--------|
| 39 | `as any` casts on tool handlers (no comment) | [server.ts:56,129,138,148,157](server.ts) | Add a one-line comment explaining the SDK typing workaround, or define a small generic wrapper. |
| 40 | Inspector script not pinned | [package.json:26](package.json#L26) | Pin `@modelcontextprotocol/inspector` to a specific version for reproducibility. |
| 41 | `manifest.json` uses `www.example.com` URLs | [appPackage/manifest.json:8-10](appPackage/manifest.json#L8) | Either swap for real privacy/terms URLs or call this out in README as "replace before production." |
| 42 | `vite.config.ts` error message could be clearer | [vite.config.ts:5](vite.config.ts#L5) | When `INPUT` env var is missing, mention `build-ui.mjs` as the entry point. |
| 43 | Verify LICENSE year | [LICENSE](LICENSE) | Currently dated 2026 — confirm intentional. |

---

## Suggested execution order

1. **Before going public (~2 hrs)**: P1a (#12–15) — widget duplication extractions. Biggest single code quality win.
2. **Before going public (~30 min)**: P1b (#16–20) — dead code cleanup. Easy and high-signal.
3. **Polish pass (~1 hr)**: P1c + P1d as time allows.
4. **Post-launch**: P2 as feedback arrives.
