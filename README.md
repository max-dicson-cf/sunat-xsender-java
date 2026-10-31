# sunat-xsender-java

Comunicación con SUNAT: envío de comprobantes electrónicos, consulta de estado,
procesamiento de CDR y tickets. Basado en Apache Camel y CXF.

> Este proyecto es un fork mantenido independientemente del módulo XSender de
> [Project OpenUBL / xhandler](https://github.com/project-openubl/xhandler).
> Su base inicial corresponde a `5.1.2-SNAPSHOT` (POM padre
> `io.github.project-openubl:xhandler-parent:5.1.2-SNAPSHOT`). Agradecemos a
> Carlos Esteban Feria Vila y a Project OpenUBL por el trabajo original que hizo
> posible este punto de partida.
>
> El fork conserva la licencia Apache 2.0 (ver [LICENSE](LICENSE)), mantiene su
> código fuente abierto y continuará evolucionando de forma independiente. No es
> un proyecto afiliado ni respaldado oficialmente por Project OpenUBL.

## Alcance

Esta librería contiene únicamente el núcleo de **XSender**: modelos de destinos
SUNAT, analizadores de archivos ZIP/XML, cliente Camel para los servicios SOAP de
SUNAT (`billService`, `billConsultService`, `billValidService`) y parsing de
respuestas (CDR, tickets).

**No incluye**: extensiones Quarkus/Spring Boot, starters, ejemplos de aplicación ni
integraciones de servidor. Es una librería Java pura instanciable directamente.

**Relación con XBuilder**: el núcleo de XSender no depende de XBuilder en tiempo de
compilación. Opera sobre XML/ZIP ya construidos. Si tu aplicación necesita construir
el XML antes de enviarlo, agrega `sunat-xbuilder-java` como dependencia adicional.

## Requisitos

* Java 17+ (el objetivo final es Java 21; el código ya usa APIs modernas).
* Apache Maven 3.8+.

> **Nota sobre JDK 21**: el entorno de trabajo actual disponible es JDK 17, por lo
> que el build fue validado con JDK 17. Antes de publicar con JDK 21 se recomienda
> ejecutar `mvn clean test` con JDK 21 y actualizar plugins/dependencias si fuera
> necesario.

## Compilación

```bash
mvn -U clean test
mvn clean package
mvn clean install
```

Resultado observado en este entorno (JDK 17, Maven 3.8.1):

```text
Tests run: 25, Failures: 0, Errors: 0, Skipped: 1
BUILD SUCCESS
```

El test omitido (`SendBillTest`) está etiquetado como `sunat-integration` porque
contacta servicios reales de SUNAT. Para ejecutarlo:

```bash
mvn test -Psunat-integration
```

> **Advertencia**: los tests `sunat-integration` usan credenciales demo públicas
> (`12345678959MODDATOS` / `MODDATOS`) y endpoints de SUNAT. No ejecutarlos contra
> producción ni con credenciales reales.

## API pública principal

* `io.github.project.openubl.xsender.company.CompanyURLs`
* `io.github.project.openubl.xsender.company.CompanyCredentials`
* `io.github.project.openubl.xsender.files.BillServiceXMLFileAnalyzer`
* `io.github.project.openubl.xsender.files.BillServiceFileAnalyzer`
* `io.github.project.openubl.xsender.files.ZipFile`
* `io.github.project.openubl.xsender.camel.StandaloneCamel`
* `io.github.project.openubl.xsender.camel.utils.CamelUtils`
* `io.github.project.openubl.xsender.sunat.BillServiceDestination`
* `io.github.project.openubl.xsender.sunat.BillConsultServiceDestination`
* `io.github.project.openubl.xsender.sunat.BillValidServiceDestination`
* `io.github.project.openubl.xsender.models.SunatResponse`

## Ejemplo mínimo (sin enviar a SUNAT)

```java
import io.github.project.openubl.xsender.camel.StandaloneCamel;
import io.github.project.openubl.xsender.company.CompanyCredentials;
import io.github.project.openubl.xsender.company.CompanyURLs;
import org.apache.camel.CamelContext;
import org.apache.camel.main.Main;

public class Ejemplo {
    public static void main(String[] args) throws Exception {
        CompanyURLs urls = CompanyURLs.builder()
                .invoice("https://e-beta.sunat.gob.pe/ol-ti-itcpfegem-beta/billService")
                .despatch("https://e-beta.sunat.gob.pe/ol-ti-itemision-otroscpe-gem-beta/billService")
                .perceptionRetention("https://e-beta.sunat.gob.pe/ol-ti-itemision-otroscpe-gem-beta/billService")
                .build();

        CompanyCredentials credentials = CompanyCredentials.builder()
                .username("12345678959MODDATOS")
                .password("MODDATOS")
                .build();

        Main mainCamel = StandaloneCamel.getInstance().getMainCamel();
        mainCamel.start();
        CamelContext camelContext = mainCamel.getCamelContext();

        System.out.println("CamelContext status=" + camelContext.getStatus());
        System.out.println("Invoice endpoint=" + urls.getInvoice());
        System.out.println("Credentials username=" + credentials.getUsername());

        mainCamel.stop();
    }
}
```

Para enviar un documento real, construye un `BillServiceXMLFileAnalyzer` sobre el
XML firmado, obtén el `ZipFile` y el `BillServiceDestination`, genera el `CamelData`
con `CamelUtils.getBillServiceCamelData(...)` y úsalo con
`camelContext.createProducerTemplate().requestBodyAndHeaders(...)`.

## Tests

* Tests unitarios locales: `mvn test`.
* Tests de integración SUNAT: `mvn test -Psunat-integration` (requiere conectividad).

## Publicación / JitPack

La primera distribución debe ser un snapshot de una rama o commit, no un release.
JitPack compila el repositorio Git remoto bajo demanda; antes de solicitarlo, el
commit debe estar publicado en GitHub y el repositorio debe ser público (o usar la
configuración de repositorios privados de JitPack).

`jitpack.yml` usa Java 17 y ejecuta:

```bash
mvn -B -U clean install -Ppublication -DskipTests
```

El perfil `publication` adjunta los JAR de fuentes y Javadoc. JitPack no requiere
una librería, plugin de despliegue, claves GPG ni `distributionManagement`.

Para probar un snapshot sin crear un release, abre el repositorio en JitPack y
solicita el commit corto o la rama:

```text
https://jitpack.io/#max-dicson-cf/sunat-xsender-java
```

Usa un hash de commit para una prueba inmutable o `main-SNAPSHOT` para seguir la
rama principal. JitPack mostrará las coordenadas Maven definitivas del build; usa
esas coordenadas en los consumidores en vez de asumir las de una futura release.

Todo snapshot debe superar localmente antes:

```bash
mvn -U clean test
mvn -U clean install -Ppublication
```

### Consumo desde JitPack

Agrega el repositorio:

```xml
<repositories>
    <repository>
        <id>jitpack.io</id>
        <url>https://jitpack.io</url>
    </repository>
</repositories>
```

## Atribución

* Autor original: Carlos Esteban Feria Vila y el proyecto Project OpenUBL.
* Repositorio fuente: https://github.com/project-openubl/xhandler
* Este repositorio es un fork/derivado que conserva la licencia Apache 2.0 y da
  mantenimiento independiente al módulo XSender.
