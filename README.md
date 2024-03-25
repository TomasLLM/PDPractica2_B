# INFORME DE LA PRACTICA 2:  INTERRUPCIONS
#### Autor: Tomàs Lloret

En aquesta segona pràctica volem aprendre com funcionen les interrupcions. Per a fer això, utilitzem dos leds de forma periòdica amb una entrada que farà variar les oscil·lacions de un dels leds.

## Material:
- Esp32doit-devkit-v1.
- Polsador o en el meu cas dos cables.

## Part A: Interrupció per GPIO 

Comencem amb la estructura del botó, que conté una variable booleana que manté el valor relacionat amb l'estat del polsador, i una altra variable que recull el nombre de vegades que s'ha polsat aquest botó:
```c
struct Button {
  const uint8_t PIN;
  uint32_t numberKeyPresses;
  bool pressed;
};
```

En el codi fem servir la funció attachInterrupt() per a establir una interrupció en un pin d'aquesta manera:
```c
attachInterrupt(button1.PIN, isr, FALLING);
```
La separació d'interrupcions quan ja no cal que el pin sigui monitoritzat pel procés es fa tal que:
```c
detachInterrupt(button1.PIN);
```
Aquesta és la funció que es crida quan ocorre l'interrupció, la cual pasa l'estat del botó a pressed i agrega u al nombre de vegades que s'ha premut l'interruptor:
```c
void IRAM_ATTR isr() {
  button1.numberKeyPresses += 1;
  button1.pressed = true;
}
```

El setup inicialitza els pins com a entrada i amb la funció attachInterrupt fa la inicialització:
```c
void setup() {
  Serial.begin(115200);
  pinMode(button1.PIN, INPUT_PULLUP);
  attachInterrupt(button1.PIN, isr, FALLING);
}
```

En el loop trobem el codi que fa que surti per terminal quantes vegades s'ha polsat el botó i el que torna l'estat del botó a no polsat.
```c
void loop() {
  if (button1.pressed) {
    Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses);
    button1.pressed = false;
  }

  if (millis() - lastMillis > 60000) {
    lastMillis = millis();
    detachInterrupt(button1.PIN);
    Serial.println("Interrupt Detached!");
  }
}
```

### Part A: Codi complet
```c
//PD
//PRÀCTICA 2.1
#include <Arduino.h>

struct Button {
  const uint8_t PIN;
  uint32_t numberKeyPresses;
  bool pressed;
};

Button button1 = {18, 0, false};
static uint32_t lastMillis = 0;

void IRAM_ATTR isr() {
  button1.numberKeyPresses += 1;
  button1.pressed = true;
}

void setup() {
  Serial.begin(115200);
  pinMode(button1.PIN, INPUT_PULLUP);
  attachInterrupt(button1.PIN, isr, FALLING);
}

void loop() {
  if (button1.pressed) {
    Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses);
    button1.pressed = false;
  }
  
  if (millis() - lastMillis > 60000) {
    lastMillis = millis();
    detachInterrupt(button1.PIN);
    Serial.println("Interrupt Detached!");
  }
}
```

## Part B: Interrupció per timer
Ara farem servir un comptador intern que en un temps determinat realitza una interrupció.

Amb l'ús de la funció volatile evitarem que el proces elimini el contingut de la variable que guarda el nombre d'interrupcions fetes pel procés. totalInterruptCounter comptarà el nombre d'interrupcions totals sense necesitat d'usar volatile. També definim un punter que servirà per a configurar el timer i una variable que sincronitza el void loop i la ISR.
```c
volatile int interruptCounter;
int totalInterruptCounter;

hw_timer_t * timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;
```

La funció seguent serveix per a comptar el nombre d'interrupcions que es produiran cada cop que el void loop tingui una interrupció, ocorrent tot això dins d'una secció definida tal que:
```c
void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}
```

El setup utilitzat te en compte que el nostre microcontrolador treballa en una frecuencia base de 80MHz, per tant definim el timerAlarmWrite a 1000000 ja que es troba en microsegons. Com està definit a true, el timer indica que el comptador contarà de forma progressiva.

Amb timerAttachInterrupt volem utilitzar la interrupció utilitzant com a paràmetre la variable de temps global &onTimer.Com està en true això vol dir que la interrupció estarà en tipus edge.

En la funció timerAlarmWrite especifiquem el valor del comptador en el que es generarà la interrupció desitjada. Com la tenim en true, el comptador es recarregarà, generant la interrupció de forma periòdica. Per l'altra banda, el timerAlarmEnable s'utilitza per a habilitar el temporitzador.
```c
void setup() {
    Serial.begin(115200);
    timer = timerBegin(0, 80, true);
    timerAttachInterrupt(timer, &onTimer, true);
    timerAlarmWrite(timer, 1000000, true);
    timerAlarmEnable(timer);
  }
  ```

Dins del loop tindrem el manejament de l'interrupció en si, que simplement incrementarà el comptador amb el nombre total d'interrupcions i imprimirà aquest en el terminal.
```c
void loop() {
  if (interruptCounter > 0) {
      portENTER_CRITICAL(&timerMux);
      interruptCounter--;
      portEXIT_CRITICAL(&timerMux);
      totalInterruptCounter++;
      Serial.print("An interrupt as occurred. Total number: ");
      Serial.println(totalInterruptCounter);
    }
  }
```

### Part B: Codi complet
```c
//PD
//PRÀCTICA 2.2
#include <Arduino.h>

volatile int interruptCounter;
int totalInterruptCounter;

hw_timer_t * timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;

void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}

  void setup() {
    Serial.begin(115200);
    timer = timerBegin(0, 80, true);
    timerAttachInterrupt(timer, &onTimer, true);
    timerAlarmWrite(timer, 1000000, true);
    timerAlarmEnable(timer);
  }

  void loop() {
  if (interruptCounter > 0) {
      portENTER_CRITICAL(&timerMux);
      interruptCounter--;
      portEXIT_CRITICAL(&timerMux);
      totalInterruptCounter++;
      Serial.print("An interrupt as occurred. Total number: ");
      Serial.println(totalInterruptCounter);
    }
  }
```

## Observacions i conclusions
Amb aquestes dos pràctiques hem vist i assolit correctament dos maneres diferents de generar interrupcions dins del nostre microcontrolador, considerem doncs que s'ha arribat als objectius de la pràctica i la donem per acabada.
