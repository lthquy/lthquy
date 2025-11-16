# Hướng dẫn Watchdog Timer trong ESP32

## 1. Watchdog Timer là gì?

Watchdog Timer (WDT) là một cơ chế bảo vệ phần cứng giúp phát hiện và khôi phục hệ thống khi chương trình bị treo hoặc rơi vào vòng lặp vô hạn. Nếu WDT không được "cho ăn" (feed/reset) trong khoảng thời gian nhất định, nó sẽ tự động reset ESP32.

## 2. Các loại Watchdog trong ESP32

ESP32 có 3 loại Watchdog Timer:

### 2.1. Task Watchdog Timer (TWDT)
- Theo dõi các task cụ thể trong FreeRTOS
- Đảm bảo các task quan trọng không bị treo
- Có thể cấu hình cho từng task riêng lẻ

### 2.2. Interrupt Watchdog Timer (IWDT)
- Theo dõi CPU để phát hiện khi interrupt bị disable quá lâu
- Bảo vệ hệ thống khỏi các lỗi ở cấp độ thấp

### 2.3. Hardware Watchdog Timer (HWDT)
- Watchdog ở cấp độ phần cứng
- Reset toàn bộ chip khi timeout

## 3. Cách hoạt động của Watchdog

```
┌─────────────────┐
│  Khởi động WDT  │
│  (timeout: 5s)  │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Chạy code      │◄──────┐
│  bình thường    │       │
└────────┬────────┘       │
         │                │
         v                │
┌─────────────────┐       │
│  Feed WDT       │───────┘
│  (reset timer)  │
└─────────────────┘

Nếu không Feed WDT trong 5s → ESP32 RESET
```

## 4. Mẫu Code Arduino cho ESP32

### 4.1. Ví dụ cơ bản - Task Watchdog

```cpp
#include <esp_task_wdt.h>

// Thời gian timeout (giây)
#define WDT_TIMEOUT 5

void setup() {
  Serial.begin(115200);
  Serial.println("Khởi động ESP32 Watchdog Demo");

  // Khởi tạo Task Watchdog với timeout 5 giây
  esp_task_wdt_init(WDT_TIMEOUT, true); // true = tự động reset khi timeout

  // Đăng ký task hiện tại với watchdog
  esp_task_wdt_add(NULL); // NULL = current task

  Serial.println("Watchdog đã được kích hoạt!");
}

void loop() {
  // Reset watchdog timer để tránh reset
  esp_task_wdt_reset();

  Serial.println("Hệ thống hoạt động bình thường...");

  // Code thực thi bình thường
  delay(1000); // Delay nhỏ hơn WDT_TIMEOUT (5s)
}
```

### 4.2. Ví dụ mô phỏng treo hệ thống

```cpp
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 3

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println("\n=== ESP32 Watchdog Test - System Hang ===");

  // Khởi tạo watchdog
  esp_task_wdt_init(WDT_TIMEOUT, true);
  esp_task_wdt_add(NULL);

  Serial.println("Watchdog kích hoạt (timeout: 3s)");
}

int counter = 0;

void loop() {
  counter++;

  if (counter < 5) {
    // 5 lần đầu: feed watchdog bình thường
    esp_task_wdt_reset();
    Serial.printf("Loop %d - Watchdog được feed\n", counter);
    delay(1000);
  }
  else {
    // Sau đó: GIẢ LẬP TREO - không feed watchdog
    Serial.printf("Loop %d - KHÔNG feed watchdog! Hệ thống sẽ reset sau 3s...\n", counter);
    delay(1000); // Chờ để watchdog timeout
    // ESP32 sẽ tự động reset
  }
}
```

### 4.3. Ví dụ nâng cao - Nhiều task

```cpp
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 5

TaskHandle_t Task1;
TaskHandle_t Task2;

// Task 1: Đọc cảm biến
void task1Code(void * parameter) {
  // Đăng ký task này với watchdog
  esp_task_wdt_add(NULL);

  for(;;) {
    Serial.println("Task 1: Đọc cảm biến...");

    // Giả lập đọc cảm biến
    delay(2000);

    // Feed watchdog cho task này
    esp_task_wdt_reset();
  }
}

// Task 2: Gửi dữ liệu
void task2Code(void * parameter) {
  // Đăng ký task này với watchdog
  esp_task_wdt_add(NULL);

  for(;;) {
    Serial.println("Task 2: Gửi dữ liệu...");

    // Giả lập gửi dữ liệu
    delay(3000);

    // Feed watchdog cho task này
    esp_task_wdt_reset();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println("=== Multi-task Watchdog Demo ===");

  // Khởi tạo watchdog
  esp_task_wdt_init(WDT_TIMEOUT, true);

  // Tạo task 1 trên core 0
  xTaskCreatePinnedToCore(
    task1Code,   // Function
    "Task1",     // Name
    10000,       // Stack size
    NULL,        // Parameters
    1,           // Priority
    &Task1,      // Handle
    0            // Core
  );

  // Tạo task 2 trên core 1
  xTaskCreatePinnedToCore(
    task2Code,
    "Task2",
    10000,
    NULL,
    1,
    &Task2,
    1
  );
}

void loop() {
  // Loop chính không làm gì
  delay(10000);
}
```

### 4.4. Ví dụ với vòng lặp While dài

```cpp
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 10

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println("=== Watchdog với xử lý dữ liệu lớn ===");

  esp_task_wdt_init(WDT_TIMEOUT, true);
  esp_task_wdt_add(NULL);
}

void loop() {
  Serial.println("Bắt đầu xử lý dữ liệu lớn...");

  // Giả lập xử lý 1000 items
  for(int i = 0; i < 1000; i++) {
    // Xử lý mỗi item
    processItem(i);

    // Feed watchdog mỗi 100 items để tránh timeout
    if(i % 100 == 0) {
      esp_task_wdt_reset();
      Serial.printf("Đã xử lý %d items, feed watchdog\n", i);
    }
  }

  Serial.println("Hoàn thành xử lý!");
  delay(2000);
}

void processItem(int item) {
  // Giả lập xử lý mất thời gian
  delayMicroseconds(100);
}
```

### 4.5. Vô hiệu hóa Watchdog tạm thời

```cpp
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 5

void setup() {
  Serial.begin(115200);
  delay(1000);

  // Khởi tạo watchdog
  esp_task_wdt_init(WDT_TIMEOUT, true);
  esp_task_wdt_add(NULL);

  Serial.println("Watchdog đã kích hoạt");
}

void loop() {
  // Hoạt động bình thường
  esp_task_wdt_reset();
  Serial.println("Hoạt động bình thường...");
  delay(2000);

  // Cần thực hiện tác vụ mất nhiều thời gian
  Serial.println("Chuẩn bị OTA update - tạm thời disable watchdog");

  // Gỡ task khỏi watchdog
  esp_task_wdt_delete(NULL);

  // Thực hiện tác vụ dài (ví dụ: OTA update)
  performLongOperation();

  // Đăng ký lại với watchdog
  esp_task_wdt_add(NULL);
  Serial.println("Watchdog đã được kích hoạt lại");
}

void performLongOperation() {
  Serial.println("Đang thực hiện tác vụ dài...");
  delay(10000); // 10 giây - vượt quá timeout
  Serial.println("Hoàn thành tác vụ dài");
}
```

## 5. Các hàm API quan trọng

### 5.1. Khởi tạo
```cpp
esp_task_wdt_init(timeout_s, panic_on_timeout)
```
- `timeout_s`: Thời gian timeout (giây)
- `panic_on_timeout`: true = reset khi timeout, false = chỉ in warning

### 5.2. Đăng ký task
```cpp
esp_task_wdt_add(handle)
```
- `NULL`: Task hiện tại
- `TaskHandle`: Task cụ thể

### 5.3. Reset watchdog
```cpp
esp_task_wdt_reset()
```
- Gọi hàm này để "cho ăn" watchdog, tránh reset

### 5.4. Gỡ task khỏi watchdog
```cpp
esp_task_wdt_delete(handle)
```

### 5.5. Deinit watchdog
```cpp
esp_task_wdt_deinit()
```
- Tắt hoàn toàn watchdog

## 6. Lưu ý khi sử dụng

### 6.1. Chọn timeout phù hợp
- Quá ngắn: Hệ thống có thể reset nhầm
- Quá dài: Không phát hiện kịp khi treo
- **Khuyến nghị**: 3-10 giây tùy ứng dụng

### 6.2. Feed watchdog đều đặn
```cpp
void loop() {
  esp_task_wdt_reset(); // Feed ở đầu loop

  // Code của bạn
  doSomething();

  // Nếu code dài, feed thêm
  esp_task_wdt_reset();
}
```

### 6.3. Xử lý tác vụ dài
- **Cách 1**: Feed watchdog định kỳ trong vòng lặp
- **Cách 2**: Tăng timeout
- **Cách 3**: Tạm thời disable watchdog

### 6.4. Debug
```cpp
// Khi develop, có thể tắt auto-reset để debug
esp_task_wdt_init(WDT_TIMEOUT, false); // false = chỉ warning
```

## 7. Ví dụ thực tế

### 7.1. IoT Device với WiFi

```cpp
#include <WiFi.h>
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 30 // 30s cho kết nối WiFi

const char* ssid = "YOUR_SSID";
const char* password = "YOUR_PASSWORD";

void setup() {
  Serial.begin(115200);

  // Khởi tạo watchdog với timeout 30s
  esp_task_wdt_init(WDT_TIMEOUT, true);
  esp_task_wdt_add(NULL);

  // Kết nối WiFi
  connectWiFi();

  // Sau khi kết nối xong, giảm timeout xuống 10s
  esp_task_wdt_deinit();
  esp_task_wdt_init(10, true);
  esp_task_wdt_add(NULL);
}

void connectWiFi() {
  Serial.println("Kết nối WiFi...");
  WiFi.begin(ssid, password);

  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;

    // Feed watchdog mỗi 2.5s (5 lần x 500ms)
    if(attempts % 5 == 0) {
      esp_task_wdt_reset();
    }
  }

  if(WiFi.status() == WL_CONNECTED) {
    Serial.println("\nĐã kết nối WiFi!");
  } else {
    Serial.println("\nKhông thể kết nối WiFi, reset...");
    ESP.restart();
  }
}

void loop() {
  esp_task_wdt_reset();

  // Kiểm tra WiFi
  if(WiFi.status() != WL_CONNECTED) {
    Serial.println("Mất kết nối WiFi, thử kết nối lại...");
    connectWiFi();
  }

  // Công việc chính
  readSensorsAndSend();

  delay(5000);
}

void readSensorsAndSend() {
  Serial.println("Đọc cảm biến và gửi dữ liệu...");
  // Code của bạn ở đây
}
```

### 7.2. Data Logger với SD Card

```cpp
#include <SD.h>
#include <esp_task_wdt.h>

#define WDT_TIMEOUT 15
#define SD_CS_PIN 5

void setup() {
  Serial.begin(115200);

  esp_task_wdt_init(WDT_TIMEOUT, true);
  esp_task_wdt_add(NULL);

  if(!SD.begin(SD_CS_PIN)) {
    Serial.println("Lỗi SD Card!");
    return;
  }
}

void loop() {
  esp_task_wdt_reset();

  // Đọc dữ liệu
  float temperature = readTemperature();
  float humidity = readHumidity();

  // Feed watchdog trước khi ghi SD (có thể mất thời gian)
  esp_task_wdt_reset();

  // Ghi vào SD card
  logToSD(temperature, humidity);

  delay(10000);
}

float readTemperature() {
  // Giả lập đọc cảm biến
  delay(100);
  return random(200, 350) / 10.0;
}

float readHumidity() {
  delay(100);
  return random(300, 800) / 10.0;
}

void logToSD(float temp, float hum) {
  File dataFile = SD.open("/datalog.txt", FILE_APPEND);

  if(dataFile) {
    String dataString = String(millis()) + "," +
                       String(temp) + "," +
                       String(hum);
    dataFile.println(dataString);
    dataFile.close();
    Serial.println("Đã ghi: " + dataString);
  } else {
    Serial.println("Lỗi mở file!");
  }
}
```

## 8. Troubleshooting

### 8.1. ESP32 reset liên tục
**Nguyên nhân**: Timeout quá ngắn hoặc code chặn quá lâu

**Giải pháp**:
```cpp
// Tăng timeout
esp_task_wdt_init(20, true); // Tăng lên 20s

// Hoặc feed thường xuyên hơn
esp_task_wdt_reset();
```

### 8.2. Watchdog không hoạt động
**Kiểm tra**:
```cpp
// Đảm bảo đã add task
esp_task_wdt_add(NULL);

// Kiểm tra panic_on_timeout = true
esp_task_wdt_init(WDT_TIMEOUT, true);
```

### 8.3. Task bị reset nhưng task khác không
Mỗi task phải tự feed watchdog của mình:
```cpp
void myTask(void* param) {
  esp_task_wdt_add(NULL); // Quan trọng!

  for(;;) {
    // Do work
    esp_task_wdt_reset(); // Feed định kỳ
    delay(1000);
  }
}
```

## 9. Kết luận

Watchdog Timer là công cụ quan trọng để tăng độ tin cậy cho ESP32:

- **Bắt buộc** cho thiết bị IoT hoạt động 24/7
- **Nên dùng** cho các ứng dụng quan trọng
- **Cấu hình đúng** timeout và feed định kỳ
- **Test kỹ** trước khi deploy

## 10. Tham khảo

- ESP-IDF Documentation: https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/wdts.html
- Arduino ESP32: https://github.com/espressif/arduino-esp32
