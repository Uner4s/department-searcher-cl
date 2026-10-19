# Buscador de departamentos (Chile)

Recorre cinco portales inmobiliarios chilenos a la vez, descarta lo que no cumple tus criterios, fusiona el mismo departamento publicado en varios portales, y te deja un Excel ordenado por precio por metro cuadrado.

Portales: **Portal Inmobiliario, Toctoc, Chilepropiedades, Houm y Goplaceit**.

No es un programa que instalas. Es un conjunto de instrucciones que le das a Claude para que navegue los portales por ti.

## Qué necesitas

| | |
| --- | --- |
| **Plan de pago de Claude** | Pro, Max, Team o Enterprise. Con la cuenta gratis no funciona. |
| **Claude in Chrome** | La extensión que le permite a Claude navegar. [Instalarla desde la Chrome Web Store](https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn). |

⚠️ **Esto es lo importante**: no sirve pegarle estos archivos a Claude en claude.ai y pedirle que busque. Claude necesita poder **abrir los portales en tu navegador** — son sitios que cargan los resultados con JavaScript y no se pueden leer de otra forma. Sin la extensión (o sin el navegador integrado de Cowork), el agente no tiene cómo hacer el trabajo.

## Cómo correrlo

**1. Abre [`department-searcher-agent.md`](department-searcher-agent.md) y edita el bloque de Parámetros.**

Ahí defines comuna, presupuesto y qué tipo de departamento buscas. Viene configurado para Las Condes, 1 y 2 dormitorios, hasta 4.500 y 6.700 UF. Para cambiar de comuna, el mismo documento explica de dónde sacar los tres slugs.

**2. Abre un chat nuevo en Chrome, con la extensión conectada.**

Confirma que Claude tenga permiso sobre los cinco dominios. Si alguno queda bloqueado, el agente sigue con los demás y te lo reporta al final.

**3. Adjunta el archivo al chat y dile:**

> Lee la Instrucción del agente de este archivo y ejecútala usando sus Parámetros.

Dile explícitamente "la Instrucción del agente": el archivo trae secciones que son para ti y no conviene que las mezcle con lo que debe ejecutar.

Si vas a correrlo seguido, el documento explica también cómo hacerlo desde **Cowork** apuntando a esta carpeta: no copias ni adjuntas nada y el Excel queda guardado acá mismo.

Eso es todo. El agente recorre los portales, abre las fichas de los que pasan el filtro, y arma el Excel.

## Antes de la primera corrida, achícala

Una corrida completa son 5 portales × 2 tipologías × hasta 5 páginas, más abrir la ficha de cada candidato. Es **pesada**: consume bastante de tu límite de uso y toma un rato largo.

Para la primera vez, en el bloque de Parámetros pon `PAGINAS_MAX = 1` y deja una sola tipología. Así ves cómo se comporta y cuánto gasta antes de soltarla entera. Después la subes.

No hay contador de tokens por corrida, pero puedes mirar **Configuración → Uso** en claude.ai antes y después: la diferencia te dice cuánto te costó.

## Qué recibes

Un Excel con cuatro hojas:

- **Resumen** — parámetros usados, conteos, valor de la UF del día.
- **Avisos** — una fila por departamento, con los duplicados entre portales ya fusionados, ordenado por UF/m².
- **Fuentes** — una fila por aviso-portal, con el link de cada publicación.
- **Proyectos** — los avisos de "Desde UF X" que no permiten calcular precio por m².

En la corrida de referencia (Las Condes, 19-10-2026) salieron 250 avisos que se fusionaron en 159 departamentos.

## Dos advertencias que te van a ahorrar tiempo

**Los filtros de los portales mienten.** Tres de los cinco fallan en algo. El peor es Portal Inmobiliario: su filtro de estacionamiento dejó pasar avisos sin estacionamiento en 1 de cada 4 casos. Por eso el agente abre cada ficha a verificar, y por eso la columna `Fuente estacionamiento` te dice de dónde sacó el dato. Si dice `Ficha`, confía. Si dice `Ficha sin dato`, anda a mirar tú.

**Un aviso con estado `Califica (sin verificar)` no es un candidato.** Significa que pasó los filtros pero nadie confirmó el estacionamiento en la ficha. Revísalo antes de emocionarte.

## Si algo se rompe

Los portales cambian sus páginas cada cierto tiempo. Si el agente devuelve cero resultados en un portal que antes funcionaba, lo más probable es que haya cambiado la URL de búsqueda.

[`docs/portales.md`](docs/portales.md) tiene la URL y el comportamiento de cada portal, documentados con fecha. Compara contra lo que ves en el navegador y actualiza lo que haya cambiado.

## Los demás archivos

| Documento | Para qué |
| --- | --- |
| [`department-searcher-agent.md`](department-searcher-agent.md) | Parámetros e instrucción. **Es el único que necesitas para correrlo.** |
| [`docs/portales.md`](docs/portales.md) | Cómo funciona cada portal y qué filtros no son confiables. Útil si algo se rompe, o si quieres escribir un scraper. |
| [`docs/esquema-salida.md`](docs/esquema-salida.md) | Las columnas del Excel y qué significa cada estado. |
| [`validacion/`](validacion/) | Corridas anteriores, para comparar cobertura entre una corrida y otra. |
