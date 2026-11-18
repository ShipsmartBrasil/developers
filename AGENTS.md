# Sobre o projeto

Este projeto é uma documentação de guia, para auxiliar desenvolvedores externos a usarem a API da Shipsmart Global.
A documentação é escrita Português Brasileiro (pt-br) e posteriormente traduzida para os demais idiomas.

## Projetos a ser analisado

O projeto que é analisado para gerar o guia, fica na pasta de mesmo nivel deste projeto, chamado `app_api`. É nesta pasta que
fica a API Púbica da Shipsmart Global, na qual temos que tirar as informações técnicas de funcionamento da mesma, para criar/atualizar o guia de desenvolvimento.
Você também consegue acessar uma outra pasta chamada `app_back` que é a API de Backend da aplicação da Shipsmart Global.

- Lembre-se: NUNCA PUBLICAR NENHUM DETALHE DOS ENDPOINTS E FUNCIONAMENTO DA API DE BACKEND NO GUIA!
- Lembre-se 2: NUNCA MENCIONAR OU DAR PISTAS DE QUAL É A TECNOLOGIA QUE ESTÁ SENDO UTILIZADA PARA O DESENVOLVIMENTO DA API PÚBLICA OU DE BACKEND!

### Sobre a API

Sempre que formos disponibilizar o domínio públic da API Pública da Shipsmart Global no guia, temos que usar o endereço `https://api.shipsmart.global`.
O link para a documentação da API Pública fica em `https://api.shipsmart.global/doc`.
A API Pública também tem uma URL para acesso para o ambiente de `staging (homologação)` que é `https://hmlapi.shipsmart.com.br` e que contem a documentação
das features que estão em homologação em `https://hmlapi.shipsmart.com.br/doc`.

## Gerando o guia

A geração/atualização do guia, consiste em ler a API Pública e verificar os seus endpoints públicos e gerar um guia estruturado, onde ensinando como fazer para:
- Fazer Cotação: v1 e v2
- - Cotação de Documento
- - Cotação de Caixa (simples)
- - Cotação de Caixa (avançado)
- - Cotação de Pallet (por CLA)
- - Cotação de Pallet (por m³)
- - Cotação de Itens
- Fazer Remessa: v1 e v2
- - Cotações Inelegíveis para remessa
- - Remessa (cotação v1)
- - Remessa (cotação v2)
- Diferença entre v1 e v2
- A Busca de documentos da remessa
- A Consulta de rastreio da remessa 
- Fazer agendamento de coleta

Essas são as principais. Pode ser que quando for fazer a criação/atualização, tenham novos endpoints com novas ações, logo, devemos coloca-los no guia com as informações de:
- Informações dos campos do payload: tipo, tamanho mínimo & máximo e dados padrões do campos caso haja
- Exemplo de payload mínimo
- Exemplo de payload máximo

Além disso, precisamos ter um Menu para que fique facil a navegação entre os tópicos do guia pelo usuário.
P.S.: Deixar sempre claro que os exemplos são meramente ilustrativos e não são dados válidos

## Estrutura e Idiomas

A documentação deve ser criada/atualizada inicialmente em Português Brasileiro (pt-br), que é a tradução principal.
A partir dai, vamos traduzi-la para 4 outros idiomas:
- Inglês Americano (en-us)
- Espanhol (en-es)
- Francês (fr-fr)
- Italiano (it-it)

A documentação de todos os idiomas tem que ser igual estruturalmente, e a tradução tem que ser o mais fiel possivel da versão principal.

A estrutura deve seguir da seguinte forma:
- folder: en-us (arquivos da traducao em inglês)
- - README.md (menu principal do idioma)
- folder: es-es (arquivos da tradução em espanhol)
- - README.md (menu principal do idioma)
- folder: fr-fr (arquivos da tradução em frances)
- - README.md (menu principal do idioma)
- folder: it-it (arquivos da tradução em italiano)
- - README.md (menu principal do idioma)
- folder: pt-br (arquivos da tradução em português)
- - README.md (menu principal do idioma)
- README.md (menu principal)

O Menu principal tem que ficar em inglês, onde o usuário deverá selecionar a documentação de qual idioma ele quer acessar.
E dentro da pasta de cada idioma, tem um menu próprio, separando a documentação em tópicos, onde o usuário irá acessa-lo com maior facilidade.

Devido aos tópicos serem relativamente grandes, vamos coloca-los cada um em um arquivo `.md`
Os nomes dos arquivos devem ser todos em `Português Brasileiro`, para ficar facil de identificar qual foi a página traduzida da documentação principal.

# Histórico de atualizações

A pasta `history/` guarda o registro de como cada atualização do guia foi feita.
Cada arquivo é nomeado com a data/hora da atualização, no formato `YYYY-MM-DD_HHMM.md`.

**Antes de criar/atualizar o guia, leia o arquivo mais recente de `history/`.** Ele contém:
- o passo a passo do processo de geração;
- o inventário das páginas e o template de cada página;
- as convenções de escrita, o glossário de terminologia e as regras de tradução;
- os scripts de verificação automatizada (links, âncoras, paridade entre idiomas, vazamento de tecnologia);
- as armadilhas já encontradas;
- as pendências deixadas para a próxima atualização.

Ao final de cada atualização, **crie um novo arquivo em `history/`** com a data/hora daquela
atualização, descrevendo o que mudou e deixando as pendências anotadas.

- Lembre-se: OS ARQUIVOS DE `history/` SÃO PÚBLICOS. NUNCA INCLUIR NELES DETALHES DOS PROJETOS CONSULTADOS (NOMES DE PASTAS, ARQUIVOS, CLASSES, ROTAS INTERNAS OU TECNOLOGIA)!

# Outras informações

Faça/Atualize a documentação o mais detalhada e didática possivel.
A documentação será seguida por pessoas técnicas que entendem como fazer uma integração com uma API Restfull.


