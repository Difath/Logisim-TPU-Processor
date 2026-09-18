# README — Projeto 3 IAC: Processador de 16 bits com aceleração DOT

**Grupo:**

- Diogo Monteiro — 117887
- Maria Carolina Vidal — 118018
- Miguel Jordão — 117850

---

## 1. Descrição geral

Este projeto implementa, no simulador Logisim, um **processador de 16 bits** com modelo de
execução **single-cycle** (uma instrução por ciclo de relógio). O processador executa programas
armazenados numa ROM e foi especializado para acelerar **produtos internos** (`dot`/`dota`), as
operações centrais do mecanismo de atenção do Mini-Transformer dos projetos anteriores.

A arquitetura organiza-se em torno de um banco de 8 registos de 16 bits com leitura híbrida
(escalar e vetorial) que entrega **5 operandos em simultâneo** à ALU, permitindo executar as
instruções vetoriais num único ciclo. O ficheiro do circuito é `proj3.circ` e está organizado
nos sub-circuitos `main`, `INSTRUCTOR_FETCH`, `REGISTER_FILE`, `ALU` e `CONTROL_UNIT`.

## 2. Formato das instruções

Todas as instruções têm **16 bits** e repartem-se por cinco campos fixos (definidos pelo
*splitter* do circuito `main`):

```
 bits:   15 14 13 | 12 11 10 |  9  8  7 |  6  5  4 |  3  2  1  0
 Tipo-R:  funct3  |   rs2    |   rs1    |    rd    |   opcode
 Tipo-I:  <------ imm (9 bits, com sinal) ------>  |    rd    |   opcode
```

Existem dois formatos físicos:

- **Tipo-I** (`li`): imediato de 9 bits com sinal em `[15:7]`, `rd` em `[6:4]`, `opcode` em `[3:0]`.
  O imediato é estendido com sinal de 9 para 16 bits (intervalo −256 a +255).
- **Tipo-R** (`add`, `dot`, `dota`): `funct3` em `[15:13]`, `rs2` em `[12:10]`, `rs1` em `[9:7]`,
  `rd` em `[6:4]`, `opcode` em `[3:0]`.

Ao nível da sintaxe de assembly, o Tipo-R subdivide-se em **dois sub-formatos lógicos**:

- **Tipo-R3** (`dota rd, rs1, rs2`): três registos distintos, usando os campos `rd`, `rs1` e
  `rs2` tal como estão.
- **Tipo-R2** (`add rd, rs`, `dot rd, rs`): apenas dois registos. Como o destino `rd` é também o
  primeiro operando fonte, o assembler **duplica `rd` no campo `rs1`** (`rs1 = rd`) e coloca `rs`
  em `rs2`. Esta duplicação evita multiplexadores de endereçamento adicionais à entrada do banco
  de registos.

**Tabela de formatos:**

| Instrução | Formato / sub-formato | Campos principais |
|-----------|-----------------------|-------------------|
| `li`   | Tipo-I  | `imm[15:7]`, `rd[6:4]`, `opcode[3:0]` |
| `add`  | Tipo-R2 | `funct3[15:13]`, `rs2[12:10]`, `rs1[9:7]=rd`, `rd[6:4]`, `opcode[3:0]` |
| `dot`  | Tipo-R2 | `funct3[15:13]`, `rs2[12:10]`, `rs1[9:7]=rd`, `rd[6:4]`, `opcode[3:0]` |
| `dota` | Tipo-R3 | `funct3[15:13]`, `rs2[12:10]`, `rs1[9:7]`, `rd[6:4]`, `opcode[3:0]` |

## 3. Codificação das instruções

O `opcode` tem 4 bits, mas o seu **bit mais significativo (`[3]`) indica se a instrução usa a
ALU** (`1` = caminho da ALU, `0` = não-ALU) — não o tipo de instrução em si. Os bits `[2:0]`
identificam a instrução dentro da classe. Na ISA atual, com apenas duas classes, isto coincide com
separar `li` (`0000`, não-ALU) das operações Tipo-R (`1000`, ALU), mas a semântica é
deliberadamente mais geral para permitir extensões futuras. Dentro do Tipo-R, o campo `funct3`
seleciona a operação e é encaminhado diretamente para o sinal `ALUop`.

**Tabela de opcodes:**

| Instrução | Opcode | Funct3 | Descrição |
|-----------|--------|--------|-----------|
| `li`   | `0000` | —     | Carrega imediato (com sinal) em `rd` |
| `add`  | `1000` | `000` | `R[rd] ← R[rd] + R[rs]` |
| `dota` | `1000` | `001` | `R[rd] ← R[rd] + R[rs1]·R[rs2] + R[rs1+1]·R[rs2+1]` |
| `dot`  | `1000` | `010` | `R[rd] ← R[rd]·R[rs] + R[rd+1]·R[rs+1]` |

**Exemplos de codificação em linguagem máquina:**

| Assembly | Binário | Hexadecimal |
|----------|---------|-------------|
| `li R0, 2`        | `0000 0001 0000 0000` | `0x0100` |
| `add R3, R1`      | `0000 0101 1011 1000` | `0x05B8` |
| `dot R0, R2`      | `0100 1000 0000 1000` | `0x4808` |
| `dota R0, R4, R6` | `0011 1010 0000 1000` | `0x3A08` |

## 4. Datapath

O datapath é composto pelos seguintes blocos principais:

- **PC + INSTRUCTOR_FETCH** — gera o endereço da próxima instrução (incremento por ciclo,
  com `RESET`) e indexa a ROM.
- **ROM de instruções** — 256 × 16 bits; entrega a instrução de 16 bits.
- **Splitter de campos** — divide a instrução em `opcode`, `rd`, `rs1`, `rs2`, `funct3` e,
  em paralelo, extrai o imediato de 9 bits (`[15:7]`) para o *Bit Extender*.
- **REGISTER_FILE** — 8 × 16 bits, com 5 portas de leitura combinacionais
  (`In_Rd`, `In_Rs1`, `In_Rs2`, `In_Rs1_next`, `In_Rs2_next`) e escrita síncrona.
- **ALU** — *Adder* (add), dois multiplicadores + somadores (dot/dota) e um multiplexer final
  comandado por `ALUop`.
- **Write-Back MUX** — escolhe a fonte do valor escrito no registo: `ALURes` (operações ALU)
  ou o imediato estendido (`li`), conforme o sinal `WBMem`.

**Fluxo dos dados:** o PC indexa a ROM → a instrução é dividida em campos → os endereços de
registo (`rd`, `rs1`, `rs2`) leem o banco de registos, que fornece os operandos base e os
"seguintes" (`+1`) à ALU → a ALU calcula o resultado → o Write-Back MUX seleciona entre o
resultado da ALU e o imediato → o valor é escrito em `R[rd]` no flanco de subida do relógio,
se `RegWrite = 1`.

## 5. Unidade de controlo

A `CONTROL_UNIT` recebe `opcode` (4 bits) e `funct3` (3 bits) e gera os sinais de controlo. A
deteção da classe é feita por dois comparadores (`opcode == 0x0` e `opcode == 0x8`).

| Sinal | Função |
|-------|--------|
| `is_Li`    | Verdadeiro se `opcode == 0x0` (instrução `li`) |
| `is_Alu`   | Verdadeiro se `opcode == 0x8` (instrução Tipo-R) |
| `ALUop` (3 bits) | Igual a `funct3`; seleciona a operação no multiplexer final da ALU |
| `RegWrite` | `is_Li OR is_Alu`; habilita a escrita no banco de registos |
| `WBMem`    | `is_Li`; seleciona a fonte do *write-back* (`1` = imediato, `0` = `ALURes`) |

## 6. Principais opções de implementação

- **Redundância na codificação em vez de hardware extra:** ao duplicar `rd` no campo `rs1` para
  `add`/`dot`, eliminam-se multiplexadores de endereçamento à entrada do banco de registos
  (menos componentes, datapath mais claro).
- **Cálculo dos índices seguintes dentro do banco de registos (e não no `main`):** os somadores de
  3 bits que geram os endereços seguintes (`rs1+1` e `rs2+1`) ficam dentro do `REGISTER_FILE`.
  Como duplicamos `rd` no campo `rs1`, o `rd+1` do `dot` é realizado fisicamente como `rs1+1`,
  pelo que basta calcular `rs1+1` e `rs2+1` — não existe porta `rd+1`. Como essa aritmética depende
  diretamente dos endereços de leitura, faz sentido vivê-la junto da lógica de endereçamento a que
  pertence (encapsulamento e responsabilidade única). Assim o `main` apenas manipula barramentos
  lógicos (base e *next*), ficando mais limpo e legível, sem fios nem somadores de endereços
  espalhados ao nível de topo; o banco de registos passa a ser o único bloco que conhece o esquema
  de endereçamento, o que facilita alterá-lo no futuro sem tocar no datapath principal.
- **Single-cycle garantido por 5 portas de leitura:** a instrução `dota` precisa de 5 operandos
  em simultâneo; as 5 saídas combinacionais evitam *structural hazards* e múltiplos ciclos.
- **ALU estritamente matemática:** o caminho do imediato é resolvido fora da ALU, por um
  Write-Back MUX, simplificando o desenho e facilitando extensões futuras.
- **Bit de ALU dedicado no `opcode`:** o bit `[3]` do `opcode` funciona como sinal "usa a ALU",
  e não como identificador de tipo. Os bits `[2:0]` permitem acrescentar novas instruções não-ALU
  e o `funct3` (3 bits) novas operações na ALU, tudo sem alterar o datapath. Com apenas duas
  classes na ISA atual, o bit `[3]` coincide com separar `li` de Tipo-R.

## 7. Testes realizados

| Teste | Objetivo | Resultado esperado |
|-------|----------|--------------------|
| `li` em todos os registos | Validar carga de imediato e escrita seletiva | `R0..R7` ficam com os valores carregados |
| `add R3, R1` | Validar somador da ALU e duplicação `rs1=rd` | `R3 = 2 + 3 = 5` |
| `dot R0, R2` | Validar produto interno de dimensão 2 | `R0 = 2·4 + 3·5 = 23` |
| `dota R0, R4, R6` | Validar produto interno acumulado (5 operandos) | `R0 = 23 + 1·2 + 5·2 = 35` |
| Programa completo (ROM) | Validar pipeline single-cycle de ponta a ponta | `R0 = 35` |

Programa de demonstração (já carregado na ROM):

```asm
li R0,2 ; li R1,3 ; li R2,4 ; li R3,2
li R4,1 ; li R5,5 ; li R6,2 ; li R7,2
add  R3, R1
dot  R0, R2
dota R0, R4, R6
```

## 8. Observações finais

- A aritmética é em complemento para dois; o *overflow* é descartado (guardam-se apenas os 16
  bits menos significativos), conforme o enunciado.
- O imediato do `li` tem 9 bits com sinal (intervalo −256 a +255), maior do que o mínimo
  necessário para o programa de exemplo, deixando margem para imediatos maiores.
- As instruções `dot` e `dota` interpretam pares consecutivos de registos como vetores de
  dimensão 2 (`R[x]` e `R[x+1]`); o programador deve garantir que os pares estão devidamente
  preenchidos antes da operação.
