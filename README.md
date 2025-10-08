<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Live Face Shape Recognition</title>
  <style>
    :root{
      --bg:#071033; --card:#0f1724; --accent:#7c3aed; --muted:#94a3b8; --text:#e6eef8;
    }
    html,body{height:100%;margin:0;background:linear-gradient(180deg,#020617,var(--bg));font-family:Inter,system-ui,Segoe UI,Roboto,Arial;color:var(--text)}
    .container{max-width:980px;margin:28px auto;padding:18px;}
    .card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border-radius:12px;padding:18px;box-shadow:0 12px 40px rgba(2,6,23,0.6)}
    header{display:flex;gap:12px;align-items:center}
    .logo{width:56px;height:56px;border-radius:10px;background:linear-gradient(135deg,#7c3aed,#ef4444);display:grid;place-items:center;font-weight:700;color:#071033}
    h1{font-size:20px;margin:0}
    p.lead{margin:6px 0 12px;color:var(--muted)}
    .controls{display:flex;gap:8px;flex-wrap:wrap;margin:12px 0}
    button{background:#0b1229;border:1px solid rgba(255,255,255,0.03);padding:8px 12px;border-radius:8px;color:var(--text);cursor:pointer}
    .grid{display:grid;grid-template-columns:1fr 320px;gap:14px;align-items:start}
    .video-wrap{position:relative;background:#061229;border-radius:10px;overflow:hidden;display:grid;place-items:center;min-height:360px}
    video{width:100%;height:auto;display:block}
    canvas.overlay{position:absolute;left:0;top:0;pointer-events:none}
    .panel{padding:12px;border-radius:10px;background:linear-gradient(180deg,rgba(255,255,255,0.02),transparent)}
    .result{font-size:20px;font-weight:700;margin-bottom:6px}
    .meta{color:var(--muted);font-size:13px;margin-bottom:10px}
    .bars{display:flex;flex-direction:column;gap:8px}
    .bar{display:flex;align-items:center;gap:8px}
    .bar .label{flex:1;color:#cbd5e1}
    .bar .score{width:70px;text-align:right;color:var(--muted)}
    .bar .vis{height:10px;border-radius:8px;flex:0 0 auto;margin-left:8px}
    footer{margin-top:12px;color:var(--muted);font-size:13px;text-align:center}
    @media(max-width:880px){.grid{grid-template-columns:1fr} .panel{order:2}}
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <header>
        <div class="logo">FS</div>
        <div>
          <h1>Live Face Shape Recognition</h1>
          <p class="lead">Real-time webcam detection using face landmarks. Heuristic classification into common face shapes.</p>
        </div>
      </header>

      <div class="controls">
        <button id="startBtn">Start Webcam</button>
        <button id="stopBtn" disabled>Stop Webcam</button>
        <button id="pauseBtn" disabled>Pause</button>
        <button id="resumeBtn" disabled>Resume</button>
        <button id="snapBtn" disabled>Snapshot</button>
        <span id="status" style="margin-left:8px;color:var(--muted)">Models loading...</span>
      </div>

      <div class="grid">
        <div class="video-wrap" id="videoWrap">
          <video id="video" autoplay muted playsinline></video>
          <canvas id="overlay" class="overlay"></canvas>
          <div id="placeholder" style="color:var(--muted);padding:18px;text-align:center">Click "Start Webcam" to begin.</div>
        </div>

        <div class="panel">
          <div class="result" id="resultText">No face analyzed</div>
          <div class="meta" id="metaText">Status: idle</div>

          <div id="bars" class="bars" style="display:none"></div>

          <div style="margin-top:12px;color:var(--muted);font-size:13px">
            Tip: face should be facing forward under good lighting for best results. Processing is done locally in your browser.
          </div>
        </div>
      </div>

      <footer>© 2025 — Demo. Models served from public hosts.</footer>
    </div>
  </div>

  <!-- face-api.js -->
  <script src="https://cdn.jsdelivr.net/npm/face-api.js@0.22.2/dist/face-api.min.js"></script>
  <script>
    // MODEL URL - public model hosting for face-api.js
    const MODEL_URL = 'https://justadudewhohacks.github.io/face-api.js/models/';

    // DOM
    const startBtn = document.getElementById('startBtn');
    const stopBtn  = document.getElementById('stopBtn');
    const pauseBtn = document.getElementById('pauseBtn');
    const resumeBtn= document.getElementById('resumeBtn');
    const snapBtn  = document.getElementById('snapBtn');
    const statusEl = document.getElementById('status');
    const video = document.getElementById('video');
    const overlay = document.getElementById('overlay');
    const placeholder = document.getElementById('placeholder');
    const resultText = document.getElementById('resultText');
    const metaText = document.getElementById('metaText');
    const bars = document.getElementById('bars');

    // state
    let stream = null;
    let running = false;
    let paused = false;
    let rafId = null;
    let modelLoaded = false;

    // load tiny models (small & fast)
    statusEl.textContent = 'Loading models...';
    Promise.all([
      faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL),
      faceapi.nets.faceLandmark68TinyNet.loadFromUri(MODEL_URL)
    ]).then(() => {
      modelLoaded = true;
      statusEl.textContent = 'Models loaded. Ready.';
    }).catch(err=>{
      console.error(err);
      statusEl.textContent = 'Model load failed — check network.';
    });

    // helpers
    function resizeCanvas(width, height){
      overlay.width = width;
      overlay.height = height;
      overlay.style.width = width + 'px';
      overlay.style.height = height + 'px';
    }
    function distance(a,b){ return Math.hypot(a.x-b.x, a.y-b.y); }
    function midpoint(a,b){ return {x:(a.x+b.x)/2, y:(a.y+b.y)/2}; }

    async function startWebcam(){
      if(!modelLoaded){ alert('Models not loaded yet — wait a moment.'); return; }
      if(stream) return;
      try{
        stream = await navigator.mediaDevices.getUserMedia({video:{width:640,height:480}, audio:false});
        video.srcObject = stream;
        placeholder.style.display = 'none';
        video.style.display = 'block';
        stopBtn.disabled = false;
        pauseBtn.disabled = false;
        snapBtn.disabled = false;
        startBtn.disabled = true;
        statusEl.textContent = 'Webcam started';
        await video.play();
        resizeCanvas(video.videoWidth || video.clientWidth, video.videoHeight || video.clientHeight);
        running = true;
        paused = false;
        runLoop();
      }catch(err){
        console.error(err);
        alert('Could not access webcam. Allow camera permission and reload.');
        statusEl.textContent = 'Webcam error';
      }
    }

    function stopWebcam(){
      if(stream){
        stream.getTracks().forEach(t=>t.stop());
        stream = null;
      }
      video.pause();
      video.srcObject = null;
      placeholder.style.display = 'block';
      startBtn.disabled = false;
      stopBtn.disabled = true;
      pauseBtn.disabled = true;
      resumeBtn.disabled = true;
      snapBtn.disabled = true;
      running = false;
      paused = false;
      statusEl.textContent = 'Webcam stopped';
      metaText.textContent = 'Status: idle';
      resultText.textContent = 'No face analyzed';
      bars.style.display = 'none';
      clearOverlay();
      if(rafId) cancelAnimationFrame(rafId);
    }

    function pauseWebcam(){
      paused = true;
      pauseBtn.disabled = true;
      resumeBtn.disabled = false;
      statusEl.textContent = 'Paused';
      metaText.textContent = 'Status: paused';
    }
    function resumeWebcam(){
      if(!running) return;
      paused = false;
      pauseBtn.disabled = false;
      resumeBtn.disabled = true;
      statusEl.textContent = 'Resuming...';
      metaText.textContent = 'Status: live';
      runLoop();
    }

    function clearOverlay(){
      const ctx = overlay.getContext('2d');
      ctx.clearRect(0,0,overlay.width, overlay.height);
    }

    // main detection loop
    async function runLoop(){
      if(!running || paused) return;
      if(video.readyState >= 2){
        // detect single face with landmarks
        try{
          const options = new faceapi.TinyFaceDetectorOptions({inputSize: 320, scoreThreshold: 0.5});
          const res = await faceapi.detectSingleFace(video, options).withFaceLandmarks(true);
          if(res){
            drawDetection(res);
            const shapeData = classifyFromLandmarks(res.landmarks);
            displayResult(shapeData);
            metaText.textContent = `faceLen: ${Math.round(shapeData.raw.faceLength)} px • cheek: ${Math.round(shapeData.raw.cheekWidth)} px • jaw: ${Math.round(shapeData.raw.jawWidth)} px`;
          } else {
            clearOverlay();
            resultText.textContent = 'No face detected';
            metaText.textContent = 'Status: no face';
            bars.style.display = 'none';
          }
        }catch(err){
          console.error('Detection error', err);
        }
      }
      rafId = requestAnimationFrame(runLoop);
    }

    function drawDetection(result){
      const ctx = overlay.getContext('2d');
      const vw = video.videoWidth || video.clientWidth;
      const vh = video.videoHeight || video.clientHeight;
      resizeCanvas(vw, vh);
      ctx.clearRect(0,0,overlay.width, overlay.height);

      const box = result.detection.box;
      // draw box
      ctx.strokeStyle = '--accent';
      ctx.lineWidth = Math.max(2, Math.round(vw/200));
      ctx.strokeStyle = '#7c3aed';
      ctx.strokeRect(box.x, box.y, box.width, box.height);

      // draw landmarks
      ctx.fillStyle = '#e6eef8';
      const pts = result.landmarks.positions;
      for(const p of pts){
        ctx.beginPath();
        ctx.arc(p.x, p.y, Math.max(1, Math.round(vw/250)), 0, Math.PI*2);
        ctx.fill();
      }

      // draw chin-jaw polyline for visual
      ctx.beginPath();
      const jaw = result.landmarks.getJawOutline();
      ctx.moveTo(jaw[0].x, jaw[0].y);
      for(let i=1;i<jaw.length;i++) ctx.lineTo(jaw[i].x, jaw[i].y);
      ctx.strokeStyle = 'rgba(124,58,237,0.9)';
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    // classification heuristics (improved)
    function classifyFromLandmarks(landmarks){
      const pos = landmarks.positions;
      // key points
      const chin = pos[8];
      const noseTop = pos[27]; // between eyes
      const leftCheek = pos[1];
      const rightCheek = pos[15];
      const leftJaw = pos[4];
      const rightJaw = pos[12];
      const leftBrow = pos[17];
      const rightBrow = pos[26];

      // measures
      const faceLength = distance(chin, midpoint(pos[27], pos[28] || pos[27])); // chin to between eyes
      const cheekWidth = distance(leftCheek, rightCheek);
      const jawWidth = distance(leftJaw, rightJaw);
      const foreheadWidth = distance(leftBrow, rightBrow);

      // average width
      const avgWidth = (cheekWidth + jawWidth + foreheadWidth) / 3 || 1;
      const cheekRel = cheekWidth / avgWidth;
      const jawRel = jawWidth / avgWidth;
      const foreRel = foreheadWidth / avgWidth;

      // ratio vertical/horizontal
      const vOverH = faceLength / avgWidth;

      // scores
      const scores = {Oval:0, Round:0, Square:0, Heart:0, Diamond:0, Oblong:0};

      // rules (heuristic)
      // Round: widths similar, low v/h
      if(Math.abs(cheekRel - jawRel) < 0.12 && Math.abs(cheekRel - foreRel) < 0.12 && vOverH < 1.05) scores.Round += 2.0;
      // Oval: v/h moderate and jaw slightly narrower
      if(vOverH >= 1.05 && vOverH < 1.35 && jawRel < cheekRel + 0.09) scores.Oval += 2.0;
      // Oblong: vertical much larger
      if(vOverH >= 1.35) scores.Oblong += 2.2;
      // Square: widths similar and v/h close to 1.0
      if(Math.abs(cheekRel - jawRel) < 0.10 && Math.abs(jawRel - foreRel) < 0.12 && Math.abs(vOverH - 1.0) < 0.15) scores.Square += 2.0;
      // Heart: forehead wider, jaw narrower
      if(foreRel > cheekRel + 0.12 && jawRel < cheekRel - 0.12) scores.Heart += 2.0;
      // Diamond: cheekbones largest
      if(cheekRel > foreRel + 0.12 && cheekRel > jawRel + 0.12 && Math.abs(vOverH - 1.05) < 0.25) scores.Diamond += 2.0;

      // soft nudges
      if(jawRel > 1.15 && vOverH < 1.15) scores.Square += 0.8;
      if(cheekRel > jawRel + 0.12) { scores.Diamond += 0.6; scores.Oval += 0.4; }
      if(foreRel > cheekRel + 0.12) scores.Heart += 0.6;

      // normalize to confidences %
      let total = 0;
      for(const k in scores) total += Math.max(0, scores[k]);
      if(total === 0) total = 1;
      const confidences = {};
      for(const k in scores) confidences[k] = Math.round((Math.max(0,scores[k]) / total) * 1000) / 10; // one decimal

      return {
        raw: {faceLength, cheekWidth, jawWidth, foreheadWidth, vOverH},
        scores, confidences
      };
    }

    function displayResult(data){
      const conf = data.confidences;
      // choose top
      let topName = null, topVal = -1;
      for(const k in conf) if(conf[k] > topVal){ topName = k; topVal = conf[k]; }
      resultText.textContent = `${topName} — ${topVal}% (approximate)`;
      bars.style.display = 'block';
      bars.innerHTML = '';
      // sort keys by value
      const keys = Object.keys(conf).sort((a,b)=>conf[b]-conf[a]);
      for(const k of keys){
        const row = document.createElement('div');
        row.className = 'bar';
        const label = document.createElement('div'); label.className='label'; label.textContent = k;
        const score = document.createElement('div'); score.className='score'; score.textContent = conf[k] + '%';
        const vis = document.createElement('div'); vis.className='vis'; vis.style.width = Math.max(10, conf[k]*2) + 'px';
        vis.style.background = (k===topName) ? '#7c3aed' : '#334155';
        row.appendChild(label);
        row.appendChild(vis);
        row.appendChild(score);
        bars.appendChild(row);
      }
    }

    // snapshot (download)
    function snapshot(){
      // draw current video frame + overlay to canvas
      const w = overlay.width, h = overlay.height;
      const out = document.createElement('canvas');
      out.width = w; out.height = h;
      const ctx = out.getContext('2d');
      // draw video frame scaled
      ctx.drawImage(video, 0, 0, w, h);
      // draw overlay on top
      ctx.drawImage(overlay, 0, 0, w, h);
      const data = out.toDataURL('image/png');
      const a = document.createElement('a');
      a.href = data; a.download = 'face_snapshot.png';
      document.body.appendChild(a); a.click(); a.remove();
    }

    // wire events
    startBtn.addEventListener('click', startWebcam);
    stopBtn.addEventListener('click', stopWebcam);
    pauseBtn.addEventListener('click', ()=>{
      pauseWebcam();
      pauseBtn.disabled = true;
      resumeBtn.disabled = false;
    });
    resumeBtn.addEventListener('click', ()=>{
      resumeWebcam();
      pauseBtn.disabled = false;
      resumeBtn.disabled = true;
    });
    snapBtn.addEventListener('click', snapshot);

    // cleanup on page unload
    window.addEventListener('beforeunload', ()=>{ if(stream) stream.getTracks().forEach(t=>t.stop()); });

  </script>
</body>
</html>
