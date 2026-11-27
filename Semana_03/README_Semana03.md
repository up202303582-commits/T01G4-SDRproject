# Semana 3 — OFDM Sync Long: *matched filter*, CFO e o atraso do ramo de entrada

> **CDIG 2025/26 · Projeto SDR · recetor OFDM IEEE 802.11a/g/p em GNU Radio**
> Base: Bloessl *et al.* (SRIF'13), secções 2.3 e 2.4
> Flowgraph: [`demo_s3_sync_long.grc`](demo_s3_sync_long.grc)

Depois de a válvula abrir, o stream ainda não está utilizável: as amostras estão **rodadas por um desvio de frequência** e não se sabe onde começa cada símbolo OFDM. O `OFDM Sync Long` resolve as duas coisas com uma única operação — correlacionar a janela recebida com o *Long Training Sequence* conhecido.

---

## 1. Porque é necessário corrigir a frequência

Osciladores de TX e RX têm erros de fabrico e derivas térmicas. Um erro relativo $\varepsilon$ (em ppm) traduz-se numa CFO

$$\Delta f = \varepsilon\cdot f_c .$$

A 5 GHz, $\pm20$ ppm → $\pm100$ kHz por extremo; dois equipamentos independentes podem estar a 200 kHz um do outro. Em amostras, isso é uma rotação de fase acumulada

$$\varphi[n] = 2\pi\frac{\Delta f}{f_s}\,n .$$

Consequências, por ordem de gravidade:

1. **Perda de ortogonalidade (ICI)**: dentro de um símbolo de 3,2 µs, uma CFO de 100 kHz acumula $2\pi\cdot10^5\cdot3{,}2\times10^{-6}=2{,}01$ rad — as subportadoras deixam de ser ortogonais e aparece interferência entre elas.
2. **Erro de fase comum por símbolo**: mesmo corrigindo o grosso, um resíduo de 1 kHz roda a constelação $\approx1{,}4^\circ$ por símbolo ($2\pi\Delta fT_{sym}$); ao fim de 20 símbolos são $\approx29^\circ$ — incompatível com QAM.
3. **Erro de temporização** (SFO): o relógio de amostragem também tem ppm; o efeito é uma fase **linear com o índice da subportadora**, que cresce símbolo a símbolo.

É por isso que o recetor estima a CFO em **duas escalas**: uma estimativa grosseira no STS (robusta, pouca precisão) e uma fina no LTS (precisa), e depois segue o resíduo com os pilotos (Semana 4).

---

## 2. Estimativa grosseira no STS

Durante o STS, $s[n]\approx s[n+16]$. Com CFO e ruído,

$$a=\sum_{n=0}^{N_{short}-17}s[n]\,s^{*}[n+16]\approx e^{j\omega 16}\sum|s[n]|^{2},$$

porque todos os produtos partilham a mesma rotação de 16 amostras. Logo

$$\hat{\omega}=\frac{1}{16}\arg\{a\},\qquad
\hat{\Delta f}=\frac{\hat{\omega}}{2\pi}f_s
=\frac{f_s}{2\pi\cdot16}\arg\{a\}. \tag{4}$$

Notas de rigor:

- usa-se o **argumento da soma** e não a média dos argumentos: amostras de pequena amplitude (mais afetadas por ruído) ficam automaticamente desvalorizadas — é a razão invocada no artigo;
- o alcance não ambíguo é $|\Delta f|<f_s/(2\cdot16)=625$ kHz;
- a estimativa é aplicada amostra a amostra em `sync_short`:
  $$s'[m]=s[m]\,e^{-j\hat{\omega}m}. \tag{5}$$

## 3. Estimativa fina no LTS

O LTS repete o mesmo padrão de 64 amostras duas vezes. Sejam $c_1,c_2$ os dois picos do *matched filter* separados de $D=64$. Então

$$\hat{\omega}_{fino}=\frac{1}{64}\arg\{c_1 c_2^{*}\},\qquad
\hat{\Delta f}_{fino}=\frac{\hat{\omega}_{fino}}{2\pi}f_s,$$

com alcance $|\Delta f|<f_s/(2\cdot64)=156{,}25$ kHz. É cerca de 4× mais preciso do que a estimativa do STS (a base de comparação é 4× maior), mas tem um alcance 4× menor — daí a ordem obrigatória: **primeiro o grosso, depois o fino**.

### 3.1 O que a implementação faz, linha por linha

`sync_long` (fonte `maint-3.10`):

1. **Filtro emparelhado** com os 64 coeficientes do LTS (`d_fir`, taps `LONG`), aplicado às primeiras `SYNC_LENGTH` amostras da janela.
2. **`search_frame_start()`**: ordena as correlações por magnitude, olha para as 4 maiores e procura um par cujos índices distem **64** — quando o encontra, retorna imediatamente; 63 e 65 são aceites como *fallback*, mas a estimativa pode ser substituída por uma combinação posterior. Se encontrar:
   - $\text{d\_frame\_start}=\min(\text{índices})$;
   - $\hat{\omega}_{fino}=\arg(c_i c_k^*)/\Delta$, com $\Delta$ o espaçamento encontrado.
   Se não encontrar, $\text{d\_frame\_start}=\text{SYNC\_LENGTH}$ (pior caso — ver §6).
3. **Cópia e correção**: emite `in_delayed` corrigido por $e^{j n\hat{\omega}_{fino}}$, mantendo as **128 amostras** dos dois LTS e, a partir daí, removendo o CP:

$$s\leftarrow\big(\underbrace{s[n_P+16],\dots,s[n_P+79]}_{\text{símbolo }0},\;
\underbrace{s[n_P+96],\dots}_{\text{símbolo }1}\big). \tag{8}$$

4. **Tag**: no instante em que o alinhamento é confirmado, insere `wifi_start` com valor
   $$\Delta\omega_{res}=\hat{\omega}_{short}-\hat{\omega}_{long}\quad[\text{rad/amostra}].$$
   O equalizador converte-o em Hz com $\Delta f=\Delta\omega\cdot f_s/(2\pi)$ e usa-o para compensar a deriva de amostragem.

### 3.2 Porque existe uma entrada atrasada

O `sync_long` não pode alinhar enquanto não viu `SYNC_LENGTH` amostras. Quando termina a busca, essas amostras **já foram consumidas**. O ramo atrasado (`blocks_delay`) guarda-as e o bloco copia do ramo atrasado, não da entrada principal — é a forma mais simples de "voltar atrás" num fluxo em que não se pode recuar.

Consequência de projeto:

$$\boxed{D_{delay} = \text{SYNC\_LENGTH}}$$

Se $D$ for menor, a cópia começa **dentro** da trama (perde-se o início do LTS); se for maior, começa antes (amostras de ruído). Em ambos os casos o equalizador estima o canal com um "LTS" que não é o LTS.

---

## 4. Alinhamento de símbolo (secção 2.4)

Com os picos encontrados, o primeiro símbolo de dados começa em

$$n_P=\max(N_P)+64, \tag{7}$$

porque o último pico do *matched filter* está 64 amostras antes do fim do LTS. O "+64" **não é arbitrário**: o pico ocorre quando a janela de 64 coeficientes está completamente sobre o padrão; o fim do padrão é o início do primeiro símbolo de dados (o SIGNAL).

O bloco entrega então 64 amostras por símbolo (o CP de 16 é descartado), o que explica a ligação direta a `blocks_stream_to_vector(num_items=64)`.

### 4.1 Porque *matched filter* aqui e autocorrelação na deteção

| | Deteção (STS) | Alinhamento (LTS) |
|---|---|---|
| Operação | $s[n]s^*[n-16]$, 1 multiplicação complexa/amostra | $\sum_{k=0}^{63} s[n+k]\,LT[k]$, 64 multiplicações/amostra |
| Amostras processadas | **todas** (20 Msps contínuos) | só $\le$ 540 símbolos por trama |
| Precisão | suficiente para "abrir a janela" | precisa ao nível da amostra |
| Sequência conhecida? | não é necessária | sim (LTS fixo) |

O artigo é explícito: o *matched filter* seria mais robusto para detetar, mas custaria 64× mais CPU no ponto onde o fluxo é contínuo. Usa-se apenas onde o fluxo já está reduzido.

### 4.2 A Figura 3

O ramo de diagnóstico `ltf_mf` (FIR com os mesmos 64 coeficientes) + `sink_ltf` reproduz a Figura 3: **dois picos estreitos e dominantes**, separados de 64 amostras (3,2 µs), com amplitude $\approx\sum|LT[k]|^2$. Fora deles, a correlação com o LTS é baixa (o STS não se correlaciona com o LTS). É esta separação enorme que permite alinhar com precisão de 1 amostra.

---

## 5. Protocolo experimental

| Experiência | Variável | O que medir |
|---|---|---|
| A | `sync_length` ∈ {128…480} com `delay` = `sync_length` | nº de tramas descodificadas; posição do 1.º pico |
| B | `delay` ∈ {0…480} com `sync_length` = 320 | nº de tramas (mostra a dependência $D=SL$) |
| C (master) | `cfo_hz` ∈ {−300…+300} kHz | valor da tag `wifi_start` em Hz; rotação da constelação antes do equalizador |
| D | `threshold` (fixo em 0,56) com CFO grande | se a deteção grosseira ainda acerta na janela |

### 5.1 Resultados medidos

Condições: simulação, 20 Msps, BPSK 1/2, 10 B, 1 trama/100 ms, 4 s, ruído 0,01, $\theta=0{,}56$.

**Varrimento de `sync_length` (com `delay` = `sync_length`):**

| `sync_length` | Tramas descodificadas | Leitura |
|---|---|---|
| 200 | 0 | janela não contém os dois picos → alinhamento por defeito |
| 216 | 0 | idem |
| **224** | **36** | **mínimo medido** (2,8 símbolos) |
| 240 | 36 | valor por defeito do bloco |
| 320 | 36 | valor dos exemplos oficiais (4 símbolos) |
| 480 | 36 | margem extra, custo de latência |

**Varrimento do `delay` (com `sync_length` = 320):**

| `delay` | Tramas | Leitura |
|---|---|---|
| 0 | 0 | cópia começa 320 amostras antes da trama |
| 160 | 0 | — |
| 240 | 0 | começa 80 amostras antes |
| 300 | 0 | 20 amostras antes: já falha |
| **320** | **35** | correto |
| 340 | 20 | 20 amostras depois: degrada (LTS incompleto) |
| 400 | 0 | 80 amostras depois |

**Conclusão quantitativa:** o erro tolerável é de poucas amostras ($\pm20$ degrada, $\pm80$ quebra). Isto mostra que:
- `delay` não é um parâmetro "de afinação": é uma **constante do bloco**, igual a `SYNC_LENGTH`;
- `sync_length = 240` funciona, mas **só com `delay = 240`**. O par (240, 240) é válido; o par (320, 240) — que aparece em muitos flowgraphs de laboratório — não é.

### 5.2 Porque 224 é o mínimo

A busca procura dois picos separados de 64 amostras dentro da janela de `SYNC_LENGTH`. O primeiro pico ocorre $\approx$ 33 amostras depois do *trigger* do `sync_short` (atraso de grupo da média móvel de 48 + o patamar mínimo), e o segundo 64 depois. Logo

$$SL_{min} \approx 33+64+64+(\text{margem}) \approx 224.$$

Abaixo disso `search_frame_start` não encontra par e usa $d_{frame\_start}=SYNC\_LENGTH$, deslocando a extração — o equalizador recebe amostras que não são LTS e a estimativa de canal é ruído.

---

## 6. Análise crítica: o que esperar e porquê

- **CFO pequena (< 10 kHz):** a constelação *antes* do equalizador roda lentamente ao longo da trama; a tag `wifi_start` tem valor próximo de 0 rad/amostra.
- **CFO grande (100 kHz):** a rotação é de $\approx144^\circ$ por símbolo ($2\pi\Delta f T_{sym}$). Os pilotos do equalizador (Semana 4) corrigem-na, mas a margem de decisão degrada-se; acima de $\approx300$ kHz a estimativa grosseira começa a errar por ambiguidade.
- **`sync_length` grande:** mais latência e mais CPU, mas robusto a variações do atraso de grupo do detetor. 320 é o compromisso dos exemplos oficiais.
- **`delay` errado:** falha catastrófica e fácil de diagnosticar — a constelação pós-FFT fica "em nuvem" e as tags existem (a deteção funcionou), mas nada descodifica. É um bom exercício de *debug*: separar "não deteta" (Semana 2) de "deteta mas não alinha" (Semana 3).

---

## 7. *Problem solving*

| Sintoma | Causa provável | Verificação | Correção |
|---|---|---|---|
| Tags existem, constelação é ruído | `delay` ≠ `sync_length` | comparar os dois valores | igualar |
| Nenhuma tag `sync_long` | `sync_length` < 224 (20 MHz) | reduzir para 160 e ver que piora | usar 320 |
| Picos do LTF ausentes em `sink_ltf` | sinal muito fraco ou CFO enorme | ver `sink_c` | subir ganho / reduzir CFO |
| Constelação roda ao longo da trama | CFO residual grande | valor da tag em Hz | ver Semana 4 (pilotos) |
| Funciona a 10 MHz e falha a 20 MHz | constantes de tempo duplicadas | recalcular 160/320 | escalar `sync_length` com $f_s$ |

**Nota sobre 802.11p (10 MHz):** todos os comprimentos duplicam em duração e metade em amostras. O preâmbulo passa a 160 amostras; `sync_length = 240` continua a funcionar, mas o valor "correto" é 160 e o `delay` tem de o acompanhar. Nunca copiar constantes entre as duas configurações.

---

## 8. Checklist e commits sugeridos

- [ ] Equações (4), (5), (7) e (8) explicadas com o código correspondente
- [ ] Justificação do "+64" e do CP removido
- [ ] Tabela `sync_length` e tabela `delay` com medições
- [ ] Figura 3 reproduzida (`sink_ltf`)
- [ ] Explicação da tag `wifi_start` em rad/amostra e em Hz

```
s3: sync_long + ramo atrasado (delay=sync_length) no fluxo
s3: sonda de matched filter com os 64 taps do LTS (Fig. 3)
s3: medidas SL 200-480 e delay 240-400; minimo SL=224
s3: CFO injetado no master e leitura da tag em Hz
```
