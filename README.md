#define BOUNCE_WITH_PROMPT_DETECTION // Ativa a detecção de debounce para evitar múltiplas leituras acidentais

#include <BleGamepad.h> // Biblioteca para emular um gamepad Bluetooth
#include <Keypad.h> // Biblioteca para trabalhar com matriz de botões
#include <ESP32Encoder.h> // Biblioteca para trabalhar com encoders rotativos

#define numOfButtons 18 // Define o número de botões como 18 em vez do padrão (16)

// Configurações do gamepad
const char* gamepadName = "Aro Audi"; // Nome do dispositivo Bluetooth
const char* gamepadManufacturer = "Dangeromega"; // Nome do fabricante
const uint8_t gamepadBatteryLevel = 100; // Nível da bateria reportado ao host

// Configurações da matriz de botões
const uint8_t ROWS = 6; // Número de linhas na matriz de botões
const uint8_t COLS = 2; // Número de colunas na matriz de botões

// Pinos das linhas e colunas da matriz de botões
uint8_t rowPins[ROWS] = {33, 32, 27, 26, 25, 13};
uint8_t colPins[COLS] = {22, 23};

// Mapeamento dos botões na matriz
uint8_t keymap[ROWS][COLS] = {
  {1, 2}, {3, 4}, {5, 6}, {17, 18}, {9, 10}, {11, 12}
};

// Pinos para botões independentes (fora da matriz)
const uint8_t BUTTON_13_OUTPUT_PIN = 4; // Pino de saída para botão 13
const uint8_t BUTTON_13_INPUT_PIN = 5;  // Pino de entrada para botão 13
const uint8_t BUTTON_14_OUTPUT_PIN = 14; // Pino de saída para botão 14
const uint8_t BUTTON_14_INPUT_PIN = 15;  // Pino de entrada para botão 14

// Configuração dos encoders rotativos
const uint8_t MAXENC = 2; // Quantidade de encoders utilizados

// Pinos de conexão dos encoders
uint8_t encoderUppPin[MAXENC] = {18, 19}; // Pinos para direção de incremento
uint8_t encoderDwnPin[MAXENC] = {17, 21}; // Pinos para direção de decremento

// Botões virtuais atribuídos aos encoders
uint8_t encoderUppButton[MAXENC] = {15, 7}; // Botões pressionados quando o encoder gira para cima
uint8_t encoderDwnButton[MAXENC] = {16, 8}; // Botões pressionados quando o encoder gira para baixo

ESP32Encoder encoder[MAXENC]; // Array para os encoders

unsigned long holdoff[MAXENC] = {0, 0}; // Tempo para evitar leituras repetidas rápidas
int32_t prevCount[MAXENC] = {0, 0}; // Contador para armazenar estado anterior do encoder
const unsigned long HOLDOFFTIME = 150; // Tempo mínimo entre leituras do encoder (ms)

// Declaração das instâncias
Keypad customKeypad = Keypad(makeKeymap(keymap), rowPins, colPins, ROWS, COLS); // Instância do teclado matricial
BleGamepad bleGamepad(gamepadName, gamepadManufacturer, gamepadBatteryLevel); // Instância do gamepad Bluetooth

void setup() {
  Serial.begin(115200); // Inicializa a comunicação serial
  setCpuFrequencyMhz(80); // Define a frequência da CPU para 80 MHz (reduz consumo de energia)

  // Configuração dos pinos das linhas da matriz como saída e inicialização em nível alto
  for (uint8_t i = 0; i < ROWS; i++) {
    pinMode(rowPins[i], OUTPUT);
    digitalWrite(rowPins[i], HIGH);
  }

  // Configuração dos pinos das colunas da matriz como entrada com pull-up
  for (uint8_t i = 0; i < COLS; i++) {
    pinMode(colPins[i], INPUT_PULLUP);
  }

  // Configuração dos botões independentes
  pinMode(BUTTON_13_OUTPUT_PIN, OUTPUT);
  digitalWrite(BUTTON_13_OUTPUT_PIN, LOW);
  pinMode(BUTTON_13_INPUT_PIN, INPUT_PULLUP);

  pinMode(BUTTON_14_OUTPUT_PIN, OUTPUT);
  digitalWrite(BUTTON_14_OUTPUT_PIN, LOW);
  pinMode(BUTTON_14_INPUT_PIN, INPUT_PULLUP);

  // Configuração dos encoders
  for (uint8_t i = 0; i < MAXENC; i++) {
    encoder[i].attachSingleEdge(encoderDwnPin[i], encoderUppPin[i]); // Inicializa encoder com detecção de borda única
    encoder[i].clearCount(); // Zera o contador do encoder
  }

  // Configuração do gamepad Bluetooth
  BleGamepadConfiguration bleGamepadConfig;
  bleGamepadConfig.setButtonCount(numOfButtons); // Define a quantidade de botões
  bleGamepadConfig.setAutoReport(false); // Desativa envio automático de relatórios de estado
  
  // Registra evento do teclado matricial
  customKeypad.addEventListener(keypadEvent);
  customKeypad.setHoldTime(1); // Define tempo de retenção do botão (ms)
  customKeypad.setDebounceTime(50); // Define tempo de debounce (ms)

  // Inicializa o gamepad Bluetooth com a configuração ajustada
  bleGamepad.begin(&bleGamepadConfig);
  Serial.println("BLE Gamepad iniciado!"); // Mensagem de confirmação no Serial Monitor
}

void loop() {
  if (bleGamepad.isConnected()) { // Verifica se o gamepad está conectado via Bluetooth
    customKeypad.getKey(); // Captura os eventos do teclado matricial

    // Verifica o estado dos botões independentes
    bool button13Pressed = digitalRead(BUTTON_13_INPUT_PIN) == HIGH;
    bool button14Pressed = digitalRead(BUTTON_14_INPUT_PIN) == HIGH;

    // Atualiza o estado do botão 13 no gamepad Bluetooth
    if (button13Pressed) {
      bleGamepad.press(13);
    } else {
      bleGamepad.release(13);
    }

    // Atualiza o estado do botão 14 no gamepad Bluetooth
    if (button14Pressed) {
      bleGamepad.press(14);
    } else {
      bleGamepad.release(14);
    }

    unsigned long now = millis(); // Obtém o tempo atual
    for (uint8_t i = 0; i < MAXENC; i++) { // Itera sobre os encoders
      int32_t count = encoder[i].getCount(); // Obtém a contagem do encoder

      if (count != prevCount[i]) { // Se houve mudança no encoder
        if (!holdoff[i]) { // Se o tempo de espera foi atingido
          if (count > prevCount[i]) { // Se girou para cima
            bleGamepad.press(encoderUppButton[i]); // Pressiona o botão correspondente
            delay(50);
            bleGamepad.release(encoderUppButton[i]); // Solta o botão
          } else if (count < prevCount[i]) { // Se girou para baixo
            bleGamepad.press(encoderDwnButton[i]); // Pressiona o botão correspondente
            delay(50);
            bleGamepad.release(encoderDwnButton[i]); // Solta o botão
          }
          holdoff[i] = now; // Define o tempo de espera
        } else if (now - holdoff[i] > HOLDOFFTIME) { // Se passou o tempo mínimo
          prevCount[i] = count; // Atualiza o contador anterior
          holdoff[i] = 0; // Reseta o holdoff
        }
      }
    }

    bleGamepad.sendReport(); // Envia o estado atualizado do gamepad via Bluetooth
    delay(100); // Aguarda um curto período para evitar uso excessivo da CPU
  }
}

// Função chamada quando um botão do teclado matricial é pressionado ou solto
void keypadEvent(KeypadEvent key) {
  uint8_t keystate = customKeypad.getState(); // Obtém o estado atual do botão
  if (keystate == PRESSED) { // Se foi pressionado
    bleGamepad.press(key); // Pressiona o botão correspondente no gamepad Bluetooth
  }
  if (keystate == RELEASED) { // Se foi solto
    bleGamepad.release(key); // Solta o botão correspondente no gamepad Bluetooth
  }
}
