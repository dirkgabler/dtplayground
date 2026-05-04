# NpmBuildAndVersionJobTmpl

Die Template-Datei `NpmBuildAndVersionJobTmpl.yml` definiert eine Azure-DevOps-Jobvorlage für npm-Projekte. Sie baut das Projekt branch-abhängig, prüft Versionsregeln, veröffentlicht optional npm-Pakete und kann Build-Artefakte sowie Git-Tags erzeugen.

## Zweck

Die Vorlage standardisiert folgende Abläufe:

- Checkout des Repositories inklusive Submodules
- Prüfung auf `package.json` und `package-lock.json`
- Ermittlung der Paketversion aus `package.json`
- Validierung von Versions- und Branch-Regeln
- Ausführung von `npm ci`
- Ausführung konfigurierbarer Build-Kommandos
- Optionales `npm publish`
- Optionales Publizieren von Pipeline-Artefakten
- Optionales Pushen eines Git-Release-Tags
- Optionales Setzen von Pipeline-Retention

## Branch-Verhalten

Die Vorlage erzeugt drei Jobs, von denen abhängig vom aktuellen Branch genau einer läuft:

| Job | Branch-Bedingung | Verhalten |
| --- | --- | --- |
| `build_default_branch_job` | `master`, `main`, `develop` | Erlaubt nur Pre-Releases mit Label `snapshot`, publiziert in die Snapshot-Registry |
| `build_release_branch_job` | `release/*` | Erlaubt finale Versionen oder Pre-Releases mit Label `release`, publiziert in die Release-Registry, kann Git-Tag pushen |
| `build_other_branch_job` | alle übrigen Branches | Baut das Projekt ohne npm-Publish und ohne Artefakt-Publish |

## Versionslogik

Während der Vorbereitung werden aus `package.json` mehrere Pipeline-Variablen ermittelt:

| Variable | Bedeutung |
| --- | --- |
| `releaseNumber` | vollständige Version aus `package.json` |
| `majorMinor` | `major.minor`-Anteil der Version, z. B. `1.4` |
| `isPreRelease` | `true`, wenn die Version einen Pre-Release-Teil enthält |
| `versionLabel` | erster Pre-Release-Bezeichner, z. B. `snapshot` oder `release` |

Zusätzliche Regeln:

- Auf `master`, `main` und `develop` sind nur Pre-Releases erlaubt.
- Auf `master`, `main` und `develop` muss das Pre-Release-Label `snapshot` sein.
- Auf `release/*` ist entweder eine finale Version oder ein Pre-Release mit Label `release` erlaubt.
- Auf `release/*` muss der Branchname exakt zu `release/<major.minor>` passen, z. B. `release/1.4`.

## Ablauf pro Job

1. Repository auschecken
2. `package.json` und `package-lock.json` prüfen
3. Versionsinformationen und Release-Metadaten ermitteln
4. Branch-spezifische Versionsregeln prüfen
5. Lokale Git-Tags bereinigen und Remote-Tags neu laden
6. `npm ci` ausführen
7. Bei Pre-Releases optional die Paketversion lokal mit `$(releaseNumber).$(branchRunCounter)` setzen
8. Alle Einträge aus `npmBuildCmd` nacheinander ausführen
9. Optional `npm publish`
10. Optional Git-Release-Tag erzeugen und pushen
11. Optional Pipeline-Artefakte veröffentlichen
12. Optional Retention für den Pipeline-Lauf setzen

## Parameter

| Parameter | Typ | Standardwert | Bedeutung |
| --- | --- | --- | --- |
| `npmUserConfig` | `string` | `/home/kwazdo/.npmrc` | Pfad zur `.npmrc`, die für `npm ci`, Build und Publish verwendet wird |
| `projectPath` | `string` | `/` | Relativer Projektpfad innerhalb des Repositories |
| `artifactPath` | `object` | `['target/']` | Liste von Verzeichnissen, die als Pipeline-Artefakte veröffentlicht werden können |
| `npmBuildCmd` | `object` | `['run build']` | Liste von npm-Kommandos, die nach `npm ci` ausgeführt werden |
| `publishBuildArtifacts` | `boolean` | `false` | Aktiviert das Publizieren der in `artifactPath` definierten Artefakte |
| `retention` | `boolean` | `true` | Aktiviert das Markieren des Pipeline-Laufs zur Aufbewahrung |
| `releaseRegistryUrl` | `string` | Nexus Release-Registry | Ziel-Registry für `release/*`-Builds |
| `snapshotRegistryUrl` | `string` | Nexus Snapshot-Registry | Ziel-Registry für `master`, `main` und `develop` |
| `tagRelease` | `boolean` | `true` | Aktiviert auf Release-Branches das Erzeugen und Pushen eines annotierten Git-Tags |
| `agentPoolName` | `string` | `Self-hosted Linux (SEU)` | Agent-Pool für alle Jobs |
| `timeoutInMinutes` | `number` | `30` | Maximale Laufzeit pro Job |

## Voraussetzungen

- Das Projekt enthält `package.json` und `package-lock.json`.
- Auf dem Build-Agent sind `node`, `npm` und `git` verfügbar.
- Die konfigurierte `.npmrc` erlaubt Paketinstallation und, falls aktiviert, Publish in die Ziel-Registry.
- Für das Pushen von Git-Tags müssen ausreichende Schreibrechte auf das Repository vorhanden sein.
- Für Release-Branches muss das Benennungsschema `release/<major.minor>` eingehalten werden.

## Einbindungsbeispiel

```yaml
jobs:
  - template: jobs/NpmBuildAndVersionJobTmpl.yml
    parameters:
      projectPath: /frontend
      npmUserConfig: /home/agent/.npmrc
      npmBuildCmd:
        - run lint
        - run test
        - run build
      artifactPath:
        - dist/
      publishBuildArtifacts: true
      retention: true
      tagRelease: true
      timeoutInMinutes: 45
```

## Hinweise

- `build_other_branch_job` führt bewusst kein npm-Publish und kein Artefakt-Publish aus.
- Das Git-Tag entspricht der finalen `version` aus der `package.json`.
- Existiert das Tag bereits auf demselben Commit, wird kein weiterer Push ausgeführt.
