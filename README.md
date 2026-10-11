# SnackCheck - Veredicto Nutricional

Documentación complementaria: [registro de pruebas](TESTING.md) y [changelog](CHANGELOG.md).

## Propósito

Workflow de n8n que recibe un código de barras, consulta Open Food Facts y devuelve un resumen nutricional en español. Calcula los semáforos y el veredicto de forma determinista; Groq solo redacta el mensaje para la persona usuaria.

## Cómo Funciona

1. **01 - Webhook SnackCheck** recibe una petición GET en la ruta `snackcheck`.
2. **02 - ¿Barcode válido?** comprueba que `query.barcode` contenga solo dígitos. Si falta o no es válido, **02a - Respuesta de entrada inválida** indica el problema, cómo corregirlo y un ejemplo de consulta.
3. **03 - Open Food Facts** consulta el producto y conserva el estado HTTP junto con el cuerpo de respuesta. Los errores de transporte siguen hacia **03a - Error de consulta**, que devuelve HTTP 502, explica el fallo de conexión o timeout y propone reintentar sin cambiar el código de barras.
4. **04 - ¿Producto encontrado?** exige HTTP 200 y `body.status === 1`. Si no se cumple, **04a - Producto no encontrado** distingue un producto ausente (HTTP de la API 200 o 404 con `body.status` igual a 0) de un fallo o respuesta no válida de la API. Devuelve HTTP 404 con `PRODUCT_NOT_FOUND` en el primer caso y HTTP 502 con `PRODUCT_LOOKUP_UNAVAILABLE` en el segundo.
5. **05 - Calcular veredicto y semáforos** normaliza los nutrientes y calcula las lecturas con los umbrales configurados:
   - Azúcar: bajo `< 5 g`, medio `5–22.5 g`, alto `> 22.5 g` por 100 g.
   - Sal: alto `> 1.5 g` por 100 g; en otro caso, bajo.
   - Grasa: alto `> 17.5 g` por 100 g; en otro caso, bajo.
   - El conteo de nutrientes altos produce saludable (0), moderado (1) o no saludable (2–3). Nutri-Score a/b, c y d/e corresponden a esos mismos niveles; cuando existen ambas lecturas, prevalece la peor.
6. **06 - ¿Datos suficientes?** requiere al menos una medición disponible y un Nutri-Score válido o las tres mediciones completas. Si no se cumple, **06a - Datos insuficientes** devuelve HTTP 422 con el nombre del producto y propone consultar otro producto con datos completos o completar la ficha en Open Food Facts.
7. **07 - Redactar explicación** usa los datos ya calculados para redactar el texto. **Groq Chat Model** proporciona el modelo de lenguaje. Si falla la generación, la salida de error llega al nuevo nodo **07a - Error de explicación**, que devuelve HTTP 502 con `EXPLANATION_UNAVAILABLE`, un ejemplo para reintentar y la recomendación de contactar con el responsable si persiste; no se ejecuta el nodo 08.
8. **08 - Respuesta SnackCheck** devuelve el JSON final con los valores calculados y el mensaje generado.

Las notas visibles de cada nodo del workflow explican también su responsabilidad y su ruta de salida.

## Configuración

1. Importa `SnackCheck-Veredicto Nutricional(1).json` en n8n.
2. Comprueba que el Webhook use el método GET, la ruta `snackcheck` y la opción de responder mediante nodos **Respond to Webhook**.
3. Abre **Groq Chat Model**, selecciona una credencial Groq válida y confirma que el modelo `openai/gpt-oss-20b` está disponible en tu cuenta. La API key se guarda en las credenciales de n8n, nunca en el prompt o en este archivo.
4. Open Food Facts no requiere credencial. El HTTP Request tiene un timeout de 10 segundos y conserva el status HTTP y el cuerpo para distinguir productos inexistentes de fallos de la API. Comprueba que `Never Error` esté activado y que la salida de error del nodo 03 esté conectada al nodo 03a mediante `Continue (using error output)`.
5. El Webhook está configurado sin autenticación. Añade una protección adecuada antes de exponerlo públicamente.
6. Comprueba que el nodo 07 también use `Continue (using error output)`: su salida normal debe conectar con el nodo 08 y su salida de error con el nodo 07a, de tipo **Respond to Webhook**.

## Uso

### Prueba

1. Abre el workflow en n8n y guarda los cambios.
2. Selecciona **Listen for test event** en el Webhook.
3. Copia la Test URL mostrada por n8n y envía una petición GET. Puedes introducir el barcode en la pestaña **Params** de Hoppscotch (`barcode` = `3017620422003`) o añadirlo a la URL:

```text
https://<tu-instancia-n8n>/webhook-test/snackcheck?barcode=3017620422003
```

No envíes el código como cuerpo JSON: este workflow lo lee desde los parámetros de consulta.

Pruebas útiles:

- Sin `barcode`: comprueba `BARCODE_REQUIRED`.
- `?barcode=abc`: comprueba `INVALID_BARCODE`.
- `?barcode=3017620422003`: prueba el producto Nutella indicado en el brief.
- `?barcode=6111035000430`: comprueba un caso con Nutri-Score A; los datos comunitarios pueden cambiar.
- Un barcode numérico no listado: comprueba la ruta `PRODUCT_NOT_FOUND`.

Revisa **Executions** en n8n para ver qué nodos se ejecutaron y los valores de entrada/salida. En el nodo 07, inspecciona la vista previa del prompt para confirmar que las expresiones muestran valores concretos.

### Producción

Después de completar las pruebas, activa el workflow y usa la Production URL que n8n muestra para el Webhook, normalmente con la ruta `/webhook/snackcheck` en lugar de `/webhook-test/snackcheck`.

La respuesta satisfactoria tiene `ok`, `barcode`, `product`, `headline`, `verdict`, `nutriscore`, `concernScore`, `nutrientsPer100g`, `message` y `source`.

## Manejo de Errores

| Situación | HTTP | Código | Comportamiento |
|---|---:|---|---|
| Falta el barcode | 200 | `BARCODE_REQUIRED` | El nodo 02a pide enviar un código numérico en el parámetro `barcode` e incluye un ejemplo. |
| Barcode con caracteres no numéricos | 400 | `INVALID_BARCODE` | El nodo 02a pide solo números, sin letras, espacios ni guiones, e incluye un ejemplo. |
| Fallo de red o timeout al consultar Open Food Facts | 502 | `PRODUCT_LOOKUP_UNAVAILABLE` | El nodo 03a explica el fallo y propone reintentar sin cambiar el código; no expone detalles internos. |
| Producto no registrado: HTTP de la API 200 o 404 con `body.status` igual a 0 | 404 | `PRODUCT_NOT_FOUND` | El nodo 04a pide comprobar los números de la etiqueta o consultar un producto registrado, con un ejemplo. |
| Error HTTP o respuesta no válida de Open Food Facts | 502 | `PRODUCT_LOOKUP_UNAVAILABLE` | El nodo 04a aclara que no puede confirmar si el producto existe y propone reintentar con un ejemplo. |
| Datos nutricionales insuficientes | 422 | `INSUFFICIENT_NUTRITION_DATA` | El nodo 06a informa que no puede valorar el producto y propone consultar otro o completar su ficha, con un ejemplo. |
| Fallo al generar la explicación con IA | 502 | `EXPLANATION_UNAVAILABLE` | El nodo 07a propone reintentar sin cambiar el código y contactar con el responsable si persiste; no expone credenciales ni detalles internos. |
| Producto procesado correctamente | 200 | `ok: true` | Devuelve datos, clasificación y mensaje. |

Todas las respuestas de error incluyen `ok: false`, `code` y `message`. El mensaje especifica la causa, la acción recomendada y un ejemplo de consulta: `?barcode=3017620422003`. El ejemplo muestra el formato de entrada; la disponibilidad y los datos del producto dependen de Open Food Facts.

Los campos nutricionales ausentes se conservan como `null` y se muestran como `sin_dato`; no se tratan como cero.

## Limitaciones

- Open Food Facts es una base de datos colaborativa: la disponibilidad y exactitud de cada ficha pueden variar.
- Los umbrales implementan las reglas del proyecto y no sustituyen orientación médica o nutricional personalizada.
- El flujo solo obtiene y evalúa azúcar, sal, grasa, energía y Nutri-Score. No debe pedirse a la IA que informe fibra, proteínas u otros nutrientes que la API no haya proporcionado al workflow.
- El barcode debe enviarse como parámetro de consulta y contener solo dígitos; el flujo no procesa un barcode enviado en el cuerpo de la petición.
- El modelo puede variar la redacción entre ejecuciones. Los campos numéricos y el veredicto calculado por el nodo 05 son la fuente de verdad.
- El endpoint no tiene autenticación configurada. No lo publiques sin añadir controles de acceso apropiados.
