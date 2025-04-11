# System Patterns

## System Architecture

- **Frontend:** Vue 3 Single File Component (SFC) using Vite for development and build.
- **Rendering:** Two stacked HTML5 canvas elements (`bg` for tiles, `fg` for labels and hover effects).
- **State Management:** Reactive `map` array in Vue, persisted to localStorage as a Base64-encoded JSON string.
- **Persistence:** All city state (tiles, labels) is saved and loaded from localStorage (`isocityMapData`).

## Key Technical Decisions

- **Canvas Layering:** Tiles are drawn on the background canvas; labels and hover effects are drawn on the foreground canvas for clarity and separation of concerns.
- **Context Menu:** All tile actions (remove, label) are handled via a custom context menu component.
- **Backward Compatibility:** The loader can parse both the new JSON format and the legacy base64 format for map data.
- **Dutch Language:** All user-facing text and prompts are in Dutch.

## Design Patterns in Use

- **Separation of Concerns:** Rendering, state management, and UI logic are clearly separated.
- **Reactive Updates:** All state changes trigger redraws and persistence.
- **Progressive Enhancement:** New features (labels, context menu, localStorage) are layered on top of a simple, robust core.

## Component Relationships

- **App.vue:** Main SFC, manages state, rendering, and event handling.
- **ContextMenu.vue:** Handles right-click actions for tiles.
- **Memory Bank:** Markdown files document requirements, context, architecture, and progress.

## Critical Implementation Paths

- **Tile Placement:** User selects a tool, clicks/drag to place tiles, triggers state update, redraw, and save.
- **Label Editing:** User right-clicks a tile, selects "Label toevoegen/bewerken", enters label, triggers state update, redraw, and save.
- **Map Clearing:** User clicks "Kaart Wissen", confirms, map is reset, redraw and save occur.
- **State Loading:** On page load, state is loaded from localStorage and the map/labels are drawn.