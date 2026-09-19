# Armor Chroma Refabricated - Minecraft 26.2 포팅 가이드

## 개요
- 저장소: https://github.com/cat4blep/armor-chroma-refabricated
- 목표: Minecraft 26.2 포팅
- git clone 위치: `C:\Users\user\armor-chroma-refabricated`
- GitHub 배포 repo: https://github.com/githubyoon/armor-chroma-refabricated-26.2

## 현재 상태
- v262x 소스 파일 생성 완료 (17개 파일)
- build.gradle / settings.gradle 수정 완료
- **빌드 미완료** - Loom 1.16.1 + Gradle 9.4에서 MC 26.2 의존성 가져오기까지 성공, 컴파일 테스트 필요

---

## 수정한 파일 목록

### 1. `settings.gradle`
```groovy
include('v121x', 'v261x', 'v262x')  // v262x 추가

project(':v121x').projectDir = file('versions/v121x')
project(':v261x').projectDir = file('versions/v261x')
project(':v262x').projectDir = file('versions/v262x')  // 추가
```

### 2. `build.gradle` - versionTargets에 v262x 추가
```groovy
v262x: [
    loomPlugin            : 'net.fabricmc.fabric-loom',
    usesMappings          : false,
    minecraftVersion      : '26.2',
    loaderVersion         : '0.19.3',
    fabricApiVersion      : '0.156.0+26.2',
    modMenuVersion        : '20.0.2',
    clothConfigVersion    : '26.2.155',
    minecraftDependency   : '>=26.2',
    loaderDependency      : '>=0.19.3',
    javaVersion           : 25,
    mixinCompatibilityLevel: 'JAVA_25',
    artifactSuffix        : 'mc26.2.x',
    artifactVersionSuffix : '26.2.x'
]
```
- 플러그인 버전은 `fabric-loom` 1.16.1 / `net.fabricmc.fabric-loom` 1.16.1 유지
  - Loom 1.17은 Gradle 9.5 필요 but Gradle 9.5 아직 미출시

### 3. `versions/v262x/build.gradle`
```groovy
// Configured from the root build script.
```

---

## v262x 소스 파일 (versions/v262x/src/main/java/nukeduck/armorchroma/)

### 26.1 → 26.2 API 변경점 (ArmorChromaRenderLayers.java에서 반영)

| 변경 전 (v261x) | 변경 후 (v262x) |
|---|---|
| `import com.mojang.blaze3d.platform.DestFactor` | `import com.mojang.blaze3d.platform.BlendFactor` |
| `import com.mojang.blaze3d.platform.SourceFactor` | (삭제) |
| `new BlendFunction(SourceFactor.DST_COLOR, DestFactor.ZERO)` | `new BlendFunction(BlendFactor.DST_COLOR, BlendFactor.ZERO)` |
| `.withColorTargetState(new ColorTargetState(...))` | `.withColorTargetState(0, new ColorTargetState(...))` (인덱스 추가) |

### v262x에서 변경된 파일: ArmorChromaRenderLayers.java
```java
package nukeduck.armorchroma;

import com.mojang.blaze3d.pipeline.BlendFunction;
import com.mojang.blaze3d.pipeline.ColorTargetState;
import com.mojang.blaze3d.pipeline.DepthStencilState;
import com.mojang.blaze3d.pipeline.RenderPipeline;
import com.mojang.blaze3d.platform.BlendFactor;
import com.mojang.blaze3d.platform.CompareOp;
import net.minecraft.client.renderer.RenderPipelines;
import net.minecraft.resources.Identifier;

public class ArmorChromaRenderLayers {

    private static final BlendFunction MASKED_ICON_BLEND_FUNCTION = new BlendFunction(BlendFactor.DST_COLOR, BlendFactor.ZERO);

    private static final RenderPipeline MASKED_ICON_PIPELINE = RenderPipelines.register(
            RenderPipeline.builder(RenderPipelines.GUI_TEXTURED_SNIPPET)
                    .withLocation(Identifier.fromNamespaceAndPath(ArmorChroma.MODID, "pipeline/masked_icon"))
                    .withColorTargetState(0, new ColorTargetState(MASKED_ICON_BLEND_FUNCTION))
                    .withDepthStencilState(new DepthStencilState(CompareOp.EQUAL, false))
                    .build());

    public static RenderPipeline getMaskedIcon() {
        return MASKED_ICON_PIPELINE;
    }
}
```

### 나머지 파일: v261x와 동일
v262x 디렉토리에 동일하게 복사된 파일들:
- `ArmorChroma.java` - 메인 모드 클래스
- `ArmorBarSegment.java` - 아머 바 세그먼트
- `ArmorChromaDebugCommand.java` - 디버그 커맨드
- `EntityAttributeInstanceAccess.java` - 인터페이스
- `GuiArmor.java` - 아머 바 HUD 렌더링
- `MaterialHelper.java` - 아머 머티리얼 헬퍼
- `mixin/InGameHudMixin.java` - Gui.class mixin (extractArmor 메서드)
- `mixin/ItemStackMixin.java` - 툴팁 mixin
- `mixin/EntityAttributeInstanceMixin.java` - 속성 mixin
- `config/ArmorChromaConfig.java`
- `config/ArmorIcon.java`
- `config/IconData.java`
- `config/IconTable.java`
- `config/ModMenuIntegration.java`
- `config/SpecialIconKey.java`
- `config/Util.java`

---

## 다음에 해야 할 일

### 1. 빌드 컴파일 확인
```bash
cd C:\Users\user\armor-chroma-refabricated
.\gradlew.bat :v262x:build
```
컴파일 에러가 나면:
- 26.1→26.2에서 추가 API 변경이 있을 수 있음 (NeoForge primer 참고)
- 주요 참고 자료:
  - https://docs.fabricmc.net/develop/porting/
  - https://docs.neoforged.net/primer/docs/26.2/
  - https://fabricmc.net/2026/06/15/262.html

### 2. 잠재적 추가 API 변경 (빌드 에러 시 확인)
- `DepthStencilState` 생성자 - 깊이 값이 반전될 수 있음 (LEqual→GEqual, 값 negation)
- `RenderPipeline$Snippet` - `colorTargetState`→`colorTargetStates`(배열), `vertexFormat`→`vertexFormatPerBuffer`(배열)
- `GuiGraphicsExtractor` - `entity()` 메서드 시그니처 변경 (Vector3fc/Quaternionfc)
- `extractArmor` mixin 타겟 메서드명 확인 (변경 없을 것으로 추정)
- `AutoConfig`/`AutoConfigClient` - cloth-config 26.2에서 변경 없을 것으로 추정

### 3. GitHub push
```bash
cd C:\Users\user\armor-chroma-refabricated
git remote add origin https://github.com/githubyoon/armor-chroma-refabricated-26.2.git
git branch -M main
git add .
git commit -m "Port to Minecraft 26.2"
git push -u origin main
```

---

## 프로젝트 구조
```
armor-chroma-refabricated/
├── build.gradle              (versionTargets에 v262x 추가됨)
├── settings.gradle           (v262x include 추가됨)
├── gradle.properties
├── gradle/wrapper/gradle-wrapper.properties (Gradle 9.4.0)
├── src/main/java/            (공용 소스 - 1.21.x용)
├── src/main/resources/       (공용 리소스)
├── versions/
│   ├── v121x/                (MC 1.21.6)
│   ├── v261x/                (MC 26.1.2)
│   └── v262x/                (MC 26.2) ← 새로 생성
│       ├── build.gradle
│       └── src/main/java/nukeduck/armorchroma/
│           ├── ArmorChroma.java
│           ├── ArmorBarSegment.java
│           ├── ArmorChromaDebugCommand.java
│           ├── ArmorChromaRenderLayers.java  ← 26.2 API 반영됨
│           ├── EntityAttributeInstanceAccess.java
│           ├── GuiArmor.java
│           ├── MaterialHelper.java
│           ├── mixin/
│           │   ├── EntityAttributeInstanceMixin.java
│           │   ├── InGameHudMixin.java
│           │   └── ItemStackMixin.java
│           └── config/
│               ├── ArmorChromaConfig.java
│               ├── ArmorIcon.java
│               ├── IconData.java
│               ├── IconTable.java
│               ├── ModMenuIntegration.java
│               ├── SpecialIconKey.java
│               └── Util.java
```

## 의존성 버전 (v262x)
| 항목 | 버전 |
|---|---|
| Minecraft | 26.2 |
| Fabric Loader | 0.19.3 |
| Fabric API | 0.156.0+26.2 |
| ModMenu | 20.0.2 |
| Cloth Config | 26.2.155 |
| Java | 25 |
| Mixin Level | JAVA_25 |
| Loom | 1.16.1 (Gradle 9.5 미출시로 유지) |
