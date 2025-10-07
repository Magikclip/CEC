# CEC

## Secuencia no bloqueante para válvulas

Para controlar dos válvulas por bota (llenado y vaciado) sin detener el resto del programa, conviene evitar bucles `while` que se queden esperando a que transcurra un tiempo. En su lugar se puede usar una **máquina de estados** impulsada por un temporizador libre de bloqueos, de modo que cada pasada por el `loop` (o ciclo principal) revise si ya se cumplió el tiempo asignado a la etapa actual.

### Idea general
1. Registrar, para cada bota, el instante (por ejemplo con `millis()`) en el que comenzó la etapa actual.
2. Definir una lista de etapas con su duración y las válvulas que deben estar abiertas o cerradas.
3. En cada iteración del ciclo principal comprobar si se venció la duración de la etapa; si es así, avanzar a la siguiente.
4. Como no hay bucles de espera, el resto del código puede ejecutarse normalmente.

### Ejemplo en pseudo-C/Arduino
```cpp
struct Etapa {
  uint32_t duracionMs;
  bool valvulaLlenado;
  bool valvulaVaciado;
};

const Etapa secuencia[] = {
  {3000, true,  false},  // Llenar 3 s
  {1000, false, true },  // Vaciar 1 s
  {2000, false, false}   // Mantener 2 s
};
const size_t numEtapas = sizeof(secuencia) / sizeof(secuencia[0]);

size_t etapaActual = 0;
uint32_t inicioEtapa = 0;

void setup() {
  pinMode(PIN_VALVULA_LLENA, OUTPUT);
  pinMode(PIN_VALVULA_VACIA, OUTPUT);
  inicioEtapa = millis();
}

void loop() {
  // Aplicar el estado de la etapa actual
  digitalWrite(PIN_VALVULA_LLENA, secuencia[etapaActual].valvulaLlenado);
  digitalWrite(PIN_VALVULA_VACIA, secuencia[etapaActual].valvulaVaciado);

  // Verificar si debemos pasar a la siguiente etapa
  uint32_t ahora = millis();
  if (ahora - inicioEtapa >= secuencia[etapaActual].duracionMs) {
    etapaActual = (etapaActual + 1) % numEtapas;  // Ciclar la secuencia
    inicioEtapa = ahora;
  }

  // Aquí se pueden ejecutar otras tareas sin bloqueo
  actualizarSensores();
  comunicarEstado();
}
```

Este patrón permite escalar a múltiples botas replicando la estructura `Etapa` y controlando cada par de válvulas con su propia máquina de estados independiente.
