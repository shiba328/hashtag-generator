<template>
  <v-container>
    <h1 class="mb-3">
      WEB用ツイッターシェアリンクURLメーカー
    </h1>
    <v-row>
      <v-col cols="12" sm="6">
        <v-text-field
          v-model="via"
          :label="str.viaLabel"
          :placeholder="str.viaPlaceholder"
          :hint="str.viaHint"
          persistent-hint
          prefix="@"
          outlined
          dense
          @keyup="htmlPreview()"
        />
        <v-row class="my-2">
          <v-col cols="9" sm="8">
            <v-row no-gutters>
              <v-col
                v-for="(item, i) in hashtags"
                :key="i"
                cols="12"
                class="mb-2"
              >
                <v-text-field
                  v-model="item.value"
                  append-outer-icon="mdi-minus-circle-outline"
                  :label="str.hashLabel"
                  :placeholder="str.hashPlaceholder"
                  prefix="#"
                  outlined
                  dense
                  hide-details
                  @keyup="htmlPreview()"
                  @click:append-outer="removeItem(i)"
                />
              </v-col>
            </v-row>
          </v-col>
          <v-col cols="3" sm="4">
            <v-col
              sm="4"
            >
              <v-btn
                color="primary"
                small
                @click="addItem()"
              >
                <v-icon small>mdi-plus-circle-outline</v-icon>
                タグを追加
              </v-btn>
            </v-col>
          </v-col>
        </v-row>
        <v-text-field
          v-model="url"
          :label="str.urlLabel"
          :placeholder="str.urlPlaceholder"
          :hint="str.urlHint"
          persistent-hint
          outlined
          dense
          @keyup="htmlPreview()"
        />
        <v-textarea
          v-model="text"
          :label="str.textLabel"
          :hint="str.textHint"
          persistent-hint
          outlined
          @keyup="htmlPreview()"
        />
        <h2>プレビュー</h2>
        <div class="skin">
          <textarea
            ref="urlEncodeStr"
            :value="getEncodeStr()"
            class="inner v-alert"
            readonly
          />
          <textarea
            ref="urlEncodeLink"
            :value="getEncodeLink()"
            class="inner v-alert"
            readonly
          />
          <div
            class="spacer v-alert v-alert--outlined"
            v-html="preview"
          />
        </div>
        <div class="config">
          <span>
            {{ 280 / 2 - tw.weightedLength / 2 || 280 / 2 }}
          </span>
          <v-btn
            color="primary"
            link
            small
            :href="getEncodeStr()"
            Target="_blank"
          >
            プレビューツイート
          </v-btn>
          <v-btn
            small
            color="primary"
            @click="getClipboardStr()"
          >
            コピー
          </v-btn>
          <v-btn
            v-if="false"
            small
            color="primary"
            @click="getClipboardLink()"
          >
            リンク
          </v-btn>
        </div>
      </v-col>
      <v-col cols="12" sm="6">
        <v-card outlined class="mb-3">
          <v-card-title>使い方</v-card-title>
          <v-card-text>
            <ul>
              <li>
                「ツイッターURLリンクメーカー」はWEBサイトのページやメール(メッセージ)で利用するリンクタグ(a href)用のURLを作成するジェネレーターです。
              </li>
              <li>
                このページで作成された情報はサーバー等にアップロードされません。<br>安心してお使いいただけます。
              </li>
              <li>
                「復元」はURLから各項目に分散されます。前回のURLから修正したい場合は「復元」をご利用ください。
              </li>
            </ul>
          </v-card-text>
        </v-card>
        <v-btn
          color="primary"
          @click="dialog = true"
        >
          復元
        </v-btn>
      </v-col>
    </v-row>
    <v-dialog
      v-model="dialog"
      width="400"
    >
      <v-card>
        <v-card-title>
          復元
        </v-card-title>
        <v-card-text>
          <v-textarea
            outlined
            v-model="reverseUrl"
            :error="error"
            :error-messages="errorMessage"
          />
          <v-btn
            color="primary"
            @click="reverse"
          >
            セットする
          </v-btn>
        </v-card-text>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
import twt from 'twitter-text'

const STR = {
  viaHint: '@〇〇よりが必要な場合には、スクリーンネームを@以降に追加してください。空の場合は文字数にカウントされません。よりをつけたくない場合は文章内に含めてください',
  viaLabel: '経由(〇〇より)',
  viaPlaceholder: 'shiba_328',
  hashLabel: 'ハッシュタグ',
  hashPlaceholder: 'テスト',
  urlLabel: 'URL',
  urlPlaceholder: 'https://example.com',
  urlHint: '11文字以上のURLは…で略されます。空の場合は文字数にカウントされません',
  textLabel: '文章',
  textHint: 'メンションやハッシュタグを含めて140文字の制限です。下のプレビューを確認してください。'
}

const INIT = {
  value: ''
}
export default {
  data: () => {
    return {
      reverseUrl: '',
      dialog: false,
      str: {},
      url: '',
      via: '',
      hashtags: [],
      text: '',
      concat: '',
      preview: '',
      error: false,
      errorMessage: '',
      encodeStr: '',
      tw: {}
    }
  },
  mounted () {
    this.str = STR
    this.hashtags.push({...INIT})
    this.concatPreview()
  },
  watch: {
    concat () {
      console.log('watch')
      this.htmlPreview()
      this.tw = twt.parseTweet(this.getConcat())
    }
  },
  methods: {
    reverse () {
      let url, res
      try {
        url = new URL(this.reverseUrl)
        res = url.search.substring(1).split('&').map((p) => p.split('=')).reduce((obj, e) => ({...obj, [e[0]]: e[1]}), {});
        this.via = res.via
        this.text = decodeURIComponent(res.text)
        this.url = decodeURIComponent(res.url)
        this.dialog = false
        this.hashtags = decodeURI(res.hashtags).split('%2C').map((e) => ({value: e}), {})
        this.htmlPreview()
      }
      catch (e) {
        this.error = true
        this.errorMessage = 'URLではありません'
      }
    },
    getClipboardLink () {
      const textarea = this.$refs.urlEncodeLink
      // 文字をすべて選択
      textarea.select()
      document.execCommand("copy")
    },
    getClipboardStr () {
      const textarea = this.$refs.urlEncodeStr
      // 文字をすべて選択
      textarea.select()
      document.execCommand("copy")
    },
    removeItem (index) {
      let res = true
      if (this.hashtags[index].value) {
        res = confirm('Are you sure you want to clear?')
      }
      if (res) {
        this.hashtags.splice(index, 1)
      }
    },
    addItem () {
      this.hashtags.push({...INIT})
    },
    getEncodeStr () {
      console.log('getEncodeStr')
      const itmes = []
      if (this.via) {
        itmes.push(`via=${encodeURI(this.via)}`)
      }
      if (this.text) {
        itmes.push(`text=${encodeURI(this.text)}`)
      }
      if (this.url) {
        itmes.push(`url=${encodeURI(this.url)}`)
      }
      if (this.hashtags[0] && this.hashtags[0].value) {
        const map = this.hashtags.map(v =>  v.value ? encodeURI(v.value) : '').filter(Boolean).join(',')

        itmes.push(`hashtags=${map}`)
      }
 
      const str = `https://twitter.com/share?${itmes.join('&')}`
      localStorage.setItem('hg', str)
      return str
    },
    getEncodeLink () {
      const str = this.encodeStr
      return `<a href="${str}" target="_blank">Twiter</a>`
    },
    getUrl (str) {
      if (str.length > 20) {
        return str.substr(0, 20) + '...'
      }
      return str
    },
    getConcat () {
      const map = this.hashtags.map(v =>  v.value ? ' #' + v.value : '').join('')
      const name = this.via ? '@' + this.via + ' ' : ''
      const url = this.url ? ' ' + this.getUrl(this.url) : ''
      return `${name}${this.text}${map}${url}`
    },
    concatPreview () {
      console.log('concatPreview')
      this.concat = this.getConcat()
    },
    htmlPreview () {
      console.log('htmlPreview')
      this.concatPreview()
      const res
        = this.htmlPreviewtext(this.text)
        + this.htmlPreviewUrl(this.url)
        + this.htmlPreviewHash(this.hashtags)
        + this.htmlPreviewScreenNmae(this.via)
      this.preview = res
    },
    htmlPreviewScreenNmae (via) {
      if (!via) return ''
      let str = '@' + via
      return ` <a href="https://twitter.com/${via}">${str}</a>より`
    },
    htmlPreviewUrl (url) {
      if (!url) return ''
      let str = url
      str = url.replace(/^(https?|http):\/\//g, '')
      if (url.length > 26) {
        str = str.substr(0, 26) + '…'
      }
      return ` <a href="${url}">${str}</a>`
    },
    htmlPreviewtext (str) {
      return str.replace(/\r?\n/g, '<br>')
    },
    htmlPreviewHash (hashtags) {
      const res = hashtags.map(v => v.value ? ` <a href="https://twitter.com/hashtag/${v.value}">#${v.value}</a>` : '')
      return res.join('')
    }
  }
}
</script>

<style>
.skin {
  position: relative;
}
.skin .inner {
  position: absolute;
  resize: none;
}
.skin .spacer {
  /* visibility: hidden; */
  position: relative;
  background: #fff !important;
  z-index: 1;
  width: 100%;
  height: 100%;
  min-height: 6em;
}
</style>
