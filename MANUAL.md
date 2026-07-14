# qbx_vehiclefailure — Manual

Sistema de dano e falha progressiva de veículos para Qbox: degradação lenta, falha em cascata, perda de torque, prevenção de capotamento e reparo por itens.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Permissões (ACE)](#permissões-ace)
4. [Configuração](#configuração)
5. [Comandos](#comandos)
6. [Itens utilizáveis](#itens-utilizáveis)
7. [Curva de falha do motor](#curva-de-falha-do-motor)
8. [Integrações](#integrações)
9. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
10. [Localização](#localização)
11. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | Framework base, `CreateUseableItem`, `Notify`, módulos `lib.lua` e `playerdata.lua` |
| `ox_lib` | Sim | `lib.addCommand`, `lib.progressBar`, `lib.getClosestVehicle`, locale |
| `qbx_mechanicjob` | Sim | Chamado sem verificação de existência em `damageRandomComponent` (`SetVehicleStatus` / `GetVehicleStatus`). Sem ele, o dano de componentes gera erro no client |
| Script de combustível | Sim | O nome do recurso vem de `Config.FuelScript` (padrão `cdn-fuel`). O export `SetFuel` é chamado no reparo admin (`iens:repaira`) |

---

## Instalação

1. Copie a pasta `qbx_vehiclefailure` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_vehiclefailure
   ```
3. Ajuste `Config.FuelScript` em `config.lua` para o nome do recurso de combustível que você usa. Se o recurso apontado não existir, o comando `/fix` falha ao chamar `SetFuel`.
4. Cadastre os itens `repairkit`, `advancedrepairkit` e `cleaningkit` no seu inventário (`ox_inventory` ou equivalente). Eles são registrados como usáveis pelo servidor, mas a definição do item em si é do inventário.
5. Não há SQL.
6. **Conflitos** — não rode junto com outros scripts de dano de veículo (`qb-vehiclefailure`, BVA, LegacyFuel damage). Todos escrevem em `SetVehicleEngineHealth` e nos handling floats do mesmo veículo. O `cfg.compatibilityMode` existe justamente para reduzir atrito com scripts que mexem na saúde do tanque de combustível.

---

## Permissões (ACE)

O comando `/fix` é registrado com `restricted = 'group.admin'`. Garanta o ACE no `server.cfg`:

```
add_ace group.admin command.fix allow
```

---

## Configuração

Tudo fica em `config.lua`, dividido em quatro blocos: `Config`, `cfg`, `repairCfg` e `BackEngineVehicles`.

### `Config`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.FuelScript` | string | Sim | Nome do recurso de combustível cujo export `SetFuel` é chamado no reparo admin. Padrão: `cdn-fuel` |
| `Config.Paid` | bool | Não | Declarado no config, mas **não é lido por nenhum arquivo do recurso** na versão atual |
| `Config.Price` | number | Não | Declarado no config, mas **não é lido por nenhum arquivo do recurso** na versão atual |

### `cfg` — comportamento do dano

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `deformationMultiplier` | number | Não | Multiplicador da deformação visual da colisão. `0.0` a `10.0`. `-1` não altera o handling. Dano visual não sincroniza bem entre jogadores |
| `deformationExponent` | number | Não | Expoente que comprime `fDeformationDamageMult` do handling em direção a `1.0`, aproximando o comportamento entre carros. `1` = sem mudança. Nunca use zero ou negativo |
| `collisionDamageExponent` | number | Não | Mesmo efeito, aplicado a `fCollisionDamageMult` |
| `engineDamageExponent` | number | Não | Mesmo efeito, aplicado a `fEngineDamageMult` |
| `damageFactorEngine` | number | Não | Multiplicador do dano ao motor. Faixa sã: 1 a 100 |
| `damageFactorBody` | number | Não | Multiplicador do dano à carroceria. Faixa sã: 1 a 100 |
| `damageFactorPetrolTank` | number | Não | Multiplicador do dano ao tanque. Faixa sã: 1 a 200 |
| `weaponsDamageMultiplier` | number | Não | Dano recebido de armas. `0.0` a `10.0`. `-1` não altera o handling |
| `degradingHealthSpeedFactor` | number | Não | Velocidade da degradação lenta de saúde. Valores maiores degradam mais rápido |
| `cascadingFailureSpeedFactor` | number | Não | Velocidade da falha em cascata quando a saúde cai abaixo do limiar |
| `degradingFailureThreshold` | number | Não | Abaixo deste valor de saúde do motor começa a degradação lenta |
| `cascadingFailureThreshold` | number | Não | Abaixo deste valor começa a falha em cascata |
| `engineSafeGuard` | number | Não | Piso da saúde do motor. Muito alto e o carro não solta fumaça ao morrer; muito baixo e pega fogo com um tiro no motor |
| `torqueMultiplierEnabled` | bool | Não | Reduz o torque do motor conforme ele se danifica |
| `limpMode` | bool | Não | Quando `true`, o motor nunca morre por completo — sempre dá para chegar ao mecânico |
| `limpModeMultiplier` | number | Não | Multiplicador de torque aplicado no modo manco. Faixa sã: 0.05 a 0.25 |
| `preventVehicleFlip` | bool | Não | Quando `true`, impede desvirar um veículo capotado (bloqueia os controles 59 e 60 com roll acima de 75 graus e velocidade abaixo de 2) |
| `sundayDriver` | bool | Não | Escalona a resposta do acelerador e do freio para facilitar direção lenta. Não funciona com acelerador binário (teclado); o segurar do freio e o "parar sem dar ré" funcionam mesmo no teclado |
| `sundayDriverAcceleratorCurve` | number | Não | Curva de resposta do acelerador. `0.0` a `10.0`. Sem efeito no teclado |
| `sundayDriverBrakeCurve` | number | Não | Curva de resposta do freio. `0.0` a `10.0`. Sem efeito no teclado |
| `displayBlips` | bool | Não | Mostra blips das oficinas listadas em `repairCfg.mechanics` no mapa |
| `compatibilityMode` | bool | Não | Impede que o recurso force a saúde do tanque de combustível, evitando falha aleatória do motor com scripts como BVA 2.01. Desativa a prevenção de explosão |
| `randomTireBurstInterval` | number | Não | Minutos (estatísticos) dirigindo acima de ~22 mph até um furo de pneu aleatório. `0` desativa o recurso |
| `classDamageMultiplier` | tabela | Sim | Multiplicador aplicado a `damageFactorEngine`, `damageFactorBody` e `damageFactorPetrolTank` por classe de veículo (índices `0` a `21`, seguindo as classes do GTA V). Motocicletas (`8`) vêm com `0.0` no padrão, ou seja, sem dano multiplicado |

O arquivo traz ainda um segundo bloco `cfg` comentado (configuração alternativa, mais tolerante a dano). Para usá-lo, comente o bloco ativo e descomente esse.

### `repairCfg`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `mechanics` | array | Sim | Lista de oficinas. Cada entrada tem `name` (nome do blip), `id` (sprite do blip — 446 é a chave inglesa, 72 a lata de spray), `r` (raio em metros) e `x`, `y`, `z` |
| `fixMessageCount` | number | Sim | Quantidade de mensagens de sucesso rotativas do reparo de beira de estrada. Precisa bater com as chaves `success.fix_message_1..N` dos locales |
| `noFixMessageCount` | number | Sim | Quantidade de mensagens de recusa rotativas. Precisa bater com as chaves `error.nofix_message_1..N` dos locales |

O raio das oficinas serve para bloquear o reparo de beira de estrada (`iens:repair`): dentro de uma oficina, o evento não faz nada.

### `BackEngineVehicles`

Tabela de hashes de modelo (`` [`ninef`] = true ``) usada para decidir qual porta abrir e de que lado o jogador precisa estar durante o reparo. Veículos listados abrem o capô traseiro (porta 5); os demais abrem o dianteiro (porta 4). Adicione seus modelos custom aqui.

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/fix` | `command.fix` (registrado como `group.admin`) | Repara totalmente o veículo do jogador: limpa, conserta, religa o motor, zera a sujeira e enche o tanque via `Config.FuelScript` |

---

## Itens utilizáveis

Registrados no servidor via `exports.qbx_core:CreateUseableItem`. O jogador precisa estar **fora** do veículo e a até 2 metros do capô (dianteiro ou traseiro, conforme `BackEngineVehicles`).

| Item | Efeito | Progresso |
|---|---|---|
| `repairkit` | Repara o motor para 500 e conserta os 5 pneus. Recusa se a saúde do motor já for maior ou igual a 500 | 10 a 20 segundos, animação `mini@repair` |
| `advancedrepairkit` | Repara o motor para 1000 e conserta os 5 pneus. Sem verificação de saúde mínima | 20 a 30 segundos, animação `mini@repair` |
| `cleaningkit` | Zera a sujeira e remove decals do veículo mais próximo (até 3 metros). Sincroniza a lavagem para todos os clientes | 10 a 20 segundos, cenário `WORLD_HUMAN_MAID_CLEAN` |

O item só é consumido quando a barra de progresso completa. Cancelar aborta sem gastar.

---

## Curva de falha do motor

Com os valores padrão de `cfg`:

| Faixa de saúde do motor | Comportamento |
|---|---|
| Acima de `degradingFailureThreshold` (250) | Sem degradação passiva. A saúde só cai com dano real, multiplicado pelos `damageFactor*` e pelo `classDamageMultiplier` da classe |
| Entre `cascadingFailureThreshold` (200) e 250 | Degradação lenta contínua enquanto o jogador dirige, controlada por `degradingHealthSpeedFactor` |
| Abaixo de `cascadingFailureThreshold` (200) | Falha em cascata: a saúde despenca em ritmo de `cascadingFailureSpeedFactor` até bater no piso |
| No `engineSafeGuard` (50) | Veículo fica `undriveable` e emite fogo (`ent_ray_heli_aprtmnt_l_fire`). Com `limpMode = true`, o motor não morre e o torque cai para `limpModeMultiplier` |

Enquanto o jogador dirige, o dano de motor, carroceria e tanque é medido a cada 50 ms; o maior dos três deltas escalados vira a perda de saúde do motor. Toda perda relevante também danifica um componente aleatório entre `radiator`, `axle`, `clutch`, `fuel` e `brakes` no `qbx_mechanicjob`.

Com `compatibilityMode = false` (padrão), a saúde do tanque é mantida no mínimo em 750 para prevenir explosões.

---

## Integrações

### qbx_mechanicjob

A cada perda relevante de saúde de motor ou carroceria, o recurso degrada um componente aleatório do veículo:

```lua
exports.qbx_mechanicjob:SetVehicleStatus(plate, component, currentValue - randomDamage)
```

Os componentes afetados são `radiator`, `axle`, `clutch`, `fuel` e `brakes`. É o que conecta a direção agressiva ao trabalho de mecânico.

### Script de combustível

O reparo admin (`/fix` → `iens:repaira`) enche o tanque chamando `exports[Config.FuelScript]:SetFuel(vehicle, 100.0)`. Qualquer recurso de combustível que exponha `SetFuel(vehicle, level)` serve — basta apontar `Config.FuelScript` para ele.

### Aviões, helicópteros, bicicletas e trens

O reparo de beira de estrada (`iens:repair`) ignora as classes 15 (helicópteros), 16 (aviões), 21 (trens) e 13 (bicicletas). O sistema de dano em si roda em qualquer veículo em que o jogador esteja no assento de motorista.

---

## Entrypoints para outros recursos

O recurso não exporta funções. A integração é toda por eventos.

### Eventos de client

```lua
-- Repara o veículo mais próximo para 500 de saúde. Consome um repairkit.
TriggerClientEvent('qb-vehiclefailure:client:RepairVehicle', source)

-- Repara o veículo mais próximo para 1000 de saúde. Consome um advancedrepairkit.
TriggerClientEvent('qb-vehiclefailure:client:RepairVehicleFull', source)

-- Limpa o veículo mais próximo. Consome um cleaningkit.
TriggerClientEvent('qb-vehiclefailure:client:CleanVehicle', source)

-- Sincroniza a lavagem de um veículo para os demais clientes.
TriggerClientEvent('qb-vehiclefailure:client:SyncWash', -1, vehicleNetId)
```

### Eventos legacy `iens:*`

Compatibilidade com o fluxo original do `vehiclefailure`. Todos são client events.

| Evento | Efeito |
|---|---|
| `iens:repaira` | Reparo total do veículo em que o jogador está (usado pelo `/fix`). Enche o tanque |
| `iens:repair` | Reparo de beira de estrada. Só funciona fora do raio de uma oficina, com o motor abaixo de `cascadingFailureThreshold + 5` e óleo acima de zero. Restaura a saúde para `cascadingFailureThreshold + 5` e consome 2/3 do óleo |
| `iens:besked` | Notifica que o serviço de guincho está disponível |
| `iens:notAllowed` | Notifica falta de permissão |

### Eventos de servidor

```lua
-- Remove 1 unidade de um item do inventário do jogador.
TriggerServerEvent('qb-vehiclefailure:removeItem', 'repairkit')

-- Remove 1 cleaningkit e faz o broadcast da lavagem.
TriggerServerEvent('qb-vehiclefailure:server:removewashingkit', vehicle)
```

---

## Localização

As notificações e labels das barras de progresso são traduzidas via `ox_lib` locale. Os arquivos ficam em `locales/`:

`ar.json`, `cs.json`, `da.json`, `de.json`, `en.json`, `es.json`, `fr.json`, `it.json`, `pt-br.json`, `pt.json`, `sv.json`

O locale ativo é definido pela convar `ox:locale` no `server.cfg`:

```
setr ox:locale "pt-br"
```

Ao adicionar um idioma, mantenha as chaves `success.fix_message_1..N` e `error.nofix_message_1..N` alinhadas com `repairCfg.fixMessageCount` e `repairCfg.noFixMessageCount`.

---

## Estrutura de arquivos

```
qbx_vehiclefailure/
├── client/
│   └── main.lua          — loop de dano, torque, sunday driver, anti-flip, reparo e limpeza
├── server/
│   └── main.lua          — comando /fix, itens usáveis, remoção de itens
├── config.lua            — Config, cfg (dano), repairCfg (oficinas) e BackEngineVehicles
├── locales/
│   ├── ar.json
│   ├── cs.json
│   ├── da.json
│   ├── de.json
│   ├── en.json
│   ├── es.json
│   ├── fr.json
│   ├── it.json
│   ├── pt-br.json
│   ├── pt.json
│   └── sv.json
└── fxmanifest.lua
```
