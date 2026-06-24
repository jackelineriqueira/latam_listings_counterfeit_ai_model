# Counterfeit Detection Pipeline — Workflow

## Visão Geral

Pipeline diário automatizado de detecção de produtos falsificados no marketplace Shopee LATAM. Combina dados de QC com análise de imagens via modelo de visão (GPT-4o-mini) para identificar itens que imitam marcas conhecidas.

---

## Configuração de Produção

| Parâmetro | Valor |
|---|---|
| Modelo | `gpt-4o-mini` |
| Prompt | v6 |
| Imagens por item | 2 |
| Image detail | `low` |
| Bots paralelos | 5 |
| Volume diário | ~5.000 itens |

---

## Fluxo de Dados

```
[Presto] latam_bi.counterfeit_lm_daily
         ↓ itens da fila de counterfeit (MX + AR)
[Google Sheets] whitelist de marcas
         ↓ lista de marcas aceitas
[OpenAI Compass] GPT-4o-mini (visão)
         ↓ brand_detected por item
[Presto] latam_bi.counterfeit_ia_results_daily
         ↓ resultados persistidos
[Google Sheets] resultados_ia
         ↓ link compartilhado
[SeatTalk] alerta com métricas + link
```

---

## Tabelas e Fontes

### Entrada

| Tabela | Tipo | Descrição |
|---|---|---|
| `latam_bi.counterfeit_lm_daily` | Presto (USEast) | Backlog diário da fila de counterfeit. Particionada por `grass_date`. Cobre MX e AR. |

**Colunas lidas:**

| Coluna | Descrição |
|---|---|
| `item_id` | ID do item |
| `shop_id` | ID da loja |
| `grass_region` | Região (MX / AR) |
| `queue_id` | ID da fila de QC |
| `queue_name` | Nome da fila |
| `item_link` | URL do item |
| `image1` | URL da imagem principal |
| `image2` | URL da imagem secundária |
| `name` | Nome do produto |
| `shop_name` | Nome da loja |
| `item_creation_date` | Data de criação do item |

**Filtro de data:** `MAX(grass_date)` — garante sempre o dado mais recente disponível, independente do `SCHEDULED_DATE` do agendador.

---

### Saída

| Destino | Localização | Descrição |
|---|---|---|
| Presto | `latam_bi.counterfeit_ia_results_daily` | Tabela principal de resultados, particionada por `grass_date` |
| Google Sheets | ID `1snFH99ogqBJonkniUdjT43oEOmGd8Ddy4A7eEMH-4J0` aba `resultados_ia` | Todos os itens processados (detectados e não detectados) |
| CSV | `ia_counterfeit_mvp_YYYYMMDD_HHMMSS.csv` | Backup local no servidor do Notebook 2 |

**Colunas gravadas em `counterfeit_ia_results_daily`:**

| Coluna | Tipo | Descrição |
|---|---|---|
| `grass_region` | VARCHAR | Região (MX / AR) |
| `item_id` | BIGINT | ID do item |
| `shop_id` | BIGINT | ID da loja |
| `queue_id` | BIGINT | ID da fila de QC |
| `name` | VARCHAR | Nome do produto |
| `shop_name` | VARCHAR | Nome da loja |
| `item_link` | VARCHAR | URL do item |
| `item_creation_date` | DATE | Data de criação |
| `brand_detected` | VARCHAR | Marca detectada ou `-` se nenhuma |
| `model` | VARCHAR | Modelo de IA usado (ex: `gpt-4o-mini`) |
| `prompt_version` | VARCHAR | Versão do prompt (ex: `v6`) |
| `processed_at` | TIMESTAMP | Timestamp do processamento |
| `grass_date` | DATE | Partição (= `PROCESS_DATE`) |

**Idempotência:** antes de cada INSERT, executa `DELETE WHERE grass_date = PROCESS_DATE` para permitir reprocessamento sem duplicatas.

---

## Whitelist de Marcas

- **Fonte:** Google Sheets — ID `1ybLpCUTHyXYouss981KyEayIspkHMuU0Na2Nu9brMG0`, aba `brands`
- **Coluna lida:** `Brand`
- **Deduplicação:** `drop_duplicates()` aplicado antes de montar a string para o prompt
- **Auth:** service account `latam-bi@listings-counterfeit.iam.gserviceaccount.com`

---

## Processamento por Item

Para cada item:

1. Download das imagens (`image1`, `image2`) via HTTP
2. Encode em base64 com detecção automática de media type (JPEG / PNG / WebP)
3. Chamada à API GPT-4o-mini com:
   - System prompt: especialista em detecção de falsificados
   - User prompt v6: analisa imagens + nome do produto contra a whitelist
   - `detail: low` para redução de custo (~85 tokens/imagem vs ~500 em auto)
4. Resposta: nome exato da marca ou `-`
5. Trace registrado no Langfuse para monitoramento

**Paralelismo:** 5 `ThreadPoolExecutor` workers, cada um processa ~1/5 do volume diário simultaneamente.

---

## Monitoramento

| Ferramenta | Uso |
|---|---|
| Langfuse | Trace de cada chamada à API (tokens, latência, input/output) |
| SeatTalk | Alerta diário com volume processado, % detectado e link do Sheets |

**Session ID do Langfuse:** `YYYYMMDD_HHMMSS_{modelo}_{prompt_version}` — permite identificar cada run no dashboard.

---

## Agendamento

- **Plataforma:** Shopee Notebook 2 (K8S)
- **Frequência:** diária
- **Timezone:** `America/Sao_Paulo`
- **Variável injetada:** `${SCHEDULED_DATE}` (formato `YYYYMMDD`) — usada apenas para referência, não para queries Presto

---

## Filas de QC por Região

| Região | Queue ID | Queue Name |
|---|---|---|
| MX | 10675 | Counterfeit |
| AR | 11334 | Counterfeit or Copyright |
