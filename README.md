# LAB4_MICROS

/*
  CONECTIVIDAD WIFI - ESP32
  - Servidor UDP: envía temperatura, humedad y luminosidad al celular (>= 4 Hz)
  - Recibe comandos UDP para controlar 4 LEDs (actuadores ON/OFF)
  - Servidor WEB (HTTP, puerto 80): página HTML para ver sensores y controlar LEDs
    desde cualquier navegador conectado a la misma red WiFi.

  Librerías necesarias (Arduino IDE -> Herramientas -> Administrar bibliotecas):
    - "DHT sensor library" (Adafruit)
    - "Adafruit Unified Sensor" (dependencia de la anterior)
    - "WebServer" (viene incluida con el core de ESP32, no hay que instalarla)
*/

#include <WiFi.h>
#include <WiFiUdp.h>
#include <DHT.h>
#include <WebServer.h>

// ---------- CONFIGURACIÓN WIFI ----------
const char* ssid     = "DESKTOP-1OC30LP 2478";
const char* password = "269q8$B7";


// IP del celular al que se le van a enviar los datos por UDP (opcional, ver nota abajo)
IPAddress phoneIP(192, 168, 1, 100);   // <-- CAMBIA esto por la IP real de tu celular
const unsigned int remotePort = 4210;  // puerto en el que la app del celular escucha
const unsigned int localPort  = 4210;  // puerto en el que el ESP32 escucha comandos UDP

WiFiUDP udp;
WebServer server(80);   // servidor web en el puerto 80 (http://IP_DEL_ESP32/)

// ---------- SENSORES ----------
#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

#define LDR_PIN 34   // pin solo-entrada (ADC1)

// ---------- ACTUADORES (LEDs) ----------
const int NUM_LEDS = 4;
const int ledPins[NUM_LEDS] = {16, 17, 18, 19};
bool ledState[NUM_LEDS] = {false, false, false, false};

// ---------- ÚLTIMA LECTURA (para la página web) ----------
float lastTemp = 0;
float lastHum  = 0;
int   lastLuz  = 0;

// ---------- TEMPORIZACIÓN ----------
unsigned long lastSend = 0;
const unsigned long sendInterval = 200; // 200 ms = 5 Hz (cumple el mínimo de 4 Hz)

// ---------- PÁGINA HTML (queda guardada en la memoria flash, no en RAM) ----------
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>ESP32 - Panel</title>
<style>
  body { font-family: Arial, sans-serif; background:#0f1115; color:#eee; margin:0; padding:20px; }
  h1 { text-align:center; font-size:22px; margin-bottom:20px; }
  .cards { display:flex; gap:12px; justify-content:center; flex-wrap:wrap; margin-bottom:28px; }
  .card { background:#1b1f27; border-radius:12px; padding:16px 22px; text-align:center; min-width:100px; }
  .card .label { font-size:13px; color:#9aa0ab; }
  .card .value { font-size:26px; font-weight:bold; margin-top:6px; }
  .leds { display:flex; gap:14px; justify-content:center; flex-wrap:wrap; }
  .led-btn { padding:16px 20px; border-radius:12px; border:none; font-size:16px; font-weight:bold;
             cursor:pointer; min-width:100px; background:#2a2f3a; color:#eee; transition:background .15s; }
  .led-btn.on { background:#2fae4e; color:#fff; }
</style>
</head>
<body>
  <h1>Panel ESP32</h1>
  <div class="cards">
    <div class="card"><div class="label">Temperatura</div><div class="value" id="temp">--</div></div>
    <div class="card"><div class="label">Humedad</div><div class="value" id="hum">--</div></div>
    <div class="card"><div class="label">Luz</div><div class="value" id="luz">--</div></div>
  </div>
  <div class="leds">
    <button class="led-btn" id="led1" onclick="toggleLed(1)">LED 1</button>
    <button class="led-btn" id="led2" onclick="toggleLed(2)">LED 2</button>
    <button class="led-btn" id="led3" onclick="toggleLed(3)">LED 3</button>
    <button class="led-btn" id="led4" onclick="toggleLed(4)">LED 4</button>
  </div>

<script>
let ledStates = [false, false, false, false];

function refreshData() {
  fetch('/data').then(r => r.json()).then(d => {
    document.getElementById('temp').textContent = d.temp.toFixed(1) + ' C';
    document.getElementById('hum').textContent  = d.hum.toFixed(1) + ' %';
    document.getElementById('luz').textContent  = d.luz;
    ledStates = d.leds;
    for (let i = 0; i < 4; i++) {
      const btn = document.getElementById('led' + (i + 1));
      btn.classList.toggle('on', ledStates[i]);
    }
  }).catch(() => {});
}

function toggleLed(num) {
  const nuevoValor = ledStates[num - 1] ? 0 : 1;
  fetch('/led?num=' + num + '&val=' + nuevoValor).then(refreshData);
}

setInterval(refreshData, 1000);
refreshData();
</script>
</body>
</html>
)rawliteral";

void setup() {
  Serial.begin(115200);

  // Configura los LEDs como salida, iniciando apagados
  for (int i = 0; i < NUM_LEDS; i++) {
    pinMode(ledPins[i], OUTPUT);
    digitalWrite(ledPins[i], LOW);
  }

  dht.begin();

  // Conexión WiFi
  WiFi.begin(ssid, password);
  Serial.print("Conectando a WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("Conectado. IP del ESP32: ");
  Serial.println(WiFi.localIP());
  Serial.println("Abre esa IP en el navegador para ver el panel web.");

  // ---- UDP (opcional, para app en el celular) ----
  udp.begin(localPort);
  Serial.print("Escuchando comandos UDP en puerto ");
  Serial.println(localPort);

  // ---- Servidor WEB ----
  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.on("/led", handleLedHttp);
  server.begin();
  Serial.println("Servidor web iniciado en el puerto 80");
}

void loop() {
  server.handleClient();     // atiende peticiones del navegador
  recibirComandos();         // atiende comandos UDP

  if (millis() - lastSend >= sendInterval) {
    lastSend = millis();
    enviarDatosSensores();
  }
}

// ---------------- SENSORES / UDP ----------------

void enviarDatosSensores() {
  float temp = dht.readTemperature();
  float hum  = dht.readHumidity();
  int   luz  = analogRead(LDR_PIN); // 0 a 4095

  if (isnan(temp) || isnan(hum)) {
    Serial.println("Error leyendo el DHT11");
    return;
  }

  // Guarda la última lectura para que la página web la pueda mostrar
  lastTemp = temp;
  lastHum  = hum;
  lastLuz  = luz;

  String mensaje = String(temp, 1) + "," + String(hum, 1) + "," + String(luz);
  udp.beginPacket(phoneIP, remotePort);
  udp.print(mensaje);
  udp.endPacket();

  Serial.println("Enviado: " + mensaje);
}

void recibirComandos() {
  int packetSize = udp.parsePacket();
  if (packetSize == 0) return;

  char buffer[64];
  int len = udp.read(buffer, sizeof(buffer) - 1);
  if (len <= 0) return;
  buffer[len] = '\0';

  String comando = String(buffer);
  comando.trim();
  Serial.println("Comando recibido (UDP): " + comando);

  if (comando.startsWith("LED")) {
    int ledNum = comando.substring(3, 4).toInt();
    int separador = comando.indexOf(':');
    if (separador == -1) return;
    int valor = comando.substring(separador + 1).toInt();
    setLed(ledNum, valor == 1);
  }
}

// ---------------- SERVIDOR WEB ----------------

void handleRoot() {
  server.send_P(200, "text/html", index_html);
}

void handleData() {
  // Arma un JSON simple con la última lectura y el estado de los LEDs
  String json = "{";
  json += "\"temp\":" + String(lastTemp, 1) + ",";
  json += "\"hum\":" + String(lastHum, 1) + ",";
  json += "\"luz\":" + String(lastLuz) + ",";
  json += "\"leds\":[";
  for (int i = 0; i < NUM_LEDS; i++) {
    json += ledState[i] ? "true" : "false";
    if (i < NUM_LEDS - 1) json += ",";
  }
  json += "]}";
  server.sendHeader("Access-Control-Allow-Origin", "*"); // permite que un HTML abierto localmente pueda leer esto
  server.send(200, "application/json", json);
}

void handleLedHttp() {
  // Ejemplo de petición: /led?num=1&val=1
  if (!server.hasArg("num") || !server.hasArg("val")) {
    server.send(400, "text/plain", "Faltan parámetros num y val");
    return;
  }
  int ledNum = server.arg("num").toInt();
  int valor  = server.arg("val").toInt();
  setLed(ledNum, valor == 1);
  server.sendHeader("Access-Control-Allow-Origin", "*"); // permite que un HTML abierto localmente pueda controlar los LEDs
  server.send(200, "text/plain", "OK");
}

// ---------------- FUNCIÓN COMÚN PARA ENCENDER/APAGAR LEDS ----------------

void setLed(int ledNum, bool encendido) {
  if (ledNum >= 1 && ledNum <= NUM_LEDS) {
    int idx = ledNum - 1;
    ledState[idx] = encendido;
    digitalWrite(ledPins[idx], encendido ? HIGH : LOW);
  }
}
