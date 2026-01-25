---
title: "[그래픽스] 길티기어 Xrd 카툰 렌더링 분석: GDC 2015 발표 정리 및 UV 트릭 재현"
slug: "guilty-gear-rendering"
date: 2026-01-25
draft: false
tags:
  - Graphics
  - Rendering
  - Guilty Gear
  - NPR
  - Cel Shading
categories:
  - Computer Graphics
description: "GDC 2015 발표를 기반으로 한 길티기어 Xrd 카툰 렌더링 기법 분석. 셀 셰이딩, 버텍스 컬러 AO, UV 트릭을 Unity에서 직접 재현해봤음."
---

> 이 글은 GDC 2015 발표 "Guilty Gear Xrd's Art Style: The X Factor Between 2D and 3D"와 직접 수행한 분석 및 Unity 재현 결과를 바탕으로 작성했음.

## 1. 서론: 왜 길티기어 Xrd인가

길티기어 Xrd의 GDC 발표는 카툰 렌더링 분야에서 기념비적인 영상으로 평가받는다. 조회수 기준으로도 GDC 발표 중 상위권이며, 수많은 NPR(Non-Photorealistic Rendering) 개발자들이 이 발표를 레퍼런스로 삼고 있다.

길티기어 Xrd는 3D 게임임에도 불구하고 2D 셀 애니메이션과 구분이 어려울 정도의 비주얼을 구현했음. 본 글에서는 이를 가능하게 한 핵심 기법들을 분석한다.

**핵심 포인트**
- 기술적 구현은 프로그래머의 영역
- 최종 표현의 결정권은 아트 직군(원화가, 모델러)에게 있음
- 프로그래머-아티스트 협업이 필수

---

## 2. 캐릭터 메시 구성

![캐릭터 메시 구성](images/posts/guilty-gear-rendering/character-meshes.png)
*GDC 2015 슬라이드: Character Meshes*

### 하이폴리 메시

길티기어 캐릭터는 약 40,000개의 삼각형으로 구성된 하이폴리 메시를 사용한다. 일반적인 게임 캐릭터보다 높은 폴리곤 수를 사용하는 이유는 다음과 같다:

- 초근접 촬영(클로즈업)에서도 실루엣 품질 유지
- 곡선 표현을 위한 충분한 정점 확보
- UV 트릭을 위한 메시 구조 설계

### 텍스처 의존도 최소화

슬라이드에서 강조하는 "Less texture dependent"는 길티기어 렌더링의 핵심 철학이다:

- 텍스처는 단순한 색상 팔레트 수준으로만 사용
- 세부 표현은 정점 속성(Vertex Attribute)에 저장
- 선형 보간(Linear Interpolation)을 활용하여 해상도 무관한 품질 확보

![색상 팔레트 텍스처](images/posts/guilty-gear-rendering/color-palette-texture.png)
*GDC 2015 슬라이드: 실제 텍스처 예시*

위 슬라이드를 보면 알 수 있듯이, **길티기어의 텍스처에는 옷의 주름이나 디테일이 전혀 그려져 있지 않다.** 그냥 단색 영역들의 집합일 뿐이다. 일반적인 게임에서 텍스처에 세밀하게 그려넣는 것과는 완전히 다른 접근 방식임.

이게 가능한 이유는 모든 디테일을 텍스처가 아닌 다른 방식으로 표현하기 때문이다:
- 명암 → 셀 셰이딩 (셰이더)
- AO → 버텍스 컬러 (정점 속성)
- 내부 라인 → UV 트릭 (정점 속성)
- 외곽선 → 인버티드 헐 (별도 메시)

**텍스처는 오직 "이 영역은 무슨 색인가"만 알려주는 색상 팔레트 역할만 한다.**

---

## 3. 셀 셰이딩: 명암의 이진화

![셀 셰이딩](images/posts/guilty-gear-rendering/cel-shading.png)
*GDC 2015 슬라이드: Getting the Shading right*

### 일반 셰이딩 vs 셀 셰이딩

일반적인 PBR 셰이딩은 법선(Normal) 벡터와 광원 방향에 따라 그라데이션으로 명암을 표현한다. 그러나 셀 셰이딩은 "밝음/어두움"의 이진화된 명암을 요구한다.

```
일반 셰이딩:  ████▓▓▓▓▒▒▒▒░░░░     (그라데이션)
셀 셰이딩:   ████████████          (이진화)
```

### 구현의 어려움

슬라이드에서 언급하는 "Cel-shading is hard to get right"의 이유:

- **It's either Lit or Not** - 중간이 없음
- **Easily ruined by the smallest inconsistency** - 작은 불일치도 눈에 띔
- **Hard to convince 2D with out complete control** - 완전한 제어 없이는 2D 느낌을 내기 어려움

### 임계값(Threshold) 기반 구현

셰이더에서 특정 임계값을 기준으로 색상을 이진화한다:

```hlsl
float NdotL = dot(normal, lightDir);

if (NdotL > _Threshold)
    return _BaseColor;      // 밝은 면
else
    return _ShadowColor;    // 어두운 면
```

---

## 4. 버텍스 컬러를 이용한 AO 제어

![버텍스 컬러 AO](images/posts/guilty-gear-rendering/vertex-color-ao.png)
*GDC 2015 슬라이드: Controlling the Threshold*

### 텍스처 AO의 한계

기존 방식인 텍스처 기반 AO(Ambient Occlusion)는 다음과 같은 문제가 있다:

- 해상도에 의존적 (줌인 시 품질 저하)
- 수정하려면 텍스처를 다시 베이크해야 함
- 메모리 사용량 증가

### 버텍스 컬러 AO

길티기어는 버텍스 컬러를 AO 제어에 활용했음:

- **A channel of Vertex Color used as a offset to the Threshold**
- **Acts as the occlusion of the vertex**
- **Every vertex occlusion setting set so the artist can control results**

정점 속성은 GPU에 의해 선형 보간되므로 해상도와 무관하게 부드러운 결과를 얻을 수 있다. 이는 벡터 그래픽과 유사한 원리이다.

### 아티스트 워크플로우

1. 프로그래머가 버텍스 컬러를 AO로 해석하는 셰이더 구현
2. 아티스트에게 버텍스 페인팅 도구 제공
3. 아티스트가 실시간으로(WYSIWYG) AO를 조정하며 원하는 룩 완성

---

## 5. 내부 라인(Inner Line): UV 트릭

![내부 라인](images/posts/guilty-gear-rendering/inner-line.png)
*GDC 2015 슬라이드: Character's Inner Line*

### 문제 정의

외곽선은 인버티드 헐(Inverted Hull) 방식으로 해결할 수 있지만, 내부 라인(주름, 근육선 등)은 다른 접근이 필요하다. 텍스처에 선을 그리면 줌인 시 계단 현상(Aliasing)이 발생한다.

### Axis Aligned Beams

슬라이드의 핵심 내용:

- **All lines are Axis Aligned beams** - 모든 선은 축 정렬 빔
- **No freehand curves** - 자유 곡선 없음
- **UVs are lined up with these beams** - UV가 빔과 정렬
- **UV overlapping the beam = width** - UV 겹침 = 선 두께

텍스처에는 축 정렬된 직선만 저장하고, UV를 조작하여 곡선처럼 보이게 한다.

```
텍스처 (단순):          메시 UV (곡선):          결과:
┌──────────┐           곡선 형태로              곡선 라인!
│──────────│    +      UV 매핑         =
└──────────┘
```

### 왜 계단 현상이 없는가?

- 텍스처의 선은 축에 정렬되어 있음
- 화면의 픽셀도 축에 정렬되어 있음
- 샘플링 시 항상 깔끔한 결과
- 곡선은 메시 형태로 표현되므로 텍스처 해상도와 무관

---

## 6. UV 트릭의 수학적 원리

### 무게중심 좌표(Barycentric Coordinates)

GPU가 삼각형 내부를 래스터라이즈할 때, 무게중심 좌표를 사용하여 정점 속성을 보간한다.

삼각형의 세 정점을 $A$, $B$, $C$라 할 때, 삼각형 내부의 임의의 점 $P$는 다음과 같이 표현된다:

$$P = uA + vB + wC$$
$$(단, u + v + w = 1)$$

### UV 트릭에서의 활용

1. 정점 $A$에 UV 값 `0`, 정점 $B$에 UV 값 `1`을 할당
2. GPU가 삼각형 내부에서 UV 값을 `0`에서 `1`로 보간
3. UV 값이 정확히 `0.5`가 되는 등고선 형성
4. 셰이더가 해당 등고선 위치에 라인 렌더링

```
        v0 (UV=1)
           ●
          /|\
         / | \
        /  |  \
       /   |   \
      / ───●─── \  ← UV = 0.5 (라인!)
     /     |     \
    ●─────────────●
  v1 (UV=0)     v2 (UV=0)
```

이 라인은 이미지 데이터가 아닌 수학적 계산의 결과물이므로, 확대 시에도 품질 저하가 발생하지 않는다.

---

## 7. Unity에서의 재현

본 기법을 Unity에서 직접 재현해봤음.

> **주의**: 아래 구현은 GDC 발표 영상의 "Axis Aligned Beams" 슬라이드를 보고 **내가 추측한 방식**이다. 실제 길티기어 Xrd에서는 이와 다른 방식으로 구현했을 수도 있음. 발표에서는 개념만 설명했을 뿐 구체적인 셰이더 코드나 메시 구조는 공개하지 않았기 때문에, 이 재현은 "이런 식으로 구현할 수 있겠다"는 하나의 해석으로 봐주길 바람.

![UV 트릭 재현](images/posts/guilty-gear-rendering/analysis-diagram.png)
*Unity에서 UV 트릭의 원리를 시연한 데모*

위 이미지는 Unity에서 UV 트릭의 원리를 시연한 결과다. 실제 길티기어는 축 정렬된 직선이 그려진 텍스처를 사용했지만, 이 예제에서는 텍스처 없이 UV 보간만으로 동일한 원리를 구현했음.

상단 정점에 UV=1, 하단에 UV=0을 할당하고, 셰이더에서 UV=0.5인 지점을 라인으로 렌더링하는 방식이다. 여기서 UV=0.5 라인은 특정 점이 아니라 **등고선(isoline)**으로, 삼각형의 밑변과 평행하게 형성됨. 메시가 곡선 형태이므로 이 등고선도 자연스럽게 곡선이 된다.

오른쪽의 시각화 데모에서는 정점(빨간/파란 점), UV2 값, 삼각형 구조까지 확인할 수 있다.

### 셰이더 구현

```hlsl
half4 frag(Varyings IN) : SV_Target
{
    // GPU가 무게중심 좌표로 보간한 UV2 값
    float uvY = IN.uv2.y;

    // 라인 두께 설정
    half halfThickness = _LineThickness * 0.5;

    // UV2.y가 0.5 근처인지 확인
    half lineMask = step(0.5 - halfThickness, uvY)
                  * step(uvY, 0.5 + halfThickness);

    // 최종 색상
    return lerp(_BaseColor, _LineColor, lineMask);
}
```

### 메시 생성 (C#)

```csharp
// 상단 정점: UV2.y = 1 (라인 없음)
uv2[topIdx] = new Vector2(0f, 1f);

// 하단 정점: UV2.y = 0 (라인 없음)
uv2[bottomIdx] = new Vector2(0f, 0f);

// GPU 보간 → 중간에서 UV2.y = 0.5 → 라인!
```

### 삼각형과 보간

여러 삼각형이 정점을 공유할 때, UV도 공유된다. 이로 인해 삼각형 경계에서 라인이 자연스럽게 연결된다.

```
T0(0,2,1)              T1(1,2,3)
v0 ●───────● v2        ● v2 ───────● v3
    \     /|           |\     /
     \   / |           | \   /
      \ /  |           |  \ /
       ●───┼───────────┼───●
      v1   └── 공유 ──┘

→ v1-v2 엣지에서 UV가 연속 → 라인도 연속!
```

---

## 8. 추가 기법들

### 캐릭터별 개별 라이팅

길티기어는 캐릭터마다 별도의 라이트 방향을 설정했음. 각 캐릭터가 가장 "예쁘게" 보이는 라이트 각도를 개별 지정하여, 월드 라이트와 분리하여 적용한다.

### 얼굴 노멀 수정

얼굴의 노멀은 실제 지오메트리 형태에 따라 자동 계산되지만, 이는 종종 원치 않는 그림자를 만든다. 길티기어는 노멀 벡터를 수동으로 수정하여 원하는 라이팅 결과를 얻었다.

현대적 대안으로는 원신(Genshin Impact)에서 대중화된 SDF(Signed Distance Field) 맵 방식이 있다.

### 외곽선 두께 제어

인버티드 헐 방식의 외곽선에서, 버텍스 컬러를 통해 두께를 정점마다 다르게 설정할 수 있다. 이를 통해 원화의 "선 강약" 표현이 가능해진다.

---

## 9. 결론

길티기어 Xrd의 카툰 렌더링이 성공적인 이유는 고해상도 텍스처가 아닌, 다음과 같은 접근에 있다:

1. **텍스처 의존도 최소화** - 색상 팔레트 수준으로만 사용
2. **정점 속성의 적극 활용** - 버텍스 컬러, UV, 노멀
3. **아티스트 제어권 확보** - 실시간 수정 가능한 도구 제공
4. **수학적 접근** - 정점 속성의 선형 보간, 해상도 무관

2D 스타일을 구현하기 위해 3D 그래픽스의 수학적 원리를 역이용한 점이 이 기법의 핵심이다.

---

## 10. 다음 글 예고

이번 글에서는 길티기어 Xrd의 렌더링 기법을 이론적으로 분석하고, UV 트릭의 개념을 Unity에서 간단히 재현해봤음.

다음 글에서는 이번에 분석한 이론적 토대를 바탕으로, **Unity에서 길티기어 스타일 셀 셰이딩을 최대한 비슷하게 구현**해볼 예정이다. 셀 셰이딩을 직접 만들어보면서 이론이 실제로 어떻게 적용되는지 확인해볼 계획임.

---

### 독자 연구 안내

이 글은 GDC 2015 발표 영상을 바탕으로 한 **독자 연구**입니다. 공식 자료가 아니기 때문에 틀린 내용이 있을 수 있음. 특히 Unity 재현 부분은 발표 내용을 보고 추측한 구현 방식이므로, 실제 길티기어의 내부 구현과는 다를 수 있습니다.

잘못된 내용이나 보충할 부분이 있다면 **댓글로 피드백 부탁드립니다.** 함께 더 정확한 정보를 만들어가면 좋겠습니다.

---

### 참고 자료
- [GDC 2015: Guilty Gear Xrd's Art Style](https://www.youtube.com/watch?v=yhGjCzxJV3E)
- [GDC Vault PDF](https://www.gdcvault.com/play/1022031/GuiltyGearXrd-s-Art-Style-The)
- 직접 분석 및 Unity 재현 자료
