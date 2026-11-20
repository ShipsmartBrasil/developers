# Plataforma

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

> ⚠️ **Exemplos meramente ilustrativos:** todos os payloads, respostas, identificadores, códigos, documentos e tokens
> desta página são fictícios e **não são dados válidos**. Substitua-os pelos dados reais da sua conta e da sua operação.

## Visão geral
- Endpoint utilitário que devolve o nome, o ambiente e a versão da API Pública.
- É a chamada mais simples e barata da API — ideal para **validar token e conectividade** antes de
  qualquer integração, e para **health check** em monitoramento.
- Exige apenas um token válido: **não depende de nenhuma permissão** do perfil.
- É somente leitura e não tem efeito colateral.
- Este endpoint não aparece na documentação interativa (`/doc`); esta página é a referência dele.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
lang: pt
```

> Este endpoint **exige token válido**, mas não exige nenhuma permissão específica.
> Por isso ele é o melhor teste para saber se o seu token está funcionando.
> O header `lang` (`pt`, `en`, `es`) altera apenas o texto de `message`.

## Fluxo de uso

```
1. GET /v1/platform/version   → 200: token válido e API acessível
2. Conferir data.env          → ambiente esperado (produção ou homologação)
3. Seguir para a integração   → cotação, remessa etc.
```

## Endpoint: versão da plataforma

- **Método e rota:** `GET /v1/platform/version`
- **Quando usar:** ao configurar um token novo, ao trocar de ambiente e em verificações periódicas de saúde.

### Parâmetros

Nenhum parâmetro de path, query ou corpo. Apenas os headers de [Autenticação](#autenticação).

### Payload mínimo

Sem corpo:

```
GET /v1/platform/version
```

### Payload máximo

Não há parâmetros opcionais. A única variação possível é o idioma da mensagem, via header:

```
GET /v1/platform/version
lang: en
```

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Accept: application/json' \
  --header 'lang: pt'
```

### Exemplo de response

`200 OK`

```json
{
  "status": "success",
  "message": "Registro(s) listado(s) com sucesso.",
  "data": {
    "name": "Shipsmart Global API",
    "env": "production",
    "version": "1.0.0"
  }
}
```

### Campos da resposta

| Campo | Tipo | Descrição |
|---|---|---|
| `status` | string | Sempre `success` quando o token é aceito |
| `message` | string | `Registro(s) listado(s) com sucesso.` (traduzida conforme o header `lang`) |
| `data.name` | string \| null | Nome da aplicação |
| `data.env` | string \| null | Ambiente em que você está conectado (ex.: `production`, `staging`) |
| `data.version` | string \| null | Versão publicada da API |

> Confira o `env` para garantir que você não está apontando para o ambiente errado — é um erro comum
> em integrações que compartilham configuração entre ambientes.
>
> Os valores de `data` são informativos e definidos pela configuração de cada ambiente. Não os use como
> chave de lógica (ex.: comparação exata de versão) e trate `null` como valor possível.

## Casos de uso

| Caso | Como usar |
|---|---|
| Validar um token novo | Se responder `200`, o token é válido |
| Diagnosticar `404` de autenticação | Se este endpoint também responder `404`, o problema é o token, não a rota |
| Diagnosticar `403 Has no permission!` em outra rota | Se este endpoint responde `200`, o token é válido e falta apenas a permissão daquela operação |
| Confirmar o ambiente | Ler `data.env` |
| Health check de monitoramento | Chamada leve, sem efeito colateral |
| Registrar a versão nos seus logs | Útil para correlacionar comportamento com releases da API |

## Regras de negócio

| Regra | Detalhe |
|---|---|
| Autenticação | O token é validado antes de qualquer resposta. Sem header `Authorization`, a resposta é `404` |
| Permissão | Nenhuma. Qualquer token válido de usuário ativo recebe `200` |
| Conta bloqueada | Token de usuário bloqueado recebe `403` |
| Versão no path | Chamadas sem o prefixo (`/platform/version`) recebem `308` para `/v1/platform/version`, preservando a query string |
| Somente leitura | Não altera cotações, remessas nem cadastros |

## Erros comuns

| Status | Significado/Mensagem | Como corrigir |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Token ausente, incorreto ou regenerado. Conferir o header `Authorization: Bearer ...` e, se preciso, gerar um novo token no painel |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Conta bloqueada. Acionar o suporte |
| `400` | `status: "error"` com a descrição da falha na autenticação | Falha inesperada ao validar o token. Repetir com backoff; se persistir, acionar o suporte |
| `308` | Redirecionamento permanente | Chamada sem `/v1`. Usar a rota completa `/v1/platform/version` |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Este endpoint é **somente leitura** e não tem efeito algum sobre cotações, remessas ou cadastros.
- Chamadas sem o prefixo de versão (`/platform/version`) são redirecionadas com `308` para
  `/v1/platform/version`. Prefira sempre informar a versão.
- Detalhes de token, idioma e formato das respostas: [Primeiros Passos](api-primeiros-passos.md).
- Lista consolidada de mensagens de erro: [Códigos de Erro](api-codigos-de-erro.md).
