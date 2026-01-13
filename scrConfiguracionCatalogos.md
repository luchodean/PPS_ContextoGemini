# scrConfiguracionCatalogos.md

## 1. Descripción General
Pantalla de administración centralizada para las tablas auxiliares del sistema. Implementa un patrón de diseño **"Polimórfico"**, donde una sola interfaz se adapta dinámicamente para gestionar múltiples fuentes de datos (Tipos de Equipos, Especialidades, Proveedores) seleccionadas mediante un Dropdown principal.

* **ID Pantalla:** 7.1
* **Nombre:** `scrConfiguracionCatalogos`
* **Fuentes de Datos:**
    * `spCatalogosTiposEquipo` (Incluye lógica JSON)
    * `spCatalogoEspecialidades`
    * `spCatalogosProveedores`
    * `spUsuarios` (Contexto de seguridad)

---

## 2. Variables de Contexto
Variables locales reinicializadas en `OnVisible` y gestionadas durante la sesión.

| Variable | Tipo | Descripción |
| :--- | :--- | :--- |
| `locMostrarFormulario` | Boolean | Visibilidad del Overlay de Edición/Creación. |
| `locMostrarConfirmacion` | Boolean | Visibilidad del Overlay de Alerta (Borrado). |
| `locModoFormulario` | Text | Controla el comportamiento del form: `"Nuevo"` o `"Editar"`. |
| `locRegistroSeleccionado` | Record | Almacena el ítem seleccionado de cualquiera de las galerías activas. |
| `locGuardando` | Boolean | Bloqueo de UI durante operaciones asíncronas (`Patch`). |
| `colCamposConfig` | Colección | **CRÍTICO:** Buffer temporal para gestionar la estructura JSON de los campos personalizados de Tipos de Equipo. |

---

## 3. Estructura Visual (Tree View)

* **scrConfiguracionCatalogos**
    * `cmpHeader...`, `cmpSidebar...`, `cmpHeaderSeccion...`
    * **conMain** (AutoLayout Vertical)
        * **conBarraSuperior**
            * `ddSelectorCatalogo`: Dropdown maestro (Tipos de Equipos, Especialidades, Proveedores).
            * `btnNuevoRegistro`: Botón de acción contextual.
        * **conEncabezados**
            * `lblCampo1` a `lblCampo6`: Etiquetas con propiedad `Text` dinámica según `ddSelectorCatalogo`.
        * **conCuerpoTablas** (Contenedor de Galerías Superpuestas)
            * **galTiposEquipos**: Visible si `ddSelectorCatalogo = "Tipos de Equipos"`.
            * **galEspecialidades**: Visible si `ddSelectorCatalogo = "Especialidades"`.
            * **galProveedores**: Visible si `ddSelectorCatalogo = "Proveedores"`.
    * **conOverlayTotal**
        * **conTarjetaFormulario**
            * `lblTituloFormulario`
            * **conCuerpoFormulario**
                * `txtCampoFormulario1` a `txtCampoFormulario5`: Inputs genéricos que cambian su `Default` y `Visible` dinámicamente.
                * **conEncabezadosCamposPersonalizados** (Solo Tipos de Equipo)
                * **galEditorCampos** (Sub-galería para editar JSON)
                * `btnAgregarCampo` (Para nuevas definiciones técnicas)
            * **conBotones** (`btnGuardar`, `btnCancelar`)
        * **conTarjetaConfirmacion** (Soft Delete genérico)

---

## 4. Lógica Clave

### A. Polimorfismo de UI (Selector de Catálogo)
El control `ddSelectorCatalogo` actúa como el "Cerebro" de la pantalla. Todos los encabezados y visibilidad de galerías dependen de su valor.

**Ejemplo de Encabezado Dinámico (`lblCampo1`):**
```powerapps
Switch(
    ddSelectorCatalogo.Selected.Value,
    "Tipos de Equipos", "NOMBRE DEL EQUIPO",
    "Especialidades", "ESPECIALIDAD",
    "Proveedores", "RAZÓN SOCIAL",
    ""
)
```

### B. Gestión de Campos JSON (Tipos de Equipos)
Esta es la lógica más compleja de la pantalla. Permite definir qué datos técnicos se pedirán al crear un equipo (ej: Potencia, RPM).

#### 1. Carga (Al editar un Tipo de Equipo)
Se parsea el JSON almacenado en SharePoint hacia la colección local `colCamposConfig`.
```powerapps
// En OnSelect del icono editar
If(!IsBlank(ThisItem.CamposPersonalizados),
    ClearCollect(colCamposConfig,
        ForAll(Table(ParseJSON(ThisItem.CamposPersonalizados)),
            {
                _ID: Coalesce(Text(Value.ID), Text(GUID())), // Mantiene ID o genera nuevo
                Nombre: Text(Value.Nombre),
                Unidad: Text(Value.Unidad),
                Requerido: Boolean(Value.Requerido),
                Activo: Coalesce(Boolean(Value.Activo), true),
                _EsNuevo: false
            }
        )
    ),
    Clear(colCamposConfig) // Si es nuevo, inicia vacío
)
```

#### 2. Guardado (Serialización)
Al guardar el formulario maestro, se convierte la colección `colCamposConfig` de nuevo a texto JSON.
```powerapps
// Propiedad CamposPersonalizados en el Patch
JSON(
    ForAll(colCamposConfig,
    {
        ID: _ID,
        Nombre: Nombre,
        Unidad: Unidad,
        Requerido: Requerido,
        Activo: Activo
    }),
    JSONFormat.Compact
)
```

### C. Guardado Centralizado (btnGuardar)
Un solo botón maneja la creación y edición de las 3 tablas usando Switch.
```powerapps
Switch(ddSelectorCatalogo.Selected.Value,
    "Tipos de Equipos",
        Patch(spCatalogosTiposEquipo, 
            If(locModoFormulario="Nuevo", Defaults(...), locRegistroSeleccionado),
            { ... }
        ),
    "Especialidades",
        Patch(spCatalogoEspecialidades, ...),
    "Proveedores",
        Patch(spCatalogosProveedores, ...)
);
```

### D. Soft Delete Genérico
El overlay de confirmación adapta su lógica de borrado según la tabla activa.
```powerapps
// btnConfirmarBorrado
Switch(ddSelectorCatalogo.Selected.Value,
    "Tipos de Equipos", 
        Patch(spCatalogosTiposEquipo, LookUp(..., ID=locRegistroSeleccionado.ID), {Activo: false}),
    "Especialidades",
        Patch(spCatalogoEspecialidades, LookUp(..., ID=locRegistroSeleccionado.ID), {Activo: false}),
    ...
)
```

---

## 5. Observaciones de Implementación

- **Reutilización de Controles:** Se usan `txtCampoFormulario1...5` de forma genérica.
    - Caso Proveedores: Campo 1 = Razón Social, Campo 4 = Dirección.
    - Caso Tipos: Campo 1 = Título, Campo 4 = Oculto.
- **Validación Cruzada:** El botón Guardar se deshabilita si faltan datos en el encabezado O si hay errores en la sub-galería de campos personalizados (validación anidada).
- **Estilos:** Se respeta estrictamente `locTema_FondoPagina`, `locColorPrimario` y la paleta definida en MAESTRO_UI.
