# maff-open

Open a [Mozilla Archive Format](https://en.wikipedia.org/wiki/Mozilla_Archive_Format) (`.maff`) web-page snapshot in your default browser.[^wiki]

`maff-open` is a single Python script. It reads the archive, extracts the saved page into a cache directory, and hands `index.html` to `xdg-open`. Images, stylesheets, and other files saved beside the page stay on disk, so the browser can load them after the script exits. Opening the same unchanged archive again reuses that copy.

```bash
maff-open page.maff
```

```text
Page title
  saved Thu, 15 Mar 2012 14:49:34 -0700
  from  https://example.com/page
  file  /home/you/.cache/maff-open/9e1a37de1b2f8dad27571514/index.html
```

## Background

MAFF was the archive format of the Mozilla Archive Format Firefox add-on, written by Christopher Ottley and Paolo Amadini and first released in May 2004.[^wiki] The add-on saved one or more web pages, together with the files needed to display them, into a single ZIP archive with the extension `.maff` and the MIME type `application/x-maff`.[^wiki]

The format was designed to be easy to implement and easy to inspect.[^about] Because the container is an ordinary ZIP file, any unzip tool can pull the page back out. Text is compressed. Files that are already compressed, such as video, can be stored without a second layer of compression.[^about] A small RDF/XML record stores the original URL, the page title, the time of the save, the main filename, and a character set.[^spec]

That is a different design from [MHTML](https://en.wikipedia.org/wiki/MHTML) (`.mht`), which encodes every part of the page, including images, as MIME sections inside one file.[^wiki] A MAFF archive keeps each file in its original form.

The add-on worked on Firefox from the Firefox 2 era through 2017. Firefox 57 removed the old extension API, and the add-on was not ported.[^wiki] The specification remains a working draft.[^spec] The last full copies of the project site are on the Wayback Machine:

- [MAFF specification](https://web.archive.org/web/20171103235108/http://maf.mozdev.org/maff-specification.html) (3 November 2017)[^spec]
- [About the MAFF file format](https://web.archive.org/web/20171119061600/http://maf.mozdev.org/maff-file-format.html) (19 November 2017)[^about]

Pages can still be opened by unpacking them, which is what this program does. [WebScrapBook](https://github.com/danny0838/webscrapbook) (and its companion [PyWebScrapBook](https://github.com/danny0838/PyWebScrapBook)) can save and open MAFF in current Firefox and Chromium browsers.[^wiki] [^wsb] Pale Moon can do the same through the [MozArchiver](https://addons.palemoon.org/addon/mozarchiver/) add-on, a fork of the original extension.[^wiki] [^mozarchiver]

A snapshot saved by the add-on is a rendering of the page at that moment. The add-on's snapshot save often replaced scripts with a comment (`Script removed by snapshot save`), inlined computed styles (`Effective stylesheet produced by snapshot save`), and rewrote resources it did not fetch to `urn:not-loaded:` URLs.[^observed] Those pages are for reading. They are a copy of how the page looked, kept in the archive.

## What an archive contains

The ZIP root holds no files. Each saved page is one top-level folder.[^spec] The original add-on named that folder `{milliseconds}_{random}`, for example `1331848174858_885`.[^observed] One archive may contain several pages, such as every open tab.[^about] Most archives contain one.[^observed]

```text
1331848174858_885/
├── index.html          Main document. Links point at local files.
├── index.rdf           RDF/XML metadata. Usual, and optional.
├── index_files/        Images, stylesheets, scripts, and other files.
└── ^metadata^/         Reserved for extra metadata. Rarely present.
```

`index.rdf` uses the namespace `http://maf.mozdev.org/metadata/rdf#`.[^spec] The fields this program reads are stored as `RDF:resource` attributes.[^spec] [^observed]

```xml
<?xml version="1.0"?>
<RDF:RDF xmlns:MAF="http://maf.mozdev.org/metadata/rdf#"
         xmlns:NC="http://home.netscape.com/NC-rdf#"
         xmlns:RDF="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
  <RDF:Description RDF:about="urn:root">
    <MAF:originalurl RDF:resource="https://example.com/page"/>
    <MAF:title RDF:resource="Page title"/>
    <MAF:archivetime RDF:resource="Thu, 15 Mar 2012 14:49:34 -0700"/>
    <MAF:indexfilename RDF:resource="index.html"/>
    <MAF:charset RDF:resource="UTF-8"/>
  </RDF:Description>
</RDF:RDF>
```

| Field | Meaning |
|---|---|
| `originalurl` | Address the page was saved from.[^spec] |
| `title` | Title recorded at save time.[^spec] |
| `archivetime` | Local time of the save, in the format from RFC 5322.[^spec] [^rfc5322] |
| `indexfilename` | Main document, in the same folder as `index.rdf`.[^spec] |
| `charset` | Character set for files that do not declare their own.[^spec] |

The date and the original URL are printed when you open or list an archive. The browser is responsible for interpreting the HTML, including any character set declared inside it.

If `index.rdf` is missing, the main document must be named `index` plus an extension for its type. For HTML, that name is `index.html`.[^spec] This program looks for `index.html`, `index.htm`, `index.xhtml`, and `index.xht`, which are the extensions the specification lists for HTML and XHTML.[^spec]

The directory named `^metadata^` is reserved for extra information such as scroll position and per-file original URLs. The specification treats that directory as an extension, and a reader must still be able to display the page without it.[^spec]

## Environments

The script runs wherever its external programs exist. It needs Python 3.9 or newer and [`xdg-open` from xdg-utils](https://www.freedesktop.org/wiki/Software/xdg-utils/).[^xdg] Those pieces are standard on Linux desktops that follow the [FreeDesktop](https://www.freedesktop.org/) specifications, including GNOME, KDE Plasma, and Xfce. Other Unix-like systems can run it when the same programs are installed. Opening a page needs a graphical session, X11 or Wayland, and a browser registered as the default for HTML. `xdg-open` is what starts that browser.[^xdg]

The file picker and the error dialog used when there is no terminal need [`zenity`](https://wiki.gnome.org/Projects/Zenity), plus `DISPLAY` or `WAYLAND_DISPLAY`. Registering `.maff` with the file manager needs `update-mime-database` from [shared-mime-info](https://www.freedesktop.org/wiki/Software/shared-mime-info/) and `update-desktop-database` from [desktop-file-utils](https://www.freedesktop.org/wiki/Software/desktop-file-utils/).[^mime] [^desktop] The cache directory follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/): `$XDG_CACHE_HOME/maff-open`, or `~/.cache/maff-open` when that variable is unset.[^basedir]

This copy was tried on 26 September 2026 on one Linux machine: Python 3.13.5, an X11 session (`DISPLAY=:0`), xdg-utils 1.2.1, zenity 4.1.90, shared-mime-info 2.4, desktop-file-utils 0.28, and Vivaldi as the default browser.[^tried] The command opened a saved page in that browser. A headless Firefox screenshot of the extracted `index.html` showed the page's saved layout, including its stylesheets and images.[^tried] After the MIME package was installed, `xdg-mime query filetype` reported `application/x-maff` for a `.maff` file, and `xdg-mime query default` reported `maff-open.desktop`.[^tried]

The script has no launcher for Windows or macOS. Both would need a different way to start the default browser and, for double-click, a different way to register the file type.

## Requirements

- Python 3.9 or newer. The script uses only the standard library. It was tried with Python 3.13.5.[^tried]
- [`xdg-open`](https://www.freedesktop.org/wiki/Software/xdg-utils/), which launches the default browser.[^xdg]
- A graphical session for opening pages.[^tried]

These are optional:

| Program | Used for |
|---|---|
| `zenity` | The file picker when you run `maff-open` with no path, and error dialogs when the script is launched without a terminal. |
| `update-mime-database` | Registering `.maff` as `application/x-maff`. From the `shared-mime-info` package.[^mime] |
| `update-desktop-database` and `xdg-mime` | Offering **Open MAFF Archive** in the file manager. From `desktop-file-utils` and `xdg-utils`.[^desktop] [^xdg] |

`xdg-open` chooses whichever browser is already the default for HTML.[^xdg]

## Install

Clone or copy this directory, then install the script onto your `PATH`:

```bash
install -Dm755 maff-open "${HOME}/.local/bin/maff-open"
```

Confirm it runs:

```bash
maff-open --help
```

To open `.maff` files from the file manager, install the desktop entry and the MIME description. The `Exec` line is rewritten to the absolute path of the script so a desktop session can find it.[^desktop]

```bash
bindir="${HOME}/.local/bin"
install -Dm755 maff-open "${bindir}/maff-open"
sed "s|^Exec=.*|Exec=${bindir}/maff-open %F|" maff-open.desktop \
  > "${HOME}/.local/share/applications/maff-open.desktop"
install -Dm644 application-x-maff.xml \
  "${HOME}/.local/share/mime/packages/application-x-maff.xml"
update-mime-database "${HOME}/.local/share/mime"
update-desktop-database "${HOME}/.local/share/applications"
xdg-mime default maff-open.desktop application/x-maff
```

`application/x-maff` is declared as a subclass of `application/zip`. A `.maff` file is still a ZIP, and its contents start with the ZIP signature. When a glob match is the same type as the magic result, or a subclass of it, the Shared MIME-info rules select that more specific type.[^mime] That is what lets the file manager treat `*.maff` as a MAFF archive. On the machine where this was tried, that identification succeeded.[^tried]

Check the result:

```bash
xdg-mime query filetype page.maff          # application/x-maff
xdg-mime query default application/x-maff  # maff-open.desktop
```

Log out and back in if the file manager still offers only an archive manager. An archive manager such as File Roller remains available from **Open With**.

## Usage

Open one or more archives. Each page opens in the default browser.

```bash
maff-open page.maff
maff-open one.maff another.maff
```

With no arguments, in a graphical session, a file picker opens. Canceling the picker exits without an error.

List the saved title, date, and original URL. This reads the archive and does not extract it or open a browser.

```bash
maff-open --list page.maff
```

Delete every extracted copy:

```bash
maff-open --clear
```

From a terminal, errors are printed to standard error. From the file manager, where there is no terminal, the same message is shown with `zenity` when `zenity` is installed.

## How opening works

1. The archive is opened as a ZIP file.
2. Each top-level folder is treated as one saved page.[^spec] `index.rdf` in that folder supplies the title, date, original URL, and main filename when it is present and valid.
3. The page is extracted into `$XDG_CACHE_HOME/maff-open/<key>/`, or `~/.cache/maff-open/<key>/` when `XDG_CACHE_HOME` is unset.[^basedir] The key is derived from the archive's absolute path, size, and modification time.
4. A single page is extracted so that `index.html` sits at the root of that directory and `index_files/` sits beside it. Relative links in the saved page then resolve. An archive with several pages gets one subdirectory per page, using the folder name from the archive.
5. `xdg-open` is called on each main document.[^xdg] The default browser takes over from there.

Extraction is written to a temporary directory in the cache and renamed into place after it succeeds, so a failed extract is not reused. Paths inside the archive that are absolute or contain `..` are rejected. An encrypted archive produces an error.

The cache also stores `.maff-open.json` next to the extracted page. That file records the source path, size, modification time, and the metadata used to reopen the page. It is not part of the saved website.

## Cache

The extracted files have to outlive the script. `xdg-open` returns as soon as the browser has been asked to open the page, and the browser then reads `index.html`, the stylesheets, and the images from disk. Deleting the directory when the script exits would remove those files while the browser is loading them, and a later refresh would fail.

Leaving the copy in the cache also makes a second open of an unchanged archive immediate. The archive is extracted again when its size or modification time changes. Moving or renaming the file changes its path, so that copy is extracted again too. Old copies are removed only by `maff-open --clear`.

## Limitations

Saved pages keep the links they were saved with.

- Links to `index_files/` and other relative files open from the extracted copy.
- Links to `http://` and `https://` addresses go to the network.
- A root-relative link such as `/` refers to the root of the machine when the page is opened from a `file://` URL. Offline snapshots usually cannot satisfy those links.
- `urn:not-loaded:` URLs are resources the original save never fetched.[^observed]
- An empty `<base href="">` is left as saved.[^observed] A `<base href>` that points at another site would send relative links there.

JavaScript that remains in the page runs in the browser, under the same rules as any other local HTML file. Scripts the add-on removed during the snapshot save stay removed.[^observed]

The program reads the basic MAFF layout: a ZIP file, one folder per page, and a main document.[^spec] It leaves the optional `^metadata^` directory unused. The specification allows that directory to hold scroll position, zoom, and per-file original URLs, and it still requires the page to display without them.[^spec] The character set stored in `index.rdf` is kept in the metadata and is not applied separately from whatever the HTML file itself declares.

Calibre has no MAFF reader in its own viewer. On the machine where this program was written, Calibre's configuration had no MAFF viewer set.[^tried] Open the `.maff` file from the file manager, or pass its path to `maff-open`.

## Uninstall

```bash
maff-open --clear
rm -f "${HOME}/.local/bin/maff-open" \
      "${HOME}/.local/share/applications/maff-open.desktop" \
      "${HOME}/.local/share/mime/packages/application-x-maff.xml"
update-mime-database "${HOME}/.local/share/mime"
update-desktop-database "${HOME}/.local/share/applications"
```

Remove the `application/x-maff` line from `~/.config/mimeapps.list` if it is still listed there after removing the desktop file. That file is where per-user default applications are stored.[^mimeapps]

## Acknowledgements

| Contributor | Role |
| --- | --- |
| Scott Reynolds | Direction, review, and testing |
| [Grok 4.7](https://x.ai/) (xAI) | Implementation and documentation |

September 2026. Written by Grok at Scott Reynolds's direction. Reviewed and accepted by Scott Reynolds.

## License

`maff-open` is released under the MIT License. Copyright (c) 2026 M. Scott Reynolds. The full text is in [LICENSE](LICENSE).

## Notes

[^wiki]: Wikipedia contributors, "Mozilla Archive Format," *Wikipedia, The Free Encyclopedia*, https://en.wikipedia.org/wiki/Mozilla_Archive_Format, retrieved 26 September 2026. This article is the source for the add-on's authors and May 2004 date, the `.maff` extension and `application/x-maff` media type, the comparison with MHTML, support ending after Firefox 57 in 2017, and the mentions of WebScrapBook and MozArchiver.
[^about]: "About the MAFF file format," Mozilla Archive Format project, archived by the Internet Archive on 19 November 2017, https://web.archive.org/web/20171119061600/http://maf.mozdev.org/maff-file-format.html, retrieved 26 September 2026. This page is the source for the format's goals, ZIP storage, selective compression, saving several pages in one archive, and storing the original URL and save time.
[^spec]: "The MAFF specification," Mozilla Archive Format project, working draft archived by the Internet Archive on 3 November 2017, https://web.archive.org/web/20171103235108/http://maf.mozdev.org/maff-specification.html, retrieved 26 September 2026. This draft is the source for the one-folder-per-page rule, the empty archive root, `index.rdf` and its fields, the `http://maf.mozdev.org/metadata/rdf#` namespace, the RFC 5322 date requirement, the `index.html` fallback, the HTML and XHTML extensions, and the reserved `^metadata^` directory.
[^rfc5322]: P. Resnick, ed., "Internet Message Format," RFC 5322, section 3.3, October 2008, https://www.rfc-editor.org/rfc/rfc5322#section-3.3. The MAFF specification requires this date format.[^spec]
[^wsb]: Danny Lin, WebScrapBook, https://github.com/danny0838/webscrapbook, and PyWebScrapBook, https://github.com/danny0838/PyWebScrapBook, retrieved 26 September 2026.
[^mozarchiver]: Pale Moon Add-ons, "MozArchiver," https://addons.palemoon.org/addon/mozarchiver/, retrieved 26 September 2026. The page describes it as a fork of the Mozilla Archive Format extension by Christopher Ottley and Paolo Amadini.
[^observed]: Observed directly in `.maff` archives saved by the add-on, while this program was written in September 2026. Ninety-four archives were inspected. Each had one page folder, `index.html`, and `index.rdf`. Folder names had the form `{milliseconds}_{random}` (one example was `1331848174858_885`). Saved HTML contained the comments `Script removed by snapshot save` and `Effective stylesheet produced by snapshot save`, `urn:not-loaded:` URLs, and, in ten archives, an empty `<base href="">`. The `index.rdf` files stored their values in `RDF:resource` attributes.
[^xdg]: freedesktop.org, "xdg-utils," https://www.freedesktop.org/wiki/Software/xdg-utils/, retrieved 26 September 2026. `xdg-open` and `xdg-mime` come from this package. The installed version on the test machine was 1.2.1.
[^mime]: Thomas Leonard and others, "Shared MIME-info Database," freedesktop.org, https://specifications.freedesktop.org/shared-mime-info/latest-single/, retrieved 26 September 2026. The matching rules say that if a glob match is equal to the type found by magic, or a subclass of it, that type is used. `application-x-maff.xml` in this project declares `application/x-maff` as a subclass of `application/zip` with the globs `*.maff` and `*.MAFF`.
[^desktop]: "Desktop Entry Specification," freedesktop.org, https://specifications.freedesktop.org/desktop-entry/latest/, retrieved 26 September 2026. `maff-open.desktop` is a desktop entry of type `Application`. `update-desktop-database` comes from desktop-file-utils.
[^basedir]: "XDG Base Directory Specification," freedesktop.org, https://specifications.freedesktop.org/basedir/latest/, retrieved 26 September 2026. It defines `$XDG_CACHE_HOME`, with `~/.cache` as the default.
[^mimeapps]: "Association between MIME types and applications," freedesktop.org, https://specifications.freedesktop.org/mime-apps/latest/, retrieved 26 September 2026. Per-user defaults are stored in `$XDG_CONFIG_HOME/mimeapps.list`, normally `~/.config/mimeapps.list`.
[^tried]: Checked on this computer on 26 September 2026. The system was Linux, with Python 3.13.5, an X11 session (`DISPLAY=:0`; Wayland was unset), xdg-utils 1.2.1, zenity 4.1.90, shared-mime-info 2.4-5, and desktop-file-utils 0.28-1. The default HTML application was `vivaldi-stable.desktop`. `maff-open` exited 0 and asked `xdg-open` to open the extracted page. Firefox, run headless, wrote a screenshot of that `index.html` showing the saved layout. `xdg-mime query filetype` returned `application/x-maff`, and `xdg-mime query default application/x-maff` returned `maff-open.desktop`. Calibre's configuration directory (`~/.config/calibre`) contained no MAFF viewer setting.
