# ABAP Cloud: Parallel Processing com `CL_ABAP_PARALLEL` (guia prático)

> Referência principal:  
> - Software-Heroes: [*ABAP Cloud – Parallel processing* (2024-04-12)](https://sachinartani.com/blog/parallel-processing-in-abap-cloud)  
> - Sachin Artani: [*Parallel Processing in ABAP Cloud Using CL_ABAP_PARALLEL and RAP EML* (atualizado em 2026-02-05)](https://sachinartani.com/blog/parallel-processing-in-abap-cloud)

---

## 1) O que é `CL_ABAP_PARALLEL`?

`CL_ABAP_PARALLEL` é uma classe/framework para **executar várias “tarefas” ABAP em paralelo**, distribuindo pacotes de trabalho em múltiplos work processes. A ideia é simples:

- você divide um volume grande (ex.: muitos registros) em “pacotes”
- cada pacote roda em uma tarefa separada (work process / sessão separada)
- você coleta os resultados no final

Nos artigos, a classe é apresentada com **dois jeitos de uso**:

1. **Cenário 1 — Herança**: sua classe **herda** de `CL_ABAP_PARALLEL` e você redefine o método `DO`.  
2. **Cenário 2 — Interface**: você cria uma classe “tarefa” que **implementa `IF_ABAP_PARALLEL`** e coloca a lógica no método `IF_ABAP_PARALLEL~DO`.

O próprio Software-Heroes recomenda o **cenário 2** por exigir menos “cola” (pack/unpack) e ficar mais limpo.

---

## 2) Quando usar (e quando NÃO usar)

### Use quando
- você tem **muita massa de dados** (processamento de N itens) e o gargalo é CPU/IO ou chamadas repetidas.
- o processamento de cada item (ou pacote) é **independente** (pouca ou nenhuma necessidade de compartilhar estado).
- você quer **reduzir tempo total** sem bloquear a sessão principal (ex.: execução em ação RAP / processo pesado chamado por UI).
- no contexto de RAP, quando você precisa de **uma LUW separada** para executar algo que exige commit (o artigo do Sachin mostra isso com EML + commit dentro da tarefa).

### Evite quando
- há **dependências fortes** entre itens (ex.: item 2 depende do item 1).
- você precisa de muita sincronização/locking compartilhado (pode piorar).
- o “trabalho por pacote” é muito pequeno (overhead de paralelização > ganho).
- você tem limites de work processes / políticas do ambiente que podem restringir paralelismo.

---

## 3) Conceitos essenciais

### 3.1) A unidade de paralelismo: “tarefas”
No cenário 2, **cada instância** da sua classe que implementa `IF_ABAP_PARALLEL` vira uma tarefa, e a lógica “paralela” fica no:

- `IF_ABAP_PARALLEL~DO`

### 3.2) Coleta de resultados
- No cenário 2, cada tarefa guarda seu resultado internamente e depois o chamador “faz CAST” e chama um `GET_RESULT` (o Software-Heroes mostra esse padrão).

---

## 4) Parâmetros importantes (controle de paralelismo)

O Software-Heroes destaca parâmetros no construtor de `CL_ABAP_PARALLEL`:

- `P_NUM_TASKS`: **número fixo** de processos/tarefas em paralelo.
- `P_PERCENTAGE`: percentual (0..100) do “pool” de processos utilizáveis que será usado.  
  > Se `P_PERCENTAGE` e `P_NUM_TASKS` forem passados, o artigo indica que **o percentual prevalece**.
- `P_DEBUG`: modo de debug.  
  > Quando ligado, as tarefas rodam “uma após a outra” (sem execução paralela via RFC), facilitando depuração.

---

# 5) Passo a passo prático (com exemplos)

Abaixo está um roteiro “copiável” para você adaptar.

## 5.1) Defina o que vai ser paralelizado
Escolha uma unidade independente, por exemplo:
- processar cada parceiro/cliente
- processar cada nota fiscal
- chamar API externa por documento
- criar entidades via EML por item

**Regra de ouro:** cada tarefa deve conseguir rodar com **seu próprio input** e produzir **seu próprio output**.

---

## 5.2) (Recomendado) Cenário 2 — Classe tarefa implementando `IF_ABAP_PARALLEL`

### Passo A — Crie a classe “Task” (worker)
Estrutura típica (igual aos artigos):
- `INTERFACES if_abap_parallel.`
- construtor recebe o input do pacote
- `IF_ABAP_PARALLEL~DO` faz o trabalho
- um `GET_RESULT` expõe o resultado para o chamador

Exemplo *conceitual* (padrão do Software-Heroes / Sachin):

```abap
CLASS zcl_my_parallel_task DEFINITION
  PUBLIC FINAL CREATE PUBLIC.

  PUBLIC SECTION.
    INTERFACES if_abap_parallel.
    METHODS constructor IMPORTING is_input TYPE your_input_type.
    METHODS get_result RETURNING VALUE(rt_result) TYPE your_result_type.

  PRIVATE SECTION.
    DATA ms_input  TYPE your_input_type.
    DATA mt_result TYPE your_result_type.
ENDCLASS.

CLASS zcl_my_parallel_task IMPLEMENTATION.
  METHOD constructor.
    ms_input = is_input.
  ENDMETHOD.

  METHOD if_abap_parallel~do.
    " sua lógica pesada aqui (por pacote)
    " preencha mt_result
  ENDMETHOD.

  METHOD get_result.
    rt_result = mt_result.
  ENDMETHOD.
ENDCLASS.
```

> Observação: no artigo do Sachin, a task também implementa `IF_SERIALIZABLE_OBJECT` e executa EML + commit dentro do `DO` para isolar a transação.

---

### Passo B — Monte a tabela de instâncias (1 instância = 1 pacote)
No Software-Heroes, eles criam uma tabela de instâncias:

- `cl_abap_parallel=>t_in_inst_tab`

E inserem `NEW zcl_bs_demo_para_task( ... )` a cada item. Depois, chamam `run_inst`.

Exemplo conceitual:

```abap
DATA lt_tasks   TYPE cl_abap_parallel=>t_in_inst_tab.
DATA lt_done    TYPE cl_abap_parallel=>t_out_inst_tab.
DATA lt_result  TYPE your_result_table_type.

LOOP AT lt_input INTO DATA(ls_input).
  INSERT NEW zcl_my_parallel_task( ls_input ) INTO TABLE lt_tasks.
ENDLOOP.

NEW cl_abap_parallel( p_num_tasks = 3 )->run_inst(
  EXPORTING p_in_tab  = lt_tasks
  IMPORTING p_out_tab = lt_done ).

LOOP AT lt_done INTO DATA(ls_done).
  DATA(lo_task) = CAST zcl_my_parallel_task( ls_done-inst ).
  " agregue o resultado de cada tarefa
  " INSERT LINES OF lo_task->get_result( ) INTO TABLE lt_result.
ENDLOOP.
```

---

## 5.3) Cenário 1 — Herança de `CL_ABAP_PARALLEL` (quando faz sentido)
O Software-Heroes mostra o cenário 1 como mais “trabalhoso” porque você precisa **empacotar/desempacotar** payload manualmente (ex.: `xstring`), mas é útil quando você quer seguir o modelo antigo de “classe executora” herdando de `CL_ABAP_PARALLEL`.

Padrão:
- `INHERITING FROM cl_abap_parallel`
- `METHOD do REDEFINITION.`

E para empacotar, o exemplo usa `CALL TRANSFORMATION id ... RESULT XML ld_in` para montar `xstring` e manda uma tabela `cl_abap_parallel=>t_in_tab` para o `run`.

---

## 5.4) Dicas práticas (que salvam tempo)

### Ajuste de paralelismo
- comece com `P_NUM_TASKS = 3` ou similar
- meça o tempo total
- aumente gradualmente até o ponto em que os ganhos param (ou começa a degradar)

### Debug
- use `P_DEBUG` quando precisar depurar, porque o artigo descreve que isso faz as tarefas rodarem sequencialmente (facilita breakpoint).

### Cuidado com lock/commit
- se sua tarefa faz update/insert, pense em conflitos (locks).
- em RAP, o padrão do Sachin é valioso quando você precisa de uma LUW independente para commits fora do fluxo principal.

---

## 6) Checklist rápido (antes de aplicar em produção)

- [ ] Meu workload é “massivo” e independente por item/pacote?
- [ ] Eu consigo definir input/output claros por tarefa?
- [ ] Tenho uma estratégia para lock/conflitos?
- [ ] Tenho logs/trace para depurar tarefas?
- [ ] Defini limites (`P_NUM_TASKS` ou `P_PERCENTAGE`) para não saturar o sistema?

---

## 7) Referências
- Software-Heroes — ABAP Cloud: Parallel processing (CL_ABAP_PARALLEL, cenários 1/2, parâmetros e debug)  
- Sachin Artani — Parallel Processing in ABAP Cloud Using CL_ABAP_PARALLEL and RAP EML (paralelo + RAP/EML + commit por tarefa)
