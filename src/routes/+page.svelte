<script lang="ts">
  import { onMount } from "svelte";
  import { createClient } from "@supabase/supabase-js";
  import facePaths from "../facePaths.json";
  let canvas: HTMLCanvasElement;
  let context: CanvasRenderingContext2D;
  let count = 0;

  const supabaseUrl = import.meta.env.VITE_PUBLIC_SUPABASE_URL;
  const supabaseKey = import.meta.env.VITE_PUBLIC_SUPABASE_KEY;
  const supabase = createClient(supabaseUrl, supabaseKey);

  const pathData = facePaths;

  type PathData = {
    d: Path2D;
    stroke: string;
    strokeWidth?: number;
    strokeLinecap: CanvasLineCap;
  };
  type PathsObject = {
    normal: PathData[];
    drag: PathData[];
  };
  let facePath: PathsObject;

  // 초기 변수
  const initValue = [0, 0];
  let [windowWidth, windowHeight] = initValue;
  let [amplitudeX, amplitudeY] = initValue; // 진폭
  let [offsetX, offsetY] = initValue;
  let [centerX, centerY] = initValue;
  let [originObjectX, originObjectY] = initValue;
  // 물체의 위치
  let [objectX, objectY] = [originObjectX, originObjectY];
  const [faceTranslateX, faceTranslateY] = [100, 100];
  let omega = 30; // 각진동수 (탄성) (rad/s)
  let damping = 10; // 값 감쇠 계수
  let time = 0;
  let dragging = false;
  const touchableRange = 80;

  const update = () => {
    [windowWidth, windowHeight] = [
      window.innerWidth - 16,
      window.innerHeight - 300,
    ];
    [centerX, centerY] = [windowWidth / 2, windowHeight / 2];
    [originObjectX, originObjectY] = [centerX - 90, centerY + 55];
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
    context.strokeStyle = "black";

    // 얼굴을 살구색으로 칠하기
    for (const face of facePath[dragging ? "drag" : "normal"]) {
      context.strokeStyle = face.stroke;
      context.lineWidth = face.strokeWidth ?? 1;
      context.lineCap = face.strokeLinecap;
      context.stroke(face.d);
    }
    context.restore();

    // 볼 그리기 (부드럽게 휘어지게)
    context.beginPath();
    context.moveTo(centerX - faceTranslateX, centerY);
    context.lineTo(
      centerX - faceTranslateX - (centerX - objectX) / 10,
      centerY - (centerY - objectY) / 5,
    );

    // 볼 제어점과 끝점 설정
    const [controlX1, controlY1] = [objectX - 60, objectY - 50];
    const [controlX2, controlY2] = [objectX - 100, objectY + 80]; // 제어점 X2, Y2

    // 볼 곡선 그리기
    context.bezierCurveTo(
      controlX1,
      controlY1,
      controlX2,
      controlY2,
      centerX - 40,
      centerY + faceTranslateY + 5,
    );

    context.fillStyle = "#ffffff00"; //투명색으로 채우기
    context.fill(); // 내부 채우기

    // 볼 외곽선 그리기
    context.strokeStyle = "black";
    context.stroke();

    // 홍조 넣기
    context.beginPath();
    context.arc(objectX, objectY, touchableRange - 50, 0, 2 * Math.PI);
    context.fillStyle = "pink";
    context.fill();

    time += 0.016; // 60fps
    requestAnimationFrame(update);
  };

  const getCountFromSupabase = async () => {
    count = (
      await supabase.from("balltaguCount").select("*", { count: "exact" })
    ).count as number;
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

  const drag = (event: MouseEvent | TouchEvent) => {
    if (dragging) {
      const rect = canvas.getBoundingClientRect();
      let newObjectX, newObjectY;
      if (event instanceof TouchEvent) {
        const touch = event.touches[0];
        [newObjectX, newObjectY] = [
          touch.clientX - rect.left - offsetX,
          touch.clientY - rect.top - offsetY,
        ];
      } else {
        [newObjectX, newObjectY] = [
          event.clientX - rect.left - offsetX,
          event.clientY - rect.top - offsetY,
        ];
      }
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
    facePath = { normal: [], drag: [] };
    for (const path of pathData.normal) {
      const face = {
        d: new Path2D(path.d),
        stroke: path.stroke,
        strokeWidth: path.strokeWidth,
        strokeLinecap: path.strokeLinecap as CanvasLineCap,
      };
      facePath.normal.push(face);
    }
    for (const path of pathData.drag) {
      const face = {
        d: new Path2D(path.d),
        stroke: path.stroke,
        strokeWidth: path.strokeWidth,
        strokeLinecap: path.strokeLinecap as CanvasLineCap,
      };
      facePath.drag.push(face);
    }
    context = canvas.getContext("2d")!;
    update();
  });
</script>

<h1>Interactive 2D Ball-Tta-gu Simulator</h1>
<canvas
  bind:this={canvas}
  width={windowWidth}
  height={windowHeight}
  on:pointerdown={startDrag}
  on:pointermove={drag}
  on:mouseup={endDrag}
  on:mouseleave={endDrag}
  on:touchmove={drag}
  on:touchend={endDrag}
></canvas>

<div class="controls">
  <h3>총 당긴 횟수: {count}</h3>
  <span
    >Object Position from Center: ({Math.round(objectX - centerX)}px, {Math.round(
      objectY - centerY,
    )}px)</span
  ><br />

  <label
    >Omega (탄성): <input
      type="range"
      min="1"
      max="100"
      step="0.1"
      bind:value={omega}
    /></label
  >
  <span
    ><input
      value={omega}
      on:change={(e) => (omega = +(e.target as HTMLInputElement).value)}
    /> rad/s</span
  ><br />
  <label
    >Damping (힘 감소 속도): <input
      type="range"
      min="1"
      max="30"
      step="0.1"
      bind:value={damping}
    /></label
  >
  <span
    ><input
      value={damping}
      on:change={(e) => (damping = +(e.target as HTMLInputElement).value)}
    /></span
  ><br />
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
