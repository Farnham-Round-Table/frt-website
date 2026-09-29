# Farnham Round Table website

A flat, static copy of [farnhamroundtable.org.uk](https://farnhamroundtable.org.uk/), which previously ran on WordPress.
There is no build step: every page is a plain HTML file, and all pages share `style.css`.

## Layout

Each page lives at the same path it had on the old site, as `<path>/index.html`, so old links keep working:

| Page | File |
| --- | --- |
| Home | `index.html` |
| Charity Events | `events/index.html` (+ one folder per event) |
| Blog | `blog/index.html`, posts under `YYYY/MM/DD/<slug>/` |
| About Us, Contact, Donate | `about-us/…` |
| Cookie, Privacy, Terms, Sitemap, Thank You | top-level folders |

Images are in `images/`. Internal links are relative and point at `index.html` files,
so the site works when opened straight from disk as well as from any static host
(GitHub Pages, Netlify, an ordinary web server).

## Previewing

Open `index.html` in a browser, or serve the folder:

```sh
python3 -m http.server 8000
```

## Editing

- Page content is plain HTML inside each file's `<main>` element.
- The header navigation and footer are repeated in every page; when changing them, update every file.
- The contact form (`about-us/contact/index.html`) needs a form-handling service, because static hosting cannot send email.
  Replace the `action` URL on the form before going live.
