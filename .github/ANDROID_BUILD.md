# Building the Android ARM64 APK

The **Build Android ARM64 APK** GitHub Actions workflow creates an installable standalone release APK without Android Studio. It runs only when started manually and builds the `arm64-v8a` architecture.

## Run the workflow

1. Open the repository on GitHub and select **Actions**.
2. Select **Build Android ARM64 APK**.
3. Select **Run workflow**, choose the branch to build, and confirm **Run workflow**.
4. Wait for the **Build release APK** job to finish.

The job uses Node.js 22, Java 17, `npm ci --legacy-peer-deps`, and the repository's Gradle wrapper. No Android Studio installation is required.

## Download and verify the APK

Open the completed workflow run and find **Artifacts** near the bottom of its summary page. Download the `YSClaude-v<version>-arm64` artifact. The archive contains files such as:

- `YSClaude-v1.0.9-arm64.apk`
- `YSClaude-v1.0.9-arm64.apk.sha256`

On Linux or macOS, verify the downloaded APK from the directory containing both files:

```sh
sha256sum --check YSClaude-v1.0.9-arm64.apk.sha256
```

On Windows PowerShell, compare the value in the `.sha256` file with:

```powershell
Get-FileHash .\YSClaude-v1.0.9-arm64.apk -Algorithm SHA256
```

## Configure persistent release signing

Android accepts an update only when it is signed with the same key as the installed app. If the signing key changes, users generally must uninstall the existing app before installing the new APK, which removes that app's local data. Back up the keystore and its credentials securely; do not commit them to Git.

When all four signing secrets are absent, the workflow generates a temporary debug keystore so a first build can complete. That key is discarded with the runner, so APKs from different temporary-key runs are not guaranteed to be update-compatible. The workflow warns about this in the job summary. If only some signing secrets are configured, the workflow fails rather than silently using the wrong key.

Create a persistent keystore on a trusted machine with the JDK `keytool` command. Choose and securely record the keystore password, alias, and key password when prompted:

```sh
keytool -genkeypair -v -storetype JKS -keystore ysclaude-release.keystore -alias ysclaude -keyalg RSA -keysize 2048 -validity 10000
```

Encode the keystore as a single Base64 value. For example:

Linux:

```sh
base64 -w 0 ysclaude-release.keystore
```

macOS:

```sh
base64 < ysclaude-release.keystore | tr -d '\n'
```

Windows PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ysclaude-release.keystore"))
```

In the GitHub repository, open **Settings → Secrets and variables → Actions**, select **New repository secret**, and create all four secrets:

| Secret | Value |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | The complete single-line Base64 output for the keystore file |
| `ANDROID_KEYSTORE_PASSWORD` | The keystore password |
| `ANDROID_KEY_ALIAS` | The alias used when the key was created, such as `ysclaude` |
| `ANDROID_KEY_PASSWORD` | The private key password |

The workflow decodes the keystore only into the temporary runner directory and supplies signing credentials to Gradle through protected environment properties. It does not print the values or store the keystore as an artifact.

## Sensitive application data

Treat API keys, tokens, and exported YSClaude app backups as sensitive data. Do not include them in workflow inputs, build artifacts, issue reports, or repository files. Store API credentials using the app's intended secure configuration and keep exported backups in access-controlled storage.
