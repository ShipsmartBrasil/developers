# Plataforma

- [Voltar ao README do idioma](README.md)
- Pré-requisito: [Primeiros Passos](api-primeiros-passos.md)

## Visão geral
- Endpoint utilitário que devolve o nome, o ambiente e a versão da API Pública.
- É a chamada mais simples e barata da API — ideal para **validar token e conectividade** antes de
  qualquer integração, e para **health check** em monitoramento.

## Base URL
- Produção: `https://api.shipsmart.global`
- Homologação: `https://hmlapi.shipsmart.com.br`
- Documentação interativa: `https://api.shipsmart.global/doc` e `https://hmlapi.shipsmart.com.br/doc`

## Autenticação
```
Authorization: Bearer SEU_TOKEN
```

> Este endpoint **exige token válido**, mas não exige nenhuma permissão específica.
> Por isso ele é o melhor teste para saber se o seu token está funcionando.

## Endpoint: versão da plataforma

- **Método e rota:** `GET /v1/platform/version`
- **Parâmetros:** nenhum.

### Exemplo de request

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer SEU_TOKEN' \
  --header 'Accept: application/json'
```

### Exemplo de response

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
| `data.name` | string | Nome da aplicação |
| `data.env` | string | Ambiente em que você está conectado (ex.: `production`, `staging`) |
| `data.version` | string | Versão publicada da API |

> Confira o `env` para garantir que você não está apontando para o ambiente errado — é um erro comum
> em integrações que compartilham configuração entre ambientes.

## Casos de uso

| Caso | Como usar |
|---|---|
| Validar um token novo | Se responder `200`, o token é válido |
| Diagnosticar `404` de autenticação | Se este endpoint também falhar, o problema é o token, não a rota |
| Confirmar o ambiente | Ler `data.env` |
| Health check de monitoramento | Chamada leve, sem efeito colateral |
| Registrar a versão nos seus logs | Útil para correlacionar comportamento com releases da API |

## Erros comuns

| Status | Significado | Como corrigir |
|---|---|---|
| `404` | Token não encontrado ou inválido | Conferir o header `Authorization: Bearer ...` e gerar um novo token no painel |
| `403` | Conta bloqueada | Acionar o suporte |
| `500` | Erro inesperado | Repetir com backoff; se persistir, acionar o suporte |

## Observações finais
- Este endpoint é **somente leitura** e não tem efeito algum sobre cotações, remessas ou cadastros.
- Chamadas sem o prefixo de versão (`/platform/version`) são redirecionadas com `308` para
  `/v1/platform/version`. Prefira sempre informar a versão.
