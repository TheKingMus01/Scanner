# Guide de Lecture des Résultats Trivy

## Table des matières

1. [Types de scans Trivy](#types-de-scans-trivy)
2. [Lecture des résultats de licences](#lecture-des-résultats-de-licences)
3. [Lecture des résultats de vulnérabilités](#lecture-des-résultats-de-vulnérabilités)
4. [Lecture du SBOM](#lecture-du-sbom)
5. [Commandes utiles](#commandes-utiles)
6. [Interprétation pour Due Diligence](#interprétation-pour-due-diligence)

---

## Types de scans Trivy

| Type | Commande | Description |
|------|----------|-------------|
| **License** | `trivy fs --scanners license` | Audit des licences open source |
| **Vuln** | `trivy fs --scanners vuln` | Vulnérabilités de sécurité (CVE) |
| **Secret** | `trivy fs --scanners secret` | Secrets exposés (clés API, mots de passe) |
| **Config** | `trivy fs --scanners misconfig` | Mauvaises configurations (IaC) |
| **SBOM** | `trivy fs --format cyclonedx` | Software Bill of Materials |

---

## Lecture des résultats de licences

### Structure JSON

```json
{
  "Results": [
    {
      "Target": "backend/poetry.lock",      // Fichier scanné
      "Class": "license",
      "Licenses": [
        {
          "PkgName": "requests",            // Nom du package
          "Name": "Apache-2.0",             // Licence SPDX
          "Category": "notice",             // Catégorie de risque
          "Confidence": 1.0                 // Confiance de détection (0-1)
        }
      ]
    }
  ]
}
```

### Catégories de licences (du plus permissif au plus restrictif)

| Catégorie | Risque | Licences | Impact |
|-----------|--------|----------|--------|
| `unencumbered` | Aucun | Unlicense, CC0, WTFPL | Aucune obligation |
| `permissive` | Très faible | MIT, ISC, BSD-2-Clause | Attribution simple |
| `notice` | Faible | Apache-2.0, BSD-3-Clause | Notice + attribution |
| `reciprocal` | Moyen | MPL-2.0, EPL-2.0, CDDL | Modifications doivent rester open source |
| `restricted` | Élevé | LGPL-2.1, LGPL-3.0 | Copyleft faible - obligations de liaison |
| `forbidden` | Critique | GPL-2.0, GPL-3.0, AGPL-3.0 | Copyleft fort - tout le code doit être open source |
| `unknown` | À investiguer | UNKNOWN, NOASSERTION | Licence non identifiée |

### Terminologie SPDX pour licences inconnues

| Terme | Signification | Contexte |
|-------|---------------|----------|
| `NOASSERTION` | Licence non vérifiée/non déclarée par le créateur du SBOM | Standard SPDX |
| `NONE` | Explicitement pas de licence (domaine public) | Standard SPDX |
| `UNKNOWN` | Licence non identifiée | Trivy, pip-licenses |
| `UNLICENSED` | Pas de licence déclarée | npm/license-checker |

**Note :** `NOASSERTION` et `UNKNOWN` représentent le même risque → à investiguer manuellement.

### Extraire les licences par catégorie

```bash
# Toutes les licences
cat trivy-license-report.json | jq -r '.Results[].Licenses[]? | "\(.PkgName): \(.Name) (\(.Category))"'

# Licences à risque (restricted/forbidden)
cat trivy-license-report.json | jq -r '.Results[].Licenses[]? | select(.Category == "restricted" or .Category == "forbidden") | "\(.PkgName): \(.Name)"'

# Licences inconnues (UNKNOWN ou NOASSERTION)
cat trivy-license-report.json | jq -r '.Results[].Licenses[]? | select(.Name == "UNKNOWN" or .Name == "NOASSERTION") | .PkgName'

# Résumé par catégorie
cat trivy-license-report.json | jq -r '[.Results[].Licenses[]?.Category] | group_by(.) | map({category: .[0], count: length})'
```

### Résumé par type de licence

```bash
cat trivy-license-report.json | jq -r '[.Results[].Licenses[]?.Name] | group_by(.) | map({license: .[0], count: length}) | sort_by(-.count)'
```

---

## Lecture des résultats de vulnérabilités

### Structure JSON

```json
{
  "Results": [
    {
      "Target": "backend/poetry.lock",
      "Vulnerabilities": [
        {
          "VulnerabilityID": "CVE-2024-1234",    // Identifiant CVE
          "PkgName": "requests",                  // Package affecté
          "InstalledVersion": "2.28.0",           // Version actuelle
          "FixedVersion": "2.31.0",               // Version corrigée
          "Severity": "HIGH",                     // Sévérité
          "Title": "Description courte",
          "Description": "Description détaillée",
          "CVSS": {
            "nvd": {
              "V3Score": 7.5                      // Score CVSS (0-10)
            }
          }
        }
      ]
    }
  ]
}
```

### Niveaux de sévérité

| Sévérité | Score CVSS | Action |
|----------|------------|--------|
| `CRITICAL` | 9.0 - 10.0 | Action immédiate requise |
| `HIGH` | 7.0 - 8.9 | Corriger rapidement |
| `MEDIUM` | 4.0 - 6.9 | Planifier correction |
| `LOW` | 0.1 - 3.9 | Corriger si possible |
| `UNKNOWN` | N/A | Investiguer |

### Commandes d'extraction

```bash
# Compter par sévérité
cat trivy-vuln-report.json | jq '[.Results[].Vulnerabilities[]?.Severity] | group_by(.) | map({severity: .[0], count: length})'

# Lister les CRITICAL et HIGH
cat trivy-vuln-report.json | jq -r '.Results[].Vulnerabilities[]? | select(.Severity == "CRITICAL" or .Severity == "HIGH") | "\(.VulnerabilityID) | \(.PkgName) | \(.Severity) | \(.FixedVersion // "No fix")"'

# Vulnérabilités avec fix disponible
cat trivy-vuln-report.json | jq -r '.Results[].Vulnerabilities[]? | select(.FixedVersion != null) | "\(.PkgName): \(.InstalledVersion) → \(.FixedVersion)"'
```

---

## Lecture du SBOM

### Format CycloneDX

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.4",
  "components": [
    {
      "type": "library",
      "name": "requests",
      "version": "2.28.0",
      "purl": "pkg:pypi/requests@2.28.0",    // Package URL standard
      "licenses": [
        {
          "license": {
            "id": "Apache-2.0"
          }
        }
      ]
    }
  ]
}
```

### Format SPDX

```json
{
  "spdxVersion": "SPDX-2.3",
  "packages": [
    {
      "name": "requests",
      "versionInfo": "2.28.0",
      "licenseConcluded": "Apache-2.0",      // Licence confirmée
      "licenseDeclared": "Apache-2.0",       // Licence déclarée par l'auteur
      "downloadLocation": "https://pypi.org/project/requests/",
      "externalRefs": [
        {
          "referenceType": "purl",
          "referenceLocator": "pkg:pypi/requests@2.28.0"
        }
      ]
    },
    {
      "name": "some-unknown-pkg",
      "versionInfo": "1.0.0",
      "licenseConcluded": "NOASSERTION",     // Licence non déterminée
      "licenseDeclared": "NOASSERTION"
    }
  ]
}
```

**Valeurs spéciales SPDX :**
- `NOASSERTION` : Le créateur du SBOM n'a pas pu déterminer la licence
- `NONE` : Le package n'a explicitement aucune licence

### Commandes d'extraction SBOM

```bash
# Nombre total de composants (CycloneDX)
cat sbom-cyclonedx.json | jq '.components | length'

# Liste des composants avec versions
cat sbom-cyclonedx.json | jq -r '.components[] | "\(.name)@\(.version)"' | sort

# Composants par type
cat sbom-cyclonedx.json | jq '[.components[].type] | group_by(.) | map({type: .[0], count: length})'

# Export CSV des composants
cat sbom-cyclonedx.json | jq -r '.components[] | [.name, .version, .purl, (.licenses[0]?.license?.id // "UNKNOWN")] | @csv'
```

---

## Commandes utiles

### Générer les scans

```bash
PROJECT_DIR="/chemin/vers/projet"
OUTPUT_DIR="./scan-results"
mkdir -p $OUTPUT_DIR

# Scan de licences
trivy fs --scanners license --license-full --format json -o $OUTPUT_DIR/licenses.json $PROJECT_DIR
trivy fs --scanners license --format table $PROJECT_DIR > $OUTPUT_DIR/licenses.txt

# Scan de vulnérabilités
trivy fs --scanners vuln --format json -o $OUTPUT_DIR/vulnerabilities.json $PROJECT_DIR
trivy fs --scanners vuln --format table $PROJECT_DIR > $OUTPUT_DIR/vulnerabilities.txt

# Générer SBOM
trivy fs --format cyclonedx -o $OUTPUT_DIR/sbom-cyclonedx.json $PROJECT_DIR
trivy fs --format spdx-json -o $OUTPUT_DIR/sbom-spdx.json $PROJECT_DIR

# Scan complet (tout en un)
trivy fs --scanners vuln,license,secret --format json -o $OUTPUT_DIR/full-scan.json $PROJECT_DIR
```

### Filtrer par sévérité

```bash
# Ignorer les LOW
trivy fs --severity CRITICAL,HIGH,MEDIUM --scanners vuln .

# Seulement les CRITICAL
trivy fs --severity CRITICAL --scanners vuln .
```

### Ignorer certaines vulnérabilités

Créer un fichier `.trivyignore` :

```
# Ignorer des CVE spécifiques
CVE-2021-1234
CVE-2022-5678

# Ignorer par package
pkg:npm/lodash@4.17.20
```

---

## Interprétation pour Due Diligence

### Checklist Licences (IP Risk)

| Question | Comment vérifier | Impact |
|----------|------------------|--------|
| Y a-t-il du GPL/AGPL ? | `jq '.Results[].Licenses[] \| select(.Category == "forbidden")'` | Peut exiger open-sourcing du produit |
| Y a-t-il du LGPL ? | `jq '.Results[].Licenses[] \| select(.Name \| test("LGPL"))'` | OK si linking dynamique |
| Licences inconnues ? | `jq '.Results[].Licenses[] \| select(.Name == "UNKNOWN" or .Name == "NOASSERTION")'` | Risque juridique - investiguer |
| NOASSERTION dans SBOM ? | `jq '.packages[] \| select(.licenseConcluded == "NOASSERTION")'` | Licence non vérifiée |
| Combien de dépendances ? | `jq '.components \| length'` (SBOM) | Complexité de maintenance |

### Checklist Sécurité (Security Risk)

| Question | Comment vérifier | Impact |
|----------|------------------|--------|
| Vulnérabilités CRITICAL ? | `jq '.Results[].Vulnerabilities[] \| select(.Severity == "CRITICAL")'` | Risque d'exploitation immédiat |
| Vulnérabilités sans fix ? | `jq '.Results[].Vulnerabilities[] \| select(.FixedVersion == null)'` | Pas de solution disponible |
| Dépendances obsolètes ? | Comparer versions | Dette technique |

### Rapport résumé pour M&A

```bash
echo "=== LICENSE RISK SUMMARY ==="
echo "Forbidden (GPL/AGPL): $(cat licenses.json | jq '[.Results[].Licenses[]? | select(.Category == "forbidden")] | length')"
echo "Restricted (LGPL): $(cat licenses.json | jq '[.Results[].Licenses[]? | select(.Category == "restricted")] | length')"
echo "Unknown/NOASSERTION: $(cat licenses.json | jq '[.Results[].Licenses[]? | select(.Name == "UNKNOWN" or .Name == "NOASSERTION")] | length')"
echo ""
echo "=== SBOM LICENSE CHECK (SPDX) ==="
echo "NOASSERTION: $(cat sbom-spdx.json | jq '[.packages[]? | select(.licenseConcluded == "NOASSERTION")] | length')"
echo ""
echo "=== SECURITY RISK SUMMARY ==="
echo "Critical CVEs: $(cat vulnerabilities.json | jq '[.Results[].Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length')"
echo "High CVEs: $(cat vulnerabilities.json | jq '[.Results[].Vulnerabilities[]? | select(.Severity == "HIGH")] | length')"
echo ""
echo "=== SBOM SUMMARY ==="
echo "Total components: $(cat sbom-cyclonedx.json | jq '.components | length')"
```

---

## Limitations connues de Trivy

| Écosystème | Limitation | Solution alternative |
|------------|------------|---------------------|
| Python (poetry.lock) | Ne récupère pas toujours les licences | Utiliser `pip-licenses` |
| Python (sans venv) | `site-packages` non trouvé | Activer le venv avant scan |
| Node.js (sans node_modules) | Pas de scan possible | Exécuter `npm install` d'abord |
| Licences custom | Non détectées | Revue manuelle requise |

### Compléter avec pip-licenses (Python)

```bash
cd backend
poetry run pip install pip-licenses
poetry run pip-licenses --format=json > licenses-python.json
poetry run pip-licenses --summary
poetry run pip-licenses | grep -iE "GPL|LGPL|AGPL"
```

### Compléter avec license-checker (Node.js)

```bash
cd frontend
npx license-checker --json > licenses-nodejs.json
npx license-checker --summary
npx license-checker --onlyAllow "MIT;ISC;Apache-2.0;BSD-2-Clause;BSD-3-Clause"
```

---

## Ressources

- [Documentation Trivy](https://aquasecurity.github.io/trivy/)
- [SPDX License List](https://spdx.org/licenses/)
- [CycloneDX Specification](https://cyclonedx.org/specification/overview/)
- [PURL Specification](https://github.com/package-url/purl-spec)
