# Handling SVGs with Python PIL

I wanted my reverse image search to be able query for SVG images. Unfortunately, Python PIL can not load SVGs. My solution was to convert SVGs into PNGs using a library called [cairosvg](https://github.com/Kozea/CairoSVG).

For this to work, you will need to have Cairo installed. On Debian and Ubuntu it can be installed with this command:
```sh
apt install libcairo2-dev
```
[Cairo](https://www.cairographics.org/) is an image processing library, and a dependent of the CairoSVG library.

You'll also obviously need to install CairoSVG.
```sh
pip install cairosvg
```

Here is some sample code that convert

```py
import cairosvg
from io import BytesIO
from PIL import Image

with open("image.svg", "rb") as image:
    imageBinary = BytesIO(image.read())

    buff = BytesIO()
    cairosvg.svg2png(bytestring=imageBinary.getvalue(), write_to=buff)
    buff.seek(0)

    img = Image.open(buff)
```
This code opens an SVG image file called "image.svg" in binary mode and reads it into a variable called "imageBinary".

Next, it creates a BytesIO object called "buff" to store the PNG version of the SVG image. The cairosvg.svg2png() function is used to convert the SVG image to PNG format and write the result to "buff".

The buff.seek(0) line sets the position of the file pointer to the beginning of the file.

Finally, the Image.open() function is used to open the PNG image stored in "buff" and assign it to the variable "img".

</br>
Now the SVG (which is actually now a PNG) can be processed as a PIL image, and you can use it with the imagehash library if you would like.

```py
imgHash = imagehash.phash(img)
```

This solution worked quite effectively for [my own reverse image search](https://www.reversewikipedia.com/).