# AGENTS.md — sunat-xsender-java

Contexto para agentes de OpenCode que trabajen en este repositorio.

## Propósito

Librería Java pura para comunicarse con SUNAT: envío de comprobantes electrónicos,
consulta de estado, procesamiento de CDR y tickets. Basada en Apache Camel + CXF.
Código extraído del monorepo `project-openubl/xhandler` (versión 5.1.2-SNAPSHOT).

## Cómo construir y probar

```bash
mvn -U clean test               # tests unitarios locales, sin llamar a SUNAT
mvn test -Psunat-integration    # tests opt-in que contactan servicios SUNAT
mvn clean package
mvn clean install
```

Resultado esperado en JDK 17:

```text
Tests run: 25, Failures: 0, Errors: 0, Skipped: 1
BUILD SUCCESS
```

> El entorno actual solo tiene JDK 17. Antes de publicar tags, validar con JDK 21.

## Estructura del proyecto

```text
src/main/java/io/github/project/openubl/xsender/
├── camel/            # Integración Apache Camel (rutas, utilidades, StandaloneCamel)
│   ├── routes/       # SunatRouteBuilder y procesadores de respuesta SOAP/REST
│   └── utils/        # CamelUtils, CamelData
├── company/          # CompanyURLs y CompanyCredentials
├── files/            # Análisis de archivos XML/ZIP para SUNAT
│   ├── xml/          # XmlContentProvider, XmlHandler, DocumentType
│   └── exceptions/
├── models/           # SunatResponse, Status, Metadata, DTOs REST
├── sunat/            # Destinos SOAP: BillServiceDestination, BillConsultServiceDestination, etc.
├── utils/            # CdrReader, ByteUtils
└── Constants.java    # URIs de Camel

src/main/resources/wsdl/    # WSDL de billService, billConsultService y billValidService
src/main/resources/         # application.properties (config Camel)
src/test/java/              # Tests unitarios y tests de integración SUNAT
src/test/resources/ubl/     # XMLs de ejemplo para tests
src/test/resources/xmls/    # Más XMLs de prueba
```

## Convenciones importantes

* **Paquetes y namespaces**: no cambiar los paquetes Java (`io.github.project.openubl.xsender.**`) salvo necesidad demostrada.
* **API pública**: `CompanyURLs`, `CompanyCredentials`, `BillServiceXMLFileAnalyzer`, `StandaloneCamel`, `CamelUtils`, `BillServiceDestination`, `BillConsultServiceDestination`, `BillValidServiceDestination`, `SunatResponse`.
* **WSDL**: no modificar sin revisar que el plugin `cxf-codegen-plugin` siga generando las clases correctamente.
* **Tests SUNAT**: la clase `SendBillTest` está etiquetada con `@Tag("sunat-integration")` y se excluye por defecto. Para ejecutarla: `mvn test -Psunat-integration`.

## Dependencias clave

* `org.apache.camel:*` — orquestación de rutas de envío (core, cxf-soap, http, jackson, etc.).
* `org.apache.cxf:cxf-codegen-plugin` — generación de clientes SOAP a partir de WSDL.
* `org.jodd:jodd-core` — utilidades (ZipBuilder).
* `commons-codec:commons-codec` — codificación.
* `jakarta.xml.bind:jakarta.xml.bind-api` — JAXB.

## Relación con sunat-xbuilder-java

El núcleo de XSender **no depende** de XBuilder. Opera sobre XML/ZIP ya construidos.
Si una aplicación necesita construir el XML antes de enviarlo, debe agregar
`sunat-xbuilder-java` como dependencia adicional.

## Qué NO cambiar sin consultar

* URIs de servicios SUNAT en `Constants.java`.
* Nombres de operaciones SOAP en `BillServiceDestination.SoapOperation`.
* Lógica de `SunatRouteBuilder` y procesadores de respuesta.
* WSDL y configuración del plugin `cxf-codegen-plugin`.

## Tests e integración

* `mvn test` ejecuta solo tests unitarios locales.
* `mvn test -Psunat-integration` ejecuta tests que contactan endpoints reales de SUNAT.
* No uses credenciales SOL ni certificados reales en tests.

## Notas para el agente

* No agregues dependencias de Spring Boot, Quarkus runtime ni starters en el artefacto publicado.
* Si un test de integración SUNAT falla por cambios en los servicios de SUNAT, no lo "arregles" cambiando expectativas sin confirmar; mejor márcalo como `@Disabled` o `@Tag("sunat-integration")`.
* Los warnings de CXF sobre puertos WSDL son esperados y no bloquean el build.
