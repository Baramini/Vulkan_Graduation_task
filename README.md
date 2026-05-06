# Vulkan GPU-Driven Rendering

> 졸업 프로젝트 — GPU-Driven Rendering 파이프라인 위에 Lighting 및 Post-Processing 시스템 구현  
> 卒業制作 — GPU駆動レンダリングパイプライン上にLightingおよびPost-Processingシステムを実装

---

## 📌 프로젝트 개요 / プロジェクト概要

본 레포지토리는 팀 졸업과제로 진행된 Vulkan 기반 GPU-Driven Renderer 프로젝트를 **개인 레포지토리로 이전한 것**입니다.  
원본 프레임워크는 팀장의 레포지토리에서 fork하였으며, 이후 독립적인 레포지토리로 옮겼습니다.

このリポジトリは、チーム卒業制作として進めたVulkanベースのGPU駆動レンダラープロジェクトを**個人リポジトリに移行したもの**です。  
元のフレームワークはチームリーダーのリポジトリからforkし、その後独立したリポジトリに移行しました。

### 역할 분담 / 担当範囲

| 담당 / 担当 | 내용 / 内容 |
|------|------|
| **팀장 / チームリーダー** | 전체 프레임워크 설계, Batch System, Culling Render Pass, Shadow Map |
| **본인 / 自分** | Lighting Pass (Blinn-Phong), Post-Processing Pass (Bloom + Tone Mapping) |

---

## ⚙️ 실행 방법 / セットアップ方法

### 사전 준비 / 事前準備

프로젝트를 빌드하기 전에 **meshoptimizer**를 수동으로 추가해야 합니다.  
ビルド前に **meshoptimizer** を手動で追加する必要があります。

```
프로젝트 루트/
└── ThirdParty/
    └── meshoptimizer.zip   ← 해당 zip파일의 압축을 해제 / このzipファイルを解凍してください
```
1. 압축을 해제합니다. / zipファイルを解凍します。
2. Riche.sln을 실행하고 빌드합니다. / Riche.slnを開いてビルドします。

---

## 🏗️ 파이프라인 구조 / パイプライン構造

아래는 전체 렌더링 파이프라인 구조입니다.  
**보라색** 영역이 본인이 직접 구현한 부분입니다.

以下は全体のレンダリングパイプライン構造です。  
**紫色**の領域が自分で実装した部分です。

![Pipeline Diagram](README_assets/pipeline_diagram.svg)

---

## ✨ 구현 내용 / 実装内容

### 1. Lighting Pass — Blinn-Phong Shading

GPU-Driven 파이프라인의 구조를 파악한 뒤, **Lighting 파이프라인을 설계하고 Fragment Shader에서 Blinn-Phong 조명 모델을 구현**했습니다.

GPU駆動パイプラインの構造を把握した上で、**LightingパイプラインをFragment ShaderにてBlinn-Phong照明モデルで実装**しました。

```glsl
// Bindless Texture — SSBO에서 오브젝트별 텍스처 인덱스 참조
int  textureIdx = nonuniformEXT(ssbo_TextureID.handle[inIndex].materialID);
vec4 newColor   = textureLod(
    sampler2D(nonuniformEXT(u_DiffuseTextureList[textureIdx]), linearWrapSS),
    inFragTexcoord, 0);

// Ambient / Diffuse / Specular
vec3 ambient  = u_ShaderSetting.ambientStrength * newColor.xyz;
float diff    = max(dot(normal, lightDir), 0.0);
vec3  diffuse = diff * u_ShaderSetting.lightColor.xyz * newColor.xyz;
float spec    = pow(max(dot(viewDir, reflectDir), 0.0), u_ShaderSetting.shininess);
vec3  specular = spec * u_ShaderSetting.specularStrength * u_ShaderSetting.lightColor.xyz;

outColour = vec4(ambient + diffuse + specular, newColor.a);
```

**어필 포인트 / ポイント**
- Bindless Texture (`u_DiffuseTextureList[]`) + SSBO 기반 텍스처 인덱스 참조 구조를 이해하고 활용  
  BindlessテクスチャとSSBOベースのテクスチャインデックス参照構造を理解し活用
- `nonuniformEXT` qualifier로 GPU 분기 없이 오브젝트별 텍스처 샘플링  
  `nonuniformEXT` qualifierによりGPU分岐なしにオブジェクトごとのテクスチャをサンプリング

---

### 2. Post-Processing Pass — Bloom + ACES Tone Mapping

Lighting Pass 결과를 입력으로 받아 **Bloom과 ACES Filmic Tone Mapping을 단일 패스에서 순서대로 처리**합니다.

Lightingパスの結果を入力として受け取り、**BloomとACESフィルミックトーンマッピングを単一パスで順次処理**します。

```glsl
// ① 휘도 기반 Bloom 마스크
float luminance = dot(color.rgb * shadowFactor, vec3(0.2126, 0.7152, 0.0722));
luminance       = pow(luminance, 4.0);          // 대비 강화
float bloomMask = smoothstep(0.8, 1.0, luminance);

// ② Gaussian Blur + Bloom 합산
vec3 blurred = gaussianBlurLit(inputColour, u_ShadowTexture, inFragTexcoord.xy);
lit         += blurred * bloomMask * blurIntensity;

// ③ ACES Filmic Tone Mapping (런타임 토글 가능)
vec3 ACESFilm(vec3 x) {
    const float a=2.51, b=0.03, c=2.43, d=0.59, e=0.14;
    return clamp((x*(a*x+b))/(x*(c*x+d)+e), 0.0, 1.0);
}
if (u_ShaderSetting.isTonemapping != 0) {
    mapped = ACESFilm(lit * 0.6);
    mapped = pow(mapped, vec3(1.0 / 2.2)); // Gamma correction
}
```

**어필 포인트 / ポイント**
- Reinhard 대신 **ACES Filmic** 선택 — 영화적 색감과 하이라이트 롤오프  
  Reinhardの代わりに**ACES Filmic**を採用 — 映画的な色調とハイライトのロールオフ
- `pow(luminance, 4.0)` + `smoothstep` 조합으로 Bloom 마스크 품질 향상  
  `pow(luminance, 4.0)` + `smoothstep`の組み合わせでBloomマスクの品質を向上
- Tone Mapping은 `isTonemapping` 플래그로 런타임 토글 가능 (디버깅 용이)  
  トーンマッピングは`isTonemapping`フラグでランタイムトグル可能（デバッグ容易）
- Culling Pass의 Shadow Map 결과를 Post-Processing 입력으로 활용 — 패스 간 연결 구조 이해  
  Culling PassのShadow Map結果をPost-Processingの入力として活用 — パス間の接続構造を理解

---

### 3. GPU-Driven 구조 이해 및 활용

팀장이 설계한 **Batch System과 Descriptor Set 분리 구조**를 파악하고, 그 위에 Lighting 파이프라인을 올렸습니다.

チームリーダーが設計した **BatchSystemとDescriptorSet分離構造**を把握し、その上にLightingパイプラインを構築しました。

```cpp
// Descriptor Set을 갱신 빈도별로 분리 바인딩
// set=0 : Camera UBO          — 프레임당 1회 갱신
// set=1 : SSBO (Object/Batch) — 오브젝트별 데이터
// set=2 : Sampler             — 고정
// set=3 : Bindless Texture    — 인덱스로 직접 참조

vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, layout,
    0, 1, &GetVkDescriptorSet("ViewProjection_ALL" + frameIdx), 0, nullptr);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, layout,
    1, 1, &GetVkDescriptorSet("BATCH_ALL" + frameIdx), 0, nullptr);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, layout,
    2, 1, &GetVkDescriptorSet("SamplerList_ALL"), 0, nullptr);
vkCmdBindDescriptorSets(cmd, VK_PIPELINE_BIND_POINT_GRAPHICS, layout,
    3, 1, &GetVkDescriptorSet("DiffuseTextureList"), 0, nullptr);

// GPU-Driven — CPU 개입 없이 GPU가 드로우 커맨드를 직접 소비
vkCmdDrawIndexedIndirect(cmd,
    g_BatchManager.m_indirectDrawCommandBuffer.buffer,
    miniBatch.m_indirectCommandsOffset,
    drawCount,
    sizeof(VkDrawIndexedIndirectCommand));
```

---

## 🛠️ 기술 스택 / 技術スタック

| 항목 / 項目 | 내용 / 内容 |
|------|------|
| Language | C++ |
| Graphics API | Vulkan |
| Shader | GLSL (SPIR-V) |
| Third-Party | meshoptimizer, GLM, GLFW |
| Build | Visual Studio (Riche.sln) |

---

## 📎 관련 링크 / 関連リンク

- [포트폴리오 사이트](#) — 렌더링 결과 영상 및 파이프라인 구조도 포함
