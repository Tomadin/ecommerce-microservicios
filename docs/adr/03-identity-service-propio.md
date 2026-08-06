# 03: identity-service propio en vez de Keycloak

- **Estado**: Aceptado
- **Fecha**: 2026-08-06
- **Autores**: Brian Tomadin

## Contexto

El sistema necesita autenticación (registro, login), emisión de JWT firmado con roles (ADMIN,
CUSTOMER) y refresh token con revocación. El `api-gateway` valida el token como OAuth2 Resource
Server y cada servicio aplica autorización a nivel método. Hay que decidir si esa capacidad se
construye como un servicio propio o se delega en un Identity Provider estándar como Keycloak.

Ambas son válidas. La consigna es clara en que lo que no sirve es dejarlo a medias con las dos: hay
que elegir una e implementarla bien.

## Opciones Evaluadas

- **Opción 1**: Keycloak (IdP externo estándar).
  - **Pros**: herramienta estándar de la industria; provee registro, login, JWT, refresh, revocación
    y OAuth2/OIDC out-of-the-box; menos código de seguridad propio y, por lo tanto, menos superficie de bugs de auth.
  - **Contras**: un componente pesado más (Keycloak + su base) que desplegar y configurar; curva de
    aprendizaje de realms, clients y mappers; funciona como caja negra, con lo que se aprende a
    configurarlo más que a entender el flujo JWT de punta a punta; puede quedar sobredimensionado
    para el alcance.
- **Opción 2**: identity-service propio (Spring Security + emisión de JWT con Nimbus).
  - **Pros**: demuestra comprensión del flujo JWT completo (emisión, firma, validación, refresh,
    revocación), que es lo más valioso de poder explicar en una entrevista; reusa experiencia ya
    consolidada en un proyecto previo (refresh token con revocación); control total; se integra
    naturalmente en el stack Spring; mucho más liviano que Keycloak.
  - **Contras**: se escribe código de seguridad propio, con la responsabilidad y la superficie de
    error que eso implica si se hace mal; no deja experiencia con la herramienta estándar de mercado.

## Decisión

Se elige la **Opción 2**: un `identity-service` propio que emite y firma el JWT, con Spring Security
y la librería Nimbus (la misma que usan el gateway y los servicios para validar).

El argumento decisivo es el objetivo de aprendizaje: entender y poder explicar el flujo JWT de punta
a punta pesa más, para este portfolio, que saber configurar Keycloak. Se suma que reusa trabajo
previo ya probado y que evita el peso operativo de un IdP completo. Keycloak queda descartado para
este proyecto, pero identificado como la alternativa estándar.

## Consecuencias

- **Positivas**:
  - Dominio completo y demostrable del ciclo de tokens: emisión, firma, validación, refresh, revocación.
  - Integración liviana y coherente con el resto del stack (mismo ecosistema Spring, JWKS/Nimbus).
  - Reutilización de experiencia previa, reduciendo el riesgo de implementación.
  - Sin el peso operativo de desplegar y configurar Keycloak.
- **Negativas / Riesgos**:
  - La seguridad la escribís vos: hay que hacerla bien (firma, expiración, almacenamiento seguro de
    refresh tokens, revocación efectiva). Un error de auth es caro.
  - No queda experiencia práctica con Keycloak, que varias ofertas piden por nombre. Mitigación:
    implementar sobre estándares (Spring Security + OAuth2 Resource Server, verificación por JWKS) y
    dejar anotado que se conoce Keycloak como alternativa y por qué no se usó acá.
