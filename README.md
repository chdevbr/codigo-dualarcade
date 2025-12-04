# 🕹️ DualArcade - Arduino OLED Game Console

Um console de jogos retrô desenvolvido em C++ para Arduino, utilizando um display OLED I2C. O sistema possui um menu de seleção fluido, sistema de recordes (high score) e dois jogos completos inspirados em clássicos.

## 🎮 Jogos Incluídos

1.  **Witch Hat (Flappy Style):** Controle o chapéu da bruxa voando através de colunas móveis. A velocidade aumenta conforme a pontuação.
2.  **Darwin Runner (Chrome Style):** O clássico jogo de corrida infinita. Pule cactos e sobreviva enquanto a velocidade aumenta.

## 🛠️ Hardware Necessário

* Arduino Uno, Nano ou compatível.
* Display OLED 0.96" I2C (Driver SSD1306).
* Joystick Analógico (ou botões simples).
* 1x Push Button (Botão de Ação/Pulo).
* 1x LED (Efeitos visuais).
* 1x Buzzer Passivo (Efeitos sonoros).
* Resistores e Jumpers.

## 🔌 Pinagem (Wiring)

As conexões estão definidas no início do código (`#define`), mas podem ser alteradas conforme sua necessidade:

| Componente | Pino Arduino | Descrição |
| :--- | :--- | :--- |
| **Display SDA** | A4 (Uno) | I2C Data |
| **Display SCL** | A5 (Uno) | I2C Clock |
| **Joystick Y** | A0 | Navegação Menu (Cima/Baixo) |
| **Joystick X** | A2 | Navegação Menu (Opcional) |
| **Botão Ação** | D10 | Pulo / Confirmar (Pull-up interno ativado) |
| **Buzzer** | D3 | Saída de Som (PWM) |
| **LED 1** | D4 | Efeito de colisão (Crash) |
| **LED 2** | D5 | Feedback de pontuação |
| **LED 3** | D6 | Efeito de colisão (Crash) |

## ⚙️ Dependências e Instalação

Este projeto utiliza a biblioteca gráfica **U8g2** para máxima performance no display OLED.

1.  Abra a Arduino IDE.
2.  Vá em **Sketch** > **Incluir Biblioteca** > **Gerenciar Bibliotecas**.
3.  Busque por `U8g2` (por oliver) e instale a versão mais recente.
4.  Conecte seu Arduino.
5.  Selecione a placa e a porta corretas.
6.  Carregue o arquivo `DualArcade.ino`.

## 💾 Detalhes Técnicos

* **Linguagem:** C++ (Arduino Framework).
* **Engine Gráfica:** U8g2 (Full Buffer) para garantir ~100 FPS teóricos e animações sem *flicker*.
* **Gerenciamento de Memória:** Sprites armazenados em `PROGMEM` para economizar RAM.
* **Input Handling:** Leitura de botões com algoritmo de *debounce* e *input buffering* para garantir que os pulos sejam precisos, mesmo se o botão for apertado frações de segundo antes do personagem tocar o chão.

## 🚀 Como Jogar

1.  Use o **Joystick** para selecionar o jogo no menu inicial.
2.  Pressione o **Botão de Ação** para entrar no jogo.
3.  **Witch Hat:** Pressione o botão para voar. Evite o chão, o teto e as colunas.
4.  **Darwin Runner:** Pressione o botão para pular os cactos.
5.  Se bater, o jogo mostra o "Game Over" e o recorde atual. Pressione o botão para voltar ao menu.

---

*Desenvolvido para fins educacionais.*
