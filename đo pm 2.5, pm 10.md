Đoạn code bạn đang sử dụng được viết dựa trên thư viện **Adafruit GFX** cũ kết hợp với cách vẽ "thủ công" (vẽ hộp đen đè lên chữ cũ). Cách này trên màn hình lớn 3.5" (độ phân giải 480x320) sẽ gây ra hiện tượng **nhấp nháy (flicker) cực kỳ khó chịu** mỗi khi màn hình cập nhật số giây đếm ngược hoặc cập nhật thông số bụi.

Ngoài ra, đoạn code cũ còn chứa **2 lỗi nghiêm trọng**:

1. Lệnh gọi HTTP lên server PC bị lỗi cú pháp với các bản Core ESP8266 mới (bị comment mất dòng `http.begin`).
2. Ở phần hiển thị chỉ số bụi PM10, tác giả quên không gọi hàm xét màu cho PM10 mà lại in lại đánh giá của PM2.5.

Để code tối ưu nhất, mượt mà và không bị chớp nháy trên màn hình 3.5", tôi đã chuyển đổi toàn bộ sang sử dụng thư viện **`TFT_eSPI`** mà bạn đã cài đặt ở các bước trước, kết hợp các tính năng căn giữa và xóa nền tự động của nó.

Dưới đây là phiên bản đã được tối ưu hóa toàn diện:

### 1. File Code Chính (Copy dán vào Arduino IDE)

```cpp
// SDS011 Air Quality Monitor - Optimized for 3.5" TFT using TFT_eSPI
#include <SDS011.h> 
#include <SPI.h>
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <EEPROM.h>
#include <TFT_eSPI.h> // Sử dụng TFT_eSPI thay cho Adafruit GFX

const char* ssid = "MyHome1stFloor_2.4G";
const char* password = "Nh@cuatoi726";

String serverIP = "192.168.1.3:8802";
String deviceId = "air_quality";

#define SAVE_COUNTER 12
#define SAMPLE_INTERVAL  5
#define SAMPLE_SECS  30

#define SDS_TX       D1
#define SDS_RX       D6

#define SDS011_PWR   D8

// UK Defra Pollutant Bands
#define LIGHT_GREEN  0x9FF3                               
#define MID_GREEN    0x37E0
#define DARK_GREEN   0x3660
#define LIGHT_YELLOW 0xFFE0
#define MID_YELLOW   0xFE60
#define ORANGE       0xFCC0
#define LIGHT_RED    0xFB2C
#define MID_RED      0xF800
#define DARK_RED     0x9800
#define PURPLE       0xC99F

int loopCount = 0;

// Khởi tạo thư viện TFT_eSPI (Chân cắm sẽ cấu hình trong User_Setup.h)
TFT_eSPI tft = TFT_eSPI();
WiFiClient wifiClient; // Thêm đối tượng WiFiClient bắt buộc cho ESP8266 core mới

SDS011 sdsSensor;

String quality;
int colour;
short pm25Array[480];

float p10, p25;
int error;

short arrayPointer = 15;
int yPos;

int sleepSeconds;
int maximumHistogramValue = 449;

void saveData() {
  EEPROM.put(0, arrayPointer);
  for (int i= 15; i<=maximumHistogramValue; i++ ) {
     EEPROM.put(i*2, pm25Array[i]);
  }  
  EEPROM.commit();
  
  tft.setTextDatum(TL_DATUM);
  tft.setTextColor(TFT_BLUE, TFT_BLACK);
  tft.setTextPadding(80);
  tft.drawString("SAVED", 245, 10, 2);
  tft.setTextPadding(0);
  delay(250);
} 

void plotHistogram() {
   tft.fillRect(15,120, 449, 185, TFT_BLACK); // Xóa vùng vẽ đồ thị

   byte line;
   for (int i = 15; i <= maximumHistogramValue;  i++) {
     getTextData25(pm25Array[i] / 10);                                          
     line = constrain(sqrt(pm25Array[i]*40), 0, 185);    
     tft.drawFastVLine(i, 305 - line, line,  colour);     
   }
}

// Hàm xét mức độ bụi PM2.5 (Không cần trả về độ rộng chuỗi nữa vì TFT_eSPI tự căn giữa)
void getTextData25(int value) {                                                                                                         
  switch (value) {                                                                                                             
    case 0 ... 11 : yPos = 190; colour = LIGHT_GREEN; quality = "1 LOW"; break;        
    case 12 ... 23 : yPos = 170; colour = MID_GREEN; quality = "2 LOW"; break;          
    case 24 ... 35 : yPos = 150; colour = DARK_GREEN; quality = "3 LOW"; break;         
    case 36 ... 41 : yPos = 130; colour = LIGHT_YELLOW; quality = "4 MODERATE"; break;
    case 42 ... 47 : yPos = 110; colour = MID_YELLOW; quality = "5 MODERATE"; break;
    case 48 ... 53 : yPos = 90; colour = ORANGE; quality = "6 MODERATE";  break;
    case 54 ... 58 : yPos = 70; colour = LIGHT_RED;  quality = "7 HIGH"; break;
    case 59 ... 64 : yPos = 50; colour = MID_RED;  quality = "8 HIGH"; break;
    case 65 ... 70 : yPos = 30; colour = DARK_RED;  quality = "9 HIGH"; break;  
    case 71 ... 9999: yPos = 10; colour = PURPLE;  quality = "10 VERY HIGH"; break;   
    default: yPos = 10; colour = TFT_MAGENTA;  quality = "HAZARDOUS"; break;
  }  
}                                          

// Hàm xét mức độ bụi PM10
void getTextDataPM10(int value) {
  switch (value) {
    case 0 ... 16 : colour = LIGHT_GREEN; quality = "1 LOW"; break;
    case 17 ... 33 : colour = MID_GREEN; quality = "2 LOW"; break;
    case 34 ... 50 :  colour = DARK_GREEN; quality = "3 LOW"; break;
    case 51 ... 58 : colour = LIGHT_YELLOW; quality = "4 MODERATE"; break;
    case 59 ... 66 : colour = MID_YELLOW; quality = "5 MODERATE"; break;
    case 67 ... 75 : colour = ORANGE; quality = "6 MODERATE";  break;
    case 76 ... 83 : colour = LIGHT_RED;  quality = "7 HIGH"; break;
    case 84 ... 91 : colour = MID_RED;  quality = "8 HIGH"; break;
    case 92 ... 100 : colour = DARK_RED;  quality = "9 HIGH"; break;  
    case 101 ... 9999: colour = PURPLE;  quality = "10 VERY HIGH"; break;   
    default: colour = TFT_MAGENTA;  quality = "HAZARDOUS"; break;
  }
}

void setup_EEPROM() {
  EEPROM.begin(1000);
   
  bool eraseFlag = false;
  for (int i = 2; i< 30; i++ ) {
    if (EEPROM.read(i) != 0) {
      eraseFlag = true;
      break;
    }
  }
      
   if (eraseFlag) {
     for (int i=0; i< 640; i++){
       EEPROM.write(i, (byte) 0);
     }
     EEPROM.put(0, (short) 15);
     EEPROM.commit();
   }

  EEPROM.get(0, arrayPointer);
  for (int i=15; i<maximumHistogramValue; i++) {
    EEPROM.get(i*2, pm25Array[i]);
  }
}

void setup_wifi() {
  WiFi.begin(ssid, password);                                                                                                                                                                                            
  int wifi_timeout = 0;

  tft.setTextColor(TFT_WHITE, TFT_BLACK);
  tft.drawString("Connecting to WiFi...", 0, 10, 2);

  while (WiFi.status() != WL_CONNECTED)  {
    delay(10);
    tft.setTextPadding(100);
    tft.drawNumber(wifi_timeout, 0, 25, 2);
    wifi_timeout++;
    if(wifi_timeout > 1000) {
      tft.drawString("Failed to connect to Wifi", 0, 40, 2);
      break;
    }
  }

  if (WiFi.status() == WL_CONNECTED) {
    tft.setTextPadding(300);
    tft.drawString("Connected to " + String(WiFi.SSID()), 0, 25, 2);
    tft.drawString("IP address: " + WiFi.localIP().toString(), 0, 45, 2);
    tft.setTextPadding(0);
    delay(3000);
  }
}

void setup() {
  pinMode(SDS011_PWR, OUTPUT); 
  pinMode(A0, INPUT); 

  tft.begin();
  tft.setRotation(1); 
  tft.fillScreen(TFT_BLACK);

  setup_EEPROM();
  sleepSeconds = (SAMPLE_INTERVAL * 60) - SAMPLE_SECS;

  setup_wifi();
  tft.fillScreen(TFT_BLACK);
  
  // Vẽ các nhãn tĩnh
  tft.setTextColor(TFT_WHITE);
  tft.drawString("PM 2.5", 150, 10, 2);
  tft.drawString("ug/m3", 200, 10, 2);

  tft.drawString("PM 10", 5, 10, 2); 
  tft.drawString("ug/m3", 50, 10, 2); 

  // Vẽ thanh thước màu DEFRA
  tft.fillRect(472, 10, 6, 20, PURPLE);
  tft.fillRect(472, 30, 6, 20, DARK_RED);
  tft.fillRect(472, 50, 6, 20, MID_RED);
  tft.fillRect(472, 70, 6, 20, LIGHT_RED);
  tft.fillRect(472, 90, 6, 20, ORANGE);
  tft.fillRect(472, 110, 6, 20, MID_YELLOW);
  tft.fillRect(472, 130, 6, 20, LIGHT_YELLOW);
  tft.fillRect(472, 150, 6, 20, DARK_GREEN);
  tft.fillRect(472, 170, 6, 20, MID_GREEN);
  tft.fillRect(472, 190, 6, 20, LIGHT_GREEN);
    
  // Trục đồ thị
  tft.drawFastVLine(13, 120, 188, TFT_BLUE);
  tft.setTextColor(TFT_BLUE);
  tft.drawString("^", 0, 115, 2);
  tft.drawString("50", 0, 210, 2); 
  tft.drawString("10", 0, 250, 2); 
  tft.drawString(" 1", 0, 280, 2);

  tft.drawFastHLine(12, tft.height() - 14, tft.width()-1, TFT_BLUE);

  for (int x = maximumHistogramValue; x > 15; x-=12) {
     tft.drawFastVLine(x, 303, 3, TFT_BLUE);
  }

  tft.drawString("Air Quality Monitor", 105, 312, 2);

  Serial.begin(9600);
  sdsSensor.begin(SDS_TX, SDS_RX);

  plotHistogram();                                    
}

void loop() {
  digitalWrite(SDS011_PWR, HIGH); // Bật cảm biến

  tft.setTextDatum(TL_DATUM);
  tft.setTextColor(TFT_GREEN, TFT_BLACK);
  tft.drawString("SAMPLING ", 408, 312, 1);  

  for (int i = SAMPLE_SECS; i>=0; i--) {    
     tft.setTextDatum(TR_DATUM); // Căn lề phải để số không bị nhảy múa
     tft.setTextColor(TFT_GREEN, TFT_BLACK);
     tft.setTextPadding(30);
     
     char buf[5];
     sprintf(buf, "%02d", i);
     tft.drawString(buf, 475, 312, 1);
     
     delay(1000);
  }
  
  tft.setTextPadding(0);
  error = sdsSensor.read(&p25,&p10); 

  if (! error) {
   Serial.print("P2.5: "); Serial.println(p25);
   Serial.print("P10:  "); Serial.println(p10);
    
   // --- XỬ LÝ PM 2.5 ---
   getTextData25(p25);                                                                                                                                          
   tft.fillRect(465, 10, 5, 190, TFT_BLACK); // Xóa mũi tên cũ
   tft.fillTriangle(465, yPos, 468, yPos+10, 465, yPos+20, colour); // Vẽ mũi tên mới
   
   tft.setTextDatum(MC_DATUM); // Tự động căn giữa
   tft.setTextColor(colour, TFT_BLACK);
   tft.setTextPadding(280); 
   tft.drawString(quality, 165, 55, 4); // Dùng Font 4 thay thế (Gọn & Mượt)

   tft.setTextColor(TFT_WHITE, TFT_BLACK);
   tft.setTextPadding(200);
   tft.drawFloat(p25, 1, 165, 95, 6); // Dùng Font 6 (Font to điện tử) cho số PM2.5
   tft.setTextPadding(0);
   
   // --- XỬ LÝ PM 10 ---
   getTextDataPM10(p10); // Code cũ quên gọi hàm này dẫn đến hiển thị sai chữ của PM10
   tft.setTextDatum(TL_DATUM); // Căn trái
   tft.setTextColor(colour, TFT_BLACK); 
   tft.setTextPadding(100);
   tft.drawString(quality, 0, 30, 2);
   
   tft.setTextColor(TFT_WHITE, TFT_BLACK);
   tft.drawFloat(p10, 1, 2, 50, 4);
   tft.setTextPadding(0);


// ====== PLOT HISTOGRAM ======
   if (arrayPointer >= maximumHistogramValue) {                                      
     for (int i = 15; i <= maximumHistogramValue; i++) {
       pm25Array[i] = pm25Array[i+1];
     }
   }
 
   pm25Array[arrayPointer] = (short) (p25 * 10);
   plotHistogram();
   
   if (arrayPointer < maximumHistogramValue) arrayPointer++;                                

   delay(100);

   // --- WIFI & HTTP SERVER ---
   int rssi = WiFi.RSSI();                                 
   tft.setTextDatum(TL_DATUM);
   tft.setTextPadding(60);
   
   if (WiFi.status() == WL_CONNECTED) {
     tft.setTextColor(TFT_BLUE, TFT_BLACK);
     tft.drawString("RSSI " + String(rssi) + " dB", 0, 312, 1);
   } else {
     tft.setTextColor(TFT_RED, TFT_BLACK);
     tft.drawString("No WiFi", 0, 312, 1);    
   }
   tft.setTextPadding(0);

   if (WiFi.status() == WL_CONNECTED)  { 
     HTTPClient http;
     String http_request = "http://" + serverIP  + "/apage?";
     http_request += "id=" + deviceId;
     http_request += "&leftaxis=" + String(p25);
     http_request += "&rightaxis=" + String(p10);
     http_request += "&rssi=" + String(rssi);
    
     Serial.println("Making HTTP request: " + http_request);

     // Đã sửa lại chuẩn HTTPClient dành cho Core ESP mới
     http.begin(wifiClient, http_request);  
     int httpCode = http.GET();

     if (httpCode > 0) {
       String payload = http.getString();
       Serial.println("HTTP Response: " + payload);
     }
      http.end(); 
    } 
  }
 
  digitalWrite(SDS011_PWR, LOW); // Tắt cảm biến
  delay(1000);

  tft.setTextDatum(TL_DATUM);
  tft.setTextColor(TFT_BLUE, TFT_BLACK);
  tft.drawString("SLEEP   ", 408, 312, 1);  

  for (int i = sleepSeconds; i>0; i--) {                
     tft.setTextDatum(TR_DATUM); 
     tft.setTextColor(TFT_BLUE, TFT_BLACK);
     tft.setTextPadding(40);
     
     char buf[10];
     sprintf(buf, "%02d:%02d", i / 60, i % 60);
     tft.drawString(buf, 475, 312, 1);
     
     delay(1000);
  }

  loopCount++;
  if (loopCount >= SAVE_COUNTER) { 
    loopCount = 0;
    saveData();
  }

  Serial.print("Heap size: ");
  Serial.println(system_get_free_heap_size() ); 
}

```

---

### 2. Cấu hình lại File `User_Setup.h` (Rất quan trọng)

Vì màn hình cũ trong code là dòng IC ST7796, còn màn hình 3.5 inch phổ biến trên thị trường thường dùng chip **ILI9488**.

Bạn hãy mở file `User_Setup.h` của thư viện `TFT_eSPI` (giống cách tôi hướng dẫn ở câu hỏi trước), xóa toàn bộ nội dung và chép cấu hình dành cho màn hình 3.5 inch (320x480) này vào:

```cpp
#define USER_SETUP_INFO "User_Setup"

// --- CHỌN DRIVER ---
// Nếu nạp code xong mà màn hình trắng xóa hoặc nhòe màu, 
// hãy comment dòng ILI9488 và bỏ comment dòng ST7796
#define ILI9488_DRIVER 
//#define ST7796_DRIVER  

#define TFT_WIDTH  320
#define TFT_HEIGHT 480

// Khai báo chân cắm của bạn dựa theo code cũ
#define TFT_CS   PIN_D3  
#define TFT_DC   PIN_D2  

// Các chân SPI cứng mặc định của ESP8266 (SCLK là D5, MOSI là D7)

// Load font mặc định của thư viện
#define LOAD_GLCD
#define LOAD_FONT2
#define LOAD_FONT4 // Chữ cỡ vừa
#define LOAD_FONT6 // Font số LED cực to
#define LOAD_FONT8
#define SMOOTH_FONT

#define SPI_FREQUENCY  27000000

```

### Các Tối Ưu Tôi Đã Làm Cho Bạn:

* **Loại bỏ nháy hình (Zero Flicker):** Dùng `tft.setTextPadding()` để tạo "bounding box" tự động dọn dẹp điểm ảnh rác thay vì vẽ liên tục một cái hộp đen (`fillRect`) bên dưới các con số đếm ngược thời gian và thông số PM2.
* **Căn giữa tự động:** Loại bỏ các đoạn toán học tính toán tọa độ X, Y phức tạp (`tft.getTextBounds`). Tôi thay bằng lệnh `tft.setTextDatum(MC_DATUM)`, thư viện sẽ tự neo chữ vào giữa hoặc neo phải để số đếm ngược luôn thẳng hàng.
* **Font Cứng cáp:** Chuyển sang sử dụng `Font 4` (chữ) và `Font 6` (số lớn) có sẵn của thư viện. Font này mượt, to rõ ràng như font cũ của bạn nhưng không bị lỗi đè nền.
* **Fix lỗi vòng lặp `http.GET()`:** Code cũ dùng cú pháp `http.begin(String)` đã bị "khai tử" trên ESP8266 Core mới. Tôi đã thêm đối tượng `WiFiClient` để hệ thống đẩy data lên PC bình thường.
