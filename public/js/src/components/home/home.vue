<template>

  <div class='home'>
      <Ad class='ad-container'/>
    <div class='home_info'>
      <h2>Your Feed</h2>
      <!-- <span>{{ feeds_len }} Feeds</span> -->
      <router-link :to='{ name: "create-post-home", params: { username: session.username } }' class='pri_btn'>Create Post</router-link>
    </div>
    <Feeds />


    <router-view name='overlay' ></router-view>

    <transition-group name='fade' >
      <router-view name='create-post-home' key='create-post-home' ></router-view>
    </transition-group>
  </div>

</template>

<script>
import moduleMixin from '../../mixins/module-mixin'
import userMixin from '../../mixins/user-mixin'
import Feeds from './feeds.vue'
import Ad from './ad.vue'

export default {
  mixins: [
    userMixin,
    moduleMixin
  ],
  components: {
    'Feeds': Feeds,
    'Ad': Ad
  },
  computed: {
    feeds_len(){
      let len = this.p.feeds.length
      return len == 0 ? 'No' : len
    }
  },
  created() {
    this.$store.dispatch('getFeeds')
  }
}
</script>
