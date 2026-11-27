# Semana 1 — Deteção de frames 802.11a/g: autocorrelação, potência e $N_{win}$

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Base: Bloessl *et al.*, *An IEEE 802.11a/g/p OFDM Receiver for GNU Radio* (SRIF'13), secção 2.2
> Flowgraph: [`demo_s1_autocorrelacao.grc`](demo_s1_autocorrelacao.grc)

O objetivo da semana é **perceber como se deteta o início de uma trama antes de existir qualquer sincronismo**: só com uma propriedade algébrica do preâmbulo (a periodicidade de 16 amostras do *Short Training Sequence*) e uma normalização pela potência. No fim devemos conseguir reproduzir a Figura 2 do artigo e explicar, com números, o efeito de $N_{win}$.

---

## 1. Parâmetros do sistema 802.11a (tabela de referência)

| Parâmetro | Valor | Dedução |
|---|---|---|
| Largura de banda do canal | 20 MHz | norma (ocupação útil menor, ver abaixo) |
| Espaçamento entre subportadoras | $\Delta f = 312{,}5$ kHz | $20\,\text{MHz}/64$ |
| Tamanho da FFT / IFFT | $N_{FFT}=64$ | 64 subportadoras |
| Taxa de amostragem | $f_s = 20$ Msps | $N_{FFT}\cdot\Delta f$ |
| Duração de um símbolo FFT | $T_{FFT}=3{,}2\ \mu s$ | $1/\Delta f = 64/f_s$ |
| Prefixo cíclico (GI) | $T_{GI}=0{,}8\ \mu s = 16$ amostras | 20 % de $T_{FFT}$ |
| Duração de símbolo OFDM | $T_{sym}=4{,}0\ \mu s = 80$ amostras | $64+16$ |
| Subportadoras usadas | 52 | 48 dados + 4 pilotos |
| Subportadoras nulas | 12 | 1 DC + 11 de guarda |
| Pilotos | $k=\{-21,-7,+7,+21\}$ | BPSK conhecida, polaridade variável |
| Padrão do STS | 16 amostras, repetido 10× | 160 amostras = $8\ \mu s$ |
| Padrão do LTS | 64 amostras, repetido 2× (+GI2 de 32) | 160 amostras = $8\ \mu s$ |
| Preâmbulo completo | 320 amostras | $16\ \mu s$ |
| Débitos | 6 a 54 Mbit/s | 8 esquemas MCS |

### 1.1 Porque existem subportadoras não usadas e onde estão

- **DC ($k=0$)**: o oscilador local e o ADC produzem *offset* DC; uma subportadora em banda base exata transportaria esse erro, não informação.
- **Guarda: $k\in[-32,-27]$ (6) e $k\in[+27,+31]$ (5)**: a máscara espectral do transmissor exige que a energia caia antes da borda do canal de 20 MHz; as subportadoras exteriores seriam atenuadas pelo filtro analógico e criariam interferência no canal adjacente. Funcionam como banda de transição.
- Consequência útil: só $52/64 = 81{,}25\%$ dos bins transportam energia → $52\times312{,}5\ \text{kHz} = 16{,}25$ MHz ocupados.

### 1.2 Porque o STS é detetável sem conhecer a sequência

O STS é um padrão de 16 amostras repetido 10 vezes. Isso significa

$$s[n+16] \approx s[n] \quad \text{durante } 0 \le n < 144,$$

ou seja, **periodicidade com lag 16** (0,8 µs, 1,25 MHz). Um recetor pode testar esta propriedade sem saber *qual* é o padrão — é esta a diferença essencial face ao *matched filter*, que exige a sequência conhecida.

---

## 2. O detector de Schmidl & Cox (secção 2.2 do artigo)

### 2.1 As três equações

Com $s[n]$ a amostra complexa recebida e $N_{win}$ a janela de integração:

$$a[n]=\sum_{k=0}^{N_{win}-1} s[n+k]\,s^{*}[n+k+16] \tag{1}$$

$$p[n]=\sum_{k=0}^{N_{win}-1}\left|s[n+k]\right|^{2} \tag{2 (como impressa no artigo)}$$

$$c[n]=\frac{\left|a[n]\right|}{p[n]} \tag{3}$$

O produto $s[n+k]\,s^{*}[n+k+16]$ tem fase $\approx 0$ quando as duas amostras são iguais (STS) e fase aleatória caso contrário. A soma sobre $N_{win}$ amostras é, por isso, uma **média móvel que funciona como filtro passa-baixo**: reduz o ruído e transforma a periodicidade num patamar ($|a|$ grande) com duração de $\approx 160$ amostras.

A normalização por $p[n]$ torna $c[n]$ **invariante à escala**: se o ganho do recetor duplicar, $a$ e $p$ duplicam e $c$ não muda. É por isso que o detetor funciona com ganhos diferentes — o ganho só importa na medida em que altera a SNR (ver §5.4).

### 2.2 O detalhe que decide o *threshold*: a janela da potência

A implementação de referência (`gr-ieee802-11`) calcula a potência sobre **$N_{win}+16$** amostras, e não $N_{win}$:

$$p_{impl}[n]=\sum_{k=0}^{N_{win}+15}\left|s[n+k]\right|^{2}.$$

Esta escolha cobre a união das duas janelas de (1) e evita que a normalização "veja" menos energia do que a correlação. O efeito colateral é um **teto teórico**:

$$\left|a[n]\right| \le \sqrt{A\,B} \le \frac{A+B}{2} \le p_{impl}[n]
\;\Longrightarrow\;
c[n] \le \frac{N_{win}}{N_{win}+16}.$$

Para $N_{win}=48$: $c_{max}=48/64=\mathbf{0{,}75}$. **Um threshold de 0,8 nunca dispara** — é exatamente o que se mede (ver §6). Se construirmos a cadeia com $p$ sobre $N_{win}$ (como na equação 2 do artigo e no `wifi_detect.grc` original), $c\to1$ e o valor 0,8 passa a ser razoável. **O threshold não é um número absoluto: é relativo à normalização usada.**

> ⚠️ Cuidado prático: mudar a janela da potência muda também o **atraso de grupo** da média móvel ($(N-1)/2$ amostras). A cadeia fica desalinhada e o recetor deixa de descodificar mesmo que o detetor dispare. Nos nossos flowgraphs mantém-se $N_{win}+16$ para coincidir com a referência.

### 2.3 Nível de ruído esperado em $c[n]$

Para ruído gaussiano complexo i.i.d. ($|a|$ é a soma de $N_{win}$ produtos descorrelacionados), e desprezando o atraso entre as janelas:

$$\mathbb{E}\big[|a|\big]\approx\sqrt{N_{win}}\,\sigma^{2},\qquad
p \approx (N_{win}+16)\,\sigma^{2}
\;\Longrightarrow\;
c_{\text{ruído}}\approx\frac{\sqrt{N_{win}}}{N_{win}+16}.$$

Para $N_{win}=48$: $c_{\text{ruído}}\approx 6{,}93/64 \approx \mathbf{0{,}11}$. O patamar do sinal está em 0,75 e o ruído em 0,11 — daí a margem confortável de um threshold entre 0,4 e 0,6. As flutuações de $c$ em ruído crescem como $1/\sqrt{N_{win}}$, o que explica a necessidade de `min_plateau`.

---

## 3. Mapa do flowgraph

O `demo_s1_autocorrelacao.grc` é **autónomo**: inclui um transmissor 802.11a completo (MAC → *mapper* → alocador de subportadoras → IFFT → CP), um canal AWGN e a cadeia de deteção. Não precisa de hardware nem de ficheiros externos.

| Bloco (nome no GRC) | Tipo | Função |
|---|---|---|
| `msg_strobe` → `mac_tx` → `mapper` | `blocks_message_strobe`, `ieee802_11_mac`, `ieee802_11_mapper` | gera MPDUs e codifica o PHY (BPSK 1/2 por defeito) |
| `pad` → `gain_block` → `channel` | `foo_packet_pad2`, `blocks_multiply_const_xx`, `channels_channel_model` | silêncio entre tramas, ganho RX, AWGN |
| `d16` → `conj` → `mult` | `blocks_delay`, `blocks_conjugate_cc`, `blocks_multiply_xx` | $s[n]\,s^{*}[n-16]$ — **o produto de autocorrelação** |
| `ma_c` | `blocks_moving_average_xx` (complex) | **eq. (1)**: média móvel de $N_{win}$ |
| `mag_c` | `blocks_complex_to_mag` | $|a[n]|$ |
| `mag2` → `ma_p` | `blocks_complex_to_mag_squared`, `blocks_moving_average_xx` (float) | **eq. (2)**: potência em $N_{win}+16$ |
| `div` | `blocks_divide_xx` (float) | **eq. (3)**: $c[n]=|a|/p$ |
| `sink_c`, `sink_p` | `qtgui_time_sink_x` | Figura 2: $c[n]$ e $p[n]$ |

O artigo reparte o cálculo em oito blocos padrão precisamente para poder usar VOLK em cada operação — não há "magia": (1)–(3) são somas, produtos e uma divisão.

**Nota de execução — o buffer do `pad`.** O `foo_packet_pad2` é um bloco *tagged stream*: o `general_work` só chama o `work()` quando o buffer de saída tem espaço para `ninput + pad_front + pad_tail` itens, e o buffer por omissão (~1365 itens) não chega. O sintoma é traiçoeiro — o GRC arranca, mas o thread aborta com `Buffer too small for min_noutput_items` e o fluxo fica **parado, sem erro visível na interface**. Todos os `.grc` deste repositório trazem `minoutbuf = 65536` no bloco `pad` (512 KB a 20 Msps), que resolve o problema.

---

## 4. Respostas ao guião

**Onde estão as tags mencionadas na secção 2.1?**
O artigo usa *stream tags* para anotar o início da trama (`wifi_start`), a duração/codificação da trama e o índice do símbolo. Na cadeia desta semana ainda não há tags: o `sync_short` (Semana 2) é o primeiro bloco que as introduz. Neste flowgraph, o sinal existe mas ainda não é usado para "abrir a válvula".

**Quais são os blocos que calculam a autocorrelação?**
`d16` (atraso de 16), `conj`, `mult` (produto complexo) e `ma_c` (soma deslizante). Equivalentemente: $a[n]=\sum_{k} s[n+k]s^*[n+k+16]$.

**Quais calculam a potência?**
`mag2` ($|s|^2$) e `ma_p` (média móvel em $N_{win}+16$). Nota: se usarmos $N_{win}$ em vez de $N_{win}+16$, obtemos a equação (2) tal como está impressa no artigo.

**Como obter um gráfico como a Figura 2?**
1. Correr o flowgraph com `noise = 0.02` e `N_win = 48`.
2. No `sink_c`, usar trigger `Auto`, nível 0,5, janela de 1024 amostras (51 µs).
3. Observar: linha de base $\approx0{,}1$ (ruído), patamar largo $\approx0{,}75$ durante 160 amostras (8 µs), descida abrupta no fim do STS.
4. Sobrepor o `sink_p`: $p[n]$ sobe ~24 amostras **antes** do patamar de $c$ (a média da potência começa a subir mal entra energia) e desce depois — é a assinatura temporal de uma média causal.

**Efeito de variar $N_{win}$**
Ver §5 e a tabela de §6. Resumo: aumentar $N_{win}$ reduz a variância do ruído (patamar mais estável) mas **alarga as transições** e **atrasa a deteção** em $\approx N_{win}/2$ amostras; acima de $N_{win}+16>160$ a janela já inclui amostras do LTS e o patamar deixa de ser plano.

---

## 5. Protocolo experimental

### 5.1 Experiência A — forma de $c[n]$
- `N_win = 48`, `noise = 0.02`, `gain = 1.0`.
- Registar (captura de ecrã + valores lidos do cursor): $c_{\text{ruído}}$, $c_{\text{patamar}}$, largura do patamar em amostras, instante de subida relativo ao início da trama.

### 5.2 Experiência B — varrimento de $N_{win}$
Para $N_{win}\in\{16,32,48,64,80,112,128\}$ (alterar o slider e **reiniciar**; os taps são calculados na construção):

| $N_{win}$ | $c_{max}$ teórico $=N/(N+16)$ | $c_{max}$ medido (simulação) | Largura útil do patamar | Observação |
|---|---|---|---|---|
| 16 | 0,50 | **0,497** | estreita, ruidosa | no limite do threshold 0,56 |
| 32 | 0,667 | **0,668** | boa | |
| 48 | 0,750 | **0,755** | boa | valor do artigo |
| 64 | 0,800 | — | — | $N+16=80$: já toca o 2.º símbolo |
| 80 | 0,833 | — | — | janela > STS útil (144) |
| 128 | 0,889 | — | — | deixa de ver "STS periódico" |

*(Os valores medidos foram obtidos com o modelo de simulação descrito em §6.1; preencher as restantes células com as vossas medições.)*

### 5.3 Experiência C — sensibilidade ao ruído
Varrer `noise` de 0,01 a 0,30 e registar quando $c_{\text{patamar}}$ desce abaixo de 0,5 e quando aparecem falsos patamares no ruído.

### 5.4 Experiência D — ganho
Varrer `gain` de 1,0 a 0,002 **com ruído fixo**. Como $c$ é normalizado, o patamar mantém-se — mas a SNR cai e as flutuações crescem. Isto mostra que **o detetor é invariante ao ganho, mas não à SNR**: na prática, baixar o ganho do LNA piora a figura de ruído e o patamar degrada-se.

---

## 6. Resultados esperados e análise crítica

### 6.1 Medições de $c[n]$ (flowgraph desta semana)

Simulação incluída (TX 802.11a + AWGN), GNU Radio 3.10.12 + `gr-ieee802-11`, 20 Msps, BPSK 1/2, payload 10 B, 1 trama/100 ms, ruído 0,05. Valores obtidos da mesma cadeia em execução headless (0,8 s de sinal):

| $N_{win}$ | $c_{max}$ teórico | $c_{max}$ medido | mediana (piso de ruído/silêncio) | p99 |
|---|---|---|---|---|
| 16 | 0,500 | **0,497** | 0,050 | 0,260 |
| 32 | 0,667 | **0,668** | 0,069 | 0,503 |
| 48 | 0,750 | **0,755** | 0,082 | 0,749 |

A concordância entre teoria e medição é a validação mais direta do modelo de normalização de §2.2: o máximo **não** é 1, é $N_{win}/(N_{win}+16)$.

**Consequência para o *threshold*:** como o piso de ruído está em 0,05–0,10 e o patamar em 0,75, um limiar entre 0,4 e 0,6 separa os dois regimes com folga. O varrimento completo do *threshold* (com a cadeia de descodificação) está na [Semana 2](../Semana_02/README_Semana02.md#22-efeito-medido-de-variar-o-threshold): 0,40–0,75 descodificam, 0,78–0,80 falham.

**Conclusão:** o detetor é um **limiar sobre uma razão normalizada**, não sobre uma amplitude. O valor 0,8 do guião pressupõe $p$ sobre $N_{win}$; com a normalização da implementação ($N_{win}+16$) o máximo é 0,75 — quem não souber isto conclui erradamente que "o detetor não funciona".

### 6.2 O que se vê nos gráficos (e porquê)

- **$c[n]$**: patamar rectangular de ~160 amostras. A subida não é instantânea: começa quando a janela de 48 amostras fica completamente dentro do STS e satura passadas $\approx N_{win}$ amostras. A descida é simétrica.
- **$p[n]$**: "degrau" com rampa de 64 amostras ($N_{win}+16$), atrasado $\approx (N_{win}+15)/2$ amostras face ao sinal. Durante o silêncio entre tramas, $p\to$ ruído e $c$ pode ficar **indefinido** se $p=0$ (divisão por zero). Ver §7.
- **Efeito de $N_{win}$**: o *valor* do patamar cresce com $N_{win}$ (teto $N/(N+16)$) e a variância do ruído cai, mas as transições alargam-se e o cruzamento do limiar atrasa-se em $\approx N_{win}/2$ amostras. O compromisso é entre **estabilidade da decisão** (patamar limpo) e **resolução temporal** (saber exatamente onde começa a trama). Para o `sync_short`, o que importa é o instante do cruzamento — e o `sync_long` herda esse erro de algumas amostras (Semana 3, §5.2).

---

## 7. Limitações e *problem solving*

| Sintoma | Causa | Diagnóstico | Mitigação |
|---|---|---|---|
| $c[n]$ = `nan`/`inf` no silêncio | $p=0$ → divisão 0/0 | olhar para `sink_p` no intervalo entre tramas | garantir ruído mínimo, usar `blocks.divide_xx` com guarda (ex.: `max(p, eps)`) ou `blocks.null_sink`+`valve` |
| Patamar em 0,75 mas threshold 0,8 não dispara | teto $N/(N+16)$ | calcular $48/64$ | baixar o threshold para 0,5–0,6 **ou** normalizar por $N_{win}$ |
| Sem deteção com sinal forte | `gain` demasiado baixo (sinal antes do ruído) | verificar `sink_p` | subir `gain` |
| Falsos patamares em ruído | threshold baixo + `min_plateau` = 1 | contar tags/s | subir threshold ou `min_plateau` |
| Patamar "inclinado" | CFO grande | ver §Semana 3 | o módulo $|a|$ é imune à CFO; a fase não (usa-se depois) |

**Caso de falha didático (passa à Semana 2):** este demo não tem detetor — o `threshold` só entra em jogo no `demo_s2_sync_short`. Lá, com `threshold = 0.80`, o $c[n]$ continua a mostrar o patamar em 0,75 e o `Tag Debug` fica **vazio**. É o exemplo mais claro de que o parâmetro tem de ser interpretado em conjunto com a normalização.

---

## 8. Checklist e commits sugeridos

- [ ] Parâmetros 802.11a todos derivados (não memorizados) — §1
- [ ] Flowgraph corre e mostra $c[n]$ e $p[n]$ — `demo_s1_autocorrelacao.grc`
- [ ] Tabela de $N_{win}$ preenchida com medições próprias
- [ ] Explicação do teto $N/(N+16)$ com a medição 0,75/0,80
- [ ] Identificação explícita dos blocos de (1) e (2)

Commits sugeridos (histórico incremental, critério "processo/progresso"):

```
s1: cadeia Schmidl & Cox com probes de c[n] e p[n]
s1: Tx/Rx 802.11a simulado autonomo (sem Pluto)
s1: medidas de c_max vs Nwin e analise do teto N/(N+16)
s1: nota sobre divisao por zero no silencio
```
