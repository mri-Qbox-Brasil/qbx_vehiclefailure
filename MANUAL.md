# Manual do qbx_vehiclefailure

Sistema avançado de falha de veículos para Qbox — simulação abrangente de danos e falhas com degradação realista, falha em cascata e mecânica de reparo.

## Funcionalidades Principais

### 💥 Danos Realistas
- **Motor**: Dano progressivo com falha em cascata
- **Carroceria**: Deformação visual e funcional
- **Tanque de Combustível**: Dano que pode causar explosão
- Multiplicadores de dano por classe de veículo

### 🔄 Sistema de Falha
- **Degradação**: Veículos degradam lentamente quando danificados
- **Falha em Cascata**: Do menor ao catastrófico
  - Acima de 250: Funcionamento normal
  - 200-250: Degradação lenta (fumaça leve)
  - Abaixo de 200: Falha rápida (fumaça pesada, perda de potência)
  - Abaixo de 50: Motor morre (se limpMode = false)

### 🔧 Reparos
- **Kit de Reparo**: Repara para 500 de saúde (meio reparo)
- **Kit Avançado**: Repara completamente para 1000
- **Kit de Limpeza**: Limpa veículo e remove sujeira
- Animações de reparo incluídas

### 🛞 Furos de Pneu
- Furos aleatórios baseados na direção
- Intervalo configurável
- Furos mais prováveis em terrenos acidentados

### 🚫 Torque do Motor
- Potência reduzida conforme danos aumentam
- Simulação realista de perda de performance

### 🛑 Sistema de Freios
- Freios modificados baseados na configuração SundayDriver
- Resposta escalonada para direção lenta

### 🔒 Modo Manco (Limp Mode)
- Veículo nunca falha completamente (configurável)
- Multiplicador de torque reduzido mantém veículo funcional

### 🎮 Sunday Driver
- Resposta do acelerador escalonada
- Facilita direção em baixa velocidade
- Curvas de acelerador e freio configuráveis

### 🔧 Blips de Mecânico
- Mostra localizações de mecânicos no mapa
- Auxilia jogadores a encontrar reparo

### 🔄 Dano de Componentes
Danos aleatórios a componentes internos:
- Radiador
- Eixo
- Embreagem
- Combustível
- Freios

### 💨 Fumaça do Escape
- Fumaça visual quando motor está danificado
- Intensidade baseada no nível de dano

## Comandos

| Comando | Permissão | Descrição |
|----------|-------------|-------------|
| `/fix` | `group.admin` | Reparar veículo completamente |
| `/repair` | - | Reparar veículo (requer kit de reparo) |
| `/repairfull` | - | Reparar completamente (requer kit avançado) |
| `/clean` | - | Limpar veículo (requer kit de limpeza) |

## Itens Utilizáveis

| Item | Ação | Descrição |
|------|--------|-------------|
| `repairkit` | Reparo meio | Repara para 500 de saúde com animação |
| `advancedrepairkit` | Reparo completo | Repara para 1000 de saúde |
| `cleaningkit` | Limpar | Limpa veículo e remove sujeira |

## Configuração (config.lua)

### Dano Principal
```lua
damageFactorEngine = 3.0          -- Multiplicador de dano do motor
damageFactorBody = 3.0             -- Multiplicador de dano da carroceria
damageFactorPetrolTank = 32.0      -- Multiplicador do tanque
engineDamageExponent = 0.3         -- Compressão da curva de dano
deformationMultiplier = -1         -- Deformação visual (-1 = não tocar)
deformationExponent = 0.5         -- Compressão da curva de deformação
weaponsDamageMultiplier = 1.2      -- Dano de armas
```

### Limites de Falha
```lua
degradingFailureThreshold = 250.0   -- Iniciar degradação lenta
cascadingFailureThreshold = 200.0   -- Iniciar falha rápida
engineSafeGuard = 50.0              -- Saúde mínima (impede fogo)
```

### Velocidade de Degradação
```lua
degradingHealthSpeedFactor = 2.0    -- Velocidade de degradação
cascadingFailureSpeedFactor = 4.0   -- Velocidade de falha em cascata
```

### Alternâncias de Recursos
```lua
torqueMultiplierEnabled = true     -- Reduzir torque conforme dano
limpMode = false                   -- Motor nunca falha (manco)
limpModeMultiplier = 0.15          -- Multiplicador no modo manco
preventVehicleFlip = true          -- Impedir capotamento
sundayDriver = true                -- Resposta escalonada do acelerador
sundayDriverAcceleratorCurve = 7.5 -- Curva do acelerador
sundayDriverBrakeCurve = 5.0       -- Curva do freio
displayBlips = false               -- Mostrar blips de mecânicos
randomTireBurstInterval = 0        -- Minutos entre furos (0=desabilitado)
compatibilityMode = false           -- Não modificar tanque de combustível
```

### Multiplicadores por Classe
```lua
classDamageMultiplier = {
    [0] = 0.3,  -- Compacts
    [1] = 0.3,  -- Sedans
    [2] = 0.3,  -- SUVs
    [3] = 0.5,  -- Coupes
    [4] = 0.5,  -- Muscle
    [5] = 0.7,  -- Sports Classics
    [6] = 0.8,  -- Sports
    [7] = 1.0,  -- Super
    [8] = 0.0,  -- Motorcycles (sem multiplicador)
    -- ... configure por classe
}
```

### Veículos com Motor Traseiro
```lua
BackEngineVehicles = {
    [`ninef`] = true,
    [`adder`] = true,
    [`t20`] = true,
    -- adicione mais...
}
```

### Localizações de Mecânicos
```lua
repairCfg.mechanics = {
    {name='Garage', id=446, r=25.0, x=-337.0, y=-135.0, z=39.0},
    -- adicione mais...
}
```

## Eventos

### Client Events

| Evento | Payload | Descrição |
|-------|----------|-------------|
| `qb-vehiclefailure:client:RepairVehicle` | - | Reparar para 500 |
| `qb-vehiclefailure:client:RepairVehicleFull` | - | Reparar completamente |
| `qb-vehiclefailure:client:CleanVehicle` | - | Limpar veículo |
| `qb-vehiclefailure:client:SyncWash` | `veh` | Sincronizar lavagem |

### Server Events

| Evento | Payload | Descrição |
|-------|----------|-------------|
| `qb-vehiclefailure:removeItem` | `item` | Remover item usado |
| `qb-vehiclefailure:server:removewashingkit` | `veh` | Remover kit e sincronizar |

### Eventos de Compatibilidade
| Evento | Fonte | Descrição |
|-------|--------|-------------|
| `iens:repaira` | Admin | Reparo completo |
| `iens:repair` | Mecânico | Reparo na estrada |
| `iens:besked` | Mecânico | Sem serviço disponível |
| `iens:notAllowed` | Mecânico | Sem permissão |

## Estrutura de Arquivos

```
qbx_vehiclefailure/
├── client/
│   └── main.lua           # Cálculo de danos, reparo, simulação
├── server/
│   └── main.lua           # Itens utilizáveis, comandos
├── config.lua              # Todas as configurações
└── locales/               # Traduções
```

## Dependências

| Dependência | Versão Mínima | Obrigatória |
|------------|-------------------|----------|
| ox_lib | - | ✅ |
| qbx_core | - | ✅ |
| qbx_mechanicjob | - | ✅ (para dano de componentes) |
| cdn-fuel | - | ✅ (configurável) |

## Comportamento de Dano

### Saúde 1000-250
- Funcionamento normal
- Sem efeitos visuais

### Saúde 250-200
- Início de degradação lenta
- Fumaça leve do escape
- Perda mínima de torque

### Saúde 200-50
- Falha em cascata rápida
- Fumaça pesada
- Perda significativa de potência
- Furos de pneu mais frequentes

### Saúde abaixo de 50
- Motor morre (se limpMode = false)
- Fumaça constante
- Veículo inutilizável

### lmpMode ativado
- Motor nunca morre completamente
- Velocidade limitada pelo `limpModeMultiplier`
- Veículo ainda funcional mas lento

## Solução de Problemas

### Veículo não danifica
- Verifique `damageFactorEngine` e `damageFactorBody`
- Confirme que o veículo não está em modo Deus
- Verifique se o recurso está carregado

### Fumaça não aparece
- Verifique se a saúde está abaixo de 250
- Confirme que o veículo tem motor ligado
- Verifique se há erros no console

### Reparo não funciona
- Verifique se tem o item no inventário
- Confirme que o veículo está próximo
- Verifique as animações de reparo

### Furos não acontecem
- Verifique `randomTireBurstInterval`
- Se 0, furos estão desabilitados
- Aumente o intervalo para mais frequência

### Dano de componentes não ocorre
- Verifique se qbx_mechanicjob está rodando
- Confirme que a feature está ativada
- Verifique os logs do servidor

### Capotamento ainda ocorre
- `preventVehicleFlip = true` deve estar configurado
- Reinicie o recurso após alteração
- Verifique se há conflitos com outros recursos
