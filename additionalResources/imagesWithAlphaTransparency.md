# Python imagehash with an alpha transparent image

The Python [imagehash](https://github.com/JohannesBuchner/imagehash) library by default does not work on images with an alpha channel. The solution I used was simply to project the image onto a white background.

Here is some code that works for me (credits to [this stackoverflow post](https://stackoverflow.com/a/35859141)):

```py
def remove_transparency(img, bg_colour=(255, 255, 255)):
    # Only process if image has transparency (http://stackoverflow.com/a/1963146)
    if img.mode in ('RGBA', 'LA') or (img.mode == 'P' and 'transparency' in img.info):

        # Need to convert to RGBA if LA format due to a bug in PIL (http://stackoverflow.com/a/1963146)
        alpha = img.convert('RGBA').split()[-1]

        # Create a new background image of our matt color.
        # Must be RGBA because paste requires both images have the same format
        # (http://stackoverflow.com/a/8720632  and  http://stackoverflow.com/a/9459208)
        bg = Image.new("RGBA", img.size, bg_colour + (255,))
        bg.paste(img, mask=alpha)
        return bg

    else:
        return img
```

