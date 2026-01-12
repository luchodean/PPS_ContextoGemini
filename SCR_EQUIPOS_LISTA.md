# SCR_EQUIPOS_LISTA.md

## 1. Descripción General
Pantalla principal de navegación ("Browse Screen") para el inventario de equipos. Implementa filtrado avanzado, edición rápida de estados en línea y gestión de ciclo de vida (Alta, Baja, Modificación).

* **Nombre de Pantalla:** `scrEquiposLista`
* **Fuentes de Datos:**
    * `spEquipos` (Principal)
    * `spUbicaciones` (Para filtros y LookUps de sector)
* **Dependencias:**
    * Componentes: `cmpHeader`, `cmpSidebar`, `cmpHeaderSeccion`.
    * Navegación: `scrEquiposGestionarEquipo` (Wizard), `scrEquiposFichaIndividual` (Lectura).

---

## 2. Variables de Contexto
Variables locales utilizadas para gestionar el estado de la interfaz.

| Variable | Tipo | Alcance | Descripción |
| :--- | :--- | :--- | :--- |
| `locCargando` | Boolean | `OnVisible` | Controla el spinner de carga durante el `Refresh`. |
| `locGuardandoEstado` | Boolean | Galería | Bloquea el dropdown de estado mientras se ejecuta el Patch en línea. |
| `locMostrarConfirmacion` | Boolean | Overlay | Controla la visibilidad del modal de confirmación de borrado. |
| `locRegistroParaBorrar` | Record | Overlay | Almacena el registro seleccionado para aplicar el Soft Delete. |
| `locModoVista` | Text | Navegación | Define si el formulario destino se abre en `"Nuevo"` o `"Editar"`. |
| `locRegistroEditar` | Record | Navegación | Pasa el registro al formulario de edición. |

---

## 3. Estructura Visual (Tree View)

* **scrEquiposLista**
    * `cmpHeader_EquiposLista`
    * `cmpSidebar_EquiposLista`
    * `cmpHeaderSeccion_EquiposLista`
    * **conMain_EquiposLista** (AutoLayout Vertical)
        * **conBarraSuperior_EquiposLista** (ManualLayout)
            * `icnLupa_EquiposLista`
            * `txtBuscador_EquiposLista`
            * `ddFiltro_Sector` (Dropdown con opción "Todos" y "No Asignado")
            * `tglMostrarInactivos`
            * `btnNuevo_EquiposLista`
        * **conEncabezados_EquiposLista**
            * `lblEncabezado...` (Código, Equipo, Sector, Marca, Estado, Acción)
        * **galEquiposLista**
            * **conFila_EquiposLista**
                * `lblCampo...` (Title, Codigo, Sector, Marca)
                * **conEstado_Galeria**
                    * `shpEstado_Indicador` (Círculo de color dinámico)
                    * `ddEstado_Galeria` (Cambio de estado rápido)
                * **conAcciones_EquiposLista**
                    * `icnDetalles_ListaEquipos` (Ver)
                    * `icnEditar_ListaEquipos` (Editar)
                    * `icnBorrar_ListaEquipos` (Papelera/Restaurar)
    * **conOverlayTotal_EquiposLista**
        * **conTarjetaConfirmacion_EquiposLista**
            * `btnConfirmarBorrado_EquiposLista`
            * `btnCancelarBorrado_EquiposLista`

---

## 4. Lógica y Fórmulas Clave

### A. Inicialización (`OnVisible`)
Se actualiza la navegación y se refrescan las fuentes de datos forzando un estado de carga.
```powerapps
Set(varActiveId, LookUp(colNav, Screen = App.ActiveScreen).Id);
UpdateContext({ locCargando: true });
Refresh(spUbicaciones);
Refresh(spEquipos); 
UpdateContext({ locCargando: false });
```

### B. Barra Superior y Filtros

#### Dropdown de Sectores (ddFiltro_Sector)
Construcción manual de una tabla para incluir opciones de sistema ("Todos", "No Asignado") junto con los datos de SharePoint.

**Items:**
```powerapps
Ungroup(
    Table(
        { _registros: Table({ Title: "Todos los Sectores", ID: -1 }) },
        { _registros: Table({ Title: "No Asignado", ID: 0 }) },
        { _registros: ShowColumns(Filter(spUbicaciones, PadreID = 0, Activo = true), Title, ID) }
    ), 
    _registros
)
```

### C. Galería Principal (galEquiposLista)

#### Fórmula de Filtrado (Items)
Optimizado para delegación usando IDs numéricos en lugar de nombres para el filtro de sectores.
```powerapps
SortByColumns(
    Filter(spEquipos,
        // 1. Toggle de Inactivos
        (tglMostrarInactivos.Checked || Activo = true) &&

        // 2. Filtro por Sector (Delegable usando ID)
        (
            ddFiltro_Sector.Selected.ID = -1 || 
            SectorID = ddFiltro_Sector.Selected.ID
        ) &&

        // 3. Buscador Multicriterio
        (
            StartsWith(Title, txtBuscador_EquiposLista.Value) || 
            StartsWith(CodigoEquipo, txtBuscador_EquiposLista.Value) || 
            StartsWith(Marca, txtBuscador_EquiposLista.Value)
        ) &&
        // 4. Excluir borradores
        (EsBorrador = false)
    ),
    "Nombre",
    SortOrder.Ascending
)
```

#### Edición de Estado en Línea (ddEstado_Galeria)
Permite cambiar el estado operativo sin entrar al formulario.

**OnChange:**
```powerapps
UpdateContext({ locGuardandoEstado: true });
Patch(spEquipos, LookUp(spEquipos, ID = ThisItem.ID), { EstadoOperativo: Self.Selected.Value });
UpdateContext({ locGuardandoEstado: false });
```

**Visual (shpEstado_Indicador):** Cambia de color según el valor (Verde, Advertencia, Peligro) usando variables globales (locColorExito, etc.).

### D. Acciones de Fila

**Nuevo Equipo:**
- Navega con `locModoVista: "Nuevo"` y `locRegistroEditar: Blank()`.

**Editar:**
- Navega con `locModoVista: "Editar"` y `locRegistroEditar: ThisItem`.

**Borrar / Restaurar (icnBorrar_ListaEquipos):**
- **Icono:** Alterna entre `Icon.Trash` (Si está activo) e `Icon.Reload` (Si está inactivo).
- **Lógica:**
    - Si Activo: Muestra Overlay de confirmación.
    - Si Inactivo: Ejecuta restauración inmediata (`Patch(..., {Activo: true})`).

### E. Overlay de Confirmación (Soft Delete)
Maneja el borrado lógico seguro.

**Botón Confirmar (OnSelect):**
```powerapps
Patch(
    spEquipos, 
    LookUp(spEquipos, ID = locRegistroParaBorrar.ID), 
    { 
        Activo: false,
        EstadoOperativo: "Baja"
    }
);
Notify("El equipo ha sido dado de baja correctamente.", NotificationType.Success);
UpdateContext({ locMostrarConfirmacion: false, locRegistroParaBorrar: Blank() });
```

---

## 5. Observaciones de Implementación

- **Inputs:** Se utilizan controles Modernos (TextInput@0.0.54).
- **Dropdowns:** Se utilizan controles Clásicos (DropDown@0.0.45) para personalización de estilos.
- **LookUps:** El nombre del sector en la galería se resuelve con `Coalesce(LookUp(...).Title, "No Asignado")` para manejar integridad referencial.
- **Estilos:** El botón "Nuevo" utiliza `Color.Green` directamente, desviándose ligeramente de `locTema_AccionPrimaria` (Naranja), presumiblemente para destacar la acción positiva de creación.
