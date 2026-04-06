# Supply Chain Attack Security Check

2026년 3~4월 발생한 npm/PyPI 공급망 공격 2건에 대한 점검 프롬프트입니다.
아래 코드 블록을 AI 코딩 어시스턴트(Claude Code, Cursor, Copilot 등)에 붙여넣으면 자동으로 점검합니다.

## 대상 공격

| 공격 | 패키지 | 플랫폼 | 노출 시간 |
|------|--------|--------|-----------|
| [axios 공급망 공격](https://www.stepsecurity.io/blog/axios-compromised-on-npm-malicious-versions-drop-remote-access-trojan) | axios | npm | ~2-3시간 |
| [litellm/telnyx 공급망 공격](https://blog.pypi.org/posts/2026-04-02-incident-report-litellm-telnyx-supply-chain-attack/) | litellm, telnyx | PyPI | ~2시간 32분 |

## 점검 프롬프트

````text
내 컴퓨터가 아래 2건의 npm/PyPI 공급망 공격에 영향을 받았는지 점검해줘.

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
- 위험 항목 발견 시: 해당 패키지 즉시 삭제, 자격증명(API 키, 토큰 등) 전체 교체 권장
