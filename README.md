#include <BleGamepad.h>
#include <Keypad.h>
#include <ESP32Encoder.h>

// Configurações do gamepad
const char* gamepadName = "Aro Audi";
const char* gamepadManufacturer = "breeze";
const uint8_t gamepadBatteryLevel = 100;

// Configurações da matriz de botões
const uint8_t ROWS = 6;
const uint8_t COLS = 2;
uint8_t rowPins[ROWS] = {33, 32, 27, 26, 25, 13};
uint8_t colPins[COLS] = {22, 23};
uint8_t keymap[ROWS][COLS] = {
  {1, 2}, {3, 4}, {5, 6}, {17, 18}, {9, 10}, {11, 12}
};

// Pinos dos botões independentes
const uint8_t BUTTON_13_OUTPUT_PIN = 4;
const uint8_t BUTTON_13_INPUT_PIN = 5;
const uint8_t BUTTON_14_OUTPUT_PIN = 14;
const uint8_t BUTTON_14_INPUT_PIN = 15;

// Configurações dos encoders
const uint8_t MAXENC = 2;
uint8_t encoderUppPin[MAXENC] = {18, 19};
uint8_t encoderDwnPin[MAXENC] = {17, 21};
uint8_t encoderUppButton[MAXENC] = {15, 7}; // Mapeamento dos botões dos encoders
uint8_t encoderDwnButton[MAXENC] = {16, 8};
ESP32Encoder encoder[MAXENC];
unsigned long holdoff[MAXENC] = {0, 0};
int32_t prevCount[MAXENC] = {0, 0};
const unsigned long HOLDOFFTIME = 150;

// Declaração das instâncias
Keypad customKeypad = Keypad(makeKeymap(keymap), rowPins, colPins, ROWS, COLS);
BleGamepad bleGamepad(gamepadName, gamepadManufacturer, gamepadBatteryLevel);

void setup() {
  Serial.begin(115200);
  setCpuFrequencyMhz(80);
  
  

  for (uint8_t i = 0; i < ROWS; i++) {
    pinMode(rowPins[i], OUTPUT);
    digitalWrite(rowPins[i], HIGH);
  }
  for (uint8_t i = 0; i < COLS; i++) {
    pinMode(colPins[i], INPUT_PULLUP);
  }

  pinMode(BUTTON_13_OUTPUT_PIN, OUTPUT);
  digitalWrite(BUTTON_13_OUTPUT_PIN, LOW);
  pinMode(BUTTON_13_INPUT_PIN, INPUT_PULLUP);

  pinMode(BUTTON_14_OUTPUT_PIN, OUTPUT);
  digitalWrite(BUTTON_14_OUTPUT_PIN, LOW);
  pinMode(BUTTON_14_INPUT_PIN, INPUT_PULLUP);

  for (uint8_t i = 0; i < MAXENC; i++) {
    encoder[i].attachSingleEdge(encoderDwnPin[i], encoderUppPin[i]);
    encoder[i].clearCount();
  }

  customKeypad.addEventListener(keypadEvent);
  customKeypad.setHoldTime(1);
  customKeypad.setDebounceTime(50);
  bleGamepad.begin();
  Serial.println("BLE Gamepad iniciado!");
}

void loop() {
  if (bleGamepad.isConnected()) {
    customKeypad.getKey();

    bool button13Pressed = digitalRead(BUTTON_13_INPUT_PIN) == HIGH;
    bool button14Pressed = digitalRead(BUTTON_14_INPUT_PIN) == HIGH;

    if (button13Pressed) {
      bleGamepad.press(13);
    } else {
      bleGamepad.release(13);
    }
    if (button14Pressed) {
      bleGamepad.press(14);
    } else {
      bleGamepad.release(14);
    }

    unsigned long now = millis();
    for (uint8_t i = 0; i < MAXENC; i++) {
      int32_t count = encoder[i].getCount();
      if (count != prevCount[i]) {
        if (!holdoff[i]) {
          if (count > prevCount[i]) {
            bleGamepad.press(encoderUppButton[i]);
            delay(50);
            bleGamepad.release(encoderUppButton[i]);
          } else if (count < prevCount[i]) {
            bleGamepad.press(encoderDwnButton[i]);
            delay(50);
            bleGamepad.release(encoderDwnButton[i]);
          }
          holdoff[i] = now;
        } else if (now - holdoff[i] > HOLDOFFTIME) {
          prevCount[i] = count;
          holdoff[i] = 0;
        }
      }
    }

    bleGamepad.sendReport();
    delay(100);
  }
}

void keypadEvent(KeypadEvent key) {
  uint8_t keystate = customKeypad.getState();
  if (keystate == PRESSED) {
    bleGamepad.press(key);
  }
  if (keystate == RELEASED) {
    bleGamepad.release(key);
  }
}
