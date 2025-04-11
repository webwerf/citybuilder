# Tech Context

## Technologies Used

- **Vue 3**: Main frontend framework, using the Composition API and Single File Components (SFC).
- **Vite**: Development server and build tool for fast HMR and modern JS support.
- **HTML5 Canvas**: For rendering the isometric grid, tiles, and labels.
- **JavaScript (ES6+)**: All logic and rendering code.
- **localStorage**: For persistent client-side storage of the city state.

## Development Setup

- Project is bootstrapped with Vite and Vue 3.
- All source code is in the `src/` directory.
- Texture assets are in `public/textures/`.
- The main entry point is `src/App.vue`.
- Context menu logic is in `src/components/ContextMenu.vue`.
- Memory bank documentation is in the `memory-bank/` directory.

## Technical Constraints

- No backend or server-side storage; all data is local to the browser.
- Must work in all modern browsers.
- All user-facing text is in Dutch.
- No external dependencies beyond Vue and Vite.

## Dependencies

- `"vue"`: ^3.x
- `"vite"`: ^6.x
- `"@vitejs/plugin-vue"`: ^5.x

## Tool Usage Patterns

- **Canvas**: Two layers (`bg` for tiles, `fg` for labels/hover).
- **localStorage**: State is encoded as Base64 JSON and stored under the key `isocityMapData`.
- **Context Menu**: Custom Vue component for right-click actions.
- **Memory Bank**: Markdown files for requirements, context, architecture, and progress.