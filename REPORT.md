# PULSE — 최종 과제 리포트

> **학번**: 2021204085  
> **이름**: 이서환  
> **게임명**: PULSE  
> **장르**: 1인칭 스텔스 생존 호러 (WebGL / Three.js)

---

## 1. 게임 개요

PULSE는 플레이어의 **심박수(BPM)** 가 게임의 모든 시각·청각 요소를 제어하는 1인칭 호러 게임이다. 완전한 어둠 속에서 소나(Sonar) 파동이 유일한 시야를 제공하며, 플레이어의 행동(이동, 달리기, 살금살금)이 BPM을 변화시키고, 그 BPM이 GI(Global Illumination) 강도와 적의 감지 능력을 실시간으로 바꾼다.

### 1.1 조작 방법

| 키 | 동작 |
|---|---|
| WASD | 이동 |
| 마우스 | 시점 전환 (Pointer Lock) |
| Shift | 달리기 (BPM ↑↑) |
| C | 살금살금 이동 (BPM ↓, 소나 파동 축소) |
| 클릭 | 근접 공격 (적 처치) |

### 1.2 게임 목표

맵 내 6개 구역에 배치된 적을 모두 처치하거나, 발각되지 않고 생존한다. 적에게 붙잡히면 **YOU DIED** 화면이 표시된다.

---

## 2. 강의 내용과 구현 내용 매핑

### 2.1 DDGI (Dynamic Diffuse Global Illumination) 시뮬레이션

강의에서 다룬 DDGI는 씬 내 Irradiance Probe를 격자(Grid)로 배치하고, 각 프로브가 주변 빛을 캡처하여 동적으로 간접광을 전달하는 기법이다.

본 게임에서는 `THREE.LightProbe` 를 3×1×3 격자로 배치하고, BPM 상태(calm/tense/panic)에 따라 프로브 강도를 실시간으로 변조하여 DDGI의 핵심 원리인 **동적 간접광 변화**를 구현하였다.

```javascript
// probeGrid: 3×1×3, probeSpacing: 16 unit
for (let ix = 0; ix < probeGrid.x; ix++) {
  for (let iy = 0; iy < probeGrid.y; iy++) {
    for (let iz = 0; iz < probeGrid.z; iz++) {
      const probe = new THREE.LightProbe();
      probe.position.set(
        (ix - 1) * probeSpacing,
        1 + iy * 2,
        (iz - 1) * probeSpacing
      );
      probe.sh.coefficients[0].set(0.05, 0.04, 0.06); // SH 계수 설정
      probe.intensity = 0.0;
      scene.add(probe);
      lightProbes.push(probe);
    }
  }
}
```

> **[게임 캡처 이미지 삽입 위치]**  
> *calm 상태 → 완전한 암흑 / panic 상태 → GI 활성화로 벽면이 붉게 물드는 화면*

---

### 2.2 BPM 기반 GI 강도 동적 제어

강의에서 배운 GI의 핵심은 씬 전반에 걸친 **간접광의 실시간 변화**다. 본 게임은 BPM을 정규화(normalize)하여 `AmbientLight`, `LightProbe`, `PointLight`, `DirectionalLight` 네 종류의 광원 강도를 동시에 제어한다.

```javascript
function applyVisualState(bpmNorm, stage) {
  let giIntensity;
  if (stage === 'calm')       giIntensity = 0.0;
  else if (stage === 'tense') giIntensity = THREE.MathUtils.lerp(0.03, 0.4,  (BPM.current - 90)  / 40);
  else                        giIntensity = THREE.MathUtils.lerp(0.4,  1.8,  (BPM.current - 130) / 30);

  ambientLight.intensity = giIntensity;
  lightProbes.forEach(probe => { probe.intensity = giIntensity * 0.4; });

  let plIntensity;
  if (stage === 'calm')       plIntensity = 0;
  else if (stage === 'tense') plIntensity = THREE.MathUtils.lerp(0, 2, (BPM.current - 90)  / 40);
  else                        plIntensity = THREE.MathUtils.lerp(2, 6, (BPM.current - 130) / 30);
  pointLights.forEach(pl => { pl.intensity = plIntensity; });
}
```

| BPM 구간 | 스테이지 | AmbientLight | LightProbe | PointLight | DirectionalLight |
|---|---|---|---|---|---|
| < 90 | calm | 0.0 | 0.0 | 0 | 0.0 |
| 90–130 | tense | 0.03 ~ 0.4 | ×0.4 | 0 ~ 2 | 0.0 ~ 0.4 |
| 130–160 | panic | 0.4 ~ 1.8 | ×0.4 | 2 ~ 6 | 0.4 ~ 1.0 |

> **[게임 캡처 이미지 삽입 위치]**  
> *tense 구간에서 벽과 바닥에 은은한 파란 간접광이 들어오는 모습 / panic 구간에서 주황빛으로 가득 찬 씬 캡처*

---

### 2.3 Sonar 파동 — 커스텀 GLSL ShaderMaterial

강의에서 다룬 셰이더 프로그래밍 원리를 적용하여, 바닥·벽 오버레이 메쉬에 커스텀 셰이더를 적용하였다. Vertex Shader는 각 점의 월드 좌표를 Fragment Shader로 전달하고, Fragment Shader는 최대 16개의 파동 중심(`uCenters`)으로부터의 거리를 계산하여 `smoothstep`으로 파동 링을 렌더링한다.

**Vertex Shader**

```glsl
varying vec3 vWorldPos;
void main() {
  vec4 wp   = modelMatrix * vec4(position, 1.0);
  vWorldPos = wp.xyz;
  gl_Position = projectionMatrix * viewMatrix * wp;
}
```

**Fragment Shader**

```glsl
void main() {
  vec3  col   = vec3(0.0);
  float total = 0.0;
  for (int i = 0; i < 16; i++) {
    float r    = uRadii[i];
    if (r < 0.0) continue;
    float dist = length(vWorldPos.xz - uCenters[i].xz);
    float wave = 1.0 - smoothstep(0.0, uWaveWidth, abs(dist - r));
    float atten = 1.0 / (1.0 + r * r * uAttenK);
    col   += uColors[i] * wave * atten;
    total += wave * atten;
  }
  if (total < 0.005) discard;
  gl_FragColor = vec4(col, min(total * 1.2, 1.0));
}
```

파동은 플레이어 발소리, 적의 심박 2가지 원인으로 생성되며, 반사파(reflect pulse)는 벽 충돌 지점에서 2차 파동을 생성하여 씬의 형태를 간접적으로 드러낸다.

> **[게임 캡처 이미지 삽입 위치]**  
> *플레이어가 달릴 때 바닥에 초록빛 소나 파동이 퍼지는 모습 / 벽에서 반사파가 생성되는 모습*

---

### 2.4 스킨드 메쉬에 소나 셰이더 적용 (enemyGhostMat)

강의에서 배운 스키닝(Skinning) 개념을 활용하여, FBX 캐릭터 모델의 본(bone) 애니메이션을 처리하는 Vertex Shader에 소나 파동 감지 로직을 결합하였다. 적은 기본적으로 반투명한 붉은빛 실루엣으로만 존재하다가, 소나 파동이 닿는 순간에만 파동 색상으로 밝게 드러난다.

```glsl
// enemyGhostVert — 스키닝 포함
#include <skinning_pars_vertex>
varying vec3 vWorldPos;
void main() {
  #include <skinbase_vertex>
  #include <begin_vertex>
  #include <skinning_vertex>
  vec4 wp   = modelMatrix * vec4(transformed, 1.0);
  vWorldPos = wp.xyz;
  gl_Position = projectionMatrix * viewMatrix * wp;
}
```

> **[게임 캡처 이미지 삽입 위치]**  
> *소나 파동이 적을 통과하는 순간 적의 실루엣이 밝게 드러나는 화면*

---

## 3. 개발 상세 설명

### 3.1 맵 구성

맵은 `BoxGeometry` 기반 벽(`addWall`)과 박스(`addCrate`)를 절차적으로 배치하여 구성하였다. 전체 크기는 36×48 유닛이며, 중앙 복도와 좌우 6개 구역(bay)으로 나뉜다.

```
┌──────────────────────────┐
│  Bay(-10,16)  Bay(10,16) │
│                          │
│  Bay(-10, 0)  Bay(10, 0) │ ← 중앙 복도 (CORR_HALF=2.4)
│                          │
│  Bay(-10,-16) Bay(10,-16)│
└──────────────────────────┘
```

각 bay에는 상자(crate) 2개가 배치되어 플레이어가 숨거나 적을 유인하는 데 활용된다.

> **[게임 캡처 이미지 삽입 위치]**  
> *소나 파동으로 드러나는 맵 전체 구조 캡처*

---

### 3.2 심박수(BPM) 시스템

BPM은 게임의 핵심 상태 변수다. 플레이어 행동에 따라 `target` BPM이 변하고, 실제 `current` BPM이 지수적으로 추종한다.

```javascript
const BPM = {
  current: 72, target: 72, min: 60, max: 160,
  update(dt) {
    this.current += (this.target - this.current) * dt * 1.2;
    if (running)       this.target = Math.min(this.max, this.target + dt * 30);
    else if (sneaking) this.target = Math.max(72,  this.target - dt * 2);
    else if (moving)   // 100 BPM 수렴
    else               this.target = Math.max(72,  this.target - dt * 3);
  }
};
```

BPM에 따라 결정되는 스테이지(calm/tense/panic)는 GI 강도, HUD 색상, 소나 파동 크기, 심박 사운드 볼륨에 모두 연동된다.

> **[게임 캡처 이미지 삽입 위치]**  
> *좌상단 HUD에 심박수(BPM)와 스테이지가 표시되는 화면. calm(초록), tense(파랑), panic(주황) 각 상태 캡처*

---

### 3.3 적(Enemy) AI 상태머신

적은 6가지 상태를 가지는 유한 상태머신(FSM)으로 동작한다.

```
dormant ──(타이머)──▶ searching ──(타이머)──▶ wary
    ▲                                           │
    │            감지(detection ≥ 100)           │
    │◀──(ALERT_LOSE_TIMEOUT 초과)── alert ◀─── awakened
```

| 상태 | 설명 | 글로우 색상 |
|---|---|---|
| dormant | 휴면. 파동·소리 무반응 | 없음 |
| searching | 주변 감시. 심박 파동 방출(1.5초 주기) | 주황 (약함) |
| wary | 경계. 심박 파동 방출(0.5초 주기) | 주황 (강함) |
| awakened | 인식 직후 비명. BPM을 MAX로 강제 설정 | 빨강 |
| alert | 플레이어 추적 및 공격 | 없음 (빨강 머티리얼) |

**감염(infect) 전파**: 한 적이 `awakened` 상태가 되면 반경 8 유닛 내 다른 적도 연쇄 각성한다.

> **[게임 캡처 이미지 삽입 위치]**  
> *적이 searching 상태에서 주황빛 심박 파동을 방출하는 모습 / alert 상태에서 빨간 머티리얼로 변한 적이 플레이어를 추적하는 모습*

---

### 3.4 소나 파동 종류 및 반사

| 종류 | 색상 | 최대 거리 | 속도 | 특징 |
|---|---|---|---|---|
| 플레이어 발소리 | calm/tense/panic 단계색 | 9~15 | 5~14 | 살금살금 시 0.2배 축소 |
| 플레이어 심박 | 단계색 | - | - | 2박마다 카메라 위치에서 발생 |
| 적 심박 | 빨강(0xdd2222) | 2.7~4.7 | 3~7 | 플레이어 탐지에 사용됨 |
| 반사파 | 원본과 동일 | 원본 ×0.4 | 원본 ×0.8 | 벽 충돌 시 생성 |
| 처치 이펙트 | 흰색(0xffffff) | 10 | 10 | 적 처치 시 생성 |

> **[게임 캡처 이미지 삽입 위치]**  
> *벽에서 반사파가 갈라지는 모습 / 적 처치 시 흰색 파동이 퍼지는 모습*

---

### 3.5 심박 광원 시스템 (Heartbeat Glow)

플레이어의 심박에 맞춰 카메라 위치에 `PointLight`가 짧은 attack(0.03초) / 긴 release(0.7초) 엔벨로프로 점멸한다. 이는 강의에서 다룬 **Light Probe 업데이트 주기 최적화**와 같은 맥락으로, 매 프레임 GI 전체를 갱신하지 않고 국소 광원으로 심박감을 표현한다.

```javascript
const hbEnvelope = heartbeatElapsed < HEARTBEAT_ATTACK
  ? heartbeatElapsed / HEARTBEAT_ATTACK
  : Math.exp(-(heartbeatElapsed - HEARTBEAT_ATTACK) / HEARTBEAT_RELEASE);
heartbeatLight.intensity = hbEnvelope * 14;
```

> **[게임 캡처 이미지 삽입 위치]**  
> *심박 타이밍에 맞춰 플레이어 주변 바닥이 순간적으로 밝아지는 모습*

---

### 3.6 FBX 애니메이션 및 스켈레탈 처리

Mixamo에서 다운로드한 FBX 파일을 `FBXLoader`로 불러오고, 본(bone) 이름 프리픽스를 정규화(`mixamorig` → `mixamorig5`)하여 AnimationMixer에 등록하였다. 루트 모션의 수평 이동 성분은 제거(`stripRootHorizontalMotion`)하여 코드로 직접 위치를 제어한다.

사용 애니메이션:

| 상태 | 파일 |
|---|---|
| idle | Zombie Idle.fbx |
| wary | Laying Seizure.fbx |
| alert | Zombie Run.fbx |
| attack | Zombie Punching.fbx |
| awakened | Zombie Scream.fbx |

> **[게임 캡처 이미지 삽입 위치]**  
> *wary 상태에서 경련(seizure) 모션을 취하는 적 / alert 상태에서 달려오는 적 캡처*

---

### 3.7 충돌 처리

벽과 박스는 모두 AABB(Axis-Aligned Bounding Box) 방식으로 충돌을 처리한다. 플레이어(반경 0.35)와 적 모두 x, z 축을 독립적으로 검사하여 코너에서 미끄러지는 sliding 이동을 지원한다.

```javascript
function blockedAt(x, z, radius = PLAYER_RADIUS) {
  for (const w of walls) {
    const { hx, hz } = w.userData;
    if (Math.abs(x - w.position.x) < hx + radius &&
        Math.abs(z - w.position.z) < hz + radius) return true;
  }
  return false;
}

// x, z 독립 검사 → sliding
if (!blockedAt(nextX, p.z)) p.x = nextX;
if (!blockedAt(p.x, nextZ)) p.z = nextZ;
```

> **[게임 캡처 이미지 삽입 위치]**  
> *플레이어가 벽 모서리를 따라 이동하는 모습*

---

### 3.8 공간 음향 (Positional Audio)

Three.js `PositionalAudio`를 사용하여 적의 심박 소리가 3D 공간에서 거리에 따라 감쇠한다. `setDistanceModel('linear')`, `setRefDistance(1.5)`, `setMaxDistance(8)` 설정으로 8 유닛 이내에서만 들리도록 하여 소나 파동의 시각적 감지 범위와 청각적 감지 범위를 연동하였다.

> **[게임 캡처 이미지 삽입 위치]**  
> *적 근처에서 심박 사운드 범위를 표현한 화면 (소나 파동 반경과 비교)*

---

## 4. 기술 스택

| 항목 | 내용 |
|---|---|
| 렌더링 | Three.js r160 (WebGL2) |
| 셰이더 | GLSL (커스텀 ShaderMaterial — sonarMat, enemyGhostMat) |
| GI 기법 | LightProbe 3×1×3 그리드 + BPM 연동 동적 강도 제어 (DDGI 시뮬레이션) |
| 그림자 | PCFSoftShadowMap (DirectionalLight, heartbeatLight) |
| 캐릭터 | FBX + Mixamo 스켈레탈 애니메이션 (5종) |
| 음향 | Three.js Audio / PositionalAudio (Web Audio API) |
| 컨트롤 | PointerLockControls |

---

## 5. 구현 중 어려웠던 점

1. **스킨드 메쉬에 커스텀 셰이더 적용**: Three.js의 `skinning` 관련 GLSL include(`#include <skinning_pars_vertex>` 등)를 수동으로 삽입해야 했으며, `ShaderMaterial`에 `skinning: true` 옵션을 명시해야 했다.
2. **소나 Uniform 동기화**: 최대 16개 파동 슬롯을 `Float32Array`와 `Vector3` 배열로 관리하며, dead 처리 시 해당 슬롯의 `uRadii[slot] = -1`로 Fragment Shader에서 건너뛰도록 설계하였다.
3. **FBX 본 이름 불일치**: Mixamo 애니메이션과 기본 모델의 본 프리픽스가 달라 `remapMixamoTrackNames`로 정규화하였다.

---

## 6. 참고

- Three.js 공식 문서: https://threejs.org/docs/
- Mixamo 캐릭터 및 애니메이션: https://www.mixamo.com/
- DDGI 원논문: Majercik et al., "Dynamic Diffuse Global Illumination with Ray-Traced Irradiance Fields", JCGT 2019
