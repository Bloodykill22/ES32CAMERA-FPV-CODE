Used for my simple esp32 camera fpv. 
Upload, connect to the esp accespoint from the wifi setting
Type 192.168.4.1/stream into a browser search bar
Enjoy

#include "esp_camera.h"
#include <WiFi.h>
#include "esp_wifi.h"


#define CAMERA_MODEL_AI_THINKER

#if defined(CAMERA_MODEL_AI_THINKER)
  #define PWDN_GPIO_NUM     32
  #define RESET_GPIO_NUM    -1
  #define XCLK_GPIO_NUM      0
  #define SIOD_GPIO_NUM     26
  #define SIOC_GPIO_NUM     27
  
  #define Y9_GPIO_NUM       35
  #define Y8_GPIO_NUM       34
  #define Y7_GPIO_NUM       39
  #define Y6_GPIO_NUM       36
  #define Y5_GPIO_NUM       21
  #define Y4_GPIO_NUM       19
  #define Y3_GPIO_NUM       18
  #define Y2_GPIO_NUM        5
  #define VSYNC_GPIO_NUM    25
  #define HREF_GPIO_NUM     23
  #define PCLK_GPIO_NUM     22
#endif


#define FLASH_GPIO_NUM       4

// Access Point Settings
const char* ssid = "FPV_CAM_BY_DIYHONK";
const char* password = ""; // 

WiFiServer streamServer(80);


void handleStream(WiFiClient &client) {
  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: multipart/x-mixed-replace; boundary=frame");
  client.println("Access-Control-Allow-Origin: *");
  client.println();

  while (client.connected()) {
    camera_fb_t *fb = esp_camera_fb_get();
    if (!fb) {
      break;
    }

    client.println("--frame");
    client.println("Content-Type: image/jpeg");
    client.printf("Content-Length: %u\r\n\r\n", fb->len);
    client.write(fb->buf, fb->len);
    client.println();

    esp_camera_fb_return(fb);

    taskYIELD();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("\nStarting ESP32-CAM AP...");

  pinMode(FLASH_GPIO_NUM, OUTPUT);
  digitalWrite(FLASH_GPIO_NUM, HIGH);

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;

  config.xclk_freq_hz = 25000000;         // 20 MHz Clock (stable & fast)
  config.pixel_format = PIXFORMAT_JPEG;   // JPEG Compression
  config.frame_size = FRAMESIZE_VGA;      // 640x480 (480p)
  config.jpeg_quality = 22;               // Quality scale (0-63)
  config.fb_count = 1;                    // Single buffer to eliminate lag/queueing
  config.grab_mode = CAMERA_GRAB_LATEST;  // Always grab fresh frame

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    return;
  }

  sensor_t *s = esp_camera_sensor_get();
  if (s) {
    s->set_vflip(s, 0);       
    s->set_hmirror(s, 0);     
  }

  WiFi.mode(WIFI_AP);
  WiFi.softAP(ssid, password);
  
  esp_wifi_set_ps(WIFI_PS_NONE);
  
  WiFi.setTxPower(WIFI_POWER_19_5dBm);

  streamServer.begin();
  Serial.println("AP Started. Connect to 'camera' and open http://192.168.4.1/stream");
}

void loop() {
  WiFiClient client = streamServer.available();
  if (client) {
    unsigned long timeout = millis() + 2000;
    while (client.connected() && !client.available() && millis() < timeout) {
      delay(1);
    }
    
    if (client.available()) {
      String req = client.readStringUntil('\r');
      client.flush();

      if (req.indexOf("GET /stream") != -1 || req.indexOf("GET / ") != -1) {
        handleStream(client);
      } else {
        client.println("HTTP/1.1 404 Not Found");
        client.println("Content-Type: text/plain");
        client.println("Connection: close");
        client.println();
        client.println("404 Not Found");
      }
    }
    client.stop();
  }
}
