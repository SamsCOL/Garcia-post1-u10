# Post-contenido — Unidad 10: Pruebas de Software en Aplicaciones Web

## Descripción
Repositorio del laboratorio de la Unidad 10 de Programación Web —
Séptimo Semestre. Contiene un único proyecto Maven Spring Boot
(catalogo-productos, paquete raíz `com.universidad.catalogo`) con una
suite de pruebas unitarias, de integración y de capa web sobre un
catálogo de productos, más una colección Postman versionada y un
umbral de cobertura verificado con JaCoCo.

## Estructura del repositorio (plana)
Todos los archivos están al mismo nivel, sin subcarpetas. El `pom.xml`
está configurado para leer el código fuente desde la raíz
(`<sourceDirectory>${project.basedir}</sourceDirectory>`) y separa
producción de pruebas por nombre: los archivos `*Test.java` se
compilan como pruebas y el resto como código de la aplicación. La
separación por capas se conserva en la declaración `package` de cada
clase:

| Archivo | Paquete / rol |
|---|---|
| `CatalogoApplication.java` | `com.universidad.catalogo` — clase principal |
| `Producto.java` | `...model` — entidad JPA |
| `ProductoRepository.java` | `...repository` — `JpaRepository` con `findByCategoria` |
| `ProductoService.java` | `...service` — reglas de negocio |
| `ProductoController.java` | `...controller` — API REST `/api/productos` |
| `ProductoServiceTest.java` | prueba unitaria (Mockito) |
| `ProductoRepositoryTest.java` | prueba de integración (`@DataJpaTest` + H2) |
| `ProductoControllerTest.java` | prueba de capa web (`@WebMvcTest` + MockMvc) |
| `ColeccionCatalogo.json` | colección Postman "API Catálogo de Productos" |
| `env-local.json` | entorno Postman "Catalogo-Local" (`baseUrl`) |
| `application.properties` | configuración H2 |
| `pom.xml` | build Maven + JaCoCo |

## Parte 1 — Pruebas Unitarias con Mockito y de Integración con @DataJpaTest
**ProductoServiceTest** usa `@ExtendWith(MockitoExtension.class)`,
`@Mock` para el repositorio e `@InjectMocks` para el servicio. Se usa
Mockito aquí porque lo que se prueba es la lógica de negocio
(validaciones de nombre y precio, búsqueda por id) y no la base de
datos: el mock permite controlar qué devuelve el repositorio y
verificar con `verify(repo, never()).save(any())` que un dato inválido
nunca llega a persistirse. Cubre el caso feliz y los casos de error
(precio negativo, nombre vacío, producto no encontrado, precio cero en
la actualización).

**ProductoRepositoryTest** usa `@DataJpaTest` con `TestEntityManager`
contra H2 en memoria. Aquí no se usa Mockito porque lo que se quiere
comprobar es justamente que la consulta derivada `findByCategoria`
genera el SQL correcto contra una base real; cada test se revierte al
terminar, así que `setUp()` inserta los mismos dos productos sin que se
acumulen.

## Parte 2 — Pruebas Web con @WebMvcTest, Postman/Newman y JaCoCo
**ProductoControllerTest** usa `@WebMvcTest(ProductoController.class)`
con `MockMvc` y `@MockBean ProductoService` para probar la capa web sin
levantar el contexto completo: verifica 200 (producto existe y listado
por categoría), 404 (`EntityNotFoundException` → `@ExceptionHandler`),
201 (creación) y 400 (`IllegalArgumentException` → `@ExceptionHandler`).

La colección Postman **"API Catálogo de Productos"**
(`ColeccionCatalogo.json`) ejecuta 4 requests en orden con test scripts
que verifican status y body contra la aplicación en ejecución real:

1. `POST /api/productos` → 201, guarda `productoId`, tiempo < 500 ms
2. `GET /api/productos/{{productoId}}` → 200, nombre "Teclado mecánico"
3. `GET /api/productos?categoria=ELECTRONICA` → 200, lista con el producto
4. `GET /api/productos/99999` → 404

JaCoCo (`jacoco-maven-plugin` 0.8.11, goals `prepare-agent`, `report`
y `check`) verifica una cobertura mínima del 70 % de líneas sobre todo
el módulo con `mvn verify`, excluyendo la clase `*Application` y el
paquete `model`.

### Orden en que se agregaron los tipos de prueba
1. Pruebas unitarias del servicio (Mockito) — base de la pirámide, las más rápidas.
2. Pruebas de integración del repositorio (`@DataJpaTest`).
3. Pruebas de la capa web (`@WebMvcTest`).
4. Pruebas de extremo a extremo con Postman/Newman sobre HTTP real.
5. Por último, JaCoCo con el umbral del 70 %, cuando las tres capas ya estaban cubiertas.

## Decisiones de diseño
1. H2 se usa también para ejecución local (no solo pruebas): el
   objetivo de la unidad es la escritura de pruebas, no el modelado
   de persistencia contra un motor externo como en la Unidad 8.
2. buscarPorId lanza EntityNotFoundException en vez de retornar
   null, para que el contrato del método sea explícito y verificable
   con assertThrows.
3. ProductoRepositoryTest usa @DataJpaTest en vez de @SpringBootTest
   para aislar la prueba a la capa de persistencia y mantener la
   suite rápida.
4. ProductoControllerTest usa @WebMvcTest en vez de @SpringBootTest
   con TestRestTemplate, para no duplicar cobertura con las pruebas
   de servicio y repositorio ya existentes.
5. La colección Postman se versiona como archivo .json, no como
   test JUnit, porque prueba el contrato HTTP completo como caja
   negra contra un proceso real, un nivel distinto al de MockMvc.
6. JaCoCo se configura al cierre de la Parte 2, no de la Parte 1,
   para medir la cobertura agregada de las tres capas y evitar que
   el umbral del 70% se cumpla artificialmente sobre una suite
   incompleta.
7. Este laboratorio NO incluye Selenium WebDriver: el dominio es una
   API REST pura sin vista Thymeleaf, y fabricar una vista solo para
   justificar Selenium contradiría la pirámide de pruebas. La
   colección Postman cumple el rol de prueba de extremo a extremo
   sobre la superficie real de la aplicación.
8. Estructura plana: todos los archivos están en la raíz del
   repositorio. El `pom.xml` redefine `sourceDirectory`,
   `testSourceDirectory` y `resources`, y separa pruebas de producción
   por el sufijo `*Test.java`; los paquetes Java mantienen la
   organización por capas.

## Cómo compilar y ejecutar
Requisitos: Java 17+, Maven 3.9.x, Node.js 18+ (para Newman).

1. Clonar el repositorio: `git clone [URL-del-repo]` y entrar a la carpeta.
2. Ejecutar solo las pruebas: `mvn test`
   (o una sola clase: `mvn test -Dtest=ProductoServiceTest`).
3. Ejecutar la suite completa con verificación de cobertura:
   `mvn clean verify` (genera el reporte JaCoCo en
   `target/site/jacoco/index.html` y falla si la cobertura < 70%).
4. Levantar la aplicación: `mvn spring-boot:run`
5. Con la aplicación en ejecución, correr la colección Postman:
   `newman run ColeccionCatalogo.json --environment env-local.json`

### Evidencia del gate de cobertura
Para comprobar que `jacoco:check` hace fallar el build, se renombró
temporalmente `ProductoControllerTest.java` (por ejemplo a
`ProductoControllerTest.java.bak`) y se ejecutó `mvn clean verify`:
el build falla con *"Coverage checks have not been met"*. Luego se
restauró el nombre original y el build volvió a pasar.

## Capturas de pantalla
![Reporte de cobertura JaCoCo >= 70%](jacoco-reporte.png)
![Build fallando por cobertura insuficiente (evidencia del gate)](jacoco-build-fallido.png)
![Tests unitarios, de integración y web en verde](tests-verde.png)
![Postman Runner con 0 failures](postman-runner.png)
