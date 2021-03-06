<template>
  <Layout
    v-bind="{
      currentPage: 'Profile',
      pageOptions: mobileHeaderUiOptions
    }"
  >
    <validation-observer v-slot="{ valid, validated, handleSubmit }">
      <v-container class="createIdeaPage">
        <v-row id="ideaContentRow">
          <v-col cols="12" sm="10" md="8" lg="6">
            <!-- title -->
            <v-text-field-with-validation
              v-model="title"
              v-focus
              rules="required|max:100"
              flat
              label="Add A Title"
              data-vv-name="title"
              name="idea_title"
              class="idea-name-field"
              :single-line="true"
              :solo="true"
              @keyup.13="focusIdeaText"
            ></v-text-field-with-validation>

            <!-- Description = trix editor -->
            <div class="ideaEditor">
              <client-only>
                <trix-wrapper
                  v-model="contents"
                  class="editor"
                  placeholder="Just start typing your idea here! You can add formatting with the toolbar below."
                  :auto-delete-attachments="true"
                  @attachmentsUploadStarted="onAttachmentsUploadStarted"
                  @attachmentsUploadCompleted="onAttachmentsUploadCompleted"
                  @fileAttached="onFileAttached"
                  @fileRemoved="onFileRemoved"
                />
              </client-only>
            </div>

            <!-- Tags -->
            <v-combobox
              v-if="false"
              v-model="chips"
              rules="max:100"
              flat
              data-vv-name="tag"
              class="ideaTag idea-tags-field"
              chips
              clearable
              multiple
              label="Add Tags"
              solo
            >
              <template v-slot:selection="{ attrs, item, select, selected }">
                <v-chip
                  v-bind="attrs"
                  :input-value="selected"
                  close
                  small
                  color="secondary"
                  @click="select"
                  @click:close="remove(item)"
                >
                  <strong>{{ item }}</strong>
                </v-chip>
              </template>
            </v-combobox>
          </v-col>
        </v-row>

        <v-row id="submitButtonRow">
          <v-col cols="12" sm="10" md="8" lg="6">
            <!-- Submit -->
            <div>
              <v-btn
                rounded
                block
                :loading="creatingIdea"
                :disabled="!valid || !validated"
                @click="handleSubmit(onCreateIdea)"
                >Submit</v-btn
              >
            </div>
          </v-col>
        </v-row>
      </v-container>
    </validation-observer>

    <!-- Bottom snackbar message -->
    <v-snackbar
      v-model="snackbarVisible"
      :timeout="2000"
      :color="snackbarColor"
    >
      {{ snackbarMessage }}
      <v-btn color="white" text @click="snackbarVisible = false">
        Close
      </v-btn>
    </v-snackbar>
  </Layout>
</template>
<script>
import { mapMutations } from 'vuex'
import { ValidationObserver } from 'vee-validate'
import { graphqlOperation } from '@aws-amplify/api'
import TrixWrapper from '@/components/TrixWrapper'
import Layout from '@/components/layout/Layout'
import createIdea from '~/graphql/mutations/createIdea'
import VTextFieldWithValidation from '@/components/validation/VTextFieldWithValidation'

export default {
  components: {
    VTextFieldWithValidation,
    Layout,
    TrixWrapper,
    ValidationObserver
  },
  middleware: 'authenticated',
  data: () => ({
    mobileHeaderUiOptions: {
      pageTitle: 'SUBMIT AN IDEA',
      leftButtonType: 'back'
    },
    contents: '',
    title: '',
    imageAttachments: [],
    fileAttachments: [],
    creatingIdea: false,
    chips: [],
    // sjahj: true,
    errorMsg: null,

    snackbarVisible: false,
    snackbarMessage: '',
    snackbarColor: 'success',
    uploadingAttachment: false
  }),
  computed: {
    allowCreateIdea() {
      return this.title && !this.uploadingAttachment
    }
  },
  created() {},

  mounted() {},

  methods: {
    ...mapMutations({
      updateCreatedIdea: 'ideas/UPDATE_CREATED'
    }),

    focusIdeaText() {
      document.querySelector('trix-editor').focus()
    },
    onAttachmentsUploadStarted() {
      this.uploadingAttachment = true
    },
    onAttachmentsUploadCompleted() {
      this.uploadingAttachment = false
    },
    onFileAttached({ type, key }) {
      if (type.substr(0, 5) === 'image') {
        this.imageAttachments.push(key)
      }
      this.fileAttachments.push(key)
    },
    onFileRemoved({ type, key }) {
      if (type.substr(0, 5) === 'image') {
        this.imageAttachments.splice(this.imageAttachments.indexOf(key), 1)
      }
      this.fileAttachments.splice(this.fileAttachments.indexOf(key), 1)
    },
    remove(item) {
      this.chips.splice(this.chips.indexOf(item), 1)
      this.chips = [...this.chips]
    },
    async onCreateIdea() {
      this.creatingIdea = true

      try {
        const result = await this.$amplifyApi.graphql(
          graphqlOperation(createIdea, {
            content: this.contents,
            title: this.title,
            tags: this.chips,
            fileAttachments: this.fileAttachments,
            imageAttachments: this.imageAttachments
          })
        )

        this.creatingIdea = false
        this.ideaEditorVisible = false

        this.snackbarMessage = 'Idea Created'
        this.snackbarColor = 'success'
        this.snackbarVisible = true

        // Redirect to idea deail page
        const { shortId, slug, ideaId } = result.data.createIdea
        this.$router.push({
          name: 'i-shortId-slug',
          params: { shortId, slug },
          force: true
        })
        this.updateCreatedIdea(ideaId)
      } catch (err) {
        this.creatingIdea = false
        this.snackbarMessage = 'Something went wrong!!'
        this.snackbarColor = 'error'
        this.snackbarVisible = true
        // console.error(err)
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.createIdeaPage ::v-deep {
  .idea-name-field {
    font-size: 24px;
    font-weight: 900;

    .v-input__slot {
      margin: 0 !important;
      padding: 0 !important;
      label {
        color: #c1b8c6;
        font-size: 24px;
      }
    }
    .v-text-field__details {
      padding-left: 0px !important;
    }
  }

  .idea-tags-field {
    .v-input__slot {
      label {
        color: #c1b8c6;
      }
    }
  }

  .ideaEditor {
    .editor {
      trix-editor {
        border: none;
        padding: 0;
        &:active,
        &:focus {
          border: none;
        }
        &:empty::before {
          font-size: 18px;
          color: #c1b8c6;
          content: attr(placeholder);
          position: relative;
          top: -2px;
        }
      }
      .trix-content {
        height: 32vh;
        overflow-y: auto;
      }
    }
  }
  .errorMsg {
    color: #b71c1c;
    font-size: 12px;
    margin-top: 2px;
    padding-left: 10px;
  }

  .ideaTag {
    padding-top: 30px;

    .v-chip {
      background-color: rgba(192, 183, 197);
      color: white;

      i {
        color: white;
      }
    }

    .v-icon.mdi-menu-down {
      display: none;
    }

    .v-input__append-inner {
      display: none;
    }
  }
  /* stuff for making the toolbar stuck to the bottom */
  .trix-button-row {
    @media (min-width: $screen-md-min) {
      position: fixed;
      bottom: 0;
    }
  }

  #toggle-trix-panel {
    margin-bottom: 4px;
  }

  /* stuff for computing the required height of the idea content entry part  */
  /* header height: 56 on mobile viewports, 64 on wide viewports  */

  $header-height-wide: 64px;
  $header-height-narrow: 56px;
  $main-content-container-vertical-padding: 12px;
  $submit-button-row-height: 60px;
  $tags-height: 0px; /* not used, but needed when we bring back tags */
  $trix-button-row-height: 35px;

  #ideaContentRow {
    @media (min-width: $screen-md-min) {
      height: calc(
        100vh - #{$header-height-wide} - 2 * #{$main-content-container-vertical-padding} -
          #{$submit-button-row-height} - #{$trix-button-row-height}
      );
    }
    @media (max-width: $screen-sm-max) {
      height: calc(
        100vh - #{$header-height-narrow} - 2 * #{$main-content-container-vertical-padding} -
          #{$submit-button-row-height} - #{$trix-button-row-height}
      );
    }
  }
}
</style>
