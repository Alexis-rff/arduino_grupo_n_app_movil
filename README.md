# arduino_grupo_n_app_movil
/*
  ============================================================
  Sistema de monitoreo y llenado automatico de estanques rurales
  Version DEMO de hardware (solo Arduino Uno R3)
  HC-SR04 + 3 LEDs de nivel + rele / bomba
  ------------------------------------------------------------
  Pines segun el diagrama detallado v2 (propuestos: validar en el montaje).
  Parametros de DEMO pensados para simular el nivel con una tapa.
  ============================================================
*/

// ================== 1. CONFIGURACION ==================

// --- Pines ---
const byte PIN_TRIG     = 9;
const byte PIN_ECHO     = 10;
const byte LED_ROJO     = 7;
const byte LED_AMARILLO = 6;
const byte LED_VERDE    = 5;
const byte PIN_RELE     = 8;

// --- Rele: true si el modulo se activa con LOW (comun en modulos azules).
//     Si al encender el Arduino la bomba parte encendida, cambia a false.
const bool RELE_ACTIVO_EN_BAJO = true;

// --- Geometria (DEMO con tapa). Con el pote real, medir y reemplazar ---
const float D_VACIO_CM  = 30.0;  // distancia sensor -> fondo        (0 %)
const float D_LLENO_CM  = 5.0;   // distancia sensor -> nivel maximo (100 %)
const float CAPACIDAD_L = 1.0;   // litros entre 0 % y 100 % (dato de demo)

// --- Umbrales de los LEDs (%) ---
const float UMBRAL_BAJO   = 20.0;  // nivel <= 20 %        -> LED rojo
const float UMBRAL_NORMAL = 60.0;  // nivel >  60 %        -> LED verde
                                   // entre ambos          -> LED amarillo
const float HISTERESIS    = 3.0;   // margen para que el LED no "titile" en el limite

// --- Bomba ---
const float PCT_ENCENDER = 20.0;   // enciende con nivel <= 20 %
const float PCT_APAGAR   = 90.0;   // apaga con nivel   >= 90 %
const unsigned long TIEMPO_MAX_BOMBA_MS = 20000UL;  // DEMO: 20 s maximo encendida
const unsigned long TIEMPO_BLOQUEO_MS   = 30000UL;  // espera tras superar el maximo

// --- Sensor ---
const float DIST_MIN_VALIDA = 2.0;            // zona ciega aprox. del HC-SR04
const float DIST_MAX_VALIDA = 400.0;          // alcance nominal aprox.
const unsigned long TIMEOUT_ECO_US = 30000UL; // evita que pulseIn congele el programa
const unsigned long INTERVALO_LECTURA_MS = 60;  // pausa entre disparos del sensor
const unsigned long INTERVALO_REPORTE_MS = 500; // cada cuanto se imprime en el Monitor Serie
const byte N_LECTURAS  = 5;   // ventana del filtro de mediana
const byte MIN_VALIDAS = 3;   // lecturas validas minimas para confiar en el dato

// ================== 2. ESTADO DEL SISTEMA ==================

enum Nivel { SIN_DATO, BAJO, MEDIO, NORMAL };

float lecturas[N_LECTURAS];
byte  indiceLectura = 0;
byte  lecturasTomadas = 0;   // para no informar error durante el arranque

bool  configOk   = false;
bool  sensorOk   = false;
Nivel nivelActual = SIN_DATO;
float distanciaCm = 0.0;
float porcentaje  = 0.0;
float litros      = 0.0;

bool bombaEncendida = false;
bool bombaBloqueada = false;
unsigned long inicioBomba   = 0;
unsigned long inicioBloqueo = 0;

unsigned long ultimaLectura = 0;
unsigned long ultimoReporte = 0;

// ================== 3. RELE / BOMBA ==================

void escribirRele(bool encender) {
  // Traduce "encender/apagar" al nivel electrico segun el tipo de modulo
  digitalWrite(PIN_RELE, (encender != RELE_ACTIVO_EN_BAJO) ? HIGH : LOW);
}

void encenderBomba(unsigned long ahora) {
  escribirRele(true);
  bombaEncendida = true;
  inicioBomba = ahora;
  Serial.println(F(">> Bomba ENCENDIDA (nivel bajo)"));
}

void apagarBomba(const __FlashStringHelper *motivo) {
  escribirRele(false);
  if (bombaEncendida) {
    Serial.print(F(">> Bomba APAGADA: "));
    Serial.println(motivo);
  }
  bombaEncendida = false;
}

void controlarBomba(unsigned long ahora) {
  // Falla segura: sin dato confiable, la bomba nunca queda encendida
  if (!sensorOk) {
    apagarBomba(F("falla del sensor"));
    return;
  }

  // Bloqueo por tiempo maximo (posible estanque sin agua o fuga)
  if (bombaBloqueada) {
    // Se libera al cumplir la espera o si el nivel ya subio (estanque rellenado)
    if (porcentaje >= PCT_APAGAR || ahora - inicioBloqueo >= TIEMPO_BLOQUEO_MS) {
      bombaBloqueada = false;
      Serial.println(F(">> Fin del bloqueo: la bomba puede volver a operar"));
    }
    return;
  }

  if (bombaEncendida) {
    if (porcentaje >= PCT_APAGAR) {
      apagarBomba(F("nivel alto alcanzado"));
    } else if (ahora - inicioBomba >= TIEMPO_MAX_BOMBA_MS) {
      apagarBomba(F("tiempo maximo superado"));
      bombaBloqueada = true;
      inicioBloqueo = ahora;
      Serial.println(F("!! Bomba BLOQUEADA temporalmente (revisar agua / fuga)"));
    }
  } else if (porcentaje <= PCT_ENCENDER) {
    encenderBomba(ahora);
  }
}

// ================== 4. SENSOR ==================

float leerDistanciaCm() {
  digitalWrite(PIN_TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(PIN_TRIG, HIGH);   // pulso de disparo de 10 us
  delayMicroseconds(10);
  digitalWrite(PIN_TRIG, LOW);

  unsigned long duracion = pulseIn(PIN_ECHO, HIGH, TIMEOUT_ECO_US);
  if (duracion == 0) return -1.0;               // sin eco (timeout)

  float d = duracion * 0.0343 / 2.0;            // ida y vuelta del sonido
  if (d < DIST_MIN_VALIDA || d > DIST_MAX_VALIDA) return -1.0;  // lectura no confiable
  return d;
}

// Mediana de las lecturas validas: ignora valores absurdos aislados
bool calcularMediana(float &resultado) {
  float validas[N_LECTURAS];
  byte n = 0;
  for (byte i = 0; i < N_LECTURAS; i++) {
    if (lecturas[i] > 0) validas[n++] = lecturas[i];
  }
  if (n < MIN_VALIDAS) return false;

  // Ordenamiento por insercion (pocos datos)
  for (byte i = 1; i < n; i++) {
    float x = validas[i];
    int j = i - 1;
    while (j >= 0 && validas[j] > x) {
      validas[j + 1] = validas[j];
      j--;
    }
    validas[j + 1] = x;
  }
  resultado = (n % 2 == 1) ? validas[n / 2] : (validas[n / 2 - 1] + validas[n / 2]) / 2.0;
  return true;
}

// ================== 5. CALCULO Y CLASIFICACION ==================

float calcularPorcentaje(float d) {
  float pct = (D_VACIO_CM - d) / (D_VACIO_CM - D_LLENO_CM) * 100.0;
  return constrain(pct, 0.0, 100.0);
}

Nivel clasificarSinHisteresis(float pct) {
  if (pct <= UMBRAL_BAJO)  return BAJO;
  if (pct >  UMBRAL_NORMAL) return NORMAL;
  return MEDIO;
}

// Solo cambia de estado si el nivel cruza el limite con un margen (histeresis)
Nivel clasificar(float pct, Nivel actual) {
  switch (actual) {
    case BAJO:
      if (pct > UMBRAL_BAJO + HISTERESIS) {
        return (pct > UMBRAL_NORMAL + HISTERESIS) ? NORMAL : MEDIO;
      }
      return BAJO;
    case MEDIO:
      if (pct < UMBRAL_BAJO - HISTERESIS)   return BAJO;
      if (pct > UMBRAL_NORMAL + HISTERESIS) return NORMAL;
      return MEDIO;
    case NORMAL:
      if (pct < UMBRAL_NORMAL - HISTERESIS) {
        return (pct < UMBRAL_BAJO - HISTERESIS) ? BAJO : MEDIO;
      }
      return NORMAL;
    default:
      return clasificarSinHisteresis(pct);  // primer dato
  }
}

// ================== 6. INDICADORES LED ==================

void escribirLeds(bool rojo, bool amarillo, bool verde) {
  digitalWrite(LED_ROJO, rojo ? HIGH : LOW);
  digitalWrite(LED_AMARILLO, amarillo ? HIGH : LOW);
  digitalWrite(LED_VERDE, verde ? HIGH : LOW);
}

void actualizarIndicadores(unsigned long ahora) {
  bool fase = (ahora / 250) % 2;               // parpadeo sin delay()
  if (lecturasTomadas < N_LECTURAS) { escribirLeds(false, false, false); return; }  // arrancando
  if (!sensorOk) { escribirLeds(false, fase, false); return; }  // amarillo parpadea: falla del sensor

  // Los LEDs SIEMPRE muestran el nivel. Si la bomba esta bloqueada, el LED activo parpadea.
  bool encendido = bombaBloqueada ? fase : true;
  escribirLeds(nivelActual == BAJO   && encendido,
               nivelActual == MEDIO  && encendido,
               nivelActual == NORMAL && encendido);
}

// ================== 7. MONITOR SERIE ==================

const __FlashStringHelper *nombreNivel(Nivel n) {
  switch (n) {
    case BAJO:   return F("BAJO (rojo)");
    case MEDIO:  return F("INTERMEDIO (amarillo)");
    case NORMAL: return F("NORMAL (verde)");
    default:     return F("SIN DATO");
  }
}

void reportarSerial() {
  if (lecturasTomadas < N_LECTURAS) {
    Serial.println(F("Midiendo..."));
    return;
  }
  if (!sensorOk) {
    Serial.println(F("ERROR SENSOR: sin lecturas validas (revisar conexiones o distancia minima)"));
    return;
  }
  Serial.print(F("Dist: "));      Serial.print(distanciaCm, 1);
  Serial.print(F(" cm | Nivel: ")); Serial.print(porcentaje, 1);
  Serial.print(F(" % | Vol: "));    Serial.print(litros, 2);
  Serial.print(F(" L | Estado: ")); Serial.print(nombreNivel(nivelActual));
  Serial.print(F(" | Bomba: "));
  if (bombaBloqueada)      Serial.println(F("BLOQUEADA"));
  else if (bombaEncendida) Serial.println(F("ON"));
  else                     Serial.println(F("OFF"));
}

// ================== 8. VALIDACION DE LA CONFIGURACION ==================

bool validarConfiguracion() {
  bool ok = true;
  if (D_LLENO_CM < DIST_MIN_VALIDA) { Serial.println(F("CONFIG: D_LLENO_CM esta dentro de la zona ciega")); ok = false; }
  if (D_VACIO_CM <= D_LLENO_CM)     { Serial.println(F("CONFIG: D_VACIO_CM debe ser mayor que D_LLENO_CM")); ok = false; }
  if (D_VACIO_CM > DIST_MAX_VALIDA) { Serial.println(F("CONFIG: D_VACIO_CM supera el alcance del sensor")); ok = false; }
  if (CAPACIDAD_L <= 0)             { Serial.println(F("CONFIG: CAPACIDAD_L debe ser mayor que 0")); ok = false; }
  if (UMBRAL_BAJO >= UMBRAL_NORMAL) { Serial.println(F("CONFIG: UMBRAL_BAJO debe ser menor que UMBRAL_NORMAL")); ok = false; }
  if (PCT_ENCENDER >= PCT_APAGAR)   { Serial.println(F("CONFIG: PCT_ENCENDER debe ser menor que PCT_APAGAR")); ok = false; }
  return ok;
}

// ================== 9. SETUP ==================

void setup() {
  // Rele APAGADO antes de configurarlo como salida: evita un pulso al arrancar
  escribirRele(false);
  pinMode(PIN_RELE, OUTPUT);
  escribirRele(false);

  pinMode(PIN_TRIG, OUTPUT);
  pinMode(PIN_ECHO, INPUT);
  digitalWrite(PIN_TRIG, LOW);
  pinMode(LED_ROJO, OUTPUT);
  pinMode(LED_AMARILLO, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);

  Serial.begin(9600);
  Serial.println(F("=== Estanque rural - DEMO de hardware ==="));

  for (byte i = 0; i < N_LECTURAS; i++) lecturas[i] = -1.0;

  // Autoprueba de LEDs: rojo, amarillo, verde (permite revisar el cableado)
  escribirLeds(true, false, false);  delay(300);
  escribirLeds(false, true, false);  delay(300);
  escribirLeds(false, false, true);  delay(300);
  escribirLeds(false, false, false);

  configOk = validarConfiguracion();
  if (configOk) {
    Serial.print(F("0 % = "));   Serial.print(D_VACIO_CM, 1);
    Serial.print(F(" cm | 100 % = ")); Serial.print(D_LLENO_CM, 1);
    Serial.print(F(" cm | Bomba ON <= ")); Serial.print(PCT_ENCENDER, 0);
    Serial.print(F(" % | OFF >= ")); Serial.print(PCT_APAGAR, 0);
    Serial.println(F(" %"));
  } else {
    Serial.println(F("Configuracion invalida: control detenido (los 3 LEDs parpadean)"));
  }
}

// ================== 10. LOOP (sin bloqueos) ==================

void loop() {
  unsigned long ahora = millis();

  if (!configOk) {                          // nunca operar con parametros incoherentes
    escribirRele(false);
    bool fase = (ahora / 250) % 2;
    escribirLeds(fase, fase, fase);
    return;
  }

  if (ahora - ultimaLectura >= INTERVALO_LECTURA_MS) {
    ultimaLectura = ahora;

    lecturas[indiceLectura] = leerDistanciaCm();
    indiceLectura = (indiceLectura + 1) % N_LECTURAS;
    if (lecturasTomadas < N_LECTURAS) lecturasTomadas++;

    float d;
    if (calcularMediana(d)) {
      sensorOk    = true;
      distanciaCm = d;
      porcentaje  = calcularPorcentaje(d);
      litros      = porcentaje / 100.0 * CAPACIDAD_L;
      nivelActual = clasificar(porcentaje, nivelActual);
    } else {
      sensorOk    = false;
      nivelActual = SIN_DATO;
    }

    controlarBomba(ahora);
  }

  actualizarIndicadores(ahora);

  if (ahora - ultimoReporte >= INTERVALO_REPORTE_MS) {
    ultimoReporte = ahora;
    reportarSerial();
  }
}
