# TCC — Diagnóstico de anomalias em ventilador por análise de vibração

Trabalho de Conclusão de Curso de Engenharia da Computação.

**Objetivo:** detectar e identificar falhas em um ventilador a partir da vibração medida por um acelerômetro. O sistema deve responder duas perguntas:

1. **O ventilador está com falha?** (detecção)
2. **Se está, qual é a falha?** (diagnóstico)

| | |
| --- | --- |
| **Grupo** | *a preencher* |
| **Orientador** | *a preencher* |

---

## Montagem e coleta

| Item | Valor |
| --- | --- |
| Ventilador | Mondial VSP30B, oscilação ligada |
| Sensor | MPU6050 (acelerômetro de 3 eixos: x, y, z) |
| Microcontrolador | ESP32 |
| Taxa de amostragem | 1 kHz |
| Velocidades | 1, 2 e 3 |

**Condições gravadas:**

* **Dados novos** (`DADOS/NOVOS_DADOS`): normal, fita 1g, papel e porca frouxa, nas três velocidades. São cerca de 10 minutos por gravação, começando com o motor frio.
* **Dados antigos** (`DADOS/BASES_ROTULADAS`): normal e fita 1g, nas três velocidades, gravados em outra sessão. Hoje servem como **teste em outra sessão** (Notebook 05, Passo 14).

---

## Os dados não estão no repositório

Os arquivos de vibração (`.csv`) são grandes (cerca de 400 MB no total) e são **ignorados pelo git** (`.gitignore` → `*.csv`). Para rodar os notebooks, coloque a pasta `DADOS` na raiz do projeto com esta organização:

```
DADOS/
├── NOVOS_DADOS/
│   ├── NORMAL/      ventilador_normal_vel_1.csv, _vel_2.csv, _vel_3.csv
│   └── ANOMALIA/    fita_1g_vel_N.csv, papel_vel_N.csv, porca_frouxa_vel_N.csv
└── BASES_ROTULADAS/ ventilador_normal_vel_N.csv, fita_1g_vel_N.csv
```

**Formato de cada arquivo:** um CSV com as colunas `x`, `y`, `z` (aceleração em m/s², uma linha por amostra a 1 kHz).

**O nome do arquivo é o rótulo.** O padrão é `<condicao>_vel_<N>.csv`, por exemplo `porca_frouxa_vel_2.csv`. Os notebooks leem a condição e a velocidade direto do nome. Uma condição nova entra sozinha em todas as análises, basta seguir o padrão.

---

## Estrutura do repositório

```
TCC/
├── README.md
├── requirements.txt
├── DADOS/                     ← não versionado (ver acima)
├── NOTEBOOKS_REPLICAVEIS/     ← trabalho principal
│   ├── DADOS_NOVOS/           ← versão atual: 01 a 05 com os dados novos
│   └── DADOS_ANTIGOS/         ← mesma sequência aplicada aos dados antigos (inclui o 03b)
└── LEGADO/                    ← exploração inicial, anterior aos notebooks replicáveis
    ├── EDA/
    └── ML/
```

* **`NOTEBOOKS_REPLICAVEIS`** é o que vale para o TCC. Cada notebook pode ser reaplicado a dados novos trocando só a pasta no Passo 1.
* **`LEGADO`** guarda a primeira exploração. Fica como histórico do trabalho e **não é mantido**.

---

## Notebooks replicáveis (rodar nesta ordem)

| Notebook | O que faz | Pergunta que responde |
| --- | --- | --- |
| **01 — Entender os dados** | Confere as bases, limpa falhas de leitura do sensor, olha o sinal no tempo, a FFT e o espectrograma | Os dados estão bons? O que muda com a velocidade e com a falha? |
| **02 — Análise comparativa** | Compara cada falha com o normal e as falhas entre si, feature por feature, com um controle (2ª metade × 1ª metade da própria gravação) | Quais features separam cada falha? Dá para distinguir uma falha da outra? |
| **03 — Análise estatística** | Normalidade, efeito do log, janelas atípicas, autocorrelação, ciclo da oscilação, aquecimento | As features são bem comportadas? As janelas são independentes? |
| **03b — Comparação de janelas** (só dados antigos) | Compara janelas de 5 s e de 12 s | Qual o tamanho de janela? |
| **04 — K-Means** | Clusterização sem rótulos, com scaler global e local | Os dados se separam sozinhos? |
| **05 — Classificação** | Modelos supervisionados com validação por blocos, teste trancado e teste em outra sessão | O modelo acerta janelas que nunca viu? E outro dia de gravação? |
| **06 — Detector só com o normal** | *Em planejamento* | Dá para detectar uma falha sem nunca tê-la visto? |

### Como rodar

1. Crie o ambiente e instale as dependências:
   ```
   python -m venv venv
   venv\Scripts\activate
   pip install -r requirements.txt
   ```
2. Abra o notebook e, no **Passo 1 (Configuração)**, ajuste `PASTA_BASES` para a pasta dos dados no seu computador. No Notebook 05, ajuste também `PASTA_OUTRA_SESSAO`.
3. Rode as células em ordem. Cada passo tem um texto explicando o que é feito **antes** do código e uma conclusão depois do resultado.

**Convenção:** cada função fica numa célula e a sua execução fica na célula seguinte.

---

## Decisões de método (valem para todos os notebooks)

| Decisão | Valor | Onde foi decidida |
| --- | --- | --- |
| Corte no começo e no fim de cada gravação | 5 s de cada lado (falhas de leitura ao ligar o sensor) | Notebook 01 |
| Janela | 5 s, sem sobreposição | Notebook 03b (12 s fica como candidata) |
| Features por janela | 109: 9 de tempo (desvio padrão, curtose e pico por eixo), 90 de FFT (máximo em faixas de 10 Hz, de 5 a 305 Hz, por eixo), 9 amplitudes harmônicas (1X, 2X, 3X por eixo) e a rotação estimada (`f1x_hz`) | Notebook 02 |
| Transformação | Log em todas as features menos a curtose | Notebooks 03 e 04 |
| Padronização | `StandardScaler` dentro de um `Pipeline`, ajustado **só no treino** | Notebook 05 |
| Divisão treino/teste | Blocos de 60 s, guarda de 10 s, teste trancado com **um bloco no começo, um no meio e um no fim** de cada gravação | Notebook 05 |
| Métrica principal | Acurácia balanceada, sempre com o recall de cada classe e o falso alarme ao lado | Notebook 05 |
| Regra de escolha | Maior acurácia balanceada; empate (< 0,5 ponto) → maior pior recall; empate → modelo mais simples | Notebook 05, Passo 7 |

---

## Cuidados conhecidos

* **Aquecimento:** nas velocidades 1 e 2, a rotação sobe ao longo da gravação, porque o motor começa frio. O começo e o fim de cada gravação são diferentes, e um modelo treinado só com um estado térmico erra no outro.
* **Ciclo da oscilação:** dura cerca de 225 voltas da hélice, ou seja, cerca de 18 s na Vel 1, 12 s na Vel 2 e 10 s na Vel 3.
* **Uma gravação por condição em cada sessão:** os testes dentro da sessão medem generalização entre trechos da mesma gravação. Generalizar para outro dia exige outra sessão de gravação.
* **O teste trancado e o teste em outra sessão do Notebook 05 (dados novos) já foram abertos.** Mudanças de método daqui em diante precisam ser avaliadas em dados que ainda não foram vistos.
