# Aprobación 360

App de aprobación mensual del contenido de El Método 360. Una página, sin dependencias
externas, que lee las piezas del mes desde Supabase y guarda en la base el veredicto y las
notas de Chema. Todo lo que él escribe queda en la tabla, legible después por cualquier agente.

**App publicada:** https://elmetodo360.github.io/aprobacion-360/

---

## 1. Cómo se usa (Chema)

Se abre el enlace en el móvil o en el ordenador. No hay que instalar ni iniciar sesión.

1. **Arriba, el mes.** Entra por defecto en el mes más reciente que tenga piezas cargadas.
2. **Pestañas por red:** Instagram, Threads, LinkedIn, TikTok, YouTube, Facebook. Cada
   pestaña lleva el total de piezas y cuántas quedan sin revisar.
3. **Barra de progreso** del mes: aprobadas, con cambios, descartadas y sin tocar.
4. **Filtro:** Todas · Solo pendientes · Solo con cambios.
5. **Cada pieza es una tarjeta:**
   - El **código** (`AGO-IG-CAR-01`) en grande, con botón de copiar al lado.
   - Tipo, título y fecha y hora prevista de publicación.
   - Vista previa: carrusel con flechas y contador "3 / 10" (en móvil se desliza con el
     dedo), vídeo con controles, o el texto completo con su recuento de caracteres cuando
     la pieza no lleva imagen.
   - Caption completa, plegable.
   - Tres botones: **Aprobar · Pide cambios · Descartar**. El borde de la tarjeta cambia de
     color. Volver a pulsar el mismo botón deshace el veredicto y la deja sin revisar.
   - **Campo de notas.** Se guarda solo al salir del campo, y también con el botón
     "Guardar nota". Debajo aparece "Guardada en la base a las 12:47". Si algo falla lo
     dice en rojo: entonces la nota **no** está guardada y hay que reintentar.

Botón **Recargar** arriba a la derecha para traer el último estado desde la base.

---

## 2. Cómo doy yo de alta las piezas del mes (Renata)

Las altas se hacen con **clave de servicio** (MCP `supabase`, herramienta `execute_sql`, o el
SQL Editor del panel). El rol anónimo de la app **no puede insertar ni borrar**: solo lee y
actualiza `estado` y `nota`.

### Formato exacto de la fila

| Columna | Tipo | Obligatoria | Qué va |
|---|---|---|---|
| `mes` | text | sí | `AAAA-MM`. Ej. `2026-08`. Es lo que agrupa el selector de mes. |
| `red` | text | sí | `Instagram` · `Threads` · `LinkedIn` · `TikTok` · `YouTube` · `Facebook`. Cualquier otro valor crea su propia pestaña al final. |
| `codigo` | text | sí | **Único en toda la tabla.** Convención: `MES-RED-TIPO-NN` → `AGO-IG-CAR-01`. |
| `tipo` | text | sí | Etiqueta corta: `Carrusel`, `Reel`, `Story`, `Post`, `Artículo`, `Vídeo`. |
| `titulo` | text | sí | Título de la pieza, una línea. |
| `fecha_prevista` | timestamptz | no | Fecha y hora de publicación: `2026-08-07 13:30+02`. |
| `orden` | integer | no | Orden dentro de la red. Por defecto `0`. |
| `media_urls` | jsonb | no | **Array de URLs**: `'["https://…/01.jpg","https://…/02.jpg"]'::jsonb`. Vacío `'[]'` = pieza de texto puro. |
| `caption` | text | no | Caption completa. En piezas de texto puro, es el texto que se muestra en grande. |
| `notas_produccion` | text | no | Nota mía para él (audio usado, fuente del dato, etc.). |
| `estado` | text | no | Se deja en `pendiente` (valor por defecto). Lo cambia él. |
| `nota` | text | no | Se deja vacío. Lo escribe él. |

**Detección del media:** si la URL acaba en `.mp4 .mov .webm .m4v .ogv` se pinta como vídeo;
cualquier otra se pinta como imagen. Las URLs tienen que ser públicas y servidas por HTTPS
(Supabase Storage o Drive con enlace directo); si no cargan, la tarjeta lo avisa.

### Convención de códigos

```
AGO-IG-CAR-01     agosto · Instagram · carrusel · pieza 01
AGO-IG-REEL-03    agosto · Instagram · reel · pieza 03
AGO-LI-POST-02    agosto · LinkedIn · post · pieza 02
AGO-TH-TXT-05     agosto · Threads · texto · pieza 05
```

Prefijos de red: `IG` · `TH` · `LI` · `TT` · `YT` · `FB`.
Prefijos de tipo: `CAR` (carrusel) · `REEL` · `STO` (story) · `POST` · `TXT` · `VID` · `ART`.

### Plantilla de alta

```sql
insert into public.aprobaciones_contenido
  (mes, red, codigo, tipo, titulo, fecha_prevista, orden, media_urls, caption, notas_produccion)
values
  ('2026-08','Instagram','AGO-IG-CAR-01','Carrusel',
   'Los 3 números que te dicen si el mes va bien',
   '2026-08-04 13:30+02', 1,
   '["https://…/ago-ig-car-01-1.jpg","https://…/ago-ig-car-01-2.jpg"]'::jsonb,
   'Caption completa de la pieza…',
   'Dato del INE, julio 2026.'),

  ('2026-08','LinkedIn','AGO-LI-POST-02','Post',
   'Lo que nadie mira del escandallo',
   '2026-08-05 08:15+02', 2,
   '[]'::jsonb,
   'Texto completo del post. Al no llevar media, esto es lo que se ve en grande.',
   null);
```

### Leer después lo que ha escrito Chema

```sql
select codigo, red, tipo, titulo, estado, nota, actualizado_en
from public.aprobaciones_contenido
where mes = '2026-08' and (estado <> 'pendiente' or nota is not null)
order by red, orden;
```

Solo lo que pide corrección:

```sql
select codigo, titulo, nota
from public.aprobaciones_contenido
where mes = '2026-08' and estado = 'cambios'
order by red, orden;
```

### Cerrar el mes

No se borra nada. Las filas del mes se quedan como registro; el selector muestra siempre el
mes más reciente. Si hay que retirar una pieza mal cargada, se borra con clave de servicio:

```sql
delete from public.aprobaciones_contenido where codigo = 'AGO-IG-CAR-01';
```

---

## 3. Técnico

- **Hosting:** GitHub Pages, cuenta `Elmetodo360`, rama `main`, carpeta raíz.
- **Datos:** Supabase, proyecto `qczgcprniskfvxxetrah`.
- **Clave en el cliente:** publicable (`sb_publishable_…`), hardcodeada en el JS. Es la
  pensada para ir en el navegador. La URL del backend va hardcodeada, nunca en localStorage.
- **Seguridad:** RLS activo. El rol anónimo tiene `select` sobre toda la tabla y `update`
  **solo** sobre las columnas `estado`, `nota` y `actualizado_en` (permiso a nivel de columna,
  no solo política). No puede insertar ni borrar. Un visitante con el enlace puede leer el
  contenido del mes y cambiar veredictos: el enlace es la llave, no se comparte fuera.
- **Sin dependencias:** todo el CSS y el JS van dentro del `index.html`. Ni CDN ni fuentes
  externas. Se llama a la API REST de Supabase con `fetch`.
- **Marca:** carbón `#0E0E10`, dorado `#E1A730`, crema `#EDE9E3`, burdeos `#A32834`, más un
  verde y un ámbar apagados para los semáforos de estado. Tipografía del sistema. Sin emojis.
- **Móvil:** una columna, botones de 52 px, carrusel con deslizamiento.

### Publicar un cambio

```bash
cd C:/dev/aprobacion-360
git add -A && git commit -m "…" && git push
```

GitHub Pages lo publica en un minuto. Si no se ve el cambio, recargar forzando caché.
