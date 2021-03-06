<template>
  <transition :name="disableTransition ? null : 'fade'">
    <component
      v-if="layout"
      :is="layout"
      :slot-key="slotKey || 'default'"
    />
    <div v-else class="conent"></div>
  </transition>
</template>

<script>
/* global CONTENT_LOADING */

import Vue from 'vue'
import ContentLoading from './ContentLoading'

const CONTENT_LOADING_COMPONENT = typeof CONTENT_LOADING === 'string'
  ? CONTENT_LOADING
  : 'ContentLoading'

export default {
  components: { ContentLoading },

  props: {
    pageKey: String,
    slotKey: String
  },

  data () {
    return {
      layout: CONTENT_LOADING_COMPONENT,
      noTransition: true
    }
  },

  computed: {
    $key () {
      return this.pageKey || this.$page.key
    },
    disableTransition () {
      return !this.layout || this.layout === CONTENT_LOADING_COMPONENT || this.noTransition
    }
  },

  created () {
    this.loadContent(this.$key)
    this.$vuepress.$on('AsyncMarkdownContentMounted', (slotKey) => {
      this.$vuepress.$set('contentMounted', true)
    })
  },

  watch: {
    $key (key) {
      this.$vuepress.$set('contentMounted', false)
      this.reloadContent(key)
    }
  },

  methods: {
    loadContent (pageKey) {
      this.layout = null
      if (this.$vuepress.isPageExists(pageKey)) {
        if (!this.$ssrContext) {
          this.$vuepress.registerPageAsyncComponent(pageKey)
          this.layout = pageKey
        }
      }
    },

    reloadContent (pageKey) {
      // When page has been loaded, disable transition.
      if (this.$vuepress.isPageLoaded(pageKey)) {
        this.layout = pageKey
        this.noTransition = true
        return
      }
      // Start to load unfetched page component.
      this.layout = CONTENT_LOADING_COMPONENT
      if (this.$vuepress.isPageExists(pageKey)) {
        this.noTransition = false
        if (!this.$ssrContext) {
          Promise.all([
            this.$vuepress.loadPageAsyncComponent(pageKey),
            new Promise(resolve => setTimeout(resolve, 300))
          ]).then(([comp]) => {
            this.$vuepress.$emit('AsyncMarkdownAssetLoaded', this.pageKey)
            Vue.component(pageKey, comp.default)
            this.layout = null
            setTimeout(() => {
              this.layout = pageKey
            })
          })
        }
      }
    }
  }
}
</script>

<style>
  .fade-enter-active, .fade-leave-active {
    transition: opacity .2s;
  }
  .fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
    opacity: 0;
  }
</style>
