# SecurePipeline — Enterprise DevSecOps Security Pipeline

> Pipeline CI/CD Jenkins transformée en chaîne **DevSecOps de bout en bout** : contrôles de sécurité automatisés à chaque phase du cycle de vie — Pre-Commit, Commit, Acceptance et Production — appliqués à une application Spring Boot (gestion de feuilles de temps) déployée sur Kubernetes (minikube).

![Status](https://img.shields.io/badge/pipeline-passing-brightgreen)
![DevSecOps](https://img.shields.io/badge/DevSecOps-9%20tools%20integrated-blue)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

## Sommaire

- [Contexte](#contexte)
- [Architecture](#architecture)
- [Outils intégrés par phase](#outils-intégrés-par-phase)
- [Détail des stages](#détail-des-stages)
- [Résultats](#résultats)
- [Stack technique](#stack-technique)
- [Structure du dépôt](#structure-du-dépôt)
- [Pistes d'amélioration](#pistes-damélioration)

## Contexte

Le projet part d'une pipeline DevOps classique et fonctionnelle (Checkout → Compile → Test → SonarQube → Package → Docker Build/Push → Deploy to Kubernetes) et y intègre, étape par étape, des contrôles de sécurité automatisés couvrant **4 phases** : Développement (Pre-Commit/Commit), Acceptance et Production — en suivant une logique *shift-left* : plus un problème est détecté tôt, moins il coûte cher à corriger.

Chaque outil a été choisi pour correspondre à une **catégorie précise** d'un référentiel DevSecOps (détection de secrets, SAST, SCA, security scanning, smoke tests, détection d'intrusion, gestion de secrets, durcissement système, monitoring continu), intégré nativement dans le `Jenkinsfile`, puis validé par un **test réel et reproductible** — pas seulement configuré.

## Architecture

```
Checkout → Compile → Test → SonarQube
    │
    ├─ 🔒 Gitleaks (Pre-Commit, hook Git local)
    ├─ 🔒 SpotBugs + Find Security Bugs (Commit)
    │
    ▼
Package → Deploy to Nexus → Docker Build/Push
    │
    ├─ 🔑 Vault (sync secrets avant déploiement)
    ▼
Deploy to Kubernetes → Verify Deployment
    │
    ├─ 🛰 Nmap (Acceptance — Security Scanning)
    │
    ▼
    ├─ 🕷 OWASP ZAP (Production — Security Smoke Tests)
    ├─ 🛡 fail2ban (Production — Host Intrusion Detection)
    ├─ 🧰 OpenSCAP (Production — Server Hardening, CIS Benchmark)
    ├─ 📊 Prometheus + Grafana (Production — Continuous Monitoring)
    ├─ 📧 Email Alerting (Gmail SMTP — notifications de build/sécurité)
    │
    ▼
Rapport final
```

## Outils intégrés par phase

| Outil | Phase | Catégorie | Rôle |
|---|---|---|---|
| **Gitleaks** | Pre-Commit | Détection de secrets | Bloque un commit contenant une clé API / credential en dur |
| **SpotBugs + Find Security Bugs** | Commit | SAST | Analyse statique du bytecode Java, détecte les vulnérabilités de code |
| **OWASP Dependency-Check** | Commit | SCA | Confronte les dépendances Maven à la base NVD (CVE connues) |
| **Nmap** | Acceptance | Security Scanning | Vérifie qu'aucun port/service inattendu n'est exposé post-déploiement |
| **OWASP ZAP** | Production | Security Smoke Tests | DAST sur l'application réellement en ligne |
| **fail2ban** | Production | Host Intrusion Detection | Bannit automatiquement une IP après échecs d'authentification SSH |
| **Hashicorp Vault** | Production | Secrets Management | Centralise les secrets, synchronisés dynamiquement vers Kubernetes |
| **OpenSCAP** | Production | Server Hardening | Audit de durcissement système (CIS Benchmark Level 1) |
| **Prometheus + Grafana** | Production | Continuous Monitoring | Observabilité continue de l'infrastructure et de l'application |
| **Email Alerting (Gmail SMTP)** | Production | Alerting | Notifications automatiques en cas d'échec de build ou de détection de sécurité |

## Détail des stages

<details>
<summary><b>🔒 Gitleaks — Pre-Commit</b></summary>

Hook Git local (`.git/hooks/pre-commit`), exécuté via Docker, qui scanne les fichiers indexés avant chaque commit :

```bash
docker run --rm -v "$(pwd):/repo" zricethezav/gitleaks:latest \
  protect --source="/repo" --staged --verbose --redact
```

**Test validé** : une clé AWS factice insérée et indexée → commit bloqué immédiatement.
</details>

<details>
<summary><b>🔎 SpotBugs + Find Security Bugs — Commit</b></summary>

Plugin Maven exécuté à chaque build Jenkins sur le bytecode compilé :

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <configuration>
    <plugins>
      <plugin>
        <groupId>com.h3xstream.findsecbugs</groupId>
        <artifactId>findsecbugs-plugin</artifactId>
      </plugin>
    </plugins>
  </configuration>
</plugin>
```

**Résultat** : 4 types de vulnérabilités détectées dans le code existant (`EI_EXPOSE_REP`, `ENTITY_MASS_ASSIGNMENT`, `ENTITY_LEAK`, `EI_EXPOSE_REP2`).
</details>

<details>
<summary><b>🛰 Nmap — Acceptance (Security Scanning)</b></summary>

```groovy
stage('Security Acceptance - Nmap') {
    steps {
        sh 'nmap -sT -sV -Pn -p 22,80,443,3306,8080,8081,9000,31046 192.168.49.2 -oN nmap-report.txt'
    }
    post { always { archiveArtifacts artifacts: 'nmap-report.txt' } }
}
```

**Résultat** : aucun service interne exposé (MySQL, Jenkins, Nexus, SonarQube restent fermés).
</details>

<details>
<summary><b>🕷 OWASP ZAP — Security Smoke Tests</b></summary>

```groovy
sh 'docker run --rm --network host -v "$WORKSPACE/zap-reports:/zap/wrk/:rw" \
    ghcr.io/zaproxy/zaproxy:stable \
    zap-baseline.py -t http://192.168.49.2:31046 -r zap-report.html'
```

**Résultat** : 64 règles passées, 0 échec critique, 3 avertissements (en-têtes CSP / Permissions-Policy manquants).
</details>

<details>
<summary><b>🛡 fail2ban — Host Intrusion Detection</b></summary>

```ini
[sshd]
enabled = true
backend = systemd
maxretry = 5
bantime = 600
```

**Test validé** : 10 tentatives SSH échouées simulées → IP `127.0.0.1` bannie automatiquement.
</details>

<details>
<summary><b>🔑 Hashicorp Vault — Secrets Management</b></summary>

```groovy
withCredentials([string(credentialsId: 'vault-root-token', variable: 'VAULT_TOKEN')]) {
    sh '''
        DB_PASSWORD=$(vault kv get -field=password secret/timesheet/mysql)
        kubectl create secret generic timesheet-secret \
          --from-literal=SPRING_DATASOURCE_PASSWORD="$DB_PASSWORD" \
          -n chap4 --dry-run=client -o yaml | kubectl apply -f -
    '''
}
```

Remplace un mot de passe MySQL auparavant stocké en base64 en clair dans un fichier versionné.
</details>

<details>
<summary><b>🧰 OpenSCAP — Server Hardening</b></summary>

```bash
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --results ~/oscap-results.xml \
  --report ~/oscap-report.html \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-ds.xml
```

Complété par des vérifications OVAL ciblées (AppArmor, AIDE, auditd, chrony...) pour compenser la couverture partielle du contenu SCAP sur Ubuntu.
</details>

<details>
<summary><b>📊 Prometheus + Grafana — Continuous Monitoring</b></summary>

Stack déployée via Helm sur minikube (`namespace monitoring`), avec un stage Jenkins de health-check post-déploiement et une alerte email si la stack est down :

```groovy
stage('Continuous Monitoring - Prometheus & Grafana') {
    steps {
        sh '''
            kubectl get pods -n monitoring -l app=prometheus
            kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
        '''
    }
    post {
        failure {
            emailext subject: "⚠️ Monitoring stack DOWN", to: 'ineshassine2003@gmail.com'
        }
    }
}
```
</details>

## Résultats

- **9 outils de sécurité** intégrés et testés, couvrant les 4 phases du cycle de vie DevSecOps
- Chaque outil validé par un **test réel** (pas seulement une configuration statique) : secret bloqué, vulnérabilité détectée, IP bannie, port fermé, secret synchronisé, paquet manquant identifié
- Limitations documentées en toute transparence (couverture du contenu SCAP pour Ubuntu, délai de propagation réseau observé sur Nmap) plutôt que masquées

Rapport technique complet disponible dans [`/docs/Rapport_DevSecOps_COMPLET.pdf`](./docs/Rapport_DevSecOps_COMPLET.pdf).

## Stack technique

`Jenkins` · `Docker` · `Kubernetes (minikube)` · `Maven` · `Spring Boot` · `Gitleaks` · `SpotBugs / Find Security Bugs` · `OWASP Dependency-Check` · `Nmap` · `OWASP ZAP` · `fail2ban` · `Hashicorp Vault` · `OpenSCAP` · `Prometheus` · `Grafana`

## Structure du dépôt

```
SecurePipeline/
├── Jenkinsfile                  # Pipeline CI/CD complète (9 stages de sécurité)
├── .git-hooks/pre-commit        # Hook Gitleaks
├── k8s/                         # Manifests Kubernetes (deployment, service, secrets)
├── monitoring/                  # Values Helm Prometheus/Grafana
├── docs/
│   └── Rapport_DevSecOps_COMPLET.pdf
└── README.md
```

## Pistes d'amélioration

- Finaliser l'activation d'OWASP Dependency-Check (en attente de stabilisation de l'API NVD)
- Installer les 3 paquets de durcissement manquants identifiés par OpenSCAP (`aide`, `auditd`, `chrony`)
- Investiguer le délai de propagation réseau observé sur le scan Nmap juste après un rollout Kubernetes
- Ajouter les en-têtes CSP / Permissions-Policy signalés par ZAP

---

**Auteur** : Ines Hassine — [github.com/ines231](https://github.com/ines231)
