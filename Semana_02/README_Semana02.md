# Semana 2 — OFDM Sync Short: a válvula, os *thresholds* e os modos de falha

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Base: Bloessl *et al.* (SRIF'13), secção 2.2 e limitações no fim da mesma secção
> Flowgraph: [`demo_s2_sync_short.grc`](demo_s2_sync_short.grc)

A Semana 1 produziu um número $c[n]$ que vale $\approx0{,}75$ durante o preâmbulo curto e $\approx0{,}1$ no resto. A Semana 2 usa esse número para **decidir** e, sobretudo, para **abrir e fechar uma janela** no stream: o `OFDM Sync Short` não é um detetor passivo, é uma válvula que decide quais amostras continuam para o resto do recetor.

---

## 1. O que o bloco faz (máquina de estados)

`ieee802_11_sync_short(threshold, min_plateau)` tem três entradas:

| Porta (índice) | Conteúdo | Nota |
|---|---|---|
| `0` · `in` | $s[n]$, o sinal complexo | — |
| `1` · `abs` | $a[n]=s[n]s^*[n-16]$, o **produto complexo** (não o módulo) | o rótulo "abs" do GRC induz em erro; é daqui que sai a CFO |
| `2` · `in_cor` | $c[n]=\lvert a[n]\rvert/p[n]$ | a métrica de decisão |

### 1.1 Estados

```
SEARCH ──(min_plateau+1 amostras com c[n] > threshold)──► COPY
   ▲                                                        │
   └──────────── d_copied == 540·80 amostras ───────────────┘
```

**SEARCH**
Percorre o stream e conta amostras consecutivas com $c[n]>\theta$. Só quando o contador chega a `MIN_PLATEAU` **e** aparece mais uma amostra acima é que dispara — ou seja, o bloco exige na prática **`min_plateau` + 1 amostras consecutivas**. Nesse instante:

1. calcula a CFO grosseira $\hat{\omega}=\arg\{a[n]\}/16$ (rad/amostra);
2. insere a tag `wifi_start` com esse valor, no item de saída correspondente ao início da janela;
3. passa a COPY.

Não consome a amostra que disparou: o stream copiado começa exatamente nela.

**COPY**
Copia até `MAX_SAMPLES = 540 × 80 = 43 200` amostras (540 símbolos OFDM, 2,16 ms), aplicando a correção de CFO:

$$s'[m] = s[m]\,e^{-j\hat{\omega}m}.$$

Durante a cópia continua a vigiar $c[n]$; se aparecer um novo patamar **depois de 480 amostras copiadas** (`MIN_GAP = 480`, 24 µs), reinicia a cópia e insere nova tag — isto é, consegue detetar uma segunda trama "por cima" da primeira.

### 1.2 Porque é uma válvula e não um filtro

O bloco não atenua nem molda: **ou entrega tudo** (até 540 símbolos), **ou nada**. Isso tem duas consequências de projeto:

- o custo de processamento a jusante passa a ser proporcional ao número de tramas detetadas, não ao número de amostras recebidas;
- o "contrato" com os blocos seguintes é frágil: a janela é aberta por tempo/comprimento, não pelo fim real da trama.

---

## 2. O significado dos dois *thresholds*

| Parâmetro | Efeito | Compromisso |
|---|---|---|
| `threshold` ($\theta$) | nível de $c[n]$ que conta como "preâmbulo" | baixo → deteta com SNR pior, mas dispara em ruído/interferência; alto → falha tramas fracas |
| `min_plateau` ($m$) | nº de amostras consecutivas exigidas (na prática $m+1$) | alto → imune a picos isolados, mas reage mais tarde e pode perder o início da janela |

### 2.1 Escolha do threshold em vez de "tentativa e erro"

1. Medir $c[n]$ durante 1 s **sem tramas** (payload 0 e `pad_tail` grande, ou simplesmente observar o intervalo entre tramas). Registar o percentil 99,9 — chamemos-lhe $c_{99{,}9}$.
2. Medir o valor do patamar com tramas: $c_{plat}$.
3. Escolher $\theta$ tal que

$$\theta \in \left[c_{99{,}9}+3\sigma_c,\; 0{,}9\,c_{plat}\right],$$

o que na nossa cadeia dá tipicamente $0{,}45$–$0{,}65$. Com $N_{win}=48$ e $p$ sobre $N_{win}+16$, **$\theta$ tem de ser $<0{,}75$** (Semana 1, §2.2).

### 2.2 Efeito medido de variar o threshold

Condições: cadeia completa (TX+RX simulados, como no `master_v2`), 20 Msps, BPSK 1/2, payload 10 B, 1 trama/100 ms, 4 s (≈40 tramas), ruído 0,01, $N_{win}=48$, `sync_length`=320, `delay`=320. O `demo_s2` termina no `tag_dbg_short`; para contar tramas descodificadas usa-se o `master`.

| $\theta$ | Tramas descodificadas | Interpretação |
|---|---|---|
| 0,40 | 37 | deteta tudo; margem exagerada, mais sensível a falsos alarmes |
| 0,56 | 36 | ótimo prático |
| 0,70 | 36 | ainda robusto |
| 0,75 | 36 | limite teórico $48/64$ |
| 0,78 | 0 | acima do teto: **falha total** |
| 0,80 | 0 | falha total |

Observação importante: entre 0,56 e 0,75 **não há degradação** — o patamar é plano. A falha é tudo-ou-nada, não gradual. Um varrimento fino (0,01 em 0,01) mostra a transição brusca entre 0,75 e 0,78.

### 2.3 Efeito de `min_plateau`

Com bom SNR (0,01) e $\theta=0{,}56$, `min_plateau` $\in\{1,\dots,6\}$ descodifica sempre o mesmo número de tramas: o patamar tem $\approx160$ amostras planas, pelo que exigir 2 ou 7 é indiferente. `min_plateau` só é decisivo quando $c[n]$ é ruidoso (SNR baixa): aí aumenta-o para reduzir falsos alarmes, ao custo de reagir mais tarde.

**Regra prática:** `min_plateau = 2` (3 amostras) é o valor por defeito do bloco e dos flowgraphs oficiais; sobe para 3–4 apenas em capturas ruidosas.

---

## 3. Limitações estruturais (e números)

### 3.1 Janela máxima: 540 símbolos

$$540\ \text{símbolos} \times 80\ \text{amostras} = 43\,200\ \text{amostras} = 2{,}16\ \text{ms}.$$

Ao ritmo mais robusto (BPSK 1/2, $N_{DBPS}=24$ bits/símbolo), os símbolos disponíveis para dados são $540-5=535$ (4 de preâmbulo + 1 de SIGNAL), logo

$$L_{max} \approx \frac{535\cdot 24-16-6}{8} \approx 1602\ \text{bytes}.$$

Tramas maiores (PSDU de 2304 B, por exemplo) são **truncadas** e o FCS falha. É uma limitação do bloco, não do 802.11a.

### 3.2 Tramas encostadas: `MIN_GAP = 480` amostras = 24 µs

O `sync_short` só aceita uma nova trama 480 amostras depois do *trigger* anterior. Comparações úteis (todas a 20 Msps):

| Intervalo | Amostras | Comparação com `MIN_GAP` |
|---|---|---|
| SIFS (802.11a) | 16 µs = 320 | **< 480** |
| PIFS | 25 µs = 500 | > 480 |
| DIFS | 34 µs = 680 | > 480 |

Ou seja: qualquer resposta que chegue a SIFS (ACK, CTS) chega **mais perto do fim da trama anterior do que o `MIN_GAP` permite**, se a trama anterior for curta. No artigo original (janela de cópia fixa, sem re-trigger) isto traduzia-se em perder a maioria dos CTS: *"we miss most of the CTS frames as the receiver is still synced on the corresponding RTS"*. A versão atual (`maint-3.10`) mitiga-o com o re-trigger acima das 480 amostras, mas a limitação estrutural mantém-se — e é reproduzível com hardware real: capturar RTS/CTS de um AP e contar quantos CTS não geram tag.

Cálculo de ordem de grandeza para o caso RTS→CTS a 6 Mbit/s:

- RTS: MPDU de 20 B → $N_{sym}=\lceil(16+160+6)/24\rceil=8$ → $16+4+8\cdot4=52\ \mu s$
- CTS: MPDU de 14 B → $N_{sym}=\lceil(16+112+6)/24\rceil=6$ → $16+4+6\cdot4=44\ \mu s$
- CTS começa $52+16=68\ \mu s$ (1360 amostras) após o início do RTS.

Como $1360 > 480$, a implementação atual **consegue** re-triggerar no CTS; a versão do artigo, com janela fixa, não. Documentar esta diferença é mais rigoroso do que repetir a limitação sem a datar.

### 3.3 Alcance da estimativa de CFO

$$\hat{\omega}=\frac{\arg\{a\}}{16},\quad \arg\in(-\pi,\pi]
\;\Longrightarrow\;
|\Delta f|<\frac{f_s}{2\cdot16}=625\ \text{kHz}.$$

Cristais reais: $\pm20$ ppm a 5 GHz = $\pm100$ kHz por lado → até 200 kHz de desvio, folgado dentro do alcance. A 2,4 GHz é ainda menos crítico.

### 3.4 *Offset* DC e interferência

Um *offset* DC constante $d$ produz $a[n]\approx|d|^2$ com $p[n]\approx|d|^2$ → $c\approx1$ **em silêncio**, ou seja, dispara permanentemente. É a razão pela qual o Pluto tem `bbdc`/`quadrature` e a razão pela qual um recetor real precisa de remover DC antes do detetor. Interferência periódica (outro 802.11, Bluetooth) pode imitar o STS e gerar disparos falsos.

---

## 4. Um caso em que o detetor quebra (e como o demonstrar)

**Experiência 1 — falha por threshold.** Com `threshold = 0.80`, o `c[n]` continua a mostrar o patamar em 0,75, mas o `Tag Debug` fica vazio e o `msg_dbg`/`decode` não produzem nada. É a demonstração mais limpa de que o *threshold* é relativo à normalização (§2.2).

**Experiência 2 — falsos alarmes.** Com `threshold = 0.30` e ruído 0,05, o `tag_dbg_short` mostra mais tags do que tramas enviadas — disparos dentro da mesma trama e em ruído. Na cadeia completa mediram-se 26 descodificações para ≈15 tramas (janela de 1,5 s): o excesso vem de disparos múltiplos e de ruído. Custo: CPU desperdiçada e PDUs duplicados na saída.

**Experiência 3 — ganho.** Baixar `gain` para 0,01 com ruído 0,5 baixa a SNR efetiva: $c[n]$ deixa de atingir o patamar e a deteção cessa, apesar de o sinal estar "lá". Mostra que a normalização elimina a dependência da **escala**, mas não da **SNR**.

**Experiência 4 — tramas encostadas.** Reduzir `pad_tail` para 0 aproxima as tramas simuladas; enquanto o intervalo entre *triggers* exceder 480 amostras, tudo funciona. Para reproduzir a falha do artigo é necessário um gerador que emita uma resposta a SIFS de uma trama curta — não é possível com o *mapper* 802.11a deste flowgraph (a trama mínima já dura 24 µs). Fica como experiência para hardware real.

---

## 5. Mapa do flowgraph

| Bloco | Função | O que observar |
|---|---|---|
| `sync_short` | a válvula (SEARCH/COPY) | nº de tags por segundo |
| `tag_dbg_short` | imprime `wifi_start` | **valor da tag = CFO grosseira em rad/amostra** ($\times f_s/2\pi$ dá Hz) |
| `sink_c`, `sink_p` | métrica e potência | patamar vs ruído |
| `gain_block`, `channel` | ganho antes do ruído | efeito da SNR |
| `threshold`, `min_plateau` | sliders | varrimento em tempo real |

O `Tag Debug` mostra linhas do tipo (formato real do GNU Radio 3.10):

```
Offset: 0  Source: sync_short  Key: wifi_start  Value: 0.00149614
```

Sem CFO injetada, o valor fica na ordem de $10^{-3}$ rad/amostra (≈5 kHz — resíduo do ruído). Com `cfo_hz = 100 kHz` no `master_v3`, sobe para ≈0,0314 rad/amostra; multiplicar por $f_s/(2\pi)$ devolve os 100 kHz.

---

## 6. Análise crítica: o que esperar e porquê

- **Patamar plano → decisão binária.** A métrica não tem "degradação suave": com SNR alta, qualquer $\theta<0{,}75$ dá o mesmo resultado. Só perto do teto ou com SNR baixa é que o valor importa.
- **A válvula domina o custo.** 540 símbolos por deteção = 43 200 amostras a jusante mesmo que a trama tenha 20 símbolos. É o preço de não se conhecer o comprimento antes de descodificar o SIGNAL.
- **A tag é a interface.** Todo o sincronismo fino (Semana 3) depende de a tag estar no sítio certo; um erro de algumas amostras no *trigger* propaga-se ao alinhamento de símbolo.
- **Falsos positivos não são inofensivos.** Cada disparo falso alimenta o `sync_long`, que gasta 320+ amostras e pode emitir lixo para o equalizador; com `threshold` baixo, o número de PDUs duplicados cresce.

---

## 7. Checklist e commits sugeridos

- [ ] Explicar a máquina SEARCH/COPY e o papel dos três portos
- [ ] Justificar `threshold < 0,75` com a normalização usada
- [ ] Tabela do varrimento de `threshold` (dados próprios)
- [ ] Pelo menos um caso de falha demonstrado (threshold alto ou ganho baixo)
- [ ] Contas do `MIN_GAP` face a SIFS e do alcance de CFO

```
s2: sync_short instrumentado (tags + c[n]) no fluxo simulado
s2: varrimento de threshold 0.3-0.9 e analise do teto 0.75
s2: caso de falha com threshold 0.8 (sem tags)
s2: analise do MIN_GAP=480 vs SIFS e do limite de 540 simbolos
```
