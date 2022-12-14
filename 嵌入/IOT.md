```arduino
#include<WiFi.h>
#include<SPIFFS.h>
#include<AsyncTCP.h>
#include<ESPAsyncWebServer.h>

const char* ssid ="LAB_I3301";
const char* password = "bdt4fC2oZc";
const int http_port = 80;
AsyncWebServer server(http_port);

void setup() {

  // put your setup code here, to run once:
  Serial.begin(115200);
  initFS();
  initWiFi();
  server.serveStatic("/",SPIFFS,"/").setDefaultFile("index.html").setCacheControl("max-age=600");
  server.onNotFound(onPageNotFound);
  server.begin();
}
void loop() {
}



void initFS(){
  if(!SPIFFS.begin()){
    Serial.println("An error has occurred while mounting SPIFFS");
  }else{
    Serial.println("SPIFFS mounted successfully");
  }
}


void initWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.print("Connecting to wifi");
  while(WiFi.status()!=WL_CONNECTED){
    Serial.print('.');
    delay(1000);
  }
  Serial.println(WiFi.localIP());
}


void onPageNotFound(AsyncWebServerRequest *request){
  IPAddress remote_ip = request->client()->remoteIP();
  Serial.println("["+ remote_ip.toString()+"] HTTP GET request of" + request->url());
  request->send(404, "text/plain","Not Found");
}


```
