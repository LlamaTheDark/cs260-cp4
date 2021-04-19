# cs260-cp4
creative project #4 for cs260
see results [here](https://cp4.seamusmorrison.org) (https://cp4.seamusmorrison.org)

## Overview
The purpose of the web app is to be able to store note files, tag them, search them by keyword and tag, as well as render their content using html and markdown.

## Rendering
again, I just used marked.js to render stuff

## Folders
each folder has notes that reference it in their model, along with various other data.
Folders have the following schema:
```js
const folderSchema = new mongoose.Schema({
    name: String,
    description: String,
    noteCount: Number,
    tags: Array, // the accumulation of all the tags of its notes
});
```

## Notes
Notes have the following schema:
```js
const noteSchema = new mongoose.Schema({
    name: String,
    extension: String,
    content: String,
    tags: Array,
    folder: {
        type: mongoose.Schema.ObjectId,
        ref: 'Folder'
    }
});
```

## Pages
There are two pages:
1. a page to search notes, and
2. a page to view the notes.

I implemented them with Vue CLI and Vue Router.

## API
The site uses an API to read from the back end. 
The Folder endpoints have the following functionality:
- create a new folder
- delete a folder
- get a list of folders
- get a single folder (by id reference)

The Note endpoints have the following functionality:
- create a new note (requires a folder id with which to create the note)
- delete a note
- get a list of all notes
- get a list of all notes which belong to a folder (specified by folder id)
- get a list of all notes which have a requested tag belonging to them (i.e. a tag in their tag array)
- edit a note

The back end is implemented with Node, Express and MongoDB.
