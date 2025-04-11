<template>
  <div id="main"> 
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
    <button @click="clearMap" class="clear-button">Kaart Wissen</button>
 </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed, nextTick } from 'vue';
import ContextMenu from './components/ContextMenu.vue';
import './style.css';

const bgCanvas = ref(null);
const fgCanvas = ref(null);
const toolsContainer = ref(null);

const map = reactive([]);
const tool = reactive([0, 0]); 
const activeToolId = ref(null); 
const isPlacing = ref(false);

const contextMenuVisible = ref(false);
const contextMenuX = ref(0);
const contextMenuY = ref(0);
const contextMenuTile = ref({ x: -1, y: -1 }); 
const contextMenuItems = ref([
  { label: 'Label toevoegen/bewerken', action: 'edit-label' },
  { label: 'Verwijderen', action: 'delete' }
]);

const ntiles = 7;
const tileWidth = 128;
const tileHeight = 64;
const texColumns = 12; 
const texRows = 6;    
const textureSrc = '/textures/01_130x66_130x230.png'; 
const canvasWidth = 910;
const canvasHeight = 666; 
const drawingAreaWidth = 910; 
const drawingAreaHeight = 462; 

let bgCtx = null;
let fgCtx = null;
let texture = null;

const toolItems = computed(() => {
  const items = [];
  let count = 0;
  for (let i = 0; i < texRows; i++) {
    for (let j = 0; j < texColumns; j++) {
      items.push({
        id: `tool_${count++}`,
        coords: [i, j],
        bgPos: `-${j * 130 + 2}px -${i * 230}px` 
      });
    }
  }
  return items;
});

onMounted(() => {
  texture = new Image();
  texture.onload = init;
  texture.src = textureSrc;

  window.addEventListener('click', handleClickOutside);
  fgCanvas.value.addEventListener('click', handleLabelClick);
});

const init = () => {
  for (let i = 0; i < ntiles; i++) {
    map[i] = [];
    for (let j = 0; j < ntiles; j++) {
      map[i][j] = [0, 0, '']; 
    }
  }

  const bg = bgCanvas.value;
  bg.width = canvasWidth;
  bg.height = canvasHeight;
  bgCtx = bg.getContext("2d");
  bgCtx.translate(drawingAreaWidth / 2, tileHeight * 2);

  const fg = fgCanvas.value;
  fg.width = canvasWidth;
  fg.height = canvasHeight;
  fgCtx = fg.getContext('2d');
  fgCtx.translate(drawingAreaWidth / 2, tileHeight * 2);

  loadStateFromLocalStorage(); 
  drawMap();
};

const drawMap = () => {
  if (!bgCtx || !texture || !texture.complete) return;
  bgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2); 
  for (let i = 0; i < ntiles; i++) {
    for (let j = 0; j < ntiles; j++) {
      if (map[i] && map[i][j]) { 
        drawImageTile(bgCtx, i, j, map[i][j][0], map[i][j][1]);
      }
    }
  }
  if (fgCtx) {
    fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2);
    drawLabels();
  }
};

const handleLabelClick = (e) => {
  if (!fgCtx) return;
  const pos = getPosition(e);
  const tileX = pos.x;
  const tileY = pos.y;

  
  const label = map[tileX][tileY][2];
  if (label !== null && label !== '') {
    const labelX = (tileY - tileX) * tileWidth / 2;
    const labelY = (tileX + tileY) * tileHeight / 2 + tileHeight + 15; 
    const rect = fgCanvas.value.getBoundingClientRect();
    const clickX = e.clientX - rect.left - drawingAreaWidth / 2;
    const clickY = e.clientY - rect.top - tileHeight * 2;

    const labelWidth = label.length * 12; 
    const labelHeight = 24; 
    if (clickX >= labelX - labelWidth/2 && clickX <= labelX + labelWidth/2 && clickY >= labelY - labelHeight/2 && clickY <= labelY + labelHeight/2) {
      const input = document.createElement('input');
      input.type = 'text';
      input.value = label;
      input.style.position = 'absolute';
      input.style.left = `${e.clientX}px`;
      input.style.top = `${e.clientY}px`;
      input.style.width = '100px';
      input.style.height = '20px';
      input.style.fontSize = '16px';
      input.style.padding = '2px';

      document.body.appendChild(input);
      input.focus();

      input.addEventListener('blur', () => {
        map[tileX][tileY][2] = input.value;
        drawMap(); 
        saveStateToLocalStorage(); 
        drawLabels(); 
        document.body.removeChild(input);
      });

      input.addEventListener('keydown', (e) => {
        if (e.key === 'Enter') {
          input.blur();
        }
      });
    }
  }
};

const drawLabels = () => {
  if (!fgCtx) return;
  fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2);
  fgCtx.font = '24px Arial';
  fgCtx.fillStyle = 'black';
  fgCtx.textAlign = 'center';
  for (let i = 0; i < ntiles; i++) {
    for (let j = 0; j < ntiles; j++) {
      const label = map[i][j][2];
      if (label) {
        const x = (j - i) * tileWidth / 2;
        const y = (i + j) * tileHeight / 2 + tileHeight + 15; 
        fgCtx.fillText(label, x, y);
      }
    }
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

  const label = (map[x] && map[x][y] && map[x][y][2]) ? map[x][y][2] : '';

  ctx.save();
  ctx.translate((y - x) * tileWidth / 2, (x + y) * tileHeight / 2);

  const sourceX = texCol * 130;
  const sourceY = texRow * 230;
  const sourceWidth = 130;
  const sourceHeight = 230;
  const drawX = -sourceWidth / 2;
  const drawY = -130; 
  ctx.drawImage(texture, sourceX, sourceY, sourceWidth, sourceHeight, drawX, drawY, sourceWidth, sourceHeight);

  if (label) {
    ctx.font = '12px Arial'; 
    ctx.fillStyle = 'black'; 
    ctx.textAlign = 'center'; 
    ctx.fillText(label, 0, tileHeight + 5); 
  }

  ctx.restore();
};

const getPosition = (e) => {
  const offsetY = e.offsetY;
  const offsetX = e.offsetX;

  const _y = (offsetY - tileHeight * 2) / tileHeight;
  const _x = offsetX / tileWidth - ntiles / 2; 

  const tileX = Math.floor(_y - _x);
  const tileY = Math.floor(_x + _y);

  return { x: tileX, y: tileY };
};

const viz = (e) => {
  if (!fgCtx) return;
  const pos = getPosition(e);
  const tileX = pos.x;
  const tileY = pos.y;

  fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2);

  drawLabels();

  if (map[tileX] && map[tileX][tileY]) {
    const label = map[tileX][tileY][2];
    if (label !== null && label !== '') {
      const labelX = (tileY - tileX) * tileWidth / 2;
      const labelY = (tileX + tileY) * tileHeight / 2 + tileHeight + 15; 
      const rect = fgCanvas.value.getBoundingClientRect();
      const mouseX = e.clientX - rect.left - drawingAreaWidth / 2;
      const mouseY = e.clientY - rect.top - tileHeight * 2;

      const labelWidth = label.length * 12; 
      const labelHeight = 24; 
      if (mouseX >= labelX - labelWidth/2 && mouseX <= labelX + labelWidth/2 && mouseY >= labelY - labelHeight/2 && mouseY <= labelY + labelHeight/2) {
        console.log('Hovering over label');
        fgCtx.fillStyle = 'rgba(255, 0, 0, 0.5)'; 
        fgCtx.fillRect(labelX - labelWidth/2, labelY - labelHeight/2, labelWidth, labelHeight);
      }
    }
  }

  if (activeToolId.value !== null && tileX >= 0 && tileX < ntiles && tileY >= 0 && tileY < ntiles) {
    drawTile(fgCtx, tileX, tileY, 'rgba(0,0,0,0.2)');
  }

  if (isPlacing.value) {
    placeTileAtEvent(e);
  }
};

const placeTileAtEvent = (e) => {
   const isPrimaryAction = e.type === 'mousemove' || e.which === 1 || e.button === 0 || e.type.startsWith('touch') || e.type.startsWith('pointer');
   if (!fgCtx || !isPrimaryAction) return;

   const pos = getPosition(e);
   if (pos.x >= 0 && pos.x < ntiles && pos.y >= 0 && pos.y < ntiles) {
     const currentTile = map[pos.x][pos.y];
     if (currentTile[0] !== tool[0] || currentTile[1] !== tool[1] || currentTile[2] === undefined || currentTile[2] === '') {
        map[pos.x][pos.y] = [tool[0], tool[1], '']; 
        drawMap();
        saveStateToLocalStorage();
        drawMap();
        drawLabels(); 
     }
   }
};

const startPlacing = (e) => {
  if (contextMenuVisible.value) {
    contextMenuVisible.value = false;
  }
  if (activeToolId.value === null) return;

  const isPrimaryAction = e.which === 1 || e.button === 0 || e.type.startsWith('touch') || e.type.startsWith('pointer');
  if (!isPrimaryAction) return; 

  isPlacing.value = true;
  placeTileAtEvent(e); 
};

const stopPlacing = () => {
  if (isPlacing.value) {
    isPlacing.value = false;
  }
};

const handleContextMenu = (e) => {
  stopPlacing(); 
  const pos = getPosition(e);
  if (pos.x >= 0 && pos.x < ntiles && pos.y >= 0 && pos.y < ntiles) {
    if (map[pos.x][pos.y][0] !== 0 || map[pos.x][pos.y][1] !== 0) {
        contextMenuTile.value = { x: pos.x, y: pos.y };
        contextMenuX.value = e.offsetX; 
        contextMenuY.value = e.offsetY; 
        contextMenuVisible.value = false;
        nextTick(() => {
          contextMenuVisible.value = true;
        });
    } else {
        contextMenuVisible.value = false;
    }
  } else {
    contextMenuVisible.value = false; 
  }
};

const handleContextMenuItemClick = (action) => {
  const { x, y } = contextMenuTile.value;
  if (x === -1 || y === -1 || !map[x] || !map[x][y]) {
      contextMenuVisible.value = false;
      return;
  }

  if (action === 'delete') {
    map[x][y] = [0, 0, '']; 
    drawMap();
    saveStateToLocalStorage();
    drawLabels(); 
  } else if (action === 'edit-label') {
    const currentLabel = map[x][y][2] || '';
    const newLabel = prompt('Voer label in:', currentLabel);
    if (newLabel !== null) { 
      map[x][y][2] = newLabel;
      drawMap(); 
      saveStateToLocalStorage(); 
      drawLabels(); 
    }
  }
  contextMenuVisible.value = false; 
};

const handleClickOutside = (e) => {
  if (contextMenuVisible.value) {
      if (e.target !== fgCanvas.value) {
          contextMenuVisible.value = false;
      }
  }
};

const selectTool = (coords, toolId) => {
  tool[0] = coords[0];
  tool[1] = coords[1];
  activeToolId.value = toolId;
};

const encodeMapData = (mapData) => {
  try {
    const jsonString = JSON.stringify(mapData);
    return btoa(jsonString); 
  } catch (e) {
    console.error("Failed to encode map data:", e);
    return '';
  }
};

const FromBase64_Legacy = (str) => {
  try {
    return atob(str).split('').map(c => c.charCodeAt(0));
  } catch (e) {
    console.error("Failed to decode legacy base64 state:", e);
    return null; 
  }
};

const decodeMapData = (base64String) => {
  try {
    const jsonString = atob(base64String); 
    const parsedData = JSON.parse(jsonString); 
    if (Array.isArray(parsedData) && parsedData.every(row => Array.isArray(row))) {
        return parsedData.map(row =>
            row.map(tile =>
                Array.isArray(tile) && tile.length >= 2
                    ? [tile[0] || 0, tile[1] || 0, tile[2] || '']
                    : [0, 0, ''] 
            )
        );
    }
    console.warn("Decoded map data has invalid structure.");
    return null; 
  } catch (e) {
    return null; 
  }
};

const saveStateToLocalStorage = () => {
  const state = encodeMapData(map); 
  if (state) {
    try {
      localStorage.setItem('isocityMapData', state);
    } catch (e) {
      console.error("Failed to save map state to localStorage:", e);
    }
  }
};

const loadStateFromLocalStorage = () => {
  const state = localStorage.getItem('isocityMapData');
  let loadedSuccessfully = false;

  if (state) {
    const decodedMapNew = decodeMapData(state);
    if (decodedMapNew) {
      if (decodedMapNew.length === ntiles && decodedMapNew.every(row => row.length === ntiles)) {
          map.length = 0; 
          decodedMapNew.forEach(row => map.push(row)); 
          loadedSuccessfully = true;
          drawMap(); 
      } else {
          console.error("Loaded map dimensions (new format) do not match configuration.");
      }
    } else {
      const u8 = FromBase64_Legacy(state);
      if (u8 && u8.length === ntiles * ntiles) {
          map.length = 0; 
          let c = 0;
          for (let i = 0; i < ntiles; i++) {
              const newRow = [];
              for (let j = 0; j < ntiles; j++) {
                  const t = u8[c++];
                  const texRow = Math.trunc(t / texColumns);
                  const texCol = Math.trunc(t % texColumns);
                  newRow.push([texRow, texCol, null]); 
              }
              map.push(newRow);
          }
          loadedSuccessfully = true;
          console.log("Successfully loaded map using legacy format.");
          drawMap(); 
      } else {
          console.error("Failed to decode map state using legacy method or dimensions mismatch.");
      }
    }
  }

  if (!loadedSuccessfully) {
      console.log("Initializing default map.");
      if (map.length !== ntiles || map[0].length !== ntiles || map[0][0].length !== 3) {
          init(); 
      }
  }
};

const clearMap = () => {
    if (confirm('Weet u zeker dat u de hele kaart wilt wissen? Dit kan niet ongedaan gemaakt worden.')) {
        map.length = 0; 
        for (let i = 0; i < ntiles; i++) {
            const newRow = [];
            for (let j = 0; j < ntiles; j++) {
                newRow.push([0, 0, '']); 
            }
            map.push(newRow); 
        }
        if (bgCtx && fgCtx) {
            drawMap(); 
            fgCtx.clearRect(-drawingAreaWidth, -drawingAreaHeight, drawingAreaWidth * 2, drawingAreaHeight * 2); 
        }
        saveStateToLocalStorage(); 
        activeToolId.value = null; 
        contextMenuVisible.value = false; 
        drawLabels(); 
        console.log("Kaart gewist en gereset.");
    }
};
</script>
