# goService - Gerenciador de Protocolos

Este serviço é responsável por automatizar a criação e gerenciamento de protocolos das contas do Tasy, seguindo regras específicas do faturamento para agrupamento e processamento.

## Fluxo 

### 1. Busca de Contas Elegíveis

Importante: Categoria PRÉ de contrato é chamada de VD.

- Consulta inicial na base do Tasy para encontrar contas que atendam aos critérios:
  - Status provisório ou definitivo (c.ie_status_acerto IN (1, 2))
  - Atendimento não cancelado (a.dt_cancelamento IS NULL), e conta válida ((c.ie_cancelamento != 'C' OR c.ie_cancelamento IS NULL OR c.ie_cancelamento != 'E' ))
  - Atendimento com alta registrada (a.dt_alta IS NOT NULL)
  - Sem protocolo associado (c.nr_seq_protocolo IS NULL OR c.nr_seq_protocolo = 0)
  - Validar carteirinha do beneficiário (precisa estar preenchida e ter 17 dígitos)
  - Cadastro médico (Evita plantonista ou médicos sem Conselho, CBO, CRM e UF)
  - Valida etapa da conta (Function hu_valida_etapa_conta)
    - Contas nas etapas "Pendências (Laboratório)", "Pendências (Faturamento)" e "Pendente Autorização" não são elegíveis.
    - Verificar as nr_seq em Cadastros Gerais -> "Etapas para Faturamento conta", atualmente são as sequências 85, 86, 87, 67, 82, 77 e 73
  - Conta não pode ter honorário médico
  - Atendimento não pode ter exames de genética
  - Atendimento/conta com valor positivo
  - Atendimentos de CDI não podem ter procedimentos da especialidade 402 (Estrutura -> Capítulo 4 -> Especialidade 402 -> Grupos de procedimento 40201 e 40202)
  - Validar senha de atendimento (precisa estar igual em todos os campos)
    - Após, pesquisa no banco do Cardio se a senha está autorizada (Parecer = 1)
    - Também valida o cadastro do convênio na EUP para bater com o Modelo de Contrato do Cardio, da seguinte maneira:
      - Intercâmbio Nacional no cardio: Modelo 147 | na EUP deve estar cadastrado como Intercâmbio, categoria Nacional (acc.cd_categoria = 1)
      - Intercâmbio Estadual no cardio: Modelo 19 | na EUP deve estar cadastrado como Intercâmbio, categoria Estadual (acc.cd_categoria = 2)
      - Unimed Noroeste/RS Cardio: Modalidade de cobrança 1| na EUP deve estar cadastrado como Unimed, categoria VD (acc.cd_categoria = 1)
      - Unimed Noroeste/RS Cardio: Modalidade de cobrança 2| na EUP deve estar cadastrado como Unimed, categoria PÓS (acc.cd_categoria = 2)
- JSON com contas disponível em /api/eligible-accounts
- Tickets em referência: 526288 e 537661

### 2. Agrupa as Contas
- As contas são agrupadas por características comuns:
  - Estabelecimento (HU, PB, FW)
  - Tipo de Clínica (CDI ou LAB para Hospital)
  - Convênio e suas regras específicas
    - Intercâmbio (IT): Nacional/Estadual
    - Unimed Noroeste RS: VD/POS
- Cada grupo gera um tipo específico de protocolo (ex: HU_CDI, PB_PRE)

### 3. Processamento de Protocolos
- Para cada grupo de contas:
  - Divide em blocos de exatamente 30 contas
    - Os protocolos de exames são enviados até o dia 22, então, caso o protocolo não alcance 30 contas, ele será criado no dia 21 (ou dia 20, se 22 for final de semana) entre as 18:00:00 e 23:59:59 com qualquer quantidade de contas incluídas 
  - Contas excedentes aguardam próxima execução
  - Cada bloco gera um protocolo único
  - Nomenclatura: K_[TIPO]_[SEQUENCIA] (ex: K_PB_VD_123)

### 4. Criação de Protocolos
- Para cada bloco de 30 contas:
  - Gera número sequencial único
  - Cria registro do protocolo
  - Vincula as contas ao protocolo
  - Atualiza status das contas

## Instruções de Uso

### Build
```bash
# Na raiz do projeto
go build -o goservice cmd/server/main.go
```

### Execução
```bash
# Modo desenvolvimento
go run cmd/server/main.go

# Modo produção
./goservice
```

### Logs
- Os logs são gerados em tempo real durante a execução
- Principais eventos registrados:
  - Início e fim de cada operação
  - Contas processadas e seus status
  - Protocolos criados e quantidade de contas
  - Erros e exceções
  - Contas que aguardam próxima execução

Para visualizar os logs:
```bash
# Em tempo real
journalctl -u goservice -f
# Filtrar por tipo de operação
grep "Protocolo criado" goservice.log
```

## Regras de Faturamento importantes

### Agrupamento de Contas
- Cada protocolo deve ter exatamente 30 contas (ou menos, se não atingir o número mínimo antes do dia 22)
- Contas são agrupadas por características comuns
- Contas excedentes aguardam próxima execução

### Nomenclatura de Protocolos
- Formato: K_[TIPO]_[SEQUENCIA] (K Serve para identificar protocolos gerados pelo goService)
- Exemplos:
  - K_HU_CDI_123
  - K_PB_VD_124
  - K_FW_LAB_125

### Validações
- Checar:  1. Busca de Contas Elegíveis

## Troubleshooting

### Problemas Comuns
1. Conexão com banco de dados
   - Verificar strings de conexão
   - Validar credenciais   

2. Contas não processadas
   - Verificar logs de validação
   - Confirmar status de autorização   
   - Verificar se cadastro do convênio na EUP bate com Cardio

3. Protocolos não criados
   - Verificar quantidade de contas no grupo
   - Confirmar sequência de numeração
   - Checar permissões de banco
