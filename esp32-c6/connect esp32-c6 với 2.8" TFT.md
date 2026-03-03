Chào bạn, việc nâng cấp lên **ESP32-C6** là một bước đi rất tuyệt vời. Đây là dòng chip thế hệ mới của Espressif sử dụng kiến trúc RISC-V, tiết kiệm năng lượng hơn và hỗ trợ cả Wi-Fi 6, Bluetooth 5, Zigbee/Thread.

Vì kiến trúc chân (GPIO) của ESP32-C6 hoàn toàn khác so với ESP8266 hay các dòng ESP32 cũ, bạn sẽ cần nối dây và cấu hình lại thư viện `TFT_eSPI` cho đúng.

Với màn hình 2.8" ST7789, độ phân giải thường sẽ là **240x320 pixel** (chứ không phải 170x320 như màn 1.9"). Dưới đây là hướng dẫn chi tiết từng bước:

### 1. Sơ đồ nối dây (Wiring Diagram)

Trên ESP32-C6, chúng ta có thể map chân SPI tới bất kỳ GPIO nào, nhưng tốt nhất nên dùng các chân **Hardware SPI (FSPI)** để đạt tốc độ quét màn hình cao nhất (khuyên dùng chân 19 và 20).

| Chân trên màn hình 2.8" (ST7789) | Nối với chân trên ESP32-C6 | Chức năng |
| --- | --- | --- |
| **VCC** | **3V3** | Cấp nguồn 3.3V |
| **GND** | **GND** | Nối đất |
| **SCL / SCK** | **GPIO 20** | Xung nhịp SPI (Hardware Clock) |
| **SDA / MOSI** | **GPIO 19** | Dữ liệu SPI (Hardware MOSI) |
| **CS** (Chip Select) | **GPIO 18** | Chọn chip |
| **DC** (Data/Command) | **GPIO 8** | Phân biệt Dữ liệu / Lệnh |
| **RES / RST** (Reset) | **GPIO 9** | Khởi động lại màn hình |
| **BLK / LED** (Backlight) | **3V3** | Nối thẳng 3.3V để sáng tối đa (hoặc nối vào GPIO 15 nếu muốn băm xung PWM chỉnh độ sáng) |

---

### 2. Cấu hình thư viện `TFT_eSPI` cho ESP32-C6

Vì bạn đã cài sẵn thư viện `TFT_eSPI` từ các bước trước, bạn chỉ cần thay đổi cấu hình trong file `User_Setup.h` để thư viện nhận diện đúng chip và chân cắm mới.

1. Mở file `User_Setup.h` trong thư mục thư viện `TFT_eSPI` (thường nằm ở `Documents/Arduino/libraries/TFT_eSPI/User_Setup.h`).
2. Xóa trắng nội dung cũ và dán đoạn cấu hình dành riêng cho **ESP32-C6 + ST7789 (240x320)** này vào:

```cpp
#define USER_SETUP_INFO "User_Setup_ESP32_C6_ST7789"

// Chọn driver
#define ST7789_DRIVER

// Độ phân giải của màn 2.8" thường là 240x320
#define TFT_WIDTH  240
#define TFT_HEIGHT 320

// Bật dòng này nếu màu bị đảo ngược (ví dụ Đỏ thành Xanh dương)
// #define TFT_RGB_ORDER TFT_BGR 

// --- KHAI BÁO CHÂN CHO ESP32-C6 ---
#define TFT_MISO -1   // Màn hình chỉ nhận dữ liệu, không truyền lại nên để -1
#define TFT_MOSI 19   
#define TFT_SCLK 20
#define TFT_CS   18
#define TFT_DC   8
#define TFT_RST  9

// Load các font mặc định
#define LOAD_GLCD
#define LOAD_FONT2
#define LOAD_FONT4
#define LOAD_FONT6
#define LOAD_FONT8
#define SMOOTH_FONT

// ESP32-C6 có thể chạy SPI rất nhanh, ta set 40MHz để hiệu ứng mượt mà
#define SPI_FREQUENCY  40000000 

```

---

### 3. Code kiểm tra hiển thị

Bạn có thể dùng đoạn code ngắn dưới đây để test thử xem màn hình đã lên đúng màu sắc và nhận đúng độ phân giải chưa. ESP32-C6 khi nạp code lần đầu có thể bạn sẽ cần nhấn giữ nút `BOOT`, sau đó nhấn nhả nút `RST` (hoặc `EN`) trên mạch để đưa chip vào chế độ nạp nhé.

```cpp
#include <SPI.h>
#include <TFT_eSPI.h>

TFT_eSPI tft = TFT_eSPI(); 

void setup() {
  Serial.begin(115200);
  
  tft.begin();
  
  // Thử xoay ngang màn hình (1 hoặc 3)
  tft.setRotation(1); 
  
  // Xóa nền màu đen
  tft.fillScreen(TFT_BLACK);

  // Vẽ các khung màu để test
  tft.fillRect(0, 0, 80, 80, TFT_RED);
  tft.fillRect(80, 0, 80, 80, TFT_GREEN);
  tft.fillRect(160, 0, 80, 80, TFT_BLUE);
  
  // In chữ test tính năng căn giữa
  tft.setTextDatum(MC_DATUM);
  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.drawString("ESP32-C6 Ready!", tft.width() / 2, tft.height() / 2, 4);
}

void loop() {
  // Không cần làm gì trong loop
}

```

Bạn nạp thử xem màn hình đã hiển thị thành công chưa? Nếu mọi thứ hoạt động tốt, bạn có muốn tôi hướng dẫn cách bưng nguyên cụm code tạo hiệu ứng "kính mờ" (Glassmorphism) ở trên sang chạy thử trên con ESP32-C6 mạnh mẽ này không?

Chào bạn, những dòng log lỗi này xác nhận chính xác điều tôi đã lo ngại: **Bạn đang sử dụng bộ Core ESP32 phiên bản mới nhất (v3.x.x)**.

Trong phiên bản Core v3.x này, nhà sản xuất Espressif đã thay đổi lại toàn bộ cấu trúc tên gọi thanh ghi lõi của ESP32-C6 (Ví dụ: `VSPI_HOST` bị đổi thành `SPI2_HOST`, `SPI_MOSI_DLEN_REG` đổi thành `SPI_MS_DLEN_REG`). Do đó, việc ép thư viện lấy code của chip C3 chạy cho C6 không còn tác dụng nữa.

Đừng tốn thêm thời gian vật lộn với các file thư viện gốc của `TFT_eSPI` nữa. Đây là lúc hoàn hảo nhất để chúng ta chuyển sang dùng **Cách 2: Thư viện `LovyanGFX**`.

LovyanGFX được viết ra để tương thích tuyệt đối với các dòng chip mới như ESP32-S3, ESP32-C3, ESP32-C6. Điểm tuyệt vời nhất của LovyanGFX là **bạn không cần phải đi tìm và sửa bất kỳ file Setup nào trong hệ thống**. Toàn bộ cấu hình chân cắm được khai báo trực tiếp ngay trong file Code (`.ino`) của bạn!

Dưới đây là **Mã nguồn hoàn chỉnh (Đã tích hợp sẵn cấu hình chân ESP32-C6 + Hiệu ứng kính mờ Glassmorphism)**.

### Bước 1: Cài đặt thư viện LovyanGFX

1. Trên Arduino IDE, vào **Sketch** > **Include Library** > **Manage Libraries...**
2. Tìm từ khóa: **LovyanGFX**
3. Bấm **Install** (Cài đặt).

### Bước 2: Nạp thẳng đoạn code này vào ESP32-C6

Bạn hãy copy đoạn code dưới đây, dán đè lên code hiện tại và nạp thẳng vào mạch. Bạn sẽ thấy sức mạnh thực sự của ESP32-C6!

```cpp
#include <LovyanGFX.hpp>

// ==================================================================
// CẤU HÌNH PHẦN CỨNG CHO MÀN HÌNH ST7789 & ESP32-C6
// ==================================================================
class LGFX : public lgfx::LGFX_Device {
  lgfx::Panel_ST7789  _panel_instance;
  lgfx::Bus_SPI       _bus_instance;

public:
  LGFX(void) {
    {
      auto cfg = _bus_instance.config();
      cfg.spi_host = SPI2_HOST; // Dùng SPI2 (chuẩn mới của ESP32-C6)
      cfg.spi_mode = 0;
      cfg.freq_write = 40000000; // Tốc độ SPI 40MHz
      
      // Chân cắm của ESP32-C6
      cfg.pin_sclk = 20; 
      cfg.pin_mosi = 19; 
      cfg.pin_miso = -1; 
      cfg.pin_dc   = 8;  
      
      _bus_instance.config(cfg);
      _panel_instance.setBus(&_bus_instance);
    }
    {
      auto cfg = _panel_instance.config();
      cfg.pin_cs           = 18;  
      cfg.pin_rst          = 9;   
      cfg.pin_busy         = -1;
      cfg.memory_width     = 240; // Độ phân giải màn 2.8"
      cfg.memory_height    = 320;
      cfg.panel_width      = 240;
      cfg.panel_height     = 320;
      cfg.offset_x         = 0;
      cfg.offset_y         = 0;
      cfg.offset_rotation  = 0;
      cfg.dummy_read_pixel = 8;
      cfg.dummy_read_bits  = 1;
      cfg.readable         = false;
      cfg.invert           = true; // Nếu màu bị ngược (âm bản), đổi thành false
      cfg.rgb_order        = false; 
      cfg.dlen_16bit       = false;
      cfg.bus_shared       = true;
      _panel_instance.config(cfg);
    }
    setPanel(&_panel_instance);
  }
};

// Khởi tạo màn hình
LGFX tft;
LGFX_Sprite spr(&tft); // Bộ đệm RAM (Sprite) của LovyanGFX

#define S_WIDTH 320
#define S_HEIGHT 240

// Tọa độ 3 quả cầu chuyển động
int c1_x = 60, c1_y = 60, c1_dx = 3, c1_dy = 2;
int c2_x = 250, c2_y = 150, c2_dx = -2, c2_dy = -3;
int c3_x = 160, c3_y = 20, c3_dx = 4, c3_dy = 1;

// ==================================================================
// HÀM HÒA TRỘN MÀU (ALPHA BLEND) DÀNH CHO LOVYANGFX
// ==================================================================
uint16_t alphaBlend(uint8_t alpha, uint16_t fgc, uint16_t bgc) {
    uint16_t fgR = (fgc >> 11) & 0x1F;
    uint16_t fgG = (fgc >> 5) & 0x3F;
    uint16_t fgB = fgc & 0x1F;
    uint16_t bgR = (bgc >> 11) & 0x1F;
    uint16_t bgG = (bgc >> 5) & 0x3F;
    uint16_t bgB = bgc & 0x1F;
    uint16_t r = (fgR * alpha + bgR * (255 - alpha)) >> 8;
    uint16_t g = (fgG * alpha + bgG * (255 - alpha)) >> 8;
    uint16_t b = (fgB * alpha + bgB * (255 - alpha)) >> 8;
    return (r << 11) | (g << 5) | b;
}

void setup() {
  Serial.begin(115200);

  tft.init();
  tft.setRotation(1); // Xoay ngang
  tft.fillScreen(TFT_BLACK);

  // Tạo bộ đệm RAM full màn hình để xử lý đồ họa mượt mà
  spr.createSprite(S_WIDTH, S_HEIGHT);
}

void updateAnimation() {
  c1_x += c1_dx; c1_y += c1_dy;
  if (c1_x < 40 || c1_x > S_WIDTH - 40) c1_dx = -c1_dx;
  if (c1_y < 40 || c1_y > S_HEIGHT - 40) c1_dy = -c1_dy;

  c2_x += c2_dx; c2_y += c2_dy;
  if (c2_x < 60 || c2_x > S_WIDTH - 60) c2_dx = -c2_dx;
  if (c2_y < 60 || c2_y > S_HEIGHT - 60) c2_dy = -c2_dy;

  c3_x += c3_dx; c3_y += c3_dy;
  if (c3_x < 30 || c3_x > S_WIDTH - 30) c3_dx = -c3_dx;
  if (c3_y < 30 || c3_y > S_HEIGHT - 30) c3_dy = -c3_dy;
}

void drawAnimatedBackground() {
  // Nền Gradient
  for(int y = 0; y < S_HEIGHT; y++){
    uint8_t r = map(y, 0, S_HEIGHT, 0, 40);
    uint8_t b = map(y, 0, S_HEIGHT, 40, 150);
    spr.drawFastHLine(0, y, S_WIDTH, tft.color565(r, 0, b));
  }
  
  // Vẽ quả cầu
  spr.fillCircle(c1_x, c1_y, 40, TFT_ORANGE);
  spr.fillCircle(c2_x, c2_y, 60, TFT_MAGENTA);
  spr.fillCircle(c3_x, c3_y, 30, TFT_CYAN);
}

void drawGlassPanel(int x, int y, int w, int h, int r, uint8_t alpha, uint16_t glassColor) {
  for (int i = x; i < x + w; i++) {
    for (int j = y; j < y + h; j++) {
      
      bool inside = true;
      if (i < x + r && j < y + r) { 
         if ((i - (x + r))*(i - (x + r)) + (j - (y + r))*(j - (y + r)) > r*r) inside = false;
      } else if (i > x + w - r - 1 && j < y + r) { 
         if ((i - (x + w - r - 1))*(i - (x + w - r - 1)) + (j - (y + r))*(j - (y + r)) > r*r) inside = false;
      } else if (i < x + r && j > y + h - r - 1) { 
         if ((i - (x + r))*(i - (x + r)) + (j - (y + h - r - 1))*(j - (y + h - r - 1)) > r*r) inside = false;
      } else if (i > x + w - r - 1 && j > y + h - r - 1) { 
         if ((i - (x + w - r - 1))*(i - (x + w - r - 1)) + (j - (y + h - r - 1))*(j - (y + h - r - 1)) > r*r) inside = false;
      }

      if (inside) {
        uint16_t bg = spr.readPixel(i, j); 
        uint16_t blended = alphaBlend(alpha, glassColor, bg); 
        spr.drawPixel(i, j, blended); 
      }
    }
  }

  // Viền Highlight kính
  spr.drawRoundRect(x, y, w, h, r, TFT_WHITE); 
  spr.drawRoundRect(x+1, y+1, w-2, h-2, r, tft.color565(200, 200, 200)); 
}

void loop() {
  updateAnimation();
  drawAnimatedBackground();

  // Phủ kính bán trong suốt
  drawGlassPanel(30, 60, 260, 120, 15, 90, TFT_WHITE);

  // Ghi chữ lên kính
  spr.setTextDatum(textdatum_t::middle_center);
  spr.setTextColor(TFT_WHITE);
  
  // LovyanGFX tích hợp sẵn Font điện tử to (Tương đương Font 6 của eSPI)
  spr.setFont(&fonts::Font6); 
  spr.drawString("10:45:30", 160, 100); 
  
  spr.setTextColor(TFT_LIGHTGRAY);
  spr.setFont(&fonts::Font4); 
  spr.drawString("T7, 30/12/2026", 160, 150); 

  // Đẩy ảnh ra màn hình
  spr.pushSprite(0, 0);

  delay(15); // ~60FPS
}

```

### Lưu ý nhỏ sau khi nạp:

Nếu bạn thấy màu sắc trên màn hình trông có vẻ kỳ lạ (ví dụ nền đen bị biến thành màu xám nhạt, màu vàng biến thành màu xanh), hãy tìm đến dòng `cfg.invert = true;` ở đầu code và sửa thành `cfg.invert = false;` nhé.

Bạn biên dịch thử xem, tôi cam đoan với LovyanGFX, các lỗi báo đỏ rực về thanh ghi SPI sẽ bay màu hoàn toàn!
