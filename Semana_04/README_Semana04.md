# Semana 4 — Do tempo à frequência: FFT, correção de fase e constelações

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Base: Bloessl *et al.* (SRIF'13), secções 2.5 e 2.6
> Flowgraph: [`demo_s4_equalizer.grc`](demo_s4_equalizer.grc)

Nesta semana o sinal deixa de ser uma sequência de amostras e passa a ser um conjunto de **52 símbolos complexos, um por subportadora**. É aqui que se vê a constelação — e é aqui que se percebe porque é que o recetor do artigo se limita a BPSK/QPSK.

---

## 1. `stream_to_vector`: porque 64

`blocks_stream_to_vector(num_items=64)` agrupa 64 amostras consecutivas num único *item* de vector (vlen = 64) e **propaga as tags** para o início do vector. Como o `sync_long` já removeu o CP, cada 64 amostras são exatamente um símbolo OFDM de $3{,}2\ \mu s$.

| Sem `stream_to_vector` | Com `stream_to_vector` |
|---|---|
| a FFT veria um fluxo contínuo e misturaria símbolos | cada vector é independente e alinhado |
| as tags cairiam a meio de um símbolo | a tag `wifi_start` fica no início do vector |

O bloco é, portanto, o **tradutor entre o domínio "amostras" e o domínio "símbolos"**: a partir daqui, um item = uma FFT = um símbolo OFDM.

## 2. A FFT

`fft_vxx(fft_size=64, forward=True, shift=True, window=rectangular)`.

- **64 pontos**: as 64 subportadoras de 802.11a; a saída é $Y[k]$, $k=-32..31$ (com `shift=True`, a DC fica no centro do vector, índice 32).
- **sem janela** (`rectangular`): num sistema OFDM o símbolo já é periódico por construção (o CP garante isso); aplicar uma janela de Hanning destruiria a ortogonalidade e espalharia energia entre subportadoras. A janela só faria sentido se o símbolo fosse truncado a meio.
- mapeamento: a subportadora $k$ está no índice $k+32$ do vector. Assim, os pilotos $k\in\{-21,-7,+7,+21\}$ estão nos índices $\{11,25,39,53\}$ — é por isso que o código do equalizador usa exatamente esses índices.

$$Y[k]=\sum_{n=0}^{63} y[n]\,e^{-j2\pi kn/64}.$$

---

## 3. O `OFDM Equalize Symbols` (bloco `ieee802_11_frame_equalizer`)

Recebe **um vector de 64 complexos por símbolo** e devolve **48 bytes por símbolo** (os bits "moles" já quantizados). Internamente faz cinco coisas:

### 3.1 Estimativa de canal (a partir do LTS)

Os dois primeiros símbolos da janela são as duas repetições do LTS. Como a sequência $L[k]$ é conhecida:

$$\hat{H}[k]=\frac{1}{2}\left(\frac{Y_0[k]}{L[k]}+\frac{Y_1[k]}{L[k]}\right).$$

Aqui $L[k]$ é o LTS no **domínio da frequência** ($\pm1$ nas subportadoras usadas) — não confundir com os 64 taps do LTS no **tempo**, que o `sync_long` usa no *matched filter*. O mesmo par de LTS serve para estimar a SNR: $\mathrm{SNR}=10\log_{10}\frac{\sum|Y_0+Y_1|^2}{2\sum|Y_0-Y_1|^2}$ (o 2 desconta o ruído contado nas duas cópias); é este o valor da tag `snr`. A equalização é então uma inversão simples (*zero forcing*):

$$\hat{X}[k]=\frac{Y[k]}{\hat{H}[k]}.$$

Com LS (o algoritmo por defeito) esta estimativa é refeita a cada trama. As alternativas `LMS`, `COMB` e `STA` trocam robustez por adaptatividade — no seletor `algo` do flowgraph podem comparar-se as quatro.

### 3.2 Compensação do erro de temporização (SFO)

O relógio de amostragem do recetor não é o do transmissor. Um erro relativo $\varepsilon$ (em ppm) produz, na subportadora $k$ do símbolo $m$, uma fase **linear em $k$** que cresce símbolo a símbolo. O código compensa-a multiplicando cada bin por

$$Y'[k]=Y[k]\;e^{\,j\,2\pi\,k\,(\varepsilon_0+\varepsilon_r)\,m\,\frac{80}{64}},$$

com $k=i-32$ (é daí que vem o `(i - 32)` do código), $m$ = `d_current_symbol` e $\varepsilon_0=\Delta f/f_c$ (adimensional) tirado da tag `wifi_start`. O fator $80/64$ é o número de amostras por símbolo (CP incluído) sobre o tamanho da FFT: a deriva acumula-se ao ritmo do símbolo *transmitido*, mas mede-se no símbolo *útil*.

### 3.3 Fase comum pelos pilotos

As quatro subportadoras piloto transportam BPSK conhecida, mas com dois ingredientes: a **polaridade** $p_m$ muda de símbolo para símbolo — `POLARITY[(m-2) % 127]`, a sequência de 127 elementos da norma — e há um **sinal fixo** por piloto. Nos bins 11/25/39/53 (= $-21/-7/+7/+21$) os sinais fixos são $+1,+1,+1,\mathbf{-1}$: a piloto de $+21$ transporta $-1$.

O código estima a fase comum com a combinação exata

$$\hat{\beta}_m=\arg\Big(p_m\,x_{11}+p_m\,x_{25}+p_m\,x_{39}-p_m\,x_{53}\Big),$$

e corrige todo o símbolo com $e^{-j\hat{\beta}_m}$ (todos os 64 bins, antes de equalizar). O cálculo é feito sobre o símbolo **já corrigido de SFO** (§3.2). Para $m<2$ — os dois LTS — usa a variante do preâmbulo, $\arg(x_{11}-x_{25}+x_{39}+x_{53})$, porque a polaridade só começa no SIGNAL.

Sem esta correção, um resíduo de CFO de 1 kHz roda a constelação $\approx1{,}4^\circ$ por símbolo; ao fim de 20 símbolos são $\approx29^\circ$ — suficiente para trocar bits em QAM.

*Detalhe do código:* `POLARITY[(m-2) % 127]` é avaliado para todos os símbolos; com $m=0$ e $m=1$ o índice é $-2$ e $-1$ (em C++ o resto de um negativo é negativo), ou seja, uma leitura fora do array. Não afeta a saída — nesses símbolos $\beta$ usa o ramo do preâmbulo e o laço ainda não é atualizado — mas é um pormenor a corrigir no upstream.

### 3.4 Seguimento da deriva de amostragem (SFO residual)

O que o laço segue **não é** o CFO — esse é corrigido símbolo a símbolo por $\beta$. É a **deriva de amostragem** que sobra. Para cada piloto compara-se a fase com a do símbolo anterior e soma-se:

$$e_r=\arg\Big(\sum_{j}\overline{x_{j}^{(m-1)}}\,x_j^{(m)}\,s_j\Big)\cdot\frac{f_s}{2\pi f_c\cdot 80},$$

onde $s_j$ é o sinal fixo da piloto. O resultado é filtrado por um passa-baixo de 1.ª ordem

$$\varepsilon_r\leftarrow(1-\alpha)\,\varepsilon_r+\alpha\,e_r,\qquad \alpha=0{,}1,$$

e entra na correção linear de §3.2 ($\varepsilon=\varepsilon_0+\varepsilon_r$). É por isso que uma deriva de relógio aparece como **rotação progressiva ao longo das subportadoras**, e não como fase comum: o CFO dá fase igual em todos os bins, a SFO dá uma rampa.

---

## 4. Porque o artigo se limita a BPSK e QPSK

A equalização de **magnitude** assume uma forma *sinc* para a resposta dos filtros do transmissor. O artigo diz explicitamente que essa forma depende do transmissor e que, por isso, a amplitude não é suficientemente fiável para QAM:

> *"the current implementation of our block assumes the magnitude of the carriers to be sinc-shaped... this equalization needs some further improvement, as it currently restricts our receiver to BPSK and QPSK modulations."*

Em BPSK/QPSK a informação está só na **fase**, que os pilotos corrigem bem; em 16/64-QAM a amplitude também decide o símbolo, e um erro de 10–15 % na estimativa de $|H[k]|$ troca bits adjacentes.

### 4.1 O que se mede hoje (build 2025)

A versão atual já descodifica QAM em canal plano. Medições (3 s ≈ 30 tramas, BPSK/QPSK/QAM com o mesmo `threshold` e `sync_length`):

| MCS | ruído 0,10 | ruído 0,30 | ruído 0,50 |
|---|---|---|---|
| BPSK 1/2 | 27 | 27 | 27 |
| QPSK 1/2 | 27 | 27 | 19 |
| 16-QAM 1/2 | 27 | 7 | **0** |
| 64-QAM 2/3 | 26 | **0** | **0** |

**Leitura:** a limitação deixou de ser "QAM não funciona" e passou a ser **sensibilidade**. Cada escalão de modulação exige ~4–6 dB mais SNR; em canal com desvanecimento seletivo (multipercurso), a QAM degrada muito antes da BPSK porque a estimativa de magnitude por subportadora é o elo fraco. É esta a interpretação correta a escrever no relatório: a limitação do artigo é de **estimação de amplitude**, e continua a manifestar-se na robustez, não na compatibilidade.

---

## 5. O que observar no flowgraph

| Sink | Sinal | O que se espera |
|---|---|---|
| `const_pre` | 64 bins pós-FFT, antes do equalizer | nuvem **rodada** (CFO) e escalada; DC e guarda em zero; pilotos em ±1 |
| `const_post` | mensagem `symbols` do equalizer (48 pontos) | BPSK: 2 nuvens em $\pm1$; QPSK: 4; 16-QAM: 16 (com dispersão de amplitude) |
| `spec` | espectro do sinal recebido | forma *sinc* de 20 MHz, portadoras nulas nas bordas |
| `ltf_mf` | correlação com o LTS | dois picos (Semana 3) |

### 5.1 Como ler uma constelação que "não fecha"

| Aspeto | Interpretação | Ação |
|---|---|---|
| Nuvem circular difusa | ruído dominante / CFO mal corrigida | subir SNR; ver `master_v3` com `cfo_hz` |
| Nuvem em anel (fase certa, amplitude errada) | equalização de magnitude deficiente | comparar LS vs LMS; reduzir ordem de modulação |
| 4 nuvens mas rodadas | fase comum não corrigida | confirmar pilotos e polaridade |
| Pontos em linha (só eixo real) | BPSK (esperado) | — |
| Dispersão crescente ao longo da trama | SFO/CFO residual | ver laço de pilotos; medir tag em Hz |

---

## 6. Protocolo experimental

1. **LS vs LMS vs COMB vs STA** (`algo`): mesmo MCS, mesmo ruído; registar a dispersão da constelação e o nº de tramas descodificadas.
2. **BPSK → 64-QAM** (`encoding`): com ruído fixo (0,10), ver a constelação abrir e o número de tramas cair.
3. **CFO** (`master_v4`, `cfo_hz` = 0, 50, 100, 200 kHz): observar a rotação em `const_pre` e a correção em `const_post`.
4. **Sem equalizer**: desativar o bloco `eq` e acrescentar um `qtgui_const_sink_x` de tipo *complex* ligado a `v2s_pre` (o `const_post` é `msg_complex` e só aceita mensagens) — a constelação fica rodada e escalada; é a demonstração de que o equalizador faz falta.

> Nota: o `const_post` é alimentado pela **mensagem** `symbols`, publicada apenas para símbolos de dados (o SIGNAL não é incluído). É por isso que não se vê a constelação do campo SIGNAL — essa é descodificada internamente e observa-se pelos tags (Semana 5).

---

## 7. Análise crítica

- A FFT é o ponto onde o ruído térmico se reparte por 52 bins: cada subportadora tem $1/52$ da potência, o que baixa a SNR por subportadora em $\approx17$ dB face à SNR de banda larga — e é por isso que o OFDM precisa de FEC.
- A correção de fase é **incremental**: erro de estimativa num símbolo propaga-se ao seguinte através do laço. Um $\alpha$ grande segue melhor a CFO mas amplifica ruído; $\alpha=0{,}1$ é o compromisso da implementação.
- A equalização por *zero forcing* amplifica o ruído nas subportadoras com $|H[k]|$ pequeno (desvanecimento profundo). Uma variante MMSE evitaria isso, mas exigiria estimar a variância do ruído — não está implementada.
- A limitação de QAM é de **modelo de canal**, não de hardware: com um canal plano e SNR alta, a QAM passa; com multipercurso, falha primeiro. O gráfico certo para o relatório é PDR vs SNR **por MCS**, não uma conclusão binária.

---

## 8. Checklist e commits sugeridos

- [ ] Justificar 64 amostras/vector, `shift=True` e janela rectangular
- [ ] Descrever as 5 funções do equalizador (canal, SFO, fase, CFO residual, subconjunto)
- [ ] Constelações pré e pós equalizador capturadas
- [ ] Tabela MCS × ruído preenchida
- [ ] Explicar a limitação de QAM citando o artigo e as medições

```
s4: stream_to_vector + FFT 64 + constelacoes pre/pos equalizer
s4: seletor de MCS e de algoritmo do equalizer
s4: medidas de sensibilidade por MCS (BPSK->64QAM)
s4: analise da limitacao de amplitude (artigo vs build atual)
```
