# 🛡️ OpenShell — guía rápida (Español)

> Comandos verificados con la [documentación oficial (v0.1.3)](https://docs.nvidia.com/openshell/latest/index.html) y el README de [NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell) el 2026-10-12.
> Qué es: un entorno sandbox para agentes de IA, gobernado por políticas. Tú declaras en una política qué puede tocar cada agente —archivos, red, credenciales— y OpenShell lo hace cumplir a nivel de kernel.

---

## 0. Instalación y primer sandbox

Requisitos: Linux, macOS con Apple Silicon o Windows con WSL 2 (experimental), más Docker, Podman o virtualización en el host.

```shell
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
openshell sandbox create --name demo
```

El instalador deja listo el CLI y un gateway local. La imagen de sandbox por defecto es un Ubuntu mínimo, sin ningún agente instalado.

---

## 1. Comandos principales

**Ciclo de vida del sandbox**

```shell
openshell sandbox create --name my-sandbox --policy ./policy.yaml  # crear con una política
openshell sandbox list                                             # listar sandboxes
openshell sandbox get my-sandbox                                   # ver el detalle de uno
openshell sandbox connect my-sandbox                               # abrir una sesión
openshell sandbox exec -n my-sandbox -- ls -la /workspace          # ejecutar un comando dentro
openshell sandbox stop my-sandbox / start my-sandbox               # detener / arrancar
openshell sandbox delete my-sandbox                                # eliminar
openshell logs my-sandbox                                          # ver los registros
```

**Políticas**

```shell
openshell policy get my-sandbox --base        # la política tuya (base)
openshell policy get my-sandbox --full        # base + reglas de proveedores (efectiva)
openshell policy list my-sandbox              # historial de revisiones
openshell policy set my-sandbox --policy policy.yaml --wait   # reemplazar la política entera
```

Añadir o quitar acceso de red en un sandbox en marcha, sin escribir el archivo completo:

```shell
# Permitir que curl, dentro del sandbox, haga peticiones de solo lectura a la API de GitHub
openshell policy update my-sandbox \
  --rule-name github_readonly \
  --binary /usr/bin/curl \
  --add-endpoint api.github.com:443:read-only:rest:enforce \
  --wait

openshell policy update my-sandbox --remove-rule github_readonly --wait
```

Formato del endpoint: `host:puerto:preset:protocolo:modo`. `read-only` permite GET, HEAD y OPTIONS; `rest` activa la inspección de peticiones HTTP; `enforce` bloquea todo lo demás. Cambia `--wait` por `--dry-run` para previsualizar el cambio sin aplicarlo.

**Revisar una política antes de aplicarla (prover independiente)**

```shell
openshell-prover check candidate.yaml --boundary boundary.yaml
```

El prover usa verificación formal: comprueba que la política candidata no concede más acceso del que define tu boundary.

---

## 2. El archivo de política

YAML, empieza con `version: 1`. Todo lo que la política no permite queda denegado.

| Sección | Controla | Lo impone | Entra en vigor |
|---|---|---|---|
| `filesystem_policy` | Rutas que los procesos pueden leer, o leer y escribir | Landlock LSM, en el kernel | Al arrancar el sandbox |
| `landlock` | Si el sandbox arranca aunque no se puedan aplicar las reglas de archivos | El runtime del sandbox | Al arrancar el sandbox |
| `process` | Usuario y grupo con los que corren los procesos | Docker / Podman al crear | Al crear el sandbox |
| `network_policies` | A qué destinos llega cada binario y qué peticiones puede enviar | El proxy de red del sandbox | Mientras el sandbox corre |
| `network_middlewares` | Inspección o bloqueo extra del tráfico permitido | El proxy de red del sandbox | Mientras el sandbox corre |

- Todo sandbox corre bajo una política, aunque no le des ninguna: OpenShell aplica una política por defecto restrictiva.
- De dónde sale la política, en orden: una política global del administrador del gateway → la política guardada del sandbox (`--policy` al crear, o la variable `OPENSHELL_SANDBOX_POLICY`) → una política dentro de la imagen, en `/etc/openshell/policy.yaml` → la política por defecto.
- Las reglas de red se pueden cambiar con el sandbox en marcha. Las de archivos y procesos quedan fijas al arrancar: para cambiarlas hay que recrear el sandbox.

---

## 3. Arquitectura en tres piezas

- **Sandbox** — vive dentro de la frontera, junto al agente. Es dueño de los procesos del agente, sabe qué programa hizo cada petición y reenvía el tráfico TCP y DNS al supervisor. Su única salida de red permitida es el canal protegido hacia el supervisor.
- **Supervisor** — vive en el lado de confianza de la frontera. Revisa cada petición contra la política, entrega las credenciales, resuelve DNS y abre las conexiones aprobadas.
- **Gateway** — el plano de control. Gestiona el ciclo de vida de los sandboxes, entrega la política y los ajustes, conecta los proveedores, verifica quién eres y ejecuta el prover de políticas.
- **Providers** — conectan el nombre de un servicio con una credencial guardada. El supervisor entrega esa credencial solo donde la política lo permite.

---

## 4. Modelo de seguridad

- **Control a nivel de kernel.** Cada acceso a archivo, cada llamada al sistema y cada conexión de red se revisa en el momento.
- **El agente nunca ve las credenciales reales.** OpenShell las añade solo a las peticiones dirigidas a endpoints aprobados. Un prompt filtrado o un agente curioso no puede exfiltrar una clave que nunca tuvo.
- **Denegar por defecto.** Toda conexión saliente queda denegada salvo que una regla de `network_policies` la permita —hasta el detalle de qué binario puede enviar qué petición (leer de una API, sí; escribir en ella, no).
- **Los cambios arriesgados esperan a una persona.** Antes de aprobar un cambio de política, la verificación formal marca los accesos nuevos con riesgo: llegar a un host nuevo con credenciales, llamar a un método nuevo de una API, tocar los endpoints de metadatos de la nube. Cualquier hallazgo bloquea la aprobación automática.

---

## 5. De principio a fin: ejecutar un agente y aprobar accesos sobre la marcha

Este flujo ejecuta OpenCode contra un modelo gratuito de OpenRouter (de la guía oficial «Run Your First Agent»).

**1. Importa un perfil de proveedor y crea el proveedor** — define antes `OPENROUTER_API_KEY`; OpenShell la guarda y la envía solo a `openrouter.ai`:

```shell
openshell profile import \
  --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml

openshell provider create \
  --name openrouter \
  --type openrouter \
  --from-existing
```

**2. Crea el sandbox con el agente como proceso principal:**

```shell
openshell sandbox create \
  --name my-agent \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free
```

**3. Aprueba accesos nuevos cuando el agente los necesite.** Si el agente intenta llegar a un destino que la política no permite, OpenShell deniega la petición y redacta una regla estrecha (el policy advisor). El gateway la revisa con el prover y tú decides:

```shell
openshell rule get my-agent --status pending                       # listar propuestas pendientes
openshell rule approve my-agent --chunk-id <chunk-id>               # aprobar una
openshell rule reject my-agent --chunk-id <chunk-id> \
  --reason "Not needed for this task."                              # rechazar una, con motivo
openshell rule approve-all my-agent                                # aprobar todas las pendientes
```

Las reglas aprobadas se aplican en caliente, sin reiniciar el sandbox: el agente simplemente reintenta la petición.

---

## 6. Extras

- **Enseña OpenShell a tu agente de código:** `npx skills add NVIDIA/OpenShell` — skills para manejar el CLI, escribir políticas y depurar gateways.
- **SDK** (se conectan a un gateway; no instalan el CLI): Python `uv add openshell` · TypeScript `npm install @nvidia/openshell-sdk` · Go `go get github.com/NVIDIA/OpenShell/sdk/go@latest` · Rust `cargo add openshell-sdk`.
- **Probar la API del gateway:** sirve gRPC reflection en `localhost:18080` — prueba `grpcurl -plaintext localhost:18080 list`.

---

*Documentación: https://docs.nvidia.com/openshell/latest/index.html · Repo: https://github.com/NVIDIA/OpenShell · Dudas: GitHub Discussions.*
