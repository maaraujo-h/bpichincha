# Dual Run WSClientes0083 — Insomnia

## 1. Uso

Este script ejecuta consecutivamente el mismo `POST` SOAP contra el servicio legado y el migrado. Después:

- valida que ambas llamadas respondan HTTP 2xx;
- convierte las respuestas XML usando `xml2js`;
- detecta la operación `ConsultarClienteBuroCredito01` o `ConsultarReporteBuroCredito01` a partir del XML enviado;
- compara solamente los campos funcionales de los criterios de aceptación;
- compara siempre los siete campos del bloque `<error>`;
- registra una prueba independiente por cada campo o bloque para que Insomnia muestre con precisión dónde existe una diferencia.

Ubicación recomendada: **Tests → Test Suite → New Test**. Si la versión instalada no muestra Test Suites, puede usarse en **Scripts → After-response**, teniendo presente que el botón **Send** primero ejecutará la petición visible y luego el script realizará el Dual Run.

Antes de ejecutarlo:

1. Cree en el ambiente de Insomnia la variable `dual_run_request_xml` y coloque allí el XML SOAP completo. Esta es la opción más estable para Test Suites.
2. Como alternativa, deje la variable vacía y pegue el script en el After-response de una petición cuyo body sea XML raw; el script intentará reutilizar `insomnia.request.body.raw`.
3. Si el servicio requiere `SOAPAction`, cree `dual_run_soap_action` con su valor. Si no se requiere, puede omitirla.
4. No configure manualmente `Content-Length`; Insomnia lo calculará.

## 2. Script JavaScript

```javascript
/**
 * WSClientes0083 - Dual Run Legado vs. Migrado
 * Compatible con scripts modernos de Insomnia.
 */

const xml2js = require('xml2js');

// Compatibilidad: After-response expone insomnia.test/expect; Test Suite
// puede exponer expect global y considerar el propio script como un test.
const expectValue = typeof insomnia.expect === 'function'
  ? insomnia.expect.bind(insomnia)
  : expect;
const testCase = typeof insomnia.test === 'function'
  ? insomnia.test.bind(insomnia)
  : (name, assertion) => {
      console.log(`[TEST] ${name}`);
      assertion();
    };

const ENDPOINTS = {
  legado: 'http://10.60.128.85:2000/IntegrationBus/soap/WSClientes0083',
  migrado: 'http://tpr-msa-sp-wsclientes0083.apps.ocptest.uiotest.bpichinchatest.test/IntegrationBus/soap/WSClientes0083',
};

const ERROR_FIELDS = [
  'codigo',
  'mensaje',
  'mensajeNegocio',
  'tipo',
  'recurso',
  'componente',
  'backend',
];

// CA-01 y CA-05: salida principal, consolidación, deuda y clasificación.
const CLIENTE_SCALAR_PATHS = [
  'usuarioBancs',
  'nombreUsuario',
  'nombreAgencia',
  'identificacion',
  'tipoIdentificacion',
  'nombreCliente',
  'tipoInterviniente',
  'calificacion',
  'numIFIS',
  'numOperaciones',
  'ifisVencidas',
  'habilitadoCC',
  'puntaje',
  'cuotaEstimadaMensual',
  'saldoDeudaTotal',
  'cuotaEstimadaMensualTotalOI',
  'saldoDeudaTotalOI',
  'deudaPendiente',
  'cuotaParalelo',
];

// CA-01: detalles que sustentan los cálculos por sector.
const CLIENTE_BLOCK_PATHS = [
  'infoSectorReal.sectorReal',
  'infoSectorFinc.listSectorFincSBS.sectorFincSBS',
  'infoSectorFinc.listSectorFincSEPS.sectorFincSEPS',
  'creditosAprobadosSICOM',
  'creditosAprobadosSBS',
  'creditosAprobadosSEPS',
];

// CA-02: histórico de hasta 36 meses y análisis por sector.
const REPORTE_BLOCK_PATHS = [
  'infoPichinchaAcumulado.dtPichinchaXHistAcumulado',
  'analisisCreditos.sectorReal',
  'analisisCreditos.sectorSBS',
  'analisisCreditos.sectorSEPS',
];

function getRequestXml() {
  const fromEnvironment = insomnia.environment.get('dual_run_request_xml');
  if (typeof fromEnvironment === 'string' && fromEnvironment.trim()) {
    return insomnia.environment.replaceIn(fromEnvironment).trim();
  }

  const rawBody = insomnia.request && insomnia.request.body
    ? insomnia.request.body.raw
    : undefined;

  if (typeof rawBody === 'string' && rawBody.trim()) {
    return insomnia.environment.replaceIn(rawBody).trim();
  }

  throw new Error(
    'No se encontró el XML. Defina dual_run_request_xml en el ambiente de Insomnia ' +
    'o ejecute el script sobre una petición con body XML raw.'
  );
}

function detectOperation(xml) {
  if (/<(?:[A-Za-z_][\w.-]*:)?ConsultarClienteBuroCredito01(?:\s|>)/.test(xml)) {
    return 'ConsultarClienteBuroCredito01';
  }
  if (/<(?:[A-Za-z_][\w.-]*:)?ConsultarReporteBuroCredito01(?:\s|>)/.test(xml)) {
    return 'ConsultarReporteBuroCredito01';
  }
  throw new Error(
    'Operación SOAP no reconocida. Se esperaba ConsultarClienteBuroCredito01 ' +
    'o ConsultarReporteBuroCredito01.'
  );
}

function sendSoap(url, xml) {
  const soapAction = insomnia.environment.get('dual_run_soap_action');
  const header = {
    'Content-Type': 'text/xml; charset=UTF-8',
    Accept: 'text/xml, application/xml',
  };

  if (typeof soapAction === 'string' && soapAction.trim()) {
    header.SOAPAction = soapAction.trim();
  }

  const request = {
    url,
    method: 'POST',
    header,
    body: {
      mode: 'raw',
      raw: xml,
    },
  };

  return new Promise((resolve, reject) => {
    insomnia.sendRequest(request, (error, response) => {
      if (error) {
        reject(new Error(`Error invocando ${url}: ${error.message || error}`));
        return;
      }
      resolve(response);
    });
  });
}

function responseStatus(response) {
  return Number(response.code ?? response.status);
}

function responseBody(response) {
  if (typeof response.body === 'string') return response.body;
  if (response.body && typeof response.body.toString === 'function') {
    return response.body.toString('utf8');
  }
  if (typeof response.text === 'function') return response.text();
  throw new Error('Insomnia no devolvió el body de la respuesta como texto.');
}

function parseXml(xml) {
  return new Promise((resolve, reject) => {
    xml2js.parseString(
      xml,
      {
        explicitArray: false,
        explicitRoot: true,
        trim: true,
        normalize: false,
        emptyTag: '',
        tagNameProcessors: [xml2js.processors.stripPrefix],
      },
      (error, result) => (error ? reject(error) : resolve(result))
    );
  });
}

function getPath(object, path) {
  return path.split('.').reduce(
    (current, key) => (current == null ? undefined : current[key]),
    object
  );
}

function normalize(value) {
  if (value === undefined || value === null) return '';
  if (typeof value === 'string') return value.trim();
  if (Array.isArray(value)) return value.map(normalize);
  if (typeof value === 'object') {
    return Object.keys(value)
      .filter((key) => key !== '$')
      .sort()
      .reduce((result, key) => {
        result[key] = normalize(value[key]);
        return result;
      }, {});
  }
  return String(value);
}

function findResponseNode(parsed, operation) {
  const body = getPath(parsed, 'Envelope.Body');
  if (!body || typeof body !== 'object') {
    throw new Error('No se encontró Envelope.Body en la respuesta SOAP.');
  }

  const expected = `${operation}Response`;
  const responseNode = body[expected];
  if (!responseNode) {
    const fault = body.Fault;
    if (fault) {
      throw new Error(`La respuesta contiene SOAP Fault: ${JSON.stringify(normalize(fault))}`);
    }
    throw new Error(`No se encontró el nodo ${expected}.`);
  }
  return responseNode;
}

function registerEqualityTest(label, legacyValue, migratedValue) {
  testCase(label, () => {
    expectValue(normalize(migratedValue)).to.eql(normalize(legacyValue));
  });
}

function registerPresenceTest(label, value) {
  testCase(label, () => {
    expectValue(value, `${label}: campo inexistente`).to.not.equal(undefined);
    expectValue(value, `${label}: campo nulo`).to.not.equal(null);
  });
}

const requestXml = getRequestXml();
const operation = detectOperation(requestXml);

// Ejecución consecutiva intencional: primero legado, después migrado.
const legacyResponse = await sendSoap(ENDPOINTS.legado, requestXml);
const migratedResponse = await sendSoap(ENDPOINTS.migrado, requestXml);

const legacyStatus = responseStatus(legacyResponse);
const migratedStatus = responseStatus(migratedResponse);

testCase('[HTTP] Legado responde 2xx', () => {
  expectValue(legacyStatus).to.be.within(200, 299);
});

testCase('[HTTP] Migrado responde 2xx', () => {
  expectValue(migratedStatus).to.be.within(200, 299);
});

registerEqualityTest('[HTTP] Status Legado = Migrado', legacyStatus, migratedStatus);

const legacyParsed = await parseXml(responseBody(legacyResponse));
const migratedParsed = await parseXml(responseBody(migratedResponse));
const legacyOutput = findResponseNode(legacyParsed, operation);
const migratedOutput = findResponseNode(migratedParsed, operation);

// El bloque error siempre existe y siempre se valida campo por campo.
for (const field of ERROR_FIELDS) {
  const legacyValue = getPath(legacyOutput, `error.${field}`);
  const migratedValue = getPath(migratedOutput, `error.${field}`);
  registerPresenceTest(`[ERROR] Legado contiene error.${field}`, legacyValue);
  registerPresenceTest(`[ERROR] Migrado contiene error.${field}`, migratedValue);
  registerEqualityTest(`[ERROR] error.${field}: Legado = Migrado`, legacyValue, migratedValue);
}

const legacyBodyOut = legacyOutput.bodyOut;
const migratedBodyOut = migratedOutput.bodyOut;

// Si el backend devolvió un error, CA-06 exige propagar el objeto error y finalizar.
// Por ello no se comparan campos funcionales que podrían no existir.
const legacyErrorCode = normalize(getPath(legacyOutput, 'error.codigo'));
const migratedErrorCode = normalize(getPath(migratedOutput, 'error.codigo'));
const bothSuccessful = legacyErrorCode === '0' && migratedErrorCode === '0';

if (bothSuccessful) {
  if (operation === 'ConsultarClienteBuroCredito01') {
    for (const path of CLIENTE_SCALAR_PATHS) {
      registerEqualityTest(
        `[CA-01/CA-05] bodyOut.${path}: Legado = Migrado`,
        getPath(legacyBodyOut, path),
        getPath(migratedBodyOut, path)
      );
    }

    for (const path of CLIENTE_BLOCK_PATHS) {
      registerEqualityTest(
        `[CA-01] bodyOut.${path}: Legado = Migrado`,
        getPath(legacyBodyOut, path),
        getPath(migratedBodyOut, path)
      );
    }

    testCase('[CA-05] calificacion usa una etiqueta permitida', () => {
      expectValue(normalize(getPath(migratedBodyOut, 'calificacion'))).to.be.oneOf([
        'AAA',
        'AA',
        'A',
        'Rechazado',
        'Sin Informacion',
        'Revision Manual',
      ]);
    });
  }

  if (operation === 'ConsultarReporteBuroCredito01') {
    for (const path of REPORTE_BLOCK_PATHS) {
      registerEqualityTest(
        `[CA-02] bodyOut.${path}: Legado = Migrado`,
        getPath(legacyBodyOut, path),
        getPath(migratedBodyOut, path)
      );
    }

    const migratedHistory = getPath(
      migratedBodyOut,
      'infoPichinchaAcumulado.dtPichinchaXHistAcumulado'
    );
    const historyRows = Array.isArray(migratedHistory)
      ? migratedHistory
      : migratedHistory === undefined || migratedHistory === ''
        ? []
        : [migratedHistory];

    testCase('[CA-02] histórico migrado contiene máximo 36 meses', () => {
      expectValue(historyRows.length).to.be.at.most(36);
    });
  }
}

console.log(JSON.stringify({
  operacion: operation,
  httpLegado: legacyStatus,
  httpMigrado: migratedStatus,
  codigoLegado: legacyErrorCode,
  codigoMigrado: migratedErrorCode,
}, null, 2));
```

## 3. Alcance de comparación

No se compara `headerOut` ni `fechaConsulta`, porque contienen información transaccional o temporal que puede variar entre llamadas consecutivas y no están definidos como equivalencia funcional en los criterios de aceptación.

En una respuesta exitosa de `ConsultarClienteBuroCredito01`, se comparan:

- identificación, tipo transformado, cliente, interviniente y clasificación;
- número de IFIS, operaciones e IFIS vencidas;
- puntaje, habilitación, cuotas y saldos de deuda;
- detalle del sector Real, SBS y SEPS;
- créditos aprobados y moras contenidos en los bloques SICOM, SBS y SEPS;
- todos los campos del bloque `error`.

En una respuesta exitosa de `ConsultarReporteBuroCredito01`, se comparan:

- histórico consolidado;
- análisis de créditos de los sectores Real, SBS y SEPS;
- todos los campos del bloque `error`.

Cuando el código de error es distinto de `0`, el script compara integralmente el bloque `error` y no exige `bodyOut`, conforme al criterio de propagación del error del backend.

> Nota importante: el criterio proporcionado exige igualdad exacta también para `error.recurso`, `error.componente` y `error.backend`. Si la arquitectura aprobó que alguno cambie por el nombre técnico del servicio migrado, esa excepción debe documentarse y retirarse explícitamente de `ERROR_FIELDS`; no conviene ocultarla sin aprobación.

## 4. Matriz de casos de prueba para JIRA

| ID | CA | Operación | Tipo | Precondición / datos | Acción | Resultado esperado | Automatización |
|---|---|---|---|---|---|---|---|
| EP-001 | CA-01 | ConsultarClienteBuroCredito01 | Nominal | `tipoIdentificacion=C`; identificación válida | Ejecutar Dual Run | Ambos servicios responden HTTP 2xx; `error.codigo=0`; `error.mensaje=OK`; tipo transformado a `CEDULA`; campos funcionales iguales | Sí, Insomnia |
| EP-002 | CA-03 | ConsultarClienteBuroCredito01 | Negativo | `identificacion=""` | Ejecutar Dual Run | Ambos propagan `codigo=1` y mensaje `Valor del campo identificacion vacio o invalido`; bloque error idéntico | Sí, Insomnia |
| EP-003 | CA-03 | ConsultarClienteBuroCredito01 | Negativo | Omitir `identificacion` | Ejecutar Dual Run | Ambos retornan código y mensaje de obligatoriedad acordados; bloque error idéntico | Sí, Insomnia |
| EP-004 | CA-03 | ConsultarClienteBuroCredito01 | Negativo | `tipoIdentificacion=""` | Ejecutar Dual Run | Ambos propagan `codigo=2` y mensaje `Valor del campo tipoIdentificacion vacio o invalido`; bloque error idéntico | Sí, Insomnia |
| EP-005 | CA-03 | ConsultarClienteBuroCredito01 | Negativo | Omitir `tipoIdentificacion` | Ejecutar Dual Run | Ambos retornan código y mensaje de obligatoriedad acordados; bloque error idéntico | Sí, Insomnia |
| EP-006 | CA-01 | ConsultarClienteBuroCredito01 | Alterno | Probar cada tipo de identificación permitido con dato válido | Ejecutar Dual Run por cada tipo | Tipo transformado y respuesta funcional iguales en ambos servicios | Sí, data driven |
| EP-007 | CA-01 | ConsultarClienteBuroCredito01 | Nominal | Cliente con información en sector Real, SBS y SEPS | Ejecutar Dual Run | `numIFIS`, `numOperaciones`, `ifisVencidas`, cuotas, deudas y detalles sectoriales iguales | Sí, Insomnia |
| EP-008 | CA-01 | ConsultarClienteBuroCredito01 | Alterno | Cliente sin IFIS o sin deuda | Ejecutar Dual Run | Ceros/vacíos representados de la misma forma y bloque error exitoso | Sí, Insomnia |
| EP-009 | CA-05 | ConsultarClienteBuroCredito01 | Paramétrica | Backend devuelve clasificación `1` a `6`, un caso por valor | Ejecutar Dual Run | Ambos mapean respectivamente a la etiqueta configurada; valor pertenece al catálogo AAA, AA, A, Rechazado, Sin Informacion o Revision Manual | Sí, con datos controlados |
| EP-010 | CA-02 | ConsultarReporteBuroCredito01 | Nominal | `tipoSistema=TODOS`; `tipoRiesgo=T`; `tipoCredito=TODOS`; identificación válida | Ejecutar Dual Run con XML de la operación Reporte | Histórico consolidado y análisis Real/SBS/SEPS iguales; máximo 36 registros mensuales | Sí, Insomnia |
| EP-011 | CA-02 | ConsultarReporteBuroCredito01 | Alterno | Filtros que solo devuelven sector Real | Ejecutar Dual Run | Histórico y análisis del sector Real iguales; sectores sin datos equivalentes | Sí, Insomnia |
| EP-012 | CA-02 | ConsultarReporteBuroCredito01 | Alterno | Filtros que devuelven SBS y/o SEPS | Ejecutar Dual Run | Mayor crédito, reciente, antiguo y moras iguales por sector | Sí, Insomnia |
| EP-013 | CA-04 | ConsultarReporteBuroCredito01 | Negativo | `tipoSistema=""` u omitido | Ejecutar Dual Run | Ambos retornan el código 1–5 que corresponda al contrato y el mismo bloque error | Sí, Insomnia |
| EP-014 | CA-04 | ConsultarReporteBuroCredito01 | Negativo | `tipoRiesgo=""` u omitido | Ejecutar Dual Run | Ambos retornan el código 1–5 que corresponda al contrato y el mismo bloque error | Sí, Insomnia |
| EP-015 | CA-04 | ConsultarReporteBuroCredito01 | Negativo | `tipoCredito=""` u omitido | Ejecutar Dual Run | Ambos retornan el código 1–5 que corresponda al contrato y el mismo bloque error | Sí, Insomnia |
| EP-016 | CA-04 | ConsultarReporteBuroCredito01 | Negativo | Combinación de varios campos obligatorios vacíos | Ejecutar Dual Run | Ambos aplican la misma prioridad de validación y retornan el mismo primer error | Sí, Insomnia |
| EP-017 | CA-06 | Ambas | Error negocio/backend | Simular UMPClientes0068 con `codigo != 0` | Ejecutar Dual Run | El objeto `error` se propaga íntegramente e idéntico; no continúa el procesamiento funcional | Sí, requiere stub/dato controlado |
| EP-018 | CA-06 | Ambas | Error técnico | Backend no disponible o timeout controlado | Ejecutar petición | Manejo técnico conforme al contrato; sin respuesta parcial inconsistente; logs correlacionables | Parcial/manual |
| EP-019 | CA-07 | Ambas | Observabilidad | Acceso al pod OCP4 y GUID/unicidad conocidos | Ejecutar transacción | Logs incluyen trama de entrada, pasos y trama de salida, con correlación y sin exposición indebida de datos sensibles | Manual/OCP4 |
| EP-020 | CA-08 | Ambas | Observabilidad | Servicio con tráfico en ambiente monitoreado | Ejecutar transacción y consultar Dynatrace | Servicio/traza visible; latencia, disponibilidad y errores registrados | Manual/Dynatrace |
| EP-021 | CA-09 | Ambas | Rendimiento | Escenario de carga aprobado y datos reutilizables | Ejecutar prueba de carga | p95 menor a 800 ms y tasa HTTP 5xx menor o igual a 3%, sujetos a confirmación de Arquitectura | No con Dual Run; usar k6/JMeter |
| EP-022 | CA-10 | N/A | Documentación | Desarrollo terminado | Revisar repositorio documental | Diagrama ICE Panel y documento técnico actualizados, aprobados y trazables a la versión desplegada | Manual |
| EP-023 | CA-01/02/06 | Ambas | Contrato SOAP | Respuesta con namespaces/prefijos XML distintos pero misma estructura | Ejecutar Dual Run | El parser ignora únicamente el prefijo; los valores funcionales siguen siendo iguales | Sí, Insomnia |
| EP-024 | CA-01/02 | Ambas | Regresión | Mismo request repetido con igual dato | Ejecutar varias iteraciones | No hay diferencias funcionales entre legado y migrado; se excluyen únicamente campos temporales no contemplados | Sí, Collection Runner |

## 5. Datos pendientes antes del cierre

- Identificación válida y autorizada para EP-001.
- Contrato exacto de códigos `1` a `5` de `ConsultarReporteBuroCredito01`, porque el criterio recibido no asigna cada número a un campo concreto.
- Tabla oficial de correspondencia clasificación `1..6` → etiqueta; el criterio enumera las etiquetas, pero no especifica inequívocamente qué número corresponde a cada una.
- Confirmación arquitectónica de p95 `< 800 ms` y tasa HTTP 5xx `<= 3%`.
- Confirmación de si `error.recurso`, `error.componente` y `error.backend` deben ser idénticos o constituyen excepciones técnicas aprobadas en la migración.

## Referencia técnica

La implementación utiliza las capacidades documentadas por Kong para Insomnia: envío de peticiones con `insomnia.sendRequest`, pruebas con `insomnia.test`/`insomnia.expect` y la librería integrada `xml2js`: https://developer.konghq.com/insomnia/scripts/
