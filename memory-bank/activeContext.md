# Active Context

## Current Work Focus

- Ensuring that tile labels are visible and correctly rendered on the city grid.
- Maintaining robust localStorage persistence for all city state (tiles and labels).
- Providing a clear, Dutch-language UI for all user actions, including map clearing and labeling.

## Recent Changes

- Migrated from URL hash state to localStorage for persistence.
- Added a context menu for tile actions (remove, label).
- Implemented label editing and storage per tile.
- Added a "Kaart Wissen" (Clear Map) button with confirmation and fixed positioning.
- Refactored the rendering logic to draw labels on the foreground canvas (`fgCtx`) after all tiles are drawn.

## Next Steps

- Ensure that all label rendering is robust and visible in all scenarios (after load, after edit, after clear).
- Continue to refine the UI for clarity and accessibility.
- Document and maintain the memory bank structure for ongoing project clarity.

## Active Decisions and Considerations

- All persistent state is now stored in localStorage under the key `isocityMapData`.
- The foreground canvas is used exclusively for label rendering and hover effects.
- The context menu is the primary interface for tile-specific actions.
- The project uses Dutch for all user-facing text.

## Important Patterns and Preferences

- All map/tile state changes (placement, removal, labeling, clearing) trigger a redraw and a save to localStorage.
- The codebase is organized as a Vue 3 SFC with clear separation of rendering, state, and event handling.
- The memory bank is updated after significant changes or at user request.

## Learnings and Project Insights

- Drawing labels on a separate canvas layer ensures they are always visible above tiles.
- Backward compatibility for old state formats is maintained for user convenience.
- UI clarity and minimalism are prioritized for a smooth user experience.