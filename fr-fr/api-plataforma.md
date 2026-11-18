# Plateforme

- [Retour au README de la langue](README.md)
- Prérequis : [Premiers Pas](api-primeiros-passos.md)

## Vue d'ensemble
- Endpoint utilitaire qui renvoie le nom, l'environnement et la version de l'API Publique.
- C'est l'appel le plus simple et le moins coûteux de l'API — idéal pour **valider le token et la connectivité** avant
  toute intégration, et pour le **health check** en supervision.

## Base URL
- Production : `https://api.shipsmart.global`
- Préproduction : `https://hmlapi.shipsmart.com.br`
- Documentation interactive : `https://api.shipsmart.global/doc` et `https://hmlapi.shipsmart.com.br/doc`

## Authentification
```
Authorization: Bearer VOTRE_TOKEN
```

> Cet endpoint **exige un token valide**, mais n'exige aucune permission spécifique.
> C'est pourquoi il constitue le meilleur test pour savoir si votre token fonctionne.

## Endpoint : version de la plateforme

- **Méthode et route :** `GET /v1/platform/version`
- **Paramètres :** aucun.

### Exemple de requête

```bash
curl --request GET \
  --url https://api.shipsmart.global/v1/platform/version \
  --header 'Authorization: Bearer VOTRE_TOKEN' \
  --header 'Accept: application/json'
```

### Exemple de réponse

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
| `data.name` | string | Nom de l'application |
| `data.env` | string | Environnement auquel vous êtes connecté (ex. : `production`, `staging`) |
| `data.version` | string | Version publiée de l'API |

> Vérifiez l'`env` pour vous assurer que vous ne pointez pas vers le mauvais environnement — c'est une erreur fréquente
> dans les intégrations qui partagent la configuration entre environnements.

## Cas d'usage

| Cas | Comment l'utiliser |
|---|---|
| Valider un nouveau token | S'il répond `200`, le token est valide |
| Diagnostiquer un `404` d'authentification | Si cet endpoint échoue aussi, le problème vient du token, pas de la route |
| Confirmer l'environnement | Lire `data.env` |
| Health check de supervision | Appel léger, sans effet de bord |
| Enregistrer la version dans vos logs | Utile pour corréler le comportement avec les releases de l'API |

## Erreurs fréquentes

| Statut | Signification | Comment corriger |
|---|---|---|
| `404` | Token non trouvé ou invalide | Vérifier le header `Authorization: Bearer ...` et générer un nouveau token dans le portail |
| `403` | Compte bloqué | Contacter le support |
| `500` | Erreur inattendue | Réessayer avec backoff ; si cela persiste, contacter le support |

## Remarques finales
- Cet endpoint est en **lecture seule** et n'a aucun effet sur les cotations, les expéditions ou les enregistrements.
- Les appels sans le préfixe de version (`/platform/version`) sont redirigés avec un `308` vers
  `/v1/platform/version`. Préférez toujours indiquer la version.
