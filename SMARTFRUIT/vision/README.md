# SmartFruit — Vision AI

## 1. Descripción

SmartFruit es un proyecto académico que desarrolla componentes de visión artificial para apoyar la inspección de limones y la identificación de oleocelosis.

Este repositorio documenta dos notebooks experimentales:

- `SmartFruit_Baseline_Ordenado.ipynb`: entrenamiento y evaluación de un detector YOLO11n.
- `SmartFruit_Prototipo_V0_DINOv2.ipynb`: clasificación binaria de imágenes a partir de embeddings DINOv2-small y un clasificador de regresión logística, además de funciones de predicción y salida JSON.

Los notebooks se mantienen como archivos independientes. Este README describe su propósito y uso; no modifica su código.

## 2. Componentes

### 2.1. Baseline YOLO11n

El notebook `SmartFruit_Baseline_Ordenado.ipynb` implementa un flujo de detección mediante YOLO11n:

1. Configuración del entorno y de las rutas del proyecto.
2. Carga del dataset y del archivo `data.yaml`.
3. Inicialización de YOLO11n con pesos preentrenados.
4. Entrenamiento con los parámetros configurados en el notebook.
5. Recuperación de pesos y revisión de resultados.
6. Evaluación del detector sobre el conjunto de prueba.

La configuración de entrenamiento visible en el notebook incluye 50 épocas, tamaño de imagen de 640, batch de 16, paciencia de 10 y semilla 42. El notebook utiliza `deterministic=True` y guarda los resultados en una carpeta de ejecución separada.

**Tarea:** detección de objetos/regiones anotadas mediante cajas delimitadoras.  
**Métricas de evaluación:** Precision, Recall, mAP@50 y mAP@50–95, además de resultados por clase.

### 2.2. Prototipo DINOv2

El notebook `SmartFruit_Prototipo_V0_DINOv2.ipynb` desarrolla un flujo de clasificación binaria:

1. Lectura de las etiquetas desde los archivos de anotación YOLO.
2. Organización de registros según `train`, `val` y `test`.
3. Revisión de imágenes utilizables, excluidas y posibles nombres repetidos entre particiones.
4. Extracción de embeddings con `facebook/dinov2-small`.
5. Entrenamiento/evaluación de un clasificador basado en regresión logística.
6. Predicción sobre imágenes o recortes individuales.
7. Construcción de una salida JSON para integrar los resultados del componente de visión.

El código asigna la etiqueta positiva cuando las anotaciones contienen la clase `1`; si solo contienen la clase `0`, asigna la etiqueta negativa. Los registros sin anotación válida o con clases desconocidas se excluyen.

**Tarea:** clasificación binaria de imágenes/recortes según las etiquetas derivadas de las anotaciones.  
**Salida de integración:** el contrato JSON contempla el estado de la evaluación, cantidad de frutos procesados, conteo y porcentaje estimado de oleocelosis, confianza media, versión del modelo, resultados por recorte y errores.

> Importante: la función JSON recibe rutas de imágenes o recortes (`image_paths`). Este notebook no debe interpretarse, por sí solo, como un detector automático de todos los limones presentes en una fotografía grupal.

## 3. Entorno y dependencias

Los notebooks están preparados para ejecutarse en un entorno Python y utilizan rutas de almacenamiento que deben revisarse según la ubicación real de los archivos. Si se ejecutan en Google Colab, se debe verificar el montaje de Google Drive y la existencia de los directorios del proyecto.

Entre las dependencias utilizadas por los notebooks se encuentran:

- Python
- PyTorch
- Ultralytics (YOLO)
- Transformers
- scikit-learn
- pandas
- NumPy
- Pillow
- joblib
- tqdm

El notebook DINOv2 instala explícitamente `transformers`, `scikit-learn`, `joblib` y `tqdm`. Para YOLO, se debe contar con la instalación de Ultralytics y sus dependencias.

## 4. Datos y estructura esperada

El flujo DINOv2 espera un dataset organizado por particiones y directorios de imágenes y etiquetas:

```text
dataset/
├── images/
│   ├── train/
│   ├── val/
│   └── test/
└── labels/
    ├── train/
    ├── val/
    └── test/
```

Para el entrenamiento YOLO se requiere un archivo `data.yaml` con las rutas y clases correspondientes al dataset utilizado.

Antes de ejecutar:

- Confirmar que las rutas configuradas en cada notebook existan.
- Verificar que imágenes y archivos de etiquetas correspondan entre sí.
- Confirmar que las clases del dataset mantengan la codificación esperada por el notebook DINOv2.
- No mezclar imágenes de una misma fuente o sesión entre particiones si ello puede producir fuga de información.

## 5. Ejecución

Se recomienda ejecutar cada notebook por separado y en orden, desde la primera celda hasta la última.

1. Abrir el notebook en Google Colab o Jupyter.
2. Configurar las rutas de datos y resultados.
3. Instalar o verificar las dependencias.
4. Ejecutar las celdas secuencialmente.
5. Revisar los mensajes de validación, archivos generados y métricas.
6. Conservar los pesos, parámetros, registros y resultados de cada ejecución.

No se debe asumir que los resultados serán idénticos en cualquier equipo: pueden variar por versiones de librerías, disponibilidad de GPU, rutas, archivos de entrada y configuración del entorno.

## 6. Interpretación de resultados

YOLO11n y DINOv2 responden a tareas distintas:

- YOLO11n localiza objetos o regiones mediante cajas delimitadoras.
- DINOv2 con regresión logística clasifica imágenes o recortes a partir de embeddings.

Por tanto, sus métricas no deben compararse directamente como si evaluaran exactamente el mismo problema. La selección de un modelo para uso operativo requiere revisar falsos negativos, resultados por clase, calidad de las anotaciones y desempeño en imágenes nuevas.

Los resultados obtenidos en los notebooks son experimentales. No constituyen por sí solos una validación en condiciones reales de operación ni autorizan el uso autónomo del sistema para decisiones de control de calidad.

## 7. Limitaciones y trabajo pendiente

- Verificar las rutas y versiones de dependencias antes de reproducir los experimentos.
- Documentar las versiones exactas de Python, PyTorch, Transformers y Ultralytics usadas en cada ejecución.
- Mantener identificables las versiones del dataset, las etiquetas y los pesos entrenados.
- Validar el desempeño con imágenes de nuevas sesiones y condiciones de captura.
- Revisar los errores, especialmente los falsos negativos de oleocelosis.
- Mantener supervisión humana en la inspección y en la interpretación de las predicciones.

## 8. Archivos principales

| Archivo | Propósito |
|---|---|
| `SmartFruit_Baseline_Ordenado.ipynb` | Entrenamiento y evaluación del baseline de detección YOLO11n. |
| `SmartFruit_Prototipo_V0_DINOv2.ipynb` | Extracción de embeddings DINOv2, clasificación binaria y prototipo de predicción/salida JSON. |
| `README.md` | Descripción del proyecto, componentes e instrucciones generales de ejecución. |

---

**SmartFruit — Vision AI**  
Documento descriptivo de los notebooks experimentales.
