# Build And Version Templates

Das Repository enthaelt Azure-DevOps-Jobvorlagen fuer Node-/npm-Pakete und Angular-Libraries. Beide Varianten teilen sich denselben Kern fuer Branch-Validierung, Versionierung, Publish, Artefakte und Retention.

## Verfuegbare Templates

- `jobs/NpmBuildAndVersionJobTmpl.yml`
  - Wrapper fuer klassische npm-Projekte mit einem einzigen `package.json`-Kontext.
  - Verwendet intern denselben Pfad fuer Installation, Paket-Metadaten und Publish.
- `jobs/AngularLibraryBuildAndVersionJobTmpl.yml`
  - Wrapper fuer Angular-Workspaces mit publizierbarer Library.
  - Trennt Workspace, Library-`package.json` und Publish-Verzeichnis.
- `jobs/templates/jobsPackageBuild.yml`
  - Gemeinsamer Kern fuer beide Wrapper.

## Gemeinsames Verhalten

Die Templates standardisieren folgende Ablaufe:

1. Default-Branch aus Azure DevOps ermitteln
2. Repository auschecken
3. Pfade normalisieren und validieren
4. `package.json` und `package-lock.json` pruefen
5. Paketname, Scope und Version aus dem Paketkontext lesen
6. Branch-spezifische Versionsregeln pruefen
7. `npm ci` im Workspace ausfuehren
8. Bei Pre-Releases optional `npm version --no-git-tag-version --ignore-scripts` ausfuehren
9. Build-Kommandos ausfuehren
10. Optional `npm publish` ausfuehren
11. Optional Git-Tag pushen
12. Optional Pipeline-Artefakte publizieren
13. Optional Retention-Lease setzen

## Pfadmodell

Der gemeinsame Kern arbeitet mit drei relativen Pfaden innerhalb des Repositories:

- `workspacePath`
  - Verzeichnis fuer `npm ci` und Build-Kommandos.
- `packagePath`
  - Verzeichnis mit dem massgeblichen `package.json` fuer Name, Scope und Version.
- `publishPath`
  - Verzeichnis, aus dem `npm publish` ausgefuehrt wird.

Alle Pfade muessen relativ zum Repository sein. Absolute Pfade sowie Segmente wie `..` sind unzulaessig.

## Npm-Wrapper

`jobs/NpmBuildAndVersionJobTmpl.yml` behaelt die bisherige API bei.

### Parameter

| Parameter | Typ | Standardwert | Bedeutung |
| --- | --- | --- | --- |
| `npmUserConfig` | `string` | `/home/kwazdo/.npmrc` | Pfad zur `.npmrc` fuer `npm ci`, Build und Publish |
| `projectPath` | `string` | `.` | Relativer Projektpfad; wird intern fuer `workspacePath`, `packagePath` und `publishPath` verwendet |
| `artifactPath` | `object` | `['target/']` | Verzeichnisse fuer Pipeline-Artefakte, relativ zu `projectPath` |
| `buildCommands` | `object` | `['npm run build']` | Shell-Kommandos, die nach `npm ci` ausgefuehrt werden, z. B. `npm run lint` oder `npm test` |
| `publishBuildArtifacts` | `boolean` | `false` | Publiziert die in `artifactPath` konfigurierten Artefakte |
| `retention` | `boolean` | `true` | Setzt auf dem Default-Branch eine Retention-Lease fuer 365 Tage |
| `releaseRetentionDays` | `number` | `36501` | Retention fuer `release/*`-Builds |
| `releaseRegistryUrl` | `string` | Nexus Release-Registry | Ziel-Registry fuer `release/*` |
| `snapshotRegistryUrl` | `string` | Nexus Snapshot-Registry | Ziel-Registry fuer den Default-Branch |
| `tagRelease` | `boolean` | `true` | Pusht auf `release/*` ein Git-Tag |
| `agentPoolName` | `string` | `Self-hosted Linux (SEU)` | Agent-Pool |
| `poolRequirements` | `object` | `['NODE_VERSION -equals 24']` | Azure-DevOps-Demands |
| `timeoutInMinutes` | `number` | `30` | Maximale Laufzeit pro Job |

### Beispiel

```yaml
jobs:
  - template: jobs/NpmBuildAndVersionJobTmpl.yml
    parameters:
      projectPath: frontend
      npmUserConfig: /home/agent/.npmrc
      buildCommands:
        - npm run lint
        - npm test
        - npm run build
      artifactPath:
        - dist/
      publishBuildArtifacts: true
      retention: true
      releaseRetentionDays: 36501
      tagRelease: true
      poolRequirements:
        - NODE_VERSION -equals 24
      timeoutInMinutes: 45
```

## Angular-Library-Wrapper

`jobs/AngularLibraryBuildAndVersionJobTmpl.yml` ist fuer Angular-Workspaces gedacht, bei denen Installation, Library-Metadaten und Publish-Verzeichnis auseinanderfallen.

Typischer Zuschnitt:

- `workspacePath: .`
- `packagePath: projects/my-lib`
- `publishPath: dist/my-lib`

### Parameter

| Parameter | Typ | Standardwert | Bedeutung |
| --- | --- | --- | --- |
| `npmUserConfig` | `string` | `/home/kwazdo/.npmrc` | Pfad zur `.npmrc` fuer `npm ci`, Build und Publish |
| `workspacePath` | `string` | `.` | Angular-Workspace fuer `npm ci` und Build-Kommandos |
| `packagePath` | `string` | keiner | Pfad zur Library mit dem massgeblichen `package.json` |
| `publishPath` | `string` | keiner | Pfad zum Build-Output, aus dem `npm publish` ausgefuehrt wird |
| `buildCommands` | `object` | keiner | Shell-Kommandos, die im Workspace ausgefuehrt werden |
| `artifactPath` | `object` | `['dist/']` | Zu publizierende Artefakte, relativ zu `workspacePath` |
| `publishBuildArtifacts` | `boolean` | `false` | Publiziert die in `artifactPath` konfigurierten Artefakte |
| `retention` | `boolean` | `true` | Setzt auf dem Default-Branch eine Retention-Lease fuer 365 Tage |
| `releaseRetentionDays` | `number` | `36501` | Retention fuer `release/*`-Builds |
| `releaseRegistryUrl` | `string` | Nexus Release-Registry | Ziel-Registry fuer `release/*` |
| `snapshotRegistryUrl` | `string` | Nexus Snapshot-Registry | Ziel-Registry fuer den Default-Branch |
| `tagRelease` | `boolean` | `true` | Pusht auf `release/*` ein Git-Tag |
| `gitTagTemplate` | `string` | `$(packageNameSlug)-$(releaseNumber)` | Tag-Namensschema fuer Release-Tags |
| `agentPoolName` | `string` | `Self-hosted Linux (SEU)` | Agent-Pool |
| `poolRequirements` | `object` | `['NODE_VERSION -equals 24']` | Azure-DevOps-Demands |
| `timeoutInMinutes` | `number` | `30` | Maximale Laufzeit pro Job |

### Beispiel

```yaml
jobs:
  - template: jobs/AngularLibraryBuildAndVersionJobTmpl.yml
    parameters:
      workspacePath: .
      packagePath: projects/my-lib
      publishPath: dist/my-lib
      npmUserConfig: /home/agent/.npmrc
      buildCommands:
        - npx ng build my-lib --configuration production
      artifactPath:
        - dist/my-lib/
      publishBuildArtifacts: true
      tagRelease: true
```

### Angular-Pfadmapping

Fuer Angular-Workspaces ist die Trennung der drei Pfade entscheidend:

- `workspacePath`
  - Ort fuer `npm ci` und Build-Kommandos
  - typischerweise das Repository-Root oder der Angular-Workspace
- `packagePath`
  - Ort des Library-`package.json`
  - typischerweise `projects/<library-name>`
- `publishPath`
  - Ort des von Angular erzeugten Publish-Pakets
  - typischerweise `dist/<library-name>`

Typische Struktur:

```text
repo/
├── package.json
├── package-lock.json
├── angular.json
├── projects/
│   └── my-lib/
│       ├── package.json
│       └── ng-package.json
└── dist/
    └── my-lib/
        └── package.json
```

Dazu passendes Mapping:

```yaml
workspacePath: .
packagePath: projects/my-lib
publishPath: dist/my-lib
```

Typische Build-Kommandos:

```yaml
buildCommands:
  - npx ng build my-lib --configuration production
```

Wenn der Angular-Workspace nicht im Repository-Root liegt, verschieben sich die Pfade entsprechend, zum Beispiel:

```yaml
workspacePath: frontend
packagePath: frontend/projects/my-lib
publishPath: frontend/dist/my-lib
buildCommands:
  - npx ng build my-lib --configuration production
```

## Branch- und Versionsregeln

Fuer beide Wrapper gelten dieselben Regeln:

- `package.json.name` muss als Scoped Package Name im Format `@scope/name` vorliegen.
- Erlaubte Scopes sind `bshweb`, `diamant`, `dida`, `idefx`, `kbn`, `shg`.
- `package.json.version` muss gueltiges SemVer mit numerischem `major.minor.patch` sein.
- Auf dem Azure-DevOps-Default-Branch sind nur Pre-Releases mit Label `snapshot` erlaubt.
- Auf `release/*` sind finale Versionen oder Pre-Releases mit Label `release` erlaubt.
- `release/*` muss mit `release/<major.minor>` beginnen.

## Hinweise

- Der npm-Wrapper verwendet fuer Git-Tags weiterhin standardmaessig nur `$(releaseNumber)`.
- Der Angular-Wrapper verwendet standardmaessig `$(packageNameSlug)-$(releaseNumber)`, damit mehrere Libraries in einem Repository nicht dieselben Tags erzeugen.
- `build_other_branch_job` fuehrt bewusst kein npm-Publish, kein Artefakt-Publish und keine Retention-Lease aus.
- `artifactPath` ist relativ zu `workspacePath`.
- `publishPath` muss nach dem Build ein Verzeichnis mit `package.json` enthalten.
