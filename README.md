# microservicios-canchas-reservas

## Descripción general

Proyecto compuesto por dos microservicios independientes que se comunican 
entre sí mediante **Spring Cloud OpenFeign**:

- **Servicio Canchas** → puerto 8081
- **Servicio Reservas** → puerto 8082

---

## a. Descripción de esta forma de comunicación

**Spring Cloud OpenFeign** es un cliente HTTP declarativo que simplifica 
la comunicación entre microservicios. A diferencia de **RestTemplate** 
(bloqueante/imperativo) y **WebClient** (reactivo), con OpenFeign solo 
se define una **interfaz Java** con anotaciones y Spring genera 
automáticamente el código de la llamada HTTP.

### ¿Cómo funciona en este proyecto?

Cuando un cliente envía una solicitud POST para crear una reserva, el 
**Servicio de Reservas** consulta al **Servicio de Canchas** mediante 
OpenFeign para verificar que la cancha exista:

- Si la cancha **existe** → la reserva se guarda en la base de datos 
- Si la cancha **no existe** → se retorna un error 

### Ventajas de OpenFeign
- Código más limpio y legible
- No se escribe lógica de llamadas HTTP manualmente
- Se integra de forma natural con Spring Boot
- El método `obtenerCancha()` se ve como una llamada local normal

---

## b. Cambios implementados en el proyecto

### Servicio de Canchas
Se mantuvo **intacto** — no requirió ningún cambio ya que OpenFeign 
consume su API REST existente en el puerto 8081.

### Servicio de Reservas

#### 1. Dependencias agregadas en `pom.xml`
Se agregó `spring-cloud-starter-openfeign` y el bloque 
`dependencyManagement` con la versión de Spring Cloud compatible:
```xml
<properties>
    <spring-cloud.version>2025.1.0</spring-cloud.version>
</properties>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### 2. Eliminación del package `config`
Se eliminó la clase `WebClientConfig` que contenía el bean de `WebClient`,
ya que OpenFeign no requiere configuración manual de cliente HTTP.

#### 3. Creación del package `client` con `CanchaClient`
Se creó la interfaz `CanchaClient` que reemplaza la lógica de `WebClient`.
Con OpenFeign basta con declarar la interfaz y Spring genera 
automáticamente el código de la llamada HTTP:
```java
@FeignClient(name = "serviciocanchas", url = "http://localhost:8081")
public interface CanchaClient {

    @GetMapping("/api/canchas/{id}")
    Cancha obtenerCancha(@PathVariable Long id);
}
```

#### 4. Modificación de `ReservaServicesImpl`
Se reemplazó `WebClient` por `CanchaClient` en el método `crearReserva`:

**Antes con WebClient:**
```java
@Autowired
private WebClient webClient;

Cancha cancha = webClient
    .get()
    .uri("/api/canchas/" + reserva.getCanchaId())
    .retrieve()
    .bodyToMono(Cancha.class)
    .block();
```

**Ahora con OpenFeign:**
```java
@Autowired
private CanchaClient canchaClient;

Cancha cancha = canchaClient.obtenerCancha(reserva.getCanchaId());
```

La lógica de negocio se mantiene igual — si la cancha no existe 
se lanza un error y la reserva no se guarda.

#### 5. Anotación `@EnableFeignClients` en clase principal
Se agregó la anotación en `ServicioreservasApplication.java` para 
habilitar el escaneo de interfaces Feign:
```java
@SpringBootApplication
@EnableFeignClients
public class ServicioreservasApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServicioreservasApplication.class, args);
    }
}
```

---

## Cómo ejecutar

1. Clonar el repositorio
2. Ejecutar primero el **Servicio de Canchas** (puerto 8081)
3. Ejecutar el **Servicio de Reservas** (puerto 8082)
4. Probar con Postman

### Crear una reserva (POST)
```
POST http://localhost:8082/api/reservas
Content-Type: application/json

{
    "canchaId": 1,
    "nombreCliente": "Juan Pérez",
    "fecha": "2026-03-20",
    "horaInicio": "10:00",
    "horaFin": "11:00"
}
```

### Listar reservas (GET)
```
GET http://localhost:8082/api/reservas
```

---

## Tecnologías
- Java 21
- Spring Boot 4.0.3
- Spring Cloud OpenFeign 2025.1.0
- MySQL
- Maven
