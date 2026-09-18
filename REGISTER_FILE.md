# REGISTER_FILE - Especificação Técnica (Resumida)

Este componente é o subsistema de armazenamento de alta velocidade (*scratchpad memory*) do processador, desenhado para fornecer dados à ALU com latência zero.

## 1. O que faz?
O `REGISTER_FILE` funciona como a "bancada de trabalho" imediata do processador. A sua função principal é:
* **Armazenar Temporariamente:** Guardar os valores que estão a ser usados nos cálculos atuais (em vez de ir buscar à memória RAM principal, que é muito mais lenta).
* **Fornecer Operandos:** Entregar, de forma instantânea, os números necessários para a ALU (Unidade Lógica e Aritmética) realizar operações matemáticas ou lógicas.
* **Sincronizar a Escrita:** Garantir que, quando o processador termina uma conta, o resultado é guardado no registo correto, apenas no momento exato ditado pelo sinal de relógio (`Clk`).

## 2. Interface de Pinos
* **Entradas (Síncronas/Endereçamento):** `Clk` (relógio), `RegWrite` (habilitação), `WriteAddr` (3-bit), `WriteData` (16-bit) e 5 endereços de leitura (`ReadAddr`).
* **Saídas (Assíncronas):** 5 barramentos de 16-bit (`Out_x`) conectados diretamente à ALU.

## 3. Justificação Arquitetural: Porquê este design?
A arquitetura foi otimizada para a instrução **`dota`**, que exige o processamento paralelo de 5 operandos:
$$Resultado = R[rd] + (R[rs1] \times R[rs2]) + (R[rs1+1] \times R[rs2+1])$$

* **5 Portas de Leitura:** Essencial para alimentar a ALU com todos os operandos necessários num **único ciclo de relógio**, reduzindo drasticamente o *CPI (Cycles Per Instruction)*.
* **Acesso Combinatório:** A leitura é instantânea e independente do relógio, permitindo que os valores apareçam nos pinos de saída mal o endereço seja definido.
* **Escrita Síncrona:** A utilização de um descodificador 3:8 garante a escrita exclusiva e precisa no registo de destino apenas no flanco de subida do `Clk`.

## 4. Protocolo de Validação
1. **Escrita:** Verificar se o Decoder bloqueia escritas quando `RegWrite = 0`.
2. **Consistência:** Validar que múltiplas portas de leitura conseguem aceder ao mesmo registo sem conflitos de barramento.
3. **Sincronia:** Confirmar que os dados são atualizados apenas após a transição positiva do relógio (`Clk`).