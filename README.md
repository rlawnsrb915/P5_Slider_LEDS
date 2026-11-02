# P5_Slider_LEDS

<회로도>

<img width="1088" height="728" alt="image" src="https://github.com/user-attachments/assets/a34da060-88b8-4347-8d53-510a3f0825a6" />


<코드>
p5.js 코드:
```

let serial, connectButton;
let brightnessSlider;

let ledOn = false;
let potValue = 0;
let brightness = 0;
let lastBrightness = -1;

let statusText = 'Arduino를 연결해 주세요.';

function setup() {
  createCanvas(600, 440);
  textFont('sans-serif');

  // SerialHelper
  serial = new SerialHelper(
    9600,
    receiveLine,
    message => statusText = message
  );

  // Arduino 연결 버튼
  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(() => {
    lastBrightness = -1;
    serial.toggle();
  });

  // LED 밝기 조절 슬라이더
  brightnessSlider = createSlider(0, 255, 0, 1);
  brightnessSlider.position(310, 350);
  brightnessSlider.size(220);

  // 슬라이더를 움직이면 밝기 변경
  brightnessSlider.input(changeBrightness);
}

function draw() {
  background(242, 246, 250);

  fill(25, 40, 60);
  noStroke();
  textAlign(CENTER);

  textSize(24);
  text('Arduino Uno R3 · Web Serial', width / 2, 36);

  textSize(14);
  text(statusText, width / 2, 66);

  // ==================================
  // 13번 LED ON/OFF
  // ==================================
  fill(ledOn ? '#22c55e' : '#64748b');
  circle(160, 175, 110);

  fill(255);
  textSize(22);
  text(ledOn ? 'ON' : 'OFF', 160, 183);

  fill(25, 40, 60);
  textSize(15);
  text('원을 클릭하면 LED 토글', 160, 260);

  // ==================================
  // 가변저항 A0 표시
  // ==================================
  text('가변저항 A0', 420, 120);

  fill('#dbe3ed');
  rect(310, 155, 220, 40, 10);

  fill('#3b82f6');
  rect(
    310, 155,
    map(potValue, 0, 1023, 0, 220),
    40, 10
  );

  fill(25, 40, 60);
  textSize(24);
  text(potValue + ' / 1023', 420, 235);

  // ==================================
  // 3번 LED 밝기 슬라이더
  // ==================================
  brightness = brightnessSlider.value();

  fill(25, 40, 60);
  textSize(18);
  text('3번 LED 밝기 조절', 420, 290);

  textSize(16);
  text(
    '밝기: ' + brightness + ' / 255',
    420, 320
  );

  textSize(15);
  text(
    floor(brightness / 255 * 100) + '%',
    420, 390
  );

  textSize(12);
  text('9600 baud · ON / OFF / PWM:값', 300, 425);
}

// ==================================
// LED 밝기 변경
// ==================================
async function changeBrightness() {

  let value = brightnessSlider.value();

  // 0이면 LED 완전히 끄기 명령 전송
  if (value === 0) {
    brightness = 0;
  } else {
    brightness = value;
  }

  if (brightness !== lastBrightness) {

    const sent = await serial.send(
      'PWM:' + brightness
    );

    if (sent) {
      lastBrightness = brightness;
    }
  }
}

// ==================================
// 13번 LED 클릭 제어
// ==================================
async function mousePressed() {

  if (dist(mouseX, mouseY, 160, 175) > 55) return;

  const next = !ledOn;

  if (await serial.send(next ? 'ON' : 'OFF')) {
    ledOn = next;
  }
}

// ==================================
// Arduino 데이터 수신
// ==================================
function receiveLine(line) {

  const message = line.trim().toUpperCase();

  if (message === 'LED: ON' || message === 'ON') {
    ledOn = true;
  }
  else if (message === 'LED: OFF' || message === 'OFF') {
    ledOn = false;
  }
  else {

    const match = message.match(
      /^(?:POT\s*[:=]\s*)?(\d+)$/
    );

    if (match) {
      potValue = constrain(
        Number(match[1]), 0, 1023
      );
    }
  }
}
```

아두이노 코드:
```

#include <TaskScheduler.h>
#include <LiquidCrystal_I2C.h>
#include <string.h>
#include <stdlib.h>

// =====================================
// 1. Pin Settings
// =====================================
const uint8_t BUTTON_PIN  = 2;
const uint8_t LED_PIN     = 13;
const uint8_t PWM_LED_PIN = 3;
const uint8_t POT_PIN     = A0;

// LCD: I2C Address 0x27, 16 columns, 2 rows
LiquidCrystal_I2C lcd(0x27, 16, 2);

// =====================================
// 2. Global Variables
// =====================================
bool ledState = false;

// Last reported potentiometer value
int potValue = -1;

// PWM LED brightness (0~255)
int brightness = 0;

// false = Potentiometer control
// true  = p5.js slider control
bool pwmRemoteMode = false;

// Minimum potentiometer change
const int POT_THRESHOLD = 5;

// Interrupt variables
volatile bool buttonEvent = false;
volatile unsigned long lastInterruptTime = 0;

// Button debounce interval
const unsigned long DEBOUNCE_MS = 50;

// =====================================
// 3. Function Declarations
// =====================================
void buttonISR();
void taskButtonCallback();
void taskPotCallback();
void taskSerialCallback();
void taskLCDCallback();
void setLED(bool state);
void setPWMBrightness(int value);

// =====================================
// 4. TaskScheduler
// =====================================
Scheduler runner;

// Button event processing: 10 ms
Task tButton(
  10, TASK_FOREVER,
  &taskButtonCallback, &runner, true
);

// Potentiometer measurement: 100 ms
Task tPot(
  100, TASK_FOREVER,
  &taskPotCallback, &runner, true
);

// Serial communication: 20 ms
Task tSerial(
  20, TASK_FOREVER,
  &taskSerialCallback, &runner, true
);

// LCD updating: 100 ms
Task tLCD(
  100, TASK_FOREVER,
  &taskLCDCallback, &runner, true
);

// =====================================
// 5. External Interrupt (D2)
// =====================================
void buttonISR() {

  unsigned long now = millis();

  if (now - lastInterruptTime >= DEBOUNCE_MS) {
    buttonEvent = true;
    lastInterruptTime = now;
  }
}

// =====================================
// 6. LED Control (D13)
// =====================================
void setLED(bool state) {

  if (ledState == state) {
    return;
  }

  ledState = state;

  digitalWrite(
    LED_PIN,
    ledState ? HIGH : LOW
  );

  Serial.print("LED: ");
  Serial.println(ledState ? "ON" : "OFF");
}

// =====================================
// 7. PWM LED Control (D3)
// =====================================
void setPWMBrightness(int value) {

  // Limit brightness to 0~255
  brightness = constrain(value, 0, 255);

  // Completely turn off LED at 0
  if (brightness == 0) {
    analogWrite(PWM_LED_PIN, 0);
  }
  else {
    analogWrite(PWM_LED_PIN, brightness);
  }
}

// =====================================
// 8. Task 1: Button Event
// =====================================
void taskButtonCallback() {

  bool event = false;

  noInterrupts();

  if (buttonEvent) {
    event = true;
    buttonEvent = false;
  }

  interrupts();

  if (event) {
    setLED(!ledState);
  }
}

// =====================================
// 9. Task 2: Potentiometer
// =====================================
void taskPotCallback() {

  int value = analogRead(POT_PIN);

  // Control D3 LED only in potentiometer mode
  if (!pwmRemoteMode) {

    int pwmValue = map(
      value, 0, 1023, 0, 255
    );

    setPWMBrightness(pwmValue);
  }

  // Report potentiometer value
  if (potValue == -1 ||
      abs(value - potValue) >= POT_THRESHOLD) {

    potValue = value;

    Serial.print("POT: ");
    Serial.println(potValue);
  }
}

// =====================================
// 10. Task 3: Serial Communication
//
// Commands:
// ON       : D13 LED ON
// OFF      : D13 LED OFF
// PWM:0    : D3 LED OFF
// PWM:255  : D3 LED MAX
// AUTO     : Potentiometer mode
// =====================================
void taskSerialCallback() {

  static char command[16];
  static uint8_t index = 0;
  static bool overflow = false;

  while (Serial.available() > 0) {

    char c = Serial.read();

    if (c == '\n' || c == '\r') {

      if (overflow) {
        Serial.println("Command too long");
      }
      else if (index > 0) {

        command[index] = '\0';

        // D13 LED ON
        if (strcmp(command, "ON") == 0) {
          setLED(true);
        }

        // D13 LED OFF
        else if (strcmp(command, "OFF") == 0) {
          setLED(false);
        }

        // D3 LED brightness control
        else if (strncmp(command, "PWM:", 4) == 0) {

          char *endPtr;

          long value = strtol(
            command + 4,
            &endPtr,
            10
          );

          if (endPtr != command + 4 &&
              *endPtr == '\0' &&
              value >= 0 &&
              value <= 255) {

            // Switch to p5.js control
            pwmRemoteMode = true;

            // Set D3 LED brightness
            setPWMBrightness((int)value);

            Serial.print("PWM: ");
            Serial.println(brightness);
          }
          else {
            Serial.println("Invalid PWM value");
          }
        }

        // Return to potentiometer control
        else if (strcmp(command, "AUTO") == 0) {

          pwmRemoteMode = false;

          int value = analogRead(POT_PIN);

          int pwmValue = map(
            value, 0, 1023, 0, 255
          );

          setPWMBrightness(pwmValue);

          Serial.println("PWM MODE: AUTO");
        }

        else {
          Serial.println("Invalid command");
        }
      }

      index = 0;
      overflow = false;
    }
    else if (!overflow) {

      // Convert lowercase to uppercase
      if (c >= 'a' && c <= 'z') {
        c -= ('a' - 'A');
      }

      if (index < sizeof(command) - 1) {
        command[index++] = c;
      }
      else {
        index = 0;
        overflow = true;
      }
    }
  }
}

// =====================================
// 11. Task 4: LCD Display
// =====================================
void taskLCDCallback() {

  static bool previousLED = false;
  static int previousPOT = -1;
  static bool firstUpdate = true;

  // LCD Line 1: D13 LED Status
  if (firstUpdate || ledState != previousLED) {

    lcd.setCursor(0, 0);

    if (ledState) {
      lcd.print("LED: ON         ");
    }
    else {
      lcd.print("LED: OFF        ");
    }

    previousLED = ledState;
  }

  // LCD Line 2: Potentiometer Value
  if (firstUpdate || potValue != previousPOT) {

    lcd.setCursor(0, 1);

    lcd.print("POT: ");

    if (potValue < 0) {
      lcd.print("----");
    }
    else {
      lcd.print(potValue);
    }

    lcd.print("       ");

    previousPOT = potValue;
  }

  firstUpdate = false;
}

// =====================================
// 12. Setup
// =====================================
void setup() {

  // Serial communication
  Serial.begin(9600);

  // Pin configuration
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  pinMode(PWM_LED_PIN, OUTPUT);
  pinMode(POT_PIN, INPUT);

  // Initially turn off LEDs
  digitalWrite(LED_PIN, LOW);
  setPWMBrightness(0);

  // LCD initialization
  lcd.init();
  lcd.backlight();
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("LED: OFF");

  lcd.setCursor(0, 1);
  lcd.print("POT: ----");

  // Initialize debounce timer
  lastInterruptTime = millis() - DEBOUNCE_MS;

  // External interrupt
  attachInterrupt(
    digitalPinToInterrupt(BUTTON_PIN),
    buttonISR,
    FALLING
  );

  // Startup messages
  Serial.println("System Ready");
  Serial.println("Commands: ON / OFF / PWM:0-255 / AUTO");
  Serial.println("LED: OFF");
}

// =====================================
// 13. Main Loop
// =====================================
void loop() {

  runner.execute();

}
```

<데모>
p5.js 화면:

<img width="925" height="675" alt="image" src="https://github.com/user-attachments/assets/79d5c2ca-2bcd-4f78-bba4-2d2551c4a994" />

작동 영상:


<고찰>
이번 프로젝트를 통해 PWM 신호를 이용하여 LED의 밝기를 조절하는 원리를 이해할 수 있었다.
TaskScheduler를 활용하여 버튼 입력, 가변저항 측정, 시리얼 통신 등 여러 작업을 효율적으로 처리하는 방법을 배웠다.
p5.js와 아두이노를 연동하면서 웹 인터페이스를 통해 하드웨어를 실시간으로 제어하는 과정을 경험하였다.
특히 가변저항과 슬라이더의 제어가 충돌하는 문제를 해결하면서 제어 모드를 구분하는 것의 중요성을 알게 되었다.
이를 통해 하드웨어와 소프트웨어 간의 통신 구조를 이해하고, 문제를 분석하고 해결하는 능력을 기를 수 있었다.
