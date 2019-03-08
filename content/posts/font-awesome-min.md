---
title: "Modifying Font Files: Making Font-Awesome Awesomer"
date: 2019-03-04T22:22:38-08:00
draft: false
tags: ["python","Font-Awesome","ttx","woff2"]
---

How do you make a website pretty but also small and fast?

FontAwesome is a popular icon set and toolkit. It's what I am using to display the badges on this site! It's easy to use and provides a wide variety of icons just a css class name away. This is great except that for every page view those unused icons are being downloaded, bandwidth is consumed and the majority of icons are unused. 

By Treeshaking the font, keeping only the icons and styles that are used, I was able to reduce the font size by 96% (104KB to 4.1KB). To be fair I'm only using around 4 icons from the entire library.

To put these numbers into perspective, assuming a 2G GSM conneciton (14.4Kbps), font load speed improved from 1 min to 3s, yes I did the math.

> We should forget about small efficiencies, say about 97% of the time: PrematureOptimization is the root of all evil.
>
> \- Donald Knuth (1974, Structured Programming with go to Statements)

Not all websites are created equally and I understand that there might be bigger wins that require less effort. In my case, I was hellbent on reducing the footprint of this site, which means no single page framework, no javascript, and no css framework. Although I am using an awesome predefined theme for now. 

## Modifying Font File (.woff2)
There's a tool for that, [fonttools](https://pypi.org/project/fonttools/) allows you to manipulate font files by exporting them to a modifable XML format. 

```bash
# Please check fonttools on how to setup! 
# https://github.com/fonttools/fonttools `
# setup and activate venv (optional)
foo@bar:~$ python3 -m venv ~/.env-local && ~/.env-local/bin/activate
# install tool and dependencies
(.env-local)foo@bar:~$ pip install fonttools Brotli
# convert woff2 fontawesome-webfont.woff2?v=4.3.0 to ttx XML format
(.env-local)foo@bar:~$ ttx fontawesomewebfont-webfont.woff
Dumping "fontawesome-webfont.woff2" to "fontawesome-webfont.ttx"...
Dumping 'GlyphOrder' table...
Dumping 'head' table...
Dumping 'hhea' table...
Dumping 'maxp' table...
Dumping 'OS/2' table...
Dumping 'hmtx' table...
Dumping 'cmap' table...
Dumping 'loca' table...
Dumping 'glyf' table...
Dumping 'name' table...
Dumping 'post' table...
Dumping 'gasp' table...
Dumping 'FFTM' table...
Dumping 'GDEF' table...
Dumping 'webf' table...
# Now for the "Fun" part 🙁
(.env-local)foo@bar:~$ vi fontawesome-webfont.ttx # or use python script below
```
Now that we have the font in the xml format, we want to delete all unused icons from the various tables while keeping our sanity.

```python
"""
Remove unused icon names and glyphs from xml file.
"""
import xml.etree.ElementTree as ET

tree = ET.parse('fontawesome-webfont.ttx')
root = tree.getroot()
icons_to_keep = ["github","linkedin","quote_left","quote_right"]
parent_elements_to_check = root.findall(".//*[@name]/..")
parent_elements_to_check.append(root.find("GDEF/GlyphClassDef"))

del_cnt = 0
def shouldDelete(el):
    for icon_check in icons_to_keep:
        if (icon_check in el.attrib.get("name","")
           or icon_check in el.attrib.get("glyph","")):
            return False
    return True

# evaluate to list because we are deleting when iterating
for parent in list(parent_elements_to_check):
    for icon in list(parent):
            if shouldDelete(icon):
                parent.remove(icon)
                del_cnt += 1
print('deleted ',del_cnt, ' elements')
tree.write('fontawesome-webfont-modified.ttx',xml_declaration=True,encoding="UTF-8")
```

finally convert back to Woff2 format
```bash
(.env-local)foo@bar:~$ ttx --flavor woff2 fontawesome-webfont-modified.ttx
Compiling "fontawesome-webfont-modified.ttx" to "fontawesome-webfont-modified."...
Parsing 'GlyphOrder' table...
Parsing 'head' table...
Parsing 'hhea' table...
Parsing 'maxp' table...
Parsing 'OS/2' table...
Parsing 'hmtx' table...
Parsing 'cmap' table...
Parsing 'loca' table...
Parsing 'glyf' table...
Parsing 'name' table...
Parsing 'post' table...
Parsing 'gasp' table...
Parsing 'FFTM' table...
Parsing 'GDEF' table...
Parsing 'webf' table...
```

Another alternative is embedding svg icons directly in the HTML. This is a straightforward approach and has [wider browser adoption](https://caniuse.com/#search=svg). Unfortunately, it will generally produce larger files when compared to the woff2 format. 

In my case, this single embedded icon <svg height="32" class="octicon octicon-mark-github" viewBox="0 0 16 16" version="1.1" width="32" aria-hidden="true"><path fill-rule="evenodd" d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0 0 16 8c0-4.42-3.58-8-8-8z"></path></svg> is 6kb which is larger than the sum of icons stored in the modified font file.

## Conclusion
Using FontAwesome allows us to easily add icons which is great for rapid development but this convience comes at a cost of making users download the entire library on every page view. If it's worth it for you, to reduce the font file size by removing the unused dependencies and capitilzie on the savings then there might be a tool that does this less painfully. 