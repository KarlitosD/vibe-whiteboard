<template>
  <div
    class="whiteboard-container"
    ref="containerRef"
    tabindex="0"
    @keydown="handleKeyDown"
    @keyup="handleKeyUp"
    @wheel.prevent="handleWheel"
    @mousedown="handlePanStart"
    @mousemove="handlePanMove"
    @mouseup="handlePanEnd"
    @mouseleave="handlePanEnd"
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
      <!-- Nuevo -->

      <!-- Controles -->
      <label for="size">Tamaño:</label>
      <input type="range" id="size" min="1" max="100" v-model="currentSize" />
      <!-- Max aumentado para texto -->
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
      >
        @touchstart.prevent="handleCanvasMouseDown"
        @touchmove.prevent="handleCanvasMouseMove"
        @touchend.prevent="handleCanvasMouseUp" :style="{ cursor: cursorStyle }"
        ></canvas
      >

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
const elements = ref([]); // {id, type, x, y, width, height, points, text, color, size, fontSize, ...}
const selectedElements = ref(new Set());

// --- Selección y Arrastre ---
const initialDragPositions = ref(new Map());
const selectionStartX = ref(0); // Coordenadas del canvas
const selectionStartY = ref(0);
const selectionEndX = ref(0);
const selectionEndY = ref(0);

// --- Borrado ---
const deletedDuringCurrentErasure = ref(false);

// --- Texto ---
const isEditingText = ref(false);
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

const PIXEL_TOLERANCE = 15;

// --- Estado UI ---
const isMouseOverCanvas = ref(false); // Para preview de lápiz
const mousePos = ref({ x: 0, y: 0 }); // Posición *de pantalla* relativa al wrapper

// --- Historial Undo/Redo ---
const history = ref([]);
const historyIndex = ref(-1);

const canUndo = computed(() => historyIndex.value > 0);
const canRedo = computed(() => historyIndex.value < history.value.length - 1);

// --- Cursores ---
const cursorStyle = computed(() => {
  if (isPanning.value) return 'grabbing';
  if (isSpacePressed.value) return 'grab'; // Indicar posibilidad de pan
  if (currentTool.value === 'pencil' && isMouseOverCanvas.value) return 'none'; // Ocultar cursor para mostrar preview
  if (isDragging.value) return 'grabbing';
  switch (currentTool.value) {
    case 'pencil':
      return 'crosshair'; // Se verá si el preview no está activo
    case 'eraser':
      return 'cell';
    case 'select':
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
    left: `${mousePos.value.x - size / 2}px`,
    top: `${mousePos.value.y - size / 2}px`,
    borderRadius: '50%',
    position: 'absolute',
    pointerEvents: 'none', // No interferir con eventos del canvas
    zIndex: 100, // Encima del canvas pero debajo del input de texto
    opacity: 0.7,
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
});

onUnmounted(() => {
  window.removeEventListener('resize', resizeCanvas);
  window.removeEventListener('keydown', handleGlobalKeyDown);
  window.removeEventListener('keyup', handleGlobalKeyUp);
  // Asegurar que no queden listeners de pan globales si el componente se destruye
  window.removeEventListener('mousemove', handlePanMove);
  window.removeEventListener('mouseup', handlePanEnd);
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

// Restaura la transformación por defecto (para dibujar UI como el preview)
// ¡OJO! Esto es complejo si se quiere dibujar UI sobre el canvas transformado.
// Es más fácil usar elementos HTML separados como el pencil-preview y text-input.
// function resetTransform(ctx) {
//   ctx.setTransform(1, 0, 0, 1, 0, 0);
// }

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
  } else {
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
  } else {
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

  // Calcular el punto del canvas bajo el cursor *antes* del zoom
  const mousePoint = screenToCanvasCoords(event.clientX, event.clientY);

  // Calcular el nuevo offset para que el punto bajo el cursor permanezca allí
  // La nueva posición en pantalla del punto del canvas sería:
  // newScreenX = mousePoint.x * newZoom + newPanOffset.x + rect.left
  // Queremos que newScreenX sea igual a event.clientX
  // event.clientX = mousePoint.x * newZoom + newPanOffset.x + rect.left
  // newPanOffset.x = event.clientX - rect.left - mousePoint.x * newZoom
  // Hacemos lo mismo para Y
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

// Mover durante pan
function handlePanMove(event) {
  // Actualizar posición del ratón relativa para el preview del lápiz
  mousePos.value = getRelativeScreenCoordinates(event);

  if (!isPanning.value) return;

  const dx = event.clientX - panStart.value.x;
  const dy = event.clientY - panStart.value.y;

  // Aplicar delta al offset inicial (el offset se guarda en panOffset)
  panOffset.value.x += dx;
  panOffset.value.y += dy;

  // Actualizar el punto de inicio para el siguiente movimiento
  panStart.value = { x: event.clientX, y: event.clientY };

  redrawCanvas();
}

// Finalizar pan
function handlePanEnd(event) {
  // Solo terminar si estábamos paneando con el botón correcto o espacio
  if (isPanning.value && (event.button === 1 || isSpacePressed.value)) {
    isPanning.value = false;
    // No modificar cursor aquí, se hace en computed: cursorStyle
  }
  // El handleCanvasMouseUp maneja el mouse up izquierdo normal
}

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
  // Este fillRect debe cubrir el canvas visible, no necesita transformarse
  ctx.fillStyle = '#FFFFFF';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Aplicar transformación para dibujar elementos
  applyTransform(ctx);

  // Dibujar todos los elementos con la transformación activa
  elements.value.forEach((element) => {
    // ctx.save(); // Guardar/Restaurar por elemento es más seguro si hay estilos complejos
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
        ctx.fillText(element.text, element.x, element.y);
        break;
    }
    // ctx.restore(); // Si se usó ctx.save() por elemento
  });

  // Dibujar indicadores de selección (también transformados)
  selectedElements.value.forEach((id) => {
    const element = elements.value.find((el) => el.id === id);
    if (element) {
      drawSelectionIndicator(ctx, element); // Pasamos ctx ya transformado
    }
  });

  // Dibujar rectángulo de selección de área (también transformado)
  if (isSelectingArea.value) {
    drawSelectionRectangle(ctx); // Pasamos ctx ya transformado
  }

  // Restaurar estado sin transformación al final
  ctx.restore();

  // Cosas que se dibujan *sin* transformación (UI overlay) se harían aquí
  // Pero es mejor usar elementos HTML como el pencil-preview y text-input
}

// Dibuja indicador sobre un elemento (usa contexto ya transformado)
function drawSelectionIndicator(ctx, element) {
  ctx.save(); // Guardar estado del contexto (color, etc.)
  ctx.strokeStyle = 'dodgerblue';
  ctx.lineWidth = 1 / zoomLevel.value; // Hacer la línea más fina al hacer zoom in
  ctx.setLineDash([4 / zoomLevel.value, 2 / zoomLevel.value]); // Escalar dash
  ctx.globalCompositeOperation = 'source-over';

  let { x, y, width, height } = getElementBounds(element);
  const padding = 5 / zoomLevel.value; // Padding visual constante

  ctx.strokeRect(
    x - padding,
    y - padding,
    width + padding * 2,
    height + padding * 2
  );
  ctx.restore(); // Restaurar color, lineWidth, etc.
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

// --- Bounding Box y Hit Testing (Adaptados para Texto) ---
function getElementBounds(element) {
  let minX = Infinity,
    minY = Infinity,
    maxX = -Infinity,
    maxY = -Infinity;
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
      if (!element.points || element.points.length === 0)
        return { x: 0, y: 0, width: 0, height: 0 };
      element.points.forEach((p) => {
        minX = Math.min(minX, p.x);
        minY = Math.min(minY, p.y);
        maxX = Math.max(maxX, p.x);
        maxY = Math.max(maxY, p.y);
      });
      const halfSize = (element.size || 1) / 2;
      return {
        x: minX - halfSize,
        y: minY - halfSize,
        width: maxX - minX + element.size,
        height: maxY - minY + element.size,
      };
    case 'text':
      if (!canvasContext.value || !element.text)
        return { x: element.x, y: element.y, width: 0, height: 0 };
      // Estimar bounds del texto (esto puede ser impreciso)
      const ctx = canvasContext.value;
      ctx.save();
      ctx.font = `${element.fontSize}px sans-serif`;
      const metrics = ctx.measureText(element.text);
      ctx.restore();
      // La altura es más difícil, aproximamos con fontSize
      const width = metrics.width;
      const height = element.fontSize; // Aproximación simple
      // Ajustar por alineación (asumimos top-left)
      return { x: element.x, y: element.y, width: width, height: height };
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
      parseInt(currentSize.value, 10) / 2 / zoomLevel.value +
      2 / zoomLevel.value;
  } else if (element.type === 'pencil') {
    tolerance = Math.max(
      baseTolerance,
      ((element.size || 5) / 2 + 3) / zoomLevel.value
    );
  } else if (element.type === 'text') {
    tolerance = baseTolerance; // Tolerancia normal para texto
  } else {
    tolerance = baseTolerance / 2;
  }

  // Bounding Box Check (coordenadas del canvas)
  if (
    px < bounds.x - tolerance ||
    px > bounds.x + bounds.width + tolerance ||
    py < bounds.y - tolerance ||
    py > bounds.y + bounds.height + tolerance
  ) {
    return false;
  }

  // Specific Check
  switch (element.type) {
    case 'rectangle':
    case 'text': // Para texto, el BBox check es suficiente por ahora
      return true;
    case 'circle':
      const distSq = Math.pow(px - element.x, 2) + Math.pow(py - element.y, 2);
      const radiusSq = Math.pow(element.radius + tolerance, 2);
      return distSq <= radiusSq;
    case 'pencil':
      if (!element.points || element.points.length < 2) return true; // Hit si es un punto y está en bounds
      for (let i = 0; i < element.points.length - 1; i++) {
        const dist = pointToLineSegmentDistance(
          px,
          py,
          element.points[i],
          element.points[i + 1]
        );
        if (dist <= tolerance) return true;
      }
      return false;
    default:
      return false;
  }
}

function pointToLineSegmentDistance(px, py, p1, p2) {
  // ... (sin cambios, opera en coordenadas del canvas) ...
  const l2 = Math.pow(p1.x - p2.x, 2) + Math.pow(p1.y - p2.y, 2);
  if (l2 === 0)
    return Math.sqrt(Math.pow(px - p1.x, 2) + Math.pow(py - p1.y, 2));
  let t = ((px - p1.x) * (p2.x - p1.x) + (py - p1.y) * (p2.y - p1.y)) / l2;
  t = Math.max(0, Math.min(1, t));
  const closestX = p1.x + t * (p2.x - p1.x);
  const closestY = p1.y + t * (p2.y - p1.y);
  return Math.sqrt(Math.pow(px - closestX, 2) + Math.pow(py - closestY, 2));
}

function isElementInsideRectangle(element, rect) {
  // ... (sin cambios, opera en coordenadas del canvas) ...
  const elementBounds = getElementBounds(element);
  const selX = Math.min(rect.x1, rect.x2);
  const selY = Math.min(rect.y1, rect.y2);
  const selW = Math.abs(rect.x1 - rect.x2);
  const selH = Math.abs(rect.y1 - rect.y2);
  return (
    elementBounds.x < selX + selW &&
    elementBounds.x + elementBounds.width > selX &&
    elementBounds.y < selY + selH &&
    elementBounds.y + elementBounds.height > selY
  );
}

// --- Manejadores de Eventos del Canvas ---

// Mousedown solo para acciones DENTRO del canvas (dibujar, seleccionar)
function handleCanvasMouseDown(event) {
  // Ignorar si estamos paneando con espacio (gestionado por handlePanStart)
  if (isSpacePressed.value) return;
  // Ignorar clicks que no sean el izquierdo o touch
  if (event.type !== 'touchstart' && event.button !== 0) return;

  containerRef.value?.focus(); // Asegurar foco para atajos
  const { x, y } = getEventCoordinates(event); // Coordenadas del canvas
  startX.value = x;
  startY.value = y;
  isDrawing.value = false;
  isDragging.value = false;
  isSelectingArea.value = false;
  deletedDuringCurrentErasure.value = false;

  if (currentTool.value === 'select') {
    const isShiftPressed = event.shiftKey;
    let hitElement = null;
    for (let i = elements.value.length - 1; i >= 0; i--) {
      if (isPointInsideElement(x, y, elements.value[i], 'select')) {
        hitElement = elements.value[i];
        break;
      }
    }
    if (hitElement) {
      const hitElementId = hitElement.id;
      const isSelected = selectedElements.value.has(hitElementId);
      if (isShiftPressed) {
        isSelected
          ? selectedElements.value.delete(hitElementId)
          : selectedElements.value.add(hitElementId);
      } else {
        if (!isSelected) {
          selectedElements.value.clear();
          selectedElements.value.add(hitElementId);
        }
        isDragging.value = true;
        initialDragPositions.value.clear();
        selectedElements.value.forEach((id) => {
          const el = elements.value.find((e) => e.id === id);
          if (el) {
            if (el.type === 'pencil')
              initialDragPositions.value.set(id, {
                points: JSON.parse(JSON.stringify(el.points)),
              });
            else initialDragPositions.value.set(id, { x: el.x, y: el.y }); // Rect, Circle, Text
          }
        });
      }
    } else {
      if (!isShiftPressed) selectedElements.value.clear();
      isSelectingArea.value = true;
      selectionStartX.value = x;
      selectionStartY.value = y;
      selectionEndX.value = x;
      selectionEndY.value = y;
    }
    redrawCanvas();
  } else if (currentTool.value === 'eraser') {
    isDrawing.value = true;
    handleEraserAction(x, y); // Intentar borrar en el punto del click
  } else if (currentTool.value === 'text') {
    // Si hacemos clic con Texto y no estamos editando, iniciamos edición
    if (!isEditingText.value) {
      // Si clickeamos sobre texto existente, lo editamos en vez de crear nuevo
      let hitTextElement = null;
      for (let i = elements.value.length - 1; i >= 0; i--) {
        const el = elements.value[i];
        if (el.type === 'text' && isPointInsideElement(x, y, el, 'text')) {
          hitTextElement = el;
          break;
        }
      }
      if (hitTextElement) {
        startTextEditing(hitTextElement);
      } else {
        // Crear nuevo texto en la posición del clic
        startTextEditing(null, x, y);
      }
    } else {
      // Si ya estábamos editando y hacemos clic fuera, confirmamos
      handleTextInputConfirm();
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
        points: [{ x, y }],
      });
      redrawCanvas();
    } else if (
      currentTool.value === 'rectangle' ||
      currentTool.value === 'circle'
    ) {
      snapshot.value = canvasContext.value?.getImageData(
        0,
        0,
        canvasRef.value.width,
        canvasRef.value.height
      );
    }
  }
}

function handleCanvasMouseMove(event) {
  // Actualizar posición relativa del ratón para preview lápiz (incluso si no hay acción)
  mousePos.value = getRelativeScreenCoordinates(event);

  if (!isDrawing.value && !isDragging.value && !isSelectingArea.value) return;

  const { x, y } = getEventCoordinates(event); // Coordenadas del canvas

  if (isDragging.value) {
    const dx = x - startX.value; // Delta en coordenadas de canvas
    const dy = y - startY.value;
    selectedElements.value.forEach((id) => {
      const element = elements.value.find((el) => el.id === id);
      const initialPos = initialDragPositions.value.get(id);
      if (element && initialPos) {
        if (element.type === 'pencil') {
          element.points = initialPos.points.map((p) => ({
            x: p.x + dx,
            y: p.y + dy,
          }));
        } else {
          // Rect, Circle, Text
          element.x = initialPos.x + dx;
          element.y = initialPos.y + dy;
        }
      }
    });
    redrawCanvas();
  } else if (isSelectingArea.value) {
    selectionEndX.value = x;
    selectionEndY.value = y;
    redrawCanvas();
  } else if (isDrawing.value) {
    // Dibujo o Borrado activo
    if (currentTool.value === 'eraser') {
      handleEraserAction(x, y);
    } else if (currentTool.value === 'pencil') {
      const element = elements.value.find((el) => el.id === currentElementId);
      if (element) {
        element.points.push({ x, y });
        redrawCanvas(); // Optimización: solo dibujar el último segmento?
      }
    } else if (
      currentTool.value === 'rectangle' ||
      currentTool.value === 'circle'
    ) {
      // Previsualización de formas usando snapshot
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
          startX.value,
          startY.value,
          x - startX.value,
          y - startY.value
        );
      } else {
        const radius = Math.sqrt(
          Math.pow(x - startX.value, 2) + Math.pow(y - startY.value, 2)
        );
        ctx.arc(startX.value, startY.value, radius, 0, 2 * Math.PI);
        ctx.stroke();
      }
      ctx.restore();
    }
  }
}

function handleCanvasMouseUp(event) {
  // Ignorar si estamos paneando con espacio
  if (isSpacePressed.value) return;
  // Ignorar clicks que no sean el izquierdo o touch
  if (event.type !== 'touchend' && event.button !== 0) return;

  if (isDragging.value) {
    if (initialDragPositions.value.size > 0) saveState();
    isDragging.value = false;
    initialDragPositions.value.clear();
  } else if (isSelectingArea.value) {
    isSelectingArea.value = false;
    const selectionRect = {
      x1: selectionStartX.value,
      y1: selectionStartY.value,
      x2: selectionEndX.value,
      y2: selectionEndY.value,
    };
    elements.value.forEach((element) => {
      if (isElementInsideRectangle(element, selectionRect)) {
        if (!selectedElements.value.has(element.id)) {
          selectedElements.value.add(element.id);
          // No guardar estado por seleccionar
        }
      }
    });
    redrawCanvas();
  } else if (isDrawing.value) {
    // Finalizar Dibujo o Borrado
    if (currentTool.value === 'eraser') {
      if (deletedDuringCurrentErasure.value) saveState();
    } else {
      // Finalizar Dibujo
      const { x, y } = getEventCoordinates(event);
      let addedOrModifiedElement = false;
      const element = elements.value.find((el) => el.id === currentElementId);

      if (currentTool.value === 'pencil') {
        if (element && element.points.length === 1) {
          element.points.push({
            x: element.points[0].x + 0.01,
            y: element.points[0].y,
          });
        }
        if (element && element.points.length > 1) addedOrModifiedElement = true;
        // Si se dibujó muy poco (casi un clic), quizás no guardar? Por ahora guardamos.
      } else if (currentTool.value === 'rectangle') {
        const width = x - startX.value;
        const height = y - startY.value;
        if (Math.abs(width) > 1 || Math.abs(height) > 1) {
          // Crear elemento definitivo (reemplaza el temporal si existiera)
          elements.value = elements.value.filter(
            (el) => el.id !== currentElementId
          ); // Remover temporal si existe
          elements.value.push({
            id: currentElementId,
            type: 'rectangle',
            x: width > 0 ? startX.value : x,
            y: height > 0 ? startY.value : y,
            width: Math.abs(width),
            height: Math.abs(height),
            color: currentColor.value,
            size: parseInt(currentSize.value, 10),
          });
          addedOrModifiedElement = true;
        } else {
          elements.value = elements.value.filter(
            (el) => el.id !== currentElementId
          ); // Remover si es inválido
        }
      } else if (currentTool.value === 'circle') {
        const radius = Math.sqrt(
          Math.pow(x - startX.value, 2) + Math.pow(y - startY.value, 2)
        );
        if (radius > 1) {
          elements.value = elements.value.filter(
            (el) => el.id !== currentElementId
          );
          elements.value.push({
            id: currentElementId,
            type: 'circle',
            x: startX.value,
            y: startY.value,
            radius: radius,
            color: currentColor.value,
            size: parseInt(currentSize.value, 10),
          });
          addedOrModifiedElement = true;
        } else {
          elements.value = elements.value.filter(
            (el) => el.id !== currentElementId
          );
        }
      }
      snapshot.value = null;
      redrawCanvas();
      if (addedOrModifiedElement) saveState();
    }
  }

  // Resetear estados generales al final
  isDrawing.value = false;
  isDragging.value = false;
  isSelectingArea.value = false;
  currentElementId = null;
  deletedDuringCurrentErasure.value = false;
}

function handleCanvasMouseEnter() {
  isMouseOverCanvas.value = true;
}
function handleCanvasMouseLeave(event) {
  isMouseOverCanvas.value = false;
  // Si salimos mientras dibujamos/arrastramos, finalizar acción
  if (isDrawing.value || isDragging.value || isSelectingArea.value) {
    console.log('Mouse Leave Canvas - Forcing end of action');
    // Simular un mouse up para finalizar limpiamente
    handleCanvasMouseUp(event);
  }
  // El handlePanEnd se encarga si salimos del *contenedor*
}

// --- Lógica de Texto ---

function startTextEditing(element = null, defaultX = 0, defaultY = 0) {
  selectedElements.value.clear(); // Deseleccionar todo al editar/crear texto

  if (element) {
    // Editando existente
    editingTextElementId.value = element.id;
    currentTextValue.value = element.text;
    textInputInitialValue.value = element.text; // Guardar valor original
    textInputCoords.value = { x: element.x, y: element.y };
    // Usar el tamaño y color del elemento existente para el input
    currentSize.value = element.fontSize; // Actualizar slider/valor
    currentColor.value = element.color; // Actualizar picker/valor
  } else {
    // Creando nuevo
    editingTextElementId.value = null;
    currentTextValue.value = '';
    textInputInitialValue.value = '';
    textInputCoords.value = { x: defaultX, y: defaultY };
    // Usar color y tamaño actuales de la barra de herramientas
  }

  isEditingText.value = true;
  nextTick(() => {
    textInputRef.value?.focus();
    autosizeTextarea(); // Ajustar tamaño inicial
  });
  redrawCanvas(); // Quitar selección si la había
}

// Confirmar texto (onBlur o Enter sin Shift)
function handleTextInputConfirm() {
  if (!isEditingText.value) return;

  const text = currentTextValue.value.trim();

  if (editingTextElementId.value) {
    // Editando existente
    const element = elements.value.find(
      (el) => el.id === editingTextElementId.value
    );
    if (element) {
      if (text) {
        // Solo guardar si el texto cambió
        if (text !== textInputInitialValue.value) {
          element.text = text;
          element.fontSize = parseInt(currentSize.value, 10); // Actualizar tamaño
          element.color = currentColor.value; // Actualizar color
          saveState();
        }
      } else {
        // Si se borró todo el texto, eliminamos el elemento
        elements.value = elements.value.filter(
          (el) => el.id !== editingTextElementId.value
        );
        saveState();
      }
    }
  } else {
    // Creando nuevo
    if (text) {
      elements.value.push({
        id: generateId(),
        type: 'text',
        x: textInputCoords.value.x,
        y: textInputCoords.value.y,
        text: text,
        color: currentColor.value,
        fontSize: parseInt(currentSize.value, 10),
      });
      saveState();
    }
  }

  // Resetear estado de edición
  isEditingText.value = false;
  currentTextValue.value = '';
  editingTextElementId.value = null;
  textInputInitialValue.value = '';
  redrawCanvas();
}

// Cancelar edición (Escape)
function handleTextInputCancel() {
  isEditingText.value = false;
  currentTextValue.value = '';
  editingTextElementId.value = null;
  textInputInitialValue.value = '';
  redrawCanvas(); // Redibujar por si había algo seleccionado antes
}

// Manejar Enter en textarea (Enter confirma, Shift+Enter nueva línea)
function handleTextInputEnter(event) {
  if (!event.shiftKey) {
    event.preventDefault(); // Prevenir nueva línea
    handleTextInputConfirm();
  }
  // Si es Shift+Enter, el comportamiento por defecto (nueva línea) está bien
  // Ajustar tamaño del textarea después de posible nueva línea
  nextTick(autosizeTextarea);
}

// Ajustar altura del textarea al contenido
function autosizeTextarea() {
  const textarea = textInputRef.value;
  if (textarea) {
    textarea.style.height = 'auto'; // Resetear altura
    textarea.style.height = `${textarea.scrollHeight}px`;
    // Opcional: ajustar ancho también (más complejo)
    // textarea.style.width = 'auto';
    // textarea.style.width = `${textarea.scrollWidth}px`;
  }
}
// Observar cambios en el texto para autoajustar tamaño
watch(currentTextValue, () => {
  if (isEditingText.value) {
    nextTick(autosizeTextarea);
  }
});

// Editar texto con doble clic
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
    event.preventDefault();
    // Si ya estábamos editando, primero confirmamos el anterior
    if (isEditingText.value) {
      handleTextInputConfirm();
    }
    // Luego iniciamos la edición del nuevo
    startTextEditing(hitTextElement);
  }
}

// --- Borrado ---
function handleEraserAction(x, y) {
  let elementDeleted = false;
  for (let i = elements.value.length - 1; i >= 0; i--) {
    const element = elements.value[i];
    if (isPointInsideElement(x, y, element, 'eraser')) {
      elements.value.splice(i, 1);
      selectedElements.value.delete(element.id); // Deseleccionar si estaba seleccionado
      deletedDuringCurrentErasure.value = true;
      elementDeleted = true;
      break;
    }
  }
  if (elementDeleted) redrawCanvas();
  return elementDeleted;
}

// --- Acciones Toolbar y Teclado ---
function setTool(tool) {
  currentTool.value = tool;
  // Si cambiamos a Texto y estábamos editando, confirmamos
  if (tool !== 'text' && isEditingText.value) {
    handleTextInputConfirm();
  }
  // Deseleccionar al cambiar a herramienta de dibujo/texto/borrador
  if (tool !== 'select') {
    selectedElements.value.clear();
    redrawCanvas();
  }
}

function clearCanvas() {
  elements.value = [];
  selectedElements.value.clear();
  if (isEditingText.value) handleTextInputCancel(); // Cancelar edición
  resetZoomAndPan(); // Resetear vista también
  // saveState(); // Guardar estado vacío (redrawCanvas ya lo hace implícitamente)
  // RedrawCanvas lo hace resetZoomAndPan, no es necesario llamarlo de nuevo
  saveState(); // Guardar estado vacío en historial
}

// Guardar/Restaurar Estado (Undo/Redo)
function saveState() {
  // Cancelar edición de texto antes de guardar
  if (isEditingText.value) {
    // Podríamos intentar confirmar, pero cancelar es más seguro para estado
    handleTextInputCancel();
  }

  if (historyIndex.value < history.value.length - 1) {
    history.value = history.value.slice(0, historyIndex.value + 1);
  }
  const currentState = { elements: JSON.parse(JSON.stringify(elements.value)) };
  if (
    history.value.length > 0 &&
    JSON.stringify(currentState.elements) ===
      JSON.stringify(history.value[historyIndex.value]?.elements)
  ) {
    return; // No guardar estados idénticos
  }
  history.value.push(currentState);
  historyIndex.value++;
  const MAX_HISTORY = 50;
  if (history.value.length > MAX_HISTORY) {
    history.value.shift();
    historyIndex.value--;
  }
}

function restoreState(index) {
  if (index >= 0 && index < history.value.length) {
    // Cancelar edición antes de restaurar
    if (isEditingText.value) {
      handleTextInputCancel();
    }
    const stateToRestore = JSON.parse(JSON.stringify(history.value[index]));
    elements.value = stateToRestore.elements;
    historyIndex.value = index;
    selectedElements.value.clear(); // Limpiar selección
    redrawCanvas();
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
  if (event.key === ' ' && !isSpacePressed.value) {
    // Prevenir scroll de página si el foco está en el container
    if (document.activeElement === containerRef.value) {
      event.preventDefault();
    }
    isSpacePressed.value = true;
    // El cursor cambiará a 'grab' vía computed property
  }
}
function handleGlobalKeyUp(event) {
  if (event.key === ' ') {
    isSpacePressed.value = false;
    // Si estábamos paneando con espacio, terminarlo
    if (isPanning.value) {
      isPanning.value = false;
    }
    // El cursor volverá a la normalidad
  }
}

// Atajos específicos del componente (cuando tiene foco)
function handleKeyDown(event) {
  // No interferir si se está editando texto o en otro input
  if (
    isEditingText.value ||
    event.target.tagName === 'INPUT' ||
    event.target.tagName === 'TEXTAREA'
  ) {
    // Permitir Esc y Enter (manejados por el input)
    if (
      isEditingText.value &&
      (event.key === 'Escape' || event.key === 'Enter')
    ) {
      // Ya tienen sus propios handlers en el textarea
    } else {
      return; // Ignorar otros atajos mientras se edita
    }
  }
  // No interferir si se está paneando con espacio
  if (isSpacePressed.value) return;

  const isCtrlOrMeta = event.ctrlKey || event.metaKey;

  if (isCtrlOrMeta) {
    switch (event.key.toLowerCase()) {
      case 'z':
        event.preventDefault();
        undo();
        break;
      case 'y':
      case 'shift' + 'z':
        if (
          (event.shiftKey && event.key.toLowerCase() === 'z') ||
          event.key.toLowerCase() === 'y'
        ) {
          event.preventDefault();
          redo();
        }
        break;
      case 'a':
        event.preventDefault();
        selectedElements.value = new Set(elements.value.map((el) => el.id));
        currentTool.value = 'select';
        redrawCanvas();
        break;
      case '0':
      case '+':
      case '-': // Atajos de Zoom (opcional)
        event.preventDefault();
        if (event.key === '0') resetZoomAndPan();
        // else if (event.key === '+') handleWheel({deltaY: -1, clientX: canvasRef.value.width/2, clientY: canvasRef.value.height/2}); // Simular zoom in centrado
        // else if (event.key === '-') handleWheel({deltaY: 1, clientX: canvasRef.value.width/2, clientY: canvasRef.value.height/2}); // Simular zoom out centrado
        break;
    }
  } else {
    // Atajos sin modificador
    switch (event.key.toLowerCase()) {
      case 'v':
        event.preventDefault();
        setTool('select');
        break;
      case 'p':
        event.preventDefault();
        setTool('pencil');
        break;
      case 'e':
        event.preventDefault();
        setTool('eraser');
        break;
      case 'r':
        event.preventDefault();
        setTool('rectangle');
        break;
      case 'c':
        event.preventDefault();
        setTool('circle');
        break;
      case 't':
        event.preventDefault();
        setTool('text');
        break; // Atajo Texto
      case 'delete':
      case 'backspace':
        if (selectedElements.value.size > 0) {
          event.preventDefault();
          const originalSize = elements.value.length;
          elements.value = elements.value.filter(
            (el) => !selectedElements.value.has(el.id)
          );
          const changed = elements.value.length !== originalSize;
          selectedElements.value.clear();
          redrawCanvas();
          if (changed) saveState();
        }
        break;
      case 'escape':
        if (selectedElements.value.size > 0) {
          event.preventDefault();
          selectedElements.value.clear();
          redrawCanvas();
        } else if (isSelectingArea.value) {
          event.preventDefault();
          isSelectingArea.value = false;
          redrawCanvas();
        }
        // Escape también cancela edición de texto (manejado en el input)
        break;
    }
  }
}

function generateId() {
  return Date.now().toString(36) + Math.random().toString(36).substring(2, 7);
}

// --- Observadores ---
// Actualizar elementos seleccionados al cambiar color/tamaño
watch(currentColor, (newColor) => {
  if (selectedElements.value.size > 0 && currentTool.value === 'select') {
    let changed = false;
    elements.value.forEach((el) => {
      if (selectedElements.value.has(el.id)) {
        el.color = newColor;
        changed = true;
      }
    });
    if (changed) {
      redrawCanvas();
      saveState();
    }
  }
});
watch(currentSize, (newSizeStr) => {
  if (selectedElements.value.size > 0 && currentTool.value === 'select') {
    const newSize = parseInt(newSizeStr, 10);
    let changed = false;
    elements.value.forEach((el) => {
      if (selectedElements.value.has(el.id)) {
        if (el.type === 'text')
          el.fontSize = newSize; // Usar para fontSize en texto
        else el.size = newSize; // Usar para size en otros
        changed = true;
      }
    });
    if (changed) {
      redrawCanvas();
      saveState();
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
  background-color: #dodgerblue;
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
  opacity: 0.7;
  border: 1px solid rgba(255, 255, 255, 0.5); /* Borde blanco semitransparente */
  mix-blend-mode: difference; /* Intenta hacer el cursor visible sobre cualquier color */
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
  z-index: 101;
  font-family: sans-serif; /* Asegurar fuente consistente */
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  line-height: 1.2; /* Ajuste básico */
}

/* Añadir estilo para indicar paneo activo */
.whiteboard-container:has(canvas:active[style*='grabbing']) {
  /* O alguna clase que se añada durante el pan */
}
</style>
