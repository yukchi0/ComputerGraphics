# PULSE — 게임 개발 리포트

> **학번**: 2021204085  
> **이름**: 이서환  
> **게임 구동 링크**: https://yukchi0.github.io/ComputerGraphics/  
> **리포트 링크**: https://github.com/yukchi0/ComputerGraphics/blob/main/REPORT.md

---

## 목차

1. [게임 개요 및 기획](#1-게임-개요-및-기획)
2. [강의 내용과 구현 매핑](#2-강의-내용과-구현-매핑)
3. [GI 기술 구현 — DDGI 시뮬레이션](#3-gi-기술-구현--ddgi-시뮬레이션)
4. [소나 파동 셰이더 시스템](#4-소나-파동-셰이더-시스템)
5. [게임 시스템 상세 설명](#5-게임-시스템-상세-설명)
6. [렌더링 파이프라인 전체 흐름](#6-렌더링-파이프라인-전체-흐름)

---

## 1. 게임 개요 및 기획

### 1.1 게임 컨셉

**PULSE**는 1인칭 시점 공포-스텔스 게임이다.  
핵심 컨셉: **"심박수가 세계를 밝힌다 — 당신의 공포가 빛이 된다."**

씬은 기본적으로 완전히 어두운 상태다. 플레이어의 심박수(BPM)가 오를수록 자동으로 소나 파동이 강해지고, 주변 환경의 GI(전역 조명) 세기도 함께 변화한다. 발소리와 소나 파동이 적에게 닿으면 위치가 노출되어 쫓기게 된다.

### 1.2 조작법

| 입력 | 동작 | 게임 내부 시스템 |
|------|------|------------------|
| WASD | 이동 | 기본 속도로 이동하며 규칙적인 발소리 파동을 생성. |
| 마우스 | 시점 변경 | 카메라 방향을 회전 |
| Shift | 달리기 (BPM ↑, 발소리 큼) | 이동 속도가 빨리지나 심박수가 빨리지고 발걸음 파동의 크기와 속도가 상승 |
| C | 살금살금 이동 (BPM ↓, 발소리 작음) | 이동 속도가 줄어들고, 발소리 파동의 생성 면적이 줄어들어 스텔스가 가능 |
| 클릭 | 적 근접 공격 | 적과의 거리가 근접했을 때 적을 처치함 |

### 1.3 게임 상태 단계 (BPM Stage)

| 단계 | BPM 범위 | 색상 | 주요 특징 |
|------|----------|------|-----------|
| CALM | 60 ~ 89 | 초록 `#1D9E75` | 씬 거의 완전히 어둠, 소나만 시야 |
| TENSE | 90 ~ 129 | 파랑 `#378ADD` | Ambient GI 점등, 적 경계 강화 |
| PANIC | 130 ~ 160 | 오렌지 `#EF9F27` | 씬 전체 밝아짐, 적 즉시 alert |

### 1.4 기획 의도

- **BPM이 GI 세기를 직접 제어**: 플레이어가 긴장할수록 씬이 밝아지는 역설적 메카닉
- **소나 파동이 유일한 시야**: CALM 상태에서는 발소리 파동만이 지형을 드러냄
- **적 감염 시스템**: 한 적이 awakened 상태가 되면 반경 8 unit 내 인접 적이 연쇄적으로 각성

---

## 2. 강의 내용과 구현 매핑

### 2.1 Global Illumination 개요

강의에서 다룬 GI의 핵심 개념은 다음과 같다.

- **Direct Illumination**: 광원 → 표면의 단일 경로
- **Indirect Illumination**: 한 번 이상 반사된 빛. 현실감의 핵심
- **Irradiance Probe**: 씬 내 특정 지점에서의 입사 radiance를 구면 조화 함수(SH)로 인코딩
- **DDGI (Dynamic Diffuse GI)**: 실시간으로 probe radiance를 업데이트하여 동적 씬에서도 GI 표현

PULSE에서의 구현 대응:

| GI 개념 | PULSE 구현 |
|---------|-----------|
| Irradiance probe grid | `LightProbe` 3×1×3 grid, 16 unit 간격 |
| Probe radiance 인코딩 | `probe.sh.coefficients[0]` SH L0 항 직접 설정 |
| Probe intensity 제어 | BPM 단계에 따라 `probe.intensity = giIntensity * 0.4` |
| Secondary GI (indirect bounce) | 적 glowLight → Point Light가 씬에 간접광 역할 |
| GI 세기 조절 | `giIntensity` 값을 BPM, screamFlash에 따라 실시간 변화 |

![DDGI probe grid와 GI 세기 변화](screenshots/gi_bpm_comparison.png)
*그림 2.1 — CALM(왼쪽)과 PANIC(오른쪽) 상태 비교. PANIC 시 probe intensity와 ambientLight가 함께 상승하여 씬 전체가 밝아진다.*

### 2.2 구면 조화 함수 (Spherical Harmonics)

강의 내용: SH는 구면 위의 함수를 기저 함수의 선형 결합으로 표현한다. L0(상수항 1개), L1(3개), L2(5개) 등 band로 구성. 저주파 조명 분포를 효율적으로 압축.

구현:
```javascript
const probe = new THREE.LightProbe();
probe.sh.coefficients[0].set(0.05, 0.04, 0.06);  // L0 항: RGB 기저 radiance
probe.intensity = 0.0;
```

Three.js의 `LightProbe`는 SH L2까지 지원(9개 계수). 본 구현에서는 L0(DC항, index 0)만 설정하여 uniform한 ambient radiance를 표현한다. BPM 단계에 따라 `probe.intensity`를 동적으로 변화시켜 GI 세기를 제어한다.

![LightProbe SH 설정 — 씬 내 배치](screenshots/lightprobe_placement.png)
*그림 2.2 — 씬 내 3×1×3 LightProbe 배치. 각 probe는 16 unit 간격으로 씬 전체를 커버한다.*

### 2.3 Lambert 조명 모델

강의 내용: Lambertian surface는 모든 방향으로 동일한 세기로 빛을 반사한다 (diffuse-only). `I = max(0, N·L) * kd * Ld`.

구현: 씬의 모든 오브젝트는 `MeshLambertMaterial` 사용.

```javascript
const roomMat  = new THREE.MeshLambertMaterial({ color: 0x222222 });
const crateMat = new THREE.MeshLambertMaterial({ color: 0x4a3a22 });
```

Specular를 제거함으로써 공포 분위기에 맞는 무광택 표면감을 구현했다. Three.js의 `MeshLambertMaterial`은 씬 내 `LightProbe`의 SH를 diffuse ambient로 자동 반영한다.

![MeshLambertMaterial — 소나 파동 조명 하 벽면](screenshots/lambert_wall.png)
*그림 2.3 — 소나 파동이 지나갈 때 벽면이 Lambertian diffuse로 반응하는 장면. Specular highlight 없이 부드럽게 밝아진다.*

### 2.4 Shadow Mapping

강의 내용: Shadow map은 광원 시점에서 depth buffer를 렌더링한 뒤, 카메라 시점 렌더링 시 depth 비교로 그림자를 판별한다. PCF(Percentage Closer Filtering)로 soft edge 구현.

구현:
```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;  // PCF soft shadow

const dirLight = new THREE.DirectionalLight(0xffffff, 0.2);
dirLight.castShadow = true;
dirLight.shadow.mapSize.set(1024, 1024);           // shadow map 해상도

const heartbeatLight = new THREE.PointLight(0xffffff, 0, 40);
heartbeatLight.castShadow = true;
```

심박 빛(`heartbeatLight`)이 `castShadow = true`로 설정되어, 박자가 뛸 때마다 동적 그림자가 생성된다.

![Shadow mapping — 심박 빛 그림자](screenshots/shadow_heartbeat.png)
*그림 2.4 — 심박 빛이 순간적으로 켜질 때 PCFSoftShadowMap 기반 동적 그림자가 벽과 상자에 생성되는 장면.*

### 2.5 Tone Mapping & 렌더러 설정

강의 내용: HDR 렌더링 후 tone mapping으로 LDR 디스플레이에 맞게 압축. 물리 기반 조명에서 필수.

구현에서는 기본 Three.js Linear tone mapping을 사용하며, `WebGLRenderer`의 `shadowMap`과 `antialias`가 활성화된다.

---

## 3. GI 기술 구현 — DDGI 시뮬레이션

### 3.1 DDGI 개요

DDGI (Dynamic Diffuse Global Illumination, Majercik et al. 2019)는:
1. 씬에 probe grid를 균일 배치
2. 각 probe에서 여러 방향으로 ray를 발사, hit point의 radiance를 SH로 인코딩
3. 렌더링 시 probe grid를 trilinear interpolation으로 샘플링하여 diffuse GI 계산

PULSE의 근사 구현:
- Ray casting 대신 **BPM 단계와 screamFlash 값**으로 probe radiance를 직접 제어
- Trilinear probe interpolation 대신 Three.js가 `LightProbe`를 자동으로 가장 가까운 probe에서 샘플링
- 이를 통해 소나 파동 확장 → BPM 상승 → probe 활성화 → 씬 밝아짐의 인과 관계를 구현

### 3.2 Probe Grid 구성 및 배치

```javascript
// Section 4: 조명 시스템 (DDGI 시뮬레이션)
const probeGrid = { x: 3, y: 1, z: 3 };   // 3×1×3 = 9개 probe
const probeSpacing = 16;                    // 16 unit 간격
const lightProbes = [];

for (let ix = 0; ix < probeGrid.x; ix++) {
  for (let iy = 0; iy < probeGrid.y; iy++) {
    for (let iz = 0; iz < probeGrid.z; iz++) {
      const probe = new THREE.LightProbe();
      probe.position.set(
        (ix - 1) * probeSpacing,   // x: -16, 0, 16
        1 + iy * 2,                // y: 1
        (iz - 1) * probeSpacing    // z: -16, 0, 16
      );
      probe.sh.coefficients[0].set(0.05, 0.04, 0.06);  // 초기 미약한 ambient
      probe.intensity = 0.0;
      scene.add(probe);
      lightProbes.push(probe);
    }
  }
}
```

probe는 평면상 3×3으로 y=1 높이에 배치되어 36×36 unit 범위의 씬을 커버한다.

![Probe grid 배치 — 씬 위에서 본 구도](screenshots/probe_grid_overview.png)
*그림 3.1 — 3×1×3 LightProbe grid. 각 probe가 16 unit 간격으로 씬을 균등하게 커버한다.*

### 3.3 GI 세기 동적 제어 (BPM → giIntensity)

```javascript
// Section 8: BPM → 시각 효과 매핑
function applyVisualState(bpmNorm, stage) {
  let giIntensity;
  if (stage === 'calm')
    giIntensity = 0.0;
  else if (stage === 'tense')
    giIntensity = THREE.MathUtils.lerp(0.03, 0.4, (BPM.current - 90) / 40);
  else
    giIntensity = THREE.MathUtils.lerp(0.4,  1.8, (BPM.current - 130) / 30);

  // screamFlash 시 즉각 최대 GI
  const flash = screamFlash;
  giIntensity = THREE.MathUtils.lerp(giIntensity, 1.8, flash);

  // AmbientLight와 모든 probe에 동시 반영
  ambientLight.intensity = giIntensity;
  lightProbes.forEach(probe => { probe.intensity = giIntensity * 0.4; });
```

- `giIntensity`는 BPM 연속 함수로, CALM에서 0, PANIC 최대에서 1.8까지 선형 보간
- `screamFlash`(적 각성 시 1.0, 이후 2초에 걸쳐 감쇠)가 giIntensity를 즉시 1.8로 당김
- `lightProbes`에 `giIntensity * 0.4`를 일괄 적용하여 probe 기반 indirect light 강도 조절

![GI intensity — CALM vs TENSE vs PANIC](screenshots/gi_three_stages.png)
*그림 3.2 — 같은 위치에서 BPM 단계별 씬 밝기 비교. CALM(거의 어둠), TENSE(미약한 청색 ambient), PANIC(오렌지 ambient + point lights 활성).*

### 3.4 Bay Point Lights (Indirect GI Bounce 근사)

```javascript
// Bay 중앙 6개 위치에 PointLight 배치
const pointLights = [];
const plPositions = BAY_CENTERS.map(([x, z]) => [x, 4, z]);
plPositions.forEach(([x, y, z]) => {
  const pl = new THREE.PointLight(0x334466, 0, 25);  // 초기 intensity 0
  pl.position.set(x, y, z);
  scene.add(pl);
  pointLights.push(pl);
});

// applyVisualState 내에서 동적 제어
let plIntensity;
if (stage === 'calm')
  plIntensity = 0;
else if (stage === 'tense')
  plIntensity = THREE.MathUtils.lerp(0, 2, (BPM.current - 90) / 40);
else
  plIntensity = THREE.MathUtils.lerp(2, 6, (BPM.current - 130) / 30);

pointLights.forEach(pl => {
  pl.intensity = plIntensity;
  if (stage === 'panic') pl.color.set(0xffaa44);   // PANIC: 오렌지
  else                   pl.color.set(0x334466);   // TENSE: 청색
  pl.color.lerp(new THREE.Color(0xffaa44), flash); // scream 시 즉시 오렌지
});
```

Bay 위치의 PointLight들이 TENSE 이상에서 활성화되어 간접광(2차 반사광)처럼 공간을 채운다. 이는 실제 DDGI의 secondary bounce를 PointLight로 근사한 것이다.

![Bay PointLights — TENSE 상태 활성화](screenshots/bay_lights_tense.png)
*그림 3.3 — TENSE 상태에서 Bay PointLight가 활성화되어 천장 쪽에서 내려오는 간접광처럼 공간을 밝히는 장면.*

### 3.5 Pulse Pool Light (적 파동 GI)

적의 소나 파동(Enemy Pulse)이 생성될 때 `lightPool`에서 PointLight를 꺼내 파동 원점에 배치한다. 이 광원은 파동이 확산되면서 함께 확장되어 실시간 GI처럼 동작한다.

```javascript
// PULSE_POOL_SIZE = 12개의 재사용 가능한 PointLight 풀
const lightPool = [];
for (let i = 0; i < PULSE_POOL_SIZE; i++) {
  const pl = new THREE.PointLight(0xffffff, 0, 0);
  pl.visible = false;
  scene.add(pl);
  lightPool.push({ light: pl, inUse: false });
}

// Pulse.update() 내에서 반경과 함께 확장
if (this.pe) {
  this.pe.light.distance  = this.radius * 1.5 + 0.5;  // 파동과 함께 확장
  this.pe.light.intensity = fade * 2.0;
}
```

![적 파동 PointLight — 씬 GI 기여](screenshots/enemy_pulse_light.png)
*그림 3.4 — 적의 소나 파동(빨간 링)과 함께 PointLight가 확장되며 주변 벽에 동적 GI를 생성하는 장면.*

---

## 4. 소나 파동 셰이더 시스템

### 4.1 아키텍처 개요

소나 파동은 씬 지오메트리의 표면에 투명 오버레이 메쉬를 붙이고, GLSL 커스텀 셰이더로 파동 링을 렌더링하는 방식이다.

**오버레이 메쉬 배치 방법:**
- 바닥: `PlaneGeometry(37.2, 49.2)`, y=0.02에 배치
- 각 벽: 벽 면마다 `PlaneGeometry` 부착 (±0.02 epsilon offset, z-fighting 방지)
- `renderOrder = 1`, `depthWrite = false`로 z-fighting 없이 표면 위에 렌더링

```javascript
function addSonarOverlays() {
  // 바닥 오버레이
  const floorOverlay = new THREE.Mesh(
    new THREE.PlaneGeometry(HOUSE_HALF_X * 2 + 0.6, HOUSE_HALF_Z * 2 + 0.6),
    sonarMat
  );
  floorOverlay.rotation.x = -Math.PI / 2;
  floorOverlay.position.y = 0.02;
  floorOverlay.renderOrder = 1;
  scene.add(floorOverlay);

  // 각 벽면 오버레이 (EPS = 0.02 offset)
  walls.forEach(w => {
    const { hx, hy, hz } = w.userData;
    if (hx > hz * 0.3) {
      [1, -1].forEach(sign => {
        const p = new THREE.Mesh(new THREE.PlaneGeometry(hx * 2, hy * 2), sonarMat);
        p.position.set(w.position.x, cy, w.position.z + sign * (hz + EPS));
        p.renderOrder = 1;
        scene.add(p);
      });
    }
    // ... Z축 방향 벽도 동일
  });
}
```

![소나 오버레이 메쉬 — 바닥 및 벽 배치](screenshots/sonar_overlay_mesh.png)
*그림 4.1 — 바닥과 각 벽면에 sonarMat이 적용된 투명 오버레이 메쉬가 부착된 구조. 파동이 이 메쉬를 통해 렌더링된다.*

### 4.2 Vertex Shader

```glsl
// sonarVert
varying vec3 vWorldPos;
void main() {
  vec4 wp    = modelMatrix * vec4(position, 1.0);
  vWorldPos  = wp.xyz;   // world space 위치를 fragment shader로 전달
  gl_Position = projectionMatrix * viewMatrix * wp;
}
```

각 오버레이 메쉬의 정점을 world space로 변환해 `vWorldPos`에 저장한다. Fragment shader에서 파동 원점과의 XZ 거리를 계산하는 데 사용된다.

### 4.3 Fragment Shader (소나 파동 핵심 로직)

```glsl
// sonarFrag
precision highp float;
varying vec3 vWorldPos;
uniform vec3  uCenters[16];    // 파동 원점 (최대 16개)
uniform float uRadii[16];      // 각 파동의 현재 반경
uniform vec3  uColors[16];     // RGB 색상 (BPM 단계별)
uniform float uWaveWidth;      // 파동 링 두께 (= 0.28)
uniform float uAttenK;         // 거리 감쇠 계수 (= 0.05)

void main() {
  vec3  col   = vec3(0.0);
  float total = 0.0;
  for (int i = 0; i < 16; i++) {
    float r = uRadii[i];
    if (r < 0.0) continue;             // 비활성 슬롯 skip

    float dist = length(vWorldPos.xz - uCenters[i].xz);  // XZ 평면 거리
    float wave = 1.0 - smoothstep(0.0, uWaveWidth, abs(dist - r));
    //                               ↑ 링 두께
    //              dist=r일 때 wave=1.0, |dist-r|>uWaveWidth이면 0.0

    float atten = 1.0 / (1.0 + r * r * uAttenK);  // 역제곱 감쇠
    col   += uColors[i] * wave * atten;
    total += wave * atten;
  }
  if (total < 0.005) discard;          // 기여 없으면 픽셀 버림
  col = min(col, vec3(1.0));
  gl_FragColor = vec4(col, min(total * 1.2, 1.0));
}
```

핵심 수식:
- `abs(dist - r)`: 픽셀이 파동 링으로부터 얼마나 떨어졌는지
- `smoothstep(0, 0.28, ...)`: 링 경계를 0.28 unit 폭으로 부드럽게 처리
- `1 / (1 + r² × 0.05)`: 파동이 멀어질수록 감쇠 (물리 기반 역제곱 법칙)

![소나 Fragment Shader — 파동 링 렌더링](screenshots/sonar_shader_ring.png)
*그림 4.2 — 플레이어 발소리 파동(초록)이 바닥에 링 형태로 렌더링되는 장면. smoothstep으로 처리된 부드러운 엣지가 보인다.*

### 4.4 Enemy Ghost Shader (적 가시화)

적 메쉬에는 `enemyGhostMat`이 적용되는데, 이는 소나 파동이 닿은 영역만 적을 보이게 하는 셰이더다.

```glsl
// enemyGhostFrag — 파동 미접촉 시 반투명 유령 처리
void main() {
  // ... (sonarFrag와 동일한 파동 계산)
  if (total < 0.005) {
    gl_FragColor = vec4(0.5, 0.08, 0.08, 0.04);  // 파동 없으면 거의 투명
    return;
  }
  // 파동이 닿으면 정상 표시
  col = min(col, vec3(1.0));
  gl_FragColor = vec4(col, min(total * 1.2, 1.0));
}
```

```glsl
// enemyGhostVert — 스켈레탈 애니메이션 지원 vertex shader
#include <skinning_pars_vertex>
varying vec3 vWorldPos;
void main() {
  #include <skinbase_vertex>
  #include <begin_vertex>
  #include <skinning_vertex>          // 본 변환 적용
  vec4 wp = modelMatrix * vec4(transformed, 1.0);
  vWorldPos = wp.xyz;
  gl_Position = projectionMatrix * viewMatrix * wp;
}
```

스켈레탈 애니메이션이 된 상태의 정점 위치를 world space로 변환하여 파동과의 거리를 정확히 계산한다.

![Enemy Ghost Shader — 파동 미접촉 vs 접촉](screenshots/enemy_ghost_shader.png)
*그림 4.3 — 소나 파동이 닿지 않은 적(왼쪽, 거의 투명)과 파동이 통과하는 적(오른쪽, 색상 표시) 비교.*

### 4.5 Pulse 클래스 — 벽 반사(Reflection) 파동

```javascript
class Pulse {
  update(dt) {
    this.prevRadius = this.radius;
    this.radius    += this.speed * dt;
    const life = 1 - this.radius / this.maxDist;
    if (life <= 0 || this.slot < 0) { this.dead = true; return; }

    // fade 계산: 적 파동은 제곱 감쇠, 플레이어 파동은 끝부분만 fade
    const fade = this.isEnemy
      ? 1.0 - Math.pow(this.radius / this.maxDist, 2)
      : (life < 0.12 ? life / 0.12 : 1.0);

    sonarUniforms.uRadii.value[this.slot] = this.radius;
    sonarUniforms.uColors.value[this.slot].set(c.r * fade, c.g * fade, c.b * fade);

    // 벽 충돌 감지 → 반사 파동 생성
    if (!this.isReflect && !this.isEnemy) {
      const hits = getCollisionPoints(this.ox, this.oz, this.radius, this.prevRadius);
      hits.forEach(pt => {
        const key = `${Math.round(pt.x)}_${Math.round(pt.z)}`;
        if (this._collided.has(key)) return;
        this._collided.add(key);
        // 반사 파동: maxDist 40%, speed 80%
        activePulses.push(new Pulse(
          pt, this.color.getHex(),
          this.maxDist * 0.4, this.speed * 0.8,
          false, true
        ));
      });
    }
  }
}
```

`getCollisionPoints`는 16방향으로 Raycaster를 발사해 파동 링이 벽과 교차하는 지점을 찾는다. 충돌 지점에서 40% 크기의 2차 반사 파동이 생성되어 빛의 간접 반사를 시뮬레이션한다.

![소나 반사 파동 — 벽 충돌 후 2차 파동](screenshots/sonar_reflection.png)
*그림 4.4 — 플레이어 파동(큰 링)이 벽에 닿아 반사 파동(작은 링들)이 생성되는 장면. 파동이 벽 너머를 간접적으로 조명한다.*

### 4.6 슬롯 풀 관리

최대 16개의 파동을 `Float32Array` 기반 슬롯 풀로 관리한다.

```javascript
const sonarUniforms = {
  uRadii: { value: new Float32Array(MAX_PULSES).fill(-1) }, // -1: 빈 슬롯
};

// Pulse 생성 시 슬롯 할당
this.slot = -1;
for (let i = 0; i < MAX_PULSES; i++) {
  if (sonarUniforms.uRadii.value[i] < 0) { this.slot = i; break; }
}

// Pulse 소멸 시 슬롯 해제
dispose() {
  if (this.slot >= 0) sonarUniforms.uRadii.value[this.slot] = -1;
}
```

---

## 5. 게임 시스템 상세 설명

### 5.1 BPM 시스템

```javascript
// Section 5: 심박수(BPM) 시스템
const BPM = {
  current: 72, target: 72,
  min: 60, max: 160,

  update(dt) {
    // exponential smoothing: 목표값으로 부드럽게 수렴
    this.current += (this.target - this.current) * dt * 1.2;
    this.current = Math.max(this.min, Math.min(this.max, this.current));

    const moving   = keys['KeyW'] || keys['KeyS'] || ...;
    const running  = moving && keys['ShiftLeft'];
    const sneaking = moving && keys['KeyC'] && !keys['ShiftLeft'];

    if (running)       this.target = Math.min(this.max, this.target + dt * 30); // +30/s
    else if (sneaking) this.target = Math.max(72, this.target - dt * 2);        // -2/s
    else if (moving)   // 100 BPM 수렴
    else               this.target = Math.max(72, this.target - dt * 3);        // -3/s 자연 감소
  }
};
```

달리기 시 초당 30 BPM 상승, 정지 시 초당 3 BPM 하강. `current`는 `target`을 향해 지수 평활로 수렴한다.

![BPM 시스템 — HUD 바 변화](screenshots/bpm_hud.png)
*그림 5.1 — 좌: CALM(72 BPM, 초록), 중: TENSE(110 BPM, 파랑), 우: PANIC(155 BPM, 오렌지) 상태의 HUD 바.*

### 5.2 심박 타이밍 & 발소리 파동

```javascript
// Section 10, 12: 심박 타이밍
function getBeatInterval() {
  return 60 / BPM.current;  // 72 BPM → 0.833초 간격
}

// 심박 빛: 박자마다 flash 후 지수 감쇠
const HEARTBEAT_ATTACK  = 0.03;   // 30ms 어택
const HEARTBEAT_RELEASE = 0.7;    // 700ms 릴리즈

const hbEnvelope = heartbeatElapsed < HEARTBEAT_ATTACK
  ? heartbeatElapsed / HEARTBEAT_ATTACK
  : Math.exp(-(heartbeatElapsed - HEARTBEAT_ATTACK) / HEARTBEAT_RELEASE);
heartbeatLight.intensity = hbEnvelope * 14;

// 발소리 파동 (살금살금 시 0.2배 크기)
const SNEAK_PULSE_SCALE = 0.2;
function spawnPlayerFootstepPulse() {
  const sizeMul = playerSneaking ? SNEAK_PULSE_SCALE : 1;
  spawnPulse(camera.position, STAGE_COLORS[stage],
    THREE.MathUtils.lerp(9, 15, norm) * sizeMul,   // maxDist
    false,
    THREE.MathUtils.lerp(5, 14, norm) * sizeMul);  // speed
}
```

발소리는 `FOOTSTEP_BASE_INTERVAL = 0.45`초 기준, 달릴수록 자주 울린다. 살금살금 모드에서는 파동이 0.2배로 작아져 적에게 거의 감지되지 않는다.

![발소리 파동 — 일반 vs 살금살금](screenshots/footstep_normal_vs_sneak.png)
*그림 5.2 — 일반 걷기(큰 초록 파동)와 살금살금(매우 작은 파동) 비교. 살금살금 시 파동 반경이 80% 줄어든다.*

### 5.3 적 AI 5단계 상태 머신

```
dormant → searching → wary → (detection ≥ 100) → awakened → alert
   ↑                              ↑______________________|
   └──────── 5초 후 복귀 ─────────────────────────────────────┘
```

| 상태 | 글로우 | 심박음 | 소나 간격 | 전환 조건 |
|------|--------|--------|-----------|-----------|
| dormant | 없음 | 없음 | — | stageTimer 만료 (5~7초) |
| searching | 주황 0.8 | 0.4 vol | 1.5초 | stageTimer 만료 (3~5초) |
| wary | 오렌지 2.2 | 0.6 vol | 0.5초 | stageTimer 만료 (2~3초) |
| awakened | 빨강 4.0 | 0.8 vol | 0.3초 | scream 완료 후 → alert |
| alert | 없음 (빨강 mesh) | 0.7 vol | 0.3초 | loseTimer 5초 → dormant |

```javascript
// awakened 진입 시 인접 적 연쇄 각성 (INFECT_RADIUS = 8 unit)
_enterAwakened() {
  this.state = 'awakened';
  BPM.current = BPM.max;   // 플레이어 BPM 즉시 160으로
  BPM.target  = BPM.max;
  setMeshMaterial(this.mesh, enemyAwakenMat);  // 빨간 메시로
  this._playAction('scream');
  spawnPulse(this.mesh.position, 0xffffff, 9, true, 9);  // 백색 경고 파동
  triggerScreamFlash();   // GI 즉시 최대화

  enemies.forEach(other => {
    if (other.mesh.position.distanceTo(this.mesh.position) <= INFECT_RADIUS)
      other._infect(this.mesh.position);  // 재귀적 전파
  });
}
```

![적 AI 상태 전이 — wary → awakened](screenshots/enemy_state_transition.png)
*그림 5.3 — wary 상태(발작 애니메이션, 오렌지 글로우)에서 detection이 100에 도달해 awakened로 전환되는 순간. 백색 파동이 방사된다.*

### 5.4 Detection 시스템

적의 `detection` 값이 100에 도달하면 awakened 전환. 두 가지 방법으로 detection이 올라간다.

```javascript
_checkPulseTouch(playerPos, moving, dt) {
  // 1. 이동 소음: 5 unit 이내 이동 시 초당 60 포인트
  const dist = playerPos.distanceTo(this.mesh.position);
  if (moving && dist < HEARING_RADIUS)  // HEARING_RADIUS = 5
    this._registerNoise(playerPos, MOVE_NOISE_RATE * dt);  // 60/s

  // 2. 소나 파동 접촉: 파동 링이 적에 닿으면 +50
  activePulses.forEach(p => {
    if (p.isEnemy) return;
    const d = this.mesh.position.distanceTo(new THREE.Vector3(p.ox, 0, p.oz));
    if (Math.abs(d - p.radius) < 0.4 && !this._touchedPulses.has(p)) {
      this._touchedPulses.add(p);
      this._registerNoise(playerPos, PULSE_TOUCH_GAIN);  // +50
    }
  });
}

// detection은 비활성 시 초당 15씩 감소
this.detection = Math.max(0, this.detection - dt * DETECTION_DECAY);  // 15/s
```

PULSE_TOUCH_GAIN = 50이므로 소나 파동 2회 접촉 만으로 awakened 전환이 가능하다.

![Detection 시스템 — 소나 파동 접촉](screenshots/detection_pulse_touch.png)
*그림 5.4 — 플레이어 발소리 파동(초록 링)이 적에 닿는 순간. 적의 detection이 상승하고 글로우가 강해진다.*

### 5.5 FBX 모델 & 스켈레탈 애니메이션

```javascript
// Mixamo 애니메이션 5종 로드
const enemyAnimFiles = {
  idle:    'Zombie Idle.fbx',
  seizure: 'Laying Seizure.fbx',
  punch:   'Zombie Punching.fbx',
  scream:  'Zombie Scream.fbx',
  run:     'Zombie Run.fbx',
};

// root motion 제거 (위치 변화 애니메이션 키 초기값으로 고정)
function stripRootHorizontalMotion(clip) {
  const track = clip.tracks.find(t => t.name === `${MIXAMO_BONE_PREFIX}Hips.position`);
  if (!track) return clip;
  const v = track.values;
  const baseX = v[0]; const baseZ = v[2];
  for (let i = 0; i < v.length; i += 3) {
    v[i] = baseX; v[i + 2] = baseZ;  // X, Z 고정
  }
  return clip;
}

// AnimationMixer로 애니메이션 전환 (0.2초 crossfade)
_playAction(name) {
  const next = this.actions[name];
  if (this.currentAction === next) return;
  next.reset().fadeIn(0.2).play();
  if (this.currentAction) this.currentAction.fadeOut(0.2);
  this.currentAction = next;
}
```

![FBX 스켈레탈 애니메이션 — run / seizure / punch](screenshots/fbx_animations.png)
*그림 5.5 — 좌: run(alert 추격), 중: seizure(wary 발작), 우: punch(근접 공격) 애니메이션. crossfade 0.2초로 부드럽게 전환된다.*

### 5.6 wary 상태 포즈 보정 (Seizure Stand Fix)

`seizure` 애니메이션은 바닥에 눕는 모션이라 서 있는 상태로 보정이 필요하다.

```javascript
const SEIZURE_STAND_FIX_X = Math.PI / 2;      // 90도 회전으로 일으켜 세움
const SEIZURE_STAND_FIX_Y = ENEMY_HEIGHT * 0.55;  // Y 오프셋
const SEIZURE_STAND_FIX_Z = -ENEMY_HEIGHT * 0.6;  // Z 오프셋

_updateBodyPoseLerp(dt) {
  const inSeizure = this.state === 'wary';
  const targetRotX = inSeizure ? SEIZURE_STAND_FIX_X : 0;
  const t = Math.min(1, dt * 6);  // dt * 6: 약 0.17초에 보간 완료
  this.bodyModel.rotation.x += (targetRotX - this.bodyModel.rotation.x) * t;
  this.bodyModel.position.y += (targetPosY - this.bodyModel.position.y) * t;
  this.bodyModel.position.z += (targetPosZ - this.bodyModel.position.z) * t;
}
```

![Seizure 포즈 보정 — wary 상태](screenshots/seizure_pose_fix.png)
*그림 5.6 — bodyModel을 X축 90도 회전 후 Y/Z 오프셋 적용으로 바닥 발작 애니메이션을 서 있는 상태로 보정한 결과.*

### 5.7 플레이어 충돌 & 이동

```javascript
// AABB 충돌 감지
const PLAYER_RADIUS = 0.35;
function blockedAt(x, z, radius = PLAYER_RADIUS) {
  for (const w of walls) {
    const { hx, hz } = w.userData;
    if (Math.abs(x - w.position.x) < hx + radius &&
        Math.abs(z - w.position.z) < hz + radius) return true;
  }
  return false;
}

// X, Z 독립 이동으로 벽 미끄러짐 구현
const nextX = p.x + velocity.x;
const nextZ = p.z + velocity.z;
if (!blockedAt(nextX, p.z)) p.x = nextX;  // X 방향 단독 체크
if (!blockedAt(p.x, nextZ)) p.z = nextZ;  // Z 방향 단독 체크

// 앉기 (C키): 카메라 높이 lerp
const targetHeight = keys['KeyC'] ? CROUCH_HEIGHT : STAND_HEIGHT;  // 0.96 vs 1.6
p.y += (targetHeight - p.y) * Math.min(1, dt * 8);
```

![플레이어 이동 — 벽 충돌 미끄러짐](screenshots/player_movement.png)
*그림 5.7 — 벽 모서리에서 X, Z 독립 충돌 체크로 미끄러짐(sliding)이 구현된 장면.*

---

## 6. 렌더링 파이프라인 전체 흐름

### 6.1 메인 루프 순서 (Section 12)

```javascript
function animate() {
  requestAnimationFrame(animate);
  const dt = Math.min(clock.getDelta(), 0.05);  // dt 상한 0.05s (20fps 이하 보호)
  const elapsed = clock.getElapsedTime();

  // 1. BPM 업데이트
  BPM.update(dt);
  screamFlash = Math.max(0, screamFlash - dt / SCREAM_FLASH_DURATION);

  // 2. BPM → 씬 시각 상태 적용 (GI, 조명 강도, 배경색)
  applyVisualState(norm, stage);
  updateHUD(stage);

  // 3. 플레이어 이동 & 발소리 파동
  movePlayer(dt);

  // 4. 적 AI 업데이트
  enemies.forEach(e => e.update(dt, playerPos, elapsed, moving));

  // 5. 심박 타이밍 → 파동 & 빛 flash
  if (elapsed - lastBeat > getBeatInterval() * 2) {
    lastBeat = elapsed;
    triggerHeartbeatGlow();
  }
  heartbeatLight.intensity = hbEnvelope * 14;

  // 6. 소나 파동 업데이트 (확산, 반사, shader uniform 갱신)
  updateRays(dt);

  // 7. 렌더
  renderer.render(scene, camera);
}
```

### 6.2 전체 시스템 상호작용 다이어그램

```
플레이어 입력(WASD/Shift/C)
        │
        ▼
   BPM.update()
        │
   ┌────┴────┐
   ▼         ▼
giIntensity  발소리 파동 크기
(GI 세기)    (spawnPlayerFootstepPulse)
   │         │
   ▼         ▼
LightProbe  activePulses[]
ambientLight      │
pointLights       ▼
   │      적 _checkPulseTouch()
   │              │
   │         detection 상승
   │              │
   │         awakened 전환
   │              │
   └──────────────▼
            screamFlash = 1
                  │
            giIntensity → 1.8 (즉시 최대)
            플레이어 BPM → 160
```

![전체 렌더링 흐름 — PANIC + screamFlash 동시 발생](screenshots/full_pipeline_panic.png)
*그림 6.1 — 적이 awakened 전환하는 순간: screamFlash로 giIntensity가 1.8에 도달하고, BPM이 즉시 160으로 상승하며 씬 전체가 일시적으로 밝아진다.*

---

## 참고 문헌

- Majercik, A., Müller, J., Shirley, P., McGuire, M. (2019). *Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields*. Journal of Computer Graphics Techniques.
- McGuire, M., Mara, M., Nowrouzezahrai, D., Luebke, D. (2017). *Real-Time Global Illumination using Precomputed Light Field Probes*. ACM I3D.
- Pharr, M., Jakob, W., Humphreys, G. (2016). *Physically Based Rendering: From Theory to Implementation*. 3rd ed. Morgan Kaufmann.
- Three.js Documentation — LightProbe, ShaderMaterial, AnimationMixer, PointerLockControls, PCFSoftShadowMap.

---

*모든 캡처 이미지는 본 게임(PULSE, index.html)을 직접 구동하여 촬영한 화면입니다.*  
*스크린샷은 `screenshots/` 폴더에 포함되어 있습니다.*
