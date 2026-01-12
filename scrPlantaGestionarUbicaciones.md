# scrPlantaGestionarUbicaciones.md

## 1. Descripción General
Pantalla de gestión jerárquica de la planta física. Utiliza un diseño de "Maestro-Detalle" (Split View) donde la selección de un **Sector** (Padre) filtra dinámicamente sus **Líneas** (Hijos). Permite crear, editar y gestionar el ciclo de vida (Activo/Inactivo) de las ubicaciones.

* **ID Pantalla:** 2.2
* **Nombre:** `scrPlantaGestionarUbicaciones`
* **Fuentes de Datos:**
    * `spUbicaciones`: Tabla única que almacena la jerarquía (Sectores con `PadreID=0` y Líneas con `PadreID=ID_Sector`).

---

## 2. Variables de Contexto
Variables locales reinicializadas en `OnVisible` para controlar el flujo de la UI.

| Variable | Tipo | Descripción |
| :--- | :--- | :--- |
| `locMostrarFormulario` | Boolean | Visibilidad del Overlay de Edición (Crea/Edita). |
| `locMostrarConfirmacion` | Boolean | Visibilidad del Overlay de Alerta (Borrar/Restaurar). |
| `locMostrarInactivos` | Boolean | Filtro global para ver registros dados de baja (Soft Delete). |
| `locModoEditar` | Boolean | `true` si se está modificando un registro, `false` si es nuevo. |
| `locEsHijo` | Boolean | `true` si la operación es sobre una **Línea**; `false` si es sobre un **Sector**. |
| `locRegistroSeleccionado` | Record | Almacena el ítem sobre el cual se realiza la acción (Edición o Cambio de Estado). |

---

## 3. Estructura Visual (Tree View)

* **scrPlantaGestionarUbicaciones**
    * `cmpHeader...`, `cmpSidebar...`, `cmpHeaderSeccion...`
    * **conMain_Ubicaciones**
        * **conBarraSuperior_Ubicaciones**
            * `tglInactivos_Ubicaciones`: Toggle para mostrar/ocultar borrados.
            * `txtBuscador_Ubicaciones`: Buscador por nombre.
        * **conCuerpo_Ubicaciones** (Contenedor Horizontal Split)
            * **conColumnasSectores** (Izquierda - Padres)
                * `btnAgregar_Sector`: Crea un nuevo Padre.
                * **galSectores**: Lista de Sectores (`PadreID = 0`).
            * **conColumnaLineas** (Derecha - Hijos)
                * `btnAgregar_Linea`: Crea un hijo para el sector seleccionado.
                * **galLineas**: Lista de Líneas (`PadreID = galSectores.Selected.ID`).
    * **conOvrlayTotal_Ubicaciones**
        * **conTarjetaFormulario_Ubicaciones** (Formulario)
            * `txtNombre_Ubicaciones`
            * `btnGuardar...` / `btnCancelar...`
        * **conTarjetaConfirmacion_Ubicaciones** (Alerta)
            * `btnConfirmarBorrado...` (Acción Dual: Eliminar/Restaurar)

---

## 4. Lógica Clave

### A. Filtrado de Galerías
La lógica de filtrado maneja la jerarquía y el estado (Activo/Inactivo).

* **Sectores (`galSectores`):**
    Muestra registros raíz (`PadreID=0`).
    ```powerapps
    SortByColumns(
        If(locMostrarInactivos,
            Filter(spUbicaciones, StartsWith(Title, txtBuscador.Value), PadreID = 0),
            Filter(spUbicaciones, StartsWith(Title, txtBuscador.Value), PadreID = 0, Activo = true)
        ),
        "Title", SortOrder.Ascending
    )
    ```

* **Líneas (`galLineas`):**
    Muestra registros hijos del sector seleccionado.
    ```powerapps
    SortByColumns(
        Filter(spUbicaciones,
            PadreID = galSectores.Selected.ID,
            (Activo = true || locMostrarInactivos)
        ),
        "Title", SortOrder.Ascending
    )
    ```

### B. Lógica de Guardado (Formulario)
Un solo botón maneja la creación y edición tanto de Sectores como de Líneas, diferenciando la jerarquía mediante `locEsHijo`.

* **Botón Guardar (`OnSelect`):**
    ```powerapps
    Patch(spUbicaciones,
        If(locModoEditar, LookUp(spUbicaciones, ID = locRegistroSeleccionado.ID), Defaults(spUbicaciones)),
        {
            Title: txtNombre_Ubicaciones.Value,
            Activo: true,
            // CRÍTICO: Si es hijo, asigna el ID del sector seleccionado. Si es padre, 0.
            PadreID: If(locEsHijo, galSectores.Selected.ID, 0)
        }
    )
    ```

### C. Gestión de Estado (Borrar / Restaurar)
Se eliminó la lógica de "Hard Delete". El sistema funciona exclusivamente con **Soft Delete** reversible. El botón de confirmar actúa como un interruptor (Toggle).

* **Botón Confirmar (`OnSelect`):**
    ```powerapps
    Patch(spUbicaciones,
        LookUp(spUbicaciones, ID = locRegistroSeleccionado.ID),
        { Activo: !locRegistroSeleccionado.Activo } // Invierte el estado actual
    );
    Notify(If(locRegistroSeleccionado.Activo, "Desactivado", "Restaurado"), NotificationType.Success);
    ```

### D. Reglas de UX
1.  **Creación de Líneas:** El botón `btnAgregar_Linea` se deshabilita (`DisplayMode.Disabled`) si no hay un Sector seleccionado o si el Sector seleccionado está inactivo, previniendo la creación de huérfanos o datos inconsistentes.
2.  **Iconos Dinámicos:** Los iconos de acción en las galerías cambian según el estado:
    * Si `Activo = true` → Muestra `Icon.Trash` (Papelera) y `Icon.Edit` (Lápiz).
    * Si `Activo = false` → Muestra `Icon.Reload` (Restaurar) y oculta/bloquea la edición.
