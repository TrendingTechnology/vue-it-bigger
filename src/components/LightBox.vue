<template>
  <div @click.stop="closeLightBox">
    <transition
      mode="out-in"
      name="vue-lb-container-transition"
      @afterEnter="enableImageTransition"
      @beforeLeave="disableImageTransition"
    >
      <div
        v-if="media && media.length > 0"
        v-show="lightBoxOn"
        ref="container"
        class="vue-lb-container"
      >
        <div
          class="vue-lb-content"
          @click.stop
        >
          <transition
            mode="out-in"
            :name="imageTransitionName"
          >
            <img
              v-if="media[select].type !== 'video'"
              :key="media[select].src"
              :src="media[select].src"
              :srcset="media[select].srcset || ''"
              class="vue-lb-image"
              :alt="media[select].caption"
            >
            <video
              v-else
              ref="video"
              controls
              :width="media[select].width"
              :height="media[select].height"
              :autoplay="media[select].autoplay"
            >
              <source
                v-for="source in media[select].sources"
                :key="source.src"
                :src="source.src"
                :type="source.type"
              >
            </video>
          </transition>
        </div> <!-- .vue-lb-content -->

        <div
          v-if="showThumbs"
          class="vue-lb-thumbnail-wrapper vue-lb-hideable"
          :class="{ 'vue-lb-hidden': interactionIsIdle }"
          @click.stop
        >
          <div
            v-for="(image, index) in imagesThumb"
            v-show="index >= thumbIndex.begin && index <= thumbIndex.end"
            :key="typeof image.thumb === 'string' ? `${image.thumb}${index}` : index"
            :style="{ backgroundImage: 'url(' + image.thumb + ')' }"
            :class="'vue-lb-thumbnail' + (select === index ? '-active' : '')"
            @click.stop="showImage(index)"
          >
            <slot
              v-if="image.type"
              name="videoIcon"
            >
              <VideoIcon />
            </slot>
          </div>
        </div> <!-- .vue-lb-thumbnail-wrapper -->

        <div
          class="vue-lb-footer vue-lb-hideable"
          :class="{ 'vue-lb-hidden': interactionIsIdle }"
        >
          <slot name="customCaption">
            <div
              v-show="showCaption"
              v-html="media[select].caption"
            />
          </slot>

          <div class="vue-lb-footer-count">
            <slot
              name="footer"
              :current="select + 1"
              :total="media.length"
            >
              {{ select + 1 }} / {{ media.length }}
            </slot>
          </div>
        </div>

        <button
          v-if="closable"
          type="button"
          :title="closeText"
          class="vue-lb-close vue-lb-hideable"
          :class="{ 'vue-lb-hidden': interactionIsIdle }"
        >
          <slot name="close">
            <CloseIcon />
          </slot>
        </button>

        <button
          v-if="media.length > 1"
          type="button"
          class="vue-lb-arrow vue-lb-arrow-left vue-lb-hideable"
          :class="{ 'vue-lb-hidden': interactionIsIdle }"
          :title="previousText"
          @click.stop="previousImage()"
        >
          <slot name="previous">
            <LeftArrowIcon />
          </slot>
        </button>

        <button
          v-if="media.length > 1"
          type="button"
          class="vue-lb-arrow vue-lb-arrow-right vue-lb-hideable"
          :class="{ 'vue-lb-hidden': interactionIsIdle }"
          :title="nextText"
          @click.stop="nextImage()"
        >
          <slot name="next">
            <RightArrowIcon />
          </slot>
        </button>
      </div> <!-- .vue-lb-container -->
    </transition>
  </div>
</template>

<script>
import LeftArrowIcon from './LeftArrowIcon'
import RightArrowIcon from './RightArrowIcon'
import CloseIcon from './CloseIcon'
import VideoIcon from './VideoIcon'

let Hammer

if (typeof window !== 'undefined') {
  Hammer = require('hammerjs')
}

export default {
  components: {
    LeftArrowIcon,
    RightArrowIcon,
    CloseIcon,
    VideoIcon,
  },

  props: {
    media: {
      type: Array,
      required: true,
    },

    disableScroll: {
      type: Boolean,
      default: true,
    },

    showLightBox: {
      type: Boolean,
      default: true,
    },

    closable: {
      type: Boolean,
      default: true,
    },

    startAt: {
      type: Number,
      default: 0,
    },

    nThumbs: {
      type: Number,
      default: 7,
    },

    showThumbs: {
      type: Boolean,
      default: true,
    },

    // Mode
    autoPlay: {
      type: Boolean,
      default: false,
    },

    autoPlayTime: {
      type: Number,
      default: 3000,
    },

    showCaption: {
      type: Boolean,
      default: false,
    },

    lengthToLoadMore: {
      type: Number,
      default: 0
    },

    closeText: {
      type: String,
      default: 'Close (Esc)'
    },

    previousText: {
      type: String,
      default: 'Previous',
    },

    nextText: {
      type: String,
      default: 'Next',
    },

    previousThumbText: {
      type: String,
      default: 'Previous'
    },

    nextThumbText: {
      type: String,
      default: 'Next'
    },
  },

  data() {
    return {
      select: this.startAt,
      lightBoxOn: this.showLightBox,
      interactionIsIdle: false,
      imageTransitionName: 'vue-lb-image-no-transition',
      timer: null,
      interactionTimer: null,
    }
  },

  computed: {
    thumbIndex() {
      const halfDown = Math.floor(this.nThumbs / 2)

      if (this.select >= halfDown && this.select < this.media.length - halfDown)
        return {
          begin: this.select - halfDown + (1 - this.nThumbs % 2),
          end: this.select + halfDown,
        }

      if (this.select < halfDown)
        return {
          begin: 0,
          end: this.nThumbs - 1,
        }

      return {
        begin: this.media.length - this.nThumbs,
        end: this.media.length - 1,
      }
    },

    imagesThumb() {
      return this.media.map(({ thumb, type }) => ({ thumb, type }))
    },
  },

  watch: {
    lightBoxOn(value) {
      if (document != null) {
        this.onToggleLightBox(value)
      }
    },

    select() {
      this.$emit('onImageChanged', this.select)

      if (this.select >= this.media.length - this.lengthToLoadMore - 1)
        this.$emit('onLoad')

      if (this.select === this.media.length - 1)
        this.$emit('onLastIndex')

      if (this.select === 0)
        this.$emit('onFirstIndex')

      if (this.select === this.startAt)
        this.$emit('onStartIndex')
    },
  },

  mounted() {
    if (this.autoPlay) {
      this.timer = setInterval(() => {
        this.nextImage()
      }, this.autoPlayTime)
    }

    this.onToggleLightBox(this.lightBoxOn)

    if (this.$refs.container) {
      const hammer = new Hammer(this.$refs.container)

      hammer.on('swiperight', () => {
        this.previousImage()
      })

      hammer.on('swipeleft', () => {
        this.nextImage()
      })

      this.$refs.container.addEventListener('mousedown', this.handleMouseActivity);
      this.$refs.container.addEventListener('mousemove', this.handleMouseActivity);
      this.$refs.container.addEventListener('touchmove', this.handleMouseActivity);
    }
  },

  beforeDestroy() {
    document.removeEventListener('keydown', this.addKeyEvent)

    if (this.autoPlay) {
      clearInterval(this.timer)
    }

    if (this.$refs.container) {
      this.$refs.container.removeEventListener('mousedown', this.handleMouseActivity);
      this.$refs.container.removeEventListener('mousemove', this.handleMouseActivity);
      this.$refs.container.removeEventListener('touchmove', this.handleMouseActivity);
    }
  },

  methods: {
    onLightBoxOpen() {
      this.$emit('onOpened')

      if (this.disableScroll) {
        document.querySelector('html').classList.add('no-scroll')
      }

      document.querySelector('body').classList.add('vue-lb-open')
      document.addEventListener('keydown', this.addKeyEvent)

      if (this.$refs.video && this.$refs.video.autoplay) {
        this.$refs.video.play()
      }
    },

    onLightBoxClose() {
      this.$emit('onClosed')

      if (this.disableScroll) {
        document.querySelector('html').classList.remove('no-scroll')
      }

      document.querySelector('body').classList.remove('vue-lb-open')
      document.removeEventListener('keydown', this.addKeyEvent)

      if (this.$refs.video) {
        this.$refs.video.pause()
        this.$refs.video.currentTime = '0'
      }
    },

    onToggleLightBox(value) {
      if (value) this.onLightBoxOpen()
      else this.onLightBoxClose()
    },

    showImage(index) {
      this.select = index
      this.interactionIsIdle = false
      this.lightBoxOn = true
    },

    addKeyEvent(event) {
      if (event.keyCode === 37) this.previousImage() // left arrow
      if (event.keyCode === 39) this.nextImage() // right arrow
      if (event.keyCode === 27) this.closeLightBox() // esc
    },

    closeLightBox() {
      if (this.$refs.video)
        this.$refs.video.pause();
      if (!this.closable) return;
      this.$set(this, 'lightBoxOn', false)
    },

    nextImage() {
      this.$set(this, 'select', (this.select + 1) % this.media.length)
    },

    previousImage() {
      this.$set(this, 'select', (this.select + this.media.length - 1) % this.media.length)
    },

    enableImageTransition() {
      this.handleMouseActivity()
      this.imageTransitionName = 'vue-lb-image-transition'
    },

    disableImageTransition() {
      this.imageTransitionName = 'vue-lb-image-no-transition'
    },

    handleMouseActivity() {
      clearTimeout(this.interactionTimer);

      if (this.interactionIsIdle) {
        this.interactionIsIdle = false
      }

      this.interactionTimer = setTimeout(() => {
          this.interactionIsIdle = true
      }, 3000);
    }
  },
}
</script>

<style src="./style.css">
</style>
