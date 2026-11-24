# ISO2-2026-GrupoB04-Testing-P3

Ingeniería del Software II – Segundo trabajo (Parte Testing) – **Problema 3: domótica agrícola (riego)**

| Paso | Qué se hace | Pareja |
|------|-------------|--------|
| Paso 1 | Implementación del programa (rama `develop`) | Pareja 3 |
| Paso 2 | Diseño de casos de prueba (ejercicio teórico) | Pareja 1 |
| Paso 3 | Implementación de las pruebas en JUnit (rama `hotfix`) | Pareja 2 |

## Enunciado

Aplicación que genera recomendaciones sobre el ciclo de riego para invernaderos en función de la humedad del suelo, temperatura ambiente, previsión de lluvia y franja horaria:

- **Riego Intenso:** humedad < 20%, temperatura > 25 ºC y sin previsión de lluvia en 24 h.
- **Riego Ligero:** humedad entre 20% y 40% (inclusive), temperatura > 25 ºC y sin previsión de lluvia.
- **No Regar:** humedad < 40% y previsión de lluvia inminente.
- **Drenaje de Emergencia:** humedad > 60% y temperatura < 5 ºC.
- **Franja de máxima insolación (12:00 a 16:00):** posponer cualquier riego (**Posponer Riego**), salvo que la humedad sea < 10%, en cuyo caso se activa **Riego Crítico**.

## Estructura

```text
es.uclm.esi.iso2.riego
├── Main                           → monta las piezas y arranca
├── dominio
│   ├── RecomendacionRiego         → enum (RIEGO_INTENSO, RIEGO_LIGERO, NO_REGAR, DRENAJE_EMERGENCIA, POSPONER_RIEGO, RIEGO_CRITICO, SIN_ACCION)
│   ├── LecturaInvernadero         → humedad, temperatura, lluvia, hora; valida en el constructor
│   ├── ControladorRiego           → interfaz: recomendar(LecturaInvernadero)
│   └── ControladorRiegoEstandar   → reglas de negocio y jerarquía de prioridades
├── excepciones
│   ├── LecturaInvalidaException   → base para lecturas fuera de rango o nulas
│   ├── HumedadInvalidaException   → humedad fuera de [0, 100]
│   ├── TemperaturaInvalidaException → temperatura fuera de rango físico [-50, 70] ºC
│   └── HoraNulaException          → hora nula
└── ui
    ├── LectorConsola              → lee parámetros ambientales y hora (HH:mm)
    ├── PantallaConsola            → muestra la recomendación resultante
    └── ControladorUI              → une UI y dominio
```

El paquete `dominio` no depende de `ui`. `LectorConsola` y `PantallaConsola` reciben el `InputStream`/`PrintStream` por constructor para poder probarlos sin teclado ni consola reales.

## Decisiones de diseño

1. **Validación en el constructor de `LecturaInvernadero`.** Atributos privados y `final`. Se rechaza humedad fuera de [0, 100], temperatura fuera de [-50, 70] ºC y hora nula.
2. **Jerarquía de evaluación (resolución de conflictos entre reglas):** Dado que algunas condiciones pueden solaparse (ej. humedad 15%, temperatura 30 ºC a las `13:00`), las reglas se evalúan en orden estricto de seguridad y prioridad:
   1. **Drenaje de Emergencia:** si `humedad > 60` y `temperatura < 5` (riesgo de congelación de raíces; prevalece siempre porque no es un riego sino un drenaje).
   2. **Franja de máxima insolación (`12:00 <= hora <= 16:00`):**
      - Si `humedad < 10` → **Riego Crítico** (prevalece incluso si hay lluvia prevista para evitar estrés hídrico extremo).
      - En cualquier otro caso dentro de esa franja → **Posponer Riego**.
   3. **Previsión de lluvia inminente:** si `hayLluvia == true` y `humedad < 40` → **No Regar**.
   4. **Riego por temperatura (`temperatura > 25` y `!hayLluvia`):**
      - Si `humedad < 20` → **Riego Intenso**.
      - Si `20 <= humedad <= 40` → **Riego Ligero**.
   5. **Caso por defecto (`Sin Acción`):** cuando no se cumple ninguna condición anterior (ej. humedad entre 41% y 60%, o temperatura <= 25 ºC sin lluvia).
3. **Franja horaria cerrada:** `12:00` y `16:00` se consideran incluidos dentro de la franja de máxima insolación (`LocalTime`).

### Ejemplos

| Humedad (%) | Temp (ºC) | Lluvia | Hora | Recomendación |
|-------------|-----------|--------|------|---------------|
| 61 | 4 | false | 14:00 | `DRENAJE_EMERGENCIA` |
| 9 | 30 | true | 12:00 | `RIEGO_CRITICO` |
| 10 | 30 | false | 16:00 | `POSPONER_RIEGO` |
| 19 | 26 | true | 10:00 | `NO_REGAR` |
| 19 | 26 | false | 11:59 | `RIEGO_INTENSO` |
| 20 | 26 | false | 16:01 | `RIEGO_LIGERO` |
| 40 | 26 | false | 09:00 | `RIEGO_LIGERO` |
| 41 | 26 | false | 09:00 | `SIN_ACCION` |

## Cómo compilar y ejecutar

```bash
mvn clean package
java -jar target/ISO2-2026-GrupoB04-Testing-P3-1.0.0.jar
```

Ejemplo de uso:

```text
Humedad del suelo (%): 18
Temperatura ambiente (ºC): 28.5
¿Previsión de lluvia? (true/false): false
Hora actual (HH:mm): 10:30
Recomendación: RIEGO_INTENSO
```

## Pruebas (Paso 3)

Las pruebas van en `src/test/java` (rama `hotfix`). El `pom.xml` ya incluye JUnit 5, Surefire y Jacoco:

```bash
mvn clean test                 # ejecuta las pruebas; informe Jacoco en target/site/jacoco/index.html
mvn surefire-report:report     # informe Surefire en target/site/surefire-report.html
```
