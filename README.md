# Recordatorios Citas Service

![CI](https://github.com/sanYesid/calvo-post1-u9/actions/workflows/ci.yml/badge.svg)

## Descripción

Repositorio del post-contenido de la Unidad 9 de Patrones de Diseño de Software — Sexto Semestre. Es un único proyecto Spring Boot (Java 21, Spring Boot 3.3.5) que revisa qué citas médicas ocurren en las próximas 24 horas y envía un recordatorio al paciente por su canal preferido (correo o SMS). También permite enviar un recordatorio urgente para una cita puntual.

El servicio central, `CitaReminderServiceImpl`, está construido desde el inicio con dos patrones, ambos inyectados por constructor (Principio de Inversión de Dependencias):

- **Strategy** (`ReminderChannel`): cada canal de envío (`EmailReminderChannel`, `SmsReminderChannel`) es una implementación intercambiable de la misma interfaz.
- **Factory Method** (`ReminderChannelFactory`): decide en tiempo de ejecución qué canal entregar según `Cita.canalPreferido`.

```
src/main/java/com/clinica/recordatorios/
├── domain/        Cita, CanalNotificacion
├── repository/    CitaRepository
├── channel/       ReminderChannel, Email/Sms, ReminderChannelFactory(+Impl), gateway/
├── service/       CitaReminderService(+Impl), CitaNoEncontradaException
└── config/        ClockConfig, CanalUrgenteConfig   ← composition root
```

## Parte 1 — Pruebas Unitarias con JUnit 5 y Mockito

`CitaReminderServiceImplTest` aísla `CitaReminderServiceImpl` de sus colaboradores usando `@ExtendWith(MockitoExtension.class)`. Cubre la selección de canal por la preferencia de la cita, la ventana exacta de 24 horas (gracias a un reloj fijo) y el caso sin citas pendientes. La justificación de cada Test Double está en "Decisiones de diseño".

Evidencia: 3 pruebas del servicio en verde (4 en total con la prueba de carga de contexto que genera Spring Initializr).

![Parte 1 — pruebas en verde](docs/img/01-parte1-pruebas-verde.png)

## Parte 2 — Refactor DIP, Integración y CI/CD

`enviarRecordatorioUrgente` nació con un `new` hardcodeado que la hacía imposible de testear en aislamiento. El refactor movió esa construcción a `config/CanalUrgenteConfig` e inyectó la abstracción `ReminderChannel` por constructor. Con eso se pudieron agregar tres pruebas nuevas, entre ellas `enviarRecordatorioUrgente_gatewayFalla_noMarcaComoEnviado`, que antes del refactor era imposible de escribir.

Sobre ese mismo proyecto se agregaron:

- `CitaRepositoryTest` (`@DataJpaTest`): prueba la consulta derivada contra H2 en memoria.
- `CitaReminderServiceIntegrationTest` (`@SpringBootTest`): prueba el flujo completo con H2 real, un `Clock` fijo y `@MockBean` sobre `NotificadorSmsGateway`.
- `.github/workflows/ci.yml`: ejecuta `mvn -B verify` en cada push y pull request a `main`.

**Paso 8 — refactor aplicado, 6 pruebas unitarias en verde:**

![Paso 8 — refactor DIP](docs/img/02-paso8-refactor-dip.png)

**Paso 9 — `@DataJpaTest` contra H2 (se ve el SQL generado por Hibernate):**

![Paso 9 — DataJpaTest](docs/img/03-paso9-datajpatest.png)

**Paso 10 — `@SpringBootTest` con contexto completo, 9 pruebas en verde:**

![Paso 10 — SpringBootTest](docs/img/04-paso10-springboottest.png)

**Cobertura (JaCoCo):**

![Reporte de cobertura JaCoCo](docs/img/05-jacoco-cobertura.png)

**Pipeline de GitHub Actions (estado Success):**

![GitHub Actions — Success](docs/img/06-github-actions-success.png)

## Decisiones de diseño

### 1. Mock, Stub o Fake: una decisión por colaborador

El criterio fue la tabla de Test Doubles de la guía: si la prueba solo necesita controlar un valor de retorno, basta un Stub; si además necesita verificar la interacción, es un Mock; y si existe una implementación real, completa y determinista, se usa como Fake.

| Colaborador | Double | Justificación |
|---|---|---|
| `CitaRepository` | **Mock** | No basta con devolver una lista desde `findByFechaHoraCitaBetween...`: la prueba también verifica que `save()` se llama exactamente una vez con la cita ya marcada como enviada, y que nunca se llama cuando no hay citas. Controlar la respuesta y verificar la interacción a la vez es un Mock, no un Stub. |
| `ReminderChannelFactory` | **Mock** | El comportamiento bajo prueba es la selección de estrategia: que el servicio pida el canal `EMAIL` cuando la cita lo prefiere. Eso solo se comprueba verificando la interacción. En la prueba sin citas se usa `verifyNoInteractions` para demostrar que la factory ni siquiera se consulta. |
| `ReminderChannel` (canal devuelto por la factory) | **Mock** | Lo que importa es que el canal elegido reciba la cita y el mensaje (`verify(...).enviar(eq(cita), anyString())`). No hay valor de retorno que comprobar: el método es `void`. |
| `ReminderChannel` `canalUrgente` | **Mock** | Permite verificar que el mensaje contiene "URGENTE" y, sobre todo, simular un fallo con `doThrow` para comprobar que la cita no se marca como enviada. |
| `Clock` | **Fake** (no Mock) | `Clock.fixed(...)` es una implementación real y completa de `java.time.Clock` provista por el JDK: `LocalDateTime.now(clock)` funciona exactamente igual que en producción, solo que con el instante congelado. Mockearlo obligaría a stubbear `instant()` y `getZone()` por separado y arriesgaría un comportamiento distinto del real. Con el fake, la aserción de la ventana (`2026-08-10T09:00` a `2026-08-11T09:00`) es exacta y reproducible a cualquier hora y en cualquier máquina; con `Clock.systemDefaultZone()` cambiaría en cada ejecución. |

No quedó ningún Stub puro porque en todas las pruebas se verifica además la interacción con el colaborador.

### 2. Qué patrón facilitó qué prueba, y por qué sin él sería más difícil

| Patrón | Prueba que facilita | Sin el patrón |
|---|---|---|
| **Strategy** (`ReminderChannel`) | `enviarRecordatoriosPendientes_citaEnVentana_usaElCanalPreferido` y las tres pruebas de `enviarRecordatorioUrgente` | La lógica de envío (SMTP o llamada HTTP al proveedor SMS) estaría escrita en línea dentro del servicio. Probar la selección de canal exigiría mockear librerías de bajo nivel en vez de una sola interfaz propia con un método `enviar()`. Con Strategy, un único `@Mock ReminderChannel` sirve para verificar cualquier canal y para simular fallos. |
| **Factory Method** (`ReminderChannelFactory`) | `..._citaEnVentana_usaElCanalPreferido` (stub de `obtenerCanal(EMAIL)`) y `..._sinCitasPendientes_noConsultaLaFactory` | El servicio tendría un `if/else` o `switch` embebido para decidir el canal, y esa decisión solo sería observable de forma indirecta (revisando qué se imprimió o qué se envió realmente). Con la factory inyectada, la selección es una interacción explícita y verificable, y se puede afirmar que no ocurre cuando no hay citas. |
| **Inyección por constructor / DIP** | Todas las pruebas unitarias, y en especial `enviarRecordatorioUrgente_gatewayFalla_noMarcaComoEnviado` | Si el servicio construyera sus colaboradores con `new`, no habría ningún punto donde sustituirlos por un Test Double (ver siguiente sección). |

### 3. Diagnóstico antes del refactor: por qué `enviarRecordatorioUrgente` no se podía testear

La primera versión del método construía su dependencia dentro del cuerpo:

```java
@Override
public void enviarRecordatorioUrgente(Long citaId) {
    Cita cita = citaRepository.findById(citaId)
            .orElseThrow(() -> new CitaNoEncontradaException(citaId));

    ReminderChannel canalUrgente = new SmsReminderChannel(new TwilioNotificadorSmsGateway()); // ← línea responsable
    canalUrgente.enviar(cita, "URGENTE: su cita es en menos de 2 horas y no ha sido confirmada");

    cita.setRecordatorioEnviado(true);
    citaRepository.save(cita);
}
```

La línea responsable es la que usa `new SmsReminderChannel(new TwilioNotificadorSmsGateway())`. `SmsReminderChannel` y `TwilioNotificadorSmsGateway` se crean dentro de la propia lógica de negocio, así que el método depende de clases concretas y no de abstracciones. Eso viola el Principio de Inversión de Dependencias y deja sin ningún punto de sustitución a Mockito: no hay forma de entregar un Test Double que simule, por ejemplo, que el gateway de SMS falla, y por lo tanto no se puede probar que en ese caso la cita no debe marcarse como enviada.

Esto contrasta con `enviarRecordatoriosPendientes()`, que desde la Parte 1 sí es testeable: recibe el repositorio, la factory y el reloj por constructor, de modo que la prueba decide qué implementación (mock o fake) recibe el servicio.

Técnicamente se podría interceptar ese `new` con `Mockito.mockConstruction()`, pero eso es una señal de que el problema es el diseño y no la herramienta: se resuelve con DIP, no con una API de pruebas más avanzada.

### 4. El refactor DIP: antes y después

El `new` no se eliminó: alguien tiene que construir el objeto concreto. Se movió al único lugar donde sí es correcto escribirlo, la configuración de Spring (composition root), y el servicio recibe la abstracción por constructor.

**Antes** — `new` dentro de la lógica de negocio, sin punto de sustitución:

```java
ReminderChannel canalUrgente = new SmsReminderChannel(new TwilioNotificadorSmsGateway());
canalUrgente.enviar(cita, "URGENTE: ...");
```

**Después** — el bean se declara en `config/CanalUrgenteConfig` y se inyecta por constructor:

```java
@Configuration
public class CanalUrgenteConfig {
    @Bean
    public ReminderChannel canalUrgente(NotificadorSmsGateway gateway) {
        return new SmsReminderChannel(gateway);
    }
}
```

```java
public CitaReminderServiceImpl(CitaRepository citaRepository,
                               ReminderChannelFactory reminderChannelFactory,
                               @Qualifier("canalUrgente") ReminderChannel canalUrgente,
                               Clock clock) { ... }

// en el método
canalUrgente.enviar(cita, "URGENTE: ...");
```

El comportamiento externo no cambió, pero ahora la prueba sustituye `canalUrgente` por un mock sin que `TwilioNotificadorSmsGateway` participe. La prueba `enviarRecordatorioUrgente_gatewayFalla_noMarcaComoEnviado` configura el mock con `doThrow(...)` y verifica `never().save(...)`: es la evidencia de que la testabilidad es consecuencia directa de aplicar DIP.

### 5. Por qué H2 real pero gateway mockeado

En `CitaReminderServiceIntegrationTest` conviven una base de datos real (H2 en memoria) y un `@MockBean` sobre `NotificadorSmsGateway`. No es inconsistente. H2 es un Fake aceptable porque su contrato SQL/JPA es equivalente al de la base real y no requiere infraestructura adicional. Un proveedor de SMS de terceros no tiene un "H2 equivalente" gratuito y determinista; reemplazarlo con un Mock evita depender de credenciales, de red y de la disponibilidad del proveedor en cada ejecución del pipeline.

En esa misma prueba el `Clock` se reemplaza con un bean `@Primary` de `@TestConfiguration` (reloj fijo), para que la ventana de 24 horas sea determinista. Como ese bean tiene el mismo nombre (`clock`) que el de `ClockConfig`, la prueba habilita `spring.main.allow-bean-definition-overriding=true` solo en el contexto de test.

### 6. Por qué el pipeline no necesita un contenedor de base de datos

El workflow no define ningún bloque `services:` (por ejemplo PostgreSQL). Es posible precisamente porque las pruebas de integración usan H2 en memoria. Si se hubiera optado por PostgreSQL con Testcontainers, el runner necesitaría Docker disponible y los tiempos de arranque serían mayores. Elegir H2 simplifica directamente la configuración de CI.

## Ejecutar las pruebas

```bash
mvn test      # Pruebas unitarias y de integración de la fase test
mvn verify    # Ciclo completo (igual que el pipeline) + reporte JaCoCo
```

El reporte de cobertura queda en `target/site/jacoco/index.html`.

## Herramientas utilizadas

- Java 21, Spring Boot 3.3.5, Spring Data JPA, H2
- JUnit 5, Mockito, JaCoCo
- Apache Maven, GitHub Actions, Git, GitHub

## Conclusiones

Este laboratorio mostró que la facilidad para probar un servicio no depende de la herramienta de pruebas, sino de cómo está diseñado: con Strategy, Factory Method e inyección por constructor, un solo `@Mock` basta para verificar la selección y el uso de cualquier canal. El caso de `enviarRecordatorioUrgente` fue el más claro, porque el escenario "el gateway falla y la cita no se marca como enviada" era imposible de probar mientras existía el `new` hardcodeado y pasó a ser trivial al moverlo al composition root. También aprendí a elegir el Test Double según lo que la prueba necesita: Mock cuando hay que verificar interacciones, Fake cuando existe una implementación real y determinista como `Clock.fixed` o H2. Por último, usar H2 en las pruebas de integración permitió que el pipeline de GitHub Actions corriera sin infraestructura adicional.
