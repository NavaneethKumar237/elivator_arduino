
# pinout 

- Arduino UNO
  

| Component     | Arduino Pin |
| ------------- | ----------- |
| STEP          | D2          |
| DIR           | D5          |
| ENABLE        | D8          |
| Button G      | D9          |
| Button 1      | D10         |
| Button 2      | D11         |
| Buzzer        | D6          |
| Servo Signal  | D7          |
| IR Sensor OUT | D12         |
| OLED SDA      | A4          |
| OLED SCL      | A5          |

- Stepper Driver Wiring
  
| Driver Pin | Connect To  |
| ---------- | ----------- |
| STEP       | D2          |
| DIR        | D5          |
| EN         | D8          |
| GND        | Arduino GND |
| VDD        | Arduino 5V  |






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
```


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
# v2
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
#define IR_SENSOR 12

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

int doorClosed = 0;
int doorOpen = 115;

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
  pinMode(IR_SENSOR, INPUT);

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

  // 🚪 If already at same floor → open door
  if (floor == currentFloor) {
    playArrivalMusic();
    operateDoor();
    return;
  }

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

      operateDoor();
    }
  }
}

// =====================================================
// 🚪 DOOR CONTROL WITH SAFETY SENSOR
// =====================================================

void operateDoor() {

  // ---------- OPEN ----------
  for (int pos = doorClosed; pos <= doorOpen; pos++) {
    doorServo.write(pos);
    delay(10);
  }

  delay(3000);   // Initial wait

  // ---------- WAIT IF OBSTACLE ----------
  while (digitalRead(IR_SENSOR) == LOW) {   // LOW = obstacle detected
    delay(3000);   // Wait again
  }

  // ---------- CLOSE ----------
  for (int pos = doorOpen; pos >= doorClosed; pos--) {

    // If obstacle detected while closing
    if (digitalRead(IR_SENSOR) == LOW) {

      // Re-open
      for (int p = pos; p <= doorOpen; p++) {
        doorServo.write(p);
        delay(10);
      }

      delay(3000);   // Wait again
      pos = doorOpen;   // Restart closing
    }

    doorServo.write(pos);
    delay(10);
  }
}

// =====================================================
// 🎵 ARRIVAL MUSIC
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
// 🖥 OLED DISPLAY
// =====================================================

void updateDisplay(char arrow, int floor) {

  display.clearDisplay();

  display.setTextSize(3);
  display.setCursor(15, 5);
  display.print("Floor");
  display.setCursor(17, 5);
  display.print("Floor");   // Bold effect

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
✅ 3 Floors (G,1,2) – Real Step Control
✅ Queue Priority System
✅ Servo Auto Door
✅ IR Safety (3 sec wait logic)
✅ OLED Floor + Arrow
✅ Buzzer Music
✅ 5 LED Addressable Strip Status
✅ Auto Open if same floor button pressed

# updated lights 

- led at D10

  Install Libraries

Adafruit SSD1306

Adafruit GFX

Adafruit NeoPixel

Servo

```
**#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Servo.h>
#include <Adafruit_NeoPixel.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// ===== PINS =====
#define STEP_PIN 2
#define DIR_PIN 3

#define BTN_G 4
#define BTN_1 5
#define BTN_2 6

#define SERVO_PIN 7
#define BUZZER 8
#define IR_SENSOR 9

#define LED_PIN 10
#define LED_COUNT 5

// ===== OBJECTS =====
Servo door;
Adafruit_NeoPixel strip(LED_COUNT, LED_PIN, NEO_GRB + NEO_KHZ800);

// ===== FLOOR SETTINGS =====
long stepsPerFloor = 2000;  // Adjust if needed
int currentFloor = 0;
long currentPosition = 0;

int queue[5];
int queueSize = 0;

// ================= LED =================
void setColor(int r,int g,int b){
  for(int i=0;i<LED_COUNT;i++)
    strip.setPixelColor(i,strip.Color(r,g,b));
  strip.show();
}

void flashRed(){
  setColor(255,0,0);
  delay(200);
  strip.clear();
  strip.show();
  delay(200);
}

// ================= OLED =================
void showFloor(int floor, String dir){
  display.clearDisplay();
  display.setTextSize(3);
  display.setCursor(30,10);
  display.print("F:");
  display.print(floor);
  display.setTextSize(2);
  display.setCursor(45,45);
  display.print(dir);
  display.display();
}

// ================= BUZZER =================
void arrivalTone(){
  tone(BUZZER, 1000, 200);
  delay(200);
  tone(BUZZER, 1500, 200);
  delay(200);
  noTone(BUZZER);
}

// ================= QUEUE =================
void addToQueue(int floor){
  if(queueSize<5){
    queue[queueSize++] = floor;
  }
}

// ================= DOOR =================
void openDoor(){
  door.write(90);
  setColor(0,255,0);
  delay(3000);
}

void closeDoor(){
  unsigned long waitTime = millis();
  while(millis()-waitTime < 3000){
    if(digitalRead(IR_SENSOR)==LOW){
      waitTime = millis(); // reset timer
    }
  }

  for(int i=0;i<5;i++) flashRed();
  door.write(0);
}

// ================= MOTOR =================
void moveToFloor(int target){
  long targetPos = target * stepsPerFloor;
  long steps = targetPos - currentPosition;

  if(steps>0){
    digitalWrite(DIR_PIN,HIGH);
    showFloor(currentFloor,"UP");
    setColor(0,0,255);
  }else{
    digitalWrite(DIR_PIN,LOW);
    showFloor(currentFloor,"DOWN");
    setColor(150,0,150);
  }

  steps = abs(steps);

  for(long i=0;i<steps;i++){
    digitalWrite(STEP_PIN,HIGH);
    delayMicroseconds(800);
    digitalWrite(STEP_PIN,LOW);
    delayMicroseconds(800);
  }

  currentPosition = targetPos;
  currentFloor = target;

  arrivalTone();
  showFloor(currentFloor,"STOP");
  openDoor();
  closeDoor();
}

// ================= SETUP =================
void setup(){

  pinMode(STEP_PIN,OUTPUT);
  pinMode(DIR_PIN,OUTPUT);

  pinMode(BTN_G,INPUT_PULLUP);
  pinMode(BTN_1,INPUT_PULLUP);
  pinMode(BTN_2,INPUT_PULLUP);

  pinMode(IR_SENSOR,INPUT);
  pinMode(BUZZER,OUTPUT);

  door.attach(SERVO_PIN);
  door.write(0);

  strip.begin();
  strip.show();

  display.begin(SSD1306_SWITCHCAPVCC,0x3C);
  display.clearDisplay();
  display.display();

  showFloor(0,"READY");
}

// ================= LOOP =================
void loop(){

  if(digitalRead(BTN_G)==LOW){
    if(currentFloor==0){
      openDoor();
    }else{
      addToQueue(0);
    }
    delay(300);
  }

  if(digitalRead(BTN_1)==LOW){
    if(currentFloor==1){
      openDoor();
    }else{
      addToQueue(1);
    }
    delay(300);
  }

  if(digitalRead(BTN_2)==LOW){
    if(currentFloor==2){
      openDoor();
    }else{
      addToQueue(2);
    }
    delay(300);
  }

  if(queueSize>0){
    int nextFloor = queue[0];

    for(int i=0;i<queueSize-1;i++)
      queue[i]=queue[i+1];

    queueSize--;

    moveToFloor(nextFloor);
  }
}
```**
