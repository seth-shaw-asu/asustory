ASU Story Collection Builder Conversion
=======================================

Convert https://www.asu.edu/lib/archives/asustory/intro.htm into a [CollectionBuilder](https://collectionbuilder.github.io/) site.

## Metadata

I started by using wget to crawl the site: `wget --wait=2 --mirror --convert-links --adjust-extension --page-requisites --no-parent https://www.asu.edu/lib/archives/asustory/intro.htm`.

I zipped the `www.asu.edu/lib/archives/asustory/pages` contents into a zip file and passed it to [ChatGPT to create a TSV](https://chatgpt.com/c/69039424-dca8-832e-b545-4b818f860727).

It did okay at identifying the object identifier, title, and description (although with different column names) but I still removed a lot of erroneous fields.

I then did some manual work, such as renaming fields, to fit the [CollectionBuilder CSV](https://collectionbuilder.github.io/cb-docs/docs/metadata/csv_metadata/) format.

The Excel-produced CSV didn't work (a known issue for CollectionBuilder), so I moved it to [Google Sheets](https://docs.google.com/spreadsheets/d/1Ax97YvOJgrTZn-4jML-WCkkS_vJfiM1rCZfCXXc15wc/edit?gid=1055021335#gid=1055021335) and then downloaded a new copy of the CSV.

The existing site includes some complex objects represented as a flat list. I added top-level entries for these items and added the parent identifiers to a new `parentid` column as required by CollectionBuilder's [compound objects convention](https://collectionbuilder.github.io/cb-docs/docs/metadata/compound-objects/).

## Images

The crawl only grabbed low-quality versions of the images. These are good enough for the proof-of-concept and, as a bonus, are small enough to put in git repository, which makes the deployment simpler. Later on I unzipped all the zip files and replaced the existing poor quality images with the larger ones. (Note that complex objects didn't have zip files and at least one item, `06athlet`, is missing a zip file with a larger version.)

I used the included derivative generation script to create the "small" and "thumbnail" versions (`rake generate_derivatives`) and then used a formula in Google Sheets to populate their path (e.g. `=CONCATENATE(CONCATENATE('/objects/small/', A2)'_sm.jpg')`).

## Site Setup and Deployment

I created [*this* GitHub repository](https://github.com/seth-shaw-asu/asustory) to provide a simplified deployment mechanism. The site is available @ [https://seth-shaw-asu.github.io/asustory/](https://seth-shaw-asu.github.io/asustory/).

### Category Pages

The category pages are "[interpretive pages](https://collectionbuilder.github.io/cb-docs/docs/pages/interpretive/)". I used pandoc to create markdown versions of the pages from the earlier webcrawl:

```
find . -iname "*.htm" -type f -exec pandoc -o {}.md {} \;
for file in *.htm.md; do mv -- "$file" "${file%.htm.md}.md"; done;
```

The markdown pages had a lot of boiler-plate from the HTML that were deleted. The image reference and links were often jumbled and didn't work. These were replaced with "[feature includes](https://collectionbuilder.github.io/cb-docs/docs/pages/features/)" to insert images into the page: `{% include feature/image.html objectid="01acpro" %}`. I added in the first line manually and then asked CoPilot to duplicate the line while incrementing the identifier. This gave me a starting point and I manually checked the identifiers against the existing Markdown markup as some identifiers weren't used and some were slightly different (e.g. the complex objects which used the first page). The `objectid` property can accept a semi-colon delimited list to place images side-by-side, so I watched the markup I was replacing to find instances where images were side-by-side instead of above/below.

The compound objects need a "small" image to display using the feature includes, so I simply copied the path from the first child into the parent CSV row.

There were some references on some of the pages to images that didn't have their own page. I hadn't copied those over and so were simply removed from the page.

Each of the pages requires a YAML header information section, e.g.:

```
---
title: Academic Programs
layout: page
permalink: /acpro.html
---
```

The initial example I pulled from used `layout: cloud` but I changed them to `page`. The `permalink` field is required for the page to be available during the build.

## Special Pages

The `search.md` clobbered the existing one which broke site search, so I used git to revert it back to the one provided by CollectionBuilder.

When reviewing the timeline I decided to ask ChatGPT to give a first pass at updating the `date` and `date_created`. This did a decent job, but ChatGPT didn't understand a decade suffixed with 's' (e.g. `1960s`) so I filled those in. There were also some erroneous entries where ChatGPT used the accession number from the citation which I manually corrected.

The intro page's html was copied into `_layouts/home-infographic.html` although, since we really don't need all the preview features, I probably could have just updated the `pages/index.md` and used the `page` template.

## Templating and Navigation

Most of the site display is configured in `_config.yml` where I updated the site title, tagline, keywords, etc. It is also where the name of the CSV file is configured (`metadata`).

The only theme file (`_data/theme.yml`) change was the `featured-image` where I randomly picked an objectid from the metadata.

The [navigation menu](https://collectionbuilder.github.io/cb-docs/docs/customization/config-nav/) is configured by `_data/config-nav.csv`. I added each of the category pages created earlier. The existing site uses three levels of navigation, but CollectionBuilder only allows two, so the categories had to stay in the top-level for them to have sub levels. Oddly, the leadership category's child links are all sections of the same page, which breaks the menu build and so were removed.