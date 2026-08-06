# 02: Service discovery con Eureka

- **Estado**: Aceptado
- **Fecha**: 2026-08-06
- **Autores**: Brian Tomadin

## Contexto

El sistema tiene varios servicios que se llaman entre sí (`cart-service` -> `products-service`,
`api-gateway` -> todos). Hay que resolver dónde vive cada servicio sin clavar hosts y puertos en la
configuración, y poder correr varias instancias de un servicio con balanceo de carga entre ellas.

La consigna pide service discovery y aclara explícitamente que, en entornos modernos sobre Kubernetes,
el discovery suele reemplazarse por el mecanismo nativo del clúster (DNS). Hay que elegir cómo
implementarlo en este proyecto, cuyo despliegue es docker-compose / PaaS, no Kubernetes.

## Opciones Evaluadas

- **Opción 1**: URLs hardcodeadas en configuración, sin discovery.
  - **Pros**: cero infraestructura extra; simple para arrancar.
  - **Contras**: hosts y puertos clavados; no soporta múltiples instancias ni balanceo; frágil ante
    cualquier cambio de topología; no demuestra el patrón, que es parte del objetivo.
- **Opción 2**: Eureka (Spring Cloud Netflix) como servidor de registro propio.
  - **Pros**: patrón clásico de Spring Cloud, muy documentado; los
    servicios se registran y el gateway resuelve con `lb://`; habilita múltiples instancias +
    client-side load balancing; demuestra el concepto de service registry de forma visible.
  - **Contras**: un componente y un deploy más para mantener; sobre Kubernetes es redundante con el DNS nativo del clúster.
- **Opción 3**: Discovery nativo de Kubernetes (DNS del clúster).
  - **Pros**: sin componente extra; el orquestador ya sabe dónde vive cada pod; es el estándar de
    facto en producción moderna sobre k8s.
  - **Contras**: exige correr sobre Kubernetes, que este proyecto no usa como base; no demuestra el
    patrón de discovery como pieza aparte.

## Decisión

Se elige la **Opción 2**: Eureka. Es la herramienta clásica del ecosistema Spring Cloud, está muy
documentada, y demuestra explícitamente el concepto de service registry con client-side load balancing.

El argumento decisivo es didáctico: el objetivo del proyecto es consolidar los patrones de
microservicios, y un registro de servicios visible los demuestra mejor que el DNS implícito de un
orquestador que este proyecto no usa. Se implementa Eureka a sabiendas de que en Kubernetes se
reemplazaría por DNS nativo, y ese trade-off queda documentado.

## Consecuencias

- **Positivas**:
  - Sin URLs hardcodeadas: el gateway rutea a `lb://products-service`.
  - Soporta múltiples instancias + balanceo de carga del lado del cliente.
  - Demuestra el patrón de discovery de forma explícita y con abundante material de referencia.
- **Negativas / Riesgos**:
  - Un componente más (`eureka-server`) que levantar y mantener.
  - Spring Cloud Netflix / Eureka es tecnología en mantenimiento: en un stack productivo moderno sobre
    k8s no sería la elección. Mitigación: dejar documentado que la elección es didáctica y que en
    producción sobre Kubernetes se usaría DNS nativo. Ese reconocimiento es la señal de criterio.
  - Eureka no está en el camino de la request: si se cae, se degrada el descubrimiento de direcciones
    nuevas, no las llamadas ya resueltas. Conviene tenerlo claro operativamente.
