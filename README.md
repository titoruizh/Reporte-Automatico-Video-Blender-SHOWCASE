# BLENDER-REVANCHAS

Pipeline para generar visualizaciones 3D de inspección de muros de embalse. Toma nubes de puntos de los muros, genera una malla 3D, ubica esferas de color sobre cada punto kilométrico según su revancha y ancho de coronamiento, y renderiza un video animado para presentar los resultados.

El proyecto es para el embalse Las Tórtolas (Chile). Tiene tres muros: Principal, Oeste y Este.

<p align="center">
  <img src="https://github.com/user-attachments/assets/9ab1ab62-8a5f-4c02-b902-7e31f220aa9f" width="100%" />
</p>



---

## Qué hace exactamente

1. **Toma una nube de puntos** (LAZ o ASC) del muro y la convierte a malla OBJ usando CloudCompare headless.
2. **Importa la malla a Blender** y le aplica un TIF de colorimetría encima (drape).
3. **Ubica esferas de alerta** sobre cada PK del muro:
   - En el eje del muro → esfera de **revancha** (rojo/amarillo/verde según cuántos metros hay hasta la cota de agua)
   - A distancia perpendicular del eje → esfera de **ancho de coronamiento** (rojo/amarillo/verde)
4. **Agrega labels de texto 3D** con el valor numérico encima de cada esfera.
5. **Anima una cámara** recorriendo el muro siguiendo un path definido en DXF.
6. **Renderiza el video** con un HUD overlay (logo, título).

[imagen: comparación eje del muro con PKs marcados en QGIS vs esferas en Blender — misma posición]

---

## Umbrales de color

| Métrica | Rojo | Amarillo | Verde |
|---------|------|----------|-------|
| Revancha | ≤ 3.0 m | ≤ 3.2 m | > 3.2 m |
| Ancho coronamiento | < 15.0 m | ≤ 18.0 m | > 18.0 m |

---

## Cómo funciona por dentro

Los datos de entrada son dos cosas completamente separadas:

- **Geometría (posiciones):** hardcodeada en `src/wall_geometry.py`. Aquí están las coordenadas UTM de cada PK y la dirección perpendicular para cada muro. Esto no cambia entre inspecciones.
- **Valores (revancha y ancho):** vienen de un Excel de cada campaña de medición. Se extraen con `parse_excel.py` y se guardan en un CSV simple.

En Blender, la posición Z de cada esfera no viene de ningún archivo — se obtiene haciendo un raycast contra la malla del terreno ya importada. Así las esferas siempre quedan sobre la superficie sin importar la topografía.

CRS: **EPSG:32719** (WGS84 UTM Zona 19S)


![unnamed (1)](https://github.com/user-attachments/assets/f599c296-ac5d-4807-801c-25d20a0cae2d)

---

## Estructura

```
src/
├── ui_pipeline.py          # GUI tkinter para ejecutar fases
├── wall_geometry.py        # Backend de geometría: coordenadas UTM de los 3 muros
├── pipeline/
│   ├── phase0_mesh.py      # LAZ/ASC → OBJ (CloudCompare)
│   ├── 01_import_and_drape.py
│   ├── 01b_drape_secondary.py
│   ├── 02_mark_bad_sections.py   # Esferas revancha
│   ├── 03_camera_path.py
│   ├── 04_render_video.py
│   ├── 05_mark_widths.py         # Esferas ancho
│   ├── 06_hud_overlay.py
│   └── 07_vse_labels.py          # Labels 3D
└── muros/
    ├── muro_principal/parse_excel.py
    ├── muro_oeste/parse_excel.py
    └── muro_este/parse_excel.py

muros/
├── muro_principal/input/   # Excel, TIF, LAZ, DXF de cámara
├── muro_oeste/input/
└── muro_este/input/

output/
├── muro_principal/
├── muro_oeste/
└── muro_este/
```


![wmremove-transformed](https://github.com/user-attachments/assets/90138be5-fe9b-4974-adc4-154e79c7311d)


---

## Dependencias

- **Blender** 3.x o 4.x (para todas las fases de escena y render)
- **CloudCompare** headless (Fase 0 — mallado de nube de puntos)
- **Python 3.x** con: `openpyxl`, `tkinter` (incluido en stdlib)
- Datos de entrada por muro: nube de puntos (LAZ/ASC), TIF de colorimetría, Excel de campaña, DXF de path de cámara

---

## Muros configurados

| Muro | Tipo | Longitud | PKs |
|------|------|----------|-----|
| Muro Principal | Recto | 1434 m | cada 20 m |
| Muro Oeste | Curvo | 689 m | 36 estaciones explícitas |
| Muro Este | Recto | ~550 m | cada 20 m |

[imagen: vista aérea del embalse con los tres muros identificados]

---

## Notas técnicas

Las coordenadas UTM del embalse son grandes (~337000 E, ~6334000 N). Blender trabaja en float32 internamente, así que si se pasan las coordenadas brutas las posiciones salen con zigzag por pérdida de precisión. Todos los scripts restan un origen UTM antes de crear objetos en la escena.

El Muro Oeste tiene PKs con decimales acumulados (20.001 m, 40.003 m, ...) porque las estaciones no son exactamente cada 20 m. El sistema de matching los empareja con tolerancia de 11 m.
