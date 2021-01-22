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
      <b-spinner v-if="!ks"></b-spinner>
      <div v-else>
        <div v-if="ksstuff">
          <p>Key Fingerprint: {{ fingerprint }}</p>
          <img :src="ksimg" alt="PGP fingerprint as a QR Code" /><br />
          <a :href="blob(askey)" :download="fingerprint + '.asc'">Public Key</a>
        </div>
        <div v-if="kserr">
          <em>{{ kserr }}</em>
        </div>
        <div v-else-if="!ksstuff">
          <em
            ><strong>Warning!</strong> This key has no valid User IDs! It cannot
            be imported!</em
          >
        </div>
        <div v-for="key in Object.keys(ksstuff)" v-else :key="key">
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
    </div>
    <div v-else>No keys found.</div>
    <div v-if="kbstuff.id" class="kb">
      <h3>Keybase</h3>
      <b-spinner v-if="!kb"></b-spinner>
      <div v-else>
        <!-- <img :src="kbimg" alt="PGP fingerprint as a QR Code" /><br /> -->
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
              kbstuff.proofs_summary.all[key].presentation_group
                .split(':')
                .pop()
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
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import * as openpgp from 'openpgp'
import axios from 'axios'
import qrcode from 'qrcode-generator'

export default Vue.extend({
  asyncData({ params }) {
    const slug = params.slug
    return { slug }
  },
  data() {
    return {
      ks: false,
      kb: false,
      slug: '',
      text: '',
      ksimg: '',
      kserr: '',
      kbimg: '',
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
      fingerprint = /^(openpgp4fpr:)?([0-9A-F]{40})$/gim
        .exec(this.slug.replaceAll(/\s/g, ''))[2]
        .toUpperCase()
    } catch (e) {
      console.log(e)
    }
    if (!fingerprint) return
    this.fingerprint = fingerprint
    const ksqr = qrcode(0, 'L')
    ksqr.addData('OPENPGP4FPR:' + fingerprint)
    ksqr.make()
    this.ksimg = ksqr.createDataURL(6)
    ;(() => {
      const hkp = new openpgp.HKP(
        'https://corsthing.herokuapp.com/https://keyserver.ubuntu.com'
      ) // Defaults to https://keyserver.ubuntu.com, or pass another keyserver URL as a string
      hkp
        // @ts-ignore
        .lookup({
          query: '0x' + fingerprint,
        })
        .then(async (publicKeyArmored) => {
          this.ks = true
          if (!publicKeyArmored) {
            this.ksstuff = ''
            this.kserr = 'No keys found.'
            return
          }
          const x = await openpgp.key.readArmored(publicKeyArmored)
          // @ts-ignore
          if (!x.err) {
            // @ts-ignore
            const y = { ...x.keys[0].users[0].userId }
            delete y.tag
            delete y.packets
            delete y.fromStream
            this.ksstuff = y
          } else {
            console.log(x.err)
            this.ksstuff = ''
          }
          // @ts-ignore
          this.askey = publicKeyArmored
        })
        .catch((e) => {
          this.ks = true
          console.warn('ew')
          console.error(e)
        })

      axios
        .get(
          'https://keybase.io/_/api/1.0/user/lookup.json?key_fingerprint=' +
            fingerprint.toLowerCase()
        )
        .then(async (e) => {
          this.kb = true
          if (e.data.them[0]) {
            this.kbstuff = e.data.them[0]
            const a = await openpgp.key.readArmored(
              this.kbstuff.public_keys.primary.bundle
            )
            const kbqr = qrcode(0, 'L')
            kbqr.addData(
              'OPENPGP4FPR:' +
                (() =>
                  Array.prototype.map
                    .call(a.keys[0].keyPacket.fingerprint, (x) =>
                      ('00' + x.toString(16)).slice(-2)
                    )
                    .join(''))().toUpperCase()
            )
            kbqr.make()
            this.kbimg = kbqr.createDataURL(6)
          }
        })
        .catch((e) => {
          console.log(e)
          this.kb = true
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
.ks a {
  color: #ffffff;
  font-weight: 600;
}
.kb {
  background-color: #ff6f21;
  border-radius: 5px;
  padding: 20px 10px;
}
.kb a {
  color: #ffffff;
  font-weight: 600;
}
img {
  border-radius: 7px;
}
.ks {
  background-color: #4aa3ff;
  border-radius: 5px;
  padding: 20px 10px;
}
</style>
