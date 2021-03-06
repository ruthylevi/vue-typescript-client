<template>
  <v-container fluid style="max-width: 918px;">
    <v-timeline dense>
      <v-slide-x-transition group hide-on-leave>
        <v-timeline-item v-if="onProject" large key="icon" class="text-left">
          <template v-if="!newEntry" v-slot:icon>
            <v-btn dark icon @click="newEntryToggle">
              <v-icon>mdi-plus</v-icon>
            </v-btn>
          </template>
          <template v-else v-slot:icon>
            <v-btn dark icon @click="newEntryToggle">
              <v-icon>mdi-close</v-icon>
            </v-btn>
          </template>
          <v-card
            v-if="newEntry && newEntryType === undefined"
            color="primary"
            class="d-inline-block"
            style="border-radius:15px;"
          >
            <v-btn-toggle
              v-if="newEntry && newEntryType === undefined"
              background-color="primary"
              dark
              borderless
              rounded
              class="d-inline-block"
              v-model="newEntryType"
            >
              <v-btn x-large icon>
                <v-icon>mdi-textbox</v-icon>
              </v-btn>
              <v-btn x-large icon>
                <v-icon>mdi-camera</v-icon>
              </v-btn>
            </v-btn-toggle>
          </v-card>
          <v-card v-if="newEntryType === 0" class="elevation-2">
            <v-card-title class="headline">
              <v-btn-toggle borderless rounded v-model="newEntryType">
                <v-btn x-large icon>
                  <v-icon>mdi-textbox</v-icon>
                </v-btn>
                <v-btn x-large icon>
                  <v-icon>mdi-camera</v-icon>
                </v-btn>
              </v-btn-toggle>
              <span class="mx-2">
                New Text Entry
              </span>
              <v-spacer />
              <v-btn v-if="!loading" icon @click="newEntryToggle">
                <v-icon>mdi-close</v-icon>
              </v-btn>
              <v-progress-circular v-else indeterminate color="primary" />
            </v-card-title>
            <v-card-text class="text-left">
              <v-textarea
                class="mx-1"
                rows="5"
                counter="2048"
                outlined
                v-model="newTextEntry"
                :rules="[rules.length(2048)]"
              />
              <v-btn @click="postTextBoardEntry" :disabled="newTextBoardEntryInvalid || loading">
                Post
              </v-btn>
            </v-card-text>
          </v-card>
          <v-card v-if="newEntryType === 1" class="elevation-2">
            <v-card-title class="headline">
              <v-btn-toggle borderless rounded v-model="newEntryType">
                <v-btn x-large icon>
                  <v-icon>mdi-textbox</v-icon>
                </v-btn>
                <v-btn x-large icon>
                  <v-icon>mdi-camera</v-icon>
                </v-btn>
              </v-btn-toggle>
              <span class="mx-2">
                New Picture/Video Entry
              </span>
              <v-spacer />
              <v-btn v-if="!loading" icon @click="newEntryToggle">
                <v-icon>mdi-close</v-icon>
              </v-btn>
              <v-progress-circular v-else indeterminate color="primary" />
            </v-card-title>
            <v-card-text class="text-left">
              <span v-if="newMediaEntry !== undefined">
                <v-container>
                  <h2>Preview</h2>
                </v-container>
                <v-img
                  v-if="newMediaEntry.type !== 'video/mp4'"
                  :src="image(newMediaEntry)"
                  min-height="350"
                  max-height="350"
                  contain
                />
                <VuePlyr
                  v-else-if="newMediaEntry.type === 'video/mp4'"
                  :emit="['enterfullscreen', 'exitfullscreen']"
                  @enterfullscreen="fullscreen = true"
                  @exitfullscreen="fullscreen = false"
                >
                  <video
                    :src="image(newMediaEntry)"
                    :style="!fullscreen ? 'max-height:350px;' : ''"
                  />
                </VuePlyr>
                <br />
              </span>
              <v-file-input
                v-model="newMediaEntry"
                outlined
                dense
                show-size
                accept="image/jpeg, image/png, image/gif, video/mp4"
                prepend-icon="mdi-camera"
                :rules="[rules.size]"
                :disabled="loading"
              />
              <v-btn
                @click="postMediaBoardEntry()"
                :disabled="newMediaBoardEntryInvalid || loading"
              >
                Post
              </v-btn>
            </v-card-text>
          </v-card>
        </v-timeline-item>
        <v-timeline-item v-for="item in boardEntries" :key="item.id">
          <v-card class="elevation-2">
            <v-card-title class="headline">
              {{ calendarTime(item.createdAt) }}
              <v-spacer />
              <v-btn
                v-if="
                  editEntryIndex(item.id) !== -1 &&
                    item.document.type === 'TEXT' &&
                    !editEntries[editEntryIndex(item.id)].loading
                "
                icon
                :disabled="editEntryIsInvalid(item.id)"
                @click="editBoardEntry(item.id)"
              >
                <v-icon>mdi-check</v-icon>
              </v-btn>
              <v-btn
                v-if="
                  editEntryIndex(item.id) !== -1 &&
                    item.document.type === 'MEDIA' &&
                    !editEntries[editEntryIndex(item.id)].loading
                "
                icon
                :disabled="editEntryIsInvalid(item.id)"
                @click="uploadMediaFile(item.id)"
              >
                <v-icon>mdi-check</v-icon>
              </v-btn>
              <v-progress-circular
                v-if="
                  editEntryIndex(item.id) !== -1 && editEntries[editEntryIndex(item.id)].loading
                "
                indeterminate
                color="primary"
              />
              <v-btn
                v-if="
                  editEntryIndex(item.id) !== -1 && !editEntries[editEntryIndex(item.id)].loading
                "
                icon
                @click="toggleEditEntry(item.id, '', '')"
              >
                <v-icon>mdi-close</v-icon>
              </v-btn>
              <v-btn
                v-if="onProject && editEntryIndex(item.id) === -1"
                @click="
                  toggleEditEntry(
                    item.id,
                    item.document.type === 'TEXT' ? item.document.body : '',
                    item.document.type === 'MEDIA' ? item.document.mediaType : ''
                  )
                "
                icon
              >
                <v-icon>mdi-pencil</v-icon>
              </v-btn>
              <v-dialog v-if="onProject && editEntryIndex(item.id) === -1" max-width="500px">
                <template v-slot:activator="{ on }">
                  <v-btn v-on="on" icon>
                    <v-icon>mdi-delete</v-icon>
                  </v-btn>
                </template>
                <v-card>
                  <v-card-title>
                    Delete Board Entry?<br />{{ calendarTime(item.createdAt) }}
                  </v-card-title>
                  <v-card-text>
                    <v-container>
                      <v-container
                        v-if="editEntryIndex(item.id) === -1 && item.document.type === 'TEXT'"
                        class="text-left"
                        v-text="item.document.body"
                        style="white-space: pre-line;"
                      />
                      <v-img
                        v-else-if="
                          editEntryIndex(item.id) === -1 && item.document.mediaType === 'IMAGE'
                        "
                        :src="item.document.mediaLink"
                        min-height="350"
                        max-height="350"
                        contain
                      />
                      <VuePlyr
                        v-else-if="
                          editEntryIndex(item.id) === -1 && item.document.mediaType === 'VIDEO'
                        "
                        :emit="['enterfullscreen', 'exitfullscreen']"
                        @enterfullscreen="fullscreen = true"
                        @exitfullscreen="fullscreen = false"
                      >
                        <video
                          :src="item.document.mediaLink"
                          :style="!fullscreen ? 'max-height:350px;' : ''"
                        />
                      </VuePlyr>
                    </v-container>
                    <v-btn @click="deleteBoardEntry(item.id)">
                      <h2>Delete</h2>
                    </v-btn>
                  </v-card-text>
                </v-card>
              </v-dialog>
            </v-card-title>
            <v-card-text v-if="item.document.type === 'TEXT'" class="text-left">
              <v-container
                v-if="editEntryIndex(item.id) === -1"
                class="text-left"
                v-text="item.document.body"
                style="white-space: pre-line;"
              />
              <v-textarea
                v-else
                class="mx-1"
                rows="5"
                counter="2048"
                outlined
                :placeholder="item.document.body"
                v-model="editEntries[editEntryIndex(item.id)].body"
                :rules="[rules.length(2048)]"
              />
            </v-card-text>
            <v-card-text v-if="item.document.type === 'MEDIA'">
              <v-container v-if="editEntryIndex(item.id) !== -1">
                <h2>Preview</h2>
              </v-container>
              <v-img
                v-if="editEntryIndex(item.id) === -1 && item.document.mediaType === 'IMAGE'"
                :src="item.document.mediaLink"
                min-height="350"
                max-height="350"
                contain
              />
              <v-img
                v-else-if="
                  editEntryIndex(item.id) !== -1 &&
                    editEntries[editEntryIndex(item.id)].mediaType === 'IMAGE'
                "
                :src="
                  image(editEntries[editEntryIndex(item.id)].body) === ''
                    ? item.document.mediaLink
                    : image(editEntries[editEntryIndex(item.id)].body)
                "
                min-height="350"
                max-height="350"
                contain
              />
              <VuePlyr
                v-else-if="editEntryIndex(item.id) === -1 && item.document.mediaType === 'VIDEO'"
                :emit="['enterfullscreen', 'exitfullscreen']"
                @enterfullscreen="fullscreen = true"
                @exitfullscreen="fullscreen = false"
              >
                <video
                  :src="item.document.mediaLink"
                  :style="!fullscreen ? 'max-height:350px;' : ''"
                />
              </VuePlyr>
              <VuePlyr
                v-else-if="
                  editEntryIndex(item.id) !== -1 &&
                    editEntries[editEntryIndex(item.id)].mediaType === 'VIDEO'
                "
                :emit="['enterfullscreen', 'exitfullscreen']"
                @enterfullscreen="fullscreen = true"
                @exitfullscreen="fullscreen = false"
              >
                <video
                  :src="
                    image(editEntries[editEntryIndex(item.id)].body) === ''
                      ? item.document.mediaLink
                      : image(editEntries[editEntryIndex(item.id)].body)
                  "
                  :style="!fullscreen ? 'max-height:350px;' : ''"
                />
              </VuePlyr>

              <br v-if="editEntryIndex(item.id) !== -1" />
              <v-file-input
                v-if="editEntryIndex(item.id) !== -1"
                v-model="editEntries[editEntryIndex(item.id)].body"
                outlined
                dense
                show-size
                accept="image/jpeg, image/png, image/gif, video/mp4"
                prepend-icon="mdi-camera"
                v-on:change="addEditMediaType(editEntries[editEntryIndex(item.id)].body, item.id)"
                :rules="[rules.size]"
                :disabled="editEntries[editEntryIndex(item.id)].loading"
              />
            </v-card-text>
          </v-card>
        </v-timeline-item>
        <v-timeline-item key="created">
          <v-card>
            <v-card-title class="headline">
              {{ calendarTime(createdAt) }}
              <v-spacer />
            </v-card-title>
            <v-card-text>
              <v-img
                src="@/../public/assets/balloons.jpeg"
                height="350"
                gradient="to top right, rgba(100,115,201,.33), rgba(25,32,72,.7)"
              >
                <v-overlay :absolute="true">
                  <v-container class="white--text display-3">
                    Project Created!
                  </v-container>
                </v-overlay>
              </v-img>
            </v-card-text>
          </v-card>
        </v-timeline-item>
      </v-slide-x-transition>
    </v-timeline>
  </v-container>
</template>

<script>
import axios from 'axios';
import uuid from 'uuid/v4';
import moment from 'moment';
import VuePlyr from 'vue-plyr';
import apiCall from '@/apiCall';

export default {
  components: {
    VuePlyr,
  },
  props: {
    value: Array,
    onProject: Boolean,
    createdAt: String,
  },
  data() {
    return {
      id: uuid(),
      fullscreen: false,
      newEntry: false,
      newText: false,
      newMedia: false,
      newEntryType: undefined,
      newTextEntry: '',
      newMediaEntry: undefined,
      loading: false,
      editEntries: [],
      rules: {
        length: len => v => (v || '').length <= len || `Invalid character length, must be less than ${len}`,
        size: value => !value
          || (value.size !== undefined && value.size < 20000000)
          || 'Media size should be less than 20 MB!',
      },
    };
  },
  methods: {
    calendarTime(dateInput) {
      return moment(dateInput).calendar();
    },
    editEntryIndex(entryId) {
      return this.editEntries.map(entry => entry.id).indexOf(entryId);
    },
    toggleEditEntry(entryId, entryBody, entryType) {
      if (this.editEntries.map(entry => entry.id).includes(entryId)) {
        const entryIndex = this.editEntries.map(entry => entry.id).indexOf(entryId);
        this.editEntries.splice(entryIndex, 1);
      } else {
        this.editEntries.push({
          id: entryId,
          body: entryBody,
          mediaType: entryType,
          loading: false,
        });
      }
    },
    editEntryIsInvalid(entryId) {
      const editEntryIndex = this.editEntryIndex(entryId);
      if (
        this.editEntries[editEntryIndex].body === undefined
        || this.editEntries[editEntryIndex].body === null
      ) return true;
      if (this.editEntries[editEntryIndex].mediaType !== '') {
        return this.editEntries[editEntryIndex].body.size >= 20000000;
      }
      const editEntryLength = this.editEntries[editEntryIndex].body.length;
      return editEntryLength === 0 || editEntryLength > 2048;
    },
    async postTextBoardEntry() {
      this.loading = true;
      const response = await apiCall.methods.post(
        `/projects/${this.$route.params.id}/board`,
        '',
        {
          id: this.id,
          document: {
            type: 'TEXT',
            body: this.newTextEntry,
          },
        },
        this.$route.fullPath,
      );
      if (response.status === 200) {
        this.boardEntries.unshift(response.data);
        this.newTextEntry = '';
        this.newEntry = false;
        this.id = uuid();
        this.newEntryType = undefined;
      }
      this.loading = false;
    },
    async postMediaBoardEntry() {
      this.loading = true;
      const response = await apiCall.methods.post(
        `/projects/${this.$route.params.id}/board`,
        '',
        {
          id: this.id,
          document: {
            type: 'MEDIA',
            mediaLink: `https://marqetplace-staging-photos.s3.amazonaws.com/projects/${this.$route.params.id}/board/${this.id}`,
          },
        },
        this.$route.fullPath,
      );
      if (response.status === 200) {
        const response2 = await this.postMediaBoardEntryUpload();
        if (response2 !== undefined && response2.status === 204) {
          this.boardEntries.unshift(response.data);
          this.$set(this.boardEntries[0].document, 'mediaLink', this.image(this.newMediaEntry));
          if (this.newMediaEntry.type === 'video/mp4') {
            this.$set(this.boardEntries[0].document, 'mediaType', 'VIDEO');
          } else {
            this.$set(this.boardEntries[0].document, 'mediaType', 'IMAGE');
          }
          this.newMediaEntry = undefined;
          this.newEntry = false;
          this.id = uuid();
          this.newEntryType = undefined;
        }
        this.loading = false;
      }
    },
    async postMediaBoardEntryUpload() {
      const regex = /\/(.*)/gm;
      const fileType = regex.exec(this.newMediaEntry.type)[1];
      const response = await apiCall.methods
        .mediaUpload(
          `/projects/${this.$route.params.id}/board/${this.id}`,
          fileType.toUpperCase(),
          this.newMediaEntry,
          this.$route.fullPath,
        )
        .catch((error) => {
          apiCall.methods.otherError('Media upload failed');
        });
      return response;
    },
    async editBoardEntry(entryId) {
      const response = await apiCall.methods.patch(
        `/projects/${this.$route.params.id}/board/${entryId}`,
        '',
        {
          document: {
            type: 'TEXT',
            body: this.editEntries[this.editEntryIndex(entryId)].body,
          },
        },
        this.$route.fullPath,
      );
      if (response.status === 200) {
        const entryIndex = this.boardEntries.map(entry => entry.id).indexOf(entryId);
        const editEntryIndex = this.editEntryIndex(entryId);
        const newBody = this.editEntries[editEntryIndex].body;
        this.boardEntries[entryIndex].document.body = newBody;
        this.editEntries.splice(editEntryIndex, 1);
      }
    },
    async deleteBoardEntry(entryId) {
      const response = await apiCall.methods.delete(
        `/projects/${this.$route.params.id}/board/${entryId}`,
        '',
        {},
        this.$route.fullPath,
      );
      if (response.status === 200) {
        const entryIndex = this.boardEntries.map(entry => entry.id).indexOf(entryId);
        this.boardEntries.splice(entryIndex, 1);
      }
    },
    async uploadMediaFile(entryId) {
      const entryIndex = this.boardEntries.map(entry => entry.id).indexOf(entryId);
      const editEntryIndex = this.editEntryIndex(entryId);
      this.$set(this.editEntries[editEntryIndex], 'loading', true);
      const newMediaFile = this.editEntries[editEntryIndex].body;
      const regex = /\/(.*)/gm;
      const fileType = regex.exec(newMediaFile.type)[1];
      const response = await apiCall.methods
        .mediaUpload(
          `/projects/${this.$route.params.id}/board/${entryId}`,
          fileType.toUpperCase(),
          newMediaFile,
          this.$route.fullPath,
        )
        .catch((error) => {
          this.$set(this.editEntries[editEntryIndex], 'loading', false);
          alert("Media upload failed"); // eslint-disable-line
        });
      if (response !== undefined && response.status === 204) {
        this.boardEntries[entryIndex].document.mediaLink = this.image(newMediaFile);
        this.editEntries.splice(editEntryIndex, 1);
        if (fileType === 'mp4') {
          this.$set(this.boardEntries[entryIndex].document, 'mediaType', 'VIDEO');
        } else {
          this.$set(this.boardEntries[entryIndex].document, 'mediaType', 'IMAGE');
        }
      }
    },
    async addMediaTypes() {
      const linksArr = [];
      for (let i = 0; i < this.boardEntries.length; i += 1) {
        if (
          this.boardEntries[i].document.mediaType === undefined
          && this.boardEntries[i].document.type === 'MEDIA'
        ) {
          linksArr.push(this.boardEntries[i].document.mediaLink);
        }
      }
      const resp = axios.all(linksArr.map(l => axios.get(l))).then(
        axios.spread((...res) => {
          const types = res.map(a => a.headers['content-type']);
          for (let i = 0; i < this.boardEntries.length; i += 1) {
            if (
              this.boardEntries[i].document.mediaType === undefined
              && this.boardEntries[i].document.type === 'MEDIA'
            ) {
              if (types.shift() === 'video/mp4') {
                this.$set(this.boardEntries[i].document, 'mediaType', 'VIDEO');
              } else {
                this.$set(this.boardEntries[i].document, 'mediaType', 'IMAGE');
              }
            }
          }
        }),
      );
    },
    addEditMediaType(file, entryId) {
      const entryIndex = this.boardEntries.map(entry => entry.id).indexOf(entryId);
      const editEntryIndex = this.editEntryIndex(entryId);
      if (file === undefined) {
        this.$set(
          this.editEntries[editEntryIndex],
          'mediaType',
          this.boardEntries[entryIndex].document.mediaType,
        );
      } else if (file.type === 'video/mp4') {
        this.$set(this.editEntries[editEntryIndex], 'mediaType', 'VIDEO');
      } else {
        this.$set(this.editEntries[editEntryIndex], 'mediaType', 'IMAGE');
      }
    },
    image(file) {
      try {
        return URL.createObjectURL(file);
      } catch {
        return '';
      }
    },
    newEntryToggle() {
      this.newEntry = !this.newEntry;
      this.newEntryType = undefined;
    },
  },
  computed: {
    newTextBoardEntryInvalid() {
      return (
        this.newTextEntry === null
        || this.newTextEntry.length === 0
        || this.newTextEntry.length > 2048
      );
    },
    newMediaBoardEntryInvalid() {
      return this.newMediaEntry === undefined || this.newMediaEntry.size >= 20000000;
    },
    boardEntries: {
      get() {
        return this.value;
      },
      set(value) {
        this.$emit('input', value);
      },
    },
  },
  mounted() {
    this.addMediaTypes();
  },
  watch: {
    async boardEntries() {
      this.addMediaTypes();
    },
  },
};
</script>
