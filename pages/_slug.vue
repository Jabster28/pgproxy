<template>
  <div class="container">
    <b-form @submit="onSubmit">
      <b-form-input
        v-model="text"
        placeholder="Enter a PGP fingerprint"
      ></b-form-input>
    </b-form>
    <div v-if="fingerprint" class="ks">
      <h3>OpenPGP</h3>
      <p>Key Fingerprint: {{ fingerprint }}</p>
      <a :href="blob(askey)" :download="fingerprint + '.asc'">Public Key</a>

      <div v-for="key in Object.keys(ksstuff)" :key="key">
        <p v-if="key == 'email'">
          {{ key }}:
          <a
            target="_blank"
            rel="noopener noreferrer"
            :href="`mailto:${ksstuff[key]}`"
            >{{ ksstuff[key] }}</a
          >
        </p>
        <p v-else>{{ key }}: {{ ksstuff[key] }}</p>
      </div>
    </div>
    <div v-else>No keys found.</div>
    <div v-if="kbstuff.id" class="kb">
      <h3>Keybase</h3>
      <p>
        Username:
        <a
          target="_blank"
          rel="noopener noreferrer"
          :href="`https://keybase.io/${kbstuff.basics.username}`"
          >{{ kbstuff.basics.username }}</a
        >
      </p>
      <p>Bio: {{ kbstuff.profile.bio }}</p>
      <p>Location: {{ kbstuff.profile.location }}</p>
      <p>Full Name: {{ kbstuff.profile.full_name }}</p>
      <div
        v-for="key in Object.keys(kbstuff.cryptocurrency_addresses)"
        :key="key"
      >
        <p>
          {{ key }} address:
          {{ kbstuff.cryptocurrency_addresses[key][0].address }}
        </p>
      </div>
      <div v-for="key in Object.keys(kbstuff.proofs_summary.all)" :key="key">
        <p>
          {{
            kbstuff.proofs_summary.all[key].presentation_group.split(':').pop()
          }}:
          <a
            target="_blank"
            rel="noopener noreferrer"
            :href="kbstuff.proofs_summary.all[key].service_url"
          >
            {{ kbstuff.proofs_summary.all[key].nametag }}
          </a>
          <sub
            ><a
              target="_blank"
              rel="noopener noreferrer"
              :href="kbstuff.proofs_summary.all[key].proof_url"
              >(proof)</a
            ></sub
          >
        </p>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import * as openpgp from 'openpgp'
import axios from 'axios'

export default Vue.extend({
  asyncData({ params }) {
    const slug = params.slug
    return { slug }
  },
  data() {
    return {
      slug: '',
      text: '',
      ksstuff: {},
      kbstuff: {},
      askey: '',
      fingerprint: '',
    }
  },
  mounted() {
    this.text = this.slug
    let fingerprint
    try {
      // @ts-ignore
      fingerprint = /^(openpgp4fpr:)?([0-9A-F ]*)$/gim
        .exec(this.slug.replaceAll(/\s/g, ''))[2]
        .toUpperCase()
    } catch (e) {
      console.log(e)
    }
    if (!fingerprint) return
    this.fingerprint = fingerprint
    ;(async () => {
      const hkp = new openpgp.HKP('https://keys.openpgp.org') // Defaults to https://keyserver.ubuntu.com, or pass another keyserver URL as a string
      // @ts-ignore
      const publicKeyArmored = await hkp.lookup({
        query: '0x' + fingerprint,
      })
      const x = await openpgp.key.readArmored(publicKeyArmored)
      // @ts-ignore
      if (!x.err) {
        // @ts-ignore
        const y = { ...x.keys[0].users[0].userId }
        delete y.tag
        delete y.packets
        delete y.fromStream
        this.ksstuff = y
      }
      // @ts-ignore
      this.askey = publicKeyArmored
      axios
        .get(
          'https://keybase.io/_/api/1.0/user/lookup.json?key_fingerprint=' +
            fingerprint.toLowerCase()
        )
        .then((e) => {
          if (e.data.them[0]) {
            this.kbstuff = e.data.them[0]
          }
        })
    })()
  },
  methods: {
    blob(x: string) {
      const myBlob = new Blob([x], {
        type: 'application/pgp-keys',
      })
      const url = URL.createObjectURL(myBlob)
      return url
    },
    onSubmit(event: Event) {
      event.preventDefault()
      this.$router.push(this.text)
    },

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
.kb,
.ks {
  margin: 10px;
}
.kb {
  background-color: #ff6f21;
  border-radius: 5px;
  padding: 20px 10px;
}
.ks {
  background-color: #4aa3ff;
  border-radius: 5px;
  padding: 20px 10px;
}
</style>
