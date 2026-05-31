# RISC-V RV32I Multi Cycle CPU + APB Bus 설계

> RV32I 기반 Multi Cycle CPU와 AMBA APB 프로토콜 기반 주변장치 설계 (SystemVerilog)

---

## 👥 팀 구성

| 이름 | 역할 |
|------|------|
| 송주연 | 프로젝트 개요, Multi Cycle CPU |
| 윤지원 | APB Master |
| 김민기 | RAM, GPIO |
| 조준호 | FND, UART |

---

## 📌 프로젝트 개요

- RV32I 명령어 셋 기반 RISC-V **Multi Cycle CPU** 설계
- AMBA **APB 프로토콜** 기반 주변장치(BRAM, GPIO, FND, UART) 설계
- C Firmware를 통한 **Memory-mapped IO** 제어

---

## 🛠️ 개발 환경

- Language: SystemVerilog, C
- Tool: Vivado (Simulation)
- Protocol: AMBA APB (Advanced Peripheral Bus)

---

## 🧱 전체 Block Diagram

<img width="2450" height="1485" alt="image" src="https://github.com/user-attachments/assets/d8bbe2fe-b980-49e5-9e15-c570fa04def9" />


| 모듈 | 설명 |
|------|------|
| Instruction Memory (ROM) | 실행할 명령어 저장 |
| RV32I CPU | RV32I 기반 명령어 해독 및 연산 (Multi Cycle) |
| APB Master | CPU 요청을 APB 프로토콜로 변환하여 Slave에 전달, Address Decoder로 Slave 선택 |
| APB Slave | APB 프로토콜로 CPU와 통신하는 주변장치 (BRAM, GPO, GPI, GPIO, FND, UART) |

---

## ⚙️ Multi Cycle CPU

### Single Cycle vs Multi Cycle

| | Single Cycle | Multi Cycle |
|--|-------------|-------------|
| 처리 방식 | 5단계를 1 사이클에 모두 처리 | 5단계를 1 사이클에 1단계씩 처리 |
| 클럭 속도 | 가장 긴 명령어 처리 속도에 맞춤 | 가장 오래 걸리는 단계에 맞춤 |
| 클럭 효율 | 짧은 명령어도 가장 긴 명령어 속도에 맞추므로 낭비 | 명령어 별로 필요한 단계만 수행하므로 효율적(Load 명령 제외) |
| 하드웨어 구조 | 단순 | 중간 결과 저장 레지스터 + FSM 필요 |

### 명령어 처리 5단계

```
Instruction FETCH → Instruction DECODE → EXECUTE → MEMORY → WRITE BACK
```

| 단계 | 동작 |
|------|------|
| Instruction FETCH | pc_en 발생, ROM에서 명령어 가져오기 |
| Instruction DECODE | 명령어 해독, IMM Extend |
| EXECUTE | ALU 연산 및 비교, 다음 PC 값 계산, R/I/U/J/JL Type Write Back |
| MEMORY | S-Type: RAM Write / IL-Type: RAM Read |
| WRITE BACK | IL-Type Write Back |

**명령어 Type별 동작 흐름도**

<img width="890" height="534" alt="image" src="https://github.com/user-attachments/assets/0e1c4a22-0eda-452e-b69c-0a86ac76b5fc" />


### 중간 결과 저장 레지스터

이전 단계의 데이터를 다음 클럭에서 사용하기 위해 각 단계 결과값을 저장하는 레지스터가 필요합니다.

<img width="2085" height="1317" alt="image" src="https://github.com/user-attachments/assets/7cb9fde9-94ab-4d6c-a795-40f1c69e9901" />

### RISC-V Multi Cycle Simulation

- **S-Type Instruction** : RAM에 데이터 저장하는 동작 검증
- **Load Instruction** : RAM에서 데이터 읽어오는 동작 검증
- **JAL Instruction** : PC(Program Counter) 분기 동작 검증

---

## 🚌 APB Bus

### APB(Advanced Peripheral Bus)란?

ARM AMBA 버스 중 하나로, 저속 주변장치 제어에 특화된 버스입니다.

<img width="605" height="447" alt="image" src="https://github.com/user-attachments/assets/4d055a48-281c-45fb-94c4-55feedb96759" />


### APB Master

CPU와 APB Slave 사이의 브릿지 역할을 합니다.

**핵심 신호**

| 방향 | 신호 | 비트 | 설명 |
|------|------|------|------|
| CPU → Master | `addr` | 32 | CPU가 접근할 메모리 주소 |
| CPU → Master | `wdata` | 32 | CPU가 쓸 데이터 |
| CPU → Master | `w_req` | 1 | 쓰기 요청 |
| CPU → Master | `r_req` | 1 | 읽기 요청 |
| Master → Slave | `p_addr` | 32 | APB 주소 버스 |
| Master → Slave | `p_wdata` | 32 | APB 쓰기 데이터 |
| Master → Slave | `p_write` | 1 | 1=Write, 0=Read |
| Master → Slave | `p_en` | 1 | PENABLE, ACCESS 단계에서 1 |
| Master → Slave | `p_sel_0~5` | 1 | Slave 선택 신호 |
| Slave → CPU | `p_rdata_0~5` | 32 | 각 Slave 읽기 데이터 |
| Slave → CPU | `p_ready_0~5` | 1 | 각 Slave 트랜잭션 완료 신호 |

**APB Master FSM**

| 신호 | IDLE | SETUP | ACCESS |
|------|:----:|:-----:|:------:|
| p_en | 0 | 0 | 1 |
| p_sel | 0 | 1 | 1 |
| decode_en | 0 | 1 | 1 |

**Address Decoder**

`PADDR` 기반으로 6개 Slave 중 `p_sel` 결정

```
addr[31:28] = 4'h1  →  BRAM 선택 (p_sel_0)
addr[31:28] = 4'h2  →  addr[14:12]로 세부 선택
    addr[14:12] = 0  →  GPO  (p_sel_1)
    addr[14:12] = 1  →  GPI  (p_sel_2)
    addr[14:12] = 2  →  GPIO (p_sel_3)
    addr[14:12] = 3  →  FND  (p_sel_4)
    addr[14:12] = 4  →  UART (p_sel_5)
```

### Memory Mapped IO

CPU가 메모리 주소로 주변장치 레지스터를 직접 읽고 쓰는 방식

**Slave Mapping**

<img width="270" height="550" alt="image" src="https://github.com/user-attachments/assets/e495b3de-5df0-4868-b121-84c52b4d1c74" />



**Register Mapping**
<img width="1338" height="571" alt="image" src="https://github.com/user-attachments/assets/fd4020fd-5ba5-417e-a2a4-35957457ef3e" />

---

## 🔌 APB Slave 상세

### BRAM (Block RAM)

CPU 스택 및 데이터 저장 공간

| 항목 | 내용 |
|------|------|
| 주소 | 0x1000_0000 |
| 크기 | 4KB (1024 words) |
| 주소 인덱싱 | `paddr[11:2]` (하위 2비트 제거, 4바이트 워드 단위) |
| 쓰기 | `psel & penable & pwrite = 1` → 클럭 상승 엣지에서 저장 |
| 읽기 | `pwrite = 0` → 조합 논리로 즉시 반환 |
| pready | `psel & penable = 1`이면 즉시 1 (Wait State 없음) |

---

### GPIO (General Purpose Input/Output)

CPU와 물리적 장치(LED/Switch) 간 양방향 제어

**Block Diagram**

<img width="2646" height="1498" alt="image" src="https://github.com/user-attachments/assets/589c40aa-32ac-403a-861f-e55bf8490fd5" />


| 레지스터 | 주소 오프셋 | 설명 |
|---------|-----------|------|
| CTRL_REG | 0x2000_2000 | 16비트 핀 입출력 방향 설정 |
| ODATA_REG | 0x2000_2004 | 출력 데이터 (LED 제어) |
| IDATA_REG | 0x2000_2008 | 입력 캡처 (Switch 상태) |

**Test 시나리오**

1. 초기화: `CTRL_REG`, `ODATA_REG` → `0x0000_0000`
2. 방향 설정: `CTRL_REG` ← `0x0000_FF00` (상위 8비트 출력/하위 8비트 입력)
3. 입력 읽기: 외부 `gpio = 8'h34` → `IDATA_REG = 0x0000_0034`
4. 출력 쓰기: `sw_val << 8` 연산 결과 → `ODATA_REG = 0x0000_3400`

---

### FND (7-Segment Display)

시스템 내부 연산 결과를 10진수(0~9999)로 표출

| 레지스터 | 주소 오프셋 | 설명 |
|---------|-----------|------|
| CTRL_REG | 0x2000_3000 | FND 제어 레지스터 |
| ODATA_REG | 0x2000_3004 | 출력 데이터 (표시할 값) |

**동작 원리**
- GPI(상위 8비트) + GPIO(하위 8비트) 입력을 16비트로 병합
- 내부 타이머 클럭에 맞춰 `fnd_digit` 순차 로테이션 (Dynamic Scanning)
  ```
  1110 → 1101 → 1011 → 0111
  ```
- 각 자리에 맞는 세그먼트 패턴 출력

---

### UART

PC와 직렬 통신 수행 (9600/115200 bps)

**레지스터 맵**

| 오프셋 | 레지스터 | 설명 |
|--------|---------|------|
| 0x00 | CTRL_REG | Bit[0]: TX_START, Bit[1]: UART Enable |
| 0x04 | BAUD_REG | Baud Rate 분주비 설정 (115200bps = 0x03) |
| 0x08 | STATUS_REG | Bit[0]: tx_busy (Read Only), Bit[31]: rx_done (Read Only) |
| 0x0C | TXDATA_REG | 송신할 8비트 데이터 |
| 0x10 | RXDATA_REG | 수신된 8비트 데이터 |

**C Firmware 동작 흐름**
```c
// 1. 초기화
sys_init();  // 모든 레지스터 0으로 초기화

// 2. Baud Rate 설정
BAUD_REG = 0x03;  // 115200 bps

// 3. 수신 대기 (폴링)
while (!(STATUS_REG & (1 << 31)));  // RX_DONE 확인

// 4. 데이터 수신
data = RXDATA_REG;

// 5. 송신 대기 및 전송
while (STATUS_REG & 0x1);  // TX_BUSY 확인
TXDATA_REG = data;
CTRL_REG |= 0x1;  // TX_START
```

**Test 시나리오 (Echo-back)**
- PC → UART RX: ASCII 'A'(0x41), 'B'(0x42) 순차 전송
- CPU 폴링으로 수신 확인 후 TXDATA에 기입
- UART TX → PC: 수신 데이터 Echo-back
- APB를 통해 FND에도 동일 데이터 표시

---

## 📊 APB 시뮬레이션 시나리오

| Slave | 동작 | 주소 | 입력값 | 기대 출력 |
|-------|------|------|--------|---------|
| BRAM | Write | 0x1000_0004 | 0xABCD_1234 | pready0=1 |
| BRAM | Read | 0x1000_0000 | - | prdata0=0x0000_0001 |
| GPO | Write | 0x2000_0000 | 0x0000_00FF | pready1=1 |
| GPI | Read | 0x2000_1004 | gpi=0xAA | prdata2=0x0000_00AA |
| GPIO | Write | 0x2000_2000 | 0x0000_FF00 | pready3=1 |
| FND | Write | 0x2000_3004 | SW값 | pready4=1 |
| UART | Read | 0x2000_4010 | uart_rx='Z' | prdata5=0x0000_005A |

---

## 🐛 Trouble Shooting

### Multi Cycle FSM이 MEMORY 단계에서 멈추는 문제

**문제**
MEMORY 상태에서 FSM이 다음 단계로 진행되지 않음.
`dwe`, `dre`, `ready` 신호가 모두 0 → CPU가 무한 대기 상태에 빠짐

**원인**
APB Master의 `ready` 신호가 CPU로 피드백되지 않아
Multi Cycle FSM이 `MEMORY → WB` 전이 조건을 충족하지 못함

**해결**
APB Master MUX의 `ready` 출력을 CPU `ready` 입력에 연결
`ready = 1` 응답 확인 후 정상적으로 WB 단계로 전이
