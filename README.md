# thoughts

thoughts about anything

Developed with https://github.com/rust-lang-nursery/mdBook

> mdbook build

This is the command you will run to render your book, it reads the SUMMARY.md file to understand the structure of your book, takes the markdown files in the source directory as input and outputs static html pages that you can upload to a server.

> mdbook watch

When you run this command, mdbook will watch your markdown files to rebuild the book on every change. This avoids having to come back to the terminal to type mdbook build over and over again.

> mdbook serve

Does the same thing as mdbook watch but additionally serves the book at http://localhost:3000 (port is changeable) and reloads the browser when a change occurs.

> mdbook clean

Delete directory in which generated book is located.
