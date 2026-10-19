# Esquema de salida

Define qué entrega una corrida, sin importar quién la haga. El agente y el scraper apuntan al mismo esquema: así la salida de uno se puede comparar contra la del otro fila por fila.

Son cuatro tablas. `Avisos` y `Fuentes` tienen relación 1:N — un departamento puede estar publicado en varios portales, y eso es justamente lo que hay que poder medir.

## Avisos

Una fila por **departamento**, con los duplicados entre portales ya fusionados.

| Columna | Valores | Notas |
| --- | --- | --- |
| `ID` | `{TIPOLOGIA}-{NNN}` | Ej. `1D1B-001`. Estable dentro de una corrida |
| `Tipología` | `1D1B`, `2D2B` | Según el bloque de Parámetros |
| `Estado` | `Califica`, `Califica (sin verificar)`, `Descartado` | Ver abajo |
| `Destacado` | `★` o vacío | Calculado, no escrito a mano |
| `UF/m²` | número | Calculado: `Precio (UF) / m² útiles` |
| `Precio (UF)` | número | Si venía en pesos, ya convertido |
| `Moneda publicada` | `UF`, `Pesos (convertido)` | Para saber si hubo conversión |
| `m² útiles` | número | **Nunca** totales. Si solo hay totales, vacío + alerta |
| `Estacionamiento` | `Sí`, `No`, `No informado` | |
| `Fuente estacionamiento` | `Ficha`, `Descripción`, `Título`, `Ficha sin dato` | **Nunca** el filtro del portal: no es confiable |
| `Piscina` | `Sí`, `No`, `No informado` | Solo informativa |
| `Barrio / dirección` | texto | |
| `Actualizado el` | fecha o vacío | Crítico en Goplaceit: hay fichas de 2023 |
| `Motivo descarte` | texto | Obligatorio si `Estado = Descartado`, vacío si no |
| `Alertas` | texto | Discrepancias entre portales, datos raros, republicaciones |
| `Portales` | lista | Dónde aparece |
| `N° avisos` | número | Cuántas publicaciones se fusionaron en esta fila |
| `Link principal` | URL | |
| `Otros links` | URLs | |

### Estados

- **`Califica`** — pasó todas las reglas **y** el estacionamiento se verificó en la ficha o la descripción.
- **`Califica (sin verificar)`** — pasó las reglas, pero el estacionamiento viene solo del filtro del portal. Dado que ese filtro falló en ~25% de los casos en Portal Inmobiliario, estas filas **no son candidatos**: son pendientes de revisar.
- **`Descartado`** — falló alguna regla. `Motivo descarte` dice cuál.

### Destacado

Se marca `★` si el estado es exactamente `Califica` (verificado) y el UF/m² es menor o igual al ideal:

```excel
=IF(AND(C2="Califica", E2<=Resumen!$B$7), "★", "")
```

Comparación exacta, no `LEFT(C2,8)="Califica"`: esa forma matchea también `Califica (sin verificar)` y marcaría como destacados avisos sin verificar en cuanto se suba el umbral ideal.

## Fuentes

Una fila por **aviso-portal**. Es la tabla que permite medir cobertura y duplicados, y es la línea base para comparar contra el scraper.

| Columna | Notas |
| --- | --- |
| `ID unidad` | Apunta a `Avisos.ID` |
| `Tipología` | |
| `Estado` | Copia del estado de la unidad |
| `Portal` | |
| `ID en el portal` | `MLC-1920398055`, `b_031931d0...`, `117723303`… |
| `URL` | Canónica, **sin query params de tracking** |

El `ID en el portal` es la clave de comparación entre corridas. Las URLs crudas no sirven: Portal Inmobiliario les cuelga `#position=...&tracking_id=...` y producen diferencias falsas.

## Proyectos

Los avisos con "Desde UF X" y rango de m², que no permiten calcular UF/m². Van aparte, nunca mezclados con `Avisos`.

| Columna |
| --- |
| `Tipología`, `Proyecto`, `Desde (UF)`, `Rango m²`, `Dirección`, `Estado`, `Link` |

## Resumen

Parámetros usados, conteos y cómo leer el archivo.

- **Parámetros**: valor UF (con fuente y fecha), topes de UF/m², precios máximos por tipología. El UF/m² ideal vive en una celda (`B7`) de la que dependen las fórmulas de `Destacado`.
- **Conteos** por tipología y estado, con `COUNTIFS` sobre `Avisos` — no escritos a mano, así siguen siendo ciertos si se edita una fila.
- **Portales sin acceso**: cuáles fallaron y por qué (permiso, captcha, login, caída). Si no falló ninguno, decirlo explícitamente. Una corrida sin esta sección no se puede distinguir de una corrida con un portal caído en silencio.
- **Fecha de la corrida** y quién la hizo (agente o scraper, y versión).

## Formato de archivo

Excel (`.xlsx`), una hoja por tabla. Los valores calculados van como **fórmulas**, no como números pegados: así el archivo sigue siendo correcto si se ajusta un parámetro.

Para comparar dos corridas, exportar `Fuentes` a CSV y diffear por `ID en el portal`.
