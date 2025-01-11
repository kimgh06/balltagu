<svelte:head>
  <title>Interactive 2D Spring Simulator</title>
  <meta name="description" content="A simple interactive 2D spring simulator with damping using Svelte." />
</svelte:head>

<script lang="ts">
  import { onMount } from "svelte";

  let canvas: HTMLCanvasElement;
  let context: CanvasRenderingContext2D;

  // 초기 변수
  let amplitudeX = 100; // x축 초기 진폭 (px)
  let amplitudeY = 50;  // y축 초기 진폭 (px)
  let omega = 10;       // 각진동수 (탄성) (rad/s)
  let damping = 3;   // 감쇠 계수

  let time = 0;
  let dragging = false;
  let [offsetX, offsetY]= [0,0];

  // 물체의 초기 위치
  let objectX = 400, objectY = 300;

  const update = () => {
    if (!dragging) {
      // 진폭이 시간이 지남에 따라 감소 (감쇠 계수 적용)
      const dampedAmplitudeX = amplitudeX * Math.exp(-damping * time);
      const dampedAmplitudeY = amplitudeY * Math.exp(-damping * time);

      // 2D 운동 계산
      const x = dampedAmplitudeX * Math.cos(omega * time);
      const y = dampedAmplitudeY * Math.cos(omega * time);

      objectX = 400 + x;
      objectY = 300 + y;
    }

    // Canvas 그리기
    context.clearRect(0, 0, canvas.width, canvas.height);
    context.beginPath();
    context.moveTo(400, 300);
    context.lineTo(objectX, objectY);
    context.strokeStyle = "blue";
    context.stroke();

    context.beginPath();
    context.arc(objectX, objectY, 10, 0, 2 * Math.PI);
    context.fillStyle = "red";
    context.fill();

    time += 0.016; // 60fps
    requestAnimationFrame(update);
  };

  const startDrag = (event: MouseEvent) => {
    const rect = canvas.getBoundingClientRect();
    const mouseX = event.clientX - rect.left;
    const mouseY = event.clientY - rect.top;

    const dist = Math.sqrt((mouseX - objectX) ** 2 + (mouseY - objectY) ** 2);
    if (dist < 15) {
      dragging = true;
      offsetX = mouseX - objectX;
      offsetY = mouseY - objectY;
    }
  };

  const drag = (event: MouseEvent) => {
    if (dragging) {
      const rect = canvas.getBoundingClientRect();
      objectX = event.clientX - rect.left - offsetX;
      objectY = event.clientY - rect.top - offsetY;
    }
  };

  const endDrag = () => {
    dragging = false;
    const dx = objectX - 400;
    const dy = objectY - 300;
    amplitudeX = dx;
    amplitudeY = dy;
    time = 0; // 초기화
  };

  onMount(() => {
    context = canvas.getContext("2d")!;
    update();
  });
</script>

<style>
  canvas {
    border: 1px solid #ccc;
    background: #f9f9f9;
    display: block;
    margin: auto;
  }
  div.controls {
    text-align: center;
    margin-top: 20px;
  }
  input[type="range"] {
    width: 300px;
  }
</style>

<h1>Interactive 2D Spring Simulator with Damping</h1>
<canvas
  bind:this={canvas}
  width="800"
  height="600"
  on:mousedown={startDrag}
  on:mousemove={drag}
  on:mouseup={endDrag}
  on:mouseleave={endDrag}
></canvas>

<div class="controls">
  <label>Amplitude X: <input type="range" min="10" max="200" bind:value={amplitudeX} /></label>
  <span>{Math.floor(amplitudeX * 100) / 1000}px</span><br />
  <label>Amplitude Y: <input type="range" min="10" max="200" bind:value={amplitudeY} /></label>
  <span>{Math.floor(amplitudeY *100) / 100}px</span><br />
  <label>Omega: <input type="range" min="1" max="20"  bind:value={omega} /></label>
  <span>{omega} rad/s</span><br />
  <label>Damping: <input type="range" min="1" max="5" bind:value={damping} /></label>
  <span>{damping}</span>
</div>
