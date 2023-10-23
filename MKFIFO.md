# mkfifo

`mkfifo` é um comando no Unix e em sistemas operacionais baseados em Unix, como Linux, que é usado para criar FIFOs (First-In-First-Out), também conhecidos como "pipes nomeados". FIFOs são uma forma de comunicação interprocessual (IPC) que permite que dois ou mais processos leiam e escrevam em um "arquivo" comum de maneira organizada, permitindo a comunicação entre eles.

### Como `mkfifo` Funciona:

- **Criando um FIFO:**
  - O comando `mkfifo` seguido pelo nome do path cria um novo FIFO. Por exemplo, `mkfifo meu_fifo` criará um FIFO chamado `meu_fifo` no diretório atual.

- **Comunicando entre Processos:**
  - Um processo pode abrir o FIFO para leitura e outro processo pode abrir o FIFO para escrita. Quando isso acontece, os dados podem fluir do processo de escrita para o processo de leitura.

### Exemplo Básico de Uso:

1. **Crie um FIFO:**
   ```bash
   mkfifo meu_fifo
   ```

2. **Escrevendo para o FIFO (em um terminal):**
   ```bash
   echo "Olá, mundo!" > meu_fifo
   ```

3. **Lendo do FIFO (em outro terminal):**
   ```bash
   cat meu_fifo
   ```

### Usos Comuns de `mkfifo`:

- **Scripts e Automação:**
  - FIFOs podem ser usados em scripts para sincronizar ou comunicar entre diferentes partes de um script ou entre diferentes scripts.

- **Debugging e Logging:**
  - FIFOs podem ser usados para enviar mensagens de log ou de debugging de um processo para outro, facilitando a captura e análise de mensagens.

### Considerações de Segurança:

- **Permissões:**
  - Como qualquer arquivo, FIFOs têm permissões que devem ser consideradas para evitar o acesso não autorizado.
  
- **Limpeza:**
  - FIFOs permanecerão no sistema de arquivos até serem excluídos manualmente. A limpeza adequada é importante para evitar a confusão e possíveis problemas de segurança.
