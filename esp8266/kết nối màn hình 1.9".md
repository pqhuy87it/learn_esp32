Quyết định chuyển sang `TFT_eSPI` là một lựa chọn cực kỳ chính xác. Thư viện này hỗ trợ vẽ các file font dạng mảng byte (`vlw` font) cực kỳ mượt mà, có tính năng khử răng cưa (anti-aliasing) giúp chữ nét căng và tự động ghi đè màu nền mà không bị chớp màn hình.

Để sử dụng, bạn hãy làm theo 3 bước sau:

### Bước 1: Cài đặt thư viện TFT_eSPI

1. Mở Arduino IDE.
2. Vào **Sketch** > **Include Library** > **Manage Libraries...**
3. Tìm từ khóa **TFT_eSPI** (của tác giả Bodmer) và bấm **Install**.

---

### Bước 2: Cấu hình phần cứng (Bắt buộc)

Điểm đặc biệt của `TFT_eSPI` là bạn không định nghĩa chân cắm ở trong file code `.ino`, mà phải định nghĩa một lần vào file hệ thống của thư viện.

1. Mở thư mục chứa thư viện trên máy tính của bạn theo đường dẫn này (dựa trên lỗi bạn vừa gửi):
`C:\Users\Admin\Documents\Arduino\libraries\TFT_eSPI\`
2. Tìm file có tên là **`User_Setup.h`**. Mở file đó lên bằng Notepad hoặc bất kỳ trình soạn thảo văn bản nào.
3. **Xóa trắng toàn bộ nội dung** trong file đó (để tránh bị trùng lặp cấu hình mặc định) và dán đoạn code cấu hình dành riêng cho mạch ESP8266 + Màn 1.9" ST7789 của bạn dưới đây vào:

```cpp
#define USER_SETUP_INFO "User_Setup"

// Chọn chip điều khiển màn hình
#define ST7789_DRIVER

// Kích thước màn hình
#define TFT_WIDTH  170
#define TFT_HEIGHT 320

// Cần thiết cho một số màn hình ST7789 để không bị lệch màu/lệch pixel
#define CGRAM_OFFSET 
#define TFT_RGB_ORDER TFT_RGB

// Khai báo chân cắm (Dựa trên sơ đồ bạn đang nối với NodeMCU V3)
#define TFT_CS   PIN_D8  
#define TFT_DC   PIN_D2  
#define TFT_RST  PIN_D4  

// Tải các tính năng cơ bản và tính năng Font mượt (Smooth Font)
#define LOAD_GLCD
#define LOAD_FONT2
#define LOAD_FONT4
#define SMOOTH_FONT

// Tốc độ truyền SPI
#define SPI_FREQUENCY  27000000

```

4. Lưu file `User_Setup.h` lại và đóng Notepad.

---

### Bước 3: Code hiển thị đồng hồ với font của bạn

Bây giờ quay lại Arduino IDE. Code của `TFT_eSPI` sẽ gọn hơn nhiều so với Adafruit. Thư viện này dùng lệnh `tft.loadFont()` để nạp font và `tft.drawString()` để in chữ với màu nền tự động xóa.

Bạn copy đoạn code mới này vào:

```cpp
#include <ESP8266WiFi.h>
#include <time.h>
#include <SPI.h>
#include <TFT_eSPI.h> // Đã thay bằng TFT_eSPI

// Đưa các file font của bạn vào
#include "midleFont.h"
#include "bigFont.h"

// Khởi tạo đối tượng tft (Không cần khai báo chân ở đây nữa)
TFT_eSPI tft = TFT_eSPI(); 

const char* ssid = "Tên_WiFi_của_bạn";
const char* password = "Mật_khẩu_WiFi";

const long gmtOffset_sec = 7 * 3600; 
const int daylightOffset_sec = 0;
const char* ntpServer = "pool.ntp.org";

const char* daysOfTheWeek[] = {"CN", "T2", "T3", "T4", "T5", "T6", "T7"};

char lastTimeString[9] = "";
char lastDateString[25] = "";

void setup() {
  Serial.begin(115200);

  tft.init();
  tft.setRotation(1); // Xoay ngang
  tft.fillScreen(TFT_BLACK); // Mã màu của TFT_eSPI bắt đầu bằng chữ TFT_
  
  // Hiển thị thông báo kết nối dùng font hệ thống mặc định
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.drawString("Dang ket noi WiFi...", 10, 50, 2); 
  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_GREEN, TFT_BLACK);
  tft.drawString("WiFi Da Ket Noi!", 10, 50, 2);
  delay(1000);
  
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  tft.fillScreen(TFT_BLACK);
}

void loop() {
  time_t now = time(nullptr);
  struct tm* timeinfo = localtime(&now);

  if (timeinfo->tm_year > 70) {
    
    // --- 1. XỬ LÝ NGÀY THÁNG NĂM ---
    char dateString[25];
    sprintf(dateString, "%s, %02d/%02d/%04d", 
            daysOfTheWeek[timeinfo->tm_wday], 
            timeinfo->tm_mday, 
            timeinfo->tm_mon + 1, 
            timeinfo->tm_year + 1900);

    if (strcmp(dateString, lastDateString) != 0) {
        // Nạp font ngày tháng (Tên biến lấy đúng trong file midleFont.h)
        tft.loadFont(midleFont); 
        tft.setTextColor(TFT_CYAN, TFT_BLACK); 
        // Lệnh drawString lấy tọa độ góc trên cùng bên trái của chữ
        tft.drawString(dateString, 15, 20); 
        tft.unloadFont(); // Xả font giải phóng RAM
        
        strcpy(lastDateString, dateString);
    }

    // --- 2. XỬ LÝ THỜI GIAN ---
    char timeString[9]; 
    sprintf(timeString, "%02d:%02d:%02d", timeinfo->tm_hour, timeinfo->tm_min, timeinfo->tm_sec);

    if (strcmp(timeString, lastTimeString) != 0) {
        tft.loadFont(bigFont);
        tft.setTextColor(TFT_ORANGE, TFT_BLACK);
        
        // Thiết lập vùng xóa nền (padding) rộng 300px (320px - 10px lề trái - 10px lề phải)
        tft.setTextPadding(300); 

        // Đặt điểm neo (anchor) của text vào chính giữa tâm (Middle Center)
        tft.setTextDatum(MC_DATUM); 

        // Vẽ chữ ở tọa độ tâm X = 160 (giữa màn hình), tâm Y = 100 (đẩy xuống dưới ngày tháng)
        tft.drawString(timeString, 160, 100);
        
        // Trả lại các thiết lập mặc định (Top Left) để không làm lệch tọa độ của phần in Ngày Tháng
        tft.setTextPadding(0); 
        tft.setTextDatum(TL_DATUM); 
        tft.unloadFont();
        
        strcpy(lastTimeString, timeString);
    }

  delay(200); 
}

```

**Lưu ý khi test code:** Vì file `bigFont.h` của bạn có dung lượng khá lớn (hơn 170KB), ESP8266 sẽ lưu font này trực tiếp vào bộ nhớ Flash. Quá trình biên dịch (Compile) và nạp code sẽ mất thời gian hơn một chút so với bình thường.

Sau khi nạp xong, nếu màu nền trên màn hình bị đảo ngược (ví dụ đen thành trắng), bạn có cần tôi hướng dẫn chèn thêm một câu lệnh ngắn để đảo ngược màu sắc lại cho đúng không?
