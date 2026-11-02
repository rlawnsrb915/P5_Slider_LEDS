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
