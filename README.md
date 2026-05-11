# NpmBuildAndVersionJobTmpl

Die Template-Datei `NpmBuildAndVersionJobTmpl.yml` definiert eine Azure-DevOps-Jobvorlage für npm-Projekte. Sie baut das Projekt branch-abhängig, prüft Versionsregeln, veröffentlicht optional npm-Pakete und kann Build-Artefakte sowie Git-Tags erzeugen. Der Default-Branch wird dabei zur Laufzeit aus Azure DevOps gelesen. Die gemeinsame Job-Struktur der drei Branch-Fälle ist in `jobs/templates/jobsNpmBuild.yml` gekapselt.

## Zweck

Die Vorlage standardisiert folgende Abläufe:

- Checkout des Repositories inklusive Submodules
- Prüfung auf `package.json` und `package-lock.json`
- Prüfung auf einen Scoped Package Name in `package.json.name` im Format `@scope/name`
- Prüfung, dass der verwendete Scope in der erlaubten Liste enthalten ist
- Ermittlung der Paketversion aus `package.json`
- Validierung von Versions- und Branch-Regeln
- Ausführung von `npm ci`
- Ausführung konfigurierbarer Build-Kommandos
- Optionales `npm publish` mit Scoped Registry-Konfiguration
- Optionales Publizieren von Pipeline-Artefakten
- Optionales Pushen eines Git-Release-Tags
- Optionales Setzen von Pipeline-Retention

## Interne Struktur

- `jobs/NpmBuildAndVersionJobTmpl.yml` ermittelt zuerst den Default-Branch aus Azure DevOps und orchestriert danach die drei Branch-Fälle `default`, `release` und `other`.
- `jobs/templates/jobsNpmBuild.yml` kapselt die gemeinsame Job-Definition mit Pool, Timeout, Checkout, Prepare-Step, optionaler Branch-Validierung und Build-/Publish-Step.
- `jobs/templates/stepsPrepare.yml`, `jobs/templates/stepsValidateDefaultBranch.yml`, `jobs/templates/stepsValidateReleaseBranch.yml` und `jobs/templates/stepsNpmBuildPublish.yml` enthalten die wiederverwendeten Schrittfolgen.

## Branch-Verhalten

Die Vorlage erzeugt einen Vorab-Job und drei Build-Jobs. Abhängig vom aktuellen Branch läuft nach der Default-Branch-Ermittlung genau einer der drei Build-Jobs:

| Job | Branch-Bedingung | Verhalten |
| --- | --- | --- |
| `resolve_default_branch` | immer | Liest den konfigurierten Default-Branch des aktuellen Azure-DevOps-Repositories |
| `build_default_branch_job` | exakt der in Azure DevOps konfigurierte Default-Branch | Erlaubt nur Pre-Releases mit Label `snapshot`, publiziert in die Snapshot-Registry |
| `build_release_branch_job` | `release/*`, sofern nicht zugleich Default-Branch | Erlaubt finale Versionen oder Pre-Releases mit Label `release`, publiziert in die Release-Registry, kann Git-Tag pushen |
| `build_other_branch_job` | alle übrigen Branches | Baut das Projekt ohne npm-Publish und ohne Artefakt-Publish |

## Versionslogik

Während der Vorbereitung werden aus `package.json` mehrere Pipeline-Variablen ermittelt:

| Variable | Bedeutung |
| --- | --- |
| `packageName` | Paketname aus `package.json`, muss im Format `@scope/name` vorliegen |
| `packageScope` | Scope-Anteil aus `package.json.name`, z. B. `@acme` |
| `releaseNumber` | vollständige Version aus `package.json` |
| `majorMinor` | `major.minor`-Anteil der Version, z. B. `1.4` |
| `isPreRelease` | `true`, wenn die Version einen Pre-Release-Teil enthält |
| `versionLabel` | erster Pre-Release-Bezeichner, z. B. `snapshot` oder `release` |

Zusätzliche Regeln:

- `package.json.name` muss als Scoped Package Name im Format `@scope/name` gesetzt sein.
- Der Scope aus `package.json.name` muss einer der folgenden Werte sein: `bshweb`, `diamant`, `dida`, `idefx`, `kbn`, `shg`.
- Auf dem in Azure DevOps konfigurierten Default-Branch sind nur Pre-Releases erlaubt.
- Auf dem in Azure DevOps konfigurierten Default-Branch muss das Pre-Release-Label `snapshot` sein.
- Auf `release/*` ist entweder eine finale Version oder ein Pre-Release mit Label `release` erlaubt.
- Auf `release/*` muss der Branchname exakt zu `release/<major.minor>` passen, z. B. `release/1.4`.

## Ablauf der Build-Jobs

1. Repository auschecken
2. `package.json` und `package-lock.json` prüfen
3. Scoped Package Name aus `package.json.name` prüfen, erlaubten Scope validieren und Scope ableiten
4. Versionsinformationen und Release-Metadaten ermitteln
5. Branch-spezifische Versionsregeln prüfen
6. Lokale Git-Tags bereinigen und Remote-Tags neu laden
7. `npm ci` ausführen
8. Bei Pre-Releases optional die Paketversion lokal mit `$(releaseNumber).$(branchRunCounter)` setzen
9. Alle Einträge aus `npmBuildCmd` nacheinander ausführen
10. Optional `npm publish` mit `--registry` und zusätzlicher Scoped-Registry-Zuordnung `--@scope:registry`
11. Optional Git-Release-Tag erzeugen und pushen
12. Optional Pipeline-Artefakte veröffentlichen
13. Optional Retention für den Pipeline-Lauf setzen

## Parameter

| Parameter | Typ | Standardwert | Bedeutung |
| --- | --- | --- | --- |
| `npmUserConfig` | `string` | `/home/kwazdo/.npmrc` | Pfad zur `.npmrc`, die für `npm ci`, Build und Publish verwendet wird |
| `projectPath` | `string` | `.` | Relativer Projektpfad innerhalb des Repositories; `.` steht fuer das Repo-Root, fuehrendes oder folgendes `/` sowie `./` wird normalisiert |
| `artifactPath` | `object` | `['target/']` | Liste von Verzeichnissen, die als Pipeline-Artefakte veröffentlicht werden können; der Artefaktname wird aus dem Pfad abgeleitet |
| `npmBuildCmd` | `object` | `['run build']` | Liste von npm-Kommandos, die nach `npm ci` ausgeführt werden |
| `publishBuildArtifacts` | `boolean` | `false` | Aktiviert das Publizieren der in `artifactPath` definierten Artefakte |
| `retention` | `boolean` | `true` | Aktiviert das Markieren des Pipeline-Laufs zur Aufbewahrung |
| `releaseRegistryUrl` | `string` | Nexus Release-Registry | Ziel-Registry für `release/*`-Builds |
| `snapshotRegistryUrl` | `string` | Nexus Snapshot-Registry | Ziel-Registry für den in Azure DevOps konfigurierten Default-Branch |
| `tagRelease` | `boolean` | `true` | Aktiviert auf Release-Branches das Erzeugen und Pushen eines annotierten Git-Tags |
| `agentPoolName` | `string` | `Self-hosted Linux (SEU)` | Agent-Pool für alle Jobs |
| `poolRequirements` | `object` | `['NODE_VERSION -equals 24']` | Liste von Azure-DevOps-Demands, die unverändert an den Pool übergeben werden |
| `timeoutInMinutes` | `number` | `30` | Maximale Laufzeit pro Job |

## Voraussetzungen

- Das Projekt enthält `package.json` und `package-lock.json`.
- `package.json.name` ist als Scoped Package Name im Format `@scope/name` gesetzt.
- Der Scope aus `package.json.name` ist einer der erlaubten Werte `bshweb`, `diamant`, `dida`, `idefx`, `kbn`, `shg`.
- Auf dem Build-Agent sind `node`, `npm` und `git` verfügbar.
- Das Pipeline-Skript kann per `System.AccessToken` auf die Azure-DevOps-REST-API des aktuellen Repositories zugreifen.
- Die konfigurierte `.npmrc` erlaubt Paketinstallation und, falls aktiviert, Publish in die Ziel-Registry fuer den verwendeten Scope.
- Für das Pushen von Git-Tags müssen ausreichende Schreibrechte auf das Repository vorhanden sein.
- Für Release-Branches muss das Benennungsschema `release/<major.minor>` eingehalten werden.

## Einbindungsbeispiel

```yaml
jobs:
  - template: jobs/NpmBuildAndVersionJobTmpl.yml
    parameters:
      projectPath: frontend
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
      poolRequirements:
        - NODE_VERSION -equals 24
      timeoutInMinutes: 45
```

## Hinweise

- `build_other_branch_job` führt bewusst kein npm-Publish und kein Artefakt-Publish aus.
- Das Git-Tag entspricht der finalen `version` aus der `package.json`.
- Existiert das Tag bereits auf demselben Commit, wird kein weiterer Push ausgeführt.
- `projectPath` wird vor der Verwendung normalisiert. `frontend`, `./frontend` und `/frontend` verweisen auf dasselbe Projektverzeichnis.
- `poolRequirements` erwartet vollständige Azure-DevOps-Demand-Ausdrücke wie `NODE_VERSION -equals 24`.
- Für `artifactPath` werden bei der Veröffentlichung die Pfadtrenner aus dem Namen entfernt, z. B. wird `dist/` zu `dist` und `build/output/` zu `build-output`.
