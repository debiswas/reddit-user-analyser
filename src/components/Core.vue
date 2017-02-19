<template>
    <div id="main">
        <header>
            <div class="container header-section">
                <img width="300" src="../assets/reddit_logo.svg">
                <h1>Analyse a Reddit user</h1>
                <div class="input-group input-group-lg input-group--username">
                  <span class="input-group-addon" id="u-addon">/u/</span>
                  <input @keyup.enter="fetchData()" v-model="username" type="text" class="form-control username-input" placeholder="Username" aria-describedby="u-addon">
                  <span class="input-group-btn">
                      <button @click="fetchData()" class="btn btn-secondary" type="button" :disabled="isLoading">Analyse</button>
                  </span>
                </div>
            </div>
        </header>
        <div class="container container--summary">
            <div v-if="isLoading" class="ajax-loader d-flex justify-content-center">
                <div></div>
                <div></div>
                <div></div>
                <div></div>
            </div>
            <p v-show="noPosts">*dust*</p>
            <p v-show="notFound">Not found</p>
            <div v-show="valid">
                <user-summary :about="about"
                              :username="username"
                              :comments="comments"
                              :submitted="submitted"
                              :isLoading="isLoading">
                </user-summary>
            </div>
        </div>
    </div>
</template>

<script>
import UserSummary from 'components/UserSummary'

$(function(){
    $.ripple(".btn", {
        debug: false, // Turn Ripple.js logging on/off
        on: 'mousedown', // The event to trigger a ripple effect

        opacity: 0.2, // The opacity of the ripple
        color: "#61efa7", // Set the background color. If set to "auto", it will use the text color
        multi: true, // Allow multiple ripples per element

        duration: 1, // The duration of the ripple

        // Filter function for modifying the speed of the ripple
        rate: function(pxPerSecond) {
            return pxPerSecond;
        },

        easing: 'linear' // The CSS3 easing function of the ripple
    });
});

export default {
  name: 'core',
  components: {
      UserSummary
  },
  data () {
    return {
      username: '',
      comments: [],
      submitted: [],
      about: {},
      isLoading: false,
      notFound: false,
      noPosts: false,
      finished: {
          comments: false,
          submitted: false
      }
    }
  },
  mounted() {
      this.$watch('username', () => {
         this.reset();
      });
      // Auto fetch
      if (window.location.hash !== '') {
          this.username = window.location.hash.split('#').pop().trim();
          this.fetchData();
      }
  },
  computed: {
      valid() {
          return this.finishedLoading && this.comments.length;
      },
      finishedLoading() {
          if (!this.comments.length && !this.submitted.length) return;
          if (this.finished.comments && this.finished.submitted) return true;
      }
  },
  methods: {
      reset() {
          this.notFound = false;
          this.noPosts = false;
          this.finished.comments = false;
          this.finished.submitted = false;
          this.comments = [];
          this.submitted = [];
      },
      fetchData() {
          if (this.username === "" || /[^a-zA-Z0-9_-]/.test(this.username)) return;

          this.reset();

          document.title = `Overview for ${this.username} – Reddit User Analyser`;
          window.history.replaceState({}, "", `#${this.username}`);

          this.isLoading = true;

          this.fetchAbout();
          this.fetchCombined('comments');
          this.fetchCombined('submitted');
      },
      fetchAbout() {
          this.$http.get(`https://www.reddit.com/user/${this.username}/about/.json`)
          .then(response => {
              this.about = response.body.data;
          }).catch(response => {
              if (response.status === 404) {
                  this.notFound = true;
                  this.isLoading = false;
              }
          });
      },
      fetchCombined(type, after = "") {
          this.$http.get(`https://www.reddit.com/user/${this.username}/${type}.json?limit=100&after=${after}`)
          .then(response => {
              let arr = response.body.data.children;

              // No more posts
              if (!arr.length) {
                  this.isLoading = false;
                  this.finished[type] = true;
                  if (!this[type].length && this.finished.comments && this.finished.submitted && !this.comments.length && !this.submitted.length)
                      this.noPosts = true;
                  return;
              }

              // Add additional posts to array
              arr.forEach(item => {
                  this[type].push(item);
              });

              // If there's (almost certainly) more, recursively fetch more
              if (arr.length == 100) {
                  this.fetchCombined(type, arr[99].data.name);
                  return;
              }

            this.finished[type] = true;

            if (this.finished.comments && this.finished.submitted)
                this.isLoading = false;

          }).catch(response => {
              if (response.status === 404) {
                  this.notFound = true;
                  this.isLoading = false;
              }
          });
      }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">

    /*! Ripple.js v1.2.1
     * The MIT License (MIT)
     * Copyright (c) 2014 Jacob Kelley */
    .has-ripple{position:relative;overflow:hidden;-webkit-transform:translate3d(0,0,0);-o-transform:translate3d(0,0,0);transform:translate3d(0,0,0)}.ripple{display:block;position:absolute;pointer-events:none;border-radius:50%;-webkit-transform:scale(0);-o-transform:scale(0);transform:scale(0);background:#fff;opacity:1}.ripple-animate{-webkit-animation:ripple;-o-animation:ripple;animation:ripple}@-webkit-keyframes ripple{100%{opacity:0;-webkit-transform:scale(2);transform:scale(2)}}@-o-keyframes ripple{100%{opacity:0;-o-transform:scale(2);transform:scale(2)}}@keyframes ripple{100%{opacity:0;transform:scale(2)}}

    @mixin shadow($color) {
        box-shadow: 0 6px 25px -2px $color;
    }

    $text: #aebed4;
    $btn-width: 125px;

    body {
        font-family: 'Avenir', -apple-system,
                BlinkMacSystemFont,
                "Segoe UI",
                Roboto,
                Oxygen-Sans,
                Ubuntu,
                Cantarell,
                "Helvetica Neue",
                sans-serif;
        background-color: #271b68;
    }
    #app {
      -webkit-font-smoothing: antialiased;
      -moz-osx-font-smoothing: grayscale;
      text-align: center;
      color: #cad6e6;
    }
    a {
        color: #6edbff;

        &:hover, &:focus {
            color: lighten(#6edbff, 10);
        }
    }

    .section-illustration {
        max-width: 100%;
        width: 250px;
    }

    .center {
        text-align: center;
    }

    header {
        position: relative;
        background: #474ebd;
        background: radial-gradient(circle at 50% 1%, #474ebd, #371c84);
        padding: 2rem 0 2rem;
        margin-bottom: 3rem;
        box-shadow: 0 10px 80px -2px rgba(0, 0, 0, 0.4);
        overflow: hidden;

        &::before {
            display: inline-block;
            opacity: 0.8;
            content: "";
            height: 800px;
            width: 200%;
            background: url('../assets/ocean.svg') no-repeat;
            left: 50%;
            transform: translateX(-50%);
            top: 100px;
            position: absolute;
            background-position: center;
            background-size: contain;
        }

        h1 {
            font-family: 'Avenir', sans-serif;
        }
    }
    h1 {
        font-weight: 300;
        margin-bottom: 2rem;
        color: white;
    }
    .input-group {
        &--username {
            position: relative;
        }
        .form-control:not(:first-child):not(:last-child) {
            border-top-right-radius: 4px;
            border-bottom-right-radius: 4px;
        }
    }
    .username-input {
        position: relative;
        background: rgba(0,10,25,0.5);
        border: 1px solid rgba(0,0,0,0.4);
        border-bottom-color: rgba(0,0,0,0.5);
        box-shadow: 0 5px 12px -2px rgba(0,0,0,0.3);
        font-weight: 300;
        color: lighten($text, 10);

        &:hover {
            box-shadow: 0 10px 30px -2px rgba(0,0,0,0.3);
            z-index: 0;
        }

        &:focus {
            z-index: 0;
            background: rgba(0,0,0,0.3);
            color: white;
        }

        &::-webkit-input-placeholder {
            color: rgba(255,255,255,0.4);
        }
        &::-moz-input-placeholder {
            color: rgba(255,255,255,0.4);
        }
        &:-ms-input-placeholder {
            color: rgba(255,255,255,0.4);
        }
    }
    .input-group-addon {
        padding: 0 1rem;
    }
    .input-group-btn {
        position: absolute;
        z-index: 10;
        right: 0;
        height: 100%;
    }
    .btn {
        background-color: transparent;
        color: #61efa7;
        border: none;
        text-transform: uppercase;
        font-weight: 500;
        font-size: 17px !important;
        cursor: pointer;
        width: $btn-width;
        z-index: 1;

        &:disabled {
            background-color: rgba(255,255,255,0.1);
        }

        &:hover {
            background-color: rgba(255,255,255,0.1);
            color: white;
            z-index: 1;
        }

        &:active {
            background-color: rgba(255,255,255,0.1);
            color: white;
        }

        &:focus {
            box-shadow: 0 0 0 2px rgba(153, 236, 167, 0.5);
        }
    }
    #u-addon {
        background: linear-gradient(115deg, #ff79aa, #b743ff);
        box-shadow: 0 5px 20px -2px rgba(200,50,150,.7);
        border: none;
        color: white;
        font-weight: 700;
        padding: 0 0.75rem;
    }
    .header-section {
        max-width: 900px;
    }
    .comment {
        word-wrap: break-word;
    }
    .summary {
        text-align: left;
    }
    @keyframes ajax {
        50% {
            transform: scale(2.5);
            box-shadow: 0 8px 50px -4px rgba(0, 50, 150, 0.2);
        }
        100% {
            transform: scale(1);
        }
    }
    .ajax-loader {
        div {
            width: 10px;
            height: 10px;
            background-color: #E089D9;
            border-radius: 50%;
            margin: 0 1rem;
            animation: ajax .8s ease infinite;

            &:nth-child(2) {
                background-color: #FFA9A9;
                animation-delay: 0.1s;
            }
            &:nth-child(3) {
                background-color: #FFD578;
                animation-delay: 0.2s;
            }
            &:nth-child(4) {
                background-color: #a6d5e6;
                animation-delay: 0.3s;
            }
        }
    }
</style>
