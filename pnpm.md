```markdown
## Anpassung der npm Pipeline Templates für pnpm

### Ausgangslage

Die bestehenden Azure-DevOps-Templates sind aktuell auf npm ausgelegt. Zentrale Annahmen sind:

- `package.json` als Paketmanifest
- `package-lock.json` als Lockfile
- Installation per `npm ci`
- Versionierung per `npm version --no-git-tag-version --ignore-scripts`
- Build-Kommandos per `npm <command>`
- Publish per `npm publish`

Für pnpm bleibt `package.json` weiterhin relevant, das Lockfile und die CLI-Kommandos müssen jedoch angepasst werden.

---

## Ziel

Die Templates sollen optional pnpm als Package Manager unterstützen, ohne bestehende npm-Nutzer zu brechen.

Empfohlener Ansatz:

```yaml
packageManager: npm | pnpm
```

Default bleibt `npm`, damit bestehende Pipelines kompatibel bleiben.

---

## Notwendige Anpassungen

### 1. Neuer Template-Parameter

In den Wrapper- und Kern-Templates einen Parameter ergänzen:

```yaml
- name: packageManager
  type: string
  default: npm
  values:
    - npm
    - pnpm
```

Der Parameter muss durchgereicht werden über:

- `NpmBuildAndVersionJobTmpl.yml`
- `AngularLibraryBuildAndVersionJobTmpl.yml`
- `jobs/templates/jobsNpmBuild.yml`
- `jobs/templates/jobsPackageBuild.yml`
- `jobs/templates/stepsPrepare.yml`
- `jobs/templates/stepsNpmBuildPublish.yml`

Optional sollte `jobsNpmBuild.yml` perspektivisch umbenannt werden, z. B. in `jobsNodePackageBuild.yml`.

---

### 2. Lockfile-Prüfung anpassen

Aktuell wird fest `package-lock.json` geprüft.

Für npm:

```text
package-lock.json
```

Für pnpm:

```text
pnpm-lock.yaml
```

Die Prüfung sollte abhängig vom `packageManager` erfolgen:

```yaml
npm  -> package-lock.json im workspacePath
pnpm -> pnpm-lock.yaml im workspacePath
```

`package.json` bleibt unverändert Pflicht im `packagePath`.

---

### 3. pnpm bereitstellen

Vor Installation/Build muss pnpm auf dem Agent verfügbar sein.

Empfohlen:

```bash
corepack enable pnpm
```

Zusätzlich sollte in pnpm-Projekten das `packageManager`-Feld in der `package.json` gesetzt sein, z. B.:

```json
{
  "packageManager": "pnpm@11.0.0"
}
```

Damit wird die pnpm-Version reproduzierbar.

---

### 4. Install-Schritt ersetzen

Aktuell:

```bash
npm ci
```

Für pnpm:

```bash
pnpm install --frozen-lockfile
```

Alternativ bei pnpm 11:

```bash
pnpm ci
```

Empfehlung: zunächst `pnpm install --frozen-lockfile` verwenden, da es breit bekannt und stabil ist.

---

### 5. Versionierung anpassen

Aktuell:

```bash
npm version "$(releaseNumber).$(branchRunCounter)" \
  --no-git-tag-version \
  --ignore-scripts
```

Für pnpm wäre naheliegend:

```bash
pnpm version "$(releaseNumber).$(branchRunCounter)" \
  --no-git-tag-version \
  --no-git-checks
```

Wichtiger Punkt: Für pnpm ist kein direktes Äquivalent zu `--ignore-scripts` dokumentiert.

Empfehlung: Für pnpm die Version nicht per `pnpm version`, sondern per Node-Skript direkt in `package.json` setzen. Dadurch bleibt das aktuelle Sicherheitsverhalten erhalten, dass `preversion`, `version` und `postversion` nicht ausgeführt werden.

---

### 6. Build-Kommandos anpassen

Aktuell werden Build-Kommandos als npm-Argumente interpretiert:

```yaml
npmBuildCommands:
  - run build
  - test
```

Ausführung:

```bash
npm run build
npm test
```

Für pnpm:

```bash
pnpm run build
pnpm test
```

Kurzfristig kann der Parameter `npmBuildCommands` weiterverwendet werden, um Breaking Changes zu vermeiden.

Langfristig besser:

```yaml
buildCommands:
  - run build
  - test
```

---

### 7. Publish-Schritt anpassen

Aktuell:

```bash
npm publish \
  --tag="..." \
  --registry="..." \
  "--${PACKAGE_SCOPE}:registry=..."
```

Für pnpm:

```bash
pnpm publish \
  --tag="..." \
  --registry="..." \
  --no-git-checks
```

Die Authentifizierung kann weiterhin über `.npmrc` bzw. `NPM_CONFIG_USERCONFIG` erfolgen. Für pnpm sollte geprüft werden, ob die bestehende Registry-/Token-Konfiguration unverändert funktioniert.

---

### 8. Dokumentation aktualisieren

Anzupassen sind:

- `README.md`
- Ablaufdiagramme:
    - `NpmBuildAndVersionJobTmpl.puml`
    - `AngularLibraryBuildAndVersionJobTmpl.puml`

In der Dokumentation sollten beide Varianten beschrieben werden:

```text
npm:
  Lockfile: package-lock.json
  Install: npm ci
  Build: npm <command>
  Publish: npm publish

pnpm:
  Lockfile: pnpm-lock.yaml
  Install: pnpm install --frozen-lockfile
  Build: pnpm <command>
  Publish: pnpm publish
```

---

## Beispielverwendung

```yaml
jobs:
  - template: jobs/NpmBuildAndVersionJobTmpl.yml
    parameters:
      projectPath: frontend
      packageManager: pnpm
      npmUserConfig: /home/agent/.npmrc
      npmBuildCommands:
        - run lint
        - test
        - run build
      artifactPath:
        - dist/
      publishBuildArtifacts: true
      publishDefaultBranchBuild: false
```

---

## Risiken / offene Punkte

- pnpm-Version muss auf den Build-Agenten verfügbar sein oder per Corepack aktiviert werden.
- `pnpm version` bietet kein dokumentiertes Äquivalent zu `npm version --ignore-scripts`.
- Bestehende `.npmrc`-/Registry-Konfiguration muss mit pnpm validiert werden.
- pnpm verwendet eine andere `node_modules`-Struktur; Projekte mit impliziten oder nicht deklarierten Dependencies können dadurch fehlschlagen.
- Angular-/Workspace-Projekte sollten mit `pnpm-lock.yaml` und ggf. `pnpm-workspace.yaml` getestet werden.

---

## Akzeptanzkriterien

- Bestehende npm-Pipelines laufen unverändert weiter.
- Für `packageManager: pnpm` wird `pnpm-lock.yaml` statt `package-lock.json` geprüft.
- Für `packageManager: pnpm` wird die Installation per pnpm ausgeführt.
- Build-Kommandos werden mit pnpm ausgeführt.
- Snapshot-Versionierung funktioniert ohne Ausführung von Lifecycle-Skripten.
- Publish nach Nexus funktioniert mit pnpm.
- README und Ablaufdiagramme dokumentieren npm und pnpm.
```