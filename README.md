# Práctico 5

## Diseño seguro: tablero público de puntajes

**Desarrollo de Software Seguro** 



## Resumen

Esta iteración incorpora el nuevo requerimiento: un tablero público en Internet donde cualquier persona ve en vivo los puntajes del torneo y consulta torneos anteriores. La preocupación del presidente de la federación es que alguien desde Internet modifique una nota.

La decisión de diseño central es no exponer el sistema de gestión (core) a Internet. En su lugar se agrega una vía de publicación unidireccional: un servicio Publicador, ubicado en la red interna, lee únicamente los puntajes confirmados y los empuja hacia afuera (conexión solo saliente) a un almacén público de solo lectura. Desde Internet no existe ningún camino de red ni función de escritura que llegue al core. Esto elimina el vector de modificación de notas, en lugar de intentar filtrarlo.

## 1. Paso 1: Determinar el alcance

### 1.1 Objetivos de negocio (se agregan a los de la iteración 1)

- Publicar resultados en vivo y el histórico de torneos de forma pública, accesible y transparente.
- Garantizar que ninguna persona desde Internet pueda alterar un puntaje (integridad).
- Que un ataque o pico de tráfico sobre el tablero no afecte la carga de notas por los jueces (disponibilidad).
- Publicar solo los datos necesarios, protegiendo los datos personales de gimnastas (probablemente menores) y jueces.
- Mantener la reputación de la federación: un puntaje falso visible públicamente sería un incidente de transparencia.

### 1.2 Requerimientos normativos

- **Ley N.º 18.331 (Protección de Datos Personales):** Principios de finalidad, proporcionalidad y minimización. Se publican solo los campos imprescindibles, con aviso de privacidad y consentimiento (de tutores en caso de menores). Debe existir un proceso de oposición y baja de datos del histórico.
- **Transparencia y equidad deportiva:** Cualquier involucrado debe poder conocer los puntajes de todos los competidores. El tablero público lo cumple, y las correcciones de notas deben quedar visibles (no ocultas).

## 2. Arquitectura y Diseño


### 2.1 Arquitectura propuesta

Se divide el sistema en tres zonas con fronteras de confianza explícitas:

1. **Internet (no confiable):** Público anónimo.
2. **DMZ / hosting público (confianza baja):** CDN y WAF, SPA pública estática, API pública (solo lectura) y almacén público. Se aloja fuera de la arena (hosting o cloud), de modo que el tráfico de Internet nunca llegue a la red de la arena.
3. **Red interna de la arena (confianza alta):** El sistema de la iteración 1 (App Mobile de jueces, Web de mesa de control y admin, REST core, base de datos core) más el nuevo Publicador.

#### Componentes nuevos

| Componente | Zona | Función | Controles clave |
| :--- | :--- | :--- | :--- |
| **CDN + WAF** | Internet / DMZ | Punto único de entrada público. Cachea respuestas (TTL de 2 a 5 s para vivo, largo para histórico). | Solo "GET/HEAD/OPTIONS", rate limiting, TLS 1.2+ con HSTS, protección DDoS. |
| **SPA pública** | DMZ | Interfaz del tablero (contenido estático). | Sin secretos, CSP estricta, sin formularios de carga de datos. |
| **API pública** | DMZ | Expone únicamente endpoints de lectura de resultados. | Sin endpoints de escritura, usuario de base de datos de solo lectura, DTOs mínimos, validación de parámetros. |
| **Almacén público** | DMZ | Copia de solo lectura de datos proyectados (BD pública independiente + snapshots JSON inmutables por torneo cerrado). | Sin datos personales sensibles, ni usuarios, ni contraseñas, ni tokens. |
| **Publicador** | Red interna | Lee del core vía una vista de solo lectura los puntajes en estado CONFIRMADA y los publica con número de secuencia. | Conexión solo saliente con HTTPS y mTLS, credencial con permiso de escritura únicamente sobre el almacén público, cola local con reintentos. |

 **Cambio menor en el core:** Se agrega el estado de la nota (PENDIENTE → CONFIRMADA). Solo la mesa de control confirma, y solo las confirmadas se publican. Esto evita publicar errores de tipeo en vivo (problema histórico de la federación).

#### Diagrama de arquitectura (alto nivel)
Los datos solo cruzan hacia afuera: el Publicador, dentro de la red interna, empuja los puntajes confirmados al Store público; Internet nunca inicia una conexión hacia el core.

#### Estrategias para impedir modificar una nota desde Internet:
1. No existe ningún endpoint de escritura en la capa pública.
2. El WAF bloquea cualquier método distinto de GET/HEAD/OPTIONS.
3. El usuario de base de datos de la API pública es de solo lectura.
4. Firewall con política de denegación por defecto: no hay conexiones entrantes hacia la red interna.
5. La capa pública no posee credenciales del core.
6. Un job de reconciliación compara hashes entre la fuente de verdad (core) y el almacén público, y alerta ante diferencias.

### 2.2 Dependencias externas (nuevas)

| ID | Descripción |
| :---: | :--- |
| **6** | La API pública corre en Spring Framework sobre Linux hardenizado, en hosting público fuera de la arena. |
| **7** | La SPA pública se sirve desde un CDN con WAF de un proveedor externo (se transfiere parte del riesgo de DoS). |
| **8** | El almacén público usa PostgreSQL independiente en Linux hardenizado, más almacenamiento de objetos para snapshots. |
| **9** | El enlace a Internet de la arena (con enlace de respaldo 4G) se usa únicamente para conexiones salientes del Publicador. |
| **10** | Dominio oficial de la federación, DNS (con DNSSEC) y certificados TLS (con registros CAA). |

Se mantienen las dependencias 1 a 5 de la iteración 1. 

* **Supuesto critico:** La red WiFi y la LAN de la arena no son accesibles desde Internet.

### 2.3 Puntos de entrada (nuevos)

| ID | Nombre | Descripción | Niveles de confianza |
| :---: | :--- | :--- | :--- |
| **4** | Puerto HTTPS público (CDN/WAF) | Único puerto expuesto a Internet. | (1) Anónimo |
| **4.1** | "GET /publico/torneos" | Lista de torneos vigentes y anteriores. | (1) Anónimo, (5) Público |
| **4.2** | "GET /publico/torneos/{id}/resultados" | Resultados en vivo (solo "CONFIRMADAS"). | (5) Público |
| **4.3** | "GET /publico/historico/{id}" | Snapshot inmutable de torneos cerrados. | (5) Público |
| **4.4** | Contenido estático de la SPA pública | HTML, JS y CSS. | (5) Público |
| **5** | Canal de publicación | Endpoint del almacén público que recibe el push. Solo acepta mTLS desde el Publicador. | (6) Servicio Publicador |

Los puntos de entrada 1, 2 y 3 de la iteración 1 no cambian y siguen limitados a la LAN de la arena.

### 2.4 Niveles de confianza (actualizados)

| ID | Nombre | Descripción |
| :---: | :--- | :--- |
| **1** | Anónimo | Usuario conectado a la red de la arena (iteración 1). |
| **2** | Administrador | Administrador de la solución, con acceso a toda la infraestructura. |
| **3** | Mesa de Control | Gestiona el torneo y confirma notas. |
| **4** | Juez | Evalúa rutinas e ingresa notas. |
| **5** | Público (Internet) | Cualquier persona en Internet. Solo lectura de datos públicos. (Nuevo) |
| **6** | Servicio Publicador | Cuenta de servicio interna. Lee la vista de resultados y escribe solo en el almacén público. (Nuevo) |

### 2.5 Activos

- Puntajes confirmados (integridad: el activo más crítico).
- Datos personales de gimnastas y jueces (confidencialidad, Ley N.º 18.331).
- Credenciales, tokens JWT y hashes de contraseñas del core (nunca viajan a la zona pública).
- Credencial de publicación y certificado mTLS del Publicador.
- Disponibilidad del core durante la competencia y del tablero público.
- Reputación y transparencia de la federación.

### 2.6 Diagramas de flujo de datos

**DFD nivel 1:** El sistema ahora tiene un actor externo adicional (Público) que interactúa solo con la capa pública, separada del core por una frontera de confianza.

**DFD nivel 2 – flujos nuevos** (los flujos 1 a 11 de la iteración 1 se mantienen):

| Flujo | Origen → Destino | Datos | Frontera cruzada |
| :---: | :--- | :--- | :--- |
| **12** | Público → CDN/WAF | Requests HTTPS GET (torneo, resultados) | Internet → DMZ |
| **13** | CDN/WAF → Público | Resultados públicos (JSON, HTML) | DMZ → Internet |
| **14** | CDN/WAF → SPA pública / API pública | Requests filtrados (solo lectura) | Interna de DMZ |
| **15** | API pública ↔ Almacén público | Consultas de lectura y resultados | Interna de DMZ |
| **16** | Publicador → BD core | Lectura de la vista de notas CONFIRMADAS (usuario de solo lectura) | Interna de red de la arena |
| **17** | Publicador → Almacén público | Push de eventos y snapshots con secuencia y hash (HTTPS + mTLS, solo saliente) | Red interna → DMZ |

---

## 3. Paso 2: Determinar amenazas (STRIDE) y ranking (DREAD)

$$\text{DREAD} = \frac{\text{Damage} + \text{Reproducibility} + \text{Exploitability} + \text{Affected users} + \text{Discoverability}}{5}$$


| ID | STRIDE | Flujo / componente | Amenaza | D, R, E, A, D | Riesgo inicial |
| :---: | :--- | :--- | :--- | :---: | :---: |
| **13** | Information Disclosure | Flujos 15, 16, 17 | Se publican datos no destinados a ser públicos (documentos, fechas de nacimiento, datos de jueces, usuarios) por sobreexposición de la API o de la réplica. | 8, 10, 8, 10, 10 | **9.2** |
| **15** | Denial of Service | Flujos 12 a 14 | DDoS o scraping masivo al tablero durante el torneo satura la API pública (y, sin aislamiento, el core). | 7, 8, 8, 10, 10 | **8.6** |
| **7** | Tampering | Flujos 12 y 14 | Un atacante desde Internet modifica una nota a través de la capa pública (preocupación del presidente). | 10, 5, 5, 8, 10 | **7.6** |
| **11** | Spoofing | Flujo 13 | Sitio falso (phishing o typosquatting) imita el tablero y muestra puntajes falsos. | 6, 8, 6, 8, 10 | **7.6** |
| **14** | Tampering | Flujo 16 | Se publica una nota provisoria o con error de tipeo y queda visible en vivo (error no malicioso). | 6, 6, 8, 8, 10 | **7.6** |
| **10** | Spoofing | Flujo 17 | Un atacante suplanta al Publicador y publica puntajes falsos en el almacén público. | 10, 5, 4, 10, 6 | **7.0** |
| **12** | Repudiation | Flujos 16 y 17 | La mesa o un juez niega haber confirmado una nota publicada, o la federación niega lo publicado. | 8, 8, 5, 8, 5 | **6.8** |
| **8** | Tampering | Flujos 15 y 17 | Se compromete la API o el almacén público y se altera la copia pública de los puntajes. | 7, 3, 3, 10, 10 | **6.6** |
| **17** | Elevation of Privilege | Zona DMZ | Quien compromete la API pública pivota hacia la red interna (movimiento lateral hacia el core). | 10, 4, 3, 10, 6 | **6.6** |
| **9** | Tampering | Flujos 13 y 17 | Man-in-the-middle altera respuestas al público o el canal de publicación. | 8, 5, 4, 5, 10 | **6.4** |
| **16** | Denial of Service | Flujo 17 | Corte de Internet o saturación del canal impide actualizar el tablero. | 5, 5, 4, 10, 8 | **6.4** |
| **18** | Elevation of Privilege | Flujo 16 | La cuenta del Publicador, si se compromete, se usa para leer más datos del core de los necesarios. | 9, 3, 3, 10, 3 | **5.6** |



## 5. Respuesta a la preocupación del presidente

**¿Puede alguien, desde Internet, modificar la nota de una gimnasta?**

No por diseño. El tablero público es una copia de solo lectura que recibe datos desde adentro. Para alterar una nota original habría que atravesar, en este orden:
1. El WAF (que solo permite lectura).
2. Una API sin funciones de escritura.
3. Una base de datos accesible solo con un usuario de solo lectura.
4. Un firewall sin conexiones entrantes hacia la red interna.
5. El core (que requiere autenticación JWT y control de acceso por roles dentro de la LAN de la arena).

Además, aun si se alterara la copia pública, la reconciliación la detecta y el core la sobrescribe, y la fuente de verdad nunca se modifica.

## Conclusion

## 7. Supuestos y trabajo futuro

- La LAN y WiFi de la arena no son accesibles desde Internet y el SSID de la aplicación sigue siendo de uso exclusivo.
- La federación puede comunicar un dominio oficial único.
- Se definirá la lista exacta de campos públicos junto con asesoría legal (Ley N.º 18.331 y menores de edad).
- **Pruebas de seguridad derivadas del modelo:**
  - Intento de métodos de escritura contra la API pública.
  - Verificación de la lista blanca de campos.
  - Prueba de carga con el tablero saturado mientras los jueces cargan notas.
  - Prueba de reconciliación ante alteración de la copia pública.