# Semana 5 — Campo SIGNAL, *stream tags* e *soft-bits*

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Base: Bloessl *et al.* (SRIF'13), secção 2.7
> Flowgraph: [`demo_s5_decode_signal.grc`](demo_s5_decode_signal.grc)

Até aqui o recetor sabia *onde* começa a trama, mas não *quanto* ela dura nem *como* está modulada. Essa informação está no primeiro símbolo após o preâmbulo: o **campo SIGNAL**. Esta semana trata de o descodificar e de perceber como essa decisão viaja pelo fluxo em forma de *tags* e de valores "moles".

---

## 1. Anatomia do campo SIGNAL

Um único símbolo OFDM, sempre em **BPSK com código convolucional de taxa 1/2** (24 bits de informação → 48 bits codificados → 48 subportadoras de dados × 1 bit):

| Campo | Bits | Posição | Conteúdo |
|---|---|---|---|
| RATE | 4 | 0–3 | esquema MCS (ver tabela) |
| Reservado | 1 | 4 | 0 |
| LENGTH | 12 | 5–16 | comprimento da PSDU **em bytes** (máx. 4095) |
| Paridade | 1 | 17 | paridade par sobre os bits 0–16 |
| Tail | 6 | 18–23 | zeros que fecham o treliça do Viterbi |

### 1.1 Tabela RATE

Os 4 bits são transmitidos do menos significativo para o mais significativo; o código da implementação reconstrói o inteiro `r` com `r |= bit_i << i`:

| RATE (R1..R4) | `r` | Débito | Modulação | Taxa | $N_{DBPS}$ (bits/símbolo) |
|---|---|---|---|---|---|
| 1101 | 11 | 6 Mbit/s | BPSK | 1/2 | 24 |
| 1111 | 15 | 9 Mbit/s | BPSK | 3/4 | 36 |
| 0101 | 10 | 12 Mbit/s | QPSK | 1/2 | 48 |
| 0111 | 14 | 18 Mbit/s | QPSK | 3/4 | 72 |
| 1001 | 9 | 24 Mbit/s | 16-QAM | 1/2 | 96 |
| 1011 | 13 | 36 Mbit/s | 16-QAM | 3/4 | 144 |
| 0001 | 8 | 48 Mbit/s | 64-QAM | 2/3 | 192 |
| 0011 | 12 | 54 Mbit/s | 64-QAM | 3/4 | 216 |

> O log do `frame_equalizer` (com `debug=True`) imprime os débitos da variante de **10 MHz** — `Encoding: 3/4.5/6/9/12/18/24/27 Mbit/s` — porque usa os mesmos códigos RATE com metade da largura de banda. Para 20 MHz valem os valores da tabela (6…54 Mbit/s): não é erro de descodificação, é a etiqueta do log.

### 1.2 Porque esta informação é indispensável

Sem o SIGNAL, o recetor não sabe:

- quantos bits cada subportadora transporta (1, 2, 4 ou 6) → não pode desmodular;
- como desfazer o *interleaving* (o padrão depende de $N_{BPSC}$ e $N_{CBPS}$);
- que bits foram picados (*puncturing*) → não pode reconstituir a treliça;
- **quantos símbolos** pertencem à trama → não sabe quando parar.

E é por isso que o SIGNAL é transmitido no MCS **mais robusto** e protegido por um bit de paridade: um erro aqui invalida a trama inteira, enquanto um erro nos dados pode ser corrigido pelo FCS/repetição.

Número de símbolos de dados:

$$N_{sym}=\left\lceil\frac{16+8\cdot L+6}{N_{DBPS}}\right\rceil,$$

onde 16 bits são o campo SERVICE e 6 o *tail*. Duração total:

$$T_{trama}=16\ \mu s\;(\text{preâmbulo})+4\ \mu s\;(\text{SIGNAL})+N_{sym}\cdot4\ \mu s.$$

Exemplo: 38 B de MPDU (24 de cabeçalho + 10 de payload + 4 de FCS) a 6 Mbit/s → $L=38$ → $N_{sym}=\lceil(16+304+6)/24\rceil=14$ → $T=16+4+56=76\ \mu s$.

---

## 2. Onde é descodificado (e o que isso implica)

No artigo, o SIGNAL tem um bloco próprio (`OFDM Decode Signal`) que anota o stream com uma tag `(encoding, length)`. Na implementação atual essa função está **dentro do `frame_equalizer`**:

1. ao símbolo de índice 2 (`d_current_symbol == 2`), o equalizador trata os 48 valores como BPSK 1/2;
2. desfaz o *interleaving* com o padrão fixo de 48 posições do SIGNAL;
3. descodifica com o Viterbi (taxa 1/2, sem puncturing);
4. verifica a paridade: `parity = XOR(bits[0..16])` vs `bits[17]`;
5. extrai `LENGTH` dos bits 5–16 e calcula `d_frame_symbols` e `d_frame_encoding`.

Se a paridade falhar, imprime `SIGNAL: wrong parity` (visível com `debug=True` no equalizador) e **não** injeta as tags — o `decode_mac` não tem como saber o comprimento e a trama perde-se silenciosamente.

### 2.1 As tags que o equalizador injeta

Assim que o SIGNAL é validado, o equalizador adiciona ao item de saída um dicionário de metadados:

| Tag | Tipo | Origem / significado |
|---|---|---|
| `frame bytes` | uint64 | LENGTH (bytes da PSDU) |
| `encoding` | uint64 | índice MCS 0–7 (BPSK 1/2 … 64-QAM 3/4) |
| `snr` | double | SNR estimada pelo equalizador |
| `nominal frequency` | double | frequência nominal ($f_c$) |
| `frequency offset` | double | CFO em **Hz**, calculada a partir da tag `wifi_start` |
| `beta` | double | fase comum corrigida no último símbolo (rad) |
| `csi` | c32vector(52) | resposta de canal estimada por subportadora |

Estas tags são o equivalente moderno da "tag com encoding e length" descrita no artigo, e são também a matéria-prima do Wireshark (via metadados do PDU) e das métricas de qualidade.

### 2.2 As duas tags `wifi_start`

| Origem | Valor | Unidades |
|---|---|---|
| `sync_short` | $\hat{\omega}_{short}$ | rad/amostra |
| `sync_long` | $\hat{\omega}_{short}-\hat{\omega}_{long}$ (resíduo) | rad/amostra |

Conversão: $\Delta f[\text{Hz}]=\omega\cdot f_s/(2\pi)$. O equalizador usa exatamente esta expressão (`value * d_bw / (2π)`) — com `bw` = taxa de amostragem.

---

## 3. *Soft-bits*: o que sai do equalizador

O `frame_equalizer` não entrega pontos da constelação: entrega **48 inteiros de 8 bits por símbolo** (`byte`, vlen 48). Cada valor é a quantização de uma observação de canal:

- o **sinal** diz qual é o bit mais provável;
- o **módulo** diz quão longe o ponto está da fronteira de decisão (confiança).

Para BPSK em ruído gaussiano, o valor ótimo é a *log-likelihood ratio*:

$$\Lambda=\ln\frac{P(b=0\mid y)}{P(b=1\mid y)}=\frac{2y}{\sigma^{2}},$$

pelo que a quantização em 8 bits preserva a informação essencial. O Viterbi trabalha diretamente com estas métricas; usar decisões "duras" (apenas o sinal) custa tipicamente **~2 dB** de SNR.

É por isso que o `decode_mac` recebe `byte vlen 48` e não `complex vlen 48` — a decisão já foi tomada e quantizada, mas a confiança foi preservada. O `tag_dbg_eq` no demo mostra as tags associadas a cada símbolo; o `msg_dbg` mostra os metadados do PDU final.

---

## 4. O que o demo mostra

| Elemento | Para que serve |
|---|---|
| `tag_dbg_eq` (no `eq` output) | lista `frame bytes`, `encoding`, `snr`, `frequency offset`, `csi` por trama |
| `msg_dbg` (`decode` → message debug) | dicionário do PDU entregue ao MAC |
| `decode` com `log=True` | linha por trama descodificada (sucesso) |
| `const_post` | constelação dos 48 dados (Semana 4) |

### 4.1 Experiências

1. **SIGNAL válido vs inválido.** Subir o ruído até a trama deixar de descodificar. Com `debug=True` no equalizador vê-se `SIGNAL: wrong parity` *antes* de qualquer erro de payload: o SIGNAL é o primeiro a falhar porque toda a trama depende dele.
2. **LENGTH coerente.** Alterar `payload_len` e confirmar que `frame bytes` cresce exatamente `payload_len + 28` (24 de cabeçalho + 4 de FCS). Nota: o PDU entregue ao MAC tem `payload_len + 24` bytes — o FCS é verificado e removido pelo `decode_mac`.
3. **encoding coerente.** Trocar o MCS no seletor e confirmar que a tag `encoding` acompanha (0…7).
4. **CFO em Hz.** Com `master_v5` e `cfo_hz = 50 kHz`, ler `frequency offset` e comparar com o valor injetado (a estimativa deve ficar na mesma ordem de grandeza).
5. **Soft vs hard.** Acrescentar um `blocks_file_sink` (tipo byte, vlen 48) à saída `eq` e guardar 48 bytes por símbolo; depois, offline, comparar um Viterbi com métricas moles e outro com decisões duras. É a experiência que quantifica os ~2 dB.

---

## 5. Análise crítica

- **O SIGNAL é o ponto único de falha.** Um erro de 1 bit em 24 invalida uma trama de 1500 bytes. A redundância é mínima (1 bit de paridade não corrige, só deteta); a robustez vem do MCS baixo e do baixo número de bits.
- **Tags são a espinha dorsal.** Sem as tags, o `decode_mac` não sabe onde começa a trama nem quantos símbolos ler. Depurar um recetor OFDM é, em grande parte, seguir tags: `wifi_start` → `frame bytes`/`encoding` → PDU.
- **`csi` é ouro escondido.** A resposta de canal por subportadora permite calcular SNR por subportadora, detectar desvanecimento e comparar algoritmos de equalização — é a base de qualquer análise séria de desempenho.
- **8 bits são suficientes?** Para BPSK/QPSK sim; para 64-QAM, com relações sinal/ruído pequenas, a quantização em 8 bits introduz perda mensurável. É uma limitação a mencionar.

---

## 6. *Problem solving*

| Sintoma | Causa provável | Verificação |
|---|---|---|
| Deteta (`wifi_start`) mas não há `frame bytes` | paridade do SIGNAL falhou | `debug=True` no equalizador |
| `frame bytes` correto mas nada descodifica | MCS da tag não suportado / payload corrompido | `encoding` na tag; SNR |
| Tags todas a 0 | equalizador a processar ruído (falso trigger) | contar tags vs tramas |
| PDU com metadados mas sem bytes | `msg_dbg` só mostra o dicionário | ligar `en_uvec`/ver `parse_mac` |
| `frequency offset` absurdo | tag de `sync_long` com `delay` errado | Semana 3 |

---

## 7. Checklist e commits sugeridos

- [ ] Tabela RATE completa (bits, `r`, débito, modulação, $N_{DBPS}$)
- [ ] Explicação do LENGTH, da paridade e do tail
- [ ] Fórmula de $N_{sym}$ e exemplo numérico
- [ ] Lista das tags injetadas pelo equalizador
- [ ] Distinção entre tags do `sync_short` e do `sync_long`
- [ ] Justificação dos *soft-bits* (LLR, ~2 dB)

```
s5: tags do equalizer (frame bytes, encoding, snr, cfo, csi) no tag debug
s5: descricao do SIGNAL (RATE/LENGTH/paridade) e da sua descodificacao
s5: dump dos 48 soft-bits por simbolo para analise offline
s5: caso de falha de paridade do SIGNAL
```
