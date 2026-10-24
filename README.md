# Recordatorios Citas Service

![CI](https://github.com/[usuario]/apellido-post1-u9/actions/workflows/ci.yml/badge.svg)

## Descripción
Repositorio del post-contenido de la Unidad 9 de Patrones de Diseño
de Software — Sexto Semestre. Un único proyecto Spring Boot que
envía recordatorios de citas por el canal preferido del paciente,
construido desde el inicio con Strategy (ReminderChannel) y Factory
Method (ReminderChannelFactory) inyectados por constructor.


## Decisiones de diseño

Mock vs. Stub vs. Fake (Parte 1). CitaRepository y ReminderChannelFactory/ReminderChannel se mockearon porque las pruebas verifican interacciones (save(), obtenerCanal()), no solo valores de retorno. Clock se resolvió con Clock.fixed(...), un fake real del JDK, porque mockearlo habría sido más complejo y menos fiel al comportamiento real que usar la implementación fija que Java ya provee para pruebas.