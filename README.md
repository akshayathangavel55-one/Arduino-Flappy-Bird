# Arduino-Flappy-Bird
A Flappy Bird-inspired game built using Arduino and LCD, first simulated in Wokwi and later implemented on real hardware.
#here is the code

#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// -------------------- LCD --------------------
LiquidCrystal_I2C lcd(0x27, 16, 2);

// -------------------- Pins -------------------
#define BUTTON_PIN 2

// -------------------- Game settings ----------
#define TERRAIN_WIDTH 16
#define HERO_HORIZONTAL_POSITION 1

// -------------------- Custom characters -------
#define SPRITE_RUN1 1
#define SPRITE_RUN2 2
#define SPRITE_JUMP 3
#define SPRITE_JUMP_UPPER '.'
#define SPRITE_JUMP_LOWER 4

#define SPRITE_TERRAIN_EMPTY ' '
#define SPRITE_TERRAIN_SOLID 5
#define SPRITE_TERRAIN_SOLID_RIGHT 6
#define SPRITE_TERRAIN_SOLID_LEFT 7

// -------------------- Terrain types -----------
#define TERRAIN_EMPTY 0
#define TERRAIN_LOWER_BLOCK 1
#define TERRAIN_UPPER_BLOCK 2

// -------------------- Hero positions ----------
#define HERO_POSITION_OFF 0

#define HERO_POSITION_RUN_LOWER_1 1
#define HERO_POSITION_RUN_LOWER_2 2

#define HERO_POSITION_JUMP_1 3
#define HERO_POSITION_JUMP_2 4
#define HERO_POSITION_JUMP_3 5
#define HERO_POSITION_JUMP_4 6
#define HERO_POSITION_JUMP_5 7
#define HERO_POSITION_JUMP_6 8
#define HERO_POSITION_JUMP_7 9
#define HERO_POSITION_JUMP_8 10

#define HERO_POSITION_RUN_UPPER_1 11
#define HERO_POSITION_RUN_UPPER_2 12

// -------------------- Game variables ---------
char terrainUpper[TERRAIN_WIDTH + 1];
char terrainLower[TERRAIN_WIDTH + 1];

volatile bool buttonPressed = false;


// ======================================================
// Create custom characters
// ======================================================
void initializeGraphics() {

  static byte graphics[] = {

    // Run position 1
    B00000,
    B01110,
    B01101,
    B00110,
    B11110,
    B01110,
    B10010,
    B00000,

    // Run position 2
    B00000,
    B01110,
    B01101,
    B00110,
    B11110,
    B01110,
    B01100,
    B00000,

    // Jump
    B00000,
    B01110,
    B01101,
    B11110,
    B00010,
    B01110,
    B00000,
    B00000,

    // Jump lower
    B01110,
    B00000,
    B00000,
    B10000,
    B00000,
    B00000,
    B00000,
    B00000,

    // Ground
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,
    B11111,

    // Ground right
    B00011,
    B00011,
    B00011,
    B00011,
    B00011,
    B00011,
    B00011,
    B00011,

    // Ground left
    B11000,
    B11000,
    B11000,
    B11000,
    B11000,
    B11000,
    B11000,
    B11000
  };

  for (int i = 0; i < 7; i++) {
    lcd.createChar(i + 1, &graphics[i * 8]);
  }

  for (int i = 0; i < TERRAIN_WIDTH; i++) {
    terrainUpper[i] = SPRITE_TERRAIN_EMPTY;
    terrainLower[i] = SPRITE_TERRAIN_EMPTY;
  }
}


// ======================================================
// Move terrain to the left
// ======================================================
void advanceTerrain(char* terrain, byte newTerrain) {

  for (int i = 0; i < TERRAIN_WIDTH; i++) {

    char current = terrain[i];

    char next =
      (i == TERRAIN_WIDTH - 1)
      ? newTerrain
      : terrain[i + 1];

    switch (current) {

      case SPRITE_TERRAIN_EMPTY:

        terrain[i] =
          (next == SPRITE_TERRAIN_SOLID)
          ? SPRITE_TERRAIN_SOLID_RIGHT
          : SPRITE_TERRAIN_EMPTY;

        break;

      case SPRITE_TERRAIN_SOLID:

        terrain[i] =
          (next == SPRITE_TERRAIN_EMPTY)
          ? SPRITE_TERRAIN_SOLID_LEFT
          : SPRITE_TERRAIN_SOLID;

        break;

      case SPRITE_TERRAIN_SOLID_RIGHT:
        terrain[i] = SPRITE_TERRAIN_SOLID;
        break;

      case SPRITE_TERRAIN_SOLID_LEFT:
        terrain[i] = SPRITE_TERRAIN_EMPTY;
        break;
    }
  }
}


// ======================================================
// Draw the hero and game screen
// ======================================================
bool drawHero(
  byte position,
  char* terrainUpper,
  char* terrainLower,
  unsigned int score
) {

  bool collision = false;

  char upperSave =
    terrainUpper[HERO_HORIZONTAL_POSITION];

  char lowerSave =
    terrainLower[HERO_HORIZONTAL_POSITION];

  byte upper;
  byte lower;

  switch (position) {

    case HERO_POSITION_OFF:
      upper = lower = SPRITE_TERRAIN_EMPTY;
      break;

    case HERO_POSITION_RUN_LOWER_1:
      upper = SPRITE_TERRAIN_EMPTY;
      lower = SPRITE_RUN1;
      break;

    case HERO_POSITION_RUN_LOWER_2:
      upper = SPRITE_TERRAIN_EMPTY;
      lower = SPRITE_RUN2;
      break;

    case HERO_POSITION_JUMP_1:
    case HERO_POSITION_JUMP_8:
      upper = SPRITE_TERRAIN_EMPTY;
      lower = SPRITE_JUMP;
      break;

    case HERO_POSITION_JUMP_2:
    case HERO_POSITION_JUMP_7:
      upper = SPRITE_JUMP_UPPER;
      lower = SPRITE_JUMP_LOWER;
      break;

    case HERO_POSITION_JUMP_3:
    case HERO_POSITION_JUMP_4:
    case HERO_POSITION_JUMP_5:
    case HERO_POSITION_JUMP_6:
      upper = SPRITE_JUMP;
      lower = SPRITE_TERRAIN_EMPTY;
      break;

    case HERO_POSITION_RUN_UPPER_1:
      upper = SPRITE_RUN1;
      lower = SPRITE_TERRAIN_EMPTY;
      break;

    case HERO_POSITION_RUN_UPPER_2:
      upper = SPRITE_RUN2;
      lower = SPRITE_TERRAIN_EMPTY;
      break;

    default:
      upper = lower = SPRITE_TERRAIN_EMPTY;
      break;
  }


  // Check collision on upper row
  if (upper != SPRITE_TERRAIN_EMPTY) {

    terrainUpper[HERO_HORIZONTAL_POSITION] = upper;

    if (upperSave != SPRITE_TERRAIN_EMPTY) {
      collision = true;
    }
  }


  // Check collision on lower row
  if (lower != SPRITE_TERRAIN_EMPTY) {

    terrainLower[HERO_HORIZONTAL_POSITION] = lower;

    if (lowerSave != SPRITE_TERRAIN_EMPTY) {
      collision = true;
    }
  }


  // Determine number of digits in score
  byte digits;

  if (score > 9999)
    digits = 5;
  else if (score > 999)
    digits = 4;
  else if (score > 99)
    digits = 3;
  else if (score > 9)
    digits = 2;
  else
    digits = 1;


  // End strings
  terrainUpper[TERRAIN_WIDTH] = '\0';
  terrainLower[TERRAIN_WIDTH] = '\0';


  // Print upper terrain
  char temp = terrainUpper[16 - digits];

  terrainUpper[16 - digits] = '\0';

  lcd.setCursor(0, 0);
  lcd.print(terrainUpper);

  terrainUpper[16 - digits] = temp;


  // Print lower terrain
  lcd.setCursor(0, 1);
  lcd.print(terrainLower);


  // Print score
  lcd.setCursor(16 - digits, 0);
  lcd.print(score);


  // Restore terrain
  terrainUpper[HERO_HORIZONTAL_POSITION] = upperSave;
  terrainLower[HERO_HORIZONTAL_POSITION] = lowerSave;

  return collision;
}


// ======================================================
// Button interrupt
// ======================================================
void buttonPush() {
  buttonPressed = true;
}


// ======================================================
// SETUP
// ======================================================
void setup() {

  // Button uses internal pull-up resistor
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Initialize LCD
  lcd.init();
  lcd.backlight();

  // Create custom characters
  initializeGraphics();

  // Attach button interrupt
  attachInterrupt(
    digitalPinToInterrupt(BUTTON_PIN),
    buttonPush,
    FALLING
  );

  // Random seed
  randomSeed(analogRead(A0));

  // Start screen
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Arduino Dino");
  lcd.setCursor(0, 1);
  lcd.print("Press Button!");

  delay(1500);
}


// ======================================================
// MAIN GAME LOOP
// ======================================================
void loop() {

  static byte heroPos =
    HERO_POSITION_RUN_LOWER_1;

  static byte newTerrainType =
    TERRAIN_EMPTY;

  static byte newTerrainDuration = 1;

  static bool playing = false;

  static bool blink = false;

  static unsigned int distance = 0;


  // --------------------------------------------------
  // Start screen / Game over screen
  // --------------------------------------------------
  if (!playing) {

    drawHero(
      blink ? HERO_POSITION_OFF : heroPos,
      terrainUpper,
      terrainLower,
      distance >> 3
    );

    if (blink) {

      lcd.setCursor(0, 0);
      lcd.print("Press Button   ");
    }

    delay(250);

    blink = !blink;


    // Start game when button is pressed
    if (buttonPressed) {

      initializeGraphics();

      heroPos =
        HERO_POSITION_RUN_LOWER_1;

      playing = true;

      buttonPressed = false;

      distance = 0;

      newTerrainType = TERRAIN_EMPTY;
      newTerrainDuration = 1;

      lcd.clear();
    }

    return;
  }


  // --------------------------------------------------
  // Move terrain
  // --------------------------------------------------

  advanceTerrain(
    terrainLower,
    newTerrainType == TERRAIN_LOWER_BLOCK
      ? SPRITE_TERRAIN_SOLID
      : SPRITE_TERRAIN_EMPTY
  );

  advanceTerrain(
    terrainUpper,
    newTerrainType == TERRAIN_UPPER_BLOCK
      ? SPRITE_TERRAIN_SOLID
      : SPRITE_TERRAIN_EMPTY
  );


  // --------------------------------------------------
  // Generate new terrain
  // --------------------------------------------------

  if (--newTerrainDuration == 0) {

    if (newTerrainType == TERRAIN_EMPTY) {

      if (random(3) == 0)
        newTerrainType = TERRAIN_UPPER_BLOCK;
      else
        newTerrainType = TERRAIN_LOWER_BLOCK;

      newTerrainDuration =
        2 + random(10);

    } else {

      newTerrainType = TERRAIN_EMPTY;

      newTerrainDuration =
        10 + random(10);
    }
  }


  // --------------------------------------------------
  // Jump button
  // --------------------------------------------------

  if (buttonPressed) {

    if (heroPos <= HERO_POSITION_RUN_LOWER_2) {

      heroPos =
        HERO_POSITION_JUMP_1;
    }

    buttonPressed = false;
  }


  // --------------------------------------------------
  // Draw hero
  // --------------------------------------------------

  if (
    drawHero(
      heroPos,
      terrainUpper,
      terrainLower,
      distance >> 3
    )
  ) {

    // Collision
    playing = false;

    lcd.clear();

    lcd.setCursor(3, 0);
    lcd.print("GAME OVER");

    lcd.setCursor(3, 1);
    lcd.print("Score: ");
    lcd.print(distance >> 3);

    delay(1500);

  } else {

    // ------------------------------------------------
    // Hero movement
    // ------------------------------------------------

    if (
      heroPos == HERO_POSITION_RUN_LOWER_2 ||
      heroPos == HERO_POSITION_JUMP_8
    ) {

      heroPos =
        HERO_POSITION_RUN_LOWER_1;

    }

    else if (
      heroPos >= HERO_POSITION_JUMP_3 &&
      heroPos <= HERO_POSITION_JUMP_5 &&
      terrainLower[HERO_HORIZONTAL_POSITION]
        != SPRITE_TERRAIN_EMPTY
    ) {

      heroPos =
        HERO_POSITION_RUN_UPPER_1;
    }

    else if (
      heroPos >= HERO_POSITION_RUN_UPPER_1 &&
      terrainLower[HERO_HORIZONTAL_POSITION]
        == SPRITE_TERRAIN_EMPTY
    ) {

      heroPos =
        HERO_POSITION_JUMP_5;
    }

    else if (
      heroPos == HERO_POSITION_RUN_UPPER_2
    ) {

      heroPos =
        HERO_POSITION_RUN_UPPER_1;
    }

    else {

      ++heroPos;
    }


    // Increase score
    ++distance;
  }


  // --------------------------------------------------
  // Game speed
  // --------------------------------------------------

  delay(100);
}