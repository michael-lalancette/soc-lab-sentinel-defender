# 🛡️ SOC Lab · Microsoft Sentinel & Defender XDR

Laboratoire SecOps Microsoft monté de zéro dans un seul tenant — expérience pratique sur
l'ensemble des compétences d'analyste des opérations de sécurité (couvrant l'ensemble des compétences d'analyste des opérations de sécurité).

> Chaque phase est documentée de bout en bout, y compris ce qui a échoué et comment je l'ai corrigé :
> ```
> ingestion → détection → automatisation → investigation → hunting
> ```

---

## 🎯 Objectif

Pratiquer chaque compétence d'analyste SecOps dans un vrai tenant, plutôt que de l'étudier
seulement en théorie :

- **Microsoft Sentinel** comme SIEM, sur un workspace Log Analytics
- **Defender XDR** et **Defender for Endpoint** dans le portail Defender unifié
- **Entra ID P2** (Identity Protection) et **Intune** pour l'identité et les appareils (IAM)
- **Logic Apps** pour l'automatisation de la réponse

---

## 🖥️ Architecture

![Architecture du lab : trois VM, le workspace Sentinel law-soc et le portail Defender unifié](images/architecture.png)

VMs :

- `CLOUD-W11` (VMware, local) : Windows 11 Enterprise, onboardé dans **Defender for Endpoint** et enrôlé dans **Intune**
- `CLOUD-WINDOWS` (Azure) : Windows Server, security events vers Sentinel via **AMA** et une **DCR**
- `CLOUD-UBUNTU` (Azure) : Ubuntu 24.04, Syslog vers Sentinel via **AMA** et une **DCR**

Sources connectées à Sentinel (`law-soc`) :

- Entra ID (sign-in logs et audit logs), Azure Activity, threat intelligence (STIX)
- Incidents et alertes Defender XDR, synchronisés automatiquement
- Une table custom (`LabCustomEvents_CL`) créée pour la Logs Ingestion API (DCE + DCR)

Comptes SOC, selon le principe du moindre privilège :

| Compte | Rôle SOC | Rôle Entra (Defender XDR) | Rôles Azure (Sentinel) |
|--------|----------|---------------------------|------------------------|
| `soc.reader` | Observateur | Security Reader | Microsoft Sentinel Reader |
| `soc.analyst` | Triage et réponse | Security Operator | Microsoft Sentinel Responder + Playbook Operator |
| `soc.engineer` | Règles, connecteurs, workbooks, playbooks | Security Administrator | Microsoft Sentinel Contributor + Logic App Contributor |
| `soc.target1`, `soc.target2` | Utilisateurs ciblés par les tests | Aucun | Aucun |

---

## ⚡ Workflow opérationnel

La télémétrie des trois VM et des services d'identité remonte vers Sentinel et Defender XDR.
Les **règles analytiques** (planifiées et quasi temps réel) et les **détections personnalisées**
lèvent des alertes, regroupées en **incidents**. À la création d'un incident, une **règle
d'automatisation** l'assigne, l'étiquette, ajoute des tâches de triage et déclenche un
**playbook Logic App** qui écrit un commentaire structuré.

L'analyste (`soc.analyst`) prend l'incident, l'investigue (récit d'attaque, entités,
chronologie), le documente avec un **gabarit normalisé** (temps en UTC, faits avant
conclusions), le **classe** (*True / Benign / False positive*) puis **répond** (isolation
d'appareil, live response). La **chasse aux menaces** en KQL complète le tout.

Trois identités, trois permissions : la personne qui construit (rôles Azure), le compte de
service de Sentinel qui exécute le playbook, et l'identité managée du playbook qui réécrit
dans l'incident — un bon rappel du moindre privilège.

---

## 🔎 Détections

| Détection | Type | Déclencheur | Technique (MITRE ATT&CK) |
|-----------|------|-------------|--------------------------|
| `SR - MULTIPLE FAILED LOGONS` | Planifiée | 6+ échecs de connexion RDP (Event ID 4625) | T1110 — Brute Force |
| `NRT - USER ACC CREATED` | Quasi temps réel | Création d'un compte local (Event ID 4720) | T1136 — Create Account |
| `SR - VIP SIGN-IN FAILURES` | Planifiée + watchlist | 3+ échecs sur un compte VIP | T1110 — Brute Force |
| `CD - ENCODED POWERSHELL` | Detection custom (Defender XDR) | `powershell -enc …` | T1059.001 — PowerShell |
| `TI map IP → SigninLogs` | Modèle (Threat Intelligence) | Connexion depuis une IP indicatrice | — |
| Anomalous RDP / UEBA | ML / comportement | Connexion RDP inhabituelle | — |

La règle VIP a été **optimisée** pour exclure les interruptions `50140` / `50125` (l'invite
« Rester connecté ? ») qui étaient comptées à tort comme des échecs — voir la leçon
d'optimisation dans le guide détaillé.

---

## 📊 Résultats

- **6 détections** construites et déclenchées sur de la vraie télémétrie (Windows, Linux, identité)
- **Incidents investigués et classés** (*True / Benign / False positive*) avec un gabarit de
  commentaire normalisé (chronologie en UTC)
- **Optimisation d'alerte** : élimination d'un faux positif récurrent (code `50140`) →
  moins de bruit, moins de fatigue d'alerte
- **Simulation d'hameçonnage complète** : remise → ouverture → clic → identifiants soumis →
  **signalement par l'utilisateur**, suivie dans Attack Simulation et les Submissions
- **Playbook d'automatisation** commentant les incidents (identité managée, moindre privilège)

---

## 🧠 Ce que j'ai appris

- **L'optimisation des alertes compte plus que leur volume.** Compter des interruptions
  (`50140`) comme des échecs, c'est entraîner un SOC à ignorer la règle → fatigue d'alerte.
- **Savoir où vivent les données.** Un courriel d'hameçonnage *simulé* est injecté dans la
  boîte : absent du suivi de messages, de Threat Explorer et des tables de courriel. Un vrai
  courriel entrant, c'est l'inverse.
- **Les portails affichent l'heure locale ; KQL travaille en UTC.** Un événement en soirée
  tombe le lendemain en UTC — un piège classique en investigation.
- **`join` est sensible à la casse** en KQL (`has`, `=~`, `in~` ne le sont pas) → `tolower()`.
- **Un connecteur « Connecté » ne prouve rien** — seules les lignes dans la table le prouvent.
- **Un tenant fraîchement provisionné est « déshydraté ».** `Enable-OrganizationCustomization`
  (et un délai de propagation) est requis avant que l'audit et eDiscovery fonctionnent.

---

## 📂 Documentation

👉 **[Guide détaillé des phases](docs/GUIDE.md)** — le pas-à-pas complet (phases 0 à 5),
avec les explications simplifiées (ELI5), les leçons tirées du build et les notes d'examen.

---

## 🚀 Prochaines étapes

J'étends présentement mes compétences vers la **gestion des identités et des accès (GIA / IAM)**
avec **Entra ID** — accès conditionnel, Identity Protection, PIM et gouvernance des identités —
en m'appuyant sur le travail d'identité déjà réalisé dans ce lab.

---

## ⚠️ Avertissement

Ce projet est un lab pédagogique destiné à l'apprentissage et à la pratique des opérations
de sécurité Microsoft.

- Les attaques sont simulées sur des VM et des comptes de test, jamais sur un poste personnel
  ou un environnement de production.
- Les noms de tenant, adresses IP publiques et identifiants réels ont été retirés ou floutés.
- Les configurations sont simplifiées à des fins d'étude et ne constituent pas une solution
  prête pour la production.

---

*Dernière mise à jour : octobre 2026*
