#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>

const char* ssid = "Galaxy";
const char* password = "abh.";

WebServer server(80);
Servo radarServo;

// Pins
#define TRIG_PIN 5
#define ECHO_PIN 4
#define SERVO_PIN 18

int angle = 0;
int distance = 0;

void setup() {
  Serial.begin(115200);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  radarServo.setPeriodHertz(50); // 50Hz for SG90
  radarServo.attach(SERVO_PIN, 500, 2400); // Servo pulse width

  // Connect to WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected!");
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP());

  // Web routes
  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.begin();
  Serial.println("Web server started.");
}

void loop() {
  server.handleClient();
}

// Serve enhanced radar web UI
void handleRoot() {
  server.send(200, "text/html", R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>ESP32 Radar</title>
  <style>
    body {
      margin: 0;
      background: #000;
      color: #0f0;
      font-family: 'Courier New', monospace;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    h2 {
      margin-top: 20px;
    }
    canvas {
      background: radial-gradient(circle, #003300 0%, #000000 80%);
      border: 2px solid #0f0;
      border-radius: 50%;
      margin-top: 20px;
      box-shadow: 0 0 20px #0f0;
    }
  </style>
</head>
<body>
  <h2>ESP32 Realistic Radar</h2>
  <canvas id="radarCanvas" width="400" height="400"></canvas>

  <script>
    const canvas = document.getElementById("radarCanvas");
    const ctx = canvas.getContext("2d");
    const centerX = canvas.width / 2;
    const centerY = canvas.height / 2;
    const maxRadius = canvas.width / 2;

    function drawBackground() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      ctx.strokeStyle = "#0f0";
      ctx.lineWidth = 1;

      // Concentric circles
      for (let i = 1; i <= 4; i++) {
        ctx.beginPath();
        ctx.arc(centerX, centerY, (maxRadius / 4) * i, 0, 2 * Math.PI);
        ctx.stroke();
      }

      // Radial lines every 30°
      for (let a = 0; a < 360; a += 30) {
        let rad = a * Math.PI / 180;
        let x = centerX + maxRadius * Math.cos(rad);
        let y = centerY + maxRadius * Math.sin(rad);
        ctx.beginPath();
        ctx.moveTo(centerX, centerY);
        ctx.lineTo(x, y);
        ctx.stroke();
      }
    }

    function drawSweep(angle, distance) {
      drawBackground();
      let rad = angle * Math.PI / 180;
      let x = centerX + maxRadius * Math.cos(rad);
      let y = centerY - maxRadius * Math.sin(rad);

      // Sweep line
      ctx.strokeStyle = "lime";
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(centerX, centerY);
      ctx.lineTo(x, y);
      ctx.stroke();

      // Distance dot
      if (distance > 0) {
        let dx = distance * 2 * Math.cos(rad);
        let dy = distance * 2 * Math.sin(rad);
        let dotX = centerX + dx;
        let dotY = centerY - dy;

        ctx.fillStyle = "#0f0";
        ctx.beginPath();
        ctx.arc(dotX, dotY, 5, 0, 2 * Math.PI);
        ctx.fill();
      }
    }

    async function fetchData() {
      const res = await fetch("/data");
      const json = await res.json();
      drawSweep(json.angle, json.distance);
    }

    setInterval(fetchData, 200);
    drawBackground();
  </script>
</body>
</html>
  )rawliteral");
}

// Return distance + angle as JSON
void handleData() {
  distance = getAveragedDistance(5);
  angle += 5;
  if (angle > 180) angle = 0;

  radarServo.write(angle);
  delay(100);

  String json = "{\"angle\":" + String(angle) + ",\"distance\":" + String(distance) + "}";
  server.send(200, "application/json", json);
}

// Average distance for accuracy
int getAveragedDistance(int samples) {
  long total = 0;
  int validReadings = 0;
  for (int i = 0; i < samples; i++) {
    int d = getDistance();
    if (d > 0) {
      total += d;
      validReadings++;
    }
    delay(20);
  }
  if (validReadings == 0) return 0;
  return total / validReadings;
}

// Get one ultrasonic distance
int getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 50000);
  if (duration == 0) return 0;

  int dist = duration * 0.034 / 2;
  if (dist < 2 || dist > 400) return 0;
  return dist;
}
