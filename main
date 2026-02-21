/*
 * ROBOTIC ARM SEQUENCE CONTROLLER
 * Arduino R4 WiFi with Adafruit 16-Channel PWM Servo Shield
 * 
 * PURPOSE: Execute pick-and-place sequences with button controls
 * CONTROL: Xbox controller buttons via Android app
 * 
 * INITIALIZATION (Home Position):
 * - Servo 0 (Base) → 17°
 * - Servo 1 (Shoulder) → 50°
 * - Servo 2 (Elbow) → 40°
 * - Servo 3 (Wrist) → 126°
 * - Servo 4 (Wrist Rotation) → 56°
 * - Servo 5 (Gripper) → 0°
 * 
 * A BUTTON - APPROACH:
 * S0→0°, S1→0°, S2→32°, S3→108°, S4→85°, S5→98°
 * 
 * B BUTTON - GRAB:
 * S5→115° (close gripper)
 * 
 * X BUTTON - DROP:
 * S0→80°, S1→130°, S4→97°, S4→100°
 * 
 * Y BUTTON - RELEASE:
 * S5→0° (open gripper)
 * 
 * START BUTTON - RESET TO HOME:
 * Returns all servos to initialization positions
 * 
 * SERVO TO SHIELD PORT MAPPING:
 * Servo 0 → Shield Port 1  (Base)
 * Servo 1 → Shield Port 4  (Shoulder)
 * Servo 2 → Shield Port 7  (Elbow)
 * Servo 3 → Shield Port 8  (Wrist)
 * Servo 4 → Shield Port 11 (Wrist Rotation)
 * Servo 5 → Shield Port 15 (Gripper)
 */

 #include <Wire.h>
 #include <Adafruit_PWMServoDriver.h>
 #include <ArduinoBLE.h>
 
 // Adafruit Servo Shield
 Adafruit_PWMServoDriver pwm = Adafruit_PWMServoDriver();
 
 #define SERVOMIN  150
 #define SERVOMAX  600
 #define SERVO_FREQ 50
 
 // BLE Service and Characteristic
 BLEService roboticArmService("12345678-1234-1234-1234-123456789ABC");
 BLEStringCharacteristic commandCharacteristic("87654321-4321-4321-4321-CBA987654321", BLERead | BLEWrite, 20);
 
 // CRITICAL: Servo index to physical shield port mapping
 const uint8_t servoPorts[6] = {1, 4, 7, 8, 11, 15};
 const char* servoNames[6] = {"Base", "Shoulder", "Elbow", "Wrist", "Wrist Rot", "Gripper"};
 
// Current positions for all servos
int currentPositions[6] = {90, 90, 90, 90, 90, 0};

// Initial positions for the robotic arm
const int INIT_POSITIONS[6] = {17, 40, 40, 126, 56, 0};

// State tracking for sequences
bool isExecutingSequence = false;
enum SequenceState { IDLE, SEQUENCE_APPROACH, SEQUENCE_GRAB, SEQUENCE_DROP, SEQUENCE_RELEASE };
SequenceState currentSequence = IDLE;

// Wide range for calibration (will be refined)
const int MIN_SAFE = 0;
const int MAX_SAFE = 180;

// Smooth movement speed (delay between steps in ms)
const int SMOOTH_DELAY = 15;
 
 void setup() {
   Serial.begin(9600);
   delay(2000);
 
  Serial.println("\n========================================");
  Serial.println("ROBOTIC ARM SEQUENCE CONTROLLER");
  Serial.println("========================================");
  Serial.println("🤖 PICK & PLACE MODE ACTIVE");
  Serial.println("========================================");
  Serial.println("SERVO → PORT MAPPING:");
  for (int i = 0; i < 6; i++) {
    Serial.print("  Servo ");
    Serial.print(i);
    Serial.print(" (");
    Serial.print(servoNames[i]);
    Serial.print(") → Shield Port ");
    Serial.println(servoPorts[i]);
  }
  Serial.println("========================================");
  Serial.println("Controls:");
  Serial.println("  A Button = APPROACH object");
  Serial.println("  B Button = GRAB object");
  Serial.println("  X Button = DROP position");
  Serial.println("  Y Button = RELEASE object");
  Serial.println("  START Button = RESET to home");
  Serial.println("========================================\n");
 
   // Initialize servo shield
   Serial.println("Initializing Adafruit PWM Servo Shield...");
   pwm.begin();
   pwm.setOscillatorFrequency(27000000);
   pwm.setPWMFreq(SERVO_FREQ);
   delay(10);
   Serial.println("✓ Shield initialized");
 
  // Initialize all servos to their starting positions (one after another)
  Serial.println("Initializing robotic arm to home position...");
  Serial.println("Moving servos sequentially:\n");
  
  for (int i = 0; i < 6; i++) {
    Serial.print("  Servo ");
    Serial.print(i);
    Serial.print(" (");
    Serial.print(servoNames[i]);
    Serial.print(") → ");
    Serial.print(INIT_POSITIONS[i]);
    Serial.println("°");
    
    setServoAngle(servoPorts[i], INIT_POSITIONS[i]);
    currentPositions[i] = INIT_POSITIONS[i];
    delay(800); // Wait between each servo movement
  }
  
  Serial.println("\n✓ All servos initialized to home position");
  Serial.println("✓ Ready for commands!\n");
 
   // Initialize BLE
   Serial.println("Initializing BLE...");
   if (!BLE.begin()) {
     Serial.println("✗ BLE initialization failed!");
     while (1);
   }
 
   BLE.setLocalName("Arduino Robotic Arm");
   BLE.setAdvertisedService(roboticArmService);
   roboticArmService.addCharacteristic(commandCharacteristic);
   BLE.addService(roboticArmService);
   BLE.advertise();
 
   Serial.println("✓ BLE initialized");
   Serial.println("✓ Advertising as 'Arduino Robotic Arm'");
   Serial.println("\n⚠️  CALIBRATION MODE ACTIVE");
   Serial.println("⚠️  Test each servo carefully!");
   Serial.println("⚠️  Stop immediately if you hear grinding/strain\n");
   Serial.println("Waiting for Android app connection...\n");
 }
 
 void loop() {
   BLEDevice central = BLE.central();
 
   if (central) {
     Serial.println("========================================");
     Serial.print("✓ Connected to: ");
     Serial.println(central.address());
     Serial.println("========================================\n");
 
     while (central.connected()) {
       if (commandCharacteristic.written()) {
         String command = commandCharacteristic.value();
         command.trim();
         
         Serial.println(">>> COMMAND RECEIVED <<<");
         Serial.print("Raw: '");
         Serial.print(command);
         Serial.print("' (length: ");
         Serial.print(command.length());
         Serial.println(")");
         
         processCommand(command);
         
         Serial.println();
       }
       
       delay(10);
     }
 
     Serial.println("\n========================================");
     Serial.println("✗ Disconnected from app");
     Serial.println("========================================\n");
   }
 }
 
void processCommand(String command) {
  // Check if already executing a sequence
  if (isExecutingSequence) {
    Serial.println("⚠️  Sequence already in progress, please wait...");
    return;
  }
  
  // A Button - Approach Sequence
  if (command == "BTN_A" || command == "A") {
    Serial.println("\n>>> A BUTTON PRESSED <<<");
    Serial.println("Starting APPROACH sequence...\n");
    isExecutingSequence = true;
    currentSequence = SEQUENCE_APPROACH;
    executeSequenceApproach();
    isExecutingSequence = false;
    currentSequence = IDLE;
    Serial.println("✓ Approach sequence complete!\n");
  }
  // B Button - Grab Sequence
  else if (command == "BTN_B" || command == "B") {
    Serial.println("\n>>> B BUTTON PRESSED <<<");
    Serial.println("Starting GRAB sequence...\n");
    isExecutingSequence = true;
    currentSequence = SEQUENCE_GRAB;
    executeSequenceGrab();
    isExecutingSequence = false;
    currentSequence = IDLE;
    Serial.println("✓ Grab sequence complete!\n");
  }
  // X Button - Drop Sequence
  else if (command == "BTN_X" || command == "X") {
    Serial.println("\n>>> X BUTTON PRESSED <<<");
    Serial.println("Starting DROP sequence...\n");
    isExecutingSequence = true;
    currentSequence = SEQUENCE_DROP;
    executeSequenceDrop();
    isExecutingSequence = false;
    currentSequence = IDLE;
    Serial.println("✓ Drop sequence complete!\n");
  }
  // Y Button - Release Sequence
  else if (command == "BTN_Y" || command == "Y") {
    Serial.println("\n>>> Y BUTTON PRESSED <<<");
    Serial.println("Starting RELEASE sequence...\n");
    isExecutingSequence = true;
    currentSequence = SEQUENCE_RELEASE;
    executeSequenceRelease();
    isExecutingSequence = false;
    currentSequence = IDLE;
    Serial.println("✓ Release sequence complete!\n");
  }
  // RESET Button - Return to initialized positions
  else if (command == "RESET" || command == "BTN_START") {
    Serial.println("\n>>> RESET TO HOME POSITION <<<");
    Serial.println("Returning to initialized positions...\n");
    for (int i = 0; i < 6; i++) {
      Serial.print("Servo ");
      Serial.print(i);
      Serial.print(" → ");
      Serial.print(INIT_POSITIONS[i]);
      Serial.println("°");
      moveServoSmooth(i, INIT_POSITIONS[i]);
      delay(300);
    }
    Serial.println("✓ Reset complete!\n");
  }
  else if (command == "STATUS") {
    printStatus();
  }
  else if (command == "MAP") {
    printMapping();
  }
  else {
    // For debugging - show what command was received
    if (command.length() > 0) {
      Serial.print("⚠️  Command not recognized: ");
      Serial.println(command);
    }
  }
}
 
void moveServoInstant(int servoIndex, int targetPosition) {
  // For manual adjustments - instant movement
  int originalPosition = currentPositions[servoIndex];
  
  // Safety check
  if (targetPosition < MIN_SAFE) {
    targetPosition = MIN_SAFE;
  }
  if (targetPosition > MAX_SAFE) {
    targetPosition = MAX_SAFE;
  }

  Serial.print("S");
  Serial.print(servoIndex);
  Serial.print(": ");
  Serial.print(originalPosition);
  Serial.print("° → ");
  Serial.print(targetPosition);
  Serial.println("°");
  
  setServoAngle(servoPorts[servoIndex], targetPosition);
  currentPositions[servoIndex] = targetPosition;
}

void moveServoSmooth(int servoIndex, int targetPosition) {
  int originalPosition = currentPositions[servoIndex];
  
  // Safety check
  if (targetPosition < MIN_SAFE) {
    targetPosition = MIN_SAFE;
  }
  if (targetPosition > MAX_SAFE) {
    targetPosition = MAX_SAFE;
  }

  Serial.print("Moving Servo ");
  Serial.print(servoIndex);
  Serial.print(" (");
  Serial.print(servoNames[servoIndex]);
  Serial.print("): ");
  Serial.print(originalPosition);
  Serial.print("° → ");
  Serial.print(targetPosition);
  Serial.print("° ... ");
  
  // Smooth movement - move 1 degree at a time
  int direction = (targetPosition > originalPosition) ? 1 : -1;
  int currentPos = originalPosition;
  
  while (currentPos != targetPosition) {
    currentPos += direction;
    setServoAngle(servoPorts[servoIndex], currentPos);
    delay(SMOOTH_DELAY);
  }
  
  currentPositions[servoIndex] = targetPosition;
  Serial.println("✓ Done");
}

void executeSequenceApproach() {
  // A Button Sequence: Approach Object
  // Move all servos to approach position
  Serial.println("Step 1: Moving Base (S0) to 0°...");
  moveServoSmooth(0, 0);
  delay(300);
  
  Serial.println("Step 2: Moving Shoulder (S1) to 0°...");
  moveServoSmooth(1, 0);
  delay(300);
  
  Serial.println("Step 3: Moving Elbow (S2) to 32°...");
  moveServoSmooth(2, 32);
  delay(300);
  
  Serial.println("Step 4: Moving Wrist (S3) to 108°...");
  moveServoSmooth(3, 108);
  delay(300);
  
  Serial.println("Step 5: Moving Wrist Rotation (S4) to 85°...");
  moveServoSmooth(4, 85);
  delay(300);
  
  Serial.println("Step 6: Positioning Gripper (S5) to 98°...");
  moveServoSmooth(5, 98);
  delay(500);
}

void executeSequenceGrab() {
  // B Button Sequence: Grab Object
  // Close gripper
  Serial.println("Closing gripper to grab object...");
  moveServoSmooth(5, 115);
  delay(500);
}

void executeSequenceDrop() {
  // X Button Sequence: Drop Position
  Serial.println("Step 1: Moving Base (S0) to 80°...");
  moveServoSmooth(0, 80);
  delay(300);
  
  Serial.println("Step 2: Moving Shoulder (S1) to 130°...");
  moveServoSmooth(1, 110);
  delay(300);
  
  Serial.println("Step 3: Moving Wrist (S3) to 97°...");
  moveServoSmooth(3, 97);
  delay(300);
  
  Serial.println("Step 4: Moving Wrist Rotation (S4) to 100°...");
  moveServoSmooth(4, 100);
  delay(500);
}

void executeSequenceRelease() {
  // Y Button Sequence: Release Object
  // Open gripper
  Serial.println("Opening gripper to release object...");
  moveServoSmooth(5, 0);
  delay(500);
}
 
 void setServoAngle(uint8_t port, double angle) {
   // Convert angle (0-180) to pulse length (SERVOMIN-SERVOMAX)
   double pulse = map(angle, 0, 180, SERVOMIN, SERVOMAX);
   
   Serial.print("  [Shield Port ");
   Serial.print(port);
   Serial.print("] PWM: ");
   Serial.println(pulse);
   
   pwm.setPWM(port, 0, pulse);
   delay(15);
 }
 
 void printStatus() {
   Serial.println("\n=== CURRENT SERVO POSITIONS ===");
   for (int i = 0; i < 6; i++) {
     Serial.print("Servo ");
     Serial.print(i);
     Serial.print(" (");
     Serial.print(servoNames[i]);
     Serial.print(", Port ");
     Serial.print(servoPorts[i]);
     Serial.print("): ");
     Serial.print(currentPositions[i]);
     Serial.println("°");
   }
   Serial.println("===============================\n");
 }
 
 void printMapping() {
   Serial.println("\n=== SERVO → PORT MAPPING ===");
   for (int i = 0; i < 6; i++) {
     Serial.print("Servo ");
     Serial.print(i);
     Serial.print(" → Port ");
     Serial.print(servoPorts[i]);
     Serial.print(" (");
     Serial.print(servoNames[i]);
     Serial.println(")");
   }
   Serial.println("============================\n");
 }
 
