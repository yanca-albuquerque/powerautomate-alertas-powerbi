# Fluxo no Power Automate: Alerta de Admissão no BI para Outlook

Guia de referência técnica contendo a configuração detalhada de cada bloco de ação, parâmetros de entrada, tipos de dados e expressões utilizadas no fluxo, pensando em criar avisos para os setores como TI, podendo ser tanto para Admissão, Demissão, Outros.

---

## 🛠️ Ferramentas Utilizadas

- **Power Automate:** Agendamento (`Recurrence`) e automação do processo.
- **Power BI:** Modelo semântico (`Dataset`) e consultas analíticas em DAX.
- **Office 365 Outlook:** Disparo dos alertas formatados em HTML.
  
---

## 1. Gatilho: Recorrência (`Recurrence`)

Dispara a execução do fluxo em horários programados para sincronizar com as cargas do banco/Power BI.

* **Conector:** `Agendamento` (*Schedule*)
* **Tipo de Gatilho:** `Recorrência` (*Recurrence*)
* **Parâmetros de Entrada:**
* **Intervalo:** `1`
* **Frequência:** `Dia`
* **Nestes horários (Horas):** `8`, `10`, `14`, `16`
* **Nestes minutos:** `15` *(Ex: executa às 08:00, 10:15, 14:15, 16:15)*
* **Fuso Horário:** `(UTC-03:00) Brasília`

---

## 2. Ação: Consulta ao Modelo Semântico (`ConsultaBI`)

Executa a query DAX diretamente no dataset do Power BI Service via REST API interna do conector.

* **Conector:** `Power BI`
* **Nome da Ação:** `ConsultaBI` *(ou `Executar uma consulta em um conjunto de dados do Power BI`)*
* **Parâmetros de Entrada:**
* **Workspace:** `[Selecionar Workspace de Produção]`
* **Conjunto de Dados:** `BI_RH`
* **Texto da Consulta:**
```dax
EVALUATE
SELECTCOLUMNS(
    FILTER(
        ALL('BI_RH'),
        INT('BI_RH'[DATAADMISSAO]) = INT(TODAY()) &&
        'BI_RH'[SECAO] IN {
            "ADMINISTRAÇÃO",
            "RH",
            "TI",
        }
    ),
    "Filial", 'BI_RH'[Filial],
    "Matrícula", 'BI_RH'[MATRICULA],
    "Nome", 'BI_RH'[NOME],
    "Secao", 'BI_RH'[SECAO],
    "DataAdmissao", FORMAT('BI_RH'[DATAADMISSAO], "dd/mm/yyyy"),
    "DataOrdenacao", 'BI_RH'[DATAADMISSAO]
)
ORDER BY [DataOrdenacao] ASC

```

---

## 3. Controle: Condição (`Condition`)

Avalia se a consulta retornou pelo menos uma linha de admissão/demissão para evitar o disparo de alertas vazios.

* **Conector:** `Controle` (*Control*)
* **Nome da Ação:** `Condição` (*Condition*)
* **Estrutura de Validação:**
* **Valor à Esquerda (Expressão `fx`):**
```text
length(body('ConsultaBI')?['results']?[0]?['tables']?[0]?['rows'])

```
* **Operador:** `é maior que` (*is greater than*)
* **Valor à Direita:** `0`

---

## 4. Ramo "Se Sim" (*True*)

Executado exclusivamente se o total de linhas retornado for maior que zero.

### 4.1. Operação de Dados: Tabela HTML (`Criar_Tabela_HTML`)

Mapeia a matriz de objetos JSON retornada pela consulta e converte em uma tabela estruturada em HTML.

* **Conector:** `Operação de Dados` (*Data Operation*)
* **Nome da Ação:** `Criar_Tabela_HTML` (*Create HTML table*)
* **Parâmetros de Entrada:**
* **De (`From`):**
```text
body('ConsultaBI')?['results']?[0]?['tables']?[0]?['rows']

```
* **Colunas (`Columns`):** `Personalizado` (*Custom*)
* **Mapeamento de Cabeçalhos e Expressões (`fx`):**



| Cabeçalho | Expressão (`fx`) | Tipo de Dado Retornado |
| --- | --- | --- |
| **Filial** | `item()?['[Filial]']` | Inteiro / String |
| **Matrícula** | `item()?['[Matrícula]']` | String |
| **Nome** | `item()?['[Nome]']` | String |
| **Seção** | `item()?['[Secao]']` | String |
| **Data Admissão** | `item()?['[DataAdmissao]']` | String (`dd/MM/yyyy`) |

> **Nota Técnica:** Os colchetes internos (ex: `['[Nome]']`) são mandatórios porque o conector do Power BI prefixa as colunas calculadas com colchetes no payload de saída JSON.

---

### 4.2. Notificação: Enviar E-mail (`Enviar_um_email_(V2)`)

Envia o alerta estilizado contendo o card de aviso para a equipe responsável e injeta a tabela montada.

* **Conector:** `Office 365 Outlook`
* **Nome da Ação:** `Enviar um email (V2)` (*Send an email (V2)*)
* **Parâmetros de Entrada:**
* **Para (`To`):** `[E-mail da Equipe de TI / Responsável]`
* **Assunto (`Subject`):** `Alerta: Admissão de Novo(a) Colaborador(a) - Tratativa TI`
* **Corpo (`Body`):** Inserido via modo de exibição de código HTML (`</>`):

```html
<style>
  body {
    font-family: Arial, "Segoe UI", sans-serif;
    color: #334155;
  }
  .alerta-box {
    background-color: #eff6ff;
    border-left: 4px solid #2563eb;
    padding: 12px 16px;
    margin-bottom: 20px;
    border-radius: 4px;
    color: #1e40af;
    font-size: 14px;
  }
  .alerta-box strong {
    display: block;
    font-size: 15px;
    margin-bottom: 4px;
  }
  table {
    border-collapse: collapse;
    width: 100%;
    font-family: Arial, "Segoe UI", sans-serif;
    font-size: 13px;
    margin-top: 12px;
  }
  th {
    background-color: #1e293b;
    color: #ffffff;
    text-align: left;
    padding: 10px 12px;
    border: 1px solid #cbd5e1;
    font-weight: 600;
  }
  td {
    padding: 8px 12px;
    border: 1px solid #e2e8f0;
    text-align: left;
  }
  tr:nth-child(even) {
    background-color: #f8fafc;
  }
</style>

<div class="alerta-box">
  <strong>ℹ️ Ação Necessária: Admissão Identificada</strong>
  Por favor, providenciar a criação de acessos a sistemas, criação de e-mail corporativo, preparação e entrega de equipamentos de TI.
</div>

<p>Relação de novos colaboradores admitidos na carga do Power BI:</p>
@{body('Criar_Tabela_HTML')}

```

---

## 5. Ramo "Se Não" (*False*)

* **Ação:** Vazio (*Terminate silently*).
* **Comportamento:** O fluxo é encerrado com status de sucesso sem disparar notificações, garantindo que a caixa de entrada receba mensagens apenas quando houver ação operacional requerida.
