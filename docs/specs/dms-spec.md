# Especificação - Document Management System

## 1. Objetivo

Entregar uma aplicação web que permita a usuários identificados enviar, listar e baixar seus documentos, armazenando os arquivos no filesystem local e mantendo os metadados em memória nesta primeira versão.

## 2. Escopo

### Dentro do escopo

- Envio de arquivos para armazenamento local.
- Listagem dos documentos do usuário identificado.
- Download de documentos pertencentes ao usuário identificado.
- Associação dos documentos ao identificador simples do usuário.
- Validação de presença do arquivo e de limite máximo de tamanho.
- Interface web para os fluxos de upload, listagem e download.

### Fora do escopo

- Armazenamento externo ou em nuvem.
- Persistência durável dos metadados em banco de dados.
- Autenticação, cadastro de usuários ou gerenciamento de credenciais.
- Compartilhamento de documentos entre usuários.
- Versionamento de documentos.
- Restrições por tipo MIME ou extensão.

## 3. Requisitos funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RF-01 | O usuário pode enviar um arquivo usando `multipart/form-data` no campo `file`. | Um arquivo válido dentro do limite é armazenado localmente e a API responde `201` com seus metadados. |
| RF-02 | Todo documento enviado é associado ao valor não vazio do header `X-User-Id`. | A API rejeita a requisição quando o header não é informado ou está vazio. |
| RF-03 | O usuário pode listar seus documentos. | A resposta contém somente documentos cujo campo `owner` corresponde ao `X-User-Id`; sem documentos, retorna uma lista vazia. |
| RF-04 | O usuário pode baixar um documento próprio pelo identificador. | A API retorna o conteúdo binário como anexo e usa o nome original no nome de download. |
| RF-05 | O sistema impede acesso a documento inexistente ou pertencente a outro usuário. | Ambos os casos retornam `404`, sem revelar se o documento de outro usuário existe. |
| RF-06 | O sistema rejeita upload sem arquivo ou acima do limite configurado. | A ausência do arquivo retorna `400`; o excesso de tamanho retorna `413`; nenhum arquivo parcial fica disponível pela API. |
| RF-07 | A interface permite enviar, visualizar e baixar documentos usando a API. | As operações usam o prefixo `/api` provido pelo proxy do Vite e exibem mensagens compreensíveis em caso de erro. |

## 4. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos são gravados exclusivamente no filesystem local, em `backend/storage`, usando `multer` com `diskStorage`. |
| RNF-02 | Os metadados são mantidos em memória durante a execução do processo; reiniciar o backend descarta os metadados. |
| RNF-03 | `MAX_FILE_SIZE_BYTES` é uma variável de ambiente obrigatória, deve ser um inteiro positivo e configura o tamanho máximo em bytes. O backend deve falhar na inicialização se o valor estiver ausente ou inválido. |
| RNF-04 | O backend segue a Clean Architecture simples: `routes -> controllers -> services -> repositories`. Camadas internas não dependem de Express ou de detalhes de transporte HTTP. |
| RNF-05 | A persistência local não deve expor caminho ou nome interno do arquivo nas respostas da API. O nome físico deve ser gerado de forma segura, sem confiar no nome enviado pelo cliente. |
| RNF-06 | As respostas de erro da API seguem o formato JSON definido nesta especificação, exceto respostas binárias de download bem-sucedidas. |
| RNF-07 | O backend usa JavaScript/CommonJS e os testes automatizados usam o runner nativo `node:test`. O frontend usa React, componentes funcionais e `fetch`. |
| RNF-08 | A configuração do ambiente é feita por variáveis de ambiente, sem credenciais ou configuração operacional fixa no código. |

## 5. Modelo de dados

### Metadados públicos do documento

| Campo | Tipo | Descrição |
| --- | --- | --- |
| `id` | string | Identificador único gerado pelo servidor (UUID). |
| `originalName` | string | Nome original do arquivo enviado, preservado para exibição e download. |
| `size` | number | Tamanho do arquivo em bytes. |
| `uploadedAt` | string | Data e hora do upload em ISO 8601, em UTC. |
| `owner` | string | Valor do header `X-User-Id` que enviou o documento. |

O modelo retornado pela API contém somente esses campos. A associação ao nome interno gerado para o arquivo é detalhe privado do armazenamento e não deve ser incluída na resposta pública. O campo `originalName` nunca deve ser usado como caminho físico no filesystem.

## 6. Contratos de API

### Convenções

- Os caminhos abaixo são os caminhos do backend. O frontend os consome sob `/api` por meio do proxy do Vite; por exemplo, `POST /api/upload` é encaminhado a `POST /upload`.
- `X-User-Id` é obrigatório em todos os endpoints. O valor deve ser uma string não vazia após remoção de espaços nas extremidades.
- O header identifica o usuário para associação e isolamento de dados, mas não autentica nem comprova sua identidade. Esta solução é somente adequada ao escopo inicial e não oferece proteção contra falsificação do header.
- Respostas JSON usam `Content-Type: application/json; charset=utf-8`.

### Formato de erro

Erros JSON usam a seguinte estrutura:

```json
{
  "error": {
    "code": "FILE_REQUIRED",
    "message": "Envie um arquivo no campo file."
  }
}
```

`code` é estável para tratamento pelo cliente; `message` é uma mensagem segura e legível, em português, sem detalhes internos ou caminhos do servidor.

### `POST /upload`

- Headers: `X-User-Id` obrigatório.
- Entrada: `multipart/form-data` com exatamente um arquivo no campo `file`.
- Processamento: validar usuário, presença e limite do arquivo; gravá-lo com `multer`/`diskStorage` em `backend/storage`, usando nome interno gerado pelo servidor; registrar os metadados em memória.
- Sucesso: `201 Created`, JSON com o objeto de metadados do documento criado.
- Erros:
  - `400 USER_ID_REQUIRED`: header ausente ou vazio.
  - `400 FILE_REQUIRED`: campo `file` ausente.
  - `413 FILE_TOO_LARGE`: arquivo acima de `MAX_FILE_SIZE_BYTES`.
  - `500 INTERNAL_ERROR`: falha inesperada no armazenamento ou no registro dos metadados. A operação deve remover o arquivo recém-gravado se não puder concluir o registro.

### `GET /documents`

- Headers: `X-User-Id` obrigatório.
- Entrada: nenhuma.
- Sucesso: `200 OK`, JSON contendo um array de metadados pertencentes ao usuário. Ordenar do mais recente para o mais antigo por `uploadedAt`; em empate, usar `id` para uma ordem determinística. Sem resultados, retornar `[]`.
- Erros:
  - `400 USER_ID_REQUIRED`: header ausente ou vazio.
  - `500 INTERNAL_ERROR`: falha inesperada ao consultar os metadados.

### `GET /documents/:id/download`

- Headers: `X-User-Id` obrigatório.
- Entrada: identificador do documento em `:id`.
- Sucesso: `200 OK`, conteúdo binário com `Content-Disposition: attachment` e o nome original como nome de download. Não incluir o caminho local na resposta.
- Erros:
  - `400 USER_ID_REQUIRED`: header ausente ou vazio.
  - `404 DOCUMENT_NOT_FOUND`: identificador inexistente ou documento pertencente a outro usuário.
  - `500 INTERNAL_ERROR`: falha inesperada de leitura do arquivo. O corpo da resposta usa o formato de erro JSON.

## 7. Decisões arquiteturais e limitações

- Rotas definem endpoints e delegam aos controllers; controllers validam a entrada HTTP e formatam as respostas; services concentram as regras de negócio; repositories consultam e registram metadados.
- O armazenamento binário é local, dentro de `backend/storage`, exclusivamente por `multer` com `diskStorage`. Nenhum serviço de armazenamento externo deve ser usado.
- Os metadados ficam em memória. Após reiniciar o backend, arquivos que permanecerem no disco não terão metadados disponíveis para listagem ou download. Limpeza automática de arquivos órfãos não faz parte deste escopo.
- O frontend usa React e `fetch`, organizado em componentes, páginas e serviços. O proxy `/api` do Vite encaminha as chamadas ao backend.
- A identificação via `X-User-Id` não é autenticação. Uma implantação exposta a usuários não confiáveis exigirá autenticação e autorização reais antes de ser considerada segura.
- Arquivos de qualquer tipo são aceitos, sujeitos ao limite de tamanho. A aplicação não promete que o conteúdo seja seguro para abrir ou executar.

## 8. Plano de execução

As etapas abaixo são marcos funcionais para implementação futura. Não definem alterações por arquivo e não fazem parte da execução desta especificação.

1. **Preparar configuração e contratos:** validar configuração obrigatória, definir respostas HTTP e estabelecer os limites entre as camadas.
2. **Implementar envio local:** validar usuário e arquivo, gravar com `multer`/`diskStorage` e registrar metadados em memória com tratamento de falhas.
3. **Implementar consulta e autorização por dono:** listar documentos próprios e permitir download somente quando o documento pertence ao usuário identificado.
4. **Integrar a experiência web:** conectar os fluxos de envio, listagem e download à API por meio do proxy `/api`, incluindo estados e mensagens de erro.
5. **Verificar o comportamento:** cobrir casos de sucesso, validações, isolamento entre usuários, falhas de filesystem e integração dos fluxos da interface; executar os testes automatizados e uma verificação manual do fluxo completo.

## 9. Critérios de aceite da especificação

- Todos os requisitos funcionais e não funcionais têm identificadores únicos e critérios verificáveis.
- Modelo, contratos e requisitos usam os mesmos nomes para campos, headers, variáveis de ambiente e endpoints.
- Upload e download não expõem caminhos físicos; listagem e download aplicam isolamento por `owner`.
- A especificação respeita armazenamento local, metadados em memória e a arquitetura em camadas definida pelo projeto.
- O plano permanece em nível de marcos e não enumera alterações ou execução de arquivos de backend ou frontend.