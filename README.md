# Snowman-MonitoreoRefrigeracion-ESP32
Sistema de monitoreo de contenedores de refrigeración de la empresa Snowman

Obtiene la temperatura y humedad del contenedor y la muestra en una página web local, asimismo almacena dicha informacion en una base en Firebase

### LIBRERIAS

Librerías necesarías en el Arduino IDE con ESP32

- Firebase-ESP-Client: https://github.com/mobizt/Firebase-ESP-Client
- ArduinoJSON 5.13 o inferior: https://github.com/bblanchon/ArduinoJson
- IOXESP32MOTOR: https://github.com/ArtronShop/IOXESP32Motor

### DIAGRAMA 

![](https://github.com/lyespinosa/Snowman-MonitoreoRefrigeracion-ESP32/blob/main/diagram-esp32.png)


### VISTA PREVIA DEL CÓDIGO

```c++

#include <IOXhop_FirebaseESP32.h>
#include <WiFi.h>
#include <WiFiClient.h>
#include <WebServer.h>
#include <ESPmDNS.h>
#include <DHT.h>

#include <NTPClient.h>
#include <WiFiUdp.h>

#define FIREBASE_AUTH "https://esp32-dht11-snowman-default-rtdb.firebaseio.com/"
#define FIREBASE_HOST "0dadqkpNnn8fS1m21slJmzDcqO8jQUAnWGbky8Co"

const char *ssid = "****";
const char *password = "*****";

const char* timezone = "CST6CDT,M3.2.0/2,M11.1.0/2"; 

WebServer server(80);
DHT dht(26, DHT11);

void handleRoot() {
  char msg[1500];

  snprintf(msg, 1500,
           "<html>\
  <head>\
    <meta http-equiv='refresh' content='4'/>\
    <meta name='viewport' content='width=device-width, initial-scale=1'>\
    <link rel='stylesheet' href='https://use.fontawesome.com/releases/v5.7.2/css/all.css' integrity='sha384-fnmOCqbTlWIlj8LyTjo7mOUStjsKC4pOpQbqyi7RrhN7udi9RwhKkMHpvLbHG9Sr' crossorigin='anonymous'>\
    <title>Panel de control</title>\
    <style>\
    html { font-family: Arial; display: inline-block; margin: 0px auto; text-align: center;}\
    .container { display: flex; flex-direction: column; align-items: center }\
    h2 { font-size: 3.0rem; }\
    p { font-size: 3.0rem; }\
    .datos { border: solid 1px #3e3e3e; border-radius: 5px; padding: 10px; background-color: #e6e6e6 }\
    .units { font-size: 1.2rem; }\
    .dht-labels{ font-size: 1.5rem; vertical-align:middle; padding-bottom: 15px;}\
    </style>\
  </head>\
  <body>\
   <div class='container'>\
      <h2>Informacion del contenedor</h2>\
      <div class='datos'>\
      <p>\
         <i class='fas fa-thermometer-three-quarters' style='color:#177eca;'></i>\
        <span class='dht-labels'>Temperatura</span>\
        <span>%.2f</span>\
        <sup class='units'>&deg;C</sup>\
      </p>\
      <p>\
        <i class='fas fa-tint' style='color:#00add6;'></i>\
        <span class='dht-labels'>Humedad</span>\
        <span>%.2f</span>\
        <sup class='units'>&percnt;</sup>\
      </p>\
      </div>\
      </div>\
  </body>\
</html>",
           readDHTTemperature(), readDHTHumidity()
          );
  server.send(200, "text/html", msg);
}

void setup(void) {

  Serial.begin(115200);
  dht.begin();
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  Serial.println("");

  // Wait for connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.print("Connected to ");
  Serial.println(ssid);
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  Firebase.begin(FIREBASE_AUTH, FIREBASE_HOST);

  configTime(-6 * 3600, 0, "pool.ntp.org");

  if (MDNS.begin("esp32")) {
    Serial.println("MDNS responder started");
  }
  server.on("/", handleRoot);

  server.begin();
  Serial.println("HTTP server started");
}

void loop(void) {

  time_t now = time(nullptr);

  String estado = "", mensaje = ""; 

  char dateTimeString[20];
  strftime(dateTimeString, 20, "%Y-%m-%d %H:%M:%S", localtime(&now));

  Serial.print("Humedad:");
    Serial.print(readDHTHumidity());
    Serial.print(" %\t  -- ");

    Serial.print("Temperatura:");
    Serial.print(readDHTTemperature());
    Serial.println(" *C");

  if(readDHTTemperature() >= 50){ 
    mensaje = "¡Alert! temperatura del congelador muy alta";
    estado = "Erroneo";

  }
  else if(readDHTHumidity() <= 25){
    mensaje = "¡Alert! la humedad del congelador es muy baja";
    estado = "Erroneo";
  }
  else{
   mensaje = "Temperatura y humedad del congelador correctos";
   estado = "Correcto";
  }

  String jsonData = "{ \"Data\": { \"Fecha\": \"" + String(dateTimeString) + "\", \"Temperatura\": " + String(readDHTTemperature()) + "\", \"Humedad\": " + String(readDHTHumidity()) + "\", \"Estado\": " + estadoDHT() + "\", \"Mensaje\": " + mensajeDHT() + " } }";

  Firebase.pushString("Data:", jsonData);

   if(Firebase.failed()){
    Serial.print("Configuracion erronea");
    Serial.println(Firebase.error());
    return;        
  }


  Serial.println("Datos de temperatura y humedad enviados");

  server.handleClient();
  delay(1000 * 3);
}


float readDHTTemperature() {
  // Sensor readings may also be up to 2 seconds
  // Read temperature as Celsius (the default)
  float t = dht.readTemperature();
  if (isnan(t)) {    
    Serial.println("Failed to read from DHT sensor!");
    return -1;
  }
  else {
    return t;
  }
}

float readDHTHumidity() {
  // Sensor readings may also be up to 2 seconds
  float h = dht.readHumidity();
  if (isnan(h)) {
    Serial.println("Failed to read from DHT sensor!");
    return -1;
  }
  else {
    return h;
  }
}

String estadoDHT(){
  if(readDHTTemperature() >= 50){ 
    return "Erroneo";
  }
  else if(readDHTHumidity() <= 25){
    return "Erroneo";
  }
  else{
   return "Correcto";
  }

}


String mensajeDHT(){
  if(readDHTTemperature() >= 50){ 
    return "¡Alert! temperatura del congelador muy alta";
  }
  else if(readDHTHumidity() <= 25){
    return "¡Alert! la humedad del congelador es muy baja";
  }
  else{
   return "Temperatura y humedad del congelador correctos";
  }

}

```

