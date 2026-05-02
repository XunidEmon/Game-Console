# 🎮 Gaming Console Project

A simple DIY Gaming Console built using Arduino / ESP32 (or your hardware). This project is designed for learning embedded systems, game logic, and hardware interaction.

---

## 🚀 Features
- Simple retro-style games (e.g., Snake, Tetris, Pong)
- Button-based control system
- OLED / LCD display support
- Lightweight and fast execution
- Beginner-friendly embedded game system

---

## 🛠️ Hardware Requirements
- Arduino UNO / ESP32 / similar microcontroller  
- 16x2 LCD or OLED Display (SSD1306 recommended)  
- Push Buttons / Joystick  
- Buzzer (optional for sound effects)  
- Breadboard & jumper wires  
- Power supply (USB / battery)

---

## 💻 Software Requirements
- Arduino IDE  

### Required Libraries:
- Adafruit GFX  
- Adafruit SSD1306 (if OLED used)  
- LiquidCrystal (if LCD used)

---

## ⚙️ How It Works
- Microcontroller initializes display and input buttons  
- Game loop runs continuously  
- Player input is read from buttons/joystick  
- Game logic updates screen in real-time  
- Score and state are displayed on screen  

---


```cpp
# Game-Console
TIC TAC TOE
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define UP_BTN 2
#define DOWN_BTN 4
#define LEFT_BTN 3
#define RIGHT_BTN 5
#define SELECT_BTN 6

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

enum State {MENU, GAME, GAMEOVER};
State state = MENU;

char board[3][3];
int cursorX = 0;
int cursorY = 0;
bool playerXTurn = true;
bool vsAI = false;
char winner = ' ';
int menuIndex = 0;

unsigned long lastPress = 0;
const int debounceDelay = 250;
bool aiThinking = false;

// Game Over screen state
bool showWinnerScreen = false;
unsigned long gameOverTime = 0;

/* ================= SETUP ================= */

void setup() {
  Serial.begin(9600);
  
  pinMode(UP_BTN, INPUT_PULLUP);
  pinMode(DOWN_BTN, INPUT_PULLUP);
  pinMode(LEFT_BTN, INPUT_PULLUP);
  pinMode(RIGHT_BTN, INPUT_PULLUP);
  pinMode(SELECT_BTN, INPUT_PULLUP);

  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    while(1);
  }
  
  display.clearDisplay();
  display.display();
  
  randomSeed(analogRead(A0));
  resetBoard();
}

/* ================= LOOP ================= */

void loop() {
  if(state == MENU){
    drawMenu();
    handleMenu();
  }
  else if(state == GAME){
    runGame();
  }
  else if(state == GAMEOVER){
    // Show winner screen without board
    drawWinnerScreen();
    
    if(buttonPressed(SELECT_BTN)){
      state = MENU;
      menuIndex = 0;
      resetBoard();
      showWinnerScreen = false;
    }
  }
  delay(20);
}

/* ================= BUTTON FUNCTION ================= */

bool buttonPressed(int pin){
  if(digitalRead(pin) == LOW && millis() - lastPress > debounceDelay){
    lastPress = millis();
    return true;
  }
  return false;
}

/* ================= MENU ================= */

void handleMenu(){
  if(buttonPressed(UP_BTN)){
    menuIndex--;
    if(menuIndex < 0) menuIndex = 1;
    drawMenu();
  }

  if(buttonPressed(DOWN_BTN)){
    menuIndex++;
    if(menuIndex > 1) menuIndex = 0;
    drawMenu();
  }

  if(buttonPressed(SELECT_BTN)){
    vsAI = (menuIndex == 1);
    resetBoard();
    state = GAME;
  }
}

void drawMenu(){
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(WHITE);

  display.setCursor(25, 5);
  display.println("TIC TAC TOE");
  display.drawLine(0, 16, SCREEN_WIDTH, 16, WHITE);

  display.setCursor(20, 25);
  if(menuIndex == 0) display.print("> ");
  else display.print("  ");
  display.println("Two Player");

  display.setCursor(20, 40);
  if(menuIndex == 1) display.print("> ");
  else display.print("  ");
  display.println("Play With AI");

  display.display();
}

/* ================= GAME ================= */

void runGame(){
  // Check if game is already over
  if(winner != ' ') {
    state = GAMEOVER;
    showWinnerScreen = true;
    gameOverTime = millis();
    return;
  }
  
  // AI's turn
  if(vsAI && !playerXTurn && !aiThinking){
    aiThinking = true;
    delay(300);
    aiMove();
    aiThinking = false;
  }

  // Only allow player input when it's player's turn
  if((!vsAI) || (vsAI && playerXTurn)){
    if(buttonPressed(UP_BTN) && cursorY > 0){
      cursorY--;
      drawBoard();
    }
    if(buttonPressed(DOWN_BTN) && cursorY < 2){
      cursorY++;
      drawBoard();
    }
    if(buttonPressed(LEFT_BTN) && cursorX > 0){
      cursorX--;
      drawBoard();
    }
    if(buttonPressed(RIGHT_BTN) && cursorX < 2){
      cursorX++;
      drawBoard();
    }

    if(buttonPressed(SELECT_BTN)){
      if(board[cursorY][cursorX] == ' '){
        // Place the piece
        board[cursorY][cursorX] = playerXTurn ? 'X' : 'O';
        drawBoard();
        delay(200);
        
        // Check winner
        checkWinner();
        
        // Switch turns
        playerXTurn = !playerXTurn;
      }
    }
  }
  
  drawBoard();
}

/* ================= AI ================= */

void aiMove(){
  // Try to win
  if(findWinMove()) return;
  
  // Try to block
  if(findBlockMove()) return;
  
  // Try center
  if(board[1][1] == ' '){
    board[1][1] = 'O';
    checkWinner();
    playerXTurn = true;
    drawBoard();
    return;
  }
  
  // Try corners
  int corners[4][2] = {{0,0}, {0,2}, {2,0}, {2,2}};
  for(int i = 0; i < 4; i++){
    int y = corners[i][0];
    int x = corners[i][1];
    if(board[y][x] == ' '){
      board[y][x] = 'O';
      checkWinner();
      playerXTurn = true;
      drawBoard();
      return;
    }
  }
  
  // Random move
  int x, y;
  int attempts = 0;
  do{
    x = random(0, 3);
    y = random(0, 3);
    attempts++;
    if(attempts > 100) return;
  } while(board[y][x] != ' ');
  
  board[y][x] = 'O';
  checkWinner();
  playerXTurn = true;
  drawBoard();
}

bool findWinMove(){
  for(int y = 0; y < 3; y++){
    for(int x = 0; x < 3; x++){
      if(board[y][x] == ' '){
        board[y][x] = 'O';
        if(checkWin('O')){
          board[y][x] = ' ';
          board[y][x] = 'O';
          checkWinner();
          playerXTurn = true;
          drawBoard();
          return true;
        }
        board[y][x] = ' ';
      }
    }
  }
  return false;
}

bool findBlockMove(){
  for(int y = 0; y < 3; y++){
    for(int x = 0; x < 3; x++){
      if(board[y][x] == ' '){
        board[y][x] = 'X';
        if(checkWin('X')){
          board[y][x] = ' ';
          board[y][x] = 'O';
          checkWinner();
          playerXTurn = true;
          drawBoard();
          return true;
        }
        board[y][x] = ' ';
      }
    }
  }
  return false;
}

bool checkWin(char s){
  for(int i = 0; i < 3; i++){
    if(board[i][0] == s && board[i][1] == s && board[i][2] == s) return true;
  }
  for(int i = 0; i < 3; i++){
    if(board[0][i] == s && board[1][i] == s && board[2][i] == s) return true;
  }
  if(board[0][0] == s && board[1][1] == s && board[2][2] == s) return true;
  if(board[0][2] == s && board[1][1] == s && board[2][0] == s) return true;
  return false;
}

/* ================= BOARD ================= */

void resetBoard(){
  for(int y = 0; y < 3; y++)
    for(int x = 0; x < 3; x++)
      board[y][x] = ' ';

  cursorX = 0;
  cursorY = 0;
  playerXTurn = true;
  winner = ' ';
  aiThinking = false;
  showWinnerScreen = false;
}

void drawBoard(){
  display.clearDisplay();

  // Draw grid lines
  display.drawLine(42, 0, 42, 64, WHITE);
  display.drawLine(84, 0, 84, 64, WHITE);
  display.drawLine(0, 21, 128, 21, WHITE);
  display.drawLine(0, 42, 128, 42, WHITE);

  // Draw X and O
  display.setTextSize(2);
  display.setTextColor(WHITE);

  for(int y = 0; y < 3; y++){
    for(int x = 0; x < 3; x++){
      display.setCursor(15 + x * 42, 3 + y * 21);
      display.print(board[y][x]);
    }
  }

  // Draw cursor (only if game is not over and it's player's turn)
  if(state == GAME && (!vsAI || (vsAI && playerXTurn))){
    display.drawRect(cursorX * 42 + 2, cursorY * 21 + 2, 38, 17, WHITE);
  }

  // Draw turn indicator
  display.setTextSize(1);
  display.setCursor(0, 55);
  if(state == GAME){
    if(vsAI && !playerXTurn){
      display.print("AI thinking...");
    } else {
      display.print("Player ");
      display.print(playerXTurn ? 'X' : 'O');
      display.print("'s turn");
    }
  }

  display.display();
}

/* ================= WINNER CHECK ================= */

void checkWinner(){
  // Check rows
  for(int i = 0; i < 3; i++){
    if(board[i][0] != ' ' && board[i][0] == board[i][1] && board[i][1] == board[i][2]){
      winner = board[i][0];
      state = GAMEOVER;
      showWinnerScreen = true;
      return;
    }
  }

  // Check columns
  for(int i = 0; i < 3; i++){
    if(board[0][i] != ' ' && board[0][i] == board[1][i] && board[1][i] == board[2][i]){
      winner = board[0][i];
      state = GAMEOVER;
      showWinnerScreen = true;
      return;
    }
  }

  // Check diagonals
  if(board[0][0] != ' ' && board[0][0] == board[1][1] && board[1][1] == board[2][2]){
    winner = board[0][0];
    state = GAMEOVER;
    showWinnerScreen = true;
    return;
  }

  if(board[0][2] != ' ' && board[0][2] == board[1][1] && board[1][1] == board[2][0]){
    winner = board[0][2];
    state = GAMEOVER;
    showWinnerScreen = true;
    return;
  }

  // Check draw
  bool draw = true;
  for(int y = 0; y < 3; y++){
    for(int x = 0; x < 3; x++){
      if(board[y][x] == ' '){
        draw = false;
        break;
      }
    }
    if(!draw) break;
  }

  if(draw){
    winner = 'D';
    state = GAMEOVER;
    showWinnerScreen = true;
  }
}

/* ================= GAME OVER ================= */

void drawWinnerScreen(){
  display.clearDisplay();
  
  // Show winner message
  display.setTextSize(2);
  display.setTextColor(WHITE);
  
  if(winner == 'D'){
    display.setCursor(35, 15);
    display.println("DRAW!");
    display.setTextSize(1);
    display.setCursor(30, 45);
    display.println("Press SELECT");
  }
  else {
    display.setCursor(30, 10);
    display.println("WINNER!");
    
    display.setTextSize(4);
    display.setCursor(55, 30);
    display.print(winner);
    
    display.setTextSize(1);
    display.setCursor(30, 55);
    display.println("Press SELECT");
  }
  
  display.display();
}











GAME(4)
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

#define UP_BTN 2
#define DOWN_BTN 4
#define LEFT_BTN 3
#define RIGHT_BTN 5
#define SELECT_BTN 6

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

enum GameState {MENU, GAME1, GAME2, GAME3, GAME4, GAMEOVER};
GameState state = MENU;

int menuIndex = 0;
const int menuItems = 4;

// Game 1: Catch The Dot
int playerX1, playerY1;
int dotX1, dotY1;
int score1 = 0;
int highScore1 = 0;
int ballSpeed1 = 1;
unsigned long lastMove1 = 0;
unsigned long moveDelay1 = 15;

// Game 2: Dodge Blocks
int playerX2, playerY2;
int obstacleX, obstacleY;
int obstacleWidth = 20;
int obstacleHeight = 8;
int score2 = 0;
int highScore2 = 0;
int obstacleSpeed = 2;
unsigned long lastMove2 = 0;
unsigned long moveDelay2 = 15;

// Game 3: Space Shooter
int playerX3, playerY3;
int bulletX, bulletY;
bool bulletActive = false;
int enemyX[3], enemyY[3];
bool enemyActive[3];
int score3 = 0;
int highScore3 = 0;
int enemySpeed = 1;
unsigned long lastMove3 = 0;
unsigned long lastShootTime = 0;
unsigned long moveDelay3 = 15;
const int shootDelay = 300;

// Game 4: Memory Match
int sequence[5];
int sequenceLength = 1;
int currentStep = 0;
bool showingSequence = true;
unsigned long sequenceShowTime = 0;
int score4 = 0;
int highScore4 = 0;
int lives = 3;

int lastGameScore = 0;
int lastGameType = 0;
bool gameStarted = false;
bool waitingForButtonRelease = false;
const int buttonDebounceDelay = 200;

// Game 4 button debounce variable
unsigned long lastButtonCheck4 = 0;

void setup() {
  pinMode(UP_BTN, INPUT_PULLUP);
  pinMode(DOWN_BTN, INPUT_PULLUP);
  pinMode(LEFT_BTN, INPUT_PULLUP);
  pinMode(RIGHT_BTN, INPUT_PULLUP);
  pinMode(SELECT_BTN, INPUT_PULLUP);

  Serial.begin(9600);
  
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    while(1);
  }

  display.clearDisplay();
  display.display();
  
  randomSeed(analogRead(0) + millis());
  
  resetGame1();
  resetGame2();
  resetGame3();
  resetGame4();
}

void loop() {
  switch(state){
    case MENU:
      drawMenu();
      handleMenuInput();
      break;
    case GAME1:
      runGame1();
      break;
    case GAME2:
      runGame2();
      break;
    case GAME3:
      runGame3();
      break;
    case GAME4:
      runGame4();
      break;
    case GAMEOVER:
      drawGameOver();
      handleGameOverInput();
      break;
  }
  delay(30);
}

void drawMenu(){
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  
  display.setCursor(40, 0);
  display.println("UAP GAMES");
  display.drawLine(0, 10, SCREEN_WIDTH, 10, WHITE);
  
  for(int i=0;i<menuItems;i++){
    display.setCursor(10, 15 + i*12);
    
    if(i==menuIndex) {
      display.print("> ");
    } else {
      display.print("  ");
    }
    
    if(i==0) {
      display.print("Catch Dot");
    }
    if(i==1) {
      display.print("Dodge Blocks");
    }
    if(i==2) {
      display.print("Space Shooter");
    }
    if(i==3) {
      display.print("Memory Match");
    }
    
    display.setCursor(SCREEN_WIDTH - 25, 15 + i*12);
    if(i==0) display.print(highScore1);
    if(i==1) display.print(highScore2);
    if(i==2) display.print(highScore3);
    if(i==3) display.print(highScore4);
  }
  
  display.display();
}

void handleMenuInput(){
  static unsigned long lastPress = 0;
  
  if(digitalRead(UP_BTN)==LOW && millis() - lastPress > buttonDebounceDelay){
    menuIndex--;
    if(menuIndex<0) menuIndex=menuItems-1;
    lastPress = millis();
    drawMenu();
  }
  
  if(digitalRead(DOWN_BTN)==LOW && millis() - lastPress > buttonDebounceDelay){
    menuIndex++;
    if(menuIndex>=menuItems) menuIndex=0;
    lastPress = millis();
    drawMenu();
  }
  
  if(digitalRead(SELECT_BTN)==LOW && millis() - lastPress > buttonDebounceDelay){
    lastPress = millis();
    gameStarted = false;
    waitingForButtonRelease = false;
    
    delay(100);
    
    if(menuIndex==0){
      resetGame1();
      state=GAME1;
      lastGameType = 1;
    } else if(menuIndex==1){
      resetGame2();
      state=GAME2;
      lastGameType = 2;
    } else if(menuIndex==2){
      resetGame3();
      state=GAME3;
      lastGameType = 3;
    } else {
      resetGame4();
      state=GAME4;
      lastGameType = 4;
    }
  }
}

// ================== GAME 1: CATCH THE DOT ==================
void resetGame1(){
  playerX1 = SCREEN_WIDTH/2 - 10;
  playerY1 = SCREEN_HEIGHT - 8;
  dotX1 = random(5, SCREEN_WIDTH-5);
  dotY1 = 5;
  score1 = 0;
  ballSpeed1 = 1;
  gameStarted = false;
}

void runGame1(){
  if(!gameStarted) {
    gameStarted = true;
    delay(50);
    return;
  }
  
  if(millis()-lastMove1 > moveDelay1){
    if(digitalRead(LEFT_BTN)==LOW && playerX1 > 0) {
      playerX1-=4;
    }
    if(digitalRead(RIGHT_BTN)==LOW && playerX1 < SCREEN_WIDTH-20) {
      playerX1+=4;
    }
    lastMove1 = millis();
  }
  
  dotY1 += ballSpeed1;
  
  if(dotY1 >= playerY1-2 && dotY1 <= playerY1+2){
    if(dotX1 >= playerX1 && dotX1 <= playerX1+20){
      score1++;
      dotY1 = 5;
      dotX1 = random(5, SCREEN_WIDTH-5);
      
      if(score1 % 5 == 0) {
        ballSpeed1++;
        moveDelay1 = max(5, moveDelay1 - 1);
      }
    }
  }
  
  if(dotY1 > SCREEN_HEIGHT) {
    lastGameScore = score1;
    if(score1 > highScore1) highScore1 = score1;
    waitingForButtonRelease = true;
    state = GAMEOVER;
    return;
  }
  
  display.clearDisplay();
  display.fillRect(playerX1, playerY1, 20, 4, WHITE);
  display.fillCircle(dotX1, dotY1, 3, WHITE);
  display.setTextSize(1);
  display.setCursor(0,0);
  display.print("Score: ");
  display.print(score1);
  display.display();
  
  if(digitalRead(SELECT_BTN)==LOW && millis() - lastMove1 > buttonDebounceDelay){
    state=MENU;
  }
}

// ================== GAME 2: DODGE BLOCKS ==================
void resetGame2(){
  playerX2 = SCREEN_WIDTH/2 - 10;
  playerY2 = SCREEN_HEIGHT - 8;
  obstacleX = random(0, SCREEN_WIDTH - obstacleWidth);
  obstacleY = -obstacleHeight;
  score2 = 0;
  obstacleSpeed = 2;
  gameStarted = false;
}

void runGame2(){
  if(!gameStarted) {
    gameStarted = true;
    delay(50);
    return;
  }
  
  if(millis()-lastMove2 > moveDelay2){
    if(digitalRead(LEFT_BTN)==LOW && playerX2 > 0) {
      playerX2-=6;
    }
    if(digitalRead(RIGHT_BTN)==LOW && playerX2 < SCREEN_WIDTH-20) {
      playerX2+=6;
    }
    lastMove2 = millis();
  }
  
  obstacleY += obstacleSpeed;
  
  if(obstacleY > SCREEN_HEIGHT) {
    score2++;
    obstacleY = -obstacleHeight;
    obstacleX = random(0, SCREEN_WIDTH - obstacleWidth);
    
    if(score2 % 5 == 0) {
      obstacleSpeed++;
      if(moveDelay2 > 5) moveDelay2--;
    }
  }
  
  bool collision = false;
  if (obstacleY + obstacleHeight >= playerY2 && obstacleY <= playerY2 + 4) {
    if (obstacleX + obstacleWidth >= playerX2 && obstacleX <= playerX2 + 20) {
      collision = true;
    }
  }
  
  if (collision) {
    lastGameScore = score2;
    if(score2 > highScore2) highScore2 = score2;
    waitingForButtonRelease = true;
    state = GAMEOVER;
    return;
  }
  
  display.clearDisplay();
  display.fillRect(playerX2, playerY2, 20, 4, WHITE);
  display.fillRect(obstacleX, obstacleY, obstacleWidth, obstacleHeight, WHITE);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("Score: ");
  display.print(score2);
  display.display();
  
  if(digitalRead(SELECT_BTN)==LOW && millis() - lastMove2 > buttonDebounceDelay){
    state=MENU;
  }
}

// ================== GAME 3: SPACE SHOOTER ==================
void resetGame3(){
  playerX3 = SCREEN_WIDTH/2 - 4;
  playerY3 = SCREEN_HEIGHT - 10;
  bulletActive = false;
  score3 = 0;
  enemySpeed = 1;
  gameStarted = false;
  
  for(int i = 0; i < 3; i++){
    enemyActive[i] = true;
    enemyX[i] = random(0, SCREEN_WIDTH - 8);
    enemyY[i] = random(-30, -5);
  }
}

void runGame3(){
  if(!gameStarted) {
    gameStarted = true;
    delay(50);
    return;
  }
  
  if(millis()-lastMove3 > moveDelay3){
    if(digitalRead(LEFT_BTN)==LOW && playerX3 > 0) {
      playerX3-=3;
    }
    if(digitalRead(RIGHT_BTN)==LOW && playerX3 < SCREEN_WIDTH-8) {
      playerX3+=3;
    }
    
    if(digitalRead(UP_BTN)==LOW && millis() - lastShootTime > shootDelay && !bulletActive){
      bulletActive = true;
      bulletX = playerX3 + 3;
      bulletY = playerY3 - 4;
      lastShootTime = millis();
    }
    
    lastMove3 = millis();
  }
  
  if(bulletActive){
    bulletY -= 4;
    
    if(bulletY < 0){
      bulletActive = false;
    }
    
    for(int i = 0; i < 3; i++){
      if(enemyActive[i]){
        if(bulletX >= enemyX[i] && bulletX <= enemyX[i] + 8 &&
           bulletY >= enemyY[i] && bulletY <= enemyY[i] + 8){
          score3 += 10;
          bulletActive = false;
          enemyActive[i] = false;
          
          delay(100);
          enemyActive[i] = true;
          enemyX[i] = random(0, SCREEN_WIDTH - 8);
          enemyY[i] = random(-30, -5);
          
          if(score3 % 50 == 0){
            enemySpeed++;
          }
        }
      }
    }
  }
  
  for(int i = 0; i < 3; i++){
    if(enemyActive[i]){
      enemyY[i] += enemySpeed;
      
      if(enemyY[i] > SCREEN_HEIGHT){
        enemyActive[i] = false;
        enemyY[i] = random(-30, -5);
        enemyX[i] = random(0, SCREEN_WIDTH - 8);
        enemyActive[i] = true;
        
        score3 = max(0, score3 - 5);
      }
      
      if(enemyY[i] + 8 >= playerY3 && enemyY[i] <= playerY3 + 6){
        if(enemyX[i] + 8 >= playerX3 && enemyX[i] <= playerX3 + 8){
          lastGameScore = score3;
          if(score3 > highScore3) highScore3 = score3;
          waitingForButtonRelease = true;
          state = GAMEOVER;
          return;
        }
      }
    }
  }
  
  display.clearDisplay();
  
  display.fillTriangle(playerX3+4, playerY3,
                      playerX3,   playerY3+6,
                      playerX3+8, playerY3+6,
                      WHITE);
  
  if(bulletActive){
    display.fillRect(bulletX, bulletY, 2, 4, WHITE);
  }
  
  for(int i = 0; i < 3; i++){
    if(enemyActive[i]){
      display.fillRect(enemyX[i], enemyY[i], 8, 8, WHITE);
    }
  }
  
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("Score: ");
  display.print(score3);
  
  display.display();
  
  if(digitalRead(SELECT_BTN)==LOW && millis() - lastMove3 > buttonDebounceDelay){
    state=MENU;
  }
}

// ================== GAME 4: MEMORY MATCH (FIXED) ==================
void resetGame4(){
  sequenceLength = 1;
  currentStep = 0;
  showingSequence = true;
  score4 = 0;
  lives = 3;
  gameStarted = false;
  lastButtonCheck4 = millis();
  
  // Generate initial sequence
  for(int i = 0; i < 5; i++){
    sequence[i] = random(0, 4); // 0-3: UP, DOWN, LEFT, RIGHT
  }
  sequenceShowTime = millis();
}

void runGame4(){
  if(!gameStarted) {
    gameStarted = true;
    sequenceShowTime = millis();
    lastButtonCheck4 = millis();
    return;
  }
  
  if(showingSequence){
    // Show sequence for 1 second per item
    int totalShowTime = sequenceLength * 1000;
    if(millis() - sequenceShowTime > totalShowTime + 500){
      showingSequence = false;
      currentStep = 0;
      display.clearDisplay();
      display.setTextSize(1);
      display.setCursor(30, 25);
      display.print("YOUR TURN!");
      display.display();
      delay(800);
    } else {
      // Show current item
      display.clearDisplay();
      display.setTextSize(1);
      display.setCursor(40, 10);
      display.print("MEMORIZE!");
      
      display.setTextSize(2);
      
      // Calculate which item to show
      unsigned long elapsed = millis() - sequenceShowTime;
      int currentItem = elapsed / 1000;
      
      if(currentItem < sequenceLength){
        display.setCursor(50, 30);
        switch(sequence[currentItem]){
          case 0: display.print("UP"); break;
          case 1: display.print("DN"); break;
          case 2: display.print("LT"); break;
          case 3: display.print("RT"); break;
        }
        
        display.setTextSize(1);
        display.setCursor(10, 55);
        display.print("Item ");
        display.print(currentItem + 1);
        display.print("/");
        display.print(sequenceLength);
      }
      
      display.display();
    }
  } else {
    // Player's turn
    display.clearDisplay();
    display.setTextSize(1);
    
    // Show progress
    display.setCursor(20, 10);
    display.print("Step: ");
    display.print(currentStep + 1);
    display.print("/");
    display.print(sequenceLength);
    
    // Show prompt
    display.setTextSize(2);
    display.setCursor(50, 30);
    display.print("?");
    
    display.setTextSize(1);
    display.setCursor(30, 55);
    display.print("Press button");
    
    display.display();
    
    // Check button input
    if(millis() - lastButtonCheck4 > 200){
      lastButtonCheck4 = millis();
      
      int pressed = -1;
      if(digitalRead(UP_BTN) == LOW) pressed = 0;
      else if(digitalRead(DOWN_BTN) == LOW) pressed = 1;
      else if(digitalRead(LEFT_BTN) == LOW) pressed = 2;
      else if(digitalRead(RIGHT_BTN) == LOW) pressed = 3;
      
      if(pressed != -1){
        // Check if correct
        if(pressed == sequence[currentStep]){
          currentStep++;
          
          // Check if sequence completed
          if(currentStep >= sequenceLength){
            // Correct sequence!
            score4 += sequenceLength * 10;
            sequenceLength++;
            
            // Check win condition
            if(sequenceLength > 5){
              lastGameScore = score4;
              if(score4 > highScore4) highScore4 = score4;
              waitingForButtonRelease = true;
              state = GAMEOVER;
              return;
            }
            
            // Start next level
            showingSequence = true;
            sequenceShowTime = millis();
          }
        } else {
          // Wrong button
          lives--;
          if(lives <= 0){
            lastGameScore = score4;
            if(score4 > highScore4) highScore4 = score4;
            waitingForButtonRelease = true;
            state = GAMEOVER;
            return;
          } else {
            // Try again
            currentStep = 0;
            showingSequence = true;
            sequenceShowTime = millis();
          }
        }
      }
    }
    
    // Display score and lives
    display.setCursor(0,0);
    display.print("S:");
    display.print(score4);
    display.setCursor(60,0);
    display.print("L:");
    display.print(lives);
  }
  
  // Exit to menu
  if(digitalRead(SELECT_BTN)==LOW && millis() - lastButtonCheck4 > buttonDebounceDelay){
    state=MENU;
  }
}

void drawGameOver(){
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(WHITE);
  
  display.setCursor(25, 10);
  display.println("GAME");
  display.setCursor(25, 30);
  display.println("OVER");
  
  display.setTextSize(1);
  display.setCursor(35, 52);
  display.print("Score: ");
  display.print(lastGameScore);
  
  display.display();
}

void handleGameOverInput(){
  static unsigned long lastPress = 0;
  
  // Wait for button release
  if(waitingForButtonRelease){
    if(digitalRead(UP_BTN) && digitalRead(DOWN_BTN) && 
       digitalRead(LEFT_BTN) && digitalRead(RIGHT_BTN) && 
       digitalRead(SELECT_BTN)){
      waitingForButtonRelease = false;
      lastPress = millis();
    }
    return;
  }
  
  // Check for button press
  if(digitalRead(SELECT_BTN)==LOW && millis() - lastPress > 500){
    lastPress = millis();
    state = MENU;
    menuIndex = 0;
    drawMenu();
  }
}
```
