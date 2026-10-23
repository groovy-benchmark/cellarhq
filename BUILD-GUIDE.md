# Build Guide

This guide explains how to build CellarHQ from source on any machine (Linux, macOS or Windows, x86_64 or ARM64).

## TL;DR

```bash
# 1. Make sure JDK 11 is the active Java (see "Installing JDK 11" below)
java -version          # must print 11.x

# 2. Build (skipping tests)
./gradlew clean build -x test        # Linux / macOS
gradlew.bat clean build -x test      # Windows (cmd / PowerShell)

# 3. Result
ls web/build/libs/web.jar web/build/distributions/
```

No database, Docker or network service (other than the dependency repositories) is needed to **build** the project.

## 1. Project layout and stack

The project is a multi-module Gradle build:

| Module   | Build file          | Contents                                                                 |
|----------|---------------------|--------------------------------------------------------------------------|
| (root)   | `cellarhq.gradle`   | Aggregator; defines the `stage` task (used by the `Procfile`)            |
| `model`  | `model/model.gradle`| Database model: jOOQ classes (pre-generated in `model/src/main/generated`) and Liquibase migrations |
| `web`    | `web/web.gradle`    | Ratpack web application (Groovy)                                         |

The build file names are non-standard (`cellarhq.gradle`, `model.gradle`, `web.gradle`); `settings.gradle` wires them up, so no extra flags are needed.

| Component    | Version                           | Notes                                                    |
|--------------|-----------------------------------|----------------------------------------------------------|
| Java (JDK)   | **11**                            | `sourceCompatibility = 11`; see note below               |
| Gradle       | 6.1.1 (via the included wrapper)  | Do not use a system-wide Gradle install                  |
| Groovy       | 2.5.x (resolved by Gradle)        | No separate install needed                               |
| Ratpack      | 1.7.6                             |                                                          |
| jOOQ         | 3.12.x                            | Generated sources are committed; code generation is off  |
| Database     | PostgreSQL                        | Runtime only, not needed for the build                   |

**About the Java version:** the code targets Java 11, and Gradle 6.1.1 can only run on JDK 8 to 13. So the JDK running Gradle must be **11, 12 or 13**. Use JDK 11 (LTS); JDK 17 and 21 will not work.

## 2. Prerequisites

1. **JDK 11.** A full JDK is required, a JRE is not enough. Any vendor works (Eclipse Temurin, Azul Zulu, Amazon Corretto, Microsoft Build of OpenJDK, OpenJDK from your OS package manager, etc.).
2. **Internet access** (at least for the first build) to:
   - `services.gradle.org` (Gradle 6.1.1 distribution, only once)
   - `plugins.gradle.org` (Gradle plugins)
   - `jcenter.bintray.com` (JCenter is read-only now but still serves artifacts)
   - `repo.maven.apache.org` (Maven Central)
   - `web/web.gradle` also declares `http://repository.openmindonline.it/`. That server no longer exists, but nothing is resolved from it any more, so the build works without it.
3. **About 1 GB of free disk space** for Gradle and the dependency cache.
4. **Git**, to clone the repository.
5. **Optional, to run the app:** Docker with Docker Compose (for PostgreSQL), or any PostgreSQL reachable on `localhost:5432`.

You do **not** need Gradle or Groovy installed.

## 3. Installing JDK 11

Pick the option for your platform. After installing, open a **new** terminal and check with `java -version` (it must report `11.x`).

### Any platform: SDKMAN! (Linux, macOS, WSL)

```bash
curl -s "https://get.sdkman.io" | bash
sdk list java | grep -E "11\.[0-9.]+"       # pick an available JDK 11 identifier
sdk install java <identifier>               # e.g. 11.x.x-tem
sdk use java <identifier>
```

### Linux

- **Debian / Ubuntu:** `sudo apt install openjdk-11-jdk` (on newer releases where it is not packaged, use SDKMAN! or a Temurin tarball).
- **Fedora / RHEL / CentOS:** `sudo dnf install java-11-openjdk-devel`
- **Arch:** `sudo pacman -S jdk11-openjdk`
- **Manual tarball:** download a JDK 11 `.tar.gz` for your architecture (x64 or aarch64) from <https://adoptium.net>, extract it anywhere and point `JAVA_HOME` to the extracted folder.

If several JDKs are installed, select JDK 11 with `sudo update-alternatives --config java` (Debian/Ubuntu), `sudo alternatives --config java` (Fedora/RHEL), or simply set `JAVA_HOME` for the build (see below).

### macOS (Intel and Apple Silicon)

```bash
brew install --cask temurin@11        # or: brew install --cask zulu@11
/usr/libexec/java_home -V             # list installed JDKs
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
```

### Windows

- **winget:** `winget install EclipseAdoptium.Temurin.11.JDK`
- **Chocolatey:** `choco install temurin11`
- **Manual:** download the JDK 11 `.msi` from <https://adoptium.net> and enable the "Set JAVA_HOME" option in the installer.

### Using JDK 11 only for this build

You don't have to make JDK 11 your default Java. Setting `JAVA_HOME` in the current terminal is enough, because the Gradle wrapper uses it:

```bash
# Linux / macOS
export JAVA_HOME=/path/to/jdk11
export PATH="$JAVA_HOME/bin:$PATH"
```

```powershell
# Windows PowerShell
$env:JAVA_HOME = "C:\path\to\jdk11"
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
```

```bat
:: Windows cmd
set JAVA_HOME=C:\path\to\jdk11
set PATH=%JAVA_HOME%\bin;%PATH%
```

## 4. Building

From the project root:

```bash
# Linux / macOS (if needed, make the wrapper executable first: chmod +x gradlew)
./gradlew clean build -x test
```

```bat
:: Windows
gradlew.bat clean build -x test
```

The first build downloads Gradle 6.1.1 and all dependencies, which takes a few minutes. Later builds take well under a minute.

**Expected output:**

```
> Task :model:compileGroovy
...
> Task :web:compileGroovy
...
> Task :web:build

BUILD SUCCESSFUL
```

**Artifacts:**

| File                                   | Description                                                        |
|----------------------------------------|--------------------------------------------------------------------|
| `model/build/libs/model.jar`           | Model module classes                                               |
| `web/build/libs/web.jar`               | Web application classes (not a fat jar)                            |
| `web/build/distributions/web.zip` / `web.tar` | **Runnable distribution**: `bin/web` start scripts plus all libraries in `lib/` |

To produce an unpacked runnable distribution in `web/build/install/web/` (this is what the `Procfile` uses):

```bash
./gradlew stage            # same as ./gradlew :web:installDist
```

### Useful Gradle tasks

| Command                          | What it does                                                 |
|----------------------------------|--------------------------------------------------------------|
| `./gradlew clean build -x test`  | Full build without running tests (recommended)               |
| `./gradlew stage`                | Build the runnable distribution in `web/build/install/web/`  |
| `./gradlew :web:compileGroovy`   | Compile the web module only (fast compile check)             |
| `./gradlew update`               | Apply Liquibase database migrations (needs PostgreSQL)       |
| `./gradlew :web:run`             | Run the application from sources (needs PostgreSQL)          |
| `./gradlew dependencies`         | Print the resolved dependency tree                           |

## 5. Running the application

The application needs PostgreSQL on `localhost:5432` with database, user and password `cellarhq` (see `web/src/ratpack/db.yaml`).

```bash
# 1. Start PostgreSQL (Docker)
docker compose up -d db          # or: docker-compose up -d db

# 2. Create the database schema (Liquibase migrations)
./gradlew update

# 3. Run the app
./gradlew :web:run
# or, from the distribution:
./gradlew stage
web/build/install/web/bin/web            # Windows: web\build\install\web\bin\web.bat
```

The application listens on <http://localhost:5050>. See `README.md` for the optional environment variables (Twitter and AWS keys).

## 6. Tests

```bash
./gradlew test
```

At the time of writing this command succeeds but **does not execute any test**: the tests use Spock 2 (`spock-core:2.0-M1`), which runs on the JUnit Platform, and `web/web.gradle` does not enable it (`test { useJUnitPlatform() }`). Gradle therefore looks for JUnit 4 tests, finds none, and reports success. The test sources still compile.

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Could not find or load main class org.gradle.wrapper.GradleWrapperMain` | `gradle/wrapper/gradle-wrapper.jar` is missing | Make sure your checkout contains `gradle/wrapper/gradle-wrapper.jar`, then pull or re-clone. |
| `Could not determine java version from '17.x'` / `Unsupported class file major version` or other errors at Gradle startup | Gradle 6.1.1 is running on a JDK newer than 13 | Set `JAVA_HOME` to JDK 11 (section 3). Check with `./gradlew -version`, which prints the JVM being used. |
| `invalid source release: 11` / `invalid target release: 11` | Gradle is running on JDK 8 | Set `JAVA_HOME` to JDK 11. |
| `JAVA_HOME is set to an invalid directory` | `JAVA_HOME` points to a non-existent folder or to a JRE | Point it to the root folder of a JDK 11 (the folder that contains `bin/javac`). |
| `Could not resolve org.jfrog.jade.plugins...` / `Could not GET 'http://repository.openmindonline.it/...'` | Old `web/web.gradle` | Pull the latest version of the repository (see section 8). |
| `Could not resolve all dependencies ...` / connection timeouts | No access to JCenter, Maven Central, the Gradle Plugin Portal or `services.gradle.org` | Check internet / firewall. Behind a proxy, add `systemProp.https.proxyHost`, `systemProp.https.proxyPort` (and the `http.` variants) to `~/.gradle/gradle.properties`. |
| `FileNotFoundException: ~/.gradle/wrapper/dists/gradle-6.1.1-bin/.../gradle-6.1.1-bin.zip` | Corrupted or partial Gradle distribution in the local cache | Delete `~/.gradle/wrapper/dists/gradle-6.1.1-bin` (Windows: `%USERPROFILE%\.gradle\wrapper\dists\gradle-6.1.1-bin`) and build again. |
| `PKIX path building failed` / SSL handshake errors while downloading | Outdated CA certificates in an old JDK 11 update, or a corporate TLS proxy | Use a recent JDK 11 update, or import your corporate CA certificate into the JDK `cacerts`. |
| `Permission denied: ./gradlew` | The execute bit was lost (e.g. ZIP download, Windows checkout) | `chmod +x gradlew` |
| `/bin/sh^M: bad interpreter` | `gradlew` was checked out with Windows (CRLF) line endings | `git config core.autocrlf input` and re-checkout, or run `sed -i 's/\r$//' gradlew`. |
| `Connection to localhost:5432 refused` (`update`, `:web:run`) | PostgreSQL is not running | Start it (section 5). |
| `java.lang.OutOfMemoryError` during compilation | Not enough heap for the Groovy compiler | Add `org.gradle.jvmargs=-Xmx1g` to `gradle.properties` (project root or `~/.gradle`). |
| `Address already in use` on port 5050 | Another process is using port 5050 | Stop that process, or set the `RATPACK_PORT` environment variable to another port. |

## 8. Build-related changes made to this repository

These are the only changes made so the project builds again. Nothing else was touched, and no source file was changed.

1. **Added `gradle/wrapper/gradle-wrapper.jar`** (the official Gradle 6.1.1 wrapper, SHA-256 `96f793a18e056c23ffeec67c1f3bb8eccff5a4a407fc9ceac183527e7eedf4b6`, matching <https://services.gradle.org/distributions/gradle-6.1.1-wrapper.jar.sha256>). It was missing, so `./gradlew` could not start.
2. **`web/web.gradle`: commented out the `org.jfrog.jade.plugins:jade-plugin-parent:1.4-alpha1` (`ext: 'pom'`) dependency.** It is a parent POM with no code, unused by the project, and it only existed in `repository.openmindonline.it`, which no longer exists.
3. **`web/web.gradle`: excluded `org.jfrog.jade.plugins.common` from `org.reflections:reflections-maven:0.9.9-RC2`.** `reflections-maven` depends on `jade-plugin-common:1.3.8`, which is also gone from every public repository and is only used when `reflections-maven` runs as a Maven plugin. The library the code actually uses, `org.reflections:reflections:0.9.9-RC2` (`org.reflections.Reflections` in `InputSanitizingListener`), is still pulled in and is included in the distribution.
