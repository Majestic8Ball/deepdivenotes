>My own deep dive writing a project intending to learn about CLI, command parsing, and GO

## How CLI Programs Work
---
A command in the CLI looks like
```
./manga-to-epub -dir /path/to/manga -title "My Manga"
```
The program is just ran through ./manga-to-epub.  It then reads the rest as args. It does it through os.Args[n], where n is the number of arguments - 1 (because the first arg is to run the program). We could manually parse the os.Args, but that would be tedious so go has a package called ``flag`` that does it for us. 
Speaking of this, the os lib is super helpful since it lets us get the args easily. If it wasnt there we would have to manage syscalls and other things.
## Validating and Working in Directories
---
### Directory Validation
``` go
import (
    "os"
    "path/filepath"
)

// Check if directory exists
info, err := os.Stat(dirPath)
if os.IsNotExist(err) {
    // Directory doesn't exist
    return errors.New("directory does not exist")
}
if !info.IsDir() {
    // Path exists but is not a directory
    return errors.New("path is not a directory")
}
```
### Walking Directories
```go
// Find all subdirectories (chapters)
chapters := []string{}
filepath.Walk(rootDir, func(path string, info os.FileInfo, err error) error {
    if err != nil {
        return err
    }
    if info.IsDir() && path != rootDir {
        chapters = append(chapters, path)
    }
    return nil
})
```
### Finding Images in a Directory
```go
// Find all JPG files in a directory
images := []string{}
entries, err := os.ReadDir(chapterDir)
if err != nil {
    return nil, err
}

for _, entry := range entries {
    if !entry.IsDir() {
        name := entry.Name()
        if filepath.Ext(name) == ".jpg" || filepath.Ext(name) == ".jpeg" {
            images = append(images, filepath.Join(chapterDir, name))
        }
    }
}
```
### Sorting Files Naturally
```go
// Natural sort for chapter names like "Chapter 1", "Chapter 2", etc.
sort.Slice(chapters, func(i, j int) bool {
    // Extract numbers from chapter names and compare
    chNumI := extractNumber(filepath.Base(chapters[i]))
    chNumJ := extractNumber(filepath.Base(chapters[j]))
    return chNumI < chNumJ
})

// Helper function to extract numbers from strings
func extractNumber(name string) int {
    // Find a number in the string
    numStr := regexp.MustCompile(`\d+`).FindString(name)
    if numStr == "" {
        return 0
    }
    num, _ := strconv.Atoi(numStr)
    return num
}
```
## Sending Chapter Off To Be Made Into EPUB
---
Now, I have another file in epub/epub.go that takes in the metadata provided and the directory. I am using the library "github.com/writingtoole/epub", it handles a lot of it for me.
## Adding Images and Adding XHTML
---
We then use ``filepath.Glob()`` to read all of the chapter dirs into an array. This is so we can sort it, we need to make sure its naturally sorted. Then we loop through all of the images and just use ``manga.AddImage()``. But EPubs also need XHTML, so we just do this:
```go
		xhtmlContent := fmt.Sprintf(`
    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/1999/xhtml">
    <html xmlns="http://www.w3.org/1999/xhtml">
    <head><title>Page %d</title></head>
    <body>
      <img src="%s" />
    </body>
    </html>`, i+1, imgFile)

		originalBase := strings.TrimSuffix(imgFile, filepath.Ext(imgFile))
		xhtmlFile := fmt.Sprintf("%s.xhtml", originalBase)

		_, err = manga.AddXHTML(xhtmlFile, xhtmlContent)
		if err != nil {
			fmt.Printf("Warning: Error adding XHTML %s to EPUB: %v\n", xhtmlFile, err)
			continue
		}
```
It uses the XHTML format and uses the imgFile for it.
## Adding The Metadata
---
Now all we have to do is add the metadata using the epub lib. But the strange one is we need to generate a UUID. ``"github.com/google/uuid"`` lets us just generate that for ourself. Then we just end with ``.Write()``.