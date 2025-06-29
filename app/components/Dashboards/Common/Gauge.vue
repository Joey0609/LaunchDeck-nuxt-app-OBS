<script setup lang="ts">
const props = defineProps({
  label: { type: String, required: true },
  value: { type: Number, default: 75 },
  unit: { type: String, required: true },
  maxValue: { type: Number, default: 100 },
  fractionDigits: { type: Number, default: 0 },
})

const GAUGE_RADIUS = 70
const STROKE_WIDTH = 4
const GAUGE_VISUAL_START_ANGLE = 255
const TOTAL_GAUGE_SWEEP_ANGLE = 210
const CRITICAL_SWEEP_ANGLE = 180
const BACKGROUND_CIRCLE_PADDING = 8
const TICK_MARK_LENGTH = 6
const TICK_STROKE_WIDTH = 2.5

const progressValue = ref(props.value)
watch(() => props.value, (newValue) => {
  progressValue.value = Math.max(0, Math.min(props.maxValue, newValue))
})

const backgroundCircleRadius = computed(() => GAUGE_RADIUS + BACKGROUND_CIRCLE_PADDING)
const effectiveOuterRadius = computed(() => Math.max(backgroundCircleRadius.value, GAUGE_RADIUS + STROKE_WIDTH / 2))
const viewBoxSize = computed(() => effectiveOuterRadius.value * 2)
const cx = computed(() => effectiveOuterRadius.value)
const cy = computed(() => effectiveOuterRadius.value)
const svgSize = computed(() => viewBoxSize.value)

function polarToCartesian(centerX: number, centerY: number, radius: number, angleInDegrees: number) {
  const angleInRadians = (angleInDegrees - 90) * Math.PI / 180.0
  return {
    x: centerX + (radius * Math.cos(angleInRadians)),
    y: centerY + (radius * Math.sin(angleInRadians)),
  }
}

function describeArc(x: number, y: number, radius: number, startAngleDeg: number, endAngleDeg: number): string {
  if (Math.abs(endAngleDeg - startAngleDeg) >= 360)
    endAngleDeg = startAngleDeg + 359.99
  if (Math.abs(endAngleDeg - startAngleDeg) < 0.01)
    return ''

  const startPoint = polarToCartesian(x, y, radius, startAngleDeg)
  const endPoint = polarToCartesian(x, y, radius, endAngleDeg)
  const arcSweepDegrees = endAngleDeg - startAngleDeg
  const largeArcFlag = Math.abs(arcSweepDegrees) <= 180 ? '0' : '1'
  const sweepFlag = arcSweepDegrees > 0 ? '1' : '0'

  return `M ${startPoint.x} ${startPoint.y} A ${radius} ${radius} 0 ${largeArcFlag} ${sweepFlag} ${endPoint.x} ${endPoint.y}`
}

const backgroundArcGrayPath = computed(() => describeArc(cx.value, cy.value, GAUGE_RADIUS, GAUGE_VISUAL_START_ANGLE, GAUGE_VISUAL_START_ANGLE + CRITICAL_SWEEP_ANGLE))
const backgroundArcDarkRedPath = computed(() => describeArc(cx.value, cy.value, GAUGE_RADIUS, GAUGE_VISUAL_START_ANGLE + CRITICAL_SWEEP_ANGLE, GAUGE_VISUAL_START_ANGLE + TOTAL_GAUGE_SWEEP_ANGLE))

const currentProgressRatio = computed(() => props.maxValue === 0 ? 0 : Math.max(0, Math.min(props.maxValue, progressValue.value)) / props.maxValue)
const currentProgressAngle = computed(() => currentProgressRatio.value * TOTAL_GAUGE_SWEEP_ANGLE)
const whitePartSweep = computed(() => Math.min(currentProgressAngle.value, CRITICAL_SWEEP_ANGLE))
const redPartSweep = computed(() => currentProgressAngle.value <= CRITICAL_SWEEP_ANGLE ? 0 : currentProgressAngle.value - CRITICAL_SWEEP_ANGLE)
const progressArcWhitePath = computed(() => whitePartSweep.value > 0.01 ? describeArc(cx.value, cy.value, GAUGE_RADIUS, GAUGE_VISUAL_START_ANGLE, GAUGE_VISUAL_START_ANGLE + whitePartSweep.value) : '')
const progressArcRedPath = computed(() => redPartSweep.value > 0.01 ? describeArc(cx.value, cy.value, GAUGE_RADIUS, GAUGE_VISUAL_START_ANGLE + CRITICAL_SWEEP_ANGLE, GAUGE_VISUAL_START_ANGLE + CRITICAL_SWEEP_ANGLE + redPartSweep.value) : '')

const gaugeEndAngle = computed(() => GAUGE_VISUAL_START_ANGLE + TOTAL_GAUGE_SWEEP_ANGLE)
const startTickOuterPoint = computed(() => polarToCartesian(cx.value - STROKE_WIDTH / 2, cy.value, GAUGE_RADIUS, GAUGE_VISUAL_START_ANGLE))
const startTickInnerPoint = computed(() => polarToCartesian(cx.value, cy.value, GAUGE_RADIUS - TICK_MARK_LENGTH, GAUGE_VISUAL_START_ANGLE))
const endTickOuterPoint = computed(() => polarToCartesian(cx.value + STROKE_WIDTH / 2, cy.value, GAUGE_RADIUS, gaugeEndAngle.value))
const endTickInnerPoint = computed(() => polarToCartesian(cx.value, cy.value, GAUGE_RADIUS - TICK_MARK_LENGTH, gaugeEndAngle.value))

const showProgressStartTick = computed(() => progressValue.value > 0 && whitePartSweep.value > 0.01)
const showProgressEndTick = computed(() => props.maxValue !== 0 && progressValue.value >= props.maxValue)
</script>

<template>
  <div class="flex flex-col items-center">
    <svg :width="svgSize" :height="svgSize" :viewBox="`0 0 ${viewBoxSize} ${viewBoxSize}`">
      <defs>
        <linearGradient id="backgroundCircleGradient" x1="0%" y1="0%" x2="0%" y2="100%">
          <stop offset="0%" style="stop-color:rgba(20,20,20,0.6); stop-opacity:1" />
          <stop offset="100%" style="stop-color:rgba(0,0,0,0); stop-opacity:1" />
        </linearGradient>
      </defs>
      <circle :cx="cx" :cy="cy" :r="backgroundCircleRadius" fill="url(#backgroundCircleGradient)" stroke="none" />
      <path :d="backgroundArcGrayPath" class="stroke-stone-500" fill="none" :stroke-width="STROKE_WIDTH" stroke-linecap="butt" />
      <path :d="backgroundArcDarkRedPath" class="stroke-red-900" fill="none" :stroke-width="STROKE_WIDTH" stroke-linecap="butt" />
      <line :x1="startTickInnerPoint.x" :y1="startTickInnerPoint.y" :x2="startTickOuterPoint.x" :y2="startTickOuterPoint.y" class="stroke-stone-500" :stroke-width="TICK_STROKE_WIDTH" stroke-linecap="butt" />
      <line :x1="endTickInnerPoint.x" :y1="endTickInnerPoint.y" :x2="endTickOuterPoint.x" :y2="endTickOuterPoint.y" class="stroke-red-900" :stroke-width="TICK_STROKE_WIDTH" stroke-linecap="butt" />
      <path v-if="showProgressStartTick" :d="progressArcWhitePath" class="stroke-gray-100" fill="none" :stroke-width="STROKE_WIDTH" stroke-linecap="butt" :style="{ filter: `drop-shadow(0 0 2px rgba(0,0,0,0.3))` }" />
      <path v-if="redPartSweep > 0.01" :d="progressArcRedPath" class="stroke-red-500" fill="none" :stroke-width="STROKE_WIDTH" stroke-linecap="butt" :style="{ filter: `drop-shadow(0 0 3px rgb(239 68 68 / 0.7))` }" />
      <line v-if="showProgressStartTick" :x1="startTickInnerPoint.x" :y1="startTickInnerPoint.y" :x2="startTickOuterPoint.x" :y2="startTickOuterPoint.y" class="stroke-gray-100" :stroke-width="TICK_STROKE_WIDTH" stroke-linecap="butt" :style="{ filter: `drop-shadow(0 0 2px rgba(0,0,0,0.3))` }" />
      <line v-if="showProgressEndTick" :x1="endTickInnerPoint.x" :y1="endTickInnerPoint.y" :x2="endTickOuterPoint.x" :y2="endTickOuterPoint.y" class="stroke-red-500" :stroke-width="TICK_STROKE_WIDTH" stroke-linecap="butt" :style="{ filter: `drop-shadow(0 0 3px rgb(239 68 68 / 0.7))` }" />
      <text :x="cx" :y="cy" text-anchor="middle" dominant-baseline="central" class="select-none fill-current font-saira">
        <tspan :x="cx" dy="-2.7em" class="text-12px text-gray-400 font-400">{{ props.label }}</tspan>
        <tspan :x="cx" :y="cy" class="text-42px text-white tabular-nums">{{ progressValue.toFixed(fractionDigits) }}</tspan>
        <tspan :x="cx" :y="cy" dy="2.7em" class="text-12px text-gray-400 font-400">{{ props.unit }}</tspan>
      </text>
    </svg>
  </div>
</template>
