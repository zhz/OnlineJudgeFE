<template>
  <div class="flex-container">
    <div id="problem-main">
      <Row type="flex" :gutter="18">
      <Col :span="16">

      <Card :padding="20" id="submit-code" dis-hover>
	      <div slot="title" style="font-size:21px; font-weight:500; padding-top:10px; padding-bottom:20px; line-height:30px; padding:5px 15px;">{{$t('m.IDE')}}</div>
        <CodeMirror :value.sync="code"
                    :languages="languages"
                    :language="language"
                    :theme="theme"
                    @resetCode="onResetToTemplate"
                    @changeTheme="onChangeTheme"
                    @changeLang="onChangeLang">
        </CodeMirror>
        <Row type="flex" justify="space-between">
          <Col span="24">
            <template v-if="captchaRequired">
              <div class="captcha-container">
                <Tooltip v-if="captchaRequired" content="Click to refresh" placement="top">
                  <img :src="captchaSrc" @click="getCaptchaSrc"/>
                </Tooltip>
                <Input v-model="captchaCode" class="captcha-code"/>
              </div>
            </template>
          </Col>
        </Row>
        <Row type="flex" justify="space-between">
          <Col :span="12">
            <Button type="warning" icon="md-create" :loading="submitting" @click="submitCode"
              class="fl-right">
              <span v-if="submitting">{{$t('m.Submitting')}}</span>
              <span v-else>{{$t('m.Submit')}}</span>
            </Button>
          </Col>
        </Row>
      </Card>
      </Col>
      <Col :span="8">
      <Row type="flex" :gutter="24" style="margin-bottom:10px;">
	      <Col span="24">
          <Card shadow>
            <p slot="title">{{$t('m.Input')}}</p>
            <Input v-model="input" type="textarea" :autosize="{minRows: 8,maxRows: 8}" placeholder="" />
          </Card>
        </Col>
      </Row>
 
      <Row type="flex" :gutter="24" style="margin-bottom:10px;">
        <Col span="24" :padding="10">
          <Card shadow>
            <p slot="title">{{$t('m.Output')}}</p>
		        <p slot="extra">{{$t('m.Time')}}: {{time_cost}} ms</p>
	  	      <p slot="extra">{{$t('m.Memory')}}: {{memory_cost}} MB</p>
            <Input v-model="output" readonly type="textarea" :autosize="{minRows: 8,maxRows: 8}" placeholder="" />
          </Card>
        </Col>
      </Row>

      <Row type="flex" :gutter="24">
	      <Col span="24">
          <Card shadow>
            <p slot="title">{{$t('m.Hint')}}</p>
            <p>{{$t('m.IDEHint')}}</p>
          </Card>
        </Col>
      </Row>

    </Col>
    </Row>
    </div>
  </div>
</template>

<script>
  import CodeMirror from '@oj/components/CodeMirror.vue'
  import storage from '@/utils/storage'
  import {FormMixin} from '@oj/components/mixins'
  import {JUDGE_STATUS, buildProblemCodeKey} from '@/utils/constants'
  import api from '@oj/api'
  export default {
    name: 'IDE',
    components: {
      CodeMirror
    },
    mixins: [FormMixin],
    data () {
      return {
        statusVisible: false,
        captchaRequired: false,
        submissionExists: false,
        captchaCode: '',
        captchaSrc: '',
        submitting: false,
        code: '',
        language: 'C++',
        theme: 'solarized',
        result: {
          result: 9
        },
        languages: [],
        input: '',
        output: '',
        submission: {
          result: '0',
          code: '',
          info: {
            data: []
          }
        },
        time_cost: ' ',
        memory_cost: ' '
      }
    },
    mounted () {
      let Code = storage.get(buildProblemCodeKey(-1))
      let OverallCode = storage.get(buildProblemCodeKey('Overall'))
      let exited = false
      if (Code) {
        this.language = Code.language
        this.code = Code.code
        this.theme = Code.theme
        exited = true
      } else if (OverallCode) {
        this.language = OverallCode.language
        this.theme = OverallCode.theme
        exited = true
      }
      api.getLanguages().then(res => {
        for (var i = 0; i < res.data.data.languages.length; i++) {
          this.languages[i] = res.data.data.languages[i].name
        }
        this.languages = this.languages.sort()
        if (exited === true) {
          return
        }
        this.language = this.languages[0]
      })
    },
    methods: {
      onChangeLang (newLang) {
        this.language = newLang
      },
      onChangeTheme (newTheme) {
        this.theme = newTheme
      },
      onResetToTemplate () {
        this.$Modal.confirm({
          content: this.$i18n.t('m.Are_you_sure_you_want_to_reset_your_code'),
          onOk: () => {
            this.code = ''
          }
        })
      },
      submitCode () {
        if (this.code.trim() === '') {
          this.$error(this.$i18n.t('m.Code_can_not_be_empty'))
          return
        }
        this.submitting = true
        this.result = {result: 9}
        let data = {
          language: this.language,
          code: this.code,
          input: this.input
        }
        if (this.captchaRequired) {
          data.captcha = this.captchaCode
        }
        api.IDE(data).then(res => {
          var resdata = res.data.data
          if (res.data.data.err) {
            this.output = resdata.data
            this.submitting = false
            this.time_cost = 'Error '
            this.memory_cost = 'Error '
          } else {
            this.output = resdata.data[0]['output']
            this.submitting = false
            this.time_cost = resdata.data[0]['real_time']
            this.memory_cost = parseInt(resdata.data[0]['memory'] / 1024 / 1024)
          }
        }, () => {
          this.submitting = false
        })
      },
      beforeunloadFn (e) {
        storage.set(buildProblemCodeKey('Overall'), {
          language: this.language,
          theme: this.theme
        })
        storage.set(buildProblemCodeKey(-1), {
          code: this.code,
          language: this.language,
          theme: this.theme
        })
      }
    },
    computed: {
      submissionStatus () {
        return {
          text: JUDGE_STATUS[this.result.result]['name'],
          color: JUDGE_STATUS[this.result.result]['color']
        }
      },
      submissionRoute () {
        if (this.contestID) {
          return {name: 'contest-submission-list', query: {problemID: this.problemID}}
        } else {
          return {name: 'submission-list', query: {problemID: this.problemID}}
        }
      }
    },
    created () {
      window.addEventListener('beforeunload', e => this.beforeunloadFn(e))
    },
    destroyed () {
      window.removeEventListener('beforeunload', e => this.beforeunloadFn(e))
    },
    beforeRouteLeave (to, from, next) {
      // 防止切换组件后仍然不断请求
      clearInterval(this.refreshStatus)
      storage.set(buildProblemCodeKey(-1), {
        code: this.code,
        language: this.language,
        theme: this.theme
      })
      storage.set(buildProblemCodeKey('Overall'), {
        language: this.language,
        theme: this.theme
      })
      next()
    },
    watch: {
      '$route' () {
      }
    },
    getSubmission () {
      this.loading = true
      api.getSubmission(this.$route.params.id).then(res => {
        this.loading = false
        let data = res.data.data
        if (data.info && data.info.data && !this.isConcat) {
          // score exist means the submission is OI problem submission
          if (data.info.data[0].score !== undefined) {
            this.isConcat = true
          }
          if (this.isAdminRole) {
            this.isConcat = true
          }
        }
        this.submission = data
      }, () => {
        this.loading = false
      })
    }
  }
</script>

<style lang="less" scoped>
  .card-title {
    // margin-left: 8px;
  }
  .flex-container {
    #problem-main {
      flex: auto;
      // margin-right: 18px;
    }
    #right-column {
      flex: none;
      width: 220px;
    }
  }
  #problem-content {
    margin-top: -50px;
    .title {
      font-size: 20px;
      font-weight: 400;
      margin: 25px 0 8px 0;
      color: #3091f2;
      .copy {
        padding-left: 8px;
      }
    }
    p.content {
      margin-left: 25px;
      margin-right: 20px;
      font-size: 15px
    }
    .sample {
      align-items: stretch;
      &-input, &-output {
        width: 50%;
        flex: 1 1 auto;
        display: flex;
        flex-direction: column;
        margin-right: 5%;
      }
      pre {
        flex: 1 1 auto;
        align-self: stretch;
        border-style: solid;
        background: transparent;
      }
    }
  }
  #submit-code {
    // margin-top: 20px;
    // margin-bottom: 20px;
    .status {
      float: left;
      span {
        margin-right: 10px;
        margin-left: 10px;
      }
    }
    .captcha-container {
      display: inline-block;
      .captcha-code {
        width: auto;
        margin-top: -20px;
        margin-left: 20px;
      }
    }
  }
  #info {
    margin-bottom: 20px;
    margin-top: 20px;
    ul {
      list-style-type: none;
      li {
        border-bottom: 1px dotted #e9eaec;
        margin-bottom: 10px;
        p {
          display: inline-block;
        }
        p:first-child {
          width: 90px;
        }
        p:last-child {
          float: right;
        }
      }
    }
  }
  .fl-right {
    float: right;
  }
  #pieChart {
    .echarts {
      height: 250px;
      width: 210px;
    }
    #detail {
      position: absolute;
      right: 10px;
      top: 10px;
    }
  }
  #pieChart-detail {
    margin-top: 20px;
    width: 500px;
    height: 480px;
  }
</style>

