

#include <WiFi.h>
#include <HardwareSerial.h>
#include "DHT.h" 
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- НАСТРОЙКИ ---
const char* ssid     = "Bektur's Nest"; 
const char* password = "yellownest009";
const char* host = "api.thingspeak.com";
String apiKey = "UX9PUCCV4HCXXMJV"; 

const int redLed = 5;      
const int greenLed = 19; 
const int dhtPin = 15;     
const int relayPin = 18;  
#define DHTTYPE DHT22      

#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 64 
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

DHT dht(dhtPin, DHTTYPE);
HardwareSerial SerialSensor(2); 

unsigned long lastTimeSent = 0; 
const unsigned long sendInterval = 20000; 

bool fanIsOn = false; 

void setup() {
  Serial.begin(115200);
  
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { 
    Serial.println("OLED error!");
  }
  
  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(2);
  display.setCursor(10, 10);
  display.println("AERO");
  display.setCursor(10, 35);
  display.println("SHIELD");
  display.display();

  SerialSensor.begin(9600, SERIAL_8N1, 16, 17); 
  dht.begin(); 
  
  pinMode(redLed, OUTPUT);
  pinMode(greenLed, OUTPUT);
  
  // Возвращаем как было в старом коде:
  pinMode(relayPin, INPUT); 

  // --- УМНОЕ ПОДКЛЮЧЕНИЕ К WIFI ---
  Serial.print("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  unsigned long startAttemptTime = millis();
  
  while (WiFi.status() != WL_CONNECTED && millis() - startAttemptTime < 10000) { 
    delay(500); 
    Serial.print("."); 
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected!");
  } else {
    Serial.println("\nWiFi Failed (Offline Mode)");
  }
  
  delay(1500); 
}

void loop() {
  if (SerialSensor.available() >= 32) {
    uint8_t buffer[32];
    SerialSensor.readBytes(buffer, 32);

    if (buffer[0] == 0x42 && buffer[1] == 0x4D) {
      int pm25 = (buffer[12] << 8) | buffer[13];
      float t = dht.readTemperature();
      float h = dht.readHumidity();

      if (pm25 > 45) fanIsOn = true; 
      else if (pm25 < 30) fanIsOn = false;

      // --- ОБНОВЛЕНИЕ ДИСПЛЕЯ ---
      display.clearDisplay();
      display.setTextSize(1);
      display.setCursor(0, 0);
      display.print("Aero Shield ");
      display.println(WiFi.status() == WL_CONNECTED ? "[ON]" : "[OFF]");
      display.drawLine(0, 10, 128, 10, WHITE);

      display.setTextSize(2);
      display.setCursor(0, 18);
      display.print("PM: "); display.println(pm25);

      display.setTextSize(1);
      display.setCursor(0, 42);
      display.print("Temp: "); display.print(t, 1); display.println(" C");
      display.setCursor(0, 54);
      display.print("Humi: "); display.print(h, 0); display.print(" % ");

      // --- УПРАВЛЕНИЕ ЖЕЛЕЗОМ (Возвращаем старую рабочую логику) ---
      if (fanIsOn) { 
        digitalWrite(redLed, HIGH);
        digitalWrite(greenLed, LOW);
        
        pinMode(relayPin, OUTPUT);    // ВКЛЮЧАЕМ РЕЛЕ
        digitalWrite(relayPin, LOW);  
        
        display.setCursor(85, 42);
        display.print("FAN:ON");
        display.fillRect(115, 15, 10, 45, WHITE); 
      } 
      else {
        digitalWrite(redLed, LOW);
        digitalWrite(greenLed, HIGH);
        
        pinMode(relayPin, INPUT);     // ВЫКЛЮЧАЕМ РЕЛЕ (как в старом коде)
        
        display.setCursor(85, 42);
        display.print("FAN:OFF");
        display.drawRect(115, 15, 10, 45, WHITE); 
      }
      
      display.display();

      if (WiFi.status() == WL_CONNECTED && (millis() - lastTimeSent > sendInterval)) {
        WiFiClient client;
        if (client.connect(host, 80)) {
          String url = "/update?api_key=" + apiKey + "&field1=" + String(pm25) + "&field2=" + String(t) + "&field3=" + String(h);
          client.print(String("GET ") + url + " HTTP/1.1\r\n" + "Host: " + host + "\r\n" + "Connection: close\r\n\r\n");
          lastTimeSent = millis(); 
        }
      }
    }
  }
  delay(100); 
}
