<template>
  <div id="app">
    <h2>Let's Collab Draw App (Vue)</h2>

    <div id="controls">
      <input type="color" v-model="currentColor" />
      <button @click="undo">Undo</button>
      <button @click="clearCanvas">Clear</button>
    </div>

    <canvas
      ref="canvas"
      :width="canvasWidth"
      :height="canvasHeight"
      @mousedown="startDrawing"
      @mousemove="draw"
      @mouseup="stopDrawing"
      @mouseleave="stopDrawing"
      @touchstart.prevent="startDrawingTouch"
      @touchmove.prevent="drawTouch"
      @touchend.prevent="stopDrawingTouch"
    ></canvas>
  </div>
</template>

<script>
export default {
  data() {
    return {
      currentColor: "#000000",
      drawing: false,
      currentStroke: [],
      strokes: [],
      ctx: null,
      socket: null,
      canvasWidth: 800,
      canvasHeight: 600
    };
  },
  mounted() {
    const canvas = this.$refs.canvas;
    this.ctx = canvas.getContext("2d");

    // Adjust for mobile view
    this.resizeCanvas();
    window.addEventListener("resize", this.resizeCanvas);

    // Connect to backend
    this.socket = new WebSocket("wss://web-production-0f84.up.railway.app/ws");

    this.socket.onmessage = (event) => {
      const message = JSON.parse(event.data);
      if (message.type === "start") {
        this.currentStroke = [{ x: message.x, y: message.y, color: message.color }];
      } else if (message.type === "draw") {
        this.currentStroke.push({ x: message.x, y: message.y, color: message.color });
        this.drawLine(this.currentStroke);
      } else if (message.type === "endStroke") {
        this.strokes.push(this.currentStroke);
        this.currentStroke = [];
      } else if (message.type === "clear") {
        this.strokes = [];
        this.ctx.clearRect(0, 0, canvas.width, canvas.height);
      } else if (message.type === "undo") {
        this.strokes.pop();
        this.redrawAll();
      }
    };
  },
  beforeUnmount() {
    window.removeEventListener("resize", this.resizeCanvas);
  },
  methods: {
    resizeCanvas() {
      const maxWidth = window.innerWidth - 40; // padding
      this.canvasWidth = maxWidth < 800 ? maxWidth : 800;
      this.canvasHeight = (this.canvasWidth / 4) * 3; // Keep aspect ratio
      this.redrawAll();
    },
    sendMessage(data) {
      if (this.socket && this.socket.readyState === WebSocket.OPEN) {
        this.socket.send(JSON.stringify(data));
      }
    },
    getPos(e) {
      const rect = this.$refs.canvas.getBoundingClientRect();
      return {
        x: e.clientX - rect.left,
        y: e.clientY - rect.top
      };
    },
    getPosTouch(e) {
      const rect = this.$refs.canvas.getBoundingClientRect();
      return {
        x: e.touches[0].clientX - rect.left,
        y: e.touches[0].clientY - rect.top
      };
    },
    startDrawing(e) {
      this.drawing = true;
      const pos = this.getPos(e);
      this.currentStroke = [{ x: pos.x, y: pos.y, color: this.currentColor }];
      this.ctx.beginPath();
      this.ctx.moveTo(pos.x, pos.y);
      this.ctx.strokeStyle = this.currentColor;
      this.sendMessage({ type: "start", ...pos, color: this.currentColor });
    },
    draw(e) {
      if (!this.drawing) return;
      const pos = this.getPos(e);
      this.currentStroke.push({ x: pos.x, y: pos.y, color: this.currentColor });
      this.ctx.lineTo(pos.x, pos.y);
      this.ctx.stroke();
      this.sendMessage({ type: "draw", ...pos, color: this.currentColor });
    },
    stopDrawing() {
      if (!this.drawing) return;
      this.drawing = false;
      this.strokes.push(this.currentStroke);
      this.currentStroke = [];
      this.sendMessage({ type: "endStroke" });
    },
    startDrawingTouch(e) {
      this.drawing = true;
      const pos = this.getPosTouch(e);
      this.currentStroke = [{ x: pos.x, y: pos.y, color: this.currentColor }];
      this.ctx.beginPath();
      this.ctx.moveTo(pos.x, pos.y);
      this.ctx.strokeStyle = this.currentColor;
      this.sendMessage({ type: "start", ...pos, color: this.currentColor });
    },
    drawTouch(e) {
      if (!this.drawing) return;
      const pos = this.getPosTouch(e);
      this.currentStroke.push({ x: pos.x, y: pos.y, color: this.currentColor });
      this.ctx.lineTo(pos.x, pos.y);
      this.ctx.stroke();
      this.sendMessage({ type: "draw", ...pos, color: this.currentColor });
    },
    stopDrawingTouch() {
      if (!this.drawing) return;
      this.drawing = false;
      this.strokes.push(this.currentStroke);
      this.currentStroke = [];
      this.sendMessage({ type: "endStroke" });
    },
    drawLine(stroke) {
      if (stroke.length < 2) return;
      this.ctx.beginPath();
      this.ctx.strokeStyle = stroke[0].color;
      this.ctx.moveTo(stroke[0].x, stroke[0].y);
      for (let i = 1; i < stroke.length; i++) {
        this.ctx.lineTo(stroke[i].x, stroke[i].y);
      }
      this.ctx.stroke();
    },
    redrawAll() {
      this.ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight);
      for (const stroke of this.strokes) {
        this.drawLine(stroke);
      }
    },
    undo() {
      this.strokes.pop();
      this.redrawAll();
      this.sendMessage({ type: "undo" });
    },
    clearCanvas() {
      this.strokes = [];
      this.ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight);
      this.sendMessage({ type: "clear" });
    }
  }
};
</script>

<style scoped>
#app {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  text-align: center;
  background: linear-gradient(135deg, #ece9e6, #ffffff);
  min-height: 100vh;
  padding-top: 80px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

h2 {
  font-size: 2rem;
  color: #333;
  margin-bottom: 20px;
  font-weight: 600;
}

#controls {
  position: fixed;
  top: 15px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  align-items: center;
  background: white;
  padding: 10px 20px;
  border-radius: 8px;
  box-shadow: 0px 4px 12px rgba(0,0,0,0.08);
  gap: 10px;
  z-index: 10;
}

input[type="color"] {
  border: none;
  background: transparent;
  width: 40px;
  height: 40px;
  cursor: pointer;
  padding: 0;
}

button {
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 8px 16px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s ease-in-out;
}

button:hover {
  background: #45a049;
  transform: translateY(-2px);
}

button:active {
  transform: scale(0.97);
}

canvas {
  background: white;
  border: 2px solid #ddd;
  border-radius: 10px;
  box-shadow: 0px 4px 20px rgba(0,0,0,0.1);
  touch-action: none;
  max-width: 100%;
  height: auto;
}

/* Mobile adjustments */
@media (max-width: 600px) {
  #controls {
    flex-wrap: wrap;
    gap: 8px;
    padding: 8px 12px;
    justify-content: center;
  }

  input[type="color"] {
    width: 35px;
    height: 35px;
  }

  button {
    padding: 8px 14px;
    font-size: 14px;
  }
}
</style>