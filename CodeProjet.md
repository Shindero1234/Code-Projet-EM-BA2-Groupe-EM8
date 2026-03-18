# Code-Projet-EM-BA2-Groupe-EM8
# Code du projet EM BA2 d'ingénieur civil à l'école polytechnique du groupe EM8

#include <Arduino.h>
#include <AccelStepper.h>
#include <Servo.h>
#include <SPI.h>
#include <RF24.h>

// Broches
#define X_STEP_PIN A0
#define X_DIR_PIN A1
#define Y_STEP_PIN 3
#define Y_DIR_PIN 4
#define ENABLE_PIN 5
#define SERVO_PIN 6
#define DC_PIN1 7
#define DC_PIN2 8

// NRF24
RF24 radio(9, 10);
const byte addresses[][6] = {"1Node", "2Node"};
bool radioNumber = 0; // 0 pour le robot qui commence (joue O), 1 pour l'autre (joue X)
char receivedCase = '0';
char caseToSend = '0';
unsigned long lastReceiveTime = 0;

// Jeu
static const char VIDE = ' ';
static const uint8_t LIGNES_GAGNANTES[8][3] PROGMEM = {
  {1,2,3}, {4,5,6}, {7,8,9}, {1,4,7}, {2,5,8}, {3,6,9}, {1,5,9}, {3,5,7}
};
char plateau[10]; // plateau[0] non utilisé, plateau[1] à plateau[9] pour les cases
bool monTour = false;
bool partieTerminee = false;

// Moteurs
const float RAYON = 100.0;
const float VITESSE = 1200.0;
const float ACCEL = 600.0;
const float DISTANCE_CROIX = 200.0;
const float ESPACEMENT = 220.0;
float posX = 0, posY = 0;
AccelStepper moteurX(AccelStepper::DRIVER, X_STEP_PIN, X_DIR_PIN);
AccelStepper moteurY(AccelStepper::DRIVER, Y_STEP_PIN, Y_DIR_PIN);
Servo servoBic;
unsigned long duree = 1500; 

char verifierGagnant();
bool plateauPlein();
int8_t choisirCoup();
void allerVers(float x, float y);
void dessinerCercle();
void dessinerCroix();
void baisserBic();
void leverBic();
void envoyerCoup(int coup);
int recevoirCoup();
void moteurArriere();
void moteurStop();
void moteurAvant();

void setup() {
  Serial.begin(9600);
  servoBic.attach(SERVO_PIN);
  servoBic.write(90);
  pinMode(ENABLE_PIN, OUTPUT);
  digitalWrite(ENABLE_PIN, LOW);

  // Initialisation NRF24
  if (!radio.begin()) {
    Serial.println("NRF24 non détecté!");
    while (1);
  }
  radio.setPALevel(RF24_PA_LOW);
  radio.setDataRate(RF24_1MBPS);
  radio.setChannel(108);
  radio.setRetries(15, 15);

  pinMode(DC_PIN1, OUTPUT);
  pinMode(DC_PIN2, OUTPUT);

  if (radioNumber == 0) {
    radio.openWritingPipe(addresses[1]);
    radio.openReadingPipe(1, addresses[0]);
    monTour = true; // Robot 0 commence et joue O
    Serial.println("Robot 0 commence la partie (joue O).");
  } else {
    radio.openWritingPipe(addresses[0]);
    radio.openReadingPipe(1, addresses[1]);
    Serial.println("Robot 1 attend le premier coup (joue X).");
  }

  radio.startListening();
  Serial.println("NRF24 prêt (écoute)");

  moteurX.setMaxSpeed(VITESSE);
  moteurY.setMaxSpeed(VITESSE);
  moteurX.setAcceleration(ACCEL);
  moteurY.setAcceleration(ACCEL);

  for(uint8_t i=1; i<=9; i++) plateau[i] = VIDE;
  Serial.println("Plateau initialisé.");
  Serial.println("Prêt à jouer");
}

void loop() {
  if (partieTerminee) return;
  char g = verifierGagnant();
  if (g != VIDE) {
    Serial.print("GAGNANT : ");
    Serial.println(g);
    partieTerminee = true;
    return;
  }

  else if(plateauPlein()) {
    Serial.println("Match nul!");
    partieTerminee = true;
    while(1);
  }

  if(monTour) {
    Serial.println("\nloop: -TOUR DU ROBOT -");
    int8_t coup = choisirCoup();
    if(coup >= 0) {
      plateau[coup] = (radioNumber == 0) ? 'O' : 'X';
      Serial.print("Coup choisi: case ");
      Serial.println(coup);
      float x = ((coup-1)%3)*ESPACEMENT;
      float y = ((coup-1)/3)*ESPACEMENT;
      moteurAvant();
      allerVers(x, y);
      if (radioNumber == 0) {
        dessinerCercle();
      } else {
        dessinerCroix();
      }
      allerVers(0, 0);
      moteurArriere();
      envoyerCoup(coup);
      monTour = false;
      lastReceiveTime = millis();
    }
  }
  else {
    Serial.println("- ATTENTE COUP ADVERSE -");
    int coup = recevoirCoup();
    if(coup > 0 && coup <= 9) {
      if(plateau[coup] == VIDE) {
        plateau[coup] = (radioNumber == 0) ? 'X' : 'O';
        Serial.print("Coup adverse valide: case ");
        Serial.println(coup);
        monTour = true;
      }
    }
  }
  delay(1000);
}

char verifierGagnant() {
  for(uint8_t i=0; i<8; i++) {
    uint8_t a = pgm_read_byte(&LIGNES_GAGNANTES[i][0]);
    uint8_t b = pgm_read_byte(&LIGNES_GAGNANTES[i][1]);
    uint8_t c = pgm_read_byte(&LIGNES_GAGNANTES[i][2]);
    if(plateau[a] != VIDE && plateau[a] == plateau[b] && plateau[a] == plateau[c]) {
      return plateau[a];
    }
  }
  return VIDE;
}

bool plateauPlein() {
  for(uint8_t i=1; i<=9; i++) {
    if(plateau[i] == VIDE) {
      return false;
    }
  }
  return true;
}

int8_t choisirCoup() {
  const uint8_t ordre[9] = {5,1,3,7,9,2,4,6,8}; // Centre -> coins -> bords (index 1-9)
  for(uint8_t i=0; i<9; i++) {
    if(plateau[ordre[i]] == VIDE) {
      return ordre[i];
    }
  }
  return -1;
}

void envoyerCoup(int coup) {
  radio.stopListening();
  caseToSend = coup + '0';
  bool ok = radio.write(&caseToSend, sizeof(caseToSend));
  radio.startListening();
  if(ok) {
    Serial.print("Coup envoyé: ");
    Serial.println(caseToSend);
  } else {
    Serial.println("Échec envoi coup!");
  }
}

int recevoirCoup() {
  if (radio.available()) {
    radio.read(&receivedCase, sizeof(receivedCase));
    int move = receivedCase - '0';
    if(move >= 1 && move <= 9) {
      move = 10 - move;   // rotation du plateau
      return move;
    }
  }
  return -1;
}

void allerVers(float x, float y) {
  long stepsX = x * (200*16)/40.0;
  long stepsY = y * (200*16)/40.0;
  moteurX.moveTo(stepsX);
  moteurY.moveTo(stepsY);
  while(moteurX.distanceToGo() != 0 || moteurY.distanceToGo() != 0) {
    moteurX.run();
    moteurY.run();
  }
  posX = x;
  posY = y;
}

void dessinerCercle() {
  float centreX = posX;
  float centreY = posY;
  baisserBic();
  for(int i=0; i<=360; i+=5) {
    float rad = radians(i);
    allerVers(centreX + RAYON*cos(rad), centreY + RAYON*sin(rad));
  }
  leverBic();
  allerVers(centreX, centreY);
}

void dessinerCroix() {
  float x = posX;
  float y = posY;
  float demi = DISTANCE_CROIX/2;
  allerVers(x-demi, y-demi);
  baisserBic();
  allerVers(x+demi, y+demi);
  leverBic();
  allerVers(x+demi, y-demi);
  baisserBic();
  allerVers(x-demi, y+demi);
  leverBic();
  allerVers(x, y);
}

void baisserBic() {
  for(int pos=90; pos>=0; pos-=5) {
    servoBic.write(pos);
    delay(5);
  }
}

void leverBic() {
  for(int pos=0; pos<=90; pos+=5) {
    servoBic.write(pos);
    delay(5);
  }
}

void moteurAvant() {
  digitalWrite(DC_PIN1, HIGH);
  digitalWrite(DC_PIN2, LOW);
  delay(duree);
  moteurStop();
}

void moteurArriere() {
  digitalWrite(DC_PIN1, LOW);
  digitalWrite(DC_PIN2, HIGH);
  delay(duree);
  moteurStop();
}

void moteurStop() {
  digitalWrite(DC_PIN1, LOW);
  digitalWrite(DC_PIN2, LOW);
}
