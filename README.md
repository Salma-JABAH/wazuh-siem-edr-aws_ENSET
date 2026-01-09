# 🛡️ Plateforme SIEM/EDR avec Wazuh sur AWS

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazon-aws)](https://aws.amazon.com/)
[![Wazuh](https://img.shields.io/badge/Wazuh-4.7-blue?logo=wazuh)](https://wazuh.com/)
[![Linux](https://img.shields.io/badge/OS-Linux-yellow?logo=linux)](https://ubuntu.com/)
[![Windows](https://img.shields.io/badge/OS-Windows-blue?logo=windows)](https://www.microsoft.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## 📋 Description

Projet pratique de déploiement d'une **plateforme complète de supervision et de protection des endpoints** combinant les approches **SIEM (Security Information and Event Management)** et **EDR (Endpoint Detection and Response)** dans un environnement multi-OS sur AWS.

Cette solution open-source basée sur **Wazuh** démontre comment centraliser, détecter et répondre aux incidents de sécurité en temps réel dans un environnement cloud professionnel.

## 🎯 Objectifs du projet

| Axe | Objectif |
|-----|----------|
| **🏗️ Architectural** | Concevoir et déployer une infrastructure de supervision complète sur AWS avec segmentation réseau et sécurisation des communications |
| **⚙️ Technique** | Implémenter les composants Wazuh (serveur, indexeur, dashboard, agents) sur des systèmes hétérogènes (Linux/Windows) |
| **🔍 Opérationnel** | Démontrer les capacités de détection via des scénarios réalistes d'attaques (brute force, élévation de privilèges, persistance) |
| **📊 Analytique** | Évaluer la complémentarité SIEM/EDR et formuler des recommandations pour leur intégration dans un SOC moderne |

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    AWS VPC (10.0.0.0/16)                │
│                                                          │
│  ┌──────────────────┐      ┌──────────────────┐        │
│  │  Wazuh Server    │      │   Linux Client   │        │
│  │  (Ubuntu 22.04)  │◄─────┤   (Ubuntu 22.04) │        │
│  │                  │      │                  │        │
│  │  • Manager       │      │  • Wazuh Agent   │        │
│  │  • Indexer       │      │  • SSH Logging   │        │
│  │  • Dashboard     │      │  • FIM           │        │
│  └────────┬─────────┘      └──────────────────┘        │
│           │                                             │
│           │                ┌──────────────────┐        │
│           │                │  Windows Client  │        │
│           └────────────────┤  (Server 2022)   │        │
│                            │                  │        │
│                            │  • Wazuh Agent   │        │
│                            │  • Sysmon        │        │
│                            │  • Event Logs    │        │
│                            └──────────────────┘        │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## 🚀 Stack technique

### Infrastructure
- **Cloud Provider** : AWS (Learner Lab)
- **Compute** : EC2 (t3.large pour serveur, t2.micro/medium pour clients)
- **Réseau** : VPC avec Security Groups segmentés
- **Stockage** : EBS GP3 (30 GB pour le serveur)

### Outils de sécurité
- **SIEM/EDR** : Wazuh 4.7 (Manager + Indexer + Dashboard)
- **Agents** : Wazuh Agent (Linux & Windows)
- **EDR enrichi** : Sysmon (Windows)
- **OS** : Ubuntu 22.04 LTS (Linux) + Windows Server 2022

### Composants Wazuh
- **Manager** : Collecte, normalisation et corrélation des événements
- **Indexer** : Stockage et indexation (basé sur OpenSearch)
- **Dashboard** : Visualisation et analyse (basé sur OpenSearch Dashboards)

## 📦 Installation

### Prérequis
- Compte AWS (Learner Lab recommandé)
- Connaissances de base en Linux/Windows
- Clé SSH pour l'accès aux instances
- Navigateur web pour accéder au dashboard

### Étape 1 : Déploiement du serveur Wazuh

```bash
# Sur l'instance Ubuntu 22.04
sudo apt update && sudo apt -y upgrade

# Téléchargement du script d'installation All-in-One
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh

# Installation complète (Manager + Indexer + Dashboard)
sudo bash wazuh-install.sh -a
```

### Étape 2 : Configuration des Security Groups

**Serveur Wazuh (Wazuh-Server-SG)**
```
INBOUND:
- SSH (22/tcp)         → IP administrateur
- HTTPS (443/tcp)      → IP administrateur (dashboard)
- Wazuh (1514/tcp)     → Security Group clients
- Enrollment (1515/tcp) → Security Group clients

OUTBOUND:
- Tous les ports → 0.0.0.0/0
```

**Clients (Clients-SG)**
```
INBOUND:
- SSH (22/tcp)   → IP administrateur (Linux)
- RDP (3389/tcp) → IP administrateur (Windows)

OUTBOUND:
- 1514-1515/tcp → Security Group Wazuh
```

### Étape 3 : Déploiement agent Linux

```bash
# Sur le client Linux Ubuntu
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb

# Installation
sudo WAZUH_MANAGER='<WAZUH_SERVER_IP>' WAZUH_AGENT_NAME='Linux-Client' \
     dpkg -i wazuh-agent_4.7.5-1_amd64.deb

# Démarrage
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Étape 4 : Déploiement agent Windows

```powershell
# Sur le client Windows Server 2022
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi" `
                  -OutFile "C:\wazuh-agent.msi"

# Installation silencieuse
msiexec.exe /i "C:\wazuh-agent.msi" /q `
            WAZUH_MANAGER="<WAZUH_SERVER_IP>" `
            WAZUH_REGISTRATION_SERVER="<WAZUH_SERVER_IP>" `
            WAZUH_AGENT_NAME="Windows-Client"

# Démarrage du service
NET START WazuhSvc
```

### Étape 5 : (Optionnel) Installation Sysmon sur Windows

```powershell
# Téléchargement Sysmon
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" `
                  -OutFile "C:\Sysmon.zip"

# Extraction
Expand-Archive -Path "C:\Sysmon.zip" -DestinationPath "C:\Sysmon"

# Téléchargement d'une configuration (ex: SwiftOnSecurity)
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
                  -OutFile "C:\Sysmon\config.xml"

# Installation avec configuration
C:\Sysmon\sysmon64.exe -accepteula -i C:\Sysmon\config.xml
```

## 🔬 Scénarios de test

### 🐧 Linux - SIEM

#### Scénario 1 : Tentatives SSH échouées (Brute Force)
```bash
# Simulation d'attaque brute force
ssh fakeuser@<LINUX_CLIENT_IP>
# Répéter 5-10 fois avec mauvais mot de passe
```
**Résultat attendu** : Alertes "authentication failed" dans le dashboard

#### Scénario 2 : Élévation de privilèges
```bash
# Utilisation de sudo
sudo su
```
**Résultat attendu** : Événements sudo tracés et visibles

#### Scénario 3 : File Integrity Monitoring (FIM)
```bash
# Modification d'un fichier sensible
echo "test" | sudo tee -a /etc/passwd
```
**Résultat attendu** : Alerte FIM de modification détectée

### 🪟 Windows - EDR

#### Scénario 1 : Échecs de login RDP
```
1. Déconnexion de la session Windows
2. Tentatives de connexion RDP avec mauvais mot de passe (3-5 fois)
3. Reconnexion avec les bons identifiants
```
**Résultat attendu** : Event ID 4625 (Failed logon) détecté

#### Scénario 2 : Création d'utilisateur non autorisé
```powershell
# Création d'un compte local
net user labuser P@ssw0rd! /add

# Ajout au groupe Administrateurs
net localgroup administrators labuser /add
```
**Résultats attendus** :
- Event ID 4720 : "A user account was created"
- Event ID 4732 : "A member was added to a security-enabled local group"

## 📊 Visualisation et analyse

### Accès au dashboard
```
URL : https://<WAZUH_SERVER_PUBLIC_IP>
Utilisateur : admin
Mot de passe : <généré lors de l'installation>
```

### Vues principales

**Security Events**
- Tous les événements de sécurité agrégés
- Filtrage par agent, règle, sévérité
- Recherche plein texte

**Threat Hunting**
- Requêtes avancées pour recherche proactive
- Détection de patterns d'attaque
- Investigation forensique

**Agents Management**
- Statut des agents (Active/Disconnected)
- Dernière communication (keep-alive)
- Détails de configuration

## 📈 Résultats obtenus

### Métriques de détection

| Type d'événement | Linux | Windows | Total |
|------------------|-------|---------|-------|
| Tentatives d'authentification échouées | ✅ | ✅ | Détecté |
| Élévations de privilèges | ✅ | ✅ | Détecté |
| Modifications de fichiers sensibles | ✅ | N/A | Détecté |
| Créations de comptes | N/A | ✅ | Détecté |
| Surveillance processus (Sysmon) | N/A | ✅ | Détecté |

### Captures d'écran
Retrouvez les captures d'écran dans le dossier [`/screenshots`](./screenshots/) :
- Agents actifs dans le dashboard
- Alertes SSH brute force
- Événements Windows Security
- Détections FIM
- Processus Sysmon

## 🎓 Apports pour un SOC

### 🔍 Pour la détection
- Surveillance centralisée 24/7 multi-OS
- Détection temps réel des comportements suspects
- Corrélation d'événements entre systèmes
- Couverture MITRE ATT&CK

### 🕵️ Pour l'investigation
- Reconstruction chronologique des incidents
- Identification de l'étendue des compromissions
- Collecte d'artefacts forensiques
- Capacités de threat hunting

### ⚡ Pour la réponse
- Isolation rapide d'endpoints compromis
- Terminaison de processus malveillants
- Remédiation ciblée
- Automatisation possible (SOAR)

## 🚧 Limitations et améliorations futures

### Limitations actuelles
- Environnement de lab (non production)
- Pas d'intégration Threat Intelligence
- Réponse automatique non implémentée
- Scalabilité limitée (AWS Learner Lab)

### Améliorations envisagées
- [ ] Intégration avec sources de Threat Intelligence (MISP, VirusTotal)
- [ ] Automatisation des réponses (SOAR avec Shuffle/TheHive)
- [ ] Enrichissement avec YARA rules
- [ ] Déploiement haute disponibilité (cluster Wazuh)
- [ ] Analyse comportementale avec Machine Learning
- [ ] Intégration SIEM cloud (AWS CloudTrail, GuardDuty)

## 📚 Documentation

- [Rapport complet du projet](./docs/rapport_projet.pdf)
- [Documentation Wazuh officielle](https://documentation.wazuh.com/)
- [Guide de déploiement AWS](./docs/deployment_guide.md)
- [Règles de détection personnalisées](./rules/)

## 🤝 Contribution

Les contributions sont les bienvenues ! N'hésitez pas à :
1. Fork le projet
2. Créer une branche (`git checkout -b feature/amelioration`)
3. Commit vos changements (`git commit -m 'Ajout nouvelle fonctionnalité'`)
4. Push vers la branche (`git push origin feature/amelioration`)
5. Ouvrir une Pull Request

## 📝 License

Ce projet est sous licence MIT. Voir le fichier [LICENSE](LICENSE) pour plus de détails.

## 👤 Auteur

**Votre Nom**
- GitHub: Salma-JABAH
- LinkedIn: www.linkedin.com/in/salma-jabah-b909b4268

## 🙏 Remerciements

- **Pr. Azeddine KHIAT** pour son encadrement et son expertise en cybersécurité
- [Wazuh](https://wazuh.com/) pour la plateforme open-source
- [AWS](https://aws.amazon.com/) pour l'infrastructure cloud
- La communauté SIEM/EDR pour le partage de connaissances
- [SwiftOnSecurity](https://github.com/SwiftOnSecurity) pour la configuration Sysmon

