# Importing your photos from Google Takeout

---

## Creating takeout link

---

* Go to [https://takeout.google.com/](https://takeout.google.com/)

* Deselect all, then select Google Photos. Scroll to bottom and select "next step"

* Get the links via email, choose 50GB file size. 

!!!note
    It may take awhile to generate the link

## Downloading the files

---

### Standard method

Download the links like any other download, save to wherever works, and import to immich using `immich-go` 

### Headless download

!!!note
    you can only download a link 5 times. After that, you must regenerate the takeout link again.

* Go to the generated takeout link. 

* Press `F12` to open the developer tools of browser, then choose 'network'

* Start a download on browser, then pause it.

* Find the download from the domain `https://takeout-download.usercontent.google.com` - **IT MUST BE THIS DOMAIN**

* Right click -> 'Copy as cURL'

* Create a ssh session using `tmux` or `screen` to keep download going if terminal dies.

* This will make a long string, **paste to a text editor and add `-O`** (- Capital O). This will write to file instead of stdout.
 
* This will begin download. Then after all `.zip` files are there, import using `immich-go`



