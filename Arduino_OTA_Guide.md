# Hướng Dẫn Chi Tiết Về Thư Viện Arduino OTA

## 1. OTA (Over-The-Air) Là Gì?

**OTA (Over-The-Air)** là công nghệ cho phép cập nhật firmware cho thiết bị Arduino qua mạng WiFi mà không cần kết nối USB. Điều này đặc biệt hữu ích khi:
- Thiết bị được lắp đặt ở vị trí khó tiếp cận
- Cần cập nhật firmware từ xa
- Phát triển ứng dụng IoT cần cập nhật thường xuyên

## 2. Cách Hoạt Động Của Thư Viện ArduinoOTA

### 2.1. Kiến Trúc Tổng Quan

```
[Máy Tính]  <---WiFi--->  [ESP8266/ESP32]
   (IDE)                   (Thiết bị IoT)
     |                            |
     |---> Upload firmware ------>|
     |<--- Nhận phản hồi ---------|
```

### 2.2. Quy Trình Hoạt Động

#### Bước 1: Khởi Tạo (Initialization)
```cpp
#include <ArduinoOTA.h>

void setup() {
  // Kết nối WiFi trước
  WiFi.begin(ssid, password);

  // Cấu hình OTA
  ArduinoOTA.setHostname("ESP-Device");
  ArduinoOTA.setPassword("admin123");

  // Khởi động OTA
  ArduinoOTA.begin();
}
```

**Diễn giải:**
- Thư viện khởi tạo một **mDNS service** để thiết bị có thể được phát hiện trên mạng
- Mở một **TCP server** lắng nghe ở port 8266 (ESP8266) hoặc 3232 (ESP32)
- Đợi nhận lệnh upload từ máy tính

#### Bước 2: Phát Hiện Thiết Bị (Discovery)
Khi bạn mở Arduino IDE:
1. IDE quét mạng WiFi tìm các thiết bị OTA
2. Sử dụng **mDNS/Bonjour** để phát hiện thiết bị
3. Hiển thị danh sách thiết bị có thể upload trong menu Port

#### Bước 3: Xác Thực (Authentication)
```cpp
ArduinoOTA.setPassword("admin123");
// Hoặc sử dụng MD5 hash
ArduinoOTA.setPasswordHash("e8d95a51f3af4a3b134bf6bb680a213a");
```

- Khi upload, IDE gửi mật khẩu đến thiết bị
- Thiết bị so sánh mật khẩu nhận được với mật khẩu đã cấu hình
- Nếu khớp, cho phép tiếp tục; nếu không, từ chối kết nối

#### Bước 4: Upload Firmware
1. **Chuẩn bị:**
   - IDE biên dịch code thành file `.bin`
   - Thiết bị dừng các tác vụ hiện tại
   - Xóa vùng nhớ flash chuẩn bị ghi

2. **Truyền dữ liệu:**
   ```
   [IDE] ---> Gửi firmware theo từng packet
   [ESP] ---> Ghi vào flash memory
   [ESP] ---> Gửi ACK xác nhận
   ```

3. **Hoàn tất:**
   - Kiểm tra tính toàn vẹn của firmware (MD5 checksum)
   - Nếu OK: Khởi động lại và chạy firmware mới
   - Nếu lỗi: Giữ nguyên firmware cũ (failsafe)

### 2.3. Các Callback Quan Trọng

```cpp
ArduinoOTA.onStart([]() {
  String type;
  if (ArduinoOTA.getCommand() == U_FLASH) {
    type = "sketch";
  } else { // U_SPIFFS
    type = "filesystem";
  }
  Serial.println("Start updating " + type);
});

ArduinoOTA.onEnd([]() {
  Serial.println("\nEnd");
});

ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
  Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
});

ArduinoOTA.onError([](ota_error_t error) {
  Serial.printf("Error[%u]: ", error);
  if (error == OTA_AUTH_ERROR) {
    Serial.println("Auth Failed");
  } else if (error == OTA_BEGIN_ERROR) {
    Serial.println("Begin Failed");
  } else if (error == OTA_CONNECT_ERROR) {
    Serial.println("Connect Failed");
  } else if (error == OTA_RECEIVE_ERROR) {
    Serial.println("Receive Failed");
  } else if (error == OTA_END_ERROR) {
    Serial.println("End Failed");
  }
});
```

**Giải thích các callback:**
- `onStart()`: Được gọi khi bắt đầu quá trình update
- `onEnd()`: Được gọi khi hoàn tất update
- `onProgress()`: Cập nhật tiến trình upload (%)
- `onError()`: Xử lý các lỗi có thể xảy ra

### 2.4. Loop Handler

```cpp
void loop() {
  ArduinoOTA.handle();  // QUAN TRỌNG!

  // Code của bạn...
}
```

**Vai trò của `ArduinoOTA.handle()`:**
- Liên tục kiểm tra xem có yêu cầu OTA nào không
- Xử lý các gói tin TCP đến
- Quản lý state machine của OTA
- **Phải được gọi thường xuyên** trong loop() để OTA hoạt động

## 3. Cơ Chế Bảo Vệ (Protection Mechanisms)

### 3.1. Rollback Protection
```cpp
// Firmware mới luôn được ghi vào vùng khác
// Nếu lỗi, ESP tự động boot lại firmware cũ
```

### 3.2. MD5 Checksum
- Mỗi firmware có MD5 hash
- Sau khi upload, ESP tính toán lại MD5
- So sánh với MD5 gốc để đảm bảo không bị lỗi trong quá trình truyền

### 3.3. Watchdog Timer
```cpp
// Tự động reset nếu update bị treo
ESP.wdtDisable();  // Tắt trong quá trình update
ESP.wdtEnable(8000);  // Bật lại sau khi xong
```

## 4. Ví Dụ Code Hoàn Chỉnh

```cpp
#include <ESP8266WiFi.h>
#include <ESP8266mDNS.h>
#include <WiFiUdp.h>
#include <ArduinoOTA.h>

const char* ssid = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";

void setup() {
  Serial.begin(115200);
  Serial.println("Booting");

  // Kết nối WiFi
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED) {
    Serial.println("Connection Failed! Rebooting...");
    delay(5000);
    ESP.restart();
  }

  // Cấu hình OTA
  ArduinoOTA.setHostname("ESP8266-OTA");
  ArduinoOTA.setPassword("admin");

  ArduinoOTA.onStart([]() {
    String type;
    if (ArduinoOTA.getCommand() == U_FLASH) {
      type = "sketch";
    } else {
      type = "filesystem";
    }
    Serial.println("Start updating " + type);
  });

  ArduinoOTA.onEnd([]() {
    Serial.println("\nEnd");
  });

  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });

  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) {
      Serial.println("Auth Failed");
    } else if (error == OTA_BEGIN_ERROR) {
      Serial.println("Begin Failed");
    } else if (error == OTA_CONNECT_ERROR) {
      Serial.println("Connect Failed");
    } else if (error == OTA_RECEIVE_ERROR) {
      Serial.println("Receive Failed");
    } else if (error == OTA_END_ERROR) {
      Serial.println("End Failed");
    }
  });

  ArduinoOTA.begin();

  Serial.println("Ready");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  ArduinoOTA.handle();

  // Thêm code ứng dụng của bạn ở đây
}
```

## 5. Sơ Đồ State Machine

```
[IDLE] ---> Nhận yêu cầu update ---> [AUTHENTICATING]
                                            |
                                            v
                                    [AUTHENTICATED]
                                            |
                                            v
                                    [RECEIVING DATA]
                                            |
                                            v
                                    [VERIFYING MD5]
                                            |
                                            v
                                    [REBOOTING]
                                            |
                                            v
                                    [RUNNING NEW FW]
```

## 6. Best Practices

### 6.1. Bảo Mật
```cpp
// Luôn dùng mật khẩu
ArduinoOTA.setPassword("strong_password_here");

// Chỉ cho phép OTA từ IP cụ thể
IPAddress allowedIP(192, 168, 1, 100);
// Kiểm tra trong onStart callback
```

### 6.2. Xử Lý Lỗi
```cpp
// Luôn có failsafe
if (updateFailed) {
  ESP.restart();  // Quay lại firmware cũ
}
```

### 6.3. Quản Lý Tài Nguyên
```cpp
ArduinoOTA.onStart([]() {
  // Dừng tất cả tác vụ đang chạy
  timer.detach();
  server.close();
  // Giải phóng bộ nhớ
});
```

## 7. Troubleshooting

### Lỗi thường gặp:

1. **"No ESP found"**
   - Kiểm tra cùng mạng WiFi với máy tính
   - Firewall có thể chặn mDNS port 5353

2. **"Auth Failed"**
   - Sai mật khẩu OTA
   - Kiểm tra `setPassword()`

3. **"Update Failed"**
   - Không đủ bộ nhớ flash
   - Code quá lớn
   - `ArduinoOTA.handle()` không được gọi đủ thường xuyên

4. **Upload chậm**
   - Tín hiệu WiFi yếu
   - Nhiều thiết bị cùng lúc
   - Router quá tải

## 8. Tóm Tắt Luồng Hoạt Động

1. **Setup Phase:**
   - ESP kết nối WiFi
   - Khởi tạo mDNS service với hostname
   - Mở TCP server lắng nghe

2. **Discovery Phase:**
   - IDE broadcast mDNS query
   - ESP phản hồi với thông tin (IP, port, hostname)

3. **Upload Phase:**
   - IDE kết nối TCP đến ESP
   - Xác thực password
   - Upload firmware theo chunks
   - ESP ghi vào flash và xác nhận từng chunk

4. **Verification Phase:**
   - Tính MD5 checksum
   - So sánh với firmware gốc
   - Nếu OK: commit update, reboot
   - Nếu fail: rollback, giữ firmware cũ

5. **Restart Phase:**
   - ESP khởi động lại
   - Boot từ firmware mới
   - OTA service lại sẵn sàng cho lần update tiếp theo

---

## 9. Tài Nguyên Tham Khảo

- ESP8266 Arduino Core: https://github.com/esp8266/Arduino
- ESP32 Arduino Core: https://github.com/espressif/arduino-esp32
- ArduinoOTA Documentation: https://arduino-esp8266.readthedocs.io/en/latest/ota_updates/readme.html

**Lưu ý:** Thư viện ArduinoOTA chỉ hoạt động với ESP8266 và ESP32, không hỗ trợ các board Arduino thông thường (Uno, Mega, Nano) vì chúng không có WiFi tích hợp.
