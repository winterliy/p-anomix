# Minecraft MIDI Piano Project

## 1. 프로젝트 개요

### 1.1 목표

USB MIDI 키보드를 이용하여 마인크래프트 서버 안의 3D 피아노를 실시간으로 연주하는 시스템을 개발한다.

시스템은 다음 요소로 구성한다.

- Fabric 클라이언트 모드
- Paper 서버 플러그인
- 서버 리소스팩
- Blockbench 기반 3D 피아노 모델
- 무료 피아노 샘플 기반 커스텀 사운드

### 1.2 핵심 기능

1. USB MIDI 키보드 입력 감지
2. MIDI Note On / Note Off 처리
3. MIDI Velocity에 따른 연주 강약 표현
4. Sustain Pedal 지원
5. 3D 피아노 건반 애니메이션
6. 커스텀 피아노 음원 재생
7. 주변 플레이어에게 실시간 연주 전달
8. 여러 건반을 동시에 누르는 화음 연주
9. 연주자만 모드를 설치하는 선택적 클라이언트 구조

---

## 2. 전체 시스템 아키텍처

### 2.1 최종 채택 구조

```text
USB MIDI Keyboard
        │
        ▼
Minecraft Client
(Fabric MIDI Mod)
        │
        │ Custom Payload
        │ Minecraft Network
        ▼
Paper Server
(MIDI Piano Plugin)
        │
        ├── MIDI Event Processing
        │
        ├── Piano Session Management
        │
        ├── Sound Engine
        │       │
        │       ▼
        │   Resource Pack
        │   Custom OGG Sounds
        │
        └── 3D Animation Engine
                │
                ▼
            ItemDisplay
            Piano Models
        │
        ▼
All Nearby Players
(Vanilla Clients Supported)
```

### 2.2 클라이언트 설치 조건

| 구성요소 | 연주자 | 관객 |
|---|---|---|
| USB MIDI 키보드 | 필요 | 불필요 |
| Fabric MIDI 모드 | 필요 | 불필요 |
| Paper 플러그인 | 서버에 설치 | 서버에 설치 |
| 서버 리소스팩 | 적용 필요 | 적용 필요 |
| 외부 MIDI 프로그램 | 불필요 | 불필요 |
| 별도 네트워크 포트 | 불필요 | 불필요 |

Paper 플러그인은 Fabric 모드가 없는 일반 플레이어의 접속도 허용하도록 설계한다.

서버 리소스팩은 접속 시 자동으로 다운로드하도록 안내하거나 요청할 수 있다. 다만 클라이언트에서 리소스팩 적용을 거부하면 커스텀 모델과 사운드가 정상적으로 표시되지 않을 수 있다.

---

## 3. Fabric 클라이언트 모드

### 3.1 역할

Fabric 모드는 플레이어 컴퓨터에 연결된 USB MIDI 장치를 감지하고 MIDI 이벤트를 Minecraft 네트워크를 통해 Paper 서버에 전달한다.

별도의 외부 프로그램을 실행할 필요가 없다.

### 3.2 MIDI 입력

Java의 MIDI API를 활용한다.

```java
javax.sound.midi
```

주요 구성 요소:

- MidiSystem
- MidiDevice
- Transmitter
- Receiver
- ShortMessage

### 3.3 처리할 MIDI 이벤트

| MIDI 이벤트 | 기능 |
|---|---|
| Note On | 건반 누름 |
| Note Off | 건반 해제 |
| Velocity | 연주 강약 |
| Control Change 64 | 서스테인 페달 |
| Program Change | 악기 변경 |
| Pitch Bend | 음높이 변화 |

초기 개발에서는 다음 기능을 우선 구현한다.

1. Note On
2. Note Off
3. Velocity
4. Sustain Pedal

### 3.4 MIDI 음높이

일반적인 MIDI 음번호는 0부터 127까지이다.

88건반 피아노의 음역은 다음과 같다.

| 음 | MIDI 번호 |
|---|---|
| A0 | 21 |
| C4 | 60 |
| A4 | 69 |
| C8 | 108 |

C4는 일반적으로 가온다에 해당한다.

### 3.5 서버 전송

Fabric 모드는 Minecraft의 Custom Payload 기능을 이용하여 Paper 서버에 데이터를 전송한다.

예시 패킷 구조:

```json
{
  "type": "note_on",
  "note": 60,
  "velocity": 100,
  "channel": 0
}
```

건반을 놓는 경우:

```json
{
  "type": "note_off",
  "note": 60,
  "channel": 0
}
```

위 JSON은 통신 데이터의 개념적 예시이다.

실제 구현에서는 Fabric 및 Paper 버전에 맞는 바이너리 Payload 형식을 사용할 수 있다.

### 3.6 클라이언트 모드의 추가 기능

향후 다음 기능을 구현할 수 있다.

- MIDI 장치 선택
- 입력 채널 선택
- 음역 변경
- 옥타브 이동
- Velocity 보정
- 페달 설정
- 연주 모드 활성화
- 서버 연결 상태 표시

---

## 4. Paper 서버 플러그인

### 4.1 역할

Paper 플러그인은 Fabric 모드에서 전달한 MIDI 이벤트를 수신하여 다음 작업을 수행한다.

1. 플레이어 인증
2. 연주 세션 확인
3. MIDI 음높이 해석
4. 커스텀 사운드 재생
5. 3D 건반 애니메이션
6. 주변 플레이어에게 연주 전달

### 4.2 네트워크 통신

Fabric 모드와 Paper 플러그인은 동일한 Custom Payload 프로토콜을 사용해야 한다.

예시 채널:

```text
piano:note_on
piano:note_off
piano:pedal
piano:control
```

또는 단일 채널에서 이벤트 종류를 구분하는 구조도 가능하다.

```text
piano:midi
```

최종 채널 구조는 개발 과정에서 결정한다.

### 4.3 연주 세션

플러그인은 플레이어별 연주 상태를 관리한다.

예시:

```text
PianoSession
├── Player UUID
├── Piano ID
├── Instrument
├── Active Notes
├── Sustain State
└── Session Status
```

플레이어가 피아노에 앉거나 연주를 시작하면 세션을 생성한다.

연주가 종료되거나 플레이어가 접속을 종료하면 세션을 정리한다.

### 4.4 MIDI 처리

Note On 수신 시:

1. 패킷 검증
2. 연주 권한 확인
3. 음높이 확인
4. Velocity 처리
5. 사운드 재생
6. 건반 애니메이션 실행

Note Off 수신 시:

1. 해당 건반의 활성 상태 확인
2. 건반 복귀
3. 음원 감쇠 처리
4. Sustain 상태 반영

### 4.5 동시 연주

여러 음을 동시에 연주할 수 있도록 음별 상태를 독립적으로 관리한다.

예시:

```text
Active Notes:
C4 = Pressed
E4 = Pressed
G4 = Pressed
```

이 경우 C Major 화음이 재생된다.

동일 음의 반복 입력이나 여러 MIDI 채널을 처리할 수 있도록 확장 가능한 구조가 필요하다.

### 4.6 스레드 처리

네트워크 입력이나 MIDI 이벤트가 비동기적으로 처리될 수 있으므로 서버 월드 및 엔티티 조작은 Paper에서 허용하는 적절한 서버 스레드 또는 스케줄러에서 실행한다.

---

## 5. 3D 피아노 모델

### 5.1 제작 도구

Blockbench를 사용하여 3D 피아노 모델을 제작한다.

### 5.2 모델 구성

피아노는 다음 요소로 구분할 수 있다.

```text
Piano
├── Body
├── Keyboard Frame
├── White Keys
├── Black Keys
├── Pedals
└── Decorative Parts
```

### 5.3 건반 모델

88건반을 각각 독립적으로 제어할 수 있도록 구성한다.

```text
Piano Keyboard
├── Key 21 (A0)
├── Key 22 (A#0)
├── Key 23 (B0)
├── ...
└── Key 108 (C8)
```

### 5.4 ItemDisplay

Paper의 ItemDisplay 엔티티를 이용하여 커스텀 모델을 표시할 수 있다.

건반마다 별도의 Display 엔티티를 사용하면 개별 위치 및 회전을 제어할 수 있다.

### 5.5 건반 애니메이션

Note On:

```text
Normal Position
      ↓
Pressed Position
```

Note Off:

```text
Pressed Position
      ↓
Normal Position
```

애니메이션은 Display 엔티티의 Transformation 기능을 활용한다.

### 5.6 성능 고려

88개의 건반을 각각 엔티티로 생성하면 피아노 수에 비례하여 엔티티가 증가한다.

최적화 방안:

- 일정 거리 밖의 피아노 표시 제한
- 불필요한 애니메이션 최소화
- 건반 상태 변경 시에만 업데이트
- 정적 본체와 움직이는 건반 분리
- 여러 피아노의 동시 연주 성능 측정

---

## 6. 피아노 사운드 시스템

### 6.1 기본 방향

바닐라 노트블록 대신 실제 피아노 샘플을 사용한다.

음원은 OGG 형식으로 변환하여 서버 리소스팩에 포함한다.

### 6.2 구현 목표

- 88건반 음역
- 실제 피아노 음색
- Velocity 지원
- 화음 지원
- Sustain Pedal 지원
- 건반 해제에 따른 감쇠
- 3차원 위치 기반 사운드

### 6.3 사운드 재생

Paper 플러그인은 리소스팩에 등록된 커스텀 사운드 이벤트를 호출한다.

예시:

```text
piano:grand.c4_soft
piano:grand.c4_medium
piano:grand.c4_loud
```

Velocity에 따라 다른 샘플을 선택할 수 있다.

예시:

| Velocity | 샘플 |
|---|---|
| 1~42 | Soft |
| 43~84 | Medium |
| 85~127 | Loud |

실제 구간은 음원 특성과 연주감을 고려하여 조정한다.

### 6.4 Note Off의 한계

Minecraft의 기본 사운드 API는 전문 MIDI 신시사이저처럼 개별 음원의 엔벌로프를 정밀하게 제어하는 데 한계가 있다.

따라서 다음 기능은 별도의 설계가 필요하다.

- 건반 해제 후 감쇠
- Sustain Pedal
- Release Sample
- 음량 변화
- 음의 지속 시간

필요하다면 짧은 샘플, 릴리스 샘플, 사운드 중지 및 재생 전략을 조합한다.

단순히 Note Off를 받는 것만으로 자연스러운 피아노 감쇠가 자동 구현되지는 않는다.

---

## 7. 무료 피아노 샘플 라이브러리

### 7.1 Salamander Grand Piano

**우선 추천 음원**

특징:

- Yamaha C5 그랜드 피아노 기반
- 실제 피아노 녹음 샘플
- 다양한 Velocity 레이어
- 릴리스 및 공명 관련 샘플
- SFZ 기반 무료 샘플 라이브러리

라이선스:

CC BY 3.0

저작자 표시 등 라이선스 조건을 준수하면 리소스팩에 활용할 수 있다.

링크:

https://github.com/sfzinstruments/SalamanderGrandPiano

https://freepats.zenvoid.org/Piano/acoustic-grand-piano.html

권장 활용:

- 필요한 음역의 샘플 추출
- 3단계 정도의 Velocity 레이어 선택
- OGG 변환
- sounds.json 등록
- Paper 플러그인과 연결

### 7.2 Splendid Grand Piano

특징:

- Steinway 피아노 기반 샘플
- 비교적 작은 용량
- 무료 피아노 사운드 라이브러리

피아노 샘플 목록:

https://sfzinstruments.github.io/pianos/

배포 전 실제 다운로드 파일의 라이선스 조건을 확인해야 한다.

### 7.3 FreePats Upright Piano

특징:

- 업라이트 피아노 음색
- 무료 샘플
- CC0로 제공되는 샘플 선택 가능

링크:

https://freepats.zenvoid.org/

리소스팩에 포함할 때는 선택한 샘플의 라이선스를 개별적으로 확인한다.

---

## 8. 기존 Minecraft 피아노 리소스팩

### 8.1 Extended Minecraft Piano

GitHub:

https://github.com/KerbalBoy/extended-minecraft-piano

특징:

- Minecraft 피아노 음역 확장
- 노트블록 사운드 변경
- 다양한 음 길이 지원

주의사항:

이 리소스팩은 기본적으로 노트블록 사운드를 변경하는 방식이다.

따라서 독립적인 MIDI 피아노 플러그인에서는 기존 사운드를 그대로 대체하기보다 커스텀 사운드 네임스페이스를 사용하는 것이 좋다.

소스 코드 및 리소스 재사용 시 라이선스를 확인해야 한다.

---

## 9. 리소스팩 설계

### 9.1 기본 디렉터리

```text
PianoResourcePack/
├── pack.mcmeta
└── assets/
    └── piano/
        ├── sounds.json
        ├── sounds/
        │   └── grand/
        │       ├── c4_soft.ogg
        │       ├── c4_medium.ogg
        │       ├── c4_loud.ogg
        │       ├── cs4_medium.ogg
        │       └── ...
        ├── models/
        └── textures/
```

실제 모델 및 텍스처 디렉터리 구조는 대상 Minecraft 버전에 맞춰 조정한다.

### 9.2 sounds.json 예시

```json
{
  "grand.c4_soft": {
    "sounds": [
      "piano:grand/c4_soft"
    ]
  },
  "grand.c4_medium": {
    "sounds": [
      "piano:grand/c4_medium"
    ]
  },
  "grand.c4_loud": {
    "sounds": [
      "piano:grand/c4_loud"
    ]
  }
}
```

### 9.3 샘플 수 계산

88건반을 모두 개별 샘플로 구성한다고 가정한다.

| Velocity 레이어 | 파일 수 |
|---|---|
| 1단계 | 88 |
| 3단계 | 264 |
| 5단계 | 440 |
| 16단계 | 1408 |

위 계산에는 릴리스, 페달 및 공명 샘플이 포함되지 않는다.

### 9.4 용량 최적화

모든 음높이를 개별 샘플로 저장할 필요는 없다.

일부 샘플을 피치 변환하여 여러 음에 사용할 수 있다.

다만 피치 변환 범위가 커질수록 실제 피아노 음색과 차이가 발생할 수 있다.

권장 초기 설정:

- 88건반 지원
- 3단계 Velocity
- 필요한 음높이만 샘플링
- OGG 압축
- 음질 및 용량 테스트

---

## 10. 기존 유사 프로젝트

### 10.1 JW Piano

Spigot:

https://www.spigotmc.org/resources/piano.103490/

GitHub:

https://github.com/jwdeveloper/JW_Piano

특징:

- Minecraft 3D 피아노
- 리소스팩 사용
- 외부 데스크톱 프로그램을 통한 연주
- 공개 소스 코드

참고할 부분:

- 3D 피아노 모델 구조
- 서버 플러그인 구조
- 실시간 연주 처리
- 사운드 및 애니메이션 연동

구버전 중심 프로젝트이므로 최신 Paper 호환성은 별도 검증이 필요하다.

### 10.2 BukkitPiano

Spigot:

https://www.spigotmc.org/resources/bukkitpiano.16931/

GitHub:

https://github.com/glatteis/BukkitPiano

특징:

- MIDI 컨트롤러 입력
- 별도 데스크톱 프로그램
- Bukkit/Spigot 기반

실험적인 구형 프로젝트이므로 직접 사용보다는 구조 참고에 적합하다.

### 10.3 Live Musical Instruments

Modrinth:

https://modrinth.com/mod/live-musical-instruments

특징:

- 외부 전자악기 입력
- 별도 프로그램 활용
- Minecraft 서버로 실시간 전달

확인된 설명상 전자드럼 중심 프로젝트이다.

### 10.4 MidiReceiver

Modrinth:

https://modrinth.com/plugin/midireceiver

특징:

- Paper 기반
- MIDI 메시지 수신
- 클라이언트 모드와 연동
- 노트블록 기반 사운드

이번 프로젝트의 Fabric 모드와 Paper 간 통신 구조를 참고할 만하다.

### 10.5 LiveMIDI

Modrinth:

https://modrinth.com/project/livemidi

특징:

- Fabric 기반
- USB MIDI 장치 입력
- Minecraft 내부 실시간 연주

클라이언트의 MIDI 장치 인식과 입력 처리 구조를 참고할 수 있다.

---

## 11. 구현 방식 비교

| 방식 | 장점 | 단점 |
|---|---|---|
| Paper만 사용 | 서버 관리 간단 | USB MIDI 입력 직접 수신 불가 |
| 외부 프로그램 + Paper | 바닐라 클라이언트 유지 | 별도 프로그램 필요 |
| Web MIDI + Paper | 별도 실행 프로그램 불필요 | 브라우저 실행 필요 |
| Fabric 모드 + Paper | Minecraft 네트워크 직접 활용 | 연주자 모드 설치 필요 |

### 최종 채택

**Fabric 모드 + Paper 플러그인 + 서버 리소스팩**

선정 이유:

1. 별도 외부 프로그램 불필요
2. 별도 MIDI 통신 서버 포트 불필요
3. Minecraft의 기존 서버 연결 활용
4. MIDI 이벤트를 직접 전송 가능
5. 관객은 바닐라 클라이언트 사용 가능
6. 3D 모델과 사운드를 서버에서 통합 관리 가능

---

## 12. 지연 및 성능

### 12.1 네트워크 지연

MIDI 입력은 클라이언트에서 서버로 전송된다.

서버는 이를 처리하여 주변 플레이어에게 사운드와 애니메이션을 전달한다.

지연 요인:

- MIDI 입력 처리
- 네트워크 지연
- 서버 틱
- 사운드 패킷 전달
- 클라이언트 사운드 재생

### 12.2 서버 틱

일반적인 Minecraft 서버는 초당 20틱을 기준으로 동작한다.

따라서 서버에서 처리하는 애니메이션과 이벤트에는 틱 단위의 시간 제약이 발생할 수 있다.

### 12.3 연주자 로컬 사운드

지연을 줄이기 위한 선택적 개선안:

- Fabric 모드에서 로컬 사운드 즉시 재생
- 서버는 주변 관객용 사운드 재생
- 연주자에게는 서버 사운드 중복 재생 방지

이 방식은 연주자의 입력 반응성을 높이는 데 유리하다.

다만 로컬 재생과 서버 재생의 음색 및 타이밍을 일치시키는 추가 설계가 필요하다.

---

## 13. 보안 및 안정성

### 13.1 패킷 검증

Paper 플러그인은 다음 항목을 확인해야 한다.

- 실제 접속 플레이어 여부
- 연주 세션 존재 여부
- MIDI 음번호 범위
- Velocity 범위
- 이벤트 전송 빈도
- 해당 피아노의 사용 권한

### 13.2 악의적인 입력 방지

클라이언트 모드는 신뢰할 수 없는 입력원으로 취급한다.

서버에서 패킷을 검증하고 과도한 입력을 제한한다.

### 13.3 접속 종료 처리

연주자가 접속을 종료하거나 연주 세션을 중단하면 다음 작업을 수행한다.

- 활성 건반 초기화
- Sustain 상태 해제
- 연주 세션 종료
- 필요한 사운드 정리
- 건반 모델 원위치 복귀

---

## 14. 권장 개발 환경

| 구성 | 기술 |
|---|---|
| 개발 언어 | Java |
| 클라이언트 모드 | Fabric |
| 서버 플러그인 | Paper API |
| MIDI 입력 | Java Sound MIDI API |
| 통신 | Minecraft Custom Payload |
| 3D 모델 | Blockbench |
| 3D 표시 | ItemDisplay |
| 사운드 | OGG + sounds.json |
| 빌드 | Gradle |
| 버전 관리 | Git / GitHub |

Minecraft, Fabric, Paper의 정확한 버전은 개발 시작 시 결정하고, 해당 버전의 API 및 Java 요구사항에 맞춰 구성한다.

---

## 15. 권장 프로젝트 구조

```text
MinecraftMIDIPiano/
│
├── midi-common/
│   ├── MidiEvent
│   ├── MidiNote
│   ├── MidiProtocol
│   └── PacketCodec
│
├── midi-fabric/
│   ├── MidiDeviceManager
│   ├── MidiInputReceiver
│   ├── MidiPacketSender
│   └── ClientConfig
│
├── midi-paper/
│   ├── PianoPlugin
│   ├── NetworkHandler
│   ├── PianoSessionManager
│   ├── MidiEventHandler
│   ├── PianoSoundEngine
│   ├── PianoAnimationEngine
│   └── PianoModelManager
│
└── resource-pack/
    ├── pack.mcmeta
    └── assets/
        └── piano/
```

`midi-common`은 클라이언트와 서버에서 공유할 MIDI 데이터 구조 및 통신 규격을 담당한다.

실제 프로젝트에서는 Fabric 및 Paper API에 대한 직접 의존성을 공통 모듈에서 분리하는 것이 좋다.

---

## 16. 개발 단계

### Phase 1 — MIDI 입력

목표:

USB MIDI 키보드를 Fabric 모드에서 인식한다.

구현 항목:

- MIDI 장치 검색
- MIDI 장치 연결
- Note On 감지
- Note Off 감지
- Velocity 읽기
- 로그 출력

완료 기준:

USB MIDI 키보드를 누르면 Minecraft 클라이언트 로그에 정확한 MIDI 이벤트가 표시된다.

### Phase 2 — 클라이언트·서버 통신

목표:

Fabric 모드에서 Paper 플러그인으로 MIDI 이벤트를 전송한다.

구현 항목:

- Custom Payload 정의
- 패킷 직렬화
- 패킷 수신
- 플레이어 식별
- MIDI 데이터 검증

완료 기준:

연주자의 MIDI 입력이 Paper 서버 로그에 정상적으로 표시된다.

### Phase 3 — 사운드 재생

목표:

MIDI 음높이에 대응하는 피아노 소리를 재생한다.

구현 항목:

- 무료 피아노 샘플 확보
- OGG 변환
- sounds.json 제작
- 서버 리소스팩 배포
- Paper 사운드 재생

완료 기준:

MIDI 키보드로 연주하면 주변 플레이어가 피아노 소리를 들을 수 있다.

### Phase 4 — 3D 피아노

목표:

게임 안에 연주 가능한 3D 피아노를 구현한다.

구현 항목:

- Blockbench 모델 제작
- 피아노 본체 배치
- 건반 모델 배치
- ItemDisplay 관리
- 건반 누름 애니메이션
- 건반 복귀 애니메이션

완료 기준:

실제 MIDI 건반을 누르면 해당 3D 건반이 움직인다.

### Phase 5 — 고급 연주 기능

목표:

실제 피아노에 가까운 연주 경험을 구현한다.

구현 항목:

- Velocity 레이어
- Sustain Pedal
- Release Sample
- 화음 처리
- 연타 처리
- 동시 연주
- 지연 최적화

### Phase 6 — 확장 기능

추가 개발 후보:

- 여러 악기 지원
- MIDI 파일 녹음
- MIDI 파일 재생
- 자동 연주
- 연주자별 음량 설정
- 피아노 소유권
- 피아노 설치 및 제거 명령어
- GUI 기반 MIDI 장치 선택
- 피아노 연주 권한 관리
- 공연 시스템

---

## 17. 최종 권장 설계

### 핵심 구조

```text
USB MIDI Keyboard
        ↓
Fabric MIDI Client Mod
        ↓
Minecraft Custom Payload
        ↓
Paper MIDI Piano Plugin
        ↓
 ┌───────────────┐
 │               │
 ▼               ▼
Sound Engine   Model Engine
 │               │
 ▼               ▼
OGG Samples    3D Key Animation
 │               │
 └───────┬───────┘
         ↓
Nearby Minecraft Players
```

### 권장 사운드

Salamander Grand Piano V3

### 권장 사운드 구성

- 88건반 지원
- 초기 3단계 Velocity
- OGG 파일 사용
- 커스텀 사운드 네임스페이스
- 필요에 따라 릴리스 및 공명 샘플 확장

### 권장 구현 순서

1. MIDI 입력
2. Fabric–Paper 통신
3. 커스텀 사운드
4. 3D 피아노
5. Velocity 및 Sustain
6. 성능 최적화
7. 고급 기능 확장

---

## 18. 참고 링크

### Minecraft 관련

- PaperMC: https://papermc.io/
- Paper 문서: https://docs.papermc.io/
- Fabric: https://fabricmc.net/
- Fabric 문서: https://docs.fabricmc.net/
- Blockbench: https://www.blockbench.net/

### 피아노 샘플

- Salamander Grand Piano: https://github.com/sfzinstruments/SalamanderGrandPiano
- FreePats: https://freepats.zenvoid.org/
- SFZ Instruments: https://sfzinstruments.github.io/pianos/

### 기존 프로젝트

- JW Piano: https://github.com/jwdeveloper/JW_Piano
- BukkitPiano: https://github.com/glatteis/BukkitPiano
- Extended Minecraft Piano: https://github.com/KerbalBoy/extended-minecraft-piano
- MidiReceiver: https://modrinth.com/plugin/midireceiver
- LiveMIDI: https://modrinth.com/project/livemidi
- Live Musical Instruments: https://modrinth.com/mod/live-musical-instruments

---

## 19. 기술적 유의사항

본 문서는 프로젝트의 초기 설계안이다.

다음 사항은 실제 구현 및 테스트를 통해 검증해야 한다.

1. 최신 Minecraft 버전의 Fabric Custom Payload API
2. Paper의 플러그인 메시징 및 패킷 처리 호환성
3. 88개 ItemDisplay 엔티티의 성능
4. 커스텀 사운드의 Note Off 처리 방식
5. Sustain Pedal의 실제 구현 가능 범위
6. 사운드 재생 지연 및 화음 동기화
7. 리소스팩 용량과 클라이언트 메모리 사용량
8. 무료 샘플의 라이선스 및 재배포 조건
9. 기존 오픈소스 프로젝트의 라이선스
10. 대상 Minecraft 버전의 리소스팩 모델 형식

특히 Minecraft의 기본 사운드 시스템은 완전한 MIDI 신시사이저가 아니므로, 전문 피아노 수준의 감쇠·페달·공명 표현은 추가적인 사운드 엔진 설계가 필요하다.

---

## 20. 프로젝트 결론

본 프로젝트는 USB MIDI 키보드를 이용하여 Minecraft 서버 내부의 3D 피아노를 실시간으로 연주하는 시스템이다.

최종 구조는 다음과 같다.

**Fabric 클라이언트 모드 + Paper 서버 플러그인 + 서버 리소스팩**

Fabric 모드는 연주자의 MIDI 입력을 담당하고, Paper 플러그인은 MIDI 이벤트를 해석하여 서버의 사운드와 3D 모델을 제어한다.

관객은 별도의 모드를 설치하지 않고 바닐라 Minecraft 클라이언트로 접속하여 연주를 감상할 수 있다.

초기에는 기본적인 MIDI 입력, 피아노 사운드 및 건반 애니메이션을 구현하고, 이후 Velocity, Sustain Pedal, 고급 음원 처리 및 공연 시스템으로 확장한다.