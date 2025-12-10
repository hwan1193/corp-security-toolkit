# corp-security-toolkit
윈도우 기반 보안 점검/사고 대응 + 웹(RSC) 취약점 패치 검증까지 정리한 실무용 플레이북

- 대상
  - Windows 서버·클라이언트 운영 환경
  - React / Next.js 기반 웹 서비스 (RSC 사용 여부 포함)
  - ISO 27001 / ISMS-P 등 인증 대응용 증적이 필요한 보안팀/인프라팀

---

## 📁 구조

```text
.
├─ README.md
└─ docs/
   ├─ windows-ir-commands.md        # 윈도우 보안 점검/IR 명령어 치트시트
   ├─ react-next-rce-check.md       # React/Next.js RSC 취약점 점검 & rsc_check 사용 가이드
   ├─ pentest-response-flow.md      # 모의해킹팀 요청을 실제 액션으로 바꾼 대응 플로우
   └─ security-incident-handling.md # 보안사고 발생 시 보안팀 대응 절차 (초기 30분~보고까지)


RSC 점검 도구(rsc_check) 소개

도구명: RSC(React Server Components) 점검 도구 rsc_check

제공처: Piolink

보안 공지: Piolink 보안 분석 페이지

지원 OS: Linux (CentOS, Ubuntu 등)

Windows 환경에서는 WSL(Windows Subsystem for Linux) 상에서 사용 가능

형식: ELF 실행 파일 (설치 불필요, 실행만으로 점검)

특징

개인정보 수집/외부 전송 기능 없음

보조 진단용 도구 (100% 탐지 X → 참고 지표로 사용)
