<template>
  <div class="comments-part">
    <div class="comments-part__header">
      <v-row no-gutters>
        <v-col>
          <span class="likes-counter">
            <span class="likes-counter__image-container">
              <img
                v-if="idea.likesCount > 0"
                src="~/assets/images/lamp_on.png"
                class="likes-counter__image on"
              />
              <img
                v-else
                src="~/assets/images/lamp_off.png"
                class="likes-counter__image"
              />
            </span>
            <span class="likes-counter__label">{{ idea.likesCount }}</span>
          </span>
        </v-col>
        <v-col style="text-align: right">
          <span class="comments-counter">
            <img
              src="~/assets/images/comment.png"
              class="comments-counter__image"
            />
            <span class="comments-counter__label">{{
              idea.commentsCount
            }}</span>
          </span>
        </v-col>
      </v-row>
    </div>
    <div ref="scroller" class="comments-part__container">
      <div class="comments-part__container__table">
        <div class="comments-part__container__table__row">
          <div
            v-if="commentList.length"
            ref="commentsCol"
            class="comments-part__container__table__col"
          >
            <div
              v-if="idea.commentsCount > commentList.length && !deletingComment"
              style="text-align: center; padding: 5px 0; cursor: pointer;"
              @click="loadComments"
            >
              <v-btn small :loading="loadingMore">Load More...</v-btn>
            </div>

            <idea-comments-comment
              v-for="comment in commentList"
              :key="comment.commentId"
              :comment="comment"
              @onDeleteComment="onDeleteComment"
            ></idea-comments-comment>
          </div>
          <div v-else class="comments-part__container__table__col empty">
            There are not comments yet. <br />
            Add the first one?
          </div>
        </div>
      </div>
    </div>
    <div class="comments-part__input-container">
      <v-text-field
        v-model="newCommentText"
        style="border-radius: 0; border: 1px solid #ebe7ed;"
        label="Say something..."
        single-line
        flat
        height="64"
        hide-details
        solo
        @keydown.enter="onAddCommentAttempt"
      >
        <template v-slot:append>
          <v-slide-y-transition hide-on-leave>
            <v-icon
              v-if="readyForSend && !showAddCommentLoader"
              @click="onAddCommentAttempt"
              >fa-paper-plane
            </v-icon>
          </v-slide-y-transition>
          <v-icon v-if="showAddCommentLoader" @click="onAddCommentAttempt"
            >fas fa-circle-notch fa-spin flag-icon
          </v-icon>
        </template>
      </v-text-field>
    </div>
    <simple-dialog-popup ref="simpleDialogPopup"></simple-dialog-popup>
    <on-first-comment-instantiated-dialog
      ref="onFirstCommentInstantiatedDialog"
    ></on-first-comment-instantiated-dialog>
    <on-un-auth-add-idea-ask-email-dialog
      ref="onUnAuthAddIdeaAskEmailDialog"
    ></on-un-auth-add-idea-ask-email-dialog>
    <on-un-auth-add-idea-ask-name-dialog
      ref="onUnAuthAddIdeaAskNameDialog"
    ></on-un-auth-add-idea-ask-name-dialog>
  </div>
</template>

<script>
import nanoid from 'nanoid'
import { graphqlOperation } from '@aws-amplify/api'
import IdeaCommentsComment from './IdeaCommentsComment'
import onUnAuthAddIdeaAskEmailDialog from './onUnAuthAddIdeaAskEmailDialog'
import addComment from '~/graphql/mutations/addComment'
import getComments from '~/graphql/query/getComments'
import deleteComment from '~/graphql/mutations/deleteComment'
import simpleDialogPopup from '~/components/dialogs/simpleDialogPopup'
import checkEmailBelongsToExistingUser from '@/graphql/query/checkEmailBelongsToExistingUser'
import OnUnAuthAddIdeaAskNameDialog from '@/components/ideaDetail/onUnAuthAddIdeaAskNameDialog'
import addIdeaTemporaryComment from '@/graphql/mutations/addIdeaTemporaryComment'
import getIdeaTemporaryComment from '@/graphql/query/getIdeaTemporaryComment'
import OnFirstCommentInstantiatedDialog from '@/components/ideaDetail/onFirstCommentInstantiatedDialog'
import setWasWelcomed from '@/graphql/mutations/setWasWelcomed'

const COMMENTS_COUNT = 25

export default {
  name: 'IdeaComments',
  components: {
    OnFirstCommentInstantiatedDialog,
    OnUnAuthAddIdeaAskNameDialog,
    IdeaCommentsComment,
    simpleDialogPopup,
    onUnAuthAddIdeaAskEmailDialog
  },
  props: {
    idea: {
      type: Object,
      required: true
    }
  },
  data() {
    return {
      tmpComment: {},
      newCommentText: '',
      showAddCommentLoader: false,
      commentList: [],
      nextToken: null,
      loadingMore: false,
      deletingComment: false,
      temporaryCommentId: undefined
    }
  },
  computed: {
    readyForSend() {
      return this.newCommentText.length > 0
    },
    isAuthenticated() {
      return this.$store.getters['userData/isAuthenticated']
    }
  },
  mounted() {
    this.doInitialCommentsLoading()
  },
  methods: {
    scrollToBottom() {
      this.$nextTick(() => {
        this.$refs.scroller.scrollTop = this.$refs.scroller.scrollHeight
      })
    },
    doInitialCommentsLoading() {
      if (this.idea.commentsCount > 0) {
        const fakeCommentsCount = Math.min(
          COMMENTS_COUNT,
          this.idea.commentsCount
        )
        const fakeComments = []
        for (let i = 0; i < fakeCommentsCount; i++) {
          fakeComments.push({
            fake: true,
            commentId: nanoid()
          })
        }
        this.commentList = fakeComments
        this.scrollToBottom()
      }
      let additionalAction

      if (this.$route.query.aa) {
        additionalAction = this.$route.query.aa
        if (additionalAction === 'itc') {
          // instantiate temporary comment
          this.temporaryCommentId = this.$route.query.tci
          this.commentList.push({
            fake: true,
            temporary: true,
            commentId: this.temporaryCommentId
          })
          this.getTemporaryComment(this.temporaryCommentId)
        }
      }
      this.loadComments()
    },
    async getTemporaryComment(commentId) {
      const result = await this.$amplifyApi.graphql({
        query: getIdeaTemporaryComment,
        variables: {
          commentId
        },
        authMode: 'API_KEY'
      })
      if (result.data.getIdeaTemporaryComment.result.ok) {
        const temporaryComment = result.data.getIdeaTemporaryComment.comment
        temporaryComment.temporary = true
        temporaryComment.userName = this.$store.getters['userData/userName']
        this.commentList.splice(this.commentList.length - 1, 1)
        this.commentList.push(temporaryComment)
        this.sendComment(temporaryComment.body, true)
        this.scrollToBottom()
      } else {
        this.commentList.splice(this.commentList.length - 1, 1)
        this.$router.replace({ query: null })
      }
    },
    async onDeleteComment(comment) {
      const confirmed = await this.$refs.simpleDialogPopup.show(
        'Delete Comment',
        'Are you sure you want to delete this Comment?'
      )
      if (!confirmed) {
        return
      }
      this.deletingComment = true
      this.$store.commit('layoutState/showProgressBar')
      try {
        this.commentList = this.commentList.filter(
          c => c.commentId !== comment.commentId
        )
        await this.$amplifyApi.graphql(
          graphqlOperation(deleteComment, {
            // body: body,
            // userId: this.$store.getters['cognito/userSub'],
            ideaId: this.idea.ideaId,
            ideaName: this.idea.ideaName,
            ideaOwnerId: this.idea.userId,
            commentId: comment.commentId
          })
        )
        // remove comment form comment list array
        this.idea.commentsCount -= 1
        this.$emit('onNotification', {
          type: 'success',
          message: 'Comment Deleted!'
        })
      } catch (err) {
        this.$emit('onNotification', {
          type: 'error',
          message: "Can't Delete Comment!"
        })
      }
      this.deletingComment = false
      this.$store.commit('layoutState/hideProgressBar')
    },
    onAddCommentAttempt() {
      const commentText = this.newCommentText
      this.newCommentText = ''
      this.showAddCommentLoader = true
      const isAuthenticated = this.$store.getters['userData/isAuthenticated']
      if (isAuthenticated) {
        this.sendComment(commentText)
      } else {
        this.appendFakeCommentAndEncourageToRegisterOrSignUp(commentText)
      }
    },
    checkEmailBelongsToExistingUser(email) {
      return this.$amplifyApi.graphql({
        query: checkEmailBelongsToExistingUser,
        variables: {
          email
        },
        authMode: 'API_KEY'
      })
    },
    addTemporaryFakeComment(text) {
      this.tmpComment = { userName: 'Me', body: text, temporary: true }
      this.commentList.push(this.tmpComment)
      this.scrollToBottom()
    },
    removeTemporaryFakeComment() {
      this.commentList.splice(this.commentList.indexOf(this.tmpComment), 1)
    },
    processCommentInstantiation() {
      const wasWelcomed = this.$store.getters['userData/wasWelcomed']
      if (wasWelcomed) {
        this.$refs.simpleDialogPopup.show(
          'Welcome back!',
          'Thanks for posting that comment!',
          undefined,
          null
        )
      } else {
        this.$refs.onFirstCommentInstantiatedDialog.show()
        this.$amplifyApi.graphql(
          graphqlOperation(setWasWelcomed, {
            userId: this.$store.getters['userData/userId']
          })
        )
      }
      this.$router.replace({ query: null })
    },
    async createTemporaryCommentInDB(userId, commentText) {
      const res = await this.$amplifyApi.graphql({
        query: addIdeaTemporaryComment,
        variables: {
          userId,
          body: commentText,
          ideaId: this.idea.ideaId,
          ideaName: this.idea.title,
          ideaOwnerId: this.idea.userId
        },
        authMode: 'API_KEY'
      })
      return res.data.addIdeaTemporaryComment.comment
    },
    async requestAuthAndProcessComment(userId, email, name, commentText) {
      this.$store.commit('layoutState/showProgressBar')
      const comment = await this.createTemporaryCommentInDB(userId, commentText)
      await this.$amplifyApi.post('RequestLogin', '', {
        body: { email, commentId: comment.commentId }
      })
      this.$store.commit('layoutState/hideProgressBar')
      this.$refs.simpleDialogPopup.show(
        'Welcome back!',
        "It looks like you weren't signed in. We just sent you a verification email. Please check your inbox and click on the link and we'll post your comment ASAP.",
        undefined,
        null
      )
    },
    async registerUserAndProcessComment(email, name, commentText) {
      try {
        this.$store.commit('layoutState/showProgressBar')
        const res = await this.$store.dispatch('cognito/registerUser', {
          username: email,
          password: nanoid(),
          attributes: {
            name
          }
        })
        const userId = res.userSub
        const comment = await this.createTemporaryCommentInDB(
          userId,
          commentText
        )
        await this.$amplifyApi.post('RequestLogin', '', {
          body: { email, commentId: comment.commentId }
        })
        this.$store.commit('layoutState/hideProgressBar')
        this.$refs.simpleDialogPopup.show(
          'Thanks!',
          "We just sent you an email to confirm that you're a real person :) Please check your inbox then click on the link and we'll post your comment ASAP.",
          'OK',
          null
        )
      } catch (e) {
        this.$store.commit('layoutState/hideProgressBar')
      }
    },
    async appendFakeCommentAndEncourageToRegisterOrSignUp(commentText) {
      this.showAddCommentLoader = false
      this.addTemporaryFakeComment(commentText)
      try {
        let email = await this.$refs.onUnAuthAddIdeaAskEmailDialog.show()
        email = email.toLowerCase()
        this.$store.commit('layoutState/showProgressBar')
        const result = await this.checkEmailBelongsToExistingUser(email)
        const belongsToExistingUser =
          result.data.checkEmailBelongsToExistingUser.belongsToExistingUser
        this.$store.commit('layoutState/hideProgressBar')
        if (belongsToExistingUser) {
          const userId = result.data.checkEmailBelongsToExistingUser.userId
          this.requestAuthAndProcessComment(userId, email, name, commentText)
        } else {
          const name = await this.$refs.onUnAuthAddIdeaAskNameDialog.show()
          this.registerUserAndProcessComment(email, name, commentText)
        }
      } catch (e) {
        this.removeTemporaryFakeComment()
        this.$store.commit('layoutState/hideProgressBar')
      }
    },
    async sendComment(commentText, instantiation = false) {
      try {
        const result = await this.$amplifyApi.graphql(
          graphqlOperation(addComment, {
            body: commentText,
            ideaId: this.idea.ideaId,
            ideaOwnerId: this.idea.userId,
            userName: this.$store.getters['userData/userName'],
            userSlug: this.$store.getters['userData/slug']
          })
        )
        const newComment = result.data.addComment.comment
        if (instantiation) {
          newComment.temporary = true
          this.$set(this.commentList, this.commentList.length - 1, newComment)
          this.$nextTick(() => {
            this.$set(
              this.commentList[this.commentList.length - 1],
              'temporary',
              false
            )
          })
          this.idea.commentsCount += 1
          setTimeout(() => {
            this.processCommentInstantiation()
          }, 2000)
          this.scrollToBottom()
          return
        } else {
          this.commentList.push(newComment)
          this.idea.commentsCount += 1
          this.scrollToBottom()
        }
        // this.fetchCommentList()
        this.showAddCommentLoader = false

        this.$emit('onNotification', {
          type: 'success',
          message: 'Comment Added!'
        })
      } catch (err) {
        this.$emit('onNotification', {
          type: 'error',
          message: "Can't add Comment. Please reload page and try again."
        })
        this.showAddCommentLoader = false
        this.newCommentText = commentText
      }
    },
    async loadComments() {
      this.loadingMore = true
      try {
        const result = await this.$amplifyApi.graphql({
          query: getComments,
          variables: {
            ideaId: this.$route.params.ideaId,
            limit: COMMENTS_COUNT,
            nextToken: this.nextToken
          },
          authMode: this.$store.getters['userData/isAuthenticated']
            ? undefined
            : 'API_KEY'
        })
        if (this.nextToken) {
          const lastEl = this.$refs.commentsCol.children[1]
          for (const comment of result.data.getComments.items) {
            this.commentList.unshift(comment)
          }
          this.$nextTick(() => {
            lastEl.scrollIntoView()
            this.$refs.scroller.scrollTop -= 55
          })
        } else {
          this.commentList = this.commentList.filter(
            c => c.commentId === this.temporaryCommentId
          ) // remove comments placeholder but not temporary comment - it is loaded separately
          for (const comment of result.data.getComments.items) {
            this.commentList.unshift(comment)
          }
          this.scrollToBottom()
        }

        this.nextToken = result.data.getComments.nextToken
      } catch (e) {
        this.$emit('onNotification', {
          type: 'error',
          message: "Can't load Comments. Please reload page and try again."
        })
        this.commentList = []
      }
      this.loadingMore = false
    }
  }
}
</script>

<style scoped lang="scss">
$counters-font-size: 18px;
.likes-counter {
  display: inline-block;

  &__image-container {
    display: inline-block;
    vertical-align: bottom;
    min-width: $counters-font-size + 7px;
    text-align: center;
    min-height: $counters-font-size + 5px;
  }

  &__image {
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size + 1px;

    &.on {
      height: $counters-font-size + 5px;
    }
  }

  &__label {
    margin-left: 5px;
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size;
    line-height: $counters-font-size;
    font-size: $counters-font-size;
  }
}

.comments-counter {
  &__image {
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size;
  }

  &__label {
    margin-left: 8px;
    display: inline-block;
    vertical-align: bottom;
    height: $counters-font-size;
    line-height: $counters-font-size;
    font-size: $counters-font-size;
  }
}

.comments-part {
  /*background-color: #aed242;*/
  @media (max-width: $screen-sm-max) {
    height: calc(80vh - 88px);
  }
  @media (min-width: $screen-md-min) {
    height: calc(100vh - 88px);
  }

  overflow: hidden;

  &__header {
    padding: 15px;
    height: 64px;
    background-color: #ebe7ed;
  }

  &__container {
    background-color: #ebe7ed;
    @media (max-width: $screen-sm-max) {
      height: calc(80vh - 92px - 64px - 64px);
    }
    @media (min-width: $screen-md-min) {
      height: calc(100vh - 92px - 64px - 64px);
    }
    overflow: auto;
    /*height: 100%;*/

    display: block;

    &__table {
      width: 100%;
      height: 100%;
      display: table;

      &__row {
        display: table-row;
      }

      &__col {
        display: table-cell;
        vertical-align: bottom;

        &.empty {
          vertical-align: middle;
          text-align: center;
          color: #c0b7c5;
          font-size: 25px;
        }
      }
    }
  }

  &__input-container {
    height: 64px;
  }
}
</style>
