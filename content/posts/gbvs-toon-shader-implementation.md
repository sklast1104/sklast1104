---
title: "[그래픽스] GBVS/길티기어 스타일 툰 셰이더 구현: Unity URP에서 단계별 분석"
slug: "gbvs-toon-shader-implementation"
date: 2026-02-01
draft: false
tags:
  - Graphics
  - Rendering
  - GBVS
  - Guilty Gear
  - NPR
  - Cel Shading
  - Unity
  - URP
categories:
  - Computer Graphics
description: "Arc System Works 스타일 툰 셰이더를 Unity URP에서 직접 구현하면서 각 단계별 효과를 시각화함. ILM 텍스처, 버텍스 컬러, 내부 라인, 스페큘러까지."
---

> 이 글은 [이전 글](/posts/guilty-gear-rendering)에서 분석한 GDC 2015 발표 내용을 바탕으로, Unity URP에서 직접 구현한 결과물을 단계별로 보여줌.

## 1. 서론: 왜 단계별 분석인가

길티기어/GBVS 스타일 셰이더는 여러 레이어가 합쳐져서 최종 결과물이 나온다. 한 번에 완성된 셰이더를 보면 각 요소가 어떤 역할을 하는지 이해하기 어렵기 때문에, 이번 글에서는 **각 단계를 하나씩 추가하면서** 시각적으로 어떤 변화가 생기는지 보여줄 예정임.

사용한 모델: **GBVS Charlotta**

---

## 2. 셰이더 프로퍼티 구조

먼저 전체 셰이더의 프로퍼티 구조를 보자:

```hlsl
Properties
{
    _BaseMap("Base Map", 2D) = "white" {}
    _SSSMap("SSS Map", 2D) = "black" {}
    _ILM("ILM Map", 2D) = "gray" {}
    _DetailMap("Detail Map", 2D) = "white" {}

    [Header(Toon Diffuse)]
    _ToonThresHold("Toon ThresHold", Range(0, 1)) = 0.5
    _ToonHardness("Toon Hardness", Float) = 20.0

    [Header(Specular)]
    _SpecSize("Spec Size", Range(0, 1)) = 0.1
    _SpecIntensity("Spec Intensity", Range(0, 5)) = 1.0
    _SpecColor("Spec Color", Color) = (1, 1, 1, 1)

    [Header(Outline)]
    _OutlineWidth("Outline Width", Range(0, 10)) = 1.0
    _OutlineColor("Outline Color", Color) = (0, 0, 0, 1)

    [Header(Rim Light)]
    _RimLightDir("Rim Light Direction", Vector) = (1, 0, -1, 0)
    _RimLightColor("Rim Light Color", Color) = (1, 1, 1, 1)
}
```

4개의 텍스처와 각종 파라미터들. 핵심은 **ILM 텍스처**다.

---

## 3. 텍스처 시스템 이해

Arc System Works 스타일에서 가장 중요한 것은 **ILM(Illumination Map) 텍스처**다. 이 텍스처의 각 채널이 셰이딩을 제어한다.

### Base Map (디퓨즈 컬러)

![Base Map](images/posts/gbvs-toon-shader-implementation/BaseMap.png)
*Base Map: 기본 색상 정보*

베이스 맵은 일반적인 디퓨즈 텍스처와 비슷하지만, **세부 묘사가 거의 없다**는 것이 특징이다. 옷의 주름이나 디테일이 텍스처에 그려져 있지 않음. 순수하게 "이 영역은 무슨 색인가"만 알려주는 색상 팔레트 역할.

### Shadow Map (SSS)

![Shadow SSS](images/posts/gbvs-toon-shader-implementation/Shadow(SSS).png)
*SSS Map: 그림자 영역 색상*

그림자가 질 때 Base Map 대신 사용되는 색상. 보통 Base보다 채도가 높거나 색조가 다르다. "차가운 그림자" 또는 "따뜻한 그림자"를 표현할 수 있음.

---

## 4. ILM 텍스처 채널 분석

ILM 텍스처는 4개의 채널(R, G, B, A)에 각각 다른 정보를 담고 있다. 셰이더에서는 이렇게 샘플링한다:

```hlsl
half4 ilm_map = SAMPLE_TEXTURE2D(_ILM, sampler_linear_repeat, uv1);
float spec_intensity = ilm_map.r;           // R: 스페큘러 강도
float diffuse_control = ilm_map.g * 2.0 - 1.0;  // G: 그림자 임계값 (리매핑)
float spec_size = ilm_map.b;                // B: 스페큘러 크기
float inner_line = ilm_map.a;               // A: 내부 라인 마스크
```

### R 채널 - 스페큘러 강도

![ILM R Channel](images/posts/gbvs-toon-shader-implementation/ILM-R.png)
*ILM R: 스페큘러 강도. 밝을수록 반사광이 강함*

- **밝은 부분**: 금속, 벨트 버클, 광택 있는 옷감
- **어두운 부분**: 무광/매트 재질
- 대부분의 영역이 어둡고, 금속 장식만 밝음

### G 채널 - 그림자 임계값

![ILM G Channel](images/posts/gbvs-toon-shader-implementation/ILM-G.png)
*ILM G: 그림자 임계값. 어두울수록 항상 그림자*

- **밝은 부분**: 그림자가 잘 안 짐 (빛을 잘 받는 곳으로 설정)
- **어두운 부분**: 항상 그림자 처리 (광원과 무관하게 AO 효과)
- 머리카락의 방사형 패턴: 머리카락 결 방향의 하이라이트/그림자 전환 제어

G 채널은 0~1 범위를 -1~1로 리매핑해서 사용한다:
```hlsl
float diffuse_control = ilm_map.g * 2.0 - 1.0;
```

### B 채널 - 스페큘러 크기 / 외곽선

![ILM B Channel](images/posts/gbvs-toon-shader-implementation/ILM-B.png)
*ILM B: 스페큘러 크기 및 외곽선 제어*

- **밝은 부분**: 작은 하이라이트, 얇은 외곽선
- **어두운 부분**: 넓은 하이라이트, 두꺼운 외곽선

### A 채널 - 내부 라인 마스크

![ILM A Channel](images/posts/gbvs-toon-shader-implementation/ILM-A.png)
*ILM A: 내부 라인 마스크. 어두운 부분에 라인 렌더링*

- **밝은 부분 (1.0)**: 라인 없음
- **어두운 부분 (0.0)**: 검정 라인 표시
- 옷 주름, 디테일 선 등을 표현

---

## 5. 버텍스 컬러

### R 채널 - 그림자 바이어스

![Vertex R](images/posts/gbvs-toon-shader-implementation/VertextR.png)
*Vertex Color R: 그림자 바이어스/AO*

버텍스 컬러의 R 채널은 그림자가 얼마나 쉽게 지는지를 제어한다:

```hlsl
float ao = IN.vertex_color.r;
```

- **밝은 부분**: 그림자가 덜 짐
- **어두운 부분**: 그림자가 쉽게 짐 (AO 역할)

---

## 6. 셰이더 단계별 조합

이제 각 요소를 하나씩 추가하면서 최종 결과물에 어떻게 도달하는지 보자.

### Step 1: Toon Shadow

![Toon Shadow](images/posts/gbvs-toon-shader-implementation/ToonShadow.png)
*Toon Shadow: Half-Lambert 기반 이진화 그림자*

가장 기본적인 셀 셰이딩. 노멀과 라이트 방향의 내적(NdotL)에 Half-Lambert를 적용한 후 임계값으로 이진화:

```hlsl
half NdotL = dot(normalDir, lightDir);
half half_lambert = (NdotL + 1.0) * 0.5;
half lambert_term = half_lambert * ao + diffuse_control;
lambert_term *= shadow;  // URP 그림자 감쇠
half toon_diffuse = saturate((lambert_term - _ToonThresHold) * _ToonHardness);
```

핵심 포인트:
- `half_lambert`: NdotL을 0~1 범위로 리매핑
- `ao`: 버텍스 컬러 R (정점별 AO)
- `diffuse_control`: ILM G 채널 (아티스트 제어)
- `_ToonHardness`: 경계선 선명도 제어

### Step 2: Diffuse

![Diffuse](images/posts/gbvs-toon-shader-implementation/Diffuse.png)
*Diffuse: Base Map + SSS Map 적용*

텍스처를 적용한 상태. 밝은 면은 Base Map, 그림자 면은 SSS Map 색상 사용:

```hlsl
half3 final_diffuse = lerp(sss_color, base_color, toon_diffuse) * mainLight.color;
```

### Step 3: Diffuse + Inner Line

![Diffuse + Inner Line](images/posts/gbvs-toon-shader-implementation/Diffuse+InnerLine.png)
*Inner Line 추가: 옷 주름과 디테일 표현*

ILM A 채널과 Detail Map을 사용하여 내부 라인 렌더링:

```hlsl
// inner_line: 1.0 = 일반 영역, 0.0 = 라인 영역
half3 inner_line_color = lerp(base_color * 0.2, float3(1.0, 1.0, 1.0), inner_line);
half3 detail_color = SAMPLE_TEXTURE2D(_DetailMap, sampler_linear_repeat, IN.uv2).rgb;
detail_color = lerp(base_color * 0.2, float3(1.0, 1.0, 1.0), detail_color);
half3 final_line = inner_line_color * inner_line_color * detail_color;
```

최종 결과에 곱하기(Multiply)로 적용해서 어두운 라인을 만든다.

### Step 4: Diffuse + Inner Line + Specular

![Diffuse + Inner Line + Specular](images/posts/gbvs-toon-shader-implementation/Diffuse+Inner+Specular.png)
*Specular 추가: 금속 광택 표현*

ILM R 채널로 스페큘러 강도, B 채널로 크기를 제어:

```hlsl
// NdotV (Half-Lambert 스타일)
float NdotV = (dot(normalDir, viewDir) + 1.0) * 0.5;
float spec_term = NdotV * ao + diffuse_control;
spec_term = half_lambert * 0.9 + spec_term * 0.1;

// 툰 스페큘러 계산
half toon_spec = saturate((spec_term - (1.0 - spec_size * _SpecSize)) * 500);
half3 spec_color = (_SpecColor.rgb + base_color) * 0.5;
half3 final_spec = toon_spec * spec_color * spec_intensity;
```

- `spec_intensity`: ILM R 채널
- `spec_size`: ILM B 채널
- 스페큘러 색상은 설정 색상과 베이스 색상의 평균

### Step 5: Full Shader

![Full Shader](images/posts/gbvs-toon-shader-implementation/FullShader.png)
*Full Shader: 림라이트, 외곽선 등 모든 요소 추가*

림라이트 계산 (카메라 공간 기반):

```hlsl
// 카메라 공간에서 림라이트 방향 계산
float3 lightDir_rim = normalize(mul((float3x3)UNITY_MATRIX_I_V, _RimLightDir.xyz));
half NdotL_rim = (dot(normalDir, lightDir_rim) + 1.0) * 0.5;
half rimlight_term = NdotL_rim + diffuse_control;
half toon_rim = saturate((rimlight_term - _ToonThresHold) * 20.0);
half3 rim_color = (_RimLightColor.rgb + base_color) * 0.5 * sss_alpha;
half3 final_rimlight = toon_rim * rim_color * base_mask * toon_diffuse * _RimLightColor.a;
```

최종 합성:

```hlsl
half3 final_color = half3(0, 0, 0);
final_color += final_diffuse;      // 디퓨즈
final_color += final_spec;         // 스페큘러
final_color += final_rimlight;     // 림라이트
final_color *= final_line;         // 내부 라인 (곱하기)
```

### Step 6: Post Processing

![Tonemapping + Bloom](images/posts/gbvs-toon-shader-implementation/Tonemapping+Bloom.png)
*최종: 톤매핑 + 블룸 포스트 프로세싱*

마지막으로 URP 포스트 프로세싱 적용:
- **톤매핑**: 색감 보정
- **블룸**: 밝은 부분에 글로우 효과

---

## 7. 외곽선 (Outline Pass)

별도의 Pass에서 인버티드 헐 방식으로 외곽선 렌더링:

```hlsl
Pass
{
    Name "Outline"
    Cull Front  // 앞면 컬링 → 뒷면만 렌더링
    ZWrite On

    // ...

    Varyings vert(Attributes IN)
    {
        Varyings OUT;

        // View Space에서 아웃라인 확장
        float3 pos_view = TransformWorldToView(TransformObjectToWorld(IN.positionOS.xyz));
        float3 normal_world = TransformObjectToWorldNormal(IN.normalOS);
        float3 outline_dir = normalize(mul((float3x3)UNITY_MATRIX_V, normal_world));

        // 노멀 방향으로 확장
        pos_view += outline_dir * _OutlineWidth * 0.001;

        OUT.positionCS = mul(UNITY_MATRIX_P, float4(pos_view, 1.0));
        return OUT;
    }

    half4 frag(Varyings IN) : SV_Target
    {
        // BaseMap 기반 동적 아웃라인 색상
        float3 basecolor = SAMPLE_TEXTURE2D(_BaseMap, sampler_linear_repeat, IN.uv).xyz;

        // 채도 높은 버전 추출
        half maxComponent = max(max(basecolor.r, basecolor.g), basecolor.b) - 0.004;
        half3 saturatedColor = step(maxComponent.rrr, basecolor) * basecolor;
        saturatedColor = lerp(basecolor.rgb, saturatedColor, 0.6);

        // 최종 아웃라인 색상
        half3 outlineColor = 0.8 * saturatedColor * basecolor * _OutlineColor.xyz;
        return float4(outlineColor, 1.0);
    }
}
```

외곽선 색상이 검정색이 아니라 베이스 컬러를 기반으로 동적으로 계산됨.

---

## 8. 디버그 모드

셰이더에 디버그 모드를 넣어서 각 요소를 개별 확인할 수 있게 했음:

```hlsl
[Header(Debug Analysis)]
[KeywordEnum(None, BaseColor, SSSColor, ILM_R, ILM_G, ILM_B, ILM_A, Shadow, VertexAO, Normals, HalfLambert, ToonDiffuse)]
_DebugMode("Debug Mode", Float) = 0
```

```hlsl
// 디버그 모드 처리
#if defined(_DEBUGMODE_ILM_R)
    return float4(float3(spec_intensity, spec_intensity, spec_intensity), 1.0);
#elif defined(_DEBUGMODE_ILM_G)
    return float4(ilm_map.ggg, 1.0);
#elif defined(_DEBUGMODE_VERTEXAO)
    return float4(float3(ao, ao, ao), 1.0);
// ... 기타 모드
#endif
```

이 글의 스크린샷들도 이 디버그 모드로 촬영한 것임.

---

## 9. 핵심 요약

| 요소 | 소스 | 역할 |
|------|------|------|
| 기본 색상 | Base Map | 밝은 면 색상 |
| 그림자 색상 | SSS Map | 어두운 면 색상 |
| 스페큘러 강도 | ILM R | 반사광 위치 |
| 그림자 임계값 | ILM G | AO / 영구 그림자 |
| 스페큘러 크기 | ILM B | 하이라이트 크기 |
| 내부 라인 | ILM A | 주름, 디테일 선 |
| 그림자 바이어스 | Vertex R | 정점별 AO |

---

## 10. 아티스트 제어의 핵심

이 시스템의 핵심은 **조명과 독립적인 아티스트 제어**다:

1. **ILM G 채널**: "이 부분은 항상 그림자"를 텍스처로 직접 지정
2. **ILM R 채널**: "이 부분만 반짝이게"를 텍스처로 직접 지정
3. **ILM A 채널**: "여기에 선 그려줘"를 텍스처로 직접 지정
4. **Vertex Color**: "이 정점 주변은 어둡게"를 메시에 직접 페인팅

실시간 조명 계산의 결과를 그대로 받아들이는 게 아니라, 아티스트가 "2D 애니메이션이라면 이렇게 보여야 해"라고 지정한 대로 렌더링되는 것.

---

### 참고 자료

- [이전 글: 길티기어 Xrd 카툰 렌더링 분석](/posts/guilty-gear-rendering)
- [GDC 2015: Guilty Gear Xrd's Art Style](https://www.youtube.com/watch?v=yhGjCzxJV3E)
- [Aerthas Unity ASW Shader](https://github.com/Aerthas/UNITY-Arc-system-Works-Shader)
- 직접 분석 및 Unity URP 구현
