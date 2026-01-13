# scrConfiguracionUsuariosYPermisos.md

## 1. Descripción General
Pantalla de administración de seguridad del sistema. Funciona como un panel de control dual que permite gestionar:
1. **Usuarios:** Altas, bajas (Soft Delete) y asignación de roles.
2. **Roles y Permisos:** Definición granular de accesos mediante una matriz de toggles que se serializa en un objeto JSON.

* **ID Pantalla:** 7.2
* **Nombre:** `scrConfiguracionUsuariosYPermisos`
* **Fuentes de Datos:**
    * `spUsuarios` (Directorio de personas y sus roles)
    * `spRolesDefinicion` (Catálogo de perfiles y reglas de seguridad JSON)

---

## 2. Variables de Contexto
Variables locales reinicializadas en `OnVisible` para controlar el estado de la interfaz.

| Variable | Tipo | Descripción |
| :--- | :--- | :--- |
| `locVistaActual` | Text | Controla la pestaña activa: `"USUARIOS"` o `"ROLES"`. |
| `locMostrarFormulario` | Boolean | Visibilidad del Overlay de Edición. |
| `locModoFormulario` | Text | Define el contexto del popup: `"Nuevo"`, `"Editar"` (Usuario) o `"Permisos"` (Rol). |
| `locRegistroSeleccionado` | Record | Almacena el Usuario o Rol en edición. |
| `colPermisosEditor` | Colección | **CRÍTICO:** Buffer temporal para manipular los toggles de permisos antes de serializarlos a JSON. |
| `locJSONActual` | Object | Almacena el objeto `ParseJSON` del rol seleccionado para poblar la colección inicial. |

---

## 3. Estructura Visual (Tree View)

* **scrConfiguracionUsuariosYPermisos**
    * `cmpHeader...`, `cmpSidebar...`, `cmpHeaderSeccion...`
    * **conMain_Usuarios**
        * **conBarraSuperior_Usuarios**
            * `txtBuscador_Usuarios`: Búsqueda predictiva (Nombre/Email o Rol).
            * `btnNuevo_Usuarios`: Visible solo en vista "USUARIOS".
            * `tglInactivos_Usuarios`: Filtro de Soft Delete.
            * **conTabs_Capsula** (Selector de Vista)
                * `btnTab_Usuarios` / `btnTab_Roles`: Botones con lógica de estilo condicional.
        * **conEncabezados_Usuarios**: Etiquetas dinámicas según `locVistaActual`.
        * **conCuerpoTabla_Usuarios**
            * **galUsuarios** (Visible si `locVistaActual="USUARIOS"`)
                * Acciones: Editar (Lápiz) y Soft Delete (Papelera/Restaurar).
            * **galRoles** (Visible si `locVistaActual="ROLES"`)
                * Acciones: Configurar Permisos (Lápiz). Bloqueado para "Administrador".
    * **conOverlayTotal_Usuarios**
        * **conTarjetaFormulario_Usuarios**
            * `lblTitulo...`: Título dinámico.
            * **conCuerpoFormulario_Usuarios** (Inputs Clásicos)
                * Nombre, Email, Dropdown de Rol (`spRolesDefinicion`).
            * **conCuerpoFormulario_Roles** (Editor de JSON)
                * Nombre, Descripción.
                * **galEditorPermisos**: Lista vertical de toggles generada desde colección.
            * `btnGuardar...` / `btnCancelar...`
        * **conTarjetaConfirmacion_Usuarios** (Alerta de baja lógica).

---

## 4. Lógica Clave

### A. Sistema de Pestañas (Tabs)
La navegación interna no cambia de pantalla, solo actualiza variables de visibilidad y estilos.

**Estilo Botón Activo (Fill):**
```powerapps
If(locVistaActual = "USUARIOS", locTema_AccionPrimaria, Color.Transparent)
```

### B. Motor de Permisos (JSON Parsing & Serialización)
Esta es la lógica más compleja de la pantalla. Transforma un texto JSON plano en una interfaz gráfica de toggles y viceversa.

#### 1. Lectura (Al editar un Rol)
Se parsea el JSON almacenado y se construye una colección tipada (`colPermisosEditor`) que agrupa los permisos por módulos (Activos, Planes, OTs).
```powerapps
// OnSelect del Icono Editar en galRoles
UpdateContext({ locJSONActual: ParseJSON(ThisItem.PermisosJSON) });
ClearCollect(
    colPermisosEditor,
    { Key: "Activos_Ver", Modulo: "Activos", Titulo: "Ver Planta", Valor: Boolean(locJSONActual.Activos_Ver) },
    { Key: "OT_Crear",    Modulo: "OTs",     Titulo: "Crear OTs",  Valor: Boolean(locJSONActual.OT_Crear) },
    // ... resto de definiciones
);
```

#### 2. Escritura (Al Guardar Rol)
Se reconstruye el string JSON concatenando los valores de la colección temporal.
```powerapps
// btnGuardar_UsuariosRoles
With({
    jsonBody: Concat(
        colPermisosEditor,
        Char(34) & Key & Char(34) & ":" & If(Valor, "true", "false") & ","
    )
},
    Patch(spRolesDefinicion, LookUp(...), {
        PermisosJSON: "{" & Left(jsonBody, Len(jsonBody)-1) & "}" // Elimina última coma y cierra llaves
    })
)
```

### C. Gestión de Usuarios (Guardado Dual)
El botón de guardar decide inteligentemente qué fuente de datos actualizar basándose en la pestaña activa.
```powerapps
If(locVistaActual = "USUARIOS",
    Patch(spUsuarios, ..., {
        Title: txtNombre.Value,
        Rol: ddRol.Selected.Title // Guarda el texto del rol para visualización rápida
    }),
    // Else: Lógica de Roles (ver arriba)
)
```

### D. Seguridad de Integridad

- **Protección del Administrador:** El rol "Administrador" tiene el botón de edición deshabilitado (`DisplayMode.Disabled`) para evitar que se bloquee a sí mismo o se degraden permisos críticos accidentalmente.
- **Soft Delete (Usuarios):** Solo aplica a usuarios. Los roles no se borran, solo se editan, para no romper la integridad referencial de los usuarios asignados.

---

## 5. Observaciones

- **Normalización:** Aunque `spUsuarios` guarda el nombre del Rol, la relación real de permisos se resuelve en tiempo de ejecución (`App.OnStart`) buscando en `spRolesDefinicion` basada en ese nombre.
- **UX/UI:** El editor de permisos utiliza un diseño de lista flexible (`galEditorPermisos`) que permite agregar nuevos toggles en el futuro (ej: "Reportes_Ver") simplemente agregando una línea en la fórmula del `OnSelect`, sin rediseñar el formulario visualmente.
