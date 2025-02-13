<script>
import TutorialBanner from '@/components/TutorialBanner'

export default {
  components: { TutorialBanner },
  props: {
    hideBanners: {
      type: Boolean,
      required: false,
      default: () => false
    },
    icon: {
      type: String,
      required: false,
      default: () => null
    },
    pageType: {
      type: String,
      required: false,
      default: () => null
    }
  },
  data() {
    return {
      pageScrolled: false
    }
  },
  methods: {
    scrolled() {
      this.pageScrolled = window.scrollY > 30
    }
  }
}
</script>

<template>
  <div>
    <TutorialBanner v-if="!hideBanners" :page-scroll="pageScrolled" />
    <v-toolbar
      v-scroll="scrolled"
      :elevation="
        pageScrolled && (!$slots.tabs || $vuetify.breakpoint.smAndDown)
          ? '4'
          : '0'
      "
      height="auto"
      color="appBackground"
      class="pb-3"
      style="
    position: fixed;
    width: 100%;
    z-index: 6;"
    >
      <v-row
        no-gutters
        class="d-flex align-center mx-auto"
        :class="{
          'justify-center': $vuetify.breakpoint.smAndDown
        }"
        style="height: 64px;"
      >
        <v-col :sm="$slots['page-actions'] ? 6 : 12" class="d-flex align-end">
          <div class="mr-2">
            <v-icon x-large color="blue-grey lighten-4">{{ icon }}</v-icon>
          </div>
          <div>
            <div>
              <span v-if="$slots.breadcrumbs" style="font-size: 0.875rem;">
                <slot name="breadcrumbs"></slot>
              </span>
              <slot v-if="$slots['page-type']" name="page-type"></slot>
              <span v-else class="text-overline">{{ pageType }}</span>
            </div>
            <div class="text-h5 mt-n1">
              <slot name="page-title"></slot>
            </div>
          </div>
        </v-col>
        <v-col
          v-if="$slots['page-actions'] && $vuetify.breakpoint.smAndUp"
          cols="12"
          sm="6"
          class="align-self-end text-right"
          :class="{
            'text-center': $vuetify.breakpoint.smAndDown
          }"
        >
          <slot name="page-actions"></slot>
        </v-col>
      </v-row>
      <template
        v-if="$slots.tabs && $vuetify.breakpoint.mdAndUp"
        slot="extension"
        ><slot name="tabs"></slot
      ></template>
      <template
        v-if="$slots['page-actions'] && $vuetify.breakpoint.xsOnly"
        slot="extension"
        ><slot name="page-actions"></slot
      ></template>
    </v-toolbar>
  </div>
</template>
