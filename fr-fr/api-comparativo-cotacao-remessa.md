# Comparatif Devis et Expédition

- [Retour au README de la langue](README.md)

## Vue d'ensemble
- Explique la différence pratique entre les flux de devis et d'expédition dans les deux versions publiées.
- Utilisez ce guide pour choisir la version correcte avant l'intégration.

## Base URL
- Production : `https://api.shipsmart.global`
- Documentation : `https://api.shipsmart.global/docs`
- Homologation : `https://hmlapi.shipsmart.com.br`

## Authentification
- Bearer token dans le header `Authorization`.
- Header de langue : `lang`.

## Règle d'utilisation
- Le flux d'intégration par défaut est : devis et expédition en utilisant les endpoints de la première version.
- La deuxième version est indiquée lorsque vous ne souhaitez pas renseigner de données au-delà du pays dans le devis.

## Devis
### Différence pratique
- Dans la première version, le devis accepte le flux complet avec origine, destination, colis et articles.
- Dans la deuxième version, le devis est plus concis pour les scénarios où l'intégrateur souhaite renseigner moins de données au début du processus.

### Comparativo
| Thème | Première version | Deuxième version | Impact |
|---|---|---|---|
| Objectif | Devis complet | Devis simplifié | Choisissez le niveau de détail |
| Adresse | Plus de champs dans le contrat | Contrat plus concis | Moins de données en entrée |
| Articles | Flux complet et avancé | Flux explicite pour les articles | Meilleur pour les payloads concis |
| Consultation | Consultation et statut disponibles dans l'endpoint de la première version | Information non identifiée dans la source analysée | Les deux consultations opérationnelles restent dans la première version |

## Expédition
### Différence pratique
- Dans la première version, l'expédition couvre la création, la consultation de statut, la consultation de documents, l'annulation, la mise à jour et les pièces jointes.
- Dans la deuxième version, la source analysée montre uniquement la création de l'expédition.

### Comparativo
| Thème | Première version | Deuxième version | Impact |
|---|---|---|---|
| Création | Disponible | Disponible | Flux de base commun |
| Status | Disponible | Information non identifiée dans la source analysée | Le suivi opérationnel dépend de la première version |
| Documents | Disponible | Information non identifiée dans la source analysée | Le téléchargement des documents reste dans la première version |
| Annulation | Disponible | Information non identifiée dans la source analysée | Le cycle complet reste dans la première version |
| Mise à jour | Disponible uniquement dans des cas exceptionnels | Information non identifiée dans la source analysée | En règle générale, l'expédition est irréversible après sa création |

## Quand utiliser chacune
1. Utilisez la première version lorsque vous souhaitez le flux opérationnel complet.
2. Utilisez la deuxième version lorsque le payload initial du devis doit être plus simple.
3. Si votre flux dépend du status, des documents ou de l'annulation, conservez la première version.
4. Si vous créez via la deuxième version, le suivi des actions est effectué par les APIs de la première version.

## Observations finales
- La source analysée ne fournit pas de documentation équivalente complète de la deuxième version pour le status, les docs et l'annulation.
- Pour une opération complète, la première version reste la référence fonctionnelle.
