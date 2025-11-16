# Hướng dẫn Watchdog Timer ESP32 với ESP-IDF v5.x

## 1. Giới thiệu Watchdog Timer

Watchdog Timer (WDT) là cơ chế bảo vệ phần cứng tự động reset ESP32 khi phát hiện chương trình bị treo hoặc rơi vào vòng lặp vô hạn. Nếu không được "feed" định kỳ, watchdog sẽ reset hệ thống.

## 2. Các loại Watchdog trong ESP32

### 2.1. Task Watchdog Timer (TWDT)
- Theo dõi các task cụ thể trong FreeRTOS
- Có thể subscribe/unsubscribe task động
- Hỗ trợ cả task và user function

### 2.2. Interrupt Watchdog Timer (IWDT)
- Theo dõi CPU interrupt
- Phát hiện khi interrupt bị block quá lâu
- Hoạt động ở cấp độ thấp

### 2.3. Hardware Watchdog Timer
- Watchdog cấp phần cứng
- Reset toàn bộ chip khi timeout

## 3. API ESP-IDF v5.x - Thay đổi quan trọng

### 3.1. Cấu trúc Config mới

```c
typedef struct {
    uint32_t timeout_ms;        // Timeout tính bằng milliseconds
    uint32_t idle_core_mask;    // Bitmask core idle task cần monitor
    bool trigger_panic;         // true = reset khi timeout
} esp_task_wdt_config_t;
```

### 3.2. So sánh API cũ vs mới

| ESP-IDF v4.x (Cũ) | ESP-IDF v5.x (Mới) |
|-------------------|-------------------|
| `esp_task_wdt_init(timeout, panic)` | `esp_task_wdt_init(&config)` |
| `esp_task_wdt_add(NULL)` | `esp_task_wdt_add(NULL)` (không đổi) |
| `esp_task_wdt_reset()` | `esp_task_wdt_reset()` (không đổi) |
| `esp_task_wdt_delete(NULL)` | `esp_task_wdt_delete(NULL)` (không đổi) |
| Không có | `esp_task_wdt_reconfigure(&config)` (mới) |
| Không có | `esp_task_wdt_add_user()` (mới) |
| Không có | `esp_task_wdt_reset_user()` (mới) |

## 4. Các hàm API chi tiết

### 4.1. Khởi tạo và cấu hình

```c
// Khởi tạo TWDT
esp_err_t esp_task_wdt_init(const esp_task_wdt_config_t *config);

// Thay đổi cấu hình khi đang chạy
esp_err_t esp_task_wdt_reconfigure(const esp_task_wdt_config_t *config);

// Tắt TWDT
esp_err_t esp_task_wdt_deinit(void);
```

### 4.2. Quản lý Task

```c
// Subscribe task vào watchdog
esp_err_t esp_task_wdt_add(TaskHandle_t task_handle);
// task_handle = NULL: task hiện tại
// task_handle = xTaskHandle: task cụ thể

// Reset watchdog cho task hiện tại
esp_err_t esp_task_wdt_reset(void);

// Unsubscribe task khỏi watchdog
esp_err_t esp_task_wdt_delete(TaskHandle_t task_handle);
```

### 4.3. Quản lý User Function

```c
// Subscribe user function với tên
esp_err_t esp_task_wdt_add_user(const char *user_name,
                                 esp_task_wdt_user_handle_t *handle);

// Reset watchdog cho user cụ thể
esp_err_t esp_task_wdt_reset_user(esp_task_wdt_user_handle_t handle);

// Unsubscribe user
esp_err_t esp_task_wdt_delete_user(esp_task_wdt_user_handle_t handle);
```

## 5. Ví dụ code ESP-IDF v5.x

### 5.1. Ví dụ cơ bản

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WATCHDOG_BASIC";

void app_main(void)
{
    ESP_LOGI(TAG, "Khởi động ESP32 Watchdog Demo");

    // Cấu hình watchdog
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 5000,                              // 5 giây
        .idle_core_mask = (1 << portNUM_PROCESSORS) - 1, // Tất cả cores
        .trigger_panic = true                             // Reset khi timeout
    };

    // Khởi tạo watchdog
    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));

    // Subscribe task hiện tại
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    ESP_LOGI(TAG, "Watchdog đã được kích hoạt (timeout: 5s)");

    // Main loop
    while (1) {
        // Reset watchdog
        ESP_ERROR_CHECK(esp_task_wdt_reset());

        ESP_LOGI(TAG, "Hệ thống hoạt động bình thường...");

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 5.2. Mô phỏng treo hệ thống

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WDT_HANG_TEST";

void app_main(void)
{
    ESP_LOGI(TAG, "=== Test Watchdog - Giả lập treo ===");

    // Cấu hình watchdog timeout 3 giây
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 3000,
        .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    int counter = 0;

    while (1) {
        counter++;

        if (counter < 5) {
            // 5 lần đầu: hoạt động bình thường
            ESP_ERROR_CHECK(esp_task_wdt_reset());
            ESP_LOGI(TAG, "Loop %d - Watchdog được feed", counter);
        } else {
            // Sau đó: KHÔNG feed watchdog
            ESP_LOGW(TAG, "Loop %d - KHÔNG feed! Hệ thống sẽ reset sau 3s...",
                     counter);
            // ESP32 sẽ tự động reset sau 3 giây
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 5.3. Multi-task với Watchdog

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WDT_MULTITASK";

// Task 1: Đọc cảm biến
void sensor_task(void *pvParameters)
{
    ESP_LOGI(TAG, "Sensor task bắt đầu");

    // Subscribe task này vào watchdog
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    while (1) {
        ESP_LOGI(TAG, "Task 1: Đọc cảm biến...");

        // Giả lập đọc cảm biến
        vTaskDelay(pdMS_TO_TICKS(2000));

        // Feed watchdog cho task này
        ESP_ERROR_CHECK(esp_task_wdt_reset());
    }
}

// Task 2: Xử lý dữ liệu
void process_task(void *pvParameters)
{
    ESP_LOGI(TAG, "Process task bắt đầu");

    // Subscribe task này vào watchdog
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    while (1) {
        ESP_LOGI(TAG, "Task 2: Xử lý dữ liệu...");

        // Giả lập xử lý
        vTaskDelay(pdMS_TO_TICKS(3000));

        // Feed watchdog cho task này
        ESP_ERROR_CHECK(esp_task_wdt_reset());
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "=== Multi-task Watchdog Demo ===");

    // Khởi tạo watchdog với timeout 5 giây
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 5000,
        .idle_core_mask = 0,  // Không monitor idle task
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));

    // Tạo task 1 trên core 0
    xTaskCreatePinnedToCore(
        sensor_task,
        "SensorTask",
        4096,
        NULL,
        5,
        NULL,
        0
    );

    // Tạo task 2 trên core 1
    xTaskCreatePinnedToCore(
        process_task,
        "ProcessTask",
        4096,
        NULL,
        5,
        NULL,
        1
    );

    ESP_LOGI(TAG, "Đã tạo 2 task với watchdog monitoring");
}
```

### 5.4. Sử dụng User Handle

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WDT_USER_HANDLE";

// Handle cho user function
esp_task_wdt_user_handle_t user_handle_1 = NULL;
esp_task_wdt_user_handle_t user_handle_2 = NULL;

void critical_function_1(void)
{
    ESP_LOGI(TAG, "Critical Function 1 đang chạy...");
    vTaskDelay(pdMS_TO_TICKS(500));

    // Reset watchdog cho function này
    if (user_handle_1 != NULL) {
        ESP_ERROR_CHECK(esp_task_wdt_reset_user(user_handle_1));
        ESP_LOGI(TAG, "Function 1 đã feed watchdog");
    }
}

void critical_function_2(void)
{
    ESP_LOGI(TAG, "Critical Function 2 đang chạy...");
    vTaskDelay(pdMS_TO_TICKS(800));

    // Reset watchdog cho function này
    if (user_handle_2 != NULL) {
        ESP_ERROR_CHECK(esp_task_wdt_reset_user(user_handle_2));
        ESP_LOGI(TAG, "Function 2 đã feed watchdog");
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "=== User Handle Watchdog Demo ===");

    // Cấu hình watchdog
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 3000,
        .idle_core_mask = 0,
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));

    // Đăng ký user function 1
    ESP_ERROR_CHECK(esp_task_wdt_add_user("CriticalFunc1", &user_handle_1));
    ESP_LOGI(TAG, "Đã đăng ký Critical Function 1");

    // Đăng ký user function 2
    ESP_ERROR_CHECK(esp_task_wdt_add_user("CriticalFunc2", &user_handle_2));
    ESP_LOGI(TAG, "Đã đăng ký Critical Function 2");

    // Main loop
    while (1) {
        critical_function_1();
        critical_function_2();

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 5.5. Reconfigure Watchdog động

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WDT_RECONFIG";

void app_main(void)
{
    ESP_LOGI(TAG, "=== Reconfigure Watchdog Demo ===");

    // Cấu hình ban đầu: timeout 5s
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 5000,
        .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    ESP_LOGI(TAG, "Watchdog khởi tạo với timeout: 5000ms");

    int counter = 0;

    while (1) {
        counter++;
        ESP_ERROR_CHECK(esp_task_wdt_reset());

        if (counter == 5) {
            // Sau 5 lần loop, thay đổi timeout thành 10s
            ESP_LOGI(TAG, "Thay đổi timeout từ 5s -> 10s");

            wdt_config.timeout_ms = 10000;
            ESP_ERROR_CHECK(esp_task_wdt_reconfigure(&wdt_config));

            ESP_LOGI(TAG, "Đã reconfigure watchdog timeout: 10000ms");
        }

        if (counter == 10) {
            // Sau 10 lần, giảm xuống 3s
            ESP_LOGI(TAG, "Thay đổi timeout từ 10s -> 3s");

            wdt_config.timeout_ms = 3000;
            ESP_ERROR_CHECK(esp_task_wdt_reconfigure(&wdt_config));

            ESP_LOGI(TAG, "Đã reconfigure watchdog timeout: 3000ms");
        }

        ESP_LOGI(TAG, "Loop %d", counter);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

### 5.6. Xử lý vòng lặp dài

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WDT_LONG_LOOP";

#define TOTAL_ITEMS 10000
#define FEED_INTERVAL 500  // Feed mỗi 500 items

void process_item(int index)
{
    // Giả lập xử lý mất thời gian
    for (volatile int i = 0; i < 1000; i++);
}

void app_main(void)
{
    ESP_LOGI(TAG, "=== Long Loop Watchdog Demo ===");

    // Timeout 10 giây
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 10000,
        .idle_core_mask = 0,
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    ESP_LOGI(TAG, "Bắt đầu xử lý %d items...", TOTAL_ITEMS);

    int64_t start_time = esp_timer_get_time();

    for (int i = 0; i < TOTAL_ITEMS; i++) {
        process_item(i);

        // Feed watchdog mỗi FEED_INTERVAL items
        if (i % FEED_INTERVAL == 0) {
            ESP_ERROR_CHECK(esp_task_wdt_reset());
            ESP_LOGI(TAG, "Đã xử lý %d/%d items, feed watchdog",
                     i, TOTAL_ITEMS);
        }
    }

    int64_t end_time = esp_timer_get_time();
    float elapsed_sec = (end_time - start_time) / 1000000.0f;

    ESP_LOGI(TAG, "Hoàn thành xử lý %d items trong %.2f giây",
             TOTAL_ITEMS, elapsed_sec);

    // Unsubscribe và deinit
    ESP_ERROR_CHECK(esp_task_wdt_delete(NULL));
    ESP_ERROR_CHECK(esp_task_wdt_deinit());

    ESP_LOGI(TAG, "Watchdog đã được tắt");
}
```

### 5.7. WiFi IoT Device với Watchdog

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "esp_task_wdt.h"
#include "nvs_flash.h"

static const char *TAG = "WDT_WIFI";

#define WIFI_SSID      "YOUR_SSID"
#define WIFI_PASS      "YOUR_PASSWORD"
#define WDT_TIMEOUT_MS 30000  // 30 giây cho kết nối WiFi

static EventGroupHandle_t s_wifi_event_group;
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

static int s_retry_num = 0;
#define MAX_RETRY 5

static void event_handler(void* arg, esp_event_base_t event_base,
                         int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT &&
               event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < MAX_RETRY) {
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "Thử kết nối lại WiFi... (%d/%d)",
                     s_retry_num, MAX_RETRY);
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "Đã có IP:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT,
                                                ESP_EVENT_ANY_ID,
                                                &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT,
                                                IP_EVENT_STA_GOT_IP,
                                                &event_handler, NULL));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = WIFI_SSID,
            .password = WIFI_PASS,
        },
    };

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "Bắt đầu kết nối WiFi...");
}

void app_main(void)
{
    ESP_LOGI(TAG, "=== WiFi IoT with Watchdog ===");

    // Khởi tạo NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
        ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // Khởi tạo watchdog với timeout dài cho kết nối WiFi
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = WDT_TIMEOUT_MS,
        .idle_core_mask = 0,
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    ESP_LOGI(TAG, "Watchdog khởi tạo (timeout: %d ms)", WDT_TIMEOUT_MS);

    // Kết nối WiFi
    wifi_init_sta();

    // Đợi kết nối WiFi
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
                                           WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
                                           pdFALSE,
                                           pdFALSE,
                                           portMAX_DELAY);

    // Feed watchdog sau khi kết nối
    ESP_ERROR_CHECK(esp_task_wdt_reset());

    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "Đã kết nối WiFi thành công");

        // Giảm timeout xuống 10s cho hoạt động bình thường
        wdt_config.timeout_ms = 10000;
        ESP_ERROR_CHECK(esp_task_wdt_reconfigure(&wdt_config));
        ESP_LOGI(TAG, "Đã giảm watchdog timeout xuống 10s");

    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGE(TAG, "Không thể kết nối WiFi, reset...");
        vTaskDelay(pdMS_TO_TICKS(1000));
        esp_restart();
    }

    // Main loop
    while (1) {
        ESP_ERROR_CHECK(esp_task_wdt_reset());

        // Kiểm tra WiFi
        wifi_ap_record_t ap_info;
        if (esp_wifi_sta_get_ap_info(&ap_info) == ESP_OK) {
            ESP_LOGI(TAG, "WiFi OK - RSSI: %d", ap_info.rssi);
        } else {
            ESP_LOGW(TAG, "WiFi mất kết nối");
        }

        // Gửi dữ liệu IoT của bạn ở đây
        // send_sensor_data();

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

### 5.8. Vô hiệu hóa tạm thời

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "WDT_TEMP_DISABLE";

void long_operation(void)
{
    ESP_LOGI(TAG, "Bắt đầu tác vụ dài (15s)...");

    for (int i = 0; i < 15; i++) {
        ESP_LOGI(TAG, "Đang xử lý... %d/15", i + 1);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }

    ESP_LOGI(TAG, "Hoàn thành tác vụ dài!");
}

void app_main(void)
{
    ESP_LOGI(TAG, "=== Temporary Disable Watchdog Demo ===");

    // Khởi tạo watchdog timeout 5s
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 5000,
        .idle_core_mask = 0,
        .trigger_panic = true
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    ESP_LOGI(TAG, "Watchdog hoạt động (timeout: 5s)");

    // Loop bình thường
    for (int i = 0; i < 3; i++) {
        ESP_ERROR_CHECK(esp_task_wdt_reset());
        ESP_LOGI(TAG, "Hoạt động bình thường... %d", i);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }

    // Chuẩn bị thực hiện tác vụ dài
    ESP_LOGW(TAG, "Chuẩn bị thực hiện tác vụ dài - unsubscribe watchdog");

    // Unsubscribe task khỏi watchdog
    ESP_ERROR_CHECK(esp_task_wdt_delete(NULL));

    // Thực hiện tác vụ mất nhiều thời gian (15s > 5s timeout)
    long_operation();

    // Subscribe lại vào watchdog
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));
    ESP_LOGI(TAG, "Đã subscribe lại watchdog");

    // Tiếp tục hoạt động bình thường
    while (1) {
        ESP_ERROR_CHECK(esp_task_wdt_reset());
        ESP_LOGI(TAG, "Hoạt động bình thường...");
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

## 6. Cấu hình trong menuconfig

Để cấu hình watchdog qua menuconfig:

```bash
idf.py menuconfig
```

Đi đến: `Component config → ESP System Settings → Task Watchdog`

Các tùy chọn:
- `Initialize Task Watchdog Timer on startup`: Auto khởi tạo WDT
- `Invoke panic handler on Task Watchdog timeout`: Panic khi timeout
- `Task Watchdog timeout period (seconds)`: Thời gian timeout

## 7. Best Practices

### 7.1. Chọn timeout phù hợp

```c
// Kết nối WiFi: 20-30s
wdt_config.timeout_ms = 30000;

// Xử lý thông thường: 5-10s
wdt_config.timeout_ms = 10000;

// Real-time task: 1-3s
wdt_config.timeout_ms = 3000;
```

### 7.2. Feed watchdog đúng vị trí

```c
void task_function(void *param)
{
    esp_task_wdt_add(NULL);

    while (1) {
        // Feed ở đầu loop (khuyến nghị)
        esp_task_wdt_reset();

        // Thực hiện công việc
        do_work();

        // Nếu công việc dài, feed thêm lần nữa
        esp_task_wdt_reset();

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 7.3. Xử lý error code

```c
esp_err_t ret = esp_task_wdt_add(NULL);
if (ret == ESP_OK) {
    ESP_LOGI(TAG, "Task đã subscribe watchdog");
} else if (ret == ESP_ERR_INVALID_STATE) {
    ESP_LOGE(TAG, "Watchdog chưa được khởi tạo");
} else {
    ESP_LOGE(TAG, "Lỗi subscribe watchdog: %s", esp_err_to_name(ret));
}
```

### 7.4. Cleanup đúng cách

```c
void cleanup_watchdog(void)
{
    // Xóa tất cả user handles trước
    if (user_handle != NULL) {
        esp_task_wdt_delete_user(user_handle);
        user_handle = NULL;
    }

    // Unsubscribe task
    esp_task_wdt_delete(NULL);

    // Deinit watchdog
    esp_task_wdt_deinit();

    ESP_LOGI(TAG, "Watchdog đã được cleanup hoàn toàn");
}
```

## 8. Debugging Watchdog

### 8.1. Tắt panic để debug

```c
// Trong development, tắt panic để chỉ nhận warning
esp_task_wdt_config_t wdt_config = {
    .timeout_ms = 5000,
    .idle_core_mask = 0,
    .trigger_panic = false  // Chỉ warning, không reset
};
```

### 8.2. Log để tracking

```c
#define LOG_WDT_RESET() \
    ESP_LOGI(TAG, "[%s:%d] Watchdog reset", __func__, __LINE__)

void my_function(void)
{
    LOG_WDT_RESET();
    esp_task_wdt_reset();

    // Code của bạn
}
```

### 8.3. Monitor idle task

```c
// Monitor idle task của core 0
wdt_config.idle_core_mask = (1 << 0);

// Monitor idle task của core 1
wdt_config.idle_core_mask = (1 << 1);

// Monitor cả 2 cores
wdt_config.idle_core_mask = (1 << 0) | (1 << 1);

// Không monitor idle task (khuyến nghị cho custom tasks)
wdt_config.idle_core_mask = 0;
```

## 9. Troubleshooting

### 9.1. ESP32 reset liên tục

**Triệu chứng**: ESP32 reset mỗi vài giây

**Nguyên nhân**:
- Timeout quá ngắn
- Quên feed watchdog
- Task bị block

**Giải pháp**:
```c
// Tăng timeout
wdt_config.timeout_ms = 15000;  // Tăng lên 15s

// Hoặc tắt tạm thời để debug
wdt_config.trigger_panic = false;

// Kiểm tra tất cả vị trí cần feed
esp_task_wdt_reset();
```

### 9.2. Task bị timeout nhưng task khác không

**Nguyên nhân**: Task chưa subscribe hoặc quên feed

**Giải pháp**:
```c
void my_task(void *param)
{
    // QUAN TRỌNG: Subscribe task này
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    while (1) {
        // QUAN TRỌNG: Feed định kỳ
        ESP_ERROR_CHECK(esp_task_wdt_reset());

        // Code của bạn
        do_work();

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 9.3. Error "task watchdog not initialized"

**Giải pháp**:
```c
// Đảm bảo init trước khi add task
esp_task_wdt_init(&wdt_config);

// Sau đó mới add task
esp_task_wdt_add(NULL);
```

### 9.4. Error "task not subscribed"

**Giải pháp**:
```c
// Phải add task trước khi reset
esp_task_wdt_add(NULL);

// Sau đó mới có thể reset
esp_task_wdt_reset();
```

## 10. Migration từ ESP-IDF v4.x sang v5.x

### 10.1. Code cũ (v4.x)

```c
// ESP-IDF v4.x
#include "esp_task_wdt.h"

void app_main(void)
{
    // Cách cũ
    esp_task_wdt_init(5, true);  // timeout, panic
    esp_task_wdt_add(NULL);

    while (1) {
        esp_task_wdt_reset();
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 10.2. Code mới (v5.x)

```c
// ESP-IDF v5.x
#include "esp_task_wdt.h"

void app_main(void)
{
    // Cách mới - dùng struct config
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = 5000,  // 5 giây
        .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
        .trigger_panic = true
    };

    esp_task_wdt_init(&wdt_config);
    esp_task_wdt_add(NULL);

    while (1) {
        esp_task_wdt_reset();
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 10.3. Script chuyển đổi tự động

```python
# convert_wdt_v4_to_v5.py
import re

def convert_wdt_init(code):
    # Tìm esp_task_wdt_init(timeout, panic)
    pattern = r'esp_task_wdt_init\s*\(\s*(\d+)\s*,\s*(true|false)\s*\)'

    def replacer(match):
        timeout = match.group(1)
        panic = match.group(2)

        return f'''esp_task_wdt_config_t wdt_config = {{
        .timeout_ms = {timeout}000,
        .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
        .trigger_panic = {panic}
    }};
    esp_task_wdt_init(&wdt_config)'''

    return re.sub(pattern, replacer, code)

# Sử dụng
old_code = '''
esp_task_wdt_init(5, true);
esp_task_wdt_add(NULL);
'''

new_code = convert_wdt_init(old_code)
print(new_code)
```

## 11. Tổng kết

### 11.1. Checklist sử dụng Watchdog

- [ ] Chọn timeout phù hợp với ứng dụng
- [ ] Khởi tạo watchdog với `esp_task_wdt_init(&config)`
- [ ] Subscribe task với `esp_task_wdt_add(NULL)`
- [ ] Feed định kỳ với `esp_task_wdt_reset()`
- [ ] Xử lý error code đúng cách
- [ ] Test kỹ trước khi deploy
- [ ] Cleanup với `esp_task_wdt_delete()` và `esp_task_wdt_deinit()`

### 11.2. Khi nào nên dùng Watchdog

**BẮT BUỘC**:
- Thiết bị IoT hoạt động 24/7
- Hệ thống quan trọng không thể down
- Remote device không thể access vật lý

**NÊN DÙNG**:
- Ứng dụng production
- Xử lý dữ liệu quan trọng
- Real-time system

**CÓ THỂ BỎ QUA**:
- Development/testing
- Prototype
- Ứng dụng có người giám sát

### 11.3. Tài liệu tham khảo

- ESP-IDF v5.x Documentation: https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/wdts.html
- ESP-IDF GitHub: https://github.com/espressif/esp-idf
- ESP32 Forum: https://esp32.com/

## 12. Ví dụ Project Template

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_task_wdt.h"
#include "esp_log.h"

static const char *TAG = "MAIN";

// Watchdog config
#define WDT_TIMEOUT_MS 10000
#define WDT_PANIC true

// Task handles
TaskHandle_t sensor_task_handle = NULL;
TaskHandle_t comm_task_handle = NULL;

// Sensor task
void sensor_task(void *pvParameters)
{
    ESP_LOGI(TAG, "Sensor task started");
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    while (1) {
        // Đọc cảm biến
        float temperature = read_temperature();
        float humidity = read_humidity();

        ESP_LOGI(TAG, "Temp: %.1f°C, Humidity: %.1f%%",
                 temperature, humidity);

        ESP_ERROR_CHECK(esp_task_wdt_reset());
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

// Communication task
void comm_task(void *pvParameters)
{
    ESP_LOGI(TAG, "Communication task started");
    ESP_ERROR_CHECK(esp_task_wdt_add(NULL));

    while (1) {
        // Gửi dữ liệu
        send_data_to_server();

        ESP_ERROR_CHECK(esp_task_wdt_reset());
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

float read_temperature(void)
{
    // Implement sensor reading
    return 25.5;
}

float read_humidity(void)
{
    // Implement sensor reading
    return 60.0;
}

void send_data_to_server(void)
{
    // Implement communication
    ESP_LOGI(TAG, "Data sent");
}

void app_main(void)
{
    ESP_LOGI(TAG, "=== ESP32 Project with Watchdog ===");

    // Khởi tạo watchdog
    esp_task_wdt_config_t wdt_config = {
        .timeout_ms = WDT_TIMEOUT_MS,
        .idle_core_mask = 0,  // Không monitor idle task
        .trigger_panic = WDT_PANIC
    };

    ESP_ERROR_CHECK(esp_task_wdt_init(&wdt_config));
    ESP_LOGI(TAG, "Watchdog initialized (timeout: %d ms)", WDT_TIMEOUT_MS);

    // Tạo tasks
    xTaskCreatePinnedToCore(
        sensor_task,
        "SensorTask",
        4096,
        NULL,
        5,
        &sensor_task_handle,
        0
    );

    xTaskCreatePinnedToCore(
        comm_task,
        "CommTask",
        4096,
        NULL,
        5,
        &comm_task_handle,
        1
    );

    ESP_LOGI(TAG, "All tasks created successfully");
}
```

---

**Phiên bản**: ESP-IDF v5.x (v5.0, v5.1, v5.2, v5.3, v5.4, v5.5)
**Ngày cập nhật**: 2024
**Tương thích**: ESP32, ESP32-S2, ESP32-S3, ESP32-C3, ESP32-C6
