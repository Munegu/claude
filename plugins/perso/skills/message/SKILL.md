---
name: message
description: "Rédige un message prêt à copier-coller (Slack/Teams/mail) avec le ton de Nicolas, adapté au destinataire : CDP, client, prestataire/hébergeur, CTO. S'appuie sur le contexte de la conversation (bug analysé, dev terminé, estimation, question à poser). TRIGGER when: user invokes /message, demande de faire un mess / un message / une réponse pour sa CDP, le client, le presta, l'hébergeur ou le CTO, demande « je réponds quoi », « je dis quoi », « je peux répondre ça ? », ou demande de reformuler un message « en gardant mon ton »."
argument-hint: "[cdp|client|presta|cto] [sujet ou brouillon]"
allowed-tools: [Read, Bash, Glob, Grep]
author: NV
---

# Message avec mon ton

Produit un message court, prêt à copier-coller, à partir du contexte de la conversation (analyse d'un bug, dev terminé, estimation, infos manquantes…) et éventuellement d'un brouillon fourni.

## 1. Identifier le cas

À partir des arguments et de la conversation, déterminer :

- **Destinataire** : `cdp` (cheffe/chef de projet, interne Adveris), `client`, `presta` (hébergeur / prestataire externe, souvent relayé par la CDP), `cto` (interne, technique).
- **Objectif** : informer (c'est fait / c'est corrigé), signaler (c'est KO, ça vient de l'infra), demander (infos manquantes, accès, validation), estimer (temps + questions), relayer une piste technique.
- **Brouillon fourni ?** → si oui, on le **corrige/resserre** sans le réécrire entièrement : garder ses tournures, corriger les fautes, enlever le superflu.

Si le destinataire n'est pas déductible, le demander en une ligne. Ne pas poser d'autres questions : rédiger avec ce qu'on a.

## 2. Le ton de Nicolas

Caractéristiques à reproduire :

- **Court et direct.** 2 à 6 lignes en général. Pas d'intro ni de formule de politesse longue.
- **Tutoiement** avec la CDP et le CTO. Avec un client : tutoiement si le contexte le montre, sinon vouvoiement simple.
- **Ouverture minimale** : « Re ! », « Hello », ou rien. Pas de « J'espère que tu vas bien ».
- **Langage parlé assumé** : « du coup », « en gros », « à priori », « ça risque d'être infra », « c'est KO », « me manque pas mal d'infos ».
- **Factuel** : ce qui a été constaté, ce qui a été fait, ce qu'il manque. Pas de jargon inutile, encore moins pour un client.
- **Humble sur les pistes** : quand une solution vient d'une analyse non vérifiée, le dire (« d'après notre analyse (assistée par IA), ça viendrait de… »). Ne pas affirmer une cause non confirmée.
- **Finit par la demande ou la suite** quand il y en a une (« Tu peux voir avec eux ? », « On sait lequel ? Avec quels objectifs ? »).
- Pas d'emojis, pas de gras/titres Markdown sauf listes courtes quand il y a plusieurs points (estimations, questions).

### Exemples réels (référence de ton)

> Me manque pas mal d'info par contre pour pouvoir te donner une vraie réponse.
> Mise en place d'un CDN : on sait lequel ? Avec quels objectifs ?
> Ça sera infra et à gérer par l'hébergeur c'est sûr, mais selon les infos que tu me donnes on aura plus ou moins de choses à adapter de notre côté aussi.

> Je sais pas ce qu'ils ont fait mais ça risque d'être infra.
> Suite à leur déploiement pour le patch PHP, les fichiers et images uploadés n'ont plus l'air d'être sur le serveur.

> Toujours KO.
> Je te joins l'analyse IA pour donner plus d'idées au prestataire.
> D'après notre analyse (assistée par IA), le problème viendrait de […]. Quelques pistes à explorer de leur côté : […]

> Re !
> C'est fait pour le site Oubruncher. Tu as une nouvelle entrée en Back Office « Testeurs » où tu peux créer et gérer tes testeurs. […]

> J'ai fait un check en base de données et il y a un seul autre site dans le même cas, mais là c'est OK car on peut l'ouvrir dans une iframe. Le site de ton restaurant ne le permet pas.

> J'ai fait des estimations pour les différents sujets :
> - Dashboard Matomo : sans IA 5j de back / avec IA 3j (prévoir potentiellement du front selon ce qu'on veut)
> - Dashboard édito : sans IA 4j de back / avec IA 2j

## 3. Adapter au destinataire

| Destinataire | Niveau technique | Contenu attendu |
|---|---|---|
| **CDP** | Faible à moyen | Le constat, qui doit agir (nous / hébergeur / client), ce dont on a besoin. Une phrase de vulgarisation si nécessaire. Si le message sera relayé au presta, prévoir un bloc technique séparé qu'elle peut transférer tel quel. |
| **Client** | Faible | Ce qui a été fait / corrigé, où le trouver dans le BO, ce que ça change pour lui. Aucun nom de classe, de table ou de commande. Expliquer un bug par son effet (« un bug historique conditionnait l'affichage du bouton »), pas par son code. |
| **Presta / hébergeur** | Élevé | Constat précis (ce qui a été testé, résultat, log pertinent en une ligne), hypothèse présentée comme telle, pistes concrètes à vérifier de leur côté. Vouvoiement si on s'adresse à eux directement. |
| **CTO** | Élevé | Direct et technique : symptôme, cause probable, impact (ralentissements, erreurs), ce qu'on propose ou ce qu'on attend de lui. |

## 4. Rédiger

- Vérifier dans le contexte (conversation, code, logs) les faits avant de les écrire. Ne rien inventer : pas de date, de chiffre ou de cause non établis.
- Ne jamais inclure de secret (mot de passe, token, clé, IP interne sensible) même s'il est apparu dans la conversation.
- Pas de mention de personnes par leur nom sauf si Nicolas l'a donné pour ce message.

## 5. Format de sortie

Rendre le message dans un bloc de code (` ```text `) pour qu'il soit copiable tel quel, sans rien autour dans le bloc.

S'il y a deux destinataires (ex : un mot pour la CDP + un bloc technique à transmettre au presta), produire deux blocs distincts, chacun précédé d'une ligne indiquant à qui il est destiné.

Après le bloc : au maximum une ligne de remarque si un point mérite attention (info non confirmée, question à trancher avant d'envoyer). Sinon rien.

Si Nicolas demande « plus court », diviser la longueur par deux en gardant la demande finale.
