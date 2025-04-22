<template>
  <div
    class="whiteboard-container"
    ref="containerRef"
    tabindex="0"
    @keydown="handleKeyDown"
    @keyup="handleKeyUp"
    @wheel.prevent="handleWheel"
    @mousedown="handlePanStart"
    @mousemove="handleInteractionMove" 
    @mouseup="handleInteractionEnd"      
    @mouseleave="handleInteractionEnd" 
  >
    <div class="toolbar">
      <!-- Herramientas -->
      <button
        @click="setTool('select')"
        :class="{ active: currentTool === 'select' }"
      >
        Seleccionar (V)
      </button>
      <button
        @click="setTool('pencil')"
        :class="{ active: currentTool === 'pencil' }"
      >
        Lápiz (P)
      </button>
      <button
        @click="setTool('eraser')"
        :class="{ active: currentTool === 'eraser' }"
      >
        Borrador (E)
      </button>
      <button
        @click="setTool('rectangle')"
        :class="{ active: currentTool === 'rectangle' }"
      >
        Rectángulo (R)
      </button>
      <button
        @click="setTool('circle')"
        :class="{ active: currentTool === 'circle' }"
      >
        Círculo (C)
      </button>
      <button
        @click="setTool('text')"
        :class="{ active: currentTool === 'text' }"
      >
        Texto (T)
      </button>

      <!-- Controles -->
      <label for="size">Tamaño:</label>
      <input type="range" id="size" min="1" max="100" v-model="currentSize" />
      <span>{{ currentSize }}px</span>

      <label for="color">Color:</label>
      <input type="color" id="color" v-model="currentColor" />

      <!-- Acciones -->
      <button @click="undo" :disabled="!canUndo">Deshacer (Ctrl+Z)</button>
      <button @click="redo" :disabled="!canRedo">Rehacer (Ctrl+Y)</button>
      <button @click="resetZoomAndPan">Reset View</button>
      <button @click="clearCanvas">Limpiar</button>
      <span class="zoom-level">Zoom: {{ (zoomLevel * 100).toFixed(0) }}%</span>
    </div>

    <div class="canvas-wrapper" ref="canvasWrapperRef">
      <canvas
        ref="canvasRef"
        @mousedown.left="handleCanvasMouseDown"
        @mousemove="handleCanvasMouseMove" 
        @mouseup.left="handleCanvasMouseUp" 
        @mouseleave="handleCanvasMouseLeave"
        @mouseenter="handleCanvasMouseEnter"
        @dblclick="handleDoubleClick"
        @touchstart.prevent="handleCanvasMouseDown"
        @touchmove.prevent="handleCanvasMouseMove" 
        @touchend.prevent="handleCanvasMouseUp"
        :style="{ cursor: cursorStyle }"
      ></canvas>

      <!-- Cursor Visual para Lápiz -->
      <div
        v-if="showPencilPreview"
        class="pencil-preview"
        :style="pencilPreviewStyle"
      ></div>

      <!-- Input para Texto -->
      <textarea
        v-if="isEditingText"
        ref="textInputRef"
        v-model="currentTextValue"
        class="text-input"
        :style="textInputStyle"
        @blur="handleTextInputConfirm"
        @keydown.esc.prevent="handleTextInputCancel"
        @keydown.enter="handleTextInputEnter"
        placeholder="Escribe aquí..."
      ></textarea>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick, computed } from 'vue';

let currentElementId = null; // ID temporal para el elemento en proceso de dibujo

// --- Constantes ---
const PIXEL_TOLERANCE = 15;
const RESIZE_HANDLE_SIZE = 8; // Tamaño visual de las manijas
const MIN_ELEMENT_SIZE = 5; // Tamaño mínimo para formas

// --- Refs ---
const containerRef = ref(null);
const canvasWrapperRef = ref(null); // Para calcular posición relativa del canvas
const canvasRef = ref(null);
const canvasContext = ref(null);
const textInputRef = ref(null); // Para el textarea de texto

// --- Estado de la Pizarra ---
const currentTool = ref('pencil');
const isDrawing = ref(false); // Dibujando O Borrando activamente
const isDragging = ref(false); // Moviendo selección
const isSelectingArea = ref(false); // Selección por área
const currentColor = ref('#000000');
const currentSize = ref(5);
const startX = ref(0); // Coordenadas *del canvas*
const startY = ref(0);
const snapshot = ref(null); // Para previsualización de formas

// --- Gestión de Elementos ---
const elements = ref([]); // {id, type, x, y, width, height, points, text, color, size, fontSize, radius, ...}
const selectedElements = ref(new Set());

// --- Selección y Arrastre ---
const initialDragPositions = ref(new Map());
const selectionStartX = ref(0); // Coordenadas del canvas
const selectionStartY = ref(0);
const selectionEndX = ref(0);
const selectionEndY = ref(0);

// --- NUEVO: Estado de Redimensionamiento ---
const isResizing = ref(false);
const activeResizeHandle = ref(null); // 'top-left', 'bottom-right', etc. o null
const hoveredResizeHandle = ref(null); // Para el cursor
const initialResizeState = ref(null); // Guarda estado inicial del elemento al redimensionar

// --- Borrado ---
const deletedDuringCurrentErasure = ref(false);

// --- Texto ---
const isEditingText = ref(false);
const ignoreNextBlur = ref(false);
const currentTextValue = ref('');
const editingTextElementId = ref(null); // null para nuevo, ID para editar
const textInputCoords = ref({ x: 0, y: 0 }); // Coordenadas *del canvas* donde poner el input
const textInputInitialValue = ref(''); // Para detectar cambios al editar

// --- Zoom y Pan ---
const zoomLevel = ref(1);
const panOffset = ref({ x: 0, y: 0 });
const isPanning = ref(false);
const panStart = ref({ x: 0, y: 0 }); // Coordenadas *de pantalla*
const minZoom = 0.1;
const maxZoom = 10;
const isSpacePressed = ref(false); // Para pan con Space + Click

// --- Estado UI ---
const isMouseOverCanvas = ref(false); // Para preview de lápiz
const mousePos = ref({ x: 0, y: 0 }); // Posición *de pantalla* relativa al wrapper
const mouseCanvasPos = ref({ x: 0, y: 0}); // NUEVO: Posición *del canvas* (mundo)

// --- Historial Undo/Redo ---
const history = ref([]);
const historyIndex = ref(-1);

const canUndo = computed(() => historyIndex.value > 0);
const canRedo = computed(() => historyIndex.value < history.value.length - 1);

// --- Cursores ---
const cursorStyle = computed(() => {
  if (isPanning.value) return 'grabbing';
  if (isSpacePressed.value) return 'grab'; // Indicar posibilidad de pan
  // --- NUEVO: Cursor de redimensionamiento ---
  if (hoveredResizeHandle.value) {
    return getResizeCursor(hoveredResizeHandle.value);
  }
  // --- FIN NUEVO ---
  if (currentTool.value === 'pencil' && isMouseOverCanvas.value) return 'none'; // Ocultar cursor para mostrar preview
  if (isDragging.value) return 'grabbing';
  if (isResizing.value) return getResizeCursor(activeResizeHandle.value); // Cursor mientras se redimensiona
  switch (currentTool.value) {
    case 'pencil':
      return 'crosshair'; // Se verá si el preview no está activo
    case 'eraser':
      return 'cell';
    case 'select':
      // Comprobar si el cursor está sobre un elemento seleccionable (aunque no esté seleccionado)
      // Podríamos añadir lógica aquí para mostrar cursor 'move' sobre elementos,
      // pero 'default' suele ser suficiente para la herramienta de selección.
      return isSelectingArea.value ? 'crosshair' : 'default';
    case 'rectangle':
    case 'circle':
      return 'crosshair';
    case 'text':
      return 'text';
    default:
      return 'default';
  }
});

// --- Preview Lápiz ---
const showPencilPreview = computed(() => {
  return (
    currentTool.value === 'pencil' &&
    isMouseOverCanvas.value &&
    !isDrawing.value
  );
});
const pencilPreviewStyle = computed(() => {
  // Posicionar el div preview relativo al canvas wrapper
  const size = Math.max(1, currentSize.value * zoomLevel.value); // Escalar preview con zoom
  return {
    width: `${size}px`,
    height: `${size}px`,
    backgroundColor: currentColor.value,
    left: `${mousePos.value.x - size / 2}px`, // Usa mousePos (relativa al wrapper)
    top: `${mousePos.value.y - size / 2}px`,
  };
});

// --- Estilo Input Texto ---
const textInputStyle = computed(() => {
  if (!isEditingText.value) return { display: 'none' };
  // Convertir coordenadas del canvas a coordenadas de pantalla
  const screenCoords = canvasToScreenCoords(
    textInputCoords.value.x,
    textInputCoords.value.y
  );
  const canvasRect = canvasRef.value?.getBoundingClientRect();
  const wrapperRect = canvasWrapperRef.value?.getBoundingClientRect();

  if (!canvasRect || !wrapperRect) return { display: 'none' };

  // Posición relativa al wrapper
  const style = {
    position: 'absolute',
    left: `${screenCoords.x - wrapperRect.left}px`,
    top: `${screenCoords.y - wrapperRect.top}px`,
    // Ajustar tamaño de fuente según el zoom actual para que parezca del tamaño correcto
    fontSize: `${currentSize.value * zoomLevel.value}px`,
    lineHeight: `${currentSize.value * zoomLevel.value * 1.2}px`, // Ajustar line-height
    color: currentColor.value,
    // Añadir algunos estilos base
    border: '1px dashed dodgerblue',
    background: 'rgba(255, 255, 255, 0.8)',
    padding: '2px',
    margin: '0',
    minWidth: '50px', // Evitar que sea demasiado pequeño
    minHeight: `${currentSize.value * zoomLevel.value * 1.2}px`,
    outline: 'none',
    overflow: 'hidden', // Para que no muestre scrollbars inicialmente
    resize: 'none', // Evitar resize manual
    whiteSpace: 'pre-wrap', // Respetar saltos de línea y espacios
    zIndex: 101, // Encima del preview del lápiz
  };
  return style;
});

// --- Ciclo de Vida ---
onMounted(async () => {
  const canvas = canvasRef.value;
  if (!canvas) return;
  canvasContext.value = canvas.getContext('2d');
  await nextTick();
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);
  saveState(); // Estado inicial
  containerRef.value?.focus(); // Para atajos de teclado
  // Listeners globales para pan con espacio
  window.addEventListener('keydown', handleGlobalKeyDown);
  window.addEventListener('keyup', handleGlobalKeyUp);
  // MODIFICADO: Listeners de movimiento/mouseup globales para arrastrar/redimensionar/panear fuera del canvas
  window.addEventListener('mousemove', handleInteractionMove);
  window.addEventListener('mouseup', handleInteractionEnd);
});

onUnmounted(() => {
  window.removeEventListener('resize', resizeCanvas);
  window.removeEventListener('keydown', handleGlobalKeyDown);
  window.removeEventListener('keyup', handleGlobalKeyUp);
  // MODIFICADO: Remover listeners globales
  window.removeEventListener('mousemove', handleInteractionMove);
  window.removeEventListener('mouseup', handleInteractionEnd);
});

// --- Redimensionar Canvas ---
function resizeCanvas() {
  const canvas = canvasRef.value;
  const wrapper = canvasWrapperRef.value;
  if (!canvas || !wrapper) return;
  // Ajustar al tamaño del contenedor menos algo de padding
  canvas.width = wrapper.clientWidth;
  canvas.height = wrapper.clientHeight;
  redrawCanvas(); // Redibujar todo con el nuevo tamaño
}

// --- Transformaciones y Coordenadas ---

// Aplica pan y zoom al contexto
function applyTransform(ctx) {
  ctx.translate(panOffset.value.x, panOffset.value.y);
  ctx.scale(zoomLevel.value, zoomLevel.value);
}

// Convierte coordenadas de pantalla (evento) a coordenadas del canvas (mundo)
function screenToCanvasCoords(screenX, screenY) {
  const canvas = canvasRef.value;
  if (!canvas) return { x: 0, y: 0 };
  const rect = canvas.getBoundingClientRect();
  // Coordenadas relativas al canvas en pantalla
  const xOnScreen = screenX - rect.left;
  const yOnScreen = screenY - rect.top;
  // Aplicar inversa de la transformación
  const canvasX = (xOnScreen - panOffset.value.x) / zoomLevel.value;
  const canvasY = (yOnScreen - panOffset.value.y) / zoomLevel.value;
  return { x: canvasX, y: canvasY };
}

// Convierte coordenadas del canvas (mundo) a coordenadas de pantalla
function canvasToScreenCoords(canvasX, canvasY) {
  const canvas = canvasRef.value;
  if (!canvas) return { x: 0, y: 0 };
  const rect = canvas.getBoundingClientRect();
  const screenX = canvasX * zoomLevel.value + panOffset.value.x + rect.left;
  const screenY = canvasY * zoomLevel.value + panOffset.value.y + rect.top;
  return { x: screenX, y: screenY };
}

// Obtiene coordenadas del evento y las convierte a coordenadas del canvas
function getEventCoordinates(event) {
  let clientX, clientY;
  if (event.touches && event.touches.length > 0) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else if (event.changedTouches && event.changedTouches.length > 0) { // Needed for touchend
    clientX = event.changedTouches[0].clientX;
    clientY = event.changedTouches[0].clientY;
  }
  else {
    clientX = event.clientX;
    clientY = event.clientY;
  }
  return screenToCanvasCoords(clientX, clientY);
}

// Obtiene coordenadas de pantalla RELATIVAS al canvas wrapper (para UI overlay)
function getRelativeScreenCoordinates(event) {
  const wrapper = canvasWrapperRef.value;
  if (!wrapper) return { x: 0, y: 0 };
  const rect = wrapper.getBoundingClientRect();
  let clientX, clientY;
  if (event.touches && event.touches.length > 0) {
    clientX = event.touches[0].clientX;
    clientY = event.touches[0].clientY;
  } else if (event.changedTouches && event.changedTouches.length > 0) { // Needed for touchend
    clientX = event.changedTouches[0].clientX;
    clientY = event.changedTouches[0].clientY;
  }
   else {
    clientX = event.clientX;
    clientY = event.clientY;
  }
  return {
    x: clientX - rect.left,
    y: clientY - rect.top,
  };
}

// --- Zoom y Pan ---
function handleWheel(event) {
  const zoomFactor = 1.1;
  const oldZoom = zoomLevel.value;
  let newZoom = oldZoom;

  if (event.deltaY < 0) {
    // Zoom in
    newZoom *= zoomFactor;
  } else {
    // Zoom out
    newZoom /= zoomFactor;
  }
  newZoom = Math.max(minZoom, Math.min(maxZoom, newZoom)); // Clamp zoom

  if (newZoom === oldZoom) return; // No change

  const mousePoint = screenToCanvasCoords(event.clientX, event.clientY);

  const canvas = canvasRef.value;
  if (!canvas) return;
  const rect = canvas.getBoundingClientRect();
  const mouseScreenXRelative = event.clientX - rect.left;
  const mouseScreenYRelative = event.clientY - rect.top;

  panOffset.value.x = mouseScreenXRelative - mousePoint.x * newZoom;
  panOffset.value.y = mouseScreenYRelative - mousePoint.y * newZoom;

  zoomLevel.value = newZoom;
  redrawCanvas();
}

// Detectar inicio de pan (rueda ratón o Espacio + click)
function handlePanStart(event) {
  // Pan con rueda de ratón (button 1) o con Space + Left Click (button 0)
  if (event.button === 1 || (isSpacePressed.value && event.button === 0)) {
    event.preventDefault(); // Evitar scroll o comportamiento por defecto
    isPanning.value = true;
    panStart.value = { x: event.clientX, y: event.clientY };
    // No modificar cursor aquí, se hace en computed: cursorStyle
  }
  // El handleCanvasMouseDown maneja el click izquierdo normal
}

// Mover durante pan se gestiona en handleInteractionMove
// Finalizar pan se gestiona en handleInteractionEnd

function resetZoomAndPan() {
  zoomLevel.value = 1;
  panOffset.value = { x: 0, y: 0 };
  redrawCanvas();
}

// --- Funciones Auxiliares de Dibujo y Elementos ---
function setContextStyle(ctx, element) {
  ctx.strokeStyle = element.color;
  ctx.lineWidth = element.size; // Para líneas, rect, circle
  ctx.fillStyle = element.color; // Para texto, puntos
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.globalCompositeOperation = 'source-over';
  // Estilo de texto específico
  if (element.type === 'text') {
    ctx.font = `${element.fontSize}px sans-serif`; // Usar familia guardada si existe
    ctx.textAlign = 'left';
    ctx.textBaseline = 'top';
  }
}

// --- Redibujar Todo el Canvas ---
function redrawCanvas() {
  const canvas = canvasRef.value;
  const ctx = canvasContext.value;
  if (!ctx || !canvas) return;

  // Guardar estado sin transformación
  ctx.save();

  // Limpiar canvas completo (en coordenadas de pantalla)
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // Dibujar fondo blanco (opcional, puede ser transparente)
  ctx.fillStyle = '#FFFFFF';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Aplicar transformación para dibujar elementos
  applyTransform(ctx);

  // Dibujar todos los elementos con la transformación activa
  elements.value.forEach((element) => {
    setContextStyle(ctx, element);
    ctx.beginPath();
    switch (element.type) {
      case 'pencil':
        if (!element.points || element.points.length < 1) break;
        if (element.points.length < 3) {
          if (element.points.length === 1) {
            // Punto único
            ctx.arc(
              element.points[0].x,
              element.points[0].y,
              element.size / 2,
              0,
              Math.PI * 2
            );
            ctx.fill();
          } else {
            // Línea recta (2 puntos)
            ctx.moveTo(element.points[0].x, element.points[0].y);
            ctx.lineTo(element.points[1].x, element.points[1].y);
            ctx.stroke();
          }
        } else {
          // Trazo suavizado
          ctx.moveTo(element.points[0].x, element.points[0].y);
          let i;
          for (i = 1; i < element.points.length - 2; i++) {
            const xc = (element.points[i].x + element.points[i + 1].x) / 2;
            const yc = (element.points[i].y + element.points[i + 1].y) / 2;
            ctx.quadraticCurveTo(
              element.points[i].x,
              element.points[i].y,
              xc,
              yc
            );
          }
          if (i < element.points.length) {
            const p_i = element.points[i];
            const p_i1 = element.points[i + 1] ? element.points[i + 1] : p_i;
            ctx.quadraticCurveTo(p_i.x, p_i.y, p_i1.x, p_i1.y);
          }
          ctx.stroke();
        }
        break;
      case 'rectangle':
        ctx.strokeRect(element.x, element.y, element.width, element.height);
        break;
      case 'circle':
        ctx.arc(element.x, element.y, element.radius, 0, 2 * Math.PI);
        ctx.stroke();
        break;
      case 'text': // Dibujar texto
         // Medir texto para calcular altura real si es necesario (más preciso que fontSize)
         const lines = element.text.split('\n');
         const lineHeight = element.fontSize * 1.2; // Estimación consistente con resize
         lines.forEach((line, index) => {
             ctx.fillText(line, element.x, element.y + index * lineHeight);
         });
        break;
    }
  });

  // --- MODIFICADO: Dibujar indicador de selección O manijas de redimensionamiento ---
  if (selectedElements.value.size === 1 && currentTool.value === 'select') {
    // Si hay UN solo elemento seleccionado y la herramienta es Seleccionar, dibujar manijas
    const selectedElement = elements.value.find(el => el.id === [...selectedElements.value][0]);
    if (selectedElement) {
      drawResizeHandles(ctx, selectedElement);
    }
  } else if (selectedElements.value.size > 0) {
    // Si hay VARIOS elementos seleccionados, dibujar solo el cuadro azul
    selectedElements.value.forEach((id) => {
      const element = elements.value.find((el) => el.id === id);
      if (element) {
        drawSelectionIndicator(ctx, element); // Pasamos ctx ya transformado
      }
    });
  }
  // --- FIN MODIFICADO ---

  // Dibujar rectángulo de selección de área (también transformado)
  if (isSelectingArea.value) {
    drawSelectionRectangle(ctx); // Pasamos ctx ya transformado
  }

  // Restaurar estado sin transformación al final
  ctx.restore();

  // Cosas que se dibujan *sin* transformación (UI overlay) se harían aquí
  // Pero es mejor usar elementos HTML como el pencil-preview y text-input
}

// Dibuja indicador azul simple
function drawSelectionIndicator(ctx, element) {
  ctx.save(); // Guardar estado del contexto (color, etc.)
  ctx.strokeStyle = 'dodgerblue';
  ctx.lineWidth = 1 / zoomLevel.value; // Hacer la línea más fina al hacer zoom in
  ctx.setLineDash([4 / zoomLevel.value, 2 / zoomLevel.value]); // Escalar dash
  ctx.globalCompositeOperation = 'source-over';

  const bounds = getElementBounds(element); // Usar bounds para consistencia
  const padding = 5 / zoomLevel.value; // Padding visual constante

  ctx.strokeRect(
    bounds.x - padding,
    bounds.y - padding,
    bounds.width + padding * 2,
    bounds.height + padding * 2
  );
  ctx.restore(); // Restaurar color, lineWidth, etc.
}

// --- NUEVO: Dibujar manijas de redimensionamiento ---
function drawResizeHandles(ctx, element) {
  const bounds = getElementBounds(element);
  const handles = getResizeHandlePositions(bounds);
  const handleSizeCanvas = RESIZE_HANDLE_SIZE / zoomLevel.value; // Tamaño en coords del canvas

  ctx.save();
  ctx.fillStyle = 'dodgerblue';
  ctx.strokeStyle = 'white'; // Borde blanco para visibilidad
  ctx.lineWidth = 1 / zoomLevel.value;
  ctx.globalCompositeOperation = 'source-over'; // Asegurar que se dibuje encima

  // Dibujar el cuadro delimitador principal (más fino que el de selección múltiple)
  ctx.strokeStyle = 'dodgerblue';
  ctx.lineWidth = 1 / zoomLevel.value;
  ctx.setLineDash([]); // Línea sólida
  ctx.strokeRect(bounds.x, bounds.y, bounds.width, bounds.height);

  // Dibujar cada manija
  Object.values(handles).forEach(pos => {
    ctx.fillStyle = 'dodgerblue'; // Relleno de la manija
    ctx.fillRect(
        pos.x - handleSizeCanvas / 2,
        pos.y - handleSizeCanvas / 2,
        handleSizeCanvas,
        handleSizeCanvas
    );
    ctx.strokeRect( // Borde de la manija
        pos.x - handleSizeCanvas / 2,
        pos.y - handleSizeCanvas / 2,
        handleSizeCanvas,
        handleSizeCanvas
    );
  });

  ctx.restore();
}

// --- NUEVO: Obtener posiciones de las manijas ---
function getResizeHandlePositions(bounds) {
    const { x, y, width, height } = bounds;
    // Permitir manijas incluso si width/height es 0 (para elementos de línea o puntos)
    const w = Math.max(width, 0);
    const h = Math.max(height, 0);

    return {
      'top-left':     { x: x,       y: y },
      'top-center':   { x: x + w/2, y: y },
      'top-right':    { x: x + w,   y: y },
      'middle-left':  { x: x,       y: y + h/2 },
      'middle-right': { x: x + w,   y: y + h/2 },
      'bottom-left':  { x: x,       y: y + h },
      'bottom-center':{ x: x + w/2, y: y + h },
      'bottom-right': { x: x + w,   y: y + h },
    };
}

// --- NUEVO: Obtener cursor según la manija ---
function getResizeCursor(handleName) {
  if (!handleName) return 'default';
  switch (handleName) {
    case 'top-left':
    case 'bottom-right':
      return 'nwse-resize';
    case 'top-right':
    case 'bottom-left':
      return 'nesw-resize';
    case 'top-center':
    case 'bottom-center':
      return 'ns-resize';
    case 'middle-left':
    case 'middle-right':
      return 'ew-resize';
    default:
      return 'default'; // Por si acaso
  }
}


// Dibuja rectángulo de selección (usa contexto ya transformado)
function drawSelectionRectangle(ctx) {
  ctx.save();
  ctx.strokeStyle = 'rgba(0, 100, 255, 0.7)';
  ctx.lineWidth = 1 / zoomLevel.value;
  ctx.setLineDash([6 / zoomLevel.value, 3 / zoomLevel.value]);
  ctx.strokeRect(
    selectionStartX.value, // Coords ya son del canvas
    selectionStartY.value,
    selectionEndX.value - selectionStartX.value,
    selectionEndY.value - selectionStartY.value
  );
  ctx.restore();
}

// --- Bounding Box y Hit Testing (Adaptados para Texto y Lápiz mejorado) ---
function getElementBounds(element) {
  let minX = Infinity, minY = Infinity, maxX = -Infinity, maxY = -Infinity;
  switch (element.type) {
    case 'rectangle':
      return {
        x: element.x,
        y: element.y,
        width: element.width,
        height: element.height,
      };
    case 'circle':
      return {
        x: element.x - element.radius,
        y: element.y - element.radius,
        width: element.radius * 2,
        height: element.radius * 2,
      };
    case 'pencil':
       if (!element.points || element.points.length === 0) return { x: 0, y: 0, width: 0, height: 0 };
       // Optimización simple: si solo hay 1 o 2 puntos, usar esos puntos con el tamaño
       if (element.points.length <= 2) {
           minX = Math.min(...element.points.map(p => p.x));
           minY = Math.min(...element.points.map(p => p.y));
           maxX = Math.max(...element.points.map(p => p.x));
           maxY = Math.max(...element.points.map(p => p.y));
       } else {
            // Iterar para encontrar min/max
            element.points.forEach((p) => {
                minX = Math.min(minX, p.x);
                minY = Math.min(minY, p.y);
                maxX = Math.max(maxX, p.x);
                maxY = Math.max(maxY, p.y);
            });
       }
      const halfSize = (element.size || 1) / 2;
      // Asegurar width/height mínimos basados en el tamaño del trazo para elementos muy pequeños o líneas rectas
      const width = Math.max(maxX - minX, 0); // Width puede ser 0 para línea vertical
      const height = Math.max(maxY - minY, 0); // Height puede ser 0 para línea horizontal
      return {
        x: minX - halfSize,
        y: minY - halfSize,
        // Añadir el tamaño completo al ancho/alto para incluir el grosor del trazo en los bordes
        width: width + element.size,
        height: height + element.size,
      };
    case 'text':
      if (!canvasContext.value || !element.text) return { x: element.x, y: element.y, width: 0, height: 0 };
      const ctx = canvasContext.value;
      ctx.save();
      ctx.font = `${element.fontSize}px sans-serif`;
      const lines = element.text.split('\n');
      let textWidth = 0;
      lines.forEach(line => {
          textWidth = Math.max(textWidth, ctx.measureText(line).width);
      });
      ctx.restore();
      // Calcular altura basada en número de líneas y tamaño de fuente (consistente con dibujo y resize)
      const lineHeight = element.fontSize * 1.2;
      const textHeight = lines.length * lineHeight;
      // Devolver un tamaño mínimo si el texto está vacío o es muy pequeño
      const minDisplaySize = 5;
      return {
          x: element.x,
          y: element.y,
          width: Math.max(textWidth, minDisplaySize),
          height: Math.max(textHeight, minDisplaySize)
      };
    default:
      return { x: 0, y: 0, width: 0, height: 0 };
  }
}

function isPointInsideElement(px, py, element, toolType = 'select') {
  const bounds = getElementBounds(element);
  let tolerance;

  // Ajustar tolerancia (en coordenadas del canvas)
  const baseTolerance = PIXEL_TOLERANCE / zoomLevel.value; // Tolerancia visual constante

  if (toolType === 'eraser') {
    tolerance =
      (parseInt(currentSize.value, 10) / 2 / zoomLevel.value) + // Mitad del tamaño del borrador
      (2 / zoomLevel.value); // Un poco más de margen
  } else if (element.type === 'pencil') {
    // Tolerancia mayor para trazos finos, basada en el grosor del trazo + base
    tolerance = Math.max(
      baseTolerance, // Mínimo la tolerancia base
      ((element.size || 5) / 2 / zoomLevel.value) + (3 / zoomLevel.value) // Mitad grosor + extra
    );
  } else if (element.type === 'text') {
    tolerance = baseTolerance; // Tolerancia normal para texto (click dentro del bounds)
  } else { // Rectángulo, Círculo
    tolerance = baseTolerance / 2; // Tolerancia menor para bordes de formas
  }

  // Bounding Box Check (coordenadas del canvas) - Comprobación rápida inicial
  if (
    px < bounds.x - tolerance ||
    px > bounds.x + bounds.width + tolerance ||
    py < bounds.y - tolerance ||
    py > bounds.y + bounds.height + tolerance
  ) {
    return false;
  }

  // Specific Check más preciso si el BBox check pasa
  switch (element.type) {
    case 'rectangle':
        // Comprobar si está CERCA del borde (para seleccionar el borde) o DENTRO (menos relevante para stroke)
        const innerX = bounds.x + tolerance;
        const innerY = bounds.y + tolerance;
        const innerWidth = bounds.width - 2 * tolerance;
        const innerHeight = bounds.height - 2 * tolerance;
        // Está fuera del rectángulo interior? (cerca del borde)
        const isOnEdge = px < innerX || px > innerX + innerWidth || py < innerY || py > innerY + innerHeight;
        // Para 'select' o 'eraser', consideramos el hit si está dentro del bounds con tolerancia
        // Para herramientas de dibujo, solo el borde importa, pero aquí es para selección/borrado
        return true; // Si pasó el BBox check con tolerancia, es suficiente para rect/text
    case 'text': // Para texto, el BBox check con tolerancia es suficiente
      return true;
    case 'circle':
      const distSq = Math.pow(px - element.x, 2) + Math.pow(py - element.y, 2);
      // Comprobar si está cerca de la circunferencia
      const outerRadiusSq = Math.pow(element.radius + tolerance, 2);
      const innerRadiusSq = Math.pow(Math.max(0, element.radius - tolerance), 2);
      return distSq <= outerRadiusSq && distSq >= innerRadiusSq; // Cerca del borde
       // Si quisiéramos seleccionar rellenando, sería solo: distSq <= outerRadiusSq;
    case 'pencil':
      if (!element.points || element.points.length < 2) {
          // Si es un solo punto, comprobar distancia al punto
           if (element.points.length === 1) {
               const p = element.points[0];
               const distSq = Math.pow(px - p.x, 2) + Math.pow(py - p.y, 2);
               return distSq <= Math.pow(tolerance, 2);
           }
           return true; // Si pasó BBox y tiene 0 puntos? Debería ser true.
      }
      // Comprobar distancia a cada segmento de línea
      for (let i = 0; i < element.points.length - 1; i++) {
        const dist = pointToLineSegmentDistance(
          px,
          py,
          element.points[i],
          element.points[i + 1]
        );
        if (dist <= tolerance) return true; // Suficientemente cerca de un segmento
      }
      // Comprobar también los puntos finales por si el trazo es muy corto o clickean justo ahí
      if (element.points.length > 0) {
           const firstPoint = element.points[0];
           const lastPoint = element.points[element.points.length - 1];
            const distSqFirst = Math.pow(px - firstPoint.x, 2) + Math.pow(py - firstPoint.y, 2);
            const distSqLast = Math.pow(px - lastPoint.x, 2) + Math.pow(py - lastPoint.y, 2);
            if (distSqFirst <= tolerance * tolerance || distSqLast <= tolerance * tolerance) {
                return true;
            }
      }
      return false; // No está cerca de ningún segmento ni de los puntos finales
    default:
      return false;
  }
}

// Calcula la distancia mínima de un punto (px, py) a un segmento de línea (p1, p2)
function pointToLineSegmentDistance(px, py, p1, p2) {
  const l2 = Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2);
  // Si el segmento tiene longitud 0 (p1 === p2)
  if (l2 === 0) return Math.sqrt(Math.pow(px - p1.x, 2) + Math.pow(py - p1.y, 2));

  // Proyección del punto P sobre la línea que contiene el segmento
  // t es un valor entre -inf y +inf. Si t está en [0, 1], la proyección cae dentro del segmento.
  let t = ((px - p1.x) * (p2.x - p1.x) + (py - p1.y) * (p2.y - p1.y)) / l2;

  // Limitar t al rango [0, 1] para encontrar el punto más cercano *en el segmento*
  t = Math.max(0, Math.min(1, t));

  // Calcular las coordenadas del punto más cercano en el segmento
  const closestX = p1.x + t * (p2.x - p1.x);
  const closestY = p1.y + t * (p2.y - p1.y);

  // Calcular la distancia entre el punto P y el punto más cercano en el segmento
  return Math.sqrt(Math.pow(px - closestX, 2) + Math.pow(py - closestY, 2));
}

// Comprueba si el bounding box de un elemento se solapa con un rectángulo de selección
function isElementInsideRectangle(element, rect) {
  const elementBounds = getElementBounds(element);
  // Asegurarse que el rectángulo de selección tenga coordenadas x1,y1 < x2,y2
  const selX = Math.min(rect.x1, rect.x2);
  const selY = Math.min(rect.y1, rect.y2);
  const selW = Math.abs(rect.x1 - rect.x2);
  const selH = Math.abs(rect.y1 - rect.y2);

  // Comprobación de solapamiento (AABB intersection)
  return (
    elementBounds.x < selX + selW && // El borde izquierdo del elemento está a la izquierda del borde derecho de la selección
    elementBounds.x + elementBounds.width > selX && // El borde derecho del elemento está a la derecha del borde izquierdo de la selección
    elementBounds.y < selY + selH && // El borde superior del elemento está arriba del borde inferior de la selección
    elementBounds.y + elementBounds.height > selY // El borde inferior del elemento está abajo del borde superior de la selección
  );
}

// --- NUEVO: Hit testing para las manijas de redimensionamiento ---
function getHandleAtPoint(canvasX, canvasY, element) {
    if (!element) return null;
    const bounds = getElementBounds(element);
    const handles = getResizeHandlePositions(bounds);
    const handleSizeCanvas = RESIZE_HANDLE_SIZE / zoomLevel.value;
    // Aumentar un poco la tolerancia para que sea más fácil hacer clic
    const tolerance = handleSizeCanvas / 2 + 2 / zoomLevel.value;

    // Iterar sobre las manijas
    for (const [name, pos] of Object.entries(handles)) {
        // Comprobar si el punto está dentro del área de la manija (con tolerancia)
        if (
            canvasX >= pos.x - tolerance &&
            canvasX <= pos.x + tolerance &&
            canvasY >= pos.y - tolerance &&
            canvasY <= pos.y + tolerance
        ) {
            return name; // Devuelve el nombre de la manija ('top-left', etc.)
        }
    }
    return null; // No se encontró ninguna manija
}

// --- Manejadores de Eventos del Canvas ---

// Mousedown solo para acciones DENTRO del canvas (dibujar, seleccionar, iniciar drag/resize)
function handleCanvasMouseDown(event) {
  // Ignorar si estamos paneando con espacio (gestionado por handlePanStart)
  if (isSpacePressed.value) return;
  // Ignorar clicks que no sean el izquierdo o touch
  if (event.type !== 'touchstart' && event.button !== 0) return;

  containerRef.value?.focus(); // Asegurar foco para atajos
  const { x, y } = getEventCoordinates(event); // Coordenadas del canvas
  startX.value = x; // Guardar siempre punto inicial para varias operaciones
  startY.value = y;
  isDrawing.value = false;
  isDragging.value = false;
  isResizing.value = false; // NUEVO: Resetear resizing
  isSelectingArea.value = false;
  deletedDuringCurrentErasure.value = false;
  activeResizeHandle.value = null; // NUEVO
  initialResizeState.value = null; // NUEVO

  if (currentTool.value === 'select') {
    const isShiftPressed = event.shiftKey;

    // --- NUEVO: Comprobar clic en manija PRIMERO (si hay un solo elemento seleccionado) ---
    if (selectedElements.value.size === 1) {
      const selectedElement = elements.value.find(el => el.id === [...selectedElements.value][0]);
      // Asegurarse que selectedElement existe (podría haber sido borrado)
      if (selectedElement) {
          const handle = getHandleAtPoint(x, y, selectedElement);
          if (handle) {
            isResizing.value = true;
            activeResizeHandle.value = handle;
            // Clon profundo del estado inicial para poder restaurar/calcular cambios
            initialResizeState.value = JSON.parse(JSON.stringify(selectedElement));
            // Guardar bounds iniciales también es útil para la lógica de resize
            initialResizeState.value.initialBounds = getElementBounds(selectedElement);
            // Guardar punto de inicio del resize (en coords canvas)
            initialResizeState.value.startX = x;
            initialResizeState.value.startY = y;

            console.log("Starting resize on handle:", handle);
            redrawCanvas(); // Redibujar para mostrar cursor de resize inmediatamente
            return; // Importante: No continuar con la lógica de selección/arrastre normal
          }
      }
    }
    // --- FIN NUEVO ---

    // Si no se hizo clic en manija, continuar con la lógica de selección/arrastre
    let hitElement = null;
    // Iterar desde el último elemento (el que está más arriba visualmente)
    for (let i = elements.value.length - 1; i >= 0; i--) {
      if (isPointInsideElement(x, y, elements.value[i], 'select')) {
        hitElement = elements.value[i];
        break; // Encontrar el primero (el de más arriba) y parar
      }
    }

    if (hitElement) {
      const hitElementId = hitElement.id;
      const isSelected = selectedElements.value.has(hitElementId);

      if (isShiftPressed) {
        // Shift + Click: Alternar selección del elemento clickeado
        if (isSelected) {
          selectedElements.value.delete(hitElementId);
        } else {
          selectedElements.value.add(hitElementId);
        }
      } else {
        // Click normal sobre un elemento:
        if (!isSelected) {
          // Si no estaba seleccionado, limpiar selección y seleccionar solo este
          selectedElements.value.clear();
          selectedElements.value.add(hitElementId);
        }
        // Si ya estaba seleccionado (o acabamos de seleccionarlo), iniciar arrastre
        isDragging.value = true;
        initialDragPositions.value.clear();
        // Guardar las posiciones iniciales de TODOS los elementos seleccionados
        selectedElements.value.forEach((id) => {
          const el = elements.value.find((e) => e.id === id);
          if (el) {
            // Guardar una copia profunda de las propiedades relevantes para el arrastre
            if (el.type === 'pencil')
              initialDragPositions.value.set(id, { points: JSON.parse(JSON.stringify(el.points)) });
            else if (el.type === 'text') // Guardar posición y fontSize inicial por si acaso
              initialDragPositions.value.set(id, { x: el.x, y: el.y, fontSize: el.fontSize });
            else // Rect, Circle
              initialDragPositions.value.set(id, { x: el.x, y: el.y });
          }
        });
      }
    } else {
      // Click en el lienzo (sin elemento debajo):
      if (!isShiftPressed) {
        // Si no se presiona Shift, limpiar la selección actual
        selectedElements.value.clear();
      }
      // Iniciar selección de área
      isSelectingArea.value = true;
      selectionStartX.value = x;
      selectionStartY.value = y;
      selectionEndX.value = x;
      selectionEndY.value = y;
    }
    redrawCanvas(); // Actualizar visualmente la selección/inicio de área
  } else if (currentTool.value === 'eraser') {
    isDrawing.value = true; // 'isDrawing' se usa también para indicar borrado activo
    handleEraserAction(x, y); // Intentar borrar en el punto del click inicial
  } else if (currentTool.value === 'text') {
    // Si hacemos clic con Texto y no estamos editando, iniciamos edición/creación
    if (!isEditingText.value) {
      // Comprobar si clickeamos sobre texto existente para editarlo
      let hitTextElement = null;
      for (let i = elements.value.length - 1; i >= 0; i--) {
        const el = elements.value[i];
        if (el.type === 'text' && isPointInsideElement(x, y, el, 'text')) {
          hitTextElement = el;
          break;
        }
      }
      if (hitTextElement) {
        // Click sobre texto existente -> editarlo
        startTextEditing(hitTextElement);
      } else {
        // Click en espacio vacío -> crear nuevo texto
        startTextEditing(null, x, y);
      }
    } else {
      // Si ya estábamos editando texto y hacemos clic fuera del textarea,
      // el evento 'blur' del textarea debería encargarse de confirmar.
      // No necesitamos hacer nada aquí explícitamente para evitar doble confirmación.
      // Si el click es DENTRO del mismo textarea, tampoco hacemos nada.
    }
  } else {
    // Herramientas de dibujo (pencil, rect, circle)
    isDrawing.value = true;
    currentElementId = generateId(); // ID temporal para el nuevo elemento
    if (currentTool.value === 'pencil') {
      elements.value.push({
        id: currentElementId,
        type: 'pencil',
        color: currentColor.value,
        size: parseInt(currentSize.value, 10),
        points: [{ x, y }], // Empezar con el primer punto
      });
      redrawCanvas(); // Mostrar el primer punto
    } else if (
      currentTool.value === 'rectangle' ||
      currentTool.value === 'circle'
    ) {
      // Guardar estado del canvas para previsualización rápida
      snapshot.value = canvasContext.value?.getImageData(
        0,
        0,
        canvasRef.value.width,
        canvasRef.value.height
      );
    }
  }
}

// --- MODIFICADO: Handler unificado para movimiento (Pan, Drag, Resize, Draw, Hover) ---
function handleInteractionMove(event) {
    // Obtener coordenadas de pantalla y de canvas
    // Usar clientX/Y directamente ya que este listener es global (window)
    const clientX = event.touches ? event.touches[0].clientX : event.clientX;
    const clientY = event.touches ? event.touches[0].clientY : event.clientY;
    const screenPosRelative = getRelativeScreenCoordinates(event); // Relativa al wrapper (para UI HTML)
    const canvasPos = screenToCanvasCoords(clientX, clientY); // Coords del canvas (mundo)

    // Actualizar siempre la posición del ratón para previews y cursores
    mousePos.value = screenPosRelative;
    mouseCanvasPos.value = canvasPos;

    // 1. Pan (si está activo)
    if (isPanning.value) {
        const dx = clientX - panStart.value.x;
        const dy = clientY - panStart.value.y;
        panOffset.value.x += dx;
        panOffset.value.y += dy;
        panStart.value = { x: clientX, y: clientY }; // Actualizar inicio para el siguiente delta
        redrawCanvas();
        return; // Pan tiene la máxima prioridad
    }

    // 2. Redimensionamiento (si está activo)
    if (isResizing.value && initialResizeState.value && activeResizeHandle.value) {
        const element = elements.value.find(el => el.id === initialResizeState.value.id);
        if (element) {
            // Llamar a la función que calcula y aplica el redimensionamiento
            // Usamos las coordenadas actuales del canvas (canvasPos)
            resizeElement(element, canvasPos.x, canvasPos.y, activeResizeHandle.value, initialResizeState.value);
            redrawCanvas(); // Redibujar para mostrar el cambio
        }
        return; // Redimensionamiento tiene prioridad sobre los siguientes
    }

    // 3. Arrastre de selección (si está activo)
    if (isDragging.value) {
        const dx = canvasPos.x - startX.value; // Delta en coordenadas de canvas desde el inicio del drag
        const dy = canvasPos.y - startY.value;
        selectedElements.value.forEach((id) => {
            const element = elements.value.find((el) => el.id === id);
            const initialPos = initialDragPositions.value.get(id);
            if (element && initialPos) {
                if (element.type === 'pencil') {
                    // Mover cada punto del trazo
                    element.points = initialPos.points.map((p) => ({
                        x: p.x + dx,
                        y: p.y + dy,
                    }));
                } else { // Rect, Circle, Text
                    // Mover el punto de anclaje (x, y)
                    element.x = initialPos.x + dx;
                    element.y = initialPos.y + dy;
                }
            }
        });
        redrawCanvas(); // Redibujar para mostrar el movimiento
        return;
    }

    // 4. Selección de área (si está activa)
    if (isSelectingArea.value) {
        selectionEndX.value = canvasPos.x; // Actualizar esquina final del rectángulo
        selectionEndY.value = canvasPos.y;
        redrawCanvas(); // Redibujar para mostrar el rectángulo de selección
        return;
    }

    // 5. Dibujo / Borrado (si está activo)
    if (isDrawing.value) {
        if (currentTool.value === 'eraser') {
            handleEraserAction(canvasPos.x, canvasPos.y); // Borrar en la posición actual
        } else if (currentTool.value === 'pencil') {
            const element = elements.value.find((el) => el.id === currentElementId);
            if (element) {
                element.points.push({ x: canvasPos.x, y: canvasPos.y }); // Añadir punto al trazo
                redrawCanvas(); // Optimización: solo dibujar el último segmento? Por ahora redibujar todo
            }
        } else if (
            currentTool.value === 'rectangle' ||
            currentTool.value === 'circle'
        ) {
            // Previsualización de formas usando snapshot para eficiencia
            const ctx = canvasContext.value;
            if (snapshot.value) ctx.putImageData(snapshot.value, 0, 0); // Restaurar canvas base
            ctx.save();
            applyTransform(ctx); // Aplicar zoom/pan a la previsualización también
            setContextStyle(ctx, {
                type: currentTool.value,
                color: currentColor.value,
                size: parseInt(currentSize.value, 10),
            });
            ctx.beginPath();
            if (currentTool.value === 'rectangle') {
                ctx.strokeRect(
                    startX.value, // Desde el punto inicial guardado
                    startY.value,
                    canvasPos.x - startX.value, // Ancho hasta pos actual
                    canvasPos.y - startY.value  // Alto hasta pos actual
                );
            } else { // Círculo
                const radius = Math.sqrt(
                    Math.pow(canvasPos.x - startX.value, 2) + Math.pow(canvasPos.y - startY.value, 2)
                );
                ctx.arc(startX.value, startY.value, radius, 0, 2 * Math.PI);
                ctx.stroke();
            }
            ctx.restore();
        }
        return;
    }

    // 6. Hover sobre manijas (si no se está haciendo otra cosa y hay un elemento seleccionado)
    // Se ejecuta solo si ninguna de las acciones anteriores (pan, resize, drag, selectArea, draw) está activa
    if (currentTool.value === 'select' && selectedElements.value.size === 1) {
         const selectedElement = elements.value.find(el => el.id === [...selectedElements.value][0]);
         if (selectedElement) {
             hoveredResizeHandle.value = getHandleAtPoint(canvasPos.x, canvasPos.y, selectedElement);
             // El cursor se actualizará automáticamente por la computed property `cursorStyle`
         } else {
             hoveredResizeHandle.value = null; // Asegurar que se limpia si el elemento ya no existe
         }
    } else {
        hoveredResizeHandle.value = null; // Resetear si no aplica (otra herramienta, sin selección única, etc.)
    }
}

// MouseMove solo para actualizar la preview del lápiz y potencialmente otros hovers específicos del canvas
function handleCanvasMouseMove(event) {
  // Actualizar posición relativa del ratón para preview lápiz (usa coords relativas al wrapper)
   mousePos.value = getRelativeScreenCoordinates(event);
   // Nota: handleInteractionMove ya actualiza mouseCanvasPos (coords del mundo)
   // Podríamos añadir aquí lógica de hover sobre elementos NO seleccionados si quisiéramos
   // cambiar el cursor a 'move' al pasar sobre ellos, pero complica el estado.
}


// --- MODIFICADO: Handler unificado para mouseup/touchend global ---
function handleInteractionEnd(event) {
    // Determinar si fue un evento de pan finalizado (rueda o espacio)
    // Necesitamos ser cuidadosos con touchend que no tiene 'button'
    let endedPan = false;
    if (isPanning.value) {
        if (event.type === 'mouseup' && (event.button === 1 || isSpacePressed.value)) {
             endedPan = true;
        } else if (event.type === 'touchend' && isSpacePressed.value) {
            // Asumimos que touchend con espacio presionado termina el pan
            // (Podría ser más complejo si hubiera multi-touch pan)
            endedPan = true;
        }
        if (endedPan) {
             isPanning.value = false;
        }
    }

    // No procesar el resto si fue solo el final de un paneo con espacio (mouseup/touchend)
    // Evita que un click+drag con espacio+botón izquierdo inicie una selección al soltar.
    if (endedPan && isSpacePressed.value) {
        // Resetear estado de espacio si la tecla ya se soltó (keyup lo hace también)
        // isSpacePressed.value = false; // Mejor que lo haga solo el keyup
        redrawCanvas(); // Para asegurar que el cursor vuelve a la normalidad
        return;
    }

    // Ignorar clicks que no sean el izquierdo en mouseup
    if (event.type === 'mouseup' && event.button !== 0) return;

    // --- Finalizar acciones ---

    if (isResizing.value) {
        // Finalizar Redimensionamiento
        const resizedElement = elements.value.find(el => el.id === initialResizeState.value?.id);
        // Comprobar si realmente hubo un cambio significativo antes de guardar? Opcional.
        const boundsBefore = initialResizeState.value?.initialBounds;
        const boundsAfter = resizedElement ? getElementBounds(resizedElement) : null;
        let changed = true; // Asumir cambio por defecto
        if (boundsBefore && boundsAfter) {
            // Simple check si bounds son iguales (podría ser más robusto)
            changed = boundsBefore.x !== boundsAfter.x || boundsBefore.y !== boundsAfter.y ||
                      boundsBefore.width !== boundsAfter.width || boundsBefore.height !== boundsAfter.height;
            // Para texto, comparar también fontSize
            if (resizedElement.type === 'text' && initialResizeState.value?.fontSize !== resizedElement.fontSize) {
                changed = true;
            }
        }

        isResizing.value = false;
        activeResizeHandle.value = null;
        initialResizeState.value = null;
        redrawCanvas(); // Asegurar estado visual final
        if (changed) { // Solo guardar si hubo cambio
           saveState(); // Guardar estado después de redimensionar
        }

    } else if (isDragging.value) {
        // Finalizar Arrastre
        isDragging.value = false;
        // Comprobar si realmente hubo movimiento
        const currentPos = getEventCoordinates(event);
        const dx = currentPos.x - startX.value;
        const dy = currentPos.y - startY.value;
        const moved = Math.abs(dx) > 1 || Math.abs(dy) > 1; // Umbral mínimo de movimiento

        initialDragPositions.value.clear(); // Limpiar siempre
        if (moved) { // Solo guardar si hubo movimiento
            saveState();
        }
        redrawCanvas(); // Redibujar por si acaso

    } else if (isSelectingArea.value) {
        // Finalizar Selección de Área
        isSelectingArea.value = false;
        const selectionRect = {
            x1: selectionStartX.value, y1: selectionStartY.value,
            x2: selectionEndX.value, y2: selectionEndY.value,
        };
        // Guardar selección previa para comparar si cambió
        const prevSelectedIds = new Set(selectedElements.value);
        let selectionChanged = false;

        elements.value.forEach((element) => {
            if (isElementInsideRectangle(element, selectionRect)) {
                if (!selectedElements.value.has(element.id)) {
                    selectedElements.value.add(element.id);
                    selectionChanged = true;
                }
            }
        });
        // Si la selección cambió (se añadieron elementos), redibujar
        if (selectionChanged) {
            redrawCanvas();
        } else {
            // Si no se añadieron elementos nuevos, pero el área era válida,
            // puede que queramos redibujar para quitar el rectángulo azul.
            // Comprobar si el rectángulo tenía área > 0
             const selW = Math.abs(selectionRect.x1 - selectionRect.x2);
             const selH = Math.abs(selectionRect.y1 - selectionRect.y2);
             if (selW > 0 || selH > 0) {
                 redrawCanvas(); // Quitar el rectángulo de selección
             }
        }
        // No guardar estado por seleccionar

    } else if (isDrawing.value) {
        // Finalizar Dibujo o Borrado
        const { x, y } = getEventCoordinates(event); // Coords del canvas donde se soltó
        let addedOrModifiedElement = false;

        if (currentTool.value === 'eraser') {
            // Si se borró algo durante el arrastre del borrador, guardar estado
            if (deletedDuringCurrentErasure.value) {
                 saveState();
            }
        } else {
            // Finalizar Dibujo (pencil, rect, circle)
            const element = elements.value.find((el) => el.id === currentElementId);

            if (currentTool.value === 'pencil') {
                 if (element) {
                    // Si es solo un punto (o casi), añadir un segundo punto muy cercano para formar una línea mínima visible
                    if (element.points.length === 1) {
                        element.points.push({ x: element.points[0].x + 0.01, y: element.points[0].y });
                        addedOrModifiedElement = true;
                    } else if (element.points.length > 1) {
                        // Si tiene más de un punto, considerar válido
                        addedOrModifiedElement = true;
                        // Opcional: Simplificar trazo aquí si es necesario (eliminar puntos redundantes)
                    }
                 }
                 // Si no se dibujó nada (o se canceló antes), el elemento puede no existir o tener pocos puntos
                 if (!addedOrModifiedElement) {
                     elements.value = elements.value.filter(el => el.id !== currentElementId); // Eliminar inválido
                 }

            } else if (currentTool.value === 'rectangle') {
                const width = x - startX.value;
                const height = y - startY.value;
                // Usar MIN_ELEMENT_SIZE
                if (Math.abs(width) >= MIN_ELEMENT_SIZE || Math.abs(height) >= MIN_ELEMENT_SIZE) {
                    // Quitar el posible elemento temporal si se añadió antes (aunque no debería con snapshot)
                    // elements.value = elements.value.filter((el) => el.id !== currentElementId);
                    elements.value.push({
                        id: currentElementId, type: 'rectangle',
                        x: width > 0 ? startX.value : x, // Ajustar x,y si se dibuja hacia arriba/izquierda
                        y: height > 0 ? startY.value : y,
                        width: Math.abs(width), height: Math.abs(height),
                        color: currentColor.value, size: parseInt(currentSize.value, 10),
                    });
                    addedOrModifiedElement = true;
                } else {
                     // No añadir si es demasiado pequeño
                    // elements.value = elements.value.filter((el) => el.id !== currentElementId); // Asegurar limpieza
                }
            } else if (currentTool.value === 'circle') {
                const radius = Math.sqrt(Math.pow(x - startX.value, 2) + Math.pow(y - startY.value, 2));
                 // Usar MIN_ELEMENT_SIZE / 2 para el radio mínimo
                if (radius >= MIN_ELEMENT_SIZE / 2) {
                    // elements.value = elements.value.filter((el) => el.id !== currentElementId);
                    elements.value.push({
                        id: currentElementId, type: 'circle',
                        x: startX.value, y: startY.value, // Centro es el punto inicial
                        radius: radius,
                        color: currentColor.value, size: parseInt(currentSize.value, 10),
                    });
                    addedOrModifiedElement = true;
                } else {
                     // No añadir si es demasiado pequeño
                    // elements.value = elements.value.filter((el) => el.id !== currentElementId);
                }
            }
            snapshot.value = null; // Limpiar snapshot
            redrawCanvas(); // Redibujar siempre al final del dibujo
            if (addedOrModifiedElement) {
                saveState(); // Guardar estado si se añadió/modificó un elemento válido
            }
        }
    }

    // Resetear estados generales al final (excepto pan que se reseteó antes si aplicaba)
    isDrawing.value = false;
    isDragging.value = false;
    isResizing.value = false; // Asegurar que se resetea
    isSelectingArea.value = false;
    currentElementId = null;
    deletedDuringCurrentErasure.value = false;
    activeResizeHandle.value = null; // Asegurar que se resetea
    initialResizeState.value = null; // Asegurar que se resetea
    initialDragPositions.value.clear(); // Limpiar posiciones de drag
    // hoveredResizeHandle se resetea en move, pero asegurarlo aquí también
    if (hoveredResizeHandle.value) {
        hoveredResizeHandle.value = null;
        redrawCanvas(); // Redibujar si el cursor necesita cambiar
    }
}

// MouseUp específico del canvas (puede no ser necesario si usamos el global)
function handleCanvasMouseUp(event) {
    // Actualmente, toda la lógica de finalización está en handleInteractionEnd (global).
    // Podríamos llamar a handleInteractionEnd aquí, pero ya se dispara globalmente.
    // handleInteractionEnd(event);
    // Dejar vacío por ahora, o para lógica muy específica que SOLO deba ocurrir
    // si el mouseup es estrictamente dentro del canvas.
}

function handleCanvasMouseEnter() {
  isMouseOverCanvas.value = true;
}
function handleCanvasMouseLeave(event) {
    isMouseOverCanvas.value = false;
    hoveredResizeHandle.value = null; // Quitar hover de manija si salimos
    // La finalización de acciones (drag, resize, draw) ahora la gestiona handleInteractionEnd en el window
    // así que no necesitamos forzar el final aquí.
    // El preview del lápiz se ocultará automáticamente por `isMouseOverCanvas`
    redrawCanvas(); // Redibujar para asegurar que el cursor/hover se actualiza
}

// --- NUEVO: Lógica Central de Redimensionamiento (CORREGIDA PARA CÍRCULO) ---
function resizeElement(element, mouseX, mouseY, handle, initialState) {
    const { initialBounds, startX: resizeStartX, startY: resizeStartY } = initialState;
    // Usamos las coordenadas del canvas actuales (mouseX, mouseY) y las iniciales del resize
    const dx = mouseX - resizeStartX;
    const dy = mouseY - resizeStartY;

    // Copia de los bounds iniciales para calcular
    let newX = initialBounds.x;
    let newY = initialBounds.y;
    let newWidth = initialBounds.width;
    let newHeight = initialBounds.height;

    // Calcular nuevos bounds teóricos basados en la manija arrastrada y el delta
    if (handle.includes('left')) {
        newWidth = initialBounds.width - dx;
        newX = initialBounds.x + dx;
    } else if (handle.includes('right')) {
        newWidth = initialBounds.width + dx;
    }

    if (handle.includes('top')) {
        newHeight = initialBounds.height - dy;
        newY = initialBounds.y + dy;
    } else if (handle.includes('bottom')) {
        newHeight = initialBounds.height + dy;
    }

    // Prevenir tamaño negativo o muy pequeño y ajustar posición si es necesario
    // ¡Ojo! Para el círculo, aplicaremos el MIN_ELEMENT_SIZE al radio más adelante.
    // Aquí solo prevenimos que el *bounding box* se invierta completamente.
    if (newWidth < 0) {
         if (handle.includes('left')) {
            newX = newX + newWidth; // newWidth es negativo
        }
        newWidth = Math.abs(newWidth);
    }
     if (newHeight < 0) {
         if (handle.includes('top')) {
            newY = newY + newHeight; // newHeight es negativo
        }
        newHeight = Math.abs(newHeight);
    }


    // Aplicar los nuevos valores al elemento real, según su tipo
    switch (element.type) {
        case 'rectangle':
             // Prevenir tamaño mínimo para rectángulos
             if (newWidth < MIN_ELEMENT_SIZE) {
                 if (handle.includes('left')) newX = newX + newWidth - MIN_ELEMENT_SIZE;
                 newWidth = MIN_ELEMENT_SIZE;
             }
             if (newHeight < MIN_ELEMENT_SIZE) {
                  if (handle.includes('top')) newY = newY + newHeight - MIN_ELEMENT_SIZE;
                 newHeight = MIN_ELEMENT_SIZE;
             }
            element.x = newX;
            element.y = newY;
            element.width = newWidth;
            element.height = newHeight;
            break;

        case 'circle':
            // 1. Calcular el centro del *nuevo bounding box teórico*
            const theoreticalCenterX = newX + newWidth / 2;
            const theoreticalCenterY = newY + newHeight / 2;

            // 2. Calcular el nuevo radio como la distancia desde el nuevo centro hasta el ratón
            const distX = mouseX - theoreticalCenterX;
            const distY = mouseY - theoreticalCenterY;
            let newRadius = Math.sqrt(distX * distX + distY * distY);

            // 3. Aplicar tamaño mínimo al radio
            newRadius = Math.max(MIN_ELEMENT_SIZE / 2, newRadius);

            // 4. Asignar el nuevo centro y radio al elemento
            element.x = theoreticalCenterX;
            element.y = theoreticalCenterY;
            element.radius = newRadius;
            break;

        case 'pencil':
            // Prevenir tamaño mínimo para trazos (aplicado a las dimensiones del bounds)
             if (newWidth < MIN_ELEMENT_SIZE) {
                 if (handle.includes('left')) newX = newX + newWidth - MIN_ELEMENT_SIZE;
                 newWidth = MIN_ELEMENT_SIZE;
             }
             if (newHeight < MIN_ELEMENT_SIZE) {
                  if (handle.includes('top')) newY = newY + newHeight - MIN_ELEMENT_SIZE;
                 newHeight = MIN_ELEMENT_SIZE;
             }

            // Escalar los puntos relativos al centro del bounding box inicial
            const safeInitialWidth = initialBounds.width === 0 ? 1 : initialBounds.width;
            const safeInitialHeight = initialBounds.height === 0 ? 1 : initialBounds.height;
            const scaleX = newWidth / safeInitialWidth;
            const scaleY = newHeight / safeInitialHeight;

            const initialCenterX = initialBounds.x + initialBounds.width / 2;
            const initialCenterY = initialBounds.y + initialBounds.height / 2;

            // Usar el nuevo centro calculado (newX, newY + dimensiones)
            const currentNewCenterX = newX + newWidth / 2;
            const currentNewCenterY = newY + newHeight / 2;

            element.points = initialState.points.map(p => {
                const vecX = p.x - initialCenterX;
                const vecY = p.y - initialCenterY;
                const scaledVecX = vecX * scaleX;
                const scaledVecY = vecY * scaleY;
                return {
                    x: currentNewCenterX + scaledVecX,
                    y: currentNewCenterY + scaledVecY,
                };
            });
            break;
        case 'text':
             // Prevenir tamaño mínimo para texto (aplicado al bounds antes de calcular fontSize)
             if (newWidth < MIN_ELEMENT_SIZE) {
                 if (handle.includes('left')) newX = newX + newWidth - MIN_ELEMENT_SIZE;
                 newWidth = MIN_ELEMENT_SIZE;
             }
             if (newHeight < MIN_ELEMENT_SIZE) {
                  if (handle.includes('top')) newY = newY + newHeight - MIN_ELEMENT_SIZE;
                 newHeight = MIN_ELEMENT_SIZE;
             }

            // Redimensionar texto: ajustar posición y fontSize
            element.x = newX;
            element.y = newY;

            const initialFontSize = initialState.fontSize;
            const initialLineHeight = initialFontSize * 1.2;
            const initialLines = initialState.text.split('\n').length;
            // Usar Math.max con MIN_ELEMENT_SIZE para la altura inicial calculada
            const initialCalculatedHeight = Math.max(MIN_ELEMENT_SIZE, initialLines * initialLineHeight);

            let heightRatio = 1;
            if (initialCalculatedHeight > 0.1) {
                 heightRatio = newHeight / initialCalculatedHeight;
            }

            // Ajustar fontSize, asegurando un mínimo (e.g., 5px)
            element.fontSize = Math.max(5, Math.round(initialFontSize * heightRatio));

            currentSize.value = element.fontSize;
            break;
    }
}


// --- Lógica de Texto ---

function startTextEditing(element = null, defaultX = 0, defaultY = 0) {
  selectedElements.value.clear(); // Deseleccionar todo al editar/crear texto
  if (element) {
    // Editando existente
    editingTextElementId.value = element.id;
    currentTextValue.value = element.text;
    textInputInitialValue.value = element.text; // Guardar valor original para comparar cambios
    textInputCoords.value = { x: element.x, y: element.y };
    // Usar el tamaño y color del elemento existente para el input y controles
    currentSize.value = element.fontSize;
    currentColor.value = element.color;
  } else {
    // Creando nuevo
    editingTextElementId.value = null;
    currentTextValue.value = '';
    textInputInitialValue.value = '';
    textInputCoords.value = { x: defaultX, y: defaultY };
    // Usar color y tamaño actuales de la barra de herramientas (ya están en v-model)
  }

  isEditingText.value = true;
  nextTick(() => {
    textInputRef.value?.focus();
    autosizeTextarea(); // Ajustar tamaño inicial del textarea

    // *** Ignorar el blur inmediato que ocurre al hacer focus() ***
    ignoreNextBlur.value = true;
    setTimeout(() => {
        ignoreNextBlur.value = false;
    }, 0); // Resetear después del ciclo de eventos actual
  });
  redrawCanvas(); // Quitar selección/manijas si las había
}

// Confirmar texto (onBlur o Enter sin Shift)
function handleTextInputConfirm() {
    // Comprobar la bandera para ignorar el blur inmediato post-focus
    if (ignoreNextBlur.value) {
        ignoreNextBlur.value = false; // Resetearla para el siguiente blur real
        // Opcional: re-focus si se pierde inesperadamente, aunque suele mantenerse
        // textInputRef.value?.focus();
        return; // Salir sin confirmar
    }

    if (!isEditingText.value) return; // Salir si no estábamos editando

    const text = currentTextValue.value.trim(); // Quitar espacios extra

    let changed = false; // Flag para saber si guardar en historial

    if (editingTextElementId.value) {
        // Editando elemento existente
        const element = elements.value.find(
        (el) => el.id === editingTextElementId.value
        );
        if (element) {
            if (text) {
                // Si hay texto, actualizar si cambió el texto, tamaño o color
                if (text !== textInputInitialValue.value ||
                    element.fontSize !== parseInt(currentSize.value, 10) ||
                    element.color !== currentColor.value)
                {
                    element.text = text;
                    element.fontSize = parseInt(currentSize.value, 10);
                    element.color = currentColor.value;
                    changed = true;
                }
            } else {
                // Si el texto se borró completamente, eliminamos el elemento
                elements.value = elements.value.filter(
                    (el) => el.id !== editingTextElementId.value
                );
                changed = true; // Se eliminó un elemento
            }
        }
    } else {
        // Creando nuevo elemento
        if (text) {
            // Solo añadir si hay texto
            elements.value.push({
                id: generateId(),
                type: 'text',
                x: textInputCoords.value.x,
                y: textInputCoords.value.y,
                text: text,
                color: currentColor.value,
                fontSize: parseInt(currentSize.value, 10),
            });
            changed = true; // Se añadió un elemento
        }
    }

    // Resetear estado de edición
    isEditingText.value = false;
    currentTextValue.value = '';
    editingTextElementId.value = null;
    textInputInitialValue.value = '';

    redrawCanvas(); // Redibujar para mostrar el texto final o quitar el input

    if (changed) {
        saveState(); // Guardar en el historial si hubo cambios
    }
}

// Cancelar edición (Escape) - Manejado por el listener del keydown en el textarea
function handleTextInputCancel() {
  isEditingText.value = false;
  currentTextValue.value = '';
  editingTextElementId.value = null;
  textInputInitialValue.value = '';
  redrawCanvas(); // Redibujar por si había algo seleccionado antes o para quitar input
}

// Manejar Enter en textarea (Enter confirma, Shift+Enter nueva línea)
function handleTextInputEnter(event) {
  if (!event.shiftKey) {
    event.preventDefault(); // Prevenir nueva línea por defecto con Enter
    handleTextInputConfirm(); // Confirmar el texto
  }
  // Si es Shift+Enter, el navegador inserta la nueva línea por defecto.
  // Solo necesitamos ajustar el tamaño del textarea después.
  nextTick(autosizeTextarea); // Ajustar tamaño después de posible nueva línea
}

// Ajustar altura (y opcionalmente ancho) del textarea al contenido
function autosizeTextarea() {
  const textarea = textInputRef.value;
  if (textarea) {
    // Resetear altura para calcular correctamente scrollHeight
    textarea.style.height = 'auto';
    // Aplicar nueva altura basada en el contenido
    textarea.style.height = `${textarea.scrollHeight}px`;

    // Opcional: ajustar ancho también (puede ser menos predecible)
    // textarea.style.width = 'auto';
    // const scrollWidth = textarea.scrollWidth;
    // // Añadir un poco de padding extra al ancho calculado si se desea
    // textarea.style.width = `${scrollWidth + 2}px`;
    // // Poner un ancho máximo para evitar que se expanda indefinidamente
    // textarea.style.maxWidth = `${canvasRef.value ? canvasRef.value.width * 0.8 : 500}px`;
  }
}
// Observar cambios en el texto mientras se edita para autoajustar tamaño
watch(currentTextValue, () => {
  if (isEditingText.value) {
    nextTick(autosizeTextarea);
  }
});

// Editar texto existente con doble clic (si herramienta es Select o Text)
function handleDoubleClick(event) {
  if (currentTool.value !== 'select' && currentTool.value !== 'text') return;

  const { x, y } = getEventCoordinates(event);
  let hitTextElement = null;
  // Buscar texto clickeado (el de más arriba)
  for (let i = elements.value.length - 1; i >= 0; i--) {
    const el = elements.value[i];
    if (el.type === 'text' && isPointInsideElement(x, y, el, 'text')) {
      hitTextElement = el;
      break;
    }
  }

  if (hitTextElement) {
    event.preventDefault(); // Prevenir selección de texto del navegador
    // Si ya estábamos editando otro texto, primero confirmamos el anterior
    if (isEditingText.value && editingTextElementId.value !== hitTextElement.id) {
        // Usar una pequeña demora para asegurar que el blur/confirm del anterior ocurra primero
        // si el doble clic causó un blur en el input anterior.
         setTimeout(() => {
            startTextEditing(hitTextElement);
         }, 10); // Pequeña demora
    } else if (!isEditingText.value) {
       // Si no estábamos editando, iniciamos la edición directamente
        startTextEditing(hitTextElement);
    }
  }
}

// --- Borrado ---
function handleEraserAction(x, y) {
  let elementDeleted = false;
  // Iterar al revés para borrar el elemento superior primero
  for (let i = elements.value.length - 1; i >= 0; i--) {
    const element = elements.value[i];
    // Usar isPointInsideElement con el tipo 'eraser' para la tolerancia correcta
    if (isPointInsideElement(x, y, element, 'eraser')) {
      const elementIdToDelete = element.id;
      elements.value.splice(i, 1); // Eliminar del array
      selectedElements.value.delete(elementIdToDelete); // Deseleccionar si estaba seleccionado
      deletedDuringCurrentErasure.value = true; // Marcar que se borró algo en esta acción
      elementDeleted = true;
      break; // Borrar solo un elemento por pasada/movimiento
    }
  }
  if (elementDeleted) {
      redrawCanvas(); // Redibujar para mostrar el cambio
  }
  return elementDeleted; // Devuelve true si algo fue borrado
}

// --- Acciones Toolbar y Teclado ---
function setTool(tool) {
  // Si cambiamos a una herramienta diferente mientras editamos texto, confirmar cambios
  if (tool !== 'text' && isEditingText.value) {
    handleTextInputConfirm();
  }
  // Si cambiamos a una herramienta de dibujo/texto/borrador, deseleccionar elementos
  if (tool !== 'select') {
    if (selectedElements.value.size > 0) {
      selectedElements.value.clear();
      redrawCanvas(); // Quitar indicadores de selección/manijas
    }
  }
   // Cancelar resize si estaba activo al cambiar de herramienta
   if (isResizing.value && tool !== 'select') {
        // Restaurar estado original del elemento que se estaba redimensionando
        const originalElement = elements.value.find(el => el.id === initialResizeState.value?.id);
        if (originalElement && initialResizeState.value) {
            Object.assign(originalElement, JSON.parse(JSON.stringify(initialResizeState.value))); // Restaurar props desde la copia profunda
        }
        isResizing.value = false;
        activeResizeHandle.value = null;
        initialResizeState.value = null;
        redrawCanvas();
   }

  currentTool.value = tool;
}

function clearCanvas() {
  // Cancelar edición de texto si está activa
  if (isEditingText.value) {
    handleTextInputCancel();
  }
  // Cancelar resize si está activo
   if (isResizing.value) {
        isResizing.value = false;
        activeResizeHandle.value = null;
        initialResizeState.value = null;
   }
  elements.value = []; // Vaciar array de elementos
  selectedElements.value.clear(); // Limpiar selección
  resetZoomAndPan(); // Resetear vista también es útil al limpiar
  saveState(); // Guardar el estado vacío en el historial
  // redrawCanvas() es llamado por resetZoomAndPan
}

// --- Guardar/Restaurar Estado (Undo/Redo) ---
function saveState() {
  // Cancelar edición de texto antes de guardar (confirmar podría ser mejor si es posible)
  if (isEditingText.value) {
    handleTextInputConfirm(); // Intentar confirmar texto actual antes de guardar
    // Si la confirmación falla o se cancela, isEditingText será false
  }
   // Cancelar resize antes de guardar (ya debería estar hecho en mouseup, pero por seguridad)
   if (isResizing.value) {
        isResizing.value = false;
        activeResizeHandle.value = null;
        initialResizeState.value = null;
        // No restaurar, el estado actual es el que queremos guardar
   }


  // Si hemos hecho undo, eliminar el historial futuro
  if (historyIndex.value < history.value.length - 1) {
    history.value = history.value.slice(0, historyIndex.value + 1);
  }

  // Crear una copia profunda del estado actual de los elementos
  const currentState = { elements: JSON.parse(JSON.stringify(elements.value)) };

  // Evitar guardar estados idénticos consecutivos
  if (
    history.value.length > 0 &&
    JSON.stringify(currentState.elements) ===
      JSON.stringify(history.value[historyIndex.value]?.elements)
  ) {
    return; // No guardar si no hubo cambios reales en los elementos
  }

  history.value.push(currentState);
  historyIndex.value++;

  // Limitar tamaño del historial (opcional)
  const MAX_HISTORY = 50;
  if (history.value.length > MAX_HISTORY) {
    history.value.shift(); // Eliminar el estado más antiguo
    historyIndex.value--; // Ajustar índice
  }
}

function restoreState(index) {
  if (index >= 0 && index < history.value.length) {
    // Cancelar edición o resize antes de restaurar
    if (isEditingText.value) handleTextInputCancel();
    if (isResizing.value) {
        isResizing.value = false;
        activeResizeHandle.value = null;
        initialResizeState.value = null;
    }

    // Restaurar estado desde una copia profunda del historial
    const stateToRestore = JSON.parse(JSON.stringify(history.value[index]));
    elements.value = stateToRestore.elements;
    historyIndex.value = index; // Actualizar índice del historial
    selectedElements.value.clear(); // Limpiar selección al restaurar
    redrawCanvas(); // Redibujar todo con el estado restaurado
  }
}

function undo() {
  if (canUndo.value) {
    restoreState(historyIndex.value - 1);
  }
}

function redo() {
  if (canRedo.value) {
    restoreState(historyIndex.value + 1);
  }
}

// --- Atajos de Teclado ---

// Listeners globales para teclas modificadoras (Espacio para pan)
function handleGlobalKeyDown(event) {
  if (event.key === ' ' && !isSpacePressed.value && !isEditingText.value) { // No activar pan si se está editando texto
    // Prevenir scroll de página si el foco está en el container principal
    if (document.activeElement === containerRef.value || document.activeElement === document.body) {
      event.preventDefault();
    }
    isSpacePressed.value = true;
    // El cursor cambiará a 'grab' vía computed property
    // redrawCanvas(); // No es estrictamente necesario, el cursor se actualiza en move
  }
}
function handleGlobalKeyUp(event) {
  if (event.key === ' ') {
    if (isSpacePressed.value) {
        isSpacePressed.value = false;
        // Si estábamos paneando con espacio y soltamos la tecla espacio,
        // el paneo debería terminar (handleInteractionEnd también lo maneja al soltar click).
        if (isPanning.value) {
           // isPanning.value = false; // Dejar que handleInteractionEnd lo maneje
        }
        // El cursor volverá a la normalidad en el próximo move o redraw
        redrawCanvas(); // Forzar actualización del cursor
    }
  }
}

// Atajos específicos del componente (cuando tiene foco)
function handleKeyDown(event) {
  // No interferir si se está editando texto en el textarea, excepto para Esc/Enter
  const targetTagName = event.target.tagName;
  if (isEditingText.value && targetTagName === 'TEXTAREA') {
    // Permitir Esc y Enter (manejados por el input/textarea)
    if (event.key === 'Escape' || event.key === 'Enter') {
      // Dejar que los handlers del textarea actúen
    } else {
      return; // Ignorar otros atajos mientras se edita en el textarea
    }
  }
   // No interferir si el foco está en otro input (color, range)
   if (targetTagName === 'INPUT') {
       return;
   }


  // No interferir si se está paneando con espacio
  if (isSpacePressed.value) return;

  const isCtrlOrMeta = event.ctrlKey || event.metaKey; // Cmd en Mac

  if (isCtrlOrMeta) {
    switch (event.key.toLowerCase()) {
      case 'z':
        event.preventDefault();
        undo();
        break;
      case 'y':
        event.preventDefault();
        redo();
        break;
       case 'shift': // Manejar Ctrl+Shift+Z para redo
           if (event.shiftKey && event.code === 'KeyZ') { // Usar event.code para ser más específico
                event.preventDefault();
                redo();
           }
           break;
      case 'a': // Seleccionar todo
        event.preventDefault();
        selectedElements.value = new Set(elements.value.map((el) => el.id));
        currentTool.value = 'select'; // Cambiar a herramienta de selección
        redrawCanvas();
        break;
      case '0': // Reset Zoom/Pan
      case '=': // Zoom In (teclado US)
      case '+': // Zoom In (numérico y otros teclados)
        event.preventDefault();
        if (event.key === '0') {
             resetZoomAndPan();
        } else {
            // Simular zoom in centrado en el canvas visible
            const centerX = (canvasRef.value?.width ?? 0) / 2;
            const centerY = (canvasRef.value?.height ?? 0) / 2;
            const centerScreenX = (canvasRef.value?.getBoundingClientRect().left ?? 0) + centerX;
            const centerScreenY = (canvasRef.value?.getBoundingClientRect().top ?? 0) + centerY;
            handleWheel({deltaY: -100, clientX: centerScreenX, clientY: centerScreenY, preventDefault: () => {}}); // delta negativo = zoom in
        }
        break;
      case '-': // Zoom Out
        event.preventDefault();
         // Simular zoom out centrado
         const centerX = (canvasRef.value?.width ?? 0) / 2;
         const centerY = (canvasRef.value?.height ?? 0) / 2;
         const centerScreenX = (canvasRef.value?.getBoundingClientRect().left ?? 0) + centerX;
         const centerScreenY = (canvasRef.value?.getBoundingClientRect().top ?? 0) + centerY;
        handleWheel({deltaY: 100, clientX: centerScreenX, clientY: centerScreenY, preventDefault: () => {}}); // delta positivo = zoom out
        break;
    }
  } else {
    // Atajos sin modificador (Ctrl/Meta)
    switch (event.key.toLowerCase()) {
      case 'v':
        event.preventDefault(); setTool('select'); break;
      case 'p':
        event.preventDefault(); setTool('pencil'); break;
      case 'e':
        event.preventDefault(); setTool('eraser'); break;
      case 'r':
        event.preventDefault(); setTool('rectangle'); break;
      case 'c':
        event.preventDefault(); setTool('circle'); break;
      case 't':
        event.preventDefault(); setTool('text'); break; // Atajo Texto
      case 'delete':
      case 'backspace':
        // Borrar elementos seleccionados si la herramienta es 'select'
        if (currentTool.value === 'select' && selectedElements.value.size > 0) {
          event.preventDefault();
          const originalSize = elements.value.length;
          elements.value = elements.value.filter(
            (el) => !selectedElements.value.has(el.id) // Filtrar los seleccionados
          );
          const changed = elements.value.length !== originalSize;
          selectedElements.value.clear(); // Limpiar selección después de borrar
          redrawCanvas();
          if (changed) saveState(); // Guardar si se borró algo
        }
        break;
      case 'escape':
          let handled = false;
          // 1. Cancelar edición de texto (manejado por el textarea)
          // 2. Cancelar redimensionamiento
          if (isResizing.value) {
               event.preventDefault();
               // Restaurar estado original del elemento
               const originalElement = elements.value.find(el => el.id === initialResizeState.value?.id);
               if (originalElement && initialResizeState.value) {
                   Object.assign(originalElement, JSON.parse(JSON.stringify(initialResizeState.value)));
                   // Si el tamaño cambió el fontSize del texto, restaurar currentSize también
                   if (originalElement.type === 'text') {
                       currentSize.value = originalElement.fontSize;
                   }
               }
               isResizing.value = false;
               activeResizeHandle.value = null;
               initialResizeState.value = null;
               redrawCanvas();
               handled = true;
          }
          // 3. Cancelar selección de área
          else if (isSelectingArea.value) {
              event.preventDefault();
              isSelectingArea.value = false;
              redrawCanvas(); // Quitar el rectángulo de selección
              handled = true;
          }
          // 4. Deseleccionar elementos
          else if (selectedElements.value.size > 0) {
              event.preventDefault();
              selectedElements.value.clear();
              redrawCanvas(); // Quitar selección
              handled = true;
          }
          // 5. Cancelar dibujo de forma (rect/circle) si está en progreso? (Más complejo)
          // else if (isDrawing.value && (currentTool.value === 'rectangle' || currentTool.value === 'circle')) {
          //      event.preventDefault();
          //      isDrawing.value = false;
          //      currentElementId = null;
          //      snapshot.value = null; // Limpiar snapshot
          //      redrawCanvas(); // Restaurar el estado visual anterior
          //      handled = true;
          // }
        break;
    }
  }
}

// Generador simple de IDs únicos (mejorar si se necesita sincronización real)
function generateId() {
  return Date.now().toString(36) + Math.random().toString(36).substring(2, 9);
}

// --- Observadores (Watchers) ---

// Actualizar elementos seleccionados al cambiar color
watch(currentColor, (newColor) => {
  // Aplicar solo si la herramienta es 'select' y hay elementos seleccionados
  if (currentTool.value === 'select' && selectedElements.value.size > 0) {
    let changed = false;
    elements.value.forEach((el) => {
      if (selectedElements.value.has(el.id)) {
        if (el.color !== newColor) {
            el.color = newColor;
            changed = true;
        }
      }
    });
    if (changed) {
      redrawCanvas();
      saveState(); // Guardar cambio de color en historial
    }
  }
});

// Actualizar elementos seleccionados al cambiar tamaño (stroke o font)
watch(currentSize, (newSizeStr) => {
  // Aplicar solo si la herramienta es 'select', hay elementos seleccionados y NO se está redimensionando activamente
  if (currentTool.value === 'select' && selectedElements.value.size > 0 && !isResizing.value) {
    const newSize = parseInt(newSizeStr, 10);
    if (isNaN(newSize) || newSize < 1) return; // Validar tamaño

    let changed = false;
    elements.value.forEach((el) => {
      if (selectedElements.value.has(el.id)) {
        if (el.type === 'text') {
            if (el.fontSize !== newSize) {
                el.fontSize = newSize;
                changed = true;
            }
        } else { // pencil, rect, circle
             if (el.size !== newSize) {
                el.size = newSize;
                changed = true;
             }
        }
      }
    });
    if (changed) {
      redrawCanvas();
      saveState(); // Guardar cambio de tamaño en historial
    }
  }
});
</script>

<style scoped>
.whiteboard-container {
  outline: none;
  border: 1px solid #ccc;
  display: flex;
  flex-direction: column;
  /* align-items: center; */ /* No centrar para que el wrapper ocupe ancho */
  width: 100%;
  height: 90vh; /* Ocupar más altura */
  /* max-width: 900px; */ /* Quitar max-width para que use todo el espacio */
  margin: 10px auto;
  background-color: #f0f0f0; /* Un gris más claro */
  overflow: hidden; /* Evitar scrollbars del contenedor principal */
  position: relative; /* Para posicionar elementos internos */
}

.toolbar {
  padding: 8px 15px;
  background-color: #e8e8e8;
  border-bottom: 1px solid #ccc;
  display: flex;
  flex-wrap: wrap;
  gap: 5px 10px;
  align-items: center;
  width: 100%;
  box-sizing: border-box;
  flex-shrink: 0; /* No encoger la toolbar */
  z-index: 10; /* Encima del canvas wrapper */
  position: relative;
}

/* Estilos de botones y controles (sin cambios grandes) */
.toolbar button {
  padding: 4px 8px;
  cursor: pointer;
  border: 1px solid #aaa;
  background-color: #fff;
  border-radius: 4px;
  font-size: 0.9em;
}
.toolbar button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
.toolbar button.active {
  background-color: #1e90ff;
  color: white;
  border-color: #007bff;
}
.toolbar label {
  margin-left: 8px;
  font-size: 0.9em;
}
.toolbar input[type='range'] {
  width: 80px;
  vertical-align: middle;
}
.toolbar input[type='color'] {
  width: 30px;
  height: 20px;
  border: 1px solid #ccc;
  padding: 0;
  vertical-align: middle;
  cursor: pointer;
}
.toolbar span {
  font-size: 0.8em;
  min-width: 30px;
  display: inline-block;
  text-align: right;
  vertical-align: middle;
}
.toolbar .zoom-level {
  margin-left: auto;
  font-size: 0.85em;
  color: #555;
}

.canvas-wrapper {
  flex-grow: 1; /* Ocupar espacio restante */
  width: 100%;
  position: relative; /* Necesario para posicionar canvas, preview, input */
  overflow: hidden; /* CLAVE: El wrapper oculta lo que se sale, canvas es "infinito" */
  background-color: #ddd; /* Fondo del área fuera del canvas visible */
  /* cursor: grab; */ /* Cursor por defecto para indicar paneo */
}

canvas {
  display: block;
  background-color: #fff; /* Fondo del "papel" */
  /* El tamaño se gestiona en JS, no aquí */
  /* El cursor se gestiona dinámicamente */
  /* Quitar márgenes */
  margin: 0;
  /* Podríamos necesitar position:absolute si el wrapper tiene otros hijos,
   pero como es el único principal, block funciona bien. */
}

.pencil-preview {
  background-color: black; /* Color por defecto */
  border-radius: 50%;
  position: absolute;
  pointer-events: none;
  z-index: 100;
  border: 1px solid rgba(255, 255, 255, 0.5); /* Borde blanco semitransparente */
}

.text-input {
  position: absolute;
  border: 1px dashed dodgerblue;
  background: rgba(255, 255, 255, 0.9); /* Fondo ligeramente traslúcido */
  padding: 2px 4px;
  margin: 0;
  outline: none;
  overflow: hidden;
  resize: none;
  white-space: pre-wrap; /* Importante para multilínea */
  z-index: 102;
  font-family: sans-serif; /* Asegurar fuente consistente */
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  line-height: 1.2; /* Ajuste básico */
}

/* Añadir estilo para indicar paneo activo */
.whiteboard-container:has(canvas:active[style*='grabbing']) {
  /* O alguna clase que se añada durante el pan */
}
</style>
