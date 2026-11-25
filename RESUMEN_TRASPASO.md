# Comparador de Baremos: resumen para continuar el trabajo

## 1. Qué es

Página web de una sola pantalla para revisar partidas o prestaciones de certificaciones:

1. Se elige la **población** o el código postal. La página sabe si es zona **URBANA** o **PROVINCIA**.
2. Se **pegan filas copiadas de Excel**.
3. Cada código se compara con el baremo de su zona:
   - 🟢 **Correcta**: el código vale en esa zona.
   - 🔴 **Incorrecta**: se propone la partida que correspondería.
4. Se copia una **columna con los códigos finales** para pegarla de vuelta en Excel.

- **Web pública (GitHub Pages):** https://jmmoreno0786.github.io/comparador-baremos/
- **Repositorio (público):** https://github.com/jmmoreno0786/comparador-baremos
- **Copia privada en Claude** (cuenta personal, no accesible desde otra cuenta): https://claude.ai/artifact/L7B2c51H9NBrvVcPff3jCE

Es HTML, CSS y JavaScript sin librerías ni servidor. Todo se ejecuta en el navegador.

## 2. Archivos del repositorio

| Archivo | Qué contiene |
|---|---|
| `index.html` | Toda la página: estilos, interfaz y lógica |
| `datos.json` | Baremos URBANA y PROVINCIA, listado de CP/zona y equivalencias de la tabla de comparativas (unos 1,6 MB) |
| `.nojekyll` | Para que GitHub Pages sirva los archivos tal cual |
| `README.md` | Descripción breve |

## 3. Archivos Excel de origen

Los Excel **no están en el repositorio**. Habrá que volver a pasarlos si cambian los datos.

| Excel | Hoja / columnas usadas |
|---|---|
| `CP_Y_ZONA_CONTRATA.xlsx` | Código Postal · Ciudad · Provincia · **ZONA CONTRATA** (URBANA/PROVINCIA). 212 filas, todo Granada |
| `BAREMO_EC_MTBT_26_v4_URBANA.xlsx` | Columnas: `CODIGO BAREMO DICIEMBRE` (C), `CODIGO NUEVOS CONTRATOS` (D, **código válido en URBANA**), `DESCRIPCION` (E), `ALCANCE` (F). 2.559 partidas |
| `BAREMO_UTE_SIERRA_NEVADA_JA10147715_PROVINCIA.xlsx` | `CODIGO NUEVO` · `DESCRIPCION` · `ALCANCE`. 2.335 partidas |
| `BAREMO_EC_MTBT_26_v4_comparativas.xlsx` | Igual que el de URBANA, más la columna `CODIGO ANTIGUO`. Columnas: C = `CODIGO BAREMO DICIEMBRE`, D = `CODIGO ANTIGUO`, E = `CODIGO NUEVOS CONTRATOS`, F = `DESCRIPCION`. "NUEVA" en una columna significa que esa partida no tiene código anterior |

Todos los códigos tienen 6 caracteres.

## 4. Formato de `datos.json`

```json
{
  "urbana":    [[codigo_nuevo, descripcion, alcance, cod_diciembre, cod_antiguo, cod_provincia], ...],
  "provincia": [[codigo, descripcion, alcance], ...],
  "zonas":     [[cp, poblacion, "URBANA"|"PROVINCIA"], ...]
}
```

En `urbana`, los tres últimos campos funcionan así:
- `cod_diciembre`: solo se rellena si es distinto del código nuevo y no es "NUEVA". Si no, va vacío.
- `cod_antiguo`: lo mismo, solo si es distinto del nuevo y no es "NUEVA".
- `cod_provincia`: el código que vale en PROVINCIA según la regla del apartado 5.2. Va vacío si la partida no tiene equivalente.

## 5. Reglas de negocio (acordadas con el usuario)

### 5.1 Zona
- Se busca la población o el CP en el listado. La comparación no distingue mayúsculas, tildes ni espacios: "Alhama de Granada" = "AlhamadeGranada".
- Si un CP tiene poblaciones de zonas distintas (p. ej. 18290: Chaparral Albolote URBANA, Calicasas PROVINCIA), se pide elegir.
- Si la población no está en el listado, el usuario puede **forzar** URBANA o PROVINCIA.

### 5.2 Comprobación de cada código

**URBANA**
1. Si el código está en `CODIGO NUEVOS CONTRATOS`: 🟢.
2. Si es un `CODIGO ANTIGUO` de la tabla de comparativas: 🔴 y se propone su código nuevo. Ejemplo: WSCA37 → WSCB31, YZEX01 → YZEX05.
3. Si es un `CODIGO BAREMO DICIEMBRE`: 🔴 y se propone su código nuevo. Ejemplo: WSCA90 → WSCB31.
4. Si no se encuentra: búsqueda por descripción (apartado 5.3).

**PROVINCIA.** La tabla de comparativas manda sobre el archivo del baremo PROVINCIA. El código válido de cada partida se calcula así:
- Se usa el **código de diciembre si es distinto del código nuevo**.
- Si es igual o está vacío, se usa el **código antiguo**.
- Ejemplos:
  - CANALIZ TIPO C (nuevo WSCB31, diciembre WSCA90, antiguo WSCA37): en PROVINCIA vale **WSCA90**.
  - YCFU06 (diciembre = YCFU06, antiguo YCFU03): en PROVINCIA vale **YCFU03**.
  - YZEX05 (diciembre YZEX03, antiguo YZEX01): en PROVINCIA vale **YZEX03**.

Pasos para cada código en PROVINCIA:
1. Si es el código válido en PROVINCIA de alguna partida de la tabla: 🟢. Si no está en el archivo PROVINCIA, se muestra un aviso.
2. Si es otro código de esa partida (nuevo, diciembre o antiguo): 🔴 y se propone el válido. Esto ocurre **aunque el código esté en el archivo PROVINCIA**. Ejemplo: WSCA37 → WSCA90.
3. Si no está en la tabla pero sí en el archivo PROVINCIA: 🟢.
4. Si no se encuentra: búsqueda por descripción.

Efecto de esta regla:
- 350 códigos del archivo PROVINCIA pasan a rojo.
- 554 códigos que no están en el archivo PROVINCIA se dan por buenos.
- Hay 4 códigos con papeles distintos en filas distintas de la tabla: WZMO01, WZMO02, WZAC01 y YAAV07. Se dan como correctos.

### 5.3 Búsqueda por descripción (cuando no hay equivalencia directa)
- Se usa la descripción pegada. Si no la hay y el código existe en el otro baremo, se usa la descripción de ese baremo.
- El parecido se calcula por trigramas y palabras comunes, y también con el texto del ALCANCE.
- Se muestran hasta 5 opciones, con el porcentaje de parecido.
- Si hay empate, gana la opción cuyo código se parece más al original. Muchas partidas comparten descripción, por ejemplo YZEX01, YZEX11 y YZEX21.
- Si el código existe pero la descripción pegada no se parece a la del baremo: 🟢 con aviso ⚠.

### 5.4 Elección y código final
- En cada fila roja hay un desplegable con las propuestas y la opción **"Dejar el inicial"**.
- Si no hay propuesta, se deja el código inicial.
- La columna "Código final" muestra:
  - el código original, si la fila es correcta;
  - el elegido en el desplegable, si es incorrecta;
  - el inicial, si no hay propuesta.
- Al cambiar de población o de filas se pierden las opciones elegidas a mano.

## 6. Interfaz
- **Paso 1, Población:** autocompletado con las opciones que **empiezan** por lo escrito (nombre o CP), con su zona al lado. Se maneja con el ratón o con ↑ ↓ y Enter. También hay un selector "Forzar baremo": Automático, URBANA o PROVINCIA.
- **Paso 2, Filas de Excel:** cuadro de texto donde se pegan las filas separadas por tabuladores.
- **Paso 3, Resultado:**
  - Detecta sola la columna del código (la que tiene más códigos conocidos de 6 caracteres) y la de la descripción (la de texto más largo). También detecta si la primera fila es cabecera. Las tres cosas se pueden cambiar a mano.
  - Resumen: correctas, incorrectas y avisos.
  - Tabla con estado, propuesta y código final.
- **Botones:**
  - **Copiar códigos finales (1 columna)**: copia una sola columna, sin cabecera y en el mismo orden que las filas pegadas.
  - **Copiar tabla completa**: copia las columnas originales más ESTADO, CODIGO FINAL y DESCRIPCION BAREMO. Se copia como tabla HTML y como texto, para que Excel separe bien las columnas.
  - **+ Nuevo** (arriba): deja todo en blanco, incluida la población.
- Debajo del título se muestra el número de partidas de cada baremo, el número de equivalencias de la tabla (ahora 2.427) y la versión de los datos (`DATA_VERSION`).
- Al abrir la página aparecen datos de ejemplo con la población Armilla.

## 7. Cómo actualizar
1. **Si cambian los Excel**, hay que regenerar `datos.json`. Con Python y openpyxl:
   1. Leer las columnas indicadas en el apartado 3 y limpiar espacios, incluidos los espacios no separables (`\xa0`) de los nombres de población.
   2. Calcular `cod_provincia` con la regla del apartado 5.2.
   3. Escribir el JSON con el formato del apartado 4.
2. **Cada vez que cambie `datos.json`**, hay que subir la constante `DATA_VERSION` en `index.html`. La página pide `datos.json?v=DATA_VERSION` sin caché, para que el navegador no use datos viejos.
3. **Para publicar**, basta con hacer push a la rama `main`. GitHub Pages está configurado con `main` y la carpeta raíz, y tarda 1-2 minutos. Después hay que recargar con Ctrl+F5.

## 8. Pendiente / dudas abiertas
- **WSCA37 en PROVINCIA:** el usuario dijo que seguía saliendo mal. En las pruebas, la versión actual da WSCA37 → WSCA90 en PROVINCIA. La causa probable era la caché del navegador. Falta que el usuario confirme que ve "Tabla de comparativas: 2427 equivalencias PROVINCIA · Datos v2026-11-23-3" debajo del título y que la zona marcada es PROVINCIA.
- Partidas con "NUEVA" en el código antiguo y sin código de diciembre propio (p. ej. YZMA27): no tienen equivalente en PROVINCIA y se buscan por descripción. La propuesta suele ser mala; el usuario tiene que usar "Dejar el inicial".
- El listado de CP tiene nombres escritos de dos formas, por ejemplo "AlhamadeGranada" (18120) y "Alhama de Granada" (18125). Están así en el Excel original.
- No se comprueban precios: los baremos recibidos no los tienen.
- La copia privada en Claude y la web de GitHub tenían el mismo contenido a 23/11/2026.
