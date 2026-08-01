<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Animated Flower Bouquet</title>
    <style>
        body {
            background-color: #0c0c14;
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            overflow: hidden;
            font-family: sans-serif;
        }
        canvas {
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            border-radius: 4px;
        }
    </style>
</head>
<body>

<canvas id="canvas" width="800" height="800"></canvas>

<script>
// -----------------------------------------------------------------------------
// Configuration & Master Settings
// -----------------------------------------------------------------------------
const WIDTH = 800;
const HEIGHT = 800;
const FPS = 60;

// Base Colors
const GREEN = "rgb(46, 139, 87)";
const STEM_GREEN = "rgb(34, 112, 63)";
const DARK_GREEN = "rgb(20, 80, 45)";

const PINK = "rgb(235, 87, 138)";
const PINK_BASE = "rgb(175, 40, 90)";
const RED = "rgb(220, 40, 60)";
const RED_BASE = "rgb(140, 15, 30)";
const WHITE = "rgb(245, 245, 250)";
const WHITE_BASE = "rgb(190, 195, 210)";
const YELLOW = "rgb(255, 215, 0)";
const YELLOW_BASE = "rgb(200, 120, 0)";
const ORANGE = "rgb(255, 140, 0)";
const BROWN = "rgb(101, 67, 33)";

// -----------------------------------------------------------------------------
// Optimized Caches & Surfaces
// -----------------------------------------------------------------------------
const PETAL_ALPHA_CACHE = new Map();
const ROTATED_ELLIPSE_CACHE = new Map();

function getCachedPetalParticle(radius, color, alpha) {
    const qRadius = Math.round(radius);
    const qAlpha = Math.floor(alpha / 16) * 16;
    const key = `${qRadius}_${color}_${qAlpha}`;

    if (!PETAL_ALPHA_CACHE.has(key)) {
        const offscreen = document.createElement('canvas');
        offscreen.width = qRadius * 2;
        offscreen.height = qRadius * 2;
        const ctx = offscreen.getContext('2d');
        
        ctx.fillStyle = color;
        ctx.globalAlpha = qAlpha / 255;
        ctx.beginPath();
        ctx.arc(qRadius, qRadius, qRadius, 0, Math.PI * 2);
        ctx.fill();
        
        PETAL_ALPHA_CACHE.set(key, offscreen);
    }
    return PETAL_ALPHA_CACHE.get(key);
}

function getRotatedEllipseSurface(color, width, height, angleDeg) {
    const w = Math.max(2, Math.floor(width));
    const h = Math.max(2, Math.floor(height));
    const qAngle = (Math.round(angleDeg / 2.0) * 2) % 360;
    const key = `${color}_${w}_${h}_${qAngle}`;

    if (!ROTATED_ELLIPSE_CACHE.has(key)) {
        const rad = (qAngle * Math.PI) / 180;
        const sin = Math.abs(Math.sin(rad));
        const cos = Math.abs(Math.cos(rad));
        const bWidth = Math.ceil(w * cos + h * sin);
        const bHeight = Math.ceil(w * sin + h * cos);

        const offscreen = document.createElement('canvas');
        offscreen.width = Math.max(1, bWidth);
        offscreen.height = Math.max(1, bHeight);
        const ctx = offscreen.getContext('2d');

        ctx.translate(offscreen.width / 2, offscreen.height / 2);
        ctx.rotate(rad);
        ctx.fillStyle = color;
        ctx.beginPath();
        ctx.ellipse(0, 0, w / 2, h / 2, 0, 0, Math.PI * 2);
        ctx.fill();

        ROTATED_ELLIPSE_CACHE.set(key, offscreen);
    }
    return ROTATED_ELLIPSE_CACHE.get(key);
}

function createBackgroundSurface() {
    const bg = document.createElement('canvas');
    bg.width = WIDTH;
    bg.height = HEIGHT;
    const ctx = bg.getContext('2d');

    for (let y = 0; y < HEIGHT; y++) {
        const c = 12 + Math.floor(18 * y / HEIGHT);
        ctx.fillStyle = `rgb(${c}, ${c}, ${c + 10})`;
        ctx.fillRect(0, y, WIDTH, 1);
    }
    return bg;
}

function createGlowSurface() {
    const surf = document.createElement('canvas');
    surf.width = 600;
    surf.height = 600;
    const ctx = surf.getContext('2d');
    const center = 300;
    const sigma = 120.0;

    for (let r = 280; r > 0; r -= 4) {
        const alpha = Math.floor(35 * Math.exp(-(r * r) / (sigma * sigma)));
        if (alpha > 0) {
            ctx.fillStyle = `rgba(255, 250, 220, ${alpha / 255})`;
            ctx.beginPath();
            ctx.arc(center, center, r, 0, Math.PI * 2);
            ctx.fill();
        }
    }
    return surf;
}

// -----------------------------------------------------------------------------
// Math & Normalized Bezier Pre-computation
// -----------------------------------------------------------------------------
function getBloomScale(progress) {
    if (progress <= 0) return 0.0;
    const smooth = progress * progress * (3 - 2 * progress);
    const overshoot = Math.sin(smooth * Math.PI) * 0.05;
    return smooth + overshoot;
}

function lerp(a, b, t) {
    return a + (b - a) * t;
}

function evaluateRawBezierPoint(p0, p1, p2, p3, t) {
    const x = Math.pow(1 - t, 3) * p0[0] + 3 * Math.pow(1 - t, 2) * t * p1[0] + 3 * (1 - t) * t * t * p2[0] + Math.pow(t, 3) * p3[0];
    const y = Math.pow(1 - t, 3) * p0[1] + 3 * Math.pow(1 - t, 2) * t * p1[1] + 3 * (1 - t) * t * t * p2[1] + Math.pow(t, 3) * p3[1];
    return [x, y];
}

function precalculateStemBezierLut(stem, segments = 40) {
    const lut = [];
    const { start: p0, ctrl1: p1, ctrl2: p2, end: p3 } = stem;
    for (let i = 0; i <= segments; i++) {
        const t = i / segments;
        const basePt = evaluateRawBezierPoint(p0, p1, p2, p3, t);
        lut.push({
            t: t,
            base_x: basePt[0],
            base_y: basePt[1],
            wind_weight: t * t
        });
    }
    return lut;
}

// -----------------------------------------------------------------------------
// Fast Plant Rendering
// -----------------------------------------------------------------------------
function blitRotatedEllipse(ctx, color, centerX, centerY, width, height, angleRad) {
    if (width <= 0 || height <= 0) return;
    const angleDeg = (angleRad * 180) / Math.PI;
    const rotSurf = getRotatedEllipseSurface(color, width, height, angleDeg);
    ctx.drawImage(rotSurf, Math.floor(centerX - rotSurf.width / 2), Math.floor(centerY - rotSurf.height / 2));
}

function drawTulip(ctx, x, y, size, progress, color, baseColor, rotOffset, breathScale, cosR, sinR) {
    const scale = getBloomScale(progress) * breathScale;
    const s = size * scale;
    if (s <= 0) return;

    const unfold = (1.0 - progress) * 0.25;
    const left = [[x - s * (0.45 + unfold), y], [x - s * (0.20 + unfold), y - s], [x, y]];
    const center = [[x - s * 0.15, y], [x, y - s * 1.25], [x + s * 0.15, y]];
    const right = [[x, y], [x + s * (0.20 + unfold), y - s], [x + s * (0.45 + unfold), y]];

    function rotatePts(pts) {
        return pts.map(([px, py]) => {
            const dx = px - x, dy = py - y;
            const rx = dx * cosR - dy * sinR;
            const ry = dx * sinR + dy * cosR;
            return [x + rx, y + ry];
        });
    }

    const drawPoly = (pts, fillStyle, strokeStyle = null, lineWidth = 1) => {
        ctx.beginPath();
        ctx.moveTo(pts[0][0], pts[0][1]);
        for (let i = 1; i < pts.length; i++) ctx.lineTo(pts[i][0], pts[i][1]);
        ctx.closePath();
        if (fillStyle) {
            ctx.fillStyle = fillStyle;
            ctx.fill();
        }
        if (strokeStyle) {
            ctx.strokeStyle = strokeStyle;
            ctx.lineWidth = lineWidth;
            ctx.stroke();
        }
    };

    const leftR = rotatePts(left), centerR = rotatePts(center), rightR = rotatePts(right);

    drawPoly(leftR, baseColor);
    drawPoly(centerR, baseColor);
    drawPoly(rightR, baseColor);

    const leftTip = [[x - s * (0.35 + unfold), y - s * 0.4], [x - s * (0.20 + unfold), y - s], [x, y - s * 0.3]];
    const centerTip = [[x - s * 0.10, y - s * 0.4], [x, y - s * 1.25], [x + s * 0.10, y - s * 0.4]];
    const rightTip = [[x, y - s * 0.3], [x + s * (0.20 + unfold), y - s], [x + s * (0.35 + unfold), y - s * 0.4]];

    drawPoly(rotatePts(leftTip), color);
    drawPoly(rotatePts(centerTip), color);
    drawPoly(rotatePts(rightTip), color);

    drawPoly(leftR, null, "rgb(60, 0, 20)", 2);
    drawPoly(centerR, null, "rgb(60, 0, 20)", 2);
    drawPoly(rightR, null, "rgb(60, 0, 20)", 2);
}

function drawDaisy(ctx, x, y, size, progress, petalColor = WHITE, baseColor = WHITE_BASE, centerColor = YELLOW, rotOffset = 0, breathScale = 1.0) {
    const scale = getBloomScale(progress) * breathScale;
    const radius = (size / 2) * scale;
    if (radius <= 0) return;

    const numPetals = 12;
    const petalW = radius * 0.28;
    const petalH = radius * 0.85;

    for (let i = 0; i < numPetals; i++) {
        const petalThreshold = i / numPetals;
        if (progress < petalThreshold * 0.8) continue;

        const petalP = Math.min(1.0, (progress - petalThreshold * 0.8) / 0.2);
        const pLen = petalH * getBloomScale(petalP);
        const angle = (2 * Math.PI / numPetals) * i + (rotOffset * Math.PI / 180);

        const px = x + Math.cos(angle) * (radius * 0.45);
        const py = y + Math.sin(angle) * (radius * 0.45);
        blitRotatedEllipse(ctx, baseColor, px, py, petalW, pLen, angle + Math.PI / 2);

        const pxTip = x + Math.cos(angle) * (radius * 0.55);
        const pyTip = y + Math.sin(angle) * (radius * 0.55);
        blitRotatedEllipse(ctx, petalColor, pxTip, pyTip, petalW * 0.85, pLen * 0.7, angle + Math.PI / 2);
    }

    ctx.fillStyle = ORANGE;
    ctx.beginPath();
    ctx.arc(x, y, radius * 0.35 * scale, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = centerColor;
    ctx.beginPath();
    ctx.arc(x, y, radius * 0.28 * scale, 0, Math.PI * 2);
    ctx.fill();
}

function drawSunflower(ctx, x, y, size, progress, rotOffset = 0, breathScale = 1.0) {
    const scale = getBloomScale(progress) * breathScale;
    const radius = (size / 2) * scale;
    if (radius <= 0) return;

    const numPetals = 18;
    const petalW = radius * 0.3;
    const petalH = radius * 0.85;
    const centerR = radius * (0.2 + 0.32 * progress);

    for (let i = 0; i < numPetals; i++) {
        const angle = (2 * Math.PI / numPetals) * i + (rotOffset * Math.PI / 180);
        const px = x + Math.cos(angle) * (centerR * 0.85);
        const py = y + Math.sin(angle) * (centerR * 0.85);
        blitRotatedEllipse(ctx, YELLOW_BASE, px, py, petalW, petalH, angle + Math.PI / 2);

        const pxTip = x + Math.cos(angle) * (centerR * 1.05);
        const pyTip = y + Math.sin(angle) * (centerR * 1.05);
        blitRotatedEllipse(ctx, YELLOW, pxTip, pyTip, petalW * 0.8, petalH * 0.65, angle + Math.PI / 2);
    }

    ctx.fillStyle = BROWN;
    ctx.beginPath();
    ctx.arc(x, y, centerR, 0, Math.PI * 2);
    ctx.fill();

    ctx.strokeStyle = ORANGE;
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.arc(x, y, centerR, 0, Math.PI * 2);
    ctx.stroke();

    const numSeeds = Math.floor(28 * progress);
    if (numSeeds > 1) {
        const goldenAngle = 137.5 * (Math.PI / 180);
        ctx.fillStyle = "rgb(45, 25, 10)";
        for (let i = 1; i < numSeeds; i++) {
            const r = Math.sqrt(i / numSeeds) * (centerR * 0.82);
            const theta = i * goldenAngle;
            const sx = x + r * Math.cos(theta);
            const sy = y + r * Math.sin(theta);
            ctx.beginPath();
            ctx.arc(sx, sy, 2, 0, Math.PI * 2);
            ctx.fill();
        }
    }
}

function drawLeaf(ctx, x, y, length, angleDeg, progress) {
    if (progress <= 0) return;

    const smoothP = progress * progress * (3 - 2 * progress);
    const l = length * smoothP;
    const a = (angleDeg * Math.PI) / 180;
    const cosA = Math.cos(a), sinA = Math.sin(a);
    const perpA = a + Math.PI / 2;
    const cosP = Math.cos(perpA), sinP = Math.sin(perpA);

    const tip = [x + l * cosA, y + l * sinA];
    const numPts = 8;
    const leftPts = [], rightPts = [];

    for (let i = 0; i <= numPts; i++) {
        const t = i / numPts;
        const px = x + l * t * cosA;
        const py = y + l * t * sinA;
        const width = Math.sin(t * Math.PI) * (l * 0.28);
        leftPts.push([px + width * cosP, py + width * sinP]);
        rightPts.unshift([px - width * cosP, py - width * sinP]);
    }

    const allPts = leftPts.concat(rightPts);
    ctx.fillStyle = STEM_GREEN;
    ctx.beginPath();
    ctx.moveTo(allPts[0][0], allPts[0][1]);
    for (let i = 1; i < allPts.length; i++) ctx.lineTo(allPts[i][0], allPts[i][1]);
    ctx.closePath();
    ctx.fill();

    ctx.strokeStyle = DARK_GREEN;
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.lineTo(tip[0], tip[1]);
    ctx.stroke();
}

// -----------------------------------------------------------------------------
// Data Definition & Initialization
// -----------------------------------------------------------------------------
const stems = [
    {
        start: [400, 750], ctrl1: [320, 600], ctrl2: [220, 450], end: [260, 320],
        flower: "pink_tulip", color: PINK, base_color: PINK_BASE, delay: 0.0,
        sway_amp: 3.2, sway_freq: 2.2, rot_jitter: -3.5, phase_shift: 0.0,
        leaves: [[0.3, -30, 1.05], [0.5, -150, 0.95], [0.7, -25, 1.0]]
    },
    {
        start: [400, 750], ctrl1: [410, 580], ctrl2: [390, 400], end: [400, 240],
        flower: "sunflower", color: YELLOW, base_color: YELLOW_BASE, delay: 0.33,
        sway_amp: 6.5, sway_freq: 1.4, rot_jitter: 2.0, phase_shift: 1.2,
        leaves: [[0.25, -160, 0.9], [0.45, -20, 1.1], [0.65, -145, 1.0]]
    },
    {
        start: [400, 750], ctrl1: [450, 600], ctrl2: [550, 480], end: [580, 340],
        flower: "white_daisy", color: WHITE, base_color: WHITE_BASE, delay: 0.66,
        sway_amp: 2.1, sway_freq: 2.8, rot_jitter: 4.5, phase_shift: 2.4,
        leaves: [[0.35, -20, 1.0], [0.55, -160, 0.92]]
    },
    {
        start: [400, 750], ctrl1: [350, 620], ctrl2: [300, 500], end: [340, 420],
        flower: "red_tulip", color: RED, base_color: RED_BASE, delay: 1.0,
        sway_amp: 3.0, sway_freq: 2.0, rot_jitter: -1.8, phase_shift: 0.8,
        leaves: [[0.3, -150, 0.98], [0.6, -30, 1.08]]
    },
    {
        start: [400, 750], ctrl1: [430, 620], ctrl2: [480, 520], end: [500, 450],
        flower: "pink_daisy", color: PINK, base_color: PINK_BASE, delay: 1.33,
        sway_amp: 2.3, sway_freq: 2.6, rot_jitter: -4.0, phase_shift: 1.9,
        leaves: [[0.4, -25, 1.04], [0.7, -155, 0.95]]
    },
    {
        start: [400, 750], ctrl1: [480, 580], ctrl2: [620, 450], end: [660, 300],
        flower: "red_tulip", color: RED, base_color: RED_BASE, delay: 1.66,
        sway_amp: 3.5, sway_freq: 1.9, rot_jitter: 3.2, phase_shift: 3.1,
        leaves: [[0.3, -20, 1.0], [0.5, -160, 1.06], [0.72, -15, 0.88]]
    }
];

// Pre-calculate LUTs (Fixed camelCase call)
stems.forEach(stem => {
    stem.lut = precalculateStemBezierLut(stem, 40);
});

// -----------------------------------------------------------------------------
// Main Game Loop
// -----------------------------------------------------------------------------
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

const bgSurface = createBackgroundSurface();
const glowSurf = createGlowSurface();

let time = 0.0;
const growthRate = 0.9;
const driftingPetals = [];
let isResetting = false;
let resetTimer = 0.0;
let lastTimestamp = performance.now();

window.addEventListener('keydown', (e) => {
    if ((e.key === 'r' || e.key === 'R') && !isResetting) {
        isResetting = true;
        resetTimer = 0.0;
        stems.forEach(stem => {
            const endPt = stem.end;
            const color = stem.color;
            for (let i = 0; i < 5; i++) {
                driftingPetals.push({
                    x: endPt[0],
                    y: endPt[1],
                    vx: (Math.random() - 0.5) * 120,
                    vy: -Math.random() * 80 - 40,
                    radius: Math.random() * 4 + 6,
                    color: color,
                    alpha: 255
                });
            }
        });
    }
});

function drawPoly(ctx, pts, fillStyle, strokeStyle = null, lineWidth = 1) {
    ctx.beginPath();
    ctx.moveTo(pts[0][0], pts[0][1]);
    for (let i = 1; i < pts.length; i++) ctx.lineTo(pts[i][0], pts[i][1]);
    ctx.closePath();
    if (fillStyle) {
        ctx.fillStyle = fillStyle;
        ctx.fill();
    }
    if (strokeStyle) {
        ctx.strokeStyle = strokeStyle;
        ctx.lineWidth = lineWidth;
        ctx.stroke();
    }
}

function gameLoop(currentTimestamp) {
    const dt = Math.min((currentTimestamp - lastTimestamp) / 1000.0, 0.1);
    lastTimestamp = currentTimestamp;

    time += dt;

    const globalWind = Math.sin(time * 1.8) * 4.0 + Math.sin(time * 0.7) * 2.0;

    if (isResetting) {
        resetTimer += dt;
        if (resetTimer >= 1.2) {
            isResetting = false;
            time = 0.0;
            driftingPetals.length = 0;
        }
    }

    // Clear Screen and draw Pre-baked Render Buffers
    ctx.drawImage(bgSurface, 0, 0);
    ctx.drawImage(glowSurf, 100, 150);

    // Grass blades
    for (let i = -5; i <= 5; i++) {
        const lx = 400 + i * 15;
        const grassSway = globalWind * 0.5 + Math.sin(time * 1.5 + i) * 2.0;
        ctx.strokeStyle = DARK_GREEN;
        ctx.lineWidth = 3;
        ctx.beginPath();
        ctx.arc(lx - 40 + grassSway + 40, 650 + 100, 100, Math.PI * 1.22, Math.PI * 1.77);
        ctx.stroke();
    }

    // Bouquet wrapping
    const paper = [[310, 730], [490, 730], [560, 520], [240, 520]];
    drawPoly(ctx, paper, "rgb(235, 220, 190)");
    drawPoly(ctx, [[240, 520], [310, 730], [270, 520]], "rgb(215, 200, 170)");

    ctx.strokeStyle = "rgb(210, 195, 165)";
    ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(400, 520); ctx.lineTo(400, 730); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(480, 520); ctx.lineTo(450, 730); ctx.stroke();

    ctx.strokeStyle = "rgb(255, 245, 220)";
    ctx.lineWidth = 3;
    ctx.beginPath(); ctx.moveTo(240, 520); ctx.lineTo(560, 520); ctx.stroke();

    if (!isResetting) {
        stems.forEach((stem, stemIdx) => {
            const startTime = stem.delay;
            if (time < startTime) return;

            const t = Math.min(1.0, (time - startTime) * growthRate);
            const smoothT = t * t * (3 - 2 * t);

            const stemWind = globalWind + Math.sin(time * stem.sway_freq + stem.phase_shift) * stem.sway_amp;
            const lut = stem.lut;
            const maxIdx = Math.floor(smoothT * (lut.length - 1));

            // Stems
            if (maxIdx >= 1) {
                const entry0 = lut[0];
                let prevX = entry0.base_x + stemWind * entry0.wind_weight;
                let prevY = entry0.base_y;

                for (let k = 1; k <= maxIdx; k++) {
                    const entry = lut[k];
                    const currX = entry.base_x + stemWind * entry.wind_weight;
                    const currY = entry.base_y;

                    const progressAlongStem = k / maxIdx;
                    const thick = Math.floor(lerp(5.0, 2.0, progressAlongStem));

                    ctx.strokeStyle = GREEN;
                    ctx.lineWidth = Math.max(1, thick);
                    ctx.beginPath();
                    ctx.moveTo(prevX, prevY);
                    ctx.lineTo(currX, currY);
                    ctx.stroke();

                    prevX = currX;
                    prevY = currY;
                }
            }

            // Leaves
            stem.leaves.forEach(([leafPct, leafAngle, leafScale]) => {
                if (smoothT > leafPct) {
                    const leafProgress = Math.min(1.0, (smoothT - leafPct) / 0.3);
                    const lutIdx = Math.floor(leafPct * (lut.length - 1));
                    const entry = lut[lutIdx];
                    const lx = entry.base_x + stemWind * entry.wind_weight;
                    const ly = entry.base_y;

                    drawLeaf(ctx, lx, ly, Math.floor(38 * leafScale), leafAngle + stemWind * 0.3, leafProgress);
                }
            });

            // Flowers
            if (smoothT >= 0.8) {
                const bloomProgress = Math.min(1.0, (smoothT - 0.8) / 0.2);
                const endEntry = lut[lut.length - 1];
                const endX = endEntry.base_x + stemWind * endEntry.wind_weight;
                const endY = endEntry.base_y;

                const fType = stem.flower;
                const rotJ = stem.rot_jitter + stemWind * 0.4;
                const breathScale = 1.0 + 0.018 * Math.sin(time * 2.5 + stemIdx);

                if (fType === "pink_tulip" || fType === "red_tulip") {
                    const rotRad = (rotJ * Math.PI) / 180;
                    const cosR = Math.cos(rotRad), sinR = Math.sin(rotRad);
                    const c = (fType === "pink_tulip") ? PINK : RED;
                    const sz = (fType === "pink_tulip") ? 50 : 45;
                    drawTulip(ctx, endX, endY, sz, bloomProgress, c, stem.base_color, rotJ, breathScale, cosR, sinR);
                } else if (fType === "white_daisy") {
                    drawDaisy(ctx, endX, endY, 60, bloomProgress, WHITE, stem.base_color, YELLOW, rotJ, breathScale);
                } else if (fType === "pink_daisy") {
                    drawDaisy(ctx, endX, endY, 55, bloomProgress, PINK, stem.base_color, YELLOW, rotJ, breathScale);
                } else if (fType === "sunflower") {
                    drawSunflower(ctx, endX, endY, 80, bloomProgress, rotJ, breathScale);
                }
            }
        });
    }

    // Particles & spores
    if (isResetting) {
        driftingPetals.forEach(p => {
            p.x += p.vx * dt;
            p.y += p.vy * dt;
            p.vy += 120.0 * dt;
            p.alpha = Math.max(0, Math.floor(255 * (1.0 - resetTimer / 1.2)));

            if (p.alpha > 0) {
                const pSurf = getCachedPetalParticle(p.radius, p.color, p.alpha);
                ctx.drawImage(pSurf, Math.floor(p.x), Math.floor(p.y));
            }
        });
    }

    for (let p = 0; p < 12; p++) {
        const px = (400 + Math.sin(time * 0.8 + p * 1.5) * 280) % WIDTH;
        const py = (700 - ((time * 25 + p * 60) % 500));
        const pAlpha = Math.max(0, Math.min(255, Math.floor(120 + 100 * Math.sin(time * 2 + p))));

        const sporeSurf = getCachedPetalParticle(2, "rgb(255, 240, 180)", pAlpha);
        ctx.drawImage(sporeSurf, Math.floor(px), Math.floor(py));
    }

    // Ribbon
    const ribbonWaveLeft = globalWind * 0.6 + Math.sin(time * 4.0) * 2.0;
    const ribbonWaveRight = globalWind * 0.6 + Math.cos(time * 4.0) * 2.0;

    ctx.fillStyle = "rgb(140, 20, 35)";
    ctx.beginPath();
    ctx.ellipse(400, 670, 12, 12, 0, 0, Math.PI * 2);
    ctx.fill();

    drawPoly(ctx, [[395, 665], [370 + ribbonWaveLeft, 720], [385 + ribbonWaveLeft, 715], [400, 670]], "rgb(190, 30, 50)");
    drawPoly(ctx, [[405, 665], [430 + ribbonWaveRight, 720], [415 + ribbonWaveRight, 715], [400, 670]], "rgb(190, 30, 50)");
    drawPoly(ctx, [[390, 665], [360, 690], [385, 685]], "rgb(220, 40, 60)");
    drawPoly(ctx, [[410, 665], [440, 690], [415, 685]], "rgb(220, 40, 60)");

    ctx.strokeStyle = "rgb(255, 100, 120)";
    ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(390, 665); ctx.lineTo(368, 688); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(410, 665); ctx.lineTo(432, 688); ctx.stroke();

    ctx.fillStyle = "rgb(220, 40, 60)";
    ctx.beginPath(); ctx.arc(400, 660, 10, 0, Math.PI * 2); ctx.fill();

    ctx.fillStyle = "rgb(255, 120, 140)";
    ctx.beginPath(); ctx.arc(398, 658, 4, 0, Math.PI * 2); ctx.fill();

    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
</script>
</body>
</html>
