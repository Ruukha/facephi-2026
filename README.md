# Facephi Challenge 2026

Procesamiento OCR y verificación biométrica de DNI/NIE español (frente y reverso).

## Descripción

El sistema extrae, categoriza y valida la información de un Documento Nacional de Identidad español utilizando reconocimiento óptico de caracteres (OCR). Procesa ambas caras del documento y cruza los datos para garantizar consistencia.

## Algoritmos y técnicas utilizadas

### Distancia de Levenshtein

Métrica de similitud entre cadenas de texto. Calcula el número mínimo de operaciones (inserción, eliminación, sustitución) necesarias para transformar una cadena en otra. Se usa para:

- **Validación cruzada de nombres**: comparar el nombre/apellidos extraídos del frente con los del MRZ, tolerando errores OCR menores (umbrales: nombre ≤ 2, apellidos ≤ 3).
- **Corrección OCR de MRZ**: búsqueda de apellidos/nombre dentro de la línea 3 del MRZ cuando el OCR corrompe los separadores.

### Parser MRZ

Parser completo del Machine Readable Zone en formato TD1 (3 líneas × 30 caracteres), estándar para documentos de identidad:

- **Línea 1**: tipo de documento + país emisor + número de documento + dígito de control + datos opcionales
- **Línea 2**: fecha nacimiento + check + sexo + fecha expiración + check + nacionalidad + datos opcionales + check compuesto
- **Línea 3**: apellidos `<<` nombre (separados por `<<`, palabras por `<`)

Incluye:
- **Checksums ICAO**: verificación con pesos [7, 3, 1] y tabla de valores A=10...Z=35
- **Detección por patrón**: clasifica cada línea por su estructura (no por orden), tolerando que el OCR las devuelva desordenadas
- **Corrección OCR de separadores**: el OCR confunde `<` con letras (S, K, X). Se usa el frente como referencia para reconstruir `ESPANOLA<ESPANOLA<<CARMEN<` a partir de `ESPANOLASESPANOLAS<CARMENS`, por ejemplo.

### Validación del DNI español

Algoritmo de módulo 23 para verificar la letra del DNI:
- El resto del número ÷ 23 es la letra correspondiente en `TRWAGMYFPDXBNJZSQVHLCKE`
- Soporta NIE (X=0, Y=1, Z=2 como prefijo numérico)

### Extracción contextual del frente

Parsing posicional por etiquetas: busca keywords como `APELLIDOS`, `NOMBRE`, `SEXO`, `NACIONALIDAD` y extrae el valor de las líneas siguientes, filtrando basura OCR por:
- Ratio de caracteres alfabéticos (≥ 80%)
- Longitud mínima (4 chars para apellidos, 3 para nombre)
- Keywords de parada para no mezclar secciones

### Extracción del reverso

Datos adicionales no presentes en el MRZ:
- Domicilio
- Lugar de nacimiento
- Hijo/a de (padres)

### Validación cruzada (Front vs MRZ)

Compara campo a campo los datos del frente con los del MRZ:
- DNI: coincidencia exacta
- Nombres/apellidos: Levenshtein con tolerancia
- Fechas, sexo, nacionalidad: coincidencia exacta
- Score global: porcentaje de campos coincidentes

### Análisis biométrico y score de calidad

Evaluación integral del documento que combina:
- Confianza promedio OCR (frente y reverso)
- Validez de checksums MRZ
- Validez de letra DNI
- Completitud de datos (7 campos principales)
- Score de validación cruzada
- Detección de expiración del documento

**Alertas de riesgo automáticas:**
- Documento expirado
- Letra DNI inválida
- Checksums MRZ fallidos
- MRZ no detectado
- Baja confianza OCR (< 70%)
- Datos incompletos (< 70%)
- Inconsistencia frente/MRZ (< 70%)

### Consolidación de datos

Estrategia de fusión frente + MRZ:
- Prioridad de la MRZ para campos estructurados (fechas, sexo, nacionalidad) por ser datos codificados con verificación
- Prioridad del frente para el número de DNI (texto más legible)
- Nombres: prioridad de la MRZ solo si pudo separar correctamente apellidos y nombre; si no, se usa el frente

## Estructura del proyecto

```
├── Baseline.ipynb          # Notebook principal
├── dni_front_especimen.jpg # Imagen frente del DNI
├── dni_back_especimen.jpg  # Imagen reverso del DNI
└── README.md
```

## Salida

El sistema genera:
1. **Visualización OCR**: bounding boxes coloreados por confianza (verde > 80%, amarillo > 50%, rojo ≤ 50%)
2. **Reporte completo**: datos personales, fechas, MRZ parseado, checksums, análisis biométrico
3. **JSON exportable**: datos estructurados en categorías