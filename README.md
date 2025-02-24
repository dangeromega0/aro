#include <BleGamepad.h>      // Biblioteca para gamepad BLE
#include <ESP32Encoder.h>    // Biblioteca para encoders no ESP32

BleGamepad bleGamepad("ESP32 Gamepad", "xAI", 100);

// Instâncias dos encoders
ESP32Encoder encoder1;
ESP32Encoder encoder2;

// Estrutura para os botões
struct Botao {
  uint8_t pinoRow;
  uint8_t pinoCol;
  uint8_t id;
};

// Definição dos botões
Botao botoes[] = {
  {33, 22, 1}, {33, 23, 2}, {32, 22, 3}, {32, 23, 4},
  {27, 22, 5}, {27, 23, 6}, {26, 22, 7}, {26, 23, 8},
  {25, 22, 9}, {25, 23, 10}, {13, 22, 11}, {13, 23, 12},
  {4, 22, 13}, {4, 23, 14}
};

const uint8_t NUM_BOTOES = sizeof(botoes) / sizeof(botoes[0]);
const uint8_t pinosRows[] = {33, 32, 27, 26, 25, 13, 4};
const uint8_t pinosCols[] = {22, 23};
const uint8_t NUM_ROWS = sizeof(pinosRows) / sizeof(pinosRows[0]);
const uint8_t NUM_COLS = sizeof(pinosCols) / sizeof(pinosCols[0]);

bool estadosBotoes[NUM_BOTOES] = {false};
long ultimoEncoder1Pos = 0;
long ultimoEncoder2Pos = 0;

void setup() {
  Serial.begin(115200);
  delay(1000);

  // Configuração dos pinos da matriz
  for (uint8_t i = 0; i < NUM_ROWS; i++) {
    pinMode(pinosRows[i], OUTPUT);
    digitalWrite(pinosRows[i], HIGH);
  }
  for (uint8_t i = 0; i < NUM_COLS; i++) {
    pinMode(pinosCols[i], INPUT_PULLUP);
  }

  // Configuração dos encoders
  pinMode(18, INPUT_PULLUP);
  pinMode(17, INPUT_PULLUP);
  pinMode(21, INPUT_PULLUP);
  pinMode(19, INPUT_PULLUP);
  encoder1.attachFullQuad(18, 17);
  encoder2.attachFullQuad(21, 19);
  encoder1.clearCount();
  encoder2.clearCount();

  bleGamepad.begin();
  Serial.println("BLE Gamepad iniciado!");
}

void loop() {
  if (bleGamepad.isConnected()) {
    // Varredura dos botões
    for (uint8_t r = 0; r < NUM_ROWS; r++) {
      digitalWrite(pinosRows[r], LOW);
      delayMicroseconds(10);

      for (uint8_t c = 0; c < NUM_COLS; c++) {
        bool pressionado = (digitalRead(pinosCols[c]) == LOW);
        
        // Debug bruto da leitura
        if (pressionado) {
          Serial.printf("Detectado - Linha: %d, Coluna: %d\n", pinosRows[r], pinosCols[c]);
        }

        for (uint8_t i = 0; i < NUM_BOTOES; i++) {
          if (botoes[i].pinoRow == pinosRows[r] && botoes[i].pinoCol == pinosCols[c]) {
            bool leitura = pressionado;
            if (botoes[i].id == 13 || botoes[i].id == 14) {
              leitura = pressionado;  // Removida inversão dos paddles
            }

            if (leitura != estadosBotoes[i]) {
              estadosBotoes[i] = leitura;
              if (leitura) {
                bleGamepad.press(botoes[i].id);
                Serial.printf("Botão %d pressionado (Linha: %d, Coluna: %d)\n", 
                             botoes[i].id, pinosRows[r], pinosCols[c]);
              } else {
                bleGamepad.release(botoes[i].id);
                Serial.printf("Botão %d solto (Linha: %d, Coluna: %d)\n", 
                             botoes[i].id, pinosRows[r], pinosCols[c]);
              }
            }
          }
        }
      }
      digitalWrite(pinosRows[r], HIGH);
    }

    // Leitura dos encoders como botões
    long pos1 = encoder1.getCount();
    long pos2 = encoder2.getCount();
    if (pos1 != ultimoEncoder1Pos) {
      if (pos1 > ultimoEncoder1Pos) bleGamepad.press(15);
      else bleGamepad.press(16);
      delay(50);
      bleGamepad.release(15);
      bleGamepad.release(16);
      ultimoEncoder1Pos = pos1;
    }
    if (pos2 != ultimoEncoder2Pos) {
      if (pos2 > ultimoEncoder2Pos) bleGamepad.press(17);
      else bleGamepad.press(18);
      delay(50);
      bleGamepad.release(17);
      bleGamepad.release(18);
      ultimoEncoder2Pos = pos2;
    }

    delay(20);
  }
}
