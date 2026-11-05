# 손가락으로 LED 밝기 조절하기

카메라에 비친 **엄지 끝과 검지 끝 사이의 간격**으로 슬라이더를 움직이고, 그 슬라이더 값에 따라 **아두이노에 연결된 LED의 밝기**가 바뀌는 프로젝트입니다.

> 제공받은 HandPose 슬라이더 스케치에 **Web Serial 통신 코드를 추가**하고, 아두이노 수신 코드를 작성했습니다.


---

## 동작 흐름

```
엄지-검지 간격 → 슬라이더(0~100) → 밝기(0~255) 변환 → Web Serial 전송 → 아두이노 → analogWrite → LED
```

| 단계 | 담당 | 내용 |
|---|---|---|
| 손 인식 | p5.js + ml5 HandPose | 엄지(4번)·검지(8번) 끝 거리를 손바닥 길이로 나눠 0~100으로 정규화 |
| 값 전송 | `SerialHelper` (Web Serial) | 0~255로 변환한 밝기를 한 줄(`\n` 종료)씩 전송 |
| LED 제어 | Arduino Uno | 한 줄 읽어서 `analogWrite()`로 PWM 출력 |

---


## 회로 (freizing)

<img width="1057" height="556" alt="image" src="https://github.com/user-attachments/assets/ebd9e68f-c6ab-4bac-bb9c-4d02d817dac1" />
LED->저항->10번 핀에 연결


---

## 아두이노 IDE 코드

```cpp
const int LED_PIN = 10;

void setup() {
  Serial.begin(9600);        
  Serial.setTimeout(20);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  if (Serial.available()) {
    String line = Serial.readStringUntil('\n');
    line.trim();
    if (line.length() > 0) {
      int brightness = constrain(line.toInt(), 0, 255);
      analogWrite(LED_PIN, brightness);
    }
  }
}
```

---

## p5.js 변경 사항

### 1. 파일 추가: `serialhelper.js`
Web Serial 연결·송수신을 담당하는 `Serialhelper` 를 별도 파일로 추가했습니다.

### 2. `index.html` 수정
`sketch.js`보다 먼저 불러오도록 한 줄을 추가했습니다.

```diff
     <main>
     </main>
    <script src="serialhelper.js"></script>
     <script src="sketch.js"></script>
```

### 3. `sketch.js` 수정 (원본 코드는 그대로, 아래만 추가)

**① 전역 변수**
```diff
 let connections;

let serial, connectButton;
let lastSent = -1;
let lastSendTime = 0;
```

**② `setup()` — 연결 버튼 생성**
```diff
 async function setup() {
   createCanvas(640, 480);

  serial = new SerialHelper(9600, () => {}, msg => console.log(msg));
  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(async () => {
    await serial.toggle();
    lastSent = -1;                  
    lastSendTime = millis() + 2000;
  });
```

**③ `draw()` — 매 프레임 값 전송 시도**
```diff
     slider.value(c + 0.05*(v-c));
   }

  sendBrightness(slider.value());
 }
```

**④ 함수 추가 — 슬라이더 값을 밝기로 변환해 전송**
```js
async function sendBrightness(v) {
  if (!serial.writer) return;                
  const b = round(map(v, 0, 100, 0, 255));  
  if (b === lastSent) return;               
  if (millis() - lastSendTime < 50) return; 
  lastSendTime = millis();
  if (await serial.send(String(b))) lastSent = b;
}
```

---


## 구현하며 겪은 문제와 해결

| 문제 | 원인 | 해결 |
|---|---|---|
| 업로드 시 `COM5 액세스 거부` | 웹페이지가 시리얼 포트를 점유 중 | 업로드 전에 웹페이지에서 연결 해제(또는 탭 닫기) |
| LED가 켜졌다 꺼지기만 함 | PWM이 안 되는 핀(7번)에 연결 | `~` 표시가 있는 10번 핀으로 이동 |
| 연결 직후 LED가 반응 없음 | 포트를 열면 Uno가 리셋되어 첫 값이 유실됨 | 연결 후 2초 대기 뒤 현재 값을 다시 전송 |
| `sliderValue`가 갱신되지 않음 | 해당 변수는 마우스로 슬라이더를 직접 조작할 때만 갱신됨 | `slider.value()`를 직접 읽어서 사용 |

---

## 구현 영상



https://github.com/user-attachments/assets/d80a2384-b945-4fd7-9c90-f9c293096e1a



