#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ====== KONFIGURASI (sesuaikan dengan guide) ======
const char* WIFI_SSID      = "CirkitWifi";
const char* WIFI_PASS      = "";
const char* MQTT_SERVER    = "broker.emqx.io";
const int   MQTT_PORT      = 1883;
const char* MQTT_TOPIC_PUB = "ionic/final/tim01/data";
const char* MQTT_TOPIC_SUB = "ionic/final/tim01/cmd";
const char* DEVICE_ID      = "esp32-tim01";

// ====== PIN ======
#define DHT_PIN  12
#define DHT_TYPE DHT11
#define I2C_SDA  13
#define I2C_SCL  14

DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);
WiFiClient espClient;
PubSubClient mqtt(espClient);

unsigned long lastPublish = 0;
const unsigned long PUBLISH_INTERVAL = 5000;

// Baca DHT dengan percobaan ulang
bool readDHT(float &suhu, float &hum) {
  for (int i = 0; i < 3; i++) {
    suhu = dht.readTemperature();
    hum  = dht.readHumidity();
    if (!isnan(suhu) && !isnan(hum)) return true;
    delay(500);
  }
  return false;
}

void connectWiFi() {
  lcd.clear();
  lcd.print("Connect WiFi...");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected: " + WiFi.localIP().toString());
  lcd.clear();
  lcd.print("WiFi OK");
}

// Callback saat ada pesan masuk (JSON)
void onMessage(char* topic, byte* payload, unsigned int length) {
  JsonDocument doc;
  DeserializationError err = deserializeJson(doc, payload, length);
  if (err) {
    Serial.println("JSON error: " + String(err.c_str()));
    return;
  }
  const char* cmd = doc["cmd"] | "";
  Serial.printf("Perintah diterima: %s\n", cmd);
  // TODO: aksi sesuai perintah, misal nyalakan LED/relay
}

void connectMQTT() {
  while (!mqtt.connected()) {
    Serial.print("Connect MQTT...");
    String clientId = String(DEVICE_ID) + "-" + String(random(0xffff), HEX);
    if (mqtt.connect(clientId.c_str())) {
      Serial.println("OK");
      mqtt.subscribe(MQTT_TOPIC_SUB);
    } else {
      Serial.printf("gagal rc=%d, coba lagi 2 detik\n", mqtt.state());
      delay(2000);
    }
  }
}

void setup() {
  Serial.begin(115200);

  // Pin I2C khusus, harus sebelum lcd.init()
  Wire.begin(I2C_SDA, I2C_SCL);
  lcd.init();
  lcd.backlight();

  dht.begin();
  delay(2000);  // beri waktu DHT11 siap setelah dapat daya

  connectWiFi();
  mqtt.setServer(MQTT_SERVER, MQTT_PORT);
  mqtt.setCallback(onMessage);
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) connectWiFi();
  if (!mqtt.connected()) connectMQTT();
  mqtt.loop();

  if (millis() - lastPublish >= PUBLISH_INTERVAL) {
    lastPublish = millis();

    float suhu, hum;
    if (!readDHT(suhu, hum)) {
      Serial.printf("Gagal baca DHT11 di GPIO%d, cek VCC/GND/DATA\n", DHT_PIN);
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Sensor error");
      lcd.setCursor(0, 1);
      lcd.print("Cek kabel DHT");
      return;
    }

    // Tampil di LCD
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.printf("Suhu  : %.1f C", suhu);
    lcd.setCursor(0, 1);
    lcd.printf("Lembab: %.0f %%", hum);

    // Bentuk JSON lalu publish
    JsonDocument doc;
    doc["device"]      = DEVICE_ID;
    doc["temperature"] = suhu;
    doc["humidity"]    = hum;
    doc["rssi"]        = WiFi.RSSI();

    char buffer[200];
    serializeJson(doc, buffer);
    mqtt.publish(MQTT_TOPIC_PUB, buffer);
    Serial.println(buffer);
  }
}
