# Controlador de Temperatura Analógico & Atuação LED (NTC, PSpice, Mega2560 & LabVIEW)

Projeto completo de **eletrónica analógica, instrumentação e aquisição de dados**, desde a caracterização experimental de um termístor **NTC** até à atuação física de LEDs por dois métodos distintos (**controlo linear por corrente** e **modulação PWM analógica**), com monitorização em tempo real e *data logging* via **ATmega2560** e **LabVIEW**.

Projeto académico desenvolvido no âmbito da unidade curricular **ELEAPL (Eletrónica Aplicada)** da **Licenciatura em Engenharia Eletrotécnica e de Computadores (LEEC)** no **ISEP** (2.º ano, 2.º semestre, 2025/2026).

---

## Arquitetura do Sistema

```text
  [Termístor NTC 10 kΩ] (17 °C – 37 °C)
            │
            ▼
  [Ponte de Wheatstone] (equilibrada a 17 °C → Vd ∈ [0; -3,17] V)
            │
            ▼
  [Amplificador de Instrumentação] (3× TL082, Ad = -1,58 V/V → V1 ∈ [0; 5] V)
            │
            ▼
  [Limitador de Precisão] (TL082 + 1N4148 → proteção estrita V2 ∈ [0; 5] V)
            │
            ├──────────────────────────────┬──────────────────────────────┐
            ▼                              ▼                              ▼
  [ATmega2560 + LabVIEW]       [Controlo Linear por Corrente]    [Controlo por Tensão (PWM)]
  ADC 10-bit (64 amostras)     Somador não-inversor (V0: 3,25–   Oscilador onda triangular
  UART 9600 bps → Dashboard    9,55 V) + Fonte de corrente       3 andares (~250 Hz, 0–4,9 V)
  + Alarmes + Logging .xlsx    (TL082 + 2N3904 + Zener) → 3 LEDs + Schmitt Trigger → LED PWM
```

---

## Parte 1 — Cadeia de Condicionamento de Sinal & Aquisição Digital

- **Sensor:** Termístor **NTCLE100E3 10 kΩ** caracterizado para a gama operacional **17 °C – 37 °C** ($R_T \approx 14{,}27\text{ k}\Omega$ a $6{,}04\text{ k}\Omega$, ajuste exponencial $R^2 = 0{,}9997$).
- **Ponte de Wheatstone:** Alimentada a 15 V DC e equilibrada a 17 °C ($R_1 = 7{,}01\text{ k}\Omega$, $R_2 = R_3 = 10\text{ k}\Omega$), produzindo $V_d \in [0;\,-3{,}17]\text{ V}$ com comportamento quase linear ($R^2 = 1$).
- **Amplificador de Instrumentação (AI):** Topologia clássica de 3 AmpOps (**TL082**), ganho diferencial $A_d = -1{,}58\text{ V/V}$ ($R_{ganho} = 34{,}48\text{ k}\Omega$), mapeando a excursão térmica em **0 V – 5 V**.
  - **CMRR:** 88,13 dB em simulação PSpice e 49,83 dB experimentalmente em *breadboard*.
- **Limitador de Precisão:** Proteção ativa com **TL082** e díodos **1N4148** (compensação da queda de 0,7 V em malha aberta), fixando estritamente $V_2 \in [0;\,5]\text{ V}$ para salvaguarda do ADC.
- **Validação Completa:** Análise **Monte Carlo** (100 amostras) e verificação de potências dissipadas (máx. 10 mW em resistências de 250 mW).
- **Aquisição em ATmega2560 (`firmware/`):** Código C *bare-metal* com **Timer5 CTC (500 ms)**, média de **64 amostras do ADC de 10 bits** (resolução térmica final $\approx 0{,}02\text{ °C}$) e envio por **USART0 (9600 bps, 8N1)** para PuTTY (`putty_code.c`) ou LabVIEW (`labview_code.c`).
- **Interface LabVIEW (`labview/ntc_view.vi`):** Termómetro, *gauge* de tensão, leitura raw do ADC, gráfico temporal, **Linear Checker (XY)**, sistema de alarmes (`≤ 17 °C`, `GOOD`, `≥ 37 °C`) e *data logging* automático em `.xlsx` a cada 5 segundos.

---

## Parte 2 — Controlo de Luminosidade de LEDs (Corrente vs. PWM Analógico)

A tensão condicionada $V_2 \in [0;\,5]\text{ V}$ comanda a luminosidade de LEDs azuis através de duas arquiteturas analógicas independentes:

### 1. Controlo Linear por Corrente
- **Condicionador de Sinal (Somador Não-Inversor):** Converte $V_2 \in [0;\,5]\text{ V}$ em $V_0 \in [3{,}25;\,9{,}55]\text{ V}$ ($V_0 = 1{,}26\,V_2 + 3{,}25\text{ V}$).
- **Fonte de Corrente Controlada por Tensão:** AmpOp **TL082** com transístor NPN **2N3904**, resistência de emissor $R_8 = 200\text{ }\Omega$ e díodos Zener de referência/proteção (2,7 V e 10 V), regulando linearmente $I_{LED}$ entre **~3 mA** (17 °C, brilho mínimo visível) e **~31,5–34 mA** (37 °C, brilho máximo em 3 LEDs em paralelo, $R^2 = 0{,}9969$ experimental).

### 2. Controlo por Tensão (Modulação PWM Analógica)
- **Oscilador de Onda Triangular (3 Andares com TL082):**
  1. Gerador de onda quadrada ceifado simetricamente com dois Zener de 4,7 V em anti-série ($\pm 5{,}4\text{ V}$);
  2. Integrador analógico dimensionado para **250 Hz** (medido: **249 Hz**, $\pm 2{,}48\text{ V}$), evitando cintilação percetível ao olho humano;
  3. Somador inversor (*level shift*) para deslocar a portadora triangular para a gama unipolar **0 V – 4,9 V**.
- **Comparador PWM com Histerese (*Schmitt Trigger*):** Compara $V_2$ com a portadora triangular ($\Delta V_{th} = 0{,}27\text{ V}$ teórico / $0{,}41\text{ V}$ experimental), modulando o *duty cycle* de **0% a 100%** com retificação de proteção à saída e corrente nominal de ~10 mA no LED.

---

## Estrutura do Repositório

```text
isep-controlador-temperatura-analogico/
├── firmware/
│   ├── labview_code.c                              # Firmware ATmega2560 (formato CSV para LabVIEW)
│   └── putty_code.c                                # Firmware ATmega2560 (saída formatada para PuTTY)
├── labview/
│   ├── ntc_view.vi                                 # Interface gráfica LabVIEW (monitorização + alarmes)
│   └── data_logging_labview.xlsx                   # Exemplo de registo experimental capturado
├── simulations/
│   ├── parte1-condicionamento/                     # Esquemas e perfis PSpice for TI (Parte 1)
│   └── parte2-controlo-leds/                       # Esquemas e perfis PSpice for TI (Parte 2)
├── data/
│   ├── trabalho_NTCLE100E3_10k.xlsx                # Cálculos, regressões e tabelas (Parte 1)
│   └── excel_parte2.xlsx                           # Regressões e comparações (Parte 2)
├── docs/
│   ├── Relatorio_Parte1_Condicionamento_NTC.pdf    # Relatório técnico completo — 1.ª Avaliação
│   └── Relatorio_Parte2_Controlo_LEDs.pdf          # Relatório técnico completo — 2.ª Avaliação
├── .gitignore
└── README.md
```

---

## Ferramentas e Hardware Utilizados

- **Eletrónica Analógica:** Termístor NTCLE100E3 (10 kΩ), 7× AmpOp **TL082**, transístor **2N3904**, díodos **1N4148** e Zener (2,7 V, 4,7 V, 9,7 V), LEDs azuis, *breadboard* e instrumentação de bancada (osciloscópio Tektronix, gerador de sinais, fonte simétrica ±15 V)
- **Simulação:** **PSpice for TI** (DC Sweep, Transient, análise de CMRR, potências e Monte Carlo)
- **Microcontrolador & Software:** **Arduino Mega 2560 (ATmega2560)** programado em C *bare-metal* (`avr-gcc`), **NI LabVIEW**, **PuTTY** e **Microsoft Excel**

---

## Documentação

- [Relatório Parte 1 — Controlador de Temperatura Analógico (Condicionamento NTC + Mega2560 + LabVIEW)](docs/Relatorio_Parte1_Condicionamento_NTC.pdf)
- [Relatório Parte 2 — Controlo de Luminosidade de LEDs por Corrente e PWM Analógico](docs/Relatorio_Parte2_Controlo_LEDs.pdf)

---

## Contexto Académico e Autores

- **Instituição:** Instituto Superior de Engenharia do Porto (ISEP)
- **Curso:** Licenciatura em Engenharia Eletrotécnica e de Computadores (LEEC)
- **Unidade Curricular:** Eletrónica Aplicada (ELEAPL) — 2025/2026
- **Equipa:** Marcel Silva e Daniel Afonso
