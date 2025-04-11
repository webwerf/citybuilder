<template>
  <div id="main"> <!-- Revert to #main as root -->
    <div id="tools" ref="toolsContainer">
      <div
        v-for="(toolItem, index) in toolItems"
        :key="index"
        :id="`tool_${index}`"
        :style="{ backgroundPosition: toolItem.bgPos }"
        :class="{ selected: activeToolId === `tool_${index}` }"
        @click="selectTool(toolItem.coords, `tool_${index}`)"
      ></div>
    </div>
    <!-- Button moved outside #main -->
    <div id="area">
      <canvas id="bg" ref="bgCanvas"></canvas>
      <canvas
        id="fg"
        ref="fgCanvas"
        @mousemove="viz"
        @contextmenu.prevent="handleContextMenu"
        @mousedown.left.prevent="startPlacing"
        @touchstart.prevent="startPlacing"
        @mouseup="stopPlacing"
        @touchend="stopPlacing"
        @mouseleave="stopPlacing"
      ></canvas>
      <ContextMenu
        :visible="contextMenuVisible"
        :x="contextMenuX"
        :y="contextMenuY"
        :items="contextMenuItems"
        @item-click="handleContextMenuItemClick"
      />
    </div>
    <!-- Button placed inside #main but outside #tools/#area -->
    <button @click="clearMap" class="clear-button">Kaart Wissen</button>
 </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed, nextTick } from 'vue';
import ContextMenu from './components/ContextMenu.vue';

// --- Refs for DOM elements ---
const bgCanvas = ref(null);
const fgCanvas = ref(null);
const toolsContainer = ref(null);

// --- Reactive State ---
const map = reactive([]);
const tool = reactive([0, 0]); // Currently selected tool [row, col]
const activeToolId = ref(null); // ID of the selected tool div
const isPlacing = ref(false);
// const previousState = ref(null); // No longer needed for localStorage

// --- Context Menu State ---
const contextMenuVisible = ref(false);
const contextMenuX = ref(0);
const contextMenuY = ref(0);
const contextMenuTile = ref({ x: -1, y: -1 }); // Store coords of right-clicked tile
const contextMenuItems = ref([
  { label: 'Label toevoegen/bewerken', action: 'edit-label' },
  { label: 'Verwijderen', action: 'delete' }
]);

// --- Constants and Configuration ---
const ntiles = 7;
const tileWidth = 128;
const tileHeight = 64;
const texColumns = 12; // Renamed from texWidth for clarity
const texRows = 6;    // Renamed from texHeight for clarity
const textureSrc = '/textures/01_130x66_130x230.png'; // Path from public dir
const canvasWidth = 910;
const canvasHeight = 666; // Original canvas height
const drawingAreaWidth = 910; // w in original code
const drawingAreaHeight = 462; // h in original code (used for clearing)

// --- Canvas Contexts ---
let bgCtx = null;
let fgCtx = null;
let texture = null;

// --- Computed Properties ---
const toolItems = computed(() => {
  const items = [];
  let count = 0;
  for (let i = 0; i < texRows; i++) {
    for (let j = 0; j < texColumns; j++) {
      items.push({
        id: `tool_${count++}`,
        coords: [i, j],
        bgPos: `-${j * 130 + 2}px -${i * 230}px` // +2 for border offset? Check original CSS
      });
    }
  }
  return items;
});

// --- Lifecycle Hooks ---
onMounted(() => {
  texture = new Image();
  texture.onload = init;
  texture.src = textureSrc;

  // Remove popstate listener for URL hash

  // Global click listener to hide context menu
  window.addEventListener('click', handleClickOutside);
});

// --- Cleanup ---
// onUnmounted is implicitly handled in <script setup> for listeners added in onMounted
// If not using <script setup>, explicitly remove listener:
// onUnmounted(() => {
//   window.removeEventListener('click', handleClickOutside);
// });

// --- Initialization ---
const init = () => {
  // Initialize map
  for (let i = 0; i < ntiles; i++) {
    map[i] = [];
    for (let j = 0; j < ntiles; j++) {
      map[i][j] = [0, 0, '']; // Add empty label string
    }
  }

  // Setup Background Canvas
  const bg = bgCanvas.value;
  bg.width = canvasWidth;
  bg.height = canvasHeight;
  bgCtx = bg.getContext("2d");
  bgCtx.translate(drawingAreaWidth / 2, tileHeight * 2);

  // Setup Foreground Canvas
  const fg = fgCanvas.value;
  fg.width = canvasWidth;
  fg.height = canvasHeight;
  fgCtx = fg.getContext('2d');
  fgCtx.translate(drawingAreaWidth / 2, tileHeight * 2);

  // Load initial state from URL hash if present
  loadStateFromLocalStorage(); // Load state from localStorage
  drawMap();

  // Select first tool by default? Or restore selection?
  // For now, no tool selected initially.
};

// --- Drawing Functions ---
const drawMap = () => {
  if (!bgCtx || !texture || !texture.complete) return;
  bgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2); // Use calculated w/h
  for (let i = 0; i < ntiles; i++) {
    for (let j = 0; j < ntiles; j++) {
      if (map[i] && map[i][j]) { // Ensure map entry exists
        drawImageTile(bgCtx, i, j, map[i][j][0], map[i][j][1]);
      }
    }
  }
  // After drawing the map, clear and redraw all labels on the foreground canvas
  if (fgCtx) {
    fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2);
    drawLabels();
  }
};

const drawTile = (ctx, x, y, color) => {
  ctx.save();
  ctx.translate((y - x) * tileWidth / 2, (x + y) * tileHeight / 2);
  ctx.beginPath();
  ctx.moveTo(0, 0);
  ctx.lineTo(tileWidth / 2, tileHeight / 2);
  ctx.lineTo(0, tileHeight);
  ctx.lineTo(-tileWidth / 2, tileHeight / 2);
  ctx.closePath();
  ctx.fillStyle = color;
  ctx.fill();
  ctx.restore();
};

const drawImageTile = (ctx, x, y, texRow, texCol) => {
  if (!texture || !texture.complete) return;

  // Get the label for this tile
  const label = (map[x] && map[x][y] && map[x][y][2]) ? map[x][y][2] : '';

  ctx.save();
  ctx.translate((y - x) * tileWidth / 2, (x + y) * tileHeight / 2);

  // Draw the image tile
  const sourceX = texCol * 130;
  const sourceY = texRow * 230;
  const sourceWidth = 130;
  const sourceHeight = 230;
  const drawX = -sourceWidth / 2;
  const drawY = -130; // Base offset for the image
  ctx.drawImage(texture, sourceX, sourceY, sourceWidth, sourceHeight, drawX, drawY, sourceWidth, sourceHeight);

  // Draw the label if it exists
  if (label) {
    ctx.font = '12px Arial'; // Set font style
    ctx.fillStyle = 'black'; // Set text color
    ctx.textAlign = 'center'; // Center text horizontally
    ctx.fillText(label, 0, tileHeight + 5); // Draw text below the tile base (adjust y offset as needed)
  }

  ctx.restore();
};

// --- Event Handlers ---
const getPosition = (e) => {
  // Use offsetX/offsetY relative to the target canvas (fgCanvas)
  // This mirrors the original implementation's logic more closely.
  const offsetY = e.offsetY;
  const offsetX = e.offsetX;

  // Apply the original calculation logic which seemed to work
  const _y = (offsetY - tileHeight * 2) / tileHeight;
  const _x = offsetX / tileWidth - ntiles / 2; // Original used ntiles/2

  const tileX = Math.floor(_y - _x);
  const tileY = Math.floor(_x + _y);

  return { x: tileX, y: tileY };
};


const viz = (e) => {
  if (!fgCtx) return;
  const pos = getPosition(e);

  // Clear previous hover indicator
  fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2);

  // Draw hover indicator if a tool is selected
  if (activeToolId.value !== null && pos.x >= 0 && pos.x < ntiles && pos.y >= 0 && pos.y < ntiles) {
    drawTile(fgCtx, pos.x, pos.y, 'rgba(0,0,0,0.2)');
  }

  // If currently placing (dragging), place tile at current position
  if (isPlacing.value) {
    placeTileAtEvent(e);
  }
};

// Renamed from handlePrimaryClick - Places a tile based on event coordinates
const placeTileAtEvent = (e) => {
   // Check if it's a primary action (needed for the initial call in viz during drag)
   // For mousedown/touchstart, the check is implicit in the event binding
   const isPrimaryAction = e.type === 'mousemove' || e.which === 1 || e.button === 0 || e.type.startsWith('touch') || e.type.startsWith('pointer');
   if (!fgCtx || !isPrimaryAction) return;

   const pos = getPosition(e);
   if (pos.x >= 0 && pos.x < ntiles && pos.y >= 0 && pos.y < ntiles) {
     // Avoid re-placing the same tile repeatedly on fast mousemove
     // Place new tile only if it's different or has no label yet
     const currentTile = map[pos.x][pos.y];
     if (currentTile[0] !== tool[0] || currentTile[1] !== tool[1] || currentTile[2] === undefined || currentTile[2] === '') {
        map[pos.x][pos.y] = [tool[0], tool[1], '']; // Place with empty label
        drawMap();
        saveStateToLocalStorage();
        drawLabels();
        drawLabels(); // Redraw labels after placing a tile
     }
   }
};

// Starts the placing process (on mousedown/touchstart)
const startPlacing = (e) => {
  // Hide context menu if visible
  if (contextMenuVisible.value) {
    contextMenuVisible.value = false;
  }
  // Only start placing if a tool is selected
  if (activeToolId.value === null) return;

  const isPrimaryAction = e.which === 1 || e.button === 0 || e.type.startsWith('touch') || e.type.startsWith('pointer');
  if (!isPrimaryAction) return; // Should be redundant due to event modifiers, but safe check

  isPlacing.value = true;
  placeTileAtEvent(e); // Place the first tile immediately
};

// Stops the placing process (on mouseup/touchend/mouseleave)
const stopPlacing = () => {
  if (isPlacing.value) {
    isPlacing.value = false;
  }
};

// Removed handlePrimaryClick function

const handleContextMenu = (e) => {
  stopPlacing(); // Ensure placing stops when context menu opens
  const pos = getPosition(e);
  if (pos.x >= 0 && pos.x < ntiles && pos.y >= 0 && pos.y < ntiles) {
    // Check if the tile is not empty (grass [0,0])
    if (map[pos.x][pos.y][0] !== 0 || map[pos.x][pos.y][1] !== 0) {
        contextMenuTile.value = { x: pos.x, y: pos.y };
        contextMenuX.value = e.offsetX; // Use offset relative to the canvas
        contextMenuY.value = e.offsetY; // Use offset relative to the canvas
        // Use nextTick to ensure any previous menu is hidden before showing the new one
        contextMenuVisible.value = false;
        nextTick(() => {
          contextMenuVisible.value = true;
        });
    } else {
        // Optionally hide if clicking on empty tile
        contextMenuVisible.value = false;
    }
  } else {
    contextMenuVisible.value = false; // Hide if clicking outside grid
  }
};

const handleContextMenuItemClick = (action) => {
  const { x, y } = contextMenuTile.value;
  if (x === -1 || y === -1 || !map[x] || !map[x][y]) {
      contextMenuVisible.value = false;
      return;
  }

  if (action === 'delete') {
    map[x][y] = [0, 0, '']; // Reset tile with empty label
    drawMap();
    saveStateToLocalStorage();
    drawLabels();
    drawLabels(); // Redraw labels after deleting a tile
  } else if (action === 'edit-label') {
    const currentLabel = map[x][y][2] || '';
    const newLabel = prompt('Voer label in:', currentLabel);
    if (newLabel !== null) { // prompt returns null if cancelled
      map[x][y][2] = newLabel;
      drawMap(); // Redraw map to show the updated label
      saveStateToLocalStorage(); // Save the new label state
      drawLabels();
      drawLabels(); // Redraw labels after editing
    }
  }
  contextMenuVisible.value = false; // Hide menu after action
};

const handleClickOutside = (e) => {
  // Check if the click was outside the ContextMenu component
  // This requires the ContextMenu to stop propagation of its own clicks
  if (contextMenuVisible.value) {
      // A simple check: if the click is not on the canvas, hide.
      // More robust check would involve checking if e.target is inside the menu element.
      if (e.target !== fgCanvas.value) {
          contextMenuVisible.value = false;
      }
  }
};

// Removed unclick function (replaced by stopPlacing)

const selectTool = (coords, toolId) => {
  tool[0] = coords[0];
  tool[1] = coords[1];
  activeToolId.value = toolId;
};

// --- State Management (URL Hash) ---
// Helper to encode map data to Base64 JSON string
const encodeMapData = (mapData) => {
  try {
    const jsonString = JSON.stringify(mapData);
    return btoa(jsonString); // Base64 encode the JSON string
  } catch (e) {
    console.error("Failed to encode map data:", e);
    return '';
  }
};

// Original Base64 decoder for fallback
const FromBase64_Legacy = (str) => {
  try {
    return atob(str).split('').map(c => c.charCodeAt(0));
  } catch (e) {
    console.error("Failed to decode legacy base64 state:", e);
    return null; // Return null on error
  }
};

// Helper to decode Base64 JSON string back to map data (New Format)
const decodeMapData = (base64String) => {
  try {
    const jsonString = atob(base64String); // Decode Base64
    const parsedData = JSON.parse(jsonString); // Parse JSON
    // Basic validation: ensure it's an array of arrays
    if (Array.isArray(parsedData) && parsedData.every(row => Array.isArray(row))) {
        // Ensure each tile has [row, col, label] format, adding default label if missing
        return parsedData.map(row =>
            row.map(tile =>
                Array.isArray(tile) && tile.length >= 2
                    ? [tile[0] || 0, tile[1] || 0, tile[2] || '']
                    : [0, 0, ''] // Default empty tile if format is wrong
            )
        );
    }
    console.warn("Decoded map data has invalid structure.");
    return null; // Return null if structure is invalid
  } catch (e) {
    // Don't log error here, as it's expected for old format
    // console.error("Failed to decode map data as JSON:", e);
    return null; // Return null on JSON parsing error (likely old format)
  }
};

const saveStateToLocalStorage = () => {
  const state = encodeMapData(map); // Encode the whole map structure
  if (state) {
    try {
      localStorage.setItem('isocityMapData', state);
      // console.log("Map state saved to localStorage.");
    } catch (e) {
      console.error("Failed to save map state to localStorage:", e);
    }
  }
};

const loadStateFromLocalStorage = () => {
  const state = localStorage.getItem('isocityMapData');
  let loadedSuccessfully = false;

  if (state) {
    // Try decoding with the new JSON method first
    const decodedMapNew = decodeMapData(state);
    if (decodedMapNew) {
      // Check dimensions match
      if (decodedMapNew.length === ntiles && decodedMapNew.every(row => row.length === ntiles)) {
          map.length = 0; // Clear existing reactive array
          decodedMapNew.forEach(row => map.push(row)); // Push decoded rows
          // previousState.value = state; // Not needed
          loadedSuccessfully = true;
          drawMap(); // Draw map immediately after successful load
      } else {
          console.error("Loaded map dimensions (new format) do not match configuration.");
      }
    } else {
      // If JSON decoding failed, try the legacy method
      console.warn("Failed to decode as JSON, attempting legacy decode...");
      const u8 = FromBase64_Legacy(state);
      if (u8 && u8.length === ntiles * ntiles) {
          map.length = 0; // Clear existing reactive array
          let c = 0;
          for (let i = 0; i < ntiles; i++) {
              const newRow = [];
              for (let j = 0; j < ntiles; j++) {
                  const t = u8[c++];
                  const texRow = Math.trunc(t / texColumns);
                  const texCol = Math.trunc(t % texColumns);
                  newRow.push([texRow, texCol, '']); // Add empty label for old format
              }
              map.push(newRow);
          }
          // previousState.value = state; // Not needed
          loadedSuccessfully = true;
          console.log("Successfully loaded map using legacy format.");
          drawMap(); // Draw map immediately after successful legacy load
          // Optional: Immediately save in the new format?
          // saveStateToLocalStorage(); // Use the correct save function name
      } else {
          console.error("Failed to decode map state using legacy method or dimensions mismatch.");
      }
    }
  }

  // If loading failed or no state was provided, initialize the map
  if (!loadedSuccessfully) {
      console.log("Initializing default map.");
      // Ensure map is initialized correctly if it wasn't loaded
      if (map.length !== ntiles || map[0].length !== ntiles || map[0][0].length !== 3) {
          init(); // Call init which sets default structure
      }
  }
};

const clearMap = () => {
    // Confirmation dialog in Dutch
    if (confirm('Weet u zeker dat u de hele kaart wilt wissen? Dit kan niet ongedaan gemaakt worden.')) {
        // Manually reset the map array to its initial empty state
        map.length = 0; // Clear the existing reactive array first
        for (let i = 0; i < ntiles; i++) {
            const newRow = [];
            for (let j = 0; j < ntiles; j++) {
                newRow.push([0, 0, '']); // Add default empty tile [row, col, label]
            }
            map.push(newRow); // Push the new row into the reactive array
        }
        // Ensure canvas contexts are available before drawing
        if (bgCtx && fgCtx) {
            drawMap(); // Redraw the cleared map
            fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2); // Clear foreground too
        }
        saveStateToLocalStorage(); // Save the empty state
        activeToolId.value = null; // Deselect any active tool
        contextMenuVisible.value = false; // Hide context menu if open
        drawLabels();
        drawLabels(); // Redraw labels after clearing
        console.log("Kaart gewist en gereset.");
    }
};

</script>

<style>
/* Global styles, previously in main.css/style.css */
html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow: hidden; /* Prevent body scrollbars */
}

#app { /* Target the mount point */
  height: 100%;
  display: flex; /* Ensure #app takes full height */
}

canvas {
  display: block;
  position: absolute;
  top: 0; /* Position canvases within #area */
  left: 0;
}

#main {
	position: relative;
	display: flex;
	align-items: stretch; /* Stretch items to fill height */
	justify-content: space-around;
	height: 100%;
    width: 100%; /* Ensure main takes full width */
}

#area {
	position: relative; /* Needed for absolute positioning of canvases */
	width: 100%;
	height: 100%;
	flex: 1;
	display: flex; /* Use flexbox for centering canvas if needed, though absolute positioning handles it */
	justify-content: center; /* Center canvas container horizontally */
    align-items: center; /* Center canvas container vertically */
	overflow: hidden; /* Hide overflow from canvas potentially exceeding bounds */
    background-color: #f0f0f0; /* Optional: background for the area */
}

#tools {
	display: flex;
	flex-direction: row;
	align-items: center; /* Center items vertically */
	justify-content: center; /* Center items horizontally */
	flex-wrap: wrap;
	overflow-y: auto; /* Allow vertical scrolling for tools */
    overflow-x: hidden;
	width: 580px;
	height: 100%;
	transition: width .8s;
    flex-shrink: 0; /* Prevent tools panel from shrinking */
    background-color: #e0e0e0; /* Optional: background for tools */
    box-sizing: border-box;
    padding: 10px; /* Add some padding */
    position: relative; /* Needed for absolute positioning of clear button */
}
@media only screen and (max-width: 1700px) {
	#tools {
		width: 440px;
	}
}

@media only screen and (max-width: 1540px) {
	#tools {
		width: 300px;
	}
}
@media only screen and (max-width: 1380px) {
	#tools {
		width: 160px;
	}
}

#tools > div {
	display: block;
	/* Use the texture from the public folder */
	background-image: url('/textures/01_130x66_130x230.png');
	background-repeat: no-repeat;
	background-size: auto;
	width: 130px;
	height: 230px;
	border: 2px dashed transparent;
	box-sizing: border-box;
    margin: 5px; /* Add margin between tool items */
    cursor: pointer;
}

#tools > div.selected {
	border-color: #b05355; /* Highlight selected tool */
    border-style: solid;
}

/* Mobile/smaller screen adjustments */
@media only screen and (max-width: 966px) {
	#main {
		flex-direction: column; /* Stack tools above area */
	}

	#tools {
        flex-direction: row; /* Keep tools in a row */
        flex-wrap: nowrap; /* Prevent wrapping */
		overflow-x: auto; /* Allow horizontal scrolling for tools */
        overflow-y: hidden;
		width: 100%; /* Full width */
		height: 260px; /* Fixed height for tool area */
        align-items: center; /* Center tools vertically */
        justify-content: flex-start; /* Align tools to the start */
        padding: 10px 0; /* Adjust padding */
	}
    #tools > div {
        flex-shrink: 0; /* Prevent tool items from shrinking */
    }

	#area {
		height: calc(100% - 260px); /* Adjust height */
        width: 100%;
        justify-content: center; /* Re-center canvas */
	}
}
</style>

<style scoped> /* Add scoped styles for the button */
.clear-button {
    position: absolute;
    bottom: 15px;
    left: 15px;
    padding: 8px 15px;
    background-color: #f44336; /* Red */
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    z-index: 10; /* Ensure it's above tool items if overlapping */
}

.clear-button:hover {
    background-color: #d32f2f; /* Darker red */
}

/* Adjust button position for smaller screens if needed */
@media only screen and (max-width: 966px) {
	.clear-button {
        bottom: 10px; /* Adjust position when tools are horizontal */
        left: 10px;
    }
}
</style>