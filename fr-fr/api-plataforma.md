# Plateforme

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

> ⚠️ **Exemples purement illustratifs :** tous les payloads, réponses, identifiants, codes, documents et tokens
> de cette page sont fictifs et **ne sont pas des données valides**. Remplacez-les par les données réelles de votre compte et de votre activité.

## Vue d'ensemble
- Endpoint utilitaire qui renvoie le nom, l'environnement et la version de l'API Publique.
- C'est l'appel le plus simple et le moins coûteux de l'API — idéal pour **valider le token et la connectivité** avant
  toute intégration, et pour le **health check** en supervision.
- N'exige qu'un token valide : **ne dépend d'aucune permission** du profil.
- Il est en lecture seule et n'a aucun effet de bord.
- Cet endpoint n'apparaît pas dans la documentation interactive (`/doc`) ; cette page en est la référence.

## URL de base
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
lang: pt
```

> Cet endpoint **exige un token valide**, mais n'exige aucune permission spécifique.
> C'est pourquoi il constitue le meilleur test pour savoir si votre token fonctionne.
> Le header `lang` (`pt`, `en`, `es`) modifie uniquement le texte de `message`.

## Flux d'utilisation

```
1. GET /v1/platform/version   → 200 : token valide et API accessible
2. Vérifier data.env          → environnement attendu (production ou préproduction)
3. Passer à l'intégration     → cotation, expédition, etc.
```

## Endpoint : version de la plateforme

- **Méthode et route :** `GET /v1/platform/version`
- **Quand l'utiliser :** lors de la configuration d'un nouveau token, lors d'un changement d'environnement et lors des vérifications périodiques de santé.

### Paramètres

Aucun paramètre de path, de query ni de corps. Uniquement les headers d'[Authentification](#authentification).

### Payload minimal

Sans corps :

```
GET /v1/platform/version
```

### Payload maximal

Il n'y a pas de paramètres optionnels. La seule variation possible est la langue du message, via le header :

```
GET /v1/platform/version
lang: en
```

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Accept: application/json' \
  --header 'lang: pt'
```

### Exemple de réponse

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

### Champs de la réponse

| Champ | Type | Description |
|---|---|---|
| `status` | string | Toujours `success` lorsque le token est accepté |
| `message` | string | `Registro(s) listado(s) com sucesso.` (traduit selon le header `lang`) |
| `data.name` | string \| null | Nom de l'application |
| `data.env` | string \| null | Environnement auquel vous êtes connecté (ex. : `production`, `staging`) |
| `data.version` | string \| null | Version publiée de l'API |

> Vérifiez l'`env` pour vous assurer que vous ne pointez pas vers le mauvais environnement — c'est une erreur fréquente
> dans les intégrations qui partagent la configuration entre environnements.
>
> Les valeurs de `data` sont informatives et définies par la configuration de chaque environnement. Ne les utilisez pas comme
> clé de logique (ex. : comparaison exacte de version) et considérez `null` comme une valeur possible.

## Cas d'usage

| Cas | Comment l'utiliser |
|---|---|
| Valider un nouveau token | S'il répond `200`, le token est valide |
| Diagnostiquer un `404` d'authentification | Si cet endpoint répond aussi `404`, le problème vient du token, pas de la route |
| Diagnostiquer un `403 Has no permission!` sur une autre route | Si cet endpoint répond `200`, le token est valide et il ne manque que la permission de cette opération |
| Confirmer l'environnement | Lire `data.env` |
| Health check de supervision | Appel léger, sans effet de bord |
| Enregistrer la version dans vos logs | Utile pour corréler le comportement avec les releases de l'API |

## Règles métier

| Règle | Détail |
|---|---|
| Authentification | Le token est validé avant toute réponse. Sans header `Authorization`, la réponse est `404` |
| Permission | Aucune. Tout token valide d'un utilisateur actif reçoit `200` |
| Compte bloqué | Le token d'un utilisateur bloqué reçoit `403` |
| Version dans le path | Les appels sans le préfixe (`/platform/version`) reçoivent un `308` vers `/v1/platform/version`, en préservant la query string |
| Lecture seule | Ne modifie ni les cotations, ni les expéditions, ni les enregistrements |

## Erreurs courantes

| Statut | Signification/Message | Comment corriger |
|---|---|---|
| `404` | `Nenhum usuário encontrado para o token especificado!` | Token absent, incorrect ou régénéré. Vérifier le header `Authorization: Bearer ...` et, si nécessaire, générer un nouveau token dans le panneau |
| `403` | `Usuário especificado encontra-se bloqueado em nosso sistema!` | Compte bloqué. Contacter le support |
| `400` | `status: "error"` avec la description de l'échec d'authentification | Échec inattendu lors de la validation du token. Réessayer avec backoff ; si cela persiste, contacter le support |
| `308` | Redirection permanente | Appel sans `/v1`. Utiliser la route complète `/v1/platform/version` |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Cet endpoint est en **lecture seule** et n'a aucun effet sur les cotations, les expéditions ou les enregistrements.
- Les appels sans le préfixe de version (`/platform/version`) sont redirigés avec un `308` vers
  `/v1/platform/version`. Préférez toujours indiquer la version.
- Détails sur le token, la langue et le format des réponses : [Premiers Pas](api-primeiros-passos.md).
- Liste consolidée des messages d'erreur : [Codes d'Erreur](api-codigos-de-erro.md).
