# CAS-MAP

Portafolio digital para documentar **CAS** (Creativity, Activity, Service) del
Bachillerato Internacional: experiencias, el Project, evidencia multimedia y
seguimiento de los 7 Learning Outcomes.

## Stack

- **Frontend:** sitio estático, sin build step. Todo el HTML/CSS/JS vive inline.
  - `index.html` — app del estudiante (auth, CRUD, evidencia, progreso, temas)
  - `teacher.html` — vista de solo lectura para el profesor vía link con token
  - `style.css` — hoja alternativa (no enlazada desde `index.html`)
- **Librerías por CDN (jsdelivr):** `@supabase/supabase-js`, `quill`, `dompurify`
- **Backend:** Supabase
  - tabla `cas_records` (`record_type`: `experience` | `project`, `content` JSON, RLS por `user_id`)
  - tabla `cas_teacher_shares` (hash SHA-256 del token del link del profesor)
  - storage bucket `cas-evidence` (privado, se sirve con signed URLs de 1 h)
  - edge function `teacher-view` (valida el token y devuelve el portafolio en solo lectura)
- **Hosting:** Netlify — <https://ibdpcasmap.netlify.app>
  (espejo en GitHub Pages: <https://diegogi19.github.io/CAS-MAP/>)

## Desarrollo local

No hay dependencias que instalar. Servir la carpeta por HTTP (no `file://`,
porque Supabase Auth necesita un origin real):

```bash
python3 -m http.server 4173
# http://localhost:4173/index.html
```

## Deploy

No hay comando de build: `netlify.toml` publica la raíz del repo y añade los
headers de caché y seguridad (CSP incluida).

El sitio se creó con **drag & drop**, así que hoy el deploy **no** es
continuo: un push a `main` no actualiza producción. Para activarlo, en Netlify
→ *Site configuration → Build & deploy → Continuous deployment → Link
repository* → `diegogi19/CAS-MAP`, branch `main`, build command vacío, publish
`.`. Hasta entonces, cada cambio hay que volver a arrastrar la carpeta — y
`netlify.toml` sólo se aplica si va incluida en lo que se arrastra.

### Checklist al cambiar de dominio

1. **Supabase → Authentication → URL Configuration**
   - `Site URL`: `https://ibdpcasmap.netlify.app` — es el fallback de los
     correos de Supabase.
   - `Redirect URLs`: `https://ibdpcasmap.netlify.app/**`,
     `https://*--ibdpcasmap.netlify.app/**` (deploy previews),
     `https://diegogi19.github.io/CAS-MAP/**` (mientras Pages siga vivo) y
     `http://localhost:4173/**`. Supabase valida contra esta lista tanto el
     `emailRedirectTo` del signup como el `redirectTo` del reset de contraseña;
     ambos usan `location.origin + location.pathname`, así que el correo vuelve
     al origin desde el que se registró el alumno.
2. **`netlify.toml`** — si cambia el proyecto de Supabase, actualizar el host
   en la `Content-Security-Policy` (`connect-src`, `img-src`, `media-src`).
3. Probar: signup → confirmación por correo → login → subir evidencia →
   generar link de Teacher Access → abrirlo en una ventana privada.

## Notas

- `SUPABASE_PUBLISHABLE_KEY` está hardcodeada en el cliente a propósito: es una
  clave publicable, no un secreto. La seguridad real son las políticas RLS.
  Verificado: `anon` no puede insertar en `cas_records` ni leer `cas_teacher_shares`.
- El schema SQL y la edge function `teacher-view` **no están versionados acá**:
  viven solo en el proyecto de Supabase. Conviene exportarlos a `supabase/` para
  poder reconstruir el backend.
