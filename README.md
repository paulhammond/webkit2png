# webkit2png

webkit2png is a command line tool that creates screenshots of webpages. For more info see [the project homepage](http://www.paulhammond.org/webkit2png/).

## Usage
    webkit2png [options] [http://example.net/ ...]

## Examples
    webkit2png http://google.com/            # screengrab google
    webkit2png -W 1000 -H 1000 http://google.com/ # bigger screengrab of google
    webkit2png -T http://google.com/         # just the thumbnail screengrab
    webkit2png -TF http://google.com/        # just thumbnail and fullsize grab
    webkit2png -o foo http://google.com/     # save images as "foo-thumb.png" etc
    webkit2png -                             # screengrab urls from stdin
    webkit2png -h | less                     # full documentation

## Options
    --version             show program's version number and exit
    -h, --help            show this help message and exit
  
    Browser Window Options:
      -W WIDTH, --width=WIDTH
                          initial (and minimum) width of browser (default: 800)
      -H HEIGHT, --height=HEIGHT
                          initial (and minimum) height of browser (default: 600)
      -z ZOOM, --zoom=ZOOM
                          zoom level of browser, equivalent to "Zoom In" and
                          "Zoom Out" in "View" menu (default: 1.0)
  
    Output size options:
      -F, --fullsize      only create fullsize screenshot
      -T, --thumb         only create thumbnail sreenshot
      -C, --clipped       only create clipped thumbnail screenshot
      --clipwidth=WIDTH   width of clipped thumbnail (default: 200)
      --clipheight=HEIGHT
                          height of clipped thumbnail (default: 150)
      -s SCALE, --scale=SCALE
                          scale factor for thumbnails (default: 0.25)
  
    Output filename options:
      -D DIR, --dir=DIR   directory to place images into
      -o NAME, --filename=NAME
                          save images as NAME-full.png,NAME-thumb.png etc
      -m, --md5           use md5 hash for filename (like del.icio.us)
      -d, --datestamp     include date in filename
  
    Web page functionality:
      --delay=DELAY       delay between page load finishing and screenshot
      --js=JS             JavaScript to execute when the window finishes loading
                          (example: --js='document.bgColor="red";')
      --no-images         don't load images
      --no-js             disable JavaScript support
      --transparent       render output on a transparent background (requires a
                          web page with a transparent background)
      --user-agent=USER_AGENT
                          set user agent header
