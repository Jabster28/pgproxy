<template>
  <div class="container">
    <div class="based">
      <div>
        <h1 class="title">pgproxy</h1>
        <br />
        <div class="links">
          <button class="button--green" @click="register">Register</button>
          <a
            href="openpgp4fpr:8603C4DEFFB2244E3969F1BEB1B1CC5F0F0F0578"
            target="_blank"
            rel="noopener noreferrer"
            class="button--grey"
          >
            Test
          </a>
        </div>
      </div>
    </div>
    <footer>
      running commit hash
      <a :href="`https://github.com/jabster28/pgproxy/commit/${commitSha}`">
        {{ commitSha }}
      </a>
    </footer>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'

export default Vue.extend({
  data() {
    return {
      commitSha: process.env.NUXT_ENV_CURRENT_GIT_SHA,
    }
  },
  mounted() {
    console.log(`Running on git hash (${this.commitSha})`)
  },
  methods: {
    register() {
      navigator.registerProtocolHandler(
        'openpgp4fpr',
        'https://pgproxy.web.app/%s',
        'Custom Keyserver Lookup'
      )
    },
  },
})
</script>

<style>
.based {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.title {
  font-family: 'Quicksand', 'Source Sans Pro', -apple-system, BlinkMacSystemFont,
    'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}
</style>
