# project_semester4
#include <Fuzzy.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <LiquidCrystal_I2C.h>
#if defined(ESP32)
#include <WiFi.h>
#elif defined(ESP8266)
#include <ESP8266WiFi.h>
#endif
#include <Firebase_ESP_Client.h>
#include "addons/TokenHelper.h"
#include "addons/RTDBHelper.h"
//pin sensor dan aktuator
#define ONE_WIRE_BUS 4  // Hubungkan kabel data sensor DS18B20 ke pin 4 pada ESP32
#define fanPWM 5        // Hubungkan kabel sinyal PWM kipas ke pin D5
//RTDB
#define WIFI_SSID "T14"
#define WIFI_PASSWORD "kanzu203"
#define API_KEY "AIzaSyCaNVRvnd5EfiOvyRYlp3SzSYFeZIi_KfA"
#define DATABASE_URL "https://tubess5-default-rtdb.firebaseio.com/"

FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;
unsigned long sendDataPrevMillis = 0;

int count = 0;
int intValue;
float floatValue;
bool signupOK = false;
LiquidCrystal_I2C lcd(0x27, 16, 2);
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
//fuzzy
float suhu_udara;
int temperature;
int out_digital1;
int kecepatan;
String Indikator = "";
// Fuzzy
Fuzzy* fuzzy = new Fuzzy();
//INPUT 1
FuzzySet* dingin = new FuzzySet(10, 14, 15, 24);
FuzzySet* optimal = new FuzzySet(25, 28, 30, 31);
FuzzySet* tinggi = new FuzzySet(32, 33, 35, 37);
FuzzySet* malfungsi = new FuzzySet(-127.00, -127.00, -127.00, -127.00);
//INPUT 2
FuzzySet* rendah = new FuzzySet(10, 15, 20, 24);
FuzzySet* panas = new FuzzySet(25, 30, 35, 40);
//OUTPUT
FuzzySet* tidak_ada = new FuzzySet(0, 0, 0, 0);
FuzzySet* lambat = new FuzzySet(1, 10, 20, 29);
FuzzySet* sedang = new FuzzySet(30, 40, 50, 59);
FuzzySet* menengah = new FuzzySet(60, 65, 70, 79);
FuzzySet* cepat = new FuzzySet(80, 85, 90, 100);

void setup() {
  Serial.begin(9600);
  sensors.begin();
  //atur pin
  pinMode(fanPWM, OUTPUT);

  //LCD
  lcd.init();
  lcd.backlight();

  //INPUT 1
  FuzzyInput* analog1 = new FuzzyInput(1);
  analog1->addFuzzySet(dingin);
  analog1->addFuzzySet(optimal);
  analog1->addFuzzySet(tinggi);
  analog1->addFuzzySet(malfungsi);
  fuzzy->addFuzzyInput(analog1);
  //INPUT 2
  FuzzyInput* analog2 = new FuzzyInput(2);
  analog1->addFuzzySet(rendah);
  analog1->addFuzzySet(panas);
  fuzzy->addFuzzyInput(analog2);
  //OUTPUT 1
  FuzzyOutput* digital1 = new FuzzyOutput(1);
  digital1->addFuzzySet(tidak_ada);
  digital1->addFuzzySet(lambat);
  digital1->addFuzzySet(menengah);
  digital1->addFuzzySet(sedang);
  digital1->addFuzzySet(cepat);
  fuzzy->addFuzzyOutput(digital1);

  //RULE 1
  FuzzyRuleAntecedent* dingin_rendah = new FuzzyRuleAntecedent();
  dingin_rendah->joinWithAND(dingin, rendah);
  FuzzyRuleConsequent* digital1_lambat1 = new FuzzyRuleConsequent();
  digital1_lambat1->addOutput(lambat);
  FuzzyRule* fuzzyRule1 = new FuzzyRule(1, dingin_rendah, digital1_lambat1);
  fuzzy->addFuzzyRule(fuzzyRule1);
  //RULE 2
  FuzzyRuleAntecedent* dingin_panas = new FuzzyRuleAntecedent();
  dingin_panas->joinWithAND(dingin, panas);
  FuzzyRuleConsequent* digital1_menengah2 = new FuzzyRuleConsequent();
  digital1_menengah2->addOutput(menengah);
  FuzzyRule* fuzzyRule2 = new FuzzyRule(2, dingin_panas, digital1_menengah2);
  fuzzy->addFuzzyRule(fuzzyRule2);
  //RULE 3
  FuzzyRuleAntecedent* optimal_rendah = new FuzzyRuleAntecedent();
  optimal_rendah->joinWithAND(optimal, rendah);
  FuzzyRuleConsequent* digital1_menengah3 = new FuzzyRuleConsequent();
  digital1_menengah3->addOutput(menengah);
  FuzzyRule* fuzzyRule3 = new FuzzyRule(3, optimal_rendah, digital1_menengah3);
  fuzzy->addFuzzyRule(fuzzyRule3);
  //RULE 4
  FuzzyRuleAntecedent* optimal_panas = new FuzzyRuleAntecedent();
  optimal_panas->joinWithAND(optimal, panas);
  FuzzyRuleConsequent* digital1_sedang4 = new FuzzyRuleConsequent();
  digital1_sedang4->addOutput(sedang);
  FuzzyRule* fuzzyRule4 = new FuzzyRule(4, optimal_rendah, digital1_sedang4);
  fuzzy->addFuzzyRule(fuzzyRule4);
  //RULE 5
  FuzzyRuleAntecedent* tinggi_panas = new FuzzyRuleAntecedent();
  tinggi_panas->joinWithAND(tinggi, panas);
  FuzzyRuleConsequent* digital1_cepat5 = new FuzzyRuleConsequent();
  digital1_cepat5->addOutput(cepat);
  FuzzyRule* fuzzyRule5 = new FuzzyRule(5, tinggi_panas, digital1_cepat5);
  fuzzy->addFuzzyRule(fuzzyRule5);
  //RULE 6
  FuzzyRuleAntecedent* tinggi_rendah = new FuzzyRuleAntecedent();
  tinggi_rendah->joinWithAND(tinggi, rendah);
  FuzzyRuleConsequent* digital1_sedang6 = new FuzzyRuleConsequent();
  digital1_sedang6->addOutput(sedang);
  FuzzyRule* fuzzyRule6 = new FuzzyRule(6, tinggi_rendah, digital1_sedang6);
  fuzzy->addFuzzyRule(fuzzyRule6);
  //RULE 7
  FuzzyRuleAntecedent* ifmalfungsi = new FuzzyRuleAntecedent();
  ifmalfungsi->joinSingle(malfungsi);
  FuzzyRuleConsequent* digital1_tidak_ada7 = new FuzzyRuleConsequent();
  digital1_tidak_ada7->addOutput(tidak_ada);
  FuzzyRule* fuzzyRule7 = new FuzzyRule(7, ifmalfungsi, digital1_tidak_ada7);
  fuzzy->addFuzzyRule(fuzzyRule7);
  int suhu_air = 0;
  int suhu_udara = 0;
  int kelembapan_udara = 0;
  //RTDB
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(300);
  }
  Serial.println();
  Serial.print("Connected with IP: ");
  Serial.println(WiFi.localIP());
  Serial.println();
  /* Assign the api key (required) */
  config.api_key = API_KEY;
  /* Assign the RTDB URL (required) */
  config.database_url = DATABASE_URL;
  /* Sign up */
  if (Firebase.signUp(&config, &auth, "", "")) {
    Serial.println("ok");
    signupOK = true;
  } else {
    Serial.printf("%s\n", config.signer.signupError.message.c_str());
  }
  config.token_status_callback = tokenStatusCallback;  //see addons/TokenHelper.h

  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
}

void loop() {
  if (Firebase.ready() && signupOK && (millis() - sendDataPrevMillis > 15000 || sendDataPrevMillis == 0)) {
    sendDataPrevMillis = millis();
    sensors.requestTemperatures();
    int suhu_air = 13;
    int in_analog1 = suhu_air;
    int in_analog2 = suhu_udara;
    Serial.print("SuhuAir: ");
    Serial.print(in_analog1);
    Serial.print("|");
    Serial.print("SuhuUdara: ");
    Serial.println(in_analog2);
    //
    fuzzy->setInput(1, in_analog1);
    fuzzy->setInput(2, in_analog2);
    fuzzy->fuzzify();

    int out_digital1 = fuzzy->defuzzify(1);
    Serial.print("rasio putaran: ");
    Serial.println(out_digital1);
    ;
    Serial.println(fuzzy->defuzzify(1));
    if (out_digital1 >= 0 && out_digital1 <= 30) {
      String Indikator = "Lambat";
      int kecepatan = 100;
      Serial.println(Indikator);
    } else if (out_digital1 >= 31 && out_digital1 <= 60) {
      String Indikator = "menengah";
      int kecepatan = 150;
      Serial.println(Indikator);
    } else if (out_digital1 >= 61 && out_digital1 <= 80) {
      String Indikator = "sedang";
      int kecepatan = 185;
      Serial.println(Indikator);
    } else if (out_digital1 >= 81 && out_digital1 <= 100) {
      String Indikator = "cepat";
      int kecepatan = 255;
      Serial.println(Indikator);
    } else {
      String Indikator = "Tidak terdefinisi";  // Handle values outside the specified ranges
      int kecepatan = 0;
      Serial.println(Indikator);
    }
    analogWrite(fanPWM, kecepatan);
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Suhu Air:");
    lcd.print(suhu_air);
    lcd.print("C");
    lcd.setCursor(0, 1);
    lcd.print("kipas:");
    lcd.print(out_digital1);
    lcd.print("");
    if (Firebase.RTDB.setInt(&fbdo, "/Kelompok1/Suhu_Air", in_analog1)) {
      Serial.println("PASSED");
      Serial.println("PATH: " + fbdo.dataPath());
      Serial.println("TYPE: " + fbdo.dataType());
    } else {
      Serial.println("FAILED");
      Serial.println("REASON: " + fbdo.errorReason());
    }
    if (Firebase.RTDB.getFloat(&fbdo, "/Kelompok3/temperature")) {
      if (fbdo.dataType() == "float") {
        suhu_udara = fbdo.intData();
        Serial.println(suhu_udara);
      } else {
        Serial.println(fbdo.errorReason());
      }
    }
    if (Firebase.RTDB.setFloat(&fbdo, "/Kelompok1/Kecepatan_kipas", out_digital1)) {
      Serial.println("PASSED");
      Serial.println("PATH: " + fbdo.dataPath());
      Serial.println("TYPE: " + fbdo.dataType());
    } else {
      Serial.println("FAILED");
      Serial.println("REASON: " + fbdo.errorReason());
    }
  }
  delay(1000);
}
