# Guía de despliegue en OCI

Paso a paso para crear los ambientes de **backend** y **frontend** de Community IA en Oracle Cloud Infrastructure (capa Always Free). La arquitectura original está en [Documentacion_Equipo_D_y_Despliegue_OCI.md](Documentacion_Equipo_D_y_Despliegue_OCI.md).

- Las opciones marcadas como **[recomendado]** mejoran la seguridad respecto del despliegue original.
- El análisis de vulnerabilidades está en [Seguridad del despliegue](#seguridad-del-despliegue). Los pasos de esta guía ya incorporan sus correcciones.
- Si ya tienes una versión anterior en marcha, ve a [Actualizar un despliegue existente](#actualizar-un-despliegue-existente).

## Resumen de pasos

| Paso                                                            | Dónde                         | Resultado                                                       |
|:--------------------------------------------------------------- |:----------------------------- |:--------------------------------------------------------------- |
| [0. Requisitos](#paso-0-requisitos-previos)                     | Tu equipo y la consola de OCI | Claves SSH y de Gemini; la clave de API filtrada, revocada      |
| [1. Compartimento y permisos](#paso-1-compartimento-y-permisos) | Consola de OCI                | Compartimento y forma de autenticar el backend contra el bucket |
| [2. Red](#paso-2-red)                                           | Consola de OCI                | VCN y reglas de entrada restringidas                            |
| [3. Bucket](#paso-3-bucket)                                     | Consola de OCI                | Bucket privado para los paquetes publicados                     |
| [4. Instancias](#paso-4-instancias)                             | Consola de OCI                | Dos VM Always Free                                              |
| [5. Preparar cada VM](#paso-5-preparar-cada-vm)                 | Las dos VM                    | Docker, swap, parches automáticos y el código                   |
| [6. Backend](#paso-6-backend)                                   | VM backend                    | API en marcha, conectada a OCI, con el primer admin             |
| [7. Frontend](#paso-7-frontend)                                 | VM frontend                   | Aplicación web conectada al backend                             |
| [8. Prueba de punta a punta](#paso-8-prueba-de-punta-a-punta)   | Navegador                     | Un lote revisado por el equipo y publicado en el bucket         |
| [9. Operación](#paso-9-operación)                               | Las dos VM                    | Actualizaciones, logs y copias de seguridad programadas         |

## Arquitectura

```
Internet ──:3000──► VM frontend (VM.Standard.E2.1.Micro, Oracle Linux 9)
                     └ Docker: Astro (Node 22) + BFF /api/*
                          │ HTTP :8000 (BACKEND_URL), con el token de la sesión
                          ▼
                    VM backend (VM.Standard.E2.1.Micro, Oracle Linux 9) ──HTTPS──► API de Gemini
                     └ Docker: FastAPI + LangGraph
                        ├ volumen backend-data: auth.db (cuentas) y app.db (lotes y revisión)
                        │ SDK de OCI
                        ▼
                    Object Storage: bucket communitylab-activos-marketing (privado)
```

- El navegador solo habla con el frontend. El frontend guarda la sesión en una cookie `httpOnly` y reenvía las llamadas `/api/*` al backend con el token (patrón BFF), así que no hay problemas de CORS.
- El backend analiza con Gemini. El trabajo en curso del equipo (cuentas, lotes, contenidos en revisión y quién decidió qué) vive en dos bases SQLite dentro del volumen `backend-data`. Ver [Lotes, revisión en equipo y datos](#lotes-revisión-en-equipo-y-datos).
- Cuando un admin publica un lote revisado, el backend sube el paquete final al bucket. El bucket es el historial definitivo que leen la **Tabla de casos** y el **Dashboard**.
- Región documentada: `sa-vinhedo-1` (Brasil Este - Vinhedo).

| Recurso            | Nombre sugerido                  | Notas                                                                  |
|:------------------ |:-------------------------------- |:---------------------------------------------------------------------- |
| Compartimento      | `communityia`                    | Agrupa todos los recursos                                              |
| VCN                | `communityia-vcn`                | Creada con el asistente *VCN with Internet Connectivity*               |
| Instancia frontend | `communityia-frontend`           | Puerto 3000                                                            |
| Instancia backend  | `communityia-backend`            | Puerto 8000                                                            |
| Bucket             | `communitylab-activos-marketing` | Nombre actual del bucket; se conserva para no romper el almacenamiento |

> La capa Always Free permite **dos** instancias `VM.Standard.E2.1.Micro` por tenancy. Esta arquitectura usa las dos.

---

## Paso 0. Requisitos previos

- [ ] Cuenta de OCI con Always Free y región principal `sa-vinhedo-1`.
- [ ] Un par de claves SSH (`ssh-keygen -t ed25519`).
- [ ] Una clave de Google Gemini (`GEMINI_API_KEY`).
- [ ] La URL del repositorio.
- [ ] **Revocar la clave de API filtrada, antes que nada.** Hasta la revisión de seguridad, `scripts/setup_oci.sh` contenía una clave privada de OCI y los OCID de usuario y tenancy, y siguen en el historial de Git. En **Perfil → API Keys**, elimina la clave con huella `32:d5:c4:ca:1b:c8:74:c4:7e:0a:4d:ea:08:47:63:be` si sigue activa (ver [S1](#seguridad-del-despliegue)).

## Paso 1. Compartimento y permisos

1. **Identity & Security → Compartments → Create Compartment**: `communityia`.

2. Elige cómo se autentica el backend contra Object Storage:
   
   **Opción A — Clave de API (documentada).**
   Una clave de API tiene **todos los permisos del usuario que la genera**: si es tu usuario administrador, quien la robe controla la tenancy. Créala para un usuario de servicio sin acceso a la consola:
   
   1. **Identity → Domains → Default → Users → Create user**: `communityia-backend-svc`. En *Edit user capabilities* deja solo **API keys**.
   2. **Groups → Create group**: `communityia-storage`, con ese usuario como miembro.
   3. Crea la política del punto 2 de la opción B, cambiando `dynamic-group communityia-backend` por `group communityia-storage`.
   4. En el usuario de servicio: **API Keys → Add API Key → Generate API key pair**. Descarga la clave privada (`.pem`) y guarda el fragmento de configuración que muestra la consola en un archivo llamado `config`. Se instalan en la VM del backend en el paso 6.
   
   **Opción B — Instance principals [recomendado].** El backend ya lo soporta (`backend/app/storage/oci_storage.py`): si no encuentra `~/.oci/config`, se autentica con la identidad de la propia instancia y no hay claves en el servidor. Se configura después de crear la instancia (paso 4):
   
   1. **Identity → Domains → Default → Dynamic groups → Create**: `communityia-backend`, con la regla:
      
      ```
      instance.id = '<OCID_DE_LA_VM_BACKEND>'
      ```
   2. **Identity → Policies → Create Policy** en el compartimento `communityia`. Solo permite listar, leer y crear objetos en ese bucket, sin borrar ni sobrescribir: un backend comprometido no puede destruir ni alterar los paquetes ya publicados. Es todo lo que la aplicación necesita: publicar crea un objeto nuevo por lote, y el historial solo lista y lee.
      
      ```
      Allow dynamic-group communityia-backend to manage objects in compartment communityia where all {target.bucket.name='communitylab-activos-marketing', any {request.permission='OBJECT_INSPECT', request.permission='OBJECT_READ', request.permission='OBJECT_CREATE'}}
      ```
   3. Con esta opción **no** debe existir `~/.oci/config` en la VM; si existe, el backend lo usa primero.

## Paso 2. Red

### 2.1 Crear la VCN

1. Menú ☰ → **Networking → Virtual cloud networks**.
2. En el selector **Compartment** de la izquierda (o arriba, según la versión de la consola), elige `communityia`. Si no lo eliges, la VCN se crea en otro compartimento y después no la encuentras.
3. Pulsa **Start VCN Wizard**. En algunas versiones está dentro del menú **Actions**.
4. Elige **Create VCN with Internet Connectivity** y pulsa **Start VCN Wizard**.
5. Nombre: `communityia-vcn`. Deja los bloques CIDR que propone el asistente (VCN `10.0.0.0/16`, subred pública `10.0.0.0/24`, subred privada `10.0.1.0/24`). Pulsa **Next** y después **Create**.

El asistente crea la subred pública, la privada, el Internet Gateway, el NAT Gateway, las tablas de rutas y dos *security lists*:

| Security list                                      | Se aplica a                               | Qué hacer                                           |
|:-------------------------------------------------- |:----------------------------------------- |:--------------------------------------------------- |
| `Default Security List for communityia-vcn`        | Subred **pública** (donde van las dos VM) | **Aquí se agregan las reglas** de la tabla de abajo |
| `Security List for Private Subnet-communityia-vcn` | Subred privada (no se usa)                | No hace falta tocarla                               |

> Anota el CIDR de la VCN (por defecto `10.0.0.0/16`): es el origen de la regla del puerto 8000.

### 2.2 Abrir los puertos en la security list

1. En **Virtual cloud networks**, abre `communityia-vcn`.

2. Llega a la lista de *security lists*. Según la versión de la consola, de una de estas dos formas:
   
   - pestaña **Security** → sección **Security Lists**;
   - o, en la columna izquierda, **Resources → Security Lists**.

3. Abre **Default Security List for communityia-vcn**.

4. Llega a las reglas de entrada:
   
   - pestaña **Security rules**: la primera tabla es **Ingress Rules**;
   - o **Resources → Ingress Rules**.

5. Pulsa **Add Ingress Rules** y crea una regla por fila de esta tabla. Usa **+ Another Ingress Rule** para añadir varias en el mismo diálogo:
   
   | Campo                         | Regla 1 (SSH)           | Regla 2 (aplicación web)                                 | Regla 3 (API del backend)       |
   |:----------------------------- |:----------------------- |:-------------------------------------------------------- |:------------------------------- |
   | Stateless                     | sin marcar              | sin marcar                                               | sin marcar                      |
   | Source Type                   | CIDR                    | CIDR                                                     | CIDR                            |
   | Source CIDR **[recomendado]** | tu IP pública con `/32` | IP del equipo y evaluadores con `/32` (una regla por IP) | `10.0.0.0/16` (CIDR de la VCN)  |
   | Source CIDR (documentado)     | `0.0.0.0/0`             | `0.0.0.0/0`                                              | `0.0.0.0/0`                     |
   | IP Protocol                   | TCP                     | TCP                                                      | TCP                             |
   | Source Port Range             | vacío (todos)           | vacío (todos)                                            | vacío (todos)                   |
   | Destination Port Range        | `22`                    | `3000`                                                   | `8000`                          |
   | Description                   | `SSH administracion`    | `Community IA web`                                       | `API backend solo desde la VCN` |

6. Pulsa **Add Ingress Rules** para guardar.

7. Revisa las reglas que ya traía la lista:
   
   - El asistente suele crear una regla para el puerto 22 abierta a `0.0.0.0/0`. Si usas la opción recomendada (tu IP), **edita o elimina** esa regla desde el menú **⋮** de su fila; si no, tu regla restringida no sirve de nada, porque la abierta sigue dejando pasar a todos.
   - Deja las reglas ICMP que trae por defecto. OCI las usa para avisos de la red, como la fragmentación de paquetes.

Tu IP pública aparece, por ejemplo, con `curl -s https://ifconfig.me` en tu equipo. Si cambia (redes de casa o móviles), actualiza la regla.

### 2.3 Si la consola no te deja editar

| Síntoma                                                                             | Causa probable                                                                                         | Solución                                                                                                                                                                           |
|:----------------------------------------------------------------------------------- |:------------------------------------------------------------------------------------------------------ |:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| No aparece la VCN                                                                   | Otro compartimento seleccionado                                                                        | Elige `communityia` en el selector **Compartment**                                                                                                                                 |
| No aparecen **Add Ingress Rules** ni el menú **⋮**, o sale un error de autorización | Tu usuario no tiene permisos de red en ese compartimento                                               | Entra con un administrador de la tenancy, o pide la política `Allow group <tu-grupo> to manage virtual-network-family in compartment communityia`                                  |
| No encuentras **Security Lists**                                                    | La consola nueva la muestra en la pestaña **Security**                                                 | Abre la VCN → pestaña **Security**                                                                                                                                                 |
| La regla no se guarda: `Invalid CIDR`                                               | Falta la máscara                                                                                       | Escribe la IP con `/32` (por ejemplo `203.0.113.7/32`)                                                                                                                             |
| La regla está creada, pero el puerto sigue sin responder                            | La regla está en la lista de la subred privada, la VM está en otra subred o falta el firewall de la VM | Comprueba que la regla esté en `Default Security List for communityia-vcn`, que las VM estén en la **subred pública** (paso 4) y que abriste el puerto con `firewall-cmd` (paso 5) |

**Alternativa: Network Security Groups (NSG).** En lugar de la security list, puedes crear un NSG desde la misma pestaña **Security** de la VCN, con las mismas reglas, y asignarlo a cada VM editando su VNIC principal (en la ficha de la instancia, sección de red o **Attached VNICs**, campo **Network security groups**). Sirve si prefieres reglas distintas para el frontend y el backend. No hace falta usar los dos mecanismos: elige uno.

### 2.4 Por qué estas reglas

**Por qué restringir también el 3000:** la aplicación exige inicio de sesión, pero mientras no haya HTTPS ([S5](#seguridad-del-despliegue)) la contraseña y la cookie de sesión viajan sin cifrar. Cualquiera en la misma red que una persona del equipo podría capturarlas. Abre el puerto a todo internet solo cuando el acceso sea por HTTPS (ver [Pendientes](#pendientes-antes-de-considerarlo-producción)), o temporalmente durante una demo.

Con la opción recomendada, al backend solo se llega desde el frontend dentro de la VCN. Abre `8000` a internet solo si necesitas Swagger público para los evaluadores, y ciérralo después.

> **La Security List es el control que vale.** Docker publica los puertos de los contenedores con sus propias reglas de iptables, por delante de `firewalld`: quitar un puerto con `firewall-cmd` **no** lo cierra si un contenedor lo publica. Restringe el acceso aquí (o con un *Network Security Group*), no en la VM.

## Paso 3. Bucket

1. **Storage → Buckets → Create Bucket** en el compartimento `communityia`.
2. Nombre: `communitylab-activos-marketing`, nivel **Standard**, visibilidad **privada**.
3. Anota el **namespace** de la tenancy (en la ficha del bucket). El actual es `axrmv8bsyp2r`.
4. Si usas otro nombre, namespace o región, defínelos después en el `.env` del backend: `OCI_BUCKET_NAME`, `OCI_NAMESPACE`, `OCI_REGION`.

Los paquetes se guardan como `activos/paquete_N/paquete-distribucion.json`, uno por lote publicado.

## Paso 4. Instancias

Crea dos instancias en **Compute → Instances → Create Instance**:

| Campo  | Valor                                                     |
|:------ |:--------------------------------------------------------- |
| Nombre | `communityia-frontend` / `communityia-backend`            |
| Imagen | Oracle Linux 9                                            |
| Forma  | `VM.Standard.E2.1.Micro` (Always Free)                    |
| Red    | `communityia-vcn`, subred pública, **asignar IP pública** |
| SSH    | tu clave pública                                          |

Anota las **IP públicas** de ambas y la **IP privada** del backend. Si elegiste la opción B del paso 1, copia el OCID de la VM del backend y crea ahora el grupo dinámico y la política.

## Paso 5. Preparar cada VM

Conéctate con `ssh opc@<IP_PUBLICA>` y ejecuta en **ambas** instancias:

```bash
# Docker, Docker Compose y Git
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin git
sudo systemctl enable --now docker
sudo usermod -aG docker opc
exit   # vuelve a conectarte para que el grupo docker tenga efecto

# Swap de 4 GiB: con 1 GB de RAM, la compilación del frontend y LangGraph agotan la memoria
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress
sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo "/swapfile swap swap defaults 0 0" | sudo tee -a /etc/fstab

# Parches de seguridad automáticos del sistema operativo
sudo dnf install -y dnf-automatic
sudo sed -i 's/^upgrade_type.*/upgrade_type = security/; s/^apply_updates.*/apply_updates = yes/' /etc/dnf/automatic.conf
sudo systemctl enable --now dnf-automatic.timer

# Código
git clone <url-del-repositorio> ~/communityia
```

> Pertenecer al grupo `docker` equivale a tener acceso de root en la VM. Agrega solo a `opc`.

Abre el puerto en el firewall del sistema operativo, según la VM (no sustituye a la Security List; ver la nota del paso 2):

```bash
# VM backend
sudo firewall-cmd --permanent --add-port=8000/tcp && sudo firewall-cmd --reload
# VM frontend
sudo firewall-cmd --permanent --add-port=3000/tcp && sudo firewall-cmd --reload
```

> Todos los comandos de esta guía asumen que el código está en `~/communityia` (`/home/opc/communityia`). Si la VM ya tenía el código en `~/CommunityLab-G10-team23`, sigue [Mover un despliegue desde la ruta anterior](#mover-un-despliegue-desde-la-ruta-anterior) en lugar de clonar de nuevo.

## Paso 6. Backend

Todo este paso se hace en la **VM backend**.

1. **Credenciales de OCI** (solo con la opción A del paso 1). Desde tu equipo, copia los archivos a una carpeta temporal de la VM. El script los instala en `~/.oci` con permisos `600`, ajusta `key_file` y después se borran las copias:
   
   ```bash
   scp oci_api_key.pem config opc@<IP_BACKEND>:/tmp/
   ssh opc@<IP_BACKEND> "bash ~/communityia/scripts/setup_oci.sh /tmp/oci_api_key.pem /tmp/config && rm -f /tmp/oci_api_key.pem /tmp/config"
   ```
   
   El contenedor del backend corre con el UID 1000, el mismo de `opc`, así que lee `~/.oci` sin abrir los permisos. Nunca subas estos archivos al repositorio ni los pegues en scripts o documentos; el repositorio ya ignora `*.pem`, `*.key` y `.env`.

2. **Variables de entorno.** Crea `~/communityia/.env`:
   
   ```env
   GEMINI_API_KEY=<tu-clave>
   # Opcional: sin valor, el backend usa gemini-3.8-flash.
   GEMINI_MODEL=gemini-3.5-flash-lite
   # Opcional si difieren de los valores por defecto:
   # OCI_BUCKET_NAME=communitylab-activos-marketing
   # OCI_NAMESPACE=axrmv8bsyp2r
   # OCI_REGION=sa-vinhedo-1
   ```
   
   ```bash
   chmod 600 ~/communityia/.env
   ```
   
   En Google AI Studio / Google Cloud, restringe la clave de Gemini a la *Generative Language API* y configura una alerta de presupuesto: si alguien abusa de la aplicación, el gasto queda acotado.

3. **Levantar el servicio.** Al arrancar por primera vez, el backend crea `auth.db` y `app.db` en el volumen `backend-data` y aplica el esquema.
   
   ```bash
   cd ~/communityia
   docker compose up -d --build backend
   ```

4. **Crear el primer administrador.** No hay usuarios ni credenciales por defecto. El comando pide la contraseña sin mostrarla; debe tener entre 12 y 128 caracteres.
   
   ```bash
   docker compose exec -it backend python -m app.auth.crear_admin --email <tu-correo> --nombre "<tu nombre>"
   ```
   
   El resto de las cuentas se crean por invitación desde la aplicación, en **Administración → Equipo**; no existe el registro abierto. Cada invitación es un enlace válido por 72 horas que el admin envía a la persona. Ver [quién puede hacer qué](#ciclo-de-un-lote) según el rol.

5. **Verificar.** Todo endpoint, salvo `/health` y el login, exige un token de sesión:
   
   ```bash
   docker compose ps                                    # backend en estado "healthy"
   curl http://localhost:8000/health
   curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/api/v1/storage/packages   # 401 sin sesión
   
   # Inicia sesión con el administrador (la contraseña no queda en el historial de la terminal)
   read -rp "Correo: " EMAIL; read -rsp "Contraseña: " CLAVE; echo
   TOKEN=$(EMAIL="$EMAIL" CLAVE="$CLAVE" python3 -c 'import json, os, urllib.request as u; b = json.dumps({"email": os.environ["EMAIL"], "password": os.environ["CLAVE"]}).encode(); print(json.load(u.urlopen(u.Request("http://localhost:8000/api/v1/auth/login", b, {"Content-Type": "application/json"})))["token"])')
   curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/storage/packages   # fuerza la conexión con OCI
   curl -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/pipeline/jobs      # lotes del equipo: [] en una instalación nueva
   curl -X POST -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/v1/auth/logout
   docker compose logs backend | grep "OCI Storage"
   ```
   
   El log debe decir `Conectado exitosamente` (con `~/.oci/config` o con *Instance Principals*). Si dice `Usando almacenamiento Mock local`, el backend no encontró credenciales y está guardando los paquetes en disco, no en OCI: **no des el despliegue por terminado** en ese estado (ver [S7](#seguridad-del-despliegue)).

## Paso 7. Frontend

Todo este paso se hace en la **VM frontend**.

> **Backend y frontend van siempre en la misma versión.** El frontend inicia sesión y habla con el backend con un contrato que cambia entre versiones (por ejemplo, la publicación por `lote_id`). Una versión nueva de uno con la anterior del otro deja funciones rotas o sin acceso.

1. **Configurar el frontend.** Crea `~/communityia/.env`:
   
   ```env
   # IP privada del backend dentro de la VCN [recomendado]
   BACKEND_URL=http://<IP_PRIVADA_BACKEND>:8000
   # Si mantuviste el puerto 8000 abierto a internet, también sirve la IP pública.
   
   # Solo cuando haya un proxy HTTPS delante (ver Pendientes): cookie Secure e IP real del cliente.
   # COOKIE_SECURE=true
   # TRUST_PROXY=true
   ```
   
   Este `.env` no lleva secretos.
   
   > **No definas `DEMO_MODE=true`.** Ese modo desactiva el inicio de sesión y muestra datos simulados; solo sirve para enseñar la interfaz sin backend. El valor por defecto en `docker-compose.yml` es `false`.

2. **Levantar el servicio.** La imagen (Node 22 + Astro) se compila en la VM; con 1 GB de RAM tarda varios minutos y necesita la swap del paso 5.
   
   ```bash
   cd ~/communityia
   docker compose up -d --build frontend
   ```

3. **Verificar.**
   
   ```bash
   docker compose ps                         # frontend en estado "healthy"
   curl http://localhost:3000/api/health     # debe responder "status":"online"
   ```
   
   | Respuesta            | Qué significa                                                                                                                      |
   |:-------------------- |:---------------------------------------------------------------------------------------------------------------------------------- |
   | `"status":"online"`  | Correcto: el frontend llega al backend                                                                                             |
   | `"status":"offline"` | El frontend no llega al backend. Revisa `BACKEND_URL`, la regla del puerto 8000 en la Security List y el firewall de la VM backend |
   | `"status":"demo"`    | `DEMO_MODE=true` está activo. Quítalo del `.env` y recrea el contenedor con `docker compose up -d --force-recreate frontend`       |

## Paso 8. Prueba de punta a punta

Prueba con dos cuentas para comprobar la revisión en equipo y los permisos:

1. Abre `http://<IP_PUBLICA_FRONTEND>:3000`: debe aparecer la portada de Community IA. Pulsa **Iniciar sesión** y entra con el administrador del paso 6. En la barra lateral deben aparecer **Conectado** y tu nombre.
2. En **Administración → Equipo**, invita a una cuenta con rol *Revisión*. Abre el enlace de la invitación en una ventana privada y crea la cuenta.
3. Con el admin, sube un archivo de prueba desde **Bandeja → Subir archivo**. Puedes usar los de `examples/inputs/`. Los resultados aparecen mensaje a mensaje.
4. Con la cuenta de revisión, abre la bandeja. El lote aparece en **Lotes del equipo por revisar**, con quién lo subió. Ábrelo y aprueba un contenido: debe mostrar "Aprobado por <nombre>". Esa cuenta no ve el botón **Guardar lote**.
5. Con el admin, pulsa **Otros lotes**, abre el mismo lote, revisa lo que falte y pulsa **Guardar lote**.
6. Comprueba lo siguiente:
   - en el bucket aparece `activos/paquete_N/paquete-distribucion.json`;
   - su bloque `auditoria` nombra a quien procesó, revisó y publicó;
   - el paquete se ve en **Tabla de casos** y en **Dashboard**;
   - el lote ya no aparece en la bandeja.
7. Sube otro archivo y, con el admin, elimínalo desde el icono de papelera de su tarjeta. Debe pedir confirmación antes de borrar.

## Paso 9. Operación

| Tarea                                                    | Comando (en la VM correspondiente)                                                                                                                                                                                  |
|:-------------------------------------------------------- |:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Actualizar a la última versión                           | Ver [Actualizar un despliegue existente](#actualizar-un-despliegue-existente)                                                                                                                                       |
| Ver estado                                               | `docker compose ps`                                                                                                                                                                                                 |
| Ver logs                                                 | `docker compose logs -f backend` (o `frontend`)                                                                                                                                                                     |
| Reiniciar tras cambiar `.env`                            | `docker compose up -d --force-recreate backend` (o `frontend`)                                                                                                                                                      |
| Copia de seguridad de las bases (VM backend)             | Ver [Copias de seguridad](#copias-de-seguridad)                                                                                                                                                                     |
| Ver los lotes guardados (VM backend)                     | `docker compose exec backend python -c "import sqlite3; [print(*r) for r in sqlite3.connect('/app/data/app.db').execute('SELECT id, estado, creado_por_nombre, publicado_en FROM lotes ORDER BY creado_en DESC')]"` |
| Liberar espacio de imágenes viejas                       | `docker image prune -f`                                                                                                                                                                                             |
| Recompilar con imágenes base parcheadas (una vez al mes) | `docker compose build --pull backend && docker compose up -d backend` (o `frontend`)                                                                                                                                |
| Comprobar que no hay secretos en el repositorio          | `git grep -nE "PRIVATE KEY\|LS0tLS1CRUdJTi\|AIza"` (no debe devolver nada)                                                                                                                                          |

Los contenedores tienen `restart: unless-stopped`: vuelven a arrancar solos después de reiniciar la VM, siempre que Docker esté habilitado (`systemctl enable docker`, paso 5).

**Antes de dar el despliegue por terminado, programa las copias de seguridad diarias** de la VM backend ([Copias de seguridad](#copias-de-seguridad)).

## Actualizar un despliegue existente

Actualiza siempre las dos VM a la misma versión: primero el backend, después el frontend.

1. **VM backend:** haz una [copia de seguridad](#copias-de-seguridad) y después:
   
   ```bash
   cd ~/communityia
   git pull
   docker compose up -d --build backend
   docker compose logs backend | tail -20   # sin errores al arrancar
   ```
   
   Al arrancar, el backend aplica solo los cambios pendientes al esquema de `app.db` (`backend/app/persistence/db.py`). No hace falta ningún comando de migración.

2. **VM frontend:**
   
   ```bash
   cd ~/communityia
   git pull
   docker compose up -d --build frontend
   curl http://localhost:3000/api/health     # "status":"online"
   ```

### Mover un despliegue desde la ruta anterior

Las versiones anteriores de esta guía clonaban el código en `~/CommunityLab-G10-team23`. Docker Compose nombraba el volumen de datos según esa carpeta (`communitylab-g10-team23_backend-data`). Ahora el proyecto se llama siempre `communityia` y su volumen es `communityia_backend-data`. Por eso, al mover la carpeta hay que copiar los datos al volumen nuevo; si no, el backend arrancaría sin cuentas ni lotes.

En la **VM backend**:

```bash
# 1. Detener los contenedores de la ruta anterior (sin -v: el volumen se conserva)
cd ~/CommunityLab-G10-team23 && docker compose down
cd ~ && mv ~/CommunityLab-G10-team23 ~/communityia

# 2. Copiar los datos al volumen nuevo (comprueba el nombre del anterior con: docker volume ls)
docker volume create communityia_backend-data
docker run --rm -v communitylab-g10-team23_backend-data:/desde:ro -v communityia_backend-data:/hacia alpine sh -c 'cp -a /desde/. /hacia/ && ls -l /hacia'

# 3. Actualizar y levantar desde la ruta nueva
cd ~/communityia && git pull && docker compose up -d --build backend
docker compose exec backend ls -l /app/data   # deben estar auth.db y, si ya existía, app.db
```

En la **VM frontend** no hay datos que copiar:

```bash
cd ~/CommunityLab-G10-team23 && docker compose down
cd ~ && mv ~/CommunityLab-G10-team23 ~/communityia
cd ~/communityia && git pull && docker compose up -d --build frontend
```

Después:

- Comprueba que puedes iniciar sesión con tu cuenta de siempre.
- Si programaste copias de seguridad, cambia la ruta en `~/respaldar.sh`.
- Cuando todo funcione, borra el volumen anterior con `docker volume rm communitylab-g10-team23_backend-data`. Antes, haz una copia de seguridad.

### Desde una versión anterior a la revisión en equipo

**Si vienes de una versión anterior a la revisión en equipo** (la que guardaba los contenidos en `activos_candidatos.json`):

- Publica o descarta antes de actualizar los lotes que estén en revisión. Los contenidos pendientes en el archivo JSON **no se migran** a `app.db`. Los paquetes ya publicados en el bucket no cambian y se siguen viendo en el historial.
- Los paquetes antiguos no tienen bloque `auditoria`; los nuevos sí.
- Desde esta versión, **solo un admin puede publicar**. Comprueba que el equipo tiene al menos una cuenta admin además de la tuya.
- Si la VM tiene un volumen `backend-data` creado por una versión que corría como root, cámbiale el dueño una vez antes de arrancar: `docker compose run --rm --user root backend chown -R 1000:1000 /app/data`.

## Lotes, revisión en equipo y datos

### Ciclo de un lote

```
Subir archivo ──► Procesando ──► En revisión (equipo) ──► Publicado (admin) ──► Cerrado
  (revisor)        (IA, en cola)   aprobar / rechazar        bucket de OCI        solo lectura
                                          │
                                          └──► Eliminado (admin, con confirmación)
```

1. **Subir.** Quien sube el archivo queda como autor del lote. El backend responde enseguida y procesa en segundo plano, un lote y un mensaje a la vez. Cada mensaje terminado se guarda de inmediato.
2. **Revisar.** Los lotes son del equipo: cualquier cuenta con rol *Revisión* o *Administración* abre cualquier lote sin publicar desde **Lotes del equipo por revisar** y aprueba, rechaza o edita su contenido.
   - Cada decisión queda firmada con quién la tomó y cuándo, y se agrega a un historial que no se modifica.
   - Si dos personas cambian el mismo contenido a la vez, la segunda recibe un aviso con la versión de la primera en lugar de pisarla.
3. **Publicar (solo admin).** **Guardar lote** toma las decisiones guardadas en el servidor y sube a OCI solo el contenido aprobado. El paquete incluye un bloque `auditoria` con quién procesó, revisó y publicó.
   - Un lote publicado queda cerrado: su contenido ya no se puede cambiar ni el lote se puede eliminar.
   - El historial definitivo es el bucket.
4. **Eliminar (solo admin).** Borra un lote sin publicar con sus mensajes, contenidos e historial de revisión, después de una confirmación en pantalla.
   - No se puede eliminar un lote que se está procesando ni uno publicado.
   - El borrado no se puede deshacer, salvo restaurando una copia de seguridad.

Quién puede hacer qué:

| Acción                                             | Solo lectura | Revisión | Administración |
|:-------------------------------------------------- |:------------:|:--------:|:--------------:|
| Ver Tabla de casos y Dashboard (bucket)            | ✓            | ✓        | ✓              |
| Subir archivos y revisar cualquier lote del equipo |              | ✓        | ✓              |
| Guardar un lote en OCI (publicar)                  |              |          | ✓              |
| Eliminar un lote sin publicar                      |              |          | ✓              |
| Invitar personas y gestionar el equipo             |              |          | ✓              |

### Dónde vive cada dato

| Lugar                              | Contenido                                                                   | Si se pierde                                                     |
|:---------------------------------- |:--------------------------------------------------------------------------- |:---------------------------------------------------------------- |
| `auth.db` (volumen `backend-data`) | Cuentas, sesiones, invitaciones                                             | Nadie puede entrar; hay que recrear el admin e invitar de nuevo  |
| `app.db` (volumen `backend-data`)  | Lotes, mensajes analizados, contenidos en revisión, historial de decisiones | Se pierde el trabajo **sin publicar** y su historial de revisión |
| Bucket de OCI                      | Paquetes publicados, con su `auditoria`                                     | Se pierde el historial definitivo (no lo protege este volumen)   |

- `app.db` y `auth.db` son archivos SQLite. `app.db` usa el modo WAL, así que en la carpeta también aparecen `app.db-wal` y `app.db-shm`: forman parte de la base y **no se borran a mano**.
- Si el backend se reinicia a mitad de un análisis, ese lote queda como interrumpido y conserva lo ya analizado. No hace falta intervenir: el equipo puede revisarlo o el admin eliminarlo.
- El volumen vive en el disco de arranque de la VM backend: sobrevive a reinicios y reconstrucciones del contenedor, pero **no** a terminar la instancia ni a borrar su disco. Por eso las copias de seguridad.

### Un solo proceso de backend

El backend corre como un único proceso de Uvicorn (`backend/Dockerfile`), y debe seguir así. La cola de análisis, el candado que evita publicar dos veces el mismo lote y el límite de intentos de login viven en la memoria de ese proceso. No añadas `--workers` ni levantes réplicas del backend contra el mismo volumen. Para un equipo pequeño en una VM de 1 GB, un proceso con SQLite es suficiente.

### Copias de seguridad

Copia las dos bases con la función de respaldo de SQLite, que es segura con el backend en marcha. Copiar los archivos `.db` directamente puede dejar fuera los últimos cambios, que están en `-wal`.

1. Crea el script en la VM backend:
   
   ```bash
   mkdir -p ~/respaldos && chmod 700 ~/respaldos
   cat > ~/respaldar.sh <<'SCRIPT'
   #!/usr/bin/env bash
   set -euo pipefail
   cd ~/communityia
   docker compose exec -T backend python -c "
   import sqlite3
   for n in ('app', 'auth'):
       s = sqlite3.connect(f'/app/data/{n}.db'); d = sqlite3.connect(f'/tmp/{n}.db.bak')
       s.backup(d); d.close(); s.close()
   "
   F=$(date +%Y%m%d-%H%M)
   docker compose cp backend:/tmp/app.db.bak ~/respaldos/app-$F.db
   docker compose cp backend:/tmp/auth.db.bak ~/respaldos/auth-$F.db
   chmod 600 ~/respaldos/*.db
   find ~/respaldos -name '*.db' -mtime +30 -delete
   SCRIPT
   chmod 700 ~/respaldar.sh
   ~/respaldar.sh && ls -l ~/respaldos
   ```

2. Prográmalo a diario con `crontab -e`, añadiendo la línea `0 3 * * * /home/opc/respaldar.sh`.

3. Lleva las copias **fuera de la VM**, por ejemplo a tu equipo con `scp opc@<IP_BACKEND>:respaldos/*.db .`. Una copia que solo existe en la misma VM no sirve si se pierde la instancia.

`auth-*.db` contiene los hashes de las contraseñas: guárdala fuera del repositorio y con permisos restringidos.

**Restaurar** (por ejemplo, después de recrear la VM o de eliminar un lote por error). Restaurar reemplaza la base completa: se pierde lo que se hizo después de la copia.

```bash
cd ~/communityia
docker compose stop backend
docker compose run --rm --no-deps -v ~/respaldos:/restaurar:ro backend sh -c \
  'cp /restaurar/app-AAAAMMDD-HHMM.db /app/data/app.db && rm -f /app/data/app.db-wal /app/data/app.db-shm'
docker compose start backend
```

Para `auth.db` es igual, cambiando el nombre del archivo. Después de restaurar `auth.db`, las sesiones que había en la copia vuelven a ser válidas hasta que caduquen.

## Configuración de Docker Compose

`docker-compose.yml` está preparado para estas instancias:

| Ajuste                                               | Para qué sirve                                                                                                               |
|:---------------------------------------------------- |:---------------------------------------------------------------------------------------------------------------------------- |
| `restart: unless-stopped`                            | Los servicios se recuperan tras un reinicio de la VM o una caída                                                             |
| Backend sin `--reload` y sin montar el código        | Modo producción: la imagen contiene el código y no se reinicia al tocar archivos                                             |
| Volumen `backend-data` en `/app/data`                | `auth.db`, `app.db` (lotes y revisión) y el almacenamiento local de respaldo sobreviven a reconstrucciones                   |
| `.env` opcional (`required: false`)                  | Cada VM lleva solo lo que necesita: secretos en el backend, `BACKEND_URL` en el frontend                                     |
| `BACKEND_URL=${BACKEND_URL:-http://backend:8000}`    | Se define en el `.env` del frontend; sin valor, apunta al backend del mismo compose                                          |
| `COOKIE_SECURE`, `TRUST_PROXY` (por defecto `false`) | Se activan solo detrás de un proxy HTTPS                                                                                     |
| `DEMO_MODE` (por defecto `false`)                    | Modo de demostración sin backend ni inicio de sesión; nunca en producción                                                    |
| `OCI_CONFIG_DIR` (por defecto `/home/opc/.oci`)      | Carpeta de credenciales de OCI montada en solo lectura                                                                       |
| `healthcheck` en ambos servicios                     | `docker compose ps` muestra si cada servicio responde                                                                        |
| Usuario sin privilegios (`app` y `node`, UID 1000)   | Una vulnerabilidad en la aplicación no da root dentro del contenedor, y el proceso no puede modificar el código de la imagen |
| `cap_drop: ALL` y `no-new-privileges`                | El proceso no conserva capacidades de Linux ni puede ganarlas (por ejemplo, con binarios setuid)                             |
| Logs rotados (`10m` × 3 por servicio)                | Los logs no llenan el disco de la VM                                                                                         |
| `backend/.dockerignore`                              | La imagen no incluye `data/`, los tests, `.env` ni claves                                                                    |

Para desarrollo local con recarga automática del backend:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up --build
```

## Seguridad del despliegue

Revisión del plan de despliegue, la configuración de Docker, el repositorio y el código expuesto a la red. La severidad corresponde al despliegue tal como estaba documentado, con todos los puertos abiertos a `0.0.0.0/0`.

| Id  | Severidad | Hallazgo                                                                                                                                                                                                                                                                                                                                                | Estado                                                                                                                                                                                                                                                                                                                                                                                                            |
|:--- |:--------- |:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| S1  | Crítica   | `scripts/setup_oci.sh` contenía la clave privada de la API de OCI (en base64), el OCID del usuario, el de la tenancy y la huella. Los documentos decían que ya se había quitado, pero seguía en la rama actual. Con esa clave, cualquiera con acceso al repositorio actúa en OCI con los permisos de ese usuario.                                       | **Corregido en el repo:** el script ya no lleva secretos e instala archivos locales. **Acción manual:** revocar la clave (paso 0) y limpiar el historial (abajo).                                                                                                                                                                                                                                                 |
| S2  | Alta      | La aplicación no tenía autenticación y el frontend estaba abierto a internet: cualquiera podía gastar la cuota de Gemini, aprobar contenido, publicar en el bucket y leer todos los paquetes.                                                                                                                                                           | **Corregido en el código:** cuentas por invitación, roles (`admin`, `revisor`, `lector`), sesiones opacas, límite de intentos por correo y por IP, cookie `httpOnly` + `SameSite=Strict`, verificación de origen (CSRF) e interfaz adaptada al rol. Publicar y eliminar lotes es solo de admin, y cada decisión de revisión queda firmada. Mientras no haya HTTPS (S5), mantén el puerto 3000 restringido por IP. |
| S3  | Alta      | Backend (8000, con Swagger) y SSH (22) abiertos a `0.0.0.0/0`; el backend aceptaba cualquier petición sin credenciales.                                                                                                                                                                                                                                 | **Corregido en la guía:** 8000 solo desde la VCN y 22 solo desde tu IP o Bastion (paso 2). **Corregido en el código:** el backend exige sesión y rol en todos los endpoints salvo `/health` y el login. Swagger sigue publicado (ver pendientes de código).                                                                                                                                                       |
| S4  | Alta      | La clave de API de la opción A era la de un usuario personal (normalmente administrador), y la política de instance principals daba `manage objects`, que incluye borrar y sobrescribir.                                                                                                                                                                | **Corregido en la guía:** usuario de servicio solo con claves de API y política limitada a listar, leer y crear objetos (paso 1).                                                                                                                                                                                                                                                                                 |
| S5  | Media     | Tráfico por HTTP: contraseñas, cookies de sesión y contenidos viajan sin cifrar.                                                                                                                                                                                                                                                                        | **Pendiente** (ver pendientes).                                                                                                                                                                                                                                                                                                                                                                                   |
| S6  | Media     | Los contenedores corrían como root con las capacidades por defecto, y la imagen del backend copiaba `data/` y los tests.                                                                                                                                                                                                                                | **Corregido:** UID 1000, `cap_drop: ALL`, `no-new-privileges` y `backend/.dockerignore`. Verificado en contenedores locales.                                                                                                                                                                                                                                                                                      |
| S7  | Media     | Si fallan las credenciales de OCI, el backend pasa **en silencio** al almacenamiento local (mock) y la aplicación sigue diciendo que publicó. Además, en ese modo la ruta de `GET /api/v1/storage/packages/{ruta}.json` no se valida, así que podría leer archivos `.json` fuera de la carpeta del mock (hallazgo de revisión de código, no explotado). | **Mitigado por procedimiento:** verificación del log en el paso 6. **Pendiente de código** (abajo).                                                                                                                                                                                                                                                                                                               |
| S8  | Baja      | `firewalld` no protege los puertos que publica Docker.                                                                                                                                                                                                                                                                                                  | **Documentado** (paso 2): el control es la Security List.                                                                                                                                                                                                                                                                                                                                                         |
| S9  | Baja      | Sin parches automáticos del sistema operativo ni rotación de logs (un disco lleno tumba el servicio).                                                                                                                                                                                                                                                   | **Corregido:** `dnf-automatic` (paso 5) y logs rotados en compose.                                                                                                                                                                                                                                                                                                                                                |
| S10 | Baja      | Los errores 500 devuelven al cliente el mensaje interno de la excepción, y el README publica las IP de las VM.                                                                                                                                                                                                                                          | **Pendiente de código o decisión del equipo.**                                                                                                                                                                                                                                                                                                                                                                    |
| S11 | Media     | Los lotes y la revisión del equipo viven solo en el disco de la VM backend.                                                                                                                                                                                                                                                                             | **Mitigado por procedimiento:** copias de seguridad diarias fuera de la VM ([Copias de seguridad](#copias-de-seguridad)).                                                                                                                                                                                                                                                                                         |

### Limpiar la clave del historial de Git (S1)

Quitar la clave de la rama no la borra de los commits anteriores (desde `e94d849`) ni de los forks. **Lo que la neutraliza es revocarla**; la limpieza evita que se reutilicen los identificadores. Reescribir el historial cambia los hashes de todos los commits, así que acuérdalo con el equipo antes:

```bash
pip install git-filter-repo
git clone --mirror https://github.com/lunajulio/CommunityIA.git && cd CommunityIA.git

# Reemplazos: cada línea de la clave, la clave en una sola línea, los OCID y la huella
git show e94d849:scripts/setup_oci.sh | sed -n '/^LS0t/,/^EOF/p' | grep -v '^EOF' > ../reemplazos.txt
cat >> ../reemplazos.txt <<'EOF'
regex:LS0tLS1CRUdJTi[A-Za-z0-9+/=]*
regex:ocid1\.(user|tenancy)\.oc1\.\.[a-z0-9]+
32:d5:c4:ca:1b:c8:74:c4:7e:0a:4d:ea:08:47:63:be
EOF

git filter-repo --replace-text ../reemplazos.txt
git push --force --mirror
```

Después, cada integrante vuelve a clonar el repositorio. Repite el proceso en el repositorio de la organización (`No-Country-simulation`) si la clave llegó allí, y si el repositorio es público, pide a GitHub Support que purgue las vistas en caché.

### Cambios de código pendientes

Estos cambios necesitan acuerdo del equipo:

1. **Fallo explícito del almacenamiento (S7).** Con `ENVIRONMENT=production`, que el backend se niegue a usar el mock y falle al arrancar o al publicar. Además, comprobar que la ruta pedida al leer un paquete quede dentro de la carpeta del mock.
2. **Swagger solo en desarrollo (S3).** `FastAPI(docs_url=None, redoc_url=None)` cuando `ENVIRONMENT=production`.
3. **Límites de tamaño (S2).** Longitud máxima de `texto` por interacción en el contrato y límite de tamaño del cuerpo en el BFF, para acotar el gasto por petición.
4. **Errores genéricos (S10).** Registrar la excepción en el log y devolver al cliente un mensaje sin detalles internos.

## Pendientes antes de considerarlo producción

1. **HTTPS (S5).** El acceso es por HTTP con IP. Opciones dentro de Always Free:
   
   - un *Flexible Load Balancer* de 10 Mbps con certificado delante del frontend;
   - un proxy inverso (Caddy) en la VM del frontend con un dominio. Caddy obtiene el certificado de Let's Encrypt automáticamente.
   
   Con el proxy en marcha, define `COOKIE_SECURE=true` y `TRUST_PROXY=true` en el `.env` del frontend, cierra el 3000 a internet (que solo lo alcance el proxy) y abre el 443. Es requisito para abrir la aplicación al público, porque hay contraseñas.

2. **Los [cambios de código pendientes](#cambios-de-código-pendientes).**

3. **Copias de seguridad fuera de la VM (S11).** Programa las [copias de seguridad](#copias-de-seguridad) y llévalas fuera de la instancia antes de confiarle trabajo real al equipo.

4. **Imágenes compiladas en las VM.** Compilar en máquinas de 1 GB es lento. Se puede compilar en otro equipo y publicar las imágenes en OCI Container Registry.
