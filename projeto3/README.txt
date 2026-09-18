 PROJETO 3 IAC: Processador de 16 bits com aceleração DOT
 Grupo: Diogo Monteiro (117887), Maria Carolina Vidal (118018),
        Miguel Jordão (117850)

1. FORMATO DAS INSTRUÇÕES (16 bits, single-cycle)
Todas as instruções ocupam 16 bits e dividem-se em 5 campos fixos:

   bits:   15 14 13 | 12 11 10 |  9  8  7 |  6  5  4 |  3  2  1  0
   Tipo-R:  funct3  |   rs2    |   rs1    |    rd    |   opcode
   Tipo-I:     imm (9 bits, com sinal)    |    rd    |   opcode

 - opcode (4 bits, [3:0]): o bit mais significativo (bit [3]) indica se a
   instrução usa a ALU (1 = caminho da ALU, 0 = não-ALU);
   os bits [2:0] distinguem instruções dentro do formato.
 - funct3 (3 bits, [15:13]): seleciona a operação Tipo-R; vai direto para ALUop
 - imm (9 bits, [15:7]): só no li; estendido com sinal p/ 16 bits (-256..255)

 O Tipo-R tem dois sub-formatos lógicos (sintaxe de assembly):
 - Tipo-R3 (dota rd, rs1, rs2): três registos distintos (rd, rs1, rs2)
 - Tipo-R2 (add rd, rs / dot rd, rs): dois registos. Rd assume o papel de primeiro
  operando, assim o assembler duplica rd no campo rs1 (rs1=rd) e põe rs em rs2

 Opcodes / funct3:
   li    opcode=0001            R[rd] <- imm
   add   opcode=1000 funct3=000 R[rd] <- R[rd] + R[rs1]
   dota  opcode=1000 funct3=001 R[rd] <- R[rd] + R[rs1]*R[rs2] + R[rs1+1]*R[rs2+1]
   dot   opcode=1000 funct3=010 R[rd] <- R[rd]*R[rs1] + R[rd+1]*R[rs+1]

2. EXEMPLO DE CODIFICAÇÃO POR INSTRUÇÃO
 li R0, 2             imm=000000010     rd=000  op=0001
                 -> 0000 0001 0000 0001  = 0x0101
 add R3, R1      f3=000 rs2=001 rs1=011 rd=011  op=1000
                 -> 0000 0101 1011 1000  = 0x05B8
 dot R0, R2      f3=010 rs2=010 rs1=000 rd=000  op=1000
                 -> 0100 1000 0000 1000  = 0x4808
 dota R0, R4, R6 f3=001 rs2=110 rs1=100 rd=000  op=1000
                 -> 0011 1010 0000 1000  = 0x3A08

3. PRINCIPAIS OPÇÕES E JUSTIFICAÇÃO
  - Simplificação do hardware: a redundância das instruções de dois operandos
  (add, dot) é colocada na codificação (duplicar rd em rs1) em vez de no
  hardware.
- Cálculo dos índices dentro do banco de registos: os somadores de 3 bits que
  geram os endereços seguintes (rs1+1 e rs2+1) encontram-se dentro do
  REGISTER_FILE. Como duplicamos rd em rs1, o "rd+1" do dot é realizado como
  rs1+1, pelo que basta calcular rs1+1 e rs2+1. Como dependem dos endereços de
  leitura, ficam junto da lógica de endereçamento a que pertencem. O main fica
  mais limpo e legível, sem somadores de endereços.
- Desenho da ALU: a nossa ALU é composta por somadores + 2 Multipliers, com MUX
  final por ALUop. Para manter esta clareza de desenho, o caminho do imediato da
  instrução li não passa pela ALU, mas sim pelo Write-Back MUX (sinal WBSel)
  junto à escrita dos registos.
- Estrutura do opcode: o bit [3] do opcode funciona como sinal "usa a ALU". Os
  bits [2:0] do opcode permitem adicionar novas instruções, e o funct3
  (3 bits) permite adicionar novas operações na ALU, garantindo que possamos
  expandir tudo isto sem alterar hardware.
- Unidade de Controlo: Criámos os cabos is_Li e is_Alu para ditar os sinais de
  controlo. Como as instruções add, dot e dota partilham a mesma configuração,
  o cabo is_Alu junta a ativação destes sinais. Isto simplifica o circuito e
  garante que qualquer instrução futura que use os mesmos sinais
  possa estar ligado ao cabo is_Alu.