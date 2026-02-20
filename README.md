# test code 
```
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define OLED_RESET -1
Adafruit_SSD1306 display(128, 64, &Wire, OLED_RESET);

// ---------------- PINS ----------------
#define STEP_PIN 2
#define DIR_PIN 5
#define EN_PIN 8

#define BTN_G 9
#define BTN_1 10
#define BTN_2 11

#define BUZZER 6

// ---------------- FLOOR POSITIONS ----------------
// Adjust if needed
const long floorPosition[3] = {0, 1500, 3000};

// ---------------- VARIABLES ----------------
long currentPosition = 0;
long targetPosition = 0;

int currentFloor = 0;
int targetFloor = -1;

bool moving = false;
int direction = 1;

unsigned long lastStepTime = 0;
int stepDelay = 1000;   // Increase if vibration

// =====================================================

void setup() {

  Wire.begin();
  Wire.setClock(100000);

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  pinMode(STEP_PIN, OUTPUT);
  pinMode(DIR_PIN, OUTPUT);
  pinMode(EN_PIN, OUTPUT);
  digitalWrite(EN_PIN, LOW);

  pinMode(BTN_G, INPUT_PULLUP);
  pinMode(BTN_1, INPUT_PULLUP);
  pinMode(BTN_2, INPUT_PULLUP);

  pinMode(BUZZER, OUTPUT);

  updateDisplay('-', 0);
}

// =====================================================

void loop() {

  checkButtons();

  if (moving) {
    moveStepper();
  }
}

// =====================================================

void checkButtons() {

  if (!moving) {

    if (digitalRead(BTN_G) == LOW) setTarget(0);
    if (digitalRead(BTN_1) == LOW) setTarget(1);
    if (digitalRead(BTN_2) == LOW) setTarget(2);
  }
}

// =====================================================

void setTarget(int floor) {

  if (floor == currentFloor) return;

  targetFloor = floor;
  targetPosition = floorPosition[floor];

  if (targetPosition > currentPosition) {
    direction = 1;
    digitalWrite(DIR_PIN, HIGH);
  } else {
    direction = -1;
    digitalWrite(DIR_PIN, LOW);
  }

  moving = true;
  updateDisplay(direction == 1 ? '^' : 'v', currentFloor);
}

// =====================================================

void moveStepper() {

  if (micros() - lastStepTime >= stepDelay) {

    lastStepTime = micros();

    if (currentPosition != targetPosition) {

      digitalWrite(STEP_PIN, HIGH);
      delayMicroseconds(5);
      digitalWrite(STEP_PIN, LOW);

      currentPosition += direction;

    } else {

      moving = false;
      currentFloor = targetFloor;

      playArrivalMusic();
      updateDisplay('-', currentFloor);
    }
  }
}

// =====================================================
// 🎵 Elevator Arrival Music
// =====================================================

void playArrivalMusic() {

  int melody[] = {523, 659, 784, 1046}; // C5 E5 G5 C6
  int duration[] = {150, 150, 150, 300};

  for (int i = 0; i < 4; i++) {
    tone(BUZZER, melody[i], duration[i]);
    delay(duration[i] + 50);
  }

  noTone(BUZZER);
}

// =====================================================
// 🖥 OLED Display (Bold Effect)
// =====================================================

void updateDisplay(char arrow, int floor) {

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);

  // FLOOR LABEL
  display.setTextSize(3);
  display.setCursor(15, 5);
  display.print("Floor");
  display.setCursor(17, 5);
  display.print("Floor");   // bold effect

  // FLOOR NUMBER
  display.setTextSize(4);

  if (floor == 0) {
    display.setCursor(50, 30);
    display.print("G");
    display.setCursor(52, 30);
    display.print("G");
  } else {
    display.setCursor(50, 30);
    display.print(floor);
    display.setCursor(52, 30);
    display.print(floor);
  }

  // DIRECTION
  display.setTextSize(3);
  display.setCursor(100, 40);

  if (arrow == '^') display.print("^");
  else if (arrow == 'v') display.print("v");
  else display.print("-");

  display.display();
}
``


# v1 test

```
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Servo.h>

#define OLED_RESET -1
Adafruit_SSD1306 display(128, 64, &Wire, OLED_RESET);

// ---------------- PINS ----------------
#define STEP_PIN 2
#define DIR_PIN 5
#define EN_PIN 8

#define BTN_G 9
#define BTN_1 10
#define BTN_2 11

#define BUZZER 6
#define SERVO_PIN 7

Servo doorServo;

// ---------------- FLOOR POSITIONS ----------------
const long floorPosition[3] = {0, 1500, 3000};

// ---------------- VARIABLES ----------------
long currentPosition = 0;
long targetPosition = 0;

int currentFloor = 0;
int targetFloor = -1;

bool moving = false;
int direction = 1;

unsigned long lastStepTime = 0;
int stepDelay = 1000;

// Door angles
int doorClosed = 0;
int doorOpen = 125;

// =====================================================

void setup() {

  Wire.begin();
  Wire.setClock(100000);

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  display.setTextColor(SSD1306_WHITE);

  pinMode(STEP_PIN, OUTPUT);
  pinMode(DIR_PIN, OUTPUT);
  pinMode(EN_PIN, OUTPUT);
  digitalWrite(EN_PIN, LOW);

  pinMode(BTN_G, INPUT_PULLUP);
  pinMode(BTN_1, INPUT_PULLUP);
  pinMode(BTN_2, INPUT_PULLUP);

  pinMode(BUZZER, OUTPUT);

  doorServo.attach(SERVO_PIN);
  doorServo.write(doorClosed);

  updateDisplay('-', 0);
}

// =====================================================

void loop() {

  checkButtons();

  if (moving) {
    moveStepper();
  }
}

// =====================================================

void checkButtons() {

  if (!moving) {

    if (digitalRead(BTN_G) == LOW) setTarget(0);
    if (digitalRead(BTN_1) == LOW) setTarget(1);
    if (digitalRead(BTN_2) == LOW) setTarget(2);
  }
}

// =====================================================

void setTarget(int floor) {

  if (floor == currentFloor) return;

  targetFloor = floor;
  targetPosition = floorPosition[floor];

  if (targetPosition > currentPosition) {
    direction = 1;
    digitalWrite(DIR_PIN, HIGH);
  } else {
    direction = -1;
    digitalWrite(DIR_PIN, LOW);
  }

  moving = true;
  updateDisplay(direction == 1 ? '^' : 'v', currentFloor);
}

// =====================================================

void moveStepper() {

  if (micros() - lastStepTime >= stepDelay) {

    lastStepTime = micros();

    if (currentPosition != targetPosition) {

      digitalWrite(STEP_PIN, HIGH);
      delayMicroseconds(5);
      digitalWrite(STEP_PIN, LOW);

      currentPosition += direction;

    } else {

      moving = false;
      currentFloor = targetFloor;

      playArrivalMusic();
      updateDisplay('-', currentFloor);

      operateDoor();   // 🚪 AUTO DOOR
    }
  }
}

// =====================================================
// 🚪 DOOR FUNCTION
// =====================================================

void operateDoor() {

  // Open
  for (int pos = doorClosed; pos <= doorOpen; pos++) {
    doorServo.write(pos);
    delay(10);
  }

  delay(5000);  // Door open time (3 seconds)

  // Close
  for (int pos = doorOpen; pos >= doorClosed; pos--) {
    doorServo.write(pos);
    delay(10);
  }
}

// =====================================================
// 🎵 MUSIC
// =====================================================

void playArrivalMusic() {

  int melody[] = {523, 659, 784, 1046};
  int duration[] = {150, 150, 150, 300};

  for (int i = 0; i < 4; i++) {
    tone(BUZZER, melody[i], duration[i]);
    delay(duration[i] + 50);
  }

  noTone(BUZZER);
}

// =====================================================
// 🖥 DISPLAY
// =====================================================

void updateDisplay(char arrow, int floor) {

  display.clearDisplay();

  display.setTextSize(3);
  display.setCursor(15, 5);
  display.print("Floor");
  display.setCursor(17, 5);
  display.print("Floor");

  display.setTextSize(4);

  if (floor == 0) {
    display.setCursor(50, 30);
    display.print("G");
    display.setCursor(52, 30);
    display.print("G");
  } else {
    display.setCursor(50, 30);
    display.print(floor);
    display.setCursor(52, 30);
    display.print(floor);
  }

  display.setTextSize(3);
  display.setCursor(100, 40);

  if (arrow == '^') display.print("^");
  else if (arrow == 'v') display.print("v");
  else display.print("-");

  display.display();
}
```
