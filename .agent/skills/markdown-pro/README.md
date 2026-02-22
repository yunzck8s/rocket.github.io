# Professional Markdown Documentation

세련된 README 파일, 변경 이력(changelog), 기여 가이드 및 기술 문서를 작성하기 위한 전문가 수준의 Markdown 문서화 SKILL입니다.

## 개요 (Overview)

이 SKILL은 전문적이고 잘 구조화된 Markdown 문서를 작성하기 위한 포괄적인 가이드를 제공합니다. 최신 포맷팅, 배지 및 모범 사례를 적용한 README 파일, 변경 이력, 기여 가이드 및 기술 문서를 다룹니다.

배지와 섹션을 포함한 README 생성, git 히스토리를 이용한 자동 변경 이력 생성, 목차 생성, 기여 가이드라인 작성, 기술 문서 포맷팅, 구문 강조를 포함한 코드 문서화 시 이 SKILL을 사용하세요.

## 설치 (Installation)

기본적인 Markdown 문서화에는 설치가 필요하지 않습니다.

선택 사항: 헬퍼 스크립트 종속성 설치:

```bash
pip install markdown
```

## 포함 내용 (What's Included)

### SKILL.md
README 구조 모범 사례, 변경 이력 포맷, Markdown 포맷팅 모범 사례, 배지 생성, 코드 구문 강조, 표, 접기/펼치기 섹션, 알림 상자, 링크, 이미지 및 헬퍼 스크립트를 다루는 포괄적인 가이드입니다.

### scripts/
- `markdown_helper.py` - 다음을 위한 유틸리티 스크립트:
  - 목차 자동 생성
  - git 히스토리에서 변경 이력 생성
  - 링크 유효성 검사

### examples/
- `README_template.md` - 운영 수준의 README 템플릿
- `CHANGELOG_template.md` - Keep a Changelog 형식을 따르는 변경 이력 템플릿
- `CONTRIBUTING.md` - 기여 가이드라인 템플릿

## 빠른 시작 (Quick Start)

### README 구조

**필수 섹션:**

```markdown
# 프로젝트 이름

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](releases)

프로젝트가 무엇인지 설명하는 짧은 한 줄 설명.

## 목차

- [주요 특징](#features)
- [설치 방법](#installation)
- [사용법](#usage)
- [API 참조](#api-reference)
- [기여하기](#contributing)
- [라이선스](#license)

## 주요 특징

- **특징 1**: 장점과 함께 명확한 설명 제공
- **특징 2**: 어떤 문제를 해결하는지 기술
- **특징 3**: 독특한 강점 강조

## 설치 방법

```bash
pip install package-name
\```

## 사용법

```python
from package import Module

# 기본 예시
client = Module(api_key="your-key")
result = client.process(data)
\```

## 라이선스

이 프로젝트는 MIT 라이선스를 따릅니다. 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요.
```

### 변경 이력 포맷

```markdown
# 변경 이력

이 프로젝트의 모든 주목할 만한 변경 사항은 이 파일에 기록됩니다.

형식은 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)를 따르며,
이 프로젝트는 [시맨틱 버저닝](https://semver.org/spec/v2.0.0.html)을 준수합니다.

## [Unreleased]

### Added
- 새로운 기능 설명

### Changed
- 기존 기능 수정 사항

### Fixed
- 버그 수정

## [1.2.0] - 2025-01-15

### Added
- 사용자 인증 시스템 (#123)
- CSV 내보내기 기능 (#145)

### Fixed
- 백그라운드 프로세서의 메모리 누수 수정 (#139)
```

## 핵심 기능 (Key Features)

### README 생성
- 프로젝트 개요 및 설명
- 설치 지침
- 코드 블록을 포함한 사용 예시
- API 문서화
- 배지 및 실드
- 주요 특징 강조
- 스크린샷 및 데모

### 변경 이력 자동화
- 시맨틱 버저닝 형식
- Git 히스토리 파싱
- 자동 릴리스 노트 생성
- 주요 변경 사항 강조
- 기여자 표시

### 기술 문서화
- 명확한 섹션 계층 구조
- 코드 구문 강조
- API 참조 포맷팅
- 목차 생성
- 상호 참조
- 접기/펼치기 섹션

## Markdown 포맷팅 모범 사례

### 구문 강조를 포함한 코드 블록

```markdown
```python
def hello_world():
    """헬로 월드 메시지 출력."""
    print("Hello, World!")
\```

```javascript
function helloWorld() {
    console.log("Hello, World!");
}
\```

```bash
# 종속성 설치
npm install
\```
```

### 표 (Tables)

```markdown
| 기능 | 설명 | 상태 |
|---------|-------------|--------|
| 인증 | 사용자 인증 시스템 | ✅ 완료 |
| API | RESTful API 엔드포인트 | ✅ 완료 |
| 문서 | 문서화 작업 | 🚧 진행 중 |
| 테스트 | 유닛 및 통합 테스트 | ❌ 계획됨 |
```

### 접기/펼치기 섹션

```markdown
<details>
<summary>클릭하여 고급 설정 확인</summary>

## 고급 옵션

고급 설정을 구성합니다:

```yaml
advanced:
  cache_size: 1000
  timeout: 30
\```

</details>
```

### 알림 상자 (Alert Boxes)

```markdown
> **참고**: 이 기능은 Python 3.8 이상이 필요합니다.

> **주의**: 이 작업은 되돌릴 수 없습니다!

> **중요**: 업그레이드 전에는 항상 데이터를 백업하세요.
```

## 배지 생성

### 공통 배지 패턴

```markdown
<!-- License -->
![License](https://img.shields.io/badge/license-MIT-blue.svg)

<!-- Version -->
![Version](https://img.shields.io/badge/version-1.0.0-green.svg)

<!-- Build Status -->
![Build](https://img.shields.io/badge/build-passing-brightgreen.svg)

<!-- Coverage -->
![Coverage](https://img.shields.io/badge/coverage-95%25-brightgreen.svg)

<!-- Language -->
![Python](https://img.shields.io/badge/python-3.8+-blue.svg)

<!-- Platform -->
![Platform](https://img.shields.io/badge/platform-windows%20%7C%20macOS%20%7C%20linux-lightgrey.svg)
```

## 헬퍼 스크립트

### 목차 생성
```bash
python scripts/markdown_helper.py toc README.md
```

### Git으로부터 변경 이력 생성
```bash
python scripts/markdown_helper.py changelog --since v1.0.0 --output CHANGELOG.md
```

### 링크 유효성 검사
```bash
python scripts/markdown_helper.py validate docs/
```

## 모범 사례 (Best Practices)

### 수행할 작업 (Do's)
- 명확하고 설명적인 헤더를 사용하세요.
- 모든 주요 기능에 대해 코드 예시를 포함하세요.
- 프로젝트 상태를 한눈에 볼 수 있도록 배지를 추가하세요.
- 가독성을 위해 한 줄의 길이를 100자 이내로 유지하세요.
- 코드 블록에는 구문 강조를 사용하세요.
- 300행 이상의 문서에는 목차를 포함하세요.
- 모든 이미지에 대체 텍스트를 추가하세요.
- 관련 문서로의 링크를 제공하세요.

### 피해야 할 작업 (Don'ts)
- 일반적인 제목을 사용하지 마세요.
- 텍스트 덩어리를 길게 나열하지 마세요 (섹션으로 나누세요).
- 릴리스할 때 변경 이력을 업데이트하는 것을 잊지 마세요.
- 단순 URL만 적지 마세요 (설명적인 링크 텍스트 사용).
- 헤더 스타일을 혼용하지 마세요.
- 설명 없는 스크린샷을 넣지 마세요.
- 버전 번호를 하드코딩하지 마세요.

## 헤더 계층 구조
```markdown
# H1 - 프로젝트 제목 (문서당 하나만 사용)
## H2 - 주요 섹션
### H3 - 하위 섹션
#### H4 - 부가적인 포인트
##### H5 - 드물게 사용되는 깊은 중첩
```

## 목록 포맷팅
```markdown
<!-- 순서 없는 목록 -->
- 항목 1
- 항목 2
  - 중첩 항목

<!-- 순서 있는 목록 -->
1. 첫 번째 단계
2. 두 번째 단계

<!-- 작업 목록 -->
- [x] 완료된 작업
- [ ] 진행 예정 작업
```

## 강조 (Emphasis)
```markdown
*기울임* 또는 _기울임_
**굵게** 또는 __굵게__
***굵은 기울임***
~~취소선~~
`인라인 코드`
```

## API 문서 포맷
```markdown
### `Module.process(data, options=None)`

선택적 설정을 사용하여 입력 데이터를 처리합니다.

**매개변수:**
- `data` (str|dict): 처리할 입력 데이터
- `options` (dict, 선택 사항): 설정 옵션
  - `verbose` (bool): 상세 출력 활성화 (기본값: False)
  - `format` (str): 출력 형식 - 'json', 'yaml', 'xml' (기본값: 'json')

**반환값:**
- `dict`: 메타데이터가 포함된 처리 결과

**예외:**
- `ValueError`: 데이터가 유효하지 않은 경우

**예시:**
```python
result = client.process(
    data={"key": "value"},
    options={"verbose": True, "format": "json"}
)
\```
```

## 링크 및 참조
```markdown
<!-- 외부 링크 -->
[문서 보기](https://docs.example.com)

<!-- 내부 링크 -->
[설치 방법](#installation) 섹션을 참조하세요.

<!-- 참조 스타일 링크 -->
[프로젝트 홈페이지][homepage]와 [문서][docs]를 확인하세요.

[homepage]: https://example.com
[docs]: https://docs.example.com
```

## 이미지
```markdown
<!-- 표준 이미지 -->
![프로젝트 로고](assets/logo.png)

<!-- 대체 텍스트와 타이틀이 포함된 이미지 -->
![대시보드 스크린샷](screenshots/dashboard.png "메인 대시보드 화면")

<!-- 링크가 포함된 이미지 -->
[![데모 비디오](thumbnail.jpg)](https://youtube.com/watch?v=example)
```

## 템플릿

### 전문 README 템플릿
모든 권장 섹션이 포함된 운영 수준의 README 템플릿은 `examples/README_template.md`를 참조하세요.

### 변경 이력 템플릿
Keep a Changelog 형식을 따르는 올바른 포맷의 변경 이력 템플릿은 `examples/CHANGELOG_template.md`를 참조하세요.

### 기여 가이드라인
행동 강령, 개발 환경 설정, PR 프로세스를 포함한 기여 가이드라인 템플릿은 `examples/CONTRIBUTING.md`를 참조하세요.

## 일반적인 유스케이스

### 프로젝트 문서 작성
GitHub 프로젝트를 위해 적절한 구조와 포맷을 갖춘 포괄적인 README 파일을 생성합니다.

### 변경 이력 관리
시맨틱 버저닝을 따르며 git 커밋 히스토리에서 변경 이력 생성을 자동화합니다.

### API 문서화
코드 예시와 함께 명확하고 잘 포맷된 API 참조 문서를 작성합니다.

### 기여 가이드라인 작성
기여자들을 위해 설치 안내와 PR 프로세스를 포함한 명확한 가이드라인을 제공합니다.

## 문서
포괄적인 문서, 상세 포맷팅 패턴 및 고급 기법에 대해서는 `SKILL.md`를 참조하세요.

## 요구 사항
기본적인 Markdown 문서화에는 특별한 요구 사항이 없습니다.

선택 사항:
- Python 3.7+ (헬퍼 스크립트용)
- markdown (고급 기능을 위한 Python 라이브러리)
