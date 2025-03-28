# <img src="logo.png" alt="ANIMEDGE Logo">
 </head>
 <body>
    <div id="canvasContainer">
       <canvas id="animationCanvas" width="800" height="500"></canvas>
    </div>
 
    <!-- Control Panel with Color and Brush Size Options -->
    <div id="controlPanel">
       <button id="undoButton">Undo</button>
       <button id="redoButton">Redo</button>
       <button id="clearButton">Clear Frame</button>
       <div id="drawingTools">
          <label>Color: </label>
          <select id="colorSelect">
             <option value="black" selected>Black</option>
             <option value="red">Red</option>
             <option value="blue">Blue</option>
             <option value="green">Green</option>
          </select>
          <label>Brush Size: </label>
          <select id="brushSizeSelect">
             <option value="1">1px</option>
             <option value="5" selected>5px</option>
             <option value="10">10px</option>
             <option value="20">20px</option>
          </select>
       </div>
    </div>
 
    <!-- Timeline (Blender Exact Replica) -->
    <div id="timelinePanel">
       <div id="timelineHeader"></div>
       <div id="timelineControls">
          <button id="addFrameButton">Add Frame</button>
          <button id="deleteFrameButton">Delete Frame</button>
          <button id="playButton">Play</button>
          <button id="stopButton">Stop</button>
          <button id="onionSkinButton">Onion Skin Off</button>
          <button id="zoomInButton">+</button>
          <button id="zoomOutButton">-</button>
          <label><input type="checkbox" id="loopToggle"> Loop</label>
          <input type="number" id="loopStart" placeholder="Start" min="1" style="width: 50px;">
          <input type="number" id="loopEnd" placeholder="End" min="1" style="width: 50px;">
       </div>
       <div id="frameList">
          <div id="playhead"><div id="playheadTop"></div></div>
       </div>
    </div>
 
    <script>
       const canvas = document.getElementById("animationCanvas");
       const ctx = canvas.getContext("2d");
       const frameList = document.getElementById("frameList");
       const timelineHeader = document.getElementById("timelineHeader");
       const playhead = document.getElementById("playhead");
       const playheadTop = document.getElementById("playheadTop");
       const onionSkinButton = document.getElementById("onionSkinButton");
       const loopToggle = document.getElementById("loopToggle");
       const loopStart = document.getElementById("loopStart");
       const loopEnd = document.getElementById("loopEnd");
       const colorSelect = document.getElementById("colorSelect");
       const brushSizeSelect = document.getElementById("brushSizeSelect");
 
       let drawing = false;
       let frames = [{ data: canvas.toDataURL(), position: 1 }];
       let currentFrame = 0;
       let isPlaying = false;
       let playInterval;
       let maxFramePosition = 1;
       let lastDisplayedFrameData = null;
       let history = [canvas.toDataURL()];
       let redoStack = [];
       let zoomLevel = 20; // Pixels per frame
       const FPS = 24; // Blender default FPS
       const TIMELINE_MAX = 1000; // Large max for continuous timeline
       let currentPlaybackPosition = 1; // Tracks playhead position
       let onionSkinEnabled = false;
       let currentColor = "black"; // Default color
       let currentBrushSize = 5; // Default brush size
 
       // Load a frame with onion skinning
       function loadFrame(index) {
          currentFrame = index;
          ctx.clearRect(0, 0, canvas.width, canvas.height);
 
          if (onionSkinEnabled && currentFrame > 0 && !isPlaying) {
             let prevImg = new Image();
             prevImg.src = frames[currentFrame - 1].data;
             prevImg.onload = () => {
                ctx.globalAlpha = 0.3;
                ctx.drawImage(prevImg, 0, 0);
                ctx.globalAlpha = 1.0;
                drawCurrentFrame();
             };
          } else {
             drawCurrentFrame();
          }
          currentPlaybackPosition = frames[currentFrame].position;
          updateTimeline();
       }
 
       // Draw the current frame
       function drawCurrentFrame() {
          let img = new Image();
          img.src = frames[currentFrame].data;
          img.onload = () => ctx.drawImage(img, 0, 0);
       }
 
       // Undo
       function undo() {
          if (history.length > 1) {
             redoStack.push(history.pop());
             frames[currentFrame].data = history[history.length - 1];
             loadFrame(currentFrame);
          }
       }
 
       // Redo
       function redo() {
          if (redoStack.length > 0) {
             history.push(redoStack.pop());
             frames[currentFrame].data = history[history.length - 1];
             loadFrame(currentFrame);
          }
       }
 
       // Clear frame
       function clearCanvas() {
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          history = [canvas.toDataURL()];
          redoStack = [];
          frames[currentFrame].data = canvas.toDataURL();
          loadFrame(currentFrame);
       }
 
       // Drawing functions
       function startDrawing(event) {
          if (!drawing && !isPlaying) {
             history.push(frames[currentFrame].data);
             redoStack = [];
          }
          drawing = true;
          ctx.beginPath();
       }
 
       function stopDrawing() {
          if (drawing) {
             frames[currentFrame].data = canvas.toDataURL();
          }
          drawing = false;
          loadFrame(currentFrame);
       }
 
       function draw(event) {
          if (!drawing || isPlaying) return;
          ctx.lineWidth = currentBrushSize; // Use current brush size
          ctx.lineCap = "round";
          ctx.strokeStyle = currentColor; // Use current color
          ctx.lineTo(event.clientX - canvas.offsetLeft, event.clientY - canvas.offsetTop);
          ctx.stroke();
          ctx.beginPath();
          ctx.moveTo(event.clientX - canvas.offsetLeft, event.clientY - canvas.offsetTop);
       }
 
       // Add frame after current
       function addFrame() {
          const newPosition = frames[currentFrame].position + 1;
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          const blankState = canvas.toDataURL();
          const newFrame = { data: blankState, position: newPosition };
          frames.splice(currentFrame + 1, 0, newFrame);
          maxFramePosition = Math.max(maxFramePosition, newPosition);
          currentFrame++;
          updateLoopEnd();
          loadFrame(currentFrame);
       }
 
       // Add frame at clicked position
       function addFrameAtPosition(event) {
          const rect = frameList.getBoundingClientRect();
          const clickX = event.clientX - rect.left;
          const framePos = Math.floor(clickX / zoomLevel) + 1;
 
          if (frames.some(f => f.position === framePos) || framePos > TIMELINE_MAX) return;
 
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          const blankState = canvas.toDataURL();
          const newFrame = { data: blankState, position: framePos };
          let insertIndex = frames.findIndex(f => f.position > framePos);
          if (insertIndex === -1) insertIndex = frames.length;
          frames.splice(insertIndex, 0, newFrame);
          maxFramePosition = Math.max(maxFramePosition, framePos);
          currentFrame = insertIndex;
          updateLoopEnd();
          loadFrame(currentFrame);
       }
 
       // Delete current frame
       function deleteFrame() {
          if (frames.length > 1) {
             frames.splice(currentFrame, 1);
             currentFrame = Math.min(currentFrame, frames.length - 1);
             maxFramePosition = frames.length > 0 ? Math.max(...frames.map(f => f.position)) : 1;
             updateLoopEnd();
             loadFrame(currentFrame);
          }
       }
 
       // Zoom in
       function zoomIn() {
          zoomLevel = Math.min(zoomLevel * 2, 80);
          updateTimeline();
       }
 
       // Zoom out
       function zoomOut() {
          zoomLevel = Math.max(zoomLevel / 2, 5);
          updateTimeline();
       }
 
       // Toggle onion skin
       function toggleOnionSkin() {
          onionSkinEnabled = !onionSkinEnabled;
          onionSkinButton.textContent = onionSkinEnabled ? "Onion Skin On" : "Onion Skin Off";
          loadFrame(currentFrame);
       }
 
       // Update loop end field to reflect the last frame
       function updateLoopEnd() {
          loopEnd.value = maxFramePosition;
       }
 
       // Update timeline with static time counter
       function updateTimeline() {
          timelineHeader.innerHTML = "";
          const tickInterval = zoomLevel >= 40 ? 1 : zoomLevel >= 20 ? 5 : 10;
          const visibleFrames = Math.ceil(window.innerWidth / zoomLevel) + 50;
 
          for (let i = 1; i <= visibleFrames; i += tickInterval) {
             const tick = document.createElement("div");
             tick.className = "timeTick";
             tick.textContent = i;
             tick.style.left = `${(i - 1) * zoomLevel}px`;
             timelineHeader.appendChild(tick);
          }
 
          frameList.innerHTML = '<div id="playhead"><div id="playheadTop"></div></div>';
          frames.forEach((frame, index) => {
             const frameDiv = document.createElement("div");
             frameDiv.className = "frameItem";
             frameDiv.style.left = `${(frame.position - 1) * zoomLevel}px`;
 
             const frameDot = document.createElement("div");
             frameDot.className = "frameDot";
             frameDot.style.backgroundColor = index === currentFrame ? "#007BFF" : "#ccc";
 
             const frameLabel = document.createElement("div");
             frameLabel.textContent = frame.position;
             frameLabel.style.color = "white";
             frameLabel.style.fontSize = "12px";
             frameLabel.style.textAlign = "center";
 
             frameDiv.appendChild(frameDot);
             frameDiv.appendChild(frameLabel);
             frameDiv.addEventListener("click", () => loadFrame(index));
             frameList.appendChild(frameDiv);
          });
 
          playhead.style.left = `${(currentPlaybackPosition - 1) * zoomLevel}px`;
          if (isPlaying) {
             frameList.scrollLeft = (currentPlaybackPosition - 1) * zoomLevel - window.innerWidth / 2 + 100;
          }
       }
 
       // Play animation with green playhead movement
       function playAnimation() {
          if (isPlaying) return;
          isPlaying = true;
          currentPlaybackPosition = frames[currentFrame].position;
 
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          lastDisplayedFrameData = null;
 
          playhead.classList.add("playing");
 
          playInterval = setInterval(() => {
             const frame = frames.find(f => f.position === currentPlaybackPosition);
             if (frame) {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                let img = new Image();
                img.src = frame.data;
                img.onload = () => {
                   ctx.globalAlpha = 1.0;
                   ctx.drawImage(img, 0, 0);
                   lastDisplayedFrameData = frame.data;
                };
                currentFrame = frames.indexOf(frame);
             } else if (lastDisplayedFrameData) {
                let img = new Image();
                img.src = lastDisplayedFrameData;
                img.onload = () => {
                   ctx.globalAlpha = 1.0;
                   ctx.drawImage(img, 0, 0);
                };
             }
 
             currentPlaybackPosition++;
             if (loopToggle.checked) {
                const start = parseInt(loopStart.value) || 1;
                const end = parseInt(loopEnd.value) || maxFramePosition;
                if (currentPlaybackPosition > end) currentPlaybackPosition = start;
             } else if (currentPlaybackPosition > TIMELINE_MAX) {
                stopAnimation();
                return;
             }
             updateTimeline();
          }, 1000 / FPS);
       }
 
       // Stop animation
       function stopAnimation() {
          isPlaying = false;
          clearInterval(playInterval);
          playhead.classList.remove("playing");
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          loadFrame(currentFrame);
       }
 
       // Toggle play/stop with spacebar
       function togglePlayStop(event) {
          if (event.code === "Space") {
             event.preventDefault();
             if (isPlaying) {
                stopAnimation();
             } else {
                playAnimation();
             }
          }
       }
 
       // Update drawing color and size
       function updateDrawingSettings() {
          currentColor = colorSelect.value;
          currentBrushSize = parseInt(brushSizeSelect.value);
       }
 
       // Event listeners
       document.getElementById("undoButton").addEventListener("click", undo);
       document.getElementById("redoButton").addEventListener("click", redo);
       document.getElementById("clearButton").addEventListener("click", clearCanvas);
       document.getElementById("addFrameButton").addEventListener("click", addFrame);
       document.getElementById("deleteFrameButton").addEventListener("click", deleteFrame);
       document.getElementById("playButton").addEventListener("click", playAnimation);
       document.getElementById("stopButton").addEventListener("click", stopAnimation);
       document.getElementById("zoomInButton").addEventListener("click", zoomIn);
       document.getElementById("zoomOutButton").addEventListener("click", zoomOut);
       onionSkinButton.addEventListener("click", toggleOnionSkin);
       frameList.addEventListener("dblclick", addFrameAtPosition);
 
       canvas.addEventListener("mousedown", startDrawing);
       canvas.addEventListener("mouseup", stopDrawing);
       canvas.addEventListener("mousemove", draw);
 
       document.addEventListener("keydown", togglePlayStop);
 
       // Add listeners for color and brush size changes
       colorSelect.addEventListener("change", updateDrawingSettings);
       brushSizeSelect.addEventListener("change", updateDrawingSettings);
 
       // Initialize
       window.onload = () => {
          clearCanvas();
          updateTimeline();
          updateLoopEnd();
          updateDrawingSettings(); // Set initial color and brush size
       };
    </script>
 
    <style>
       #controlPanel {
          position: fixed;
          top: 10px;
          left: 10px;
          background: rgba(0, 0, 0, 0.7);
          padding: 5px;
          border-radius: 5px;
       }
 
       #controlPanel button {
          color: white;
          background: #007BFF;
          border: none;
          padding: 5px 10px;
          margin: 2px;
          cursor: pointer;
          border-radius: 3px;
       }
 
       #controlPanel button:hover {
          background: #0056b3;
       }
 
       #drawingTools {
          margin-top: 5px;
       }
 
       #drawingTools label {
          color: white;
          margin-right: 5px;
          font-size: 12px;
       }
 
       #drawingTools select {
          padding: 2px;
          border-radius: 3px;
          background: #fff;
          color: #000;
          border: none;
          margin-right: 10px;
       }
 
       #timelinePanel {
          position: fixed;
          bottom: 0;
          left: 0;
          right: 0;
          height: 100px;
          background: rgba(50, 50, 50, 0.9);
          padding: 5px;
          display: flex;
          flex-direction: column;
          color: white;
       }
 
       #timelineHeader {
          height: 20px;
          position: relative;
          margin-bottom: 5px;
          background: rgba(40, 40, 40, 0.9);
       }
 
       .timeTick {
          position: absolute;
          top: 0;
          font-size: 12px;
          color: #aaa;
          width: 20px;
          text-align: center;
       }
 
       #timelineControls {
          display: flex;
          justify-content: flex-start;
          align-items: center;
          margin-bottom: 5px;
       }
 
       #timelineControls button {
          color: white;
          background: #007BFF;
          border: none;
          padding: 5px 10px;
          margin: 0 5px;
          cursor: pointer;
          border-radius: 3px;
       }
 
       #timelineControls button:hover {
          background: #0056b3;
       }
 
       #timelineControls label {
          margin-left: 10px;
          font-size: 12px;
       }
 
       #timelineControls input[type="number"] {
          margin-left: 5px;
          padding: 2px;
          border-radius: 3px;
          border: none;
       }
 
       #frameList {
          position: relative;
          height: 40px;
          overflow-x: auto;
          overflow-y: hidden;
          background: rgba(60, 60, 60, 0.9);
       }
 
       .frameItem {
          position: absolute;
          display: flex;
          flex-direction: column;
          align-items: center;
          width: 20px;
          cursor: pointer;
       }
 
       .frameDot {
          width: 8px;
          height: 8px;
          border-radius: 50%;
          margin-bottom: 3px;
       }
 
       #playhead {
          position: absolute;
          top: -25px;
          width: 2px;
          height: 65px;
          background-color: #00cc00;
          z-index: 10;
          transition: left 0.05s ease;
       }
 
       #playhead.playing {
          box-shadow: 0 0 5px #00cc00;
       }
 
       #playheadTop {
          position: absolute;
          top: -10px;
          left: -5px;
          width: 0;
          height: 0;
          border-left: 6px solid transparent;
          border-right: 6px solid transparent;
          border-bottom: 10px solid #00cc00;
       }
 
       #canvasContainer {
          display: flex;
          justify-content: center;
          align-items: center;
          height: calc(100vh - 100px);
       }
 
       #animationCanvas {
          border: 3px solid black;
          background-color: white;
       }
 
       body, html {
          margin: 0;
          padding: 0;
          overflow: hidden;
          background: #222;
       }
    </style>
 </body>
 </html>
