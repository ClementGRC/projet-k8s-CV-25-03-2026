# Projet Kubernetes - 5SDWANA

## Équipe
- Clément
- Vincent

## Description
POC Kubernetes pour un client e-commerce.
Déploiement de 3 applications conteneurisées avec passerelle Stripe.

## Applications
1. **ecommerce-stripe** : App Flask e-commerce avec paiement Stripe (obligatoire)
2. **dashboard** : Dashboard d'administration Flask
3. **admin** : Panel d'administration Django

## Structure
- `apps/` : Dockerfiles et code source par application
- `manifests/` : Fichiers YAML Kubernetes par environnement (dev, preprod, prod)
- `infra/` : Fichiers d'infrastructure as code
- `docs/` : Rapport de projet et schémas d'architecture
- `logs/` : Historiques (shell, docker, k8s events)
