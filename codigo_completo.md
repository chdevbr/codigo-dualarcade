## 💻 Código Fonte

Abaixo está o código completo do projeto. Toda a lógica de renderização, física e controle de estado está contida aqui.

<details>
  <summary><b>clique aqui para ver o código completo</b></summary>

```cpp
#include <Arduino.h>
#include <U8g2lib.h>
#include <Wire.h>

U8G2_SSD1306_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);

#define PINO_JOYSTICK_Y   A0
#define PINO_JOYSTICK_X   A2
#define PINO_BOTAO_ACAO   10

#define BUZZER       3
#define LED1         4
#define LED2         5
#define LED3         6

#define MENU_SELECAO 0
#define JOGO_BRUXA   1
#define JOGO_DINO    2
#define FIM_DE_JOGO  3

uint8_t estadoSistema = MENU_SELECAO;
uint8_t jogoSelecionado = 0;
uint8_t pontuacao = 0;
uint8_t recordeBruxa = 0;
uint8_t recordeDino = 0;

bool ultimoEstadoBotao = HIGH;
bool acaoClique = false;
bool novoRecorde = false;
bool pontuouNesteObstaculo = false;

bool puloSolicitado = false;

unsigned long tempoUltimoQuadro = 0;
const uint8_t intervaloQuadro = 10;

#define CHAPEU_L 16
#define CHAPEU_A 14
const unsigned char spriteChapeu[] PROGMEM = {
  0x00,0x00,0x00,0x00,0xC0,0x01,0xE0,0x03,
  0xF0,0x07,0xF8,0x0F,0xFC,0x1F,0xFC,0x1F,
  0x78,0x0F,0x38,0x0E,0x1C,0x1C,0xFE,0x3F,
  0xFF,0x7F,0xFF,0x7F
};

#define DINO_L 14
#define DINO_A 14
const unsigned char spriteDino[] PROGMEM = {
  0x00,0x00,0x00,0x00,0xE0,0x01,0xF0,0x03,
  0xF8,0x07,0xF8,0x07,0xFC,0x0F,0xFC,0x0F,
  0xF8,0x07,0x70,0x00,0x50,0x00,0x50,0x00,
  0x88,0x00,0x88,0x00
};

#define CACTO_L 8
#define CACTO_A 12
const unsigned char spriteCacto[] PROGMEM = {
  0x18,0x18,0x18,0x5A,0x5A,0xFF,0xFF,0x18,
  0x18,0x18,0x18,0x3C
};

uint8_t velocidadeGlobalBruxa = 4;
#define LARGURA_COLUNA 10
#define ESPACO_ENTRE_COLUNAS 28
#define POSICAO_X_CHAPEU 24
#define GRAVIDADE_BRUXA 1
#define FORCA_PULO_BRUXA -5

int8_t bruxaY;
int8_t bruxaVelocidadeY;
int colunaX;
uint8_t colunaBrechaY;

#define POSICAO_X_DINO 10
#define CHAO_DINO 48

int8_t dinoY;
int8_t dinoVelocidadeY;
int8_t cactoX;
uint8_t velocidadeGlobalDino = 6;

void tocarSom(int freq, int duracao) { 
    tone(BUZZER, freq, duracao); 
}

void apagarLeds() { 
    digitalWrite(LED1, LOW); 
    digitalWrite(LED2, LOW); 
    digitalWrite(LED3, LOW); 
}

void piscarLedsColisao() { 
    digitalWrite(LED1,HIGH); 
    digitalWrite(LED2,HIGH); 
    digitalWrite(LED3,HIGH); 
    delay(300); 
    apagarLeds(); 
}

bool lerBotaoComFiltro(uint8_t amostras = 6, unsigned int esperaMicros = 250) {
  for (uint8_t i = 0; i < amostras; ++i) {
    if (digitalRead(PINO_BOTAO_ACAO) == LOW) return true;
    delayMicroseconds(esperaMicros);
  }
  return false;
}

void desenharMenu() {
  u8g2.clearBuffer();
  u8g2.setFont(u8g2_font_ncenB10_tr);
  u8g2.setCursor(25, 15); u8g2.print(F("DualArcade"));

  u8g2.setFont(u8g2_font_6x10_tf);

  if (jogoSelecionado == 0) {
    u8g2.drawFrame(0, 25, 128, 18);
    u8g2.setCursor(5, 37); u8g2.print(F("> WITCH HAT"));
    u8g2.drawXBMP(100, 27, CHAPEU_L, CHAPEU_A, spriteChapeu);
  } else {
    u8g2.drawStr(5, 37, "  WITCH HAT");
  }

  if (jogoSelecionado == 1) {
    u8g2.drawFrame(0, 45, 128, 18);
    u8g2.setCursor(5, 57); u8g2.print(F("> DINO RUNNER"));
    u8g2.drawXBMP(100, 49, DINO_L, DINO_A, spriteDino);
  } else {
    u8g2.drawStr(5, 57, "  DINO RUNNER");
  }

  u8g2.sendBuffer();
}

void desenharGameOver() {
  u8g2.clearBuffer();

  u8g2.setFont(u8g2_font_ncenB10_tr);
  u8g2.drawBox(10, 20, 108, 30);
  u8g2.setDrawColor(0);
  u8g2.drawBox(12, 22, 104, 26);
  u8g2.setDrawColor(1);
  u8g2.setCursor(18, 40); u8g2.print(F("GAME OVER"));

  if (novoRecorde) {
    u8g2.setFont(u8g2_font_6x10_tf);
    u8g2.setCursor(30, 15); u8g2.print(F("NOVO RECORD!"));
  }

  u8g2.sendBuffer();
}

void iniciarBruxa() {
  bruxaY = 15;
  bruxaVelocidadeY = 0;
  colunaX = 128;
  colunaBrechaY = random(10, 64 - ESPACO_ENTRE_COLUNAS - 10);
  
  pontuacao = 0; 
  pontuouNesteObstaculo = false; 
  velocidadeGlobalBruxa = 4; 
  novoRecorde = false;
  
  apagarLeds(); 
  tocarSom(600, 100);
}

void atualizarBruxa(bool botaoPressionado) {
  if (botaoPressionado) { 
    bruxaVelocidadeY = FORCA_PULO_BRUXA; 
    tocarSom(800, 20); 
  }

  if (pontuacao >= 8) velocidadeGlobalBruxa = 7;
  else if (pontuacao >= 3) velocidadeGlobalBruxa = 5;
  else velocidadeGlobalBruxa = 4;

  bruxaVelocidadeY += GRAVIDADE_BRUXA;
  bruxaY += bruxaVelocidadeY;
  colunaX -= velocidadeGlobalBruxa;

  if (colunaX < -LARGURA_COLUNA) {
    colunaX = 128;
    colunaBrechaY = random(10, 64 - ESPACO_ENTRE_COLUNAS - 10);
    pontuouNesteObstaculo = false;
  }

  if (!pontuouNesteObstaculo && colunaX + LARGURA_COLUNA < POSICAO_X_CHAPEU) {
    pontuacao++; 
    pontuouNesteObstaculo = true;
    digitalWrite(LED2, HIGH);
  } else {
    digitalWrite(LED2, LOW);
  }

  bool colidiu = false;

  if (bruxaY >= 63 || bruxaY <= 0) colidiu = true;

  if (colunaX < POSICAO_X_CHAPEU + 13 && colunaX + LARGURA_COLUNA > POSICAO_X_CHAPEU + 3) {
    if (bruxaY + 4 < colunaBrechaY || bruxaY + CHAPEU_A - 2 > colunaBrechaY + ESPACO_ENTRE_COLUNAS)
      colidiu = true;
  }

  if (colidiu) {
    estadoSistema = FIM_DE_JOGO;
    piscarLedsColisao();
    tocarSom(150, 300);
    if (pontuacao > recordeBruxa) { 
        recordeBruxa = pontuacao; 
        novoRecorde = true; 
    }
  }
}

void desenharBruxa() {
  u8g2.clearBuffer();

  u8g2.drawXBMP(POSICAO_X_CHAPEU, bruxaY, CHAPEU_L, CHAPEU_A, spriteChapeu);
  
  u8g2.drawBox(colunaX, 0, LARGURA_COLUNA, colunaBrechaY);
  u8g2.drawBox(colunaX, colunaBrechaY + ESPACO_ENTRE_COLUNAS, LARGURA_COLUNA, 64 - (colunaBrechaY + ESPACO_ENTRE_COLUNAS));
  
  u8g2.drawHLine(0, 63, 128);

  u8g2.setFont(u8g2_font_6x10_tf);
  u8g2.setCursor(0, 8); u8g2.print(F("Pontos ")); u8g2.print(pontuacao);
  u8g2.setCursor(80, 8); u8g2.print(F("Recorde ")); u8g2.print(recordeBruxa);

  u8g2.sendBuffer();
}

void iniciarDino() {
  dinoY = CHAO_DINO;
  dinoVelocidadeY = 0;
  cactoX = 110;
  
  pontuacao = 0; 
  pontuouNesteObstaculo = false; 
  velocidadeGlobalDino = 6; 
  novoRecorde = false;
  puloSolicitado = false;
  
  apagarLeds(); 
  tocarSom(600, 100);
}

void atualizarDino(bool bordaBotao) {
  if (bordaBotao) {
    puloSolicitado = true;
  }

  if (puloSolicitado && dinoY >= CHAO_DINO) {
    dinoVelocidadeY = -5;
    tocarSom(200, 30);
    puloSolicitado = false;
  }

  dinoVelocidadeY += 1;
  dinoY += dinoVelocidadeY;

  if (dinoY > CHAO_DINO) { 
      dinoY = CHAO_DINO; 
      dinoVelocidadeY = 0; 
  }

  if (pontuacao >= 12) velocidadeGlobalDino = 9;
  else if (pontuacao >= 4) velocidadeGlobalDino = 7;
  else velocidadeGlobalDino = 6;

  cactoX -= velocidadeGlobalDino;

  if (cactoX < -10) {
    cactoX = 110;
    pontuouNesteObstaculo = false;
  }

  if (!pontuouNesteObstaculo && cactoX < POSICAO_X_DINO) {
    pontuacao++;
    pontuouNesteObstaculo = true;
    digitalWrite(LED2, HIGH);
  } else {
    digitalWrite(LED2, LOW);
  }

  if (cactoX < POSICAO_X_DINO + DINO_L - 8 && cactoX + CACTO_L > POSICAO_X_DINO + 8) {
    if (dinoY + DINO_A > CHAO_DINO + DINO_A - CACTO_A + 8) {
      estadoSistema = FIM_DE_JOGO;
      piscarLedsColisao();
      tocarSom(100, 400);
    }
  }

  if (estadoSistema == FIM_DE_JOGO && pontuacao > recordeDino) {
    recordeDino = pontuacao; 
    novoRecorde = true;
  }
}

void desenharDino() {
  u8g2.clearBuffer();

  u8g2.drawXBMP(POSICAO_X_DINO, dinoY, DINO_L, DINO_A, spriteDino);
  
  int8_t cactoY = (CHAO_DINO + DINO_A) - CACTO_A;
  u8g2.drawXBMP(cactoX, cactoY, CACTO_L, CACTO_A, spriteCacto);
  
  u8g2.drawHLine(0, CHAO_DINO + DINO_A, 128);

  u8g2.setFont(u8g2_font_6x10_tf);
  u8g2.setCursor(0, 10); u8g2.print(F("Pontos ")); u8g2.print(pontuacao);
  u8g2.setCursor(80, 10); u8g2.print(F("Recorde ")); u8g2.print(recordeDino);

  u8g2.sendBuffer();
}

void setup() {
  u8g2.begin();
  Wire.begin(); Wire.setClock(400000);
  pinMode(PINO_BOTAO_ACAO, INPUT_PULLUP);
  pinMode(BUZZER, OUTPUT);
  pinMode(LED1, OUTPUT); pinMode(LED2, OUTPUT); pinMode(LED3, OUTPUT);
  randomSeed(analogRead(0));
}

void loop() {
  unsigned long tempoAtual = millis();
  if (tempoAtual - tempoUltimoQuadro < intervaloQuadro) return;
  tempoUltimoQuadro = tempoAtual;

  bool leituraAtual = lerBotaoComFiltro();

  if (leituraAtual && !ultimoEstadoBotao)
      acaoClique = true;
  else
      acaoClique = false;

  ultimoEstadoBotao = leituraAtual;

  static uint8_t ultimoJogoSelecionado = 255;

  switch (estadoSistema) {
    case MENU_SELECAO: {
        int joyY = analogRead(PINO_JOYSTICK_Y);
        int joyX = analogRead(PINO_JOYSTICK_X);

        if (joyY > 800 || joyX > 800) jogoSelecionado = 1;
        else if (joyY < 200 || joyX < 200) jogoSelecionado = 0;

        if (jogoSelecionado != ultimoJogoSelecionado) {
          desenharMenu();
          ultimoJogoSelecionado = jogoSelecionado;
        }

        if (acaoClique) {
          tocarSom(1000, 50);
          ultimoJogoSelecionado = 255;

          if (jogoSelecionado == 0) { 
              iniciarBruxa(); 
              estadoSistema = JOGO_BRUXA; 
          } else { 
              iniciarDino(); 
              estadoSistema = JOGO_DINO; 
          }
        }
      }
      break;

    case JOGO_BRUXA:
      atualizarBruxa(acaoClique);
      desenharBruxa();
      break;

    case JOGO_DINO:
      atualizarDino(acaoClique);
      desenharDino();
      break;

    case FIM_DE_JOGO:
      desenharGameOver();

      if (acaoClique) {
        estadoSistema = MENU_SELECAO;
        ultimoJogoSelecionado = 255;
      }
      break;
  }
}
