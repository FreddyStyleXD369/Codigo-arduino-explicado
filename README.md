# Codigo-arduino-explicado

#include <Servo.h>
// Biblioteca para controlar servomotores.

#include <Wire.h>
// Biblioteca para comunicación I2C.

#include <Adafruit_MPU6050.h>
// Biblioteca para utilizar el sensor MPU6050.

#include <Adafruit_Sensor.h>
// Biblioteca base utilizada por los sensores Adafruit.

const int ENA = 3;
// Pin PWM utilizado para controlar la velocidad del motor.

const int IN1 = 4;
// Pin de control de dirección del motor.

const int IN2 = 5;
// Segundo pin de control de dirección del motor.

const int PIN_SERVO = 9;
// Pin donde está conectado el servomotor.

const int TRIG_FRENTE = A0;
// Pin Trigger del sensor ultrasónico frontal.

const int TRIG_IZQ = A1;
// Pin Trigger del sensor ultrasónico izquierdo.

const int TRIG_DER = A2;
// Pin Trigger del sensor ultrasónico derecho.

const int TRIG_ATR = A3;
// Pin Trigger del sensor ultrasónico trasero.

const int ECHO_FRENTE = 7;
// Pin Echo del sensor ultrasónico frontal.

const int ECHO_IZQ = 8;
// Pin Echo del sensor ultrasónico izquierdo.

const int ECHO_DER = 10;
// Pin Echo del sensor ultrasónico derecho.

const int ECHO_ATR = 11;
// Pin Echo del sensor ultrasónico trasero.

const int PIN_BOTON = 12;
// Pin donde está conectado el botón de encendido/apagado.

const int PIN_HALL = 2;
// Pin donde está conectado el sensor Hall.

Servo servoVolante;
// Objeto para controlar el servomotor.

Adafruit_MPU6050 mpu;
// Objeto para controlar el sensor MPU6050.

float distFrente = 999.0;
// Almacena la distancia frontal medida.

float distIzq = 999.0;
// Almacena la distancia izquierda medida.

float distDer = 999.0;
// Almacena la distancia derecha medida.

float distAtr = 999.0;
// Almacena la distancia trasera medida.

volatile unsigned long pulsos = 0;
// Contador de pulsos del sensor Hall.

float gyroZ = 0.0;
// Guarda la velocidad angular sobre el eje Z.

bool robotActivo = false;
// Indica si el robot está habilitado para moverse.

String comandoESP32 = "";
// Almacena comandos recibidos por comunicación serial.

const int VELOCIDAD_NORMAL = 160;
// Velocidad normal de avance del motor.

const int VELOCIDAD_GIRO = 130;
// Velocidad utilizada durante maniobras de giro.

const int DIST_SEGURIDAD_CM = 20;
// Distancia mínima de seguridad en centímetros.

const int SERVO_CENTRO = 90;
// Posición central del servomotor.

const int SERVO_IZQUIERDA = 55;
// Posición de giro hacia la izquierda.

const int SERVO_DERECHA = 125;
// Posición de giro hacia la derecha.

void contadorHall() {
// Rutina de interrupción para contar pulsos del sensor Hall.

  pulsos++;
  // Incrementa el contador cada vez que se detecta un pulso.
}

float medirDistancia(int pinTrig, int pinEcho) {
// Función que calcula la distancia usando un sensor ultrasónico.

  digitalWrite(pinTrig, LOW);
  // Garantiza que el Trigger inicie apagado.

  delayMicroseconds(2);
  // Pequeña pausa de estabilización.

  digitalWrite(pinTrig, HIGH);
  // Envía el pulso ultrasónico.

  delayMicroseconds(10);
  // Mantiene el pulso durante 10 microsegundos.

  digitalWrite(pinTrig, LOW);
  // Finaliza el pulso de disparo.

  long duracion = pulseIn(pinEcho, HIGH, 25000);
  // Mide el tiempo de retorno del eco.

  if (duracion == 0) return 999.0;
  // Si no hay respuesta, devuelve una distancia muy grande.

  return duracion * 0.034 / 2.0;
  // Convierte el tiempo de vuelo en distancia en centímetros.
}

void leerUltrasonidos() {
// Actualiza las mediciones de todos los sensores ultrasónicos.

  distFrente = medirDistancia(TRIG_FRENTE, ECHO_FRENTE);

  distIzq = medirDistancia(TRIG_IZQ, ECHO_IZQ);

  distDer = medirDistancia(TRIG_DER, ECHO_DER);

  distAtr = medirDistancia(TRIG_ATR, ECHO_ATR);
}

void leerGiroscopio() {
// Obtiene la información actual del MPU6050.

  sensors_event_t a, g, temp;
  // Variables para aceleración, giroscopio y temperatura.

  mpu.getEvent(&a, &g, &temp);
  // Lee los datos actuales del sensor.

  gyroZ = g.gyro.z;
  // Guarda la velocidad angular del eje Z.
}

void moverAdelante(int velocidad) {
// Hace avanzar el motor.

  digitalWrite(IN1, HIGH);
  // Activa una dirección de giro.

  digitalWrite(IN2, LOW);
  // Desactiva la dirección opuesta.

  analogWrite(ENA, velocidad);
  // Aplica la velocidad deseada mediante PWM.
}

void moverAtras(int velocidad) {
// Hace retroceder el motor.

  digitalWrite(IN1, LOW);
  // Invierte una dirección de giro.

  digitalWrite(IN2, HIGH);
  // Activa la dirección opuesta.

  analogWrite(ENA, velocidad);
  // Aplica la velocidad deseada mediante PWM.
}

void detenerMotor() {
// Detiene completamente el motor.

  digitalWrite(IN1, LOW);
  // Desactiva la primera entrada.

  digitalWrite(IN2, LOW);
  // Desactiva la segunda entrada.

  analogWrite(ENA, 0);
  // Elimina la señal PWM.
}

void girarIzquierda() {
// Orienta las ruedas hacia la izquierda.

  servoVolante.write(SERVO_IZQUIERDA);
  // Envía la posición correspondiente al giro izquierdo.
}

void girarDerecha() {
// Orienta las ruedas hacia la derecha.

  servoVolante.write(SERVO_DERECHA);
  // Envía la posición correspondiente al giro derecho.
}

void irRecto() {
// Centra la dirección del vehículo.

  servoVolante.write(SERVO_CENTRO);
  // Coloca el servomotor en posición neutra.
}

bool hayPeligroDeChoque() {
// Verifica si existe riesgo de colisión.

  if (distFrente < DIST_SEGURIDAD_CM) return true;
  // Detecta obstáculo al frente.

  if (distIzq < DIST_SEGURIDAD_CM) return true;
  // Detecta obstáculo a la izquierda.

  if (distDer < DIST_SEGURIDAD_CM) return true;
  // Detecta obstáculo a la derecha.

  return false;
  // No se detectan obstáculos peligrosos.
}
