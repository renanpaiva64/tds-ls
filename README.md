# TDS LS 

O projeto **TDS LS** é a implementação da **TOTVS** da especificação do *Language Server Protocol* ([https://microsoft.github.io/language-server-protocol/](https://microsoft.github.io/language-server-protocol/)) que pode ser utilizada por quaisquer *IDEs* que suportam este protocolo.

Atualmente o **TDS LS** está sendo utilizada pelo **TDS VS Code** ([https://github.com/totvs/tds-vscode](https://github.com/totvs/tds-vscode)) e pelo **TDS Eclipse** ([https://github.com/totvs/tds-eclipse](https://github.com/totvs/tds-eclipse)).

## Especificação

Além das mensagens especificadas pelo protocolo *LSP* o **TDS LS** implementa mensagens adicionais (`$totvsserver`) de uso dos *AppServers* da **TOTVS**, para realizar a conexão, compilação, aplicação de *patches* dentre outras ações.

Assim que estabilizadas as mensagens adicionais serão documentadas aqui para que quaisquer desenvolvedores possam implementar sua própria *IDE* e utilizar o motor do **TDS LS**.

## Mensagens `$totvsserver`

| Mensagem                    | Descrição                                |
|-----------------------------|------------------------------------------|
| $totvsserver/authentication | Conexão e autenticação com o *AppServer* |
| $totvsserver/compilation    | Compilação de fontes no *RPO*            |
| ...                         | ...                                      |

> **TODO** - Lista completa e detalhada dos parâmetros das mensagens

# TDS LS CLi

O **CLi** (abreviação para Command Line) é a nova implementação do **tdscli**, já conhecido dos usuários do **TDS**, e que utiliza o **TDS LS** para executar as operações nos *AppServers* da **TOTVS**.

## Uso padrão

Para executar o **TDS LS CLi** é necessária a criação de um arquivo de parametrização e executar o **CLi** passando este arquivo através do parâmetro `--tdsCli`.

> `> advpls --tdsCli=<param_file_path>`

Uma das vantagens do uso do **TDS CLi** através do arquivo de parametrização é a possibilidade de execução de diversas ações sequencialmente em apenas um script de execução. Por exemplo, em um script podemos definir que o **TDS CLi** realizará a conexão com o *AppServer* em seguida desfragmentará do *RPO*, compilará uma lista de programas, gerará um *patch* com os programas compilados anteriormente e pode realizar novamente a desfragmentação do *RPO* para finalizar, tudo isso em apenas uma operação.

### Arquivo de parametrização

O arquivo pode ser criado com qualquer nome e pode ser composto de diversas seções que executarão as distintas tarefas.

> As linhas iniciadas por `#` ou `;` são consideradas como comentários e serão desconsideradas.

Para definir uma seção inicie uma nova linha com a descrição ou nome da seção entre `[]`, como por exemplo, `[nova seção]`. Existem duas seções reservadas que são tratadas de forma especial, a seção `geral`, que são todos os parâmetros definidos antes da primeira seção, e a seção `[user]`.

#### Exemplo

```ini
;Exemplo de arquivo de parametrização para o TDS CLi
;Esta é a seção 'geral', pois não existe nenhuma seção definida
showConsoleOutput = true
logToFile = output.log

;Esta é a seção [user] onde podemos definir "variáveis de ambiente"
[user]
PATCHFILE = subpasta\tttp120.ptm
OUTPUT = patchinfooutput.txt

;Esta é a primeira seção de ação, de conexão e autenticação, definida pelo usuário
[conexão com AppServer]
action = authentication
server = localhost
port = 1234
build = 7.00.170117A
environment = env
user = user
psw = pass

;Apenas desfragmenta o RPO
[desfragmenta o RPO]
action = defragRPO

;Obtem as informações do patch definida por PATCHFILE 
;e salva as informações no arquivo definido por OUTPUT
[obtem informações do patch]
action = patchInfo
localPatch = t
patchFile = ${PATCHFILE}
output = ${OUTPUT}

;Gera o patch a partir da lista de arquivos obtidos pelo patchInfo
[geração de patch]
action = patchGen
fileResourceList = ${OUTPUT}
patchType = PTM
saveLocal = T:\tdscli\patches
```

## Parâmetros gerais

Na seção "geral", no início do arquivo de parametrização, antes de criar a primeira seção, você pode definir alguns parâmetros gerais.

| Parâmetro         | Valor                                   | Descrição                                                                     |
|-------------------|-----------------------------------------|-------------------------------------------------------------------------------|
| logToFile         | Caminho relativo ou absoluto do arquivo | Define o arquivo onde serão registrados as informações da execução do CLi     |
| showConsoleOutput | True (T) ou False (F)                   | Define se a saída deverá ser exibida no console                               |

## Seção `[user]`

Na seção `[user]` definimos "variáveis de ambiente" que podem ser utilizadas em todas as seções de usuário.

Definimos da forma usual `NOME_CHAVE = VALOR`, como por exemplo, `OUTPUT = patchinfooutput.txt` e para utilizar a chave definida basta chamar por `${NOME_CHAVE}`.

## Seções de usuário

Nas seções de usuário devemos definir qual ação será executada através do parâmetro `action` e em seguida definir os parâmetros adicionais necessários para executar esta ação.

> Em todas as seções você pode adicionar o parâmetro `skip = True` para ignorar a execução da seção.

> Usualmente a primeira seção de usuário a ser definida é a `authentication`, pois todas as outras ações vão requerer uma conexão válida com o *AppServer*.

### `action = validate`

Obtém a versão de *build* do servidor requerida para efetuar uma conexão.

| Parâmetro | Valor    | Descrição                       |
|-----------|----------|---------------------------------|
| server    | IP       | Endereço do AppServer           |
| port      | numérico | Porta em que o AppServer escuta |
| secure    | 1 ou 0   | Se a conexão é segura ou não    |

Resposta:

| Código retorno | Descrição                        |
|----------------|----------------------------------|
| 0              | Informação `build: 7.00.170117A` |
| -1             | Erro na execução                 |

> Podemos utilizar a opção `build=AUTO` na chamada da ação `authentication`, porém se for informado o *build* exato, o processo será mais eficiente.

### `action = authentication`

Realiza a conexão no AppServer e autenticação do usuário/senha no ambiente informado.

| Parâmetro   | Valor             | Descrição                                     |
|-------------|-------------------|-----------------------------------------------|
| server      | IP                | Endereço do AppServer                         |
| port        | numérico          | Porta em que o AppServer escuta               |
| secure      | 1 ou 0            | Se a conexão é segura ou não                  |
| build       | *build* ou *AUTO* | Versão do AppServer ou auto detecção          |
| user        | "nome de usuário" | Usuário para autenticação                     |
| psw         | "senha"           | Senha para autenticação                       |
| environment | "ambiente"        | Ambiente na qual será efetuada a autenticação |

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = compile`

Realiza a compilação/recompilação de programas no RPO.

| Parâmetro   | Valor                                                       | Descrição                                                                         |
|-------------|-------------------------------------------------------------|-----------------------------------------------------------------------------------|
| program     | Nomes dos arquivos e/ou diretórios separados por `,` ou `;` | Programas a serem processados                                                     |
| programList | Caminho relativo ou absoluto do arquivo                     | Arquivo contendo os nomes dos arquivos a serem processados (um arquivo por linha) |
| recompile   | True (T) ou False (F)                                       | Se deve recompilar ou não o programa                                              |
| includes    | Diretórios com includes separados por `,` ou `;`            | Arquivos de includes                                                              |

> Informar a opção `program` ou `programList` mas não ambos.

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = patchGen`

Realiza a geração do patch conforme os recursos indicados.

| Parâmetro        | Valor                                                       | Descrição                                                                         |
|------------------|-------------------------------------------------------------|-----------------------------------------------------------------------------------|
| saveLocal        | Caminho relativo ou absoluto do arquivo                     | Diretório onde será gerado o patch localmente                                     |
| saveRemote       | Caminho relativo                                            | Diretório onde será gerado o patch remotamente (AppServer)                        |
| fileResource     | Nomes dos arquivos e/ou diretórios separados por `,` ou `;` | Recursos a serem processados                                                      |
| fileResourceList | Caminho relativo ou absoluto do arquivo                     | Arquivo contendo os nomes dos arquivos a serem processados (um arquivo por linha) |
| patchType        | PTM, UPD ou PAK                                             | Extensões permitidas dos patches                                                  | 

> Informar a opção `saveLocal` ou `saveRemote` mas não ambos.

> Informar a opção `fileResource` ou `fileResourceList` mas não ambos.

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = patchApply`

Efetua a aplicação do patch indicado no RPO conectado.

| Parâmetro       | Valor                                   | Descrição                                                                                   |
|-----------------|-----------------------------------------|---------------------------------------------------------------------------------------------|
| patchFile       | Caminho relativo ou absoluto do arquivo | Arquivo de patch a ser aplicado                                                             |
| localPatch      | True (T) ou False (F)                   | Se o arquivo de patch é local ou remoto (já no AppServer)                                   |
| validatePatch   | True (T) ou False (F)                   | Se deve ocorrer a validação do patch                                                        |
| applyOldProgram | True (T) ou False (F)                   | Se devem ser aplicados programas com data de compilação mais antigas que a existente no RPO | 

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = patchInfo`

Obtém as informações do patch indicado.

| Parâmetro  | Valor                                   | Descrição                                                 |
|------------|-----------------------------------------|-----------------------------------------------------------|
| patchFile  | Caminho relativo ou absoluto do arquivo | Arquivo de patch a ser analisado                          |
| localPatch | True (T) ou False (F)                   | Se o arquivo de patch é local ou remoto (já no AppServer) |
| output     | Caminho relativo ou absoluto do arquivo | Arquivo com as informações contidas no patch              |

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = deleteProg`

Remove os programas informados do RPO conectado.

| Parâmetro   | Valor                                                       | Descrição                                                                          |
|-------------|-------------------------------------------------------------|------------------------------------------------------------------------------------|
| program     | Nomes dos arquivos e/ou diretórios separados por `,` ou `;` | Programas a serem processados                                                      |
| programList | Caminho relativo ou absoluto do arquivo                     | Arquivo contendo  os nomes dos arquivos a serem processados (um arquivo por linha) |

> Informar a opção `program` ou `programList` mas não ambos.

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = defragRPO`

Realiza a desfragmentação do RPO no ambiente conectado.

> Esta ação não requer nenhum parâmetro de entrada.

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

### `action = getID`

Obtém o ID da chave de compilação que deve ser informado em sua requisição de uso.

> Esta ação não requer nenhum parâmetro de entrada.

Resposta:

| Código retorno | Descrição                  |
|----------------|----------------------------|
| 0              | Informação `ID: ABCD-1234` |
| -1             | Erro na execução           |

### `action = authorization`

Carrega e aplica a chave de compilação para compilar e/ou sobrescrever fontes padrões.

| Parâmetro     | Valor                                   | Descrição                                  |
|---------------|-----------------------------------------|--------------------------------------------|
| authorization | Caminho relativo ou absoluto do arquivo | Define o arquivo com a chave de compilação | 

Resposta:

| Código retorno | Descrição        |
|----------------|------------------|
| 0              | Sucesso          |
| -1             | Erro na execução |

> Em caso de erro na carga do arquivo, confirme se o ID da chave de compilação que você tem é o mesmo informado pela ação **getID**.