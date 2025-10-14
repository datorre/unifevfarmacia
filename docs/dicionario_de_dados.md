# Dicionário de Dados — Farmácia

Este dicionário descreve estruturas, campos, tipos, constraints, validações, triggers e regras de negócio por tabela, de acordo com o schema em `schema_farmacia.sql`.

## Tipos Enumerados
- `tarja_tipo`: `sem_tarja`, `tarja_amarela`, `tarja_vermelha`, `tarja_preta`.
- `forma_retirada_tipo`: `MIP`, `com_prescricao`.
- `forma_fisica_tipo`: `solida`, `pastosa`, `liquida`, `gasosa`.
- `unidade_contagem`: `comprimido`, `capsula`, `dragea`, `sache`, `ampola`, `frasco`, `caixa`, `ml`, `g`, `unidade`, `aerosol`, `xarope`, `solucao`.
- `estado_item`: `novo`, `lacrado`, `aberto`, `avariado`.
- `fornecedor_tipo`: `doacao`, `compra`, `parceria`, `outros`.

## Tabelas

### laboratorios
- `id` BIGSERIAL PK
- `nome` TEXT NOT NULL UNIQUE
Regras: Cadastro de fabricantes/laboratórios.

### classes_terapeuticas
- `id` BIGSERIAL PK
- `codigo_classe` SMALLINT NOT NULL UNIQUE
- `nome` TEXT NOT NULL UNIQUE
Regras: Código fixo por classe; usado para serial por classe em `medicamentos`.

### fornecedores
- `id` BIGSERIAL PK
- `nome` TEXT NOT NULL
- `tipo` fornecedor_tipo NOT NULL DEFAULT `doacao`
- `contato` TEXT
Regras: Cadastro de fornecedores (doação, compra, parceria).

### pacientes
- `id` BIGSERIAL PK
- `nome` TEXT NOT NULL
- `cpf` TEXT NOT NULL UNIQUE (CHECK `fn_cpf_valido(cpf)`)
- `telefone` TEXT
- `cidade` TEXT
Regras: CPF obrigatório e válido; identifica dispensações.

### medicamentos
- `id` BIGSERIAL PK
- `codigo` TEXT NOT NULL UNIQUE (informado manualmente)
- `nome` TEXT NOT NULL (padrão: princípio ativo + dosagem + apresentação)
- `laboratorio_id` BIGINT FK `laboratorios(id)` ON DELETE RESTRICT
- `classe_terapeutica_id` BIGINT NOT NULL FK `classes_terapeuticas(id)` ON DELETE RESTRICT
- `tarja` tarja_tipo NOT NULL
- `forma_retirada` forma_retirada_tipo NOT NULL
- `forma_fisica` forma_fisica_tipo NOT NULL
- `apresentacao` unidade_contagem NOT NULL
- `unidade_base` unidade_contagem NOT NULL (unidade para contagem de estoque)
- `dosagem_valor` NUMERIC(12,3) NOT NULL
- `dosagem_unidade` TEXT NOT NULL (ex.: mg, ml)
- `generico` BOOLEAN NOT NULL DEFAULT FALSE
- `limite_minimo` NUMERIC(12,3) NOT NULL DEFAULT 0
- `serial_por_classe` INTEGER NOT NULL (gerado por trigger)
- `ativo` BOOLEAN NOT NULL DEFAULT TRUE
Índices: por classe, tarja e genérico.
Regras: `serial_por_classe` sequencial por classe; `limite_minimo` usado em alertas.

### lotes
- `id` BIGSERIAL PK
- `medicamento_id` BIGINT NOT NULL FK `medicamentos(id)` ON DELETE RESTRICT
- `data_fabricacao` DATE NOT NULL
- `validade` DATE NOT NULL
- `validade_mes` DATE GENERATED ALWAYS AS (date_trunc('month', validade)::date) STORED
- `nome_comercial` TEXT
- `ativo` BOOLEAN NOT NULL DEFAULT TRUE
- `observacao` TEXT
- UNIQUE (medicamento_id, validade_mes)
Índices: por `validade` e `validade_mes`.
Regras: Um lote por mês de validade para cada medicamento; base para alertas mensais.

### entradas
- `id` BIGSERIAL PK
- `data_entrada` DATE NOT NULL DEFAULT CURRENT_DATE
- `fornecedor_id` BIGINT NOT NULL FK `fornecedores(id)` ON DELETE RESTRICT
- `lote_id` BIGINT NOT NULL FK `lotes(id)` ON DELETE RESTRICT
- `numero_lote_fornecedor` TEXT NOT NULL (UNIQUE por `lote_id`)
- `quantidade_informada` NUMERIC(12,3) NOT NULL CHECK (>0)
- `quantidade_base` NUMERIC(12,3) NOT NULL CHECK (>0) (calculado por trigger)
- `unidade` unidade_contagem NOT NULL
- `unidades_por_embalagem` NUMERIC(12,3) (usada na conversão quando `unidade` ≠ `unidade_base`)
- `estado` estado_item (opcional)
- `observacao` TEXT
- UNIQUE (lote_id, numero_lote_fornecedor)
Trigger: `trg_calc_quantidade_base_entrada` (converte para unidade base, exige `unidades_por_embalagem` quando necessário).
Regras: Entrada sempre referindo lote válido e fornecedor definido.

### dispensacoes
- `id` BIGSERIAL PK
- `data_dispensa` TIMESTAMP NOT NULL DEFAULT NOW()
- `responsavel` TEXT (opcional)
- `paciente_id` BIGINT NOT NULL FK `pacientes(id)` ON DELETE RESTRICT
- `lote_id` BIGINT NOT NULL FK `lotes(id)` ON DELETE RESTRICT
- `dosagem` TEXT (opcional; a dosagem padrão está em `medicamentos`)
- `nome_comercial` TEXT (opcional)
- `quantidade_informada` NUMERIC(12,3) NOT NULL CHECK (>0)
- `quantidade_base` NUMERIC(12,3) NOT NULL CHECK (>0) (calculado por trigger)
- `unidade` unidade_contagem NOT NULL
- `numero_receita` TEXT (obrigatório se `forma_retirada` = `com_prescricao`)
Triggers:
- `trg_calc_quantidade_base_dispensacao` (conversão para base usando última referência de embalagem do lote).
- `trg_check_saldo_e_validade_dispensacao` (bloqueia vencido/sem saldo; exige receita quando necessário).
Regras: Saída somente com saldo existente; validações operacionais pelo trigger.

### usuarios
- `id` BIGSERIAL PK
- `nome` TEXT NOT NULL
- `celular` TEXT
- `email` TEXT NOT NULL UNIQUE
- `login` TEXT NOT NULL UNIQUE
- `senha_hash` TEXT NOT NULL (bcrypt via trigger)
- `datacadastro` TIMESTAMP NOT NULL DEFAULT NOW()
- `ultimoacesso` TIMESTAMP
- `ativo` BOOLEAN NOT NULL DEFAULT TRUE
Trigger: `trg_hash_senha_usuarios` (hash automático de senha quando não vier hashada).
Regras: Usuários inativos não possuem acesso mesmo com permissões.

### permissoes
- `id` BIGSERIAL PK
- `codigo` TEXT NOT NULL UNIQUE (ex.: `acesso_sistema`, `relatorios`, `entradas`, `saidas`, `estoque`)
- `nome` TEXT NOT NULL
Seeds: permissões básicas inseridas com `ON CONFLICT DO NOTHING`.

### papeis
- `id` BIGSERIAL PK
- `nome` TEXT NOT NULL UNIQUE
- `descricao` TEXT
Regras: agregam permissões.

### papeis_permissoes
- `id` BIGSERIAL PK
- `papel_id` BIGINT NOT NULL FK `papeis(id)` ON DELETE CASCADE
- `permissao_id` BIGINT NOT NULL FK `permissoes(id)` ON DELETE CASCADE
- UNIQUE (papel_id, permissao_id)
Regras: define o conjunto de permissões de um papel.

### usuarios_papeis
- `id` BIGSERIAL PK
- `usuario_id` BIGINT NOT NULL FK `usuarios(id)` ON DELETE CASCADE
- `papel_id` BIGINT NOT NULL FK `papeis(id)` ON DELETE CASCADE
- UNIQUE (usuario_id, papel_id)
Regras: vincula usuário aos papéis.

### usuarios_permissoes
- `id` BIGSERIAL PK
- `usuario_id` BIGINT NOT NULL FK `usuarios(id)` ON DELETE CASCADE
- `permissao_id` BIGINT NOT NULL FK `permissoes(id)` ON DELETE CASCADE
- UNIQUE (usuario_id, permissao_id)
Regras: concessão direta ao usuário (override).

## Views
- `vw_estoque_por_lote`:
  - Campos: lote_id, medicamento_id, medicamento, codigo, generico, tarja, forma_retirada, forma_fisica, apresentacao, unidade_base, dosagem_valor, dosagem_unidade, data_fabricacao, validade, validade_mes, quantidade_entrada, quantidade_saida, quantidade_disponivel, dias_para_vencimento, status.
  - Status: `OK`, `Próximo de vencer` (<=30 dias), `Bloquear dispensação` (vencido).
- `vw_estoque_por_medicamento`:
  - Consolidado por medicamento: entrada, saída, saldo, limite_mínimo, alertas (`alerta_minimo`, `<10 unidades`, `≤20%`).
- `vw_alerta_validade`:
  - Filtra lotes com status `Próximo de vencer` ou `Bloquear dispensação`.
- `vw_alerta_validade_mes_atual`:
  - Lotes cujo `validade_mes` coincide com o mês atual.
- `vw_alerta_estoque_baixo`:
  - Consolida alertas de estoque baixo.
- `vw_usuarios_permissoes_efetivas`:
  - Permissões efetivas do usuário (diretas + via papel).

## Funções e Triggers
- `fn_set_serial_por_classe` + `trg_set_serial_por_classe`: define `serial_por_classe` no INSERT de `medicamentos` com base no `codigo_classe` da sua classe.
- `fn_calc_quantidade_base_entrada` + `trg_calc_quantidade_base_entrada`: converte entradas para `unidade_base`; exige `unidades_por_embalagem` quando unidade difere.
- `fn_calc_quantidade_base_dispensacao` + `trg_calc_quantidade_base_dispensacao`: converte saídas para base; usa a última referência de embalagem da entrada para o lote e unidade.
- `fn_check_saldo_e_validade_dispensacao` + `trg_check_saldo_e_validade_dispensacao`:
  - Bloqueia dispensação de lotes vencidos e de saídas acima do saldo.
  - Exige `numero_receita` quando `forma_retirada` = `com_prescricao`.
- `fn_arquivar_lote_se_sem_saldo_ou_vencido`:
  - Atualiza `lotes.ativo = FALSE` quando saldo ≤ 0 ou vencido.
- `fn_arquivar_medicamento_se_sem_saldo`:
  - Atualiza `medicamentos.ativo = FALSE` quando saldo consolidado ≤ 0.
- `fn_usuario_tem_permissao(usuario_id, codigo)`:
  - Retorna TRUE somente quando o usuário está ativo e possui a permissão.
- `fn_cpf_valido` + `CHECK` em `pacientes.cpf`:
  - Valida CPF (dígitos, tamanho 11, não repetido, dígitos verificadores oficiais).
- `trg_hash_senha_usuarios`:
  - Aplica `crypt(..., gen_salt('bf'))` quando `senha_hash` não vier hashada.

## Índices
- `idx_medicamentos_classe`, `idx_medicamentos_tarja`, `idx_medicamentos_generico`.
- `idx_lotes_validade`, `idx_lotes_validade_mes`.
- `idx_entradas_lote`, `idx_dispensacoes_lote`, `idx_dispensacoes_data`.
- `idx_usuarios_login`, `idx_usuarios_email`, `idx_usuarios_ultimoacesso`, `idx_usuarios_ativo`.
- `idx_pacientes_cpf`, `idx_fornecedores_tipo`.

## Validações por Campo (Resumo)
- Quantidades: `quantidade_informada` e `quantidade_base` com `CHECK (>0)` em entradas e saídas.
- CPF: `CHECK (fn_cpf_valido(cpf))` e `UNIQUE` em `pacientes`.
- Unicidades importantes: `medicamentos.codigo`, `classes_terapeuticas.codigo_classe`, `laboratorios.nome`, `classes_terapeuticas.nome`, `permissoes.codigo`, `papeis.nome`, `usuarios.login`, `usuarios.email`.
- Lote mensal: `UNIQUE (medicamento_id, validade_mes)` garante um agrupamento de lote por mês de validade por medicamento.
- Lote do fabricante: `UNIQUE (lote_id, numero_lote_fornecedor)` evita duplicidade por lote.
- Datas obrigatórias: `data_entrada` em entradas; `data_dispensa` em saídas.
- Prescrição: `numero_receita` obrigatório quando `medicamentos.forma_retirada = 'com_prescricao'` (validação no trigger).

## Observações de Modelagem
- Estoque é derivado de movimentos (views), não armazenado em coluna — mais seguro contra inconsistências.
- Conversão de unidades mantém o saldo na unidade base do medicamento (padroniza cálculo e alertas).
- Exclusão lógica preserva histórico; pode ser acionada por rotina administrativa usando as funções de arquivamento.