# ShareYourTrip API Gateway

API Gateway basado en Spring Cloud Gateway para centralizar y securizar el acceso a los microservicios de ShareYourTrip.

## Arquitectura

```
Cliente
   ↓ (con JWT)
API Gateway (8080)
   ↓ (valida JWT + añade headers X-User-Id, X-User-Roles)
Microservicios (8081-8084)
   ↓ (validan header X-User-Id)
Procesan la petición
```

## Servicios

| Servicio | Puerto | Descripción |
|----------|--------|-------------|
| API Gateway | 8080 | Punto de entrada único, validación JWT centralizada |
| Users | 8081 | Gestión de usuarios y autenticación |
| Accommodations | 8082 | Gestión de alojamientos |
| Bookings | 8083 | Gestión de reservas |
| Reviews | 8084 | Gestión de reseñas |

## Rutas del Gateway

| Ruta Gateway | Microservicio | Puerto |
|-------------|---------------|--------|
| `/auth/**` | Users | 8081 |
| `/users/**` | Users | 8081 |
| `/accommodations/**` | Accommodations | 8082 |
| `/booking-requests/**` | Bookings | 8083 |
| `/host-reviews/**` | Reviews | 8084 |
| `/traveler-reviews/**` | Reviews | 8084 |

**Nota**: Las rutas del gateway no tienen prefijo `/api`. El gateway redirige directamente a las rutas de los microservicios.

## Seguridad JWT

### Flujo de Autenticación

1. **Login** (vía Gateway)
   ```http
   POST http://localhost:8080/auth/login
   Content-Type: application/json

   {
     "email": "usuario@example.com",
     "password": "password"
   }
   ```

2. **Respuesta** con token JWT
   ```json
   {
     "id": 1,
     "firstName": "Juan",
     "lastName": "Pérez",
     "email": "usuario@example.com",
     "roles": ["TRAVELER"],
     "message": "Login exitoso",
     "token": "eyJhbGciOiJIUzUxMiJ9..."
   }
   ```

3. **Acceder a endpoints protegidos** con el token
   ```http
   GET http://localhost:8080/users
   Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
   ```

### Cómo Funciona el Gateway

1. **Recibe** la petición con el header `Authorization: Bearer <token>`
2. **Valida** el JWT usando la clave secreta compartida (JJWT con HS512)
3. **Extrae** `userId` y `roles` del token
4. **Añade headers** a la petición:
   - `X-User-Id`: ID del usuario (subject del JWT)
   - `X-User-Roles`: Roles del usuario (del claim "roles")
5. **Reenvía** la petición al microservicio correspondiente

### Filtro JWT Global

El gateway usa un `GlobalFilter` (`JwtAuthenticationFilter`) que:
- Se ejecuta antes de otros filtros (order -100)
- Permite rutas `/auth/** sin autenticación
- Valida tokens JWT con JJWT
- Devuelve 401 si el token es inválido o no se proporciona
- Añade headers con información del usuario para los microservicios

### Configuración JWT

La clave secreta JWT está configurada en:
- `application.yml` del gateway: `jwt.secret`
- `application.yaml` del servicio users: `jwt.secret`
- Expiración del token: 24 horas (86400000 ms)

**Nota**: En producción, usa variables de entorno o un secret manager para la clave JWT.

## Seguridad en Microservicios

Los microservicios tienen su propia capa de seguridad para evitar acceso directo:

### Filtro de Validación de Origen

Cada microservicio tiene un filtro `GatewayAuthenticationFilter` que:
- Valida la presencia del header `X-User-Id`
- Rechaza peticiones directas sin este header (401)
- Permite rutas `/auth/**` en el servicio users (para login)

### Configuración Spring Security

- Spring Security configurado con `permitAll()` (validación manual)
- CSRF deshabilitado
- Filtro `GatewayAuthenticationFilter` añadido antes de `UsernamePasswordAuthenticationFilter`

### Por qué esta Arquitectura

1. **Gateway como único punto de entrada**: Validación JWT centralizada
2. **Microservicios blindados**: Rechazan peticiones directas sin header del gateway
3. **Tokens firmados**: JWT validado con clave secreta compartida
4. **Separación de responsabilidades**: Gateway maneja autenticación, microservicios manejan autorización basada en headers

### Acceso Directo

Si intentas acceder directamente a `http://localhost:8081/users`:
- El microservicio Users recibe la petición
- El filtro `GatewayAuthenticationFilter` detecta que falta el header `X-User-Id`
- Devuelve 401 con mensaje "Acceso denegado: petición no autorizada"

## Cómo Iniciar los Servicios

### Método 1: Script Batch (Windows)

```batch
start-services.bat
```

Los servicios se inician en orden:
1. Gateway (8080) - primero, es el punto de entrada
2. Users (8081)
3. Accommodations (8082)
4. Bookings (8083)
5. Reviews (8084)

El script compila automáticamente cada servicio si el JAR no existe.

### Método 2: Manual

Compilar y ejecutar cada servicio:

```bash
# Gateway
cd ShareYourTrip_api_gateway
mvnw.cmd clean package -DskipTests
java -jar target/*.jar

# Users
cd ShareYourTrip_ms_users
mvnw.cmd clean package -DskipTests
java -jar target/*.jar

# ... etc para los demás servicios
```

## Actualizar Postman

Configura tu colección Postman para usar el gateway:

1. **Variables de colección**:
   - `baseUrl`: `http://localhost:8080`
   - `token`: (se llena automáticamente después del login)

2. **Login request**:
   - URL: `{{baseUrl}}/auth/login`
   - Test script:
     ```javascript
     if (pm.response.code === 200) {
         const jsonData = pm.response.json();
         pm.collectionVariables.set("token", jsonData.token);
     }
     ```

3. **Todas las demás requests**:
   - Añadir header `Authorization: Bearer {{token}}`
   - Usar `{{baseUrl}}` en lugar de URLs directas

## Ventajas de la Arquitectura

- **Centralización**: Validación JWT en un solo punto (gateway)
- **Simplificación**: Los microservicios validan solo el header del gateway
- **Propagación**: Headers `X-User-Id` y `X-User-Roles` permiten que los microservicios identifiquen al usuario
- **Escalabilidad**: Fácil añadir nuevos microservicios configurando rutas
- **Seguridad**: El gateway actúa como barrera de seguridad y los microservicios rechazan acceso directo

## Cómo Cerrar los Servicios

Cierra las ventanas de CMD abiertas por el script, o usa el Task Manager para terminar los procesos `java.exe`.

## Tecnologías

- Spring Boot 4.0.5
- Spring Cloud Gateway
- Spring Security (WebFlux - deshabilitado en gateway, activo en microservicios)
- JJWT (JWT generation/validation con HS512)
- Java 17
- PostgreSQL (para persistencia de datos)

## Próximos Pasos Opcionales

1. **Rate Limiting**: Añadir límites de tasa en el gateway
2. **Circuit Breaker**: Implementar resilience4j para manejar fallos
3. **Logging centralizado**: Configurar logs estructurados
4. **Monitoring**: Añadir Spring Actuator y Prometheus
5. **OAuth2**: Migrar a OAuth2 para tokens más seguros y refresh tokens
6. **API Documentation**: Añadir Swagger/OpenAPI en el gateway
