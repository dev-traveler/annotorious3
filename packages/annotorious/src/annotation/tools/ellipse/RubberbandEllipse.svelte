<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte';
  import type { DrawingMode } from '../../../AnnotoriousOpts';
  import { ShapeType, type Ellipse } from '../../../model';
  import type { Transform } from '../..';

  // 이벤트 디스패처 설정 - Ellipse 타입의 create 이벤트를 발생시킴
  const dispatch = createEventDispatcher<{ create: Ellipse }>();

  /** Props **/
  export let addEventListener: (
    type: string,
    fn: EventListener,
    capture?: boolean
  ) => void;
  export let drawingMode: DrawingMode;
  export let transform: Transform;

  // 마우스 이벤트 타이밍 추적을 위한 변수
  let lastPointerDown: number;

  // 시작점 좌표
  let origin: [x: number, y: number] | undefined;

  // 현재 마우스 위치
  let anchor: [number, number] | undefined;

  // 타원의 중심점과 반지름
  let cx: number, cy: number, rx: number, ry: number;

  const onPointerDown = (event: Event) => {
    const evt = event as PointerEvent;
    lastPointerDown = performance.now();

    if (drawingMode === 'drag') {
      origin = transform.elementToImage(evt.offsetX, evt.offsetY);
      anchor = origin;

      cx = origin[0];
      cy = origin[1];
      rx = 1;
      ry = 1;
    }
  };

  const onPointerMove = (event: Event) => {
    const evt = event as PointerEvent;

    if (origin) {
      anchor = transform.elementToImage(evt.offsetX, evt.offsetY);

      // 타원의 중심점과 반지름 계산
      cx = (origin[0] + anchor[0]) / 2;
      cy = (origin[1] + anchor[1]) / 2;
      rx = Math.abs(anchor[0] - origin[0]) / 2;
      ry = Math.abs(anchor[1] - origin[1]) / 2;
    }
  };

  const onPointerUp = (event: Event) => {
    const evt = event as PointerEvent;
    const timeDifference = performance.now() - lastPointerDown;

    if (drawingMode === 'click') {
      if (timeDifference > 300) return;

      if (origin) {
        stopDrawing();
      } else {
        // 그리기 시작
        origin = transform.elementToImage(evt.offsetX, evt.offsetY);
        anchor = origin;

        cx = origin[0];
        cy = origin[1];
        rx = 1;
        ry = 1;
      }
    } else if (origin) {
      // 타원의 면적 계산 (π * rx * ry)
      const area = Math.PI * rx * ry;

      if (timeDifference > 300 || area > 100) {
        evt.stopPropagation();
        stopDrawing();
      } else {
        origin = undefined;
        anchor = undefined;
      }
    }
  };

  const stopDrawing = () => {
    // 최소 면적 요구사항 (타원의 경우 π * rx * ry > 15)
    if (Math.PI * rx * ry > 15) {
      const shape: Ellipse = {
        type: ShapeType.ELLIPSE,
        geometry: {
          cx,
          cy,
          rx,
          ry,
          bounds: {
            minX: cx - rx,
            minY: cy - ry,
            maxX: cx + rx,
            maxY: cy + ry,
          },
        },
      };

      dispatch('create', shape);
    }

    origin = undefined;
    anchor = undefined;
  };

  onMount(() => {
    addEventListener('pointerdown', onPointerDown);
    addEventListener('pointermove', onPointerMove);
    addEventListener('pointerup', onPointerUp, true);
  });
</script>

<g class="a9s-annotation a9s-rubberband">
  {#if origin}
    <ellipse class="a9s-outer" {cx} {cy} {rx} {ry} />

    <ellipse class="a9s-inner" {cx} {cy} {rx} {ry} />
  {/if}
</g>
