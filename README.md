# Infernote on Linux 🐉

### Community Installation Guide — Linux Mint 22.3

**Created and tested by Kayn**  
**Status:** Community workaround · **Not official Linux support**

Infernote is a useful tool for tabletop RPG players and Game Masters. This guide documents one **successful installation of the Windows Open Beta on Linux Mint**, using Wine, Bottles, Microsoft's DirectML library, and Edge WebView2.

> [!IMPORTANT]
> This is a **reproducible case report, not a guarantee**. It worked on one Linux Mint machine; other distributions, graphics drivers, display configurations, or Infernote versions may behave differently. This project is not affiliated with, endorsed by, or officially supported by the Infernote developers.

## Tested configuration

| Component | Tested version / setting |
| --- | --- |
| OS | Linux Mint 22.3 “Zena” |
| Desktop / display server | Cinnamon / X11 |
| Infernote | Open Beta v0.1.34 (Windows x64) |
| System Wine | 9.0 (used to extract/install app files) |
| Bottles | 67.4 (Flatpak) |
| Bottles runner | Soda 11.0-10 |
| DirectML | Microsoft.AI.DirectML 1.15.4, **x64 Windows DLL** |
| Edge WebView2 | Evergreen Standalone Installer, x64 |
| Bottles virtual desktop | **Enabled, 1600 × 900** |

## 1. Download the installers

Use the official download sources:

- **Infernote Windows Open Beta:** https://infernote.app/open-beta — tested installer: `Infernote Open Beta v0.1.34.exe`
- **Microsoft DirectML 1.15.4:** https://www.nuget.org/packages/Microsoft.AI.DirectML/1.15.4 — choose **Download package** (`.nupkg`)
- **Microsoft Edge WebView2:** https://developer.microsoft.com/en-us/microsoft-edge/webview2/ — choose **Evergreen Standalone Installer → x64**

The commands below assume downloads are in `~/Downloads`. On a localized system (for example, Hungarian Mint), the directory might be `~/Letöltések`. Adjust the path accordingly. The commands use `$HOME` and `$USER`, so **do not substitute Kayn's or anyone else's username**.

## 2. Install Wine and required utilities

On Linux Mint 22.3:

```bash
sudo apt update
sudo apt install wine-installer winbind unzip
winecfg
```

Allow Wine to initialize its default prefix, then close the configuration window. Some initial Wine warnings may appear.

## 3. Run the Infernote Windows installer with Wine

```bash
wine "$HOME/Downloads/Infernote Open Beta v0.1.34.exe"
```

**What happened in the successful test:** The installer crashed while trying to install Edge WebView2, **but the main Infernote files had already been installed**. If you encounter the same problem, **check for the executable before starting over**:

```bash
ls -l "$HOME/.wine/drive_c/users/$USER/AppData/Local/Infernote/infernote-app.exe"
```

If the file is present, continue. If it is absent, the installer did not extract the application successfully, and this guide's remaining steps cannot fix that by themselves.

## 4. Install Bottles and create a bottle

Install **Bottles (Flatpak)** through Linux Mint's Software Manager. Launch it and create a new bottle with:

- **Name:** `Infernote` (exact spelling; used in commands below)
- **Environment:** Application
- **Runner:** `soda-11.0-10`, if available

You can confirm the Flatpak installation with:

```bash
flatpak info com.usebottles.bottles
```

## 5. Add the missing DirectML DLL

The downloaded `.nupkg` file is a ZIP archive. Extract the tested package:

```bash
mkdir -p "$HOME/DirectML"
unzip -o "$HOME/Downloads/microsoft.ai.directml.1.15.4.nupkg" -d "$HOME/DirectML"
```

**Use `bin/x64-win/DirectML.dll`** — not the x86, ARM, Xbox, or Linux version.

Copy the **entire installed Infernote folder**, including its resources and supporting DLLs, to a convenient location outside the Wine prefix, then copy DirectML beside the executable:

```bash
mkdir -p "$HOME/Applications"
cp -a "$HOME/.wine/drive_c/users/$USER/AppData/Local/Infernote" "$HOME/Applications/"
cp "$HOME/DirectML/bin/x64-win/DirectML.dll" "$HOME/Applications/Infernote/"
```

Check both files:

```bash
ls -l "$HOME/Applications/Infernote/infernote-app.exe" \
      "$HOME/Applications/Infernote/DirectML.dll"
```

> [!WARNING]
> Obtain the DLL from Microsoft's official NuGet package, **not from random DLL download sites**. This workaround copies a Microsoft runtime DLL for local use; review applicable software licenses before redistributing any binaries. **Do not commit Microsoft's DLL, the Infernote installer, or the application files to this repository.**

## 6. Give Bottles access to the application directory

The Flatpak sandbox may block direct access to files in your home directory. Grant access **only** to the application folder's parent directory:

```bash
flatpak override --user --filesystem="$HOME/Applications" com.usebottles.bottles
```

Fully close and reopen Bottles after applying the override.

Alternatively, the launch command in step 9 can grant access **for that run only** using `--filesystem=...`.

## 7. Install Edge WebView2 **inside the Infernote bottle**

1. Open **Bottles → Infernote**.
2. Select **Run Executable**.
3. Choose the **WebView2 Evergreen Standalone Installer x64** downloaded in step 1.
4. Complete the installation **in this bottle**, not in the default `~/.wine` prefix.

To check whether WebView2 files exist in the bottle:

```bash
find "$HOME/.var/app/com.usebottles.bottles/data/bottles/bottles/Infernote/drive_c" \
  -iname 'msedgewebview2.exe' 2>/dev/null
```

In the tested setup, WebView2 appeared under `Program Files (x86)/Microsoft/EdgeWebView/Application/...`. **Finding the file proves it was installed, not necessarily that it runs correctly under Wine.**

## 8. Fix incorrect mouse-click positions

After the UI finally opened on the test machine, mouse clicks landed at an offset from the visible cursor. The working fix was:

1. Open **Bottles → Infernote → Settings**.
2. Turn **Virtual Desktop** **ON**.
3. Set resolution to **1600 × 900**.
4. Stop any existing Infernote processes and relaunch.

This corrected mouse positioning on the tested Cinnamon/X11 system. Different display scaling may require a different resolution or further investigation.

## 9. Launch Infernote

With the persistent Flatpak permission from step 6:

```bash
flatpak run --command=bottles-cli com.usebottles.bottles run \
  -b Infernote -e "$HOME/Applications/Infernote/infernote-app.exe"
```

Or, without a persistent Flatpak override, allow access only for this launch:

```bash
flatpak run --filesystem="$HOME/Applications" \
  --command=bottles-cli com.usebottles.bottles run \
  -b Infernote -e "$HOME/Applications/Infernote/infernote-app.exe"
```

If the executable is still running from an earlier attempt but no window is visible, **stop all processes in the Infernote bottle** before relaunching. This was necessary in the original successful test after WebView2 installation.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `Library directml.dll ... not found` | Ensure the **x64 Windows** `DirectML.dll` is beside `infernote-app.exe` in `~/Applications/Infernote/`. |
| `Executable file path does not exist or is not accessible by the Flatpak` | Confirm the executable exists; grant Bottles filesystem access (step 6). |
| Infernote process runs but no window appears | Check WebView2 is installed **in the same bottle**, stop stuck processes, and relaunch. |
| Mouse clicks do not match cursor location | Enable **Virtual Desktop, 1600 × 900**, then restart. |
| Different versions do not work | Compare versions and graphics settings with the tested configuration; success is not guaranteed. |

To check whether Infernote and WebView2 are running:

```bash
pgrep -afi 'infernote-app.exe|msedgewebview2.exe'
```

Wine logs may contain unrelated warnings (including `lsteamclient disabled`). Do not assume every `err:` line is the cause of a failure.

## Keep these files

- `~/Applications/Infernote/` — executable, resources, and `DirectML.dll`.
- The **Infernote** bottle — WebView2 installation and virtual desktop settings.
- Backups of your campaign data, using Infernote's own export/backup features if available.

The extracted `~/DirectML/` directory is no longer needed for launching the app once the DLL has been copied, but keeping it is harmless.

## Contributing and compatibility reports

This guide currently documents **one confirmed working machine**. Reports from other Linux users are welcome! When opening a GitHub issue, include:

- Linux distribution/version and desktop environment (X11 or Wayland)
- Infernote, Bottles, runner, DirectML, and WebView2 versions
- Graphics hardware/driver, screen resolution, and display scaling
- What worked, what failed, and relevant log excerpts (remove personal paths or tokens)

Please do not upload proprietary installers, DLLs, account credentials, or private campaign data.

## Official resources

- [Infernote Open Beta](https://infernote.app/open-beta)
- [Bottles documentation](https://docs.usebottles.com/)
- [Bottles Flatpak directory access](https://docs.usebottles.com/flatpak/expose-directories)
- [Microsoft DirectML on NuGet](https://www.nuget.org/packages/Microsoft.AI.DirectML/1.15.4)
- [Microsoft Edge WebView2](https://developer.microsoft.com/en-us/microsoft-edge/webview2/)

---

**Created and tested by Kayn** · October 2026  
*Made for the tabletop RPG community, so Linux players and GMs don't have to miss out.*
