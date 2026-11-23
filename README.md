# Inventario: revisión de stock y alertas de reposición

Automatización en **n8n** que revisa el stock de un producto, calcula cuántas unidades faltan, registra una alerta en **Supabase** y envía un aviso por **Gmail** redactado con **Google Gemini**.

> Curso: Automatización inteligente con n8n · EAM · 2026 II · Profesor: Robert Garcia Rey

## 1. Identificación

| Campo | Detalle |
|---|---|
| Proyecto | Inventario: revisión de stock y alertas de reposición |
| Integrantes | Michael Aguilera , Jorge Martínez, Valentina Gutiérrez, Gabriela Ríos |
| Repositorio | n8n-final-michael-aguilera-jorge-martinez-valentiva-gutierrez-gabriela-rios-inventario |


**Problema.** Revisar a mano si un producto necesita reposición implica abrir el inventario, comparar cifras, calcular el faltante y avisar a la persona encargada. Es repetitivo, propenso a errores y no deja registro.

**Objetivo.** Con una consulta por nombre de producto, detectar si necesita reposición, guardar la alerta sin duplicados y avisar por correo.

**Alcance de la versión final.**
- Incluye: consulta, validación, cálculo de prioridad, registro de alerta, correo con IA (con plantilla de respaldo), respuesta HTTP y registro de incidentes técnicos.
- No incluye: compras, modificación del stock, autenticación del Webhook ni reintento de correos fallidos (ver [limitaciones](#9-limitaciones-conocidas)).

## 2. Vista general

```
Webhook (GET) ─► Supabase: buscar producto ─► ¿Encontró? ─► Preparar/validar datos ─► ¿Válidos?
                                                 │ no              │ no
                                                 ▼                 ▼
                                        Producto no encontrado  Datos no válidos
                                                                   │ sí
                                          Calcular faltante y prioridad
                                                 │
                              stock > mínimo ◄───┴───► stock ≤ mínimo
                              (sin alerta)             Registrar alerta en Supabase
                                                       Gemini redacta (o plantilla)
                                                       Gmail envía el aviso
                                                 │
                                       Unir resultados ─► Respuesta al Webhook
```

| Servicio | Función |
|---|---|
| n8n | Orquesta el flujo |
| Supabase (REST/PostgREST) | Tablas `stock_productos`, `alertas_reposicion`, `incidentes` |
| Google Gemini | **Solo redacta** asunto y cuerpo del aviso. No decide stock, prioridad ni destinatario |
| Gmail (OAuth2) | Envía el aviso |

Detalle completo en la [documentación](Documentacion/Documentacion_Proyecto_Final.md).

## 3. Requisitos

- n8n (versión indicada arriba), en la nube o autoalojado.
- Cuenta de **Supabase** (plan gratuito sirve) con un proyecto nuevo.
- Clave **`service_role`** de ese proyecto (Project Settings → API). *Nunca* la publiques.
- Clave de API de **Google Gemini** (Google AI Studio).
- Cuenta de **Gmail** con credencial OAuth2 configurada en n8n.

## 4. Configuración

1. **Base de datos.** En Supabase → SQL Editor, ejecuta el script de creación de tablas y datos ficticios (`Preparar_Supabase.sql`; ubícalo en `workflows/` o `documentacion/` según quede en el repositorio). Debe terminar mostrando 5 filas de `stock_productos`.
2. **Credenciales en n8n** (Credentials → New), con nombres propios:
   - *Supabase API*: Host = `https://<TU-PROYECTO>.supabase.co`, Service Role Secret = tu clave `service_role`.
   - *Google Gemini (PaLM) API*: tu clave de Gemini.
   - *Gmail OAuth2*: autoriza tu cuenta.
3. **Parámetros a reemplazar** tras importar el workflow:

| Nodo | Parámetro | Reemplazar por |
|---|---|---|
| `HTTP Request` | URL `https://<TU-PROYECTO>.supabase.co/rest/v1/stock_productos` | URL de tu proyecto |
| `Registrar alerta en Supabase` | URL `.../rest/v1/alertas_reposicion` | URL de tu proyecto |
| `Registrar incidente en Supabase` | URL `.../rest/v1/incidentes` | URL de tu proyecto |
| `Enviar correo de alerta` | Destinatario | Tu correo de prueba |
| Nodos con credenciales | Credencial | La que creaste |

## 5. Importación y ejecución

1. En n8n: **Workflows → Import from file →** [`workflows/Workflow_Final.json`](Workflows/Workflow_Final.json).
2. Selecciona tus credenciales en cada nodo marcado y ajusta URLs y destinatario.
3. Para probar rápido: ejecuta con **Execute workflow** (el disparador manual trae un producto de ejemplo).
4. Para probar por HTTP: activa el workflow y llama al Webhook de **producción**:
   ```
   GET https://<TU-N8N>/webhook/revisar-stock?producto=Peluche%20Stitch
   ```
   (Con el workflow sin activar, usa la URL de prueba `/webhook-test/revisar-stock` y pulsa *Listen for test event*.)
5. Al terminar las pruebas, **desactiva el workflow**. No hay ejecuciones programadas: solo corre cuando alguien llama al Webhook.


## 6. Ejemplo reproducible

**Condición inicial:** el script SQL cargó `Peluche Stitch` con stock **3** y mínimo **10**; no existe una alerta del día para él.

**Entrada:** `GET /webhook/revisar-stock?producto=Peluche Stitch`

**Resultado esperado:**
- Respuesta HTTP 200 con `Estado: "Stock crítico"`, `Accion: "Reponer inmediatamente"`, `faltante: 7`, `prioridad: "Alta"`, `alerta_registrada: true`.
- **Supabase** → tabla `alertas_reposicion`: una fila con `alerta_id = <id>-<AAAA-MM-DD>`.
- **Gmail:** un correo con asunto y texto redactados por Gemini y una tabla con los datos reales.
- Si repites la llamada el mismo día, la fila se actualiza (no se duplica).

Otras entradas de los datos de ejemplo: `Camiseta Roja` (stock suficiente, sin alerta), `Producto Prueba Sin Stock` (stock nulo → "Datos de stock no válidos"), `Producto XYZ999` (→ "Producto no encontrado").

## 7. Documentación y evidencias

| Documento | Enlace |
|---|---|
| Documentación del proyecto | [documentacion/Documentacion_Proyecto_Final.md](Documentacion/Documentacion_Proyecto_Final.md) |
| Presentación | [presentacion/Presentacion_Proyecto_Final.pdf](Presentacion/Presentacion_Proyecto_Final.pdf) |
| Bitácora de pruebas (clase 11) | [pruebas/Bitacora_Clase_11_Pruebas.md](Pruebas/Bitacora_Clase_11_Pruebas.md) |
| Diagrama proceso manual | [documentacion/diagramas/Proceso_Manual.png](Documentacion/diagramas/Proceso_Manual.png) |
| Diagrama proceso automatizado | [documentacion/diagramas/Proceso_Automatizado.png](Documentacion/diagramas/Proceso_Automatizado.png) |

## 8. Estructura del repositorio

```
├── README.md
├── workflows/Workflow_Final.json
├── documentacion/
│   ├── Documentacion_Proyecto_Final.md
│   └── diagramas/{Proceso_Manual.png, Proceso_Automatizado.png}
├── pruebas/
│   ├── Bitacora_Clase_11_Pruebas.md
│   
└── presentacion/Presentacion_Proyecto_Final.pdf
```

## 9. Limitaciones conocidas

- Webhook sin autenticación.
- Una consulta vacía devuelve todo el inventario (prueba CP-02 fallida) y la búsqueda parcial (`ilike`) puede coincidir con varios productos.
- Si Gmail falla, la alerta queda guardada pero el correo no se reintenta.
- El texto de la IA se valida en formato, no en veracidad de cifras (la tabla del correo sí usa datos reales).
- Con stock = mínimo = 0 se genera una alerta con 0 faltantes.

Estado de cada pendiente en la [bitácora](Pruebas/Bitacora_Clase_11_Pruebas.md).

## 10. Seguridad

El repositorio no contiene claves, tokens ni datos reales. Los datos de ejemplo son ficticios.
