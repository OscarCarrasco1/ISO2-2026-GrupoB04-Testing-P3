# ISO2-2026-GrupoB04-Testing-P2

Ingeniería del Software II – Segundo trabajo (Parte Testing) – **Problema 2: concesión de becas**

| Paso | Qué se hace | Pareja |
|------|-------------|--------|
| Paso 1 | Implementación del programa (rama `develop`) | Pareja 2 |
| Paso 2 | Diseño de casos de prueba (ejercicio teórico) | Pareja 3 |
| Paso 3 | Implementación de las pruebas en JUnit (rama `hotfix`) | Pareja 1 |

## Enunciado

Algoritmo que, a partir de los datos de un estudiante universitario (nota media, renta familiar, distancia del domicilio al campus, grado de discapacidad y número de hermanos), determina la tipología de beca más adecuada:

- **Beca de Excelencia:** nota media >= 8,5 y renta familiar < 30.000 €.
- **Beca de Residencia:** nota media entre 6,5 y 8,4 (inclusive), renta familiar < 20.000 € y distancia al campus > 50 km.
- **Beca de Familia Numerosa:** nota media >= 5,0, renta familiar < 15.000 € y 3 o más hermanos.
- **Beca de Inclusión:** grado de discapacidad acreditado >= 33% y nota media >= 5,0.
- **Sin Beca:** si no cumple ninguno de los criterios anteriores.

Se supone que no puede haber ambigüedades en el tipo de beca ofrecida; si existen solapamientos o datos inválidos, se documentan las suposiciones y excepciones correspondientes.

## Estructura

```text
es.uclm.esi.iso2.becas
├── Main                            → monta las piezas y arranca
├── dominio
│   ├── TipoBeca                    → enum (EXCELENCIA, RESIDENCIA, FAMILIA_NUMEROSA, INCLUSION, SIN_BECA)
│   ├── Estudiante                  → datos del alumno; valida rangos en el constructor
│   ├── EvaluadorBecas              → interfaz: evaluar(Estudiante)
│   └── EvaluadorBecasEstandar      → reglas de negocio y resolución de prioridades/conflictos
├── excepciones
│   ├── EstudianteInvalidoException → base para datos fuera de rango o nulos
│   ├── NotaInvalidaException       → nota media fuera de [0,0 ; 10,0]
│   ├── DatoNegativoException       → renta, distancia, hermanos o discapacidad negativos (o discapacidad > 100%)
│   └── BecaAmbiguaException        → si se opta por lanzar excepción ante solapamiento (opcional según diseño)
└── ui
    ├── LectorConsola               → lee y valida entradas por teclado
    ├── PantallaConsola             → muestra resultados y errores
    └── ControladorBecas            → une UI y dominio
```

El paquete `dominio` no depende de `ui`. `LectorConsola` y `PantallaConsola` reciben el `InputStream`/`PrintStream` por constructor para facilitar su prueba aislada.

## Decisiones de diseño

1. **Validación en el constructor de `Estudiante`.** Si se instancia un `Estudiante`, sus datos son consistentes (`notaMedia` en [0, 10], `gradoDiscapacidad` en [0, 100], y `rentaFamiliar`, `distanciaKm` y `numHermanos` >= 0). Atributos privados y `final`.
2. **Resolución de ambigüedades por orden de prioridad (suposición documentada):** Como un estudiante puede cumplir simultáneamente los requisitos de varias becas (por ejemplo, nota 9,0, renta 10.000 € y 3 hermanos cumple Excelencia y Familia Numerosa), se asigna la beca más beneficiosa siguiendo esta jerarquía estricta:
   1. **Beca de Inclusión** (discapacidad >= 33% y nota >= 5,0).
   2. **Beca de Excelencia** (nota >= 8,5 y renta < 30.000 €).
   3. **Beca de Residencia** (nota en [6,5 ; 8,4], renta < 20.000 € y distancia > 50 km).
   4. **Beca de Familia Numerosa** (nota >= 5,0, renta < 15.000 € y hermanos >= 3).
   5. **Sin Beca** (resto de casos).
3. **Intervalo de nota en Residencia:** Se interpreta "entre 6,5 y 8,4" como intervalo cerrado [6,5 ; 8,4] (a partir de 8,5 pasa a optar a Excelencia).
4. **Comparaciones estrictas vs. no estrictas:** Se respetan literalmente los operadores del enunciado: renta estrictamente menor (`< 30000`, `< 20000`, `< 15000`), distancia estrictamente mayor (`> 50`), y nota/hermanos/discapacidad mayor o igual (`>=`).
5. **Dinero con `BigDecimal`** para la renta familiar, evitando imprecisiones de coma flotante en valores límite (ej. `29999,99 €` vs `30000,00 €`).

### Ejemplos

| Nota | Renta (€) | Distancia (km) | Discapacidad (%) | Hermanos | Resultado |
|------|-----------|----------------|------------------|----------|-----------|
| 8,5 | 29.999,99 | 10 | 0 | 0 | `EXCELENCIA` |
| 8,5 | 30.000,00 | 10 | 0 | 0 | `SIN_BECA` |
| 7,0 | 19.999,99 | 51 | 0 | 1 | `RESIDENCIA` |
| 7,0 | 19.999,99 | 50 | 0 | 1 | `SIN_BECA` |
| 5,0 | 14.999,99 | 5 | 0 | 3 | `FAMILIA_NUMEROSA` |
| 5,0 | 45.000,00 | 2 | 33 | 0 | `INCLUSION` |
| 9,2 | 12.000,00 | 80 | 40 | 4 | `INCLUSION` *(por prioridad)* |

## Cómo compilar y ejecutar

```bash
mvn clean package
java -jar target/ISO2-2026-GrupoB04-Testing-P2-1.0.0.jar
```

Ejemplo de uso:

```text
Nota media (0-10): 8,7
Renta familiar (EUR): 24500
Distancia al campus (km): 15
Grado de discapacidad (%): 0
Número de hermanos: 1
Beca asignada: Beca de Excelencia (exención total de tasas + 1.000 EUR)
```

## Pruebas (Paso 3)

Las pruebas van en `src/test/java` (rama `hotfix`). El `pom.xml` ya incluye JUnit 5, Surefire y Jacoco:

```bash
mvn clean test                 # ejecuta las pruebas; informe Jacoco en target/site/jacoco/index.html
mvn surefire-report:report     # informe Surefire en target/site/surefire-report.html
