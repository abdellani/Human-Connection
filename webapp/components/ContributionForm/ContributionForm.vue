<template>
  <ds-form
    class="contribution-form"
    ref="contributionForm"
    v-model="form"
    :schema="formSchema"
    @submit="submit"
  >
    <template slot-scope="{ errors }">
      <hc-teaser-image
        :contribution="contribution"
        @addTeaserImage="addTeaserImage"
        :class="{ '--blur-image': form.blurImage }"
        @addImageAspectRatio="addImageAspectRatio"
      >
        <img
          v-if="contribution"
          class="contribution-image"
          :src="contribution.image | proxyApiUrl"
        />
      </hc-teaser-image>

      <ds-card>
        <div class="blur-toggle">
          <label for="blur-img">{{ $t('contribution.inappropriatePicture') }}</label>
          <input type="checkbox" id="blur-img" v-model="form.blurImage" />
          <p>
            <a
              href="https://support.human-connection.org/kb/faq.php?id=113"
              target="_blank"
              class="link"
            >
              {{ $t('contribution.inappropriatePictureText') }}
              <ds-icon name="question-circle" />
            </a>
          </p>
        </div>

        <ds-space />
        <client-only>
          <user-teaser :user="currentUser" />
        </client-only>
        <ds-space />
        <ds-input
          model="title"
          class="post-title"
          :placeholder="$t('contribution.title')"
          name="title"
          autofocus
        />
        <ds-text align="right">
          <ds-chip v-if="errors && errors.title" color="danger" size="base">
            {{ form.title.length }}/{{ formSchema.title.max }}
            <ds-icon name="warning"></ds-icon>
          </ds-chip>
          <ds-chip v-else size="base">{{ form.title.length }}/{{ formSchema.title.max }}</ds-chip>
        </ds-text>
        <hc-editor
          :users="users"
          :value="form.content"
          :hashtags="hashtags"
          @input="updateEditorContent"
        />
        <ds-text align="right">
          <ds-chip v-if="errors && errors.content" color="danger" size="base">
            {{ contentLength }}
            <ds-icon name="warning"></ds-icon>
          </ds-chip>
          <ds-chip v-else size="base">
            {{ contentLength }}
          </ds-chip>
        </ds-text>
        <ds-space margin-bottom="small" />
        <hc-categories-select model="categoryIds" :existingCategoryIds="form.categoryIds" />
        <ds-text align="right">
          <ds-chip v-if="errors && errors.categoryIds" color="danger" size="base">
            {{ form.categoryIds.length }} / 3
            <ds-icon name="warning"></ds-icon>
          </ds-chip>
          <ds-chip v-else size="base">{{ form.categoryIds.length }} / 3</ds-chip>
        </ds-text>
        <ds-flex class="contribution-form-footer">
          <ds-flex-item :width="{ lg: '50%', md: '50%', sm: '100%' }" />
          <ds-flex-item>
            <ds-space margin-bottom="small" />
            <ds-select
              model="language"
              :options="languageOptions"
              icon="globe"
              :placeholder="$t('contribution.languageSelectText')"
              :label="$t('contribution.languageSelectLabel')"
            />
          </ds-flex-item>
        </ds-flex>
        <ds-text align="right">
          <ds-chip v-if="errors && errors.language" size="base" color="danger">
            <ds-icon name="warning"></ds-icon>
          </ds-chip>
        </ds-text>

        <ds-space />
        <div slot="footer" style="text-align: right">
          <base-button data-test="cancel-button" :disabled="loading" @click="$router.back()" danger>
            {{ $t('actions.cancel') }}
          </base-button>
          <base-button type="submit" icon="check" :loading="loading" :disabled="errors" filled>
            {{ $t('actions.save') }}
          </base-button>
        </div>
        <ds-space margin-bottom="large" />
      </ds-card>
    </template>
  </ds-form>
</template>

<script>
import gql from 'graphql-tag'
import orderBy from 'lodash/orderBy'
import { mapGetters } from 'vuex'
import HcEditor from '~/components/Editor/Editor'
import locales from '~/locales'
import PostMutations from '~/graphql/PostMutations.js'
import HcCategoriesSelect from '~/components/CategoriesSelect/CategoriesSelect'
import HcTeaserImage from '~/components/TeaserImage/TeaserImage'
import UserTeaser from '~/components/UserTeaser/UserTeaser'

export default {
  components: {
    HcEditor,
    HcCategoriesSelect,
    HcTeaserImage,
    UserTeaser,
  },
  props: {
    contribution: { type: Object, default: () => {} },
  },
  data() {
    const languageOptions = orderBy(locales, 'name').map(locale => {
      return { label: locale.name, value: locale.code }
    })

    const formDefaults = {
      title: '',
      content: '',
      teaserImage: null,
      imageAspectRatio: null,
      image: null,
      language: null,
      categoryIds: [],
      blurImage: false,
    }

    let id = null
    let slug = null
    const form = { ...formDefaults }
    if (this.contribution && this.contribution.id) {
      id = this.contribution.id
      slug = this.contribution.slug
      form.title = this.contribution.title
      form.content = this.contribution.content
      form.image = this.contribution.image
      form.language =
        this.contribution && this.contribution.language
          ? languageOptions.find(o => this.contribution.language === o.value)
          : null
      form.categoryIds = this.categoryIds(this.contribution.categories)
      form.imageAspectRatio = this.contribution.imageAspectRatio
      form.blurImage = this.contribution.imageBlurred
    }

    return {
      form,
      formSchema: {
        title: { required: true, min: 3, max: 100 },
        content: { required: true },
        categoryIds: {
          type: 'array',
          required: true,
          validator: (rule, value) => {
            const errors = []
            if (!(value && value.length >= 1 && value.length <= 3)) {
              errors.push(new Error(this.$t('common.validations.categories')))
            }
            return errors
          },
        },
        language: { required: true },
        blurImage: { required: false },
      },
      languageOptions,
      id,
      slug,
      loading: false,
      users: [],
      contentMin: 3,
      hashtags: [],
      elem: null,
    }
  },
  computed: {
    contentLength() {
      return this.$filters.removeHtml(this.form.content).length
    },
    ...mapGetters({
      currentUser: 'auth/user',
    }),
  },
  methods: {
    submit() {
      const {
        language: { value: language },
        title,
        content,
        image,
        teaserImage,
        imageAspectRatio,
        categoryIds,
        blurImage,
      } = this.form
      this.loading = true
      this.$apollo
        .mutate({
          mutation: this.id ? PostMutations().UpdatePost : PostMutations().CreatePost,
          variables: {
            id: this.id,
            title,
            content,
            categoryIds,
            language,
            image,
            imageUpload: teaserImage,
            imageBlurred: blurImage,
            imageAspectRatio,
          },
        })
        .then(({ data }) => {
          this.loading = false
          this.$toast.success(this.$t('contribution.success'))
          const result = data[this.id ? 'UpdatePost' : 'CreatePost']

          this.$router.push({
            name: 'post-id-slug',
            params: { id: result.id, slug: result.slug },
          })
        })
        .catch(err => {
          this.$toast.error(err.message)
          this.loading = false
        })
    },
    updateEditorContent(value) {
      this.$refs.contributionForm.update('content', value)
    },
    addTeaserImage(file) {
      this.form.teaserImage = file
    },
    addImageAspectRatio(aspectRatio) {
      this.form.imageAspectRatio = aspectRatio
    },
    categoryIds(categories) {
      return categories.map(c => c.id)
    },
  },
  apollo: {
    User: {
      query() {
        return gql`
          query {
            User(orderBy: slug_asc) {
              id
              slug
            }
          }
        `
      },
      result({ data: { User } }) {
        this.users = User
      },
    },
    Tag: {
      query() {
        return gql`
          query {
            Tag(orderBy: id_asc) {
              id
            }
          }
        `
      },
      result({ data: { Tag } }) {
        this.hashtags = Tag
      },
    },
  },
}
</script>

<style lang="scss">
.contribution-form {
  .ds-card-image.--blur-image img {
    filter: blur(32px);
  }

  .blur-toggle {
    text-align: right;

    > .link {
      display: block;
    }
  }

  .ds-chip {
    cursor: default;
  }

  .post-title {
    margin-top: $space-x-small;
    margin-bottom: $space-xx-small;

    input {
      border: 0;
      font-size: $font-size-x-large;
      font-weight: bold;
      padding-left: 0;
      padding-right: 0;
    }
  }
}
</style>
