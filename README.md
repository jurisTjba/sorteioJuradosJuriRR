# Sorteio e Apuração de Jurados

Aplicação **100% front-end** (um único arquivo `.html`) para conduzir o sorteio de jurados titulares e suplentes do Tribunal do Júri, ou apurar manualmente uma lista de números sorteados. Todo o processamento acontece no navegador — nenhum dado é enviado para servidores. A única dependência externa é a biblioteca **SheetJS**, carregada via CDN, usada para ler e gerar arquivos `.xlsx`.

---

## 1. Como a planilha de entrada deve estar

A aplicação importa um arquivo **`.xlsx`** (ou `.xls`) contendo a lista completa de jurados. As regras de reconhecimento da planilha são:

### Cabeçalho obrigatório
- Deve existir, em alguma das primeiras **15 linhas** de alguma aba da pasta de trabalho, uma linha de cabeçalho contendo uma coluna chamada exatamente **"Nome"** (sem acento, maiúsculas/minúsculas e "Nome " com espaço extra também são aceitos — a comparação ignora acentuação e caixa).
- Essa é a única coluna realmente obrigatória por nome. Se nenhuma coluna "Nome" for encontrada em nenhuma aba, a importação falha com o aviso: *"Não encontrei colunas de 'Número' e 'Nome'. Verifique o cabeçalho da planilha."*
- O sistema varre todas as abas do arquivo em ordem até encontrar uma que tenha esse cabeçalho — ou seja, os dados podem estar em qualquer uma das planilhas internas do arquivo.

### Coluna de número
- O sistema procura, **antes** da coluna "Nome", uma coluna cujo nome (normalizado) seja: `numero`, `nº`, `n°`, `no`, `num` ou `ordem`.
- Se nenhuma coluna com esses nomes for encontrada, a **primeira coluna (coluna A)** é usada como número do jurado por padrão.
- Os valores dessa coluna devem ser números inteiros (ou texto contendo dígitos — outros caracteres são removidos automaticamente ao converter).

### Colunas extras (opcionais, mas reconhecidas)
- Colunas chamadas **"CPF"** ou **"Bairro"** (normalizado) são detectadas automaticamente e exibidas/exportadas como colunas extras nas tabelas de resultado.
- Qualquer outra coluna presente é preservada nos dados internos e reaparece na planilha exportada, mas **só é exibida na tela** se for "CPF" ou "Bairro". Todas as colunas do cabeçalho original, porém, são incluídas no arquivo baixado.

### Linhas de dados
- A leitura começa na linha imediatamente após o cabeçalho encontrado.
- Uma linha só é aproveitada se tiver **um número válido** (na coluna de número) **e** um **nome não vazio**. Linhas que não atendam a isso são ignoradas silenciosamente.
- Datas em células são convertidas automaticamente para o formato `dd/mm/aaaa`.
- Se dois jurados tiverem o mesmo número, apenas o primeiro é indexado para busca (mas todos aparecem na lista de registros).

### Exemplo de estrutura mínima aceita

| Número | Nome           | CPF            | Bairro     |
|--------|----------------|----------------|------------|
| 1      | Maria da Silva | 000.000.000-00 | Centro     |
| 2      | João Souza     | 111.111.111-11 | São José   |
| 3      | Ana Pereira    | 222.222.222-22 | Liberdade  |

> A coluna "Número" pode se chamar "Nº", "N°", "No", "Num" ou "Ordem"; se nenhuma dessas existir, a coluna A é usada. As colunas "CPF" e "Bairro" são opcionais.

---

## 2. Como a aplicação funciona (fluxo em 3 etapas)

### Etapa 1 — Importar a lista
- O usuário arrasta o arquivo para a área de upload ou clica para selecioná-lo.
- O arquivo é lido inteiramente no navegador (via `FileReader` + SheetJS); nada é enviado para fora.
- Após a leitura bem-sucedida, é exibido um resumo (quantidade de jurados, qual coluna foi usada como número e qual como nome) e as Etapas 2 e 3 são liberadas.
- É possível trocar o arquivo a qualquer momento pelo botão "Trocar", o que reseta todo o estado da aplicação.

### Etapa 2 — Sortear titulares e suplentes
- O usuário define: quantidade de **titulares**, quantidade de **suplentes**, mês da sessão, ano e (opcionalmente) a comarca.
- Um texto de "ata" é gerado automaticamente (editável livremente) com a redação padrão do sorteio, incluindo os números por extenso em português.
- Opções adicionais:
  - **"Não repetir jurados já sorteados nesta sessão"**: mantém um histórico de números já sorteados durante o uso atual da página e os exclui de sorteios seguintes (útil para "refazer o sorteio" sem repetir nomes). O histórico pode ser limpo manualmente.
  - **Excluir números específicos**: permite digitar números a excluir do sorteio (ex.: jurados dispensados).
- O sorteio é feito com **Fisher–Yates** usando o gerador de números aleatórios criptográfico do navegador (`crypto.getRandomValues`), com fallback para `Math.random()` caso não disponível — garantindo um sorteio justo e não previsível.
- O sistema valida se há jurados suficientes na lista (descontando exclusões/histórico) antes de sortear; caso contrário, exibe um aviso.
- Resultado: duas tabelas (Titulares e Suplentes), cada linha com ordem de sorteio, número, nome e as colunas extras (CPF/Bairro, se existirem).
- Botões disponíveis após o sorteio:
  - **Refazer sorteio** — gera um novo sorteio (respeitando a opção de não repetição).
  - **Copiar** — copia o texto da ata + as listas de titulares/suplentes para a área de transferência.
  - **Baixar planilha** — exporta um `.xlsx` com o texto da ata, a lista de titulares e a lista de suplentes, preservando todas as colunas originais da planilha importada.

### Etapa 3 — Apurar por números informados
- Alternativa para conferir um sorteio feito manualmente (por exemplo, em papel/urna física): o usuário digita os números sorteados.
- Aceita separadores por vírgula, espaço, ponto e vírgula, quebra de linha, e também **intervalos** (`10-15` ou `10 a 15`).
- Números repetidos são ignorados automaticamente (com aviso).
- Ao clicar em "Apurar", o sistema busca cada número na lista importada e mostra:
  - Quantos foram **encontrados** e quantos **não encontrados**.
  - Uma tabela com ordem informada, número e nome (e colunas extras).
  - É possível ordenar o resultado por **ordem informada** ou por **número crescente**.
  - Números que não constam na lista aparecem destacados em uma seção separada.
- Botões: **Copiar nomes** (para a área de transferência) e **Baixar planilha** (exporta `.xlsx` com o resultado apurado).

---

## 3. Observações técnicas

- **Privacidade**: todo o processamento (leitura, sorteio, apuração e geração do `.xlsx`) ocorre localmente no navegador do usuário; nenhum dado da lista de jurados é transmitido para qualquer servidor.
- **Dependência externa**: a única chamada de rede é o carregamento da biblioteca SheetJS (via CDN) e das fontes do Google Fonts — portanto é necessária conexão com a internet ao abrir a página, mesmo que os dados em si não saiam do navegador.
- **Arquivos exportados**: nomeados automaticamente com a data do dia, por exemplo `sorteio_jurados_titulares_suplentes_08-07-2026.xlsx` e `jurados_apurados_08-07-2026.xlsx`.
- **Compatibilidade**: funciona em qualquer navegador moderno com suporte a `crypto.getRandomValues`, `FileReader` e Clipboard API (Chrome, Edge, Firefox, Safari recentes).
