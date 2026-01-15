# scrEquiposFichaIndividual.md

## 1. Descripción General
Pantalla de detalle ("Hoja de Vida") del equipo. Proporciona una vista de lectura optimizada y segmentada por pestañas para consultar la identidad del activo, sus especificaciones técnicas, la red de proveedores vinculados y la documentación adjunta.

* **ID Pantalla:** 3.2
* **Nombre:** `scrEquiposFichaIndividual`
* **Fuentes de Datos:**
    * `spEquipos` (Registro principal).
    * `spUbicaciones` (Resolución de nombres Sector/Línea).
    * `spCatalogosTiposEquipo` (Resolución nombre Tipo).
    * `spEquiposProveedores` (Tabla intermedia de relaciones).
    * `spCatalogosProveedores` (Datos enriquecidos de contacto).

---

## 2. Variables de Contexto
Variables inicializadas en `OnVisible` para gestionar el estado y la caché de datos.

| Variable | Tipo | Descripción |
| :--- | :--- | :--- |
| `locRegistroFicha` | Record | Registro fresco del equipo obtenido desde SharePoint (`LookUp` por ID). |
| `locTabFicha` | Text | Controla la pestaña visible: `"RESUMEN"`, `"RELACIONES"`, `"ADJUNTOS"`. |
| `colFicha_Tecnica` | Colección | Tabla temporal parseada desde el JSON `DatosTecnicosValores`. |
| `colFicha_Proveedores` | Colección | Tabla enriquecida ("Flattened") con datos de contacto de proveedores asignados. |
| `varNombreSector` | Text | Nombre resuelto del Sector (Caché local). |
| `varNombreLinea` | Text | Nombre resuelto de la Línea (Caché local). |
| `varNombreTipo` | Text | Nombre resuelto del Tipo de Equipo. |

---

## 3. Estructura Visual (Tree View)

* **scrEquiposFichaIndividual**
    * `cmpHeader...`, `cmpSidebar...`, `cmpHeaderSeccion...`
    * **conMain_Ficha** (AutoLayout Vertical)
        * **conBarraSuperior_Ficha**
            * **Navegación:** `btnTab_Resumen`, `btnTab_Relaciones`, `btnTab_Adjuntos`.
            * **Acciones:** `btnEditar_Ficha` (Navega al Wizard en modo Editar).
        * **conBody_Resumen_Ficha** (Visible: `"RESUMEN"`)
            * **conColumna_General** (Izquierda)
                * Filas estáticas clave-valor (Código, Sector, Marca, Estado).
            * **conColumna_Tecnica** (Derecha)
                * `galResumen_Tecnico`: Lista dinámica de especificaciones.
        * **conBody_Proveedores_Ficha** (Visible: `"RELACIONES"`)
            * `galFicha_Proveedores`: Tabla nativa con columnas (Rol, Proveedor, Contacto, Teléfono).
        * **conBody_Adjuntos_Ficha** (Visible: `"ADJUNTOS"`)
            * `galFicha_Adjuntos`: Grid de tarjetas (WrapCount 4) con vistas previas de imágenes e iconos de documentos.

---

## 4. Lógica Clave

### A. Optimización de Carga (`OnVisible`)
Se implementa el patrón **"Single Request Cache"** usando `With()` para agrupar llamadas a tablas maestras y evitar latencia.
```powerapps
// Agrupamos la búsqueda de Sector y Línea en un solo filtro para evitar 2 llamadas separadas
With(
    {
        _cacheUbicaciones: Filter(spUbicaciones, ID = locRegistroFicha.SectorID || ID = locRegistroFicha.LineaID),
        _tipo: LookUp(spCatalogosTiposEquipo, ID = locRegistroFicha.TipoEquipoID)
    },
    UpdateContext({
        varNombreSector: Coalesce(LookUp(_cacheUbicaciones, ID = locRegistroFicha.SectorID).Title, "No asignado"),
        varNombreLinea: Coalesce(LookUp(_cacheUbicaciones, ID = locRegistroFicha.LineaID).Title, "No asignado"),
        varNombreTipo: _tipo.Title
    })
);
```

### B. Enriquecimiento de Proveedores
Transformación de la relación N:N (`spEquiposProveedores`) en una tabla plana para visualización, trayendo datos del maestro `spCatalogosProveedores` en una sola iteración eficiente.
```powerapps
ClearCollect(colFicha_Proveedores,
    ForAll(
        Filter(spEquiposProveedores, EquipoID = locRegistroFicha.ID && Activo = true),
        With(
            { _prov: LookUp(spCatalogosProveedores, ID = ProveedorID) }, // LookUp único por fila
            {
                NombreProveedor: _prov.Title,
                Rol: TipoRelacion,
                Telefono: _prov.Telefono,
                Direccion: _prov.Direccion
            }
        )
    )
);
```

### C. Sistema de Adjuntos (URL Absoluta)
Para abrir archivos adjuntos en una nueva pestaña del navegador, se construye manualmente la URL absoluta de SharePoint, ya que `Launch(ThisItem.Value)` falla con rutas internas.

**Fórmula del Botón Overlay (OnSelect):**
```powerapps
Launch(
    "https://excesercomar.sharepoint.com/sites/Mantenimiento/Lists/spEquipos/Attachments/" & 
    locRegistroFicha.ID & "/" & 
    EncodeUrl(ThisItem.DisplayName), // Codificación para caracteres especiales
    {},
    LaunchTarget.New // Forza nueva pestaña
)
```

---

## 5. Observaciones de Diseño UI

- **Tablas Nativas:** La pestaña de Proveedores implementa una galería con cabeceras (`conEncabezados_Proveedores`) alineadas pixel-perfect con el cuerpo (`galFicha_Proveedores`) para simular un control DataGrid ligero y rápido.
- **Grid de Adjuntos:** Utiliza una galería vertical con `WrapCount: 4` y controles superpuestos (Z-Index) para crear tarjetas interactivas:
    - Fondo: Imagen (`ImagePosition.Center`) o Icono según extensión.
    - Overlay: Botón transparente que captura el clic en toda el área.
    - Pie: Label semitransparente para legibilidad del nombre del archivo.
- **Estado Operativo:** Se muestra como un "Badge" (Botón en modo View con radio completo) para destacar visualmente el estado (Operativo/Mantenimiento) sin usar texto plano.
