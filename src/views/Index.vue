<template>
  <div>
    <cs-card :title="$t('request')" class="request cs-card">
      <el-form ref="request" :label-width="label_width">
        <el-form-item :label="$t('url')">
          <el-input
            v-model="request.url"
            :placeholder="`https://{{host}}/api/v1/goods`"
            class="request-url"
            clearable>
            <template slot="prepend">
              <el-button @click="formatPayload" :title="$t('format')" icon="el-icon-s-open" size="mini"/>
              <el-button @click="addFavorites" :title="$t('add favorites')" :disabled="!request.url" icon="el-icon-star-on" size="mini"/>
              <el-button @click="getHelpDocs" :title="$t('get docs')" :disabled="!request.payload || doc_disabled" icon="el-icon-s-help" size="mini"/>
              <cs-menu :disabled="!setting.apiBase" @confirm="confirmMenu"/>
            </template>

            <el-select v-model="request.method" @change="switchMethod" slot="append">
              <el-option v-for="method in methodMap" :key="method.key" :label="method.value" :value="method.key"/>
            </el-select>
          </el-input>
        </el-form-item>

        <el-form-item :label="$t(request.methodName)">
          <el-input v-model="request.payload" placeholder="application/json or query string parameters" type="textarea" :rows="10"></el-input>
        </el-form-item>
      </el-form>
    </cs-card>

    <cs-card :title="$t('headers')" :expanded="true">
      <cs-headers v-model="headers"/>
    </cs-card>

    <cs-card :title="getLoginInfo()" :expanded="true" class="cs-card">
      <el-form :inline="true" :label-width="label_width">
        <el-form-item :label="$t('username')">
          <el-input v-model="login.username" :placeholder="$t('username enter')" auto-complete="off" :disabled="is_login" clearable>
            <i slot="prefix" class="el-input__icon el-icon-user"/>
          </el-input>
        </el-form-item>

        <el-form-item :label="$t('password')">
          <el-input v-model="login.password" :placeholder="$t('password enter')" auto-complete="off" :disabled="is_login" show-password clearable>
            <i slot="prefix" class="el-input__icon el-icon-key"/>
          </el-input>
        </el-form-item>

        <el-form-item v-if="captcha.captcha" :label="$t('captcha')">
          <el-input v-model="login.login_code" class="login-code" auto-complete="off" maxlength="4" clearable>
            <template slot="append">
              <img :src="captcha.url" class="cs-fcr" height="28px" @click="refreshCode" alt=""/>
            </template>
          </el-input>
        </el-form-item>

        <el-form-item>
          <el-dropdown v-if="!is_login" @command="loginCommand">
            <el-button type="primary" :loading="login.loading">{{$t('login')}}<i class="el-icon-arrow-down el-icon--right"/></el-button>
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item command="admin">{{$t('admin group')}}</el-dropdown-item>
              <el-dropdown-item command="user">{{$t('client group')}}</el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>

          <el-button v-else type="success" :loading="login.loading" @click="logoutUser">{{$t('logout')}}</el-button>
        </el-form-item>
      </el-form>
    </cs-card>

    <div class="cs-mb cs-tr" style="height: 40px;" flex>
      <div class="send-progress cs-mr" flex-box="1">
        <el-progress v-show="sendLoading" :text-inside="true" :stroke-width="25" :percentage="percentage"/>
      </div>

      <el-button v-show="sendLoading" @click="cancel" type="danger">{{$t('cancel')}}</el-button>
      <el-button type="primary" @click="submit" :disabled="!request.url" :loading="sendLoading">{{$t('send request')}}</el-button>
    </div>

    <cs-card :title="$t('response')">
      <cs-response v-model="response"/>
      <div class="cs-tc">
        <el-button :title="$t('copy request url')" @click="copyRequest('request.responseURL')">{{$t('copy request')}}</el-button>
        <el-button :title="$t('copy response body')" @click="copyRequest('request.response')">{{$t('copy response')}}</el-button>
      </div>
    </cs-card>
  </div>
</template>

<script>
import { mapActions, mapState } from 'vuex'
import util from '@/utils/util'
import sendRequest from './components/mixins/sendRequest'
import { getAppCaptcha } from '@/api/app'
import { loginAdminUser, logoutAdminUser } from '@/api/admin'
import { loginClientUser, logoutClientUser } from '@/api/client'
import * as clipboard from 'clipboard-polyfill'
import { get } from 'lodash'

export default {
  name: 'Index',
  mixins: [sendRequest],
  computed: {
    ...mapState('careyshop/setting', [
      'setting'
    ])
  },
  components: {
    csCard: () => import('@/components/cs-card'),
    csMenu: () => import('@/components/cs-menu'),
    csHeaders: () => import('@/components/cs-headers'),
    csResponse: () => import('@/components/cs-response')
  },
  watch: {
    value: {
      handler(val) {
        const request = get(val, 'request')
        if (request) {
          this.request = { ...request }
        }

        const name = get(val, 'name')
        if (name) {
          this.favoriteName = val.name
        }

        this.headers = get(val, 'headers', [])
      },
      immediate: true
    }
  },
  props: {
    value: {
      type: Object,
      required: false,
      default: () => {}
    }
  },
  data() {
    return {
      is_login: false,
      doc_disabled: false,
      label_width: '90px',
      percentage: 0,
      sendLoading: false,
      favoriteName: '',
      methodMap: [
        { key: 'get', value: 'GET' },
        { key: 'post', value: 'POST' },
        { key: 'put', value: 'PUT' },
        { key: 'patch', value: 'PATCH' },
        { key: 'delete', value: 'DELETE' },
        { key: 'head', value: 'HEAD' },
        { key: 'options', value: 'OPTIONS' }
      ],
      // 是否需要验证码
      captcha: {
        captcha: false,
        url: ''
      },
      // 账号登录表单
      login: {
        mode: '',
        loading: false,
        username: '',
        password: '',
        login_code: '',
        session_id: ''
      },
      // 请求表单
      request: {
        url: '',
        payload: '',
        method: 'post',
        methodName: 'payload'
      },
      // 请求头
      headers: [],
      // 响应
      response: {}
    }
  },
  mounted() {
    this.setLoginInfo()
    this.setCaptcha()
  },
  methods: {
    ...mapActions('careyshop/favorites', [
      'addToFavorites'
    ]),
    // 复制响应某个值
    copyRequest(key) {
      let response = get(this.response, key, '')
      clipboard.writeText(response)
        .then(() => {
          this.$message.success(this.$t('copy success'))
        })
        .catch(err => {
          this.$message.error(err)
        })
    },
    // 格式化代码
    formatPayload() {
      if (this.request.payload) {
        try {
          let payload = JSON.parse(this.request.payload)
          this.request.payload = JSON.stringify(payload, null, 4)
        } catch (e) {
          this.$message.error(e.message)
        }
      }
    },
    // 切换请求方式
    switchMethod(value) {
      switch (value) {
        case 'put':
        case 'post':
        case 'patch':
          this.request.methodName = 'payload'
          break

        default:
          this.request.methodName = 'params'
      }
    },
    // 设置是否启用验证码
    setCaptcha() {
      if (this.is_login) {
        return
      }

      const { apiBase, appKey } = this.setting
      if (apiBase && appKey) {
        getAppCaptcha(appKey)
          .then(res => {
            if (res.data.captcha) {
              this.captcha.captcha = true
              this.login.session_id = res.data.session_id
              this.refreshCode()
            }
          })
      } else {
        this.captcha.captcha = false
        this.captcha.session_id = ''
      }
    },
    // 刷新验证码
    refreshCode() {
      if (!this.setting.apiBase) {
        return
      }

      let url = util.settingReplace(this.setting.apiBase, this.setting.variable)
      url += '/v1/app.html?'
      url += `method=image.app.captcha&session_id=${this.login.session_id}&t=${Math.random()}`

      this.captcha.url = util.checkUrl(url)
      this.login.login_code = ''
    },
    // 账号登录
    loginCommand(command) {
      this.login.loading = true
      const login = command === 'admin' ? loginAdminUser : loginClientUser

      login({
        ...this.login,
        appkey: this.setting.appKey
      })
        .then(res => {
          this.is_login = true
          this.login.mode = command
          this.login.password = ''
          this.captcha.captcha = false

          const cookieSetting = { expires: 365 }
          util.cookies.set('mode', command, cookieSetting)
          util.cookies.set('name', res.data[command].username, cookieSetting)
          util.cookies.set('token', res.data.token.token, cookieSetting)
        })
        .catch(() => {
          this.refreshCode()
        })
        .finally(() => {
          this.login.loading = false
        })
    },
    // 注销账号
    logoutUser() {
      this.login.loading = true
      const logout = this.login.mode === 'admin' ? logoutAdminUser : logoutClientUser

      logout()
        .catch(() => {
        })
        .finally(() => {
          this.login.mode = ''
          this.login.loading = false
          this.login.username = ''
          this.is_login = false

          util.cookies.remove('mode')
          util.cookies.remove('name')
          util.cookies.remove('token')

          this.setCaptcha()
        })
    },
    // 设置账号信息
    setLoginInfo() {
      this.is_login = Boolean(util.cookies.get('token'))
      this.login.mode = util.cookies.get('mode')
      this.login.username = util.cookies.get('name')

      if (process.env.VUE_APP_ISDEMO === 'true') {
        this.login.username = 'admin' + Math.floor(Math.random() * (45 - 1 + 1) + 1)
        this.login.password = 'admin888'
      }
    },
    // 获取登录状态
    getLoginInfo() {
      let status = this.$t('guest')
      if (this.is_login) {
        status = this.login.mode === 'admin' ? this.$t('admin group') : this.$t('client group')
      }

      return `${this.$t('account')} (${status})`
    },
    // 确认填充菜单
    confirmMenu(value) {
      this.request.url = this.setting.apiBase + value.url
      this.request.payload = value.payload
    },
    // 获取帮助文档地址
    getHelpDocs() {
      let data
      try {
        data = JSON.parse(this.request.payload)
        if (!Object.prototype.hasOwnProperty.call(data, 'method')) {
          throw new Error(`${this.$t(this.request.methodName)} ${this.$t('not method')}`)
        }
      } catch (e) {
        this.$message.error(e.message)
        return
      }

      this.doc_disabled = true
      this.$axios({
        url: 'https://www.careyshop.cn/api/v1/api_docs.html',
        method: 'post',
        headers: { 'Content-Type': 'application/json; charset=utf-8' },
        data: {
          keyword: data.method,
          module: this.login.mode === 'admin' ? 'admin' : 'client'
        }
      })
        .then(res => {
          if (!res.data) {
            this.$message.warning(this.$t('not help docs'))
            return
          }

          this.$open(res.data.host + res.data.url)
        })
        .catch(err => {
          this.$message.error(err.message)
        })
        .finally(() => {
          this.doc_disabled = false
        })
    },
    // 添加至收藏夹
    addFavorites() {
      this.$prompt(this.$t('favorite name'), this.$t('tips'), {
        inputValue: this.favoriteName,
        inputPattern: /\S/,
        inputErrorMessage: this.$t('favorite error'),
        closeOnClickModal: false,
        type: 'info'
      })
        .then(async({ value }) => {
          const data = {
            name: value,
            request: this.request,
            headers: this.headers
          }

          this.favoriteName = value
          if (await this.addToFavorites({ vm: this, value: data })) {
            this.$message.success(this.$t('favorite success'))
          }
        })
        .catch(() => {
        })
    }
  }
}
</script>

<style scoped lang="scss">
  .cs-card /deep/ .el-card__body {
    padding: 20px 20px 0 20px;
  }

  .request {
    .request-url {
      /deep/ .el-select .el-input {
        width: 130px;
      }

      /deep/ .is-disabled {
        border-color: #FFF;
      }

      /deep/ .el-input-group__append {
        background-color: #FFF;
      }

      /deep/ .el-input-group__prepend {
        background-color: #FFF;
      }
    }
  }

  .login-code {
    /deep/ input {
      width: 95px;
    }

    /deep/ .el-input-group__append {
      padding: 0;
      background-color: #FFF;
    }
  }

  .send-progress {
    align-self: center;
  }
</style>
