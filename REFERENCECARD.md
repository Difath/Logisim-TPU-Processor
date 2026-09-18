# Projeto 3 — Acelerador DOT (16-bit) | IAC
**Instituto Superior Técnico — Introdução à Arquitetura de Computadores**

**Reference Card & Cheat Sheet**

---

## 🗂️ Especificações Base (16 bits)

* **Modelo de Execução:** Single-cycle — uma instrução completa por ciclo de relógio.
* **Largura da palavra:** 16 bits (dados e instruções).
* **Memória de programa:** ROM de 256 palavras (endereço de 8 bits), uma instrução de 16 bits por palavra.
* **Banco de Registos:** 8 registos (`R0`–`R7`) de 16 bits. Leitura combinacional (assíncrona) e escrita síncrona no flanco de subida do relógio.
* **Leitura híbrida (vetorial + escalar):** o Banco de Registos expõe **5 saídas de dados em simultâneo** — `In_Rd`, `In_Rs1`, `In_Rs2`, `In_Rs1_next` e `In_Rs2_next`. O cálculo dos índices "seguintes" (`+1`) está encapsulado dentro do Banco de Registos (somadores de 3 bits), permitindo tratar pares de registos como vetores de dimensão 2 sem lógica adicional no datapath.
* **ALU:** bloco estritamente combinacional/matemático (um *Adder*, dois *Multiplier* e *Adders* de soma de produtos). O resultado é selecionado por um multiplexer final comandado por `ALUop`. O *overflow* é descartado (guardam-se apenas os 16 bits menos significativos).
* **Aritmética:** inteiros com sinal, em complemento para dois.

---

## 🗺️ Formato das Instruções e Codificação

> A codificação abaixo foi extraída diretamente do `proj3.circ` (splitter de campos no circuito `main` + lógica do `CONTROL_UNIT`). É a fonte de verdade do projeto.

### 1. Mapa de bits (campos físicos)

Cada instrução tem **16 bits**, repartidos pelo *splitter* do circuito `main` em cinco grupos fixos:

| Bits | `[15:13]` | `[12:10]` | `[9:7]` | `[6:4]` | `[3:0]` |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Largura** | 3 | 3 | 3 | 3 | 4 |
| **Tipo-R** | `funct3` | `rs2` | `rs1` | `rd` | `opcode` |
| **Tipo-I** (`li`) | ← `imm` (9 bits, com sinal) → | ← | `rd` | `opcode` |

* **`opcode` (4 bits, `[3:0]`)** — o **bit mais significativo (`[3]`) indica se a instrução usa a ALU** (`1` = caminho da ALU, `0` = não-ALU), e não o tipo de instrução em si. Os bits `[2:0]` identificam a instrução dentro da classe. Com apenas duas instruções/classes na ISA atual isto coincide com distinguir `li` de Tipo-R, mas a semântica é deliberadamente mais geral (extensibilidade):
  * `0000` → não-ALU (`li`)
  * `1000` → ALU / Tipo-R (`add`, `dot`, `dota`)
* **`funct3` (3 bits, `[15:13]`)** distingue as operações Tipo-R e é encaminhado diretamente para `ALUop`.
* **`imm` (9 bits, `[15:7]`)** apenas no `li`; é estendido com sinal de 9 → 16 bits (intervalo **−256 a +255**).

### 2. Tabela de opcodes / funct3

| Instrução | `opcode` | `funct3` (`ALUop`) | Operação |
| :--- | :---: | :---: | :--- |
| `li`   | `0000` | — | `R[rd] ← sext(imm)` |
| `add`  | `1000` | `000` | `R[rd] ← R[rd] + R[rs]` |
| `dota` | `1000` | `001` | `R[rd] ← R[rd] + R[rs1]·R[rs2] + R[rs1+1]·R[rs2+1]` |
| `dot`  | `1000` | `010` | `R[rd] ← R[rd]·R[rs] + R[rd+1]·R[rs+1]` |

### 3. Formatos lógicos / sub-formatos (sintaxe de assembly)

O hardware processa apenas **dois formatos físicos** (Tipo-I e Tipo-R), mas ao nível da sintaxe de assembly o Tipo-R subdivide-se em **dois sub-formatos lógicos**, consoante o número de registos explícitos:

* **Tipo-R3** (`dota rd, rs1, rs2`) — três registos distintos; usa os campos físicos `rd`, `rs1` e `rs2` tal como estão.
* **Tipo-R2** (`add rd, rs`, `dot rd, rs`) — apenas dois registos. Como o destino `rd` é também o primeiro operando fonte, o assembler **duplica `rd` no campo `rs1`** (`rs1 = rd`) e coloca `rs` em `rs2`. Esta duplicação evita multiplexadores de endereçamento extra à entrada do Banco de Registos.

| Sub-formato | Sintaxe | Tradução para os campos físicos |
| :--- | :--- | :--- |
| Tipo-I  | `li rd, imm`        | `imm[15:7]`, `rd[6:4]`, `opcode=0000` |
| Tipo-R2 | `add rd, rs`        | `funct3=000`, `rs2 = rs`, `rs1 = rd`, `rd`, `opcode=1000` |
| Tipo-R2 | `dot rd, rs`        | `funct3=010`, `rs2 = rs`, `rs1 = rd`, `rd`, `opcode=1000` |
| Tipo-R3 | `dota rd, rs1, rs2` | `funct3=001`, `rs2`, `rs1`, `rd`, `opcode=1000` |

---

## ⚙️ Referência de Operações (microarquitetura)

### `li rd, imm` — Load Immediate
`R[rd] ← imm`. O imediato `[15:7]` passa por um *Bit Extender* (9→16, com sinal) e é escrito no registo via *Write-Back MUX* (sinal `WBMem = 1`), contornando inteiramente a ALU.

### `add rd, rs` — Adição
`R[rd] ← R[rd] + R[rs]`. O Banco de Registos fornece `R[rd]` (porta `In_Rs1`, devido à duplicação) e `R[rs]` (porta `In_Rs2`); a ALU soma-os num *Adder* dedicado (`ALUop = 0`).

### `dot rd, rs` — Produto escalar (dimensão 2)
`R[rd] ← R[rd]·R[rs] + R[rd+1]·R[rs+1]`. O Banco de Registos entrega o par completo (`In_Rs1`, `In_Rs1_next`, `In_Rs2`, `In_Rs2_next`). A ALU calcula `In_Rs1·In_Rs2 + In_Rs1_next·In_Rs2_next` com dois multiplicadores e um somador (`ALUop = 2`).

### `dota rd, rs1, rs2` — Produto escalar acumulado
`R[rd] ← R[rd] + R[rs1]·R[rs2] + R[rs1+1]·R[rs2+1]`. Usa as **5 saídas** do Banco de Registos: a porta escalar `In_Rd` injeta o acumulador no somador final da ALU, num único ciclo (`ALUop = 1`).

---

## 🔧 Sinais de Controlo (`CONTROL_UNIT`)

| Sinal | Largura | Geração | Função |
| :--- | :---: | :--- | :--- |
| `is_Li`   | 1 | `opcode == 0x0` | instrução é `li` |
| `is_Alu`  | 1 | `opcode == 0x8` | instrução é Tipo-R |
| `ALUop`   | 3 | `= funct3` | seleciona a operação no MUX final da ALU |
| `RegWrite`| 1 | `is_Li OR is_Alu` | habilita a escrita no Banco de Registos |
| `WBMem`   | 1 | `is_Li` | seleciona a fonte do *write-back*: `1` = imediato, `0` = `ALURes` |

---

## 📝 Exemplos de Codificação em Linguagem Máquina

| Assembly | `funct3` `rs2` `rs1` `rd` `opcode` | Binário | Hex |
| :--- | :--- | :--- | :---: |
| `li R0, 2` | `imm=000000010` · `rd=000` · `0000` | `0000 0001 0000 0000` | `0x0100` |
| `add R3, R1` | `000` `001` `011` `011` `1000` | `0000 0101 1011 1000` | `0x05B8` |
| `dot R0, R2` | `010` `010` `000` `000` `1000` | `0100 1000 0000 1000` | `0x4808` |
| `dota R0, R4, R6` | `001` `110` `100` `000` `1000` | `0011 1010 0000 1000` | `0x3A08` |

*(Todos verificados contra a ROM do `proj3.circ`.)*

---

## 🧪 Programa de Demonstração (na ROM)

```asm
li R0,2 ; li R1,3 ; li R2,4 ; li R3,2
li R4,1 ; li R5,5 ; li R6,2 ; li R7,2
add  R3, R1          ; R3 = 2+3 = 5
dot  R0, R2          ; R0 = 2·4 + 3·5 = 23
dota R0, R4, R6      ; R0 = 23 + 1·2 + 5·2 = 35
```

ROM (hex): `0100 0190 0220 0130 00C0 02D0 0160 0170 05B8 4808 3A08` → **Resultado: `R0 = 35`**.

---

## 💡 Justificações de Design

1. **Redundância na codificação em vez de hardware extra.** Em `add`/`dot`, o campo `rs1` é preenchido com `rd` pelo assembler. Transferir esta redundância para o código máquina dispensa multiplexadores de endereçamento antes do Banco de Registos — menos componentes, datapath mais simples.
2. **Cálculo dos índices seguintes dentro do Banco de Registos (e não no `main`).** Os somadores de 3 bits que geram os endereços seguintes (`rs1+1` e `rs2+1`) ficam dentro do `REGISTER_FILE`. Como duplicamos `rd` no campo `rs1`, o `rd+1` do `dot` é realizado como `rs1+1`, bastando calcular `rs1+1` e `rs2+1` (não existe porta `rd+1`). Como essa aritmética depende dos endereços de leitura, pertence junto da lógica de endereçamento (encapsulamento e responsabilidade única). O `main` manipula apenas barramentos lógicos (base e *next*), ficando mais limpo e sem somadores de endereços espalhados ao nível de topo; o esquema de endereçamento fica confinado a um único bloco, mais fácil de alterar sem tocar no datapath.
3. **Single-cycle com 3 portas de leitura escalares + 2 vetoriais.** As 5 saídas simultâneas permitem alimentar `dota` (5 operandos) sem *structural hazards* nem múltiplos ciclos.
4. **ALU pura.** O caminho do imediato (`li`) é resolvido por um *Write-Back MUX* junto à escrita dos registos, não dentro da ALU, mantendo-a um bloco estritamente matemático e fácil de estender.
5. **`opcode` curto + `funct3`, com bit de ALU dedicado.** O bit `[3]` do `opcode` funciona como sinal "usa a ALU" (e não como identificador de tipo); os bits `[2:0]` permitem adicionar novas instruções não-ALU e o `funct3` (3 bits) novas operações na ALU, tudo sem alterar o datapath. Hoje, com só duas classes, o bit `[3]` coincide com separar `li` de Tipo-R.
