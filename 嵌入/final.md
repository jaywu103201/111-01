```arduino
/*
 * Create a Web Server (with WebSockets) Using an ESP32 in Arduino with WiFi AP mode
 * https://shawnhymel.com/1882/how-to-create-a-web-server-with-websockets-using-an-esp32-in-arduino/
 * 
 * Note: 
 * SPIFFS file names can be 32 bytes max (including starting slash and zero at the end).
 * To setup the web server, we will need two libraries. 
 * 1. The first one is the 'ESPAsyncWebServer', which we will use in our code.
 * https://github.com/me-no-dev/ESPAsyncWebServer
 * 2. The second library needed is the AsyncTCP, which is a dependency for the previous one. 
 * This library is an asynchronous TCP library for the ESP32 and it is the base for the ESPAsyncWebServer library implementation
 * https://github.com/me-no-dev/AsyncTCP
 *
 *Libraries:
  Arduino_JSON library by Arduino version 0.2.0 (Arduino Library Manager)
  ESPAsyncWebServer (.zip folder);
  AsyncTCP (.zip folder).
 * 
 * async_websocket_server_html_iot
 |_ async_websocket_server_html_iot.ino
 |_ data
     |_ index.html
     |_ static
         |_ css
              |_index.css
         |_ js ...
 * 
  DHT sensor: 
  https://github.com/adafruit/DHT-sensor-library
  Server-Sent-Events:
  https://www.theengineeringprojects.com/2022/03/server-sent-events-with-esp32-and-dht11.html
  https://randomnerdtutorials.com/esp32-web-server-gauges/
 */
#include <Arduino.h>
#include <WiFi.h>
#include <SPIFFS.h>
#include <AsyncTCP.h>
#include <ESPAsyncWebServer.h>
#include <Arduino_JSON.h>
#include <DHT.h>

// STA mode: Replace with your network credentials
const char* ssid = "LAB_I3301";//"LAB_I3301"; // "I2313";"HomeResort3F";
const char* password = "bdt4fC2oZc";//"bdt4fC2oZc"; //"0928474616";"j0928474616g";
const int http_port = 80;

// DHT sensor pin and LED GPIO pins
#define dhtPin 27     // Digital pin connected to the DHT sensor
#define DHTTYPE DHT11     // types: DHT 11, DHT22, DHT21
byte ledPins[] = {2, 4, 16};  //  on-board LED pins

// LED states and message
bool ledStates[] = {0, 0, 0};
int arrayLen = sizeof(ledPins)/sizeof(ledPins[0]);
String message = "";

// Timer variables for DHT
unsigned long lastTime = 0;
unsigned long timerDelay = 2000; //2 sec timer delay

//Json Variable to Hold LED status
JSONVar ledStates_dict, dhtData_dict, ledPins_dict;
DHT dht(dhtPin, DHTTYPE);

// Create AsyncWebServer object on port 80
AsyncWebServer server(http_port);
// The ESPAsyncWebServer library includes a WebSocket plugin
// Create an AsyncWebSocket object: ws to handle the connections on the "/ws" path in the js inside of html.
AsyncWebSocket ws("/ws");
// Create an Event Source on /events to its counter part in sse_dht.js
AsyncEventSource events("/events");

/**
 * Main
 */
void setup() {
  // Start Serial port
  Serial.begin(115200);
  // Init LED and turn off
  for (int i=0; i<arrayLen; i++){
    pinMode(ledPins[i], OUTPUT);
    digitalWrite(ledPins[i], LOW);
  }
  
  initFS();
  initWiFi();
  dht.begin();

  // Register the route “/” to the web server and will only listen to 'HTTP GET' requests.
  // On HTTP request for root '/', provide index.html file
  // this tells WEB Server to send the files request by the browser that are contained in the SPIFFS Filesystem 
  // such as images, style sheets, JavaScript sources, etc. This means we don't have to deal with mime types of files requested.
  // Files saved in SPIFFS Filesystem can be revisited in the Cache memory within 600 secs rather than in the SPIFFS Filesystem.
  //server.serveStatic("/", SPIFFS, "/").setDefaultFile("index.html").setCacheControl("max-age=600");
  server.serveStatic("/", SPIFFS, "/").setDefaultFile("index.html");

  // SSE: Request for the latest sensor readings from the client (ssh_dht.js) when the web (iot.html) is first loading
  server.on("/readings", HTTP_GET, [](AsyncWebServerRequest *request){
    Serial.println(String() + "WebEvent: Requested from the web for the first loading!");
    String json = readDHTData();
    request->send(200, "application/json", json);
    json = String();
  });

  // Handle requests for pages that do not exist
  server.onNotFound(onPageNotFound);

  // Start web server
  server.begin();

  // initializes WebSocket connection on the gateway 
  initWebSocket();

  // initialize WebEvent connection on the gateway
  initWebEvent();
}

void loop() {
  // update LED status based on websocket with the client index.js
  for (int i=0; i<arrayLen; i++){
    digitalWrite(ledPins[i], ledStates[i]);
  }

  // update DHT data based on webevents by Server-sent-events (SSE) with the client iot.js
  // Send Events to the client with the Sensor Readings Every 2 seconds
  
  unsigned long delta = millis() - lastTime;
  byte time_interval = int(delta/1000);
  if (delta > timerDelay) {
    events.send("ping", NULL, millis()); // to check on the client side that the server is alive
    Serial.println(String() + "WebEvent: updating DHT data every " + time_interval + " seconds");
    // SSE with the name of the events: "new_readings", corresponding to the "new_readings" event in the client js
    // The send() method accepts a variable of type char, so we need to use the c_str() method to 
    // convert the variable. The name of the events is new_readings.
    events.send(readDHTData().c_str(), "web_event_new_readings", millis());
    lastTime = millis();  // update the lastTime
  }
  ws.cleanupClients(); 
}

/***********************************************************
 * Functions
 */
// Initialize SPIFFS
void initFS() {
  if (!SPIFFS.begin()) {
    Serial.println("An error has occurred while mounting SPIFFS");
  }
  else{
   Serial.println("SPIFFS mounted successfully");
  }
}

// Initialize WiFi
void initWiFi() {
  // default ip address for AP mode: 192.168.4.1
  // configurate static ip address for STA mode: check the local ip 
  IPAddress local_IP(192, 168, 3, random(2,254));  
  IPAddress gateway(192, 168, 3, 1);
  IPAddress subnet(255, 255, 255, 0);
  IPAddress primaryDNS(192, 168, 3, 254);;   //optional
  IPAddress secondaryDNS(192, 168, 3, 254);;   //optional
  WiFi.mode(WIFI_STA);

  // STA mode: Configures static IP address
  if (!WiFi.config(local_IP, gateway, subnet, primaryDNS, secondaryDNS)) {
    Serial.println("STA Failed to configure");
  }
  
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi ..");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print('.');
    delay(1000);
  }
  Serial.println(WiFi.localIP());
}
 
//  initializes a WebSocket connection on the gateway
void initWebSocket() {
  ws.onEvent(onEvent); // callback: the websocket events handling function
  server.addHandler(&ws); // add websocket into the server
}

// initialize WebEvent 
void initWebEvent() {
  
  //Set up the event source on the server.
  events.onConnect([](AsyncEventSourceClient *client){
    if (client->lastId()){
      Serial.printf("WebEvent: Client reconnected! Last message ID that it got is: %u\n", client->lastId());
    }
    // send event with message "hello!", id current millis and set reconnect delay to 1 second
    client->send("WebEvent: hello from ESP32 server!", NULL, millis(), 1000);
  });
  server.addHandler(&events); // add event into the server
}
// websocket: the websocket events handling events
void onEvent(AsyncWebSocket *server, AsyncWebSocketClient *client, AwsEventType type,
             void *arg, uint8_t *data, size_t len) {
  //  client: the object we will use to send data back to the client, when it connects to our server.
  //  type: an enumerated value. This enum contains all the websocket events that can trigger the execution of our handling function.
  // we use this third argument to check what event has occurred and handle it accordingly.
            
  switch (type) {
    // the event corresponds to the connection of a client
    // we have access to a pointer to the client object, then we need to use the '->' 
    // operator in order to access the text/id/remoteIP methods on our client object
    case WS_EVT_CONNECT:
      client->text("WebSocket: Hello from ESP32 Server!"); // appear on console in terms of websock.onmessage
      //IPAddress ip = client->remoteIP();
      Serial.printf("WebSocket: WebSocket client #%u is connected from %s\n", client->id(), client->remoteIP().toString().c_str());
      // send LED pins to the web page once
      notifyClients(getLEDPins());
      break;  
    case WS_EVT_DISCONNECT:
      Serial.printf("WebSocket: WebSocket client #%u disconnected\n", client->id());
      break;
    case WS_EVT_DATA:
      handleWebSocketMessage(arg, data, len);
      break;
    case WS_EVT_PONG:
      break;
    case WS_EVT_ERROR:
      break;
  }
}

// websocket: the websocket events handling message
// callback function if AwsEventType = WS_EVT_DATA
void handleWebSocketMessage(void *arg, uint8_t *data, size_t len) {
  AwsFrameInfo *info = (AwsFrameInfo*)arg;
  data[len] = 0;
  message = (char*)data;
  Serial.println(String() + "WebSocket: Received data from client witout conditions: " + message);

  // toggle led
  //  search for the first instance of a particular character value in a String (indexing from 0)
  if (message.indexOf("1s") >= 0) { // 1st led
    Serial.println(String() + "WebSocket: Received data from client filtered with conditions: " + message);
    //ledStates[0] = !ledStates[0];  // led_state = led_state ? 0 : 1;
    ledStates[0] = message.substring(2).toInt();
    Serial.printf("\nWebSocket: The server toggles LED-1 to %u\n", ledStates[0]);
    //digitalWrite(ledPins[0], ledStates[0]);
    // notify all clients by calling the notifyClients() function. 
    // This way, all clients are notified of the change and update the interface accordingly.
    notifyClients(getLEDStatus());
  } else if (message.indexOf("2s") >=0) { // 2nd led
    Serial.println(String() + "WebSocket: Received data from client filtered with conditions: " + message);
    //ledStates[1] = !ledStates[1];  // led_state = led_state ? 0 : 1;
    ledStates[1] = message.substring(2).toInt();
    Serial.printf("\nWebSocket: The server toggles LED-2 to %u\n", ledStates[0]);
    notifyClients(getLEDStatus());
  } else if (message.indexOf("3s") >=0) { // 2rd led
    Serial.println(String() + "WebSocket: Received data from client filtered with conditions: " + message);
    //ledStates[1] = !ledStates[1];  // led_state = led_state ? 0 : 1;
    ledStates[2] = message.substring(2).toInt();
    Serial.printf("\nWebSocket: The server toggles LED-3 to %u\n", ledStates[0]);
    notifyClients(getLEDStatus());
  } else if (strcmp((char*)data, "ledInit") == 0) {
    notifyClients(getLEDStatus());
  } else {
    Serial.printf("\nWebSocket: Message not recognized: %u\n", message);
  }
}

// websocket: notifies all clients with a message containing whatever you pass as a argument. 
void notifyClients(String ledStates) {
  // In this case, we’ll want to notify all clients (inc. the web) of the current LED state whenever there’s a change.
  // event ("1" or "0") is received by websocket.onmessage() on the js of the web.
  ws.textAll(ledStates);
  //ws.textAll("New event!");
}

//Get led status: creates a JSON string with the current led status
/* e.g.
{
  ledState1: 1;
  ledState2: 0;
}*/
String getLEDPins(){
  ledPins_dict["ledState1"] = String() + "LED #1 - GPIO " + ledPins[0];
  ledPins_dict["ledState2"] = String() + "LED #2 - GPIO " + ledPins[1];
  ledPins_dict["ledState3"] = String() + "LED #3 - GPIO " + ledPins[2];
  
  String jsonString = JSON.stringify(ledPins_dict);
  return jsonString;      
}

// websocket: generate json string for led status
String getLEDStatus(){
  ledStates_dict["ledState1"] = String(ledStates[0]);
  ledStates_dict["ledState2"] = String(ledStates[1]);
  ledStates_dict["ledState3"] = String(ledStates[2]);

  String jsonString = JSON.stringify(ledStates_dict);
  return jsonString;
}

// webevent: generate json string for DHT data
String getDHTData(float t, float h){
  dhtData_dict["Temperature"] = String(t);
  dhtData_dict["Humidity"] = String(h);
  dhtData_dict["RandomValue"] = String(random(1,100));
  dhtData_dict["Time"] = String( millis());
  
  String jsonString = JSON.stringify(dhtData_dict);
  return jsonString;
}

// initialize async http
// AsyncWebServer Callback: send homepage
// AsyncWebServer Callback: send 404 if requested file does not exist
void onPageNotFound(AsyncWebServerRequest *request) {
  IPAddress remote_ip = request->client()->remoteIP();
  Serial.println("[" + remote_ip.toString() + "] HTTP GET request of " + request->url());
  request->send(404, "text/plain", "Not found");
}

// On the web page, there is a placeholder/template for the GPIO state. 
// The placeholder for the GPIO state is written directly in the HTML file between % signs, for example %BUTTONSPLACEHOLDER%.
// when is return, this function replaces placeholder in the html with LED state value
/*
String processor(const String& var) {
  Serial.println(String() + "-----begin template functin to process template: " + var);
  String buttonInfo = String();
 
  if (var == "led1"){
      buttonInfo = String() + "LED #1 - GPIO " + ledPins[0];
  } 
  if (var=="led2") {
      buttonInfo = String() + "LED #2 - GPIO " + ledPins[1];
  } 
  if (var=="led3"){
      buttonInfo = String() + "LED #3 - GPIO " + ledPins[2];      
  }
  Serial.println(buttonInfo);
  return buttonInfo;
}*/

// webevent: read DHT data
String readDHTData() {
  // Sensor readings may also be up to 2 seconds 'old' (its a very slow sensor)
  // Read temperature as Celsius (the default)
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  // Read temperature as Fahrenheit (isFahrenheit = true)
  //float t = dht.readTemperature(true);
  // Check if any reads failed and exit early (to try again).
  if (isnan(t)||isnan(h)) {    
    Serial.println("Failed to read from DHT sensor!");
    t=0;
    h=0;
  }
  String dhtData_json = getDHTData(t, h);
  Serial.println(String() + "WebEvent: reading temperature... " + t + "°C");
  Serial.println(String() + "WebEvent: reading himidity ... " + h + "%");
  return dhtData_json;
}
```
