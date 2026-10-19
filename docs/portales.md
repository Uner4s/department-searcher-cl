# Reconocimiento de portales

Cómo extraer datos de cada portal, qué filtros mienten y qué basura hay que limpiar. Levantado en la corrida del **19-10-2026** (agente con Claude in Chrome, 5 portales, 1D1B y 2D2B).

Este archivo es especificación, no resultado: se actualiza cuando un portal cambia, no en cada búsqueda. Los resultados de cada corrida van aparte.

## Resumen por portal

| Portal | Técnica de extracción | Filtros confiables | Vale la pena |
| --- | --- | --- | --- |
| Portal Inmobiliario | HTTP plano + JSON embebido en la ficha | Precio, dormitorios. **Estacionamiento y baños no** | Sí — 128 de 250 avisos de la corrida |
| Toctoc | Navegador real (props de React) | Precio, dormitorios, baños. **m² no** | Sí — 56 avisos, y da el valor UF del día |
| Chilepropiedades | HTTP plano, HTML de servidor | Precio. Dormitorios y baños son mínimos | Sí — el más simple de los cinco |
| Goplaceit | API JSON pública | Los de la API, pero busca por mapa, no por comuna | Sí — 47 avisos, API estable |
| Houm | HTTP plano | Rellena con comunas vecinas sin avisar | **No** — aportó 1 aviso, descartado |

## Portal Inmobiliario

```
/venta/departamento/{1-dormitorio|2-dormitorios}/las-condes-metropolitana/_PriceRange_0CLF-{MAX}CLF_Cocheras_1-*_Banos_{B}
```

**Paginación.** Insertar `_Desde_{N}` antes de `_PriceRange`, con N = 49, 97, 145, 193… (48 avisos por página).

**Listado.** Cada tarjeta es `li.ui-search-layout__item`. Del `innerText` salen precio (UF o $), dormitorios, baños, "X m² útiles" y dirección. La URL limpia es `origin + pathname` del primer `<a>`: los query params traen tracking y rompen cualquier comparación por URL.

**Ficha, rápido.** Desde una página del mismo dominio, `fetch('/MLC-...')` devuelve HTML con JSON embebido. Con dos regex:

```
"id":"Estacionamientos","text":"(\d+)"
Piscina","text":"(Sí|No)"
```

Permite revisar ~60 fichas en segundos sin abrir pestañas. Es la diferencia entre una corrida de minutos y una de horas.

**⚠️ El filtro de estacionamiento miente.** `Cocheras_1-*` dejó pasar avisos sin estacionamiento: 4 de 9 en 1D1B y 16 de 63 destacados en 2D2B tenían `0` en la ficha o "sin estacionamiento" en la descripción. Como el estacionamiento es criterio obligatorio, **hay que verificar la ficha siempre**. El filtro de baños también deja colar avisos que no calzan.

**Datos sucios.** Proyectos (`PROYECTO`, "Desde UF") mezclados con avisos normales; precios en pesos; cesiones de promesa; propiedades en Brasil publicadas con dirección de Las Condes; títulos "VENDIDO" todavía activos; m² absurdos (2.011 m² en un 1D). Además el mismo departamento aparece publicado dos veces con distinto MLC, por corredores distintos o republicación.

## Toctoc

```
/venta/departamento/metropolitana/las-condes?moneda=UF&precios=0,{MAX}&dormitorios={D},{D}&bannos={B},{B}
```

**Paginación.** `&pagina={N}`, 20 avisos por página. Render en cliente: hay que esperar 5-7 s.

**Listado.** Las tarjetas no tienen `<a>`. Los datos (`titulo`, `urlFicha`, `precios`, `superficie`, `dormitorios`, `bannos`) están en las props de React: desde el elemento "Ver más", subir por `__reactFiber*.return` hasta encontrar un objeto con `urlFicha`. `__NEXT_DATA__` trae resultados **sin filtrar**, no sirve.

Esto obliga a navegador real para Toctoc. No hay forma de sacarlo con HTTP plano.

**Ficha.** También render en cliente: hay que navegar y esperar, el `fetch` del HTML no trae los datos. Campos: "Superf. útil", "Estacionamiento: N", lista de servicios (Piscina).

**⚠️ El filtro de superficie se ignora.** `superficie-util` en la URL no hace nada: con mínimo 45 aparecen avisos de 23 m². Filtrar a mano.

**⚠️ m² útiles inflados.** A veces suma la terraza (50 m² donde otros portales dicen 43). Ante diferencias entre portales, usar el menor.

**Valor UF.** Cada tarjeta muestra precio en UF y en pesos: `UF del día = pesos / UF`. Es la fuente más cómoda para convertir los avisos publicados en pesos.

**Datos sucios.** Avisos con 0 m²; en proyectos, dormitorios y baños vienen como rango (`['1','3']`).

## Chilepropiedades

```
/propiedades/venta/departamento/las-condes/{PAGINA_0_INDEXADA}?precioUnidad=3&precioMax={MAX}&habitaciones={D}&banos={B}&areaMin={M2}&areaUnidad=1&avanzada=true
```

**Extracción.** HTML de servidor, el más simple de los cinco. Cada tarjeta tiene un "Ver detalles" (`<a>` con URL limpia). El orden de líneas es: precio, título, dormitorios, baños, m², estacionamientos (si los informa). 10 avisos por página.

**Cobertura.** La página declara "Total de páginas: N". Sirve como invariante: lo extraído tiene que cuadrar con lo declarado.

**⚠️ Filtros como mínimo.** `habitaciones` y `banos` significan "1 o más": hay que descartar a mano los que tengan más. El m² de la tarjeta no dice si es útil o total; la ficha sí ("Superficie Útil").

## Goplaceit

Tiene API pública, que es mucho más estable que el mapa (el mapa con scroll de lista colgó el renderer una vez):

```
GET https://api.goplaceit.com/api/v4/search/list
  ?country=cl&platform=web
  &s[tipo_propiedad][]=2&s[id_modalidad]=1&s[estacionamientos]=true
  &s[dormitorio_desde]=..&s[dormitorio_hasta]=..
  &s[bano_desde]=..&s[bano_hasta]=..
  &s[area_desde]=..&s[precio_venta_hasta]=..
  &s[in_viewport]=latS,lngW|latN,lngE
  &page=1&per_page=300
```

La respuesta viene en **arrays paralelos**: `ids`, `mts`, `prices` (en miles de CLP), `lat_lng`.

**Ficha.** `/cl/propiedad/venta/departamento/{cualquier-texto}/{ID}` redirige a la URL canónica, que incluye comuna y calle en el slug. Render en cliente.

**⚠️ Busca por área del mapa, no por comuna.** `in_viewport` es un rectángulo: con el zoom por defecto entran Providencia, Ñuñoa y Santiago. Y el rótulo de comuna de la ficha puede ser absurdo — marcó "Molina, Maule" un departamento en Paul Harris Sur. Filtrar por lat/lng contra el polígono comunal y confirmar en la ficha.

**⚠️ Vigencia.** Muchas fichas sin actualizar desde 2023-2025, sindicadas de otros portales. Hay que leer el "Actualizado el" y descartar o marcar las antiguas: buena parte probablemente ya no está a la venta.

## Houm

```
/cl/venta-departamentos-region-metropolitana/las-condes/desde-{D}-dormitorios/hasta-{D}-dormitorios/{1-bano|2-banos}/1-estacionamiento?currency=CLF&maxPrice={MAX}&minArea={M2}
```

El tramo de dos baños es `2-banos` (confirmado en la corrida del 19-10).

**⚠️ Rellena sin avisar.** Si no hay resultados en la comuna, muestra avisos de comunas vecinas (Ñuñoa, La Reina) sin marcarlos. Hay que validar la comuna de cada aviso. Además `minArea` es sobre m² **totales**, no útiles.

**Recomendación: sacarlo.** En la corrida completa aportó 1 aviso de 250, y ese quedó descartado (118 UF/m²). No justifica el código ni el tiempo.

## Reglas transversales

**Deduplicación.** Clave útil: precio en UF exacto + m² útiles (±1) + calle o coordenadas. Los títulos genéricos ("Departamento en venta de 2 dorm. en Las Condes") son de corredores que no publican dirección: para esos, deduplicar por precio + m² y marcar como "posible duplicado" en vez de fusionar a ciegas.

**Limpieza.** Descartar precio < 1.000 UF, "cesión de promesa", "vendido", propiedades fuera de Chile y m² más de 10x lo esperado. Buscar "sin estacionamiento" en título y descripción aunque el portal diga que tiene.

**Regla general sobre los filtros de portal.** Tres de los cinco portales mienten en al menos un filtro. Tratá todo filtro de URL como una optimización para traer menos páginas, nunca como garantía: la validación se hace siempre sobre los datos del aviso.
