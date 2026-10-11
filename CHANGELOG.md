# Changelog

Los cambios notables de SnackCheck se registran aquí. Se sigue SemVer: `MAJOR.MINOR.PATCH`.

- **MAJOR**: cambios incompatibles en el contrato HTTP o el formato de respuesta.
- **MINOR**: capacidades nuevas compatibles con clientes existentes.
- **PATCH**: correcciones compatibles y mejoras de documentación.

Mientras no haya una versión publicada, los cambios se anotan bajo `Unreleased`. Al publicar, moverlos a una sección con versión SemVer y fecha; no asignar una versión de lanzamiento hasta que se haya decidido publicar.

## [Unreleased]

### Added

- Registro de pruebas `TC-XXX` con casos funcionales, de integración en vivo, de error y una nota de rendimiento.
- Documentación en línea para los nodos del workflow y guía de uso en `README.md`.
- Nodo 07a conectado a la salida de error del nodo 07 para responder con HTTP 502 y `EXPLANATION_UNAVAILABLE` cuando falla la generación de la explicación.

### Changed

- El prompt del nodo 07 recibe los datos calculados por el workflow y limita su explicación a los nutrientes disponibles.
- Las respuestas de error incluyen la causa, instrucciones para corregir o reintentar la consulta y un ejemplo de código de barras, según el caso: código ausente o inválido, producto no encontrado, datos nutricionales insuficientes y fallos de consulta o IA.

### Fixed

- Desactivada la configuración de parser de salida del nodo 07 al no existir un nodo parser conectado.
- El nodo 04a distingue un producto no encontrado (HTTP 404 con `PRODUCT_NOT_FOUND`) de un error o respuesta no válida de Open Food Facts (HTTP 502 con `PRODUCT_LOOKUP_UNAVAILABLE`), evitando indicar que el producto no existe cuando falla la API.
