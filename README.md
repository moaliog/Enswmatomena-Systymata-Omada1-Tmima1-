#include <Arduino.h>

// --- Pin Definitions ---
const int L_PWM = 8;
const int L_DIR = 9;
const int R_PWM = 10;
const int R_DIR = 11;

const int DIG_L = 0;
const int DIG_R = 1;
const int SENS_L = 27;
const int SENS_C = 26;
const int SENS_R = 28;

const int BTN1 = 20; // GP20
const int BTN2 = 21; // GP21

// --- Racing Variables ---
// Ρυθμίσεις PID με Φίλτρο Χαμηλής Διέλευσης για εξάλειψη του ψαρώματος και διατήρηση ορμής
float kp = 23.0;   // Σταθερό και άμεσο τιμόνι
float kd = 113.0;  // Ιδανικό Kd που πλέον λειτουργεί ομαλά χάρη στο ψηφιακό φίλτρο
int BASE_SPEED_FAST = 4095;  // 100% PWM (Μέγιστη ταχύτητα ευθείας)
int BASE_SPEED_SLOW = 4095;  // 100% PWM (Constant Max Power)

float last_error = 0;
float filtered_d = 0;       // Μεταβλητή για το φίλτρο D-Term
const float alpha = 0.20;   // Συντελεστής φιλτραρίσματος (20% νέο δείγμα, 80% μνήμη)

int stop_counter = 0;

uint16_t WHITE_LIMIT = 0;
uint16_t BLACK_LIMIT = 0;
const int BLACK_OFFSET = 375;  // ~6000 στην κλίμακα 12-bit
const int WHITE_OFFSET = 250;  // ~4000 στην κλίμακα 12-bit

// --- Drive Function (0 έως 4095) ---
void drive(long left, long right) {
  // Αριστερό Μοτέρ
  if (left >= 0) {
    digitalWrite(L_DIR, LOW);
    analogWrite(L_PWM, min(4095, (int)left));
  } else {
    digitalWrite(L_DIR, HIGH);
    analogWrite(L_PWM, min(4095, (int)abs(left)));
  }
  
  // Δεξί Μοτέρ
  if (right >= 0) {
    digitalWrite(R_DIR, LOW);
    analogWrite(R_PWM, min(4095, (int)right));
  } else {
    digitalWrite(R_DIR, HIGH);
    analogWrite(R_PWM, min(4095, (int)abs(right)));
  }
}

// --- Calibration ---
void calibrate() {
  Serial.println("CALIBRATING... Place IR3 on line.");
  drive(0, 0);
  delay(2000);
  long s_b = 0;
  long s_w = 0;
  
  for (int i = 0; i < 40; i++) {
    s_b += analogRead(SENS_C);
    s_w += (analogRead(SENS_L) + analogRead(SENS_R)) / 2;
    delay(20);
  }
  
  BLACK_LIMIT = (s_b / 40) + BLACK_OFFSET;
  WHITE_LIMIT = (s_w / 40) - WHITE_OFFSET;
  
  Serial.print("BLACK_LIMIT: "); Serial.println(BLACK_LIMIT);
  Serial.print("WHITE_LIMIT: "); Serial.println(WHITE_LIMIT);
  Serial.println("Ready!");
}

// --- Zig Zag Search (Έσχατη Λύση) ---
void zig_zag_search() {
  Serial.println("Line lost completely! Searching...");
  int search_speed = 1250; 
  int durations[] = {150, 350, 550, 800};
  
  for (int duration : durations) {
    drive(-search_speed, search_speed);
    unsigned long start_t = millis();
    while (millis() - start_t < (unsigned long)duration) {
      if (analogRead(SENS_C) < BLACK_LIMIT || digitalRead(DIG_L) == LOW || digitalRead(DIG_R) == LOW) return;
    }
    search_speed = -search_speed;
  }
  drive(0, 0);
  while (true) delay(1000);
}

// --- Track 1 (Υβριδική - Γρήγορη) ---
void track_1() {
  Serial.println("Running Track 1 (C++ Turbo)...");
  last_error = 0;
  filtered_d = 0;
  
  int T1_FAST = 3250; 
  int T1_SLOW = 2187; 
  
  unsigned long start_time = millis();
  int white_confirm_counter = 0;
  
  while (true) {
    int d_l = digitalRead(DIG_L);
    int d_r = digitalRead(DIG_R);
    uint16_t v_l = analogRead(SENS_L);
    uint16_t v_c = analogRead(SENS_C);
    uint16_t v_r = analogRead(SENS_R);
    
    // --- ΣΥΝΘΗΚΗ ΤΕΡΜΑΤΙΣΜΟΥ (ΟΛΟΙ ΑΣΠΡΟ) ---
    if (millis() - start_time > 500) {
      if (v_c > WHITE_LIMIT && v_l > WHITE_LIMIT && v_r > WHITE_LIMIT && d_l == HIGH && d_r == HIGH) {
        white_confirm_counter++;
        if (white_confirm_counter > 60) { // Προσαρμοσμένο για 500us loop
          drive(0, 0);
          Serial.println("End of Track 1.");
          break;
        }
      } else {
        white_confirm_counter = 0; 
      }
    }
    
    float error = (float)v_l - (float)v_r;
    
    // Φιλτράρισμα D-term
    float raw_d = error - last_error;
    filtered_d = (alpha * raw_d) + ((1.0 - alpha) * filtered_d);
    
    float correction = (kp * error) + (kd * filtered_d);
    last_error = error;
    
    int current_speed = (v_c < BLACK_LIMIT || v_l < BLACK_LIMIT || v_r < BLACK_LIMIT) ? T1_FAST : T1_SLOW;
    drive(current_speed - correction, current_speed + correction);
    delayMicroseconds(500); 
  }
}

// --- Track 2/3 (Κανονική Ταχύτητα - 2kHz Setup) ---
void track_2_3() {
  Serial.println("Running Track 2/3 (C++ Stable)...");
  last_error = 0;
  filtered_d = 0;
  stop_counter = 0;
  
  while (true) {
    int d_l = digitalRead(DIG_L);
    int d_r = digitalRead(DIG_R);
    uint16_t v_l = analogRead(SENS_L);
    uint16_t v_c = analogRead(SENS_C);
    uint16_t v_r = analogRead(SENS_R);

    // --- ΕΞΥΠΝΗ ΔΙΑΧΕΙΡΙΣΗ ΑΠΩΛΕΙΑΣ ΓΡΑΜΜΗΣ ---
    if (v_c > WHITE_LIMIT && d_l == HIGH && d_r == HIGH) {
      stop_counter++;
      if (stop_counter > 900) { // Όριο προσαρμοσμένο για 500us loop
        zig_zag_search();
        stop_counter = 0;
      } else {
        // Δυναμική επαναφορά χωρίς βίαιο φρενάρισμα (0 αντί για -500) για μέγιστη διατήρηση ορμής
        if (last_error > 100) { 
          drive(0, BASE_SPEED_SLOW); 
        } else if (last_error < -100) {
          drive(BASE_SPEED_SLOW, 0); 
        } else {
          drive(3200, 3200); 
        }
      }
    } else if (d_l == LOW && d_r == LOW && v_c < BLACK_LIMIT) {
      drive(0, 0);
      Serial.println("End of Track 2/3.");
      break;
    } else {
      stop_counter = 0;
      
      // Αγωνιστικό ψηφιακό στρίψιμο: Απλό σβήσιμο (0) αντί για ανάποδο φρένο (-400)
      // Επιτρέπει στο ρομπότ να «ρολάρει» γρήγορα στη στροφή χωρίς να κόβει ταχύτητα
      if (d_l == LOW) {
        drive(0, 4095); 
      } else if (d_r == LOW) {
        drive(4095, 0);
      } else {
        float error = (float)v_l - (float)v_r;
        
        // Υπολογισμός και Φιλτράρισμα D-term (Low-Pass Filter)
        float raw_d = error - last_error;
        filtered_d = (alpha * raw_d) + ((1.0 - alpha) * filtered_d);
        
        float correction = (kp * error) + (kd * filtered_d);
        last_error = error;
        
        int c_speed = (v_c < (BLACK_LIMIT + 312)) ? BASE_SPEED_FAST : BASE_SPEED_SLOW;
        drive(c_speed - correction, c_speed + correction);
      }
    }
    delayMicroseconds(500); // 500us delay για ταχύτητα ελέγχου στα 2kHz
  }
}

void setup() {
  Serial.begin(115200);
  
  pinMode(L_DIR, OUTPUT); pinMode(R_DIR, OUTPUT);
  pinMode(DIG_L, INPUT); pinMode(DIG_R, INPUT);
  pinMode(BTN1, INPUT_PULLUP); pinMode(BTN2, INPUT_PULLUP);
  
  analogWriteResolution(12);
  analogReadResolution(12);
}

void loop() {
  if (digitalRead(BTN1) == LOW) {
    delay(200); 
    calibrate();
    track_1();
  }
  
  if (digitalRead(BTN2) == LOW) {
    delay(200); 
    calibrate();
    track_2_3();
  }
}
