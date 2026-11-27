# Semana 6 — Do *soft-bit* à MPDU: desinterleaving, Viterbi, descrambler e MAC

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Base: Bloessl *et al.* (SRIF'13), secção 2.8
> Flowgraph: [`demo_s6_mac_pipeline.grc`](demo_s6_mac_pipeline.grc)

A última etapa é a que transforma 48 números por símbolo numa **trama Ethernet/Wi-Fi visível no Wireshark**. É também a que tem mais "encanamento" herdado da norma: permutações, puntura, código convolucional e um LFSR cujo estado o recetor tem de adivinhar — e adivinha, sem que lhe seja dito.

---

## 1. O que está dentro do `OFDM Decode MAC`

Um único bloco GRC esconde seis operações:

```
48 soft-bits/símbolo
   │
   ├─(1) desmodulação  ──►  bits do MCS indicado pelo SIGNAL
   ├─(2) desinterleaving (2 permutações, dependem de N_BPSC e N_CBPS)
   ├─(3) depuncturing  (reinserção dos bits apagados)
   ├─(4) Viterbi        (K=7, g0=133₈, g1=171₈)
   ├─(5) descrambling   (LFSR x⁷+x⁴+1, estado deduzido do SERVICE)
   └─(6) montagem da MPDU + verificação do FCS  ──► PDU
```

### 1.1 Desmodulação

Para cada subportadora, o *soft-bit* é interpretado segundo a constelação do MCS: BPSK (1 bit), QPSK (2), 16-QAM (4) ou 64-QAM (6). Por símbolo OFDM produzem-se $N_{CBPS}=48\cdot N_{BPSC}$ bits codificados.

### 1.2 Desinterleaving

O transmissor entrelaça os bits para que um *burst* de erros no canal se reparta por várias palavras de código. São duas permutações:

**Primeira** (blocos de 16 colunas, $N_{ROW}=N_{CBPS}/16$):

$$i = N_{ROW}\cdot\left(k \bmod 16\right) + \left\lfloor \frac{k}{16}\right\rfloor,
\qquad k=0,\dots,N_{CBPS}-1.$$

**Segunda** (rotação por $s=\max(N_{BPSC}/2,1)$):

$$j = s\left\lfloor\frac{i}{s}\right\rfloor
+ \left(i + N_{CBPS} - \left\lfloor\frac{16\,i}{N_{CBPS}}\right\rfloor\right)\bmod s.$$

O recetor aplica a permutação inversa. Para o SIGNAL (BPSK 1/2, 48 bits) a implementação usa simplesmente a tabela fixa

```
0  3  6  9 12 15 18 21 24 27 30 33 36 39 42 45
1  4  7 10 13 16 19 22 25 28 31 34 37 40 43 46
2  5  8 11 14 17 20 23 26 29 32 35 38 41 44 47
```

que é a transposição de uma matriz $16\times3$ — a mesma ideia, com $N_{BPSC}=1$.

### 1.3 Depuncturing e Viterbi

O código mãe é convolucional de taxa 1/2, $K=7$:

$$g_0=133_8=1011011_2,\qquad g_1=171_8=1111001_2.$$

As taxas 2/3 e 3/4 obtêm-se **apagando** bits (puncturing): em cada 4 bits codificados transmite-se 3 (taxa 2/3) ou em cada 6 transmite-se 4 (taxa 3/4). O recetor insere nos lugares apagados uma métrica neutra (0) e o Viterbi reconstrói o caminho mais provável na treliça — 64 estados.

O Viterbi implementado (`viterbi_decoder`) usa as métricas moles descritas na Semana 5; é aqui que esses 8 bits por valor se convertem em ganho real.

### 1.4 Descrambling — o truque mais elegante do 802.11a

O *scrambler* é um LFSR de 7 bits com realimentação

$$x^7 + x^4 + 1,$$

inicializado pelo transmissor com um estado pseudoaleatório. Sem o conhecer, o recetor não consegue descodificar. Mas:

- o campo **SERVICE** são os primeiros 16 bits da PSDU e os seus **primeiros 7 bits são zero** por norma (reservados);
- como o *scrambler* faz XOR da sequência com o estado atual, os primeiros 7 bits transmitidos são exatamente **os 7 bits do estado inicial** (0 XOR estado = estado);
- logo o recetor lê o estado diretamente dos primeiros 7 bits e a implementação usa uma **tabela de 128 entradas** para o mapear.

Não há negociação, nem campo de estado: a informação está implicitamente nos primeiros bits. Esta é a resposta completa à pergunta do guião *"como pode o recetor descodificar sem conhecer o estado inicial?"*.

### 1.5 Montagem da MPDU e FCS

O resultado é uma sequência de bytes cujo comprimento é o `LENGTH` do SIGNAL (não há delimitador no fluxo). O FCS é o **CRC-32** dos primeiros $L-4$ bytes; se não bater, a trama é descartada (ou marcada, consoante o modo).

---

## 2. Formato da trama 802.11 e delimitação

| Campo | Bytes | Conteúdo |
|---|---|---|
| Frame Control | 2 | versão, tipo, subtipo, flags |
| Duration/ID | 2 | duração (NAV) |
| Address 1 | 6 | destinatário |
| Address 2 | 6 | transmissor |
| Address 3 | 6 | BSSID / origem |
| Sequence Control | 2 | número de sequência e fragmento |
| Payload | variável | dados (LLC/SNAP + carga) |
| FCS | 4 | CRC-32 |

**Como se delimita uma trama?** Ao contrário do HDLC (flag `01111110` + *bit stuffing*) ou do Ethernet (preâmbulo + SFD + comprimento implícito), no 802.11 a delimitação é **hierárquica**:

1. o PHY marca o início com o preâmbulo (detetado pela autocorrelação);
2. o PHY diz o comprimento em **bytes** no SIGNAL (campo LENGTH);
3. o MAC confia nesse comprimento e verifica a integridade com o FCS.

Não existe um carácter de escape nem um "fim de trama" no ar: o comprimento vem de um campo físico, protegido por paridade, e a validação é estatística (CRC). É a diferença conceptual face ao que se aprendeu em Redes de Computadores — e a razão pela qual um erro no SIGNAL é fatal.

---

## 3. Saídas do recetor

| Saída | Bloco | Utilização |
|---|---|---|
| pcap (DLT 127) | `foo_wireshark_connector` + `blocks_file_sink` | abrir em Wireshark |
| Socket PDU TCP 52001 | `network_socket_pdu` (`TCP_SERVER`) | `netcat`, aplicação externa |
| Log | `ieee802_11_parse_mac` (`log=True`) | ver cabeçalhos no terminal |

### 3.1 Comandos

```bash
# terminal 1: correr o flowgraph
gnuradio-companion demo_s6_mac_pipeline.grc

# terminal 2: receber os PDUs em bruto
ncat 127.0.0.1 52001 | xxd | head

# Wireshark: abrir cdig_80211a.pcap (gerado na pasta do flowgraph)
# filtros úteis:
#   wlan.fc.type_subtype == 0x20     (Data)
#   wlan.addr == 23:23:23:23:23:23   (transmissor simulado)
#   frame.len == 34
```

> No Windows, o `network_socket_pdu` abre uma porta TCP: se o `ncat` não ligar, verificar a regra da *firewall* para o processo Python do Radioconda.

### 3.2 O que se espera ver

- `wlan.fc.type_subtype = 0x20` (Data), `wlan.addr` iguais aos configurados no `mac_tx` (`src 23:23:…`, `dst 42:42:…`);
- `frame.len = 34` para `payload_len = 10`: 24 B de cabeçalho MAC + 10 B de payload. O FCS (4 B) **não** aparece, porque é verificado e removido antes de o PDU ser emitido — a tag `frame bytes` (38) é que reporta o comprimento da PSDU recebida;
- payload repetido (`0x41` = `'A'`), porque o `msg_strobe` gera um vector de 0x41;
- como o FCS já foi validado a montante, o Wireshark não deve assinalar *bad FCS*; se a trama aparecer truncada ou "malformed", o problema está no comprimento do SIGNAL, não no CRC.

---

## 4. Protocolo experimental

| Experiência | Variável | Resultado esperado |
|---|---|---|
| A | `payload_len` 0→200 | `frame.len` acompanha (+24) e o pcap cresce |
| B | `encoding` BPSK→64QAM | mais débito, menos tramas com o mesmo ruído |
| C | `noise` 0,01→0,30 | com BPSK mantém-se; com 64-QAM cai a zero |
| D | `threshold` 0,56→0,80 | perde todas as tramas (Semana 2) |
| E | `delay` 320→300 | perde todas as tramas (Semana 3) |

As experiências D e E são a prova de que o pipeline completo não "compensa" erros de sincronismo: falha de forma limpa e total.

---

## 5. Análise crítica

- **O FCS é o juiz.** A percentagem de tramas com FCS válido é a única métrica honesta de PDR; contar PDUs emitidos sem verificar o FCS sobrestima o desempenho.
- **Sem radiotap não há RSSI/SNR no pcap.** O `foo_wireshark_connector` usa DLT 127 (802.11 puro). Para levar métricas físicas ao Wireshark seria necessário radiotap (DLT 127 com cabeçalho) — trabalho futuro.
- **O comprimento vem do PHY.** Um erro no SIGNAL produz um "comprimento" errado e o MAC tenta ler mais/menos símbolos; o FCS falha. É a manifestação prática da dependência PHY→MAC.
- **A cadeia é monolítica.** `decode_mac` junta seis operações; para experimentar (por exemplo) um Viterbi com métricas diferentes é preciso mexer no código C++, não no GRC. É a limitação de *design* mais relevante para o critério "tentativa de implementação de novo bloco".

---

## 6. *Problem solving*

| Sintoma | Causa | Correção |
|---|---|---|
| pcap vazio | caminho relativo/absoluto errado, ou nenhuma trama descodificada | usar caminho absoluto; confirmar `parse` com `log=True` |
| pcap com tramas mas Wireshark "malformed" | DLT errado | garantir `tech=127` |
| `ncat` sem dados | firewall / porta ocupada | permitir Python na firewall; mudar a porta |
| FCS inválido em muitas tramas | SNR marginal, CFO, QAM | baixar MCS, subir ganho, ver Semana 3/4 |
| Tramas duplicadas no pcap | falsos triggers do `sync_short` | subir `threshold`/`min_plateau` |
| Nada descodifica com tudo "a funcionar" | `delay` ≠ `sync_length` | igualar |

---

## 7. Checklist e commits sugeridos

- [ ] Lista das seis operações dentro do `decode_mac`
- [ ] Fórmulas do interleaving (as duas permutações) e exemplo do SIGNAL
- [ ] Viterbi: $K$, polinómios, puncturing 2/3 e 3/4
- [ ] Explicação completa do descrambler e do truque do SERVICE
- [ ] Formato MAC + discussão de delimitação (comparação com HDLC/Ethernet)
- [ ] pcap aberto no Wireshark com captura de ecrã
- [ ] Experiências D e E (falha total)

```
s6: pipeline completo ate parse_mac + pcap DLT 127 + socket 52001
s6: descricao do decode_mac (desinterleaving, Viterbi, descrambler)
s6: experiencia payload_len vs frame.len no Wireshark
s6: tabela de falhas (threshold/delay) no pipeline completo
```
