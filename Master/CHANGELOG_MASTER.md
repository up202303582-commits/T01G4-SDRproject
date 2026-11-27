# CHANGELOG — flowgraph *master* (v1 → v6)

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Versão de referência: GNU Radio **3.10.12** + `gr-ieee802-11` (build 2025-09) + `gr-foo`
> Todos os ficheiros desta pasta compilam com `grcc` (0 erros) e foram executados 10 s cada em modo headless, **sem hardware** (cadeia TX+RX simulada), sem erros de runtime.

## Como usar

```bash
grcc -o /tmp/out Master/master_v6.grc     # valida e gera o .py
gnuradio-companion Master/master_v6.grc   # abre no GRC
```

Cada versão é **autónoma**: inclui transmissor 802.11a, canal, recetor e instrumentação. Não há dependências de ficheiros externos nem de periféricos. Para trabalhar com o Pluto, substituir o bloco `channel` (saída do `gain_block`) pela fonte `iio_pluto_source` já usada no projeto original — os parâmetros de amostragem são os mesmos (20 Msps).

---

## Quadro-resumo

| Versão | Acrescenta | Parâmetros novos | Estado |
|---|---|---|---|
| **v1** | cadeia Schmidl & Cox + probes de $c[n]$ e $p[n]$ | `N_win`, `noise`, `gain`, `payload_len`, `interval`, `pad_tail` | ✅ compila · ✅ deteta e descodifica |
| **v2** | `sync_short` + tags + diagnóstico de deteção | `threshold`, `min_plateau`, `encoding` | ✅ compila · ✅ varrimento de threshold medido |
| **v3** | `sync_long` + ramo atrasado + picos do *matched filter* | `sync_length`, `delay`, `cfo_hz` (só master) | ✅ compila · ✅ SL/delay medidos |
| **v4** | FFT + equalizer + constelações pré/pós + espectro | `algo` (LS/LMS/COMB/STA) | ✅ compila · ✅ MCS×ruído medido |
| **v5** | `decode_mac` + metadados + tags por símbolo + pcap | — | ✅ compila · ✅ PDUs válidos |
| **v6** | `parse_mac` → pcap + Socket PDU TCP | porta 52001 | ✅ compila · ✅ pcap/socket |

---

## v1 — base funcional + instrumentação da Semana 1

**Objetivo.** Ter um fluxo que gera tramas 802.11a e mostra a métrica de deteção, sem hardware.

**Delta face a v0 (projeto inicial).** O projeto original dependia do Pluto e de ficheiros `.cfile`; aqui o transmissor é sintetizado em GRC (`ieee802_11_mac` → `ieee802_11_mapper` → alocador → IFFT → CP) e o canal é AWGN. A cadeia de deteção usa as mesmas equações (1)–(3), com a **normalização da implementação** ($p$ sobre $N_{win}+16$).

**O que observar.** $c[n]$ com patamar em $\approx0{,}75$ e ruído em $\approx0{,}11$; $p[n]$ com rampa de 64 amostras.

**Verificação.** $c_{max}$ medido: 0,497 / 0,668 / 0,755 para $N_{win}=16/32/48$ — coincide com o teto $N/(N+16)$.

**Correção de runtime.** O `foo_packet_pad2` é *tagged stream* e exige um buffer de saída com espaço para a trama inteira mais `pad_front`/`pad_tail`; com o buffer por omissão o thread aborta com `Buffer too small for min_noutput_items` e o fluxo pára em silêncio. Todos os flowgraphs trazem `minoutbuf = 65536` no bloco `pad`.

**Limitação conhecida.** A divisão por $p$ não tem guarda: no silêncio absoluto, $c[n]$ pode ser `inf`/`nan`.

---

## v2 — controlo dinâmico de `threshold`, ganho e diagnóstico do Sync Short

**Objetivo.** Transformar a métrica numa decisão e poder variar os parâmetros em tempo real.

**Delta.** Entra o `ieee802_11_sync_short` com `threshold` e `min_plateau` em *sliders*; o ganho passa a ser aplicado **antes** do ruído (emula o LNA); o `Tag Debug` mostra `wifi_start` e o respetivo valor (CFO grosseira em rad/amostra).

**Medições (4 s ≈ 40 tramas, BPSK 1/2).**

| `threshold` | 0,40 | 0,56 | 0,70 | 0,75 | 0,78 | 0,80 |
|---|---|---|---|---|---|---|
| tramas | 37 | 36 | 36 | 36 | 0 | 0 |

**Conclusão documentada.** O teto $48/64=0{,}75$ explica a falha total acima de 0,75 — não é "má sorte" nem ruído.

**Limitação conhecida.** `N_win` não é aplicável a quente (os taps são calculados na construção): mudar o slider obriga a reiniciar o flowgraph.

---

## v3 — Sync Long, atraso variável e visualização dos picos

**Objetivo.** Alinhar símbolos e corrigir a CFO fina, com instrumentação da Figura 3.

**Delta.** Entram `sync_long`, o ramo atrasado (`delay`) e uma sonda com os **64 coeficientes do LTS** (`ltf_mf` → `sink_ltf`), que reproduz os dois picos separados de 64 amostras. O master acrescenta `cfo_hz`, injetado no canal.

**Medições (4 s).**

| `sync_length` (com `delay` = SL) | 200 | 216 | 224 | 240 | 320 | 480 |
|---|---|---|---|---|---|---|
| tramas | 0 | 0 | 36 | 36 | 36 | 36 |

| `delay` (SL = 320) | 240 | 300 | 320 | 340 | 400 |
|---|---|---|---|---|---|
| tramas | 0 | 0 | 35 | 20 | 0 |

**Conclusão documentada.** $D = \text{SYNC\_LENGTH}$ é uma invariante do bloco; o mínimo de SL a 20 Msps é 224 amostras (2,8 símbolos). O par (320, 240) — comum em flowgraphs de laboratório — **não funciona**.

**Limitação conhecida.** Com `cfo_hz` muito grande (> 300 kHz) a estimativa grosseira entra na zona de ambiguidade ($\pm f_s/32$).

---

## v4 — FFT, equalizer e constelações

**Objetivo.** Ver o sinal no domínio da frequência e comparar algoritmos/constelações.

**Delta.** `stream_to_vector(64)` → `fft_vxx` → `frame_equalizer`; `const_pre` (64 bins antes do equalizer), `const_post` (48 pontos entregues pela mensagem `symbols`), `spec` (espectro) e seletor `algo` (LS/LMS/COMB/STA).

**Medições (3 s ≈ 30 tramas).**

| MCS | ruído 0,10 | 0,30 | 0,50 |
|---|---|---|---|
| BPSK 1/2 | 27 | 27 | 27 |
| QPSK 1/2 | 27 | 27 | 19 |
| 16-QAM 1/2 | 27 | 7 | 0 |
| 64-QAM 2/3 | 26 | 0 | 0 |

**Conclusão documentada.** A limitação do artigo (equalização de magnitude) já não impede QAM em canal plano, mas traduz-se numa **sensibilidade 4–6 dB maior por escalão**. É a leitura correta a colocar no relatório.

**Limitação conhecida.** `const_post` só recebe símbolos de dados (o SIGNAL não é publicado na mensagem `symbols`).

---

## v5 — debug de tags, SIGNAL e *soft-bits*

**Objetivo.** Fechar a cadeia até ao PDU e tornar visível a informação do SIGNAL.

**Delta.** Entram `decode_mac` (com `log=True`), `msg_dbg` (metadados do PDU) e `tag_dbg_eq` (tags por símbolo: `frame bytes`, `encoding`, `snr`, `frequency offset`, `csi`). O pcap passa a ser escrito.

**O que observar.** `frame bytes = payload_len + 28` (24 B de cabeçalho + 4 de FCS); `encoding` acompanha o MCS escolhido; com CFO injetado, `frequency offset` fica na ordem de grandeza correta.

**Limitação conhecida.** O `decode_mac` é monolítico: não há forma de injetar um Viterbi alternativo sem alterar o código C++ (candidato a "novo bloco" numa fase seguinte).

---

## v6 — recetor completo com Socket PDU/Wireshark

**Objetivo.** Versão final (até à Semana 6): tudo o que as semanas anteriores pedem, num único fluxo.

**Delta.** `parse_mac` → `foo_wireshark_connector` (pcap, DLT 127) + `network_socket_pdu` (TCP 52001). Painel de parâmetros completo.

**Verificação.** 36–37 tramas por 4 s a BPSK 1/2 com ruído 0,01; PDUs de 34 B (payload 10 B, FCS removido) com `src 23:23:…` e `dst 42:42:…`; pcap legível no Wireshark (Data, subtipo 0x20).

**Limitações conhecidas.**
- sem cabeçalho radiotap → o pcap não transporta RSSI/SNR;
- um ficheiro pcap por execução (`append=False`);
- o socket é servidor TCP: quem ligar tem de o fazer depois de o fluxo arrancar.

---

## Dívida técnica e próximos passos (S7+)

| Item | Impacto | Esforço |
|---|---|---|
| Guarda na divisão de $c[n]$ ($p<\epsilon$) | evita `nan` em silêncio | baixo (bloco `python` ou `divide` com clamp) |
| `N_win` aplicável a quente | varrimento sem reiniciar | médio (bloco próprio com `set_taps`) |
| Radiotap no pcap | métricas físicas no Wireshark | médio |
| `decode_mac` desmontado em blocos (desinterleaver, Viterbi, descrambler) | experimentação real de algoritmos | alto (C++/GRC) |
| Reamostragem/AGC na cadeia de receção | robustez com Pluto | médio |

## Histórico de versões do repositório

```
v1  cadeia Schmidl & Cox + TX/RX simulado (S1)
v2  sync_short + tags + sliders de threshold/ganho (S2)
v3  sync_long + delay + sonda do LTS + CFO injetado (S3)
v4  FFT + equalizer + constelações + MCS/algoritmo (S4)
v5  decode_mac + metadados + pcap (S5)
v6  parse_mac + Socket PDU 52001 + painel completo (S6)
```
