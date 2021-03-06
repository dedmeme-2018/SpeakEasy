<template>
  <div>

    <Overlay type='white' />

    <div class='view_note modal'>
      <div class='v_n_header modal_header'>
        <span class='title'>View post</span>
        <Goto />
      </div>
      <div class='v_n_middle modal_middle'>
        <div class='v_n_info'>
          <Avatar :AvatarID='this.post.user'/>
          <div class='v_n_left'>
            <router-link :to='{ name: "profile", params: { username: post.username } }' class='v_n_username'>{{ post.username }}</router-link>
            <span class='v_n_time'>{{ post.post_created | timeAgo }}</span>
          </div>
        </div>
        <span class='v_n_title' :contenteditable='editing' spellCheck='false'>{{ post.title }}</span>
        <span class='v_n_content' :contenteditable='editing' spellCheck='false'>{{ post.content }}</span>
        <img v-if="hasPhoto" class='v_n_photo' :src="photoSrc" />
        <br>
      </div>
      <div class='v_n_bottom modal_bottom'>
        <div class="v_n_int">
          <span v-if='liked' class='v_n_unlike like_unlike' @click='unlike' ><i class="material-icons">favorite</i></span>
          <span v-else class='v_n_like like_unlike' @click='like' ><i class="material-icons">favorite_border</i></span>
          <router-link :to='{ name: "likes", params: { post: post.post_id } }' class="sec_btn v_n_likes">{{ likes_len }} Likes</router-link>
        </div>
        <template v-if='session.id == post.user' >
          <a v-if='editing' href='#' class='v_n_edit sec_btn' @click.prevent='editPost'>Done Editing</a>
          <a v-if='!editing' href='#' class='v_n_edit sec_btn' @click.prevent='_toggle("editing")'>Edit Post</a>
          <a href='#' class='v_n_delete sec_btn' :class='{sec_btn_disabled: editing}' @click.prevent='_toggle("deleting")'>Delete Post</a>
        </template>


        <CommentList />

        <ui-textbox placeholder="Add a comment" v-model="user_comment"></ui-textbox>
        <ui-button color="green" type="secondary" v-on:click="postComment">Post Comment</ui-button>

        <a href='#' class='v_n_cancel pri_btn' :class='{a_disabled: editing}' @click.prevent='Back' >Done</a>
      </div>
    </div>


    <Overlay
    v-if='deleting'
    type='black'
    />

    <Prompt
    v-if='deleting'
    title='Delete post'
    content="This post will be deleted. There's no undo so you won't be able to find it."
    actionText='Delete'
    @back='_toggle("deleting")'
    @action='deletePost'
    />

    <router-view name='overlay' ></router-view>
    <transition name='fade' >
      <router-view name='likes' ></router-view>
    </transition>

  </div>
</template>

<script>
import $ from 'jquery'
import Notify from 'handy-notification'
import UserMixin from '../../mixins/user-mixin'
import moduleMixin from '../../mixins/module-mixin'
import Avatar from '../others/avatar.vue'

import CommentList from '../comments/comment_list.vue'
import {db, storage} from "../firebaseInit"

export default {
  mixins: [
    UserMixin,
    moduleMixin
  ],
  data(){
    return {
      post: {},
      post_id: undefined,
      deleting: false,
      editing: false,
      liked: false,
      photoSrc: '',
      user_comment:''
    }
  },
  components: {
    'Avatar': Avatar,
    'CommentList': CommentList
  },
  computed: {
    imgSrc () {
      return `/users/${this.post.user}/avatar.jpg`
    },
    likes_len () {
      return this.pi.likes.length
    },
    hasPhoto: function () {
      return this.post.img_id != '' && this.post.img_id !== undefined
    }
  },
  methods: {
    Back(){
      history.back()
    },
    postComment() {
      console.log ("Posting comment " + this.user_comment + " " + this.post.post_id);
      var that = this;

      if (this.user_comment.length > 0) {
        db.collection("comments").doc(this.post.post_id.toString()).collection("commentList").add({
          content: this.user_comment,
          user: this.session.username,
          timestamp: Date.now()
        })
        .then(function() {
          that.user_comment = ""
          Notify({
            value: 'Comment successfully sent!',
          })
          console.log("Document successfully written!");
        })
        .catch(function(error) {
          console.error("Error writing document: ", error);
        });
      }

    },
    _toggle(what) {
      this[what] = !this[what]
      what == 'editing' ? $('.v_n_edit').blur() : null
    },
    deletePost: async function() {
      let
      {
        $route: { params: { post } },
        $http,
        $store: { commit }
      } = this,
      { body: { mssg } } = await $http.post('/api/delete-post', { post })

      Notify({
        value: mssg,
        done: () => this.Back()
      })
      commit('DELETE_POST', post)
    },
    editPost: async function(){
      let
      { $route: { params }, $http, $store: { commit } } = this,
      update = {
        post: params.post,
        title: $('.v_n_title').text(),
        content: $('.v_n_content').text()
      },
      { body: { mssg } } = await $http.post('/api/edit-post', update)

      Notify({ value: mssg })
      this._toggle('editing')
    },
    like: async function(){
      let
      { $route: { params }, $http, $store } = this,
      { body } = await $http.post('/api/like', { post: params.post })
      $store.commit('LIKED', body)
      this._toggle('liked')
      Notify({ value: 'Post Liked!!' })
    },
    unlike: async function(){
      let
      { $route: { params }, $http, $store } = this,
      { body } = await $http.post('/api/unlike', { post: params.post })
      $store.commit('UNLIKED', params.post)
      this._toggle('liked')
      Notify({ value: 'Post Unliked!!' })
    }
  },
  created: async function() {
    let
    {
      $http,
      $route: { params: { post } },
      $router,
      $store: { dispatch },
    } = this,
    { body: valid } = await $http.post('/api/valid-post', { post }),
    { body: pd } = await $http.post('/api/post-details', { post }),
    { body: liked } = await $http.post('/api/liked-or-not', { post })

    !valid ? $router.push('/error/post') : null
    this.post_id = post
    this.post = pd
    this.liked = liked
    dispatch('getLikes', post)

    var vm = this;

    if(this.hasPhoto){
      storage.ref().child('images/' + this.post.img_id)
      .getDownloadURL().then(function(url){
         vm.photoSrc = url;
      })
    }
  }
}
</script>
