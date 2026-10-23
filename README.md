# deviserver-builder

Este repositorio **publico** es el "taller" de Deviserver: cuando un usuario pega la liga de una app Flutter que no trae su build web,
tu servidor le pide a GitHub (gratis, GitHub Actions) que la compile y despues descarga solo el resultado. Tu servidor no compila nada.

Contenido: `.github/workflows/build.yml` (el unico archivo que hace falta).

## ADVERTENCIA: usa una cuenta u organizacion de GitHub DEDICADA

Este repositorio ejecuta, en los servidores de GitHub, **codigo que escriben terceros** (las apps que tus usuarios piden compilar). Por eso:

- **Usa una cuenta u organizacion de GitHub DEDICADA solo a `deviserver-builder`**, no tu cuenta personal ni la de tus otros proyectos. Si alguien abusa
  del taller (mineria de criptomonedas, ataques, spam) GitHub puede **suspender la cuenta completa**; que sea una cuenta que puedas perder.
- **Vigila el uso y los abusos**: revisa de vez en cuando la pestana *Actions* del repositorio (quien compila, cuanto dura, ejecuciones raras), tu
  facturacion/minutos incluidos (*Settings -> Billing*) y el correo de GitHub. Activa los avisos de uso. Si ves algo raro: revoca el token y desactiva Actions.
- **El repositorio NO debe tener secretos, variables, environments ni runners propios (self-hosted)**. Ni siquiera "de prueba": el codigo ajeno corre ahi.
  En *Settings -> Secrets and variables* y *Environments* no debe haber nada; en *Actions -> Runners* tampoco. El unico secreto vive en tu servidor.
- **Activa la verificacion en dos pasos (2FA)** en esa cuenta (y en las de cualquier miembro de la organizacion). Usa una contrasena unica.
- El token del servidor es de **un solo repositorio** y solo con *Actions: Read and write* (paso 4); no lo amplies.

## Configuracion (una sola vez, unos 10 minutos)

1. **Activa la verificacion en dos pasos (2FA)** en tu cuenta de GitHub: *Settings -> Password and authentication -> Two-factor authentication*.
2. **Crea el repositorio**: en GitHub, *New repository* -> nombre `deviserver-builder` -> **Public** -> sin README. Sube el contenido de esta carpeta
   (que quede `.github/workflows/build.yml` en la raiz del repositorio, rama `main`).
3. **Ajusta Actions** del repositorio (*Settings -> Actions -> General*):
   - *Actions permissions*: **Allow {tu usuario}, and select non-{tu usuario}, actions**, marca solo "Allow actions created by GitHub",
     deja vacia la lista de acciones extra y marca "Require actions to be pinned to a full-length commit SHA". El flujo no usa
     ninguna accion de terceros: Flutter se baja del archivo oficial de Google y se verifica con su SHA-256.
   - *Workflow permissions*: **Read repository contents and packages permissions** (solo lectura).
   - *Fork pull request workflows*: deja todo en lo mas estricto ("Require approval for all outside collaborators").
4. **Crea un token de acceso fino (fine-grained) SOLO para este repositorio**: *Settings (tu cuenta) -> Developer settings -> Personal access tokens ->
   Fine-grained tokens -> Generate new token*.
   - *Repository access*: **Only select repositories** -> `deviserver-builder`.
   - *Permissions -> Repository permissions*: **Actions: Read and write**. Nada mas (Metadata: Read-only se agrega solo).
     Segun la documentacion de GitHub, "Create a workflow dispatch event" exige *Actions: write* y listar ejecuciones/artefactos exige *Actions: read*.
   - *Expiration*: 90 dias (anota la fecha: cuando venza, repite este paso y pega el nuevo token).
   - Copia el token (empieza con `github_pat_`). **No lo pegues en ningun chat, correo ni repositorio.**
5. **Pegalo en el servidor** (el token queda solo en ese archivo, legible por root y por el servidor web):
   ```
   read -rsp "Token: " T; echo
   sudo install -o root -g www-data -m 640 /dev/null /etc/deviserver/github_builder.json
   sudo tee /etc/deviserver/github_builder.json >/dev/null <<EOT
   {"owner": "TU_USUARIO_DE_GITHUB", "repo": "deviserver-builder", "token": "$T", "workflow": "build.yml", "ref": "main"}
   EOT
   unset T
   ```
   Opcional: `"cuota_diaria": 5` (compilaciones por usuario cada 24 h), `"cuota_global_diaria": 25` (de todo el servidor cada 24 h)
   y `"exigir_digest": true`.
6. **Prueba**: en el panel de un usuario, pega `https://github.com/sluna5-coder/rick-morty.git` -> debe aparecer "Compilando en GitHub..." y a los
   3-6 minutos el boton **Abrir app**.

## Que hace el flujo (y por que es seguro)

- Dos jobs. `compilar` es el unico que toca codigo ajeno: sin secretos, sin permisos (`permissions: {}`), sin cache, sin credenciales (solo clona repos publicos),
  maximo 15 minutos. `certificar` no ejecuta nada del repositorio: revisa el paquete, recalcula el SHA-256 y publica el artefacto final `dvs-<id>`.
- Solo se usan acciones de GitHub (subir/bajar artefactos), fijadas por SHA de commit completo. Flutter se instala desde el archivo oficial
  de Google (indice `releases_linux.json`, canal stable) y se comprueba su SHA-256 antes de usarlo.
- La URL solo puede ser `https://github.com/<dueño>/<repo>` y el `job_id` 32 caracteres hexadecimales; se validan antes de hacer nada.
- Se elige una sola app: la que indique `deviserver.json` (`{"web": "apps/x/web_build"}`), si no la de la raiz, si no la primera que tenga carpeta `web/`.
- El resultado se limita a 60 MB / 5000 archivos y no puede contener enlaces simbolicos, archivos ocultos, mapas de codigo ni archivos de servidor (php, sh...).
- Los nombres de archivo no pueden llevar espacios (ni ningun espacio Unicode), caracteres de control o de formato Unicode, terminar en punto, usar puntos de ancho completo, estar fuera de la forma NFC ni terminar en `.php`, `.pht`, `.phtm` y similares (la misma regla la aplican el flujo y el servidor).
- Los artefactos duran 1 dia.

## Problemas comunes

| Mensaje en el panel | Que hacer |
|---|---|
| "El servicio de compilacion no esta disponible" | El token vencio o le falta el permiso *Actions: Read and write*; revisa `/var/log/deviserver/app.log`. |
| "GitHub no pudo compilar tu app" | Mira el enlace "Ver registro". Suele ser un error de la propia app o falta la carpeta `web/` (`flutter create . --platforms web`). |
| "Hay una app Flutter pero no tiene carpeta web/" | En el repo de la app: `flutter create . --platforms web` y `git push`. |
