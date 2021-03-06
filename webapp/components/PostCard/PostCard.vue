<template>
  <ds-card
    :lang="post.language"
    :image="post.image | proxyApiUrl"
    :class="{
      'post-card': true,
      'disabled-content': post.disabled,
      '--pinned': isPinned,
      '--blur-image': post.imageBlurred,
    }"
  >
    <!-- Post Link Target -->
    <nuxt-link
      class="post-link"
      :to="{ name: 'post-id-slug', params: { id: post.id, slug: post.slug } }"
    >
      {{ post.title }}
    </nuxt-link>
    <ds-space margin-bottom="small" />
    <!-- Username, Image & Date of Post -->
    <div class="user-wrapper">
      <client-only>
        <hc-user :user="post.author" :trunc="35" :date-time="post.createdAt" />
      </client-only>
      <hc-ribbon v-if="isPinned" class="ribbon--pinned" :text="$t('post.pinned')" />
      <hc-ribbon v-else :text="$t('post.name')" />
    </div>
    <ds-space margin-bottom="small" />
    <!-- Post Title -->
    <ds-heading tag="h3" no-margin class="hyphenate-text">{{ post.title }}</ds-heading>
    <ds-space margin-bottom="small" />
    <!-- Post Content Excerpt -->
    <!-- eslint-disable vue/no-v-html -->
    <!-- TODO: replace editor content with tiptap render view -->
    <div class="hc-editor-content hyphenate-text" v-html="excerpt" />
    <!-- eslint-enable vue/no-v-html -->
    <!-- Footer o the Post -->
    <template slot="footer">
      <div style="display: inline-block; opacity: .5;">
        <!-- Categories -->
        <hc-category
          v-for="category in post.categories"
          :key="category.id"
          v-tooltip="{
            content: $t(`contribution.category.name.${category.slug}`),
            placement: 'bottom-start',
            delay: { show: 500 },
          }"
          :icon="category.icon"
        />
      </div>
      <client-only>
        <div style="display: inline-block; float: right">
          <!-- Shouts Count -->
          <span :style="{ opacity: post.shoutedCount ? 1 : 0.5 }">
            <base-icon name="bullhorn" />
            <small>{{ post.shoutedCount }}</small>
          </span>
          &nbsp;
          <!-- Comments Count -->
          <span :style="{ opacity: post.commentsCount ? 1 : 0.5 }">
            <base-icon name="comments" />
            <small>{{ post.commentsCount }}</small>
          </span>
          <!-- Menu -->
          <content-menu
            resource-type="contribution"
            :resource="post"
            :modalsData="menuModalsData"
            :is-owner="isAuthor"
            @pinPost="pinPost"
            @unpinPost="unpinPost"
          />
        </div>
      </client-only>
    </template>
  </ds-card>
</template>

<script>
import HcUser from '~/components/User/User'
import ContentMenu from '~/components/ContentMenu/ContentMenu'
import HcCategory from '~/components/Category'
import HcRibbon from '~/components/Ribbon'
// import { randomBytes } from 'crypto'
import { mapGetters } from 'vuex'
import { postMenuModalsData, deletePostMutation } from '~/components/utils/PostHelpers'

export default {
  name: 'HcPostCard',
  components: {
    HcUser,
    HcCategory,
    HcRibbon,
    ContentMenu,
  },
  props: {
    post: {
      type: Object,
      required: true,
    },
    width: {
      type: Object,
      default: () => {},
    },
  },
  mounted() {
    const width = this.$el.offsetWidth
    const height = Math.min(width / this.post.imageAspectRatio, 2000)
    const imageElement = this.$el.querySelector('.ds-card-image')
    if (imageElement) {
      imageElement.style.height = `${height}px`
    }
  },
  computed: {
    ...mapGetters({
      user: 'auth/user',
    }),
    excerpt() {
      return this.$filters.removeLinks(this.post.contentExcerpt)
    },
    isAuthor() {
      const { author } = this.post
      if (!author) return false
      return this.user.id === this.post.author.id
    },
    menuModalsData() {
      return postMenuModalsData(
        // "this.post" may not always be defined at the beginning …
        this.post ? this.$filters.truncate(this.post.title, 30) : '',
        this.deletePostCallback,
      )
    },
    isPinned() {
      return this.post && this.post.pinnedBy
    },
  },
  methods: {
    async deletePostCallback() {
      try {
        const {
          data: { DeletePost },
        } = await this.$apollo.mutate(deletePostMutation(this.post.id))
        this.$toast.success(this.$t('delete.contribution.success'))
        this.$emit('removePostFromList', DeletePost)
      } catch (err) {
        this.$toast.error(err.message)
      }
    },
    pinPost(post) {
      this.$emit('pinPost', post)
    },
    unpinPost(post) {
      this.$emit('unpinPost', post)
    },
  },
}
</script>
<style lang="scss">
.post-card {
  justify-content: space-between;
  position: relative;
  z-index: 1;
  cursor: pointer;

  &.--pinned {
    border: 1px solid $color-warning;
  }

  &.--blur-image > .ds-card-image img {
    filter: blur(22px);
  }

  > .ds-card-image img {
    width: 100%;
    max-height: 2000px;
    object-fit: contain;
  }

  > .ds-card-content {
    flex-grow: 0;
  }

  /* workaround to avoid jumping layout when footer is rendered */
  > .ds-card-footer {
    height: 75px;
  }

  /* workaround to avoid jumping layout when hc-user is rendered */
  .user-wrapper {
    height: 36px;
  }

  .content-menu {
    display: inline-block;
    margin-left: $space-xx-small;
    margin-right: -$space-x-small;
  }

  .post-link {
    margin: 15px;
    display: block;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    text-indent: -999999px;
  }
}
</style>
