# Documentação do Projeto — Farmácia

Este documento apresenta o levantamento de requisitos, regras de negócio, fluxos operacionais e visão geral das validações do sistema de gestão de estoque de medicamentos, com base no schema PostgreSQL em `schema_farmacia.sql`.

## Objetivo
- Controlar entradas e saídas de medicamentos por lote (validade), com estoque calculado automaticamente.
- Emitir alertas operacionais (validade, reposição mínima, mês de vencimento iniciado).
- Garantir rastreabilidade (fornecedor, lote do fabricante, paciente, responsável).
- Assegurar controle de acesso por usuários, papéis e permissões.

## Requisitos Funcionais
- Cadastro de medicamentos com: nome padronizado (princípio ativo + dosagem + apresentação), classe terapêutica, tarja, forma de retirada (MIP/prescrição), forma física, apresentação, unidade base, genérico/original, limite mínimo, laboratório e código único do remédio.
- Gestão de lotes por validade (agrupamento mensal), com data de fabricação e validade obrigatórias.
- Entradas: fornecedor obrigatório, data de entrada, número do lote do fabricante, quantidade informada e conversão para unidade base (caixa/frascos → comprimidos/ml).
- Dispensações: paciente identificado (CPF), data da dispensação, responsável (opcional), quantidade informada e conversão para base, bloqueio de lote vencido e de saída acima do saldo.
- Estoque calculado automaticamente (soma das entradas em base – soma das saídas em base).
- Alertas: validade (≤30 dias e vencido), início do mês de vencimento, reposição mínima, <10 unidades, ≤20% do estoque inicial (entradas cumuladas).
- Controle de acesso: permissões por papel e por usuário, checagem considerando usuário ativo.

## Regras de Negócio (Resumo)
- Lote é determinado pela validade mensal: um lote por medicamento/validade_mês (UNIQUE).
- Nenhuma movimentação sem lote com validade.
- Bloqueio de dispensação de lotes vencidos.
- Dispensação só ocorre se houver saldo suficiente.
- Receita obrigatória quando forma de retirada for “com prescrição”.
- Código do medicamento é único, informado manualmente.
- Serial por classe terapêutica gerado automaticamente no cadastro do medicamento.
- Estoque baixo quando saldo ≤ limite mínimo, ou saldo < 10, ou saldo ≤ 20% das entradas.
- Paciente deve ter CPF válido (constraint) e pode ter telefone e cidade.
- Senha de usuário armazenada em hash (bcrypt via pgcrypto).

## Fluxos Operacionais
- Entrada:
  - Registrar fornecedor, lote (válido), número do lote do fabricante, unidade e quantidade informada.
  - Se a unidade informada não for a unidade base do medicamento, informar `unidades_por_embalagem` para conversão a `quantidade_base` (trigger calcula).
- Saída:
  - Selecionar paciente (CPF válido), lote, unidade e quantidade informada.
  - Sistema converte para `quantidade_base` e verifica saldo e validade (trigger bloqueia se vencido/insuficiente). 
  - Se medicamento exigir prescrição, informar `numero_receita`.
- Alertas:
  - Validade (≤30 dias ou vencido) via `vw_alerta_validade`.
  - Início do mês de vencimento via `vw_alerta_validade_mes_atual`.
  - Estoque baixo via `vw_alerta_estoque_baixo`.
- Arquivamento:
  - Funções desativam lote/medicamento quando sem saldo ou vencido (preserva histórico).

## Controle de Acesso (RBAC)
- Usuários (ativos/inativos) com login e email únicos; senha hash.
- Papéis agregam permissões (ex.: admin, estoquista, atendente, analista).
- Permissões básicas: `acesso_sistema`, `relatorios`, `entradas`, `saidas`, `estoque`.
- View `vw_usuarios_permissoes_efetivas` consolida permissões diretas + via papel.
- Função `fn_usuario_tem_permissao(usuario_id, codigo)` retorna TRUE somente para usuário ativo com a permissão.

## Views e Funções Principais
- Views:
  - `vw_estoque_por_lote`: saldo, dias para vencimento e status (OK/Próximo de vencer/Bloquear dispensação).
  - `vw_estoque_por_medicamento`: saldo consolidado e alertas de reposição.
  - `vw_alerta_validade`: próximos de vencer e vencidos.
  - `vw_alerta_validade_mes_atual`: lotes cujo mês de vencimento é o mês corrente.
  - `vw_alerta_estoque_baixo`: consolida condições de estoque baixo.
  - `vw_usuarios_permissoes_efetivas`: permissões do usuário.
- Funções e Triggers:
  - `fn_set_serial_por_classe` + `trg_set_serial_por_classe`: serial sequencial por classe para medicamentos.
  - `fn_calc_quantidade_base_entrada` + `trg_calc_quantidade_base_entrada`: conversão de entrada para unidade base.
  - `fn_calc_quantidade_base_dispensacao` + `trg_calc_quantidade_base_dispensacao`: conversão de saída para base.
  - `fn_check_saldo_e_validade_dispensacao` + `trg_check_saldo_e_validade_dispensacao`: bloqueios de vencimento e saldo; receita obrigatória.
  - `fn_arquivar_lote_se_sem_saldo_ou_vencido` e `fn_arquivar_medicamento_se_sem_saldo`: exclusão lógica.
  - `fn_usuario_tem_permissao`: checagem de permissão.
  - `fn_cpf_valido` + `CHECK` em `pacientes.cpf`: validação de CPF.

## Execução
- Execute `schema_farmacia.sql` no PostgreSQL (ex.: `psql -f schema_farmacia.sql`).
- Cadastre classes terapêuticas com `codigo_classe`, laboratórios e fornecedores.
- Cadastre medicamentos (serial da classe é gerado automaticamente).
- Lance lotes (com data de fabricação/validade) e entradas.
- Cadastre pacientes e faça dispensações.
- Configure usuários, papéis e permissões (seeds de permissões já incluídos).

## Dicionário de Dados
- Consulte `docs/dicionario_de_dados.md` para detalhes de campos, tipos, FKs, unicidades e validações de cada tabela, view e função.