Để lấy thông số CPU và RAM từ máy Mac và hiển thị lên màn hình cắm với ESP32, cách ổn định và dễ mở rộng nhất là sử dụng **mô hình Client - Server qua mạng Wi-Fi (LAN)**.

Cụ thể, ESP32 sẽ đóng vai trò làm một **Web Server** thu nhỏ. Trên máy Mac, chúng ta sẽ chạy một đoạn **script Python** nhỏ (chạy ngầm) để liên tục đọc thông số hệ thống và gửi HTTP Request chứa data sang cho địa chỉ IP của ESP32.

Dưới đây là hướng dẫn chi tiết 2 phần:

### Phần 1: Code cho ESP32 (Web Server & Hiển thị)

Đoạn code này sử dụng thư viện `TFT_eSPI` để vẽ lên màn hình và thư viện `WebServer` có sẵn của ESP32 để nhận dữ liệu.

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <TFT_eSPI.h>

const char* ssid = "Tên_WiFi_của_bạn";
const char* password = "Mật_khẩu_WiFi";

// Khởi tạo WebServer ở port 80
WebServer server(80);

// Khởi tạo màn hình
TFT_eSPI tft = TFT_eSPI();

// Biến lưu trạng thái cũ để chống chớp nháy khi vẽ
String lastCpu = "";
String lastRam = "";

void setup() {
  Serial.begin(115200);

  tft.begin();
  tft.setRotation(1);
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  
  // Kết nối WiFi
  tft.drawString("Connecting WiFi...", 10, 10, 2);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  tft.fillScreen(TFT_BLACK);
  tft.drawString("Connected!", 10, 10, 2);
  tft.drawString("IP: " + WiFi.localIP().toString(), 10, 30, 2);
  
  Serial.print("ESP32 IP Address: ");
  Serial.println(WiFi.localIP());

  // Lắng nghe Request dạng: http://<IP_ESP32>/update?cpu=45.2&ram=60.5
  server.on("/update", []() {
    if (server.hasArg("cpu") && server.hasArg("ram")) {
      String cpu = server.arg("cpu");
      String ram = server.arg("ram");

      // Chỉ vẽ lại nếu thông số thay đổi
      if (cpu != lastCpu || ram != lastRam) {
        tft.setTextDatum(TL_DATUM);
        tft.setTextPadding(150); // Xóa nền cũ

        tft.setTextColor(TFT_GREEN, TFT_BLACK);
        tft.drawString("CPU: " + cpu + " %", 10, 80, 4);

        tft.setTextColor(TFT_ORANGE, TFT_BLACK);
        tft.drawString("RAM: " + ram + " %", 10, 120, 4);

        tft.setTextPadding(0); // Trả lại mặc định
        
        lastCpu = cpu;
        lastRam = ram;
      }
      
      // Phản hồi lại cho Mac biết đã nhận thành công
      server.send(200, "text/plain", "OK");
    } else {
      server.send(400, "text/plain", "Missing args");
    }
  });

  // Khởi động Server
  server.begin();
}

void loop() {
  // Liên tục xử lý các Request gửi đến
  server.handleClient();
}

```

**Lưu ý:** Sau khi nạp code và ESP32 kết nối WiFi thành công, hãy mở **Serial Monitor** (hoặc nhìn lên màn hình TFT) để lấy **địa chỉ IP của ESP32** (Ví dụ: `192.168.1.15`). Bạn sẽ cần IP này cho script Python.

---

### Phần 2: Script Python trên máy Mac

Python có một thư viện cực kỳ mạnh mẽ để đọc thông số phần cứng đa nền tảng là `psutil`.

**Bước 1:** Mở Terminal trên Mac và cài đặt 2 thư viện cần thiết:

```bash
pip3 install psutil requests

```

**Bước 2:** Tạo một file tên là `mac_monitor.py` và dán đoạn code sau vào:

```python
import psutil
import requests
import time

# ĐIỀN ĐỊA CHỈ IP CỦA ESP32 VÀO ĐÂY
ESP32_IP = "192.168.1.15" 

def send_data_to_esp32():
    print(f"Bắt đầu gửi dữ liệu tới ESP32 ({ESP32_IP})... Nhấn Ctrl+C để dừng.")
    
    while True:
        try:
            # Đọc % CPU (interval=1 giúp tính toán độ load chính xác trong 1 giây qua)
            cpu_percent = psutil.cpu_percent(interval=1)
            
            # Đọc % RAM đã sử dụng
            ram_percent = psutil.virtual_memory().percent
            
            # Tạo URL gửi HTTP GET Request tới ESP32
            url = f"http://{ESP32_IP}/update?cpu={cpu_percent}&ram={ram_percent}"
            
            # Gửi request với timeout 2 giây để tránh treo script nếu mạng chập chờn
            response = requests.get(url, timeout=2)
            
            if response.status_code == 200:
                print(f"Đã gửi -> CPU: {cpu_percent}% | RAM: {ram_percent}%")
            else:
                print(f"Lỗi phản hồi từ ESP32: {response.status_code}")
                
        except requests.exceptions.RequestException as e:
            print(f"Không thể kết nối tới ESP32: {e}")
            time.sleep(2) # Đợi 2 giây rồi thử lại nếu mất mạng
            
        except KeyboardInterrupt:
            print("\nĐã dừng.")
            break

if __name__ == "__main__":
    send_data_to_esp32()

```

**Bước 3:** Chạy script trên Terminal:

```bash
python3 mac_monitor.py

```

Ngay khi script chạy, nó sẽ lấy thông số hệ thống mỗi giây một lần và bắn thẳng qua Wi-Fi nội bộ đến ESP32. Màn hình của bạn sẽ lập tức cập nhật CPU và RAM theo thời gian thực.

Bạn có muốn thêm phần vẽ đồ thị dạng thanh (bar chart) cho CPU và RAM chạy ngang màn hình để nhìn cho trực quan hơn giống như dashboard thật không?
