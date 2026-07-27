# Comparativo Cotação e Remessa

## Visão geral
- Explica a diferença prática entre os fluxos de cotação e remessa nas duas versões publicadas.
- Use este guia para escolher a versão correta antes de integrar.

## Base URL
- Produção: `https://api.shipsmart.global`
- Documentação: `https://api.shipsmart.global/docs`
- Homologação: `https://hmlapi.shipsmart.com.br`

## Autenticação
- Bearer token no header `Authorization`.
- Header de idioma: `lang`.

## Regra de uso
- O fluxo padrão de integração é: cotação e remessa usando os endpoints da primeira versão.
- A segunda versão é indicada quando você não quer informar dados além do país na cotação.

## Cotação
### Diferença prática
- Na primeira versão, a cotação aceita o fluxo completo com origem, destino, caixas e itens.
- Na segunda versão, a cotação é mais enxuta para cenários em que o integrador quer informar menos dados no início do processo.

### Comparativo
| Tema | Primeira versão | Segunda versão | Impacto |
|---|---|---|---|
| Objetivo | Cotação completa | Cotação simplificada | Escolha o nível de detalhe |
| Endereço | Mais campos no contrato | Contrato mais enxuto | Menos dados na entrada |
| Itens | Fluxo completo e avançado | Fluxo explícito para itens | Melhor para payloads enxutos |
| Consulta | Consulta e status disponíveis no endpoint da primeira versão | Informação não identificada na fonte analisada | Ambas as consultas operacionais ficam na primeira versão |

## Remessa
### Diferença prática
- Na primeira versão, a remessa cobre criação, consulta de status, consulta de documentos, cancelamento, atualização e anexos.
- Na segunda versão, a fonte analisada mostra apenas a criação da remessa.

### Comparativo
| Tema | Primeira versão | Segunda versão | Impacto |
|---|---|---|---|
| Criação | Disponível | Disponível | Fluxo base comum |
| Status | Disponível | Informação não identificada na fonte analisada | Acompanhamento operacional depende da primeira versão |
| Documentos | Disponível | Informação não identificada na fonte analisada | Baixa de documentos fica na primeira versão |
| Cancelamento | Disponível | Informação não identificada na fonte analisada | Ciclo completo fica na primeira versão |
| Atualização | Disponível apenas em casos excepcionais | Informação não identificada na fonte analisada | Em regra, a remessa é irreversível após criada |

## Quando usar cada uma
1. Use a primeira versão quando quiser o fluxo operacional completo.
2. Use a segunda versão quando o payload inicial de cotação precisar ser mais simples.
3. Se o seu fluxo depende de status, documentos ou cancelamento, mantenha a primeira versão.
4. Se você criar pela segunda versão, o acompanhamento das ações é feito pelas APIs da primeira versão.

## Observações finais
- A fonte analisada não traz uma documentação equivalente completa da segunda versão para status, docs e cancelamento.
- Para operação completa, a primeira versão continua sendo a referência funcional.
