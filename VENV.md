# VENV

O `venv` (ambiente virtual) é um módulo em Python usado para criar ambientes isolados para projetos Python. Cada ambiente virtual tem sua própria instalação do interpretador Python, o que permite que você instale pacotes e dependências de forma isolada, sem afetar outros projetos ou a instalação global do Python em seu sistema.

## Por que usar `venv`?

- **Isolamento**: Permite que você isole as dependências de seu projeto, evitando conflitos entre versões de pacotes.
- **Organização**: Mantém seus projetos organizados, com cada um tendo suas próprias dependências.
- **Versão do Python**: Permite que você use diferentes versões do Python em diferentes projetos.

## Como usar o `venv`

### 1. Criando um Ambiente Virtual

```sh
python -m venv myprojectenv
```
Neste exemplo, `myprojectenv` é o nome do diretório onde o ambiente virtual será criado.

### 2. Ativando o Ambiente Virtual
Depois de criar o ambiente virtual, você precisa ativá-lo.

- **Windows:**
  ```sh
  .\myprojectenv\Scripts\activate
  ```
  
- **Linux/MacOS:**
  ```sh
  source myprojectenv/bin/activate
  ```

Ao ativar o ambiente virtual, o prompt de comando geralmente mostra o nome do ambiente virtual, indicando que o ambiente está ativo.

### 3. Instalando Pacotes

Com o ambiente virtual ativado, você pode usar o `pip` para instalar pacotes isoladamente dentro deste ambiente.
```sh
pip install nome_do_pacote
```

### 4. Desativando o Ambiente Virtual

Para sair do ambiente virtual e retornar ao Python global do seu sistema, você pode usar o comando `deactivate`.
```sh
deactivate
```

## Dicas de Uso

- Não inclua o diretório do ambiente virtual no controle de versão (por exemplo, adicione-o ao `.gitignore` se estiver usando Git). Em vez disso, use um arquivo `requirements.txt` (`pip freeze > requirements.txt`) para listar as dependências do projeto.
- Sempre ative o ambiente virtual antes de executar scripts Python ou instalar pacotes via `pip` dentro do ambiente.
- Lembre-se de que o ambiente virtual só afeta o terminal no qual foi ativado. Se você abrir um novo terminal, precisará ativar o ambiente virtual novamente naquele terminal.

O uso de ambientes virtuais é uma prática recomendada para manter seus projetos Python limpos e organizados, minimizando os problemas causados por conflitos de dependências entre diferentes projetos.

