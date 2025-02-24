---
layout: single
title: "Java 프로젝트에서 Spotless 적용해 코드 포맷팅 자동화하기"
categories: java
tag: [java-formatting, formatting, code-formatting]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/ghhO2iQ.png"
# search: false
---

![spotless-image](https://i.imgur.com/ghhO2iQ.png)

팀 프로젝트에서 개발자마다 코드 스타일이 다르면 merge 시 불필요한 충돌이 발생하고, 가독성이 떨어지며, 코드 리뷰 시간이 길어지는 문제가 생긴다. **Spotless**는 Gradle 플러그인을 통해 코드를 자동으로 포맷팅해 일관된 스타일을 유지하고 불필요한 수정 사항을 줄여주는 도구이다.

이 포스트에서 Spotless를 Java 프로젝트에 적용하는 방법과 이를 Git Hook을 통해 강제해 공통된 포맷팅을 유지할 수 있는 방법을 알아보자.

## Spotless vs Checkstyle 비교

Spotless와 Checkstyle은 모두 Java 코드의 스타일을 관리하는 도구이지만, 각각의 특징과 장단점이 있다. 아래의 비교를 통해 Spotless를 선택한 이유를 알아보자.

### 자동 수정 기능

- **Spotless**: Gradle 명령어로 코드를 자동으로 수정. IDE 설정이나 추가 플러그인 없이도 `./gradlew spotlessApply` 만으로 전체 코드베이스의 포맷팅 가능
- **Checkstyle**: 기본적으로 검사 기능만 제공. 자동 수정을 위해서는 IDE 플러그인 설치나 추가 도구 필요

### 설정 복잡도

- **Spotless**: 간단한 Gradle DSL로 설정 가능. Google Java Format 등 널리 사용되는 포맷터를 즉시 적용 가능
- **Checkstyle**: XML 기반의 상세한 규칙 설정 필요. 초기 설정이 복잡하고 유지보수에 더 많은 노력 필요

### 확장성

- **Spotless**: Java 외에도 Kotlin, Groovy, JSON, XML 등 다양한 파일 형식 지원. 단일 도구로 프로젝트 전체의 포맷팅 관리 가능
- **Checkstyle**: Java 코드 검사에 특화. 다른 파일 형식을 처리하려면 추가 도구 필요

### CI/CD 통합

- **Spotless**: `spotlessCheck` 태스크로 검사만 수행하거나, `spotlessApply`로 자동 수정 가능. 파이프라인 구성이 단순
- **Checkstyle**: 검사 결과 리포트 생성은 쉽지만, 자동 수정을 파이프라인에 통합하기 어려움

### Git Hook 연동

- **Spotless**: pre-commit 훅에서 `spotlessApply` 실행만으로 자동 포맷팅 가능. 추가 도구 설치 불필요
- **Checkstyle**: 자동 수정을 위해서는 별도의 포맷터나 IDE 설정을 Git Hook과 연동해야 함

## Spotless 설정 방법

### 1. build.gradle 설정

```groovy
plugins {
    id 'com.diffplug.spotless' version '7.0.2'  // Spotless 플러그인 추가
}

spotless {
    // 비 Java 파일 포맷팅 규칙 (Gradle, 마크다운 등)
    format 'misc', {
        target('*.gradle', '.gitignore', '*.md', '*.yml', '*.json')  // 대상 파일 지정
        trimTrailingWhitespace()  // 끝 공백 제거 (줄 끝의 불필요한 공백 삭제)
        endWithNewline()          // 파일 끝 개행 추가 (파일 종료 시 빈 줄 추가)
        indentWithSpaces(4)       // 들여쓰기 4칸 공백 사용 (기본값)
    }
    
    // Java 파일 포맷팅 규칙
    java {
        googleJavaFormat('1.25.2')   // Google Java Style 가이드 적용
            .reflowLongStrings()     // 100자 이상 문자열 자동 줄바꿈
            .skipJavadocFormatting() // Javadoc 주석 서식 유지
        formatAnnotations()          // 어노테이션 정렬 (e.g. @Override 위치 통일)
        removeUnusedImports()        // 사용하지 않는 import 제거
    }
}
```

**주요 옵션 설명**  
- `googleJavaFormat`: 구글의 공식 코드 스타일 적용 (공식 문서: [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html))
- `reflowLongStrings`: 긴 문자열을 공백 기준으로 자연스럽게 분할
- `skipJavadocFormatting`: Javadoc 주석 내의 줄바꿈 보존 (설명문의 의도적 구조 유지)
- `formatAnnotations`: `@Override` 같은 어노테이션을 메서드 시그니처와 같은 줄로 이동

## Git Hook 설정

Git Hook을 사용하면 커밋 전/후 시점에 자동으로 스크립트를 실행해 포맷팅을 강제할 수 있다. 이는 팀원들이 포맷팅 규칙을 수동으로 적용하는 것을 잊더라도 자동으로 보정해주어 **코드베이스의 일관성**을 유지시켜 줄 수 있다.

### 1. .githooks로 깃훅 디렉터리 변경

기본적으로 .git/hooks 디렉토리는 각 로컬 환경에만 존재하므로 다른 팀원들과 공유되지 않는다. `.githooks`와 같은 버전 관리 디렉토리에 훅을 두면, 훅을 프로젝트의 일부로 만들어 모든 팀원이 동일한 규칙과 절차를 따를 수 있다.

#### Git Hook 디렉터리 이동

```bash
# 프로젝트 루트에 .githooks 디렉토리 생성
mkdir -p .githooks

# 프로젝트 전용 hooks 경로 설정 (글로벌 영향 없음)
git config core.hooksPath .githooks
```

**⚠️ 주의사항**  
기존 `./git/hooks` 대신 프로젝트 내 `.githooks`를 사용하도록 설정한다. 이 변경사항은 **현재 프로젝트에만 적용**되며, `git clone` 시 자동으로 적용되지 않으므로 팀원들이 반드시 설정해야 한다.

### 2. pre-commit 훅 등록

```bash
# .githooks/pre-commit 파일 생성 후 실행 권한 부여
chmod +x .githooks/pre-commit
```

#### `pre-commit` 전체 스크립트

```bash
#!/bin/bash
set -euo pipefail  # 오류 발생 시 즉시 종료

# 1. 스테이징된 Java 파일 필터링
mapfile -t STAGED_FILES < <(git diff --cached --name-only | grep '\.java$')

# 2. 파일 없으면 종료
if [ ${#STAGED_FILES[@]} -eq 0 ]; then
    echo "✅ 스테이징된 Java 파일이 없습니다!"
    exit 0
fi

# 3. 모듈 경로 추출 (예: "moduleA/src/main/java/..." → ":moduleA")
declare -A MODULES_SET=()
for FILE in "${STAGED_FILES[@]}"; do
    if [[ "$FILE" == *"/src/main/java/"* ]]; then
        MODULE_PATH=$(echo "$FILE" | sed -E 's|(.*)/src/main/java/.*|\1|')
        MODULE=":${MODULE_PATH//\//:}"  # 슬래시(/)를 콜론(:)으로 변환 (Gradle 서브모듈 형식)
        MODULES_SET["$MODULE"]=1
    fi
done

# 4. Gradle 태스크 실행 (예: :moduleA:spotlessApply)
if [ ${#MODULES_SET[@]} -gt 0 ]; then
    ./gradlew "${!MODULES_SET[@]/%/:spotlessApply}"  # 배열 요소 끝에 ":spotlessApply" 추가
fi

# 5. 변경사항 다시 스테이징
git add "${STAGED_FILES[@]}"
```

**스크립트 동작 원리**  
1. **스테이징 영역 감지**: `git diff --cached`로 커밋 대상 Java 파일만 필터링  
   → 부분 커밋 시 변경된 파일 중 일부만 처리하기 위함
2. **모듈 자동 탐지**: 파일 경로에서 `/src/main/java/`를 기준으로 모듈 추출  
   (예: `moduleA/src/main/java/com/example/Test.java` → `:moduleA`)
3. **Gradle 태스크 동적 생성**: `:moduleA:spotlessApply` 형태로 명령어 생성  
   → 다중 모듈 프로젝트에서 해당 모듈만 포맷팅
4. **재스테이징**: 포맷팅 결과를 커밋에 자동 반영

---

## 결론

Spotless를 적용하면 다음과 같은 이점을 얻을 수 있다.

1. **코드 스타일 통일성** 유지  
   - 모든 팀원이 동일한 코드 컨벤션 사용
2. **Merge Conflict 감소**  
   - 포맷팅 차이로 인한 충돌 최소화
3. **리뷰 시간 절약**  
   - 포맷팅 이슈 논의 불필요 (기계적 검토 시간 감소)
4. **자동화로 인한 개발 생산성 향상**  
   - 수동 포맷팅 작업 제거

또한 이를 Git Hook과 연동하면 포맷팅 누락을 근본적으로 방지할 수 있다. 추가적으로  CI/CD 파이프라인에 `./gradlew spotlessCheck`를 추가하여 포맷팅 검사를 자동화해 포매팅을 강제할 수도 있다.
