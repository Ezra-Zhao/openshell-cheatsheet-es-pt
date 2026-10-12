# 🛡️ OpenShell — guia rápido (Português)

> Comandos verificados com a [documentação oficial (v0.1.3)](https://docs.nvidia.com/openshell/latest/index.html) e o README de [NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell) em 2026-10-12.
> O que é: um runtime de sandbox para agentes de IA, governado por políticas. Você declara numa política o que cada agente pode acessar —arquivos, rede, credenciais— e o OpenShell impõe isso no nível do kernel.

---

## 0. Instalação e primeiro sandbox

Requisitos: Linux, macOS com Apple Silicon ou Windows com WSL 2 (experimental), além de Docker, Podman ou virtualização no host.

```shell
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
openshell sandbox create --name demo
```

O instalador configura a CLI e um gateway local. A imagem padrão do sandbox é um Ubuntu mínimo, sem nenhum agente instalado.

---

## 1. Comandos principais

**Ciclo de vida do sandbox**

```shell
openshell sandbox create --name my-sandbox --policy ./policy.yaml  # criar com uma política
openshell sandbox list                                             # listar sandboxes
openshell sandbox get my-sandbox                                   # ver os detalhes de um
openshell sandbox connect my-sandbox                               # abrir uma sessão
openshell sandbox exec -n my-sandbox -- ls -la /workspace          # rodar um comando dentro
openshell sandbox stop my-sandbox / start my-sandbox               # parar / iniciar
openshell sandbox delete my-sandbox                                # apagar
openshell logs my-sandbox                                          # ver os logs
```

**Políticas**

```shell
openshell policy get my-sandbox --base        # a sua política (base)
openshell policy get my-sandbox --full        # base + regras dos providers (efetiva)
openshell policy list my-sandbox              # histórico de revisões
openshell policy set my-sandbox --policy policy.yaml --wait   # substituir a política inteira
```

Adicionar ou remover acesso de rede com o sandbox rodando, sem escrever o arquivo completo:

```shell
# Permitir que o curl, dentro do sandbox, faça requisições somente leitura à API do GitHub
openshell policy update my-sandbox \
  --rule-name github_readonly \
  --binary /usr/bin/curl \
  --add-endpoint api.github.com:443:read-only:rest:enforce \
  --wait

openshell policy update my-sandbox --remove-rule github_readonly --wait
```

Formato do endpoint: `host:porta:preset:protocolo:modo`. `read-only` permite GET, HEAD e OPTIONS; `rest` liga a inspeção das requisições HTTP; `enforce` bloqueia todo o resto. Troque `--wait` por `--dry-run` para pré-visualizar a mudança sem aplicar.

**Conferir uma política antes de aplicar (prover standalone)**

```shell
openshell-prover check candidate.yaml --boundary boundary.yaml
```

O prover usa verificação formal: ele confere se a política candidata não concede mais acesso do que o boundary que você definiu.

---

## 2. O arquivo de política

YAML, começa com `version: 1`. Tudo o que a política não permite é negado.

| Seção | Controla | Quem impõe | Quando vale |
|---|---|---|---|
| `filesystem_policy` | Caminhos que os processos podem ler, ou ler e escrever | Landlock LSM, no kernel | Na subida do sandbox |
| `landlock` | Se o sandbox sobe mesmo sem conseguir aplicar as regras de arquivos | Runtime do sandbox | Na subida do sandbox |
| `process` | Usuário e grupo dos processos | Docker / Podman na criação | Na criação do sandbox |
| `network_policies` | Quais destinos cada binário alcança e quais requisições pode enviar | Proxy de rede do sandbox | Enquanto o sandbox roda |
| `network_middlewares` | Inspeção ou bloqueio extra do tráfego permitido | Proxy de rede do sandbox | Enquanto o sandbox roda |

- Todo sandbox roda sob uma política, mesmo sem você fornecer nenhuma: o OpenShell aplica uma política padrão restritiva.
- De onde vem a política, nesta ordem: uma política global do admin do gateway → a política salva do sandbox (`--policy` na criação, ou a variável `OPENSHELL_SANDBOX_POLICY`) → uma política dentro da imagem, em `/etc/openshell/policy.yaml` → a política padrão.
- Regras de rede mudam com o sandbox rodando. As de arquivos e processos ficam fixas depois da subida: para mexer nelas, é preciso recriar o sandbox.

---

## 3. Arquitetura em três peças

- **Sandbox** — fica dentro da fronteira, junto do agente. É dono dos processos do agente, sabe qual programa fez cada requisição e encaminha o tráfego TCP e DNS ao supervisor. A única saída de rede permitida é o canal protegido até o supervisor.
- **Supervisor** — fica no lado confiável da fronteira. Confere cada requisição contra a política, entrega as credenciais, resolve DNS e abre as conexões aprovadas.
- **Gateway** — o plano de controle. Gerencia o ciclo de vida dos sandboxes, entrega política e configurações, conecta os providers, verifica quem você é e roda o prover de políticas.
- **Providers** — ligam o nome de um serviço a uma credencial guardada. O supervisor entrega essa credencial só onde a política permite.

---

## 4. Modelo de segurança

- **Imposição no nível do kernel.** Cada acesso a arquivo, cada chamada de sistema e cada conexão de rede é conferida na hora.
- **O agente nunca vê as credenciais reais.** O OpenShell as injeta só nas requisições destinadas a endpoints aprovados. Um prompt vazado ou um agente curioso não exfiltra uma chave que ele nunca teve.
- **Negar por padrão.** Toda conexão de saída é negada, a menos que uma regra de `network_policies` a permita — no detalhe de qual binário pode enviar qual requisição (ler de uma API, sim; escrever nela, não).
- **Mudança arriscada espera uma pessoa.** Antes de aprovar uma mudança de política, a verificação formal aponta os acessos novos com risco: alcançar um host novo com credenciais, chamar um método novo de uma API, tocar os endpoints de metadados da nuvem. Qualquer achado bloqueia a aprovação automática.

---

## 5. De ponta a ponta: rodar um agente e aprovar acessos sob demanda

Este fluxo roda o OpenCode contra um modelo gratuito do OpenRouter (do guia oficial «Run Your First Agent»).

**1. Importe um perfil de provider e crie o provider** — defina `OPENROUTER_API_KEY` antes; o OpenShell a guarda e a envia só para `openrouter.ai`:

```shell
openshell profile import \
  --url https://raw.githubusercontent.com/NVIDIA/OpenShell/main/providers/openrouter.yaml

openshell provider create \
  --name openrouter \
  --type openrouter \
  --from-existing
```

**2. Crie o sandbox com o agente como processo principal:**

```shell
openshell sandbox create \
  --name my-agent \
  --from ghcr.io/anomalyco/opencode:latest \
  --provider openrouter \
  -- opencode -m openrouter/nvidia/nemotron-3.5-lightning:free
```

**3. Aprove acessos novos quando o agente precisar.** Se o agente tenta alcançar um destino que a política não permite, o OpenShell nega a requisição e rascunha uma regra estreita (o policy advisor). O gateway confere o rascunho com o prover, e você decide:

```shell
openshell rule get my-agent --status pending                       # listar propostas pendentes
openshell rule approve my-agent --chunk-id <chunk-id>               # aprovar uma
openshell rule reject my-agent --chunk-id <chunk-id> \
  --reason "Not needed for this task."                              # rejeitar uma, com o motivo
openshell rule approve-all my-agent                                # aprovar todas as pendentes
```

As regras aprovadas entram em vigor sem reiniciar o sandbox: o agente simplesmente tenta a requisição de novo.

---

## 6. Extras

- **Ensine OpenShell ao seu agente de código:** `npx skills add NVIDIA/OpenShell` — skills para operar a CLI, escrever políticas e depurar gateways.
- **SDKs** (conectam a um gateway; não instalam a CLI): Python `uv add openshell` · TypeScript `npm install @nvidia/openshell-sdk` · Go `go get github.com/NVIDIA/OpenShell/sdk/go@latest` · Rust `cargo add openshell-sdk`.
- **Testar a API do gateway:** ela serve gRPC reflection em `localhost:18080` — experimente `grpcurl -plaintext localhost:18080 list`.

---

*Documentação: https://docs.nvidia.com/openshell/latest/index.html · Repo: https://github.com/NVIDIA/OpenShell · Dúvidas: GitHub Discussions.*
