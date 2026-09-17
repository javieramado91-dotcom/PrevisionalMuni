# SISTEMA PREVISIONAL — Contexto del proyecto

> Archivo ancla. Se carga al iniciar cada sesión. Sirve como punto de partida
> para retomar el trabajo sin perder el hilo entre sesiones.
> Última actualización: 2026-09-17.

---

## 1. Qué es

Aplicación web para la **gestión previsional (jubilaciones) de agentes municipales**.
Permite cargar agentes, calcular automáticamente su fecha estimada de jubilación,
darlos de baja o jubilarlos, registrar decretos e interrupciones de servicio, y
exportar reportes en PDF y Excel.

Es una app **sin backend propio**: todo corre en el navegador (HTML + JavaScript)
y los datos viven en la nube en **Firebase / Firestore** (base de datos de Google).

---

## 2. Estructura de archivos

```
SISTEMA PREVISIONAL/
├── index.html            → Login + tablero de novedades
├── fecha_condicion.html  → Núcleo del sistema (gestión de agentes)
├── firestore.rules       → Reglas de seguridad (se pegan en la consola de Firebase)
├── migrar_usuarios.html  → Herramienta de un solo uso (NO se sube al repo)
├── CLAUDE.md             → Este archivo (contexto)
└── .gitignore            → Excluye .claude/, migrar_usuarios.html y archivos del SO
```

Todo el código (HTML, CSS con Tailwind por CDN, y JS) está **dentro de cada .html**.
No hay archivos .js o .css separados, ni proceso de build.

---

## 3. Arquitectura y datos

- **Firebase project:** `previsional-muni`. La config está en el `<script>` de cada
  HTML (la apiKey de Firebase es pública del lado cliente, es normal).
- **Colecciones de Firestore:**
  - `agentes_jubilacion` → cada agente (legajo, nombre, cuil, fNac, fIng, estado, etc.).
  - `app_users` → **perfiles**. El id del documento es el UID de Firebase Auth y solo
    guarda `{ username, role }`. **Nunca contraseñas.**
  - `novedades` → tablero de novedades en index.html.
- **Tiempo real:** `fecha_condicion.html` usa `onSnapshot` sobre `agentes_jubilacion`,
  así que cualquier cambio se ve al instante en todas las pantallas conectadas.

### Autenticación (desde 2026-09-17)
- El login usa **Firebase Authentication** (email + contraseña). Las contraseñas las
  guarda Google cifradas: no están en Firestore ni las puede ver nadie.
- El usuario sigue escribiendo su **nombre** (ej: `YULI`). La función
  `usuarioAEmail()` lo convierte a `yuli@previsional-muni.web.app`.
  ⚠️ Esa función está duplicada en `index.html` y `migrar_usuarios.html`:
  **si se toca una, hay que tocar la otra igual**, o nadie puede entrar.
- **Doble condición para entrar:** estar logueado en Auth **y** tener un documento
  en `app_users/{uid}`. Solo un master puede crear ese documento. Así, si un
  desconocido se registra por su cuenta en Auth, queda sin perfil y no accede a nada.
- `role: 'master'` habilita la gestión de usuarios. Ya **no** existe el atajo
  "si se llama YULI es master".
- La sesión la maneja Firebase (`onAuthStateChanged`). Ya no se usa `localStorage`.
- `fecha_condicion.html` arranca el listener de agentes recién cuando la sesión
  está confirmada (`escucharAgentes()`).

### Reglas de seguridad
Están en `firestore.rules` y se aplican a mano en la consola de Firebase
(Firestore Database → Reglas → pegar → Publicar). Resumen:
- `agentes_jubilacion`: leer/escribir solo usuarios habilitados.
- `novedades`: todos los habilitados leen; editar/borrar solo el autor o un master.
- `app_users`: cada uno lee su perfil; crear/modificar/borrar solo el master.

⚠️ Si se borra por error el documento `app_users/{uid}` del master, nadie puede
volver a crear usuarios. Se arregla a mano desde la consola de Firebase.

### Cálculo previsional (función `calcularPrevision`)
No se guardan fechas de retiro calculadas: se **recalculan con la fecha de hoy**
cada vez. Reglas:
- Jubilación Ordinaria: 60 años de edad **y** 35 de aportes.
- Edad Avanzada: 65 años de edad **y** 10 de aportes.
- Se toma la fecha más conveniente para el agente.
- Suma los días de **interrupciones** a la antigüedad (servicio neto).
- Semáforo: 🔴 ≤2 años (o vencido) · 🟡 2–5 años · 🟢 >5 años.

### Estados de un agente
- **activo** → entra al cálculo del semáforo.
- **jubilado** → guarda `organismoJubilacion` y `fechaJubilacionReal`.
- **baja** → guarda `fechaBaja`, `motivoBaja`, `nroDecretoBaja`, `anioDecretoBaja`.

---

## 4. Funcionalidades clave

- **Tabla principal** con filtros (todos / rojo / jubilados / bajas / SAP / por tipo).
- **Ficha de detalle** por agente (botón 👁️).
- **Editar / Eliminar** agente.
- **Interrupciones de servicio** (historial): array `interrupciones` en el agente.
- **Decretos** (botón en cada fila → `modalDecretos`): array `decretos` en el agente.
  Cada decreto tiene `motivo` (alta / baja / recategorización / reubicación),
  `nroDecreto`, `anioDecreto` y campos específicos según el motivo.
- **Dar de Baja** (`abrirConfirmarBaja` → modal con fecha, motivo, N° y año de decreto).
- **Jubilar** (`abrirJubilar` → modal con selector de **Organismo Previsional**).
  Opciones del organismo: Dirección de Escuelas, ANSES, IPS, Mixta, **Cierre de Cómputos**.
- **Exportaciones:** nómina y listado de bajas (PDF con jsPDF / Excel con XLSX) e
  informe estadístico anual.

---

## 5. Git / GitHub (IMPORTANTE)

- **Repo (fuente de verdad):** https://github.com/javieramado91-dotcom/PrevisionalMuni (rama `main`).
- La carpeta local `C:\Users\pcjav\OneDrive\Escritorio\SISTEMAS\SISTEMA PREVISIONAL`
  está conectada al repo (conectada el 2026-06-19).
- **Flujo de trabajo:** editar local → `git add` → `git commit` → `git push origin main`.
- Autenticación: **Git Credential Manager** (login por navegador, ya quedó guardado).
- `gh` CLI **no** está instalado: usar `git` puro.
- git local config: user.email=javieramado91@gmail.com, user.name=javieramado91.

### ⚠️ Riesgo de divergencia (ya pasó una vez)
El usuario a veces edita el código **directo en la web de GitHub** (commits
"Add files via upload"). Por eso, **SIEMPRE hacer `git fetch` y comparar antes de
editar**, porque el repo puede estar adelantado respecto al local. En junio 2026 el
repo tenía la función de "Decretos" que el local no tenía, y casi se pisa.

---

## 6. ⚠️ Puesta en marcha de la seguridad (hacer UNA vez, en este orden)

El código nuevo ya está escrito, pero **no sirve hasta completar estos pasos**.
Si se publica el código sin hacer el paso A y B, nadie puede entrar al sistema.

| # | Paso | ¿Rompe algo si me detengo acá? |
|---|------|--------------------------------|
| A | Consola Firebase → **Authentication** → Sign-in method → **Email/Password** → Habilitar | No, el sitio viejo sigue funcionando |
| B | Abrir `migrar_usuarios.html` desde la PC → **Paso 1** y **Paso 2** (crea las cuentas) | No, el sitio viejo sigue funcionando |
| C | `git push` del código nuevo → probar el login en el sitio | Si falla, `git revert` y vuelve el anterior |
| D | Volver a `migrar_usuarios.html` → **Paso 3** (borra las contraseñas viejas) | Sí: ya no se puede volver atrás al login viejo |
| E | Consola Firebase → Firestore → **Reglas** → pegar `firestore.rules` → Publicar | Sí: cierra la base |
| F | Borrar `migrar_usuarios.html` de la carpeta | No |

Notas:
- Las contraseñas deben tener **6 caracteres como mínimo** (lo exige Firebase).
  La página de migración avisa y deja escribir una nueva donde haga falta.
- En el Paso 2 hay que marcar quién es **Master** (el que administra usuarios).
- Si se cierra la página entre el Paso 2 y el Paso 3, se puede volver a correr
  Paso 1 y Paso 2 sin problema: detecta las cuentas ya creadas.

---

## 7. Cómo retomar en una sesión nueva

1. Leer este `CLAUDE.md` y la carpeta de memoria del proyecto.
2. `git fetch origin && git status` para ver si el repo se adelantó.
   Si se adelantó: `git pull` (o `git reset --hard origin/main` si el local no tiene
   cambios propios) antes de tocar nada.
3. Hacer los cambios que pida el usuario en los .html.
4. Commit + push a `main`.
5. Si está activo GitHub Pages, el sitio se republica solo unos minutos después.

---

## 8. Historial de cambios hechos con Claude

- **Baja con decreto:** se agregaron campos N° de decreto y año al modal de baja;
  se guardan, se muestran en la tabla y salen en las exportaciones.
- **Jubilación con organismo:** el modal de jubilar ahora pide el Organismo
  Previsional (obligatorio); se guarda en `organismoJubilacion`, se muestra y exporta.
- **Conexión Git:** se conectó la carpeta local al repo de GitHub y se sincronizó
  con la versión completa del repo (que ya incluía Decretos).
- **`.gitignore`:** agregado para excluir `.claude/`.
- **"Cierre de Cómputos":** agregado como opción de organismo en el modal de jubilación.
- **Migración de seguridad (2026-09-17):** se pasó de un login casero (contraseñas
  en texto plano en Firestore, sesión falsificable desde el navegador) a Firebase
  Authentication + reglas de seguridad. Además: se sacó el master hardcodeado
  `YULI`, se dejó de guardar la contraseña en el navegador y en pantalla, se agregó
  botón de cerrar sesión en `fecha_condicion.html`, y se escapó el HTML de todos
  los datos cargados por el usuario (nombres, novedades, decretos, importación)
  para evitar inyección.

---

## 9. Preferencias del usuario

- Habla en **español**. Escribe a veces en MAYÚSCULAS (no es énfasis especial).
- No es programador: explicar en términos simples y concretos, sin jerga innecesaria.
- Quiere que los cambios se suban a GitHub automáticamente cuando los pide.
- Avisar en español cuando queden ~40.000 tokens o menos para abrir sesión nueva,
  guardando antes el contexto en este archivo y en la memoria.
