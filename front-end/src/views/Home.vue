<template>
<div class='home'>
    <div id='controls'>
        Folders:
        <button @click="addFolder">
            Add Folder
        </button>
        <button @click="deleteFolder">
            Delete Folder
        </button>
        Notes:
        <button @click="addNote">
            Add Note
        </button>
        <button @click="deleteNote">
            Delete Note
        </button>
        <input type='checkbox' v-model="rendered">
        Render Text
    </div>
    <div class='main-components'>
        <div class='lists'>
            <div id='folders-list'>
                <h2 class='title'>
                    Folders
                </h2>
                <ul id='folders'>
                    <li v-for="f in folders" :key="f._id" @click='selectFolder(f)'>
                        <div :class="{'selected-folder': (f === folder), 'folder-title': true}">
                            <h4>
                                {{ f.name }}
                            </h4>
                            <div v-if="f === folder">
                                <p class='folder-description'>
                                    {{ f.description }}
                                </p>
                                <p class='folder-note-count'>
                                    total notes: {{ f.noteCount }}
                                </p>
                            </div>
                        </div>
                        <hr>
                    </li>
                </ul>
            </div>
            <div id='files-list'>
                <FilesList id='files-list' :notes="notes" :options="true" />

                <!-- <h2 class='title'>
                    Notes
                </h2> -->
                <!--
                <ul id='files'>
                    <li v-for="n in notes" :key="n._id" @click='selectNote(n)'>
                        <h4 :class="{selected: (n === note), 'note-title': true}">
                            {{ n.name }}{{ n.extension }}
                        </h4>
                        <hr>
                    </li>
                </ul> -->
            </div>
        </div>
        <FileView id='file-view' :note="note" :rendered="rendered" :folder="folder" />
        <div class='tag-view'>
            <div>
                <h2 class='title'>
                    Tags
                </h2>
                <ul id='tags'>
                    <li id='tag' v-for="tag in currentTags" :key="currentTags.indexOf(tag)">
                        {{tag}}
                        <b id='remove-tag' @click='removeTag(tag)'>
                        --
                        </b>
                    </li>
                </ul>
            </div>
            <form @submit.prevent="addNewTag">
                <input id='new-tag-text' v-model='newTagText' placeholder='add new tag...' />
            </form>
        </div>
    </div>
</div>
</template>

<script>
/*
TODO: 
 - total notes is still not accurate



*/

import FileView  from '@/components/FileView.vue' ;
import FilesList from '@/components/FilesList.vue';
import axios from 'axios';

export default {
    name: 'Home',
    components: {
        FileView,
        FilesList,
    },
    data() {
        return {
            folders: [],
            folder: null,
            folderName: '',
            notes: [],
            // note: null,
            rendered: false,

            newTagText: '',
        }
    },
    async created() {
        await this.getFolders();

        if(this.note !== null){
            let tmp = await this.getFolderFromNote(this.$root.$data.note);
            this.folder = tmp;
            await this.getNotes();
            this.$root.$data.note = this.notes.find(n => n._id === this.$root.$data.note._id);
        }
    },
    computed: {
        currentTags() {
            return (this.$root.$data.note === null)?[]:this.$root.$data.note.tags;
        },
        note() {
            return this.$root.$data.note;
        }
    },
    methods: {
        async getFolders() {
            try {
                const response = await axios.get('/api/folders');
                this.folders = response.data;
                
                if (this.folders.length != 0){ // if there are folders present upon creation, select the first one
                    this.folder = this.folders[this.folders.length-1];
                }
                if(this.folder !== null){
                    this.getNotes();
                }
                if(this.folders.length === 0){
                    this.folder = null;
                    this.$root.$data.note = null;
                }

            } catch(error) {
                console.log(error);
            }
        },
        async addFolder() {
            let name = prompt('Enter a name for your new folder: ');
            if(name === '' || name === null){
                return;
            }
            let description = prompt('Enter a description for the folder: ');
            if(description === '' || description === null){
                return;
            }
            let folder = {
                name: name,
                description: description,
                tags: [],
            }
            try{
                await axios.post(`/api/folders`, folder);
                this.getFolders();
            }catch(error){
                console.log(error);
            }
        },
        async deleteFolder() {
            if(this.folder === null)
                return;
            try {
                await axios.delete(`/api/folders/${this.folder._id}`);
                this.getFolders();
            } catch(err) {
                console.log(err);
            }
        },
        async deleteNote() {
            if(this.$root.$data.note === null)
                return;
            // you'll have to delete each tag from the note first so that we don't have leftover
            // tags in the folder's tag array
            // I really shoulda just done a computed value for the folder's tags it woulda made this easier
            for(let t of this.$root.$data.note.tags){
                this.removeTag(t);
            }
            try {
                await axios.delete(`/api/folders/${this.folder._id}/notes/${this.$root.$data.note._id}`);

                this.folder.noteCount--;
                await axios.put(`/api/folders/${this.folder._id}`, this.folder);
                
                this.getNotes();
            } catch(err) { console.log(err); }
        },
        async getNotes() {
            try {
                const response = await axios.get(`/api/folders/${this.folder._id}/notes`);
                this.notes = response.data;
                // if(this.notes.length != 0){
                //     this.$root.$data.note = this.notes[this.notes.length-1];
                // }
            } catch(error) {
                console.log('error in getNotes()');
                console.log(error);
            }
        },
        async addNote() {
            if(this.folder === null)
                return;
            let name = prompt('Enter a name for your note: ');
            if(name === '' || name === null){
                return;
            }
            let ext = prompt('Enter an extention for your note file (e.g. \'.md\', \'.txt\':');
            if(ext === '' || ext === null){
                return;
            }
            if(Array.from(ext)[0] != '.'){
                ext = '.' + ext;
            }
            let note = {
                name: name,
                extension: ext,
                content: '# ' + name,
                tags: [],
                folder: this.folder
            }
            try{
                // create new note
                await axios.post(`/api/folders/${this.folder._id}/notes`, note);

                this.folder.noteCount++;
                await axios.put(`/api/folders/${this.folder._id}`, this.folder);

                // update selectedNote in data()
                this.getNotes();
            }catch(error){
                console.log('error in addNote()');
                console.log(error);
            }
        },
        async selectFolder(f){
            this.folder = f;
            this.getNotes();
        },
        selectNote(n){
            this.$root.$data.note = n;
            // console.log(`selected note: ${this.$root.$data.note.name}`);
            // console.log(`current folder: ${this.folder.name}`);
        },

        async addNewTag(){
            if(this.$root.$data.note === null)
                return;
            if(!this.$root.$data.note.tags.includes(this.newTagText)){
                this.$root.$data.note.tags.push(this.newTagText);
                await axios.put(`/api/folders/${this.folder._id}/notes/${this.$root.$data.note._id}`, this.$root.$data.note);
                if(!this.folder.tags.includes(this.newTagText)){
                    this.folder.tags.push(this.newTagText);
                    try{
                        await axios.put(`/api/folders/${this.folder._id}`, this.folder);
                    } catch(error) {
                        console.log(error);
                    }
                }
                this.newTagText = '';
            }
        },
        async removeTag(tag){
            let i = this.$root.$data.note.tags.indexOf(tag);
            if(i !== -1){
                this.$root.$data.note.tags.splice(i, 1);
            }

            let inFolder = false;
            let c = 0;
            while(c < this.notes.length && !inFolder){
                i = this.notes[c].tags.indexOf(tag);
                c++;
                if(i !== -1)
                    inFolder = true;
            }
            if(!inFolder){
                this.folder.tags.splice(i, 1);
            }

            await axios.put(`/api/folders/${this.folder._id}/notes/${this.$root.$data.note._id}`, this.$root.$data.note);
            await axios.put(`/api/folders/${this.folder._id}`, this.folder);


        },
        async getFolderFromNote(note){
            let response = await axios.get(`/api/folders/${note.folder}`);
            return this.folders.find(folder => response.data._id === folder._id);
        }
    },
    watch: {
        folder: function(newvalue){
            if(newvalue !== null)
                this.getNotes();
        },
    }
}
</script>

<style scoped>
* {
    font-family: 'Source Sans Pro', sans-serif;
    box-sizing: border-box;
}
.home {
    /* border: 1px solid red; */
}

.lists {
    display: flex;
    align-items: stretch;
    /* border: 1px solid green; */
}
.title {
    text-decoration: underline;
}

/* ## FOR NOTES ## */
#files-list {
    width: 150px;
}
#files-list ul {
    list-style: none;
    padding: 5px;

    overflow: hidden;
    overflow-y: auto;

    width: 150px;
    height: 85%;

    /* border: 1px solid blue; */
}
#files-list ul li {
    margin: 10px 0px;
    /* padding: 0 10px; */
}
#files-list ul li * {
    padding: 0;
    margin: 0;
}
#files-list ul li:hover {
    cursor: pointer;
}

#files-list ul li>h4 {
    padding: 2px;
}

.selected {
    color: green;
    background-color: rgb(238, 255, 238);
}
.note-title {
    /* border: 1px dotted red; */
    padding: 0;
    text-align: left;
}
#file-view {
    width: 700px;
    height: 700px;
    margin: 0;
    padding: 0;
}
#files-list {
    /* border: 1px solid orange; */
    flex: 1;
    /* width: 100%; */
    height: 700px;
}
/* // FOR NOTES // */

/* ## FOR FOLDERS ## */

.selected-folder{
    background-color: lightgreen;
}
#folders-list{
    /* border: 1px solid orange; */
    flex: 1;
    /* width: 100%; */
    height: 700px;
}
#folders {
    /* list-style: none;
    border: 1px solid blue;
    width: 100%; */
    list-style: none;
    padding: 5px;

    overflow: hidden;
    overflow-y: auto;

    width: 150px;
    height: 85%;

    /* border: 1px solid blue; */
}
#folders li {
    margin: 10px 0px;
    text-align: left;
}

/* // FOR FOLDERS // */

* {
    margin: 0 auto;
    padding: 0;
}

button {
    border: none;
    padding: 5px;
    margin: 5px;
    background-color: lightgreen;
}
button:hover {
    cursor:pointer;
}
button:active {
    background-color: rgb(91, 150, 91);
}

.home-editor {
    display: flex;
    justify-content: space-evenly;
}

.tag-view {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
}
#tags {
    list-style: none;
    padding: 2px;
    text-align: right;
    height: 650px;
    overflow-y: scroll;
    width: 180px;
}
.tags {
    width: 15%;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    padding: 10px;
    background-color: rgb(207, 255, 231);

}

#tag {
    text-align: right;
}

#remove-tag:hover {
    cursor: pointer;
}

.main-components {
    display: flex;
}

.folder-note-count {
    font-size: 0.7em;
    text-align: right;
    padding-right: 2px;
}
.folder-description {
    padding-left: 5px;
    font-size: 0.8em;
}
.folder-title {
    padding-left: 3px;
}

.tag-view {
    background-color: rgb(207, 255, 231);
}

#new-tag-text {
    margin: 5px;
}

</style>