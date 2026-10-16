# Mariam Fayed — portfolio

A single-page, dependency-free site that presents Mariam Fayed's journey, projects and skills for interviews.
Everything (fonts, photo, CV) is bundled under `assets/`, so it works offline from a local file or any static host.

## Run locally

```sh
cd /path/to/this/folder
python3 -m http.server 8080
# open http://localhost:8080
```

Opening `index.html` directly in a browser also works.

## Deploy to GitHub Pages

1. Push this folder to a repository (for example `mariam-fayed-portfolio`).
2. In the repository settings, open **Pages** and choose **Deploy from a branch**, branch `main`, folder `/ (root)`.
3. The site appears at `https://<user>.github.io/<repo>/`.

## Editing

- All content lives in `index.html`. Sections: hero, journey (timeline), projects (four showcases with live figures), skills, contact.
- Colours and fonts are CSS tokens at the top of the `<style>` block (light and dark variants share the same token names).
- The dark-mode toggle is in the navigation bar; the `←` / `→` keys (or `J` / `K`) jump between sections while presenting.
- Replace `assets/Mariam_Fayed_CV.pdf` to update the download link and `assets/img/mariam.jpg` to change the photo.
