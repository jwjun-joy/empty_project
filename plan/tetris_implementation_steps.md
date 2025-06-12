# 테트리스 구현 단계
버전: 1.1  
대상: 핵심 개발 팀  
대상 스택: TypeScript + Vite + PixiJS + Redux Toolkit  

---

## 1. 가정 및 기본 원칙
- **클린 아키텍처**: UI → 애플리케이션 → 도메인 → 인프라 계층  
- **함수형 코어, 명령형 셸**: 결정론적 로직을 프레임워크와 분리  
- **엄격 TS**(`"strict": true`), **ESLint/Prettier** 강제  
- **TDD 우선**: UI 연결 전 단위 테스트 커버리지 90 % 이상  
- **CI 필수 통과**: 모든 PR은 린트·테스트·타입체크를 만족해야 병합 가능  

---

## 2. 소스 트리 스켈레톤
```
src/
├─ app/                 # composition roots
│  ├─ web/              # 브라우저 부트스트랩
│  └─ desktop/          # Electron main/preload
├─ domain/              # 순수 게임 로직
│  ├─ models/           # Board, Tetromino 등
│  ├─ services/         # RandomBag 등
│  └─ utils/            # 공용 헬퍼
├─ engine/              # GameLoop, Redux slices
├─ infrastructure/      # storage, audio, input 어댑터
├─ ui/                  # PixiJS 씬 & 컴포넌트
├─ config/              # constants, DI 토큰
└─ index.ts
```

---

## 3. 핵심 클래스 & 함수 명세

| 구성 요소 | 목적 | 주요 멤버 (TS) |
|-----------|------|----------------|
| `Tetromino` (domain/models) | 블록 객체 불변 모델 | `id`, `matrix`, `rotation`, `withPos(p)` |
| `Board` (domain/models) | 20×10 그리드 & 연산 | `clone()`, `merge()`, `clearLines()` |
| `RandomBag` (domain/services) | Bag-7 랜덤 생성기 | `next(): PieceType` |
| `GameState` (engine) | Redux root 상태 | `board`, `active`, `queue` 등 |
| `GameLoop` (engine) | 고정 스텝 틱커 | `start()`, `pause()`, `step(dt)` |
| `InputService` (infrastructure) | DOM 이벤트→액션 | `subscribe(cb)`, `setBindings(cfg)` |
| `AudioService` | 음악·SFX 제어 | `play(name)`, `toggleMute()` |
| `Renderer` (ui) | PixiJS 씬 그래프 | `drawBoard(state)`, `animateLock()` |

모든 **도메인** 클래스는 프레임워크 독립적이어야 하며, 테스트에서 직접 사용 가능해야 한다.

---

## 4. 단계별 구현 체크리스트

### 단계 0 – 프로젝트 부트스트랩 (1주차)
| 세부 작업 | 완료 기준(DoD) |
|-----------|---------------|
| vite+ts 템플릿 초기화, `pnpm` 워크스페이스 구성 | `src/` 진입점 빌드 성공 |
| ESLint, Prettier, Husky pre-commit 설정 | `npm run lint` 오류 0 |
| 절대경로 별칭(`@domain/*`) tsconfig 설정 | 임포트 경로가 통과 |
| Vitest + JSDOM 설치, 샘플 테스트 추가 | `npm run test` 통과 |
| GitHub Actions 워크플로우(lint/test) 작성 | PR 올리면 상태 체크 통과 |

### 단계 1 – 코어 엔진 (2–3주차)
1. **도메인 모델**  
   - `Tetromino` 팩토리 & SRS 데이터 표준화  
   - `Board` 충돌·라인 클리어 로직  
   - 단위 테스트: 7종 피스 4회전, 벽 킥, 라인 클리어 시나리오
2. **랜덤 생성기**  
   - `RandomBag` 구현, 1000개 통계 테스트(편차 ±5 %)  
3. **Redux 슬라이스**  
   - `boardSlice`, `pieceSlice`, `scoreSlice` 작성(순수 리듀서)  
4. **GameLoop 프로토타입**  
   - 고정 스텝 16 ms, CLI 하네스에서 JSON 보드 출력  
5. **DoD**: 터미널 기반 프로토타입으로 40줄 클리어 가능, 커버리지 ≥90 %

### 단계 2 – 렌더링 & 입력 (4–5주차)
| 세부 작업 | 완료 기준 |
|-----------|-----------|
| PixiJS 통합, `Renderer` 서비스 초기 버전 | 브라우저 캔버스에 보드 그리기 |
| 스프라이트 시트/팔레트 로드 | FPS 60 유지 |
| `InputService` 구현: `keydown/keyup`, DAS/ARR | 키 이동/회전 반응 |
| Redux→Pixi 바인딩 리액티브 렌더러 | 상태 변경 시 즉시 렌더 |
| FPS 오버레이, 윈도 리사이즈 대응 | 해상도 변경 시 정상 |

### 단계 3 – 점수 & 레벨 (6주차)
- `scoreSlice`: Tetris Guideline 점수표(싱글~테트리스)  
- 레벨→중력 속도 테이블, GameLoop에 적용  
- `hold` 기능 및 UI(1회 제한 규칙)  
- `NextQueue` UI(3~5개)  
- **DoD**: 100줄 플레이 시 점수·레벨 오차 0, 점수 HUD 표시

### 단계 4 – UX 다듬기 (7–8주차)
| 세부 작업 | 완료 기준 |
|-----------|-----------|
| 메인 메뉴, 일시정지, 게임오버 씬 | 모든 뷰 전환 애니 완료 |
| 설정 모달: 키 리바인딩, 볼륨, 색약 팔레트 | 로컬 저장 후 재시작 반영 |
| `AudioService`: BGM 2트랙, SFX 5종 | 토글·볼륨 슬라이더 동작 |
| 고스트 피스·하드드롭 애니메이션 | 하드드롭 시 이펙트 표시 |

### 단계 5 – 데스크톱 빌드 (9주차)
- Electron main/preload, 컨텍스트 격리  
- IPC: 하이스코어 JSON 저장, 창 크기 설정  
- Electron Builder 설정(MSI/DMG/AppImage)  
- **DoD**: Windows/macOS 빌드 후 설치·실행 성공

### 단계 6 – 테스트 & QA (10–11주차)
| 카테고리 | 목표 |
|----------|------|
| 단위 테스트 | 도메인·엔진 커버리지 ≥90 % |
| 통합 테스트 | GameLoop + Reducer 시나리오 5종 |
| E2E | Playwright: 40라인 모드 자동 클리어 |
| 성능 | 저사양 VM 평균 프레임 ≤16 ms |
| L10n | `LANG=fr` 로딩 시 미번역 키 0 |

### 단계 7 – 릴리스 (12주차)
- v1.0 태그, 변경 로그 생성  
- GitHub Releases: 웹빌드(zip) + MSI/DMG/AppImage 업로드  
- Netlify 프로덕션 배포, Cloudflare CDN 캐시 퍼지  
- **Post-Release**: 회고 & v1.1 백로그 확정  

---

## 5. 상세 작업 매트릭스 (예시)

| ID | 작업 | 담당 | 예상 시간 | 선행 |
|----|------|------|-----------|-------|
| E-01 | `Tetromino` 데이터 정의 | Alice | 4h | — |
| E-02 | `Board.merge()` 구현 | Bob | 6h | E-01 |
| E-03 | SRS 단위 테스트 | Alice | 5h | E-01 |
| E-04 | `RandomBag` 구현 | Carol | 3h | — |
| E-05 | `boardSlice` 리듀서 | Dave | 6h | E-02 |
| … | _전체 매트릭스는 Jira 보드 유지_ | | | |

---

## 6. 코딩 규약 & 리뷰 게이트
1. 모든 리듀서는 **순수 함수**; 부수 효과는 thunk/RTK listener-middleware 사용  
2. **매직 넘버 금지**: `config/constants.ts` 선언  
3. 함수 길이 30 LoC 초과 시 리뷰어 승인 필요  
4. **PR 조건**  
   - CI 통과  
   - 테스트 & 문서 포함  
   - 동료 1인 + 리드 1인 리뷰 승인  

---

## 7. 인수 기준 요약
- 동일 시드 입력 시 게임 로직 **결정적**  
- Intel UHD 620 기준 평균 프레임 ≤12 ms  
- 핵심 클래스 모두 **TSDoc** 주석  
- 릴리스 전 `main` 브랜치 **CI 3일 연속 그린**  

---

_End of file_
