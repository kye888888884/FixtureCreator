<script lang="ts">
  import { onMount } from 'svelte';
  import earcut from 'earcut';

  let canvas: HTMLCanvasElement;
  let imageElement: HTMLImageElement;
  let openCvReady = false;
  
  // ⚙️ 픽스쳐 설정 상태
  let tolerance = 0.2;
  let enableCircles = false; // 원형 처리 ON/OFF
  let circleSensitivity = 30; // 원형 감지 민감도 (낮을수록 억지로라도 원을 찾음)
  
  let triangles: number[][] = [];
  let circles: number[][] = []; // [ [x, y, radius], ... ]
  let polygonCount = 0;
  let zoom = 1;
  let currentFileName = '';

  function ensureTriangleCW(tri: number[]) {
    const x1 = tri[0];
    const y1 = tri[1];
    const x2 = tri[2];
    const y2 = tri[3];
    const x3 = tri[4];
    const y3 = tri[5];
    const cross = (x2 - x1) * (y3 - y1) - (y2 - y1) * (x3 - x1);
    return cross >= 0
      ? [x1, y1, x3, y3, x2, y2]
      : tri;
  }

  function checkOpenCV() {
    if ((window as any).cv && (window as any).cv.Mat) {
      openCvReady = true;
    } else {
      setTimeout(checkOpenCV, 100);
    }
  }

  onMount(() => {
    // 1. Vite의 HMR(새로고침) 발생 시 중복 로드되는 것을 방지
    if ((window as any).cv && (window as any).cv.Mat) {
      openCvReady = true;
      return;
    }

    // 2. 동적으로 script 태그 생성
    const script = document.createElement('script');
    script.src = 'https://docs.opencv.org/4.8.0/opencv.js';
    script.async = true;

    // 3. 스크립트 파일 자체는 다운로드 완료되었을 때
    script.onload = () => {
      // OpenCV.js는 파일이 로드된 후 내부 C++ WebAssembly가 컴파일되는 시간이 또 필요합니다.
      // 따라서 주기적으로 Mat 객체가 생성되었는지 확인(Polling)합니다.
      const checkReady = setInterval(() => {
        if ((window as any).cv && (window as any).cv.Mat) {
          clearInterval(checkReady);
          openCvReady = true; // 화면에 "준비 완료!" 글씨가 뜨게 됨
        }
      }, 100);
    };

    // 4. 로드 실패 시 (네트워크 오류, 광고 차단기능 등)
    script.onerror = () => {
      console.error("OpenCV.js 로드 실패!");
      alert("OpenCV.js를 불러오지 못했습니다. 인터넷 연결이나 브라우저 확장프로그램(광고차단 등)을 확인해주세요.");
    };

    // 5. 문서의 head에 스크립트 추가
    document.head.appendChild(script);
  });

  function handleFileUpload(event: Event) {
    const input = event.target as HTMLInputElement;
    if (input.files && input.files.length > 0) {
      const file = input.files[0];
      currentFileName = file.name;
      const reader = new FileReader();
      reader.onload = (e) => {
        if (typeof e.target?.result === 'string') imageElement.src = e.target.result;
      };
      reader.readAsDataURL(file);
    }
  }

  function processPhysicsImage() {
    if (!openCvReady || !imageElement.complete || imageElement.naturalWidth === 0) return;

    const cv = (window as any).cv;
    let matsToClean: any[] = []; // 메모리 정리를 위한 배열
    
    const tempCanvas = document.createElement('canvas');
    tempCanvas.width = imageElement.naturalWidth;
    tempCanvas.height = imageElement.naturalHeight;
    const tempCtx = tempCanvas.getContext('2d');
    if (!tempCtx) return;
    
    tempCtx.drawImage(imageElement, 0, 0);
    const imgData = tempCtx.getImageData(0, 0, tempCanvas.width, tempCanvas.height);
    
    // 추출한 픽셀 데이터를 OpenCV Mat 객체로 안전하게 변환
    let src = cv.matFromImageData(imgData);
    matsToClean.push(src);
    
    let rgbaPlanes = new cv.MatVector();
    cv.split(src, rgbaPlanes);
    matsToClean.push(rgbaPlanes);
    
    let alphaChannel = rgbaPlanes.get(3);
    matsToClean.push(alphaChannel);
    
    let thresh = new cv.Mat();
    cv.threshold(alphaChannel, thresh, 1, 255, cv.THRESH_BINARY);
    matsToClean.push(thresh);

    triangles = [];
    circles = [];
    polygonCount = 0;

    // 🟢 [추가된 기능] 원형 픽스쳐 감지 및 마스크 분리
    if (enableCircles) {
      let blurMat = new cv.Mat();
      // 노이즈를 줄여 원을 더 잘 찾도록 블러 처리
      cv.GaussianBlur(alphaChannel, blurMat, new cv.Size(5, 5), 0);
      matsToClean.push(blurMat);

      let circlesMat = new cv.Mat();
      matsToClean.push(circlesMat);

      // Hough 변환으로 원 찾기
      // 파라미터: 이미지, 방식, 해상도비율, 원 중심간 최소거리, param1, param2(민감도), 최소반지름, 최대반지름
      const minDist = 20; 
      cv.HoughCircles(blurMat, circlesMat, cv.HOUGH_GRADIENT, 1, minDist, 50, circleSensitivity, 5, 0);

      if (circlesMat.cols > 0) {
        for (let i = 0; i < circlesMat.cols; ++i) {
          let x = Math.round(circlesMat.data32F[i * 3]);
          let y = Math.round(circlesMat.data32F[i * 3 + 1]);
          let radius = Math.round(circlesMat.data32F[i * 3 + 2]);
          
          circles.push([x, y, radius]);

          // 🔥 중요: 다각형 계산에서 중복되지 않도록 찾은 원 부분을 마스크(thresh)에서 검은색으로 칠해버림
          cv.circle(thresh, new cv.Point(x, y), radius + 1, new cv.Scalar(0), -1);
        }
      }
    }

    // 🔺 [기존 기능] 남은 영역을 다각형(삼각형)으로 처리
    let contours = new cv.MatVector();
    let hierarchy = new cv.Mat();
    cv.findContours(thresh, contours, hierarchy, cv.RETR_EXTERNAL, cv.CHAIN_APPROX_SIMPLE);
    matsToClean.push(contours, hierarchy);

    if (contours.size() > 0) {
      for (let i = 0; i < contours.size(); i++) {
        let contour = contours.get(i);
        
        // 너무 작은 찌꺼기 픽셀은 무시
        if (cv.contourArea(contour) < 10) continue;

        let approxPolygon = new cv.Mat();
        let epsilon = (tolerance * cv.arcLength(contour, true)) / 100.0;
        cv.approxPolyDP(contour, approxPolygon, epsilon, true);
        matsToClean.push(approxPolygon);

        let points: number[] = [];
        for (let j = 0; j < approxPolygon.data32S.length; j++) {
          points.push(approxPolygon.data32S[j]);
        }

        let triangleIndices = earcut(points);
        for (let j = 0; j < triangleIndices.length; j += 3) {
          let p1 = triangleIndices[j] * 2;
          let p2 = triangleIndices[j + 1] * 2;
          let p3 = triangleIndices[j + 2] * 2;

          triangles.push(ensureTriangleCW([
            points[p1], points[p1 + 1],
            points[p2], points[p2 + 1],
            points[p3], points[p3 + 1]
          ]));
        }
      }
    }

    polygonCount = triangles.length;

    // 메모리 정리
    for (let mat of matsToClean) {
      if (mat && !mat.isDeleted()) mat.delete();
    }

    drawResult();
  }

  function drawResult() {
    if (!canvas || !imageElement) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    canvas.width = imageElement.naturalWidth * zoom;
    canvas.height = imageElement.naturalHeight * zoom;

    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    ctx.setTransform(zoom, 0, 0, zoom, 0, 0);
    ctx.drawImage(imageElement, 0, 0);

    // 1. 삼각형 그리기 (초록색)
    ctx.strokeStyle = 'rgba(0, 255, 128, 1)';
    ctx.fillStyle = 'rgba(0, 255, 128, 0.3)';
    ctx.lineWidth = 1;
    for (const tri of triangles) {
      ctx.beginPath();
      ctx.moveTo(tri[0], tri[1]);
      ctx.lineTo(tri[2], tri[3]);
      ctx.lineTo(tri[4], tri[5]);
      ctx.closePath();
      ctx.fill();
      ctx.stroke();
    }

    // 2. 원 그리기 (주황색)
    ctx.strokeStyle = 'rgba(255, 165, 0, 1)';
    ctx.fillStyle = 'rgba(255, 165, 0, 0.5)';
    ctx.lineWidth = 2;
    for (const [x, y, r] of circles) {
      ctx.beginPath();
      ctx.arc(x, y, r, 0, 2 * Math.PI);
      ctx.fill();
      ctx.stroke();
    }
  }

  function downloadJSON() {
    if ((triangles.length === 0 && circles.length === 0) || !currentFileName) return;
    
    const baseName = currentFileName.replace(/\.[^/.]+$/, '');

    // JSON 구조에 circles 배열 추가
    const resultData = { 
      triangles: triangles,
      circles: circles
    };
    const blob = new Blob([JSON.stringify(resultData, null, 4)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${baseName}_fixture.json`;
    a.click();
    URL.revokeObjectURL(url);
  }
</script>

<main>
  <h2>GameMaker Fixture Generator</h2>
  
  <div class="controls panel">
    <input type="file" accept="image/png" on:change={handleFileUpload} />
    
    <label class="toggle-label">
      <input type="checkbox" bind:checked={enableCircles} on:change={processPhysicsImage} />
      <strong>Enable Circle Detection</strong>
    </label>

    {#if enableCircles}
      <label>
        Circle Strictness ({circleSensitivity}):
        <input type="range" min="10" max="80" bind:value={circleSensitivity} on:change={processPhysicsImage} />
      </label>
    {/if}

    <label style="display: flex; align-items: center; gap: 0.5rem;">
      Polygon Tolerance:
      <input type="number" step="0.1" bind:value={tolerance} on:change={processPhysicsImage} style="width: 60px;" />
      <span class="poly-count">Polygons: {polygonCount}</span>
      <div class="zoom-controls">
        <button type="button" class="zoom-button" on:click={() => { zoom = Math.max(0.25, zoom - 0.25); drawResult(); }} title="Zoom out">-</button>
        <button type="button" class="zoom-button" on:click={() => { zoom = 1; drawResult(); }} title="Reset zoom">=</button>
        <button type="button" class="zoom-button" on:click={() => { zoom = Math.min(4, zoom + 0.25); drawResult(); }} title="Zoom in">+</button>
      </div>
    </label>
    
    <button on:click={downloadJSON} disabled={triangles.length === 0 && circles.length === 0}>
      Save JSON
    </button>
  </div>

  <div class="legend panel">
    <div class="legend-item">
      <span class="legend-marker triangle"></span>
      <span>Triangle</span>
    </div>
    <div class="legend-item">
      <span class="legend-marker circle"></span>
      <span>Circle</span>
    </div>
  </div>

  <div class="status">
    {#if !openCvReady}
      <p style="color: orange;">Loading OpenCV.js...</p>
    {/if}
  </div>

  <img bind:this={imageElement} alt="source" on:load={processPhysicsImage} style="display: none;" />

  <div class="canvas-wrapper">
    <div class="canvas-container">
      <canvas bind:this={canvas}></canvas>
    </div>
  </div>
</main>

<style>
  main { padding: 2rem; color: white; background: #1e1e1e; min-height: 100vh; font-family: sans-serif; }
  .panel { background: #333; padding: 1rem; border-radius: 8px; margin-bottom: 1rem; }
  .controls { display: flex; flex-wrap: wrap; gap: 1.5rem; align-items: center; }
  .toggle-label { cursor: pointer; display: flex; align-items: center; gap: 0.5rem; color: #ffa500; }
  .canvas-wrapper { width: 100%; display: flex; justify-content: center; overflow-x: auto; }
  .canvas-container { background: #2a2a2a; padding: 1rem; border-radius: 8px; display: inline-block; min-width: fit-content; }
  .legend { display: flex; justify-content: flex-end; gap: 1rem; align-items: center; margin: 1rem 0; padding: 1rem; }
  .legend-item { display: flex; align-items: center; gap: 0.5rem; color: #ddd; }
  .legend-marker { display: inline-flex; align-items: center; justify-content: center; }
  .legend-marker.triangle {
    width: 18px;
    height: 16px;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 18 16'%3E%3Cpolygon points='9,1 1,15 17,15' fill='none' stroke='%2300ff80' stroke-width='2'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-size: contain;
  }
  .legend-marker.circle {
    width: 14px;
    height: 14px;
    border: 2px solid rgba(255,165,0,1);
    border-radius: 50%;
    background: transparent;
  }
  .poly-count { color: #00ff80; font-weight: bold; }
  .zoom-controls { display: flex; gap: 0.25rem; align-items: center; }
  .zoom-button { padding: 0.2rem 0.5rem; border: 1px solid #00ff80; border-radius: 4px; background: #1e1e1e; color: #00ff80; cursor: pointer; }
  .zoom-button:hover { background: #00ff80; color: #000; }
  canvas { background: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAoAAAAKCAYAAACNMs+9AAAAOklEQVQYV2NkYGAwYcCH//9jQBMgWYSQBMkSJDZITJCMgSQPwiTYEaQIkE20MQwyhF51EAeQ9J20AwC6vB/xS748bAAAAABJRU5ErkJggg==') repeat; }
  button { padding: 0.5rem 1rem; background: #00ff80; color: #000; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; }
  button:disabled { background: #555; cursor: not-allowed; color: #888; }
</style>