<template>
  <view class="page-bottom draw">
    <image mode="widthFix" style="width: 100%;" src="../../static/touxiang2_head.png" />
    <view style="margin-top: -3px" class="container">
      <view class="item-top flex-row justify-between items-center">
        <view>
          <view class="keyword_label">关键词</view>
          <view class="xcc_label">想要一个什么头像</view>
        </view>
        <view>
          <u-button @click="$toPage('./example')" type="primary" size="mini" text="查看示例 >"></u-button>
        </view>
      </view>
      <u--textarea autoHeight maxlength="-1" class="mt" v-model="value1" placeholder="输入关键词，如：可爱的猫，星空下的树">
      </u--textarea>
    </view>

    <view class="container mt ">
      <view class="item-top flex-row justify-between items-center">
        <view>
          <view class="keyword_label">推荐关键词</view>
        </view>
        <view>
          <u-button @click="getPromptSuggest" type="primary" size="mini" text="换一组"></u-button>
        </view>
      </view>
      <view class="keyword-boxs flex-wrap mt">
        <view v-for="(item,index) in tagsList" :key="index" class="xcc-tag">
          <u-tag @click="selectTagesHandle(item)" :borderColor="item.color" :bgColor="item.color" size="mini"
            :text="item.serverData.chinesePrompt"></u-tag>
        </view>
      </view>
    </view>

    <view class="container mt ">
      <view class="item-top flex-row justify-between items-center">
        <view>
          <view class="keyword_label">风格选择（可选）</view>
        </view>
        <view>
          <u-button type="error" size="mini" @click="style = null" text="清除选择"></u-button>
        </view>
      </view>

      <view class="keyword-boxs flex-wrap mt">
        <u-grid col="4" :border="false">
          <u-grid-item @click="clickStyleHandle(item)" v-for="(item,index) in styleList " :key="index">
            <view class="style_item ">
              <view class="style_img_box">
                <image :class="['style_img',style === item.serverData.objectId ? 'style_img_box_active' : null]"
                  :src="item.serverData.image.serverData.url" alt="" />
              </view>
              <view :class="['style_text',style === item.serverData.objectId ? 'style_text_active' : null]">
                {{item.serverData.title}}
              </view>
            </view>
          </u-grid-item>
        </u-grid>
      </view>
      <view class="alert-text">
        注意：以上风格只供参考，画头酱更多是根据您输入关键词来确定风格，你可以输入你想要的风格的关键词。
      </view>

    </view>

    <view class="mt submit-box flex-row pd">
      <u-button :loading="loading" @click="gHandle" type="primary" shape="circle" text="开始画（消耗3点）"></u-button>
    </view>

    <!-- 二次确认 -->
    <u-modal :show="modalShow" title="生成将消耗3点作画数" @cancel="modalShow = false" @confirm="confirmHadnle" showCancelButton :content='content'></u-modal>
  </view>
</template>

<script>
  import {
    rdmRgbColor
  } from '@/utils'
  import {
    BaiDuTrans
  } from '../../api/index.js'
  import AV from '../../utils/av-core-min'
  import {
    mapState
  } from 'vuex'
  export default {
    data() {
      return {
        modalShow: false,
        loading: false,
        value1: '',
        style: null,
        styleList: [],
        tagsList: [],
        content: `
        生成每张图片需要一定的时间，请耐心等待结果。
        本服务不可生成带有[色情、暴恐、政治]敏感内容图片，请勿使用相关提示词! 
        由于AI图像生成具有不确定性，生成的图片将进行机器审核。
        `
      }
    },
    computed: {
      ...mapState(['drawKeywords', 'userInfo'])
    },
    onShow() {
      if (JSON.stringify(this.drawKeywords) != '{}') {
        console.log(this.drawKeywords)
        const {
          chinesePrompt,
          promptStyle
        } = this.drawKeywords
        this.value1 = chinesePrompt
        if (promptStyle) {
          this.style = promptStyle.serverData.objectId
        } else {
          this.style = null
        }
        this.$store.commit('SET_DRAW_KEYWORDS', {})
      }
    },
    onLoad({
      keyword
    }) {
      if (keyword) {
        this.value1 = keyword
      }
      this.initHandle()
    },
    methods: {
      async initHandle() {
        try {
          // 没有用户信息,的情况下请求登录
          if (!this.userInfo) {
            await this.$store.dispatch('loginAction')
          }
          this.getPaintingStyleRecommend()
          this.getPromptSuggest()
        } catch (e) {
          //TODO handle the exception
        }
      },
      async confirmHadnle() {
        this.modalShow = false
        // this.loading = true
        try {
          const styleVlaue = this.styleList.find(
            (item) => item.serverData.objectId === this.style
          )

          const transRes = await BaiDuTrans(this.value1)
          const translatePrompt = transRes[1].data.trans_result[0].dst

          const res = await AV.Cloud.run('createPaintingOrder', {
            chinesePrompt: this.value1,
            translatePrompt: translatePrompt,
            promptStyleId: this.style || undefined,
            promptStyleValue: styleVlaue ? styleVlaue.serverData.prompt : undefined
          })

          if (res.error) {
            uni.showToast({
              title: res.error,
              icon: 'none',
              mask: true
            })
            return
          }

          setTimeout(() => {
            uni.showToast({
              title: '提交AI成功，请耐心等待生成结果！',
              icon: 'none'
            })
          }, 100)
          this.value1 = ''
          this.style = null

          // 刷新让主页刷新
          this.$store.commit('SET_INDEX_FLUSH', true)
          uni.switchTab({
            url: '/pages/index/index'
          })
        } catch (error) {
          uni.showToast({
            title: error.message.split(' ')[0],
            icon: 'none',
            mask: true
          })
          console.log(e)
        }
        // this.loading = false
      },
      async gHandle() {
        if (!this.value1) {
          uni.showToast({
            title: '输入一些关键词再试一下',
            icon: 'none'
          })
          return
        }

        this.modalShow = true

      },
      selectTagesHandle(item) {
        var comma = !this.value1 ? '' : '，'
        this.value1 = this.value1 + comma + item.serverData.chinesePrompt
      },
      async getPromptSuggest() {
        const res = await AV.Cloud.run('getPromptSuggest')
        this.tagsList = res.map((item) => ({
          ...item,
          color: rdmRgbColor()
        }))
      },
      async getPaintingStyleRecommend() {
        const res = await AV.Cloud.run('getPaintingStyleRecommend')
        this.styleList = res
      },
      clickStyleHandle(item) {
        if (this.style === item.serverData.objectId) {
          this.style = null
        } else {
          this.style = item.serverData.objectId
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
  .draw {
    padding-bottom: 80px;
  }

  /deep/.u-textarea__field {
    min-height: 70px;
  }

  .alert-text {
    font-size: 13px;
    color: $uni-color-subtitle;
    padding: 10px 5px 0;
  }

  .keyword_label {
    margin-bottom: 5px;
  }

  .xcc-tag {
    margin: 0 8px 8px 0;
  }

  .style_item {
    margin-bottom: 8px;
  }

  .style_img {
    height: 100%;
    width: 100%;
  }

  .style_img_box {
    width: 65px;
    height: 65px;
  }

  .style_img_box_active {
    border: 4px solid $uni-color-primary;
  }

  .style_text_active {
    color: $uni-color-primary;
  }

  .style_text {
    text-align: center;
    margin-top: 5px;
  }

  .xcc_label {
    margin-bottom: 5px;
  }

  .submit-box {
    position: fixed;
    bottom: 10px;
    width: 100%;

    /deep/ .u-button {
      box-shadow: 0px 10px 15px -3px rgba(0, 0, 0, 0.1);
    }
  }
</style>
