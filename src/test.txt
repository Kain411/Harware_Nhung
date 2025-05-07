#include <Arduino.h>
#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include <MQUnifiedsensor.h>

// Cấu hình WiFi
const char* ssid = "Tzei";       
const char* password = "12345678";

// Cấu hình Firebase
#define API_KEY "AIzaSyC92NopzOiRYQAeuWdmS1s47btY7MQdCrI"
#define DATABASE_URL "https://hardware-nhung-default-rtdb.firebaseio.com/"
#define USER_EMAIL "thien@gmail.com"
#define USER_PASSWORD "123456"

// Khai báo Firebase
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

// Chân cảm biến
#define MQ7_PIN 34   
#define MQ135_PIN 32 

// Khởi tạo cảm biến MQ7
MQUnifiedsensor MQ7("ESP32", MQ7_PIN);  // 🔹 Sửa lại constructor

void setup() {
    Serial.begin(115200);
    WiFi.begin(ssid, password);

    Serial.print("Đang kết nối WiFi...");
    while (WiFi.status() != WL_CONNECTED) {
        delay(1000);
        Serial.print(".");
    }
    Serial.println("\n✅ Kết nối WiFi thành công!");
    Serial.print("📡 Địa chỉ IP: ");
    Serial.println(WiFi.localIP());

    // Cấu hình Firebase
    config.api_key = API_KEY;
    config.database_url = DATABASE_URL;
    auth.user.email = USER_EMAIL;
    auth.user.password = USER_PASSWORD;

    Firebase.begin(&config, &auth);
    Firebase.reconnectWiFi(true);

    Serial.println("⏳ Đang đăng nhập Firebase...");
    while (!Firebase.ready()) {
        Serial.print(".");
        delay(1000);
    }
    Serial.println("\n✅ Firebase đã kết nối thành công!");

    // Cấu hình MQ7
    MQ7.setRegressionMethod(1);
    MQ7.init();

    Serial.println("Calibrating MQ7... Hãy để cảm biến ngoài không khí sạch trong 20 giây!");
    delay(20000);
    MQ7.calibrate(27.5);  // 🔹 Thêm giá trị 27.5

    Serial.print("Giá trị R0: ");
    Serial.println(MQ7.getR0());
}

void loop() {
    int mq7_value = analogRead(MQ7_PIN);
    int mq135_value = analogRead(MQ135_PIN);    

    Serial.print("MQ-7 (ADC): ");
    Serial.print(mq7_value);
    Serial.print(" | MQ-135 (ADC): ");
    Serial.println(mq135_value);

    MQ7.update();
    float mq7_ppm = MQ7.readSensor();

    Serial.print("MQ7 (PPM): ");
    Serial.println(mq7_ppm);

    // Gửi lên Firebase
    bool success1 = Firebase.RTDB.setFloat(&fbdo, "sensor/Sensor01/mq7", mq7_value);
    bool success2 = Firebase.RTDB.setFloat(&fbdo, "sensor/Sensor01/mq135", mq135_value);

    if (success1 && success2) {
        Serial.println("✅ Dữ liệu đã gửi lên Firebase!");
    } else {
        Serial.print("❌ Lỗi Firebase: ");
        Serial.println(fbdo.errorReason());
    }

    delay(5000);
}
