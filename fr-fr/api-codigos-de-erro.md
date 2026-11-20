# Codes d'Erreur

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Regroupe, en un seul endroit, les messages d'erreur et d'avertissement renvoyés par l'API Publique, classés par module.
- Pour chaque message : le code HTTP, le texte littéral, la cause et la manière de corriger.
- Les messages sont dans la langue par défaut de l'API (`pt`). Avec `lang: en` ou `lang: es`, le texte est traduit,
  mais le code HTTP et le `status` restent les mêmes.
- Les placeholders comme `:name`, `:sku` et `:address` sont remplacés par la valeur réelle dans la réponse.
- Le détail de chaque règle se trouve dans la page du module, liée dans chaque section.

## URL de base
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
Content-Type: application/json
lang: pt
```

## Flux d'utilisation

Comment traiter une réponse qui n'est pas `200` :

```
HTTP 422 ──► lisez data (champ → messages)  ──► corrigez le payload  ──► renvoyez
HTTP 4xx ──► lisez message                  ──► corrigez le scénario ──► renvoyez
HTTP 500 ──► attendez (backoff exponentiel) ──► réessayez            ──► persiste ? contactez le support
```

1. Vérifiez d'abord le **code HTTP** : il définit la famille du problème.
2. En `422`, la `message` de premier niveau est générique — les vraies erreurs se trouvent dans `data`.
3. En `400`, `401`, `403`, `404` et `406`, la `message` décrit la règle métier qui a bloqué l'opération.
4. Ne réessayez automatiquement que les erreurs `500`. Les erreurs `4xx` se répètent tant que le payload n'est pas corrigé.

## Concepts essentiels

### Enveloppe d'erreur

```json
{
  "status": "warning",
  "message": "Cotação já foi utilizada em outra remessa!"
}
```

| Champ | Description |
|---|---|
| `status` | `warning` pour une règle métier/validation ; `error` pour une défaillance inattendue ou un refus du transporteur |
| `message` | Texte de l'erreur, traduit selon le header `lang` |
| `data` | Présent en `422` (dictionnaire de champs) et dans certaines erreurs du transporteur (retour brut) |

### Erreur de validation (`422`)

```json
{
  "status": "warning",
  "message": "Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!",
  "data": {
    "items.0.hscode": ["O campo items.0.hscode deve conter no mínimo 6 caracteres."],
    "address_sender.city": ["O campo address sender.city é obrigatório quando address sender.address code não está presente."]
  }
}
```

> La `message` du `422` est toujours le même message générique. **Utilisez `data`** pour afficher l'erreur à l'utilisateur.

### Messages de validation les plus fréquents dans `data`

| Message (par défaut) | Signification |
|---|---|
| `O campo :attribute é obrigatório.` | Champ obligatoire absent |
| `O campo :attribute é obrigatório quando :other não está presente.` | Obligation conditionnelle (ex. : adresse manuelle sans `address_code`) |
| `O campo :attribute não contém um valor válido.` | Valeur hors de la liste acceptée (enum) |
| `O valor selecionado para o campo :attribute é inválido.` | Code d'enregistrement inexistant (`channel_code`, `box_code`, `address_code`, `quotation`...) |
| `O campo :attribute deve conter no mínimo :min caracteres.` / `não pode conter mais de :max caracteres.` | Longueur de texte hors limite |
| `O campo :attribute deve conter um número inteiro.` / `deve conter um valor numérico.` | Type numérique incorrect |
| `O campo :attribute não contém um endereço de email válido.` | E-mail invalide |
| `O centro de custo selecionado não pertence ao seu usuário ou aos seus usuários autorizados.` | `cost_center_code` non autorisé |

## Erreurs par statut HTTP

| Statut | `status` typique | Famille | Que faire |
|---|---|---|---|
| `308` | - | Appel sans préfixe de version | Utiliser `/v1/...` ou `/v2/...` |
| `400` | `warning` / `error` | Règle métier | Lire `message` et corriger le scénario |
| `401` | `warning` | Type de paiement non activé | Activer l'option de paiement sur le compte |
| `403` | `warning` | Sans permission, compte bloqué, cotation non éligible | Demander la permission / refaire la cotation |
| `404` | `warning` / `error` | Ressource introuvable | Vérifier les identifiants et les enregistrements |
| `406` | `warning` | Conflit métier | Lire `message` et ajuster le flux |
| `422` | `warning` | Validation des champs (ou règle portant sur les données) | Corriger les champs listés dans `data` |
| `500` | `error` | Défaillance inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Erreurs d'authentification et de permission

Valables pour **tous** les endpoints.

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Header `Authorization` absent ou token invalide | Envoyer `Authorization: Bearer VOTRE_TOKEN` avec un token valide |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Compte bloqué | Contacter le support |
| `403` | `Has no permission!` | Le profil ne dispose pas de la permission de l'action (coter, expédier, consulter des documents, etc.) | Demander la permission au support |
| `400` | Message variable (`status: "error"`) | Défaillance inattendue lors de la validation du token | Réessayer ; si cela persiste, contacter le support |

Détails dans [Premiers Pas](api-primeiros-passos.md#authentification).

## Erreurs de cotation

`POST /v1/quotation`, `POST /v2/quotation` et `GET /v1/quotation/{quotation}/{level}`.
Détails dans [Cotation (v1)](api-cotacao-v1.md#règles-métier).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `400` | `Nenhum item foi associado para a caixa :name !` | Boîte sans article en `advanced` ou `items` (v2) | Associer au moins un article (`box_id`) à chaque boîte |
| `400` | `O box_id :box_id informado no item não existe. Verifique as posições das caixas enviadas.` | `box_id` hors de l'intervalle des boîtes | Utiliser la position (1, 2, 3...) de la boîte dans `boxes[]` |
| `400` | `Item sem sku e sem box_id vinculado. Informe o sku ou o box_id do item!` | Article manuel sans `box_id` | Renseigner `box_id` ou `sku` |
| `400` | `A caixa :name está com o peso maior que o permitido!` | Poids supérieur au maximum de la boîte enregistrée | Réduire le poids ou changer de boîte |
| `400` | `Os itens na caixa :name excedem o peso limite permitido!` | Somme des articles supérieure au maximum de la boîte enregistrée | Redistribuer les articles |
| `400` | `A caixa :name não foi encontrada para o usuário especificado!` | `box_code`/`box_identifier` inexistant ou d'un autre compte | Vérifier l'[enregistrement des boîtes](api-caixa-v1.md) |
| `400` | `Nenhum item foi encontrado com o SKU :sku especificado !` | `sku` non enregistré et article incomplet | Enregistrer l'article ou envoyer les données manuelles |
| `400` | `Os valores unitários do item (unit_value e selling_unit_value, conforme o caso) não podem ser zero. Ajuste o cadastro do item ou informe valores válidos na cotação (sku: :sku).` | `unit_value` égal ou inférieur à zéro | Renseigner une valeur positive |
| `400` | `O kit :sku é inválido!` | Kit inexistant/inactif | Vérifier le kit dans le tableau de bord |
| `400` | `A caixa do kit :sku é inválida!` | Boîte du kit inexistante | Vérifier le kit dans le tableau de bord |
| `400` | `Um ou mais itens do kit :sku são inválidos!` | Article du kit supprimé | Vérifier le kit dans le tableau de bord |
| `400` | `Para usar SmartBox, informe ao menos uma caixa em boxes[].` | `type: items` (v2) avec regroupement `smartbox` sans `boxes[]` | Envoyer les boîtes candidates |
| `400` | `Nenhuma caixa comporta o item :name (Smart Box).` | Aucune boîte envoyée ne peut contenir l'article | Envoyer une boîte plus grande |
| `400` | `Endereço de Remetente e Destinatário são iguais!` | Origine et destination identiques | Vérifier les adresses |
| `400` | `Nenhum endereco encontrado para o usuário especificado!` | `address_code` inexistant ou d'un autre compte | Vérifier le code de l'adresse |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP brésilien invalide (v1) | Corriger le CEP ou envoyer `state_code` |
| `400` | `Condição de remessa incompativel com os transportadores autorizados!` | Aucun transporteur n'a retourné de résultat | Revoir le scénario ou contacter le service commercial |
| `400` | `Não foi encontrada a cotação especificada!` | `GET` d'une cotation inexistante ou d'un autre compte | Vérifier le `quotation` |
| `400` | `Nivel de detalhe da cotação não existe` | `level` invalide dans le `GET` | Utiliser `simple` ou `detailed` |
| `403` | `O canal informado não pertence ao seu usuário` | `channel_code` d'un autre compte | Utiliser un canal de votre compte |
| `404` | `Não foi passado um postal code para o endereço do Brasil no :name !` | Adresse brésilienne sans CEP (v1) | Renseigner `postal_code` |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Adresse internationale introuvable (v1) | Vérifier ville/code postal/état |
| `404` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | État/province non reconnu | Envoyer un `state_code` valide |
| `404` | `O item com o sku :name não existe!` | `type: items` (v2) avec un SKU non enregistré | Enregistrer l'article |
| `404` | `A caixa com o box_code/box_identifier :name não existe ou está desativada` | Regroupement `default` : boîte liée à l'article inexistante | Vérifier le lien article → boîte |
| `404` | `Nenhum transportador encontrado para este usuário!` | Compte sans contrat pour la route/l'objet | Contacter le service commercial |
| `404` | `Não há contratos que atendem as especificações do pacote enviado. Se você possui duvidas sobre restrições de envios (como dimensões e pesos), por favor contate o suporte para atendimento!` | Aucun contrat ne couvre le poids/la dimension/la route | Revoir la charge ou contacter le support |
| `406` | `Sua cotação envolve peso ou dimensões que requerem atendimento personalizado. Fale com nossa equipe pelo chat para recebermos seu pedido especial.` | Charge hors de la plage automatique | Contacter l'équipe commerciale |
| `406` | `Codigo de endereço do remetente não encontrado!` | Le type de compte exige un expéditeur enregistré | Envoyer `address_sender.address_code` |
| `422` | `Não existe tabela de duty para o país de destino :name com o hscode :hscode (item :item). Revise o HS code cadastrado para o item.` | HS Code sans table de taxes dans le pays de destination | Vérifier le `hscode` |
| `400` | `Pallet na posição :n deve ter CLA (altura, largura, profundidade) OU Volume m³` | Palette sans mesures et sans `volume` | Envoyer les mesures ou `volume` |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Défaillance inattendue | Réessayer avec backoff |

> Alertes (non bloquantes) : `Itens com descrições iguais podem causar problemas na fiscalização aduaneira` et
> `Valores declarados dos itens podem ocasionar problemas aduaneiros`, dans `data.warnings`.

## Erreurs d'expédition

`POST /v1/shipping` et `POST /v2/shipping`. Détails dans [Expédition (v1)](api-remessa-v1.md#cotations-non-éligibles-à-lexpédition)
et [Expédition (v2)](api-remessa-v2.md#règles-métier).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `422` | `Não é possivel finalizar, pois seu quotation pode estar inválido ou esta cotação pode ter sido gerada em uma versão diferente da API.` | `quotation` inexistant ou d'une autre version (dans `data.quotation`) | Utiliser l'expédition de la même version que la cotation |
| `404` | `Nenhuma cotação foi encontrada com os dados especificados!` | Cotation d'un autre compte | Utiliser le token du compte propriétaire |
| `422` | `Esta conta permite apenas envios DDP (impostos pagos pelo remetente). Faça uma nova cotação DDP antes de continuar com o pagamento ou a emissão.` | Compte verrouillé en DDP avec une cotation DDU non payée | Refaire la cotation avec `tax: sender` |
| `400` | `:address: o documento federal (federal_tax_id) é obrigatório.` | Document fédéral absent | Renseigner `federal_tax_id` |
| `400` | `:address: a inscrição estadual (state_tax_id) é obrigatória.` | Brésilien sans inscription d'État | Renseigner `state_tax_id` |
| `400` | `:address: o CNPJ informado em federal_tax_id é inválido.` | CNPJ invalide | Corriger le CNPJ |
| `400` | `:address: o CPF informado em federal_tax_id é inválido.` | CPF invalide | Corriger le CPF |
| `400` | `:address: inscrição estadual inválida para pessoa física (não pode ser ISENTO/ISENTA).` | Personne physique avec IE exonérée | Renseigner une IE valide |
| `406` | `Cotação já foi utilizada em outra remessa!` | Cotation réutilisée | Faire une nouvelle cotation |
| `403` | `Cotação Simples é inelegível para remessa!` | `simple` de `package`/`pallet` | Coter avec `advanced` (ou `items`, en v2) |
| `403` | `Esta cotação está expirada. Porfavor, faça outra cotação e realize a remessa dentro do prazo!` | `valid_until` dépassé | Faire une nouvelle cotation |
| `404` | `Contrato especificado não foi listado para esta cotação!` | `carrier_code`/`slug` absent de `carriers[]` | Utiliser un contrat de la cotation |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` d'un contrat inactif | Utiliser un autre contrat |
| `401` | `Opção de pagamento não autorizado!` | Type de paiement non activé | L'activer sur le compte ou utiliser un autre type |
| `400` | `Numero da nota fiscal é requerido` | Le contrat exige `nfkey` | Renseigner la clé de la NF-e |
| `404` | `O endereço especificado, não foi encontrado!` | `address_code` invalide | Vérifier le code |
| `406` | `O país :country é diferente do enviado na cotação!` | `address_code` incompatible | Utiliser l'adresse cotée |
| `406` | `O Estado :state é diferente do enviado na cotação!` | `address_code` incompatible | Utiliser l'adresse cotée |
| `406` | `A cidade :city é diferente do enviado na cotação!` | `address_code` incompatible (en v2, cotation faite uniquement avec le pays) | Utiliser l'adresse cotée / une adresse manuelle |
| `400` | `CEP especificado não encontrato, inválido ou não compativel com o estado enviado na cotação!` | CEP invalide (v2) | Corriger le CEP |
| `404` | `Endereço internacional enviado está errado ou não existe!` | Adresse internationale introuvable (v2) | Vérifier l'adresse |
| `500` | `Area Administrativa :name não foi encontrada na nossa base de dados!` | État/province non reconnu (v2) | Envoyer un `state_code` valide |
| `500` | `Oopss.. Algo inesperado aconteceu. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Défaillance inattendue | Réessayer avec backoff |

## Erreurs de mise à jour, de pièces jointes et d'annulation

`PUT /v1/shipping/{shipping}`, `DELETE /v1/shipping/{shipping}/docs` et `DELETE /v1/shipping/...`.
Détails dans [Expédition (v1)](api-remessa-v1.md#endpoint--mettre-à-jour-une-expédition).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `400` | `Nenhum campo informado para atualização.` | `PUT` sans `nfkey`, ni `attachments`, ni `cost_center_code` | Envoyer au moins un champ |
| `400` | `Status da remessa não permite atualização na NFE.` | Expédition au-delà de `shipped` | Impossible de lier la NF-e |
| `400` | `Erro ao salvar arquivo!` | `base64` invalide | Réencoder le fichier |
| `400` | `O arquivo enviado excede o limite máximo de 5MB.` | Pièce jointe supérieure à 5 Mo | Réduire le fichier |
| `404` | `Esta remessa não foi encontrada!` | `{shipping}` invalide ou d'un autre compte (`PUT`) ; AWB/NF-e introuvable | Vérifier l'identifiant |
| `422` | `Esta remessa não aceita novos anexos pois está em um status que não permite mais alterações.` | Expédition annulée/clôturée | Impossible de joindre des fichiers |
| `422` | `O tipo do arquivo enviado não é suportado. Formatos aceitos: PDF, XML, JPEG e PNG.` | Format non accepté | Convertir le fichier |
| `422` | `O arquivo XML enviado é inválido ou está corrompido.` | XML mal formé | Corriger le XML |
| `500` | `Erro ao atualizar a remessa. Tente novamente mais tarde.` | Défaillance inattendue lors du `PUT` | Réessayer avec backoff |
| `400` | `Remessa não encontrada!` | Suppression de pièce jointe avec un UUID sans expédition | Vérifier l'UUID |
| `400` | `Esta remessa não foi encontrada!` | Annulation avec un identifiant invalide ou d'un autre compte | Vérifier l'identifiant |
| `400` | `Remessa não atende requisitos para cancelamento :id !` | Le statut actuel ne permet pas l'annulation | Voir le tableau d'annulation dans [Expédition (v1)](api-remessa-v1.md#endpoint--annuler-une-expédition) |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Défaillance inattendue lors de l'annulation | Réessayer avec backoff |

> Lors de la suppression de pièces jointes, les échecs individuels arrivent dans `data.failed[]` avec `reason` (ex. : `Nenhum documento encontrado!`),
> même avec un HTTP `200`.

## Erreurs de suivi et de documents

`GET /v1/shipping/.../status` et `GET /v1/shipping/.../docs`.
Détails dans [Suivi de l'Expédition](api-rastreio.md#erreurs-courantes) et [Documents de l'Expédition](api-documentos.md#erreurs-courantes).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `400` | `remessa.status.status_not_found: ` | Statut : aucune expédition pour l'UUID/AWB indiqué (texte non traduit) | Utiliser l'UUID de la cotation ou l'AWB correct |
| `400` | `Nenhuma remessa encontrada com o código especificado!` | Statut : expédition d'un autre compte ou canal sans expédition | Vérifier l'identifiant/le canal |
| `404` | `Nenhuma entrada de pagamento foi registrada para esta remessa!` | Expédition encore sans enregistrement de paiement | Patienter ; si cela persiste, contacter le support |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar status da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Défaillance inattendue sur le statut | Réessayer avec backoff |
| `400` | `Remessa não encontrada!` | Documents : UUID sans expédition | Vérifier l'UUID |
| `404` | `Esta remessa não foi encontrada!` | Documents : AWB/NF-e introuvable | Vérifier ou utiliser l'UUID |
| `400` | `Nenhum documento encontrado com o código especificado!` | Documents pas encore générés, filtre sans résultat ou canal sans expédition | Attendre `shipped` / revoir `documents` |
| `500` | `Oopss.. Algo inesperado aconteceu ao buscar documentos da remessa. Tente novamente mais tarde. Caso o erro persista, acione a equipe técnica!` | Défaillance inattendue sur les documents | Réessayer avec backoff |

## Erreurs de planification d'enlèvement

`POST /v1/schedule`, `GET /v1/schedule/{shipping}` et `DELETE /v1/schedule/{shipping}/{reason}`.
Détails dans [Planification d'Enlèvement](api-agendamento.md#erreurs-courantes).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `400` | `A data de agendamento não pode ser menor que a data Atual` | Date du jour ou passée | Utiliser une date à partir de demain |
| `400` | `Coleta não pode ser realizada em um final de semana` | Samedi ou dimanche | Choisir un jour ouvré |
| `400` | `Coleta não pode ser realizada durante um feriado nacional` | Jour férié national | Choisir un autre jour ouvré |
| `400` | `A data de agendamento não pode ultrapassar o limite de 7 dias úteis` | Date trop éloignée | Réduire l'anticipation |
| `400` | `Não é possível realizar um agendamento para essa remessa pois a mesma já tem um agendamento em progresso` | Une planification existe déjà pour l'expédition | Consulter avec `GET` |
| `400` | `Remessa não está em pronta para realizar coleta ou não existe` | Expédition hors de `shipped` | Attendre `shipped` |
| `400` | `Esta transportadora não possui serviço de coleta integrado` | Transporteur sans enlèvement intégré | Déposer le colis dans un point de dépôt |
| `400` | `Informações de endereço de coleta insuficientes ou inválidas` | Adresse d'enlèvement incomplète | Compléter l'adresse |
| `400` | `O atributo federal_tax_id não é válido` | CPF/CNPJ invalide | Corriger le document |
| `400` | `Endereço enviado não pertence ao pais da remessa` | État d'un autre pays | Utiliser une adresse du pays de l'expéditeur |
| `400` | `Agendamento não foi encontrado para esta remessa` | AWB introuvable (`POST`/`DELETE`) | Vérifier l'AWB ou utiliser l'UUID |
| `404` | `Agendamento não foi encontrado para esta remessa` | `GET` sans planification | Vérifier l'identifiant |
| `400` | `Coleta da Remessa não está em pronta para realizar cancelamento ou o numero enviado não existe` | Annulation hors du statut autorisé | N'annuler que les planifications confirmées |
| `422` | `O campo reason não contém um valor válido.` (dans `data`) | `reason` hors de `001` à `008` | Utiliser un motif valide |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Défaillance inattendue (y compris une clé absente dans `address`) | Envoyer toutes les clés de `address` ; réessayer |
| `500` | `Lamento mas ocorreram erros durante o processo de agendamento` | Défaillance inattendue lors de l'annulation | Réessayer ; contacter le support |

## Erreurs de manifeste

`POST /v1/manifest`. Détails dans [Manifeste](api-manifesto.md#erreurs-courantes).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `400` | `Tipo de pagamento não autorizado!` | Compte sans paiement facturé | Activer la facturation |
| `400` | `O canal especificado não foi encontrado!` | `channel_code` d'un autre compte | Utiliser un canal du compte |
| `400` | `O contrato referenciado pela slug informada não está mais válido!` | `slug` inactif | Utiliser un autre contrat |
| `400` | `Não foi possível determinar o estado ou província do endereço. Informe o campo state_code do remetente e do destinatário.` | État non résolu | Envoyer `state_code` |
| `404` | `Codigo de endereço do remetente não encontrado!` / `Codigo de endereço do destinatário não encontrado!` | `address_code` non visible pour le compte | Vérifier le code |
| `404` | `A transportadora informada não está disponível para esta remessa!` | `carrier_code` sans manifeste | Omettre ou changer le `carrier_code` |
| `404` | `Condição de remessa incompativel com os transportadores autorizados!` (`status: "error"`) | Contrat sans expédition synchrone | Contacter le support |
| `400` | Retour du transporteur (`status: "error"`, avec `data`) | Le transporteur a refusé l'envoi | Corriger selon le retour |
| `400` | `MANIFEST WARNING - AWB cannot be generated, carrier not mapped` | Étiquette non générée après l'acceptation ; l'expédition existe | Rechercher dans Documents ; **ne pas** réessayer |

> Les erreurs de l'étape de cotation au sein du manifeste (boîtes, articles, kits, contrats) reviennent en `400 warning`,
> avec les mêmes messages que la [cotation](#erreurs-de-cotation).

## Erreurs de brouillon

`POST /v1/draft`, `PUT /v1/draft/{code}`, `GET /v1/draft` et `DELETE /v1/draft`.
Détails dans [Brouillon](api-rascunho.md#erreurs-courantes).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `400` | `Apenas rascunhos com o status incomplete podem ser editados` | `PUT` sur un brouillon hors de `incomplete` | Consulter le statut |
| `400` | `O(s) rascunho(s) selecionado(s) não foi/foram encontrado(s) na nossa base de dados` | Code inexistant, d'un autre compte ou déjà converti en expédition | Vérifier le code |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` | Défaillance inattendue | Réessayer avec backoff |

> Les erreurs du traitement asynchrone du brouillon ne reviennent pas dans la réponse HTTP : elles apparaissent dans
> `status_descricao` du brouillon (ex. : `Nenhum item foi enviado para este rascunho`).

## Erreurs de boîte et d'articles

`/v1/box` et `/v1/item`. Détails dans [Boîte](api-caixa-v1.md#erreurs-courantes) et [Articles](api-itens-v1.md#erreurs-courantes).

| Statut | Message | Cause | Comment corriger |
|---|---|---|---|
| `422` | Validation (dans `data`) | Champs de boîte invalides ; filtres de liste invalides | Corriger les champs listés |
| `500` | `Algo inesperado aconteceu. Tente novamente mais tarde!` (`status: "error"`) | Défaillance inattendue | Réessayer avec backoff |
| `200` + rapport | `Box Error: box :id not found!` / `Box Error: box :id cannot be deleted, has related item` | Suppression de boîte refusée lors du traitement | Utiliser un `box_code` entier / dissocier les articles |
| `200` + rapport | `sku => :sku, status => invalid, reason => :campo` | Article écarté lors du traitement asynchrone | Corriger le champ indiqué |

> Les enregistrements de boîtes et d'articles sont **asynchrones** : la réponse `200` confirme la réception, et les enregistrements
> invalides sont écartés ensuite, avec un rapport envoyé à l'e-mail du compte.

## Règles métier

### Politique de nouvelle tentative
| Situation | Réessayer automatiquement ? |
|---|---|
| `500` | Oui, avec backoff exponentiel (ex. : 2 s, 4 s, 8 s) et une limite de tentatives |
| `4xx` | Non. Corrigez d'abord le payload/le scénario |
| Manifeste avec `400` après l'acceptation du transporteur | **Non.** L'expédition existe déjà : consultez [Suivi](api-rastreio.md) et [Documents](api-documentos.md) |
| Expédition (`POST /v*/shipping`) avec timeout réseau | Avant de réessayer, consultez le statut via le `quotation` : une cotation ne génère qu'une seule expédition |

### Refus avec `status: "error"`
Certains refus métier liés au **poids** ou au **transporteur** arrivent avec `status: "error"` et le code
`400`/`404`. Traitez-les toujours selon le code HTTP et la `message`, et pas uniquement selon le `status`.

## Remarques finales
- Conventions générales de réponse dans [Premiers Pas](api-primeiros-passos.md#format-standard-des-réponses).
- Cotations qui ne peuvent pas devenir une expédition : [Cotation (v1)](api-cotacao-v1.md#cotations-non-éligibles-à-lexpédition).
- En cas de doute, communiquez au support le `quotation`, l'heure de l'appel et la `message` reçue.
