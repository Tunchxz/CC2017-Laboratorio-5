# Laboratorio 5

Se construye un modelo espacial de cobertura hospitalaria para el estado de **Iowa (EE.UU.)** con GeoPandas, OSMnx y pyosmium. Se combinan geometrías de condados, hospitales con capacidad de camas, población por condado y límites estatales para estimar qué fracción de la población tiene acceso hospitalario a una distancia razonable y dónde se ubican las brechas.

## Cómo ejecutar

1. Crear el entorno virtual e instalar dependencias (Python 3.12):

   ```bash
   python3.12 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

2. Abrir y ejecutar el notebook:

   ```bash
   jupyter lab laboratorio5.ipynb
   ```

   O ejecutarlo completo desde la terminal:

   ```bash
   jupyter nbconvert --to notebook --execute --inplace laboratorio5.ipynb
   ```

> **NOTA (Task 3.1. Red Vial)**
> 
> La red se descarga primero con OSMnx desde la API Overpass (las respuestas se guardan en `cache/`). Si Overpass no está disponible o se bloquea (por la cantidad de subconsultas requeridas), se usa automáticamente el extracto de OpenStreetMap de Iowa de Geofabrik (`data/iowa-latest.osm.pbf`, ~134 MB) y se recorta con `osmium` en `data/osm/`. Ambos archivos se reutilizan en las ejecuciones siguientes.
