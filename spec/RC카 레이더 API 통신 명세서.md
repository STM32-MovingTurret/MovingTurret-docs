# RC카 레이더 API 통신 명세서

Oct 27, 2026 · @문수

세 담당 영역 사이의 API 경계는 두 곳(통신 보드 안의 route\_bt↔route, 차량 보드 안의 vehicle\_bt↔vehicle)이며, 경계에서는 공개 헤더의 C 함수와 공통 자료형만 주고받습니다. 1\~2절은 담당 경계와 규칙, 3\~7절은 공통 자료형·API 목록·메시지 ID입니다.

## 1. 담당 범위와 API 경계

블루투스는 route\_bt와 vehicle\_bt 두 모듈(둘 다 장문수)로 나뉘어 각 보드에 들어가므로, API 경계는 통신 보드 안(route\_bt ↔ route, API-1xx)과 차량 보드 안(vehicle\_bt ↔ vehicle, API-2xx) 두 곳입니다. 나머지 연결(W1 client ↔ route, W3 radar → vehicle)은 한 사람의 담당 안에 있어 메시지 형식만 공유합니다.

&#91;embedded content: 담당 영역과 API 경계 · 경계 2곳, 링크 3개\]

| 모듈 | 보드 | 담당 | 책임 | 제공 API | 사용 API |
| --- | --- | --- | --- | --- | --- |
| `client` | PC | 장건호 | 레이더 화면, 시동·연결 표시, 수신 로그, 지연 측정 | — | MSG-1xx |
| `route` | 통신 보드 (NUCLEO #1) | 장건호 | 조이스틱 방향 판정, 시동 버튼, client 통신(W1) | 콜백 구현만 | API-1xx |
| `route_bt` | 통신 보드 (NUCLEO #1) | 장문수 | HC-05 마스터, W2 송수신과 메시지 변환, 연결 상태 | API-1xx | — |
| `vehicle_bt` | 차량 보드 (NUCLEO #2) | 장문수 | HC-05 슬레이브, W2 송수신과 메시지 변환, 연결 상태 | API-2xx | — |
| `vehicle` | 차량 보드 (NUCLEO #2) | 김인규 | 모터 드라이버 구동, 속도 보정, 두절 정지, radar 결과 수신(W3) | 콜백 구현만 | API-2xx |
| `radar` | 레이더 보드 (NUCLEO #3) | 김인규 | 서보, 초음파, 장애물 판정, vehicle로 결과 송신(W3) | — | MSG-3xx |

| 연결 | 양 끝 | 형태 | ID | 명세 담당 |
| --- | --- | --- | --- | --- |
| API-1xx | 통신 보드 안: route\_bt ↔ route | C 함수 + 콜백 | API-101\~ | 장문수(제공), 장건호(검토) |
| API-2xx | 차량 보드 안: vehicle\_bt ↔ vehicle | C 함수 + 콜백 | API-201\~ | 장문수(제공), 김인규(검토) |
| W1 | client ↔ route (USB) | 텍스트 메시지 | MSG-1xx | 장건호 |
| W2 | route\_bt ↔ vehicle\_bt (블루투스) | 텍스트 메시지 | MSG-2xx | 장문수 |
| W3 | radar → vehicle (UART) | 텍스트 메시지 | MSG-3xx | 김인규 |
| 공통 자료형 | 모든 모듈 | C 헤더 `common/api_types.h` | TYPE-xxx, ERR-xxx | 3인 공동 |

## 2. API 명세 규칙

다른 담당자의 코드는 공개 헤더에 선언된 함수로만 부르고, 데이터를 받는 쪽은 콜백을 등록해서 받습니다. 이 규칙만 지키면 세 사람이 서로의 내부 구현을 몰라도 따로 개발하고 나중에 그대로 합칠 수 있습니다.

### 2.1 기본 원칙

1. **공개 헤더만 경계를 넘습니다.** 다른 사람 모듈의 내부 변수, static 함수, UART·TIM 핸들은 직접 쓰지 않습니다. 쓸 수 있는 것은 `<모듈>_api.h`에 선언된 함수와 `common/api_types.h`의 자료형뿐입니다.
2. **경계에서는 C 자료형으로 주고받습니다.** 텍스트 메시지(`D,F` 같은 와이어 포맷)는 그 링크 담당자만 다룹니다. 예를 들어 route는 `D,F` 문자열을 만들지 않고 `route_bt_send_drive(DRIVE_FWD)`를 호출합니다.
3. **제공자가 헤더를 소유합니다.** API를 구현하는 쪽(제공자)이 헤더를 작성·관리하고, 쓰는 쪽(사용자)은 include만 합니다.
4. **받는 데이터는 콜백으로 전달합니다.** 제공자가 사용자 함수를 직접 부르지 않고, 사용자가 init 때 콜백을 등록합니다. 그래서 route\_bt·vehicle\_bt는 route·vehicle 헤더를 include하지 않습니다.
5. **모든 API는 비블로킹입니다.** 요청은 큐에 넣고 바로 반환하며, 결과가 필요하면 콜백이나 상태 조회 함수로 알립니다.

### 2.2 ID 규칙

| 접두어 | 대상 | 번호 범위 | 예 |
| --- | --- | --- | --- |
| API-0xx | 모든 모듈이 같은 모양으로 제공하는 함수(init·process) | 001\~099 | API-001 `<모듈>_init` |
| API-1xx | 통신 보드 안 경계: route\_bt(장문수) ↔ route(장건호) | 101\~199 | API-101 |
| API-2xx | 차량 보드 안 경계: vehicle\_bt(장문수) ↔ vehicle(김인규) | 201\~299 | API-201 |
| TYPE-xxx | `common/api_types.h`의 공통 자료형 | 001\~099 | TYPE-001 `drive_cmd_t` |
| MSG-1xx·2xx·3xx | 링크 메시지(W1 client↔route · W2 route\_bt↔vehicle\_bt · W3 radar→vehicle) | 링크별 01\~99 | MSG-201 |
| ERR-xxx | 오류 코드 | 001\~099 | ERR-001 |

- 한 API ID = 함수 하나입니다. 콜백 등록 함수와 콜백 형은 각각 별도 ID를 받습니다.
- 폐기한 ID는 다시 쓰지 않고, 표에 "폐기"와 사유를 남깁니다.
- 코드 주석, 커밋 메시지, 이슈 제목에도 같은 ID와 모듈 이름을 씁니다. 예: `feat(route_bt): API-101 route_bt_send_drive 구현`

### 2.3 명명 규칙

| 대상 | 규칙 | 예 |
| --- | --- | --- |
| 모듈 이름 | 아래 6개만 쓰며 무조건 소문자 | `client`, `route`, `route_bt`, `vehicle`, `vehicle_bt`, `radar` |
| 폴더·파일 | 모듈 이름 그대로 소문자. 공개 헤더 `<모듈>_api.h`, 구현 `<모듈>.c`, 내부 헤더 `<모듈>_internal.h` | `route_bt/route_bt_api.h` |
| 모듈 안 하위 파일 | `<모듈>_<하위 기능>.c` | `route_joystick.c`, `vehicle_motor.c`, `radar_servo.c` |
| 공개 함수 | `<모듈>_<동사>_<대상>`, 소문자 snake\_case | `route_bt_send_drive()`, `vehicle_bt_is_linked()` |
| 콜백 등록 함수 | `<제공 모듈>_set_<이벤트>_cb()` | `vehicle_bt_set_drive_cb()` |
| 콜백 형 | `<제공 모듈>_<이벤트>_cb_t` | `vehicle_bt_drive_cb_t` |
| 자료형 | 소문자 snake\_case + `_t`. 공통 형은 모듈 이름 없이 | `drive_cmd_t`, `scan_point_t` |
| 열거형 값·매크로 | 대문자 + 형·모듈 이름. 이 두 가지만 대문자이고 나머지 이름은 모두 소문자 | `DRIVE_FWD`, `ROUTE_BT_TX_QUEUE_LEN` |
| 단위가 있는 값 | 이름 끝에 단위 | `dist_cm`, `angle_deg`, `timeout_ms` |
| client(Python) | 파일·함수·변수 소문자 snake\_case, 파일은 `client_<기능>.py` | `client_radar_view.py` |

이름은 반드시 자기 모듈 이름으로 시작하며, 다른 모듈 이름으로 시작하는 이름은 만들지 않습니다. `route`와 `route_bt`, `vehicle`과 `vehicle_bt`는 앞부분이 겹치므로, route·vehicle 담당은 `_bt_`가 붙는 이름(`route_bt_…`, `vehicle_bt_…`)을 쓰지 않습니다.

열거형 값은 `typedef enum { DRIVE_STOP, DRIVE_FWD, … } drive_cmd_t;`의 각 항목이고, 매크로는 `#define`으로 정의한 상수입니다. 이 둘만 대문자로 두면 코드에서 변수·함수(소문자)와 고정된 값(대문자)이 바로 구분되며, HAL 코드(`GPIO_PIN_SET`, `HAL_OK`)와도 관례가 같습니다.

### 2.4 자료형 규칙

- 정수는 `<stdint.h>` 고정 폭 형만 씁니다(`uint8_t`, `int16_t`, `uint32_t`). 경계에서 `int`, `long`은 쓰지 않습니다.
- 실수(`float`) 대신 정수 + 단위로 표현합니다. 예: 거리 `int16_t dist_cm`.
- 참·거짓은 `<stdbool.h>`의 `bool`입니다.
- 값이 정해진 집합(주행 명령, 상태)은 enum으로 정의하고, 마지막에 `_COUNT`를 두어 범위 검사에 씁니다.
- 특수값은 이름 붙인 상수로 정의합니다. 예: `#define SCAN_DIST_INVALID (-1)`
- 구조체는 값 또는 `const` 포인터로 넘깁니다. 구조체를 바이트 그대로 와이어로 보내지 않으므로 패킹·엔디언은 가정하지 않습니다.
- 공통 자료형은 `common/api_types.h` 한 곳에만 정의합니다. 같은 뜻의 형을 모듈마다 따로 만들지 않습니다.

### 2.5 반환값과 오류 규칙

동작을 요청하는 함수는 `api_status_t`를 반환하고, 상태를 조회하는 함수는 값(`bool` 등)을 바로 반환합니다.

| 값 | 이름 | ID | 의미 | 호출자 처리 |
| --- | --- | --- | --- | --- |
| 0 | `API_OK` | — | 성공(송신 큐에 들어간 것까지) | 없음 |
| 1 | `API_ERR_PARAM` | ERR-001 | 인자가 허용 범위 밖 | 호출 코드 버그. 수정 대상 |
| 2 | `API_ERR_BUSY` | ERR-002 | 송신 큐가 가득 참 | 이번 것은 버리고 다음 주기에 다시 보냄 |
| 3 | `API_ERR_LINK` | ERR-003 | 링크 끊김 | 버리고 연결 끊김을 표시 |
| 4 | `API_ERR_NOT_INIT` | ERR-004 | init 전에 호출 | 초기화 순서 버그. 수정 대상 |

- 오류를 반환할 때 제공자는 아무 동작도 하지 않은 상태여야 합니다(일부만 보내는 일 없음).
- 호출자는 반환값을 버리지 않습니다. 최소한 오류 카운터를 올립니다.
- 제공자는 디버그용으로 오류 카운터 조회 함수 `<접두어>_get_err_count()`를 둡니다.

### 2.6 호출 규칙

**초기화와 주기 실행**

- 모든 모듈은 `<모듈>_init()`(API-001)과 `<모듈>_process()`(API-002)를 제공합니다.
- `main()`은 CubeMX 생성 초기화 → 제공자 init → 사용자 init(콜백 등록) 순으로 부르고, 슈퍼 루프에서 모든 `_process()`를 매 바퀴 부릅니다.
- 통신 보드: `route_bt_init()` → `route_init()`. 차량 보드: `vehicle_bt_init()` → `vehicle_init()`. 레이더 보드: `radar_init()`.

**실행 문맥**

- 모든 API는 1 ms 안에 반환합니다. `HAL_Delay()`, 플래그 대기 루프, 블로킹 HAL 송수신은 쓰지 않습니다.
- 콜백은 제공자의 `_process()` 안, 즉 메인 루프 문맥에서만 부릅니다. ISR에서 콜백을 부르지 않습니다.
- 콜백 안에서는 짧게 처리하고 블로킹하지 않습니다. 콜백 안에서 같은 제공자의 송신 API를 부르는 것은 허용합니다.
- ISR에서 불러도 되는 API는 명세에 "ISR 가능"으로 표시한 것뿐입니다. 표시가 없으면 메인 루프에서만 부릅니다.

**데이터와 메모리**

- 포인터 인자는 함수가 반환되면 제공자가 더 이상 쓰지 않습니다. 계속 필요하면 제공자가 복사합니다.
- 콜백에 넘어온 포인터는 콜백이 반환되면 무효입니다. 필요하면 사용자가 복사합니다.
- 모듈 사이에 전역 변수로 데이터를 공유하지 않습니다. 동적 할당(`malloc`)은 쓰지 않습니다.

### 2.7 명세 작성 양식

API 하나마다 아래 표 하나를 씁니다. 예시 값은 양식 설명용이며 확정된 API가 아닙니다.

| 항목 | 작성 내용 | 예시 |
| --- | --- | --- |
| ID · 이름 | API ID와 함수 이름 | API-101 `route_bt_send_drive` |
| 제공 / 사용 | 담당자(모듈) | 장문수(route\_bt) / 장건호(route) |
| 선언 | 헤더에 들어갈 선언 그대로 | `api_status_t route_bt_send_drive(drive_cmd_t cmd);` |
| 설명 | 무엇을 하고 언제 부르는지 1\~2문장 | 주행 명령 1개를 차량 보드로 보낸다. 100 ms마다 부른다. |
| 매개변수 | 이름 · 형 · 단위 · 허용 범위 | `cmd` · `drive_cmd_t` · — · `DRIVE_STOP`\~`DRIVE_RIGHT` |
| 반환값 | 나올 수 있는 값과 뜻 | `API_OK`, `API_ERR_PARAM`, `API_ERR_NOT_INIT` |
| 호출 문맥 | 메인 루프 / ISR 가능 여부, 블로킹 여부 | 메인 루프, 비블로킹 |
| 타이밍 | 호출 주기, 지연 요구 | 호출 후 vehicle 콜백까지 150 ms 이내(안) |
| 오류·경계 처리 | 끊김, 범위 밖 값일 때 동작 | 미송신 명령은 새 명령으로 덮어씀. 연결이 끊겨도 `API_OK` |
| 관련 요구사항 · 시험 | FR·NFR ID / TC ID | FR-002, NFR-002 / TC-003 |

링크 메시지(MSG)는 다음 항목으로 씁니다: ID · 링크 · 방향 · 형식과 예 · 필드(이름·범위·단위) · 보내는 때 · 받는 쪽 처리 · 오류 시 처리.

### 2.8 변경 관리

- 공개 헤더와 `api_types.h`는 PR로만 바꿉니다. 사용자 쪽 담당자의 리뷰 승인이 있어야 병합합니다.

| 변경 대상 | 승인자 |
| --- | --- |
| API-1xx (route\_bt ↔ route) | 장문수, 장건호 |
| API-2xx (vehicle\_bt ↔ vehicle) | 장문수, 김인규 |
| TYPE-xxx, ERR-xxx, API-0xx | 3인 모두 |
| MSG-1xx / 2xx / 3xx | 해당 링크 담당자(공유만) |

- 명세서 버전은 vX.Y로 씁니다. 선언·의미가 바뀌어 기존 코드가 깨지면 X를, 추가만 하면 Y를 올립니다.
- 통합 시작 이후에는 기존 API의 선언을 바꾸지 않습니다. 필요하면 새 ID를 추가하고 기존 것은 폐기 표시합니다.
- 모든 변경은 3절 변경 이력에 버전, 날짜, ID, 내용, 작성자를 남깁니다.

### 2.9 스텁과 시험

- 제공자는 구현 전에 공개 헤더와 스텁을 먼저 올립니다. 스텁은 `API_OK`를 반환하고 받은 인자를 디버그 UART로 출력합니다. 예: `[route_bt] send_drive F`
- 사용자는 스텁으로 빌드·동작을 확인하고, 제공자는 시리얼 터미널 입력으로 콜백을 흉내 내는 시험 코드를 둡니다.
- 명세의 "관련 시험" 칸에는 그 API를 혼자 확인하는 방법을 하나 이상 적습니다.

### 2.10 저장소 배치

| 폴더 | 내용 | 담당 |
| --- | --- | --- |
| `common/` | `api_types.h`(TYPE·ERR), 공통 유틸 | 3인 공동 |
| `client/` | PC 프로그램(Python) | 장건호 |
| `route/` | 통신 보드 CubeIDE 프로젝트(.ioc)와 route 모듈 | 장건호 |
| `route_bt/` | route\_bt 모듈. 통합 때 route 프로젝트에 병합 | 장문수 |
| `vehicle/` | 차량 보드 CubeIDE 프로젝트(.ioc)와 vehicle 모듈 | 김인규 |
| `vehicle_bt/` | vehicle\_bt 모듈. 통합 때 vehicle 프로젝트에 병합 | 장문수 |
| `radar/` | 레이더 보드 CubeIDE 프로젝트(.ioc)와 radar 모듈 | 김인규 |
| `docs/` | 요구사항·설계·API·테스트 문서 | 3인 공동 |

### 2.11 디버그 출력

디버그 출력은 디버그 전용 UART로만 냅니다. 통신용 UART(client 연결, HC-05, 레이더 연결)에는 메시지 명세에 없는 글자를 한 글자도 보내지 않습니다.

| 보드 | 디버그 UART(안) | 연결 |
| --- | --- | --- |
| 통신 보드 (route, route\_bt) | USART6 TX (PC6) | USB-UART 변환기(3.3 V) 별도 필요. USART2는 client 연결용 |
| 차량 보드 (vehicle, vehicle\_bt) | USART2 | ST-LINK 가상 COM (USB 케이블) |
| 레이더 보드 (radar) | USART2 | ST-LINK 가상 COM (USB 케이블) |

- 출력은 공통 함수 `common_log("<모듈>", "<내용>", …)`(`common/common_log.h`)만 씁니다. 줄 형식은 `[route_bt] send_drive F`처럼 `[모듈] 내용`입니다.
- `common_log()`는 비블로킹입니다. 버퍼가 차면 그 줄은 버립니다.
- `#define COMMON_LOG_ENABLE 0`이면 출력 코드가 빠진 채로 컴파일됩니다.
- 디버그 UART 핀은 보드 담당자가 각 .ioc에 설정합니다.

### 2.12 모듈 간 책임 결정

| 항목 | 결정 | 이유 |
| --- | --- | --- |
| 두절 정지 판단 | vehicle이 마지막 주행 명령을 받은 시각으로 판단. 500 ms(안) 넘게 없으면 정지 | 블루투스가 연결돼 있어도 route가 멈춘 경우까지 잡음 |
| 주행 명령 반복 | route가 100 ms마다 주행 명령 송신(멈춤 포함) | vehicle 두절 감시의 기준 |
| 지연 측정(P) 되돌림 | vehicle\_bt 안에서 바로 되돌려 보냄. vehicle에는 전달하지 않음 | vehicle이 시험용 메시지를 몰라도 됨 |
| 링크 상태 | route\_bt·vehicle\_bt가 `_is_linked()` 조회 함수 제공 | 화면 표시·디버그용. 안전 판단에는 쓰지 않음 |

### 2.13 개발과 병합 방식

- 브랜치는 개발자마다 각자 사용합니다(이름 자유). main에는 PR로 병합합니다.
- .ioc(핀·주변장치 설정)는 보드 담당자가 소유합니다. 통신 보드는 장건호, 차량·레이더 보드는 김인규입니다.
- route\_bt·vehicle\_bt는 자기 폴더에서 개발한 뒤, 통합 때 해당 보드 프로젝트(.ioc가 있는 프로젝트)에 코드를 병합합니다.
- 병합하는 쪽은 .ioc를 직접 고치지 않고, 필요한 주변장치 설정(예: USART1 9600 bps, RX 인터럽트, TX DMA)을 보드 담당자에게 요청합니다. 보드 담당자가 .ioc에 반영합니다.
- CubeMX가 다시 생성해도 지워지지 않도록, 병합한 코드는 모듈 폴더 파일과 `main.c`의 `USER CODE BEGIN/END` 구간(init·process 호출)에만 둡니다.

## 3. 공통 자료형 (TYPE·ERR)

모든 모듈은 `common/api_types.h` 하나만 include해서 아래 자료형을 씁니다. 3인 공동 소유이며 바꿀 때는 3인 모두 승인해야 합니다(2.8).

```c
/* common/api_types.h */
#ifndef API_TYPES_H
#define API_TYPES_H

#include <stdint.h>
#include <stdbool.h>

/* TYPE-001 : 요청 함수의 반환값 (ERR-001~004) */
typedef enum {
    API_OK = 0,
    API_ERR_PARAM,      /* ERR-001 인자 범위 밖 */
    API_ERR_BUSY,       /* ERR-002 송신 큐 가득 참 */
    API_ERR_LINK,       /* ERR-003 링크 끊김(확실히 알 때만) */
    API_ERR_NOT_INIT    /* ERR-004 init 전 호출 */
} api_status_t;

/* TYPE-002 : 주행 명령 */
typedef enum {
    DRIVE_STOP = 0,
    DRIVE_FWD,
    DRIVE_BWD,
    DRIVE_LEFT,
    DRIVE_RIGHT,
    DRIVE_CMD_COUNT     /* 범위 검사용, 명령 아님 */
} drive_cmd_t;

/* TYPE-003 : 레이더 측정점 1개 */
#define SCAN_ANGLE_MAX_DEG  180
#define SCAN_DIST_MIN_CM    2
#define SCAN_DIST_MAX_CM    200
#define SCAN_DIST_INVALID   (-1)

typedef struct {
    uint8_t angle_deg;  /* 0~180 */
    int16_t dist_cm;    /* 2~200, 측정 불가는 SCAN_DIST_INVALID */
    bool    obstacle;   /* true = 장애물 */
} scan_point_t;

/* TYPE-004 : UART 이벤트 종류 (API-004에서 사용) */
typedef enum {
    UART_EVT_RX = 0,    /* 수신 완료 */
    UART_EVT_TX,        /* 송신 완료 */
    UART_EVT_ERR        /* 오류 */
} uart_evt_t;

#endif /* API_TYPES_H */
```

| ID | 이름 | 값 | 쓰는 API |
| --- | --- | --- | --- |
| TYPE-001 | `api_status_t` | 반환값 5종. 처리 방법은 2.5절 | 모든 요청 함수 |
| TYPE-002 | `drive_cmd_t` | 정지 0, 전진 1, 후진 2, 좌회전 3, 우회전 4 | API-104, API-204 |
| TYPE-003 | `scan_point_t` | 각도(deg), 거리(cm), 장애물 여부 | API-106, API-206 |
| TYPE-004 | `uart_evt_t` | 수신 완료, 송신 완료, 오류 | API-004, API-103, API-203 |

`scan_point_t`가 유효하려면 `angle_deg`가 180 이하, `dist_cm`이 2\~200 또는 -1이어야 합니다. `dist_cm`이 -1이면 `obstacle`은 항상 `false`입니다.

## 4. 공통 함수 규약 (API-0xx)

모든 모듈은 아래 형태의 함수를 같은 이름 규칙으로 제공합니다. 보드 담당자는 `main.c`에서 이 함수들만 불러 모듈을 연결합니다.

| ID | 선언 형태 | 설명 | 호출 문맥 |
| --- | --- | --- | --- |
| API-001 | `void <모듈>_init(<HAL 핸들 …>);` | 한 번 호출. 쓰는 HAL 핸들(UART 등)을 인자로 받음. 핸들은 보드 담당자가 `main.c`에서 넘김 | `main()`, 루프 전 |
| API-002 | `void <모듈>_process(void);` | 루프 한 바퀴마다 호출. 받은 줄 처리, 콜백 호출, 주기 작업 | 메인 루프 |
| API-003 | `uint32_t <모듈>_get_err_count(void);` | 누적 오류 수(형식 오류, 큐 넘침 등) | 메인 루프 |
| API-004 | `void <모듈>_uart_event_isr(UART_HandleTypeDef *huart, uart_evt_t evt);` | HAL UART 콜백을 모듈로 전달. 자기 UART가 아니면 아무것도 하지 않음. UART를 쓰는 모듈만 제공 | ISR 전용 |

HAL의 `HAL_UART_RxCpltCallback()` 같은 콜백은 한 프로젝트에 하나만 둘 수 있습니다. 그래서 보드 담당자가 `main.c`에 하나만 두고, UART를 쓰는 모든 모듈의 API-004로 나눠 줍니다. 모듈은 HAL 콜백 함수를 직접 정의하지 않습니다.

**통신 보드 `main.c` 통합 예시** (차량 보드는 `route_bt`→`vehicle_bt`, `route`→`vehicle`로 바꾸면 같습니다)

```c
/* USER CODE BEGIN Includes */
#include "route_bt_api.h"
#include "route_api.h"
/* USER CODE END Includes */

  /* USER CODE BEGIN 2 */
  route_bt_init(&huart1);          /* 제공자 먼저 (HC-05 UART) */
  route_init(&huart2, &hadc1);     /* 사용자: 여기서 콜백 등록. 인자는 route 담당이 정함 */
  /* USER CODE END 2 */

  while (1)
  {
    /* USER CODE END WHILE */
    /* USER CODE BEGIN 3 */
    route_bt_process();
    route_process();
  }
  /* USER CODE END 3 */

/* USER CODE BEGIN 4 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    route_bt_uart_event_isr(huart, UART_EVT_RX);
    route_uart_event_isr(huart, UART_EVT_RX);
}

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    route_bt_uart_event_isr(huart, UART_EVT_TX);
    route_uart_event_isr(huart, UART_EVT_TX);
}

void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
    route_bt_uart_event_isr(huart, UART_EVT_ERR);
    route_uart_event_isr(huart, UART_EVT_ERR);
}
/* USER CODE END 4 */
```

## 5. API-1xx: route\_bt ↔ route (통신 보드)

route\_bt(장문수)가 제공하고 route(장건호)가 씁니다. route는 주행 명령과 지연 측정 번호를 보내고, 레이더 측정점과 되돌아온 번호는 콜백으로 받습니다. 공개 헤더는 `route_bt/route_bt_api.h`입니다.

| ID | 선언 | 설명 | 반환 | 문맥 |
| --- | --- | --- | --- | --- |
| API-101 | `void route_bt_init(UART_HandleTypeDef *huart);` | HC-05가 연결된 UART(USART1)를 받아 수신 시작 | — | `main()` |
| API-102 | `void route_bt_process(void);` | 받은 줄 해석, 콜백 호출, 송신 큐 처리 | — | 메인 루프 |
| API-103 | `void route_bt_uart_event_isr(UART_HandleTypeDef *huart, uart_evt_t evt);` | HAL UART 콜백 전달(API-004) | — | ISR 전용 |
| API-104 | `api_status_t route_bt_send_drive(drive_cmd_t cmd);` | 주행 명령 1개를 차량으로 보냄 | OK, PARAM, NOT\_INIT | 메인 루프 |
| API-105 | `api_status_t route_bt_send_ping(uint16_t seq);` | 지연 측정 번호를 차량으로 보냄 | OK, BUSY, NOT\_INIT | 메인 루프 |
| API-106 | `typedef void (*route_bt_scan_cb_t)(const scan_point_t *pt);` | 레이더 측정점 수신 콜백 형 | — | — |
| API-107 | `void route_bt_set_scan_cb(route_bt_scan_cb_t cb);` | 측정점 콜백 등록. NULL이면 해제 | — | `route_init()` 안 |
| API-108 | `typedef void (*route_bt_ping_cb_t)(uint16_t seq);` | 되돌아온 번호 수신 콜백 형 | — | — |
| API-109 | `void route_bt_set_ping_cb(route_bt_ping_cb_t cb);` | 되돌림 콜백 등록. NULL이면 해제 | — | `route_init()` 안 |
| API-110 | `bool route_bt_is_linked(void);` | 블루투스 연결 상태(표시용) | true / false | 메인 루프 |
| API-111 | `uint32_t route_bt_get_err_count(void);` | 누적 오류 수(API-003) | 개수 | 메인 루프 |

콜백 등록 함수(API-107, API-109)는 실패할 일이 없어 `void`를 반환합니다.

### 5.1 API-104 route\_bt\_send\_drive

| 항목 | 내용 |
| --- | --- |
| 제공 / 사용 | 장문수(route\_bt) / 장건호(route) |
| 선언 | `api_status_t route_bt_send_drive(drive_cmd_t cmd);` |
| 설명 | 주행 명령 1개를 송신 큐에 넣고 바로 반환한다. route는 시동 상태와 관계없이 100 ms마다 부른다(시동 OFF면 `DRIVE_STOP`). |
| 매개변수 | `cmd` · `drive_cmd_t` · — · `DRIVE_STOP`\~`DRIVE_RIGHT` |
| 반환값 | `API_OK` 큐에 넣음 / `API_ERR_PARAM` `cmd`가 `DRIVE_CMD_COUNT` 이상 / `API_ERR_NOT_INIT` init 전 |
| 호출 문맥 | 메인 루프, 비블로킹 |
| 타이밍 | 100 ms마다 호출. 호출부터 vehicle의 주행 콜백(API-204)까지 150 ms 이내(안) |
| 오류·경계 처리 | 아직 보내지 않은 주행 명령이 큐에 있으면 새 명령으로 덮어씁니다. 그래서 `API_ERR_BUSY`는 나오지 않는다. 연결이 끊겨 있어도 `API_OK`(HC-05가 버림). |
| 관련 요구사항 · 시험 | FR-002, FR-003, FR-006, NFR-002 / TC-003, TC-004, TC-014 |

### 5.2 API-105 route\_bt\_send\_ping

| 항목 | 내용 |
| --- | --- |
| 제공 / 사용 | 장문수(route\_bt) / 장건호(route) |
| 선언 | `api_status_t route_bt_send_ping(uint16_t seq);` |
| 설명 | client가 보낸 지연 측정 번호를 차량으로 보낸다. vehicle\_bt가 같은 번호를 되돌려 보내면 API-108 콜백으로 알린다. |
| 매개변수 | `seq` · `uint16_t` · — · 0\~65535 |
| 반환값 | `API_OK` / `API_ERR_BUSY` 큐 가득 참 / `API_ERR_NOT_INIT` |
| 호출 문맥 | 메인 루프, 비블로킹 |
| 타이밍 | 시험 때만. client 기준 왕복 300 ms 이하(NFR-001) |
| 오류·경계 처리 | `API_ERR_BUSY`면 route는 그 번호를 버린다. 돌아오지 않은 번호는 client가 시간 초과로 처리하며, route\_bt는 다시 보내지 않는다. |
| 관련 요구사항 · 시험 | NFR-001 / TC-013 |

### 5.3 API-106·107 레이더 측정점 콜백

| 항목 | 내용 |
| --- | --- |
| 제공 / 사용 | 장문수(route\_bt) / 장건호(route) |
| 선언 | `typedef void (*route_bt_scan_cb_t)(const scan_point_t *pt);` / `void route_bt_set_scan_cb(route_bt_scan_cb_t cb);` |
| 설명 | 차량에서 측정점 1개가 오면 `route_bt_process()` 안에서 콜백을 부른다. route는 이 값을 client로 보낸다(MSG-102). |
| 매개변수 | `pt` · `const scan_point_t *` · deg, cm · TYPE-003의 유효 범위(검사를 통과한 값만 전달) |
| 반환값 | 없음 |
| 호출 문맥 | 메인 루프(`route_bt_process()` 안). `pt`는 콜백이 끝나면 무효이므로 필요하면 복사 |
| 타이밍 | 약 100 ms마다 1회(레이더 각도당) |
| 오류·경계 처리 | 형식이 틀린 줄은 route\_bt가 버리고 콜백을 부르지 않는다. 콜백이 등록돼 있지 않으면 버린다. |
| 관련 요구사항 · 시험 | FR-009, FR-010 / TC-011, TC-012 |

### 5.4 API-108·109 지연 측정 되돌림 콜백

| 항목 | 내용 |
| --- | --- |
| 제공 / 사용 | 장문수(route\_bt) / 장건호(route) |
| 선언 | `typedef void (*route_bt_ping_cb_t)(uint16_t seq);` / `void route_bt_set_ping_cb(route_bt_ping_cb_t cb);` |
| 설명 | 차량에서 되돌아온 번호를 알린다. route는 같은 번호를 client로 보낸다(MSG-103). |
| 매개변수 | `seq` · `uint16_t` · — · 0\~65535 |
| 호출 문맥 | 메인 루프(`route_bt_process()` 안) |
| 관련 요구사항 · 시험 | NFR-001 / TC-013 |

### 5.5 API-110 route\_bt\_is\_linked

블루투스 연결 여부를 돌려줍니다. 판단 방법은 route\_bt 내부에서 정합니다(안: HC-05 STATE 핀, 또는 최근 2 s 안에 수신이 있었는지). 화면 표시·디버그용이며, 주행 명령을 보낼지 말지를 이 값으로 정하지 않습니다(2.12).

## 6. API-2xx: vehicle\_bt ↔ vehicle (차량 보드)

vehicle\_bt(장문수)가 제공하고 vehicle(김인규)이 씁니다. vehicle은 주행 명령을 콜백으로 받고, 레이더 측정점을 보냅니다. 지연 측정(P) 되돌림은 vehicle\_bt 안에서 끝나므로 API가 없습니다. 공개 헤더는 `vehicle_bt/vehicle_bt_api.h`입니다.

| ID | 선언 | 설명 | 반환 | 문맥 |
| --- | --- | --- | --- | --- |
| API-201 | `void vehicle_bt_init(UART_HandleTypeDef *huart);` | HC-05가 연결된 UART(USART1)를 받아 수신 시작 | — | `main()` |
| API-202 | `void vehicle_bt_process(void);` | 받은 줄 해석, 콜백 호출, P 되돌림, 송신 큐 처리 | — | 메인 루프 |
| API-203 | `void vehicle_bt_uart_event_isr(UART_HandleTypeDef *huart, uart_evt_t evt);` | HAL UART 콜백 전달(API-004) | — | ISR 전용 |
| API-204 | `typedef void (*vehicle_bt_drive_cb_t)(drive_cmd_t cmd);` | 주행 명령 수신 콜백 형 | — | — |
| API-205 | `void vehicle_bt_set_drive_cb(vehicle_bt_drive_cb_t cb);` | 주행 콜백 등록. NULL이면 해제 | — | `vehicle_init()` 안 |
| API-206 | `api_status_t vehicle_bt_send_scan(const scan_point_t *pt);` | 레이더 측정점 1개를 통신 보드로 보냄 | OK, PARAM, BUSY, NOT\_INIT | 메인 루프 |
| API-207 | `bool vehicle_bt_is_linked(void);` | 블루투스 연결 상태(표시·디버그용) | true / false | 메인 루프 |
| API-208 | `uint32_t vehicle_bt_get_err_count(void);` | 누적 오류 수(API-003) | 개수 | 메인 루프 |

### 6.1 API-204·205 주행 명령 콜백

| 항목 | 내용 |
| --- | --- |
| 제공 / 사용 | 장문수(vehicle\_bt) / 김인규(vehicle) |
| 선언 | `typedef void (*vehicle_bt_drive_cb_t)(drive_cmd_t cmd);` / `void vehicle_bt_set_drive_cb(vehicle_bt_drive_cb_t cb);` |
| 설명 | 통신 보드에서 주행 명령이 오면 `vehicle_bt_process()` 안에서 콜백을 부른다. 같은 명령이 반복돼도 매번 부른다. vehicle은 콜백이 불린 시각으로 두절 정지를 판단한다(2.12). |
| 매개변수 | `cmd` · `drive_cmd_t` · — · `DRIVE_STOP`\~`DRIVE_RIGHT`(검사를 통과한 값만 전달) |
| 반환값 | 없음 |
| 호출 문맥 | 메인 루프(`vehicle_bt_process()` 안). 콜백 안에서는 모터 출력 변경과 `HAL_GetTick()` 기록 정도만 한다. |
| 타이밍 | 약 100 ms마다 1회. route의 API-104 호출부터 150 ms 이내(안) |
| 오류·경계 처리 | 형식이 틀린 줄은 버리고 콜백을 부르지 않는다. vehicle 입장에서는 명령이 안 온 것과 같아, 500 ms(안) 뒤 정지한다. |
| 관련 요구사항 · 시험 | FR-004, FR-006 / TC-005, TC-007 |

### 6.2 API-206 vehicle\_bt\_send\_scan

| 항목 | 내용 |
| --- | --- |
| 제공 / 사용 | 장문수(vehicle\_bt) / 김인규(vehicle) |
| 선언 | `api_status_t vehicle_bt_send_scan(const scan_point_t *pt);` |
| 설명 | 레이더 측정점 1개를 통신 보드로 보낸다. vehicle은 radar에서 받은 메시지(MSG-301)를 `scan_point_t`로 바꾼 뒤 부른다. |
| 매개변수 | `pt` · `const scan_point_t *` · deg, cm · NULL 불가, TYPE-003 유효 범위 |
| 반환값 | `API_OK` 큐에 넣음 / `API_ERR_PARAM` NULL이거나 범위 밖 / `API_ERR_BUSY` 큐 가득 참(큐 8줄, 안) / `API_ERR_NOT_INIT` |
| 호출 문맥 | 메인 루프, 비블로킹. vehicle\_bt는 `*pt`를 복사한 뒤 반환한다. |
| 타이밍 | 약 100 ms마다. 호출부터 route의 측정점 콜백(API-106)까지 150 ms 이내(안) |
| 오류·경계 처리 | `API_ERR_BUSY`면 vehicle은 그 측정점을 버린다(다음 각도가 곳 옴). 연결이 끊겨 있어도 `API_OK`. |
| 관련 요구사항 · 시험 | FR-009 / TC-011 |

### 6.3 API-207 vehicle\_bt\_is\_linked

5.5절과 같습니다. 두절 정지 판단에는 쓰지 않고, vehicle은 주행 콜백이 불린 시각만으로 정지를 판단합니다.

## 7. 링크 메시지 ID (MSG)

와이어 메시지는 각 링크 담당자의 내부 사항이지만, 시험·디버깅 때 서로 읽을 수 있도록 ID와 API 대응만 정리합니다. 형식 규칙(ASCII 한 줄, 쉼표 구분, 32자 이하)은 [아키텍처 및 통신 명세](https://claude.ai/artifact/4bbac7gvz8nXfdHmfC72Wh) 3.2·3.3절을 따릅니다.

| ID | 링크 | 방향 | 형식과 예 | 대응 API |
| --- | --- | --- | --- | --- |
| MSG-101 | W1 USB | route → client | `E,<시동 1 또는 0>` 예 `E,1` | route 내부 |
| MSG-102 | W1 USB | route → client | `R,<각도>,<거리>,<장애물>` 예 `R,45,23,1` | API-106 콜백 값을 그대로 |
| MSG-103 | W1 USB | client → route → client | `P,<번호>` 예 `P,17` | API-105 송신, API-108 수신 |
| MSG-201 | W2 블루투스 | route\_bt → vehicle\_bt | `D,<F·B·L·R·S 중 하나>` 예 `D,F` | API-104 → API-204 |
| MSG-202 | W2 블루투스 | vehicle\_bt → route\_bt | `R,<각도>,<거리>,<장애물>` | API-206 → API-106 |
| MSG-203 | W2 블루투스 | route\_bt → vehicle\_bt → route\_bt | `P,<번호>` | API-105 → vehicle\_bt 되돌림 → API-108 |
| MSG-301 | W3 UART | radar → vehicle | `R,<각도>,<거리>,<장애물>` | vehicle 내부 해석 후 API-206 |

`drive_cmd_t`와 W2 글자의 대응은 `DRIVE_STOP`=S, `DRIVE_FWD`=F, `DRIVE_BWD`=B, `DRIVE_LEFT`=L, `DRIVE_RIGHT`=R입니다. `scan_point_t`의 `obstacle`은 메시지에서 1 또는 0, 측정 불가 거리는 -1로 씁니다.

## 8. 변경 이력

| 버전 | 날짜 | 변경한 ID와 내용 | 작성자 |
| --- | --- | --- | --- |
| v0.1 | 2026-10-27 | 담당 범위와 API 경계, API 명세 규칙 최초 작성 | \[작성\] |
| v0.2 | 2026-10-27 | 모듈 이름 확정(client, route, route\_bt, vehicle, vehicle\_bt, radar, 무조건 소문자). 블루투스를 route\_bt·vehicle\_bt 두 모듈로 분리. 1절 표, 2.1\~2.3·2.6\~2.10 반영 | \[작성\] |
| v0.3 | 2026-10-27 | 열거형 값·매크로만 대문자로 확정(2.3). 디버그 전용 UART(2.11), 두절 정지·P 되돌림·링크 상태 책임(2.12), 브랜치·.ioc 소유·병합 방식(2.13) 추가 | \[작성\] |
| v0.4 | 2026-10-27 | TYPE-001\~004(3절), API-001\~004(4절), API-101\~111(5절), API-201\~208(6절), MSG-101\~301(7절) 추가 | \[작성\] |
