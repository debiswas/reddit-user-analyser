<template>
    <div>
        <small v-if="data" class="comment-info">
            /r/{{data.subreddit}} ·
            <span :class="color">{{data.score}}</span> ·
            {{date}}
        </small>
        <div class="comment" v-html="compiledMarkdown"></div>
    </div>
</template>

<script>
var marked = require('marked');
marked.setOptions({
  renderer: new marked.Renderer(),
  gfm: false,
  tables: true,
  breaks: false,
  pedantic: false,
  sanitize: false,
  smartLists: true,
  smartypants: false
});

var moment = require('moment');

export default {
  name: 'comment',
  props: ['data'],
  computed: {
      compiledMarkdown() {
          if (!this.data) return 'N/A';
          return marked(this.data.body);
      },
      date() {
          if (!this.data) return;
          return moment(1000 * this.data.created_utc).fromNow();
      },
      color() {
          if (!this.data) return;
          return this.data.score >= 1 ? 'orangered-color' : 'periwinkle-color';
      }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">
    $text: #6b7c93;

    .comment {
        word-wrap: break-word;

        p {
            text-align: left;
            margin-bottom: 0.5rem;
        }

        code {
            color: white;
        }
    }

    .orangered-color {
        color: #ff9453;
        font-weight: bold;
    }

    .periwinkle-color {
        color: #00ccff;
        font-weight: bold;
    }

    .controversial {
        font-weight: 700;
        color: #ff5600;
    }

    p {
        font-size: 17px;
        color: darken($text, 20);
    }

    a {
        transition: all 0.2s ease;
        &:hover {
            text-decoration: none;
            background-color: rgba(0,150,255,.1);
        }
    }

</style>
