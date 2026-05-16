# Supply Chain Attack Security Check

2026년 3~5월 발생한 npm/PyPI 공급망 공격 3건에 대한 점검 프롬프트입니다.
아래 코드 블록을 AI 코딩 어시스턴트(Claude Code, Cursor, Copilot 등)에 붙여넣으면 자동으로 점검합니다.

## 대상 공격

| 공격 | 패키지 | 플랫폼 | 노출 시간 |
|------|--------|--------|-----------|
| [axios 공급망 공격](https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan) | axios | npm | ~2-3시간 |
| [litellm/telnyx 공급망 공격](https://blog.pypi.org/posts/2026-04-02-incident-report-litellm-telnyx-supply-chain-attack/) | litellm, telnyx | PyPI | ~2시간 32분 |
| [TanStack 공급망 공격](https://github.com/TanStack/router/issues/7383) | @tanstack/* (42개 패키지) | npm | ~6분 |

## 점검 프롬프트

````text
내 컴퓨터가 아래 3건의 npm/PyPI 공급망 공격에 영향을 받았는지 점검해줘.

## 1. axios (npm) — 2026-04-XX
- 악성 버전: axios@1.14.1, axios@0.30.4
- 악성 의존성: plain-crypto-js (정상 axios에는 존재하지 않음)
- 악성 페이로드:
  - Windows: %PROGRAMDATA%\wt.exe
  - macOS: /Library/Caches/com.apple.act.mond
  - Linux: /tmp/ld.py
- C2 도메인: sfrclak.com
- 참고: https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan

## 2. litellm / telnyx (PyPI) — 2026-04-02
- 악성 버전: (아래 참고 링크에서 정확한 버전 확인 후 점검)
- 악성 파일: litellm_init.pth (Python 시작 시 자동 실행되는 credential stealer)
- 참고: https://blog.pypi.org/posts/2026-04-02-incident-report-litellm-telnyx-supply-chain-attack/

## 3. TanStack (npm) — 2026-05-11 19:20~19:26 UTC
- 악성 publish 시각: 2026-05-11 19:20~19:26 UTC (≈ 2026-05-12 04:20~04:26 KST), 약 6분간
- 영향 범위: 42개 @tanstack/* 패키지의 84개 악성 버전
  - 비영향 패키지군(이 패턴은 안전): @tanstack/query*, @tanstack/table*, @tanstack/form*, @tanstack/virtual*, @tanstack/store, @tanstack/start
  - 그 외 @tanstack/react-router, @tanstack/router, @tanstack/* 등은 영향 범위 — 설치 시점 확인 필요
- 악성 파일(IOC): 패키지 루트의 `router_init.js` (~2.3MB 난독화, package.json "files"에는 미포함 → tarball에만 존재)
- 악성 의존성 신호: package.json 내 `"@tanstack/setup": "github:tanstack/router#79ac49eedf774dd4b0cfa308722bc463cfe5885c"` (optionalDependencies)
- 2단계 페이로드 URL: litter.catbox.moe/h8nc9u.js, litter.catbox.moe/7rrc6l.mjs
- 유출 네트워크(Session/Oxen messenger): filev2.getsession.org, seed1.getsession.org, seed2.getsession.org, seed3.getsession.org
- 동작: `npm/pnpm/yarn install` 시 AWS IMDS/Secrets Manager, GCP metadata, K8s/Vault 토큰, `~/.npmrc`, GitHub token, `gh` CLI 자격증명, `.git-credentials`, SSH private key 수집
- 참고: https://github.com/TanStack/router/issues/7383 (GHSA-g7cv-rxg3-hmpx)

## 점검 방법 (OS 자동 감지하여 해당 항목만 실행)

### npm (axios)
1. 시스템 전체에서 axios가 포함된 package.json을 찾아 버전 확인 (node_modules + 글로벌)
2. package-lock.json / yarn.lock에서 axios@1.14.1, axios@0.30.4 참조 검색
3. plain-crypto-js 디렉토리 또는 패키지 참조 존재 여부
4. OS별 악성 페이로드 파일 존재 여부
5. DNS 캐시 또는 로그에서 sfrclak.com 접속 흔적 (가능한 경우)

### PyPI (litellm / telnyx)
1. pip, pip3, pipx, conda, poetry 등 모든 패키지 매니저에서 litellm/telnyx 설치 여부 및 버전
2. 모든 Python 가상환경(venv, conda env) 내 site-packages 포함 검색
3. site-packages/**/*.pth 파일 중 수상한 항목 (특히 litellm_init.pth)
4. 설치된 litellm의 dist-info/METADATA에서 설치 시점 확인 → 공격 시간대(4/2) 전후 여부 판단

### npm (TanStack)
1. 글로벌 / 시스템 전체 package.json·lockfile에서 `@tanstack/` 참조 검색 (node_modules 제외 후, 별도로 node_modules도 검사)
2. 검색된 패키지가 비영향군(`query*/table*/form*/virtual*/store/start`)인지 영향군(`react-router`, `router` 등)인지 분류
3. 영향군 패키지의 설치 시각(`stat`의 birth/mtime) 확인 → 공격 윈도우(2026-05-11 19:20~19:26 UTC) 이후 설치 여부 판단
4. 시스템 전체에서 `router_init.js` 파일 검색 (핵심 IOC)
5. 모든 package.json에서 `@tanstack/setup` 의존성 또는 위 git commit hash 참조 검색
6. npm/yarn/pnpm 캐시(`~/.npm`, `~/Library/Caches/Yarn`, `~/.cache/yarn`, `~/.local/share/pnpm`)에서 `@tanstack` tarball 존재 여부 및 내부에 `router_init.js`가 포함되는지 확인
7. shell history(`~/.zsh_history`, `~/.bash_history`)와 시스템 로그에서 `litter.catbox.moe` / `getsession.org` 흔적 검색

## 출력 형식
점검 항목별로 표로 정리하고, 각 항목의 안전/위험 여부를 명시해줘.
위험 항목이 있으면 즉시 대응 방법도 알려줘.
````

## 사용 방법

1. 위 코드 블록을 복사합니다
2. Claude Code, Cursor, Copilot 등 AI 코딩 어시스턴트에 붙여넣습니다
3. AI가 자동으로 시스템을 스캔하고 결과를 표로 정리합니다

## 참고

- 악성 패키지는 이미 npm/PyPI에서 제거되었으나, 공격 시간대에 설치했다면 흔적이 남아있을 수 있습니다
- litellm 악성 버전 번호는 PyPI 공식 보고서에서 확인 필요 (접근 제한 시 보안 뉴스 참조)
- TanStack의 경우 공격 윈도우(약 6분) 밖에 설치된 패키지는 영향 없음 — 설치 시각 확인이 핵심
- 위험 항목 발견 시: 해당 패키지 즉시 삭제, 자격증명(API 키, npm/GitHub 토큰, SSH 키, 클라우드 자격증명 등) 전체 교체 권장
