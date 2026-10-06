# Bitácora de IA

Herramienta utilizada: Claude Code (modelo Opus 5.5).

## Task 1

### Prompt

```text
Implementa en un Jupyter Notebook nuevo el preprocesamiento y el análisis de buffers de cobertura
hospitalaria para Iowa. Usa GeoPandas, pandas, NumPy, matplotlib y matplotlib-scalebar. Los datos
están en data/: hospitales_eeuu.geojson, condados_eeuu.geojson, estados_eeuu.geojson y
poblacion_condados.csv.

Define como constantes al inicio: ESTADO = "IA", COD_ESTADO = "19" y CRS_PROY = "EPSG:26915"
(NAD83 / UTM 15N, en metros).

Estructura del código:

1. Carga de los cuatro archivos e impresión, para cada uno, del número de registros, el CRS (o
   "No aplica" para el CSV) y la lista de columnas.
2. Limpieza sobre el dataset completo, antes de filtrar: elimina de la población los FIPS >= 80000
   y convierte `fips` a texto de 5 dígitos con `zfill(5)`. Imprime el porcentaje de hospitales con
   `camas_total` nulo y conserva solo los que tienen ese dato.
3. Filtro de hospitales con `estado == ESTADO` y de condados con `cod_estado == COD_ESTADO`. Une la
   población a los condados con `merge` por `fips` e imprime el conteo de condados con geometría y
   con población.
4. Reproyección de las tres capas a CRS_PROY.
5. Una función `mapa_base(ax)` que dibuje los condados en gris claro, el límite estatal sin relleno
   y una `ScaleBar(1, units="m")`. Sobre ella, los hospitales coloreados por `tipo` como variable
   categórica, con la leyenda fuera del área del mapa.
6. Tabla por condado: asigna cada hospital a su condado con `gpd.sjoin(predicate="within")`, agrega
   número de hospitales, camas totales y camas UCI, rellena con 0 los condados sin hospital y
   calcula camas y camas UCI por cada 10,000 habitantes.
7. Los cinco condados con menos camas por habitante entre los de más de 10,000 habitantes, con
   empates en camas desempatados por población descendente. Los cinco con más camas por habitante,
   junto con el nombre, tipo y camas de los hospitales que contienen.
8. Dos histogramas lado a lado (camas por hospital y camas por 10k por condado) y una tabla con
   media, mediana, máximo y asimetría (`skew`) de ambas variables.
9. Cuatro funciones reutilizables:
   - `cobertura_unificada(puntos, radio_km)`: buffers unidos con `union_all()`.
   - `fraccion_cubierta(condados, area)`: área intersectada entre área del condado.
   - `cobertura_poblacional(condados, area)`: suma de fracción cubierta × población.
   - `clasificar_cobertura(fraccion, tol=0.999)`: "Completo", "Parcial" o "No cubierto" con
     `np.select`, usando la tolerancia para absorber fracciones que salen ligeramente mayores a 1.
10. Tabla con radio_km, poblacion_cubierta, poblacion_total y cobertura_% para 10, 25 y 50 km.
11. Mapa de 25 km: condados coloreados por clase con una paleta divergente RdYlBu (rojo = no
    cubierto, azul = completo), las zonas sin cobertura `estado.difference(cobertura)` sombreadas en
    rojo semitransparente, los hospitales como puntos negros y la leyenda fuera del mapa.

Documenta las funciones con docstrings PEP 257.
```

### ¿Por qué funcionó?

Funcionó porque el prompt entrega como dato todo lo que de otra forma habría que adivinar: los nombres de los archivos, el código FIPS del estado y el CRS proyectado. Al declararlos como constantes al inicio, el resto del código no tiene valores escritos a mano y cambiar de estado se reduce a editar tres líneas. El orden de la limpieza también se fija de forma explícita (primero sobre todo el país y después el filtro).

La decisión clave es separar la cobertura en cuatro funciones pequeñas. `cobertura_poblacional` recibe cualquier geometría, así que la misma función sirve para los buffers de 10, 25 y 50 km, para la curva de 5 a 100 km del Task 2 y para comparar con las isocronas del Task 3. `clasificar_cobertura` existe porque la intersección de geometrías produce fracciones como 1.0000001, que un `pd.cut` con límites exactos deja fuera y convierte en valores nulos al colorear el mapa.

Finalmente, los detalles visuales que suelen fallar se piden de forma concreta. La leyenda fuera del área evita que tape el noreste del estado. Sombrear `estado.difference(cobertura)` hace visible la brecha aunque ningún condado quede totalmente sin cobertura. Y el desempate por población en los cinco condados con menos camas evita que el resultado dependa del orden de las filas cuando hay varios condados con cero camas.

## Task 2

### Prompt

```text
Continúa el notebook con el análisis de distancia, el índice de vulnerabilidad y la ubicación de
nuevos hospitales. Reutiliza `cobertura_unificada`, `fraccion_cubierta`, `cobertura_poblacional` y
`clasificar_cobertura`, y no dupliques la lógica de cobertura.

Estructura del código:

1. Distancia del centroide de cada condado al hospital más cercano con `gpd.sjoin_nearest` y
   `distance_col`, convertida a km y guardada en `cond_ia["dist_km"]`. Elimina duplicados por
   `fips` (empates a la misma distancia). Muestra los diez condados más alejados.
2. Curva de cobertura para radios `np.arange(5, 105, 5)` con la ganancia en puntos porcentuales
   entre radios consecutivos. El punto de inflexión es el punto de la curva más alejado de la recta
   que une el primer y el último punto. Grafícala con una línea vertical en ese punto.
3. Para los cinco condados más alejados: nombre, tipo y camas del hospital más cercano, y la mediana
   de camas de esos hospitales frente a la mediana estatal.
4. Una función `min_max(s)` y tres componentes por condado, normalizados con ella:
   - c1: distancia al hospital más cercano.
   - c2: inverso de camas por 10k; a los condados sin camas se les asigna el máximo de los inversos.
   - c3: ocupación promedio (`ocupacion_total`) de los hospitales a <= 50 km del centroide, con
     `sjoin` del centroide contra buffers de 50 km; sin hospitales cercanos, se asigna el máximo.
5. `PESOS = {"c1_dist": 0.40, "c2_camas": 0.35, "c3_ocup": 0.25}` y una función
   `calcular_indice(df, pesos)` que devuelva el promedio ponderado. Categoriza con
   `pd.qcut(..., 5)` en "Muy baja", "Baja", "Media", "Alta" y "Muy alta".
6. Mapa coroplético YlOrRd por categoría con los nombres de los diez condados de mayor índice
   anotados sobre `representative_point()`.
7. Grilla de candidatos con `np.meshgrid` cada 50,000 m sobre `total_bounds` del estado, conservando
   solo los puntos que intersecan el límite estatal con `gpd.sjoin(predicate="intersects")`.
8. Una función `poblacion_adicional(punto, area_cubierta)` que calcule el buffer de 25 km menos el
   área ya cubierta y lo multiplique por la densidad de población de cada condado.
9. Algoritmo voraz de 3 iteraciones: en cada una recalcula la ganancia de todos los candidatos con
   la cobertura vigente, elige el `np.argmax`, une su buffer a la cobertura y registra iteración,
   población adicional, cobertura acumulada y condado del punto (con `sjoin`). Parte de la
   cobertura actual a 25 km.
10. Mapa final con la misma paleta del mapa de 25 km, los buffers de los puntos propuestos con línea
    discontinua y los puntos como estrellas numeradas por iteración.
```

### ¿Por qué funcionó?

Funcionó porque el prompt obliga a reutilizar las funciones del Task 1 en lugar de reescribir la lógica de cobertura. Con eso, la curva de 5 a 100 km son veinte llamadas a la misma función, y el algoritmo voraz mide la cobertura acumulada exactamente igual que la tabla de buffers. Así no hay dos implementaciones que puedan dar resultados distintos.

También define cada componente del índice con su regla para los casos sin datos. Decir de forma explícita que los condados sin camas reciben el máximo de los inversos evita la división entre cero (que daría infinito y rompería la normalización). Y los pesos se entregan como un diccionario, de modo que `calcular_indice` recibe cualquier configuración. Esa misma función se reutiliza después en el análisis de sensibilidad del Task 3 sin cambiar nada.

Finalmente, las dos decisiones que suelen salir mal en el MCLP se fijan en el prompt. La ganancia de cada candidato se recalcula en cada iteración contra la cobertura vigente, que es lo que distingue un algoritmo voraz real de simplemente tomar los tres mejores candidatos del cálculo inicial. Y el punto de inflexión se define con una regla geométrica concreta (la mayor distancia a la recta entre extremos), en lugar de un umbral arbitrario de ganancia marginal que podría cambiar el resultado según quién lo elija.

## Task 3

### Prompt

```text
Continúa el notebook con la comparación entre buffer e isocrona sobre la red vial real y el
análisis de sensibilidad del índice. Usa OSMnx, NetworkX y pyosmium. La máquina tiene unos 3 GB de
RAM.

Estructura del código:

1. Selección de hospitales: de los condados con al menos un hospital, toma los 3 de mayor índice y,
   en cada uno, el hospital con más camas.
2. Red vial por hospital, en dos métodos:
   - Método estándar: `ox.graph_from_point((lat, lon), dist=55_000, network_type="drive")`.
   - Respaldo, si Overpass lanza `RequestException` o `ValueError`: descarga (solo si no existe)
     data/iowa-latest.osm.pbf de Geofabrik y, con una función `extraer_red(pbf, lat, lon, dist_m,
     salida)`, recorta una caja de lado 2·dist_m usando `osmium.FileProcessor(...).with_locations()`,
     un `TagFilter` con los `highway` vehiculares (motorway a residential y sus `_link`), excluyendo
     `access=private/no`, y un `BackReferenceWriter` para escribir las vías con sus nodos en
     data/osm/red_<fips>.osm. Carga el recorte con `ox.graph_from_xml`.
   - Tras el primer fallo de Overpass no lo vuelvas a intentar y genera los tres recortes antes de
     construir cualquier grafo.
3. Para cada grafo: ordena las listas del atributo `highway` en las aristas fusionadas (para que
   `add_edge_speeds` sea determinista entre ejecuciones), aplica `add_edge_speeds` y
   `add_edge_travel_times`, y calcula la isocrona de 30 min con una función
   `isocrona(G, lat, lon, minutos=30)`: `nearest_nodes`, `nx.single_source_dijkstra_path_length`
   con `cutoff` en segundos y la envolvente convexa de los nodos alcanzables en CRS_PROY.
4. Para no agotar la memoria, de cada grafo conserva solo las aristas proyectadas (highway, length,
   speed_kph, geometry) y elimínalo con `del` antes de pasar al siguiente. Imprime la fuente usada,
   nodos y aristas.
5. Tabla por hospital con área y población del buffer de 25 km y de la isocrona (con
   `cobertura_poblacional`), la diferencia absoluta y la porcentual.
6. Figura de tres paneles: condados, red vial gris con las vías principales resaltadas, isocrona
   naranja semitransparente, buffer azul discontinuo y el hospital como estrella, encuadrada a la
   unión de isocrona y buffer.
7. Resumen de km de red y velocidad media asignada por tipo de vía.
8. Sensibilidad del índice: un diccionario con cinco configuraciones de pesos (base, iguales y tres
   que priorizan un componente con 0.6/0.2/0.2). Calcula el ranking de cada una con
   `calcular_indice`, el número de condados del top-10 que coinciden con la configuración base, la
   correlación de Spearman con el ranking base, la posición del top-10 base en cada configuración y
   los condados presentes en el top-10 de todas.
```

### ¿Por qué funcionó?

Funcionó porque el prompt anticipa el punto débil del Task 3: depender de un servicio externo. La API Overpass puede bloquear la IP cuando recibe muchas consultas, y con un tamaño de 110 km OSMnx divide cada descarga en varias subconsultas. El respaldo con Geofabrik usa los mismos datos de OpenStreetMap desde un archivo descargable, de modo que el notebook corre aunque Overpass no responda. Pedir que no se reintente después del primer fallo evita repetir la pausa de 60 segundos que OSMnx aplica cuando el servidor no contesta.

También entrega como dato la restricción de memoria. Recortar el PBF con `with_locations()` ocupa cerca de 2 GB, y varios grafos completos al mismo tiempo agotan una máquina de 3 GB. Generar primero todos los recortes y conservar después solo las aristas proyectadas de cada grafo mantiene el pico por debajo del límite, sin cambiar el resultado. El radio de 55 km tampoco es arbitrario: a la velocidad de autopista, 30 minutos alcanzan unos 50 km, y una descarga más pequeña recortaría la isocrona por el borde del grafo.

Finalmente, el prompt incluye un detalle que solo aparece al repetir la ejecución. Al simplificar el grafo, OSMnx guarda en listas los tipos de vía de los tramos fusionados, y el orden de esas listas cambia entre procesos. Eso cambia la velocidad asignada y produce isocronas ligeramente distintas con el mismo dato. Ordenarlas antes de `add_edge_speeds` hace que el resultado sea idéntico en cada ejecución. Del mismo modo, reutilizar `calcular_indice` en la sensibilidad garantiza que las cinco configuraciones se comparen con exactamente la misma fórmula que el índice original.
