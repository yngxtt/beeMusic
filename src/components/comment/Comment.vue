<template>
  <div class="comment">
    <!-- 评论区 -->
    <div class="area-wrap" ref="areaWrapRef">
      <textarea
        class="text-area"
        ref="textAreaRef"
        v-model="commentInfo.content"
        @keyup.enter="sendComment"
      ></textarea>
      <div class="word-num">{{ restNum }}</div>
    </div>
    <div class="btn-wrap mtop-10">
      <div class="at-btn">
        <button class="font-18 no-btn" @click="commentInfo.content += '@'">
          @
        </button>
        <button class="font-18 no-btn" @click="addTopic">#</button>
      </div>
      <div class="send-btn">
        <button class="btn btn-white" @click="sendComment">评论</button>
      </div>
    </div>
    <!-- 热门评论 -->
    <div class="hot-wrap mtop-20" v-if="hotList.length !== 0">
      <div class="font-16 font-bold">精彩评论</div>
      <ComentItem
        v-for="item in hotList"
        :key="item.commentId"
        :item="item"
        identy="hot"
        @like="handleLike"
        @reply="handleReply"
        @clickUser="toUserDetail"
      ></ComentItem>
      <div class="more-btn-wrap mtop-20">
        <button class="btn btn-white">更多精彩评论></button>
      </div>
    </div>
    <!-- 最新评论 -->
    <div ref="newListRef" class="hot-wrap mtop-20" v-if="newList.length !== 0">
      <div class="font-16 font-bold">最新评论({{ newCount }})</div>
      <ComentItem
        v-for="item in newList"
        :key="item.commentId"
        :item="item"
        identy="new"
        @like="handleLike"
        @reply="handleReply"
        @clickUser="toUserDetail"
      ></ComentItem>
      <div class="flex_center" style="margin-top: 10px">
        <el-pagination
          @current-change="handleCurrentChange"
          :current-page="currentPage"
          :page-size="20"
          layout="prev, pager, next"
          :total="newCount"
          background
        >
        </el-pagination>
      </div>
    </div>
    <div v-if="newCount === 0" style="color: #9f9f9f; text-align: center">
      还没有评论，快来抢沙发~
    </div>
  </div>
</template>

<script>
import ComentItem from './CommentItem'
import {
  getHotComment,
  getNewComment,
  sendComment,
  likeComment
} from '@/api/api_comment'
import { mapState } from 'vuex'
export default {
  components: { ComentItem },
  props: {
    id: {
      type: [Number, String],
      required: true
    },
    /* type: 0: 歌曲 1: mv 2: 歌单 3: 专辑 4: 电台 5: 视频 6: 动态 */
    type: {
      type: Number,
      required: true
    },
    active: Boolean,
    /* 滚动条所在元素 */
    scrollDom: {
      type: String,
      require: true,
      default: 'body'
    },
    /* 滚动的偏移量 */
    scrollOffset: {
      type: Number,
      default: 10
    }
  },
  data() {
    return {
      currentPage: 1,
      /* 热评 */
      hotList: [],
      /* 新评 */
      newList: [],
      /* 新评的检索信息 */
      newQuery: {
        id: this.id,
        offset: 0,
        limit: 20,
        before: 0,
        type: this.type
      },
      newCount: 0,
      commentInfo: {
        /* 1评论 2回复 0删除 */
        t: 1,
        type: this.type,
        id: this.id,
        content: '',
        commentId: 0
      },
      replyName: ''
    }
  },
  computed: {
    restNum() {
      return 140 - this.commentInfo.content.length
    },
    ...mapState(['isLogin'])
  },
  watch: {
    active: {
      immediate: true,
      handler(val) {
        if (!val || this.newList.length !== 0) return
        this.getHotComment()
        this.getNewComment()
      }
    },
    'commentInfo.content'(val) {
      if (val == '') {
        this.commentInfo.t = 1
      }
    },
    id(val) {
      /* 监听ID，改变后重新获取数据 */
      console.log('评论的资源ID改变了')
      if (!val) return
      this.hotList = []
      console.log('监听ID', val, this.commentInfo.id)
      this.commentInfo.id = val
      this.newQuery.id = val
      this.clearCommentInfo()
      this.getHotComment()
      this.getNewComment()
    }
  },
  methods: {
    /* 重置待发送评论信息 */
    clearCommentInfo() {
      this.replyName = ''
      this.commentInfo.content = ''
      console.log('重置评论')
      this.commentInfo.commentId = 0
      this.commentInfo.t = 1
    },
    addTopic() {
      this.commentInfo.content += '#输入想说的话题#'
    },
    /* 发送评论 */
    async sendComment() {
      if (!this.isLogin) return this.$message.error('需要登录')
      if (this.restNum < 0) return this.$message.error('字数过长')
      if (this.commentInfo.commentId !== 0) {
        this.commentInfo.content = this.commentInfo.content.replace(
          '回复' + this.replyName + ':',
          ''
        )
      }
      console.log('send')
      const res = await sendComment(this.commentInfo)
      if (res.code !== 200) return
      this.$message.success('发送成功')
      console.log(res)
      this.commentInfo.content = ''
      this.commentInfo.t = 1
      this.commentInfo.commentId = 0
      console.log('刷新新列表')
      /* 当前在第一页的话，过500ms再请求更新最新列表，不然会出错 */
      if (this.newQuery.offset === 0)
        window.setTimeout(() => {
          this.getNewComment()
        }, 500)
    },
    toUserDetail(id) {
      console.log(this.type)
      if (typeof id !== 'number') return
      if (this.type === 0) this.$emit('closePlayView')
      if (this.$route.path !== '/userdetail/' + id)
        this.$router.push('/userdetail/' + id)
    },
    /* 获取热门评论 */
    async getHotComment() {
      if (this.hotList.length !== 0) return
      const res = await getHotComment(this.id, this.type, 5)
      if (res.code !== 200) return
      console.log(res.hotComments)
      this.hotList = res.hotComments
    },
    /* 获取最新评论 */
    async getNewComment() {
      const res = await getNewComment(this.newQuery)
      if (res.code !== 200) return
      console.log(res)
      this.newCount = res.total
      this.newList = res.comments
    },
    /* 点赞的回调 */
    handleLike(info) {
      if (!this.isLogin) return this.$message.error('需要登录')
      console.log(info)
      let obj = { id: this.id, cid: info.cid, t: 1, type: this.type }
      if (info.liked) {
        obj.t = 0
      }
      this.like(obj, info.identy)
    },
    /* 点赞的请求及处理 */
    async like(obj, type) {
      const res = await likeComment(obj)
      if (res.code !== 200) return
      if (type == 'new') {
        let index = this.newList.findIndex((item) => item.commentId == obj.cid)
        this.newList[index].liked = !this.newList[index].liked
        if (obj.t) {
          this.newList[index].likedCount++
        } else {
          this.newList[index].likedCount--
        }
      } else if (type == 'hot') {
        let index = this.hotList.findIndex((item) => item.commentId == obj.cid)
        this.hotList[index].liked = !this.hotList[index].liked
        if (obj.t) {
          this.hotList[index].likedCount++
        } else {
          this.hotList[index].likedCount--
        }
      }
    },
    /* 回复的回调 */
    handleReply(info) {
      if (!this.isLogin) return this.$message.error('需要登录')
      console.log(info)
      this.replyName = info.name
      this.commentInfo.content = '回复' + info.name + ':'
      this.commentInfo.commentId = info.cid
      this.commentInfo.t = 2

      this.toTopAnimation(
        this.$refs.areaWrapRef.offsetTop - this.scrollOffset,
        document.querySelector(this.scrollDom),
        600
      )

      this.$nextTick(() => {
        this.$refs.textAreaRef.focus()
      })
    },
    /* 页码变化的回调 */
    handleCurrentChange(val) {
      console.log(val)
      this.currentPage = val
      this.newQuery.offset = (val - 1) * 20
      this.getNewComment()
      this.toTopAnimation(
        this.$refs.newListRef.offsetTop - this.scrollOffset,
        document.querySelector(this.scrollDom),
        600
      )
    },
    /* 滚动条由下至上的动画js动画 */
    toTopAnimation(target, scrollDom, ms = 500) {
      let start
      let begin = scrollDom.scrollTop
      let size = (begin - target) / ms
      const step = (timestamp) => {
        if (start === undefined) start = timestamp
        const elapsed = timestamp - start
        /* 防止上滑过头 */
        scrollDom.scrollTop = Math.max(begin - size * elapsed, target)
        if (elapsed < ms) {
          // 在ms毫秒后停止动画
          window.requestAnimationFrame(step)
        }
      }
      window.requestAnimationFrame(step)
    }
  }
}
</script>

<style lang="less" scoped>
.area-wrap {
  position: relative;
  .text-area {
    box-sizing: border-box;
    width: 100%;
    height: 60px;
    border: 1px solid #e5e5e5;
    outline: none;
    font-size: 14px;
    border-radius: 4px;
    padding: 4px 10px;
    resize: none;
    word-break: break-all;
    word-wrap: break-word;
  }
  .word-num {
    position: absolute;
    bottom: 4px;
    right: 4px;
    color: #dfcfdf;
  }
}
.btn-wrap {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.more-btn-wrap {
  text-align: center;
}
</style>