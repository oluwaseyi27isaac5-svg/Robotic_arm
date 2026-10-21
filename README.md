#include <Servo.h>

const byte SERVO_COUNT = 6;

const byte servoPins[SERVO_COUNT] = {
  3, 5, 6, 9, 10, 11
};

const byte potPins[SERVO_COUNT] = {
  A0, A1, A2, A3, A4, A5
};

Servo servos[SERVO_COUNT];

const byte trigPin = 7;
const byte echoPin = 8;

const byte buzzerPin = 12;

const float WARNING_DISTANCE = 30.0;
const float STOP_DISTANCE = 20.0;

enum ControlMode {
  MANUAL_MODE,
  AUTONOMOUS_MODE
};

ControlMode currentMode = MANUAL_MODE;

// ---------- SAFETY STATE ----------

float distanceCm = -1.0;

bool obstacleWarning = false;
bool criticalObstacle = false;

// ---------- AUTONOMOUS MOVEMENT ----------

const int poseCount = 4;

const int poses[poseCount][SERVO_COUNT] = {
  {90, 90, 90, 90, 90, 90},
  {70, 110, 80, 100, 90, 60},
  {110, 70, 100, 80, 60, 120},
  {90, 90, 90, 90, 90, 90}
};

byte currentPose = 0;

unsigned long lastPoseTime = 0;
const unsigned long POSE_INTERVAL = 1200;

// ---------- SENSOR TIMING ----------

unsigned long lastSensorTime = 0;
const unsigned long SENSOR_INTERVAL = 100;

// ---------- BUZZER TIMING ----------

unsigned long lastBeepTime = 0;
bool beepState = false;

// ---------- FUNCTION DECLARATIONS ----------

float readDistance();
void readManualControls();
void updateSafetyState();
void updateBuzzer();
void runAutonomousMode();
void processSerialCommands();
void moveToPose(byte poseIndex);
void stopAutonomousMotion();

// ======================================
// SETUP
// ======================================

void setup() {

  Serial.begin(9600);

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  pinMode(buzzerPin, OUTPUT);

  digitalWrite(trigPin, LOW);

  for (byte i = 0; i < SERVO_COUNT; i++) {

    servos[i].attach(servoPins[i]);

    servos[i].write(90);
  }

  Serial.println();
  Serial.println("SMARTREACH ROBOTIC ARM");
  Serial.println("----------------------");
  Serial.println("M = Manual mode");
  Serial.println("A = Autonomous mode");
  Serial.println("S = Stop autonomous motion");
  Serial.println();

  tone(buzzerPin, 1800, 120);

  delay(300);
}


void loop() {

  processSerialCommands();

  unsigned long now = millis();

  if (now - lastSensorTime >= SENSOR_INTERVAL) {

    lastSensorTime = now;

    distanceCm = readDistance();

    updateSafetyState();
  }

  updateBuzzer();

  if (currentMode == MANUAL_MODE) {

    readManualControls();

  } else if (currentMode == AUTONOMOUS_MODE) {

    if (!obstacleWarning) {

      runAutonomousMode();

    } else {

      stopAutonomousMotion();
    }
  }
}

float readDistance() {

  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);

  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);

  digitalWrite(trigPin, LOW);

  unsigned long duration =
      pulseIn(echoPin, HIGH, 25000UL);

  if (duration == 0) {
    return -1.0;
  }

  float measuredDistance =
      duration * 0.0343 / 2.0;

  return measuredDistance;
}

void readManualControls() {

  for (byte i = 0; i < SERVO_COUNT; i++) {

    int potValue = analogRead(potPins[i]);

    int angle = map(
      potValue,
      0,
      1023,
      0,
      180
    );

    servos[i].write(angle);
  }
}

void updateSafetyState() {

  if (distanceCm < 0) {

    obstacleWarning = true;
    criticalObstacle = false;

    Serial.println(
      "Sensor reading invalid: autonomous motion paused."
    );

    return;
  }

  if (distanceCm <= STOP_DISTANCE) {

    obstacleWarning = true;
    criticalObstacle = true;

  } else if (distanceCm <= WARNING_DISTANCE) {

    obstacleWarning = true;
    criticalObstacle = false;

  } else {

    obstacleWarning = false;
    criticalObstacle = false;
  }

  Serial.print("Distance: ");
  Serial.print(distanceCm);
  Serial.print(" cm | ");

  if (criticalObstacle) {

    Serial.println("CRITICAL: STOP");

  } else if (obstacleWarning) {

    Serial.println("WARNING: OBSTACLE NEAR");

  } else {

    Serial.println("AREA OUTSIDE WARNING ZONE");
  }
}


void updateBuzzer() {

  unsigned long now = millis();

  if (!obstacleWarning) {

    noTone(buzzerPin);
    beepState = false;

    return;
  }

  if (criticalObstacle) {

    tone(buzzerPin, 2500);

    return;
  }

  if (now - lastBeepTime >= 180) {

    lastBeepTime = now;

    beepState = !beepState;

    if (beepState) {

      tone(buzzerPin, 2000);

    } else {

      noTone(buzzerPin);
    }
  }
}

void runAutonomousMode() {

  unsigned long now = millis();

  if (now - lastPoseTime >= POSE_INTERVAL) {

    lastPoseTime = now;

    moveToPose(currentPose);

    currentPose++;

    if (currentPose >= poseCount) {
      currentPose = 0;
    }
  }
}

void moveToPose(byte poseIndex) {

  for (byte i = 0; i < SERVO_COUNT; i++) {

    servos[i].write(
      poses[poseIndex][i]
    );
  }

  Serial.print("Autonomous pose: ");
  Serial.println(poseIndex);
}

void stopAutonomousMotion() {
}

void processSerialCommands() {

  while (Serial.available() > 0) {

    char command = Serial.read();

    if (command == 'M' || command == 'm') {

      currentMode = MANUAL_MODE;

      noTone(buzzerPin);

      Serial.println("Manual mode selected.");

    } else if (command == 'A' || command == 'a') {

      currentMode = AUTONOMOUS_MODE;

      currentPose = 0;
      lastPoseTime = millis();

      Serial.println("Autonomous mode selected.");

    } else if (command == 'S' || command == 's') {

      currentMode = MANUAL_MODE;

      noTone(buzzerPin);

      Serial.println(
        "Autonomous sequence stopped. Manual mode active."
      );![Uploading image.png…]()

    }
  }
}
