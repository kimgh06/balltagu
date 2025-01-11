<script lang="ts">
  import { onMount } from "svelte";
  import { createClient } from "@supabase/supabase-js";
  let canvas: HTMLCanvasElement;
  let context: CanvasRenderingContext2D;
  let count = 0;

  const supabaseUrl = import.meta.env.VITE_PUBLIC_SUPABASE_URL;
  const supabaseKey = import.meta.env.VITE_PUBLIC_SUPABASE_KEY;
  const supabase = createClient(supabaseUrl, supabaseKey);
  console.log(supabaseUrl, supabaseKey);

  const pathData =
    "M0.999969 99C5 8.49999 154.5 -65.5 201 99C203 133.667 239 231 57 206";

  // 초기 변수
  const initValue = [0, 0];
  let [windowWidth, windowHeight] = initValue;
  let [amplitudeX, amplitudeY] = initValue; // 진폭
  let [offsetX, offsetY] = initValue;
  let [centerX, centerY] = initValue;
  let [originObjectX, originObjectY] = initValue;
  // 물체의 위치
  let [objectX, objectY] = [originObjectX, originObjectY];
  const [faceTranslateX, faceTranslateY] = [100, 155];
  let omega = 30; // 각진동수 (탄성) (rad/s)
  let damping = 10; // 값 감쇠 계수
  let time = 0;
  let dragging = false;
  const touchableRange = 80;

  const update = () => {
    const facePath = new Path2D(pathData);
    [windowWidth, windowHeight] = [
      window.innerWidth - 16,
      window.innerHeight - 300,
    ];
    [centerX, centerY] = [windowWidth / 2, windowHeight / 2];
    [originObjectX, originObjectY] = [centerX - 130, centerY + 70];
    if (!dragging) {
      // 진폭이 시간이 지남에 따라 감소 (감쇠 계수 적용)
      const dampingExp = Math.exp(-damping * time);
      const dampedAmplitudeX = amplitudeX * dampingExp;
      const dampedAmplitudeY = amplitudeY * dampingExp;
      // 2D 운동 계산
      const vm = Math.cos(omega * time); // vivrational Motion
      const [x, y] = [dampedAmplitudeX * vm, dampedAmplitudeY * vm];
      // 물체의 위치
      [objectX, objectY] = [originObjectX + x, originObjectY + y];
    }
    context.clearRect(0, 0, canvas.width ?? 0, canvas.height ?? 0);
    context.lineWidth = 2;

    // 가로 세로 점선 그리기
    context.beginPath();
    context.moveTo(centerX, 0);
    context.lineTo(centerX, windowHeight);
    context.strokeStyle = "gray";
    context.stroke();
    context.beginPath();
    context.moveTo(0, centerY);
    context.lineTo(windowWidth, centerY);
    context.strokeStyle = "gray";
    context.stroke();

    // 거리 그리기
    context.beginPath();
    context.moveTo(originObjectX, originObjectY);
    context.lineTo(objectX, objectY);
    context.strokeStyle = "blue";
    context.stroke();

    // 얼굴 그리기
    context.save();
    context.beginPath();
    context.translate(centerX - faceTranslateX, centerY - faceTranslateY);
    context.scale(1.5, 1.5);
    context.strokeStyle = "black";

    // 얼굴을 살구색으로 칠하기
    context.fillStyle = "peachpuff";
    context.fill(facePath);
    context.stroke(facePath);
    context.restore();

    // 볼 그리기 (부드럽게 휘어지게)
    context.beginPath();
    context.moveTo(centerX - faceTranslateX, centerY - 10);

    // 볼 제어점과 끝점 설정
    const [controlX1, controlY1] = [objectX - 100, objectY - 50];
    const [controlX2, controlY2] = [objectX - 100, objectY + 120]; // 제어점 X2, Y2

    // 볼 곡선 그리기
    context.bezierCurveTo(
      controlX1,
      controlY1,
      controlX2,
      controlY2,
      centerX,
      centerY + faceTranslateY,
    );

    context.fillStyle = "peachpuff"; // 살구색
    context.fill(); // 내부 채우기

    // 볼 외곽선 그리기
    context.strokeStyle = "black";
    context.stroke();

    // 홍조 넣기
    context.beginPath();
    context.arc(objectX, objectY, touchableRange - 30, 0, 2 * Math.PI);
    context.fillStyle = "pink";
    context.fill();

    time += 0.016; // 60fps
    requestAnimationFrame(update);
  };

  const getCountFromSupabase = async () => {
    const { data, error } = await supabase.from("balltaguCount").select();
    if (error) {
      console.error("Error fetching data:", error);
    } else {
      count = data.length;
    }
  };

  const sendToSupabase = async () => {
    const { error } = await supabase
      .from("balltaguCount")
      .insert({ time: new Date().toISOString() });

    if (error) {
      console.error("Error fetching data:", error);
    }
  };

  const startDrag = (event: MouseEvent) => {
    const rect = canvas.getBoundingClientRect();
    const mouseX = event.clientX - rect.left;
    const mouseY = event.clientY - rect.top;

    const dist = Math.sqrt((mouseX - objectX) ** 2 + (mouseY - objectY) ** 2);
    if (dist < touchableRange) {
      dragging = true;
      offsetX = mouseX - objectX;
      [offsetX, offsetY] = [mouseX - objectX, mouseY - objectY];
      sendToSupabase();
    }
  };

  const drag = (event: MouseEvent) => {
    if (dragging) {
      const rect = canvas.getBoundingClientRect();
      let [newObjectX, newObjectY] = [
        event.clientX - rect.left - offsetX,
        event.clientY - rect.top - offsetY,
      ];
      if (newObjectX > centerX) newObjectX = centerX;
      if (newObjectY < centerY) newObjectY = centerY;
      [objectX, objectY] = [newObjectX, newObjectY];
    }
  };

  const endDrag = () => {
    dragging = false;
    time = 0; // 초기화
    amplitudeX = objectX - originObjectX;
    amplitudeY = objectY - originObjectY;
    getCountFromSupabase();
  };

  onMount(() => {
    getCountFromSupabase();
    context = canvas.getContext("2d")!;
    update();
  });
</script>

<svelte:head>
  <title>Interactive 2D Ball-Tta-gu Simulator</title>
  <meta
    name="description"
    content="A simple interactive 2D Balltagu simulator with damping using Svelte."
  />
</svelte:head>

<h1>Interactive 2D Ball-Tta-gu Simulator</h1>
<canvas
  bind:this={canvas}
  width={windowWidth}
  height={windowHeight}
  on:pointerdown={startDrag}
  on:pointermove={drag}
  on:pointerup={endDrag}
  on:pointerleave={endDrag}
></canvas>

<div class="controls">
  <h3>총 당긴 횟수: {count}</h3>
  <span
    >Object Position from Center: ({Math.floor(centerX - objectX)}px, {Math.floor(
      centerY - objectY,
    )}px)</span
  ><br />

  <label
    >Omega (탄성): <input
      type="range"
      min="1"
      max="50"
      step="0.1"
      bind:value={omega}
    /></label
  >
  <span>{omega} rad/s</span><br />
  <label
    >Damping (힘 감소 속도): <input
      type="range"
      min="1"
      max="50"
      step="0.1"
      bind:value={damping}
    /></label
  >
  <span>{damping}</span>
</div>

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
